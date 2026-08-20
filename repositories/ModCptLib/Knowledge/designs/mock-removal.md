# MockNativeService 移除指南

> **生成时间**：2026-07-07
> **目标分支**：`rewrite/auth-flow`
> **性质**：完整 Mock 移除方案，独立维护。被 [身份与 FRB 演进](peer-identity-and-frb.md) §7.1 引用。
> **核实基准**：基于当前工作区（`flutter analyze` 0 error/0 warning）的逐行核实。

---

## 0. 背景：为什么现在可以移除 Mock

`MockNativeService` 是 UI 在「无 Rust 后端」阶段开发的纯 Dart 模拟。当前状态：

- **生产路径已完全切到 FRB**：`main.dart` 在 `main()` 里 `NativeService.instance = FrbNativeService()`，所有真实运行走 FRB。
- **Mock 仅剩两个用途**：
  1. `native_service.dart:133` 的 `static NativeService instance = MockNativeService()` —— 默认实例（进程启动后立即被 `main()` 覆盖）。
  2. `flutter/test/widget_test.dart` —— `pumpWidget(ModCptApp())` **不调用 `main()`**，因此 `NativeService.instance` 保持默认 Mock，测试隐式依赖它。
- **Mock 规模**：`mock_native_service.dart` 556 行，含自动回复、呼叫模拟、文件模拟等大量「演示用」逻辑，是当前最大的纯演示代码块。

**移除收益**：
- 删除 ~556 行不再被生产路径使用的演示代码，降低维护面。
- 消除「两个实现必须同步保持一致」的漂移风险（review 已发现 Mock `stopNode` 曾与 Frb 行为不一致）。
- 让 `NativeService.instance` 默认值语义清晰（不再是「演示态」）。

**移除风险**：
- **测试会崩**：`widget_test.dart` 隐式依赖默认 Mock。必须先引入测试替身（见 §3）。
- **AppState 单例初始化**：`static AppState instance = AppState()` 在类加载时构造，此时 `NativeService.instance` 是默认值。若默认改为 `late`/抛异常，会破坏非 `main()` 入口的启动。

---

## 1. Mock 的全部引用点（已核实）

| 位置 | 引用方式 | 移除影响 |
|------|------|------|
| `native_service.dart:38` | `part 'mock_native_service.dart';` | 删除此行 + 删文件 |
| `native_service.dart:133` | `static NativeService instance = MockNativeService();` | 默认实例需替换 |
| `native_service.dart:8-10` | 文档注释提及 Mock | 删除注释段 |
| `mock_native_service.dart`（全文 556 行） | 类定义 + `_MockGroup`/`_CachedFile` 辅助类 | 整文件删除 |
| `test/widget_test.dart:7` | 隐式依赖默认 Mock（不调 `main()`） | 必须先适配（§3） |
| `Knowledge/flutter/services/mock_native_service.md` | Knowledge 文档 | 删除 + 更新 `README.md` 索引 |
| `Knowledge/README.md:112` | 索引用「/」与 mock_native_service.md 并列 | 移除引用 |

**生产代码无任何直接引用 Mock 类**（grep `MockNativeService` 仅命中定义处和 native_service.dart）。

---

## 2. 移除方案选型

| 方案 | 做法 | 优点 | 缺点 | 推荐度 |
|------|------|------|------|------|
| **A. 完全删除** | 删 Mock 文件 + 把默认 instance 改为抛异常 + 测试用最小 Fake | 最彻底，代码最干净 | 测试需重写；默认 instance 失去「能跑」能力 | ★★★★（长期目标） |
| **B. 删除 Mock，保留默认实例为 Frb** | 删 Mock + `instance` 默认改为 `FrbNativeService()` + 测试注入 Fake | 生产路径无变化；测试隔离 | 默认 Frb 在无 FRB init 时会崩（FRB 未 init 不能创建节点） | ★★（默认值语义危险） |
| **C. Mock 降级为「测试专用 Fake」** | Mock 文件移到 `test/`，默认 instance 改为抛异常 | 演示逻辑不再进生产构建；测试仍可用 | Mock 仍存在，只是换位置 | ★★★（折中） |
| **D. 暂不移除** | 保持现状，仅标注 Mock 为 deprecated | 零风险 | 维护负担持续 | ★（现状已可接受） |

**推荐：方案 A（完全删除）+ 引入轻量测试 Fake**。理由：
- Mock 的演示逻辑（自动回复、伪 Alice/Bob/Carol、呼叫自动 accept）是早期 UI 开发的脚手架，现在 FRB 已就位，这些演示逻辑不再有产品价值。
- 测试真正需要的是「最小可控替身」（fake），而非「会自动聊天的演示」。
- 完全删除让 `NativeService` 契约更纯粹（单一真实实现）。

---

## 3. 方案 A 详细步骤（每步可编译、可测）

> 每步结束运行 `flutter analyze` + `flutter test`。任一步红则停下修复。

### Step 1 — 新建测试专用 Fake（在 `test/` 下）

在 `flutter/test/helpers/fake_native_service.dart` 新建一个**最小** NativeService 实现，仅覆盖 widget_test 实际用到的行为：

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:modcpt_app/services/native_service.dart';
import 'package:modcpt_app/models/server_models.dart';

/// Minimal NativeService fake for widget tests. No demo data, no auto-reply,
/// no timers — just enough to satisfy _Bootstrap / AppState during pumpWidget.
class FakeNativeService implements NativeService {
  @override
  bool get isNodeRunning => false;

  @override
  Future<BridgeResult<String>> startNode({String bindAddress = '0.0.0.0:0'}) async {
    return BridgeResult.ok('127.0.0.1:0');
  }

  @override
  Future<void> stopNode() async {}

  @override
  String get localAddress => '';

  @override
  Stream<BridgeEvent> get events => const Stream<BridgeEvent>.empty();

  @override
  void bindServerClient(ServerSession session) {}

  @override
  Future<BridgeResult<void>> publishPresence() async => BridgeResult.ok(null);

  @override
  Future<void> dispose() async {}

  // ... 其余 connect/send*/group 方法全部返回 ok / 空列表（见 §5 骨架）
}
```

**要点**：Fake 必须实现 `NativeService` 的**全部**抽象方法（27 个左右）。`events` 返回空流避免事件驱动逻辑。`localAddress` 返空串。

### Step 2 — 测试改为注入 Fake

`widget_test.dart` 改为在 `pumpWidget` 前注入 Fake：

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:modcpt_app/main.dart';
import 'package:modcpt_app/services/native_service.dart';
import 'helpers/fake_native_service.dart';

void main() {
  testWidgets('App boots and shows login page', (WidgetTester tester) async {
    NativeService.instance = FakeNativeService();
    await tester.pumpWidget(const ModCptApp());
    await tester.pump(const Duration(seconds: 1)); // 让 _tryAutoLogin 跑完
    expect(find.text('ModCpt'), findsWidgets);
  });
}
```

**注意**：`pumpAndSettle` 在有持续动画（`CircularProgressIndicator`）时会超时 —— 这是 pre-existing 问题，用 `pump(duration)` 代替（见 §6 已知问题）。改用 Fake 后，`_tryAutoLogin` 的 `loadPersistedSession` 读不到文件 → 返回 null → 进登录页，Fake 的 `startNode` 不会被调用。

### Step 3 — 删除 Mock 文件 + 解除 part 声明

1. 删除 `flutter/lib/services/mock_native_service.dart`。
2. `native_service.dart` 删除 `part 'mock_native_service.dart';`（`:38`）。
3. 删除文档注释中提及 Mock 的段落（`:8-13`）。

### Step 4 — 处理默认 instance（关键决策）

`native_service.dart:133` 的 `static NativeService instance = MockNativeService();` 需替换。**推荐改为抛异常的延迟初始化**：

```dart
/// Pluggable singleton. MUST be set before first use (main() sets it to
/// FrbNativeService). Throws if accessed before being assigned — prevents
/// accidental reliance on a demo/default implementation.
static late final NativeService instance = _unassigned;

static NativeService get _unassigned =>
    throw StateError('NativeService.instance not set. Call main() or assign a '
                     'test fake before using AppState.');
```

但 `late final` + 默认 getter 的组合在 Dart 里需要特别处理。**更稳妥的写法**：

```dart
static NativeService _instance =
    throw StateError('NativeService.instance not set before first use. '
                     'main() assigns FrbNativeService; tests assign a Fake.');
static NativeService get instance => _instance;
static set instance(NativeService v) => _instance = v;
```

> 这样 `AppState.instance = AppState()`（类加载时构造）会触发 throw —— **这正是我们想要的**：迫使所有入口（main / test）显式注入。`AppState` 单例的初始化也需同步调整为「延迟 + 由 main 设置」（见 Step 5）。

**替代（保守）**：若不想改 AppState 单例初始化，默认 instance 可临时指向一个「抛异常的占位」类型，但保留可赋值语义。最简单：保留 `late` 模式，把 `AppState.instance` 也改为 `late`，由 `main()` / `setUp` 显式 `= AppState(service: ...)`。

### Step 5 — 调整 AppState 单例初始化

当前 `static AppState instance = AppState();` 在类加载时跑，此刻 `NativeService.instance` 若已 throw 会崩。改为：

```dart
/// Lazily initialized. main() or test setUp assigns this before any widget
/// reads it.
static late AppState instance;
```

`main.dart` 改为：
```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await FrbRuntime.init();
  NativeService.instance = FrbNativeService();
  AppState.instance = AppState();  // 现在 NativeService.instance 已就位
  runApp(const ModCptApp());
}
```

测试 setUp 改为：
```dart
setUp(() {
  NativeService.instance = FakeNativeService();
  AppState.instance = AppState();
});
```

### Step 6 — 验证

```bash
cd flutter
flutter analyze   # 0 error / 0 warning
flutter test      # 全绿
```

### Step 7 — 文档同步

| 文档 | 改动 |
|------|------|
| `Knowledge/flutter/services/mock_native_service.md` | **删除** |
| `Knowledge/README.md` | 移除 mock_native_service.md 引用；Flutter services 文档数 6→（减 1）；总数减 1 |
| `Knowledge/flutter/services/native_service.md` | 文件概述删除「Mock/Frb 双实现」描述，改为「FrbNativeService 唯一实现，测试用 FakeNativeService」 |
| `Knowledge/logs/` | 新增日志记录本次移除 |
| `Knowledge/STRUCTURE.md` | 若提及 Mock，更新 |

---

## 4. 移除后需复核的关联点

| 关联点 | 位置 | 复核内容 |
|--------|------|------|
| `NativeService.instance` 默认值 | `native_service.dart:133` | 改为 throw-on-unassigned 后，确认无其他静态/顶层代码访问它 |
| `AppState.instance` 类加载 | `app_state.dart:35` | 改为 late 后，确认 UI 无「未 init 就读 instance」的路径 |
| `_Bootstrap._tryAutoLogin` | `main.dart:83` | 用 Fake 时，`loadPersistedSession` 返回 null → 进登录页，Fake 不会被调 |
| FRB 生成代码 | `src/rust/` | 不受影响（Mock 不依赖 FRB，但 `generateUserId` 被 mock 导入过 —— 删 mock 后该 import 仅 native_service.dart 保留，无碍） |
| `cryptography` 依赖 | `pubspec.yaml:13` | 仍被 `file_crypto.dart` 用，**不能删** |

---

## 5. FakeNativeService 完整骨架（27 方法最小实现）

> 复制即用。所有网络方法返回 ok / 空列表；所有 getter 返空/零值；事件流为空。

```dart
import 'package:modcpt_app/services/native_service.dart';
import 'package:modcpt_app/models/server_models.dart';
import 'package:modcpt_app/models/chat_models.dart';
import 'package:modcpt_app/models/file_share.dart';
import 'package:modcpt_app/models/group_log.dart';

class FakeNativeService implements NativeService {
  @override bool get isNodeRunning => false;
  @override String get localAddress => '';
  @override Stream<BridgeEvent> get events => const Stream<BridgeEvent>.empty();
  @override String get localPublicKey => '';

  @override Future<BridgeResult<String>> startNode({String bindAddress = '0.0.0.0:0'}) async =>
      BridgeResult.ok('127.0.0.1:0');
  @override Future<void> stopNode() async {}
  @override Future<void> dispose() async {}

  @override void bindServerClient(ServerSession session) {}
  @override Future<BridgeResult<void>> publishPresence() async => BridgeResult.ok(null);

  @override Future<BridgeResult<String>> connect(String address) async =>
      BridgeResult.ok('fake-peer');
  @override Future<BridgeResult<String>> connectByUserId(String userId) async =>
      BridgeResult.ok('fake-peer');
  @override Future<BridgeResult<List<PeerPresence>>> lookupByNickname(String nickname) async =>
      BridgeResult.ok(const []);
  @override Future<BridgeResult<void>> syncNickname(String nickname) async =>
      BridgeResult.ok(null);
  @override List<String> peerIds() => const [];
  @override Peer? peer(String id) => null;

  @override Future<BridgeResult<void>> sendText(String to, String content, {String? messageId}) async =>
      BridgeResult.ok(null);
  @override Future<BridgeResult<void>> sendTextGroup(String group, String content, {String? messageId}) async =>
      BridgeResult.ok(null);
  @override Future<BridgeResult<void>> sendFile(String to, String name, List<int> data) async =>
      BridgeResult.ok(null);
  @override Future<BridgeResult<void>> sendFileGroup(String group, String name, List<int> data) async =>
      BridgeResult.ok(null);
  @override Future<BridgeResult<void>> sendVoice(String to, int durationSeconds, List<int> bytes) async =>
      BridgeResult.ok(null);
  @override Future<BridgeResult<void>> sendVoiceGroup(String group, int durationSeconds, List<int> bytes) async =>
      BridgeResult.ok(null);

  @override void setUserId(String id) {}

  @override Future<BridgeResult<void>> announceFile(String group, FileAnnouncement announcement) async =>
      BridgeResult.ok(null);
  @override Future<BridgeResult<String>> requestFile(String fromPeer, String hashSha512, String name, int sizeBytes) async =>
      BridgeResult.ok('');
  @override Future<void> syncGroupLog(String groupName, String fromPeer) async {}
  @override Future<void> syncGroupHistory(String groupName, String fromPeer) async {}

  @override Future<List<int>> sign(List<int> message) async => List.filled(64, 0);
  @override Future<bool> verifySignature(List<int> message, List<int> signature, List<int> publicKey) async =>
      true;
  @override void storePeerPublicKey(String peerId, List<int> publicKey) {}
  @override List<int>? peerPublicKey(String peerId) => null;
  @override void cacheHashForRequest(String hashSha512, String filePath, List<int> bytes) {}

  @override void addToGroupLog(String groupName, GroupLogEntry entry) {}
  @override List<GroupLogEntry> getGroupLog(String groupName) => const [];

  @override Future<BridgeResult<void>> createGroup(String name, List<String> members) async =>
      BridgeResult.ok(null);
  @override Future<BridgeResult<void>> addMember(String group, String peer) async =>
      BridgeResult.ok(null);
  @override Future<BridgeResult<void>> removeMember(String group, String peer) async =>
      BridgeResult.ok(null);
  @override Future<BridgeResult<void>> deleteGroup(String group) async =>
      BridgeResult.ok(null);
  @override List<String> listGroups() => const [];
  @override Future<BridgeResult<List<String>>> groupMembers(String group) async =>
      BridgeResult.ok(const []);
}
```

---

## 6. 已知问题与注意

1. **`widget_test.dart` 的 pre-existing `pumpAndSettle` timeout**：这是当前测试就已存在的问题（`_Bootstrap` 的 `CircularProgressIndicator` 持续动画）。移除 Mock 时建议一并修复：用 `pump(Duration(seconds: 1))` 代替 `pumpAndSettle()`。
2. **`generateUserId` 的 import**：`native_service.dart:35-36` 导入并 export 了 `generateUserId`。Mock 曾在自身使用它（`_newId` 的伪 UUID 没用到，实际是 mock 注释里写的）。删 Mock 后该 import 仍被 native_service 保留供外部用，无需动。
3. **`AppState.instance` late 化的连锁**：所有 `AppState.instance` 的读取点（UI 各处）都在 `runApp` 之后，`main()` 已先赋值，安全。但需确认无顶层/静态初始化代码读它。
4. **`isStarted` 查询**：`AppState.isStarted` 转发 `_svc.isNodeRunning`。Fake 的 `isNodeRunning` 返 false，符合「未启动」语义。

---

## 7. 验收门禁

合并前全部满足：
1. `flutter analyze`：0 error / 0 warning
2. `flutter test`：全绿（`pumpAndSettle` timeout 一并修复）
3. grep `MockNativeService` / `mock_native_service` 全仓 0 命中（lib + test）
4. `NativeService.instance` 默认为 throw-on-unassigned（或经评审认可的等价方案）
5. `Knowledge/flutter/services/mock_native_service.md` 已删，README 索引已更新
6. `Knowledge/logs/` 有本次移除日志

---

## 8. 工作量估算

| 任务 | 估算 |
|------|------|
| 新建 FakeNativeService | 30 min（骨架复制 + 校验） |
| 测试适配 + 修 pumpAndSettle | 20 min |
| 删 Mock 文件 + 改 native_service.dart | 10 min |
| 默认 instance + AppState late 化 | 30 min（需仔细测无顶层访问） |
| 文档同步 | 20 min |
| 全量回归 | 30 min |
| **合计** | **~2.5 小时** |

---

> 本指南聚焦「怎么移除」。实施者按 §3 七步推进，每步对照 §7 门禁验证。方案 A 为推荐，若团队偏好保守可先用方案 C（Mock 降级到 test/）作为过渡。
