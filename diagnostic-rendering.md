# Nushell 错误诊断渲染链路分析

## 一、整体架构概览

Nushell 的错误诊断系统建立在 **`miette`** 库之上。内部错误类型实现 `miette::Diagnostic` trait，由 miette 的 ReportHandler 负责格式化输出。

**核心渲染链路：**

```
错误产生 (ParseError / ShellError / ...)
        ↓
report_*_error() 入口函数
        ↓
CliError 包装器 — 延迟绑定 SourceCode
        ↓
miette ReportHandler (MietteHandler / ShortReportHandler / NarratableReportHandler)
        ↓
CliError.source_code() 回退到 &StateWorkingSet
        ↓
StateWorkingSet.read_span() — 全局 Span → 文件定位 → 局部偏移 → 源码行
        ↓
格式化输出到 stderr
```

**关键源码文件：**

| 文件 | 职责 |
|------|------|
| [cached_file.rs](crates/nu-protocol/src/engine/cached_file.rs) | CachedFile 定义 |
| [span.rs](crates/nu-protocol/src/span.rs) | Span / Spanned 定义 |
| [state_working_set.rs](crates/nu-protocol/src/engine/state_working_set.rs) | SourceCode 实现、get_span_contents |
| [engine_state.rs](crates/nu-protocol/src/engine/engine_state.rs) | 永久状态的文件存储与 span 内容查询 |
| [report_error.rs](crates/nu-protocol/src/errors/report_error.rs) | 错误报告入口、CliError、ErrorStyle 分派 |
| [parse_error.rs](crates/nu-protocol/src/errors/parse_error.rs) | ParseError 枚举、DidYouMean 类型 |
| [shell_error/mod.rs](crates/nu-protocol/src/errors/shell_error/mod.rs) | ShellError 枚举、ErrorSite、ErrorSource |
| [shell_error/generic.rs](crates/nu-protocol/src/errors/shell_error/generic.rs) | GenericError 结构体 |
| [labeled_error.rs](crates/nu-protocol/src/errors/labeled_error.rs) | LabeledError、ErrorLabel |
| [chained_error.rs](crates/nu-protocol/src/errors/chained_error.rs) | ChainedError 链式错误 |
| [short_handler.rs](crates/nu-protocol/src/errors/short_handler.rs) | ShortReportHandler |
| [did_you_mean.rs](crates/nu-protocol/src/did_you_mean.rs) | did_you_mean 函数 |
| [lev_distance.rs](crates/nu-protocol/src/lev_distance.rs) | 编辑距离算法（源自 rustc） |

---

## 二、缓存文件结构与全局偏移

### 2.1 CachedFile — 源码缓存单元

[cached_file.rs#L6-L16](crates/nu-protocol/src/engine/cached_file.rs#L6-L16) 定义了源码缓存的最小单元：

```rust
pub struct CachedFile {
    // Use Arcs of slice types for more compact representation (capacity less)
    // Could possibly become an `Arc<PathBuf>`
    /// The file name with which the code is associated (also includes REPL input)
    pub name: Arc<str>,
    /// Source code as raw bytes
    #[debug("[...]")]
    pub content: Arc<[u8]>,
    /// global span coordinates that are covered by this [`CachedFile`]
    pub covered_span: Span,
}
```

**字段说明：**

- **`name: Arc<str>`** — 使用 `Arc<str>` 而非 `String`，因为同一文件名可能被多个 CachedFile 共享（如重复 source 同一文件）。`Arc` 的引用计数语义避免了重复分配。对于 REPL 交互式输入，固定为 `"<cli>"`。

- **`content: Arc<[u8]>`** — 使用 `Arc<[u8]>` 而非 `Vec<u8>`。源码内容是不可变的（解析后不会被修改），`Arc` 切片在 clone 时只增加引用计数，不复制数据。这对频繁 clone 的 `StateDelta` 很重要。

- **`covered_span: Span`** — 该文件在**全局偏移空间**中占据的 `[start, end)` 范围。Nushell 将所有已解析文件的源码在逻辑上拼接成一个连续地址空间，`covered_span` 记录了本文件的字节范围。

### 2.2 全局偏移分配

文件被添加到工作集时，全局偏移按顺序递增分配。入口是 [state_working_set.rs#L338-L359](crates/nu-protocol/src/engine/state_working_set.rs#L338-L359)：

```rust
pub fn add_file(&mut self, filename: &str, contents: &[u8]) -> FileId {
    // First, look for the file to see if we already have it
    for (idx, cached_file) in self.files().enumerate() {
        if &*cached_file.name == filename && &*cached_file.content == contents {
            return FileId::new(idx);
        }
    }

    let next_span_start = self.next_span_start();
    let next_span_end = next_span_start + contents.len();

    let covered_span = Span::new(next_span_start, next_span_end);

    self.delta.files.push(CachedFile {
        name: filename.into(),     // &str → Arc<str>
        content: contents.into(),  // &[u8] → Arc<[u8]>
        covered_span,
    });

    FileId::new(self.num_files() - 1)
}
```

**`next_span_start()`** 的计算逻辑（[state_working_set.rs#L307-L314](crates/nu-protocol/src/engine/state_working_set.rs#L307-L314)）：

```rust
pub fn next_span_start(&self) -> usize {
    let permanent_span_start = self.permanent_state.next_span_start();

    if let Some(cached_file) = self.delta.files.last() {
        cached_file.covered_span.end   // delta 有文件 → 接在最后一个 delta 文件后面
    } else {
        permanent_span_start            // delta 无文件 → 接在永久状态后面
    }
}
```

而 `EngineState::next_span_start()`（[engine_state.rs#L939-L945](crates/nu-protocol/src/engine/engine_state.rs#L939-L945)）返回最后一个永久文件的 `covered_span.end`，若无文件则返回 0：

```rust
pub fn next_span_start(&self) -> usize {
    if let Some(cached_file) = self.files.last() {
        cached_file.covered_span.end
    } else {
        0
    }
}
```

**示意：**
```
EngineState.files:    [file_0] [file_1] [file_2]
                      0─────10 10────30 30────60   ← covered_span 范围
                                          next_span_start = 60

StateDelta.files:     [file_3] [file_4]
                      60────85 85───100
                              next_span_start = 100
```

### 2.3 文件存储的两层结构

CachedFile 存储在两个位置：

- **`EngineState.files: Vec<CachedFile>`** — 永久状态，存放已被 merge 的文件
- **`StateDelta.files: Vec<CachedFile>`** — 增量状态，存放本次解析新增的文件

`StateWorkingSet::files()` 将两者拼接为一个迭代器（[state_working_set.rs#L317-L319](crates/nu-protocol/src/engine/state_working_set.rs#L317-L319)）：

```rust
pub fn files(&self) -> impl Iterator<Item = &CachedFile> {
    self.permanent_state.files().chain(self.delta.files.iter())
}
```

`EngineState::files()` 返回 `impl DoubleEndedIterator + ExactSizeIterator`（[engine_state.rs#L947-L951](crates/nu-protocol/src/engine/engine_state.rs#L947-L951)），支持双向遍历和精确长度：

```rust
pub fn files(
    &self,
) -> impl DoubleEndedIterator<Item = &CachedFile> + ExactSizeIterator<Item = &CachedFile> {
    self.files.iter()
}
```

---

## 三、全局偏移读取：从 Span 到源码

### 3.1 get_span_contents — 纯内容读取

[state_working_set.rs#L396-L409](crates/nu-protocol/src/engine/state_working_set.rs#L396-L409) 将全局 Span 映射到源码字节切片：

```rust
pub fn get_span_contents(&self, span: Span) -> &[u8] {
    let permanent_end = self.permanent_state.next_span_start();
    // 优化：如果 span 在 delta 范围内，只搜索 delta
    if permanent_end <= span.start {
        for cached_file in &self.delta.files {
            if cached_file.covered_span.contains_span(span) {
                return &cached_file.content[span.start - cached_file.covered_span.start
                    ..span.end - cached_file.covered_span.start];
            }
        }
    }
    // if no files with span were found, fall back on permanent ones
    self.permanent_state.get_span_contents(span)
}
```

**关键分支逻辑：** `permanent_end <= span.start` 是一个优化。如果 span 起始位置在永久状态结束之后，它只可能属于 delta 中的文件，无需遍历永久状态。若该条件不成立，则直接委托给 `EngineState::get_span_contents`。

[engine_state.rs#L776-L786](crates/nu-protocol/src/engine/engine_state.rs#L776-L786) 遍历所有文件做相同匹配：

```rust
pub fn try_get_file_contents(&self, span: Span) -> Option<&[u8]> {
    self.files.iter().find_map(|file| {
        if file.covered_span.contains_span(span) {
            let start = span.start - file.covered_span.start;
            let end = span.end - file.covered_span.start;
            Some(&file.content[start..end])
        } else {
            None
        }
    })
}
```

`get_span_contents` 是 `try_get_file_contents` 的 `unwrap_or(&[0u8; 0])` 版本。

### 3.2 read_span — SourceCode trait 实现（渲染核心）

[`impl miette::SourceCode for &StateWorkingSet<'_>`](crates/nu-protocol/src/engine/state_working_set.rs#L1098-L1178) 是诊断渲染的源码提供核心。`read_span` 不只返回原始内容，还要附带行号、列号、上下文行等信息供 miette 格式化：

```rust
fn read_span<'b>(
    &'b self,
    span: &miette::SourceSpan,          // 要读取的 span (offset + len)
    context_lines_before: usize,         // 前面显示多少行上下文
    context_lines_after: usize,          // 后面显示多少行上下文
) -> Result<Box<dyn miette::SpanContents<'b> + 'b>, miette::MietteError> {
    // 遍历所有文件（永久 + delta）
    for cached_file in self.files() {
        let (filename, start, end) = (
            &cached_file.name,
            cached_file.covered_span.start,
            cached_file.covered_span.end,
        );
        // 判断：span 是否完全落在该文件范围内
        if span.offset() >= start && span.offset() + span.len() <= end {
            // 1. 转换为文件内局部偏移
            let local_span = (span.offset() - start, span.len()).into();

            // 2. 获取整个文件内容的字节切片
            let span_contents = self.get_span_contents(cached_file.covered_span);

            // 3. 委托给 miette 的 SpanContents::read_span 做行级切分
            let span_contents = span_contents.read_span(
                &local_span,
                context_lines_before,
                context_lines_after,
            )?;

            // 4. 将局部偏移重新翻译回全局偏移
            let content_span = span_contents.span();
            let retranslated = (content_span.offset() + start, content_span.len()).into();

            let data = span_contents.data();
            // 5. 按文件类型返回不同结果
            if &**filename == "<cli>" {
                return Ok(Box::new(miette::MietteSpanContents::new(
                    data, retranslated,
                    span_contents.line(), span_contents.column(),
                    span_contents.line_count(),
                )));
            } else {
                return Ok(Box::new(miette::MietteSpanContents::new_named(
                    (**filename).to_owned(), data, retranslated,
                    span_contents.line(), span_contents.column(),
                    span_contents.line_count(),
                )));
            }
        }
    }
    Err(miette::MietteError::OutOfBounds)
}
```

**渲染关键步骤：**

1. **文件定位** — `span.offset() >= start && span.offset() + span.len() <= end` 要求 span 完全落在某个 `covered_span` 内。注意这里是 `<=`（上界包含），因为 `covered_span.end` 本身是开区间。

2. **全局→局部** — `local_span = (span.offset() - start, span.len())` 将全局偏移转为文件内从 0 开始的偏移。

3. **行级切分** — 先用 `get_span_contents(covered_span)` 获取整个文件内容，再调用 `SpanContents::read_span(&local_span, ...)` 让 miette 自行计算行号、列号和上下文范围。miette 内部根据换行符 `\n` 的位置计算行号。

4. **局部→全局** — `content_span.offset() + start` 将 miette 返回的（可能因上下文扩展而调整过的）span 重新翻译回全局坐标系。

5. **命名差异** — `<cli>` 是交互式输入，没有文件名，用 `MietteSpanContents::new` 创建无名内容；其他文件用 `new_named` 附带文件名，miette 渲染时会显示文件名。

### 3.3 Span 与 miette 的转换

[span.rs#L384-L388](crates/nu-protocol/src/span.rs#L384-L388) 的转换将 Nushell 的 `{start, end}` 转为 miette 的 `{offset, len}`：

```rust
impl From<Span> for SourceSpan {
    fn from(s: Span) -> Self {
        Self::new(s.start.into(), s.end - s.start)
    }
}
```

`SourceSpan` 存储的是起始偏移 + 长度，而非起始 + 结束。

---

## 四、错误类型体系与标签机制

### 4.1 ParseError — 解析错误

[parse_error.rs](crates/nu-protocol/src/errors/parse_error.rs) 使用 `thiserror` + `miette` 过程宏声明式定义：

```rust
#[derive(Clone, Debug, Error, Diagnostic, Serialize, Deserialize, PartialEq)]
pub enum ParseError {
    #[error("Extra tokens in code.")]
    #[diagnostic(code(nu::parser::extra_tokens), help("Try removing them."))]
    ExtraTokens(#[label = "extra tokens"] Span),

    #[error("The '{op}' operator does not work on values of type '{unsupported}'.")]
    #[diagnostic(code(nu::parser::operator_unsupported_type))]
    OperatorUnsupportedType {
        op: &'static str,
        unsupported: Type,
        #[label = "does not support '{unsupported}'"]
        op_span: Span,
        #[label("{unsupported}")]
        unsupported_span: Span,
        #[help]
        help: Option<&'static str>,
    },
    // ...
}
```

**过程宏映射：**
- `#[error("...")]` → `Display` trait 实现（错误主消息）
- `#[diagnostic(code(...))]` → `Diagnostic::code()` 返回值
- `#[diagnostic(help("..."))]` → `Diagnostic::help()` 返回值
- `#[label = "..."]` 或 `#[label("...")]` → 生成 `Diagnostic::labels()` 中的 `LabeledSpan`
- `#[help]` 标注的字段 → 条件性 help（字段值非 None 时才显示）

**一个错误可以有多个标签**，如 `OperatorUnsupportedType` 同时标记 `op_span`（"does not support..."）和 `unsupported_span`（类型名），渲染时会显示两处下划线。

### 4.2 ShellError — 运行时错误

[shell_error/mod.rs](crates/nu-protocol/src/errors/shell_error/mod.rs) 结构与 ParseError 类似，包含数十种变体。其中 `ChainedError` 变体通过 `#[diagnostic(transparent)]` 完全委托给 `ChainedError`。

### 4.3 GenericError — 通用错误构建器

[generic.rs#L28-L53](crates/nu-protocol/src/errors/shell_error/generic.rs#L28-L53) 用 struct 字段手动实现 `Diagnostic`：

```rust
pub struct GenericError {
    /// The diagnostic code for this error.
    ///
    /// Defaults to [`DEFAULT_CODE`].
    /// Use [`with_code`](Self::with_code) to override it.
    pub code: Cow<'static, str>,

    /// A short, user-facing title for the error.
    pub error: Cow<'static, str>,        // Display 输出

    /// The message describing what went wrong.
    pub msg: Cow<'static, str>,          // 标签文本

    /// The error origin: either a user span or an internal Rust location.
    pub site: ErrorSite,                 // 错误位置

    /// Optional additional guidance for the user.
    pub help: Option<Cow<'static, str>>,

    /// Related errors that provide more context.
    pub inner: Vec<ShellError>,          // related 错误

    /// Optional error source.
    pub source: Option<ErrorSource>,     // diagnostic_source
}
```

**ErrorSite**（[shell_error/mod.rs#L1694-L1703](crates/nu-protocol/src/errors/shell_error/mod.rs#L1694-L1703)）决定标签位置和源码来源：

```rust
#[derive(Debug, Clone, Eq, PartialEq)]
pub enum ErrorSite {
    /// A span in user-provided Nushell code.
    Span(Span),

    /// A [`Location`] string from Rust code where the error originated.
    ///
    /// For usage with [`miette`] it's easier to hold a string here instead of a [`Location`].
    Location(String),
}
```

当 `site` 为 `ErrorSite::Span(span)` 时，`labels()` 返回一个标签指向该 span，`source_code()` 返回 `None`（回退到 StateWorkingSet）。当 `site` 为 `ErrorSite::Location(loc)` 时，`labels()` 的偏移为 0、长度为 `loc.len()`，`source_code()` 返回 `&loc`（Location 字符串本身就是源码）。

**ErrorSource**（[shell_error/mod.rs#L1717-L1720](crates/nu-protocol/src/errors/shell_error/mod.rs#L1717-L1720)）通过 `#[error(transparent)]` + `Diagnostic` 透明转发底层错误：

```rust
// TODO: implement further chaining than just one
#[derive(Debug, Error, Clone, Diagnostic)]
#[error(transparent)]
pub struct ErrorSource(Arc<dyn StdError + Send + Sync>);
```

`GenericError::diagnostic_source()` 返回 `self.source` 中的引用，miette 可以递归展示错误链。

### 4.4 LabeledError — 协议级错误

[labeled_error.rs#L15-L34](crates/nu-protocol/src/errors/labeled_error.rs#L15-L34) 用于与插件和脚本交互，所有字段都是可序列化的：

```rust
pub struct LabeledError {
    /// The main message for the error.
    pub msg: String,
    /// Labeled spans attached to the error, demonstrating to the user where the problem is.
    pub labels: Box<Vec<ErrorLabel>>,
    /// A unique machine- and search-friendly error code to associate to the error.
    pub code: Option<String>,
    /// A link to documentation about the error, used in conjunction with `code`
    pub url: Option<String>,
    /// Additional help for the error, usually a hint about what the user might try
    pub help: Option<String>,
    /// Errors that are related to or caused this error
    pub inner: Box<Vec<ShellError>>,
}
```

**ErrorLabel → LabeledSpan 转换**（[labeled_error.rs#L194-L202](crates/nu-protocol/src/errors/labeled_error.rs#L194-L202)）：

```rust
#[derive(Debug, Default, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct ErrorLabel {
    /// Text to show together with the span
    pub text: String,
    /// Span pointing at where the text references in the source
    pub span: Span,
}

impl From<ErrorLabel> for LabeledSpan {
    fn from(val: ErrorLabel) -> Self {
        LabeledSpan::new(
            (!val.text.is_empty()).then_some(val.text),  // 空文本 → 无标签
            val.span.start,                               // 全局偏移
            val.span.end - val.span.start,                // 长度
        )
    }
}
```

空标签文本时 `then_some` 返回 `None`，miette 只显示下划线不显示文字。

---

## 五、渲染流程

### 5.1 报告入口

| 函数 | 错误类型 | 默认 code | 额外逻辑 |
|------|---------|-----------|---------|
| `report_shell_error` | `ShellError` | `nu::shell::error` | 检查 `display_errors` 配置 |
| `report_parse_error` | `ParseError` | `nu::parser::error` | 无 |
| `report_compile_error` | `CompileError` | `nu::compile::error` | 无 |
| `report_shell_warning` | `ShellWarning` | `nu::shell::warning` | 去重（ReportLog / ReportMode） |
| `report_parse_warning` | `ParseWarning` | `nu::parser::warning` | 去重 |

所有入口最终调用 `report_error` 或 `report_warning`，流程相同：

```rust
fn report_error(stack, working_set, error, default_code) {
    let report = format!("Error: {:?}", CliError::new(stack, error, working_set, Some(default_code)));
    if writeln!(std::io::stderr(), "{report}").is_err() {
        let _ = writeln!(std::io::stdout(), "{report}");
    }
}
```

### 5.2 CliError — SourceCode 延迟绑定

[CliError](crates/nu-protocol/src/errors/report_error.rs#L23-L29) 在渲染时才绑定源码：

```rust
#[derive(Error)]
#[error("{diagnostic}")]
struct CliError<'src> {
    stack: Option<&'src Stack>,
    diagnostic: &'src dyn miette::Diagnostic,
    working_set: &'src StateWorkingSet<'src>,
    // error code to use if `diagnostic` doesn't provide one
    default_code: Option<&'static str>,
}
```

它实现 `Diagnostic` 时转发所有方法到 `self.diagnostic`，唯独重写 `source_code()`：

```rust
fn source_code(&self) -> Option<&dyn SourceCode> {
    if let Some(source_code) = self.diagnostic.source_code() {
        Some(source_code)       // 错误自己提供源码（如 ErrorSite::Location）
    } else {
        Some(&self.working_set) // 否则回退到 working_set
    }
}
```

**为什么需要延迟绑定？** 错误类型需要 `Clone` + `Serialize`（用于跨线程、跨插件传递），无法持有 `&StateWorkingSet` 引用。在报告阶段才将错误和源码关联起来。

### 5.3 Debug trait 触发渲染

`format!("Error: {:?}", CliError::new(...))` 触发 `CliError::fmt::fmt`，其中根据 `config.error_style` 分派处理器：

| ErrorStyle | 处理器 | 输出形式 |
|-----------|--------|---------|
| `Fancy`（默认） | `MietteHandler` | 彩色、Unicode、源码高亮 |
| `Plain` | `NarratableReportHandler` | 纯文本叙述 |
| `Short` | `ShortReportHandler` | 单行 |
| `Nested` | `MietteHandler` + `show_related_errors_as_nested()` | 相关错误嵌套展示 |

`MietteHandlerOpts` 的关键配置：
- `.rgb_colors(RgbColors::Never)` — 使用 ANSI 16 色，兼容终端主题
- `.color(ansi_support)` / `.unicode(ansi_support)` / `.terminal_links(ansi_support)` — 由 `config.use_ansi_coloring` 控制
- `.context_lines(error_lines)` — 由 `config.error_lines` 控制上下文行数
- `.with_cause_chain()` — 启用 `std::error::Error::source()` 链展示

### 5.4 ShortReportHandler

[short_handler.rs#L8-L56](crates/nu-protocol/src/errors/short_handler.rs#L8-L56) 是 Nushell 自定义的极简处理器：

```rust
fn render_report(
    &self,
    f: &mut fmt::Formatter<'_>,
    diagnostic: &dyn Diagnostic,
) -> fmt::Result {
    write!(f, "{}: ", diagnostic)?;

    if let Some(labels) = diagnostic.labels() {
        let mut labels = labels
            .into_iter()
            .filter_map(|span| span.label().map(String::from))
            .peekable();

        while let Some(label) = labels.next() {
            let end_char = if labels.peek().is_some() { ", " } else { " " };
            write!(f, "{}{}", label, end_char)?;
        }
    }

    if let Some(help) = diagnostic.help() {
        write!(f, "({})", help)?;
    }

    Ok(())
}
```

输出格式：`Error message: label1, label2 (help text)`

---

## 六、建议生成机制

### 6.1 help 字段

所有错误类型都可携带 `help` 字段。ParseError/ShellError 通过过程宏 `#[diagnostic(help("..."))]` 或 `#[help]` 标注字段；GenericError 通过 `.with_help(...)` 构建。

### 6.2 did_you_mean 函数

[did_you_mean.rs#L1-L17](crates/nu-protocol/src/did_you_mean.rs#L1-L17) 是对外暴露的拼写建议接口：

```rust
pub fn did_you_mean<'a, 'b, I, S>(possibilities: I, input: &'b str) -> Option<String>
where
    I: IntoIterator<Item = &'a S>,
    S: AsRef<str> + 'a + ?Sized,
{
    let possibilities: Vec<&str> = possibilities.into_iter().map(|s| s.as_ref()).collect();
    let suggestion =
        crate::lev_distance::find_best_match_for_name_with_substrings(&possibilities, input, None)
            .map(|s| s.to_string());
    if let Some(suggestion) = &suggestion
        && suggestion.len() == 1
        && !suggestion.eq_ignore_ascii_case(input)
    {
        return None;
    }
    suggestion
}
```

它调用 `find_best_match_for_name_with_substrings`，传入 `dist: None`（使用默认限制 `max(lookup.len(), 3) / 3`）。

### 6.3 lev_distance — 编辑距离算法（源自 rustc）

[lev_distance.rs](crates/nu-protocol/src/lev_distance.rs) 从 rustc 复制而来，提供三级匹配策略：

**[find_best_match_for_name_impl](crates/nu-protocol/src/lev_distance.rs#L134-L175) 的匹配优先级：**

1. **大小写不敏感精确匹配** — `candidates.iter().find(|c| c.to_uppercase() == lookup_uppercase)`，找到则立即返回。

2. **Levenshtein / 子串距离匹配** — 遍历所有候选，使用 `lev_distance` 或 `lev_distance_with_substrings` 计算距离。`dist` 默认值为 `max(lookup.len(), 3) / 3`（至少 1），找到更近的候选时动态收窄阈值。

3. **排序词匹配** — 如果上述都无结果，调用 `find_match_by_sorted_words`：将候选和输入按下划线 `_` 分割并排序后比较。例如 `"foo_bar"` 和 `"bar_foo"` 排序后相同，视为匹配。

**[lev_distance_with_substrings](crates/nu-protocol/src/lev_distance.rs#L72-L98) 的子串评分逻辑：**

在基础 Levenshtein 距离上，扣除长度差异的代价（`score = lev - len_diff`），然后分三种情况调整：

- 精确子串匹配（score == 0 且有长度差异且长度不太悬殊）→ 返回 1（不是完全匹配但很接近）
- 长度不太悬殊 → `score + len_diff.div_ceil(2)`（半价补偿长度差异）
- 长度悬殊（一个超过另一个的两倍）→ `score + len_diff`（完全加回长度差异，防止 "in" 匹配到 "shrink"）

### 6.4 DidYouMean 类型

[parse_error.rs#L670-L702](crates/nu-protocol/src/errors/parse_error.rs#L670-L702) 是 ParseError 内部的 newtype，接收字节切片输入：

```rust
#[derive(Clone, Debug, Serialize, Deserialize, PartialEq)]
pub struct DidYouMean(Option<String>);

fn did_you_mean_impl(possibilities_bytes: &[&[u8]], input_bytes: &[u8]) -> Option<String> {
    let input = from_utf8(input_bytes).ok()?;
    let possibilities = possibilities_bytes
        .iter()
        .map(|p| from_utf8(p))
        .collect::<Result<Vec<&str>, Utf8Error>>()
        .ok()?;
    did_you_mean(&possibilities, input)
}

impl DidYouMean {
    pub fn new(possibilities_bytes: &[&[u8]], input_bytes: &[u8]) -> DidYouMean {
        DidYouMean(did_you_mean_impl(possibilities_bytes, input_bytes))
    }
}

impl Display for DidYouMean {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        if let Some(suggestion) = &self.0 {
            write!(f, "Did you mean '{suggestion}'?")
        } else {
            write!(f, "")
        }
    }
}
```

**设计要点：**
- 接收 `&[&[u8]]` 和 `&[u8]`（字节切片），因为解析器内部用 `&[u8]` 表示标识符
- 内部做 UTF-8 转换后调用 `did_you_mean`
- `Display` 实现格式化为 `"Did you mean 'xxx'?"` 或空字符串
- 作为 ParseError 变体的字段嵌入标签，如 `ExpectedWithDidYouMean(&'static str, DidYouMean, #[label("expected {0}. {1}")] Span)`，`{1}` 渲染为 `"Did you mean 'xxx'?"` 或 `""`

---

## 七、相关错误展示

### 7.1 miette 的 related() 机制

`miette::Diagnostic::related()` 返回一个 Diagnostic 迭代器，miette 处理器会为每个 related error 递归渲染。这是多错误展示的核心接口。

### 7.2 GenericError.inner

[generic.rs#L196-L202](crates/nu-protocol/src/errors/shell_error/generic.rs#L196-L202) 通过 `inner: Vec<ShellError>` 存储相关错误：

```rust
fn related<'a>(&'a self) -> Option<Box<dyn Iterator<Item = &'a dyn Diagnostic> + 'a>> {
    match &self.inner.is_empty() {
        true => None,
        false => Some(Box::new(
            self.inner.iter().map(|err| err as &dyn Diagnostic),
        )),
    }
}
```

构建方式：`.with_inner(vec![error1, error2])`。`inner` 为空时返回 `None`。

### 7.3 LabeledError.inner

[labeled_error.rs#L413-L415](crates/nu-protocol/src/errors/labeled_error.rs#L413-L415) 始终暴露 `inner`：

```rust
fn related<'a>(&'a self) -> Option<Box<dyn Iterator<Item = &'a dyn Diagnostic> + 'a>> {
    Some(Box::new(self.inner.iter().map(|r| r as _)))
}
```

即使 `inner` 为空也返回 `Some`（迭代器为空），与 GenericError 的行为不同。

### 7.4 ChainedError — 链式错误

[chained_error.rs#L14-L19](crates/nu-protocol/src/errors/chained_error.rs#L14-L19) 是 `ShellError::ChainedError` 变体的内部类型：

```rust
pub struct ChainedError {
    first: bool,                // 是否为链中首个错误
    pub(crate) sources: Vec<ShellError>,
    span: Span,                 // 当前链节点的 span
}
```

**两种构建方式，两种行为模式：**

- [`ChainedError::new(source, span)`](crates/nu-protocol/src/errors/chained_error.rs#L32-L38) — `first = true`，行为与原始错误完全一致：所有 `Diagnostic` 方法转发到 `sources[0]`，就像一个透明包装。`Display` 输出 `sources[0]` 的消息。

- [`ChainedError::new_chained(sources, span)`](crates/nu-protocol/src/errors/chained_error.rs#L40-L46) — `first = false`，将 `sources`（整个 ChainedError）作为 `ShellError::ChainedError` 推入 `self.sources`。此时 `related()` 返回所有 sources，自身只提供一个标签 `"error happened when running this"`。`Display` 输出 `"oops"`。

**Diagnostic 方法转发的完整规则：**

| 方法 | `first = true` | `first = false` |
|------|---------------|-----------------|
| `related()` | 转发 `sources[0].related()` | `Some(self.sources.iter())` |
| `code()` | 转发 `sources[0].code()` | `Some("chained_error")` |
| `severity()` | 转发 `sources[0].severity()` | `None` |
| `help()` | 转发 `sources[0].help()` | `None` |
| `url()` | 转发 `sources[0].url()` | `None` |
| `labels()` | 转发 `sources[0].labels()` | 标签 `"error happened when running this"` + `self.span` |
| `source_code()` | 转发 `sources[0].source_code()` | `None` |
| `diagnostic_source()` | 转发 `sources[0].diagnostic_source()` | `None` |

### 7.5 into_chained — 构建入口

[shell_error/mod.rs#L1532-L1555](crates/nu-protocol/src/errors/shell_error/mod.rs#L1532-L1555) 是链式错误的构建入口：

```rust
/// Convert self error to a [`ShellError::ChainedError`] variant.
pub fn into_chained(self, span: Span) -> Self {
    Self::ChainedError(match self {
        // 已经是 ChainedError → 嵌套一层 (first = false)
        Self::ChainedError(inner) => ChainedError::new_chained(inner, span),
        // 普通错误 → 创建首个链节点 (first = true)
        other => {
            // If it's not already a chained error, it could have more errors below
            // it that we want to chain together
            let error = other.clone();
            let mut now = ChainedError::new(other, span);
            if let Some(related) = error.related() {
                let mapped = related
                    .map(|s| {
                        let shellerror: Self = Self::from_diagnostic(s);
                        shellerror
                    })
                    .collect::<Vec<_>>();
                if !mapped.is_empty() {
                    now.sources = [now.sources, mapped].concat();
                };
            }
            now
        }
    })
}
```

**递归链式构建：** 反复调用 `into_chained` 会层层嵌套。第一次调用创建 `ChainedError { first: true, sources: [original] }`，第二次调用将其包入新的 `ChainedError { first: false, sources: [ChainedError(first=true)] }`，第三次再包一层，依此类推。

**related 提取：** 首次构建时如果原错误有 `related()`，会将这些相关错误也展平到 `sources` 中，避免丢失。

### 7.6 ErrorSource 与 diagnostic_source

[shell_error/mod.rs#L1717-L1720](crates/nu-protocol/src/errors/shell_error/mod.rs#L1717-L1720) 通过 `Diagnostic::diagnostic_source()` 展示错误链：

```rust
// TODO: implement further chaining than just one
#[derive(Debug, Error, Clone, Diagnostic)]
#[error(transparent)]
pub struct ErrorSource(Arc<dyn StdError + Send + Sync>);
```

GenericError 的 `diagnostic_source()` 返回 `self.source` 中的引用。这与 `related()` 不同——`related()` 展示平行关系（"也发生了以下错误"），`diagnostic_source()` 展示因果关系（"由以下原因导致"）。miette 对两者有不同的渲染策略：related 会并列展示，diagnostic_source 会形成缩进链。

### 7.7 Nested 模式

`ErrorStyle::Nested` 使用 `MietteHandlerOpts::show_related_errors_as_nested().build()`，让 miette 将 related errors 以缩进嵌套形式展示，而非默认的并列平铺。

---

## 八、完整渲染示例

以 `ExtraTokens` 错误为例的完整流程：

**1. 错误产生**
```rust
return Err(ParseError::ExtraTokens(extra_span));
```

**2. 报告入口**
```rust
report_parse_error(stack, working_set, &parse_error);
```

**3. CliError 包装**
- `diagnostic` = `&ParseError::ExtraTokens`
- `working_set` = 当前 `StateWorkingSet`
- `default_code` = `"nu::parser::error"`

**4. Debug 格式化** — 读取配置，创建 `MietteHandler`

**5. miette 收集诊断信息**
- `diagnostic.code()` → `Some("nu::parser::extra_tokens")` （由 `#[diagnostic(code(...))]` 提供，覆盖 default_code）
- `diagnostic.to_string()` → `"Extra tokens in code."`
- `diagnostic.help()` → `Some("Try removing them.")`
- `diagnostic.labels()` → 一个 `LabeledSpan { offset: span.start, len: span.end - span.start, label: Some("extra tokens") }`
- `diagnostic.source_code()` → `None`，回退到 `working_set`

**6. 源码读取** — miette 调用 `working_set.read_span(span, context_lines_before, context_lines_after)`
- 遍历 `files()` 找到 `covered_span` 包含该 span 的 CachedFile
- 全局→局部偏移：`local_span = (span.offset() - file_start, span.len())`
- 获取文件全部内容，委托 `SpanContents::read_span` 计算行号/列号/上下文
- 局部→全局偏移：`retranslated = (content_span.offset() + file_start, content_span.len())`
- 文件名为 `"<cli>"` → 无名 `MietteSpanContents`

**7. 格式化输出**
```
Error: nu::parser::extra_tokens

  × Extra tokens in code.
   ╭─[entry #1:1:1]
 1 │ echo hello world extra
   ·              ─────┬────
   ·                   ╰── extra tokens
   ╰────
  help: Try removing them.
```

---

## 九、关键设计决策

### 9.1 Arc<str> / Arc<[u8]> 而非 String / Vec<u8>

CachedFile 使用 `Arc` 切片类型是为了：
- **clone 零拷贝** — `StateDelta` 频繁 clone，`Arc` 只增加引用计数
- **内存紧凑** — `Arc<str>` 比 `String` 少一个 capacity 字段
- **不可变语义** — 源码内容解析后不再修改，不需要 `Vec` 的可变性

### 9.2 全局 Span 的代价与收益

**收益：** 错误只需携带一个 `Span` 即可指向任意文件中的位置，无需额外存储文件引用。

**代价：** 渲染时需要 O(n) 遍历文件列表定位。优化措施：`get_span_contents` 中用 `permanent_end <= span.start` 快速判断只搜索 delta。

### 9.3 CliError 延迟 SourceCode

错误类型需要 `Clone` + `Serialize`（跨线程/插件传递），无法持有 `&StateWorkingSet`。在报告时才绑定源码，保持错误类型轻量且可序列化。

### 9.4 ErrorSite 双模式

`ErrorSite::Span` 用于用户可见错误，miette 可以在源码中高亮。`ErrorSite::Location` 用于内部错误，将 Rust 调用位置（文件:行号:列号）作为伪源码展示，方便开发者调试。
