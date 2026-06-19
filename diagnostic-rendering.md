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

- **`name: Arc<str>`** — 使用 `Arc<str>` 而非 `String`，因为同一文件名可能被多个 CachedFile 共享（如重复 source 同一文件）。`Arc` 的引用计数语义避免了重复分配。文件名按**场景动态分配**（详见下表），并非固定值。

- **`content: Arc<[u8]>`** — 使用 `Arc<[u8]>` 而非 `Vec<u8>`。源码内容是不可变的（解析后不会被修改），`Arc` 切片在 clone 时只增加引用计数，不复制数据。这对频繁 clone 的 `StateDelta` 很重要。

- **`covered_span: Span`** — 该文件在**全局偏移空间**中占据的 `[start, end)` 范围。Nushell 将所有已解析文件的源码在逻辑上拼接成一个连续地址空间，`covered_span` 记录了本文件的字节范围。

### 2.2 文件名分配规则

`name` 字段并非固定值，而是由调用方在 `add_file(filename, contents)` 时传入。不同入口场景使用不同的命名约定，直接决定了诊断渲染时文件名框中的显示内容：

| 场景 | 实际调用位置 | name 值 | 说明 |
|------|------------|---------|------|
| **REPL 逐条输入** | [repl.rs#L114](crates/nu-cli/src/repl.rs#L114), [repl.rs#L1086](crates/nu-cli/src/repl.rs#L1086) | `"repl_entry #0"`、`"repl_entry #1"` … | 每条输入递增编号，`entry_num` 从 0 开始 |
| **`nu script.nu` 脚本执行** | [eval_file.rs#L205](crates/nu-cli/src/eval_file.rs#L205) | 脚本文件实际路径（经 `expand_to_real_path` 展开） | 如 `C:/dev/test.nu` |
| **`nu -c "cmd"` 命令行参数** | [eval_file.rs#L200](crates/nu-cli/src/eval_file.rs#L200) | `"<commandline>"` | 带尖括号的伪文件名 |
| **文件读取失败时构造伪源码** | [eval_file.rs#L54](crates/nu-cli/src/eval_file.rs#L54) | `"<commandline>"` | 将 `nu script.nu args…` 整体作为伪命令行 |
| **`parse()` 无文件名默认** | [parse_captures_compile.rs#L523](crates/nu-parser/src/parse_captures_compile.rs#L523) | `"source"` | `fname == None` 时的兜底 |
| **`use mod` 模块导入** | [parse_module.rs#L734](crates/nu-parser/src/parse_module.rs#L734) | 模块文件实际路径（经 `to_string_lossy`） | 如 `C:/dev/mymod.nu` |
| **Tab 补全器内部解析** | [complete.rs#L95](crates/nu-cli/src/commands/commandline/complete.rs#L95) | `"completer"` | 补全时的临时源码 |
| **环境变量伪文件** | [util.rs#L98](crates/nu-cli/src/util.rs#L98) | `"Host Environment Variables"` | 启动时收集环境变量的伪文件 |
| **banner 显示命令** | [repl.rs#L166](crates/nu-cli/src/repl.rs#L166), [repl.rs#L176](crates/nu-cli/src/repl.rs#L176) | `"show short banner"` / `"show_banner"` | 内部短命令 |

> **关于 `"<cli>"` 的澄清：** [state_working_set.rs#L1148](crates/nu-protocol/src/engine/state_working_set.rs#L1148) 中确实存在 `if &**filename == "<cli>"` 的特殊分支，但是**在当前版本全仓库中没有任何一处调用 `add_file("<cli>", …)`** —— 该分支属于历史保留代码，是一条实际上永不触发的死路径。当前 REPL 输入走的是 `"repl_entry #N"` 命名（会走到 `else` 分支，使用 `MietteSpanContents::new_named`，因此用户能看到 `repl_entry #N` 字样）。

### 2.3 全局偏移分配

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

### 2.4 文件存储的两层结构

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

5. **命名分支** — 代码中特殊判断 `if &**filename == "<cli>"` 时走无名 `MietteSpanContents::new`（不显示文件名），其他所有情况（包括 REPL 的 `"repl_entry #N"`、脚本路径、`"<commandline>"`、`"source"` 等）都走 `MietteSpanContents::new_named(filename, …)`，miette 渲染时会在文件名框中显示该名字。如前所述，`"<cli>"` 分支在当前版本中没有实际入口，用户几乎总是看到命名输出。

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

**GenericError 的 code 回退链（最完整）：**
1. `GenericError::new()` 时 `self.code` 默认设为 `DEFAULT_CODE = "nu::shell::error"`（[generic.rs#L15](crates/nu-protocol/src/errors/shell_error/generic.rs#L15)、[generic.rs#L74](crates/nu-protocol/src/errors/shell_error/generic.rs#L74)）
2. 可通过 `.with_code("nu::custom::code")` 覆盖 `self.code`
3. `GenericError::code()` 总是返回 `Some(Box::new(self.code.as_ref()))`（[generic.rs#L169-L171](crates/nu-protocol/src/errors/shell_error/generic.rs#L169-L171)），**从不返回 None**
4. 因此 CliError 的 `default_code` 对 GenericError 永远不会触发——GenericError 自己已经有了默认 code

对比 `LabeledError`：它的 `self.code` 是 `Option<String>`，若创建时未设置则返回 `None`，这才会触发 CliError 的 default_code 回退。

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
```

**LabeledError 的 code 转发**（[labeled_error.rs#L395-L396](crates/nu-protocol/src/errors/labeled_error.rs#L395-L396)）：
```rust
fn code<'a>(&'a self) -> Option<Box<dyn fmt::Display + 'a>> {
    self.code.as_ref().map(Box::new).map(|b| b as _)
}
```
由于 `self.code` 是 `Option<String>`，创建时若未传 code 则返回 `None`，触发 CliError 的 `default_code` 回退机制。这与 GenericError 不同——GenericError 总有默认 code。

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
| `report_experimental_option_warning` | `dyn Diagnostic` | `nu::experimental_option::warning` | 无 |
| `format_cli_error` | `dyn Diagnostic` | 调用方传入（可选 `None`） | 仅格式化不打印 |

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

它实现 `Diagnostic` 时，大部分方法透明转发到 `self.diagnostic`，但重写了两个关键方法：`code()`（补充默认值回退）和 `source_code()`（延迟绑定 working_set）：

```rust
fn code<'a>(&'a self) -> Option<Box<dyn Display + 'a>> {
    self.diagnostic.code().or_else(|| {
        self.default_code
            .map(|code| Box::new(code) as Box<dyn Display>)
    })
}

fn source_code(&self) -> Option<&dyn SourceCode> {
    if let Some(source_code) = self.diagnostic.source_code() {
        Some(source_code)       // 错误自己提供源码（如 ErrorSite::Location）
    } else {
        Some(&self.working_set) // 否则回退到 working_set
    }
}
```

**code() 回退逻辑（优先级从高到低）：**
1. 错误自身通过 `#[diagnostic(code(...))]` 或 `self.code` 字段提供的具体错误码（如 `nu::parser::extra_tokens`）
2. 入口函数传入的 `default_code`（如 `report_shell_error` 传入 `"nu::shell::error"`）
3. 若以上都为 `None`，则不显示错误码（极少发生）

**为什么需要延迟绑定？** 错误类型需要 `Clone` + `Serialize`（用于跨线程、跨插件传递），无法持有 `&StateWorkingSet` 引用。在报告阶段才将错误和源码关联起来。

### 5.2.1 各诊断包装器的 code() 转发

Nushell 的错误类型层层嵌套，每种包装器的 `code()` 行为不同，共同决定了最终输出的错误码：

| 包装器 | code() 行为 | 调用位置 |
|--------|------------|---------|
| `ShellError` / `ParseError` / `CompileError` | 绝大多数变体通过 `#[diagnostic(code(nu::*))]` 属性宏提供具体错误码，由 miette derive 自动生成返回 `Some(Box::new("nu::*"))` | [shell_error/mod.rs#L30](crates/nu-protocol/src/errors/shell_error/mod.rs#L30) |
| `GenericError` | 总是返回 `Some(Box::new(self.code.as_ref()))`，`self.code` 在 `new()` 时默认设为 `DEFAULT_CODE = "nu::shell::error"`，可通过 `with_code()` 覆盖 | [generic.rs#L15-L171](crates/nu-protocol/src/errors/shell_error/generic.rs#L15-L171) |
| `ChainedError::first=true` | 透明转发 `self.sources[0].code()`，和首元素行为一致 | [chained_error.rs#L64-L66](crates/nu-protocol/src/errors/chained_error.rs#L64-L66) |
| `ChainedError::first=false` | 返回固定字符串 `"chained_error"`，不再向下转发 | [chained_error.rs#L68](crates/nu-protocol/src/errors/chained_error.rs#L68) |
| `LabeledError` | `self.code.as_ref().map(Box::new)`，若创建时未传 `code` 则返回 `None`，触发 CliError 的 default_code 回退 | [labeled_error.rs#L395-L396](crates/nu-protocol/src/errors/labeled_error.rs#L395-L396) |
| `IoError` | 根据 `self.kind` 动态构造，如 `nu::shell::io::not_found`、`nu::shell::io::permission_denied` | [io.rs#L490-L530](crates/nu-protocol/src/errors/shell_error/io.rs#L490-L530) |
| `DnsError` | 转发 `self.kind.code()`，每种 DNS 错误有独立 code | [network.rs#L32-L33](crates/nu-protocol/src/errors/shell_error/network.rs#L32-L33) |

**关键结论：** `CliError.code()` 是整个链路的最后一关——它先问"错误自己有 code 吗？"，没有的话再用入口函数给的 `default_code` 兜底。这意味着：
- `ShellError::ExtraTokens`（带 `#[diagnostic(code(nu::parser::extra_tokens))]`）→ 显示 `nu::parser::extra_tokens`
- 未设置 `code` 的 `LabeledError` → 回退到入口 default_code（如 `nu::shell::error`）
- `ChainedError::first=false` → 显示固定 `chained_error`
- `GenericError::new()`（未 with_code）→ 显示 `nu::shell::error`

### 5.2.2 错误码生成完整路径

从错误产生到最终显示，错误码经过四层决策链：

```
┌─────────────────────────────────────────────────────────────┐
│  1. 报告入口 — 传入 default_code 兜底值                      │
│     report_shell_error   → "nu::shell::error"                │
│     report_parse_error   → "nu::parser::error"               │
│     report_compile_error → "nu::compile::error"              │
│     report_parse_warning → "nu::parser::warning"             │
│     ...                                                     │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  2. 错误类型自身 — 通过多种方式提供具体 code                  │
│                                                             │
│  a) #[diagnostic(code(...))] 过程宏（最常见）                │
│     ShellError::ExtraTokens → "nu::parser::extra_tokens"    │
│                                                             │
│  b) 字段存储 + 手动实现                                      │
│     GenericError.code  → 总是 Some(DEFAULT_CODE)             │
│     LabeledError.code  → Option<String>（可能 None）         │
│                                                             │
│  c) 动态构造                                                │
│     IoError → "nu::shell::io::not_found" 等                  │
│     DnsError → "nu::shell::network::dns::fail" 等           │
│                                                             │
│  d) 包装器转发                                              │
│     ChainedError::first=true  → 转发 sources[0].code()       │
│     ChainedError::first=false → 固定 "chained_error"         │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  3. CliError.code() — 优先级决策                            │
│     self.diagnostic.code()      ← 先试错误自身               │
│         .or_else(default_code)  ← 没有就用入口兜底           │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  4. miette 渲染 — 若 Some(code) 则显示在 Error: 后           │
│     Error: nu::parser::extra_tokens                         │
│            ↑ 就是这里                                       │
└─────────────────────────────────────────────────────────────┘
```

**设计意图：** 两层默认值机制（GenericError 内部 DEFAULT_CODE + CliError 入口 default_code）确保了**错误码永不缺失**——即使开发者忘了设置具体 code，用户也至少能看到 `nu::shell::error` 这样的分类码，便于搜索和问题定位。

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

**5. miette 收集诊断信息（经 CliError 转发）**
- `CliError.code()` → 先调 `ParseError.code()` 返回 `Some("nu::parser::extra_tokens")`（由 `#[diagnostic(code(...))]` 提供，不为 None，因此跳过 default_code 回退）
- `CliError.to_string()` → 转发 `diagnostic.to_string()` → `"Extra tokens in code."`
- `CliError.help()` → 转发 → `Some("Try removing them.")`
- `CliError.labels()` → 转发 → 一个 `LabeledSpan { offset: span.start, len: span.end - span.start, label: Some("extra tokens") }`
- `CliError.source_code()` → `diagnostic.source_code()` 返回 `None`，回退到 `&self.working_set`

**6. 源码读取** — miette 调用 `working_set.read_span(span, context_lines_before, context_lines_after)`
- 遍历 `files()` 找到 `covered_span` 包含该 span 的 CachedFile
- 全局→局部偏移：`local_span = (span.offset() - file_start, span.len())`
- 获取文件全部内容，委托 `SpanContents::read_span` 计算行号/列号/上下文
- 局部→全局偏移：`retranslated = (content_span.offset() + file_start, content_span.len())`
- 文件名为 `"repl_entry #1"` → 走 `new_named` 分支，携带文件名到 miette

**7. 格式化输出**
（REPL 环境中，文件名来自 `"repl_entry #1"`，miette 按传入名字原样显示；以下方括号中的文件名会随实际场景变化：`"<commandline>"` 会显示为 `<commandline>`，脚本路径会显示为完整文件路径）
```
Error: nu::parser::extra_tokens

  × Extra tokens in code.
   ╭─[repl_entry #1:1:1]
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
