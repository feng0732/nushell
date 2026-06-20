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

> **阅读指引**：本节所有结论都基于同一份源码证据（见 4.2 节），表格中的 LAST_EXIT_CODE 取值都符合同一条规则——`drain()` 先设置末尾命令结果，`eval_source()` 在发生错误时再覆盖。

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

**调用链**：`check_ok()` 是所有"末尾命令退出码检查"的终点，由以下路径调用：

| 触发场景 | 调用链 | 对应代码 |
|---------|--------|---------|
| 排空 ByteStream | `drain()` → `ByteStream::drain()` → `ChildProcess::wait()` → `check_ok()` | [eval_ir.rs:L1725](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L1725) / [child.rs:L453](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs#L453) |
| 收集为 Value | `collect_reg()` / `collect()` → `into_value()` → `into_bytes()` → `check_ok()` | [eval_ir.rs:L220](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L220) / [child.rs:L403](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs#L403) |
| 管道输出打印 | `drain_to_out_dests()` → `PipelineData::drain()` → `ByteStream::drain()` → `check_ok()` | [pipeline_data.rs:L323](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L323) / [byte_stream.rs:L692](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs#L692) |

### 4.2 LAST_EXIT_CODE 的设置路径（核心证据）

`LAST_EXIT_CODE` 是环境变量，通过 `Stack::set_last_exit_code` 或 `Stack::set_last_error` 设置（[stack.rs:L309-L319](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/engine/stack.rs#L309-L319)）：

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

**整个代码库中，LAST_EXIT_CODE 只在以下 4 个调用点被显式设置**：

| 编号 | 位置 | 触发条件 | 代码 | 设置值 |
|-----|------|---------|------|--------|
| A | `drain()` [eval_ir.rs:L1726](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L1726) | `stream.drain()` **失败**（即末尾命令失败） | `ctx.stack.set_last_error(&err)` | 错误中的退出码（末尾命令的非零码，或信号码） |
| B | `drain()` [eval_ir.rs:L1739](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L1739) | `stream.drain()` **成功**（即末尾命令成功） | `ctx.stack.set_last_exit_code(0, span)` | `0` |
| C | `eval_source()` [util.rs:L253](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-cli/src/util.rs#L253) | `evaluate_source()` 返回 `Ok(failed)` | `stack.set_last_exit_code(code, Span::unknown())` | `code = false.into() = 0` |
| D | `eval_source()` [util.rs:L259](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-cli/src/util.rs#L259) | `evaluate_source()` 返回 `Err(err)` | `stack.set_last_error(&err)` | 错误中的退出码（pipefail 失败命令的退出码，或末尾命令的非零码） |

**关键观察（贯穿本节的统一规则）**：
- `drain_if_end()`、`collect()`、`collect_reg()` 都**不**设置 LAST_EXIT_CODE，它们只可能返回 `Err`，最终由 `eval_source()` 的 D 路径设置。
- LAST_EXIT_CODE 的最终取值，**取决于哪个设置点最后执行**：Drain 指令先经过 A/B，若 pipefail 检查失败会再被 D 覆盖；其他路径只有 C/D。

### 4.3 默认模式与 Pipefail 模式

#### 4.3.1 默认模式（pipefail 关闭）

只检查**最后一个命令**的退出码（通过 4.1 节中的 `check_ok()` 调用链）。`LAST_EXIT_CODE` 反映末尾命令的退出码。

#### 4.3.2 Pipefail 模式（pipefail 开启）

除检查末尾命令外，还检查**所有命令**的退出码，通过 `check_exit_status_future()` 反向遍历 exit 向量（[child.rs:L22-L29](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs#L22-L29)）：

```rust
pub fn check_exit_status_future(exit_status: Vec<Option<ExitStatusGuard>>) -> Result<(), ShellError> {
    // 反向遍历（从最后一个命令向前），遇到第一个非零就报错
    for one_status in exit_status.into_iter().rev().flatten() {
        check_exit_status_future_ok(one_status)?
    }
    Ok(())
}
```

**Pipefail 检查点**：

| 检查点 | 代码位置 | 说明 |
|-------|---------|------|
| `collect(ignore_error=false)` | [eval_ir.rs:L1707-L1709](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L1707-L1709) | TryCollect 指令 |
| `drain()` | [eval_ir.rs:L1761-L1768](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L1761-L1768) | Drain 指令（先检查末尾，再检查全部） |
| `drain_if_end()` | [eval_ir.rs:L1783-L1790](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L1783-L1790) | 块结束时 |
| `evaluate_source()` | [util.rs:L337-L342](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-cli/src/util.rs#L337-L342) | REPL 执行完后 |

**退出状态累积**发生在 `Instruction::Call`（[eval_ir.rs:L675-L713](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L675-L713)）。只有 `ByteStreamSource::Child` 类型才会产生 `ExitStatusGuard`（[pipeline_data.rs:L871-L883](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs#L871-L883)），Value / ListStream / Read / File 均为 None。

### 4.4 统一对比表（默认模式 / Pipefail / LAST_EXIT_CODE）

> 本表结论完全基于 4.2 节的 4 个设置点和 4.1 节的 check_ok 调用链。

| 场景 | 是否报错 | LAST_EXIT_CODE 最终值 | 依据（代码路径） |
|------|---------|----------------------|-----------------|
| **最后一个命令非零退出**（pipefail 关） | ❌ 报错 | 末尾命令的非零退出码 | 路径 A（Drain 指令） / 路径 D（其他路径） |
| **最后一个命令非零退出**（pipefail 开） | ❌ 报错 | 末尾命令的非零退出码 | 同左 |
| **中间命令非零、末尾命令成功**（pipefail 关） | ✅ 不报错 | `0`（末尾命令成功） | 路径 B（Drain 指令） / 路径 C（其他路径） |
| **中间命令非零、末尾命令成功**（pipefail 开） | ❌ 报错 | 中间失败命令的非零退出码 | Drain 指令：先 B 设为 0，再被 D 覆盖；其他路径：直接 D |
| **`ignore` 包裹的命令**（pipefail 关） | ✅ 不报错 | `0`（末尾命令视角） | ignore_error=true → check_ok 返回 Ok → B/C |
| **`ignore` 包裹的命令**（pipefail 开） | ✅ 不报错 | `0`（末尾命令视角） | ignore_error=true → check_ok 和 check_exit_status_future 均 Ok → B/C |
| **赋值语句中末尾命令失败**（pipefail 关） | ❌ 报错 | 末尾命令的非零退出码 | into_value → check_ok 报错 → Err 被 D 设置 |
| **赋值语句中末尾命令失败**（pipefail 开） | ❌ 报错 | 末尾命令的非零退出码 | 同左（collect_reg 清空 exit 不影响 into_value 的 check_ok） |
| **赋值语句中中间命令失败、末尾成功**（pipefail 关） | ✅ 不报错 | `0` | collect_reg 清空 exit → into_value check_ok Ok → C |
| **赋值语句中中间命令失败、末尾成功**（pipefail 开） | ✅ 不报错 | `0` | collect_reg 清空 exit → pipefail 无向量可检查 → C |
| **`complete` 命令后**（pipefail 关） | ✅ 不报错 | `0` | ignore_error=true + original_exit.clear() + wait_with_output 不调 check_ok → C |
| **`complete` 命令后**（pipefail 开） | ✅ 不报错 | `0` | 同左（exit 向量为空） |
| **SIGPIPE 信号退出** | ✅ 不报错 | `0`（被视为成功） | check_ok 中 SIGPIPE 特殊处理为 Ok → B/C |
| **其他信号退出**（SIGINT 等） | ❌ 报错 | `-signal` | check_ok 返回 Err(TerminatedBySignal) → A/D，external_exit_code 返回 `-signal` |

### 4.5 LAST_EXIT_CODE 的三条完整更新流程

以下三个场景覆盖了所有可能的 LAST_EXIT_CODE 更新路径，均引用 4.2 节的 A/B/C/D 四个设置点。

#### 4.5.1 场景一：Drain 指令触发的 pipefail

命令：`cmd1(失败, exit=1) | cmd2(成功, exit=0)`，pipefail 开，由 `Drain` 指令排空。

```
1. drain() 函数执行:
   a. stream.drain()  →  ChildProcess::wait() → check_ok(Exited(0), false, span) → Ok
   b. [路径 B] LAST_EXIT_CODE = 0

2. pipefail 检查 (L1767):
   a. check_exit_status_future([None, cmd1_guard, cmd2_guard])
   b. 反向遍历: cmd2→Ok, cmd1→Err(NonZeroExitCode{exit_code:1})
   c. 返回 Err

3. 错误向上传播，最终到 eval_source():
   a. [路径 D] set_last_error(&err) → set_last_exit_code(1)

最终: LAST_EXIT_CODE = 1  ✅（被路径 D 覆盖了路径 B 的 0）
```

#### 4.5.2 场景二：赋值语句（collect_reg）

命令：`let x = (^false | ^echo ok)`，pipefail 开。

```
1. collect_reg() 执行 [eval_ir.rs:L208-L221]:
   a. data.exit.clear()   ← 清空 pipefail 向量，跳过中间命令检查
   b. body.into_value(span)
        → ByteStream::into_value()
        → ByteStream::into_bytes()     [byte_stream.rs:L662]
        → ChildProcess::into_bytes()   [child.rs:L376]
        → collect_bytes(stdout) 成功
        → check_ok(Exited(0), false, span) → Ok

2. 赋值成功，返回 Ok(Value)

3. 最终 eval_source() [路径 C]: set_last_exit_code(0)

最终: LAST_EXIT_CODE = 0  ✅（末尾命令成功）
```

若为 `let x = ^false`，则 step 1b 的 check_ok 返回 Err，被 eval_source() 的路径 D 设为 1。

#### 4.5.3 场景三：`complete` 命令

命令：`^false | complete`，pipefail 开。

```
1. ^false 执行:
   → 生成 exit 向量: [None, false_guard]

2. Instruction::Call 执行 complete [eval_ir.rs:L696-L698]:
   → original_exit.clear()              ← 清空继承的 exit 向量
   → complete 内部:
       child.ignore_error(true)         ← 标记忽略
       child.wait_with_output()         ← 不调 check_ok，只收集
       返回 Value(Record{exit_code:1})
   → clone_exit_status_future() → None  ← Value 无退出状态
   → 新 exit 向量: [None]                ← 只有 complete 自己的 None

3. 管道输出打印 → drain_to_out_dests → Ok

4. pipefail 检查: check_exit_status_future([None]) → 全部 Ok

5. eval_source() [路径 C]: set_last_exit_code(0)

最终: LAST_EXIT_CODE = 0  ✅（不反映子进程退出码，子进程码只在 exit_code 字段中）
```

### 4.6 错误提取逻辑：external_exit_code()

定义于 [shell_error/mod.rs:L1481-L1497](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/errors/shell_error/mod.rs#L1481-L1497)

```rust
pub fn external_exit_code(&self) -> Option<Spanned<i32>> {
    let (item, span) = match *self {
        Self::NonZeroExitCode { exit_code, span } => (exit_code.into(), span),
        #[cfg(unix)]
        Self::TerminatedBySignal { signal, span, .. }
        | Self::CoreDumped { signal, span, .. } => (-signal, span),
        _ => return None,
    };
    Some(Spanned { item, span })
}

pub fn exit_code(&self) -> Option<i32> {
    match self {
        Self::Return { .. } | Self::Break { .. } | Self::Continue { .. } => None,
        _ => self.external_exit_code().map(|e| e.item).or(Some(1)),
    }
}
```

**用于 4.2 节路径 A 和 D 的 `set_last_error(&err)`**：
- `NonZeroExitCode` → 提取 `exit_code` 字段
- 信号终止 → 提取 `-signal`（如 SIGINT=2 → -2）
- 其他 ShellError → `external_exit_code()` 返回 None，`exit_code()` 回退到 `Some(1)`

### 4.7 ignore_error 标志与三种收集指令

#### 4.7.1 ignore_error 的作用

`ignore_error` 是每个命令独立的 `Arc<Mutex<bool>>`，在两处生效：
1. `check_ok()` 中：为 true 时非零退出不报错（[child.rs:L51-L52](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs#L51-L52)）
2. `check_exit_status_future_ok()` 中：同理（[child.rs:L44-L45](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs#L44-L45)）

由以下方式设置：
- `ignore` 命令：标记后续命令
- `complete` 命令：`child.ignore_error(true)` 标记子进程（[complete.rs:L70](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/complete.rs#L70)）

#### 4.7.2 三种收集指令对比

| 指令 | pipefail 检查 | 末尾命令检查 | 是否设置 LAST_EXIT_CODE |
|------|-------------|-------------|----------------------|
| `collect_reg` [eval_ir.rs:L208-L221](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L208-L221) | ❌（`exit.clear()`） | ✅（`into_value` → `check_ok`） | ❌（由 eval_source 的 C/D 设置） |
| `Collect` [eval_ir.rs:L421-L429](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L421-L429) | ❌（`collect(..., ignore_error=true)`） | ✅（`into_value` → `check_ok`） | ❌（由 eval_source 的 C/D 设置） |
| `TryCollect` [eval_ir.rs:L430-L438](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L430-L438) | ✅（`collect(..., ignore_error=false)`） | ✅（`into_value` → `check_ok`） | ❌（由 eval_source 的 C/D 设置） |

**`collect_reg` 注释辨析**：注释 "It doesn't check exit status when collecting"（[eval_ir.rs:L207](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L207)）**仅指 pipefail 的 exit 向量**，不影响末尾命令的 `into_value → check_ok`。因此 `let x = ^false` 仍会失败。

### 4.8 complete 命令的三重保护

`complete` 通过三层机制确保不报错、不污染 LAST_EXIT_CODE：

1. **`child.ignore_error(true)`**（[complete.rs:L70](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/complete.rs#L70)）：即使有人调 `check_ok` 也不会报错
2. **`wait_with_output()` 不调用 `check_ok`**（[child.rs:L478-L480](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs#L478-L480)）：只收集输出、等待退出
3. **`original_exit.clear()`**（[eval_ir.rs:L696-L698](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L696-L698)）：清空 exit 向量，pipefail 检查跳过

最终 `complete` 返回 Value(Record)，`clone_exit_status_future()` 返回 None，下游 exit 向量只有一个 None。LAST_EXIT_CODE 由路径 C 设为 `0`，子进程退出码仅通过 `exit_code` 字段暴露。

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

LAST_EXIT_CODE:
  - 默认模式（pipefail 关）: 末尾命令（cmd3）的退出码
    → 路径 B（成功=0）/ 路径 A（失败=cmd3 退出码），或路径 C/D
  - pipefail 模式（pipefail 开）: 第一个失败命令的退出码（从后往前找）
    → 若全部成功: 路径 B/C 设为 0
    → 若中间命令失败: 先 B 设为 0，再被路径 D 覆盖为失败命令退出码
（完整机制见 4.2 节 A/B/C/D 四个设置点、4.4 节统一对比表）
```

---

## 第七部分：stderr 单独入管道与完整捕获的流向

### 7.1 stderr 管道模式对比

Nushell 支持四种 stderr 管道模式，由 `OutDest` 枚举和 swap 参数控制。退出码行为与第四部分（4.2 节 A/B/C/D 设置点、4.4 节统一对比表）完全一致：

| 模式 | 语法 | stack.stdout | stack.stderr | swap | 行为 |
|------|------|-------------|-------------|------|------|
| 默认继承 | `external` | `OutDest::Inherit` | `OutDest::Inherit` | false | stderr 直接输出到终端，退出码按 4.4 节统一规则 |
| 合并管道 | `external o+e>| other` | `OutDest::Pipe` | `OutDest::Pipe` | false | stderr 与 stdout 合并后传递给下游，退出码按 4.4 节统一规则 |
| stderr 管道 | `external e>| other` | `OutDest::Inherit` | `OutDest::Pipe` | true | stderr swap 到 stdout 位置传递给下游，退出码按 4.4 节统一规则 |
| 完整捕获 | `external \| complete` | `OutDest::PipeSeparate` | `OutDest::PipeSeparate` | false | stdout 和 stderr 分离捕获，LAST_EXIT_CODE 恒为 0（见 4.8 节三重保护） |

### 7.2 OutDest::Pipe 模式（合并流）

#### 7.2.1 配置逻辑

定义于 [run_external.rs:L240-L246](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/run_external.rs#L240-L246)

```rust
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

**关键点**：
1. 使用 `os_pipe::pipe()` 创建单个管道
2. stdout 和 stderr 都重定向到同一个写端
3. 读端作为 `merged_stream` 包装到 `ChildProcess` 中
4. 下游无法区分 stdout 和 stderr

#### 7.2.2 ChildProcess 包装

```rust
let (stdout, stderr) = match reader {
    Some(combined) => (Some(combined), None),  // ⚠️ stderr 为 None
    None => {
        let stdout = child.as_mut().stdout.take().map(convert_file);
        let stderr = child.as_mut().stderr.take().map(convert_file);
        if swap { (stderr, stdout) } else { (stdout, stderr) }
    }
};
```

**合并流向图**：
```
外部进程 stdout ──┐
                  ├─ os_pipe::pipe() ──> ChildProcess.stdout ──> 下游
外部进程 stderr ──┘
                                         ChildProcess.stderr = None
```

### 7.3 stderr 管道模式（`e>|`）：swap 机制

#### 7.3.1 编译时重定向

`e>|` 的语法意味着 "仅将 stderr 管道传递给下一个命令"。编译器会做特殊处理（[compile/mod.rs:L119-L131](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/compile/mod.rs#L119-L131)）：

```rust
// 如果是 e>|，则不设置 stdout 管道重定向
if modes.out.is_none()
    && !matches!(
        element.redirection,
        Some(PipelineRedirection::Single {
            source: RedirectionSource::Stderr,
            target: RedirectionTarget::Pipe { .. }
        })
    )
{
    modes.out = Some(RedirectMode::Pipe.into_spanned(pipe_span));
}
```

对于 `e>|`，编译器只设置 `err = RedirectMode::Pipe`，不设置 `out`。

#### 7.3.2 运行时 swap

在 `run_external.rs` 中，`swap` 参数决定是否交换 stdout 和 stderr 的位置（[run_external.rs:L263](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/run_external.rs#L263)）：

```rust
let child = ChildProcess::new(
    child,
    merged_stream,
    matches!(stderr, OutDest::Pipe),  // swap = true 当 stderr 为 Pipe
    call.head,
    Some(PostWaitCallback::for_job_control(...)),
)?;
```

**对于 `e>|`**：
- `stack.stdout()` = `OutDest::Inherit` → 进程 stdout 继承终端
- `stack.stderr()` = `OutDest::Pipe` → 进程 stderr 通过管道
- `swap` = `matches!(OutDest::Pipe, OutDest::Pipe)` = **true**

#### 7.3.3 swap 在 ChildProcess 中的效果

在 [child.rs:L296-L300](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs#L296-L300)：

```rust
let (stdout, stderr) = match reader {
    Some(combined) => (Some(combined), None),  // 合并模式
    None => {
        let stdout = child.as_mut().stdout.take().map(convert_file);  // None（继承终端）
        let stderr = child.as_mut().stderr.take().map(convert_file);  // Some(PipeReader)
        if swap { (stderr, stdout) } else { (stdout, stderr) }
        //        ↑ swap=true: stderr 放入 stdout 位置！
    }
};
```

**swap 的效果**：进程的 stderr PipeReader 被放入 `ChildProcess.stdout` 位置，而原本为 None 的 stdout 被放入 `ChildProcess.stderr` 位置。

**流向图**：
```
e>| 模式:

进程 stdout ──→ Stdio::inherit() ──→ 终端
进程 stderr ──→ Stdio::piped()  ──→ PipeReader
                                       │
                         swap=true ──── ┘
                                       ↓
                                 ChildProcess.stdout = Some(PipeReader)  ← 实际是 stderr！
                                 ChildProcess.stderr = None               ← 实际是 stdout(None)

ByteStream 读取 ChildProcess.stdout → 读到的是 stderr 数据 ✅
```

#### 7.3.4 `e>|` 的零拷贝支持

由于 stderr 被交换到 `ChildProcess.stdout` 位置，`into_stdio()` 可以正常工作：

```rust
pub fn into_stdio(mut self) -> Result<Stdio, Self> {
    match self.stream {
        ByteStreamSource::Child(child) => {
            if let ChildProcess {
                stdout: Some(ChildPipe::Pipe(stdout)),  // ← 实际是 swap 后的 stderr
                stderr, ..
            } = *child {
                debug_assert!(stderr.is_none());
                Ok(stdout.into())  // ✅ 零拷贝成功！
            } else {
                Err(self)
            }
        }
        // ...
    }
}
```

**关键发现**：`e>|` 支持零拷贝！swap 机制确保 stderr 数据在 `ChildProcess.stdout` 位置，`into_stdio()` 可以直接传递文件描述符。

#### 7.3.5 `e>|` 与 `o+e>|` 的对比

| 特性 | `e>|` | `o+e>|` |
|------|-------|---------|
| stdout 去向 | 终端 | 合并到管道 |
| stderr 去向 | 管道（swap 到 stdout 位置） | 合并到管道 |
| swap | true | false |
| 合并流 | 无 | 有（os_pipe::pipe） |
| 零拷贝 | ✅ 支持 | ✅ 支持 |
| 能否区分 stdout/stderr | 不需要（只有 stderr） | ❌ 不能 |
| ChildProcess.stderr | None（原 stdout 位置） | None（合并流） |

#### 7.3.6 错误值处理：stderr_pipe_separate

当 stderr 为 `PipeSeparate`（如 `complete` 上游的配置）时，eval_call 中有特殊处理（[eval_ir.rs:L1230-L1235](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs#L1230-L1235)）：

```rust
let stderr_pipe_separate = matches!(
    redirect_err.as_ref(),
    Some(Redirection::Pipe(OutDest::PipeSeparate))
);

match result {
    // 当 stderr 是 PipeSeparate 时，错误包装为 Value 传递给下游
    Err(err) if stderr_pipe_separate => Ok(PipelineData::Value(Value::error(err, head), None)),
    result => result,
}
```

**这意味着**：`do { ^nonexistent } e>| complete` 中，如果命令不存在，错误会被包装为 `Value::Error` 传递给 complete，而不是直接抛出。

**注意**：`e>|` 使用 `OutDest::Pipe`（不是 `PipeSeparate`），所以 `stderr_pipe_separate` 为 false，错误不会被包装。

### 7.4 complete 完整捕获流程

`complete` 命令通过 `pipe_redirection()` 请求分离的 stdout 和 stderr（[complete.rs:L97-L99](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/complete.rs#L97-L99)）：

```rust
fn pipe_redirection(&self) -> (Option<OutDest>, Option<OutDest>) {
    (Some(OutDest::PipeSeparate), Some(OutDest::PipeSeparate))
}
```

#### 7.4.1 完整执行流程

以 `do { ^external arg1 } | complete` 为例：

```
阶段 1: 编译时重定向解析
  └─ compile_redirection 时，查询 complete 的 pipe_redirection()
     └─ 返回 (Some(PipeSeparate), Some(PipeSeparate))
     └─ 上游 external 的 stdout 和 stderr 都设为 PipeSeparate

阶段 2: external 执行
  ├─ stdout: OutDest::PipeSeparate → Stdio::piped()
  ├─ stderr: OutDest::PipeSeparate → Stdio::piped()
  ├─ spawn 进程
  ├─ ChildProcess {
  │    stdout: Some(PipeReader),  // 独立管道
  │    stderr: Some(PipeReader),  // 独立管道
  │    exit_status: ...,
  │    ignore_error: false,
  │  }
  └─ 返回 ByteStream(ChildProcess)

阶段 3: 管道传递
  ├─ 不触发合并逻辑（因为是 PipeSeparate，不是 Pipe）
  └─ ByteStream 传递给 complete

阶段 4: complete 执行
  ├─ stream.into_child() → 获取 ChildProcess
  ├─ child.ignore_error(true) → 标记忽略
  ├─ child.wait_with_output()
  │  ├─ stdout 线程: collect_bytes(child.stdout)
  │  ├─ stderr 线程: collect_bytes(child.stderr)
  │  └─ 等待进程退出（不检查退出码）
  ├─ 构造 record { stdout, stderr, exit_code }
  └─ 返回 PipelineData::Value(Record)
```

#### 7.4.2 wait_with_output 的并发收集

定义于 [child.rs:L464-L492](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs#L464-L492)

```rust
pub fn wait_with_output(self) -> Result<ProcessOutput, ShellError> {
    let (stdout, stderr) = match (self.stdout, self.stderr) {
        (Some(stdout), Some(stderr)) => {
            // ⚠️ 在独立线程中收集 stderr，避免死锁
            let stderr_handle = thread::Builder::new()
                .spawn(move || collect_bytes(stderr))
                .map_err(&from_io_error)?;

            // 在当前线程收集 stdout
            let stdout = collect_bytes(stdout).map_err(&from_io_error)?;

            // 等待 stderr 线程完成
            let stderr = stderr_handle
                .join()
                .map_err(...)
                .and_then(|r| r.map_err(&from_io_error))?;

            (Some(stdout), Some(stderr))
        }
        // ... 其他情况
    };

    let mut exit_status = self.exit_status.lock()...;
    let exit_status = exit_status.wait(self.span)?;  // 只等待，不检查

    Ok(ProcessOutput { stdout, stderr, exit_status })
}
```

**为什么需要独立线程？**
- 如果先收集 stdout 再收集 stderr，当 stderr 缓冲区满时，子进程会阻塞写 stderr
- 同时子进程等待 stdout 被读取，造成死锁
- 使用独立线程并发收集 stdout 和 stderr 避免死锁

### 7.5 四种 stderr 模式的对比表

> 表中的退出码与 LAST_EXIT_CODE 结论均与第四部分（4.2 节的 A/B/C/D 四个设置点，4.4 节统一对比表）完全一致。

| 特性 | 默认(Inherit) | 合并(o+e>\|) | stderr 管道(e>\|) | complete 捕获 |
|------|--------------|-------------|------------------|--------------|
| stack.stdout | Inherit | Pipe | Inherit | PipeSeparate |
| stack.stderr | Inherit | Pipe | Pipe | PipeSeparate |
| swap | false | false | true | false |
| stdout 去向 | 终端 | 合并流 | 终端 | 独立捕获 |
| stderr 去向 | 终端 | 合并流 | swap 到 stdout 位置 | 独立捕获 |
| 能否区分来源 | 能（终端） | ❌ 不能 | 不需要（只有 stderr） | ✅ 能 |
| 零拷贝传递 | N/A | ✅ 能 | ✅ 能（swap 后） | ❌ 不需要 |
| 死锁风险 | 无 | 无 | 无 | 已通过线程解决 |
| 末尾命令退出码检查 | ✅ check_ok（4.1 节） | ✅ check_ok | ✅ check_ok | ❌ wait_with_output 不调 check_ok（4.8 节） |
| pipefail 检查（全部命令） | 默认关 / 开时 check_exit_status_future | 同左 | 同左 | ❌ original_exit.clear() + 结果为 Value → exit 向量只有 None |
| LAST_EXIT_CODE 最终值 | 末尾命令退出码（A/B/C/D） | 同左 | 同左 | 恒为 `0`（4.4 节 complete 行，路径 C） |
| 非零退出如何暴露 | 直接报错 + LAST_EXIT_CODE | 直接报错 + LAST_EXIT_CODE | 直接报错 + LAST_EXIT_CODE | 仅 `exit_code` 字段（4.8 节三重保护） |

### 7.6 stderr 管道边界问题

1. **`e>|` 的 swap 机制是核心设计**
   - stderr 通过 swap 放入 `ChildProcess.stdout` 位置
   - 这使得 `into_stdio()` 零拷贝可以正常工作
   - `debug_assert!(stderr.is_none())` 确保 swap 后不会混淆

2. **错误值传递的边界**
   - `stderr_pipe_separate` 仅在 `OutDest::PipeSeparate` 时触发
   - `e>|` 使用 `OutDest::Pipe`，错误不会被包装为 `Value::Error`
   - `complete` 上游使用 `PipeSeparate`，错误会被包装为 `Value::Error`

3. **`Pipe` 与 `PipeSeparate` 的语义分工**
   - `Pipe`：用于 `o+e>|`（合并）和 `e>|`（swap 到 stdout 位置）
   - `PipeSeparate`：用于 `complete`（保持 stdout 和 stderr 分离）
   - `e>|` 不使用 `PipeSeparate`，因为管道只传递一个流（stderr），不需要分离

---

## 第六部分：边界问题与设计权衡

### 6.1 管道边界的模糊点

1. **`Pipe` vs `PipeSeparate` 的语义边界**
   - `Pipe`：用于管道传递（`o+e>|` 合并流，`e>|` swap stderr 到 stdout 位置）
   - `PipeSeparate`：用于 `complete`（保持 stdout 和 stderr 分离）
   - `e>|` 不使用 `PipeSeparate`，因为管道只传递一个流

2. **零拷贝的条件限制**
   - 仅 `ChildPipe::Pipe` 支持零拷贝，`ChildPipe::Tee` 不支持
   - `tee` 命令会破坏零拷贝链
   - 内部命令产生的 `ByteStreamSource::Read` 类型无法零拷贝
   - `e>|` 通过 swap 机制支持零拷贝

3. **合并流的信息丢失**
   - stdout 和 stderr 合并后，下游无法区分原始来源
   - 这是 `Pipe` 模式的设计取舍

### 6.2 退出码的边界

> 本节内容是第四部分结论的摘要，所有结论均引用 4.2 节的 A/B/C/D 四个设置点和 4.1 节的 `check_ok` 调用链。

1. **`LAST_EXIT_CODE` 与 pipefail 的关系**
   - Drain 指令：先被路径 B 设为 `0`（末尾命令成功），若 pipefail 发现中间命令失败，再被路径 D 覆盖为失败命令的退出码
   - 非 Drain 路径（赋值、drain_if_end、complete）：只有路径 C（成功→0）或路径 D（失败→错误码）
   - ✅ 最终反映的是**导致失败的命令**的退出码，不会被错误覆盖

2. **`LAST_EXIT_CODE` 的设置路径差异**
   - `drain()` 函数：设置（路径 A/B）
   - `drain_if_end()`、`collect()`、`collect_reg()`：**不**设置，依赖 `eval_source()` 的路径 C/D
   - `eval_source()`：最终设置（路径 C/D）

3. **赋值语句中末尾命令的检查**
   - `collect_reg` 的注释 "doesn't check exit status" 仅指 pipefail 的 exit 向量
   - `into_value()` → `into_bytes()` → `check_ok()` 仍会检查末尾命令
   - `let x = ^false` 会失败，`let x = (^false | ^echo ok)` 会成功

4. **错误提取逻辑：external_exit_code()**
   - `NonZeroExitCode` → 提取 `exit_code` 字段
   - `TerminatedBySignal` / `CoreDumped` → 提取 `-signal`
   - 其他错误 → `external_exit_code()` 返回 None，`exit_code()` 回退到 `Some(1)`

5. **ignore_error 的传播**
   - `ignore_error` 是每个命令独立的 `Arc<Mutex<bool>>`
   - 在 `check_ok()` 和 `check_exit_status_future_ok()` 两处生效
   - 设置方：`ignore` 命令、`complete.child.ignore_error(true)`

6. **三种收集指令的差异（见 4.7.2 节完整表）**
   - `collect_reg`：清空 exit → 不检查 pipefail，但检查末尾命令
   - `Collect`：`ignore_error=true` → 不检查 pipefail，但检查末尾命令
   - `TryCollect`：`ignore_error=false` → 检查 pipefail + 末尾命令

7. **信号退出码**
   - Unix 信号终止 → 退出码为 `-signal`（如 SIGINT=2 → -2）
   - `SIGPIPE` 特殊处理：不视为错误（check_ok 中返回 Ok）

### 6.3 stderr 管道的边界

1. **`e>|` 的 swap 机制**
   - stderr 通过 swap 放入 `ChildProcess.stdout` 位置
   - 零拷贝可以正常工作，不存在之前的描述中说的"无法零拷贝"问题
   - `debug_assert!(stderr.is_none())` 确保 swap 后不会混淆

2. **错误值传递的边界**
   - `stderr_pipe_separate` 仅在 `OutDest::PipeSeparate` 时触发
   - `e>|` 使用 `OutDest::Pipe`，错误不会被包装为 `Value::Error`
   - `complete` 上游使用 `PipeSeparate`，错误会被包装为 `Value::Error`

3. **`wait_with_output` 的死锁避免**
   - 并发收集 stdout 和 stderr，避免缓冲区满导致的死锁
   - 使用独立线程收集 stderr
   - 这是 `PipeSeparate` 模式（complete）的必要开销

### 6.4 设计权衡总结

| 设计决策 | 优点 | 缺点 |
|---------|------|------|
| ByteStream 三级抽象 | 统一接口，支持多种源 | 零拷贝有条件限制 |
| PipelineExecutionData 累积退出状态 | 实现 pipefail 不需要修改命令代码 | 内存开销，需要 careful 维护 |
| collect_reg 清空 exit | 赋值语句的 pipefail 语义符合预期 | 注释误导（"doesn't check exit status"实际只指 pipefail） |
| complete 清空 exit | 语义正确（已转为数据） | 硬编码命令名，不够优雅 |
| 反向遍历检查 pipefail | 第一个错误就是最后一个命令的，符合直觉 | — |
| ignore_error Arc<Mutex<bool>> | 运行时可修改，complete 可以标记忽略 | 共享状态增加复杂度 |
| wait_with_output 并发收集 | 避免 stdout/stderr 死锁 | 额外线程开销 |
| stderr_pipe_separate 错误包装 | 错误可以通过管道传递 | 下游需要处理 Value::Error |
| e>\| swap 机制 | 零拷贝支持，语义清晰 | 内部 stdout/stderr 语义反转 |
| drain() 直接设 LAST_EXIT_CODE | 管道中间即可反映末尾命令退出码 | Drain 指令下 pipefail 失败会被 eval_source 覆盖，存在短暂不一致 |
| drain_if_end/collect/collect_reg 不设 LAST_EXIT_CODE | 逻辑集中在 eval_source，避免重复设置 | 中间阶段 LAST_EXIT_CODE 不变 |
| external_exit_code 从错误提取 | 统一的退出码提取逻辑 | 部分错误返回 None，回退到 1 |

---

## 附录：关键代码索引

| 功能 | 文件 | 行号 |
|------|------|------|
| 外部命令执行入口 | [run_external.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/run_external.rs) | L52-L350 |
| stdin 配置逻辑 | [run_external.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/run_external.rs) | L243-L274 |
| stdout/stderr 配置与 swap | [run_external.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/run_external.rs) | L205-L263 |
| 零拷贝转换 into_stdio | [byte_stream.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs) | L559-L579 |
| ByteStreamSource::reader (读取 stdout) | [byte_stream.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs) | L39-L49 |
| stdin 写入线程 | [run_external.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/run_external.rs) | L307-L325 |
| write_pipeline_data | [run_external.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/run_external.rs) | L490-L521 |
| ChildProcess 构造与 swap | [child.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs) | L281-L343 |
| 退出状态检查 check_ok | [child.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs) | L48-L89 |
| pipefail 检查 | [child.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs) | L22-L29 |
| wait_with_output | [child.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs) | L464-L505 |
| into_bytes (检查末尾退出码) | [child.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs) | L376-L406 |
| ignore_error 设置 | [child.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/process/child.rs) | L364-L370 |
| PipelineExecutionData | [pipeline_data.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs) | L1116-L1163 |
| clone_exit_status_future | [pipeline_data.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs) | L871-L883 |
| drain_to_out_dests | [pipeline_data.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/pipeline_data.rs) | L302-L327 |
| ByteStream::drain | [byte_stream.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs) | L683-L694 |
| ByteStream::into_value | [byte_stream.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs) | L662-L681 |
| ByteStream::into_child | [byte_stream.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/byte_stream.rs) | L586-L592 |
| LAST_EXIT_CODE 设置 | [stack.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/engine/stack.rs) | L309-L319 |
| set_last_error | [stack.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/engine/stack.rs) | L313-L319 |
| Call 指令处理 | [eval_ir.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs) | L675-L713 |
| collect_reg (含注释) | [eval_ir.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs) | L207-L221 |
| Collect 指令 | [eval_ir.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs) | L421-L429 |
| TryCollect 指令 | [eval_ir.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs) | L430-L438 |
| collect 函数 | [eval_ir.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs) | L1698-L1712 |
| drain 函数 | [eval_ir.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs) | L1715-L1771 |
| drain_if_end | [eval_ir.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs) | L1774-L1793 |
| stderr_pipe_separate 处理 | [eval_ir.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/eval_ir.rs) | L1230-L1235 |
| eval_source LAST_EXIT_CODE 设置 | [util.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-cli/src/util.rs) | L249-L261 |
| evaluate_source pipefail 检查 | [util.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-cli/src/util.rs) | L337-L342 |
| external_exit_code | [shell_error/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/errors/shell_error/mod.rs) | L1481-L1497 |
| complete 命令 | [complete.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/complete.rs) | L27-L88 |
| complete.pipe_redirection | [complete.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-command/src/system/complete.rs) | L97-L99 |
| e>\| 编译时重定向 | [compile/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-engine/src/compile/mod.rs) | L119-L131 |
| StackIoGuard 重定向管理 | [stack_out_dest.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/engine/stack_out_dest.rs) | L103-L185 |
| 输出目标枚举 | [out_dest.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/77-nushell/crates/nu-protocol/src/pipeline/out_dest.rs) | L5-L40 |
