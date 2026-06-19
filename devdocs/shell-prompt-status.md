# Shell Prompt 与状态行渲染求值分析

> 仓库相对路径：`devdocs/shell-prompt-status.md`
> 相关代码：`crates/nu-cli/src/prompt.rs`、`crates/nu-cli/src/prompt_update.rs`、`crates/nu-cli/src/repl.rs`

本文档分析 Nushell 中 shell prompt 和状态行（status line）的渲染求值机制，重点澄清「渲染前求值」与「渲染时求值」的容易混淆之处。

**阅读指引：** 本文严格区分两类内容：
- ✅ **代码确定事实**：可在 Nushell 仓库中通过代码直接验证
- 🔍 **reedline 外部行为**：基于 `reedline::Prompt` trait 接口契约和配置文档的合理推断，具体实现位于 reedline 库外部

## 一、核心架构概览

### 1.1 关键组件

| 组件 | 位置 | 类型 | 职责 |
|------|------|------|------|
| `NushellPrompt` | `crates/nu-cli/src/prompt.rs` | ✅ 代码事实 | 实现 `reedline::Prompt` trait，持有各部分 prompt 字符串缓存 |
| `update_prompt()` | `crates/nu-cli/src/prompt_update.rs` | ✅ 代码事实 | **渲染前** 调用，负责求值并更新 prompt 缓存 |
| `get_prompt_string()` | `crates/nu-cli/src/prompt_update.rs` | ✅ 代码事实 | 实际的求值函数，处理 String 和 Closure 两种类型 |
| `make_transient_prompt()` | `crates/nu-cli/src/prompt_update.rs` | ✅ 代码事实 | 构建 transient prompt |
| REPL 主循环 | `crates/nu-cli/src/repl.rs` | ✅ 代码事实 | 控制 prompt 更新与读取的时序 |
| `reedline::Prompt` trait | 外部依赖 | 🔍 接口契约 | 定义渲染接口，reedline 内部决定何时调用 |

- [NushellPrompt 定义](../crates/nu-cli/src/prompt.rs#L10-L19)
- [update_prompt() 函数](../crates/nu-cli/src/prompt_update.rs#L92-L124)
- [REPL 中调用 update_prompt](../crates/nu-cli/src/repl.rs#L713-L726)

### 1.2 时序与生命周期：两个关键阶段

**最容易混淆的地方在于「求值时机」与「渲染时机」的分离：**

```
          渲染前（求值阶段）              |           渲染时（展示阶段）
------------------------------------------+----------------------------------------
✅ Nushell 代码控制                       |  🔍 reedline 内部控制
                                          |
update_prompt()                           |  reedline 根据需要调用
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

**✅ 渲染前求值** — 在 [update_prompt()](../crates/nu-cli/src/prompt_update.rs#L99) 中：

```rust
let left_prompt_string = get_prompt_string(PROMPT_COMMAND, config, engine_state, stack);
// 存入 NushellPrompt.left_prompt 字段
```

**✅ 渲染时读取缓存** — 在 [render_prompt_left()](../crates/nu-cli/src/prompt.rs#L89-L105) 中：

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

**代码确定事实：**
- `left_prompt` 字段类型为 `Option<String>`，`None` 表示未设置
- 渲染时做 `\n` → `\r\n` 的换行符转换
- 回退链：用户配置 → `DefaultPrompt::default()`

### 2.2 右区域（Right Prompt / 状态行）

**✅ 渲染前求值** — 在 [update_prompt()](../crates/nu-cli/src/prompt_update.rs#L101) 中：

```rust
let right_prompt_string = get_prompt_string(PROMPT_COMMAND_RIGHT, config, engine_state, stack);
```

**✅ 渲染时读取缓存** — 在 [render_prompt_right()](../crates/nu-cli/src/prompt.rs#L107-L118) 中，逻辑与左区域对称。

**✅ 右 prompt 颜色重置保护** — 在 [get_prompt_string()](../crates/nu-cli/src/prompt_update.rs#L80-L84) 中：

```rust
// Always reset the color at the start of the right prompt
// to ensure there is no ansi bleed over
if output.is_empty() && prompt == PROMPT_COMMAND_RIGHT {
    output.insert_str(0, "\x1b[0m")
};
```

### 2.3 换行符转换处理（代码确定事实）

**✅ 发生在渲染时，而非求值时**

在 `render_prompt_left()` 和 `render_prompt_right()` 中，都会将 `\n` 替换为 `\r\n`：

```rust
prompt_string.replace('\n', "\r\n").into()
```

**确定事实：**
- 求值阶段存储在 `left_prompt` / `right_prompt` 中的是原始字符串（包含 `\n`）
- 每次 `render_prompt_*()` 调用时才执行替换
- `render_prompt_indicator()` 和 `render_prompt_multiline_indicator()` **不做** 此转换

### 2.4 右 Prompt 定位配置：render_right_prompt_on_last_line

#### 2.4.1 Nushell 代码中的确定事实

- **配置定义**：[`nu-protocol` Config 结构体](../crates/nu-protocol/src/config/mod.rs#L74) 中 `pub render_right_prompt_on_last_line: bool`
- **默认值**：[`false`](../crates/nu-protocol/src/config/mod.rs#L131)
- **字段存储**：[`NushellPrompt.render_right_prompt_on_last_line`](../crates/nu-cli/src/prompt.rs#L18)
- **更新入口**：[`update_all_prompt_strings()`](../crates/nu-cli/src/prompt.rs#L63-L81) 接收配置值并存储
- **渲染时查询**：[`right_prompt_on_last_line()`](../crates/nu-cli/src/prompt.rs#L158-L160) 直接返回字段值

**配置传递链路（代码确定事实）：**

```
config.render_right_prompt_on_last_line  (nu-protocol Config)
        │
        ▼
update_prompt() 读取 config
        │
        ▼
nu_prompt.update_all_prompt_strings(..., render_right_prompt_on_last_line)
        │
        ▼
self.render_right_prompt_on_last_line = render_right_prompt_on_last_line
        │
        ▼
right_prompt_on_last_line() -> bool     (reedline::Prompt trait 方法)
        │
        ▼
reedline 内部使用该值进行布局             （🔍 外部行为）
```

#### 2.4.2 配置文档描述（官方语义）

[`doc_config.nu`](../crates/nu-utils/src/default_files/doc_config.nu#L506-L510) 中的官方说明：

```
# render_right_prompt_on_last_line (bool): Right prompt position with multi-line left prompt.
# true: Right prompt appears on the last line of the left prompt.
# false: Right prompt appears on the first line.
# Default: false
```

**从官方文档可确定的语义：**
1. 该配置仅与「multi-line left prompt」（多行左 prompt）相关
2. `true`：右 prompt 出现在左 prompt 的**最后一行**
3. `false`：右 prompt 出现在左 prompt 的**第一行**

#### 2.4.3 🔍 reedline 外部行为边界

> 以下内容基于 `Prompt` trait 接口和配置文档的合理推断，具体实现位于 reedline 库中。

`reedline::Prompt` trait 提供的相关方法：

```rust
pub trait Prompt {
    fn render_prompt_left(&self) -> Cow<'_, str>;     // 左 prompt 字符串（可能含 \n）
    fn render_prompt_right(&self) -> Cow<'_, str>;    // 右 prompt 字符串（可能含 \n）
    fn right_prompt_on_last_line(&self) -> bool;      // 右 prompt 垂直定位开关
    fn render_prompt_indicator(&self, edit_mode: PromptEditMode) -> Cow<'_, str>;
    fn render_prompt_multiline_indicator(&self) -> Cow<'_, str>;
    // ...
}
```

**基于接口和文档的推断：**
- reedline 调用 `render_prompt_left()` 获取左 prompt 文本，按换行符分割为多行
- reedline 调用 `right_prompt_on_last_line()` 获取定位模式
- `false` → 右 prompt 锚定到左 prompt 的第 0 行（第一行）
- `true` → 右 prompt 锚定到左 prompt 的最后一行（第 N-1 行）
- 用户输入内容始终接在左 prompt 最后一行的后面

**示意图（基于官方语义）：**

```
情况 1: render_right_prompt_on_last_line = false（默认值）

    ┌──────────── 左 Prompt 第 1 行 ───────────┐    ┌─ 右 Prompt ─┐
    │  user@host  ~/projects/nushell (main)    │    │ 10:30:45 AM │
    ├──────────── 左 Prompt 第 2 行（最后一行）──┤    └─────────────┘
    │ ❯ echo "hello world"                     │
    └──────────────────────────────────────────┘
    ↑                                          ↑
 left_prompt 第一行                        右 prompt 锚定到
 的最右侧                                  第一行末尾


情况 2: render_right_prompt_on_last_line = true

    ┌──────────── 左 Prompt 第 1 行 ───────────┐
    │  user@host  ~/projects/nushell (main)    │
    ├──────────── 左 Prompt 第 2 行（最后一行）──┤    ┌─ 右 Prompt ─┐
    │ ❯ echo "hello world"                     │    │ 10:30:45 AM │
    └──────────────────────────────────────────┘    └─────────────┘
                                                   ↑
                                              右 prompt 锚定到
                                              最后一行末尾
```

## 三、回退（Fallback）逻辑详解

回退逻辑分布在「求值阶段」和「渲染阶段」两个层面，形成多层保护机制。

### 3.1 第一层：求值时的回退（get_prompt_string）

✅ **全部为代码确定事实**

[get_prompt_string()](../crates/nu-cli/src/prompt_update.rs#L51-L90) 函数内部的回退链：

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

**四个回退点（按触发顺序）：**

| 回退点 | 触发条件 | 行为 |
|--------|---------|------|
| 1 | `stack.get_env_var(...)` 返回 `None` | 函数返回 `None` |
| 2 | `Closure` 执行出错 | `ok()` 将错误转换为 `None` |
| 3 | `PipelineData` 收集字符串失败 | `ok()` 转换为 `None` |
| 4 | 环境变量类型不是 String 也不是 Closure | 直接 `return None` |

### 3.2 第二层：渲染时的回退（render_prompt_*）

✅ **全部为代码确定事实**

当 `get_prompt_string()` 返回 `None` 时，`NushellPrompt` 的对应字段为 `None`，此时在渲染时触发回退。

**左/右 Prompt 回退** — [render_prompt_left()](../crates/nu-cli/src/prompt.rs#L95-L104) / [render_prompt_right()](../crates/nu-cli/src/prompt.rs#L108-L117)：

```rust
if let Some(prompt_string) = &self.left_prompt {
    prompt_string.replace('\n', "\r\n").into()
} else {
    // 回退到 reedline 的 DefaultPrompt
    let default = DefaultPrompt::default();
    default.render_prompt_left().to_string().replace('\n', "\r\n").into()
}
```

**Indicator 回退** — [render_prompt_indicator()](../crates/nu-cli/src/prompt.rs#L120-L132)：

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

**多行 Indicator 回退** — [render_prompt_multiline_indicator()](../crates/nu-cli/src/prompt.rs#L134-L141)：

```rust
fn render_prompt_multiline_indicator(&self) -> Cow<'_, str> {
    let indicator = match &self.multiline_indicator {
        Some(indicator) => indicator.as_str(),
        None => "::: ",  // 硬编码默认回退值
    };
    indicator.to_string().into()
}
```

### 3.3 回退链全景

```
用户配置 PROMPT_COMMAND
        │
        ▼
✅ get_prompt_string() 求值
        │
        ├─→ 环境变量不存在？────────┐
        ├─→ 类型不匹配？───────────┤
        ├─→ Closure 执行失败？─────┤
        └─→ 正常返回 String        │
              │                    │
              ▼                    ▼
✅     update_all_prompt_strings()  字段为 None
              │                    │
              ▼                    ▼
✅ render_prompt_*() 调用时         回退到 DefaultPrompt
      返回缓存字符串               或默认 indicator
```

## 四、多行表现（Multiline）完整分析

### 4.1 两种「多行」概念辨析

这是最容易混淆的地方，**必须明确区分两种完全不同的多行场景**：

| 场景 | 触发原因 | 涉及的核心方法 | 配置项 | 类型 |
|------|---------|---------------|--------|------|
| **场景 A：左 Prompt 本身多行** | `PROMPT_COMMAND` 返回字符串含 `\n` | `render_prompt_left()` / `right_prompt_on_last_line()` | `render_right_prompt_on_last_line` | ✅ 左 prompt 行数可由代码验证 |
| **场景 B：用户输入多行** | 语法未闭合，reedline 进入续行模式 | `render_prompt_multiline_indicator()` | `PROMPT_MULTILINE_INDICATOR` | 🔍 由 reedline 内部控制触发 |
| **场景 C：A + B 组合** | 左 prompt 多行 + 用户输入多行 | 上述所有方法 | 上述所有配置 | — |

### 4.2 场景 A：左 Prompt 本身多行

✅ **左 prompt 可能包含换行** — 代码确定事实：`render_prompt_left()` 做 `\n` → `\r\n` 转换，说明左 prompt 可以是多行的。

🔍 **右 prompt 定位行为** — 基于配置文档的合理推断：

| 配置值 | 右 prompt 垂直位置 | 左 prompt 只有 1 行时 |
|--------|-----------------|-------------------|
| `false`（默认） | 对齐左 prompt 第 1 行 | 与 `true` 无区别（只有一行） |
| `true` | 对齐左 prompt 最后一行 | 与 `false` 无区别（只有一行） |

**关键结论：`render_right_prompt_on_last_line` 只有在左 prompt 有多行时才有实际效果。**

### 4.3 场景 B：用户输入多行（Multiline Input）

#### 4.3.1 ✅ 代码确定事实

**多行 Indicator 的求值**：
- 环境变量：`PROMPT_MULTILINE_INDICATOR`
- 求值时机：与其他 prompt 组件同时，在 `update_prompt()` 中 [第 105-106 行](../crates/nu-cli/src/prompt_update.rs#L105-L106)
- 渲染方法：[`render_prompt_multiline_indicator()`](../crates/nu-cli/src/prompt.rs#L134-L141)
- 默认值：`"::: "`（硬编码回退值）

**两个独立的 indicator 方法**：
- `render_prompt_indicator(edit_mode)` — 带编辑模式参数，用于首行/普通模式
- `render_prompt_multiline_indicator()` — 无参数，用于续行模式

#### 4.3.2 🔍 reedline 外部行为边界

**何时调用哪个 indicator 方法由 reedline 决定**，通常的理解是：

- 用户在第一行输入时 → 调用 `render_prompt_indicator()`
- 用户输入进入续行（第 2..N 行）时 → 调用 `render_prompt_multiline_indicator()`

**触发多行输入的条件（由 reedline 内部判定，Nushell 不控制）：**
- 未闭合的双引号 `"` 或单引号 `'`
- 未闭合的括号 `()`, `[]`, `{}`
- 行末尾的反斜杠 `\` 续行符
- 管道符 `|` 出现在行末
- （具体以 reedline 实现为准）

#### 4.3.3 多行 Indicator 的水平定位

🔍 **这是 reedline 外部行为**

常见的终端续行模式中，续行 indicator 通常对齐到主指示符的起始列。例如：

```
/home/user> echo "hello
::: world
::: second line"
  ↑
  多行 indicator 对齐到主 indicator 的同一列
```

**关于水平定位的合理推断（基于常见终端行为）：**
1. `render_prompt_multiline_indicator()` 只返回显示内容字符串（如 `"::: "`）
2. 水平位置由 reedline 计算，对齐到左 prompt 最后一行的指示符列
3. 用户不需要自己在 multiline indicator 中填充空格

> ⚠️ 注意：以上关于水平对齐的描述基于常见 terminal UI 模式，具体行为以 reedline 实现为准。Nushell 代码中不包含 multiline indicator 的水平偏移计算逻辑。

#### 4.3.4 用户输入多行时的右 Prompt 定位

🔍 **这是 reedline 外部行为**

当左 prompt 是单行但用户输入进入多行时，右 prompt 的定位行为：

- `right_prompt_on_last_line` 配置的语义是「相对于左 prompt 的行」，而非「相对于用户输入行」
- 因此，左 prompt 只有 1 行时，`true` 和 `false` 效果相同：右 prompt 都锚定在左 prompt 那一行的最右侧
- 用户输入的多行（续行）不会改变右 prompt 的垂直锚点

**示意图（基于语义推断）：**

```
左 Prompt: 1 行
用户输入: 3 行（未闭合字符串）
配置: render_right_prompt_on_last_line = false（或 true，效果相同）

    ┌─ 左 Prompt ─┐    echo "line one     ┌─ 右 Prompt ─┐
    │ /home/user> │                     │ 10:30:45 AM │
    │             │   ::: line two      │ └─────────────┘
    │             │   ::: line three"   │
    └─────────────┘                     └───────────────┘
                    ↑                          ↑
              用户输入行从左 prompt        右 prompt 锚定在
              最后一行之后开始            左 prompt 那一行
                                        （只有一行，true/false 无区别）
```

### 4.4 场景 C：左 Prompt 多行 + 用户输入多行（组合场景）

🔍 **以下为基于接口语义的推断组合**

当左 prompt 本身有多行，且用户输入也进入多行时，两种配置的效果差异：

#### 4.4.1 配置 render_right_prompt_on_last_line = false（默认）

```
左 Prompt: 2 行（用户信息行 + 指示符行）
用户输入: 3 行（未闭合字符串）

    ┌──────────── 左 Prompt 第 1 行 ───────────┐    ┌─ 右 Prompt ─┐
    │  user@host  ~/projects (main)            │    │ 10:30:45 AM │
    ├──────────── 左 Prompt 第 2 行（最后一行）──┤    └─────────────┘
    │ ❯ echo "This is a very long string that   │
    │     spans across multiple                 │
    │     lines"                                │
    └───────────────────────────────────────────┘

右 Prompt: 对齐左 Prompt 第 1 行（最顶部）
用户输入续行: 接在左 prompt 最后一行之后，每行有 multiline indicator
```

#### 4.4.2 配置 render_right_prompt_on_last_line = true

```
左 Prompt: 2 行
用户输入: 3 行

    ┌──────────── 左 Prompt 第 1 行 ───────────┐
    │  user@host  ~/projects (main)            │
    ├──────────── 左 Prompt 第 2 行（最后一行）──┤    ┌─ 右 Prompt ─┐
    │ ❯ echo "This is a very long string that   │    │ 10:30:45 AM │
    │     spans across multiple                 │    └─────────────┘
    │     lines"                                │
    └───────────────────────────────────────────┘

右 Prompt: 对齐左 Prompt 第 2 行（底部，与指示符和输入首行同行）
用户输入续行: 接在左 prompt 最后一行之后
```

> ⚠️ 以上组合场景的图示基于 `right_prompt_on_last_line` 配置的语义进行推导，实际渲染效果以 reedline 行为为准。

### 4.5 右 Prompt 本身多行（Right Prompt Contains Newlines）

🔍 **这是 reedline 外部行为**

右 prompt 字符串也可以包含 `\n`（因为 `render_prompt_right()` 同样做 `\n` → `\r\n` 转换）。

**推断行为：**
- 右 prompt 的锚点行由 `right_prompt_on_last_line()` 决定（相对于左 prompt 的行）
- 右 prompt 的多行内容向「下方」延伸
- 如果右 prompt 行数过多超过终端高度，可能被截断

### 4.6 行为边界总结表

| 行为 | 类型 | 依据 |
|------|------|------|
| `update_prompt()` 在 `read_line()` 前调用 | ✅ 代码事实 | [repl.rs:715](../crates/nu-cli/src/repl.rs#L715) |
| 左/右 prompt 字符串缓存于 `NushellPrompt` | ✅ 代码事实 | [prompt.rs:12-13](../crates/nu-cli/src/prompt.rs#L12-L13) |
| 渲染时做 `\n` → `\r\n` 转换 | ✅ 代码事实 | [prompt.rs:96](../crates/nu-cli/src/prompt.rs#L96) |
| `right_prompt_on_last_line` 控制右 prompt 垂直位置 | ✅ 接口语义 + 配置文档 | [doc_config.nu:506](../crates/nu-utils/src/default_files/doc_config.nu#L506) |
| `render_prompt_multiline_indicator()` 在续行时调用 | 🔍 reedline 行为 | `Prompt` trait 方法设计意图 |
| 多行 indicator 水平对齐到指示符列 | 🔍 reedline 行为 | 常见终端模式推断 |
| 用户输入多行时右 prompt 不随之下移 | 🔍 语义推导 | 配置项语义 "with multi-line left prompt" |
| 左 prompt 只有 1 行时配置无效果 | ✅ 逻辑推导 + 配置文档 | 第一行 = 最后一行 |

## 五、完整的 REPL 迭代流程

[repl.rs: loop_iteration()](../crates/nu-cli/src/repl.rs#L713-L748) 中的时序：

```
start_time = Instant::now();
let config = &engine_state.get_config().clone();

// ─── 阶段 1: 渲染前求值（Nushell 控制） ───
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

// ─── 阶段 2: 渲染与输入读取（reedline 控制） ───
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

✅ **代码确定事实** — `update_prompt()` 使用的是 **子 Stack**：

```rust
&mut Stack::with_parent(stack_arc.clone())
```

这意味着：
1. Prompt 闭包中的变量修改不会影响主 REPL 栈
2. Prompt 闭包可以读取所有环境变量和配置
3. 求值过程中的副作用被隔离

## 六、Transient Prompt 机制

[make_transient_prompt()](../crates/nu-cli/src/prompt_update.rs#L129-L176) 在执行命令后替换原有 prompt，使历史记录更简洁。

**✅ 求值时机**：与普通 prompt 相同，在 `read_line()` 前求值。

**✅ 回退逻辑**：Transient prompt 各部分的回退是「有则用之，无则继承」：

```rust
if let Some(s) = get_prompt_string(TRANSIENT_PROMPT_COMMAND, ...) {
    nu_prompt.update_prompt_left(Some(s))
}
// 若 TRANSIENT_* 未设置，则保留原 prompt 的值
```

**✅ 相同的 `render_right_prompt_on_last_line` 配置** — transient prompt 使用与普通 prompt 相同的配置值。

## 七、环境变量清单

所有可配置的 prompt 环境变量定义在 [prompt_update.rs:12-26](../crates/nu-cli/src/prompt_update.rs#L12-L26)：

| 常量名 | 环境变量名 | 用途 | 类型 |
|--------|-----------|------|------|
| `PROMPT_COMMAND` | `PROMPT_COMMAND` | 左 prompt | String / Closure |
| `PROMPT_COMMAND_RIGHT` | `PROMPT_COMMAND_RIGHT` | 右 prompt（状态行） | String / Closure |
| `PROMPT_INDICATOR` | `PROMPT_INDICATOR` | 命令输入指示器 | String / Closure |
| `PROMPT_INDICATOR_VI_INSERT` | `PROMPT_INDICATOR_VI_INSERT` | VI 插入模式指示器 | String / Closure |
| `PROMPT_INDICATOR_VI_NORMAL` | `PROMPT_INDICATOR_VI_NORMAL` | VI 正常模式指示器 | String / Closure |
| `PROMPT_MULTILINE_INDICATOR` | `PROMPT_MULTILINE_INDICATOR` | 多行输入指示器 | String / Closure |
| `TRANSIENT_*` | `TRANSIENT_*` | 对应 transient 版本 | String / Closure |

## 八、容易混淆的点总结

### 8.1 求值 vs 渲染

| 方面 | 求值阶段（渲染前） | 渲染阶段（渲染时） |
|------|------------------|------------------|
| 控制方 | Nushell REPL（✅ 代码事实） | reedline 库（🔍 外部行为） |
| 时机 | `read_line()` 之前 | 终端需要重绘时 |
| 频率 | 每次 REPL 迭代一次 | 可能多次（按键、窗口大小变化等） |
| 行为 | 执行 Closure、收集字符串 | 读取缓存字符串 |
| 可修改状态 | 是（更新 NushellPrompt 缓存） | 否（只读） |
| Stack | 使用子 Stack，隔离副作用 | 不涉及 Stack |

### 8.2 回退层级

1. **Closure 执行失败** → 静默回退，不崩溃（✅ 代码事实）
2. **环境变量不存在** → 字段设为 `None`（✅ 代码事实）
3. **渲染时字段为 `None`** → 使用 reedline 默认值或硬编码默认值（✅ 代码事实）

### 8.3 两种「多行」的本质区别

| 对比项 | 场景 A：左 Prompt 本身多行 | 场景 B：用户输入多行 |
|--------|--------------------------|-------------------|
| 定义 | `PROMPT_COMMAND` 返回字符串含 `\n` | 语法未闭合，reedline 进入续行模式 |
| 控制权 | 完全由用户配置决定（✅ Nushell 端） | 由 reedline 解析器自动判定（🔍 外部） |
| 右 prompt 锚点 | 由 `render_right_prompt_on_last_line` 控制 | 锚点相对于左 prompt 行，不随输入行移动（🔍 推断） |
| Indicator | 每行都是左 prompt 的一部分 | 首行用 indicator，续行用 multiline indicator |
| 水平对齐 | reedline 按 `\n` 分行渲染 | multiline indicator 对齐到指示符列（🔍 推断） |
| 主要配置 | `render_right_prompt_on_last_line` | `PROMPT_MULTILINE_INDICATOR` |

### 8.4 render_right_prompt_on_last_line 生效条件

**口诀：只有左 Prompt 有多行，配置才有用。**

| 左 Prompt 行数 | 配置值 | 实际效果 |
|--------------|--------|---------|
| 1 行（普通 prompt） | false | 右 prompt 在第一行 = 最后一行，无区别 |
| 1 行（普通 prompt） | true | 右 prompt 在第一行 = 最后一行，无区别 |
| 2+ 行（复杂 prompt） | false | 右 prompt 在最顶部第一行 |
| 2+ 行（复杂 prompt） | true | 右 prompt 在最底部最后一行（与输入行同行） |

### 8.5 换行符转换的时机

- **`\n` → `\r\n` 转换在渲染时发生**，不在求值时（✅ 代码事实）
- 求值阶段 `left_prompt` 字段存储的是原始字符串（含 `\n`）
- `render_prompt_left/right()` 每次调用都执行替换
- `render_prompt_multiline_indicator()` **不做**此转换
- `render_prompt_indicator()` **也不做**此转换

### 8.6 multiline indicator 的常见误解

| 误解 | 正确理解 | 类型 |
|------|---------|------|
| multiline indicator 需要自己填空格对齐 | reedline 自动计算水平偏移（🔍 推断） | 🔍 外部行为 |
| multiline indicator 每行内容可以不同 | 一次 `read_line()` 期间是固定值（✅ 代码事实：方法无参数） | ✅ 代码事实 |
| 多行输入时右 prompt 会随之下移 | 右 prompt 锚定在左 prompt 的行，不随输入移动（🔍 推断） | 🔍 外部行为 |
| `render_right_prompt_on_last_line` 控制输入多行的位置 | 它只控制相对于左 prompt 行的位置（✅ 配置文档） | ✅ 语义确定 |

### 8.7 哪些是确定的，哪些是推断的

**✅ 可以在 Nushell 代码中验证的：**
- `NushellPrompt` 的 7 个字段和各 update 方法
- `get_prompt_string()` 的求值逻辑和 4 个回退点
- `render_prompt_*()` 方法的实现（含换行符转换）
- `update_prompt()` 在 REPL 中的调用时机和子 Stack 使用
- `right_prompt_on_last_line()` 直接返回配置值
- `render_prompt_multiline_indicator()` 无参数，默认值 `"::: "`

**🔍 需要参考 reedline 行为的：**
- reedline 何时调用哪个 render 方法
- 右 prompt 在终端中的具体像素/列坐标定位
- 多行输入的判定条件（括号匹配、续行符等）
- multiline indicator 的精确水平偏移计算
- 右 prompt 包含多行时的渲染细节
