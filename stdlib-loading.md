# NuShell 标准库加载、命令注册与启动分析

## 一、整体启动流程

启动入口在 [main.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/src/main.rs)，核心初始化顺序如下：

```
1. EngineState::new()                          → 创建空引擎状态
2. add_command_context()                       → 注册所有内置 Rust 命令
   ├─ nu_cmd_lang::add_default_context()       → 核心语言命令(def, use, module...)
   ├─ nu_cmd_plugin::add_plugin_command_context()
   ├─ nu_command::add_shell_command_context()  → 系统/文件/字符串命令
   ├─ nu_cmd_extra::add_extra_command_context()
   ├─ nu_cli::add_cli_context()                → repl, hint, completion 等
   └─ nu_explore::add_explore_context()
3. 设置 NU_LIB_DIRS, NU_PLUGIN_DIRS 等环境变量
4. load_standard_library()                     → 加载标准库(除非 --no-std-lib)
5. generate_nu_constant()                      → 生成 $nu 常量
6. 加载插件(--plugins)
7. 执行 env.nu / config.nu                     → 用户配置脚本
8. 进入 REPL 或执行脚本/命令
```

命令上下文注册代码见 [command_context.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/src/command_context.rs)。

---

## 二、标准库加载机制

标准库由 `nu-std` crate 提供，核心函数是 [load_standard_library](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-std/src/lib.rs#L18-L201)。

### 2.1 虚拟文件系统

标准库并非从磁盘读取，而是通过 Rust 的 `include_str!` 宏在**编译期**将所有 `.nu` 文件嵌入二进制：

```rust
// 示例：嵌入 std/mod.nu
let std_mod_virt_file_id = create_virt_file(
    &mut working_set,
    "std/mod.nu",
    include_str!("../std/mod.nu"),
);
```

文件结构：
- `std/` — 稳定标准库（assert, bench, dirs, dt, formats, help, input, iter, log, math, util, xml, config, testing, clip, random, prelude）
- `std-rfc/` — 实验性标准库（conversions, kv, path, str, tables, iter, random, xml, pb, url）

每个子模块如 `std/assert/mod.nu` 都被创建为 `VirtualPath::File`，并组织到 `VirtualPath::Dir` 中，最终注册到引擎状态的 `virtual_paths`。

### 2.2 Prelude 自动加载

标准库加载的最后一步会解析并执行：

```nu
use std/prelude *
```

这将 prelude 模块中所有 `export` 的定义自动导入全局作用域。当前 prelude 导出的命令见 [std/prelude/mod.nu](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-std/std/prelude/mod.nu)：
- `banner` — 欢迎横幅
- `pwd` — 打印当前目录

这些是**自定义命令**（`def` 定义），非 Rust 内置。

---

## 三、命令注册机制

### 3.1 内置命令（Builtin）注册

默认上下文通过 `bind_command!` 宏注册，见 [default_context.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-cmd-lang/src/default_context.rs)：

```rust
macro_rules! bind_command {
    ( $( $command:expr ),* $(,)? ) => {
        $( working_set.add_decl(Box::new($command)); )*
    };
}
```

`StateWorkingSet::add_decl` 在 [state_working_set.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-protocol/src/engine/state_working_set.rs#L115-L125) 中：

```rust
pub fn add_decl(&mut self, decl: Box<dyn Command>) -> DeclId {
    let name = decl.name().as_bytes().to_vec();
    self.delta.decls.push(decl);
    let decl_id = DeclId::new(self.num_decls() - 1);
    self.last_overlay_mut().insert_decl(name, decl_id);
    decl_id
}
```

每个命令被分配一个递增的 `DeclId`，并插入当前活动的 `OverlayFrame` 的 `decls: HashMap<Vec<u8>, DeclId>`。

### 3.2 CommandType 分类

见 [command.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-protocol/src/engine/command.rs#L32-L40)：

| CommandType | 来源 | 说明 |
|---|---|---|
| `Builtin` | Rust impl Command | 默认类型，所有 Rust 实现的命令 |
| `Custom` | `def` 定义的自定义命令 | 标准库 prelude、用户脚本等 |
| `Keyword` | 解析器关键字 | `use`, `module` 等在解析阶段特殊处理 |
| `External` | `extern` 声明或探测到的外部命令 | |
| `Alias` | `alias` 定义的别名 | |
| `Plugin` | 插件提供的命令 | |

---

## 四、命令查找与优先级（Overlay 机制）

### 4.1 Overlay 栈结构

见 [overlay.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-protocol/src/engine/overlay.rs)：

```
EngineState.scope (ScopeFrame)
├─ overlays: Vec<(name, OverlayFrame)>   // 所有 overlay
└─ active_overlays: Vec<OverlayId>        // 当前激活的，顺序决定优先级
```

每个 `OverlayFrame` 包含：
```rust
pub struct OverlayFrame {
    pub vars: HashMap<Vec<u8>, VarId>,
    pub decls: HashMap<Vec<u8>, DeclId>,      // 命令声明
    pub modules: HashMap<Vec<u8>, ModuleId>,  // 模块
    pub visibility: Visibility,               // hide/use 可见性控制
    pub origin: ModuleId,                     // 来源模块
    pub prefixed: bool,
    ...
}
```

### 4.2 查找顺序

`StateWorkingSet::find_decl` 在 [state_working_set.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-protocol/src/engine/state_working_set.rs#L443-L477)：

```
查找优先级（从高到低）：

1. delta.scope 中的 scope frames（从新到旧）
   ├─ 每个 scope frame 的 predecls（预声明，用于别名前向引用）
   └─ 每个 scope frame 的 active overlays（从后往前，即最后激活的先查）
       ├─ overlay.predecls
       └─ overlay.decls + visibility 检查

2. permanent_state (EngineState) 的 active overlays（从后往前）
   └─ overlay.decls + visibility 检查
```

**关键点：后激活的 overlay 优先级更高**，新注册的同名命令会覆盖旧的。

### 4.3 用户模块与内置命令的优先级

用户在 REPL 或脚本中定义的命令（`def`、`use`）会被添加到**新的或当前的 overlay**，因此优先级高于启动时注册的内置命令。

例如用户执行：
```nu
def pwd [] { "custom pwd" }
```
这个自定义 `pwd` 会被插入到当前 overlay 的 `decls`，由于 overlay 是后激活的，`find_decl("pwd")` 会先找到自定义版本。

### 4.4 `%` 前缀强制内置

即使同名被覆盖，使用 `%command` 语法可强制查找内置命令。实现见 [find_builtin_decl](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-engine/src/scope.rs#L595-L609)：

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

它反向遍历全部 `decls`，跳过非 Builtin 类型。

### 4.5 Visibility 隐藏机制

`hide` 命令通过将 `Visibility.decl_ids[decl_id]` 设为 `false` 来隐藏声明，而不是从 HashMap 删除。见 [Visibility](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-protocol/src/engine/overlay.rs#L8-L44)。

---

## 五、模块文件查找优先级

`use std/assert` 或 `use my_module` 的文件查找在 [find_in_dirs_env](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-engine/src/env.rs#L240-L299)：

```
查找顺序：

1. 相对当前 FILE_PWD（如果在模块内）或 PWD 查找
   - 先找 <name>.nu
   - 再找 <name>/mod.nu

2. 遍历 $NU_LIB_DIRS 常量中的目录依次查找
   $NU_LIB_DIRS 组成（main.rs#L342-L413）：
   ├─ 用户通过 -I 标志传入的路径
   ├─ 用户 env.nu 中 $env.NU_LIB_DIRS 设置的路径
   ├─ <config_dir>/scripts/         (默认)
   └─ <data_dir>/nushell/completions/ (默认)

3. 虚拟文件系统（std, std-rfc 已通过 virtual_paths 注册）
```

`$NU_LIB_DIRS` 是**常量**（`const_val`），优先级保证用户路径在默认路径之前。

---

## 六、文档来源

### 6.1 内置命令文档

来自 Rust `Command` trait 的方法实现，见 [command.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-protocol/src/engine/command.rs)：

| Trait 方法 | 对应帮助内容 |
|---|---|
| `name()` | 命令名 |
| `signature()` | 参数、标志、输入输出类型、分类 |
| `description()` | 单行简短描述 |
| `extra_description()` | 扩展描述 |
| `examples()` | 代码示例列表 |
| `search_terms()` | 搜索关键词 |
| `attributes()` | `@category`, `@deprecated` 等属性 |

### 6.2 自定义命令/模块文档

从源代码中的注释提取。解析时，`def` 或 `module` 之前的注释行被记录为 Span，然后通过 [build_desc](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-protocol/src/engine/description.rs#L38-L91) 处理：

```rust
pub(super) fn build_desc(comment_lines: &[&[u8]]) -> (String, String) {
    // 1. 去掉每行开头的 '#' 和对齐空格
    // 2. 拼接所有注释行
    // 3. 用 "\n\n" 分割：第一段 = brief_desc，其余 = extra_desc
}
```

模块注释存储在 `Doccomments.module_comments: HashMap<ModuleId, Vec<Span>>`，通过 `EngineState::build_module_desc` 访问。

### 6.3 help 命令查找流程

[help_.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-command/src/help/help_.rs) 中的查找顺序：

```
help <name>
├─ 先查 alias
├─ 再查 command
└─ 最后查 module

help %<name>  → 强制查找内置命令 (find_builtin_decl)
```

另外，`help` 命令本身可以被用户自定义命令覆盖（只要定义名为 `help` 的命令）。

---

## 七、启动成本分析

### 7.1 低成本项（几乎可忽略）

| 阶段 | 开销 | 原因 |
|---|---|---|
| 内置命令注册 | O(n) n≈数百 | 只是 `Box::new` + `Vec::push` + `HashMap::insert`，无 I/O |
| 生成 $nu 常量 | 很小 | 构造几个 Value |
| 环境变量转换 | 很小 | 字符串转 Value |

### 7.2 中等成本项

| 阶段 | 开销 | 原因 |
|---|---|---|
| 标准库加载 | 解析 prelude | 虚拟文件创建是纯内存（O(1) 字符串引用），只有 `use std/prelude *` 需要运行 nu 解析器解析 + 执行 |
| 插件加载 | 每个插件 = 进程启动 + IPC | 通过 `--plugins` 注册的插件需要 spawn 子进程 + 握手 |

### 7.3 高成本项（用户可控）

| 阶段 | 开销 | 原因 |
|---|---|---|
| env.nu / config.nu | 取决于用户脚本 | 执行用户配置的所有 nu 代码 |
| REPL 初始化（reedline） | 中等 | 终端设置、补全引擎、历史记录加载 |

### 7.4 性能保障设计

1. **`include_str!` 编译期嵌入**：标准库无磁盘 I/O，避免启动时文件系统访问
2. **虚拟文件系统**：`std` 和 `std-rfc` 所有模块作为内存字符串存在，按需解析
3. **`Arc` 共享**：`EngineState.decls`, `blocks`, `modules` 用 `Arc<Vec<...>>` 包装，克隆成本低
4. **`StateDelta` 增量合并**：解析时修改暂存 delta，解析成功后一次性 merge 到永久状态

### 7.5 启动性能追踪

代码中大量使用 `perf!` 宏（main.rs），支持 `--log-level info` 查看各阶段耗时：

```
set_config_path
$env.config setup
gather env vars
Convert path to list
$env.NU_LIB_DIRS/$NU_LIB_DIRS setup
load_standard_library (隐含)
create_nu_constant
...
```

---

## 八、关键代码索引

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
| 模块文件查找 | [env.rs (find_in_dirs_env)](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-engine/src/env.rs) |
| Command trait 与类型 | [command.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-protocol/src/engine/command.rs) |
| 文档构建 | [description.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-protocol/src/engine/description.rs) |
| help 命令实现 | [help_.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/80-nushell/crates/nu-command/src/help/help_.rs) |
