# 协议、Schema、数据库与错误注册表

> **状态**：当前实现盘点；M4/M5/M6 gates 已满足。M6 W27-W32 产品组合、官方 FRB codegen、19+14 定向矩阵与 root full validation 11/11 已于 2026-08-06 通过。M7 业务域/发布工作仍不能视为已实现能力。
>
> **权威顺序**：源码、schema、迁移和可重复测试优先于本表；本表是这些分散事实的防碰撞索引。路线和未实施目标仍以 [Roadmap](ROADMAP.md) 及 [身份与寻址 v2](designs/identity-and-addressing-v2.md) 为准。

## 1. 登记规则

### 状态定义

| 状态 | 含义 | 复用规则 |
|---|---|---|
| `implemented` | 当前源码实际编码、解码、迁移、导出或拒绝该值。可选配置的路径会明确标注为可选。 | 不改变字节/存储/API 语义；不兼容变更必须先登记新的已批准版本边界。 |
| `reserved` | 当前源码已经命名或明确保留，但 core 没有把该值接入相应的业务消费者。 | 保留给当前语义，不能当作空闲编号重新分配；先更新本表并取得对应实现授权。 |
| `deprecated` | 源码仍保留名称或兼容入口以便明确拒绝/过渡，但新路径不得使用。 | 永不复用；保留历史语义和拒绝/迁移说明。 |

### Owner 与证据

- `Owner` 的前半段是编号登记单写者 `M1-02`；斜线后的模块是实现事实源。它不表示未分配的人员或未来任务已经获授权。
- 证据按 `路径::符号` 定位，避免依赖会漂移的行号。`M0-03` 与 `M0-04` 的已审阅事实分别见 [M0-03 卡](https://github.com/StarrySky7D4/ModCptLib/blob/add9e4024f8e10055508770aaf96c944aab6cceb/agents_work/tasks/M0-03-p2p-bind-version.md) 和 `database.rs` 的迁移测试。
- 本表没有分配数字型错误码。Rust enum 变体是当前稳定类别；展示文本和 FRB 的 `String` 不是可供客户端匹配的错误码。

### 变更与废弃

1. 新 wire、schema、迁移、FRB 公开形状或错误类别先更新本表，再实现生产消费者；未登记值只能作为 `proposed` 留在评审材料，不能进入 wire、storage 或生成代码。
2. 固定布局、字段序、字节序、canonical 编码或语义变化必须使用已批准的版本/major 边界，不能在同一 ALPN 下猜测布局。
3. 已废弃值不可回收。保留值也不是功能、权限或兼容承诺。
4. Cap'n Proto 字段编号只可追加，SQLite 版本只可向前迁移，公开 FRB 签名/DTO 改动按 breaking API 对待，直到协议生命周期另有正式裁定。

## 2. ALPN、Major 与 Capability

### P2P ALPN / major

| 值 | 状态 | Owner | 兼容与复用策略 | 源码证据 |
|---|---|---|---|---|
| `modcpt-p2p/3`，major `3` | `implemented` | M3-06 / `net::p2p` | v2 可信会话 ALPN。使用已有的 16-byte BIND 布局，加上 DeviceHello/credential 检疫（`verify_device_hello`，M3-05 纯契约）。要求 mTLS 和 `V2TrustedConfig`。不得回退到 `/2` 或 `IdentityFrame`。也可在 `/3` 下运行无检疫的纯明文 `/2` 流量。 | `rust/core/src/net/p2p.rs::P2P_ALPN_V3`、`P2P_PROTOCOL_MAJOR_V3`、`require_supported_alpn`、`validate_alpn_auth` |
| `modcpt-p2p/2`，major `2` | `implemented` | M1-02 / `net::p2p`（M0-03） | v1 传输检疫 ALPN（`TransportAuthenticator` / `IdentityFrame`）。承装 16-byte BIND。产品路径只通过 `connect_verified`（`/3`）建立可信 PeerPrincipal；`/2` 不能产生 v2 授权主体。 | `rust/core/src/net/p2p.rs::P2P_ALPN_V2`、`P2P_PROTOCOL_MAJOR_V2`、`require_supported_alpn` |
| `modcpt-p2p/1`，major `1` | `deprecated` | M1-02 / `net::p2p`（M0-03） | 仅为命名拒绝值保留。旧 8-byte `LinkId` 布局不能协商；永不复用该 ALPN 或 major。 | `rust/core/src/net/p2p.rs::P2P_ALPN_V1`、`require_supported_alpn`; `agents_work/tasks/M0-03-p2p-bind-version.md::固定契约` |

`P2pConfig::default()` 使用 `/2`。`with_v2_trusted` 自动切换到 `/3` 并启用 mTLS。任何未知 ALPN 在 endpoint 创建前以 `P2pError::UnsupportedProtocol` 失败。

### FRB native/generated runtime binding

这些值属于本机 generated Dart/native library 的启动兼容门，不是网络 ALPN、业务 capability、身份 assertion 或 artifact 签名。真实工件来源仍由 release digest、平台签名与 attestation 验证。

| 值 | 状态 | Owner | 兼容与复用策略 | 源码证据 |
|---|---|---|---|---|
| binding contract `1`；app/native API `1`；native app window `1..=1` | `implemented` | M7-02 / FRB binding | `FrbRuntime.init` 在任何产品工作前要求 window 命中；too-old/too-new 均 `incompatible_version`。扩大到 N−1 前必须增加真实旧工件 fixture。 | `flutter/rust/src/api/binding.rs`、`flutter/lib/services/frb_runtime.dart` |
| profile `modcpt.frb.binding.v1`；FRB `2.12.0`；fingerprint `7be2c617...b6e96ce` | `implemented` | M7-02 / FRB binding | fingerprint 是 canonical public profile manifest 的 SHA-256，不是签名；profile/fingerprint/codegen 任一 mismatch fail-closed。 | `binding.rs::fingerprint_matches_the_canonical_public_profile_manifest`、Dart compatibility tests |
| binding capability bits `0..3` = typed operation、FileTransferV2 cancel ACK、server operation result、product v2 node；mask `0x0f` | `implemented` | M7-02 / FRB binding | app required bits 必须为 native mask 子集；额外 native bits 可忽略。不可复用已登记 bit。 | `binding.rs::BINDING_CAPABILITIES_V1`、`check_native_binding_compatibility_v1` |

### 身份 assertion capability mask

这些位属于 `IdentityAssertion.capabilities` 的 canonical assertion 编码，不是 ALPN 协商字段，也不证明相应 Group/Room wire 消费者已经接入。

| 值 | 状态 | Owner | 兼容与复用策略 | 源码证据 |
|---|---|---|---|---|
| `CAP_GROUP_V2 = 1 << 0` (`0x0000000000000001`) | `implemented` | M1-02 / `modcpt_identity` | 只可作为已验证 assertion mask 的 bit 0；不得重解释或复用为另一能力。 | `rust/identity/src/lib.rs::CAP_GROUP_V2`、`IdentityAssertion::validate_fields` |
| `CAP_ROOM_V1 = 1 << 1` (`0x0000000000000002`) | `implemented` | M1-02 / `modcpt_identity` | 只可作为已验证 assertion mask 的 bit 1；不得重解释或复用为另一能力。 | `rust/identity/src/lib.rs::CAP_ROOM_V1`、`IdentityAssertion::validate_fields` |
| `ALL_CAPABILITIES = 0x0000000000000003` | `implemented` | M1-02 / `modcpt_identity` | 是上述两个已登记 bit 的允许掩码，不是第三个 capability；未知/零 mask 被拒绝。未登记 bit 不在本表分配。 | `rust/identity/src/lib.rs::ALL_CAPABILITIES`、`IdentityAssertion::validate_fields` |

### Realm manifest / ServerProfileV2 canonical contract

这些值只属于 `modcpt_identity` 的 canonical profile/manifest 编码，不是 P2P ALPN、Cap'n Proto、SQLite 或 FRB 编号。v1 `realm_id()` 和 `ServerProfile` 保持当前语义，不能将新值当作其兼容替换。

| 值 | 状态 | Owner | 兼容与复用策略 | 源码证据 |
|---|---|---|---|---|
| manifest version `1`，domain `modcpt.identity.realm-manifest.v1` | `implemented` | M1-02 / `modcpt_identity::realm` | root 签名的 realm manifest canonical 版本；未知版本 fail-closed，不复用为 assertion/P2P version。 | `rust/identity/src/realm.rs::MANIFEST_VERSION`、`MANIFEST_DOMAIN` |
| ServerProfileV2 version `2` | `implemented` | M1-02 / `modcpt_identity::realm` | 仅 v2 profile；不能解释为 v1 `IDENTITY_VERSION` 或自动升级当前 `ServerProfile`。 | `rust/identity/src/realm.rs::SERVER_PROFILE_V2_VERSION` |
| signer usage `0x01` `AssertionIssuance` | `implemented` | M1-02 / `modcpt_identity::realm` | manifest 中在线 signer 的唯一当前用途；未知 usage fail-closed，不分配 device/transport 权限。 | `rust/identity/src/realm.rs::SignerUsage` |
| maximum authorized signers `2`；maximum overlap `ASSERTION_TTL_SECS + CLOCK_SKEW_SECS`（24h05m） | `implemented` | M3-02 / `modcpt_identity::realm` | 仅允许旧/新 signer 的有界正常轮换；拒绝第三 signer 或超期 overlap，撤销附着于有界条目。 | `rust/identity/src/realm.rs::MAX_AUTHORIZED_SIGNERS`、`MAX_SIGNER_OVERLAP_SECS` |

### Device provision canonical contract

| 值 | 状态 | Owner | 兼容与复用策略 | 源码证据 |
|---|---|---|---|---|
| challenge domain `modcpt.identity.provision-challenge.v1` | `implemented` | M3-03 / `modcpt_identity::provision` | 只绑定 RealmId、RequestId、challenge 与 expiry；不得复用 v1 possession/rotation 域。 | `rust/identity/src/provision.rs::PROVISION_CHALLENGE_DOMAIN` |
| request domain `modcpt.identity.provision-request.v1` | `implemented` | M3-03 / `modcpt_identity::provision` | 绑定 realm/user/device/request/challenge/CSR hash/key/capability；私钥不编码，未知或缺失输入由消费者副作用前拒绝。 | `rust/identity/src/provision.rs::PROVISION_REQUEST_DOMAIN` |
| devices/user `8`；pending/user `2`；challenge TTL `600s` | `implemented`（contract limits） | M3-03 / `modcpt_identity::provision` | 已裁定 admission 上限；server schema/RPC 尚未消费，不能描述为已生效服务端限流。 | `MAX_DEVICES_PER_USER`、`MAX_PENDING_PROVISIONS_PER_USER`、`PROVISION_CHALLENGE_TTL_SECS` |

### DeviceHello / trusted-session canonical contract（M3-05）

这些值只属于 `modcpt_identity::device_hello` 的纯可信会话 canonical 编码，不是 P2P ALPN、Cap'n Proto、SQLite、FRB、control tag 或 application `MsgType`。`InstanceId` 与 lease epoch 仅为 host 供给绑定字段，不实现 presence lease/TTL/renewal/仲裁。

| 值 | 状态 | Owner | 兼容与复用策略 | 源码证据 |
|---|---|---|---|---|
| DeviceHello `protocolVersion = 1`，domain `modcpt.identity.device-hello.v1` | `implemented`（pure contract） | M3-05 / `modcpt_identity::device_hello` | 独立命名空间；不是 ALPN major（`2`）、quarantine payload version（`1`）、manifest version（`1`）、profile version（`2`）或 application header version（`1`）。未知/未来版本 fail-closed。 | `rust/identity/src/device_hello.rs::DEVICE_HELLO_PROTOCOL_VERSION`、`DEVICE_HELLO_DOMAIN` |
| credential assertion domain `modcpt.identity.credential-assertion.v1`，version `1` | `implemented`（pure contract） | M3-05 / `modcpt_identity::device_hello` | manifest 授权 `AssertionIssuance` signer 签发的设备 credential 绑定 canonical 版本；不复用 v1 `assertion.v1` 或 provision 域。 | `rust/identity/src/device_hello.rs::CREDENTIAL_ASSERTION_DOMAIN`、`CREDENTIAL_VERSION`、`CredentialAssertion` |
| credential status domain `modcpt.identity.credential-status.v1`，version `1` | `implemented`（pure contract） | M3-05 / `modcpt_identity::device_hello` | 同一授权 signer 签发的可刷新状态；`Active(0)`/`Revoked(1)`；状态必须与 assertion 同 credential/key_version，否则 fail-closed。 | `rust/identity/src/device_hello.rs::CREDENTIAL_STATUS_DOMAIN`、`CredentialStatusProof`、`CredentialStatus` |
| SPKI hash `32` 字节；nonce `32` 字节；role `0/1` | `implemented`（pure contract） | M3-05 / `modcpt_identity::device_hello` | SPKI hash 由证书 DER 的 SubjectPublicKeyInfo 经 SHA-256 预计算；nonce/role 进设备签名防重放/反射。长度或 role 越界 fail-closed。 | `rust/identity/src/device_hello.rs::spki_hash_from_cert`、`DEVICE_HELLO_NONCE_LEN`、`HelloRole` |
| 复用常量 `ALL_CAPABILITIES`、`ASSERTION_TTL_SECS`、`CLOCK_SKEW_SECS`、`MAX_P2P_CERT_BYTES`、`MAX_SIGNER_OVERLAP_SECS` | `implemented`（pure contract） | M3-05 / `modcpt_identity::device_hello` | 不发明新数值上限；仅新增有界 wire 长度上限 `MAX_CREDENTIAL_ASSERTION_WIRE`/`MAX_CREDENTIAL_STATUS_WIRE`。 | `rust/identity/src/device_hello.rs` |

### Device lifecycle / directory / presence canonical contract（M5-01）

这些值属于 `modcpt_identity::device_lifecycle` 的纯 canonical 域；不分配
ALPN、Cap'n Proto、application MsgType、SQLite 或 FRB 编号。

| 值 | 状态 | Owner | 兼容与复用策略 | 源码证据 |
|---|---|---|---|---|
| lifecycle version `1`；domains `modcpt.identity.instance-begin.v1`、`instance-renew.v1`、`presence-lease.v1`、`device-directory.v1`、`device-directory-entries.v1`、`credential-rotation-request.v1` | `implemented`（pure contract） | M5-01 / `modcpt_identity::device_lifecycle` | 固定字段序、长度前缀和大端整数；未知版本/domain、尾随、截断、签名/realm/principal mismatch 均 fail-closed。改变字段或 signer 语义需要新 version/domain。 | `DEVICE_LIFECYCLE_VERSION` 与各 `*_DOMAIN`；canonical/tamper tests |
| presence TTL `90s`、renew hint `30s`、new-signature skew `30s`、Host candidates `8`、presence wire `2048` bytes、directory wire `16384` bytes | `implemented`（pure contract） | M5-01 / identity | `now < expires_at` 才有效；candidate canonical sorted/unique，拒绝 port 0、unspecified、multicast 和 IPv6 link-local。directory 最多 8 个 current Active device entries。 | `PRESENCE_*`、`MAX_HOST_CANDIDATES`、`MAX_*_WIRE_BYTES` |
| `MAX_DIRECT_FANOUT_RECIPIENTS = 15` | `implemented`（pure contract） | M5-01 / identity direct+lifecycle | 逻辑集合精确为 peer active devices 最多 8 + origin 其他 devices 最多 7；同一 MessageId 和完整 recipient-set v1 commitment。单个 Envelope cap 仍为 8，direct/attachment 每 wire 仍精确一个 recipient；禁止截断或拆第二 logical message。<=8 的 commitment bytes 不变。 | `direct::MAX_DIRECT_FANOUT_RECIPIENTS`、`FrozenDirectFanout` |

### 不可变信封 canonical 契约（M4-01）

这些值只属于 `modcpt_identity` 的 canonical 业务 ID 与信封编码，不是 P2P ALPN、Cap'n Proto、SQLite、FRB、control tag 或 application `MsgType` 编号。信封签名只覆盖本区登记的 canonical field set，绝不签 JSON、Cap'n Proto bytes 或字符串拼接（M2-03 §4.1）。E2EE 内容 commitment（HeaderBytes/EnvelopeCommitment/附件 descriptor）由 M4-04/M4-05 在本区登记后接入。

| 值 | 状态 | Owner | 兼容与复用策略 | 源码证据 |
|---|---|---|---|---|
| `ConversationId` 文本编码 `d:` + 64 小写 hex（Direct）／`g:` + 规范 UUIDv7 文本（Group）；canonical 签名字节 = kind byte `0x01`/`0x02` + 内层 ID 字节（32/16） | `implemented`（pure contract） | M4-01 / `modcpt_identity::ids` | 独立文本命名空间；`d:` 与 `g:` 前缀不可互换，UUIDv7 文本不得解析为 Direct、64-hex 不得解析为 Group；未知 kind/非规范文本 fail-closed。 | `rust/identity/src/ids.rs::ConversationId`、`CONVERSATION_KIND_DIRECT`、`CONVERSATION_KIND_GROUP` |
| envelope domain `modcpt.identity.envelope.v1`、envelope version `1` | `implemented`（pure contract） | M4-01 / `modcpt_identity::envelope` | 只绑定 RealmId、ConversationId、origin UserId/DeviceId、MessageId、content kind、createdAtMs、canonical 排序 recipient set 与 payload；未知 version/kind/长度 fail-closed。 | `rust/identity/src/envelope.rs::ENVELOPE_DOMAIN`、`ENVELOPE_VERSION` |
| envelope wire/构造上限 `8 MiB - 16 KiB`；recipient cap `8` | `implemented`（pure contract） | M4-01 / `modcpt_identity::envelope` | 8 MiB application frame 减去既有 16 KiB control-record cap，给 Cap'n Proto 和 8-byte route header 留出可传输 headroom；超 8 个 canonical recipient 在签名、decode、route 和存储前拒绝。 | `MAX_ENVELOPE_WIRE_BYTES`、`MAX_ENVELOPE_RECIPIENTS`；`p2p_gateway::MAX_FRAME_SIZE`、`p2p::MAX_CTRL_MSG_SIZE` |
| content kind `1` = legacy opaque application content | `implemented`（非 direct E2EE） | M4-01 / `modcpt_identity::envelope` | 保留 M4-01/M4-03 foundation 语义；不得解释为 direct encrypted content、不得进入 M4-05 post-decrypt subscriber。 | `rust/identity/src/envelope.rs::CONTENT_KIND_APPLICATION` |
| content kind `2` = delivery receipt v1 | `implemented` | M4-03 / `modcpt_identity::receipt` | receipt 作为 recipient-device-signed immutable Envelope payload；只表达该 recipient device 已接受原 envelope，不代表全部设备或已读。 | `rust/identity/src/envelope.rs::CONTENT_KIND_DELIVERY_RECEIPT` |
| content kind `3` = native Olm v2 bootstrap control | `implemented` | M4-05 / `modcpt_identity::direct` | 只承载 payload-free `Offer/Candidates/Confirm/Ready`；不能携带业务 plaintext，未知/错 phase 在副作用前拒绝。 | `CONTENT_KIND_DIRECT_BOOTSTRAP`、`BootstrapControl` |
| content kind `4` = native Olm v2 direct message | `implemented` | M4-05 / `modcpt_identity::direct` | 每个 Envelope 精确一个 recipient device；payload 是 canonical `DirectCiphertext`，只有 `Active` winner session 可 seal/open。不得复制到多个 device 或解释为 kind `1`。 | `CONTENT_KIND_DIRECT_MESSAGE`、`DirectCiphertext`、`DirectE2eeRuntime` |
| content kind `5` = attachment ciphertext chunk v1 | `implemented` | M4-04-I1 / identity + core | 每个 Envelope 精确一个 recipient device；payload 是 canonical public `AttachmentChunk`，不含 file key/nonce/plaintext。只走 authenticated Envelope ingress/egress，不分配新 application MsgType、不回退 legacy FileCtl。 | `CONTENT_KIND_ATTACHMENT_CHUNK`、`AttachmentChunk`、`AttachmentRuntime` |
| direct content version `1` / profile `1` = `vodozemac =0.10.0` `SessionConfig::version_2()` | `implemented` | M4-05 / identity + core | native Olm v2 AES-256-CBC + full HMAC-SHA-256；不兼容 Olm v1 truncated-MAC，也不声称实现历史 ChaCha/external-AD proposal。unknown profile/version fail-closed，无 fallback。 | `DIRECT_CONTENT_VERSION`、`DIRECT_PROFILE_OLM_V2`、`rust/core/Cargo.toml` |
| direct canonical domains：`modcpt.content.olm-v2.prekey-record.v1`、`prekey-bundle.v1`、`prekey-claim.v1`、`plaintext.v1`、`ciphertext.v1`、`envelope-commitment.v1`、`bootstrap-control.v1`；recipient set `modcpt.content.recipient-set.v1` | `implemented` | M4-05 / `modcpt_identity::direct` | 全部使用长度前缀、固定字段序与大端整数；JSON/Cap'n Proto/map iteration 不进入签名或 commitment。改变任一 domain/field set 是新 profile/version。 | `rust/identity/src/direct.rs` constants + fixed/tamper tests |
| delivery receipt domain `modcpt.delivery.receipt.v1` | `implemented` | M4-03 / `modcpt_identity::receipt` | canonical fields：version、realm、original origin device、original MessageId、recipient device、M2-03 `EnvelopeCommitment`、acceptedAtMs；receipt Envelope origin/signature 必须绑定 recipient device。wire digest 只用于本地 dedup，不得冒充 commitment。 | `rust/identity/src/receipt.rs::DELIVERY_RECEIPT_DOMAIN` |
| attachment ciphertext digest domain `modcpt.content.attachment-ciphertext.v1` | `implemented`（pure contract） | M4-04-C1 / `transfer_v2` | `SHA-256(domain || TransferId || chunk_count_u32_be || (chunk_index_u32_be || SHA-256(chunk_ciphertext))* )`；只覆盖 opaque ciphertext，不放置 file key/nonce/plaintext，不分配 wire number。 | `rust/core/src/transfer_v2.rs::ATTACHMENT_CIPHERTEXT_DIGEST_DOMAIN` |
| attachment version `1` / profile `1` = ChaCha20-Poly1305；tag 16 bytes；nonce = random prefix 8 bytes + chunk index u32 BE | `implemented` | M4-04-I1 / identity + core | 每 TransferId 使用独立 CSPRNG file key/prefix；unknown version/profile fail-closed，无 crypto fallback。secret descriptor 逐 recipient 通过 M4-05 Olm v2 direct plaintext 传递。 | `ATTACHMENT_VERSION`、`ATTACHMENT_PROFILE_CHACHA20_POLY1305`、`seal_attachment` |
| attachment canonical domains：`modcpt.content.attachment-context.v1`、`attachment-descriptor.v1`、`attachment-chunk-aad.v1`、`attachment-chunk.v1`、`attachment-chunk-commitment.v1` | `implemented` | M4-04-I1 / `modcpt_identity::attachment` | context commitment 不含最终 ciphertext digest并进入 AEAD AD；final descriptor commitment 覆盖完整 descriptor/digest，消除循环依赖。全部使用长度前缀、固定字段序与大端整数；改变任一 domain/field set 需要新 version/profile。 | `rust/identity/src/attachment.rs` constants + canonical/tamper tests |

### M4 delivery and transfer resource limits

下列上限均由当前消费者和拒绝测试执行。native Olm 内部限制保持上游精确值，不通过 fork 或 private API 修改。

| 值 | 状态 | Owner | 兼容与复用策略 | 证据 / 依据 |
|---|---|---|---|---|
| single Envelope recipient cap `8`；logical `DeliveryPlan` cap `15` | `implemented` | M4-01 + M5-01 / envelope, DeliveryPlanner | Envelope canonical/decode/storage 仍在 8 前拒绝；逻辑 plan 为逐 recipient direct/attachment 扩到 peer 8 + self 7。generic Envelope 发送仍要求 envelope recipient set 与 plan 完全一致，因此不能借 plan cap 绕过 wire cap。 | `MAX_ENVELOPE_RECIPIENTS=8`；`delivery_v2::MAX_RECIPIENTS_PER_MESSAGE=MAX_DIRECT_FANOUT_RECIPIENTS=15` |
| logical direct fan-out `15`；directory devices `8`；audit `4096/user`、`30d`；mutation/renew `6/device/min`、directory/presence read `60/device/min` | `implemented` | M5-01 | 15 不改变 Envelope/DeliveryPlanner 的单-wire cap 8。StoreV5 使用独立 per-actor lifecycle rate namespace；presence mutation 6/min，directory/presence read 各 60/min；audit 每 user 4096 且严格 30 天。prekey publish/claim 继续使用原独立 6/60 namespace。 | `device_lifecycle.rs`、`store_v2.rs`、`M5_PLAN.md` D0-A/C |
| `MAX_OUTBOX_ENTRIES = MAX_DEDUP_CACHE_ENTRIES = 4096`；`ACK_TIMEOUT_SECS = 600`；`MAX_RETRY_ATTEMPTS = 5`；`OUTBOX_ENTRY_MAX_AGE_SECS = 2592000` | `implemented` | M4-03 / delivery state | outbox/inbox、dedup、receipt 和 retry 同时有界；重试到第 5 次后终止，30 天后过期。 | `rust/core/src/delivery_state_v2.rs`；4096 对齐已登记事件日志 cap，600s 对齐 provision challenge TTL。 |
| LocalOnlyV1 due scan / exact optimistic claim / exact subset route | `implemented`（W22 foundation） | M5-03 / core delivery+router | due scan 只读轻量字段；只有 authoritative `Routed` exact candidates 才递增 attempt，Offline/Rejected 不变。direct retry 校验完整 frozen ciphertext set 后只发 DeviceId subset；attachment 使用 exact `(MessageId,DeviceId)` target。无 mailbox/relay/push、后台拨号或 legacy fallback。 | `DueOutboxCandidate`、`claim_exact_due`、`send_direct_envelopes_to_devices`、`send_attachment_chunk_envelopes_to_targets`；定向竞态/expiry/subset tests |
| `MAX_TRANSFER_BYTES = MAX_REASSEMBLY_BUFFER_BYTES = 8 MiB`；`MAX_CHUNK_BYTES = 16 KiB`；`MAX_CHUNKS_PER_TRANSFER = 8192`；`MAX_CONCURRENT_TRANSFERS_PER_CONVERSATION = 4`；`MAX_ACTIVE_TRANSFERS_GLOBAL = 256`；`MAX_ACTIVE_CONVERSATIONS = 64`；`MAX_CHUNK_ENTRIES_GLOBAL = 65536`；idle TTL `60s` | `implemented`（pure foundation） | M4-04 / transfer_v2 | TransferId 重组、取消和恢复状态必须先检验 bytes、per-conversation/global transfer、conversation 和 chunk-entry metadata cap；零字节 chunk 在创建 state 前拒绝；60s idle expiry 释放 chunk bytes、entry、slot 和 dedup binding。 | 用户 2026-07-31 批准：256 复用 legacy `MAX_PENDING_SENDERS`，64 复用 `p2p::MAX_CONCURRENT_INCOMING`，65536 = 8 registered recipient devices × 8192 chunks；其余复用既有 frame/control/reassembly cap。 |
| attachment plaintext chunk `16368` bytes、plaintext total `8,380,416` bytes、name `255` UTF-8 bytes、MIME `127` ASCII bytes；completion receipt dedup TTL `60s` | `implemented` | M4-04-I1 / identity + core | plaintext chunk = 16 KiB ciphertext cap - 16-byte tag；total plaintext 当前限制为 512 个最大块，确保加 tag 后不超过 8 MiB。active tracker + pending completion receipt 共享 global 256 cap；receipt durable queue 后立即释放。 | `MAX_ATTACHMENT_*`、`AttachmentRuntime` tests |
| OTK/public bundle：`MAX_ONE_TIME_PREKEYS=50`、bundle wire `16 KiB`、TTL `604800s`；claim `60/device/min`、publish `6/device/min` | `implemented` | M4-05 / identity + core + server | 50 来自 vodozemac public server-key recommendation；16 KiB 复用已登记 control-record cap；7 天限制 stale bundle；runtime 可原子刷新 fallback/新 OTK bundle。server 在 publish transaction 提交前强制每设备未领取且未过期的 OTK 聚合不超过 50，越界时 bundle、keys 与 rate 整体回滚；已领取后可补充至 50。claim transaction 保证一个 OTK 只有一个 winner，fallback 只在 OTK 空时复用。 | `direct.rs`、`DirectE2eeRuntime::refresh_prekey_bundle`、`store_v2.rs`；refresh/aggregate publish rollback/concurrent claim/exhaustion/rate/revoke tests |
| direct plaintext `1 MiB`；Olm message `2 MiB`；encrypted account/session object `256 KiB`；aggregate encrypted state `64 MiB` | `implemented` | M4-05 / identity + core SQLite v8 | 都低于 envelope/frame bound；object/global 两级拒绝，数据库只保存 encrypted pickle/ciphertext。 | `MAX_DIRECT_PLAINTEXT_BYTES`、`MAX_OLM_MESSAGE_BYTES`、`MAX_ENCRYPTED_CONTENT_STATE_BYTES*` |
| content sessions `4096`；pending bootstraps `256`；candidate/device pair `2` | `implemented` | M4-05 / core | 4096 对齐 outbox/dedup cap；256 对齐 active transfer global cap；2 来自双方各最多一个 offer。达到上限前拒绝，无未持久化 session。 | `MAX_CONTENT_SESSIONS`、`MAX_PENDING_BOOTSTRAPS`、`BootstrapState` |
| native Olm receive limits：message gap `2000`、skipped keys `40/receive chain`、receive chains `5` | `implemented`（upstream profile） | M4-05 / vodozemac 0.10.0 | 由精确依赖内部执行；项目测试覆盖乱序正向和 gap 超限，不复制或放宽实现。升级需 D3 re-review。 | exact Cargo pin；`native_olm_v2_skipped_keys_tamper_and_profile_downgrade_fail_closed` |

### 检疫 payload version

| 值 | 状态 | Owner | 兼容与复用策略 | 源码证据 |
|---|---|---|---|---|
| quarantine `protocolVersion = 1` | `implemented` | M1-02 / `data_ctl::transport_identity` | 仅适用于 `IdentityHello`/`IdentityProof` quarantine payload；值不匹配 fail-closed，不能替代 ALPN major。 | `rust/core/src/data_ctl/transport_identity.rs::PROTOCOL_VERSION`、`TransportAuthenticator`; `rust/core/src/net/p2p.rs::authenticate_initiator` |

## 3. P2P 控制面与固定布局

### Stream discriminator

| 值 | 状态 | Owner | 兼容与复用策略 | 源码证据 |
|---|---|---|---|---|
| `0x00` `Control` | `implemented` | M1-02 / `net::p2p` | 首字节只在 P2P stream namespace 中解释；不得与 application `MsgType` 数值混用或复用。 | `rust/core/src/net/p2p.rs::StreamType`、`send_ctrl_msg`、`spawn_stream_dispatcher` |
| `0x01` `AppData` | `implemented` | M1-02 / `net::p2p` | 应用可靠流前缀；不得改变为控制布局。 | `rust/core/src/net/p2p.rs::StreamType`、`ReliableChannel::open` |
| `0x02` `Quarantine` | `implemented`（可选 `transport_auth`） | M1-02 / `net::p2p` | 仅在配置 `P2pConfig::transport_auth` 时走认证前隔离流；不得把它当作普通应用流或复用。 | `rust/core/src/net/p2p.rs::StreamType`、`P2pConfig::transport_auth`、`authenticate_initiator` |

### 控制帧 tag 与布局

控制 tag、stream discriminator、application `MsgType` 是三个独立 namespace；例如三个 namespace 都可以出现 `0x01`，这不是重复分配。

| Tag / layout | 状态 | Owner | 兼容与复用策略 | 源码证据 |
|---|---|---|---|---|
| `0x01` `BIND`，34 bytes：`tag:1 | TransportSessionId(UUIDv4):16 | LinkId(UUIDv4):16 | role:1` | `implemented` | M1-02 / `net::p2p`（M0-03） | 仅 v2 ALPN；精确长度、字段顺序和 UUID version 均固定。截断、尾随或无效字段是 `MalformedFrame`，不得重用 tag。 | `rust/core/src/net/p2p.rs::BIND_FRAME_LEN`、`ControlMsg::Bind`、`ControlMsg::decode` |
| `0x02` `BIND_ACK`，同一 34-byte layout | `implemented` | M1-02 / `net::p2p`（M0-03） | 仅回应 BIND；`LinkId` 与 role 必须匹配。不得在 v1 或其他布局中复用。 | `rust/core/src/net/p2p.rs::ControlMsg::BindAck`、`bind_handshake` |
| BIND role `0x01` `Active` / `0x02` `Standby` | `implemented` | M1-02 / `net::p2p`（M0-03） | 仅为 BIND/BIND_ACK 最后一字节；其他值畸形，两个值均不得重解释。 | `rust/core/src/net/p2p.rs::LINK_ROLE_ACTIVE_TAG`、`LINK_ROLE_STANDBY_TAG`、`decode_role` |
| `0x03` `Switch`：`[tag][0x00]` 为 2 bytes；或 `[tag][0x01][0x04][IPv4:4][port BE:2]` 为 9 bytes；或 `[tag][0x01][0x06][IPv6:16][port BE:2]` 为 21 bytes | `implemented` | M1-02 / `net::p2p` | 三种精确布局；未知 presence/family、截断或尾随均畸形。不得复用 tag 或地址 family tag。 | `rust/core/src/net/p2p.rs::ControlMsg::Switch`、`ControlMsg::decode`、`P2PChannel::request_migration` |
| `0x04` `IdentityHello` raw control payload | `reserved` | M1-02 / `net::p2p` | `ControlMsg` 可编/解码且 payload 非空，但当前 P2P 流程不发送/消费该 control variant；实际认证使用 `Quarantine` 的 length-prefixed Cap'n Proto frame。不得把 tag 当作已接入 DeviceHello。 | `rust/core/src/net/p2p.rs::CTRL_TAG_IDENTITY_HELLO`、`ControlMsg::IdentityHello`；`authenticate_initiator` |
| `0x05` `IdentityProof` raw control payload | `reserved` | M1-02 / `net::p2p` | 同 `0x04`；保留现有语义，不能重分配。 | `rust/core/src/net/p2p.rs::CTRL_TAG_IDENTITY_PROOF`、`ControlMsg::IdentityProof`；`authenticate_responder` |
| `0xFF` `Close`，1 byte | `implemented` | M1-02 / `net::p2p` | 精确单字节；控制循环识别关闭语义。不得复用为扩展 tag。 | `rust/core/src/net/p2p.rs::CTRL_TAG_CLOSE`、`ControlMsg::Close`、`ControlMsg::decode` |
| Quarantine record：`length:u32 BE | payload`，`1..=16384` bytes | `implemented`（可选 `transport_auth`） | M1-02 / `net::p2p` | length 为独立 record layout，不与 control tag 混用；零或超限拒绝，最大值不得静默增大。 | `rust/core/src/net/p2p.rs::MAX_CTRL_MSG_SIZE`、`write_quarantine_frame`、`read_quarantine_frame` |

未知 control tag 映射为 `UnsupportedProtocol`；已知 tag 的形状错误映射为 `MalformedFrame`。M0-03 的固定向量、跨版本、长度、尾随和重复 `LinkId` 测试是此布局的验收证据。

## 4. Application Header、MsgType 与 Cap'n Proto

### Application header

| 值 | 状态 | Owner | 兼容与复用策略 | 源码证据 |
|---|---|---|---|---|
| 8-byte header：`msg_type:u8 | version:u8 | flags:u16 BE | payload_len:u32 BE`；current version `0x01` | `implemented` | M1-02 / `net::route`、`data_ctl::convert::serializer` | 字段序和 BE 编码固定；`Message::from_bytes` 拒绝非 `0x01` 或长度不匹配。`flags` 没有已登记 bit，不能自行分配。 | `rust/core/src/net/route.rs::MessageHeader`、`Message::from_bytes`; `rust/core/src/data_ctl/convert/serializer.rs::Header::CURRENT_VERSION` |
| 同一 header 中 version `0x02`（仅 immutable envelope frame 使用，M4-01） | `implemented` | M4-01 / `net::route` | 只允许 `MsgType::Envelope(0xB0)` 携带 version `0x02`；v1 消息类型保持 `0x01`，绝不被重新解释为 v2；Envelope 帧携带 `0x01` 直接拒绝。v1 解析路径（`Message::from_bytes`）继续拒绝非 `0x01`，v2 Envelope 帧在 v1 路径返回 `None`（零副作用）。`0x02` 不是 ALPN major、capnp Header.version 或 DeviceHello protocolVersion。 | `rust/core/src/net/route.rs::MessageHeader::ENVELOPE_VERSION`、`EnvelopeFrame::from_bytes` |

### MsgType registry

`implemented` 表示当前 application router/DataCtl 有实际 handler、subscriber 或 sender/receiver组合。`reserved` 表示 enum 已命名、但没有在此 application routing path 接入消费者；独立 raw-gateway 的 broadcast/DHT 代码不使其同名 `MsgType` 自动变成已接入。

| Value | 名称 | 状态 | Owner | 兼容与复用策略 | 源码证据 |
|---|---|---|---|---|---|
| `0x00` | `Ping` | `implemented` | M1-02 / `net::route` | 内建 Ping handler；固定含义，不复用。 | `rust/core/src/net/route.rs::MsgType::Ping`、`PingHandler` |
| `0x01` | `Pong` | `implemented` | M1-02 / `net::route` | 仅 Ping 的回应；固定含义，不复用。 | `rust/core/src/net/route.rs::MsgType::Pong`、`Message::pong` |
| `0x02` | `IdentityHello` | `reserved` | M1-02 / `net::route` | 已命名 application tag，但认证当前走 quarantine stream，不可当作已完成 v2 授权或重分配。 | `rust/core/src/net/route.rs::MsgType::IdentityHello`; `rust/core/src/data_ctl/transport_identity.rs` 模块说明 |
| `0x03` | `IdentityProof` | `reserved` | M1-02 / `net::route` | 同 `0x02`；保留，不复用。 | `rust/core/src/net/route.rs::MsgType::IdentityProof`; `rust/core/src/data_ctl/transport_identity.rs` 模块说明 |
| `0x10` | `Text` | `implemented` | M1-02 / `data_ctl::msg` | 当前直接文本通道；wire byte 不复用。 | `rust/core/src/data_ctl/msg.rs::MsgCtl` |
| `0x11` | `GroupText` | `implemented` | M1-02 / `data_ctl::msg` | 当前群文本通道；wire byte 不复用。 | `rust/core/src/data_ctl/msg.rs::GroupMsgCtl` |
| `0x12` | `GroupCreate` | `implemented` | M1-02 / `data_ctl` | 当前接收时自动注册群组；不等同安全群组协议，wire byte 不复用。 | `rust/core/src/data_ctl/mod.rs` 的 `GroupCreate` subscriber |
| `0x13` | `Identity` / `IdentityFrame` | `deprecated` | M1-02 / `data_ctl` | v1 自报身份兼容路径仍有 serializer/subscriber；不得用于 v2 授权，永不复用。 | `rust/core/src/data_ctl/mod.rs` 的 `Identity` subscriber；`Knowledge/designs/identity-and-addressing-v2.md::当前迁移约束` |
| `0x14` | `GroupEvent` | `reserved` | M1-02 / `net::route` | enum/schema 已命名但无 application consumer；不复用。 | `rust/core/src/net/route.rs::MsgType::GroupEvent`; `rust/core/schema/message.capnp::GroupEvent` |
| `0x15` | `GroupSecurityNotice` | `reserved` | M1-02 / `net::route` | enum/schema 已命名但无 application consumer；不复用。 | `rust/core/src/net/route.rs::MsgType::GroupSecurityNotice`; `rust/core/schema/message.capnp::GroupSecurityNotice` |
| `0x16` | `RoomEvent` | `reserved` | M1-02 / `net::route` | enum/schema 已命名但无 application consumer；不复用。 | `rust/core/src/net/route.rs::MsgType::RoomEvent`; `rust/core/schema/message.capnp::RoomEvent` |
| `0x17` | `GroupSyncRequest` | `reserved` | M1-02 / `net::route` | enum/schema 已命名但无 application consumer；不复用。 | `rust/core/src/net/route.rs::MsgType::GroupSyncRequest`; `rust/core/schema/message.capnp::GroupSyncRequest` |
| `0x18` | `GroupSyncChunk` | `reserved` | M1-02 / `net::route` | enum/schema 已命名但无 application consumer；不复用。 | `rust/core/src/net/route.rs::MsgType::GroupSyncChunk`; `rust/core/schema/message.capnp::GroupSyncChunk` |
| `0x19` | `RoomSync` | `reserved` | M1-02 / `net::route` | enum/schema 已命名但无 application consumer；不复用。 | `rust/core/src/net/route.rs::MsgType::RoomSync`; `rust/core/schema/message.capnp::RoomSync` |
| `0x1A` | `SecureGroupPayload` | `reserved` | M1-02 / `net::route` | enum/schema 已命名但无 application consumer；不复用。 | `rust/core/src/net/route.rs::MsgType::SecureGroupPayload`; `rust/core/schema/message.capnp::SecureGroupPayload` |
| `0x20` | `FileMeta` | `implemented` | M1-02 / `data_ctl::file` | 文件传输元数据；wire byte 不复用。 | `rust/core/src/data_ctl/file.rs::FileCtl` |
| `0x21` | `FileChunk` | `implemented` | M1-02 / `data_ctl::file` | 文件传输分块；wire byte 不复用。 | `rust/core/src/data_ctl/file.rs::FileCtl` |
| `0x22` | `FileComplete` | `implemented` | M1-02 / `data_ctl::file` | 文件传输完成标记；wire byte 不复用。 | `rust/core/src/data_ctl/file.rs::FileCtl` |
| `0x23` | `FileTransferV2` | `implemented` | M7-02 / `data_ctl::file_transfer_v2` | `MCFTV201` profile 1；UUIDv7 TransferId-bound meta/chunk/complete/cancel/complete-ack/cancel-ack。旧 0x20..0x22 不重解释；该 byte 永不复用。 | `rust/core/src/data_ctl/file_transfer_v2.rs::FileTransferFrameV2`; `data_ctl/file.rs::FileCtl` |
| `0x30` | `VoiceFrame` | `implemented` | M1-02 / `data_ctl::voice` | 当前语音 frame 通道；wire byte 不复用。 | `rust/core/src/data_ctl/voice.rs::VoiceCtl` |
| `0x40` | `VideoFrame` | `implemented` | M1-02 / `data_ctl::video` | 当前视频 frame 通道；wire byte 不复用。 | `rust/core/src/data_ctl/video.rs::VideoCtl` |
| `0x50` | `RelayAlloc` | `reserved` | M1-02 / `net::route` | 未来外部 relay 的既有保留值；core 无 handler，不能分配、转发或授权。 | `rust/core/src/net/route.rs::MsgType::RelayAlloc` |
| `0x51` | `RelayData` | `reserved` | M1-02 / `net::route` | 同 `0x50`；不复用。 | `rust/core/src/net/route.rs::MsgType::RelayData` |
| `0x60` | `Broadcast` | `reserved` | M1-02 / `net::route` | 同名 `MsgType` 无 application consumer；`center_broadcast` 走 raw gateway data，不能据此重解释或复用本 byte。 | `rust/core/src/net/route.rs::MsgType::Broadcast`; `rust/core/src/net/center_broadcast.rs::BroadcastCenter` |
| `0x61` | `BroadcastJoin` | `reserved` | M1-02 / `net::route` | 已命名但无 application consumer；不复用。 | `rust/core/src/net/route.rs::MsgType::BroadcastJoin` |
| `0x62` | `BroadcastLeave` | `reserved` | M1-02 / `net::route` | 已命名但无 application consumer；不复用。 | `rust/core/src/net/route.rs::MsgType::BroadcastLeave` |
| `0x70` | `DhtQuery` | `reserved` | M1-02 / `net::route` | 同名 `MsgType` 无 application consumer；`DhtNode` 走 raw gateway data，不能复用。 | `rust/core/src/net/route.rs::MsgType::DhtQuery`; `rust/core/src/net/dht.rs::DhtNode` |
| `0x71` | `DhtResponse` | `reserved` | M1-02 / `net::route` | 已命名但无 application consumer；不复用。 | `rust/core/src/net/route.rs::MsgType::DhtResponse` |
| `0x80` | `CallOffer` | `reserved` | M1-02 / `net::route` | 已命名但无 application consumer；不复用。 | `rust/core/src/net/route.rs::MsgType::CallOffer` |
| `0x81` | `CallAccept` | `reserved` | M1-02 / `net::route` | 已命名但无 application consumer；不复用。 | `rust/core/src/net/route.rs::MsgType::CallAccept` |
| `0x82` | `CallReject` | `reserved` | M1-02 / `net::route` | 已命名但无 application consumer；不复用。 | `rust/core/src/net/route.rs::MsgType::CallReject` |
| `0x90` | `RelayRegister` | `reserved` | M1-02 / `net::route` | 既有外部 relay/pipe 保留值；core 无 handler，不复用。 | `rust/core/src/net/route.rs::MsgType::RelayRegister` |
| `0x91` | `RelayResolve` | `reserved` | M1-02 / `net::route` | 既有外部 relay/pipe 保留值；core 无 handler，不复用。 | `rust/core/src/net/route.rs::MsgType::RelayResolve` |
| `0x92` | `PipeFrame` | `reserved` | M1-02 / `net::route` | 既有外部 relay/pipe 保留值；core 无 handler，不复用。 | `rust/core/src/net/route.rs::MsgType::PipeFrame` |
| `0x93` | `PathProbe` | `reserved` | M1-02 / `net::route` | 既有外部 relay/pipe 保留值；core 无 handler，不复用。 | `rust/core/src/net/route.rs::MsgType::PathProbe` |
| `0xA0` | `MeshAnnounce` | `reserved` | M1-02 / `net::route` | 已命名但无 application consumer；不复用。 | `rust/core/src/net/route.rs::MsgType::MeshAnnounce` |
| `0xA1` | `CapabilityQuery` | `reserved` | M1-02 / `net::route` | application tag，与 assertion capability bit 是不同 namespace；无 consumer，不复用。 | `rust/core/src/net/route.rs::MsgType::CapabilityQuery` |
| `0xA2` | `RelayOffer` | `reserved` | M1-02 / `net::route` | 既有外部 relay 保留值；core 无 handler，不复用。 | `rust/core/src/net/route.rs::MsgType::RelayOffer` |
| `0xA3` | `ReputationUpdate` | `reserved` | M1-02 / `net::route` | 已命名但无 application consumer；不复用。 | `rust/core/src/net/route.rs::MsgType::ReputationUpdate` |
| `0xB0` | `Envelope` | `implemented` | M4-01 / `net::route`、`data_ctl::convert::serializer` | v2 不可变消息信封帧，仅携带 route header version `0x02`；不得以 `0x01` 或 v1 MsgType 语义解释。`0xB1-0xBF` 未分配（当前保持 `Custom(u8)` fallback，不批量宣称 capability）。 | `rust/core/src/net/route.rs::MsgType::Envelope`；`rust/core/schema/message.capnp::MessageEnvelope` |
| `0xE0` | `Custom(0xE0)` / `EXT_MSG_TYPE` | `implemented` | M1-02 / `data_ctl::custom` | 唯一现有应用扩展载体，使用 `ExtMessage.ext_type` 子命名空间；byte 与子类型均不得无登记碰撞。 | `rust/core/src/data_ctl/custom.rs::EXT_MSG_TYPE`、`CustomCtl` |
| `0xF0` | `Custom(0xF0)` / `RAW_MSG_TYPE` | `implemented` | M1-02 / `data_ctl::bytes` | 唯一现有 raw-bytes passthrough；不得赋予另一语义。 | `rust/core/src/data_ctl/bytes.rs::RAW_MSG_TYPE`、`BytesCtl` |
| `0xE1-0xEF`、`0xF1-0xFF` | existing custom reserved ranges | `reserved` | M1-02 / `net::route` | 仅此两个源码明示的 future custom ranges；每个 byte 在实现前必须单独登记，不能批量宣称 capability。 | `rust/core/src/net/route.rs::MsgType::Custom` 注释 |
| 其他未命名 byte | `Custom(u8)` fallback | `implemented`（fallback 行为） | M1-02 / `net::route` | `from_u8` 将未知 byte 表示为 `Custom`，但这不分配语义、不绕过保留范围，也不表示存在 handler。 | `rust/core/src/net/route.rs::MsgType::from_u8` |

### Cap'n Proto schema 与 union

| Entry | 状态 | Owner | 兼容与复用策略 | 源码证据 |
|---|---|---|---|---|
| schema ID `@0x9c4e1a7b3d5f6082` | `implemented` | M1-02 / `core/schema/message.capnp` | Cap'n Proto 文件 identity，不得替换或复用为另一 schema；schema 本身没有独立的 global schema-version 常量。 | `rust/core/schema/message.capnp` 文件头 |
| `Header`、`Envelope`，字段 `Header.version` 当前 `1` | `implemented` | M1-02 / `data_ctl::convert::serializer` | 字段 ordinal 和含义不可重排；新字段只追加。`Header.version` 与 route header version 都是 `1`，不能把它当 ALPN major。 | `rust/core/schema/message.capnp::Header/Envelope`; `rust/core/src/data_ctl/convert/serializer.rs::Header::CURRENT_VERSION` |
| `TextMessage`、`PingMessage`、`PongMessage`、`FileMeta`、`FileChunk`、`VoiceFrame`、`VideoFrame` | `implemented` | M1-02 / `data_ctl::convert::serializer` | 当前 serializer 有双向 packed 编解码；字段 ordinal 不复用。 | `rust/core/src/data_ctl/convert/serializer.rs::Serializer::{serialize,deserialize}_*` |
| `GroupTextMessage`、`GroupCreateMessage` | `implemented` | M1-02 / `data_ctl::convert::serializer` | 当前 serializer 与 DataCtl 的群文本/建群路径使用；字段 ordinal 不复用。 | `rust/core/src/data_ctl/convert/serializer.rs::serialize_group_text`、`serialize_group_create`; `rust/core/src/data_ctl/mod.rs` |
| `IdentityFrame` | `deprecated` | M1-02 / `data_ctl::convert::serializer` | 仍为 v1 兼容 serializer/subscriber；不构成 v2 principal/authorization，不得复用字段或名称。 | `rust/core/src/data_ctl/convert/serializer.rs::IdentityFrame`; `Knowledge/designs/identity-and-addressing-v2.md::当前迁移约束` |
| `IdentityHello`、`IdentityProof` | `implemented`（可选 quarantine） | M1-02 / `data_ctl::transport_identity` | 由 `transport_auth` 的 BIND 前 quarantine 使用；字段 ordinal 不复用，不能描述为未实现的 DeviceHello cutover。 | `rust/core/schema/message.capnp::IdentityHello/IdentityProof`; `rust/core/src/net/p2p.rs::authenticate_*` |
| `ExtMessage` | `implemented` | M1-02 / `data_ctl::custom` | 只在 `0xE0` 载体上使用；`ext_type` 是应用子命名空间，不分配新的 `MsgType`。 | `rust/core/schema/message.capnp::ExtMessage`; `rust/core/src/data_ctl/custom.rs::CustomCtl` |
| `SecureGroupPayload`、`GroupEvent`、`GroupSecurityNotice`、`GroupSyncRequest`、`GroupSyncChunk`、`GroupMemberStateWire`、`GroupStateSnapshot` | `reserved` | M1-02 / `core/schema/message.capnp` | 已声明字段但 serializer 未导出这些 payload 的编解码，application `MsgType` 无 consumer；不得按名称宣称 wire 已实现或复用 field ordinal。 | `rust/core/schema/message.capnp` 相应 structs；`rust/core/src/data_ctl/convert/serializer.rs` 的公开方法集合 |
| `RoomEvent`、`RoomSync` | `reserved` | M1-02 / `core/schema/message.capnp` | 已声明字段但未接入 application `MsgType` consumer；不得复用 field ordinal。 | `rust/core/schema/message.capnp::RoomEvent/RoomSync`; `rust/core/src/data_ctl/room.rs` 模块说明 |
| `MessageEnvelope`（M4-01 新 struct） | `implemented` | M4-01 / `core/schema/message.capnp` | 字段 ordinal 冻结：`protocolVersion@0`、`realmId@1`、`conversationKind@2`、`conversationId@3`、`originUserId@4`、`originDeviceId@5`、`messageId@6`、`contentKind@7`、`createdAtMs@8`、`recipients@9`、`payload@10`、`signature@11`；只追加不重排。conversation kind `1=Direct/2=Group` 与 ID 长度（32/16）必须匹配，protocolVersion 非 `1`、signature 非 64 字节、realm/ID 长度错误均 fail-closed。该 struct 与 v1 `Envelope(Header+payload)` 是不同命名空间。 | `rust/core/schema/message.capnp::MessageEnvelope` |
| `EnvelopeRecipient`（M4-01 新 struct） | `implemented` | M4-01 / `core/schema/message.capnp` | ordinal 冻结：`userId@0`(16 bytes)、`deviceId@1`(16 bytes)；recipient 列表必须为 canonical 排序的 (UserId, DeviceId)，重复项拒绝。 | `rust/core/schema/message.capnp::EnvelopeRecipient` |

当前 `message.capnp` **没有** `union` 声明，因此不存在已登记或可分配的 union discriminant；新增 union 前必须先建立独立登记项，不能把“没有 union”当作预留编号。

## 5. SQLite Local Schema 与迁移号

本区只登记 `rust/core/src/data_ctl/database.rs::DatabaseCtl` 的本地 SQLite `PRAGMA user_version` 序列，不把账号服务器的独立 SQLite store 混入同一版本域。

| Version / step | 状态 | Owner | 兼容与复用策略 | 源码证据 |
|---|---|---|---|---|
| `0 -> 1`，v1 基线：`messages`、`files`、`groups`、`group_members`、`contacts`、`local_identity`、`serve_cache`、`seeders`、`group_files` 及 `idx_messages_conv` | `implemented` | M1-02 / `data_ctl::database`（M0-04） | v1 基线已冻结；只允许后续迁移，不能重写其历史 version 或把 `0` 当已存在 schema。 | `rust/core/src/data_ctl/database.rs::migrate_to_v1` |
| `1 -> 2`，`idx_group_files_group_timestamp` | `implemented` | M1-02 / `data_ctl::database`（M0-04） | 只向前增加索引；version `2` 不复用。 | `rust/core/src/data_ctl/database.rs::migrate_to_v2` |
| `2 -> 3`，`node_keypair` | `implemented` | M1-02 / `data_ctl::database`（M0-04） | 只向前增加加密本地 key 表；version `3` 不复用。 | `rust/core/src/data_ctl/database.rs::migrate_to_v3` |
| `3 -> 4`，`secure_group_state`、`secure_group_members`、`secure_group_event_log`、`room_control_state`、`room_event_log` 与两个 audit-log indexes | `implemented` | M1-02 / `data_ctl::database`（M0-04） | 历史 v4 DDL 固定；version `4` 不复用，v5 前的自由文本 ID 是 v4 历史事实。 | `rust/core/src/data_ctl/database.rs::migrate_to_v4` |
| `4 -> 5`，current `SCHEMA_VERSION = 5`：对 v4 的 typed-ID 文本、room JSON 内嵌 ID 和唯一声明 FK 做只读验证 | `implemented` | M1-02 / `data_ctl::database`（M0-04） | 同一事务内校验并推进 `user_version`；失败为 `DbError::Invalid`、回滚并保持 v4，不删除/改写/生成 ID。version `5` 不复用。 | `rust/core/src/data_ctl/database.rs::SCHEMA_VERSION`、`init`、`migrate_to_v5` |
| `5 -> 6`，`v2_messages`（typed v2 信封表，PK `(realm_id, origin_device_id, message_id)`）与 `legacy_v5_quarantine`（v1 旧行归档表） | `implemented` | M4-01 / `data_ctl::database` | **additive**：只新增表，绝不改写 v1 `messages`/`files` 行、绝不自动把 v5 的 `m<seq>`/字符串行改写为 UUID 或生成替代 ID。`quarantine_legacy_v1` 把无法 canonical 类型化的 v1 行复制进 `legacy_v5_quarantine`（原表保持原值）。`insert_v2_message` 以 `(realm, origin device, message id)` 为唯一键：相同键且字节完全相同 = 幂等 duplicate（无第二次副作用）；相同键不同内容 = conflict（拒绝且不改写，审计首条 accepted）。version `6` 不复用。 | `rust/core/src/data_ctl/database.rs::SCHEMA_VERSION`、`migrate_to_v6`、`insert_v2_message`、`quarantine_legacy_v1` |
| `6 -> 7`，`v2_outbox` per-recipient 状态与 `v2_inbox_dedup` durable digest | `implemented` | M4-03 / `data_ctl::database` | additive；v6 行原值保留。outbox 上限 4096，inbox cache 4096 但 durable `v2_messages` 唯一键仍防 eviction 后 replay；DDL/version 同 transaction，单 `DatabaseCtl` mutex writer。version `7` 不复用。 | `rust/core/src/data_ctl/database.rs::SCHEMA_VERSION`、`migrate_to_v7` |
| `7 -> 8`，`v2_content_accounts`、`v2_content_sessions`、`v2_content_bootstraps`、`v2_receipt_outbox` | `implemented` | M4-05 / `data_ctl::database` | additive schema；secret state 仅为 vodozemac encrypted pickle。ratchet/account advance 与 exact outbox/inbox/receipt/bootstrap 在单 transaction 中乐观 revision 提交；失败回滚，不复用前进 state。真实 v7 fixture、故障 rollback/retry 与 fresh schema 测试。 | `SCHEMA_VERSION=8`、`migrate_to_v8`、`commit_*_direct` |
| `8 -> 9`，receipt retry state/attempt/next/deadline/expiry | `implemented` | M5-03 / `data_ctl::database` | additive + deterministic backfill；旧 receipt wire 原字节保留并从 Queued/attempt0 开始。600s、5 attempts、30d 边界不允许第六次或过期复活。 | `migrate_to_v9`、receipt due/claim/restart tests |
| `9 -> 10`，`v2_remote_content_lifecycle_tombstones` | `implemented` | M5-01 / `data_ctl::database` | revoke tombstone 永久阻止同 remote device rebootstrap；rotation 保存 minimum new key version，仅经验证的更高版本 prekey 可清除。重开不得因 session 已删除而恢复旧授权。 | `SCHEMA_VERSION=10`、`migrate_to_v10`、`admit_verified_remote_prekey_bundle` |
| `10 -> 11`，directory/credential lifecycle restart pins | `implemented` | M5-01 / `data_ctl::database` | additive 三表一索引；已验证 directory revision/commitment 与 exact-device credential status sequence 跨重启单调。signed-body expiry 不删除 pin；迁移失败与 version 写回同事务回滚。 | `SCHEMA_VERSION=11`、`migrate_to_v11`、`pin_verified_directory_v1`、`pin_verified_credential_status_v1` |

`init` 拒绝负版本和高于 `11` 的版本。每个 step 与 `user_version` 写回在同一 transaction，故没有已登记的 downgrade 或 alternate migration branch；未来数据根/cutover 不能借用本地 schema 语义。

### 账号服务器 v2 store（独立版本域，M3-03）

本域**只**登记 `rust/server/src/store_v2.rs::StoreV2` 的独立 SQLite `PRAGMA user_version` 序列。它与上方 core `DatabaseCtl` 的 v5、v1 `store.rs`（无 `user_version`）是三个互不共享的版本域，永不相互借用或导入行。

| Version / step | 状态 | Owner | 兼容与复用策略 | 源码证据 |
|---|---|---|---|---|
| `0 -> 1`，v2 基线：`v2_users`、`v2_tokens`、`v2_devices`、`v2_provision_challenges` 及其索引；`foreign_keys=ON` | `implemented` | M3-03 / `server::store_v2` | 整个 DDL + 索引 + `user_version` 回写在单一 transaction；负版本或高于 1 的版本 fail-closed；已迁移库重开为 no-op（`IF NOT EXISTS`）。version `1` 不复用，且不与 core v5 或 v1 store 混用。 | `rust/server/src/store_v2.rs::SCHEMA_VERSION_V2`、`init` |
| `1 -> 2`，device realm binding + `v2_content_prekey_bundles`、`v2_content_one_time_prekeys`、`v2_content_prekey_rate` | `implemented` | M4-05 / `server::store_v2` | backfill device realm from consumed provision challenge；publish 验 active owner/realm/device signing key；claim 验 requester active/same realm，OTK 原子删除，fallback only，revoke cascade 清理。独立版本域 current=2。 | `SCHEMA_VERSION_V2=2`、`publish_content_prekeys`、`claim_content_prekey` |
| `2 -> 3`，credential/status/directory/presence/audit | `implemented` | M5-01 / `server::store_v2` | 旧 active row 不具 realm signer，迁移保持 `credentialized=0/directory_visible=0`，不伪造 assertion/status。directory revision、presence lease 和 audit 同属独立 server v2 域。 | `SCHEMA_VERSION_V2`、`v2_device_directory_revisions`、`v2_presence_state` |
| `3 -> 4`，credential history 与 rotation receipts | `implemented` | M5-01 / `server::store_v2` | continuous rotation 保留 DeviceId，credential/key/assertion sequence 严格递增；old exact receipt 可重放，old ordinary authority 失效。 | `v2_device_credentials`、`v2_credential_rotations`、rotation CAS tests |
| `4 -> 5`，per-actor lifecycle rate + per-user audit index | `implemented` | M5-01 / `server::store_v2` | presence mutation 6/min，directory/presence read 各60/min；audit 每 user 4096 且严格30天。N+1 与 audit-write failure 同事务零副作用。 | `SCHEMA_VERSION_V2=5`、`v2_device_lifecycle_rate`、resource/concurrency tests |

### 客户端 local-v2 身份 store（独立版本域，M3-04）

本域**只**登记 `rust/client/src/local_v2.rs::LocalV2Store` 的独立 SQLite `PRAGMA user_version` 序列。它与上方 core `DatabaseCtl` 的 v5、账号服务器 v1 `store.rs`（无 `user_version`）和账号服务器 v2 `store_v2.rs`（版本域 `0 -> 1`）是四个互不共享的版本域，永不相互借用或导入行。该 store 只保存管理员带外导入并已 `validate(now)` 过的 v2 身份，**不**接线到活动 QUIC 客户端，不导入 v1 profile/数据，不在失败时生成替代身份。

| Version / step | 状态 | Owner | 兼容与复用策略 | 源码证据 |
|---|---|---|---|---|
| `0 -> 1`，local-v2 基线：`local_v2_identity` 单行表（`id` 固定为 1），持久 pinned root/manifest/profile、typed realm/user/device ID、设备公钥、accepted manifest 序列与 canonical wire，以及 Argon2id+ChaCha20Poly1305 封装的设备密文与 KDF salt；无明文密钥列；`foreign_keys=ON` | `implemented` | M3-04 / `client::local_v2` | DDL + `user_version` 回写在单一 transaction；负版本或高于 1 的版本 fail-closed；已迁移库重开为 no-op（`IF NOT EXISTS`）。version `1` 不复用，且不与 core v5、v1 store 或服务器 v2 store 混用，也不导入其行。 | `rust/client/src/local_v2.rs::SCHEMA_VERSION_V2`、`init` |

### M5 local data lifecycle / PortableBackupV1 contract

| 值 | 状态 | Owner | 兼容与复用策略 | 源码证据 |
|---|---|---|---|---|
| `LifecycleAction::{Logout,RemoveDevice,RemoteRevoke,FactoryReset}` 与 truthful outcome/disposition matrix | `implemented`（contract + core storage actions） | M5-02 / client+core | 四动作不可互换。core logout 原子终止 pending authority但保留 encrypted history；remove-device exact、remote revoke 复用 exact hook、factory reset 明确只保证本地全清。remote RPC、secure-storage/filesystem 产品编排仍 pending。 | `lifecycle_policy`、`StorageLifecycleAction`、`apply_storage_lifecycle` |
| `PortableBackupV1` data-class allowlist；portable identity recovery = unsupported | `implemented`（encrypted container + quarantine staging；product import pending） | M5-02 / client | Argon2id + ChaCha20Poly1305 versioned container，header AAD，canonical manifest/entries，原子 filesystem quarantine；拒绝 authority-bearing classes、wrong key、tamper、future version、path/symlink/reparse。SQLite/WAL snapshot、fresh status/lease 和 product commit 未实现。 | `encrypt_portable_backup_v1`、`decrypt_portable_backup_v1`、staging tests |
| backup total `256 MiB`、manifest `1 MiB`、entries `16384`、nesting depth `8`、single managed media `64 MiB` | `implemented` | M5-02 / client | 独立 local archive namespace；container/KDF/length N+1 在 KDF、分配和 I/O 前拒绝。 | `MAX_BACKUP_*` constants + boundary tests |
| platform capability `WindowsOsBoundV1`；Web/Other `Unsupported` | `implemented`（core + local-v2 product DB open） | M5-02 / client+Flutter | Windows 每账户独立 random core/local-v2 keys 与 hashed account path；signed-in `createNodeWithAccountStorage` 分别打开 `core.sqlite3`/`local-v2.sqlite3`，双 key copy 均覆零，错钥匙/同路径 fail-closed。Web/Other fail-closed。 | `ensureAccountDataKeys`、`FrbNativeService.startNode`、FRB account-storage tests |

### M6 MLS SQLCipher atomic storage

| 值 | 状态 | Owner | 兼容与复用策略 | 源码证据 |
|---|---|---|---|---|
| private schema `v2_mls_*` / schema version `1` | `implemented`（W29 production adapter） | M6 W29 / core | 群头、治理、immutable wire、frozen leaf outbox、pending batch 与 1024 committed tombstone 在同一 SQLCipher DB；不是 wire/API version，不得复用于 legacy secure-group/Room。 | `rust/core/src/group_mls_sqlcipher_v1.rs`、restart/atomicity tests |
| `DirectoryLeafAuthorityV1` / `FrozenGroupLeafSetV1` | `implemented` | M6 W30 / core | 当前 signed directory 精确绑定 realm/user/device/credential/KeyId；8 users、8 devices/user、64 leaves；冻结 non-origin exact-leaf recipients 与每用户 revision/commitment pin。无 SessionId/address/legacy/relay fallback。 | `rust/core/src/group_mls_directory_v1.rs`、authority/fanout/lifecycle tests |
| `SqlCipherAtomicGroupStoreV1::open_encrypted(path,key[32])` | `implemented` | M6 W29 / core | 公开入口强制 SQLCipher engine、exact raw key、schema read 与 memory security；无明文/default/path-derived fallback。已 keyed connection seam 仅 crate-private。 | encrypted header、wrong-key、reopen tests |
| `PortableGroupAuditV1` projection | `implemented` | M6 W29 / group_mls+core | 仅 realm/group、epoch、commit digest、governance EventId；禁止 MLS private state、wire、recipient leaf、pending 和 delivery authority。 | `portable_audit`、privacy review |
| `GroupMlsTransportPayloadV1` / private wire version `1` | `implemented` | M6 W30 / core | 只作为 authenticated `/3` Envelope 内部的有界 kind/action/subject/OpenMLS bytes；unknown/truncate/trailing/tamper 在业务副作用前拒绝。不是新路由、discovery 或 relay 协议。 | `group_mls_transport_v1.rs` rejection matrix |
| `MlsGroupGovernanceApi` | `implemented` | M6 W31 / Flutter FRB | 独立于 legacy Groups/SecureGroup API；暴露 canonical exact-device snapshot、create/recover/transfer/remove/retry 与 dispatch 前取消。生成物只由 FRB 2.12.0 更新。 | `flutter/rust/src/api/mls_group.rs`、codegen drift gate |

## 6. FRB Public Boundary

| Boundary / value | 状态 | Owner | 兼容与复用策略 | 源码证据 |
|---|---|---|---|---|
| FRB scan root `crate::api`，Rust root `rust/`，Dart generated output `lib/src/rust` | `implemented` | M1-02 / `flutter/rust/src/api` | 当前没有 `FRB_API_VERSION`/FRB major 常量可登记；任何公开签名、DTO 或 error-shape 改动按 breaking boundary 处理，生成物只能由 codegen 更新。 | `flutter/flutter_rust_bridge.yaml` |
| `RustNode` 与当前子句柄 `ContactApi`、`MessagingApi`、`GroupsApi`、`FilesApi`、`UiApi`、`SecureGroupApi`、`RoomApi`、`RustServerClient` | `implemented` | M1-02 / `flutter/rust/src/api` | 以公开 Rust API 和生成 Dart binding 为边界；无隐式 API version 或自动兼容转换，修改前先登记。 | `flutter/rust/src/api/mod.rs` exports；相应 `api/*.rs` |
| `create_node_with_storage(bind_address,database_path,database_key[32])` / Dart `createNodeWithStorage` | `implemented`（additive core-only compatibility） | M5-02 W25 / FRB | exact 32-byte key 与 SQLCipher core open；不打开 local-v2，因此不再用于 signed-in 产品。旧 `create_node` 仅为 ephemeral compatibility；generated/native 必须 paired。 | `flutter/rust/src/api/mod.rs`、storage creation tests |
| `create_node_with_account_storage(bind,core_path,core_key[32],local_v2_path,local_v2_key[32])` / Dart `createNodeWithAccountStorage` | `implemented`（additive signed-in product entry） | M5-02 W25 / FRB | 两路径非空且不同，两 key exact-32 且独立；local-v2 只经 `LocalV2Store::open_encrypted` 打开并由 `RustNode` 持有。任一失败无运行节点；Dart/Rust 临时 key 均覆零。旧 core-only API 保留兼容但不再用于 signed-in 产品。 | `flutter/rust/src/api/mod.rs`、`FrbNativeService.startNode`、official FRB 2.12.0 paired codegen、21 Rust/46 Flutter tests |
| legacy session/string API：`RustNode::{connect,peers,disconnect,set_local_user_id,send_identity}`、`MessagingApi::send_text_raw`、`FilesApi::send_file_raw` | `deprecated` | M1-02 / `flutter/rust/src/api` | 入口仍存在供过渡使用，但新 API 不得以 session string 或 v1 identity 语义扩张；移除时不能复用名称来承载不兼容语义。 | `flutter/rust/src/api/mod.rs` 的 legacy sections；`messaging.rs`、`files.rs` 的 raw-method 注释 |
| FRB failure return `Result<_, String>` / `Option<String>` | `implemented`（未分类） | M1-02 / `flutter/rust/src/api` | 当前是诊断字符串，不是稳定错误类别或编号；Dart 调用方不得匹配文本。新增稳定 FRB errors 需先登记 typed code/DTO，不能复用现有文案。 | `flutter/rust/src/api/mod.rs::create_node/connect`; `contacts.rs`、`messaging.rs`、`files.rs`、`groups.rs`、`room.rs`、`server_client.rs` |
| `ConversationIdDto`、`ConversationIdKind`（`Direct`/`Group`）、`parse_conversation_id(text)` | `implemented` | M4-01 / `flutter/rust/src/api/envelope` | typed DTO 边界：非 canonical 文本、跨 kind 输入（UUIDv7 文本给 Direct、64-hex 给 Group、错误前缀）在进入 core/storage 前拒绝；解析成功后 DTO 往返保持 canonical 值。 | `flutter/rust/src/api/envelope.rs::parse_conversation_id` |
| operation contract version `1`、`requestCancelOperationV1`、typed cancel/error DTO | `implemented`（control boundary；domain wiring pending） | M7-02 W34 / FRB | Flutter 只能请求 cancel/读取固定版本，不能通用 begin 或发布 terminal；generated Dart/native 必须与 Rust 同工件。现有 message/file 领域尚未完成 cleanup 接线。 | `flutter/rust/src/api/operations.rs`、official FRB 2.12.0 codegen、4 tests |

## 7. 稳定错误类别

本表的稳定性只适用于源码 enum variant 身份及明确的映射层，不承诺 `Display` 文本稳定，也没有为这些类别分配数字码。

| Layer / class | 状态 | Owner | 兼容与复用策略 | 源码证据 |
|---|---|---|---|---|
| `P2pError::UnsupportedProtocol` | `implemented` | M1-02 / `net::p2p`（M0-03） | ALPN/major 或未知 control tag 的副作用前拒绝类别；不得并入畸形帧或复用其含义。 | `rust/core/src/net/p2p.rs::P2pError`、`require_v2_alpn`、`recv_ctrl_msg` |
| `P2pError::MalformedFrame` | `implemented` | M1-02 / `net::p2p`（M0-03） | 已知 control element 的长度、尾随、role 或固定字段错误；不得并入 protocol unsupported 或复用。 | `rust/core/src/net/p2p.rs::P2pError`、`ControlMsg::decode` |
| 其余 `P2pError`：`ConnectionFailed`、`SessionMismatch`、`StreamOpenFailed`、`DatagramFailed`、`ConfigError`、`TlsError`、`NotConnected`、`ChannelClosed`、`SelfConnection` | `implemented` | M1-02 / `net::p2p` | Rust transport/lifecycle 类别；变体含义不得重解释为 protocol frame error，未分配数字码。 | `rust/core/src/net/p2p.rs::P2pError` |
| `GatewayError::UnsupportedProtocol` / `GatewayError::MalformedFrame` | `implemented` | M1-02 / `net::p2p_gateway`（M0-03） | 必须保持对上面两个 P2P 类别的一对一映射；不得降级成一般连接失败。 | `rust/core/src/net/p2p_gateway.rs::GatewayError`、`From<P2pError>` |
| 其余 `GatewayError`：`ConnectionFailed`、`NotConnected`、`SendFailed`、`RecvFailed`、`Timeout`、`Closed`、`InvalidConfig` | `implemented` | M1-02 / `net::p2p_gateway` | gateway domain 类别；无数字码，variant 语义不得复用。 | `rust/core/src/net/p2p_gateway.rs::GatewayError` |
| `SerializeError::{Encode, Decode, Truncated}` | `implemented` | M1-02 / `data_ctl::convert::serializer` | Cap'n Proto 编解码类别；不作为 protocol-version fallback，variant 语义不复用。 | `rust/core/src/data_ctl/convert/serializer.rs::SerializeError` |
| `TransportIdentityError`：malformed/role/assertion/certificate/realm/UID/capability/version/nonce/hash/signature/codec/state 拒绝变体 | `implemented` | M1-02 / `data_ctl::transport_identity` | 认证器内部 fail-closed 类别；目前 P2P host 将其显示为 `ConnectionFailed`，不得误称其已是稳定 FRB error code。 | `rust/core/src/data_ctl/transport_identity.rs::TransportIdentityError`; `rust/core/src/net/p2p.rs::auth_error` |
| `DbError::{Sqlite, Crypto, Invalid}` | `implemented` | M1-02 / `data_ctl::database`（M0-04） | local storage 类别；v5 invalid-ID/FK 失败必须使用 `Invalid` 并回滚，不得改写为成功或自动修复。 | `rust/core/src/data_ctl/database.rs::DbError`、`migrate_to_v5` |
| `DataCtlError::{Transport, Codec, Router, InvalidArg}` 与 `RouterError::{GroupNotFound, PeerNotFound, DispatchClosed}` | `implemented` | M1-02 / `data_ctl` | core 聚合/路由类别；FRB 目前字符串化它们，不能声称 Dart 已获得 typed stable code。 | `rust/core/src/data_ctl/mod.rs::DataCtlError`; `rust/core/src/data_ctl/convert/router.rs::RouterError` |
| direct stable strings：`direct_invalid`、`direct_malformed`、`direct_version_unsupported`、`direct_profile_unsupported`、`direct_too_large`、`direct_signature_invalid`、`direct_metadata_mismatch`、`direct_crypto_failed`、`direct_session_mismatch`、`direct_bootstrap_mismatch`、`direct_resource_limit`、`direct_storage_failed` | `implemented` | M4-05 / identity + core | `code()` 是稳定类别；`Display` 不是 API。日志只记录类别/非敏感诊断，不记录 plaintext、key、pickle 或完整 ciphertext。 | `DirectContentError::code`、`DirectE2eeError::code` |
| attachment stable strings：`attachment_invalid`、`attachment_malformed`、`attachment_version_unsupported`、`attachment_profile_unsupported`、`attachment_too_large`、`attachment_metadata_mismatch`、`attachment_crypto_failed`、`attachment_resource_limit`、`attachment_transfer_rejected` | `implemented` | M4-04-I1 / identity + core | `code()` 是稳定类别，`Display` 不是 API。crypto failure 不暴露 key/plaintext/ciphertext oracle；transfer 细分仍由内部 `ChunkError` 审计。 | `AttachmentError::code`、`AttachmentE2eeError::code` |
| lifecycle stable strings：`lifecycle_invalid`、`lifecycle_malformed`、`lifecycle_version`、`lifecycle_signature`、`lifecycle_signer`、`lifecycle_realm`、`lifecycle_expired`、`credential_revoked`、`rotation_proof_invalid`、`recovery_required`、`presence_too_many_candidates`、`fanout_too_large`、`directory_stale`、`directory_conflict`、`presence_epoch_exhausted` | `implemented`（pure contract） | M5-01 / identity | `DeviceLifecycleError::code()` 是 future RPC 的唯一映射源；`Display` 不稳定。server W23 不得把 internal/SQLite/panic 映为 lifecycle invalid。 | `rust/identity/src/device_lifecycle.rs::DeviceLifecycleError::code` |
| `PublicErrorDtoV1`、16 baseline codes、14 categories、cancel control 与 terminal CAS | `implemented`（core + FRB/Rust control adapter foundation；domain wiring pending） | M7-02 W34 / core+FRB+msg_api | DTO/builder/敏感字段拒绝、bounded registry、cleanup-before-terminal adapter、FRB cancel control 与 Rust SDK/RPC controller 已实现；现有 message/file/server 业务错误仍未统一，不能替代当前 `Result<_,String>` 或宣称跨层 conformance。 | `public_error_v1.rs`、`operation_registry_v1.rs`、`public_operation_adapter_v1.rs`、FRB/RPC tests |
| FRB `String` failures | `implemented`（无稳定类别） | M1-02 / `flutter/rust/src/api` | 现有字符串不进入稳定 code namespace，不能被复用为正式 code；正式化前不得为其发放号码。 | `flutter/rust/src/api` 的 `Result<_, String>` 返回面 |

## 8. 账号服务器 v2 RPC op 与错误码（M3-03）

本区登记 `rust/server/src/rpc_v2.rs::dispatch_v2` 的并行 v2 RPC op 名与稳定错误码字符串。它与 v1 `quic.rs` op 名是不同 surface，不互译、不回退；v2 surface 当前未接线到活动 QUIC server。错误为稳定类别字符串，无数字错误码。

### v2 RPC op（字符串判别字段）

| op | 状态 | Owner | 兼容与复用策略 | 源码证据 |
|---|---|---|---|---|
| `v2_register_account`、`v2_login`、`v2_begin_provision`、`v2_complete_provision`、`v2_list_devices`、`v2_revoke_device` | `implemented` | M3-03 / `server::rpc_v2` | 并行 v2 surface；`v2_` 前缀独立于 v1 op；未知 v2 op → `not_found`；`V2State` 为 `None` 时全部返回 `upgrade_required`（副作用前）。请求体用 `deny_unknown_fields`，`complete` 体无私钥字段。 | `rust/server/src/rpc_v2.rs::dispatch_v2` 及各 `*Body` |
| `v2_publish_content_prekeys`、`v2_claim_content_prekey` | `implemented` | M4-05 / `server::rpc_v2` | publish 接 signed canonical bundle wire；claim 接 authenticated requester/target DeviceId 并返回一个 canonical claim wire。没有 private key 字段、v1 fallback 或跨 realm claim。 | `PublishContentPrekeysBody`、`ClaimContentPrekeyBody`、RPC roundtrip test |

### v2 RPC 稳定错误码（字符串）

| code | 状态 | Owner | 兼容与复用策略 | 源码证据 |
|---|---|---|---|---|
| `upgrade_required` | `implemented` | M3-03 / `server::rpc_v2` | v2 不可用（`V2State=None`）时的副作用前唯一响应；不得复用为其他拒绝类别或降级到 v1。 | `rust/server/src/rpc_v2.rs::dispatch_v2` |
| `too_many_devices`、`too_many_pending`、`challenge_expired`、`challenge_binding_mismatch` | `implemented` | M3-03 / `server::rpc_v2`、`store_v2` | 分别来自 8 devices/2 pending/10m TTL/CSR 属性绑定的副作用前拒绝；语义不复用。 | `rust/server/src/store_v2.rs::StoreV2Error`、`rpc_v2.rs::store_err_resp` |
| `prekey_exhausted`、`prekey_expired`、`rate_limited` | `implemented` | M4-05 / server | 分别表示无 OTK/fallback、bundle 到期和 per-device minute window 超限；不得映射为成功、静态 fallback 或 crypto oracle。 | `StoreV2Error`、`store_err_resp` |
| `invalid`、`invalid_signature`、`unauthorized`、`not_found`、`account_taken`、`internal` | `implemented` | M3-03 / `server::rpc_v2`、`store_v2` | 与 v1 同名类别在 v2 surface 内独立稳定；不与 v1 `StoreError` 数值互译，无数字码。 | `rust/server/src/rpc_v2.rs::store_err_resp` |

## 9. Duplicate / Conflict Audit (2026-07-29)

| Domain | Audit | Result |
|---|---|---|
| ALPN / major | `modcpt-p2p/1` 与 `/2` 名称和 major 分离；当前只接受 `/2`。 | pass；v1 标为 `deprecated` 而非可复用 reserve。 |
| P2P control | tag `0x01`、`0x02`、`0x03`、`0x04`、`0x05`、`0xFF` 各有单一 `ControlMsg` 含义；BIND/BIND_ACK 都是 34 bytes。 | pass；`0x04`/`0x05` 保留，不冒充 quarantine flow。 |
| Namespace collision | stream discriminator、control tag、application `MsgType`、assertion capability bit、Cap'n Proto field ordinal 与 SQLite version 各自独立。 | pass；跨 namespace 的相同数值已逐一注明，不是同域冲突。 |
| MsgType | 每个 `MsgType::to_u8` 显式值唯一；`0xE0` 与 `0xF0` 已由 controller 使用；现有 relay/custom ranges未新增值。 | pass；无同域重复，未分配任何未来 byte。 |
| Cap'n Proto | schema 有单一 file ID；当前没有 `union`，field ordinal 只在各 struct 内有效。 | pass；没有可冲突的 union discriminant 或已声明 field reserve。 |
| SQLite | `init` 唯一顺序分派 `0 -> 1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7 -> 8`，current `SCHEMA_VERSION` 为 `8`。 | pass；v6/v7/v8 均 additive，无重复 migration target 或 downgrade/reuse path。 |
| SQLite（v2 账号 store） | 账号服务器 v2 store `StoreV2` 拥有独立 `0 -> 1 -> 2` 域；与 core `DatabaseCtl`、v1 `store.rs`（无 `user_version`）分离，不共享、不导入。 | pass；prekey schema/version 独立登记。 |
| SQLite（客户端 local-v2 store） | 客户端 `LocalV2Store` 拥有独立 `user_version` 域，当前仅 `0 -> 1`；与 core v5、v1 store、服务器 v2 store 四域分离，不共享、不导入。 | pass；第四个独立版本域，未借用任何既有序列。 |
| v2 RPC op / 错误码 | v2 `v2_*` op 名与 v1 op 名分属不同 surface；v2 错误为稳定字符串类别，无数字码，与 v1 `StoreError` 不互译。 | pass；`v2_` 前缀隔离，未复用 v1 op/数值。 |
| Errors / FRB | Rust errors 使用 enum variant；FRB failure surface 多为 `String`。 | pass；未虚构数字错误码；FRB typed error-code 仍需单独契约。 |
| v1 identity conflict | v1 `IdentityFrame` 仍有代码 serializer/subscriber，但 v2 设计禁止它参与 v2 授权。 | resolved in registry：`deprecated`，保留现状但禁止新授权路径使用。 |
| DeviceHello canonical（M3-05） | DeviceHello `protocolVersion=1`、credential assertion/status version `1` 各自位于 `modcpt.identity.device-hello.v1` / `credential-assertion.v1` / `credential-status.v1` 独立域，与 ALPN major、quarantine payload version、manifest/profile version、application header version 同值不同域。纯契约不分配 ALPN/Cap'n Proto/control tag/SQLite/FRB 编号。 | pass；同数值跨独立命名空间逐一标注；`DeviceHelloError` 为新稳定类别（无数字码），不与 v1 `IdentityError` 互译。 |
| M4-01 header / MsgType | route header version `0x02` 只登记给 `MsgType::Envelope(0xB0)`；v1 类型继续 `0x01`；`0xB1-0xBF` 未分配。 | pass；v1/v2 双向拒绝矩阵测试；Envelope 在 v1 路径返回 None，v1 帧在 Envelope 路径被拒，零副作用。 |
| M4-01 Cap'n Proto | `MessageEnvelope`/`EnvelopeRecipient` 是新 struct，ordinal 独立；与 v1 `Envelope` 不同命名空间，不碰撞。 | pass；conversation kind 与 ID 长度、protocolVersion、signature 长度均有 fail-closed 校验。 |
| M4-01 SQLite | v6 只新增 `v2_messages`/`legacy_v5_quarantine`，不改写 v1 行、不自动生成 ID；`insert_v2_message` 唯一键 `(realm, origin device, message id)`。 | pass；`m<seq>`/字符串 v5 行在 `legacy_v5_quarantine` 原值归档；同键同字节幂等、同键异内容 conflict。 |
| M4-01 FRB | `parse_conversation_id` 以 typed DTO 边界拒绝非 canonical/跨 kind 输入。 | pass；typed boundary rejection 测试，core/storage 零副作用。 |
| M4-03 receipt / SQLite | content kind `2` 与 receipt domain 独立于 application content；v7 只新增 per-device outbox/inbox 表。 | pass；recipient-device signature/commitment binding、伪造拒绝、4096 cap、retry/expiry/restart 与 v6→v7 rollback tests 均通过。 |
| M4-04 attachment | content kind `5` 独立于 kind `1/3/4`；attachment profile 1 不复用 Olm cipher；无新 MsgType/schema/migration。 | pass；secret descriptor 逐设备 Olm seal，public chunks 使用 ChaCha20-Poly1305；context/final commitment、atomic v8 outbox、bounded reassembly、exact receipt 与 no-legacy-fallback tests。 |

## 10. Verification

执行任务卡规定的枚举命令，并将命中与本表逐项比对：

```text
rg -n "ALPN|MsgType|schema_version|SCHEMA_VERSION|UnsupportedProtocol|BIND" rust flutter Knowledge
```

复核时还必须确认：同一 namespace 无重复值、`deprecated` 值不被重分配、`reserved` 值没有被描述为已实现功能，以及 Board 仍由协调 agent 而不是本任务修改。
## 11. M7 Mailbox v1 registry

> 2026-08-06：M7-01 选择 Mailbox-first。以下值只属于独立 `modcpt_mailbox` engine/runtime/RPC/wire/transport；没有分配 core P2P ALPN、route `MsgType`、Cap'n Proto tag、FRB API 或既有 account server RPC。Room/媒体、relay 与 push 仍冻结。

| 值 | 状态 | Owner | 语义与复用规则 | 源码证据 |
|---|---|---|---|---|
| dependency `mailbox -> core` | implemented | M7-01 / `modcpt_mailbox` | core 不得反向依赖 mailbox；不创建直接 QUIC 失败 fallback | `rust/mailbox/Cargo.toml`; `rust/core/src/mailbox_adapter_v1.rs` |
| mailbox schema version `1` | implemented | M7-01 / mailbox | SQLCipher only；envelope/delivery/dedup/ack tombstone/rate/audit 表；未知版本 fail-closed | `MAILBOX_SCHEMA_VERSION_V1`; `initialize_schema` |
| delivery ID domain `modcpt.mailbox.delivery-id.v1` | implemented (storage key) | M7-01 / mailbox | SHA-256 over exact realm/origin device/message/recipient user+device；不是业务 MessageId、wire ID 或授权 token | `DELIVERY_ID_DOMAIN_V1`; `delivery_id` |
| mailbox envelope cap `1 MiB`; recipients `<=8` | implemented | M7-01 / core adapter + identity | mailbox cap 比 P2P Envelope 更严格；recipient cap 继承 canonical signed Envelope | `MAX_MAILBOX_ENVELOPE_BYTES_V1`; `MAX_ENVELOPE_RECIPIENTS` |
| TTL `60s..7d` | implemented | M7-01 / mailbox | trusted upload clock；正文、delivery/dedup/ack metadata 不得越过各自 expiry | `MIN_MAILBOX_TTL_MS_V1`; `MAX_MAILBOX_TTL_MS_V1` |
| pending/device `4096` + `64 MiB`; retained message keys `65536`; fetch `64` + `4 MiB` | implemented | M7-01 / mailbox | retained cap 包含 ack 后 TTL 内 dedup；N+1 在 admission transaction 失败；不得静默裁剪 fan-out | `MAX_PENDING_*`; `MAX_RETAINED_MESSAGE_KEYS_V1`; `MAX_FETCH_*` |
| 60s rate: upload `60`, fetch `120`, ack `600` | implemented | M7-01 / mailbox | key 是 exact verified realm/device；稳定 `mailbox_rate_limited` + retry_after | `RATE_WINDOW_MS_V1`; `*_PER_WINDOW_V1`; `consume_rate` |
| dedup `(RealmId, OriginDeviceId, MessageId)` | implemented | M7-01 / mailbox | exact digest idempotent；不同 digest `mailbox_dedup_conflict`；ack 后 tombstone 保留到 TTL | `mailbox_dedup_v1`; `MailboxStoreV1::upload` |
| stable error strings `mailbox_*` | implemented | M7-01 / core adapter + mailbox | 文案不是错误身份；authorization/malformed/origin/signature/rate/quota/capacity/conflict/storage 类别不可复用为另一语义 | `MailboxAdapterErrorV1::code`; `MailboxErrorV1::code` |
| blocking operations default/max `32`；overload retry `100 ms` | implemented (transport-neutral runtime) | M7-01 / mailbox runtime | `0` 或 `>32` 配置拒绝；许可满立即 `resource_exhausted`，不得创建无界等待队列；不代表网络连接/RPC 上限 | `DEFAULT_MAILBOX_BLOCKING_OPERATIONS_V1`; `MAX_MAILBOX_BLOCKING_OPERATIONS_V1`; `OVERLOAD_RETRY_AFTER_MS_V1` |
| stable runtime error strings `mailbox_runtime_*` | implemented (local host API) | M7-01 / mailbox runtime | invalid config/closed/executor/registry/operation/terminal-race 身份不可复用；不是网络 wire code，未来 RPC 需独立登记 | `MailboxRuntimeErrorV1::code` |
| RPC request active/terminal caps `1024/1024` | implemented (transport-neutral owner) | M7-01 / mailbox RPC | key 是 exact verified realm/user/device/instance/lease/request；active/terminal replay fail-closed；不是 wire/ALPN | `MAX_MAILBOX_RPC_REQUESTS_V1`; `MAX_MAILBOX_RPC_TERMINALS_V1` |
| stable local RPC errors `mailbox_rpc_*` | implemented (transport-neutral owner) | M7-01 / mailbox RPC | in-flight/already-terminal/capacity/state/runtime/executor 语义不可复用；未来网络错误映射需独立登记 | `MailboxRpcErrorV1::code` |
| wire domain `MCMBX001`；version `1`；op upload/fetch/ack/cancel=`1/2/3/4` | implemented | M7-01 / mailbox wire | 双向 big-endian framing；request flags=0、UUIDv4 RequestId；response status success/failed/cancel/deadline=`0/1/2/3`；exact length、有限 error code、4 MiB fetch aggregate 与 trailing reject | `wire_v1.rs` |
| ALPN `modcpt-mailbox/1` | implemented | M7-01 / mailbox transport | 独立于 core P2P/account server；TLS 1.3、严格 client CA、TLS leaf SPKI-bound initiator DeviceHello；未知/未来版本不协商、不回退 | `transport_v1.rs::MAILBOX_ALPN_V1` |
| connections `128`；business streams/connection `32`；DeviceHello `16 KiB`；auth/request timeout `10s/30s` | implemented | M7-01 / mailbox transport | N/N+1 admission、读取长度与连接期 `(DeviceId, nonce)` replay 都 fail-closed；不代表跨主机 load/SLO 已满足 | `MAX_MAILBOX_*_V1`; `MAILBOX_*_TIMEOUT_V1` |
| max mailbox request `1 MiB + 40 bytes` | implemented (framing cap) | M7-01 / mailbox wire | 总长在 body allocation 前检查；upload body 仍须 core canonical Envelope gate | `MAX_MAILBOX_WIRE_REQUEST_BYTES_V1` |
| load summary schema `modcpt.mailbox.load.v1` | implemented (validation output) | M7-01 / mailbox load | 固定 seed/count/concurrency/p99/budget/pass 的单行 JSON；不是网络/存储协议，字段语义改变需新 schema | `examples/mailbox_load_v1.rs` |

## 12. M7 release artifact registry

| 值 | 状态 | Owner | 语义与复用规则 | 证据 |
|---|---|---|---|---|
| `modcpt-server-windows-x86_64-{version}.zip` | implemented (dry-run tooling) | M7-02 / release | 首个 server release unit；ZIP exact allowlist；真实 RC 必须 clean exact tag + timestamped Authenticode | `New-ModCptServerRelease.ps1`; `Test-ModCptReleaseBundle.ps1` |
| release manifest schema `modcpt.release.server.v1` | implemented (dry-run tooling) | M7-02 / release | 绑定 version/tag/commit/dirty/dryRun、artifact/files、sidecars、signature、compatibility/toolchain；未知 schema fail-closed | `*.zip.release.json`; consumer negative matrix |
| `modcpt-mailbox-windows-x86_64-{version}.zip` / `modcpt.release.mailbox.v1` | implemented (dry-run tooling) | M7-01/M7-02 / release | Mailbox daemon exact ZIP/config/operations allowlist；ALPN/SQLite v1 exact compatibility；真实 RC 同样要求 clean tag 与 timestamped Authenticode | `New-ModCptMailboxRelease.ps1`; `Test-ModCptReleaseBundle.ps1` |
| `modcpt-app-windows-x86_64-{version}.msix` / `modcpt.release.app.v1` | implemented (dry-run tooling) | M7-02 / release | Flutter release + paired FRB DLL + MakeAppx；manifest 冻结 package identity、embedded file hashes、binding identity、内外 PE 签名与 license completeness。真实 RC 禁止 Allow* dry-run | `New-ModCptAppRelease.ps1`; `Test-ModCptAppReleaseBundle.ps1` |
| CycloneDX `1.5` + in-toto/SLSA v1 subject | implemented (unsigned dry-run) | M7-02 / release | SBOM subject digest/依赖闭包/ZIP files 与 provenance subject/commit 可消费验证；unsigned statement 不得称 attestation | `*.cdx.json`; `*.provenance.json` |
