# Nushell 插件协议：通信帧、序列化与异常隔离

本文档对照代码，厘清 Nushell 插件系统中 **协议层**（消息帧定义）、**序列化层**（字节编码）、**I/O层**（通信通道）与 **进程生命周期**（子进程管理/GC/异常隔离）的清晰边界。

> **路径约定**：本文所有代码引用均为仓库根目录下的相对路径，格式为 `crates/<crate>/src/...`。

---

## 0. 代码仓库分层概览

插件系统分为 4 个核心 crate，职责严格分层：

| Crate | 职责 | 关键文件 |
|---|---|---|
| `nu-plugin-protocol` | **纯类型定义**：消息帧枚举 + Serde 实现，**零 I/O** | `crates/nu-plugin-protocol/src/lib.rs` |
| `nu-plugin-core` | **共享逻辑**：序列化器、通信模式、流多路复用、接口抽象 | `crates/nu-plugin-core/src/serializers/mod.rs`、`crates/nu-plugin-core/src/communication_mode/mod.rs`、`crates/nu-plugin-core/src/interface/mod.rs` |
| `nu-plugin-engine` | **引擎侧**：子进程 spawn、PersistentPlugin 持久化、GC 垃圾回收、异常隔离屏障 | `crates/nu-plugin-engine/src/init.rs`、`crates/nu-plugin-engine/src/persistent.rs`、`crates/nu-plugin-engine/src/gc.rs`、`crates/nu-plugin-engine/src/interface/mod.rs` |
| `nu-plugin` | **插件侧 SDK**：供第三方插件使用的服务端循环、EngineInterface | `crates/nu-plugin/src/plugin/mod.rs` |

```
┌──────────────────────────────────────────────────────────────┐
│                    Nushell Engine 进程                        │
│  ┌───────────────────────────────────────────────────────┐   │
│  │ nu-plugin-engine: PersistentPlugin / GC / PluginProcess│   │
│  └───────────────┬───────────────────────────────────────┘   │
│                  │ InterfaceManager (读线程 + 状态机)          │
│  ┌───────────────▼───────────────────────────────────────┐   │
│  │ nu-plugin-core: Encoder / StreamManager / CommMode    │   │◄─── 协议边界
│  └───────────────┬───────────────────────────────────────┘   │
│                  │ (pipe / local socket)                       │
└──────────────────┼───────────────────────────────────────────┘
                   │ stdin/stdout 或本地 socket
┌──────────────────▼───────────────────────────────────────────┐
│                    插件子进程 (独立地址空间)                    │
│  ┌───────────────────────────────────────────────────────┐   │
│  │ nu-plugin: EngineInterfaceManager / serve_plugin()    │   │
│  └───────────────┬───────────────────────────────────────┘   │
│                  │                                             │
│  ┌───────────────▼───────────────────────────────────────┐   │
│  │ nu-plugin-core: 同左 (与引擎侧对称)                     │   │
│  └───────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

---

## 1. 通信帧 (Protocol Frames)

通信帧是**纯数据结构**，定义在 `nu-plugin-protocol` 中，只 derive `Serialize/Deserialize/Debug/Clone`，**不包含任何 I/O 或业务逻辑**。

### 1.1 三个维度的 ID 多路复用

协议用三种独立的递增 ID 将单条字节流复用为多层语义通道：

| ID 类型 | Rust 别名 | 用途 | 生命周期 |
|---|---|---|---|
| `StreamId` | `usize` | 标识一条 List/Byte 流（Data/End/Drop/Ack） | 随流开始→流结束，可能跨多个 PluginCall |
| `PluginCallId` | `usize` | 标识一次引擎→插件调用（Metadata/Signature/Run 等） | 发出 Call → 收到 CallResponse |
| `EngineCallId` | `usize` | 标识一次插件→引擎回调（GetEnv/EvalClosure 等） | 发出 EngineCall → 收到 EngineCallResponse |

见 `crates/nu-plugin-protocol/src/lib.rs` 第 42-49 行。

### 1.2 消息帧枚举

#### 引擎 → 插件：`PluginInput`

```rust
pub enum PluginInput {
    Hello(ProtocolInfo),           // 第一条：协议版本协商
    Call(PluginCallId, PluginCall<PipelineDataHeader>),  // 执行插件调用
    Goodbye,                       // 优雅关闭：不再接受新调用
    EngineCallResponse(EngineCallId, EngineCallResponse<PipelineDataHeader>),
    Data(StreamId, StreamData),    // 流数据帧
    End(StreamId),                 // 流结束帧
    Drop(StreamId),                // 流丢弃帧（读端关闭）
    Ack(StreamId),                 // 流确认帧（流量控制）
    Signal(SignalAction),          // 信号中继 (Ctrl+C)
}
```
见 `crates/nu-plugin-protocol/src/lib.rs` 第 288-311 行。

#### 插件 → 引擎：`PluginOutput`

```rust
pub enum PluginOutput {
    Hello(ProtocolInfo),           // 第一条：协议版本协商
    Option(PluginOption),          // 设置 GC 行为等
    CallResponse(PluginCallId, PluginCallResponse<PipelineDataHeader>),
    EngineCall {                   // 回调引擎
        context: PluginCallId,     // 绑定到哪个 PluginCall 的上下文
        id: EngineCallId,
        call: EngineCall<PipelineDataHeader>,
    },
    Data(StreamId, StreamData),    // 与 PluginInput 对称的流帧
    End(StreamId),
    Drop(StreamId),
    Ack(StreamId),
}
```
见 `crates/nu-plugin-protocol/src/lib.rs` 第 509-536 行。

#### 流帧的统一抽象：`StreamMessage`

`Data/End/Drop/Ack` 被抽离为 `StreamMessage`，通过 `TryFrom`/`From` 在 `PluginInput`/`PluginOutput` 之间转换，使流管理代码**与具体方向解耦**：

```rust
pub enum StreamMessage {
    Data(StreamId, StreamData),
    End(StreamId),
    Drop(StreamId),
    Ack(StreamId),
}
```
见 `crates/nu-plugin-protocol/src/lib.rs` 第 397-410 行。

### 1.3 PipelineDataHeader：管道数据的"首帧"

`PipelineData`（Value/ListStream/ByteStream）不能直接序列化，因为流需要后续帧。因此引入 **"Header + 后续流帧"** 模式：

```rust
pub enum PipelineDataHeader {
    Empty,
    Value(Value, Option<PipelineMetadata>),     // 单值：无后续帧
    ListStream(ListStreamInfo { id, span, .. }),  // 流：后续 StreamMessage::Data(id, List(v))
    ByteStream(ByteStreamInfo { id, type_, .. }), // 流：后续 StreamMessage::Data(id, Raw(bytes))
}
```
见 `crates/nu-plugin-protocol/src/lib.rs` 第 121-135 行。

### 1.4 协议版本协商：Hello 帧

启动后双方**必须**先交换 `Hello(ProtocolInfo)`：

```rust
pub struct ProtocolInfo {
    pub protocol: Protocol,   // 固定为 "nu-plugin"
    pub version: String,      // semver，如 "0.98.0"
    pub features: Vec<Feature>, // 可选特性：LocalSocket 等
}
```

兼容性判定：**低版本 semver caret 兼容高版本**（如 1.1.x 与 1.2.x 兼容，但 1.x 与 2.x 不兼容）。nightly 版本的预发布后缀在比较时被忽略。

见 `crates/nu-plugin-protocol/src/protocol_info.rs`。

---

## 2. 序列化层 (Serialization)

定义在 `crates/nu-plugin-core/src/serializers/`。序列化层的抽象是 **Encoder trait**，它**只关心如何将消息帧 ↔ 字节流**，不关心通道是什么（pipe/socket）。

### 2.1 Encoder 抽象

```rust
pub trait Encoder<T>: Clone + Send + Sync {
    fn encode(&self, data: &T, writer: &mut impl Write) -> Result<(), ShellError>;
    fn decode(&self, reader: &mut impl BufRead) -> Result<Option<T>, ShellError>;
}

pub trait PluginEncoder: Encoder<PluginInput> + Encoder<PluginOutput> {
    fn name(&self) -> &str;  // "json" 或 "msgpack"
}
```
见 `crates/nu-plugin-core/src/serializers/mod.rs` 第 12-32 行。

### 2.2 JSON 序列化器 (JsonSerializer)

- **帧边界**：每行一个 JSON 对象（NDJSON 风格），encode 时末尾追加 `\n`
- **decode**：使用 `serde_json::Deserializer::from_reader` 的自描述流，不需要显式按行分割（`\n` 在 JSON 顶层是空白字符，自动跳过）
- **EOF 判定**：`serde_json::Error::is_eof()` → 返回 `Ok(None)`

```rust
// encode: serde_json::to_writer(...) + write_all(b"\n")
// decode: PluginInput::deserialize(&mut de) → Ok(Some) 或 is_eof() → Ok(None)
```
见 `crates/nu-plugin-core/src/serializers/json.rs`。

### 2.3 MsgPack 序列化器 (MsgPackSerializer)

- **帧边界**：MsgPack 自描述长度编码，**无分隔符**
- 使用 `rmp_serde::encode::write_named`（命名字段，便于跨语言兼容）
- **EOF 判定**：`InvalidMarkerRead(UnexpectedEof)` 或 `InvalidDataRead(UnexpectedEof)` → `Ok(None)`

见 `crates/nu-plugin-core/src/serializers/msgpack.rs`。

### 2.4 编码协商握手（Encoding Negotiation）

插件启动后**立即**在第一条业务消息前发送编码标识，格式是：
```
[1字节长度N][N字节编码名]
```
如：`\x04json` 或 `\x07msgpack`。

引擎侧读取逻辑：`get_plugin_encoding()` 在 `crates/nu-plugin-engine/src/init.rs` 第 195-218 行：

```rust
// 读 1 字节长度 → 读 N 字节名称 → EncodingType::try_from_bytes(&buf)
// 支持值: b"json" → JsonSerializer, b"msgpack" → MsgPackSerializer
```

插件侧发送逻辑：`tell_nushell_encoding()` 在 `crates/nu-plugin/src/plugin/mod.rs` 第 378-392 行。

> **边界关键**：这 1+N 字节**不经过 Encoder**，是序列化层的外层引导字节。它发生在 Hello 帧之前。

### 2.5 错误分层

Encoder 的错误被严格分为三类，体现了协议层与 I/O 层的边界：

| ShellError 变体 | 触发场景 | 语义 |
|---|---|---|
| `ShellError::Io` | `write_all` / `read_exact` 系统调用失败 | **I/O层故障**：通道断了，不是数据本身问题 |
| `PluginFailedToEncode` | serde 序列化失败（Value 包含不可序列化类型等） | **数据问题**：本侧 bug |
| `PluginFailedToDecode` | serde 反序列化失败（格式错误/不匹配） | **数据问题**：对侧 bug / 版本不兼容 |

这三类错误在引擎侧 `consume_all()` 中处理方式完全不同（见 §4.2）。

---

## 3. I/O 层：通信模式 (Communication Mode)

定义在 `crates/nu-plugin-core/src/communication_mode/mod.rs`。这一层**只关心字节通道的建立**，不理解消息帧内容。

### 3.1 Stdio 模式（默认）

```
引擎 stdin 管道 ←──── 插件 stdout
引擎 stdout管道 ────→ 插件 stdin
```

- 子进程 `Stdio::piped()` 接管 stdin/stdout
- **缺点**：插件不能直接用 stdout 与终端交互（必须通过 EngineCall::EnterForeground）

### 3.2 LocalSocket 模式（可选）

- 引擎先创建本地 socket 监听器（Unix domain socket / Windows named pipe）
- 插件通过 `--local-socket <name>` 参数连接两次：一次读、一次写
- **优点**：stdio 还给插件，可直接与终端 TTY 交互
- 协商：Hello 帧 `features: [LocalSocket]` 后，引擎自动重启插件切换到此模式
- 连接超时：10 秒（`const TIMEOUT: Duration = Duration::from_secs(10)`）

见 `crates/nu-plugin-core/src/communication_mode/mod.rs` 第 180、223 行。

### 3.3 连接建立流程（3步握手）

```
引擎侧                              插件子进程
  │                                   │
  │ 1. create_command + mode.serve()  │
  │    (创建监听器 / 准备管道)         │
  │                                   │
  │ 2. spawn child                    │ ── 启动
  │                                   │
  │ 3. PreparedServerCommunication    │
  │    .connect(&mut child)           │
  │    → ServerCommunicationIo        │ ←─ 发送 [1字节长度][编码名]
  │                                   │
  │    get_plugin_encoding()          │ （在 connect 返回后，进入 init）
  │    hello() → Hello(ProtocolInfo) ─│→
  │  ← Hello(ProtocolInfo) ───────────│
  │                                   │
  │    协议版本检查通过 → 就绪          │
```

---

## 4. 子进程生命周期与异常隔离

这是最容易与协议帧混淆的部分。**协议帧定义了"说什么"，生命周期定义了"什么时候可以说 / 说错了怎么办"。**

### 4.1 子进程启动：PersistentPlugin 的懒加载

`PersistentPlugin` 是**引擎侧**对插件生命周期的唯一持有入口，位于 `crates/nu-plugin-engine/src/persistent.rs`。

```rust
pub struct PersistentPlugin {
    identity: PluginIdentity,
    mutable: Mutex<MutableState>,  // 单锁保护全部可变状态，防止死锁
}

struct MutableState {
    running: Option<RunningPlugin>,      // None = 未启动 / 已停止
    preferred_mode: Option<PreferredCommunicationMode>, // 学习到的最佳模式
    gc_config: PluginGcConfig,
    signal_guard: Option<HandlerGuard>,  // 信号处理器 RAII
    // ...
}

struct RunningPlugin {
    interface: PluginInterface,  // 可 Clone 的句柄
    gc: PluginGc,                // GC 句柄
}
```

**懒启动流程**（`PersistentPlugin::get()` → `spawn()`）：
1. 锁 `mutable`，若 `running.is_some()` → 直接 clone interface 返回
2. 否则调用 `create_command()`（见 `crates/nu-plugin-engine/src/init.rs` 第 35-101 行）：
   - 根据扩展名自动选择解释器（`.py`→python，`.nu`→`nu --stdin`，`.jar`→`java -jar` 等）
   - **关键隔离1：新进程组**！Unix `process_group(0)`，Windows `CREATE_NEW_PROCESS_GROUP`
   - **关键隔离2：CWD 隔离**：工作目录设为插件可执行文件所在目录，避免污染引擎 cwd
3. `mode.serve()` → 准备通信通道
4. `plugin_cmd.spawn()` → 产生子进程
5. 启动独立 GC 线程（见 §4.3）
6. `make_plugin_interface()` → `get_plugin_encoding()` → hello 握手
7. **模式升级尝试**：若 Hello 携带 `LocalSocket` feature → 停止当前插件，重启为 LocalSocket 模式
8. 将 `RunningPlugin` 存入 `mutable.running`

### 4.2 读线程与错误屏障：consume_all()

**这是异常隔离的核心机制**。每个插件有一个专用后台读线程（"plugin interface reader"），循环调用 `consume_all()`：

```rust
pub fn consume_all(&mut self, mut reader: impl PluginRead<PluginOutput>) -> Result<(), ShellError> {
    while let Some(msg) = reader.read().transpose() {
        if self.is_finished() { break; }
        if let Err(err) = msg.and_then(|msg| self.consume(msg)) {
            // ── 错误屏障点 ─────────────────────────────────
            // 1. 记录致命错误，所有后续调用直接返回
            let _ = self.state.error.set(err.clone());
            // 2. 向所有正在读的流广播错误
            let _ = self.stream_manager.broadcast_read_error(err.clone());
            // 3. 向所有等待 PluginCall 响应的线程发送 Error
            for subscription in std::mem::take(&mut self.plugin_call_states).into_values() {
                let _ = subscription.sender.as_ref()
                    .map(|s| s.send(ReceivedPluginCallMessage::Error(err.clone())));
            }
            result = Err(err);
            break;  // 退出读循环
        }
    }
    // 通知 GC 插件已退出
    if let Some(ref gc) = self.gc { gc.exited(); }
    result
}
```
见 `crates/nu-plugin-engine/src/interface/mod.rs` 第 417-452 行。

**引擎侧三层错误隔离**：
| 层次 | 机制 | 效果 |
|---|---|---|
| **进程级** | 新进程组 + 独立地址空间 | 插件 panic / abort / 内存泄漏 不影响引擎 |
| **通道级** | 读线程捕获 I/O / 解码错误 → 广播 → break | 所有等待方收到错误，**无死锁** |
| **调用级** | `state.error: OnceLock<ShellError>` | 后续新调用在 `plugin_call()` 入口**直接短路返回**，不会写入半关闭通道 |

**插件侧的异常隔离**（对称存在）：
插件侧 `serve_plugin_io()` 也有对应的 "engine interface reader" 读线程。此外，每个 Run 调用在独立线程中执行，并用 `std::panic::catch_unwind(AssertUnwindSafe(|| { ... }))` 捕获 panic，若发生 panic 则调用 `std::process::exit(1)` 退出进程。见 `crates/nu-plugin/src/plugin/mod.rs` 第 515-539 行。

### 4.3 插件垃圾回收 (PluginGc)

即使协议层面有 Goodbye 帧，用户也可能忘记显式卸载，或插件需要支持"空闲自动关闭"。GC 是一个独立线程，通过 mpsc 通道与主逻辑通信。

```
PluginGc (主线程持有 Clone)
    │ sender: mpsc::Sender<PluginGcMsg>
    ▼
GC 专用线程：run(receiver) 循环
    ├─ recv_timeout(next_timeout)
    │   ├─ 收到消息 → handle_message()
    │   └─ 超时 → 停止插件
```

GC 停止条件（需 **同时满足**）：
1. `config.enabled == true`
2. `disabled == false`（可被插件通过 `PluginOption::GcDisabled(true)` 覆盖）
3. `locks == 0`（无活跃调用 / 无活跃流）
4. 距 `last_update` 超过 `config.stop_after`

#### 默认值分层（容易混淆，逐一厘清）

代码中存在三种"默认值"，含义完全不同，不应混淆：

| 来源 | `stop_after` | `enabled` | 何时生效 | 代码位置 |
|---|---|---|---|---|
| `PluginGcConfig::default()` | **10 秒** | `true` | Rust `Default` trait 的硬编码默认，**仅在未加载任何用户配置时使用**（如 `Config::default()` 初始化） | `crates/nu-protocol/src/config/plugin_gc.rs` 第 54-61 行 |
| 用户配置 `$env.config.plugin_gc.default` | 由用户设定 | 由用户设定 | **运行时实际生效的值**，会覆盖 Rust Default | 经 `UpdateFromValue::update()` 合入 |
| 测试样例 `test_pair()` | 30 秒 | `true` | **仅用于单元测试**，验证 `update/reconstruct` 逻辑正确性，不是任何运行时默认值 | `crates/nu-protocol/src/config/plugin_gc.rs` 第 111-141 行 |

> **关键纠正**：30 秒只是测试用例中的断言值，**不是** `PluginGcConfigs` 的 Rust 默认值。`PluginGcConfigs::default()` 的 `default` 字段直接委托给 `PluginGcConfig::default()`，即 10 秒。如果用户在配置文件中显式写了 `stop_after: 30sec`，那运行时就是 30 秒，但这是用户配置覆盖的结果，不是代码默认值。

#### 配置覆盖链路

```
Config::default()                           // plugin_gc: PluginGcConfigs::default()
  │                                           //   → default: PluginGcConfig { enabled: true, stop_after: 10s }
  ▼
$env.config.plugin_gc.update(Value)         // 用户配置覆盖
  │
  ▼
add_plugin_to_working_set()                 // crates/nu-plugin-engine/src/init.rs 第 300-316 行
  │  gc_config = config.plugin_gc.get(name) // 按插件名查找，未指定则用 default
  │  PersistentPlugin::new(identity, gc_config)
  │  plugin.set_gc_config(&gc_config)       // 若已存在，也会热更新
  ▼
PersistentPlugin::spawn()
  │  PluginGc::new(mutable.gc_config, ...)  // GC 线程启动，持有 config 副本
  ▼
PluginGcState::run()                        // GC 线程主循环
     config.stop_after → next_timeout()
```

热更新机制：`set_gc_config()` 不仅更新 `MutableState.gc_config`，还会向运行中的 GC 线程发送 `PluginGcMsg::SetConfig`，使其**无需重启插件即可生效**。

见 `crates/nu-plugin-engine/src/persistent.rs` 第 294-306 行。

**锁计数规则**（见 `PluginGc::increment_locks / decrement_locks`）：
- **+1**：每次发起 PluginCall（`write_plugin_call` 结尾）
- **-1**：每次收到 PluginCallResponse（`consume(CallResponse)` 结尾）
- **+1**：每次开始读来自插件的流（`recv_stream_started`）
- **-1**：每次结束读来自插件的流（`recv_stream_ended` / `StreamMessage::End`）

见 `crates/nu-plugin-engine/src/gc.rs`。

### 4.4 优雅关闭链

关闭路径有三条，**不使用 SIGKILL**：

| 触发方式 | 实现 | 说明 |
|---|---|---|
| **自动**：最后一个 `PluginInterface` drop | `Drop::drop` 检查 `Arc::strong_count < 3` → 发 `Goodbye` | 最常见 |
| **手动**：用户 `plugin stop` | `RegisteredPlugin::stop()` → `PersistentPlugin::stop_internal()` → 置 `running = None` → PluginInterface drop → Goodbye | |
| **超时**：GC 线程 | `PersistentPlugin::stop()`（同上） | |

**Goodbye 语义**：
- `Goodbye` 只是"**不再接受新的 PluginCall**"（引擎发给插件，属于 `PluginInput` 枚举）
- 正在执行的调用 / 正在传输的流 **继续完成**
- 插件完成所有工作后自行退出进程（因此引擎只需等待 EOF）

### 4.5 崩溃恢复

如果插件进程异常退出（panic / kill）：
1. 读线程 `decode()` 遇到 EOF（`Ok(None)`）或 I/O Error
2. `consume_all()` 进入错误屏障路径（§4.2）
3. `gc.exited()` 通知 GC 清理
4. 下次 `PersistentPlugin::get()` 时，`running.is_none()` → **自动重新 spawn**

对上层用户完全透明。

---

## 5. 流复用与流量控制 (StreamManager)

最后澄清一个常见混淆：**流多路复用是 I/O 层之上、协议帧之下的基础设施**，不属于"通信帧"也不属于"子进程生命周期"。

### 5.1 核心数据结构

```
StreamManager (InterfaceManager 持有)
  ├─ state: Mutex<StreamManagerState>
  │   ├─ reading_streams: BTreeMap<StreamId, mpsc::Sender<...>>
  │   └─ writing_streams: BTreeMap<StreamId, Weak<StreamWriterSignal>>
```

每个 `StreamId` 都被注册为一条独立的 mpsc 通道。`StreamMessage`（Data/End/Drop/Ack）被 `handle_message()` 分发到对应通道。

见 `crates/nu-plugin-core/src/interface/stream/mod.rs`。

### 5.2 流量控制（背压）

`StreamWriterSignal` 用条件变量实现：

| 常量 | 值 | 含义 |
|---|---|---|
| `LIST_STREAM_HIGH_PRESSURE` | 100 | List 流未确认帧数阈值 |
| `RAW_STREAM_HIGH_PRESSURE` | 50 | Byte 流未确认帧数阈值 |

流程：
- 写端每 `write()` 一帧 → `unacknowledged += 1` → 超过阈值 → `wait_for_drain()`（Condvar::wait）
- 读端每 `recv()` 一帧 → 发 `Ack(id)` → 写端收到 → `notify_acknowledged()` → `unacknowledged -= 1` → `Condvar::notify_one`
- 读端 drop → 发 `Drop(id)` → 写端 `set_dropped()` → 解除阻塞，`write_all` 提前终止

### 5.3 流生命周期与 PluginCall 的绑定

一条流**可能跨越 PluginCall 的结束点继续存活**（例如 Plugin 先返回 Response，然后在后台慢慢吐流）。引擎侧用 `remaining_streams_to_read` 计数器确保：
- PluginCallResponse 到达后，若还有流未读完，**不释放** `PluginCallState`（保留上下文以处理 EngineCall）
- 所有流 End 后，计数器归零，才清理状态

见 `recv_stream_started` / `recv_stream_ended` 在 `crates/nu-plugin-engine/src/interface/mod.rs` 第 206-235 行。

---

## 6. 自定义值释放通知：包装析构、引用计数与 Dropped 消息

这是最容易误解的路径：**Dropped 通知与 PluginCall 状态释放无关**，它由 `PluginCustomValueWithSource` 的 Drop 析构触发，结合 `SharedCow` 的引用计数和 `notify_on_drop` 标记共同决定。

### 6.1 四层包装结构

```
PluginCustomValueWithSource  (引擎侧，实现 Drop trait)
  ├─ inner: PluginCustomValue  (可跨边界序列化，不含 source)
  │   └─ SharedCow<SharedContent>  (Arc 引用计数 + Clone-on-write)
  │       ├─ name: String
  │       ├─ data: Vec<u8>  (bincode 序列化后的实际自定义值)
  │       └─ notify_on_drop: bool  (来自 CustomValue::notify_plugin_on_drop())
  └─ source: Arc<PluginSource>  (指向哪个插件，不跨边界)
```

关键代码位置：
- `PluginCustomValue` 定义：`crates/nu-plugin-protocol/src/plugin_custom_value/mod.rs` 第 11-41 行
- `PluginCustomValueWithSource` 定义：`crates/nu-plugin-engine/src/plugin_custom_value_with_source/mod.rs` 第 16-25 行
- `SharedCow` 实现：`crates/nu-utils/src/shared_cow.rs` 第 4-44 行

### 6.2 标记的来源：notify_plugin_on_drop()

`notify_on_drop` 标记在**插件侧**序列化时确定，来自 `CustomValue` trait 的方法：

```rust
// 插件侧代码
pub trait CustomValue {
    /// True if the plugin should be notified when this value is dropped by the engine.
    fn notify_plugin_on_drop(&self) -> bool { false }  // 默认 false
}
```

序列化时被捕获：
```rust
// PluginCustomValue::serialize_from_custom_value()
pub fn serialize_from_custom_value(custom_value: &dyn CustomValue, span: Span) -> Result<PluginCustomValue, ShellError> {
    let notify_on_drop = custom_value.notify_plugin_on_drop();
    // ... rmp_serde::to_vec(custom_value) ...
    Ok(PluginCustomValue::new(name, data, notify_on_drop))
}
```

序列化优化：`notify_on_drop` 字段加了 `#[serde(default, skip_serializing_if = "is_false")]`，**为 false 时不序列化**，节省带宽。

### 6.3 引用计数：SharedCow<Arc<T>>

`SharedCow` 内部就是 `Arc<T>`，引用计数由 `Arc::strong_count()` 提供：

```rust
pub fn ref_count(value: &SharedCow<T>) -> usize {
    Arc::strong_count(&value.0)
}
```

每当 `PluginCustomValue` 被 `clone()`，引用计数 +1。每当一个克隆被 drop，引用计数 -1。

### 6.4 Drop 析构的触发条件（三者同时满足）

**唯一触发点**是 `PluginCustomValueWithSource::drop()`，位于 `crates/nu-plugin-engine/src/plugin_custom_value_with_source/mod.rs` 第 277-292 行：

```rust
impl Drop for PluginCustomValueWithSource {
    fn drop(&mut self) {
        // 条件 1: notify_on_drop == true （插件在序列化时设置）
        // 条件 2: ref_count() == 1 （这是最后一个持有 SharedContent 的对象）
        if self.notify_on_drop() && self.inner.ref_count() == 1 {
            // 条件 3: 能成功获取到 PluginInterface（插件进程还活着）
            self.get_plugin(None, "drop")
                .and_then(|plugin| plugin.custom_value_dropped(self.inner.clone()))
                .unwrap_or_else(|err| {
                    log::warn!("Failed to notify drop of custom value ({name}): {err}")
                });
        }
    }
}
```

> **关键纠正**：这与 `PluginCallState` 的释放**完全无关**。即使 PluginCall 早已完成，只要自定义值还在引擎中被传递、被克隆、被使用，ref_count > 1，就不会触发 Dropped 通知。

### 6.5 Dropped 消息的完整链路

```
插件侧 (创建自定义值)                          引擎侧 (持有并使用)                         插件侧 (收到通知)
  │                                              │                                             │
  │ 1. CustomValue::notify_plugin_on_drop()=true  │                                             │
  │    serialize_from_custom_value()              │                                             │
  │    → PluginCustomValue { notify_on_drop: true }│                                             │
  │                                              │                                             │
  │ ── PluginOutput::Data/CallResponse ─────────► │ 2. 接收后 add_source()                     │
  │                                              │    → PluginCustomValueWithSource            │
  │                                              │      ref_count = 1                          │
  │                                              │                                             │
  │                                              │ 3. clone() → 传入 pipeline → 存入变量       │
  │                                              │    ref_count = 2, 3, ... N                  │
  │                                              │                                             │
  │                                              │ 4. 各种 drop()，逐个 ref_count--           │
  │                                              │                                             │
  │                                              │ 5. 最后一个 WithSource drop()               │
  │                                              │    ref_count() == 1                        │
  │                                              │    notify_on_drop() == true                │
  │                                              │    → PluginInterface::custom_value_dropped()│
  │                                              │      PluginCall::CustomValueOp(Dropped) ────│──►
  │                                              │                                             │ 6. 接收并调用
  │                                              │                                             │    Plugin::custom_value_dropped()
```

### 6.6 Dropped 消息的特殊性

`CustomValueOp::Dropped` 与其他 CustomValueOp 有三个关键不同：

1. **无响应预期**：`dont_send_response = true`，不创建响应订阅，fire-and-forget
2. **无 span**：使用 `Span::unknown()`，因为 Drop 中无上下文
3. **发送方不阻塞**：`drop(self.write_plugin_call(...))`，忽略 JoinHandle

见 `crates/nu-plugin-engine/src/interface/mod.rs` 第 752-763 行、1142-1154 行。

---

## 7. 边界速查表：什么归哪层管？

| 问题 | 归哪层 | 关键代码位置 |
|---|---|---|
| "PluginCall 有几个 variant？" | 协议帧层 | `nu-plugin-protocol::PluginCall` |
| "JSON 编码时末尾加不加 \\n？" | 序列化层 | `crates/nu-plugin-core/src/serializers/json.rs` |
| "本地 socket 超时时间是多少？" | I/O层（CommMode） | `crates/nu-plugin-core/src/communication_mode/mod.rs` 中 `TIMEOUT = 10s` |
| "插件死掉了调用者会卡吗？" | 异常隔离（读线程屏障） | `PluginInterfaceManager::consume_all` |
| "什么时候会自动关掉空闲插件？" | 生命周期（GC） | `PluginGcState::next_timeout` |
| "一条流发太快会 OOM 吗？" | 流复用层（背压） | `StreamWriterSignal HIGH_PRESSURE` |
| "PluginCustomValue 什么时候通知 Dropped？" | 生命周期 + 协议帧边界 | `PluginCustomValueWithSource::drop` + `SharedCow::ref_count() == 1` + `notify_on_drop == true` |

---

## 8. 完整调用时序（Run 一个带流的命令）

```
引擎线程 A (调用 .run())                读线程 (reader loop)             插件子进程
     │                                       │                              │
     │ 1. PluginInterface::plugin_call(Run)  │                              │
     │    ├─ 分配 PluginCallId = 7           │                              │
     │    ├─ init_write_pipeline_data        │                              │
     │    │   (若输入是 ListStream → 分配     │                              │
     │    │    StreamId=42, 起写线程)         │                              │
     │    └─ write(Call(7, Run{..header..})) │ ──────────────────────────► │
     │    (GC.increment_locks(+1))           │                              │
     │                                       │                              │ 2. 解 Call(7)
     │                                       │                              │    执行命令
     │                                       │ ◄──────── Data(42, List(v)) │ (若输入有流,
     │                                       │    → StreamManager 分发      │  逐帧到达)
     │                                       │                              │
     │                                       │ ◄─ CallResponse(7, ListStream│ 3. 返回响应头
     │                                       │      Info{id=99})            │
     │                                       │                              │
     │    ├─ recv_stream_started(7, 99)      │                              │
     │    │   (GC.increment_locks(+1))       │                              │
     │    └─ 返回 PipelineData(ListStream)◄──│                              │
     │                                       │                              │
     │ 4. 用户消费 ListStream                │                              │
     │    (StreamReader 迭代)                │                              │
     │                                       │ ◄── Data(99, List(v1)) ──── │
     │    ← v1 ──  StreamManager → mpsc      │                              │
     │    → Ack(99) ─────────────────────────────────────────────────────► │
     │                                       │ ◄── Data(99, List(v2)) ──── │
     │    ← v2 ──                            │                              │
     │    → Ack(99) ─────────────────────────────────────────────────────► │
     │                                       │ ◄── End(99) ─────────────── │
     │                                       │    recv_stream_ended(99)     │
     │                                       │    (GC.decrement_locks(-1))  │
     │    ← None (迭代结束)                  │                              │
     │                                       │                              │
     │ 5. 所有 PluginInterface drop          │                              │
     │    → Goodbye ──────────────────────────────────────────────────────► │
     │                                       │                              │ 插件退出
     │                                       │ EOF → GC.exited()            │
```

---

## 9. 总结

**协议帧** = "说什么"（枚举 + Serde），**序列化** = "怎么把话编成字节"（Encoder），**I/O层** = "字节从哪走"（Stdio/Socket），**生命周期** = "谁启动/停止/出错时怎么办"（PersistentPlugin + GC + 读线程屏障）。四层通过清晰的 trait 边界（Encoder / PluginRead / PluginWrite / InterfaceManager / Interface）解耦，任何一层的改变不影响其他层。
