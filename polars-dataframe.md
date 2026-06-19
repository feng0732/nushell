# Polars 数据帧与普通值的转换边界分析

## 一、核心架构概览

Polars 插件通过 **CustomValue** 机制将 Polars 的数据结构集成到 Nushell 的值系统中。所有 Polars 对象（DataFrame、LazyFrame、Expression 等）都作为自定义值（Custom Value）存在，与 Nushell 原生 Value 之间存在明确的转换边界。

### 1.1 自定义值类型体系

在 [values/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/mod.rs) 中定义了完整的 Polars 插件对象类型枚举：

| 类型 | 枚举变体 | 说明 |
|------|----------|------|
| `NuDataFrame` | `PolarsPluginObject::NuDataFrame` | 已物化的即时数据帧 |
| `NuLazyFrame` | `PolarsPluginObject::NuLazyFrame` | 延迟求值的惰性数据帧 |
| `NuExpression` | `PolarsPluginObject::NuExpression` | 表达式对象 |
| `NuLazyGroupBy` | `PolarsPluginObject::NuLazyGroupBy` | 延迟分组对象 |
| `NuWhen` | `PolarsPluginObject::NuWhen` | When 表达式 |
| `NuDataType` | `PolarsPluginObject::NuDataType` | 数据类型对象 |
| `NuSchema` | `PolarsPluginObject::NuSchema` | 数据模式对象 |
| `NuSelector` | `PolarsPluginObject::NuSelector` | 选择器对象 |

所有这些类型都通过 `CustomValueSupport` trait 统一管理转换逻辑，该 trait 定义在 [values/mod.rs#L360-L468](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/mod.rs#L360-L468)。

---

## 二、NuDataFrame：即时数据帧

### 2.1 数据结构

在 [nu_dataframe/mod.rs#L104-L109](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/nu_dataframe/mod.rs#L104-L109) 中定义了 `NuDataFrame`：

```rust
pub struct NuDataFrame {
    pub id: Uuid,           // 唯一标识符，用于缓存
    pub df: Arc<DataFrame>, // Polars DataFrame，Arc 共享引用
    pub from_lazy: bool,    // 是否来自 lazy frame
}
```

关键设计要点：
- 使用 `Arc<DataFrame>` 实现数据的共享引用，避免深拷贝
- 每个 `NuDataFrame` 都有唯一的 `Uuid`，用于插件内部缓存
- `from_lazy` 标记记录数据帧来源，用于 `cache_and_to_value` 时的类型回退

### 2.2 CustomValue 封装

在 [nu_dataframe/custom_value.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/nu_dataframe/custom_value.rs) 中，`NuDataFrameCustomValue` 实现了 `CustomValue` trait：

```rust
pub struct NuDataFrameCustomValue {
    pub id: Uuid,
    #[serde(skip)]
    pub dataframe: Option<NuDataFrame>,
}
```

重要特性：
- `dataframe` 字段被标记为 `#[serde(skip)]`，序列化时只保留 `id`
- 反序列化后，通过 `id` 从插件缓存中恢复实际数据
- `notify_plugin_on_drop()` 返回 `true`，确保值销毁时通知插件清理缓存

### 2.3 Value → NuDataFrame 的转换

#### 2.3.1 从迭代器创建（最常用路径）

[try_from_iter](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/nu_dataframe/mod.rs#L158-L200) 方法是从 Nushell 值列表创建数据帧的核心入口：

```rust
pub fn try_from_iter<T>(
    plugin: &PolarsPlugin,
    iter: T,
    maybe_schema: Option<NuSchema>,
    span: Span,
) -> Result<Self, ShellError>
```

支持三种输入值类型：

| 输入类型 | 处理方式 |
|----------|----------|
| `Value::Custom` | 已有的 Polars 对象，通过 `try_from_value_coerce` 强制转换 |
| `Value::List` | 将列表转为记录，索引作为列名（0, 1, 2...） |
| `Value::Record` | 记录的字段名作为列名 |
| 其他（Int、Float 等） | 单列数据，列名为 "0" |

#### 2.3.2 列值累积与类型推断

转换过程中使用 `ColumnMap`（即 `IndexMap<PlSmallStr, TypedColumn>`）累积列数据。[insert_value](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/nu_dataframe/conversion.rs#L205-L244) 函数处理类型推断逻辑：

1. **有 schema 时**：直接使用 schema 指定的列类型
2. **无 schema 时**：
   - 第一行值决定列的初始类型（通过 `value_to_data_type` 推断）
   - 后续行如果类型不一致，则降级为 `DataType::Object("Value")`
   - Object 类型使用 `DataFrameValue` 包装原始 `Value`，保留 Nushell 值的原始语义

#### 2.3.3 value_to_data_type 映射

[value_to_data_type](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/nu_dataframe/conversion.rs#L246-L278) 定义了默认类型映射：

| Nushell Value 类型 | Polars DataType |
|-------------------|-----------------|
| `Value::Int` | `DataType::Int64` |
| `Value::Float` | `DataType::Float64` |
| `Value::String` | `DataType::String` |
| `Value::Bool` | `DataType::Boolean` |
| `Value::Date` | `DataType::Datetime(Nanoseconds, Some(UTC))` |
| `Value::Duration` | `DataType::Duration(Nanoseconds)` |
| `Value::Filesize` | `DataType::Int64` |
| `Value::Binary` | `DataType::Binary` |
| `Value::List` | `DataType::List(inner_type)` |
| 其他 | `None` → 最终为 Object 类型 |

### 2.4 NuDataFrame → Value 的转换

#### 2.4.1 基础值转换

[base_value](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/nu_dataframe/mod.rs#L554-L557) 方法将数据帧转换为 Nushell 列表：

```rust
fn base_value(self, span: Span) -> Result<Value, ShellError> {
    let vals = self.print(true, span)?;
    Ok(Value::list(vals, span))
}
```

输出格式：`Value::List`，其中每个元素是 `Value::Record`，包含各列数据。

#### 2.4.2 to_rows：逐行转换

[to_rows](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/nu_dataframe/mod.rs#L387-L436) 是核心转换函数，将数据帧的指定行范围转换为 Nushell 值列表。

转换流程：
1. 对每列调用 `create_column` → `create_column_from_series` → `series_to_values`
2. 将各列的迭代器按行组合为 Record
3. 可选添加 index 列

#### 2.4.3 series_to_values：Series → Vec\<Value\>

[series_to_values](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/nu_dataframe/conversion.rs#L798-L1347) 是最庞大的转换函数，按 Polars 数据类型分别处理：

| Polars 类型 | Nushell Value 类型 | 注意事项 |
|-------------|-------------------|----------|
| `UInt8..UInt64` | `Value::Int` | 统一转为 i64 |
| `Int8..Int64` | `Value::Int` | 统一转为 i64 |
| `Float32/Float64` | `Value::Float` | 统一转为 f64 |
| `Boolean` | `Value::Bool` | - |
| `String` | `Value::String` | - |
| `Binary` | `Value::Binary` | - |
| `Date` | `Value::Date` | 从物理 i32（天）转为日期时间 |
| `Datetime` | `Value::Date` | 处理时区和时间单位 |
| `Duration` | `Value::Duration` | 处理时间单位转换 |
| `List` | `Value::List` | 递归转换子元素 |
| `Struct` | `Value::Record` | 结构体字段转为记录字段 |
| `Object` | 原始 `Value` | 通过 `DataFrameValue::get_value()` 还原 |
| `Decimal` | `Value::Float` | 先 cast 为 Float64 再转换 |
| `Categorical/Enum` | `Value::List<String>` | 返回类别列表 |
| `Null` | `Value::Nothing` | - |

---

## 三、Lazy 求值机制

### 3.1 NuLazyFrame 结构

在 [nu_lazyframe/mod.rs#L21-L26](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/nu_lazyframe/mod.rs#L21-L26) 中定义：

```rust
pub struct NuLazyFrame {
    pub id: Uuid,              // 唯一标识符
    pub lazy: Arc<LazyFrame>,  // Polars LazyFrame
    pub from_eager: bool,      // 是否来自即时数据帧
}
```

与 `NuDataFrame` 类似，但包含的是 `LazyFrame` 而非 `DataFrame`。

### 3.2 Lazy → Eager 的转换（收集）

[collect](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/nu_lazyframe/mod.rs#L58-L74) 方法触发实际计算：

```rust
pub fn collect(self, span: Span) -> Result<NuDataFrame, ShellError> {
    crate::handle_panic(
        || {
            self.to_polars()
                .collect()
                .map_err(|e| ShellError::Generic(...))
                .map(|df| NuDataFrame::new(true, df))
        },
        span,
    )
}
```

关键点：
- 使用 `handle_panic` 包装，防止 Polars panic 导致崩溃
- 收集后的数据帧标记 `from_lazy = true`

### 3.3 Eager → Lazy 的转换

[lazy()](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/nu_dataframe/mod.rs#L143-L145) 方法：

```rust
pub fn lazy(&self) -> NuLazyFrame {
    NuLazyFrame::new(true, self.to_polars().lazy())
}
```

转换后 `from_eager = true`，记录来源。

### 3.4 自动类型回退

`CustomValueSupport` trait 的 [cache_and_to_value](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/mod.rs#L433-L452) 方法实现了智能类型回退：

```rust
fn cache_and_to_value(self, plugin, engine, span) -> Result<Value, ShellError> {
    match self.to_cache_value()? {
        // 如果 DataFrame 来自 Lazy，转回 LazyFrame
        PolarsPluginObject::NuDataFrame(df) if df.from_lazy => {
            let df = df.lazy();
            Ok(df.cache(plugin, engine, span)?.into_value(span))
        }
        // 如果 LazyFrame 来自 Eager，转回 DataFrame
        PolarsPluginObject::NuLazyFrame(lf) if lf.from_eager => {
            let lf = lf.collect(span)?;
            Ok(lf.cache(plugin, engine, span)?.into_value(span))
        }
        _ => Ok(self.cache(plugin, engine, span)?.into_value(span)),
    }
}
```

这意味着操作后的值类型会自动保持原始输入的类型（Lazy 保持 Lazy，Eager 保持 Eager）。

### 3.5 Schema 的惰性获取

LazyFrame 可以在不收集数据的情况下获取 schema：

```rust
pub fn schema(&mut self) -> Result<NuSchema, ShellError> {
    let internal_schema = Arc::make_mut(&mut self.lazy)
        .collect_schema()
        .map_err(|e| ...)?;
    Ok(internal_schema.into())
}
```

通过 `collect_schema()` 仅推导模式，不执行完整计算。

---

## 四、列类型系统

### 4.1 NuDataType 封装

在 [nu_dtype/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/nu_dtype/mod.rs) 中定义了 `NuDataType`，包装 Polars 的 `DataType`：

```rust
pub struct NuDataType {
    pub id: uuid::Uuid,
    dtype: DataType,
}
```

### 4.2 字符串 → DataType 解析

[str_to_dtype](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/nu_dtype/mod.rs#L147-L286) 函数支持以下类型字符串的解析：

| 类型字符串 | Polars 类型 |
|-----------|-------------|
| `bool` | `Boolean` |
| `u8/u16/u32/u64` | `UInt8/16/32/64` |
| `i8/i16/i32/i64` | `Int8/16/32/64` |
| `f32/f64` | `Float32/64` |
| `str` | `String` |
| `binary` | `Binary` |
| `date` | `Date` |
| `time` | `Time` |
| `null` | `Null` |
| `unknown` | `Unknown` |
| `object` | `Object("unknown")` |
| `list<dtype>` | `List(Box::new(dtype))` |
| `datetime<tu, tz>` | `Datetime(time_unit, timezone)` |
| `duration<tu>` | `Duration(time_unit)` |
| `decimal<precision, scale>` | `Decimal(precision, scale)` |

### 4.3 Schema 的作用

NuSchema 定义在 [nu_schema/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/nu_schema/mod.rs)：

```rust
pub struct NuSchema {
    pub id: Uuid,
    pub schema: SchemaRef,
}
```

Schema 在转换中的作用：
1. **强制列类型**：指定列应使用的 Polars 数据类型，覆盖默认推断
2. **过滤列**：只有 schema 中存在的列才会被包含在数据帧中
3. **补全缺失列**：`add_missing_columns` 函数会为 schema 中存在但数据中没有的列填充 null 值

---

## 五、导出流程分析

### 5.1 Save 命令总览

`polars save` 命令定义在 [command/core/save/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/command/core/save/mod.rs)，支持多种文件格式：

| 格式 | Eager 模式 | Lazy 模式（Sink） |
|------|-----------|-------------------|
| Parquet | ✅ | ✅ |
| Arrow/IPC | ✅ | ✅ |
| CSV | ✅ | ✅ |
| NDJSON | ✅ | ✅ |
| Avro | ✅ | ❌（需先 collect） |

### 5.2 两种保存模式

#### 5.2.1 Eager 模式（即时数据帧）

以 CSV 为例，[csv::command_eager](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/command/core/save/csv.rs#L56-L110)：

```rust
pub(crate) fn command_eager(call, df, resource) -> Result<(), ShellError> {
    let mut file = File::create(file_path)?;
    let writer = CsvWriter::new(&mut file);
    // 配置 writer...
    writer.finish(&mut df.to_polars())?;
    Ok(())
}
```

特点：
- 直接使用 Polars 的 `CsvWriter` 写入文件
- 数据已在内存中，直接序列化
- 支持本地文件系统

#### 5.2.2 Lazy 模式（Sink 流式保存）

以 CSV 为例，[csv::command_lazy](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/command/core/save/csv.rs#L16-L54)：

```rust
pub(crate) fn command_lazy(call, lazy, resource) -> Result<(), ShellError> {
    lazy.to_polars()
        .sink(
            resource.into(),
            FileWriteFormat::Csv(options),
            UnifiedSinkArgs {
                cloud_options: resource.cloud_options.map(Arc::new),
                ..Default::default()
            },
        )
        .and_then(|l| l.collect())
        .map_err(|e| polars_file_save_error(e, file_span))
}
```

特点：
- 使用 `sink` 操作，流式写出数据，内存效率更高
- 支持云存储（S3 等）
- `sink` 返回的 LazyFrame 仍需 `collect()` 触发执行

### 5.3 Cloud URL 特殊处理

在保存逻辑中，有一个重要的分支：如果是云存储 URL，即使是 Eager DataFrame 也会转为 Lazy 模式保存：

```rust
PolarsPluginObject::NuDataFrame(ref df) if resource.cloud_options.is_some() => {
    parquet::command_lazy(call, &df.lazy(), resource)
}
```

这是因为云存储写入只能通过 Sink API 完成。

### 5.4 源文件保护

[check_writing_into_source_file](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/command/core/save/mod.rs#L247-L260) 防止将数据写回源文件：

```rust
fn check_writing_into_source_file(metadata, dest) -> Result<(), ShellError> {
    let Some(DataSource::FilePath(source)) = metadata.map(|meta| &meta.data_source) else {
        return Ok(());
    };
    if &dest.item == source {
        return Err(write_into_source_error(dest.span));
    }
    Ok(())
}
```

通过 PipelineMetadata 中的 DataSource 追踪数据来源。

---

## 六、值运算的转换边界

### 6.1 数据帧与普通值的运算

`compute_with_value` 方法（定义在 [operations.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/nu_dataframe/operations.rs)）处理数据帧与右侧值的运算：

| 右侧值类型 | 处理方式 |
|-----------|----------|
| `Value::Custom`（数据帧/Series） | 两个数据帧/Series 之间的运算 |
| 普通值（Int、Float、String、Date） | 单列 Series 与标量的运算 |

### 6.2 标量与 Series 的运算

[compute_series_single_value](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/nu_dataframe/between_values.rs#L207-L453) 处理单列与标量的运算，支持：

- **数学运算**：Add、Subtract、Multiply、Divide
- **比较运算**：Equal、NotEqual、LessThan、GreaterThan 等
- **布尔运算**：And、Or（仅布尔列）
- **字符串运算**：字符串拼接、正则匹配等

类型不匹配时会尝试 cast，例如 Int32 列与 i64 运算时先转为 Int64。

---

## 七、转换边界总结

### 7.1 核心转换路径图

```
┌─────────────────────────────────────────────────────────────┐
│                    Nushell Value 系统                       │
│                                                             │
│  Value::Int  Value::Float  Value::String  Value::Record    │
│  Value::List  Value::Bool   Value::Date   Value::Custom    │
│                                              (透明黑盒)     │
└───────────────────┬─────────────────────┬──────────────────┘
                    │                     │
        try_from_iter()/try_from_value()  │
                    │                     │
                    ▼                     ▼
┌─────────────────────────────────────────────────────────────┐
│                    Conversion 层                            │
│                                                             │
│  value_to_data_type()  →  类型推断/降级                    │
│  typed_column_to_series() → 构建 Polars Series             │
│  series_to_values()  ←  类型映射转换                       │
│  DataFrameValue (Object 类型)  ↔  原始 Value 保留          │
└───────────────────┬─────────────────────┬──────────────────┘
                    │                     │
                    ▼                     ▼
┌─────────────────────────────────────────────────────────────┐
│                    Polars 世界                              │
│                                                             │
│  DataFrame ◄─── collect() ──── LazyFrame                   │
│      │                          │                           │
│      └── Series / Columns      └── Logical Plan            │
│          (Int64, Float64, ...)       (Expr, Schema)        │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 关键边界原则

1. **CustomValue 是黑盒边界**：Nushell 引擎将 Polars 对象视为不透明自定义值，只有插件能理解其内部结构

2. **类型降级策略**：当列中存在不一致类型时，不会自动强制转换，而是降级为 `Object` 类型（包装原始 Value）

3. **Schema 优先原则**：显式指定的 schema 覆盖类型推断，决定列的最终类型

4. **Lazy/Eager 透明转换**：`try_from_value_coerce` 和 `cache_and_to_value` 提供了自动转换机制

5. **Object 类型兜底**：所有无法映射到 Polars 原生类型的值都可以通过 `DataFrameValue` 包装存储，但失去 Polars 的向量化操作能力

6. **导出即物化**：LazyFrame 导出到文件时，通过 sink 操作流式执行；Eager DataFrame 直接序列化

### 7.3 常见转换场景

| 场景 | 转换方向 | 关键函数 |
|------|----------|----------|
| Nushell 表 → DataFrame | Value → DF | `try_from_iter` / `insert_value` |
| DataFrame → Nushell 表 | DF → Value | `to_rows` / `series_to_values` |
| DataFrame → LazyFrame | Eager → Lazy | `df.lazy()` |
| LazyFrame → DataFrame | Lazy → Eager | `lazy.collect()` |
| Nushell 值 → 列类型 | Value → DataType | `str_to_dtype` / `value_to_data_type` |
| Polars 类型 → Nushell 值 | DataType → Value | `dtype_to_value` |
| 保存到文件 | DF/LF → 文件 | `save` 命令（eager/sink） |
| 读取文件 → LazyFrame | 文件 → LF | `polars open`（默认 lazy 加载） |

---

## 八、代码参考索引

| 模块 | 文件路径 | 主要内容 |
|------|----------|----------|
| 数据帧核心 | [nu_dataframe/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/nu_dataframe/mod.rs) | NuDataFrame 结构及方法 |
| 自定义值封装 | [nu_dataframe/custom_value.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/nu_dataframe/custom_value.rs) | CustomValue trait 实现 |
| 类型转换 | [nu_dataframe/conversion.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/nu_dataframe/conversion.rs) | Value ↔ Series 转换 |
| 值间运算 | [nu_dataframe/between_values.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/nu_dataframe/between_values.rs) | 数据帧运算逻辑 |
| LazyFrame | [nu_lazyframe/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/nu_lazyframe/mod.rs) | NuLazyFrame 结构及方法 |
| 数据类型 | [nu_dtype/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/nu_dtype/mod.rs) | NuDataType 及类型字符串解析 |
| 数据模式 | [nu_schema/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/nu_schema/mod.rs) | NuSchema 及转换 |
| 统一值类型 | [values/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/mod.rs) | PolarsPluginObject、CustomValueSupport |
| 文件类型 | [values/file_type.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/values/file_type.rs) | PolarsFileType 枚举 |
| 保存命令 | [command/core/save/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/command/core/save/mod.rs) | SaveDF 命令主逻辑 |
| CSV 保存 | [command/core/save/csv.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/command/core/save/csv.rs) | CSV eager/lazy 保存 |
| 转 Nushell 值 | [command/core/to_nu.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/command/core/to_nu.rs) | polars into-nu 命令 |
| 转 Lazy | [command/core/to_lazy.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/74-nushell/crates/nu_plugin_polars/src/dataframe/command/core/to_lazy.rs) | polars into-lazy 命令 |
