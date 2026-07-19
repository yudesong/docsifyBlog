---
title: Flutter Widget生命周期&渲染原理详解
url: https://juejin.cn/post/6933020064299876366
publishedTime: 2021-02-25T10:19:13+08:00
---

## Widget生命周期

flutter生命周期其实就是Widget的生命周期，生命周期的回调函数体现在了State上面。 主要可以分成两方面讨论

## 1、StatelessWidget

StatelessWidget比较简单，主要有 `构造方法` 和 `build方法`

```scala
scala 代码解读复制代码class MyStatelessWidget extends StatelessWidget {
  final String title;
  MyStatelessWidget({this.title}){
    print('构造函数被调用了!');
  }
  @override
  Widget build(BuildContext context) {
    print('build方法被调用了!');
    return Container();
  }
}
```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a313890ba4fc4eff8dca867cba860dd6~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

## 2、StatefulWidget

StatefulWidget就有点复杂了，写一个StatefulWidget的时候，系统默认给我们创建了一个 `Widget` 和 `State` ，所以StatefulWidget的生命周期包括了两部分

- 1、Widget的生命周期
- 2、State的生命周期

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9e4a2b05e9a446668fb3566c679203f7~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

**大致可以看成三个阶段**

- 初始化（插入渲染树）
  - 1、Widget的构造方法
    - 2、Widget的CreateState
    - 3、State的构造方法
    - 4、State的initState方法
- 状态改变（在渲染树中存在）
  - 1、didChangeDependencies方法 (改变依赖关系)：依赖的InheritedWidget发生变化之后， 方法也会调用!
    - 2、State的build：当调用setState方法。 会重新调用build进行渲染!
    - 3、didUpdateWidget：判断是否要更新widget树
- 销毁（从渲染树种移除）
  - 1、deactivate：在dispose之前，会调用这个函数
    - 2、当Widget销毁的时候， 调用State的dispose

### 1、初始化

我们现在随便写一个 `StatefulWidget` 控件

```scala
scala 代码解读复制代码class HomePage extends StatefulWidget {
  @override
  _HomePageState createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  @override
  Widget build(BuildContext context) {
    return Container();
  }
}
```

其实上面包含了初始化阶段的所有内容了

```scala
scala 代码解读复制代码class HomePage extends StatefulWidget {
  HomePage(){
    print('Widget的构造方法');
  }
  @override
  _HomePageState createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  _HomePageState(){
    print('State的构造方法');
  }
  @override
  void initState() {
    print('State的init来了!');
    super.initState();
  }
  @override
  Widget build(BuildContext context) {
    return Container();
  }
}
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1f4a03a2ff3243ed868ea1e0af78f5a4~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

- 1、 `HomePage()` ：Widget的构造方法
- 2、 `createState()` ：createState 是 StatefulWidget 里创建 State 的方法，当要创建新的 StatefulWidget 的时候，会立即执行 createState，而且只执行一次，createState 必须要实现：
- 3、 `initState` ：前面的 createState 是在创建 StatefulWidget 的时候会调用，initState 是 StatefulWidget 创建完后调用的第一个方法，而且只执行一次，类似于 Android 的 onCreate、iOS 的 viewDidLoad()，所以在这里 View 并没有渲染，但是这时 StatefulWidget 已经被加载到渲染树里了，这时 StatefulWidget 的 mount 的值会变为 true，直到 dispose 调用的时候才会变为 false。可以在 initState 里做一些初始化的操作。

### 2、状态改变（在渲染树中存在）

- 1、 `didChangeDependencies` ：当 StatefulWidget 第一次创建的时候，didChangeDependencies 方法会在 initState 方法之后立即调用，之后当 StatefulWidget 刷新的时候，就不会调用了，除非你的 StatefulWidget 依赖的 InheritedWidget 发生变化之后，didChangeDependencies 才会调用，所以 didChangeDependencies 有可能会被调用多次

我们在上面 `State` 方法里面重写 `didChangeDependencies`

```
scss 代码解读复制代码@override
  void didChangeDependencies() {
    print('didChangeDependencies');
    super.didChangeDependencies();
  }
```

我们重新运行打印结果

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/66b48be60c4142ffa05dbe779103fead~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

- 2、 `build`

在 StatefulWidget 第一次创建的时候，build 方法会在 didChangeDependencies 方法之后立即调用，另外一种会调用 build 方法的场景是，每当 UI 需要重新渲染的时候，build 都会被调用，所以 build 会被多次调用，然后 返回要渲染的 Widget。千万不要在 build 里做除了创建 Widget 之外的操作，因为这个会影响 UI 的渲染效率。

- 3、 `didUpdateWidget` 只要在父widget中调用setState，子widget的didUpdateWidget就一定会被调用，不管父widget传递给子widget构造方法的参数有没有改变。

我们创建一个 `StfulItem`

```scala
scala 代码解读复制代码class StfulItem extends StatefulWidget {
  final title;
  StfulItem(this.title, {Key key}) : super(key: key);
  @override
  _StfulItemState createState() => _StfulItemState();
}

class _StfulItemState extends State<StfulItem> {
  @override
  void didUpdateWidget(covariant StfulItem oldWidget) {
    // TODO: implement didUpdateWidget
    print('子widget要didUpdateWidget');
    super.didUpdateWidget(oldWidget);
  }
  @override
  Widget build(BuildContext context) {
    return Container(
      width: 100,
      height: 100,
      color: randomColor(),
      child: Text(widget.title),
    );
  }
}
```

```
scss 代码解读复制代码class _HomePageState extends State<HomePage> {
  List<Widget> items = [    StfulItem(      'aaaaa',    ),    StfulItem(      'bbbbb',    ),    StfulItem(      'ccccc',    ),  ];
  _HomePageState(){
    print('State的构造方法');
  }
  @override
  void initState() {
    print('State的init来了!');
    super.initState();
  }
  @override
  void didChangeDependencies() {
    print('didChangeDependencies');
    super.didChangeDependencies();
  }

  @override
  Widget build(BuildContext context) {
    print('build方法被调用了!');
    return Column(
      children: <Widget>[
        RaisedButton(
          child: Text('移除'),
          onPressed: () {
            setState(() {
              items.removeAt(0);
            });
          },
        ),
        Row(
          mainAxisAlignment: MainAxisAlignment.center,
          children: items,
        )

      ],
    );
  }
}
```

我们在父widget和子widget中都重写 `didUpdateWidget` 方法，点击按钮打印

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dd39dc3acf4b49a8a480dea9ddf81ac3~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

### 3、销毁（从渲染树种移除）

```typescript
typescript 代码解读复制代码//当State对象从渲染树中移出的时候,就会调用!即将销毁!
  @override
  void deactivate() {
    super.deactivate();
  }
  //当Widget销毁的时候， 调用State的dispose
  @override
  void dispose() {
    print('State的dispose');
    super.dispose();
  }
```

## Widget渲染原理

在进行 App 开发时，我们往往会关注的一个问题是：如何结构化地组织视图数据，提供给渲染引擎，最终完成界面显示。

通常情况下，不同的 UI 框架中会以不同的方式去处理这一问题，但无一例外地都会用到视图树（View Tree）的概念。而 Flutter 将视图树的概念进行了扩展，把视图数据的组织和渲染抽象为三部分，即 Widget，Element 和 RenderObject。

这三部分之间的关系，如下所示：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7d342f274112437d87aa2f72da579f8e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

Flutter 是 UI 框架，Flutter 内一切皆 Widget ，每个 Widget 状态都代表了一帧，Widget 是不可变的。 那么 Widget 是怎么工作的呢？

```scala
scala 代码解读复制代码class LifeCyclePage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Widget生命周期&渲染原理'),),
      body: Container(
        child: Text('Widget生命周期&渲染原理'),
      ),
    );
  }
}
```

我们写一段简单的widget代码，我们发现widget更像是配置文件。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/362eec4255734f25be7a5f259893c1e4~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

如上图所示，当一个 Widget 被“加载“的时候，它并不是马上被绘制出来，而是会对应先创建出它的 Element ，然后通过 Element 将 Widget 的配置信息转化为 RenderObject 实现绘制。

**在 Flutter 中大部分时候我们写的是 Widget ，但是 Widget 的角色反而更像是“配置文件” ，真正触发工作的其实是 RenderObject**

- Widget 是配置文件。
- Element 是桥梁和仓库。
- RenderObject 是解析后的绘制和布局。

Widget 和我们以前的布局概念不一样，因为 Widget 是不可变的（immutable），且只有一帧，且不是真正工作的对象，每次画面变化，都会导致一些 Widget 重新 build 。

## Widget与Element

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ac1350989a804846969569321c52c760~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

并不是所有的Widget都会被独立渲染!只有继承RenderObjectWidget的才会创建RenderObject对象!

每一个Widget都会 创建一个Element对象，会隐式的调用 `createElement()` 方法，将Element加入Element树中，Element主要有三种

- 1、RenderElement主要创建RenderObject对象，继承RenderWidget的widget会创建RenderObject对象
  - 1、创建 `RenderElement`
    - 2、Flutter会调用 `mount` 方法，调用 `createRanderObject` 方法
- 2、StatelessElement继承ComponentElement，StatelessWidget会创建StatelessElement
  - 主要就是调用 `build` 方法，将自己（Element）传出去
- 3、 `StatefulElement` 继承 `ComponentElement` ，StatefulWidget会创建 `StatefulElement`
  - 1、调用 `createState` 方法，创建 `state`
    - 2、将widget赋值给state
    - 3、调用state中的build方法，并且将自己（Element）传出去

`Widget build(BuildContext context) {}` 里面的 `context` 就是 `widget的Element`

### 1、RenderElement

我们随便找一个继承RenderObjectWidget的Widget，如 `Column`

我们进入源码查看继承关系， 点击 `Column`

```
ini 代码解读复制代码Column({
    Key key,
    MainAxisAlignment mainAxisAlignment = MainAxisAlignment.start,
    MainAxisSize mainAxisSize = MainAxisSize.max,
    CrossAxisAlignment crossAxisAlignment = CrossAxisAlignment.center,
    TextDirection textDirection,
    VerticalDirection verticalDirection = VerticalDirection.down,
    TextBaseline textBaseline,
    List<Widget> children = const <Widget>[],
  }) : super()
```

我们点击 `super()` 查看 `Column` 的父类

```kotlin
kotlin 代码解读复制代码Flex({
    Key key,
    @required this.direction,
    this.mainAxisAlignment = MainAxisAlignment.start,
    this.mainAxisSize = MainAxisSize.max,
    this.crossAxisAlignment = CrossAxisAlignment.center,
    this.textDirection,
    this.verticalDirection = VerticalDirection.down,
    this.textBaseline = TextBaseline.alphabetic,
    this.clipBehavior = Clip.hardEdge,
    List<Widget> children = const <Widget>[],
  })

  super(key: key, children: children);
```

这里我们进入了 `Flex` 组件，所以 `Column` 的父类是 `Flex` 组件

我们点击我们点击 `super()` 查看 `Flex` 的父类

```scala
scala 代码解读复制代码abstract class MultiChildRenderObjectWidget extends RenderObjectWidget {

  MultiChildRenderObjectWidget({ Key key, this.children = const <Widget>[] })
      }()), // https://github.com/dart-lang/sdk/issues/29276
      super(key: key);

  final List<Widget> children;

  @override
  MultiChildRenderObjectElement createElement() => MultiChildRenderObjectElement(this);
}
```

**这里我们找到了创建 `Element` 的方法 `createElement()`** ，这个是重写父类方法，同时这个方法还调用了 `MultiChildRenderObjectElement(this)` 。同时我们还发现 `Flex` 组件是继承 `RenderObjectWidget` 组件的

我们点击 `MultiChildRenderObjectElement` ，进入到了 `RenderObjectElement`

```scala
scala 代码解读复制代码class MultiChildRenderObjectElement extends RenderObjectElement {
    MultiChildRenderObjectElement(MultiChildRenderObjectWidget widget)
    : assert(!debugChildrenHaveDuplicateKeys(widget, widget.children)),
      super(widget);
}
```

我们点击 `super`

```scala
scala 代码解读复制代码abstract class RenderObjectElement extends Element {}
```

我们找到了最底层的 `Element`

Flutter会调用 `mount` 方法，调用 `createRanderObject` 方法。

我们全局搜索 `void mount` 找到 `mount` 方法,我们在里面找到了 ` _renderObject = widget.createRenderObject(this);`方法

### 2、StatelessElement

我们随便创建一个 `StatelessWidget`

```scala
scala 代码解读复制代码class MyStatelessWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Container();
  }
}
```

`StatelessWidget` 是如何跟 `StatelessElement` 产生联系的呢？

我们点击 `StatelessWidget` 进入查看源码

```scala
scala 代码解读复制代码abstract class StatelessWidget extends Widget {
  const StatelessWidget({ Key key }) : super(key: key);

  StatelessElement createElement() => StatelessElement(this);

  @protected
  Widget build(BuildContext context);
}
```

这里我们发现了 `createElement` 方法和继承自父类的 `build` 方法

我们点击 `StatelessElement` 继续查看

```scala
scala 代码解读复制代码class StatelessElement extends ComponentElement {
  StatelessElement(StatelessWidget widget) : super(widget);

  @override
  StatelessWidget get widget => super.widget as StatelessWidget;

  @override
  Widget build() => widget.build(this);

  @override
  void update(StatelessWidget newWidget) {
    super.update(newWidget);
    assert(widget == newWidget);
    _dirty = true;
    rebuild();
  }
}
```

- 1、我们发现 `StatelessElement` 是继承自 `ComponentElement` 的
- 2、主要就是调用build方法，将自己（Element）传出去，这个 `this` 就是 `Element`

```
less 代码解读复制代码  @override
  Widget build() => widget.build(this);
```

到这里我们就找到了 `StatelessWidget` 是如何跟 `StatelessElement` 产生联系的

### 3、StatefulElement

我们随便创建一个 `StatefulWidget`

```scala
scala 代码解读复制代码class MyStatefulWidget extends StatefulWidget {
  @override
  _MyStatefulWidgetState createState() => _MyStatefulWidgetState();
}

class _MyStatefulWidgetState extends State<MyStatefulWidget> {
  @override
  Widget build(BuildContext context) {
    return Container();
  }
}
```

我们点击 `StatefulWidget` 进入源码查看

```scala
scala 代码解读复制代码abstract class StatefulWidget extends Widget {
  const StatefulWidget({ Key key }) : super(key: key);

  @override
  StatefulElement createElement() => StatefulElement(this);

  @protected
  @factory
  State createState();
}
```

我们这里找到了创建 `StatefulElement` 的方法 `createElement()`

我们点击 `StatefulElement` 进入源码查看

```scala
scala 代码解读复制代码class StatefulElement extends ComponentElement {
  StatefulElement(StatefulWidget widget)
      : _state = widget.createState(),

  @override
  Widget build() => _state.build(this);
}
```

根据上面源码分析，我们发现

StatefulElement继承ComponentElement，StatefulWidget会创建StatefulElement

- 1、调用createState方法，创建state
- 2、将widget赋值给state
- 3、调用state中的build方法，并且将自己（Element）传出去

代码地址： [github.com/SunshineBro…](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2FSunshineBrother%2FFlutterDemo "https://github.com/SunshineBrother/FlutterDemo")

