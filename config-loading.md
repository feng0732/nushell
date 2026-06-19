# Nushell 配置文件加载顺序与默认值合并机制

本文档从代码实现角度，深入梳理 Nushell 的配置加载流程、默认值合并策略以及容错启动机制。

---

## 1. 配置文件的四种层级

Nushell 为每个配置类型（环境配置 `env.nu` 和主配置 `config.nu`）都准备了**四个层级**的文件，各自承担不同职责。定义见 `ConfigFileKind` 枚举（crates/nu-utils/src/utils.rs#L90-L146）。

| 层级 | 文件名模板 | 作用 | 加载时机 |
|------|-----------|------|---------|
| **默认配置** | `default_env.nu` / `default_config.nu` | 编译时嵌入的内置默认值，保证最小可用环境 | `default_env.nu` **每次**正常启动都加载；`default_config.nu` 仅 REPL 模式、或命令/脚本模式指定 `--config`/`--login` 时加载 |
| **脚手架配置** | `scaffold_env.nu` / `scaffold_config.nu` | 首次启动时写入用户配置目录的模板文件（仅注释无代码） | 首次启动**创建**用户配置时 |
| **文档配置** | `doc_env.nu` / `doc_config.nu` | 带完整注释的文档，供 `config env --doc` 查看 | 从不自动加载，仅用户主动查阅 |
| **用户配置** | `env.nu` / `config.nu` | 用户自定义配置（位于 `$nu.config-path` 目录） | 正常启动时**在默认配置之后**加载 |

> 文件目录：crates/nu-utils/src/default_files/
> 设计说明：crates/nu-utils/src/default_files/README.md

`ConfigFileKind` 将三种编译时文件（默认、脚手架、文档）通过 `include_str!` 嵌入二进制；用户配置文件（`env.nu` / `config.nu`）**不内嵌**，由 `path()` 方法仅返回文件名字符串，运行时从文件系统读取：

```rust
// crates/nu-utils/src/utils.rs#L97-L131
impl ConfigFileKind {
    // 以下三种通过 include_str! 编译嵌入
    pub const fn default(self) -> &'static str { ... }   // default_env.nu / default_config.nu
    pub const fn scaffold(self) -> &'static str { ... }  // scaffold_env.nu / scaffold_config.nu
    pub const fn doc(self) -> &'static str { ... }       // doc_env.nu / doc_config.nu

    // 用户配置文件——仅返回文件名字符串，运行时从磁盘读取
    pub const fn path(self) -> &'static str {
        match self {
            ConfigFileKind::Config => "config.nu",
            ConfigFileKind::Env => "env.nu",
        }
    }
}
```

---

## 2. `$env.config` 从赋值到生效的完整数据通路

这是理解"默认值合并"最核心的一条链路。配置文件中写 `$env.config.foo = bar` 时，值是如何最终写到 engine_state 中的？

### 2.1 三步机制

```
① eval 引擎执行 $env.config.xxx = yyy
   │
   │  触发位置：nu-engine/src/eval_ir.rs#L517-L523
   │  当赋值目标的环境变量名为 "config" 时：
   │    stack.add_env_var("config", value)
   │    stack.update_config(engine_state)  ← 立即将 $env.config 解析为 Config 结构体
   │
   ▼
② stack.update_config() 把 $env.config Value → Config 结构体
   │
   │  实现位置：nu-protocol/src/engine/stack.rs#L226-L245
   │  步骤：
   │    a. 从 stack 环境变量中取出 $env.config 的 Value
   │    b. 调用 config.update_from_value_with_options(&old, value, ...)
   │       → 逐字段匹配，成功覆盖、失败跳过并记录错误
   │    c. 将更新后的 Config 再次写回 $env.config（确保 Value 与 Config 同步）
   │    d. 存入 stack.config = Some(Arc<Config>)
   │
   ▼
③ merge_env() 将 stack 中的 Config 刷入 engine_state
   │
   │  调用位置：每份配置文件执行完毕后
   │    - eval_default_config() 结尾（src/config_files.rs#L228-L231）
   │    - eval_config_contents() 结尾（crates/nu-cli/src/config_files.rs#L258-L260）
   │    - read_default_env_file() 结尾（src/config_files.rs#L155-L157）
   │
   │  实现位置：nu-protocol/src/engine/engine_state.rs#L365-L392
   │  关键步骤：
   │    a. 将 stack.env_vars 中所有 pending 环境变量 drain 后 merge 到 engine_state
   │    b. if let Some(config) = stack.config.take() {
   │           self.config = config;  ← 直接替换整个 engine_state.config
   │       }
   │
   ▼
  engine_state.config 更新完毕，后续 get_config() 读取新值
```

### 2.2 关键推论：`merge_env()` 中的 config 是**整体替换**而非逐字段合并

```rust
// nu-protocol/src/engine/engine_state.rs#L382-L384
//   stack.config: Option<Arc<Config>>  →  take() 解包为 Arc<Config>
//   self.config: Arc<Config>           →  赋值一侧也是 Arc<Config>
if let Some(config) = stack.config.take() {
    self.config = config;   // Arc<Config> 引用替换，零拷贝
}
```

这意味着：
- 如果用户 `config.nu` 中写了 `$env.config.color_config = { ... }`，`update_config()` 会将用户提供的完整 color_config HashMap 替换旧的，**不是**逐键合并
- `update_from_value()`（crates/nu-protocol/src/config/mod.rs#L155-L253）虽然逐字段处理，但每个字段内部是"有则覆盖，无则保留默认"——用户只设了部分子字段时，Rust 默认值兜底

---

## 3. 启动初始化（main.rs 阶段）

在加载任何配置文件之前，src/main.rs 会进行一系列**内置初始化**，这些值是配置系统的"第零层"默认值。

### 3.1 初始化流程时间线

```
main() 启动
  │
  ├─ ① 设置 NU_PLUGIN_DIRS 常量（src/main.rs#L185-L201）
  │     默认值：[配置目录/plugins, 可执行文件所在目录]
  │
  ├─ ② 设置 $env.config = Config::default()（src/main.rs#L316-L319）
  │     使用 Rust 层的 Config 结构体默认值
  │
  ├─ ③ 设置 $env.ENV_CONVERSIONS = {}（src/main.rs#L322-L325）
  │
  ├─ ④ gather_parent_env_vars() — 继承父进程环境变量（src/main.rs#L329）
  │
  ├─ ⑤ convert_env_values() — 将字符串环境变量转为 Value 类型（src/main.rs#L337-L339）
  │
  ├─ ⑥ 设置 NU_LIB_DIRS（src/main.rs#L343-L414）
  │     组合顺序：用户 NU_LIB_DIRS + -I 参数 + 默认 scripts 目录 + completions 目录
  │
  ├─ ⑦ 设置 NU_VERSION
  │
  ├─ ⑧ load_standard_library() — 加载标准库
  │
  └─ ⑨ generate_nu_constant() — 生成 $nu 常量
```

### 3.2 Rust 层 Config 默认值

所有通过 `$env.config` 访问的配置项，在 Rust 层都有对应的硬编码默认值。定义在 crates/nu-protocol/src/config/mod.rs#L94-L153：

| 配置项 | Rust 默认值 | 说明 |
|-------|------------|------|
| `recursion_limit` | 50 | 递归调用深度限制 |
| `float_precision` | 2 | 浮点数显示精度 |
| `footer_mode` | `RowCount(25)` | 超过25行显示表尾 |
| `use_ansi_coloring` | `Auto` | 自动检测终端颜色支持 |
| `show_hints` | `true` | 显示自动补全提示 |
| `bracketed_paste` | `true` | 启用括号粘贴模式 |
| `error_lines` | 1 | 错误上下文行数 |
| `auto_cd_implicit` | `false` | 隐式自动 cd |
| `color_config` | `{}` | **空 HashMap，实际默认值在 default_config.nu 中设置** |

---

## 4. 三种运行模式下的配置加载差异

根据启动方式不同（REPL / 命令执行 / 脚本执行），配置加载的完整度有显著区别。核心分发逻辑在 src/main.rs#L598-L661，具体加载流程分别在 src/run.rs 中。

### 4.1 模式对比表

| 加载项 | REPL 交互模式 | `nu -c "commands"` 命令模式 | `nu script.nu` 脚本模式 |
|--------|:------------:|:--------------------------:|:---------------------:|
| 插件文件 `plugin.msgpackz` | ✅ | ✅ | ✅ |
| `default_env.nu` | ✅ | ✅ | ✅ |
| **用户 `env.nu`** | ✅ | ⚠️ 仅 `--env-config` 或 `--login` | ⚠️ 仅 `--env-config` |
| `default_config.nu` | ✅ | ⚠️ 仅 `--config` 或 `--login` | ⚠️ 仅 `--config` |
| **用户 `config.nu`** | ✅ | ⚠️ 仅 `--config` 或 `--login` | ⚠️ 仅 `--config` |
| `login.nu` | ⚠️ 仅 `--login` | ⚠️ 仅 `--login` | ❌ |
| autoload 目录 | ✅ | ❌ | ❌ |

### 4.2 三个模式入口函数的代码逐行对照

#### REPL 模式 — `run_repl()`（src/run.rs#L183-L222）

REPL 模式将全部配置加载委托给 `setup_config()`，最完整的路径：

```rust
// src/run.rs#L192-L201
if parsed_nu_cli_args.no_config_file.is_none() {
    setup_config(
        engine_state, &mut stack,
        plugin_file,          // 插件
        config_file,          // --config
        env_file,             // --env-config
        login_shell.is_some(), // --login
    );
}
```

`setup_config()` 内部（src/config_files.rs#L234-L282）按顺序调用：
1. `read_plugin_file()` — 插件签名
2. `read_config_file(Env)` — default_env.nu + 用户 env.nu
3. `read_config_file(Config)` — default_config.nu + 用户 config.nu
4. `read_loginshell_file()` — login.nu（仅 `is_login_shell == true`）
5. `read_vendor_autoload_files()` — autoload 目录（始终执行）

#### 命令模式 — `run_commands()`（src/run.rs#L17-L109）

**不会调用 `setup_config()`**，而是自行编排加载流程：

```rust
// src/run.rs#L36-L83
if parsed_nu_cli_args.no_config_file.is_none() {
    // 1. 插件
    read_plugin_file(engine_state, ...);

    // 2. env — 仅当 --env-config 或 --login 时加载用户 env.nu，否则只读 default_env.nu
    if env_file.is_some() || login_shell.is_some() {
        read_config_file(..., Env, ..., strict_mode=true);  // ← strict_mode = true！
    } else {
        read_default_env_file(engine_state, &mut stack)     // ← 只加载 default_env.nu
    }

    // 3. config — 仅当 --config 或 --login 时加载
    if config_file.is_some() || login_shell.is_some() {
        read_config_file(..., Config, ..., strict_mode=true);
    }

    // 4. login.nu — 仅 --login
    if login_shell.is_some() {
        read_loginshell_file(engine_state, &mut stack, false);
    }

    // 5. ⚠️ 没有 read_vendor_autoload_files()！autoload 在此模式不加载
}
```

**关键差异**：
- `--login` 标志在命令模式下**会触发** env.nu + config.nu + login.nu
- `strict_mode = true`：配置文件执行出错会 `exit(1)` 终止进程
- **没有 autoload**，即使在 `--login` 模式下也没有

#### 脚本模式 — `run_file()`（src/run.rs#L111-L181）

**也不会调用 `setup_config()`**，逻辑比命令模式更精简：

```rust
// src/run.rs#L127-L162
if parsed_nu_cli_args.no_config_file.is_none() {
    // 1. 插件
    read_plugin_file(engine_state, ...);

    // 2. env — 仅当 --env-config 时加载，--login 不触发！
    if env_file.is_some() {
        read_config_file(..., Env, ..., strict_mode=true);
    } else {
        read_default_env_file(engine_state, &mut stack)
    }

    // 3. config — 仅当 --config 时加载，--login 不触发！
    if config_file.is_some() {
        read_config_file(..., Config, ..., strict_mode=true);
    }

    // 4. ⚠️ 没有 read_loginshell_file()！login.nu 在脚本模式永远不加载
    // 5. ⚠️ 没有 read_vendor_autoload_files()！autoload 也不加载
}
```

**关键差异**：
- `--login` 标志在脚本模式下**不会触发** env.nu / config.nu / login.nu 中的任何一项
- 只有显式 `--env-config` / `--config` 参数才会加载用户配置
- `strict_mode = true`：出错会终止进程

---

## 5. `read_config_file()` 核心函数详解

此函数是配置加载的中枢，定义在 src/config_files.rs#L24-L110。

### 5.1 函数签名

```rust
pub(crate) fn read_config_file(
    engine_state: &mut EngineState,
    stack: &mut Stack,
    config_file: Option<Spanned<String>>,   // 命令行 --env-config / --config 指定的路径
    config_kind: ConfigFileKind,            // Env 或 Config
    create_scaffold: bool,                  // 是否在首次启动时创建脚手架文件
    strict_mode: bool,                      // 严格模式：出错是否 exit(1)
)
```

### 5.2 分支逻辑

```
read_config_file():
  │
  ├─ 第一步（L34）：无条件执行 eval_default_config()
  │   加载对应类型的 default_*.nu（编译嵌入）
  │   → eval_source() 执行
  │   → merge_env(stack) 将变更刷入 engine_state
  │
  └─ 第二步（L39-L109）：加载用户配置（二选一）
       │
       ├─ A) 命令行显式指定了配置文件路径（L39-L57）
       │     ├─ 解析绝对路径，检查文件是否存在
       │     ├─ 存在 → eval_config_contents()
       │     └─ 不存在 → 报告 FileNotFound 错误
       │           └─ 严格模式下 exit(1)
       │
       └─ B) 使用默认配置目录路径（L58-L109）
             ├─ 若目录不存在 → 创建目录（L60-L65）
             ├─ 构造路径：配置目录 / config_kind.path()（L67）
             │           即 env.nu 或 config.nu
             ├─ 若文件不存在（L69-L106）：
             │     └─ create_scaffold 为 true 时，写入 scaffold_*.nu 模板
             │        写入失败则 return，不再 eval（L92-L96）
             └─ eval_config_contents(用户配置路径)（L108）
                   → eval_source() 执行文件
                   → merge_env(stack)
```

### 5.3 `eval_config_contents()` — 单个文件的执行与合并

定义在 crates/nu-cli/src/config_files.rs#L227-L263：

```rust
pub fn eval_config_contents(
    config_path: PathBuf,
    engine_state: &mut EngineState,
    stack: &mut Stack,
    strict_mode: bool,
) {
    if config_path.exists() & config_path.is_file() {
        let prev_file = engine_state.file.take();
        engine_state.file = Some(config_path.clone());  // 用于错误报告

        let exit_code = eval_source(engine_state, stack, &contents, &config_filename, ...);
        if exit_code != 0 && strict_mode {
            std::process::exit(exit_code)  // 严格模式：出错即死
        }

        engine_state.file = prev_file;  // 恢复

        // ← 关键：将 stack 中所有 env 变更（含 $env.config）刷入 engine_state
        engine_state.merge_env(stack);
    }
}
```

**核心要点**：每个配置文件执行后立即 `merge_env(stack)`，这意味着前一份配置的写入结果会持久化到 engine_state，下一份配置在其基础上叠加。这就是"后加载覆盖先加载"的实现基础。

---

## 6. 默认值合并的层级与覆盖顺序

### 6.1 `$env.config` 完整合并路径（按代码执行顺序）

以下按**实际代码执行顺序**排列，每一步执行后都会调用 `merge_env()` 写入 engine_state：

```
步骤 0: Rust Config::default()
         │  src/main.rs#L316-L319
         │  此时 $env.config = { color_config: {}, ... }
         │  合并：写入 engine_state.config
         ▼
步骤 1: default_env.nu
         │  设置 PROMPT_COMMAND、PROMPT_COMMAND_RIGHT
         │  此文件不修改 $env.config
         │  合并：写入 engine_state.env_vars（PROMPT 相关）
         ▼
步骤 2: 用户 env.nu
         │  用户可在此设置 ENV_CONVERSIONS、PATH 等
         │  此文件可修改 $env.config（但不常见）
         │  合并：写入 engine_state
         ▼
步骤 3: default_config.nu
         │  设置 $env.config.color_config = { 完整配色表 }
         │  ⚠️ 此步骤在用户 env.nu 之后！
         │  如果用户 env.nu 已经设置了 color_config，会被 default_config.nu 覆盖
         │  合并：写入 engine_state.config
         ▼
步骤 4: 用户 config.nu
         │  用户自定义 $env.config.* 覆盖
         │  合并：写入 engine_state.config
         ▼
步骤 5: login.nu（仅 --login 启动时）
         │  可修改 $env.config 或环境变量
         │  合并：写入 engine_state
         ▼
步骤 6: autoload 目录脚本（仅 REPL 模式）
         │  每个 .nu 文件执行后各自 merge_env
         │  可修改 $env.config 或环境变量
         │  合并：写入 engine_state
         ▼
   最终生效的 $env.config 和环境变量
```

### 6.2 容易混淆的关键细节

#### `default_config.nu` 在用户 `env.nu` 之后加载

看 `read_config_file()` 的调用顺序（src/config_files.rs#L253-L268）：

```rust
// 先加载 env 整条链（default_env.nu → 用户 env.nu）
read_config_file(engine_state, stack, env_file, ConfigFileKind::Env, ...);
// 再加载 config 整条链（default_config.nu → 用户 config.nu）
read_config_file(engine_state, stack, config_file, ConfigFileKind::Config, ...);
```

而 `read_config_file()` 内部（src/config_files.rs#L34）**先**执行 `eval_default_config()`，**再**执行用户文件。所以完整的 env → config 交叉顺序是：

```
default_env.nu → 用户 env.nu → default_config.nu → 用户 config.nu
```

**这意味着**：如果用户在 `env.nu` 中设置了 `$env.config.color_config`，该值会被随后加载的 `default_config.nu` 覆盖为内置配色。正确做法是将 `$env.config` 的修改放在 `config.nu` 中。

#### `login.nu` 和 autoload 都能覆盖 `config.nu` 中设置的主配置

按代码顺序（src/config_files.rs#L270-L274）：

```rust
// config.nu 加载完毕后
if is_login_shell {
    read_loginshell_file(engine_state, stack, false);  // ④ login.nu
}
read_vendor_autoload_files(engine_state, stack);       // ⑤ autoload
```

- `login.nu` 在 `config.nu` **之后**执行，可以对 `$env.config` 做最终覆盖
- autoload 目录又在 `login.nu` **之后**执行，是真正的"最后一公里"
- 三者都通过 `eval_config_contents()` → `merge_env()` 写回 engine_state

**实际影响**：
- `login.nu` 适合做"仅登录会话"的配置覆盖（如设置不同提示符）
- autoload 适合做"所有 REPL 会话"的最终补丁（如管理员全局覆盖）
- autoload 目录的加载顺序是 vendor → user，user 目录后加载可覆盖 vendor

### 6.3 环境变量（除 $env.config 外）合并路径

```
步骤 0: main.rs 内置设置
         ├─ NU_PLUGIN_DIRS 常量
         ├─ NU_LIB_DIRS（用户值 + 默认值 + -I 参数）
         ├─ NU_VERSION
         └─ PROMPT_INDICATOR 等 REPL 专用变量
         │
         ▼
步骤 1: gather_parent_env_vars()       ← 继承父进程（如 PATH, HOME 等）
         │  src/main.rs#L329
         ▼
步骤 2: default_env.nu                 ← PROMPT_COMMAND, PROMPT_COMMAND_RIGHT
         ▼
步骤 3: 用户 env.nu                    ← 用户自定义环境变量
         ▼
步骤 4: default_config.nu              ← 仅影响 $env.config 内部
         ▼
步骤 5: 用户 config.nu                 ← 可进一步修改 $env
         ▼
步骤 6: login.nu（--login 时）
         ▼
步骤 7: autoload 目录脚本（REPL 时）
         ▼
   最终生效的环境变量
```

---

## 7. `login.nu` 覆盖主配置的代码追踪

`login.nu` 的加载入口在 `read_loginshell_file()`（src/config_files.rs#L112-L134）：

```rust
pub(crate) fn read_loginshell_file(
    engine_state: &mut EngineState,
    stack: &mut Stack,
    strict_mode: bool,
) {
    if let Some(mut config_path) = nu_path::nu_config_dir() {
        config_path.push(LOGINSHELL_FILE);  // "login.nu"
        if config_path.exists() {
            eval_config_contents(config_path.into(), engine_state, stack, strict_mode);
            // ↑ 内部调用 merge_env(stack)，将 login.nu 的所有变更写回
        }
    }
}
```

### 7.1 调用时机——三种模式的差异

| 模式 | 代码位置 | `--login` 是否触发 | `strict_mode` |
|------|---------|:------------------:|:-------------:|
| REPL | src/config_files.rs#L271 | ✅ `is_login_shell` 控制 | false |
| 命令 | src/run.rs#L78-L80 | ✅ 直接判断 `login_shell.is_some()` | false |
| 脚本 | src/run.rs#L111-L181 | ❌ 无此调用 | — |

### 7.2 login.nu 覆盖 config.nu 的具体机制

login.nu 与 config.nu 使用**完全相同的执行路径**：`eval_config_contents()` → `eval_source()` → `merge_env()`。

如果 login.nu 中写了：

```nu
$env.config.show_banner = never
$env.PROMPT_COMMAND = { || "login> " }
```

执行链路：
1. `eval_source()` 执行 login.nu
2. 赋值 `$env.config.show_banner = never` → 触发 `stack.update_config()` → stack.config 被更新
3. 赋值 `$env.PROMPT_COMMAND = ...` → stack.env_vars 中记录 PROMPT_COMMAND
4. `merge_env(stack)` → engine_state.config 整体替换为 stack 中的新值 + engine_state.env_vars 合并

**结果**：login.nu 中对 `$env.config` 的任何修改都会完全覆盖 config.nu 中同名设置。

---

## 8. autoload 覆盖主配置的代码追踪

autoload 目录的加载入口在 `read_vendor_autoload_files()`（src/config_files.rs#L178-L211）：

```rust
pub(crate) fn read_vendor_autoload_files(
    engine_state: &mut EngineState,
    stack: &mut Stack,
) {
    get_vendor_autoload_dirs(engine_state)   // ① 先遍历 vendor 目录
        .iter()
        .chain(get_user_autoload_dirs(engine_state).iter())  // ② 再遍历 user 目录
        .for_each(|autoload_dir| {
            if autoload_dir.exists() {
                let entries = read_and_sort_directory(autoload_dir);  // ③ 字典序排序
                if let Ok(entries) = entries {
                    for entry in entries {
                        if !entry.ends_with(".nu") { continue; }
                        let path = autoload_dir.join(entry);
                        eval_config_contents(path, engine_state, stack, false);  // ④ 逐个执行
                        // ↑ 每个 .nu 文件执行后都会 merge_env
                    }
                }
            }
        });
}
```

### 8.1 autoload 仅在 REPL 模式中执行

| 模式 | 代码位置 | 是否执行 autoload |
|------|---------|:----------------:|
| REPL | src/config_files.rs#L274 | ✅ |
| 命令 | src/run.rs#L36-L83 | ❌ 无此调用 |
| 脚本 | src/run.rs#L127-L162 | ❌ 无此调用 |

即使 `nu -l -c "command"` 使用 `--login`，autoload 也不会加载。

### 8.2 autoload 目录搜索顺序

vendor 目录的搜索路径由 `get_vendor_autoload_dirs()` 决定（crates/nu-protocol/src/eval_const.rs#L311-L372）：

| 平台 | 搜索路径（按添加顺序） |
|------|---------------------|
| macOS | `/Library/Application Support/nushell/vendor/autoload` |
| Linux/Unix | `$XDG_DATA_DIRS` 各路径下的 `nushell/vendor/autoload`（**逆序添加**，先加的优先级低） |
| Windows | `%ProgramData%\nushell\vendor\autoload` |
| 全平台 | 编译时 `NU_VENDOR_AUTOLOAD_DIR` 环境变量 |
| 全平台 | `data_dir/nushell/vendor/autoload` |
| 全平台 | 运行时 `$env.NU_VENDOR_AUTOLOAD_DIR` 环境变量 |

user 目录只有一个（crates/nu-protocol/src/eval_const.rs#L375-L391）：

```rust
pub fn get_user_autoload_dirs(_: &EngineState) -> Vec<PathBuf> {
    let mut dirs = Vec::new();
    if let Some(config_dir) = nu_path::nu_config_dir() {
        dirs.push(config_dir.join("autoload"));
    }
    dirs
}
```

即 `$nu.default-config-dir/autoload`。

### 8.3 autoload 覆盖 config.nu 的具体机制

每个 autoload 目录中的 .nu 文件使用**同一个 `eval_config_contents()`**，与 config.nu / login.nu 完全一致。这意味着：

```
config.nu 加载完毕 → engine_state.config = config.nu 的最终值
  ↓
login.nu 加载完毕 → engine_state.config = login.nu 覆盖后的值
  ↓
vendor/00-foo.nu → engine_state.config = 00-foo.nu 覆盖后的值
  ↓
vendor/10-bar.nu → engine_state.config = 10-bar.nu 覆盖后的值
  ↓
user/autoload/zz-last.nu → engine_state.config = 最终值
```

**每个 autoload 文件执行后都会立即 `merge_env()`**，所以后面的文件可以看到前面文件的副作用，也可以继续覆盖。

---

## 9. 容错启动机制

Nushell 设计了**多层防御**确保即使配置出错也能启动。

### 9.1 第一层：`eval_source()` 非严格模式静默失败

| 调用场景 | `strict_mode` | 行为 |
|---------|:------------:|------|
| REPL 模式 `setup_config()` | false | 配置文件出错只打印，不退出 |
| 命令模式 `run_commands()` | true | 配置文件出错 `exit(1)` |
| 脚本模式 `run_file()` | true | 配置文件出错 `exit(1)` |
| login.nu 加载 | false | 出错不退出 |
| autoload 文件加载 | false（硬编码） | 出错不退出 |

### 9.2 第二层：配置文件不存在时的优雅处理

- 用户配置文件不存在 → 仅加载 `default_*.nu`，正常启动
- 首次启动配置目录不存在 → 写入 `scaffold_*.nu`（只包含注释的空模板），保证下次启动时有文件存在
- 写入脚手架文件失败 → return，不再 eval，仅靠 default_*.nu 的值

### 9.3 第三层：`catch_unwind` panic 保护

整个 `setup_config()` 被 `catch_unwind(AssertUnwindSafe(|| { ... }))` 包裹（src/config_files.rs#L249-L281）：

```rust
let result = catch_unwind(AssertUnwindSafe(|| {
    // 所有配置加载逻辑...
}));
if result.is_err() {
    eprintln!(
        "A panic occurred while reading configuration files, using default configuration."
    );
    engine_state.config = Arc::new(Config::default())  // ← 终极兜底：Rust 硬编码默认值
}
```

**触发场景**：配置文件中触发了 Rust 层的 panic（极少数情况），此时：
1. 打印提示信息到 stderr
2. **丢弃所有已加载的配置**，包括 `default_*.nu` 中的设置
3. 直接使用 Rust 层的 `Config::default()` 作为配置
4. 进程继续运行，保证至少能进入 REPL 让用户修复问题

> ⚠️ 注意：`catch_unwind` 仅包裹 REPL 模式的 `setup_config()`。命令模式和脚本模式不在 `catch_unwind` 保护内，配置文件中如果触发 panic，进程会直接崩溃。

### 9.4 第四层：`--no-config-file (-n)` 紧急逃生

用户可以通过 `-n` 参数完全跳过**所有**配置加载，仅使用 main.rs 中的最基础初始化，用于诊断配置问题导致的启动故障。

```rust
// src/run.rs#L36 / L127 / L192
if parsed_nu_cli_args.no_config_file.is_none() {
    // 只有这里面的配置加载代码才会执行
}
```

---

## 10. 数据流全景图

以下综合展示从进程启动到配置完全就绪的完整数据流，标注每步的代码来源：

```
进程启动
  │
  ├───────────────────────────────────────────────────────┐
  │ 步骤 0: main.rs 内置初始化                             │
  │  ├─ Config::default() → $env.config  (L316-L319)      │
  │  ├─ NU_PLUGIN_DIRS, NU_LIB_DIRS, NU_VERSION           │
  │  ├─ 继承父进程环境变量 → convert_env_values()          │
  │  └─ $nu 常量生成                                      │
  │  合并：写入 engine_state                               │
  └───────────────────────────────────────────────────────┘
                           │
                           ▼
  ┌───────────────────────────────────────────────────────┐
  │ 步骤 1: default_env.nu（编译嵌入）                      │
  │  └─ PROMPT_COMMAND, PROMPT_COMMAND_RIGHT              │
  │  合并：merge_env → engine_state                        │
  └───────────────────────────────────────────────────────┘
                           │  ← 用户 env.nu 不存在时跳过
                           ▼
  ┌───────────────────────────────────────────────────────┐
  │ 步骤 2: 用户 env.nu（$nu.env-path）                    │
  │  └─ 用户自定义环境变量、ENV_CONVERSIONS 等              │
  │  合并：merge_env → engine_state                        │
  └───────────────────────────────────────────────────────┘
                           │
                           ▼
  ┌───────────────────────────────────────────────────────┐
  │ 步骤 3: default_config.nu（编译嵌入）                   │
  │  └─ $env.config.color_config ← 完整配色表              │
  │  ⚠️ 若步骤 2 已设置 color_config，此处会覆盖            │
  │  合并：merge_env → engine_state                        │
  └───────────────────────────────────────────────────────┘
                           │  ← 用户 config.nu 不存在时跳过
                           ▼
  ┌───────────────────────────────────────────────────────┐
  │ 步骤 4: 用户 config.nu（$nu.config-path）              │
  │  ├─ $env.config.* 覆盖                                │
  │  ├─ 自定义命令、别名定义                                │
  │  └─ 任意启动代码                                       │
  │  合并：merge_env → engine_state                        │
  └───────────────────────────────────────────────────────┘
                           │  ← 非 --login 时跳过
                           ▼
  ┌───────────────────────────────────────────────────────┐
  │ 步骤 5: login.nu（仅 --login 启动时）                   │
  │  ├─ 可覆盖 $env.config                                │
  │  └─ 可覆盖环境变量                                     │
  │  合并：merge_env → engine_state                        │
  └───────────────────────────────────────────────────────┘
                           │  ← 仅 REPL 模式时执行
                           ▼
  ┌───────────────────────────────────────────────────────┐
  │ 步骤 6: autoload 目录（仅 REPL 模式）                   │
  │  ├─ vendor 目录（按 get_vendor_autoload_dirs 顺序）     │
  │  ├─ user 目录（$nu.default-config-dir/autoload）       │
  │  └─ 每个目录内字典序，每个 .nu 文件各 merge_env 一次     │
  │  合并：逐文件 merge_env → engine_state                  │
  └───────────────────────────────────────────────────────┘
                           │
                           ▼
                  配置就绪，进入 REPL / 执行命令
```

---

## 11. 核心代码文件索引

| 文件 | 关键函数/结构 | 职责 |
|------|--------------|------|
| src/main.rs | `main()` | 启动入口，第零层默认值初始化 |
| src/config_files.rs | `setup_config()` `read_config_file()` `eval_default_config()` `read_loginshell_file()` `read_vendor_autoload_files()` | 配置加载编排，REPL 模式主流程 |
| src/run.rs | `run_repl()` `run_commands()` `run_file()` | 三种运行模式的配置加载调用 |
| crates/nu-cli/src/config_files.rs | `eval_config_contents()` | 单个配置文件的执行与 env 合并 |
| crates/nu-protocol/src/config/mod.rs | `Config::default()` `UpdateFromValue` | Rust 层 Config 默认值与用户值合并逻辑 |
| crates/nu-protocol/src/engine/engine_state.rs | `merge_env()` | 将 stack 中的 env + config 刷入 engine_state |
| crates/nu-protocol/src/engine/stack.rs | `update_config()` | 将 $env.config Value 解析为 Config 结构体 |
| crates/nu-engine/src/eval_ir.rs | `$env.config = ...` 赋值处理 | 触发 `stack.update_config()` |
| crates/nu-protocol/src/eval_const.rs | `get_vendor_autoload_dirs()` `get_user_autoload_dirs()` | autoload 目录搜索路径 |
| crates/nu-utils/src/utils.rs | `ConfigFileKind` 枚举 | 配置类型定义，关联 default/scaffold/doc 文件路径 |
| crates/nu-utils/src/default_files/default_env.nu | — | 内置默认环境配置（PROMPT 等） |
| crates/nu-utils/src/default_files/default_config.nu | — | 内置默认主配置（color_config 配色） |
