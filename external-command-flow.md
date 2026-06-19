# Nushell 外部命令执行与 Shell 管道边界分析

## 概述

本文档分析 Nushell 中外部命令执行与 Shell 管道混合时的边界处理机制，包括 stdio 重定向、退出码处理和信号处理三个核心部分。

---

## 1. 核心数据结构

### 1.1 OutDest - 输出目标枚举

定义于 [out_dest.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/out_dest.rs#L5-L40)

```rust
pub enum OutDest {
    Pipe,           // 管道输出，合并 stdout/stderr
    PipeSeparate,   // 管道输出，分离 stdout/stderr
    Value,          // 收集为 Value
    Null,           // 丢弃到 /dev/null
    Inherit,        // 继承 nushell 的 stdio
    Print,          // 打印到 nushell 的 stdout/stderr
    File(Arc<File>),// 重定向到文件
}
```

**关键转换**：`OutDest` → `Stdio` 的转换逻辑（[out_dest.rs:L54-L64](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/out_dest.rs#L54-L64)）：
- `Pipe` / `PipeSeparate` / `Value` → `Stdio::piped()`
- `Null` → `Stdio::null()`
- `Print` / `Inherit` → `Stdio::inherit()`
- `File` → 克隆文件句柄

### 1.2 StackOutDest - 栈级输出目标管理

定义于 [stack_out_dest.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/engine/stack_out_dest.rs#L31-L60)

```rust
pub(crate) struct StackOutDest {
    pub pipe_stdout: Option<OutDest>,    // 下一个命令的 stdout 管道
    pub pipe_stderr: Option<OutDest>,    // 下一个命令的 stderr 管道
    pub stdout: OutDest,                 // 无管道时的 stdout 目标（仅 File/Inherit）
    pub stderr: OutDest,                 // 无管道时的 stderr 目标（仅 File/Inherit）
    pub parent_stdout: Option<OutDest>,  // 命令参数求值时的父 stdout
    pub parent_stderr: Option<OutDest>,  // 命令参数求值时的父 stderr
}
```

**优先级规则**（[stack_out_dest.rs:L79-L90](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/engine/stack_out_dest.rs#L79-L90)）：
1. `pipe_stdout` / `pipe_stderr`（管道重定向，最高优先级）
2. `stdout` / `stderr`（文件重定向）
3. `OutDest::Inherit`（继承进程 stdio，最低优先级）

### 1.3 ByteStream - 字节流抽象

定义于 [byte_stream.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs#L190-L198)

```rust
pub struct ByteStream {
    stream: ByteStreamSource,  // 数据源
    span: Span,
    signals: Signals,          // 中断信号
    type_: ByteStreamType,     // 类型标记：Binary/String/Unknown
    known_size: Option<u64>,
    caller_spans: Vec<Span>,
}
```

**数据源类型**（[byte_stream.rs:L31-L36](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs#L31-L36)）：
```rust
pub enum ByteStreamSource {
    Read(Box<dyn Read + Send + 'static>),  // 通用 reader
    File(File),                             // 文件
    #[cfg(feature = "os")]
    Child(Box<ChildProcess>),               // 子进程
}
```

### 1.4 ChildProcess - 子进程包装

定义于 [child.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs#L222-L228)

```rust
pub struct ChildProcess {
    pub stdout: Option<ChildPipe>,
    pub stderr: Option<ChildPipe>,
    exit_status: Arc<Mutex<ExitStatusFuture>>,  // 异步退出状态
    ignore_error: Arc<Mutex<bool>>,             // 是否忽略错误（用于 pipefail）
    span: Span,
}
```

**退出状态未来**（[child.rs:L124-L128](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs#L124-L128)）：
```rust
pub enum ExitStatusFuture {
    Finished(Result<ExitStatus, Box<ShellError>>),
    Running(Receiver<io::Result<ExitStatus>>),
}
```

### 1.5 ExitStatus - 退出状态

定义于 [exit_status.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-system/src/exit_status.rs#L4-L11)

```rust
pub enum ExitStatus {
    Exited(i32),
    #[cfg(unix)]
    Signaled {
        signal: i32,
        core_dumped: bool,
    },
}
```

---

## 2. stdio 处理与管道边界

### 2.1 外部命令执行入口

核心函数：[External::run()](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/run_external.rs#L52-L350)

#### 2.1.1 stdout/stderr 配置（[run_external.rs:L205-L241](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/run_external.rs#L205-L241)）

```rust
let stdout = stack.stdout();  // 从 StackOutDest 解析最终目标
let stderr = stack.stderr();

// 特殊情况：stdout 和 stderr 都需要 Pipe 时，合并为一个流
let merged_stream = if matches!(stdout, OutDest::Pipe) && matches!(stderr, OutDest::Pipe) {
    let (reader, writer) = os_pipe::pipe()?;
    command.stdout(writer.try_clone()?);
    command.stderr(writer);
    Some(reader)
} else {
    // 分别配置 stdout 和 stderr
    command.stdout(Stdio::try_from(stdout)?);
    command.stderr(Stdio::try_from(stderr)?);
    None
};
```

**管道边界关键点**：
- 当 `OutDest::Pipe` 同时作用于 stdout 和 stderr 时，使用 `os_pipe::pipe()` 创建一个合并流
- 这意味着 `o+e>|` 重定向会将两个流合并后传递给下一个命令
- 后台作业会强制将 `Inherit`/`Print` 转为 `Stdio::null()`

#### 2.1.2 stdin 配置（[run_external.rs:L243-L274](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/run_external.rs#L243-L274)）

```rust
let data_to_copy_into_stdin = match input {
    // 快速路径：ByteStream 可直接转为 Stdio（零拷贝）
    PipelineData::ByteStream(stream, metadata) => match stream.into_stdio() {
        Ok(stdin) => {
            command.stdin(stdin);
            None
        }
        Err(stream) => {
            command.stdin(Stdio::piped());
            Some(PipelineData::byte_stream(stream, metadata))
        }
    },
    // 空输入
    PipelineData::Empty => {
        command.stdin(Stdio::inherit());  // 或 Stdio::null() for MCP
        None
    }
    // 其他类型（Value/ListStream）：需要线程拷贝
    value => {
        command.stdin(Stdio::piped());
        Some(value)
    }
};
```

**零拷贝管道优化**（[byte_stream.rs:L559-L579](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs#L559-L579)）：

```rust
pub fn into_stdio(mut self) -> Result<Stdio, Self> {
    match self.stream {
        ByteStreamSource::Read(..) => Err(self),  // 通用 reader 无法直接传递
        ByteStreamSource::File(file) => Ok(file.into()),  // 文件可以直接传递
        ByteStreamSource::Child(child) => {
            // 只有 ChildPipe::Pipe 类型可以直接传递给下一个进程
            if let ChildProcess {
                stdout: Some(ChildPipe::Pipe(stdout)),
                ..
            } = *child {
                Ok(stdout.into())  // 零拷贝：直接传递管道文件描述符
            } else {
                Err(self)  // Tee 类型需要经过用户空间处理
            }
        }
    }
}
```

**管道边界图示**：

```
外部命令A → ByteStream(ChildProcess, ChildPipe::Pipe) → 外部命令B
    │                                                          │
    └── 直接传递文件描述符（零拷贝） ──────────────────────────┘

外部命令A → ByteStream(ChildProcess, ChildPipe::Tee) → 外部命令B
    │                                                          │
    └── read() → 用户空间缓冲 → write() ─────────────────────┘

内部命令 → ByteStream(Read) → 外部命令
    │                                                          │
    └── read() → 用户空间缓冲 → write() ─────────────────────┘
```

#### 2.1.3 stdin 数据拷贝线程（[run_external.rs:L307-L325](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/run_external.rs#L307-L325)）

当无法零拷贝时，创建独立线程处理 stdin 写入：

```rust
if let Some(data) = data_to_copy_into_stdin {
    let stdin = child.as_mut().stdin.take().expect("stdin is piped");
    thread::Builder::new()
        .name("external stdin worker".into())
        .spawn(move || {
            let _ = write_pipeline_data(engine_state, stack, data, stdin);
        })?;
}
```

**PipelineData 写入逻辑**（[run_external.rs:L490-L521](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/run_external.rs#L490-L521)）：
- `ByteStream` → 直接 `stream.write_to(writer)`
- `Value::Binary` → 直接写入字节
- 其他类型 → 通过 `table` 命令格式化后写入

### 2.2 StackIoGuard - RAII 重定向保护

定义于 [stack_out_dest.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/engine/stack_out_dest.rs#L103-L185)

```rust
pub struct StackIoGuard<'a> {
    stack: &'a mut Stack,
    old_pipe_stdout: Option<OutDest>,
    old_pipe_stderr: Option<OutDest>,
    old_parent_stdout: Option<OutDest>,
    old_parent_stderr: Option<OutDest>,
}

impl Drop for StackIoGuard<'_> {
    fn drop(&mut self) {
        // 恢复所有旧的输出目标设置
        self.out_dest.pipe_stdout = self.old_pipe_stdout.take();
        self.out_dest.pipe_stderr = self.old_pipe_stderr.take();
        // ... 恢复 parent stdout/stderr
    }
}
```

**作用**：确保命令执行完毕后，栈的输出目标状态被正确恢复，避免跨命令污染。

---

## 3. 退出码处理

### 3.1 退出码设置流程

#### 3.1.1 Stack::set_last_exit_code（[stack.rs:L309-L319](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/engine/stack.rs#L309-L319)）

```rust
pub fn set_last_exit_code(&mut self, code: i32, span: Span) {
    self.add_env_var("LAST_EXIT_CODE".into(), Value::int(code.into(), span));
}

pub fn set_last_error(&mut self, error: &ShellError) {
    if let Some(code) = error.external_exit_code() {
        self.set_last_exit_code(code.item, code.span);
    } else if let Some(code) = error.exit_code() {
        self.set_last_exit_code(code, Span::unknown());
    }
}
```

#### 3.1.2 退出状态检查（[child.rs:L48-L89](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs#L48-L89)）

```rust
pub fn check_ok(status: ExitStatus, ignore_error: bool, span: Span) -> Result<(), ShellError> {
    match status {
        ExitStatus::Exited(exit_code) => {
            if ignore_error {
                Ok(())
            } else if let Ok(exit_code) = exit_code.try_into() {
                Err(ShellError::NonZeroExitCode { exit_code, span })
            } else {
                Ok(())
            }
        }
        #[cfg(unix)]
        ExitStatus::Signaled { signal, core_dumped } => {
            let sig = Signal::try_from(signal);
            if sig == Ok(Signal::SIGPIPE) || (ignore_error && !core_dumped) {
                Ok(())  // SIGPIPE 不视为错误
            } else {
                Err(if core_dumped {
                    ShellError::CoreDumped { .. }
                } else {
                    ShellError::TerminatedBySignal { .. }
                })
            }
        }
    }
}
```

### 3.2 Pipefail 机制

Pipefail 是实验性功能，由 `nu_experimental::PIPE_FAIL` 控制。

#### 3.2.1 ExitStatusGuard（[child.rs:L97-L122](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs#L97-L122)）

```rust
pub struct ExitStatusGuard {
    pub exit_status_future: Arc<Mutex<ExitStatusFuture>>,
    pub ignore_error: Arc<Mutex<bool>>,  // 是否忽略此命令的错误
    pub span: Option<Span>,
}
```

#### 3.2.2 管道退出状态检查（[child.rs:L22-L29](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs#L22-L29)）

```rust
pub fn check_exit_status_future(exit_status: Vec<Option<ExitStatusGuard>>) -> Result<(), ShellError> {
    // 反向遍历管道中的每个命令的退出状态
    for one_status in exit_status.into_iter().rev().flatten() {
        check_exit_status_future_ok(one_status)?
    }
    Ok(())
}
```

**检查时机**：
1. `collect()` 时（[eval_ir.rs:L1707-L1709](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L1707-L1709)）
2. `drain()` 时（[eval_ir.rs:L1761-L1768](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L1761-L1768)）
3. `drain_if_end()` 时（[eval_ir.rs:L1783-L1790](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L1783-L1790)）
4. 源执行完毕后（[util.rs:L337-L342](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-cli/src/util.rs#L337-L342)）

### 3.3 退出码设置时机

| 场景 | 代码位置 | 说明 |
|------|---------|------|
| 源执行成功 | [util.rs:L253](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-cli/src/util.rs#L253) | 设置为 0 或实际退出码 |
| ByteStream drain 成功 | [eval_ir.rs:L1739](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L1739) | 设置为 0 |
| 错误发生时 | [stack.rs:L313-L319](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/engine/stack.rs#L313-L319) | 从 ShellError 提取退出码 |
| ignore 命令 | [ignore.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-cmd-lang/src/core_commands/ignore.rs) | 根据配置设置 0 或 1 |

---

## 4. 信号处理

### 4.1 Signals 结构

定义于 [pipeline/signals.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/signals.rs#L13-L93)

```rust
pub struct Signals {
    signals: Option<Arc<AtomicBool>>,  // 中断标志（None 表示永不中断）
}

impl Signals {
    pub fn check(&self, span: &Span) -> Result<(), ShellError> {
        if self.interrupted() {
            Err(ShellError::Interrupted { span: *span })
        } else {
            Ok(())
        }
    }

    pub fn interrupted(&self) -> bool {
        self.signals
            .as_deref()
            .is_some_and(|b| b.load(Ordering::Relaxed))
    }
}
```

### 4.2 Ctrl+C 处理初始化

定义于 [signals.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/src/signals.rs#L7-L34)

```rust
pub(crate) fn ctrlc_protection(engine_state: &mut EngineState) {
    let interrupt = Arc::new(AtomicBool::new(false));
    engine_state.set_signals(Signals::new(interrupt.clone()));

    let signal_handlers = Handlers::new();

    // 注册处理器：中断时杀死所有后台作业
    signal_handlers.register_unguarded(Box::new(move |action| {
        if action == SignalAction::Interrupt && let Ok(mut jobs) = jobs.lock() {
            let _ = jobs.kill_all();
        }
    }))?;

    // 设置全局 Ctrl+C 处理器
    ctrlc::set_handler(move || {
        interrupt.store(true, Ordering::Relaxed);
        signal_handlers.run(SignalAction::Interrupt);
    })?;
}
```

### 4.3 Unix 进程组与信号

#### 4.3.1 ForegroundChild - 前台进程管理

定义于 [foreground.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-system/src/foreground.rs#L34-L42)

```rust
pub struct ForegroundChild {
    inner: Child,
    #[cfg(unix)]
    pipeline_state: Option<Arc<(AtomicU32, AtomicU32)>>,  // (pgrp, pcnt)
    #[cfg(unix)]
    interactive: bool,
}
```

#### 4.3.2 子进程创建前的信号设置（[foreground.rs:L371-L398](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-system/src/foreground.rs#L371-L398)）

```rust
pub fn prepare_command(external_command: &mut Command, existing_pgrp: u32, background: bool) {
    unsafe {
        external_command.pre_exec(move || {
            // 设置进程组和前台
            set_foreground_pid(Pid::this(), existing_pgrp, background);

            // 将信号处理重置为默认行为（因为 nushell 可能忽略了它们）
            let default = SigAction::new(SigHandler::SigDfl, SaFlags::empty(), SigSet::empty());
            let _ = sigaction(Signal::SIGQUIT, &default);
            let _ = sigaction(Signal::SIGTSTP, &default);
            let _ = sigaction(Signal::SIGTERM, &default);

            Ok(())
        });
    }
}
```

**关键设计**：
- `SIGPIPE` 已由 `std::process` 重置为 `SIG_DFL`
- 子进程在 `pre_exec` 钩子中重置 `SIGQUIT`、`SIGTSTP`、`SIGTERM` 为默认行为
- 这确保子进程能正确响应终端信号

#### 4.3.3 进程组管理（[foreground.rs:L408-L424](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-system/src/foreground.rs#L408-L424)）

```rust
fn set_foreground_pid(pid: Pid, existing_pgrp: u32, background: bool) {
    let pgrp = if existing_pgrp == 0 {
        pid  // 新管道：创建新进程组
    } else {
        Pid::from_raw(existing_pgrp as i32)  // 已有管道：加入现有进程组
    };
    let _ = unistd::setpgid(pid, pgrp);

    if !background {
        let _ = unistd::tcsetpgrp(unsafe { stdin_fd() }, pgrp);
    }
}
```

**管道中多个外部命令的进程组关系**：

```
管道: external1 | external2 | external3

进程组创建流程:
1. external1 spawn: existing_pgrp=0 → pgrp=pid1 → 设为前台
2. external2 spawn: existing_pgrp=pid1 → 加入 pid1 进程组
3. external3 spawn: existing_pgrp=pid1 → 加入 pid1 进程组

结果: 所有外部命令属于同一个进程组，终端信号发送给整个组
```

#### 4.3.4 Drop 清理（[foreground.rs:L201-L213](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-system/src/foreground.rs#L201-L213)）

```rust
impl Drop for ForegroundChild {
    fn drop(&mut self) {
        if let Some((pgrp, pcnt)) = self.pipeline_state.as_deref()
            && pcnt.fetch_sub(1, Ordering::SeqCst) == 1
        {
            pgrp.store(0, Ordering::SeqCst);
            if self.interactive {
                child_pgroup::reset()  // 将前台归还 nushell
            }
        }
    }
}
```

### 4.4 信号检查点

`Signals::check()` 在以下关键位置被调用：
- 字节流拷贝：`copy_with_signals()`
- 行迭代：`ByteStream::lines()`
- Glob 扩展：`expand_glob()` 循环中
- 长循环操作中定期检查

---

## 5. 完整执行流程

### 5.1 外部命令执行完整流程

以 `echo "hello" | grep "h" | wc -c` 为例：

```
1. 解析管道，创建 PipelineExecution 结构
   ├─ 为每个命令准备 Redirection
   └─ 第一个命令 stdout=Pipe，最后一个命令 stdout=Inherit

2. 执行 echo "hello" (run_external)
   ├─ stdin: PipelineData::Empty → Stdio::inherit()
   ├─ stdout: OutDest::Pipe → Stdio::piped()
   ├─ spawn 进程 → ForegroundChild
   ├─ 创建 ChildProcess，启动 exit status waiter 线程
   └─ 返回 PipelineData::ByteStream(ByteStreamSource::Child(...))

3. ByteStream 传递给 grep "h"
   ├─ 调用 stream.into_stdio() → 成功（ChildPipe::Pipe）
   ├─ command.stdin(stdio)  // 零拷贝！
   ├─ 同样 spawn 进程，返回新的 ByteStream
   └─ exit_status_future 保存在 PipelineExecutionData

4. ByteStream 传递给 wc -c
   ├─ 同样零拷贝 stdin
   ├─ stdout: OutDest::Inherit → Stdio::inherit()
   └─ spawn 进程

5. 管道结束，drain 输出
   ├─ 调用 ByteStream::drain() → ChildProcess::wait()
   └─ 检查 pipefail：反向遍历所有 ExitStatusGuard
```

### 5.2 退出码处理时机

```
external1 | external2 | external3

执行顺序:
1. external1 启动 → 立即返回 ByteStream
2. external2 启动 → 立即返回 ByteStream
3. external3 启动 → 立即返回 ByteStream
4. drain external3 的输出 → wait() 得到 exit code
5. 检查 pipefail（如果启用）:
   ├─ wait external3 → 检查退出码
   ├─ wait external2 → 检查退出码
   └─ wait external1 → 检查退出码
6. 设置 LAST_EXIT_CODE 为最后一个命令的退出码
```

---

## 6. 边界问题与设计权衡

### 6.1 管道边界的模糊点

1. **`OutDest::Pipe` vs `OutDest::PipeSeparate`**
   - `Pipe`：stdout 和 stderr 合并流（`o+e>|`）
   - `PipeSeparate`：分离流（`e>|` 配合 `complete` 命令）
   - 边界：当只有 stdout 或 stderr 重定向时使用哪个？

2. **零拷贝的条件限制**
   - 仅 `ByteStreamSource::File` 和 `ByteStreamSource::Child` + `ChildPipe::Pipe` 支持
   - `ChildPipe::Tee` 和 `ByteStreamSource::Read` 必须经过用户空间
   - 这意味着 `external | tee foo.nu | external` 会有额外拷贝

3. **合并流的处理**
   - 当 stdout 和 stderr 都设为 `Pipe` 时，合并为一个流
   - 下游无法区分原始流来源

### 6.2 信号处理的边界

1. **进程组生命周期**
   - `pipeline_state` 中的 `pcnt` 原子计数跟踪进程组成员
   - 最后一个成员 drop 时重置 pgrp 为 0，归还终端控制权

2. **SIGPIPE 特殊处理**
   - `check_ok()` 中 `SIGPIPE` 不视为错误
   - 因为管道中上游命令退出时，下游收到 `SIGPIPE` 是正常行为

3. **子进程信号继承**
   - nushell 忽略的信号（如 `SIGINT` 由 `ctrlc` crate 处理）
   - 子进程在 `pre_exec` 中重置为默认行为
   - 但 `SIGINT` 处理比较特殊：由终端发送给整个前台进程组

### 6.3 退出码的边界

1. **`ignore_error` 标志**
   - 用于 `ignore` 命令或赋值语句
   - 标记为 `ignore_error=true` 的命令，非零退出码不触发 pipefail

2. **`LAST_EXIT_CODE` 与 pipefail 的关系**
   - `LAST_EXIT_CODE` 总是设置为最后一个命令的退出码
   - pipefail 是额外的错误检查，不影响 `LAST_EXIT_CODE` 的值

3. **信号退出码**
   - Unix 下被信号终止的进程，退出码为 `-signal`
   - 如 `SIGINT=2` → 退出码 `-2`

---

## 7. 关键代码索引

| 功能 | 文件 | 行号 |
|------|------|------|
| 外部命令执行入口 | [run_external.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/run_external.rs) | L52-L350 |
| stdio 合并逻辑 | [run_external.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/run_external.rs) | L205-L241 |
| 零拷贝管道转换 | [byte_stream.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs) | L559-L579 |
| 子进程包装 | [child.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs) | L280-L343 |
| 退出状态检查 | [child.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs) | L48-L89 |
| 前台进程管理 | [foreground.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-system/src/foreground.rs) | L50-L92 |
| 进程组信号设置 | [foreground.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-system/src/foreground.rs) | L371-L424 |
| 输出目标枚举 | [out_dest.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/out_dest.rs) | L5-L40 |
| 栈输出管理 | [stack_out_dest.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/engine/stack_out_dest.rs) | L31-L185 |
| 退出码设置 | [stack.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/engine/stack.rs) | L309-L319 |
| Pipefail 检查 | [eval_ir.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs) | L1761-L1790 |
| Ctrl+C 初始化 | [signals.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/src/signals.rs) | L7-L34 |
| 信号检查结构 | [pipeline/signals.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/signals.rs) | L13-L93 |
