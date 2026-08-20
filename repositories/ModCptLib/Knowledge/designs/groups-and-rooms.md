# 群组与房间架构重构方案（历史设计记录）

> **状态**：historical / non-normative，生成于 2026-07-14，2026-07-29 标记收敛。本文保留五轮讨论的完整设计和取证，**不**定义现行路线、协议、schema、API、数值上限或实现授权。
>
> **现行优先级**：[`ROADMAP.md`](../ROADMAP.md) 是唯一执行路线；[身份与寻址 v2](identity-and-addressing-v2.md) 是身份、授权、GroupId 和直接传输边界的权威模型。本文的 R1-R4、采纳结论和改动清单仅是历史检索标签，不能被领取或实施。
>
> **已撤销的历史主张**：任何 `IdentityFrame`/`publicKey` 自报身份或公钥分发都不能授权；公钥缺失时不能“未验证放行”；`(group_id, owner_uid)` 不是 GroupId 唯一性；core 不实现 DHT relay、`ViaRelay`、`ViaDht`、NAT fallback 或自动媒体路径切换。当前 Room 也没有由本文授权。
>
> **保留范围**：下文中的代码、帧、常量、时间、拓扑和测试描述是当时的提案或观察，可能已过时；它们只能作为历史证据阅读，不得与上述现行结论相冲突。
>
> **历史内容范围**：本文曾整合一次会话内五轮讨论的全部结论：
> - 第一轮：群组架构结构性问题分析（P0/P1/P2 分级缺陷）
> - 第二轮：详尽重构方案（Phase G1 协议语义补全 → G2 成员/在线解耦 → G3 权威模型）
> - 第三轮：群组通话补充（Phase G4 临时音视频聊天室、mesh、数据报、跨群转发讨论）
> - 第四轮：四大设计原则细化（ID 分离 / n(n-1)/2 mesh / GroupID+签名验签 / Room 数据报简化）
> - 第五轮：四个深度问题（拓扑详解 / 并发收敛 CRDT / GroupID 冲突仲裁 / DHT 三角色优化）
>
> 上述内容曾作为分散讨论的汇总；现只保留为历史记录。后续实现不能以本文或 [身份与 FRB 演进](peer-identity-and-frb.md) 的 `IdentityFrame`、ContactRegistry 公钥或 session 设想为依赖。

---

## 目录

1. [问题陈述与根因](#1-问题陈述与根因)
2. [技术决策记录（含被排除方案）](#2-技术决策记录含被排除方案)
3. [四大设计原则](#3-四大设计原则)
4. [四阶段演进路线（R1→R4）](#4-四阶段演进路线r1r4)
5. [Phase R1：ID 分离与公钥扩展](#5-phase-r1id-分离与公钥扩展)
6. [Phase R2：群组安全模型（签名验证）](#6-phase-r2群组安全模型签名验证)
7. [Phase R3：房间模型（数据报 + 简化生命周期）](#7-phase-r3房间模型数据报--简化生命周期)
8. [Phase R4：Mesh 拓扑与 DHT 优化](#8-phase-r4mesh-拓扑与-dht-优化)
9. [群组拓扑详解](#9-群组拓扑详解)
10. [并发收敛机制（CRDT + 仲裁）](#10-并发收敛机制crdt--仲裁)
11. [GroupID 冲突仲裁](#11-groupid-冲突仲裁)
12. [DHT 三角色优化详解](#12-dht-三角色优化详解)
13. [不变量总表](#13-不变量总表)
14. [验证总矩阵](#14-验证总矩阵)
15. [风险与回滚](#15-风险与回滚)
16. [附录 A：文件改动总览](#附录-a文件改动总览)
17. [附录 B：待决策阻塞项](#附录-b待决策阻塞项)
18. [附录 C：未覆盖的设计空白](#附录-c未覆盖的设计空白)

---

## 1. 问题陈述与根因

### 1.1 现状诊断方法

逐文件核实群组功能在五层的实现：协议层（`route.rs` `MsgType`）、序列化层（`serializer.rs`）、路由层（`router.rs` 注册表与 fan-out）、门面层（`data_ctl/mod.rs` 广播逻辑）、FRB 层（`groups.rs`）。以下缺陷按严重度分级。

### 1.2 P0 严重逻辑缺陷

| # | 缺陷 | 位置 | 根因 |
|---|------|------|------|
| **P0-1** | 删除群组反而让成员"重建"该群 | `mod.rs:1258-1268` `delete_group` 先 `notify_group_members` 广播 `GroupCreate` 帧，接收方 `create_group_with_id` 整体覆盖（`router.rs:338-342`） | `MsgType` 只有 `GroupCreate(0x12)`，无 `GroupDelete`，删除语义复用到创建帧 |
| **P0-2** | 增/删成员复用 GroupCreate，无操作语义 | `mod.rs:1270-1284` `add/remove_member` → `notify_group_members` 发整份成员列表；接收方只能整体 `insert` 覆盖 | 无版本号/向量时钟，并发变更后到者覆盖；被移除者收不到通知，其本地群组仍存在，仍可 `send_text_group` 成功（`router.rs:410-424` `resolve_group_id` 仍命中）→ **权限失效** |
| **P0-3** | 离线 = 永久踢出群组 | `router.rs:521-528` `dispatch_loop` 处理 `Disconnected` 时 `purge_user_mappings`（`:589-613`）把断线用户 user_id 从**所有**群组成员列表永久 `retain` 移除 | 成员资格与在线状态耦合在同一份数据 |
| **P0-4** | 群消息回环 + 缺去重 | `create_group` 显式把创建者塞进成员表（`groups.rs:98-112`），fan-out 发给自己一份（`router.rs:224-292`），`group_text_recv_task`（`mod.rs:291-300`）当普通入站处理；`GroupTextMessage`（`msg.rs:25-30`）**无 message_id/序号** | 无去重基础设施，发送方消息极易显示两遍 |

### 1.3 P1 设计/可扩展性

| # | 缺陷 | 位置 | 说明 |
|---|------|------|------|
| **P1-5** | fan-out 是 N 次单播，O(N²) 流量 + 全连接拓扑假设 | `router.rs:282-291` 逐成员 `gateway.send`；无多播/树形/中继聚合 | N 人群每人都发言时全网 O(N²) 流量；NAT/非直连成员 `resolve_member_node` 解析不到就 `warn` + 静默丢弃（`router.rs:274-280`），无离线消息补偿 |
| **P1-6** | 无所有者/权限/角色模型 | `mod.rs:1258-1284` 任何成员都能 `delete_group` / `add_member` / `remove_member` | 叠加 P0-2，任意成员可"踢人"并广播覆盖全网 |

### 1.4 P2 代码质量

| # | 缺陷 | 位置 |
|---|------|------|
| **P2-7** | 三路径解析逻辑重复 | `send()`（`router.rs:241-273`）与 `resolve_member_node()`（`router.rs:431-449`）各实现一遍 `user_to_session → contacts → peers`；大群下锁竞争放大 |
| **P2-8** | 错误语义误导 | `router.rs:232-234` 群组不存在返回 `GatewayError::NotConnected`，实为"群不存在" |
| **P2-9** | 内存/DB 双写无原子保证 | `persist_group`（`mod.rs:1287-1298`）与 delete 的 notify→delete→db.delete 无事务；`GroupCreate` 接收任务 router 创建与 `upsert_group`（`mod.rs:710-714`）无原子性 |

### 1.5 通话架构现状（群通话无法直接扩展的根因）

| 现状 | 位置 | 问题 |
|------|------|------|
| `CallState` 仅 `peer_user`/`peer_alias` 单数 | `ui_store.rs:127-141` | 无法表达多方参与者 |
| 信令走文本通道 `/call_invite {media}{group}` | `proto.rs:17-75` | 每条信令 = 开一条新 QUIC 流（`P2PGateway::send` 每次 `open_stream`）→ 这就是"反复握手" |
| 媒体每帧 `router.send` → `open_stream` | `voice.rs:44-59`、`video.rs:44-63`；voice.rs:1-4 注释已自认"未来优化应走 datagram" | 逐帧握手开销 |
| `send_datagram` 已存在但**无接收转发路径** | `p2p_gateway.rs:467`（send 有）、`:563-584`（recv 只循环 `accept_stream`） | `GatewayEvent` 无 `Datagram` 变体，路由 dispatch 不处理数据报 |
| `start_group_call` 只是给 1:1 状态贴 group 标签 | `mod.rs:1401-1404` | 无任何多方媒体能力 |
| 信令层 `pending: Mutex<Option<PendingCall>>` 单槽 | `signaling.rs:121-122` | 纯 1:1 设计 |
| `MAX_DATAGRAM_SIZE = 65536` | `p2p.rs:24` | 语音帧绰绰有余；视频关键帧可能超限需分片 |

### 1.6 身份与签名基础设施缺口（P0 安全根因）

| 缺口 | 位置 | 后果 |
|------|------|------|
| `IdentityFrame` **不携带公钥** | `serializer.rs:175-181` 只有 `{userId, nickname, friendRequest}` | 签名验证无信任锚 |
| `Contact.public_key` 字段存在但**恒为 None** | `contacts.rs:59` | 从未被身份握手填充 |
| 群消息**无签名** | `msg.rs:25-30` `GroupTextMessage` 无 sig 字段 | 任何人可伪造群消息 |
| `NodeKeypair` Ed25519 仅用于文件分享 | `ui_store.rs:822-861` | 未用于消息/事件签名 |
| `DhtNode` 独立网关，未与 DataRouter 集成 | `dht.rs:37-162` | 无法用于群/房间发现优化 |

---

## 2. 技术决策记录（含被排除方案）

### 2.1 成员资格数据结构：整体覆盖 vs 签名事件流

| 方案 | 评估 | 结论 |
|------|------|------|
| **整体覆盖**（现状，每次广播完整名册） | 并发变更后到者覆盖；无收敛；N 人每次 O(N) 名册序列化 | ❌ P0-1/P0-2 根因 |
| **签名事件流 + 增量构建** | 仅传 `(ScopeID, UserID, action, sig)`；接收方累积事件增量更新本地成员表；epoch + 签名收敛 | **✅ 采用（R2）** |
| **CRDT OR-Set**（事件流 + tombstone） | 并发 ADD 不同 uid 自动并集；并发 ADD/REMOVE 用 add-tag/tombstone 无冲突 merge | **✅ 大群（>8）采用（§10）** |

### 2.2 传输：可靠流 vs 不可靠数据报

| 帧/用途 | 决策 | 理由 |
|---------|------|------|
| 群成员事件、群文本消息 | **可靠流** | 低频、必须到达；复用现有 `router.send`（open_stream） |
| 房间控制信令（Invite/Join/Leave/Bye） | **可靠流** | 低频、状态正确性依赖到达 |
| 房间媒体帧（voice/video） | **不可靠数据报** | 高频、可丢、免逐帧握手；经 `gateway.send_datagram` |
| 安全公示（SecurityNotice） | **可靠流** | 必须到达，安全相关 |

### 2.3 签名粒度：逐帧签名 vs 会话密钥 HMAC

| 方案 | 开销（语音 Opus ~100B 帧） | 安全性 | 结论 |
|------|---------------------------|--------|------|
| 逐帧 Ed25519 签名（64B） | +64% 带宽 | 最强（每帧不可否认） | 群文本低频可用；媒体高频不可接受 |
| **会话密钥 ECDH + HMAC-SHA256（32B）+ 每 N 帧签一次** | +32%（且每 N 帧才一次） | 强（前向安全 + 周期不可否认） | **✅ Room 媒体推荐** |
| 不签名 | 0 | 零（任何人可伪造） | ❌ 违反原则③ |

### 2.4 Room 拓扑：全 mesh vs SFU vs DHT 中继（历史比较，已撤销 relay 选项）

| 方案 | 上行/人 | 延迟 | NAT 容忍 | 复杂度 | 结论 |
|------|---------|------|----------|--------|------|
| **全 mesh（n(n-1)/2 直连）** | (N-1)×流 | 最低（1 跳） | 差（需全直连） | 中 | ✅ N ≤ 8 优先 |
| **DHT 中继转发** | 1×流到中继 | 中（经中继） | 好（中继兜底） | 高 | **历史且已撤销**：不得作为 NAT 不通/弱终端时的 core 降级。 |
| SFU 中心转发 | 1×流 | 中 | 好（需 SFU） | 很高（需服务） | ⏸ N > 8 未来扩展，当前不做 |

**历史结论，已撤销**：本节曾提出 Room 不强制 P2P 和 `Direct / ViaRelay / ViaDht` 自适应。现行边界只允许直接 QUIC；不可直连必须报告链路失败，不得在 core 选择 relay 或 DHT 路径。

### 2.5 Group 拓扑：单一 star vs 分层混合

| 方案 | 评估 | 结论 |
|------|------|------|
| **Sender-Centric Star**（现状） | 发送者承 O(N) 上行；发送者宕机则群瘫痪；要求发送者与全员直连 | ❌ 仅小群+全直连 |
| **固定 Gossip（fan-out=k）** | O(k) 常数带宽；O(logₖ N) 跳延迟；多路径容错 | ✅ 中群（8 < N ≤ 32） |
| **DHT Overlay** | O(log N) 路由；分布式存储+就近 serve | ✅ 大群（N > 32） |
| **分层混合（按规模自适应）** | 综合 | **✅ 采用（§9）** |

### 2.6 成员并发收敛：纯 epoch/LWW vs CRDT

| 方案 | 评估 | 结论 |
|------|------|------|
| **纯 epoch LWW** | 无法表达"并发加不同人应都保留"（后到者覆盖） | ❌ 增删场景失败 |
| **CRDT OR-Set + epoch/owner 仲裁** | 增删 CRDT 无冲突 merge；破坏性操作（create/delete/transfer）epoch+owner 签名单点仲裁 | **✅ 采用（§10）** |

### 2.7 DHT 集成：独立网关 vs 共享连接池

| 方案 | 评估 | 结论 |
|------|------|------|
| **DhtNode 独立 P2PGateway**（现状 `dht.rs:47`） | 与 DataRouter 重复建链；连接池不共享 | ❌ 浪费资源 |
| **DataRouter 持有 DhtNode 引用，共享 gateway** | 单一连接池；DHT 路由表复用已有链路 | **✅ 采用（R4）** |

---

## 3. 四大设计原则

本节确立贯穿全文的四项核心原则，所有后续设计均据此推导。

> **历史说明**：本节的“原则”、`IdentityFrame` 管线、R 标签和任何 relay/DHT 描述均已撤销为现行规范；它们只记录当时的推导，不能覆盖本文开头的 v2/M0-M7 边界。

### 原则 ①：UserID / GroupID / RoomID 三个正交命名空间

| ID | 类型 | 来源 | 生命周期 | 正交性 |
|----|------|------|----------|--------|
| UserID | server 字符串 | 登录服务签发 | 永久 | 联系人/身份的唯一键 |
| GroupID | UUID v4 | `create_group` 本地生成 | 持久（DB） | 群组唯一键；**不依赖**任何 RoomID |
| RoomID | UUID v4 | `create_room` 本地生成 | 临时（最后一人离开销毁） | 房间唯一键；**可脱离**任何 GroupID 独立存在；`source_group` 仅作 UI 提示 |

**解耦点**：
- `Target` 枚举三分支：`Single(NodeRef) | Group(GroupID) | Room(RoomID)`
- 消息 `scope` 字段三分：`enum { Private(UserID), Group(GroupID), Room(RoomID) }`
- 当前 `conversation`（`ui_store.rs:67`）用 peer_user_id 或 group name 混编 → 重构后按 scope 明确分流

### 原则 ②：Room 连接数 = n(n-1)/2（mesh + DHT 优化）

n 人房间，每节点维护 n-1 条直连 QUIC 链路，总唯一链路 = n(n-1)/2。媒体帧经每条链路以数据报发送，**无中心转发**。

| n | 链路总数 | 每节点上行流 |
|---|---------|------------|
| 2 | 1 | 1 |
| 4 | 6 | 3 |
| 6 | 15 | 5 |
| 8 | 28 | 7 |

**DHT 优化的精确作用域**（非媒体转发，而是三种正交角色，§12 详解）：
- **R-元数据索引**：`RoomID/GroupID → participants + addrs` 分布式查询
- **R-转发中继**：媒体经 DHT 节点中继（NAT 不通时降级，非默认路径）
- **R-路径优化**：基于 DHT XOR 距离选近邻，优化 Gossip 传播树 + 反向 ACK/事件补发路径

### 原则 ③：Group 仅传 GroupID + 成员校验签名 + 异常公示

**核心范式**（Group 控制平面）：
```
{GroupID} → {签署的成员事件 (UserID, action, sig)} → {本地 GroupID→UserIDs 表} → {逐消息验签}
```

- 群创建/增员**不再广播完整名册**，仅经身份信道投递 `(GroupID, UserID, action, sig)`
- 每条群消息**强制 Ed25519 签名**
- 接收方校验：(a) 发送者是否在本地成员表；(b) 签名有效性
- 异常 → **签名公示**（`SecurityNotice`），向群内全员广播警示

### 原则 ④：Room 近 Group 简化 + 数据报传输

| 维度 | Group（原则③） | Room（原则④） |
|------|---------------|--------------|
| 命名空间 | GroupID | RoomID（正交） |
| 成员传播 | 签名事件 | **同**（RoomID 替代 GroupID） |
| 逐消息验签 | 是（文本流） | **同**（数据报媒体帧带签名头） |
| 传输 | 可靠流 | **数据报**（免逐帧握手） |
| 生命周期 | 持久 + owner + role | **简化**：create/invite/join/leave/bye，**无 owner/role** |
| 拓扑 | fan-out（ContactRegistry 解析） | **全 mesh** + DHT 降级 |
| 参与者表 | 持久 DB | 内存，最后一人离开即 GC |

**统一签名验证管线**（③④共用）：
```
历史且已撤销的身份层（R1）：IdentityFrame 携带公钥 → ContactRegistry.public_key
       │
       ▼
成员事件层（R2/R3）：GroupMemberEvent / RoomMemberEvent {sig}
       │  接收方验 actor 签名 → 增量更新本地 UserIDs 表
       ▼
逐消息层（R2/R3）：
  Group（流）：GroupTextMessage{sig} → 验成员表+签名 → 入库/公示
  Room（数据报）：媒体帧{sig} → 验成员表+签名 → 渲染/公示
       │
       ▼
异常处置：SecurityNotice（签名公示）→ UiStore → UI 警示条
```

---

## 4. 历史 R1-R4 映射（非执行路线）

> 本节的图、阶段表和“必须最先”等措辞均为历史提案。它们已映射到 `ROADMAP.md` 的 M3-M7 或被明确冻结，不能作为独立 DAG、依赖或交付顺序。

```
R1（ID分离+公钥分发）  ← 必须最先，信任基础设施，无破坏性
  ├→ R2（群签名验证）   ← 修复 P0-1/2/3/4 安全/语义，独立可交付
  └→ R3（房间+数据报）  ← 数据报通路 + RoomMesh
        └→ R4（Mesh+DHT优化） ← 顶层优化，依赖 R3
```

| 阶段 | 目标 | 解决 | 依赖 | 风险 |
|------|------|------|------|------|
| **R1** | ID 三分离；身份信道扩展公钥 | P0 签名根因；为 R2/R3 铺信任基础 | 无 | 低（扩展字段，向后兼容） |
| **R2** | 群签名事件流 + 逐消息验签 + 安全公示 | P0-1/2/3/4 | R1 | 中（协议帧变更，需双写过渡） |
| **R3** | Room 模型 + 数据报通路 + 简化生命周期 | 群通话能力 | R1 | 中高（数据报接收通路新建） |
| **R4** | 分层拓扑 + CRDT 并发收敛 + DHT 三角色 | P1-5/6；大规模；NAT 容忍 | R3 | 高（分布式算法复杂） |

**关键原则**：每阶段独立可合并、可回滚、有测试。R1 不阻塞 R2/R3 的设计，但 R2/R3 的验签管线**依赖** R1 的公钥分发。

---

## 5. 历史 R1：ID 分离与公钥扩展（已由 v2 取代）

> **已撤销**：此节的 `IdentityFrame` 公钥扩展不是 v2 信任根，不得变更 schema 或生成物来实现它。v2 只接受 mTLS、credential evidence 和 `DeviceHello` 产生 principal。

### 5.1 三个正交命名空间

见 §3 原则①。代码层面：

`rust/core/src/data_ctl/convert/router.rs:56-68` 当前 `Target` 仅 `Single|Group`，扩展为三分支：

```rust
pub enum Target {
    Single(NodeRef),         // 1:1
    Group(GroupID),          // 群
    Room(RoomID),            // ★ 新增：房间 mesh fan-out
}
```

`Room` fan-out 走 mesh（R3/R4），与 `Group` fan-out 实现不同：Group 走 ContactRegistry 解析；Room 走 RoomMesh 直连表。

### 5.2 身份信道扩展公钥（信任锚）

**当前缺口**：`IdentityFrame`（`serializer.rs:175-181`）只有 `{userId, nickname, friendRequest}`，无公钥；`Contact.public_key`（`contacts.rs:59`）恒为 None。

**变更 `schema/message.capnp`**（扩展现有 struct）：

```capnp
struct IdentityFrame {
    userId        @0 :Text;
    nickname      @1 :Text;
    friendRequest @2 :Bool;
    publicKey     @3 :Data;       # ★ Ed25519 32B 公钥
}
```

**变更 `contacts.rs:274-344` `bind_identity`**：接收 `public_key` 参数，写入 `Contact.public_key`。

**变更 `ui_store.rs:822-861` `NodeKeypair`**：发送 `announce_identity`（`mod.rs:1224-1234`）时注入 `NodeKeypair.public_b64()`。身份握手 recv 任务（`mod.rs:726-766`）自动落库公钥。

> **此阶段是原则③④的信任基础设施**：无公钥分发，签名验证无从谈起。R1 必须最先做。

### 5.3 改动文件

| 文件 | 改动 |
|------|------|
| `schema/message.capnp` | `IdentityFrame` +`publicKey` |
| `data_ctl/convert/serializer.rs` | `IdentityFrame` +`public_key` 字段、serialize/deserialize |
| `data_ctl/contacts.rs` | `bind_identity` 接收公钥；`Contact.public_key` 落库 |
| `data_ctl/mod.rs:726-766` | 身份 recv 任务传递公钥 |
| `data_ctl/mod.rs:1224-1234` | 发送 `announce_identity` 注入公钥 |
| `data_ctl/convert/router.rs:56-68` | `Target` +`Room(RoomID)` 分支 |

---

## 6. 历史 R2：群组安全模型（非授权提案）

> **目标**：用签名成员事件流替代"整体覆盖名册"；每条群消息强制验签；异常签名公示。修复 P0-1/2/3/4。

### 6.1 签名成员事件（替代全量名册广播）

**当前缺陷**（`router.rs:1300-1334` `notify_group_members`）：每次操作序列化完整成员列表发 `GroupCreate`，接收方整体覆盖。

**新模型**：成员资格以逐条签名事件传播。

新增 capnp struct + MsgType：

```capnp
# 单条成员事件（仅传 GroupID + 单个 UserID，非完整名册）
struct GroupMemberEvent {
    groupId   @0 :Text;          # 仅 GroupID
    userId    @1 :Text;          # 该事件涉及的 UserID
    action    @2 :UInt8;         # 0=Join, 1=Leave, 2=Create(首批), 3=Delete
    actorUid  @3 :Text;          # 操作发起者
    epoch     @4 :UInt64;        # 成员表版本号
    signature @5 :Data;          # Ed25519(actor私钥) over (groupId||userId||action||actorUid||epoch)
}
```

`route.rs` `MsgType` +`GroupMemberEvent = 0x14`。

### 6.2 本地成员表 + 增量构建

`router.rs` `DataRouter` 新增逐群成员集（取代 `groups: HashMap<String, Vec<String>>`）：

```rust
struct GroupMembership {
    members: HashSet<UserID>,     // 本地视图（由签名事件增量构建）
    epoch: u64,
    owner_uid: UserID,            // 创建者（首批 Create 事件携带）
    deleted: bool,
    // 大群（R4）扩展：
    seen_events: LruCache<EventHash, ()>,  // Gossip 去重
    add_tags: HashMap<UserID, HashSet<Tag>>, // CRDT OR-Set（§10）
    tombstones: HashSet<Tag>,
    fanout: u8,                   // 拓扑扇出（按 N 自适应）
}
groups: Arc<RwLock<HashMap<GroupID, GroupMembership>>>,
```

**构建方式**：
- `create_group(name, members)`：本地建表，向每个成员发签名 `Create` 事件（仅 GroupID + 该成员自身 + sig）。**不广播名册**。
- `add_member`：向新成员发 `Join` 事件；向现有成员发该新成员的 `Join` 事件（增量告知"X 加入了"）。
- 接收方累积事件 → 增量更新本地 `members` 集。epoch 守卫防回放/乱序。

### 6.3 逐消息强制验签（安全闸）

**变更 `GroupTextMessage`**（`msg.rs:25-30` + capnp）：

```capnp
struct GroupTextMessage {
    groupId    @0 :Text;
    content    @1 :Text;
    messageId  @2 :Text;    # 去重
    senderUid  @3 :Text;
    signature  @4 :Data;    # Ed25519 over (groupId||content||messageId||senderUid)
}
```

**接收侧验证管线**（`group_text_recv_task`，`mod.rs:291-300` 改写）：

```rust
// 伪代码
let msg = deserialize_group_text(&frame.payload)?;
// ① 解析发送者 transport session → user_id
let sender_uid = resolve_session_to_user(&frame.from);
// ② 查本地 GroupID→UserIDs 表
if !local_group_members(group_id).contains(&sender_uid) {
    // ★ 关键设计（§10.3）：不立即丢弃，宽限缓存等事件追上
    pending_verification.push((msg, deadline=now+5s));
    request_membership_refresh(group_id, &sender_uid).await;
    return;
}
// ③ 验签（公钥来自 R1 身份握手）
let pk = contacts.get(&sender_uid).public_key?;
if !NodeKeypair::verify(&signed_bytes, &msg.signature, &pk) {
    trigger_security_alert(group_id, sender_uid, "invalid signature");
    return;
}
// ④ 去重（message_id LRU）
// ⑤ 通过 → 入 UiStore
```

### 6.4 异常 → 签名公示

`trigger_security_alert` 向群内广播签名公示帧：

```capnp
struct SecurityNotice {
    groupId     @0 :Text;
    offenderUid @1 :Text;
    reason      @2 :Text;
    witnessUid  @3 :Text;   # 公示者
    signature   @4 :Data;   # witness 用自己私钥签（防伪造预警）
    ts          @5 :Int64;
}
```

`MsgType::GroupSecurityNotice = 0x15`。所有成员收到后写入 `UiStore.security_notices` 域，UI 在群消息流顶部展示警示条。公示帧本身也签名，防止攻击者用假预警骚扰。

### 6.5 这如何根治 P0

| 旧缺陷 | R2 修复 |
|--------|---------|
| 删除=重建（P0-1） | `Delete` 是独立 action，非复用 Create |
| 并发覆盖无收敛（P0-2） | 事件流而非整体覆盖；epoch + 签名收敛 |
| 被踢者仍能发消息（P0-2 权限） | 验签管线②拦截非成员；伪造消息触发公示 |
| 群消息无标识/去重（P0-4） | `messageId` 字段 + LRU 去重 |
| 离线永久踢出（P0-3） | 移除 `purge_user_mappings` 对群组的破坏（见下） |

**P0-3 修复（R2 与 G2 共识）**：`dispatch_loop` 处理 `Disconnected` 时**只清 `user_to_session` 与 `peers`，不再触碰 `groups`**（`router.rs:521-528`、`:589-613`）。群成员表始终保留完整 user_id 列表。对齐 `contacts.rs:347-355` `unbind_session` 的"保留 contact、只清 session"范式。

### 6.6 改动文件

| 文件 | 改动 |
|------|------|
| `schema/message.capnp` | +`GroupMemberEvent`、扩展 `GroupTextMessage`、+`SecurityNotice` |
| `net/route.rs` | +`GroupMemberEvent(0x14)`、+`GroupSecurityNotice(0x15)` |
| `convert/serializer.rs` | 对应 serialize/deserialize |
| `convert/router.rs` | `GroupMembership`（HashSet+epoch+owner）；`apply_member_event`；废弃 `notify_group_members` 全量广播；移除 `purge_user_mappings` 对群组破坏 |
| `data_ctl/msg.rs` | `GroupTextMessage` +`message_id/sender_uid/signature`；`send` 时签名 |
| `data_ctl/mod.rs` | 群操作改发签名事件；recv 任务加验签管线；`trigger_security_alert` |
| `data_ctl/ui_store.rs` | +`security_notices` 域 + LRU 去重 |

---

## 7. 历史 R3：房间模型（非授权提案）

> **目标**：临时音视频聊天室；数据报媒体（免反复握手）；群消息流内邀请卡片；mesh 多 frame 展示。复用 R2 签名管线。

### 7.1 房间领域模型（新建 `room.rs`）

```rust
pub struct Room {
    pub room_id: RoomID,               // UUID，独立于任何 GroupID
    pub source_group: Option<GroupID>, // 可选：发起群（仅 UI 提示，非依赖）
    pub media: u8,                     // voice/video
    pub initiator: UserID,             // 发起人（无特权，仅记录）
    pub participants: HashSet<UserID>, // 当前在线
    pub epoch: u64,
    pub created_at_ms: i64,
}
```

房间事件帧（capnp，与 R2 `GroupMemberEvent` 同构，换字段名）：

```capnp
struct RoomMemberEvent {
    roomId      @0 :Text;
    userId      @1 :Text;
    action      @2 :UInt8;   # 0=Invite, 1=Join, 2=Leave, 3=Bye(销毁)
    actorUid    @3 :Text;
    epoch       @4 :UInt64;
    signature   @5 :Data;    # 同构验签
    mediaHint   @6 :UInt8;   # Invite 携带（voice/video）
    sourceGroup @7 :Text;    # 可选来源群（仅 Invite）
}
```

`MsgType::RoomMemberEvent = 0x16`（统一事件帧 + action 子类型，省码位）。

### 7.2 数据报媒体通路（点④核心）

**(a) 打通数据报接收转发**（`p2p_gateway.rs`）：
- `GatewayEvent` +`Datagram{id, node_ref, data: Bytes}` 变体
- `accept_incoming`（`p2p_gateway.rs:563-584`）每连接额外 spawn 数据报读循环（`channel.recv_datagram()`，`p2p.rs:1464` 已有）
- `dispatch_loop`（`router.rs:515`）+`Datagram` 分支，路由到媒体订阅者

**(b) Room 媒体走数据报**：新建 `RoomMediaCtl`（room.rs 内），`send_frame` 调 `router.send_datagram(Target::Room(room_id), frame)`。

**(c) 紧凑数据报帧头**（省带宽，不走完整 capnp Message 封装）：

```text
[1B media][16B room_token][16B sender_uid_token][4B seq][8B ts][sig(64B 或 HMAC 32B)][payload]
```
- `room_token`/`sender_uid_token`：16 字节定长 token（RoomID/UserID 哈希前缀），接收方查表还原
- 视频 > `MAX_DATAGRAM_SIZE`（65536B）分片，复用 seq+frag 思路

### 7.3 Room 生命周期（简化）

```
create: 发起人本地建 Room → 对每个被邀请者发 RoomInvite(签名事件，可靠流)
   ↓ 接收方 UiStore 渲染邀请卡片（KIND_ROOM_INVITE）
join: 用户点"加入" → 本地向房间所有已知 participants 发 RoomJoin → 建 mesh 链路
   ↓ 已有参与者收到 Join → 把新人加入本地 participants → 开始向其推媒体数据报
leave/bye: 挂断 → RoomLeave 广播 → 其他人移除其流 → 最后一人离开触发 Bye(GC)
```

**无 owner/role**（点④简化）：任何人可离开，无转让逻辑。跨群转发权 = "任何当前 participant 可 Invite 其联系人"。

### 7.4 邀请卡片（群消息流内呈现）

`UiStore.MessageRecord`（`ui_store.rs:62-90`）+`kind=KIND_ROOM_INVITE(4)` + `room_id` 字段。Flutter `msg_card.dart` 检测该 kind → 渲染可点击卡片（"🎬 视频通话 · 加入"），点击调 `joinRoom(room_id)`。卡片持久化（DB `room_invites` 表），支持晚加入者从历史卡片进入（房间已销毁则 join 返回 `RoomGone`）。

### 7.5 多 frame 展示（Flutter UI）

`call_screen.dart` 当前为单 `_RemoteSurface`（call_screen.md:36-44），重构为网格：

```dart
GridView.builder(
  gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount: participants.length <= 2 ? 1 : 2,
  ),
  itemCount: participants.length + 1, // +1 本地预览
  itemBuilder: (_, i) => _ParticipantTile(
    uid: ..., videoStream: ..., isLocal: i == localIndex, isSpeaking: ...,
  ),
)
```

`CallSnapshot`（FRB）从单 `peerAlias` 扩为 `List<RoomParticipant>`。`UiStore.CallState`（`ui_store.rs:127-141`）重构为携带 `participants: Vec<ParticipantState>`。

### 7.6 改动文件

| 文件 | 改动 |
|------|------|
| `net/p2p_gateway.rs` | +`GatewayEvent::Datagram` + 接收循环 |
| `net/route.rs` | +`RoomMemberEvent(0x16)` |
| `schema/message.capnp` | +`RoomMemberEvent` + 媒体帧 schema |
| **新 `data_ctl/room.rs`** | `Room` 模型 + `RoomMediaCtl` + 生命周期 + RoomMesh |
| `convert/router.rs` | +`Datagram` dispatch 分支 + `send_datagram` + `Target::Room` |
| `data_ctl/ui_store.rs` | +`KIND_ROOM_INVITE` + `room_id` + 多方 `CallState.participants` |
| `data_ctl/database.rs` | SCHEMA_VERSION+1 + `room_invites` 表 |
| `flutter/lib/ui/main/call/call_screen.dart` | 单 tile → GridView 网格 |
| `flutter/lib/ui/main/msg/msg_card.dart` | +房间邀请卡片渲染 |

---

## 8. 历史 R4：Mesh 拓扑与 DHT 优化（已冻结）

> 本节的 DHT、Gossip、mesh relay 和 NAT 回退提案不在当前路线；未来如有产品需要，必须在 M7 后作为外部组件重新决策，不能复用本节的 core 设计。

> **目标**：分层混合拓扑；CRDT 并发收敛；DHT 三角色（索引/中继/路径优化）；Room 媒体路径自适应。修复 P1-5/6，支撑大规模与 NAT 容忍。

### 8.1 分层混合拓扑（按规模自适应）

```
┌─────────────────────────────────────────────────────┐
│ Small Group (N ≤ 8)  → Full Mesh                     │
│   每人持 N-1 直连；签名事件可靠流直投                 │
├─────────────────────────────────────────────────────┤
│ Medium Group (8 < N ≤ 32) → Gossip (fan-out=4)       │
│   每节点转发给 4 个 DHT 近邻；事件带 messageId 去重   │
│   传播 O(log₄ N) 跳                                  │
├─────────────────────────────────────────────────────┤
│ Large Group (N > 32) → DHT Overlay + 中继            │
│   DHT 路由转发，不必全直连（§12）                     │
└─────────────────────────────────────────────────────┘
```

- **控制平面**（签名成员事件）：始终 Gossip 或 DHT 传播（低频，可容忍多跳）
- **数据平面**（群文本消息）：小群直投，大群沿 DHT/Gossip 路径分发

### 8.2 Room 全 mesh n(n-1)/2

n 人房间每节点维护 n-1 条直连 QUIC 链路（复用 `P2PGateway::connect`，`p2p_gateway.rs:405`）。**上限建议 8 人**（上行 7 路编码流，家庭宽带可承受）。超出 → UI 拒绝 + 提示。

### 8.3 DHT 三角色与自适应媒体路径

历史且已撤销：本节曾以 `Direct / ViaRelay / ViaDht` 自适应媒体路径；现行 core 不允许这种选择或回退。

### 8.4 改动文件

| 文件 | 改动 |
|------|------|
| `net/dht.rs` | +`publish_room/lookup_room`；+`store_event/serve_event`；+`query_relay`；接受外部 gateway |
| `net/route.rs` | 接线 `MeshAnnounce/CapabilityQuery/RelayOffer/ReputationUpdate`（已有码位 0xA0-0xA3） |
| `schema/message.capnp` | +`MeshAnnounce/RelayOffer/MediaFeedback` struct + 中继帧头 schema |
| `data_ctl/convert/router.rs` | `DataRouter` +`dht`；`Target::Room` 支持混合路径；gossip fan-out 用 DHT 距离 |
| `data_ctl/room.rs` | `RoomParticipant.path` 自适应；中继帧收发 |
| **新 `net/mesh_relay.rs`** | 中继转发逻辑 + 信誉聚合 |

---

## 9. 群组拓扑详解

### 9.1 当前拓扑：Sender-Centric Star（缺陷型）

```
        ┌─── S ───┐
        │  (发送者) │
   ┌────┼────────┼────┐
   ▼    ▼        ▼    ▼
   M1   M2       M3   M4
```

`router.rs:224-292` `Target::Group` 实现：发送者解析成员表 → 逐成员 `gateway.send`。

| 维度 | 当前行为 | 问题 |
|------|---------|------|
| 边数 | 发送者→每成员（星型） | 发送者必须是中心枢纽 |
| 离线成员 | 静默丢弃 | 消息永久丢失 |
| 流量 | 发送者承 O(N) 上行 | 大群带宽瓶颈 |
| 容错 | 发送者宕机则群瘫痪 | 单点失效 |

### 9.2 候选拓扑对比

| 拓扑 | 边数/人 | 延迟 | 容错 | 带宽/人 | 适用 |
|------|--------|------|------|---------|------|
| Star（当前） | 发送者 N-1，其余 0 | 低（1 跳） | 差 | 发送者 O(N) | 仅小群+全直连 |
| Gossip（epidemic） | 固定扇出 k | O(log N) 跳 | 强（多路径） | O(k) 常数 | 大群、容忍延迟 |
| Overlay Tree | 树形 | O(log N) 跳 | 中 | O(1)~O(m) | 流媒体分发 |
| Full Mesh | N-1 | 最低（1 跳） | 强 | O(N) 上行 | 小群、低延迟 |
| DHT Overlay | O(log N) 路由表 | O(log N) 跳 | 强 | O(1)~O(k) | 大群+中继 |

### 9.3 Gossip 事件传播（中群）

```rust
async fn gossip_event(&self, group_id: &str, event: &GroupMemberEvent) {
    let peers = self.pick_gossip_peers(group_id, self.fanout).await; // DHT 近邻 k 个
    for p in peers {
        router.send(Target::Single(p), MsgType::GroupMemberEvent, &encoded).await;
    }
    // 接收方：验签 → 应用 → 若首次见此事件 → 继续 gossip（TTL 衰减）
}
```

**TTL 衰减**：事件携带 `ttl`（初始 ⌈log₄ N⌉+1），每跳 -1，到 0 停止。配合 `seen_events` LRU 去重，保证传播覆盖且不洪泛。

---

## 10. 并发收敛机制（CRDT + 仲裁）

### 10.1 并发场景分类

| 场景 | 描述 | 严重度 |
|------|------|--------|
| 并行 Join（良性） | 多个被邀请者同时接受 | 低 |
| 并行 Add（操作冲突） | 多个 admin 同时 add 不同人 | 中 |
| Add vs Remove 竞态 | 同一用户被同时加和踢 | 高 |
| Epoch 分配竞态 | 两个节点都 bump 到同 epoch | 高 |
| 网络分区愈合 | 分区期间各自演化，愈合后视图分叉 | 高 |

### 10.2 分层一致性

**层1：成员增删 → CRDT OR-Set（Observed-Remove Set）**

```
ADD(uid, tag)      — tag = hash(actor_uid || uid || ts || nonce)，全局唯一
REMOVE(uid, tags)  — 必须引用要移除的具体 add-tag（tombstone by tag）
```

性质：
- 并发 ADD 不同 uid → 都保留（CRDT merge = union of adds minus removed tags）
- 并发 ADD 同 uid（不同 tag）→ 都保留（uid 在 set 中一次，但有两个 add-tag 追踪）
- REMOVE 仅能删已观察到的 tag → 不会误删并发新加的（OR 语义）
- 无冲突 merge → 任意两节点 `A.members ∪ B.members`，tombstone 集合取并

**层2：create/delete/owner 转让 → epoch + owner 签名仲裁**

破坏性单点操作，不能并发：
- create/delete/transfer 仅 owner 私钥签名有效。非 owner 发此类事件 → 丢弃 + 安全公示
- epoch 严格递增；乱序到达 → 缓冲等缺口或丢弃旧的
- owner 离线超时 → 群进入"只读"（仅 CRDT add/remove 生效）直到 owner 回归

**层3：Anti-Entropy（状态分歧检测与修复）**

节点间周期性交换成员表 Merkle 根：
```rust
fn membership_root(members: &HashSet<UserID>, tombstones: &HashSet<Tag>) -> Hash {
    hash(sorted(members) || sorted(tombstones))
}
```
Gossip 交换 `(group_id, membership_root)`：
- 根相同 → 无分歧，跳过
- 根不同 → 触发全量同步：交换完整 add-tag 集 + tombstone 集，CRDT merge

周期（如 30s）+ 事件驱动双重触发。Dynamo/Cassandra 用的 anti-entropy 模式。

### 10.3 "更新不及时"的具体缓解

| 症状 | 缓解 |
|------|------|
| Join 事件还在传播中，新成员已开始发言 | 验签管线②发现非成员 → **不丢弃，缓存 N 秒**等事件追上；超时再公示 |
| Remove 后被踢者仍发消息（旧视图） | 接收方持最新 tombstone → 验签失败 → 公示；被踢者消息被全网拒绝 |
| 分区愈合视图分叉 | Merkle 根检测 → CRDT merge 自动收敛 |
| epoch 跳跃（丢事件） | 检测 `epoch > local+1` → 请求 owner 补发缺失区间（或 DHT 就近 serve，§12.4） |

**关键设计**：验签失败**不立即丢弃**，而是"宽限缓存 + 反查"，因为可能是事件传播滞后而非真攻击。

---

## 11. GroupID 冲突仲裁

### 11.1 冲突类型分类

UUID v4 碰撞概率 ≈ 2⁻¹²²，物理碰撞可忽略。真正的冲突是逻辑/安全层面：

| 类型 | 描述 | 触发条件 |
|------|------|---------|
| A. 恶意仿冒 | 攻击者生成他人 GroupID 伪造成员事件 | 无 owner 签名验证 |
| B. ID 复用 | 节点重启/迁移后生成新群恰好复用旧 GroupID | 极罕见 |
| C. 状态分叉 | 同 GroupID 在不同节点有不同成员视图 | 分区愈合（§10 已解） |
| D. 跨群同名 | 两个独立群同名（不同 GroupID） | 非冲突，正常 |
| E. 并发创建同 ID | 两节点同时生成相同 UUID | 概率≈0，理论存在 |

### 11.2 不可变身份三元组

每个 Group 有不可变身份（创建时固定，签名背书）：

```rust
struct GroupIdentity {
    group_id: GroupID,           // UUID v4
    owner_uid: UserID,           // 创建者（不可变，除非 owner 转让）
    created_at_ms: i64,          // 创建时间戳（全局锚）
    owner_pubkey: Vec<u8>,       // owner 的 Ed25519 公钥（R1.3 分发）
    genesis_sig: Vec<u8>,        // owner 对上述字段的签名 = 群的"出生证"
}
```

### 11.3 历史仲裁规则（复合 GroupId 主张已撤销）

1. **已撤销的历史主张**：曾把 GroupID 唯一性定义为 `(group_id, owner_uid)`，并将 owner 用作 UI 区分。现行规则是单一、全局唯一的 `GroupId` 由 genesis 绑定；owner 变化不改变 ID。
2. **已撤销的历史主张**：曾在复合键内用创建时间仲裁。现行实现必须拒绝同一 `GroupId` 的不同 genesis，不能通过 owner 或时间戳派生第二个群。
3. **genesis_sig 是根信任**：所有后续成员事件链式签名回溯到 genesis。无法伪造历史。

### 11.4 冲突处理流程

```
节点 X 收到声称属于 Group G 的成员事件：
  ① 查本地是否有 G 的 GroupIdentity
     ├─ 无 → 首次见到：记录 genesis，信任（信任首次见到的 owner 签名）
     └─ 有 → 比对 genesis_sig
        ├─ 一致 → 同一群，正常处理
        └─ 不一致（同 group_id 不同 owner/genesis）
           → 类型 A 仿冒 或 B/E 真冲突
           → 仲裁：以本地已有 genesis 为准，拒绝冲突事件
           → 触发 SecurityNotice 公示
           → 上报 UI 警示用户决策
```

### 11.5 防 ID 复用（类型 B）

- `create_group` 时把 `(owner_uid, created_at_ms, group_id)` 一起签名作为 genesis
- GroupID 可编码 owner 前缀（如 `grp:<owner_uid_hash8>:<uuid>`）使跨 owner 冲突在 UI 可辨

### 11.6 owner 争议（无主仲裁）

- **禁止无主转让**：transfer_owner 必须由当前 owner 签名（链式）
- **owner 心跳超时**：群进入只读，等待 owner 回归
- **极端死锁**（owner 永久失联）：需预设 `succession_chain`（owner 预先签名的继承顺序），否则群只能冻结不可改——这是有意的安全权衡

---

## 12. DHT 三角色优化详解

### 12.1 关键澄清：DHT 的三种正交角色

| 角色 | 作用 | 受益对象 |
|------|------|---------|
| **R-元数据索引** | `RoomID/GroupID → participants + addrs` 分布式查询 | Group/Room 发现 |
| **R-转发中继** | 媒体/事件经 DHT 节点中继转发（非源直投） | Room 媒体（解 NAT 不互通） |
| **R-路径优化** | 基于 DHT 距离选近邻，优化传播树 + 反向路径 | Group 事件 Gossip |

### 12.2 现有基础与缺口

**已有**（可直接利用）：
- `DhtNode`（`dht.rs:37-162`）：Kademlia 风格路由表 + `broadcast`（`dht.rs:88-115`）
- `route.rs:88-92` 预留：`MeshAnnounce(0xA0)`、`CapabilityQuery(0xA1)`、`RelayOffer(0xA2)`、`ReputationUpdate(0xA3)` —— **为自组织中继/信誉设计，未接线**

**缺口**：
- `DhtNode` 独立 `P2PGateway`（`dht.rs:47`），未与 `DataRouter` 共享连接池
- 无中继能力声明
- 无信誉/负载追踪

### 12.3 历史 R-转发：DHT 中继模型（已撤销）

> 这一整节只记录被否决的方案。core 不实现、协商或回退到 DHT relay；任何未来 relay 只能在 M7 后作为 `relay -> core` 的外部组件重新设计。

```
全 mesh（N=4，6 边，需全直连）：
  S ── M1
  │ ╲ ╱
  M2 ─ M3

DHT 中继转发（N=4，中继 R 承担分发）：
  S ──→ R ──┬──→ M1
            ├──→ M2
            └──→ M3
  边数：4（S→R, R→M1, R→M2, R→M3），非 6
```

**中继节点选择**（利用预留 MsgType）：

```capnp
struct MeshAnnounce {
    nodeId      @0 :Text;
    capabilities @1 :UInt16;  # bit0=可中继, bit1=有公网IP, bit2=高带宽
    capacity    @2 :UInt16;   # 当前可承中继流数
    reputation  @3 :UInt32;   # 信誉分
}
```

Room join 时，若检测到部分对不直连：
1. `CapabilityQuery(0xA1)` 查询附近 DHT 节点能力
2. 选高信誉 + 有余量 + 拓扑近的节点作中继（`RelayOffer(0xA2)` 协商）
3. 源把媒体数据报发给中继，中继 fan-out 给实际接收者

**中继转发协议**（紧凑，数据报承载）：
```text
[1B flags][16B room_token][16B src_uid][16B via_relay_uid][4B seq][8B ts][sig][payload]
```
中继节点收到 `via_relay_uid == self` → 查 room_token 订阅者 → fan-out。**只转发不解析 payload**（端到端加密则中继不可见内容）。

**信誉防滥用**（`ReputationUpdate`, 0xA3）：每次中继后接收方发 `ReputationUpdate{relay_uid, delta:+/-1, evidence}`，信誉低于阈值不再选中继，DHT 存储聚合值。

### 12.4 角色R-路径优化：DHT 距离感知传播 + 反向路径

**(a) Gossip 近邻选择**（XOR metric）：
```rust
async fn pick_gossip_peers(&self, group_id: &str, k: u8) -> Vec<NodeRef> {
    let members = self.group_members(group_id).await;
    let my_pos = dht_key(self.local_uid);
    members.sort_by_key(|uid| dht_key(uid) ^ my_pos); // 拓扑近 = 延迟低
    members.into_iter().take(k as usize).map(resolve_to_node).collect()
}
```

**(b) 反向传播路径**（两层含义）：

1. **ACK/NACK 与拥塞反馈回程**：媒体单向数据报，但接收方需反馈丢包率/RTT 供降码率：
```rust
struct MediaFeedback { room_token, src_uid, recv_rate, loss_rate, rtt_ms, sig }
```
走 DHT 路由回源，不必反建专门链路。

2. **事件回溯/补发**：节点缺失事件（epoch 跳跃）→ 向 DHT 中最近持有者请求补发：
```rust
// DHT 存储事件副本：key = hash(group_id, epoch_range)
// 任意节点可 serve 补发请求（DHT get）
request_missing_events(group_id, from_epoch, to_epoch) → DHT lookup → nearest holder serves
```
减轻 owner 单点压力，这是 DHT 的天然优势。

### 12.5 Room 媒体路径自适应

```rust
enum RoomMediaPath {
    Direct(NodeRef),              // 直连（mesh 优先）
    ViaRelay { relay: NodeRef },  // DHT 中继（NAT 不通时）
    ViaDht,                       // 纯 DHT 转发（移动端省电/弱链路）
}

struct RoomParticipant {
    uid: UserID,
    path: RoomMediaPath,         // 按链路质量动态切换
    last_rtt_ms: u32,
    loss_rate: f32,
}
```

**切换逻辑**（每 5s 重评估）：
- 直连 RTT < 100ms 且丢包 < 2% → `Direct`
- 直连失败或 RTT > 300ms → `ViaRelay`
- 终端弱（移动网络）→ `ViaDht`

使 Room 在小群+好网络=全 mesh 低延迟，大群/弱网=DHT 中继可用之间平滑过渡。

### 12.6 集成 DhtNode 到 DataRouter

```rust
pub struct DataRouter {
    gateway: Arc<P2PGateway>,
    dht: Option<Arc<DhtNode>>,     // ★ 新增，共享 gateway
    groups: Arc<RwLock<HashMap<GroupID, GroupMembership>>>,
    rooms: Arc<RwLock<HashMap<RoomID, Room>>>,
}
```

`DhtNode::new` 改为可接受外部 gateway，或 `DataRouter::bind`（`router.rs:143`）内部孵化 DHT 共享 endpoint。

---

## 13. 历史不变量总表（非规范）

> GR-2、GR-7、GR-10 及所有依赖 `IdentityFrame`、复合 GroupId 或 relay 的条目已经撤销；其余条目也必须由 M2/M6/M7 的现行规范重新裁定后才可实现。

| ID | 不变量 | 守护机制 |
|----|--------|---------|
| **GR-1** | UserID / GroupID / RoomID 三个正交命名空间，互不依赖 | `Target` 三分支；RoomID 不编码 GroupID |
| **GR-2** | **历史且已撤销**：每个 Contact 在身份握手后拥有非空 `public_key`（Ed25519 32B） | `IdentityFrame.publicKey`；`bind_identity` 落库 |
| **GR-3** | 每条群消息携带有效 Ed25519 签名，发送者在本地成员表内 | `GroupTextMessage.signature`；验签管线 |
| **GR-4** | 成员资格变更以签名事件传播，非整体名册覆盖 | `GroupMemberEvent`；废弃 `notify_group_members` 全量广播 |
| **GR-5** | 断线不修改群成员表（仅清 session 映射） | `purge_user_mappings` 不触碰 groups |
| **GR-6** | 群消息可去重（messageId 唯一） | `GroupTextMessage.messageId` + LRU |
| **GR-7** | **历史且已撤销**：GroupID 唯一性 = (group_id, owner_uid) 组合 | `GroupIdentity` genesis_sig 仲裁 |
| **GR-8** | Room 媒体走数据报（非逐帧 open_stream） | `RoomMediaCtl.send_datagram` |
| **GR-9** | Room 无 owner/role（纯对等，最后一人离开即 GC） | 无 transfer_owner；内存态 participants |
| **GR-10** | 中继节点只转发不解析端到端密文 | ECDH 房间密钥；中继帧 via_relay_uid 路由 |
| **GR-11** | 验签失败宽限缓存（区分"传播滞后"与"真攻击"） | `pending_verification` + 反查 |
| **GR-12** | CRDT 成员表 merge 无冲突（OR-Set 语义） | add-tag/tombstone |

---

## 14. 历史验证矩阵（非现行验收）

> 这些命令和用例不是当前测试门，尤其不能用于验证已撤销的 `IdentityFrame`、降级、DHT relay 或 Room 路线。现行验收以 `agents_work/TEST_MATRIX.md` 和 `ROADMAP.md` 的 M 阶段 gate 为准。

| 验证项 | 方法 | 关联不变量 |
|--------|------|-----------|
| IdentityFrame 公钥往返 | serializer 单元测试 | GR-2 |
| bind_identity 落库公钥 | contacts 单元测试 | GR-2 |
| GroupMemberEvent 签名往返 | serializer + 验签单元测试 | GR-3/4 |
| 成员表增量构建 | router 单元测试（create→join→leave 序列） | GR-4 |
| 并发 ADD 收敛 | CRDT 单元测试（两节点并发 add 不同 uid → merge） | GR-12 |
| 并发 ADD/REMOVE 不误删 | CRDT 单元测试（OR 语义） | GR-12 |
| epoch 守卫防回放 | router 单元测试（旧 epoch 事件被拒） | GR-4 |
| 验签失败宽限缓存 | mod.rs 集成测试（滞后事件场景） | GR-11 |
| 离线不踢出群组 | debug-harness/grouptest 集成测试 | GR-5 |
| GroupID 冲突仲裁 | router 单元测试（同 ID 不同 genesis → 公示） | GR-7 |
| 数据报媒体往返 | room.rs 单元测试（send_datagram→recv） | GR-8 |
| Room 邀请卡片渲染 | Flutter widget 测试 | — |
| Room 多 frame 展示 | Flutter widget 测试（GridView） | — |
| DHT 中继转发 | mesh_relay.rs 单元测试 | GR-10 |
| MediaFeedback 回程 | room.rs 集成测试 | — |
| Anti-entropy 分叉修复 | router 集成测试（两节点 Merkle 根不同 → merge） | GR-12 |

**命令**：
```bash
cargo test -p modcpt_core --lib data_ctl    # R1/R2 单元
cargo test -p modcpt_core --lib convert     # 序列化往返
cargo test -p modcpt_core --lib data_ctl::room  # R3 房间
cargo clippy --all-targets                  # 全量 lint
```

---

## 15. 风险与回滚

| 风险 | 缓解 |
|------|------|
| 老节点发 `0x12 GroupCreate`，新节点发 `0x14 GroupMemberEvent`，混合网络不互通 | R2 保留 `GroupCreate` 解析：新节点收到 `0x12` 时按 epoch=0 的成员事件处理；过渡期双发 |
| 身份握手公钥字段导致老节点解析失败 | **已撤销**：不得扩展 `IdentityFrame.publicKey`，也不得因字段缺失降级为“无签名验证”；旧 identity 类型应按 v2 规则拒绝。 |
| epoch 冲突（两节点并发 bump 同群） | LWW 取较大 epoch；owner 守卫后只有 owner 能 bump，从源头消除并发 |
| CRDT tombstone 膨胀 | 周期性 owner 签名"compact"快照，重置 tag 集（附录 C-1） |
| DHT 中继可见明文 | ECDH 房间对称密钥，中继只见密文（附录 C-2） |
| 信誉女巫攻击 | PoW/押金/已有信任网加权（附录 C-3） |
| DB 迁移失败 | `init()` 用 `ALTER TABLE`（SQLite 支持），`SCHEMA_VERSION` 增量；已有容错（`database.rs:127-129`） |
| 数据报接收通路新建引入死锁 | dispatch_loop `Datagram` 分支用 `try_send`（与现有 Data 分支一致），慢消费者丢帧不阻塞 |

**回滚策略**：每阶段独立 feature 分支。R1（公钥扩展）向后兼容，可随时回退。R2（签名事件）保留 `GroupCreate` 双解析，回退仅需关闭新事件发送。R3/R4（数据报/DHT）独立模块，可整体禁用而不影响群文本。

---

## 附录 A：文件改动总览

### R1（ID 分离 + 公钥）

| 文件 | 改动 |
|------|------|
| `schema/message.capnp` | `IdentityFrame` +`publicKey` |
| `data_ctl/convert/serializer.rs` | `IdentityFrame` +`public_key` |
| `data_ctl/contacts.rs` | `bind_identity` 接收公钥；`Contact.public_key` 落库 |
| `data_ctl/mod.rs:726-766` | 身份 recv 任务传递公钥 |
| `data_ctl/mod.rs:1224-1234` | `announce_identity` 注入公钥 |
| `data_ctl/convert/router.rs:56-68` | `Target` +`Room(RoomID)` |

### R2（群组安全模型）

| 文件 | 改动 |
|------|------|
| `schema/message.capnp` | +`GroupMemberEvent`、扩展 `GroupTextMessage`、+`SecurityNotice` |
| `net/route.rs` | +`GroupMemberEvent(0x14)`、+`GroupSecurityNotice(0x15)` |
| `convert/serializer.rs` | 对应 serialize/deserialize |
| `convert/router.rs` | `GroupMembership`；`apply_member_event`；废弃全量广播；移除 purge 群组破坏 |
| `data_ctl/msg.rs` | `GroupTextMessage` +签名字段；`send` 时签名 |
| `data_ctl/mod.rs` | 群操作改发签名事件；recv 验签管线；`trigger_security_alert` |
| `data_ctl/ui_store.rs` | +`security_notices` + LRU 去重 |
| `data_ctl/database.rs` | SCHEMA_VERSION+1，迁移 |

### R3（房间 + 数据报）

| 文件 | 改动 |
|------|------|
| `net/p2p_gateway.rs` | +`GatewayEvent::Datagram` + 接收循环 |
| `net/route.rs` | +`RoomMemberEvent(0x16)` |
| `schema/message.capnp` | +`RoomMemberEvent` + 媒体帧 schema |
| **新 `data_ctl/room.rs`** | `Room` + `RoomMediaCtl` + 生命周期 + RoomMesh |
| `convert/router.rs` | +`Datagram` dispatch + `send_datagram` + `Target::Room` fan-out |
| `data_ctl/voice.rs` / `video.rs` | Room 媒体改数据报路径 |
| `data_ctl/ui_store.rs` | +`KIND_ROOM_INVITE` + 多方 `CallState.participants` |
| `data_ctl/database.rs` | +`room_invites` 表 |
| `flutter/rust/src/api/` | +`calls.rs`（joinRoom/leaveRoom）或扩展 ui.rs |
| `flutter/lib/ui/main/call/call_screen.dart` | 单 tile → GridView |
| `flutter/lib/ui/main/msg/msg_card.dart` | +邀请卡片 |

### R4（Mesh + DHT 优化）

| 文件 | 改动 |
|------|------|
| `net/dht.rs` | +`publish_room/lookup_room`；+`store_event/serve_event`；+`query_relay`；接受外部 gateway |
| `net/route.rs` | 接线 `MeshAnnounce/CapabilityQuery/RelayOffer/ReputationUpdate` |
| `schema/message.capnp` | +`MeshAnnounce/RelayOffer/MediaFeedback` + 中继帧头 |
| `data_ctl/convert/router.rs` | +`dht`；`Target::Room` 混合路径；gossip DHT 距离 |
| `data_ctl/room.rs` | `RoomParticipant.path` 自适应 |
| **新 `net/mesh_relay.rs`** | 中继转发 + 信誉聚合 |

### 文档同步（每阶段）

| 文档 | 改动 |
|------|------|
| `Knowledge/rust/core/data_ctl/room.md` | **新增**（R3） |
| `Knowledge/rust/core/net/mesh_relay.md` | **新增**（R4） |
| `Knowledge/rust/core/data_ctl/convert/router.md` | `GroupMembership`、CRDT、Target::Room |
| `Knowledge/rust/core/data_ctl/msg.md` | 签名字段 |
| `Knowledge/rust/core/data_ctl/contacts.md` | public_key 落库 |
| `Knowledge/flutter/ui/main/call/call_screen.md` | GridView 多 frame |
| `Knowledge/logs/` | 每阶段新增日志 |
| `Knowledge/ROADMAP.md` 与 `Knowledge/STRUCTURE.md` | 更新路线图和结构状态 |

---

## 附录 B：待决策阻塞项

以下需明确决策后方可进入对应阶段的编码实施：

| # | 决策项 | 影响阶段 | 倾向方案 |
|---|--------|---------|---------|
| B-1 | 签名粒度：逐帧签名 vs 会话密钥 HMAC | R3 带宽/安全 | 会话密钥 ECDH + HMAC-SHA256 + 每 N 帧签一次 |
| B-2 | mesh 上限与超限行为 | R3/R4 | 8 人硬上限；超限拒绝 + 提示；N>8 未来转 SFU |
| B-3 | NAT 失败对处理 | R4 | 方案 A（降级文字提示）+ 方案 C（DHT 标记） |
| B-4 | 跨群转发权限严格度 | R3 | 当前 participant 可转发（足够），邀请卡标注 via_group |
| B-5 | owner 死锁的 succession_chain 是否实现 | R2 | 首版仅冻结；succession 作未来扩展 |
| B-6 | 音频混音实现方式 | R3 | 待定（自研 vs 平台 API）—— 见附录 C-5 |
| B-7 | 编解码器协商协议 | R3 | RoomInvite 携带 codec/bitrate |
| B-8 | 拥塞控制/自适应码率 | R4 | 应用层 RTT/丢包探测 + 降码率 |

---

## 附录 C：未覆盖的设计空白

以下问题在本方案中被识别但**未深入设计**，需在对应阶段实施前补完：

1. **CRDT tombstone 膨胀与 GC**：OR-Set 的 remove-tag 永久累积。大群长期运行需周期性 owner 签名"compact"快照重置 tag 集。
2. **历史且已撤销的 DHT 中继端到端加密设想**：若中继节点可见 payload 则有隐私风险。Room join 时 ECDH 协商房间对称密钥，中继只见密文。
3. **信誉女巫攻击**：攻击者注册大量节点刷信誉。需 PoW/押金/已有信任网加权，非纯计数。
4. **DHT 分区（sybil/churn）**：高 churn 下 DHT 路由不稳。需冗余路由（parallel lookup）+ 本地缓存。
5. **音频混音**：mesh 中每人收 N-1 路语音流，客户端必须做 PCM 混音。当前无任何混音/解码基础设施（媒体是 placeholder）。这是 Room 能否落地的最大技术阻塞。
6. **编解码器协商**：mesh 要求所有参与者用相同编解码器（Opus/H.264）与码率。需 codec 协商协议。
7. **Active Speaker 高亮**：多 frame 展示需高亮当前说话者，需音频帧加 RMS 标记或独立 `RoomSpeaking` 数据报。
8. **Room 晚加入者**：某人 30s 后加入，靠邀请卡片（持久化）入口。需处理房间已销毁时卡片失效（join 返回 `RoomGone`）。
9. **拥塞控制**：QUIC 数据报不被 ACK，无内置拥塞反馈。需应用层 RTT/丢包探测 + 自适应降码率，否则 mesh 拥塞时雪崩。
10. **录音/录屏隐私同意**：群通话录音涉及多方同意，属法律合规。建议加 `RoomRecording` 信令广播"正在录制"。
11. **回声消除/设备管理**：客户端侧 AEC，群通话下多路回声叠加更关键。
12. **跨 DHT 域**：若群成员分属不同 DHT 域（不同引导节点），需域间网关或统一 key space。当前未设计。
13. **Anti-entropy 风暴**：N 节点同时检测分叉 → 同时全量同步 → 洪泛。需随机退避 + 分批同步。

---

> **本文是历史设计证据，不是日后实施依据。** 保留它不代表采纳其中任何 R 标签、线路、schema、relay 或安全结论；现行工作只遵循 `ROADMAP.md` 和上位 v2/M2 规范。
