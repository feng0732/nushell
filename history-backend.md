# Nushell 历史记录后端：Text vs SQLite 全流程解析

本文档深入解析 Nushell 中命令历史记录的后端选择、写入、去重和跨会话合并机制。

## 一、配置结构与默认值

历史记录配置定义在 [history.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-protocol/src/config/history.rs) 中，核心结构体是 `HistoryConfig`：

```rust
pub struct HistoryConfig {
    pub max_size: i64,              // 最大条数，默认 100_000
    pub sync_on_enter: bool,        // 每次回车后同步到磁盘，默认 true
    pub file_format: HistoryFileFormat,  // 文件格式
    pub isolation: bool,            // 会话隔离（仅 SQLite 支持），默认 false
    pub path: HistoryPath,          // 存储路径
    pub ignore_space_prefixed: bool, // 忽略空格开头的命令，默认 true
}
```

`HistoryFileFormat` 是一个枚举：
- `Plaintext` → 文件 `history.txt`（默认）
- `Sqlite` → 文件 `history.sqlite3`（需启用 `sqlite` 编译特性）

## 二、后端选择逻辑

### 2.1 编译时特性门控

在 [HistoryFileFormat::from_str](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-protocol/src/config/history.rs#L23-L36) 和 [UpdateFromValue](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-protocol/src/config/history.rs#L38-L52) 中：

- **启用 `sqlite` 特性**：可选 `"sqlite"` 或 `"plaintext"`
- **未启用 `sqlite` 特性**：配置为 `Sqlite` 时自动回退到 `Plaintext`，并发出警告

### 2.2 运行时锁定机制

历史记录的核心参数是**启动锁定**的。在 [repl.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/repl.rs#L305-L309)：

```rust
// Lock the startup-only `$env.config.history.*` options
// (`path`, `max_size`, `file_format`, `isolation`) against further changes.
engine_state.history_locked_after_startup = true;
```

以下字段在启动后不可更改：`path`、`max_size`、`file_format`、`isolation`。
但 `sync_on_enter` 和 `ignore_space_prefixed` 可以运行时修改。

### 2.3 兼容性校验

`isolation`（会话隔离）仅与 SQLite 格式兼容。如果同时设置 `isolation=true` + `file_format=Plaintext`，会发出警告（见 [history.rs L217-L227](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-protocol/src/config/history.rs#L217-L227)）。

### 2.4 后端初始化

后端在 REPL 启动时通过 [setup_history](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/repl.rs#L1274-L1296) → [update_line_editor_history](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/repl.rs#L1341-L1380) 创建：

```rust
let history: Box<dyn reedline::History> = match history.file_format {
    HistoryFileFormat::Plaintext => Box::new(
        FileBackedHistory::with_file(history.max_size as usize, history_path)?,
    ),
    #[cfg(feature = "sqlite")]
    HistoryFileFormat::Sqlite => Box::new(
        SqliteBackedHistory::with_file(
            history_path,
            history_session_id,     // 会话隔离时为 Some(...)
            Some(chrono::Utc::now()),
        )?,
    ),
};
```

关键参数：
- **Plaintext**：传入 `max_size`（文件滚动容量）
- **SQLite**：传入 `history_session_id`（会话隔离标识）和启动时间

## 三、写入流程

### 3.1 整体时序

一次完整的命令执行涉及历史记录写入的完整流程（在 `RunContext` 中，[repl.rs L340-L507](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/repl.rs#L340-L507)）：

```
用户按下 Enter
    │
    ├─► 判断 history_supports_meta（仅 SQLite 为 true）
    │       │
    │       └─► prepare_history_metadata()    ← 阶段①：写入前置元数据
    │
    ├─► 执行命令（parse_operation → do_run_cmd → eval_source）
    │
    └─► fill_in_result_related_history_metadata()  ← 阶段②：补全结果元数据
            │
            └─► 下一轮循环开始时 sync_history()  ← 阶段③：同步到磁盘
```

### 3.2 阶段①：前置元数据写入（SQLite 专属）

[prepare_history_metadata](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/repl.rs#L853-L875) 在命令执行前调用：

```rust
fn prepare_history_metadata(s: &str, hostname: Option<&str>, ...) {
    line_editor.update_last_command_context(&|mut c| {
        c.start_timestamp = Some(chrono::Utc::now());  // 开始时间
        c.hostname = hostname.map(str::to_string);      // 主机名
        c.cwd = engine_state.cwd(None)...;              // 工作目录
        c
    });
}
```

写入字段：`start_timestamp`、`hostname`、`cwd`。

### 3.3 阶段②：结果元数据补全（SQLite 专属）

[fill_in_result_related_history_metadata](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/repl.rs#L880-L899) 在命令执行后调用：

```rust
fn fill_in_result_related_history_metadata(s: &str, ..., cmd_duration: Duration, ...) {
    line_editor.update_last_command_context(&|mut c| {
        c.duration = Some(cmd_duration);                              // 执行耗时
        c.exit_status = stack.get_env_var("LAST_EXIT_CODE")...;      // 退出码
        c
    });
}
```

补写字段：`duration`、`exit_status`。

### 3.4 阶段③：同步到磁盘（sync_on_enter）

在每次 REPL 循环开始时（读取用户输入前），如果 `sync_on_enter=true`，会调用 [sync_history](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/repl.rs#L692-L705)：

```rust
if history.sync_on_enter {
    line_editor.sync_history();  // 将内存中的历史条目刷入磁盘文件
}
```

这确保上一轮执行的命令在接受下一条输入前已持久化。

### 3.5 Plaintext vs SQLite 的写入差异

| 特性 | Plaintext | SQLite |
|------|-----------|--------|
| 存储内容 | 仅 `command_line` | 完整元数据（时间、主机、CWD、耗时、退出码、session_id） |
| 写入触发 | Reedline 内部自动追加 | 通过 `update_last_command_context` 分两阶段补全 |
| 容量控制 | `max_size` 参数，超出时滚动 | SQLite 自行管理，不受 max_size 限制 |
| 文件格式 | 每行一条命令 | SQLite 数据库表结构 |

## 四、去重与忽略机制

### 4.1 空格前缀忽略

由 `ignore_space_prefixed`（默认 `true`）控制。在两处生效：

1. **初始化时**：[update_line_editor_history](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/repl.rs#L1374)
2. **每轮循环时**：[loop_iteration](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/repl.rs#L695-L696)

```rust
line_editor = line_editor
    .with_history_exclusion_prefix(history.ignore_space_prefixed.then_some(" ".into()));
```

即：以空格开头的命令（如 ` secret_command`）不会被写入历史。

### 4.2 Reedline 内置去重

去重逻辑由 Reedline 库内部的 `History` trait 实现处理：
- 连续重复的相同命令会被合并（不重复存储）
- 具体策略取决于 `FileBackedHistory` / `SqliteBackedHistory` 的内部实现

## 五、跨会话合并与会话隔离

### 5.1 会话隔离（isolation）

当 `isolation=true`（仅 SQLite 可用）时，在 [setup_history](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/repl.rs#L1280-L1284) 中：

```rust
let history_session_id = if history.isolation {
    Reedline::create_history_session_id()  // 生成唯一会话 ID
} else {
    None
};
```

生成的 `session_id` 会：
1. 存入 `engine_state.history_session_id`
2. 作为参数传入 `SqliteBackedHistory::with_file`
3. 通过 `history session` 命令查询（见 [history_session.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/commands/history/history_session.rs)）

**效果**：启用隔离后，每个 Nu 会话只能看到自己写入的历史条目（通过 `session_id` 过滤），不同 shell 窗口互不干扰。

### 5.2 跨会话合并（sync_history）

当 `isolation=false` 时，`sync_history()` 实现跨会话合并：
- SQLite 后端：所有会话共享同一个数据库，无 session_id 过滤，直接读取所有历史
- Plaintext 后端：多个会话同时写入同一个文本文件，Reedline 内部负责文件锁和合并

`sync_history()` 的调用时机是**每轮 REPL 循环开始时**（即读取下一条用户输入之前），这样可以拉取其他会话在这期间写入的历史。

### 5.3 会话 ID 同步

测试 [are_session_ids_in_sync](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/repl.rs#L1616-L1633) 验证了 `line_editor` 中的 `history_session_id` 与 `engine_state.history_session_id` 保持一致。

## 六、查询与命令

### 6.1 `history` 命令

[history_.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/commands/history/history_.rs) 中的 `run` 函数根据格式分别处理：

**Plaintext 格式**：
```rust
// 构造 FileBackedHistory 读取器，通过 SearchQuery::everything 全量搜索
FileBackedHistory::with_file(history.max_size as usize, history_path)
    .search(SearchQuery::everything(SearchDirection::Forward, None))
```
返回字段：`command`、`index`。

**SQLite 格式**：
```rust
// 返回一个惰性 SQLiteQueryBuilder，指向 history 表
nu_command::SQLiteQueryBuilder::new(history_path, "history", signals)
    .with_select("start_timestamp, command_line as command, cwd, duration_ms as duration, exit_status")
    .with_order_by("rowid ASC")
```
返回字段（`--long` 时更多）：`start_timestamp`、`command`、`cwd`、`duration`、`exit_status`、`item_id`、`session_id`、`hostname`。

### 6.2 `history import` 命令

[history_import.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/commands/history/history_import.rs) 实现两种导入模式：

1. **无输入**：自动在 Plaintext ↔ SQLite 之间互转
   ```
   当前 sqlite → 从 history.txt 读取导入到 history.sqlite3
   当前 plaintext → 从 history.sqlite3 读取导入到 history.txt
   ```

2. **管道输入**：从管道导入命令行或详细记录
   ```
   echo foo | history import              # 导入单条命令
   [[command cwd]; [foo /home]] | history import  # 导入带元数据的记录
   ```

导入核心逻辑（[import](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/commands/history/history_import.rs#L149-L159)）：
```rust
fn import(dst: &mut dyn History, src: impl Iterator<Item = Result<HistoryItem, ShellError>>) {
    for item in src {
        let mut item = item?;
        item.id = None;  // 强制重置 ID，让后端重新分配
        dst.save(item)?;
    }
}
```

### 6.3 `history session` 命令

[history_session.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/commands/history/history_session.rs) 直接返回 `engine_state.history_session_id`。

### 6.4 字段定义

历史字段名统一在 [fields.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/commands/history/fields.rs) 中定义：
- 通用：`command`（COMMAND_LINE）
- SQLite 专用：`start_timestamp`、`hostname`、`cwd`、`exit_status`、`duration`、`session_id`

## 七、启动初始化完整流程

```
nu 启动
  │
  ├─► 读取 $env.config.history（env.nu / config.nu）
  │       │
  │       ├─► file_format: Plaintext(默认) | Sqlite
  │       ├─► path: Default | Custom | Disabled
  │       ├─► isolation: 仅 Sqlite 有效
  │       └─► 编译特性检查（无 sqlite 特性则回退）
  │
  ├─► setup_history()
  │       │
  │       ├─► 若 isolation=true → create_history_session_id()
  │       ├─► 计算实际文件路径（file_path()）
  │       └─► update_line_editor_history()
  │               │
  │               ├─► Plaintext → FileBackedHistory::with_file(max_size, path)
  │               └─► Sqlite    → SqliteBackedHistory::with_file(path, session_id, now)
  │                       │
  │                       └─► store_history_id_in_engine()
  │
  └─► engine_state.history_locked_after_startup = true
          （锁定 path/max_size/file_format/isolation 不再变更）
```

## 八、文件路径解析

[HistoryConfig::file_path()](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-protocol/src/config/history.rs#L98-L114) 的解析规则：

1. `HistoryPath::Disabled` → 返回 `None`（不持久化）
2. `HistoryPath::Custom(path)` → 使用用户指定路径
   - 若该路径是目录，则追加默认文件名（`history.txt` 或 `history.sqlite3`）
3. `HistoryPath::Default` → 使用 `nu_config_dir()/默认文件名`

默认文件名由 `HistoryFileFormat::default_file_name()` 返回：
- Plaintext → `history.txt`
- Sqlite → `history.sqlite3`
