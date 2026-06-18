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
4. **IR 编译** `compile_block()` — Block → IrBlock（仅无错时执行）
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

### 错误产生

词法层产生的错误主要是结构性问题：
- `UnexpectedEof` — 未闭合的字符串或定界符
- `Unbalanced` — 不匹配的闭合定界符（如 `}` 遇到开 `{`）

这些错误被记录到 `LexState.error` 中，但扫描不会停止——`lex_internal()` 会继续产出后续 token，这是第一层错误恢复机制。

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

### 错误恢复

lite_parse 的错误恢复策略是**记录错误但不中断**：

1. 重定向缺少目标 → 记录 `ParseError::Expected("redirection target")`，把重定向 span 当作普通 part 推入
2. 多重重定向 → 记录 `ParseError::MultipleRedirections`，保留第一个重定向
3. 管道末尾缺少后续 → 最终检查时返回 `ParseError::UnexpectedEof("pipeline missing end")`
4. `||` → 记录 `ParseError::ShellOrOr`，但把它当作普通 part 继续解析

关键点：**即使出错，lite_parse 仍然会产出一个（可能不完整的）LiteBlock**，下游可以继续处理。

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

---

## 错误恢复机制

Nushell 的错误恢复贯穿三个层次，核心理念是**继续解析，不因错误中断**：

### 1. 词法层恢复

- 未闭合字符串/定界符 → 产生 `UnexpectedEof` 错误但仍产出 token
- 不匹配的闭合符 → 产生 `Unbalanced` 错误但仍推进 offset
- 错误仅记录第一个（`state.error`），后续错误被覆盖

### 2. 轻量语法层恢复

- 重定向缺少目标 → 错误记录，span 作为普通 part 继续
- 管道后缺少命令 → 最终统一检查，产出 `UnexpectedEof`
- 始终产出完整的 `LiteBlock` 结构，即使内部有错误

### 3. AST 构建层恢复

这是错误恢复最丰富的层次，有几种核心策略：

#### a) Garbage 节点替代

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

#### b) 错误累积，不中断

`StateWorkingSet.parse_errors` 是一个 `Vec<ParseError>`，通过 `working_set.error()` 追加错误但不返回 `Result`。解析函数的签名几乎全部返回值本身（`Expression`、`Pipeline`、`Block`），而非 `Result`：

```rust
// 典型模式
if bad_condition {
    working_set.error(ParseError::Expected("something", span));
    return garbage(working_set, span);  // 返回占位，而非 Err
}
```

#### c) 试错回溯（Speculative Parsing）

`parse_value()` 在 `SyntaxShape::Any` 模式下，对多种候选形状逐一尝试，通过 `working_set.parse_errors.len()` 快照来判断是否成功：

```rust
let starting_error_count = working_set.parse_errors.len();
let s = parse_value(working_set, span, shape);
if starting_error_count == working_set.parse_errors.len() {
    return s;  // 无新错误 → 成功
}
working_set.parse_errors.truncate(starting_error_count);  // 回溯错误
continue;  // 尝试下一个形状
```

[parse_oneof()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-nushell/crates/nu-parser/src/parse_calls.rs#L682-L741) 更复杂：它尝试所有候选形状，选择"走得最远"（首个错误位置最靠后）的结果。

#### d) 编译门槛

`compile_block()` 只在 `parse_errors` 为空时执行：

```rust
if working_set.parse_errors.is_empty() {
    compile_block(working_set, Arc::make_mut(&mut output));
}
```

这确保 IR 编译不会在损坏的 AST 上运行，同时允许继续收集所有解析错误。

### 错误流的总图

```
lex() / lite_parse() / parse_xxx()
         │
         │ working_set.error(ParseError)
         ▼
  StateWorkingSet.parse_errors: Vec<ParseError>
         │
         ├─ parse_value(Any) / parse_oneof() → truncate 回溯
         ├─ compile_block() → 仅在 empty 时执行
         └─ 最终返回给调用方：Arc<Block> + errors 在 working_set 中
```

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

| 层次 | 输入 | 输出 | 核心职责 | 错误策略 |
|---|---|---|---|---|
| **词法层** (lex) | 字节流 | `Vec<Token>` | 分词、字符串/定界符嵌套、重定向识别 | 记录首个错误，继续扫描 |
| **轻量语法层** (lite_parse) | `Vec<Token>` | `LiteBlock` | 组织命令/管道/重定向/注释/属性 | 记录错误，产出不完整但有效的结构 |
| **AST 层** (parse_block 等) | `LiteBlock` | `Block` (含 AST) | 类型推断、关键字分发、优先级解析、变量作用域 | Garbage 占位 + 错误累积 + 试错回溯 |

三层之间的**契约**是：
1. 每层都保证产出有效的输出结构，即使存在错误
2. 错误通过 `StateWorkingSet.parse_errors` 侧通道传递，不影响正常返回值
3. 下层始终可以处理上层的输出（无 `Result` 传播）
4. 编译和执行阶段通过检查 `parse_errors.is_empty()` 来决定是否继续

这种设计使得 Nushell 的解析器能够**一次性报告所有错误**，而非遇到首个错误就终止——这对 IDE 和 REPL 场景尤为重要。
