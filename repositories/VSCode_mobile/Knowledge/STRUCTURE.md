# 目标代码组织结构

> 状态：目标结构持续演进；M0-01..M0-09 与 G0-01..G0-07 已达到开发`accepted`，G0技术结论为Go，允许进入M1。制品发布资格是休眠M8维度，当前仍`release_eligible=false`；这不把现有进程壳或G0 harness描述为可运行产品。

## 当前 M0/G0 局部基线（2026-08-15）

当前仓库只实现可真实构建、默认拒绝的最小子集：根 Cargo/pnpm/PowerShell/just 入口；三个无依赖 Rust 进程壳；Flutter 五端 runner；无 capability 的 M0 TypeScript Adapter harness；M0-02 exact toolchain；M0-03 的公共/三类internal proto、三端正式生成绑定、生成manifest，以及实际Hello/request/UI/event/snapshot/result Frame validator；M0-04 的强类型ID、稳定PublicError、显式deadline、有界idempotency registry和可checkpoint的单赢家Operation record。Client/Server Hello现闭合version、UUID、required/optional/enabled capability、permission、heartbeat与1 MiB frame预算；reserved ZSTD不可advertise。测试专用stdio进程链现由Dart生成ClientHello、Rust返回零capability/permission的ServerHello/Heartbeat/typed refusal、TypeScript正式绑定验证并交给Flutter reducer；它不创建listener或产品session。event桥接16类typed identity与连续cursor；snapshot/result在`COMPRESSION_NONE`下以16 MiB硬上限固定组装实际字节，校验part与整包SHA-256，通过官方protobuf decoder后再验证typed identity/edit digest并允许Installed或结果可读；ZSTD尚无exact有界解码器，默认拒绝。effectful/nested actual Frame现在只能生成绑定CommandMeta与matrix requirements的`FrameEffectIntent`，静态grant不满足即拒绝且runtime-ready恒false；M0-04基础库尚未接入认证session、资源ownership、dynamic precondition、持久store或任何副作用执行。Code-OSS `1.133.0` exact lock、canonical actor/startup inventory、G0-03 分类/typed-refusal registry，以及 G0-04 fixture、官方 VSIX、G0-05 临时 Rust UDS harness、G0-06 reference/golden和G0-07 clean最小构建均已存在。仍没有产品状态持久化或advertised capability；真实transport/session registry归M1，G0技术Go不等于产品Runtime或发布验收。

M0-02 已创建 `rust-toolchain.toml`、`.fvmrc`、机器可读工具链清单、Linux image定义，以及仓库内owner-only用户态Linux exact bootstrap/一命令quick入口；后者默认禁网，只有显式许可或digest离线资产才可补齐工具/锁定依赖，且在quick前后执行零浏览器门。M0-03 已创建 `buf.yaml`/`buf.gen.yaml`、三个 internal v1 proto、精确生成器锁、27-input manifest、Rust公共+internal、Dart公共、TypeScript公共+Adapter internal绑定及对应只读consumer边界。生成manifest schema v2将CRLF规范化为LF并拒绝孤立CR，source/output的bytes与SHA-256均按该规范字节计算；Linux与Windows由同一官方Write/Check入口复验。三端生成validator覆盖Hello capability协商、request、UI/QuickPick、event、snapshot与result实际Frame路由；模板本身进入manifest并受漂移门保护；Snapshot/Result的NONE字节组装、SHA-256与官方decode已在生成代码内闭合。Frame到真实transport/session归M1，ZSTD有界decoder及effect guard运行时准入归后续M1/M2，不再作为M0-03生成基线的隐藏完成条件。

M0-04 已创建 `server/crates/ids`、`error` 与 `operation-host`。它们只有workspace内 `protocol/ids/error` 依赖：不访问网络、文件、数据库、系统时钟、RNG或Runtime；幂等checkpoint只恢复canonical有序且已终结的有界entry，Operation checkpoint只恢复满足时间、revision、state/outcome与安全文本不变量的record。取消准入绑定owner或kind权限、workspace generation和exact state revision；`Cancelling`一旦赢得就不再允许success/failure覆盖，cleanup资源归零后才发布Cancelled，terminal重试返回首次winner。该内存模型不是durable事务或并发store实现。

M0-05 已创建 `server/crates/store`、`tools/sqlite-fixture-generator`、`tests/fixtures/sqlite`及Write/Check入口。它只实现通用migration机制和fixture-only catalog：exact SQLite `3.45.0`、100-byte header预检、每store独立identity/version/history、同一IMMEDIATE transaction内DDL/validator/history/version、不可逆步骤create-new backup/restore及有界故障注入。Control Plane、document、agent、audit、export的产品业务schema仍为`planned`，没有持久化Runtime producer或capability。

M0-06 已创建 `server/crates/observability`：typed trace、闭合低基数metric、canonical audit v1/hash-chain/checkpoint、固定redaction token与有界create-new本地NDJSON bundle。它没有自由文本字段、网络exporter、collector、后台线程、系统时钟/RNG、产品producer或audit持久表；业务audit schema/sink仍为`planned`。

M0-07 已创建`supply-chain/evidence-policy-v1.json`、正式evidence generator/Write/Check入口和隔离Git合同测试。只有40-hex clean HEAD、dirty=false、bounded source projection与全部required lock存在时才能生成；本地clean提交已完成真实Write/Check。`code-oss/extensions.lock.json`在M0保持空、禁止运行时安装且不可发布，固定扩展选择与认证仍归M4-06。四类骨架始终`release_eligible=false`，不替代M8完整SBOM/license/signature/provenance或consumer验证。

M0-08 已创建`governance/change-control-v1.json`、`.github/CODEOWNERS`和ADR索引/合同门；它绑定权威路径、协议三联件、风险规则与owner slug。GitHub team binding仍为`pending-repository-admin`，本地文件不等于在线审批或branch protection生效。

M0-09 将M0-03矩阵closure提升为独立零漂移工作包，并补强`schema_version`、unknown-case、default-deny与capability-union四个fail-closed scalar。默认allow、漏route、新oneof和生成物漂移均由隔离mutation拒绝；M0退出stdio链另验证三进程实际交换与capability-request拒绝，没有因此新增任何协议case、产品transport或advertised capability。

```text
VSCode_mobile/
├── AGENTS.md
├── README.md
├── rust-toolchain.toml                M0-02 建立 exact pin
├── Cargo.toml                         Rust workspace
├── buf.yaml / buf.gen.yaml             M0-03 已建立（无远程module依赖，故无buf.lock）
├── pnpm-workspace.yaml / pnpm-lock.yaml
├── .fvmrc 或受控 Flutter SDK 配置     M0-02 建立
│
├── client/flutter/                    Flutter 五端原生客户端
│   ├── android/ ios/ windows/ macos/ linux/
│   ├── lib/app/                       启动、路由、主题、依赖装配
│   ├── lib/auth/                      OIDC PKCE、设备会话
│   ├── lib/protocol/                  生成 DTO 的手写适配层
│   ├── lib/session/                   连接、恢复、心跳、能力协商
│   ├── lib/workspace/                 文件树、工作区生命周期
│   ├── lib/editor/                    原生编辑器与文档副本
│   ├── lib/language/                  补全、诊断、Hover、Code Action
│   ├── lib/terminal/                  VT 渲染和终端控制权
│   ├── lib/scm/                       Git/SCM原生界面
│   ├── lib/debug/                     DAP原生界面
│   ├── lib/agent/                     Agent会话、工具审批、Diff
│   ├── lib/state_sync/                快照、事件游标、作用域状态
│   └── test/ integration_test/
│
├── server/                            Rust workspace crates
│   ├── crates/ids/                    M0-04 强类型 ID、版本、deadline
│   ├── crates/protocol/               生成绑定与手写校验
│   ├── crates/error/                  M0-04 稳定错误投影
│   ├── crates/config/                 严格版本化配置
│   ├── crates/auth/                   OIDC、会话、权限
│   ├── crates/store/                  M0-05 SQLite migration/backup harness；PostgreSQL仍planned
│   ├── crates/event-log/              事件、快照、压缩
│   ├── crates/state-sync/             多端游标和状态作用域
│   ├── crates/workspace-manager/      生命周期与调度
│   ├── crates/export-coordinator/     容器外导出状态机、volume lock
│   ├── crates/artifact-store/         原子publish、ACL、下载session
│   ├── crates/runtime-api/            最小特权运行时契约
│   ├── crates/runtime-docker/         Docker-compatible OCI适配
│   ├── crates/operation-host/         M0-04 长操作基础模型、取消、终态
│   ├── crates/observability/          M0-06 typed trace/metric/audit/redaction/bundle基线
│   ├── crates/gateway/                HTTPS/WSS、下载、限流
│   ├── bins/ide-server/               非特权控制面
│   ├── bins/runtime-agent/            独占容器 socket 的特权代理
│   └── bins/windows-host/              Windows Service，仅启动/代理WSL2
│
├── workspace-agent/                   Coordinator容器内 Rust workspace
│   ├── crates/document/               Rope、版本、编辑日志、保存
│   ├── crates/filesystem/             路径、watch、上传/下载
│   ├── crates/terminal/               Execution PTY编排、scrollback、控制权
│   ├── crates/git/                    Execution Git编排、状态、Diff、提交
│   ├── crates/export-prepare/         writer冻结后的drain/fsync/watermark
│   ├── crates/extension-supervisor/   Adapter/ExtHost生命周期
│   ├── crates/agent/                  任务、工具、审批、checkpoint
│   └── bin/workspace-agent/
│
├── code-oss/
│   ├── upstream.json                  commit、hash、license
│   ├── patches/                       最小补丁序列
│   ├── headless-adapter/              TypeScript MainThread适配器
│   ├── extension-fixtures/            合同测试 VSIX
│   ├── compatibility/                 API映射清单与结果
│   └── extensions.lock.json           构建期固定扩展集合
│
├── protocol/
│   ├── ide/v1/ide.proto               客户端公共协议
│   ├── ide/v1/capability-matrix.yaml  顶层/嵌套case、request-response、stream/transfer/UI授权闭包
│   ├── internal/{runtime,workspace,adapter}/v1/  三个隔离IPC包
│   ├── generators.lock.json / generated-manifest.json
│   └── fixtures/                      固定向量与畸形语料
│
├── runtime/
│   ├── images/coordinator/            仅Workspace Agent的可信镜像
│   ├── images/execution/              Code-OSS/扩展/PTY/工具链镜像
│   ├── images/wsl2-rootfs/            Windows专用发行版构建
│   ├── entrypoint/
│   ├── exporter-helper/               仅导出时启动的临时第三容器镜像；RO项目卷、无网络/authority
│   ├── mount-matrix.yaml              UID、ro/rw、exec、quota、导出/备份
│   └── profiles/                      Rust/TS/Python等工具链profile
│
├── deploy/
│   ├── linux/systemd/
│   ├── linux/config/
│   ├── windows/installer/
│   ├── windows/service/
│   └── windows/wsl2/
│
├── Knowledge/                         唯一权威文档目录
├── governance/                        M0-08 owner/path/workflow机器合同
├── agents_work/tasks/                 M0/G0任务记录（当前无BOARD或独立交接矩阵文件）
├── scripts/                           统一验证、构建、发布、恢复
├── tools/sqlite-fixture-generator/    M0-05 真实SQLite fixture与manifest生成器
├── supply-chain/                      M0-07 evidence策略；完整SBOM/license/signature/provenance仍归M8
├── operations/                        metrics、alerts、dashboard、runbook
└── tests/
    ├── fixtures/sqlite/               M0-05 受manifest保护的真实旧库/拒绝fixture
    ├── protocol/ codeoss/ reconnect/ editor/
    ├── container/ export/ security/ agent/
    └── soak/
```

## 依赖方向

```text
Flutter UI
  -> protocol/client service interfaces
  -> TLS + Protobuf over WebSocket/HTTPS
  -> Rust ide-server
  -> runtime-agent (narrow privileged API; 双容器生命周期)
  <- coordinator/workspace-agent发起的mTLS control stream
  -> instance-scoped IPC
  -> execution/headless-adapter
  -> Code-OSS Extension Host
  -> fixed VSIX extensions
```

- `headless-adapter` 可依赖固定 Code-OSS 内部模块；其他自有模块不得直接依赖 Code-OSS 内部类型。
- `runtime-agent` 可访问容器引擎 socket；`ide-server`、工作区容器和客户端均不可访问。
- `workspace-agent` 是所有产品 API 介导编辑的唯一 ACK/版本协调者。Extension Host/终端可直接写项目卷；这些写入必须由 watcher+hash 转为 external revision/conflict，不能静默覆盖。
- `workspace-agent`位于Coordinator容器，`headless-adapter`、Extension Host/工具位于Execution容器；authority、extension-state、workspace、IPC与credential endpoint按`runtime/mount-matrix.yaml`隔离。
- 导出由控制面的 `export-coordinator` 持久拥有，Runtime Agent 管理 pause/helper；被冻结的 Workspace Agent 只提供 prepare checkpoint，不负责 finally/unpause。
- 公共协议与内部协议使用不同 package/major，避免特权 API 暴露给客户端。
