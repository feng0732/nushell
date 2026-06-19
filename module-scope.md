# Nushell 模块加载与作用域可见性深度解析

> 代码路径统一使用**仓库相对路径**（以项目根为基准）。

---

## 一、整体架构概览

Nushell 的模块系统由三个核心 crate 协同实现：

| Crate | 职责 | 关键文件 |
|-------|------|----------|
| `nu-parser` | 语法解析、模块/别名/导入的语法处理 | `crates/nu-parser/src/parse_module.rs`、`crates/nu-parser/src/parse_alias.rs` |
| `nu-protocol` | 核心数据结构定义、引擎状态、作用域帧、循环检测 | `crates/nu-protocol/src/module.rs`、`crates/nu-protocol/src/alias.rs`、`crates/nu-protocol/src/engine/overlay.rs`、`crates/nu-protocol/src/engine/state_working_set.rs` |
| `nu-engine` | 运行时作用域数据收集 | `crates/nu-engine/src/scope.rs` |

---

## 二、模块数据结构

### 2.1 Module 结构体

定义于 `crates/nu-protocol/src/module.rs#L36-L46`：

```rust
pub struct Module {
    pub name: Vec<u8>,
    pub decls: IndexMap<Vec<u8>, DeclId>,       // 导出的声明（命令/别名）
    pub submodules: IndexMap<Vec<u8>, ModuleId>, // 导出的子模块
    pub constants: IndexMap<Vec<u8>, VarId>,     // 导出的常量
    pub env_block: Option<BlockId>,              // `export-env { ... }` 环境块
    pub main: Option<DeclId>,                    // `export def main` 主命令
    pub span: Option<Span>,
    pub imported_modules: Vec<ModuleId>,         // 已导入的模块列表（用于变更传播）
    pub file: Option<(ParserPath, FileId)>,      // 源文件信息
}
```

**关键点**：
- `IndexMap` 保证导出顺序稳定
- `main` 字段让模块可以像命令一样直接调用
- `imported_modules` 是**热重载检测**的基础（不是循环检测）
- `env_block` 支持模块级别的环境变量设置

### 2.2 可导出符号 (Exportable)

定义于 `crates/nu-parser/src/exportable.rs#L4-L7`：

```rust
pub enum Exportable {
    Decl { name: Vec<u8>, id: DeclId },    // 命令/别名
    Module { name: Vec<u8>, id: ModuleId }, // 子模块
    VarDecl { name: Vec<u8>, id: VarId },   // 常量
}
```

### 2.3 导入模式 (ImportPattern)

定义于 `crates/nu-protocol/src/ast/import_pattern.rs#L48-L56`：

```rust
pub struct ImportPattern {
    pub head: ImportPatternHead,      // 模块名 + ID（如果已在作用域内则 ID 为 Some）
    pub members: Vec<ImportPatternMember>, // 导入成员：Glob / Name / List
    pub hidden: HashSet<Vec<u8>>,     // hide 命令隐藏的项
    pub constants: Vec<VarId>,        // 需要放入栈的常量
}
```

`ImportPatternMember` 支持三种导入方式：
- **Glob** (`*`)：通配符导入所有导出项
- **Name** (`submod`)：单个命名导入（可级联到子模块）
- **List** (`[a, b, c]`)：显式列表导入

---

## 三、模块加载流程

### 3.1 模块解析入口

模块解析的主入口在 `crates/nu-parser/src/parse_module.rs#L862-L1042` 的 `parse_module` 函数，支持两种语法：

**形式 1：从文件/目录加载**
```nu
module mymodule
```

**形式 2：内联模块块**
```nu
module mymodule {
    export def hello [] { "hello" }
}
```

### 3.2 文件/目录模块加载流程

调用链：`parse_module` → `parse_module_file_or_dir` → `parse_module_file` → `parse_module_block`

关键步骤在 `crates/nu-parser/src/parse_module.rs#L769-L860`（`parse_module_file_or_dir`）：

1. **路径解析**：通过 `find_in_dirs` 在 `$NU_LIB_DIRS` 和当前工作目录中搜索
2. **目录模块**：如果路径是目录，必须存在 `mod.nu` 作为入口文件
3. **文件模块**：直接解析 `.nu` 文件

### 3.3 模块块解析 (parse_module_block)

定义于 `crates/nu-parser/src/parse_module.rs#L489-L664`，核心流程：

```
1. enter_scope()          // 进入独立作用域
2. lex() + lite_parse()   // 词法分析和轻量解析
3. parse_def_predecl()    // 第一遍：预声明所有 def（解决前向引用）
4. 逐语句解析：
   - def/extern        → 普通声明
   - export def/alias/extern/const/use/module → 收集 Exportable
   - use               → 导入其他模块
   - export-env        → 环境块
5. 将导出项注册到 Module 结构体
6. exit_scope()           // 退出独立作用域（非导出定义被丢弃）
```

**预声明机制的作用**：通过 `parse_def_predecl` 在第一遍扫描时先注册所有函数名，解决模块内函数互相调用的前向引用问题。

### 3.4 模块内导出处理

在 `crates/nu-parser/src/parse_module.rs#L147-L364`（`parse_export_in_module`）中处理各种导出：

- **`export def main`**：特殊处理，设置 `module.main`，不进入 `decls`
- **`export def <name>`**：调用 `module.add_decl(name, id)`
- **`export module mod`**：特殊名称 `mod`，将子模块的所有导出**平铺**到当前模块（即 re-export）
- **`export module <name>`**：添加为子模块
- **`export const`**：添加到 `module.constants`
- **`export use`**：支持重新导出另一个模块的内容

---

## 四、use 导入机制

### 4.1 parse_use 核心流程

定义于 `crates/nu-parser/src/parse_module.rs#L1044-L1287`：

```
1. 解析 import pattern（模块名 + 导入成员）
2. 加载模块（优先级分支）：
   a. 若 import_pattern.head.id 是 Some → 直接使用已有 ModuleId
   b. 否则调用 parse_module_file_or_dir() 从文件系统加载（会触发缓存+循环检测）
3. module.resolve_import_pattern()  // 解析具体要导入哪些符号
4. 注册到当前作用域：
   - working_set.use_decls()      // 注册命令
   - working_set.use_modules()    // 注册子模块
   - working_set.use_variables()  // 注册常量
5. parent_module.track_imported_modules()  // 记录导入关系到 imported_modules
```

**关键点**：`import_pattern.head.id` 为 Some 的情况通常出现在 `export use` 或解析器已经在之前步骤中注册过模块时，此时跳过文件系统加载。

### 4.2 导入模式解析 (resolve_import_pattern)

定义于 `crates/nu-protocol/src/module.rs#L114-L339`，递归实现：

**无成员（`use mymodule`）**：
- 导入所有声明，名称带前缀 `mymodule <decl_name>`
- 导入 `main` 为 `mymodule` 本身
- 所有常量包装为 `$mymodule` 记录

**Glob（`use mymodule *`）**：
- 递归导入所有子模块的声明（不带前缀）
- 平铺所有导出项到当前作用域

**Name（`use mymodule submod`）**：
- 若 `submod` 是声明/常量：直接导入
- 若 `submod` 是子模块：递归解析剩余成员

**List（`use mymodule [a, b, c]`）**：
- 遍历列表，每项按 Name 规则处理

### 4.3 导入名称前缀机制

`decls_with_head` 函数在 `crates/nu-protocol/src/module.rs#L352-L369` 实现：

```rust
pub fn decls_with_head(&self, head: &[u8]) -> Vec<(Vec<u8>, DeclId)> {
    // 将 "hello" 变为 "mymodule hello"
    // main 导出直接使用模块名 "mymodule"
}
```

这就是为什么 `use mymodule` 后要写 `mymodule hello` 而不是直接 `hello`。

---

## 五、作用域可见性系统

### 5.1 作用域层级结构

```
EngineState（永久状态）
    ↓
StateDelta（变更集）
    ├── ScopeFrame 0  ← enter_scope() 推入
    │     ├── OverlayFrame "zero"  (默认 overlay)
    │     └── OverlayFrame "my_overlay" (通过 overlay use 添加)
    ├── ScopeFrame 1
    │     └── ...
    └── ...
```

### 5.2 核心结构体

#### ScopeFrame

定义于 `crates/nu-protocol/src/engine/overlay.rs#L47-L64`：

```rust
pub struct ScopeFrame {
    pub overlays: Vec<(Vec<u8>, OverlayFrame)>,  // 所有 overlay
    pub active_overlays: Vec<OverlayId>,         // 当前激活的 overlay（顺序=优先级）
    pub removed_overlays: Vec<Vec<u8>>,          // 被移除的 overlay 名
    pub predecls: HashMap<Vec<u8>, DeclId>,      // 预声明暂存
}
```

#### OverlayFrame

定义于 `crates/nu-protocol/src/engine/overlay.rs#L180-L189`：

```rust
pub struct OverlayFrame {
    pub vars: HashMap<Vec<u8>, VarId>,
    pub decls: HashMap<Vec<u8>, DeclId>,
    pub modules: HashMap<Vec<u8>, ModuleId>,
    pub visibility: Visibility,   // 控制 hide/use 可见性
    pub origin: ModuleId,         // 来源模块
    pub prefixed: bool,           // 是否带名称前缀
    // ...
}
```

#### Visibility

定义于 `crates/nu-protocol/src/engine/overlay.rs#L8-L44`：

```rust
pub struct Visibility {
    decl_ids: HashMap<DeclId, bool>, // true=可见，false=被 hide
}
```

- `use_decl_id()`：标记为可见
- `hide_decl_id()`：标记为不可见
- `merge_with()`：覆盖式合并
- `append()`：仅添加不存在的条目

### 5.3 作用域进入与退出

实现在 `crates/nu-protocol/src/engine/state_delta.rs#L134-L140`：

```rust
pub fn enter_scope(&mut self) {
    self.scope.push(ScopeFrame::new());
}

pub fn exit_scope(&mut self) {
    self.scope.pop();  // 整个 ScopeFrame 被丢弃，内部所有声明自动失效
}
```

模块解析时会调用 `working_set.enter_scope()` / `working_set.exit_scope()`，确保模块内的非导出定义不会泄漏到外部。

### 5.4 声明查找算法

`StateWorkingSet::find_decl` 在 `crates/nu-protocol/src/engine/state_working_set.rs#L443-L477` 中实现：

```
查找顺序（从新到旧）：
1. StateDelta.scope 中的每个 ScopeFrame（逆序）
   a. ScopeFrame.predecls（预声明，用于模块内前向引用）
   b. 每个激活的 OverlayFrame（逆序，后激活的优先级高）
      i.  overlay.predecls
      ii. overlay.decls + overlay.visibility 检查
2. EngineState 中的永久声明
```

查找时会累积 `Visibility`，被 `hide` 的声明即使存在也会被跳过。

### 5.5 hide 机制

`parse_hide` 在 `crates/nu-parser/src/parse_module.rs#L1289-L1466` 中实现：

1. 解析 hide 的 import pattern
2. 找到对应模块，计算要隐藏的声明名列表
3. 调用 `working_set.hide_decls()`，将这些 DeclId 在 Visibility 中标记为 false

**注意**：hide 不是删除声明，只是标记为不可见。底层 DeclId 仍然存在于 overlay 中。

---

## 六、别名 (Alias) 机制

### 6.1 Alias 数据结构

定义于 `crates/nu-protocol/src/alias.rs#L13-L20`：

```rust
pub struct Alias {
    pub name: String,
    pub command: Option<Box<dyn Command>>, // 内部命令，外部调用为 None
    pub wrapped_call: Expression,          // 被包装的调用表达式
    pub description: String,
    pub extra_description: String,
}
```

Alias 实现了 `Command` trait，本质上是一个**特殊的命令包装器**，而不是文本替换宏。

### 6.2 别名解析流程

`parse_alias` 在 `crates/nu-parser/src/parse_alias.rs#L62-L310` 中实现：

```
1. 验证语法：alias <name> = <expansion>
2. 检查别名名称合法性（不能含 #^%，不能是数字/字节大小）
3. 解析等号右侧为 Call 表达式
4. 检查：
   - 不能 alias 关键字（除非在白名单 ALIASABLE_PARSER_KEYWORDS）
   - 不能 alias 纯表达式（只能是命令调用）
   - 模块内不能 alias main
5. 创建 Alias 结构体，调用 working_set.add_decl() 注册
```

**设计取舍**：Nushell 的别名是**基于命令包装**而非文本替换，优点是类型安全、可复用完整命令管道，缺点是无法实现像 Bash 那样的任意参数位置替换。

---

## 七、循环依赖与重复导入：两层防护机制

### 7.1 概览：检测顺序

Nushell 对循环导入和重复导入有**两层机制**，且执行顺序非常关键。在 `parse_module_file`（`crates/nu-parser/src/parse_module.rs#L706-L758`）中：

```
① add_file() + get_span_for_file()
      ↓
② find_module_by_span(new_span) + module_needs_reloading()
      │  ├─ 命中缓存 → 直接返回 ModuleId（跳过后续所有检测）
      │  └─ 未命中 ↓
③ working_set.files.push(path)   ← 显式循环检测（FileStack 栈）
      │  ├─ 检测到循环 → ParseError::CircularImport
      │  └─ 无循环 ↓
④ parse_module_block()           ← 实际解析
      ↓
⑤ working_set.files.pop()
      ↓
⑥ working_set.add_module()       ← 注册到全局，为后续缓存命中做准备
```

> **关键洞察**：只有当 ② 缓存**未命中**时，才会走到 ③ 的显式循环检测。两者保护的是不同场景。

### 7.2 第一层：Span 缓存机制（重复导入复用）

**目标**：避免同一个文件被**重复解析**。

#### 7.2.1 路径解析与绝对化

在调用 `add_file` 之前，用户输入的模块路径会先经过 **`find_in_dirs` → `find_in_dirs_with_id`** 解析（定义于 `crates/nu-parser/src/parse_source.rs#L621-L690`）：

```
用户输入路径（如 "./mymod" 或 "mymod"）
    ↓
① 先查 virtual paths（内置嵌入式模块）
    ↓ 未命中
② absolute_with(filename, actual_cwd)   ← 关键：路径绝对化
    │  actual_cwd 来自 FileStack.top()（当前正在解析的文件的目录）
    │  absolute_with 会：展开 ~ / 解析 .. / 调用 std::path::absolute
    │  不会做 canonicalize（不解析软链/符号链接）
    ↓ p.exists() 为真 → 返回 ParserPath::RealPath(绝对路径)
    ↓ 否则
③ 遍历 $NU_LIB_DIRS 中的每个目录，absolute_with 后查找
    ↓ 命中 → 返回绝对化后的 ParserPath
```

因此，到达 `parse_module_file` 的 `path: ParserPath` 中的 `RealPath(p)` **已经是一条经过绝对化处理但未做物理路径规范化的绝对路径**。

#### 7.2.2 文件登记：文件名 + 内容字节逐字节比较（非哈希）

`StateWorkingSet::add_file` 在 `crates/nu-protocol/src/engine/state_working_set.rs#L339-L359` 中实现文件去重登记：

```rust
pub fn add_file(&mut self, filename: &str, contents: &[u8]) -> FileId {
    // 线性扫描已登记的所有文件（permanent_state.files + delta.files）
    for (idx, cached_file) in self.files().enumerate() {
        // 逐字节字符串比较文件名（路径字符串）
        if &*cached_file.name == filename
            // 并且逐字节比较文件内容（整份文件字节切片）
            && &*cached_file.content == contents
        {
            return FileId::new(idx);  // 命中：复用旧 FileId → 同一个 Span
        }
    }
    // 未命中：分配新的 covered_span，追加 CachedFile
    ...
}
```

**核心事实**：
- ❌ **不是哈希比较**：没有计算 MD5/SHA 等哈希值，是对 `&[u8]` 内容做**完整字节切片比较**（`slice == slice` 的 O(n) 逐字节比较）
- ❌ **不是物理文件相等判断**：仅比较路径字符串（而非 inode / 文件句柄），因此同一物理文件通过不同路径形式访问可能被视为不同文件
- ✅ **文件名来自 `path.path().to_string_lossy()`**：即上一步 absolute_with 后的绝对路径字符串（不是用户原始输入路径）
- ✅ **文件名比较是字符串级别的**：区分大小写，逐字符完全一致才算匹配

`parse_module_file` 中的调用（`crates/nu-parser/src/parse_module.rs#L734`）：
```rust
let file_id = working_set.add_file(&path.path().to_string_lossy(), &contents);
```

#### 7.2.3 缓存复用的精确边界条件

缓存复用的链路为：**文件登记去重 → Span 相同 → find_module_by_span 命中**

| 场景 | add_file 是否复用 | find_module_by_span 是否命中 | 原因 |
|------|-----------------|----------------------------|------|
| 同一路径字符串 + 内容不变 | ✅ 复用 | ✅ 命中（模块已注册后） | 路径+内容完全相同 |
| 同一物理文件，通过相对路径的不同写法（如 `./a.nu` vs `subdir/../a.nu`） | ✅ 复用 | ✅ 命中 | `absolute_with` 会解析 `..`，得到同一绝对路径 |
| 同一物理文件，通过软链 vs 真实路径分别访问 | ❌ **不复用** | ❌ 未命中 | `absolute_with` 不调用 canonicalize，软链路径保持原样，字符串不同 |
| 同一路径 + 内容有 1 字节变更 | ❌ 不复用 | ❌ 未命中 | 内容字节比较失败 |
| Windows 下 `C:\A.nu` vs `c:\a.nu` | ⚠️ 取决于 PathBuf 语义 | ⚠️ 取决于 `to_string_lossy` | PathBuf 大小写敏感，字符串比较也区分大小写 |
| 不同路径但文件内容完全相同（两份拷贝） | ❌ 不复用 | ❌ 未命中 | 路径字符串不同，即使内容一样也视为不同文件 |

**关键边界**：`add_file` 的文件名来源是已经过 `absolute_with` 绝对化的路径字符串，因此相对路径的不同写法通常会被归一化为相同路径；但**软链、大小写差异、跨驱动器等情况会导致同一物理文件被登记为多条记录**。

#### 7.2.4 Span → ModuleId 的查找

`StateWorkingSet::find_module_by_span` 在 `crates/nu-protocol/src/engine/state_working_set.rs#L1029-L1040`：

```rust
pub fn find_module_by_span(&self, span: Span) -> Option<ModuleId> {
    // 先在 delta.modules 中线性扫描（当前解析会话新增的模块）
    for (id, module) in self.delta.modules.iter().enumerate() {
        if Some(span) == module.span {
            return Some(ModuleId::new(self.permanent_state.num_modules() + id));
        }
    }
    // 再在 permanent_state.modules 中扫描（永久已合并的模块）
    for (module_id, module) in self.permanent_state.modules.iter().enumerate() {
        if Some(span) == module.span {
            return Some(ModuleId::new(module_id));
        }
    }
    None
}
```

**命中条件**（两者必须同时满足）：
1. 已有一个 Module 的 `span` 与当前文件的 Span 相等（Span 相等的前提是 `add_file` 内容+路径都匹配）
2. `module_needs_reloading()` 返回 false（文件内容未变、子模块也未变）

#### 7.2.5 时间维度：解析完成前后的缓存行为差异

缓存机制能避免重复解析，但**有严格的时间边界**（Span 相同不代表 find_module_by_span 一定命中）：

| 场景 | add_file 复用 Span | find_module_by_span 命中 | 原因 |
|------|-------------------|------------------------|------|
| 同一文件在解析完成后再次被 `use` | ✅ 复用 | ✅ 命中 | 已完成 ⑥ `add_module()`，span→ModuleId 映射存在 |
| 同一文件在**解析过程中**被循环引用 | ✅ 复用（如果路径+内容相同） | ❌ 未命中 | 还没执行到 ⑥，模块还没注册到全局模块表 |
| 文件内容发生变更 | ❌ 不复用 | ❌ 未命中 | 内容字节比较失败，add_file 分配新 Span |
| 同一物理文件通过软链 vs 真实路径分别导入 | ❌ 不复用 | ❌ 未命中 | 路径字符串不同（未 canonicalize），add_file 认为是两个文件 |

> **重要推论**：在真正的循环依赖（A→B→A）场景中，虽然 `add_file` 可能因为内容相同而复用 Span，但 **find_module_by_span 不会命中**（因为 A 尚未完成 add_module 注册），因此流程会继续走到 FileStack.push 的循环检测环节。这是两层机制的**协作关系**。

### 7.3 第二层：FileStack 显式循环检测

**目标**：捕获"解析调用栈中的循环"——即 A 解析未完成时，B 通过 `use` 链再次请求解析 A。

#### 7.3.1 入栈路径的来源：与 add_file 共享绝对化路径

在 `parse_module_file` 中，FileStack 和 add_file 使用的路径是**同一个 `ParserPath` 对象的两种表达方式**：

```rust
// crates/nu-parser/src/parse_module.rs#L734, L743
let file_id = working_set.add_file(&path.path().to_string_lossy(), &contents);
//                                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//                                 ParserPath.path() → &Path → to_string_lossy()
// ...
if let Err(e) = working_set.files.push(path.clone().path_buf(), path_span) {
//                                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//                                 ParserPath.path_buf() → PathBuf（消费 ParserPath）
```

两者都基于 `find_in_dirs` 返回的 `ParserPath::RealPath(绝对化路径)`，因此：
- **add_file 比较的**：路径字符串（`&str`）+ 内容字节（`&[u8]`）
- **FileStack 比较的**：`PathBuf`（Rust 标准库路径类型，使用平台原生的 `PartialEq` 实现）

两者底层字节一致，但数据类型不同。

#### 7.3.2 FileStack 数据结构与路径比较语义

定义于 `crates/nu-protocol/src/engine/state_working_set.rs#L1180-L1240`：

```rust
/// Files being evaluated, arranged as a stack.
/// The current active file is on the top of the stack.
/// When a file source/import another file, the new file is pushed onto the stack.
/// Attempting to add files that are already in the stack (circular import) results in an error.
///
/// Note that file paths are compared without canonicalization, so the same
/// physical file may still appear multiple times under different paths.
/// This doesn't affect circular import detection though.
pub struct FileStack(Vec<PathBuf>);
```

#### 7.3.3 循环检测核心算法

```rust
pub fn push(&mut self, path: PathBuf, span: Span) -> Result<(), ParseError> {
    // 从栈顶逆序查找：找到最近一次出现该路径的位置
    // p == &path 使用 PathBuf 的 PartialEq 实现（按字节比较原始路径组件）
    if let Some(i) = self.0.iter().rposition(|p| p == &path) {
        // 构造错误消息，展示完整的循环链
        let filenames: Vec<String> = self.0[i..]
            .iter()
            .chain(std::iter::once(&path))
            .map(|p| p.to_string_lossy().to_string())
            .collect();
        let msg = filenames.join("\nuses ");
        // 例如："a.nu\nuses b.nu\nuses a.nu"
        return Err(ParseError::CircularImport(msg, span));
    }
    self.0.push(path);
    Ok(())
}
```

错误类型定义于 `crates/nu-protocol/src/errors/parse_error.rs#L274-L276`：
```rust
CircularImport(String, #[label = "detected circular import"] Span)
```

**`PathBuf == PathBuf` 的比较语义**：
- 逐组件（component）比较字节，不做规范化
- 不解析符号链接 / 软链
- Windows 上大小写敏感（`C:\A.nu` ≠ `C:\a.nu`）
- 尾随路径分隔符不影响比较（`/a/b` ≡ `/a/b/`）

因此，和 `add_file` 一样，FileStack 也是**字符串级别的路径等价**，非物理文件级别的等价。

#### 7.3.4 FileStack 与 add_file 路径比较的对齐性

| 场景 | add_file（字符串比较） | FileStack（PathBuf 比较） | 是否一致 |
|------|----------------------|--------------------------|---------|
| 相对路径不同写法（`./a` vs `dir/../a`） | ✅ 相等（absolute_with 归一化） | ✅ 相等（absolute_with 归一化） | ✅ 一致 |
| 软链 vs 真实路径 | ❌ 不等（字符串不同） | ❌ 不等（PathBuf 字节不同） | ✅ 一致 |
| Windows 大小写不同 | ❌ 不等（字符串不同） | ❌ 不等（PathBuf 比较也区分大小写） | ✅ 一致 |
| 同文件不同驱动器路径映射 | ❌ 不等 | ❌ 不等 | ✅ 一致 |

**结论**：由于两者的路径都源自同一个 `ParserPath`（经过 `absolute_with` 处理），因此 FileStack 和 add_file 的路径判断结果在实际中是**一致**的——如果 add_file 认为是同一文件（复用 Span），FileStack 也会认为是同一路径（触发循环检测），反之亦然。

#### 7.3.5 入栈/出栈的精确位置

在文件模块解析（`parse_module_file`）中：
- **入栈**：`crates/nu-parser/src/parse_module.rs#L743`，在缓存检查**之后**、实际解析**之前**
- **出栈**：`crates/nu-parser/src/parse_module.rs#L751`，解析完成后立即弹出

在 `source`/`source-env` 命令（`crates/nu-parser/src/parse_source.rs`）中也有相同的入栈/出栈模式：
- 入栈：`#L119` / `#L317`
- 出栈：`#L135` / `#L333`

#### 7.3.6 FileStack 的边界

| 场景 | 是否触发 CircularImport | 原因 |
|------|------------------------|------|
| A.nu → use B.nu → use A.nu（真正的循环） | ✅ 报错 | A 在栈中，B 解析中再次 push A 被检测 |
| A.nu → use B.nu → use C.nu；D.nu → use B.nu | ❌ 不触发 | B 首次解析完后已 pop 出栈；D→B 走缓存命中（第二层不被执行） |
| 内联模块 `module foo { ... }` | ❌ 不触发 | 直接调用 `parse_module_block`，绕过 `parse_module_file`，不经过 FileStack |
| 同文件通过不同路径字符串导入（`./a.nu` vs `../dir/a.nu`） | ❌ 不会漏报（正常不触发） | 经过 `absolute_with` 归一化为同一路径 |
| 同物理文件通过软链和真实路径分别访问 | ⚠️ 可能漏报 | 路径未做 canonicalize，路径字符串不同 |
| `overlay use` 一个模块 | ✅ 经过检测 | `overlay use` 内部同样调用模块解析流程 |

### 7.4 典型场景推演

#### 场景 1：重复导入（安全）

```nu
# main.nu
use a
use a   # 第二次 use 同一个文件
```

执行流程：
1. 第一次 `use a`：缓存未命中 → files.push(a) → 解析 → files.pop() → add_module(A)
2. 第二次 `use a`：`find_module_by_span(A_span)` **命中** → 直接返回 A 的 ModuleId
3. **结果**：成功，无重复解析

#### 场景 2：间接循环依赖（报错）

```nu
# a.nu
use b
def fa [] { fb }

# b.nu
use a
def fb [] { fa }
```

执行流程（入口为 `use a`）：
1. 解析 a.nu：缓存未命中 → files.push(a) → 进入 parse_module_block
2. 遇到 `use b`：触发 b.nu 解析
3. 解析 b.nu：缓存未命中 → files.push(b) → 进入 parse_module_block
4. 遇到 `use a`：触发 a.nu 解析
5. **缓存检查**：find_module_by_span(A_span) → **未命中**（步骤 1 还在 parse_module_block 中，未到 add_module）
6. **files.push(a)**：检测到 a 已在栈 `[a, b]` 中 → **ParseError::CircularImport**

**结果**：报错，显示循环链 `a.nu\nuses b.nu\nuses a.nu`

#### 场景 3：DAG 依赖链（正确）

```nu
# lib.nu → 公共库
# a.nu  → use lib
# b.nu  → use lib
# main  → use a, use b
```

执行流程：
1. `use a` → A 未缓存 → push(a) → 解析 A 中 `use lib` → lib 未缓存 → push(lib) → 解析 lib → pop(lib) → add_module(Lib) → 继续 A → pop(a) → add_module(A)
2. `use b` → B 未缓存 → push(b) → 解析 B 中 `use lib` → **find_module_by_span(Lib_span) 命中** → 直接返回 Lib，不 push，不重新解析 → 继续 B → pop(b) → add_module(B)

**结果**：成功，lib.nu 只被解析一次

### 7.5 导入关系跟踪 (imported_modules)

每个 Module 维护 `imported_modules: Vec<ModuleId>`，在 `parse_use` 中通过 `track_imported_modules` 记录。**这不是循环检测用的**，而是用于：

1. **热重载检测** (`module_needs_reloading`)：递归检查所有导入的子模块是否有文件变更
2. **模块变更传播**：当依赖的模块变更时，当前模块也需要重新解析

实现于 `crates/nu-parser/src/parse_module.rs#L666-L704`：

```rust
fn module_needs_reloading(working_set: &StateWorkingSet, module_id: ModuleId) -> bool {
    // 1. 检查所有 export submodules 是否变更
    // 2. 递归检查 submodule_need_reloading（含 imported_modules）
    // 任意一个变更 → 当前模块也需要重新加载
}
```

### 7.6 前向引用（模块内循环）

模块内部的函数互相调用通过**两阶段解析**解决（不属于循环导入检测范畴）：

1. 第一阶段：`parse_def_predecl` 扫描所有 `def`，只注册函数名（预声明）
2. 第二阶段：正常解析函数体，此时所有函数名已经可用

实现在 `crates/nu-parser/src/parse_module.rs#L510-L514`：

```rust
// 第一遍：预声明
for pipeline in &output.block {
    if pipeline.commands.len() == 1 {
        parse_def_predecl(working_set, pipeline.commands[0].command_parts());
    }
}
// 第二遍：完整解析
for pipeline in output.block.iter() { ... }
```

---

## 八、运行时作用域数据收集

`ScopeData` 在 `crates/nu-engine/src/scope.rs#L9-L16` 中负责为 `scope` 系列命令收集数据：

```rust
pub struct ScopeData<'e, 's> {
    engine_state: &'e EngineState,
    stack: &'s Stack,
    vars_map: HashMap<&'e Vec<u8>, &'e VarId>,
    decls_map: HashMap<&'e Vec<u8>, &'e DeclId>,
    modules_map: HashMap<&'e Vec<u8>, &'e ModuleId>,
    visibility: Visibility,
}
```

收集方法：
- `populate_vars/decls/modules`：遍历所有激活的 OverlayFrame，收集符号
- `collect_commands/aliases/externs/modules`：按类别输出，供 `scope commands` 等命令使用
- Visibility 过滤：被 hide 的声明不会出现在结果中

---

## 九、关键设计总结

| 设计点 | 实现方式 | 位置 |
|--------|----------|------|
| **模块隔离** | `enter_scope/exit_scope` 包裹模块解析，非导出定义随 ScopeFrame 丢弃 | `crates/nu-parser/src/parse_module.rs#L494, #L661` |
| **名称前缀** | `decls_with_head` 将导出名包装为 `<module> <decl>` | `crates/nu-protocol/src/module.rs#L352-L369` |
| **可见性控制** | `Visibility` 哈希表，hide 只是标记不可见而非删除 | `crates/nu-protocol/src/engine/overlay.rs#L8-L44` |
| **前向引用** | 两阶段解析：先预声明所有 def，再解析函数体 | `crates/nu-parser/src/parse_module.rs#L510-L514` |
| **路径绝对化** | `absolute_with` 展开 `~`、解析 `..`、调用 `std::path::absolute`；**不做** canonicalize（不解析软链） | `crates/nu-path/src/expansions.rs#L73-L82`、`crates/nu-parser/src/parse_source.rs#L661-L685` |
| **文件登记去重** | `add_file()` 线性扫描，**逐字节比较**（路径字符串 `&str == &str` + 文件内容 `&[u8] == &[u8]`），无哈希 | `crates/nu-protocol/src/engine/state_working_set.rs#L339-L359` |
| **重复导入复用** | `find_module_by_span()` 线性扫描已注册模块的 span（要求模块已完成 `add_module` 注册） | `crates/nu-protocol/src/engine/state_working_set.rs#L1029-L1040` |
| **循环导入检测** | `FileStack` 栈式跟踪当前解析链，`push()` 时 `PathBuf == PathBuf` 比较，命中抛出 `CircularImport` | `crates/nu-protocol/src/engine/state_working_set.rs#L1180-L1240` |
| **检测优先级** | `add_file` → `find_module_by_span`（时间维度：解析完成后命中）→ `files.push`（时间维度：解析中循环检测） | `crates/nu-parser/src/parse_module.rs#L734-L746` |
| **别名实现** | Alias 实现 Command trait，是命令包装器而非文本替换 | `crates/nu-protocol/src/alias.rs#L22-L68` |
| **热重载** | `module_needs_reloading` 递归检查文件内容 + imported_modules 链 | `crates/nu-parser/src/parse_module.rs#L666-L704` |
| **Overlay 系统** | 每个 ScopeFrame 可有多个 Overlay，支持动态激活/移除 | `crates/nu-protocol/src/engine/overlay.rs#L47-L177` |
| **路径比较一致性** | add_file（字符串）和 FileStack（PathBuf）共享同一 `ParserPath` 来源，实际行为一致（软链/大小写场景同时不命中） | `crates/nu-parser/src/parse_module.rs#L734, L743` |
