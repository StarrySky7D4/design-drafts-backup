# 基于 UserID 的去重、中心化组件与 FRB 重写 — 历史整合记录

> **生成时间**：2026-07-07；**状态**：历史设计与实施取证，非执行路线。
> **现行授权**：唯一可执行路线是 [`ROADMAP.md`](../ROADMAP.md) 的 M0-M7；身份、设备和授权边界以[身份与寻址 v2](identity-and-addressing-v2.md)为准。本文不得独立授权代码、schema、FRB API、协议或阶段实施。
> **v2 取代说明（2026-07-26）**：本文的单 user/session 归并、IdentityFrame 身份绑定与公开 raw-session FRB API 是 v1 历史方案；后续实现不得依赖这些路径建立 v2 授权。
> **历史来源**：`rewrite/auth-flow` 分支整合此前讨论：
> - 第一轮：peer 去重修复（`PEER_DEDUP_FIX.md`（源提交中无对应文件，仅保留历史引用名））—— 临时层修补
> - 第二轮：去重加速（布隆过滤器 / HashMap / 遍历 / 归并查询入口）的取舍
> - 第三轮：Rust 中心化组件（`ContactRegistry`）—— 身份下沉
> - 第四轮：FRB 层梳理与重写 —— userId 一等公民 + 推模式
>
> 本文保留此前分散讨论、代码快照和实施记录作为追溯证据；配套的 FRB、Mock 和登录材料不以摘要替代。文中“阶段”“已完成”“待实施”和时间线均描述当时的提案或快照，不改变当前看板状态。

---

## 目录

1. [问题陈述与根因](#1-问题陈述与根因)
2. [技术决策记录（含被排除方案）](#2-技术决策记录含被排除方案)
3. [三阶段演进路线](#3-三阶段演进路线)
4. [阶段 A：临时去重修复](#4-阶段-a临时去重修复)
5. [阶段 B：ContactRegistry 中心化组件](#5-阶段-bcontactregistry-中心化组件)
6. [阶段 C：FRB 层重写](#6-阶段-cfrb-层重写)
7. [未来路径](#7-未来路径)
8. [不变量总表](#8-不变量总表)
9. [验证总矩阵](#9-验证总矩阵)
10. [风险与回滚](#10-风险与回滚)
11. [附录 A：历史文档映射](#附录-a历史文档映射)
12. [附录 B：文件改动总览](#附录-b文件改动总览)

---

## 1. 问题陈述与根因

### 1.1 两个用户报告的 Bug

| # | Bug | 复现 |
|---|-----|------|
| **P1** | 可以添加自己为 peer | 搜索框输入自己的 UserID / 昵称 → 自己出现在联系人列表 |
| **P2** | 双向加好友导致两端重复对象 | A 加 B 后，B 再加 A → 同一 userId 在联系人列表出现两个条目 |

### 1.2 根因（逐行核实）

**P1 根因：两条防线同时失效**

1. **应用层零拦截**：`connectByUserId` / `connect` / `lookupByNickname` 不检查 `target == self`（`frb_native_service.dart:252/154/278`）。
2. **Rust 层 LAN IP 漏判**：`ip_is_same_host`（`p2p.rs:986-994`）只匹配 loopback/wildcard。节点绑定 `0.0.0.0:port`，但 `publishPresence` 注册的是 `detect_lan_ip()` 得到的 LAN IP（如 `192.168.5.15`）。拨号 LAN IP 时 `ip_is_same_host(0.0.0.0, 192.168.5.15)` → wildcard 分支要求对端是 loopback → **返回 false，自连检查被绕过**。（直连 `127.0.0.1:port` 反而被 Rust 拦住。）

**P2 核心根因：缺乏 userId 维度的 peer 身份**

- peer 索引只有 `sessionId` 和 `addr` 两个维度（`_peerCache: Map<sid, Peer>`、`_addrToId: Map<addr, sid>`），**无 `userId → peer` 反向查找**。
- A 加 B 后 B 再加 A：若 A 注册的 LAN IP 与 QUIC 连接在 B 侧观察到的来源地址不同（NAT / 端口映射 / 不同网卡），`_addrToId` 复用失败 → 新 session → 同一 userId 出现两个 peer 条目。
- `/friend_request` 收到时不查"该 userId 是否已存在"，直接弹窗，叠加语义混乱。

---

## 2. 技术决策记录（含被排除方案）

本节记录讨论中评估过的方案及最终选择，作为未来评审依据。

### 2.1 去重数据结构：遍历 vs HashMap vs 布隆过滤器

| 方案 | 评估 | 结论 |
|------|------|------|
| **布隆过滤器** | 需精确判断（假阳性=误拒合法好友=功能错误）；需删除（标准布隆不支持，升级 Counting Bloom 复杂度大增）；命中后仍需二次确认；规模太小（n≤数百）无收益，前置哈希开销可能比直接查还慢 | **❌ 排除**（4 硬伤） |
| **HashMap 反向索引**（`_userIdToPeerId`） | O(1) 查询；但需在 6+ 个 `_peerCache` 写入点同步维护索引，漂移风险高；当前规模（n≤数百，低频调用路径）O(1) vs O(n) 无用户可感知差异 | **⏸ 阶段 A 过度设计；阶段 B 自然实现**（下沉 Rust 后 ContactRegistry 本就是 HashMap，但由中心化写入消除漂移） |
| **O(n) 遍历**（`_peerByUserId`） | O(n) 查询，n≤数百 → 微秒级；零漂移风险（无派生状态）；Mock 已用此模式（`mock_native_service.dart:122`）；调用频率低（加好友/握手，非热路径） | **✅ 阶段 A 采用** |

**关键洞察**：真正重要的不是 O(n) vs O(1)，而是**检查逻辑是否归并到单一入口**。无论内部实现如何，调用方应只看到一个 `_checkPeer()` 决策函数。这样内部从遍历升级到 HashMap 是单点替换，零调用方影响。

**升级触发条件**：当 peer 规模 n > ~5000（未来群组/P2P 网络规模）时，O(n) 才可能成为瓶颈，届时升级 HashMap。阶段 B 下沉 Rust 后，ContactRegistry 已是 HashMap 实现，此问题自动消解。

### 2.2 归并策略：硬归并 vs 软归并

| 策略 | 做法 | 结论 |
|------|------|------|
| **硬归并** | 检测到同 userId 多 session → 断开多余 session | ⚠️ 双方同时检测可能"双方都断导致全断" |
| **软归并** | 检测到重复 → 仅从本地列表移除重复条目，不断 session，靠 QUIC idle timeout（默认 30s）自然回收 | **✅ 阶段 A/B 起步**，零风险；稳定后可加硬归并选项 |

### 2.3 数据归属：Flutter 持有 vs Rust 中心化

| 方案 | 评估 | 结论 |
|------|------|------|
| **Flutter 持有 peer 数据**（现状） | userId↔sid 映射在 Dart 维护，靠字符串解析握手协议；脆弱、难测试、难去重 | ❌ 现状已证明不可持续（P1/P2 正是此架构的后果） |
| **Rust 中心化**（ContactRegistry） | userId 唯一事实源在 Rust；Flutter 改为查询快照；协议帧化、可测试 | **✅ 阶段 B/C 采用** |

### 2.4 FRB API 形态：扁平 vs 分域

| 方案 | 评估 | 结论 |
|------|------|------|
| **扁平**（现状 `RustNode` 18 方法） | 职责混杂，随功能增长膨胀 | ❌ 已 272 行，不可持续 |
| **分域子句柄**（`node.contacts().list()`） | 职责清晰；FRB 2.x 支持 `#[frb(opaque)]` 子句柄 | **✅ 阶段 C 采用** |

### 2.5 事件模式：拉 vs 推

| 方案 | 评估 | 结论 |
|------|------|------|
| **拉模式**（`try_recv_*` + 300ms 定时器） | Flutter 轮询开销；消息延迟 ≤300ms；与已有 `logStream` 推模式不一致 | ❌ 阶段 C 删除 |
| **推模式**（`StreamSink<NodeEvent>`） | 与 `logStream` 一致；即时推送；Flutter 无定时器 | **✅ 阶段 C 采用** |

---

## 3. 历史三阶段演进路线

```
阶段 A（临时去重修复）          阶段 B（中心化组件）           阶段 C（FRB 重写）
┌──────────────────┐      ┌──────────────────────┐      ┌──────────────────────┐
│ Flutter 应用层    │      │ Rust ContactRegistry │      │ FRB API 重塑         │
│ • _checkPeer 归并 │ ───▶ │ • userId 主键注册表   │ ───▶ │ • userId 一等公民     │
│ • _peerByUserId   │      │ • bind_identity 唯一源│      │ • 子句柄分域          │
│ • 自加拦截        │      │ • 挂 P2PNodeInner    │      │ • 推模式事件          │
│ • Rust LAN IP 兜底│      │ • 握手双写过渡        │      │ • Flutter 零数据      │
└──────────────────┘      └──────────────────────┘      └──────────────────────┘
   解决 P1/P2（临时）         身份下沉（根治重复）           数据彻底下沉 Rust
   ~2 PR                    ~3 PR                        ~3 PR
```

| 阶段 | 目标 | 解决 | 风险 |
|------|------|------|------|
| **A** | 最快止血，零架构改动 | P1/P2 症状 | 不根治（仍依赖 Flutter 解析握手） |
| **B** | 身份事实源移到 Rust | P2 根治 + 为 C 铺路 | 握手协议改动（双写过渡降险） |
| **C** | Flutter 零数据，FRB 面向 userId | 全部 + 可维护性 | 大改动，需充分回归 |

**关键原则**：每阶段独立可合并、可回滚；阶段 A 不阻塞 B/C，B 不阻塞 C，但 B 的 ContactRegistry 是 C 的 FRB 重写的数据基础。

---

## 4. 阶段 A：临时去重修复

> **目标**：最快止血 P1/P2，不改架构。仍保留 Flutter 持有 peer 数据的现状。

### 4.1 统一查询入口（核心工程改进）

把分散的自加/去重/自连检查归并成单一决策函数（无论内部用遍历还是 HashMap，调用方只看这一个入口）：

```dart
// frb_native_service.dart
sealed class PeerCheckOutcome {
  const PeerCheckOutcome();
  const factory PeerCheckOutcome.ok() = _Ok;
  const factory PeerCheckOutcome.alreadyPeer(String peerId) = _Already;
  const factory PeerCheckOutcome.blocked(String reason) = _Blocked;
  bool get isOk => this is _Ok;
  String? get error => this is _Blocked b ? b.reason : null;
}

/// Unified peer-access gate. Consolidates self-add / duplicate / self-connect
/// checks so call sites can't diverge.
PeerCheckOutcome _checkPeer({String? userId, String? addr}) {
  // (1) Self-add (identity) — O(1) constant compare.
  if (userId != null && userId.isNotEmpty && userId == _localUserId) {
    return const PeerCheckOutcome.blocked('不能添加自己为联系人');
  }
  // (2) Already a peer (identity) — O(n) via _peerByUserId (upgrade to HashMap later).
  if (userId != null && userId.isNotEmpty) {
    final existing = _peerByUserId(userId);
    if (existing != null && existing.online) {
      return PeerCheckOutcome.alreadyPeer(existing.id);
    }
  }
  // (3) Self-connect (address) — O(1) over a tiny fixed set.
  if (addr != null && _isOwnAddress(addr)) {
    return const PeerCheckOutcome.blocked('不能连接到本机地址');
  }
  return const PeerCheckOutcome.ok();
}

/// O(n) lookup over the contact list. Fine for tens-to-hundreds scale;
/// no reverse index to keep in sync. Swap to _userIdIndex HashMap when n>5000.
Peer? _peerByUserId(String? userId) {
  if (userId == null || userId.isEmpty) return null;
  for (final p in _peerCache.values) {
    if (p.userId == userId) return p;
  }
  return null;
}
```

所有调用点（`connectByUserId` / `connect` / `_connectToPresence` / `_handleText /friend_request`）统一调 `_checkPeer`：

```dart
// connectByUserId
final check = _checkPeer(userId: id);
if (check.error != null) return BridgeResult.fail(check.error!);

// _connectToPresence
final check = _checkPeer(userId: presence.userId, addr: presence.addr);
if (check.error != null) return BridgeResult.fail(check.error!);
if (check is _Already) { /* refresh nickname, reuse */ return BridgeResult.ok((check).peerId); }
```

### 4.2 软归并（解决 P2 重复条目）

`_updatePeerIdentity`（握手学习身份的唯一入口）检测到同 userId 已存在于另一 session → **软归并**（仅移除本地重复条目，不断 session）：

```dart
void _updatePeerIdentity(String peerId, String userId, String nickname) {
  final dup = _peerByUserId(userId);
  if (dup != null && dup.id != peerId) {
    debugPrint('[dedup] duplicate session for $userId: keep ${dup.id}, drop $peerId');
    _peerCache.remove(peerId);
    _addrToId.removeWhere((_, id) => id == peerId);
    if (nickname.isNotEmpty) dup.alias = nickname;
    _eventCtl.add(PeerStatusEvent(peerId: dup.id, online: true));
    return;  // 软归并：不断 session，靠 QUIC idle timeout 自然回收
  }
  // ... 正常更新逻辑
}
```

### 4.3 lookupByNickname 过滤自己

```dart
final filtered = _localUserId.isEmpty
    ? nodes
    : nodes.where((n) => n.userId != _localUserId).toList();
```

### 4.4 Rust 层 LAN IP 兜底（P1 底层防线）

扩展 `is_self_target`（`p2p.rs`）注入运行时本机网卡 IP，使拨号 LAN IP 也被识别为自连：

```rust
pub fn is_self_target(&self, remote: SocketAddr) -> bool {
    let mut locals = Self::local_addrs_from(&self.inner);
    for ip in self.known_local_ips() {   // 新增：枚举本机网卡 IP（用 if-addrs crate 或 UDP connect 技巧）
        locals.push(SocketAddr::new(ip, 0));
    }
    addr_is_self(&locals, remote)        // 既有：按端口+IP 比对
}
```

### 4.5 阶段 A 不变量

| ID | 不变量 |
|----|--------|
| DEDUP-1 | `connectByUserId` 含自加拦截 |
| DEDUP-2 | `lookupByNickname` 过滤 `_localUserId` |
| DEDUP-3 | `_checkPeer` 统一入口存在，无分散检查 |
| DEDUP-4 | `_peerByUserId` 遍历辅助存在，无新增索引字段（阶段 A） |
| DEDUP-5 | Rust `is_self_target` 对本机 LAN IP:port 返回 true |

---

## 5. 阶段 B：ContactRegistry 中心化组件

> **目标**：userId 身份事实源移到 Rust，根治重复，为阶段 C FRB 重写铺路。

### 5.1 核心数据结构（`rust/core/src/data_ctl/contacts.rs`，新增）

```rust
pub struct Contact {
    pub user_id: String,           // 主键，服务器 UserID
    pub nickname: String,
    pub sessions: HashSet<Uuid>,   // 0=离线已知，>1=去重待归并
    pub last_addr: Option<String>,
    pub friendship: Friendship,    // None/OutgoingPending/IncomingPending/Accepted/Rejected
    pub public_key: Option<Vec<u8>>,
}

pub struct ContactRegistry {
    contacts: HashMap<String, Contact>,      // userId → contact（唯一事实源）
    session_to_user: HashMap<Uuid, String>,  // 派生索引
    addr_to_user: HashMap<String, String>,   // 派生索引
    local_user_id: Option<String>,           // 本节点身份（自加检测）
}
```

挂到 `P2PNodeInner`：`contacts: Arc<RwLock<ContactRegistry>>`。`RouterState`（sessionId 路由）不变，ContactRegistry 是其上层的身份层。

### 5.2 写入唯一入口（消除漂移）

```rust
impl ContactRegistry {
    pub async fn bind_identity(&self, session: Uuid, user_id: String, nick: Option<String>) -> BindResult { ... }
    pub async fn unbind_session(&self, session: Uuid) { ... }
    pub async fn set_friendship(&self, user_id: &str, f: Friendship) { ... }
}

pub enum BindResult {
    New(String),
    Rebound(String),
    Duplicate { keep_session: Uuid, drop_session: Uuid },
    SelfAdd,
}
```

`bind_identity` 是去重的**唯一决策点**：自加返回 `SelfAdd`；同 userId 新 session 软归并（加入 `sessions` 集合，按 userId 渲染则列表无重复）。

### 5.3 握手身份绑定（双写过渡）

阶段 B 初期握手帧 `/id`、`/friend_request` 仍是文本，Flutter 解析。新增 FRB 钩子让 Flutter 解析后回调 Rust 记录身份（双写过渡）：

```rust
// flutter/rust/src/api/mod.rs
impl RustNode {
    #[frb(sync)]
    pub fn record_identity(&self, session: String, user_id: String, nickname: Option<String>) -> BindOutcome { ... }
}
```

阶段 B 后期（PR 3）：握手帧进 Cap'n Proto schema（`MsgType::Identity`），Rust 直接解析并调 `bind_identity`，Flutter 不再参与握手解析。

### 5.4 阶段 B 不变量

| ID | 不变量 |
|----|--------|
| CR-1 | `local_user_id` 设置后，`bind_identity(local_user)` 返回 `SelfAdd` |
| CR-2 | 任意时刻同一 userId 在 `contacts` 中至多 1 个条目 |
| CR-3 | `session_to_user` 与 `contacts[*].sessions` 严格双向一致 |
| CR-4 | `Duplicate` 场景调用方必须处理（去重闭环） |

### 5.5 阶段 B PR 分解

| PR | 内容 | 状态 |
|----|------|------|
| B1 | 新建 `ContactRegistry` + 单测（纯 Rust，不接 P2P） | ✅ 已完成（16 单测覆盖 CR-1~4） |
| B2a | `ContactRegistry` 接入 `DataCtl` 门面（`record_identity`/`set_local_user_id`/访问器 + 生命周期自动解绑 + `SessionId::FromStr`） | ✅ 已完成（3 单测；注：挂 `DataCtl` 而非文档原写的 `P2PNodeInner`，因 FRB 链路只达 `DataCtl`） |
| B2b | FRB `record_identity`/`set_local_user_id` 钩子 + codegen 2.12.0 重生成 Dart 绑定 + Flutter `_updatePeerIdentity`/`startNode`/`bindServerClient` 双写回调 | ✅ 已完成（`flutter analyze` 0 error/warning） |
| B3 | 握手帧进 Cap'n Proto（`MsgType::Identity` + `IdentityFrame`），Rust 自动解析并 `bind_identity`；Flutter 改收发类型化 `IdentityFrame`，移除 `/id`/`/friend_request` 文本解析；双写收敛为单写 | ✅ 已完成（Rust e2e + flutter analyze 0 err/warn） |

---

## 6. 阶段 C：FRB 层重写

> **目标**：Flutter 零数据，FRB API 面向 userId，事件推模式。

### 6.1 模块重组（`flutter/rust/src/api/`，已实施 C1）

```
api/
├── mod.rs          ← RustNode 顶层（create_node + log_stream + 子句柄访问器 + 保留旧方法 + 事件结构体）
├── contacts.rs     ✓ ContactApi：sync list/get_ + connect_by_user_id + 好友管理
├── messaging.rs    ✓ MessagingApi：send_text(to_user) + send_text_raw(to_session)
├── groups.rs       ✓ GroupsApi：群组 CRUD（既有方法迁移）
├── files.rs        ✓ FilesApi：send_file(to_user) + send_file_raw(to_session)
├── events.rs       ← ⏳ 待 C2（event_stream + NodeEvent）
└── server_client.rs ← 既存不变
```

*注：文档原计划 `node.rs` 单拆，实现在 `mod.rs` 中保留（子句柄访问器放在这里更简洁）。

### 6.2 userId 一等公民

所有"对端"参数从 `sessionId` 改为 `userId`：

```rust
impl MessagingApi {
    pub async fn send_text(&self, to_user: String, content: String) -> Result<(), String> {
        let sid = self.contacts.read().await.primary_session(&to_user)
            .ok_or("contact offline")?;
        self.ctl.lock().await.send_text(sid.to_string(), content).await
    }
}
```

Flutter 不再维护 sid↔userId 映射。

### 6.3 sync 快照查询

```rust
impl ContactApi {
    #[frb(sync)]
    pub fn list(&self) -> Vec<ContactSnapshot> { ... }   // O(n)，n≤数百，UI 直接调
    #[frb(sync)]
    pub fn get(&self, user_id: &str) -> Option<ContactSnapshot> { ... }
}
```

Flutter：`ListenableBuilder` 内 `_svc.node.contacts().list()` 同步返回，无需 await。

### 6.4 推模式事件流

```rust
pub async fn event_stream(sink: StreamSink<NodeEvent>) { /* 参照 log_stream */ }

pub enum NodeEvent {
    TextReceived { from: String, content: String, group: Option<String> },  // from=userId
    FileReceived { from: String, name: String, data: Vec<u8> },
    ContactChanged { user_id: String },
    FriendRequest { from: String, nickname: String },
    GroupCreated { name: String },
}
```

删除 `try_recv_text`/`try_recv_group_text`/`try_recv_file`/`try_recv_group_notify`。Flutter 删 `_pollTimer`，改 `node.eventStream().listen()`。

### 6.5 阶段 C 不变量（最终状态 / 当前进展）

| ID | 不变量 | 当前状态 |
|----|--------|---------|
| FRB-1 | `frb_generated.rs` 与 `frb_generated.*.dart` 无手改 | ✅ 始终由 codegen 生成 |
| FRB-2 | `RustNode` 公开方法无 sessionId 参数，全部 userId | ⚠️ 部分完成：`send_text(to_user)`/`send_file(to_user)` 已 userId；`send_text_raw`/`send_file_raw` 保留 sessionId（协议命令）；`connect`/`peers`/`disconnect` 待清理 |
| FRB-3 | `try_recv_*` 四方法全删，grep 为 0 | ✅ C2 已完成（推模式 `event_stream` + 后台 recv 循环发布到广播通道） |
| FRB-4 | `frb_native_service.dart` 无 `_peerCache`/`_addrToId`/`_pollTimer` | ✅ 已完成：Flutter 不再维护 peer/group 派生缓存；事件只作失效信号，视图读取 Rust 快照 |
| FRB-5 | ContactRegistry 单测覆盖 CR-1~4 | ✅ 16 单测覆盖 |

### 6.6 阶段 C PR 分解

| PR | 内容 | 状态 |
|----|------|------|
| C1 | FRB API 分域 + userId 寻址（`send_text(to_user)`）+ sync 快照（`contacts.list()/get_()`）+ 旧方法清理（`send_text(sid)`/`send_file(sid)`/群组方法已删，`record_identity` 已删） | ✅ 已完成（`flutter analyze` 0 err/warn + `flutter build windows --debug` 成功） |
| C2 | `event_stream` 推模式 + `NodeEvent` 结构体 + `CoreEvent` 广播通道；删 `try_recv_*` + Flutter 删 `_pollTimer`/`_syncPeers`/`_startPolling` | ✅ 已完成（`flutter analyze` 0 error） |
| C3 | Flutter 群组快照驱动（`GroupsApi::snapshots()` sync 直读，删 `_groupNamesCache`/`_groupMembersCache`/`_refreshGroupCache`）+ 事件驱动失效信号（`GroupUpdated`）；群组成员变更通知所有成员（修复延迟） | ✅ 已完成（群组 UI 零派生缓存 + `groups.snapshots()` + `notify_group_members`） |
| C4 | Flutter 删 `_peerCache`/`_addrToId`，改读 `contacts.list()`；删除剩余旧方法（`connect`/`peers`/`disconnect`）；移除阶段 A Flutter 去重兜底 | ⚠️ 产品 Dart 路径已完成：缓存和裸地址 `connect` 均已删除，文件请求也只按 UserId；Rust/generated 兼容面仍保留部分 raw/session 方法，不能宣称全阶段闭合 |

### 6.7 收益预估

| 文件 | 阶段 C 前 | 阶段 C 后 |
|------|----------|----------|
| `frb_native_service.dart` | ~1054 行 | ~400 行 |
| `app_state.dart` | 1147 行 | ~400 行 |
| 事件延迟 | 300ms 轮询 | 即时推送 |

---

## 7. 未来路径

> 身份去重三阶段完成后，项目演进路线。按优先级排列，各项独立可并行。

### 7.1 近期（承接当前分支，合并 rewrite/auth-flow 前后）

| 项 | 依赖 | 要点 |
|----|------|------|
| **登录流程收尾** | 已完成（本轮已实施 + 审查修复） | `completeLogin`/`teardown`/5 场景回归；已落地 |
| **Mock 移除** | 登录收尾 | 新建 `FakeNativeService`，删 556 行 Mock，默认 instance throw-on-unassigned（见 [mock-removal.md](mock-removal.md)） |
| **阶段 A 去重修复** | 无 | `_checkPeer` 归并入口 + 软归并 + Rust LAN IP 兜底（见本文 §4） |
| **STUN NAT 探测修复**（D1） | 无 | `probe_nat_type` 复用套接字 + IPv6 单测 + Public 分支（缺陷分析 D1） |

### 7.2 中期（身份去重根治 + 数据完整性）

| 项 | 依赖 | 要点 |
|----|------|------|
| **阶段 B：ContactRegistry** | 阶段 A | userId 中心化，握手协议帧化（本文 §5） |
| **阶段 C：FRB 重写** | 阶段 B | ✅ C1 已完成（API 分域 + userId 寻址 + sync 快照 + 旧方法清理）；⏳ C2（事件流推模式）/ C3（Flutter 零数据）待实施 |
| **DatabaseCtl 落地**（D5） | 阶段 B | 本地消息历史 + 文件索引 + 离线队列（`database.rs` 当前 25 行占位）；接 ContactRegistry 持久化 |
| **消息历史下沉** | DatabaseCtl | `app_state.dart` 的 `_messages` 移到 Rust，Flutter 改查询快照（与 contacts 同模式） |
| **P2PNode 泄漏防护**（D2） | 无 | Drop debug-assert / teardown 强制 close；把"必须显式 close"从约定升级为保证 |

### 7.3 远期（生产化 + 性能 + 可扩展）

| 项 | 依赖 | 要点 |
|----|------|------|
| **mTLS 生产网格**（D3） | serve/ca | `serve/auth.rs` 与 `build_p2p_config` 默认接线；CA 预置 root_store 引导流程 |
| **音视频迁移 QUIC 数据报**（D6） | 阶段 C | `voice.rs`/`video.rs` 从可靠流改数据报，降延迟 |
| **HashMap 去重升级** | 阶段 C + 规模增长 | peer n > 5000 时，`_peerByUserId` 遍历升级为 ContactRegistry 内部 HashMap（调用方不变） |
| **群组/文件 gossip 数据下沉** | 阶段 B | `_groupFileLogs`/`_seeders`/`_lastSync` 移到 Rust |
| **文档单一事实源治理** | 无 | `STRUCTURE-OVERVIEW` 与 `ARCHITECTURE` 并发/安全章节去重，明确谁为唯一源 |
| **代码质量整改 M1-M11** | 无 | 低优先级重构（Standby 提升、encode 常量、DHT 键、join_all 超时等） |

### 7.4 路线图时间轴（建议）

```
2026 Q3
├─ 登录收尾 ✅ + Mock 移除 + 阶段 A 去重  ─── 合并 rewrite/auth-flow → master
├─ 阶段 B ContactRegistry（B1/B2a/B2b/B3）✅
└─ 阶段 C FRB 重写（C1 ✅ / C2 ⏳ / C3 ⏳）

2026 Q4
├─ DatabaseCtl 落地 + 消息历史下沉
├─ P2PNode 泄漏防护
└─ STUN NAT 修复

2027 H1
├─ mTLS 生产网格
├─ 音视频 QUIC 数据报
└─ 文档治理 + M1-M11 整改
```

---

## 8. 不变量总表

> 合并任一阶段前，对应不变量须 grep/测试通过。

### 阶段 A（去重修复）

| ID | 不变量 | 验证 |
|----|--------|------|
| DEDUP-1 | `connectByUserId` 含自加拦截 | grep `== _localUserId` |
| DEDUP-2 | `lookupByNickname` 过滤 `_localUserId` | grep |
| DEDUP-3 | `_checkPeer` 统一入口，无分散检查 | grep 调用点 |
| DEDUP-4 | `_peerByUserId` 遍历辅助，无新索引字段 | grep `_userIdToPeer` 应为 0（阶段 A） |
| DEDUP-5 | Rust `is_self_target` 对 LAN IP:port 返回 true | 单测 |

### 阶段 B（ContactRegistry）

| ID | 不变量 | 验证 |
|----|--------|------|
| CR-1 | 自加 → `SelfAdd` | 单测 |
| CR-2 | 同 userId 至多 1 个条目 | 单测 |
| CR-3 | 派生索引双向一致 | 单测 |
| CR-4 | Duplicate 场景去重闭环 | 集成测试 |

### 阶段 C（FRB 重写）

| ID | 不变量 | 验证 |
|----|--------|------|
| FRB-1 | 生成代码无手改 | `git diff` 仅 codegen 产物 |
| FRB-2 | 无 sessionId 公开参数 | grep RustNode 方法签名 |
| FRB-3 | `try_recv_*` 全删 | grep 为 0 |
| FRB-4 | Flutter 无 peer 数据 | grep `_peerCache`/`_addrToId`/`_pollTimer` 为 0 |
| FRB-5 | ContactRegistry 单测覆盖 | `cargo test` |

---

## 9. 验证总矩阵

| # | 场景 | 阶段 | 预期 |
|---|------|------|------|
| T1 | 输入自己 UserID 加好友 | A/B/C | fail "不能添加自己" |
| T2 | 搜索自己昵称 | A/B/C | 结果不含自己 |
| T3 | 直连本机 LAN IP:port | A（Rust 层） | fail（应用层 + Rust 双拦） |
| T4 | A 加 B 后 B 加 A | A/B/C | 列表唯一 B（无重复） |
| T5 | NAT 地址漂移致 addr 不命中 | B/C | 按 userId 复用，无新 session 条目 |
| T6 | `/friend_request` 重复 userId | A/B | 不重复弹窗，仅刷新昵称 |
| T7 | Rust LAN IP 自连 | A | `open_session(lanIp:port)` → `SelfConnection` |
| T8 | ContactRegistry 自加 | B | `bind_identity(local)` → `SelfAdd` |
| T9 | ContactRegistry 重复 session 归并 | B | 软归并，列表无重复 |
| T10 | codegen 后 analyze 0 error | C | 通过 |
| T11 | `contacts.list()` 返回 userId 快照 | C | Flutter 直接渲染 |
| T12 | 事件推模式延迟 < 拉模式 | C | 消息即时到 |

---

## 10. 风险与回滚

| 阶段 | 风险 | 对策 |
|------|------|------|
| A | 软归并致 session 泄漏 | QUIC idle timeout（30s）自然回收；监控 `_peerCache` 与 Rust `peers()` 数量差 |
| A | Rust 本机 IP 枚举性能 | 缓存结果（节点生命周期内不变） |
| A | 拦截影响开发自连调试 | loopback 路径仍可用；LAN IP 路径加显式开关 |
| B | 握手协议改动回归 | 双写过渡（PR B2），一个发布周期后移除 Flutter 解析 |
| B | `if-addrs` 新依赖 | 该 crate 无传递依赖、~200 行；退路是 UDP connect 技巧（仅主 IP） |
| C | codegen 版本漂移 | `flutter_rust_bridge = "=2.12.0"` 已锁；升级需全量回归 |
| C | 子句柄 FRB 不支持 | `#[frb(opaque)]` 标注；退路是函数式 API |
| C | `#[frb(sync)]` 流上下文限制 | ContactRegistry 用 `try_read`（非阻塞） |
| 全部 | 大改动回归 | 每阶段独立 PR，`git revert` 单阶段即回退 |

---

## 附录 A：历史文档映射

本文整合并取代以下分散讨论；完整配套材料保留在本目录以备追溯：

| 归档文档 | 对应本文章节 | 状态 |
|----------|------------|------|
| `PEER_DEDUP_FIX.md`（源提交中无对应文件，仅保留历史引用名） | §4 阶段 A | 被本文 §4 取代（原 §3.2 的 HashMap 索引改为遍历，见 §2.1 决策）；原 3 层防线细节保留备查 |
| 第二轮讨论（布隆/HashMap/遍历/归并入口） | §2.1 技术决策 | 整合为 §2.1 |
| 第三轮讨论（ContactRegistry 中心化） | §5 阶段 B | 整合为 §5 |
| [frb-api-evolution.md](frb-api-evolution.md) | §6 阶段 C | 被本文 §6 压缩引用，FRB 细节（模块重组/codegen 操作/子句柄设计）见完整配套文档 |
| `DEFECT_ANALYSIS.md`（源提交中无对应文件，仅保留历史引用名） | §7 未来路径 | 缺陷编号（D1/D2/D3/D5/D6）对应本文 §7 |
| [mock-removal.md](mock-removal.md) | §7.1 近期 | Mock 移除独立指南 |
| [login-lifecycle-rewrite.md](login-lifecycle-rewrite.md) | §7.1（已完成） | 登录流程代码级实施版，已实施（参见 [登录重写日志](https://github.com/StarrySky7D4/ModCptLib/blob/add9e4024f8e10055508770aaf96c944aab6cceb/Knowledge/logs/2026-07-07-auth-flow-rewrite.md)） |

---

## 附录 B：文件改动总览

### 阶段 A

| 文件 | 改动 |
|------|------|
| `flutter/lib/services/frb_native_service.dart` | `_checkPeer`/`_peerByUserId`/`_updatePeerIdentity` 软归并/自加拦截 |
| `flutter/lib/services/native_service.dart` | 文档补充（无签名变化） |
| `rust/core/src/net/p2p.rs` | `is_self_target` 注入本机 IP + 测试 |

### 阶段 B

| 文件 | 改动 |
|------|------|
| `rust/core/src/data_ctl/contacts.rs` | **新增** ContactRegistry |
| `rust/core/src/data_ctl/mod.rs` | 导出 ContactRegistry |
| `rust/core/src/net/p2p.rs` | P2PNodeInner 挂 contacts；握手钩子调 bind_identity |
| `rust/core/schema/message.capnp` | IdentityFrame/FriendRequest 结构（PR B3） |
| `flutter/rust/src/api/mod.rs` | `record_identity` 钩子（PR B2） |

### 阶段 C

| 文件 | 改动 |
|------|------|
| `flutter/rust/src/api/{node,contacts,messaging,groups,files,events}.rs` | **新增**，拆分 mod.rs |
| `flutter/rust/src/api/mod.rs` | 瘦身为 create_node/log_stream |
| `flutter/lib/src/rust/*` | codegen 重生成（禁止手改） |
| `flutter/lib/services/frb_native_service.dart` | 大幅瘦身（~1054→~400 行） |
| `flutter/lib/services/app_state.dart` | 删数据字段（1147→~400 行） |

### 文档同步（每阶段）

| 文档 | 改动 |
|------|------|
| `Knowledge/rust/core/data_ctl/contacts.md` | **新增**（阶段 B） |
| `Knowledge/rust/core/net/p2p.md` | 更新 is_self_target（阶段 A） |
| `Knowledge/flutter/frb/*.md` | 更新 RustNode API（阶段 C） |
| `Knowledge/flutter/services/frb_native_service.md` | 更新生命周期/去重（各阶段） |
| `Knowledge/flutter/services/app_state.md` | 更新（阶段 C） |
| `Knowledge/logs/` | 每阶段新增日志 |
| `Knowledge/ROADMAP.md` 与 `Knowledge/STRUCTURE.md` | 更新路线图和结构状态 |

---

> **历史定位**：本文记录身份去重与数据下沉的历史方案、当时不变量和验证矩阵。当前实施仅按 [`ROADMAP.md`](../ROADMAP.md)、身份 v2 设计、协议注册表和对应已领取任务执行；附录 A/B 用于追溯，不授予实施权限。
