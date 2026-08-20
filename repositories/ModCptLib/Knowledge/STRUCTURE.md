# ModCptLib 代码组织结构总览

> 本文档描述 ModCptLib 项目整体的代码组织结构、分层架构与模块依赖关系。
> 每个源文件的具体细节见同目录下对应的 `.md` Knowledge 文档。
> 创建日期：2026-07-07 ｜ 最后更新：2026-08-06（M7 Mailbox-first、兼容矩阵与 Windows MSIX dry-run）

---

## 1. 项目定位

**ModCptLib** 是一个基于 **QUIC 协议** 的高性能点对点（P2P）通信库，采用 **Rust 核心引擎 + Flutter UI** 的跨平台架构，通过 `flutter_rust_bridge`（FRB）实现零开销 FFI 绑定。

核心能力：多会话管理、Active/Standby 双链路冗余、可靠流 + 不可靠数据报、群组消息、账号 + 联系地址簿、TLS 1.3 加密、DHT 分布式发现、SQLite 本地持久化、音视频通话。

---

## 2. 顶层目录结构

```
ModCptLib/
├── rust/                  ← Rust workspace（核心逻辑，8 个 crate）
│   ├── core/              ← modcpt_core   核心引擎（net / data_ctl / serve）
│   ├── client/            ← modcpt_client 账号/在线状态 QUIC 客户端
│   ├── identity/          ← modcpt_identity 共享 assertion/ServerProfile/强类型 ID 契约
│   ├── pki/               ← modcpt_pki 无网络/数据库依赖的 CA/CSR/SPKI 原语
│   ├── server/            ← modcpt_server 用户发现服务（二进制 + 库）
│   ├── msg_api/           ← modcpt_msg_api 消息 API 封装
│   ├── group_mls/         ← modcpt_group_mls OpenMLS 小群安全组件
│   └── mailbox/           ← modcpt_mailbox 外部 opaque Envelope mailbox 服务
├── flutter/               ← Flutter 客户端
│   ├── rust/              ← modcpt_frb   FRB 薄胶水层（Rust）
│   ├── lib/               ← Dart 应用代码（services / models / ui）
│   └── src/rust/          ← FRB 自动生成的 Dart 绑定
├── demo/                  ← Rust 端到端演示（10 场景）+ 集成测试
├── debug-harness/         ← 调试工具集（node/peer/server/mesh 多二进制）
├── scripts/               ← 统一验证、server/Mailbox ZIP + app MSIX 生成/消费与 RustSec 门禁
├── supply-chain/          ← 机器可读 compatibility matrix 与有期限、带 owner 的 RustSec 例外策略
├── Knowledge/             ← ★ 唯一文档目录（当前架构、设计、复核、日志、逐文件说明）
├── temp/                  ← 临时分析文件（非文档入口）
├── README.md              ← 项目说明
├── PUSH_NOTES.md          ← 本地推送/部署笔记（已 gitignore）
├── flutter_rust_bridge.yaml ← FRB 代码生成配置
└── .gitignore
```

---

## 3. 分层架构（自顶向下）

```
┌─────────────────────────────────────────────────────────────┐
│  Flutter UI 层 (Dart)                                       │
│  login / main(contact·msg·call·self) / widgets              │
│  ↓ 通过 NativeService 抽象                                  │
├─────────────────────────────────────────────────────────────┤
│  Flutter Services / Models 层 (Dart)                        │
│  FrbNativeService (实现) · app_state · file_crypto · models │
│  ↓ flutter_rust_bridge FFI                                  │
├─────────────────────────────────────────────────────────────┤
│  FRB 胶水层 (Rust: modcpt_frb)                              │
│  api/mod.rs (RustNode) · api/server_client.rs (newtype)     │
│  · log_bridge.rs (tracing→Stream)                           │
│  ↓ 依赖                                                     │
├─────────────────────────────────────────────────────────────┤
│  数据控制层 (Rust: data_ctl)                                │
│  DataCtl 门面 → MsgCtl/FileCtl/VoiceCtl/VideoCtl/BytesCtl   │
│  DataRouter (订阅/分发/群组) · Serializer (Cap'n Proto)     │
│  ↓ P2PGateway 防腐层                                        │
├─────────────────────────────────────────────────────────────┤
│  网络核心层 (Rust: net)                                     │
│  P2PNode / P2PChannel (QUIC) · route (线路协议)             │
│  dht / center_broadcast / route                              │
│  ↓ quinn + rustls                                           │
├─────────────────────────────────────────────────────────────┤
│  服务层 (Rust: serve)                                       │
│  ca(两级CA) · auth · crypto · key_store · database          │
│  bulletin_board · signaling                                 │
├─────────────────────────────────────────────────────────────┤
│  独立 crate                                                 │
│  modcpt_client (QUIC RPC 客户端) · modcpt_server (发现服务) │
│  modcpt_msg_api (异步 LLM_bot 接口)                          │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Rust Workspace 成员

| Crate | 路径 | 类型 | 职责 |
|-------|------|------|------|
| `modcpt_core` | `rust/core/` | rlib | 核心：网络层 + 数据控制层 + 服务层 |
| `modcpt_client` | `rust/client/` | lib | 账号/在线状态 QUIC 客户端（单向 TLS + JSON RPC）；含隔离 v2 本地身份 store（M3-04，独立 SQLite `user_version` 域，不接线 live provision/QUIC） |
| `modcpt_identity` | `rust/identity/` | lib | identity/realm/device canonical 契约，以及 M4 Envelope/receipt/direct-content/prekey canonical wire |
| `modcpt_pki` | `rust/pki/` | lib | 无网络/数据库依赖的 CA、CRL、证书策略、叶证书/SPKI 与 PKCS#10 CSR 原语 |
| `modcpt_server` | `rust/server/` | bin+lib | 用户发现服务；v2 store schema 2 含 realm-bound signed content-prekey publish/atomic claim/rate/revoke |
| `modcpt_msg_api` | `rust/msg_api/` | lib | 异步消息 API（包装 `DataCtl`，LLM_bot/自动化接口；含扩展消息） |
| `modcpt_group_mls` | `rust/group_mls/` | lib | M6 OpenMLS profile/runtime/authority/storage contract |
| `modcpt_mailbox` | `rust/mailbox/` | lib | M7 独立 opaque Envelope upload/fetch/ack SQLCipher engine；依赖方向 `mailbox -> core` |
| `modcpt_frb` | `flutter/rust/` | cdylib+staticlib | FRB 胶水层，桥接 core+client → Dart |
| `modcpt_debug` | `debug-harness/` | 独立 | 调试多二进制工具集 |
| `modcpt_demo` | `demo/` | 独立 | 端到端演示 + 集成测试 |

---

## 5. modcpt_core 内部分层（最关键 crate）

### 5.1 `net/` — 网络核心层
| 文件 | 行数 | 职责 | Knowledge |
|------|------|------|-----------|
| `p2p.rs` | ~2400 | ★ QUIC 会话/通道/双链路/TLS/mTLS | `Knowledge/rust/core/net/p2p.md` |
| `p2p_gateway.rs` | ~700 | 防腐层，隐藏 quinn/rustls 类型 | `p2p_gateway.md` |
| `route.rs` | ~500 | 线路帧协议 + MsgType + MessageRouter | `route.md` |
| `dht.rs` | ~200 | 分布式哈希表广播 | `dht.md` |
| `center_broadcast.rs` | ~250 | 集中式广播（中心-成员） | `center_broadcast.md` |

### 5.2 `data_ctl/` — 数据控制层
| 文件 | 职责 | Knowledge |
|------|------|-----------|
| `mod.rs` | DataCtl 门面（统一入口）+ 协议解析分发 + `now_ms` | `data_ctl/mod.md` |
| `msg.rs` | MsgCtl / GroupMsgCtl 文本 | `data_ctl/msg.md` |
| `file.rs` | FileCtl 分块传输 + 乱序重组 | `data_ctl/file.md` |
| `file_transfer_v2.rs` | TransferId 文件 frame、ACK waiter 与精确 cancel owner | `data_ctl/file_transfer_v2.md` |
| `voice.rs` / `video.rs` | 音视频帧控制器 | `voice.md` / `video.md` |
| `bytes.rs` | 原始字节通道（Custom 0xF0） | `bytes.md` |
| `contacts.rs` | ★ ContactRegistry（userId 身份注册表，阶段 B1） | `data_ctl/contacts.md` |
| `custom.rs` | 扩展消息（ExtMessage/CustomCtl） | `data_ctl/custom.md` |
| `proto.rs` | 文本协议命令解析（`/call_`、`/file_*` 等 9 类） | `data_ctl/proto.md` |
| `ui_store.rs` | ★ UiStore（消息/文件/通话/群文件单一事实源，Phase D） | `data_ctl/ui_store.md` |
| `database.rs` | ★ DatabaseCtl SQLite v8（legacy/write-behind + direct encrypted state/atomic delivery） | `data_ctl/database.md` |
| `envelope_v2.rs` | M4 `/3` principal-bound Envelope ingress gate | `data_ctl/envelope_v2.md` |
| `convert/router.rs` | ★ DataRouter 中央路由器 | `convert/router.md` |
| `convert/serializer.rs` | Cap'n Proto 序列化 | `convert/serializer.md` |

Core 根包括 `delivery_v2.rs`、`delivery_state_v2.rs`、`delivery_runtime_v2.rs`、`device_directory_v2.rs`、`transfer_v2.rs`，以及 M4-05 `direct_e2ee.rs`（native Olm v2、two-phase bootstrap、post-decrypt gate、SQLite v8 transaction owner）。M5 新增 LocalOnlyV1 精确收件人调度与有界 signed device-directory pin/cache；M6 新增 SQLCipher 原子群存储、verified directory → MLS exact-leaf authority/fan-out/lifecycle bridge、`group_mls_product_v1.rs` 单一产品 owner 与 `group_mls_transport_v1.rs` 有界认证封装。M7 在 core 只新增纯 `mailbox_adapter_v1.rs`，没有 mailbox listener 或 fallback。独立 `rust/group_mls/` 持有 OpenMLS 契约；独立 `rust/mailbox/` 以 `mailbox -> core` 实现 opaque Envelope storage engine、bounded runtime/RPC、`MCMBX001` framing、专用 mTLS/DeviceHello QUIC host/client、loopback-only OpenMetrics exporter、`mailboxd_v1` daemon、跨主机 soak harness 和 cold-backup/quarantine/restore recovery harness。逐文件文档位于对应 `Knowledge/rust/` 子目录。

### 5.3 `serve/` — 服务/基础设施层
| 文件 | 职责 | Knowledge |
|------|------|-----------|
| `ca/mod.rs` | `modcpt_pki` 兼容 re-export | `serve/ca/mod.md` |
| `auth.rs` | 证书签发/验证 + mTLS 接线 | `serve/auth.md` |
| `crypto.rs` | Argon2id + ChaCha20-Poly1305 | `serve/crypto.md` |
| `key_store.rs` | CA 私钥冷存储 | `serve/key_store.md` |
| `database.rs` | NodeStore + SqliteStore | `serve/database.md` |
| `bulletin_board.rs` | 集中式用户发现 | `serve/bulletin_board.md` |
| `signaling.rs` | 呼叫信令 + 防骚扰状态机 | `serve/signaling.md` |

---

## 6. Flutter 端组织

### 6.1 FRB 胶水层（Rust）`flutter/rust/src/`
| 文件 | 职责 |
|------|------|
| `api/mod.rs` | RustNode（`Arc<Mutex<DataCtl>>` + contacts/router/ui/server 缓存）+ create_node + log_stream + event_stream |
| `api/{contacts,messaging,groups,files,events,ui}.rs` | 阶段 C/D 拆分的分域子句柄（sync 快照 + userId 寻址 + 事件流） |
| `api/{secure_group,room}.rs` | **R2/R3 新增** 安全群组/仅直连房间状态机宿主子句柄（只读视图 + 验证门 apply 预签名事件，无网络 I/O） |
| `api/server_client.rs` | Arc 包装 modcpt_client；M7 start/status/cancel 与有界结果适配 |
| `log_bridge.rs` | tracing → mpsc → FRB StreamSink |
| `frb_generated.rs` | FRB 自动生成（勿手改） |

### 6.2 Dart 应用 `flutter/lib/`
| 子目录 | 内容 |
|--------|------|
| `services/` | native_service(契约) · frb_native_service(实现) · app_state · frb_runtime · session_lock · format_helpers |
| `test/` | Flutter 生命周期、绑定兼容、安全存储与 UI 回归；新增源码测试须在 `Knowledge/flutter/tests/` 建立逐文件说明 |
| `models/` | server_models（Phase D 唯一保留的模型文件，chat/call/file/group_log 已下沉 Rust） |
| `ui/login/` | login_page · register_page |
| `ui/main/` | main_page（三栏布局）|
| `ui/main/contact/` | contact_frame · friends_list · groups_list |
| `ui/main/msg/` | msg_frame · msg_card · search_bar |
| `ui/main/call/` | call_screen |
| `ui/main/self/` | self_frame · self_info_page · files_page · setting_page |
| `ui/widgets/` | voice_recorder · voice_player · emoji_picker · file_announce_card |
| `src/rust/` | FRB 自动生成的 Dart 绑定（勿手改） |

---

## 7. 关键数据流（端到端）

**发送方向**：
```
App → *Ctl.send() → Serializer.serialize()
  → DataRouter.send(Target, MsgType, bytes)
    → P2PGateway.send(NodeRef, wire_bytes)
      → P2PChannel.open_stream()/send_datagram()
        → QUIC Connection → 对端
```

**接收方向**：
```
QUIC Connection → Stream Dispatcher
  → P2PGateway.accept_incoming()
    → GatewayEvent::Data → dispatch_loop（按 MsgType 派发）
      → *Ctl.rx → Serializer.deserialize() → App recv()
```

---

## 8. 关键设计决策

| 决策 | 说明 |
|------|------|
| **单锁路由表** | `RwLock<RouterState>` 消除 AB-BA 死锁 |
| **显式结束会话** | `P2PNode` 实现 Drop 关闭 endpoint；主动结束会话仍调用 `P2PChannel::close()` |
| **有界通道** | 所有 mpsc 有容量上限，防恶意 OOM |
| **防腐层** | P2PGateway 隔离 quinn/rustls，DataCtl 隔离 net 类型 |
| **锁外 I/O** | 所有锁范围限状态读写，不在持锁时 await 网络 |
| **薄胶水层** | flutter/rust 仅 newtype 包装，网络逻辑全在 rust/ |
| **双链路冗余** | Active/Standby + connection_watcher 自动故障切换 |
| **单向 TLS** | modcpt_client/server 用 pin 证书，fail-closed |

---

## 9. 辅助项目

- **demo/** (`rust/main.rs`)：10 场景端到端演示（P2P 连接/消息/文件/群组/迁移）；`tests/p2p_integration.rs` 35+ 集成测试。
- **debug-harness/**：多二进制调试工具，含自定义控制面协议（`protocol.rs`），支持 node/peer/server/mesh/grouptest 五种角色，便于在真实网络中验证 serve 层（CA/认证/信令）与群组场景。
- **Knowledge/**：唯一文档目录，包含当前架构、设计、完整复核取证、日期日志和逐文件说明。

---

## 10. 文档导航

- 项目说明：`README.md`
- 当前架构：`Knowledge/ARCHITECTURE.md`
- 协议参考：`Knowledge/protocol-reference.md`
- 逐文件知识库索引：`Knowledge/README.md`
- 构建与账号指南：`Knowledge/guides/`
