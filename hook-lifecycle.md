# Nushell 钩子生命周期深度解析

> 本文档从源码层面剖析 Nushell 钩子系统的注册、触发时机、求值环境、异常传播和性能影响。

---

## 一、钩子的种类与注册

### 1.1 五种钩子类型

钩子配置定义于 [hooks.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-protocol/src/config/hooks.rs#L7-L13) 的 `Hooks` 结构体：

| 钩子名 | 类型 | 触发时机 |
|--------|------|----------|
| `pre_prompt` | `Vec<Value>` | 显示命令提示符之前 |
| `pre_execution` | `Vec<Value>` | 命令执行之前 |
| `env_change` | `HashMap<EnvName, Vec<Value>>` | 监控的环境变量发生变化时 |
| `display_output` | `Option<Value>` | 命令输出渲染之前 |
| `command_not_found` | `Option<Value>` | 用户输入了不存在的命令时 |

配置通过 `$env.config.hooks` 进行注册，`UpdateFromValue` trait 负责将用户配置反序列化。

### 1.2 单钩子的四种形式

每个钩子条目（`Value`）可以是以下四种形式之一，定义于 [hook.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-cmd-base/src/hook.rs#L71-L277) 的 `eval_hook` 函数匹配分支：

1. **String（字符串）**：每次触发时重新解析源码并执行
2. **Closure（闭包）**：配置时已编译为字节码，直接执行
3. **List（列表）**：递归调用 `eval_hooks` 依次执行列表中的每个钩子
4. **Record（记录）**：条件钩子，包含 `condition`（闭包，返回 bool）和 `code`（字符串或闭包）两个字段

---

## 二、交互流程中的触发时序

### 2.1 REPL 单轮循环的完整触发链

代码位置：[repl.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-cli/src/repl.rs)

以下是一轮 REPL 循环（从显示提示符到命令执行完毕）中钩子的精确触发顺序：

```
loop_iteration() 开始
    │
    ├─ [1] engine_state.merge_env(&mut stack)
    │      └─ 合并上一轮的环境变更到引擎永久状态
    │
    ├─ [2] eval_env_change_hook(env_change)
    │      ├─ 遍历所有监控的环境变量名
    │      ├─ 对比 previous_env_vars[env] vs stack.get_env_var(env)
    │      ├─ 若不同 → 注入 $before / $after 变量 → 执行对应钩子列表
    │      └─ 更新 previous_env_vars[env] = 新值
    │
    ├─ [3] eval_hooks(pre_prompt)
    │      └─ 在读取用户输入前执行
    │
    ├─ reedline.read_line() 等待用户输入
    │
    └─ 用户输入回车后 → run_command()
            │
            ├─ [4] eval_hooks(pre_execution)
            │      └─ 设置 repl_state.buffer = 当前命令后执行
            │
            ├─ 实际命令执行（eval_source）
            │      │
            │      ├─ 若命令产生可打印输出
            │      │    └─ [5] print_pipeline() → eval_hook(display_output)
            │      │         └─ 将 PipelineData 作为输入传给钩子
            │      │
            │      └─ 若命令名未找到
            │           └─ [6] command_not_found() → eval_hook(command_not_found)
            │                └─ 传入 $cmd_name 参数，若返回 String 则作为帮助文本
            │
            └─ 设置 CMD_DURATION_MS 等环境变量
```

### 2.2 各触发点的精确代码位置

| 序号 | 钩子 | 代码位置 |
|------|------|----------|
| [1] | `merge_env`（前置准备） | [repl.rs#L532](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-cli/src/repl.rs#L532-L534) |
| [2] | `env_change` | [repl.rs#L551](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-cli/src/repl.rs#L551-L557) |
| [3] | `pre_prompt` | [repl.rs#L562](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-cli/src/repl.rs#L562-L571) |
| [4] | `pre_execution` | [repl.rs#L374](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-cli/src/repl.rs#L374-L384) |
| [5] | `display_output` | [util.rs#L223](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-cli/src/util.rs#L223-L232) |
| [6] | `command_not_found` | [run_external.rs#L532](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-command/src/system/run_external.rs#L532-L576) |

### 2.3 特殊参数注入

不同钩子在执行时会注入不同的特殊变量：

| 钩子 | 注入变量 | 说明 |
|------|----------|------|
| `env_change` | `$before`, `$after` | 变量变化前后的值 |
| `command_not_found` | `$cmd_name` | 用户输入的未识别命令名 |
| `display_output` | PipelineData 作为输入 | 待渲染的输出流（通过管道传入） |
| 其他 | 无 | 通过 `commandline` 内置命令可获取 REPL 缓冲区 |

---

## 三、求值环境（Evaluation Context）深度分析

### 3.1 字符串钩子的求值流程

代码位置：[hook.rs#L72-L133](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-cmd-base/src/hook.rs#L72-L133)

```
每次触发 String 钩子：
    │
    ├─ 创建全新的 StateWorkingSet（独立的编译上下文）
    │
    ├─ 为特殊参数创建临时 VarId 并注入变量
    │   （如 $before / $after 对应 env_change 钩子）
    │
    ├─ parse() 解析字符串为 AST → Block
    │   └─ 解析错误：report_parse_error + 返回 GenericError
    │
    ├─ engine_state.merge_delta(delta)
    │   └─ 将新编译的块、变量定义永久合并到引擎状态
    │
    ├─ stack.add_var(var_id, val) 将临时变量加入栈
    │
    ├─ eval_block() 执行块
    │   └─ 运行时错误：report_shell_error（仅报告，不返回 Err）
    │
    ├─ stack.remove_var(var_id) 清理所有临时变量
    │
    └─ engine_state.merge_env(stack)
        └─ 将栈中的环境变更永久写回引擎状态
```

**关键特性：**
- **每次触发都会重新 parse 和 merge_delta**，这是最大的性能开销来源
- 临时变量在执行后被显式清理，不会污染外部作用域
- `merge_env` 确保 `$env.VAR = value` 这样的修改在钩子执行后可见

### 3.2 闭包钩子的求值流程

代码位置：[hook.rs#L284-L331](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-cmd-base/src/hook.rs#L284-L331)

```
每次触发 Closure 钩子：
    │
    ├─ stack.captures_to_stack_preserve_out_dest(captures)
    │   ├─ 创建新的子栈（callee_stack）
    │   ├─ 复制闭包创建时捕获的变量到子栈
    │   └─ 保留 stdout/stderr 重定向目标
    │
    ├─ callee_stack.reset_pipes() 清空管道状态
    │
    ├─ 按位置绑定参数：遍历 block.signature.required_positional
    │   └─ 将 arguments[idx] 添加到子栈对应 var_id
    │
    ├─ eval_block_with_early_return() 在子栈上执行闭包体
    │   └─ 运行时错误：? 直接向上传播 Err
    │
    ├─ 若返回 Value::Error { error } → 转换为 Err(*error)
    │
    └─ redirect_env(engine_state, caller_stack, &callee_stack)
        ├─ 步骤1：隐藏 caller 有但 callee 没有的 env var
        └─ 步骤2：将 callee 新增/修改的 env var 写回 caller
```

**关键特性：**
- **闭包在配置加载时已编译完成**，触发时无需重新 parse
- 使用独立子栈执行，变量隔离但环境通过 `redirect_env` 双向同步
- 支持 `return` 提前返回（使用 `eval_block_with_early_return`）

### 3.3 redirect_env 的环境同步语义

代码位置：[eval.rs#L369-L384](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-engine/src/eval.rs#L369-L384)

```rust
pub fn redirect_env(engine_state, caller_stack, callee_stack) {
    // 1. 删除 caller 有但 callee 没有的变量（callee 中执行了 hide-env）
    for var in caller_env_vars {
        if !callee_stack.has_env_var(var) {
            caller_stack.hide_env_var(var);
        }
    }
    // 2. 复制 callee 的所有栈级环境变量到 caller
    for (var, value) in callee_stack.get_stack_env_vars() {
        caller_stack.add_env_var(var, value);
    }
}
```

### 3.4 Record 条件钩子的特殊环境

代码位置：[hook.rs#L137-L266](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-cmd-base/src/hook.rs#L137-L266)

Record 形式的钩子先执行 `condition` 闭包判断是否需要执行：

1. **condition 阶段**：使用闭包方式执行（独立子栈 + redirect_env）
   - 必须返回 `Value::Bool`，否则返回 `RuntimeTypeMismatch`
   - 返回 `false` → 跳过 `code` 执行
   - 返回 `true` → 进入 `code` 执行阶段

2. **code 阶段**：根据 `code` 的类型（String 或 Closure）走对应分支

---

## 四、异常传播机制

### 4.1 异常传播的三层结构

钩子系统的异常处理分为三层，每一层的策略不同：

```
┌─────────────────────────────────────────────────────┐
│  第一层：REPL 顶层调用点                             │
│  pre_prompt / pre_execution / env_change            │
│  策略：捕获并报告错误，绝不中断 REPL 循环            │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│  第二层：eval_hooks / eval_hook 分发层               │
│  策略：类型相关的错误传播（最容易踩坑的地方）         │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│  第三层：实际执行层                                   │
│  parse / eval_block / run_hook 产生的原生错误        │
└─────────────────────────────────────────────────────┘
```

### 4.2 第一层：REPL 顶层的错误隔离

在 [repl.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-cli/src/repl.rs) 中，三个 REPL 钩子的调用方式完全一致：

```rust
// pre_prompt / pre_execution / env_change 都是这样调用的：
if let Err(err) = hook::eval_hooks(...) {
    report_shell_error(None, engine_state, &err);  // 打印错误
    // 没有 return，继续执行 REPL 循环
}
```

**结论：这三个钩子的任何错误都只会被打印，不会终止交互流程。**

但 `display_output` 和 `command_not_found` 不同：
- **display_output**：错误通过 `?` 传播到 `evaluate_source`，会中断当前命令的打印（但不影响 REPL）
- **command_not_found**：错误直接成为 `ShellError` 返回，替换掉原本的"命令未找到"错误

### 4.3 第二层：eval_hook 的陷阱——错误传播不一致

这是整个钩子系统**最容易让人困惑**的部分。请仔细看以下表格：

| 钩子形式 | 错误来源 | 是否返回 Err | 后续钩子是否继续 |
|----------|----------|:------------:|:----------------:|
| **String** | parse 阶段 | ✅ 是 | ❌ 停止 |
| **String** | eval_block 运行时 | ❌ 否（仅 report） | ✅ 继续 |
| **Closure** | 任何错误 | ✅ 是 | ❌ 停止 |
| **List** | 子钩子返回 Err | ✅ 是（递归传播） | ❌ 停止 |
| **Record - condition** | 非闭包/非Bool 返回 | ✅ 是 | ❌ 停止 |
| **Record - condition** | 运行时错误 | ✅ 是 | ❌ 停止 |
| **Record - code (String)** | 运行时错误 | ❌ 否（仅 report） | ✅ 继续 |
| **Record - code (Closure)** | 运行时错误 | ✅ 是 | ❌ 停止 |

**代码证据：**

String 钩子的运行时错误被吞掉（[hook.rs#L125-L128](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-cmd-base/src/hook.rs#L125-L128)）：
```rust
match eval_block(...).map(|p| p.body) {
    Ok(pipeline_data) => { output = pipeline_data; }
    Err(err) => {
        report_shell_error(Some(stack), engine_state, &err);
        // ⚠️ 注意：这里没有 return Err(err)
    }
}
```

Closure 钩子的错误直接传播（[hook.rs#L267-L269](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-cmd-base/src/hook.rs#L267-L269)）：
```rust
Value::Closure { val, .. } => {
    output = run_hook(engine_state, stack, val, input, arguments, span)?;
    // ⚠️ ? 运算符会立即返回错误
}
```

### 4.4 实际影响举例

假设你配置了这样的 `pre_prompt` 钩子列表：

```nu
$env.config.hooks.pre_prompt = [
    {|| error make {msg: "oops"} }     # 钩子1：Closure 形式抛错
    {|| print "I will never run" }     # 钩子2
]
```

→ **钩子2 不会执行**，因为 Closure 错误用 `?` 传播。

但如果写成 String 形式：

```nu
$env.config.hooks.pre_prompt = [
    'error make {msg: "oops"}'         # 钩子1：String 形式抛错
    'print "I WILL run!"'              # 钩子2
]
```

→ **钩子2 仍然会执行**，因为 String 的运行时错误只被报告不传播。

---

## 五、性能影响分析

### 5.1 String 钩子 vs Closure 钩子：数量级差异

| 操作 | String 钩子 | Closure 钩子 |
|------|:-----------:|:------------:|
| 创建 StateWorkingSet | 每次触发 ✅ | 配置时一次 |
| parse() 词法+语法分析 | 每次触发 ✅ | 配置时一次 |
| merge_delta() 合并引擎状态 | 每次触发 ✅ | 配置时一次 |
| 临时 VarId 创建/销毁 | 每次触发 ✅ | 不需要 |
| eval_block 字节码执行 | 每次触发 ✅ | 每次触发 ✅ |
| 子栈创建 + redirect_env | 不需要 | 每次触发 ✅ |

**结论：性能敏感场景下，始终使用 Closure 形式。String 钩子适合短小的一次性配置脚本。**

### 5.2 REPL 循环中的性能热点

在 [repl.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-cli/src/repl.rs) 中使用 `perf!` 宏标记了关键性能点（支持 `use_ansi_coloring` 时输出到 stderr）：

| 性能标记 | 执行频率 | 说明 |
|----------|:--------:|------|
| `merge env` | 每轮 1 次 | 合并 Stack 环境到 EngineState，涉及遍历所有 overlay 的 env vars + `std::env::set_current_dir()` 系统调用 |
| `env-change hook` | 每轮 1 次 | 对比所有监控变量，每次变化触发钩子链 |
| `pre-prompt hook` | 每轮 1 次 | 读取用户输入前执行 |
| `pre_execution_hook` | 每条命令 1 次 | 命令执行前执行 |
| `update_prompt` | 每轮 1 次 | 渲染提示符（可能包含复杂闭包） |

### 5.3 EngineState 克隆的隐性开销

[repl.rs#L193-L196](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-cli/src/repl.rs#L193-L196)：

```rust
let mut current_engine_state = previous_engine_state.clone();
let current_stack = Stack::with_parent(previous_stack_arc.clone());
```

每轮 REPL 循环都会 **完整 clone EngineState**，其中包含：
- 所有命令声明（decls）
- 所有模块定义
- 所有变量定义
- 所有 overlay 链
- 完整的 env_vars（多层 HashMap + Arc）

虽然很多结构使用了 `Arc` 共享，但仍然有大量的引用计数操作和浅拷贝。

**String 钩子会进一步放大这个问题**：每次执行 `merge_delta` 会增加 EngineState 的大小，下一轮 clone 时开销更大。

### 5.4 env_change 钩子的 O(N) 扫描

[hook.rs#L18-L38](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-cmd-base/src/hook.rs#L18-L38)：

```rust
pub fn eval_env_change_hook(env_change_hook, engine_state, stack) {
    for (env, hooks) in env_change_hook {  // 遍历所有监控的变量
        let before = engine_state.previous_env_vars.get(env);
        let after = stack.get_env_var(engine_state, env.as_str());
        if before != after {  // 值比较（可能涉及深比较大 Value）
            // ...执行钩子...
            Arc::make_mut(&mut engine_state.previous_env_vars)
                .insert(env.clone(), after);  // 更新缓存
        }
    }
}
```

**性能建议：** 不要监控 `PWD` 这种**每轮都会变**的变量，也不要监控包含大 Value 的环境变量（如完整 list/record）。

### 5.5 display_output 钩子的特殊影响

`display_output` 钩子接收整条 PipelineData 作为输入，需要特别注意：

1. **流式数据会被强制收集**：如果钩子内部需要查看完整数据（如 `table` 命令），会将流式输出全部加载到内存
2. **输出格式钩子每次打印都会执行**：包括 `print` 命令、REPL 结果渲染等
3. **错误会静默抑制输出**：如果钩子返回 Err，`print_pipeline` 提前返回，输出不会打印

---

## 六、实用建议与最佳实践

### 6.1 优先使用 Closure 而非 String

```nu
# ❌ 慢：每次触发都 parse
$env.config.hooks.pre_prompt = ['print "hello"']

# ✅ 快：配置时编译一次
$env.config.hooks.pre_prompt = [{|| print "hello" }]
```

### 6.2 列表中错误传播的注意事项

如果你的钩子列表包含多个步骤，且希望一个失败不影响其他步骤：
- 全部用 String 形式（隐式容错，不推荐，行为不透明）
- 或者在 Closure 中显式用 `try` 包裹：

```nu
$env.config.hooks.pre_prompt = [
    {|| try { some_risky_operation() } }  # 显式容错
    {|| always_run_this() }
]
```

### 6.3 env_change 钩子的条件优化

对频繁变化的变量，使用 condition 过滤不必要的执行：

```nu
$env.config.hooks.env_change.PWD = [
    {
        # 只有进入特定目录才触发实际逻辑
        condition: {|before, after| ($after | path basename) == "my-project" }
        code: {|| source-env .env }
    }
]
```

### 6.4 command_not_found 钩子的防循环保护

代码内置了金丝雀变量 `ENTERED_COMMAND_NOT_FOUND` 防止无限递归（[run_external.rs#L536-L547](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-command/src/system/run_external.rs#L536-L547)），但你仍应避免在钩子内调用可能失败的外部命令。

### 6.5 钩子修改命令行的技巧

`pre_prompt` 和 `env_change` 钩子可以修改 `engine_state.repl_state.buffer`，随后 `flush_engine_state_repl_buffer` 会将内容刷入 reedline 编辑器。这是实现"钩子自动填充命令行"功能的基础。

---

## 七、核心代码参考索引

| 功能模块 | 文件路径 | 关键行 |
|----------|----------|--------|
| Hooks 配置结构 | [hooks.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-protocol/src/config/hooks.rs) | L7-L53 |
| 钩子分发执行入口 | [hook.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-cmd-base/src/hook.rs) | L40-L282 |
| 闭包钩子实际执行 | [hook.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-cmd-base/src/hook.rs) | L284-L331 |
| REPL 主循环触发点 | [repl.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-cli/src/repl.rs) | L510-L848 |
| pre_execution 触发 | [repl.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-cli/src/repl.rs) | L339-L507 |
| display_output 触发 | [util.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-cli/src/util.rs) | L208-L237 |
| command_not_found 触发 | [run_external.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-command/src/system/run_external.rs) | L523-L639 |
| 环境重定向逻辑 | [eval.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-engine/src/eval.rs) | L369-L384 |
| EngineState 合并环境 | [engine_state.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/crates/nu-protocol/src/engine/engine_state.rs) | L365-L392 |
| 集成测试用例 | [mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/69-nushell/tests/hooks/mod.rs) | L1-L584 |
