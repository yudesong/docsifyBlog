---
title: Flutter 基础部分(二) 
publishedTime: 2026-08-04T10:00:00+08:00
---

Flutter 的状态管理是构建复杂应用时绕不开的核心话题。本文系统梳理 Flutter 中常用的几种状态管理方案，从最基础的 setState 到主流的 Provider、Riverpod、BLoC，再到 Redux 与 GetX，逐一介绍它们的适用场景、使用方式与优缺点对比。

## 一、为什么需要状态管理

### 1. Flutter 的状态是什么

Flutter 中的状态指的是组件在渲染时所依赖的数据。当这些数据发生变化时，组件会重新构建（rebuild）以反映最新的视图。状态可以分为两类：

- **局部状态（Local State）**：只在单个 Widget 内部使用的状态，例如一个按钮的选中态、一个输入框的文本内容。通常用 `setState` 即可处理。
- **全局状态（Global State）**：跨多个 Widget、多个页面共享的状态，例如用户登录信息、购物车数据、主题配置等。这类状态如果继续用 `setState` + 构造函数透传，会导致代码耦合严重、难以维护。

### 2. 状态管理的核心诉求

| 诉求 | 说明 |
|------|------|
| **跨组件共享** | 避免通过构造函数层层透传数据 |
| **响应式更新** | 状态变化后依赖该状态的 Widget 自动重建 |
| **可测试性** | 状态逻辑与视图解耦，便于单元测试 |
| **性能可控** | 只重建依赖变化状态的 Widget，避免整树重建 |
| **生命周期管理** | 状态可随依赖它的 Widget 创建与销毁，避免内存泄漏 |

---

## 二、setState —— 最基础的方式

### 1. 基本用法

`setState` 是 Flutter `StatefulWidget` 内置的状态更新方法，它是所有状态管理方案的底层基础。

```dart
class CounterWidget extends StatefulWidget {
  const CounterWidget({super.key});

  @override
  State<CounterWidget> createState() => _CounterWidgetState();
}

class _CounterWidgetState extends State<CounterWidget> {
  int _count = 0;

  void _increment() {
    setState(() {
      _count++;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text('count: $_count'),
        ElevatedButton(
          onPressed: _increment,
          child: const Text('add'),
        ),
      ],
    );
  }
}
```

### 2. 适用场景

- Widget 内部自包含的局部状态
- 父子组件通过构造函数 + 回调传递的简单场景
- Demo、原型、单页面应用

### 3. 局限性

- 状态无法跨层级共享，深层透传会导致 "prop drilling"（层层透传）
- 父组件 setState 会触发整个子树重建，性能不可控
- 业务逻辑与视图强耦合，难以测试

---

## 三、InheritedWidget —— Flutter 内置的共享方案

### 1. 基本用法

`InheritedWidget` 是 Flutter 框架提供的状态向下传递机制，Provider 的底层就是它。它的核心作用是：让子树中的任意 Widget 都能高效访问到祖先节点共享的数据，而不必通过构造函数一层层传递。

```dart
class CounterInherited extends InheritedWidget {
  final int count;
  final VoidCallback onIncrement;

  const CounterInherited({
    super.key,
    required this.count,
    required this.onIncrement,
    required super.child,
  });

  // 子树通过 dependOnInheritedWidgetOfExactType 获取实例
  static CounterInherited? of(BuildContext context) {
    return context.dependOnInheritedWidgetOfExactType<CounterInherited>();
  }

  // 数据变化时是否通知子树重建
  @override
  bool updateShouldNotify(CounterInherited oldWidget) {
    return count != oldWidget.count;
  }
}

class RootApp extends StatelessWidget {
  const RootApp({super.key});

  @override
  Widget build(BuildContext context) {
    return CounterInherited(
      count: 0,
      onIncrement: () {},
      child: const ChildWidget(),
    );
  }
}

class ChildWidget extends StatelessWidget {
  const ChildWidget({super.key});

  @override
  Widget build(BuildContext context) {
    final state = CounterInherited.of(context)!;
    return Text('${state.count}');
  }
}
```

### 2. 适用场景

- 主题、语言、媒体查询等全局共享数据
- 自研轻量状态管理方案的底层支撑

### 3. 局限性

- 数据是只读的，更新需配合 StatefulWidget 外壳
- 注册、通知逻辑需手写，样板代码较多
- 没有自动依赖追踪，精细粒度重建需要额外设计

---

## 四、Provider —— 官方推荐的基础方案

### 1. 简介

Provider 是 Flutter 官方推荐的状态管理包，本质上是 `InheritedWidget` 的封装，提供了更简洁的 API 和更完善的生命周期管理。它支持 `ChangeNotifier`、`ValueNotifier`、`Future`、`Stream` 等多种数据源。

### 2. 基本用法

**步骤 1：定义 ChangeNotifier**

```dart
class CounterViewModel extends ChangeNotifier {
  int _count = 0;
  int get count => _count;

  void increment() {
    _count++;
    notifyListeners(); // 通知依赖该 ViewModel 的 Widget 重建
  }
}
```

**步骤 2：在顶层注入**

```dart
void main() {
  runApp(
    ChangeNotifierProvider(
      create: (_) => CounterViewModel(),
      child: const MyApp(),
    ),
  );
}
```

**步骤 3：在子 Widget 中消费**

```dart
class CounterPage extends StatelessWidget {
  const CounterPage({super.key});

  @override
  Widget build(BuildContext context) {
    // 只在 count 变化时重建
    final count = context.select<CounterViewModel, int>((vm) => vm.count);
    final vm = context.read<CounterViewModel>();

    return Column(
      children: [
        Text('count: $count'),
        ElevatedButton(
          onPressed: vm.increment,
          child: const Text('add'),
        ),
      ],
    );
  }
}
```

### 3. 常用 API

| API | 作用 | 是否监听变化 |
|-----|------|------------|
| `Provider<T>` | 注入只读数据 | 否 |
| `ChangeNotifierProvider<T>` | 注入 ChangeNotifier | 是 |
| `FutureProvider<T>` | 注入 Future 结果 | 是 |
| `StreamProvider<T>` | 注入 Stream 数据流 | 是 |
| `MultiProvider` | 同时注入多个 Provider | - |
| `context.watch<T>()` | 监听整个对象变化 | 是 |
| `context.select<T, R>(selector)` | 只监听对象的某个字段 | 是（细粒度） |
| `context.read<T>()` | 获取对象但不监听 | 否 |

### 4. 适用场景

- 中小型应用
- 团队刚接触 Flutter 状态管理
- 需要官方背书的稳定方案

### 5. 局限性

- 依赖 `BuildContext`，在非 Widget 层使用不便
- `notifyListeners` 是全局通知，精细粒度依赖 `select`
- 大型项目中 Provider 嵌套层级较深

---

## 五、Riverpod —— Provider 的进化版

### 1. 简介

Riverpod 是 Provider 作者重新设计的方案，解决了 Provider 的几个核心痛点：

- 不依赖 `BuildContext`
- 编译时安全，避免运行时找不到 Provider 的异常
- 默认不可变，更易测试
- 支持自动释放

### 2. 基本用法

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';

// 1. 定义 Provider
final counterProvider = StateNotifierProvider<CounterNotifier, int>((ref) {
  return CounterNotifier();
});

class CounterNotifier extends StateNotifier<int> {
  CounterNotifier() : super(0);

  void increment() => state++;
}

// 2. 在顶层包裹 ProviderScope
void main() {
  runApp(const ProviderScope(child: MyApp()));
}

// 3. 在 Widget 中消费
class CounterPage extends ConsumerWidget {
  const CounterPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(counterProvider);
    return Column(
      children: [
        Text('count: $count'),
        ElevatedButton(
          onPressed: () => ref.read(counterProvider.notifier).increment(),
          child: const Text('add'),
        ),
      ],
    );
  }
}
```

### 3. Provider 类型

| 类型 | 说明 |
|------|------|
| `Provider` | 只读数据 |
| `StateProvider` | 简单可变状态（适合基本类型） |
| `StateNotifierProvider` | 复杂状态逻辑（推荐） |
| `FutureProvider` | 异步数据 |
| `StreamProvider` | 流数据 |
| `ChangeNotifierProvider` | 兼容旧的 ChangeNotifier |

### 4. 适用场景

- 中大型应用
- 重视编译时安全与可测试性
- 团队已熟悉 Provider，希望升级

### 5. 局限性

- 学习曲线略陡，概念较多
- 生态相比 Provider 略小
- 旧项目迁移成本较高

---

## 六、BLoC / Cubit —— 响应式流方案

### 1. 简介

BLoC（Business Logic Component）由 Google 推出，核心思想是使用 `Stream` 作为输入（Event）和输出（State）的桥梁，实现业务逻辑与 UI 的完全解耦。Cubit 是 BLoC 的简化版，去掉了 Event 概念，直接暴露方法调用。

### 2. Cubit 基本用法（推荐入门）

```dart
import 'package:flutter_bloc/flutter_bloc.dart';

// 1. 定义状态
class CounterState {
  final int count;
  const CounterState(this.count);
}

// 2. 定义 Cubit
class CounterCubit extends Cubit<CounterState> {
  CounterCubit() : super(const CounterState(0));

  void increment() => emit(CounterState(state.count + 1));
}

// 3. 顶层注入
void main() {
  runApp(
    BlocProvider(
      create: (_) => CounterCubit(),
      child: const MyApp(),
    ),
  );
}

// 4. 消费
class CounterPage extends StatelessWidget {
  const CounterPage({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocBuilder<CounterCubit, CounterState>(
      builder: (context, state) {
        return Column(
          children: [
            Text('count: ${state.count}'),
            ElevatedButton(
              onPressed: () => context.read<CounterCubit>().increment(),
              child: const Text('add'),
            ),
          ],
        );
      },
    );
  }
}
```

### 3. BLoC 进阶用法（Event + State）

```dart
// 1. 定义事件
abstract class CounterEvent {}
class IncrementEvent extends CounterEvent {}

// 2. 定义状态
class CounterState {
  final int count;
  const CounterState(this.count);
}

// 3. 定义 Bloc
class CounterBloc extends Bloc<CounterEvent, CounterState> {
  CounterBloc() : super(const CounterState(0)) {
    on<IncrementEvent>((event, emit) {
      emit(CounterState(state.count + 1));
    });
  }
}
```

### 4. 适用场景

- 大型应用，业务逻辑复杂
- 强调可测试性与状态可追溯
- 团队习惯响应式编程

### 5. 局限性

- 样板代码较多，简单场景过重
- Event / State 概念学习成本不低
- 与原生 Widget 配合需 BlocBuilder / BlocSelector，写法稍繁

---

## 七、Redux —— 单一数据源方案

### 1. 简介

Redux 借鉴自前端 React 生态，核心三要素：

- **单一数据源**：整个应用的状态存储在一棵状态树中
- **状态只读**：只能通过派发 Action 触发变更
- **纯函数变更**：通过 Reducer 处理 Action 并返回新状态

### 2. 基本用法

```dart
import 'package:redux/redux.dart';
import 'package:flutter_redux/flutter_redux.dart';

// 1. 定义 State
class AppState {
  final int count;
  const AppState({required this.count});
}

// 2. 定义 Action
enum CounterAction { increment }

// 3. 定义 Reducer
AppState counterReducer(AppState state, dynamic action) {
  if (action == CounterAction.increment) {
    return AppState(count: state.count + 1);
  }
  return state;
}

// 4. 创建 Store
final store = Store<AppState>(
  counterReducer,
  initialState: const AppState(count: 0),
);

// 5. 顶层注入
void main() {
  runApp(StoreProvider(store: store, child: const MyApp()));
}

// 6. 消费
class CounterPage extends StatelessWidget {
  const CounterPage({super.key});

  @override
  Widget build(BuildContext context) {
    return StoreConnector<AppState, int>(
      converter: (store) => store.state.count,
      builder: (context, count) {
        return Column(
          children: [
            Text('count: $count'),
            ElevatedButton(
              onPressed: () =>
                  StoreProvider.of<AppState>(context).dispatch(CounterAction.increment),
              child: const Text('add'),
            ),
          ],
        );
      },
    );
  }
}
```

### 3. 适用场景

- 团队有 React Redux 背景
- 需要状态可追溯、时间旅行调试
- 极大型应用，状态树复杂

### 4. 局限性

- 样板代码极多
- 异步处理需要引入中间件（redux_thunk、redux_epics）
- Flutter 生态活跃度低于 Provider/Riverpod/BLoC

---

## 八、GetX —— 一站式方案

### 1. 简介

GetX 是一个集状态管理、路由管理、依赖注入、国际化于一体的框架。它的核心卖点是：**简单**、**高性能**、**低侵入**。

### 2. 基本用法

```dart
import 'package:get/get.dart';

// 1. 定义 Controller
class CounterController extends GetxController {
  final count = 0.obs; // 响应式变量

  void increment() => count.value++;
}

// 2. 在视图中使用
class CounterPage extends StatelessWidget {
  const CounterPage({super.key});

  final CounterController controller = Get.put(CounterController());

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Obx(() => Text('count: ${controller.count.value}')),
        ElevatedButton(
          onPressed: controller.increment,
          child: const Text('add'),
        ),
      ],
    );
  }
}
```

### 3. 适用场景

- 快速开发、中小型项目
- 个人开发者希望少写样板代码
- 需要路由 + 状态 + 依赖注入一站式解决

### 4. 局限性

- 过度封装，隐藏 Flutter 原生机制，不利于学习底层
- 全局依赖 GetX 生态，迁移成本高
- 社区对其架构设计存在争议，大型项目慎用

---

## 九、方案横向对比

| 方案 | 学习成本 | 样板代码 | 性能 | 可测试性 | 适用规模 | 推荐度 |
|------|---------|---------|------|---------|---------|--------|
| setState | ⭐ | ⭐ | 中 | 低 | 极小 | 必学基础 |
| InheritedWidget | ⭐⭐ | ⭐⭐⭐ | 中 | 中 | 小 | 理解底层 |
| Provider | ⭐⭐ | ⭐⭐ | 中高 | 中 | 中 | ⭐⭐⭐⭐ |
| Riverpod | ⭐⭐⭐ | ⭐⭐ | 高 | 高 | 中大 | ⭐⭐⭐⭐⭐ |
| BLoC / Cubit | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 高 | 高 | 大 | ⭐⭐⭐⭐ |
| Redux | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 中高 | 高 | 极大 | ⭐⭐ |
| GetX | ⭐ | ⭐ | 高 | 中 | 中 | ⭐⭐⭐ |

### 选型建议

- **初学者**：先吃透 `setState` 与 `InheritedWidget`，再学 `Provider`
- **中小型项目**：`Provider` 或 `Riverpod`
- **大型项目**：`Riverpod` 或 `BLoC`
- **团队有前端背景**：`Redux` 或 `BLoC`
- **快速原型**：`GetX`（生产项目慎用）

---

## 十、总结

Flutter 的状态管理方案没有银弹，每种方案都有它的适用场景。理解它们背后的设计思想比记住 API 更重要：

- **setState** 是所有方案的底层基础
- **InheritedWidget** 是 Provider 的实现原理
- **Provider / Riverpod** 通过依赖注入解决共享问题
- **BLoC** 通过 Stream 实现响应式解耦
- **Redux** 通过单一数据源实现可追溯的状态变化
- **GetX** 用响应式变量 + 依赖注入换取开发效率

建议按学习路径 `setState → Provider → Riverpod → BLoC` 逐层深入，根据项目规模与团队背景选型，而不是盲目追新。
