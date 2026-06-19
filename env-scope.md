# Nushell 环境变量作用域分析

## 概述

Nushell 的环境变量系统采用**多层栈式结构**结合 **Overlay（覆盖层）** 机制，实现了灵活的作用域隔离与回滚。环境变量存储在两个层次：
- **永久层**：`EngineState.env_vars` — REPL 全局持久状态
- **栈层**：`Stack.env_vars` — 运行时临时作用域，支持多层嵌套

## 核心数据结构

### Stack（运行时栈）

定义在 [stack.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L38-L67)

```rust
pub struct Stack {
    pub vars: Vec<(VarId, Value)>,
    pub env_vars: Vec<Arc<EnvVars>>,           // 环境变量栈（多层）
    pub env_hidden: Arc<HashMap<String, HashSet<EnvName>>>,  // 隐藏标记
    pub env_hide_history: Arc<HashMap<String, HashSet<EnvName>>>, // 隐藏历史
    pub active_overlays: Vec<String>,          // 活跃 overlay 列表
    pub parent_stack: Option<Arc<Stack>>,      // 父栈（父子栈模式）
    pub parent_deletions: Vec<VarId>,          // 父栈删除记录
    // ...
}
```

关键特性：
- `env_vars` 是一个 `Vec<Arc<EnvVars>>`，每个元素代表一个作用域层
- 新作用域通过向 `env_vars` 末尾推入新层来创建
- 每个层内按 overlay 名称分组存储环境变量
- `env_hidden` 用于标记 engine_state 中被隐藏的变量

### EnvVars（环境变量映射）

定义在 [stack.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L18)

```rust
pub type EnvVars = HashMap<String, HashMap<EnvName, Value>>;
```

- 外层 key 是 overlay 名称
- 内层 key 是 `EnvName`（支持大小写不敏感查找，同时保留原始大小写）
- value 是 `Value`（可以是字符串、列表等任意 Nushell 值类型）

### EngineState（引擎状态）

定义在 [engine_state.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/engine_state.rs#L104)

```rust
pub env_vars: Arc<EnvVars>,        // 永久环境变量
pub previous_env_vars: Arc<HashMap<EnvName, Value>>, // 上一次的环境变量
```

## 环境变量的读取流程

读取时按**从栈顶到栈底，再到永久层**的顺序查找，每层内按 **overlay 逆序** 查找。

### 读取算法（get_env_var）

实现于 [Stack::get_env_var](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L531-L557)

```
查找顺序：
  1. 遍历 stack.env_vars（从最后一层往第一层）
     └─ 遍历 active_overlays（从最后一个往第一个）
        └─ 在该层该 overlay 中查找变量
  2. 遍历 engine_state.env_vars（按 active_overlays 逆序）
     └─ 检查是否被 env_hidden 标记隐藏
     └─ 未隐藏则返回
```

优先级规则：
- **栈层优先于永久层**：栈中设置的变量会覆盖 engine_state 中的同名变量
- **后进先出**：越晚推入的栈层优先级越高
- **Overlay 优先级**：越后激活的 overlay 优先级越高

### 完整获取所有环境变量

实现于 [Stack::get_env_vars](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L396-L421)

1. 先遍历所有活跃 overlay，从 engine_state 收集基础变量（过滤掉被隐藏的）
2. 再遍历所有栈层，用栈中的变量覆盖/扩展结果

## 作用域创建与回滚

### 1. 普通闭包作用域（自动回滚）

通过 [`Stack::captures_to_stack`](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L330-L357) 创建新作用域：

```rust
pub fn captures_to_stack_preserve_out_dest(&self, captures: Vec<(VarId, Value)>) -> Stack {
    let mut env_vars = self.env_vars.clone();
    env_vars.push(Arc::new(HashMap::new()));  // 推入新的空环境层

    Stack {
        vars: captures,
        env_vars,                              // 包含父栈所有层 + 新层
        env_hidden: self.env_hidden.clone(),
        active_overlays: self.active_overlays.clone(),
        parent_stack: None,                    // 注意：不是父子栈模式
        // ...
    }
}
```

**回滚机制**：闭包执行完毕后，新的 `Stack` 被丢弃，新增的环境层随之消失，自动完成回滚。

典型应用：`with-env` 命令、`do` 块、循环体等

### 2. 父子栈模式

通过 [`Stack::with_parent`](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L106-L124) 创建子栈：

```rust
pub fn with_parent(parent: Arc<Stack>) -> Stack {
    Stack {
        env_vars: parent.env_vars.clone(),      // 共享父栈的环境层
        env_hidden: parent.env_hidden.clone(),
        active_overlays: parent.active_overlays.clone(),
        vars: vec![],                           // 空变量层
        parent_stack: Some(parent),             // 保留父栈引用
        parent_deletions: vec![],               // 删除记录
        // ...
    }
}
```

特点：
- 子栈与父栈**共享环境层**（不推入新层）
- 变量查找会递归到父栈
- 删除变量时记录到 `parent_deletions`，不直接修改父栈
- 支持通过 [`with_changes_from_child`](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L132-L150) 合并回父栈

### 3. with-env 命令

实现于 [with_env.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-command/src/env/with_env.rs#L54-L80)

```rust
fn with_env(engine_state: &EngineState, stack: &mut Stack, call: &Call, input: PipelineData) -> Result<PipelineData, ShellError> {
    let env: Record = call.req(engine_state, stack, 0)?;
    let capture_block: Closure = call.req(engine_state, stack, 1)?;
    let block = engine_state.get_block(capture_block.block_id);

    // 步骤1：创建新栈（推入新环境层）
    let mut stack = stack.captures_to_stack_preserve_out_dest(capture_block.captures);

    // 步骤2：在新栈上设置临时环境变量
    for (k, v) in env {
        stack.add_env_var(k, v);
    }

    // 步骤3：执行闭包
    eval_block::<WithoutDebug>(engine_state, &mut stack, block, input).map(|p| p.body)

    // 步骤4：函数返回 → stack 被丢弃 → 新环境层自动消失 → 回滚完成
}
```

**源码核对**：
- 新栈创建：[captures_to_stack_preserve_out_dest](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L336-L357) 第338-339行推入新层
- 临时变量设置：[with_env.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-command/src/env/with_env.rs#L75-L77)
- 自动回滚：新栈是函数局部变量，返回即销毁

示例测试见 [with_env.rs 测试](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-command/tests/commands/with_env.rs)

### 4. 三种机制对比总览

| 机制 | 作用域创建 | 数据流向 | 回滚方式 | 典型应用 |
|------|-----------|---------|---------|---------|
| **临时作用域回滚** | 推入新环境层 | 单向（外部→内部） | 自动（栈丢弃） | `with-env`、`do` 块 |
| **当前作用域导入** | 推入新环境层 | 双向（外部→内部→外部） | 主动同步（redirect_env） | `source-env`、`export-env` |
| **外部命令转换** | 不创建 | 单向（Nushell→子进程） | 不回滚也不持久化 | `run-external`、外部命令 |

## 机制一：临时作用域回滚（with-env 模式）

### 完整执行流程

```
调用者 Stack                          新创建的 Stack
┌───────────────────────┐            ┌───────────────────────┐
│ env_vars: [Layer0]    │  clone +   │ env_vars: [Layer0,    │
│                       │  push new  │            Layer1]    │
│                       │ ─────────► │                       │
│                       │            │ Layer1: { FOO: BAR }  │ ← 临时变量
│                       │            │                       │
└───────────────────────┘            └───────────┬───────────┘
                                                  │
                                                  ▼
                                           执行闭包
                                                  │
                                                  ▼
                                           Stack 被丢弃
                                                  │
                                                  ▼
                                     Layer1 消失 → 自动回滚
```

### 源码关键路径

1. **创建新作用域**：[Stack::captures_to_stack_preserve_out_dest](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L336-L357)
   ```rust
   let mut env_vars = self.env_vars.clone();
   env_vars.push(Arc::new(HashMap::new()));  // 关键：推入新层
   ```

2. **设置临时变量**：[Stack::add_env_var](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L273-L296)
   - 总是写入 `env_vars.last_mut()`（最新层）

3. **读取时优先级**：[Stack::get_env_var](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L531-L557)
   - 从 `env_vars.iter().rev()` 逆序查找，新层优先

4. **自动回滚**：新栈是函数局部变量，离开作用域时 Rust 自动销毁
   - `env_vars` 中的 `Arc` 引用计数归零，新层内存被释放

### 测试验证

见测试用例 [with_env_hides_variables_in_parent_scope](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-command/tests/commands/with_env.rs#L50-L60)：
```nu
$env.FOO = "1"
let before = $env.FOO                # "1"
let during = (with-env { FOO: null } { $env.FOO })  # null
let after = $env.FOO                 # "1" （自动回滚）
```

## 机制二：当前作用域导入（source-env / export-env 模式）

与临时作用域回滚的核心区别：**执行完后主动将内部环境变更同步回外部**。

### 核心函数：redirect_env

在 Nushell 中有两个 `redirect_env` 实现，功能完全一致：

1. AST 求值版本：[eval.rs::redirect_env](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/eval.rs#L368-L388)
2. IR 求值版本：[eval_ir.rs::redirect_env](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/eval_ir.rs#L1885-L1905)

```rust
fn redirect_env(engine_state: &EngineState, caller_stack: &mut Stack, callee_stack: &Stack) {
    // 步骤1：获取 caller 当前所有环境变量名（含 engine_state）
    let caller_env_vars = caller_stack.get_env_var_names(engine_state);

    // 步骤2：移除 caller 有但 callee 没有的变量（callee 中被 hide-env 了）
    for var in caller_env_vars.iter() {
        if !callee_stack.has_env_var(engine_state, var) {
            caller_stack.hide_env_var(engine_state, var);
        }
    }

    // 步骤3：将 callee 栈层的变量同步到 caller
    // 关键：只同步栈层变量（get_stack_env_vars），不同步 engine_state 的
    for (var, value) in callee_stack.get_stack_env_vars() {
        caller_stack.add_env_var(var, value);
    }

    // 步骤4：同步 config
    caller_stack.config.clone_from(&callee_stack.config);
}
```

**源码核对**：
- 变量移除：[hide_env_var](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L684-L707)
- 变量同步：[get_stack_env_vars](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L423-L440) 只返回栈层
- 添加变量：[add_env_var](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L273-L296) 写入 caller 的最新栈层

### source-env 的完整流程

实现于 [source_env.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-command/src/env/source_env.rs#L45-L109)

```
caller_stack                          callee_stack
┌───────────────────────┐            ┌───────────────────────┐
│ env_vars: [Layer0]    │ gather_   │ env_vars: [Layer0,    │
│ $env.FOO = "1"        │ captures  │            Layer1]    │
│                       │ ─────────► │                       │
│                       │            │ 执行脚本:             │
│                       │            │   $env.FOO = "2"      │
│                       │            │   $env.BAR = "new"    │
│                       │            │ 写入 Layer1          │
│                       │            │                       │
└───────────────────────┘            └───────────┬───────────┘
                                                  │
                                          redirect_env
                                                  │
                        ┌─────────────────────────┘
                        ▼                         ▼
              同步新增/修改              同步隐藏
              $env.BAR = "new"           如果 callee 隐藏了 FOO，
              $env.FOO = "2"             caller 也会隐藏
```

**源码核对**：
- 新栈创建：[gather_captures](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L359-L393) 第374-375行推入新层
- 显式调用 redirect_env：[source_env.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-command/src/env/source_env.rs#L101-L102)

### export-env 的流程

实现于 [export_env.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-command/src/env/export_env.rs#L40-L67)

与 source-env 几乎相同，区别在于：
- source-env 是执行外部脚本文件
- export-env 是执行内联块
- 都使用 `redirect_env` 同步环境

### 自定义命令的 redirect_env

当 `def` 命令的 block 设置了 `redirect_env: true` 时，IR 求值器会自动调用：

[eval_ir.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/eval_ir.rs#L1269-L1272)
```rust
let result = eval_block_with_early_return::<D>(engine_state, &mut callee_stack, block, input)
    .map(|p| p.body);

if block.redirect_env {
    redirect_env(engine_state, &mut caller_stack, &callee_stack);
}
```

### 关键细节：get_stack_env_vars vs get_env_vars

这是理解 redirect_env 行为的核心！

| 函数 | 返回范围 | 用途 |
|------|---------|------|
| [`get_stack_env_vars()`](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L423-L440) | 只返回 `stack.env_vars` 栈层的变量 **不含** `engine_state.env_vars` | redirect_env 同步时使用，只同步运行时变更 |
| [`get_env_vars()`](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L396-L421) | 返回完整环境：`engine_state.env_vars` + `stack.env_vars` | 读取完整环境时使用 |

**为什么 redirect_env 只用 get_stack_env_vars？**
- engine_state 的变量是永久的，不应该通过 redirect_env 重复同步
- redirect_env 的目的是同步**运行时变更**，即 callee 栈层中的变更
- 如果 callee 只是读取了 engine_state 的变量但没修改，不会产生栈层记录，也就不会被同步

## 机制三：外部命令环境转换（run-external 模式）

与前两种机制不同，外部命令转换**不创建新作用域**，也**不回滚**，而是将 Nushell 的环境变量转换为字符串格式传递给子进程。

### 核心函数：env_to_strings

实现于 [env.rs::env_to_strings](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/env.rs#L175-L192)

```rust
pub fn env_to_strings(engine_state: &EngineState, stack: &Stack) -> Result<HashMap<String, String>, ShellError> {
    // 关键：获取完整环境（栈层 + engine_state）
    let env_vars = stack.get_env_vars(engine_state);

    let mut env_vars_str = HashMap::new();
    for (env_name, val) in env_vars {
        // 逐个转换为字符串
        match env_to_string(&env_name, &val, engine_state, stack) {
            Ok(val_str) => {
                env_vars_str.insert(env_name, val_str);
            }
            Err(ShellError::EnvVarNotAString { .. }) => {} // 忽略无法转换的值
            Err(e) => return Err(e),
        }
    }
    Ok(env_vars_str)
}
```

### 单个变量转换：env_to_string

实现于 [env.rs::env_to_string](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/env.rs#L129-L172)

```rust
pub fn env_to_string(env_name: &str, value: &Value, engine_state: &EngineState, stack: &Stack) -> Result<String, ShellError> {
    // 步骤1：尝试用 ENV_CONVERSIONS 的 to_string 闭包转换
    match get_converted_value(engine_state, stack, env_name, value, "to_string") {
        Ok(v) => Ok(v.coerce_into_string()?),
        Err(ConversionError::ShellError(e)) => Err(e),
        Err(ConversionError::CellPathError) => {
            // 步骤2：尝试直接 coerce 为字符串
            match value.coerce_string() {
                Ok(s) => Ok(s),
                Err(_) => {
                    // 步骤3：特殊处理 PATH 变量（硬编码 fallback）
                    if env_name.to_lowercase() == "path" {
                        match value {
                            Value::List { vals, .. } => {
                                let paths: Vec<String> = vals.iter()
                                    .filter_map(|v| v.coerce_str().ok())
                                    .map(|s| expand_tilde(&*s).to_string_lossy().into_owned())
                                    .collect();
                                std::env::join_paths(paths.iter().map(AsRef::<str>::as_ref))
                                    .map(|p| p.to_string_lossy().to_string())
                                    .map_err(|_| ShellError::EnvVarNotAString { ... })
                            }
                            _ => Err(ShellError::EnvVarNotAString { ... }),
                        }
                    } else {
                        Err(ShellError::EnvVarNotAString { ... })
                    }
                }
            }
        }
    }
}
```

**源码核对**：
- 完整环境获取：[get_env_vars](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L396-L421) 包含栈层 + engine_state
- 转换闭包调用：[get_converted_value](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/env.rs#L301-L325)
- PATH 硬编码处理：[env.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/env.rs#L141-L157)

### run-external 的完整流程

实现于 [run_external.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-command/src/system/run_external.rs#L52-L350)

```
Nushell Stack                          子进程环境
┌───────────────────────┐            ┌───────────────────────┐
│ env_vars: [Layer0,    │ env_to_    │ HashMap<String, String>│
│            Layer1]    │ strings    │ FOO: "1"              │
│                       │ ─────────► │ PATH: "/usr/bin:/bin" │
│ $env.FOO = Value(1)   │            │ ...                   │
│ $env.PATH = List(...) │            │                       │
│                       │            │ （子进程只读副本）     │
│                       │            └───────────┬───────────┘
└───────────────────────┘                        │
                                                 ▼
                                          调用 std::process
                                                 │
                                                 ▼
                子进程结束 → 环境丢失 → 不影响 Nushell 环境
```

**源码核对**：
- 环境转换：[run_external.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-command/src/system/run_external.rs#L177-L178) 第178行调用 `env_to_strings`
- 清除并设置环境：[run_external.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-command/src/system/run_external.rs#L179-L180)
  ```rust
  let envs = env_to_strings(engine_state, stack)?;
  command.env_clear();
  command.envs(envs);
  ```
- PWD 配置：[run_external.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-command/src/system/run_external.rs#L174-L175)

### 关键区别：get_env_vars vs get_stack_env_vars

这是理解外部命令转换与 redirect_env 区别的核心：

| 函数 | 数据来源 | 使用场景 | 包含 engine_state |
|------|---------|---------|------------------|
| `get_env_vars()` | `engine_state.env_vars` + `stack.env_vars` | `env_to_strings` 外部命令 | ✅ 是 |
| `get_stack_env_vars()` | 仅 `stack.env_vars` | `redirect_env` 作用域导入 | ❌ 否 |

**为什么有这个区别？**
- **外部命令**需要完整的环境快照，包括永久变量
- **redirect_env**只需要同步运行时变更，永久变量已经在 caller 的 engine_state 中

### 子进程环境的独立性

子进程的环境是**一次性副本**：
1. 通过 `std::process::Command.envs()` 传递给子进程
2. 子进程对环境的修改完全独立，不会影响 Nushell
3. 子进程退出后，环境变量随之消失
4. Nushell 不需要也无法回滚子进程的环境变更

## 环境重定向（redirect_env）

> 注：这部分内容已在「机制二」中详细分析，此处保留原有的触发条件和使用场景概述。

### 触发条件

当 block 设置了 `redirect_env: true` 时，调用后会自动执行环境重定向。

实现于 [eval_call](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/eval.rs#L352-L356)：

```rust
let result = call_eval.run(engine_state, block, input);

if block.redirect_env {
    call_eval.redirect_env(engine_state, caller_stack);
}
```

### 使用 redirect_env 的场景

1. **`source-env`** — 将脚本文件中的环境变更导入当前作用域
   - 实现：[source_env.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-command/src/env/source_env.rs)
   - 显式调用 `redirect_env(engine_state, caller_stack, &callee_stack)`

2. **`export-env`** — 将块内的环境变更导出到当前作用域
   - 实现：[export_env.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-command/src/env/export_env.rs)
   - 同样显式调用 `redirect_env`

3. **自定义命令（def）** — 当命令块标记了 `redirect_env` 时
   - 在 IR 求值中处理：[eval_ir.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/eval_ir.rs#L1269-L1272)

## 三种机制详细对比

| 对比维度 | 临时作用域回滚<br>（with-env） | 当前作用域导入<br>（source-env/export-env） | 外部命令转换<br>（run-external） |
|---------|-------------------------------|--------------------------------------------|--------------------------------|
| **作用域创建** | 推入新环境层 `env_vars.push()` | 推入新环境层 `env_vars.push()` | 不创建，使用当前栈 |
| **新栈创建** | `captures_to_stack_preserve_out_dest` | `gather_captures` | 不创建 |
| **数据流向** | 外部 → 内部（单向） | 外部 → 内部 → 外部（双向） | Nushell → 子进程（单向） |
| **读取函数** | `get_env_var()` 逆序查找 | `get_env_var()` 逆序查找 | `get_env_vars()` 完整快照 |
| **同步函数** | 无（自动回滚） | `redirect_env` + `get_stack_env_vars()` | `env_to_strings` + `get_env_vars()` |
| **回滚方式** | 新栈丢弃，新层自动释放 | 不回滚，主动同步变更 | 不回滚，子进程环境独立 |
| **环境完整性** | 临时变量 + 继承的永久变量 | 同步运行时变更 | 完整环境快照（含永久变量） |
| **数据类型** | Nushell Value（任意类型） | Nushell Value（任意类型） | 转换为 String |
| **对 caller 的影响** | 执行完无影响 | 执行完环境被修改 | 执行完无影响 |
| **典型代码** | `with-env {X: Y} { ... }` | `source-env script.nu` | `echo $env.X` |

### 关键源码索引对照表

| 操作 | with-env | source-env | run-external |
|------|----------|------------|--------------|
| 创建新栈 | [captures_to_stack_preserve_out_dest](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L336-L357) | [gather_captures](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L359-L393) | 不创建 |
| 推入新层 | L338-339 | L374-375 | 不推入 |
| 变量读取 | [get_env_var](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L531-L557) | [get_env_var](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L531-L557) | [get_env_vars](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L396-L421) |
| 同步/转换 | 无 | [redirect_env](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/eval_ir.rs#L1885-L1905) | [env_to_strings](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/env.rs#L175-L192) |
| 数据范围 | 栈层 + engine_state | 仅栈层（get_stack_env_vars） | 栈层 + engine_state |

## 环境变量转换（ENV_CONVERSIONS）

Nushell 允许环境变量不仅是字符串，还可以是列表、记录等结构化值。`ENV_CONVERSIONS` 特殊环境变量定义了字符串与结构化值之间的转换规则。

### 转换机制

定义于 [env.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/env.rs)

`ENV_CONVERSIONS` 是一个 record，每个字段对应一个环境变量的转换配置：

```nu
$env.ENV_CONVERSIONS = {
    PATH: {
        from_string: {|s| $s | split row (char esep) }
        to_string: {|v| $v | str join (char esep) }
    }
}
```

### 字符串 → 值（from_string）

- **入口**：[`convert_env_vars`](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/env.rs#L33-L64)
  - 用于在运行时转换单个变量
- **入口**：[`convert_env_values`](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/env.rs#L72-L124)
  - 用于启动时批量转换所有环境变量
  - 转换结果写回 `engine_state.env_vars`

核心转换函数 [`get_converted_value`](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/env.rs#L301-L325)：

```
流程：
  1. 从 stack 中读取 ENV_CONVERSIONS
  2. 根据变量名找到对应的转换配置
  3. 找到 from_string 或 to_string 闭包
  4. 执行闭包完成转换
```

### 值 → 字符串（to_string）

- **入口**：[`env_to_string`](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/env.rs#L129-L172)
  - 单个变量转换为字符串
- **入口**：[`env_to_strings`](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/env.rs#L175-L192)
  - 批量转换（用于外部命令执行前）

特殊处理：
- `PATH`/`Path` 变量有硬编码的 fallback 逻辑
- 如果转换失败且是 PATH 变量，会尝试直接用列表拼接

### 触发时机

1. **启动时**：在 `main()` 中调用 `convert_env_values`，将从系统继承的字符串环境变量转换为 Nushell 值
2. **运行时赋值**：当给 `$env.ENV_CONVERSIONS` 赋值时，立即触发对当前已有变量的转换
   - 见 [eval_ir.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/eval_ir.rs#L525-L527)
3. **外部命令调用前**：将环境变量转换回字符串传递给子进程

## 隐藏机制（hide-env）

### hide_env_var 方法

实现于 [Stack::hide_env_var](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L684-L707)

```
隐藏流程：
  1. 检查是否已在 hide_history 中标记过（避免重复隐藏）
  2. 先从栈层中移除该变量
  3. 记录到 hide_history
  4. 如果栈中已没有该变量的影子，隐藏 engine_state 中的版本
     └─ 在 env_hidden 中对应 overlay 添加标记
```

### 隐藏的层级

- **`env_hidden`**：控制运行时对 engine_state 值的可见性
- **`env_hide_history`**：记录隐藏历史，用于 `hide-env` 命令的语义（重复隐藏返回错误）

两者的区别见注释：[stack.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L45-L49)

## REPL 环境合并

在 REPL 模式下，每轮命令执行后，栈中的环境变更会被**永久化**到 `EngineState`。

### merge_env 方法

实现于 [EngineState::merge_env](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/engine_state.rs#L364-L392)

```rust
pub fn merge_env(&mut self, stack: &mut Stack) -> Result<(), ShellError> {
    // 将 stack.env_vars 中所有层的变量合并到 engine_state.env_vars
    for mut scope in stack.env_vars.drain(..) {
        for (overlay_name, mut env) in Arc::make_mut(&mut scope).drain() {
            if let Some(env_vars) = Arc::make_mut(&mut self.env_vars).get_mut(&overlay_name) {
                env_vars.extend(env.drain());
            } else {
                Arc::make_mut(&mut self.env_vars).insert(overlay_name, env);
            }
        }
    }
    // 同步 PWD 到系统当前目录
    // 同步 config
}
```

### REPL 中的调用时机

1. **启动初始化后**：[repl.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-cli/src/repl.rs#L118)
2. **每轮命令执行前**：[repl.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-cli/src/repl.rs#L532)
   - 将上一轮的环境变更合并到永久状态

## Overlay 机制

Overlay 是环境变量和命令的命名空间，可以激活/停用。

### 活跃 Overlay 列表

`Stack.active_overlays` 维护当前活跃的 overlay 名称列表。越靠后的 overlay 优先级越高。

### 与环境变量的关系

每个环境层（`EnvVars`）内部按 overlay 分组存储。读取时按 `active_overlays` 的逆序遍历，确保后激活的 overlay 优先级更高。

### Overlay 操作

- **激活**：[`Stack::add_overlay`](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L734-L737)
- **移除**：[`Stack::remove_overlay`](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L739-L741)

## 完整流程图

```
┌─────────────────────────────────────────────────────────┐
│                   EngineState.env_vars                  │  ← 永久层（REPL 持久化）
│  { overlay1: { VAR: val }, overlay2: { VAR: val2 } }   │
└─────────────────────────────────────────────────────────┘
                            ▲
                            │ merge_env（REPL 每轮结束后）
                            │
┌─────────────────────────────────────────────────────────┐
│                    Stack.env_vars                       │  ← 栈层（运行时）
│  ┌─────────────────────────────────────────────────┐    │
│  │  Layer 0 （初始层，共享自父栈）                    │    │
│  │  { overlay1: {...}, overlay2: {...} }           │    │
│  └─────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Layer 1 （with-env / 闭包推入）                  │    │
│  │  { overlay2: { TEMP: val } }                    │    │  ← 优先级更高
│  └─────────────────────────────────────────────────┘    │
│                     ... 更多层 ...                      │
└─────────────────────────────────────────────────────────┘
                            ▲
                            │ redirect_env（source-env / export-env）
                            │
                   ┌──────────────────┐
                   │  内部作用域栈     │
                   │ （执行完即丢弃）    │
                   └──────────────────┘
```

## 关键代码索引

### 核心数据结构

| 功能 | 文件 | 行号 |
|------|------|------|
| Stack 结构体定义 | [stack.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L38-L67) | L38-L67 |
| EnvVars 类型定义 | [stack.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L18) | L18 |
| EngineState.env_vars | [engine_state.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/engine_state.rs#L104) | L104 |

### 环境变量读写

| 功能 | 文件 | 行号 |
|------|------|------|
| 读取单个环境变量 | [Stack::get_env_var](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L531-L557) | L531-L557 |
| 获取完整环境（栈层+永久层） | [Stack::get_env_vars](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L396-L421) | L396-L421 |
| 仅获取栈层环境 | [Stack::get_stack_env_vars](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L423-L440) | L423-L440 |
| 添加环境变量 | [Stack::add_env_var](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L273-L296) | L273-L296 |
| 隐藏环境变量 | [Stack::hide_env_var](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L684-L707) | L684-L707 |

### 作用域创建

| 功能 | 文件 | 行号 |
|------|------|------|
| 闭包栈（推入新环境层） | [Stack::captures_to_stack_preserve_out_dest](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L336-L357) | L336-L357 |
| 捕获栈（source-env/export-env 用） | [Stack::gather_captures](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L359-L393) | L359-L393 |
| 父子栈模式 | [Stack::with_parent](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L106-L124) | L106-L124 |
| 子栈合并回父 | [Stack::with_changes_from_child](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs#L132-L150) | L132-L150 |

### 三种机制的核心实现

| 机制 | 功能 | 文件 | 行号 |
|------|------|------|------|
| **临时作用域回滚** | with-env 命令主逻辑 | [with_env.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-command/src/env/with_env.rs#L54-L80) | L54-L80 |
| **当前作用域导入** | redirect_env（IR 版本） | [eval_ir.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/eval_ir.rs#L1885-L1905) | L1885-L1905 |
| **当前作用域导入** | redirect_env（AST 版本） | [eval.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/eval.rs#L368-L388) | L368-L388 |
| **当前作用域导入** | source-env 命令 | [source_env.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-command/src/env/source_env.rs#L45-L109) | L45-L109 |
| **当前作用域导入** | export-env 命令 | [export_env.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-command/src/env/export_env.rs#L40-L67) | L40-L67 |
| **外部命令转换** | env_to_strings | [env.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/env.rs#L175-L192) | L175-L192 |
| **外部命令转换** | env_to_string | [env.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/env.rs#L129-L172) | L129-L172 |
| **外部命令转换** | run-external 命令 | [run_external.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-command/src/system/run_external.rs#L177-L180) | L177-L180 |

### ENV_CONVERSIONS 转换

| 功能 | 文件 | 行号 |
|------|------|------|
| 字符串 → 值（批量） | [convert_env_values](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/env.rs#L72-L124) | L72-L124 |
| 字符串 → 值（运行时） | [convert_env_vars](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/env.rs#L33-L64) | L33-L64 |
| 核心转换函数 | [get_converted_value](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/env.rs#L301-L325) | L301-L325 |

### REPL 环境管理

| 功能 | 文件 | 行号 |
|------|------|------|
| REPL 合并环境到永久层 | [EngineState::merge_env](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/engine_state.rs#L364-L392) | L364-L392 |
| REPL 每轮合并调用点 | [repl.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-cli/src/repl.rs#L529-L534) | L529-L534 |

### 测试用例

| 功能 | 文件 | 行号 |
|------|------|------|
| with-env 自动回滚测试 | [with_env.rs 测试](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-command/tests/commands/with_env.rs#L50-L60) | L50-L60 |
