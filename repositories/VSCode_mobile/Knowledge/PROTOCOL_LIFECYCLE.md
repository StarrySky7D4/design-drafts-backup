# 协议、生成物与开发/发布生命周期

> 注册值以`PROTOCOL_REGISTRY.md`为事实源。本文定义变化如何分类、验证和发布。

## 0. 双层验收模型

- `development acceptance`决定里程碑推进。M0/G0已依据最终clean实现提交的Windows、自管Linux、G0与零浏览器证据accepted，允许进入M1。
- `artifact release eligibility`只决定能否分发制品。ADR-0001使M8当前`inactive`且`release_eligible=false`；`gate-full`、SBOM/license/signature/provenance/consumer verification、安装/商店、N/N-1与canary/soak不阻塞开发验收。
- 公开源码仍执行source-publication卫生门；开发accepted不豁免secret、用户数据、机器绝对路径、LICENSE/NOTICE/上游来源、lock/generated drift、cache/构建工件及零Electron/Chromium检查。该门当前为planned/non-green，不得把文档声明当成可执行通过证据。
- `governance/change-control-v1.json`的effect-surface是路径分类事实源。仅data-only或governance-contract变化可以在对应本地门通过后继承同一实现树证据；未知路径默认按product-runtime，代码、lock、生成器、协议、运行时和安全门变化必须重跑受影响门。

## 1. 变更前置

任何公开协议、内部特权协议、持久schema、Code-OSS actor映射、稳定错误、Runtime Release或生成边界变化必须：

1. 在注册表分配owner、编号/字段、状态和资源上限。
2. 分类为`breaking`、`additive`、`behavioral`或`data-only`；多个分类取最严格门。
3. 写producer/consumer、N-1/N/N+1、旧/新storage和Flutter/Rust/Adapter的矩阵。
4. 提供正向固定向量和拒绝向量，证明失败无副作用。
5. 更新生成物并执行零漂移检查。
6. 定义升级、回滚或只能前滚的恢复方式。

| 分类 | 判定 | 最低要求 |
|---|---|---|
| breaking | 正确旧端无法无歧义、安全互操作 | 新major/schema/API major、双向拒绝、支持窗口、迁移/回滚 |
| additive | 旧端能安全忽略且旧成功路径不变 | 已登记extension point、optional/required语义、缺能力拒绝 |
| behavioral | bytes未变但授权、错误、重试、资源或结果改变 | 兼容矩阵、发布说明、资源/零副作用回归；不安全则同时breaking |
| data-only | 只改已登记文档/任务状态，不含代码、lock、生成器、协议、运行时或安全门 | 机器路径分类；治理、repository baseline、零浏览器门；证明实现树与外部结果未变 |

## 2. Protobuf规则

- Field number只追加，永不复用、重排或改变既有语义。
- 删除字段先`reserved`名称与编号，至少跨支持窗口。
- 不使用proto隐式默认值表达安全关键的“已验证/已授权/已批准”；使用显式enum并拒绝UNSPECIFIED。
- 新oneof case只有在旧端收到该消息仍能安全忽略时才additive；必需命令的未知case必须拒绝。
- map不得作为canonical签名输入；需要hash/sign时定义排序、长度和domain separation的canonical encoder。
- 每个长度、repeated、嵌套深度和解压结果在分配前检查上限。
- 公共与internal package不互相import特权request。
- Frame的`delivery` oneof是唯一调度依据；body到delivery的矩阵进入Rust/Dart/TypeScript三端validator，禁止用字段是否为空猜测response/event/snapshot语义。

## 3. Major、Minor与Capability

- 固定字段、认证材料、ID语义、principal、资源上限意义或必需顺序改变时升级major。
- 同一major不得猜测两种布局、用异常回退旧解析或根据字段长度推断版本。
- minor只用于安全optional扩展；双方握手选择能力。
- required capability缺失在副作用前拒绝；连接/缓存失败不得自动降级到较低安全能力。
- 未知major、future schema和未登记Runtime组合一律fail-closed。
- `capability-matrix.yaml`同时覆盖顶层case、嵌套action、request→response/outcome、`StreamKey`动态继承、event/snapshot identity、UI subtype和transfer状态机；任何一层缺映射都在生成/CI阶段失败，运行时默认拒绝。
- 同一stream不得按capability过滤出cursor洞。首版DOCUMENT stream将正文、lease、diagnostics和semantic tokens作为一个恢复闭包，只有四项domain capability齐全才订阅/advertise；若需更细协商，必须分裂stream key、cursor和snapshot。

## 4. 事件与恢复

- 每个stream sequence单调递增；事件提交和sequence分配同一事务或同一Actor原子单元。
- `EditAck`表示服务端已持久接受编辑但不表示文件已经显式保存；文档同时报告canonical和saved version。`StreamAck`方向相反，只表示某客户端已消费到服务端实际投递frontier，绝不表示mutation或落盘成功。
- 客户端重复命令通过绑定principal/device/client incarnation/workspace/resource的idempotency key返回原结果；`request_id`不可代替幂等键。
- 事件压缩前生成并校验snapshot；早于保留窗口的客户端必须走snapshot，不拼接不完整历史。
- Snapshot包含typed payload schema、stream generation、inclusive high-water、part count/offset和完整hash；future schema、乱序/重复part、长度或hash不符全部拒绝且不得推进游标。
- Snapshot digest覆盖服务端实际发送的精确未压缩payload/part bytes；客户端先验bytes再按声明schema解析，不能用各语言重序列化结果做hash。part index/offset必须连续并精确覆盖声明长度，解压前先做总预算。
- Snapshot创建遵循tail-first barrier：先登记live tail，再冻结high-water cut，snapshot/replay覆盖到inclusive watermark，live只发送严格大于cut的事件。统一Snapshot envelope将`SnapshotPayload`分块，使用独立snapshot ID/index且不分配事件cursor；只有客户端校验terminal hash、原子安装并发送精确匹配的`SnapshotInstalled`后才接续tail。ACL/device撤销发送aborted、清空缓冲并关闭流。
- Agent/WorkspaceEdit等请求型大结果使用独立`ResultTransfer`，没有StreamKey/generation/high-water/`SnapshotInstalled`。启动transfer的短响应已经terminal；后续只按transfer ID路由，成功End或Aborted终结transfer本身，永不推进事件frontier。
- Language request branch与result branch逐项绑定；rename的prepare/apply候选结果互斥，continuation token继承原result family并绑定session/principal/document version/extension generation。
- 短请求、Snapshot/Result transfer和UI交互均有active-state表；cancel继承原对象的capability/身份，未知/跨session ID拒绝，cancel与正常terminal/End/Aborted/timeout/disconnect只有一个赢家。
- QuickPick分页绑定active UI request、selected recipient、extension generation、items handle和page token，不能由同owner observer枚举。
- LeaseState在响应、事件和四类内嵌snapshot位置都同时校验target branch与资源UUID，防止A资源状态安装到B资源。
- ACK必须属于当前session已授权、已订阅且实际投递的stream generation，单调并且不超过delivery frontier；禁止用future ACK释放未投递历史。`StreamReset(snapshot_required=true)`后由服务端自动开始snapshot，显式`RequestStreamSnapshot`仅用于失败重试。
- Stream reset是明确协议事件，不能静默丢事件后继续sequence。

## 5. 数据库迁移与恢复

- 每个store独立schema域，只向前推进。
- Validation、DDL/转换、索引和版本写回同一transaction；失败整体回滚。
- 使用真实旧fixture覆盖合法、非法、孤儿引用、重复open、并发、中断和修复重试。
- 不静默删除、重命名用户数据或生成替代ID。
- 已发布migration不可改写；用新repair/quarantine步骤。
- transaction rollback不等于备份。破坏性或不可逆迁移前创建一致备份并完成恢复演练。
- 高schema被写入后，旧binary不得自动downgrade或回写。

## 6. Code-OSS升级

Adapter与Extension Host是一个精确发布单元。升级步骤：

1. 固定新Code-OSS commit、Node和基础镜像。
2. 生成Extension Host protocol hash、actor和contribution inventory diff。
3. 对新增/删除/签名变化逐项分类为reuse/adapt/flutter bridge/unsupported。
4. 在新commit上从零应用最小patch。
5. 运行G0启动、fixture API、同commit纯Node reference/golden与零Electron/Chromium依赖/工件/进程树对照测试。
6. 运行全部bundled extension认证和Node native ABI检查。
7. 生成license、npm/extension SBOM和Runtime Lock。
8. 发布完整新workspace image并canary；禁止热换单个Extension Host文件。

Flutter客户端不与Code-OSS commit配对；Adapter和Extension Host必须精确配对。

## 7. 生成物

以下生成物不可手改：

- Rust/Dart/TypeScript Protobuf绑定。
- Actor/contribution inventory和protocol hash。
- Runtime Lock派生摘要。
- SBOM、许可证和兼容矩阵派生文件。

PR Gate在干净临时目录用固定生成器重建并比较。缺工具是失败，不是skip。生成器版本、输入hash和输出hash进入证据。

## 8. 稳定错误与Operation

- 稳定判断只使用error code/category，不解析message。
- Error code新增可additive；删除或改变触发条件是breaking/behavioral。
- Public details使用键allowlist和长度上限，不放内部路径、代码或token。
- Operation terminal恰好一次；cancel携带当前`expected_operation_revision`，只允许operation owner或该kind授权主体发起，并与commit通过同一持久仲裁决定唯一赢家。只有真实cleanup完成后才可见为cancelled。
- `OperationSnapshot`终态使用typed outcome oneof，状态与outcome必须一致；递归`Value`和大对象不得作为operation结果，大结果只能返回有界descriptor/reference。
- 调用方Future/连接drop不自动表示cancel。Operation host持有资源到真实终态。
- commit/cancel/deadline race必须有确定仲裁和故障测试。
- Terminal/Task/Debug/Testing/Export等domain cancel只是委托同一个OperationHost；domain状态和Operation终态在同一原子提交中发布，不得各自仲裁出不同终态。

## 9. 发布窗口（M8休眠时不适用）

首个稳定版后建议：

- 公共客户端协议支持当前N和N-1 minor/明确major窗口；不兼容major只提供清晰升级错误。
- 数据库binary读取窗口至少N/N-1；写入仅当前schema，具体由release manifest声明。
- Deprecated API至少保留一个minor和90天，除非安全紧急停止扩散。
- 每个release保存上一稳定客户端、数据库fixture、workspace image、Runtime Lock和扩展认证报告。

## 10. 发布清单（恢复制品发布时启用）

ADR-0001当前禁止分发制品。未来只有新的显式ADR恢复M8后，发布前才必须确认：

- 注册表、架构、风险和路线图同步。
- 协议固定/拒绝向量、生成物漂移、migration fixture和兼容矩阵通过。
- Code-OSS actor/contribution drift已处理，固定扩展全部认证。
- 容器、WSL2、断线、重启、export和operation fault矩阵通过。
- 每个工件有checksum、SBOM、license、signature、provenance和consumer验证。
- canary、停止扩散、binary/schema前滚/回滚和数据恢复演练完成。
- 证据绑定最终clean exact tag/commit，不能借用历史绿色HEAD。
