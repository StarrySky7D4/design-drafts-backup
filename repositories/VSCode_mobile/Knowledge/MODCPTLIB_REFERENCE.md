# ModCptLib 参考审计与采用结论

> 审计对象（只读）：`C:\Users\Administrator\Desktop\ModCptLib`
> 审计时主仓库 HEAD：`ea1c6246cbbb69c37c4b448d0266930cc8680f80`
> 审计日期：2026-08-12

本项目参考 ModCptLib 的工程治理和边界设计，不复制其业务协议，也不把它当作 VS Code、容器或多端 IDE 的现成底座。`ModCptLib - back` 只作为历史比较对象；本方案的参考事实源是上面的主目录。

## 1. 值得采用的模式

### 1.1 Rust 事实源，Flutter 为薄投影

ModCptLib 将关键状态和业务动作集中在Rust，Flutter service/controller消费强类型snapshot和结构化invalidation event，而不是在多个Widget中各自维护一份事实；现有`NodeEvent`仍以字符串`kind`判别，并非完整typed/sequence/cursor stream。这一原则适合本项目，但部署位置不同：

- ModCptLib 的 Rust core 与 Flutter 可通过 flutter_rust_bridge 本地连接。
- 本项目所有业务与文件操作在服务端，Flutter 通过 TLS WebSocket/HTTPS 连接 Rust；客户端不得嵌入可写的本地 Rust 业务核心。
- Flutter reducer 可以做 optimistic projection，但服务端 DocumentActor/version/event 才是权威。

参考位置：

- `flutter/lib/services/app_state.dart`
- `flutter/lib/services/native_service.dart`
- `flutter/lib/services/frb_native_service.dart`
- `flutter/rust/src/api/events.rs`
- `flutter/rust/src/api/operations.rs`

### 1.2 Snapshot + invalidation/event

ModCptLib 的状态层采用 snapshot 与事件/失效信号组合，避免 Flutter 直接拼装底层数据。迁移到本项目后需要升级为服务端持久化语义：每个 stream 有 generation、sequence、ACK、保留窗口、snapshot fallback 和明确 reset；不能简单照搬本地事件流。

采用位置：workspace、document、terminal、agent、extension、operation、debug、testing、SCM、export、task、output 共12类同步流，以及 Flutter 的 reducer/store。

### 1.3 Binding/生成兼容门禁

ModCptLib把runtime binding契约和生成漂移分别纳入门禁：`flutter/rust/src/api/binding.rs`、`flutter/lib/services/frb_runtime.dart`及`flutter/test/frb_binding_compatibility_test.dart`验证API window/capability/profile/fingerprint/codegen version的fail-closed；`scripts/Validate-ModCptLib-core.ps1`与CI `drift-frb`域负责重生成漂移。本项目不使用FRB作为跨网络协议，但沿用相同纪律：

- `ide.proto` 是公共源，生成 Rust/Dart/TypeScript binding。
- clean 环境重新生成并比较，手工改生成物立即失败。
- 固定正向/拒绝二进制向量，三语言共同消费。
- Code-OSS actor/contribution inventory 也是生成物并检查漂移。

### 1.4 协议注册与生命周期

以下文档给出了可复用的治理框架：

- `Knowledge/PROTOCOL_REGISTRY.md`
- `Knowledge/PROTOCOL_LIFECYCLE.md`
- `Knowledge/THREAT_MODEL.md`
- `Knowledge/CONTENT_SECURITY.md`
- `Knowledge/DEFECTS.md`

本方案据此建立了独立的 capability、stable error、schema、Runtime Release 和 Code-OSS compatibility registry。只采用方法，不复用 ModCptLib 的消息编号、身份模型或 wire bytes。

### 1.5 Typed 长操作与真实取消

ModCptLib 的 operation 设计强调：调用者 Future/连接结束不等于真实取消，资源 owner 必须持续到 cleanup 完成，并且 terminal state 恰好一次。本项目把该模式用于：workspace start/stop、terminal/task/debug、Agent task、import/export、migration 和 image update。

当前实现参考位置：

- `rust/core/src/public_async_operation_v1.rs`
- `rust/core/src/operation_registry_v1.rs`
- `rust/core/src/public_operation_adapter_v1.rs`
- `rust/core/src/public_error_v1.rs`
- `flutter/rust/src/api/operations.rs`
- `Knowledge/designs/m7-w33-operation-cancel-cleanup.md`
- `Knowledge/designs/m7-w33-public-error-contract.md`

两份W33文档是冻结合同/历史设计依据；其早期“尚无统一实现”状态不能覆盖上述当前源码。

### 1.6 有界资源、统一 Gate 与证据绑定

ModCptLib 的 bounded transfer、quick/full validation、release evidence、兼容矩阵和供应链策略适合作为工程方法参考：

- `scripts/Validate-ModCptLib.ps1`
- `scripts/Validate-ModCptLib-core.ps1`
- `agents_work/TEST_MATRIX.md`
- `agents_work/templates/TASK.md`
- `agents_work/templates/HANDOFF.md`
- `supply-chain/compatibility-matrix-v1.json`
- `supply-chain/license-policy-v1.json`
- `supply-chain/m7-gate-policy-v1.json`
- `supply-chain/m7-gate-evidence.template.json`

本项目采用`gate-quick`/`gate-full`、每边界长度/队列/时间上限、最终clean commit证据、SBOM/license/signature/provenance和N/N-1 fixture要求。ModCptLib当前兼容矩阵仍是first-stable bootstrap（没有可复用的prior stable/N-1工件），且其M7 Gate仍为OPEN；可参考policy/evidence设计，不能将其描述为已完成的生产发布证明。

## 2. 不应复用的部分

| ModCptLib 内容 | 本项目结论 | 原因 |
|---|---|---|
| P2P/QUIC/mailbox，以及历史或保留的relay设计 | 不复用 | 本项目是客户端到受控服务端的IDE会话；ModCptLib当前没有relay实现，且拓扑、威胁与一致性完全不同 |
| PKI/MLS/消息身份协议 | 不复用 wire/ID | 本项目使用 OIDC、设备登记、workspace ACL 和服务端 opaque ID |
| flutter_rust_bridge 主边界 | 不复用为产品边界 | 所有插件/文件写入在服务端；公共边界必须是可版本化网络协议 |
| 本地文件/下载模型 | 不直接复用 | 本项目需要容器 VFS、DocumentActor、导出屏障和 artifact ACL |
| Web 平台与现有 UI | 不作为五端模板 | 审计提交仅签入`flutter/windows`与`flutter/web` runner目录，不能证明Android/iOS/macOS/Linux已具备 |
| 当前部署/运维脚本 | 不作为 Runtime | ModCptLib 没有本项目要求的 WSL2 distribution、workspace 容器、Code-OSS Runtime Lock |
| 多设备消息生命周期 | 只借鉴纪律 | IDE 文档需要 canonical version、lease、journal、snapshot/replay，而非消息收件箱语义 |

## 3. 需要重新实现的核心能力

ModCptLib 没有以下本项目必要能力，因此方案不得标记为“从参考项目继承”：

- Code-OSS Extension Host 启动、MainThread actor、contribution 与扩展认证。
- Headless Workbench Adapter 和 Flutter 原生 VS Code UI surface 映射。
- 原生代码编辑器、IME、UTF-16 位置、diagnostic/semantic token/diff。
- Linux/WSL2 workspace 容器生命周期、镜像/扩展锁和 Runtime Agent。
- DocumentActor、Mutation Broker、写租约、编辑 journal 和多流 cursor 恢复。
- PTY、SCM、LSP、DAP、Testing、Agent tool broker。
- 一致性工作区导出和 Windows 无工作区约束。

## 4. 采用映射

| 参考原则 | 本项目落点 | 验证方式 |
|---|---|---|
| Rust 是事实源 | Control Plane + Workspace Agent + DocumentActor | Flutter 重启/双端观察仍得到相同 canonical state |
| 薄 UI 边界 | Dart protocol/reducer，Widget 不直接写文件 | 架构 lint、UI/unit/E2E |
| snapshot + event | 12类 stream cursor/replay/reset | 断线、压缩、长离线 fault/property test |
| binding compatibility | Protobuf 三语言 + Code-OSS inventory | 生成零漂移、golden、Buf breaking |
| operation lifecycle | typed Operation host | cancel/commit/deadline race、exactly-once terminal |
| bounded resources | frame/edit/file/queue/log/terminal/export 配额 | fuzz、资源耗尽、soak |
| 单一 Gate | `just gate-quick`/`just gate-full` | clean commit evidence manifest |
| 任务/交接模板 | ID/owner/boundary/test/evidence | PR/里程碑审查 |
| 供应链证据 | Runtime Lock、SBOM、license、signature | 发行消费者验证负向工件 |

## 5. 参考使用规则

1. ModCptLib 保持只读；未经单独许可不修改、不复制 third-party 源码或生成物。
2. 引用时记录相对路径和审计 commit；未来更新参考仓库后重新核对结论。
3. 只复制抽象模式时重新命名领域概念，避免把 mailbox/realm/peer 等语义泄漏到 IDE 协议。
4. 任何直接代码复用先核查许可证、依赖、错误语义、线程/取消模型与测试，不因同为 Rust/Flutter 就默认兼容。
5. 本项目的权威文档、协议和 Gate 位于自身仓库；ModCptLib 只能提供背景证据，不能覆盖当前决策。

## 6. 审计结论

ModCptLib 最有价值的是其工程纪律：明确事实源、薄桥接、协议/风险注册、可恢复 operation、有界资源、生成兼容和发布证据。它不能降低 Code-OSS 兼容层、Flutter 编辑器、容器运行时或 IDE 多端一致性的核心研发风险。正确的使用方式是复用治理模式、重做领域实现，并首先通过 G0 门禁验证最难的假设。
