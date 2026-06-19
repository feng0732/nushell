# Shell Prompt 与状态行渲染求值分析

本文档分析 Nushell 中 shell prompt 和状态行（status line）的渲染求值机制，重点澄清「渲染前求值」与「渲染时求值」的容易混淆之处，并详细说明左右区域、回退逻辑和多行表现的实现。

## 一、核心架构概览

### 1.1 关键组件

| 组件 | 位置 | 职责 |
|------|------|------|
| `NushellPrompt` | [prompt.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/71-nushell/crates/nu-cli/src/prompt.rs) | 实现 `reedline::Prompt` trait，持有各部分 prompt 字符串，提供渲染方法 |
| `update_prompt()` | [prompt_update.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/71-nushell/crates/nu-cli/src/prompt_update.rs#L92-L124) | **渲染前** 调用，负责求值并更新 prompt 字符串缓存 |
| `get_prompt_string()` | [prompt_update.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/71-nushell/crates/nu-cli/src/prompt_update.rs#L51-L90) | 实际的求值函数，处理 String 和 Closure 两种类型 |
| `make_transient_prompt()` | [prompt_update.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/71-nushell/crates/nu-cli/src/prompt_update.rs#L129-L176) | 构建 transient prompt |
| REPL 主循环 | [repl.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/71-nushell/crates/nu-cli/src/repl.rs#L713-L743) | 控制 prompt 更新与读取的时序 |

### 1.2 时序与生命周期：两个关键阶段

**最容易混淆的地方在于「求值时机」与「渲染时机」的分离：**

```
          渲染前（求值阶段）              |           渲染时（展示阶段）
------------------------------------------+----------------------------------------
update_prompt()                           |  reedline 内部调用
  ↳ get_prompt_string()                   |    ↳ render_prompt_left()
    ↳ 读取环境变量                        |    ↳ render_prompt_right()
    ↳ 若是 Closure 则执行求值             |    ↳ render_prompt_indicator()
    ↳ 结果存入 NushellPrompt 字段         |    ↳ render_prompt_multiline_indicator()
  ↳ update_all_prompt_strings()           |    ↳ right_prompt_on_last_line()
                                          |
  时机：每次 read_line() 调用前            |  时机：reedline 需要重绘时（可能多次）
  频率：每个 REPL 迭代一次                 |  频率：取决于终端事件，可能多次
  状态：持有最新的字符串缓存               |  行为：只读，从缓存中读取
```

## 二、左右区域渲染逻辑

### 2.1 左区域（Left Prompt）

**渲染前**在 [update_prompt()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-nushell/crates/nu-cli/src/prompt_update.rs#L99) 中求值：

```rust
let left_prompt_string = get_prompt_string(PROMPT_COMMAND, config, engine_state, stack);
nu_prompt.update_prompt_left(left_prompt_string);
```

**渲染时**在 [render_prompt_left()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-nushell/crates/nu-cli/src/prompt.rs#L89-L105) 中读取缓存：

```rust
fn render_prompt_left(&self) -> Cow<'_, str> {
    if let Some(prompt_string) = &self.left_prompt {
        prompt_string.replace('\n', "\r\n").into()
    } else {
        // 回退到 reedline 默认 prompt
        let default = DefaultPrompt::default();
        default.render_prompt_left().to_string().replace('\n', "\r\n").into()
    }
}
```

### 2.2 右区域（Right Prompt / 状态行）

**渲染前**在 [update_prompt()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-nushell/crates/nu-cli/src/prompt_update.rs#L101) 中求值：

```rust
let right_prompt_string = get_prompt_string(PROMPT_COMMAND_RIGHT, config, engine_state, stack);
```

**渲染时**在 [render_prompt_right()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-nushell/crates/nu-cli/src/prompt.rs#L107-L118) 中读取缓存，逻辑与左区域对称。

### 2.3 右区域位置控制

通过配置项 `render_right_prompt_on_last_line` 控制右 prompt 渲染位置：

- **配置定义**：[config/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/71-nushell/crates/nu-protocol/src/config/mod.rs#L74)
- **渲染时查询**：[right_prompt_on_last_line()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-nushell/crates/nu-cli/src/prompt.rs#L158-L160)

```
render_right_prompt_on_last_line = false:

    [left prompt]  [command input]                [right prompt]
    /home/user>   echo "hello"                   10:30 AM

render_right_prompt_on_last_line = true:

    [left prompt]  [command input]
    /home/user>   echo "hello"                    [right prompt]
                                                 10:30 AM
```

## 三、回退（Fallback）逻辑详解

回退逻辑分布在「求值阶段」和「渲染阶段」两个层面，形成多层保护机制。

### 3.1 第一层：求值时的回退（get_prompt_string）

[get_prompt_string()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-nushell/crates/nu-cli/src/prompt_update.rs#L51-L90) 函数内部的回退链：

```rust
fn get_prompt_string(prompt: &str, ...) -> Option<String> {
    let mut output = match stack.get_env_var(engine_state, prompt)? {
        // Case 1: 环境变量是字符串 → 直接使用
        Value::String { val, .. } => val.clone(),

        // Case 2: 环境变量是 Closure → 执行求值
        Value::Closure { val, .. } => {
            let result = ClosureEvalOnce::new(engine_state, stack, val.as_ref().clone())
                .run_with_input(PipelineData::empty());

            // Closure 执行失败 → 静默回退
            let result_string = result
                .map_err(|err| report_shell_error(None, engine_state, &err))
                .ok()  // 错误转换为 None
                .and_then(|pd| pd.collect_string("", config).ok());

            result_string?  // 若为 None，整个函数返回 None
        }

        // Case 3: 其他类型 → 返回 None，进入下一层回退
        _ => return None,
    };

    // 右 prompt 的特殊处理：即使为空也要插入颜色重置
    if output.is_empty() && prompt == PROMPT_COMMAND_RIGHT {
        output.insert_str(0, "\x1b[0m")
    };

    Some(output)
}
```

**回退点 1**：`stack.get_env_var(...)` 返回 `None` → 函数返回 `None`
**回退点 2**：`Closure` 执行出错 → `ok()` 转换为 `None`
**回退点 3**：`PipelineData` 收集字符串失败 → `ok()` 转换为 `None`
**回退点 4**：环境变量类型不匹配 → 直接 `return None`

### 3.2 第二层：渲染时的回退（render_prompt_*）

当 `get_prompt_string()` 返回 `None` 时，`NushellPrompt` 的对应字段为 `None`，此时在渲染时触发回退：

**左/右 Prompt 回退**：
```rust
// render_prompt_left() 和 render_prompt_right() 逻辑相同
if let Some(prompt_string) = &self.left_prompt {
    prompt_string.replace('\n', "\r\n").into()
} else {
    // 回退到 reedline 的 DefaultPrompt
    let default = DefaultPrompt::default();
    default.render_prompt_left().to_string().replace('\n', "\r\n").into()
}
```

**Indicator 回退**：
```rust
fn render_prompt_indicator(&self, edit_mode: PromptEditMode) -> Cow<'_, str> {
    let indicator: &str = match edit_mode {
        PromptEditMode::Default => self.prompt_indicator.as_deref().unwrap_or("> "),
        PromptEditMode::Emacs => self.prompt_indicator.as_deref().unwrap_or("> "),
        PromptEditMode::Vi(vi_mode) => match vi_mode {
            PromptViMode::Normal => self.vi_normal_prompt_indicator.as_deref().unwrap_or("> "),
            PromptViMode::Insert => self.vi_insert_prompt_indicator.as_deref().unwrap_or(": "),
        },
        PromptEditMode::Custom(str) => &self.default_wrapped_custom_string(str),
    };
    indicator.to_string().into()
}
```

**多行 Indicator 回退**：
```rust
fn render_prompt_multiline_indicator(&self) -> Cow<'_, str> {
    let indicator = match &self.multiline_indicator {
        Some(indicator) => indicator.as_str(),
        None => "::: ",  // 默认回退值
    };
    indicator.to_string().into()
}
```

### 3.3 回退链全景

```
用户配置 PROMPT_COMMAND
        │
        ▼
get_prompt_string() 求值
        │
        ├─→ 环境变量不存在？────────┐
        ├─→ 类型不匹配？───────────┤
        ├─→ Closure 执行失败？─────┤
        └─→ 正常返回 String        │
              │                    │
              ▼                    ▼
      update_all_prompt_strings() 字段为 None
              │                    │
              ▼                    ▼
render_prompt_*() 调用时         回退到 DefaultPrompt
      返回缓存字符串             或默认 indicator
```

## 四、多行表现（Multiline）

### 4.1 多行 Indicator 的求值

**渲染前**在 [update_prompt()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-nushell/crates/nu-cli/src/prompt_update.rs#L105-L106) 中求值：

```rust
let prompt_multiline_string =
    get_prompt_string(PROMPT_MULTILINE_INDICATOR, config, engine_state, stack);
```

**渲染时**在 [render_prompt_multiline_indicator()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-nushell/crates/nu-cli/src/prompt.rs#L134-L141) 中使用，默认值为 `"::: "`。

### 4.2 多行输入的显示效果

当用户输入未闭合的括号、引号或使用 `\` 换行时，reedline 进入多行编辑模式：

```
# 普通单行模式
/home/user> echo "hello world"
           ↑
     render_prompt_indicator() 返回 "> "

# 多行模式
/home/user> echo "hello
::: world
::: second line"
           ↑
     render_prompt_multiline_indicator() 返回 "::: "
```

### 4.3 换行符处理

在 `render_prompt_left()` 和 `render_prompt_right()` 中，会将 `\n` 替换为 `\r\n`：

```rust
prompt_string.replace('\n', "\r\n").into()
```

这确保了 prompt 本身包含换行时（如多行 prompt）在终端中正确显示。

## 五、完整的 REPL 迭代流程

[repl.rs: loop_iteration()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-nushell/crates/nu-cli/src/repl.rs#L713-L748) 中的时序：

```
start_time = Instant::now();
let config = &engine_state.get_config().clone();

// ─── 阶段 1: 渲染前求值 ───
// 创建子 stack 用于 prompt 求值（避免污染主 stack）
prompt_update::update_prompt(
    config,
    engine_state,
    &mut Stack::with_parent(stack_arc.clone()),  // 子 stack
    nu_prompt,
);

// 同样创建 transient prompt
let transient_prompt = prompt_update::make_transient_prompt(
    config,
    engine_state,
    &mut Stack::with_parent(stack_arc.clone()),  // 另一个子 stack
    nu_prompt,
);

perf!("update_prompt", start_time, use_color);

// ─── 阶段 2: 渲染与输入读取 ───
line_editor = line_editor.with_transient_prompt(transient_prompt);

// reedline 内部会多次调用 render_prompt_*() 方法
// 这些调用仅读取缓存，不进行求值
let input = line_editor.read_line(nu_prompt);

// ─── 阶段 3: 清理 ───
line_editor = line_editor
    .with_highlighter(Box::<NoOpHighlighter>::default())
    .with_completer(Box::<DefaultCompleter>::default());
```

### 5.1 关于 Stack 的重要细节

注意 `update_prompt()` 使用的是 **子 Stack**：

```rust
&mut Stack::with_parent(stack_arc.clone())
```

这意味着：
1. Prompt 闭包中的变量修改不会影响主 REPL 栈
2. Prompt 闭包可以读取所有环境变量和配置
3. 求值过程中的副作用被隔离

## 六、Transient Prompt 机制

[make_transient_prompt()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-nushell/crates/nu-cli/src/prompt_update.rs#L129-L176) 在执行命令后替换原有 prompt，使历史记录更简洁。

**求值时机**：与普通 prompt 相同，在 `read_line()` 前求值。

**回退逻辑**：Transient prompt 各部分的回退是「有则用之，无则继承」：

```rust
if let Some(s) = get_prompt_string(TRANSIENT_PROMPT_COMMAND, ...) {
    nu_prompt.update_prompt_left(Some(s))
}
// 若 TRANSIENT_* 未设置，则保留原 prompt 的值
```

## 七、环境变量清单

所有可配置的 prompt 环境变量定义在 [prompt_update.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/71-nushell/crates/nu-cli/src/prompt_update.rs#L12-L26)：

| 常量名 | 环境变量名 | 用途 |
|--------|-----------|------|
| `PROMPT_COMMAND` | `PROMPT_COMMAND` | 左 prompt |
| `PROMPT_COMMAND_RIGHT` | `PROMPT_COMMAND_RIGHT` | 右 prompt（状态行） |
| `PROMPT_INDICATOR` | `PROMPT_INDICATOR` | 命令输入指示器 |
| `PROMPT_INDICATOR_VI_INSERT` | `PROMPT_INDICATOR_VI_INSERT` | VI 插入模式指示器 |
| `PROMPT_INDICATOR_VI_NORMAL` | `PROMPT_INDICATOR_VI_NORMAL` | VI 正常模式指示器 |
| `PROMPT_MULTILINE_INDICATOR` | `PROMPT_MULTILINE_INDICATOR` | 多行输入指示器 |
| `TRANSIENT_*` | `TRANSIENT_*` | 对应 transient 版本 |

## 八、容易混淆的点总结

### 8.1 求值 vs 渲染

| 方面 | 求值阶段（渲染前） | 渲染阶段（渲染时） |
|------|------------------|------------------|
| 触发方 | Nushell REPL | reedline 库 |
| 时机 | `read_line()` 之前 | 终端需要重绘时 |
| 频率 | 每次 REPL 迭代一次 | 可能多次（按键、窗口大小变化等） |
| 行为 | 执行 Closure、收集字符串 | 读取缓存字符串 |
| 可修改状态 | 是（更新缓存） | 否（只读） |
| Stack | 使用子 Stack，隔离副作用 | 不涉及 Stack |

### 8.2 回退层级

1. **Closure 执行失败** → 静默回退，不崩溃
2. **环境变量不存在** → 字段设为 `None`
3. **渲染时字段为 `None`** → 使用 reedline 默认值或硬编码默认值

### 8.3 左右区域的差异

- 求值逻辑完全相同，只是环境变量名不同
- 右 prompt 有额外的颜色重置保护（`\x1b[0m`）
- 右 prompt 位置可配置是否显示在最后一行

### 8.4 多行 vs 单行

- 多行 indicator 是独立求值的，可单独配置
- 多行模式由 reedline 根据语法分析自动触发
- 多行 indicator 仅在换行后显示
