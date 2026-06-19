# NuShell 标准库加载、命令注册与启动分析

## 一、核心结论速览

| 问题 | 答案 |
|---|---|
| 虚拟 std 路径和用户库路径是同一套优先级吗？ | **不是**。文件查找优先级：虚拟路径 > 当前目录 > NU_LIB_DIRS |
| 命令覆盖和模块查找是同一套优先级吗？ | **不是**。两套独立机制：文件查找 vs Overlay 注册顺序 |
| 用户模块能覆盖标准库模块吗？ | **文件查找层面不能**（虚拟路径优先级最高），但 **use/def 注册层面能**（后激活的 Overlay 优先级更高） |
| `%cmd` 为什么总能找到内置命令？ | 它不通过 Overlay 查找，直接遍历所有 Decl 找 `CommandType::Builtin` |
| 标准库加载对启动成本影响大吗？ | **不大**。虚拟文件是编译期嵌入的字符串引用，只有 prelude 需要解析 |

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

## 三、模块文件查找优先级详解

### 3.1 完整查找顺序

核心函数是 [`find_in_dirs_with_id`](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-parser/src/parse_source.rs#L631-L690)：

```
优先级从高到低：

1. 虚拟路径精确匹配
   find_virtual_path("std/assert") → 命中直接返回

2. 虚拟路径绝对路径匹配
   abs_virtual_filename = cwd + filename
   find_virtual_path(abs_virtual_filename) → 命中返回

3. 当前工作目录相对路径
   absolute_with(filename, cwd) && p.exists()

4. NU_LIB_DIRS 常量遍历（按顺序，先到先得）
   for lib_dir in $NU_LIB_DIRS:
       path = lib_dir / filename
       if path.exists() → 返回
```

**关键结论：虚拟路径优先级最高，用户磁盘上的同名模块无法覆盖标准库模块。**

### 3.2 NU_LIB_DIRS 的组成

在 [main.rs#L342-L415](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/src/main.rs#L342-L415) 中设置：

```
$NU_LIB_DIRS（常量版本，解析时使用）优先级从高到低：

  1. -I <path> 命令行参数传入的路径（多个，按顺序）
  2. 父环境中 NU_LIB_DIRS 环境变量的路径
  3. <config_dir>/scripts/          （默认脚本目录）
  4. <data_dir>/nushell/completions/ （默认补全目录）
```

`$env.NU_LIB_DIRS`（运行时环境变量版本）则不包含默认路径，只包含用户设置的路径。

### 3.3 虚拟路径的反向遍历

[`find_virtual_path`](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-protocol/src/engine/state_working_set.rs#L1045-L1062) 从后往前遍历：

```rust
for (virtual_name, virtual_path) in self.delta.virtual_paths.iter().rev() { ... }
for (virtual_name, virtual_path) in self.permanent_state.virtual_paths.iter().rev() { ... }
```

这意味着**后注册的虚拟路径优先级更高**。如果 delta 中有同名虚拟文件，会覆盖 permanent_state 中的。

---

## 四、命令/模块覆盖优先级（Overlay 机制）

### 4.1 Overlay 激活 = 优先级提升

[`add_overlay`](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-protocol/src/engine/state_working_set.rs#L922-L967) 的关键行为：

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
| 同一 Overlay 内注册同名 decl | HashMap::insert，后者覆盖前者 | [add_decl](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-protocol/src/engine/state_working_set.rs#L115-L125) |
| 不同 Overlay 同名 decl | 后激活的 Overlay 先被查到 | [find_decl](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-protocol/src/engine/state_working_set.rs#L443-L477) |
| 用户 `def pwd` 覆盖内置 `pwd` | ✅ 可以覆盖，因为用户 def 在更新的 overlay | 同上 |
| 用户 `use std/pwd` 覆盖内置 `pwd` | ✅ 可以覆盖，标准库 prelude 的自定义 pwd 也是如此 | 同上 |
| 用户磁盘同名模块覆盖 std 模块 | ❌ 不能，文件查找时虚拟路径优先级更高 | [find_in_dirs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-parser/src/parse_source.rs#L631-L690) |

### 4.3 `%` 前缀强制内置：绕过 Overlay

[`find_builtin_decl`](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-engine/src/scope.rs#L595-L609) 完全不经过 Overlay 查找：

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

[`ParserPath`](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-protocol/src/parser_path.rs#L16-L20) 是虚拟路径和真实路径的统一抽象：

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

虚拟文件通过 [`create_virt_file`](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-std/src/lib.rs) 创建时，内容被注册为 `FileId`，之后的读取走 `get_contents_of_file`，和真实文件走同一套缓存机制。

---

## 六、启动成本深度分析

### 6.1 标准库加载的真实开销

[`load_standard_library`](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-std/src/lib.rs#L18-L201) 的成本构成：

| 操作 | 成本 | 说明 |
|---|---|---|
| `create_virt_file` × N | **极低** | 只是 `include_str!` 的 `&'static str` 引用 + `Vec::push`，无拷贝、无 I/O |
| `create_virt_dir` × M | **极低** | 构建目录树结构，纯内存操作 |
| `use std/prelude *` | **中等** | 需要解析 prelude 模块 + 执行导入，这是 std 加载的主要耗时点 |

关键优化：标准库所有子模块**不会在启动时全部解析**，只有 `use` 到时才解析。prelude 是唯一启动时强制解析的。

### 6.2 模块缓存与重复 use

[`parse_module_file`](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-parser/src/parse_module.rs#L706-L758) 有缓存机制：

```rust
if let Some(module_id) = working_set.find_module_by_span(new_span)
    && !module_needs_reloading(working_set, module_id)
{
    return Some(module_id);  // 直接复用，不重复解析
}
```

多个地方 `use std/assert` 只会解析一次。虚拟文件因为内容不变（编译期嵌入），永远不会触发 `module_needs_reloading`。

### 6.3 各阶段启动成本量化

根据代码中的 `perf!` 埋点和实现复杂度估算：

| 阶段 | 相对耗时 | 说明 |
|---|---|---|
| 内置命令注册 | ~5% | 几百个命令的 `Box::new` + HashMap 插入 |
| `$env.config` setup | ~2% | 默认 Config 构造 |
| gather env vars | ~3% | 系统环境变量转 Value |
| NU_LIB_DIRS setup | ~2% | 字符串拼接和常量设置 |
| **标准库加载** | **~10-15%** | 主要是 prelude 解析 + 虚拟文件注册 |
| `$nu` 常量生成 | ~3% | 元信息收集 |
| 插件加载（如有） | 可变 | 每个插件 = 子进程启动 + IPC 握手 |
| **env.nu + config.nu** | **~40-70%+** | 用户脚本，高度可变 |
| REPL 初始化 | ~10-20% | reedline 终端设置、历史加载 |

### 6.4 性能保障设计

1. **`include_str!` 编译期嵌入**：零运行时 I/O，直接从二进制镜像读取
2. **按需解析**：虚拟文件系统注册了所有模块元数据，但只有真正 `use` 到才会解析内容
3. **模块缓存**：基于 `Span` 的缓存，重复引用零成本
4. **`Arc` 共享**：decls/blocks/modules 用 `Arc<Vec>` 包装，`EngineState` 克隆成本低
5. **`StateDelta` 增量**：解析期修改暂存 delta，成功后一次性 merge，失败直接丢弃
6. **虚拟路径反向遍历**：后注册的优先命中，最新的模块版本先返回

### 6.5 启动性能观测

设置 `--log-level info` 可以看到各阶段 `perf!` 输出：

```
set_config_path          ... 0.1ms
$env.config setup        ... 0.2ms
gather env vars          ... 0.5ms
Convert path to list     ... 0.3ms
$env.NU_LIB_DIRS setup   ... 0.2ms
create_nu_constant       ... 0.4ms
load plugins             ... （如果有插件）
```

`load_standard_library` 没有单独的 perf 点，但它在 NU_LIB_DIRS setup 和 create_nu_constant 之间执行。

---

## 七、标准库加载完整流程

### 7.1 启动顺序总览

```
1. EngineState::new()
   ↓ 空引擎状态
2. add_command_context()  → 注册所有 Rust 内置命令到 root overlay
   ↓ 数百个 Builtin 命令就绪
3. 设置 NU_LIB_DIRS 常量
   ↓ 常量列表就绪
4. load_standard_library()
   ├─ 创建 std/ 虚拟文件树
   ├─ 创建 std-rfc/ 虚拟文件树
   └─ use std/prelude *   → 解析 prelude，导入 banner/pwd 等
   ↓ 虚拟文件系统就绪，prelude 命令激活
5. generate_nu_constant() → $nu 常量
   ↓
6. 加载插件（--plugins）
   ↓
7. 执行 env.nu / config.nu  → 用户配置，可覆盖任何命令
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

从源码注释中提取，由 [`build_desc`](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-protocol/src/engine/description.rs#L38-L91) 处理：
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

## 九、关键代码索引

| 功能 | 文件 |
|---|---|
| 启动主流程 | [main.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/src/main.rs) |
| 命令上下文注册 | [command_context.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/src/command_context.rs) |
| 默认上下文（语言命令）| [default_context.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-cmd-lang/src/default_context.rs) |
| 标准库加载 | [nu-std/src/lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-std/src/lib.rs) |
| 引擎状态结构 | [engine_state.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-protocol/src/engine/engine_state.rs) |
| 工作集与命令注册 | [state_working_set.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-protocol/src/engine/state_working_set.rs) |
| Overlay/优先级机制 | [overlay.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-protocol/src/engine/overlay.rs) |
| 命令查找与内置强制 | [scope.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-engine/src/scope.rs) |
| 模块文件查找核心 | [parse_source.rs (find_in_dirs)](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-parser/src/parse_source.rs#L621-L741) |
| 模块解析与缓存 | [parse_module.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-parser/src/parse_module.rs) |
| 统一路径抽象 | [parser_path.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-protocol/src/parser_path.rs) |
| Command trait 与类型 | [command.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-protocol/src/engine/command.rs) |
| 文档构建 | [description.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-protocol/src/engine/description.rs) |
| help 命令实现 | [help_.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-command/src/help/help_.rs) |
