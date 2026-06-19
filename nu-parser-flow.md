# Nushell Parser 流程分析：从源码到 AST

## 总体架构

Nushell 的解析链路分为三个核心阶段，形成 `源文本 → Token 流 → LiteBlock（语法节点）→ AST` 的管道：

```
Source bytes
    │
    ▼
┌─────────┐    ┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐
│  lex()   │───▶│ lite_parse()  │───▶│ parse_block()    │───▶│ compile_block()  │
│ 词法分析  │    │ 轻量语法分析   │    │ 语法 → AST 构建   │    │ IR 编译           │
└─────────┘    └──────────────┘    └──────────────────┘    └──────────────────┘
    Token[]        LiteBlock           Block                  IrBlock
```

入口函数是 [parse()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_captures_compile.rs#L512-L622)，它串联了完整管线：

1. **词法分析** `lex()` — 字节流 → Token 流
2. **轻量语法分析** `lite_parse()` — Token 流 → LiteBlock
3. **语法 → AST** `parse_block()` — LiteBlock → Block（AST 顶层容器）
4. **IR 编译** `compile_block()` — Block → IrBlock（仅无解析错误时执行）
5. **闭包捕获分析** `discover_captures_in_closure()` — 遍历 AST 计算变量捕获

---

## 第一层：词法分析（Lexing）

**核心文件**: [lex.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/lex.rs)

### Token 结构

```rust
pub struct Token {
    pub contents: TokenContents,  // 语义类别
    pub span: Span,               // 源码位置
}
```

[TokenContents](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/lex.rs#L4-L20) 枚举定义了词法层面的语义分类：

| TokenContents | 含义 | 示例 |
|---|---|---|
| `Item` | 通用项（标识符、字面量等） | `ls`, `42`, `"hello"`, `$var` |
| `Comment` | 行注释 | `# this is a comment` |
| `Pipe` | 管道符 | `\|` |
| `PipePipe` | 双管道符 | `\|\|` |
| `AssignmentOperator` | 赋值运算符 | `=`, `+=`, `-=`, `*=`, `/=`, `++=` |
| `Semicolon` | 分号 | `;` |
| `Eol` | 行尾 | `\n` |
| `OutGreaterThan` 等 | 重定向 | `out>`, `err>>`, `out+err>\|` 等 |

### 词法扫描流程

入口函数 [lex_internal()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/lex.rs#L538-L695) 对输入字节流逐字符扫描：

1. **单字符 Token**：`|`、`;`、`\n`、`#` 直接生成为对应 Token
2. **空白跳过**：空格、制表符等被忽略
3. **复合项**：其余情况调用 [lex_item()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/lex.rs#L87-L387)

### lex_item() 的工作机制

`lex_item()` 是词法层的核心，它从当前位置开始"吞噬"一个完整的 baseline token：

1. **字符串字面量**：遇到 `'`、`"`、`` ` `` 进入字符串模式，直到匹配的引号出现（双引号支持 `\` 转义）
2. **原始字符串**：遇到 `r#` 模式，调用 [lex_raw_string()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/lex.rs#L389-L448)，支持 `r#'...'#`、`r##'...'##` 等多前缀形式
3. **配对定界符**：`[`/`]`、`(`/`)`、`{`/`}` 跟踪嵌套层级（`block_level: Vec<BlockKind>`），只有当层级为空时才认为遇到终止符
4. **终止条件**：层级为空 + 遇到空白/`|`/`;` 或 `special_tokens` 中的字符时，token 结束
5. **重定向识别**：如 `o>`、`err>` 等重定向前缀后接 `|` 时特殊处理

**关键设计**：词法层不关心语法结构，只负责把字节流切成带语义标签的片段。它通过 `block_level` 跟踪嵌套来正确处理跨行表达式，但不理解"这是什么语句"。

### 词法层的错误记录边界

**错误载体**：`Option<ParseError>` — 通过函数返回值携带，**不访问 `StateWorkingSet`**。

函数签名：
```rust
pub fn lex(input: &[u8], ...) -> (Vec<Token>, Option<ParseError>)
fn lex_item(...) -> (Token, Option<ParseError>)
```

**累积方式**：只保留**第一个**错误。

[lex_internal()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/lex.rs#L538-L695) 中：
```rust
let (token, err) = lex_item(...);
if state.error.is_none() {  // 仅当当前无错时才更新
    state.error = err;
}
```

同样，`;` 处检查不完整管道时也用 `if !is_complete && state.error.is_none()` 保护，确保只记录第一个错误。

**错误类型**：
- `UnexpectedEof` — 未闭合的字符串或转义序列（文件末尾仍未闭合）
- `Unbalanced` — 不匹配的闭合定界符（如 `}` 对应不到 `{`）
- `ExtraTokens` — 管道不完整时遇到 `;`

**与下一层的边界**：词法层的错误由 [parse()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_captures_compile.rs#L535-L538) 注入到全局错误列表：
```rust
let (output, err) = lex(contents, new_span.start, &[], &[], false);
if let Some(err) = err {
    working_set.error(err)  // 注入到 StateWorkingSet.parse_errors
}
```

---

## 第二层：轻量语法分析（Lite Parsing）

**核心文件**: [lite_parser.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/lite_parser.rs)

### 数据结构层级

```
LiteBlock
  └── Vec<LitePipeline>         // 由分号/换行分隔的管道组
        └── Vec<LiteCommand>    // 由管道符连接的命令序列
              ├── pipe: Option<Span>        // 管道符位置
              ├── comments: Vec<Span>       // 附加注释
              ├── parts: Vec<Span>          // 命令各部分的 Span
              ├── redirection: Option<LiteRedirection>
              └── attribute_idx: Vec<usize> // 属性边界索引
```

[LiteCommand](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/lite_parser.rs#L62-L70) 是轻量语法层最核心的结构，它把 token 流中的 spans 收集成逻辑分组：

- `parts` 保存命令名和参数的 Span（指向源码中的位置，而非文本）
- `redirection` 保存重定向信息（到文件 / 到管道）
- `attribute_idx` 标记属性（`@attr`）与实际命令之间的边界

### lite_parse() 的状态机

[lite_parse()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/lite_parser.rs#L219-L520) 用三种模式（[Mode](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/lite_parser.rs#L212-L217)）处理 token 流：

| Mode | 触发条件 | 行为 |
|---|---|---|
| `Normal` | 默认 | 按标准规则组织命令、管道、重定向 |
| `Assignment` | 遇到 `AssignmentOperator` | 消费后续所有 token 直到 `;`/`Eol`，吸收管道符和重定向 |
| `Attribute` | 行首遇到 `@` 开头的 Item | 消费 token 直到 `;`/`Eol`，不处理管道/重定向 |

**Normal 模式下的关键转换规则**：

```
TokenContents::Pipe          → 当前命令入 pipeline，创建新命令
TokenContents::Semicolon     → 当前命令入 pipeline，pipeline 入 block
TokenContents::Eol           → 若前一个有效 token 是 Pipe 则续行，否则终结 pipeline
TokenContents::Item          → 添加到当前命令的 parts
TokenContents::Comment       → 附加到当前命令的 comments
TokenContents::OutGreaterThan 等 → 设置 file_redirection 期望态
```

**重定向处理**：遇到重定向 token 时，`lite_parse()` 进入一个"期望重定向目标"的中间态（`file_redirection = Some(...)`）。下一个 token 必须是 `Item`（文件路径），否则报错。重定向通过 [try_add_redirection()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/lite_parser.rs#L83-L138) 组织成 `LiteRedirection`，支持 `Single`（一个方向）和 `Separate`（stdout+stderr 分别重定向）两种形态。

**注释处理**：注释与命令的绑定遵循"前附"原则——若注释前有空行，则注释归属于下一个命令；若注释紧跟在命令后，则附加到当前命令。这通过 `curr_comment` 状态变量实现。

**续行机制**：当 `Eol` 之前有 `Pipe` token 时，`Eol` 不终结 pipeline，实现了跨行管道。`last_non_comment_token()` 函数跳过注释行来查找真正的管道符。

### 轻量语法层的错误记录边界

**错误载体**：`Option<ParseError>` — 通过函数返回值携带。

函数签名：
```rust
pub fn lite_parse(tokens: &[Token], working_set: &StateWorkingSet) -> (LiteBlock, Option<ParseError>)
```

注意：`working_set` 是 **不可变引用**（`&StateWorkingSet`），lite_parse **不修改**全局错误列表。它只用于读取（如查询内置命令信息等）。

**累积方式**：也是只保留**第一个**错误，使用 `Option::or()` 语义：
```rust
error = error.or(Some(ParseError::Expected("redirection target", token.span)));
error = error.or(Some(ParseError::ShellOrOr(token.span)));
```

如果已有错误（`error.is_some()`），新错误被丢弃。

**错误类型**：
- `Expected("redirection target", _)` — 重定向后缺少目标文件
- `ShellOrOr` — 遇到 `||`（shell 风格的逻辑或，nushell 不支持）
- `MultipleRedirections` — 同一命令出现多重重定向
- `UnexpectedEof("pipeline missing end")` — 管道末尾缺少后续命令

**恢复行为**：即使出错，也会把有问题的 span 作为普通 part 推入命令，保证 LiteBlock 结构完整。

**与下一层的边界**：lite_parse 的错误由 [parse_block()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_pipelines.rs#L118-L121) 注入到全局错误列表：
```rust
let (lite_block, err) = lite_parse(tokens, working_set);
if let Some(err) = err {
    working_set.error(err);  // 注入到 StateWorkingSet.parse_errors
}
```

---

## 第三层：语法 → AST 构建

**核心文件**: 
- [parse_pipelines.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_pipelines.rs) — Block/Pipeline 构建
- [parse_expressions.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_expressions.rs) — 表达式解析
- [parse_calls.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_calls.rs) — 命令调用解析
- [parser.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parser.rs) — 重导出枢纽

### AST 核心类型

```
Block (AST 顶层容器)
  ├── signature: Box<Signature>     // 块签名（闭包参数等）
  ├── pipelines: Vec<Pipeline>      // 管道列表
  ├── captures: Vec<(VarId, Span)>  // 闭包捕获的变量
  ├── ir_block: Option<IrBlock>     // 编译后的 IR
  └── span: Option<Span>

Pipeline
  └── elements: Vec<PipelineElement>
        ├── pipe: Option<Span>                  // 管道符位置
        ├── expr: Expression                     // 表达式
        └── redirection: Option<PipelineRedirection>

Expression
  ├── expr: Expr           // 表达式语义类别
  ├── span: Span           // 源码范围
  ├── span_id: SpanId      // 去重用
  └── ty: Type             // 推断的类型
```

[Expr](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-protocol/src/ast/expr.rs#L14-L58) 枚举是 AST 的语义核心，包含 40+ 种变体，从字面量（`Bool`、`Int`、`Float`、`String`）到复合结构（`BinaryOp`、`Call`、`List`、`Table`、`Record`、`Closure` 等）。

### parse_block() 主流程

[parse_block()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_pipelines.rs#L111-L183) 是 AST 构建的入口：

```
tokens → lite_parse() → LiteBlock
                           │
                ┌──────────┴──────────┐
                │ parse_def_predecl() │  预声明（允许同块内互引用）
                └──────────┬──────────┘
                           │
                ┌──────────┴──────────┐
                │  for each pipeline:  │
                │  parse_pipeline()    │
                └──────────┬──────────┘
                           │
                ┌──────────┴──────────┐
                │ $in 变量 Collect 包装 │  （非子表达式时）
                └──────────┬──────────┘
                           │
                ┌──────────┴──────────┐
                │ type_check()         │  输入输出类型检查
                └─────────────────────┘
```

**预声明机制**：`parse_def_predecl()` 在正式解析前扫描所有 pipeline，遇到 `def` 命令时提前注册声明到 `StateWorkingSet`，使得同块内的定义可以互相引用（如递归函数）。

### parse_pipeline() 的分叉逻辑

[parse_pipeline()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_pipelines.rs#L86-L109) 对单管道和多管道有不同处理：

- **多命令管道**（`commands.len() > 1`）：每个命令通过 `parse_pipeline_element()` 解析为 `PipelineElement`，非首元素若含 `$in` 变量则自动包装 `Collect` 表达式
- **单命令管道**：委托给 [parse_builtin_commands()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_expressions.rs#L1606-L1750)，因为单命令可能是关键字（`def`、`let`、`for` 等）需要特殊处理

### parse_builtin_commands() — 关键字分发器

[parse_builtin_commands()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_expressions.rs#L1606-L1750) 是语法层的"路由器"，根据命令名的首 token 分发到专门的处理函数：

| 关键字 | 处理函数 | 说明 |
|---|---|---|
| `def` | `parse_def()` | 函数定义 |
| `extern` | `parse_extern()` | 外部命令签名 |
| `export` | `parse_export_in_block()` | 导出声明 |
| `let` | `parse_let()` | 不可变变量绑定 |
| `const` | `parse_const()` | 常量声明 |
| `mut` | `parse_mut()` | 可变变量绑定 |
| `for` | `parse_for()` | for 循环 |
| `alias` | `parse_alias()` | 别名定义 |
| `module` | `parse_module()` | 模块定义 |
| `use` | `parse_use()` | 导入 |
| `where` | `parse_where()` | where 过滤 |
| 其他 | `parse_pipeline_element()` | 按普通表达式解析 |

带属性（`@attr`）的命令走 `parse_attribute_block()` 路径。

### parse_expression() — 表达式解析核心

[parse_expression()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_expressions.rs#L1435-L1604) 是最复杂的解析函数，处理优先级从高到低：

1. **环境简写**：`FOO=bar` 形式，循环消费后包装为 `with-env` 调用
2. **赋值表达式**：若 spans 中含赋值运算符，走 `parse_assignment_expression()`
3. **数学表达式**：若首 span 像"数学表达式"（`is_math_expression_like()`），走 `parse_math_expression()`
4. **命令调用**：否则走 `parse_call()`

### parse_math_expression() — 运算符优先级处理

[parse_math_expression()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_expressions.rs#L1205-L1433) 使用**优先级爬升法**（precedence climbing）：

- 维护一个 `expr_stack`，随着运算符优先级递增而增长
- 遇到更低或相同优先级时，折叠栈为 `BinaryOp` 节点
- `**`（幂运算）是右结合的，不折叠
- `not` 前缀运算符可叠加，产生嵌套的 `UnaryNot`

### parse_call() — 命令调用解析

[parse_call()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_calls.rs#L1290-L1340) 负责把 spans 解析为 `Expr::Call`：

1. **`^` 前缀** → 强制外部命令，走 `parse_external_call()`
2. **`%` 前缀** → 强制内置命令
3. **名称查找**：在 `StateWorkingSet` 中查找声明 `find_decl()`
4. **内置命令** → `parse_internal_call()`：按 Signature 解析参数（长标志、短标志、位置参数、展开参数）
5. **外部命令** → `parse_external_call()`：外部参数有特殊解析规则

[parse_internal_call()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_calls.rs#L882-L1288) 是最复杂的参数解析器，按 `Signature` 定义逐个消费参数：

- 长标志（`--flag`/`--flag=value`）
- 短标志（`-f`/`-abc` 组合）
- 位置参数（通过 `parse_multispan_value()` 按 `SyntaxShape` 解析）
- 展开参数（`...$list`）
- `--` 终止符（停止标志解析）

### parse_value() — 值解析分发

[parse_value()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_expressions.rs#L786-L935) 根据 `SyntaxShape` 和首字符分发：

- `$` → `parse_dollar_expr()`（变量/单元格路径）
- `(` → `parse_paren_expr()`（子表达式/分组）
- `{` → `parse_brace_expr()`（闭包/块/记录）
- `[` → 列表/表/签名
- `r#` → 原始字符串

对于 `SyntaxShape::Any`，使用**试错回溯**策略：依次尝试 `Binary`、`Range`、`Filesize`、`Duration`、`DateTime`、`Int`、`Number`、`String`，第一个无错的形状胜出。

### AST 构建层的错误记录边界

**错误载体**：`StateWorkingSet.parse_errors: Vec<ParseError>` — 全局累积列表，**所有 AST 层函数共享**。

全局状态定义在 [state_working_set.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-protocol/src/engine/state_working_set.rs)：
```rust
pub struct StateWorkingSet<'a> {
    // ...
    pub parse_errors: Vec<ParseError>,
    pub parse_warnings: Vec<ParseWarning>,
    pub compile_errors: Vec<CompileError>,
}
```

通过方法追加：
```rust
pub fn error(&mut self, parse_error: ParseError) {
    self.parse_errors.push(parse_error)
}
```

**累积方式**：**全部追加**，不丢弃。每个解析函数都接收 `&mut StateWorkingSet`，发现错误时调用 `working_set.error(err)` 追加后继续执行。

函数签名的典型模式：
```rust
pub fn parse_value(working_set: &mut StateWorkingSet, span: Span, shape: &SyntaxShape) -> Expression
pub fn parse_pipeline(working_set: &mut StateWorkingSet, pipeline: &LitePipeline) -> Pipeline
pub fn parse_block(working_set: &mut StateWorkingSet, ...) -> Block
```

**没有 `Result` 返回**——函数始终返回有效的 AST 节点，错误通过 side channel 传递。

---

## 嵌套结构中的递归解析与错误传播

Nushell 的复合结构（闭包、块、子表达式、列表、表、记录等）在解析时会触发**递归的 lex → lite_parse → parse_block 循环**，每一层循环都会经历自己的三层解析，并产生独立的错误处理路径。

### 嵌套结构的入口分发

所有嵌套结构从 [parse_value()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_expressions.rs#L804-L828) 的首字符分派开始：

```
首字符 $ → parse_dollar_expr()   （可能含嵌套子表达式）
首字符 ( → parse_paren_expr()    （子表达式 / Range 试探）
首字符 { → parse_brace_expr()    （闭包 / 块 / 记录 歧义和分发）
首字符 [ → parse_table_expression → parse_list_expression （列表/表）
         或 parse_signature （签名）
```

### 每一种嵌套结构的完整递归路径

#### 路径一：闭包 `{ |x| ... }`

**入口**：首字符 `{` → [parse_brace_expr()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_literals.rs#L518-L590) → `parse_closure_expression()`

完整流程：
```
parse_brace_expr()
  │
  │ ① 探测性 lex（只看结构，不处理错误）
  │   let (tokens, _) = lex(bytes, ...);   // 第二个返回值被丢弃 !
  │
  │ ② 根据 tokens 形状判断是闭包
  │   [Pipe, ..] → 肯定是闭包
  │   SyntaxShape::Any → 默认走闭包路径
  ▼
parse_closure_expression(working_set, shape, span)
  │
  │ ③ 提取内部字节（去掉首尾 {}）
  │   bytes = strip_curly_braces(bytes)
  │
  │ ④ 真正的 lex（处理错误并注入）
  │   let (output, err) = lex(source, start, ...);
  │   if let Some(err) = err {
  │       working_set.error(err);   ←─── 错误跨层注入点 A
  │   }
  │
  │ ⑤ enter_scope() 进入新作用域
  │
  │ ⑥ 参数签名解析 |x, y|
  │   如出错 → working_set.error(err)
  │
  │ ⑦ 递归进入 parse_block（再次经历完整的三层）
  ▼
parse_block(working_set, &tokens, ...)
  │
  │ ⑧ 第二层 lite_parse
  │   let (lite_block, err) = lite_parse(tokens, working_set);
  │   if let Some(err) = err {
  │       working_set.error(err);   ←─── 错误跨层注入点 B
  │   }
  │
  │ ⑨ 递归解析每个 pipeline/expression
  │   （可能包含更多嵌套结构）
  │
  │ ⑩ type_check（追加类型错误）
  ▼
返回 Block
  │
  │ ⑪ 闭包编译门槛（只看 parse_errors）
  │   if working_set.parse_errors.is_empty() {
  │       compile_block(working_set, &mut output);  ←─── 编译门槛 C
  │   }
  │
  │ ⑫ exit_scope()
  │
  │ ⑬ add_block() 存入块表
  ▼
Expr::Closure(block_id)
```

**关键特征**：`parse_brace_expr()` 的第一步 **lex 探测不处理错误**（`let (tokens, _) = lex(...)`），它只用 token 形状判断结构类型，错误完全丢弃。真正的 lex 在 `parse_closure_expression()` 中重新执行，那一次才处理错误。

#### 路径二：块表达式 `{ ... }`

**入口**：首字符 `{` → `parse_brace_expr()` → 当 shape 为 `SyntaxShape::Block` 时 → [parse_block_expression()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_expressions.rs#L384-L444)

与闭包的区别：
- **没有立即编译**（编译留给父 Block 的统一编译）
- 遇到 `|` 开头会报错（`Expected("block but found closure")`）
- 同样经历自己的 lex → parse_block 递归

#### 路径三：子表达式 `( expr )`

**入口**：首字符 `(` → [parse_paren_expr()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_literals.rs#L472-L516)

完整流程：
```
parse_paren_expr(working_set, span, shape)
  │
  │ ① 快照 A，先试探 Range
  │   let starting_error_count = working_set.parse_errors.len();
  │   if let Some(expr) = parse_range(working_set, span) {
  │       return expr;                 // Range 成功，直接返回
  │   }
  │   working_set.parse_errors.truncate(starting_error_count);  ←─── 回滚①
  │   // 注意：truncate 回滚的包括 parse_range 内部所有嵌套产生的注入错误
  │
  │ ② 按 shape 分流（Signature / ExternalSignature）
  │
  │ ③ 快照 B，试探 FullCellPath（含 parse_block 递归）
  │   let fcp_expr = parse_full_cell_path(working_set, None, span);
  │   // parse_full_cell_path 内部会：
  │   //   - lex 一次（注入词法错误）
  │   //   - 若 head 是 (expr)，则重新 lex + parse_block
  │   //       * parse_block 内又有 lite_parse 注入
  │   //       * type_check 错误
  │   // 所有这些都追加到 parse_errors 中
  │
  │ ④ 判断是否是"结构损坏"的子表达式
  │   if fcp_error_count > starting_error_count {
  │       let malformed_subexpr = 检查首个新增错误是否是 Unclosed(")") 或 Unbalanced
  │       if malformed_subexpr {
  │           working_set.parse_errors.truncate(starting_error_count);  ←─── 回滚②
  │           // 把结构损坏的 () 当作字符串插值或 glob 处理
  │           parse_glob_pattern(span) 或 parse_string_interpolation(span)
  │       } else {
  │           fcp_expr  // 保留其他类型的错误（非结构性的）
  │       }
  │   } else {
  │       fcp_expr  // 无错误，直接返回
  │   }
```

**关键特征**：`parse_paren_expr()` 有**两层试探**，每层试探都可能触发深层嵌套解析（里面可能经历 lex→lite_parse→parse_block→更深层递归），产生的错误全部追加到全局 `parse_errors`，然后通过 `truncate()` 一次性回滚到快照点。

回滚条件很严格：只有当首个新增错误**恰好是** `Unclosed(")")` 或 `Unbalanced("(", ")")` 时才回滚。其他类型错误（如变量未定义、类型不匹配等）说明 `()` 虽然语法结构正确但语义有错，不应被当作"格式错了的 glob"处理。

#### 路径四：列表和表 `[1, 2, 3]`

**入口**：首字符 `[` → [parse_table_expression()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_expressions.rs#L222-L347) → 检查是否为表结构，否则回退到 `parse_list_expression()`

完整流程：
```
parse_table_expression(working_set, span, list_element_shape)
  │
  │ ① 剥离 []，提取内部 span
  │
  │ ② lex（错误注入）
  │   let (tokens, err) = lex(source, ...);
  │   if let Some(err) = err {
  │       working_set.error(err);   ←─── 错误注入点 D
  │   }
  │
  │ ③ 判断是否为表格式：[[a b]; [1 2]; [3 4]]
  │   匹配：[first, second, rest @ ..]
  │          条件：first starts_with [ ; second == Semicolon ; !rest.is_empty()
  │
  │ ④ 不匹配 → parse_list_expression()（逐元素 parse_value）
  │   匹配   → parse_table_row() 解析表头和每行
  │
  │ ⑤ 表行解析中：
  │   - 错误行数不对 → working_set.error(MissingColumns / ExtraColumns)
  │   - 行不是列表形式 → working_set.error("Table item not list")
  │
  │ ⑥ 快照 C：表类型推断
  │   let errors = working_set.parse_errors.len();
  │   if parse_errors.len() == errors {
  │       // 无新错误 → 精确计算列类型
  │       let (ty, errs) = table_type(&head, &rows);
  │       working_set.parse_errors.extend(errs);
  │   } else {
  │       Type::table()  // 有错误 → 退化为泛型表类型
  │   }
```

**关键特征**：表解析使用**"有错误就退化"**策略——如果解析过程中已经有错误，那么列类型就直接用 `Type::table()`，不再做精确推断。

#### 路径五：记录 `{ a: 1, b: 2 }`

**入口**：首字符 `{` → `parse_brace_expr()` → 当探测到 `:` 分隔符或 shape 为 Any 且不是闭包时 → [parse_record()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_expressions.rs#L1795-L19xx)

完整流程：
```
parse_record(working_set, span)
  │
  │ ① 增量词法循环（与其他结构不同！）
  │   while !lex_state.input.is_empty() {
  │
  │       lex_n_tokens(... 1);  // 读 1 个 token（键）
  │       if Unbalanced("{","}") → extra_tokens，结束
  │
  │       lex_n_tokens(... 1);  // 读 1 个 token（冒号 :）
  │       lex_n_tokens(... 1);  // 读 1 个 token（值）
  │   }
  │
  │ ② 词法层错误处理
  │   unclosed { → working_set.error(Unclosed("}"))
  │   extra_tokens → working_set.error(ExtraTokensAfterClosingDelimiter)
  │   lex_state.error → working_set.error(err)
  │
  │ ③ 解析键值对（递归！）
  │   for each 键值对：
  │       普通对 → parse_value(working_set, value_span, &SyntaxShape::Any)
  │                ↑ 这里可能递归触发任何嵌套结构
  │
  │       Spread → parse_value(working_set, inner_span, &SyntaxShape::record())
  │                ↑ 这里可能递归解析另一个记录
```

**关键特征**：记录使用**增量 lex**（`lex_n_tokens` 每次只 lex 1 个 token）而非一次性 lex，这是因为记录中 `:` 分隔符和 `,` 分隔符的位置需要精确控制。循环的终止通过检查 `lex_state.error` 中的 `Unbalanced("{","}")` 来实现——当闭合 `}` 被过度消费时触发此错误。

---

## 试探解析（Speculative Parsing）的回滚边界

Nushell 的试探解析基于一个简单机制：**对 `parse_errors` 做快照 + `truncate` 回滚**。但这个简单机制在嵌套递归的环境中会产生复杂的交互。

### 三类试探解析模式

| 试探函数 | 快照粒度 | 回滚条件 | 失败时的行为 |
|---|---|---|---|
| `parse_value(SyntaxShape::Any)` | 每个 shape 尝试前快照 | 只在 `Expected`/`ExpectedWithStringMsg` 时回滚 | 其他类型错误直接保留（认为是确定性错误） |
| `parse_oneof(SyntaxShape::OneOf)` | 每个 shape 尝试前快照 | 无条件回滚（总是 truncate） | 保留走得最远的尝试的错误 |
| `parse_paren_expr()` | 两次快照（Range 前、FCP 前） | 仅结构性错误（Unclosed/Unbalanced）回滚 | 其他错误保留，返回 FCP 结果 |

### 回滚的边界与嵌套错误的保留

**核心问题**：当试探一个 shape 时，内部可能触发深层嵌套（例如解析 `Closure(Any)` 参数时会经历 lex→lite_parse→parse_block→...），产生多个层级的错误。这些错误全部追加到同一个全局 `Vec` 中，那么哪些会被回滚、哪些会保留？

**精确答案**：

1. **全部被回滚**——除非显式判断"某些错误不该回滚"。
   `truncate(starting_error_count)` 是一个绝对操作：它把数组截断到快照长度，**所有**在快照后追加的错误（无论来自词法注入、lite_parse 注入、深层递归解析、还是 type_check）都会被删除。

2. **`parse_value(Any)` 的选择性保留**：它不总是执行 `truncate`。执行 `truncate` 前提是 `new_errors[0]` 是 `Expected`/`ExpectedWithStringMsg` 类型。如果不是（例如是嵌套解析内部产生的 `Unclosed` 或 `UnexpectedEof`），则**完全不回滚**，直接返回有错误的结果。

3. **`parse_oneof` 的"最优猜测"恢复**：每次尝试后无条件 `truncate`，但在最后会把"走得最远"的那次尝试的错误**重新追加**回来。这样全局 `Vec` 最终只包含最优猜测的错误，而不是每次尝试累积的错误叠加。

### 为什么 Expected 类错误可以回溯，其他不行？

看 [parse_value() Any 分支](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_expressions.rs#L912-L923)：
```rust
match working_set.parse_errors.get(starting_error_count) {
    // Expected 类错误 → 回溯，继续尝试下一个形状
    Some(ParseError::Expected(_, _) | ParseError::ExpectedWithStringMsg(_, _)) => {
        working_set.parse_errors.truncate(starting_error_count);
        continue;
    }
    // 其他类型错误 → 认为是确定性错误，直接返回
    _ => return s,
}
```

**原理**：
- `Expected("integer", span)` 这类错误的含义是"在当前 shape 约束下，span 内容不符合预期"——但**换一个 shape 就可能正确**。它是"形状不匹配"信号而非"代码错误"信号。
- 而 `Unclosed("}", span)` 的含义是"代码写到这里少了一个 }"——**不管换什么 shape 解析，这个代码都是错的**。
- 更进一步：如果深层嵌套内部产生了非 Expected 类错误（例如在解析 Closure shape 时，闭包体内部有语法错误），说明代码确实有问题，应该把这些错误暴露给用户，而不是换个 shape 再试。

### 试探解析与深层递归的协作图

以 `parse_value(Any, span)` 解析形如 `{ a: (1 + ) }` 的代码为例：

```
parse_value(Any, span="{ a: (1 + ) }")
  │
  ├── 尝试 SyntaxShape::Binary
  │    快照 N = parse_errors.len()
  │    parse_value(Binary, span) → 失败
  │    错误 Expected("binary", span) 追加（位于 N）
  │    first error 是 Expected → truncate(N) 回滚  ←─ 干干净净
  │
  ├── 尝试 SyntaxShape::Range
  │    快照 N = parse_errors.len()
  │    parse_range(working_set, span) → 失败
  │    错误 Expected("at least one range bound set", span) 追加
  │    truncate(N) 回滚
  │
  ├── ...（Filesize, Duration, DateTime, Int, Number 都类似）
  │
  └── 尝试 SyntaxShape::String
       快照 N = parse_errors.len()
       parse_value(String, span)
         parse_brace_expr()
           探测 lex（不处理错误）
           决定走 parse_record()
             parse_record()
               增量 lex（3 个 token: `a`, `:`, `(1 + )`）
               parse_value(Any, value_span="(1 + )")
                 parse_paren_expr()
                   快照 N2 = parse_errors.len()
                   parse_range() 失败 → truncate(N2) 回滚
                   parse_full_cell_path()
                     lex("(1 + )", ...) → 产生 token，但不完整
                     parse_block(内部 tokens, ...)
                       lite_parse(tokens, ...) → 错误 Option
                         if let Some(err) = err → working_set.error(err)  追加 ①
                       parse_pipeline(...)
                         parse_pipeline_element(...)
                           parse_math_expression("1 + ", ...)
                             缺少右操作数 → garbage()
                             working_set.error(Expected("expression"))   追加 ②
                       type_check → 可能追加 ③
                   判断 fcp 新增错误
                     错误①是 lite_parse 注入的 Expected？
                     错误②是 math expr 注入的 Expected？
                     错误① start span > 起始 → 不是 Unclosed/Unbalanced
                     → 结构正确，不回滚（保留 ①②③）

       parse_errors 新增了 [①, ②, ③]
       first error ① 的类型 = ?
         如果 ① 是 Expected 类型 → truncate 回滚！继续下一个 shape？
           但已经没有下一个 shape 了（String 是 shapes 数组最后一个）
           → 最终走 Expected("any shape") + garbage()
         如果 ① 不是 Expected 类型（如 UnexpectedEof）
           → 直接返回 s，不做进一步尝试
```

**关键洞察**：当深层嵌套产生的错误被外层试探看到时，试探解析会根据**新增错误的第一个**的类型决定是否回溯。这个第一个错误可能来自：词法层注入、lite_parse 注入、或深层 AST 解析本身——它是**递归路径中最早出错的点**。

---

## 跨层注入、错误保留与编译门槛的全局配合

### 跨层注入点全景图

全局 `StateWorkingSet.parse_errors` 中的错误来源可以精确归类：

| 注入点 | 来源层 | 注入代码 | 产生场景 |
|---|---|---|---|
| **注入点 A** | 词法层 | 各嵌套结构入口：`parse_closure_expression` L676-678、`parse_block_expression` L411-413、`parse_table_expression` L248-250、`parse_record` L1870-1872、`parse_full_cell_path` L1046-1048、`parse_match_block_expression` L471-473 | 嵌套结构内部 lex 时检测到未闭合字符串/定界符 |
| **注入点 B** | 轻量语法层 | `parse_block()` L118-121（每次递归 parse_block 都会触发） | 嵌套块内部 lite_parse 时检测到重定向缺目标 / `||` / 管道不完整 |
| **注入点 C** | AST 层（作用域） | `working_set.error()` 直接调用 | 变量未定义、重复定义、参数约束违规、Signature 不匹配等语义错误 |
| **注入点 D** | AST 层（表达式） | `parse_math_expression`、`parse_internal_call` 等 | 缺少操作数、参数数量不对、类型不匹配等语法错误 |
| **注入点 E** | AST 层（类型检查） | `parse_block()` L177-180 `type_check::check_block_input_output` | Pipeline 输入输出类型不兼容 |

**关键理解**：每一次递归进入 `parse_block()`（解析闭包体、子表达式体、`do` 块体等），都会再次执行 `lite_parse` → 注入点 B 就会被触发一次。也就是说，在整个解析过程中，注入点 B 可能被执行 N 次（N = Block 的嵌套层数）。

### 编译门槛的层级分布与条件

编译门槛（是否执行 `compile_block`）的判断条件是：**全局 `parse_errors.is_empty()`**。

需要特别注意：**所有编译门槛都检查同一个全局变量**——不是"当前块有没有错误"，而是"整个解析过程中累计的所有错误都为空"。

这意味着：
- 顶层块有 1 个错误 → 所有子闭包**都不编译**
- 某个嵌套很深的闭包内部有 1 个错误 → 顶层块和所有兄弟闭包**都不编译**

让我们看所有编译触发点的行为：

| 触发位置 | 调用形式 | 调用方是否检查 | `compile_block` 内部检查 | 行为 |
|---|---|---|---|---|
| [parse()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_captures_compile.rs#L546-L548) 顶层 | `compile_block(working_set, &mut block)` | **是**：`if parse_errors.is_empty()` | 是 | 双重保险，正确 |
| [parse_closure_expression()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_expressions.rs#L767-L769) | `compile_block(working_set, &mut block)` | **是**：`if parse_errors.is_empty()` | 是 | 双重保险，正确 |
| [parse_def.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_def.rs#L495) 函数体 | `compile_block_with_id(working_set, block_id)` | **否**：直接调用 | 是 | 依赖内部检查，有错误时只打 log |
| [parse_module.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_module.rs#L449) 模块体 | `compile_block_with_id(working_set, block_id)` | **否**：直接调用 | 是 | 同上 |
| [parse_signatures.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_signatures.rs#L507) RowCondition | `compile_block(working_set, &mut block)` | **否**：直接调用 | 是 | 同上 |
| [parse_source.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_source.rs#L130-L132) source 命令 | `compile_block(working_set, block_mut)` | **间接**：检查 `ir_block.is_none()`（暗示可能没编译） | 是 | 有额外判断 |

**一个重要的实际效果**：`parse_def.rs` 和 `parse_module.rs` 等入口**不检查** `parse_errors` 就调用 `compile_block_with_id`。但是 `compile_block_with_id` 内部会做同样的检查，如果有错误就 `return`（并打 error log）。从功能角度是等价的，只是 log 中会出现 `compile_block_with_id called with parse errors`——这被注释认为"可能是 parser 的一个 bug，但不会致命"。

### 编译门槛与试探回溯的交互

试探回溯通过 `truncate` 删除错误，**可能让本来不为空的 `parse_errors` 重新变为空**。这个顺序很重要。

看这个场景：
```
初始：parse_errors = []  ← 空
  │
  ▼
parse_value(Any, span)
  尝试 Int：快照 N=0
    parse_value(Int, "(1+2)")
      parse_paren_expr()
        parse_full_cell_path()
          lex 出错 → working_set.error(Unclosed(")"))  追加 ①
          parse_block()
            lite_parse 出错 → working_set.error(Expected(...))  追加 ②
        parse_errors = [①, ②]
      first error ① = Unclosed，不是 Expected
      → 不回溯，直接返回这个结果（Int 尝试"失败"但保留错误）
    注意：此时 parse_errors != [], 因为 first error 不是 Expected
    这个分支实际上立即 return s，不继续尝试其他 shape
  parse_errors = [①, ②]
  返回 s（带错误）
  │
  ▼
parse_closure_expression 末尾编译检查：
  if parse_errors.is_empty() → FALSE（有 ① ②）
  → 不编译
```

如果场景变成 first error 是 Expected：
```
初始：parse_errors = []
  │
  ▼
parse_value(Any, span)
  尝试 Int：快照 N=0
    parse_value(Int, "0x123")  ← 十六进制 Int 识别不出来
      working_set.error(Expected("integer"))  追加 ①
    first error ① = Expected("integer")
    → truncate(0) 回滚！
  parse_errors = []  ← 回到空！
  │
  继续尝试 Number：
    快照 N=0
    parse_value(Number, "0x123")
      working_set.error(Expected("number"))  追加 ①
    → truncate(0) 回滚！
  parse_errors = []
  ...
  │
  最终尝试 String 成功
  parse_errors = []  ← 仍然为空
  │
  ▼
parse_closure_expression 末尾编译检查：
  if parse_errors.is_empty() → TRUE
  → 编译！✓
```

**顺序依赖**：回溯必须发生在编译门槛检查之前。由于回溯发生在 `parse_value` / `parse_oneof` / `parse_paren_expr` 内部，这些都是**解析阶段**的操作，而编译发生在**解析完成之后**（parse_block 返回后），所以顺序天然满足。

### 全局机制的全景时序图

```
parse() 顶层入口
  │
  │ parse_errors 初始: []
  │
  ├─ ① lex(顶层) → err: Option
  │      if Some → working_set.error(err)  ←─ 词法层顶层注入
  │
  ├─ ② parse_block(顶层 tokens)
  │      │
  │      ├─ lite_parse(tokens) → err: Option
  │      │      if Some → working_set.error(err)  ←─ B (顶层)
  │      │
  │      ├─ parse_def_predecl()
  │      │
  │      ├─ for each pipeline:
  │      │      parse_pipeline()
  │      │        parse_pipeline_element()
  │      │          parse_expression()
  │      │            parse_call()
  │      │              parse_internal_call()
  │      │                parse_value(Any)  ←─ 试错回溯 1
  │      │                parse_value(Closure)
  │      │                  parse_brace_expr()  ←─ 探测性 lex (丢弃错误)
  │      │                    parse_closure_expression()
  │      │                      │
  │      │                      ├─ lex(closure body) → err
  │      │                      │   if Some → error(err)  ←─ A (闭包1)
  │      │                      │
  │      │                      ├─ parse_block(closure tokens)
  │      │                      │     │
  │      │                      │     ├─ lite_parse → err
  │      │                      │     │   if Some → error(err)  ←─ B (闭包1)
  │      │                      │     │
  │      │                      │     ├─ for pipeline in closure:
  │      │                      │     │    parse_pipeline()
  │      │                      │     │      parse_expression()
  │      │                      │     │        parse_value(Any)
  │      │                      │     │          parse_paren_expr() ← 试错回溯 2
  │      │                      │     │            truncate ←─ 删除 N3..N 的所有错误
  │      │                      │     │
  │      │                      │     └─ type_check → errors
  │                      │              working_set.parse_errors.extend(errors)  ←─ E
  │                      │
  │                      ├─ 闭包编译门槛:
  │                      │    if parse_errors.is_empty() {
  │                      │        compile_block()  ← 内部再次检查 is_empty()
  │                      │    }
  │                      │
  │                      └─ 返回 Expr::Closure(block_id)
  │
  ├─ ③ 顶层编译门槛:
  │      if parse_errors.is_empty() {
  │          compile_block(working_set, &mut output)
  │      }
  │
  ├─ ④ discover_captures_in_closure() 计算捕获
  │      (发生错误也 working_set.error(err) 追加)
  │
  └─ ⑤ 返回 Arc<Block>
      最终 parse_errors 在 StateWorkingSet 中供上层使用
```

---

## IR 编译的触发时序与全局状态交互

### compile_block() / compile_block_with_id() 的自我保护

[compile_block()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_captures_compile.rs#L11-L26) 和 [compile_block_with_id()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_captures_compile.rs#L28-L45) 自身都有同一道防线：

```rust
pub fn compile_block(working_set: &mut StateWorkingSet<'_>, block: &mut Block) {
    if !working_set.parse_errors.is_empty() {
        log::error!("compile_block called with parse errors");
        return;   // 不编译，Block.ir_block 保持 None
    }
    match nu_engine::compile(working_set, block) {
        Ok(ir_block) => { block.ir_block = Some(ir_block); }
        Err(err) => working_set.compile_errors.push(err),
    }
}

pub fn compile_block_with_id(working_set: &mut StateWorkingSet<'_>, block_id: BlockId) {
    if !working_set.parse_errors.is_empty() {
        log::error!("compile_block_with_id called with parse errors");
        return;   // 不编译，Block.ir_block 保持 None
    }
    match nu_engine::compile(working_set, working_set.get_block(block_id)) {
        Ok(ir_block) => {
            working_set.get_block_mut(block_id).ir_block = Some(ir_block);
        }
        Err(err) => working_set.compile_errors.push(err),
    }
}
```

**关键理解**：
- 两个函数检查的都是 `parse_errors`（解析错误），不是 `compile_errors`
- 即使解析成功，编译也可能产生 `compile_errors`——这是独立通道
- 编译成功时 `block.ir_block = Some(ir_block)`；编译失败或跳过时 `ir_block` 保持 `None`
- **没有回退机制**：一旦 `ir_block` 被设为 `Some(...)`，永远不会被清回 `None`

### 所有编译触发点的完整清单

按在 `parse()` → `parse_block()` 递归中的执行先后排列：

| 序号 | 触发位置 | 调用形式 | 调用方是否检查 | 编译对象 | 何时执行 |
|---|---|---|---|---|---|
| ① | `parse_closure_expression()` [L767-769](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_expressions.rs#L767-L769) | `compile_block(working_set, &mut output)` | **是**：`if parse_errors.is_empty()` | 闭包体的 `Block`（在 `add_block` 之前） | parse_block 递归中 |
| ② | `parse_expression()` with-env 简写 [L1566](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_expressions.rs#L1566) | `compile_block(working_set, &mut block)` | **否**：直接调用 | 环境简写包装的临时 `Block` | parse_block 递归中 |
| ③ | `parse_def()` 函数体 [L495](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_def.rs#L495) | `compile_block_with_id(working_set, *block_id)` | **否**：直接调用 | `def` 的闭包体 Block（已 `add_block`） | parse_block 递归中 |
| ④ | `parse_module()` export-env 块 [L449](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_module.rs#L449) | `compile_block_with_id(working_set, block_id)` | **否**：直接调用 | export-env 的 Block（已 `add_block`） | parse_block 递归中 |
| ⑤ | `parse_row_condition()` [L507](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_signatures.rs#L507) | `compile_block(working_set, &mut block)` | **否**：直接调用 | RowCondition 的 `Block`（在 `add_block` 之前） | parse_block 递归中 |
| ⑥ | `parse_source()` source 命令 [L130-132](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_source.rs#L130-L132) | `compile_block(working_set, block_mut)` | **间接**：`if block.ir_block.is_none()` | source 进来的文件 Block | parse_block 递归中 |
| ⑦ | `parse()` 顶层 [L546-548](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_captures_compile.rs#L546-L548) | `compile_block(working_set, Arc::make_mut(&mut output))` | **是**：`if parse_errors.is_empty()` | 顶层 `Block` | parse_block 返回后 |

### 编译触发的时序图

以解析 `def foo [] { 1 + 2 }; let x = { 3 + 4 }` 为例：

```
parse() 顶层入口
  │
  ├─ lex(顶层) → tokens
  │   如有词法错误 → working_set.error(err)    parse_errors: [E0?]
  │
  ├─ parse_block(顶层 tokens)
  │     │
  │     ├─ lite_parse(tokens) → 如有错 → error(err)   parse_errors: [..., E1?]
  │     │
  │     ├─ parse_def_predecl()  注册 foo 的声明
  │     │
  │     ├─ pipeline 1: "def foo [] { 1 + 2 }"
  │     │    parse_builtin_commands() → parse_def()
  │     │      parse_internal_call() → 解析参数（签名 + 闭包体）
  │     │        参数2: 闭包 { 1 + 2 }
  │     │          parse_closure_expression()
  │     │            lex("{ 1 + 2 }")  → 如有错 → error(err)
  │     │            parse_block(闭包 tokens)
  │     │              lite_parse → 如有错 → error(err)
  │     │              parse_pipeline → parse_math_expression → 1 + 2 → 成功
  │     │            返回 Block (无错误)
  │     │            ┌─────────────────────────────────────────┐
  │     │            │ ① compile_block(闭包 Block)             │ ← 此时检查 parse_errors
  │     │            │    parse_errors.is_empty() = true       │    若为空，编译成功
  │     │            │    → Block.ir_block = Some(IrBlock)     │    ir_block 已写入
  │     │            │    → add_block() 存入块表               │
  │     │            └─────────────────────────────────────────┘
  │     │          返回 Expr::Closure(block_id)
  │     │
  │     │      parse_internal_call 返回 Call
  │     │      Call.positional[2] = Expr::Closure(block_id)  ← 闭包已带 IrBlock
  │     │      ┌──────────────────────────────────────────────┐
  │     │      │ ③ compile_block_with_id(block_id)            │ ← def 体再次编译
  │     │      │    parse_errors.is_empty() = true            │    闭包 Block 已有 ir_block
  │     │      │    → nu_engine::compile 再次执行              │    覆盖写入 ir_block
  │     │      │    → Block.ir_block = Some(new_IrBlock)      │
  │     │      │    → 设置 signature                          │
  │     │      └──────────────────────────────────────────────┘
  │     │
  │     ├─ pipeline 2: "let x = { 3 + 4 }"
  │     │    parse_builtin_commands() → parse_let()
  │     │      parse_value(Closure) → 闭包 { 3 + 4 }
  │     │        parse_closure_expression()
  │     │          lex → parse_block → 成功
  │     │          ┌─────────────────────────────────────────┐
  │     │          │ ① compile_block(第二个闭包)              │
  │     │          │    parse_errors.is_empty() = true       │
  │     │          │    → ir_block = Some(IrBlock)            │
  │     │          └─────────────────────────────────────────┘
  │     │        返回 Expr::Closure(block_id_2)
  │     │
  │     └─ type_check → 可能追加类型错误      parse_errors: [..., E2?]
  │
  │  返回 Block
  │
  ├─ ⑦ compile_block(顶层 Block)
  │    parse_errors.is_empty() ?
  │    若为 true → 顶层 Block.ir_block = Some(IrBlock)
  │    若为 false → 不编译
  │
  ├─ discover_captures_in_closure()
  │    如出错 → working_set.error(err)        parse_errors: [..., E3?]
  │
  └─ 返回 Arc<Block>
```

### 已编译闭包不会被后续错误"撤销"

**核心事实**：`compile_block()` 成功后，`block.ir_block = Some(IrBlock)` 被写入。代码中**没有任何路径**会将 `ir_block` 从 `Some` 重置回 `None`。

这意味着一个关键的时间窗口问题：

```
时间 →
  │
  ├─ T1: 闭包 A 解析完成，parse_errors = []（空）
  │        compile_block(闭包 A) → ir_block = Some(IrBlock_A)  ✓ 已写入
  │
  ├─ T2: 继续解析后续代码
  │        遇到错误 → parse_errors = [E1]
  │
  ├─ T3: 闭包 B 解析完成，parse_errors = [E1]（非空）
  │        compile_block(闭包 B) → 内部检查 is_empty() = false → return
  │        → 闭包 B.ir_block = None  ✗ 未编译
  │
  ├─ T4: 顶层 Block 解析完成，parse_errors = [E1]（非空）
  │        compile_block(顶层) → 检查 is_empty() = false → 不调用
  │        → 顶层 Block.ir_block = None  ✗ 未编译
  │
  └─ 最终状态：
       闭包 A.ir_block = Some(IrBlock_A)   ← 已经编译，不受后续错误影响
       闭包 B.ir_block = None               ← 因为 T2 的错误而被跳过
       顶层 Block.ir_block = None           ← 同上
```

**设计含义**：
- **早期闭包可能"逃过"错误**：如果闭包 A 在 T1 编译时 `parse_errors` 恰好为空，它的 `ir_block` 就会一直保留，即使后续 T2 出现了错误
- 这不是 bug，而是一种**时序依赖**的结果：编译发生在解析过程中（边解析边编译），而不是全部解析完再统一编译
- **编译是否执行取决于调用瞬间的全局状态**，而非整个解析过程的最终状态

### def 体闭包的二次编译

[parse_def()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_def.rs#L490-L496) 中的逻辑值得特别注意：

```rust
match call.positional_iter().nth(2) {
    Some(Expression { expr: Expr::Closure(block_id), .. }) => {
        compile_block_with_id(working_set, *block_id);
        *working_set.get_block_mut(*block_id).signature = sig.clone();
    }
    // ...
}
```

这里对 `def` 的闭包体做了**第二次编译**：
- **第一次**：在 `parse_closure_expression()` 中，闭包体闭包刚解析完时编译（序号①）
- **第二次**：在 `parse_def()` 中，闭包体闭包被识别为 `def` 的函数体后再编译（序号③）

两次编译都会覆盖 `ir_block`。第二次编译还会**设置 `signature`**——这是因为 `def` 的函数签名需要在闭包 Block 上标注，而 `parse_closure_expression()` 编译时签名尚未设置。

**时序效果**：如果第一次编译时 `parse_errors` 为空（成功编译），但第一次和第二次之间出现了新错误，那么第二次 `compile_block_with_id` 会因为 `parse_errors` 非空而跳过——闭包保留第一次编译的 `ir_block`，但**没有 `signature`**。

### source 命令的特殊检查

[parse_source()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_source.rs#L130-L132) 中的编译触发点有一个其他调用方都没有的额外检查：

```rust
let mut block = parse(working_set, Some(&path), &contents, scoped);
if block.ir_block.is_none() {
    let block_mut = Arc::make_mut(&mut block);
    compile_block(working_set, block_mut);
}
```

**为什么检查 `ir_block.is_none()`**？因为 `parse()` 函数内部已经会调用 `compile_block`。如果 `parse()` 成功编译了（`ir_block` 不为 `None`），就不需要再编译一次。只有当 `parse()` 因为有错误而没有编译时，才需要尝试重新编译——但此时 `compile_block` 内部的 `parse_errors.is_empty()` 检查仍然会阻止编译。

**实际效果**：这个 `ir_block.is_none()` 检查可以避免在 `parse()` 已经成功编译时进行冗余的 `compile_block` 调用（减少一条 error log），但不改变功能语义。

### RowCondition 和 with-env 简写的"无检查"编译

[parse_row_condition()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_signatures.rs#L507) 和 [parse_expression() with-env 简写](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_expressions.rs#L1566) 都**不检查** `parse_errors` 就直接调用 `compile_block`。

它们依赖 `compile_block` 内部的保护。如果 `parse_errors` 非空，内部检查会阻止编译并打一条 error log。这些 `Block` 的 `ir_block` 将保持 `None`。

### 全局 parse_errors 与编译时机的精确时序图

```
                parse_errors 状态          编译操作
                ──────────────          ─────────

parse() 开始    []
  │
  ├─ lex()      [E0?]                  （可能注入词法错误）
  │
  ├─ parse_block()
  │    │
  │    ├─lite_parse  [E0?, E1?]        （可能注入语法错误）
  │    │
  │    ├─ 递归解析过程中产生的错误:
  │    │    [E0?, E1?, E2, E3, ...]
  │    │
  │    │    ┌──────────────────────────────────────────────┐
  │    │    │ 在解析每个闭包/def/module/RowCondition 时:   │
  │    │    │                                            │
  │    │    │  当前 parse_errors 状态 = ?                │
  │    │    │        │                                   │
  │    │    │        ├─ 为空 → compile_block 执行        │
  │    │    │        │        → ir_block = Some(...)     │
  │    │    │        │                                   │
  │    │    │        └─ 非空 → compile_block 跳过        │
  │    │    │                 → ir_block = None         │
  │    │    │                 → log::error!() 打印      │
  │    │    └──────────────────────────────────────────────┘
  │    │
  │    └─ type_check   [..., E_type?]   （可能追加类型错误）
  │
  │  返回 Block
  │
  ├─ compile_block(顶层)
  │    当前 parse_errors 状态 = ?
  │    为空 → 顶层 ir_block = Some(...)
  │    非空 → 顶层 ir_block = None
  │
  ├─ discover_captures_in_closure()
  │    [..., E_capture?]              （可能追加捕获分析错误）
  │
  └─ 返回 Arc<Block>
      注意：discover_captures_in_closure 的错误发生在顶层编译之后
      这些错误不会影响已完成的编译决策
```

### 编译时序的三个关键特性

#### 特性一：边解析边编译（Eager Compilation）

编译不是"全部解析完再统一编译"，而是在解析过程中**逐个编译**。每个闭包/def/module/RowCondition 解析完就立即尝试编译。

- **优点**：闭包的 `Block` 存储在 `StateWorkingSet` 块表中，而 IR 编译器只有 `&StateWorkingSet` 不可变访问权，无法回填 `IrBlock`。因此必须在解析阶段（`working_set` 可变时）完成编译。
- **后果**：早期闭包可能在 `parse_errors` 尚为空时成功编译，而后期闭包可能因为中间产生的错误而无法编译。

#### 特性二：已编译的 IrBlock 不可撤回

`block.ir_block` 一旦被设为 `Some(IrBlock)`，代码中没有任何路径会将其重置为 `None`。即使后续产生了 `parse_errors`，已编译的 `IrBlock` 仍然保留在 `Block` 中。

从运行时的角度：当 `parse_errors` 非空时，整个 `Block` 不会被提交到 `EngineState`（由上层调用方决定），因此残留的 `IrBlock` 不会被使用。但在 `StateWorkingSet` 的生命周期内，`IrBlock` 确实存在。

#### 特性三：顶层编译是"最终兜底"

[parse() L546-548](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_captures_compile.rs#L546-L548) 的顶层编译是最晚执行的。它的语义是：如果整个解析过程结束后 `parse_errors` 仍为空，则编译顶层 `Block`。

但这个编译**只编译顶层 Block 本身**，不编译子闭包（子闭包已经在各自解析时编译过了）。顶层 `Block` 的 `IrBlock` 包含的是顶层 pipeline 的 IR，其中通过 `BlockId` 引用子闭包的 `IrBlock`。

---

## 三层错误载体对比总结

| 层次 | 错误载体 | 累积方式 | 访问 WorkingSet | 注入到全局的位置 |
|---|---|---|---|---|
| 词法层 | `Option<ParseError>`（返回值） | 只保留第一个 | 不访问 | 顶层 `parse()` L535-538 + 各嵌套结构入口 Lx（共 6+ 处） |
| 轻量语法层 | `Option<ParseError>`（返回值） | 只保留第一个 | 只读（`&StateWorkingSet`） | `parse_block()` L118-121（每次递归 parse_block 都会注入） |
| AST 构建层 | `Vec<ParseError>`（`StateWorkingSet` 字段） | 全部追加 | 可变（`&mut StateWorkingSet`） | 直接追加，分散各处 + 试探回滚用 truncate |

---

## 总结：核心机制协作关系

### 1. 错误收集的层级关系

- **词法和轻量语法层**：各自用 `Option<ParseError>` 返回值独立运行，都只保留自己层次中遇到的**第一个**错误，彼此互不知道。它们的错误通过调用方的 `if let Some(err) = err { working_set.error(err) }` 注入到全局 `Vec`——这个模式在代码中出现了 **8+ 处**（顶层 parse、每个嵌套结构的 lex 后、每个 parse_block 的lite_parse 后）。

- **AST 构建层**：直接操作共享的 `Vec<ParseError>`，可以累积任意数量的错误。但在试探解析场景下，可以通过"快照 + `truncate`"**有选择地删除**刚追加的一批错误。

### 2. 试探回溯与跨层注入的协作

- 回溯基于"错误数量快照 + truncate"，是**绝对操作**——不管新错误来自词法层、lite_parse 层还是深层语义检查，全部一次性清除。
- 但是否执行回溯，取决于**新增第一个错误的类型**：
  - `Expected` / `ExpectedWithStringMsg` → 回溯（这是"形状不匹配"，试下一个形状）
  - 其他类型（`Unclosed`、`Unbalanced`、`UnexpectedEof` 等） → 不回溯（这是"代码有问题"，直接报出）
- `parse_oneof` 更激进：总是回溯，但最后会把"最优猜测"（走得最远的那次）的错误再补回来。

### 3. 编译门槛的全局语义与时序依赖

- **判断对象是同一个全局 `parse_errors`**。不是"当前块"或"当前作用域"的错误，而是"调用 `compile_block` 的**那一瞬间**全局 `parse_errors` 的状态"。
- 编译发生在解析过程中（边解析边编译），不是全部解析完再统一编译。因此：
  - 早期闭包可能恰好在 `parse_errors` 为空时编译成功
  - 后续错误**不会撤销**已编译闭包的 `IrBlock`（`ir_block` 一旦设为 `Some` 不会被重置为 `None`）
  - 后期闭包因 `parse_errors` 非空而跳过编译（`ir_block` 保持 `None`）
- 顶层编译在 `parse_block()` 返回后、`discover_captures_in_closure()` 之前执行，是最后一个编译机会
- `def` 体闭包被编译**两次**：第一次在 `parse_closure_expression()` 中，第二次在 `parse_def()` 中（覆盖 `ir_block` 并设置 `signature`）

### 4. 全局一致的设计哲学

1. **每层独立产出有效输出**：即使存在错误，每层也保证产出结构完整的输出（token 流、LiteBlock、Block）
2. **两层 Option + 一层 Vec**：词法和轻量语法层各自只记首个错（通过返回值 `Option`），AST 层累积所有错（通过 `StateWorkingSet` 共享的 `Vec`）
3. **错误侧传**：错误通过返回值或全局状态的 side channel 传递，不影响函数的正常返回值路径
4. **无 Result 传播**：解析函数几乎都返回值本身，而非 `Result`，确保下游总能拿到可用的结构
5. **试探基于全局错误快照**：truncate 绝对回滚，但 first error 类型决定"要不要回滚"
6. **边解析边编译**：编译时机与解析交织，检查的是调用瞬间的全局状态，已编译的 `IrBlock` 不可撤回

这种设计使得 Nushell 的解析器能够**一次性报告尽可能多的错误**（AST 层不中断），同时在形状歧义场景下能高效试错（truncate 回滚干净）。编译门槛基于调用瞬间的全局 `parse_errors` 状态——早期无错时编译的闭包不会因后续错误而被撤销，但整个 `Block` 是否被提交到运行时仍由上层调用方根据最终 `parse_errors` 状态决定。
