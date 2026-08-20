# ADR：M7 W33 release artifact、版本、支持与回滚

- 状态：Accepted for W33 contract；implementation pending
- 日期：2026-08-05
- 范围：M7-02 产品无关发布合同
- 非声明：没有真实 release workflow/tag、平台签名、可信 attestation、canary 或 rollback 演练已完成；2026-08-06 新增的 server artifact/SBOM/checksum/provenance dry-run 不改变这些阻塞

## 1. 决策

### 1.1 第一阶段 release units

| unit | planned target | 包格式 | consumer | 当前状态 |
|---|---|---|---|---|
| Windows desktop app | `x86_64-pc-windows-msvc`，Windows 11 x64 | MSIX；包含 Flutter assets、paired generated Dart 与 `modcpt_frb` native library | 终端用户 | MSIX dry-run/consumer/negative matrix implemented；真实签名与安装/回滚未执行 |
| account/lifecycle server | `x86_64-pc-windows-msvc`，Windows Server 2022 x64 | ZIP，含 `modcpt_server.exe`、config schema/示例和操作说明，不含私钥/realm bundle | 运维者 | planned；release build/package 尚未建立 |
| Rust crates | workspace source | 不在第一阶段发布到 crates.io | 仓库内 consumer | internal；`pub` 不等于外部稳定发布 |
| native SDK | `modcpt_frb` | 不单独分发 | 仅随 app | paired-only；无 mixed-version 支持 |

新增 OS/arch/package 是 additive release decision，但在该 target 的 build、测试、签名、SBOM、安装/卸载/回滚证据完成前标记 unsupported。Web 当前不承诺本地安全存储或 native app release。

### 1.2 Sidecars

每个实际发布 unit 必须有：SHA-256 checksum manifest、artifact-scoped CycloneDX SBOM、版本/commit/toolchain manifest、provenance/attestation bundle、消费者验证说明。Windows app 还必须有 Authenticode/MSIX 平台签名与可信时间戳；attestation 不替代平台签名。2026-08-06 已实现 server/Mailbox ZIP 与 app MSIX 的 unsigned dry-run、checksum/CycloneDX 1.5/provenance subject/消费者验证及负面矩阵；真实 release 仍缺 repository/FRB license、可信签名与 attestation。

### 1.3 Version source 与 identity

- release identity 来自 annotated Git tag `vMAJOR.MINOR.PATCH` 或 `vMAJOR.MINOR.PATCH-rc.N`，且 tag 必须指向 clean、已验证 commit。
- release manifest 是聚合权威，记录 app、server、各 Cargo package、FRB generator、Flutter/Rust toolchain、protocol/API/storage read/write window 与 artifact digest。
- `flutter/pubspec.yaml`、Cargo versions、Windows resource 与 artifact name 必须由后续 release tooling 从 release version 校验一致；W33 不声称当前分散版本已同步。
- artifact name：`modcpt-app-windows-x86_64-{version}.msix`、`modcpt-server-windows-x86_64-{version}.zip`；sidecar 以完整 artifact name 加 `.sha256`、`.cdx.json`、`.provenance.json`/签名 bundle 后缀。
- generated Dart、native library 和手写 FRB API 共享同一 build identity；启动握手固定 contract/API/capability/profile/fingerprint/FRB codegen 并在业务副作用前拒绝错配。当前 exact API window 仍只能同 bundle 配对，握手不授权 mixed-version。

### 1.4 Semver 与兼容

- breaking public API/DTO/error shape、同版本 wire/storage 的不兼容改义：major。
- backwards-compatible additive API/optional capability：minor。
- 不改 public contract 的兼容修复：patch；安全收紧使旧 consumer 无法安全解释时仍为 major。
- protocol、content profile、RPC 与 SQLite 保留各自版本；app semver 不替代它们。每个 release manifest 明确 read/write matrix。
- legacy API 正式 deprecate 后至少跨一个 minor 且不少于 90 days；必须提供替代入口和告警。删除只在 major，不允许 SessionId/IdentityFrame fallback 伪装兼容。

### 1.5 支持窗口

- stable channel 同时支持最新 stable `N` 与前一 stable `N-1`；`N-1` 支持至 N GA 后 90 days。RC/nightly 无生产支持承诺。
- “支持”只覆盖 release manifest 明示的 OS/arch、artifact pairing、protocol/API/storage matrix。超矩阵组合在加载、握手或 DB write 前 fail-closed。
- 保留 N 与 N-1 的已签名 artifacts、sidecars 与验证说明，至少到 N-1 支持期结束后 30 days；安全/法律撤回时保留 tombstone 与公告，不继续提供受影响二进制。
- server/client rolling rollout 只有在 manifest 明示 N/N-1 protocol/RPC 互操作且测试通过时允许；否则采用停机升级或并行 endpoint，不做静默降级。

### 1.6 Signing identity 与 custody

- Windows code-signing certificate 的法律主体、CA/provider、hardware-backed custody、rotation/revocation owner 当前未指定，是 release blocker。最终私钥只能在隔离的最小权限 release job 使用，不导出到仓库或普通 CI。
- provenance 采用受保护 release workflow 的 OIDC identity/短期签名；issuer/subject 与 verifier policy 当前未指定，也是 blocker。
- 普通 CI 保持只读，不因生成 SBOM/验证依赖而获得发布权限。release owner、security/key custodian、on-call、artifact verifier 是四个角色；具体人员须在 RC 前签字。

## 2. Rollout

1. 在 clean tag commit 构建、生成 sidecars并从消费者环境独立验证。
2. 对可恢复的 DB 执行声明的 pre-upgrade backup；验证 digest/restore 可读性。
3. server 先 canary，再按 manifest 允许的 client window 扩量；app 先内部/小比例 canary。
4. 观察 W33 metrics schema 对应信号；尚无审定 SLO 时只使用发布前签字的停止条件，不虚构 error budget。
5. 达到停止条件立即停止扩散，保留 evidence，选择 binary rollback 或 forward recovery。

## 3. Rollback 决策树

| 情况 | 允许动作 | 禁止动作 |
|---|---|---|
| binary 未写入更高 storage/protocol state | 回滚到 manifest 指定的最低兼容、已签名 N-1；验证 config 与 artifact digest | 任意旧 binary、未签名替换 |
| DB 已迁移但 N-1 声明可读写该 schema | 停服务、备份当前库、用 N-1 按 runbook 打开并验证 | 在线双写未知 schema |
| DB 已迁移且 N-1 不支持 | 保持新 DB 隔离；forward repair/quarantine，或恢复升级前已验证 backup 并处理期间数据 | SQLite downgrade、改写已发布 migration、旧 binary 回写 |
| wire/RPC 行为缺陷但 storage 未变 | 停止扩散、回滚 server/app 至兼容 pair，撤回问题 artifact | 自动降级 ALPN/认证 |
| signing/provenance 验证失败 | 隔离/撤回 artifact、轮换或撤销 signer、发布公告与新版本 | 以 checksum 代替平台签名/attestation |
| 数据完整性/权限恢复风险 | 停止服务与写入，保存 evidence，走 quarantine/forward fix/受控 restore | 自动“修复”、恢复已撤销权限 |

SQLite 始终只向前迁移。transaction rollback 不是 release rollback；identity cutover 的有限 rollback 也不等于 app/server binary rollback。

## 4. Responsibility contract

| 角色 | RC 前必须签字的责任 |
|---|---|
| release owner | version/tag、artifact matrix、canary/stop/撤回与公告 |
| security/key custodian | signing identity、custody、rotation/revocation、verifier policy |
| storage owner | migration read/write window、backup/restore/forward repair 与数据 reconciliation |
| service on-call | server rollout、监测、停止扩散、运行时恢复 |
| artifact verifier | 从消费者环境验证 checksum、SBOM schema/subject、平台签名、provenance |

具体人员、联系路径与演练时间当前未指定；缺任一签字则 RC gate 失败。

## 5. Consequences 与待实现 gate

- 优点：最小首发平台，FRB/native pairing 明确，SQLite 回滚不会诱导 downgrade，签名与 provenance 分离。
- 成本：W34/W35 需新增 release build、MSIX、版本同步校验、artifact-scoped SBOM、license/advisory policy、签名与消费者验证、N-version fixture、backup/repair/canary drill。
- 不存在或未验证的 SLO、签名身份、attestation、Flutter artifact 与演练必须显式失败，不能 silent skip。dry-run manifest 永久标记 `dryRun=true`，不能升级为 RC evidence。
