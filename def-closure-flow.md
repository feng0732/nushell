# Nushell 自定义命令与闭包变量捕获流程

本文档从代码实现角度梳理 Nushell 中自定义命令（`def`）和闭包捕获变量之间的关系，覆盖定义、调用和递归三大场景。

---

## 一、核心数据结构

### 1.1 Block（代码块）

所有自定义命令的主体和闭包最终都以 `Block` 的形式存储在引擎状态中。

定义位置：[block.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-protocol/src/ast/block.rs#L5-L14)

```rust
pub struct Block {
    pub signature: Box<Signature>,       // 函数签名（参数定义）
    pub pipelines: Vec<Pipeline>,        // 具体代码内容
    pub captures: Vec<(VarId, Span)>,    // **需要捕获的变量列表**（编译时确定）
    pub redirect_env: bool,              // 是否将环境变量回传给调用方（def --env）
    pub ir_block: Option<IrBlock>,       // 编译后的 IR 指令
    pub span: Option<Span>,
}
```

关键点：`captures` 字段在**解析阶段**被填充，记录了该 Block 需要从外部作用域"捕获"的所有变量的 `VarId` 及其定义位置。

### 1.2 Closure（闭包值）

闭包作为运行时的 `Value`，在求值时产生，与 `Block` 的区别在于它携带了**变量的实际值快照**。

定义位置：[closure.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-protocol/src/engine/closure.rs#L10-L14)

```rust
pub struct Closure {
    pub block_id: BlockId,                  // 指向哪个 Block
    pub captures: Vec<(VarId, Value)>,      // **已捕获的变量值**（运行时快照）
}
```

关键点：`captures` 这里存的是变量的**值**，而不是 ID+Span。这是闭包与普通代码块的根本区别。

### 1.3 Expr::Block vs Expr::Closure

在 AST 表达式层，二者都是 `BlockId`，但语义不同：

定义位置：[expr.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-protocol/src/ast/expr.rs#L31-L32)

- `Expr::Block(BlockId)`：普通代码块，如 `if` 的分支、循环体等
- `Expr::Closure(BlockId)`：闭包字面量，如 `{ |x| $x + 1 }`、`def` 的函数体

二者在变量捕获计算时处理逻辑略有不同（见下文）。

### 1.4 Stack（运行时栈）

Stack 是变量在运行时的载体。

定义位置：[stack.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-protocol/src/engine/stack.rs#L38-L67)

```rust
pub struct Stack {
    pub vars: Vec<(VarId, Value)>,        // 当前栈的局部变量
    pub recursion_count: u64,             // 递归深度计数器
    pub parent_stack: Option<Arc<Stack>>, // 父栈（用于 with_parent 场景）
    // ... 环境变量、输出重定向等
}
```

Stack 设计理念：不使用传统的帧式调用栈，而是通过"捕获变量快照"的方式创建新 Stack（见 [stack.rs 注释](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-protocol/src/engine/stack.rs#L21-L36)）。

---

## 二、自定义命令定义流程

### 2.1 解析概述

自定义命令的解析分两个阶段：

1. **预声明（predeclaration）**：先把命令名登记在册，使后续代码（包括函数体内部）可以引用它
2. **完整解析**：解析签名、函数体，计算捕获变量，生成最终的 Command

### 2.2 预声明阶段（parse_def_predecl）

实现位置：[parse_def_predecl](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-parser/src/parse_def.rs#L51-L161)

核心流程：
1. 识别 `def` / `export def` / `extern` 关键字
2. 解析命令名，验证合法性
3. `working_set.enter_scope()` → 进入新作用域解析签名
4. `parse_full_signature()` → 只提取签名信息（参数名、类型）
5. `working_set.exit_scope()` → 退出作用域
6. 构造最小化的 `Signature`，通过 `add_predecl()` 登记

这一步的目的：**让递归和互相递归的函数在解析时能找到彼此**。例如：

```nu
def bob [] { sam }    # 解析 bob 时 sam 还未定义，但 predecl 会先登记
def sam [] { 3 }
bob
```

测试验证：[predecl_check](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/tests/repl/test_custom_commands.rs#L170-L172)

### 2.3 完整解析阶段（parse_def_inner）

实现位置：[parse_def_inner](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-parser/src/parse_def.rs#L402-L651)

关键步骤：

1. **进入新作用域**：`working_set.enter_scope()`（第 442 行）
2. **通过 parse_internal_call 解析 def 调用**：将 def 本身当作内置命令来解析，提取出三个位置参数：
   - 位置 0：命令名（字符串）
   - 位置 1：签名 `[a: int, --flag]`
   - 位置 2：函数体闭包 `{ ... }`
3. **编译函数体 Block**：

```rust
// 第 494-496 行
Some(Expression { expr: Expr::Closure(block_id), .. }) => {
    compile_block_with_id(working_set, *block_id);    // AST → IR
    *working_set.get_block_mut(*block_id).signature = sig.clone();
}
```

4. **将 Block 包装为 Custom Command**：

```rust
// 第 613-615 行
*declaration = signature
    .clone()
    .into_block_command(block_id, attribute_vals, examples);
```

此时生成的 `Command` 实现了 `block_id()` 方法，返回这个闭包 Block 的 ID，调用时通过它找到 Block。

5. **设置 redirect_env**：

```rust
// 第 619 行
block.redirect_env = has_env;  // def --env 才为 true
```

### 2.4 变量捕获计算（discover_captures_in_closure）

这是整个机制最核心的部分。捕获分析在 `parse()` 函数的最后阶段进行。

实现位置：[parse_captures_compile.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-parser/src/parse_captures_compile.rs#L512-L622)

核心思想：**遍历整个 AST，对每个 Block/Closure，找出它引用了哪些在外部定义的变量**。

#### 2.4.1 算法流程

`discover_captures_in_closure` 函数（第 47-81 行）：

1. 先把 Block 自身签名中定义的所有参数变量加入 `seen` 列表（"已在本作用域定义"）
2. 遍历 Block 的所有 Pipeline
3. 递归进入每个 Expression

`discover_captures_in_expr` 函数（第 154-448 行）处理各种表达式类型：

| 表达式类型 | 处理方式 |
|-----------|---------|
| `Expr::Var(var_id)` | 如果 var_id 不在 `seen` 中，且不是内置变量（$env/$nu/$in），加入 `output` 作为捕获 |
| `Expr::VarDecl(var_id)` | 将 var_id 加入 `seen`（本作用域新定义的变量） |
| `Expr::Closure(block_id)` | 新建独立 `seen` 列表，递归分析子闭包，然后把子闭包需要但本闭包未定义的变量上抛 |
| `Expr::Block(block_id)` | 同上，但不检查可变变量捕获限制 |
| `Expr::Call(call)` | **关键**：如果被调用的是自定义命令（有 block_id），递归分析那个 Block 的捕获需求并传播 |
| `Expr::Collect(var_id, expr)` | var_id 加入 seen，然后分析 expr |
| `Expr::MatchBlock` | 模式中定义的变量加入 seen |
| 其他（字面量、运算符等） | 递归进入子表达式 |

#### 2.4.2 自定义命令调用的捕获传播

这是容易被忽略的关键点。当函数体内调用另一个自定义命令时：

```rust
// 第 229-280 行
Expr::Call(call) => {
    let decl = working_set.get_decl(call.decl_id);
    if let Some(block_id) = decl.block_id() {
        // 被调用者也是自定义命令，分析它需要什么捕获
        // 如果那些捕获变量本函数也未定义，就需要继续向上传播
    }
}
```

这确保了捕获需求能**跨函数边界传播**。

#### 2.4.3 结果写回

分析完成后，将捕获列表写入每个 Block：

```rust
// 第 596-618 行
for (block_id, captures) in seen_blocks.into_iter() {
    if !captures.is_empty()
        && block_captures_empty
        && block_id.get() >= working_set.permanent_state.num_blocks()  // 避免修改永久状态中的 Block
    {
        let block = working_set.get_block_mut(block_id);
        block.captures = captures;
    }
}
```

注意条件 `block_id.get() >= working_set.permanent_state.num_blocks()`：这是为了**防止递归定义场景下修改已经固化的 Block**（第 605-611 行注释中有详细说明）。

#### 2.4.4 可变变量捕获限制

```rust
// 第 186-192 行
for (var_id, span) in results.iter() {
    if !seen.contains(var_id)
        && let Some(variable) = working_set.get_variable_if_possible(*var_id)
        && variable.mutable
    {
        return Err(ParseError::CaptureOfMutableVar(*span));
    }
}
```

**闭包不允许捕获 `mut` 定义的可变变量**，因为闭包值可能在迭代器等场景中被多次调用，语义上会产生混乱。

---

## 三、闭包求值流程

### 3.1 闭包值的产生（eval_row_condition_or_closure）

当表达式求值遇到 `Expr::Closure(block_id)` 时：

实现位置：[eval.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-engine/src/eval.rs#L810-L835)

```rust
fn eval_row_condition_or_closure(...) -> Result<Value, ShellError> {
    let captures = engine_state
        .get_block(block_id)
        .captures
        .iter()
        .map(|(id, span)| {
            stack
                .get_var(*id, *span)  // 从当前栈读取变量值
                .or_else(|_| engine_state.get_var(*id).const_val.clone()...)
                .map(|var| (*id, var))
        })
        .collect::<Result<_, _>>()?;

    Ok(Value::closure(Closure { block_id, captures }, span))
}
```

关键动作：**在此时刻快照所有捕获变量的值**，存入 `Closure.captures`。之后闭包值带着这些值可以到处传递。

### 3.2 Stack::captures_to_stack —— 从闭包值创建执行栈

实现位置：[stack.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-protocol/src/engine/stack.rs#L331-L357)

```rust
pub fn captures_to_stack_preserve_out_dest(&self, captures: Vec<(VarId, Value)>) -> Stack {
    Stack {
        vars: captures,                    // 闭包快照的值直接作为新栈的变量
        env_vars: self.env_vars.clone(),   // 环境变量继承
        recursion_count: self.recursion_count,
        parent_stack: None,                // 注意：无父栈！
        // ...
    }
}
```

这就是 Nushell 的"闭包模型"：不是通过作用域链向上查找，而是在创建闭包值时**把需要的变量值复制一份**。

### 3.3 ClosureEval —— 闭包求值封装

实现位置：[closure_eval.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-engine/src/closure_eval.rs)

`ClosureEval`（可多次调用）和 `ClosureEvalOnce`（单次调用）是外部命令（如 `each`, `filter`, `map`）调用闭包的标准接口。

以 `ClosureEvalOnce::new` 为例（第 209-224 行）：

```rust
pub fn new(engine_state: &'a EngineState, stack: &Stack, closure: Closure) -> Self {
    let block = engine_state.get_block(closure.block_id);
    let callee_stack = stack.captures_to_stack(closure.captures);  // 用捕获值创建新栈
    let call_eval = CallEval::new(callee_stack, ...);
    Self { engine_state, block, call_eval, caller_stack: None }
}
```

添加参数（第 280-284 行）：
```rust
pub fn add_arg(mut self, value: Value) -> Result<Self, ShellError> {
    self.call_eval.add_positional(&self.block.signature, Cow::Owned(value))?;
    Ok(self)
}
```

最终执行（第 298-304 行）：
```rust
pub fn run_with_input(mut self, input: PipelineData) -> Result<PipelineData, ShellError> {
    self.call_eval.run(self.engine_state, self.block, input)
}
```

### 3.4 CallEval —— 统一的调用参数处理

`CallEval` 为自定义命令调用和闭包调用提供统一的参数绑定逻辑。

实现位置：[eval.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-engine/src/eval.rs#L29-L289)

`finalize_arguments` 方法（第 221-288 行）处理：
- 必填位置参数缺失报错
- 可选参数默认值填充
- rest 参数列表组装
- 命名参数/开关默认值填充

---

## 四、自定义命令调用流程

自定义命令的调用有两条路径：**AST 求值路径**和 **IR 求值路径**。

### 4.1 AST 求值路径（eval_call）

实现位置：[eval_call](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-engine/src/eval.rs#L292-L366)

```rust
pub fn eval_call<D: DebugContext>(
    engine_state: &EngineState,
    caller_stack: &mut Stack,
    call: &Call,
    input: PipelineData,
) -> Result<PipelineData, ShellError> {
    if let Some(block_id) = decl.block_id() {
        // 自定义命令分支
        let block = engine_state.get_block(block_id);

        // ① 收集捕获变量，创建被调用栈
        let mut callee_stack = caller_stack.gather_captures(engine_state, &block.captures);

        // ② 递归深度检查
        callee_stack.recursion_count += 1;
        if callee_stack.recursion_count > maximum_call_stack_depth {
            return Err(ShellError::RecursionLimitReached { ... });
        }

        // ③ 组装 CallEval
        let mut call_eval = CallEval::new(callee_stack, ...);

        // ④ 求值位置参数（在 caller_stack 上！）
        for arg in call.positional_iter() {
            let result = eval_expression::<D>(engine_state, caller_stack, arg)?;
            call_eval.add_positional(&decl.signature(), Cow::Owned(result))?;
        }

        // ⑤ 求值命名参数
        for call_named in call.named_iter() { ... }

        // ⑥ 执行
        let result = call_eval.run(engine_state, block, input);

        // ⑦ 环境变量回传（def --env）
        if block.redirect_env {
            call_eval.redirect_env(engine_state, caller_stack);
        }

        result
    } else {
        // 内置命令分支：直接 decl.run()
        decl.run(engine_state, caller_stack, &call.into(), input)
    }
}
```

关键点：参数求值在 **caller_stack** 上完成，然后把值传给 **callee_stack**。

### 4.2 Stack::gather_captures —— 自定义命令专用

实现位置：[stack.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-protocol/src/engine/stack.rs#L359-L393)

```rust
pub fn gather_captures(&self, engine_state: &EngineState, captures: &[(VarId, Span)]) -> Stack {
    let mut vars = Vec::with_capacity(captures.len());
    for (capture, _) in captures {
        if let Ok(value) = self.get_var(*capture, fake_span) {
            vars.push((*capture, value));
        } else if let Some(const_val) = &engine_state.get_var(*capture).const_val {
            vars.push((*capture, const_val.clone()));
        }
    }
    // 创建新栈，vars = 捕获到的值
    Stack { vars, ..., parent_stack: None, recursion_count: self.recursion_count }
}
```

与 `captures_to_stack` 的区别：`gather_captures` 是**从当前栈读取值**（动态查找），而 `captures_to_stack` 是**使用闭包中已经快照好的值**。

### 4.3 IR 求值路径（eval_ir.rs）

IR 路径中 `Instruction::Call` 最终调用 `eval_call` 函数。

实现位置：[eval_ir.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-engine/src/eval_ir.rs#L1213-L1312)

```rust
fn eval_call<D: DebugContext>(...) -> Result<PipelineData, ShellError> {
    if let Some(block_id) = decl.block_id() {
        let block = engine_state.get_block(block_id);

        // ① 收集捕获
        let mut callee_stack = caller_stack.gather_captures(engine_state, &block.captures);

        // ② 从 Argument Stack 取出参数并绑定到 callee_stack
        gather_arguments(engine_state, block, &mut caller_stack, &mut callee_stack, ...);

        // ③ 递归计数 +1
        callee_stack.recursion_count += 1;

        // ④ 执行 Block
        let result = eval_block_with_early_return::<D>(
            engine_state, &mut callee_stack, block, input
        ).map(|p| p.body);

        // ⑤ 环境变量回传
        if block.redirect_env {
            redirect_env(engine_state, &mut caller_stack, &callee_stack);
        }

        result
    }
}
```

IR 路径与 AST 路径的主要区别：参数不是通过重新求值表达式得到的，而是从 **Argument Stack**（编译时 Push，运行时 Pop）中取出。

---

## 五、递归场景分析

### 5.1 递归深度限制

两处递归深度检查形成双重保护：

| 位置 | 检查时机 |
|------|---------|
| [eval.rs:314-322](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-engine/src/eval.rs#L314-L322) | AST 路径调用前（callee_stack 创建后） |
| [eval_ir.rs:53-63](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-engine/src/eval_ir.rs#L53-L63) | IR 路径 eval_ir_block 入口处 |

```rust
let maximum_call_stack_depth: u64 = engine_state.config.recursion_limit as u64;
callee_stack.recursion_count += 1;
if callee_stack.recursion_count > maximum_call_stack_depth {
    return Err(ShellError::RecursionLimitReached { ... });
}
```

默认限制为 50。测试验证：[infinite_recursion_does_not_panic](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/tests/repl/test_custom_commands.rs#L223-L228)

### 5.2 递归与变量捕获

考虑经典的阶乘函数：

```nu
def fact [n: int] {
    if $n <= 1 { 1 } else { $n * (fact ($n - 1)) }
}
fact 5
```

追踪 `fact 5` 的调用过程：

1. **第一次调用 `fact 5`**
   - `gather_captures`：fact 的 Block.captures 为空（`$n` 是参数，非外部变量）
   - callee_stack.vars = `[(n_var_id, Value::int(5))]`
   - recursion_count = 1
   - 执行条件判断，进入 else 分支

2. **求值 `fact ($n - 1)`（即 `fact 4`）**
   - 先在 caller_stack 上求值 `$n - 1` → `4`
   - 新 callee_stack 创建：vars = `[(n_var_id, Value::int(4))]`
   - recursion_count = 2
   - ... 以此类推，直到 n=1

3. **n=1 时返回 1，递归展开**
   - 每层的 `$n` 都是独立的 Stack 副本，互不干扰
   - 返回时各层的 callee_stack 被丢弃，caller_stack 不受影响

### 5.3 闭包中的递归

闭包中的递归需要特殊处理，因为闭包创建时需要捕获自身引用：

```nu
let fact = { |n|
    if $n <= 1 { 1 } else { $n * ($fact ($n - 1)) }  # 引用 $fact 自身
}
```

问题：定义 `$fact` 的闭包时需要捕获 `$fact`，但 `$fact` 此时还未完成定义。

Nushell 的解析器在处理自定义命令递归时通过 **predecl** 机制解决了命令名可见性问题，但闭包变量的自引用通常需要变通方案（如使用 `def` 而非闭包值）。

### 5.4 互相递归

```nu
def is_even [n: int] { if $n == 0 { true } else { is_odd ($n - 1) } }
def is_odd  [n: int] { if $n == 0 { false } else { is_even ($n - 1) } }
is_even 4
```

**解析阶段**：
1. `parse_def_predecl` 先登记 `is_even` 和 `is_odd`（占位符）
2. 解析 `is_even` 体时，调用 `is_odd` 能找到 predecl
3. 解析 `is_odd` 体时，调用 `is_even` 也能找到 predecl
4. 计算捕获：两个 Block 的 captures 都为空（参数和互相调用均不涉及外部变量）

**运行阶段**：
- `is_even 4` → recursion_count=1 → 调用 `is_odd 3` → recursion_count=2 → ...
- 两个函数各自的参数 `$n` 在独立的 Stack 中，互不影响

测试验证：[infinite_mutual_recursion_does_not_panic](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/tests/repl/test_custom_commands.rs#L235-L240)

### 5.5 捕获分析中对递归 Block 的保护

在捕获写回阶段有一个关键判断：

```rust
// 第 612-618 行
if !captures.is_empty()
    && block_captures_empty
    && block_id.get() >= working_set.permanent_state.num_blocks()  // 只修改本次 delta 中的 Block
{
    let block = working_set.get_block_mut(block_id);
    block.captures = captures;
}
```

原因：递归定义的命令在解析过程中，其 Block 可能先被存入 permanent_state。如果此时在外部再次分析并尝试修改它的 captures，会导致逻辑错误。此条件确保只修改本次解析过程中新创建的 Block。

---

## 六、作用域隔离验证

### 6.1 自定义命令不泄漏内部变量

```nu
def foo [] { let $x = 10; $x }
foo
$x  # 错误：Variable not found
```

原因：`foo` 的 callee_stack 在调用结束后被丢弃，`$x` 存在于那个栈中，caller_stack 永远看不到。

测试：[no_scope_leak1](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/tests/repl/test_custom_commands.rs#L7-L12)

### 6.2 自定义命令不能隐式访问调用者的变量

```nu
def foo [] { $x }
def bar [] { let $x = 10; foo }
bar  # 错误：Variable not found
```

原因：`foo` 在解析时 captures 为空（它的体中没有引用外部变量），调用时 `gather_captures` 不会从 caller_stack 复制任何变量。`$x` 对 `foo` 不可见。

测试：[no_scope_leak2](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/tests/repl/test_custom_commands.rs#L14-L20)

### 6.3 显式捕获外部变量（闭包）

```nu
let $x = 10
def foo [] { $x }
foo  # 输出 10
```

原因：解析 `foo` 时，`discover_captures_in_closure` 发现 `$x` 在函数体中被引用但未在参数中定义 → 加入 captures。运行时 `gather_captures` 从 caller_stack 读取 `$x` 的值并放入 callee_stack。

测试：[simple_var_closing](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/tests/repl/test_custom_commands.rs#L165-L167)

### 6.4 参数优先于捕获

```nu
def foo [$x] { $x }
def bar [] { let $x = 10; foo 20 }
bar  # 输出 20，不是 10
```

原因：`$x` 是 `foo` 的参数，加入 `seen` 列表，不计入 captures。运行时参数绑定覆盖同名外部变量。

测试：[no_scope_leak3](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/tests/repl/test_custom_commands.rs#L23-L28)

---

## 七、总结：端到端流程图

### 7.1 定义阶段

```
源代码: def foo [a: int] { let $b = 1; $a + $x + $b }
            │
            ▼
parse_def_predecl ──→ 登记 "foo" predecl（使递归可见）
            │
            ▼
parse_def_inner
  ├─ enter_scope()
  ├─ parse_internal_call("def", "foo", "[a: int]", "{ ... }")
  │    └─ 得到 Expr::Closure(block_id) ← Block 已创建，含 pipelines
  ├─ compile_block_with_id(block_id)   ──→ AST → IR
  ├─ signature.into_block_command(block_id)  ──→ 注册为 Custom Command
  └─ exit_scope()
            │
            ▼
parse() 末尾: discover_captures_in_closure()
  ├─ 遍历 foo 的 Block
  ├─ seen = [a_var_id]                  ← 参数 a
  ├─ 发现 $a → 在 seen 中，忽略
  ├─ 发现 $b → VarDecl，加入 seen
  ├─ 发现 $x → 不在 seen 中，也不是内置变量
  │    └─ output.push((x_var_id, span))
  └─ block.captures = [(x_var_id, span)]
```

### 7.2 调用阶段

```
源代码: let $x = 100; foo 42
                      │
                      ▼
                eval_call(foo_decl_id)
                      │
                      ├─ decl.block_id() → Some(block_id)
                      ├─ block = engine_state.get_block(block_id)
                      │    └─ block.captures = [(x_var_id, span)]
                      │
                      ├─ ① gather_captures(caller_stack, captures)
                      │    └─ caller_stack.get_var(x_var_id) → Value::int(100)
                      │       callee_stack.vars = [(x_var_id, 100)]
                      │
                      ├─ ② callee_stack.recursion_count += 1
                      │
                      ├─ ③ 在 caller_stack 上求值参数: eval_expression("42") → Value::int(42)
                      │
                      ├─ ④ CallEval.add_positional(a, 42)
                      │    └─ callee_stack.vars 追加 [(a_var_id, 42)]
                      │
                      ├─ ⑤ finalize_arguments() 处理默认值/rest 等
                      │
                      ├─ ⑥ eval_block(engine_state, &mut callee_stack, block, input)
                      │    ├─ $a → callee_stack 中查找 → 42
                      │    ├─ $x → callee_stack 中查找 → 100
                      │    ├─ $b → VarDecl，加入 callee_stack → 1
                      │    └─ 42 + 100 + 1 = 143
                      │
                      └─ ⑦ 返回 PipelineData::Value(143)
                             callee_stack 被丢弃（作用域隔离）
```

### 7.3 闭包值传递阶段

```
源代码: let $x = 10; let $f = { || $x }; do $f
                      │
                      ▼
              求值 { || $x }
                      │
              eval_row_condition_or_closure()
                      ├─ block.captures = [(x_var_id, span)]
                      ├─ stack.get_var(x_var_id) → Value::int(10)
                      └─ Value::Closure(Closure {
                             block_id,
                             captures: [(x_var_id, Value::int(10))]  ← 快照！
                         })
                      │
                      ▼
              do $f 调用
                      │
              ClosureEvalOnce::new(engine_state, stack, closure)
                      ├─ stack.captures_to_stack(closure.captures)
                      │    └─ new_stack.vars = [(x_var_id, Value::int(10))]
                      ├─ CallEval 参数绑定
                      └─ eval_block → 得到 10
```

---

## 八、关键文件索引

| 模块 | 文件 | 作用 |
|------|------|------|
| 解析器 | [parse_def.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-parser/src/parse_def.rs) | `def` 命令解析 |
| 解析器 | [parse_captures_compile.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-parser/src/parse_captures_compile.rs) | 变量捕获发现算法 |
| 引擎求值 | [eval.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-engine/src/eval.rs) | AST 求值、CallEval、eval_call |
| 引擎求值 | [eval_ir.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-engine/src/eval_ir.rs) | IR 求值、递归深度检查 |
| 引擎求值 | [closure_eval.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-engine/src/closure_eval.rs) | 闭包求值封装 |
| 引擎编译 | [compile/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-engine/src/compile/mod.rs) | AST → IR 编译 |
| 引擎编译 | [compile/call.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-engine/src/compile/call.rs) | Call 指令编译 |
| 协议层 AST | [ast/block.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-protocol/src/ast/block.rs) | Block 结构定义 |
| 协议层 AST | [ast/expr.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-protocol/src/ast/expr.rs) | Expr 枚举（Block/Closure） |
| 协议层引擎 | [engine/closure.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-protocol/src/engine/closure.rs) | Closure 值结构 |
| 协议层引擎 | [engine/stack.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-protocol/src/engine/stack.rs) | Stack 实现（gather_captures、captures_to_stack） |
| 协议层引擎 | [engine/command.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/crates/nu-protocol/src/engine/command.rs) | Command trait（block_id 方法） |
| 测试 | [test_custom_commands.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/tests/repl/test_custom_commands.rs) | 自定义命令测试 |
| 测试 | [test_closures.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/67-nushell/tests/repl/test_closures.rs) | 闭包测试 |
