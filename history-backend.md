# Nushell 历史记录后端：Text vs SQLite 全流程解析

本文档对照 Reedline 库源码，深入解析 Nushell 中命令历史记录的后端选择、保存、读取、去重和跨会话合并机制。

Reedline 相关源码文件（nushell/reedline 仓库）：
- `src/history/file_backed.rs` — `FileBackedHistory`（Plaintext）
- `src/history/sqlite_backed.rs` — `SqliteBackedHistory`（SQLite）
- `src/history/base.rs` — `History` trait、`SearchQuery`、`SearchFilter` 等

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

`HistoryFileFormat` 枚举：
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
- **SQLite**：传入 `history_session_id`（会话隔离标识）和**当前会话启动时间** `session_timestamp`（用于跨会话合并判断）

## 三、History trait 统一接口

Reedline 中所有历史后端都实现同一个 `History` trait（`base.rs`），核心方法如下：

| 方法 | 说明 |
|------|------|
| `save(&mut self, h: HistoryItem)` | 保存一条历史条目，可能去重 |
| `load(&self, id: HistoryItemId)` | 按 ID 加载单条 |
| `search(&self, query: SearchQuery)` | 按查询条件搜索 |
| `count(&self, query: SearchQuery)` | 统计匹配条数 |
| `update(&mut self, id, updater)` | 更新已有条目（SQLite 支持，Text 不支持） |
| `sync(&mut self)` | 同步内存与磁盘 |
| `clear()` / `delete()` | 清空 / 删除单条 |
| `session()` | 返回当前会话 ID |

`HistoryItem` 包含完整元数据：`id`、`start_timestamp`、`command_line`、`session_id`、`hostname`、`cwd`、`duration`、`exit_status`、`more_info`。

---

## 四、保存流程（写入）详解

### 4.1 Nushell 上层时序

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

**阶段①：前置元数据写入（SQLite 专属）**

[prepare_history_metadata](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/repl.rs#L853-L875) 在命令执行前调用：

```rust
line_editor.update_last_command_context(&|mut c| {
    c.start_timestamp = Some(chrono::Utc::now());  // 开始时间
    c.hostname = hostname.map(str::to_string);      // 主机名
    c.cwd = engine_state.cwd(None)...;              // 工作目录
    c
});
```

**阶段②：结果元数据补全（SQLite 专属）**

[fill_in_result_related_history_metadata](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/repl.rs#L880-L899) 在命令执行后调用：

```rust
line_editor.update_last_command_context(&|mut c| {
    c.duration = Some(cmd_duration);                              // 执行耗时
    c.exit_status = stack.get_env_var("LAST_EXIT_CODE")...;      // 退出码
    c
});
```

**阶段③：同步到磁盘（sync_on_enter）**

在每次 REPL 循环开始时（读取用户输入前），如果 `sync_on_enter=true`，会调用 [sync_history](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/repl.rs#L692-L705)：

```rust
if history.sync_on_enter {
    line_editor.sync_history();  // 将内存中的历史条目刷入磁盘文件
}
```

---

### 4.2 Plaintext 后端保存（FileBackedHistory::save）

Reedline 源码 `file_backed.rs` 中的 `save` 方法：

```rust
fn save(&mut self, h: HistoryItem) -> Result<HistoryItem> {
    let entry = h.command_line;
    // Don't append if the preceding value is identical or the string empty
    let entry_id =
        if (self.entries.back() != Some(&entry))   // ← 关键：与最后一条比较，相同则跳过
            && !entry.is_empty()
            && self.capacity > 0
        {
            if self.entries.len() == self.capacity {
                // History is "full", so we delete the oldest entry first
                self.entries.pop_front();
                self.len_on_disk = self.len_on_disk.saturating_sub(1);
            }
            self.entries.push_back(entry.to_string());
            Some(HistoryItemId::new((self.entries.len() - 1) as i64))
        } else {
            None  // 重复或空，不保存
        };
    Ok(FileBackedHistory::construct_entry(entry_id, entry))
}
```

**Plaintext 保存规则总结**：

1. **连续重复去重**：`self.entries.back() != Some(&entry)` — 与内存中最后一条相同则跳过（不会保存）
2. **空命令不存**：`!entry.is_empty()`
3. **容量为 0 不存**：`self.capacity > 0`
4. **容量滚动**：超出 `capacity` 时 `pop_front()` 丢弃最旧条目
5. **仅保存命令文本**：通过 `construct_entry` 构造，所有元数据字段（时间、主机、CWD、duration、exit_status、session_id）全部设为 `None`

数据结构：`entries: VecDeque<String>` — 仅存命令行字符串的双端队列。

---

### 4.3 SQLite 后端保存（SqliteBackedHistory::save）

Reedline 源码 `sqlite_backed.rs` 中的 `save` 方法使用 **UPSERT**（INSERT ... ON CONFLICT DO UPDATE）：

```rust
fn save(&mut self, mut entry: HistoryItem) -> Result<HistoryItem> {
    let ret: i64 = self
        .db
        .prepare(
            "insert into history
            (id, start_timestamp, command_line, session_id, hostname, cwd,
             duration_ms, exit_status, more_info)
            values (:id, :start_timestamp, :command_line, :session_id, :hostname,
                    :cwd, :duration_ms, :exit_status, :more_info)
            on conflict (history.id) do update set
                start_timestamp = excluded.start_timestamp,
                command_line    = excluded.command_line,
                session_id      = excluded.session_id,
                hostname        = excluded.hostname,
                cwd             = excluded.cwd,
                duration_ms     = excluded.duration_ms,
                exit_status     = excluded.exit_status,
                more_info       = excluded.more_info
            returning id",
        )?
        .query_row(
            named_params! {
                ":id": entry.id.map(|id| id.0),
                ":start_timestamp": entry.start_timestamp.map(|e| e.timestamp_millis()),
                ":command_line": entry.command_line,
                ":session_id": entry.session_id.map(|e| e.0),
                ":hostname": entry.hostname,
                ":cwd": entry.cwd,
                ":duration_ms": entry.duration.map(|e| e.as_millis() as i64),
                ":exit_status": entry.exit_status,
                ":more_info": entry.more_info.as_ref()
                    .map(|e| serde_json::to_string(e).unwrap())
            },
            |row| row.get(0),
        )?;
    entry.id = Some(HistoryItemId::new(ret));
    Ok(entry)
}
```

**SQLite 保存规则总结**：

1. **无内置去重**：与 FileBackedHistory 不同，`save` 方法本身**不检查**命令是否与上一条重复。重复检查由上层（Reedline 引擎或 Nushell `ignore_space_prefixed`）负责
2. **UPSERT 语义**：
   - 若 `entry.id = None`（新条目）：执行 `INSERT`，自动生成自增 `id`
   - 若 `entry.id = Some(...)`（已有条目）：执行 `UPDATE`，覆盖所有字段
3. **两阶段写入利用 UPSERT**：Nushell 的 `prepare_history_metadata` 先 `INSERT`（id=None）得到 id，然后 `fill_in_result_related_history_metadata` 通过 `update` 方法（内部 load+save）用**相同 id** 补全剩余字段
4. **完整元数据存储**：时间戳、session_id、hostname、cwd、duration_ms、exit_status、more_info(JSON) 全部持久化

数据库表结构（`sqlite_backed.rs` 的 `from_connection` 中创建）：

```sql
create table if not exists history (
    id            integer primary key autoincrement,
    command_line  text not null,
    start_timestamp integer,
    session_id    integer,
    hostname      text,
    cwd           text,
    duration_ms   integer,
    exit_status   integer,
    more_info     text
) strict;
```

索引：`idx_history_time`、`idx_history_cwd`、`idx_history_exit_status`、`idx_history_cmd`、`idx_history_cmd`(session_id)。

性能优化 PRAGMA：`journal_mode=wal`、`synchronous=normal`、`mmap_size=1000000000`、`foreign_keys=on`。

---

## 五、重复命令处理规则（完整）

重复命令过滤发生在**三个层级**，按顺序依次生效：

### 层级 1：Nushell 配置 `ignore_space_prefixed`

由 `HistoryConfig.ignore_space_prefixed`（默认 `true`）控制。在两处生效：

1. **初始化时**：[update_line_editor_history](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/repl.rs#L1374)
2. **每轮循环时**：[loop_iteration](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/repl.rs#L695-L696)

```rust
line_editor = line_editor
    .with_history_exclusion_prefix(
        history.ignore_space_prefixed.then_some(" ".into())
    );
```

效果：**以空格开头的命令**（如 ` secret_command`）在 Reedline 引擎层面就被排除，根本不会调用 `History::save()`。

### 层级 2：Reedline 引擎内部

Reedline 核心引擎在调用 `History::save()` 之前还会进行检查（如 `HISTORY_IGNORE` 模式匹配、是否已存在等）。

### 层级 3：后端 `save()` 方法内部

| 后端 | 去重行为 | 代码位置 |
|------|----------|----------|
| **Plaintext** (`FileBackedHistory`) | `self.entries.back() != Some(&entry)`：**连续相同**则不重复保存 | `file_backed.rs` 的 `save()` |
| **SQLite** (`SqliteBackedHistory`) | **无内置去重**，相同命令会重复写入（除非上层已过滤） | `sqlite_backed.rs` 的 `save()` |

**重要差异**：Plaintext 后端会在 `save()` 内对连续重复命令进行静默去重；而 SQLite 后端完全依赖上层过滤，`save()` 本身不做任何去重判断。

---

## 六、读取与搜索流程详解

### 6.1 查询构造（SearchQuery / SearchFilter）

所有搜索都通过 `SearchQuery` 结构体（`base.rs`），它包含：

- `direction`：`Forward`（从旧到新）或 `Backward`（从新到旧，默认）
- `start_time` / `end_time`：时间范围过滤
- `start_id` / `end_id`：ID 范围过滤
- `limit`：结果条数限制
- `filter`：`SearchFilter`，含 `command_line`(前缀/子串/精确)、`hostname`、`cwd_exact`、`cwd_prefix`、`exit_successful`、`session`

常用查询构造函数：
- `SearchQuery::everything(direction, session)` — 全量
- `SearchQuery::all_that_contain_rev(contains)` — 反向子串搜索
- `SearchQuery::last_with_prefix(prefix, session)` — 最近前缀匹配

---

### 6.2 Plaintext 后端搜索（FileBackedHistory::search）

Reedline `file_backed.rs`：

```rust
fn search(&self, query: SearchQuery) -> Result<Vec<HistoryItem>> {
    // 1. 不支持的过滤直接报错
    if query.start_time.is_some() || query.end_time.is_some() {
        return Err(HistoryFeatureUnsupported { feature: "filtering by time" });
    }
    if query.filter.hostname.is_some() || query.filter.cwd_exact.is_some()
        || query.filter.cwd_prefix.is_some() || query.filter.exit_successful.is_some()
    {
        return Err(HistoryFeatureUnsupported { feature: "filtering by extra info" });
    }

    // 2. 计算 ID 范围（min_id, max_id）
    let (min_id, max_id) = { /* 根据 direction 交换 start_id/end_id */ };

    // 3. 迭代 entries，应用过滤器
    let filter = |(idx, cmd): (usize, &String)| {
        // 命令行匹配：Prefix / Substring / Exact
        if !match &query.filter.command_line {
            Some(CommandLineSearch::Prefix(p)) => cmd.starts_with(p),
            Some(CommandLineSearch::Substring(p)) => cmd.contains(p),
            Some(CommandLineSearch::Exact(p)) => cmd == p,
            None => true,
        } { return None; }
        // 排除当前命令（not_command_line，用于上箭头导航时避免重复）
        if let Some(str) = &query.filter.not_command_line {
            if cmd == str { return None; }
        }
        Some(FileBackedHistory::construct_entry(
            Some(HistoryItemId::new(idx as i64)),
            cmd.to_string(),
        ))
    };

    // 4. 根据方向 forward/rev 收集结果
    let iter = self.entries.iter().enumerate()
        .skip(min_id as usize).take(intrinsic_limit as usize);
    if let SearchDirection::Backward = query.direction {
        Ok(iter.rev().filter_map(filter).take(limit).collect())
    } else {
        Ok(iter.filter_map(filter).take(limit).collect())
    }
}
```

**Plaintext 搜索能力限制**：
- ✅ 支持：ID 范围、命令行前缀/子串/精确匹配、方向、limit、not_command_line
- ❌ 不支持：时间范围、hostname、cwd（精确/前缀）、exit_status 成功过滤（这些字段根本没有存储）

---

### 6.3 SQLite 后端搜索（SqliteBackedHistory::search + construct_query）

Reedline `sqlite_backed.rs` 的 `construct_query` 方法将 `SearchQuery` 动态翻译成 SQL，这是理解**会话过滤**和**跨会话合并**的关键：

```rust
fn construct_query<'a>(
    &self,
    query: &'a SearchQuery,
    select_expression: &str,   // "*" 或 "coalesce(count(*), 0)"
) -> (String, BoxedNamedParams<'a>)
```

以下逐步拆解 SQL 构造逻辑：

**步骤 1：方向与排序**

```rust
let (is_asc, asc) = match query.direction {
    SearchDirection::Forward  => (true, "asc"),
    SearchDirection::Backward => (false, "desc"),
};
// ORDER BY id {asc}
```

**步骤 2：时间/ID 范围**

```rust
// start_time: Forward → "start_timestamp > :start_time"
//             Backward → "start_timestamp < :start_time"
// 类似处理 end_time、start_id、end_id
```

**步骤 3：命令行匹配**

```rust
match command_line {
    CommandLineSearch::Exact(e)   => "command_line == :command_line"
    CommandLineSearch::Prefix(prefix) => "instr(command_line, :command_line) == 1"
    CommandLineSearch::Substring(cont) => "instr(command_line, :command_line) >= 1"
}
```

**步骤 4：hostname / CWD / exit_status**

```rust
hostname: "hostname = :hostname"
cwd_exact: "cwd = :cwd"
cwd_prefix: "cwd like :cwd_like"   -- 如 "/home/me%"
exit_successful: "exit_status = 0" 或 "exit_status != 0"
```

**步骤 5（核心）：会话过滤与跨会话合并**

```rust
if let (Some(session_id), Some(session_timestamp)) =
    (query.filter.session, self.session_timestamp)
{
    // ★ 关键条件 ★
    wheres.push("(session_id = :session_id OR start_timestamp < :session_timestamp)");
    params.push((":session_id", Box::new(session_id)));
    params.push((":session_timestamp",
                 Box::new(session_timestamp.timestamp_millis())));
}
```

这是整个历史系统最精妙的设计。完整 SQL 形如：

```sql
SELECT * FROM history
WHERE (
    -- 其他条件 AND
    (session_id = :session_id OR start_timestamp < :session_timestamp)
)
ORDER BY id desc
LIMIT :limit
```

**条件含义**：返回的历史条目满足以下**任一**条件：
1. `session_id = :session_id` — **当前会话自己写入**的条目
2. `start_timestamp < :session_timestamp` — **在当前会话启动之前**写入的条目（来自其他会话或历史会话）

**最终效果**：
- `isolation=false`（`session=None`）：`if let` 分支不执行，**无条件返回所有历史** → 完全跨会话共享
- `isolation=true`（`session=Some(id)`）：只看自己写入的 + 自己启动前的历史 → **启动后其他新会话写入的条目不可见**，实现"软隔离"

这解释了为什么 `SqliteBackedHistory::with_file` 需要同时接收 `session_id` **和** `session_timestamp` 两个参数。

---

### 6.4 Nushell `history` 命令中的查询

[history_.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/commands/history/history_.rs) 中的 `run` 函数：

**Plaintext**：
```rust
FileBackedHistory::with_file(history.max_size as usize, history_path)?
    .search(SearchQuery::everything(SearchDirection::Forward, None))
```
调用 `SearchQuery::everything(Forward, None)` — `session=None`，无隔离，返回全部。

**SQLite**：
```rust
nu_command::SQLiteQueryBuilder::new(history_path, "history", signals)
    .with_select("start_timestamp, command_line as command, cwd,
                  duration_ms as duration, exit_status")
    .with_order_by("rowid ASC")
```
`SQLiteQueryBuilder` 直接裸查 `history` 表，**绕过** Reedline 的 `History::search()`，因此不受会话过滤条件约束 —— 这意味着 `history` 命令始终可以看到完整历史，不受 isolation 影响。

---

## 七、同步到磁盘（sync）与跨会话共享

### 7.1 Plaintext 后端同步（FileBackedHistory::sync）

这是 Plaintext 实现跨会话共享的核心。Reedline `file_backed.rs`：

```rust
fn sync(&mut self) -> std::io::Result<()> {
    if let Some(fname) = &self.file {
        // 1. 计算尚未写入磁盘的内存条目
        let own_entries = self.entries.range(self.len_on_disk..);

        // 2. ★ 文件锁 ★ — 防止多进程并发写冲突
        let mut f_lock = fd_lock::RwLock::new(
            OpenOptions::new()
                .create(true).write(true).read(true).truncate(false)
                .open(fname)?,
        );
        let mut writer_guard = f_lock.write()?;

        // 3. 读取磁盘上已有的全部条目（可能来自其他会话）
        let (mut foreign_entries, truncate) = {
            let reader = BufReader::new(writer_guard.deref());
            let mut from_file: VecDeque<_> = reader.lines()
                .map(|o| o.map(|i| decode_entry(&i)))
                .collect::<std::io::Result<VecDeque<_>>>()?;
            // 容量检查：总条目 > capacity 则截断旧的
            if from_file.len() + own_entries.len() > self.capacity {
                (from_file.split_off(
                    from_file.len() - (self.capacity.saturating_sub(own_entries.len())),
                ), true)
            } else {
                (from_file, false)
            }
        };

        // 4. 写入
        let mut writer = BufWriter::new(writer_guard.deref_mut());
        if truncate {
            // 需要截断：从头覆写
            writer.rewind()?;
            for line in &foreign_entries {
                writer.write_all(encode_entry(line).as_bytes())?;
                writer.write_all("\n".as_bytes())?;
            }
        } else {
            // 无需截断：追加到末尾
            writer.seek(SeekFrom::End(0))?;
        }
        for line in own_entries {
            writer.write_all(encode_entry(line).as_bytes())?;
            writer.write_all("\n".as_bytes())?;
        }
        writer.flush()?;

        if truncate {
            // 截断到当前写入位置
            let file = writer_guard.deref_mut();
            let file_len = file.stream_position()?;
            file.set_len(file_len)?;
        }

        // 5. 合并到内存：foreign_entries（磁盘的） + own_entries（本会话新的）
        let own_entries = self.entries.drain(self.len_on_disk..);
        foreign_entries.extend(own_entries);
        self.entries = foreign_entries;
        self.len_on_disk = self.entries.len();
    }
}
```

**Plaintext sync 关键点**：

| 步骤 | 作用 |
|------|------|
| **fd_lock::RwLock 写锁** | 多进程安全，防止两个 Nu 同时写乱文件 |
| **先读后写** | 读取磁盘完整内容（含其他会话新增）再合并 |
| **foreign_entries** | 磁盘上的已有条目（含其他会话写入的） |
| **own_entries** | 本会话新增未刷盘的条目（`entries.range(len_on_disk..)`） |
| **容量滚动** | `from_file + own_entries > capacity` 时丢弃最旧条目 |
| **`self.entries = foreign_entries`** | sync 后内存包含磁盘的全部内容（跨会话合并完成） |
| **`len_on_disk = entries.len()`** | 标记全量已同步 |

**跨会话合并流程**（Plaintext）：
```
会话 A 写入 "ls" → entries = ["cd /", "ls"], len_on_disk=1
    ↓ sync()
    加锁 → 读文件得 ["cd /", "pwd"] (会话 B 追加了 "pwd")
         → 合并 foreign_entries=["cd /", "pwd"] + own_entries=["ls"]
         → 容量 OK，追加 "ls" 到文件
         → entries = ["cd /", "pwd", "ls"], len_on_disk=3
    解锁
结果：会话 A 现在能看到会话 B 写入的 "pwd"
```

编码规则：多行命令中的 `\n` 在文件中转义为 `<\n>`（`encode_entry` / `decode_entry`）。

---

### 7.2 SQLite 后端同步（SqliteBackedHistory::sync）

Reedline `sqlite_backed.rs`：

```rust
fn sync(&mut self) -> std::io::Result<()> {
    // no-op (todo?)
    Ok(())
}
```

**SQLite 的 sync 是空操作**。为什么？因为：
1. SQLite 本身使用 **WAL（Write-Ahead Logging）模式**（`journal_mode=wal`），写入已立即持久化
2. 多进程通过 SQLite 内置的**文件锁机制**并发访问同一 DB 文件，不需要应用层加锁
3. 每次 `search()` 查询都直接读取最新 DB 内容，天然包含其他会话已写入的条目

**跨会话合并流程**（SQLite）：
```
会话 A 执行 "ls"
    ↓ save() 立即 INSERT 到 DB（无需等 sync）
会话 B 执行 history 命令或按上箭头
    ↓ search() → SELECT * FROM history WHERE ...
    直接看到会话 A 刚写入的 "ls"（无需 sync 拉取）
```

SQLite 的跨会话共享是**实时**的，不需要 `sync()` 做显式合并。

---

## 八、会话隔离（isolation）完整机制

### 8.1 隔离开启时的初始化流程

当 `isolation=true`（仅 SQLite 可用）时，在 [setup_history](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/repl.rs#L1280-L1284) 中：

```rust
let history_session_id = if history.isolation {
    Reedline::create_history_session_id()  // 生成唯一会话 ID（i64）
} else {
    None
};
```

生成的 `session_id` 会：
1. 存入 `engine_state.history_session_id`
2. 作为参数传入 `SqliteBackedHistory::with_file(file, session_id, Some(now))`
   - `now` 即 `session_timestamp`，用于查询条件
3. 通过 `history session` 命令查询

### 8.2 隔离如何在查询中生效

如 6.3 节所述，SQL 查询会自动附加：

```sql
(session_id = :session_id OR start_timestamp < :session_timestamp)
```

每个会话 `save()` 时，`HistoryItem` 上会带上自己的 `session_id`。因此：

| 场景 | `isolation=false` | `isolation=true` |
|------|-------------------|------------------|
| 看自己写入的 | ✅ | ✅ |
| 看会话启动前的旧历史 | ✅ | ✅ |
| 看**其他会话在本会话启动后**新写入的 | ✅ | ❌ |

这是一种"**写时共享旧历史，之后互不干扰**"的软隔离模型。

`FileBackedHistory` 不支持 isolation，因为它没有 `session_id` 字段，也无法在 `search()` 中按 session 过滤（Plaintext search 会报错 `HistoryFeatureUnsupported`）。

---

## 九、`history import` 命令的导入机制

[history_import.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/commands/history/history_import.rs) 实现两种导入模式：

### 9.1 自动格式互转（无输入）

```
当前 sqlite    → 从 history.txt  读取导入到 history.sqlite3
当前 plaintext → 从 history.sqlite3 读取导入到 history.txt
```

### 9.2 管道输入导入

```
echo foo | history import              # 导入单条命令
[[command cwd]; [foo /home]] | history import  # 导入带元数据的记录
```

核心导入逻辑（[import](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/commands/history/history_import.rs#L149-L159)）：

```rust
fn import(dst: &mut dyn History,
          src: impl Iterator<Item = Result<HistoryItem, ShellError>>) {
    for item in src {
        let mut item = item?;
        item.id = None;  // ★ 强制重置 ID，让后端重新分配自增 id
        dst.save(item)?;
    }
}
```

关键点：`item.id = None` 强制触发 INSERT 而非 UPDATE，保证导入数据作为新条目追加。

---

## 十、启动初始化完整流程

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
  │               │       │        (内部立即 sync() 读入磁盘已有内容)
  │               │       └─► with_history_exclusion_prefix(ignore_space_prefixed)
  │               │
  │               └─► Sqlite    → SqliteBackedHistory::with_file(path, session_id, now)
  │                       │        (建表 + 设 PRAGMA，无需读历史)
  │                       ├─► with_history_exclusion_prefix(ignore_space_prefixed)
  │                       └─► store_history_id_in_engine()
  │
  └─► engine_state.history_locked_after_startup = true
          （锁定 path/max_size/file_format/isolation 不再变更）
```

---

## 十一、Text vs SQLite 全面对比

| 维度 | Plaintext (FileBackedHistory) | SQLite (SqliteBackedHistory) |
|------|-------------------------------|-------------------------------|
| **存储内容** | 仅 `command_line`（字符串） | 完整 9 字段含元数据 |
| **多行编码** | `\n` → `<\n>` 转义 | 原生支持 TEXT |
| **容量控制** | `capacity` 参数，`VecDeque` 滚动 | 无上限（SQLite 自行管理） |
| **save 内去重** | ✅ 连续相同命令不重复存储 | ❌ 无内置去重 |
| **update 支持** | ❌ 不支持 | ✅ UPSERT 覆盖 |
| **sync 作用** | 加锁读文件→合并→写回→更新内存 | no-op（SQLite WAL 实时持久化） |
| **跨会话共享** | sync 时通过文件锁合并 | search 时直接读 DB，天然共享 |
| **会话隔离** | ❌ 不支持 | ✅ `session_id + session_timestamp` 软隔离 |
| **过滤能力** | 仅命令行前缀/子串/精确匹配 | 时间、CWD、hostname、exit_status、session 全部支持 |
| **文件锁** | `fd_lock::RwLock` 应用层 | SQLite 内置文件锁 |
| **history 命令查询** | `FileBackedHistory.search()` 读文件 | `SQLiteQueryBuilder` 直接查 `history` 表（不过滤 session） |

---

## 十二、文件路径解析

[HistoryConfig::file_path()](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-protocol/src/config/history.rs#L98-L114) 的解析规则：

1. `HistoryPath::Disabled` → 返回 `None`（不持久化）
2. `HistoryPath::Custom(path)` → 使用用户指定路径
   - 若该路径是目录，则追加默认文件名（`history.txt` 或 `history.sqlite3`）
3. `HistoryPath::Default` → 使用 `nu_config_dir()/默认文件名`

默认文件名由 `HistoryFileFormat::default_file_name()` 返回：
- Plaintext → `history.txt`
- Sqlite → `history.sqlite3`
