# 群组与房间架构：实施级历史记录

> **状态**：historical / non-normative，生成于 2026-07-16，2026-07-29 标记收敛。本记录保存曾提出的状态机、帧和裁定，不定义当前实施细节、协议或验收。
>
> **现行优先级**：[`ROADMAP.md`](../ROADMAP.md) 是唯一执行路线；[身份与寻址 v2](identity-and-addressing-v2.md) 约束身份、GroupId 和直接传输。父文档及本文的 R1-R4、14 个裁定、`Scope`/`ConnectionHandle`、常量和帧号均是历史检索材料，不可独立实施。
>
> **已撤销的历史主张**：`IdentityFrame`/自报 `publicKey` 不能分发或授权 principal；空公钥不能导致“未验证”放行；`(group_id, owner_uid)` 不是 GroupId；core 没有 DHT relay、NAT fallback 或 `ViaRelay` 路径。Room/媒体需要 M7 后的独立产品决策。

---

## 目录

1. [文档定位与关系](#1-文档定位与关系)
2. [身份与公钥分发（R1 实施细节）](#2-身份与公钥分发r1-实施细节)
3. [Group 路由模型（R2 实施细节）](#3-group-路由模型r2-实施细节)
4. [Room 路由模型（R3 实施细节）](#4-room-路由模型r3-实施细节)
5. [连接管理与分层复用（架构增量）](#5-连接管理与分层复用架构增量)
6. [并发与一致性（R4 实施细节）](#6-并发与一致性r4-实施细节)
7. [14 个模糊点逐一裁定](#7-14-个模糊点逐一裁定)
8. [阻塞项合并清单（统一最终版）](#8-阻塞项合并清单统一最终版)
9. [需回补父文档的架构增量](#9-需回补父文档的架构增量)
10. [附录：状态机与协议帧速查](#10-附录状态机与协议帧速查)

---

## 1. 文档定位与关系

### 1.1 两文档分工

| 文档 | 层次 | 内容 | 状态 |
|------|------|------|------|
| [groups-and-rooms.md](groups-and-rooms.md) | 历史原则与架构层 | 历史 R1→R4 路线、CRDT/仲裁/DHT 三角色、四大设计原则、不变量 GR-1~12 | historical / non-normative |
| **本文档** | 历史实施级与裁定层 | 历史状态机、协议帧字段、`Scope` 连接抽象、14 模糊点裁定、阻塞项合并 | historical / non-normative 补充 |

### 1.2 一致性结论

两文档在**架构原则上完全一致**（CRDT/epoch 分工、数据报、mesh、签名管线均对齐）。差异仅集中在：
- 阻塞项编号碰撞（父 B-7/B-8 与本文 B-7/B-8 指向不同项）→ §8 统一编号
- 个别模糊点细节裁决需收紧 → §7 逐项裁定
- 本文新增 `Scope`/`ConnectionHandle` 连接管理抽象（父文档未涉及）→ §5 + §9

---

## 2. 身份与公钥分发（R1 实施细节）

### 2.1 历史 `IdentityFrame` 协议（已撤销，禁止实现）

> 以下结构和发送/接收描述仅保留作历史证据。v2 的 `PeerPrincipal` 不能从它或其 `publicKey` 派生；旧 identity MsgType 必须按 `UnsupportedProtocol` 处理。

```capnp
struct IdentityFrame {
    userId        @0 :Text;
    nickname      @1 :Text;
    friendRequest @2 :Bool;
    publicKey     @3 :Data;       # Ed25519 公钥 32 字节
}
```

**发送端**：`announce_identity`（`mod.rs:1224-1234`）时，从 `NodeKeypair`（`ui_store.rs:822-861`）获取公钥原始字节，填入 `publicKey`。

**接收端**：`bind_identity`（`contacts.rs:274-344`）提取公钥，存入 `Contact.public_key`（`contacts.rs:59`，当前恒 None，R1 后填充）。

### 2.2 历史公钥缺失降级（已撤销，禁止放行）

> 本节表格记录被否决的兼容设想。当前认证材料不足必须 fail-closed，不能入库、显示“未验证”或继续验签管线。

**历史且已撤销**：空公钥曾拟降级为「未验证」，并以身份握手时存储的 `Contact.public_key`（而非消息内字段）作为判据。

| 公钥状态 | 验签行为 | UI 标记 | 是否触发 SecurityNotice |
|---------|---------|---------|------------------------|
| 非空 + 验签通过 | 正常入库 | ✓ 已验证 | 否 |
| 非空 + 验签失败 | 丢弃 + 公示 | — | 是（伪造） |
| 空（发送方未升级 R1） | **历史且已撤销**：入库但不验签 | ⚠ 未验证（发送方未升级） | 否（非恶意） |

### 2.3 Target 枚举扩展（裁定见 §7 模糊点 2）

```rust
pub enum Target {
    Single(NodeRef),
    Group(GroupID),
    Room(RoomID),  // R1 引入变体，R3 前返回 NotSupported
}
```

**R1 阶段处理**：`router.send()` 收到 `Target::Room` 时返回 `Err(RouterError::NotSupported("Room routing arrives in R3"))`——**显式运行期错误**，禁止 `unreachable!()`（panic 风险）或 `#[allow(unreachable_patterns)]`（掩盖遗漏）。

---

## 3. Group 路由模型（R2 实施细节）

### 3.1 GroupMemberEvent 与成员表构建

```capnp
struct GroupMemberEvent {
    groupId   @0 :Text;
    userId    @1 :Text;
    action    @2 :UInt8;    # 0=Create, 1=Join, 2=Leave, 3=Delete
    actorUid  @3 :Text;
    epoch     @4 :UInt64;
    signature @5 :Data;     # Ed25519(actor) over (groupId||userId||action||actorUid||epoch)
}
```

**成员表状态机**：

```
初始态 NULL
  │
  ▼
Create(uid=owner, epoch=0)  ← 只有 owner 能产生（genesis 签名）
  │
  ├─▶ Join(uid, ...)    ← owner 签名 或 受邀者本人接受邀请时签名（CRDT ADD）
  ├─▶ Leave(uid, ...)   ← owner 签名 或 成员自愿离开时自签名（CRDT REMOVE）
  └─▶ Delete(...)       ← 仅 owner 签名（epoch bump）
```

**本地成员表**（`router.rs` `GroupMembership`）：
```rust
struct GroupMembership {
    members: HashSet<UserID>,      // 本地视图（CRDT 增量构建）
    epoch: u64,                    // 仅 create/delete/transfer bump（裁定 B-10）
    owner_uid: UserID,
    add_tags: HashMap<UserID, HashSet<Tag>>,  // CRDT OR-Set
    tombstones: HashSet<Tag>,
    deleted: bool,
}
```

**epoch 递增权**（裁定见 §7 模糊点 3 / 12）：
- **epoch 仅用于 create/delete/owner-transfer**（破坏性单点操作）
- **成员增删（Join/Leave）完全由 CRDT OR-Set 管理，不触及 epoch**
- epoch 可标记"成员表快照版本"供 anti-entropy 使用，但不参与增删仲裁

### 3.2 群消息验证管线（含宽限缓存）

`GroupTextMessage` 增加字段：
```capnp
struct GroupTextMessage {
    groupId    @0 :Text;
    content    @1 :Text;
    messageId  @2 :Text;    # UUID，去重用
    senderUid  @3 :Text;
    signature  @4 :Data;    # Ed25519 签名 over (groupId||content||messageId||senderUid)
}
```

**验证流程**（伪代码，含裁定后的宽限缓存）：
```
1. 从传输层获取 sender_uid (session→user 映射)
2. 查本地 GroupMembership.members 是否包含 sender_uid
   - 否：启动宽限窗口（VERIFY_GRACE_SECS = 5s，可配），缓存消息
        向 owner + 随机 2 个在线成员请求成员表刷新（指数退避 5s→10s→20s）
        窗口内收到事件追上 → 继续步骤 3
        超时 → 丢弃 + SecurityNotice（可能是真攻击，也可能是事件滞后）
   - 是：继续
3. 从 ContactRegistry 获取 sender_uid 的公钥
    - **历史且已撤销**：空（未升级 R1）时降级入库，UI 标"⚠ 未验证"，不触发 SecurityNotice
4. 验签：verify(sender_pk, signature, groupId||content||messageId||senderUid)
   - 失败：丢弃 + SecurityNotice（伪造）
5. messageId 去重（LRU）
6. 通过 → 入 UiStore
```

**常量定义**：
```rust
pub const VERIFY_GRACE_SECS: u64 = 5;
pub const VERIFY_RETRY_BACKOFF: &[u64] = &[5, 10, 20]; // 指数退避序列
```

### 3.3 SecurityNotice 与公示风暴抑制

```capnp
struct SecurityNotice {
    groupId     @0 :Text;
    offenderUid @1 :Text;
    messageId   @2 :Text;    # ★ 欺诈消息 ID（去重键）
    reason      @3 :Text;
    witnessUid  @4 :Text;
    signature   @5 :Data;    # witness 签名（防伪造预警）
    ts          @6 :Int64;
}
```

**风暴抑制**（裁定见 §7 模糊点 6）：
- 客户端按 `(offenderUid, messageId)` 去重，同一欺诈消息仅显示一次
- 同一攻击者短时间内对不同 messageId 的同类攻击，UI 限频：每群每分钟最多 N 条，超出折叠为"⚠ 检测到 {n} 次异常（来自 {uid}）"

---

## 4. Room 路由模型（R3 实施细节）

### 4.1 RoomMemberEvent 与生命周期

```capnp
struct RoomMemberEvent {
    roomId      @0 :Text;
    userId      @1 :Text;
    action      @2 :UInt8;   # 0=Invite, 1=Join, 2=Leave, 3=Bye
    actorUid    @3 :Text;
    epoch       @4 :UInt64;
    signature   @5 :Data;
    mediaHint   @6 :UInt8;   # 1=audio, 2=video, 3=both
    sourceGroup @7 :Text;    # 可选（跨群转发标注）
}
```

**状态转换**：
```
NULL
 │
 ▼
Invite(sender→target) → 接收方 UI 显示邀请卡片（TTL 5min，从最后活动起算可重置）
 │
 ▼
Join(target 接受) → 本地创建 Room，开始建立 mesh 连接（每人 N-1 条）
 │
 ├─▶ Leave（自愿或超时）→ 其他人移除其流
 └─▶ Bye（最后一人离开）→ 本地 GC（无需广播，自然消亡）
```

**拓扑上限**：`MAX_ROOM_PARTICIPANTS = 8`。超过拒绝 Join 并提示。

### 4.2 全 Mesh 建连与媒体路径

**建连时机**：参与者收到 Join 事件后，向 `Room.participants` 中所有已有成员建立 QUIC 连接（如不存在），标记 `Scope::Room(room_id)`（见 §5）。

**媒体帧封装**（数据报，紧凑二进制头）：
```
[1B type=0x01][16B room_token][16B sender_token][4B seq][8B ts][HMAC 32B][payload]
```
- HMAC 使用房间 ECDH 协商的对称密钥（裁定 B-1：会话密钥模式）
- `room_token`/`sender_token` 为 RoomID/UserID 的 16 字节哈希截断（避免每帧传完整 UUID 36B）

**历史 R3 NAT 失败处理**（已撤销，裁定见 §7 模糊点 8 / 阻塞项 B-3）：
- R3 仅支持全直连（局域网或公网 IP）
- **部分连通不阻断整个房间**：A-B 不通但 A-C、B-C 通，通话对 AC/BC 对仍可用，仅 AB 对在 UI 标灰
- 被否决的旧路径曾提出 R4 DHT 中继兜底；当前 core 只能报告直接拨号失败，不能接入任何 relay/NAT fallback。

### 4.3 房间邀请卡片

`UiStore.MessageRecord`（`ui_store.rs:62-90`）增加 `kind=KIND_ROOM_INVITE(4)` 和 `room_id`。

**重复邀请处理**（裁定见 §7 模糊点 9）：
- 按 `room_id` 去重
- **已加入 → 静默忽略新卡片**（仅 toast "已在通话中"），**禁止自动导航**（尊重用户当前操作，呼应 `signaling.rs` 用户确认反骚扰原则）
- 未加入 → 展示卡片等待用户点击

**卡片有效期**（裁定见 §7 模糊点 7 / B-12）：
- 默认 5min，**从最后一次活动**（任意 Join/媒体帧）起算，支持晚加入者维持房间活跃
- 持久化 DB `room_invites` 表，过期后 UI 标"已结束"，点击提示"房间已结束"

---

## 5. 连接管理与分层复用（架构增量）

> 本节是本文档相对父文档的**最重要增量**。父文档只描述了 `peers: HashMap<String, NodeRef>`，未解决"好友连接与群连接复用"问题。

### 5.1 Scope 抽象

```rust
/// 一个 QUIC 连接可同时服务于多个逻辑作用域。
enum Scope {
    Contact,             // 好友（不可主动移除，仅由联系人删除触发）
    Group(GroupID),      // 群成员
    Room(RoomID),        // 房间参与者
}

struct ConnectionHandle {
    inner: Arc<QuicConnection>,
    scopes: RwLock<HashSet<Scope>>,   // 引用计数
}
```

**生命周期规则**：
- 建立连接时指定初始 scope 加入集合
- 移除某 scope 后，若集合为空 → **延迟 10s 关闭连接**（吸收 churn，避免频繁重连）
- `Scope::Contact` 不可移除（只能由联系人删除触发）
- 自己退出群 → 遍历该群 scope 的连接，移除 `Scope::Group(gid)`，若集合空且无 Contact → 延迟关闭
- 成员被踢 → 收到 Leave 事件后移除本地成员 + 对应 scope

### 5.2 重连优先级（裁定见 §7 模糊点 10）

**按用户感知紧急度**排序（非按 scope 类型固定排序）：

| 优先级 | Scope | 理由 |
|--------|-------|------|
| 1（最高） | `Room`（活跃媒体） | 媒体中断毫秒级可感知（卡顿/掉线） |
| 2 | `Contact` | 社交期待，秒级可接受 |
| 3 | `Group`（异步文本） | 文本重连秒级可接受 |

延迟关闭定时器（§5.1）已大幅减少闲连接重连风暴，故优先级不影响资源。

### 5.3 连接对 UI 的不可见性

- 只有 `ContactRegistry` 的变化才通知 UI 上线/下线
- 群/房间连接的状态仅影响 `GroupMembership` 内部的在线集合，用于消息路由，**不推送给用户**

**房间内"半可见"状态**（裁定见 §7 模糊点 11）：
- 房间参与者列表可见（谁正在说话/静音），但这不是"好友在线"
- `RoomParticipant`（内存态，离开即销毁）与 `ContactRegistry`（持久身份）严格分离
- 离开房间即销毁房间内状态，不写入 `ContactRegistry`

---

## 6. 历史并发与一致性（R4 实施细节，非规范）

### 6.1 CRDT OR-Set 与 epoch 的分工（裁定见 §7 模糊点 3 / 12）

**两文档共识**：

| 操作类型 | 机制 | epoch |
|---------|------|-------|
| Join（成员加入） | CRDT ADD（add-tag） | 不 bump |
| Leave（成员离开） | CRDT REMOVE（tombstone by tag） | 不 bump |
| Create（建群） | owner genesis 签名 | bump（epoch=0→1） |
| Delete（删群） | owner 签名 | bump |
| Transfer（转让 owner） | 当前 owner 链式签名 | bump |

**CRDT Merge 语义**：
```
ADD(uid, tag)      — tag = hash(actor_uid || uid || ts || nonce)，全局唯一
REMOVE(uid, tags)  — 必须引用要移除的具体 add-tag
S_merged = (S_A ∪ S_B) \ (T_A ∪ T_B)   # 并集减去并集 tombstone
```

**epoch 的辅助用途**：标记"成员表快照版本"供 anti-entropy Merkle 根计算，但不参与增删仲裁。

### 6.2 Anti-Entropy 与同步风暴抑制

**周期同步**：节点间周期（如 30s）交换 `MerkleRoot(members, tombstones)`。

**风暴抑制**（裁定见 §7 模糊点 13）：
- 检测到分叉（Merkle 根不同）→ **随机延迟 0-2s** 后发送同步请求
- 延迟后**再次校验 Merkle 根**（闸门），仍不同才发同步请求
- 每节点每群每周期最多触发一次全量同步
- 每个分区事件只触发一次同步

### 6.3 GroupID 冲突仲裁（裁定见 §7 模糊点 14）

**`GroupIdentity`**（父文档 §11.2）：
```rust
struct GroupIdentity {
    group_id: GroupID,
    owner_uid: UserID,
    created_at_ms: i64,
    owner_pubkey: Vec<u8>,       // R1 分发
    genesis_sig: Vec<u8>,        // owner 对上述字段的签名
}
```

**冲突处理 — 自动化分层仲裁**（不依赖用户判断密码学有效性）：

```
节点 X 收到声称属于 Group G 的成员事件：
  ① 查本地是否有 G 的 GroupIdentity
     ├─ 无 → 首次见到：记录 genesis，信任
     └─ 有 → 比对 genesis_sig
        ├─ 一致 → 同一群，正常处理
        └─ 不一致 → 进入仲裁：
           【第一关】owner 公钥校验：
              用声称 owner_uid 的 R1 公钥验签 genesis_sig
              ├─ 验签失败 → 非 owner 伪造 → 丢弃 + SecurityNotice（非 owner 无权签名）
              └─ 验签通过 → 第二关
           【第二关】双有效冲突：
              两个 genesis 都验签通过（两个不同真 owner 各自签名）
              → 取 created_at_ms 最早者
              → 其余发 SecurityNotice
           【第三关】用户介入：
              仅当时间戳相同（极端）才上报 UI
```

**关键**：owner 公钥经 R1 身份握手建立（传输认证），伪造 genesis 无法过第一关，"先到者胜"的分裂风险被消除。

---

## 7. 14 个模糊点逐一裁定

> 格式：`问题 / 倾向 / 裁定 / 补充理由`

### 模糊点 1：公钥缺失下的历史降级策略（已撤销）

- **倾向**：空公钥 → 降级"未验证"，UI 标记，不触发 SecurityNotice
- **历史裁定，已撤销**：曾接受以存储公钥区分“未验证”消息的方案。
- **现行结论**：`IdentityFrame` 和 `Contact.public_key` 不能建立 v2 principal；credential evidence 缺失、状态未知或验证失败时必须在业务副作用前 fail-closed，没有“发送方未升级”的放行 UI。

### 模糊点 2：Target::Room 在 R1 的穷举处理

- **倾向**：`#[allow(unreachable_patterns)]` 临时放行
- **裁定**：❌ **拒绝**。改用**显式运行期错误**：`Target::Room` 在 R1 引入（保持三命名空间正交性清晰），`router.send()` 收到时返回 `Err(RouterError::NotSupported("Room routing arrives in R3"))`。
- **理由**：`unreachable!()` 误触 panic（生产事故）；`allow` 掩盖未来真实遗漏。显式错误自文档化、编译安全、R3 移除时一处删除。

### 模糊点 3：非 owner 的 Leave 事件 epoch 管理

- **倾向**：(a) Leave 脱离 epoch，仅走 CRDT REMOVE
- **裁定**：✅ **确认接受**（两文档共识，非真冲突）
- **理由**：即父文档 §10.2 的"层1 CRDT / 层2 epoch"分工。增删走 CRDT，create/delete/transfer 才 bump epoch。

### 模糊点 4：宽限缓存窗口与反查机制

- **倾向**：5s 可配，UI 显示"验证中…"，超时"验证失败"
- **裁定**：✅ **接受，扩展反查目标**
- **补充**：反查对象不应仅 owner（可能离线）= owner + 随机 2 个在线成员（迷你 gossip）。重复失败用指数退避（5s→10s→20s）。R4 DHT 就近补发后误报率显著下降。
- **常量**：`VERIFY_GRACE_SECS = 5`，`VERIFY_RETRY_BACKOFF = [5, 10, 20]`。

### 模糊点 5：连接引用计数粒度

- **倾向**：`ConnectionHandle { scopes: HashSet<Scope> }`
- **裁定**：✅ **接受并提升为正式架构组件**（§5）
- **理由**：本文档相对父文档的最重要增量——父文档未解决"好友连接与群连接复用"。
- **延迟关闭**：移除 scope 后若集合空 → 延迟 10s 关闭（吸收 churn）。

### 模糊点 6：重复公示与风暴

- **倾向**：按 `(offenderUid, messageId)` 去重
- **裁定**：✅ **接受，加窗内类聚去重**
- **补充**：同一攻击者短时间对不同 messageId 的同类攻击，UI 限频（每群每分钟最多 N 条），超出折叠为"⚠ 检测到 {n} 次异常（来自 {uid}）"。

### 模糊点 7：房间创建者角色 + 卡片过期

- **倾向**：5min 过期，发起人离开不销毁（除非最后一人）
- **裁定**：✅ **接受，TTL 重置细化**
- **关键细化**：TTL 从**最后一次活动**（任意 Join/媒体帧）起算，而非创建时刻——支持晚加入者维持房间活跃。卡片持久化 DB 但过期后 UI 标"已结束"。

### 模糊点 8：历史全 Mesh NAT 回退（已撤销）

- **倾向**：R3 仅直连，UI 提示部分不通
- **历史裁定，已撤销**：曾允许部分直连并由 R4 DHT 中继兜底。
- **现行结论**：当前没有获授权的 Room 产品路径，更没有 core relay；未来产品必须明确报告每条直接链路失败。

### 模糊点 9：重复邀请与已加入状态 UI

- **倾向**：按 room_id 去重，已加入则直接 join + 导航
- **裁定**：⚠️ **部分修正**
- **修正**："已加入 → 直接导航通话界面"会抢占用户当前操作。改为：**已加入 → 静默忽略新卡片**（仅 toast "已在通话中"）；未加入 → 展示卡片等待用户点击。
- **理由**：呼应 `signaling.rs` 用户确认反骚扰原则，用户始终掌握加入时机。

### 模糊点 10：好友与群/房间共享连接时的重连优先级

- **倾向**：Contact > Room > Group
- **裁定**：⚠️ **否决此排序**
- **修正**：改为**按用户感知紧急度**：`Room(活跃媒体) > Contact > Group(异步文本)`。
- **理由**：媒体中断毫秒级可感知（卡顿/掉线），文本重连秒级可接受。延迟关闭定时器已减少闲连接重连风暴，优先级不影响资源。

### 模糊点 11：群成员"在线"状态半可见

- **倾向**：房间参与者列表可见，不写入 ContactRegistry
- **裁定**：✅ **接受**
- **补充**：`RoomParticipant`（内存态，离开即销毁）与 `ContactRegistry`（持久身份）严格分离。UI 说话/静音态仅房间内可见。

### 模糊点 12：CRDT 与 epoch 的关系

- **裁定**：✅ **两文档共识确认**（同模糊点 3）
- **补充**：epoch 标记成员表快照版本供 anti-entropy，但不参与增删仲裁。

### 模糊点 13：同步风暴抑制

- **倾向**：随机 0-2s 延迟，每分叉事件一次
- **裁定**：✅ **接受，加 Merkle 根闸门**
- **补充**：延迟后再次校验 Merkle 根（闸门），仍不同才发同步请求；每节点每群每周期最多一次全量同步。

### 模糊点 14：首次收到冲突 Genesis 的处理

- **倾向**：暂停群操作，要求用户确认
- **裁定**：⚠️ **否决"问用户"**（用户无法判断密码学有效性）
- **修正**：自动化分层仲裁（§6.3）：
  1. 第一关：owner 公钥校验（非 owner 伪造必然验签失败 → 丢弃 + 公示）
  2. 第二关：双有效冲突取 created_at_ms 最早者
  3. 第三关：仅时间戳相同才上报 UI
- **理由**：owner 公钥经 R1 身份握手建立，伪造 genesis 无法过第一关。

### 裁定统计

| 裁定类型 | 数量 | 模糊点 |
|---------|------|--------|
| ✅ 接受倾向 | 9 | 1, 3, 4, 5, 6, 7, 8, 11, 12, 13 |
| ✅ 接受 + 扩展 | 4 | 4（反查目标）, 6（类聚去重）, 7（TTL 重置）, 13（Merkle 闸门） |
| ⚠️ 部分否决/修正 | 3 | 9（禁止自动导航）, 10（优先级重排）, 14（否决问用户） |
| ❌ 拒绝 | 1 | 2（拒绝 unreachable/allow） |

---

## 8. 历史阻塞项清单（非现行裁定）

> 此表的“已定”“待定”、R 标签、上限、codec、relay 和媒体结论均没有现行效力。安全与协议决策由 M2，群组由 M6，Room/媒体产品边界由 M7 重新裁定。

> 父文档 B-1~B-8 编号保留（已推送 GitHub，不重排）。新文档项追加为 B-9~B-14。NAT 项合并去重（B-3 ≡ 原 B-12）。

| 编号 | 决策项 | 影响阶段 | 裁决状态 | 倾向/结论 |
|------|--------|---------|---------|----------|
| B-1 | 签名粒度 | R3 | 待定 | 会话密钥 ECDH + HMAC-SHA256 + 每 N 帧签一次 |
| B-2 | mesh 上限 | R3/R4 | 待定 | 8 人硬上限；超限拒绝 + 提示；N>8 未来转 SFU |
| **B-3** | NAT 失败对处理 | R3/R4 | **历史且已撤销** | 曾拟 R3 直连、部分连通不阻断，并由 R4 DHT 中继兜底；现行 core 不采用。 |
| B-4 | 跨群转发权限 | R3 | 待定 | participant 可转发 + via_group 标注 |
| B-5 | owner succession_chain | R2 | 待定 | 首版冻结，succession 未来扩展 |
| **B-6** | 音频混音实现 | R3 | 待定（**最大阻塞**） | — |
| B-7 | 编解码器协商 | R3 | 待定 | RoomInvite 携带 codec/bitrate |
| B-8 | 拥塞控制/自适应码率 | R4 | 待定 | 应用层 RTT/丢包探测 + 降码率 |
| **B-9** | 公钥强制时间线 | R1/R2 | **已裁定**（模糊点1） | R2 初期可选降级（存储公钥判据），flag day 后强制 |
| **B-10** | Leave 是否脱离 epoch | R2 | **已解决**（模糊点3/12） | 脱离，纯 CRDT REMOVE；epoch 仅 create/delete/transfer |
| **B-11** | 宽限窗口默认值 | R2 | **已裁定**（模糊点4） | 5s 常量可配；反查 owner+随机2成员；指数退避 [5,10,20] |
| **B-12** | 邀请卡片有效期 | R3 | **已裁定**（模糊点7） | 5min，从最后活动起算可重置；过期 UI 标"已结束" |
| **B-13** | ConnectionHandle scope 实现 | R1/R2 | **已裁定**（模糊点5） | `HashSet<Scope>` + 移除后延迟 10s 关闭 |
| **B-14** | 重连优先级 | R2/R3 | **已裁定**（模糊点10） | Room(媒体) > Contact > Group（非 Contact>Room>Group） |

**裁决统计**：14 项中 6 项（B-9/10/11/12/13/14）经本次**已可定案**，从"阻塞"转为"已决策"，剩余 8 项待定。**B-6（音频混音）仍是 R3 落地的最大技术阻塞**。

---

## 9. 需回补父文档的架构增量

以下 3 处是本文档相对 [群组与房间架构](groups-and-rooms.md) 的增量，建议作为“附录 D”回补进父文档（本文档已完整记录，父文档修订时可引用本文）：

1. **`Scope` + `ConnectionHandle` 抽象**（§5）——父文档未涉及连接复用。R1 引入 `Scope { Contact, Group(GroupID), Room(RoomID) }`，`ConnectionHandle.scopes` 引用计数，移除后延迟 10s 关闭。
2. **群消息验证管线宽限缓存**（§3.2）——父文档 §6.3 已有管线，但"宽限窗口 + 反查 owner+随机成员 + 指数退避"细节（B-11 裁决）应补入。
3. **重连优先级按紧急度**（§5.2）——父文档无此设计。`Room(活跃媒体) > Contact > Group(异步文本)`。

**不变量补充**（父文档 GR-1~12 之外）：

| ID | 不变量 | 守护机制 |
|----|--------|---------|
| **GR-13** | 单个 QUIC 连接可同时承载多个 Scope，移除 Scope 不立即断连（延迟 10s） | `ConnectionHandle.scopes: HashSet<Scope>` 引用计数 |
| **GR-14** | 公钥缺失降级判据基于传输认证的存储公钥，非消息内字段 | `Contact.public_key`（mTLS 建立） |
| **GR-15** | 验签失败先宽限缓存（区分"传播滞后"与"真攻击"），超时才公示 | `pending_verification` + VERIFY_GRACE_SECS |
| **GR-16** | Room 邀请不自动抢占用户焦点（已加入静默忽略，未加入等点击） | 呼应 signaling.rs 反骚扰原则 |
| **GR-17** | GroupID 冲突仲裁自动化分层，用户仅介入极端时间戳相同情况 | §6.3 三关仲裁 |

---

## 10. 附录：状态机与协议帧速查

### 10.1 协议帧总览

| MsgType | 码位 | capnp struct | 传输 | 阶段 |
|---------|------|-------------|------|------|
| `GroupMemberEvent` | 0x14 | `GroupMemberEvent` | 可靠流 | R2 |
| `GroupSecurityNotice` | 0x15 | `SecurityNotice` | 可靠流 | R2 |
| `RoomMemberEvent` | 0x16 | `RoomMemberEvent` | 可靠流（控制） | R3 |
| Room 媒体帧 | 数据报 | 紧凑二进制头（非 capnp） | 不可靠数据报 | R3 |
| `MeshAnnounce` | 0xA0 | `MeshAnnounce` | 可靠流 | R4 |
| `CapabilityQuery` | 0xA1 | — | 可靠流 | R4 |
| `RelayOffer` | 0xA2 | `RelayOffer` | 可靠流 | R4 |
| `ReputationUpdate` | 0xA3 | — | 可靠流 | R4 |

### 10.2 Group 成员表状态机速查

```
NULL ──Create(owner, epoch=0, genesis_sig)──▶ ACTIVE
                                               │
                 ┌─────────────────────────────┼─────────────────────────┐
                 ▼                             ▼                          ▼
        Join(uid, CRDT ADD)         Leave(uid, CRDT REMOVE)      Delete(owner, epoch++)
        （不 bump epoch）            （不 bump epoch）             （bump epoch → DELETED）
```

### 10.3 Room 生命周期状态机速查

```
NULL ──Invite──▶ INVITED ──Join──▶ ACTIVE ──┬──Leave──▶ (其他人移除其流)
                                              └──Bye(最后一人)──▶ GC
```

### 10.4 连接 Scope 状态机速查

```
建连(scope=A) ──加 scope=B──▶ {A,B} ──移除 A──▶ {B}
                                         │
                                         └─{B}非空 → 保留
                                            {B}空 → 延迟10s → 关闭
```

---

> **本文是 [群组与房间架构](groups-and-rooms.md) 的实施级历史补充。** 任何仍有价值的设计输入必须由 M2/M6/M7 的现行规范重新接受；不得从 R1 或本文的 `IdentityFrame`、公钥降级、复合 GroupId、DHT relay 或 Room 路线起步实施。
