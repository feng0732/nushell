# NuShell 标准库加载、命令注册与启动分析

## 一、核心结论速览

| 问题 | 答案 |
|---|---|
| 虚拟 std 路径和用户库路径是同一套优先级吗？ | **不是**。文件查找优先级：虚拟路径 > 当前目录 > NU_LIB_DIRS |
| 命令覆盖和模块查找是同一套优先级吗？ | **不是**。两套独立机制：文件查找 vs Overlay 注册顺序 |
| 用户模块能覆盖标准库模块吗？ | **文件查找层面不能**（虚拟路径优先级最高），但 **use/def 注册层面能**（后激活的 Overlay 优先级更高） |
| `%cmd` 为什么总能找到内置命令？ | 它不通过 Overlay 查找，直接遍历所有 Decl 找 `CommandType::Builtin` |
| env.nu 中 `$env.NU_LIB_DIRS = [...]` 能影响后续模块查找吗？ | **不能**。解析期查找用的是常量 `$NU_LIB_DIRS`，env.nu 只改了环境变量版本，常量不随之更新 |

---

## 二、两套独立的优先级机制

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

## 三、NU_LIB_DIRS 生命周期：初始设置 → env.nu 覆盖 → 解析期常量

这是最容易混淆的部分。`$env.NU_LIB_DIRS`（环境变量）和 `$NU_LIB_DIRS`（常量）是**两个独立的存储**，在不同时机被读写，对模块查找的影响完全不同。

### 3.1 启动时序图（带代码行号）

```
时间线 ──────────────────────────────────────────────────────────────►

[1] main.rs L354-386  初始设置
    │
    │  读取父进程环境变量 NU_LIB_DIRS → user_lib_dirs
    │  追加 -I 命令行参数 → user_lib_dirs
    │  追加默认路径 → all_lib_dirs = user_lib_dirs + defaults
    │
    │  同时写入两处：
    │    $env.NU_LIB_DIRS = all_lib_dirs   ← L397 add_env_var
    │    $NU_LIB_DIRS 常量 = all_lib_dirs  ← L411 set_variable_const_val
    │                       ↓
    │              merge_delta 到 EngineState
    │
    ├─ 此时 $env 和 $常量 内容完全相同
    │
[2] main.rs L423-425  标准库加载
    │
    │  load_standard_library()
    │  ├─ 创建虚拟文件 std/, std-rfc/  （不走 NU_LIB_DIRS 查找）
    │  └─ parse("use std/prelude *")   （走虚拟路径查找，命中 std/prelude）
    │
    │  ⚠ stdlib 加载时 $NU_LIB_DIRS 常量已是 [1] 中设置的值，
    │    但虚拟路径优先级更高，所以 NU_LIB_DIRS 对 std 模块查找无影响
    │
[3] main.rs L491      $nu 常量生成
    │
    │  generate_nu_constant()
    │
[4] run.rs / config_files.rs  env.nu 执行
    │
    │  eval_config_contents(env.nu, engine_state, stack)
    │  ├─ eval_source() → 执行用户 nu 代码
    │  │   如果用户写了 $env.NU_LIB_DIRS = ["/my/libs"]
    │  │   → 修改的是 stack.env_vars 中的环境变量
    │  │
    │  └─ engine_state.merge_env(stack)  ← config_files.rs#L258
    │      → 将 stack.env_vars 合并回 engine_state.env_vars
    │      → $env.NU_LIB_DIRS 被更新为用户值 ✅
    │      → $NU_LIB_DIRS 常量不变 ❌（merge_env 只处理 env_vars，不处理 const_val）
    │
    ├─ 此时 $env.NU_LIB_DIRS = 用户值（来自 env.nu）
    │      $NU_LIB_DIRS 常量 = 初始值（来自 [1]）
    │
[5] run.rs / config_files.rs  config.nu 执行
    │
    │  同 [4] 的流程，$env 可再改，常量仍不变
    │
[6] 后续 REPL/脚本中的 "use xxx"
    │
    │  find_in_dirs() → find_dirs_var() → 只读 $NU_LIB_DIRS 常量
    │  → 使用的是 [1] 中设置的初始值，不受 env.nu/config.nu 影响
```

### 3.2 关键代码证据

**merge_env 只更新环境变量，不更新常量**（[engine_state.rs#L365-L392](crates/nu-protocol/src/engine/engine_state.rs#L365-L392)）：

```rust
pub fn merge_env(&mut self, stack: &mut Stack) -> Result<(), ShellError> {
    for mut scope in stack.env_vars.drain(..) {
        for (overlay_name, mut env) in Arc::make_mut(&mut scope).drain() {
            // 只操作 self.env_vars，不涉及 variables/const_val
            if let Some(env_vars) = Arc::make_mut(&mut self.env_vars).get_mut(&overlay_name) {
                env_vars.extend(env.drain());
            } else {
                Arc::make_mut(&mut self.env_vars).insert(overlay_name, env);
            }
        }
    }
    // ... 也没有任何更新 const_val 的逻辑
}
```

**解析时只认常量**（[parse_source.rs#L615-L618](crates/nu-parser/src/parse_source.rs#L615-L618)）：

```rust
pub fn find_dirs_var(working_set: &StateWorkingSet, var_name: &str) -> Option<VarId> {
    working_set
        .find_variable(format!("${var_name}").as_bytes())
        .filter(|var_id| working_set.get_variable(*var_id).const_val.is_some())
}
```

### 3.3 三张表的对比

| 属性 | `$env.NU_LIB_DIRS` | `$NU_LIB_DIRS` 常量 | 虚拟路径 |
|---|---|---|---|
| 存储 | `engine_state.env_vars` | `engine_state.variables[var_id].const_val` | `engine_state.virtual_paths` |
| 设置时机 | main.rs 初始 + env.nu 可覆盖 | main.rs 初始，之后不变 | load_standard_library 时注册 |
| env.nu 能改吗 | ✅ 通过 merge_env | ❌ merge_env 不涉及常量 | ❌ 不受影响 |
| 模块查找用哪个 | 不用 | ✅ find_dirs_var 只认常量 | ✅ 优先级最高 |
| `use` 命令查找顺序 | — | 第 4 优先级 | 第 1 优先级 |

### 3.4 对用户的影响

如果用户在 env.nu 中写了 `$env.NU_LIB_DIRS = ["/my/libs"]`：
- ✅ `$env.NU_LIB_DIRS` 会变为 `["/my/libs"]`
- ❌ `use my_module` **不会**在 `/my/libs` 中查找
- ❌ `$NU_LIB_DIRS` 常量仍是初始值
- 要让自定义路径生效，用户必须用 `nu -I /my/libs` 启动，或在脚本中通过 `const NU_LIB_DIRS` 重定义常量

---

## 四、模块文件查找优先级详解

### 4.1 完整查找顺序（带代码证据）

核心函数是 [`find_in_dirs_with_id`](crates/nu-parser/src/parse_source.rs#L631-L690)：

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

4. $NU_LIB_DIRS 常量遍历（按列表顺序，先到先得）
   for lib_dir in $NU_LIB_DIRS:
       path = lib_dir / filename
       if path.exists() → 返回
```

**关键结论：虚拟路径优先级最高，用户磁盘上的同名模块无法覆盖标准库模块。**

### 4.2 NU_LIB_DIRS 初始设置逻辑

在 [main.rs#L342-L415](src/main.rs#L342-L415) 中，代码注释：

```rust
// Set up NU_LIB_DIRS: constant = defaults + env + -I, env = env + -I
```

构造步骤：

| 步骤 | 来源 | 代码位置 |
|---|---|---|
| 1 | 从父进程 `NU_LIB_DIRS` 环境变量读取 | [main.rs#L355-L371](src/main.rs#L355-L371) |
| 2 | 追加 `-I` 命令行参数（分隔符 `:`, `;`, `\x1e`）| [main.rs#L374-L377](src/main.rs#L374-L377) |
| 3 | 追加默认路径 `<config>/scripts/` | [main.rs#L380-L384](src/main.rs#L380-L384) |
| 4 | 追加默认路径 `<data>/nushell/completions/` | 同上 |
| 5 | `user_lib_dirs.into_iter().chain(default_paths)` → 用户路径在前 | [main.rs#L386](src/main.rs#L386) |
| 6 | 同时写入 `$env.NU_LIB_DIRS` 和 `$NU_LIB_DIRS` 常量 | [main.rs#L397-L413](src/main.rs#L397-L413) |

其中：
- `nushell_config_path` = `nu_path::nu_config_dir()`，通常是 `~/.config/nushell/`（受 XDG_CONFIG_HOME 影响）
- `data_dir` = `nu_path::data_dir()`，平台相关的数据目录

### 4.3 虚拟路径的反向遍历

[`find_virtual_path`](crates/nu-protocol/src/engine/state_working_set.rs#L1045-L1062) 从后往前遍历：

```rust
for (virtual_name, virtual_path) in self.delta.virtual_paths.iter().rev() { ... }
for (virtual_name, virtual_path) in self.permanent_state.virtual_paths.iter().rev() { ... }
```

**后注册的虚拟路径优先级更高**。delta 中的虚拟路径优先于 permanent_state 中的同名路径。

---

## 五、命令/模块覆盖优先级（Overlay 机制）

### 5.1 Overlay 激活 = 优先级提升

[`add_overlay`](crates/nu-protocol/src/engine/state_working_set.rs#L922-L967)：

```rust
last_scope_frame.active_overlays.retain(|id| id != &overlay_id);
last_scope_frame.active_overlays.push(overlay_id);
```

**每次 `use` 或 `overlay use` 都会把对应 overlay 移到 active_overlays 末尾**，而 `find_decl` 从后往前查，因此后激活的优先级更高。

### 5.2 同名覆盖的完整场景

| 场景 | 行为 | 代码位置 |
|---|---|---|
| 同一 Overlay 内注册同名 decl | HashMap::insert，后者覆盖前者 | [add_decl](crates/nu-protocol/src/engine/state_working_set.rs#L115-L125) |
| 不同 Overlay 同名 decl | 后激活的 Overlay 先被查到 | [find_decl](crates/nu-protocol/src/engine/state_working_set.rs#L443-L477) |
| 用户 `def pwd` 覆盖内置 `pwd` | ✅ 可以覆盖 | 同上 |
| 标准库 prelude `pwd` 覆盖内置 `pwd` | ✅ 可以覆盖 | 同上 |
| 用户磁盘同名模块覆盖 std 模块 | ❌ 不能，文件查找时虚拟路径优先级更高 | [find_in_dirs](crates/nu-parser/src/parse_source.rs#L631-L690) |

### 5.3 `%` 前缀强制内置：绕过 Overlay

[`find_builtin_decl`](crates/nu-engine/src/scope.rs#L595-L609) 直接遍历全部 `decls` 数组，跳过非 `CommandType::Builtin`，不走 Overlay 查找。

### 5.4 Visibility：隐藏不是删除

`hide` 命令通过 `Visibility.decl_ids` 的布尔值标记隐藏，不修改 `decls` HashMap。

---

## 六、可迁移链接（ParserPath 抽象）

[`ParserPath`](crates/nu-protocol/src/parser_path.rs#L16-L20) 统一了 `RealPath`、`VirtualFile`、`VirtualDir` 三种路径：

```rust
pub enum ParserPath {
    RealPath(PathBuf),
    VirtualFile(PathBuf, usize),         // 路径 + file_id
    VirtualDir(PathBuf, Vec<ParserPath>), // 路径 + 子项列表
}
```

所有变体提供相同 API（`is_file`, `read`, `join`, `parent` 等），标准库模块和用户磁盘模块在解析逻辑上**同构**，可以无缝迁移。虚拟文件通过 [`create_virt_file`](crates/nu-std/src/lib.rs#L11-L16) 创建，内容注册为 `FileId`，读取走同一套缓存机制。

---

## 七、启动成本（只含可观察证据）

### 7.1 perf 埋点机制

[`perf!` 宏](crates/nu-utils/src/utils.rs#L495-L515) 通过 `log::info!` 输出耗时，需配合 `--log-level info` 使用：

```rust
macro_rules! perf {
    ($msg:expr, $dur:expr, $use_color:expr) => {
        log::info!("perf: {}:{}:{} {} took {:?}", file!(), line!(), column!(), $msg, $dur.elapsed());
    };
}
```

### 7.2 main.rs 中的 perf 埋点

| 埋点名称 | 代码行 | 包含的操作 |
|---|---|---|
| `start logging` | L286 | 日志初始化 |
| `set_config_path` | L305 | config.nu / env.nu 路径设置 |
| `acquire_terminal` | L311 | Unix 终端获取（仅 Unix） |
| `$env.config setup` | L320 | 默认 Config 值注入 |
| `gather env vars` | L330 | 父环境变量收集为字符串 |
| `Convert path to list` | L340 | PATH 等环境变量转列表 |
| `$env.NU_LIB_DIRS/$NU_LIB_DIRS setup` | L415 | NU_LIB_DIRS 常量和环境变量设置 |
| *(load_standard_library)* | L423-L425 | **无单独埋点** |
| `run test_bins` | L476 | --testbin 模式分发 |
| `redirect stdin` | L486 | 标准输入重定向设置 |
| `create_nu_constant` | L491 | 生成 $nu 常量 |
| `load plugins specified in --plugins` | L535 | 插件加载（仅指定 --plugins 时） |
| `mcp starting` / `lsp starting` | L542/L582 | MCP/LSP 模式启动 |

### 7.3 run.rs / config_files.rs 中的 perf 埋点

| 埋点名称 | 代码位置 | 包含的操作 |
|---|---|---|
| `read plugins` | [run.rs#L40](src/run.rs#L40) | 插件签名文件读取 |
| `read env.nu` | [run.rs#L57](src/run.rs#L57) | 环境配置脚本执行 |
| `read config.nu` | [run.rs#L74](src/run.rs#L74) | 用户配置脚本执行 |
| `read login.nu` | [run.rs#L82](src/run.rs#L82) | 登录 shell 配置（仅 login 模式） |
| `setup_config` | [run.rs#L209](src/run.rs#L209) | REPL 模式完整配置（plugins + env + config + autoload） |
| `evaluate_commands` | [run.rs#L103](src/run.rs#L103) | 实际命令执行 |
| `evaluate_file` | [run.rs#L175](src/run.rs#L175) | 脚本文件执行 |
| `evaluate_repl` | [run.rs#L219](src/run.rs#L219) | REPL 主循环启动 |

### 7.4 eval_source 中的 perf 埋点

每个 `eval_source` 调用（包括 env.nu/config.nu 的执行）都产生一条 `eval_source <fname>` 埋点（[util.rs#L270-L277](crates/nu-cli/src/util.rs#L270-L277)）。

### 7.5 load_standard_library 无单独埋点

`load_standard_library` 在 [main.rs#L423-L425](src/main.rs#L423-L425) 调用，夹在 `$env.NU_LIB_DIRS/$NU_LIB_DIRS setup`（L415）和 `run test_bins`（L476）之间，没有自己的 perf 埋点。如需单独测量，可观察这两个相邻埋点的时间差，或添加埋点后重新编译。

### 7.6 标准库加载的可观察特征

| 操作 | 可观察性 | 代码证据 |
|---|---|---|
| `create_virt_file` × N | 编译期嵌入 `include_str!`，运行时只有 `add_file` + `add_virtual_path` | [crates/nu-std/src/lib.rs#L11-L16](crates/nu-std/src/lib.rs#L11-L16) |
| `create_virt_dir` × M | 纯内存 `Vec::push` | 同上 |
| `use std/prelude *` | 需要 `parse()` + 执行导入，是 stdlib 加载的主要耗时操作 | [crates/nu-std/src/lib.rs#L173-L189](crates/nu-std/src/lib.rs#L173-L189) |
| 子模块（assert, math 等） | 启动时不解析，只有 `use` 时才按需解析 | [parse_module_file](crates/nu-parser/src/parse_module.rs#L706-L758) 有缓存 |

### 7.7 模块缓存

```rust
// parse_module_file 中的缓存判断
if let Some(module_id) = working_set.find_module_by_span(new_span)
    && !module_needs_reloading(working_set, module_id)
{
    return Some(module_id);  // 直接复用，不重复解析
}
```

虚拟文件内容不变（编译期嵌入），永远不会触发 `module_needs_reloading`。

### 7.8 性能保障设计

1. **`include_str!` 编译期嵌入**：零运行时 I/O
2. **按需解析**：只有 `use` 到的模块才解析内容
3. **模块缓存**：基于 Span，重复引用零成本
4. **`Arc` 共享**：decls/blocks/modules 用 `Arc<Vec>` 包装
5. **`StateDelta` 增量**：成功后一次性 merge，失败直接丢弃

---

## 八、标准库加载完整流程

### 8.1 启动顺序总览

```
1. EngineState::new()
   ↓ 空引擎状态
2. add_command_context()  → 注册所有 Rust 内置命令到 root overlay
   ↓ 数百个 Builtin 命令就绪
3. 设置 NU_LIB_DIRS 常量 + 环境变量（内容相同）
   ↓ perf 点：$env.NU_LIB_DIRS/$NU_LIB_DIRS setup
4. load_standard_library()  ← 无单独 perf 点
   ├─ 创建 std/ 虚拟文件树（include_str! 嵌入，不走 NU_LIB_DIRS）
   ├─ 创建 std-rfc/ 虚拟文件树
   └─ parse("use std/prelude *") → 虚拟路径命中，解析 prelude
   ↓ 虚拟文件系统就绪，prelude 命令激活
5. generate_nu_constant() → $nu 常量
   ↓ perf 点：create_nu_constant
6. 加载插件（--plugins）
   ↓ perf 点：load plugins specified in --plugins
7. 执行 env.nu
   ├─ eval_source + merge_env
   ├─ $env.NU_LIB_DIRS 可能被用户覆盖 ✅
   └─ $NU_LIB_DIRS 常量不变 ❌
   ↓ perf 点：read env.nu
8. 执行 config.nu
   └─ 同上，$env 可再改，常量仍不变
   ↓ perf 点：read config.nu
9. 进入 REPL / 执行脚本 / 执行命令
   ↓ 后续 "use xxx" 走 $NU_LIB_DIRS 常量（初始值），不受 env.nu 影响
```

### 8.2 标准库加载的两个层面

| 层面 | 内容 | 时机 | 能否被覆盖 |
|---|---|---|---|
| 虚拟文件系统 | std/ 和 std-rfc/ 的所有 .nu 文件 | `load_standard_library` 时注册 | 文件查找层面：不能（虚拟路径优先级最高） |
| Prelude 命令 | banner, pwd 等自定义命令 | `use std/prelude *` 时注册到 Overlay | Overlay 层面：能（用户后定义的优先级更高） |

---

## 九、文档来源

### 9.1 内置命令文档

来自 Rust `Command` trait：`name()`, `signature()`, `description()`, `extra_description()`, `examples()`, `search_terms()`, `attributes()`。

### 9.2 自定义命令/模块文档

从源码注释中提取，由 [`build_desc`](crates/nu-protocol/src/engine/description.rs#L38-L91) 处理：
- 去掉每行开头的 `#` 和对齐空格
- 第一段空行前的内容 = brief description
- 其余内容 = extra description

### 9.3 help 命令查找顺序

```
help <name>
  ├─ alias 查找
  ├─ command 查找（走 Overlay 优先级）
  └─ module 查找

help %<name>  → 强制内置（绕过 Overlay）
```

---

## 十、关键代码索引

| 功能 | 文件 |
|---|---|
| 启动主流程 + perf 埋点 | [main.rs](src/main.rs) |
| 配置脚本执行 + perf 埋点 | [run.rs](src/run.rs) |
| 配置文件读取与 merge_env | [config_files.rs](src/config_files.rs) |
| 命令上下文注册 | [command_context.rs](src/command_context.rs) |
| 默认上下文（语言命令）| [default_context.rs](crates/nu-cmd-lang/src/default_context.rs) |
| 标准库加载 | [crates/nu-std/src/lib.rs](crates/nu-std/src/lib.rs) |
| 引擎状态 + merge_env | [engine_state.rs](crates/nu-protocol/src/engine/engine_state.rs) |
| 工作集 + add_decl/find_decl | [state_working_set.rs](crates/nu-protocol/src/engine/state_working_set.rs) |
| Overlay/优先级机制 | [overlay.rs](crates/nu-protocol/src/engine/overlay.rs) |
| 命令查找与内置强制 | [scope.rs](crates/nu-engine/src/scope.rs) |
| 模块文件查找 + find_dirs_var | [parse_source.rs](crates/nu-parser/src/parse_source.rs) |
| 模块解析与缓存 | [parse_module.rs](crates/nu-parser/src/parse_module.rs) |
| ParserPath 抽象 | [parser_path.rs](crates/nu-protocol/src/parser_path.rs) |
| Command trait 与类型 | [command.rs](crates/nu-protocol/src/engine/command.rs) |
| 文档构建 | [description.rs](crates/nu-protocol/src/engine/description.rs) |
| help 命令实现 | [help_.rs](crates/nu-command/src/help/help_.rs) |
| perf 宏定义 | [utils.rs](crates/nu-utils/src/utils.rs#L495-L515) |
| eval_config_contents + merge_env | [crates/nu-cli/src/config_files.rs](crates/nu-cli/src/config_files.rs) |
