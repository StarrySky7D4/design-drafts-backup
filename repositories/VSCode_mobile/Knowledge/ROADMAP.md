# 执行路线图与工作包

> 路线图以技术门禁而非日期推进。开发`accepted`决定阶段推进；制品`release_eligible`是独立且当前休眠的M8维度。二者不得混用。

## 1. 推进原则

项目最大不确定性不是 Rust 控制面，而是“无 Workbench UI 的 Code-OSS Extension Host 是否能以可维护成本提供足够兼容性”和“Flutter 原生编辑器能否达到日常编程质量”。因此先验证不可逆风险，再扩展产品面。

阶段关系：

```text
G0 Extension Host 可行性 ─┬─ M1 Runtime ─ M2 文档事实源 ─┬─ M3 Flutter 编辑器 ─┐
M0 仓库/协议/安全 ─────────┘                              └─ M4 扩展 ─ M5 工作流 ┤
                                                               └─ M6 Agent ───┤
                                                   M5 + M6 ────── M7 恢复 ─────┤
                                                                              └─ M8 发布
```

G0 是 Go/No-Go 门，M0可与其并行建立正式工程基线；最终clean提交`d4b70b4`已使G0 Go与M0开发验收同时成立，M1可以开始。M2 的 canonical edit path 是 M3、M4和M6 Rust Agent Core的共同前置；M6的C3 Code-OSS Bridge另依赖M4。M7基础sync可提前开发，但阶段accepted必须等待M5和M6全部producer/快照闭环。M8制品发布体系由ADR-0001休眠，不参与当前开发阶段依赖图。

## 2. 建议团队切分

建议核心团队 7–10 人；不是硬性编制：

| 工作流 | 建议职责 |
|---|---|
| Architecture/Protocol | 架构 owner、版本/错误/schema 注册、跨语言合同 |
| Code-OSS/TypeScript | Adapter、MainThread actor、扩展 fixture 与升级 |
| Rust Control Plane | Gateway、auth、store、sync、operation、export |
| Runtime/Platform | Linux 容器、WSL2 distribution、镜像、网络、观测 |
| Flutter Editor | 编辑器 model/render/input、应用状态与五端平台 |
| IDE Services | terminal、SCM、LSP、DAP、testing 映射 |
| Agent/Security | tool broker、approval、secret、威胁测试 |
| Quality/Release | E2E、fault、性能、兼容矩阵、供应链与交付 |

每个工作包只有一个 accountable owner；跨模块改动仍由协议 owner 先冻结边界。人员不足时优先缩减 C1-C3 功能，不删除 G0、文档一致性、安全或恢复门禁。

## 3. G0：Headless Extension Host 可行性门

### 目标

用固定 Code-OSS commit 和 Node 版本证明：不启动 Electron、浏览器或 Workbench DOM，也能由自维护 Adapter 驱动真实 Extension Host，并将核心 VS Code API 映射到结构化服务端协议。

### 工作包

- `G0-01` 选择并记录 Code-OSS commit、Node exact version/modules ABI、构建工具与许可证。状态：`accepted`（开发验收；Code-OSS `1.133.0`/commit `a5b500...`与Runtime Node `24.18.0`/ABI `137` exact lock已在最终clean自管Linux重放）。
- `G0-02` 从该 commit 提取 Extension Host 启动链和 ExtHost/MainThread RPC actor inventory，生成 hash。状态：`accepted`（87 MainContext/81 ExtHostContext actor、8段启动链与跨平台canonical inventory已在最终clean自管Linux重放）。
- `G0-03` 建立最小 Headless Workbench Adapter，拒绝所有未实现 actor。状态：`accepted`（87 actor闭合分类、843-file无浏览器runtime closure与typed refusal通过；不是产品Runtime）。
- `G0-04` 编写 fixture VSIX并覆盖核心、语言与原生UI映射。状态：`accepted`（exact Extension Host fixture与生命周期合同通过；不构成VSIX分发许可）。
- `G0-05` 通过 UDS 将 UI/编辑请求转给临时 Rust harness并回送响应。状态：`accepted`（最终clean Local-Linux2为24 frames/22 observations）。
- `G0-06` 在同 commit 纯 Node reference harness 与真实UDS Headless Adapter上运行同一fixture并执行零浏览器门。状态：`accepted`（22条reference observation、真实UDS及前后进程树一致）。
- `G0-07` 输出 API 分类、patch面、升级成本与No-Go结论。状态：`accepted`、技术结论`Go`（最终clean提交`d4b70b4`在Local-Linux2通过完整G0/G0-runtime；store为10,452 entries/307 packages/10,193 CAS；临时根、相关进程和tracked dirty均为0）。制品发布仍inactive/No-Go，不影响开发验收。

G0 当前技术结论为`accepted`/`Go`：上述No-Go条件在exact commit `a5b500...`的已测范围内均未触发。该结论只解除Extension Host架构可行性疑问；M1仍需逐切片实现，不能把G0 harness当产品Runtime。

### 验收

- 无 GUI/DOM 进程，真实 Extension Host 可重复启动、激活/停用 fixture extension。
- 文档 open/change/save、command、completion、diagnostic、WorkspaceEdit 完整往返。
- QuickPick、TreeView、Output、PTY 至少各一条端到端映射。
- actor inventory 可生成并在源变更时触发失败；未知 actor 明确拒绝。
- 连续启动/退出 100 次无孤儿进程、socket 或未完成 Operation。
- 报告列出最小 Code-OSS patch，证明不需要在 Rust 中重写私有 RPC。

### No-Go 条件

核心文档/命令/language API 无法在无 Workbench 状态下构造；最小源码闭包、构建或测试强制要求 Electron/Chromium；需要长期 fork 大片 Extension Host 私有协议；actor 无法稳定盘点；或 fixture 与同 commit 纯 Node reference/golden 的差异无法被有界 Adapter 修复。出现任一条件时暂停产品主线，重新评估不含浏览器运行时的受限扩展 API 或放弃 VS Code 插件兼容；不得改用另一种完整浏览器框架绕过。

## 4. M0：仓库、契约与工程基线

### 工作包

- `M0-01` 按 `STRUCTURE.md` 建 monorepo、Rust workspace、Flutter app、Adapter npm workspace、`justfile`。状态：`accepted`（开发验收；最终clean提交`d4b70b4`的Windows与Local-Linux2相关门通过；不表示产品capability已advertise或制品可发布）。
- `M0-02` 固定 Rust/Flutter/Dart/Node/Protobuf/Buf 版本，建立 devcontainer/CI image。状态：`accepted`（开发验收；最终clean提交`d4b70b4`的Windows与Local-Linux2相关门通过；不表示产品capability已advertise或制品可发布）。
- `M0-03` 冻结公共 `ide.v1`、capability-matrix 与三类 internal proto，生成 Rust/Dart/TypeScript 绑定及穷尽validator。状态：`accepted`（开发验收；最终clean提交`d4b70b4`的Windows与Local-Linux2相关门通过；不表示产品capability已advertise或制品可发布）。
- `M0-04` 实现 stable ID、PublicError、deadline、idempotency、Operation 基础库。状态：`accepted`（开发验收；最终clean提交`d4b70b4`的Windows与Local-Linux2相关门通过；不表示产品capability已advertise或制品可发布）。
- `M0-05` 建 SQLite migration harness、fixture 与每 store 独立 schema version。状态：`accepted`（开发验收；最终clean提交`d4b70b4`的Windows与Local-Linux2相关门通过；不表示产品capability已advertise或制品可发布）。
- `M0-06` 建 tracing、metrics、audit schema、redaction 与本地诊断 bundle。状态：`accepted`（开发验收；最终clean提交`d4b70b4`的Windows与Local-Linux2相关门通过；不表示产品capability已advertise或制品可发布）。
- `M0-07` 建 `gate-quick`、`gate-full`、生成零漂移、SBOM/license/provenance 骨架。状态：`accepted`（开发验收；最终clean提交`d4b70b4`的Windows与Local-Linux2相关门通过；不表示产品capability已advertise或制品可发布）。
- `M0-08` 建威胁模型、ADR、风险/协议注册流程和 CODEOWNERS。状态：`accepted`（开发验收；最终clean提交`d4b70b4`的Windows与Local-Linux2相关门通过；不表示产品capability已advertise或制品可发布）。
- `M0-09` Proto顶层/嵌套oneof与action、request→response/outcome/result、StreamKey→domain capability、permission/effect guard、snapshot/event identity、UI subtype和transfer状态机矩阵零漂移Gate；未映射或未协商case负向测试。状态：`accepted`（开发验收；最终clean提交`d4b70b4`的Windows与Local-Linux2相关门通过；不表示产品capability已advertise或制品可发布）。

### 验收

- 空白环境一条命令完成生成、构建和 quick gate；无未固定下载。
- 同一协议 fixture 被三语言解析；未知 major/capability 在副作用前拒绝。
- 最小 Rust service、TS adapter harness、Flutter reducer 可交换 Hello/Error/Heartbeat。
- clean commit 生成可追溯 evidence manifest；手改生成文件导致 Gate 失败。

## 5. M1：统一 Runtime 与工作区生命周期

### 工作包

- `M1-01` Rust Control Plane modular monolith：auth、ACL、workspace metadata、scheduler、operation。状态：`active`；M1-01A/B/C/D/E/F/G/H/I/J/K/L/M/N/O/P/Q/R/S/T/U/V/W/X/Y/Z/AA/AB/AC/AD/AE/AF/AG/AH/AI/AJ/AK/AL/AM/AN/AO/AP/AQ/AR/AS/AT/AU/AV/AW/AX/AY/AZ/BA五十三个有界切片已`implemented`。A-I闭合准入、CAS、active journal、device tombstone与effect/claim/handoff/Applied事实；J把exact Runtime spec digest绑定到wire与durable事实；K-W逐步闭合ticket/device challenge、OIDC/PKCE、OS CSPRNG、HTTPS provider、callback、TLS/WebSocket及protobuf session/liveness；X-AI闭合route、启动材料、single-owner数据库、具体OIDC owner及first-run；AJ建立durable Ed25519 device public-key事实，AK绑定owner OIDC enrollment候选，AL以exact durable revision CAS登记设备，AM确保真实callback只在durable commit和maintenance/lease均健康后返回200，AN把该边界接入一次性真实provider/外置浏览器/有界TLS callback设备登记进程，AO以独立owner OIDC用途和exact access/durable revision CAS创建首个Stopped/Absent workspace，AP把session callback与one-time bootstrap置于callback前后exact durable/access/resource snapshot fence并闭合精确取消，AQ以active durable device的exact Ed25519签名授权未来login-start，AR以容量1的HMAC receipt owner闭合进程内start/result/cancel/ack重放，AS以schema v3独立32-byte启动密钥提供重启稳定HMAC root，AT增加key-bound receipt checkpoint，AU增加session-purpose OIDC transaction与ticket digest checkpoint，AV把三类checkpoint置入容量1、exact-revision、integrity-bound SQLite原子bundle，AW从AS root、exact receipt binding与完整ticket issue派生可重建bearer并用AU digest验证，无需保存明文secret，AX使AV bundle成为三个owner的联合投影/恢复入口，AY把该bundle接入`ide-server` coordinator并闭合Pending signed-rebind、callback commit-before-response、Ready重放、cancel及ticket消费后ACK，AZ把该durable owner接入真实TLS/WebSocket handoff并在101前闭合ticket消费与空bundle ACK，BA以官方生成standalone protobuf和exact HTTPS POST路由闭合start/result/cancel、result ACK-loss、SessionId challenge binding及cancel重启恢复。产品远程workspace/device enrollment wire/receipt、客户端private-key生成与secure storage/轮换、数据库handle到SQLite的跨平台原子绑定、部署账户/DACL、外部provider互操作证据、既有数据库长期listener/session loop、protobuf业务request/event/snapshot/result/resume、长驻scheduler、可信Runtime真实副作用与capability仍未实现，故M1-01整体未完成。
  - 当前BB更新：M1-01现有A..BA/BB五十四个`implemented`切片。BB已实现Flutter平台secure storage设备私钥、device-signed BA start、单pending receipt重启恢复、外置系统浏览器及fail-closed Ready handoff；它取代上行“客户端private-key生成与secure storage未实现”的旧时点描述，但密钥轮换/恢复、远程设备登记、WebSocket 101客户端、业务Frame/resume和长期listener仍未完成。
  - 当前BC更新：M1-01现有A..BC五十五个`implemented`切片。BC闭合Flutter exact WSS 101、唯一子协议和pending ACK/失败关闭边界；ClientHello/Heartbeat、UI接线、业务Frame/resume、长期listener和capability仍未完成。
  - 当前BD更新：M1-01现有A..BD五十六个`implemented`切片。BD闭合Flutter官方protobuf ClientHello/ServerHello零capability准入及caller-driven heartbeat状态；分配前有界的原始WSS收包、真实Flutter↔Rust协议E2E、业务Frame/resume、长期listener和capability仍未完成。
  - 当前BE更新：M1-01现有A..BE五十七个`implemented`切片。BE以原始`SecureSocket`闭合WSS Upgrade、分配前frame length验证、1 MiB/16 fragments/4096 shared frames及BD Hello/heartbeat接线；UI连接生命周期、后台heartbeat、真实Flutter↔Rust可信TLS E2E、业务Frame/resume、长期listener和capability仍未完成。
  - 当前BF更新：M1-01现有A..BF五十八个`implemented`切片。BF把Ready接入BE/BD并由Flutter Widget单独持有15秒heartbeat timer与inactive/dispose cleanup；真实Flutter↔Rust可信TLS E2E、业务Frame/workspace UI、resume、长期listener和capability仍未完成。
  - 当前BG更新：M1-01现有A..BG五十九个`implemented`切片。BG使`ide-server` caller-driven service容量1持有升级session、逐步驱动并在close/error后撤销恢复；产品`--serve`/取消监督、并发session、真实Flutter↔Rust E2E、业务Frame/workspace UI、resume与capability仍未完成。
  - 当前BH更新：M1-01现有A..BH六十个`implemented`切片。BH以固定派生session库和state-before-network preflight组合startup materials、provider、listener、durable coordinator与BG service；长期process supervision、真实端到端session及业务能力仍未完成。
  - 当前BI更新：M1-01现有A..BI六十一个`implemented`切片。BI以step/runtime/idle硬预算、逐步observer及每步前后cancel建立finite service epoch；OS signal、main长期serve、真实端到端session及业务能力仍未完成。
  - 当前BJ更新：M1-01现有A..BJ六十二个`implemented`切片。BJ把Windows console cancellation安全投影到单一finite `--serve` epoch；SCM、自动epoch轮换、真实Flutter端到端、业务Frame与capability仍未完成。
  - 当前BK更新：M1-01现有A..BK六十三个`implemented`切片。BK只在HTTPS连接预算精确耗尽且无active/upgraded连接时续期，并以observer step、总step/runtime和1,024 epochs四重边界约束Windows serve；SCM/process restart、真实Flutter端到端、业务Frame与capability仍未完成。
  - 当前BL更新：M1-01现有A..BL六十四个`implemented`切片。BL把authenticated session绑定到建立时exact durable revision/checkpoint，并在每次TLS read前以已释放的短lease拒绝外部winner漂移；业务request durable CAS、scheduler/Runtime、真实Flutter端到端与capability仍未完成。
  - 当前BM更新：M1-01现有A..BM六十五个`implemented`切片。BM首次把公共workspace start/stop(false)接入容量1 authenticated session、wire语义幂等准入及短lease SQLite CAS，durable commit后才发送唯一terminal accepted/error；scheduler/Runtime不消费operation，Flutter workspace UI、并发request/resume与capability仍未完成。
  - 当前BN更新：M1-01现有A..BN六十六个`implemented`切片。BN让既有durable scheduler按调用方逐步消费BM operation，闭合exact CAS的start/prepare、commit-before-I/O Apply、重启Query-only及按时success/超时terminal；生产scheduler epoch、可信Runtime resolver/连接工厂、取消cleanup、真实副作用与capability仍未完成。
  - 当前BO更新：M1-01现有A..BO六十七个`implemented`切片。BO在BN上增加最多1,024步、逐步取消且遇非进展立即停止的finite epoch；生产input source、可信Runtime resolver/连接工厂、取消cleanup、真实副作用与capability仍未完成。
  - 当前BP更新：M1-01现有A..BP六十八个`implemented`切片。BP让Flutter manage会话通过既有公共wire发出单个start/stop(false)，严格绑定terminal request/generation/idempotency/operation身份；accepted仅显示durable operation已受理与Runtime pending。operation status/cancel、durable client outbox、可信Runtime副作用及capability仍未完成。
  - 当前BQ更新：M1-01现有A..BQ六十九个`implemented`切片。BQ让Runtime preparation只能由live validated profile、signed/locked release与exact workspace/generation构造，并在scheduler CAS前重验有效期和scope；生产input source/连接工厂、取消cleanup、真实engine副作用与capability仍未完成。
  - 当前BR更新：M1-01现有A..BR七十个`implemented`切片。BR让既有`OperationCommand.get`从同scope authenticated session读取active/terminal事实，并以Control Plane schema v9把新终态快照与idempotency retention成组持久化；v8历史终态不伪造补齐。CancelOperation、cleanup、push stream、Flutter状态UI与capability仍未完成。
- `M1-02` 独立 Runtime Agent 与最小权限 UDS API；容器 engine driver trait。状态：`active`；A切片最初登记effect-fact wire/validator，M1-01J现把当前合同提升为major 1/minor 2并强制Runtime spec digest；B切片已实现默认不接线的bounded session core及Linux socket metadata+exact `SO_PEERCRED`单次accept library，C切片已实现不接线的durable Pending/Applied SQLite事实仓库，现为schema v2；D切片以显式feature policy把session和事实协调器组合为无listener endpoint，E切片新增Control Plane侧typed runtime client envelope并只让exact APPLIED形成CAS候选，F切片以durable scheduler单步协调闭合CAS后Apply、重启Query及Applied CAS，G切片实现Control Plane fresh challenge/Release/sequence握手、有界Linux UDS client及失败poison后fresh连接Query恢复，H切片在Runtime Agent侧组合一次trusted Hello、协商后frame上限、单在途与总frame预算并对任一失败永久poison连接，I切片新增一次一步、总连接有界且故障分层poison的调用方驱动连接生命周期owner，J切片以已验证profile/release本地重建完整spec并在repository/driver I/O前核对M1-01J摘要，K切片固定Linux profile report来源并与签名release/lock组合，resolver构造时即闭合freshness与profile SHA；首次Absent必须live，历史Query/Applied重放只读。minor 0/1不能协商当前feature 1；runtime-agent main仍拒绝启动且产品capability不advertise。当前HEAD自管Linux复验、profile report真实producer/原子publish、socket bind/长期supervisor loop、terminal retention/ACK watermark、safe redispatch、真实engine driver与副作用尚未完成。
- `M1-03` Linux/WSL2统一Docker Engine Runtime Profile Spike：状态`active`；A切片已`implemented` generator-owned schema v1 profile与无I/O Rust admission，B切片已`implemented` exact profile/freshness/incarnation绑定的typed probe evidence consumer，C切片已`implemented` 4 KiB canonical LF本地report codec并在解析后复用B的全部语义拒绝，D切片已`implemented` 固定4 KiB/4096非空chunk且cancel/error后永久关闭的调用方驱动intake，E切片已`implemented` preflight→same-boot/new-engine→post exact复验后才发布report的typed编排，F切片已`implemented` Linux-only有界只读host facts collector并保持main/capability关闭；G切片为`active`候选，以完成态probe、create-new临时文件、file/directory fsync、原子rename及最终identity复验发布固定report。Local-Linux当前HEAD的F/G cfg编译与真实文件系统/Docker行为、restart、quota/freezer/network黑盒、资源cleanup、可信attestation与Runtime spec回读仍pending；不依赖Docker Desktop，能力缺失即No-Go。
- `M1-04` 专用 WSL2 distribution 安装/检测/保数据升级：状态`active`；A切片已`implemented` distribution identity/preserve-data lifecycle admission、最长300秒外置签名manifest、staged切换与exact两阶段删数据合同；B切片已`implemented` durable SQLite/CAS checkpoint owner、2048-byte canonical codec与commit maintenance语义；C切片已`implemented` initial/staged Provisioning的durable-before-handoff与重启exact恢复；D切片已`implemented` signed rootfs manifest与Windows防替换artifact lease；E切片已`implemented` canonical data-root与固定WSL import plan lease；F切片已`implemented` signed exact `wsl.exe` launcher lease；G切片已`implemented` bounded process-owner port与cleanup terminal合同；H切片已`implemented` suspended-create/assign-before-resume的Win32 Job Object adapter及真实process-tree cleanup合同；I切片已`implemented` exit-zero后exact typed platform fact到固定receipt的只读归并；J切片已`implemented` receipt的Registry前置与SQLite/CAS schema v2持久化；K切片已`implemented` receipt后guest manifest验证与Ready的exact durable CAS owner；L切片已`implemented` exit-zero/store preflight后的exact fact-source→receipt CAS编排；M切片已`implemented` 有界只读HKCU Lxss Registry provider与exact adapter，Windows Host binary继续fail-closed。service SID/hive绑定、production guest provider、签名/rootfs来源、Authenticode/component servicing、真实WSL/VHD副作用与正向平台证据、receipt后guest启动与连接、禁DrvFs/interop/PATH注入、Firewall/NAT/mirrored网络及故障恢复仍未实现。
- `M1-05` 每工作区Coordinator/Execution双镜像、mount matrix、instance IPC、health/readiness、graceful stop与crash recovery。状态：`active`；A切片已`implemented` fresh validated profile到deterministic双容器/mount/network candidate，逐值绑定workspace generation、release、分离image/asset digest与engine incarnation；B切片已`implemented` canonical spec digest及exact规范化engine readback admission；C切片已`implemented` 可checkpoint的Inspect→Absence→Prepare→Reinspect→Complete协调，未知结果禁止盲目重复ensure；D切片已`implemented` 独立SQLite/CAS checkpoint owner；E切片已`implemented` absence/exact readback的durable CAS import；F切片已`implemented` schema v2 post-ensure cancel→Cleanup→Reinspect→Cancelled协调；G切片已`implemented` 最长5秒且每步一次I/O的durable engine port协调，effect错误后强制inspect；H切片已`implemented` fixed schema/time/key/signature release manifest到不可伪造verified token准入；I切片已`implemented` Runtime Agent内HMAC-SHA-512生产verifier、real/effective非root进程身份绑定与Linux owner-only key-file准入；J切片已`implemented` fixed 292-byte root-owned manifest file到verified token的一体化准入；K切片已`implemented` schema v3 partial-owned资源的commit-before-cleanup、reinspect及absence后fresh ensure恢复；L切片已`implemented`最多13项container/mutable mount source/network的canonical name/五标签/projection digest；M切片已`implemented` generator-owned十项mount source/backend/lifecycle/export/backup/delete矩阵并把source SHA绑定到spec与projection。密钥轮换/多主体非对称信任、真实engine adapter与可信container/mount/network观察、真实path/volume/quota、image pull、Runtime/scheduler长期接线、IPC、health/readiness、stop与真实crash cleanup仍未实现。
- `M1-06` Runtime Lock 与 image/Code-OSS/Node/Adapter/extension digest 验证。状态：`active`；A切片已`implemented`固定canonical Runtime Release lock及签名manifest/profile交叉准入；B切片已`implemented`由现有权威输入和实际构建Adapter树派生的generator-owned 275-byte lock、漂移门及Runtime Agent root-owned组合文件准入；C切片已`implemented`固定root-owned Runtime Node archive的128 MiB有界流式hash与lock digest强token；D切片已`implemented`固定安装根canonical Code-OSS artifact manifest bytes与lock closure digest的2 MiB流式绑定；E切片已`implemented`在exact digest之后无依赖严格解析canonical v1的schema/commit/2,863-entry路径、类型、mode、hash、native与预算；F切片已`implemented`以exact set及before/open/after identity等facts闭合live-tree reconciliation决策；G切片已`implemented` Runtime Agent固定root/fd-relative/O_NOFOLLOW collector及manifest同遍历复验；H切片已`implemented`两个角色的OCI manifest/config/layer/diff-id closure准入与G→H强token组合；I切片已`implemented` fresh同engine双角色manifest/config image ID/RootFS readback与H→I强token；J切片已`implemented` I token与M1-05 runtime spec/container/mount/network readback的exact组合；K切片已`implemented`固定Docker CLI/UDS、双host probe及list-before/inspect/list-after的可信image readback collector，并关闭公开caller-facts I入口；L切片已`implemented`同instance label下container/Docker volume/network的双snapshot exact name/type/五标签只读发现并只生成partial/absence事实；M切片已`implemented`固定2 container+1 network双inspect的image/security/mount topology/internal bridge结构采集；N切片已`implemented`固定container PID与procfs mountinfo窗口的access/nodev/nosuid/noexec/tmpfs回读；O切片已`implemented`host与Execution target同一受保护broker Unix socket inode回读。当前HEAD的Linux cfg/真实Docker行为、真实Node/Code-OSS安装树、OCI layout/blob/image构建、硬quota/firewall黑盒、完整J token collector及engine副作用仍pending。
- `M1-06P` tmpfs硬quota readback：状态`implemented`（Windows parser合同）；在N的同一mountinfo窗口中把两个container tmpfs的actual `size`/`nr_inodes` exact绑定到profile并写入v2摘要。persistent volume project/qgroup、broker parent quota、restart超限黑盒与Linux真实重放仍pending，因此M1-06保持`active`。
- `M1-07` 端口预览反向代理、出站策略和运行日志限额。

### 验收

- Linux 与 Windows/WSL2 通过同一create/start/stop/delete/pause/quota/network黑盒suite；M1 Spike不能证明硬quota/freezer时必须No-Go或更换底层。
- 重复命令、Control Plane/Runtime Agent/container 重启不创建重复资源。
- Execution看不到authority/Runtime socket/控制凭据或Coordinator进程；Coordinator不含项目执行工具。mount matrix、硬quota、默认拒网和双容器独立pause负向测试通过。
- WSL在空白service identity安装；`/mnt/c`/Windows interop、动态IP直连和普通卸载删VHD全部被拒绝。
- Runtime Release 不匹配时 fail-closed，错误可操作且无半启动工作区。

## 6. M2：Workspace Agent 与文档事实源

### 工作包

- `M2-01` Workspace Agent 生命周期、Adapter supervision、UDS nonce/权限。
- `M2-02` 安全 workspace-relative VFS、watcher、search 和大文件策略。
- `M2-03` DocumentActor：rope/piece table、canonical/saved version、journal、snapshot。
- `M2-04` Mutation Broker：client、extension、formatter、Agent、external change 的单一路径。
- `M2-05` 单写 lease/epoch、observer、幂等 EditBatch、revision conflict。
- `M2-06` event log/cursor、compaction、snapshot fallback、crash recovery。
- `M2-07` language/diagnostic/contribution 的内部投影入口。

### 验收

- model/property/fault tests 证明 Ack edit 不丢失、不重复，过期 edit 无部分副作用。
- UTF-8 文件与 VS Code UTF-16 position 转换通过共享语料。
- 多客户端 observer 顺序一致；单写租约无双 holder；断线恢复可预测。
- 所有产品API介导写入经Mutation Broker；Node/PTY/Git直写被watcher+metadata/hash可靠提升为external revision，dirty重叠进入显式冲突且不伪造ACK。
- M2只在内部fixture/harness验证文档stream；因为首版DOCUMENT恢复闭包还包含Language UI投影，M4完整闭包就绪前产品服务不得advertise该组合。

## 7. M3：Flutter 原生客户端与编辑器

### 工作包

- `M3-01` app shell、OIDC PKCE、secure storage、connection/session reducer。
- `M3-02` 工作区树、command palette、QuickPick/InputBox/notification/状态栏原生组件。
- `M3-03` editor core：文本结构、viewport、glyph layout、selection、cursor、undo/redo。
- `M3-04` TextInput/IME、快捷键、clipboard、drag/drop、accessibility。
- `M3-05` decoration、diagnostic、semantic token、completion、hover、rename/diff。
- `M3-06` 离线/断线 projection、pending edit UI、revision conflict/lease handoff UX。
- `M3-07` Android/iOS/Windows/macOS/Linux packaging 和平台适配。

### 验收

- 源码和依赖扫描无 WebView/HTML editor；五端共享 editor core。
- 中文 IME、emoji、组合字符、RTL、硬件键盘与 accessibility 核心矩阵通过。
- 1/10/32 MiB 文件达到已登记性能预算；超限文件明确只读/分页降级。
- 网络抖动、后台恢复和 lease 丢失不会静默丢字或伪造保存状态。
- M3可使用协议fixture验证编辑器，但不得把缺少M4语言投影producer的部分实现作为可协商DOCUMENT stream发布。

## 8. M4：C0/C1 扩展兼容

### 工作包

- `M4-01` 将 G0 Adapter 工程化，绑定 Runtime Release，supervision/telemetry/cleanup。
- `M4-02` C0：commands、configuration、workspace.fs、documents/editors、languages、diagnostics、search。
- `M4-03` C1 MainThread/Flutter投影：QuickPick/InputBox、TreeView、StatusBar、OutputChannel，以及以确定性stub验证的Terminal/Task actor；不在本阶段实现PTY/Task执行后端。
- `M4-04` contribution registry snapshot/delta、可信command policy registry/digest与Flutter renderer registry；未知command默认拒绝。
- `M4-05` extension memento/secret storage、URI、clipboard/openExternal 安全投影。
- `M4-06` 固定 `extensions.lock`、VSIX cache、认证报告、native Node ABI smoke test。
- `M4-07` Unsupported API 统一错误与扩展熔断。

### 验收

- C0/C1 fixture 与批准的实际扩展清单逐项认证；失败功能在 UI 可见。
- M4只认证无需真实PTY/Task执行后端的C0/C1路径；Terminal/Task actor仅跑确定性stub，禁止advertise `CAP_TERMINAL_V1`/`CAP_TASK_V1`。对应实际扩展与端到端能力在M5认证。
- DOCUMENT stream的`STATE_RESUME + DOCUMENT_EDIT + DOCUMENT_LEASE + LANGUAGE_UI`正文/租约/diagnostic/semantic完整恢复闭包通过后，才可对产品客户端整体advertise；不允许只打开部分能力后过滤同一cursor中的事件。
- `activeTextEditor`等单窗口全局状态只由独立`workbench_control_lease`持有者驱动；document/terminal/debug租约互不授权，observer不污染Extension Host状态。
- Adapter/Extension Host 崩溃重启后贡献、文档和 UI 请求不重复或悬挂。
- Webview/Custom Editor/DOM/Electron 调用明确拒绝；无隐藏 HTML 回退。

## 9. M5：IDE 工作流

### 工作包

- `M5-01` 将M4的Terminal actor接入真实Execution PTY：生命周期、output offset/scrollback、input lease、resize/reattach。
- `M5-02` Git/SCM group/resource/diff/stage/commit 投影与凭据 broker。
- `M5-03` 将M4的Task actor接入真实执行后端：Task执行、problem matcher、日志、cancel/cleanup。
- `M5-04` DAP session、breakpoints、stack/variables、console 与 reconnect 语义。
- `M5-05` Testing tree/discovery/run/coverage 投影。
- `M5-06` language request 的 cancel、stale result、partial progress 和资源预算。

### 验收

- 终端、任务、调试和测试均使用 typed Operation，terminal 恰好一次。
- observer 只读，输入/调试控制权显式；断线/转移可恢复。
- 凭据不进入 terminal env、日志或 Flutter；Git 危险操作有显式确认策略。
- 至少一个主流语言扩展和 Git/Debug/Testing fixture 完成完整工作流认证。

## 10. M6：服务端 Agent

### 工作包

- `M6-01` provider-neutral Agent task/checkpoint/event schema。
- `M6-02` Tool Registry 与 broker：read/search/edit/terminal/test/git/network。
- `M6-03` 风险分类、结构化 approval、deadline、cancel、budget、secret injection。
- `M6-04` candidate patch/diff/review/apply，复用 DocumentActor/lease。
- `M6-05` prompt/context 预算、工作区索引、诊断/测试反馈循环。
- `M6-06` 多端观察、审批归属、接管和任务恢复。
- `M6-07` prompt injection、批准重放、数据外泄测试与审计。
- `M6-08` C3 Code-OSS Bridge（依赖M4）：LM Tool、Chat Participant/Session、MCP与Agent diff/comments actor映射。
- `M6-09` C3 fixture、同 commit 纯 Node reference harness/API golden 可观察行为对照、Adapter/ExtHost重启与snapshot恢复，并重复零 Electron/Chromium 工件与进程树门禁。

### 验收

- 模型无直接文件/shell/network 权限；所有副作用可归因到 tool call 和批准。
- 参数改变、lease epoch 改变或批准过期必然重新授权。
- Agent edit 与用户/扩展 edit 使用同一冲突模型，不可绕过版本。
- cancel/timeout/restart 后资源清理、task terminal 与 checkpoint 自洽。
- Rust native Agent core与C3 bridge分别通过合同；M4未完成时可开发core，但M6不能accepted或宣称C3兼容。

## 11. M7：多端同步、一致性导出与恢复

### 工作包

- `M7-01` workspace/document/terminal/agent/extension/operation/scm/debug/testing/export/task/output每流cursor与ACK，并按StreamKey继承完整domain capability。
- `M7-02` reconnect planner：replay、snapshot、reset 与 backpressure。
- `M7-03` recent、theme、keymap、固定标签等用户工作区偏好同步和冲突策略；窗口/面板布局、滚动、光标、选择保持device-local。
- `M7-04` 写租约申请、续约、释放与同owner设备接管 UX；跨principal attach首版拒绝。
- `M7-05` 容器外durable ExportCoordinator、steady-state双容器freeze、临时Exporter Helper容器、volume lock、syncfs、manifest、artifact原子publish、exchange→Range download session。
- `M7-06` control/workspace DB migration、backup、restore、quarantine 和灾难恢复 runbook。
- `M7-07` 多设备/多网络/长时间离线/fault/soak suite。

### 验收

- 断线窗口内准确 replay，超窗口明确 snapshot/reset；无静默 gap。
- 同一owner的两个及以上不同平台设备可观察同一工作区，租约转移后旧 writer 被拒绝；不同principal attach失败且无状态泄漏。
- 成功导出manifest与内容hash一致；所有pause/helper/publish/unpause故障点可恢复且无永久pause/半成品下载；屏障期间edit有明确排队/拒绝语义。
- Windows 端只通过 WSL2 容器 volume 工作，导出可在外部校验/解压；不引入 Windows 工作区。
- 备份恢复演练重建 ACL、schema、cursor、secret 和工作区可用状态。

## 12. M8：生产硬化与发行（休眠）

状态：`dormant`。ADR-0001冻结当前不分发任何制品、不运营公共托管服务。以下工作包和验收条件保留为未来恢复制品发布时的完整清单，不阻塞开发里程碑；恢复必须有新的显式ADR。

### 工作包

- `M8-01` 全平台安装/升级/卸载：Flutter 客户端、Linux service、Windows WSL2 distribution。
- `M8-02` 可观测性 dashboard/alert、支持 bundle、隐私与保留策略。
- `M8-03` 性能、容量、72h soak、故障注入、安全评审/渗透。
- `M8-04` Code-OSS/扩展许可证与品牌、SBOM、签名、provenance。
- `M8-05` N/N-1 兼容、canary、停止扩散、rollback/forward-only migration 演练。
- `M8-06` 运维 runbook、SLO、支持矩阵、已知限制和用户文档。

### 验收

- `gate-full` 在最终 clean tag 全绿，无 skip；所有工件 hash 与签名由消费者验证。
- Linux 和受支持 Windows build 的安装/升级/恢复/卸载在空白机器通过。
- 关键 SLO、有界资源、告警和事故开关经演练验证。
- 活跃 Critical/High 风险为零；Medium 风险有 owner、到期和已接受范围。

## 13. 首版范围裁剪顺序

遇到资源或进度压力时，按以下顺序延期：

1. C3 Agent 深度集成和第三方 provider 数量。
2. C2 Debug/Testing/Comments/Auth 等长尾扩展 API。
3. 移动端高级快捷键、复杂 diff/merge 与超大文件编辑。
4. macOS/Linux 桌面发行包装（核心仍保持可编译与测试）。
5. SCM 高级能力、端口预览策略 UI。

不得裁掉：G0、无WebView、服务端对API介导写入的唯一ACK/版本权威、外部写归并、Code-OSS exact lock、双容器隔离、公共协议验证、单写租约、导出一致性、安全拒绝、恢复测试和生成物Gate。

## 14. 任务模板与状态

每个任务至少写明：ID、目标/非目标、owner、依赖、契约变化、修改边界、资源上限、威胁、测试矩阵、观测、回滚/清理、验收证据。状态只使用：

- `planned`：尚未开始。
- `active`：owner 已开始，依赖满足。
- `blocked`：具体外部条件阻塞，列出解除条件。
- `implemented`：代码完成，局部门禁通过。
- `accepted`：集成提交的完整验收证据通过。
- `superseded`：由显式 ADR/任务取代，保留链接。

阶段完成不按任务数量计算；任何硬性验收失败，阶段仍未接受。
