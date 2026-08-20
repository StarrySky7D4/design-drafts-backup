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

- `M1-01` Rust Control Plane modular monolith：auth、ACL、workspace metadata、scheduler、operation。
- `M1-02` 独立 Runtime Agent 与最小权限 UDS API；容器 engine driver trait。
- `M1-03` Linux/WSL2统一Docker Engine Runtime Profile Spike：engine权限、双容器、volume硬quota/inode、cgroup freezer、network deny-first与admission；不依赖Docker Desktop。
- `M1-04` 专用 WSL2 distribution 安装/检测/保数据升级：真实service SID、禁DrvFs/interop/PATH注入、Host↔WSL manifest握手、Firewall/NAT/mirrored网络和VHD精确生命周期。
- `M1-05` 每工作区Coordinator/Execution双镜像、mount matrix、instance IPC、health/readiness、graceful stop与crash recovery。
- `M1-06` Runtime Lock 与 image/Code-OSS/Node/Adapter/extension digest 验证。
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
