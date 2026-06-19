# Nushell 历史记录后端：Text vs SQLite 全流程解析

本文档对照 Reedline 库源码，深入解析 Nushell 中命令历史记录的后端选择、保存、读取、去重和跨会话合并机制。

Reedline 相关源码文件（nushell/reedline 仓库）：
- `src/history/file_backed.rs` — `FileBackedHistory`（Plaintext）
- `src/history/sqlite_backed.rs` — `SqliteBackedHistory`（SQLite）
- `src/history/base.rs` — `History` trait、`SearchQuery`、`SearchFilter` 等
- `src/history/cursor.rs` — `HistoryCursor` 导航游标
- `src/history/item.rs` — `HistoryItem`、`HistoryItemId`、`HistorySessionId`
- `src/engine.rs` — Reedline 核心引擎、`submit_buffer`、`handle_event`、`enter_history_search`
- `src/hinter/cwd_aware.rs` — `CwdAwareHinter` 历史提示
- `src/completion/history.rs` — `HistoryCompleter` 历史菜单补全

Nushell 相关源码：
- `crates/nu-protocol/src/config/history.rs` — `HistoryConfig`
- `crates/nu-cli/src/repl.rs` — `setup_history`、`prepare_history_metadata` 等
- `crates/nu-cli/src/reedline_config.rs` — 键绑定配置（Ctrl+R / Ctrl+Q）
- `crates/nu-cli/src/commands/history/` — `history` 子命令

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

`HistorySessionId(i64)`：基于启动时的时间纳秒生成，唯一标识一个 Nu 会话。

---

## 四、保存链路：按代码事实校准的过滤层级

保存一条命令的完整链路经过**引擎层**和**后端层**的多重检查，不同层级所属位置和行为有严格区别。

### 4.1 保存链路全景（按调用顺序）

```
用户按 Enter
    │
    ▼
submit_buffer()  ← 引擎层（Reedline engine.rs L2265）
    │
    ├─► 层级 0：空输入跳过（引擎层最外层）
    │       └─► buffer.is_empty() → 整个跳过，不创建 HistoryItem
    │
    ├─► 构造 HistoryItem::from_command_line(&buffer)
    │       并设置 entry.session_id = self.get_history_session_id()
    │
    ├─► 层级 1：空格前缀过滤（引擎层内部）
    │       ├─► 命中 starts_with(exclusion_prefix)
    │       │       → 设 id=FILTERED_ITEM_ID (i64::MAX)
    │       │       → 存入 history_excluded_item（仅内存，不持久化）
    │       │       → 不调用 history.save()
    │       └─► 未命中 → 调用 history.save(entry)
    │
    ▼
history.save()  ← 后端层，行为因后端而异
    │
    ├─► FileBackedHistory（Plaintext）：
    │       ├─► 层级 2a：空命令检查 !entry.is_empty()
    │       ├─► 层级 2b：容量限制 self.capacity > 0
    │       ├─► 层级 2c：连续重复去重（entries.back() != entry）
    │       └─► 全部通过 → VecDeque::push_back（超容量则 pop_front）
    │
    └─► SqliteBackedHistory（SQLite）：
            ├─► 无空命令检查：空字符串也会被 INSERT
            ├─► 无容量限制：max_size 参数未传入 SQLite 后端，不生效
            ├─► 无去重检查：完全依赖上层过滤
            └─► UPSERT：id=None→INSERT（自增）；id=Some→UPDATE 全字段覆盖
```

---

### 4.2 层级 0：引擎层 — 空输入跳过

`Reedline.submit_buffer()`（`engine.rs` L2265）最外层的条件：

```rust
fn submit_buffer(&mut self, prompt: &dyn Prompt) -> io::Result<EventStatus> {
    let buffer = self.editor.get_buffer().to_string();
    if !buffer.is_empty() {          // ← 层级 0：空 buffer 直接跳过
        // ... 以下逻辑仅对非空输入执行
    }
    self.input_behavior = InputBasic;
    Ok(EventStatus::Exits(prompt.submit(&buffer)))
}
```

**行为**：用户直接按 Enter（空命令行）时，不创建 `HistoryItem`，不触发任何保存或元数据更新流程。

这与**层级 2a**（`FileBackedHistory::save` 内部的 `!entry.is_empty()`）是两道独立防线：
- 层级 0 在 `submit_buffer` 入口（引擎层）
- 层级 2a 在 `FileBackedHistory::save` 内部（后端层）—— 即使有其他代码路径绕过层级 0 直接调用 `save()`，也能防住空命令

**SQLite 后端没有层级 2a**：如果有代码路径绕过层级 0 直接向 SQLite `save()` 传入空 `command_line`，空字符串会被成功 INSERT。

---

### 4.3 层级 1：引擎层 — 空格前缀过滤

仍然在 `submit_buffer()` 内部（`engine.rs` L2265）：

```rust
let mut entry = HistoryItem::from_command_line(&buffer);
entry.session_id = self.get_history_session_id();  // ★ session_id 在引擎层设置

if self.history_exclusion_prefix
    .as_ref()
    .map(|prefix| buffer.starts_with(prefix))
    .unwrap_or(false)
{
    // ★ 被排除的命令：不写入历史，只存在 Reedline 内存中
    entry.id = Some(Self::FILTERED_ITEM_ID);  // i64::MAX
    self.history_last_run_id = entry.id;
    self.history_excluded_item = Some(entry);
} else {
    // 正常保存到历史后端
    entry = self.history.save(entry).expect("todo: error handling");
    self.history_last_run_id = entry.id;
    self.history_excluded_item = None;
}
```

**关键事实**：
- `session_id` 在保存**之前**就由引擎层设置到 `entry` 上，不是后端自己加的
- 被排除的命令仍然有完整的 `HistoryItem` 结构（含 session_id、command_line），只是 `id = FILTERED_ITEM_ID`（`i64::MAX`）
- `history_excluded_item` 存在于 Reedline 实例内存中，不持久化，会话结束即丢失

Nushell 配置通过 [repl.rs L695-L696](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/repl.rs#L695-L696) 和 [repl.rs L1374](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/repl.rs#L1374) 传入空格前缀 `" "`：

```rust
line_editor = line_editor
    .with_history_exclusion_prefix(history.ignore_space_prefixed.then_some(" ".into()));
```

---

### 4.4 被排除命令的元数据更新机制

`update_last_command_context()`（`engine.rs` L748）分两条路径：

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
        Some(r) => self.history.update(*r, f),  // 正常命令：调用后端 update()
        None => Err("No command run".into()),
    }
}
```

**含义**：即使命令因空格前缀被排除（不写入历史），Nushell 的 `prepare_history_metadata` 和 `fill_in_result_related_history_metadata` 仍然可以通过 `update_last_command_context` 为其补全元数据。但这些元数据只存在内存中，不会写入历史文件/数据库。

`has_last_command_context()` 判断 `history_last_run_id.is_some()` —— 被排除的命令也返回 `true`，因为 `FILTERED_ITEM_ID` 也是 `Some`。

---

### 4.5 层级 2a/2b/2c：后端层 — Plaintext 的三重检查

`FileBackedHistory::save()`（`file_backed.rs`）：

```rust
fn save(&mut self, h: HistoryItem) -> Result<HistoryItem> {
    let entry = h.command_line;
    let entry_id =
        if (self.entries.back() != Some(&entry))   // 2c：连续重复去重（仅 Plaintext）
            && !entry.is_empty()                    // 2a：空命令检查
            && self.capacity > 0                    // 2b：容量限制（仅 Plaintext）
        {
            if self.entries.len() == self.capacity {
                self.entries.pop_front();  // 容量滚动：满了就丢最旧的
                self.len_on_disk = self.len_on_disk.saturating_sub(1);
            }
            self.entries.push_back(entry.to_string());
            Some(HistoryItemId::new((self.entries.len() - 1) as i64))
        } else {
            None  // 任一条件不满足，返回 id=None 表示未保存
        };
    Ok(FileBackedHistory::construct_entry(entry_id, entry))
}
```

**Plaintext 保存规则（按优先级）**：

1. **空命令不存**（2a）：`!entry.is_empty()` —— 空字符串不存
2. **零容量不存**（2b）：`self.capacity > 0` —— 容量为 0 时所有命令都不存
3. **连续重复不存**（2c）：`entries.back() != entry` —— 与内存中最后一条相同则跳过
4. **容量滚动**：达到 `capacity` 时 `pop_front()` 丢弃最旧条目
5. **仅存命令文本**：通过 `construct_entry` 构造返回值，元数据（时间、主机、CWD、duration、exit_status、session_id）全部设为 `None`
6. **返回值语义**：`id=None` → 未实际保存；`id=Some(i)` → 已保存，i 为在 VecDeque 中的下标

**capacity 的来源**：Nushell 通过 `FileBackedHistory::with_file(history.max_size as usize, ...)` 传入，对应 `HistoryConfig.max_size`（默认 100_000）。

---

### 4.6 SQLite 后端保存：无内置检查，纯 UPSERT

`SqliteBackedHistory::save()`（`sqlite_backed.rs`）—— **不做任何检查**：

```sql
INSERT INTO history
  (id, start_timestamp, command_line, session_id, hostname, cwd,
   duration_ms, exit_status, more_info)
VALUES (:id, :start_timestamp, :command_line, :session_id, :hostname,
        :cwd, :duration_ms, :exit_status, :more_info)
ON CONFLICT (history.id) DO UPDATE SET
  start_timestamp = excluded.start_timestamp,
  command_line    = excluded.command_line,
  session_id      = excluded.session_id,
  hostname        = excluded.hostname,
  cwd             = excluded.cwd,
  duration_ms     = excluded.duration_ms,
  exit_status     = excluded.exit_status,
  more_info       = excluded.more_info
RETURNING id
```

**SQLite 保存事实**：

| 检查项 | 是否存在 | 说明 |
|--------|----------|------|
| 空命令检查（层级 2a） | ❌ 不存在 | 空字符串会被 INSERT（但层级 0 通常已拦住） |
| 容量限制（层级 2b） | ❌ 不存在 | `max_size` 参数**未传入** SQLite 后端，完全不生效 |
| 连续重复去重（层级 2c） | ❌ 不存在 | 相同命令会在 DB 中产生多条 id 不同的记录 |
| UPSERT 语义 | ✅ 存在 | `id=None` → INSERT（自增 id）；`id=Some` → UPDATE 覆盖 |
| 元数据存储 | ✅ 完整 | 9 个字段全部持久化 |

**Nushell 对 UPSERT 的利用**：
- 阶段① `prepare_history_metadata`：新命令 id=None → INSERT，自增生成 id，写入 `start_timestamp`、`hostname`、`cwd`
- 阶段② `fill_in_result_related_history_metadata`：通过 `update(id, f)` → load + save（同一个 id）→ UPDATE，补全 `duration`、`exit_status`

数据库表结构（`sqlite_backed.rs` 中创建）：

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

性能优化 PRAGMA：`journal_mode=wal`、`synchronous=normal`、`mmap_size=1000000000`、`foreign_keys=on`。

---

### 4.7 去重规则汇总（完整且校准后）

| 层级 | 代码位置 | 规则 | 适用范围 |
|------|----------|------|----------|
| **0** 空输入跳过 | `submit_buffer()` 最外层（engine.rs L2265） | `buffer.is_empty()` → 整个跳过 | 全部后端 |
| **1** 空格前缀 | `submit_buffer()` 内部（engine.rs） | `starts_with(" ")` → 跳过 `save()`，仅存内存 | 全部后端 |
| **2a** 空命令 | `FileBackedHistory::save()` | `!entry.is_empty()` | **仅 Plaintext** |
| **2b** 零容量 | `FileBackedHistory::save()` | `capacity > 0` | **仅 Plaintext** |
| **2c** 连续重复 | `FileBackedHistory::save()` | `entries.back() != entry` | **仅 Plaintext** |
| **展示层** 导航去重 | `HistoryCursor::get_search_filter()` | `skip_dupes=true` + `not_command_line` | 全部后端（仅影响导航显示） |
| **展示层** 菜单去重 | `HistoryCompleter::search_unique()` | `HashSet` 按 `command_line` 去重 | 全部后端（仅影响菜单显示） |

> **SQLite 去重真相**：数据库里会有重复记录（连续相同命令多条），但用户在**交互界面上感知不到**——因为导航（上箭头）通过 `HistoryCursor.skip_dupes` 自动跳过与当前命令行相同的条目，菜单（Ctrl+R）通过 `HashSet` 去重。只有用 `history` 命令查询 SQLite 原始表时才能看到重复。

---

## 五、快捷键与交互：Ctrl+R（菜单）vs Ctrl+Q（搜索模式）

### 5.1 键绑定定义（校准后）

Nushell 在 [reedline_config.rs L773-L805](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/reedline_config.rs#L773-L805) 中定义：

```rust
// Ctrl+R → 调出历史菜单
keybindings.add_binding(
    KeyModifiers::CONTROL,
    KeyCode::Char('r'),
    ReedlineEvent::Menu("history_menu".to_string()),
);

// Ctrl+Q → 进入反向历史搜索模式
keybindings.add_binding(
    KeyModifiers::CONTROL,
    KeyCode::Char('q'),
    ReedlineEvent::SearchHistory,
);
```

**两个快捷键走的是完全不同的代码路径**，会话隔离行为也截然不同。

---

### 5.2 Ctrl+R（历史菜单）：❌ 不受会话隔离影响

Ctrl+R 触发 `ReedlineEvent::Menu("history_menu")`，在 `handle_editor_event`（`engine.rs` L1151）中激活菜单：

```rust
ReedlineEvent::Menu(name) => {
    if let Some(menu) = self.menus.iter_mut().find(|menu| menu.name() == name) {
        menu.menu_event(MenuEvent::Activate(self.quick_completions));
        // ... 后续更新菜单值时调用 HistoryCompleter
    }
}
```

`HistoryCompleter`（`completion/history.rs`）的 `search_unique`：

```rust
fn search_unique(
    completer: &HistoryCompleter,
    line: &str,
) -> Result<impl Iterator<Item = HistoryItem>> {
    let values = completer.0.search(SearchQuery::all_that_contain_rev(
        parsed.remainder.to_string(),
    ))?;
    // HashSet 按 command_line 去重 ...
}
```

关键在于 `SearchQuery::all_that_contain_rev(contains)` 只有一个参数！内部实现（`base.rs`）：

```rust
pub fn all_that_contain_rev(contains: String) -> SearchQuery {
    SearchQuery {
        direction: SearchDirection::Backward,
        filter: SearchFilter::from_text_search(
            CommandLineSearch::Substring(contains),
            None,   // ★ session = None！
        ),
        // ...
    }
}
```

**session=None → SQLite 后端的 `construct_query` 不会附加 `(session_id = :session_id OR ...)` 条件 → 返回全部历史，不受 isolation 约束。**

---

### 5.3 Ctrl+Q（反向历史搜索模式）：✅ 受会话隔离影响

Ctrl+Q 触发 `ReedlineEvent::SearchHistory`，在 `handle_editor_event`（`engine.rs` L1474）中：

```rust
ReedlineEvent::SearchHistory => {
    self.enter_history_search();
    Ok(EventStatus::Handled)
}
```

`enter_history_search()`（`engine.rs` L1642）：

```rust
fn enter_history_search(&mut self) {
    self.history_cursor = HistoryCursor::new(
        HistoryNavigationQuery::SubstringSearch("".to_string()),
        self.get_history_session_id(),  // ★ session=Some(当前会话ID)！
    );
    self.input_mode = InputMode::HistorySearch;
}
```

**进入 `InputMode::HistorySearch` 后**，事件分发路径改变（`engine.rs` L1015）：

```rust
fn handle_event(&mut self, prompt: &dyn Prompt, event: ReedlineEvent) -> Result<EventStatus> {
    if self.input_mode == InputMode::HistorySearch {
        self.handle_history_search_event(event)   // ← 走这个分支
    } else {
        self.handle_editor_event(prompt, event)
    }
}
```

在 `handle_history_search_event`（`engine.rs` L1023）中：

```rust
ReedlineEvent::PreviousHistory | ReedlineEvent::Up | ReedlineEvent::SearchHistory => {
    self.history_cursor.back(self.history.as_ref())
}
ReedlineEvent::NextHistory | ReedlineEvent::Down => {
    self.history_cursor.forward(self.history.as_ref())
}
Esc => { self.input_mode = InputMode::Regular; }  // Esc 退出搜索模式
```

**在搜索模式下**，后续的 Up/Down/Ctrl+R（`SearchHistory`）都通过 `HistoryCursor` 导航，而 `HistoryCursor` 持有 `session=Some(当前会话ID)`，所以会受到会话隔离条件的约束。

---

### 5.4 两种快捷键的全面对比

| 维度 | Ctrl+R（历史菜单） | Ctrl+Q（反向搜索模式） |
|------|-------------------|------------------------|
| ReedlineEvent | `Menu("history_menu")` | `SearchHistory` |
| 事件处理 | `handle_editor_event` | `handle_editor_event` 中 `enter_history_search()` 切换模式 |
| 后续输入模式 | 不切换，仍在 `Regular` | 切换到 `InputMode::HistorySearch` |
| 查询构造 | `SearchQuery::all_that_contain_rev(contains)` | `HistoryCursor::new(SubstringSearch, self.get_history_session_id())` |
| **session 参数** | **`None`** | **`Some(当前会话ID)`** |
| **受 isolation 影响？** | ❌ **否**，始终看全量 | ✅ **是**，只看自己 + 旧历史 |
| 去重方式 | `HashSet` 按 command_line | `HistoryCursor.skip_dupes` |
| 匹配方式 | 子串匹配（instr ≥ 1） | 子串匹配（instr ≥ 1） |
| 退出方式 | 选中 / Esc | Esc / Enter 接受当前 |
| 显示形式 | 弹出式菜单列表 | 行内搜索 `(reverse-i-search)` 提示符 |

---

## 六、会话编号（session_id）对各功能的影响（校准后）

### 6.1 session_id 的设置与传递

1. **生成**：`Reedline::create_history_session_id()` → 基于启动时纳秒时间戳的 `i64`
2. **存入引擎**：`with_history_session_id(session)` → `self.history_session_id = session`
3. **保存时设置**：`submit_buffer()` 中 `entry.session_id = self.get_history_session_id()`（引擎层，在空格前缀判断之前）
4. **导航时使用**：
   - 正常导航：`HistoryCursor::new(query, self.get_history_session_id())`
   - Ctrl+Q 搜索模式：同上
   - Ctrl+R 菜单：`all_that_contain_rev()` 用 `session=None`

---

### 6.2 各功能受隔离影响汇总（完整）

| 功能 | 触发方式 | 受 isolation 影响？ | session 参数 |
|------|----------|---------------------|--------------|
| 上/下箭头（逐条） | Up / Down 或 PreviousHistory / NextHistory | ✅ 是 | `HistoryCursor::new(Normal, session)` |
| fish 风格前缀搜索 | buffer 非空 + 光标在末尾 + 按上箭头 | ✅ 是 | `HistoryCursor::new(PrefixSearch, session)` |
| **Ctrl+Q** 反向搜索模式 | `Ctrl+Q` → 进入 HistorySearch | ✅ 是 | `HistoryCursor::new(SubstringSearch, session)` |
| Hinter 提示（CwdAwareHinter） | 自动在右侧显示灰色 | ✅ 是 | `last_with_prefix_and_cwd(..., history.session())` / `last_with_prefix(..., history.session())` |
| **Ctrl+R** 历史菜单 | `Ctrl+R` → 弹出列表菜单 | ❌ 否 | `all_that_contain_rev()` → `session=None` |
| `history` 命令（Plaintext） | 命令行执行 `history` | ❌ 否 | `everything(Forward, None)` → `session=None` |
| `history` 命令（SQLite） | 命令行执行 `history` | ❌ 否 | `SQLiteQueryBuilder` 直接查表，**绕过** History trait |

> **结论**：只有「**实时交互导航类**」功能（上箭头逐条、fish 前缀、Ctrl+Q 搜索模式、hinter 提示）受会话隔离影响。`Ctrl+R` 历史菜单和 `history` 命令始终能看到完整历史。

---

## 七、搜索查询与会话过滤机制

### 7.1 SearchFilter::anything(session) 的 SQL 展开

在 SQLite 后端的 `construct_query` 中，当 `session` 和 `session_timestamp` 都存在时，会附加 WHERE 条件：

```sql
(session_id = :session_id OR start_timestamp < :session_timestamp)
```

参数来源：
- `session_id`：`query.filter.session`（来自 `HistoryCursor` 或 `history.session()`）
- `session_timestamp`：`self.session_timestamp`（创建 `SqliteBackedHistory` 时传入的 `Some(now)`，即当前 Nu 会话的启动时间）

**完整语义**：
- `session_id = :session_id` — 当前会话**自己写入**的条目
- `start_timestamp < :session_timestamp` — **在本会话启动之前**就存在的条目（来自其他会话或更早的历史会话）

**软隔离模型**：旧历史大家共享，本会话启动后其他新会话写入的条目对我不可见，反之亦然。

---

### 7.2 Plaintext 的 search 能力

Plaintext 后端只支持部分过滤条件，不支持的会返回 `HistoryFeatureUnsupported` 错误：

| 过滤维度 | Plaintext 支持？ |
|----------|-----------------|
| 命令行（前缀/子串/精确） | ✅ |
| ID 范围（start_id、end_id） | ✅ |
| `not_command_line`（导航去重用） | ✅ |
| 时间范围 | ❌ 返回 Unsupported |
| hostname | ❌ |
| cwd（精确 / 前缀） | ❌ |
| exit_status 成功 | ❌ |
| session 过滤 | ❌ 根本不识别，等同于无过滤（但 Plaintext 也没有 session 字段） |

---

## 八、同步与跨会话共享

### 8.1 Plaintext：sync() 是跨会话共享的唯一通道

`FileBackedHistory::sync()`（`file_backed.rs`）：

1. **加写锁**（`fd_lock::RwLock`）→ 防止多个 Nu 进程并发写乱文件
2. **读磁盘全部内容** → `foreign_entries`（包含其他会话在本会话运行期间新增的条目）
3. **合并** → `foreign_entries + own_entries`（本会话新增但尚未刷盘的，即 `entries.range(len_on_disk..)`）
4. **容量检查** → `foreign_entries + own_entries > capacity` 时截断最旧条目
5. **写回磁盘** → 无需截断就追加，需要截断就从头覆写
6. **更新内存** → `self.entries = foreign_entries`（现在内存包含磁盘上的全部历史）
7. **更新游标** → `self.len_on_disk = self.entries.len()`

`sync()` 调用时机：在 Nushell 每轮 REPL 循环开始时（读取下一条用户输入之前），如果 `sync_on_enter=true`（默认）。参见 [repl.rs L692-L705](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/repl.rs#L692-L705)。

---

### 8.2 SQLite：sync() 是空操作

```rust
fn sync(&mut self) -> std::io::Result<()> {
    Ok(())
}
```

原因：
1. SQLite 使用 **WAL**（Write-Ahead Logging）模式，`INSERT` 已立即持久化到磁盘
2. 多进程通过 **SQLite 内置文件锁** 并发访问同一 DB 文件，不需要应用层加锁
3. 每次 `search()` 都直接读取最新的 DB 内容，天然包含其他会话刚写入的条目，无需显式同步

**跨会话共享是实时的**，不需要等 sync。

---

## 九、`history import` 导入机制

[history_import.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-cli/src/commands/history/history_import.rs)

**自动格式互转**（无管道输入时）：
```
当前 sqlite    → 从 history.txt  读取导入到 history.sqlite3
当前 plaintext → 从 history.sqlite3 读取导入到 history.txt
```

核心导入逻辑：
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

`item.id = None` 保证：
- Plaintext：新元素压入 VecDeque 尾部
- SQLite：触发 INSERT（而非 UPDATE），生成新的自增 id

---

## 十、启动初始化完整流程

```
nu 启动
  │
  ├─► 读取 $env.config.history（env.nu / config.nu）
  │       ├─► file_format: Plaintext(默认) | Sqlite
  │       ├─► path: Default | Custom | Disabled
  │       ├─► isolation: 仅 Sqlite 有效
  │       └─► 编译特性检查（无 sqlite 特性则 Sqlite→Plaintext 回退）
  │
  ├─► setup_history()
  │       │
  │       ├─► isolation=true → create_history_session_id()
  │       ├─► 计算文件路径（file_path()）
  │       └─► update_line_editor_history()
  │               │
  │               ├─► 创建后端（FileBackedHistory / SqliteBackedHistory）
  │               ├─► with_history_session_id(history_session_id)
  │               ├─► with_history_exclusion_prefix(ignore_space_prefixed.then_some(" "))
  │               ├─► 配置 hinter（默认 CwdAwareHinter，或自定义 ExternalHinter）
  │               └─► store_history_id_in_engine()
  │
  └─► engine_state.history_locked_after_startup = true
          （锁定 path/max_size/file_format/isolation 不再变更）
```

---

## 十一、Text vs SQLite 全面对比（校准后）

| 维度 | Plaintext (FileBackedHistory) | SQLite (SqliteBackedHistory) |
|------|-------------------------------|-------------------------------|
| **存储内容** | 仅 `command_line`（字符串） | 9 字段完整元数据 |
| **空命令过滤** | ✅ `!entry.is_empty()`（save 内） | ❌ 无（依赖层级 0） |
| **容量限制** | ✅ `capacity` 参数，VecDeque 滚动 | ❌ `max_size` 未传入，不生效 |
| **save 内去重** | ✅ 连续相同不存 | ❌ 无（DB 中会有重复） |
| **update 支持** | ❌ 不支持 | ✅ UPSERT 全字段覆盖 |
| **sync 作用** | 加锁读→合并→写回→更新内存 | no-op（WAL 实时持久化） |
| **跨会话共享** | sync 时通过文件锁合并 | search 时直接读 DB，天然实时 |
| **会话隔离** | ❌ 完全不支持 | ✅ 软隔离（旧共享+新隔离） |
| **过滤能力** | 仅命令行匹配 | 时间/CWD/hostname/exit/session 全支持 |
| **文件锁** | `fd_lock::RwLock` 应用层 | SQLite 内置 |
| **session_id 设置位置** | 引擎层 submit_buffer | 引擎层 submit_buffer |
| **上箭头导航可见性** | 全量（session 被忽略） | 受 isolation 约束 |
| **Ctrl+Q 搜索可见性** | 全量（session 被忽略） | 受 isolation 约束 |
| **Ctrl+R 菜单可见性** | 全量 | 全量（菜单用 `all_that_contain_rev`，session=None） |
| **history 命令可见性** | 全量 | 全量（直接查表） |
| **多行命令编码** | `\n` → `<\n>` 转义 | 原生 TEXT 支持 |

---

## 十二、文件路径解析

[HistoryConfig::file_path()](file:///d:/fz/0601-2/solo-dogfeeding/code/70-nushell/crates/nu-protocol/src/config/history.rs#L98-L114) 的解析规则：

1. `HistoryPath::Disabled` → 返回 `None`（不持久化，内存历史，退出即丢）
2. `HistoryPath::Custom(path)` → 使用用户指定路径
   - 若该路径是目录，则追加默认文件名（`history.txt` 或 `history.sqlite3`）
3. `HistoryPath::Default` → 使用 `nu_config_dir()/默认文件名`

默认文件名由 `HistoryFileFormat::default_file_name()` 返回：
- Plaintext → `history.txt`
- Sqlite → `history.sqlite3`
