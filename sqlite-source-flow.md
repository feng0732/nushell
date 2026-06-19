# SQLite 数据源执行链路分析

本文档详细解析 Nushell 中 SQLite 数据源从连接建立到结果映射的完整执行链路，涵盖查询执行、参数绑定和大结果集处理。

## 目录

- [整体架构](#整体架构)
- [连接建立与管理](#连接建立与管理)
- [查询执行流程](#查询执行流程)
- [参数绑定机制](#参数绑定机制)
- [结果映射：SQLite → Nu Value](#结果映射sqlite--nu-value)
- [大结果集处理](#大结果集处理)
- [懒查询构建器与 SQL 下推优化](#懒查询构建器与-sql-下推优化)
- [数据流全景图](#数据流全景图)

---

## 整体架构

SQLite 功能主要位于 `crates/nu-command/src/database/` 目录下，基于 `rusqlite` crate 实现。核心组件包括：

| 组件 | 文件 | 职责 |
|------|------|------|
| `SQLiteDatabase` | [sqlite.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/values/sqlite.rs#L27-L36) | 数据库自定义值，封装路径与信号 |
| `SQLiteQueryBuilder` | [sqlite.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/values/sqlite.rs#L809-L821) | 懒查询构建器，支持 SQL 下推优化 |
| `query db` 命令 | [query_db.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/commands/query_db.rs) | 执行任意 SQL 查询 |
| `into sqlite` 命令 | [into_sqlite.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/commands/into_sqlite.rs) | 将 Nu 数据写入 SQLite |
| `schema` 命令 | [schema.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/commands/schema.rs) | 查看数据库结构 |

### 入口点

1. **文件打开**：`open foo.db` — 通过 `SQLiteDatabase::try_from_path` 检测魔术字节识别 SQLite 文件
2. **内存数据库**：`stor open` — 直接创建基于 `MEMORY_DB` 的内存数据库
3. **历史命令**：`history` — SQLite 格式历史记录返回 `SQLiteQueryBuilder` 懒查询对象

---

## 连接建立与管理

### 连接创建函数

连接通过多个层次的函数创建，形成清晰的调用链：

```
open_sqlite_db(path, span)
    └─→ Connection::open(path) 或 open_connection_in_memory_custom()
        └─→ 设置 busy_handler (sleeper)
```

### 关键函数

#### `open_sqlite_db` — 通用入口

[sqlite.rs:410-423](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/values/sqlite.rs#L410-L423)

```rust
pub fn open_sqlite_db(path: &Path, call_span: Span) -> Result<Connection, ShellError>
```

**逻辑**：
- 内存数据库（`MEMORY_DB`）→ 调用 `open_connection_in_memory_custom()`
- 文件数据库 → `Connection::open(path)`
- 错误转换为 `ShellError::Generic` 附带调用 span

#### `open_connection_in_memory_custom` — 带标志的内存连接

[sqlite.rs:776-794](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/values/sqlite.rs#L776-L794)

- 使用 `OpenFlags::default()` 打开共享内存数据库
- URI: `file:memdb1?mode=memory&cache=shared`
- 设置 `busy_handler` 处理并发

#### `SQLiteDatabase::open_connection` — 实例方法

[sqlite.rs:112-131](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/values/sqlite.rs#L112-L131)

实例级连接方法，与 `open_sqlite_db` 类似但属于实例方法。

### 并发处理：Busy Handler

[sqlite.rs:133-137](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/values/sqlite.rs#L133-L137)

```rust
fn sleeper(attempts: i32) -> bool {
    log::warn!("SQLITE_BUSY, retrying after 250ms (attempt {attempts})");
    std::thread::sleep(std::time::Duration::from_millis(250));
    true
}
```

**设计特点**：
- 遇到 `SQLITE_BUSY` 时，每 250ms 重试一次
- 无限重试（返回 `true` 表示继续等待）
- 通过警告日志提示用户

### 连接生命周期

**重要设计决策**：`SQLiteDatabase` **不存储连接对象**，只存储路径。每次查询都新建连接。

[sqlite.rs:28-30](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/values/sqlite.rs#L28-L30) 注释说明了原因：
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
       │    └─ 创建 rusqlite::Connection
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

[sqlite.rs:425-434](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/values/sqlite.rs#L425-L434)

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
- `conn`: 已打开的数据库连接
- `sql`: 带 span 信息的 SQL 语句
- `params`: 查询参数（位置或命名）
- `signals`: 取消信号
- `column_adapters`: 列适配器（如时间戳转换）

#### `prepared_statement_to_nu_list` — 语句结果转 Nu 列表

[sqlite.rs:595-663](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/values/sqlite.rs#L595-L663)

这是结果处理的核心函数，步骤如下：

1. **列信息提取**：通过 `stmt.columns()` 获取所有列的名称和声明类型
2. **参数绑定与执行**：根据参数类型选择不同的 `query_map` 调用
3. **行收集**：遍历结果集，逐行转换
4. **信号检查**：每行检查一次取消信号

---

## 参数绑定机制

### 参数类型枚举

[sqlite.rs:480-489](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/values/sqlite.rs#L480-L489)

```rust
pub enum NuSqlParams {
    List(Vec<Box<dyn ToSql>>),      // 位置参数
    Named(Vec<(String, Box<dyn ToSql>)>),  // 命名参数
}
```

### Nu 值 → SQL 参数转换

#### `nu_value_to_params` — 入口函数

[sqlite.rs:491-532](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/values/sqlite.rs#L491-L532)

根据输入值类型分派：

| 输入类型 | 参数类型 | 说明 |
|----------|----------|------|
| `Value::Record` | `Named` | 记录字段名作为参数名，自动加 `:` 前缀 |
| `Value::List` | `List` | 列表元素按顺序作为位置参数 |
| `Value::Nothing` | `List` (空) | 无参数 |
| 其他 | 错误 | 类型不匹配 |

**命名参数自动补前缀**：如果参数名不以 `:`、`@` 或 `$` 开头，自动插入 `:`。

#### `value_to_sql` — 单值转换

[sqlite.rs:437-467](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/values/sqlite.rs#L437-L467)

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

[sqlite.rs:628-660](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/values/sqlite.rs#L628-L660)

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

---

## 结果映射：SQLite → Nu Value

### 行转换：`convert_sqlite_row_to_nu_value`

[sqlite.rs:693-719](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/values/sqlite.rs#L693-L719)

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

[sqlite.rs:581-593](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/values/sqlite.rs#L581-L593)

```rust
pub struct TypedColumn {
    pub name: String,
    pub decl_type: Option<DeclType>,
}
```

`DeclType` 目前只特殊处理 `JSON` 和 `JSONB` 类型列。

### 值转换：`convert_sqlite_value_to_nu_value`

[sqlite.rs:753-774](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/values/sqlite.rs#L753-L774)

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

[sqlite.rs:721-728](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/values/sqlite.rs#L721-L728)

适配器用于对特定列进行语义转换：

| 适配器 | 作用 |
|--------|------|
| `UnixMillisToDate` | 整数列（Unix 时间戳毫秒）→ `Value::Date` |
| `MillisToDuration` | 整数列（毫秒数）→ `Value::Duration` |

转换逻辑在 `convert_sqlite_value_to_nu_value_with_adapter` 中实现：
[sqlite.rs:729-751](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/values/sqlite.rs#L729-L751)

**注意**：只对 `Integer` 类型应用适配器，其他类型回退到普通转换。

---

## 大结果集处理

### 当前实现：全量加载

**现状**：当前实现使用 `collect_row_values` 将所有行一次性收集到 `Vec<Value>` 中。

[sqlite.rs:608-623](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/values/sqlite.rs#L608-L623)

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

`history` 命令返回 `SQLiteQueryBuilder` 而不是立即执行查询：

[history_.rs:156-185](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-cli/src/commands/history/history_.rs#L156-L185)

这样用户可以通过管道操作（如 `first`、`where` 等）触发 SQL 下推优化，避免全量加载。

---

## 懒查询构建器与 SQL 下推优化

### `SQLiteQueryBuilder` 结构

[sqlite.rs:808-821](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/values/sqlite.rs#L808-L821)

```rust
pub struct SQLiteQueryBuilder {
    pub db_path: PathBuf,
    pub table_name: String,
    pub sql_select: Option<String>,
    pub sql_where: Option<String>,
    pub sql_params: Vec<String>,
    pub sql_order_by: Option<String>,
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

[sqlite.rs:923-940](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/values/sqlite.rs#L923-L940)

```rust
pub fn build_sql(&self) -> String {
    let select = self.sql_select.as_deref().unwrap_or("*");
    let mut sql = format!("SELECT {} FROM [{}]", select, self.table_name);
    // 追加 WHERE / ORDER BY / LIMIT
}
```

#### 3. 列投影下推

[sqlite.rs:894-921](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/values/sqlite.rs#L894-L921)

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
| `follow_path_int` | 执行后按索引访问（未优化） |
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
│      └─ 逐行 convert_sqlite_row_to_nu_value        │
└──────────────────────┬──────────────────────────────┘
                       │ Value::List
                       ▼
                  输出结果
```

### 场景 2：写入数据

```
用户: [[a b]; [1 2] [3 4]] | into sqlite test.db -t my_table
      │
      ▼
┌─────────────────────────────────────────────────────┐
│ into sqlite 命令                                    │
│   ├─ 打开/创建数据库连接                            │
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

### 场景 3：懒查询与下推优化

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

---

## 相关文件索引

| 文件路径 | 主要内容 |
|---------|---------|
| [database/values/sqlite.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/values/sqlite.rs) | 核心实现：连接、查询、转换、构建器 |
| [database/commands/query_db.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/commands/query_db.rs) | query db 命令 |
| [database/commands/into_sqlite.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/commands/into_sqlite.rs) | into sqlite 命令 |
| [database/commands/schema.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/database/commands/schema.rs) | schema 命令 |
| [filesystem/open.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/filesystem/open.rs) | open 命令（SQLite 检测入口） |
| [stor/open.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-command/src/stor/open.rs) | stor open 命令（内存数据库入口） |
| [commands/history/history_.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/73-nushell/crates/nu-cli/src/commands/history/history_.rs) | history 命令（懒查询示例） |
