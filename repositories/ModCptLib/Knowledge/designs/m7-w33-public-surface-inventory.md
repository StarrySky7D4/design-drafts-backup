# M7 W33 公共面清单与变更分类

> 状态：W33 合同，基于 2026-08-05 工作树的只读审计。本文是 inventory，不表示这些入口已达到稳定 SDK、兼容门已实现或可以发布。当前事实仍以源码、`PROTOCOL_REGISTRY.md` 与 `PROTOCOL_LIFECYCLE.md` 为准。

## 1. 分类规则

| 分类 | 本文判定 | 最低发布动作 |
|---|---|---|
| `breaking` | 删除/重命名公开项，改变类型、固定字节、认证结果、必需字段、持久语义，或让正确旧 consumer 无法无歧义处理 | major/新版本、双向拒绝矩阵、支持与恢复方案 |
| `additive` | 新独立入口或真正 optional 且 safely-ignorable 的值，不改变旧成功路径、安全语义或资源上限 | capability/版本登记、旧入口回归、缺能力零副作用 |
| `behavioral` | 形状不变，但错误、重试、授权、资源、并发或副作用改变 | 兼容矩阵、发布说明、回归与零副作用测试；必要时同时 `breaking` |
| `data-only` | 只改文档、测试向量、fixture 或内部数据，外部 parser/schema/API/业务结果均不变 | 用 diff 与重复输入证明无外部变化 |

同一变更命中多类时执行最严格门。SQLite 内容约束、FRB 错误形状和 canonical 签名输入不能按“只是实现细节”处理。

## 2. Rust SDK

| 面 | 当前 producer | 当前 consumer | 当前公开内容 | 默认分类 |
|---|---|---|---|---|
| `modcpt_identity` 0.1.0 | `rust/identity` | core、client、server、FRB | 强类型 ID；Realm/assertion/provision/DeviceHello；Envelope、receipt、direct、attachment、device lifecycle 的 canonical DTO/encoder/validator；Ed25519 helpers | canonical/layout/domain/version 改动 `breaking`；新独立 type `additive`；校验收紧 `behavioral`，旧端不安全时兼 `breaking` |
| `modcpt_pki` 0.1.0 | `rust/pki` | core、server | CA/CRL、CSR、leaf/SPKI、证书策略原语 | 证书策略/DER 语义 `behavioral` 或 `breaking`；新 helper 通常 `additive` |
| `modcpt_core` 0.1.0 | `rust/core` | FRB、msg_api、demo、debug-harness | `data_ctl`、delivery/directory/direct/attachment/transfer、`net`、`serve` 模块；crate root re-export delivery planning 与 P2P node/channel/config/errors | 公开 Rust 项变化按 semver；会话/授权/路由/持久副作用变化至少 `behavioral` |
| `modcpt_client` 0.1.0 | `rust/client` | FRB、测试/工具 | v1 account client、`ClientConfig`、`RustServerClient`、`AuthResult`、`PeerPresence`；local-v2 与 data lifecycle 模块 | 方法/DTO变化 `breaking`/`additive`；RPC/retry/TLS 行为 `behavioral`；local-v2格式另见 storage |
| `modcpt_server` 0.3.0 library | `rust/server` | server binary、client integration tests、部署者 | `cutover`、v1/v2 QUIC、RPC、Store/StoreV2 模块与状态类型 | RPC/wire/storage 取最严格分类；Rust-only helper 仍受 crate semver |
| `modcpt_msg_api` 0.1.0 | `rust/msg_api` | 自动化/LLM consumer | `DataCtl` 包装、消息/扩展消息发送接收 API | 签名/返回类型 `breaking`；新增独立 method `additive`；路由/副作用 `behavioral` |

所有 crate 仍处于 `0.x`（server 为 `0.3.0`），当前没有仓库级 SDK version handshake 或已验证 N-version 支持。W33 不把现有 `pub` 自动等同“稳定承诺”。

## 3. FRB 手写 Rust 面

producer 是 `flutter/rust/src/api/**`，consumer 是 FRB codegen。它再生产 generated Dart 与 native symbol glue；应用不得绕过生成边界。

| 域 | 公开手写面（按职责归组） | 当前错误/兼容事实 |
|---|---|---|
| node/lifecycle | `create_node`、存储感知 node 创建入口、`RustNode::{local_addr,close,event_stream,log_stream}`；UI/contacts/messaging/groups/files/room/secure-group 子句柄 | 多数 legacy 失败仍为 `Result<_, String>`；任何 node/DB/network API 前先通过 binding handshake |
| legacy transport | `connect`、`peers`、`disconnect`、`send_identity`、`set_local_user_id` | 暴露 session/address 语义；不是 business-ID 授权兼容层。删除/改义是 `breaking`，不得以 SessionId fallback 延寿 |
| business/UI | Contact/UserId lookup、messaging/file/group commands；UiApi snapshots；NodeEvent invalidation | DTO 字段/事件 kind/默认语义变化可能 `breaking` 或 `behavioral` |
| content/group candidates | typed ConversationId parser；Room/SecureGroup verification-gated state-machine views | 当前 Room/SecureGroup 不代表 M6/M7 产品已完成；所选产品汇合前不得扩成发布承诺 |
| account client | `create_server_client`、AuthResult/PeerPresence、account/token/address lookup operations | 仍把底层字符串错误抛给 Dart；token 与 TLS pin 生命周期属于安全行为 |
| diagnostics/crypto helpers | log stream、sign/verify、hash/announcement helpers | diagnostics 不是 metrics；安全 helper 的输入/输出变化至少 `behavioral` |

FRB 公开形状只允许修改手写 Rust API，再运行固定版本官方 codegen；禁止手改 generated Rust/Dart。

## 4. Generated Dart、native library 与应用抽象

| 面 | Producer | Consumer | 当前事实 | 变更分类 |
|---|---|---|---|---|
| `flutter/lib/src/rust/api*.dart` | FRB 2.12.0 codegen | `FrbNativeService`、AppState/UI | 生成 `RustNode`、各子句柄、DTO、Future/Stream；与手写 Rust 成对 | 任何生成形状漂移按对应手写变更分类；单独编辑无效且禁止 |
| `flutter/rust/src/frb_generated.rs` 与 `frb_generated*.dart` | FRB 2.12.0 codegen | native FFI runtime | ABI、wire codec、opaque handles；不是人工 API owner | native/generated 混配当前一律不支持；变更至少 paired-artifact `breaking` |
| `modcpt_frb` 0.1.0 (`cdylib`,`staticlib`) | `flutter/rust` build | Flutter Windows bundle | native library 与 generated Dart 被视为单一工件；无 build identity handshake | 拆分或允许混配必须先新增 handshake，属 `breaking`/`additive` 组合 |
| `NativeService` / `BridgeResult<T>` | `flutter/lib/services/native_service.dart` | AppState/UI/test doubles | Dart 应用内公共抽象；`BridgeResult.error` 仍是字符串 | 换 typed error DTO 为 API `breaking`，需受控迁移；新增不改变旧实现的方法可能 `additive` |
| `FrbNativeService` | Dart service | `NativeService` consumers | 命令转发、node/server/session/stream lifecycle；不应持业务数据 | 相同签名下 lifecycle/retry/cleanup 变化为 `behavioral` |

当前 generated Dart 与 native library 在 `RustLib.init()` 后、任何产品 API/DB/network/identity 工作前执行 contract/API/capability/profile/fingerprint/codegen handshake。发布矩阵仍只支持同一次构建的 exact app API 1 配对；旧/new 混配会在业务副作用前 fail-closed。

## 5. Server RPC

| 边界 | Producer | Consumer | 当前操作族 | 分类规则 |
|---|---|---|---|---|
| v1 JSON-over-QUIC，ALPN `modcpt-server/1` | server `quic.rs` | `modcpt_client` / FRB server client | `health`、account register/login/nickname/password/logout、`set_addr`、identity set/recover/revoke/get、lookup/lookup_nick | op/必需字段/响应形状/认证变化 `breaking`；安全可忽略 optional 才可能 `additive`；错误/retry 改义 `behavioral` |
| v2 JSON-over-QUIC+mTLS，ALPN `modcpt-server/2` | server `rpc_v2.rs`,`quic_v2.rs` | v2 provisioning/lifecycle clients与集成测试 | register/login、begin/complete provision、issue/rotate credential、directory、presence begin/renew、list/revoke device、publish/claim content prekey | token+mTLS principal、证书 SPKI、rotation receipt 等属于授权边界；改变均为 `breaking`+`behavioral` |
| JSON response envelope | server | clients | success `ok=true`；failure `ok=false,code,error` | 当前 code 分散且 message 可能来自内部 Display；迁移到统一 DTO 是 API/RPC `breaking`，需版本化或双入口 |

不得把 v1/v2 的相同字符串字段解释成相同授权语义。未知 op、未知版本或缺少认证材料必须在副作用前拒绝。

## 6. P2P/wire/canonical 面

| 面 | Producer | Consumer | 当前已登记边界 | 默认分类 |
|---|---|---|---|---|
| P2P ALPN/BIND | core P2P | remote core | `/2` BIND v2；`/3` DeviceHello+mTLS trusted session；`/1` 不兼容 | 布局、顺序、principal 产生条件为 `breaking`，必须新 major |
| route frame | DataRouter/serializer | remote DataRouter | 8-byte header；v1 `0x01`，Envelope frame `0x02`; `MsgType::Envelope=0xB0` | header/type/version改动 `breaking`；新 safely-ignorable type 才可 `additive` |
| Cap'n Proto message schema | core build/schema | core peers | committed schema+snapshot+generated binding | ordinal 只追加；重排/复用/改义 `breaking`；unknown security union fail-closed |
| canonical identity/content | identity crate | core/server/client | realm/assertion/provision/DeviceHello、Envelope、receipt、direct、attachment、device lifecycle | domain/field set/order/length/version/profile 改动 `breaking` |
| local backup wire | client data lifecycle | backup/restore path | `PortableBackupV1` 合同；尚非完整发布 archive pipeline | header/KDF/AEAD/manifest 实现后成为独立 versioned storage boundary；不得当 identity archive |

producer 与 consumer 必须记录精确版本、能力和拒绝向量。未知/未协商值不允许自动降级到旧 ALPN、IdentityFrame、SessionId 或 relay fallback。

## 7. SQLite/storage 面

| Store | Producer | Consumer | 当前版本/表族 | 当前安全与兼容事实 |
|---|---|---|---|---|
| core `DatabaseCtl` | core data layer | DataCtl/direct/attachment/delivery/UI | `user_version=11`；UI/contacts、legacy group/room、v2 messages/outbox/inbox、Olm account/session/bootstrap、receipt retry、lifecycle tombstone 与 monotonic restart pins | 只向前迁移；显式 encrypted open/migration 路径存在于当前工作树。DDL、约束和解释变化按 storage `breaking`/`additive`+`behavioral` |
| client `LocalV2Store` | client local-v2 | provisioning/restart identity load | 独立 `user_version=1`，`local_v2_identity` | 与 core DB 独立；显式 encrypted open；identity archive fail-closed，不得并入 data backup |
| server `StoreV2` | v2 server | v2 RPC/lifecycle | 独立 `user_version=5`；users/tokens/devices/provision/prekey/directory/presence/lifecycle/credential/rotation/rate/audit | server SQLCipher dependency不等于库已加密；当前 open/key contract须按实际代码审计，不得凭 feature 宣称静态加密 |
| server v1 `Store` | v1 server | v1 RPC | accounts/tokens/identity_credentials；无已冻结 `user_version` 迁移契约 | schema/授权持久语义变化属发布边界；cutover 与 StoreV2 隔离 |
| core serve stores | core serve/auth/key store | legacy/internal serve users | nodes/local_identity/crl/ca_keystore 等独立表族 | 不得与产品 core DB 或 server-v2 schema 混用；公开发布前需单独 version owner |

所有 migration 只前进，validation/转换/index/`user_version` 必须同一事务；旧 binary 不得回写更高 schema。transaction rollback 不是 backup/restore。

## 8. W33 版本与弃用冻结

- Rust crate、FRB/native/app、server RPC、wire 与 SQLite 分别版本化；app semver 不能替代 protocol/storage version。
- API breaking 使用 app/SDK major；additive feature 使用 minor；仅兼容 bugfix 使用 patch。安全行为收紧若旧 consumer 无法正确解释，也升级 major。
- legacy session/address FRB 面进入“待正式弃用”清单：`connect`、`peers`、`disconnect`、`send_identity`、`send_text_raw`、`send_file_raw`。W33 不删除它们；正式 deprecation 必须给替代入口、warning 与支持窗口，移除只在 major。
- generated Dart/native 只支持当前 exact binding identity；handshake 不等同于 mixed-version 支持。first-stable matrix 明示当前没有 N−1 签名工件，N+1 必须拒绝。
- 每个后续变更记录 boundary、owner、producer、consumer、分类、支持窗口、测试向量/fixture、回滚或前滚路径。
