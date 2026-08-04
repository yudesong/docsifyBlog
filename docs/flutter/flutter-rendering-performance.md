---
title: Flutter 基础部分(四)
publishedTime: 2026-08-04T12:00:00+08:00
---

本文聚焦 Flutter 性能优化中面试常考、线上常踩的几个关键点：离屏渲染、自定义 ImageCache、单控件超时渲染、渲染上屏链路、Skia/OpenGL 选择、光栅缓存、GIF/Lottie 动画优化。理解这些内容，既能应对面试加分项，也能解决真实项目的卡顿与内存问题。

## 一、离屏渲染

### 1. 什么是离屏渲染

离屏渲染（Offscreen Rendering）指在屏幕外的缓冲区（离屏 surface）先绘制内容，再将其作为整体贴到屏幕上。它是一把双刃剑：

- **必要场景**：合成透明叠加层、做遮罩、生成阴影、模糊滤镜
- **性能代价**：多一次绘制 + 多一次采样上屏，显存与带宽翻倍

在 Flutter 中，离屏渲染主要通过 `Canvas.saveLayer` / `Canvas.restore` 实现。Skia 在 `saveLayer` 时分配一个独立 layer，后续绘制指令写入该 layer，`restore` 时整体合成到父 layer。

### 2. Flutter 触发离屏渲染的场景

| 触发场景 | 原因 |
|---------|------|
| `Opacity` Widget（非 0/1） | 需要 alpha 混合整层 |
| `ClipRRect` / `ClipPath`（复杂路径） | saveLayer 实现裁剪 |
| `ColorFilter` / `ImageFiltered` | 滤镜作用整层 |
| `BackdropFilter`（毛玻璃） | 必须先离屏渲染背景 |
| `ShaderMask` | 着色器作用于整层 |
| `BoxShadow`（带 blur） | 阴影需先离屏再模糊 |
| 透明通道叠加 | 多个半透明层叠加 |

### 3. 优化策略

**1）用 `Opacity` 的 0/1 边界优化**

```dart
// ❌ 性能差：透明度 0 时仍会 saveLayer
Opacity(opacity: isVisible ? 1.0 : 0.0, child: ...)

// ✅ 推荐：完全透明时直接不渲染
Visibility(visible: isVisible, child: ...)
// 或
if (isVisible) child
```

**2）`AnimatedOpacity` 替代 `Opacity`**

`AnimatedOpacity` 在 opacity 接近 0 或 1 时会自动切到 `Opacity`，但仍会触发 saveLayer。可考虑用 `FadeTransition` + `AnimatedBuilder`，或直接控制是否挂载。

**3）裁剪用 `ClipRect` 代替 `ClipRRect`**

矩形裁剪走快速路径，不触发 saveLayer；圆角裁剪默认走 saveLayer。Flutter 3.7+ 优化了 `ClipRRect` 的硬件加速路径，复杂路径仍需谨慎。

**4）阴影用 `BoxShadow` 的 `spreadRadius` + `blurRadius=0`**

```dart
// ❌ 触发离屏
BoxShadow(color: Colors.black26, blurRadius: 10, offset: Offset(0, 4))

// ✅ 无模糊，性能优
BoxShadow(color: Colors.black26, blurRadius: 0, offset: Offset(0, 4))
```

**5）`RepaintBoundary` 隔离复杂区域**

将触发离屏渲染的部分用 `RepaintBoundary` 包裹，独立绘制到自己的 layer，避免影响兄弟节点。

---

## 二、图片加载与自定义 ImageCache

### 1. 图片加载流程回顾

简版链路（详见基础部分三）：

```
Image widget
  → ImageProvider.resolve(cacheKey)
  → PaintingBinding.instance.imageCache.putIfAbsent(key, loader)
  → 命中缓存？直接返回 ImageStream
  → 未命中：load() 加载字节 → Skia 解码 → ui.Image
  → ImageStreamCompleter 通知 → setState 重建
```

### 2. ImageCache 默认策略

`ImageCache` 是单例，默认配置：

| 参数 | 默认值 | 含义 |
|------|--------|------|
| `maximumSize` | 1000 | 最大缓存图片数量 |
| `maximumSizeBytes` | 100 MB | 最大缓存字节数 |
| `maximumSizeTracked` | 500 | 跟踪中的待加载图片数 |

淘汰策略为 **LRU（Least Recently Used）**：访问时更新最近使用时间，超限时从最久未访问的开始驱逐。

### 3. 为什么需要自定义 ImageCache

默认配置在以下场景不合适：

- **图片密集型应用**（电商、相册）：100MB 远不够，频繁驱逐导致重复解码卡顿
- **低端机型**：100MB 易触发 OOM
- **大图场景**：单张 10MB 解码图，10 张就撑爆默认缓存

### 4. 自定义 ImageCache 实现

**方式 1：调整全局策略**

```dart
void main() {
  // 在 runApp 之前设置
  PaintingBinding.instance.imageCache.maximumSize = 500;        // 数量
  PaintingBinding.instance.imageCache.maximumSizeBytes = 200 * 1024 * 1024; // 200MB
  runApp(const MyApp());
}
```

**方式 2：自定义 ImageCache 子类**

```dart
class TrackedImageCache extends ImageCache {
  @override
  void putIfAbsent(Object key, ImageStreamCompleter loader(),
      {ImageErrorListener? onError}) {
    // 监控命中率
    final hit = containsKey(key);
    if (!hit) {
      _missCount++;
      // 上报到 APM
      APM.reportImageCacheMiss(key, _missCount, _hitCount);
    } else {
      _hitCount++;
    }
    super.putIfAbsent(key, loader, onError: onError);
  }

  int _hitCount = 0;
  int _missCount = 0;
}

void main() {
  // 替换默认实现
  PaintingBinding.instance.imageCache = TrackedImageCache();
  runApp(const MyApp());
}
```

**方式 3：自定义 ImageProvider 配合磁盘缓存**

默认 `ImageCache` 只缓存解码后的 `ui.Image`（内存层）。要做磁盘缓存，需自定义 `ImageProvider`，例如 `cached_network_image` 插件的实现思路：

```dart
class CachedNetworkImageProvider extends ImageProvider<CachedNetworkImageProvider> {
  @override
  ImageStreamCompleter load(key, decode) {
    return MultiFrameImageStreamCompleter(
      codec: _loadWithDiskCache(key, decode),
      scale: scale,
    );
  }

  Future<ui.Codec> _loadWithDiskCache(key, decode) async {
    // 1. 查磁盘缓存
    final File? cached = await _diskCache.get(key.url);
    if (cached != null) {
      return decode(await cached.readAsBytes());
    }
    // 2. 未命中则网络下载
    final Uint8List bytes = await _download(key.url);
    // 3. 写入磁盘缓存
    await _diskCache.put(key.url, bytes);
    return decode(bytes);
  }
}
```

### 5. 自定义 ImageCache 的实际作用

| 作用 | 说明 |
|------|------|
| **内存治理** | 根据设备内存动态调整 `maximumSizeBytes`，低端机降到 50MB |
| **命中率监控** | 上报 hit/miss，发现热点图片预热 |
| **优先级缓存** | 首页图保留，详情页图可淘汰 |
| **磁盘二级缓存** | 避免重复网络下载与解码 |
| **解码尺寸控制** | 配合 `cacheWidth` 缩放解码，降低单图内存 |

---

## 三、单控件渲染超过 16ms 的优化

### 1. 16ms 帧预算

60FPS 要求每帧 ≤ 16.67ms，120FPS 要求 ≤ 8.33ms。Flutter 一帧的耗时分布：

```
UI 线程（Dart）       │  Raster 线程（Skia）
─────────────────────┼──────────────────────
Widget 构建           │  光栅化
Element diff         │  合成
RenderObject 布局     │  上屏
绘制指令生成          │
```

任一阶段超时都会导致掉帧。单控件超时通常集中在 **UI 线程的 build / layout** 或 **Raster 线程的 paint**。

### 2. 不带动画时的优化策略

**策略 1：拆分 Widget，缩小 rebuild 范围**

```dart
// ❌ 整体重建
class BadCard extends StatelessWidget {
  final int count;
  final String avatarUrl;
  final String title;
  // count 变化时整张卡片重建
}

// ✅ 拆分
class GoodCard extends StatelessWidget {
  final int count;
  final String avatarUrl;
  final String title;

  @override
  Widget build(BuildContext context) {
    return Column(children: [
      Avatar(url: avatarUrl),     // const 子树不重建
      Title(text: title),
      CountText(count: count),    // 仅这部分重建
    ]);
  }
}
```

**策略 2：`const` Widget**

```dart
// ✅ const 编译期常量，永不重建
const Padding(
  padding: EdgeInsets.all(8),
  child: Text('static'),
)
```

**策略 3：`RepaintBoundary` 隔离重绘**

当某个子树绘制耗时但变化少时，用 `RepaintBoundary` 让它独立成 layer，避免父节点 rebuild 时重新绘制。

```dart
RepaintBoundary(
  child: ComplexChart(data: ...),
)
```

**策略 4：`AutomaticKeepAliveClientMixin` 保活**

ListView 滚动时不重建已滑出的 item：

```dart
class _ItemState extends State<Item> with AutomaticKeepAliveClientMixin {
  @override
  bool get wantKeepAlive => true;

  @override
  Widget build(BuildContext context) {
    super.build(context);
    return ...;
  }
}
```

**策略 5：`Builder` / `LayoutBuilder` 懒构建**

只在需要时构建：

```dart
Builder(builder: (context) {
  // 仅在 rebuild 触发时构建
  return expensiveWidget;
});
```

**策略 6：复杂计算下沉到 Isolate**

```dart
// ❌ 阻塞 UI 线程
final result = heavyCompute(data);

// ✅ 下沉到 Isolate
final result = await compute(heavyCompute, data);
```

**策略 7：避免深嵌套**

深嵌套导致 build / layout 时间线性增长，可用 `CustomMultiChildLayout` 替代多层嵌套的 `Row/Column`。

### 3. 带动画时的优化

动画是性能重灾区，因为每帧都要重建。关键优化点：

**1）`vsync` 必须传入**

```dart
class _MyState extends State<MyWidget> with SingleTickerProviderStateMixin {
  late AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,  // ✅ 关键：屏幕不可见时自动停止
      duration: const Duration(seconds: 1),
    );
  }
}
```

不带 `vsync` 的动画会在页面不可见时继续跑，浪费 CPU/GPU。

**2）用 `AnimatedBuilder` / `Tween` 隔离重建**

```dart
// ❌ 整个 widget 重建
AnimatedBuilder(
  animation: _controller,
  builder: (context, child) {
    return Transform.rotate(
      angle: _controller.value * 2 * pi,
      child: ExpensiveWidget(),  // ❌ 每帧重建
    );
  },
);

// ✅ child 提取，仅 Transform 重建
AnimatedBuilder(
  animation: _controller,
  builder: (context, child) {
    return Transform.rotate(
      angle: _controller.value * 2 * pi,
      child: child,  // ✅ 复用
    );
  },
  child: const ExpensiveWidget(),  // ✅ 只构建一次
);
```

**3）优先用 `Transform` / `Opacity` 等 RenderObject 层动画**

`Transform` 直接修改 `RenderTransform` 的 `transform` 矩阵，不触发 build：

```dart
// ✅ 性能优：仅修改矩阵
Transform.translate(
  offset: Offset(_controller.value * 100, 0),
  child: child,
)

// ❌ 性能差：触发 build 重建
Positioned(
  left: _controller.value * 100,
  child: child,
)
```

**4）`removeListener` + `addPostFrameCallback`**

避免在动画回调中做耗时操作：

```dart
_controller.addListener(() {
  // ❌ 每帧执行，避免重逻辑
  saveToDisk(_controller.value);
});

// ✅ 只在帧结束时保存
_controller.addStatusListener((status) {
  if (status == AnimationStatus.completed) {
    saveToDisk();
  }
});
```

**5）`disableAnimations` 适配无障碍**

```dart
if (MediaQuery.disableAnimationsOf(context)) {
  // 关闭动画，直接显示终态
  return const FinalWidget();
}
return AnimatedWidget(...);
```

**6）`TickerMode` 控制子树动画**

```dart
// 页面不可见时停止子树所有动画
TickerMode(enabled: isForeground, child: ...)
```

---

## 四、渲染到上屏过程（加分项）

### 1. 整体流水线

```
UI 线程（Dart）                     Raster 线程（Skia / Impeller）
─────────────────────              ─────────────────────────
1. WidgetsBinding.drawFrame()      5. Engine 接收 Scene
2.   build phase (Widget→Element)  6.   LayerTree → Skia Picture
3.   layout / paint                7.   光栅化（命令提交到 GPU）
4.   生成 LayerTree                 8.   Swapchain 上屏
   ↓                                ↑
   └── SceneBuilder.close()  ──────┘
```

### 2. UI 线程：生成 Scene

`WidgetsBinding.drawFrame()` 执行三步：

```dart
// 1. build：dirty Element 重建
buildOwner.buildScope(renderViewElement);

// 2. layout：RenderObject 计算尺寸与位置
pipelineOwner.flushLayout();

// 3. paint：RenderObject 生成 Layer
pipelineOwner.flushPaint();
```

`flushPaint` 时 `RenderObject` 调用 `PaintingContext` 记录 `PictureRecorder`，最终生成 `PictureLayer` / `OffsetLayer` / `TransformLayer` 等，组成 `LayerTree`。

```dart
// RenderView 是根 RenderObject
final SceneBuilder builder = SceneBuilder();
renderView.layer!.addToScene(builder);
final Scene scene = builder.build();
// 通过 ui.window.render 提交到 Engine
ui.window.render(scene);
```

### 3. Raster 线程：光栅化上屏

Engine 拿到 `Scene` 后，在 Raster Runner 执行：

1. **LayerTree 遍历**：将每个 `Layer` 转换为 Skia 操作
2. **Picture 回放**：`SkPicture::playback` 执行绘制指令
3. **光栅化**：矢量图转像素位图
4. **合成**：多层合成到 Surface
5. **SwapChain**：交换缓冲区，上屏

### 4. VSync 与帧调度

```
VSync 信号
    ↓
Platform Runner 接收
    ↓
Engine 触发 BeginFrame
    ↓
UI 线程执行 drawFrame
    ↓
UI 线程提交 Scene
    ↓
Raster 线程光栅化
    ↓
SwapChain 上屏
    ↓
等待下一个 VSync
```

如果某帧超时，下一个 VSync 来临时 UI 线程仍在跑，导致掉帧。

---

## 五、Skia 与 OpenGL 选择

### 1. Skia 在 Flutter 中的角色

Skia 是 Google 开源的 2D 图形库（Chrome、Android 系统都用），Flutter 用它作为默认渲染后端。Skia 本身不直接操作 GPU，而是通过 **GPU 后端** 与硬件交互：

| 后端 | 平台 | 说明 |
|------|------|------|
| **Skia + OpenGL** | Android / iOS / Linux | 传统后端，使用 OpenGL ES 3.0+ |
| **Skia + Vulkan** | Android 7.0+ | 更现代，性能更好但兼容性差 |
| **Skia + Metal** | iOS / macOS | Apple 强制要求 |
| **Impeller** | iOS / Android（新） | Flutter 自研，预编译 Shader |

### 2. Skia 何时选择 OpenGL

Flutter Engine 在启动时根据平台与设备能力选择后端：

- **iOS**：默认 Metal（Flutter 3.0+，Metal 是 Apple 主推）
- **Android**：默认 OpenGL ES（兼容性好），可选 Vulkan
- **桌面**：OpenGL / Vulkan

判断逻辑在 `engine/src/flutter/shell/gpu/gpu_surface_delegate.cc` 中，依据 `enable_impeller`、`enable_vulkan` 等配置开关。

### 3. 顶点着色器与片段着色器

OpenGL 渲染管线关键两步：

```
顶点数据
   ↓
[顶点着色器 Vertex Shader]   ← 计算顶点位置（模型/视图/投影矩阵变换）
   ↓
图元装配 / 裁剪
   ↓
[片段着色器 Fragment Shader] ← 计算每个像素颜色（纹理采样、混合）
   ↓
帧缓冲
```

Flutter 中常见的着色器用途：

| Shader 类型 | 用途 |
|------------|------|
| 顶点着色器 | 矩阵变换、裁剪、投影 |
| 片段着色器 | 颜色填充、纹理采样、alpha 混合 |
| 自定义 Shader（`Fragment` API） | 模糊、滤镜、自定义特效 |

### 4. Shader 编译卡顿（Jank）

Skia 在运行时根据需要编译 Shader，首次编译会卡帧（"Shader compilation jank"）。这是 Flutter 长期痛点。

**Impeller 的解决方案**：

- 构建时预编译所有 Shader 为 Metal Library / SPIR-V
- 运行时直接绑定，零编译耗时
- iOS 在 Flutter 3.10+ 默认启用，Android 在 3.27+ 预览

**Skia 时代的缓解方案**：`SkSL warmup`，在启动时预跑一遍绘制路径，触发 Shader 编译。

---

## 六、光栅化与光栅缓存

### 1. 什么是光栅化

光栅化（Rasterization）将矢量图（线段、曲线、填充区域）转换为像素位图。Skia 的 `GrDrawingManager` 负责将 `SkPicture` 转为 GPU 绘制命令。

光栅化是 Raster 线程最耗时的部分，复杂路径、大区域填充、模糊滤镜都慢。

### 2. 光栅缓存（Raster Cache）

**核心思想**：将复杂的静态 Picture 预渲染为位图，下次直接贴图，避免重复光栅化。

**触发条件**（Flutter Engine 默认）：

| 条件 | 默认值 |
|------|--------|
| 同一 Picture 连续出现帧数 | ≥ 3 帧 |
| Picture 的指令数 | ≥ 10 条 |
| Picture 估算绘制耗时 | 阈值以上 |
| Layer 尺寸 | 不超过最大缓存尺寸 |

**工作流程**：

```
帧 N：绘制 Picture P
  → Engine 记录 P 出现 1 次
帧 N+1：再次绘制 P
  → 出现 2 次，未达阈值
帧 N+2：再次绘制 P
  → 出现 3 次，触发 Raster Cache
  → 在后台线程将 P 光栅化为位图 texture
帧 N+3：再次绘制 P
  → 命中 cache，直接 drawImage(texture)
  → 跳过光栅化，性能大幅提升
```

### 3. Raster Cache 的优化建议

**1）`RepaintBoundary` 主动触发缓存**

`RepaintBoundary` 会强制生成独立 Layer，使该子树被 Raster Cache 识别为可缓存 Picture：

```dart
RepaintBoundary(
  child: StaticComplexWidget(),  // 不变化，可被缓存
)
```

**2）避免每帧变化的子树**

```dart
// ❌ 每帧 value 变化，无法被缓存
RepaintBoundary(
  child: Text('${DateTime.now().millisecond}'),
)

// ✅ 静态内容，可缓存
RepaintBoundary(
  child: Text('static'),
)
```

**3）`Opacity` 与 Raster Cache 的冲突**

带 alpha 的 Layer 默认不进入 Raster Cache，因为合成时需实时混合。可改为把不透明内容做 `RepaintBoundary`，外层薄 `Opacity`。

**4）监控 Raster Cache 命中率**

```dart
// 通过 Performance Overlay 观察 Raster 线程耗时
// 触发 cache 后 Raster 时间应明显下降
MaterialApp(
  showPerformanceOverlay: true,
  ...
)
```

DevTools 的 Performance 面板可看到每帧的 Raster Cache 命中情况。

---

## 七、GIF / Lottie 等动画优化

### 1. GIF 动画的性能问题

GIF 本质是 **多帧位图**，每帧都是一张完整图片。GIF 在 Flutter 中的性能痛点：

| 问题 | 原因 |
|------|------|
| 解码耗时 | 每帧都需解码 |
| 内存占用 | 所有帧同时驻留内存（`ui.Codec.frameCount`） |
| 文件体积大 | 无压缩优化（LZW 压缩率低） |
| 不支持半透明 | GIF 只有 1bit alpha，边缘锯齿明显 |
| 帧率受限 | GIF 帧延迟最小 10ms（约 100FPS），常见 50ms（20FPS） |

### 2. GIF 加载流程

```dart
// NetworkImage.load 返回 MultiFrameImageStreamCompleter
MultiFrameImageStreamCompleter(
  codec: _loadAsync(...),  // 解码整个 GIF → ui.Codec
  scale: scale,
);
```

`ui.Codec` 内部缓存所有帧。`MultiFrameImageStreamCompleter` 通过 `codec.getNextFrame()` 逐帧上报，配合 `Timer` 控制帧延迟。

### 3. GIF 优化方案

**方案 1：转换为 WebP / APNG**

WebP 动图体积更小、支持半透明、压缩率更高：

```bash
# 将 GIF 转 WebP
cwebp -q 80 animation.gif -o animation.webp
```

Flutter 原生支持 WebP，加载方式与 GIF 一致，性能与体积都更优。

**方案 2：转换为 Lottie JSON**

如果是矢量动画（如 loading 动画），转 Lottie 体积可缩小 10~100 倍：

```bash
# After Effects 导出 Lottie
# 使用 Bodymovin 插件导出 JSON
```

**方案 3：预解码所有帧到内存**

```dart
class PreloadedGif extends StatefulWidget {
  final String url;
  const PreloadedGif({super.key, required this.url});

  @override
  State<PreloadedGif> createState() => _PreloadedGifState();
}

class _PreloadedGifState extends State<PreloadedGif> {
  final List<ui.Image> _frames = [];

  @override
  void initState() {
    super.initState();
    _preload();
  }

  Future<void> _preload() async {
    final data = await rootBundle.load(widget.url);
    final codec = await ui.instantiateImageCodec(data.buffer.asUint8List());
    for (int i = 0; i < codec.frameCount; i++) {
      final frame = await codec.getNextFrame();
      _frames.add(frame.image);
    }
    if (mounted) setState(() {});
  }

  @override
  Widget build(BuildContext context) {
    if (_frames.isEmpty) return const SizedBox();
    return _AnimatedFramePlayer(frames: _frames);
  }
}
```

适合 GIF 帧数少（< 30 帧）的场景。

**方案 4：抽帧降帧率**

将 30FPS 的 GIF 抽成 15FPS：

```dart
// 每 2 帧只取 1 帧
for (int i = 0; i < codec.frameCount; i++) {
  final frame = await codec.getNextFrame();
  if (i % 2 == 0) {
    _frames.add(frame.image);
  }
}
```

降低视觉流畅度，但内存与 CPU 翻倍优化。

**方案 5：停止时释放**

```dart
@override
void dispose() {
  for (final frame in _frames) {
    frame.dispose();  // 释放 GPU 资源
  }
  super.dispose();
}
```

**方案 6：用 `RawImage` + `AnimationController` 替代 `Image`**

`Image` widget 内部的 `MultiFrameImageStreamCompleter` 会持续循环播放，即使页面不可见。可自定义播放器：

```dart
class GifPlayer extends StatefulWidget {
  final List<ui.Image> frames;
  final List<Duration> delays;
  const GifPlayer({super.key, required this.frames, required this.delays});

  @override
  State<GifPlayer> createState() => _GifPlayerState();
}

class _GifPlayerState extends State<GifPlayer>
    with SingleTickerProviderStateMixin {
  late final Ticker _ticker;
  int _current = 0;
  Duration _elapsed = Duration.zero;

  @override
  void initState() {
    super.initState();
    _ticker = Ticker(_onTick);
    _ticker.start();
  }

  void _onTick(Duration elapsed) {
    _elapsed += elapsed;
    if (_elapsed >= widget.delays[_current]) {
      _elapsed = Duration.zero;
      setState(() {
        _current = (_current + 1) % widget.frames.length;
      });
    }
  }

  @override
  void dispose() {
    _ticker.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return RawImage(image: widget.frames[_current]);
  }
}
```

配合 `TickerMode` 在页面不可见时自动停止。

### 4. Lottie 动画优化

Lottie 是 Airbnb 开源的矢量动画方案，基于 JSON 描述动画。性能优于 GIF，但仍有优化空间：

**问题点**：

| 问题 | 原因 |
|------|------|
| 复杂动画卡顿 | 每帧重新计算路径、变换 |
| 文件大 | 复杂动画 JSON 可达几 MB |
| 阴影 / 模糊慢 | Lottie 对 `DropShadow` 支持差 |

**优化方案**：

1. **`LottieCache` 预编译**

```dart
// 启动时预编译常用动画
LottieComposition composition = await LottieComposition.fromBytes(bytes);
```

2. **降低帧率**

```dart
Lottie.asset(
  'animations/loading.json',
  frameRate: FrameRate(30),  // 默认 60，降到 30
)
```

3. **`repeat` 控制循环**

```dart
final controller = AnimationController(
  vsync: this,
  duration: composition.duration,
)..repeat();
// 不需要时停止
controller.stop();
```

4. **拆分动画**

复杂动画拆成多个小 Lottie，按需加载。

5. **用 `Rive` 替代**

Rive 是新一代矢量动画工具，运行时基于自研渲染器，性能比 Lottie 更好，支持状态机：

```dart
RiveAnimation.asset(
  'assets/animations/button.riv',
  artboard: 'Button',
  stateMachines: const ['State Machine 1'],
)
```

### 5. 特效拉满的优化思路

当应用有大量粒子、光效、3D 变换时：

**1）用 `CustomPainter` + `Canvas` 直接绘制**

避免 Widget 树开销，直接调用 Skia API：

```dart
class ParticlePainter extends CustomPainter {
  final List<Particle> particles;
  ParticlePainter(this.particles);

  @override
  void paint(Canvas canvas, Size size) {
    final paint = Paint()..color = Colors.white;
    for (final p in particles) {
      canvas.drawCircle(p.position, p.radius, paint);
    }
  }

  @override
  bool shouldRepaint(covariant ParticlePainter old) =>
      particles != old.particles;
}
```

**2）`RepaintBoundary` + 离屏缓存**

静态背景层缓存，动态层独立绘制。

**3）`Canvas.drawPicture` 复用**

将复杂绘制录制成 `Picture`，多帧复用：

```dart
final recorder = PictureRecorder();
final canvas = Canvas(recorder);
// ... 复杂绘制
final picture = recorder.endRecording();

// 每帧直接回放
canvas.drawPicture(picture);
```

**4）Shader 着色器加速**

Flutter 3.7+ 支持自定义 Fragment Shader：

```glsl
// shader.frag
#include <flutter/runtime_effect.glsl>

uniform vec2 uResolution;
uniform float uTime;
out vec4 fragColor;

void main() {
  vec2 uv = FlutterFragCoord().xy / uResolution;
  fragColor = vec4(uv.x, uv.y, sin(uTime), 1.0);
}
```

```dart
final shader = await FragmentShader.fromAsset('shaders/water.frag');
shader.setFloat(0, size.width);
shader.setFloat(1, size.height);
shader.setFloat(2, time);
canvas.drawRect(rect, Paint()..shader = shader);
```

GPU 加速，性能远高于 Canvas 逐像素绘制。

---

## 八、总结

| 主题 | 核心要点 |
|------|---------|
| 离屏渲染 | `saveLayer` 触发；避免 `Opacity` 中间值、复杂 Clip、模糊阴影；用 `RepaintBoundary` 隔离 |
| 自定义 ImageCache | 调整 `maximumSize`/`maximumSizeBytes`；子类化实现命中率监控；配合磁盘二级缓存 |
| 单控件超时优化 | 拆分 Widget、`const`、`RepaintBoundary`、`AutomaticKeepAlive`、Isolate 下沉；动画用 `vsync`、`AnimatedBuilder` child 复用、RenderObject 层动画 |
| 渲染上屏 | UI 线程 build/layout/paint 生成 LayerTree → Raster 线程光栅化 → SwapChain 上屏 |
| Skia/OpenGL | Skia 是 2D 库，通过 OpenGL/Vulkan/Metal 后端操作 GPU；顶点着色器算位置，片段着色器算颜色；Shader 编译卡顿由 Impeller 解决 |
| 光栅缓存 | 静态 Picture 连续 ≥3 帧触发缓存为位图；`RepaintBoundary` 主动触发；避免每帧变化的子树 |
| GIF 优化 | 转 WebP/APNG/Lottie；预解码帧；抽帧降帧率；`dispose` 释放；自定义播放器控制循环 |
| Lottie 优化 | 预编译、降帧率、拆分；复杂场景考虑 Rive |
| 特效拉满 | `CustomPainter` 直接绘制；`Picture` 复用；Fragment Shader GPU 加速 |

性能优化的核心思路永远是：**先测量，再优化**。用 DevTools 的 Performance 面板定位瓶颈在 UI 线程还是 Raster 线程，针对性采取措施，避免盲目优化。
