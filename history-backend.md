# Nushell 历史记录后端：Text vs SQLite 全流程解析

本文档对照 Reedline 库源码，深入解析 Nushell 中命令历史记录的后端选择、保存、读取、去重和跨会话合并机制。

Reedline 相关源码文件（nushell/reedline 仓库）：
- `src/history/file_backed.rs` — `FileBackedHistory`（Plaintext）
- `src/history/sqlite_backed.rs` — `SqliteBackedHistory`（SQLite）
- `src/history/base.rs` — `History` trait、`SearchQuery`、`SearchFilter` 等
- `src/history/cursor.rs` — `HistoryCursor` 导航游标
- `src/engine.rs` — Reedline 核心引擎、`submit_buffer` 保存逻辑
- `src/hinter/cwd_aware.rs` — `CwdAwareHinter` 历史提示
- `src/completion/history.rs` — `HistoryCompleter` 历史菜单补全

---

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

---

## 二、后端选择逻辑

### 2.1 编译时特性门控

在 [HistoryFileFormat::from_str](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-protocol/src/config/history.rs#L23-L36) 和 [UpdateFromValue](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-protocol/src/config/history.rs#L38-L52) 中：

- **启用 `sqlite` 特性**：可选 `"sqlite"` 或 `"plaintext"`
- **未启用 `sqlite` 特性**：配置为 `Sqlite` 时自动回退到 `Plaintext`，并发出警告

### 2.2 运行时锁定机制

历史记录的核心参数是**启动锁定**的。在 [repl.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/repl.rs#L305-L309)：

```rust
engine_state.history_locked_after_startup = true;
```

启动后不可更改：`path`、`max_size`、`file_format`、`isolation`。
可运行时修改：`sync_on_enter`、`ignore_space_prefixed`。

### 2.3 兼容性校验

`isolation=true` + `file_format=Plaintext` → 发出警告（Plaintext 不支持隔离）。见 [history.rs L217-L227](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-protocol/src/config/history.rs#L217-L227)。

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
            Some(chrono::Utc::now()),  // session_timestamp：用于隔离判断
        )?,
    ),
};
```

---

## 三、History trait 统一接口

所有历史后端都实现同一个 `History` trait（`base.rs`），核心方法：

| 方法 | 说明 |
|------|------|
| `save(&mut self, h: HistoryItem)` | 保存/更新一条历史 |
| `load(&self, id)` | 按 ID 加载 |
| `search(&self, query: SearchQuery)` | 按查询搜索 |
| `count(&self, query)` | 统计匹配数 |
| `update(&mut self, id, updater)` | 更新已有条目（Text 不支持） |
| `sync(&mut self)` | 同步内存与磁盘 |
| `session()` | 返回当前 session_id |

`HistoryItem` 字段：`id`、`start_timestamp`、`command_line`、`session_id`、`hostname`、`cwd`、`duration`、`exit_status`、`more_info`。

---

## 四、保存链路：三层过滤 + 两种后端差异

保存一条命令的完整链路经过**三层过滤**，按顺序执行。

### 4.1 保存链路全景

```
用户按 Enter
    │
    ▼
submit_buffer()  ← 引擎层（Reedline engine.rs）
    │
    ├─► 层级①：history_exclusion_prefix 空格前缀过滤
    │       ├─► 命中 → 设 FILTERED_ITEM_ID，存入 history_excluded_item（内存）
    │       └─► 未命中 → 调用 history.save()
    │
    ▼
history.save()  ← 后端层（FileBackedHistory / SqliteBackedHistory）
    │
    ├─► 层级②：空命令 / capacity=0 检查
    │
    └─► 层级③：连续重复去重（仅 Plaintext）
            ├─► 命中 → 不保存，返回 id=None
            └─► 未命中 → 真正写入
```

---

### 4.2 层级①：引擎层 — 空格前缀过滤

`Reedline.submit_buffer()`（`engine.rs` L2265）是保存的入口：

```rust
fn submit_buffer(&mut self, prompt: &dyn Prompt) -> io::Result<EventStatus> {
    let buffer = self.editor.get_buffer().to_string();
    if !buffer.is_empty() {
        let mut entry = HistoryItem::from_command_line(&buffer);
        entry.session_id = self.get_history_session_id();  // 先设置 session_id

        if self.history_exclusion_prefix
            .as_ref()
            .map(|prefix| buffer.starts_with(prefix))
            .unwrap_or(false)
        {
            // ★ 被排除的命令：不写入历史，只存在内存中
            entry.id = Some(Self::FILTERED_ITEM_ID);  // i64::MAX
            self.history_last_run_id = entry.id;
            self.history_excluded_item = Some(entry);
        } else {
            // 正常保存到历史后端
            entry = self.history.save(entry).expect("todo: error handling");
            self.history_last_run_id = entry.id;
            self.history_excluded_item = None;
        }
    }
    // ...
}
```

**关键事实**：
- `session_id` 在保存**之前**就被设置到 `entry` 上，由引擎层保证，不是后端自己加的
- 被排除的命令仍然有完整的 `HistoryItem` 结构，只是 `id = FILTERED_ITEM_ID`（`i64::MAX`）
- `history_excluded_item` 存在于 Reedline 实例内存中，不持久化

---

### 4.3 被排除命令的元数据更新

`update_last_command_context()`（`engine.rs` L748）也分两条路径：

```rust
pub fn update_last_command_context(
    &mut self,
    f: &dyn Fn(HistoryItem) -> HistoryItem,
) -> crate::Result<()> {
    match &self.history_last_run_id {
        Some(Self::FILTERED_ITEM_ID) => {
            // 被排除的命令：更新内存中的副本
            self.history_excluded_item = Some(f(self.history_excluded_item.take().unwrap()));
            Ok(())
        }
        Some(r) => self.history.update(*r, f),  // 正常命令：通过后端 update()
        None => Err("No command run".into()),
    }
}
```

**含义**：即使命令因空格前缀被排除，Nushell 仍然可以通过 `update_last_command_context` 为其补全元数据（duration、exit_status 等），但这些元数据只存在内存中，不会写入历史文件/数据库。

`has_last_command_context()` 判断 `history_last_run_id.is_some()` —— 被排除的命令也返回 `true`，因为 `FILTERED_ITEM_ID` 也是 `Some`。

---

### 4.4 层级②③：后端层 — Plaintext 的保存与去重

`FileBackedHistory::save()`（`file_backed.rs`）：

```rust
fn save(&mut self, h: HistoryItem) -> Result<HistoryItem> {
    let entry = h.command_line;
    let entry_id =
        if (self.entries.back() != Some(&entry))   // 层级③：连续重复去重
            && !entry.is_empty()                    // 层级②：空命令不存
            && self.capacity > 0                    // 层级②：容量为0不存
        {
            if self.entries.len() == self.capacity {
                self.entries.pop_front();  // 容量滚动
                self.len_on_disk = self.len_on_disk.saturating_sub(1);
            }
            self.entries.push_back(entry.to_string());
            Some(HistoryItemId::new((self.entries.len() - 1) as i64))
        } else {
            None  // 重复或空，返回 id=None
        };
    Ok(FileBackedHistory::construct_entry(entry_id, entry))
}
```

**Plaintext 保存规则**：
1. 空命令不存
2. capacity=0 不存
3. 与最后一条相同则不存（连续去重）
4. 超出容量丢弃最旧的
5. 仅存 `command_line`，元数据全部为 `None`
6. 返回值 `id=None` 表示没保存，`id=Some` 表示保存成功

---

### 4.5 层级②：后端层 — SQLite 的保存（无内置去重）

`SqliteBackedHistory::save()`（`sqlite_backed.rs`）使用 UPSERT：

```sql
INSERT INTO history
  (id, start_timestamp, command_line, session_id, hostname, cwd,
   duration_ms, exit_status, more_info)
VALUES (:id, :start_timestamp, :command_line, :session_id, :hostname,
        :cwd, :duration_ms, :exit_status, :more_info)
ON CONFLICT (history.id) DO UPDATE SET
  start_timestamp = excluded.start_timestamp,
  command_line    = excluded.command_line,
  ... -- 所有字段全部覆盖
RETURNING id
```

**SQLite 保存规则**：
- `id=None` → INSERT 新条目，自增 id
- `id=Some` → UPDATE 覆盖所有字段（UPSERT）
- **无内置去重**：相同命令会重复写入，去重完全靠上层
- **完整元数据**：9 个字段全部持久化

---

### 4.6 去重规则汇总

| 层级 | 位置 | 规则 | 适用后端 |
|------|------|------|----------|
| ① 空格前缀 | `submit_buffer()` (engine.rs) | 以 `history_exclusion_prefix` 开头的命令整个跳过 save | 全部 |
| ② 空命令 / 零容量 | `History::save()` | 空字符串或 capacity=0 不存 | 全部 |
| ③ 连续重复 | `FileBackedHistory::save()` | 与内存中最后一条相同则跳过 | **仅 Plaintext** |
| ④ 导航去重 | `HistoryCursor` (cursor.rs) | `skip_dupes=true` 跳过与当前命令行相同的条目 | 全部（仅影响导航展示） |
| ⑤ 补全去重 | `HistoryCompleter` | `HashSet` 去重相同命令行 | 全部（仅影响菜单展示） |

> **重要差异**：SQLite 后端的 `save()` 不做任何去重。连续执行相同命令会在数据库中产生多条记录。但**历史导航**（上/下箭头）通过 `HistoryCursor.skip_dupes` 自动跳过重复，**历史菜单补全**通过 `HashSet` 去重。所以用户感知上似乎去重了，但数据库里实际有重复条目。

---

## 五、会话编号（session_id）对各功能的影响

`isolation=true` 时，每个 Nu 会话生成唯一的 `history_session_id`。这个 ID 如何影响不同的历史消费方式？

### 5.1 session_id 的设置与传递

1. **生成**：`Reedline::create_history_session_id()` → 基于当前时间纳秒的 `i64`
2. **存入引擎**：`with_history_session_id(session)` → `self.history_session_id = session`
3. **保存时设置**：`submit_buffer()` 中 `entry.session_id = self.get_history_session_id()`
4. **导航时使用**：`HistoryCursor::new(query, self.get_history_session_id())`

---

### 5.2 历史导航（上/下箭头、前缀搜索）：✅ 受隔离影响

`HistoryCursor`（`cursor.rs`）是历史导航的核心状态机：

```rust
pub struct HistoryCursor {
    query: HistoryNavigationQuery,  // Normal | PrefixSearch | SubstringSearch
    current: Option<HistoryItem>,   // 当前指向的条目
    skip_dupes: bool,               // 是否跳过重复（默认 true）
    session: Option<HistorySessionId>,  // ★ 会话过滤
}
```

每次 `back()` / `forward()` 调用时：

```rust
fn navigate_in_direction(&mut self, history: &dyn History, direction: SearchDirection) -> Result<()> {
    // ...
    let next = history.search(SearchQuery {
        start_id: self.current.as_ref().and_then(|e| e.id),
        direction,
        limit: Some(1),
        filter: self.get_search_filter(),  // ★ 包含 session 过滤
    })?;
    // ...
}

fn get_search_filter(&self) -> SearchFilter {
    let filter = match self.query.clone() {
        HistoryNavigationQuery::Normal(_) => SearchFilter::anything(self.session),
        HistoryNavigationQuery::PrefixSearch(prefix) => {
            SearchFilter::from_text_search(CommandLineSearch::Prefix(prefix), self.session)
        }
        HistoryNavigationQuery::SubstringSearch(substring) => {
            SearchFilter::from_text_search(CommandLineSearch::Substring(substring), self.session)
        }
    };
    // skip_dupes 处理 ...
    filter
}
```

**三种导航模式**：
- `Normal`：空 buffer 或光标不在末尾 → bash 风格逐条遍历
- `PrefixSearch`：非空 buffer 且光标在末尾 → fish 风格前缀搜索
- `SubstringSearch`：Ctrl+R 反向搜索模式

**所有模式都传入 session**，所以：
- `isolation=true` → 只能看到自己写入的 + 会话启动前的历史
- `isolation=false` → `session=None`，看到全部历史

---

### 5.3 历史提示（Hinter）：✅ 受隔离影响

Nushell 默认使用 `CwdAwareHinter`（`cwd_aware.rs`）：

```rust
fn handle(&mut self, line: &str, _pos: usize, history: &dyn History,
          use_ansi_coloring: bool, cwd: &str) -> String {
    let with_cwd = history
        .search(SearchQuery::last_with_prefix_and_cwd(
            line.to_string(),
            cwd.to_string(),
            history.session(),  // ★ 传入当前 session
        ))
        .or_else(|err| {
            // Plaintext 不支持 cwd 过滤，回退到纯前缀
            history.search(SearchQuery::last_with_prefix(
                line.to_string(),
                history.session(),  // ★ 仍然传入 session
            ))
        })
        .unwrap_or_default();
    // ... 取第一条的后半部分作为提示
}
```

**两级回退**：
1. 优先：带 CWD + session 的前缀匹配（SQLite 才支持）
2. 回退：仅前缀 + session（Plaintext / SQLite 都支持）

**两种都传入 session**，所以 hinter 受会话隔离影响。

---

### 5.4 历史菜单（HistoryMenu）：❌ 不受隔离影响

默认 `history_menu` 使用 `ReedlineMenu::HistoryMenu`（当没有配置 `source` 时），内部使用 `HistoryCompleter`（`completion/history.rs`）：

```rust
fn search_unique(completer: &HistoryCompleter, line: &str) -> Result<impl Iterator<Item = HistoryItem>> {
    let values = completer.0.search(SearchQuery::all_that_contain_rev(
        parsed.remainder.to_string(),
    ))?;
    // HashSet 去重 ...
}
```

**关键**：`SearchQuery::all_that_contain_rev(contains)` 的签名只有一个参数！内部实现：

```rust
pub fn all_that_contain_rev(contains: String) -> SearchQuery {
    SearchQuery {
        direction: SearchDirection::Backward,
        // ...
        filter: SearchFilter::from_text_search(CommandLineSearch::Substring(contains), None),
        //                                                                         ↑
        //                                                               session = None！
    }
}
```

**历史菜单的 session 是 `None`**，意味着：
- 即使 `isolation=true`，历史菜单（Ctrl+R 调出的列表）也能看到**所有会话**的历史
- 这是一个有意的设计或已知的不一致

---

### 5.5 `history` 命令：❌ 不受隔离影响

[history_.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/commands/history/history_.rs) 中的实现：

**Plaintext 格式**：
```rust
FileBackedHistory::with_file(history.max_size as usize, history_path)?
    .search(SearchQuery::everything(SearchDirection::Forward, None))
//                                                              ↑
//                                                       session = None
```

**SQLite 格式**：
```rust
SQLiteQueryBuilder::new(history_path, "history", signals)
    .with_select("...")
    .with_order_by("rowid ASC")
```
直接裸查 `history` 表，**完全绕过** History trait 的 `search()` 方法，也就不受 session 过滤条件约束。

---

### 5.6 各功能受隔离影响汇总

| 功能 | 受 isolation 影响？ | 查询函数 | session 参数 |
|------|---------------------|----------|--------------|
| 上/下箭头导航 | ✅ 是 | `SearchFilter::anything(session)` | `self.get_history_session_id()` |
| 前缀搜索（fish 风格） | ✅ 是 | `SearchFilter::from_text_search(Prefix, session)` | 同上 |
| Ctrl+R 反向搜索 | ✅ 是 | `SearchFilter::from_text_search(Substring, session)` | 同上 |
| Hinter 提示（CwdAwareHinter） | ✅ 是 | `last_with_prefix_and_cwd(prefix, cwd, session)` / `last_with_prefix(prefix, session)` | `history.session()` |
| 历史菜单（HistoryMenu） | ❌ 否 | `all_that_contain_rev(contains)` | **`None`** |
| `history` 命令（Plaintext） | ❌ 否 | `everything(Forward, None)` | `None` |
| `history` 命令（SQLite） | ❌ 否 | SQLiteQueryBuilder 直接查表 | 绕过 History trait |

> **结论**：只有「交互导航」（上箭头、前缀搜索、反向搜索、提示）受会话隔离影响。`history` 命令和历史菜单始终能看到完整历史。

---

## 六、搜索查询与会话过滤机制

### 6.1 SearchFilter::anything(session) 的 SQL 展开

在 SQLite 后端的 `construct_query` 中，当 `session` 和 `session_timestamp` 都存在时，会附加：

```sql
(session_id = :session_id OR start_timestamp < :session_timestamp)
```

完整语义：
- `session_id = :session_id` — 当前会话自己写入的
- `start_timestamp < :session_timestamp` — 在本会话启动之前就存在的（来自其他会话的旧历史）

**软隔离模型**：旧历史共享，新写入互不干扰。

---

### 6.2 Plaintext 的 search 能力

Plaintext 后端只支持部分过滤：
- ✅ 命令行：前缀 / 子串 / 精确匹配
- ✅ ID 范围
- ✅ `not_command_line`（导航去重用）
- ❌ 时间范围 → 返回 `HistoryFeatureUnsupported`
- ❌ hostname、cwd、exit_status → 返回 `HistoryFeatureUnsupported`
- ❌ session 过滤 → 根本不识别，等同于无过滤

---

## 七、同步与跨会话共享

### 7.1 Plaintext：sync() 合并

`FileBackedHistory::sync()` 是跨会话共享的唯一通道：

1. **加写锁**（`fd_lock::RwLock`）→ 防并发写乱
2. **读磁盘全部内容** → `foreign_entries`（含其他会话新增的）
3. **合并** → `foreign_entries + own_entries`（本会话未刷盘的）
4. **容量检查** → 超了就截断最旧的
5. **写回磁盘** → 追加 or 全量覆写
6. **更新内存** → `self.entries = foreign_entries`（现在包含全部历史）

`sync()` 调用时机：每轮 REPL 循环开始时（`sync_on_enter=true`）。

---

### 7.2 SQLite：sync() 是空操作

```rust
fn sync(&mut self) -> std::io::Result<()> {
    Ok(())
}
```

因为 SQLite 用 WAL 模式，写入即持久化；每次 `search()` 直接读最新数据，天然跨会话共享。不需要显式 sync。

---

## 八、`history import` 导入机制

[history_import.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/commands/history/history_import.rs)

**自动格式互转**（无输入时）：
```
当前 sqlite    → 从 history.txt  导入到 history.sqlite3
当前 plaintext → 从 history.sqlite3 导入到 history.txt
```

核心导入逻辑：
```rust
fn import(dst: &mut dyn History,
          src: impl Iterator<Item = Result<HistoryItem, ShellError>>) {
    for item in src {
        let mut item = item?;
        item.id = None;  // 强制重置 ID，后端重新分配自增 id
        dst.save(item)?;
    }
}
```

---

## 九、启动初始化完整流程

```
nu 启动
  │
  ├─► 读取 $env.config.history
  │
  ├─► setup_history()
  │       │
  │       ├─► isolation=true → create_history_session_id()
  │       ├─► 计算文件路径
  │       └─► update_line_editor_history()
  │               │
  │               ├─► 创建后端（FileBackedHistory / SqliteBackedHistory）
  │               ├─► with_history_session_id(history_session_id)
  │               ├─► with_history_exclusion_prefix(ignore_space_prefixed.then_some(" "))
  │               ├─► 配置 hinter（默认 CwdAwareHinter，或自定义 ExternalHinter）
  │               └─► store_history_id_in_engine()
  │
  └─► engine_state.history_locked_after_startup = true
```

---

## 十、Text vs SQLite 全面对比

| 维度 | Plaintext | SQLite |
|------|-----------|--------|
| 存储内容 | 仅 `command_line` | 9 字段完整元数据 |
| save 内去重 | ✅ 连续相同去重 | ❌ 无 |
| update 支持 | ❌ | ✅ UPSERT |
| sync 作用 | 加锁读→合并→写回→更新内存 | no-op（WAL 实时持久化） |
| 跨会话共享 | sync 时合并 | search 时天然共享 |
| 会话隔离 | ❌ | ✅（软隔离：旧共享新隔离） |
| 过滤能力 | 仅命令行匹配 | 时间/CWD/hostname/exit_status/session |
| 文件锁 | `fd_lock::RwLock` 应用层 | SQLite 内置 |
| session_id 设置位置 | 引擎层 submit_buffer | 引擎层 submit_buffer |
| 导航可见性 | 全量（session 被忽略） | 受 isolation 约束 |
| 菜单可见性 | 全量 | 全量（菜单用 all_that_contain_rev，session=None） |
| history 命令可见性 | 全量 | 全量（直接查表） |

---

## 十一、文件路径解析

[HistoryConfig::file_path()](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-protocol/src/config/history.rs#L98-L114)：

1. `Disabled` → `None`（不持久化）
2. `Custom(path)` → 用户指定路径；若为目录则追加默认文件名
3. `Default` → `nu_config_dir()/默认文件名`

默认文件名：
- Plaintext → `history.txt`
- Sqlite → `history.sqlite3`
