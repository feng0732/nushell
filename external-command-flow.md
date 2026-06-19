# Nushell 外部命令执行与管道边界深度分析

## 概述

本文档深入分析 Nushell 中**值流与外部命令 stdio 的交互机制**，以及**默认模式和 pipefail 模式下的退出码检查机制**。

---

## 第一部分：核心数据结构

### 1.1 PipelineData - 管道数据

定义于 [pipeline_data.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L49-L54)

```rust
pub enum PipelineData {
    Empty,
    Value(Value, Option<PipelineMetadata>),
    ListStream(ListStream, Option<PipelineMetadata>),
    ByteStream(ByteStream, Option<PipelineMetadata>),
}
```

四种类型代表了管道数据的不同形态：
- **Empty**：空输入（无数据）
- **Value**：单个值（字符串、数字、列表等）
- **ListStream**：值流（可迭代的数据序列）
- **ByteStream**：字节流（原始二进制数据）

### 1.2 ByteStream - 字节流抽象

定义于 [byte_stream.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs#L190-L198)

```rust
pub struct ByteStream {
    stream: ByteStreamSource,
    span: Span,
    signals: Signals,
    type_: ByteStreamType,     // Binary / String / Unknown
    known_size: Option<u64>,
    caller_spans: Vec<Span>,
}
```

**数据源类型**（[byte_stream.rs:L31-L36](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs#L31-L36)）：

```rust
pub enum ByteStreamSource {
    Read(Box<dyn Read + Send + 'static>),  // 通用 reader（内部命令产生）
    File(File),                             // 文件
    #[cfg(feature = "os")]
    Child(Box<ChildProcess>),               // 子进程（外部命令产生）
}
```

### 1.3 ChildProcess - 子进程包装

定义于 [child.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs#L222-L228)

```rust
pub struct ChildProcess {
    pub stdout: Option<ChildPipe>,
    pub stderr: Option<ChildPipe>,
    exit_status: Arc<Mutex<ExitStatusFuture>>,  // 异步退出状态
    ignore_error: Arc<Mutex<bool>>,             // pipefail 忽略标记
    span: Span,
}
```

**ChildPipe 类型**：
- `ChildPipe::Pipe(PipeReader)` - 原始管道，可零拷贝传递给下一个进程
- `ChildPipe::Tee(Box<dyn Read + Send>)` - Tee 分流，经过用户空间处理

### 1.4 PipelineExecutionData - 带退出状态的管道数据

定义于 [pipeline_data.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L1116-L1120)

```rust
pub struct PipelineExecutionData {
    pub body: PipelineData,
    #[cfg(feature = "os")]
    pub exit: Vec<Option<ExitStatusGuard>>,  // 管道中所有命令的退出状态
}
```

**ExitStatusGuard**（[child.rs:L97-L122](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs#L97-L122)）：

```rust
pub struct ExitStatusGuard {
    pub exit_status_future: Arc<Mutex<ExitStatusFuture>>,
    pub ignore_error: Arc<Mutex<bool>>,  // true 表示此命令的非零退出不触发 pipefail
    pub span: Option<Span>,
}
```

### 1.5 OutDest - 输出目标枚举

定义于 [out_dest.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/out_dest.rs#L5-L40)

| 变体 | 说明 | 对外部命令的行为 |
|------|------|-----------------|
| `Pipe` | 管道输出（合并流） | stdout 和 stderr 都设为 `Stdio::piped()`，且合并为一个流 |
| `PipeSeparate` | 管道输出（分离流） | stdout 和 stderr 分别设为 `Stdio::piped()`，保持分离 |
| `Value` | 收集为 Value | 设为 `Stdio::piped()`，后续收集为 Value |
| `Null` | 丢弃 | 设为 `Stdio::null()` |
| `Inherit` | 继承 nushell 的 stdio | 设为 `Stdio::inherit()`，直接输出到终端 |
| `Print` | 打印 | 设为 `Stdio::inherit()`，同时对 ListStream/Value 也打印 |
| `File` | 重定向到文件 | 将文件句柄作为 stdout/stderr |

---

## 第二部分：值流接入外部命令 stdin 的机制

### 2.1 stdin 配置总览

外部命令的 stdin 配置发生在 [run_external.rs:L243-L274](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/run_external.rs#L243-L274)。

核心逻辑是：**优先零拷贝直连，失败则回退到用户空间拷贝**。

```rust
let data_to_copy_into_stdin = match input {
    // 快速路径：ByteStream 尝试零拷贝转换
    PipelineData::ByteStream(stream, metadata) => match stream.into_stdio() {
        Ok(stdin) => {
            command.stdin(stdin);  // 直接传递文件描述符
            None
        }
        Err(stream) => {
            command.stdin(Stdio::piped());  // 创建管道
            Some(PipelineData::byte_stream(stream, metadata))  // 需要后续拷贝
        }
    },
    // 空输入
    PipelineData::Empty => {
        command.stdin(Stdio::inherit());  // 或 Stdio::null() for MCP
        None
    },
    // 其他类型（Value / ListStream）：必须用户空间拷贝
    value => {
        command.stdin(Stdio::piped());
        Some(value)
    }
};
```

### 2.2 零拷贝条件：ByteStream::into_stdio()

定义于 [byte_stream.rs:L559-L579](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs#L559-L579)

```rust
pub fn into_stdio(mut self) -> Result<Stdio, Self> {
    match self.stream {
        ByteStreamSource::Read(..) => Err(self),        // ❌ 通用 reader 无法零拷贝
        ByteStreamSource::File(file) => Ok(file.into()), // ✅ 文件可以零拷贝
        ByteStreamSource::Child(child) => {
            if let ChildProcess {
                stdout: Some(ChildPipe::Pipe(stdout)),   // ✅ 只有 ChildPipe::Pipe 可以零拷贝
                stderr, ..
            } = *child {
                debug_assert!(stderr.is_none());
                Ok(stdout.into())  // 直接传递管道文件描述符
            } else {
                self.stream = ByteStreamSource::Child(child);
                Err(self)  // ❌ ChildPipe::Tee 无法零拷贝
            }
        }
    }
}
```

**零拷贝的充要条件**：
1. 输入类型为 `ByteStream`
2. 数据源为 `ByteStreamSource::File` 或 `ByteStreamSource::Child`
3. 如果是 `Child`，其 stdout 必须是 `ChildPipe::Pipe` 类型（不能是 Tee）

### 2.3 用户空间拷贝：external stdin worker

当无法零拷贝时，创建独立线程处理 stdin 写入（[run_external.rs:L307-L325](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/run_external.rs#L307-L325)）：

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

**write_pipeline_data 的分层处理**（[run_external.rs:L490-L521](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/run_external.rs#L490-L521)）：

| 输入类型 | 处理方式 |
|---------|---------|
| `ByteStream` | 调用 `stream.write_to(writer)`，逐字节拷贝 |
| `Value::Binary` | 直接写入字节数组 |
| 其他 Value / ListStream | 通过 `table` 命令格式化后逐行写入 |

### 2.4 stdin 数据流向图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        外部命令 stdin 接入路径                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  输入类型         零拷贝?    处理方式            性能                │
│  ─────────────────────────────────────────────────────────────────  │
│                                                                     │
│  ByteStream(Child)    ✅      直接传递 fd        最高（零拷贝）       │
│    └─ ChildPipe::Pipe                                                │
│                                                                     │
│  ByteStream(File)     ✅      直接传递 fd        最高（零拷贝）       │
│                                                                     │
│  ByteStream(Child)    ❌      用户空间线程拷贝   中等                 │
│    └─ ChildPipe::Tee                                                │
│                                                                     │
│  ByteStream(Read)     ❌      用户空间线程拷贝   中等                 │
│    (内部命令输出)                                                    │
│                                                                     │
│  Value(List/..)      ❌      table 格式化+拷贝     较低              │
│  ListStream                                                         │
│                                                                     │
│  Empty                —        Stdio::inherit/null   —              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 第三部分：外部命令 stdout/stderr 接入值流的机制

### 3.1 stdout/stderr 配置

外部命令启动前配置 stdout/stderr（[run_external.rs:L205-L241](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/run_external.rs#L205-L241)）：

```rust
let stdout = stack.stdout();
let stderr = stack.stderr();

// 特殊情况：stdout 和 stderr 都为 Pipe 时，合并为一个流
let merged_stream = if matches!(stdout, OutDest::Pipe) && matches!(stderr, OutDest::Pipe) {
    let (reader, writer) = os_pipe::pipe()?;
    command.stdout(writer.try_clone()?);
    command.stderr(writer);       // stdout 和 stderr 写入同一个管道
    Some(reader)
} else {
    command.stdout(Stdio::try_from(stdout)?);
    command.stderr(Stdio::try_from(stderr)?);
    None
};
```

### 3.2 ChildProcess 包装

外部命令 spawn 后，包装为 `ChildProcess`（[run_external.rs:L330-L344](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/run_external.rs#L330-L344)）：

```rust
let child = ChildProcess::new(
    child,              // ForegroundChild
    merged_stream,      // 合并后的流（如有）
    matches!(stderr, OutDest::Pipe),  // 是否交换 stdout/stderr
    call.head,
    Some(PostWaitCallback::for_job_control(...)),
)?;
```

**ChildProcess::new 内部逻辑**（[child.rs:L281-L343](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs#L281-L343)）：

```rust
pub fn new(
    mut child: ForegroundChild,
    reader: Option<PipeReader>,  // 合并流
    swap: bool,                  // 是否交换 stdout/stderr
    span: Span,
    callback: Option<PostWaitCallback>,
) -> Result<Self, ShellError> {
    let (stdout, stderr) = match reader {
        Some(combined) => (Some(combined), None),  // 合并模式
        None => {
            let stdout = child.as_mut().stdout.take().map(convert_file);
            let stderr = child.as_mut().stderr.take().map(convert_file);
            if swap { (stderr, stdout) } else { (stdout, stderr) }
        }
    };

    // 启动 exit status waiter 线程
    let (exit_status_sender, exit_status) = mpsc::channel();
    thread::Builder::new()
        .name("exit status waiter".into())
        .spawn(move || {
            let matched = match child.wait() {
                Ok(wait_status) => {
                    let next = match &wait_status {
                        ForegroundWaitStatus::Frozen(_) => ExitStatus::Exited(0),
                        ForegroundWaitStatus::Finished(exit_status) => *exit_status,
                    };
                    if let Some(callback) = callback {
                        (callback.0)(wait_status);
                    }
                    Ok(next)
                }
                Err(err) => Err(err),
            };
            exit_status_sender.send(matched)
        })?;

    Ok(Self::from_raw(stdout, stderr, Some(exit_status), span))
}
```

### 3.3 ByteStream::child() 桥接

`ChildProcess` 被包装为 `ByteStream`（[byte_stream.rs:L356-L363](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs#L356-L363)）：

```rust
pub fn child(child: ChildProcess, span: Span) -> Self {
    Self::new(
        ByteStreamSource::Child(Box::new(child)),
        span,
        Signals::empty(),       // 子进程有自己的信号处理，不使用 Signals
        ByteStreamType::Unknown, // 外部命令输出类型未知
    )
}
```

最终返回给管道的是：

```rust
Ok(PipelineData::byte_stream(
    ByteStream::child(child, call.head),
    None,
))
```

### 3.4 输出消费：drain_to_out_dests

管道结束时，根据 `OutDest` 决定如何消费输出（[pipeline_data.rs:L302-L327](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L302-L327)）：

```rust
pub fn drain_to_out_dests(
    mut self,
    engine_state: &EngineState,
    stack: &mut Stack,
) -> Result<Self, ShellError> {
    match stack.pipe_stdout().unwrap_or(&OutDest::Inherit) {
        OutDest::Print => {
            self.print_table(engine_state, stack, false, false)?;
            Ok(Self::Empty)
        }
        OutDest::Pipe | OutDest::PipeSeparate => Ok(self),  // 不消费，传递给下游
        OutDest::Value => {
            let span = self.span().unwrap_or(Span::unknown());
            self.into_value(span).map(|val| Self::Value(val, metadata))
        }
        OutDest::File(file) => {
            self.write_to(file.as_ref())?;
            Ok(Self::Empty)
        }
        OutDest::Null | OutDest::Inherit => {
            self.drain()?;    // 排空流（触发 wait()）
            Ok(Self::Empty)
        }
    }
}
```

### 3.5 ByteStream::drain() 与进程等待

`ByteStream::drain()` 会排空输出并等待进程退出（[byte_stream.rs:L683-L694](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs#L683-L694)）：

```rust
pub fn drain(self) -> Result<(), ShellError> {
    match self.stream {
        ByteStreamSource::Read(read) => {
            copy_with_signals(read, io::sink(), self.span, &self.signals)?;
            Ok(())
        }
        ByteStreamSource::File(_) => Ok(()),  // 文件无需 drain
        ByteStreamSource::Child(child) => child.wait(), // 等待子进程退出
    }
}
```

**ChildProcess::wait()**（[child.rs:L408-L454](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs#L408-L454)）会：
1. 读取并丢弃所有 stdout/stderr 数据（如果有的话）
2. 调用 `exit_status.wait()` 阻塞等待进程退出
3. 调用 `check_ok()` 检查退出状态

### 3.6 stdout/stderr 数据流向图

```
┌──────────────────────────────────────────────────────────────────────┐
│                     外部命令 stdout 输出流向                          │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  外部进程 stdout → ChildProcess → ByteStream → 下游消费               │
│                              │                                        │
│                              ├── 零拷贝传递给下一个外部命令             │
│                              │    (ChildPipe::Pipe → into_stdio())     │
│                              │                                        │
│                              ├── 收集为 Value                          │
│                              │    (into_value / into_string)           │
│                              │                                        │
│                              ├── 写入文件                              │
│                              │    (write_to)                           │
│                              │                                        │
│                              ├── 打印/丢弃                             │
│                              │    (print / drain)                      │
│                              │                                        │
│                              └── 错误检查                              │
│                                   (check_ok + pipefail)               │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 第四部分：退出码检查机制

### 4.1 退出状态检查：check_ok()

定义于 [child.rs:L48-L89](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs#L48-L89)

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
                    ShellError::CoreDumped { signal_name, signal, span }
                } else {
                    ShellError::TerminatedBySignal { signal_name, signal, span }
                })
            }
        }
    }
}
```

### 4.2 LAST_EXIT_CODE 的设置

`LAST_EXIT_CODE` 是一个环境变量，通过 `Stack::set_last_exit_code` 设置（[stack.rs:L309-L319](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/engine/stack.rs#L309-L319)）：

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

**设置时机**：
| 场景 | 代码位置 | 行为 |
|------|---------|------|
| 源执行成功 | [util.rs:L253](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-cli/src/util.rs#L253) | 设置为 0 或实际退出码 |
| ByteStream drain 成功 | [eval_ir.rs:L1739](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L1739) | 设置为 0 |
| 发生错误 | [stack.rs:L313-L319](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/engine/stack.rs#L313-L319) | 从 ShellError 提取退出码 |
| ignore 命令 | [ignore.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-cmd-lang/src/core_commands/ignore.rs) | 根据配置设置 0 或 1 |

### 4.3 默认模式下的退出码检查

**默认模式（pipefail 关闭）**：
- 只关心**最后一个命令**的退出码
- 管道中上游命令的非零退出码被忽略
- `LAST_EXIT_CODE` 总是反映最后一个命令的退出码

**检查点**：
1. `ByteStream::drain()` 时调用 `ChildProcess::wait()` → `check_ok()`
2. 如果最后一个命令非零退出，drain 返回错误
3. 错误被设置到 `LAST_EXIT_CODE`

### 4.4 Pipefail 模式下的退出码检查

Pipefail 由 `nu_experimental::PIPE_FAIL` 控制，是实验性功能。

#### 4.4.1 退出状态追踪：PipelineExecutionData

`PipelineExecutionData` 的 `exit` 字段是一个 `Vec<Option<ExitStatusGuard>>`，按**执行顺序**保存管道中所有命令的退出状态。

**退出状态的累积**发生在 `Instruction::Call` 处理中（[eval_ir.rs:L675-L713](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L675-L713)）：

```rust
Instruction::Call { decl_id, src_dst } => {
    let input = ctx.take_reg(*src_dst);
    let input_data = input.body;
    let result = eval_call::<D>(ctx, *decl_id, *span, input_data)?;

    #[cfg(feature = "os")]
    {
        let mut original_exit = input.exit;  // 上游命令的退出状态列表

        // complete 命令：清空继承的退出状态
        if ctx.engine_state.get_decl(*decl_id).name() == "complete" {
            original_exit.clear();
        }

        // 追加当前命令的退出状态
        let result_exit_status_future = result
            .clone_exit_status_future()
            .map(|f| f.with_span(*span));
        original_exit.push(result_exit_status_future);

        ctx.put_reg(*src_dst, PipelineExecutionData {
            body: result,
            exit: original_exit,  // 累积所有命令的退出状态
        });
    }
}
```

**clone_exit_status_future** 从 PipelineData 提取退出状态（[pipeline_data.rs:L871-L883](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L871-L883)）：

```rust
pub fn clone_exit_status_future(&self) -> Option<ExitStatusGuard> {
    match self {
        PipelineData::Empty | PipelineData::Value(..) | PipelineData::ListStream(..) => None,
        PipelineData::ByteStream(stream, ..) => match stream.source() {
            ByteStreamSource::Read(..) | ByteStreamSource::File(..) => None,
            ByteStreamSource::Child(c) => {
                let exit_future = c.clone_exit_status_future();
                let ignore_error = c.clone_ignore_error();
                Some(ExitStatusGuard::new(exit_future, ignore_error))
            }
        },
    }
}
```

**只有 `ByteStreamSource::Child` 类型才有退出状态**，其他类型返回 None。

#### 4.4.2 Pipefail 检查：check_exit_status_future()

定义于 [child.rs:L22-L29](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs#L22-L29)

```rust
pub fn check_exit_status_future(
    exit_status: Vec<Option<ExitStatusGuard>>,
) -> Result<(), ShellError> {
    // 反向遍历（从最后一个命令向前）
    for one_status in exit_status.into_iter().rev().flatten() {
        check_exit_status_future_ok(one_status)?
    }
    Ok(())
}
```

**检查顺序**：从最后一个命令向前遍历，遇到第一个非零退出就报错。

#### 4.4.3 Pipefail 检查时机

Pipefail 在以下时机被检查：

| 检查点 | 代码位置 | 说明 |
|-------|---------|------|
| `collect()` | [eval_ir.rs:L1707-L1709](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L1707-L1709) | 收集为 Value 时 |
| `drain()` | [eval_ir.rs:L1761-L1768](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L1761-L1768) | 排空管道时 |
| `drain_if_end()` | [eval_ir.rs:L1783-L1790](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L1783-L1790) | 块结束时 |
| 源执行完毕 | [util.rs:L337-L342](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-cli/src/util.rs#L337-L342) | REPL 执行完命令后 |

#### 4.4.4 collect_reg 与退出状态

`collect_reg` 用于将寄存器内容收集为 Value（赋值场景），它会**清空退出状态**（[eval_ir.rs:L208-L221](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L208-L221)）：

```rust
fn collect_reg(&mut self, reg_id: RegId, fallback_span: Span) -> Result<Value, ShellError> {
    #[cfg(feature = "os")]
    let body = {
        let mut data = self.take_reg(reg_id);
        data.exit.clear();  // 清空退出状态，赋值不触发 pipefail
        data.body
    };
    let span = body.span().unwrap_or(fallback_span);
    body.into_value(span)
}
```

**这意味着**：`let x = (external1 | external2)` 中，即使 pipefail 开启，赋值语句也不会触发 pipefail 错误。

#### 4.4.5 complete 命令的特殊处理

`complete` 命令将外部命令的退出状态转换为数据（`exit_code` 字段），它会**清空继承的退出状态**（[eval_ir.rs:L696-L698](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L696-L698)）：

```rust
if ctx.engine_state.get_decl(*decl_id).name() == "complete" {
    original_exit.clear();  // 下游不再检查这些退出状态
}
```

### 4.5 ignore_error 标志

`ignore_error` 标志用于控制单个命令的退出码是否触发 pipefail：

- **`ignore` 命令**：将下游命令的 `ignore_error` 设为 true
- **赋值语句**：通过 `collect_reg` 清空退出状态（效果类似但机制不同）
- **`complete` 命令**：清空继承的退出状态

### 4.6 退出码检查对比表

| 场景 | 默认模式 | Pipefail 模式 |
|------|---------|--------------|
| 最后一个命令非零退出 | ❌ 报错 | ❌ 报错 |
| 中间命令非零退出 | ✅ 忽略 | ❌ 报错 |
| `ignore` 包裹的命令 | ✅ 忽略 | ✅ 忽略（ignore_error=true） |
| 赋值语句中的管道 | ✅ 忽略（收集为值） | ✅ 忽略（collect_reg 清空 exit） |
| `complete` 命令后 | ✅ 忽略 | ✅ 忽略（exit 被清空） |
| SIGPIPE 信号退出 | ✅ 忽略 | ✅ 忽略 |
| 其他信号退出 | ❌ 报错 | ❌ 报错 |

---

## 第五部分：完整管道执行流程

以 `echo "hello" | grep "h" | wc -c` 为例，完整追踪管道执行和退出码检查。

### 5.1 执行流程

```
阶段 1：IR 评估开始
  └─ eval_ir_block()
     ├─ 初始化寄存器
     └─ registers[0] = input (PipelineExecutionData::from(PipelineData::Empty))

阶段 2：执行 echo "hello"
  └─ Instruction::Call { decl_id: "run-external", src_dst: 0 }
     ├─ input = take_reg(0) → exit: []
     ├─ eval_call() → spawn echo 进程
     ├─ result = PipelineData::ByteStream(ChildProcess)
     ├─ result.exit_status_future = Some(exit_status_future_echo)
     ├─ original_exit = [None] (input.exit 的内容)
     ├─ original_exit.push(Some(ExitStatusGuard::new(echo_future, ignore=false)))
     └─ put_reg(0, { body: result, exit: [None, Some(echo_guard)] })

阶段 3：执行 grep "h"
  └─ Instruction::Call { decl_id: "run-external", src_dst: 0 }
     ├─ input = take_reg(0) → exit: [None, Some(echo_guard)]
     ├─ stream.into_stdio() → 成功（零拷贝传递 echo 的 stdout）
     ├─ eval_call() → spawn grep 进程
     ├─ result = PipelineData::ByteStream(ChildProcess)
     ├─ result.exit_status_future = Some(exit_status_future_grep)
     ├─ original_exit = [None, Some(echo_guard)]
     ├─ original_exit.push(Some(ExitStatusGuard::new(grep_future, ignore=false)))
     └─ put_reg(0, { body: result, exit: [None, Some(echo_guard), Some(grep_guard)] })

阶段 4：执行 wc -c
  └─ Instruction::Call { decl_id: "run-external", src_dst: 0 }
     ├─ input = take_reg(0) → exit: [None, Some(echo_guard), Some(grep_guard)]
     ├─ stream.into_stdio() → 成功（零拷贝传递 grep 的 stdout）
     ├─ eval_call() → spawn wc 进程
     ├─ result = PipelineData::ByteStream(ChildProcess)
     ├─ result.exit_status_future = Some(exit_status_future_wc)
     ├─ original_exit = [None, Some(echo_guard), Some(grep_guard)]
     ├─ original_exit.push(Some(ExitStatusGuard::new(wc_future, ignore=false)))
     └─ put_reg(0, { body: result, exit: [None, echo, grep, wc] })

阶段 5：管道结束（drain_if_end）
  └─ drain_if_end()
     ├─ data.body.drain_to_out_dests()
     │  └─ OutDest::Inherit → ByteStream::drain()
     │     └─ ChildProcess::wait()
     │        └─ 读取所有 stdout → wait wc 进程 → check_ok(wc_status)
     └─ 检查 pipefail（如果启用）：
        └─ check_exit_status_future(exit: [None, echo, grep, wc])
           ├─ 反向遍历
           ├─ wc: check_ok() → 如果非零，立即报错
           ├─ grep: check_ok() → 如果非零，立即报错
           └─ echo: check_ok() → 如果非零，立即报错
```

### 5.2 退出状态的累积与传播

```
管道: cmd1 | cmd2 | cmd3

exit 向量: [ None,     // Empty 输入产生
             Some(cmd1_guard),
             Some(cmd2_guard),
             Some(cmd3_guard) ]

pipefail 检查顺序: cmd3 → cmd2 → cmd1 (反向)

LAST_EXIT_CODE: 始终等于最后一个命令（cmd3）的退出码
```

---

## 第六部分：边界问题与设计权衡

### 6.1 管道边界的模糊点

1. **`Pipe` vs `PipeSeparate` 的语义边界**
   - `Pipe`：stdout/stderr 合并流，用于 `o+e>|`
   - `PipeSeparate`：分离流，用于 `e>|` 配合 `complete`
   - 只有一个流重定向时，使用 `Pipe` 还是 `PipeSeparate`？

2. **零拷贝的条件限制**
   - 仅 `ChildPipe::Pipe` 支持零拷贝，`ChildPipe::Tee` 不支持
   - `tee` 命令会破坏零拷贝链
   - 内部命令产生的 `ByteStreamSource::Read` 类型无法零拷贝

3. **合并流的信息丢失**
   - stdout 和 stderr 合并后，下游无法区分原始来源
   - 这是 `Pipe` 模式的设计取舍

### 6.2 退出码的边界

1. **`LAST_EXIT_CODE` 与 pipefail 的关系**
   - `LAST_EXIT_CODE` 只反映最后一个命令的退出码
   - pipefail 是额外的错误检查机制
   - pipefail 报错后，`LAST_EXIT_CODE` 仍为最后一个命令的值

2. **ignore_error 的传播**
   - `ignore_error` 是每个命令独立的标志
   - `ignore` 命令设置后续命令的 ignore_error
   - 赋值语句通过清空 exit 向量来"忽略"错误

3. **信号退出码**
   - Unix 下信号终止的进程，退出码为 `-signal`
   - 如 `SIGINT=2` → 退出码 `-2`
   - `SIGPIPE` 特殊处理：不视为错误

### 6.3 设计权衡总结

| 设计决策 | 优点 | 缺点 |
|---------|------|------|
| ByteStream 三级抽象 | 统一接口，支持多种源 | 零拷贝有条件限制 |
| PipelineExecutionData 累积退出状态 | 实现 pipefail 不需要修改命令代码 | 内存开销，需要 careful 维护 |
| collect_reg 清空 exit | 赋值语句行为符合预期 | 机制不够直观 |
| complete 清空 exit | 语义正确（已转为数据） | 硬编码命令名，不够优雅 |
| 反向遍历检查 pipefail | 第一个错误就是最后一个命令的，符合直觉 | — |

---

## 附录：关键代码索引

| 功能 | 文件 | 行号 |
|------|------|------|
| 外部命令执行入口 | [run_external.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/run_external.rs) | L52-L350 |
| stdin 配置逻辑 | [run_external.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/run_external.rs) | L243-L274 |
| stdout/stderr 配置逻辑 | [run_external.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/run_external.rs) | L205-L241 |
| 零拷贝转换 into_stdio | [byte_stream.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs) | L559-L579 |
| stdin 写入线程 | [run_external.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/run_external.rs) | L307-L325 |
| write_pipeline_data | [run_external.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/run_external.rs) | L490-L521 |
| ChildProcess 构造 | [child.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs) | L281-L343 |
| 退出状态检查 check_ok | [child.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs) | L48-L89 |
| pipefail 检查 | [child.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs) | L22-L29 |
| PipelineExecutionData | [pipeline_data.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs) | L1116-L1163 |
| clone_exit_status_future | [pipeline_data.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs) | L871-L883 |
| drain_to_out_dests | [pipeline_data.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs) | L302-L327 |
| ByteStream::drain | [byte_stream.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs) | L683-L694 |
| LAST_EXIT_CODE 设置 | [stack.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/engine/stack.rs) | L309-L319 |
| Call 指令处理 | [eval_ir.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs) | L675-L713 |
| collect_reg | [eval_ir.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs) | L208-L221 |
| drain pipefail 检查 | [eval_ir.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs) | L1761-L1768 |
| drain_if_end | [eval_ir.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs) | L1774-L1793 |
| 输出目标枚举 | [out_dest.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/out_dest.rs) | L5-L40 |
