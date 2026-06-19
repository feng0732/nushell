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

### 4.6 钩子报错时 previous_env_vars 是否更新

`eval_env_change_hook` 的代码顺序对缓存更新至关重要（`crates/nu-cmd-base/src/hook.rs` L18-L37）：

```rust
for (env, hooks) in env_change_hook {
    let before = engine_state.previous_env_vars.get(env);
    let after = stack.get_env_var(engine_state, env.as_str());
    if before != after {
        let before = before.cloned().unwrap_or_default();
        let after = after.cloned().unwrap_or_default();

        eval_hooks(engine_state, stack,
            vec![("$before".into(), before), ("$after".into(), after.clone())],
            hooks, "env_change",
        )?;                    // ← 注意 ? 运算符

        // ↓↓↓ 只有 eval_hooks 成功返回后才会执行到这里 ↓↓↓
        Arc::make_mut(&mut engine_state.previous_env_vars)
            .insert(env.clone(), after);
    }
}
```

**结论：钩子执行成功才会更新 `previous_env_vars`；一旦报错（通过 `?` 传播），后续的 `.insert()` 不会执行。**

这导致一个可观察的行为：**如果 env_change 钩子链中的某个 Closure 钩子报错，下一轮 REPL 循环会再次触发同一个 env_change 钩子**（因为 `before` 仍然是旧值，`before != after` 仍然成立）。

| 场景 | eval_hooks 返回值 | previous_env_vars 是否更新 | 下一轮是否再次触发 |
|------|:-----------------:|:--------------------------:|:------------------:|
| 所有钩子成功执行 | `Ok(())` | ✅ 更新 | ❌（因为 before==after） |
| String 钩子运行时报错（被吞掉） | `Ok(())` | ✅ 更新 | ❌（因为最终还是 Ok） |
| String 钩子 parse 报错 | `Err` | ❌ 不更新 | ✅（下次 before 还是旧值） |
| Closure 钩子任何报错 | `Err` | ❌ 不更新 | ✅（下次 before 还是旧值） |
| Record 的 condition 报错 | `Err` | ❌ 不更新 | ✅ |
| Record 的 code Closure 报错 | `Err` | ❌ 不更新 | ✅ |
| Record 的 code String 运行时报错 | `Ok(())` | ✅ 更新 | ❌（错误被吞，最终 Ok） |

注意这里的不对称性：**String 钩子的运行时错误被 `report_shell_error` 吞掉不返回 Err，因此 eval_hooks 仍然返回 Ok，previous_env_vars 仍然会更新。但 Closure 钩子的任何错误都通过 `?` 提前返回，导致缓存不更新，下次循环再次触发。**

### 4.7 隐藏/删除变量对 env_change 检测的影响

`get_env_var` 的查找链（`crates/nu-protocol/src/engine/stack.rs` L531-L557）：

```rust
pub fn get_env_var(engine_state, name) {
    // 阶段1：从栈上的 env_vars 多层级查找（栈级覆盖值）
    for scope in self.env_vars.iter().rev() { ... }
    // 阶段2：从 engine_state 基线查找，但先检查 is_env_hidden_in_overlay
    for active_overlay in self.active_overlays.iter().rev() {
        if !self.is_env_hidden_in_overlay(active_overlay, &env_name)
            && let Some(env_vars) = engine_state.env_vars.get(active_overlay)
            && let Some(v) = env_vars.get(&env_name)
        {
            return Some(v);
        }
    }
    None
}
```

**关键结论**：只有当 `is_env_hidden_in_overlay` 返回 `true`（即基线被显式标记为隐藏）时，即使 `engine_state` 基线里有这个变量，`get_env_var` 才返回 `None`。如果只是栈上的值被删除，但基线隐藏标记没有设置，`get_env_var` 会回退到基线值。

#### remove_env_var 的 `||` 短路删除

`crates/nu-protocol/src/engine/stack.rs` L591-L596：

```rust
pub fn remove_env_var(&mut self, engine_state: &EngineState, name: &str) -> bool {
    let env_name = EnvName::from(name);

    // ⚠️ Rust 的 || 是短路的：左边为 true 时右边不会执行
    self.remove_env_var_from_stack(&env_name)
        || self.hide_engine_state_env_var(engine_state, &env_name)
}
```

这意味着存在两条完全不同的执行路径：

**路径 A（栈上有值 → `||` 短路）**：
1. `remove_env_var_from_stack` 返回 `true`（栈上找到了并删除）
2. `||` 短路，`hide_engine_state_env_var` **不执行**
3. `env_hidden` 标记**不会被设置**
4. `get_env_var` 结果：栈上找不到，但 `is_env_hidden_in_overlay` 为 false → 从基线查到 → **返回 `Some(基线值)`**

**路径 B（栈上无值 → 执行 `hide_engine_state_env_var`）**：
1. `remove_env_var_from_stack` 返回 `false`（栈上根本没有）
2. 执行 `hide_engine_state_env_var`：若基线有值，在 `env_hidden` 中标记
3. `get_env_var` 结果：栈上找不到 + 基线被标记隐藏 → **返回 `None`**

这就是注释「Use this for temporary bookkeeping removals (e.g. `FILE_PWD`, canary variables)」的语义：对于栈上临时设置的金丝雀变量，`remove_env_var` 只删除栈上的值，不会影响基线的可见性。

#### hide_env_var 的精确语义

`crates/nu-protocol/src/engine/stack.rs` L684-L707：

```rust
pub fn hide_env_var(&mut self, engine_state: &EngineState, name: &str) -> bool {
    if self.is_env_var_hide_recorded(&env_name) { return false; }

    if self.remove_env_var_from_stack(&env_name) {
        self.record_env_var_hide_in_active_overlay(&env_name);

        // 关键：栈上删除后，如果所有栈层都没有残留 shadow，才标记基线隐藏
        if !self.has_env_var_in_stack(&env_name) {
            self.hide_engine_state_env_var(engine_state, &env_name);
        }
        return true;
    }

    // 栈上没有任何值，直接尝试标记基线隐藏
    if self.hide_engine_state_env_var(engine_state, &env_name) {
        self.record_env_var_hide_in_active_overlay(&env_name);
        return true;
    }
    false
}
```

`hide_env_var` 比 `remove_env_var` 多出的步骤：
1. 检查 `env_hide_history`，防止同一 scope 重复 hide 报错
2. 无论栈上有没有值，都会调用 `record_env_var_hide_in_active_overlay` 记录隐藏历史
3. 栈上删除后，额外检查 `has_env_var_in_stack`——只有在所有栈层都没有残留值时，才标记基线隐藏

一个容易忽视的细节：如果栈上有**多层 shadow**（例如外层 scope 设了 FOO="a"，内层 scope 又设 FOO="b"），`remove_env_var_from_stack` 只删除**找到的第一个**（按 active_overlays.rev() 顺序）。如果删除后仍有残留的栈级 shadow（外层的 "a"），`has_env_var_in_stack` 返回 `true`，则基线隐藏标记**不会**被设置——因为仍然有栈级值在遮蔽基线。

#### hide_env_var vs remove_env_var 对比表

| 场景 | 操作 | 栈上的结果 | `env_hidden` 标记是否设置 | `get_env_var` 返回 | 对 `env_change` 钩子的影响 |
|------|------|-----------|:--------------------------:|--------------------|---------------------------|
| 栈上有 FOO="new"，基线有 FOO="bar" | `remove_env_var("FOO")` | 栈上的值被删除 | ❌ 不设置（`||` 短路） | `Some("bar")`（回退基线） | `after` 从 "new" 变为 "bar"，与缓存的 `before` 可能不同 → **可能触发** |
| 栈上有 FOO="new"，基线有 FOO="bar" | `hide_env_var("FOO")` | 栈上的值被删除 | ✅ 设置（因为栈上已无其他 shadow） | `None` | `after` 从 "new" 变为 `None` → **触发**，`$after = null` |
| 栈上有 FOO="inner" 和 FOO="outer" 两层 shadow，基线有 FOO="bar" | `hide_env_var("FOO")` | 只删除了找到的一层（"inner"），"outer" 还在 | ❌ 不设置（`has_env_var_in_stack` 仍为 true） | `Some("outer")`（残留栈值） | `after` 从 "inner" 变为 "outer" → **触发** |
| 栈上无值，基线有 FOO="bar" | `remove_env_var("FOO")` | 无变化 | ✅ 设置 | `None` | `after` 从 "bar" 变为 `None` → **触发**，`$after = null` |
| 栈上无值，基线有 FOO="bar" | `hide_env_var("FOO")` | 无变化 | ✅ 设置，同时记录 hide_history | `None` | `after` 从 "bar" 变为 `None` → **触发**，`$after = null` |
| 栈上无值，基线也无值 | 两者任一 | 无变化 | ❌ | `None` | 无 |

#### 实际场景对 env_change 缓存的影响

假设用户配置了 `env_change.FOO` 钩子，基线 `FOO="bar"`，用户通过 `$env.FOO = "new"` 在栈上覆盖了值，且缓存 `previous_env_vars["FOO"] = Some("new")`：

| 场景 | `get_env_var("FOO")` 返回 | `after`（比较时） | `before`（缓存） | 是否触发 | 触发后缓存更新为 |
|------|:------------------------:|:-----------------:|:----------------:|:--------:|:----------------:|
| 正常不变 | `Some("new")` | `Some("new")` | `Some("new")` | ❌ | — |
| `hide-env FOO` | `None` | `None` | `Some("new")` | ✅ | `None` |
| `remove_env_var("FOO")`（栈上有值时） | `Some("bar")` | `Some("bar")` | `Some("new")` | ✅ | `Some("bar")` |
| `hide-env FOO` 执行了，钩子报错（Closure）缓存未更新，下一轮 | `None` | `None` | `Some("new")`（仍为旧值） | ✅，再次触发 | —（仍然报错，不更新） |
| `remove_env_var("FOO")` 执行了，钩子报错（Closure）缓存未更新，下一轮 | `Some("bar")` | `Some("bar")` | `Some("new")`（仍为旧值） | ✅，再次触发 | — |

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
| `redirect_env` | `crates/nu-engine/src/eval.rs` L369-L388 | Stack → Stack | 将 callee 子栈的环境变量复制回 caller 栈，支持 hide-env |
| `merge_env` | `crates/nu-protocol/src/engine/engine_state.rs` L365-L392 | Stack → EngineState | 将 Stack 中的所有环境变量、config 变更永久写入 EngineState + 切换系统 cwd |

```rust
// redirect_env：Stack 间同步
pub fn redirect_env(engine_state, caller_stack, callee_stack) {
    let caller_env_vars = caller_stack.get_env_var_names(engine_state);

    // 步骤1：遍历 caller 中存在的所有变量名，用 callee 的 has_env_var 检查
    // callee 中"看不到"的变量（栈上没有 + 基线被隐藏）→ 在 caller 中执行 hide_env_var
    for var in caller_env_vars.iter() {
        if !callee_stack.has_env_var(engine_state, var) {
            caller_stack.hide_env_var(engine_state, var);
        }
    }

    // 步骤2：将 callee 栈级的所有环境变量（get_stack_env_vars 只返回栈上的值）复制回 caller
    for (var, value) in callee_stack.get_stack_env_vars() {
        caller_stack.add_env_var(var, value);  // 会自动清除隐藏标记
    }

    // 步骤3：同步 config 更新
    caller_stack.config.clone_from(&callee_stack.config);
}
```

**关键纠正**：`redirect_env` 的步骤1 **不使用 `get_hidden_env_vars`**，而是用 `has_env_var` 检查 callee 是否能看到每个 caller 的变量。只要 `callee_stack.has_env_var` 返回 `false`，就说明 callee 执行了 `hide-env` 或栈上删除且基线隐藏，于是在 caller 中也执行 `hide_env_var`。

另一个重要区别：`get_stack_env_vars()` 只返回**栈上存在的**环境变量，不包括 engine_state 基线值。这意味着 callee 中如果只是看到了基线值但没有在栈上重新赋值，不会触发 caller 侧的复制——caller 本来就有这个基线值。

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

## 七、报错与环境写回的精确控制流

本章深入分析：在 `eval_hook` 和 `run_hook` 中，**当错误发生时，已经在栈上修改过的环境变量是否被写回**。这取决于：
1. 错误发生在哪个阶段（parse vs 运行时）
2. 钩子形式（String vs Closure）
3. `merge_env` 与 `redirect_env` 相对于 `?` 的代码位置

### 7.1 eval_hook 函数的控制流全景

`crates/nu-cmd-base/src/hook.rs` L60-L282（`eval_hook` 函数）：

```
match value {
    String { val } => {
        parse() 编译字符串
            └─ parse 错误 → return Err(...) ──────┐
                                                     │ 跳过 L279 merge_env
        engine_state.merge_delta(delta)              │
        stack.add_var(var_id, val)                   │
        eval_block() 执行                             │
            ├─ Ok → output = pipeline                │
            └─ Err → report_shell_error（不 return） │
        stack.remove_var(var_id)                     │
    }                                                │
    List { vals } => {                               │
        eval_hooks(...)?; ← ? 提前返回 ───────────────┤
    }                                                │
    Record { val } => {                              │
        condition 阶段：                              │
            run_hook(...)?; ← ? 提前返回 ─────────────┤
            返回非 Bool → return Err(...) ────────────┤
        code String 分支：                            │
            parse 错误 → return Err(...) ─────────────┤
            eval_block() 运行时错误 → report（继续）   │
        code Closure 分支：                           │
            run_hook(...)?; ← ? 提前返回 ─────────────┤
        code 其他类型 → return Err(...) ──────────────┤
    }                                                │
    Closure { val } => {                             │
        run_hook(...)?; ← ? 提前返回 ─────────────────┤
    }                                                │
    other => return Err(...) ────────────────────────┘
}
                                               ↓
                      L279: engine_state.merge_env(stack)?;
                                               ↓
                                     Ok(output) 正常返回
```

**L279 的 `merge_env` 只有在所有 match 分支执行完（没有提前 return Err）的情况下才会执行。**

### 7.2 String 钩子的环境写回

String 钩子直接操作传入的 `&mut Stack`，不创建子栈。

| 错误阶段 | 是否提前 return | 栈上的修改是否保留 | L279 merge_env 是否执行 | 最终效果 |
|----------|:--------------:|:-----------------:|:----------------------:|----------|
| parse 阶段报错 | ✅ 是 | ❌ eval_block 未执行，栈未变 | ❌ 未执行 | 无影响 |
| merge_delta 报错 | ✅ 是（通过 `?`） | ❌ eval_block 未执行，栈未变 | ❌ 未执行 | 无影响 |
| eval_block 运行时报错 | ❌ 否（仅 report） | ✅ **已保留**（直接在传入的 stack 上修改） | ✅ 已执行 | **栈上的环境变更被永久写入 EngineState** |
| 正常无错误 | ❌ 否 | ✅ 已保留 | ✅ 已执行 | 正常写回 |

**关键发现——String 钩子的陷阱**：即使 String 钩子的 `eval_block` 运行时报错（例如 `$env.FOO = "new"; error make {...}`），`$env.FOO = "new"` 这条赋值**已经在传入的 stack 上生效了**。由于错误没有被 return，代码继续执行到 L279 `merge_env`，这个赋值会被永久写回 EngineState。**错误发生前的副作用不会被回滚。**

### 7.3 Closure 钩子的环境写回

Closure 钩子在独立的 `callee_stack`（子栈）上执行，环境同步分两步：

1. `run_hook` 函数内（L328）：`redirect_env(engine_state, stack, &callee_stack)`
2. `eval_hook` 末尾（L279）：`engine_state.merge_env(stack)`

**run_hook 内部控制流**（`crates/nu-cmd-base/src/hook.rs` L284-L331）：

```
captures_to_stack_preserve_out_dest() → 创建 callee_stack
绑定参数到 callee_stack
eval_block_with_early_return(engine_state, &mut callee_stack, ...)
    └─ ? 提前返回 ────────────────────────────┐
                                              │ 跳过 L328 redirect_env
如果返回值是 Value::Error → return Err(...) ──┤
                                              │
                                  L328: redirect_env(engine_state, stack, &callee_stack)
                                              │
                                              ↓
                                     Ok(pipeline_data) 返回
```

| 错误阶段 | 是否提前 return | callee_stack 上的修改 | redirect_env 是否执行 | merge_env（L279）是否执行 | 最终效果 |
|----------|:--------------:|:---------------------:|:--------------------:|:------------------------:|----------|
| 参数不匹配（位置参数过多） | ✅ 是（L307-310） | ❌ eval_block 未执行 | ❌ 未执行 | ❌ 未执行（eval_hook 内 ? 提前返回） | 无影响 |
| eval_block_with_early_return 运行时错误 | ✅ 是（L315-321 `?`） | ✅ 已保留在 callee_stack | ❌ 未执行 | ❌ 未执行 | **callee_stack 的修改完全丢失**（子栈销毁） |
| 返回 Value::Error | ✅ 是（L323-325） | ✅ 已保留在 callee_stack | ❌ 未执行 | ❌ 未执行 | **callee_stack 的修改完全丢失** |
| 正常无错误 | ❌ 否 | ✅ 已保留在 callee_stack | ✅ 已执行 → 写回 caller stack | ✅ 已执行 → 写回 EngineState | 正常写回 |

**关键发现——Closure 钩子的原子性**：Closure 钩子的环境变更具有"全有或全无"的特性。如果执行过程中任何时候报错，所有在 callee_stack 上进行的环境修改（`$env.VAR = value`、`hide-env` 等）**都会丢失**，不会影响调用方。这与 String 钩子「部分成功的副作用会被写回」的行为形成鲜明对比。

### 7.4 redirect_env 中的双向同步语义

`redirect_env`（`crates/nu-engine/src/eval.rs` L369-L388）在 Closure 成功执行后被调用，它做了三件事：

```rust
pub fn redirect_env(engine_state, caller_stack, callee_stack) {
    let caller_env_vars = caller_stack.get_env_var_names(engine_state);

    // 步骤1：找出 caller 能看到但 callee 看不到的变量，在 caller 中 hide
    for var in caller_env_vars.iter() {
        // has_env_var 的语义：栈上有值 OR（基线有值 AND 未被标记为 env_hidden）
        if !callee_stack.has_env_var(engine_state, var) {
            caller_stack.hide_env_var(engine_state, var);
        }
    }

    // 步骤2：将 callee 栈级的所有环境变量复制回 caller
    // 注意：get_stack_env_vars 只返回栈上的值，不包括 engine_state 基线
    for (env, value) in callee_stack.get_stack_env_vars() {
        caller_stack.add_env_var(env, value);
    }

    // 步骤3：同步 config 更新
    caller_stack.config.clone_from(&callee_stack.config);
}
```

**关键纠正**：步骤1 **不使用 `get_hidden_env_vars`**（Stack 上的另一个方法，语义完全不同：获取排除某个 overlay 后 caller 自身标记为隐藏的变量）。实际逻辑是：遍历 caller 中所有可见的变量名，用 `callee_stack.has_env_var()` 逐个检查 callee 是否还能看到。如果 callee 看不到（说明 callee 中执行了 `hide-env`，或者 callee 的栈上没有值同时 callee 的基线被标记隐藏），则在 caller 中也执行 `hide_env_var`。

`has_env_var` 的查找链与 `get_env_var` 完全一致（`crates/nu-protocol/src/engine/stack.rs` L559-L582），只是不返回 Value。这意味着：
- callee 栈上有值 → `true`
- callee 栈上无值 + 基线有值 + 基线未隐藏 → `true`
- callee 栈上无值 + 基线隐藏或不存在 → `false`

#### add_env_var 的一个副作用

`crates/nu-protocol/src/engine/stack.rs` L273-L307：

```rust
pub fn add_env_var(&mut self, var: String, value: Value) {
    let env_name = EnvName::from(var);
    // 关键：在赋值之前先清除隐藏标记
    self.clear_env_var_marks_in_active_overlay(&last_overlay, &env_name);
    // ...写入 env_vars...
}

fn clear_env_var_marks_in_active_overlay(&mut self, overlay, env_name) {
    if let Some(env_hidden) = Arc::make_mut(&mut self.env_hidden).get_mut(overlay) {
        // Re-assigning re-activates a previously hidden env var in this overlay.
        env_hidden.remove(env_name);  // ← 重新赋值会清除隐藏标记
    }
    if let Some(hide_history) = Arc::make_mut(&mut self.env_hide_history).get_mut(overlay) {
        hide_history.remove(env_name);  // ← 同时清除 hide 历史记录
    }
}
```

**结论：对一个变量重新执行 `$env.VAR = value`，会自动清除之前对它执行的 `hide-env` 标记（包括 `env_hidden` 和 `env_hide_history`）。** 这是合理的行为——如果隐藏后又显式赋值，说明需要让它重新可见。

#### redirect_env 的实际效果：先隐藏再赋值

由于步骤1（`hide_env_var`）在步骤2（`add_env_var`）之前执行，一个有趣的交互是：

如果 callee 中先 `hide-env FOO` 再 `$env.FOO = "new"`：
1. 步骤1：callee 的 `has_env_var("FOO")` 返回 `true`（因为栈上有 "new"）→ **不会**在 caller 上执行 hide
2. 步骤2：将 "new" 复制到 caller，同时清除 caller 上 FOO 的任何隐藏标记
3. 结果：caller 上 `FOO = "new"`，完全可见

如果 callee 中只 `hide-env FOO`，不再赋值：
1. 步骤1：callee 的 `has_env_var("FOO")` 返回 `false`（栈上无值 + 基线被隐藏）→ 在 caller 上执行 `hide_env_var`
2. 步骤2：callee 栈上没有 FOO，不复制
3. 结果：caller 上 FOO 也被隐藏

### 7.5 Record 钩子的两阶段控制流

Record 形式的钩子执行 `condition` 和 `code` 两个阶段，需要分别分析：

**condition 阶段**（闭包形式）：报错 → `?` 提前 return → `eval_hook` 的 L279 merge_env 不执行 → caller 栈和 callee 栈上的环境修改全部丢失。

**code 阶段**（取决于 code 的类型）：
- `code: String` → 同 7.2 String 钩子的规则
- `code: Closure` → 同 7.3 Closure 钩子的规则

**特别注意**：condition 阶段通过 `run_hook` 在独立子栈上执行，它的环境修改通过 `redirect_env` 写回 caller。如果 condition **成功执行**并返回 `true`，则 condition 阶段的环境变更已经生效。然后 code 阶段在此基础上继续修改。如果 condition 报错，则 condition 阶段的修改连同 code 阶段一起全部丢失。

### 7.6 总结：环境写回 vs 错误场景的决策矩阵

| 钩子形式 | 错误场景 | 执行前栈上的修改 | 执行到报错点前的修改 | 报错点之后的修改 | L279 merge_env |
|----------|----------|:---------------:|:-------------------:|:---------------:|:--------------:|
| **String** | parse 错误 | 未执行 | 未执行 | 未执行 | ❌ 跳过 |
| **String** | 运行时错误（被吞） | — | ✅ 已在 stack 上（不会回滚） | 未执行 | ✅ 执行，**前面修改被永久写回** |
| **String** | 无错误 | — | ✅ | ✅ | ✅ 执行 |
| **Closure** | 任何错误（通过 ? 传播） | — | ❌ 在子栈上，**全部丢失** | 未执行 | ❌ 跳过 |
| **Closure** | 无错误 | — | ✅ 通过 redirect_env 写回 caller | ✅ | ✅ 执行 |
| **List** | 子钩子返回 Err | 前面成功的子钩子已部分写回 | 本钩子未执行 | 后续钩子未执行 | ❌ 跳过（但**之前已成功的子钩子的修改不会回滚**） |
| **Record condition** | condition 报错 | — | ❌ condition 在子栈上，全部丢失 | code 未执行 | ❌ 跳过 |
| **Record condition** | condition 成功，返回 true | ✅ condition 的修改已通过 redirect_env 写回 | — | — | 取决于 code 阶段 |
| **Record code String** | String 运行时错误（被吞） | condition 已写回 | ✅ 已在 stack 上 | 未执行 | ✅ 执行 |
| **Record code Closure** | Closure 报错 | condition 已写回 | ❌ **本阶段的修改丢失** | 未执行 | ❌ 跳过（但 condition 的修改**仍保留**） |

**最容易踩坑的两种情况**：

1. **String 钩子中报错前的赋值泄漏**：
   ```nu
   $env.config.hooks.pre_prompt = ['$env.FOO = "changed"; error make {msg: "fail"}']
   ```
   → **`$env.FOO` 仍然被永久改为 `"changed"`**，即使钩子打印了错误。

2. **List 钩子中早期子钩子的成功修改不回滚**：
   ```nu
   $env.config.hooks.pre_prompt = [
       {|| $env.HOOK1 = "done" }       # 成功执行
       {|| error make {msg: "fail"} }  # Closure 报错，? 提前返回
       {|| $env.HOOK3 = "never" }      # 不会执行
   ]
   ```
   → **`$env.HOOK1 = "done"` 不会被回滚**，即使整个列表返回了 Err。下一轮循环中 `HOOK1` 已经是新值。但 `HOOK3` 永远不会被赋值。

---

## 八、性能影响分析

### 8.1 String 钩子 vs Closure 钩子：数量级差异

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

### 8.2 REPL 循环中的性能热点

在 `crates/nu-cli/src/repl.rs` 中使用 `perf!` 宏标记了关键性能点（支持 `use_ansi_coloring` 时输出到 stderr）：

| 性能标记 | 执行频率 | 说明 |
|----------|:--------:|------|
| `merge env` | 每轮 1 次 | 合并 Stack 环境到 EngineState，涉及遍历所有 overlay 的 env vars + `std::env::set_current_dir()` 系统调用 |
| `env-change hook` | 每轮 1 次 | 对比所有监控变量，每次变化触发钩子链 |
| `pre-prompt hook` | 每轮 1 次 | 读取用户输入前执行 |
| `pre_execution_hook` | 每条命令 1 次 | 命令执行前执行 |
| `update_prompt` | 每轮 1 次 | 渲染提示符（可能包含复杂闭包） |

### 8.3 EngineState 克隆的隐性开销

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

### 8.4 env_change 钩子的 O(N) 扫描

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

### 8.5 display_output 钩子的性能影响

`display_output` 钩子接收整条 PipelineData 作为管道输入，需要特别注意：

1. **流式数据可能被强制收集**：如果钩子代码（如默认的 `table` 命令）需要查看完整数据，会将流式输出全部加载到内存
2. **只在 REPL/脚本求值结果打印时触发**：`print` 命令和命令内部输出不经过此钩子，不会产生额外开销
3. **错误会静默抑制输出**：如果钩子返回 Err，`print_pipeline` 通过 `?` 提前返回，该命令的输出不会被打印
4. **钩子输出使用 print_raw**：避免了再次触发 display_output 的无限递归

---

## 九、实用建议与最佳实践

### 9.1 优先使用 Closure 而非 String

```nu
# ❌ 慢：每次触发都 parse
$env.config.hooks.pre_prompt = ['print "hello"']

# ✅ 快：配置时编译一次
$env.config.hooks.pre_prompt = [{|| print "hello" }]
```

### 9.2 列表中错误传播的注意事项

如果你的钩子列表包含多个步骤，且希望一个失败不影响其他步骤：
- 全部用 String 形式（隐式容错，不推荐，行为不透明）
- 或者在 Closure 中显式用 `try` 包裹：

```nu
$env.config.hooks.pre_prompt = [
    {|| try { some_risky_operation() } }  # 显式容错
    {|| always_run_this() }
]
```

### 9.3 env_change 钩子的条件优化

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

### 9.4 command_not_found 钩子的防循环保护

代码内置了金丝雀变量 `ENTERED_COMMAND_NOT_FOUND` 防止无限递归（`crates/nu-command/src/system/run_external.rs` L536-L547）。若钩子内部再次调用不存在的命令，会收到专门的错误信息而非再次触发钩子。但你仍应避免在钩子内调用可能失败的外部命令。

### 9.5 钩子修改命令行的技巧

`pre_prompt` 和 `env_change` 钩子可以修改 `engine_state.repl_state.buffer`，随后 `flush_engine_state_repl_buffer` 会将内容刷入 reedline 编辑器。这是实现"钩子自动填充命令行"功能的基础。

### 9.6 display_output 钩子的安全重置

`display_output` 钩子对 REPL 输出有全局影响。如果配置了有问题的钩子（例如语法错误），可能导致**所有命令输出消失**。默认配置文档中的警告（`crates/nu-utils/src/default_files/doc_config.nu` L553）：

> WARNING: A malformed hook can suppress all Nushell output.

重置方法：

```nu
$env.config.hooks.display_output = null   # 完全禁用，回到 print_table 默认行为
$env.config.hooks.display_output = ""     # 空字符串也有效
```

---

## 十、核心代码参考索引

| 功能模块 | 文件路径 | 关键行 |
|----------|----------|--------|
| Hooks 配置结构 | `crates/nu-protocol/src/config/hooks.rs` | L7-L53 |
| 钩子分发执行入口 | `crates/nu-cmd-base/src/hook.rs` | L40-L282 |
| eval_hook 末尾 merge_env | `crates/nu-cmd-base/src/hook.rs` | L279 |
| 闭包钩子实际执行 run_hook | `crates/nu-cmd-base/src/hook.rs` | L284-L331 |
| run_hook 中 redirect_env 调用点 | `crates/nu-cmd-base/src/hook.rs` | L328 |
| env_change 对比与缓存更新 | `crates/nu-cmd-base/src/hook.rs` | L13-L38 |
| Stack::add_env_var（清除隐藏标记） | `crates/nu-protocol/src/engine/stack.rs` | L273-L296 |
| Stack::get_env_var（检查 env_hidden） | `crates/nu-protocol/src/engine/stack.rs` | L531-L557 |
| Stack::remove_env_var | `crates/nu-protocol/src/engine/stack.rs` | L591-L596 |
| Stack::hide_env_var（含 hide_history） | `crates/nu-protocol/src/engine/stack.rs` | L684-L707 |
| Stack::is_env_hidden_in_overlay | `crates/nu-protocol/src/engine/stack.rs` | L671-L675 |
| Stack::captures_to_stack_preserve_out_dest | `crates/nu-protocol/src/engine/stack.rs` | L336-L357 |
| 环境重定向 redirect_env | `crates/nu-engine/src/eval.rs` | L369-L384 |
| EngineState::merge_env（永久写入） | `crates/nu-protocol/src/engine/engine_state.rs` | L365-L392 |
| previous_env_vars 初始化（空 HashMap） | `crates/nu-protocol/src/engine/engine_state.rs` | L199 |
| REPL 主循环触发点 | `crates/nu-cli/src/repl.rs` | L510-L848 |
| pre_execution 触发 | `crates/nu-cli/src/repl.rs` | L339-L507 |
| display_output 触发 | `crates/nu-cli/src/util.rs` | L208-L237 |
| print 命令实现 | `crates/nu-cli/src/commands/print.rs` | L50-L97 |
| print_table（不经过钩子） | `crates/nu-protocol/src/pipeline/pipeline_data.rs` | L725-L754 |
| print_raw（不经过钩子） | `crates/nu-protocol/src/pipeline/pipeline_data.rs` | L763-L792 |
| drain_to_out_dests | `crates/nu-protocol/src/pipeline/pipeline_data.rs` | L302-L327 |
| command_not_found 触发 | `crates/nu-command/src/system/run_external.rs` | L523-L639 |
| 集成测试用例 | `tests/hooks/mod.rs` | L1-L584 |
