# M7 W33 operation、取消、deadline 与 cleanup 合同

> 状态：W33 冻结合同，尚无统一实现。现有 transfer cancel、QUIC close 和局部 timeout 是实现输入，不等于本合同已满足。

## 1. Operation identity 与状态机

- 只有可能跨 await、网络、磁盘、后台 task 或用户等待的操作需要公开 `operation_id`；短同步 snapshot/纯 parser 不创建 operation。
- ID 是进程内生成的随机 UUIDv4，不携带 user/device/message/time，不可用作授权、持久业务主键或 metrics label。
- 状态机：`running -> cancelling -> cancelled`，或 `running/cancelling -> succeeded|failed|deadline_exceeded`；terminal 不可变。
- cancel 幂等结果：首次接收为 `accepted`；正在取消为 `already_requested`；保留期内终态为 `already_terminal`；从未存在或终态记录过期为 `unknown_operation`。
- operation 终态记录必须有界；具体容量/保留时长由 release 配置声明。记录过期只影响 cancel 查询，不删除业务审计或 durable receipt。

## 2. race 决议

单一 terminal arbiter 使用 monotonic clock 与原子 compare-and-set：

1. 若业务 success/failure 已提交 terminal，后到 cancel/deadline 返回 already-terminal。
2. arbiter 判定时若 `now >= deadline_at` 且尚无 terminal，`deadline_exceeded` 优先于尚未提交的 cancel。
3. deadline 未到时，已被 arbiter 接受的 cancel 优先于后续业务提交。
4. future/drop、Dart subscription cancel 或 caller disconnect 不自动算成功取消；只有领域 cleanup 完成后才发布 cancelled terminal。

每个 operation 恰好发布一个 terminal result/event。late callback、receipt 或 waiter 必须识别 terminal generation 并丢弃，不得再次写入。

## 3. Operation/cleanup matrix

| 操作族 | 可取消点 | deadline | terminal 前必须释放/处理 | 取消后禁止 |
|---|---|---|---|---|
| P2P connect、TLS/DeviceHello/BIND | DNS/拨号、握手、BIND wait 各 await 点 | 必需，monotonic absolute | connect future、QUIC connection/stream、permit、registry candidate、watcher task | 可见 session、SID/address map、自动旧 ALPN/relay fallback |
| server RPC round trip | open stream、write、read wait | 必需；覆盖完整 attempt，不按步骤重置 | stream halves、connection-local permit、waiter；是否已提交必须按 op 幂等语义处理 | 盲重试可能已提交的 mutation |
| local delivery drain/claim/send | claim 前、每 recipient send 前后 | batch 与 per-send 均有界 | runner permit、未使用 claim；已发送项按 exact durable 状态 settle | 改 wire、重复 attempt/receipt、扩 recipient snapshot |
| attachment/file transfer | begin、chunk read/write、digest/commit 前 | idle TTL 与 explicit deadline 分离 | transfer slot、chunk budget、buffer、temp file、open handle、waiter | 保留 partial 为成功、重复 completion/receipt |
| backup/restore | scan、每 entry stage、verify、commit 前 | 必需；commit 使用独立 bounded phase | archive handle、staging dir、temp file、memory key、permit；失败保留原 DB | partial overwrite、restore identity/authority、越过 allowlist |
| DB open/migration/transaction | 开始前可取消；transaction 开始后不接受异步打断，只等待原子 commit/rollback | admission deadline；transaction 有内部 watchdog/资源门 | connection、lock/permit、transaction rollback、temp migration file、zeroized key | 半 migration、改写已发布 step、自动 downgrade |
| credential provision/rotate/revoke | 验证前可取消；原子事务提交后为 success/already-terminal | 必需 | challenge/permit/secret buffer；事务 rollback；transport waiter | 伪装撤销成功、恢复已撤销权限、旧 credential fallback |
| node/channel close、disconnect | close 本身幂等；不可被普通 operation cancel 阻止 | bounded shutdown deadline | 调用 `P2PChannel::close()`、stream/session/watcher/event subscription、active+standby links | 仅依赖 Drop、留下后台发送/事件 |
| event/log stream subscription | 等待/转发点 | shutdown deadline | sink/subscription、sender/receiver、producer task | stream 关闭后发事件、把 log 当 metrics |
| snapshot/parser/短状态 apply | 不公开 cancel；在调用内原子完成或失败 | 通常无公开 deadline | 临时 buffer/lock | 因引入 cancel 产生部分状态 |

## 4. 通用 cleanup 顺序

1. 原子关闭新 work admission，标记 cancelling。
2. 通知 child tasks；停止新网络 I/O/DB claim/chunk reserve。
3. 结束或回滚当前领域原子单元；不在未知提交状态下伪报 cancelled。
4. 释放 permit、预算、claim、waiter、stream/session、文件与 task handle；secret buffer zeroize。
5. 验证 durable 状态与资源计数，提交唯一 terminal。
6. 发布至多一次 terminal event，并保留有界 cancel-query tombstone。

panic/unwind、runtime shutdown、peer disconnect 和显式 cancel 必须汇入同一 cleanup primitive；只能改变 terminal category，不能绕过释放步骤。

## 5. Deadline 与 retry 组合

- deadline 是 absolute monotonic instant；跨 RPC 传输时只传剩余预算，不传可伪造本机 monotonic 值。
- retry attempt 共享原 operation 总 deadline；重连不得重置预算。
- `retry_after_ms` 大于剩余预算时直接返回原 retryable failure或 deadline，不启动 attempt。
- cancellation 永不消耗新的 delivery attempt；已经执行网络提交的 attempt 按领域 exact 状态计数。

## 6. W34 conformance matrix

每个操作族至少验证：cancel before start、每个 await 点、commit race、deadline race、duplicate cancel、unknown/expired ID、disconnect/close、panic injection、restart（适用时）、permit/bytes/temp/DB/session/task 归零、exactly-one terminal、无重复 receipt/event。测试必须有固定 seed 与资源快照；尚未建立这些门前不能宣称 cancellation 已实现。
