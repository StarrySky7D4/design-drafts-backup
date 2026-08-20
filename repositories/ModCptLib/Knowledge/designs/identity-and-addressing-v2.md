# 身份与寻址 v2

> **状态**：权威目标设计，2026-07-26；实施状态更新至 2026-08-04。它取代旧设计中以 `user_id + SessionId + 单 credential`、`IdentityFrame` 自报身份、`(group_id, owner_uid)` 唯一性和 core relay 回退为前提的部分。各历史 ID 标签只按下表判断；未完成的联系人/FRB/attachment/M5/M6 范围不得被已完成的可信会话或 direct-text 能力代替。

## 目标和边界

身份与传输分为两条正交轴：

```text
RealmId -> UserId -> DeviceId -> InstanceId -> TransportSessionId -> LinkId
ConversationId / MessageId / TransferId / EventId / GroupId / RoomId / RequestId
```

- `RealmId` 绑定 assertion signer；`UserId` 是账号主体；`DeviceId` 是不可克隆的安装实体；`InstanceId` 是进程生命周期；`TransportSessionId` 与 `LinkId` 只代表临时传输状态。
- core 不实现 relay、STUN、CID relay 或 NAT fallback。发现仅返回候选地址，不能承诺在线或可达。
- 所有签名覆盖共享 canonical encoder 输出，绝不签 JSON、Cap'n Proto bytes 或字符串拼接。
- Rust 内部 API 使用共享 ID newtype；JSON、FRB、SQLite 与 Cap'n Proto 边界必须显式 parse/encode。

## ID 和编码

| 范畴 | 类型 | 编码/生成 |
|---|---|---|
| 持久实体 | `UserId`、`DeviceId`、`CredentialId`、`GroupId`、`RoomId`、`MessageId`、`EventId`、`TransferId` | UUIDv7，规范小写文本 |
| 临时实体 | `InstanceId`、`TransportSessionId`、`LinkId`、`RequestId` | UUIDv4，规范小写文本 |
| hash 实体 | `RealmId`、`KeyId`、`TransportKeyId`、`DirectId` | 固定 32 bytes / 64 个小写 hex |

`RealmId` 是 assertion signer public key 的 SHA-256。`KeyId`、`TransportKeyId` 与 `DirectId` 使用不同的长度前缀 hash domain；`DirectId` 计算时排序两个不同 `UserId`，因此对称且拒绝 self。UUID version、非规范 UUID 文本与非小写 hash 文本均须拒绝。

## 可信闭环

1. 账号服务生成 `UserId`，调用方不能选择它；handle 只用于展示/搜索。
2. 新安装通过 challenge-bound PKCS#10 CSR 和 Ed25519 proof provision 新 `DeviceId`，私钥永不上传。
3. realm CA 验证 TLS 链，独立 assertion signer 签发绑定 TLS SPKI hash 的 device credential。
4. 新 ALPN/major version 的 `DeviceHello` 在 BIND 前验证 credential assertion、status proof、SPKI、`InstanceId` 与 lease epoch，输出不可变 `SessionBinding` / `PeerPrincipal`。
5. `IdentityFrame` 不再授权身份；旧 identity MsgType 必须返回 `UnsupportedProtocol`，不得按帧猜测版本降级。

## 会话、presence 与投递

- 一个 `DeviceId` 同时仅一个 active `InstanceId`。`begin_instance` 原子替换 lease epoch；presence 90 秒、每 30 秒续租，首版最多 8 个 Host QUIC candidates。
- `SessionRegistry` 是唯一会话事实源，保存 `DeviceId -> InstanceId -> primary TransportSessionId -> active/standby LinkId`。断链只移除链路/会话，绝不删除联系人、设备或群成员。
- primary 仲裁：assertion sequence 降序、规范发起者、`TransportSessionId` 升序。网络 I/O 必须在状态锁外；淘汰候选主动调用 `P2PChannel::close()`。
- `DeliveryPlanner` 冻结目标 `DeviceId` 集；`TransportRouter` 只从该集路由到 primary session。消息用 `(RealmId, MessageId)` 去重；文件用 `(RealmId, TransferId)` 重组并固定 authenticated origin device/conversation。

## 发布与数据根

ID-2 是唯一破坏性发布：显式 `init-realm`/`cutover-v2` 完整归档 v1 bundle，再原子切换空 v2 realm、manifest、CA、account TLS 与数据库。正常 `serve` 不生成或迁移；旧 RPC 返回 `upgrade_required`。客户端归档旧身份数据后必须由管理员带外导入并验证 `ServerProfileV2`，禁止 TOFU。

## 分阶段状态

| 阶段 | 目标 | 状态 |
|---|---|---|
| ID-1 | 共享强类型 ID，传输会话/链路单一身份 | **进行中**：`modcpt_identity::ids` 已提供全套 UUID/hash newtype；P2P、Room、安全群组和其 SQLite 验证态已迁为 typed ID，BIND/watcher/standby 同一 LinkId；联系人、路由、消息、文件和公开 FRB 的业务 String ID 尚待迁移 |
| ID-2 | `modcpt_pki`、manifest、v2 store/RPC、provision、显式 cutover | **已完成（M3）**：M3-01 至 M3-04 提供 PKI、RealmManifest、server/client v2 stores 与 provision；M3-07 接线 8422 mTLS endpoint、client certificate 签发、显式 init/cutover/rollback 和 v1 archive。 |
| ID-3 | 新 ALPN、DeviceHello、PeerPrincipal、BIND 前闭环 | **已完成（M3）**：M3-05 提供 canonical DeviceHello/credential；M3-06 接线 `/3` mTLS quarantine、BIND 前验证、replay guard、immutable PeerPrincipal/SessionBinding 与独立 SessionRegistry。presence lease/TTL/仲裁与单 active instance 属于后续阶段。 |
| ID-4 | IdentityStore、PresenceDirectory、SessionRegistry | **部分完成**：M3-06 已提供独立 SessionRegistry；IdentityStore、PresenceDirectory、`begin_instance` 和签名 PresenceLease 属于后续多设备/presence 工作。 |
| ID-5 | DeliveryPlanner、TransportRouter、immutable envelope | **部分完成/review**：M4-01 已提供 canonical immutable Envelope；M4-02 已实现 authoritative verified-session snapshot、deterministic primary 与 authenticated Envelope ingress/egress；M4-03 已实现 per-device receipt/outbox/inbox；M4-05 已实现 per-recipient native Olm v2 direct wire。review/CI 与最终汇合仍待记录。 |
| ID-6 | Conversation/Message/Transfer/Event 业务轴和 local-v2 schema | **部分完成**：ConversationId/MessageId 已贯通 M4 Envelope、SQLite 与 direct text；TransferId 有 bounded tracker foundation。EventId/RequestId、attachment runtime、UiStore/FRB 全面迁移与 M5 local-v2 数据生命周期仍未完成。 |
| ID-7 | Flutter 设备治理、群组验签与端到端负面矩阵 | 未开始 |

## 当前迁移约束

- 旧 `ContactRegistry` 的 `String user_id`、`IdentityFrame`、`DataRouter` user/session fallback、FRB raw-session API、按 session 文件重组和 UiStore 的 `conversation + is_group` 仍存在，仅可作为 v1 兼容代码；不得把它们接到 v2 授权决策。
- 新生产 feature 不应建立在旧群组/房间文档的 IdentityFrame 公钥分发、owner-in-ID 或 core relay 设想上。
- 每一源码修改同步逐文件 Knowledge、索引与日期日志；协议/schema/FRB 变更另遵循 snapshot/codegen 流程。
