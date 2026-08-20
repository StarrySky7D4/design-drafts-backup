# 执行路线图（M0-M7）

> **唯一可执行路线**：本文定义 M0-M7 的依赖、阶段门和退出条件。`agents_work/BOARD.md` 是任务领取与集成状态的唯一索引；任务卡只记录各自交接证据。设计、审计、旧方案和日期日志均不单独授权实现。
>
> **2026-08-09 soak 结果**：当前工作树中的网络容错实现已在 Windows `192.168.5.16` 与 Linux VM `192.168.5.15` 上完成真实 non-loopback 86400 秒/85445-round soak；五类 fault 全过，fetched/acked `85445/85445`，p99 `370 ms`，1 次 `2066 ms` 有界恢复，170891 条连接全部闭合，client released，server stopped/active=0。聚合报告位于 `.m7-evidence/two-host-soak.json`。M0-M6 gates 已满足；M7-01 仍只选择 Mailbox-first，Room/媒体、relay 与 push 留作未来独立里程碑。`996fde3` 及后续 AutoSave CI/readiness 曾因 GitHub 账户付款失败或 spending limit 在零步骤状态失败。生产 SLO、真实 signing/attestation、生产运维与 RC/canary/rollback 仍开放，因此 M7 Gate 保持 OPEN。

## 1. 不变边界

- Rust 是业务状态事实源；Flutter 只读取快照和接收失效通知，FRB 生成绑定不得手改。
- 生产主体链是 `RealmId -> UserId -> DeviceId -> InstanceId -> TransportSessionId -> LinkId`。`TransportSessionId`/`LinkId` 只表示临时传输，不能成为联系人、业务对象或授权主键。
- `PeerPrincipal` 只能由 mTLS、credential evidence 和新 ALPN 的 `DeviceHello` 产生。`IdentityFrame`、自报 `user_id`、裸公钥或 session fallback 不能创建授权主体，也没有兼容降级路径。
- `GroupId` 是单一 genesis 绑定的全局唯一 UUIDv7；owner/治理是可变状态，绝不是 ID 组成部分。联系人按 `UserId` 聚合，授权、撤销、投递和审计落到 `DeviceId`/`CredentialId`。
- core 只支持直接 QUIC。它不实现 STUN、TURN、CID relay、DHT relay、NAT 探测或自动 fallback；外部 Mailbox/未来 relay 的依赖方向只能是 `mailbox/relay -> core`。
- 未知版本、畸形输入、认证不足、跨 realm、重放、撤销状态不可确认或 future schema 必须在业务副作用前 fail-closed。
- dry-run、loopback 自检、历史绿色 SHA 或保留协议编号不能替代更新后 HEAD 的 full/deep/release 证据。

## 2. 唯一路线和依赖

```text
M0 可靠性与版本止血
 ├─> M1 计划与协议登记收敛
 └─> M2 安全规范冻结
          M1 + M2
             └─> M3 可信身份与会话闭环
                    └─> M4 直接消息、投递与业务 ID
                           └─> M5 多设备、恢复与数据生命周期
                                  └─> M6 小群安全
                                         └─> M7 产品分流、SDK 与发布
```

M1/M2 可在 M0 汇合前起草，但只能在吸收 M0 的最终协议、迁移和验证结论后集成。M4-M7 在各自前置 Gate 通过前保持冻结；不得以临时兼容路径、Room、DHT/Gossip 或 core relay 绕过顺序。

| 阶段 | 唯一目标 | 前置 | 当前交付与退出状态 |
|---|---|---|---|
| M0 | 可靠性、版本和验证止血 | 无 | FRB typed-ID、BIND major、SQLite v5 旧数据策略和根验证入口已合并；Gate closed。 |
| M1 | 计划和协议登记收敛 | M0 结果可审计 | `ROADMAP.md` 为唯一路线；协议/schema/DB/error registry 已建立；Gate closed。 |
| M2 | 安全规范冻结 | M0/M1 结果可审计 | 威胁模型、协议生命周期、一对一文本与附件安全结论冻结；Gate closed。 |
| M3 | 可信身份和会话纵向闭环 | M0-M2 gates | `modcpt_pki`、realm/profile、server v2、client local-v2、DeviceHello、trusted `/3`、cutover/recovery 已闭合；Gate closed。 |
| M4 | 直接消息、投递和业务 ID | M3 gate | immutable Envelope、delivery/receipt、bounded transfer、direct-text E2EE、attachment descriptor/file-key/chunk runtime 已实现并验证；Gate closed。 |
| M5 | 多设备、恢复和本地数据生命周期 | M4 gate | M5-01/02/03 在 `ae6459a14bb3290228577cf1549d781590241516` / run `30995212103` 全部通过；Gate closed。 |
| M6 | 小群安全 | M5 gate | W27-W32、19+14 定向矩阵、root full 11/11 与 SHA `36e2a0699f20e35e805b8979b2ddb81ff3edc82e` / run `31067283213` 全部通过；Gate closed。 |
| M7 | 产品分流、SDK 和发布 | M6 gate；部分 SDK/运维准备可自 M4 后推进 | Mailbox-first、engine/runtime/RPC/wire/QUIC/daemon/recovery、真实 non-loopback 24h soak、业务 operation、binding/matrix、三工件 dry-run、RustSec、license、API support 与最终证据合同已落地；21 个 domain 已注册。生产 SLO、真实可信发布、生产运维与回滚开放；Gate OPEN。 |

## 3. 阶段门

| Gate | 唯一通过条件 | 当前状态 |
|---|---|---|
| M0 | M0-01 至 M0-04 在集成分支完成定向和统一验证；BIND v2、SQLite v5、FRB Rust 与根验证入口可复现 | `closed` |
| M1 | `ROADMAP.md` 为唯一路线；历史设计无现行冲突；协议注册表登记当前实现值 | `closed` |
| M2 | 威胁模型、协议生命周期获评审；一对一文本与附件至少有冻结结论 | `closed` |
| M3 | 生产入口只接受可信会话；持久身份重启加载、显式 cutover 与双节点降级攻击矩阵通过 | `closed` |
| M4 | 投递、业务 ID、资源边界、重试/恢复和一对一内容安全支持矩阵通过 | `closed` |
| M5 | 多设备、撤销、恢复、同步与本地数据生命周期不恢复撤销权限或产生未界定副作用 | `closed` |
| M6 | 小群成员变更、内容保密、离线收敛、恶意事件、历史边界和 N±1 资源可重复验证 | `closed` |
| M7 | 产品边界、公共 API/support window、运行指标/SLO、容量、每工件 SBOM/签名/attestation、RC/canary/rollback 责任全部冻结并经真实发布验证 | `OPEN` |

## 4. 当前 M7 唯一执行路线

### 4.1 已选择与已实现

- W36：Mailbox-first 已接受；Room/媒体、relay、push 冻结。
- W37：独立 `rust/mailbox/` 已实现 SQLCipher opaque Envelope engine、bounded Tokio runtime/RPC、双向限长 framing、严格 mTLS/SPKI-bound DeviceHello QUIC、loopback OpenMetrics、daemon、fixed-seed load、cross-host host/client harness 与 recovery drill。首次长期运行约 9 小时 26 分后超时失败；补齐同进程有界恢复后，第二次真实 Windows/Linux non-loopback 运行从零完成 86400 秒/85445 rounds，五类 fault、恢复预算、连接闭合与服务端零 active-operation 均通过并生成最终报告。
- M7-02：stable typed error、UUIDv4 operation、cancel/deadline/exactly-one terminal、message/file/server operation wiring、startup binding/native handshake 与 first-stable compatibility matrix已实现。
- Release dry-run：Windows server ZIP、Mailbox ZIP、Flutter app MSIX 的 release build/package、SHA-256、artifact-scoped CycloneDX、provenance subject 与消费者负面验证已实现。
- Supply chain：根 MIT LICENSE、FRB/local crate SPDX、四个 locked Cargo graph 与 Dart dependency license gate 已实现；四 Rust lockfile RustSec policy/gate 的例外带 owner、理由与 expiry。
- API/release governance：first stable `1.0.0`、N/N-1、至少 90 天/一个 minor 的弃用窗口、exact generated/native pairing、21-domain readiness、31-asset final evidence 与四角色受保护 Environment 合同已机器化。

### 4.2 当前阻断

1. 基于已完成的两主机 24h baseline，由产品/运维 owner 接受的 SLI/SLO、retention、dashboard、alert 与 error budget。
2. legacy API 在 first stable 的实际 warning/迁移落地，以及更广 DB/connect/backup operation conformance；semver/support/deprecation policy 已冻结但尚无真实 GA/N-1 运行证据。
3. production secret provision、Windows service ACL、rolling key/cert rotation、在线 backup、RPO/RTO 与部署验证。
4. RustSec 上游公告消除；当前九个有界例外已于 2026-08-07 复审并在 2026-09-06 硬到期前必须升级或再次复审。
5. 真实 Authenticode/MSIX 时间戳、可信 provenance attestation、clean exact tag RC 与消费者信任验证。
6. canary、停止扩散、撤回、restore/forward-repair 与 binary/service rollback 演练和责任签字。
7. 首个稳定签名版发布后建立真实 N-1 artifact fixture；first-stable 机器可读矩阵不能替代不存在的旧工件。

## 5. 旧编号映射（仅检索与历史）

下表不创建第二条路线，也不授予实现权限。旧 ID/N/R/A/B/C 标签只用于检索已有审计、设计和提交；排期、依赖和 Gate 一律使用 M0-M7。

| 旧材料 | M0-M7 映射 | 状态解释 |
|---|---|---|
| ID-1 | M0 typed-ID 与 M3/M4 跨层迁移 | W0 只证明基础，不能替代可信会话。 |
| ID-2、ID-3、ID-4 | M3 | v2 数据根、设备、DeviceHello、presence 与会话目录按 M3 依赖实施。 |
| ID-5、ID-6 | M4 | 投递路由与业务 ID 轴属于 M4。 |
| ID-7 | M5 与 M6 | 设备治理归 M5；群组治理/内容安全归 M6。 |
| N0、N1 | M1 与 M0 | 规范收敛与可信变更基线已吸收进 Gate。 |
| N2、N3、N4 | M3、M4、M5 | 可信会话、消息投递、多设备/数据生命周期。 |
| N5、N6、N7 | M6、M7、M7 | 安全群组、产品分流、SDK/生产发布。 |
| N8、N9 | 外部产品边界 | Mailbox 已按 M7-01 在独立 crate/service 实现；relay/push/SFU 与可选去中心化仍冻结。 |
| R1、R2 | M3/M4/M6 历史输入 | `IdentityFrame` 公钥和未验证放行已否决，不能按 R 标签授权。 |
| R3 | M7 产品决策输入 | Room/媒体冻结，未来需新立项。 |
| R4 | 无当前实现映射 | DHT、Gossip、relay 与 NAT fallback 不在本路线。 |
| A/B/C 等旧阶段名 | 按问题重新归入 M 阶段 | 不是现行状态、Gate 或交付物名称。 |

## 6. 验证入口与证据解释

从仓库根运行统一验证：

```powershell
pwsh scripts/Validate-ModCptLib.ps1 -Mode quick
pwsh scripts/Validate-ModCptLib.ps1 -Mode full
```

脚本覆盖 Rust workspace、独立 `flutter/rust`、demo、debug-harness、Flutter、FRB/Cap'n Proto drift，以及当前注册的 Mailbox load/network/recovery、release dry-run、RustSec、binding compatibility 和 compatibility matrix domains。缺少 Cap'n Proto、Flutter、签名、fixture、seed 或其他必需工具必须显式失败，不得以 silent skip 代替通过。

- M6 GitHub 关闭证据：`36e2a069…` / run `31067283213`，四个 Job success。
- M7 本地证据：原 19-domain full `19/19`，`release-dry-run,app-release-dry-run,m7-gate-contract,licenses` 四域定向通过，approval/API 与 server/Mailbox/app 发布消费者正负矩阵通过；真实 non-loopback 24h soak 报告 SHA-256 为 `8f94fa9b1116d023fdcbcc94d8595434a5b47008d58d21b90980d882390efaa5`。`b3a06e2` readiness 已暴露并促成 Linux Dart URI 修复；`996fde3` 的 CI/readiness runner 因 GitHub billing/spending limit 未启动。以上仍不证明生产 SLO 或真实 release。
- 本次状态同步让统一最小只读 CI 监听 `AutoSave`，并移除过期的 W28 自改源码 workflow。配置变更本身不算成功，合并后实际 run conclusion 才是更新后 HEAD 的 GitHub 证据。

## 7. 明确非目标

- core 内置 STUN、TURN、CID relay、DHT relay、NAT 探测或失败自动 fallback。
- 以 `IdentityFrame`、自报公钥、旧 `user_id -> SessionId` fallback、裸 session 字符串或未验证联系人建立 v2 授权。
- `(group_id, owner_uid)` 复合 GroupId、owner-in-ID 或 owner 变化导致 ID 变化。
- 未经 ADR/registry 冻结的 Room 媒体、DHT/Gossip、大群传播、push、SFU 或 relay；Mailbox 只允许 `mailbox-first-v1` ADR 的独立 opaque Envelope 范围。
- 把 loopback 自检、dry-run SBOM/provenance、历史绿色 Gate 或机器可读 first-stable matrix描述为生产 SLO、可信签名发布或真实 N-1 artifact evidence。

未来 relay 必须是独立 crate、仓库或服务，依赖方向只能为 `relay -> core`，并在新产品决策后遵循 `Knowledge/designs/relay-module-contract.md`。

## 8. 维护规则

- 新工作先映射到一个 M 阶段，写明问题、依赖、边界、验收、回滚和 `TEST_MATRIX` 用例；无法映射时先建立决策项。
- 实现状态以已合并源码、schema、迁移和可重复测试为准；`BOARD.md` 只记录领取与集成状态，不能改变本路线或安全裁定。
- 新风险进入 `Knowledge/DEFECTS.md`；设计论证、旧方案和完整取证分别保留在 `Knowledge/designs/`、`Knowledge/reviews/` 与 `Knowledge/logs/`。
- M6 关闭事实只由其最终 SHA/Run 证明；M7 状态只在 `M7_PLAN.md`、`BOARD.md`、任务卡和实际验证结果之间同步，不借用早期 candidate 或历史 Gate。
- 只有第 3 节 M7 Gate 的全部条件在最终集成 SHA 可追溯通过后，才允许把 M7 从 `OPEN` 改为 `closed`。
