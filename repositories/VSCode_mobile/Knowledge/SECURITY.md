# 安全模型与隔离基线

> 本文描述首版必须具备的安全边界。容器是工作区隔离手段，不应被宣称为可承载完全不可信恶意租户的强虚拟机边界。

ADR-0001冻结项目为作者自用、最多公开源码，当前不分发制品或运营公共服务。该定位只让M8发布材料休眠，不降低本文件的运行时安全边界；一旦增加网络监听，认证、会话、ACL、资源归属和TLS仍须在任何业务副作用前闭合，单用户部署不得裸露公网服务。

## 1. 威胁模型

需要防御的主体包括：被盗客户端凭据、恶意或有漏洞的 VS Code 扩展、项目中的恶意构建脚本、被提示注入诱导的 Agent、越权用户、异常客户端、受污染的依赖和被攻陷的单个工作区容器。

首版不承诺在同一宿主机上安全运行主动攻击内核的敌对租户。若业务需要该能力，必须把高风险工作区切换到独立 VM、microVM 或独占节点，并保持本协议不变。

主要资产：

- 用户身份、设备密钥、刷新令牌与恢复码。
- 工作区文件、Git 凭据、扩展 secret storage 与 Agent provider 凭据。
- Extension Host、终端、调试器和 Agent 工具的执行权限。
- 审计记录、导出产物、运行时镜像和供应链签名。
- 多端状态流的顺序、完整性和租约所有权。

## 2. 信任区与网络边界

| 区域 | 权限 | 可达性 | 原则 |
|---|---|---|---|
| Flutter 客户端 | 用户交互、设备私钥 | 仅 Gateway TLS | 永不持有容器/宿主机凭据 |
| Gateway/Control Plane | 身份、ACL、调度元数据 | 公网入口；内部访问 Runtime Agent | 不执行项目代码 |
| Runtime Agent | 容器生命周期、挂载与网络策略 | 仅本机 UDS/命名管道 | 独立进程、最小特权、严格命令白名单 |
| 工作区容器 | 项目、Extension Host、工具链 | 默认无入站；受控出站 | 按工作区隔离，不能访问控制面存储 |
| Workspace Coordinator | 文档事实源、协调写入、外部变更归并、导出准备 | 独立可信容器，主动mTLS控制流/实例UDS | 只拥有 authority + workspace，不执行项目代码 |
| Workspace Execution | Adapter、扩展、PTY、语言与构建进程 | 独立非可信容器，经窄IPC连接Coordinator | 只拥有workspace/extension-state，不可见authority/control socket |
| Artifact Store | 只存短期导出对象 | 经签名下载描述访问 | 每对象 ACL、TTL、hash |

外部流量只允许 TLS 1.3/1.2；生产禁用明文监听。内部 UDS 使用文件权限和进程身份鉴权，消息仍携带工作区实例随机 nonce，防止错误连接到旧 socket。

## 3. 身份、设备与会话

- 使用 OIDC Authorization Code + PKCE；移动端/桌面端都通过系统浏览器，不嵌入 WebView。
- Access token 短期有效；refresh token 存系统安全存储。服务端数据库只保存可撤销会话记录和 token hash，不保存明文。
- 首次登记生成 `DeviceId` 与设备密钥。敏感操作可要求设备签名 challenge；设备名称不参与授权。
- WebSocket 建连先完成 HTTPS 鉴权，再由一次性 connection ticket 升级；ticket 单次使用、短 TTL、绑定用户和目标工作区。
- 每条命令在业务副作用前校验 user、workspace ACL、capability、resource ownership 和 idempotency key。
- 断线、token 刷新、设备撤销与工作区 ACL 变更不会延长原授权；服务端主动终止已失效会话。

首版一个工作区只绑定一个可交互owner principal；多端同步是该owner的多设备同步，不是跨用户协作。管理员只能管理生命周期/配额，默认无项目内容或Workbench访问权。`editor/viewer`角色编号仅为后续协议保留且首版attach必须拒绝；未来开启前必须按`(user, workspace)`隔离Adapter、Extension Host、extension-state、SecretStorage与资源预算，不能让不同用户共享一个ExtHost。

## 4. 容器与 WSL2 基线

首版Linux与Windows/WSL2统一由Runtime Agent驱动Docker Engine，不依赖Docker Desktop；engine自身是否rootless由M1 Runtime Profile实测冻结，但engine socket始终只对最小特权Runtime Agent可见，Coordinator/Execution工作负载始终non-root。Podman/containerd仅保留后续driver。Windows只在指定WSL2 distribution内运行Linux双容器，不把Windows目录作为工作区根。

每个工作区由独立的Coordinator容器和Execution容器组成。两个容器默认策略：

- 非 root 用户；只读 rootfs；按机器可读mount matrix挂载workspace、authority、extension-state、IPC、tmp和tool-cache。
- 删除全部 Linux capabilities，按功能逐项增加；默认 `no-new-privileges`。
- seccomp、AppArmor/SELinux（可用时）、PID/CPU/内存/磁盘 inode/进程数限制。
- 禁止 privileged、host PID/network、Docker socket、宿主设备、任意 bind mount。
- cgroup v2 限额；OOM、磁盘满、fork bomb 必须转为稳定错误并可审计。硬磁盘/ inode quota 的底层（如XFS project quota或受限volume）必须在M1冻结并在WSL2实测，不能把Docker named volume当作天然有硬quota。
- 容器镜像按 digest 固定；运行时只接受已签名 allowlist 镜像和与之匹配的 Runtime Lock。
- Coordinator只主动建立到Control Plane的出站mTLS流，不开放工作区入站；Execution不对宿主/LAN发布端口。端口预览由Gateway建立受鉴权的临时反向代理。
- 网络策略必须在容器ready前以deny-all安装，Docker/WSL/network重启后先恢复deny再放行。显式阻断Docker/WSL/Windows host gateway、RFC1918、link-local/metadata（含169.254.169.254）、IPv6替代路径、未授权DNS/代理和DNS rebinding；按解析时与连接时IP复核。出站例外以域名/IP/端口/协议/TTL登记并审计。

Windows 不支持把 `C:\...` 作为项目工作区，也不向容器透传 Windows 凭据。项目只能导入容器 volume，并通过一致性导出产生可下载归档。

Windows 额外基线：

- `remote-ide-host.exe` 以专用、非交互 Windows service identity 运行；WSL distribution 必须由同一身份导入、启动、升级和卸载，并在空白服务身份下做安装测试。
- distribution 的 `/etc/wsl.conf` 禁用 DrvFs automount、Windows interop 与 Windows PATH 注入；项目、secret 和 runtime socket 不得经 `/mnt/*` 暴露。
- 除业务 Gateway 外所有 WSL 服务只监听 loopback/UDS。Windows Firewall 对 WSL vEthernet/NAT/mirrored 网络默认拒绝入站，测试必须证明不能绕过 Gateway 直达工作区端口。
- Host与WSL在ready前交换并验证签名Runtime manifest/version；不匹配时不启动工作区。
- VHD/distro升级采用新实例校验后切换并保留旧工件。普通软件卸载保留distribution/VHD数据；仅显式“删除全部数据”在备份提示后二次确认，且只删除与登记distribution id、VHD path、generation三者精确匹配的资源。未知VHD进入quarantine，绝不猜测删除。

### 4.1 双容器身份与mount matrix

- Coordinator容器只运行`workspace-agent`（`ide-agent` UID），独占`/ide/authority`（0700）document DB/journal/checkpoint；不包含Node、shell、编译器或项目依赖。
- Execution容器运行Headless Adapter、Extension Host、VSIX、terminal、LSP/DAP、Git、MCP和Agent工具。它能写`/workspace`和独立`/ide/extension-state`，但根本不挂载authority、Runtime Agent socket、控制面凭据或其他工作区volume。
- 两者只共享`/workspace`与instance-scoped IPC volume。每次runtime generation生成新socket/nonce/key；协议校验workspace/generation/hash。能可靠获得时再叠加`SO_PEERCRED`，不把容器UID映射当唯一认证。
- 独立PID/mount namespace使Execution不能signal、ptrace或读取Coordinator的`/proc`/FD。两个容器均drop capabilities/no-new-privileges且禁止setuid文件。

`runtime/mount-matrix.yaml`是事实源，每个mount登记：容器、source kind、target、owner UID/GID、ro/rw、exec/noexec、nodev/nosuid、quota、是否进入export、backup与删除策略。Runtime Agent按表生成实际spec并在ready前回读断言。

Node `fs`、终端和构建工具仍可直接写 `/workspace`。因此“唯一入口”只适用于会被服务端 ACK 的结构化编辑；原始文件系统不是可强制中介的单写 API。Watcher 发现外部写后必须按 inode/metadata/hash 复核：无 dirty 文档时生成新的 canonical external-change version；有 dirty 文档时进入显式冲突/重载流程，禁止静默覆盖。

## 5. 路径、文件与导出安全

所有公开文件标识使用规范化 workspace-relative URI。服务端必须：

1. UTF-8 解码、分隔符与 Unicode 规范化只执行一次。
2. 拒绝绝对路径、空段、`.`/`..`、NUL、设备名和超长路径。
3. 逐段以 `openat2`/等价安全方式解析，禁止 symlink、junction、mount escape 和 TOCTOU。
4. 检查最终 inode/mount 仍位于工作区允许根。
5. 不在 PublicError、日志或客户端消息泄漏宿主机绝对路径。

受控FileMutation与WorkspaceEdit中的create/overwrite/rename/delete必须在workspace mutation barrier中校验source/target的`must_not_exist`、exact metadata revision或content digest；WorkspaceEdit candidate digest覆盖这些前置条件，`ignore`/`overwrite`不能放宽它们。与open/dirty DocumentActor重叠时要求对应LeaseFence或显式冲突。递归大操作由OperationHost执行。布尔`overwrite`本身从不构成并发授权。

导出由容器外的 durable Export Coordinator（Control Plane 状态 + Runtime Agent 执行）拥有，不能由即将被冻结的 Workspace Agent 独自拥有。阶段为 `admitting → volume_locked → writers_frozen → reconciled → container_frozen → copying → published → unfreezing → lock_released → terminal`，每次转换持久化export id、workspace/runtime generation、fencing token、lock handle和deadline。

1. Export Coordinator先请求Runtime Agent取得generation/fencing-bound volume operation lock并持久记录handle；锁由Runtime Agent代表durable operation持有。只有取得并记录成功后，Control Plane才拒绝新 mutation、exec、terminal input、Agent tool 和扩展 UI command。
2. 持锁的Runtime Agent冻结整个Execution容器；Coordinator保持运行，排空 watcher、处理direct-write，保存已ACK文档、fsync authority DB并返回带tree metadata的prepare record。
3. Runtime Agent校验fencing token后冻结Coordinator容器，再启动签名digest锁定的临时Exporter Helper容器；它不属于工作区steady-state双容器。Helper无网络、无authority/extension-state/control socket，根文件系统只读、drop-all/no-new-privileges/seccomp，只以只读方式挂载精确workspace volume和有界scratch。Helper只继承已持锁operation的受限只读上下文，绝不拥有/释放lock；它执行卷级`syncfs`/等价flush，然后生成归档/manifest/hash。Runtime Agent自身不把workspace volume挂载到宿主路径读取。
4. 成功路径等待helper terminal并关闭FD后publish；失败路径在helper曾启动时cancel/wait/unmount，helper未启动时由Runtime Agent回读证明无helper container/mount/FD。随后先解冻Coordinator并按watermark对账，再解冻Execution，最后解除准入并凭对应`ExportCleanupEvidence`释放lock。

每个 pause token 有短 TTL，但 TTL 到期不直接假定成功；Runtime Agent watchdog 和 Control Plane reconciler依据operation、三个component label、helper、mount和真实pause状态幂等恢复。任何进程/WSL崩溃都不能留下永久冻结容器或可下载的半成品。同卷export/import/backup/migrate/delete/stop互斥。归档拒绝越界 symlink、socket、device、异常 sparse/hardlink、超过预算的文件/总量和压缩炸弹。

下载采用两步式：一次性exchange token换取短期download session；session绑定user、device、export、published artifact version、内容hash、长度和TTL，允许在有效期内发起多个并发/续传Range。每个Range请求重新校验ACL、device/session撤销、范围和artifact仍为published；ACL撤销立即使后续Range失败。默认24小时后删除；审计不包含归档内容。

Stream Snapshot与请求型Result Transfer在每个Begin/Chunk/End发送前复核session、device与资源授权；撤销或deadline触发typed Aborted并销毁缓冲。Result Transfer没有stream cursor/高水位，不能借候选diff伪造`SnapshotInstalled`或释放事件历史。

M0授权validator由能力矩阵和协议identity表状态化生成：先校验顶层frame/case，再校验嵌套action、request→response/outcome、`StreamKey`完整domain能力、资源Uuid等值与transfer/UI关联。DOCUMENT stream不按capability过滤事件，四项闭包缺一即拒绝订阅。Agent candidate读取/应用必须同时具备Agent、DocumentEdit和DocumentLease；Operation cancel即使位于baseline消息内也必须验证owner/kind权限、workspace generation、expected state revision与cancel-vs-commit仲裁。方案阶段的静态门只证明case闭合和Lease字段路径；三端运行时值比较须通过M0错配fixture后才能称为implemented。

## 6. 扩展、终端与 Agent

扩展是工作区内的不可信代码：

- 首版只安装服务端固定 `extensions.lock` 中的扩展；用户无独立插件管理器。
- 每个扩展记录来源、版本、VSIX hash、许可证、publisher 签名（若有）、native module ABI 和兼容等级。
- 通用`ExecuteCommand`只允许命中可信command policy registry并匹配policy digest；服务端动态合并capability/permission/Trust/Workbench要求。未知或扩展自报的未分类命令拒绝，命令不能替代WorkspaceEdit/Task/Terminal/Debug/SCM各自授权。
- 禁止 Webview、Custom Editor、任意 HTML/DOM/Electron API；调用返回 `extension_api_not_supported`，不能静默降级。
- Extension Host只继承Execution容器权限，不能看到Coordinator authority/PID、连接Runtime Agent/Control Plane socket或读取控制面secret。
- 扩展状态、secret storage、日志和缓存设独立配额；崩溃循环触发熔断。

终端默认需要 editor 权限和显式创建。输入采用单写租约，观察者只读；终端输出按字节预算截断并明确发出 gap/reset。禁止把 shell 控制序列解释为客户端 UI 命令。

Task、Debug与DAP evaluate可能执行项目代码：不可信工作区默认禁用，启用后仍要求相应permission、Workbench/debug控制租约、workspace/runtime generation和stop epoch。continue/step后旧frame/variable handle全部失效。Clipboard只路由到当前控制设备；read/write均受设备能力和用户会话授权约束，内容不进入durable event、日志或审计，后台调用被拒绝时返回稳定错误。

Agent 必须经过 Mutation Broker：

- 模型输出本身没有文件或 shell 权限。
- 每次工具调用包含结构化参数、目标、风险级别、deadline 和幂等键。
- 写文件、运行命令、网络访问、读取 secret、Git push 等分别授权；高风险操作要求用户批准。
- 批准绑定精确tool call、参数digest、workspace generation、每个受影响资源的`LeaseFence(target,generation,epoch)`和有效期；多文档不能共用裸epoch，任一参数/target/fence变化后旧批准失效。
- Secret 仅由broker在执行点通过每进程credential-helper、短期pipe/memfd或一次性回调注入；不放全容器env或共享tmpfs，永不进入prompt、事件流、diff、日志或导出manifest。
- Agent patch 与人工编辑走同一文档版本/租约/冲突通道，不能直接覆盖磁盘。

## 7. Secret 管理

- 控制面 master key 来自 OS key store/KMS，不存仓库、镜像或数据库同目录。
- 每用户/工作区使用 envelope encryption；密文携带 key id、算法、nonce、schema 和 AAD。
- 扩展 `SecretStorage` 只返回给同一 extension id；Agent provider key 只交给provider client。Execution内凭据endpoint绑定进程身份、operation、用途与TTL，执行完成即撤销并关闭FD。
- 日志、trace、crash dump、terminal replay、protocol fixture 有统一敏感字段过滤器和熵/格式扫描。
- Key rotation 支持双读单写；完成后清理旧密文并留审计，不静默丢失。

## 8. 审计与隐私

必须审计：登录/撤销、ACL 变化、工作区启动停止、镜像/扩展锁变化、租约转移、终端/任务启动、Agent 工具批准与执行、导出创建下载、策略拒绝和管理员动作。

Canonical audit event 至少包含 schema、event id、UTC 时间、actor、device、workspace、action、decision、target opaque id、policy version、request/operation id 和结果 code。禁止记录源码、prompt 全文、文件内容、token、环境变量或终端输出。高价值事件写 append-only sink，并使用 hash chain/远端不可变存储检测篡改。

遥测默认最小化；内容型遥测必须单独明确同意。用户删除流程需要覆盖控制面数据、工作区 volume、扩展状态、Agent checkpoint、导出和备份保留策略。

## 9. 供应链、源码公开与休眠发布体系

- Rust、Dart、npm、扩展、容器基础镜像全部使用 lockfile/digest；验证禁止未登记的浮动下载。
- 公开源码前必须扫描secret、用户数据、机器绝对路径、cache/大型工具链/构建工件，复核LICENSE/NOTICE/上游来源，并通过lock/generated drift与零Electron/Chromium门；该source-publication gate当前planned/non-green，不得公开当前树。
- CycloneDX/SPDX SBOM、完整许可证清单、构建provenance、签名与consumer verification属于未来制品发布资格；当前M8 inactive且不得分发制品，缺失不阻塞开发accepted。
- 若未来ADR恢复生产发布，只接受签名manifest；Runtime Agent在启动容器前仍校验镜像digest、Runtime Lock与允许的Code-OSS/Node/Adapter组合。
- Code-OSS 与 Microsoft 品牌、Marketplace 授权是不同问题。默认使用 Code-OSS 合法来源和允许分发的扩展，不能把 Marketplace 可见性等同于再分发权。
- 安全更新若改变协议或 schema，仍遵循生命周期；紧急阻断可撤销镜像/扩展，但必须留下兼容错误和恢复路径。

## 10. 备份、恢复与事故响应

- 控制面数据库使用加密、版本化备份；工作区 volume 的备份是显式产品策略，不把导出误当备份。
- 每季度（或每个重大存储版本）执行 restore drill，验证 ACL、hash、schema、事件 cursor 和 secret 可解密性。
- 可独立撤销 user session、device、extension、runtime image 与 workspace；撤销传播目标不超过 5 分钟。
- 事故模式可关闭新工作区、Agent 工具、网络出口或导出而保留只读访问。
- 证据保留按隐私政策；所有时间使用服务端 UTC，时钟漂移进入监控。

## 11. 必须通过的安全门禁

- 路径遍历、symlink race、zip/tar slip、压缩炸弹和恶意文件名测试。
- 协议 fuzz、长度/嵌套/repeated 预算、未知 oneof/major 和鉴权零副作用测试。
- 容器逃逸基线扫描、capability/seccomp/mount/network 策略断言。
- 越权矩阵：user/device/workspace/document/terminal/export/Agent approval。
- Secret 泄漏扫描覆盖日志、trace、crash、fixture、export 与客户端缓存。
- 扩展和项目脚本无法访问 Runtime Agent socket、控制面数据库或其他工作区 volume。
- 导出屏障在崩溃、磁盘满和断线下不会产生被标记为成功的不一致 artifact。
- 软件成分、许可证、签名与 provenance 完整，发布消费者会实际验证。

未通过上述门禁时，只能在受控开发环境运行，不能对外标记为生产就绪。
