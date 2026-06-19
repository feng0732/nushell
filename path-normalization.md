# 跨平台路径归一化细节

本文对照 Nushell 源码，详细说明路径归一化中最容易遗漏的四个方面：
**用户目录展开、UNC 路径、符号链接解析、大小写差异**。

---

## 1. 用户目录展开（Tilde Expansion）

波浪号 `~` 是 Shell 中最常见的路径语法，但跨平台的实现细节远比想象复杂。

### 入口函数

核心实现在 [tilde.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/tilde.rs)，公开 API 为 `expand_tilde`：

```rust
pub fn expand_tilde(path: impl AsRef<Path>) -> PathBuf {
    expand_tilde_with_home(path, dirs::home_dir())
}
```
（见 [tilde.rs:L154-L156](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/tilde.rs#L154-L156)）

### 展开的层级结构

| 形式 | 示例 | 处理函数 |
|------|------|----------|
| 当前用户 `~` | `~/foo` | `expand_tilde_with_home` |
| 其他用户 `~user` | `~alice/foo` | `expand_tilde_with_another_user_home` + `user_home_dir` |

### 当前用户展开的边界情况

`expand_tilde_with_home` 中处理了多个容易漏掉的 Corner Case（见 [tilde.rs:L20-L61](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/tilde.rs#L20-L61)）：

1. **`home` 为 `/`（根目录）的情况**：
   ```rust
   if h == Path::new("/") {
       // 不要变成 "//foo"，直接去掉 ~
       path.strip_prefix("~").unwrap_or(path).into()
   }
   ```

2. **空相对路径不追加多余分隔符**：
   当路径为 `~/` 时，strip_prefix 得到空字符串，`PathBuf::push("")` 会额外添加分隔符，因此需要判断：
   ```rust
   if p != Path::new("") {
       h.push(p)
   }
   ```

3. **保留尾随斜杠**：
   如果输入以 `/` 或 `\` 结尾，展开后也要保留。这通过检查最后一个字符实现：
   ```rust
   let need_trailing_slash = path_last_char == Some('/') || path_last_char == Some('\\');
   if need_trailing_slash {
       h.push("");  // push 空串添加尾随斜杠
   }
   ```

4. **波浪号不在开头不展开**：
   `foo~bar` 或 `1~1` 应原封不动返回。只在 `starts_with("~")` 或字节首字符为 `b'~'` 时展开。

### 其他用户目录展开的跨平台差异

`~alice` 这种形式需要查找另一个用户的主目录，不同平台的策略完全不同：

- **Linux（非 macOS/Android）**：使用 `pwd` crate 的 `Passwd::from_name` 查询 `/etc/passwd`，失败时 fallback
  （见 [tilde.rs:L68-L75](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/tilde.rs#L68-L75)）

- **macOS / Windows**：先 `dirs::home_dir()` 拿到当前用户目录，再替换最后一级目录名为目标用户名，检查目录是否存在
  （见 [tilde.rs:L77-L112](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/tilde.rs#L77-L112)）

- **Android**：与 macOS/Windows 共享同一函数体，但行为有本质区别——由 `!cfg!(target_os = "android")` 守卫的"替换最后一级目录名"逻辑在 Android 上**被跳过**。即：
  - `dirs::home_dir()` 返回 `None`：若是 Termux 则返回 `/data/data/com.termux/files/home`，否则走 fallback
  - `dirs::home_dir()` 返回 `Some(current_home)`：**不替换用户名**，直接把当前用户的 home 当作目标用户的 home，检查是否为目录，否则 fallback
  （见 [tilde.rs:L94-L103](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/tilde.rs#L94-L103)）

- **Android Termux 特殊处理**：仅当 `dirs::home_dir()` 返回 `None` 时触发，检测 `TERMUX_VERSION` 环境变量，固定使用 `/data/data/com.termux/files/home`。
  若 `dirs::home_dir()` 成功拿到值，Termux 分支不会被执行。
  （见 [tilde.rs:L82-L87](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/tilde.rs#L82-L87) 和 [tilde.rs:L121-L126](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/tilde.rs#L121-L126)）

- **WASM**：读 `HOME` 环境变量，否则用 `/`

所有平台的 fallback 基目录不同：

| 平台 | FALLBACK_USER_HOME_BASE_DIR |
|------|------------------------------|
| macOS | `/Users` |
| Windows | `C:\Users\` |
| Linux | `/home` |
| Android | `/data` |

（见 [tilde.rs:L5-L15](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/tilde.rs#L5-L15)）

### 展开的调用链

在实际使用中，波浪号展开通常不是独立调用，而是嵌入更完整的路径展开流程：

```
expand_to_real_path()      只展开 ~ 和 ndots
  └─ expand_tilde()
  └─ expand_ndots()

expand_path()              展开 ~ + ndots + . 和 ..（纯词法，不访盘）
  └─ expand_tilde()
  └─ expand_ndots()
  └─ expand_dots()

expand_path_with()         基于某个 cwd 的 expand_path
  └─ join_path_relative()
  └─ expand_path()

absolute_with()            转绝对路径但不解算 symlink
  └─ join_path_relative()
  └─ expand_tilde()
  └─ expand_ndots()
  └─ std::path::absolute()

canonicalize_with()        完全规范化（解算 symlink + 访盘）
  └─ join_path_relative()
  └─ canonicalize()
      └─ expand_tilde()
      └─ expand_ndots()
      └─ std::fs::canonicalize()
      └─ to_winuser_path() (Windows)
```

（见 [expansions.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/expansions.rs)）

其中 `expand_to_real_path` 是"最轻量"的展开——只做系统一定能接受的事，不触碰磁盘：
```rust
pub fn expand_to_real_path<P>(path: P) -> PathBuf {
    let path = expand_tilde(path);
    expand_ndots(path)
}
```
（见 [expansions.rs:L122-L128](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/expansions.rs#L122-L128)）

---

## 2. UNC 路径与 Verbatim 路径（Windows 特有）

Windows 路径体系远比 Unix 复杂。除了常规的 `C:\foo` 磁盘路径，还有 UNC、Verbatim、Device Namespace 等多种前缀形式。

### 前缀类型体系

Rust 标准库的 `std::path::Prefix` 枚举了 7 种前缀（见 [dots.rs:L3](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/dots.rs#L3)）：

| 前缀种类 | 示例 | 说明 |
|----------|------|------|
| `Prefix::Disk(letter)` | `C:` | 磁盘盘符（最常见） |
| `Prefix::UNC(server, share)` | `\\server\share` | UNC 网络共享路径 |
| `Prefix::Verbatim(path)` | `\\?\path` | Verbatim 路径（绕过 260 字符限制） |
| `Prefix::VerbatimUNC(server, share)` | `\\?\UNC\server\share` | Verbatim UNC |
| `Prefix::VerbatimDisk(letter)` | `\\?\C:` | Verbatim 磁盘 |
| `Prefix::DeviceNS(device)` | `\\.\device` | Win32 设备命名空间 |

### 特殊前缀的 `RootDir` 吞掉逻辑

在 `expand_ndots` 和 `expand_dots` 中都有一段关键代码，处理非磁盘前缀的 `RootDir` 组件：

```rust
Component::Prefix(prefix) => {
    match prefix.kind() {
        Prefix::Disk(_) => { /* 正常，后续 RootDir 也要 */ }
        _ => {
            has_special_prefix = true;
        }
    }
    result.push(component)
}
Component::RootDir if has_special_prefix => {
    // 忽略！这会添加一个不在输入中的反斜杠
}
```

（见 [dots.rs:L30-L44](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/dots.rs#L30-L44) 和 [dots.rs:L80-L94](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/dots.rs#L80-L94)）

**为什么这样做？** 对于 `\\server\share`，标准库解析出的组件是：
1. `Prefix(UNC(...))` → 包含 `\\server\share`
2. `RootDir` → 代表 `\`

如果把两者都 push 到 PathBuf，会变成 `\\server\share\`，多了一个不在原输入中的反斜杠。因此 UNC 等特殊前缀后面的 `RootDir` 必须跳过。磁盘前缀 `C:` 则没有这个问题——因为 `C:` 之后确实需要 `\` 才是完整的 `C:\`。

### Verbatim → WinUser 路径转换

Windows 上 `std::fs::canonicalize()` 返回的是 Verbatim 路径（`\\?\C:\...`），这种路径对用户不友好。Nushell 使用 `omnipath` crate 的 `to_winuser_path()` 将其转回常规格式：

```rust
#[cfg(windows)]
fn canonicalize_path(path: &std::path::Path) -> std::io::Result<std::path::PathBuf> {
    path.canonicalize()?.to_winuser_path()
}
```
（见 [expansions.rs:L37-L40](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/expansions.rs#L37-L40)）

同样，在 `expand_dots` 的最后一步 `simiplified` 也会做转换：
```rust
#[cfg(windows)]
fn simiplified(path: &std::path::Path) -> PathBuf {
    path.to_winuser_path().unwrap_or_else(|_| path.to_path_buf())
}
```
（见 [dots.rs:L125-L134](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/dots.rs#L125-L134)）

转换效果示例（来自测试用例）：

| Verbatim 输入 | WinUser 输出 |
|---------------|--------------|
| `\\?\UNC\server\share` | `\\server\share` |
| `\\?\UNC\server\share\dir\file.nu` | `\\server\share\dir\file.nu` |
| `\\?\C:\` | `C:\` |

（见 [dots.rs:L353-L373](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/dots.rs#L353-L373)）

### Windows 保留设备名检测

还有一类特殊路径需要识别——Windows 的保留设备名。即使它们看起来像普通文件名，也不能直接操作：

```rust
pub fn is_windows_device_path(path: &Path) -> bool {
    // 1) 设备命名空间前缀 \\.\
    match path.components().next() {
        Some(Component::Prefix(prefix)) if matches!(prefix.kind(), Prefix::DeviceNS(_)) => {
            return true;
        }
        _ => {}
    }
    // 2) 28 个保留名：CON, PRN, AUX, NUL, COM1-9, LPT1-9（含上标变体）
    let special_paths: [&Path; 28] = [
        Path::new("CON"), Path::new("PRN"), Path::new("AUX"), Path::new("NUL"),
        Path::new("COM1"), /* ... COM2-9, COM¹-COM³ ... */
        Path::new("LPT1"), /* ... LPT2-9, LPT¹-LPT³ ... */
    ];
    special_paths.contains(&path)
}
```
（见 [helpers.rs:L54-L96](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/helpers.rs#L54-L96)）

注意其中包含了 **上标数字**（`COM¹`、`COM²` 等），这是 Windows 兼容性中的一个冷知识。

### path parse 中的 prefix 列

在 `path parse` 命令中，Windows 会额外输出 `prefix` 列，就是用 `Component::Prefix` 提取的：

```rust
#[cfg(windows)]
{
    let prefix = match path.components().next() {
        Some(Component::Prefix(prefix_component)) => {
            prefix_component.as_os_str().to_string_lossy()
        }
        _ => "".into(),
    };
    record.push("prefix", Value::string(prefix, span));
}
```
（见 [parse.rs:L189-L200](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-command/src/path/parse.rs#L189-L200)）

---

## 3. 符号链接解析

符号链接（symlink）的处理是路径归一化中最容易出 Bug 的地方，因为它本质上需要访盘，而且尾随斜杠的语义至关重要。

### 三种展开函数的区别

| 函数 | 解析 symlink | 访盘 | 转绝对 | 适用场景 |
|------|:---:|:---:|:---:|----------|
| `expand_path_with` | ❌ | ❌ | ✅ | 需要稳定的词法归一（不要求文件存在） |
| `absolute_with` | ❌ | 仅读取 cwd | ✅ | 需要绝对路径但不想跟随链接 |
| `canonicalize_with` | ✅ | ✅ | ✅ | 需要真实物理路径 |

（见 [expansions.rs:L55-L82](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/expansions.rs#L55-L82)）

### path expand 命令的实现

`path expand` 命令有两个关键 flag：
- `--strict` / `-s`：失败时抛错而不是 fallback
- `--no-symlink` / `-n`：不解算符号链接

```rust
fn expand(path: &Path, span: Span, args: &Arguments) -> Value {
    if args.strict {
        match canonicalize_with(path, &args.cwd) {
            Ok(p) => {
                if args.not_follow_symlink {
                    // 即使成功了，如果用户不要 symlink，也返回词法展开
                    Value::string(expand_path_with(...).to_string_lossy(), span)
                } else {
                    Value::string(p.to_string_lossy(), span)
                }
            }
            Err(_) => /* 报错 */,
        }
    } else if args.not_follow_symlink {
        // 非严格 + 不要 symlink，直接走 expand_path_with
        Value::string(expand_path_with(...).to_string_lossy(), span)
    } else {
        // 默认：先尝试 canonicalize，失败则 fallback 到 expand_path_with
        canonicalize_with(path, &args.cwd)
            .map(|p| Value::string(p.to_string_lossy(), span))
            .unwrap_or_else(|_| Value::string(expand_path_with(...).to_string_lossy(), span))
    }
}
```
（见 [expand.rs:L145-L183](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-command/src/path/expand.rs#L145-L183)）

### 尾随斜杠与 symlink 的语义

POSIX 规定：带尾随斜杠的路径必须跟随 **最后的** symlink。这是一个极容易忽略的细节：

```sh
mkdir foo
ln -s foo link

cp -r link bar      # bar 是 symlink，复制链接本身
cp -r link/ baz     # baz 是目录，跟随链接后复制
```

Rust 标准库的 `Path::components()` **会吞掉尾随斜杠**，`Path::parent()` 也一样。为了保留语义，Nushell 在 `components` 模块中自己包装了一层：

```rust
pub fn components(path: &Path) -> impl Iterator<Item = Component<'_>> {
    let mut final_component = Some(Component::Normal(OsStr::new("")));
    path.components().chain(std::iter::from_fn(move || {
        if has_trailing_slash(path) {
            final_component.take()   // 追加一个空组件
        } else {
            None
        }
    }))
}
```
（见 [components.rs:L53-L62](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/components.rs#L53-L62)）

重建路径时，`PathBuf::push("")` 恰好会在末尾补一个分隔符，这样尾随斜杠就被保留了。

尾随斜杠的检测也是跨平台的：
- **Windows**：检查最后一个宽字符是否为 `\` 或 `/`
- **Unix**：检查最后一个字节是否为 `/`
- **WASM**：用字符串检查尾 `/`

（见 [trailing_slash.rs:L22-L44](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/trailing_slash.rs#L22-L44)）

### expand_dots 中对 symlink 的保守处理

纯词法的 `expand_dots` 只在 **最后一个组件是 Normal（普通目录/文件名）** 时才回退 `..`：

```rust
Component::ParentDir if last_component_is_normal(&result) => {
    result.pop();
}
```
（见 [dots.rs:L74-L75](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/dots.rs#L74-L75)）

这意味着如果前面的组件可能是 symlink，比如 `link/..`，就不会被词法地消掉——因为 `link` 的父目录未必是当前目录，必须访盘才能确定。同样，根目录之后的 `..` 也被跳过：

```rust
if prev_component == Some(Component::RootDir) && component == Component::ParentDir {
    continue;
}
```
（见 [dots.rs:L97-L99](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/dots.rs#L97-L99)）

### AbsolutePath 的 canonicalize 方法

`AbsolutePath::canonicalize()` 在 Windows 上会额外调用 `to_winuser_path()`，把 `std::fs::canonicalize()` 返回的 Verbatim 路径转回普通路径：

```rust
#[cfg(windows)]
pub fn canonicalize(&self) -> io::Result<CanonicalPathBuf> {
    let path = self.inner.canonicalize()?.to_winuser_path()?;
    Ok(CanonicalPathBuf::new_unchecked(path))
}
```
（见 [path.rs:L1051-L1057](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/path.rs#L1051-L1057)）

---

## 4. 大小写差异

Unix 与 Windows/macOS 文件系统的大小写敏感性差异是路径比较中的经典陷阱。

### 文件系统敏感性检测

Nushell 使用编译时配置来判断：

```rust
fn is_case_insensitive_filesystem() -> bool {
    cfg!(any(target_os = "windows", target_os = "macos"))
}
```
（见 [relative_to.rs:L172-L175](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-command/src/path/relative_to.rs#L172-L175)）

注意这是一个 **基于平台的推断**，并非 100% 准确：
- Windows 上 NTFS 默认不区分大小写，但可以启用区分大小写
- macOS 上 APFS 默认可不区分，但可以格式化为区分大小写
- Linux 上 ext4 默认区分，但也存在不区分大小写的文件系统

### path relative-to 的大小写不敏感回退

`path relative-to` 命令会先用严格的 `strip_prefix`，失败后如果判断当前平台是大小写不敏感文件系统，则尝试大小写不敏感的前缀剥离：

```rust
match lhs.strip_prefix(&rhs) {
    Ok(p) => Value::string(p.to_string_lossy(), span),
    Err(e) => {
        if is_case_insensitive_filesystem()
            && let Some(relative_path) = try_case_insensitive_strip_prefix(&lhs, &rhs)
        {
            return Value::string(relative_path.to_string_lossy(), span);
        }
        /* 报错 */
    }
}
```
（见 [relative_to.rs:L148-L168](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-command/src/path/relative_to.rs#L148-L168)）

### try_case_insensitive_strip_prefix 的实现细节

```rust
fn try_case_insensitive_strip_prefix(lhs: &Path, rhs: &Path) -> Option<std::path::PathBuf> {
    loop {
        match (lhs_components.next(), rhs_components.next()) {
            // Normal 组件：转小写后比较
            (Component::Normal(lhs_name), Component::Normal(rhs_name)) => {
                if lhs_name.to_string_lossy().to_lowercase()
                    != rhs_name.to_string_lossy().to_lowercase()
                {
                    return None;
                }
            }
            // 非 Normal 组件（RootDir, Prefix, CurDir, ParentDir）必须完全匹配
            _ if lhs_comp != rhs_comp => {
                return None;
            }
            _ => {}
            // rhs 用完，收集 lhs 剩余部分
            (Some(lhs_comp), None) => { /* push 剩余 component */ return Some(result); }
            (None, Some(_)) => return None,   // lhs 用完，rhs 还有 → 不是前缀
            (None, None) => return Some(PathBuf::new()),  // 恰好相等
        }
    }
}
```
（见 [relative_to.rs:L178-L226](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-command/src/path/relative_to.rs#L178-L226)）

**关键点：非 Normal 组件必须精确匹配。** 比如 Windows 的盘符前缀 `C:` 虽然是 ASCII，但不能把它当作普通字符串去 lower——它通过 `Component::Prefix` 精确比较。同样 `RootDir`（`/` 或 `\`）也不参与大小写比较。

### 用户手动大小写规范化

`Path` 类型暴露了 `as_mut_os_str()`，配合 `make_ascii_lowercase()` 可以手动规范化（仅限 ASCII）：

```rust
let mut path = PathBuf::from("Foo.TXT");
path.as_mut_os_str().make_ascii_lowercase();
assert_eq!(path, Path::new("foo.txt"));
```
（示例来自 [path.rs:L648-L654](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/path.rs#L648-L654)）

### 测试中的不同平台预期

大小写不敏感比较的测试用例使用同一个函数在 Linux 上应失败、在 Windows/macOS 上应成功的设计：

```rust
let result = relative_to(Path::new("/etc"), Span::test_data(), &args);
if is_case_insensitive_filesystem() {
    match result {
        Value::String { val, .. } => assert_eq!(val, ""),
        _ => panic!("Expected string result on case-insensitive filesystem"),
    }
} else {
    match result {
        Value::Error { .. } => { /* 预期在区分大小写系统上失败 */ }
        _ => panic!("Expected error on case-sensitive filesystem"),
    }
}
```
（见 [relative_to.rs:L238-L268](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-command/src/path/relative_to.rs#L238-L268)）

---

## 小结：归一化函数选择决策树

```
需要路径归一化？
├─ 只需要 ~ 和 ndots → expand_to_real_path()
├─ 需要词法消除 . 和 ..，但文件可能不存在 → expand_path() 或 expand_path_with()
├─ 需要绝对路径，不要跟随 symlink → absolute_with()
├─ 需要真实物理路径（消除所有 symlink）→ canonicalize_with()
│   └─ Windows 上会自动 Verbatim → WinUser
└─ 只处理尾随斜杠 → trailing_slash 模块

比较路径时：
├─ 需要严格相等 → 直接使用 ==
├─ 需要考虑大小写不敏感（Windows/macOS）→ try_case_insensitive_strip_prefix()
└─ 检测 Windows 特殊路径 → is_windows_device_path()
```

---

## 涉及文件索引

| 文件 | 主要职责 |
|------|----------|
| [tilde.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/tilde.rs) | 波浪号展开（当前用户 + 其他用户） |
| [expansions.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/expansions.rs) | 各级展开组合：expand_path / absolute_with / canonicalize_with |
| [dots.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/dots.rs) | ndots 展开、. 和 .. 词法消去、UNC 前缀特殊处理 |
| [components.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/components.rs) | 保留尾随斜杠的 components 包装 |
| [trailing_slash.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/trailing_slash.rs) | 跨平台尾随斜杠检测与剥离 |
| [helpers.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/helpers.rs) | Windows 设备路径检测、home/config 目录获取 |
| [path.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-path/src/path.rs) | Path/PathBuf 类型系统，canonicalize() 方法 |
| [expand.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-command/src/path/expand.rs) | `path expand` 命令实现 |
| [parse.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-command/src/path/parse.rs) | `path parse` 命令中的 prefix 提取 |
| [relative_to.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/75-nushell/crates/nu-command/src/path/relative_to.rs) | 大小写不敏感前缀剥离 |
