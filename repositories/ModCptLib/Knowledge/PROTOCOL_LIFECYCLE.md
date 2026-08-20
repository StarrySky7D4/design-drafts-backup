# 协议生命周期与发布规则

> **状态**：M2-02 规范，2026-07-29。当前编号、布局、迁移和错误类别只以[协议注册表](PROTOCOL_REGISTRY.md)为事实源。本页定义跨 wire、schema、SQLite 和 FRB 的变更分类与发布门；不分配新值，也不把规划中的 DeviceHello、群组或 Room 描述为已实现。

## 1. 变更前置

每个外部或持久边界变更必须先：

1. 由注册表 owner 登记编号、状态和命名空间；未登记值只能留在评审材料。
2. 标注本页的一个或多个分类；多重分类采用最严格门。
3. 写 producer/consumer、旧/新 storage 与 Rust/FRB/Dart 的兼容矩阵，明确拒绝类别与零副作用断言。
4. 提供固定向量、边界 corpus 或真实旧版本 fixture，并在集成分支重跑。

| 分类 | 判定 | 最低处理 |
|---|---|---|
| `breaking` | 固定字节、持久语义、公开 API、认证或授权结果改变，使正确旧端不能无歧义互操作 | 新已批准版本/major、双向拒绝矩阵、支持窗口和回滚/前滚方案 |
| `additive` | 旧端可安全不识别，且不改变旧成功路径、认证、资源上限或必需结果 | 先登记 extension point；明确 optional/required 与缺能力拒绝 |
| `behavioral` | 布局未变但验证、错误、资源、重试或授权语义改变 | 兼容矩阵、发布说明、回归与零副作用测试；旧端无法安全解释时同时是 breaking |
| `data-only` | 只改文档、测试向量、语料或内部数据，不改变外部 parser、schema、API 或业务结果 | 证明外部契约未变并保存可重复输入 |

SQLite 内容约束即使没有 DDL 也会改变旧库能否打开，是 storage `breaking` 加数据迁移。FRB `String -> typed ID` 是 API `breaking` 加 `behavioral` 收紧，不能以隐式 parse 或 session fallback 伪装为兼容。

## 2. ALPN、Major 与 Capability

- 固定控制帧长度、字段序、字节序、canonical 编码、字段语义、握手顺序、认证材料或 BIND 前后授权边界改变，必须是 `breaking`，先登记新的 ALPN major；禁止在同一 ALPN 下尝试两种解析。
- 未知、废弃、未配置或未协商 major 必须在可见 session、数据库写入、事件投递或应用 handler 前以 protocol-unsupported 失败。失败、超时、能力缺失或旧缓存不得自动回退到较低 major。
- capability 仅表达已登记 major 内、可安全关闭的 optional 行为；不得改变固定布局、canonical 输入、principal 产生条件、撤销、资源限制或必需投递结果。required capability 缺失必须在副作用前拒绝。
- 当前 BIND v2 和 v1 拒绝事实见[注册表](PROTOCOL_REGISTRY.md#2-alpnmajor-与-capability)；本页不重新分配这些值。

## 3. 未知、保留与畸形输入

| 输入 | 唯一处理 | 禁止 |
|---|---|---|
| 未知/未协商 ALPN major，未知 control tag | `UnsupportedProtocol`，关闭连接 | 旧 ALPN 回退、双长度猜测、临时 session |
| 已知 control tag 的长度、尾随、固定字段、role 或编码错误 | `MalformedFrame`，关闭连接 | 截断读取、忽略尾随、修正字段或无界等待 |
| 未知 application `MsgType` | 无 registered consumer 时拒绝；仅已登记、bounded、safely-ignorable extension 可跳过 | 因 fallback enum 就执行、路由或授权 |
| `reserved` / `deprecated` 编号 | 不发送、不 advertise、不作为新功能消费；deprecated 保留历史拒绝语义 | 重分配、暗中启用或借 raw gateway 绕过 |
| 未知 schema version / union | fail-closed | 尽力解析、默认 variant 或当作 absent |
| 未知 schema field | 仅已冻结为 optional、non-security、non-resource-affecting 且 safely-ignorable 时可忽略 | 忽略身份、授权、长度、canonical 签名或必需结果字段 |

当前 Cap'n Proto 没有已登记 union discriminant。新增 union 前必须登记版本和 unknown-case 行为。

## 4. Schema、Canonical 编码与向量

1. Cap'n Proto 只追加 field ordinal，绝不重排、复用或改变既有语义。
2. schema 源、snapshot 与生成物是原子契约；有 `capnp` 时通过生成流程更新，无工具时只使用提交的 snapshot。FRB 仅由 `flutter_rust_bridge_codegen generate` 生成。
3. 签名、assertion 和安全 transcript 使用共享 canonical encoder；JSON、Cap'n Proto packed bytes、map iteration 与展示文本均不是替代品。改变 canonical field 集、排序、长度前缀、字节序或 domain separation 是 `breaking`。
4. 每个边界改动提交正向及拒绝向量：round-trip、截断、尾随、错误版本/字段/字节序、未知项与适用的 `0/N-1/N/N+1`。向量不得包含 token、私钥、真实内容或可重放凭据。

## 5. SQLite 与恢复

- `user_version` 只向前推进；单个 migration 的 validation、转换、索引/约束和版本写回必须在同一 transaction，失败整体回滚。
- 每个 migration 用真实旧 fixture 验证成功、非法数据、引用错误、中断、并发/重复 open 和新库/迁移库等价；不得静默删除、随机改写或生成替代 ID。
- 每次持久语义变更的发布说明必须声明 source version、target、升级路径、备份要求、恢复操作、支持窗口及停止支持条件。transaction rollback 不等于备份/恢复能力。
- 发现已发布 migration/线路缺陷时，停止扩散；可恢复时按声明备份 restore，否则创建新的前向 repair/quarantine migration。禁止改写已发布 step、自动 downgrade 或用旧二进制回写更高版本库。

## 6. FRB/Dart 公共边界

FRB Rust API、生成 Dart binding 与 native library 是单一发布工件。当前没有 runtime version handshake；未列入矩阵的旧/new binding 与 native library 混配一律不支持。

| 变更 | 分类 | 必须门 |
|---|---|---|
| 删除、重命名或改变公开函数、DTO、错误形状、ID 表示或默认语义 | breaking；SDK/app major | paired codegen、独立 `flutter/rust` build/test、Flutter analyze/test、旧/新 artifact matrix |
| 新增独立 API 或真正 optional DTO 字段且不改变旧边界 | 可能 additive | 旧入口回归、生成物零漂移、缺能力零副作用 |
| 内部重构、文档或向量且 public/generated 面不变 | data-only | `git diff` 证明公开面未变 |

## 7. 验证职责与发布清单

- PR quick gate：fmt、静态检查、受影响 unit/component、固定向量/畸形 corpus、生成物零漂移与受影响旧/新矩阵。
- scheduled deep gate：长 fuzz、故障注入、并发/网络扰动、soak、泄漏与资源检查。seed、预算、工具版本必须记录；panic、hang、超限或错误分类漂移均失败。
- 最小化 fuzz 失败必须回灌非敏感固定 corpus；不得 skip、吞退出码或无限重试。

发布前必须确认：注册表已更新；分类/major/API semver/支持窗口已评审；固定向量、真实旧库、拒绝零副作用和矩阵通过；schema/FRB 生成零漂移；适用时有备份/恢复演练；deep-gate 结果可追溯精确提交。

## 8. M2-02 决策走查

| 用例 | 唯一结论 |
|---|---|
| M2-02-T01，BIND LinkId 8 -> 16 | breaking；新 ALPN major、固定向量与旧/新双向拒绝，禁止同 ALPN 双解析。 |
| M2-02-T02，SQLite v4 -> v5 | storage breaking 加数据迁移；transaction、真实 fixture、repair/retry、备份/恢复和支持窗口。 |
| M2-02-T03，FRB String -> typed ID | API breaking 加 behavioral；SDK major、paired codegen、独立 crate 和 Dart artifact matrix，禁止 fallback。 |
| M2-02-T04，optional field / unknown union | 仅 optional non-security field 可 additive；unknown union fail-closed。 |
| M2-02-T05，新 DeviceHello ALPN | breaking；先批准/登记独立 ALPN major，在 BIND 前授权，禁止使用旧身份或保留 tag。 |
| M2-02-T06，发布后缺陷 | 停止扩散，按支持窗口 restore 或新前向 repair；不改写 step、不自动 downgrade。 |
| M2-02-T07，PR corpus / deep fuzz | corpus 通常 data-only，但 gate 策略是 behavioral；PR 负责短确定性 corpus，scheduled job 负责 deep fuzz。 |
| M4-01，route header version `0x02` + `MsgType::Envelope(0xB0)` | breaking 边界内新增已批准版本：v1 header/帧保持 v1 only（`Message::from_bytes` 不变），v2 信封帧只接受 `0x02`；双向拒绝矩阵、稳定 `EnvelopeFrameError` 类别与零副作用断言。旧 v1 类型不得被重新解释为 v2，v2 帧不得被 v1 路径猜测解析。 |
| M4-01，SQLite `5 -> 6` | additive storage 迁移：只新增 `v2_messages`/`legacy_v5_quarantine`，不改写 v1 行、不自动生成 ID；v5 旧行经 `quarantine_legacy_v1` 原值归档；同一事务推进 `user_version`，重复打开幂等。 |
| M4-01，FRB `parse_conversation_id` typed DTO | API additive：新增独立 typed DTO 入口，旧边界不动；非 canonical/跨 kind 输入在副作用前拒绝，生成物由官方 codegen 更新（零漂移）。 |
| M4-05-D2/D3，native Olm v2 content profile | security behavioral + new breaking content-profile boundary：显式退役未实施的 ChaCha/external-AD strict proposal，登记 profile `1`、content kind `3/4`、canonical inner wire、prekey RPC 和 SQLite `7 -> 8`。旧端对 kind `3/4` fail-closed；没有同-kind 双解析或降级。`7 -> 8` 为 additive storage migration，但 encrypted state、atomic commit 与 post-decrypt gate 是 behavioral。vodozemac version/config、canonical field set、two-phase bootstrap 或 signature input 的任何改变都需要新 profile/version、双向拒绝矩阵、固定 vectors 和支持窗口。 |
| M5-01 device lifecycle v1 与 logical direct fan-out 8→15 | 新增 versioned canonical local/control contract；recipient-set commitment 的资源行为从 8 放宽到 15，但单 Envelope/route/storage cap 仍为 8，direct/attachment wire 仍逐 recipient。<=8 commitment bytes 完全不变，旧 receiver 不解析完整 recipient set；因此为 behavioral resource expansion，不改 ALPN/Envelope/content profile。15-device producer 必须先验证所有对端均支持 M5 lifecycle contract，缺能力时副作用前拒绝，不得截断、拆 MessageId 或回退。 |
| M5-02 PortableBackupV1 encrypted container | 新的 versioned local storage boundary：Argon2id 参数、salt/nonce、ChaCha20Poly1305、完整 header AAD 与 canonical manifest/entries 已冻结并实现；unknown/future/tamper/wrong-key/path/symlink/reparse 在产品写入前 fail-closed。它仍是 data-only backup，不包含 identity authority；SQLite/WAL 一致 snapshot、fresh status/lease 和 product DB commit 尚未实现。 |
| M5-02 W25 `createNodeWithStorage` / `createNodeWithAccountStorage` | FRB API additive：保留 core-only exact-32 入口；新增 signed-in 双路径/双 key 入口，同时打开 core 与 local-v2 SQLCipher store。旧 `createNode` 仅保留 ephemeral compatibility。官方 FRB 2.12.0 paired codegen、Flutter Rust/Flutter tests 通过；generated Dart/native 不支持混配。 |
| M7-02 W34/W37 public operation v1 | API additive + behavioral：FRB/Rust SDK 新增 start/status/cancel/result 入口，legacy await API 保留。UUIDv4、1..300000 ms、1024 active/terminal/result、有限 error mapper、child abort+await、cleanup-before-terminal 与 mutation unknown-commit 已进入 paired codegen/conformance。 |
| M7-02 `FileTransferV2 (0x23)` | 新独立 application wire，旧 0x20..0x22 不重解释。`MCFTV201`/profile 1、UUIDv7 TransferId、meta/chunk/complete/cancel/ACK shape 与 bounds 已冻结；unknown/truncated/trailing fail-closed。CancelAck 只在 exact partial 删除和 60 秒 tombstone 成功后发送。 |
## 9. M7 Mailbox v1 lifecycle addendum

Mailbox v1 是独立 storage/transport contract，不改变 core P2P ALPN、route wire 或自动投递策略。兼容单位是 `modcpt_mailbox` crate + schema version 1 + `modcpt-mailbox/1` ALPN；`runtime_v1` 已冻结 admission、deadline/cancel、Future drop、唯一终态与 graceful shutdown，`rpc_v1` 冻结 exact-session RequestId owner，network host 要求严格 mTLS 与 DeviceHello。

- schema 只允许向前迁移；未知 `schema_version` fail-closed，不得 plaintext/default-key 打开或自动 downgrade。
- `(RealmId, OriginDeviceId, MessageId)` 的 digest 在 TTL 内不可改变；正文因全部 ack 删除后仍保留 dedup tombstone，防止旧 wire 重新投递。
- 调低 quota/rate/fetch bounds 属于行为兼容但可能拒绝过去成功的 workload，发布前必须进入容量/rollback 评审；提高 envelope、TTL、recipient 或 metadata retention 上限需重新安全/隐私评审。
- stable `mailbox_*` code 的触发条件不可静默改变；message text 可改但不是兼容标识。
- blocking 工作开始前 cancel/deadline 必须零 store 副作用；开始后同步 transaction 的真实结果胜出。调用方 Future drop 不等于取消，operation/permit 由 managed task 持有到真实完成；shutdown 必须停止 admission 并等待全部许可释放。
- 默认/最大 32 blocking operations 与 100 ms overload retry hint 是本地 runtime 行为合同；修改上限、排队模型或终态竞争优先级必须新增容量/取消 conformance 与回滚评审。
- transport-neutral RPC owner 以 exact verified session incarnation + UUIDv4 RequestId 绑定 server operation；active/terminal 各 1024。prepare replay、跨 session cancel 与 N+1 必须在 store 前失败；Future drop 后 managed owner 保留到真实终态。该本地合同仍不等于已发布 network wire。
- `MCMBX001`/v1 冻结 upload/fetch/ack/cancel 双向 bytes；unknown version/op/flags/status、非 UUIDv4 ID、截断、超长、非法 terminal/error 组合与 trailing 一律拒绝。当前 client 只支持 v1，未来 N-version 必须用显式新 ALPN/兼容矩阵，不能在 `/1` 下猜测布局。
- `modcpt-mailbox/1` 只接受 CA-verified client certificate，随后以 TLS leaf SPKI 验证 initiator DeviceHello；连接期 `(DeviceId, nonce)` replay、128 连接、32 业务流、10 秒认证与 30 秒请求均为硬边界。证书轮换/跨版本/跨主机部署仍需 release evidence。
- Room/媒体、push ciphertext、relay、STUN/TURN/CID/NAT fallback 不是 mailbox v1 capability。加入任一项需要新 ADR、registry owner、攻击/容量 vectors 和独立 release gate。
- `modcpt_core` 对 `modcpt_mailbox` 的反向依赖、core send failure 自动调用 mailbox、或 request 自行携带未验证 `SessionBinding` 都是 fail gate。

## 10. M7 server release artifact addendum

Windows server ZIP 的本地合同是 `modcpt.release.server.v1` 聚合 manifest + SHA-256 + artifact-scoped CycloneDX 1.5 + in-toto/SLSA v1 provenance subject。消费者必须从 sidecar 重新计算 artifact 和 archive file digest，并确认 SBOM/provenance exact subject；生成 sidecar 但不消费验证不能通过。

dry-run 允许 dirty/unsigned 仅用于工具负面矩阵，manifest 必须记录 `dryRun=true`，`RequireReleaseCandidate` 必须拒绝。真实 RC 必须来自干净 exact `vMAJOR.MINOR.PATCH` tag、server Cargo version 一致、release profile build，并对 ZIP 内 EXE 验证 Valid Authenticode 和可信时间戳。unsigned provenance JSON 不是 attestation；未来 issuer/subject/custody policy 与签名 bundle 需新登记和消费者验证。

server v2 SQLite 只向前迁移，release manifest 当前声明 schema 1..5 read、5 write。已写高版本状态后不得用旧 binary 回写或自动 downgrade；只能按兼容 manifest 恢复 backup 或 forward repair。
