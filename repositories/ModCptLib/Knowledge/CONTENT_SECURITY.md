# 内容安全与密钥生命周期

> **状态**：M2-03 于 2026-08-02 完成 M4-05 security behavioral re-review。direct text 当前采用精确固定的 `vodozemac =0.10.0` native Olm v2 profile、credential-bound content prekey、逐设备 ciphertext、SQLite v8 原子状态事务和 post-decrypt type gate。附件 file-key/chunk runtime、群组 rekey 与 Room 媒体加密仍未实现；这些未完成域不得借 direct-text 结果宣称端到端安全。

> **边界**：本页选择经过审查的协议族和标准原语，不授权自定义 ECDH+HMAC、TLS-only、签名-only 或 HMAC-only 内容方案。所有未来 wire profile identifier、ALPN、capability、schema field、resource limit 和错误码必须先进入[协议注册表](PROTOCOL_REGISTRY.md)，再实现消费者。

## 1. 安全结论与范围

### 1.1 一对一内容 profile

一对一文本和附件必须使用以下分层 profile：

| 层 | 冻结结论 | 不能替代的属性 |
|---|---|---|
| 设备认证 | M3 的 `PeerPrincipal`、credential/status、realm、SPKI、持钥 proof 与 lease 决定设备是否可建立内容会话 | 传输 TLS、昵称、IP、`IdentityFrame`、裸公钥或 session string |
| 异步首次会话 | native Olm 3DH：独立 X25519 account identity key、initiator base key 与 server 原子领取的 OTK；OTK 耗尽时只允许已签名 fallback key。identity/OTK/fallback canonical record 均由 M3 credential 的 Ed25519 key 签名 | Ed25519→X25519 转换、服务端生成 private prekey、静态 key/TLS fallback |
| 会话消息 | `vodozemac 0.10.0` `SessionConfig::version_2()` native Olm Double Ratchet；X25519 ratchet、库原生 root/chain KDF、单次 message key、full MAC | Olm v1 truncated-MAC、静态会话 key、自定义 receive-key callback |
| 内容保护 | native Olm v2 AES-256-CBC + full HMAC-SHA-256。该 profile **没有 external AD API，也不是原冻结 ChaCha20-Poly1305 profile** | 沿用旧 ChaCha/AD 审计声明、把 outer TLS/signature 当作内容加密 |
| 内容来源与 metadata | outer Envelope signature 认证 exact transport/storage wire；inner Ed25519 signature 只签每个 recipient 的 `EnvelopeCommitment`。realm/conversation/origin credential/recipient set/recipient/message/kind/time/session/ciphertext digest 同时进入 commitment，且相同 metadata 复制进 Olm-encrypted plaintext，open 后逐字段比较 | 只验证 QUIC peer、只验证 outer signature、未比较 decrypted metadata |

X25519 使用 RFC 7748，Ed25519 使用 RFC 8032；ratchet/cipher/MAC 只通过精确固定的 vodozemac native API 使用。项目没有 fork、暴露 receive key 或自行拼装 cipher/KDF。2022 Least Authority audit 是上游历史证据，不自动覆盖 0.10.0 之后的全部变化；升级 vodozemac、启用其他 session config 或改变 wrapper canonical fields 必须重新走本页和协议生命周期评审。

M4-05 已登记 direct profile/version、content kind、canonical domains、RPC、SQLite v8、资源参数和稳定字符串错误类别；没有新增 Cap'n Proto ordinal、MsgType 或 ALPN。

### 1.2 设备对与收件人集合

一个 `ConversationId::Direct` 的每个 **发送 DeviceId -> 接收 DeviceId** 对拥有独立 ratchet session。向对方用户的多个活动设备及发送者其他活动设备投递时，发送者为每个目标设备分别 ratchet-encrypt 同一不可变内容 commitment；不得用一个 pairwise ciphertext 或一个 session key 广播给多个设备。

每条业务消息冻结 recipient set：按 canonical `(UserId, DeviceId)` 排序后的目标集合及其 SHA-256 commitment。该集合、`RealmId`、`ConversationId`、origin credential/device、`MessageId`、内容类型和 profile version 必须同时进入每份 signed `EnvelopeCommitment` 与 Olm-encrypted metadata duplication。native Olm v2 没有项目 external AD；接收方不在集合中、集合 commitment 不匹配或本机 `DeviceId` 不匹配时，必须在业务投递前拒绝。

## 2. 密钥层级与生命周期

```text
Device Ed25519 signing key       Device X25519 identity-DH key
          |                                  |
          +-- signs signed prekey -----------+
                                             |
authenticated X3DH/prekey transcript -> initial root key -> Double Ratchet root key
                                                       |             |
                                                   send chain     receive chain
                                                       |             |
                                                   message key   message key (single use)

random attachment file key -> per-chunk AEAD keys/nonces
attachment key envelope -> each recipient's ratchet-encrypted content payload
```

| 材料 | 创建与绑定 | 使用/销毁 | 禁止 |
|---|---|---|---|
| Device signing key | 由 M3 credential 绑定 `RealmId/UserId/DeviceId/KeyId`；私钥只在设备安全存储 | 签 identity/OTK/fallback canonical records 与 content commitment；credential revoke/设备擦除时销毁 | 作为 X25519 DH key、上传到 service、共享给另一设备 |
| Device identity-DH key | X25519 key，与同一 device credential/key id 绑定 | 仅 X3DH/ratchet DH；rotation 后新会话使用新 key | 从 Ed25519 bytes 隐式转换、跨 realm/DeviceId 重用 |
| content identity/prekey record | X25519 identity/OTK/fallback，带用途、key id、7-day validity、realm/user/device/credential/profile binding 与 Ed25519 signature | identity 长期留在 encrypted account pickle；OTK 只在 inbound prekey 完整 decrypt 成功后删除；fallback 可复用且每次补充轮换 | 无限期接受、未验证签名/credential、把 Ed25519 bytes 转为 DH key |
| one-time prekey | X25519，server 在单 SQLite transaction 中最多交给一个 initiator；并发 claim 只有一个 winner | 首次 handshake 成功后删除 private OTK；失败时 account clone 不提交；本地补充新 OTK 后 server 与未领取 OTK 合并 | 一次交给多个 initiator、服务端生成 private key、失败后误消费 |
| root/chain key | 仅由 native Olm 3DH/Double Ratchet KDF 得出 | 由 vodozemac 内部推进并随 encrypted pickle 保存必要状态 | 由项目读取/记录、用于附件、跨 session/realm 复用 |
| message key | 只由 native Olm 对应 chain step 得出，项目 API 不暴露 | 只 seal/open 一个 native Olm message；生命周期由 vodozemac 管理，项目不单独持久化 | 暴露或复用 key、用于多个 recipient、持久备份明文 key |
| attachment file key | 32-byte CSPRNG，每个 `TransferId` 一个 | 仅该 attachment chunks；在每份 recipient content payload 内封装；transfer 结束/撤销/擦除后删除 | 从 MessageId 或内容 hash 派生、跨 transfer 使用、放入公共路由 header |

Olm account/session state 与 skipped keys 只以 vodozemac encrypted pickle 存入 core SQLite v8；wrapping key 由 local-v2 高熵 `device_secret` 经 `modcpt.content.olm-v2.state-wrapping-key.v1` 域分隔 SHA-256 导出并在 runtime drop 时 zeroize。数据库不保存 wrapping key 或 plaintext。M5 仍负责平台 secure-storage、备份/导出和完整静态数据治理；M4-05 不允许备份 active ratchet/prekey state。

## 3. 首次会话、同时发起与消息状态机

### 3.1 首次会话

1. 发送方只从已认证、current、未撤销且有有效 prekey bundle 的目标 `DeviceId` 建立 session。
2. 发送方先验证 canonical signed claim 对 `RealmId/UserId/DeviceId/CredentialId/key version/profile/validity/purpose/key id/public bytes` 的绑定，再把 claim 中 exact identity/OTK-or-fallback X25519 public keys交给 native Olm `create_outbound_session(version_2)`。项目 metadata 不被伪装成 native Olm KDF context；其绑定由签名 claim、session tag、inner content commitment、outer Envelope 与 encrypted metadata comparison共同完成。任一不匹配即 fail-closed。
3. 首条 prekey message 的内容只可在完整 transcript 与 signed prekey 验证后解密；one-time prekey 的消费与首次 session 建立须原子，失败不得留下可授权 state。
4. prekey bundle 缺失、耗尽、过期、未知或 status 不确定时不得退回静态 key、旧 `IdentityFrame`、TLS session、裸 key 或 server-readable fallback；发送方保留待投递项并等待重新 provision 的 bundle 或报告明确失败。

### 3.2 同时发起

两端可并发发送合法 payload-free prekey offer，但一个 unordered `(RealmId, DeviceId_A, DeviceId_B, profile)` 最终只保留一个 canonical ratchet session。双方持久化相同 candidate set 后以 session tag 字典序较小者为 winner；另一个 session 标记 `superseded`。业务 payload 在 Active winner 产生前不 seal，因此无需把 loser 上的业务密文重新封装或交付。

该规则不允许按到达顺序、IP、临时 QUIC session、local clock 或“谁先成功”决定 winner；不同设备对的 session 相互独立。

M4-05 使用持久化 payload-free `Offer → Candidates → Confirm → Ready` 协调。candidate set 最多 2 个，双方在相同集合上按 session tag 字典序选 winner；SQLite v8 保存 phase/candidates，Ready 与 winner session/outbox/inbox 同事务提交，loser session 标记 `superseded` 且其 pending bootstrap outbox 终止。业务 seal 只接受 `Active` winner，禁止 timeout/到达顺序仲裁。

### 3.3 乱序、丢失、重放与取消

- 项目不解析或重编码 native Olm ratchet header。`session_tag = SHA-256(native session id)`、Olm message type 与 SHA-256(Olm bytes) 进入 signed `EnvelopeCommitment`；接收方只把 exact Olm bytes交给已认证、匹配 tag/principal 的 `SessionConfig::version_2()` state。
- 乱序/skipped-key 由 exact vodozemac 0.10.0 执行：message gap 2000、每 receive chain 40 个 skipped keys、最多 5 条 receive chains；项目 session 4096、pending bootstrap 256。超限、回退 counter、未知 session 或缺失 key fail-closed，不 fork 或放宽上游限制。
- retransmit 使用相同 `MessageId`、相同 canonical commitment 与字节完全相同的 ciphertext；不得用已消费 message key 重新 seal 不同明文。
- dedup key 至少为 `(RealmId, OriginDeviceId, MessageId)`，并保存首个 accepted commitment/ciphertext digest。相同 key 且字节相同是幂等、不重复 UI/业务副作用；相同 key 但不同 commitment/ciphertext 是 origin conflict，拒绝并审计。
- auth、AEAD、signature、AD、counter、recipient、ratchet 或 resource 验证失败时，不创建 message/event/attachment/receipt，不推进可见业务状态。已获得但未使用的临时 key 必须释放；取消、超时和断线不得泄漏 pending session 或 skipped keys。

## 4. AEAD、nonce 与 associated data

> **M4-05 re-review 覆盖说明**：本节原 ChaCha20-Poly1305 + external `HeaderBytes` AD 是未实施 strict profile 的历史设计，不是当前 direct-text wire。当前 profile 由 native Olm v2 自行生成 AES-CBC IV/message key/full HMAC；wrapper 不访问 message key，也不伪造 external AD。当前唯一规范 binding 是 `modcpt.content.olm-v2.envelope-commitment.v1`：长度前缀 canonical metadata + session tag + Olm message type + SHA-256(Olm bytes)，再由 M3 device Ed25519 key签名；相同 metadata 进入 `modcpt.content.olm-v2.plaintext.v1` 并在解密后逐字段比较。以下旧 nonce/HeaderBytes 公式只保留为 rejected strict-profile 取证，不得由实现或测试引用。

### 4.1 文本内容

每个 Double Ratchet message key 只能加密一个确定性 content ciphertext。该 key 的 nonce 为：

```text
HKDF-Expand(message_key,
  "modcpt.content.message-nonce" || HeaderCommitment, 12)
```

同一 message key 的重传必须复用原 ciphertext，禁止再次 seal。不同 message key 即使得到相同 96-bit nonce 也不构成 nonce reuse；nonce 唯一性的责任是 **一个 key 仅一条 plaintext/ciphertext**。实现必须对重复 seal fail-closed，而不能依赖随机 nonce 偶然不碰撞。

所有 commitment 使用同一个有长度前缀、固定字段顺序的共享 canonical encoder；不得用 JSON、map iteration、Cap'n Proto packed bytes 或展示文本代替。其关系固定如下：

| 名称 | 精确输入 | 用途 |
|---|---|---|
| `RecipientSetCommitment` | `SHA-256(domain "modcpt.content.recipient-set" || canonical sorted (UserId, DeviceId) set)` | 绑定每个目标设备与同一冻结投递集合 |
| `RatchetHeaderCommitment` | `SHA-256(domain "modcpt.content.ratchet-header" || RealmId || ConversationId || OriginDeviceId || RecipientDeviceId || bootstrap transcript hash || current DH ratchet public key || previous-chain length || message number)` | 绑定赢家 session 和标准 Double Ratchet header；不得从临时 QUIC session、IP 或到达顺序派生 |
| `HeaderBytes` | 下列 AEAD AD 的有长度前缀 canonical field set，不含 ciphertext digest | AEAD associated data 的唯一 preimage |
| `HeaderCommitment` | `SHA-256(HeaderBytes)` | message nonce 派生、ratchet/session commitment 与 envelope binding |
| `AttachmentDescriptorCommitment` | `SHA-256(domain "modcpt.content.attachment-descriptor" || complete descriptor including transfer_nonce_prefix)` | 绑定 file key、TransferId、长度、chunk 规则及附件 metadata |
| `EnvelopeCommitment` | `SHA-256(domain "modcpt.content.envelope" || HeaderCommitment || AttachmentDescriptorCommitment-or-empty || ciphertext digest)` | Ed25519 content signature、dedup conflict 判断与 receipt binding |

AEAD associated data 就是 `HeaderBytes`，即 `HeaderCommitment` 的唯一 canonical preimage，固定包含：

```text
domain = "modcpt.content.direct"
profile version (registered before implementation)
RealmId
ConversationId
OriginUserId, OriginDeviceId, origin credential/key id
RecipientSetCommitment, RecipientDeviceId
MessageId
RatchetHeaderCommitment
content kind and attachment-descriptor commitment (if any)
```

签名输入只能是 `EnvelopeCommitment`；签名、AEAD AD、业务 envelope、receipt 和 dedup 必须引用这些命名 commitment，不能各自重新排序或省略 `RealmId`、origin、recipient set、conversation、message/transfer 标识。M4 必须用公开合成 fixture 建立正向、篡改、替换、跨 realm、错误 recipient、错误 origin 和错误字节序固定向量。

M4-03 `DeliveryReceipt` 已登记为 recipient-device-signed Envelope content kind `2`。M4-05 现在提供每个 recipient entry 的真实 `EnvelopeCommitment` producer；成功 decrypt、metadata comparison、session/inbox commit 后，同一 SQLite transaction 写 exact receipt outbox。canonical outer wire SHA-256 仍只用于本地 dedup，不得冒充内容 commitment。

### 4.2 附件

M4-04-I1 当前唯一实现 profile 是 attachment version `1`、profile `1 = ChaCha20-Poly1305`。附件 descriptor 位于每个 recipient 独立的 Olm-v2-encrypted direct plaintext 中，包含 `TransferId`、plaintext/ciphertext 长度、chunk count/rule、`AttachmentCiphertextDigest`、mime/name、32-byte CSPRNG file key、随机 8-byte `transfer_nonce_prefix` 和 `RecipientSetCommitment`。公开 content-kind 5 chunk Envelope 只能看到 ciphertext、路由主体、recipient 与 commitments；server、relay、mailbox 和公开路由 header 不得看到 file key、nonce prefix 或明文 descriptor。

旧草案若直接把最终 `AttachmentDescriptorCommitment`（其中包含 ciphertext digest）放入用于生成该 ciphertext 的 AEAD AD，会形成循环依赖。实现与现行规范因此冻结两层 commitment：

- `AttachmentContextCommitment = SHA-256("modcpt.content.attachment-context.v1" || canonical descriptor fields excluding final ciphertext digest)`；它覆盖 file key、nonce prefix、版本/profile、realm/conversation/origin、recipient set、TransferId、长度/count/rule 和 metadata，并作为 chunk AEAD AD 的核心。
- `AttachmentDescriptorCommitment = SHA-256("modcpt.content.attachment-descriptor.v1" || complete canonical descriptor including final ciphertext digest)`；它在加密完成后生成，绑定最终完整 descriptor。
- `AttachmentCiphertextDigest = SHA-256("modcpt.content.attachment-ciphertext.v1" || TransferId || chunk_count_u32_be || (chunk_index_u32_be || SHA-256(chunk_ciphertext))*)`；只接受 canonical ordered concatenation，不使用可选 Merkle 变体。

每个 `TransferId` 使用独立 file key 与随机 nonce prefix。chunk nonce 为 `transfer_nonce_prefix[8] || chunk_index_u32_be[4]`；AEAD AD 使用 `modcpt.content.attachment-chunk-aad.v1` 并覆盖 context commitment 与 chunk index。公开 chunk 同时携带 context/descriptor commitments、最终 digest、recipient 与 exact length/count，因此 ciphertext、key、TransferId、descriptor、recipient、index 或长度任一错配都在业务交付前失败。

profile 上限为 ciphertext chunk 16 KiB、plaintext chunk 16,368 bytes、总 ciphertext 8 MiB、总 plaintext 8,380,416 bytes、8192 chunks；当前 plaintext 上限由 512 个最大 plaintext chunk 推导，避免 AEAD tags 使 8 MiB ciphertext cap 溢出。name 最大 255 UTF-8 bytes且禁止路径分隔符/控制字符；MIME 最大 127 ASCII bytes且禁止空格/控制字符。

附件 dedup key 为 `(RealmId, OriginDeviceId, TransferId)`；相同 key 的不同 descriptor/digest 是冲突而非新附件。文件完整性只在每个 chunk AEAD、index coverage、精确长度和最终 ciphertext digest 均验证后成立。完成后短期保留 exact final-chunk receipt basis，重试只重建回执而不重复交付；回执原子进入 durable queue 后释放。

## 5. 设备轮换、撤销、克隆、历史和恢复

| 事件 | 唯一规则 |
|---|---|
| continuous credential/key rotation | M3 验证连续 proof 后，新出站内容立即以新 credential/key 和新/重建 ratchet session 加密。旧 session 仅在双方旧 credential 仍 current、未到期且收到内容前状态复核成功时可消费；不得用旧 session 掩盖 revoked/unknown status。 |
| revoke | status refresh 发现 revoked 后，停止该 device 的新 session，将本地 session 置为不可用并终止 queued delivery；revoked/superseded bootstrap 在 crypto 前拒绝且不能恢复为 Pending。后续可按数据治理策略物理清理 encrypted pickle。无法撤回该设备已拥有的历史 plaintext、message key 或 attachment file key。 |
| software-device clone | 不宣称密码学可区分两个相同私钥副本。lease/instance 审计与 credential revoke 负责处置；克隆可在撤销前读取其复制的本地 ratchet state。 |
| new device | 新设备只从加入/认证后的新 prekey session 接收未来消息。**默认不能解密历史消息、历史 ratchet state、历史 attachment file key 或备份中的旧 session。** 用户显式重新发送历史内容时，它是新的 MessageId/TransferId 和新的加密事件。 |
| backup/restore | 备份不得把 active ratchet root/chain/message keys、one-time prekeys、unwrapped attachment file keys 或旧 lease 作为可恢复授权材料。恢复设备必须重新验证 current credential/status，必要时作为新 device/recovery 处理。 |
| logout/device erase | 删除 local ratchet state、skipped keys、prekeys、attachment keys 和安全存储包装；服务端 revoke/lease 状态由 M3/M5 流程处理，不能仅依赖本地删除。 |

## 6. 密钥泄漏影响与销毁时点

| 泄漏材料 | 影响范围 | 不能得出的结论 | 必须处置 |
|---|---|---|---|
| 单 message key | 对应一条 ciphertext | 不推出 chain/root/其他 message 或附件 key | 解密/发送后擦除；记录安全事件时不记录 key/明文 |
| attachment file key | 对应 TransferId 的可得 chunks | 不推出 ratchet、其他 TransferId 或其他 recipient key | transfer 完成/撤销/擦除后删除；必要时重发新 TransferId |
| send/receive chain key | 同一方向在下一 DH ratchet 前的未来消息可能受影响；已擦除 message keys 的过去消息不由该 key 直接恢复 | 不保证对方、另一方向或其他 session 失陷 | 触发受认证 DH ratchet/resync，审计并按 credential policy 评估 revoke |
| 当前 root/完整 ratchet state | 可能读取保留 skipped keys、当前链未来消息；攻击者移除后，未来成功 DH ratchet 可恢复 post-compromise security | 不保证历史已擦除 keys、其他 device pair 或 attachment keys 全部泄漏 | 关闭/重建 session，撤销或轮换受影响 credential，擦除旧 state |
| identity-DH/prekey private key | 可能危及仍可用 prekey 的首次 session；不自动解密已完成且已 ratchet 前进的所有历史 | 不等于 Ed25519 signing key 或本地内容库泄漏 | 停止 prekey、发布/验证新 bundle、轮换 credential，拒绝旧 bundle |
| device signing key | 可伪造该设备的未来 signed prekey/content commitment，直到撤销生效 | 不自动解密过去已擦除 message/file keys | 紧急 revoke/replace device；所有对端停止新 session 并提示安全事件 |
| 已解锁本地端点 | 可读其可访问 plaintext/state | 不存在纯协议自动补救或“安全存储保证” | 用户/平台处置、logout/erase、credential revoke 与新 device recovery |

“前向保密”只适用于已按此表擦除的 message/chain state，且不包含已解锁端点、已下载内容、未擦除备份或 identity signing key 被长期控制的情形。

## 7. 群组与 Room：明确 blocked 边界

### 7.1 小群

M6 前群组内容安全保持 **blocked**。唯一允许的目标协议族是经审查的 MLS/树形 group-ratchet 类方案：成员 leaf key、group epoch、commit/transcript hash、application secret、sender data/ratchet 与成员资格状态必须显式绑定。群治理事件的签名或 HMAC 只能认证治理/完整性，不能充当群内容保密。

- 成员加入、退出、撤销或 device key rotation 必须创建新 epoch；新 epoch 内容不得在成员变更 commit 验证前投递。
- 被移除成员可以保留其已解密的历史，但不得解密之后 epoch；新成员默认不得解密加入前历史，除非单独、显式、可审计地重新加密分享。
- group `MessageId/EventId/TransferId` 的 dedup 必须包含 origin device；管理员角色不能绕过 AEAD、sender authorization、epoch 或 replay 验证。
- 当前 secure-group、Room state machine、签名和 HMAC 不得被描述为实现了这些内容安全结论。

### 7.2 Room 媒体

M7 已选择 Mailbox-first，Room/媒体保持 **frozen for a future milestone**。若未来重新批准，最低要求是：每媒体帧使用 AEAD 而非 HMAC-only；AD 绑定 RealmId、RoomId、origin device、stream/track、epoch、frame sequence、codec configuration commitment 和 recipient/membership epoch；重放窗口有已登记上限；成员/密钥变化立即推进 epoch；已应用新 epoch 后旧 epoch frame 一律拒绝，不以乱序为由无限接受旧 key。媒体 nonce、key distribution、SFU/relay 可见元数据和 retransmit 规则必须在实现前登记并建立独立 vectors。

### 7.3 Mailbox v1

Mailbox 只保存通过 current SessionBinding exact-origin 和 sender signature 验证的 canonical Envelope wire，不解密 payload。服务可见 realm、origin/recipient device、MessageId、时间、大小、TTL 和投递状态，因此不声称隐藏社交图或流量模式；audit 只保存有界 event kind/time。dedup `(RealmId, OriginDeviceId, MessageId)` 在正文删除后继续保留至 TTL，防止 ack 后重放。SQLCipher 保护静态数据库，但不消除服务运行时可见 metadata。详见 `designs/mailbox-first-v1.md`。

## 8. 观察者可见性

| 观察者 | 可见 | 不应可见 | 剩余风险 |
|---|---|---|---|
| 直接网络路径/对端 transport | IP/端口、时间、包大小、频率、已协商 transport metadata | 内容 plaintext、attachment key、ratchet state | 流量分析、DoS、连接关联 |
| realm service / future relay / mailbox | 账户与投递所需的 realm、目标 device/routing、时间、大小、状态与审计元数据 | 文本、附件 plaintext、file key、ratchet/message key、完整 content descriptor | 可丢弃、延迟、重放密文；最小发布不隐藏 recipient set/社交图/流量特征 |
| 未获授权 device | 公共路由最小 metadata | 不能通过复制 ciphertext、TransferId、MessageId 或旧 epoch 解密内容 | 若持有被复制的本地 state/合法旧 key，按泄漏表处置 |
| 本地存储窃取者 | 加密 blob、受保护 metadata 与可能的文件密文 | 在 M5 静态保护完成后不应得到未包装 key/明文 | 已解锁设备、弱平台保护或旧备份可暴露可访问内容 |

日志、错误、fixture 和审计不得记录 token、私钥、prekey private material、message/file key、完整 plaintext、可重放 ciphertext 或完整 credential。

## 9. M2-03 安全评审走查

下列是 `TEST_MATRIX.md` M2-03 的唯一设计结论。均为 document-review，后续 M4/M5/M6 必须转为固定向量、组件和端到端测试；在实现证据产生前不得标为产品能力。

| ID | 冻结行为 |
|---|---|
| M2-03-T01 | 首次 session 必须验证 signed Olm prekey claim；同时发起通过 payload-free candidate protocol 选择最小 session tag winner，Active 前不 seal business；乱序使用 native 有界 skipped keys。 |
| M2-03-T02 | message key/IV/MAC 由 native Olm v2 管理且项目不暴露；Olm bytes、signed commitment、outer Envelope 或 encrypted metadata 任一篡改不交付 plaintext。 |
| M2-03-T03 | MessageId/TransferId dedup 至少含 RealmId 与 OriginDeviceId；同 ID 不同 ciphertext/descriptor 为冲突，不能冒充另一 origin。 |
| M2-03-T04 | file key 只在匹配 TransferId descriptor 的 recipient ratchet payload 内；chunk AD 绑定 descriptor/transfer/origin/recipient/index，错配必须失败。 |
| M2-03-T05 | rotation 重新建立新 outbound session；revoke 停止新会话并销毁本地待用 key；clone 按 lease/audit/revoke 处置，不声称可物理识别。 |
| M2-03-T06 | 新 device 默认无历史 ratchet/file key；backup 不恢复授权/旧 lease；历史共享只能显式重发为新加密事件。 |
| M2-03-T07 | native message key 不暴露给项目；file key 仍限单对象。chain/root/pickle/identity key 的不同影响和擦除/轮换责任按第 6 节执行。 |
| M2-03-T08 | 群组 MLS/epoch/rekey/history 规则在 M6 前 blocked；退出后不读新 epoch，加入者默认不读旧 epoch，HMAC 不是加密。 |
| M2-03-T09 | Room 媒体在 M7 前 blocked；若批准，AEAD、epoch、frame sequence 和有界 replay window 为最低门，旧 epoch replay 拒绝。 |
| M2-03-T10 | service/relay/mailbox/local storage 的内容与元数据可见性按第 8 节；TLS/直接拨号不构成应用内容保密或元数据匿名。 |

## 10. 实现门

M4-05 direct-text 实现在当前工作树已完成：profile/version/资源已登记，固定 native Olm v2 vector、per-recipient、tamper/downgrade、skipped-gap、simultaneous-init、atomic OTK、crash/replay、rotation/revoke/erase 测试已落地；最终 full gate、review、CI 与汇合 SHA 仍是发布门。M4-04-I1 仍须独立实现附件 descriptor/file-key/chunk AEAD 并完成附件矩阵；M5/M6 仍须分别关闭平台静态数据治理与群组内容安全。任何 vodozemac/profile/canonical 变更必须重跑 D3/D2-B 和安全矩阵。
