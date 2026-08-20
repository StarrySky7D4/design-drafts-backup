# 登录流程混乱治理 — 具体实施方案（代码级）【已实施】

> **生成时间**：2026-07-07
> **状态**：✅ **已实施**（登录流程重构已落地并通过审查修复，见 [`Knowledge/logs/2026-07-07-auth-flow-rewrite.md`](../Knowledge/logs/2026-07-07-auth-flow-rewrite.md)）。本文为代码级实施记录，保留备查。
> **目标分支**：`rewrite/auth-flow`
> **文档定位**：本文保留登录生命周期重写的**代码级实施记录**：改哪些文件、改成什么签名、按什么顺序改、如何验证。当前账号行为和部署以 [账号与发现指南](../guides/account-and-discovery.md) 为准。
> **性质**：完整实施记录；不要以当前架构短摘要覆盖其中的重构理由、失败处理和验证细节。
> **核实基准**：基于当前工作区 `f042c6b` 的真实源码逐行核实（行号见下表）。

---

## 0. 与 AUTH_REWRITE_PLAN 的差异（本文新增）

AUTH_REWRITE_PLAN 诊断的 5 点（FRB 散落、God start()、三路径、_svc 重建、_started 脱节）**全部属实**。逐行核实源码后，本文**额外发现 4 个隐患**（plan 未提）：

| # | 隐患 | 证据（行号） | 影响 |
|---|------|------|------|
| X1 | **双重 Ed25519 密钥对**：`AppState.start()` 与 `FrbNativeService.start()` 各自生成一个，互不相同 | `app_state.dart:539` `_localKeyPair = FileCrypto.generateKeyPair()`；`frb_native_service.dart:113` `await _initKeyPair()` | 群组文件签名/验签语义不一致（ AppState 用 `_localKeyPair` 签 hash 于 `:789`；Frb 用自己的于 `sign()` `:501`）。到底谁是"本节点身份签名"？两套密钥对会让对端无法稳定验签 |
| X2 | **`_sub ??=` 订阅脆弱**：事件订阅用"若已存在则跳过"，依赖 signOut 先 cancel | `app_state.dart:545` `_sub ??= _svc.events.listen(_onEvent)`；signOut `:484-485` cancel | 若 start() 失败、或未来出现"不重建 service 的登出"路径，`??=` 会复用旧 service 的死流或漏订阅新 service，事件静默丢失 |
| X3 | **全局单例与字段双重引用**：`NativeService.instance` 与 `AppState._svc` 都指向 service，signOut 只改两者之一就漂移 | `native_service.dart:133` static `instance`；`app_state.dart:39` `_svc`；signOut `:477-478` 同时改 | 任何写入只改一处都会让 UI（用 `_svc`）与全局（用 `instance`）看到不同 service，调试灾难 |
| X4 | **presence 双触发点靠 null 短路兜底**：`start()` 内与 `setServerSession()` 内各有一个 `_publishAddr()` 触发点，当前每条路径"恰好"只成功 1 次，全靠 `_publishAddr` 内 `client==null/token==null` 早返回 | `frb_native_service.dart:119-121` 与 `:198-200`；短路于 `:207-209` | 任何人删掉短路检查、或调整顺序，立即双发/竞态。这正是 plan 说的"靠副作用"，但 plan 没指出**当前正确性纯靠防御代码维持**，比想象的更脆 |

---

## 1. 现状核实（代码级事实表，含行号）

### 1.1 NativeService 抽象接口（`native_service.dart`）

| 方法 | 行号 | 级别 | 备注 |
|------|------|------|------|
| `start({bindAddress})` | `:138` | 会话 | God method 入口 |
| `setServerSession(session)` | `:155` | 账号 | 注释仍写"publishes presence and heartbeats"（心跳已废弃，注释过时） |
| `dispose()` | `:290` | — | 释放全部资源，无 stop-only |

**结论**：接口缺少 `startNode/stopNode/bindServerClient/publishPresence/isNodeRunning` 五个方法——这是重构的契约面。

### 1.2 FrbNativeService 实现（`frb_native_service.dart`）

| 成员 | 行号 | 现状 |
|------|------|------|
| `_node` / `_server` / `_session` | `:30/:55/:56` | 会话状态混在同一对象 |
| `_frbReady` + `ensureFrbInitialized()` | `:80-94` | static 进程级守卫，**逻辑上属于"进程级"，却挂在会话级类上** |
| `_ready` | `:65` | service 内部 ready 标志，**与 AppState._started 是两套** |
| `start()` | `:96-135` | 7 件事，含 FRB init / logStream / createNode / keypair / localAddr / 条件 publish / polling |
| `dispose()` | `:138-146` | 仅 close+close events，**无独立 stopNode** |
| `setServerSession()` | `:192-201` | 建 client + 条件 publish |
| `_publishAddr()` | `:206-217` | 真正发 `setAddr`，靠 null 短路 |

### 1.3 AppState 编排（`app_state.dart`）

| 成员 | 行号 | 现状 |
|------|------|------|
| `_svc` / `instance` | `:39` / `native_service.dart:133` | 双重引用（X3） |
| `_sub` | `:40` | `??=` 订阅（X2） |
| `_started` / `isStarted` | `:57-58` | 标志脱节 |
| `_localKeyPair` | `:55/:539` | 与 service 重复生成（X1） |
| `restoreSession()` | `:67-72` | 仅设本地字段（无副作用） |
| `applySession()` | `:76-83` | 设本地字段 **+** `setServerSession`（有副作用） |
| `loginAccount()` / `registerAccount()` | `:406/:383` | 都 → `_installSession()` |
| `_installSession()` | `:426-459` | lock → 构造 session → 复用判断 → `start()` → `applySession()` → persist |
| `signOut()` | `:462-489` | logout → dispose → **rebuild service** → reset |
| `start()` | `:528-551` | `_svc.start()` + 生成 keypair + `??=` 订阅 |

### 1.4 三条调用路径（已核实，当前时序）

```
路径 A 手动登录:
  LoginPage._submit → loginAccount → _installSession
    → acquireLoginLock → 构造 session → (复用判断) → start()
    → applySession() → setServerSession() → [_localAddress 已非空] → _publishAddr() ✓(1次)
路径 B 自动登录 (main.dart:_tryAutoLogin):
  loadPersistedSession → acquireLoginLock → restoreSession(仅设字段)
    → start() → [start内 _session!=null 但 _server==null] → _publishAddr() 短路返回
    → applySession() → setServerSession() → [_localAddress 已非空] → _publishAddr() ✓(1次)
路径 C 注册: registerAccount → _installSession → 同路径 A
```

**关键**：三条路径目前都"恰好"只发 1 次 presence，但路径 B 的正确性依赖 `_publishAddr` 的 `client==null` 早返回（X4）。这是脆弱的巧合正确性。

---

## 2. 目标状态机与不变量

### 2.1 三层 + 单向依赖（不变）

```
ProcessLayer   (main.dart)         ── FrbRuntime.init()  一次性
   │ 单向
NodeLayer      (FrbNativeService)  ── startNode/stopNode  可重启，不知账号
   │ 单向
AuthLayer      (AppState)          ── completeLogin/teardown  账号编排
```

### 2.2 节点生命周期状态机（新）

```
                ┌──────────┐
   init ───────▶│ Stopped  │
                └────┬─────┘
            startNode │ ▲ stopNode
                     ▼ │
                ┌──────────┐
                │ Running  │── bindServerClient ──┐
                └────┬─────┘                       ▼
                  stopNode               ┌──────────────┐
                     │                   │  Bound       │ (有 session)
                     ▼                   └──────┬───────┘
                ┌──────────┐            publishPresence │
                │ Stopped  │◀───────────────────────────┘
                └──────────┘
```

### 2.3 强制不变量（重构后必须成立，用作 review 检查表）

| ID | 不变量 | 验证手段 |
|----|--------|------|
| INV-1 | `RustLib.init()` 全进程只调 1 次，入口唯一在 `FrbRuntime.init()` | grep 全仓 `RustLib.init()` 应只 1 处 |
| INV-2 | `startNode()` 幂等：Running 态再调为 no-op；返回前 `_node!=null` | 单测 + isNodeRunning 断言 |
| INV-3 | presence 只有一个触发点：`publishPresence()`，且仅 AuthLayer 显式调用 | grep `_publishAddr`/`setAddr`，调用方仅 1 处 |
| INV-4 | 节点与账号解耦：`startNode` 不读 `_session`；`bindServerClient` 不读 `_node?.localAddr` 副作用 | 方法体审查 |
| INV-5 | 单一 service 引用：`AppState._svc` 与 `NativeService.instance` 永远指向同一对象，由 1 个 setter 统一维护 | 引入 `_setService()` 私有方法，grep 赋值点 |
| INV-6 | 单一 Ed25519 密钥对：签名/验签只用 service 持有的那一份（X1） | grep `generateKeyPair`，仅 1 处 |
| INV-7 | 事件订阅随节点生命周期：`startNode` 建、`stopNode` 拆（非 `??=`） | 审查 startNode/stopNode |
| INV-8 | 无 `_started` 布尔，状态查询一律走 `service.isNodeRunning` | grep `_started` 应为 0 |

---

## 3. 接口契约（新 NativeService 抽象）

> 改动 `native_service.dart`。所有方法都加文档契约，Mock 与 Frb 双实现。

```dart
abstract class NativeService {
  static NativeService instance = MockNativeService(); // 仍可变，但赋值统一走 AppState._setService

  // ── 节点生命周期（NodeLayer）──
  /// 绑定 P2P 节点到 [bindAddress]，启动轮询/日志订阅/密钥对。幂等：
  /// 已 Running 时直接返回当前 localAddress。不触碰账号/presence。
  Future<BridgeResult<String>> startNode({String bindAddress = '0.0.0.0:0'});

  /// 停止节点：关 _node、停轮询、停日志订阅、关事件流前 drain。幂等。
  Future<void> stopNode();

  /// 节点是否在运行（取代 AppState._started）。
  bool get isNodeRunning;

  /// 本机拨号地址（startNode 后非空）。
  String get localAddress;

  /// 本节点 Ed25519 公钥（startNode 后非空，单一密钥对来源，INV-6）。
  String get localPublicKey;

  Stream<BridgeEvent> get events;

  // ── 账号绑定（AuthLayer 显式调用）──
  /// 安装 server 会话：仅建 RustServerClient + 存 token，**不发布 presence**。
  /// 幂等：重复调用同 session 为 no-op；换 session 先释放旧 client。
  void bindServerClient(ServerSession session);

  /// 把当前 localAddress 发布到服务器地址簿。前置：startNode + bindServerClient 均完成。
  /// 唯一 presence 触发点（INV-3）。
  Future<BridgeResult<void>> publishPresence();

  // …（connect/send*/group 等业务方法保持不变）…

  // ── 兼容期桥接（重构过渡用，最终移除）──
  @Deprecated('用 startNode 代替；过渡期内部转发')
  Future<BridgeResult<String>> start({String bindAddress = '0.0.0.0:0'})
      => startNode(bindAddress: bindAddress);
  @Deprecated('用 bindServerClient + publishPresence 代替')
  void setServerSession(ServerSession s) { bindServerClient(s); }

  /// 释放全部资源（= stopNode + 清账号态）。仍保留供 signOut/退出用。
  Future<void> dispose();
}
```

> **删除**：旧 `start`、`setServerSession` 在过渡期用 `@Deprecated` 转发，全仓切换完成后删除。Mock 实现新增方法全部 no-op 或最小模拟（见 §6）。

---

## 4. 实现级设计（Dart 代码骨架）

### 4.1 新增 `FrbRuntime`（进程级，独立文件 `frb_runtime.dart`）

```dart
/// 进程级一次性初始化。从 FrbNativeService 抽离，消除"static 守卫挂在会话类上"。
class FrbRuntime {
  static bool _initialized = false;
  static Completer<void>? _completer;

  /// 幂等。并发调用共享同一 Completer。
  static Future<void> init() async {
    if (_initialized) return;
    if (_completer != null) return _completer!.future;
    final c = Completer<void>();
    _completer = c;
    try {
      await RustLib.init();          // INV-1：全进程唯一调用点
      _initialized = true;
      c.complete();
    } catch (e) {
      _completer = null;             // 失败可重试
      c.completeError(e);
    }
    return c.future;
  }
}
```

`main.dart` 改为 `await FrbRuntime.init();`，删除对 `FrbNativeService.ensureFrbInitialized` 的引用。`FrbNativeService` 删除 `_frbReady`/`ensureFrbInitialized`（迁移到 FrbRuntime）。

### 4.2 FrbNativeService 重构（关键方法骨架）

```dart
class FrbNativeService implements NativeService {
  frb.RustNode? _node;
  StreamSubscription? _logSub;
  Timer? _pollTimer;
  rsc.RustServerClient? _server;
  ServerSession? _session;
  String _localAddress = '';
  String _localPublicKey = '';
  SimpleKeyPairData? _keyPair;       // INV-6：唯一密钥对
  // …缓存字段不变…

  @override
  bool get isNodeRunning => _node != null && _pollTimer != null;

  @override
  Future<BridgeResult<String>> startNode({String bindAddress = '0.0.0.0:0'}) async {
    if (isNodeRunning) return BridgeResult.ok(_localAddress);  // INV-2 幂等
    try {
      await FrbRuntime.init();                 // 进程级（不再自己持有 _frbReady）
      _logSub = frb.logStream().listen(_onLog);
      _node = await frb.createNode(bindAddress: bindAddress);
      _keyPair = await FileCrypto.generateKeyPair();          // INV-6 唯一处
      _localPublicKey = base64Encode(await _keyPair!.extractPublicKeyBytes());
      _localAddress = await _node!.localAddr();
      _ensureDownloadDir();
      _startPolling();                         // INV-7 订阅在 startNode 建
      unawaited(_refreshGroupCache());
      return BridgeResult.ok(_localAddress);
    } catch (e) {
      await _cleanupNodePartial();             // 失败回滚半启动状态
      return BridgeResult.fail('startNode failed: $e');
    }
  }

  @override
  Future<void> stopNode() async {              // INV-7 订阅在 stopNode 拆
    _pollTimer?.cancel(); _pollTimer = null;
    await _logSub?.cancel(); _logSub = null;
    await _node?.close(); _node = null;
    _localAddress = '';
    // 注意：不清 _server/_session（账号态独立，见 INV-4）
  }

  @override
  void bindServerClient(ServerSession session) {
    if (_session?.token == session.token) return;   // 幂等：同 token no-op
    _session = session;
    _server = rsc.createServerClient(baseUrl: session.baseUrl, serverCertDer: null);
    // 不再在这里 publish（INV-3/INV-4）
  }

  @override
  Future<BridgeResult<void>> publishPresence() async {   // INV-3 唯一触发点
    final client = _server, token = _session?.token;
    if (!isNodeRunning || client == null || token == null || _localAddress.isEmpty) {
      return BridgeResult.fail('presence prerequisites not met');
    }
    _lanIp ??= await rsc.detectLanIp();
    final addr = _dialableAddr(_localAddress);
    try {
      await client.setAddr(token: token, addr: addr);
      _registeredAddr = addr;
      return BridgeResult.ok(null);
    } catch (e) {
      return BridgeResult.fail('set_addr failed: $e');
    }
  }

  @override
  Future<void> dispose() async {               // = stopNode + 清账号
    await stopNode();
    _server = null; _session = null; _registeredAddr = '';
    await _eventCtl.close();
  }
}
```

### 4.3 AppState 编排（单一入口 completeLogin）

```dart
class AppState extends ChangeNotifier {
  NativeService _svc;
  StreamSubscription<BridgeEvent>? _sub;
  // 删除：_started, _localKeyPair（X1）

  bool get isStarted => _svc.isNodeRunning;     // INV-8：查询代替标志

  /// 统一登录入口（手动登录 / 注册 / 自动登录 都走这里）。
  /// [performAuth]：返回 AuthResult（手动登录/注册走 RPC；自动登录直接返回持久化的）。
  Future<BridgeResult<String>> completeLogin({
    required String serverUrl,
    required Future<rsc.AuthResult> Function() performAuth,
  }) async {
    _setBusy(true);
    try {
      final auth = await performAuth().timeout(const Duration(seconds: 12));

      // 1. 重复登录锁
      final lockErr = acquireLoginLock(auth.userId);
      if (lockErr != null) { _setError(lockErr); return BridgeResult.fail('duplicate_login'); }

      final session = ServerSession(
        baseUrl: serverUrl, userId: auth.userId, nickname: auth.nickname, token: auth.token,
      );

      // 2. 节点启动（幂等，同账号复用）
      final nodeRes = await _svc.startNode().timeout(const Duration(seconds: 15));
      if (!nodeRes.isOk) {
        releaseLoginLock();
        return BridgeResult.fail(nodeRes.error ?? 'node start failed');
      }
      _localAddress = nodeRes.value ?? '';

      // 3. 绑定账号（不发布）
      _svc.bindServerClient(session);

      // 4. 显式发布 presence（此时 node+client 都就绪，INV-3）
      await _svc.publishPresence();

      // 5. 装载本地态 + 持久化
      _installLocalState(session);
      _sub ??= _svc.events.listen(_onEvent);    // 过渡保留 ??=；§8 任务 T7 改为显式
      _persistSession();
      notifyListeners();
      return BridgeResult.ok(auth.userId);
    } catch (e) {
      _setError('$e');
      return BridgeResult.fail('$e');
    } finally {
      _setBusy(false);
    }
  }

  void _installLocalState(ServerSession s) {
    _session = s; _serverUrl = s.baseUrl; _userId = s.userId; _alias = s.nickname;
  }

  // 旧入口转为 completeLogin 的薄封装：
  Future<BridgeResult<String>> loginAccount({required String serverUrl, required String loginAccount, required String password}) {
    final c = rsc.createServerClient(baseUrl: serverUrl, serverCertDer: null);
    return completeLogin(serverUrl: serverUrl, performAuth: () => c.login(loginAccount: loginAccount.trim(), password: password));
  }
  // registerAccount 同理，performAuth 调 c.registerAccount(...)。
}
```

### 4.4 自动登录收敛

`main.dart:_tryAutoLogin` 与 `AppState.restoreSession` 合并为：

```dart
// main.dart
final session = await AppState.instance.loadPersistedSession();
if (session == null) { /* → 登录页 */ return; }
if (AppState.instance.acquireLoginLock(session.userId) != null) { /* → 提示 + 登录页 */ return; }
final res = await AppState.instance.completeLogin(
  serverUrl: session.baseUrl,
  performAuth: () async => rsc.AuthResult(   // 持久化 token 直接构造，无 RPC
    userId: session.userId, nickname: session.nickname, token: session.token,
  ),
);
// res.isOk → MainPage；否则清持久化 + 释放锁 → 登录页
```

`AppState.restoreSession` 与 `applySession` **删除**，职责并入 `completeLogin` + `_installLocalState`。

### 4.5 signOut → teardown（不复用 service 重建）

```dart
Future<void> signOut() async {
  // 1. 服务器登出（best-effort）
  if (_svc is FrbNativeService && _session != null) {
    try { await (_svc as FrbNativeService).logout(_session!.token); } catch (_) {}
  }
  // 2. 停节点（关 socket、停轮询、停订阅）
  await _svc.stopNode();
  // 3. 清账号态
  _session = null; _serverUrl = ''; _userId = ''; _alias = '';
  await _sub?.cancel(); _sub = null;            // 显式拆订阅（X2）
  // 4. 锁 + 持久化
  releaseLoginLock(); await clearPersistedSession();
  // 5. UI 态
  _peers.clear(); _groups.clear(); _localAddress = '';
  notifyListeners();
  // 不再 rebuild service：下次 completeLogin 的 startNode 幂等，service 可复用
}
```

> **决策点**：保留 service 复用还是 rebuild？复用更符合 INV-2 幂等语义，且消除 X3 的双重赋值风险。若仍倾向 rebuild，必须通过唯一 setter `_setService(s)` 同时更新 `_svc` 与 `NativeService.instance`（见 §7 任务 T6）。

---

## 5. 文件改动清单（精确）

| 文件 | 改动类型 | 要点 |
|------|------|------|
| `flutter/lib/services/frb_runtime.dart` | **新增** | `FrbRuntime.init()`，抽离 `_frbReady` |
| `flutter/lib/services/native_service.dart` | 改接口 | 新增 `startNode/stopNode/isNodeRunning/bindServerClient/publishPresence`；`start/setServerSession` 标 `@Deprecated` 转发；修正 `setServerSession` 过时注释（心跳已废弃） |
| `flutter/lib/services/frb_native_service.dart` | 重构 | §4.2 骨架；删 `ensureFrbInitialized/_frbReady`；`_initKeyPair` 改为 startNode 内唯一生成（INV-6） |
| `flutter/lib/services/mock_native_service.dart` | 适配 | 新方法最小实现（startNode 模拟端口、stopNode 置 `_started=false`、bindServerClient no-op、publishPresence no-op、isNodeRunning 返 `_started`） |
| `flutter/lib/services/app_state.dart` | 重构 | §4.3/4.5；删 `_started/_localKeyPair/restoreSession/applySession/_installSession`；`loginAccount/registerAccount` 改为 completeLogin 薄封装；`signOut` 改 teardown 语义；`start()` 删除（被 startNode 取代） |
| `flutter/lib/main.dart` | 改 | `FrbRuntime.init()` 替换 `FrbNativeService.ensureFrbInitialized`；`_tryAutoLogin` 改用 completeLogin（§4.4） |
| `flutter/lib/ui/login/login_page.dart` | 基本不动 | 仍调 `loginAccount`（内部已是 completeLogin）；仅核对错误展示 |
| `flutter/lib/ui/login/register_page.dart` | 基本不动 | 同上 |
| `flutter/lib/ui/main/self/*`（含登出入口） | 核对 | 调 `signOut` 的地方行为不变（返回 Future） |
| `Knowledge/flutter/services/frb_native_service.md` | 同步 | 重写"生命周期"章节 |
| `Knowledge/flutter/services/app_state.md` | 同步 | 更新登录编排描述 |
| `Knowledge/guides/account-and-discovery.md` | 同步 | 移除 presence/heartbeat 残留描述 |
| `Knowledge/logs/2026-07-07-auth-flow-rewrite.md` | 同步 | 标注实施完成、引用本文 |

---

## 6. Mock 适配要点（避免遗漏）

`MockNativeService` 当前 `start()` 模拟端口+种子数据、`setServerSession` no-op。适配：

```dart
@override bool get isNodeRunning => _started;
@override Future<BridgeResult<String>> startNode({String bindAddress = '0.0.0.0:0'}) async {
  if (_started) return BridgeResult.ok(_localAddress);
  await Future.delayed(const Duration(milliseconds: 80));
  _localAddress = '127.0.0.1:${40000 + (_peerCounter % 5000)}';
  _started = true; _seedDemoData();
  return BridgeResult.ok(_localAddress);
}
@override Future<void> stopNode() async { _started = false; _peers.clear(); }
@override void bindServerClient(ServerSession s) {}   // no-op
@override Future<BridgeResult<void>> publishPresence() async => BridgeResult.ok(null);
```

---

## 7. 增量迁移步骤（每步可编译、可测、可单独提交）

> 每步结束运行：`flutter analyze` + `flutter test`。任一步红则停下修复，不累积债务。

**Step 1 — 抽 FrbRuntime（无行为变化）**
- 新增 `frb_runtime.dart`；`FrbNativeService.ensureFrbInitialized` 内部改为转发 `FrbRuntime.init()`，暂保留 static 方法作为兼容入口。
- `main.dart` 改调 `FrbRuntime.init()`。
- 验证：grep `RustLib.init()` 仅 FrbRuntime 1 处（INV-1）。

**Step 2 — 接口加新方法（双实现就位，旧方法不删）**
- `native_service.dart` 加 `startNode/stopNode/isNodeRunning/bindServerClient/publishPresence` 抽象方法 + `@Deprecated start/setServerSession` 默认转发实现。
- Frb/Mock 两个实现补全新方法（§4.2/§6），但**旧 start/setServerSession 暂保留原逻辑**。
- 验证：`flutter analyze` 无 warning（旧方法标 Deprecated 会产生 info，可接受）。

**Step 3 — 引入 completeLogin（并存）**
- AppState 加 `completeLogin`，`loginAccount/registerAccount` **新增**一条"内部调 completeLogin"的代码路径，用 feature flag（如 `static bool _useUnifiedLogin = true`）切换。
- 旧 `_installSession` 保留，flag=false 时走旧路径。
- 验证：两条路径都能跑通手动登录。

**Step 4 — 自动登录切换到 completeLogin**
- `main.dart:_tryAutoLogin` 改用 completeLogin（§4.4）；flag 控制下并存。
- 验证：自动登录场景通过。

**Step 5 — signOut 改 teardown 语义**
- signOut 改为 stopNode + 显式拆订阅（§4.5），flag 控制下并存旧 rebuild 路径。
- 验证：登出→重登场景通过。

**Step 6 — 统一 service 引用（X3）**
- 引入 `AppState._setService(s)`，同时赋 `_svc` 与 `NativeService.instance`；所有 service 赋值改走它。
- 验证：grep `NativeService.instance =` 与 `_svc =` 仅 `_setService` 内部。

**Step 7 — 切除过渡层（flag 永久 true，删旧代码）**
- 删 `_installSession/restoreSession/applySession/start()/_started/_localKeyPair`。
- 删 FrbNativeService 旧 `start()/setServerSession()`、`ensureFrbInitialized`。
- 删 NativeService `@Deprecated start/setServerSession`。
- `_sub ??=` 改为 startNode 内 `_sub = ...`、stopNode 内 `await _sub?.cancel(); _sub=null;`（X2）。
- 验证：INV-1~8 全部 grep 通过（§8 检查表）。

**Step 8 — 文档同步**
- §5 文档清单同步；更新 `Knowledge/logs/` 日志。

---

## 8. 不变量验证检查表（合并前 grep 断言）

```powershell
# INV-1: RustLib.init 唯一调用点
rg -n "RustLib\.init\(\)" flutter/lib            # 期望: 仅 frb_runtime.dart 1 行
# INV-3: presence 唯一触发
rg -n "_publishAddr|setAddr" flutter/lib/services/frb_native_service.dart  # publishPresence 内 1 处
# INV-6: 密钥对唯一生成
rg -n "generateKeyPair" flutter/lib              # 期望: 仅 startNode 内 1 处（+file_crypto 定义）
# INV-8: 无 _started 标志
rg -n "_started" flutter/lib/services            # 期望: 仅 mock 内部模拟用，AppState 0 处
# 双重引用消除
rg -n "NativeService\.instance\s*=" flutter/lib  # 期望: 仅 _setService 内
```

---

## 9. 回归测试矩阵

| # | 场景 | 入口 | 预期 | 当前是否覆盖 |
|---|------|------|------|------|
| R1 | 首次手动登录 | login_page | 进 MainPage，presence 发 1 次，session.json 落盘 | 部分 |
| R2 | 注册新账号 | register_page | 同 R1 | 部分 |
| R3 | 自动登录（有持久化） | app 冷启 | 跳过登录页进 MainPage，presence 发 1 次 | 部分 |
| R4 | 自动登录（无持久化） | app 冷启 | 进登录页，不启动节点 | ✅ |
| R5 | 自动登录被重复锁拦截 | 锁已存在 | 进登录页 + 警告，不启动节点 | ✅ |
| R6 | 登出 → 重登 | self 页 signOut | 旧节点 stopNode，新登录 startNode，无 socket 泄漏 | 需补 |
| R7 | 相同账号重登（未登出） | 登录态再 login | 复用节点，presence 刷新 1 次 | 需补 |
| R8 | 节点启动失败 | mock 注入 startNode fail | 锁释放，回登录页，无半启动态 | 需补 |
| R9 | presence 前置缺失 | 未 bind 就 publish | 返回 fail，不发 setAddr | 需补（单测） |
| R10 | 多次 startNode 幂等 | 连续调 2 次 | 第 2 次返回同地址，不重建节点 | 需补（单测） |
| R11 | signOut 后事件流 | 登出→新消息 | 不崩溃，新登录后恢复订阅 | 需补 |

**R8/R9/R10/R11 建议补 widget/单元测试**（放 `flutter/test/`），Mock 注入失败路径。

---

## 10. 风险与回滚

| 风险 | 缓解 |
|------|------|
| 接口改动同时影响 Frb+Mock，遗漏实现 → 编译红 | Step 2 一次性补全双实现，`flutter analyze` 门禁 |
| completeLogin 编排错误导致 presence 漏发/双发 | INV-3 grep + R1/R3/R9 测试；publishPresence 内前置断言 |
| service 复用导致跨账号状态残留（登出 A 登 B） | stopNode 不清账号态、teardown 清账号态；R6 验证；bindServerClient 幂等判断 token |
| 过渡期 flag 两套路径并存增加复杂度 | Step 3-5 每步小而独立，Step 7 果断删除旧路径，flag 生命周期 ≤ 1 个 PR |
| `_localKeyPair` 移除后，依赖它的 `signHashes(:789)` 行为变化 | X1 核实：signHashes 改用 `service.sign()`（Frb 内同一密钥对），单测覆盖签名-验签闭环 |

**回滚**：每步独立提交，任一步引入回归可 `git revert` 该步。Step 7 之前旧路径完整可走，flag=false 即回退。

---

## 11. 验收门禁（合并 rewrite/auth-flow → master 前全部通过）

1. `cd flutter && flutter analyze` 0 error
2. `cd flutter && flutter test` 全绿（含新增 R8-R11）
3. §8 不变量 grep 全部满足
4. R1-R11 手动/自动回归全过
5. `Knowledge/flutter/services/{frb_native_service,app_state}.md` 已更新
6. `Knowledge/guides/account-and-discovery.md` 无 presence/heartbeat 残留
7. 无 `_started`、无双 `generateKeyPair`、无双 `RustLib.init`

---

> 本方案聚焦"怎么做"。实施者按 §7 八步推进，每步对照 §2.3 不变量与 §9 测试矩阵验证。
