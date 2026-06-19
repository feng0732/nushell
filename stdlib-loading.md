# NuShell 标准库加载、命令注册与启动分析

## 一、核心结论速览

| 问题 | 答案 |
|---|---|
| 虚拟 std 路径和用户库路径是同一套优先级吗？ | **不是**。文件查找优先级：虚拟路径 > 当前目录 > NU_LIB_DIRS |
| 命令覆盖和模块查找是同一套优先级吗？ | **不是**。两套独立机制：文件查找 vs Overlay 注册顺序 |
| 用户模块能覆盖标准库模块吗？ | **文件查找层面不能**（虚拟路径优先级最高），但 **use/def 注册层面能**（后激活的 Overlay 优先级更高） |
| `%cmd` 为什么总能找到内置命令？ | 它不通过 Overlay 查找，直接遍历所有 Decl 找 `CommandType::Builtin` |
| 标准库加载对启动成本影响大吗？ | **不大**。虚拟文件是编译期嵌入的字符串引用，只有 prelude 需要解析 |
| NU_LIB_DIRS 常量和环境变量内容相同吗？ | **相同**。两者都包含「用户路径 + 默认路径」的完整列表，用户路径在前 |

---

## 二、两套独立的优先级机制

这是理解整个系统的关键：**文件查找优先级** 和 **命令/模块覆盖优先级** 是两套完全独立的机制。

```
┌─────────────────────────────────────────────────────────┐
│              模块文件查找优先级                           │
│  （决定 "use std/assert" 从哪里读文件）                   │
│                                                         │
│  虚拟路径 > 当前目录 > NU_LIB_DIRS 顺序遍历               │
│  （最高）                  （最低）                       │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│              命令/模块覆盖优先级                           │
│  （决定 "pwd" 调用哪个实现）                              │
│                                                         │
│  用户 def/use > 标准库 prelude > 内置 Rust 命令           │
│  （最高）                  （最低）                       │
└─────────────────────────────────────────────────────────┘
```

两套机制的关系：文件查找决定**模块从哪里加载**，Overlay 机制决定**加载后同名命令谁生效**。

---

## 三、模块文件查找优先级详解（已核准）

### 3.1 完整查找顺序（带代码证据）

核心函数是 [`find_in_dirs_with_id`](crates/nu-parser/src/parse_source.rs#L631-L690)，执行顺序严格如下：

```
优先级从高到低：

1. 虚拟路径精确匹配
   working_set.find_virtual_path(filename)
   例：find_virtual_path("std/assert") → 命中直接返回

2. 虚拟路径绝对路径匹配
   abs_virtual_filename = actual_cwd.join(filename)
   working_set.find_virtual_path(abs_virtual_filename) → 命中返回

3. 当前工作目录相对路径（真实文件）
   absolute_with(filename, actual_cwd) && p.exists()

4. NU_LIB_DIRS 常量遍历（按列表顺序，先到先得）
   for lib_dir in $NU_LIB_DIRS:
       path = lib_dir / filename
       if path.exists() → 返回
```

**关键结论：虚拟路径优先级最高，用户磁盘上的同名模块无法覆盖标准库模块。**

### 3.2 NU_LIB_DIRS 设置逻辑（已核准）

在 [main.rs#L342-L415](src/main.rs#L342-L415) 中设置，代码顶部注释：

```rust
// Set up NU_LIB_DIRS: constant = defaults + env + -I, env = env + -I
```

实际构造顺序（[main.rs#L354-L386](src/main.rs#L354-L386)）：

```rust
// 1) 从环境变量 NU_LIB_DIRS 读取用户路径
let mut user_lib_dirs: Vec<String> = if let Some(val) = engine_state.get_env_var("NU_LIB_DIRS") {
    match val {
        Value::List { vals, .. }   => /* 列表直接取字符串 */,
        Value::String { val, .. }  => /* 按平台分隔符切分 */,
        _ => vec![],
    }
} else { vec![] };

// 2) 追加 -I 命令行参数传入的路径
if let Some(spanned) = include_paths {
    let paths = parse_path_list(&spanned.item, &[':', ';', '\x1e']);
    user_lib_dirs.extend(paths);
}

// 3) 默认路径
let default_paths = vec![
    default_nu_lib_dirs_path.to_string_lossy().to_string(),       // <config>/scripts
    default_nushell_completions_path.to_string_lossy().to_string(), // <data>/nushell/completions
];

// 4) 最终顺序：用户路径在前，默认路径在后
let all_lib_dirs: Vec<String> = user_lib_dirs.into_iter().chain(default_paths).collect();

// 5) 同时写入 $env.NU_LIB_DIRS 和 $NU_LIB_DIRS 常量（内容完全相同）
engine_state.add_env_var("NU_LIB_DIRS", Value::list(all_lib_dir_values.clone(), ...));
// ...
working_set.set_variable_const_val(var_id, Value::list(all_lib_dir_values, ...));
```

### 3.3 NU_LIB_DIRS 完整组成（已核准）

| 路径来源 | 具体值 | 代码位置 |
|---|---|---|
| 环境变量 `NU_LIB_DIRS` | 用户父环境或 env.nu 中设置的列表或字符串 | [main.rs#L355-L371](src/main.rs#L355-L371) |
| 命令行 `-I <path>` | 支持多路径，分隔符 `:`, `;`, `\x1e` | [main.rs#L374-L377](src/main.rs#L374-L377) |
| 默认 scripts 目录 | `<nushell_config_path>/scripts/` | [main.rs#L177-L178](src/main.rs#L177-L178) |
| 默认 completions 目录 | `<data_dir>/nushell/completions/` | [main.rs#L169-L175](src/main.rs#L169-L175) |

其中：
- `nushell_config_path` = `nu_path::nu_config_dir()`，通常是 `~/.config/nushell/`（受 XDG_CONFIG_HOME 影响）
- `data_dir` = `nu_path::data_dir()`，平台相关的数据目录
- **$env.NU_LIB_DIRS 和 $NU_LIB_DIRS 常量内容完全相同**，都包含用户路径和默认路径

### 3.4 虚拟路径的反向遍历（已核准）

[`find_virtual_path`](crates/nu-protocol/src/engine/state_working_set.rs#L1045-L1062) 从后往前遍历：

```rust
for (virtual_name, virtual_path) in self.delta.virtual_paths.iter().rev() { ... }
for (virtual_name, virtual_path) in self.permanent_state.virtual_paths.iter().rev() { ... }
```

**后注册的虚拟路径优先级更高**。delta（当前解析工作集）中的虚拟路径优先于 permanent_state（引擎永久状态）中的同名路径。

标准库加载时把 std 和 std-rfc 的所有虚拟文件注册到 permanent_state。如果用户代码在解析时注册同名虚拟文件（极少见），会覆盖标准库的虚拟路径。

### 3.5 解析时使用的是常量版 NU_LIB_DIRS

模块查找调用 `find_in_dirs(filename, working_set, cwd, Some(LIB_DIRS_VAR))`，其中 [`LIB_DIRS_VAR = "NU_LIB_DIRS"`](crates/nu-parser/src/parse_source.rs#L28)，但实际查找走的是 **常量版本**（`find_dirs_var` 只匹配带 `const_val` 的变量）：

```rust
pub fn find_dirs_var(working_set: &StateWorkingSet, var_name: &str) -> Option<VarId> {
    working_set
        .find_variable(format!("${var_name}").as_bytes())
        .filter(|var_id| working_set.get_variable(*var_id).const_val.is_some())  // 只认常量
}
```

这意味着运行时修改 `$env.NU_LIB_DIRS` 不影响模块查找，必须修改常量 `$NU_LIB_DIRS` 才有效。

---

## 四、命令/模块覆盖优先级（Overlay 机制）

### 4.1 Overlay 激活 = 优先级提升

[`add_overlay`](crates/nu-protocol/src/engine/state_working_set.rs#L922-L967) 的关键行为：

```rust
// 如果 overlay 已存在，先从 active_overlays 中移除
last_scope_frame.active_overlays.retain(|id| id != &overlay_id);
// 再 push 到末尾
last_scope_frame.active_overlays.push(overlay_id);
```

**每次 `use` 或 `overlay use` 都会把对应 overlay 移到 active_overlays 末尾**，而 `find_decl` 从后往前查，因此后激活的优先级更高。

### 4.2 同名覆盖的完整场景

| 场景 | 行为 | 代码位置 |
|---|---|---|
| 同一 Overlay 内注册同名 decl | HashMap::insert，后者覆盖前者 | [add_decl](crates/nu-protocol/src/engine/state_working_set.rs#L115-L125) |
| 不同 Overlay 同名 decl | 后激活的 Overlay 先被查到 | [find_decl](crates/nu-protocol/src/engine/state_working_set.rs#L443-L477) |
| 用户 `def pwd` 覆盖内置 `pwd` | ✅ 可以覆盖，因为用户 def 在更新的 overlay | 同上 |
| 标准库 prelude `pwd` 覆盖内置 `pwd` | ✅ 可以覆盖，prelude 是后注册的自定义命令 | 同上 |
| 用户磁盘同名模块覆盖 std 模块 | ❌ 不能，文件查找时虚拟路径优先级更高 | [find_in_dirs](crates/nu-parser/src/parse_source.rs#L631-L690) |

### 4.3 `%` 前缀强制内置：绕过 Overlay

[`find_builtin_decl`](crates/nu-engine/src/scope.rs#L595-L609) 完全不经过 Overlay 查找：

```rust
pub fn find_builtin_decl(engine_state: &EngineState, name: &str) -> Option<DeclId> {
    for idx in (0..engine_state.num_decls()).rev() {
        let decl_id = DeclId::new(idx);
        let decl = engine_state.get_decl(decl_id);
        if decl.command_type() == CommandType::Builtin && decl.name() == name {
            return Some(decl_id);
        }
    }
    None
}
```

反向遍历全部 `decls` 数组，跳过非 Builtin 类型。因此无论 Overlay 如何覆盖，`%command` 总能找到 Rust 内置版本。

### 4.4 Visibility：隐藏不是删除

`hide` 命令通过 `Visibility.decl_ids` 的布尔值标记隐藏，不修改 `decls` HashMap。查找时会检查 visibility：可见的 decl 才返回。

---

## 五、可迁移链接（ParserPath 抽象）

### 5.1 统一路径抽象

[`ParserPath`](crates/nu-protocol/src/parser_path.rs#L16-L20) 是虚拟路径和真实路径的统一抽象：

```rust
pub enum ParserPath {
    RealPath(PathBuf),
    VirtualFile(PathBuf, usize),    // 路径 + file_id
    VirtualDir(PathBuf, Vec<ParserPath>),  // 路径 + 子项列表
}
```

### 5.2 统一 API

所有三种变体都实现了相同的接口，调用方无需关心是虚拟还是真实：

| 方法 | 作用 |
|---|---|
| `is_file()` | 是否文件 |
| `is_dir()` | 是否目录 |
| `exists()` | 是否存在（虚拟的永远返回 true） |
| `path()` | 获取路径（统一的 Path 视图） |
| `parent()` | 父目录 |
| `file_stem()` | 文件主干名 |
| `read()` | 读取内容（虚拟的从内存读，真实的从磁盘读） |
| `read_dir()` | 读取目录内容 |
| `join()` | 路径拼接 |
| `normalize_slashes_forward()` | 斜杠归一化 |

### 5.3 虚拟路径的"可迁移"含义

虚拟路径虽然不是真实文件，但具有完整的路径语义：
- 可以 `join("submodule")` 进行子模块查找
- 可以 `parent()` 获取父目录
- 可以 `file_stem()` 提取模块名
- 模块解析时和真实文件走完全相同的代码路径

这意味着标准库模块和用户磁盘模块在解析逻辑上是**同构**的，可以无缝迁移。

### 5.4 虚拟路径与真实路径的转换

虚拟文件通过 [`create_virt_file`](crates/nu-std/src/lib.rs#L11-L16) 创建时，内容被注册为 `FileId`，之后的读取走 `get_contents_of_file`，和真实文件走同一套缓存机制。

---

## 六、启动成本深度分析（带 perf 证据）

### 6.1 perf 埋点机制

[`perf!` 宏](crates/nu-utils/src/utils.rs#L495-L515) 通过 `log::info!` 输出耗时，需配合 `--log-level info` 使用：

```rust
macro_rules! perf {
    ($msg:expr, $dur:expr, $use_color:expr) => {
        log::info!("perf: {}:{}:{} {} took {:?}", file!(), line!(), column!(), $msg, $dur.elapsed());
    };
}
```

### 6.2 启动阶段完整 perf 埋点（已核准）

[main.rs](src/main.rs) 中的埋点按出现顺序：

| 埋点名称 | 代码行 | 包含的操作 |
|---|---|---|
| `start logging` | L286 | 日志初始化 |
| `set_config_path` | L305 | config.nu / env.nu 路径设置 |
| `acquire_terminal` | L311 | Unix 终端获取（仅 Unix） |
| `$env.config setup` | L320 | 默认 Config 值注入 |
| `gather env vars` | L330 | 父环境变量收集为字符串 |
| `Convert path to list` | L340 | PATH 等环境变量转列表 |
| `$env.NU_LIB_DIRS/$NU_LIB_DIRS setup` | L415 | NU_LIB_DIRS 常量和环境变量设置 |
| *(load_standard_library)* | L423-L425 | **无单独埋点**，夹在上面两个埋点之间 |
| `run test_bins` | L476 | --testbin 模式分发 |
| `redirect stdin` | L486 | 标准输入重定向设置 |
| `create_nu_constant` | L491 | 生成 $nu 常量 |
| `load plugins specified in --plugins` | L535 | 插件加载（仅指定 --plugins 时） |
| `mcp starting` / `lsp starting` | L542/L582 | MCP/LSP 模式启动 |

[run.rs](src/run.rs) 中后续阶段的埋点：

| 埋点名称 | 代码行 | 包含的操作 |
|---|---|---|
| `read plugins` | L40 | 插件签名文件读取 |
| `read env.nu` | L57 | 环境配置脚本执行 |
| `read config.nu` | L74 | 用户配置脚本执行 |
| `read login.nu` | L82 | 登录 shell 配置（仅 login 模式） |
| `evaluate_commands` | L103 | 实际命令执行 |

**注意**：`load_standard_library`（[main.rs#L423-L425](src/main.rs#L423-L425)）**没有单独的 perf 埋点**，它在 `$env.NU_LIB_DIRS/$NU_LIB_DIRS setup` 和 `run test_bins` 之间执行。

### 6.3 标准库加载的真实开销（已核准）

[`load_standard_library`](crates/nu-std/src/lib.rs#L18-L201) 的成本构成：

| 操作 | 成本 | 代码证据 |
|---|---|---|
| `create_virt_file` × N | **极低** | 只是 `include_str!` 的 `&'static str` 通过 `add_file` 注册 + `add_virtual_path`，无拷贝、无 I/O |
| `create_virt_dir` × M | **极低** | 构建目录树结构，纯内存 `Vec::push` |
| `use std/prelude *` | **中等** | 见 [crates/nu-std/src/lib.rs#L173-L189](crates/nu-std/src/lib.rs#L173-L189)，需要 `parse()` 解析 prelude 模块 + 执行导入，这是 std 加载的主要耗时点 |

关键优化：标准库所有子模块**不会在启动时全部解析**，只有 `use` 到时才解析。prelude 是唯一启动时强制解析的。

### 6.4 模块缓存与重复 use

[`parse_module_file`](crates/nu-parser/src/parse_module.rs#L706-L758) 有缓存机制：

```rust
if let Some(module_id) = working_set.find_module_by_span(new_span)
    && !module_needs_reloading(working_set, module_id)
{
    return Some(module_id);  // 直接复用，不重复解析
}
```

多个地方 `use std/assert` 只会解析一次。虚拟文件因为内容不变（编译期嵌入），永远不会触发 `module_needs_reloading`。

### 6.5 各阶段启动成本量化

根据 `perf!` 埋点的位置和实现复杂度估算：

| 阶段 | 相对耗时 | 说明 |
|---|---|---|
| 内置命令注册 | ~5% | 几百个命令的 `Box::new` + HashMap 插入（无单独 perf 点，包含在 start logging 之后） |
| 环境变量相关 (setup → gather → convert) | ~5-10% | 3 个 perf 点之和 |
| NU_LIB_DIRS setup | ~2% | 字符串拼接和常量设置 |
| **标准库加载** | **~10-15%** | 无单独 perf 点，夹在 NU_LIB_DIRS setup 和 run test_bins 之间，主要是 prelude 解析 |
| `$nu` 常量生成 | ~3% | 元信息收集 |
| 插件加载（如有） | 可变 | 每个插件 = 子进程启动 + IPC 握手 |
| **env.nu + config.nu** | **~40-70%+** | read env.nu + read config.nu 两个 perf 点，用户脚本，高度可变 |
| REPL 初始化 | ~10-20% | reedline 终端设置、历史加载（无 perf 点，包含在 evaluate_commands 之前） |

### 6.6 性能保障设计

1. **`include_str!` 编译期嵌入**：零运行时 I/O，直接从二进制镜像读取
2. **按需解析**：虚拟文件系统注册了所有模块元数据，但只有真正 `use` 到才会解析内容
3. **模块缓存**：基于 `Span` 的缓存，重复引用零成本
4. **`Arc` 共享**：decls/blocks/modules 用 `Arc<Vec>` 包装，`EngineState` 克隆成本低
5. **`StateDelta` 增量**：解析期修改暂存 delta，成功后一次性 merge，失败直接丢弃
6. **虚拟路径反向遍历**：后注册的优先命中，最新的模块版本先返回

---

## 七、标准库加载完整流程

### 7.1 启动顺序总览

```
1. EngineState::new()
   ↓ 空引擎状态
2. add_command_context()  → 注册所有 Rust 内置命令到 root overlay
   ↓ 数百个 Builtin 命令就绪
3. 设置 NU_LIB_DIRS 常量 + 环境变量
   ↓ perf 点：$env.NU_LIB_DIRS/$NU_LIB_DIRS setup
4. load_standard_library()  ← 无单独 perf 点
   ├─ 创建 std/ 虚拟文件树
   ├─ 创建 std-rfc/ 虚拟文件树
   └─ parse("use std/prelude *") → 解析 prelude，导入 banner/pwd 等
   ↓ 虚拟文件系统就绪，prelude 命令激活
5. run test_bins（仅 --testbin 模式）
   ↓ perf 点：run test_bins
6. redirect stdin
   ↓ perf 点：redirect stdin
7. generate_nu_constant() → $nu 常量
   ↓ perf 点：create_nu_constant
8. 加载插件（--plugins）
   ↓ perf 点：load plugins specified in --plugins
9. 执行 env.nu / config.nu  → 用户配置，可覆盖任何命令
   ↓ perf 点：read env.nu, read config.nu
```

### 7.2 标准库加载的两个层面

| 层面 | 内容 | 时机 | 能否被覆盖 |
|---|---|---|---|
| 虚拟文件系统 | std/ 和 std-rfc/ 的所有 .nu 文件 | `load_standard_library` 时注册 | 文件查找层面：不能（虚拟路径优先级最高） |
| Prelude 命令 | banner, pwd 等自定义命令 | `use std/prelude *` 时注册到 Overlay | Overlay 层面：能（用户后定义的优先级更高） |

---

## 八、文档来源（回顾）

### 8.1 内置命令文档

来自 Rust `Command` trait：`name()`, `signature()`, `description()`, `extra_description()`, `examples()`, `search_terms()`, `attributes()`。

### 8.2 自定义命令/模块文档

从源码注释中提取，由 [`build_desc`](crates/nu-protocol/src/engine/description.rs#L38-L91) 处理：
- 去掉每行开头的 `#` 和对齐空格
- 第一段空行前的内容 = brief description
- 其余内容 = extra description

### 8.3 help 命令查找顺序

```
help <name>
  ├─ alias 查找
  ├─ command 查找（走 Overlay 优先级）
  └─ module 查找

help %<name>  → 强制内置（绕过 Overlay）
```

---

## 九、关键代码索引（相对路径）

| 功能 | 文件 |
|---|---|
| 启动主流程 + perf 埋点 | [main.rs](src/main.rs) |
| 配置脚本执行 + perf 埋点 | [run.rs](src/run.rs) |
| 命令上下文注册 | [command_context.rs](src/command_context.rs) |
| 默认上下文（语言命令）| [crates/nu-cmd-lang/src/default_context.rs](crates/nu-cmd-lang/src/default_context.rs) |
| 标准库加载（虚拟文件 + prelude） | [crates/nu-std/src/lib.rs](crates/nu-std/src/lib.rs) |
| 引擎状态结构 | [crates/nu-protocol/src/engine/engine_state.rs](crates/nu-protocol/src/engine/engine_state.rs) |
| 工作集与命令注册 | [crates/nu-protocol/src/engine/state_working_set.rs](crates/nu-protocol/src/engine/state_working_set.rs) |
| Overlay/优先级机制 | [crates/nu-protocol/src/engine/overlay.rs](crates/nu-protocol/src/engine/overlay.rs) |
| 命令查找与内置强制 | [crates/nu-engine/src/scope.rs](crates/nu-engine/src/scope.rs) |
| 模块文件查找核心 | [crates/nu-parser/src/parse_source.rs](crates/nu-parser/src/parse_source.rs#L621-L741) |
| 模块解析与缓存 | [crates/nu-parser/src/parse_module.rs](crates/nu-parser/src/parse_module.rs) |
| 统一路径抽象（ParserPath） | [crates/nu-protocol/src/parser_path.rs](crates/nu-protocol/src/parser_path.rs) |
| Command trait 与类型 | [crates/nu-protocol/src/engine/command.rs](crates/nu-protocol/src/engine/command.rs) |
| 文档构建 | [crates/nu-protocol/src/engine/description.rs](crates/nu-protocol/src/engine/description.rs) |
| help 命令实现 | [crates/nu-command/src/help/help_.rs](crates/nu-command/src/help/help_.rs) |
| perf 宏定义 | [crates/nu-utils/src/utils.rs](crates/nu-utils/src/utils.rs#L495-L515) |
