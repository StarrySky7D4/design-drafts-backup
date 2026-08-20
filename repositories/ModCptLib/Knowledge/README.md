# ModCptLib Knowledge 索引

> **知识库用途**：本目录对项目每一个源代码文件进行独立的中文文档化分析，供人类开发者与 AI Agent 快速检索理解。
>
> **如何使用**：要理解某个源文件，直接打开 `Knowledge/` 下对应路径的同名 `.md` 文件。
> 例：想了解 `rust/core/src/net/p2p.rs` → 读 `Knowledge/rust/core/net/p2p.md`。
>
> **维护规则**：逐文件文档保留实现细节；当前项目级结论以 Architecture、Roadmap 与 Defects 为准。完整设计、实施记录、复核取证和逐日变更也位于本目录的 `designs/`、`reviews/` 与 `logs/`；不再使用独立 `docs/` 目录。

---

## 📂 总览

### 当前权威入口

- [Architecture](ARCHITECTURE.md) — 当前运行架构与直接拨号边界
- [Roadmap](ROADMAP.md) — 唯一可执行的 M0-M7 路线、阶段门与非目标
- [Defects](DEFECTS.md) — 活跃缺陷登记
- [路线图补充意见](ROADMAP_SUPPLEMENT.md) — 历史横向分析与 N 标签映射，不是执行路线
- [协议、Schema、数据库与错误注册表](PROTOCOL_REGISTRY.md) — 当前已实现、保留与废弃值的防碰撞索引
- [first-stable 兼容矩阵](supply-chain/compatibility-matrix-v1.md) — app/native、wire、backup、SQLite N/N+1 机器可读门禁与显式 N−1 工件缺口
- [协议生命周期](PROTOCOL_LIFECYCLE.md) — 跨 wire、schema、SQLite 与 FRB 的版本、兼容、发布与恢复规则
- [构建指南](guides/build.md)、[账号与发现](guides/account-and-discovery.md)、[调试工具](guides/debug-harness.md)
- [群组与房间设计](designs/groups-and-rooms.md)、[群组与房间实施细化](designs/groups-and-rooms-implementation.md) — 历史/non-normative 设计证据；现行实施只遵循 Roadmap 与上位规范
- [身份与 FRB 演进](designs/peer-identity-and-frb.md)、[FRB API 演进](designs/frb-api-evolution.md) — 历史设计/实施取证，非执行路线；[未来中继契约](designs/relay-module-contract.md)
- [身份与寻址 v2](designs/identity-and-addressing-v2.md) — 当前 major upgrade 的权威模型与分阶段状态
- [身份与寻址 v2 实施计划](designs/identity-and-addressing-v2-implementation-plan.md) — ID-1 至 ID-7 执行顺序、验收条件与当前完成度
- M7 W33 合同：[公共面清单](designs/m7-w33-public-surface-inventory.md)、[typed error](designs/m7-w33-public-error-contract.md)、[取消与清理](designs/m7-w33-operation-cancel-cleanup.md)、[隐私安全指标](designs/m7-w33-privacy-safe-metrics-schema.md)、[发布/版本/回滚 ADR](designs/m7-w33-release-version-rollback-adr.md) — 已冻结的准备合同；实现、SLO、SBOM、签名与发布证据仍待后续 gate
- [v2 威胁模型与信任边界](THREAT_MODEL.md) — Realm root/manifest、signer、设备、攻击者与安全属性边界
- [内容安全与密钥生命周期](CONTENT_SECURITY.md) — 一对一文本/附件、撤销与历史访问的冻结规范
- [完整复核与取证记录](reviews/README.md)、[复核方法学](guides/review-methodology.md)

以下索引保留逐文件详细说明。文档数量不作为维护目标。

| 板块 | 文档数 | 目录 |
|------|--------|------|
| 代码组织结构 | 1 | [`STRUCTURE.md`](STRUCTURE.md) |
| Rust workspace | 1 | `rust/Cargo.md` |
| Rust core / net | 6 | `rust/core/net/` |
| Rust core / data_ctl | 15 | `rust/core/data_ctl/` |
| Rust core / serve | 7 + 1(兼容层) | `rust/core/serve/` |
| Rust PKI | 6 | `rust/pki/` |
| Rust identity | 10 | `rust/identity/` |
| Rust core 根 | 17 | `rust/core/` |
| Rust client | 5 | `rust/client/` |
| Rust server | 7 | `rust/server/` |
| Rust msg_api | 5 | `rust/msg_api/` |
| Flutter FRB(Rust) | 15 | `flutter/frb/` |
| Flutter FRB(Dart生成) | 4 | `flutter/frb/` |
| Flutter services | 6 | `flutter/services/` |
| Flutter models | 1 | `flutter/models/` |
| Flutter UI | 19 | `flutter/ui/` |
| Flutter 入口 | 2 | `flutter/` |
| demo | 3 | `demo/` |
| debug-harness | 9 | `debug-harness/` |

---

## 🦀 Rust 核心（modcpt_core）

### 网络层 `rust/core/src/net/`
- [p2p.md](rust/core/net/p2p.md) — ★ QUIC 会话/通道/双链路/TLS（核心模块）
- [p2p_gateway.md](rust/core/net/p2p_gateway.md) — 防腐网关
- [route.md](rust/core/net/route.md) — 线路帧协议 + MsgType + 路由
- [dht.md](rust/core/net/dht.md) — 分布式哈希表广播
- [center_broadcast.md](rust/core/net/center_broadcast.md) — 集中式广播
- [mod.md](rust/core/net/mod.md) — 模块声明

### 数据控制层 `rust/core/src/data_ctl/`
- [mod.md](rust/core/data_ctl/mod.md) — ★ DataCtl 门面（Phase D：UiStore + 协议解析 + 本地身份 + 密钥对；Phase F：bind_with_db + DB 钩子）
- [msg.md](rust/core/data_ctl/msg.md) — 文本消息控制器
- [file.md](rust/core/data_ctl/file.md) — 文件分块传输 + 重组
- [file_transfer_v2.md](rust/core/data_ctl/file_transfer_v2.md) — M7 TransferId 文件 wire、complete/cancel ACK 与迟到帧 tombstone
- [voice.md](rust/core/data_ctl/voice.md) / [video.md](rust/core/data_ctl/video.md) — 音视频帧
- [bytes.md](rust/core/data_ctl/bytes.md) — 原始字节
- [custom.md](rust/core/data_ctl/custom.md) — 扩展消息（未来/未知类型，ExtMessage/CustomCtl）
- [contacts.md](rust/core/data_ctl/contacts.md) — ★ ContactRegistry（userId 身份注册表，阶段 B1）
- [database.md](rust/core/data_ctl/database.md) — ★ DatabaseCtl SQLite v8（legacy + typed Envelope + direct encrypted atomic state）
- [envelope_v2.md](rust/core/data_ctl/envelope_v2.md) — M4 authenticated Envelope ingress gate（principal/recipient/generation/signature before DB）
- [ui_store.md](rust/core/data_ctl/ui_store.md) — ★ UiStore（Phase D：消息/文件/通话/群组文件单一事实源；Phase F：DB write-behind 镜像 + load_from_state）
- [proto.md](rust/core/data_ctl/proto.md) — 文本协议命令解析（Phase D：9 类命令解析器）
- [convert/mod.md](rust/core/data_ctl/convert/mod.md) — convert 子模块
- [convert/router.md](rust/core/data_ctl/convert/router.md) — ★ DataRouter 中央路由器
- [convert/serializer.md](rust/core/data_ctl/convert/serializer.md) — Cap'n Proto 序列化
- [transport_identity.md](rust/core/data_ctl/transport_identity.md) — 传输隔离身份认证器（fail-closed，未接线的 quarantine 模块）

### 服务层 `rust/core/src/serve/`
- [mod.md](rust/core/serve/mod.md) — 模块声明
- [auth.md](rust/core/serve/auth.md) — ★ 证书签发/验证 + mTLS
- [crypto.md](rust/core/serve/crypto.md) — Argon2id + ChaCha20-Poly1305
- [key_store.md](rust/core/serve/key_store.md) — CA 私钥冷存储
- [database.md](rust/core/serve/database.md) — NodeStore + SqliteStore
- [bulletin_board.md](rust/core/serve/bulletin_board.md) — 集中式用户发现
- [signaling.md](rust/core/serve/signaling.md) — 呼叫信令 + 防骚扰
- [ca/mod.md](rust/core/serve/ca/mod.md) — `modcpt_pki` 兼容 re-export；实际 PKI 见下方

### core 根文件
- [lib.md](rust/core/lib.md) / [build.md](rust/core/build.md) / [message.capnp.md](rust/core/message.capnp.md) / [Cargo.md](rust/core/Cargo.md)
- [delivery_v2.md](rust/core/delivery_v2.md) - M4 recipient freeze and deterministic verified-session arbitration
- [delivery_state_v2.md](rust/core/delivery_state_v2.md) - M4 signed per-device receipt, bounded persistent outbox/inbox and retry state
- [delivery_runtime_v2.md](rust/core/delivery_runtime_v2.md) - M5 LocalOnlyV1 exact-recipient deferred direct dispatcher
- [public_error_v1.md](rust/core/public_error_v1.md) - M7-02 W34 typed-error、敏感字段边界、cancel control 与 exactly-one terminal 纯 Rust 骨架
- [operation_registry_v1.md](rust/core/operation_registry_v1.md) / [public_operation_adapter_v1.md](rust/core/public_operation_adapter_v1.md) - M7 有界 operation owner 与 cleanup→terminal→metric 适配器
- [public_async_operation_v1.md](rust/core/public_async_operation_v1.md) / [data_ctl_public_operation_v1.md](rust/core/data_ctl_public_operation_v1.md) - 真实 child 监督、异步 cleanup 与有限 DataCtl 错误投影
- [observability_v1.md](rust/core/observability_v1.md) - M7 固定枚举、隐私有界的 metrics foundation
- [group_mls_sqlcipher_v1.md](rust/core/group_mls_sqlcipher_v1.md) - M6 W29 SQLCipher 原子群存储、restart/recovery 与 portable audit 边界
- [group_mls_directory_v1.md](rust/core/group_mls_directory_v1.md) - M6 W30 verified directory authority、exact-leaf fan-out 与生命周期规划
- [group_mls_product_v1.md](rust/core/group_mls_product_v1.md) / [group_mls_transport_v1.md](rust/core/group_mls_transport_v1.md) - M6 W30 产品 owner、认证传输封装、原子收敛与历史边界
- [device_directory_v2.md](rust/core/device_directory_v2.md) - M5 signed device-directory pin/cache and bounded 15-device direct fanout
- [transfer_v2.md](rust/core/transfer_v2.md) - M4 bounded TransferId chunk tracker
- [direct_e2ee.md](rust/core/direct_e2ee.md) - ★ M4-05 exact native Olm v2 runtime、two-phase bootstrap、post-decrypt type gate 与 atomic state
- [attachment_e2ee.md](rust/core/attachment_e2ee.md) - ★ M4-04-I1 secret descriptor、chunk AEAD、bounded reassembly、atomic outbox 与 receipt runtime
- [mailbox_adapter_v1.md](rust/core/mailbox_adapter_v1.md) - M7 current-session + exact-origin canonical Envelope upload validation seam

---

## 🦀 Rust 其他 crate

### group_mls（modcpt_group_mls）`rust/group_mls/`
- [authority_v1.md](rust/group_mls/authority_v1.md) — exact leaf credential/KeyPackage authority 与 deterministic provider snapshot
- [runtime_v1.md](rust/group_mls/runtime_v1.md) — OpenMLS create/join/add/remove/rotate/application runtime
- [storage_contract_v1.md](rust/group_mls/storage_contract_v1.md) — 原子 epoch/application/receipt 存储契约与 portable audit 边界
- [profile_v1.md](rust/group_mls/profile_v1.md) — RFC 9420/OpenMLS 固定 profile、suite 与资源上限

### mailbox（modcpt_mailbox）`rust/mailbox/`
- [lib.md](rust/mailbox/lib.md) — M7 opaque Envelope upload/fetch/ack、SQLCipher、TTL/quota/rate/dedup/ack engine
- [runtime_v1.md](rust/mailbox/runtime_v1.md) — M7 有界 Tokio blocking host、cancel/deadline、Future drop、唯一终态与 graceful shutdown
- [metrics_http_v1.md](rust/mailbox/metrics_http_v1.md) — M7 loopback-only 有界 OpenMetrics HTTP exporter、抓取上限与 shutdown owner
- [rpc_v1.md](rust/mailbox/rpc_v1.md) — M7 exact-session RequestId/server-operation owner、显式取消与 1024 active/terminal replay bounds
- [wire_v1.md](rust/mailbox/wire_v1.md) — M7 `MCMBX001` canonical bounded upload/fetch/ack/cancel request/response framing
- [transport_v1.md](rust/mailbox/transport_v1.md) — M7 专用 ALPN、严格 mTLS + DeviceHello QUIC host/client、replay/连接/流上限与双节点 E2E
- [bin_mailboxd_v1.md](rust/mailbox/bin_mailboxd_v1.md) — M7 production-oriented daemon config、secret/file bounds、QUIC/metrics composition 与 graceful shutdown
- [mailbox_load_v1.md](rust/mailbox/examples/mailbox_load_v1.md) — M7 固定 seed SQLCipher load/fault gate、p99/总时长硬预算与 JSON 证据
- [mailbox_cross_host_v1.md](rust/mailbox/examples/mailbox_cross_host_v1.md) — M7 独立 host/client 跨主机 soak harness、非 loopback 证据门与本地组合自检
- [mailbox_recovery_v1.md](rust/mailbox/examples/mailbox_recovery_v1.md) — M7 SQLCipher cold backup、corruption quarantine/restore 与 future-schema fail-before-mutation 演练
- [Cargo.md](rust/mailbox/Cargo.md) — `mailbox -> core` 单向依赖与 SQLCipher dependencies
- [Mailbox-first ADR](designs/mailbox-first-v1.md) — 产品选择、Room 冻结、隐私/资源/非目标边界

### client（modcpt_client）`rust/client/`
- [lib.md](rust/client/lib.md) — ★ QUIC 长连接 + 单向 TLS + JSON RPC + 重连
- [local_v2.md](rust/client/local_v2.md) — ★ **M3-04** 隔离 v2 本地身份 store（独立 SQLite `user_version` 域；Argon2id+ChaCha20Poly1305 封装；import validate(now) 前置；重启重验证+解密 fail-closed；不接线 live provision/QUIC）
- [data_lifecycle_v1.md](rust/client/data_lifecycle_v1.md) — M5 本地数据动作矩阵、平台能力、备份 allowlist/caps 与恢复状态机
- [e2e_quic.md](rust/client/e2e_quic.md) — 端到端集成测试
- [remote_debug.md](rust/client/remote_debug.md) / [Cargo.md](rust/client/Cargo.md)

### identity（modcpt_identity）`rust/identity/`
- [lib.md](rust/identity/lib.md) — IdentityAssertion、canonical 编码、ServerProfile、rotation/recovery proof
- [ids.md](rust/identity/ids.md) — ★ **M4-01** 共享强类型 ID 与 `ConversationId::Direct|Group`（`d:`/`g:` 前缀文本 + kind byte canonical 编码）
- [envelope.md](rust/identity/envelope.md) — ★ **M4-01** canonical 不可变 `MessageEnvelope`（RealmId/ConversationId/Origin/MessageId/recipients/sender 签名、canonical wire、固定向量）
- [receipt.md](rust/identity/receipt.md) — ★ **M4-03** canonical per-device delivery receipt payload（绑定 M2-03 EnvelopeCommitment）
- [direct.md](rust/identity/direct.md) — ★ **M4-05** canonical signed prekey、recipient commitment、direct ciphertext/plaintext 与 bootstrap control
- [device_lifecycle.md](rust/identity/device_lifecycle.md) — M5 credential rotation/revoke、signed device directory、presence lease 与 frozen direct fanout contract
- [attachment.md](rust/identity/attachment.md) — ★ **M4-04-I1** canonical secret descriptor、public chunk、two-layer commitments 与 profile bounds
- [Cargo.md](rust/identity/Cargo.md) — 共享身份 crate 依赖
- [realm.md](rust/identity/realm.md) — RealmRoot、RealmManifest、有限 signer rotation 与 ServerProfileV2
- [provision.md](rust/identity/provision.md) — v2 device provision challenge/request canonical transcript
- [device_hello.md](rust/identity/device_hello.md) — ★ **M3-05** 纯可信会话契约（DeviceHello、CredentialEvidence、PeerPrincipal、SessionBinding、无状态验证器）

### pki（modcpt_pki）`rust/pki/`
- [lib.md](rust/pki/lib.md) / [Cargo.md](rust/pki/Cargo.md) — 纯 PKI 公共契约与依赖边界
- [root_ca.md](rust/pki/root_ca.md) / [intermediate.md](rust/pki/intermediate.md) / [crl.md](rust/pki/crl.md) — 两级 CA 与 CRL
- [leaf.md](rust/pki/leaf.md) / [csr.md](rust/pki/csr.md) — 叶证书/SPKI 与 PKCS#10 CSR 原语

### server（modcpt_server）`rust/server/`
- [lib.md](rust/server/lib.md) / [quic.md](rust/server/quic.md) — ★ QUIC 传输层（单向 TLS + JSON RPC）
- [store.md](rust/server/store.md) — ★ SQLite 三表 + Argon2id + 查重
- [store_v2.md](rust/server/store_v2.md) — ★ v2 schema 2：账户/设备/provision + M4-05 realm-bound signed public content-prekey store
- [rpc_v2.md](rust/server/rpc_v2.md) — ★ v2 RPC dispatcher：provision/device + signed prekey publish/atomic claim，由 mTLS `quic_v2` endpoint 消费
- [quic_v2.md](rust/server/quic_v2.md) / [cutover.md](rust/server/cutover.md) — ★ **M3-07** v2 mTLS endpoint 与显式 realm cutover
- [main.md](rust/server/main.md) — 二进制入口 / [Cargo.md](rust/server/Cargo.md)

### msg_api `rust/msg_api/`
- [lib.md](rust/msg_api/lib.md) / [mod.md](rust/msg_api/mod.md) / [msg_api.md](rust/msg_api/msg_api.md) / [Cargo.md](rust/msg_api/Cargo.md) — ★ 异步 LLM_bot/自动化接口（包装 DataCtl）
- [public_operations.md](rust/msg_api/public_operations.md) — M7 Rust SDK/RPC typed operation、cancel 与 cleanup-terminal controller

### workspace
- [rust/Cargo.md](rust/Cargo.md) — workspace 根（8 成员）

### scripts
- [New-ModCptServerRelease.md](scripts/New-ModCptServerRelease.md) — Windows server ZIP、checksum、artifact-scoped CycloneDX、provenance 与严格 RC/signature admission
- [New-ModCptMailboxRelease.md](scripts/New-ModCptMailboxRelease.md) — Windows Mailbox wrapper、daemon config/secret 边界与共用 release admission
- [Test-ModCptReleaseBundle.md](scripts/Test-ModCptReleaseBundle.md) — 消费者 digest/SBOM/provenance/archive/AuthentiCode 验证
- [Test-ModCptReleaseTooling.md](scripts/Test-ModCptReleaseTooling.md) — full gate dry-run 与 unsigned/non-tag/tamper 负面矩阵
- [Test-ModCptAdvisories.md](scripts/Test-ModCptAdvisories.md) — 四个 Cargo lockfile 的 RustSec 公告门禁与有期限例外
- [New-ModCptAppRelease.md](scripts/New-ModCptAppRelease.md) — Flutter Windows release、paired FRB、MSIX/SBOM/checksum/provenance 与签名/license admission
- [Test-ModCptAppReleaseBundle.md](scripts/Test-ModCptAppReleaseBundle.md) — 消费者 MSIX identity/content/binding/SBOM/签名验证
- [Test-ModCptAppReleaseTooling.md](scripts/Test-ModCptAppReleaseTooling.md) — app dry-run 与 unsigned/non-RC/tamper 负面矩阵
- [Validate-ModCptLib.md](scripts/Validate-ModCptLib.md) — quick/full 统一验证入口、生成物漂移门与 Windows FRB SDK snapshot/shim 调用约束
- [Validate-ModCptLib-core.md](scripts/Validate-ModCptLib-core.md) — 稳定入口复用的原 19-domain validator
- [Test-ModCptLicenses.md](scripts/Test-ModCptLicenses.md) — repository/Cargo/Dart 许可证与跨平台规范化摘要 gate
- [Test-M7GateEvidence.md](scripts/Test-M7GateEvidence.md) — exact-tag Release 最终证据 evaluator
- [Test-M7GateEvidence-core.md](scripts/Test-M7GateEvidence-core.md) — 最终证据精确字段与阶段语义 evaluator
- [Test-M7GateContract.md](scripts/Test-M7GateContract.md) — 最终证据策略 fail-closed 负面矩阵
- [New-M7SoakEvidence.md](scripts/New-M7SoakEvidence.md) — 两主机长期 soak client/server JSONL 的严格聚合与 manifest-ready 报告
- [Test-M7ApiPolicy.md](scripts/Test-M7ApiPolicy.md) — API 支持、弃用窗口与 legacy surface 策略
- [Test-M7ApiPolicyContract.md](scripts/Test-M7ApiPolicyContract.md) — API 支持策略及 JSON 类型绕过负面矩阵
- [Test-M7ApprovalPolicy.md](scripts/Test-M7ApprovalPolicy.md) — 四角色受保护 Environment 静态策略
- [Test-M7ApprovalPolicyContract.md](scripts/Test-M7ApprovalPolicyContract.md) — 四角色审批策略负面矩阵
- [Test-M7ApprovalRuntime.md](scripts/Test-M7ApprovalRuntime.md) — exact-tag GitHub 审批运行与 Release 资产复核

### supply-chain
- [compatibility-matrix-v1.md](supply-chain/compatibility-matrix-v1.md) — first-stable app/native、wire、backup 与 SQLite 版本窗口及 N−1 工件边界
- [rustsec-policy-v1.md](supply-chain/rustsec-policy-v1.md) — RustSec owner/reason/expiry 机器可读策略

---

## 🎯 Flutter 端

### 入口
- [main.md](flutter/main.md) — ModCptApp + ValueNotifier theme
- [pubspec.md](flutter/pubspec.md) — 依赖配置

### FRB 胶水层（Rust）`flutter/rust/`
- [api_mod.md](flutter/frb/api_mod.md) — ★ RustNode + create_node + log_stream（阶段 C→D：子模块拆分 + 子句柄 + UiStore + 密钥对 + 通话管理）
- [contacts.md](flutter/frb/contacts.md) — ★ ContactApi（sync list/get_ + connect_by_user_id + 好友管理，阶段 C1）
- [messaging.md](flutter/frb/messaging.md) — MessagingApi（userId 寻址发送，阶段 C1）
- [groups.md](flutter/frb/groups.md) — ★ GroupsApi + `snapshots()` sync 快照（群组 CRUD + C3 快照驱动）
- [mls_group.md](flutter/frb/mls_group.md) — M6 product RFC 9420 群治理、精确设备 DTO 与取消/重试边界
- [secure_group.md](flutter/frb/secure_group.md) — ★ **R2 新增** SecureGroupApi（安全群组状态机宿主：只读视图 + 验证门 apply 预签名事件/消息）
- [room.md](flutter/frb/room.md) — ★ **R3 新增** RoomApi（仅直连房间状态机宿主：只读视图 + 验证门 apply 预签名事件，公钥取自 ContactRegistry）
- [files.md](flutter/frb/files.md) — FilesApi（userId 文件发送，阶段 C1）
- [events.md](flutter/frb/events.md) — NodeEvent + event_stream（阶段 C2→D：域失效信号 + 8 种事件种类）
- [ui.md](flutter/frb/ui.md) — **新增** UiApi（Phase D：7 个 sync 快照方法 + 4 个 DTO）
- [envelope.md](flutter/frb/envelope.md) — ★ **M4-01 新增** typed ConversationId FRB 边界（`ConversationIdDto`/`parse_conversation_id`，非 canonical/跨 kind 输入拒绝）
- [operations.md](flutter/frb/operations.md) — M7 typed cancel/error DTO、process-wide adapter seam 与 operation contract version
- [binding.md](flutter/frb/binding.md) — M7 generated Dart/native contract、API window、capability/profile/fingerprint/codegen runtime compatibility gate
- [server_client.md](flutter/frb/server_client.md) — ★ 账号 server start/status/cancel 与有界 take-once 结果
- [log_bridge.md](flutter/frb/log_bridge.md) — tracing → Dart Stream
- [lib.md](flutter/frb/lib.md) / [Cargo.md](flutter/frb/Cargo.md) / [frb_generated.md](flutter/frb/frb_generated.md)
- FRB 生成 Dart：[api.md](flutter/frb/api.md) / [server_client_dart.md](flutter/frb/server_client_dart.md) / [log_bridge_dart.md](flutter/frb/log_bridge_dart.md) / [frb_generated_dart.md](flutter/frb/frb_generated_dart.md)

### services `flutter/lib/services/`
- [native_service.md](flutter/services/native_service.md) — 纯命令接口（Phase D：删 BridgeEvent + data accessors）
- [frb_native_service.md](flutter/services/frb_native_service.md) — ★ 薄命令转发（Phase D：零数据缓存）
- [app_state.md](flutter/services/app_state.md) — ★ 薄 ViewModel（Phase D：快照驱动、句柄缓存）
- [session_lock.md](flutter/services/session_lock.md) — 重复登录防护（锁文件）
- [frb_runtime.md](flutter/services/frb_runtime.md) — 进程级 FRB 初始化（INV-1）
- [format_helpers.md](flutter/services/format_helpers.md) — **新增** 格式化辅助函数

### tests `flutter/test/`
- [app_state_lifecycle_test.md](flutter/tests/app_state_lifecycle_test.md) — 登录等待 FRB 账号绑定、失败回滚与锁文件隔离

### models `flutter/lib/models/`
- [server_models.md](flutter/models/server_models.md) — 服务器通信模型（Phase D 唯一保留的模型文件）

### UI `flutter/lib/ui/`
- login：[login_page.md](flutter/ui/login/login_page.md) / [register_page.md](flutter/ui/login/register_page.md)
- main：[main_page.md](flutter/ui/main/main_page.md) — 三栏布局
- contact：[contact_frame.md](flutter/ui/main/contact/contact_frame.md) / [friends_list.md](flutter/ui/main/contact/friends_list.md) / [groups_list.md](flutter/ui/main/contact/groups_list.md) / [mls_group_governance_dialog.md](flutter/ui/main/contact/mls_group_governance_dialog.md)
- msg：[msg_frame.md](flutter/ui/main/msg/msg_frame.md) / [msg_card.md](flutter/ui/main/msg/msg_card.md) / [search_bar.md](flutter/ui/main/msg/search_bar.md)
- call：[call_screen.md](flutter/ui/main/call/call_screen.md)
- self：[self_frame.md](flutter/ui/main/self/self_frame.md) / [self_info_page.md](flutter/ui/main/self/self_info_page.md) / [files_page.md](flutter/ui/main/self/files_page.md) / [setting_page.md](flutter/ui/main/self/setting_page.md)
- widgets：[voice_recorder.md](flutter/ui/widgets/voice_recorder.md) / [voice_player.md](flutter/ui/widgets/voice_player.md) / [emoji_picker.md](flutter/ui/widgets/emoji_picker.md) / [file_announce_card.md](flutter/ui/widgets/file_announce_card.md) / [server_profile_card.md](flutter/ui/widgets/server_profile_card.md)

---

## 🛠 辅助项目

### demo `demo/`
- [main.md](demo/main.md) — ★ 10 场景端到端演示
- [p2p_integration.md](demo/p2p_integration.md) — 35+ 集成测试 / [Cargo.md](demo/Cargo.md)

### debug-harness `debug-harness/`
- [lib.md](debug-harness/lib.md) / [protocol.md](debug-harness/protocol.md) — ★ 控制面协议
- [bin_node.md](debug-harness/bin_node.md) / [bin_peer.md](debug-harness/bin_peer.md) / [bin_server.md](debug-harness/bin_server.md) / [bin_mesh.md](debug-harness/bin_mesh.md) / [grouptest.md](debug-harness/grouptest.md)
- [run-debug-ps1.md](debug-harness/run-debug-ps1.md) / [run-debug-sh.md](debug-harness/run-debug-sh.md) / [Cargo.md](debug-harness/Cargo.md)

---

## 📝 更新日志

所有知识库的更新记录保存在 [`logs/`](logs/) 目录，按日期命名。

- [2026-08-09 M7 两主机 24h soak 完成](logs/2026-08-09-m7-two-host-soak-complete.md) — 86400 秒/85445 rounds、五 fault、有界网络恢复、连接闭合、server active=0 与最终报告摘要。
- [2026-08-07 Flutter/FRB 边界审查](logs/2026-08-07-flutter-frb-boundary-review.md) — 登录提交点、UserId 寻址、停止清理、测试隔离与 generated API 版本边界。
- [2026-08-07 M7 soak 短时网络容错](logs/2026-08-07-m7-soak-network-tolerance.md) — 同进程连续时钟、幂等续轮、90/240 秒恢复预算与 resilience/Gate 证据绑定。

- [2026-08-06 M6/M7 候选分支集成审查](logs/2026-08-06-m6-m7-integration-review.md) — SQLCipher 入口收紧、临时写权限 CI 移除、FRB codegen/lock 收敛与 gate 证据。
- [2026-08-06 M6 gate closure](logs/2026-08-06-m6-gate.md) — W27-W32 产品闭环、19+14 定向矩阵与 11-domain full gate 证据。
- [2026-08-06 M7 defect closure audit](logs/2026-08-06-m7-defect-closure-audit.md) — D-06/08/09 并发与资源隔离修复、历史缺陷复核关闭、M7-01 与真实发布签名剩余输入。
- [2026-08-06 M7 Mailbox-first](logs/2026-08-06-m7-mailbox-first.md) — 产品路线决策、Room 冻结、独立 mailbox/core adapter、SQLCipher/攻击/容量证据。

- [2026-08-02 M4-05 blocker review](logs/2026-08-02-m4-05-blocker-review.md) - 候选库核验、协议/存储接点缺口、D2 决策门与保持 blocked 的依据。
- [2026-08-04 M4-05 direct E2EE 临时归档](logs/2026-08-04-m4-05-direct-e2ee-worktree-archive.md) - native Olm v2 实现、生命周期补强、已执行验证与中止的 full gate。
- [2026-08-04 M4-05 最终本地审计与验证](logs/2026-08-04-m4-05-final-local-audit.md) - 多 Agent 安全审计修复、锁文件收敛、定向回归与 11-domain full gate 证据。
- [2026-08-04 M5 多 Agent 并行计划审计](logs/2026-08-04-m5-parallel-planning-audit.md) - M5-01/02/03 的并行边界、决策阻断、LocalOnlyV1 与后续实施波次。
- [2026-08-04 M4-04 attachment E2EE](logs/2026-08-04-m4-04-attachment-e2ee.md) - attachment profile re-review、runtime/atomic receipt 实现与 M4 gate 验证记录。
- [2026-08-04 M5 W24 server continuous credential rotation](logs/2026-08-04-m5-w24-server-rotation.md) - token+mTLS+old-key rotation 授权、StoreV4 原子事务、exact receipt、并发/重放/真实 bundle 证据。
- [2026-08-04 M5 W22-W24 implementation](logs/2026-08-04-m5-w22-w24-implementation.md) - 多设备、目录、SQLCipher、LocalOnlyV1 与数据生命周期基础切片。
- [2026-08-05 M5 W24 core registry lifecycle](logs/2026-08-05-m5-w24-core-registry-lifecycle.md) - credential/directory 单调 pin、registry 清理与当前重启持久化边界。
- [2026-08-05 M5 W25 core SQLCipher product wiring](logs/2026-08-05-m5-w25-core-sqlcipher-product-wiring.md) - OS-bound core key 到 DataCtl/FRB/Flutter 的显式产品链及 codegen 状态。
- [2026-08-05 M5 W26 real QUIC lifecycle](logs/2026-08-05-m5-w26-server-quic-lifecycle.md) - 真实 mTLS SPKI、rotation、directory、presence 与 revoke E2E。
- [2026-08-05 M5 server lifecycle resources](logs/2026-08-05-m5-server-lifecycle-resource-contract.md) - per-user audit、rate limit、wire bound 与 StoreV5 证据。
- [2026-08-05 M5 portable backup encryption](logs/2026-08-05-m5-02-portable-backup-encryption.md) - data-only 加密容器与原子 quarantine staging；产品 DB 导入仍未完成。
- [2026-08-05 M5 dispatcher runtime evidence](logs/2026-08-05-m5-03-dispatcher-runtime-evidence.md) - `drain_once` 并发、关闭、失败和重试组件证据；产品触发接线仍未完成。
- [2026-08-05 M7 W33 contract freeze](logs/2026-08-05-m7-w33-contract-freeze.md) - API/error/cancel/metrics/release 合同冻结与未实现边界。
- [2026-08-05 M7 W34 public error skeleton](logs/2026-08-05-m7-w34-public-error-skeleton.md) - Rust typed error/CAS 最小骨架；既有 API/FRB/RPC 适配仍未完成。
- [2026-08-05 M5 server lifecycle resource contract](logs/2026-08-05-m5-server-lifecycle-resource-contract.md) - StoreV5 per-actor presence/directory rate、每用户 4096/30 天 audit、presence read RPC 与 wire admission 边界。
- [2026-08-05 M7 W34 public error skeleton](logs/2026-08-05-m7-w34-public-error-skeleton.md) - baseline typed-error DTO、敏感字段拒绝、cancel control 与 exactly-one terminal CAS 纯 Rust 骨架。
- [2026-08-05 M5-M7 汇总验证](logs/2026-08-05-m5-m7-final-validation.md) - 汇总 Rust/Flutter/FRB/demo/debug 验证结果、Cap'n Proto 1.0.2 工具链阻塞与 M5 gate 剩余项。
- [2026-08-05 M5 W25 local-v2 产品打开接线](logs/2026-08-05-m5-w25-local-v2-product-wiring.md) - signed-in 双 SQLCipher DB、独立 key/路径、官方 FRB codegen 与跨层测试证据。
