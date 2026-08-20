# 身份与寻址 v2：ID 映射、实施审计与验证

> **状态**：权威身份模型的实施审计和历史 ID 标签映射，2026-07-26，2026-07-29 收敛，实施证据更新至 2026-08-04。架构模型见 [身份与寻址 v2](identity-and-addressing-v2.md)。本页不定义独立执行顺序、任务状态或阶段 gate；唯一可执行路线是 [`ROADMAP.md`](../ROADMAP.md) 的 M0-M7。
>
> `ID-1` 至 `ID-7` 只保留为历史检索和设计范围标签。它们不能授权实现，也不能覆盖 `agents_work/BOARD.md` 的领取状态、M0-M7 的依赖或安全边界。

## 当前完成度

历史百分比估算已废止；当前只按源码、任务 gate 与下表逐项判断。ID-2/ID-3 已由 M3 完成，ID-5/ID-6 已有 M4 foundation/direct-text 实现，但 review/CI/汇合状态和未完成子域必须分别保留。

| 历史标签 | 审计状态 | ROADMAP 映射 | 当前证据 |
|---|---|---|---|
| ID-1 | 进行中 | M0、M3、M4 | `rust/identity/src/ids.rs` 已提供共享 UUID/hash ID；`TransportSessionId`/`LinkId`、P2P、Room、安全群组和部分 SQLite 验证态已开始迁移。联系人、路由、消息、文件、UiStore、server/client 和公开 FRB 仍有业务裸字符串。 |
| ID-2 | 已完成（M3） | M3 | `e933ba1` 提供 `modcpt_pki`；`6631c2c` 提供 RealmManifest/ServerProfileV2；`5b0be30`、`241d364` 提供独立 server v2 store/RPC/provision；`02af3b6` 提供 client local-v2；`2d538ad` 接线 v2 mTLS server endpoint、client certificate 签发、init/cutover/rollback 与 v1 archive。 |
| ID-3 | 已完成（M3） | M3 | `02af3b6` 提供 DeviceHello/credential canonical contract；`fdc1083` 接线 `/3` mTLS quarantine、BIND 前验证、replay guard 与 SessionRegistry；`2d538ad` 提供 v2 server endpoint。presence lease/TTL/仲裁属于后续阶段。 |
| ID-4 | 部分完成 | M3 | `fdc1083` 提供独立 SessionRegistry；IdentityStore、PresenceDirectory、`begin_instance` 和签名 PresenceLease 仍属后续多设备/presence 工作。 |
| ID-5 | 部分完成/review | M4 | M4-01 immutable Envelope；M4-02 authoritative snapshot/planner/router；M4-03 per-device receipt/outbox/inbox；M4-05 per-recipient direct ciphertext。仍待 review/CI/汇合及 attachment runtime。 |
| ID-6 | 部分完成 | M4 | ConversationId/MessageId 已进入 Envelope/SQLite/direct；TransferId 有 bounded tracker。EventId/RequestId、attachment wire、UiStore/FileCtl/FRB 全面 typed migration 未完成。 |
| ID-7 | 未开始 | M5、M6 | 未发现设备治理 UI、credential 轮换/撤销闭环或 v2 多设备攻击性端到端矩阵。 |

### 已完成的 ID-1 范围

- 新增 `modcpt_identity::ids` 作为共享 ID 表示、解析与编码的起点。
- P2P 开始使用 `TransportSessionId` 与统一 `LinkId`；Room、安全群组和部分持久状态开始使用 typed ID。
- 设计、逐文件 Knowledge、索引、路线图和日期日志已同步。

### 尚未解除的 v1 风险

- `IdentityFrame { user_id, nickname, friend_request }` 仍能驱动身份绑定，不能作为 v2 授权依据。
- `ContactRegistry` 仍使用 `String user_id`，并将多 session 作为重复处理。
- `DataRouter` 仍存在 `user_to_session`、contacts 和直接 session 字符串等多路径解析/兜底。
- `FileCtl` 仍按发送 session 字符串重组；`UiStore`/SQLite 仍使用 `conversation:String + is_group` 与本地消息序号。
- v1 server/address-book 路径仍是用户级兼容模型；v2 server 已有多 DeviceId、credential/provision/revoke 与 content prekey，但完整 presence lease、设备治理 UI 和旧路径删除仍未完成。

## 不变量

- 主体链固定为 `RealmId -> UserId -> DeviceId -> InstanceId -> TransportSessionId -> LinkId`；业务对象绝不以 SessionId 为持久键。
- `RealmId` 是 assertion signer hash；持久业务实体使用 UUIDv7，临时运行/传输实体使用 UUIDv4，hash ID 使用域隔离 SHA-256。
- UserId、DeviceId、SessionId 等必须是共享 Rust newtype；IP、昵称、handle、DER、token 都不是主体 ID。
- DeviceCredential 是 DeviceId 的证明材料，不是主体层；TLS 与应用签名密钥通过 credential 绑定但保持分离。
- PeerPrincipal 只能由 mTLS、DeviceHello、credential assertion/status proof 产生；消息载荷绝不可创建或改变 principal。
- 断链只删除 link/session，永不删除联系人、设备或群成员。
- GroupId 单独全局唯一，genesis 不同即拒绝；owner 是状态而非 ID 组成部分。

## 历史 ID 范围到 M0-M7 的映射

以下内容保留原 ID 审计的交付边界，**不是执行顺序**。实现工作必须先在 `ROADMAP.md` 对应的 M 阶段获得授权并满足其 gate。

### 历史 ID-1：统一强类型 ID（M0、M3、M4）

1. 完成 `modcpt_identity::ids` 在 core、server、client、msg_api、FRB 非生成代码的迁移。
2. 删除 `room.rs` 私有 ID 宏与 `secure_group.rs` 裸别名；所有 JSON/SQLite/Cap'n Proto/FRB 边界立即 parse。
3. 完成 `SessionId -> TransportSessionId` 和 `LinkId` 单一身份迁移，删除平行连接 ID。

验收：无业务层 `type UserId = String`/`type GroupId = String`；无公开 session 字符串 API；ID 类型不可互换；UUID/hash 格式和域隔离测试通过。

### 历史 ID-2：v2 数据根、PKI、账号与设备（M3）

子阶段仅在全部完成后执行一次 cutover：

1. ID-2A：建立无网络/数据库依赖的 `modcpt_pki`，从 core 提取 CA、PKCS#10 CSR、SPKI 和证书策略；依赖方向只能是 `core/server -> pki`。
2. ID-2B：实现 `realm-manifest`、`ServerProfileV2`、离线 root CA、在线 intermediate、独立 assertion signer；正常 serve 缺材料必须失败关闭。
3. ID-2C：实现 users、handles、devices、device credentials/assertions/status、provision challenges/requests、tokens 的 v2 server schema 和 RPC。
4. ID-2D：实现客户端 profile import、安全存储和最小 local-v2 identity schema。
5. ID-2E：执行显式 `cutover-v2`，完整归档 v1 bundle，在临时目录创建并校验空 v2 realm 后原子切换。客户端归档后必须等待管理员带外导入 profile；不迁移 v1 数据。

设备 provision：账号密码重认证取得单次 challenge；客户端本地生成 TLS key/CSR，并以 CSR signed attribute 绑定 challenge/RealmId/RequestId；Ed25519 key 对 canonical request 签名。私钥永不上传。Provision 使用 pending -> active 可恢复状态机，RequestId 幂等，pending 永不授权。

### 历史 ID-3：DeviceHello 与可信会话闭环（M3）

1. 定义 DeviceCredential、CredentialAssertion、CredentialStatusProof、PeerPrincipal、CredentialEvidence 与 DeviceHello。
2. 在新 ALPN/major version 的 BIND 前验证 CA chain、SPKI、realm、credential、status、nonce 和角色。
3. 写入不可变 SessionBinding；未认证连接不得产生 gateway visible session、联系人、副作用或应用帧。
4. 删除 IdentityFrame 自报绑定/旧授权路径，旧类型明确 `UnsupportedProtocol`。

### 历史 ID-4：身份、presence 与会话目录（M3）

1. IdentityStore 持久保存验证过的用户、设备、credential/assertion/status 历史。
2. `begin_instance` 创建服务端 lease epoch；每 DeviceId 仅一个 active InstanceId。
3. 实现 90 秒 TTL、30 秒续租、最多 8 个 Host candidates 的 PresenceLease。
4. 将 ContactRegistry 改为 user -> devices 视图；SessionRegistry 成为唯一内存会话事实源。

### 历史 ID-5：投递与传输路由（M4）

1. 拆出 DeliveryPlanner 与 TransportRouter，删除 user/session 三路 fallback。
2. `(DeviceId, InstanceId)` 选择 primary session：assertion sequence 降序、规范发起者、TransportSessionId 升序。
3. 一对一投递到对方全部 active devices，并同步本用户非源设备；记录冻结 recipient set 与逐设备结果。
4. 引入不可变 MessageEnvelope、DeliveryFrame、显式补发和 `MAX_HOPS=2`。

### 历史 ID-6：业务 ID 轴（M4）

1. 用 `ConversationId::Direct(DirectId)|Group(GroupId)` 替换 `conversation + is_group`。
2. MessageId 取代本地消息序号，认证后按 `(RealmId, MessageId)` 去重。
3. TransferId 进入 FileMeta/FileChunk/FileComplete，按 `(RealmId, TransferId)` 重组并固定 origin device。
4. EventId/RequestId 贯通群组、Room、补发与 RPC；GroupId 只接受单一 genesis。
5. 扩展 local-v2 schema，不导入已归档的 v1 行；更新 Cap'n Proto snapshot。

### 历史 ID-7：治理与端到端验证（M5、M6）

1. 完成 Flutter/FRB 设备列表、撤销、轮换、credential 到期、handle 与安全存储 UX。
2. 群组按 UserId 授权、按 DeviceId/CredentialId/KeyId 审计并验证历史 status proof。
3. 完成多设备、轮换、撤销、instance 替换、会话续期、伪造 subject、跨 realm 重放、SPKI/status/lease/message/transfer/genesis 冲突测试矩阵。

## 根目录验证命令

从仓库根运行。统一入口包含 Rust workspace、独立 `flutter/rust` crate、demo、debug-harness、Flutter 和生成物漂移检查；缺少必需工具会显式失败。

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File scripts/Validate-ModCptLib.ps1 -Mode full
```

定向诊断也必须从根目录显式选择 manifest，尤其不能漏掉独立 FRB Rust crate：

```text
cargo fmt --manifest-path rust/Cargo.toml --all -- --check
cargo clippy --manifest-path rust/Cargo.toml --workspace --all-targets -- -D warnings
cargo test --manifest-path rust/Cargo.toml --workspace --all-targets
cargo fmt --manifest-path flutter/rust/Cargo.toml --all -- --check
cargo clippy --manifest-path flutter/rust/Cargo.toml --all-targets -- -D warnings
cargo test --manifest-path flutter/rust/Cargo.toml --all-targets
cargo test --manifest-path demo/Cargo.toml --all-targets
cargo test --manifest-path debug-harness/Cargo.toml --all-targets
```

Flutter 的 `pub get`、analyze 和 test 由统一根脚本在 `flutter/` 子目录运行；不得依赖前一条 shell 命令遗留的工作目录。每个 M 阶段还必须通过 `ROADMAP.md` 的 gate 和对应 TEST_MATRIX 用例。M3 前不得把 v2 ID 基础描述为已认证生产会话；M4 前不得建立基于 v2 假设的新群组/Room 产品能力。
