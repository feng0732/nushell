# NuShell 标准库加载、命令注册与启动分析

## 一、核心结论速览

| 问题 | 答案 |
|---|---|
| 虚拟 std 路径和用户库路径是同一套优先级吗？ | **不是**。文件查找优先级：虚拟路径 > 当前目录 > NU_LIB_DIRS 常量 > NU_LIB_DIRS 环境变量 |
| 命令覆盖和模块查找是同一套优先级吗？ | **不是**。两套独立机制：文件查找 vs Overlay 注册顺序 |
| 用户模块能覆盖标准库模块吗？ | **文件查找层面不能**（虚拟路径优先级最高），但 **use/def 注册层面能**（后激活的 Overlay 优先级更高） |
| `%cmd` 为什么总能找到内置命令？ | 它不通过 Overlay 查找，直接遍历所有 Decl 找 `CommandType::Builtin` |
| env.nu 中 `$env.NU_LIB_DIRS = [...]` 能影响后续模块查找吗？ | **能**，但只作为常量未命中时的回退。常量优先，环境变量兜底 |

---

## 二、两套独立的优先级机制

```
┌───────────────────────────────────────────────────────────────┐
│                模块文件查找优先级                               │
│  （决定 "use std/assert" 从哪里读文件）                         │
│                                                               │
│  虚拟路径 > 当前目录 > $NU_LIB_DIRS 常量 > $env.NU_LIB_DIRS  │
│  （最高）                                   （最低）           │
└───────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────┐
│                命令/模块覆盖优先级                               │
│  （决定 "pwd" 调用哪个实现）                                    │
│                                                               │
│  用户 def/use > 标准库 prelude > 内置 Rust 命令                 │
│  （最高）                       （最低）                        │
└───────────────────────────────────────────────────────────────┘
```

两套机制的关系：文件查找决定**模块从哪里加载**，Overlay 机制决定**加载后同名命令谁生效**。

---

## 三、NU_LIB_DIRS 查找的两层回退机制

### 3.1 之前文档的错误

> ❌ 旧结论："env.nu 中 `$env.NU_LIB_DIRS = [...]` 不影响后续模块查找"

这个结论是**错误的**。实际上 NU_LIB_DIRS 有**常量优先 + 环境变量回退**两层机制，env.nu 修改的环境变量在回退层生效。

### 3.2 解析期查找：find_in_dirs（parse_source.rs）

[`find_in_dirs`](crates/nu-parser/src/parse_source.rs#L621-L741) 的完整逻辑：

```
find_in_dirs(filename, working_set, cwd, Some("NU_LIB_DIRS"))
  │
  ├─ [第一层] find_in_dirs_with_id()  ← L631-L690
  │   │
  │   ├─ 1. 虚拟路径精确匹配
  │   │     working_set.find_virtual_path(filename)
  │   │
  │   ├─ 2. 虚拟路径绝对路径匹配
  │   │     working_set.find_virtual_path(cwd/filename)
  │   │
  │   ├─ 3. 当前目录真实文件
  │   │     absolute_with(filename, cwd) && p.exists()
  │   │
  │   └─ 4. $NU_LIB_DIRS 常量遍历
  │         find_dirs_var → 只认 const_val → 遍历列表中的目录
  │
  └─ [第二层] find_in_dirs_old()       ← L692-L736（回退）
      │
      ├─ 1. 当前目录真实文件（与上面第3步重复，但用不同实现）
      │
      └─ 2. $env.NU_LIB_DIRS 环境变量遍历
            working_set.get_env_var("NU_LIB_DIRS") → 遍历列表中的目录
```

关键代码（[parse_source.rs#L738-L741](crates/nu-parser/src/parse_source.rs#L738-L741)）：

```rust
find_in_dirs_with_id(filename, working_set, cwd, dirs_var_name).or_else(|| {
    find_in_dirs_old(filename, working_set, cwd, dirs_var_name).map(ParserPath::RealPath)
})
```

**只有第一层全部未命中时，才会进入第二层回退**。

### 3.3 运行时查找：find_in_dirs_env（env.rs）

[`find_in_dirs_env`](crates/nu-engine/src/env.rs#L240-L299) 用于 `use`/`overlay use` 的 export-env 阶段，逻辑类似：

```
find_in_dirs_env(filename, engine_state, stack, dirs_var)
  │
  ├─ 选择 cwd：FILE_PWD 优先，否则 PWD
  │
  └─ check_dir() 两次：
      │
      ├─ 第一次：常量
      │   let lib_dirs = dirs_var.and_then(|var_id| engine_state.get_var(var_id).const_val.as_ref());
      │   check_dir(lib_dirs)
      │
      └─ 第二次：环境变量回退  ← L294-L298
          let lib_dirs_fallback = stack.get_env_var(engine_state, "NU_LIB_DIRS");
          check_dir(lib_dirs).or_else(|| check_dir(lib_dirs_fallback))
```

关键代码（[env.rs#L294-L298](crates/nu-engine/src/env.rs#L294-L298)）：

```rust
let lib_dirs = dirs_var.and_then(|var_id| engine_state.get_var(var_id).const_val.as_ref());
// TODO: remove (see #8310)
let lib_dirs_fallback = stack.get_env_var(engine_state, "NU_LIB_DIRS");

Ok(check_dir(lib_dirs).or_else(|| check_dir(lib_dirs_fallback)))
```

**注释 `TODO: remove (see #8310)` 表明环境变量回退是遗留兼容逻辑，未来可能移除**。

### 3.4 两层回退的生效条件

| 层 | 数据源 | 何时生效 | env.nu 能改吗 |
|---|---|---|---|
| 常量层 | `$NU_LIB_DIRS` 的 `const_val` | 常量列表中有路径命中时 | ❌ merge_env 不更新 const_val |
| 环境变量回退层 | `$env.NU_LIB_DIRS` | 常量未命中且环境变量列表中有路径命中时 | ✅ merge_env 更新 env_vars |

---

## 四、NU_LIB_DIRS 生命周期：按启动阶段逐帧分析

### 4.1 启动时序图（带代码行号和生效标注）

```
时间线 ──────────────────────────────────────────────────────────────►

[1] main.rs L354-415  初始设置
    │
    │  读取父进程环境变量 NU_LIB_DIRS → user_lib_dirs
    │  追加 -I 命令行参数 → user_lib_dirs
    │  追加默认路径 → all_lib_dirs = user_lib_dirs + defaults
    │
    │  同时写入两处（内容完全相同）：
    │    $env.NU_LIB_DIRS = all_lib_dirs   ← L397 add_env_var → engine_state.env_vars
    │    $NU_LIB_DIRS 常量 = all_lib_dirs  ← L411 set_variable_const_val → variables
    │                       ↓
    │              merge_delta 到 EngineState
    │
    ├─ 此时 常量 = 环境变量 = [用户路径..., <config>/scripts, <data>/completions]
    │  常量层已能覆盖所有默认路径，回退层几乎不会被触发
    │
[2] main.rs L423-425  标准库加载（解析期）
    │
    │  load_standard_library()
    │  ├─ 创建虚拟文件 std/, std-rfc/
    │  │  → 走 find_in_dirs_with_id 的第1-2步（虚拟路径），常量/环境变量都不参与
    │  └─ parse("use std/prelude *")
    │     → 虚拟路径命中 std/prelude，常量/环境变量都不参与
    │
    │  ⚠ 此阶段"use std/xxx"全部由虚拟路径命中，NU_LIB_DIRS 两层都不参与
    │
[3] main.rs L491      $nu 常量生成
    │
    │  generate_nu_constant()
    │
[4] config_files.rs  env.nu 执行（运行时）
    │
    │  eval_config_contents → eval_source → merge_env
    │  如果用户写了 $env.NU_LIB_DIRS = ["/my/libs"]
    │  → stack.env_vars 被修改 → merge_env 合并回 engine_state.env_vars ✅
    │  → engine_state.variables 中的 const_val 不变 ❌
    │
    ├─ 此时：
    │    常量 = [1] 的初始值（不变）
    │    环境变量 = 用户值（可能被 env.nu 覆盖）
    │
    │  env.nu 中 "use xxx" 的查找：
    │    解析期（parse_source.rs）：先常量 → 常量有默认路径，通常命中
    │    export-env 期（env.rs）：先常量 → 再环境变量回退
    │
[5] config_files.rs  config.nu 执行
    │  同 [4]，环境变量可再改，常量仍不变
    │
[6] 后续 REPL/脚本中的 "use xxx"
    │
    │  解析期查找（parse_source.rs）：
    │    常量层通常已包含默认路径 → 大多数情况在常量层命中
    │    只有常量层全部未命中时，才进入环境变量回退层
    │
    │  export-env 查找（env.rs）：
    │    同上：常量优先，环境变量回退
    │
    │  如果 env.nu 增加了新路径（如 /my/libs）且常量层没有该路径：
    │    → 常量层未命中 → 环境变量回退层命中 → ✅ 生效
```

### 4.2 关键代码证据

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

**解析期 find_dirs_var 只认常量**（[parse_source.rs#L615-L618](crates/nu-parser/src/parse_source.rs#L615-L618)）：

```rust
pub fn find_dirs_var(working_set: &StateWorkingSet, var_name: &str) -> Option<VarId> {
    working_set
        .find_variable(format!("${var_name}").as_bytes())
        .filter(|var_id| working_set.get_variable(*var_id).const_val.is_some())
}
```

但 find_in_dirs 整体通过 `find_in_dirs_old` 提供环境变量回退（L738-L741）。

**运行时 find_in_dirs_env 显式两层**（[env.rs#L294-L298](crates/nu-engine/src/env.rs#L294-L298)）：

```rust
let lib_dirs = dirs_var.and_then(|var_id| engine_state.get_var(var_id).const_val.as_ref());
// TODO: remove (see #8310)
let lib_dirs_fallback = stack.get_env_var(engine_state, "NU_LIB_DIRS");
Ok(check_dir(lib_dirs).or_else(|| check_dir(lib_dirs_fallback)))
```

### 4.3 三张表的对比（已修正）

| 属性 | `$env.NU_LIB_DIRS` | `$NU_LIB_DIRS` 常量 | 虚拟路径 |
|---|---|---|---|
| 存储 | `engine_state.env_vars` | `engine_state.variables[var_id].const_val` | `engine_state.virtual_paths` |
| 设置时机 | main.rs 初始 + env.nu 可覆盖 | main.rs 初始，之后不变 | load_standard_library 时注册 |
| env.nu 能改吗 | ✅ 通过 merge_env | ❌ merge_env 不涉及常量 | ❌ 不受影响 |
| 解析期模块查找 | 回退层（常量未命中时） | 主层（优先查找） | 最优先（命中后不查 NU_LIB_DIRS） |
| 运行时 export-env 查找 | 回退层（常量未命中时） | 主层（优先查找） | 不参与（find_in_dirs_env 不查虚拟路径） |
| 查找优先级 | 最低（兜底） | 中 | 最高 |

### 4.4 对用户的影响（已修正）

如果用户在 env.nu 中写了 `$env.NU_LIB_DIRS = ["/my/libs"]`：
- ✅ `$env.NU_LIB_DIRS` 会变为 `["/my/libs"]`
- ✅ `use my_module` **能**在 `/my/libs` 中查找——如果常量层未命中，回退层会使用环境变量
- ⚠ 但如果常量层已包含同名目录（如默认路径中有匹配），则环境变量层的路径**不会**被用到
- ❌ `$NU_LIB_DIRS` 常量仍是初始值

**实际效果**：因为初始设置时常量和环境变量内容相同，常量层已有默认路径，env.nu 新增的路径只有在常量层找不到模块时才会在回退层生效。典型场景：用户添加了一个新的库目录，模块名在常量的默认路径中不存在 → 回退层命中用户新增目录 → 生效。

---

## 五、模块文件查找优先级详解

### 5.1 完整查找顺序（带代码证据，已修正）

解析期 [`find_in_dirs`](crates/nu-parser/src/parse_source.rs#L621-L741) 的完整优先级：

```
优先级从高到低：

[find_in_dirs_with_id — 主层]
1. 虚拟路径精确匹配
   working_set.find_virtual_path(filename) → 命中直接返回

2. 虚拟路径绝对路径匹配
   working_set.find_virtual_path(cwd/filename) → 命中返回

3. 当前目录真实文件
   absolute_with(filename, cwd) && p.exists()

4. $NU_LIB_DIRS 常量遍历（按列表顺序，先到先得）
   find_dirs_var → const_val.as_list → 逐目录检查

[find_in_dirs_old — 回退层，仅当主层全部未命中]
5. 当前目录真实文件（重复检查，不同实现）

6. $env.NU_LIB_DIRS 环境变量遍历（按列表顺序，先到先得）
   working_set.get_env_var("NU_LIB_DIRS").as_list → 逐目录检查
```

运行时 [`find_in_dirs_env`](crates/nu-engine/src/env.rs#L240-L299) 的完整优先级：

```
优先级从高到低：

1. 当前目录或 FILE_PWD 下的真实文件
   absolute_with(filename, cwd) && path.exists()

2. $NU_LIB_DIRS 常量遍历
   engine_state.get_var(var_id).const_val → check_dir

3. $env.NU_LIB_DIRS 环境变量遍历（回退，标注 TODO: remove）
   stack.get_env_var("NU_LIB_DIRS") → check_dir
```

**注意**：`find_in_dirs_env` **不查虚拟路径**。运行时 export-env 阶段不需要虚拟文件系统，因为它只是定位已解析模块的源文件路径。

### 5.2 NU_LIB_DIRS 初始设置逻辑

在 [main.rs#L342-L415](src/main.rs#L342-L415) 中：

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

### 5.3 虚拟路径的反向遍历

[`find_virtual_path`](crates/nu-protocol/src/engine/state_working_set.rs#L1045-L1062) 从后往前遍历：

```rust
for (virtual_name, virtual_path) in self.delta.virtual_paths.iter().rev() { ... }
for (virtual_name, virtual_path) in self.permanent_state.virtual_paths.iter().rev() { ... }
```

**后注册的虚拟路径优先级更高**。delta 中的虚拟路径优先于 permanent_state 中的同名路径。

---

## 六、命令/模块覆盖优先级（Overlay 机制）

### 6.1 Overlay 激活 = 优先级提升

[`add_overlay`](crates/nu-protocol/src/engine/state_working_set.rs#L922-L967)：

```rust
last_scope_frame.active_overlays.retain(|id| id != &overlay_id);
last_scope_frame.active_overlays.push(overlay_id);
```

**每次 `use` 或 `overlay use` 都会把对应 overlay 移到 active_overlays 末尾**，而 `find_decl` 从后往前查，因此后激活的优先级更高。

### 6.2 同名覆盖的完整场景

| 场景 | 行为 | 代码位置 |
|---|---|---|
| 同一 Overlay 内注册同名 decl | HashMap::insert，后者覆盖前者 | [add_decl](crates/nu-protocol/src/engine/state_working_set.rs#L115-L125) |
| 不同 Overlay 同名 decl | 后激活的 Overlay 先被查到 | [find_decl](crates/nu-protocol/src/engine/state_working_set.rs#L443-L477) |
| 用户 `def pwd` 覆盖内置 `pwd` | ✅ 可以覆盖 | 同上 |
| 标准库 prelude `pwd` 覆盖内置 `pwd` | ✅ 可以覆盖 | 同上 |
| 用户磁盘同名模块覆盖 std 模块 | ❌ 不能，文件查找时虚拟路径优先级更高 | [find_in_dirs](crates/nu-parser/src/parse_source.rs#L631-L690) |

### 6.3 `%` 前缀强制内置：绕过 Overlay

[`find_builtin_decl`](crates/nu-engine/src/scope.rs#L595-L609) 直接遍历全部 `decls` 数组，跳过非 `CommandType::Builtin`，不走 Overlay 查找。

### 6.4 Visibility：隐藏不是删除

`hide` 命令通过 `Visibility.decl_ids` 的布尔值标记隐藏，不修改 `decls` HashMap。

### 6.5 export-env 阶段

`use` 命令执行时分为两个阶段：

```
use std/assert
  │
  ├─ [解析期] parse_use → find_in_dirs → 定位模块文件 → 解析内容 → 注册 overlay
  │
  └─ [运行时] export-env → find_in_dirs_env → 定位源文件路径 → 执行 env_block
              （如果模块有 export-env {} 块，在此阶段执行）
```

export-env 使用 `find_in_dirs_env`（不查虚拟路径），用常量+环境变量两层回退定位文件。对于虚拟文件（如 std 模块），export-env 阶段的文件路径是通过解析期记录的 span 推导的，不依赖运行时路径查找。

---

## 七、可迁移链接（ParserPath 抽象）

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

## 八、启动成本（只含可观察证据）

### 8.1 perf 埋点机制

[`perf!` 宏](crates/nu-utils/src/utils.rs#L495-L515) 通过 `log::info!` 输出耗时，需配合 `--log-level info` 使用：

```rust
macro_rules! perf {
    ($msg:expr, $dur:expr, $use_color:expr) => {
        log::info!("perf: {}:{}:{} {} took {:?}", file!(), line!(), column!(), $msg, $dur.elapsed());
    };
}
```

### 8.2 main.rs 中的 perf 埋点

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

### 8.3 run.rs / config_files.rs 中的 perf 埋点

| 埋点名称 | 代码位置 | 包含的操作 |
|---|---|---|
| `read plugins` | [run.rs#L40](src/run.rs#L40) | 插件签名文件读取 |
| `read env.nu` | [run.rs#L57](src/run.rs#L57) | 环境配置脚本执行 |
| `read config.nu` | [run.rs#L74](src/run.rs#L74) | 用户配置脚本执行 |
| `read login.nu` | [run.rs#L82](src/run.rs#L82) | 登录 shell 配置（仅 login 模式） |
| `setup_config` | [run.rs#L209](src/run.rs#L209) | REPL 模式完整配置 |
| `evaluate_commands` | [run.rs#L103](src/run.rs#L103) | 实际命令执行 |
| `evaluate_file` | [run.rs#L175](src/run.rs#L175) | 脚本文件执行 |
| `evaluate_repl` | [run.rs#L219](src/run.rs#L219) | REPL 主循环启动 |

### 8.4 eval_source 中的 perf 埋点

每个 `eval_source` 调用（包括 env.nu/config.nu 的执行）都产生一条 `eval_source <fname>` 埋点（[util.rs#L270-L277](crates/nu-cli/src/util.rs#L270-L277)）。

### 8.5 load_standard_library 无单独埋点

`load_standard_library` 在 [main.rs#L423-L425](src/main.rs#L423-L425) 调用，夹在 `$env.NU_LIB_DIRS/$NU_LIB_DIRS setup`（L415）和 `run test_bins`（L476）之间，没有自己的 perf 埋点。如需单独测量，可观察这两个相邻埋点的时间差，或添加埋点后重新编译。

### 8.6 标准库加载的可观察特征

| 操作 | 可观察性 | 代码证据 |
|---|---|---|
| `create_virt_file` × N | 编译期嵌入 `include_str!`，运行时只有 `add_file` + `add_virtual_path` | [crates/nu-std/src/lib.rs#L11-L16](crates/nu-std/src/lib.rs#L11-L16) |
| `create_virt_dir` × M | 纯内存 `Vec::push` | 同上 |
| `use std/prelude *` | 需要 `parse()` + 执行导入，是 stdlib 加载的主要耗时操作 | [crates/nu-std/src/lib.rs#L173-L189](crates/nu-std/src/lib.rs#L173-L189) |
| 子模块（assert, math 等） | 启动时不解析，只有 `use` 时才按需解析 | [parse_module_file](crates/nu-parser/src/parse_module.rs#L706-L758) 有缓存 |

### 8.7 模块缓存

```rust
// parse_module_file 中的缓存判断
if let Some(module_id) = working_set.find_module_by_span(new_span)
    && !module_needs_reloading(working_set, module_id)
{
    return Some(module_id);  // 直接复用，不重复解析
}
```

虚拟文件内容不变（编译期嵌入），永远不会触发 `module_needs_reloading`。

### 8.8 性能保障设计

1. **`include_str!` 编译期嵌入**：零运行时 I/O
2. **按需解析**：只有 `use` 到的模块才解析内容
3. **模块缓存**：基于 Span，重复引用零成本
4. **`Arc` 共享**：decls/blocks/modules 用 `Arc<Vec>` 包装
5. **`StateDelta` 增量**：成功后一次性 merge，失败直接丢弃

---

## 九、标准库加载完整流程

### 9.1 启动顺序总览

```
1. EngineState::new()
   ↓ 空引擎状态
2. add_command_context()  → 注册所有 Rust 内置命令到 root overlay
   ↓ 数百个 Builtin 命令就绪
3. 设置 NU_LIB_DIRS 常量 + 环境变量（内容相同）
   ↓ perf 点：$env.NU_LIB_DIRS/$NU_LIB_DIRS setup
4. load_standard_library()  ← 无单独 perf 点
   ├─ 创建 std/ 虚拟文件树（include_str! 嵌入，走虚拟路径查找）
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
   ↓ 后续 "use xxx" 查找：常量优先 → 环境变量回退
```

### 9.2 标准库加载的两个层面

| 层面 | 内容 | 时机 | 能否被覆盖 |
|---|---|---|---|
| 虚拟文件系统 | std/ 和 std-rfc/ 的所有 .nu 文件 | `load_standard_library` 时注册 | 文件查找层面：不能（虚拟路径优先级最高） |
| Prelude 命令 | banner, pwd 等自定义命令 | `use std/prelude *` 时注册到 Overlay | Overlay 层面：能（用户后定义的优先级更高） |

---

## 十、文档来源

### 10.1 内置命令文档

来自 Rust `Command` trait：`name()`, `signature()`, `description()`, `extra_description()`, `examples()`, `search_terms()`, `attributes()`。

### 10.2 自定义命令/模块文档

从源码注释中提取，由 [`build_desc`](crates/nu-protocol/src/engine/description.rs#L38-L91) 处理：
- 去掉每行开头的 `#` 和对齐空格
- 第一段空行前的内容 = brief description
- 其余内容 = extra description

### 10.3 help 命令查找顺序

```
help <name>
  ├─ alias 查找
  ├─ command 查找（走 Overlay 优先级）
  └─ module 查找

help %<name>  → 强制内置（绕过 Overlay）
```

---

## 十一、关键代码索引

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
| 解析期模块查找（两层回退） | [parse_source.rs](crates/nu-parser/src/parse_source.rs#L615-L741) |
| 运行时模块查找（常量+回退） | [env.rs](crates/nu-engine/src/env.rs#L240-L299) |
| 模块解析与缓存 | [parse_module.rs](crates/nu-parser/src/parse_module.rs) |
| ParserPath 抽象 | [parser_path.rs](crates/nu-protocol/src/parser_path.rs) |
| Command trait 与类型 | [command.rs](crates/nu-protocol/src/engine/command.rs) |
| 文档构建 | [description.rs](crates/nu-protocol/src/engine/description.rs) |
| help 命令实现 | [help_.rs](crates/nu-command/src/help/help_.rs) |
| perf 宏定义 | [utils.rs](crates/nu-utils/src/utils.rs#L495-L515) |
| use 命令（export-env 阶段） | [use_.rs](crates/nu-cmd-lang/src/core_commands/use_.rs) |
