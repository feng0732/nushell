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

实现于 [with_env.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-command/src/env/with_env.rs)

```
执行流程：
  1. 通过 captures_to_stack_preserve_out_dest 创建新栈（推入新环境层）
  2. 在新栈上设置临时环境变量
  3. 执行闭包
  4. 闭包执行完成 → 新栈丢弃 → 环境自动回滚
```

示例测试见 [with_env.rs 测试](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-command/tests/commands/with_env.rs)

## 环境重定向（redirect_env）

有些场景需要将内部作用域的环境变更**持久化**到外部作用域，而不是自动回滚。这通过 `redirect_env` 机制实现。

### redirect_env 函数

实现于 [eval.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/eval.rs#L368-L388)

```rust
pub fn redirect_env(engine_state: &EngineState, caller_stack: &mut Stack, callee_stack: &Stack) {
    let caller_env_vars = caller_stack.get_env_var_names(engine_state);

    // 1. 移除 caller 有但 callee 没有的变量（callee 隐藏了它们）
    for var in caller_env_vars.iter() {
        if !callee_stack.has_env_var(engine_state, var) {
            caller_stack.hide_env_var(engine_state, var);
        }
    }

    // 2. 将 callee 栈层的变量添加到 caller
    for (var, value) in callee_stack.get_stack_env_vars() {
        caller_stack.add_env_var(var, value);
    }

    // 3. 同步 config
    caller_stack.config.clone_from(&callee_stack.config);
}
```

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

| 功能 | 文件 | 位置 |
|------|------|------|
| Stack 定义 | [stack.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs) | L38-L67 |
| 环境变量读取 | [stack.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs) | L531-L557 |
| 添加环境变量 | [stack.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs) | L273-L296 |
| 隐藏环境变量 | [stack.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs) | L684-L707 |
| 创建闭包栈 | [stack.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs) | L337-L357 |
| 父子栈创建 | [stack.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs) | L106-L124 |
| 子栈合并回父 | [stack.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-protocol/src/engine/stack.rs) | L132-L150 |
| 环境重定向 | [eval.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/eval.rs) | L368-L388 |
| with-env 命令 | [with_env.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-command/src/env/with_env.rs) | L54-L80 |
| source-env 命令 | [source_env.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-command/src/env/source_env.rs) | L45-L109 |
| export-env 命令 | [export_env.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-command/src/env/export_env.rs) | L40-L67 |
| 环境转换 | [env.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-engine/src/env.rs) | L33-L124 |
| REPL 环境合并 | [repl.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/68-nushell/crates/nu-cli/src/repl.rs) | L529-L534 |
