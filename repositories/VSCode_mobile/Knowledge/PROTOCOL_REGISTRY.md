# 协议、Capability、Schema与稳定错误注册表

> 状态：initial planned registry。编号进入代码前需由协议owner确认；已分配编号不得复用。

## 1. 登记规则

状态：

- `planned`：已分配、尚无集成实现。
- `implemented`：当前HEAD存在producer/consumer和合同测试。
- `reserved`：保留，任何producer不得发送。
- `deprecated`：仅保留历史解析/拒绝，禁止新用途。
- `unsupported`：产品明确拒绝，不等同reserved编号。

每个外部或持久边界必须登记owner、版本、资源上限、错误、固定向量、生成器和兼容矩阵。未知、未协商或future major在业务副作用前拒绝；编号、field ordinal和错误code不得重新解释。

## 2. 协议域

| 域 | 标识 | Major | 状态 | Owner |
|---|---|---:|---|---|
| Flutter实时会话 | WebSocket subprotocol `remote-ide.pb.v1` | 1 | planned | protocol |
| 公共HTTPS API | `/api/v1` | 1 | planned | gateway |
| Control Plane↔Runtime Agent | UDS package `remote_ide.internal.runtime.v1` | 1 | planned | runtime |
| Control Plane↔Workspace Agent | package `remote_ide.internal.workspace.v1` | 1 | planned | workspace |
| Workspace Agent↔Headless Adapter | package `remote_ide.internal.adapter.v1` | 1 | planned | codeoss |
| Adapter↔Extension Host | Code-OSS protocol hash | exact commit | planned | codeoss |
| Export manifest | `remote_ide.export.v1` | 1 | planned | export |
| Runtime release manifest | `remote_ide.runtime_release.v1` | 1 | planned | release |

客户端公共protocol major和内部major互不兼容、互不复用消息。任何内部端点不得对客户端网络可达。

## 3. Capability数值分配

`CapabilitySet`在wire上使用repeated enum，避免不同语言的大整数问题。未知required capability拒绝；未知optional capability忽略但不启用。

| 值 | 名称 | 状态 | 说明 |
|---:|---|---|---|
| 0 | `CAP_UNSPECIFIED` | reserved | sentinel，非能力且不得advertise |
| 1 | `CAP_DOCUMENT_EDIT_V1` | planned | 文档snapshot/edit/ack |
| 2 | `CAP_STATE_RESUME_V1` | planned | 多流cursor恢复 |
| 3 | `CAP_DOCUMENT_LEASE_V1` | planned | 单写租约 |
| 4 | `CAP_LANGUAGE_UI_V1` | planned | completion/diagnostic/hover等 |
| 5 | `CAP_TERMINAL_V1` | planned | PTY output/input lease |
| 6 | `CAP_SCM_V1` | planned | Git/SCM投影 |
| 7 | `CAP_DEBUG_V1` | planned | DAP投影 |
| 8 | `CAP_TESTING_V1` | planned | Test tree/run |
| 9 | `CAP_AGENT_V1` | planned | Agent session/tool/approval |
| 10 | `CAP_EXPORT_V1` | planned | 一致性导出 |
| 11 | `CAP_ZSTD_CHUNKS_V1` | reserved | 有界chunk压缩，未实现前不得启用 |
| 12 | `CAP_TASK_V1` | planned | Task定义、执行、终止与终态投影 |
| 13 | `CAP_OUTPUT_V1` | planned | Output append/replace/clear/show/hide与恢复 |
| 14-63 | — | reserved | v1公共能力保留 |

Capability不能改变认证、ID语义、固定字段或安全上限。需要改变这些内容时分配新major或独立versioned message。

### 3.1 可执行门控矩阵

`protocol/ide/v1/capability-matrix.yaml`是顶层与嵌套case的机器可读事实源；M0由Rust/Dart/TypeScript validator生成器把它编译为穷尽match，未知case默认拒绝。它登记`StreamKey→domain capability`、request→response/outcome/result-kind、nested action→effect guard、UI subtype闭包、event/snapshot分支identity与Snapshot/Result transfer状态机，并显式列出已知但不可advertise的reserved capability。M0-03当前生成物已把全部matrix路径编译成三端只读查询表，并实现protocol major/minor窗口、ClientHello required/optional唯一与不相交、required的known/zero/reserved拒绝和future optional忽略、ServerHello selected version/session/workspace/generation/build/runtime/heartbeat/frame budget/permission/enabled capability闭包、UUIDv4/v7、snapshot/event/lease逐值identity、顶层及nested request-response/outcome/effect关联、Language result family/rename/continuation关联、UI recipient/subtype/routing/handle/token/唯一终态，以及Snapshot/Result的part/offset/bytes/digest/cancel/End/Aborted/install状态；Hello/Error/Heartbeat canonical bytes与同一状态向量已由三端测试。三端先以最小envelope校验官方生成的`ClientFrame`/`ServerFrame`对象，再由`FrameRequestState`把实际list/search/language请求、response part、cancel与Language document/version/generation/result family原子送入状态机；该执行状态仍拒绝所有effectful请求。独立`FrameEffectIntent`对当前全部direct/nested effect分支提取实际`CommandMeta`，绑定session、client incarnation、principal、request/idempotency key、frame/command workspace generation与双deadline，并从matrix生成request/effect capability、permission-all/any、公共与路由动态前置条件；静态grant缺项即拒绝。Intent的`ready_for_execution/readyForExecution`固定为false，直到真实认证session、ACL、资源ownership/precondition与idempotency registry逐项闭合，因此结构分类绝不等于准入或副作用执行。`FrameUiState`另把实际`ServerFrame.request.ui`、`ClientFrame.server_response.ui`、QuickPick分页request/terminal response/cancel及server UI cancel原子接入selected session/client incarnation、workspace/extension generation、UI subtype/outcome、handle/token与唯一终态校验。`FrameEventState`、`FrameSnapshotState`与`FrameResultState`分别接入实际event、snapshot和result delivery；result只允许`GetAgentCandidate`或`WorkspaceEditCommand.get`创建，并把terminal Started、Begin kind/digest、transfer ID、candidate/agent identity、Chunk/End/Aborted及Cancel绑定为一个原子状态。Snapshot/Result在`COMPRESSION_NONE`下按Begin声明的精确总长建立不超过16 MiB的固定缓冲区，每个Chunk先校验part SHA-256再按连续offset写入，End再校验Begin/End/实际整包SHA-256三者相等，随后只通过官方protobuf decoder得到typed payload并执行identity/edit-digest校验；解码或校验失败不提交payload ready/install，Abort清空缓冲。`COMPRESSION_ZSTD`在尚无exact、受预算约束的解码器前继续拒绝，不能因reserved capability绕过。分页只允许一个在途短请求，`page_size/items`受65536 repeated-element上限约束；成功页原子推进token或进入耗尽，terminal error恢复原token以供新request ID重试。失败不提交page、frontier、payload verified或terminal。M0-03已达到`implemented`但未`accepted`；真实transport/session registry及effect运行时准入归M1/M2，当前minor 0没有previous-minor成员，首次minor升级前必须加入N-1 golden。所有capability仍不advertise。仅匹配envelope或case从不等于认证、授权或业务准入成功。

首版`DOCUMENT` stream是不可过滤的恢复闭包，统一要求`STATE_RESUME + DOCUMENT_EDIT + DOCUMENT_LEASE + LANGUAGE_UI`：文档正文、租约、diagnostics和semantic tokens共享cursor，不能按capability删去中间事件。这样会使M2/M3只能通过内部fixture验证，直到M4具备完整闭包后才允许对产品客户端advertise这组能力。`WORKSPACE`、`TERMINAL`和`DEBUG` snapshot含租约投影，分别也继承`DOCUMENT_LEASE`；若未来要独立协商，必须先分裂StreamKey/cursor/snapshot，而不能条件隐藏字段。

| 顶层消息/载荷 | 必需Capability | 额外门 |
|---|---|---|
| Hello/Heartbeat/Cancel、PublicError、Workspace/File只读、Operation查询 | baseline v1，无额外capability | 当前workspace generation、ACL/资源归属 |
| Workspace/File有副作用 | baseline v1 | `WORKSPACE_MANAGE`或`DOCUMENT_WRITE`、CommandMeta、exact file precondition |
| Resume/ACK/StreamReset/所有stream snapshot envelope | `CAP_STATE_RESUME_V1` + `stream_key_capabilities`中的完整domain集合 | 当前session订阅/frontier；Snapshot identity闭包 |
| Document open/edit/save/event/snapshot | `CAP_STATE_RESUME_V1 + CAP_DOCUMENT_EDIT_V1 + CAP_DOCUMENT_LEASE_V1 + CAP_LANGUAGE_UI_V1` | 写入另需`DOCUMENT_WRITE`；同一stream不按cap过滤 |
| Lease command/state/fence | `CAP_DOCUMENT_LEASE_V1` | typed target、principal/device、generation/epoch |
| WorkspaceEdit candidate/get/apply/result transfer | `CAP_DOCUMENT_EDIT_V1` + `CAP_DOCUMENT_LEASE_V1` | apply另需`DOCUMENT_WRITE`、digest/TTL/逐资源fence |
| 文档LanguageRequest/Response与Diagnostics/Semantic | DOCUMENT四项完整组合 | extension generation、document version；结果可能携带WorkspaceEdit candidate，不能漏Lease能力 |
| 非文档UI/Contribution/Workbench | `CAP_LANGUAGE_UI_V1`（Workbench另需Lease） | Clipboard另需device permission；command另取动态policy并集 |
| Terminal | `CAP_TERMINAL_V1` | create/input permission；input lease |
| SCM | `CAP_SCM_V1` | action permission、revision/precondition |
| Debug | `CAP_DEBUG_V1` | Workspace Trust、debug permission/control lease/stop epoch |
| Testing | `CAP_TESTING_V1` | 执行时Workspace Trust、task permission、Workbench lease |
| Agent及只读事件 | `CAP_AGENT_V1` | run/approve permission、tool digest、risk |
| Agent candidate读取/应用 | `CAP_AGENT_V1 + CAP_DOCUMENT_EDIT_V1 + CAP_DOCUMENT_LEASE_V1` | candidate digest/TTL、逐资源fence与精确文件前置条件 |
| Export | `CAP_EXPORT_V1` | export permission、artifact ACL |
| Task | `CAP_TASK_V1` | Workspace Trust、task permission、Workbench lease |
| Output | `CAP_OUTPUT_V1` | channel ownership；客户端只投影，不执行字符串命令 |
| 任一Begin声明ZSTD | 对应domain capability + `CAP_ZSTD_CHUNKS_V1` | 解压后预算、精确raw payload digest |

Capability依赖取并集，不允许用较宽能力替代较窄能力。每个ACK/resume/reset/snapshot控制按其`StreamKey`动态继承domain集合；每个transfer part继承创建时的kind、capability和资源绑定。Operation cancel即使位于baseline `OperationCommand`内，也必须命中`operation_cancel` guard：调用者是operation owner或拥有该kind权限，携带当前workspace generation与`expected_operation_revision`，并由统一cancel-vs-commit仲裁产生唯一终态。矩阵、Proto oneof/enum清单和生成validator必须在CI零漂移；未列出的新增case编译/门禁失败。

### 3.2 Permission数值分配

Permission来自当前ACL快照，不由客户端声明。租约不是权限替代品；必须同时满足target-specific capability、permission、资源归属和epoch。

| 值 | 名称 | 状态 | 说明 |
|---:|---|---|---|
| 0 | `PERMISSION_UNSPECIFIED` | reserved | sentinel，永不授予 |
| 1 | `PERMISSION_WORKSPACE_READ` | planned | 读取本工作区可见投影 |
| 2 | `PERMISSION_WORKSPACE_MANAGE` | planned | 工作区生命周期、结构与SCM管理 |
| 3 | `PERMISSION_DOCUMENT_WRITE` | planned | 文档与受控文件写入 |
| 4 | `PERMISSION_TERMINAL_CREATE` | planned | 创建终端 |
| 5 | `PERMISSION_TERMINAL_INPUT` | planned | 输入、resize、关闭终端 |
| 6 | `PERMISSION_TASK_RUN` | planned | Task和Testing执行/取消 |
| 7 | `PERMISSION_DEBUG_CONTROL` | planned | 启动/控制/求值调试会话 |
| 8 | `PERMISSION_AGENT_RUN` | planned | 创建/推进Agent任务 |
| 9 | `PERMISSION_AGENT_APPROVE` | planned | 批准工具与Agent候选应用；写入仍另需DocumentWrite |
| 10 | `PERMISSION_EXPORT` | planned | 创建、取消、下载导出 |
| 11 | `PERMISSION_ADMIN` | planned | 管理面高风险操作，不隐含普通资源权限 |
| 12 | `PERMISSION_DEVICE_CLIPBOARD` | planned | 当前前台控制设备剪贴板 |
| 13-63 | — | reserved | v1保留 |

`LeaseCommand`按`LeaseTarget`再路由：workbench要求owner的可控制设备和WorkspaceRead；document要求DocumentWrite及完整DOCUMENT stream能力；terminal要求TerminalInput；debug要求DebugControl。一个target的permission不能取得另一target租约。

扩展命令不是`CAP_LANGUAGE_UI`下的万能后门。每个可执行命令必须存在于由固定扩展清单与管理员策略生成的可信command policy registry，descriptor包含policy digest、动态capability/permission并集、Workspace Trust和Workbench lease要求；请求回传精确digest，策略要求控制权时还回传精确Workbench lease epoch，未知、漂移、ABA或未分类命令拒绝。Extension Host自报字段不作为授权事实。

## 4. ID语义

| 类型 | 生成 | 用途 | 禁止 |
|---|---|---|---|
| `UserId` | 服务端opaque UUID | 用户主体 | 使用OIDC subject裸值作为公开ID |
| `DeviceId` | 服务端登记opaque UUID | 设备公钥主体 | 使用平台设备名授权 |
| `ClientInstanceId` | 客户端UUIDv4，每次进程实例 | 幂等/连接来源 | 长期用户主键 |
| `SessionId` | 服务端UUIDv4 | 单次连接 | 业务对象或授权主键 |
| `WorkspaceId` | UUIDv7 | 工作区 | 容器ID或路径 |
| `DocumentId` | UUIDv7/稳定URI映射 | 打开文档/journal | 文件句柄或临时tab ID |
| `OperationId` | UUIDv4 | 长操作 | 重用 |
| `RequestId` | UUIDv4 | 请求/响应/cancel | 用户可控自由文本 |
| `TerminalId` | UUIDv7 | PTY会话 | PID |
| `AgentTaskId` | UUIDv7 | Agent任务 | Extension Host内部handle |
| `ExportId` | UUIDv7 | 导出artifact | 文件系统路径 |
| `DebugSessionId` / `TestControllerId` / `TestRunId` / `ScmProviderId` | UUIDv7 | 工作流实体 | PID、DAP/扩展裸handle |
| `TaskId` / `OutputChannelId` | UUIDv7 | Task与Output实体 | label或contribution字符串 |
| `SnapshotId` / `UiRequestId` / `ProgressId` | UUIDv4 | 一次传输/交互 | 业务实体主键 |
| `ChallengeId` / `DiagnosticSetId` / `TokenSetId` | UUIDv4 | nonce或有界集合传输 | 跨上下文复用 |
| `ToolCallId` / `ApprovalId` / `CandidateId` / `TranscriptEntryId` | UUIDv4 | Agent/候选交互实例 | 资源授权主键 |
| `IdempotencyKey` | UUIDv4，由调用方每个逻辑mutation生成 | 去重 | request ID、跨principal复用 |

所有公开ID用canonical小写文本或16-byte字段，具体字段以proto为准。除表中服务端opaque `UserId`与`DeviceId`外，未另列的持久业务实体ID使用UUIDv7，单次请求/交互/集合/nonce使用UUIDv4；新增Uuid字段必须在注册表显式归类。解析拒绝非canonical、nil和错误版本ID。

M0-04 的 `remote-ide-ids` 已把本表全部登记类型实现为不可隐式互换的Rust newtype：UUIDv4/v7严格检查RFC variant/version，`UserId`与`DeviceId`只接受服务端opaque v4/v7，文本只接受36-byte小写hyphen canonical形式；deadline和workspace/state revision使用非零强类型且不隐藏读取系统时钟。状态为`implemented`、未`accepted`；其他语言适配和数据库表示仍由其owner闭合。

## 5. 公共消息族

`protocol/ide/v1/ide.proto`为源。消息族：

| Family | 关键消息 | 状态 |
|---|---|---|
| Session | `ClientHello`, `ServerHello`, `ClientFrame`, `ServerFrame`, `Heartbeat` | planned |
| Resume | `StreamCursor`, `ResumePlan`, `StreamReset` | planned |
| Workspace/File | `WorkspaceSnapshot/Event/Command`, `ListDirectory`, `Search`, `FileMutationCommand` | planned |
| Document | `OpenDocumentRequest`, `DocumentSnapshotPayload`, `EditBatch`, `EditAck`, `DocumentEvent` | planned |
| Workspace Edit | `WorkspaceEditReference`, `WorkspaceEditCommand`, `WorkspaceEditCandidatePayload`, `WorkspaceEditApplied` | planned |
| Lease | `AcquireLease`, `RenewLease`, `ReleaseLease`, `LeaseState` | planned |
| UI/Command Policy | `WorkbenchContextUpdate`, `UiRequest/Response`, `CancelUiRequest`, `Notification`, contributions/tree、`CommandPolicyDescriptor` | planned |
| Language | `LanguageRequest`, `LanguageResponse`, `DiagnosticSet`, `SemanticTokensEvent` | planned |
| Terminal | `TerminalCommand`, `TerminalChunk`, `TerminalSnapshot`, `LeaseState` | planned |
| SCM/Debug/Test | `Scm*`, `Debug*`, `Testing*` | planned |
| Agent | `AgentCommand`, `AgentEvent`, `ToolApprovalRequest/Response` | planned |
| Operation | `OperationCommand`, `OperationSnapshot`, `OperationEvent` | planned |
| Export | `CreateExport`, `ExportSnapshot`, `DownloadExchangeDescriptor` | planned |
| Task/Output | `TaskCommand/Event/Snapshot`, `OutputCommand/Event/Snapshot` | planned |
| Snapshot/Result | stream用`SnapshotTransferMeta/Begin/Chunk/End/Installed`；请求大结果用无cursor的`ResultTransfer*`/`ResultPayload` | planned |
| Error | `PublicError` | planned |

`SaveDocument`、`RevertDocument`、`SetDocumentLanguage`都同时携带expected document version与document lease epoch；`ExecuteCommandRequest`在可信策略要求Workbench控制时携带expected workbench lease epoch。validator在准入处检查，长Operation在最终commit再检查同一fence，关闭失去后重获的ABA。

任何oneof未知case在该消息被要求时视为`unsupported_feature`/`incompatible_version`；不能把空body当默认命令执行。安全无关的未来optional字段可由旧端忽略。`ClientFrame`与`ServerFrame`方向分离；每帧再通过`delivery` oneof严格属于request/response/event/snapshot/server-request/control之一。`request_id`只做短请求相关性，effectful command必须另有`idempotency_key`。短请求、长Operation和服务端UI请求使用三套取消消息。请求只能由矩阵列出的response/outcome/result kind终结；相同capability不能把错误响应类型变成合法。Language的10个request branch进一步绑定具体result branch；rename按`prepare_only`分流，continuation token绑定原result family。UI同样按subtype绑定唯一response family，Clipboard还强制controller routing、前台设备会话和设备权限。

投递矩阵由validator强制：短响应只能使用`ServerResponse`并携带`ResponseMeta`；多块Search响应的`part_index`从0连续递增且恰有一个`TERMINAL`，error必为terminal；可恢复状态变化只能使用`ServerEvent`并携带唯一`StreamEventMeta`；stream完整状态只能使用`ServerSnapshot`，请求型大候选只能使用无cursor的`ServerResultTransfer`。启动transfer的响应本身是terminal，后续只按transfer ID路由；abort不得推进snapshot frontier。`UiRequest`只能走`ServerRequest`；Hello/heartbeat/reset/notification只能走control。禁止response同时带event cursor、result transfer携带StreamKey、snapshot同时消费event cursor、空delivery、terminal之后继续发送，Flutter Future和stream reducer不能消费同一帧。

CancelRequest、CancelSnapshotTransfer、CancelResultTransfer及UI response/cancel都维护active-state表：ID绑定原session/client incarnation/selected recipient、extension generation及原请求/transfer capability。cancel与正常terminal/End/Aborted/timeout/disconnect通过同一状态仲裁只允许一个赢家；未知或跨session ID拒绝。成功取消Result/Snapshot transfer产生唯一Aborted，原短请求仍自行产生唯一terminal响应。

`QuickPickPageRequest`不是普通全局查询：它必须匹配仍active的QuickPick `ui_request_id + extension_generation + selected recipient session + items_handle`，page token再绑定该四元组；observer或其他会话即使属于同owner也不能读取控制设备的候选页。

## 6. Stream类型与游标

| Stream type | ID构造 | sequence语义 | 压缩后恢复 |
|---|---|---|---|
| `WORKSPACE` | workspace ID | 独立event sequence | workspace snapshot |
| `DOCUMENT` | document ID | 独立event sequence；canonical version仅在payload | document snapshot |
| `TERMINAL` | terminal ID | 独立event sequence；raw byte offset仅在payload | scrollback/control snapshot |
| `AGENT` | task ID | 独立event sequence | task snapshot + tail |
| `EXTENSION` | workspace ID；extension重启由stream generation表示 | 独立event sequence；contribution revision在payload | contribution snapshot |
| `OPERATION` | operation ID | 独立event sequence | terminal/current snapshot |
| `SCM/DEBUG/TESTING` | provider/session/controller ID | 各自独立event sequence | 对应domain snapshot |
| `EXPORT/TASK/OUTPUT` | export/task/channel ID | 各自独立event sequence | 对应domain snapshot |

Identity validator必须逐帧执行，不能把错配payload重路由：

`WorkspaceEvent.workspace_id`、`ContributionDelta.workspace_id`、`TreeViewInvalidated.workspace_id`与`ContributionSnapshot.workspace_id`是必填的payload侧资源身份；后三者对应workspace-scoped `ExtensionStreamKey.workspace_id`。validator不能从`StreamKey`回填或推断这些字段。三端`FrameEventState`由已订阅的expected key/generation/frontier创建，只接受实际`ServerFrame.event`；它先分别提取typed key与payload identity，再校验允许路由、UUID完全相等、extension generation（适用时）及严格连续cursor。重复、洞、错generation、错key或错payload均不得推进本地frontier。

| StreamKey branch | 允许的主要Event | SnapshotPayloadKind | 必须相等的payload身份 |
|---|---|---|---|
| workspace | WorkspaceEvent、workbench LeaseState | WORKSPACE | `WorkspaceEvent.workspace_id` / workbench target |
| document | DocumentEvent、DiagnosticSet、SemanticTokensEvent、document LeaseState | DOCUMENT | document ID / document target |
| terminal | TerminalEvent、terminal LeaseState | TERMINAL | terminal ID / terminal target |
| agent | AgentEvent | AGENT | agent task ID |
| extension | ContributionDelta、TreeViewInvalidated | CONTRIBUTIONS | payload中的显式`workspace_id`；stream generation就是extension generation |
| operation | OperationEvent | OPERATION | operation ID |
| debug | DebugEvent、debug LeaseState | DEBUG | debug session ID / debug target |
| testing | TestingEvent | TESTING | controller ID；active run均属该controller |
| scm | ScmEvent | SCM | provider ID |
| export | ExportEvent | EXPORT | export ID |
| task | TaskEvent | TASK | task ID |
| output | OutputEvent | OUTPUT | output channel ID |

Snapshot Begin的kind、解码后的payload branch/内部ID与meta key必须完全匹配；ResultPayload没有StreamKey且不得出现在ServerSnapshot。三端`FrameSnapshotState`只由实际terminal `SnapshotTransferStarted`与首个`ServerSnapshot.begin`的完全相同transfer tuple创建；后续实际Chunk/End/Aborted必须重复同一tuple、连续part/offset并满足固定bytes/digest预算。End后必须由有界decoder提交已计算且等于Begin/End的wire SHA-256，再从实际`SnapshotPayload`提取kind/ID；只有随后完全相同的`SnapshotInstalled`才能原子置为installed。validator本身不把“已解码”冒充“已验hash”。任何错配都以protocol violation关闭会话并记录安全审计，不向另一资源投递。

三端`FrameResultState`只由实际`GetAgentCandidate`或`WorkspaceEditCommand.get`、对应该request的terminal `ResultTransferStarted`和首个`ServerResultTransfer.begin`创建。它固定workspace generation、request/transfer ID、result kind、candidate ID、可选agent task ID及expected edit digest；后续Chunk/End/Aborted和`CancelResultTransfer`必须命中同一transfer。End后只有内部固定缓冲完成连续组装、逐part与整包SHA-256均通过，并由官方protobuf decoder得到的实际`AgentCandidateEdit`或`WorkspaceEditCandidatePayload`同时匹配创建路由、candidate/agent identity、edit digest且携带edit，结果才可读。Result永不含StreamKey、不推进frontier、也不产生SnapshotInstalled；错kind、错resource、错digest、重复part/cancel、malformed payload或terminal后帧均不提交状态。

LeaseState另有逐字段identity闭包：LeaseCommand响应target分支和UUID必须等于原请求并按target重算能力；Workspace/Document/Terminal/Debug snapshot内嵌lease的target类型和UUID同时等于snapshot key与payload资源ID；lease event的target类型及UUID等于event StreamKey。仅匹配“document/terminal/debug”类型而UUID不同仍是协议违规。

序号从1开始，0表示无游标。sequence溢出前必须进入新major或新stream generation，禁止wrap。

`StreamKey`是typed oneof，不使用`type + opaque bytes/string`；ACK是repeated per-stream `(key,generation,acknowledged_cursor)`，该cursor就是独立event sequence，不存在全局ACK。服务端仅接受同一已订阅key/generation、单调且不超过该session实际投递frontier的ACK，future ACK和跨session ACK必须拒绝。Snapshot先登记live tail，再在Actor barrier获取`generation + high_watermark`；snapshot/replay覆盖到inclusive watermark，live只从严格大于watermark开始。统一Snapshot transfer以`snapshot_id + part_index + payload schema + terminal digest`传输序列化的typed `SnapshotPayload`，不消耗event sequence；客户端只有在Begin声明、连续chunk、End计数/hash全部匹配后才能原子安装并发送`SnapshotInstalled`。该消息必须精确匹配当前transfer的key/generation/ID/H/digest，服务端才释放已缓冲tail。`ResumePlan`要求snapshot时服务端自动启动；失败重试可显式发送`RequestStreamSnapshot`。授权撤销会清除缓冲并终止流。

每种snapshot必须形成其stream的完整可见状态闭包：workspace snapshot包含当前owner工作区偏好和Workbench控制租约；document snapshot包含文本、写租约、完整diagnostics与semantic tokens；terminal snapshot包含输入租约和VT render checkpoint；debug/testing snapshot包含控制租约或活动run。压缩后不能依赖已删除事件重建这些状态。个性化workspace snapshot按principal隔离缓存和hash，禁止跨用户复用。

## 7. 计划资源上限

以下是初始安全上限，G0协议夹具、M3编辑器容量与M5工作流基准后冻结；修改需资源/兼容评审：

| 资源 | 默认/最大计划 |
|---|---:|
| WebSocket frame | 1MiB |
| EditBatch edit数 | 256 |
| EditBatch插入bytes | 256KiB |
| Snapshot chunk | 256KiB |
| Result transfer chunk | 256KiB |
| WorkspaceEdit candidate | 8MiB/4096 operations，超限拒绝 |
| Command policy registry | 4096条/工作区；单命令32项capability+permission/effect并集 |
| 普通可编辑文档 | 32MiB |
| 客户端发送缓冲 | 8MiB |
| Terminal chunk | 64KiB |
| 单终端scrollback | 4MiB或100k行 |
| 单工作区终端 | 16 |
| 单客户端挂起UI请求 | 64 |
| Workspace事件重放 | 10k条或24h |
| Operation active | 1024/服务实例，按用户/工作区另限 |
| 导出artifact TTL | 24h默认，管理员可缩短 |

协议parser必须先验证声明长度、计数和总预算，再分配。压缩功能未实现前不得advertise。

## 8. Stable Public Error Codes

| Code | Category | Retryable默认 | 语义 |
|---|---|---:|---|
| `invalid_argument` | client | false | 结构/范围/ID/路径无效 |
| `unauthenticated` | auth | false | 无有效会话 |
| `permission_denied` | auth | false | ACL或操作权限不足 |
| `not_found` | domain | false | 当前主体不可见的资源不存在 |
| `conflict` | state | false | 一般状态冲突 |
| `revision_conflict` | state | true | 文档base version过期 |
| `write_lease_held` | state | true | 其他客户端持有租约 |
| `failed_precondition` | state | false | 前置状态不满足 |
| `workspace_starting` | availability | true | 工作区尚未ready |
| `workspace_unavailable` | availability | true | 工作区degraded/failed |
| `extension_host_unavailable` | availability | true | Adapter/ExtHost不可用 |
| `extension_api_not_supported` | compatibility | false | 明确不支持的VS Code API |
| `resource_exhausted` | resource | true | quota/admission/queue满 |
| `deadline_exceeded` | timeout | true | deadline终态 |
| `operation_cancelled` | cancellation | false | cleanup后取消终态 |
| `unsupported_feature` | compatibility | false | capability未实现 |
| `incompatible_version` | compatibility | false | protocol/runtime不兼容 |
| `export_too_large` | resource | false | 导出预算超限 |
| `data_corrupted` | storage | false | hash/integrity/schema损坏 |
| `internal` | internal | false | 安全投影后的未知故障 |

code语义一旦发布不得静默改变。新增code为additive；删除/重解释为breaking。

M0-04 的 `remote-ide-error` 已穷尽实现本表20个code、11个category及默认retryable映射，并投影正式Rust Proto；safe detail只接受固定key和ASCII安全值，最多16项/4 KiB，单值256 bytes，retry-after最多24h，内部路径、URI分隔符、控制字符、任意error chain和自由localized fallback不能进入公共对象。状态为`implemented`、未`accepted`；这不是传输层错误producer已上线。

## 9. 存储Schema域

不同数据库各自拥有版本域，不共用一个整数：

| Store | 初始schema | 状态 | Owner |
|---|---:|---|---|
| Control Plane SQLite | 1 | planned | store |
| Workspace document DB | 1 | planned | document |
| Extension state DB/layout | Code-OSS exact | planned | codeoss |
| Agent checkpoint DB | 1 | planned | agent |
| Audit log canonical event | 1 | planned | security |
| Export manifest | 1 | planned | export |

Future schema必须fail-closed、先备份、禁止旧binary写回。详细规则见`PROTOCOL_LIFECYCLE.md`。

M0-05 已把上述规则实现为通用`remote-ide-store` migration harness：每个fixture store独立绑定nonzero SQLite `application_id`、canonical store key、`user_version`与migration digest history；future/cross-store/corrupt/history drift在写入前拒绝，DDL/validator/history/version同一IMMEDIATE transaction，不可逆步骤先在锁内完成有界create-new backup与恢复演练。专用generator提供Control Plane、Workspace document、Agent checkpoint的fixture-only v0/v1及future/invalid/orphan/corrupt真实库。该状态为`implemented`、未`accepted`，只证明机制；表中产品业务schema状态仍全部保持`planned`，Extension state仍由Code-OSS exact owner管理。

M0-06 的`remote-ide-observability`另实现canonical audit event schema v1的纯Rust机制：UUIDv7 event ID、连续sequence、UTC millis、actor category+opaque fingerprint、device/workspace、闭合action/decision/target/result、policy version、request或operation correlation，以及覆盖previous hash与全部字段的SHA-256 event hash。append和导入verify均重新验证semantic contract，checkpoint绑定sequence/hash。该状态为`implemented`、未`accepted`；它没有公共wire producer、Audit log产品表、append-only远端sink或业务事件接线，因此上表`Audit log canonical event`持久schema仍保持`planned`。

## 10. Code-OSS Runtime Registry

每个Runtime Release登记：

```text
runtime semver
Code-OSS full commit/API version
Node exact version/modules ABI
extension host protocol hash
actor inventory hash
contribution inventory hash
adapter commit/protocol major
coordinator image digest
execution image digest
exporter-helper image digest
wsl2 rootfs digest（Linux发行显式为null/not_applicable，字段不可省略）
extensions.lock digest
public/internal protocol windows
read/write storage schema windows
```

Adapter与Extension Host要求exact组合。Flutter只要求公共协议兼容，不绑定Code-OSS commit。

## 11. Extension Compatibility Status

| 等级 | 状态 | 说明 |
|---|---|---|
| C0 Core | planned | command/config/fs/document/language/diagnostic/search |
| C1 Native Adapted | planned | QuickPick/Dialog/Progress/Clipboard/Tree/Status/Output/Terminal/Task |
| C2 Workflow | planned | SCM/Debug/Testing/Auth/Comments |
| C3 Agent | planned | LM Tool/Chat/MCP/Agent Diff |
| U Unsupported | frozen | Webview/Custom Editor/DOM/Electron/HTML renderer |

每个bundled extension必须登记分类和认证报告；“加载成功”不等于功能认证。

Capability只在producer、consumer、授权、恢复快照和端到端合同全部存在后advertise。M4的Terminal/Task actor stub不启用`CAP_TERMINAL_V1`或`CAP_TASK_V1`；两者在M5真实Execution后端和恢复/安全门通过后才能启用。`CAP_OUTPUT_V1`同样要求append/replace/clear/show/hide及snapshot闭环，而不是只有contribution descriptor。
