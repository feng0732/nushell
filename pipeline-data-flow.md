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

---

## 7. 命令协作链：select / columns / values 与 get / where / each

这六条命令构成了 Nushell 中 Record 和 List 字段操作的核心工具链。它们在字段选择、数据收集、流式保留和错误传播上的行为差异，是理解 pipeline 数据流的关键。

### 7.1 命令功能定位与输入输出类型

| 命令 | 核心语义 | 输入类型 | 输出类型 | 操作粒度 |
|------|---------|---------|---------|---------|
| `get` | 按 cell path 提取值 | record / table / list | any（取决于路径） | **单值提取** |
| `select` | 按 cell path 保留字段，删除其余 | record / table / list | 与输入同构（record→record, table→table） | **结构裁剪** |
| `columns` | 提取所有列名 | record / table | `list<string>` | **元数据提取** |
| `values` | 提取所有列值 | record / table | `list<any>` | **值提取** |
| `where` | 按条件过滤行 | list / table | list / table | **行过滤** |
| `each` | 对每个元素执行闭包 | list / table | list | **逐元素映射** |

### 7.2 `get` 与 `select` 的根本区别

`get` 和 `select` 都接受 cell path，但语义和实现截然不同。

**`get`：提取路径指向的值，输出是裸值**

- 单路径 + 列名（如 `get name`）→ 对 ListStream 逐元素 `follow_cell_path`，**保留流式**（[get.rs#L276-L308](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/get.rs#L276-L308)）
- 单路径 + 行号（如 `get 0`）→ 走 `PipelineData::follow_cell_path`，**全量收集**
- 多路径（如 `get a b c`）→ `into_value(span)` **全量收集**后再逐路径 `follow_cell_path`（[get.rs#L245-L267](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/get.rs#L245-L267)）

```rust
// get 单路径 → 流式保留
pub fn follow_cell_path_into_stream(data, signals, cell_path, head) {
    match data {
        PipelineData::ListStream(stream, ..) if !has_int_member => {
            stream.into_iter()
                .map(|value| value.follow_cell_path(&cell_path)
                    .map(Cow::into_owned)
                    .unwrap_or_else(|err| Value::error(err, span)))  // 错误嵌入流
                .into_pipeline_data(head, signals)
        }
        _ => data.follow_cell_path(&cell_path, head).map(|x| x.into_pipeline_data())
    }
}
```

**`select`：保留指定字段，删除其余，输出与输入同构**

- 对 `Value::List`：逐元素 `follow_cell_path`，构建新 `Record`，输出为 `ListStream`（[select.rs#L302-L327](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/select.rs#L302-L327)）
- 对 `Value::Record`：直接 `follow_cell_path` 构建新 `Record`，输出为 `Value`（[select.rs#L329-L343](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/select.rs#L329-L343)）
- 对 `ListStream`：通过 `.map()` 逐元素构建 `Record`，**保留流式**（[select.rs#L345-L362](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/select.rs#L345-L362)）
- 行选择（整数路径）：通过 `NthIterator` 跳过不需要的行，**保留流式**（[select.rs#L258-L274](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/select.rs#L258-L274)）

```rust
// select 对 ListStream 的处理 → 流式保留
PipelineData::ListStream(stream, metadata, ..) => Ok(stream
    .map(move |x| {
        let mut record = Record::new();
        for path in &columns {
            match x.follow_cell_path(&path.members) {
                Ok(value) => record.push(path.to_column_name(), value.into_owned()),
                Err(e) => return Value::error(e, call_span),  // 错误嵌入流
            }
        }
        Value::record(record, call_span)
    })
    .into_pipeline_data_with_metadata(call_span, engine_state.signals().clone(), metadata))
```

**关键差异对比**：

| 维度 | `get` | `select` |
|------|-------|----------|
| 语义 | 提取路径指向的值 | 保留指定字段，删除其余 |
| 输出类型 | 裸值（与路径深度有关） | 与输入同构（record→record, table→table） |
| 多路径时 | 全量收集 | 流式保留（每个元素构建新 Record） |
| 错误处理 | `Value::error` 嵌入流（单路径）或 `Err` 传播（多路径） | 统一用 `Value::error` 嵌入流 |
| 行选择 | 全量收集 | `NthIterator` 流式跳过 |

### 7.3 `columns` 与 `values` 的全量收集行为

`columns` 和 `values` 都必须"看完整个输入"才能产出结果，这是它们语义上的内在需求：

**`columns`：提取所有列名**

- `Value::List`：调用 `get_columns(&input_vals)` 遍历所有元素收集列名（[columns.rs#L80-L86](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/columns.rs#L80-L86)）
- `Value::Record`：直接迭代 `val.columns()`，**无需收集**（[columns.rs#L96-L100](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/columns.rs#L96-L100)）
- `Value::Custom`：调用 `to_base_value()` **展开**后再提取列名（[columns.rs#L87-L95](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/columns.rs#L87-L95)）
- `ListStream`：`stream.into_iter().collect::<Vec<_>>()` **全量收集**（[columns.rs#L117-L118](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/columns.rs#L117-L118)）
- `Value::Error`：直接 `return Err(*error)` **短路提升**（[columns.rs#L102](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/columns.rs#L102)）

**`values`：提取所有列值**

- `Value::List`（即 table）：调用 `get_values(&vals)` 按**列优先**重组（[values.rs#L99-L130](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/values.rs#L99-L130)），输出为 `list<list<any>>`
- `Value::Record`：直接 `val.values().cloned().collect()`，输出为 `list<any>`（[values.rs#L159-L163](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/values.rs#L159-L163)）
- `Value::Custom`：调用 `to_base_value()` **展开**（[values.rs#L150-L152](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/values.rs#L150-L152)）
- `ListStream`：`stream.into_iter().collect()` **全量收集**（[values.rs#L174-L175](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/values.rs#L174-L175)）
- `Value::Error`：直接 `return Err(*error)` **短路提升**（[values.rs#L165](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/values.rs#L165)）
- `get_values` 内部遇到 `Value::Error` 元素时 **提前终止** 并返回 `Err`（[values.rs#L117](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/values.rs#L117)）

**`columns` 与 `values` 的关键区别**：

| 维度 | `columns` | `values` |
|------|-----------|----------|
| Record 输入 | 直接迭代列名，**零收集** | `val.values().cloned()`，**克隆值** |
| List（table）输入 | `get_columns` 遍历收集列名 | `get_values` 按**列优先**重组为 `list<list>` |
| ListStream 输入 | **全量收集** | **全量收集** |
| ByteStream 输入 | `Err(OnlySupportsThisInputType)` **报错** | `Err(OnlySupportsThisInputType)` **报错** |
| 遇到 Error 元素 | Record 级别短路（`Value::Error`→`Err`），元素级别不检查 | `get_values` 内部**逐元素检查**，遇 Error 立即 `Err` |
| Custom 值 | **展开**后处理 | **展开**后处理 |

### 7.4 `where` 与 `each` 的流式保留与错误传播

`where` 和 `each` 是 pipeline 中最常用的流式处理命令，它们在流式保留和错误传播上的策略直接影响 pipeline 的行为。

**`where`：条件过滤**

- 输入通过 `into_iter_strict(head)` 转为迭代器（[where_.rs#L71](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/where_.rs#L71)）
- `into_iter_strict` 对 `ListStream` 直接返回内部迭代器，**保留流式**；对 `Value::List` 和 `Value::Range` 创建新 `ListStream`
- `into_iter_strict` 对 `ByteStream` 通过 `stream.chunks()` 转为 `Chunks` 迭代器，**也支持流式处理**（[pipeline_data.rs#L410-L416](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L410-L416)）
- `chunks()` 返回 `Option<Chunks>`，当 `stream.reader()` 为 `None` 时返回 `PipelineIteratorInner::Empty`（空流，非报错）
- 闭包求值失败时，**不过滤掉该元素**，而是将原始值替换为 `Value::error(err, head)` 放入输出（[where_.rs#L77-L79](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/where_.rs#L77-L79)）
- 闭包求值成功但条件为 false 时，元素被 `filter_map` 丢弃
- 输出总是 `ListStream`（通过 `into_pipeline_data_with_metadata`）

```rust
// where 的核心逻辑
.filter_map(move |value| {
    match closure.run_with_value(value.clone())
        .and_then(|data| data.into_value(head))
    {
        Ok(cond) => cond.is_true().then_some(value),
        Err(err) => Some(Value::error(err, head)),  // 错误：保留元素但标记为 Error
    }
})
```

**`each`：逐元素映射**

- 对 `Value::Range`、`Value::List`、`ListStream`：通过 `into_iter()` 转为迭代器，**保留流式**（[each.rs#L145-L147](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/each.rs#L145-L147)）
- 对 `ByteStream`：通过 `stream.chunks()` 逐块处理，**保留流式**（[each.rs#L197-L222](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/each.rs#L197-L222)）
- 对 `Value::Custom`（可迭代）：通过 `into_iter()` 展开，**保留流式**（[each.rs#L172-L196](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/each.rs#L172-L196)）
- 对其他 `Value`（非迭代）：直接执行闭包一次，**不创建流**（[each.rs#L226-L229](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/each.rs#L226-L229)）
- `--flatten` 模式：闭包返回的流被 `flat_map` 展开，而非 `map` + `into_value` 收集
- 非 flatten 模式：`each_map` 通过 `pipeline_data.into_value(head)` **收集闭包输出**（[each.rs#L241-L248](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/each.rs#L241-L248)），如果闭包返回流则被全部收集
- `--keep-empty` 为 false（默认）：结果通过 `PipelineData::filter` **过滤掉 Nothing 值**（[each.rs#L235](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/each.rs#L235)）

```rust
// each_map：非 flatten 模式下每个元素的处理
fn each_map(value: Value, closure: &mut ClosureEval, head: Span) -> Result<Value, ShellError> {
    let span = value.span();
    let is_error = value.is_error();
    closure.run_with_value(value)
        .and_then(|pipeline_data| pipeline_data.into_value(head))  // 收集闭包输出
        .map_err(|error| chain_error_with_input(error, is_error, span))  // 错误链式增强
}
```

**`where` 与 `each` 的错误传播对比**：

| 维度 | `where` | `each` |
|------|---------|--------|
| 闭包失败时 | `Value::error(err, head)` **替代原始值** | `Value::error(error, head)` **替代闭包返回值** |
| 错误元素是否保留 | 是（`Some(Value::error)`） | 是（`unwrap_or_else` → `Value::error`） |
| 错误链式增强 | 无（直接包装） | 有（`chain_error_with_input`，[utils.rs#L7-L19](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/utils.rs#L7-L19)），当输入本身不是错误时额外包装为 `EvalBlockWithInput` |
| 闭包输出为流时 | `into_value(head)` 收集 | 非 flatten 模式：`into_value(head)` 收集；flatten 模式：`flat_map` 展开 |
| 闭包输出 Nothing | 不适用（条件判断不产出 Nothing） | 默认过滤掉；`--keep-empty` 保留 |
| 流式保留 | **始终保留**（`into_iter_strict` → `filter_map`） | **始终保留**（`into_iter` → `map`/`flat_map`） |
| ByteStream 处理 | 通过 `chunks()` 逐块处理（`into_iter_strict` 内部转换） | 通过 `chunks()` 逐块处理（显式分支） |
| ByteStream 错误处理 | `PipelineIterator::next` 包装为 `Value::error` | `chunks.map(and_then)` 后 `unwrap_or_else` 包装为 `Value::error` |

### 7.5 Record 与 List 在字段选择上的行为差异

Record 和 List 在 Nushell 中是两种不同的结构化容器，命令对它们的处理方式有根本区别。

**字段选择（`select` / `get`）**：

| 输入类型 | `select name` | `get name` |
|---------|--------------|-----------|
| `Record` | 返回只含 `name` 字段的 Record | 返回 `name` 字段的裸值 |
| `List`（table） | 逐元素提取 `name`，返回只含 `name` 列的 table | 逐元素提取 `name`，返回 `list<any>`（所有行的 name 值） |
| `ListStream` | 流式逐元素提取，输出 `ListStream` | 流式逐元素提取，输出 `ListStream`（单路径时） |
| `ByteStream` | 静默返回空（fallthrough → `PipelineData::empty()`） | `Err(IncompatiblePathAccess)` 报错 |

**数据收集**：

| 输入类型 | `columns` | `values` | `select` | `get`（多路径） |
|---------|----------|---------|----------|-------------|
| `Record` | 零收集（直接迭代列名） | 克隆值（`val.values().cloned()`） | 零收集（直接构建新 Record） | 零收集（`follow_cell_path` 借用） |
| `Value::List` | 遍历所有元素收集列名 | `get_values` 按列优先重组 | 逐元素构建 Record，输出 `ListStream` | **全量收集**（`into_value`） |
| `ListStream` | **全量收集** | **全量收集** | **流式保留** | **全量收集**（`into_value`） |
| `ByteStream` | **报错**（`OnlySupportsThisInputType`） | **报错**（`OnlySupportsThisInputType`） | 静默空（fallthrough） | **全量收集**（`into_value` 转为 string 或 binary） |

**核心差异**：Record 作为单行结构，字段操作天然不需要收集；List（table）作为多行结构，列级操作需要遍历所有行；ListStream 的特殊性在于——`select` 和 `each`/`where` 可以逐元素处理保持流式，但 `columns`/`values` 和 `get`（多路径）因为需要全量视角，不得不收集；ByteStream 则因为语义差异，在字段选择命令中表现各不相同（`get` 报错、`select` 静默空、`columns`/`values` 报错）。

### 7.6 命令链组合中的数据流变迁

以下典型命令链展示了数据在 pipeline 中的形态转换：

**链 1：`ls | select name size` — 流式裁剪**

```
ls → PipelineData::ListStream(ListStream, metadata)
  ↓ select
  ListStream.map(逐元素构建新 Record {name, size})
  → PipelineData::ListStream(ListStream, metadata)
```

- 流式保留：✅ 整个链路不发生全量收集
- 错误传播：`follow_cell_path` 失败 → `Value::error` 嵌入流

**链 2：`ls | get name size` — 多路径时全量收集**

```
ls → PipelineData::ListStream(ListStream, metadata)
  ↓ get (多路径)
  into_value(span) → Value::List  [全量收集！]
  follow_cell_path("name") → Value::List
  follow_cell_path("size") → Value::List
  → PipelineData::ListStream([name_list, size_list], metadata)
```

- 流式保留：❌ `into_value` 触发全量收集
- 错误传播：`follow_cell_path` 失败 → `Err(ShellError)` 立即中断

**链 3：`ls | where type == file | select name` — 流式过滤 + 流式裁剪**

```
ls → PipelineData::ListStream(ListStream, metadata)
  ↓ where
  into_iter_strict → ListStream 的原始迭代器
  filter_map(闭包求值) → ListStream [流式保留]
  → PipelineData::ListStream(ListStream, metadata)
  ↓ select
  ListStream.map(构建 Record {name})
  → PipelineData::ListStream(ListStream, metadata)
```

- 流式保留：✅ 全程流式
- 错误传播：`where` 中闭包失败 → `Value::error` 替代原始值；`select` 中路径追踪失败 → `Value::error` 替代

**链 4：`ls | columns` — 全量收集**

```
ls → PipelineData::ListStream(ListStream, metadata)
  ↓ columns
  into_iter().collect() → Vec<Value>  [全量收集！]
  get_columns → 列名 Vec<String>
  → PipelineData::Value(Value::List([col1, col2, ...]), metadata)
```

- 流式保留：❌ 必须收集全部元素才能确定列名
- 错误传播：`Value::Error` 作为顶层值 → `Err` 短路；元素级 Error 不被 `get_columns` 检查（遇到非 Record 元素返回空列表）

**链 5：`{a:1 b:2} | values` — Record 零收集**

```
{a:1 b:2} → PipelineData::Value(Value::Record, metadata)
  ↓ values
  val.values().cloned() → [1, 2]
  → PipelineData::Value(Value::List([1, 2]), metadata)
```

- 流式保留：N/A（Record 是标量，不涉及流）
- 错误传播：`Value::Error` 作为 Record 本身 → `Err` 短路

**链 6：`ls | each { |r| $r.name }` vs `ls | get name` — 等价但错误传播不同**

```
ls | each { |r| $r.name }
  → 闭包返回值通过 into_value 收集
  → 闭包失败 → chain_error_with_input 增强错误信息
  → 输出 Value::error（如果输入本身是 Error 则不增强）

ls | get name
  → follow_cell_path_into_stream 逐元素追踪
  → 追踪失败 → Value::error(err, span)（无增强）
  → 输出 ListStream
```

- 两者都保留流式，但 `each` 会通过 `chain_error_with_input` 在错误信息中附加上下文，`get` 则只保留原始错误

### 7.7 错误传播策略的完整对比

| 命令 | 错误来源 | 传播方式 | 是否中断流 | 错误增强 |
|------|---------|---------|----------|---------|
| `get`（单路径） | cell path 追踪失败 | `Value::error` 嵌入流 | 否 | 无 |
| `get`（多路径） | cell path 追踪失败 | `Err(ShellError)` | 是 | 无 |
| `get`（ByteStream） | ByteStream 不支持路径访问 | `Err(IncompatiblePathAccess)` | 是 | 无 |
| `select` | cell path 追踪失败 | `Value::error` 嵌入流 | 否 | 无 |
| `select`（ByteStream） | fallthrough 空分支 | 静默返回 `PipelineData::empty()` | 否（空流） | N/A |
| `columns` | 输入为 `Value::Error` | `Err(*error)` 短路 | 是 | 无 |
| `columns` | `get_columns` 遇非 Record 元素 | 静默返回空列表 | 否 | N/A |
| `columns`（ByteStream） | 类型不匹配 | `Err(OnlySupportsThisInputType)` | 是 | 无 |
| `values` | 输入为 `Value::Error` | `Err(*error)` 短路 | 是 | 无 |
| `values` | `get_values` 遇 `Value::Error` 元素 | `Err(*error)` 提前终止 | 是 | 无 |
| `values` | `get_values` 遇非 Record 元素 | `Err(OnlySupportsThisInputType)` | 是 | 无 |
| `values`（ByteStream） | 类型不匹配 | `Err(OnlySupportsThisInputType)` | 是 | 无 |
| `where` | 闭包求值失败 | `Value::error` 替代原始值 | 否（流继续，后续元素可能正常） | 无 |
| `where`（ByteStream） | chunks IO / UTF-8 错误 | `Value::error` 嵌入流（经 PipelineIterator 包装） | 产出一个错误值后流结束（chunks 设置 error 标记） | 无 |
| `each` | 闭包求值失败 | `Value::error` 替代闭包返回值 | 否（流继续，后续元素可能正常） | `chain_error_with_input` |
| `each`（ByteStream） | chunks IO / UTF-8 错误 | `Value::error` 嵌入流（经 unwrap_or_else 包装） | 产出一个错误值后流结束（chunks 设置 error 标记） | 无 |

**关键发现**：

1. `columns` 和 `values` 对 `Value::Error` 的处理比 `select` 和 `each` **更严格**：前者将顶层 `Value::Error` 立即提升为 `Err`，后者将其嵌入流中延迟传播
2. `get` 在多路径模式下**最严格**：直接 `Err` 中断整个 pipeline
3. `where` 的错误处理**最宽松**：闭包失败的元素被保留为 `Value::Error`，允许下游命令看到错误并决定如何处理
4. `each` 的 `chain_error_with_input` 是唯一提供**错误上下文增强**的命令——当输入本身不是错误时，会包装为 `EvalBlockWithInput` 错误
5. **ByteStream 的错误是"一次性的"**：chunks 迭代器设置 `self.error = true` 后，下一次 `next()` 直接返回 `None`，即错误元素之后不会再有更多元素流出——但从外部看，错误被包装为 `Value::Error` 嵌入流中，表现为"一个错误元素 + 流结束"
6. `select` 对 ByteStream 的处理**最不直观**：既不报错也不处理，直接返回空流，用户可能困惑为什么没有输出

### 7.8 `into_stream_or_original` 与 `into_iter_strict` 的分水岭

`columns` 和 `values` 都使用了 `PipelineData::into_stream_or_original`（[columns.rs#L70](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/columns.rs#L70)，[values.rs#L137](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/values.rs#L137)），而 `where` 使用了 `into_iter_strict`。这两个方法的行为差异决定了命令对流的态度：

| 方法 | 语义 | `Value::List` 行为 | `ListStream` 行为 | `ByteStream` 行为 | `Value::Error` 行为 |
|------|------|-------------------|-------------------|-------------------|-------------------|
| `into_stream_or_original` | 尽量转为流，否则保持原样 | 转为 `ListStream` | 透传 | 保持 `ByteStream` | 保持 `Value` |
| `into_iter_strict` | 必须转为迭代器（返回 `Result`） | 转为 `ListStream` | 透传 | **转为 `Chunks` 迭代器**（`stream.chunks()`） | **报错**（`return Err(*error)`） |

`into_stream_or_original` 的关键代码（[pipeline_data.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs)）：

```rust
pub fn into_stream_or_original(self, engine_state: &EngineState) -> PipelineData {
    match self {
        PipelineData::Value(Value::List { .. }, ..) =>
            self.into_iter().into_pipeline_data(...),  // Value::List → ListStream
        PipelineData::Value(Value::Range { .. }, ..) =>
            self.into_iter().into_pipeline_data(...),  // Value::Range → ListStream
        PipelineData::Value(Value::Custom { ref val, .. }, ..) if val.is_iterable() =>
            self.into_iter().into_pipeline_data(...),  // 可迭代 Custom → ListStream
        other => other,  // 其余保持原样（包括 Record、ByteStream、Error）
    }
}
```

**不直观之处**：`columns` 和 `values` 对 `Value::List` 先转为 `ListStream` 再在 match 的 `ListStream` 分支中收集——等价于先拆再装，比直接在 `Value` 分支处理多了一次 `Box<dyn Iterator>` 的间接层。

### 7.9 Record 与 List 在 `Record` 内部的结构差异

`Record`（定义于 [record.rs#L17-L19](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/value/record.rs#L17-L19)）的底层是 `Vec<(String, Value)>`：

```rust
pub struct Record {
    inner: Vec<(String, Value)>,
}
```

这意味着：

1. **Record 是紧凑的列式存储**：所有字段连续存储在 `Vec` 中，通过 `Deref` 到 `CasedRecord<CaseSensitive>` 提供大小写敏感的列查找（[record.rs#L130-L136](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/value/record.rs#L130-L136)）
2. **列查找是 O(n) 线性扫描**：`CasedRecord::index_of` 使用 `rposition`（[record.rs#L49-L52](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/value/record.rs#L49-L52)），用 `rposition` 而非 `position` 是因为 `push` 可能重复键，取最后一个
3. **Record 的 `push` 允许重复键**（[record.rs#L353-L355](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/value/record.rs#L353-L355)），但 `insert` 会替换（[record.rs#L77-L88](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/value/record.rs#L77-L88)）。`select` 使用 `record.push` 构建新 Record，理论上可能产生重复列名
4. **`get_columns` 提取列名时使用 `HashSet` 去重**（[column.rs#L4-L20](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-engine/src/column.rs#L4-L20)），遇到非 Record 元素直接返回空列表

**`Value::List`（table）与 `Value::Record` 在字段选择上的结构性差异**：

| 维度 | `Value::Record` | `Value::List`（table，即 `Vec<Record>`） |
|------|-----------------|--------------------------------------|
| 字段存储 | 单行：`Vec<(String, Value)>` | 多行：每行是独立的 `Record` |
| `get name` | 返回单值 | 返回所有行的 `name` 值组成的 `List` |
| `select name` | 返回单行 Record | 逐行构建 Record，返回 table |
| `columns` | 直接迭代 `Record::columns()` | 遍历所有行，`HashSet` 去重合并列名 |
| `values` | 直接 `Record::values()` | 按**列优先**重组（每列的值组成一个 List） |
| 错误检查粒度 | 整个 Record | 逐元素（`get_values` 遇 Error 提前终止） |

### 7.10 `select` 对行选择与列选择的分流

`select` 是唯一同时支持行选择和列选择的命令，它通过 `NthIterator` 和 `follow_cell_path` 两条路径实现：

**行选择**（整数 cell path，如 `select 0 2`）：

```rust
// [select.rs#L258-L274] 行选择 → NthIterator 流式跳过
let pipeline_iter: PipelineIterator = input.into_iter();
NthIterator {
    input: pipeline_iter,
    rows: unique_rows.into_iter().peekable(),
    current: 0,
}
.into_pipeline_data_with_metadata(call_span, engine_state.signals().clone(), metadata)
```

`NthIterator`（[select.rs#L367-L393](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-command/src/filters/select.rs#L367-L393)）的逻辑：
- 维护 `current` 计数器和 `rows` 有序集合的 peek 迭代器
- 对每个输入元素：如果 `current == *row`，产出该元素并推进 `rows`；否则丢弃
- 当 `rows` 耗尽后，返回 `None` 停止迭代——**不消费剩余输入**

**列选择**（字符串 cell path，如 `select name size`）：

在行选择之后的 input 上，对每个元素调用 `follow_cell_path` 提取指定字段，构建新 Record。

**行+列混合**（如 `select 0 name size`）：

先通过 `NthIterator` 选择行，再对筛选后的元素做列选择。两条路径都**保持流式**。

**与 `get` 的行选择对比**：`get 0` 通过 `PipelineData::follow_cell_path` 收集整个流后索引，而 `select 0` 通过 `NthIterator` 流式跳过。对于大型流，`select 0` 远比 `get 0` 高效。

### 7.11 完整协作链数据流图

```
┌───────────────────────────────────────────────────────────────────────────┐
│                         Pipeline 数据流全景                                 │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ls / open / 外部命令                                                      │
│    │                                                                      │
│    ├── PipelineData::ListStream ───────────────────────────────────┐     │
│    │     │                                                          │     │
│    │     ├── get (单路径, 列名) ── 流式保留 ──→ ListStream           │     │
│    │     ├── get (单路径, 行号) ── 全量收集 ──→ Value                │     │
│    │     ├── get (多路径)      ── 全量收集 ──→ ListStream            │     │
│    │     │                                                          │     │
│    │     ├── select (列名)     ── 流式保留 ──→ ListStream           │     │
│    │     ├── select (行号)     ── 流式跳过 ──→ ListStream           │     │
│    │     ├── select (行+列)    ── 流式保留 ──→ ListStream           │     │
│    │     │                                                          │     │
│    │     ├── where             ── 流式保留 ──→ ListStream           │     │
│    │     ├── each              ── 流式保留 ──→ ListStream           │     │
│    │     │                                                          │     │
│    │     ├── columns           ── 全量收集 ──→ Value::List          │     │
│    │     └── values            ── 全量收集 ──→ Value::List          │     │
│    │                                                                │     │
│    ├── PipelineData::Value(Value::Record) ────────────────────┐     │     │
│    │     │                                                       │     │     │
│    │     ├── get (单路径)     ── 零收集  ──→ Value (裸值)        │     │
│    │     ├── select          ── 零收集  ──→ Value (Record)         │     │
│    │     ├── columns         ── 零收集  ──→ Value::List          │     │
│    │     ├── values          ── 克隆值  ──→ Value::List          │     │
│    │     ├── where           ── 不适用  (Record 不是列表)        │     │
│    │     └── each            ── 单次执行 ──→ 闭包返回值          │     │
│    │                                                             │     │
│    └── PipelineData::ByteStream ────────────────────────┐      │     │
│          │                                                │      │     │
│          ├── each             ── chunks() 流式 ──→ ListStream           │     │
│          ├── where            ── chunks() 流式 ──→ ListStream           │     │
│          ├── get              ── Err(IncompatiblePathAccess)               │     │
│          ├── select           ── 静默空 (fallthrough → empty)                   │     │
│          └── columns/values   ── Err(OnlySupportsThisInputType)              │     │
│                                                                         │
│  ByteStream chunks 的错误传播:                                                   │
│    Binary type: 始终生成 Value::Binary 块                                        │
│    String type: 生成 Value::String，UTF-8 失败 → Err 停止                  │
│    Unknown type: 先试 String，UTF-8 失败 → 永久切换到 Binary 模式              │
│    IO 错误: 设置 error 标记，产出 Err(ShellError::Io)                              │
│    信号中断: 立即停止，产出 None                                              │
│    chunks Item: 全部经过 PipelineIterator → 包装为 Value::error                  │
│                                                                         │
│  Value::Error 传播总览:                                                      │
│    Value 中 → Err 短路 (columns/values/select(record分支)                         │
│    Stream 中 → 嵌入流延迟 (get单路径/select/each/where)                    │
│    闭包失败 → Value::error 嵌入流 (each/where)                               │
│    cell path 失败 → Value::error 嵌入流 (get单路径/select)                  │
│    cell path 失败 → Err 中断 (get多路径)                                    │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

### 7.12 ByteStream `chunks()` 迭代器的详细行为

`chunks()` 定义于 [byte_stream.rs#L542-L545](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs#L542-L545)，是 ByteStream 与值流之间的桥梁：

```rust
pub fn chunks(self) -> Option<Chunks> {
    let reader = self.stream.reader()?;
    Some(Chunks::new(reader, self.span, self.signals, self.type_))
}
```

`Chunks` 迭代器（`impl Iterator for Chunks`）的 `type Item = Result<Value, ShellError>`。不同 `ByteStreamType` 的行为差异：

| ByteStreamType | 产出值类型 | UTF-8 失败行为 | 错误后是否停止 | 信号中断行为 |
|----------------|-----------|--------------|------------|------------|
| `Binary` | `Value::Binary`（字节块） | N/A | 是（`self.error = true`） | 是（`signals.interrupted()` → None） |
| `String` | `Value::String`（UTF-8 文本块） | `Err(NonUtf8Custom)`，**永久停止** | 是 | 是 |
| `Unknown` | 先 `Value::String`，失败后切换到 `Value::Binary` | 首块非空 → **切换到 Binary 模式**继续；首块空或后续块错误 → 停止 | 是（彻底失败后） | 是 |

**关键行为细节**：

1. **Binary 模式**：通过 `reader.fill_buf()` 读取可用字节，产出 `Value::binary(buf, span)`，消费后继续。每次 `next()` 返回的字节块大小取决于内核缓冲区中的可用字节数。

2. **String 模式**：调用 `self.next_string()`（[byte_stream.rs#L1066-L1072](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs#L1066-L1072)），遇到 UTF-8 错误时设置 `self.error = true`，下一次 `next()` 直接返回 `None`。

3. **Unknown 模式**（最复杂）：先尝试 `next_string()`，成功则产出 `Value::String`；失败但缓冲区非空时，**不可逆地切换到 `ByteStreamType::Binary`**（[byte_stream.rs#L1075-L1088](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs#L1075-L1088)），将当前缓冲区内容作为 `Value::Binary` 产出，之后按 Binary 模式继续。这意味着同一个 Chunks 迭代器可能先产出 String 后产出 Binary，类型不一致。

4. **错误传播链**：

```
ByteStream::chunks() → Chunks::next() → Result<Value, ShellError>
  ↓ (PipelineIterator::next())
  Value::error(err, Span::unknown())  // FIXME: 包装为 Value::Error 嵌入流
  ↓ (where 命令：into_iter_strict 内部转换)
  filter_map 中继续传递
  ↓ (each 命令：显式 chunks() 调用)
  chunks.map(and_then...).unwrap_or_else
  ↓ (消费点：into_value / drain / collect_reg)
  Err(ShellError)  // 提升为结构化错误
```

`PipelineIterator::next()` 中的转换（[pipeline_data.rs#L1013-L1031](file:///d:/fz/0601-2/solo-dogfeeding/code/65-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L1013-L1031)）将 Chunks 的 `Result` 错误包装为 `Value::Error` 嵌入流中，使用 `Value::error(err, Span::unknown()) // FIXME`，注释表明 span 信息丢失了。
