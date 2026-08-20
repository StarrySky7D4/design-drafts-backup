# VSCode_mobile Knowledge 索引

本目录是项目架构、协议、风险与实施路线的唯一权威文档入口。项目采用了 ModCptLib 中“事实源明确、协议登记、生成边界、统一 Gate、风险不丢失”的工程方法，但不复用其 P2P、FRB 或本地 Rust 核心实现。

## 权威入口

- [完整项目编码方案](PROJECT_CODING_PLAN.md)
- [当前目标架构](ARCHITECTURE.md)
- [代码组织结构](STRUCTURE.md)
- [Code-OSS Extension Host 兼容层](CODE_OSS_EXTENSION_HOST.md)
- [协议注册表](PROTOCOL_REGISTRY.md)
- [协议生命周期](PROTOCOL_LIFECYCLE.md)
- [公共Proto草案](../protocol/ide/v1/ide.proto)
- [能力与权限矩阵](../protocol/ide/v1/capability-matrix.yaml)
- [安全模型](SECURITY.md)
- [测试、CI与发布](TEST_STRATEGY.md)
- [执行路线图](ROADMAP.md)
- [活跃风险登记](DEFECTS.md)
- [ModCptLib 参考结论](MODCPTLIB_REFERENCE.md)

## 状态解释

- `frozen`：产品边界已由用户确认；更改需要显式决策。
- `planned`：设计已给出，尚未通过可执行验证。
- `implemented`：代码或生成物已完成并通过规定的局部门禁，但尚不等于集成提交验收。
- `accepted`：当前 clean 集成提交上的完整验收证据通过。
- `unsupported`：当前版本明确拒绝，不得通过隐式降级实现。

最初设计包中的业务模块均为`planned`；截至2026-08-18，M0-01 至 M0-09、G0-01 至 G0-07 均达到开发`accepted`。最终clean提交`d4b70b416ce28d486cb2af81eb53d5c13663b7db`在本机Windows和Local-Linux2通过完整quick；同一提交在Local-Linux2通过G0/G0-runtime、audited pnpm store（10,452 entries/307 packages/10,193 CAS）、pure Node reference、fixture、Rust UDS及前后零浏览器进程树，远端tracked dirty、残留进程和临时目录均为0。该开发验收允许开始M1，但不表示产品可运行或可发布，现有Headless Adapter/Extension Host fixture仍不advertise产品capability。

项目交付定位由[ADR-0001](ADRs/0001-source-only-distribution.md)冻结为作者自用、最多公开源码。开发`accepted`与制品`release_eligible`分离：M8发布体系当前`inactive`，evidence继续固定`gate_results=[]`、`release_eligible=false`，full SBOM、许可证审查、签名、provenance、consumer verification、digest-pinned发布镜像、平台安装/商店、N/N-1与canary/soak均未完成，但不再阻塞M0/G0验收或M1进入。source-publication卫生门仍为`planned`/non-green：secret/用户数据/机器路径、LICENSE/NOTICE/上游来源、lock/generated drift、cache/构建工件和零Electron/Chromium检查尚未形成完整可执行准入，因此当前不能宣称源码公开候选就绪。验收不依赖托管CI；GitHub Actions只可作为可选镜像。

用户确认的产品边界（全生命周期零Electron/Chromium、无WebView、Rust服务端、服务端写入/扩展、纯Node Code-OSS Extension Host、WSL2 Linux容器、无插件管理器、无Windows工作区、Flutter五端、多端状态同步）为`frozen`。M1+仍须逐项实现和验收，不得把M0/G0开发accepted描述为production-ready。

## 维护规则

1. 外部协议、持久化 schema、稳定错误、Code-OSS API映射或 capability 变化，先更新注册表。
2. 架构边界变化更新 `ARCHITECTURE.md`；代码目录变化更新 `STRUCTURE.md`。
3. 新风险进入 `DEFECTS.md`，关闭时保留证据和关闭条件。
4. 阶段状态只由当前集成提交上的可重复 Gate 改变。
5. 设计推导可存入未来的 `Knowledge/designs/`，但不能覆盖本目录的权威结论。
