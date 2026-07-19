---
title: Flutter 跨组件传递数据都有哪些方式
url: https://juejin.cn/post/6930880609883553800
publishedTime: 2021-02-19T15:57:29+08:00
---

## 1、InheritedWidget

InheritedWidget是Flutter中非常重要的一个功能型Widget，它可以高效的将数据在Widget树中向下传递、共享，这在一些需要在Widget树中共享数据的场景中非常方便，如Flutter中，正是通过InheritedWidget来共享应用主题(Theme)和Locale(当前语言环境)信息的。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f29dfa233527435b9ad1d7ea7bdb1a3b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

假如我们Widget要往Widget3传值，一般情况下我们是这么写的。一层一层的传值，这样写十分麻烦。

点击按钮，count+1，在最下面的一个widget上面显示

```scala
scala 代码解读复制代码class InheritedDemo extends StatefulWidget {
  @override
  _InheritedDemoState createState() => _InheritedDemoState();
}

class _InheritedDemoState extends State<InheritedDemo> {
  int count = 0;
  @override
  Widget build(BuildContext context) {
    return Center(
      child: Column(
        children: [
          Test1(count),
          RaisedButton(
              child: Text('我是按钮'),
              onPressed: () => setState(() {
                count++;
              }))
        ],
      ),
    );
  }
}

class Test1 extends StatelessWidget {
 final count;
 Test1(this.count);
  @override
  Widget build(BuildContext context) {
    return Test2(count);
  }
}
class Test2 extends StatelessWidget {
 final count;
 Test2(this.count);
  @override
  Widget build(BuildContext context) {
    return Test3(count);
  }
}
class Test3 extends StatefulWidget {
 final count;
 Test3(this.count);
  @override
  _Test3State createState() => _Test3State();
}
class _Test3State extends State<Test3> {
  @override
  Widget build(BuildContext context) {
    return Text(widget.count.toString());
  }
}
```

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3608ef33ac594fb09e17cb9e51347afb~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

Flutter 给我们提供了一个InheritedWidget组件，来帮助我们完成上面功能。

首先我们需要定义一个继承InheritedWidget的widget MyData

```scala
scala 代码解读复制代码class MyData extends InheritedWidget {
  MyData({this.data, Widget child}) : super(child: child);

  final int data; //需要在子树中共享的数据，保存点击次数

  //定义一个便捷方法，方便子树中的widget获取共享数据
  static MyData of(BuildContext context) {
    return context.dependOnInheritedWidgetOfExactType<MyData>();
  }

  //该回调决定当data发生变化时，是否通知子树中依赖data的Widget
  @override
  bool updateShouldNotify(MyData old) {
    //如果返回true，则子树中依赖(build函数中有调用)本widget
    //的子widget的\`state.didChangeDependencies\`会被调用
    return old.data != data;
  }
}
```

然后使用 `MyData` 包裹整个widget

```scala
scala 代码解读复制代码class InheritedDemo extends StatefulWidget {
  @override
  _InheritedDemoState createState() => _InheritedDemoState();
}

class _InheritedDemoState extends State<InheritedDemo> {
  int count = 0;
  @override
  Widget build(BuildContext context) {
    return MyData(
        data: count,
        child: Column(
          children: <Widget>[
            Test1(),
            RaisedButton(
              child: Text('我是按钮'),
              onPressed: () => setState(() {
                count++;
              }),
            )
          ],
        ));
  }
}

class Test1 extends StatelessWidget {
//  final count;
//  Test1(this.count);
  @override
  Widget build(BuildContext context) {
    return Test2();
  }
}

class Test2 extends StatelessWidget {
//  final count;
//  Test2(this.count);
  @override
  Widget build(BuildContext context) {
    return Test3();
  }
}

class Test3 extends StatefulWidget {
//  final count;
//  Test3(this.count);
  @override
  _Test3State createState() => _Test3State();
}

class _Test3State extends State<Test3> {
  @override
  Widget build(BuildContext context) {
    return Text(MyData.of(context).data.toString());
  }

  @override
  void didChangeDependencies() {
    print("didChangeDependencies来了!");
    super.didChangeDependencies();
  }
}
```

## 2、Notification

Notification 是 Flutter 中进行跨层数据共享的另一个重要的机制。如果说 InheritedWidget 的数据流动方式是从父 Widget 到子 Widget 逐层传递，那 Notificaiton 则恰恰相反，数据流动方式是从子 Widget 向上传递至父 Widget。这样的数据传递机制适用于子 Widget 状态变更，发送通知上报的场景。

Flutter中将这种由子向父的传递通知的机制称为通知冒泡（Notification Bubbling）。通知冒泡和用户触摸事件冒泡是相似的，但有一点不同：通知冒泡可以中止，但用户触摸事件不行。

**如果想要实现自定义通知，我们首先需要继承 Notification 类。Notification 类提供了 dispatch 方法，可以让我们沿着 context 对应的 Element 节点树向上逐层发送通知**

### 1、定义一个通知类，要继承自Notification类

```scala
scala 代码解读复制代码class MyNotification extends Notification {
  MyNotification(this.msg);
  final String msg;
}
```

### 2、分发通知

Notification有一个dispatch(context)方法，它是用于分发通知的，我们说过context实际上就是操作Element的一个接口，它与Element树上的节点是对应的，通知会从context对应的Element节点向上冒泡。

```scala
scala 代码解读复制代码//子widget
class ChildNotificationWidget extends StatefulWidget {
  @override
  _ChildNotificationWidgetState createState() => _ChildNotificationWidgetState();
}

class _ChildNotificationWidgetState extends State<ChildNotificationWidget> {
  @override
  Widget build(BuildContext context) {
    return RaisedButton(
      // 按钮点击时分发通知
      onPressed: () => MyNotification("你好，Flutter").dispatch(context),
      child: Text("Fire Notification"),
    );
  }
}

//父widget
class parentNotificationWidget extends StatefulWidget {
  @override
  _parentNotificationWidgetState createState() => _parentNotificationWidgetState();
}

class _parentNotificationWidgetState extends State<parentNotificationWidget> {
  String _msg = " 通知：";
  @override
  Widget build(BuildContext context) {
    // 监听通知
    return NotificationListener<MyNotification>(
        onNotification: (notification) {
          setState(() {_msg += notification.msg+"  ";});// 收到子 Widget 通知，更新 msg
        },
        child:Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            Text(_msg),
            ChildNotificationWidget()],// 将子 Widget 加入到视图树中
        )
    );
  }
}
```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dc67bf5ee27b4588ad2c0f36fd5d7e15~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

## 3、事件总线EventBus

无论是 InheritedWidget 还是 Notificaiton，它们的使用场景都需要依靠 Widget 树，也就意味着只能在有父子关系的 Widget 之间进行数据共享。但是，组件间数据传递还有一种常见场景：这些组件间不存在父子关系。这时，事件总线 EventBus 就登场了。

事件总线是在 Flutter 中实现跨组件通信的机制。它遵循发布 / 订阅模式，允许订阅者订阅事件，当发布者触发事件时，订阅者和发布者之间可以通过事件进行交互。发布者和订阅者之间无需有父子关系，甚至非 Widget 对象也可以发布 / 订阅。这些特点与其他平台的事件总线机制是类似的。

接下来，我们通过一个跨页面通信的例子，来看一下事件总线的具体使用方法。需要注意的是，EventBus 是一个第三方插件，因此我们需要在 pubspec.yaml 文件中声明它：

```yaml
yaml 代码解读复制代码dependencies:
  event_bus: ^1.1.1
```

本节我们实现一个简单的全局事件总线，我们使用单例模式，代码如下：

```
ini 代码解读复制代码//订阅者回调签名
typedef void EventCallback(arg);

class EventBus {
  //私有构造函数
  EventBus._internal();

  //保存单例
  static EventBus _singleton = new EventBus._internal();

  //工厂构造函数
  factory EventBus()=> _singleton;

  //保存事件订阅者队列，key:事件名(id)，value: 对应事件的订阅者队列
  var _emap = new Map<Object, List<EventCallback>>();

  //添加订阅者
  void on(eventName, EventCallback f) {
    if (eventName == null || f == null) return;
    _emap[eventName] ??= new List<EventCallback>();
    _emap[eventName].add(f);
  }

  //移除订阅者
  void off(eventName, [EventCallback f]) {
    var list = _emap[eventName];
    if (eventName == null || list == null) return;
    if (f == null) {
      _emap[eventName] = null;
    } else {
      list.remove(f);
    }
  }

  //触发事件，事件触发后该事件所有订阅者会被调用
  void emit(eventName, [arg]) {
    var list = _emap[eventName];
    if (list == null) return;
    int len = list.length - 1;
    //反向遍历，防止订阅者在回调中移除自身带来的下标错位
    for (var i = len; i > -1; --i) {
      list[i](arg);
    }
  }
}

//定义一个top-level（全局）变量，页面引入该文件后可以直接使用bus
var bus = new EventBus();
```

> 注意：Dart中实现单例模式的标准做法就是使用static变量+工厂构造函数的方式，这样就可以保证new EventBus()始终返回都是同一个实例，

上面代码转载自： [事件总线](https://link.juejin.cn/?target=https%3A%2F%2Fbook.flutterchina.club%2Fchapter8%2Feventbus.html "https://book.flutterchina.club/chapter8/eventbus.html")

```scala
scala 代码解读复制代码class EventBusPage extends StatefulWidget {
  @override
  _EventBusPageState createState() => _EventBusPageState();
}

class _EventBusPageState extends State<EventBusPage> {
  String msg = "通知：";
  @override
  void initState() {
    // TODO: implement initState
    bus.on("eventName", (arg) {
      print(arg);
      setState(() {
        msg += arg;
      });
    });
    super.initState();
  }
  dispose() {
    bus.off("eventName");//State销毁时，清理注册
    super.dispose();
  }
  @override
  Widget build(BuildContext context) {
    return Container(
      child: Column(
        children: [
          Text(msg),
          RaisedButton(
            child: Text('进入二级界面'),
              onPressed: (){
                Navigator.push(context, MaterialPageRoute(builder: (BuildContext context){
                  return EventBusTwoPage();
                }));
              })
        ],
      ),
    );
  }
}

class EventBusTwoPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text("EventBusTwoPage"),),
      body: Center(
        child: RaisedButton(
          child: Text('二级界面'),
          onPressed: (){
            bus.emit("eventName","二级页面来通知了");
            Navigator.pop(context);
          },
        ),
      )
    );
  }
}
```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/699c7e4d3cb244eb9880391d6a3601a9~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

代码地址： [github.com/SunshineBro…](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2FSunshineBrother%2FFlutterDemo "https://github.com/SunshineBrother/FlutterDemo")

