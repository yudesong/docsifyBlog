---
title: SwiftUI 开发实战指南
publishedTime: 2026-08-04T14:30:00+08:00
---

SwiftUI 是 Apple 在 2019 年 WWDC 推出的声明式 UI 框架，用于构建 iOS / macOS / watchOS / tvOS / visionOS 应用。本文系统介绍 SwiftUI 的核心概念、布局系统、状态管理、动画、导航、与 UIKit 互操作等实战内容。

## 一、SwiftUI 概述

### 1. 与 UIKit 的对比

| 维度 | SwiftUI | UIKit |
|------|---------|-------|
| 范式 | 声明式（描述 UI 应该是什么） | 命令式（描述如何构建 UI） |
| 状态驱动 | UI 是状态的函数 | 手动同步状态与 UI |
| 布局 | HStack / VStack / ZStack | Auto Layout / Manual |
| 预览 | Xcode Previews 实时预览 | Storyboard / Interface Builder |
| 跨平台 | 一套代码多端 | 仅 iOS / iPadOS |
| 数据流 | `@State` / `@Binding` / `@Observable` | Delegate / KVO / Closure |
| 学习曲线 | 上手快，进阶深 | 入门稍复杂 |

### 2. 最小示例

```swift
import SwiftUI

struct ContentView: View {
    var body: some View {
        Text("Hello, SwiftUI")
            .padding()
    }
}

// 预览
#Preview {
    ContentView()
}
```

`View` 是一个协议，`body` 是返回 `some View` 的计算属性。`some View` 是不透明类型（Opaque Type），表示"某种具体的 View 类型"，但调用方无需知道具体类型。

### 3. 视图的本质

SwiftUI 的 `View` 不是屏幕上的对象，而是**描述 UI 的轻量数据结构**。系统根据 View 树生成 `ViewGraph`，再 diff 后驱动渲染。

```swift
// View 是值类型 struct
struct ContentView: View {
    var body: some View {
        VStack {
            Text("Hello")
            Text("World")
        }
    }
}
```

每次状态变化，SwiftUI 重新计算 body，通过 diff 决定哪些部分需要更新。

---

## 二、基础视图与修饰器

### 1. 常用基础视图

```swift
// 文本
Text("Hello")
    .font(.title)
    .foregroundColor(.blue)
    .bold()
    .italic()

// 图片
Image("logo")
    .resizable()
    .aspectRatio(contentMode: .fit)
    .frame(width: 100, height: 100)

// SF Symbols
Image(systemName: "star.fill")
    .font(.system(size: 30))
    .foregroundColor(.yellow)

// 按钮
Button(action: {
    print("tapped")
}) {
    Text("Click Me")
        .padding()
        .background(Color.blue)
        .foregroundColor(.white)
        .cornerRadius(8)
}

// 递增/递减按钮
Button("Increment") {
    count += 1
}
.buttonStyle(.borderedProminent)

// 文本框
TextField("输入用户名", text: $username)
    .textFieldStyle(.roundedBorder)
    .padding()

SecureField("密码", text: $password)

// 开关
Toggle("启用通知", isOn: $notificationsEnabled)

// 滑块
Slider(value: $volume, in: 0...1)

// 选择器
Picker("选择", selection: $selected) {
    Text("A").tag(0)
    Text("B").tag(1)
}
.pickerStyle(.segmented)

// 进度
ProgressView(value: 0.7)
```

### 2. 修饰器（Modifier）

修饰器是链式调用，包装 View 形成洋葱结构：

```swift
Text("Hello")
    .padding()
    .background(Color.blue)
    .foregroundColor(.white)
    .cornerRadius(8)

// 等价于
CornerRadiusShape(
    shape: ForegroundColorView(
        view: BackgroundView(
            view: PaddingView(
                view: Text("Hello")
            )
        )
    )
)
```

**修饰器顺序很重要**：

```swift
// padding 在 background 外，背景只占文字区域
Text("Hello")
    .background(Color.red)
    .padding()

// padding 在 background 内，背景包含 padding 区域
Text("Hello")
    .padding()
    .background(Color.red)
```

### 3. 列表

```swift
// 静态列表
List {
    Text("Item 1")
    Text("Item 2")
    Text("Item 3")
}

// 动态列表
List(items) { item in
    Text(item.name)
}

// 带分区
List {
    Section("水果") {
        ForEach(fruits) { fruit in
            Text(fruit)
        }
    }
    Section("蔬菜") {
        ForEach(vegetables) { vegetable in
            Text(vegetable)
        }
    }
}

// 可删除、可移动
List {
    ForEach(items) { item in
        Text(item.name)
    }
    .onDelete { indexSet in
        items.remove(atOffsets: indexSet)
    }
    .onMove { source, destination in
        items.move(fromOffsets: source, toOffset: destination)
    }
}
```

---

## 三、布局系统

### 1. HStack / VStack / ZStack

```swift
// 水平排列
HStack(spacing: 20) {
    Text("Left")
    Text("Center")
    Text("Right")
}

// 垂直排列
VStack(alignment: .leading, spacing: 10) {
    Text("Line 1")
    Text("Line 2")
}

// 叠加
ZStack {
    Color.gray
    Text("Overlay")
}
```

### 2. 对齐与间距

```swift
HStack(alignment: .center, spacing: 16) {
    Circle().frame(width: 40, height: 40)
    VStack(alignment: .leading) {
        Text("Title")
        Text("Subtitle")
    }
    Spacer()       // 弹性空间
    Image(systemName: "chevron.right")
}
.padding()
```

### 3. Spacer

```swift
HStack {
    Text("Left")
    Spacer()          // 占据剩余空间
    Text("Right")
}

HStack {
    Spacer()
    Text("Center")
    Spacer()
}
```

### 4. frame

```swift
Text("Hello")
    .frame(width: 200, height: 50)

// 最大尺寸
Circle()
    .frame(maxWidth: .infinity, maxHeight: .infinity)
    .background(Color.blue)
```

### 5. 自定义布局（iOS 16+）

```swift
struct HFlow: Layout {
    var spacing: CGFloat = 8

    func sizeThatFits(proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) -> CGSize {
        let result = HFlowLayoutResult(subviews: subviews, proposal: proposal, spacing: spacing)
        return result.size
    }

    func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) {
        let result = HFlowLayoutResult(subviews: subviews, proposal: proposal, spacing: spacing)
        for (index, subview) in subviews.enumerated() {
            subview.place(at: CGPoint(x: bounds.minX + result.frames[index].minX,
                                       y: bounds.minY + result.frames[index].minY),
                          proposal: ProposedViewSize(result.frames[index].size))
        }
    }
}

// 使用
HFlow(spacing: 8) {
    ForEach(tags, id: \.self) { tag in
        Text(tag)
            .padding(.horizontal, 8)
            .padding(.vertical, 4)
            .background(Color.gray.opacity(0.2))
            .cornerRadius(8)
    }
}
```

### 6. GeometryReader

```swift
GeometryReader { geo in
    Text("Width: \(geo.size.width)")
        .frame(width: geo.size.width,
               height: geo.size.height,
               alignment: .center)
}
```

注意：`GeometryReader` 不像普通 View 那样自适应内容，它占满父容器提供的全部空间。

---

## 四、状态管理

### 1. @State（局部状态）

```swift
struct CounterView: View {
    @State private var count = 0

    var body: some View {
        VStack {
            Text("Count: \(count)")
            Button("Increment") {
                count += 1
            }
        }
    }
}
```

`@State` 用于值类型的局部状态。状态变化时，SwiftUI 自动重新计算 `body`。

### 2. @Binding（双向绑定）

```swift
struct ToggleView: View {
    @Binding var isOn: Bool

    var body: some View {
        Toggle("Enabled", isOn: $isOn)
    }
}

struct ParentView: View {
    @State private var enabled = false

    var body: some View {
        ToggleView(isOn: $enabled)   // 传递绑定
    }
}
```

`$` 前缀创建绑定，子视图修改会反映到父视图。

### 3. @ObservedObject / @StateObject

```swift
class UserViewModel: ObservableObject {
    @Published var name = "Swift"
    @Published var age = 10
}

struct UserView: View {
    @StateObject var viewModel = UserViewModel()  // 持有

    var body: some View {
        VStack {
            Text(viewModel.name)
            Button("Change") {
                viewModel.name = "SwiftUI"
            }
        }
    }
}
```

- `@StateObject`：视图拥有并管理生命周期（首次创建后保留）
- `@ObservedObject`：视图不拥有，由外部传入
- `@Published`：属性变化时自动触发 UI 更新

### 4. @EnvironmentObject（全局共享）

```swift
@main
struct MyApp: App {
    @StateObject var appState = AppState()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(appState)
        }
    }
}

class AppState: ObservableObject {
    @Published var currentUser: User?
    @Published var theme: Theme = .light
}

// 任意子视图获取
struct ProfileView: View {
    @EnvironmentObject var appState: AppState

    var body: some View {
        Text(appState.currentUser?.name ?? "未登录")
    }
}
```

### 5. @Observable（iOS 17+ / macOS 14+）

```swift
@Observable
class UserModel {
    var name = "Swift"
    var age = 10
}

struct UserView: View {
    let user = UserModel()

    var body: some View {
        VStack {
            Text(user.name)
            Button("Change") {
                user.name = "New"
            }
        }
    }
}
```

`@Observable` 宏替代 `ObservableObject` + `@Published`，更简洁高效，无需逐个属性标注。

### 6. 数据流总览

| 属性包装器 | 适用场景 | 持有方 | 生命周期 |
|------------|---------|--------|---------|
| `@State` | 视图局部值类型状态 | 当前视图 | 与视图同寿命 |
| `@Binding` | 子视图依赖父视图状态 | 父视图 | 与父视图同寿命 |
| `@StateObject` | 视图拥有的引用类型模型 | 当前视图 | 与视图同寿命 |
| `@ObservedObject` | 外部传入的引用类型模型 | 外部 | 外部决定 |
| `@EnvironmentObject` | 全局共享状态 | 上游注入 | 上游决定 |
| `@Observable` | 新一代模型（iOS 17+） | 任意 | 任意 |

---

## 五、导航

### 1. NavigationStack（iOS 16+）

```swift
NavigationStack {
    List(items) { item in
        NavigationLink(value: item) {
            Text(item.name)
        }
    }
    .navigationDestination(for: Item.self) { item in
        DetailView(item: item)
    }
    .navigationTitle("Items")
}
```

### 2. NavigationView（旧版，已废弃）

```swift
NavigationView {
    List {
        NavigationLink("Detail") {
            DetailView()
        }
    }
    .navigationTitle("Home")
}
```

### 3. TabView

```swift
TabView {
    HomeView()
        .tabItem {
            Image(systemName: "house")
            Text("Home")
        }
    ProfileView()
        .tabItem {
            Image(systemName: "person")
            Text("Profile")
        }
}
```

### 4. Sheet / Popover

```swift
struct ContentView: View {
    @State private var showSheet = false

    var body: some View {
        Button("Show") {
            showSheet = true
        }
        .sheet(isPresented: $showSheet) {
            ModalView()
        }
    }
}
```

### 5. 路径编程式导航

```swift
struct RootView: View {
    @State private var path = NavigationPath()

    var body: some View {
        NavigationStack(path: $path) {
            HomeView()
                .navigationDestination(for: String.self) { route in
                    if route == "detail" {
                        DetailView()
                    }
                }
        }
        .onChange(of: someTrigger) { _, _ in
            path.append("detail")
        }
    }
}
```

---

## 六、动画

### 1. 隐式动画

```swift
struct AnimatedView: View {
    @State private var scale: CGFloat = 1

    var body: some View {
        Circle()
            .frame(width: 100, height: 100)
            .scaleEffect(scale)
            .animation(.easeInOut(duration: 0.3), value: scale)
            .onTapGesture {
                scale = scale == 1 ? 1.5 : 1
            }
    }
}
```

### 2. 显式动画

```swift
Button("Animate") {
    withAnimation(.spring()) {
        scale = 1.5
    }
}
```

### 3. 动画曲线

```swift
.linear(duration: 1)
.easeIn(duration: 0.5)
.easeOut(duration: 0.5)
.easeInOut(duration: 0.5)
.spring(response: 0.5, dampingFraction: 0.6)
.bouncy                    // iOS 17+
.smooth                    // iOS 17+
.interpolatingSpring(stiffness: 50, damping: 1)
```

### 4. 过渡动画

```swift
struct TransitionView: View {
    @State private var show = false

    var body: some View {
        VStack {
            if show {
                Rectangle()
                    .fill(Color.red)
                    .frame(width: 100, height: 100)
                    .transition(.scale.combined(with: .opacity))
            }
            Button("Toggle") {
                withAnimation { show.toggle() }
            }
        }
    }
}
```

### 5. 自定义动画（iOS 17+）

```swift
struct CustomAnimation: Animation {
    func animate(value: Double, time: Double, context: inout AnimationContext<Double>) -> Double {
        // 自定义插值函数
        return value * sin(time * .pi)
    }
}
```

### 6. PhaseAnimator（iOS 17+）

```swift
PhaseAnimator([false, true]) { phase in
    Circle()
        .scaleEffect(phase ? 1.5 : 1)
} animation: { phase in
    phase ? .bouncy : .linear
}
```

### 7. KeyframeAnimator

```swift
KeyframeAnimator(initialValue: Rocket()) { content in
    content
        .rotationEffect(.degrees(content.angle))
        .offset(x: content.x, y: content.y)
} keyframes: { _ in
    KeyframeTrack(\.x) {
        CubicKeyframe(0, duration: 0.5)
        CubicKeyframe(100, duration: 0.5)
    }
    KeyframeTrack(\.y) {
        CubicKeyframe(0, duration: 0.5)
        CubicKeyframe(-50, duration: 0.5)
        CubicKeyframe(0, duration: 0.5)
    }
    KeyframeTrack(\.angle) {
        CubicKeyframe(0, duration: 1)
        CubicKeyframe(360, duration: 1)
    }
}
```

---

## 七、手势

### 1. 点击

```swift
Text("Tap Me")
    .onTapGesture {
        print("tapped")
    }

Text("Double Tap")
    .onTapGesture(count: 2) {
        print("double tapped")
    }
```

### 2. 长按

```swift
Text("Long Press")
    .onLongPressGesture(minimumDuration: 1) {
        print("long pressed")
    }
```

### 3. 拖动

```swift
DraggableCircle()
    .gesture(
        DragGesture()
            .onChanged { value in
                offset = value.translation
            }
            .onEnded { _ in
                // 复位或保存位置
            }
    )
```

### 4. 缩放与旋转

```swift
Image("photo")
    .scaleEffect(scale)
    .rotationEffect(angle)
    .gesture(
        MagnificationGesture()
            .onChanged { value in scale = value }
    )
    .gesture(
        RotationGesture()
            .onChanged { value in angle = value }
    )
```

### 5. 组合手势

```swift
.simultaneousGesture(dragGesture)
.sequenceGesture(longPress, drag)
.exclusively(before: tap)
```

---

## 八、绘制与自定义视图

### 1. Shape

```swift
struct Triangle: Shape {
    func path(in rect: CGRect) -> Path {
        var path = Path()
        path.move(to: CGPoint(x: rect.midX, y: rect.minY))
        path.addLine(to: CGPoint(x: rect.minX, y: rect.maxY))
        path.addLine(to: CGPoint(x: rect.maxX, y: rect.maxY))
        path.closeSubpath()
        return path
    }
}

Triangle()
    .fill(Color.red)
    .frame(width: 100, height: 100)
```

### 2. Canvas

```swift
Canvas { context, size in
    // 绘制矩形
    context.fill(Path(CGRect(x: 0, y: 0, width: 50, height: 50)),
                 with: .color(.blue))

    // 绘制文字
    context.draw(Text("Hello"), at: CGPoint(x: 100, y: 100))

    // 绘制图片
    if let image = context.resolveSymbol(id: 1) {
        context.draw(image, at: CGPoint(x: 200, y: 200))
    }
} symbols: {
    Image("logo").tag(1)
}
```

### 3. 自定义 View

```swift
struct CircularProgressView: View {
    var progress: Double

    var body: some View {
        ZStack {
            Circle()
                .stroke(Color.gray.opacity(0.3), lineWidth: 10)
            Circle()
                .trim(from: 0, to: progress)
                .stroke(Color.blue, style: StrokeStyle(lineWidth: 10, lineCap: .round))
                .rotationEffect(.degrees(-90))
                .animation(.easeInOut, value: progress)
            Text("\(Int(progress * 100))%")
                .font(.title)
        }
        .frame(width: 150, height: 150)
    }
}

CircularProgressView(progress: 0.75)
```

### 4. TimelineView 定时刷新

```swift
TimelineView(.animation) { context in
    let angle = context.date.timeIntervalSinceReferenceDate * 0.5
    HandView().rotationEffect(.radians(angle))
}
```

---

## 九、与 UIKit 互操作

### 1. UIViewRepresentable

```swift
struct MapView: UIViewRepresentable {
    @Binding var region: MKCoordinateRegion

    func makeUIView(context: Context) -> MKMapView {
        let map = MKMapView()
        map.delegate = context.coordinator
        return map
    }

    func updateUIView(_ uiView: MKMapView, context: Context) {
        uiView.setRegion(region, animated: true)
    }

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }

    class Coordinator: NSObject, MKMapViewDelegate {
        var parent: MapView
        init(_ parent: MapView) { self.parent = parent }

        func mapView(_ mapView: MKMapView, regionDidChangeAnimated animated: Bool) {
            parent.region = mapView.region
        }
    }
}
```

### 2. UIViewControllerRepresentable

```swift
struct ImagePicker: UIViewControllerRepresentable {
    @Binding var image: UIImage?
    @Environment(\.dismiss) var dismiss

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.delegate = context.coordinator
        picker.sourceType = .photoLibrary
        return picker
    }

    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {}

    func makeCoordinator() -> Coordinator { Coordinator(self) }

    class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        let parent: ImagePicker
        init(_ parent: ImagePicker) { self.parent = parent }

        func imagePickerController(_ picker: UIImagePickerController,
                                    didFinishPickingMediaWithInfo info: [String : Any]) {
            parent.image = info[.originalImage] as? UIImage
            parent.dismiss()
        }
    }
}
```

### 3. 在 UIKit 中嵌入 SwiftUI

```swift
let swiftUIView = ContentView()
let hostingController = UIHostingController(rootView: swiftUIView)

// 添加到 UIKit 视图
addChild(hostingController)
view.addSubview(hostingController.view)
hostingController.didMove(toParent: self)
```

---

## 十、列表性能优化

### 1. Identifiable

```swift
struct Item: Identifiable {
    let id: UUID
    var name: String
}

List(items) { item in
    Text(item.name)
}
```

确保 `id` 唯一稳定，避免不必要的 diff。

### 2. 避免在 body 中做重计算

```swift
// ❌ 每次重建都格式化
struct BadItem: View {
    let date: Date
    var body: some View {
        Text(date.formatted(date: .complete, time: .complete))
    }
}

// ✅ 预计算
struct GoodItem: View {
    let formattedDate: String
    var body: some View {
        Text(formattedDate)
    }
}
```

### 3. Equatable 视图

```swift
struct ItemRow: View, Equatable {
    let item: Item

    var body: some View {
        Text(item.name)
    }

    static func == (lhs: ItemRow, rhs: ItemRow) -> Bool {
        lhs.item.id == rhs.item.id && lhs.item.name == rhs.item.name
    }
}

// 使用
List(items) { item in
    ItemRow(item: item).equatable()
}
```

SwiftUI 在 diff 时优先调用 `==`，相等则跳过 body 重算。

### 4. LazyVStack / LazyHStack

```swift
ScrollView {
    LazyVStack {
        ForEach(items) { item in
            ItemRow(item: item)
        }
    }
}
```

类似 `ListView` 的懒加载，仅渲染可见项。

---

## 十一、环境与偏好

### 1. Environment

```swift
// 读取系统环境
struct ThemeView: View {
    @Environment(\.colorScheme) var colorScheme
    @Environment(\.locale) var locale
    @Environment(\.horizontalSizeClass) var sizeClass

    var body: some View {
        Text(colorScheme == .dark ? "Dark" : "Light")
    }
}

// 自定义环境值
struct ThemeKey: EnvironmentKey {
    static let defaultValue: Color = .blue
}
extension EnvironmentValues {
    var themeColor: Color {
        get { self[ThemeKey.self] }
        set { self[ThemeKey.self] = newValue }
    }
}

// 注入
ContentView()
    .environment(\.themeColor, .red)
```

### 2. PreferenceKey

```swift
struct SizePreferenceKey: PreferenceKey {
    static var defaultValue: CGSize = .zero
    static func reduce(value: inout CGSize, nextValue: () -> CGSize) {
        value = nextValue()
    }
}

struct MeasuredView: View {
    var body: some View {
        GeometryReader { geo in
            Text("Hello")
                .preference(key: SizePreferenceKey.self, value: geo.size)
        }
    }
}

// 父视图读取
ParentView()
    .onPreferenceChange(SizePreferenceKey.self) { size in
        print("子视图尺寸: \(size)")
    }
```

---

## 十二、预览

### 1. 基础预览

```swift
#Preview {
    ContentView()
}

#Preview("带数据") {
    ContentView()
        .environmentObject(AppState.preview)
}
```

### 2. 多设备预览

```swift
#Preview {
    ContentView()
        .previewDevice("iPhone 15 Pro")
}

#Preview("iPad") {
    ContentView()
        .previewDevice("iPad Pro 12.9")
}
```

### 3. 暗黑模式预览

```swift
#Preview {
    ContentView()
        .preferredColorScheme(.dark)
}
```

### 4. 状态预览

```swift
#Preview("加载中") {
    ContentView(state: .loading)
}
#Preview("成功") {
    ContentView(state: .loaded)
}
#Preview("失败") {
    ContentView(state: .error)
}
```

---

## 十三、常见问题与最佳实践

### 1. 避免在 View 中持有可变状态

```swift
// ❌ View 应该是纯数据结构
struct BadView: View {
    var counter = 0      // ❌ 不会被保留
    var body: some View { Text("\(counter)") }
}

// ✅ 用 @State
struct GoodView: View {
    @State private var counter = 0
    var body: some View { Text("\(counter)") }
}
```

### 2. 拆分大 View

```swift
// ❌ 单一巨型 View
struct MegaView: View {
    var body: some View {
        VStack {
            header()
            content()
            footer()
            // ... 100 行
        }
    }
}

// ✅ 拆分
struct ContentView: View {
    var body: some View {
        VStack {
            HeaderView()
            ContentView2()
            FooterView()
        }
    }
}
```

### 3. 提取子视图

```swift
struct ItemRow: View {
    let item: Item

    var body: some View {
        HStack {
            ImageView(url: item.imageUrl)
            VStack(alignment: .leading) {
                Text(item.name).font(.headline)
                Text(item.desc).font(.caption)
            }
        }
    }
}
```

### 4. 避免过度使用 GeometryReader

`GeometryReader` 会让父容器失去自适应能力，优先用 `Spacer` / `frame(maxWidth: .infinity)` 等替代。

### 5. 使用 `@MainActor`

SwiftUI 视图操作必须在主线程，模型更新若涉及后台线程需 `@MainActor`：

```swift
@MainActor
class DataModel: ObservableObject {
    @Published var items: [Item] = []

    func load() async {
        let data = await fetchItems()  // 后台
        items = data                    // 主线程更新
    }
}
```

---

## 十四、总结

| 主题 | 核心要点 |
|------|---------|
| 基础 | `View` 协议、`body` 计算属性、`some View` 不透明类型、修饰器链式调用 |
| 视图 | Text/Image/Button/TextField/Toggle/Picker/List |
| 布局 | HStack/VStack/ZStack、Spacer、frame、自定义 Layout、GeometryReader |
| 状态 | `@State`/`@Binding`/`@StateObject`/`@ObservedObject`/`@EnvironmentObject`/`@Observable` |
| 导航 | NavigationStack、TabView、sheet、路径编程式导航 |
| 动画 | 隐式 `animation(_:value:)`、显式 `withAnimation`、过渡、PhaseAnimator、Keyframe |
| 手势 | tapGesture、DragGesture、MagnificationGesture、组合手势 |
| 绘制 | Shape、Canvas、TimelineView、自定义 View |
| 互操作 | UIViewRepresentable、UIViewControllerRepresentable、UIHostingController |
| 性能 | Identifiable、LazyVStack、Equatable View、避免 body 重计算 |
| 环境 | Environment 读取与自定义、PreferenceKey 测量 |
| 预览 | `#Preview`、多设备、暗黑模式、状态预览 |

SwiftUI 的核心理念是 **"UI 是状态的函数"**：状态变化驱动 UI 更新，开发者只需描述 UI 在不同状态下的样子。掌握状态管理（`@State` / `@Binding` / `@Observable`）是写出 Swifty 代码的关键。

进阶方向：
- **Swift Charts**：数据可视化
- **WidgetKit**：桌面小组件
- **App Intents**：Siri 与 Shortcuts 集成
- **StoreKit 2**：应用内购
- **SwiftData**：持久化框架（替代 Core Data）
