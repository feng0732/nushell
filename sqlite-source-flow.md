# SQLite 数据源执行链路分析

本文档详细解析 Nushell 中 SQLite 数据源从连接建立到结果映射的完整执行链路，涵盖查询执行、参数绑定和大结果集处理。

## 目录

- [整体架构](#整体架构)
- [三种连接方式详解](#三种连接方式详解)
- [查询执行流程](#查询执行流程)
- [参数绑定机制与限制](#参数绑定机制与限制)
- [结果映射：SQLite → Nu Value](#结果映射sqlite--nu-value)
- [大结果集处理与限制](#大结果集处理与限制)
- [懒查询构建器与 SQL 下推优化](#懒查询构建器与-sql-下推优化)
- [数据流全景图](#数据流全景图)

---

## 整体架构

SQLite 功能主要位于 `crates/nu-command/src/database/` 目录下，基于 `rusqlite` crate 实现。核心组件包括：

| 组件 | 文件 | 职责 |
|------|------|------|
| `SQLiteDatabase` | `database/values/sqlite.rs` | 数据库自定义值，封装路径与信号 |
| `SQLiteQueryBuilder` | `database/values/sqlite.rs` | 懒查询构建器，支持 SQL 下推优化 |
| `query db` 命令 | `database/commands/query_db.rs` | 执行任意 SQL 查询 |
| `into sqlite` 命令 | `database/commands/into_sqlite.rs` | 将 Nu 数据写入 SQLite |
| `schema` 命令 | `database/commands/schema.rs` | 查看数据库结构 |

### 入口点

1. **文件打开**：`open foo.db` — 通过 `SQLiteDatabase::try_from_path` 检测魔术字节识别 SQLite 文件
2. **内存数据库**：`stor open` — 直接创建基于 `MEMORY_DB` 的内存数据库
3. **历史命令**：`history` — SQLite 格式历史记录返回 `SQLiteQueryBuilder` 懒查询对象

---

## 三种连接方式详解

代码中存在三种创建 SQLite 连接的路径，它们在 **rusqlite 调用方式**、**busy handler 设置**、**错误报告风格** 和 **使用场景** 上都有明显差异。理解这些差异是读懂连接重试逻辑的关键。

### 方式 1：`open_sqlite_db` — 通用文件查询连接

**定义位置**：`database/values/sqlite.rs` `open_sqlite_db` 函数

**调用者**：
- `SQLiteDatabase::query()` — `query db` 命令的核心
- `SQLiteQueryBuilder::execute()` — 懒查询最终执行
- `SQLiteQueryBuilder::count()` — 懒查询计数
- `into_sqlite::Table::new()` — 写入数据库

**逻辑流程**：

```
open_sqlite_db(path, call_span)
  │
  ├─ path == MEMORY_DB ?
  │     YES → open_connection_in_memory_custom()
  │             (共享内存连接 + busy_handler)
  │
  │     NO  → Connection::open(path)
  │             (普通文件连接, 无 busy_handler!)
  │
  └─ 错误: ShellError::Generic (附带用户可见的 call_span)
```

**关键特征**：
- 文件数据库**不设置 busy_handler**，遇到 SQLITE_BUSY 直接返回错误
- 内存数据库会委托给 `open_connection_in_memory_custom()`，自动获得 busy_handler
- 错误使用 `GenericError::new` 附带用户调用位置 span，便于报错定位

### 方式 2：`open_connection_in_memory_custom` — 共享内存连接

**定义位置**：`database/values/sqlite.rs` `open_connection_in_memory_custom` 函数

**调用者**：
- `open_sqlite_db` 当路径为 MEMORY_DB 时
- `SQLiteDatabase::open_connection` 当路径为 MEMORY_DB 时

**逻辑流程**：

```
open_connection_in_memory_custom()
  │
  ├─ Connection::open_with_flags(MEMORY_DB, OpenFlags::default())
  │     URI: "file:memdb1?mode=memory&cache=shared"
  │
  ├─ conn.busy_handler(Some(SQLiteDatabase::sleeper))
  │     设置重试处理器
  │
  └─ 错误: ShellError::Generic (使用 Span::test_data(), 非用户调用位置)
```

**关键特征**：
- 使用 `open_with_flags` + 共享缓存 URI，不是 `Connection::open_in_memory()`
- **始终设置 busy_handler**，因为共享内存数据库可能被多个连接并发访问
- 错误 span 使用 `Span::test_data()`，是一个固定测试 span，不反映用户调用位置

### 方式 3：`SQLiteDatabase::open_connection` — 实例级连接

**定义位置**：`database/values/sqlite.rs` `SQLiteDatabase::open_connection` 方法

**调用者**：
- `schema` 命令 — 查看数据库结构
- `stor` 操作 — 内存数据库的 CRUD
- 备份/恢复操作

**逻辑流程**：

```
self.open_connection()
  │
  ├─ path == MEMORY_DB ?
  │     YES → open_connection_in_memory_custom()
  │             (委托给方式 2)
  │
  │     NO  → Connection::open(&self.path)
  │           + conn.busy_handler(Some(SQLiteDatabase::sleeper))
  │           (文件连接 + busy_handler!)
  │
  └─ 错误: GenericError::new_internal (内部错误, 不含用户 span)
```

**关键特征**：
- 文件数据库**也设置 busy_handler**（与方式 1 的关键区别！）
- 错误使用 `GenericError::new_internal`，标记为内部错误而非用户错误
- 适用于需要持连接做多次操作的场景（schema 查询需要多次 prepare）

### 三种方式对比总表

| 特性 | `open_sqlite_db` | `open_connection_in_memory_custom` | `open_connection` |
|------|------------------|-----------------------------------|-------------------|
| **文件连接** | `Connection::open` | — | `Connection::open` |
| **内存连接** | 委托 → 方式2 | `open_with_flags` + 共享URI | 委托 → 方式2 |
| **busy_handler** | ❌ 文件连接不设置 | ✅ 始终设置 | ✅ 始终设置 |
| **错误风格** | 用户可见 span | `Span::test_data()` | `new_internal` |
| **典型场景** | 一次性查询 | 共享内存数据库 | 持连接多操作 |
| **SQLITE_BUSY 行为** | 直接报错 | 250ms 重试，无限 | 250ms 重试，无限 |

### Busy Handler 详解

`SQLiteDatabase::sleeper` 是连接重试的核心，仅在被设置了 busy_handler 的连接上生效：

```rust
fn sleeper(attempts: i32) -> bool {
    log::warn!("SQLITE_BUSY, retrying after 250ms (attempt {attempts})");
    std::thread::sleep(std::time::Duration::from_millis(250));
    true  // true = 继续重试; false = 停止重试并返回 SQLITE_BUSY 错误
}
```

**工作原理**：
- 当 SQLite 返回 `SQLITE_BUSY` 时，rusqlite 不会立即抛错，而是调用注册的 busy_handler
- `sleeper` 每次等待 250ms 后返回 `true`，表示"请继续重试"
- 由于始终返回 `true`，这构成**无限重试**，直到获取锁为止
- `attempts` 参数由 rusqlite 递增传入，目前代码中仅用于日志，未做上限判断

**哪些场景会触发**：
- 方式 1（`open_sqlite_db`）打开文件数据库 → **不设置** → 遇 BUSY 直接报错
- 方式 2（内存连接）→ **设置** → 遇 BUSY 自动重试
- 方式 3（`open_connection`）→ **设置** → 遇 BUSY 自动重试

**潜在问题**：方式 1 中 `query db` 对文件数据库不设 busy_handler。如果文件被其他进程锁定，查询会立即失败而非等待重试。而 `schema` 命令（使用方式 3）则能自动重试。

### 第四种变体：`open_connection_in_memory` — 私有内存连接

```rust
pub fn open_connection_in_memory() -> Result<Connection, ShellError> {
    Connection::open_in_memory().map_err(...)
}
```

- 使用 `Connection::open_in_memory()`，创建**非共享**的私有内存数据库
- **不设置 busy_handler**
- **仅在单元测试中使用**
- 与 `open_connection_in_memory_custom` 的区别：后者用共享 URI 可跨连接访问同一内存库

### 连接生命周期

**重要设计决策**：`SQLiteDatabase` **不存储连接对象**，只存储路径。每次查询都新建连接。

源码注释（`SQLiteDatabase` 结构体定义处）说明了原因：
1. YAGNI 原则
2. 连接克隆语义不明确
3. 状态管理复杂

---

## 查询执行流程

### 完整调用链

以 `open foo.db | query db "SELECT * FROM Bar"` 为例：

```
open 命令
  │
  ├─ SQLiteDatabase::try_from_path(path, span, signals)
  │    ├─ 读取文件前 16 字节
  │    └─ 比较 SQLITE_MAGIC_BYTES ("SQLite format 3\0")
  │
  └─ 返回 SQLiteDatabase CustomValue
        │
        ▼
query db 命令
  │
  ├─ SQLiteDatabase::try_from_pipeline(input, call.head)
  │    └─ 从 PipelineData 中提取 SQLiteDatabase
  │
  ├─ nu_value_to_params(engine_state, params_value, call.head)
  │    └─ 将 Nu 值转换为 SQL 参数
  │
  └─ SQLiteDatabase::query(sql, params, call_span)
       │
       ├─ open_sqlite_db(&self.path, call_span)
       │    └─ 创建 rusqlite::Connection (方式1, 无 busy_handler)
       │
       └─ run_sql_query(conn, sql, params, signals, None)
            │
            ├─ conn.prepare(&sql.item)        → 创建 Statement
            │
            └─ prepared_statement_to_nu_list(stmt, params, ...)
                 │
                 ├─ stmt.columns()             → 获取列名和类型
                 │
                 ├─ stmt.query_map(params, |row| { ... })
                 │    └─ 逐行转换为 Nu Value
                 │
                 └─ Value::list(row_values, call_span)
```

### 核心查询函数

#### `run_sql_query` — 执行 SQL 查询

**定义位置**：`database/values/sqlite.rs` `run_sql_query` 函数

```rust
fn run_sql_query(
    conn: Connection,
    sql: &Spanned<String>,
    params: NuSqlParams,
    signals: &Signals,
    column_adapters: Option<&BTreeMap<String, SQLiteColumnAdapter>>,
) -> Result<Value, SqliteOrShellError>
```

**参数**：
- `conn`: 已打开的数据库连接（所有权转移，查询后连接被消耗）
- `sql`: 带 span 信息的 SQL 语句
- `params`: 查询参数（位置或命名）
- `signals`: 取消信号
- `column_adapters`: 列适配器（如时间戳转换）

#### `prepared_statement_to_nu_list` — 语句结果转 Nu 列表

**定义位置**：`database/values/sqlite.rs` `prepared_statement_to_nu_list` 函数

这是结果处理的核心函数，步骤如下：

1. **列信息提取**：通过 `stmt.columns()` 获取所有列的名称和声明类型
2. **参数绑定与执行**：根据参数类型选择不同的 `query_map` 调用
3. **行收集**：遍历结果集，逐行转换
4. **信号检查**：每行检查一次取消信号

---

## 参数绑定机制与限制

### 参数类型枚举

**定义位置**：`database/values/sqlite.rs` `NuSqlParams` 枚举

```rust
pub enum NuSqlParams {
    List(Vec<Box<dyn ToSql>>),              // 位置参数
    Named(Vec<(String, Box<dyn ToSql>)>),   // 命名参数
}
```

### Nu 值 → SQL 参数转换

#### `nu_value_to_params` — 入口函数

**定义位置**：`database/values/sqlite.rs` `nu_value_to_params` 函数

根据输入值类型分派：

| 输入类型 | 参数类型 | 说明 |
|----------|----------|------|
| `Value::Record` | `Named` | 记录字段名作为参数名，自动加 `:` 前缀 |
| `Value::List` | `List` | 列表元素按顺序作为位置参数 |
| `Value::Nothing` | `List` (空) | 无参数 |
| 其他 | 错误 | 类型不匹配 |

**命名参数自动补前缀**：如果参数名不以 `:`、`@` 或 `$` 开头，自动插入 `:`。

#### `value_to_sql` — 单值转换

**定义位置**：`database/values/sqlite.rs` `value_to_sql` 函数

Nu 值类型到 SQLite 参数的映射：

| Nu 类型 | SQLite 类型 | 说明 |
|---------|------------|------|
| `Bool` | `bool` | 直接映射 |
| `Int` | `i64` | 直接映射 |
| `Float` | `f64` | 直接映射 |
| `Filesize` | `i64` | 取字节数 |
| `Duration` | `i64` | 纳秒数 |
| `Date` | `DateTime` | chrono 类型 |
| `String` | `String` | 直接映射 |
| `Binary` | `Vec<u8>` | 直接映射 |
| `Nothing` | `Null` | NULL 值 |
| 其他（List/Record 等） | JSON 字符串 | 序列化为 JSON 文本 |

### 参数绑定执行

**定义位置**：`database/values/sqlite.rs` `prepared_statement_to_nu_list` 函数

**位置参数**：
```rust
let refs: Vec<&dyn ToSql> = params.iter().map(|value| &**value).collect();
stmt.query_map(refs.as_slice(), |row| { ... })
```

**命名参数**：
```rust
let refs: Vec<_> = pairs.iter()
    .map(|(column, value)| (column.as_str(), &**value))
    .collect();
stmt.query_map(refs.as_slice(), |row| { ... })
```

**设计要点**：
- 两种参数风格需要分开调用 `query_map`，因为 rusqlite 对位置和命名参数使用不同的引用类型
- 行处理逻辑通过 `collect_row_values` 函数共享

### 参数绑定限制

#### 1. 懒查询构建器不传递参数

`SQLiteQueryBuilder::execute()` 中存在已知问题：

```rust
let params = NuSqlParams::List(Vec::new()); // FIXME: handle params properly
```

虽然 `SQLiteQueryBuilder` 有 `sql_params` 字段和 `with_where` 方法可以设置 WHERE 参数，但 `execute()` 始终传入空参数列表。这意味着：

- 通过懒查询路径执行的查询**无法正确绑定 WHERE 参数**
- `count()` 方法使用 `sql_params` 但将所有参数转为 `String` 类型（丢失原始类型信息）
- 懒查询的 WHERE 子句如果含占位符 `?`，运行时将因参数数量不匹配而报错

#### 2. count() 参数类型丢失

```rust
let params: Vec<Box<dyn ToSql>> = self
    .sql_params
    .iter()
    .map(|s| Box::new(s.clone()) as Box<dyn ToSql>)
    .collect();
```

`sql_params` 是 `Vec<String>`，所有参数都被当作文本字符串传入，整数/浮点数等类型信息丢失。SQLite 的类型亲和性机制通常能缓解此问题，但在严格类型比较场景下可能产生意外结果（如 `WHERE id = ?` 传入字符串 `"1"` 与整数 `1` 的比较行为不同）。

#### 3. 命名参数前缀策略单一

`nu_value_to_params` 自动补 `:` 前缀，但 SQLite 原生支持三种命名参数前缀：
- `:name` — 冒号前缀（Nu 默认使用）
- `@name` — at 前缀
- `$name` — 美元前缀

Nu 只在不以这三种字符开头时自动补 `:`，但如果用户 SQL 中使用 `@` 或 `$` 前缀参数，需要手动在 Record key 中写明前缀，否则参数名不匹配。

#### 4. 复杂类型的 JSON 序列化回读依赖列声明

`value_to_sql` 将 List/Record 序列化为 JSON 字符串存储。但读回时，`convert_sqlite_value_to_nu_value` 只有在列声明类型为 `JSON` 或 `JSONB` 时才会自动反序列化。如果列声明为 `TEXT`，读回的将是原始 JSON 字符串而非 Nu 结构化值。

#### 5. --params 的编译期类型检查缺失

`query db` 命令的 `--params` 参数使用 `SyntaxShape::Any`，在解析阶段不检查参数是否为 Record 或 List，错误延迟到运行时才报告。

---

## 结果映射：SQLite → Nu Value

### 行转换：`convert_sqlite_row_to_nu_value`

**定义位置**：`database/values/sqlite.rs` `convert_sqlite_row_to_nu_value` 函数

将 SQLite 行转换为 Nu Record：

```rust
pub fn convert_sqlite_row_to_nu_value(
    row: &Row,
    span: Span,
    columns: &[TypedColumn],
    column_adapters: Option<&BTreeMap<String, SQLiteColumnAdapter>>,
) -> Value
```

**流程**：
1. 遍历每一列（索引 + 列信息）
2. 查找该列是否有适配器
3. 调用 `convert_sqlite_value_to_nu_value_with_adapter` 转换值
4. 收集为 Record

### 列类型感知：`TypedColumn`

**定义位置**：`database/values/sqlite.rs` `TypedColumn` 结构体

```rust
pub struct TypedColumn {
    pub name: String,
    pub decl_type: Option<DeclType>,
}
```

`DeclType` 目前只特殊处理 `JSON` 和 `JSONB` 类型列。

### 值转换：`convert_sqlite_value_to_nu_value`

**定义位置**：`database/values/sqlite.rs` `convert_sqlite_value_to_nu_value` 函数

SQLite 值类型到 Nu 值的映射：

| SQLite ValueRef | Nu 类型 | 说明 |
|-----------------|---------|------|
| `Null` | `Nothing` | NULL 值 |
| `Integer(i)` | `Int` | 64 位整数 |
| `Real(f)` | `Float` | 64 位浮点数 |
| `Text(buf)` | `String` 或 解析后的值 | UTF-8 文本；JSON/JSONB 列自动解析 |
| `Blob(u)` | `Binary` | 二进制数据 |

#### JSON 列特殊处理

如果列声明类型是 `JSON` 或 `JSONB`，文本值会自动解析为 Nu 结构化值：

```rust
ValueRef::Text(buf) => match (std::str::from_utf8(buf), decl_type) {
    (Ok(txt), Some(DeclType::Json | DeclType::Jsonb)) => {
        match crate::try_json_str_to_value(txt, span, false) {
            Ok(val) => val,
            Err(err) => Value::error(err, span),
        }
    }
    // ...
}
```

### 列适配器：`SQLiteColumnAdapter`

**定义位置**：`database/values/sqlite.rs` `SQLiteColumnAdapter` 枚举

适配器用于对特定列进行语义转换：

| 适配器 | 作用 |
|--------|------|
| `UnixMillisToDate` | 整数列（Unix 时间戳毫秒）→ `Value::Date` |
| `MillisToDuration` | 整数列（毫秒数）→ `Value::Duration` |

转换逻辑在 `convert_sqlite_value_to_nu_value_with_adapter` 中实现。

**注意**：只对 `Integer` 类型应用适配器，其他类型回退到普通转换。若适配器返回值转换失败（如时间戳超出 `chrono::DateTime::from_timestamp_millis` 范围），回退为原始 `Int` 值而非报错。

---

## 大结果集处理与限制

### 当前实现：全量加载

**现状**：当前实现使用 `collect_row_values` 将所有行一次性收集到 `Vec<Value>` 中。

**定义位置**：`database/values/sqlite.rs` `collect_row_values` 函数

```rust
fn collect_row_values(
    row_results: impl IntoIterator<Item = Result<Value, SqliteError>>,
    signals: &Signals,
    call_span: Span,
) -> Result<Vec<Value>, SqliteOrShellError> {
    let mut row_values = vec![];
    for row_result in row_results {
        signals.check(&call_span)?;
        if let Ok(row_value) = row_result {
            row_values.push(row_value);
        }
    }
    Ok(row_values)
}
```

### 取消信号检查

每行处理前检查一次信号，支持用户中断大查询：
```rust
signals.check(&call_span)?;
```

### 现有优化手段

虽然没有流式输出，但有以下优化方式：

1. **SQL 下推 LIMIT**：通过 `SQLiteQueryBuilder` 将 `first`/`last` 等操作下推为 SQL 的 `LIMIT`
2. **列选择下推**：`select` 操作下推为 SQL 的 `SELECT` 列选择
3. **`count()` 优化**：直接执行 `SELECT COUNT(*)` 而不是加载全部数据

### 历史命令的懒加载模式

`history` 命令返回 `SQLiteQueryBuilder` 而不是立即执行查询，这样用户可以通过管道操作（如 `first`、`where` 等）触发 SQL 下推优化，避免全量加载。

### 大结果集限制

#### 1. 全量物化，无流式输出

所有查询结果都被完整加载到内存中的 `Vec<Value>`。对于大表（百万行级），这意味着：
- 内存占用与结果集大小成正比
- 用户必须等待所有行处理完毕才能看到任何输出
- 没有分页或游标机制

相比之下，Nu 的 `ListStream` 可以逐项产生输出，但 SQLite 查询路径未使用此能力。

#### 2. 错误行被静默跳过

```rust
if let Ok(row_value) = row_result {
    row_values.push(row_value);
}
```

如果某行转换失败（`row_result` 为 `Err`），该行会被静默丢弃，不报错也不记录。在大结果集中，这可能导致数据悄然丢失而不易察觉。

#### 3. `read_entire_sqlite_db` 读取全部表全部行

`open foo.db` 在不指定表名时会调用 `read_entire_sqlite_db`，遍历 `sqlite_master` 中的所有表并对每张表执行 `SELECT *`。对于包含多张大表的数据库，这可能消耗大量内存和时间。

#### 4. 无行数限制

`query db` 命令本身没有 `--limit` 参数，用户必须自行在 SQL 中加 `LIMIT` 子句来控制结果大小。没有安全阀防止用户执行 `SELECT * FROM huge_table` 导致内存溢出。

#### 5. 连接不复用

由于 `SQLiteDatabase` 不持有连接，每次操作（包括 `to_base_value`、`follow_path_int` 等）都重新打开连接。对于频繁访问同一数据库的场景，连接创建开销会累积。

---

## 懒查询构建器与 SQL 下推优化

### `SQLiteQueryBuilder` 结构

**定义位置**：`database/values/sqlite.rs` `SQLiteQueryBuilder` 结构体

```rust
pub struct SQLiteQueryBuilder {
    pub db_path: PathBuf,
    pub table_name: String,
    pub sql_select: Option<String>,      // e.g. "column1, column2" or "*"
    pub sql_where: Option<String>,       // e.g. "column = ?"
    pub sql_params: Vec<String>,         // parameters for the where clause
    pub sql_order_by: Option<String>,    // e.g. "id DESC"
    pub sql_limit: Option<i64>,
    pub column_adapters: BTreeMap<String, SQLiteColumnAdapter>,
    signals: Signals,
}
```

### 核心特性

#### 1. 链式构建 API

```rust
table
    .with_select("id, name".to_string())
    .with_where("id > ?".to_string(), vec!["10".to_string()])
    .with_order_by("id DESC".to_string())
    .with_limit(5)
    .with_unix_millis_datetime_column("created_at".to_string())
```

#### 2. SQL 生成

**定义位置**：`database/values/sqlite.rs` `SQLiteQueryBuilder::build_sql` 方法

```rust
pub fn build_sql(&self) -> String {
    let select = self.sql_select.as_deref().unwrap_or("*");
    let mut sql = format!("SELECT {} FROM [{}]", select, self.table_name);
    // 追加 WHERE / ORDER BY / LIMIT
}
```

#### 3. 列投影下推

**定义位置**：`database/values/sqlite.rs` `SQLiteQueryBuilder::project_output_columns` 方法

`project_output_columns` 方法支持在已有 SELECT 投影上进一步选择列，保持别名不变。

**示例**：
- 当前投影：`command_line as command, duration_ms as duration`
- 请求输出：`command`
- 重写后：`command_line as command`

解析器实现了轻量级 SQL 投影解析：
- `parse_sql_select_projection` — 解析投影列表
- `split_select_expressions` — 按顶层逗号分割（处理引号和括号）
- `parse_projection_expression` — 解析单个表达式（别名、限定名等）
- `split_alias` — 识别 `AS` 关键字

#### 4. 执行与计数

- `execute(span)` — 构建 SQL 并执行，返回 `PipelineData`
- `count(span)` — 执行 `SELECT COUNT(*)` 优化查询

### 作为 CustomValue 的行为

`SQLiteQueryBuilder` 实现了 `CustomValue` trait，使其可以像普通值一样在管道中传递：

| 方法 | 行为 |
|------|------|
| `to_base_value` | 执行查询并返回完整结果 |
| `follow_path_int` | 执行后按索引访问（未优化，无 LIMIT 下推） |
| `follow_path_string` | 执行后按列名访问（未优化） |
| `is_iterable` | 返回 `true`，支持迭代 |

---

## 数据流全景图

### 场景 1：打开文件并查询

```
用户: open foo.db | query db "SELECT * FROM users"
      │
      ▼
┌─────────────────────────────────────────────────────┐
│ open 命令                                           │
│   ├─ 检测文件头 (SQLITE_MAGIC_BYTES)                │
│   └─ 创建 SQLiteDatabase { path, signals }          │
│      封装为 Value::Custom 输出                      │
└──────────────────────┬──────────────────────────────┘
                       │ PipelineData (CustomValue)
                       ▼
┌─────────────────────────────────────────────────────┐
│ query db 命令                                       │
│   ├─ 从输入提取 SQLiteDatabase                      │
│   ├─ 解析 --params 参数 (list/record → NuSqlParams) │
│   └─ 调用 db.query(sql, params, span)               │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│ SQLiteDatabase::query                               │
│   ├─ open_sqlite_db(path) → Connection              │
│   │   ⚠ 文件连接: 无 busy_handler                   │
│   └─ run_sql_query(conn, sql, params, signals)     │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│ run_sql_query                                       │
│   ├─ conn.prepare(sql) → Statement                  │
│   └─ prepared_statement_to_nu_list(stmt, params)    │
│      ├─ 获取列信息 (TypedColumn)                    │
│      ├─ query_map 绑定参数并执行                    │
│      ├─ ⚠ 全量加载到 Vec<Value>                     │
│      └─ 逐行 convert_sqlite_row_to_nu_value        │
└──────────────────────┬──────────────────────────────┘
                       │ Value::List
                       ▼
                  输出结果
```

### 场景 2：查看数据库结构

```
用户: open foo.db | schema
      │
      ▼
┌─────────────────────────────────────────────────────┐
│ open 命令 → SQLiteDatabase CustomValue              │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│ schema 命令                                         │
│   ├─ SQLiteDatabase::try_from_pipeline(input, span)  │
│   ├─ db.open_connection()                           │
│   │   ✅ 文件连接: 有 busy_handler (方式3)           │
│   ├─ db.get_tables(&conn)                           │
│   ├─ db.get_columns(&conn, &table)                  │
│   ├─ db.get_constraints(&conn, &table)              │
│   ├─ db.get_foreign_keys(&conn, &table)             │
│   └─ db.get_indexes(&conn, &table)                  │
│      (同一连接执行多次查询)                          │
└──────────────────────┬──────────────────────────────┘
                       │ Value::Record (schema 信息)
                       ▼
                  输出结果
```

### 场景 3：写入数据

```
用户: [[a b]; [1 2] [3 4]] | into sqlite test.db -t my_table
      │
      ▼
┌─────────────────────────────────────────────────────┐
│ into sqlite 命令                                    │
│   ├─ open_sqlite_db(path) → Connection (方式1)      │
│   │   ⚠ 文件连接: 无 busy_handler                   │
│   ├─ 首行推断表结构 (nu_value_to_sqlite_type)       │
│   ├─ CREATE TABLE IF NOT EXISTS                     │
│   └─ 开启事务                                       │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│ 批量插入                                            │
│   ├─ prepare INSERT 语句                            │
│   ├─ values_to_sql 转换每一行                       │
│   ├─ execute 执行                                   │
│   └─ 逐行检查信号                                   │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
                  提交事务
```

### 场景 4：懒查询与下推优化

```
用户: history | where command =~ "cargo" | first 5
      │
      ▼
┌─────────────────────────────────────────────────────┐
│ history 命令 (SQLite 模式)                          │
│   └─ 返回 SQLiteQueryBuilder (懒查询)               │
│      - table: "history"                             │
│      - select: "command_line as command, ..."       │
│      - order_by: "rowid ASC"                        │
│      - 列适配器: timestamp→date, duration→duration  │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│ where / first 等命令                                │
│   └─ 通过 CustomValue 接口操作                       │
│      (理想情况下触发 SQL 下推, 目前部分实现)        │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
              最终执行查询并返回结果
```

---

## 关键设计决策总结

### 1. 无状态设计
`SQLiteDatabase` 只存路径不存连接，简单但每次查询都有连接开销。

### 2. CustomValue 抽象
通过 `CustomValue` trait，数据库和查询构建器可以无缝融入 Nu 的类型系统和管道。

### 3. 双参数模式
同时支持位置参数和命名参数，通过 `NuSqlParams` 枚举统一处理。

### 4. JSON 列自动解析
声明为 JSON/JSONB 的列自动反序列化为 Nu 结构化值，提供无缝体验。

### 5. 列适配器模式
通过 `SQLiteColumnAdapter` 扩展列类型转换能力，如时间戳/持续时间转换。

### 6. SQL 下推优化
`SQLiteQueryBuilder` 支持将部分 Nu 操作下推为 SQL，减少数据传输。

### 7. 信号检查
每行检查一次取消信号，保证用户可以中断大查询。

### 8. 连接策略差异
文件查询（`open_sqlite_db`）不设 busy_handler，遇锁直接报错；实例连接（`open_connection`）设 busy_handler 自动重试。这导致 `query db` 和 `schema` 对同一文件在并发场景下行为不同。

---

## 相关文件索引

| 文件路径 | 主要内容 |
|---------|---------|
| `database/values/sqlite.rs` | 核心实现：连接、查询、转换、构建器 |
| `database/commands/query_db.rs` | query db 命令 |
| `database/commands/into_sqlite.rs` | into sqlite 命令 |
| `database/commands/schema.rs` | schema 命令 |
| `filesystem/open.rs` | open 命令（SQLite 检测入口） |
| `stor/open.rs` | stor open 命令（内存数据库入口） |
| `commands/history/history_.rs` | history 命令（懒查询示例） |
