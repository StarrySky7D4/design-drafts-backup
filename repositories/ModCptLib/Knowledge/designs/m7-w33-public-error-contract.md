# M7 W33 typed public error 合同

> 状态：W33 冻结形状与语义，尚未实现。现有 Rust typed errors、server `{code,error}` 与 FRB/Dart 字符串错误仍需 W34 适配；本文不证明跨层 conformance 已通过。

## 1. DTO v1

所有稳定 SDK/FRB/RPC 公共失败最终映射到一个不可矛盾的 DTO：

```text
PublicErrorDtoV1 {
  version: u16 = 1,
  code: string,
  category: PublicErrorCategory,
  message: string,
  retryable: bool,
  retry_after_ms: u64?,
  operation_id: string?,
  details: map<string,string>?
}
```

- `code` 是错误 identity：ASCII lowercase snake_case，1..64 bytes；同一 code 不得改义或复用。
- `category` 是有限枚举：`invalid_argument`、`authentication`、`authorization`、`not_found`、`conflict`、`failed_precondition`、`resource_exhausted`、`unavailable`、`deadline`、`cancelled`、`unsupported`、`incompatible`、`data_loss`、`internal`。
- `message` 是非本地化、安全、无秘密的 fallback，最多 256 UTF-8 bytes；UI 以 `code` 本地化，不能解析 message。
- `retry_after_ms` 仅在 `retryable=true` 且服务明确给出退避时出现；上限 86,400,000 ms。没有该字段不表示立即重试。
- `operation_id` 是本次长操作的随机 UUIDv4 文本，仅用于本机/同一请求关联；不是主体、幂等键、授权凭据或 metrics label。
- `details` 最多 8 项、总编码不超过 1024 bytes；key 为 1..32 bytes snake_case，value 最多 256 bytes。未知 key 只可在非安全、可忽略时跳过。

内部 cause、SQL、文件路径、堆栈、地址、token、证书/密钥、user/device/group/message/transfer ID、plaintext/ciphertext 不得进入 `message` 或 `details`。内部 cause 只进按隐私规则脱敏的诊断日志。

## 2. 公共 code 基线

| code | category | 默认 retryable | 安全 message | 副作用合同 |
|---|---|---:|---|---|
| `invalid_argument` | invalid_argument | false | `The request is invalid.` | 业务副作用前拒绝 |
| `unauthenticated` | authentication | false | `Authentication is required.` | 不创建授权主体/会话 |
| `permission_denied` | authorization | false | `The operation is not permitted.` | 不泄漏目标是否存在 |
| `not_found` | not_found | false | `The requested resource was not found.` | 不创建替代对象 |
| `already_exists` | conflict | false | `The resource already exists.` | 保留已存在对象 |
| `conflict` | conflict | false | `The operation conflicts with current state.` | 原子拒绝或返回已提交终态 |
| `failed_precondition` | failed_precondition | false | `A required precondition was not met.` | 不做部分提交 |
| `resource_exhausted` | resource_exhausted | true only with bounded policy | `A resource limit was reached.` | 释放本次 reserve/permit |
| `unavailable` | unavailable | true only for idempotent/resumable operation | `The service is temporarily unavailable.` | 不暗示提交与否；需幂等/查询语义 |
| `deadline_exceeded` | deadline | false | `The operation deadline was exceeded.` | 进入统一 cleanup；不得后台继续提交 |
| `operation_cancelled` | cancelled | false | `The operation was cancelled.` | cancel 已成为唯一终态且 cleanup 完成后才返回 |
| `unsupported_platform` | unsupported | false | `This operation is not supported on this platform.` | 不回退到不安全实现 |
| `unsupported_feature` | unsupported | false | `This feature is not supported.` | 不隐式降级或分配保留协议值 |
| `incompatible_version` | incompatible | false | `The component version is incompatible.` | 在加载/解析/写入前 fail-closed |
| `data_corrupted` | data_loss | false | `Stored data could not be verified.` | 保留原数据供 quarantine/repair |
| `internal` | internal | false | `The operation failed.` | 不暴露内部 cause；是否可重试由新明确 code 表达 |

领域 code（例如 direct、attachment、directory、lifecycle、prekey）可以保留更具体 identity，但必须由单一表映射到上述 category/retry/safe message。现有 `invalid`、`unauthorized`、`corrupted` 等旧 code 不自动成为新公共 code；W34 adapter 必须显式映射且测试每一 variant。

## 3. retry 合同

- `retryable=true` 表示“按同一操作的幂等/恢复合同和退避可重试”，不表示 SDK 自动重试。
- authentication、authorization、invalid、unsupported、incompatible、data loss 与 cancel 永不自动重试。
- deadline 默认不重试；调用方只有在业务操作有稳定 idempotency key 或查询提交结果后才能创建新 operation。
- `unavailable` 或 `resource_exhausted` 若可能已提交，必须返回查询/幂等信息；否则 `retryable=false`。
- server 指示的 `retry_after_ms` 是最小等待，不得绕过全局 attempt/deadline/circuit policy。

## 4. cancellation/deadline 唯一语义

- 公共取消错误唯一为 `operation_cancelled/cancelled`，不是普通 `internal` 或字符串 `cancelled`。
- 公共 deadline 错误唯一为 `deadline_exceeded/deadline`。
- cancel request 的控制结果不是 operation error：`accepted`、`already_requested`、`already_terminal`、`unknown_operation`。重复 cancel 不创建第二终态。
- success、business error、cancelled、deadline 只能赢得一次 terminal compare-and-set；取消 DTO 只在 cleanup 完成后可见。

## 5. 跨边界编码

- Rust 使用非穷尽内部错误可以演进，但 public adapter 必须穷尽映射；未知内部 variant 映射 `internal` 并在测试中显式失败提醒 owner 登记。
- FRB/Dart 输出结构化 DTO/exception；不得继续依赖 `to_string()` 文本解析。
- RPC failure envelope 在新版本中携带 DTO v1；现有 `{ok:false,code,error}` 不原地改义，必须版本化或提供明确迁移期。
- JSON/Dart 的缺失 optional 字段按 `null`；未知 category/code 不得伪装成已知语义：category 未知为 `incompatible_version`，未知领域 code 可保留 code 但按已知 category 安全显示。

## 6. W34 conformance 必需项

每个公开 error variant 需固定向量并验证 Rust→FRB→Dart/RPC：code/category/message/retry/details 一致；无秘密；边界长度；unknown field/category；旧字符串 adapter；retry 零副作用；cancel/deadline exactly-one terminal。完成这些测试前，本文只能称合同，不能称实现。
