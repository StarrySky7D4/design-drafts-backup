# Mailbox-first v1 ADR

- 状态：Accepted（2026-08-06）
- 决策：M7-01 先实现外部 opaque Envelope mailbox；Room/媒体、relay 与 push 保持冻结，留待 M7 后独立立项。
- 实现：`rust/mailbox/`；纯验证适配器：`rust/core/src/mailbox_adapter_v1.rs`。

## 决策与范围

Mailbox v1 只提供 `upload / fetch / ack`。服务保存 sender-signed canonical `MessageEnvelope` wire，不解析或解密 payload。它不是 relay、presence、discovery、push payload、Room/SFU，也不会在直接 QUIC 失败后自动触发 fallback。

依赖方向固定为：

```text
mailbox host -> modcpt_mailbox -> modcpt_core mailbox_adapter_v1
                                      -> modcpt_identity
```

`modcpt_core` 不依赖 `modcpt_mailbox`，core 网络路径没有 mailbox listener、retry loop 或 fallback。独立 network host 使用 `modcpt-mailbox/1` 严格 mTLS + DeviceHello，验证 TLS leaf SPKI 后才把内部 `SessionBinding` 和可信时钟传给 store；request wire 不能自行构造授权主体。同步 SQLite 调用通过 `runtime_v1` 运行在默认/最大 32 并发的 Tokio blocking pool admission 后，满载立即拒绝且不形成无界队列。

## 身份与内容边界

- 上传者的 current `SessionBinding` 必须与 Envelope 的 exact `RealmId / OriginUserId / OriginDeviceId` 一致。
- Envelope 必须是 canonical wire，并由 binding 中的 current device signing key 验证。
- recipient authority 来自 sender-signed canonical recipient set；mailbox 不按 SessionId、IP、地址簿或 v1 `IdentityFrame` 寻址。
- fetch/ack 只接受 exact current `(RealmId, UserId, DeviceId)`；跨设备 delivery ID 查询返回 `Unknown`，不泄漏其存在性。
- mailbox 不接触 plaintext、Olm ratchet state、message key、attachment file key、secret descriptor 或 MLS secret。

## 冻结资源合同

| 项 | v1 上限/语义 |
|---|---|
| envelope wire | 1 MiB；比 P2P Envelope 上限更严格 |
| recipients | 继承 canonical Envelope 上限 8 |
| TTL | 60 秒至 7 天；由上传时的可信时钟计算 |
| 每 recipient device pending | 4096 项、64 MiB |
| 全局 retained message keys | 65536；包含正文已 ack 但 dedup 仍在 TTL 内的记录，避免 upload+ack 绕过容量 |
| fetch | 每次最多 64 项、4 MiB；`uploaded_at_ms, delivery_id` 稳定排序 |
| ack | exact-device、幂等；正文最后一份 delivery ack 后删除 |
| dedup | `(RealmId, OriginDeviceId, MessageId)`；同 digest 幂等，不同 digest 冲突 |
| rate | exact verified device 每 60 秒 upload 60、fetch 120、ack 600 |
| audit | 4096 条 metadata-only event kind/time；不含 ID、wire、内容或密钥 |
| ack tombstone | 硬上限 8192 条且不超过原 delivery TTL |

authenticated request 的 rate admission 先以独立 `BEGIN IMMEDIATE` 事务持久计数，随后配额、去重和 fan-out 在另一个 `BEGIN IMMEDIATE` 事务中完成；因此 quota/conflict 失败也不能逃避 limiter，同时不会留下半份 Envelope/delivery。signature/origin 等未通过 core adapter 的请求不触碰 mailbox DB。最后一个 ack 删除正文，但独立 dedup tombstone 保留到原 TTL，防止 ack 后重放重新投递。

## 存储与隐私

生产入口只接受调用方持有的 exact 32-byte SQLCipher key，没有 plaintext/default-key constructor。schema v1 包含 envelope、delivery、dedup、ack tombstone、rate 和 metadata-only audit 表。服务仍能观察 realm、设备投递关系、时间、大小、状态与流量模式；最小版本不声称隐藏社交图或抵抗全局流量分析。

## 失败语义

公开稳定类别包括 authority expired、malformed/non-canonical/too-large Envelope、origin/signature failure、invalid request、rate limited（带 `retry_after_ms`）、recipient quota、global capacity、dedup conflict 与 storage failure。文案不是错误身份。

## 运行时、取消与关闭

`MailboxRuntimeV1` 复用 core 的有界 operation registry、typed terminal 与固定枚举 metrics。blocking 工作开始前，cancel 或 deadline 产生零 store 副作用；开始后同步 SQLCipher transaction 不可安全中断，因此真实 store 结果优先于迟到的 cancel/deadline。调用方丢弃等待 Future 不代表取消：独立 managed task 继续持有 operation 和 semaphore permit，直至真实工作完成并只提交一次终态。`shutdown()` 停止 admission 并等待所有已开始工作释放许可。

`runtime_v1` 保持 transport-neutral；`rpc_v1`、`wire_v1` 与 `transport_v1` 在独立 mailbox crate 中分配 exact-session owner、`MCMBX001` 双向 framing 和 `modcpt-mailbox/1` ALPN，不分配 core MsgType、Cap'n Proto tag、FRB API 或既有 server RPC。host 已完成严格 mTLS/DeviceHello、request/response size/time budget、连接/stream/replay 限流；生产部署、证书轮换与可观测性 exporter 仍需单独接线。

## 明确冻结

- Room/媒体及现有 Room state machine 不进入本路线。
- relay、STUN、TURN、CID relay、DHT relay、NAT 探测/穿透与 core 自动 fallback 不实现。
- push 及平台 token 生命周期不实现；若未来仅作 wake-up，也需独立隐私和 payload 评审。
- mailbox 的跨主机部署 topology、证书轮换、生产 SLO、平台发布与回滚仍属于 W37/W38；当前 crate 已有存储/授权引擎、bounded runtime/RPC、双向 wire、QUIC host/client 与真实两主机 24h soak，但不等于生产服务已部署。

## 验证

定向测试覆盖 valid upload/fetch/ack、重复上传、ack 后重放、相同 key 不同 wire 冲突、伪造签名、跨设备 ack、TTL、N/N+1 quota/rate/global capacity、稳定 fetch bound、SQLCipher 重开/错误 key/非 plaintext header，运行时 cancel/deadline/overload/Future drop/唯一终态/graceful shutdown，双向 framing，以及真实 mTLS/DeviceHello fetch/ack 与 SPKI mismatch 拒绝。workspace full、跨主机 load/soak 与 release gates 仍按 M7 计划执行。
