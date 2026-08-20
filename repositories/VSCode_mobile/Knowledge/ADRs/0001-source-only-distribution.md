# ADR-0001: 作者自用与仅源码公开交付定位

- 状态：`accepted`
- 日期：2026-08-18
- Owner：architecture / security / release
- 影响的风险/协议/schema/Runtime Release：R-019、R-020、R-022、R-027、R-041；不修改公共协议、数据库schema或Runtime Release

## 背景与约束

本项目仅供作者自用，最多公开源码。当前不发布或分发二进制、安装包、容器镜像、VSIX集合或WSL2 rootfs，也不运营公共托管服务，不承诺稳定版本、兼容性窗口或生产支持。既有治理把开发阶段验收与制品发布资格绑定，导致已由同一clean commit证明的M0/G0开发基线仍被M8发布材料阻塞。

零Electron/Chromium/WebView、Code-OSS exact lock、协议默认拒绝、服务端事实源、资源上限、凭据与路径隔离、数据库迁移恢复和容器最小权限仍是开发硬边界。若产品开始监听网络，认证、会话、ACL与资源归属也仍是前置，单用户不等于允许裸公网服务。

## 候选方案

### A. 继续把开发验收绑定完整制品发布资格

- 优点：单一门禁模型。
- 缺点/风险：要求不存在的分发物通过签名、商店、consumer verification与canary，阻塞作者自用开发。
- 验证成本：每次阶段推进都必须构造并不准备发布的制品。
- 退出/迁移方式：改用双层验收模型。

### B. 分离开发验收、源码公开卫生与制品发布资格

- 优点：开发阶段按真实代码/安全证据推进；发布材料只在确有分发意图时启用。
- 缺点/风险：必须防止把开发accepted误写成production-ready或可发布。
- 验证成本：维护机器可读双层状态、路径effect-surface和source-publication门。
- 退出/迁移方式：由新的显式ADR重新激活M8。

## 决策

采用方案B。`development acceptance`用于里程碑推进；`artifact release eligibility`属于休眠的M8，当前固定`inactive`且`release_eligible=false`。`gate-full`、full SBOM、完整license review、signature、provenance、consumer verification、digest-pinned发布镜像、五端安装/商店、N/N-1、canary/soak不再是M0/G0进入M1的前置。

公开源码仍执行source-publication卫生门：secret/用户数据/机器绝对路径、LICENSE/NOTICE/上游来源、lock/generated drift、cache/大型工具链/构建工件和零Electron/Chromium检查。该门当前planned/non-green，现存历史机器路径尚未完成可执行脱敏/allowlist，因此当前树不得宣称可公开。文档/任务的data-only变化可在机器路径分类与治理门通过后继承同一实现树证据；代码、lock、生成器、协议、运行时或安全门变化必须重跑对应门。

基于最终clean提交`d4b70b416ce28d486cb2af81eb53d5c13663b7db`在本机Windows完整quick、Local-Linux2完整quick、G0/G0-runtime及前后零浏览器进程树均通过，M0-01..M0-09与G0-01..G0-07达到开发`accepted`，G0为technical Go，允许开始M1。这不表示产品可运行、可发布或production-ready。

## 后果

- 正向影响：M1可以在不伪造发布物的前提下开始；开发验收与源码卫生仍有可执行门。
- 负向影响：若未来分发制品，需要重新承担完整M8供应链、兼容、平台和运维成本。
- 安全/隐私：安全硬边界不降级；网络监听前仍须完成认证、会话、ACL和资源归属。
- 兼容/数据迁移：当前不承诺N/N-1；已登记协议与schema仍禁止静默重解释，迁移/恢复规则不变。
- 运维/回滚：当前没有发布物、公共服务或外部消费者需要迁移。

## 验证与复审触发器

- 自动化证据：治理合同、路径effect-surface mutation、repository baseline、零浏览器正向/拒绝门与Windows quick。
- 复审日期/条件：用户决定向他人分发任何二进制、安装包、镜像、VSIX集合或WSL2 rootfs，或运营公共服务时立即复审。
- 强制重审：引入制品下载、签名、商店、公共SLO、兼容承诺、付费/公共用户或第三方托管分发。

## 回滚

恢复制品发布只能新建显式ADR，将`artifact_release.status`改为`active`，恢复M8工作包与`gate-full`发布前置，补齐SBOM/license/signature/provenance/consumer verification、平台安装、兼容与canary/soak证据。不得通过直接把`release_eligible`改为true或复用开发accepted状态恢复发布。
