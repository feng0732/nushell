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

### 2.3 多行场景分类：两种「多行」概念

在分析右 prompt 定位方式之前，必须先区分两种完全不同的「多行」场景，这是最容易混淆的地方：

| 场景 | 触发原因 | 涉及的核心方法 | 配置项 |
|------|---------|---------------|--------|
| **场景 A：左 Prompt 本身多行** | `PROMPT_COMMAND` 返回的字符串包含 `\n` | `render_prompt_left()` / `render_prompt_right()` | `render_right_prompt_on_last_line` |
| **场景 B：用户输入多行** | 未闭合括号、引号、或 `\` 换行 | `render_prompt_multiline_indicator()` | `PROMPT_MULTILINE_INDICATOR` 环境变量 |
| **场景 C：A + B 组合** | 左 prompt 多行 + 用户输入多行 | 上述所有方法 | 上述所有配置 |

### 2.4 场景 A：右 Prompt 在左 Prompt 多行时的定位

配置项 `render_right_prompt_on_last_line` **仅在左 Prompt 本身是多行时生效**。配置文档定义在 [doc_config.nu](file:///d:/fz/0601-2/solo-dogfeeding/code/71-nushell/crates/nu-utils/src/default_files/doc_config.nu#L506-L510)：

```
# render_right_prompt_on_last_line (bool): Right prompt position with multi-line left prompt.
# true: Right prompt appears on the last line of the left prompt.
# false: Right prompt appears on the first line.
# Default: false
```

- **配置定义**：[config/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/71-nushell/crates/nu-protocol/src/config/mod.rs#L74)
- **字段存储**：[prompt.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/71-nushell/crates/nu-cli/src/prompt.rs#L18) 中的 `render_right_prompt_on_last_line: bool`
- **渲染时查询**：[right_prompt_on_last_line()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-nushell/crates/nu-cli/src/prompt.rs#L158-L160)

**定位方式图解（左 Prompt 为两行的情况）：**

```
情况 1: render_right_prompt_on_last_line = false（默认值）
右 prompt 与左 prompt 的「第一行」对齐

    ┌──────────── 左 Prompt 第一行 ────────────┐    ┌─ 右 Prompt ─┐
    │  user@host  ~/projects/nushell (main)    │    │ 10:30:45 AM │
    ├──────────── 左 Prompt 第二行 ────────────┤    └─────────────┘
    │ ❯ echo "hello world"                     │
    └──────────────────────────────────────────┘


情况 2: render_right_prompt_on_last_line = true
右 prompt 与左 prompt 的「最后一行」对齐

    ┌──────────── 左 Prompt 第一行 ────────────┐
    │  user@host  ~/projects/nushell (main)    │
    ├──────────── 左 Prompt 第二行（最后一行）──┤    ┌─ 右 Prompt ─┐
    │ ❯ echo "hello world"                     │    │ 10:30:45 AM │
    └──────────────────────────────────────────┘    └─────────────┘
```

**reedline 内部的实现原理（推断，基于 trait 方法签名）：**

`Prompt` trait 提供了两个独立的方法，reedline 根据 `right_prompt_on_last_line()` 的返回值决定右 prompt 的垂直位置：

```rust
// NushellPrompt 作为 reedline::Prompt 的实现
pub trait Prompt {
    fn render_prompt_left(&self) -> Cow<'_, str>;    // 返回左 prompt 字符串（可能包含 \n）
    fn render_prompt_right(&self) -> Cow<'_, str>;   // 返回右 prompt 字符串
    fn right_prompt_on_last_line(&self) -> bool;     // 告诉 reedline：右 prompt 放哪一行
    // ... 其他方法
}
```

reedline 的渲染引擎逻辑：
1. 调用 `render_prompt_left()`，按 `\n` 分割得到 N 行左 prompt
2. 调用 `right_prompt_on_last_line()` 获取定位模式
3. 若返回 `false` → 右 prompt 放在行号 0（第一行）的最右侧
4. 若返回 `true` → 右 prompt 放在行号 N-1（最后一行）的最右侧
5. 用户输入内容始终追加在左 prompt 最后一行的末尾

### 2.5 换行符转换处理

在 `render_prompt_left()` 和 `render_prompt_right()` 中，都会将 `\n` 替换为 `\r\n`：

```rust
prompt_string.replace('\n', "\r\n").into()
```

这确保了：
1. 左 prompt 包含的显式换行（场景 A）在终端中正确回车换行
2. 右 prompt 如果包含换行，同样会被正确转换为多行右 prompt
3. 跨平台兼容（Windows 需要 `\r\n`，Unix 只需要 `\n`，但终端通常都接受 `\r\n`）

**重要：** 换行符转换发生在「渲染时」而非「求值时」，这意味着：
- 求值阶段存储在 `left_prompt: Option<String>` 中的是原始字符串（包含 `\n`）
- 每次 `render_prompt_*()` 调用时才进行替换（虽然替换结果相同，但这是 reedline 要求的接口返回格式）

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

## 四、多行表现（Multiline）完整分析

本章节完整分析三种多行场景下的布局表现。

### 4.1 场景 B：用户输入多行（Multiline Input）

当用户输入未闭合的括号、引号或使用 `\` 进行续行时，**reedline 解析器检测到语法未闭合**，自动进入多行输入模式。

#### 4.1.1 多行 Indicator 的求值

**渲染前**在 [update_prompt()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-nushell/crates/nu-cli/src/prompt_update.rs#L105-L106) 中求值（与其他 prompt 组件同时求值）：

```rust
let prompt_multiline_string =
    get_prompt_string(PROMPT_MULTILINE_INDICATOR, config, engine_state, stack);
```

**渲染时**在 [render_prompt_multiline_indicator()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-nushell/crates/nu-cli/src/prompt.rs#L134-L141) 中调用，默认值为 `"::: "`：

```rust
fn render_prompt_multiline_indicator(&self) -> Cow<'_, str> {
    let indicator = match &self.multiline_indicator {
        Some(indicator) => indicator.as_str(),
        None => "::: ",  // 硬编码默认回退值
    };
    indicator.to_string().into()
}
```

#### 4.1.2 单行输入 vs 多行输入的 Indicator 切换

**关键区别：** `render_prompt_indicator()` vs `render_prompt_multiline_indicator()`

| 方法 | 调用时机 | 典型返回值 |
|------|---------|-----------|
| `render_prompt_indicator(edit_mode)` | 第一行输入，且语法未进入多行时 | `"> "` 或 `": "`（VI insert） |
| `render_prompt_multiline_indicator()` | 第 2..=N 行输入，reedline 判定需要续行时 | `"::: "` 或用户自定义值 |

**触发多行模式的条件（由 reedline 内部判定）：**
1. 未闭合的双引号 `"` 或单引号 `'`
2. 未闭合的括号 `()`, `[]`, `{}`
3. 行末尾的反斜杠 `\` 续行符
4. 管道符 `|` 出现在行末

#### 4.1.3 右 Prompt 在用户输入多行时的定位

当**左 Prompt 是单行**，但用户输入进入多行时，右 Prompt 的定位行为：

```
配置: render_right_prompt_on_last_line = false（默认）
左 Prompt: 单行
用户输入: 多行

    ┌─ 左 Prompt ─┐
    │ /home/user> │ echo "line one           ┌─ 右 Prompt ─┐
    │             │                        │ │ 10:30:45 AM │
    │     :::     │   line two             │ └─────────────┘
    │     :::     │   line three"          │
    └─────────────┘                        └───────────────┘
                    ↑                          ↑
            右 prompt 位置不变，始终        因为左 prompt 只有一行，
            与左 prompt 的第一行对齐        所以 true/false 无区别
```

**结论：** 当左 Prompt 是单行时，`render_right_prompt_on_last_line` 对用户输入的多行模式**没有影响**，因为左 Prompt 只有一行，「第一行」和「最后一行」是同一行。

### 4.2 场景 C：左 Prompt 多行 + 用户输入多行（组合场景）

这是最复杂的场景，两种多行同时发生。下面图解两种配置下的差异：

#### 4.2.1 配置 render_right_prompt_on_last_line = false（默认）

```
左 Prompt: 2 行（用户信息行 + 指示符行）
右 Prompt: 对齐左 Prompt「第一行」
用户输入: 3 行（未闭合字符串）

    ┌──────────── 左 Prompt 第一行 ────────────┐    ┌─ 右 Prompt ─┐
    │  user@host  ~/projects (main)            │    │ 10:30:45 AM │
    ├──────────── 左 Prompt 第二行（最后一行）──┤    └─────────────┘
    │ ❯ echo "This is a very long string that   │
    │     spans across multiple                 │
    │     lines"                                │
    └───────────────────────────────────────────┘

行 1: 左 Prompt 两行 + 用户输入第 1 行
行 2: 无左 Prompt，只显示多行 indicator + 用户输入第 2 行
行 3: 无左 Prompt，只显示多行 indicator + 用户输入第 3 行
右 Prompt: 始终对齐左 Prompt 第一行（最顶部）
```

#### 4.2.2 配置 render_right_prompt_on_last_line = true

```
左 Prompt: 2 行
右 Prompt: 对齐左 Prompt「最后一行」（即指示符所在行）
用户输入: 3 行

    ┌──────────── 左 Prompt 第一行 ────────────┐
    │  user@host  ~/projects (main)            │
    ├──────────── 左 Prompt 第二行（最后一行）──┤    ┌─ 右 Prompt ─┐
    │ ❯ echo "This is a very long string that   │    │ 10:30:45 AM │
    │     spans across multiple                 │    └─────────────┘
    │     lines"                                │
    └───────────────────────────────────────────┘

行 1: 左 Prompt 第一行（无用户输入，无右 prompt）
行 2: 左 Prompt 第二行 + 用户输入第 1 行 + 右 Prompt
行 3: 多行 indicator + 用户输入第 2 行
行 4: 多行 indicator + 用户输入第 3 行
右 Prompt: 对齐左 Prompt 最后一行（与指示符和输入第一行同行）
```

#### 4.2.3 多行 Indicator 与左 Prompt 的对齐关系

在多行输入模式中，第 2..N 行的多行 indicator **并不继承左 Prompt 的宽度**，而是独立显示：

```
左 Prompt 宽度计算（最后一行）:
    "  user@host ~/projects (main)  \n❯ "
                                    ↑
                              这里是最后一行的起点，indicator 从这里开始对齐

实际上 reedline 做了宽度对齐：
    ❯ echo "hello          ← 左 prompt 最后一行的起始列: col = X
    ::: world              ← 多行 indicator 也从 col = X 开始
    ::: second line"       ← 多行 indicator 始终对齐到同一列
```

这意味着：
1. `render_prompt_multiline_indicator()` 返回的字符串只负责显示内容（如 `"::: "`）
2. 该字符串的**水平位置由 reedline 自动计算**，对齐到左 Prompt 最后一行的指示符列
3. 用户不需要自己在 multiline indicator 中添加填充空格

### 4.3 右 Prompt 本身多行（Right Prompt Contains Newlines）

右 Prompt 也可以包含换行（虽然不常见），此时的行为由 reedline 处理：

```
右 prompt 返回: "10:30:45 AM\nCPU: 23%"

    ┌─ 左 Prompt ─┐    ┌─ 右 Prompt 第 1 行 ─┐
    │ /home/user> │    │     10:30:45 AM     │
    │             │    ├─ 右 Prompt 第 2 行 ─┤
    │             │    │       CPU: 23%      │
    └─────────────┘    └──────────────────────┘
```

这种情况下：
- 右 Prompt 的行锚点仍然是 `right_prompt_on_last_line()` 决定的那一行（第一行或最后一行）
- 右 Prompt 的多行向「下方」延伸
- 如果右 Prompt 行数过多超过终端，可能被截断或溢出

### 4.4 布局叠加总结表

| 场景 | 左 Prompt 行数 | 用户输入行数 | `right_prompt_on_last_line` | 右 Prompt 锚点行 |
|------|--------------|------------|----------------------------|----------------|
| 单行 | 1 | 1 | false | 左 Prompt 第 1 行 |
| 单行 | 1 | 1 | true | 左 Prompt 第 1 行（同 false） |
| 左多行 | 2 | 1 | false | 左 Prompt 第 1 行（顶部） |
| 左多行 | 2 | 1 | true | 左 Prompt 第 2 行（底部） |
| 单行+输入多行 | 1 | 3 | false | 左 Prompt 第 1 行（顶部） |
| 单行+输入多行 | 1 | 3 | true | 左 Prompt 第 1 行（同 false） |
| 左多行+输入多行 | 2 | 3 | false | 左 Prompt 第 1 行（顶部） |
| 左多行+输入多行 | 2 | 3 | true | 左 Prompt 第 2 行（底部） |

### 4.5 换行符转换处理

在 `render_prompt_left()` 和 `render_prompt_right()` 中，会将 `\n` 替换为 `\r\n`：

```rust
prompt_string.replace('\n', "\r\n").into()
```

这确保了：
1. 左 prompt 包含的显式换行（场景 A）在终端中正确回车换行
2. 右 prompt 如果包含换行，同样会被正确转换为多行右 prompt（场景 4.3）
3. 跨平台兼容（Windows 需要 `\r\n`，Unix 只需要 `\n`，但终端通常都接受 `\r\n`）

**重要：** 换行符转换发生在「渲染时」而非「求值时」，这意味着：
- 求值阶段存储在 `left_prompt: Option<String>` 中的是原始字符串（包含 `\n`）
- 每次 `render_prompt_*()` 调用时才进行替换（虽然替换结果相同，但这是 reedline 要求的接口返回格式）
- `render_prompt_multiline_indicator()` **不做** 换行符转换（multiline indicator 不应包含换行）

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

### 8.4 两种「多行」的本质区别

这是最容易混淆的点，**必须明确区分：**

| 对比项 | 场景 A：左 Prompt 本身多行 | 场景 B：用户输入多行 |
|--------|--------------------------|-------------------|
| **定义** | `PROMPT_COMMAND` 返回的字符串包含 `\n` | 用户输入语法未闭合，reedline 进入续行模式 |
| **控制权** | 完全由用户配置决定 | 由 reedline 解析器自动判定 |
| **右 prompt 锚点** | 由 `render_right_prompt_on_last_line` 控制：第一行 or 最后一行 | **不受配置影响**，始终与左 Prompt 第一行/最后一行对齐（同左 Prompt 行数决定） |
| **Indicator** | 每行都有左 prompt 的完整内容 | 只有第一行有左 prompt + indicator，后续行只有 multiline indicator |
| **水平对齐** | 各行独立渲染，reedline 负责 `\n` 换行 | multiline indicator 对齐到左 prompt 最后一行的指示符列 |
| **配置项** | `render_right_prompt_on_last_line` | `PROMPT_MULTILINE_INDICATOR` |

### 8.5 render_right_prompt_on_last_line 生效条件

**口诀：只有左 Prompt 有多行，配置才有用。**

| 左 Prompt 行数 | 配置值 | 实际效果 |
|--------------|--------|---------|
| 1 行（普通 prompt） | false | 右 prompt 在第一行 = 最后一行，无区别 |
| 1 行（普通 prompt） | true | 右 prompt 在第一行 = 最后一行，无区别 |
| 2+ 行（复杂 prompt） | false | 右 prompt 在最顶部第一行 |
| 2+ 行（复杂 prompt） | true | 右 prompt 在最底部最后一行（与输入行同行） |

### 8.6 换行符转换的时机

- **`\n` → `\r\n` 转换在渲染时发生**，不在求值时
- 求值阶段 `left_prompt` 字段存储的是原始字符串（含 `\n`）
- `render_prompt_left/right()` 每次调用都执行替换（虽然结果可缓存，但接口要求每次返回 Cow）
- `render_prompt_multiline_indicator()` **不做**此转换（multiline indicator 理论上不应有换行）
- `render_prompt_indicator()` **也不做**此转换

### 8.7 multiline indicator 的常见误解

❌ **错误理解**：multiline indicator 需要自己填充空格来对齐到指示符列
✅ **正确理解**：reedline 自动计算水平偏移，multiline indicator 只需返回显示内容（如 `"::: "`），位置由 reedline 控制

❌ **错误理解**：multiline indicator 每行的内容可以不同
✅ **正确理解**：multiline indicator 在一次 read_line() 调用期间是固定值，所有续行显示相同的 indicator
