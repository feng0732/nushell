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

## 三、输入循环

### 3.1 REPL 主循环

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

7. **预填缓冲区**（从 engine_state 恢复）
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

#### 命令执行

输入处理后，通过 `run_command` 函数（`crates/nu-cli/src/repl.rs`）执行命令：

```rust
fn run_command(ctx: RunContext) -> Reedline
```

执行流程：
1. 准备历史记录元数据
2. 触发 pre_execution 钩子
3. 解析并执行命令（parse_operation -> do_run_cmd）
4. 更新历史记录结果元数据
5. 运行 shell integration ANSI 序列

### 3.2 input 命令输入循环

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

## 四、历史记录预填路径

### 4.1 REPL 历史记录

REPL 的历史记录在 `get_line_editor` 阶段初始化后持续存在，跨循环迭代复用。历史条目自动从文件加载，新的输入自动追加。

关键文件：`crates/nu-cli/src/repl.rs` 中的 `setup_history` 和 `update_line_editor_history` 函数。

### 4.2 input 命令历史记录

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

## 五、默认值预填路径

### 5.1 REPL 缓冲区恢复

REPL 使用 `ReplState` 机制在循环迭代间传递缓冲区内容，通过 `flush_engine_state_repl_buffer` 函数（`crates/nu-cli/src/repl.rs`）预填：

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

**用途**：
- pre_prompt / env_change 钩子可以修改 `repl.buffer` 来改变命令行内容
- `ExecuteHostCommand` 等特殊操作后保留缓冲区状态
- 实现跨迭代的缓冲区状态传递

### 5.2 input 命令默认值预填

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

### 5.3 预填技术对比

| 特性 | REPL (flush_engine_state_repl_buffer) | input (prefill_reedline_buffer) |
|------|--------------------------------------|----------------------------------|
| 实现方式 | `run_edit_commands` 批量执行 | `run_edit_commands` 分两次执行 |
| 命令序列 | Clear + InsertString + MoveToPosition | Clear + InsertString |
| 光标位置 | 可指定位置 | 自动在末尾 |
| 额外功能 | 支持立即提交（accept） | 无 |
| 数据来源 | engine_state.repl_state | 命令参数 `--default` |
| 调用频率 | 每次循环迭代 | 仅在 input 命令执行时 |

---

## 六、三者关系图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        evaluate_repl (主循环)                         │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                     loop_iteration (单次迭代)                   │  │
│  │                                                               │  │
│  │  ┌─────────────┐    ┌──────────────────┐    ┌──────────┐     │  │
│  │  │  环境/钩子  │───▶│ Reedline 配置构建 │───▶│ read_line│     │  │
│  │  │  预处理     │    │ (键位/高亮/补全…) │    │  (阻塞)  │     │  │
│  │  └─────────────┘    └──────────────────┘    └────┬─────┘     │  │
│  │                                                  │            │  │
│  │  ┌─────────────┐    ┌─────────────┐             │            │  │
│  │  │  命令执行   │◀───│  信号分发   │◀────────────┘            │  │
│  │  └─────────────┘    └─────────────┘                          │  │
│  │                                                               │  │
│  │  ┌───────────────────────────────────────────┐                │  │
│  │  │  缓冲区预填: flush_engine_state_repl_buffer│                │  │
│  │  │  来源: engine_state.repl_state            │                │  │
│  │  └───────────────────────────────────────────┘                │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  历史记录: setup_history (REPL 启动时一次性初始化, 跨迭代复用)       │
└─────────────────────────────────────────────────────────────────────┘

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
└─────────────────────────────────────────────────────────────────────┘
```

---

## 七、关键设计要点

### 7.1 REPL：每次迭代重新配置

Reedline 的大部分配置（高亮、补全、菜单、键位等）在**每次循环迭代**时重新构建，而非一次性初始化。原因：

- 配置可能在运行时改变（如修改键位绑定）
- 补全和高亮需要最新的 engine_state 和 stack 状态
- 提示符内容需要动态更新

### 7.2 REPL：Stack 引用管理

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

### 7.3 键位模式与光标形状

键位模式与光标形状配置联动，通过 `CursorConfig` 设置（`crates/nu-cli/src/repl.rs`）：

```rust
let cursor_config = CursorConfig {
    vi_insert: map_nucursorshape_to_cursorshape(config.cursor_shape.vi_insert),
    vi_normal: map_nucursorshape_to_cursorshape(config.cursor_shape.vi_normal),
    emacs: map_nucursorshape_to_cursorshape(config.cursor_shape.emacs),
};
```

不同模式下光标形状不同，提供视觉反馈。

### 7.4 提示符与模式

`NushellPrompt`（`crates/nu-cli/src/prompt.rs`）实现了 reedline 的 `Prompt` trait，根据编辑模式显示不同的提示符指示器：

```rust
fn render_prompt_indicator(&self, edit_mode: PromptEditMode) -> Cow<'_, str> {
    match edit_mode {
        PromptEditMode::Default | PromptEditMode::Emacs => "> ",
        PromptEditMode::Vi(vi_mode) => match vi_mode {
            PromptViMode::Normal => "> ",   // vi normal 模式
            PromptViMode::Insert => ": ",   // vi insert 模式
        },
        PromptEditMode::Custom(str) => format!("({str})"),
    }
}
```

### 7.5 input 命令：两种输入模式

input 命令支持两种输入模式，通过是否启用 reedline 区分：

- **Legacy 模式**：逐字符读取，不使用 reedline，功能简单
- **Reedline 模式**：完整的行编辑能力，支持历史、多行编辑等

设计意图：
- Legacy 模式保持向后兼容，适合简单的单字符输入场景
- Reedline 模式提供更丰富的交互体验，但需要 TTY 支持

---

## 八、相关文件清单

| 文件 | 作用 |
|------|------|
| `crates/nu-cli/src/repl.rs` | REPL 主循环、输入处理、reedline 初始化与配置 |
| `crates/nu-cli/src/reedline_config.rs` | 键位绑定创建、菜单配置 |
| `crates/nu-cli/src/prompt.rs` | Nushell 提示符实现（REPL 使用） |
| `crates/nu-protocol/src/config/reedline.rs` | reedline 相关配置数据结构 |
| `crates/nu-protocol/src/engine/engine_state.rs` | ReplState 定义、engine_state 结构 |
| `crates/nu-command/src/platform/input/input_.rs` | input 命令实现（含 reedline 模式） |
| `crates/nu-command/src/platform/input/reedline_prompt.rs` | input 命令使用的简单提示符 |
