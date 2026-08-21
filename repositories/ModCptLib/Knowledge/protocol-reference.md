# 协议与实现参考

> **迁移说明**：本文由根 `ARCHITECTURE.md` 完整迁入，保留字节级协议、控制时序、数据流和测试背景，不以短摘要替代原内容。
>
> **当前边界优先**：当前运行架构见 [Architecture](ARCHITECTURE.md)。本文中关于 STUN、TURN、CID relay、HMAC relay token、NAT fallback 及其测试的旧描述是迁移保留的历史技术背景，不能视为当前能力或后续路线。现行 relay 边界只以 [Relay 模块契约](designs/relay-module-contract.md) 为准。
>
> **维护方式**：协议仍存在且由当前源码使用时，应同步更新本文与逐文件 Knowledge；已删除模块的背景保留为 Git 可追溯材料，不应新增新的兼容实现。

---

## 0. 文档地图（读什么看什么）

| 你想了解 | 看这份文档 |
|---|---|
| 项目是什么、快速开始、编译运行 | [`README.md`](../README.md) |
| 当前系统边界、数据路径和安全约束 | [Architecture](ARCHITECTURE.md) |
| 代码组织和逐文件说明 | [Structure](STRUCTURE.md)、[Knowledge index](README.md) |
| 账号、联系地址簿与部署 | [账号与发现指南](https://github.com/StarrySky7D4/ModCptLib/blob/add9e4024f8e10055508770aaf96c944aab6cceb/Knowledge/guides/account-and-discovery.md) |
| 群组、房间和 FRB 演进设计 | [`designs/`](designs/) |
| 活跃缺陷与未完成工作 | [Defects](https://github.com/StarrySky7D4/ModCptLib/blob/add9e4024f8e10055508770aaf96c944aab6cceb/Knowledge/DEFECTS.md)、[Roadmap](ROADMAP.md) |
| 逐日实施记录 | [`logs/`](https://github.com/StarrySky7D4/ModCptLib/tree/add9e4024f8e10055508770aaf96c944aab6cceb/Knowledge/logs) |
| AI Agent 工作指南 | [`AGENTS.md`](https://github.com/StarrySky7D4/ModCptLib/blob/add9e4024f8e10055508770aaf96c944aab6cceb/AGENTS.md) |

本文以下内容用于查阅仍存在的线路帧、控制消息和传输约束；当前模块边界以本节链接的项目级文档为准。

---

## 1. P2P 线路帧协议（字节级）

**位置**：`rust/core/src/net/route.rs`

每个应用消息在线上传输使用 **8 字节定长帧头 + 变长载荷**：

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├───────┬───────┬───────────────┬───────────────────────────────┤
│MsgType│ Ver   │     Flags     │          Payload Len           │
│ (8)   │ (8)   │     (16 BE)   │            (32 BE)            │
├───────┴───────┴───────────────┴───────────────────────────────┤
│                                                               │
│                    Payload（Cap'n Proto packed 编码）           │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

- `msg_type` (u8) — 消息类型标识
- `version` (u8) — 协议版本号（当前 `0x01`）；`Message::from_bytes()` 校验不匹配返回 `None`
- `flags` (u16 big-endian) — 标志位
- `payload_len` (u32 big-endian) — 载荷长度
- `payload` — 类型化的 Cap'n Proto 序列化负载

### 1.1 MsgType 完整枚举

```rust
pub enum MsgType {
    Ping = 0x00, Pong = 0x01,
    Text = 0x10,
    FileMeta = 0x20, FileChunk = 0x21, FileComplete = 0x22,
    VoiceFrame = 0x30, VideoFrame = 0x40,
    RelayAlloc = 0x50, RelayData = 0x51,
    Broadcast = 0x60, BroadcastJoin = 0x61, BroadcastLeave = 0x62,
    DhtQuery = 0x70, DhtResponse = 0x71,
    Custom(u8),   // 用户自定义扩展（BytesCtl 用 Custom(0xF0)）
}
```

### 1.2 MessageRouter 分派模型

```
MessageRouter::bind(addr)
    ├─ 创建 P2PGateway
    ├─ handlers: HashMap<u8, Arc<dyn MessageHandler>>
    ├─ 注册默认 PingHandler
    └─ 分派循环：
        ├─ GatewayEvent::Connected/Disconnected → lifecycle_tx
        └─ GatewayEvent::Data → 解析 Message → 按 msg_type 查 handler
            ├─ 有 handler → handle() → 可选回复
            └─ 无 handler → debug 日志
```

`MessageHandler` trait 返回 `Pin<Box<dyn Future>>`（保持 `dyn` trait 对象兼容）：

```rust
pub trait MessageHandler: Send + Sync + 'static {
    fn handle<'a>(&'a self, peer: &'a NodeRef, msg: Message)
        -> Pin<Box<dyn Future<Output = RouteResult> + Send + 'a>>;
}
pub enum RouteResult { Handled { reply: Option<Message> }, Unhandled }
```

---

## 2. P2P 控制消息协议（字节级）

**位置**：`rust/core/src/net/p2p.rs`（内部 `ControlMsg`）

控制消息经 QUIC 双向流传输，**首字节 `StreamType::Control (0x00)`**。

### 2.1 Bind 消息（`0x01`）

```
目的：注册一条新链路到现有会话

Byte 0      : 0x01 (消息标签)
Byte 1-16   : 对端 SessionId (UUID v4, 16 字节)
Byte 17-24  : LinkId (u64 little-endian)
Byte 25     : LinkRole (0x01=Active, 0x02=Standby)
```

### 2.2 BindAck 消息（`0x02`）

```
目的：确认链路绑定

Byte 0      : 0x02
Byte 1-16   : 本端 SessionId
Byte 17-24  : LinkId (需与 Bind 中的 LinkId 匹配)
Byte 25     : LinkRole (需与 Bind 中的 role 匹配)
```

### 2.3 Switch 消息（`0x03`）

```
目的：请求对端迁移到新地址

Byte 0      : 0x03
Byte 1      : 是否包含地址 (0x00=无, 0x01=有)
如果有地址:
  Byte 2    : 地址族 (0x04=IPv4, 0x06=IPv6)
  Byte 3-6  : IPv4 (4 字节) / IPv6 (16 字节, Byte 3-18)
  Byte 7-8  : 端口 (u16 big-endian) / IPv6 时 Byte 19-20
```

### 2.4 Close 消息（`0xFF`）

```
目的：通知对端关闭
Byte 0 : 0xFF，无额外负载
```

### 2.5 握手时序

```
发起方 (Active)                     接收方 (入站)
     │                                   │
     ├─ QUIC 连接建立 ─────────────────▶ │
     ├─ 发送 Bind(session_id, link_id, role) ─▶│
     │                                   ├─ 查找/创建会话
     │                                   ├─ 存储连接 (store_active/store_standby)
     │                                   └─ 回复 BindAck ──▶
     ◀── BindAck 到达，握手完成 ──────────┤
```

### 2.6 SessionId / LinkId 语义

| 类型 | 生成方式 | 用途 |
|---|---|---|
| `SessionId` | UUID v4 (16 字节) | 标识一个本地会话；两端可用不同 ID 指代同一逻辑会话 |
| `LinkId` | `rand::random()` (u64) | 标识一条具体物理链路，用于 BIND 握手双向校验 |

**路由键**：底层路由表使用**对端的 SessionId**（而非 IP 地址）作为键，正确处理 NAT 重绑定与地址迁移。

---

## 3. Cap'n Proto Schema

**位置**：`rust/core/schema/message.capnp`（ID `@0x9c4e1a7b3d5f6082`，由 `build.rs` 编译）

```capnp
struct Header   { msgType @0 :UInt8; version @1 :UInt8; flags @2 :UInt16; }
struct Envelope { header @0 :Header; payload @1 :Data; }
struct TextMessage  { content @0 :Text; }
struct PingMessage  { timestamp @0 :UInt64; }
struct PongMessage  { timestamp @0 :UInt64; }
struct FileMeta     { name @0 :Text; size @1 :UInt64; numChunks @2 :UInt32; }
struct FileChunk    { seq @0 :UInt32; total @1 :UInt32; data @2 :Data; }
struct VoiceFrame   { seq @0 :UInt32; timestamp @1 :UInt64; data @2 :Data; }
struct VideoFrame   { seq @0 :UInt32; timestamp @1 :UInt64; width @2 :UInt16; height @3 :UInt16; data @4 :Data; }
```

### MsgType ↔ Schema ↔ 序列化器映射

| MsgType | 常量 | schema 类型 | `Serializer` 方法 |
|---|---|---|---|
| `0x00` | `Ping` | `PingMessage` | `serialize_ping` / `deserialize_ping` |
| `0x01` | `Pong` | `PongMessage` | `serialize_pong` / `deserialize_pong` |
| `0x10` | `Text` | `TextMessage` | `serialize_text` / `deserialize_text` |
| `0x20` | `FileMeta` | `FileMeta` | `serialize_file_meta` / `deserialize_file_meta` |
| `0x21` | `FileChunk` | `FileChunk` | `serialize_file_chunk` / `deserialize_file_chunk` |
| `0x30` | `VoiceFrame` | `VoiceFrame` | `serialize_voice_frame` / `deserialize_voice_frame` |
| `0x40` | `VideoFrame` | `VideoFrame` | `serialize_video_frame` / `deserialize_video_frame` |
| 通用 | — | `(Header, Data)` | `serialize_envelope` / `deserialize_envelope` |

`Serializer` 为纯函数式静态方法，使用 packed 编码；`read_packed!` 宏统一处理空数据（返回 `Truncated`）与解析。

---

## 4. 数据流时序（端到端深度版）

### 4.1 发送端（App → 对端）

```
Application (Flutter/Rust)
    │  DataCtl::send_text("peer-1", "Hello")
    ▼
MsgCtl::send(target, "Hello")
    │  Serializer::serialize_text("Hello") → Vec<u8>
    ▼
DataRouter::send(&Target::Single(node_ref), MsgType::Text, &payload)
    │  encode_wire(MsgType::Text, &payload) → wire_bytes
    ▼
P2PGateway::send(&node_ref, &wire_bytes)
    │  DataRouter::peer(id) → NodeRef{ session_id }
    │  P2PNode::get_channel(session_id) → P2PChannel
    ▼
P2PChannel::open_stream()
    │  state.read().await.active.conn.clone()
    │  conn.open_bi() → (SendStream, RecvStream)
    │  send.write_all(StreamType::AppData + payload)
    ▼
QUIC Connection ⟶ 对端
```

### 4.2 接收端（对端 → App）

```
QUIC Connection
    │
    ▼
Stream Dispatcher (per-connection task)
    │  conn.accept_bi() → (send, recv)
    │  recv.read_exact(&[StreamType])
    │  StreamType::AppData → app_rx.try_send((send, recv))
    ▼
P2PGateway::accept_incoming (data loop)
    │  channel.accept_stream() → recv buf
    │  GatewayEvent::Data { addr, node_ref, data }
    ▼
DataRouter::dispatch_loop
    │  Message::from_bytes(&data) → { header, payload }
    │  subs[header.msg_type].try_send(InboundFrame{ from, msg_type, payload })
    ▼
MsgCtl::recv()
    │  rx.recv() → InboundFrame
    │  Serializer::deserialize_text(&payload) → "Hello"
    ▼
Application → TextMessage{ from: "peer-1", content: "Hello" }
```

### 4.3 文件传输（分块 + 乱序容忍）

```
发送方:
  FileMeta{ name: "doc.pdf", size: 1MB, num_chunks: 17 } ─▶ FileMeta (0x20)
  FileChunk{ seq:0, total:17, data: chunk0 }             ─▶ FileChunk (0x21)
  ...
  FileChunk{ seq:16, total:17, data: chunk16 }           ─▶ FileChunk
  FileComplete (empty payload)                            ─▶ FileComplete (0x22)

接收方 (per-sender Reassembly{ meta, chunks: HashMap, received: u32 }):
  ├─ 收到 FileMeta   → meta = Some(...)
  ├─ 收到 FileChunk  → chunks.insert(seq, data); received += 1
  │     校验: seq < MAX_CHUNKS_PER_FILE(4096)
  │            total ≤ MAX_CHUNKS_PER_FILE
  │            data.len() ≤ MAX_CHUNK_BYTES(256KB)
  │     违规 → 丢弃该 sender 的整个重组状态
  ├─ 收到 FileComplete → 忽略（完整性由 chunk 接收驱动）
  └─ is_complete() → assemble() → ReceivedFile{ from, name, data }
```

**安全常量**：`FILE_CHUNK_SIZE=60000`、`MAX_CHUNKS_PER_FILE=4096`、`MAX_CHUNK_BYTES=256KB`、`MAX_PENDING_SENDERS=256`、`REASSEMBLY_TTL=60s`、`MAINTENANCE_INTERVAL=5s`。

### 4.4 双链路故障转移时序

```
初始状态: Active=conn1, Standby=conn2

conn1 断开:
  connection_watcher(conn1, Active) 检测到 conn.closed()
    ├─ 检查 conn_id 匹配（防新旧连接竞争）
    ├─ active = None; standby = Some(conn2)
    ├─ promote_standby: active = Some(conn2); standby = None
    ├─ spawn_connection_watcher(conn2, new_id, Active)
    ├─ event_tx.send(LinkSwitch)
    └─ event_tx.send(Connected)

后续操作:
    ├─ ch.open_stream() → 使用新的 active (原 standby)
    └─ conn2 也断开 → 清理路由表（local_sessions.remove + remote_sid_to_local.retain）
```

---

## 5. 事件与错误体系

### 5.1 NodeEvent / ChannelEvent

| 事件 | 触发时机 |
|---|---|
| `NodeEvent::NewSession { session_id, remote, event_rx }` | 入站连接创建新会话 |
| `ChannelEvent::Connected { session_id }` | Active 链路首次建立 |
| `ChannelEvent::Disconnected { session_id, link_role, reason }` | 某条链路断开 |
| `ChannelEvent::LinkBound { session_id, link_role }` | 新链路绑定完成 |
| `ChannelEvent::LinkSwitch { session_id }` | Standby 提升为 Active |
| `ChannelEvent::Migration { session_id, new_addr }` | 收到对端迁移请求 |

### 5.2 P2pError

```rust
pub enum P2pError {
    ConnectionFailed(String), StreamOpenFailed(String),
    DatagramFailed(String),   ConfigError(String),
    TlsError(String),         SessionMismatch,
    UnsupportedProtocol(String), MalformedFrame(String),
    NotConnected,             ChannelClosed, SelfConnection(SocketAddr),
}
```

`P2PGateway` 将其转为领域错误；`UnsupportedProtocol` 与 `MalformedFrame` 保持独立，其他类别映射为 `ConnectionFailed/NotConnected/SendFailed/RecvFailed/Timeout/Closed/InvalidConfig`。不存在 core relay 错误类别。

### 5.3 P2pConfig 关键字段

```rust
pub struct P2pConfig {
    pub keep_alive_interval: Option<Duration>,  // 默认 5s
    pub max_idle_timeout: Option<Duration>,     // 默认 30s
    pub initial_rtt: Duration,                  // 默认 100ms
    pub alpn_protocol: Vec<u8>,                 // 仅接受 b"modcpt-p2p/2"
    pub server_name: String,                    // 默认 "modcpt.local"
    pub tls: Option<TlsIdentity>,               // TLS 身份
    pub root_store: Option<Arc<RootCertStore>>, // 自定义 CA 信任库
}
```

---

## 6. 并发模型与通道容量

> 锁策略总表见 [`STRUCTURE.md`](STRUCTURE.md)。本节补充通道容量与关键决策。

### 6.1 通道容量

| 通道 | 容量 | 满时行为 |
|---|---|---|
| 控制流分发 (ctrl_rx) | 64 | `try_send` 失败 → warn → 关闭流 |
| 应用流分发 (app_rx) | 256 | `try_send` 失败 → warn → drop |
| NodeEvent / ChannelEvent | 64 | mpsc channel |
| GatewayEvent | 256 | mpsc channel |
| 订阅者 InboundFrame | 128 | `try_send` 失败 → warn → drop 该帧 |
| 待发送缓冲 (pending) | 64/type | 满则丢弃旧帧 |
| 生命周期通知 | 256 | `try_send` 失败 → warn → drop |

### 6.2 关键并发决策

- **单锁路由表** `RwLock<RouterState>`：`global_accept_loop` 与 `connection_watcher` 之间无 AB-BA 死锁路径。
- **锁外异步 I/O**：所有锁范围严格限定状态读写，绝不持锁 `await` 网络；广播/DHT "锁内 clone，锁外发送"。
- **洪泛防护**：入站 `Semaphore(64)` 在 `spawn` **之前** `acquire_owned()`；TURN 独立 `Semaphore(32)`；`modcpt_server` 全局 `Semaphore(256)` + per-connection `Semaphore(64)` + 流读超时 10s。

---

## 7. 测试用例清单（逐项）

> 总数：**195 passed / 0 failed**（core 174 + server 12 + client 3 + e2e 2 + msg_api 4）。

### 7.1 p2p.rs 单元测试

| 测试 | 说明 |
|---|---|
| `test_session_id_unique` | SessionId 全局唯一 |
| `test_link_id_unique` | LinkId 全局唯一 |
| `test_control_msg_roundtrip` | 所有控制消息编码/解码循环 |
| `test_switch_bad_address_type` | 无效地址类型解码返回 None |
| `test_stream_type_roundtrip` | StreamType 编解码 |
| `test_generate_identity` | 自签名证书生成 |
| `test_node_bind_and_close` | 节点绑定与关闭生命周期 |
| `test_accept_stream_no_connection` | 无连接时 accept_stream 返回 NotConnected |

### 7.2 route.rs 单元测试

| 测试 | 说明 |
|---|---|
| `test_header_encode_decode_roundtrip` | 帧头编解码循环 |
| `test_header_decode_insufficient_data` | 截断数据解码返回 None |
| `test_msg_type_roundtrip_all_variants` | 所有 15 个 MsgType 编解码 |
| `test_msg_type_custom_roundtrip` | Custom 类型编解码 |
| `test_message_text/ping/pong` | 消息构造器 |
| `test_message_encode_decode_roundtrip` | 消息编解码循环 |
| `test_message_from_bytes_wrong_version` | 版本不匹配拒绝 |
| `test_message_from_bytes_truncated_payload` | 截断载荷拒绝 |
| `test_ping_handler_responds_with_pong` | PingHandler 回复 Pong |
| `test_ping_handler_non_ping_returns_unhandled` | 非 Ping 返回 Unhandled |

### 7.3 serializer.rs 单元测试

| 测试 | 说明 |
|---|---|
| `test_envelope_roundtrip` | 信封编解码 |
| `test_envelope_empty_payload` / `_large_payload(100KB)` | 空载荷 / 大载荷 |
| `test_envelope_empty_input_truncated` | 空输入返回 Truncated |
| `test_envelope_garbage_decode_fails` | 垃圾数据解码失败 |
| `test_text_roundtrip` | 文本编解码（含 Unicode） |
| `test_ping_roundtrip` / `test_pong_roundtrip` | Ping/Pong 编解码 |
| `test_file_meta_roundtrip` | 文件元数据编解码 |
| `test_file_chunk_roundtrip` / `_empty_data` | 文件块编解码 |
| `test_voice_frame_roundtrip` / `test_video_frame_roundtrip` | 音视频帧编解码 |

### 7.4 file.rs 单元测试

| 测试 | 说明 |
|---|---|
| `test_assemble_in_order` | 顺序块正常组装 |
| `test_assemble_without_meta_is_err` | 无元数据不完成 |
| `test_reassembly_out_of_order_completes` | 乱序块正确重组 |
| `test_reassembly_rejects_oversized_chunk` | 超大块拒绝 |
| `test_reassembly_rejects_too_many_chunks` | 超多块拒绝 |

### 7.5 集成测试（demo/tests/p2p_integration.rs，12 组）

| 测试模块 | 覆盖场景 |
|---|---|
| `certificate` | 有效证书、多域名、可克隆 |
| `node_binding` | 临时端口、回环地址、端口冲突、关闭后重建、事件通道、endpoint 访问 |
| `session` | 基础建立、IDs 不同、连接拒绝、多重会话、事件触发、多消息序列、大载荷(65KB)、二进制载荷、双向流 |
| `datagram` | 单数据报、空数据报、近 MTU |
| `unreliable_channel` | 基础收发、最大数据报大小 |
| `status_query` | 建连后状态、Standby 后状态、关闭后状态 |
| `standby` | 建立 Standby、Active 故障转移、重复连接 |
| `migration` | 请求迁移、Standby+迁移 |
| `concurrent_streams` | 3 路 / 10 路并发、不同大小载荷并发 |
| `error_handling` | 无效 session_id、超时接收、关闭后操作、无 Standby 提升 |
| `lifecycle` | 完整生命周期、三节点拓扑 |

### 7.6 端到端测试（client + server）

- `rust/server/src/quic.rs` — 12 个 dispatch/store/cert 单测。
- `rust/client/tests/e2e_quic.rs` — register→login→set_addr→lookup→lookup_nick→health 全链路，走真实 QUIC/TLS + pin 证书；含 **ALPN/realm 漂移检测**（unilateral 修改 client/server 常量会测试失败）。

### 7.7 测试辅助模式

```rust
// 创建双节点会话
async fn two_node_session() -> (P2PNode, P2PNode, P2PChannel, P2PChannel) {
    let (node_a, _a_ev) = bind_node().await;
    let (node_b, mut b_ev) = bind_node().await;
    let addr_b = node_b.local_addr().unwrap();
    let (ch_a, _a_ch_ev) = node_a.open_session(addr_b).await.expect("open_session");
    let ch_b = match timeout(Duration::from_secs(5), b_ev.recv()).await... {
        NodeEvent::NewSession { session_id, .. } =>
            node_b.get_channel(session_id).expect("get_channel B")
    };
    (node_a, node_b, ch_a, ch_b)
}
```

---

## 8. 已知待修复项（M1-M11）

当前缺陷状态与关闭条件见 [`DEFECTS.md`](https://github.com/StarrySky7D4/ModCptLib/blob/add9e4024f8e10055508770aaf96c944aab6cceb/Knowledge/DEFECTS.md)。本节的历史编号仅用于理解旧测试背景。

**已修复**（2026-06-28 审查后）：5 项 P0（C1-C5：TLS 互信/消息截断/资源泄漏/TURN 转发/Flutter 命令注入）+ 8 项 P1（H1-H8）+ 4 项 P2（M2/M3/M7/TurnRelay 自省）。

**待修复（低优先级整改）**：

| 编号 | 位置 | 说明 |
|---|---|---|
| M1 | `p2p.rs` Standby→Active 提升两条路径 | 应统一为单一路径 |
| M4 | `p2p.rs` `accept_stream` | 只读 `active.app_rx`，standby 应用流可能积压 |
| M5 | `p2p.rs` encode/decode 常量 | 用不同常量（27 vs 26），需命名常量 |
| M6 | `p2p.rs` 连接关闭日志 | info 噪声，可降为 debug |
| M8 | `p2p.rs` 套接字探测 | 弱非阻塞探测，宜用 `nonblocking()` |
| M9 | `dht.rs` 路由表键 | namespace 双重键 + bootstrap 键泄漏 |
| M10 | 多处 `join_all` | 群组/DHT 广播缺每成员超时 |
| M11 | `dht.rs` / `center_broadcast.rs` | 事件转发 `.await` send 可能反压 gateway |

STUN/NAT 探测实现已从 core 移除；与之相关的旧缺陷不再是当前代码项，也不能作为恢复该模块的依据。

---

## 9. 自签名互信约束（重要安全限制）

`build_client_tls_config` 把**本端**身份证书加入自己信任库：

```rust
if let Some(ref tls_id) = config.tls {
    for cert in &tls_id.cert_chain {
        let _ = root_store.add(cert.clone());  // 把"自己的证书"加入自己的信任库
    }
}
```

⚠️ **重要限制**：上述逻辑只满足"一个节点信任它**自己**生成的证书"。当两个节点**各自独立**生成身份时：

- 节点 A 的信任库只含 A 的证书，而服务端 B 呈现的是 B 的证书——A **无法**校验 B，反之亦然。
- `P2pConfig::default()`（`tls = None`）下，两个独立节点**无法**通过 TLS 校验互连。
- 该自动互信**仅在双方共享同一份身份**（同一证书+私钥）时才成立。
- 当前唯一"开箱即用"的互连方式是启用 `dangerous-skip-tls-verify`（demo 与 Flutter Debug 默认启用），但这会跳过全部证书校验，**仅限开发/测试**。
- **生产互信**应通过 `serve/ca` 子系统（`RootCa`+`IntermediateCa`）签发证书并将根证书预置到各节点 `root_store`，或显式钉扎对端证书。

---

## 10. 访问控制状态

| 功能 | 机制 | 状态 |
|---|---|---|
| Relay 权限 | core 不签发、不验证、不转发 relay 数据；保留线编号仅供未来外部模块契约 | 当前非目标 |
| 连接限流 | Semaphore 上限（入站 64 / TURN 32 / server 256+64） | ✅ 已实现 |
| 通道背压 | 有界 mpsc + `try_send` | ✅ 已实现 |
| 消息认证 | `serve/auth.rs` mTLS 接线（`build_p2p_config`） | ✅ 已实现（生产需 CA） |
| 账号服务 TLS | 单向 TLS + pin 证书，fail-closed | ✅ 已实现 |
| 重复登录防护 | Flutter `SessionLock`（userId+pid 锁文件） | ✅ 已实现（rewrite/auth-flow） |

---

> **维护约定**：本文档聚焦协议字节布局、数据流时序、测试清单和安全约束等深度细节。项目结构、模块职责和进度路线图更新至 [`STRUCTURE.md`](STRUCTURE.md)、[`ARCHITECTURE.md`](ARCHITECTURE.md) 与对应逐文件 Knowledge 文档；修改源码后同步受影响章节。
