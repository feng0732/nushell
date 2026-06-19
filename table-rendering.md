# Nushell 表格渲染流程分析

本文档结合源码分析 Nushell 表格输出系统在**列宽分配**、**内容换行/截断策略**、**主题色彩**三者之间的权衡取舍，梳理完整渲染链路与大表性能优化手段。

> **阅读说明**：文中标注「⚠️ 推断」的条目是基于代码结构的合理推测，尚未通过运行时验证；其余结论均可在引用的源码行中找到直接证据。

---

## 一、整体架构概览

表格渲染涉及 4 个核心 crate 的协作：

| 模块 | 仓库路径 | 职责 |
|------|---------|------|
| `nu-command/viewers` | `crates/nu-command/src/viewers/table.rs` | `table` 命令入口，参数解析、流式分页调度 |
| `nu-table` | `crates/nu-table/src/table.rs` | 核心渲染引擎（列宽估计、截断/换行、主题加载） |
| `nu-protocol/config` | `crates/nu-protocol/src/config/table.rs` | 配置类型定义（`TableMode` / `TrimStrategy` / `TableConfig`） |
| `nu-color-config` | `crates/nu-color-config/src/style_computer.rs` | 单元格颜色计算 |

数据从 `table` 命令入口到最终字符串的处理流水线：

```
PipelineData (List/ListStream/Record)
        ↓
handle_row_stream / handle_record       (数据分流)
        ↓
PagingTableCreator::next()              (流式分批 + 缩写)
        ↓
create_table / create_table_with_header_and_index
                                        (构建 NuTable，写入数据/样式)
        ↓
colorize_space                          (第 2 遍空格着色，仅 list_table)
        ↓
configure_table                         (加载主题、footer、边框色)
        ↓
NuTable::draw(termwidth)                ← 核心渲染
  ├─ table_truncate                     (列宽估计，输出 WidthEstimation)
  ├─ draw_table
  │   ├─ set_styles                     (对齐 + 单元格颜色配置)
  │   ├─ set_indent                     (左右 padding)
  │   ├─ load_theme                     (边框字符 + 边框色)
  │   ├─ truncate_table → DimensionCtrl (应用 Truncate/Wrap 策略)
  │   └─ table_set_border_header        (header-on-separator 模式)
  └─ table.to_string()
```

---

## 二、列宽分配：三种截断策略的取舍

### 2.1 策略选择的分水岭 `maybe_truncate_columns`

`crates/nu-table/src/table.rs` 第 849–882 行是整个列宽分配的总入口，通过两个布尔条件在三种策略间切换：

```rust
fn maybe_truncate_columns(..., termwidth: usize, truncate_by_head: bool) -> WidthEstimation {
    const TERMWIDTH_THRESHOLD: usize = 120;
    let preserve_content = termwidth > TERMWIDTH_THRESHOLD;

    if truncate_by_head {
        truncate_columns_by_head(...)        // 策略 A：以表头宽度为主导
    } else if preserve_content {
        truncate_columns_by_columns(...)     // 策略 B：宽终端，尽量保留内容，均匀挤列
    } else {
        truncate_columns_by_content(...)     // 策略 C：窄终端，保内容完整性优先
    }
}
```

三个关键的设计决策点（均有直接代码证据）：

1. **`truncate_by_head` 条件**：当启用 `header_on_separator` 且无显式优先级列时，策略 A 激活——把列压缩到「刚好能放下列名」的宽度，最大化**列数量**（可读性牺牲最大）。
2. **终端宽度阈值 120**：宽度 >120 认为是宽屏，策略 B 的最小列宽为 10（`MIN_ACCEPTABLE_WIDTH = 10`），保证**每列至少可读**。
3. **窄终端（≤120）**：策略 C 的最小可接受宽度仅 5 字符，宁愿**少显示几列**也要把前几列展示完整。

### 2.2 策略 A —— 按表头宽度 `truncate_columns_by_head`

位置：`crates/nu-table/src/table.rs` 第 1475–1581 行

**核心思想**：每个数据列先尝试完整内容宽度，放不下就退化为 `header_width + padding`。

```rust
for (i, &column_width) in widths_original.iter().enumerate() {
    let head_width = NuRecordsValue::width(&data[0][i]) + pad;
    let mut use_width = column_width;
    let mut next_move = use_width + vertical_width;
    if width + next_move > termwidth {
        use_width = head_width;           // 退化成表头宽度
        next_move = use_width + vertical_width;
        if width + next_move > termwidth {
            break;
        }
    }
    widths.push(use_width);
    ...
}
```

**与主题色的交互**：此策略下数据列被强力压缩，但因 tabled 启用了 `ansi` 特性（见 `crates/nu-table/Cargo.toml`）且 `Truncate` 调用了 `.suffix_try_color(true)`（第 774 行），截断操作是 ANSI 感知的，不会从 ESC 序列中间切断。

### 2.3 策略 B —— 宽终端 `truncate_columns_by_columns`

位置：`crates/nu-table/src/table.rs` 第 1058–1210 行

**两阶段分配**：
- **第一轮**：每列先给 `min(10, original_width)`，尽可能多地塞进列。
- **第二轮**：剩余空间 `distribute_available_width` 分配 —— 先轮询（round-robin）给显式优先级列，再按原始需求比例分给其余列。

```rust
fn distribute_available_width_round_robin(...) -> usize {
    while available > 0 {
        for &column in width_priority_columns {
            if column >= widths.len() { continue; }
            let used_width = widths[column];
            let col_width = widths_original[column];
            if used_width < col_width {
                widths[column] += 1;       // 一单位一单位地加，保证公平
                available -= 1;
                consumed_in_round += 1;
            }
        }
        if consumed_in_round == 0 { break; }
    }
}
```

当有 `width_priority_columns` 时（如 `ps` 的 `name` 列，通过 pipeline metadata 传入），`compact_partial_visibility_for_priority` 会**主动砍掉右侧非优先列**，把空间让给优先列——这是三者权衡中最激进的「列宽 > 列数」选择。

### 2.4 策略 C —— 窄终端 `truncate_columns_by_content`

位置：`crates/nu-table/src/table.rs` 第 885–1046 行

**设计哲学**：「少而全」。最小列宽仅 5 字符，若首列连 5+vertical+trailing 的空间都没有，直接返回 `WidthEstimation { needed: vec![] }`，最终会触发 `Couldn't fit table into N columns!` 的错误提示。

当列被截断时，最后一列替换为 `...`（宽度 3 + padding），通过 `push_empty_column` / `truncate_rows` 完成。

---

## 三、内容换行 vs 截断：`TrimStrategy` 的交互

### 3.1 两种策略的配置

`crates/nu-protocol/src/config/table.rs` 第 159–198 行定义了枚举：

```rust
pub enum TrimStrategy {
    Wrap { try_to_keep_words: bool },    // 换行（默认），保留单词边界
    Truncate { suffix: Option<String> }, // 截断 + 可选后缀（如 "…"）
}
```

默认值：`TrimStrategy::Wrap { try_to_keep_words: true }`（第 193–197 行）。

### 3.2 应用时机：`DimensionCtrl::change`

`crates/nu-table/src/table.rs` 第 694–723 行根据 `width.truncate` 标志走不同路径：

```rust
impl TableOption<...> for DimensionCtrl {
    fn change(self, recs: &mut NuRecords, cfg: &mut ColoredConfig, dims: &mut CompleteDimension) {
        if self.width.truncate {
            width_ctrl_truncate(self, recs, cfg, dims);  // 列被压缩 → 需要 Wrapping/Truncating
            return;
        }
        if self.expand {
            width_ctrl_expand(self, recs, cfg, dims);    // --expand 模式，整体撑满
            return;
        }
        dims.set_heights(self.heights);                  // 无需调整，直接用缓存
        dims.set_widths(self.width.needed);
    }
}
```

**高度重算的两种路径**（代码证据在 `width_ctrl_truncate` 第 736–784 行）：

- `Wrap` 模式：每列调用 `CellOption::change(wrap, ...)` 后，**手动遍历该列所有行**调用 `recs.count_lines()` 更新高度。只扫受影响列，而非全表。
- `Truncate` 模式：调用 `CellOption::change(truncate, ...)` 后，**不手动更新高度**。但 `hint_change()` 方法（第 711–722 行）在 `matches!(self.trim_strategy, TrimStrategy::Truncate { .. })` 时返回 `Some(Entity::Row(0))`，提示 tabled 内部需要重算行高。

> **⚠️ 推断**：从注释 `// NOTE: Only truncation case must be relaclucated in term of height.` 来看，作者认为截断后行高可能变化（例如原文本有 `\n`，截断后行数变少）。但 Wrap 模式代码中手动重算了高度，与注释表述不完全一致。

### 3.3 与列宽 + 主题色的三角关系

| 维度 | Wrap 换行 | Truncate 截断 | 代码依据 |
|------|----------|--------------|---------|
| **ANSI 安全性** | ✅ tabled `ansi` 特性下换行识别转义序列 | ✅ `.suffix_try_color(true)` 显式支持颜色感知 | `Cargo.toml` features; `table.rs` L774 |
| **行高膨胀** | ❌ 换行会拉高整行高度 | ✅ 通常每行仅 1 行（原文本有 `\n` 除外） | `table.rs` L766-L769 |
| **信息完整性** | ✅ 全部可见，只是折行 | ❌ 超出部分被丢弃 | `TrimStrategy` 枚举定义 |
| **与 `header_on_separator` 配合** | ⚠️ 数据行可换行，但 header 行用 `Truncate` 不换行（`SetLineHeaders`） | ✅ header 通常短于列宽 | `table.rs` L1821 |

**推荐配置组合**（基于代码行为的合理建议）：
- **日常 REPL（默认）**：`TrimStrategy::Wrap { try_to_keep_words: true }` + `mode: rounded`
- **自动化脚本 / grep 管道**：`TrimStrategy::Truncate { suffix: Some("…".into()) }` + `mode: none`
- **大屏查数（>160 列）**：`header_on_separator: true` + Wrap，充分利用宽终端塞下最多列

---

## 四、主题色系统：着色链路与副作用

### 4.1 样式计算链：四种渲染模式的着色路径

颜色来源于 `StyleComputer`（由 `$env.config.color_config` 生成）。`table` 命令的四种视图模式对应四条独立的着色链路（分派点：`crates/nu-command/src/viewers/table.rs` 第 690–746 行的 `build_table_kv` / `build_table_batch`）。

所有链路的共同点：`clean_charset`（字符集规整化：`\t`→4 空格、去 `\r`）都发生在**颜色写入文本之前**。

---

#### 链路 1：普通 List 表（`TableView::General` + `Value::List`）

**入口**：`crates/nu-command/src/viewers/table.rs` 第 733 行 → `JustTable::table` → `list_table`（`crates/nu-table/src/types/general.rs` 第 27 行）

**着色方式：样式分离路径**。文本与样式分开存储，渲染阶段再合并。

```
构建阶段（create_table → create_table_with_* → get_string_value）：
1. get_value_style()                  → 返回 (text, style) 元组，文本与样式分离
2. clean_charset                      → 文本规整化【仅 String 类型】
3. table.insert(pos, text)            → 写入纯文本
4. table.insert_style(pos, style)     → 写入样式（独立存储于 NuTable.styles）

第二遍空格着色（list_table 第 34–35 行）：
5. colorize_space(out.table.get_records_mut(), ...)
                                       → 全表扫描，对所有单元格内首尾空格着色
                                       【TODO 注释：应合并到构建阶段一次完成】

渲染阶段（NuTable::draw → set_styles → colorize_table）：
6. colorize_table(table, colors, structure)
                                       → 通过 tabled 的 Color 配置把样式应用到单元格
```

**代码证据**：
- `crates/nu-table/src/types/general.rs` 第 220–229 行（`get_string_value`：样式分离存储）
- `crates/nu-table/src/types/general.rs` 第 34–35 行（第二遍 `colorize_space`）
- `crates/nu-table/src/table.rs` 第 579–583 行（`set_styles`：`colorize_table` 应用）

---

#### 链路 2：普通 Record 键值表（`TableView::General` + `Value::Record`）

**入口**：`crates/nu-command/src/viewers/table.rs` 第 697 行 → `JustTable::kv_table`（`crates/nu-table/src/types/general.rs` 第 47 行）

**着色方式：直接着色路径**。颜色直接写入字符串，tabled 把它当普通文本处理。

```
构建阶段（kv_table 第 52–58 行 for 循环）：
1. val.to_abbreviated_string()        → 纯文本
2. style_computer.style_primitive()   → 拿到 nu_ansi_term::Style（样式信息）
3. clean_charset                      → 文本规整化【仅 String 类型】
4. color.paint(text).to_string()      → 包裹 ANSI 前缀/后缀（真正上色）
5. colorize_space_str                 → 行首尾空格再包一层颜色【仅 String 类型】
6. table.insert((i, 1), value)        → 直接插入带 ANSI 的字符串
```

**代码证据**（`nu_value_to_string_colored`，`crates/nu-table/src/common.rs` 第 42–59 行）：

```rust
if is_string {
    text = clean_charset(&text);        // ← 先规整（L47）
}
if let Some(color) = style.color_style {
    text = color.paint(text).to_string(); // ← 后上色（L51）
}
```

**注意**：此链路**没有**第二遍 `colorize_space` 全表扫描，因为空格着色已在步骤 5 中按单元格完成。

---

#### 链路 3：Expanded 渲染（`TableView::Expanded`）

**入口**：`crates/nu-command/src/viewers/table.rs` 第 704 / 740 行 → `ExpandedTable::build_map` / `build_list`（`crates/nu-table/src/types/expanded.rs` 第 41 / 46 行）

**着色方式：混合链路**——叶子节点直接着色，嵌套节点渲染成子表后作为纯文本嵌入。

```
expand_entry（L520）/ expand_value（L451）递归遍历：

├─ 叶子值（非 Record/List）：
│  1. nu_value_to_string_clean        → 返回 (text, style)，clean_charset + colorize_space_str
│  2. (或) nu_value_to_string         → 返回 (text, style)，仅文本+样式分离
│  3. (或) nu_value_to_string_colored → 直接上色（仅 value_to_wrapped_string_clean L667）
│  4. 存入 CellOutput.styled / CellOutput.text
│
└─ 嵌套值（Record/List）：
   1. 递归调用 expanded_table_kv / expand_list → 渲染成完整子表（含主题、着色）
   2. 子表 to_string() 得到纯文本（已带 ANSI 颜色）
   3. 用 CellOutput.clean() 存入（无额外样式，样式已在文本里）
```

**代码证据**：
- 叶子直接着色：`crates/nu-table/src/types/expanded.rs` 第 522 行（`nu_value_to_string_clean`）、第 667 行（`nu_value_to_string_colored`）
- 嵌套子表内嵌：`crates/nu-table/src/types/expanded.rs` 第 466–468 行（`out.table.draw_unchecked(width)` → `CellOutput::clean`）

> **⚠️ 推断**：Expanded 链路混用了三种字符串转换函数（`nu_value_to_string` / `_clean` / `_colored`），看起来是逐步叠加功能的结果，并非统一设计。不同分支选择哪个函数似乎取决于该值是否还需要进一步处理（如 wrap 换行）。

---

#### 链路 4：Collapsed 渲染（`TableView::Collapsed`）

**入口**：`crates/nu-command/src/viewers/table.rs` 第 708 / 744 行 → `CollapsedTable::build`（`crates/nu-table/src/types/collapse.rs`）

**着色方式：树状遍历着色**——先递归遍历整个 Value 树把颜色写进字符串，再交给 tabled 展示。

```
CollapsedTable::build：
1. colorize_value(&mut value, ...)    → 递归遍历 Value 树
│     ├─ Record：每个 key 调 colorize_text(header, style.color_style)
│     │            每个 value 递归 colorize_value
│     └─ List：每个元素递归 colorize_value
│            叶子值：nu_value_to_string_clean → colorize_text(text, style.color_style)
2. NuTable 插入已着色的文本
3. configure_table → draw
```

**代码证据**：
- `crates/nu-table/src/types/collapse.rs` 第 35 行（`colorize_value` 入口）
- `crates/nu-table/src/types/collapse.rs` 第 63–64 行（叶子：`nu_value_to_string_clean` → `colorize_text`）

---

#### 四种链路的对比汇总

| 视图模式 | 着色策略 | 文本与样式关系 | 第二遍 colorize_space | 适用场景 |
|---------|---------|--------------|---------------------|---------|
| General + List | **样式分离** | 分离存储，渲染阶段合并 | ✅ 有（全表扫描） | `table` 命令主路径，最常用 |
| General + Record | **直接着色** | 颜色写进字符串 | ❌ 无 | 单个 Record 的键值展示 |
| Expanded | **混合** | 叶子着色 + 子表内嵌文本 | ❌ 无 | `table -e` 递归展开 |
| Collapsed | **树状遍历** | 递归着色后写进字符串 | ❌ 无 | `table -c` 紧凑折叠视图 |

**颜色来源汇总**（代码事实，适用于所有链路）：
- `style_primitive()`：基础类型颜色（int/float/string/bool/...）
- `compute("row_index", _)`：索引列
- `compute("header", _)`：表头
- `compute("separator", _)`：边框线颜色
- `LS_COLORS` 覆写：文件路径颜色（在 `table` 命令层通过 `path_columns` metadata 处理，位置：`crates/nu-command/src/viewers/table.rs` 第 802–839 行）

### 4.2 空格着色与 tabled padding 的边界

`crates/nu-table/src/util.rs` 第 103–175 行的 `colorize_space` 使用 fancy_regex 匹配每行首尾空格并包裹颜色。

**明确的事实**：
- 着色的是**单元格文本内部**的首尾空格。
- tabled 的 `Padding`（`crates/nu-table/src/table.rs` 第 633–635 行 `set_indent` 函数）是在单元格外部追加的补位空格，不在文本内，因此**不会被** `colorize_space` 着色。
- 当单元格文本宽度 + 左右 indent 刚好等于列宽时，视觉上左右 padding 区域显示默认背景色，而文本内部首尾空格（若有）会被着色 —— 两者颜色可能不一致。

### 4.3 主题与列宽的定量关系

不同 `TableMode` 使用的边框字符不同（都是单宽字符），但**垂直线数量**直接占用列宽预算。`get_total_width2`（`table.rs` 第 1583–1589 行）计算：

```rust
fn get_total_width2(widths: &[usize], cfg: &ColoredConfig) -> usize {
    let total = widths.iter().sum::<usize>();
    let countv = cfg.count_vertical(widths.len());   // 垂直线数量
    let margin = cfg.get_margin();
    total + countv + margin.left.size + margin.right.size
}
```

以 N 列数据为例（代码事实）：

| 主题 | 外边框 | 内部垂直线 | 总垂直线占用 | 相当于内容宽度 |
|------|--------|-----------|-------------|--------------|
| `none` (blank) | 无 | 无 | 0 | 0 |
| `light` | 无 | 无（仅 header 横线） | 0 | 0 |
| `basic` / `rounded` / `thin` | 有 | 有 | N + 1 | 约 1 列宽（当每列 ≈ 6 字符时）|
| `heavy` / `double` | 有 | 有 | N + 1 | 同上（字符更粗，字符宽度仍为 1）|

> **⚠️ 推断**：对于 80 列窄终端，切换到 `none` 或 `light` 主题可省出 N+1 ≈ 6 个字符，这一空间往往处在「能渲染完整表 vs 报 `Couldn't fit table`」的边界上。此判断基于常见列数（5–10 列）的经验估计，未做精确边界测试。

### 4.4 `SetLineHeaders` 与截断

`crates/nu-table/src/table.rs` 第 1788–1851 行是 `header_on_separator: true` 下的特殊渲染路径：表头从数据行中**剥离**（`remove_header` 函数），作为分割线标题单独绘制。

```rust
let columns = self.head.values.into_iter()
    .zip(widths.iter().cloned())
    .map(|(s, width)| Truncate::truncate(&s, width - pad).into_owned())
    .collect::<Vec<_>>();
let mut names = ColumnNames::new(columns)
    .line(self.line)
    .alignment(Alignment::from(self.head.align));
if let Some(color) = self.head.color {
    names = names.color(color);
}
```

**明确事实**：header 文字在这里使用 `Truncate` 而非 `Wrap`，所以即使全局配置了 `Wrap` 策略，分割线上的表头也不会换行。这是刻意设计，保证分割线视觉干净。

---

## 五、大表性能优化：流式分页与缩写

### 5.1 `PagingTableCreator` 迭代器

位置：`crates/nu-command/src/viewers/table.rs` 第 906–1030 行

`PagingTableCreator` 实现了 `Iterator<Item = Result<Vec<u8>>>`，每迭代一次产出一批渲染好的表文本字节，直接喂给 `ByteStream::from_result_iter`，实现流式输出。

两个关键参数（均来自 `$env.config.table`，代码证据）：

- **`stream_page_size`**：默认 1000 行（`NonZeroU16::new(1000)`，见 `crates/nu-protocol/src/config/table.rs` 第 395 行）。每批最多行数。
- **`batch_duration`**：默认 1 秒（`Duration::from_secs(1)`，见第 394 行）。即使没攒够 1000 行，到时间也强制输出，保证首屏响应。

```rust
fn stream_collect(stream, size, batch_duration, signals) -> (Vec<Value>, bool) {
    let start_time = Instant::now();
    let mut batch = Vec::with_capacity(size);
    for (i, item) in stream.enumerate() {
        batch.push(item);
        if (Instant::now() - start_time) >= batch_duration { end = false; break; }
        if i + 1 == size { end = false; break; }
        if signals.interrupted() { break; }
    }
    (batch, end)
}
```

### 5.2 `abbreviated` 缩写模式

位置：`crates/nu-command/src/viewers/table.rs` 第 1065–1107 行

当设置了 `table --abbreviated N` 或 `$env.config.table.abbreviated_row_count`，`stream_collect_abbreviated` 只保留**前 N 条 + 后 N 条**，中间插一条「...」占位行：

```rust
fn stream_collect_abbreviated(..., size: usize, ...) -> (Vec<Value>, usize, bool) {
    for item in stream {
        read += 1;
        if read <= size {
            head.push(item);
        } else if tail.len() < size {
            tail.push_back(item);
        } else {
            let _ = tail.pop_front();   // 固定大小滑窗
            tail.push_back(item);
        }
    }
    // 合并 head + [dummy] + tail
}
```

内存占用稳定在 O(2N) 而非 O(总行数)，对于百万行级表是数量级的优化。

### 5.3 维度缓存优化

`NuTable` 在数据插入阶段就预先计算了 `widths` 和 `heights`（`insert_value` 函数，`crates/nu-table/src/table.rs` 第 102–108 行），避免 tabled 内部的全表 `PeekableGridDimension::width/height` 扫描。

只有在**列被压缩**（触发 Wrap/Truncate）时，才按需对受影响列重算行高（Wrap 模式手动扫、Truncate 模式通过 `hint_change` 触发 tabled 内部重算），其余场景 `DimensionCtrl::change` 直接 `set_heights(self.heights)` 跳过计算。

### 5.4 宽度分配的时间复杂度

`distribute_available_width_round_robin`（第 1706–1742 行）每轮消耗 O(优先级列数) 的时间。常见配置下优先级列 < 10，开销可忽略。

> **⚠️ 推断**：如果用户配置了上百个优先级列，复杂度退化为 O(预算 × 列数)，可能成为瓶颈。目前未见此类配置的实际用例。

---

## 六、列宽 / 换行 / 主题 的三难选择与推荐实践

### 6.1 三个优化目标的权衡

| 优化目标 | 倾向的配置 | 代价 | 代码依据 |
|---------|-----------|------|---------|
| **信息密度最高**（每行内容全可见） | Wrap + light/none 主题 + 关 header_on_separator | 行高膨胀，总输出长度↑，滚屏慢 | `TrimStrategy::Wrap`; `TableMode` 边框占用 |
| **列数最多**（尽量把所有列塞下） | Truncate + basic/rounded 主题 + 开 header_on_separator | 每列内容被截断，需 `get` 等命令补全查看 | `truncate_columns_by_head`; `SetLineHeaders` |
| **视觉最完整**（色彩/边框/内容都完整） | Wrap + 彩色主题 + 关 header_on_separator | 三者组合占空间最多，窄终端可能直接放不下 | `colorize_space`; `load_theme` |

### 6.2 常见场景的推荐配置

#### 场景 1：日常 REPL（默认配置）
```nu
$env.config.table = {
    mode: rounded
    trim: { methodology: wrapping, wrapping_try_keep_words: true }
    header_on_separator: false
    index_mode: always
}
```
代码路径：`maybe_truncate_columns → truncate_columns_by_columns`（宽屏）或 `by_content`（窄屏），Wrap 保住信息完整性。

#### 场景 2：脚本 CI / 重定向到文件
```nu
$env.config.use_ansi_coloring = false
$env.config.table = {
    mode: basic_compact
    trim: { methodology: truncating, truncating_suffix: "…" }
    header_on_separator: false
}
```
配合 `is_color_empty` 判断（`table.rs` 第 809–820 行），`colorize_table` 全程跳过，性能最佳。

#### 场景 3：80 列终端 + 长列名（如数据库查询）
```nu
$env.config.table = {
    mode: none
    trim: { methodology: wrapping, wrapping_try_keep_words: true }
    header_on_separator: false
}
```
去掉所有边框垂直线，省出 N+1 字符宽度给内容列。

---

## 七、源码中的 TODO 与已知局限

以下条目均在代码中有明确注释标注：

1. **高度维度缺失**：`crates/nu-table/src/table.rs` 第 1–6 行 TODO 指出，表格目前仅按宽度控制，当终端高度不够时 `table -e`（expand）模式不会中途停止构建。

2. **左列优先分配未实现**：`crates/nu-table/src/table.rs` 第 1048–1057 行注释提到策略 B 可以按「左列优先多给空间」的思路（如 `[15,10,5]` 而非均匀 `[10,10,10]`），目前尚未实现。

3. **SetLineHeaders 索引列对齐**：`crates/nu-table/src/table.rs` 第 1831–1843 行 FIXME 指出，因 tabled 的 bug，`SetLineHeaders` 没有给索引列单独设置对齐方式，header 行的 `#` 与数据行的右对齐索引可能不对齐。

4. **colorize_space 两遍扫描**：`crates/nu-table/src/types/general.rs` 第 34 行 TODO 指出 `list_table` 中对空格着色做了第二遍全表扫描，应合并到构建阶段一次完成。

---

## 八、关键调用链速查表

| 功能 | 文件 | 函数 |
|------|------|------|
| 命令入口 | `crates/nu-command/src/viewers/table.rs` | `Table::run` → `handle_table_command` |
| 流式分页 | `crates/nu-command/src/viewers/table.rs` | `PagingTableCreator::next` |
| 数据填充 + 样式 | `crates/nu-table/src/types/general.rs` | `create_table_with_header_and_index` |
| 列宽策略分发 | `crates/nu-table/src/table.rs` | `maybe_truncate_columns` |
| 宽终端列均分 | `crates/nu-table/src/table.rs` | `truncate_columns_by_columns` |
| 窄终端保内容 | `crates/nu-table/src/table.rs` | `truncate_columns_by_content` |
| 表头主导模式 | `crates/nu-table/src/table.rs` | `truncate_columns_by_head` |
| 优先级列压缩 | `crates/nu-table/src/table.rs` | `compact_partial_visibility_for_priority` |
| Wrap / Truncate 应用 | `crates/nu-table/src/table.rs` | `width_ctrl_truncate` |
| 主题加载 | `crates/nu-table/src/table.rs` | `load_theme` |
| 颜色 + 对齐 | `crates/nu-table/src/table.rs` | `set_styles` |
| 表头分割线模式 | `crates/nu-table/src/table.rs` | `SetLineHeaders::change` |
| 空格着色 | `crates/nu-table/src/util.rs` | `colorize_space` |
| ANSI 安全的字符串宽度 | `crates/nu-table/src/util.rs` | `string_width`（底层 tabled `get_text_width`） |
| 主题枚举 | `crates/nu-protocol/src/config/table.rs` | `TableMode` |
| 剪裁策略枚举 | `crates/nu-protocol/src/config/table.rs` | `TrimStrategy` |
| 全部配置结构 | `crates/nu-protocol/src/config/table.rs` | `TableConfig` |
