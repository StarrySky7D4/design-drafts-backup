# M7 W33 privacy-safe metrics schema

> 状态：W33 逻辑指标合同。仓库当前只有 tracing/log bridge；没有 metrics exporter、dashboard、alert、baseline 或已冻结 SLO。本文不选择 exporter，也不设 SLO 数值。

## 1. 通用规则

- metric namespace 为 `modcpt_`；counter 以 `_total`，duration 使用 milliseconds，age 使用 seconds，bytes 使用 bytes。
- counters/gauges 100% 记录；duration 初始 test sink 记录原始值。exporter bucket 在 W34 baseline 后单独版本化，不能反向伪造基线。
- label key 与 value 必须来自冻结低基数枚举。未知动态字符串归为 `other`，不可直接透传。
- operation/request/user/device/credential/group/conversation/message/transfer/session/link ID，IP/SocketAddr/域名/文件路径，account/nickname，token/cookie，证书/SPKI/密钥/hash，plaintext、ciphertext、payload size fingerprint 均禁止进入 labels。
- `operation_id` 仅可出现在受控脱敏 trace/log correlation，不进入 metrics。不得用 hash 后的用户/设备 ID 规避禁止项。
- 安全拒绝与内部错误完整计数；采样不能漏掉 rare failure。部署 retention、export endpoint 与访问策略由 release manifest/运维配置声明。

## 2. 冻结低基数 vocabulary

| label | 允许值 |
|---|---|
| `component` | `core`,`client`,`server`,`frb`,`flutter` |
| `result` | `success`,`failure`,`cancelled`,`deadline` |
| `error_category` | public error category 集合加 `none` |
| `direction` | `inbound`,`outbound` |
| `transport` | `quic_stream`,`quic_datagram`,`local` |
| `auth_profile` | `none`,`server_v1_token`,`server_v2_mtls`,`p2p_v2`,`p2p_v3_trusted` |
| `ip_family` | `ipv4`,`ipv6`,`unknown` |
| `delivery_state` | `queued`,`routed`,`sent`,`accepted`,`failed`,`expired` |
| `content_class` | `application`,`receipt`,`direct_bootstrap`,`direct_message`,`attachment_chunk`,`other` |
| `store` | `core`,`local_v2`,`server_v1`,`server_v2`,`serve_internal` |
| `db_action` | `open`,`migrate`,`transaction`,`backup`,`restore`,`quarantine`,`repair` |
| `cancel_outcome` | `accepted`,`already_requested`,`already_terminal`,`unknown_operation` |

`operation`/`rpc_op` 只能来自源码登记的枚举表；新增值需 privacy/cardinality review，不能使用任意 method/path 字符串。

## 3. 指标清单 v1

### API/operation

| metric | type/unit | labels | 边界 |
|---|---|---|---|
| `modcpt_sdk_operations_total` | counter/operations | component,operation,result,error_category | 每个公开长/业务 operation 恰一次 terminal |
| `modcpt_sdk_operation_duration_ms` | histogram/ms | component,operation,result | 从 admission 到 cleanup 完成 |
| `modcpt_sdk_operations_active` | gauge/operations | component,operation | running+cancelling；不得按 caller 标识拆分 |
| `modcpt_operation_cancel_requests_total` | counter/requests | component,operation,cancel_outcome | cancel control result，不等于 terminal result |

### verified session/transport

| metric | type/unit | labels | 边界 |
|---|---|---|---|
| `modcpt_p2p_connect_attempts_total` | counter/attempts | result,error_category,auth_profile,ip_family | 每次 physical dial attempt |
| `modcpt_p2p_connect_duration_ms` | histogram/ms | result,auth_profile,ip_family | dial 至 trusted/failed terminal |
| `modcpt_p2p_sessions_active` | gauge/sessions | auth_profile,ip_family | active logical sessions；不含 peer label |
| `modcpt_p2p_disconnects_total` | counter/disconnects | auth_profile,reason_class | reason_class 为 frozen enum：local_close,remote_close,transport,auth_revoked,expired,protocol,shutdown,other |
| `modcpt_p2p_auth_rejections_total` | counter/rejections | auth_profile,reason_class | 不记录证书、principal 或 remote address |

### delivery/receipt

| metric | type/unit | labels | 边界 |
|---|---|---|---|
| `modcpt_delivery_outbox_entries` | gauge/rows | delivery_state,content_class | DB authoritative snapshot |
| `modcpt_delivery_outbox_oldest_age_seconds` | gauge/s | delivery_state,content_class | 无行时不发 sample，而非 0 |
| `modcpt_delivery_attempts_total` | counter/attempts | result,error_category,content_class | 只计真正消耗 attempt 的 exact recipient send |
| `modcpt_delivery_receipts_total` | counter/receipts | result,error_category | accepted/rejected/conflict 由 result+category 表达 |
| `modcpt_delivery_receipt_latency_ms` | histogram/ms | result,content_class | sender Sent 到 valid receipt terminal；无 user/message label |

### transfer/attachment

| metric | type/unit | labels | 边界 |
|---|---|---|---|
| `modcpt_transfers_active` | gauge/transfers | direction,content_class | tracker authoritative active count |
| `modcpt_transfer_bytes_total` | counter/bytes | direction,content_class,result | 实际 accepted/sent bytes，不使用声明 size |
| `modcpt_transfer_rejections_total` | counter/rejections | direction,error_category,reason_class | reason_class 为 limits,digest,metadata,replay,expired,other |
| `modcpt_transfer_cancellations_total` | counter/transfers | direction,cancel_outcome | 与 operation terminal 分开 |
| `modcpt_transfer_ttl_purges_total` | counter/transfers | direction,content_class | idle TTL purge 数 |

### SQLite/data lifecycle

| metric | type/unit | labels | 边界 |
|---|---|---|---|
| `modcpt_db_actions_total` | counter/actions | store,db_action,result,error_category | 每个 open/migration/transaction/backup/restore terminal |
| `modcpt_db_action_duration_ms` | histogram/ms | store,db_action,result | 包含 rollback/cleanup 时间 |
| `modcpt_db_migrations_total` | counter/migrations | store,from_schema,to_schema,result | schema labels 仅为发布登记的小整数文本 |
| `modcpt_db_contention_total` | counter/events | store,reason_class | busy,lock_timeout,permit,other |
| `modcpt_backup_entries_total` | counter/entries | result,data_class | data_class 只用冻结 allowlist 类别，不记录路径 |
| `modcpt_backup_bytes_total` | counter/bytes | result,data_class | 实际 verified bytes |

### server RPC

| metric | type/unit | labels | 边界 |
|---|---|---|---|
| `modcpt_server_rpc_requests_total` | counter/requests | rpc_version,rpc_op,result,error_category,auth_profile | rpc_op 为冻结 op enum；未知归 other |
| `modcpt_server_rpc_duration_ms` | histogram/ms | rpc_version,rpc_op,result | request read 完成至 response terminal；transport handshake另计 |
| `modcpt_server_connections_active` | gauge/connections | rpc_version,auth_profile,ip_family | 不按 remote/address 拆分 |
| `modcpt_server_streams_active` | gauge/streams | rpc_version | 有界并发 stream handlers |

## 4. SLI 与环境边界

- SDK SLI：operation terminal result、duration、cancel completion；按 client/server 组件分开。
- server SLI：受理后的 RPC success/error/duration；认证失败不混入 availability success，单独报告安全拒绝。
- direct-connect SLI 必须限定“地址已提供且测试环境保证端到端 UDP 可达”。不能把公网 NAT 不可达归因于未实现的 traversal SLO，也不能因此触发 relay fallback。
- delivery SLI 分 exact recipient；logical message partial success 不压成单一 success。
- DB SLI 区分新建、正常 open、migration、restore/repair，不用快路径掩盖 migration failure。

本合同不提供 availability、latency、capacity 或 error-budget 数值。只有固定 commit/seed/tool/config/budget 的 baseline+soak 可重复后，产品/运维 owner 才能另行冻结 SLO。

## 5. Baseline 方法

每次 baseline 保存：精确 commit、dirty-state、OS/arch、CPU/RAM/disk、Rust/Flutter/FRB/SQLCipher/Cap'n Proto 版本、release/debug profile、seed、并发/数据规模、网络 fault profile、运行时长、warm-up、metric schema version、原始结果 digest。client、server、受控直连环境分别运行；至少重复多轮并报告分布，不只报最佳值。W34 PoC 与 W35 soak 未完成前不得写 SLO 数字或“容量已验证”。

## 6. Privacy conformance

W34 test sink 必须拒绝未知 label、超 vocab 值、ID/address/token/key/path/content 模式与无界 op 名；以 canary secrets 注入验证不出现在 metrics export。日志与 metrics 分别审计，禁用 exporter 时不得影响业务正确性。任何 exporter、dashboard 或 retention 的选择均是后续实现/运维决策。
