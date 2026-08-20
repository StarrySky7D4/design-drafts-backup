# 完整项目编码方案

> 工作目录名：`VSCode_mobile`
> 内部代码命名：`remote_ide`（发布品牌待定）
> 状态：架构基线；M0-01/M0-02与G0-01至G0-06局部spike已实现，产品Runtime尚未实现
> 日期：2026-08-12

## 1. 项目目标

构建一个不依赖WebView的原生多端远程IDE：Flutter负责五端原生交互和代码呈现；Windows或Linux服务器负责项目、文档、扩展、语言服务、终端、调试、Git和Agent；服务端通过工作区容器隔离项目，并保留Code-OSS Node Extension Host以运行固定的VS Code扩展集合。

产品不是“把Electron移植到Flutter”，而是重新实现客户端Workbench，并在服务端维护一个兼容层。零 Electron/Chromium 是全生命周期合同：依赖图、CI、测试/开发工具、安装包、容器镜像、运行时、SBOM和发布工件均不得下载、安装、打包或启动 Electron/Chromium、Chromedriver、`@vscode/test-electron`、Playwright/Puppeteer、sidecar、WebView或隐藏浏览器；只允许纯 Node.js Extension Host、最小同 commit 源码闭包和经审计的 Headless Adapter。完整上游源码只能隔离只读提取；若最小构建强制浏览器依赖，G0 No-Go。

```text
Flutter Native Workbench
  ↕ stable product protocol
Rust control plane + workspace state
  ↕ stable internal protocol
Headless Code-OSS MainThread Adapter
  ↕ pinned Code-OSS internal protocol
Code-OSS Extension Host + fixed extensions
```

成功标准：

- Flutter五端共享业务与编辑器核心，分别适配触控、键鼠、窗口和系统能力。
- 100ms RTT下输入仍即时显示，服务端ACK后形成可恢复权威版本。
- 已ACK编辑在客户端、服务、容器或WSL2重启后不丢失。
- 固定语言扩展可以提供补全、诊断、跳转、格式化和Code Action。
- 多设备可以共享工作区、Agent、终端和任务状态，并明确区分共享状态与设备状态。
- Windows服务器无需Windows工作区，统一以WSL2 Linux容器运行；项目可安全导出。

## 2. 冻结决策与实施假设

### 2.1 已冻结

1. Flutter不使用WebView。
2. Windows服务端使用WSL2作为Linux运行时；Linux服务端直接使用相同运行时。
3. 项目写入、插件和开发工具全部在服务端。
4. 保留Code-OSS Node Extension Host。
5. 工作区运行在Linux容器内。
6. 暂不开发插件管理器或用户插件SDK。
7. 暂不支持Windows工作区。
8. 支持一致性项目导出。
9. Flutter覆盖Android、iOS、Windows、macOS、Linux；首版不含Web。
10. 支持服务端驱动的多端状态同步。

### 2.2 首版技术裁定

- 采用“模块化单体 `ide-server` + 最小特权 `runtime-agent` + 每工作区 `workspace-agent`”，不从微服务起步。
- 每工作区采用Coordinator/Execution双容器：前者只运行Rust Workspace Agent并独占authority；后者运行Adapter、Extension Host、PTY和工具，两者只共享项目卷和instance IPC。
- 公共实时协议使用TLS/WSS上的Protobuf二进制帧；大文件、上传和导出使用HTTPS流。
- 控制面元数据第一版使用SQLite WAL；接口保留PostgreSQL实现空间。
- 工作区文件使用独立持久卷；源代码不映射到Windows路径。
- OCI运行时首版在Linux与WSL2统一使用Docker Engine API，不依赖Docker Desktop；engine socket只对Runtime Agent可见，工作负载始终non-root。是否使用rootless engine、磁盘硬quota与cgroup freezer必须在M1 Spike后冻结为一个Runtime Profile；Podman/containerd仅作为后续driver，不进入首版验收矩阵。
- Flutter不引入本地Rust/FRB主链路；远程协议直接生成Dart类型。只有性能证据证明必要时才增加本地Rust库。
- 首版每个工作区只允许owner这一名交互principal，并只有一个逻辑Workbench Session和Extension Host；该owner可从一个控制设备和多个观察设备接入。跨用户editor/viewer协作不在首版，避免共享ExtHost的Memento、SecretStorage和窗口上下文串用户。
- 同一文档一个写入租约；不在首版引入CRDT。
- 扩展集合由镜像构建期的`extensions.lock.json`锁定，运行时只读。

## 3. 从ModCptLib继承的工程方法

采用：

- Rust作为业务事实源，Flutter通过命令、版本化快照和结构化事件交互；ModCptLib现有event仍有字符串kind，不把它描述为完整typed stream。
- 桥接/防腐层保持薄，不承载领域状态；本项目对应Headless Adapter。
- 所有channel、任务、资源和输入有硬上限，网络/磁盘I/O在锁外。
- 强类型业务ID、稳定公共错误、operation ID、deadline、cancel、cleanup后唯一终态。
- 协议注册表、协议生命周期、N/N-1/N+1兼容要求和生成物漂移Gate；ModCptLib当前first-stable bootstrap没有真实N-1工件证据。
- `Knowledge/`权威文档、`AGENTS.md`工作边界、统一quick/full验证入口。
- 工件hash、SBOM、provenance、签名和consumer-side验证。

不采用：

- ModCptLib的P2P/QUIC业务拓扑和身份模型。
- Flutter与本地Rust之间的FRB主链路。
- Cap'n Proto、JSON RPC和自定义wire混用；本项目主边界统一Protobuf。
- 其Windows CMake/FRB DLL构建和调试TLS跳过逻辑。

详见`MODCPTLIB_REFERENCE.md`。

## 4. 技术栈与工具链

版本不在本文硬编码；仓库初始化时固定到可重复的精确版本并提交所有lockfile。

### 4.1 Rust

计划组件：

| 能力 | 基线选择 |
|---|---|
| 异步运行时 | Tokio |
| HTTP/WSS | Axum + Tower + rustls |
| Protobuf | prost；内部需要RPC语义时由tonic生成 |
| 数据库 | sqlx SQLite；后续PostgreSQL |
| 文档Rope | ropey或经基准验证的等价实现 |
| 容器API | Docker-compatible client（首选bollard） |
| 文件watch | notify + 内容hash复核 |
| Git | 结构化参数调用容器内固定Git CLI，解析porcelain v2 |
| PTY | Linux PTY封装；实现前对portable-pty/rustix做Spike |
| 压缩/归档 | zstd、tar、zip；全部流式有界 |
| ID/时间 | UUIDv7用于业务ID、UUIDv4用于operation/request；服务端UTC |
| 错误 | thiserror内部错误 + 稳定PublicError投影 |
| 观测 | tracing + OpenTelemetry + Prometheus低基数指标 |

使用`rust-toolchain.toml`固定工具链；所有crate默认`#![forbid(unsafe_code)]`，确需unsafe的平台crate必须单独ADR、封装并接受Miri/审计。

### 4.2 Flutter

| 能力 | 基线选择 |
|---|---|
| UI | Flutter stable，五端原生runner |
| 状态作用域 | Riverpod式作用域容器；禁止进程级万能单例 |
| 导航 | 声明式路由 |
| 网络 | 生成Protobuf + WSS二进制通道 + HTTPS客户端 |
| 凭据 | 平台Keychain/Keystore安全存储 |
| 编辑器 | 自定义RenderObject/TextInputClient，不基于TextField |
| 测试 | flutter_test、integration_test、golden、平台真机测试 |

`pubspec.lock`必须提交。平台插件应采用统一接口和federated实现，避免UI层出现大量平台判断。

### 4.3 TypeScript/Code-OSS

- Code-OSS固定完整commit，不追上游浮动分支。
- 使用上游要求的精确Node和包管理工具，由Corepack或等价方式锁定。
- Adapter启用TypeScript strict；上游内部类型只能出现在`code-oss/headless-adapter`。
- 生成Extension Host actor/contribution inventory和protocol hash，升级漂移时CI失败。

### 4.4 协议与供应链

- Protobuf源和生成结果同时提交；Buf lint/breaking检查。
- Rust、Dart、Node、容器镜像分别生成SBOM。
- 所有外部扩展保存来源、version、SHA-256、license和认证结果。
- 若使用GitHub Actions或其他托管CI插件，必须固定完整commit digest；它们仅是可选镜像，验收和发布证据必须可由本机Windows与自管Linux独立产生，不得依赖远程CI可用性。

## 5. 运行进程

### 5.1 Linux主机

```text
systemd
├── docker.service
├── remote-ide-runtime-agent.service     最小特权
├── remote-ide-server.service            非特权
├── remote-ide-reconcile.service/timer
├── remote-ide-export-gc.timer
└── remote-ide-backup.timer
```

### 5.2 Windows主机

```text
Windows Service Manager
└── remote-ide-host.exe
    ├── 启动固定名称的专用WSL2发行版
    ├── 维护Windows稳定监听端口到WSL2的字节转发
    ├── 健康检查、日志和签名升级
    └── 不终止业务TLS、不接触工作区内容

WSL2 + systemd
├── remote-ide-server
├── remote-ide-runtime-agent
├── Docker Engine
└── Linux workspace containers
```

发行版必须由运行Windows Service的专用身份导入、枚举和拥有，并以该真实service SID在空白机器测试，避免WSL按用户注册导致服务不可见。`/etc/wsl.conf`禁用DrvFs automount、Windows interop和Windows PATH注入；数据留在WSL2 ext4 VHD，不放在`/mnt/c`，不依赖Docker Desktop。Host与WSL在ready前交换签名Runtime manifest/version；Windows Firewall只开放Host稳定代理端口并覆盖NAT/mirrored网络，WSL动态IP不得对LAN直达。普通卸载保留distribution/VHD数据；只有显式“删除全部数据”流程在备份提示和精确ID/路径/generation复核后才unregister。升级/compact失败保留旧VHD和签名工件。

### 5.3 每工作区双容器

```text
coordinator container (trusted, non-root)
└── workspace-agent (Rust; authority DB/document journal)

execution container (untrusted, non-root)
└── headless-workbench (Node/TypeScript)
    └── extensionHostProcess (Code-OSS Node)
        └── fixed extension / PTY / LSP / DAP / Git / MCP processes
```

两容器只共享项目卷与instance-scoped IPC volume；authority volume只挂载到Coordinator。Workspace Agent负责文档事实、watch归并、导出prepare、Agent/PTY/Git等操作编排和Execution监管；实际项目命令、PTY、Git、语言工具与扩展在Execution执行。适配器只负责Code-OSS MainThread语义转换。Runtime Agent能独立pause/recreate Execution，不依赖单容器内动态UID/cgroup切换。

## 6. Rust控制面编码

### 6.1 Crate职责

| Crate | 职责 |
|---|---|
| `remote_ide_ids` | 强类型User/Device/Workspace/Client/Document/Operation ID |
| `remote_ide_protocol` | 生成Protobuf、帧验证、capability握手 |
| `remote_ide_error` | 稳定错误枚举和安全投影 |
| `remote_ide_config` | 严格版本化配置、未知字段策略 |
| `remote_ide_auth` | OIDC PKCE、设备challenge、workspace ACL |
| `remote_ide_store` | schema、迁移、transaction和备份 |
| `remote_ide_event_log` | 工作区事件、快照、压缩和重放 |
| `remote_ide_state_sync` | 多流游标、背压、租约和UI路由 |
| `remote_ide_operation` | start/status/cancel/result和资源清理 |
| `remote_ide_workspace` | desired/observed状态机和reconciler |
| `remote_ide_export` | 容器外durable ExportCoordinator、volume lock、artifact状态 |
| `remote_ide_artifact` | 本地/远端artifact原子publish与download session |
| `remote_ide_runtime_api` | 狭窄特权RPC和Runtime trait |
| `remote_ide_runtime_docker` | Docker-compatible实现，只链接到runtime-agent |
| `remote_ide_gateway` | HTTP/WSS、限流、下载、公开错误 |

### 6.2 Runtime trait

```rust
#[async_trait::async_trait]
pub trait WorkspaceRuntime: Send + Sync {
    async fn ensure_volume(
        &self,
        spec: VolumeSpec,
        op: &OperationContext,
    ) -> Result<VolumeHandle, RuntimeError>;

    async fn ensure_workspace_runtime(
        &self,
        // Contains distinct Coordinator/Execution image, mount, identity,
        // network and readiness specifications.
        spec: WorkspaceRuntimeSpec,
        op: &OperationContext,
    ) -> Result<WorkspaceRuntimeInstance, RuntimeError>;

    async fn inspect(
        &self,
        workspace: WorkspaceId,
    ) -> Result<Option<RuntimeObservation>, RuntimeError>;

    async fn start(
        &self,
        workspace: WorkspaceId,
        expected_generation: u64,
        op: &OperationContext,
    ) -> Result<WorkspaceRuntimeInstance, RuntimeError>;

    async fn stop(
        &self,
        workspace: WorkspaceId,
        expected_generation: u64,
        mode: StopMode,
        op: &OperationContext,
    ) -> Result<(), RuntimeError>;

    async fn ensure_paused(
        &self,
        workspace: WorkspaceId,
        component: WorkspaceComponent,
        expected_generation: u64,
        pause_id: PauseId,
        reason: PauseReason,
        op: &OperationContext,
    ) -> Result<PersistentPauseObservation, RuntimeError>;

    async fn ensure_resumed(
        &self,
        workspace: WorkspaceId,
        component: WorkspaceComponent,
        expected_generation: u64,
        pause_id: PauseId,
        op: &OperationContext,
    ) -> Result<PersistentPauseObservation, RuntimeError>;

    async fn destroy(
        &self,
        workspace: WorkspaceId,
        expected_generation: u64,
        registered_runtime_identity: RuntimeIdentity,
        deletion_fencing_token: DeletionFencingToken,
        op: &OperationContext,
    ) -> Result<(), RuntimeError>;
}
```

`WorkspaceRuntime`只负责steady-state工作区生命周期，不是完整的导出边界。临时Exporter Helper和volume lock由独立窄接口管理：

```rust
#[async_trait::async_trait]
pub trait ExportRuntime: Send + Sync {
    async fn acquire_volume_operation_lock(
        &self,
        workspace: WorkspaceId,
        expected_generation: u64,
        export_id: ExportId,
        fencing_token: FencingToken,
        op: &OperationContext,
    ) -> Result<VolumeOperationLock, RuntimeError>;

    async fn ensure_export_helper(
        &self,
        spec: ExportHelperSpec,
        lock: &VolumeOperationLock,
        op: &OperationContext,
    ) -> Result<ExportHelperObservation, RuntimeError>;

    async fn inspect_export_helper(
        &self,
        export_id: ExportId,
    ) -> Result<Option<ExportHelperObservation>, RuntimeError>;

    async fn cancel_export_helper(
        &self,
        export_id: ExportId,
        fencing_token: FencingToken,
        op: &OperationContext,
    ) -> Result<(), RuntimeError>;

    async fn wait_export_helper_terminal_and_mounts_closed(
        &self,
        export_id: ExportId,
        deadline: Deadline,
    ) -> Result<ExportHelperTerminal, RuntimeError>;

    async fn release_volume_operation_lock(
        &self,
        lock: VolumeOperationLock,
        cleanup: ExportCleanupEvidence,
        op: &OperationContext,
    ) -> Result<(), RuntimeError>;
}
```

`ExportHelperSpec`只接受签名release中的helper digest、精确只读workspace volume、有界scratch和预定义归档策略；接口不暴露宿主路径、任意mount参数或Docker JSON。`ExportCleanupEvidence`是穷尽enum：`HelperNeverStarted { verified_no_helper_container, verified_no_helper_mount_or_fd }`或`HelperTerminalAndMountsClosed(ExportHelperTerminal)`。Runtime Agent必须回读实际component label/mount/FD再生成证据；拿锁后但helper启动前失败走前一分支，helper曾启动走后一分支。`release_volume_operation_lock`不能接受调用方布尔断言，证明失败时由reconciler继续持锁收敛，不能先解冻writer。

Runtime Agent RPC不接收任意Docker JSON或Shell字符串。每个请求（包括stop/destroy）包含已验证的WorkspaceId、expected generation、预定义操作、资源上限和idempotency key；destroy还需匹配已登记Runtime/volume identity与deletion fencing token，未知或漂移资源只隔离审计、不删除。pause是可持久对账的显式`pause_id`，不是只依赖进程内RAII guard；Runtime Agent和Control Plane重启后都能从容器label/operation记录收敛。

### 6.3 Workspace状态

```rust
pub struct WorkspaceRecord {
    pub id: WorkspaceId,
    pub owner_id: UserId,
    pub desired_state: DesiredState,
    pub observed_state: ObservedState,
    pub runtime_generation: u64,
    pub coordinator_image_digest: ImageDigest,
    pub execution_image_digest: ImageDigest,
    pub workspace_volume_id: VolumeId,
    pub authority_volume_id: VolumeId,
    pub extension_state_volume_id: VolumeId,
    pub limits: ResourceLimits,
    pub network_policy: NetworkPolicyId,
    pub runtime_release: RuntimeReleaseId,
    pub last_error: Option<PublicErrorCode>,
}
```

Reconciler规则：

- desired running而容器缺失：创建并启动。
- 容器generation过期：停止并隔离。
- 数据库无记录的容器：标记orphan，宽限后清理，绝不立即删除未知卷。
- 容器ready但Workspace Agent/Extension Host未握手：degraded并有界重启。
- 连续崩溃：指数退避并进入failed，防止无限循环。

## 7. Workspace Agent编码

### 7.1 模块

```text
workspace_agent
├── document_actor
├── mutation_broker
├── filesystem_service
├── file_watcher
├── terminal_service
├── git_service
├── export_service
├── agent_orchestrator
├── extension_supervisor
├── adapter_port
├── checkpoint_service
└── workspace_health
```

每个工作区Agent由一个顶层`CancellationToken`和`JoinSet`拥有所有子任务。关闭时停止准入、取消子任务、等待清理、验证资源计数后退出；不能依赖进程被kill完成正常语义。

### 7.2 DocumentActor

```rust
pub struct EditBatch {
    pub document_id: DocumentId,
    pub client_instance_id: ClientInstanceId,
    pub client_sequence: u64,
    pub base_version: u64,
    pub lease_epoch: u64,
    pub edits: Vec<TextEdit>,
}

pub struct TextEdit {
    pub start: Utf16Position,
    pub end: Utf16Position,
    pub new_text: String,
}
```

处理顺序不可改变：

1. 校验会话、ACL、租约epoch、base version、数量和字节上限。
2. 用服务端认证上下文绑定的`(user_id, device_id, client_instance_id, workspace_id, document_id, client_sequence)`查幂等结果；客户端自报ID不能跨principal取回旧Ack。
3. 对候选Rope应用编辑，不先改变权威内存。
4. 在事务中追加durable journal和新version。
5. 提交成功后替换内存Rope并ACK。
6. 广播文档事件，更新Adapter副本。
7. 有界debounce后原子保存文件并推进`saved_version`。

ACK只能在journal事务成功后返回。恢复时从最后文件/文档快照重放后续日志；快照hash、version和文件saved version全部验证后才压缩旧日志。

位置统一使用0-based UTF-16 code unit。Dart字符串天然使用UTF-16索引，但光标和删除操作仍需按grapheme cluster约束，避免拆分组合字符；Rust维护UTF-8 byte与UTF-16位置索引。

### 7.3 Mutation Broker

所有标准写入来源必须串行进入：

```text
Flutter EditBatch
Extension WorkspaceEdit
Agent CandidateEdit
Format/CodeAction
Save Participant
Git checkout/merge（需要工作区屏障）
```

首版每个Flutter文档只允许一个in-flight batch；后续按顺序排队并乐观显示。Extension/Agent编辑在活动客户端队列flush后建立mutation barrier，再应用并广播，避免在首版实现复杂OT。无法拦截的Node `fs`或子进程写入由watcher转成external change/conflict。

Completion、Rename、Formatting、Code Action与扩展Bulk Edit首先只生成绑定principal/device/workspace/runtime/extension generation、base或文件digest、TTL及完整edit digest的不可变candidate。create/rename/delete的source/target `FilePrecondition`也是candidate正文和digest的一部分，`overwrite`/`ignore`选项不能放宽candidate生成时的精确状态。大candidate经无cursor的`ResultTransfer`传输供Diff，绝不伪装成stream snapshot；客户端显式发送带逐资源`LeaseFence`的`ApplyWorkspaceEdit`后才进入Broker。请求语言结果本身没有写副作用；过期、任一前置条件或任何绑定变化都拒绝。

### 7.4 保存

保存使用独立Operation：调用Adapter的will-save participants，限制参与者数量、总编辑量和总超时；Rust再次校验版本、应用编辑、写临时文件、fsync并原子替换，再触发did-save。插件超时不能无限阻塞保存。

Save/Revert/SetLanguage除expected document version外必须携带`expected_lease_epoch`，防止客户端失去租约后又以新epoch重获时，旧排队命令发生ABA。SaveOperation在准入和最终commit两次校验相同workspace generation、document ID、lease epoch和version；任一变化无副作用失败。

### 7.5 文件路径

公开API只接受强类型`WorkspaceRelativePath`：

```rust
pub struct WorkspaceRelativePath(String);
```

构造时拒绝绝对路径、NUL、`..`逃逸和平台不一致分隔符；实际打开前以`openat2`/等价安全遍历验证解析目标仍在工作区。安全判断不能只用字符串`canonicalize`，因为存在TOCTOU和symlink切换。

Create/rename/delete等FileMutation在workspace mutation barrier下执行；创建必须声明`must_not_exist=true`或覆盖目标的exact revision/digest，rename校验source和target，delete校验target。与打开/dirty文档重叠时转交DocumentActor并要求对应LeaseFence，无法证明无竞态则返回显式冲突。递归或超预算操作升级为typed Operation；旧文件树客户端不得凭`overwrite=true`覆盖并发新内容。

## 8. Flutter客户端编码

### 8.1 分层

```text
Widgets / Screens
  -> Feature Controllers (workspace/editor/terminal/agent)
  -> Repositories / Commands / State projections
  -> SessionClient + StateSyncClient
  -> Generated Protobuf transport
```

UI不能直接调用WebSocket或生成DTO；生成层外必须有手写防腐适配器和领域ID类型。每个workspace会话拥有独立状态容器，支持桌面多窗口而不污染其他会话。

### 8.2 原生编辑器组件

```text
RemoteCodeEditor
├── DocumentReplica             服务端文档的可丢弃副本
├── PendingEditQueue            有界未ACK操作
├── LineIndex                   UTF-16/行/字节映射
├── ViewportController          可见区与预取
├── EditorRenderBox             可视行虚拟化
├── TextInputController         TextInputClient/IME
├── CursorSelectionController   光标、选择、多光标后置
├── DecorationLayer             诊断、Git、搜索、inline hints
├── CompletionOverlay
├── HoverAndCodeActionOverlay
├── ShortcutController          手机快捷条+桌面Actions/Shortcuts
└── AccessibilityBridge
```

实现规则：

- 不为每行或每token创建长期Widget树；使用自定义RenderObject只布局可视行及小缓冲。
- 文本塑形使用Flutter Paragraph/TextPainter并缓存行布局；字体、字号、tab size、wrap width变化使相关缓存失效。
- 光标、selection、诊断线、搜索结果和remote change分别绘制，避免重排正文。
- 第一版只支持等宽主字体，fallback必须稳定；主题来自服务端解析的VS Code theme快照。
- 标准可编辑文件计划上限32MiB；超过后进入只读大文件查看模式。具体限制在M3编辑器容量基准与M5语言服务基准后冻结。
- 单次普通EditBatch计划不超过256项/256KiB插入；超大paste走有界上传后由服务端形成一次候选编辑。

### 8.3 输入与IME

- 实现`TextInputClient`组合态，不把每次composition update立即持久化为最终编辑。
- 组合提交时生成一个原子EditBatch；候选栏变化只影响本地composing range。
- 中文、日文、韩文、emoji、ZWJ、variation selector、组合音标、CRLF必须进入固定测试语料。
- 移动端提供Esc/Ctrl/Alt/Tab/方向键快捷条；桌面使用Actions/Shortcuts并支持用户映射。
- 浏览器保留快捷键问题不适用，但各桌面OS系统快捷键冲突必须建立平台矩阵。

### 8.4 乐观编辑

```text
confirmed replica version N
 + in-flight batch A
 + pending batches B..K
 = 当前渲染副本
```

- 每帧或短debounce合并输入，只有一个batch在网络中飞行。
- ACK后从队列移除并推进confirmed version。
- 重复ACK按client sequence幂等处理。
- version conflict、租约丢失或snapshot reset时，冻结输入，取得权威快照并尝试重放尚未确认的本地操作；无法安全重放则展示明确Diff，不静默丢弃。
- pending队列按编辑数和总字节有硬上限；达到阈值转只读并提示连接问题。

### 8.5 自适应布局

| 形态 | 布局 |
|---|---|
| 手机竖屏 | 单主面板，底部导航，文件/Agent/终端全屏切换 |
| 手机横屏/小平板 | 编辑器 + 抽屉或可切换辅助面板 |
| 大平板 | 编辑器 + 固定辅助栏 |
| 桌面 | Activity栏、侧栏、编辑区、底部面板，可多窗口 |

共享状态不等于共享布局。光标、滚动、面板宽度和窗口位置只在设备本地。

## 9. 公共协议与传输

### 9.1 通道

- `WSS /api/v1/session`：实时命令、结果、事件、游标、租约和小块数据。
- HTTPS：OIDC回调、工作区管理、归档上传、文档大快照、项目导出和Range下载。
- Control Plane ↔ Runtime Agent：两者使用独立service UID；Runtime Agent拥有的systemd UDS为专用group `0660`，该group只包含`ide-server`身份，并以`SO_PEERCRED`精确allowlist Control Plane UID。消息还校验internal major、启动nonce、workspace/runtime generation与幂等键；其他宿主UID和所有容器身份均拒绝。
- Control Plane ↔ Workspace Agent：由Coordinator主动建立的出站mTLS控制流；证书绑定workspace/generation，Coordinator不监听可被局域网或其他工作区访问的端口。
- Workspace Agent ↔ Headless Adapter：共享instance IPC volume上的UDS + length-prefixed Protobuf；每generation challenge key与协议hash认证。
- Adapter ↔ Extension Host：固定Code-OSS内部父子IPC，不对网络开放。

公共协议不采用“尽力解析”。major未知或必需capability缺失时，在建立工作区副作用前拒绝。minor只能添加安全可忽略的optional字段/消息。

`capability-matrix.yaml`不是简单的顶层case列表：它机器登记每个嵌套command/action的effect guard、每个请求允许的response/outcome/result kind、`StreamKey`对应的完整domain capability、event/snapshot分支身份、UI subtype闭包以及Snapshot/Result transfer状态机；`PROTOCOL_REGISTRY.md`的identity表补充资源Uuid逐值等式。M0生成validator逐层fail-closed，不能因为两个消息“需要同一capability”就允许错误响应或绕过子命令授权。交付包静态脚本只证明分支与Lease字段路径闭合，不替代三端运行时identity实现。

首版DOCUMENT stream将正文、write lease、diagnostics和semantic tokens视为同一不可过滤恢复闭包，必须同时协商`STATE_RESUME + DOCUMENT_EDIT + DOCUMENT_LEASE + LANGUAGE_UI`。M2/M3可以通过内部fixture独立开发，但M4四项producer/consumer和恢复测试完成前不得对产品客户端advertise部分document stream；将来若要细粒度协商，先拆分StreamKey/cursor/snapshot。Workspace/Terminal/Debug中嵌入的租约状态同样继承Lease capability。

### 9.2 握手

```text
ClientHello
├── protocol major/minor
├── required/optional capability repeated enums
├── platform/build
├── device ID + server challenge signature
├── client instance ID
├── requested workspace
└── stream cursors

ServerHello
├── selected version/capabilities
├── session ID
├── runtime release
├── heartbeat policy
├── permission snapshot
└── per-stream resume plans
```

Resume plan只能为：`replay`、`snapshot_required`、`stream_unavailable`或`permission_denied`。客户端不得猜测丢失区间。

### 9.3 帧与背压

客户端与服务端使用方向分离的`ClientFrame`/`ServerFrame`。每帧再通过`delivery` oneof严格分为短请求/响应、durable event、stream snapshot、request result transfer、服务端UI请求或control；Flutter的Future、stream reducer、snapshot installer和result assembler互不消费彼此的帧。短请求的多块响应有连续`part_index`且恰有一个terminal marker。Hello后每个ClientFrame携带`expected_workspace_generation`；effectful `CommandMeta`重复携带并必须一致，旧generation排队命令在任何副作用前拒绝。请求相关性用`request_id`，有副作用命令另带`idempotency_key`；创建终端、Agent、导出、调试、测试和Task返回typed resource ID。多流确认用repeated typed `StreamAck(key,generation,acknowledged_cursor)`，其中cursor就是该stream的独立event sequence，不能用单个全局ack。短请求取消、长Operation取消和服务端UI请求取消使用不同消息。计划初始限制：

每个`ClientRequest`统一携带`RequestMeta(request_id, deadline)`；响应报告observed workspace generation。启动stream snapshot或请求大结果transfer的`ServerResponse`本身就是该请求唯一terminal，后续分别只按`snapshot_id`或`transfer_id`路由。前者绑定StreamKey/generation/high-water并需要`SnapshotInstalled`，后者绝不带cursor或推进恢复frontier。短请求取消是control frame，原请求仍负责发出唯一terminal；长工作统一由OperationHost取消。

`CancelOperation`携带`CommandMeta`和`expected_operation_revision`，并按operation owner/operation kind权限重新鉴权；它与成功commit进入同一持久仲裁，重试只返回首次赢家。把`OperationCommand`列为baseline只表示消息可解析，不赋予取消权限。

`ExecuteCommand`只允许调用可信command policy registry中的条目。registry由固定扩展manifest、项目内批准策略和管理员配置构建并签入Runtime Release派生数据，不能信任扩展运行时自报；每项记录动态required capabilities/permissions、Workspace Trust、Workbench控制租约和effect类别。Flutter回传`expected_policy_digest`；策略要求Workbench控制时还必须回传精确`expected_workbench_control_lease_epoch`防ABA。服务端重新求并集并在调用Extension Host前校验，未知或策略漂移命令拒绝。命令随后产生的WorkspaceEdit、Task、Terminal、Debug和SCM操作仍必须经过各自Broker，不能借通用command跳过窄权限。

| 项目 | 计划默认 | 处理 |
|---|---:|---|
| 单WSS frame | 1MiB | 超限拒绝 |
| EditBatch插入总量 | 256KiB | 超大paste走上传 |
| 单batch edits | 256 | 超限拒绝 |
| Snapshot chunk | 256KiB | 分块+hash |
| Terminal chunk | 64KiB | 分块 |
| 客户端待发送缓冲 | 8MiB | reset后断开慢客户端 |
| UI请求挂起数 | 64/客户端 | resource_exhausted |
| workspace event replay | 10,000条或24h | 过期走snapshot |

这些数值在容量测试后登记为frozen；协议字段不写死服务器策略值。

禁止在主事件流发送完整大项目、无限终端输出、任意插件对象或Code-OSS内部DTO。

### 9.4 稳定公共错误

```rust
pub struct PublicErrorV1 {
    pub code: PublicErrorCode,
    pub category: ErrorCategory,
    pub retryable: bool,
    pub retry_after_ms: Option<u64>,
    pub operation_id: Option<OperationId>,
    pub safe_details: BTreeMap<String, String>,
}
```

基线code：

```text
invalid_argument, unauthenticated, permission_denied, not_found,
conflict, revision_conflict, write_lease_held, failed_precondition,
workspace_starting, workspace_unavailable, extension_host_unavailable,
extension_api_not_supported, resource_exhausted, deadline_exceeded,
operation_cancelled, unsupported_feature, incompatible_version,
export_too_large, data_corrupted, internal
```

Flutter根据code本地化。公共错误不得包含SQL、容器ID、宿主路径、命令、token、用户代码、堆栈或任意内部错误链。

协议草案见`protocol/ide/v1/ide.proto`，治理见协议注册表与生命周期文档。

## 10. 数据库与持久化

### 10.1 控制面SQLite

第一版单主机使用SQLite WAL和`sqlx`。关键状态写入使用强同步策略；数据库只由`ide-server`打开。未来多节点迁移到PostgreSQL时通过trait替换，不让SQL类型泄漏到领域层。

主要表：

| 表 | 关键字段与用途 |
|---|---|
| `users` | OIDC subject、状态、创建时间 |
| `devices` | user、public key、platform、revoked、last_seen |
| `client_sessions` | device、workspace、protocol、heartbeat、control lease |
| `workspaces` | owner、desired/observed、generation、image、volume、limits |
| `runtime_instances` | generation、container、health、exit、timestamps |
| `workspace_events` | workspace、sequence、schema、kind、payload、time |
| `workspace_snapshots` | workspace、sequence、payload、sha256 |
| `user_workspace_state` | user/workspace/key/revision/payload |
| `operations` | ID、kind、state、deadline、generation、terminal error |
| `export_jobs` | durable phase、generation、pause/fencing、format、terminal |
| `volume_operations` | workspace/volume互斥owner、generation、deadline、recovery |
| `artifacts` | store key、published version、size、sha256、state、expiry |
| `download_sessions` | exchange/session hash、user/device/export、TTL、revoked |
| `audit_events` | append sequence、actor/action/result、previous hash/hash |
| `schema_metadata` | schema、runtime release、migration状态 |

文档正文和细粒度journal保存在Coordinator私有authority volume，避免控制面数据库成为高频编辑热点，也不让不可信Execution看到/修改权威日志；控制面只保留恢复路由checkpoint和流游标。authority数据库由Workspace Agent单写。

### 10.2 文档存储

```text
shared workspace volume:
  /workspace/<files>                    两容器可见用户文件
coordinator-only authority volume:
  /ide/authority/document.db            version/journal/snapshot/lease
  /ide/authority/task-checkpoints/      Agent/操作恢复
execution-only extension-state volume:
  /ide/extension-state/                 Memento/global/workspace storage
```

`DocumentJournal`接口：

```rust
#[async_trait::async_trait]
pub trait DocumentJournal: Send + Sync {
    async fn append_edit(&self, edit: AcceptedEdit)
        -> Result<DurableEditAck, StoreError>;
    async fn find_duplicate(&self, key: BoundMutationKey)
        -> Result<Option<DurableEditAck>, StoreError>;
    async fn load_after(&self, document: DocumentId, version: u64)
        -> Result<Vec<AcceptedEdit>, StoreError>;
    async fn save_snapshot(&self, snapshot: DocumentSnapshot)
        -> Result<(), StoreError>;
}
```

快照只有在内容hash、version、文件saved version和数据库commit全部一致时才可用于压缩旧日志。磁盘满或fsync失败时停止新写并进入只读降级，绝不发送虚假ACK。

### 10.3 Schema迁移

- `user_version`/schema只前进；validation、转换和version写回同一transaction。
- 每次迁移提供真实旧fixture、合法/非法、重复open、并发、故障中断、恢复和N/N+1拒绝测试。
- 已发布migration不改写；缺陷用新forward-repair或quarantine步骤。
- 不支持安全二进制回滚的迁移在升级前强制备份，并阻止旧binary回写高版本库。
- Runtime Release manifest声明可读/可写schema窗口。

## 11. 多端状态同步

### 11.1 状态作用域

工作区共享：文件/文档、诊断、Git、终端、调试、测试、Agent、Extension Host贡献和工作区配置。

用户工作区：最近文件、固定标签、断点、搜索/命令历史、Agent会话索引和用户设置。

设备本地：窗口、布局、滚动、光标、选择、键盘、缩放和面板尺寸。

设备状态默认不上行，避免手机改变桌面端上下文。插件“窗口”语义由当前Workbench控制租约持有者提供。

### 11.2 多流游标

```text
workspace control  -> independent event_sequence
document           -> independent event_sequence; versions stay in payload
terminal           -> independent event_sequence; raw byte offset stays in payload
agent task         -> independent event_sequence
extension state    -> extension_generation + independent event_sequence
```

客户端重连：在保留窗口内重放；早于压缩点则获取快照；流已终止或无权限则明确返回。服务端先登记live tail再在Actor barrier获取`generation + event high_watermark`快照，只发送严格大于cut的事件。统一Snapshot envelope将typed `SnapshotPayload`按`snapshot_id + part_index`传输，Begin声明schema/大小/part count/hash，End重复完整hash；digest覆盖服务端发出的精确未压缩payload bytes/part bytes，客户端验bytes后再解析，不靠跨语言重新序列化。快照不消耗event sequence。客户端验证连续part和完整hash后原子安装，发送精确匹配key/generation/ID/H/digest的`SnapshotInstalled`才接tail。ACK只接受当前session已投递frontier以内的单调值。key/payload kind/内部resource ID必须一一匹配。权限/设备在snapshot或replay途中撤销时发送abort、立即丢弃待发内容并关闭流。

慢客户端不能阻塞工作区Actor。control lane预留独立额度发送stream reset；若control lane也不可用则立即关闭连接，并把下次Hello的该流强制为snapshot。

### 11.3 文档写租约

```text
DocumentLease
├── document_id
├── owner_client_id
├── lease_epoch
├── acquired_at
└── expires_at
```

心跳和TTL属于配置，不属于wire常量。转移租约增加epoch；Control Plane、Workspace Agent、容器或WSL incarnation变化使所有瞬时租约变为unowned并持久提升epoch，旧设备的Edit/Renew/Input/Debug全部拒绝。Workbench control、文档写入、终端输入和调试控制是四类不同租约，互不授予权限。

`LeaseCommand`按typed target应用不同授权：Workbench要求owner的可控制设备；Document要求DocumentWrite和完整文档stream能力；Terminal要求TerminalInput；Debug要求DebugControl。统一消息只复用wire结构，不意味着`PERMISSION_DOCUMENT_WRITE`可以取得其他三类租约。

### 11.4 UI请求路由

插件QuickPick、InputBox、Dialog、URI、Agent审批等携带`origin_client_id`、request ID、extension generation、deadline和响应策略：

- 用户发起命令：只路由发起端。
- 后台通知：当前控制端或全部观察端，按类型配置。
- Agent审批：授权设备可见，首次有效响应获胜。
- 交互式QuickPick/InputBox：控制端；断开后取消。
- 调试/终端控制：对应租约持有者。

## 12. 终端、Git、调试、测试与语言服务

### 12.1 终端

服务端PTY保存有界scrollback和单调byte offset。多端可观察，一个客户端拥有输入租约。客户端重连按offset获取增量；过旧则获取带版本化normalized VT checkpoint的最新窗口快照（screen/scrollback、style table、cursor、alternate-screen/modes和checkpoint byte offset），安装后再重放后续raw bytes。只有明确标记`CLEAR_AND_RESET_DEGRADED`时可清屏复位并提示历史显示降级，不能从任意截断字节猜测VT状态。终端渲染使用Flutter原生VT parser/renderer，不将ANSI解释放在普通Text widget中。

初始预算：单终端scrollback 4MiB/100,000行取较小值；单工作区终端数16；具体值经基准后冻结。控制字符、OSC链接、clipboard和文件下载序列默认禁用或严格allowlist。

### 12.2 Git

- 使用容器内固定Git CLI和结构化参数，禁止Shell拼接。
- status使用porcelain v2，diff/commit/log返回typed DTO或受限文本流。
- checkout/merge/rebase等可能大量改写文件的动作先获取workspace mutation barrier。
- 凭据由短期AskPass/SSH agent代理提供，不写`.git/config`或环境日志。
- Git操作后的文件变化通过内容hash和watcher统一进入Document Service。

### 12.3 LSP/DAP/Test

扩展启动语言服务器和调试器，Adapter实现相应MainThread actor并把结果投影为稳定协议。DAP变量树、测试树和SCM资源均按handle分页，禁止一次发送无限节点。

语言请求携带document version、request ID和CancellationToken；迟到的旧version响应丢弃。补全列表、locations和code actions有continuation，diagnostics/semantic full必须适配单事件上限；超限时发snapshot-required并由包含完整派生状态的document snapshot恢复，禁止可被中途ACK的multipart事件。Completion/Rename/Formatting/CodeAction只返回不可变WorkspaceEdit candidate，显式Apply才产生副作用。

## 13. Agent方案

首版不创建新的插件系统。固定扩展可使用Code-OSS已有Chat/Language Model Tool/MCP API，Headless Adapter负责注册和调用映射；Rust Agent Orchestrator负责任务、模型、工具策略、审批、checkpoint和多端事件。

```text
Flutter Agent UI
 -> Rust Agent Orchestrator
 -> Model Gateway
 -> Tool Registry
    ├── Code-OSS Language Model Tools
    ├── MCP servers in container
    ├── Document/Filesystem
    ├── Git/Test/Terminal broker
    └── approved network tools
```

Agent任务持久字段：prompt引用、模型ID、状态、步骤、工具调用、审批、document versions、Git checkpoint、event sequence和终态；不重复保存不必要的敏感完整上下文。每次批准绑定tool call ID、精确参数digest、workspace generation，以及每个受影响document/terminal/debug target的`LeaseFence(target,generation,epoch)`；多文档操作不能用一个裸epoch代表全部资源。

文件变更流程：Agent创建候选WorkspaceEdit，Rust校验base versions和策略，Flutter展示Diff；批准后通过Mutation Broker提交。Agent不得直接把“已生成内容”标记为已写入。

Node扩展仍可使用`fs`/`child_process`绕过标准API，所以安全依赖固定扩展审计、容器、Workspace Trust和外部变更监视；一个Extension Host内没有per-extension强隔离。

## 14. 项目创建、导入与导出

### 14.1 创建来源

虽然Windows不提供本地工作区，产品仍需提供服务器侧入口：

- 空白项目/受控模板。
- Git URL clone；凭据由短期broker注入。
- 上传ZIP或`tar.zst`；先在隔离staging校验路径、类型、文件数、膨胀比和总大小，再原子导入。
- 管理员预置项目卷。

导入也是长Operation，失败不能留下半可见工作区。

### 14.2 一致性导出

```text
CreateExport
 -> Control Plane持久化phase/generation/fencing token
 -> Runtime Agent获取同卷互斥锁，Control Plane持久记录lock handle后关闭新准入
 -> 持锁Runtime Agent冻结Execution容器
 -> Coordinator drain watcher、归并direct write、flush已ACK文档/authority
 -> Runtime Agent冻结Coordinator容器
 -> Runtime Agent启动digest锁定的临时Exporter Helper容器（无网络/authority，RO项目卷）
 -> Helper执行卷级syncfs并只读遍历项目卷；Runtime Agent不宿主挂载读取
 -> 流式归档 + SHA-256 + manifest
 -> helper terminal/关闭FD后原子发布artifact并提交job terminal
 -> 解冻Coordinator并按watermark对账，再解冻Execution和准入
 -> 一次性exchange token换取可多Range续传的短期download session
```

ExportCoordinator位于容器外，由Control Plane持久拥有。pause使用显式`pause_id`，Runtime Agent watchdog可在Control Plane/WSL重启后根据容器label/helper/mount状态恢复；同卷export/import/backup/migrate/delete/stop互斥。成功路径只有helper terminal并关闭FD后才能解冻；失败发生在helper启动后时严格`cancel helper → wait → unmount → unpause`，发生在helper创建前时由Runtime Agent回读`HelperNeverStarted`证据后解冻，两者最终才释放volume lock。首版明确显示短暂维护状态，不虚假宣称在线快照。后续若底层引入LVM/Btrfs/ZFS真实快照，再优化暂停时间。

导出安全：不跟随越界symlink；拒绝device/socket/FIFO和异常hardlink；默认排除IDE状态、secret和构建产物；限制文件数、单文件、总大小、归档大小和时间；失败隔离`.partial`并确保unpause。ZIP不可靠保留Unix语义，涉及执行位/链接时推荐`tar.zst`。`git bundle`只包含已提交refs/history，不包含未提交working tree，必须在manifest和UI中与项目归档明确区分。

## 15. 认证、授权与秘密

- 使用OIDC Authorization Code + PKCE，通过用户设备的外置系统浏览器登录；Flutter不嵌入登录WebView，服务端与发布包不捆绑或启动浏览器，测试也不得用 Playwright/Puppeteer/Chromedriver 替代该边界。
- Access token短期有效；refresh token旋转并存平台安全存储。
- 每个设备生成Ed25519密钥，握手签名服务端challenge以绑定设备session。
- 权限至少分：workspace read、document write lease、terminal input、debug control、agent approve、export、admin。
- Gateway和领域Actor均复核资源归属，不能只信任URL路由。
- Secret通过绑定operation/进程身份/用途/TTL的credential broker，在执行点以短期pipe/memfd/一次性回调注入；禁止放入全容器env/共享tmpfs、项目卷、全局Extension Host环境或导出。
- Workspace Trust独立于登录；不信任项目默认禁Task、Debug、自动Agent shell及会执行项目代码的扩展功能。

完整威胁和容器边界见`SECURITY.md`。

## 16. 编码规范与模块契约

### 16.1 ID和时间

- 业务实体ID使用强类型newtype，不暴露裸String。
- Workspace/Document等可排序实体建议UUIDv7；Request/Operation使用UUIDv4。
- 传输Session ID不能作为用户、工作区、文档或授权主键。
- 服务端时间全部UTC；租约判断使用单调时钟，持久时间用于审计而非精确超时恢复。

### 16.2 并发

- Actor按工作区/文档串行化关键状态；跨actor用有界channel。
- Mutex只保护短状态，不跨`await`做网络/磁盘/Node/容器/模型I/O。
- 每个spawned task有owner、取消路径和JoinSet；禁止fire-and-forget。
- 所有semaphore permit由RAII对象持有到真实完成，不因调用方Future drop提前释放。

### 16.3 错误与日志

- 内部错误保留cause chain，仅进入受控日志。
- 公共错误通过穷尽match投影；未知内部错误映射`internal`并生成correlation ID。
- 日志字段allowlist；禁止代码内容、token、secret、完整命令、用户路径、自由文本插件错误进入指标label。
- 审计事件只记录actor类别、动作、资源类别、结果和安全原因；可用hash链检测篡改。

### 16.4 配置

- TOML/YAML/JSON配置必须有`schema_version`、严格类型和未知字段策略。
- secret不放普通配置或发行包。
- 资源上限只能由管理员在安全范围内调整；超过编译最大值拒绝启动。
- Runtime Release指纹不匹配时在插件加载和工作区写入前fail-closed。

## 17. 实施阶段与定义完成

详细任务依赖见`ROADMAP.md`。总体顺序：

1. `G0`：Headless Extension Host可行性Spike；使用隔离的bootstrap spike工作区。
2. `M0`：仓库、工具链、协议、错误、operation、CI骨架；可与G0并行启动，但M1及产品主线必须同时等待G0 Go和M0验收。
3. `M1`：Linux/WSL2统一Runtime Agent、双容器、卷/配额、reconciler。
4. `M2`：Workspace Agent、DocumentActor、Mutation Broker、可靠ACK。
5. `M3`：Flutter五端原生Workbench与编辑器。
6. `M4`：Adapter C0与C1 UI投影、Terminal/Task actor stub、固定扩展的非执行能力认证；完整文档恢复闭环通过后才advertise DOCUMENT四项组合。
7. `M5`：Terminal/Task真实执行、Git/SCM、Debug、Testing与其语言工作流，并在端到端门通过后advertise对应执行capability。
8. `M6`：Agent API、工具审批、任务恢复。
9. `M7`：多端恢复、租约、状态同步、一致性导入/导出和灾难恢复。
10. `M8`：安全加固、供应链、soak、canary和发布。

每项功能只有同时具备代码、unit/contract/integration测试、文档、资源上限、错误投影、可观测和故障清理才算完成。

## 18. 建议验收指标

| 领域 | 初始目标 |
|---|---|
| 编辑持久性 | 已ACK编辑经服务、容器、WSL重启100%恢复 |
| 服务端编辑提交 | 本地处理p95≤15ms、p99≤50ms，网络另计 |
| 视觉输入 | 不等待网络；100ms RTT下ACK p95≤RTT+75ms |
| 状态恢复 | 10k可重放事件≤2s；快照首个可用视图≤5s |
| 工作区启动 | 已运行宿主冷启动p95≤20s；预热≤5s |
| WSL恢复 | `wsl --shutdown`后ready≤45s |
| Extension Host恢复 | 非连续故障重新ready≤15s |
| 多端传播 | 稳定网络下已提交状态p95≤RTT+100ms |
| 导出 | 1GiB/100k文件≤180s且树hash一致 |
| 隔离 | 宿主、其他工作区、runtime socket、metadata地址100%拒绝 |
| 资源清理 | 24h soak后task/FD/PTY/container/operation回到基线 |
| 长操作 | fault矩阵中每个operation恰好一个终态 |

指标必须绑定固定机器、镜像digest、工具版本、seed、并发和原始报告。没有重复证据前不得把目标描述为已达成SLO。

## 19. 明确暂缓

- 用户插件市场、运行时插件安装和任意VSIX。
- browser-only extension、Webview、Custom Editor、完整Notebook/Interactive Window。
- Windows工作区和Windows容器。
- Flutter Web或PWA。
- 多写者协同编辑、离线本地工程。
- 多节点控制面、跨主机实时迁移和Kubernetes调度。
- 恶意公网多租户安全承诺；首版以可信组织/自托管为部署模型。

这些能力只能通过新的ADR、协议登记、安全/资源矩阵和独立里程碑进入实现。
