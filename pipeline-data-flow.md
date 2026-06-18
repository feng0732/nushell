# Nushell Pipeline 数据流分析：惰性处理、值/流/错误传播协作

## 1. 核心类型体系概览

Nushell 的 pipeline 数据流围绕三个核心抽象构建：

### 1.1 `PipelineData` — pipeline 传输的顶层容器

定义于 [pipeline_data.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L49-L54)：

```rust
pub enum PipelineData {
    Empty,
    Value(Value, Option<PipelineMetadata>),
    ListStream(ListStream, Option<PipelineMetadata>),
    ByteStream(ByteStream, Option<PipelineMetadata>),
}
```

四种变体代表了数据在 pipeline 中的三种存在形态：

| 变体 | 语义 | 惰性程度 |
|------|------|---------|
| `Empty` | 无数据 | N/A |
| `Value` | 已完全求值的具体值 | **严格值**（非惰性） |
| `ListStream` | 逐元素惰性产出的值流 | **惰性**（元素按需迭代） |
| `ByteStream` | 原始字节流 | **惰性**（字节按需读取） |

### 1.2 `Value` — 结构化数据的原子单位

定义于 [mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/value/mod.rs#L77-L219)，包含 `Bool`、`Int`、`String`、`Record`、`List`、`Range`、`Custom`、`Error` 等变体。

关键惰性相关变体：
- **`Value::Range`**：携带 `Option<Signals>`，迭代时才计算元素
- **`Value::List`**：携带 `Option<Signals>`，`vals` 是 `Vec<Value>` 已全部求值，但通过 `signals` 保留中断能力
- **`Value::Custom`**：包裹 `Box<dyn CustomValue>`，可通过 `to_base_value()` 惰性展开
- **`Value::Error`**：作为值嵌入流中的错误，惰性传播的关键机制

### 1.3 `ListStream` — 值迭代器的惰性包装

定义于 [list_stream.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/list_stream.rs#L7-L20)：

```rust
pub type ValueIterator = Box<dyn Iterator<Item = Value> + Send + 'static>;

pub struct ListStream {
    stream: ValueIterator,
    span: Span,
    caller_spans: Vec<Span>,
}
```

核心特征：**一次性消费**。`ListStream` 内部是 `Box<dyn Iterator>`，只能迭代一次，观察即消耗。

### 1.4 `ByteStream` — 字节流的惰性包装

定义于 [byte_stream.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs#L191-L198)：

```rust
pub struct ByteStream {
    stream: ByteStreamSource,  // Read / File / Child
    span: Span,
    signals: Signals,
    type_: ByteStreamType,     // Binary / String / Unknown
    known_size: Option<u64>,
    caller_spans: Vec<Span>,
}
```

`ByteStreamType` 决定了流在收集时的类型归属：
- `Binary` → 总是 `Value::Binary`
- `String` → 总是 `Value::String`（UTF-8 验证失败则报错）
- `Unknown` → 先尝试 UTF-8，失败则回退到 `Binary`（**运行时才决定类型**）

---

## 2. 惰性处理不够直观的关键位置

### 2.1 `PipelineData::map`/`filter`/`flat_map` 对 ByteStream 的强制收集

**位置**: [pipeline_data.rs#L477-L527](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L477-L527)

当输入是 `ByteStream` 时，`map`、`filter`、`flat_map` 全部**立即收集**整个流为值，再应用变换：

```rust
// PipelineData::map 中的 ByteStream 分支
PipelineData::ByteStream(stream, metadata) => {
    Ok(f(stream.into_value()?).into_pipeline_data_with_metadata(metadata))
}
```

```rust
// PipelineData::filter 中的 ByteStream 分支
PipelineData::ByteStream(stream, metadata) => {
    // TODO: is this behavior desired / correct ?
    let span = stream.span();
    let value = match String::from_utf8(stream.into_bytes()?) { ... };
    ...
}
```

**不直观之处**：
- 用户可能认为 `| filter` 或 `| map` 会逐元素惰性处理，但对于 `ByteStream`，这些方法会先全部读入内存
- `flat_map` 对 `ByteStream` 更是先做 `into_bytes()` 再做 UTF-8 解码（[pipeline_data.rs#L579-L586](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L579-L586)），对大文件或无限流会造成内存溢出
- 源码中已有 `// TODO: is this behavior desired / correct ?` 的注释，说明行为本身未定

### 2.2 `PipelineData::follow_cell_path` 对 ListStream 的全量收集

**位置**: [pipeline_data.rs#L455-L474](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L455-L474)

```rust
pub fn follow_cell_path(self, cell_path: &[PathMember], head: Span) -> Result<Value, ShellError> {
    match self {
        PipelineData::ListStream(stream, ..) =>
            Value::list(stream.into_iter().collect(), head)
                .follow_cell_path(cell_path)
                .map(Cow::into_owned),
        ...
    }
}
```

**不直观之处**：
- `follow_cell_path` 遇到 `ListStream` 时，先用 `.collect()` 收集为 `Vec<Value>`，再执行路径追踪
- 即便路径只是取第 0 个元素，也会收集整个流
- 对比 `get` 命令的实现（[get.rs#L276-L308](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/get.rs#L276-L308)）使用了 `follow_cell_path_into_stream` 来保持流式，但底层的 `PipelineData::follow_cell_path` 本身并不做此优化

### 2.3 `get` 命令在多路径模式下的全量收集

**位置**: [get.rs#L245-L267](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/get.rs#L245-L267)

```rust
// 多个 cell path 时的分支
let input = input.into_value(span)?;  // 立即收集！
for path in paths {
    output.push(input.follow_cell_path(&path.members)?.into_owned());
}
```

**不直观之处**：
- 单个 cell path 时通过 `follow_cell_path_into_stream` 保持流式
- 传入多个 cell path（如 `get a b c`）时，`into_value()` 将整个 `ListStream` 收集成 `Value::List`
- 行为不一致且未在文档中说明，用户无法预期何时流被收集

### 2.4 `Value::Custom` 的惰性展开（`to_base_value`）时机不确定

**位置**: [custom_value.rs#L27](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/value/custom_value.rs#L27)

```rust
fn to_base_value(&self, span: Span) -> Result<Value, ShellError>;
```

**不直观之处**：
- `CustomValue` 的展开时机完全取决于具体命令是否调用 `to_base_value()`
- 在 `PipelineData::into_iter` 中，可迭代的 Custom 值被立即展开为 List 或 Range（[pipeline_data.rs#L982-L997](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L982-L997)）
- 在 `PipelineData::try_into_stream` 中，Custom 值也会被尝试展开（[pipeline_data.rs#L223-L243](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L223-L243)）
- 但在 `PipelineData::map`/`filter` 等方法中，`Value::Custom` 不在特殊处理分支中，会被当作标量值直接传给闭包
- 同一个 Custom 值在不同命令中可能被展开也可能不被展开，行为不可预测
- `is_iterable()` 方法已被标记为 `deprecated`（[custom_value.rs#L170-L177](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/value/custom_value.rs#L170-L177)），预示着这套机制即将变更

### 2.5 `ByteStreamType::Unknown` 的运行时类型决策

**位置**: [byte_stream.rs#L662-L681](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs#L662-L681)

```rust
ByteStreamType::Unknown => match String::from_utf8(self.into_bytes()?) {
    Ok(mut str) => { ... Value::string(str, span) }
    Err(err) => Value::binary(err.into_bytes(), span),
}
```

**不直观之处**：
- 外部命令产生的 `ByteStream` 默认类型为 `Unknown`
- 最终是 string 还是 binary 取决于**整个流的 UTF-8 合法性**，这只有在完全读取后才能判断
- `Chunks` 迭代器在遇到非法 UTF-8 时会从 String 模式**不可逆地切换到 Binary 模式**（[byte_stream.rs#L1075-L1088](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs#L1075-L1088)），之后即使后续数据是合法 UTF-8，也全部作为 binary 输出
- 这意味着同一个流的类型在迭代过程中可能不一致，违反了类型系统的静态预期

### 2.6 `collect_reg` 的隐式收集与错误传播

**位置**: [eval_ir.rs#L208-L221](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-engine/src/eval_ir.rs#L208-L221)

```rust
fn collect_reg(&mut self, reg_id: RegId, fallback_span: Span) -> Result<Value, ShellError> {
    let body = self.take_reg(reg_id).body;
    let span = body.span().unwrap_or(fallback_span);
    body.into_value(span)
}
```

**不直观之处**：
- IR 评估器中，许多操作（`StoreVariable`、`PushPositional`、`BinaryOp` 等）通过 `collect_reg` 将 `PipelineData` 收集为 `Value`
- 这个收集过程会触发 `ListStream` 的全量迭代和 `ByteStream` 的全量读取
- 如果流中某个元素是 `Value::Error`，`into_value()` 会在 `ListStream::into_value` 中通过 `unwrap_error` 传播（[list_stream.rs#L81-L88](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/list_stream.rs#L81-L88)）
- 错误不是在产生时立即报告，而是在下游消费时才暴露——这是惰性求值的经典陷阱

---

## 3. 值、流和错误传播的协作机制

### 3.1 双重错误传播路径

Nushell 存在两套并行的错误传播机制：

**路径 A: `Result<_, ShellError>` — 结构化错误**

评估器和命令通过 `Result` 类型返回错误，这是 Rust 的标准错误处理：

```
eval_ir_block → eval_ir_block_impl → eval_call → decl.run()
    ↓ Err(ShellError)
← Err(ShellError)
```

在 [eval_ir.rs#L1566](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-engine/src/eval_ir.rs#L1566) 中，`check_input_types` 发现 `PipelineData::Value(Value::Error, ..)` 时也会立即转成 `Err`：

```rust
PipelineData::Value(Value::Error { error, .. }, ..) => return Err(*error.clone()),
```

**路径 B: `Value::Error` — 嵌入流中的错误**

错误也可以作为 `Value::Error` 变体存在于流中，随数据一起惰性传播：

- `each` 命令中，闭包执行失败时将错误包装为 `Value::error(err, head)` 放入输出流（[each.rs#L155-L156](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/each.rs#L155-L156)）
- `where` 命令中，条件求值失败时同样嵌入 `Value::Error`（[where_.rs#L78](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/where_.rs#L78)）
- `get` 命令中，cell path 追踪失败时用 `Value::error` 替代（[get.rs#L297](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/get.rs#L297)）

**两条路径的交汇点**：

| 场景 | 转换位置 | 行为 |
|------|---------|------|
| `ListStream::into_value` | [list_stream.rs#L81-L88](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/list_stream.rs#L81-L88) | `unwrap_error` 将 `Value::Error` 提升为 `Err(ShellError)` |
| `PipelineData::drain` | [pipeline_data.rs#L329-L337](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L329-L337) | `Value::Error` → `Err`，其余静默消费 |
| `collect_reg` | [eval_ir.rs#L208-L221](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-engine/src/eval_ir.rs#L208-L221) | 通过 `into_value` 间接触发 |
| `PipelineIterator::next` | [pipeline_data.rs#L1013-L1031](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L1013-L1031) | `ByteStream` 的错误被包装为 `Value::Error`，但 `ListStream` 的错误直接透传 |
| `write_all_and_flush` | [pipeline_data.rs#L806-L808](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L806-L808) | 遍历元素时遇到 `Value::Error` 立即返回 `Err` |

### 3.2 `PipelineIterator` 的不一致错误处理

**位置**: [pipeline_data.rs#L1013-L1031](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L1013-L1031)

```rust
impl Iterator for PipelineIterator {
    type Item = Value;

    fn next(&mut self) -> Option<Self::Item> {
        match &mut self.0 {
            PipelineIteratorInner::ByteStream(stream) => stream.next().map(|x| match x {
                Ok(x) => x,
                Err(err) => Value::error(err, Span::unknown()), //FIXME
            }),
            PipelineIteratorInner::ListStream(stream, ..) => stream.next(),  // 直接透传 Value::Error
            ...
        }
    }
}
```

**不直观之处**：
- `ByteStream::Chunks` 产出的错误被包装为 `Value::Error` 放回流中
- `ListStream` 产出的 `Value::Error` 直接透传，不做额外处理
- 两者语义不同但外层消费者无法区分

### 3.3 `drain` 的延迟错误发现

**位置**: [pipeline_data.rs#L329-L337](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L329-L337) 和 [eval_ir.rs#L1714-L1771](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-engine/src/eval_ir.rs#L1714-L1771)

`drain` 操作用于消费并丢弃 pipeline 数据（例如分号 `;` 后的语句或赋值语句右侧的副作用命令）：

```rust
pub fn drain(self) -> Result<(), ShellError> {
    match self {
        Self::Value(Value::Error { error, .. }, ..) => Err(*error),
        Self::Value(..) => Ok(()),
        Self::ListStream(stream, ..) => stream.drain(),
        Self::ByteStream(stream, ..) => stream.drain(),
    }
}
```

**不直观之处**：
- `PipelineData::Value(Value::Error, ..)` 在 `drain` 时立即报错
- 但 `ListStream` 中的 `Value::Error` 元素在 `ListStream::drain` 中才会被发现（[list_stream.rs#L97-L104](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/list_stream.rs#L97-L104)）
- 如果 `ListStream` 从未被 drain（如被 `ignore` 命令消费），流中的错误可能被静默吞没
- [eval_ir.rs#L1276-L1282](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-engine/src/eval_ir.rs#L1276-L1282) 对 `ignore` 命令有特殊处理，跳过了输入错误传播

### 3.4 `check_input_types` 的早期拦截与 `Value::Error` 的短路

**位置**: [eval_ir.rs#L1547-L1596](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-engine/src/eval_ir.rs#L1547-L1596)

在命令执行前，引擎会检查输入类型：

```rust
match input {
    PipelineData::Value(Value::Error { error, .. }, ..) => return Err(*error.clone()),
    PipelineData::Value(Value::Custom { .. }, ..) => return Ok(()),
    _ => (),
}
```

**不直观之处**：
- `Value::Error` 被提前拦截并转为 `Err`，即使命令本身可能想处理这个错误值
- `Value::Custom` 被无条件放行（bypass 类型检查），因为 Custom 值的类型在展开前无法确定
- `ListStream` 的类型被报告为 `list<any>`（[pipeline_data.rs#L173](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L173)），这意味着**任何**接受 `list<any>` 的命令都会接受 `ListStream`，而不管流内元素的实际类型

---

## 4. 流转中的惰性转换热点

### 4.1 Value → ListStream 的隐式升级

以下操作会将 `Value::List` / `Value::Range` 提升为 `ListStream`：

| 操作 | 位置 | 触发条件 |
|------|------|---------|
| `PipelineData::into_iter` | [pipeline_data.rs#L959-L1001](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L959-L1001) | 总是（List→ListStream，Range→ListStream） |
| `PipelineData::try_into_stream` | [pipeline_data.rs#L200-L246](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L200-L246) | 显式调用 |
| `PipelineData::map` | [pipeline_data.rs#L487-L490](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L487-L490) | `Value::List` 分支通过 `into_pipeline_data` 创建流 |
| `each` 命令 | [each.rs#L153-L167](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/each.rs#L153-L167) | 输出总是 `ListStream` |

**不直观之处**：用户写 `[1 2 3] | each { |x| $x + 1 }` 时，`[1 2 3]` 作为 `Value::List` 进入 `each`，但 `into_iter()` 会先将其转为 `ListStream`，再 map 后输出另一个 `ListStream`。原本紧凑的 `Vec<Value>` 经过两次装箱（`Box<dyn Iterator>`），引入了不必要的间接层。

### 4.2 ListStream → Value 的强制收集

| 操作 | 位置 | 备注 |
|------|------|------|
| `into_value(span)` | [pipeline_data.rs#L178-L191](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L178-L191) | 收集为 `Value::List` |
| `follow_cell_path` | [pipeline_data.rs#L462](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L462) | 先 `.collect()` 再追踪 |
| `collect_reg` | [eval_ir.rs#L208-L221](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-engine/src/eval_ir.rs#L208-L221) | 赋值给变量时 |
| `Drain` 指令 | [eval_ir.rs#L449-L452](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-engine/src/eval_ir.rs#L449-L452) | 消费流并检查错误 |
| `get` 多路径 | [get.rs#L250](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/get.rs#L250) | `into_value` 收集 |

### 4.3 ByteStream → Value 的类型歧义

| ByteStreamType | `into_value` 结果 | `chunks` 行为 |
|----------------|-------------------|--------------|
| `Binary` | `Value::Binary` | 直接产出 `Value::Binary` |
| `String` | `Value::String`（UTF-8 失败→Err） | 产出 `Value::String`，UTF-8 失败→Err |
| `Unknown` | 先尝试 UTF-8→String，否则→Binary | 先 String，UTF-8 失败后永久切换到 Binary |

**Unknown 的 chunks 行为**（[byte_stream.rs#L1075-L1088](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs#L1075-L1088)）是最不直观的：同一个迭代器可能先产出 String 值，后产出 Binary 值，导致下游命令面对混合类型。

---

## 5. 信号中断的传播层次

### 5.1 `Signals` 的注入点

`Signals`（定义于 [signals.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/signals.rs)）在以下层次被检查：

| 层次 | 位置 | 检查方式 |
|------|------|---------|
| `InterruptIter`（ListStream 内部） | [list_stream.rs#L177-L186](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/list_stream.rs#L177-L186) | 每次 `next()` 检查 `interrupted()` |
| `ByteStream::Reader` | [byte_stream.rs#L817-L822](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs#L817-L822) | 每次 `read()` 调用 `signals.check()` |
| `Chunks` 迭代器 | [byte_stream.rs#L1042](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs#L1042) | 每次 `next()` 检查 |
| `copy_with_signals` | [byte_stream.rs#L1155-L1179](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs#L1155-L1179) | 每次读取循环检查 |
| `write_all_and_flush` | [pipeline_data.rs#L933-L934](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L933-L934) | 每个输出块检查 |

**不直观之处**：
- `Value::List` 内嵌的 `signals: Option<Signals>` 在 `into_iter` 转换为 `ListStream` 时才被使用，但 `Value::List` 本身作为 `PipelineData::Value` 传递时不检查中断
- `Value::Range` 的 `signals` 也只在迭代时生效，直接 `into_value` 收集不检查中断
- 这意味着 `1..1000000000 | into string` 会阻塞直到完成或内存耗尽，而 `1..1000000000 | each { into string }` 可以被 Ctrl+C 中断

### 5.2 信号丢失的场景

- `PipelineData::Value(Value::List { vals, signals: None }, ..)` → `into_iter` 时用 `Signals::empty()`（[pipeline_data.rs#L969](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L969)）
- `PipelineData::Value(Value::Range { signals: None, .. }, ..)` → 同上
- `into_iter_strict` 总是用 `Signals::empty()` 构造内部 `ListStream`（[pipeline_data.rs#L350](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L350)）

---

## 6. 总结：惰性不直观的根本原因

1. **`Value` 与 `Stream` 的二义性**：同一个语义概念（如列表）既可以是 `Value::List`（已求值），也可以是 `ListStream`（惰性），两者在不同命令中的行为差异巨大，但用户无法从表面区分。

2. **错误的双重载体**：`ShellError` 通过 `Result` 和 `Value::Error` 两种路径传播，何时走哪条路取决于数据是否处于流中——但这个切换点分散在各个命令和基础设施方法中，没有统一的策略。

3. **`ByteStream::Unknown` 的运行时决策**：类型在完全消费后才确定，且 chunks 迭代中可能从 String 切换到 Binary，违反了静态类型系统的隐含假设。

4. **Custom 值的展开时机不确定**：`to_base_value` 的调用时机完全取决于具体命令实现，同一个 Custom 值在不同 pipeline 位置的行为可能截然不同。

5. **信号的注入不对称**：有些路径（`into_iter_strict`、`Signals::empty()`）丢失了中断能力，导致某些看似等价的表达式在可中断性上差异巨大。

6. **收集的隐式性**：`collect_reg`、`into_value`、`follow_cell_path` 等操作在开发者无感知的情况下触发了全量收集，破坏了惰性保证。
