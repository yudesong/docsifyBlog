---
title: Flutter 基础部分(五) - 性能优化与稳定性治理
publishedTime: 2026-08-04T13:00:00+08:00
---

本文是 Flutter 基础系列的收官篇，聚焦工程化视角的性能优化与稳定性治理。前四篇分别覆盖了基础概念、状态管理、编译启动、渲染原理；本篇从启动、包体积、内存、帧率、列表、网络六个维度讲优化，再从异常捕获、Crash 监控、内存泄漏、ANR 治理四个维度讲稳定性，最后盘点线上最常见的性能问题与排查思路。

## 一、性能优化全景

### 1. 性能优化的分层模型

Flutter 性能问题可按"发生位置"分层：

```
┌─────────────────────────────────────┐
│ 业务层：列表卡顿、过度重建、动画掉帧  │ ← 最常见
├─────────────────────────────────────┤
│ 框架层：Widget build/layout/paint    │ ← DevTools 直接可见
├─────────────────────────────────────┤
│ 渲染层：Raster Cache、Shader 编译    │ ← Performance Overlay
├─────────────────────────────────────┤
│ Engine 层：Skia、Impeller、4 线程    │ ← 需抓 systrace
├─────────────────────────────────────┤
│ 系统层：GC、IO、CPU 调度              │ ← native profiler
└─────────────────────────────────────┘
```

优化原则：**自顶向下定位，自底向上优化**。先用 DevTools 找到瓶颈在哪一层，再有针对性地解决。

### 2. 性能度量的核心指标

| 指标 | 60FPS 阈值 | 120FPS 阈值 | 度量方式 |
|------|-----------|------------|---------|
| UI 帧耗时 | ≤ 16ms | ≤ 8ms | `window.onReportTimings` |
| Raster 帧耗时 | ≤ 16ms | ≤ 8ms | Performance Overlay |
| 冷启动时间 | ≤ 2s | - | `adb shell am start` |
| 包体积 | < 20MB | - | APK Analyzer |
| 内存峰值 | < 200MB | - | DevTools Memory |
| 滚动帧率 | ≥ 55FPS | ≥ 110FPS | `flutter run --profile` |

---

## 二、启动优化

### 1. 启动阶段划分

```
[1] 原生启动（500ms ~ 2s）
    - Application.onCreate
    - FlutterMain.startInitialization（解压 libflutter.so）
    - Activity 创建

[2] Engine 初始化（200ms ~ 500ms）
    - 创建 4 个 Task Runner
    - 加载 snapshot
    - 创建 Dart Isolate

[3] Dart 入口执行（100ms ~ 1s）
    - main()
    - WidgetsFlutterBinding.ensureInitialized()
    - runApp()
    - 首帧 Widget 树构建

[4] 首帧渲染（100ms ~ 300ms）
    - build / layout / paint
    - Skia 光栅化
    - SwapChain 上屏
```

### 2. 优化策略

**策略 1：预热 Engine**

```dart
// 在 Application.onCreate 提前创建 Engine
class MyApplication : FlutterApplication() {
  lateinit var flutterEngine: FlutterEngine

  override fun onCreate() {
    super.onCreate()
    flutterEngine = FlutterEngine(this).apply {
      dartExecutor.executeDartEntrypoint(
        DartExecutor.DartEntrypoint.createDefault()
      )
    }
  }
}

// Activity 直接复用
class MainActivity : FlutterActivity() {
  override fun getFlutterEngine(): FlutterEngine? {
    return (application as MyApplication).flutterEngine
  }
}
```

冷启动可减少 300~500ms。

**策略 2：分阶段初始化**

```dart
void main() {
  WidgetsFlutterBinding.ensureInitialized();

  // 同步：仅必须的
  runApp(const App());

  // 异步：非首帧依赖
  Future.wait([
    _initAnalytics(),
    _initPush(),
    _initCrashReport(),
  ]);
}

Future<void> _initAnalytics() async {
  await Firebase.initializeApp();
  await FirebaseAnalytics.instance.logAppOpen();
}
```

**策略 3：首帧简化**

```dart
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: const SplashPage(),  // 简单启动页
      // 启动页渲染完成后再加载首页
    );
  }
}

class SplashPage extends StatefulWidget {
  @override
  State<SplashPage> createState() => _SplashPageState();
}

class _SplashPageState extends State<SplashPage> {
  @override
  void initState() {
    super.initState();
    // 首帧渲染完成后切到主页
    WidgetsBinding.instance.addPostFrameCallback((_) {
      Navigator.pushReplacement(context, MaterialPageRoute(
        builder: (_) => const HomePage(),
      ));
    });
  }

  @override
  Widget build(BuildContext context) {
    return const Scaffold(body: Center(child: Text('loading')));
  }
}
```

**策略 4：延迟加载路由**

```dart
// ❌ 启动时初始化所有路由
MaterialApp(
  routes: {
    '/home': (_) => HomePage(),
    '/profile': (_) => ProfilePage(),
    '/settings': (_) => SettingsPage(),
    // 50+ 个页面
  },
);

// ✅ 按需构建
MaterialApp(
  onGenerateRoute: (settings) {
    switch (settings.name) {
      case '/home': return MaterialPageRoute(builder: (_) => HomePage());
      case '/profile': return MaterialPageRoute(builder: (_) => ProfilePage());
      // 只在访问时构建
    }
    return null;
  },
);
```

**策略 5：避免 main() 中 await 网络**

```dart
// ❌ 阻塞启动
void main() async {
  await _fetchUserInfo();  // 网络慢，启动卡 2s
  runApp(const App());
}

// ✅ 启动后再获取
void main() {
  runApp(const App());
}

class App extends StatefulWidget {
  @override
  State<App> createState() => _AppState();
}

class _AppState extends State<App> {
  User? _user;

  @override
  void initState() {
    super.initState();
    _fetchUserInfo().then((u) => setState(() => _user = u));
  }
}
```

---

## 三、包体积优化

### 1. 包体积组成分析

Release APK 主要由四部分组成：

| 组成 | 占比 | 说明 |
|------|------|------|
| `libflutter.so` | 30~40% | Engine + Skia |
| `libapp.so` | 15~25% | Dart AOT 产物 |
| `flutter_assets/` | 5~15% | 图片、字体、JSON |
| 原生代码 + 资源 | 20~40% | Android/iOS 原生 |

### 2. 优化策略

**策略 1：ABI 拆分**

```gradle
android {
  splits {
    abi {
      enable true
      reset()
      include 'arm64-v8a', 'armeabi-v7a'
      universalApk false  // 不打通用包
    }
  }
}
```

每个 ABI 独立 APK，体积可减小 40%+。Google Play 支持按 ABI 分发。

**策略 2：资源压缩**

```bash
# PNG 压缩
pngquant --quality=70-90 *.png

# 转 WebP
cwebp -q 80 image.png -o image.webp
```

WebP 比 PNG 小 25~35%，比 JPEG 小 25%。

**策略 3：Tree Shaking 图标**

```dart
// ❌ 整个 Icons 库被引入
import 'package:flutter/material.dart';
Icon(Icons.accessibility, size: 24);

// ✅ Tree Shaking 后只保留 accessibility
// flutter build 默认开启 --tree-shake-icons
```

**策略 4：字体子集化**

```yaml
# pubspec.yaml
flutter:
  fonts:
    - family: Roboto
      fonts:
        - asset: assets/fonts/Roboto-Regular.ttf
```

```bash
# 仅保留中文字符 + ASCII
pyftsubset Roboto-Regular.ttf \
  --text-file=chars.txt \
  --output-file=Roboto-Regular-subset.ttf
```

**策略 5：deferred components（Android）**

```dart
import 'heavy_module.dart' deferred as heavy;

Future<void> loadHeavy() async {
  await heavy.loadLibrary();
  runApp(havy.HeavyApp());
}
```

Dart deferred imports 支持运行时按需下载 .so，减小首包体积。

**策略 6：移除未使用的依赖**

```bash
# 检查未使用的依赖
flutter pub deps
```

定期清理 `pubspec.yaml`，每个依赖都会增加 `libapp.so` 体积。

---

## 四、内存优化

### 1. Flutter 内存结构

```
┌─────────────────────────────────┐
│ Dart Heap（Dart 对象）           │ ← Widget、State、业务对象
├─────────────────────────────────┤
│ Native Heap（Skia/Engine）       │ ← ui.Image、纹理、Path
├─────────────────────────────────┤
│ Java/Kotlin Heap（Android）      │ ← 原生层对象
├─────────────────────────────────┤
│ 系统资源（Bitmap、File）          │ ← 真正的内存大户
└─────────────────────────────────┘
```

Flutter 内存泄漏常发生在 `ui.Image`（GPU 纹理）、`StreamSubscription`、`AnimationController`、`TextEditingController` 等。

### 2. 内存优化策略

**策略 1：及时 dispose**

```dart
class _MyState extends State<MyWidget> with TickerProviderStateMixin {
  late AnimationController _controller;
  late TextEditingController _textController;
  StreamSubscription? _sub;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(vsync: this, duration: Duration(seconds: 1));
    _textController = TextEditingController();
    _sub = _stream.listen((_) {});
  }

  @override
  void dispose() {
    _controller.dispose();      // ✅ 必须
    _textController.dispose();  // ✅ 必须
    _sub?.cancel();             // ✅ 必须
    super.dispose();
  }
}
```

**策略 2：图片缓存控制**

```dart
void main() {
  // 低端机降低缓存
  final memMB = await _getDeviceMemory();
  PaintingBinding.instance.imageCache.maximumSizeBytes =
      memMB < 2048 ? 50 * 1024 * 1024 : 100 * 1024 * 1024;
  runApp(const App());
}
```

**策略 3：WeakReference 缓存大对象**

```dart
class LargeObjectCache {
  final Expando<LargeObject> _cache = Expando();

  LargeObject? get(String key) => _cache[key];
  void set(String key, LargeObject obj) => _cache[key] = obj;
}
```

`Expando` 不会阻止 GC，适合缓存可重建的大对象。

**策略 4：图片 `cacheWidth` 缩放解码**

```dart
Image.network(
  'https://example.com/4k.jpg',
  cacheWidth: (MediaQuery.sizeOf(context).width * 2).toInt(),
)
```

避免 4K 图全尺寸解码占用 60MB+ 内存。

**策略 5：ListView 复用**

```dart
// ❌ 一次性构建所有 item
ListView(
  children: items.map((i) => ItemWidget(i)).toList(),
)

// ✅ 按需构建
ListView.builder(
  itemCount: items.length,
  itemBuilder: (_, i) => ItemWidget(items[i]),
)
```

---

## 五、帧率与滚动优化

### 1. 滚动卡顿的根因

Flutter 列表卡顿最常见的三种根因：

| 根因 | 表现 | 定位方法 |
|------|------|---------|
| build 耗时 | 滚动时 UI 线程高 | DevTools Timeline 看 build 时间 |
| layout 耗时 | 复杂嵌套导致 | Timeline 看 layout 时间 |
| 图片解码 | Raster 线程高 | Performance Overlay Raster 条 |

### 2. 优化策略

**策略 1：`ListView.builder` + `itemExtent`**

```dart
ListView.builder(
  itemCount: 1000,
  itemExtent: 80,  // 固定高度，跳过 layout 测量
  itemBuilder: (_, i) => ItemWidget(i),
)
```

`itemExtent` 让 Engine 跳过每个 item 的 layout 计算，性能提升 20%+。

**策略 2：`const` 子项**

```dart
class ItemWidget extends StatelessWidget {
  const ItemWidget(this.index, {super.key});
  final int index;

  @override
  Widget build(BuildContext context) {
    return const Padding(
      padding: EdgeInsets.all(8),
      child: Text('item'),  // 静态内容
    );
  }
}
```

**策略 3：`RepaintBoundary` 隔离**

```dart
ListView.builder(
  itemBuilder: (_, i) => RepaintBoundary(
    child: ComplexItemWidget(i),  // 独立 layer，避免影响兄弟
  ),
)
```

**策略 4：避免 `Opacity` / `ClipRRect`**

```dart
// ❌ 触发 saveLayer
Opacity(opacity: 0.5, child: ...)

// ✅ 用 Color.alpha 替代
Color(0x80FFFFFF)
```

**策略 5：`cacheExtent` 调整**

```dart
ListView.builder(
  cacheExtent: 500,  // 默认 250，预渲染 500 像素
  itemBuilder: ...,
)
```

适当增大可减少滚动时的突变，过大则浪费内存与 CPU。

**策略 6：避免在 build 中做重计算**

```dart
// ❌ 每次重建都格式化
class BadItem extends StatelessWidget {
  final DateTime time;
  @override
  Widget build(BuildContext context) {
    return Text(DateFormat('yyyy-MM-dd').format(time));  // ❌
  }
}

// ✅ 预计算
class GoodItem extends StatelessWidget {
  final String formattedTime;
  const GoodItem(this.formattedTime, {super.key});
  @override
  Widget build(BuildContext context) {
    return Text(formattedTime);
  }
}
```

---

## 六、网络与 IO 优化

### 1. HTTP 复用 Client

```dart
// ❌ 每次请求新建 Client
Future<Response> fetch() async {
  final client = HttpClient();  // ❌
  return client.get(...);
}

// ✅ 全局复用
final _httpClient = HttpClient()..maxConnectionsPerHost = 5;

Future<Response> fetch() async {
  return _httpClient.get(...);
}
```

### 2. 数据缓存

```dart
class DataRepository {
  final Map<String, dynamic> _cache = {};
  final Duration _ttl = Duration(minutes: 5);

  Future<dynamic> fetch(String key) async {
    if (_cache.containsKey(key)) {
      final entry = _cache[key]!;
      if (DateTime.now().difference(entry.time) < _ttl) {
        return entry.data;  // 命中缓存
      }
    }
    final data = await _fetchFromNetwork(key);
    _cache[key] = _CacheEntry(data, DateTime.now());
    return data;
  }
}
```

### 3. 预加载

```dart
// 列表加载时预加载下一页
class PagedListViewModel {
  int _page = 1;
  bool _isLoadingNext = false;

  Future<void> preloadNext() async {
    if (_isLoadingNext) return;
    _isLoadingNext = true;
    _page++;
    await _fetchPage(_page);
    _isLoadingNext = false;
  }
}
```

### 4. 数据库批量操作

```dart
// ❌ 单条插入
for (final item in items) {
  await db.insert('items', item.toMap());
}

// ✅ 批量事务
await db.transaction((txn) async {
  final batch = txn.batch();
  for (final item in items) {
    batch.insert('items', item.toMap());
  }
  await batch.commit();
});
```

---

## 七、稳定性治理：异常捕获

### 1. Dart 异常捕获机制

Dart 异常分两类：

- **同步异常**：`throw` / `try-catch`
- **异步异常**：`Future` 中的异常，需用 `catchError` 或 `await`

Flutter 框架默认会捕获 Widget build 中的异常，但不会捕获业务代码中的异步异常。

### 2. Zone 捕获全局异常

```dart
void main() {
  runZonedGuarded<Future<void>>(() async {
    WidgetsFlutterBinding.ensureInitialized();
    FlutterError.onError = (details) {
      FlutterError.presentError(details);
      _reportCrash(details.exception, details.stack);
    };
    runApp(const App());
  }, (error, stack) {
    // 捕获所有未处理的异步异常
    _reportCrash(error, stack);
  });
}

void _reportCrash(Object error, StackTrace? stack) {
  // 上报到 APM 平台
  CrashReporter.report(error.toString(), stack?.toString());
}
```

### 3. FlutterError.onError vs Zone

| 机制 | 捕获范围 | 适用场景 |
|------|---------|---------|
| `FlutterError.onError` | Flutter 框架抛出的异常（build/layout/paint） | Widget 树异常 |
| `Zone.handleUncaughtError` | 所有 Dart 异常（含异步） | 业务异常兜底 |
| `Isolate.current.addErrorListener` | 其他 Isolate 的异常 | compute / spawn 的子 Isolate |

**完整捕获示例**：

```dart
void main() {
  // 1. 框架异常
  FlutterError.onError = (details) {
    FlutterError.presentError(details);
    _report('framework', details.exception, details.stack);
  };

  // 2. Dart 异步异常
  runZonedGuarded(() {
    runApp(const App());
  }, (error, stack) {
    _report('zone', error, stack);
  });

  // 3. 子 Isolate 异常
  Isolate.current.addErrorListener(RawReceivePort((dynamic data) {
    final list = data as List;
    _report('isolate', list[0], StackTrace.fromString(list[1]));
  }).sendPort);
}
```

### 4. 原生层 Crash 捕获

Flutter 应用的原生 Crash（Java/Kotlin/ObjC）需要原生层捕获：

**Android（已崩溃时抓取 tombstone）**：

```kotlin
class MyApplication : FlutterApplication() {
  override fun onCreate() {
    super.onCreate()
    // 注册 Java 未捕获异常处理器
    Thread.setDefaultUncaughtExceptionHandler { thread, throwable ->
      CrashReporter.reportNative(throwable)
      // 退出前持久化日志
    }
  }
}
```

**iOS（Signal 捕获）**：

```objc
void signalHandler(int signal) {
  NSArray *callStack = [NSThread callStackSymbols];
  CrashReporter.reportNative(callStack);
  exit(signal);
}

void installSignalHandlers() {
  signal(SIGABRT, signalHandler);
  signal(SIGSEGV, signalHandler);
  signal(SIGBUS, signalHandler);
  signal(SIGPIPE, signalHandler);
}
```

### 5. 第三方 APM 集成

| 平台 | Flutter SDK | 特点 |
|------|------------|------|
| **Sentry** | `sentry_flutter` | 全栈监控，开源可自建 |
| **Bugsnag** | `bugsnag_flutter` | 商业方案，稳定 |
| **Firebase Crashlytics** | `firebase_crashlytics` | Google 生态，免费 |
| **腾讯 Bugly** | 原生层集成 | 国内常用 |
| **友盟 APM** | 原生层集成 | 国内常用 |

**Sentry 集成示例**：

```dart
import 'package:sentry_flutter/sentry_flutter.dart';

Future<void> main() async {
  await SentryFlutter.init(
    (options) {
      options.dsn = 'https://xxx@sentry.io/xxx';
      options.tracesSampleRate = 1.0;
    },
    appRunner: () => runApp(const App()),
  );
}

// 手动上报
try {
  await riskyOperation();
} catch (e, stack) {
  await Sentry.captureException(e, stackTrace: stack);
}
```

---

## 八、稳定性治理：内存泄漏

### 1. 常见泄漏场景

| 场景 | 原因 | 表现 |
|------|------|------|
| `AnimationController` 未 dispose | 持有 `TickerProvider`（State） | 页面退出后动画继续跑 |
| `StreamSubscription` 未 cancel | Stream 持有 listener | 数据持续推送 |
| `TextEditingController` 未 dispose | 持有 `_TextEditingController` | 内存不释放 |
| `ScrollController` 未 dispose | 持有 `_Positions` | 滚动状态泄漏 |
| `Timer` 未 cancel | 持有闭包 | 定时任务持续执行 |
| `GlobalKey` 跨页面复用 | Element 被 attach 到多棵树 | State 错乱 |
| 闭包捕获 `context` | 长生命周期对象持有 Element | 页面无法释放 |
| `ui.Image` 未 dispose | GPU 纹理不释放 | 图片类应用 OOM |

### 2. 泄漏检测工具

**DevTools Memory 面板**：

1. 打开 DevTools → Memory
2. 操作页面（进入 → 退出）
3. 点击 "GC" 强制垃圾回收
4. 观察内存是否回落
5. 不回落则存在泄漏

**`leak_tracker` 包**（Flutter 官方）：

```yaml
# pubspec.yaml
dev_dependencies:
  leak_tracker: ^10.0.0
  leak_tracker_flutter_testing: ^3.0.0
```

```dart
import 'package:leak_tracker_flutter_testing/leak_tracker_flutter_testing.dart';

void main() {
  testWidgets('should not leak', (tester) async {
    await tester.pumpWidget(const MyPage());
    await tester.pumpAndSettle();
    await tester.pumpWidget(const SizedBox());  // 卸载
    await tester.pumpAndSettle();
  });
}
```

### 3. 闭包泄漏案例

```dart
// ❌ 闭包捕获 this（State），异步完成后 State 已销毁
class _MyState extends State<MyPage> {
  @override
  void initState() {
    super.initState();
    _fetchData().then((data) {
      setState(() {});  // ❌ State 已 dispose 时崩溃
    });
  }
}

// ✅ 检查 mounted
class _MyState extends State<MyPage> {
  @override
  void initState() {
    super.initState();
    _fetchData().then((data) {
      if (!mounted) return;
      setState(() {});
    });
  }
}
```

### 4. ui.Image 泄漏

```dart
class ImagePainter extends CustomPainter {
  final ui.Image image;
  ImagePainter(this.image);

  @override
  void paint(Canvas canvas, Size size) {
    canvas.drawImage(image, Offset.zero, Paint());
  }

  @override
  bool shouldRepaint(covariant ImagePainter old) => image != old.image;
}

// 使用方需 dispose
class _MyState extends State<MyWidget> {
  ui.Image? _image;

  @override
  void dispose() {
    _image?.dispose();  // ✅ 释放 GPU 纹理
    super.dispose();
  }
}
```

---

## 九、稳定性治理：ANR

### 1. ANR 的根因

Android ANR（Application Not Responding）触发条件：

- 主线程 5 秒无响应（输入事件超时）
- `BroadcastReceiver` 10 秒未完成
- `Service` 20 秒未完成

Flutter 应用的 ANR 常见原因：

| 原因 | 说明 |
|------|------|
| Platform Channel 主线程阻塞 | `onMethodCall` 在主线程做 IO |
| 同步调用原生方法卡死 | 原生层死锁 |
| Engine 初始化慢 | 启动阶段卡住主线程 |
| Dart 主 Isolate 卡住 | 死循环、大计算 |

### 2. 排查方法

```bash
# 拉取 ANR trace
adb pull /data/anr/traces.txt

# 关键信息：
# - 主线程堆栈（"main" prio=5）
# - 锁持有情况（- locked <0x...>）
# - 是否在 Flutter native 调用中
```

### 3. 优化策略

**1）Platform Channel 切线程**

```kotlin
class DbPlugin : MethodCallHandler {
  override fun onMethodCall(call: MethodCall, result: Result) {
    // ❌ 主线程做 IO
    // val data = File("xxx").readText()

    // ✅ 切到 IO 线程
    Thread {
      try {
        val data = File("xxx").readText()
        Handler(Looper.getMainLooper()).post {
          result.success(data)
        }
      } catch (e: Exception) {
        Handler(Looper.getMainLooper()).post {
          result.error("IO_ERROR", e.message, null)
        }
      }
    }.start()
  }
}
```

**2）避免 Dart 主 Isolate 阻塞**

```dart
// ❌ 阻塞 UI 线程
final result = heavySort(bigList);

// ✅ 下沉到 Isolate
final result = await compute(heavySort, bigList);
```

**3）WatchDog 监控**

```dart
class WatchDog {
  Timer? _timer;
  DateTime _lastTick = DateTime.now();

  void start() {
    _timer = Timer.periodic(Duration(seconds: 2), (_) {
      final now = DateTime.now();
      if (now.difference(_lastTick) > Duration(seconds: 5)) {
        // UI 线程卡住，上报
        CrashReporter.reportANR(now.difference(_lastTick));
      }
      _lastTick = now;
    });
  }

  void stop() => _timer?.cancel();
}
```

---

## 十、常见性能问题速查

### 1. 卡顿问题

| 症状 | 可能原因 | 排查方法 |
|------|---------|---------|
| 滚动掉帧 | build/layout/paint 耗时 | DevTools Timeline |
| 列表首屏白屏 | 一次性构建过多 item | 改用 `ListView.builder` |
| 图片加载慢 | 网络慢、解码慢 | 预加载 + `cacheWidth` |
| 动画卡顿 | Shader 编译 / saveLayer | `--purge-persistent-cache` |
| 页面切换卡 | 路由构建复杂 | `Hero` / 预构建 |

### 2. 内存问题

| 症状 | 可能原因 | 排查方法 |
|------|---------|---------|
| 内存持续上涨 | Widget/Stream 未 dispose | leak_tracker |
| OOM 崩溃 | 图片缓存过大 | 调整 `maximumSizeBytes` |
| 切页面不回收 | `GlobalKey` 误用 | 检查 GlobalKey 复用 |
| 滚动越滚越卡 | `ListView` 缓存失控 | 调整 `cacheExtent` |

### 3. 启动问题

| 症状 | 可能原因 | 排查方法 |
|------|---------|---------|
| 冷启动 > 3s | main() 阻塞 | DevTools Timeline 启动段 |
| 白屏时间长 | Engine 初始化慢 | 预热 Engine |
| 首帧卡顿 | 首屏 Widget 复杂 | 拆分 + `addPostFrameCallback` |

### 4. 网络问题

| 症状 | 可能原因 | 排查方法 |
|------|---------|---------|
| 接口耗时波动 | DNS / TLS 握手 | HTTP DNS / 连接复用 |
| 图片加载失败 | CDN 故障 | 兜底图 + 重试 |
| 弱网卡死 | 未做超时 | `TimeoutException` |

### 5. 通信问题

| 症状 | 可能原因 | 排查方法 |
|------|---------|---------|
| Channel 调用无响应 | 原生方法卡死 | 加超时 + 切线程 |
| 数据序列化失败 | 不支持的类型 | 改用 JSON 字符串 |
| 多 Engine 通信异常 | Plugin 重复注册 | 单例 + Engine 隔离 |

---

## 十一、性能监控体系

### 1. 线上监控指标

```dart
class PerformanceMonitor {
  static final PerformanceMonitor _instance = PerformanceMonitor._();
  factory PerformanceMonitor() => _instance;
  PerformanceMonitor._();

  void startFrameTracking() {
    SchedulerBinding.instance.addTimingsCallback(_onTimings);
  }

  void _onTimings(List<FrameTiming> timings) {
    for (final t in timings) {
      final buildMs = t.buildDuration.inMilliseconds;
      final rasterMs = t.rasterDuration.inMilliseconds;
      final totalMs = t.totalSpan.inMilliseconds;

      // 上报慢帧
      if (totalMs > 16) {
        APM.reportFrame('slow', buildMs, rasterMs, totalMs);
      }
      if (totalMs > 700) {
        APM.reportFrame('frozen', buildMs, rasterMs, totalMs);
      }
    }
  }
}

void main() {
  PerformanceMonitor().startFrameTracking();
  runApp(const App());
}
```

### 2. 关键指标上报

| 指标 | 阈值 | 上报字段 |
|------|------|---------|
| 慢帧率 | > 16ms | build/raster/total |
| 卡帧率 | > 700ms | build/raster/total |
| 冷启动 | > 2s | native/engine/dart/firstFrame |
| 内存峰值 | > 200MB | dart/native/total |
| Crash 率 | > 0.1% | exception/stack/native |

### 3. DevTools 配合

开发期必用工具：

| 工具 | 用途 |
|------|------|
| **Performance Overlay** | 实时看 UI/Raster 帧时间 |
| **Timeline** | 详细分析 build/layout/paint |
| **Memory** | 内存曲线 + 堆快照 |
| **CPU Profiler** | Dart 方法耗时火焰图 |
| **Network** | HTTP 请求时序 |
| **Widget Inspector** | Widget 树检查 rebuild |

---

## 十二、总结

| 维度 | 核心要点 |
|------|---------|
| **启动优化** | 预热 Engine、分阶段初始化、首帧简化、延迟加载路由、main() 不 await 网络 |
| **包体积** | ABI 拆分、资源压缩、Tree Shaking、字体子集化、deferred components |
| **内存优化** | 及时 dispose、调整 ImageCache、`cacheWidth` 缩放解码、ListView.builder 复用 |
| **帧率优化** | `itemExtent`、`const` 子项、`RepaintBoundary`、避免 `Opacity`/`ClipRRect`、build 中不做重计算 |
| **网络优化** | 复用 HttpClient、数据缓存、预加载、数据库批量操作 |
| **异常捕获** | `FlutterError.onError` + `Zone` + `Isolate.addErrorListener` 三层兜底 |
| **内存泄漏** | AnimationController/Stream/Timer/Image 必须 dispose；`mounted` 检查 |
| **ANR 治理** | Platform Channel 切线程、Isolate 下沉计算、WatchDog 监控 |
| **线上监控** | `SchedulerBinding.addTimingsCallback` 上报帧率、APM 平台聚合 |

**性能优化的方法论**：

1. **先测量，再优化**：用 DevTools 找到瓶颈在哪一层，不要凭感觉
2. **80/20 原则**：80% 的性能问题来自 20% 的代码，集中火力
3. **回归测试**：每次优化后跑性能基线，避免回退
4. **持续监控**：线上 APM + 线下 DevTools 双管齐下
5. **不过度优化**：满足业务需求即可，不要为追求 60FPS 牺牲可读性

至此，Flutter 基础系列五篇完结：

- [基础部分(一)](flutter-classic-interview-questions.md)：Dart 与 Flutter 面试基础
- [基础部分(二)](flutter-state-management.md)：状态管理方案详解
- [基础部分(三)](flutter-compilation-startup-image.md)：编译启动与图片加载原理
- [基础部分(四)](flutter-rendering-performance.md)：渲染性能与动画优化
- [基础部分(五)](flutter-performance-stability.md)：性能优化与稳定性治理（本文）
