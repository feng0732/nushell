# Nushell 钩子生命周期深度解析

> 本文档从源码层面剖析 Nushell 钩子系统的注册、触发时机、求值环境、异常传播和性能影响。

---

## 一、钩子的种类与注册

### 1.1 五种钩子类型

钩子配置定义于 `crates/nu-protocol/src/config/hooks.rs` 的 `Hooks` 结构体（L7-L13）：

| 钩子名 | 类型 | 触发时机 |
|--------|------|----------|
| `pre_prompt` | `Vec<Value>` | 显示命令提示符之前 |
| `pre_execution` | `Vec<Value>` | 命令执行之前 |
| `env_change` | `HashMap<EnvName, Vec<Value>>` | 监控的环境变量发生变化时 |
| `display_output` | `Option<Value>` | REPL/脚本中命令求值结果渲染之前 |
| `command_not_found` | `Option<Value>` | 用户输入了不存在的命令时 |

配置通过 `$env.config.hooks` 进行注册，`UpdateFromValue` trait 负责将用户配置反序列化。

### 1.2 单钩子的四种形式

每个钩子条目（`Value`）可以是以下四种形式之一，定义于 `crates/nu-cmd-base/src/hook.rs` 的 `eval_hook` 函数匹配分支（L71-L277）：

1. **String（字符串）**：每次触发时重新解析源码并执行
2. **Closure（闭包）**：配置时已编译为字节码，直接执行
3. **List（列表）**：递归调用 `eval_hooks` 依次执行列表中的每个钩子
4. **Record（记录）**：条件钩子，包含 `condition`（闭包，返回 bool）和 `code`（字符串或闭包）两个字段

---

## 二、交互流程中的触发时序

### 2.1 REPL 单轮循环的完整触发链

代码位置：`crates/nu-cli/src/repl.rs`

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
            │      ├─ 命令求值完成后，若有可打印结果
            │      │    └─ [5] print_pipeline() → eval_hook(display_output)
            │      │         └─ 将 PipelineData 作为管道输入传给钩子
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
| [1] | `merge_env`（前置准备） | `crates/nu-cli/src/repl.rs` L532-L534 |
| [2] | `env_change` | `crates/nu-cli/src/repl.rs` L551-L557 |
| [3] | `pre_prompt` | `crates/nu-cli/src/repl.rs` L562-L571 |
| [4] | `pre_execution` | `crates/nu-cli/src/repl.rs` L374-L384 |
| [5] | `display_output` | `crates/nu-cli/src/util.rs` L223-L232 |
| [6] | `command_not_found` | `crates/nu-command/src/system/run_external.rs` L532-L576 |

### 2.3 特殊参数注入

不同钩子在执行时会注入不同的特殊变量：

| 钩子 | 注入变量 | 说明 |
|------|----------|------|
| `env_change` | `$before`, `$after` | 变量变化前后的值 |
| `command_not_found` | `$cmd_name` | 用户输入的未识别命令名 |
| `display_output` | PipelineData 作为管道输入 | 待渲染的输出流 |
| 其他 | 无 | 通过 `commandline` 内置命令可获取 REPL 缓冲区 |

---

## 三、display_output 与 print 输出的界限

### 3.1 两条独立的输出通路

Nushell 中存在两条**完全独立**的输出通路，只有其中一条经过 `display_output` 钩子：

```
                        ┌──────────────────────────────┐
                        │     命令求值完成              │
                        └──────────┬───────────────────┘
                                   │
               ┌───────────────────┼───────────────────┐
               │                   │                   │
    ┌──────────▼──────────┐  ┌─────▼──────┐  ┌────────▼─────────┐
    │  REPL/脚本求值结果   │  │  print 命令 │  │  drain_to_out    │
    │  (eval_source 返回) │  │  (Print)    │  │  _dests 管道末端 │
    └──────────┬──────────┘  └─────┬──────┘  └────────┬─────────┘
               │                   │                   │
               ▼                   ▼                   ▼
        print_pipeline()    print_table()        print_table()
               │             print_raw()          print_raw()
               ▼                   │                   │
      ┌────────────────┐          │                   │
      │ display_output │          │                   │
      │   钩子生效？    │          │                   │
      └───────┬────────┘          │                   │
         是 ↓      ↓ 否           │                   │
      eval_hook   print_table()   │                   │
               ↓     ↓            ↓                   ↓
           print_raw()        直接写入 stdout/stderr
```

### 3.2 关键区分：谁经过 display_output，谁不经过

**经过 `display_output` 的路径**——`print_pipeline()` 函数：

调用点仅有三处：
- `crates/nu-cli/src/util.rs` L335 —— `evaluate_source()` 中，命令求值结果的最终打印
- `crates/nu-cli/src/eval_file.rs` L188 —— `nu script.nu` 脚本执行结果打印
- `crates/nu-cli/src/eval_cmds.rs` L97 —— `nu -c "command"` 单命令模式结果打印

`print_pipeline` 的实现（`crates/nu-cli/src/util.rs` L215-L237）：
```rust
pub fn print_pipeline(engine_state, stack, pipeline, no_newline) {
    if let Some(hook) = stack.get_config(engine_state).hooks.display_output.clone() {
        let pipeline = eval_hook(
            engine_state, stack, Some(pipeline), vec![], &hook, "display_output"
        )?;
        pipeline.print_raw(engine_state, no_newline, to_stderr)  // 钩子输出用 print_raw
    } else {
        pipeline.print_table(engine_state, stack, no_newline, to_stderr)  // 无钩子用 print_table
    }
}
```

**不经过 `display_output` 的路径**——直接调用 `print_table` / `print_raw`：

1. **`print` 命令**（`crates/nu-cli/src/commands/print.rs` L50-L97）：
   ```rust
   // print 命令的 run() 方法直接调用 print_table 或 print_raw
   arg.into_pipeline_data().print_table(engine_state, stack, no_newline, to_stderr)?;
   // 或
   input.print_raw(engine_state, no_newline, to_stderr)?;
   ```
   `print` 命令是**主动、即时的输出**，它不走 `display_output` 钩子。

2. **`drain_to_out_dests()`**（`crates/nu-protocol/src/pipeline/pipeline_data.rs` L302-L327）：
   ```rust
   OutDest::Print => {
       self.print_table(engine_state, stack, false, false)?;  // 不走 display_output
       Ok(Self::Empty)
   }
   ```
   当管道末端的 stdout 重定向目标为 `OutDest::Print` 时，也直接走 `print_table`。

3. **`watch` 命令**（`crates/nu-command/src/filesystem/watch.rs` L348）：
   ```rust
   Ok(val) => val.print_table(engine_state, stack, false, false)?,
   ```

### 3.3 总结：display_output 的精确边界

| 输出场景 | 是否经过 display_output | 实际调用 |
|----------|:-----------------------:|----------|
| REPL 中输入 `1 + 2`，求值结果 `3` | ✅ 是 | `eval_source` → `print_pipeline` |
| `nu -c "1 + 2"` | ✅ 是 | `eval_cmds` → `print_pipeline` |
| `nu script.nu` 最后一行求值结果 | ✅ 是 | `eval_file` → `print_pipeline` |
| `print "hello"` | ❌ 否 | `Print::run()` → `print_table` |
| 管道末端的隐式打印 | ❌ 否 | `drain_to_out_dests` → `print_table` |
| `watch` 命令的输出 | ❌ 否 | 直接 `print_table` |

**核心结论**：`display_output` 只作用于"命令行求值的最终返回值"，不影响 `print` 命令和其他命令内部的主动输出。这是有意设计的——`print_table` 的文档注释明确写道（`crates/nu-protocol/src/pipeline/pipeline_data.rs` L720-L721）：

> This does not respect the display_output hook. If a value is being printed out by a command, this function should be used. Otherwise, `nu_cli::util::print_pipeline` should be preferred.

### 3.4 display_output 钩子自身的输出方式

当 `display_output` 钩子生效时，钩子的**输出**使用 `print_raw`（无格式化），而不是 `print_table`。这意味着：

- 钩子返回的结果**不会再次经过 display_output**（避免无限递归）
- 钩子的输出以原始文本形式打印，不做表格格式化

---

## 四、env_change 钩子的 PWD 触发条件

### 4.1 previous_env_vars 的工作机制

`crates/nu-cmd-base/src/hook.rs` L18-L37：

```rust
pub fn eval_env_change_hook(env_change_hook, engine_state, stack) {
    for (env, hooks) in env_change_hook {
        let before = engine_state.previous_env_vars.get(env);   // 取缓存的上一次值
        let after = stack.get_env_var(engine_state, env.as_str());  // 取当前栈上的值
        if before != after {                                      // Option<&Value> vs Option<&Value>
            let before = before.cloned().unwrap_or_default();    // None → Value::Nothing
            let after = after.cloned().unwrap_or_default();
            eval_hooks(engine_state, stack,
                vec![("$before".into(), before), ("$after".into(), after.clone())],
                hooks, "env_change",
            )?;
            Arc::make_mut(&mut engine_state.previous_env_vars)
                .insert(env.clone(), after);  // 触发后更新缓存
        }
    }
}
```

### 4.2 previous_env_vars 的初始化

`crates/nu-protocol/src/engine/engine_state.rs` L199：

```rust
previous_env_vars: Arc::new(HashMap::new()),  // 启动时为空
```

`previous_env_vars` 在 `EngineState::new()` 时初始化为**空 HashMap**。它不是从实际环境变量快照初始化的——里面一条记录都没有。

### 4.3 首次触发的必然性

由于 `previous_env_vars` 初始为空，当用户配置了 `env_change.PWD` 钩子后：

- **第一轮 REPL 循环**：`before = previous_env_vars.get("PWD")` → `None`，`after = stack.get_env_var("PWD")` → `Some("/home/user")`
- `None != Some(...)` → **必定触发**
- `$before` 被设为 `Value::Nothing`（`unwrap_or_default()`），`$after` 为 PWD 的实际值

这意味着：**只要配置了 `env_change.PWD` 钩子，REPL 启动后的第一次 `eval_env_change_hook` 调用就一定会触发该钩子**，无论 PWD 是否真的发生了变化。

### 4.4 后续触发的条件

触发一次后，`previous_env_vars["PWD"]` 被更新为当前值。之后：

| 场景 | before | after | 是否触发 |
|------|--------|-------|:--------:|
| 未执行 `cd`，PWD 不变 | `Some("/home/user")` | `Some("/home/user")` | ❌ |
| 执行 `cd /tmp` | `Some("/home/user")` | `Some("/tmp")` | ✅ |
| `cd` 后再次进入循环 | `Some("/tmp")` | `Some("/tmp")` | ❌ |
| 环境变量被 `unset` | `Some("/tmp")` | `None` | ✅，`$after` = Nothing |
| 环境变量重新出现 | `None` | `Some("/new")` | ✅，`$before` = Nothing |

**关键细节**：比较发生在 `merge_env` 之后。`merge_env`（`crates/nu-cli/src/repl.rs` L532）将 Stack 中的环境变更写回 EngineState，然后 `eval_env_change_hook` 才读取 Stack 上的当前值进行对比。所以钩子看到的 `after` 已经是合并后的最新值。

### 4.5 env_change 不只监控 PWD

`env_change` 是一个 `HashMap<EnvName, Vec<Value>>`，可以同时监控任意多个环境变量：

```nu
$env.config.hooks.env_change = {
    PWD: [{ code: {|| print "directory changed" } }]
    FOO: [{ code: {|| print "FOO changed" } }]
}
```

每个变量的 `before`/`after` 对比独立进行，互不影响。一轮循环中，可能有 0 个、1 个或多个 `env_change` 钩子被触发。

---

## 五、求值环境（Evaluation Context）深度分析

### 5.1 字符串钩子的求值流程

代码位置：`crates/nu-cmd-base/src/hook.rs` L72-L133

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
    └─ engine_state.merge_env(stack)        ← 注意：eval_hook 末尾必定执行
        └─ 将栈中的环境变更永久写回引擎状态
```

**关键特性：**
- **每次触发都会重新 parse 和 merge_delta**，这是最大的性能开销来源
- 临时变量在执行后被显式清理，不会污染外部作用域
- String 钩子直接在传入的 `&mut Stack` 上执行，不创建子栈
- `eval_hook` 末尾的 `merge_env`（L279）确保 `$env.VAR = value` 等修改在钩子执行后可见

### 5.2 闭包钩子的求值流程

代码位置：`crates/nu-cmd-base/src/hook.rs` L284-L331

```
每次触发 Closure 钩子（run_hook 函数）：
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
        ├─ 步骤1：隐藏 caller 有但 callee 没有的 env var（支持 hide-env）
        └─ 步骤2：将 callee 新增/修改的 env var 写回 caller
```

**关键特性：**
- **闭包在配置加载时已编译完成**，触发时无需重新 parse
- 使用独立子栈执行，变量隔离但环境通过 `redirect_env` 双向同步
- 支持 `return` 提前返回（使用 `eval_block_with_early_return`）
- `run_hook` 函数本身不做 `merge_env`，仅做 `redirect_env`——但 `run_hook` 的调用点在 `eval_hook` 内部，`eval_hook` 末尾仍会执行 `merge_env`

### 5.3 redirect_env vs merge_env 的区别

两个函数做的是**不同层级**的环境同步：

| 函数 | 代码位置 | 作用域 | 语义 |
|------|----------|--------|------|
| `redirect_env` | `crates/nu-engine/src/eval.rs` L369-L384 | Stack → Stack | 将 callee 子栈的环境变量复制回 caller 栈，支持 hide-env |
| `merge_env` | `crates/nu-protocol/src/engine/engine_state.rs` L365-L392 | Stack → EngineState | 将 Stack 中的所有环境变量、config 变更永久写入 EngineState + 切换系统 cwd |

```rust
// redirect_env：Stack 间同步
pub fn redirect_env(engine_state, caller_stack, callee_stack) {
    for var in caller_env_vars {
        if !callee_stack.has_env_var(var) {
            caller_stack.hide_env_var(var);  // callee 隐藏了某些变量
        }
    }
    for (var, value) in callee_stack.get_stack_env_vars() {
        caller_stack.add_env_var(var, value);  // callee 新增/修改了变量
    }
}

// merge_env：Stack → EngineState 永久写入
pub fn merge_env(&mut self, stack: &mut Stack) {
    for scope in stack.env_vars.drain(..) {
        // 遍历所有 overlay 层级的环境变量，写回 EngineState
    }
    std::env::set_current_dir(cwd)?;  // 同步系统工作目录
    if let Some(config) = stack.config.take() {
        self.config = config;  // 应用配置变更
    }
}
```

### 5.4 Record 条件钩子的特殊环境

代码位置：`crates/nu-cmd-base/src/hook.rs` L137-L266

Record 形式的钩子先执行 `condition` 闭包判断是否需要执行：

1. **condition 阶段**：使用闭包方式执行（独立子栈 + redirect_env）
   - 必须返回 `Value::Bool`，否则返回 `RuntimeTypeMismatch`
   - 返回 `false` → 跳过 `code` 执行
   - 返回 `true` → 进入 `code` 执行阶段

2. **code 阶段**：根据 `code` 的类型（String 或 Closure）走对应分支

---

## 六、异常传播机制

### 6.1 异常传播的三层结构

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

### 6.2 第一层：REPL 顶层的错误隔离

在 `crates/nu-cli/src/repl.rs` 中，三个 REPL 钩子的调用方式完全一致：

```rust
// pre_prompt / pre_execution / env_change 都是这样调用的：
if let Err(err) = hook::eval_hooks(...) {
    report_shell_error(None, engine_state, &err);  // 打印错误
    // 没有 return，继续执行 REPL 循环
}
```

**结论：这三个钩子的任何错误都只会被打印，不会终止交互流程。**

但 `display_output` 和 `command_not_found` 不同：
- **display_output**：错误通过 `?` 传播到 `evaluate_source`（`crates/nu-cli/src/util.rs` L335），会中断当前命令的打印（但不影响 REPL 主循环）
- **command_not_found**：错误直接成为 `ShellError` 返回，替换掉原本的"命令未找到"错误

### 6.3 第二层：eval_hook 的陷阱——错误传播不一致

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

String 钩子的运行时错误被吞掉（`crates/nu-cmd-base/src/hook.rs` L121-L128）：
```rust
match eval_block(...).map(|p| p.body) {
    Ok(pipeline_data) => { output = pipeline_data; }
    Err(err) => {
        report_shell_error(Some(stack), engine_state, &err);
        // ⚠️ 注意：这里没有 return Err(err)
    }
}
```

Closure 钩子的错误直接传播（`crates/nu-cmd-base/src/hook.rs` L267-L269）：
```rust
Value::Closure { val, .. } => {
    output = run_hook(engine_state, stack, val, input, arguments, span)?;
    // ⚠️ ? 运算符会立即返回错误
}
```

### 6.4 实际影响举例

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

## 七、性能影响分析

### 7.1 String 钩子 vs Closure 钩子：数量级差异

| 操作 | String 钩子 | Closure 钩子 |
|------|:-----------:|:------------:|
| 创建 StateWorkingSet | 每次触发 ✅ | 配置时一次 |
| parse() 词法+语法分析 | 每次触发 ✅ | 配置时一次 |
| merge_delta() 合并引擎状态 | 每次触发 ✅ | 配置时一次 |
| 临时 VarId 创建/销毁 | 每次触发 ✅ | 不需要 |
| eval_block 字节码执行 | 每次触发 ✅ | 每次触发 ✅ |
| merge_env（eval_hook 末尾） | 每次触发 ✅ | 每次触发 ✅ |
| 子栈创建 + redirect_env | 不需要 | 每次触发 ✅ |

**结论：性能敏感场景下，始终使用 Closure 形式。String 钩子适合短小的一次性配置脚本。**

### 7.2 REPL 循环中的性能热点

在 `crates/nu-cli/src/repl.rs` 中使用 `perf!` 宏标记了关键性能点（支持 `use_ansi_coloring` 时输出到 stderr）：

| 性能标记 | 执行频率 | 说明 |
|----------|:--------:|------|
| `merge env` | 每轮 1 次 | 合并 Stack 环境到 EngineState，涉及遍历所有 overlay 的 env vars + `std::env::set_current_dir()` 系统调用 |
| `env-change hook` | 每轮 1 次 | 对比所有监控变量，每次变化触发钩子链 |
| `pre-prompt hook` | 每轮 1 次 | 读取用户输入前执行 |
| `pre_execution_hook` | 每条命令 1 次 | 命令执行前执行 |
| `update_prompt` | 每轮 1 次 | 渲染提示符（可能包含复杂闭包） |

### 7.3 EngineState 克隆的隐性开销

`crates/nu-cli/src/repl.rs` L193-L196：

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
- `previous_env_vars`（Arc<HashMap>）

虽然很多结构使用了 `Arc` 共享，但仍然有大量的引用计数操作和浅拷贝。

**String 钩子会进一步放大这个问题**：每次执行 `merge_delta` 会增加 EngineState 的大小，下一轮 clone 时开销更大。

### 7.4 env_change 钩子的 O(N) 扫描

`crates/nu-cmd-base/src/hook.rs` L18-L37：

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

**性能注意：** `before != after` 比较的是 `Option<&Value>`，若两端均为 `Some`，则会递归比较 Value 的内容。对于包含大 list/record 的环境变量，这可能产生不可忽视的开销。

### 7.5 display_output 钩子的性能影响

`display_output` 钩子接收整条 PipelineData 作为管道输入，需要特别注意：

1. **流式数据可能被强制收集**：如果钩子代码（如默认的 `table` 命令）需要查看完整数据，会将流式输出全部加载到内存
2. **只在 REPL/脚本求值结果打印时触发**：`print` 命令和命令内部输出不经过此钩子，不会产生额外开销
3. **错误会静默抑制输出**：如果钩子返回 Err，`print_pipeline` 通过 `?` 提前返回，该命令的输出不会被打印
4. **钩子输出使用 print_raw**：避免了再次触发 display_output 的无限递归

---

## 八、实用建议与最佳实践

### 8.1 优先使用 Closure 而非 String

```nu
# ❌ 慢：每次触发都 parse
$env.config.hooks.pre_prompt = ['print "hello"']

# ✅ 快：配置时编译一次
$env.config.hooks.pre_prompt = [{|| print "hello" }]
```

### 8.2 列表中错误传播的注意事项

如果你的钩子列表包含多个步骤，且希望一个失败不影响其他步骤：
- 全部用 String 形式（隐式容错，不推荐，行为不透明）
- 或者在 Closure 中显式用 `try` 包裹：

```nu
$env.config.hooks.pre_prompt = [
    {|| try { some_risky_operation() } }  # 显式容错
    {|| always_run_this() }
]
```

### 8.3 env_change 钩子的条件优化

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

注意：由于 `previous_env_vars` 初始为空，**首次执行时 `$before` 一定是 `null`**。condition 闭包应妥善处理这种情况：

```nu
condition: {|before, after|
    ($before | is-not-empty) and  # 首次触发时 before 为 null，跳过
    ($after | path basename) == "my-project"
}
```

### 8.4 command_not_found 钩子的防循环保护

代码内置了金丝雀变量 `ENTERED_COMMAND_NOT_FOUND` 防止无限递归（`crates/nu-command/src/system/run_external.rs` L536-L547）。若钩子内部再次调用不存在的命令，会收到专门的错误信息而非再次触发钩子。但你仍应避免在钩子内调用可能失败的外部命令。

### 8.5 钩子修改命令行的技巧

`pre_prompt` 和 `env_change` 钩子可以修改 `engine_state.repl_state.buffer`，随后 `flush_engine_state_repl_buffer` 会将内容刷入 reedline 编辑器。这是实现"钩子自动填充命令行"功能的基础。

### 8.6 display_output 钩子的安全重置

`display_output` 钩子对 REPL 输出有全局影响。如果配置了有问题的钩子（例如语法错误），可能导致**所有命令输出消失**。默认配置文档中的警告（`crates/nu-utils/src/default_files/doc_config.nu` L553）：

> WARNING: A malformed hook can suppress all Nushell output.

重置方法：

```nu
$env.config.hooks.display_output = null   # 完全禁用，回到 print_table 默认行为
$env.config.hooks.display_output = ""     # 空字符串也有效
```

---

## 九、核心代码参考索引

| 功能模块 | 文件路径 | 关键行 |
|----------|----------|--------|
| Hooks 配置结构 | `crates/nu-protocol/src/config/hooks.rs` | L7-L53 |
| 钩子分发执行入口 | `crates/nu-cmd-base/src/hook.rs` | L40-L282 |
| 闭包钩子实际执行 | `crates/nu-cmd-base/src/hook.rs` | L284-L331 |
| env_change 对比逻辑 | `crates/nu-cmd-base/src/hook.rs` | L13-L38 |
| REPL 主循环触发点 | `crates/nu-cli/src/repl.rs` | L510-L848 |
| pre_execution 触发 | `crates/nu-cli/src/repl.rs` | L339-L507 |
| display_output 触发 | `crates/nu-cli/src/util.rs` | L208-L237 |
| print 命令实现 | `crates/nu-cli/src/commands/print.rs` | L50-L97 |
| print_table (不经过钩子) | `crates/nu-protocol/src/pipeline/pipeline_data.rs` | L725-L754 |
| print_raw (不经过钩子) | `crates/nu-protocol/src/pipeline/pipeline_data.rs` | L763-L792 |
| drain_to_out_dests | `crates/nu-protocol/src/pipeline/pipeline_data.rs` | L302-L327 |
| command_not_found 触发 | `crates/nu-command/src/system/run_external.rs` | L523-L639 |
| 环境重定向逻辑 | `crates/nu-engine/src/eval.rs` | L369-L384 |
| EngineState 合并环境 | `crates/nu-protocol/src/engine/engine_state.rs` | L365-L392 |
| previous_env_vars 初始化 | `crates/nu-protocol/src/engine/engine_state.rs` | L199 |
| 集成测试用例 | `tests/hooks/mod.rs` | L1-L584 |
