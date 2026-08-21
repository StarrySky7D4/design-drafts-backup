# 当前架构

> 本文描述当前可运行架构、数据边界和运行时约束。字节级线路帧、控制消息、历史测试矩阵和背景推导保留在 [协议参考](protocol-reference.md)。每个源文件的 API 与实现细节仍以对应 `Knowledge/<source-path>.md` 为准。

## 1. 系统边界

ModCptLib 由四个相互独立的运行部分构成：

| 部分 | 主要目录 | 职责 | 不负责的事情 |
|---|---|---|---|
| Rust 核心 | `rust/core/` | QUIC 会话、应用消息路由、数据控制、mTLS/证书消费、公告板和信令领域模型 | UI、账号密码数据库的公网服务 |
| PKI 原语 | `rust/pki/` | 无网络/数据库依赖的 CA、CRL、证书策略、CSR 与 SPKI 解析 | realm、设备、credential 或会话授权 |
| 账号与地址簿 | `rust/server/`、`rust/client/` 与 `rust/identity/` | 注册、登录、token、昵称、`user_id -> address` 查询，以及服务器签名的身份 assertion 目录 | NAT 可达性判断、在线状态保证、P2P 流量转发 |
| Flutter 客户端 | `flutter/` | UI、登录编排、FRB 调用和本地会话恢复 | 复制 Rust 的业务状态或协议实现 |
| 调试工具 | `debug-harness/` | 进程分离的证书、注册、发现、信令和直连诊断 | 产品级服务发现或持续回归门 |

当前产品网络边界是**直接 QUIC 拨号**。项目不实现 STUN、TURN、CID relay、NAT 类型探测或自动中继回退；发现服务返回的地址拨号失败时，调用方得到连接错误并自行决定 UI 表达。

## 2. 分层与依赖方向

```text
Flutter UI
  -> AppState / NativeService
  -> flutter_rust_bridge
  -> FRB Rust API
  -> DataCtl / UiStore / ContactRegistry
  -> DataRouter / P2PGateway
  -> P2PNode / P2PChannel / QUIC
```

依赖必须从上向下。`flutter/rust/` 是桥接层，不应承载 QUIC 协议、地址簿状态或群组业务缓存；`DataCtl` 不应依赖 Flutter；`net/` 不应依赖未来 relay 模块。`rust/client` 和 `rust/server` 互相通过稳定的 QUIC RPC 协议协作，二者及未来 core 身份消费路径共用无网络/数据库依赖的 `modcpt_identity`；账号服务仍不依赖 `modcpt_core`。

`modcpt_pki` 是底层无状态 PKI 契约，允许 `core` 和后续 server 消费但不反向依赖它们。`DataCtl` 是 Rust 业务入口，拥有消息、文件、语音、视频、联系人、群组、UI 快照和本地持久化的协调逻辑。`DataRouter` 将已编码的应用消息按 `MsgType` 分派到各控制器；`P2PGateway` 隔离 Quinn 与 Rustls 细节；`P2PNode` 管理 endpoint 和会话；`P2PChannel` 是一个可主动关闭的会话句柄。

## 3. 端到端数据路径

### 3.1 发送

```text
Flutter UI action
  -> AppState command
  -> NativeService / FRB API
  -> DataCtl 或对应 *Ctl
  -> Serializer + DataRouter
  -> P2PGateway.send
  -> P2PChannel.open_stream / send_datagram
  -> QUIC connection
```

文本、文件控制帧、群组控制帧和呼叫控制消息走可靠流。媒体使用的数据报能力由网络层提供，但房间媒体协议尚未实现。发送方不能把 `SessionId`、Quinn stream 或 Rustls 类型泄漏给 Flutter UI。

### 3.2 接收

```text
QUIC connection
  -> per-connection stream dispatcher
  -> P2PGateway event
  -> DataRouter dispatch loop
  -> typed controller receiver
  -> UiStore / ContactRegistry
  -> FRB NodeEvent invalidation signal
  -> AppState.notifyListeners
  -> Flutter snapshot read
```

收到应用数据后，Rust 是业务数据的唯一事实源。Flutter 不保留消息、群组或文件的平行可变缓存；挂件重建时通过 `UiApi`、`ContactApi`、`GroupsApi` 的同步快照读取当前状态。事件流用于通知失效，不用于在 Dart 侧重演 Rust 状态。

## 4. P2P 会话与资源生命周期

`P2PNode` 可以绑定单地址、双栈地址或仅拨号 endpoint。双栈模式使用相互独立的 IPv4 与 IPv6 endpoint，并根据远端地址族选择 endpoint；单栈环境允许降级。

会话的核心标识是 `TransportSessionId`（UUIDv4），不是 `SocketAddr`。一个会话可持有 Active 与 Standby 两条 QUIC 链路，每条链以 UUIDv4 `LinkId` 标识；主链路断开时 watcher 保留备用链的 `LinkId` 并将其提升。路由表使用统一锁维护本地会话和对端会话映射，网络 I/O 必须在锁外执行。

`P2PNode` 实现 `Drop`，用于关闭 endpoint 并让接收任务退出；但产品代码在已知的主动结束点仍必须调用 `P2PChannel::close()`。仅依赖析构无法表达远端关闭语义，也不能替代上层清理 `DataCtl`、FRB 订阅或 UI 状态。

可靠流以 EOF 为消息边界：发送路径结束发送方向，接收路径完整读取且受上限保护。文件控制器因此可以乱序重组块，但大文件仍受内存重组模型限制，见 [Defects](https://github.com/StarrySky7D4/ModCptLib/blob/add9e4024f8e10055508770aaf96c944aab6cceb/Knowledge/DEFECTS.md)。

## 5. 线路协议与保留槽

应用消息使用 8 字节头：`msg_type (u8)`、`version (u8)`、`flags (u16, BE)` 和 `payload_len (u32, BE)`，随后是类型化 payload。消息版本不一致或长度不匹配时拒绝解析。

M4-01 起存在两个已登记 route header version：v1 消息使用 `0x01`；不可变 v2 信封帧（`MsgType::Envelope=0xB0`）只使用 `0x02`。v1 解析路径（`Message::from_bytes`）行为不变，v1/v2 双向拒绝矩阵见 `net::route` 的 `EnvelopeFrame`；两种版本互不解释对方字节。

M3-08/M4-02 后，`Envelope+0x02` 只在 `/3` trusted session 上进入 DataRouter 的 authenticated path：authoritative registry snapshot 先按 fresh clock 清理 expiry，principal realm/user/device、recipient、conversation、signature 与 generation 全部通过后才产生 `AuthenticatedEnvelope`。普通 `subscribe(MsgType::Envelope)` 不接收 raw v2 frame；egress 只使用 immutable plan 中 exact local session ID，不能回退到 contacts、地址、peer list 或 v1 IdentityFrame。

M4-03 的 local-only delivery durability 继续使用 `v2_outbox` per-recipient row、`v2_inbox_dedup` bounded cache 与 `v2_messages` durable replay/conflict authority。M4-05 将同一数据库 additive 升至 SQLite v8，加入 encrypted Olm account/session pickle、persisted bootstrap 与 exact receipt outbox；ratchet/account advance 和 exact outbox/inbox/receipt 在单 transaction 提交。receipt 是 content kind `2` 的 recipient-signed Envelope，并绑定真实 per-recipient `EnvelopeCommitment`；wire digest 仅用于本地 dedup。无 mailbox/relay/push。

M5 的 `LocalDeliveryDispatcher` 只消费 `/3` authoritative trusted-session snapshot。它按逻辑消息重建完整不可变投递计划，只对当前 `Routed` 的 exact device 做 optimistic claim；`Offline`/`Rejected` 不消耗 attempt，claim 后的传输失败消耗一次有界 attempt 且保留原始 wire bytes。调度器不拨号、不发现地址，也不调用 mailbox/relay/push。signed device directory 另以 realm/user/revision pin 和有效期约束缓存；逻辑 direct fanout 上限为 15（peer 8 + self 7），物理 `MessageEnvelope` 收件人上限仍为 8。

M7-01 已选择 Mailbox-first。`rust/mailbox/` 是独立、SQLCipher-backed 的 opaque Envelope upload/fetch/ack engine，并有默认/最大 32 并发、立即 overload、deadline/cancel、唯一终态与 graceful shutdown 的 Tokio runtime；`mailbox_adapter_v1` 只验证 verifier-produced current `SessionBinding`、exact origin 与 canonical sender signature。专用 `modcpt-mailbox/1` QUIC host/client 已要求严格 mTLS、TLS leaf SPKI-bound DeviceHello、连接期 replay guard、双向限长 framing 与 exact-session RequestId owner。依赖方向仍是 `mailbox -> core`，因此 LocalDeliveryDispatcher 和 core QUIC path 不调用 mailbox，也没有失败自动 fallback。Room/媒体、relay 与 push 继续冻结。完整边界见 [Mailbox-first ADR](designs/mailbox-first-v1.md)。

direct-text 使用 Envelope content kind `3`（payload-free bootstrap）和 `4`（native Olm v2 message）。每个 recipient device 有独立 Envelope/ciphertext/session；raw authenticated Envelope 只进入 crate-private pre-decrypt ingress，成功验证 content commitment、native Olm open、encrypted metadata comparison 并持久化后才产生 `AcceptedDirectContent`。该能力不覆盖附件、群组、Room 或平台备份。

| 范围 | 当前用途 |
|---|---|
| `0x00-0x01` | Ping/Pong |
| `0x10-0x13` | 文本、群文本、群创建、类型化身份帧 |
| `0x20-0x22` | 文件元数据、块与完成标记 |
| `0x30` / `0x40` | 语音与视频帧 |
| `0x60-0x62` | 广播相关消息 |
| `0x70-0x71` | DHT 查询与响应 |
| `0x80-0x82` | 呼叫邀请、接受与拒绝 |
| `0x50-0x51`、`0x90-0x93`、`0xA2` | 仅保留线协议编号，不注册 core handler |

保留编号不是功能承诺，也不是授权。未来外部 relay 模块的兼容边界、ACL、取消、配额、防重放和端到端加密要求见 [Relay 模块契约](designs/relay-module-contract.md)。

## 6. 身份、证书与发现

### 6.1 P2P 身份

核心可用 CA 签发身份构建 P2P TLS 配置。证书链信任、对端证书验证、CRL 和策略检查是独立层次：TLS 负责链路认证，`Auth` 负责项目策略与吊销语义。开发特性 `dangerous-skip-tls-verify` 只能用于受控测试，不能被文档描述为生产可达性方案。

`CertPolicy` 的旧三字节/七字节二进制布局仍可解码，以免破坏历史证书；其中历史 relay 字段不再表示 core 权限，也不会触发中继行为。

### 6.2 账号和联系地址簿

`modcpt_server` 提供单向 TLS 的 JSON-over-QUIC RPC。用户通过 account 或 `user_id` 登录，密码使用 Argon2id 哈希，登录返回随机 token。地址簿接口 `set_addr` 以 token 认证并写入最后已知可拨号地址；它不表示在线、不启动 heartbeat，也不提供可达性保证。服务端还持久保存 assertion signer，并可为 active credential 签发 `{user_id, signing key version, P2P cert hash, capability, continuity, validity}` 绑定；客户端必须通过管理员分发的 `ServerProfile` 同时 pin 账号 TLS 证书和 assertion signer 后才验证该 assertion。core 已提供 `open_verified_session`/`connect_verified`，将 assertion 绑定的 exact DER 用于单次 QUIC 拨号及 standby；FRB/Flutter 尚未编排该路径。

Flutter 的 `completeLogin()` 统一完成认证、同机登录锁、节点启动、FRB 快照句柄缓存、服务器客户端绑定、事件订阅和会话持久化。`publishPresence()` 是保留的方法名，实际执行一次 `set_addr` 写入。详细 RPC、状态转换、部署和故障排查见 [账号与发现指南](https://github.com/StarrySky7D4/ModCptLib/blob/add9e4024f8e10055508770aaf96c944aab6cceb/Knowledge/guides/account-and-discovery.md)。

身份与寻址正在进行协议/数据 major upgrade。M3-01 至 M3-07 已完成并满足 M3 gate；M3-08 freshness/session-registry 强化与 M4 runtime 仍等待 reviewer/CI。已实现范围包括 `modcpt_pki`、RealmManifest、server v2 store/RPC、client local-v2、DeviceHello、`/3` trusted session gate、显式 cutover-v2，以及 M4-05 realm-bound signed public content-prekey publish/claim。服务器支持双端点：v1（端口 8421，ALPN `modcpt-server/1`，单向 TLS）和 v2（端口 8422，ALPN `modcpt-server/2`，mTLS）。管理员执行 `init-realm` 创建完整 v2 bundle（RealmRoot、signed manifest、CA、StoreV2），执行 `cutover` 后归档 v1 identity 数据且新 v1 RPC 返回 `upgrade_required`。30 天回滚窗口后 cutover 永久锁定。客户端可通过 v2 provision 流程（CSR + Ed25519 持钥证明）获取签名客户端证书存入 LocalV2Store。所有私钥永不上传，v1 数据绝不导入 v2。完整目标和阶段以[身份与寻址 v2](designs/identity-and-addressing-v2.md)为准。

## 7. 联系人、群组和 UI 状态

`ContactRegistry` 以 `user_id` 作为联系人身份事实源，避免以临时 QUIC session 或地址作为长期联系人键。身份帧将联系人身份传入 Rust；Flutter 仍有部分过渡期 peer 缓存。相关历史去重与 FRB 快照保留在[身份与 FRB 演进记录](designs/peer-identity-and-frb.md)，现行清理工作只按 ROADMAP 与已领取任务推进。

旧 `Group`/`secure_group`/Room 仍不是产品安全群组授权源。M6 另行实现 RFC 9420 产品路径：verified v2 device directory 授权 exact MLS leaf，OpenMLS 是唯一 epoch/content 状态机，SQLCipher 原子提交 provider/governance/immutable wire/outbox/receipt，DataCtl 只通过 authenticated `/3` Envelope 与 LocalOnlyV1 精确投递收敛。Flutter 通过独立 `MlsGroupGovernanceApi` 展示治理，绝不回退到旧群组 owner-in-ID、`IdentityFrame`、SessionId 或 Room HMAC。M7 已选择独立 Mailbox；Room、媒体协商、大规模拓扑、relay 与 push 冻结到未来独立里程碑。完整的历史架构方案与实施裁定分别见：

- [群组与房间架构](designs/groups-and-rooms.md)
- [群组与房间实施细化](designs/groups-and-rooms-implementation.md)

任何新群组/房间实现不得假设 DHT、证书或直接 QUIC 自动解决成员授权、内容保密和 NAT 可达性。

## 8. 并发、持久化和失败边界

- Rust 通过有界通道、信号量和 `try_send` 抑制无界积压；满载时哪些帧可丢弃必须由领域层明确，而不是隐式阻塞 gateway。
- `DataRouter`、会话路由和 UI 存储各自有同步原语；不得在持锁状态执行网络 I/O 或长时间数据库操作。
- SQLite 是本地状态的当前实现。数据库 schema 演进、文件内存重组、FRB 全局锁、token 落盘和服务器阻塞路径仍有明确缺陷，均已登记在 [Defects](https://github.com/StarrySky7D4/ModCptLib/blob/add9e4024f8e10055508770aaf96c944aab6cceb/Knowledge/DEFECTS.md)。
- 调试 harness 是诊断工具。它覆盖证书、公告板、直连和信令控制面，不应作为正式回归门或生产部署规范。

## 9. 文档边界

- 本文是当前行为和边界的项目级入口；已实现、保留与废弃的协议/schema/迁移/错误类别见[协议注册表](PROTOCOL_REGISTRY.md)，跨边界变更、兼容与发布门见[协议生命周期](PROTOCOL_LIFECYCLE.md)。
- [协议参考](protocol-reference.md) 保留根架构文档迁入后的字节级协议、控制时序和历史测试细节；其中与 relay、STUN 或旧在线状态相关的段落不描述当前能力。
- `Knowledge/rust/`、`Knowledge/flutter/`、`Knowledge/debug-harness/` 是源码级说明；源文件改动时必须同步。
- [Roadmap](ROADMAP.md) 只登记尚未完成的方向和非目标；[Defects](https://github.com/StarrySky7D4/ModCptLib/blob/add9e4024f8e10055508770aaf96c944aab6cceb/Knowledge/DEFECTS.md) 只登记活跃风险；[logs/](https://github.com/StarrySky7D4/ModCptLib/tree/add9e4024f8e10055508770aaf96c944aab6cceb/Knowledge/logs) 保存完整日期记录。
