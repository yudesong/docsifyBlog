---
title: Flutter 单线程的Dart为何能够流程运行UI
url: https://juejin.cn/post/6933043674343604237
publishedTime: 2021-02-25T11:51:49+08:00
---

## Dart异步原理

Dart 是一门单线程编程语言。对于平时用 iOS 的同学，首先可能会反应：那如果一个操作耗时特别长，不会一直卡住主线程吗？比如iOS，为了不阻塞UI主线程，我们不得不通过另外的线程来发起耗时操作（网络请求/访问本地文件等），然后再通过Handler来和UI线程沟通。Dart 究竟是如何做到的呢？

先给答案： **异步 IO + 事件循环**

## 1、I/O 模型

我们先来看看阻塞IO是什么样的：

```arduino
String text = io.read(buffer); //阻塞等待
```

> 注： IO 模型是操作系统层面的，这一小节的代码都是伪代码，只是为了方便理解。

当相应线程调用了 `read` 之后，它就会一直在那里等着结果返回，什么也不干，这是阻塞式的IO

**这里普及两个概念：阻塞式调用和非阻塞式调用**

阻塞和非阻塞关注的是程序在等待调用结果（消息，返回值）时的状态

- **阻塞式调用** ： 调用结果返回之前，当前线程会被挂起，调用线程只有在得到调用结果之后才会继续执行。
- **非阻塞式调用** ： 调用执行之后，当前线程不会停止执行，只需要过一段时间来检查一下有没有结果返回即可。

但我们的应用程序经常是要同时处理好几个IO的，即便一个简单的手机App，同时发生的IO可能就有：用户手势（输入），若干网络请求(输入输出)，渲染结果到屏幕（输出）；更不用说是服务端程序，成百上千个并发请求都是家常便饭

有人说，这种情况可以使用多线程啊。这确实是个思路，但受制于CPU的实际并发数，每个线程只能同时处理单个IO，性能限制还是很大，而且还要处理不同线程之间的同步问题，程序的复杂度大大增加。

如果进行IO的时候不用阻塞，那情况就不一样了：

```
while(true){
  for(io in io_array){
      status = io.read(buffer);// 不管有没有数据都立即返回
      if(status == OK){

      }
  }
}
```

有了非阻塞IO，通过轮询的方式，我们就可以对多个IO进行同时处理了，但这样也有一个明显的缺点：在大部分情况下，IO都是没有内容的（CPU的速度远高于IO速度），这样就会导致CPU大部分时间在空转，计算资源依然没有很好得到利用。

为了进一步解决这个问题，人们设计了IO多路转接（IO multiplexing），可以对多个IO监听和设置等待时间：

```
while(true){
    //如果其中一路IO有数据返回，则立即返回；如果一直没有，最多等待不超过timeout时间
    status = select(io_array, timeout);
    if(status  == OK){
      for(io in io_array){
          io.read() //立即返回，数据都准备好了
      }
    }
}
```

有了IO多路转接，CPU资源利用效率又有了一个提升。

在上面的代码中，线程依然是可能会阻塞在 select 上或者产生一些空转的，有没有一个更加完美的方案呢？

答案就是异步IO了：

```javascript
io.async_read((data) => {
  // dosomething
});
```

解决了Dart单线程进行IO也不会卡的疑问，但主线程如何和大量异步消息打交道呢？接下来我们继续讨论Dart的事件循环机制（Event Loop）。

## 2、事件循环（Event Loop）

Event Loop 完整版的流程图

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7fb48e4227064f48b9aaa57a18ba34af~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

从上图可知，Dart事件循环机制由一个消息循环(event looper)和两个消息队列构成，其中，两个消息队列是指事件队列(event queue)和微任务队列(Microtask queue)。该机制运行原理为：

- 首先，Dart程序从main函数开始运行，待main函数执行完毕后，event looper开始工作；
- 然后，event looper优先遍历执行Microtask队列所有事件，直到Microtask队列为空；
- 接着，event looper才遍历执行Event队列中的所有事件，直到Event队列为空；
- 最后，视情况退出循环。

### 微任务

微任务顾名思义，表示一个短时间内就会完成的异步任务。从上面的流程图可以看到，微任务队列在事件循环中的优先级是最高的，只要队列中还有任务，就可以一直霸占着事件循环。

微任务是由 `scheduleMicroTask` 建立的

```
scheduleMicrotask(() => print('This is a microtask'));
```

不过，一般的异步任务通常也很少必须要在事件队列前完成，所以也不需要太高的优先级，因此我们通常很少会直接用到微任务队列，就连 Flutter 内部，也只有 7 处用到了而已（比如，手势识别、文本输入、滚动视图、保存页面效果等需要高优执行任务的场景）。

### Event Queue

Dart 为 Event Queue 的任务建立提供了一层封装，叫作 Future。从名字上也很容易理解，它表示一个在未来时间才会完成的任务。

## 3、Isolate

尽管 Dart 是基于单线程模型的，但为了进一步利用多核 CPU，将 CPU 密集型运算进行隔离，Dart 也提供了多线程机制，即 Isolate。在 Isolate 中，资源隔离做得非常好，每个 Isolate 都有自己的 Event Loop 与 Queue，Isolate 之间不共享任何资源，只能依靠消息机制通信，因此也就没有资源抢占问题。

假如不同的Isolate需要通信(单向/双向)，就只能通过向对方的事件循环队列里写入任务，并且它们之间的通讯方式是通过port(端口)实现的，其中，Port又分为 `receivePort(接收端口)和sendPort(发送端口)` ，它们是成对出现的。Isolate之间通信过程：

- 首先，当前Isolate创建一个ReceivePort对象，并获得对应的SendPort对象；

```
 var receivePort = ReceivePort();
 var sendPort = receivePort.sendPort;
```

- 其次，创建一个新的Isolate，并实现新Isolate要执行的异步任务，同时，将当前Isolate的SendPort对象传递给新的Isolate，以便新Isolate使用这个SendPort对象向原来的Isolate发送事件；

```csharp
// 调用Isolate.spawn创建一个新的Isolate
// 这是一个异步操作，因此使用await等待执行完毕
var anotherIsolate = await Isolate.spawn(otherIsolateInit, receivePort.sendPort);

// 新Isolate要执行的异步任务
// 即调用当前Isolate的sendPort向其receivePort发送消息
void otherIsolateInit(SendPort sendPort) async {
  value = "Other Thread!";
  sendPort.send("BB");
}
```

- 然后，调用当前Isolate#receivePort的listen方法监听新的Isolate传递过来的数据。Isolate之间什么数据类型都可以传递，不必做任何标记

```lua
receivePort.listen((date) {
    print("Isolate 1 接受消息：data = $date");
});
```

- 最后，消息传递完毕，关闭新创建的Isolate。

```
anotherIsolate?.kill(priority: Isolate.immediate);
anotherIsolate =null;
```

## Future异步详解

## Future的介绍

在写程序的过程中，肯定会有一部分比较耗时代码是需要异步执行的。比如网络操作，我们需要异步去请求数据，并且还需要处理请求成功和请求失败的两种情况。

在Flutter中，使用Future来执行耗时操作，表示在未来会返回某个值，并可以使用then（）方法和catchError（）来注册callback来监听Future的处理结果。

```
Future<Response> respFuture = http.get('https://example.com'); //发起请求
respFuture.then((response) { //成功，匿名函数
  if (response.statusCode == 200) {
    var data = reponse.data;
  }
}).catchError((error) { //失败
   handle(error);
});
```

这种模式简化和统一了异步的处理

Future 对象封装了Dart 的异步操作，它有未完成（uncompleted）和已完成（completed）两种状态。

在Dart中，所有涉及到IO的函数都封装成Future对象返回，在你调用一个异步函数的时候，在结果或者错误返回之前，你得到的是一个uncompleted状态的Future。

**一个Future对象会有以下两种状态**

- pending：表示Future对象的计算过程仍在执行中，这个时候还没有可以用的result。
- completed：表示Future对象已经计算结束了，可能会有两种情况，一种是正确的结果，一种是失败的结果。

## 构造方法

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2f4690d863e04f3a9a5e8a433d1bbfd9~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

我看查看API发现Future总共有6个构造函数

- 1、默认构造方法 `Future(FutureOr<T> computation())`
- 2、 `Future.micortask` 构造方法
- 3、 `Future.sync(FutureOr<T> computation())` 构造方法
- 4、 `Future.value([FutureOr<T>? value])` 构造方法
- 5、 `Future.error(Object error, [StackTrace? stackTrace])` 构造方法
- 6、 `Future.delayed(Duration duration, [FutureOr<T> computation()?])` 构造方法

### 1、默认构造方法

通过Future的默认构造方法可以创建一个Future对象,。默认构造方法的签名如下

```
factory Future(FutureOr<T> computation()) {
    _Future<T> result = new _Future<T>();
    Timer.run(() {
      try {
        result._complete(computation());
      } catch (e, s) {
        _completeWithErrorCallback(result, e, s);
      }
    });
    return result;
  }
```

参数类型是 `FutureOr<T> computation()` ，表示返回值是 `FutureOr<T>` 类型的函数。

通过这个方法创建的 `Future` ， `computation函数` 会被添加到event队列中执行。

### 2、Future.micortask构造方法

```
factory Future.microtask(FutureOr<T> computation()) {
    _Future<T> result = new _Future<T>();
    scheduleMicrotask(() {
      try {
        result._complete(computation());
      } catch (e, s) {
        _completeWithErrorCallback(result, e, s);
      }
    });
    return result;
  }
```

通过 `scheduleMicrotask` 方法将computation函数添加到microtask队列中，优先于event队列执行。

```
    Future(() {
      print("default fauture");
    });

    Future.microtask(() {
      print("microtask future");
    });
```

譬如上面方法会优先打印 `microtask future`

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d8858f9ca0474df88da4f1fb1c1154f1~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

### 3、Future.sync构造方法

```javascript
factory Future.sync(FutureOr<T> computation()) {}
```

将会在当前task执行computation计算，而不是将计算过程添加到任务队列中。

### 4、Future.value()构造方法

创建一个返回指定value值的Future

```javascript
factory Future.value([FutureOr<T>? value]) {
    return new _Future<T>.immediate(value == null ? value as T : value);
```

```
var future = Future.value(1);
var future1 = Future.value('1');
print(future);
print(future1);
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a7ec1200291b43c889b64ff353687c8a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

### 5、Future.error构造方法

```javascript
factory Future.error(Object error, [StackTrace? stackTrace]) {}
```

通过 `error对象` 和可选的stackTrace创建Future,可以使用该方法创建个一个状态为failed的Future对象。

### 6、Future.delayed（）构造方法

```javascript
factory Future.delayed(Duration duration, [FutureOr<T> computation()?]) {
    if (computation == null && !typeAcceptsNull<T>()) {
      throw ArgumentError.value(
          null, "computation", "The type parameter is not nullable");
    }
    _Future<T> result = new _Future<T>();
    new Timer(duration, () {
      if (computation == null) {
        result._complete(null as T);
      } else {
        try {
          result._complete(computation());
        } catch (e, s) {
          _completeWithErrorCallback(result, e, s);
        }
      }
    });
    return result;
  }
```

创建一个延迟执行的future。 例如下面的例子，利用Future延迟两秒后可以打印出字符串。

```
var futureDelayed = Future.delayed(Duration(seconds: 2), () {
  print("Future.delayed");
  return 2;
});
```

## 静态方法

### 1、wait

```csharp
static Future<List<T>> wait<T>(Iterable<Future<T>> futures,
      {bool eagerError = false, void cleanUp(T successValue)?}){}
```

`wait静态方法可以等待多个Future执行完成，并通过List获取所有Future的结果。如果其中一个Future对象发生异常，会导致最终结果为failed`

可选参数

- 1、 `eagerError`:eagerError默认值为false，当某一个Future发生异常时，默认不会立刻使Future处于failed状态，而是等待所有Future都有结果后，改变状态为failed。如果设置为true，当其中一个Future发生异常时，会立刻导致最终结果为failed
- 2、 `cleanUp` ：如果设置了cleanUp参数，当多个Future中的一个发生异常时，其他成功的Future的(非null)结果会传递给cleanUp参数。如果没有发生异常cleanUp函数不会被调用。

### 2、forEach

```csharp
static Future forEach<T>(Iterable<T> elements, FutureOr action(T element)) {}
```

`forEach` 静态方法可以遍历 `Iterable` 中的每个元素执行一个操作，如果遍历操作返回的是Future对象，则在该Future完成后再进行下一次遍历，全部完成后返回null。如果在某一次操作中发生异常，会停止遍历，最终的Future的状态为failed。

比如下面的例子，根据{1,2,3}创建3个延迟对应秒数的Future。执行结果为1秒后打印1，再过2秒打印2，再过3秒打印3，总时间为6秒。

```dart
Future.forEach({1,2,3}, (num){
      return Future.delayed(Duration(seconds: num),(){print(num);});
    });
```

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f2036e937f864ae4b50f4654c2f8bf1a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

### 3、any

```swift
static Future<T> any<T>(Iterable<Future<T>> futures) {}
```

返回的是第一个执行完成的future的结果，不会管这个结果是正确的还是error的

### 4、doWhile

重复性地执行某一个动作，直到返回false或者Future，退出循环

```javascript
static Future doWhile(FutureOr<bool> action()) {}
```

使用场景:适用于一些需要递归操作的场景。

例如下面的例子，生成一个随机数进行等待，直到十秒之后，操作结束。

```dart
void futureDoWhile(){
  var random = new Random();
  var totalDelay = 0;
  Future
      .doWhile(() {
    if (totalDelay > 10) {
      print('total delay: $totalDelay seconds');
      return false;
    }
    var delay = random.nextInt(5) + 1;
    totalDelay += delay;
    return new Future.delayed(new Duration(seconds: delay), () {
      print('waited $delay seconds');
      return true;
    });
  })
      .then(print)
      .catchError(print);
}
//输出结果：
5

1

3

2

12

null
```

## 处理结果

### 1、then

创建完成Future对象后，可以通过then方法接收Future的结果。

```csharp
Future<R> then<R>(FutureOr<R> onValue(T value), {Function onError});
```

```
Future<Response> respFuture = http.get('https://example.com'); //发起请求
respFuture.then((response) { //成功，匿名函数
  if (response.statusCode == 200) {
    var data = reponse.data;
  }
}).catchError((error) { //失败
   handle(error);
});
```

### 2、catchError

如果Future内的函数执行发生异常，可以通过Future.catchError来处理异常：

```
Future<void> fetchUserOrder() {
  return Future.delayed(Duration(seconds: 3),
                                              () => throw Exception('Logout failed: user ID is invalid'));
}
void main() {
  fetchUserOrder().catchError((err, s){print(err);});
  print('Fetching user order...');
}
```

输出结果：

```
Fetching user order...
Exception: Logout failed: user ID is invalid
```

### 3、whenComplete

Future.whenComplete总是在Future完成后调用，不管Future的结果是正确的还是错误的。

```arduino
Future<T> whenComplete(FutureOr<void> action());
```

### 4、timeout方法

```
Future<T> timeout(Duration timeLimit, {FutureOr<T> onTimeout()})
```

timeout方法创建一个新的Future对象，接收一个Duration类型的timeLimit参数来设置超时时间。如果原Future在超时之前完成，最终的结果就是该原Future的值；如果达到超时时间后还未完成，就会产生TimeoutException异常。 该方法有一个onTimeout可选参数，如果设置了该参数，当发生超时时会调用该函数，该函数的返回值为Future的新的值，而不会产生TimeoutException。

## async 和 await

想象一个这样的场景：

- 1. 先调用登录接口；
- 1. 根据登录接口返回的token获取用户信息；
- 1. 最后把用户信息缓存到本机。 接口定义：

```javascript
Future<String> login(String name,String password){
  //登录
}
Future<User> fetchUserInfo(String token){
  //获取用户信息
}
Future saveUserInfo(User user){
  // 缓存用户信息
}
```

用Future大概可以这样写：

```
login('name','password')
.then((token) => fetchUserInfo(token))
 .then((user) => saveUserInfo(user));
```

换成 `async 和await` 则可以这样：

```csharp
void doLogin() async {
  String token = await login('name','password'); //await 必须在 async 函数体内
  User user = await fetchUserInfo(token);
  await saveUserInfo(user);
}
```

声明了 `async` 的函数，返回值是必须是Future对象。即便你在async函数里面直接返回T类型数据，编译器会自动帮你包装成 `Future<T>` 类型的对象，如果是void函数，则返回 `Future<void>` 对象。在遇到 `await` 的时候，又会把Futrue类型拆包，又会原来的数据类型暴露出来，请注意，await所在的函数必须添加async关键词

await的代码发生异常，捕获方式跟同步调用函数一样：

```csharp
void doLogin() async {
  try {
    var token = await login('name','password');
    var user = await fetchUserInfo(token);
    await saveUserInfo(user);
  } catch (err) {
    print('Caught error: $err');
  }
}
```

得益于async 和await 这对语法糖，你可以用同步编程的思维来处理异步编程，大大简化了异步代码的处理

