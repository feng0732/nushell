# Reedline 行编辑接入方式

本文从代码实现角度梳理 Nushell 中 reedline 行编辑器的初始化、键位模式和输入循环三者之间的关系。

## 概述

Reedline 是 Nushell 的行编辑引擎，负责处理用户输入、键位绑定、历史记录、补全菜单等功能。在 Nushell 中，reedline 主要在两个场景下使用：

1. **REPL 主循环**：完整的交互式 Shell 体验
2. **input 命令**：独立的行编辑输入（可启用 reedline 模式）

两者共享相同的 reedline 库，但初始化路径和配置复杂度有很大差异。

---

## 一、初始化流程

### 1.1 REPL 初始化

#### 入口函数

REPL 的入口是 `evaluate_repl` 函数，位于 `crates/nu-cli/src/repl.rs`。

```rust
pub fn evaluate_repl(
    engine_state: &mut EngineState,
    stack: Stack,
    prerun_command: Option<Spanned<String>>,
    load_std_lib: Option<Spanned<String>>,
    entire_start_time: Instant,
) -> Result<()>
```

#### Reedline 实例创建

在 `evaluate_repl` 中，通过 `get_line_editor` 函数创建初始的 Reedline 实例：

```rust
fn get_line_editor(engine_state: &mut EngineState, use_color: bool) -> Result<Reedline> {
    let mut start_time = Instant::now();
    let mut line_editor = Reedline::create();

    // 存储历史会话 ID
    store_history_id_in_engine(engine_state, &line_editor);

    // 设置历史记录
    if let Some(history) = engine_state.history_config() {
        line_editor = setup_history(engine_state, line_editor, history)?;
        engine_state.history_locked_after_startup = true;
    }
    Ok(line_editor)
}
```

**关键点**：
- 使用 `Reedline::create()` 创建基础实例
- 初始化阶段仅配置历史记录后端
- 其他配置（高亮、补全、键位等）在每次循环迭代中动态设置

#### 历史记录设置

`setup_history` 函数（`crates/nu-cli/src/repl.rs`）配置历史记录后端，支持两种格式：

- **Plaintext**：基于文件的文本历史
- **Sqlite**：SQLite 数据库历史（需启用 `sqlite` feature）

### 1.2 input 命令初始化

`input` 命令可以独立启用 reedline 模式，提供轻量级的行编辑功能。

#### 启用条件

位于 `crates/nu-command/src/platform/input/input_.rs`：

```rust
let use_reedline = [
    call.has_flag(engine_state, stack, "reedline")?,
    call.get_flag::<String>(engine_state, stack, "history-file")?.is_some(),
    call.get_flag::<i64>(engine_state, stack, "max-history")?.is_some(),
]
.iter()
.any(|x| *x);
```

满足以下任一条件时启用 reedline：
- 显式指定 `--reedline` 标志
- 指定了 `--history-file`（隐含启用 reedline）
- 指定了 `--max-history`（隐含启用 reedline）

#### 初始化流程

```rust
// 1. 创建历史记录（可选）
let history = match (history_entries.is_some(), history_file_val.is_some()) {
    (false, false) => None,
    _ => {
        let file_history = match history_file_val {
            Some(file) => FileBackedHistory::with_file(max_history, file.into()),
            None => FileBackedHistory::new(max_history),
        };
        // ... 处理历史条目预填
        Some(history)
    }
};

// 2. 创建提示符
let prompt = ReedlinePrompt {
    indicator: default_str,
    left_prompt: prompt_str.unwrap_or("".to_string()),
    right_prompt: "".to_string(),
};

// 3. 创建并配置 Reedline
let mut line_editor = Reedline::create();
line_editor = line_editor.with_ansi_colors(false);
line_editor = match history {
    Some(h) => line_editor.with_history(Box::new(h)),
    None => line_editor,
};

// 4. 预填默认值
if let Some(val) = default_val.as_ref() {
    prefill_reedline_buffer(&mut line_editor, val);
}
```

**与 REPL 的区别**：
- 不配置语法高亮、补全、菜单、验证器等复杂功能
- 使用简化的 `ReedlinePrompt` 而非 `NushellPrompt`
- 禁用 ANSI 颜色
- 键位使用 reedline 默认设置，不加载 Nushell 自定义键位

---

## 二、键位模式

### 2.1 两种编辑模式

Reedline 支持两种键位模式，通过 `EditBindings` 枚举配置（`crates/nu-protocol/src/config/reedline.rs`）：

```rust
pub enum EditBindings {
    Vi,
    #[default]
    Emacs,
}
```

### 2.2 KeybindingsMode 枚举

在 `crates/nu-cli/src/reedline_config.rs` 中定义了 `KeybindingsMode` 枚举，表示解析后的键位绑定集合：

```rust
pub enum KeybindingsMode {
    Emacs(Keybindings),
    Vi {
        insert_keybindings: Keybindings,
        normal_keybindings: Keybindings,
    },
}
```

### 2.3 键位绑定创建

`create_keybindings` 函数（`crates/nu-cli/src/reedline_config.rs`）根据配置创建键位绑定：

```rust
pub(crate) fn create_keybindings(config: &Config) -> Result<KeybindingsMode, ShellError> {
    let parsed_keybindings = &config.keybindings;

    let mut emacs_keybindings = default_emacs_keybindings();
    let mut insert_keybindings = default_vi_insert_keybindings();
    let mut normal_keybindings = default_vi_normal_keybindings();

    // 添加菜单相关键位
    match config.edit_mode {
        EditBindings::Emacs => {
            add_menu_keybindings(&mut emacs_keybindings);
        }
        EditBindings::Vi => {
            add_menu_keybindings(&mut insert_keybindings);
            add_menu_keybindings(&mut normal_keybindings);
        }
    }

    // 应用用户自定义键位
    for keybinding in parsed_keybindings {
        add_keybinding(...);
    }

    match config.edit_mode {
        EditBindings::Emacs => Ok(KeybindingsMode::Emacs(emacs_keybindings)),
        EditBindings::Vi => Ok(KeybindingsMode::Vi {
            insert_keybindings,
            normal_keybindings,
        }),
    }
}
```

**流程说明**：
1. 从 reedline 库获取默认键位绑定（emacs / vi_insert / vi_normal）
2. 添加 Nushell 自定义的菜单键位（Tab 补全、Ctrl+R 历史菜单等）
3. 应用用户配置文件中的自定义键位
4. 根据 `edit_mode` 返回对应模式的键位绑定

**注意**：input 命令的 reedline 模式不执行此流程，使用 reedline 默认键位。

### 2.4 键位模式应用（REPL 专用）

`setup_keybindings` 函数（`crates/nu-cli/src/repl.rs`）将键位绑定应用到 Reedline 实例：

```rust
fn setup_keybindings(engine_state: &EngineState, line_editor: Reedline) -> Reedline {
    match create_keybindings(engine_state.get_config()) {
        Ok(keybindings) => match keybindings {
            KeybindingsMode::Emacs(keybindings) => {
                let edit_mode = Box::new(Emacs::new(keybindings));
                line_editor.with_edit_mode(edit_mode)
            }
            KeybindingsMode::Vi {
                insert_keybindings,
                normal_keybindings,
            } => {
                let edit_mode = Box::new(Vi::new(insert_keybindings, normal_keybindings));
                line_editor.with_edit_mode(edit_mode)
            }
        },
        Err(e) => {
            report_shell_error(None, engine_state, &e);
            line_editor
        }
    }
}
```

**关键点**：
- 使用 `with_edit_mode` 方法设置编辑模式
- Emacs 模式只需一组键位
- Vi 模式需要两组键位（insert 和 normal）
- Vi 模式下，reedline 内部管理模式切换（如 Esc 进入 normal 模式，i 进入 insert 模式）

### 2.5 模式切换机制

Vi 模式的切换由 reedline 库内部处理，通过 `ViChangeMode` 事件触发：

```rust
ReedlineEvent::ViChangeMode(mode.as_str()?.to_owned())
```

用户可以在配置中自定义触发模式切换的键位。

---

## 三、菜单加载与键位触发的配合

### 3.1 菜单加载

REPL 中通过 `add_menus` 函数（`crates/nu-cli/src/reedline_config.rs`）在每次循环迭代时重新加载所有菜单：

```rust
pub(crate) fn add_menus(
    mut line_editor: Reedline,
    engine_state_ref: Arc<EngineState>,
    stack: &Stack,
    config: Arc<Config>,
) -> Result<Reedline, ShellError> {
    // 先清空已有菜单
    line_editor = line_editor.clear_menus();

    // 加载用户配置的菜单
    for menu in &config.menus {
        line_editor = add_menu(line_editor, menu, ...)?
    }

    // 检查并加载默认菜单（如果用户未覆盖）
    let default_menus = [
        ("completion_menu", DEFAULT_COMPLETION_MENU),
        ("ide_completion_menu", DEFAULT_IDE_COMPLETION_MENU),
        ("history_menu", DEFAULT_HISTORY_MENU),
        ("help_menu", DEFAULT_HELP_MENU),
    ];
    // ... 解析并添加默认菜单
}
```

**默认菜单**：
- `completion_menu`：常规补全菜单（Tab 触发）
- `ide_completion_menu`：IDE 风格补全菜单（Ctrl+Space 触发）
- `history_menu`：历史搜索菜单（Ctrl+R 触发）
- `help_menu`：帮助菜单（F1 触发）

### 3.2 菜单键位绑定

`add_menu_keybindings` 函数（`crates/nu-cli/src/reedline_config.rs`）为菜单添加默认键位绑定：

```rust
fn add_menu_keybindings(keybindings: &mut Keybindings) {
    // Tab：补全菜单（优先菜单，其次下一项，最后默认补全）
    keybindings.add_binding(
        KeyModifiers::NONE,
        KeyCode::Tab,
        ReedlineEvent::UntilFound(vec![
            ReedlineEvent::Menu("completion_menu".to_string()),
            ReedlineEvent::MenuNext,
            ReedlineEvent::Edit(vec![EditCommand::Complete]),
        ]),
    );

    // Ctrl+Space：IDE 补全菜单
    keybindings.add_binding(
        KeyModifiers::CONTROL,
        KeyCode::Char(' '),
        ReedlineEvent::UntilFound(vec![
            ReedlineEvent::Menu("ide_completion_menu".to_string()),
            ReedlineEvent::MenuNext,
            ReedlineEvent::Edit(vec![EditCommand::Complete]),
        ]),
    );

    // Shift+Tab：上一个菜单项
    keybindings.add_binding(
        KeyModifiers::SHIFT,
        KeyCode::BackTab,
        ReedlineEvent::MenuPrevious,
    );

    // Ctrl+R：历史菜单
    keybindings.add_binding(
        KeyModifiers::CONTROL,
        KeyCode::Char('r'),
        ReedlineEvent::Menu("history_menu".to_string()),
    );

    // F1：帮助菜单
    keybindings.add_binding(
        KeyModifiers::NONE,
        KeyCode::F(1),
        ReedlineEvent::Menu("help_menu".to_string()),
    );
    // ...
}
```

### 3.3 配合机制

菜单加载和键位触发通过**菜单名称**关联，形成完整的交互链路：

```
┌─────────────────┐     名称匹配      ┌─────────────────┐
│   菜单键位绑定   │ ───────────────▶ │   菜单实例       │
│ (keybindings)   │                   │ (add_menus)      │
└─────────────────┘                   └─────────────────┘
         ▲                                      │
         │                                      ▼
┌─────────────────┐                   ┌─────────────────┐
│  用户按键触发    │ ───────────────▶ │  菜单显示/交互   │
│  (reedline 内部) │                   │  (reedline 内部) │
└─────────────────┘                   └─────────────────┘
```

**关键点**：
1. **名称一致**：键位绑定中的菜单名（如 `"completion_menu"`）必须与 `add_menus` 中注册的菜单名完全一致
2. **UntilFound 机制**：Tab 键使用 `UntilFound` 事件链，依次尝试打开菜单、切换到下一项、执行默认补全，保证降级可用
3. **每次迭代重建**：菜单和键位都在每次循环迭代时重新构建，确保配置变更立即生效
4. **用户可覆盖**：用户可以在配置文件中定义同名菜单来覆盖默认菜单，也可以添加新的键位绑定

---

## 四、输入循环

### 4.1 REPL 主循环

#### 主循环结构

`evaluate_repl`（`crates/nu-cli/src/repl.rs`）中的主循环：

```rust
loop {
    let mut current_engine_state = previous_engine_state.clone();
    let current_stack = Stack::with_parent(previous_stack_arc.clone());

    let iteration_panic_state = catch_unwind(AssertUnwindSafe(|| {
        let (continue_loop, current_stack, line_editor) = loop_iteration(LoopContext {
            engine_state: &mut current_engine_state,
            stack: current_stack,
            line_editor,
            nu_prompt: &mut nu_prompt_cloned,
            temp_file: &temp_file_cloned,
            use_color,
            entry_num: &mut entry_num,
            hostname: hostname.as_deref(),
            is_hostcommand: &mut is_hostcommand,
        });
        (continue_loop, current_engine_state, current_stack, line_editor)
    }));

    // 处理迭代结果，更新状态...
}
```

#### 单次循环迭代

`loop_iteration` 函数（`crates/nu-cli/src/repl.rs`）处理单次 REPL 迭代，这是 reedline 配置最集中的地方。

**主要步骤**：

1. **环境与钩子处理**
   - 合并环境变量
   - 重置信号
   - 触发 env_change 和 pre_prompt 钩子

2. **Reedline 配置构建**（使用 builder 模式）
   ```rust
   let mut line_editor = line_editor
       .use_kitty_keyboard_enhancement(config.use_kitty_protocol)
       .use_bracketed_paste(...)
       .with_highlighter(...)       // 语法高亮
       .with_validator(...)         // 输入验证
       .with_completer(...)         // 补全
       .with_quick_completions(...)
       .with_partial_completions(...)
       .with_ansi_colors(...)
       .with_cwd(...)
       .with_cursor_config(...)     // 光标形状
       .with_abbreviations(...)     // 缩写
       .with_visual_selection_style(...)
       .with_semantic_markers(...)  // shell integration
       .with_mouse_click(...);
   ```

3. **配置 hinter（提示）**
   ```rust
   line_editor = if config.use_ansi_coloring.get(engine_state) && config.show_hints {
       line_editor.with_hinter(...)  // CwdAwareHinter 或自定义 hinter
   } else {
       line_editor.disable_hints()
   };
   ```

4. **添加菜单**
   ```rust
   line_editor = add_menus(line_editor, engine_reference, &stack_arc, config)?;
   ```

5. **设置键位绑定**
   ```rust
   line_editor = setup_keybindings(engine_state, line_editor);
   ```

6. **更新提示符**
   ```rust
   prompt_update::update_prompt(config, engine_state, &mut Stack::with_parent(stack_arc.clone()), nu_prompt);
   ```

7. **预填缓冲区**（从 engine_state 恢复，HostCommand 后跳过）
   ```rust
   if !*is_hostcommand {
       line_editor = flush_engine_state_repl_buffer(engine_state, line_editor);
   }
   ```

8. **读取输入**
   ```rust
   let input = line_editor.read_line(nu_prompt);
   ```

9. **清理并处理输入**
   - 清除 stack 引用（避免悬垂引用）
   - 根据输入信号执行相应操作

#### 读取输入：read_line

`line_editor.read_line(nu_prompt)` 是阻塞调用，直到用户完成输入。返回类型为 `Result<Signal>`。

**Signal 类型**：
- `Signal::Success(command)`：用户按下 Enter，成功提交命令
- `Signal::HostCommand(command)`：宿主命令（特殊执行模式）
- `Signal::CtrlC`：用户按下 Ctrl+C，取消当前输入
- `Signal::CtrlD`：用户按下 Ctrl+D，退出 REPL

#### HostCommand 与缓冲区处理

`Signal::HostCommand` 是一种特殊的输入信号，由用户按下绑定了 `ExecuteHostCommand` 事件的键触发。它通过 `is_hostcommand` 标志影响下一次循环迭代的 flush #2：

```rust
// 读取输入后
match input {
    Ok(Signal::HostCommand(command)) => {
        *is_hostcommand = true;  // 标记为 HostCommand
        line_editor = run_command(RunContext { ... });
    }
    // ...
}
```

在 run_command 内部，Success 和 HostCommand 共享相同的执行流程，包括末尾的 **flush #1**（将 commandline 修改同步回 line_editor 并清空 ReplState）。

在下一次循环迭代的 flush #2：

```rust
// read_line 前检查 is_hostcommand 标志
if !*is_hostcommand {
    line_editor = flush_engine_state_repl_buffer(engine_state, line_editor);
}
*is_hostcommand = false;  // 重置标志
```

**为什么 HostCommand 后要跳过 flush #2？**

根据代码注释（`crates/nu-cli/src/repl.rs`）：

> If we don't flush the engine state, then the pre_prompt and env_change hooks cannot modify the commandline. But if we always flush the engine state, then the modification to the commandline done in ExecuteHostCommand will be overridden.

结合实际代码时序，原因分析：

1. **flush #1 已清空 ReplState**：run_command 末尾的 flush #1 执行后，`ReplState.buffer=""`, `cursor_pos=0`, `accept=false`。如果 HostCommand 后继续执行 flush #2，会：
   - `EditCommand::Clear` → 清空 reedline 内部缓冲区
   - `EditCommand::InsertString("")` → 插入空字符串
   - `EditCommand::MoveToPosition { position: 0 }` → 光标移到开头

2. **flush #1 已同步 commandline 修改**：HostCommand 中通过 commandline 写入 ReplState 的内容，已经在 flush #1 中正确同步到了 reedline 编辑器。flush #2 会用空 ReplState 覆盖这些内容。

3. **reedline 缓冲区是有状态的**：reedline 实例跨循环迭代复用，其内部缓冲区在 read_line 返回后保留着用户正在编辑的内容。HostCommand 的语义是「在当前行上执行辅助操作」，执行后用户应继续在同一条输入行上编辑。

4. **pre_prompt 钩子的冲突风险**：pre_prompt / env_change 钩子可能修改 ReplState.buffer，这些修改在非 HostCommand 场景下通过 flush #2 生效。但在 HostCommand 场景下，这些修改可能与 reedline 中已有的用户编辑内容产生冲突。

因此，`is_hostcommand` 标志作为一种保护机制，确保 HostCommand 执行后 reedline 的缓冲区状态不被 ReplState 中的空值覆盖。

#### 命令执行

输入处理后，通过 `run_command` 函数（`crates/nu-cli/src/repl.rs`）执行命令：

```rust
fn run_command(ctx: RunContext) -> Reedline
```

执行流程：
1. 准备历史记录元数据
2. 将 command 写入 ReplState.buffer（供 pre_execution 钩子读取）
3. 触发 pre_execution 钩子（可调用 commandline 修改 ReplState）
4. 保存 line_editor 当前 buffer 和光标到 ReplState（供 commandline 读取真实编辑内容）
5. 解析并执行命令（parse_operation -> do_run_cmd）
6. 更新历史记录结果元数据
7. 运行 shell integration ANSI 序列
8. **flush #1**：将 ReplState 同步回 line_editor（commandline 修改 → 编辑器）

### 4.2 input 命令输入循环

input 命令的输入循环非常简单，只有单次 read_line 调用：

```rust
match line_editor.read_line(&prompt) {
    Ok(Signal::Success(buffer) | Signal::HostCommand(buffer)) => {
        buf.push_str(&buffer);
    }
    Ok(Signal::CtrlC) => {
        return Err(IoError::new(...).into());
    }
    Ok(Signal::CtrlD) => {
        return Ok(Value::nothing(call.head).into_pipeline_data());
    }
    Ok(_) => {}
    Err(event_error) => {
        return Err(from_io_error(event_error).into());
    }
}
```

**与 REPL 的区别**：
- 单次读取，没有循环
- 不保存/恢复 buffer 状态
- 没有复杂的钩子和状态管理
- Ctrl+C 返回错误，Ctrl+D 返回空值

---

## 五、历史记录预填路径

### 5.1 REPL 历史记录

REPL 的历史记录在 `get_line_editor` 阶段初始化后持续存在，跨循环迭代复用。历史条目自动从文件加载，新的输入自动追加。

关键文件：`crates/nu-cli/src/repl.rs` 中的 `setup_history` 和 `update_line_editor_history` 函数。

### 5.2 input 命令历史记录

input 命令有两种历史记录来源：

#### 方式一：管道输入列表作为历史

```rust
let history_entries = match input {
    PipelineData::Value(Value::List { vals, .. }, ..) => Some(vals),
    _ => None,
};

// 后续在 history 构建中预填
if let Some(vals) = history_entries {
    vals.iter().for_each(|val| {
        if let Value::String { val, .. } = val {
            let _ = history.save(HistoryItem::from_command_line(val.clone()));
        }
    });
}
```

示例：`[past, command, entries] | input --reedline`

#### 方式二：历史文件

```rust
let file_history = match history_file_val {
    Some(file) => FileBackedHistory::with_file(max_history, file.into()),
    None => FileBackedHistory::new(max_history),
};
```

示例：`input --reedline --history-file ./history.txt`

两种方式可以组合使用：管道输入的条目会追加到历史文件的条目之上。

---

## 六、默认值预填路径

### 6.1 REPL 缓冲区恢复

REPL 使用 `ReplState` 机制在循环迭代间传递缓冲区内容。`flush_engine_state_repl_buffer` 函数（`crates/nu-cli/src/repl.rs`）负责将 ReplState 的内容同步到 reedline 编辑器：

```rust
fn flush_engine_state_repl_buffer(
    engine_state: &mut EngineState,
    mut line_editor: Reedline,
) -> Reedline {
    let mut repl = engine_state.repl_state.lock().expect("repl state mutex");
    line_editor.run_edit_commands(&[
        EditCommand::Clear,
        EditCommand::InsertString(repl.buffer.to_string()),
        EditCommand::MoveToPosition {
            position: repl.cursor_pos,
            select: false,
        },
    ]);
    if repl.accept {
        line_editor = line_editor.with_immediately_accept(true)
    }
    repl.accept = false;
    repl.buffer = "".to_string();
    repl.cursor_pos = 0;
    line_editor
}
```

`ReplState` 结构体（`crates/nu-protocol/src/engine/engine_state.rs`）：

```rust
pub struct ReplState {
    pub buffer: String,
    pub cursor_pos: usize,
    pub accept: bool,  // 是否立即提交
}
```

#### 两次 flush 的不同职责

`flush_engine_state_repl_buffer` 在一次 REPL 循环中可能被调用**两次**，各有不同的作用：

**flush #1 — run_command 末尾（L505）**：
- 时机：命令执行完毕，返回 line_editor 之前
- 目的：将 commandline 命令（在 pre_execution 钩子或 HostCommand 中调用）对 ReplState 的修改同步回编辑器
- 执行条件：Success 和 HostCommand 信号都会触发 run_command，因此都会执行此 flush
- 效果：commandline edit 写入的 buffer/cursor_pos/accept 立即应用到 line_editor

**flush #2 — loop_iteration 开头 read_line 之前（L734-736）**：
- 时机：下一次循环迭代，更新完提示符之后，调用 read_line 之前
- 目的：将 pre_prompt / env_change 钩子对 ReplState 的修改同步到编辑器
- 执行条件：**仅当上一次信号不是 HostCommand 时执行**（`if !*is_hostcommand`）
- HostCommand 后跳过的原因：HostCommand 场景下 reedline 内部已在 flush #1 时同步了状态，且 reedline 还保留着上次 read_line 后的编辑缓冲区（用户可能正在编辑的行），再次 flush 会用 ReplState 中被清空的 buffer 覆盖掉

**用途**：
- pre_prompt / env_change 钩子可以修改 `repl.buffer` 来改变命令行内容（通过 flush #2 生效）
- pre_execution 钩子 / HostCommand 中通过 commandline 命令修改 ReplState（通过 flush #1 生效）
- 实现跨迭代的缓冲区状态传递

### 6.2 input 命令默认值预填

input 命令通过 `prefill_reedline_buffer` 函数（`crates/nu-command/src/platform/input/input_.rs`）预填默认值：

```rust
fn prefill_reedline_buffer(line_editor: &mut Reedline, default_val: &str) {
    if default_val.is_empty() {
        return;
    }

    // Start with a clean buffer. This also ensures idempotency if this function is ever called
    // more than once.
    line_editor.run_edit_commands(&[EditCommand::Clear]);
    line_editor.run_edit_commands(&[EditCommand::InsertString(default_val.to_string())]);
    // Keep cursor at end (insertion point is naturally advanced by InsertString).
}
```

调用时机：在 `read_line` 之前调用

```rust
if let Some(val) = default_val.as_ref() {
    prefill_reedline_buffer(&mut line_editor, val);
}
```

**用户体验**：
- 默认值作为可编辑的初始内容出现在输入行中
- 用户可以直接修改或按 Enter 使用默认值
- 如果用户清空所有内容并提交，返回原始默认值（回退逻辑）

```rust
match default_val {
    Some(val) if buf.is_empty() => Ok(Value::string(val, call.head).into_pipeline_data()),
    _ => Ok(Value::string(buf, call.head).into_pipeline_data()),
}
```

### 6.3 预填技术对比

| 特性 | REPL (flush_engine_state_repl_buffer) | input (prefill_reedline_buffer) |
|------|--------------------------------------|----------------------------------|
| 实现方式 | `run_edit_commands` 批量执行 | `run_edit_commands` 分两次执行 |
| 命令序列 | Clear + InsertString + MoveToPosition | Clear + InsertString |
| 光标位置 | 可指定位置 | 自动在末尾 |
| 额外功能 | 支持立即提交（accept） | 无 |
| 数据来源 | engine_state.repl_state | 命令参数 `--default` |
| 调用频率 | 每轮循环最多两次：run_command 末尾一次（Success/HostCommand）、下一轮 read_line 前一次（HostCommand 后跳过） | 仅在 input 命令执行时 |

---

## 七、commandline 命令族与 ReplState 联动

`commandline` 命令族是 Nushell 提供的一组核心命令，允许在钩子和 HostCommand 中程序化地操作 reedline 缓冲区。所有命令通过 `engine_state.repl_state`（`Arc<Mutex<ReplState>>`）间接读写 reedline 的状态。

### 7.1 ReplState：命令族与编辑器之间的桥梁

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                          ReplState (Arc<Mutex<...>>)                               │
│                                                                                    │
│   buffer: String        ← commandline / commandline edit 读写                     │
│   cursor_pos: usize     ← commandline get-cursor / set-cursor                      │
│   accept: bool          ← commandline edit --accept 写入                           │
│                                                                                    │
│   ┌──────────────┐   读取    ┌──────────────┐   写入    ┌───────────────┐          │
│   │  line_editor  │ ───────▶ │  ReplState   │ ◀─────── │ commandline   │          │
│   │  (reedline)   │          │              │          │ 命令族         │          │
│   └──────┬───────┘          └──────┬───────┘          └───────────────┘          │
│          │                         │                                              │
│          │ L388-391: read_line 前  │ flush #1 (run_command 末尾 L505)             │
│          │   保存 buffer 和光标    │  将 ReplState 写回 line_editor                 │
│          │                         │  (commandline 修改 → 编辑器)                   │
│          ▼                         ▼                                              │
│   ┌──────────────┐          ┌──────────────┐                                      │
│   │  read_line    │          │   钩子中       │                                      │
│   │  用户编辑     │          │  commandline   │                                      │
│   └──────────────┘          └──────────────┘                                      │
│                                                                                    │
│   flush #2 (loop_iteration 开头 L734-736)                                          │
│   将 ReplState 写回 line_editor (HostCommand 后跳过)                                │
│   (pre_prompt/env_change 钩子修改 → 编辑器)                                        │
└────────────────────────────────────────────────────────────────────────────────────┘
```

**关键约束**：commandline 命令族**不直接**操作 reedline 实例。它们只修改 `ReplState`，由 `flush_engine_state_repl_buffer` 将状态同步到编辑器。同步发生在两个时间点：

1. **run_command 末尾（flush #1）**：commandline 命令在 pre_execution 钩子或 HostCommand 中执行后，修改的 ReplState 会在本次循环的 run_command 返回前立即同步回 line_editor。这是**主要**的同步时机。
2. **下一次循环开头 read_line 前（flush #2）**：pre_prompt / env_change 钩子中 commandline 的修改，会在下一次循环迭代的 read_line 之前同步。

#### ReplState 与 line_editor 的双向同步

ReplState 和 line_editor 之间存在**双向**数据流动：

**line_editor → ReplState（命令执行前保存，L388-391）**：
```rust
let mut repl = engine_state.repl_state.lock().expect("repl state mutex");
repl.cursor_pos = line_editor.current_insertion_point();
repl.buffer = line_editor.current_buffer_contents().to_string();
drop(repl);
```
这发生在 run_command 中 pre_execution 钩子之后、实际执行命令之前。目的是确保 commandline 命令读取到的是当前 line_editor 中用户正在编辑的内容和光标位置。

**ReplState → line_editor（命令执行后同步，即 flush）**：
由 flush_engine_state_repl_buffer 实现，将 commandline 修改后的内容写回编辑器。

### 7.2 commandline — 读取缓冲区内容

`commandline`（`crates/nu-cli/src/commands/commandline/commandline_.rs`）读取当前缓冲区内容：

```rust
fn run(&self, engine_state: &EngineState, _stack: &mut Stack, call: &Call, _input: PipelineData) -> Result<PipelineData, ShellError> {
    let repl = engine_state.repl_state.lock().expect("repl state mutex");
    Ok(Value::string(repl.buffer.clone(), call.head).into_pipeline_data())
}
```

- 只读操作，获取 `repl.buffer` 的快照
- 返回字符串值

### 7.3 commandline edit — 修改缓冲区内容

`commandline edit`（`crates/nu-cli/src/commands/commandline/edit.rs`）提供三种修改模式和可选的立即提交：

```rust
fn run(&self, engine_state: &EngineState, stack: &mut Stack, call: &Call, _input: PipelineData) -> Result<PipelineData, ShellError> {
    let str: String = call.req(engine_state, stack, 0)?;
    let mut repl = engine_state.repl_state.lock().expect("repl state mutex");
    if call.has_flag(engine_state, stack, "append")? {
        repl.buffer.push_str(&str);
    } else if call.has_flag(engine_state, stack, "insert")? {
        let cursor_pos = repl.cursor_pos;
        repl.buffer.insert_str(cursor_pos, &str);
        repl.cursor_pos += str.len();
    } else {
        repl.buffer = str;
        repl.cursor_pos = repl.buffer.len();
    }
    repl.accept = call.has_flag(engine_state, stack, "accept")?;
    Ok(Value::nothing(call.head).into_pipeline_data())
}
```

**三种模式**：

| 模式 | 标志 | 行为 | 光标位置 |
|------|------|------|----------|
| 替换（默认） | `--replace` / 无 | `repl.buffer = str` | 移到末尾（`buffer.len()`） |
| 追加 | `--append` / `-a` | `repl.buffer.push_str(&str)` | 不变 |
| 插入 | `--insert` / `-i` | `repl.buffer.insert_str(cursor_pos, &str)` | 前进 `str.len()` |

**`--accept` / `-A` 标志**：设置 `repl.accept = true`，使得 flush 后 reedline 会立即提交缓冲区内容，等价于用户按下了 Enter。

### 7.4 commandline get-cursor — 读取光标位置

`commandline get-cursor`（`crates/nu-cli/src/commands/commandline/get_cursor.rs`）返回当前光标位置（以 Unicode grapheme 为单位）：

```rust
fn run(&self, engine_state: &EngineState, _stack: &mut Stack, call: &Call, _input: PipelineData) -> Result<PipelineData, ShellError> {
    let repl = engine_state.repl_state.lock().expect("repl state mutex");
    let char_pos = repl.buffer
        .grapheme_indices(true)
        .chain(std::iter::once((repl.buffer.len(), "")))
        .position(|(i, _c)| i == repl.cursor_pos)
        .expect("Cursor position isn't on a grapheme boundary");
    Ok(Value::int(char_pos as i64, call.head).into_pipeline_data())
}
```

**字节 vs grapheme 转换**：
- `ReplState.cursor_pos` 存储的是**字节偏移**（与 reedline 内部的 `MoveToPosition` 一致）
- `get-cursor` 返回的是**grapheme 索引**（用户视角的字符位置）
- 通过 `grapheme_indices` 迭代器将字节偏移映射为 grapheme 索引

### 7.5 commandline set-cursor — 设置光标位置

`commandline set-cursor`（`crates/nu-cli/src/commands/commandline/set_cursor.rs`）设置光标位置：

```rust
fn run(&self, engine_state: &EngineState, stack: &mut Stack, call: &Call, _input: PipelineData) -> Result<PipelineData, ShellError> {
    let mut repl = engine_state.repl_state.lock().expect("repl state mutex");
    if let Some(pos) = call.opt::<i64>(engine_state, stack, 0)? {
        repl.cursor_pos = if pos <= 0 {
            0usize
        } else {
            repl.buffer.grapheme_indices(true)
                .map(|(i, _c)| i)
                .nth(pos as usize)
                .unwrap_or(repl.buffer.len())
        };
    } else if call.has_flag(engine_state, stack, "end")? {
        repl.cursor_pos = repl.buffer.len();
    }
    Ok(Value::nothing(call.head).into_pipeline_data()))
}
```

**grapheme vs 字节转换**：
- 输入参数是 **grapheme 索引**
- 通过 `grapheme_indices` 映射为**字节偏移**存入 `repl.cursor_pos`
- 负数或超出范围则夹紧到 0 或末尾

### 7.6 commandline complete — 补全

`commandline complete`（`crates/nu-cli/src/commands/commandline/complete.rs`）使用当前缓冲区内容和光标位置进行补全：

```rust
let (buffer, cursor_pos): (Cow<_>, _) = match &input {
    PipelineData::Empty => {
        // 从 ReplState 读取缓冲区和光标位置
        let repl = engine_state.repl_state.lock().expect("repl state mutex");
        (Cow::from(repl.buffer.clone()), repl.cursor_pos)
    }
    PipelineData::Value(Value::String { val, .. }, _) => (val.as_str().into(), val.len()),
    _ => { ... }
};
// 使用 NuCompleter 获取补全建议
let completions = completer.fetch_completions_at(&buffer, cursor_pos);
```

**特殊处理**：`complete` 读取 `ReplState` 后立即释放锁，避免在补全过程中（可能执行其他代码）持锁导致死锁。

### 7.7 状态同步的完整时序

commandline 命令对 `ReplState` 的修改通过**两次 flush** 和**双向同步**机制回到编辑器。以下是一次完整循环的时序（以 Success 信号为例）：

```
用户按下 Enter (read_line 返回 Signal::Success(command))
        │
        ▼
  ┌─ run_command 开始 ─────────────────────────────────────────────┐
  │                                                                 │
  │  1. pre_execution 钩子前：将 command 写入 ReplState.buffer        │
  │     repl.buffer = command.clone()                               │
  │                                                                 │
  │  2. 执行 pre_execution 钩子（可调用 commandline 修改 ReplState）  │
  │                                                                 │
  │  3. 命令执行前：保存 line_editor 当前状态到 ReplState              │
  │     repl.cursor_pos = line_editor.current_insertion_point()    │
  │     repl.buffer = line_editor.current_buffer_contents()         │
  │     ── 目的：确保 commandline 读取到的是用户真实编辑内容            │
  │                                                                 │
  │  4. 实际执行命令（HostCommand 中 commandline 修改 ReplState）     │
  │                                                                 │
  │  5. ★ flush #1（run_command 末尾 L505）★                        │
  │     line_editor = flush_engine_state_repl_buffer(...)          │
  │     ── 执行：Clear + InsertString(buffer) + MoveToPosition(cursor_pos) │
  │     ── 如果 repl.accept=true，则 line_editor.with_immediately_accept(true) │
  │     ── 清空 ReplState：buffer="" cursor_pos=0 accept=false       │
  │     ── 效果：commandline 修改立即应用到 line_editor                │
  │                                                                 │
  │  返回 line_editor                                               │
  └─────────────────────────────────────────────────────────────────┘
        │
        ▼
  回到主循环，进入下一次 loop_iteration
        │
        ▼
  ┌─ loop_iteration (下一次) 开始 ─────────────────────────────────┐
  │  ... 环境合并、钩子处理、Reedline 配置构建 ...                   │
  │                                                                 │
  │  pre_prompt / env_change 钩子（可调用 commandline 修改 ReplState）│
  │                                                                 │
  │  6. 更新提示符                                                   │
  │                                                                 │
  │  7. ★ flush #2（L734-736）★                                     │
  │     if !*is_hostcommand {                                       │
  │         line_editor = flush_engine_state_repl_buffer(...)      │
  │     }                                                           │
  │     *is_hostcommand = false                                     │
  │     ── Success 场景：执行 flush，将 pre_prompt 钩子的修改同步       │
  │     ── HostCommand 场景：跳过 flush，避免覆盖 reedline 内部状态     │
  │                                                                 │
  │  8. line_editor.read_line(nu_prompt)  ← 以新缓冲区等待输入       │
  └─────────────────────────────────────────────────────────────────┘
```

### 7.8 HostCommand 场景的同步细节

HostCommand 的执行路径与 Success 几乎相同，关键差异在 flush #2：

```
用户按下 HostCommand 绑定键 (read_line 返回 Signal::HostCommand(command))
        │
        ▼
  *is_hostcommand = true            ← 设置标志
        │
        ▼
  run_command 执行（步骤 1-5 与 Success 完全相同）
        │
        ├─ flush #1 在 run_command 末尾正常执行
        │   commandline 对 ReplState 的修改已同步到 line_editor
        │
        ▼
  下一次 loop_iteration 开头
        │
        ▼
  pre_prompt / env_change 钩子执行（可能修改 ReplState）
        │
        ▼
  flush #2：检查 is_hostcommand=true → 跳过 flush
        │
        ▼
  *is_hostcommand = false            ← 重置标志
        │
        ▼
  line_editor.read_line(nu_prompt)
  ── reedline 内部保留着上次 read_line 后的缓冲区状态
  ── 同时 flush #1 已将 commandline 修改应用到编辑器
  ── 用户继续在当前行编辑，内容不丢失
```

**为什么 HostCommand 后要跳过 flush #2？**

flush #1 执行完毕后，`flush_engine_state_repl_buffer` 会清空 ReplState（`buffer=""`, `cursor_pos=0`, `accept=false`）。如果 HostCommand 场景下继续执行 flush #2，会发生：

1. `EditCommand::Clear` → 清空 reedline 内部已通过 flush #1 设置好的内容
2. `EditCommand::InsertString("")` → 插入空字符串
3. `EditCommand::MoveToPosition { position: 0 }` → 光标移到第 0 位

结果是：HostCommand 中 commandline 修改后已在 flush #1 正确同步到 reedline 的内容，又被 flush #2 用空内容覆盖了。用户会看到输入行被意外清空。

**同时保留 reedline 内部状态**：HostCommand 的语义是「在当前行上执行辅助操作」，操作完成后用户应该继续在**同一条输入行**上编辑。reedline 实例跨迭代复用，其内部缓冲区保留着用户正在编辑的内容，跳过 flush #2 确保这些状态不被 ReplState 中的空值覆盖。

### 7.9 典型使用场景

**场景一：pre_prompt 钩子预填命令行**

```nu
$env.config = {
    hooks: {
        pre_prompt: [{
            condition: { $nu.is-login }
            code: { commandline edit --insert "echo 'Welcome!'" }
        }]
    }
}
```

流程：
1. 上一轮循环正常结束
2. 本轮 loop_iteration → pre_prompt 钩子执行
3. `commandline edit` 写入 ReplState
4. **flush #2**（L734-736，非 HostCommand）→ 将 ReplState 同步到 reedline
5. read_line → 用户看到预填内容

**场景二：HostCommand 中修改当前行（如 fzf 历史选择）**

```nu
# keybindings 配置中
{
    name: fzf_history
    modifier: control
    keycode: char_r
    mode: [emacs vi_normal vi_insert]
    event: {
        send: ExecuteHostCommand
        cmd: "commandline edit -r (history | get command | reverse | uniq | str join (char -i 0) | fzf --read0 --scheme=history -q (commandline))"
    }
}
```

流程：
1. 用户在某行输入时按 Ctrl+R → read_line 返回 HostCommand 信号
2. `*is_hostcommand = true`
3. run_command 执行：
   - 命令执行前保存 line_editor 状态到 ReplState（`commandline` 函数读取该行内容传给 fzf）
   - fzf 选择结果后，`commandline edit -r` 将选中项写入 ReplState
   - **flush #1**（run_command 末尾 L505）→ 将 fzf 选中项同步到 reedline 编辑器，清空 ReplState
4. 下一轮 loop_iteration：
   - pre_prompt / env_change 钩子执行
   - **flush #2 被跳过**（is_hostcommand=true）→ 不覆盖 reedline 中已有的 fzf 结果
   - read_line → 用户看到 fzf 选中的历史命令，可继续编辑

**场景三：accept 立即执行**

```nu
commandline edit --append " --help" --accept
```

流程：
1. `commandline edit` 写入 ReplState.buffer += " --help"，同时 `repl.accept = true`
2. **flush #1**（run_command 末尾）：
   - `EditCommand::Clear` → 清空
   - `EditCommand::InsertString(buffer)` → 插入新内容
   - `EditCommand::MoveToPosition` → 光标到末尾
   - `repl.accept == true` → `line_editor.with_immediately_accept(true)`
3. ReplState 被清空
4. 回到 read_line 后，reedline 检测到 immediately_accept 标志，自动提交缓冲区
5. 效果等价于用户自动按了 Enter

---

## 八、提示符与模式

### 8.1 NushellPrompt 结构

`NushellPrompt`（`crates/nu-cli/src/prompt.rs`）实现了 reedline 的 `Prompt` trait，支持多种模式的提示符指示符：

```rust
pub struct NushellPrompt {
    left_prompt: Option<String>,
    right_prompt: Option<String>,
    prompt_indicator: Option<String>,       // Emacs/Default 模式指示器
    vi_insert_prompt_indicator: Option<String>,  // Vi Insert 模式指示器
    vi_normal_prompt_indicator: Option<String>,  // Vi Normal 模式指示器
    multiline_indicator: Option<String>,    // 多行指示器
    render_right_prompt_on_last_line: bool,
}
```

### 8.2 提示符更新

提示符内容通过 `update_prompt` 函数（`crates/nu-cli/src/prompt_update.rs`）从环境变量中动态获取：

```rust
pub fn update_prompt(
    config: &Config,
    engine_state: &EngineState,
    stack: &mut Stack,
    nu_prompt: &mut NushellPrompt,
) {
    // 从环境变量读取提示符配置
    let left_prompt_string = get_prompt_string(PROMPT_COMMAND, config, engine_state, stack);
    let right_prompt_string = get_prompt_string(PROMPT_COMMAND_RIGHT, config, engine_state, stack);
    let prompt_indicator_string = get_prompt_string(PROMPT_INDICATOR, config, engine_state, stack);
    let prompt_multiline_string = get_prompt_string(PROMPT_MULTILINE_INDICATOR, config, engine_state, stack);
    let prompt_vi_insert_string = get_prompt_string(PROMPT_INDICATOR_VI_INSERT, config, engine_state, stack);
    let prompt_vi_normal_string = get_prompt_string(PROMPT_INDICATOR_VI_NORMAL, config, engine_state, stack);

    // 应用到 NushellPrompt
    nu_prompt.update_all_prompt_strings(...);
}
```

**环境变量列表**：
- `PROMPT_COMMAND`：左侧提示符（字符串或闭包）
- `PROMPT_COMMAND_RIGHT`：右侧提示符
- `PROMPT_INDICATOR`：Emacs/Default 模式指示器
- `PROMPT_INDICATOR_VI_INSERT`：Vi Insert 模式指示器
- `PROMPT_INDICATOR_VI_NORMAL`：Vi Normal 模式指示器
- `PROMPT_MULTILINE_INDICATOR`：多行输入指示器
- `TRANSIENT_*`：瞬态提示符版本（命令执行后显示）

### 8.3 模式指示器渲染

`render_prompt_indicator` 方法根据当前编辑模式返回对应的指示器字符串：

```rust
fn render_prompt_indicator(&self, edit_mode: PromptEditMode) -> Cow<'_, str> {
    let indicator: &str = match edit_mode {
        PromptEditMode::Default => self.prompt_indicator.as_deref().unwrap_or("> "),
        PromptEditMode::Emacs => self.prompt_indicator.as_deref().unwrap_or("> "),
        PromptEditMode::Vi(vi_mode) => match vi_mode {
            PromptViMode::Normal => self.vi_normal_prompt_indicator.as_deref().unwrap_or("> "),
            PromptViMode::Insert => self.vi_insert_prompt_indicator.as_deref().unwrap_or(": "),
        },
        PromptEditMode::Custom(str) => &self.default_wrapped_custom_string(str),
    };
    indicator.to_string().into()
}
```

**默认值**（用户未配置环境变量时使用）：
- Emacs/Default 模式：`> `
- Vi Normal 模式：`> `
- Vi Insert 模式：`: `

### 8.4 模式切换与视觉反馈

提示符模式指示器与编辑模式联动：
1. 用户切换编辑模式（如按 Esc 从 Insert 进入 Normal）
2. reedline 内部更新模式状态
3. 重绘提示符时，reedline 调用 `render_prompt_indicator` 并传入当前模式
4. `NushellPrompt` 根据模式返回对应的指示器字符串

这为用户提供了即时的视觉反馈，让用户知道当前处于哪种编辑模式。

---

## 九、三者关系图

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                            evaluate_repl (主循环)                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │                         loop_iteration (单次迭代)                              │ │
│  │                                                                              │ │
│  │  ┌─────────────┐    ┌──────────────────┐    ┌──────────────┐    ┌──────────┐ │ │
│  │  │  环境/钩子  │───▶│ Reedline 配置构建 │───▶│  flush #2    │───▶│ read_line│ │ │
│  │  │  预处理     │    │ (键位/菜单/高亮…) │    │ (HostCommand │    │  (阻塞)  │ │ │
│  │  └─────────────┘    └──────────────────┘    │  后跳过)      │    └────┬─────┘ │ │
│  │                                             └──────────────┘         │       │ │
│  │                                                  ▲                    │       │ │
│  │                                                  │                    ▼       │ │
│  │  ┌──────────────────────────────────────────┐  │              ┌─────────────┐ │ │
│  │  │         ReplState (Arc<Mutex<...>>)       │  │              │  信号分发   │ │ │
│  │  │  buffer / cursor_pos / accept             │──┘              └──────┬──────┘ │ │
│  │  │                                            │                        │        │ │
│  │  │  ▲           commandline 读写              │                        ▼        │ │
│  │  │  │                                         │              ┌─────────────┐   │ │
│  │  │  │  执行前保存(L388-391)                   │              │  run_command │   │ │
│  │  │  │                                         │              │             │   │ │
│  │  │  └─────────────────────────────────────────│──◀── flush #1│  (末尾 L505)│   │ │
│  │  │                                            │              └─────────────┘   │ │
│  │  └──────────────────────────────────────────┘                 │  命令执行     │ │
│  │                                                               └───────────────┘ │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                    │
│  历史记录: setup_history (REPL 启动时一次性初始化, 跨迭代复用)                      │
│  菜单&键位: 每次迭代重新构建 (add_menus + setup_keybindings)                         │
│  提示符:   每次迭代从环境变量更新 (update_prompt)                                    │
│  ReplState:  ← 编辑器(L388-391)  commandline 读写  → flush #1/#2 → 编辑器           │
└──────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                         input 命令 (单次)                            │
│  ┌───────────────┐    ┌───────────┐    ┌──────────┐               │
│  │ 创建 Reedline │───▶│ 预填默认值│───▶│ read_line│               │
│  │  + 历史配置    │    │           │    │  (阻塞)  │               │
│  └───────────────┘    └───────────┘    └────┬─────┘               │
│                                              │                     │
│                                         ┌────▼────┐               │
│                                         │ 返回结果 │               │
│                                         └─────────┘               │
│                                                                     │
│  历史记录来源:                                                       │
│    - 管道输入列表 (history_entries)                                 │
│    - --history-file 文件                                            │
│  默认值来源: --default 参数                                         │
│  无菜单、无自定义键位、使用 reedline 默认配置                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 十、关键设计要点

### 10.1 REPL：每次迭代重新配置

Reedline 的大部分配置（高亮、补全、菜单、键位、提示符等）在**每次循环迭代**时重新构建，而非一次性初始化。原因：

- 配置可能在运行时改变（如修改键位绑定）
- 补全和高亮需要最新的 engine_state 和 stack 状态
- 提示符内容需要动态更新

### 10.2 REPL：Stack 引用管理

由于 reedline 的插件（highlighter, completer, hinter 等）需要引用 stack，而 stack 在 REPL 循环中会被修改，因此采用了特殊的引用管理策略：

1. 配置阶段：创建 `stack_arc`（Arc<Stack>），供各插件使用
2. read_line 期间：插件持有 stack 的只读引用
3. read_line 返回后：清除插件中的 stack 引用，恢复所有权

```rust
// 配置阶段 - 传入 stack_arc
.with_highlighter(Box::new(NuHighlighter::new(engine_reference.clone(), stack_arc.clone())))
.with_completer(Box::new(NuCompleter::new(engine_reference.clone(), stack_arc.clone())))

// read_line 返回后 - 清除引用
line_editor = line_editor
    .with_highlighter(Box::<NoOpHighlighter>::default())
    .with_completer(Box::<DefaultCompleter>::default())
```

### 10.3 键位模式与光标形状

键位模式与光标形状配置联动，通过 `CursorConfig` 设置（`crates/nu-cli/src/repl.rs`）：

```rust
let cursor_config = CursorConfig {
    vi_insert: map_nucursorshape_to_cursorshape(config.cursor_shape.vi_insert),
    vi_normal: map_nucursorshape_to_cursorshape(config.cursor_shape.vi_normal),
    emacs: map_nucursorshape_to_cursorshape(config.cursor_shape.emacs),
};
```

不同模式下光标形状不同，提供视觉反馈。

### 10.4 菜单与键位的松耦合设计

菜单和键位通过名称字符串关联，实现松耦合：
- 菜单在 `add_menus` 中注册，键位在 `add_menu_keybindings` 中绑定
- 用户可以单独添加新键位来触发已有菜单
- 用户也可以覆盖同名菜单来改变菜单行为，而无需修改键位绑定
- Tab 键的 `UntilFound` 机制提供了优雅的降级策略

### 10.5 HostCommand 的缓冲区保护机制

通过 `is_hostcommand` 标志实现 HostCommand 后的缓冲区状态保留：
- HostCommand 与 Success 共享 run_command 流程，flush #1 在末尾正常执行（commandline 修改已同步到编辑器）
- flush #1 执行后 ReplState 被清空（`buffer=""`）
- HostCommand 执行后设置 `is_hostcommand` 标志
- 下一次迭代的 flush #2 被跳过，避免用空 ReplState 覆盖 reedline 中已有的内容
- 保留 reedline 内部的缓冲区内容和光标位置，确保用户继续在当前行上编辑

### 10.6 commandline 命令族的双向同步机制

commandline 命令族通过 ReplState 间接与 reedline 交互，存在**双向同步**：
- **编辑器 → ReplState**（run_command 中命令执行前）：保存 line_editor 当前 buffer 和光标，供 commandline 读取真实编辑内容
- **ReplState → 编辑器**（flush #1 / flush #2）：将 commandline 修改写回 reedline
- **延迟生效但不跨轮次**：在 pre_execution 钩子或 HostCommand 中执行的 commandline，修改在本轮 run_command 末尾的 flush #1 就同步回编辑器
- **线程安全**：通过 `Arc<Mutex<ReplState>>` 确保多线程安全访问
- **字节/字符转换**：内部存储字节偏移（与 reedline `MoveToPosition` 一致），对外暴露 grapheme 索引（用户视角）
- **accept 语义**：通过 `with_immediately_accept` 实现自动提交

### 10.7 input 命令：两种输入模式

- **Legacy 模式**：逐字符读取，不使用 reedline，功能简单
- **Reedline 模式**：完整的行编辑能力，支持历史、多行编辑等

设计意图：
- Legacy 模式保持向后兼容，适合简单的单字符输入场景
- Reedline 模式提供更丰富的交互体验，但需要 TTY 支持

---

## 十一、相关文件清单

| 文件 | 作用 |
|------|------|
| `crates/nu-cli/src/repl.rs` | REPL 主循环、输入处理、reedline 初始化与配置 |
| `crates/nu-cli/src/reedline_config.rs` | 键位绑定创建、菜单配置、菜单与键位配合 |
| `crates/nu-cli/src/prompt.rs` | Nushell 提示符实现（REPL 使用） |
| `crates/nu-cli/src/prompt_update.rs` | 提示符更新逻辑、环境变量读取 |
| `crates/nu-cli/src/commands/commandline/commandline_.rs` | commandline 命令 — 读取缓冲区 |
| `crates/nu-cli/src/commands/commandline/edit.rs` | commandline edit 命令 — 修改缓冲区/光标/accept |
| `crates/nu-cli/src/commands/commandline/get_cursor.rs` | commandline get-cursor 命令 — 读取光标位置 |
| `crates/nu-cli/src/commands/commandline/set_cursor.rs` | commandline set-cursor 命令 — 设置光标位置 |
| `crates/nu-cli/src/commands/commandline/complete.rs` | commandline complete 命令 — 补全 |
| `crates/nu-protocol/src/config/reedline.rs` | reedline 相关配置数据结构 |
| `crates/nu-protocol/src/engine/engine_state.rs` | ReplState 定义、engine_state 结构 |
| `crates/nu-command/src/platform/input/input_.rs` | input 命令实现（含 reedline 模式） |
| `crates/nu-command/src/platform/input/reedline_prompt.rs` | input 命令使用的简单提示符 |
