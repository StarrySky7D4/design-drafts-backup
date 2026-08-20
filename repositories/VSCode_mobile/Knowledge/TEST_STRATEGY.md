# 测试、验证与质量门禁

> 测试目标不是证明“进程能启动”，而是证明跨 Rust、Flutter、Code-OSS、容器和持久化边界的行为兼容、失败可恢复且没有越权副作用。

开发验收与制品发布资格相互独立。`gate-quick`及受影响域/平台证据用于开发推进；`gate-full`是休眠M8/生产发布门，当前失败不会阻止M0/G0 accepted或M1进入。任何制品仍固定`release_eligible=false`，直到显式ADR恢复发布体系并补齐全部发布证据。

## 1. 测试分层

| 层级 | 范围 | 反馈目标 | 主要责任 |
|---|---|---|---|
| L0 静态 | format、lint、依赖、schema、生成漂移 | 分钟内 | 各模块 owner |
| L1 单元 | reducer、Actor、parser、lease、path、policy | 分钟内 | crate/package owner |
| L2 合同 | Protobuf、Rust↔TS↔Dart、Code-OSS actor | PR 内 | protocol/codeoss |
| L3 组件 | Control Plane、Workspace Agent、Adapter、Flutter editor | PR/合并队列 | 服务 owner |
| L4 单工作区 E2E | 客户端到容器完整路径 | 合并队列/夜间 | QA/platform |
| L5 平台与故障 | Linux、WSL2、多端、崩溃、网络、资源 | 夜间/发布 | release/security |
| L6 对照认证 | 同 commit 纯 Node reference harness、API fixture/golden、扩展兼容 | Code-OSS/扩展变化 | codeoss |

快门禁必须可靠、无网络和可复现；慢门禁允许真实容器/WSL2，但必须输出固定证据与可重放 seed。

## 2. L0 静态门禁

计划统一命令：

```text
just gate-quick
  cargo fmt --check
  cargo clippy --workspace --all-targets --all-features -- -D warnings
  cargo nextest run --workspace
  dart format --output=none --set-exit-if-changed .
  flutter analyze
  flutter test
  pnpm install --frozen-lockfile --offline --ignore-scripts
  pnpm run lint && pnpm test
  zero-electron-chromium dependency/image/process gate
  buf lint && buf breaking --against <last-stable>
  just generated-check

just gate-full
  gate-quick
  contract + component + e2e + fault + security + compatibility
```

`gate-quick`/`gate-full`名称、退出语义与证据目录保持固定。缺少所需工具或测试环境是失败，不能静默skip；但`gate-full`只在M8由新ADR激活后参与制品发布资格，不再反向阻塞开发accepted。

路径effect-surface由`governance/change-control-v1.json`机器分类：data-only与governance-contract变化运行治理、repository baseline、零浏览器及其登记的Windows门，可继承最近同一实现树的自管Linux证据；product-runtime、protocol/generated、lock/toolchain或安全门变化必须重跑受影响域和平台。分类未知时按product-runtime，mutation必须证明server源码或未知二进制不能伪装成data-only。

静态检查还包括：

- `cargo deny`/许可证和漏洞策略、Dart/npm lock 一致性。
- 禁止 `unsafe` 默认；例外必须模块级注释、owner 与专用测试。
- 禁止手改生成绑定、actor inventory、SBOM 和 Runtime Lock 派生数据。
- Protobuf field/enum number、稳定错误码、capability 与 schema 注册表漂移检查；能力矩阵对顶层/嵌套oneof、action enum、request-response/outcome、StreamKey、event/snapshot分支identity、UI subtype和transfer状态机做机器穷尽检查；资源Uuid逐值等式由M0生成validator及跨资源错配fixture验证。
- 禁止 Flutter 引入 WebView、HTML renderer、Electron 或客户端文件写入依赖。
- 零 Electron/Chromium 负向 Gate 必须覆盖仓库依赖图/lockfile、CI与测试/开发工具、SBOM、安装包、容器镜像、发布目录和受测进程树；拒绝 Electron/Chromium/Chromedriver、`@vscode/test-electron`、Playwright/Puppeteer、sidecar与隐藏浏览器。只读 upstream 源码提取必须与依赖恢复和发布输入隔离。

## 3. 公共协议合同测试

`protocol/fixtures/ide/v1` 应包含各语言共享的二进制 golden 与 JSON 说明：

- 最小/完整 `ClientHello`、capability 协商和不兼容 major。
- delivery矩阵：request/response/event/snapshot/result-transfer/server-request/control互斥；空delivery、response+event、无terminal、多terminal、terminal后续response part、part缺口/重复全部拒绝，Future、reducer、snapshot installer与result assembler各只消费自己的分支。transfer-start响应为terminal，后续只按transfer ID路由；Snapshot与Result绝不混用cursor/installed ACK。
- document snapshot、UTF-16 position、组合字符、emoji、CRLF/LF 和超长行。
- edit accept、重复幂等、revision conflict、lease epoch 过期与只读客户端。
- Save/Revert/SetLanguage及策略受控ExecuteCommand覆盖“失去租约→更高epoch重获→旧命令迟到”的ABA；准入与最终commit任一fence变化均零副作用拒绝。
- resume replay、snapshot fallback、snapshot→live high-water cut、统一Snapshot Begin/Chunk/End/Aborted的schema/part count/offset/大小/未压缩原始bytes完整hash、chunk不占event cursor且验hash后原子安装/ACK、key/kind/resource ID错配、ACL/device中途撤销、stream reset、重复/倒序/缺口事件；ResultTransfer单独覆盖成功/abort/cancel且从不推进frontier。
- 多流独立ACK、future/cross-session/cross-generation ACK拒绝、`SnapshotInstalled`身份/digest/H不匹配拒绝、饱和data queue下预留control reset或强制close、下次Hello强制snapshot。
- 每个resume cursor/ACK/reset/snapshot part按`StreamKey`继承完整domain capability；未协商terminal/agent/debug等key在副作用或数据投递前拒绝。DOCUMENT四项闭包缺任一能力时不得订阅，不能产生过滤后的cursor洞。
- 请求只能由矩阵列出的response/outcome/result kind终结；嵌套branch错配、transfer kind漂移、UI subtype错配和Clipboard非controller routing全部作为协议/授权拒绝。
- CancelRequest/Result/Snapshot与UI response/cancel覆盖unknown/foreign ID、跨session/recipient、能力继承、timeout/disconnect以及cancel-vs-terminal/End竞态；每个状态机只有一个赢家。
- `client_instance_id`跨session续传、跨principal/device复用拒绝、ACK丢失幂等返回原结果。
- terminal byte offset、UTF-8 分片、二进制输出、scrollback gap、normalized VT checkpoint（SGR/cursor/modes/alternate screen）安装+raw replay，以及显式clear/reset降级。
- Workbench control/document/terminal/debug四类租约隔离；定向Selection、tab group/order、WorkbenchContextUpdate、QuickPick分页、Dialog/Progress/Clipboard的UI request/response/cancel、timeout、disconnect、unknown request。
- QuickPick分页必须匹配active request、selected recipient、extension generation、items handle和page token；同owner observer与跨UI请求handle均拒绝。
- LeaseState响应/event及Workspace/Document/Terminal/Debug snapshot把target branch与UUID分别替换为另一合法资源，全部应在安装/投递前拒绝。
- LeaseCommand按target做capability/permission负向矩阵：DocumentWrite不能取得workbench/terminal/debug租约，TerminalInput/DebugControl也不能越界，旧epoch和错误资源归属无副作用。
- ExecuteCommand覆盖未知command、扩展伪造policy、digest漂移、缺动态capability/permission、Trust/Workbench lease缺失，以及command间接触发BulkEdit/Task/Terminal/Debug/SCM；全部必须在窄Broker再次鉴权。
- Tree/SCM/Testing/DAP分页与continuation过期；completion/code action resolve强制extension generation和document version；diagnostic/semantic full原子单事件、超限snapshot-required、snapshot install清除旧派生状态与legend/full/delta/reset。
- Language的10个request branch逐一做result错配拒绝；rename prepare/apply互斥，continuation token不能切换completion/location/code-action family或跨session/document/version/extension generation。
- DAP evaluate必须走带debug control lease和stop epoch的command；continue后旧frame/variables handle全部拒绝。
- Agent event只含脱敏参数与exact digest，组合risk flags、approval绑定精确参数与每target LeaseFence、多文档fence缺失/错epoch拒绝；候选大edit走无cursor的ResultTransfer，读取/应用同时要求Agent+DocumentEdit+DocumentLease，cancel/commit race与exactly-once terminal。
- Task与Output append/replace/clear/show/hide、重连快照、终态和大scrollback分块。
- FileMutation与WorkspaceEdit resource operation覆盖旧revision/digest、create/rename source/target竞态、dirty open document、recursive升级Operation：option flag不能放宽candidate中的精确前置条件，过期客户端100%冲突且无部分覆盖/删除。
- Operation cancel覆盖跨principal、缺kind权限、旧workspace generation、旧`expected_operation_revision`、ACK丢失和cancel-vs-commit竞态；只有一个持久终态且重试返回相同结果。
- export success/too large/cancel、exchange token重放、ACL撤销、并发Range、过期中断和artifact版本绑定。
- 所有 PublicError 与资源上限的正向/拒绝向量。

Rust、Dart、TypeScript 各自加载同一 bytes：解码、校验、重编码必须得到 canonical 等价结果。拒绝向量必须证明业务数据库、文件、终端和事件序号没有变化。

Protobuf fuzz 目标包括 varint、长度、嵌套、repeated、oneof、未知 enum、无效 UUID、压缩声明和 frame 拆分；先预算后分配。

## 4. 文档与多端一致性测试

DocumentActor 使用模型测试/状态机测试。操作集至少含 open、edit、save、external change、lease acquire/renew/release、client disconnect、container restart、snapshot compact 和 export barrier。

必须验证：

- 每个接受的 mutation 只产生一个 canonical version；重复幂等键返回原 Ack。
- `base_version` 过期不会部分应用；错误携带可恢复信息。
- saved version 永不领先 canonical version；落盘失败不会报告 saved。
- 单写租约任意时刻最多一个 holder；epoch 单调，旧 holder 无法继续写。
- observer 收到同一有序事件；落后超窗口时收到 snapshot/reset，不拼接缺失历史。
- 服务崩溃在 journal commit 前后都可恢复到已定义状态，不能出现 Ack 后数据永久丢失。
- 产品API介导写入经Mutation Broker；Node/PTY/Git直写被watcher+metadata/hash提升为external revision，dirty重叠显式冲突且不伪造Ack。
- server/Workspace Agent/Coordinator/Execution/WSL重启使所有瞬时租约unowned并提升epoch；旧Renew/Edit/Input/Debug请求100%拒绝。
- 同一owner多设备可观察/接管；不同principal即使持有猜测workspace/session ID也无法attach、读取个性化snapshot、extension-state或SecretStorage。

可使用属性测试生成并发操作和断线点，与简单参考状态机比较最终文本、版本、租约和事件流。

## 5. Code-OSS 与扩展兼容认证

### 5.1 G0 可行性夹具

在写大规模 Flutter UI 前，固定一个 Code-OSS commit 完成：

1. 启动 Headless Adapter 与真实 Extension Host。
2. 加载自有 fixture extension。
3. 覆盖命令注册/执行、configuration、workspace.fs、open/change/save document、completion、diagnostic、QuickPick、TreeView、OutputChannel 和 PTY。
4. Adapter 经 UDS 发送结构化请求，Workspace Agent 接收并回送 UI 响应。
5. 与同一 commit 的纯 Node reference harness 执行相同 fixture，比较结构化可观察结果、错误、顺序和 API golden；禁止 stock Electron Workbench、浏览器自动化或 mock-only oracle。
6. 检查依赖锁/SBOM/镜像与安装包文件名和内容，并观察受测进程树；发现 Electron/Chromium 下载、依赖、二进制或子进程立即失败。
7. 从无`node_modules`/`out`的clean exact checkout运行最小源码闭包构建；只允许过滤安装、`--frozen-lockfile --ignore-scripts`、Linux实际依赖inventory/CycloneDX和制品内symlink。构建前后tracked dirty必须为0。
8. Linux进程树负测用非浏览器`/bin/sleep`伪造含`chromium`的argv0，必须被拒绝；清理后正向扫描必须恢复通过。该fixture不得下载或启动真实浏览器。
9. 外置minimal runtime必须证明root/entry owner等于执行UID、全树同device、普通文件`nlink=1`，并在文件open前后比较dev/inode/uid/nlink；执行入口先复制到新建owner-only临时根、对快照重新验证，再只从快照import/执行。hardlink、owner/cross-device和“验证后修改源树”均为拒绝fixture，前后验证不能冒充不可变执行窗口。

若不能稳定启动、actor 清单无法固定，或核心 API 需要重写 Extension Host 内部协议，G0 判失败并停止主线投入。

### 5.2 每扩展认证

每个 `extensions.lock` 条目有机器可读报告：安装、激活事件、命令、所用 API、native module、C0-C3 功能、资源消耗、错误、退出清理和许可证。加载成功不等于兼容。

Code-OSS 升级必须：

- 对 actor/contribution inventory 做结构 diff。
- 在新旧 commit 上运行同一 fixture 与 bundled extension suite。
- 验证 Node exact version/modules ABI 和 native extension smoke test。
- 证明所有 Unsupported API 明确失败，不阻塞 Extension Host。

## 6. Flutter 编辑器与平台测试

Flutter 不使用 WebView，编辑器核心需分别测试 model 与 rendering：

- rope/piece table、行索引、UTF-8↔UTF-16 映射、增量布局与 viewport virtualization。
- selection、multi-cursor、undo/redo、find、fold、decoration、diagnostic、semantic token。
- Android/iOS/Windows/macOS/Linux 的键盘、快捷键、焦点、剪贴板和 accessibility semantics。
- 中文/日文/韩文 IME composing、候选窗口、emoji、RTL、组合字符、死键和硬件键盘。
- 60/120Hz 拖动、超大文件只读降级、后台/前台、低内存恢复。
- snapshot/event reducer 的黄金状态；UI screenshot/golden 只验证稳定布局，不替代行为测试。

基准数据集至少包含 1、10、32 MiB 文件，100k/1M 行，高频 diagnostics 与 10k decorations。目标阈值见 `PROJECT_CODING_PLAN.md`，CI 对 p95 和内存设回归预算。

## 7. Runtime、容器与 WSL2 测试

Linux 与 Windows/WSL2 必须运行同一黑盒 suite：

- 冷/热启动、幂等 create/start/stop/delete、重复命令与 agent 重启。
- image digest/Runtime Lock 不匹配、container name 冲突、旧 socket、stale pid。
- CPU/内存/PID/磁盘/inode 限额，OOM、磁盘满、fork bomb 和日志爆量。
- Coordinator/Execution双容器独立start/pause/recreate；Execution无法读改authority、连接Runtime socket、signal/ptrace Coordinator或看到其PID/FD。
- Runtime Agent/Control Plane独立UID与`0660`专用group socket：只有精确ide-server `SO_PEERCRED`可调用，其他宿主UID、group外进程和容器socket访问全部拒绝。
- Exporter Helper仅导出时作为临时第三容器：固定digest、无网络/authority/control socket、RO workspace；Runtime Agent无宿主项目读取mount，helper退出/重启无孤儿scratch/mount。
- mount matrix逐项回读：owner、ro/rw、exec、nodev/nosuid、硬bytes/inode quota、export/backup策略；symlink escape、禁止Docker socket/host network/privileged。
- 网络策略在ready前deny-first；阻断IPv4/IPv6 host gateway、RFC1918、link-local/metadata、literal IP、DNS rebinding、代理绕过；engine/WSL重启后先deny再恢复。
- WSL distribution未安装、停止、升级、磁盘compact失败、Windows sleep/restart和网络重建。
- 真实Windows service SID空白安装/枚举；普通用户不可见/篡改；`/mnt/c`、Windows interop/PATH注入和动态WSL IP直连全部拒绝。
- Host↔WSL Runtime manifest失配fail-closed；升级/compact/普通卸载失败保留旧VHD与数据，显式删数据要求精确ID/路径/generation。
- Runtime Agent 与 Control Plane 版本 N/N-1 窗口及未知 future major 拒绝。

WSL2测试机不能依赖开发者已有Docker Desktop状态；从声明的distribution/image cache初始化，并记录Windows build、WSL version、kernel、cgroup/freezer、container engine、quota backend与镜像digest。M1必须以实测冻结统一Runtime Profile；缺硬quota或可靠pause即失败。

## 8. 故障注入矩阵

在以下提交点前后注入进程退出、socket 断开、deadline、I/O error 和重复消息：

| 流程 | 关键提交点 | 必须保持的不变量 |
|---|---|---|
| Edit | journal commit、text apply、event append、Ack | 不丢已 Ack edit、不重复版本 |
| Save | temp write、fsync、rename、saved version | 不报告假保存、不留下越界路径 |
| Export | admit、volume lock、Execution pause、watch drain、Coordinator pause、helper create、syncfs/open、archive finalize、artifact publish、job terminal、descriptor、helper stop、Coordinator/Execution unpause | helper前失败用`HelperNeverStarted`回读证据，helper后失败用terminal+mount-close证据；成功artifact一致，失败不可下载、无永久lock/pause/孤儿mount，重叠stop/delete/backup互斥 |
| Operation | start、resource acquire、cancel、commit、terminal | terminal 恰好一次，cleanup 完成 |
| Domain cancel | terminal/task/debug/testing/export cancel与Operation cancel并发 | 全部委托同一OperationHost，domain与operation终态一致 |
| Lease | grant、renew、expiry、disconnect、server/WA/container/WSL incarnation | 同epoch不出现双writer；重启不复活旧holder |
| Resume | tail registration、snapshot cut、replay、ACL撤销、cursor persist | `> high_watermark`无缝接live，不跳过/泄漏/静默错序 |
| Workspace | container start、agent ready、adapter ready | 状态单调且重试幂等 |
| Migration | backup、transaction、version write | 失败回滚，高 schema 不被旧版写 |

每个故障用确定seed，输出operation timeline、相关log/trace id和恢复后状态摘要。Export矩阵在每个点前后分别kill ide-server、runtime-agent、export helper、Coordinator、Execution和WSL；Runtime Agent必须先cancel/wait helper、关闭mount，再按Coordinator→Execution顺序恢复。

## 9. 安全测试

安全 suite 与 `SECURITY.md` 一一映射：

- ACL/IDOR 全主体矩阵，尤其 WebSocket resume、download 和 terminal attach。
- path traversal、Unicode 混淆、symlink/junction race、tar/zip slip。
- 恶意VSIX/项目尝试读改authority DB、连接Coordinator/Runtime socket、kill/ptrace Coordinator、枚举其`/proc`/FD、读取其他volume或控制面secret。
- Agent prompt injection、批准重放/篡改、tool参数替换和secret exfiltration；恶意子进程枚举`/proc/*/environ`、共享目录、credential endpoint和遗留FD。
- frame/文件/终端/diagnostic/UI queue 资源耗尽和慢消费者。
- X.509/TLS 配置、token 撤销、ticket 重放、设备撤销。
- 日志/trace/crash/export/fixture secret 扫描。
- SBOM、签名、provenance 和许可证策略实际拒绝负向 artifact。

高严重度发现无 waiver；临时 waiver 必须有 owner、到期日、隔离措施和发布范围。

## 10. 性能、容量与长稳

核心 SLO 的可重复基准：

- 键入到本地绘制 p95 < 16 ms（本地 optimistic projection）。
- 同区域 edit Ack p95 < 120 ms；事件传播 p95 < 200 ms，明确区分网络条件。
- 热工作区 ready p95 < 5 s，冷启动 p95 < 30 s（按标准镜像/机器定义）。
- 10 MiB 文档首次可编辑、搜索、diagnostic 投影和滚动帧率分别设预算。
- 24/72 小时长稳无无界 queue、journal、PTY scrollback、Extension Host 重启循环或内存持续增长。
- 1/5/20 客户端 observer 和多工作区并发压测，单写 writer 语义保持。

基准结果绑定硬件、OS、WSL/kernel、网络模型、commit、image digest 与数据 seed。没有这些元数据的数字不进入发布判断。

## 11. 自管验证拓扑、可选CI与双层证据

- 本机Windows：L0-L3、协议fixed vectors、Rust/Flutter/Adapter、Windows runner与快速拒绝门。
- 自管Linux（当前Local-Linux2）：Linux单工作区E2E、同commit纯Node reference/golden、零Electron/Chromium供应链与进程树门、容器安全基线及Linux Flutter build。
- 自管计划任务：WSL2、Flutter真机/桌面矩阵、fault/property/fuzz与长稳抽样；调度器可以替换，但产物合同不变。
- 开发候选：在本机Windows与自管Linux对受影响实现面运行锁定门；纯data-only/governance变化按机器路径分类继承同一实现树证据。
- 源码公开候选：gate当前planned/non-green；未来必须以有界可执行规则检查secret/用户数据/机器路径、LICENSE/NOTICE/上游来源、lock/generated drift、cache/大型工具链/构建工件与零浏览器卫生门，未通过前不得公开。
- 制品发布候选：当前inactive；未来ADR恢复后才在上述自管执行面完成全矩阵、迁移/恢复演练、扩展认证、SBOM/license/signature/provenance与canary。
- GitHub Actions或其他托管/远程CI只可镜像同一锁定命令并保存辅助日志；`remote_ci_required=false`，未配置、不可用或未运行不能单独阻止开发accepted或未来发布，也不能替代自管执行结果。

每次未来制品发布生成不可变evidence manifest，至少列出：source commit、dirty=false、toolchain、Code-OSS commit、Node ABI、protocol/schema windows、Runtime Lock、image digest、extension lock、每个gate结果与artifact hash。当前M0 evidence skeleton继续为空gate results且不可发布；它不否定单独记录的开发accepted证据。

## 12. 完成定义

一个功能只有同时满足以下条件才是完成：

- 协议/持久化/错误/资源语义已经登记并有 owner。
- 正向、拒绝、断线、cancel、deadline 和 restart 路径有自动测试。
- Rust、Dart、TypeScript 合同一致，生成物零漂移。
- 指标、日志、trace、审计和用户可恢复错误已实现。
- 相关平台矩阵通过，文档、风险和 runbook 更新。
- 证据来自最终 clean commit，且没有静默 skip。
