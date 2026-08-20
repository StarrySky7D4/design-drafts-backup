# FRB 层梳理与重构方案 — 历史数据下沉记录

> **生成时间**：2026-07-07；**状态**：历史 FRB 设计与代码快照，非执行路线。
> **现行授权**：唯一可执行路线是 [`ROADMAP.md`](../ROADMAP.md) 的 M0-M7。FRB 公开边界、typed-ID、生成流程和兼容规则以[协议注册表](../PROTOCOL_REGISTRY.md)、[协议生命周期](../PROTOCOL_LIFECYCLE.md)及已领取任务为准；本文或 [身份与 FRB 演进](peer-identity-and-frb.md) 均不独立授权实现。
> **关联**：本文保留当时对 ContactRegistry、模块重组、codegen 和子句柄的讨论细节，仅供历史取证。
> **核实基准**：基于当时工作区（`f042c6b`）的 FRB 层逐行核实；文中状态不代表当前源码。

---

## 0. 现状画像：FRB 层在做什么

### 0.1 文件清单（已核实）

| 层 | 文件 | 角色 | 行数 |
|----|------|------|------|
| Rust 手写 API | `flutter/rust/src/api/mod.rs` | RustNode + DataCtl 委托 + 事件结构体 | 272 |
| Rust 手写 API | `flutter/rust/src/api/server_client.rs` | RustServerClient（账号 QUIC 客户端） | 130 |
| Rust 桥 | `flutter/rust/src/log_bridge.rs` | 日志流 | — |
| Rust 生成 | `flutter/rust/src/frb_generated.rs` | FRB 代码生成（**勿手改**） | — |
| Rust lib | `flutter/rust/src/lib.rs` | 模块声明（3 行） | 3 |
| Dart 生成 | `flutter/lib/src/rust/api.dart` | RustNode/TextEvent 等绑定 | 166 |
| Dart 生成 | `flutter/lib/src/rust/api/server_client.dart` | RustServerClient 绑定 | 109 |
| Dart 生成 | `flutter/lib/src/rust/frb_generated.dart` 等 | 生成底座（**勿手改**） | — |
| FRB 配置 | `flutter/flutter_rust_bridge.yaml` | `rust_input: crate::api`，3 行 | 3 |

### 0.2 当前 RustNode 公开 API（`mod.rs:49-252`，18 方法）

| 类别 | 方法 | 标识符维度 | 问题 |
|------|------|-----------|------|
| 生命周期 | `create_node(bind_addr)`、`local_addr()`、`close()` | — | OK |
| 连接 | `connect(addr)`→`sid`、`peers()`→`Vec<sid>`、`disconnect(sid)` | **sessionId** | ❌ 无 userId |
| 文本 | `send_text(to=sid,...)`、`send_text_group(group,...)` | sid/group | 调用方须自己映射 sid→userId |
| 文件 | `send_file(to=sid,...)` | sid | 同上 |
| 群组 | `create_group`/`delete_group`/`add_member`/`remove_member`/`list_groups`/`group_members` | sid/group | OK |
| **事件轮询** | `try_recv_text(ms)`、`try_recv_group_text(ms)`、`try_recv_file(ms)`、`try_recv_group_notify()` | sid | ⚠️ **拉模式，靠 Flutter 300ms 定时器** |

### 0.3 关键诊断：FRB 层的三个结构性问题

1. **身份缺失**：`RustNode` 全程用 `sessionId`（Rust 的 `Uuid` 字符串），**不知 userId**。userId↔sid 映射由 Flutter `_peerCache`/`_addrToId` 维护——这正是上轮去重失败的根因。
2. **拉模式事件**：`try_recv_*` 要求 Flutter 每 300ms 轮询（`frb_native_service.dart:738`）。已有 `logStream` 用 `StreamSink` 推模式，但消息/联系人事件却用拉——不一致。
3. **扁平 API**：`RustNode` 直接暴露 `send_text`/`create_group`/`try_recv_file` 等十几个方法，职责混杂（连接+消息+文件+群组+事件），随功能增长会进一步膨胀。

### 0.4 FRB 代码生成约束（AGENTS.md §注意事项）

> **不要修改 `flutter/lib/src/rust/` 与 `flutter/rust/src/frb_generated.rs`** —— 这些是 `flutter_rust_bridge` 自动生成的。
> **改动 FRB API 后需运行 `flutter_rust_bridge_codegen generate`。**

所以本方案的所有手写改动只在 `flutter/rust/src/api/*.rs`（手写侧），生成侧由 codegen 重生成。

---

## 1. 重构目标

| 目标 | 度量 |
|------|------|
| userId 成为一等公民 | `RustNode` 所有"对端"参数从 sid 改为 userId；ContactRegistry 是唯一身份源 |
| 事件推模式统一 | 删除 `try_recv_*` 拉模式，改 `StreamSink` 推送（与 `logStream` 一致） |
| API 分域 | RustNode 按职责拆分子句柄（contacts / messaging / groups / files） |
| Flutter 零数据 | `frb_native_service.dart` 删除全部 peer/group 缓存，纯转发 |

---

## 2. 目标 Rust API 设计（手写侧 `flutter/rust/src/api/`）

### 2.1 模块重组

```
flutter/rust/src/api/
├── mod.rs              ← 仅保留 create_node / log_stream + 子模块导出
├── node.rs             ← RustNode 顶层句柄（持有 DataCtl + ContactRegistry 句柄）
├── contacts.rs         ← ContactApi：身份/好友/自加防护（委托 ContactRegistry）
├── messaging.rs        ← MessagingApi：文本/语音（按 userId 寻址）
├── groups.rs           ← GroupsApi：群组（既有，改按 userId 成员）
├── files.rs            ← FilesApi：文件传输 + 公告
├── events.rs           ← 事件流（StreamSink 推送，替代 try_recv_*）
└── server_client.rs    ← 既有，不变（账号 QUIC 客户端）
```

> **为什么分文件**：当前 `mod.rs` 272 行已显臃肿。FRB codegen 扫描 `crate::api` 整个模块树，分文件不影响生成，但让手写代码可维护。

### 2.2 RustNode 顶层句柄（`node.rs`）

```rust
pub struct RustNode {
    ctl: Arc<Mutex<DataCtl>>,
    /// ContactRegistry 句柄（DataCtl 内部持有，这里缓存用于 O(1) 查询）。
    contacts: Arc<RwLock<ContactRegistry>>,
    /// 本节点 userId（login 后设置；未登录为 None）。
    local_user: Mutex<Option<String>>,
}

impl RustNode {
    /// 设置本节点身份（completeLogin 后调用）。注册到 ContactRegistry.local_user_id。
    pub async fn set_local_user(&self, user_id: String) -> Result<(), String> { ... }

    /// 子句柄访问器（FRB 支持返回引用类型）。
    pub fn contacts(&self) -> ContactApi { ContactApi { /* ... */ } }
    pub fn messaging(&self) -> MessagingApi { MessagingApi { /* ... */ } }
    pub fn groups(&self) -> GroupsApi { GroupsApi { /* ... */ } }
    pub fn files(&self) -> FilesApi { FilesApi { /* ... */ } }

    pub async fn local_addr(&self) -> Result<String, String> { /* 既有 */ }
    pub fn close(&self) { /* 既有 */ }
}
```

> **FRB 注意**：FRB 2.x 对返回子句柄需用 `#[frb(opaque)]` 或 `RustAutoOpaque`。子句柄持有 `Arc` clone，生命周期独立。

### 2.3 ContactApi（`contacts.rs`）—— 本轮核心

```rust
pub struct ContactApi {
    contacts: Arc<RwLock<ContactRegistry>>,
    ctl: Arc<Mutex<DataCtl>>,
    server: Arc<Mutex<Option<RustServerClient>>>,  // 账号客户端（用于 lookup）
    local_user: Arc<Mutex<Option<String>>>,
}

/// UI 渲染快照（ContactRegistry 派生）。
pub struct ContactSnapshot {
    pub user_id: String,
    pub nickname: String,
    pub online: bool,            // sessions 非空
    pub friendship: Friendship,  // 0=None 1=Outgoing 2=Incoming 3=Accepted 4=Rejected
    pub last_addr: Option<String>,
}

impl ContactApi {
    /// 全量快照（UI 启动 + 事件后刷新）。O(n)，n≤数百，sync 调用。
    #[frb(sync)]
    pub fn list(&self) -> Vec<ContactSnapshot> { ... }

    /// 单个联系人查询。O(1)。
    #[frb(sync)]
    pub fn get(&self, user_id: &str) -> Option<ContactSnapshot> { ... }

    /// 按 userId 拨号（自加安全 + 去重 + 服务器 lookup 内置）。
    /// 替代 Flutter 的 connectByUserId + _connectToPresence 全部逻辑。
    pub async fn connect_by_user_id(&self, user_id: String) -> Result<String, String> {
        // 1. 自加检查（ContactRegistry.CR-1）
        if self.local_user.lock().await.as_deref() == Some(&user_id) {
            return Err("不能添加自己为联系人".into());
        }
        // 2. 已是联系人且在线 → 直接复用（去重）
        if let Some(c) = self.contacts.read().await.get(&user_id) {
            if !c.sessions.is_empty() {
                return Ok(user_id);
            }
        }
        // 3. 服务器 lookup 拿地址
        let presence = self.server.lock().await.as_ref()
            .ok_or("not signed in")?
            .lookup_by_user_id(user_id.clone()).await?
            .ok_or("user not found")?;
        // 4. QUIC 拨号
        let sid = self.ctl.lock().await.connect(presence.addr).await?;
        // 5. 身份绑定（握手 /id 到达后由事件钩子调 bind_identity，这里预填 addr）
        self.contacts.write().await.pre_bind(&sid, &user_id, presence.nickname);
        Ok(user_id)
    }

    /// 接受好友请求。
    pub async fn accept_friend(&self, user_id: String) -> Result<(), String> { ... }
    /// 拒绝好友请求（断开 session）。
    pub async fn reject_friend(&self, user_id: String) -> Result<(), String> { ... }
}
```

**关键**：`list()`/`get()` 用 `#[frb(sync)]` 同步返回——Flutter 在 `ListenableBuilder` 里直接调，无需 await，UI 刷新路径极简。

### 2.4 MessagingApi（`messaging.rs`）—— 按 userId 寻址

```rust
pub struct MessagingApi {
    ctl: Arc<Mutex<DataCtl>>,
    contacts: Arc<RwLock<ContactRegistry>>,
}

impl MessagingApi {
    /// 发送文本（to = userId，不再是 sid）。内部经 ContactRegistry 解析为当前 sid。
    pub async fn send_text(&self, to_user: String, content: String) -> Result<(), String> {
        let sid = self.contacts.read().await
            .primary_session(&to_user)
            .ok_or("contact offline or unknown")?;
        self.ctl.lock().await.send_text(sid.to_string(), content).await
    }

    pub async fn send_text_group(&self, group: String, content: String) -> Result<(), String> { ... }
    /// 语音（伪装文件，既有逻辑下沉）
    pub async fn send_voice(&self, to_user: String, duration_s: i32, data: Vec<u8>) -> Result<(), String> { ... }
}
```

> **变化**：`to` 参数从 `sid` 改为 `userId`。Flutter 不再需要维护 sid↔userId 映射。ContactRegistry 内部处理"该 userId 当前用哪个 session"。

### 2.5 EventsApi（`events.rs`）—— 推模式替代 try_recv_*

```rust
/// 订阅全部 P2P 事件（推模式，替代 try_recv_* 拉模式 + 300ms 定时器）。
pub async fn event_stream(sink: StreamSink<NodeEvent>) {
    // 类似 log_stream：spawn 后台任务，从 DataCtl 的 recv 通道循环读，
    // 转成 NodeEvent 推送。
}

/// 统一事件枚举（FRB 会生成对应 Dart sealed class）。
pub enum NodeEvent {
    /// 文本消息到达（from = userId，已是身份，不再是 sid）。
    TextReceived { from: String, content: String, group: Option<String> },
    /// 文件到达。
    FileReceived { from: String, name: String, data: Vec<u8> },
    /// 联系人状态变化（替代 Flutter 的 PeerStatusEvent）。
    ContactChanged { user_id: String },
    /// 好友请求（携带对方 userId + nickname）。
    FriendRequest { from: String, nickname: String },
    /// 群组创建通知。
    GroupCreated { name: String },
}
```

**关键收益**：
- 删除 `try_recv_text`/`try_recv_group_text`/`try_recv_file`/`try_recv_group_notify` 四个拉方法。
- Flutter 的 `_pollTimer`（300ms）删除，改为 `eventStream().listen()`。
- 事件里的 `from` 直接是 userId（Rust 侧 ContactRegistry 已解析），Flutter 无需再 `_handleText` 字符串解析。

### 2.6 删除的方法清单（与上轮方案对齐）

| 删除 | 替代 |
|------|------|
| `peers()` → `Vec<sid>` | `contacts.list()` → `Vec<ContactSnapshot>` |
| `disconnect(sid)` | `contacts.reject_friend(userId)` |
| `connect(addr)` → `sid` | `contacts.connect_by_user_id(userId)`（公开）；`connect(addr)` 降级为内部 |
| `try_recv_text(ms)` | `eventStream` → `TextReceived` |
| `try_recv_group_text(ms)` | `eventStream` → `TextReceived{group}` |
| `try_recv_file(ms)` | `eventStream` → `FileReceived` |
| `try_recv_group_notify()` | `eventStream` → `GroupCreated` |

---

## 3. 身份绑定钩子：握手事件如何喂 ContactRegistry

这是数据下沉的**关键技术点**——RustNode 收到握手帧后，谁负责调 `bind_identity`？

### 3.1 方案 A（推荐，本轮）：应用层钩子（双写过渡）

**现状**：握手帧 `/id`、`/friend_request` 仍是文本，在 Flutter 解析。过渡期保留这点，但 Flutter 解析后**回调** Rust 记录身份：

```rust
// RustNode 新增：供 Flutter 在解析 /id 后回调，记录身份到 ContactRegistry。
impl RustNode {
    /// Flutter 收到 /id <userId> <nick> 后调此，把 sid↔userId 绑定写入 Registry。
    /// 这是过渡期的"双写"：Flutter 仍解析文本，但身份事实源在 Rust。
    #[frb(sync)]
    pub fn record_identity(&self, session: String, user_id: String, nickname: Option<String>) -> BindOutcome { ... }
}

pub enum BindOutcome {
    Ok,
    Duplicate { drop_session: String },  // 调用方应 disconnect 此 session
    SelfAdd,
}
```

Flutter 侧 `_handleText('/id ...')` 解析后：
```dart
final outcome = node.recordIdentity(session: peerId, userId: uid, nickname: nick);
if (outcome is Duplicate) { await node.disconnect(peerId: outcome.dropSession); }
```

**优点**：本轮不用改握手协议（不碰 Cap'n Proto schema），渐进。**缺点**：握手解析仍在 Flutter（脆弱），由方案 B 收尾。

### 3.2 方案 B（PR 3）：握手帧进 schema，Rust 解析

握手帧从文本 `/id <uid> <nick>` 改为结构化 `MsgType::Identity` 帧（Cap'n Proto）。Rust 的 DataCtl 收到后直接调 `bind_identity`，Flutter 完全不参与握手解析。详见 `CONTACT_REGISTRY_DESIGN` §4.2。

---

## 4. Dart 侧变化（生成 + 手写）

### 4.1 生成侧（`flutter/lib/src/rust/`，codegen 重生成）

codegen 运行后自动产生：
- `RustNode.contacts()` → `ContactApi`
- `ContactApi.list()` → `List<ContactSnapshot>`（sync）
- `ContactApi.connectByUserId()` → `Future<String>`
- `eventStream()` → `Stream<NodeEvent>`
- `NodeEvent` sealed class（Dart 3 exhaustive match）

**手改禁忌**：`frb_generated.*.dart`、`api.dart` 的内容不要手改。

### 4.2 手写侧：`frb_native_service.dart` 大幅瘦身

| 删除 | 行数 | 原因 |
|------|------|------|
| `_peerCache`/`_addrToId`/`_peerByUserId` | ~40 | 数据下沉 ContactRegistry |
| `_groupNamesCache`/`_groupMembersCache` | ~20 | 改读 `groups.list()` |
| `_handleText` 的 `/id`/`/friend_request`/`/friend_accept`/`/friend_reject` 分支 | ~60 | 改用 `record_identity` 或下沉 Rust |
| `_pollTimer` + `_startPolling` + `_syncPeers` | ~90 | 改 `eventStream().listen()` |
| `connect`/`_connectToPresence`/`connectByUserId` | ~80 | 改调 `contacts.connectByUserId()` |

`frb_native_service.dart` 从 ~1054 行缩到 ~400 行（仅保留文件分享 gossip、语音标记、签名密钥对等尚需在 Flutter 的逻辑）。

### 4.3 `app_state.dart` 变薄

```dart
// 删除：_peers, _groups, _peerCache 同步逻辑
// 保留：UI 临时态（activeConversation, contactFilter, busy, lastError）

List<ContactSnapshot> get contacts => _svc.node.contacts().list();  // sync
Stream<NodeEvent> get events => _eventStream;  // 订阅一次
```

`AppState` 从 1147 行缩到 ~400 行。

---

## 5. 渐进迁移路线（5 个 PR，已全部完成）

> 与上一轮 `CONTACT_REGISTRY_DESIGN` 的 5 PR 对齐，本节聚焦**每个 PR 的 FRB 改动**。

### PR 1 — ContactRegistry 内存结构（纯 Rust，FRB 不动）

**状态**：✅ 已完成（对应阶段 B1）

**FRB 改动**：无。仅在 `rust/core/src/data_ctl/contacts.rs` 新增结构 + 单测。
**验证**：`cargo test --lib -p modcpt_core`。

### PR 2 — FRB 加 `record_identity` 双写钩子（过渡）

**状态**：✅ 已完成（对应阶段 B2b，后由 B3 收敛为单写）

**FRB 改动**（`flutter/rust/src/api/mod.rs`，后已删除——B3 自动绑定后 `record_identity` 无人调用，C1 清理时移除）：
```rust
impl RustNode {
    pub fn record_identity(&self, session: String, user_id: String, nickname: Option<String>) -> BindOutcome { ... }
}
```
pub enum BindOutcome { Ok, Duplicate { drop_session: String }, SelfAdd }

**Flutter 改动** 已由 B3/阶段 C 替代（`_updatePeerIdentity` 删除，身份帧自动绑定）。
**codegen**：`flutter_rust_bridge_codegen generate`。
**验证**：`flutter analyze` + B3 e2e identity auto-bind。

### PR 3 — 握手帧进 Cap'n Proto，Rust 解析（协议层）

**状态**：✅ 已完成（对应阶段 B3）

**FRB 改动**：`record_identity` 标记 deprecated（仍可用作回退）。新增 `MsgType::Identity` 解析在 DataCtl，自动调 `bind_identity`。
**Flutter 改动**：`_handleText` 删除 `/id` `/friend_request` 解析分支。
**验证**：双机回归握手（`test_e2e_identity_autobind` 通过）。

### PR 4 — FRB API 分域 + userId 寻址（本方案主体）

**状态**：✅ 已完成（对应阶段 C1 + 旧方法清理）

**FRB 改动**（`flutter/rust/src/api/`）：
- 拆 `mod.rs` → `contacts.rs` + `messaging.rs` + `groups.rs` + `files.rs`（未单拆 `node.rs`，保留在 `mod.rs`）
- `RustNode.contacts()/messaging()/groups()/files()` 子句柄
- `ContactApi.list()/get_()/connect_by_user_id()/accept_friend()/reject_friend()`（sync list/get）
- `MessagingApi.send_text(to_user)` 等（参数 sid→userId）
- 已删除 `send_text(sid)`/`send_file(sid)`/`record_identity`/群组方法（待保留：`connect`/`peers`/`disconnect`/`try_recv_*`）
- 新增 `send_text_raw`/`send_file_raw`——协议命令保留按 sessionId 发送的能力

**Flutter 改动**：`frb_native_service.dart` 20+ 调用点升级到子句柄 API（`_peerCache` 暂留，待 C3 改读 `contacts.list()`）。
**codegen**：`flutter_rust_bridge_codegen generate`（已执行）。
**验证**：`flutter analyze` 0 err/warn + `flutter build windows --debug` 61.8s 成功。

### PR 5 — 事件流推模式（删 try_recv_*）

**状态**：✅ 已完成（对应阶段 C2）

**FRB 改动**：
- 新增 `event_stream(sink: StreamSink<NodeEvent>)` 在 RustNode 方法
- 新增 `NodeEvent` struct + 构造器
- 删除 `try_recv_text`/`try_recv_group_text`/`try_recv_file`/`try_recv_group_notify` + `try_recv_identity`

**Flutter 改动**：
- `frb_native_service.dart` 删 `_pollTimer`/`_startPolling`/`_syncPeers`（~200 行）
- `startNode` 内 `node.eventStream().listen(_onNodeEvent)`
- `_onNodeEvent` 按 `event.kind` 分发（替代 `_handleText` + 轮询排空）
- 新增 `_startEventStream`、删全部 `tryRecv*` 调用

**codegen**：`flutter_rust_bridge_codegen generate`（已执行；注意 `NodeEvent` 用 struct 而非 enum 以避免 `freezed` 依赖）。
**验证**：`flutter analyze` 0 error + `cargo build` 通过。

---

## 6. FRB 代码生成操作清单

### 6.1 触发条件

| 改动 | 需 codegen？ |
|------|------|
| 改 `flutter/rust/src/api/*.rs` 的 `pub` 项 | ✅ |
| 改 `flutter/rust/src/log_bridge.rs` | ✅（若改 pub 项） |
| 改 `rust/core/`（DataCtl/ContactRegistry） | ❌（FRB 不直接扫描 core） |
| 改 `flutter/rust/Cargo.toml` 依赖 | ❌（除非引入新 type 跨 FFI） |
| 改 `flutter/lib/src/rust/*`（生成侧） | 🚫 禁止手改 |

### 6.2 命令

```bash
cd flutter
flutter_rust_bridge_codegen generate
# 或（若 FRB 版本要求）：
flutter pub run flutter_rust_bridge_codegen generate
```

**前置**：Rust 侧必须先 `cargo build` 通过（codegen 会编译 `modcpt_frb`）。

### 6.3 codegen 后必做

```bash
cd flutter
flutter analyze   # 检查生成代码与手写代码一致性
flutter test      # widget_test 用 Fake，应不受影响
```

---

## 7. 风险与对策

| 风险 | 对策 |
|------|------|
| codegen 版本漂移（`flutter_rust_bridge = "=2.12.0"`） | 锁定等号版本已做；升级需全量回归 |
| 子句柄返回（`RustNode.contacts()`）FRB 不支持 | 用 `#[frb(opaque)]` 标注子句柄；或退化为顶层 `node_list_contacts(handle)` 函数式 API |
| `#[frb(sync)]` 在流上下文限制 | sync 方法不能在异步持有锁处调用；ContactRegistry 用 `try_read`（非阻塞） |
| 事件流 `StreamSink` 背压 | 参照 `log_stream` 已验证模式；事件量大时 `try_send` 丢弃并 warn |
| PR 4 拆分文件后 codegen 扫描失败 | `rust_input: crate::api` 已覆盖整个模块树；分文件不改配置 |
| 双写过渡期（PR 2-3）身份不一致 | record_identity 作为唯一写入点；Flutter 缓存视为只读视图 |

---

## 8. 验证矩阵

| # | 场景 | PR | 预期 |
|---|------|----|------|
| F1 | codegen 后 `flutter analyze` 0 error | 2/4/5 | 通过 |
| F2 | `contacts.list()` 返回快照含 userId | 4 | Flutter 直接渲染 |
| F3 | `connect_by_user_id(self)` 返回错误 | 4 | "不能添加自己" |
| F4 | A 加 B 后 B 加 A 无重复 | 4 | `contacts.list()` 唯一 B |
| F5 | 事件推模式延迟 < 拉模式 | 5 | 消息即时到 |
| F6 | `record_identity` 双写期一致 | 2 | Registry 与 Flutter 缓存一致 |

---

## 9. 不变量（合并前 grep + 测试）

| ID | 不变量 |
|----|--------|
| FRB-1 | `flutter/rust/src/frb_generated.rs` 与 `flutter/lib/src/rust/frb_generated.*.dart` 无手改（`git diff` 仅 codegen 产物） |
| FRB-2 | `RustNode` 公开方法中无 `sessionId`（`String` 形式的 Uuid）作为参数，全部用 `userId` |
| FRB-3 | `try_recv_*` 四方法全删，grep 为 0 |
| FRB-4 | `frb_native_service.dart` 无 `_peerCache`/`_addrToId`/`_pollTimer` |
| FRB-5 | ContactRegistry 单测覆盖 CR-1~4（自加/唯一/索引一致/去重闭环） |

---

## 10. 决策点

| 决策 | 选项 | 建议 |
|------|------|------|
| 子句柄 vs 扁平 API | (a) `node.contacts().list()`；(b) `node_list_contacts(handle)` 函数式 | **(a)** 面向对象更清晰；FRB 2.x 支持 |
| sync list/get | (a) `#[frb(sync)]` 同步；(b) Future | **(a)** UI 渲染路径无 await；ContactRegistry 用 try_read |
| 过渡期长度 | PR 2 双写保留多久 | 建议一个发布周期（PR 3 协议帧化后即移除 record_identity） |
| 事件流粒度 | (a) 单一 `eventStream`；(b) 分 `messageStream`/`contactStream` | **(a)** 单流简单；Flutter 按 enum 分发 |
| NodeEvent from 用 userId 还是 sid | userId | Rust 侧已解析身份；Flutter 无需再查 |

---

## 附录：FRB 文件改动一览

| 文件 | PR | 改动类型 |
|------|----|------|
| `flutter/rust/src/api/mod.rs` | 2/4/5 | 拆分 + 加方法 + 删 try_recv |
| `flutter/rust/src/api/node.rs` | 4 | 新增（RustNode 顶层） |
| `flutter/rust/src/api/contacts.rs` | 4 | 新增（ContactApi） |
| `flutter/rust/src/api/messaging.rs` | 4 | 新增（MessagingApi） |
| `flutter/rust/src/api/groups.rs` | 4 | 新增（GroupsApi，迁移既有群组方法） |
| `flutter/rust/src/api/files.rs` | 4 | 新增（FilesApi） |
| `flutter/rust/src/api/events.rs` | 5 | 新增（event_stream + NodeEvent） |
| `flutter/rust/src/api/server_client.rs` | — | 不变 |
| `flutter/lib/src/rust/*` | 2/4/5 | codegen 重生成（禁止手改） |
| `flutter/lib/services/frb_native_service.dart` | 2/4/5 | 大幅瘦身 |
| `flutter/lib/services/app_state.dart` | 4/5 | 删数据字段，改读快照 |
| `Knowledge/flutter/frb/*.md` | 各 PR | 同步 |

---

> **历史摘要**：本文记录当时提出的 userId 参数、事件推模式、API 分域和五个 PR 的讨论。它不是当前实施清单，也不授权依其顺序改动代码。任何新的 FRB 工作必须先映射到 [`ROADMAP.md`](../ROADMAP.md) 的阶段并遵循协议注册表、生命周期及当前 AGENTS 生成约束；届时仍须由官方 codegen 更新生成侧并运行适用验证。

## 2026-08-07 Flutter/FRB 边界复核

- `NativeService.bindServerClient` 改为可等待的登录提交点；Rust local identity 与联系人查询 client 都安装完成后才允许持久化登录状态。
- 产品 Dart 层删除裸地址 `connect(address)`，连接入口固定为 `connectByUserId`。
- 群文件请求以公告 `senderUser` 作为 UserId 调用 `MessagingApi.sendText(toUser:)`，不再误入 raw TransportSessionId API。
- 停止节点同时销毁 Dart 账号 client、session、UserId 和 cert pin，避免失败回滚/换号复用陈旧授权状态。
- 本次未改变 Rust FRB 公开形状，也未手改 generated 文件。`NodeEvent.kind` 的字符串兼容面若要枚举化，必须另立版本化变更并同步 capability、fingerprint、release contract 和官方 codegen 产物。
