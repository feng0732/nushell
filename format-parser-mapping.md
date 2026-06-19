# 结构化格式转 Nushell 内部值的规则梳理

本文档从代码实现角度梳理 Nushell 中各种结构化格式（JSON、YAML、TOML、NUON、XML、CSV、KDL、MessagePack 等）解析为内部 `Value` 类型时的类型映射、数字精度和解析策略。

---

## 一、Nushell 内部 Value 类型体系

所有格式解析的目标都是构建 [`Value`](crates/nu-protocol/src/value/mod.rs#L77-L219) 枚举实例。核心原子类型如下：

| Value 变体 | 内部表示 | 说明 |
|---|---|---|
| `Bool` | `bool` | 布尔值 |
| `Int` | `i64` | **有符号 64 位整数**（所有整数最终收敛到此） |
| `Float` | `f64` | **IEEE 754 双精度浮点数** |
| `String` | `String` | UTF-8 字符串 |
| `Nothing` | 单元值 | 对应 null/nil |
| `Date` | `DateTime<FixedOffset>` | 带时区的日期时间（chrono） |
| `Duration` | `i64`（纳秒） | 时间长度，单位为纳秒 |
| `Filesize` | `Filesize`（内部 i64） | 文件大小 |
| `Binary` | `Vec<u8>` | 二进制字节数组 |
| `Record` | `SharedCow<Record>` | 有序键值对（IndexMap 语义） |
| `List` | `Vec<Value>` | 值列表 |

> **关键约束**：Nushell 只有单一整数类型 `i64` 和单一浮点类型 `f64`，没有 `u64`、`i32`、`f32` 等细分类型。

---

## 二、JSON（`from json`）

### 2.1 代码位置

- 命令实现：[`crates/nu-command/src/formats/from/json.rs`](crates/nu-command/src/formats/from/json.rs)
- JSON/Hjson 值类型：[`crates/nu-json/src/value.rs`](crates/nu-json/src/value.rs#L50-L74)
- nu_json::Value → NuValue 转换：[`crates/nu-json/src/nu_value.rs`](crates/nu-json/src/nu_value.rs#L9-L33)

### 2.2 两种解析模式

| 模式 | 开关 | 解析库 | 特性 |
|---|---|---|---|
| **宽松模式**（默认） | 无 flag | `nu_json`（Hjson 兼容） | 支持注释、尾随逗号、无引号键、多行字符串等 |
| **严格模式** | `--strict` / `-s` | `serde_json` | 严格遵循 JSON 规范 |

### 2.3 类型映射表

| JSON 类型 | nu_json 中间类型 | Nu Value | 精度与规则 |
|---|---|---|---|
| `null` | `Value::Null` | `Value::Nothing` | — |
| `true` / `false` | `Value::Bool(bool)` | `Value::Bool(bool)` | — |
| **负整数** | `Value::I64(i64)` | `Value::Int(i64)` | 直接使用 i64 |
| **非负整数** | `Value::U64(u64)` | `Value::Int(i64)` 或 `Value::Float(f64)` | 若 `u64` ≤ `i64::MAX` 则转 `Int`，否则降为 `Float`（精度丢失！） |
| 浮点数 / 科学计数 | `Value::F64(f64)` | `Value::Float(f64)` | f64 原生精度 |
| 字符串 | `Value::String(String)` | `Value::String(String)` | — |
| 数组 | `Value::Array(Vec<Value>)` | `Value::List(Vec<Value>)` | 递归转换 |
| 对象 | `Value::Object(Map<String, Value>)` | `Value::Record(Record)` | 键必须为字符串；默认启用 `preserve_order` feature，使用 `LinkedHashMap` 保持插入顺序 |

### 2.4 关键策略

1. **U64 溢出降级**：`u64 > i64::MAX`（即 ≥ 9223372036854775808）时强制转为 `f64`，大整数高位可能丢失。见 [nu_value.rs#L17-L20](crates/nu-json/src/nu_value.rs#L17-L20)。
2. **NDJSON 支持**：`--objects` 逐行解析，空行跳过，每行错误包装为 `Value::Error`。
3. **对象键顺序**：[`nu-json/Cargo.toml#L21`](crates/nu-json/Cargo.toml#L21-L21) 中 `default = ["preserve_order"]`，默认启用，使用 `LinkedHashMap` 保持插入顺序。

---

## 三、YAML（`from yaml` / `from yml`）

### 3.1 代码位置

- 命令与转换：[`crates/nu-command/src/formats/from/yaml.rs`](crates/nu-command/src/formats/from/yaml.rs)
- 核心函数：`convert_yaml_value_to_nu_value` [L42-L153](crates/nu-command/src/formats/from/yaml.rs#L42-L153)

### 3.2 类型映射表

| YAML 类型（serde_yaml::Value） | Nu Value | 规则 |
|---|---|---|
| `Bool(bool)` | `Value::Bool(bool)` | — |
| `Number`（is_i64 为 true） | `Value::Int(i64)` | 调用 `as_i64()` 提取 |
| `Number`（is_f64 为 true） | `Value::Float(f64)` | 调用 `as_f64()` 提取 |
| `Number`（其他，如 u64 溢出） | 报错 `UnsupportedInput` | 不支持的数字直接失败 |
| `String(String)` | `Value::String(String)` | — |
| `Sequence(Vec<Value>)` | `Value::List(Vec<Value>)` | 递归 |
| `Mapping(Map)` | `Value::Record(Record)` | 使用 `IndexMap` 保证确定性顺序 |
| `Tagged(Tag, Value)` | 通常为 `Value::String` | 标签前缀拼接到值前，如 `"!Value ${TEST}"` |
| `Null` | `Value::Nothing` | — |

### 3.3 关键策略

1. **数字判定顺序**：先试 `is_i64()` → 再试 `is_f64()`；两者都不匹配则报错。
2. **多文档支持**：YAML 流中多个文档（`---` 分隔）返回 `Value::List`，单个文档直接返回该值，空文档返回 `Nothing`。
3. **Mapping 键非字符串**：键为 `Number` 或 `Bool` 时调用 `to_string()` 序列化；非基础键报错。
4. **特殊 hack**：未加引号的 `{{ something }}` 被 serde_yaml 解析为嵌套 Mapping → Null，代码会特殊还原为字符串 `"{{ something }}"`。见 [L100-L118](crates/nu-command/src/formats/from/yaml.rs#L100-L118)。
5. **Tagged 值**：YAML 标签（如 `!Tag`）与内容用空格拼接成字符串，不做语义解析。

---

## 四、TOML（`from toml`）

### 4.1 代码位置

- 命令与转换：[`crates/nu-command/src/formats/from/toml.rs`](crates/nu-command/src/formats/from/toml.rs)
- 日期转换：`convert_toml_datetime_to_value` [L60-L106](crates/nu-command/src/formats/from/toml.rs#L60-L106)

### 4.2 类型映射表

| TOML 类型（toml::Value） | Nu Value | 规则 |
|---|---|---|
| `Boolean(bool)` | `Value::Bool(bool)` | — |
| `Integer(i64)` | `Value::Int(i64)` | TOML 整数原生就是 i64 |
| `Float(f64)` | `Value::Float(f64)` | TOML 浮点原生就是 f64 |
| `String(String)` | `Value::String(String)` | — |
| `Array(Vec<Value>)` | `Value::List(Vec<Value>)` | 递归 |
| `Table(Map<String, Value>)` | `Value::Record(Record)` | 键为字符串，保持插入顺序 |
| `Datetime(Datetime)` | `Value::Date` 或 `Value::String` | 见 4.3 |

### 4.3 日期时间策略

TOML 日期时间的转换条件（**必须包含 date 部分**）：

| 组合 | 转换结果 | 说明 |
|---|---|---|
| date + time + offset | `Value::Date`（带时区） | 完整日期时间 |
| date + time（无 offset） | `Value::Date`（UTC +0） | 默认按零时区处理 |
| date only（无 time） | `Value::Date`（00:00:00 UTC） | 时间部分补零 |
| time only（无 date） | `Value::String`（原始字符串） | **不支持独立时间**，保留字符串形式 |
| offset: Z | `FixedOffset::east_opt(0)` | 等于 UTC |
| offset: Custom { minutes } | `FixedOffset::east_opt(minutes * 60)` | 正/负分钟数转换为秒偏移 |

> 注意：TOML 顶层一定是 Table，因此 `from toml` 的输出类型总是 `Record`（签名中声明为 `Type::record()`）。

---

## 五、NUON（`from nuon`）

### 5.1 代码位置

- 命令层：[`crates/nu-command/src/formats/from/nuon.rs`](crates/nu-command/src/formats/from/nuon.rs)
- 核心解析器：[`crates/nuon/src/from.rs`](crates/nuon/src/from.rs)
  - `from_nuon` [L19-L116](crates/nuon/src/from.rs#L19-L116)
  - `convert_to_value` [L129-L479](crates/nuon/src/from.rs#L129-L479)

### 5.2 解析原理

NUON 直接复用 Nushell 的 **语法解析器** `nu_parser::parse()` 将输入解析为 AST（`Expression`），然后从 AST 提取单个表达式转换为 `Value`。

**限制**：只允许单个 pipeline 元素。多余的 pipeline、多个语句、子表达式、闭包、变量等都会报错。

### 5.3 类型映射表

| NUON AST 表达式 | Nu Value | 规则 |
|---|---|---|
| `Expr::Bool` | `Value::Bool` | — |
| `Expr::Int(i64)` | `Value::Int(i64)` | — |
| `Expr::Float(f64)` | `Value::Float(f64)` | — |
| `Expr::String` / `Expr::RawString` | `Value::String` | 两种字面量归一为 String |
| `Expr::Nothing` | `Value::Nothing` | — |
| `Expr::Binary(Vec<u8>)` | `Value::Binary` | 二进制字面量 |
| `Expr::DateTime` | `Value::Date` | 日期时间字面量 |
| `Expr::CellPath` | `Value::CellPath` | 保留 CellPath（受限制） |
| `Expr::Filepath` / `Expr::Directory` | `Value::String` | 路径字面量退化为字符串 |
| `Expr::GlobPattern` | `Value::String` | Glob 字面量退化为字符串 |
| `Expr::List` | `Value::List` | 递归；**不支持 spread `...`** |
| `Expr::Record` | `Value::Record` | 键必须是字符串字面量；检测重复键报 `ColumnDefinedTwice`；不支持 spread |
| `Expr::Table` | `Value::List<Record>` | 内联表字面量 → 列表，每行是 record，检测重复列 |
| `Expr::Range` | `Value::Range` | 保留 Range 类型（**唯一原生支持的格式**） |
| `Expr::ValueWithUnit` | `Value::Duration` 或 `Value::Filesize` | 数字+单位 → 对应类型 |
| 其他（Block, Closure, Call, Var, BinaryOp 等） | 报错 | 不允许的语法元素 |

### 5.4 单位值策略（`Expr::ValueWithUnit`）

数字必须是 `Int`（非整数报错）。

#### Filesize 单位
| 单位 | 行为 |
|---|---|
| B / KB / MB / GB / TB 等 | `Filesize::from_unit(size, unit)`；溢出报错 |

#### Duration 单位
| 单位 | 转换（内部：纳秒） | 溢出处理 |
|---|---|---|
| `ns` | `size`（直接纳秒） | 无溢出 |
| `us` | `size * 1000` | 无溢出 |
| `ms` | `size * 1_000_000` | 无溢出 |
| `sec` / `s` | `size * 1e9` | 无溢出 |
| `min` / `m` | `size * 60e9` | 无溢出 |
| `hr` / `h` | `size * 3600e9` | 无溢出 |
| `day` / `d` | `checked_mul(86400e9)` | 溢出报错 |
| `wk` / `w` | `checked_mul(604800e9)` | 溢出报错 |

### 5.5 NUON 独有的特性

1. **原生支持 `Date`、`Duration`、`Filesize`、`Range`、`Binary`、`CellPath`**：其他格式需推断或无法表示这些类型。
2. **严格的语法子集**：不允许任何副作用代码（命令调用、变量、闭包等）。
3. **列名重复检测**：Record 和 Table 字面量在解析时就检查重复键。

---

## 六、XML（`from xml`）

### 6.1 代码位置

- 命令与转换：[`crates/nu-command/src/formats/from/xml.rs`](crates/nu-command/src/formats/from/xml.rs)
- 常量定义：[`crates/nu-command/src/formats/nu_xml_format.rs`](crates/nu-command/src/formats/nu_xml_format.rs)

### 6.2 节点类型与 Record 结构

XML 被统一转换为三列 Record 结构，所有节点都是 `{tag, attrs, content}` 形式的 Record：

| XML 节点类型 | tag 值 | attrs 值 | content 值 | 开关 |
|---|---|---|---|---|
| **元素节点** | 元素名字符串 | Record（属性名→属性值字符串） | List（子节点列表） | 默认开启 |
| **文本节点** | `Nothing` | `Nothing` | 字符串（trim 后非空） | 默认开启，空文本被过滤 |
| **注释节点** | `"!"` | `Nothing` | 注释字符串 | `--keep-comments` 开启 |
| **处理指令** | `"?<target名>"` | `Nothing` | PI 内容字符串 或 `Nothing` | `--keep-pi` 开启 |

### 6.3 关键策略

1. **属性值总是字符串**：不做数字/布尔推断，即使是 `"123"` 也是 `Value::String`。
2. **DTD 安全**：默认禁止 DTD（防 Billion Laughs 攻击），需 `--allow-dtd` 显式开启。
3. **嵌套结构递归**：元素的 `content` 是子节点组成的 `List`（而非扁平化）。
4. **根节点**：整个文档只返回根元素的 Record 表示。

---

## 七、CSV / TSV / SSV（分隔符格式）

### 7.1 代码位置

- CSV 命令：[`crates/nu-command/src/formats/from/csv.rs`](crates/nu-command/src/formats/from/csv.rs)
- TSV 命令：[`crates/nu-command/src/formats/from/tsv.rs`](crates/nu-command/src/formats/from/tsv.rs)（内部复用 delimited）
- 通用分隔符解析：[`crates/nu-command/src/formats/from/delimited.rs`](crates/nu-command/src/formats/from/delimited.rs)

### 7.2 类型推断流程（**关键**）

默认情况下每个字段值按以下**优先级顺序**尝试解析：

```
原始字符串 s
    ↓ 1. 尝试 s.parse::<i64>()
    ├─ 成功 → Value::Int
    ↓ 2. 尝试 s.parse::<f64>()
    ├─ 成功 → Value::Float
    ↓ 3. 回退
    └─ Value::String(s)
```

见 [delimited.rs#L62-L71](crates/nu-command/src/formats/from/delimited.rs#L62-L71)。

### 7.3 控制参数

| 参数 | 默认 | 效果 |
|---|---|---|
| `--no-infer` | 关闭 | **禁用推断**：所有字段均保持 `Value::String` |
| `--noheaders` | 关闭 | 不将第一行作为列名，自动生成 `column0`、`column1`…… |
| `--flexible` | 关闭 | 允许行列数不一致，多余字段用 `column2`、`column3`…… |
| `--separator` | `,`（TSV=`\t`，SSV=`;`） | 分隔符字符或 4 位 Unicode 码点 |
| `--comment` | 无 | 以指定字符开头的行被忽略 |
| `--quote` | `"` | 引号字符 |
| `--escape` | 无 | 转义字符 |
| `--trim` | `none` | `all` / `headers` / `fields`：trim 空白策略 |

### 7.4 注意事项

1. **无上下文推断**：不做跨列/跨行的一致性分析，每行独立解析。
2. **精度**：整数走 i64，浮点走 f64，与 JSON 相同。
3. **空字符串**：`""` 既不匹配 Int 也不匹配 Float，最终为 String（非 Nothing）。
4. **前导零**：`"0123"` 会被解析为 **Float(123.0)**。因为 Rust 的 `i64::from_str` **不允许前导零**，`"0123".parse::<i64>()` 返回 `Err(InvalidDigit)`，回退到 `parse::<f64>()` 成功。

---

## 八、KDL（`from kdl`）

### 8.1 代码位置

- 命令与转换：[`crates/nu-command/src/formats/from/kdl.rs`](crates/nu-command/src/formats/from/kdl.rs)
- 核心：`convert_kdl_value_to_nu_value` [L271-L287](crates/nu-command/src/formats/from/kdl.rs#L271-L287)

### 8.2 节点结构（Canonical Node Rows）

KDL 采用**固定的四列 Record** 表示每个节点（顶层为 List，因为 KDL 文档是节点序列）：

```
{
    name:     String,          // 节点名
    args:     List<Value>,     // 位置参数
    props:    Record,          // 命名属性（键=值）
    children: List<Record>     // 子节点列表（同样是四列结构）
}
```

### 8.3 值类型映射

| KdlValue | Nu Value | 规则 |
|---|---|---|
| `String(&str)` | `Value::String` | — |
| `Integer(i128)` | `Value::Int(i64)` | `to_i64()`，**溢出报错** |
| `Float(f64)` | `Value::Float(f64)` | — |
| `Bool(bool)` | `Value::Bool(bool)` | — |
| `Null` | `Value::Nothing` | — |

### 8.4 关键策略

1. **KDL Integer 是 i128**：范围远超 i64，超出 `i64::MAX/MIN` 时不降级，直接报错。
2. **重复属性**：与 KDL 规范一致，取最右侧值（`attr=1 attr=2` → attr=2）。
3. **元数据标记**：通过 PipelineMetadata 的 `nu_kdl_canonical` 键标记为规范节点行格式，供 `to kdl` 反向识别。
4. **重复兄弟节点名**：按出现顺序全部保留（不合并，不折叠）。

---

## 九、MessagePack（`from msgpack`）

### 9.1 代码位置

- 命令与解析：[`crates/nu-command/src/formats/from/msgpack.rs`](crates/nu-command/src/formats/from/msgpack.rs)
- 核心：`read_value` [L273-L385](crates/nu-command/src/formats/from/msgpack.rs#L273-L385)

### 9.2 类型映射表

| MessagePack Marker | Nu Value | 规则 |
|---|---|---|
| `FixPos(0-127)` / `FixNeg(-32~-1)` | `Value::Int` | 直接转 i64 |
| `U8/U16/U32` | `Value::Int` | 读入后转 i64 |
| `U64` | `Value::Int(i64)` | **溢出报错**（不降级为 f64） |
| `I8/I16/I32/I64` | `Value::Int` | 直接转 i64 |
| `F32` | `Value::Float(f64)` | 先 f32 → f64 扩展 |
| `F64` | `Value::Float(f64)` | 原生 |
| `Null` | `Value::Nothing` | — |
| `True/False` | `Value::Bool` | — |
| `FixStr/Str8/16/32` | `Value::String` | 读指定长度字节后 UTF-8 解码（失败报错） |
| `Bin8/16/32` | `Value::Binary` | 二进制字节数组 |
| `FixArray/Array16/32` | `Value::List` | 递归 |
| `FixMap/Map16/32` | `Value::Record` | 键必须可转 String，否则报错 |
| Ext type `-1` (Timestamp) | `Value::Date` | 三种长度：4B(秒) / 8B(打包纳秒+秒) / 12B(纳秒+秒) |
| 其他 Ext 类型 | 报错 | 仅支持时间戳扩展 |

### 9.3 关键策略

1. **递归深度限制**：`MAX_DEPTH = 50`，防止栈溢出。
2. **U64 溢出**：与 JSON 不同（降级），MessagePack 中 U64 > i64::MAX 直接报错。
3. **时间戳扩展**（type = -1）：
   - 4 字节：秒（u32）
   - 8 字节：高 30 位纳秒，低 34 位秒
   - 12 字节：先 u32 纳秒，再 i64 秒
   - 三者都转换为 `chrono::DateTime<Utc>` → 带固定偏移的 Date
4. **多对象模式**：`--objects` 持续读取直到 EOF，相当于 MessagePack 流。
5. **EOF 检查**：单对象模式会验证后续无多余字节（防止截断/损坏）。

---

## 十、类型映射总览对比表

| 源类型 / 特性 | JSON (宽松) | JSON (严格) | YAML | TOML | NUON | XML | CSV | KDL | MessagePack |
|---|---|---|---|---|---|---|---|---|---|
| **null** | Nothing | Nothing | Nothing | — | Nothing | — | — | Nothing | Nothing |
| **bool** | Bool | Bool | Bool | Bool | Bool | — | — | Bool | Bool |
| **i64 整数** | Int | Int | Int | Int | Int | — | Int* | Int* | Int |
| **u64 > i64::MAX** | Float (丢失) | Float (丢失) | Error | — | Int (仅限字面量) | — | Float* | Error | Error |
| **f32** | — | — | — | — | — | — | — | — | Float (扩展) |
| **f64 浮点** | Float | Float | Float | Float | Float | — | Float* | Float | Float |
| **字符串** | String | String | String | String | String | String | String | String | String |
| **数组/列表** | List | List | List | List | List | List (content) | List (行) | List (args/children) | List |
| **对象/Mapping/Table** | Record | Record | Record | Record | Record | Record (节点) | Record (行) | Record (props) | Record |
| **Date** | — | — | — | Date* | Date | — | — | — | Date* |
| **Duration** | — | — | — | — | Duration | — | — | — | — |
| **Filesize** | — | — | — | — | Filesize | — | — | — | — |
| **Binary** | — | — | — | — | Binary | — | — | — | Binary |
| **Range** | — | — | — | — | Range | — | — | — | — |
| **CellPath** | — | — | — | — | CellPath* | — | — | — | — |
| **注释** | 支持 | 不支持 | 支持 | 支持 | 支持 | 可选 | 可选 | 支持 | — |
| **尾随逗号** | 支持 | 不支持 | — | — | 不支持 | — | — | — | — |
| **数字溢出策略** | 降级 f64 | 降级 f64 | 失败 | 无此情况 | 失败(CheckedMul) | — | 降级 f64 | 失败 | 失败 |
| **列/键顺序** | 插入顺序（默认 preserve_order） | 插入顺序（默认 preserve_order） | IndexMap 顺序 | 插入顺序 | 插入顺序 | 文档顺序 | 行/列顺序 | 出现顺序 | 插入顺序 |
| **推断型解析** | 否 | 否 | 否 | 否 | 否 | 否 | **是(层级:Int→Float→String)** | 否 | 否 |

> * 标注：
> - `Int*`（CSV/KDL）：**仅当可解析时**才为 Int，否则 String
> - `Date*`（TOML）：仅在包含 date 部分时才成功转 Date，否则 String
> - `CellPath*`（NUON）：仅限不含 tail 的简单表达式
> - `Float*`（CSV）：仅 Int 解析失败后的 fallback

---

## 十一、关键差异与陷阱

### 11.1 大整数（≥ 2⁶³）跨格式行为

| 格式 | `9223372036854775808` (2⁶³) | `18446744073709551615` (2⁶⁴-1) |
|---|---|---|
| JSON 宽松 / 严格 | `Float(9.223e18)`（**丢失精度**） | `Float(1.844e19)`（**丢失精度**） |
| YAML (serde_yaml Number) | `Error`（无法匹配 is_i64/is_f64） | `Error` |
| TOML | `Error`（TOML 整数是 i64，超范围） | `Error` |
| NUON 字面量 | `Error`（字面量解析溢出） | `Error` |
| KDL (i128) | `Error`（`to_i64()` 失败） | `Error` |
| MessagePack U64 | `Error`（`try_into` i64 失败） | `Error` |
| CSV `--no-infer` | `String("9223372036854775808")` | `String("18446744073709551615")` |
| CSV 默认 | `Float(9.223e18)`（**丢失精度**） | `Float(1.844e19)`（**丢失精度**） |

### 11.2 相同数字字符串在不同格式中的结果

以 `"123"` 为例：

| 格式 | 结果 |
|---|---|
| JSON 字符串字面量 `"\"123\""` | String("123") |
| JSON 数字字面量 `123` | Int(123) |
| YAML 无引号 `123` | Int(123) |
| YAML 带引号 `"123"` | String("123") |
| CSV 字段 `123`（默认） | Int(123) |
| CSV 字段 `123`（`--no-infer`） | String("123") |
| XML 属性值 `val="123"` | **String("123")**（从不推断） |
| TOML 无引号 `a = 123` | Int(123) |
| TOML 引号 `a = "123"` | String("123") |

### 11.3 浮点数精度

- 所有格式的浮点数最终都收敛到 **`f64`（IEEE 754 双精度）**，约 15-17 位有效十进制数字。
- MessagePack `F32` 先以 f32 读取再扩展为 f64，**仅保留约 7 位有效数字**，这是信息丢失的来源。
- 格式本身不支持 NaN、Infinity 等特殊浮点（JSON 严格模式尤其严格）。

### 11.4 Record 键顺序

| 格式 | 键顺序保证 |
|---|---|
| NUON / TOML / YAML / KDL / XML | 源码/文档中出现的顺序 |
| JSON（默认，preserve_order 启用） | **插入顺序**（默认 feature，使用 LinkedHashMap） |
| JSON（关闭 preserve_order） | 字典序（使用 BTreeMap，非默认） |
| CSV (列名) | 第一行中出现的顺序 |
| MessagePack Map | 序列化时的顺序 |

### 11.5 特殊值的表示

| 语义 | NUON | JSON | YAML | TOML |
|---|---|---|---|---|
| 1 秒时间长度 | `1sec` → Duration(1e9) | 无（只能用整数/字符串约定） | 无 | 无 |
| 1 KB 文件大小 | `1kb` → Filesize(1024) | 无 | 无 | 无 |
| 1..10 范围 | `1..10` → Range | 无 | 无 | 无 |
| 2024-01-01 日期 | `2024-01-01` → Date | 无（只能字符串） | 无（可能解析为字符串） | `2024-01-01` → Date |

---

## 十二、代码文件索引

| 命令/模块 | 关键文件 | 核心函数/结构 |
|---|---|---|
| Value 定义 | [mod.rs](crates/nu-protocol/src/value/mod.rs#L77-L219) | `enum Value` |
| from json | [json.rs](crates/nu-command/src/formats/from/json.rs) | `try_str_to_value` |
| JSON 中间值 | [value.rs](crates/nu-json/src/value.rs#L50-L74) | `enum Value`（I64/U64/F64...） |
| JSON→Nu 转换 | [nu_value.rs](crates/nu-json/src/nu_value.rs#L9-L33) | `impl IntoValue for JsonValue` |
| from yaml | [yaml.rs](crates/nu-command/src/formats/from/yaml.rs#L42-L153) | `convert_yaml_value_to_nu_value` |
| from toml | [toml.rs](crates/nu-command/src/formats/from/toml.rs#L60-L130) | `convert_toml_to_value`, `convert_toml_datetime_to_value` |
| from nuon 核心 | [from.rs](crates/nuon/src/from.rs#L129-L479) | `convert_to_value`（AST 分派） |
| from xml | [xml.rs](crates/nu-command/src/formats/from/xml.rs#L195-L207) | `from_node_to_value` 分派 |
| 分隔符(CSV等) | [delimited.rs](crates/nu-command/src/formats/from/delimited.rs#L62-L71) | `from_delimited_stream` 内 values 映射 |
| from kdl | [kdl.rs](crates/nu-command/src/formats/from/kdl.rs#L226-L287) | `convert_kdl_document_to_node_rows`, `convert_kdl_value_to_nu_value` |
| from msgpack | [msgpack.rs](crates/nu-command/src/formats/from/msgpack.rs#L273-L481) | `read_value` (marker 分派), `read_ext` |
