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

### 递归解析与嵌套

解析过程中会产生递归的 lex → lite_parse → parse_block 循环。例如：

- 列表表达式 `[1, 2, 3]`：内部内容重新 lex + lite_parse
- 闭包表达式 `{ |x| $x + 1 }`：内部内容重新 lex + parse_block
- 记录表达式 `{ a: 1 }`：逐 token 增量 lex（`lex_n_tokens`）

这意味着 Nushell 的解析器不是传统的"一次性 lex 完再 parse"，而是**按需 lex**——某些复合结构会在解析时重新对子区域执行词法分析。

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

## 错误恢复机制详解

Nushell 的错误恢复贯穿三个层次，每层有独立的错误载体和恢复策略。核心理念是**继续解析，不因错误中断**。

### 三层错误载体对比

| 层次 | 错误载体 | 累积方式 | 访问 WorkingSet | 注入到全局的位置 |
|---|---|---|---|---|
| 词法层 | `Option<ParseError>`（返回值） | 只保留第一个 | 不访问 | `parse()` 第 536-538 行 |
| 轻量语法层 | `Option<ParseError>`（返回值） | 只保留第一个 | 只读（`&StateWorkingSet`） | `parse_block()` 第 118-121 行 |
| AST 构建层 | `Vec<ParseError>`（`StateWorkingSet` 字段） | 全部追加 | 可变（`&mut StateWorkingSet`） | 直接追加 |

**关键区别**：词法层和轻量语法层各自独立记录错误（都只记第一个），它们的错误由上层调用方注入到全局 `parse_errors` 中。AST 层则直接操作全局列表，可以累积多个错误。

### AST 层的错误恢复策略

#### 策略一：Garbage 节点替代

[garbage()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_helpers.rs#L5-L7) 函数产出一个 `Expr::Garbage` 表达式，类型为 `Type::Any`：

```rust
pub fn garbage(working_set: &mut StateWorkingSet, span: Span) -> Expression {
    Expression::garbage(working_set, span)  // Expr::Garbage, Type::Any
}
```

所有无法解析的位置都用 `Garbage` 填充，确保 AST 结构完整。例如：
- 不完整的数学表达式 `1 +` → 缺失的右操作数用 `Garbage` 填充
- 类型不匹配的参数 → 用 `Garbage` 替换
- 语法错误 → 整个 span 替换为 `Garbage`

#### 策略二：错误累积，不中断

解析函数的签名几乎全部返回值本身（`Expression`、`Pipeline`、`Block`），而非 `Result`：

```rust
// 典型模式
if bad_condition {
    working_set.error(ParseError::Expected("something", span));
    return garbage(working_set, span);  // 返回占位，而非 Err
}
```

这确保了无论解析过程中遇到多少错误，最终总能产出一个结构完整的 AST。

#### 策略三：试错回溯（Speculative Parsing）

当解析器遇到歧义（不知道该按哪种语法形状解析）时，会依次尝试多种可能性，通过 `parse_errors` 的长度快照判断哪种尝试"更成功"。

**情形一：`parse_value(SyntaxShape::Any)` — 按顺序尝试，首个成功即返回**

[parse_value()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_expressions.rs#L889-L928) 中的 `Any` 分支：

```rust
let shapes = [Binary, Range, Filesize, Duration, DateTime, Int, Number, String];
for shape in shapes.iter() {
    let starting_error_count = working_set.parse_errors.len();
    let s = parse_value(working_set, span, shape);

    if starting_error_count == working_set.parse_errors.len() {
        return s;  // 无新错误 → 成功，直接返回
    } else {
        match working_set.parse_errors.get(starting_error_count) {
            // Expected 类错误 → 回溯，继续尝试下一个形状
            Some(ParseError::Expected(_, _) | ParseError::ExpectedWithStringMsg(_, _)) => {
                working_set.parse_errors.truncate(starting_error_count);
                continue;
            }
            // 其他类型错误 → 认为是确定性错误，直接返回
            _ => return s,
        }
    }
}
```

回溯条件：只有当新产生的错误是 `Expected` 或 `ExpectedWithStringMsg` 类型时才回溯。这是因为 Expected 类错误通常表示"形状不匹配"而非"代码有问题"，而其他类型的错误（如 `Unbalanced`、`UnexpectedEof`）即使形状不对也说明代码本身有错，应该保留。

**情形二：`parse_oneof(SyntaxShape::OneOf)` — 全部尝试，选最优**

[parse_oneof()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_calls.rs#L682-L741) 更复杂：

```rust
for shape in possible_shapes {
    let starting_error_count = working_set.parse_errors.len();
    let value = parse_multispan_value(working_set, spans, spans_idx, shape);
    
    let new_errors = &working_set.parse_errors[starting_error_count..];
    let Some(first_error_offset) = new_errors.iter().map(|e| e.span().start).min() else {
        return value;  // 无新错误 → 成功，直接返回
    };
    
    if first_error_offset > max_first_error_offset {
        // 记录"走得更远"的结果
        max_first_error_offset = first_error_offset;
        best_guess = Some(value);
        best_guess_errors.clear();
        best_guess_errors.extend_from_slice(new_errors);
    }
    working_set.parse_errors.truncate(starting_error_count);  // 回溯
}

// 所有形状都失败 → 选择"最优猜测"的错误
if max_first_error_offset > spans[starting_spans_idx].start || propagate_error {
    working_set.parse_errors.extend(best_guess_errors);
    best_guess.unwrap()
}
```

策略：尝试所有候选形状，选择**首个错误位置最靠后**（即解析"走得最远"）的结果作为最终答案。其余尝试产生的错误通过 `truncate` 回溯掉。

---

## IR 编译的前置条件

### compile_block() 的自我保护

[compile_block()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_captures_compile.rs#L11-L26) 自身有第一道防线：

```rust
pub fn compile_block(working_set: &mut StateWorkingSet<'_>, block: &mut Block) {
    if !working_set.parse_errors.is_empty() {
        // This means there might be a bug in the parser, since calling this function
        // while parse errors are present is a logic error.
        // However, it's not fatal and it's best to continue without doing anything.
        log::error!("compile_block called with parse errors");
        return;
    }

    match nu_engine::compile(working_set, block) {
        Ok(ir_block) => { block.ir_block = Some(ir_block); }
        Err(err) => working_set.compile_errors.push(err),
    }
}
```

注意：这里比较的是 `parse_errors`（解析错误），而 `compile_errors` 是另一回事。即使解析成功，编译也可能产生编译错误。

### 顶层编译触发点

[parse() 函数第 546-548 行](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_captures_compile.rs#L546-L548)：

```rust
if working_set.parse_errors.is_empty() {
    compile_block(working_set, Arc::make_mut(&mut output));
}
```

顶层 Block 的编译**只在零解析错误时**触发。即使 `compile_block` 自身有保护，调用方也做了检查——双重保险。

### 闭包的立即编译

闭包比较特殊，它需要在解析时就立即编译。[parse_closure_expression()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_expressions.rs#L759-L769)：

```rust
// NOTE: closures need to be compiled eagerly due to these reasons:
//  - their `Block`s (which contains their `IrBlock`) are stored in the working_set
//  - Ir compiler does not have mutable access to the working_set and can't attach
//    `IrBlock`s to existing `Block`s
// so they can't be compiled as part of their parent `Block`'s compilation
if working_set.parse_errors.is_empty() {
    compile_block(working_set, &mut output);
}
```

原因：闭包的 `Block` 存储在 `StateWorkingSet` 的块表中，而 IR 编译器只有 `&StateWorkingSet` 不可变访问权，无法回填 `IrBlock`。因此闭包必须在解析阶段（此时 working_set 是可变的）就完成编译。

### 其他编译触发点

`compile_block` 和 `compile_block_with_id` 在多处被调用，包括：
- `parse_def.rs` — 函数定义的体块
- `parse_module.rs` — 模块体
- `parse_signatures.rs` — 签名相关块
- `parse_source.rs` — source 命令的块

大多数调用方都会检查 `parse_errors.is_empty()`，但由于 `compile_block` 内部有保护，即使漏检查也不会崩溃（只是会打一条 error log）。

---

## 递归解析的特例：嵌套结构

Nushell 中复合结构（列表、表、记录、闭包、块、子表达式）的解析会触发递归的 lex + parse：

| 结构 | 词法方式 | 解析入口 |
|---|---|---|
| 列表 `[a, b]` | `lex()` 一次性 | `parse_list_expression()` |
| 表 `[[a b]; [1 2]]` | `lex()` 一次性 | `parse_table_expression()` |
| 记录 `{a: 1}` | `lex_n_tokens()` 增量 | `parse_record()` |
| 闭包 `{ \|x\| ... }` | `lex()` 一次性 | `parse_closure_expression()` |
| 块 `{ ... }` | `lex()` 一次性 | `parse_block_expression()` |
| 子表达式 `(expr)` | `lex()` 一次性 | `parse_paren_expr()` → `parse_block(is_subexpression=true)` |
| 匹配块 `match { ... }` | `lex()` + 手动遍历 | `parse_match_block_expression()` |

**记录的增量词法**：[parse_record()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_expressions.rs#L1795-L2018) 使用 `lex_n_tokens()` 每次只 lex 一个 token，这是因为记录的 `:` 分隔符需要作为 `special_tokens` 处理，而键和值的边界需要精确控制。

---

## 总结：三层协作关系

### 核心数据契约

| 层次 | 输入 | 输出 | 核心职责 | 错误载体 | 错误策略 |
|---|---|---|---|---|---|
| **词法层** (lex) | 字节流 | `(Vec<Token>, Option<ParseError>)` | 分词、字符串/定界符嵌套、重定向识别 | `Option` | 记首个错，继续扫描 |
| **轻量语法层** (lite_parse) | `&[Token]` | `(LiteBlock, Option<ParseError>)` | 组织命令/管道/重定向/注释/属性 | `Option` | 记首个错，产出完整结构 |
| **AST 层** (parse_block 等) | `LiteBlock` | `Block` (含完整 AST) | 类型推断、关键字分发、优先级解析、变量作用域 | `Vec<ParseError>` in StateWorkingSet | Garbage 占位 + 全量累积 + 试错回溯 |

### 错误流总图

```
Source bytes
    │
    ▼
┌───────────────────────────────────────────────────┐
│  lex()                                            │
│  错误: Option<ParseError> (只记第一个)             │
└───────────────────────┬───────────────────────────┘
                        │
                        │ parse() 注入:
                        │   if let Some(err) = err {
                        │       working_set.error(err)
                        │   }
                        ▼
              StateWorkingSet.parse_errors
                        ▲
                        │ parse_block() 注入:
                        │   if let Some(err) = err {
                        │       working_set.error(err)
                        │   }
                        │
┌───────────────────────┴───────────────────────────┐
│  lite_parse()                                      │
│  错误: Option<ParseError> (只记第一个)             │
│  working_set: 只读访问                             │
└───────────────────────┬───────────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────────┐
│  parse_block() / parse_expression() / ...         │
│  错误: 直接 working_set.error(err) 追加            │
│  working_set: 可变访问                             │
│  Garbage 占位保证 AST 结构完整                     │
│  parse_value(Any) / parse_oneof → truncate 回溯   │
└───────────────────────┬───────────────────────────┘
                        │
                        │ 编译门槛:
                        │   if working_set.parse_errors.is_empty()
                        │
                        ▼
              ┌─────────────────────┐
              │  compile_block()    │  双重检查：内部也会判断 parse_errors
              │  产出: IrBlock      │
              └─────────────────────┘
```

### 关键设计原则

1. **每层独立产出有效输出**：即使存在错误，每层也保证产出结构完整的输出（token 流、LiteBlock、Block）
2. **两层 Option + 一层 Vec**：词法和轻量语法层各自只记首个错（通过返回值 `Option`），AST 层累积所有错（通过 `StateWorkingSet` 共享的 `Vec`）
3. **错误侧传**：错误通过返回值或全局状态的 side channel 传递，不影响函数的正常返回值路径
4. **无 Result 传播**：解析函数几乎都返回值本身，而非 `Result`，确保下游总能拿到可用的结构
5. **双重编译保险**：调用方检查 + `compile_block` 内部检查，确保 IR 绝不在有解析错误的 AST 上运行
6. **闭包立即编译**：闭包在解析期就编译 IR，因为后续编译阶段没有可变访问权来回填

这种设计使得 Nushell 的解析器能够**一次性报告尽可能多的错误**，而非遇到首个错误就终止——这对 IDE 智能提示和 REPL 体验尤为重要。
