---
title: Flow 应用场景及原理
url: https://juejin.cn/post/6989032238079803429?searchId=202607171431032AEF48318D9C3D79D7FB
publishedTime: 2021-07-26T08:55:29+08:00
---

什么是“异步数据流”？它在什么业务场景下有用武之地？它背后的原理是什么？读一读 Flow 的源码，尝试回答这些问题。

## 同步 & 异步 & 连续异步

同步和异步是用来形容“调用”的：

- 同步调用：当调用发起者触发了同步调用后，它会等待调用执行完毕并返回结果后才继续执行后续代码。显然只有当调用者和被调用者的代码执行在同一个线程中才会发生这样的串行执行效果。
- 异步调用：当调用发起者触发了异步调用后，它并不会等待异步调用中的代码执行完毕，因为异步调用会立马返回，但并不包含执行结果，执行结果会用异步的方式另行通知调用者。当调用者和被调用者的代码执行在不同线程时就会发生这种并行执行效果。

异步调用在 App 开发中随处可见，通常把耗时操作放到另一个线程执行，比如写文件：

```kotlin
suspend fun writeFile(content: String) {
    // 写文件
}

// 启动协程写文件
val content = "xxx"
coroutineScope.launch { wirteFile(content) }
```

kotlin 中的 `suspend` 方法用于表达一个异步过程，“多个连续产生的异步过程”如何表达？

for 循环是首先想到的方案：

```kotlin
val contents = listOf<String>(...) // 将要写入文件的多个字串
contents.forEach { string ->
    coroutineScope.launch { writeFile(string) }
}
```

用 for 循环的前提条件是得先拿到所有需要进行异步操作的数据。但“多个连续产生的数据”这个场景下，数据是一点一点生成的，没法一下子全部拿到。比如“倒计时 1 分钟，每 2 秒做一次耗时运算，计时结束后将所有运算结果累加并在主线程打印”。这个时候就要用“异步数据流”重新认识问题。

异步数据流用“生产者/消费”模型来解释这个场景：倒计时器是这个场景中的生产者，它每隔两秒产生一个新数据。累加器是这个场景中的消费者，他将所有异步数据累加。生产者和消费者之间就好像有一条管道，生产者从管道的一头插入数据，消费者从另一头取数据。因为管道的存在，数据是有序的，遵循先进先出的原则。

### 传统方案

在给出 Flow 的解决方案之前，先看下传统解决方案。

首先得实现一个定时器，它可以在异步线程中以一定时间间隔执行异步操作。用线程池就再合适不过了：

```kotlin
// 倒计时器
class Countdown<T>(
    private var duration: Long, // 倒计时长
    private var interval: Long, // 倒计时间隔
    private val action: (Long) -> T // 倒计时后台任务
) {
    // 任务结果累加值
    var acc: Any? = null
    // 倒计时剩余时间
    private var remainTime = duration
    // 任务开始回调
    var onStart: (() -> Unit)? = null
    // 任务结束回调
    var onEnd: ((T?) -> Unit)? = null
    // 任务结果累加器
    var accumulator: ((T, T) -> T)? = null
    // 倒计时任务包装类
    private val countdownRunnable by lazy { CountDownRunnable() }
    // 用于主线程回调的 Handler
    private val handler by lazy { Handler(Looper.getMainLooper()) }
    // 线程池
    private val executor by lazy { Executors.newSingleThreadScheduledExecutor() }

    // 启动倒计时
    fun start(delay: Long = 0) {
        if (executor.isShutdown) return
        // 向主线程回调倒计时开始
        handler.post(onStart)
        executor.scheduleAtFixedRate(countdownRunnable, delay, interval, TimeUnit.MILLISECONDS)
    }

    // 将倒计时任务包装成 Runnable
    private inner class CountDownRunnable : Runnable {
        override fun run() {
            remainTime -= interval
            // 执行后台任务并获取返回值
            val value = action(remainTime)
            // 累加任务返回值
            acc = if (acc == null) value else accumulator?.invoke(acc as T, value)
            if (remainTime <= 0) {
                // 关闭倒计时
                executor?.shutdown()
                // 向主线程回调倒计时结束
                handler.post { onEnd?.invoke(acc as? T) }
            }
        }
    }
}
```

抽象出 `Countdown` 用于执行后台倒计时任务，它使用 `scheduleAtFixedRate()` 构造线程池，并按一定间隔执行倒计时任务。

对外倒计时任务被表达成 `(Long) -> T` ，即输入倒计时时间输出异步任务结果的 lambda。在内部它又被包装成一个 Runnable，以便在 run() 方法中实现倒计时及累加逻辑。

然后就可以像这样使用：

```kotlin
Countdown(60_000, 2_000) { remianTime -> calculate(remianTime) }.apply {
    onStart = { Log.v("test", "countdown start") }
    onEnd = { ret -> Log.v("test", "countdown end, ret=$ret") }
    accumulator = { acc, value -> acc + value }
}.start()
```

虽然不得不引入一些复杂度，比如线程池、Handler、累加器。但得益于类的封装和 Kotlin 语法糖，最终调用形式还是简洁达意的。

### Flow 方案

若用 Flow 就可以省去这些复杂度：

```kotlin
fun <T> countdown(
    duration: Long,
    interval: Long,
    onCountdown: suspend (Long) -> T
): Flow<T> =
    flow { (duration - interval downTo 0 step interval).forEach { emit(it) } }
        .onEach { delay(interval) }
        .onStart { emit(duration) }
        .map { onCountdown(it) }
        .flowOn(Dispatchers.Default)
```

定义了一个顶层方法 `countdown()` ，它返回一个流实例用于在异步线程中生产倒计时，并将倒计时传入异步任务 `onCountdown()` 执行。然后就可以像这样使用：

```kotlin
val mainScope = MainScope()
mainScope.launch {
    val ret = countdown(60_000, 2_000) { remianTime -> calculate(remianTime) }
        .onStart { Log.v("test", "countdown start") }
        .onCompletion { Log.v("test", "countdown end") }
        .reduce { acc, value -> acc + value }
    Log.v("test", "coutdown acc ret = $ret")
}
```

下面就从源码出发，一点一点分析流方案背后的原理。

## Flow 如何生产并消费数据？

Flow 的定义及其简单，只包含了 2 个接口：

```kotlin
public interface Flow<out T> {
    public suspend fun collect(collector: FlowCollector<T>)
}
```

Flow 是一个接口，其中定义了一个 `collect()` 方法，表示“流可以被收集”，而收集器也是一个接口：

```kotlin
public interface FlowCollector<in T> {
    public suspend fun emit(value: T)
}
```

流收集器接口中定义了一个 `emit()` 方法表示“流收集器可以发射数据”。

若套用“生产者/消费者”模型，可理解为 **流中数据可以被消费** ， **流收集器可以生产数据** 。

一个最简单的生产和消费数据的场景：

```kotlin
// 启动协程
GlobalScope.launch {
    // 构建流
    flow { // 定义流如何生产数据
3

            // 每隔 1 秒发射 1 个数字
            delay(1000)
            emit(it)
        }
    }.collect { // 定义如何消费数据
        Log.v("test", "num=$it") // 打印数字
    }
}
```

- 通过 `flow{ block }` 构建了一个流，它是一个顶层方法：
  ```kotlin
  // 构建安全流（传入 block 定义如何生产流数据）
  public fun <T> flow(block: suspend FlowCollector<T>.() -> Unit): Flow<T> =
      SafeFlow(block)
      // 安全流继承自抽象流
      private class SafeFlow<T>(private val block: suspend FlowCollector<T>.() -> Unit) : AbstractFlow<T>() {
      override suspend fun collectSafely(collector: FlowCollector<T>) {
          collector.block()// 收集流数据时调用 block，即触发生产数据
      }
  }
  // 抽象流
  public abstract class AbstractFlow<T> : Flow<T>, CancellableFlow<T> {
      // 收集数据的具体实现
      public final override suspend fun collect(collector: FlowCollector<T>) {
          // 构建 FlowCollector 并传入 collectSafely()
          val safeCollector = SafeCollector(collector, coroutineContext)
          try {
              collectSafely(safeCollector)
          } finally {
              safeCollector.releaseIntercepted()
          }
      }
      public abstract suspend fun collectSafely(collector: FlowCollector<T>)
  }
  ```
  `flow { block }` 中的 block 定义了如何生产数据，而 block 是在 `collect()` 中被调用的。所以 **流中的数据不会自动生产，直到流被收集的那一刻。**
- 通过 `collect{ action }` 收集了这个流，其中的 action 定义了如何消费数据。 `collect()` 是 Flow 的扩展方法：
  ```kotlin
  public suspend inline fun <T> Flow<T>.collect(crossinline action: suspend (value: T) -> Unit): Unit =
      collect(object : FlowCollector<T> {
          override suspend fun emit(value: T) = action(value)
      })
  ```
  在收集数据时新建了一个流收集器，流收集器可以发射数据，发射的方式就是直接将数据传递给 action，即数据消费者。

将上述两点综合一下：

> 1. 流中的数据不会自动生产，直到流被收集的那一刻。当流被收集的瞬间，数据开始生产并被发射出去，通过流收集器将其传递给消费者。
> 2. 流和流收集器是成对出现的概念。流是一组按序产生的数据，数据的产生表现为通过流收集器发射数据，在这里流收集器像是流数据容器（虽然它不持有任何一条数据），它定义了如何将数据传递给消费者。

所以上述的实例代码，无异于如下的同步调用：

```kotlin
// 生产者消费者伪代码
flow {
    emit(data) // 生产
}.collect {
    action(data) // 消费
}

// 生产者消费者实际的调用链
Flow.collect {
    emit(data) {
        action(data)
    }
}
```

经过一些 lambda 的抽象，看上去生产者和消费者好像分居两地，但其实它们是运行在同一个线程中的同步调用链，即：

> 默认情况下，流中生产和消费数据是在同一个线程中进行的。

现在回看一下倒计时流是如何生产并消费数据的：

```kotlin
fun <T> countdown(
    duration: Long,
    interval: Long,
    onCountdown: suspend (Long) -> T
): Flow<T> =
    flow { (duration - interval downTo 0 step interval).forEach { emit(it) } }
        .onEach { delay(interval) }
        .onStart { emit(duration) }
        .map { onCountdown(it) }
        .flowOn(Dispatchers.Default)
```

countdown() 方法的第一句就定义了倒计时流中生产数据的方式：

```kotlin
flow { (duration - interval downTo 0 step interval).forEach { emit(it) } }
```

`flow {}` 构建了一个流实例。在内部创建了一个从 duration - interval 到 0 步长为 step 的值序列，它被遍历的同时调用 `emit()` 将每个值发射出去。

创建了流实例后，链式调用了一系列方法，但并没有 `collect()` ，是不是说 countdown() 方法只定义了生产数据并没有定义如何消费数据？

`collect()` 是流数据的消费者，生产者和消费者之间的管道可以插入“中间消费者”，它们优先消费上游数据后再转发给下游。正是这些中间消费者，让流产生了无穷多样的玩法。

## 中间消费者

### transform()

`transform()` 是一个最常见的中间消费者，它是一个 Flow 的扩展方法：

```kotlin
public inline fun <T, R> Flow<T>.transform(
    crossinline transform: suspend FlowCollector<R>.(value: T) -> Unit
): Flow<R> =
    // 构建下游流
    flow {
        // 收集上游数据（这里的逻辑在下游流被收集的时候调用）
        collect { value ->
            // 处理上游数据
            return@collect transform(value)
        }
}
```

transform() 做了三件事情：构建了一个新流（下游流），当下游流被收集时，会立马收集上游的流，当收集到上游数据后将其传递给 `transform` 这个 lambda。

`FlowCollector<R>.(value: T) -> Unit` 是一个带接收者的 lambda，接收者是 `FlowCollector` 。调用这种 labmda 时需要指定接收者，在 transform() 的语境中接收者是 `this` ，所以省略了，如果将其补全，就是下面这样：

```kotlin
public inline fun <T, R> Flow<T>.transform(
    crossinline transform: suspend FlowCollector<R>.(value: T) -> Unit
): Flow<R> =
    // 构建下游流
    flow { this ->
        collect { value ->
            return@collect this.transform(value)
        }
}

// 构建流的 flow {} 中的 lambda 也是带接收者的
public fun <T> flow(block: suspend FlowCollector<T>.() -> Unit): Flow<T> =
    SafeFlow(block)
```

让 `FlowCollector` 作为接收者是有好处的，这样就可以在 lambda 中方便地访问到 `FlowCollector.emit()` ，即 `transform()` 将“下游流如何生产数据”这个策略交由外部传入的 lambda 决定。（关于策略模式的详解可以点击 [一句话总结殊途同归的设计模式：工厂模式=？策略模式=？模版方法模式](https://juejin.cn/post/6844903842232926215 "https://juejin.cn/post/6844903842232926215") ），所以可以得出这样的结论：

> transform() 建立了一种在流上 **拦截并转发** 的机制：新建下游流，它生产数据的方式是通过收集上游数据，并将数据转发到一个带有发射数据能力的 lambda 中。transform() 这个中间消费者在拦截上游数据后，就可随心所欲地将其变换后再转发给下游消费者。

### onEach() & map() & 自定义中间消费者

所以 transform() 通常用于定义新的中间消费者， `onEach()` 的定义就借助于它：

```kotlin
public fun <T> Flow<T>.onEach(action: suspend (T) -> Unit): Flow<T> = transform { value ->
    action(value)
    return@transform emit(value)
}
```

所有的中间消费者都定义成 Flow 的扩展方法，而且都会返回一个新建的下游流。这样做是为了让不同的中间消费者可以方便地通过链式调用串联在一起。

onEach() 通过 transform() 构建了一个下游流，并在转发每一个上游流数据前又做了一件额外的事情，用lambda `action` 表示。

map() 也是通过 transform() 实现的：

```kotlin
public inline fun <T, R> Flow<T>.map(crossinline transform: suspend (value: T) -> R): Flow<R> =
    transform { value -> return@transform emit(transform(value)) }
```

map() 通过 transform() 构建了一个下游流，并且在拿到上游流数据时先将其进行了 `transform` 变换，然后再转发出去。

利用 transform() 的机制，可以很方便地自定义一个中间消费者：

```kotlin
fun <T, R> Flow<T>.filterMap(
    predicate: (T) -> Boolean,
    transform: suspend (T) -> R
): Flow<R> =
    transform { value -> if (predicate(value)) emit(transform(value)) }
```

filterMap() 只对上游数据中满足 predicate 条件的数据进行变换并发射。

### onStart()

onStart() 也是中间消费者，但它没有借助于 transform()，而是通过 `unsafeFlow()` 构建了一个下游流：

```kotlin
public fun <T> Flow<T>.onStart(
    action: suspend FlowCollector<T>.() -> Unit
): Flow<T> = unsafeFlow { // 构建下游流
    val safeCollector = SafeCollector<T>(this, currentCoroutineContext())
    try {
        safeCollector.action() // 在收集上游流数据之前执行动作
    } finally {
        safeCollector.releaseIntercepted()
    }
    collect(this) // 收集上游流数据
}

internal inline fun <T> unsafeFlow(crossinline block: suspend FlowCollector<T>.() -> Unit): Flow<T> {
    // 构建新流
    return object : Flow<T> {
        override suspend fun collect(collector: FlowCollector<T>) {
            collector.block()
        }
    }
}
```

unsafeFlow() 直接实例化了 `Flow` 接口，并定义了该流被收集时执行的操作，即调用 `block` 。所以 unsafeFlow() 和 transform 很类似，都新建下游流以收集了上游数据，只不过在收集动作（所有数据发射之前）之前做了一件额外的事。

### onCompletion()

```kotlin
public fun <T> Flow<T>.onCompletion(
    action: suspend FlowCollector<T>.(cause: Throwable?) -> Unit
): Flow<T> = unsafeFlow { // 构建下游流
    try {
        collect(this) // 1.先收集上游流数据
    } catch (e: Throwable) {
        ThrowingCollector(e).invokeSafely(action, e)
        throw e
    }
    val sc = SafeCollector(this, currentCoroutineContext())
    try {
        sc.action(null) // 2.再执行动作
    } finally {
        sc.releaseIntercepted()
    }
}
```

onCompletion() 的实现和 onStart() 很类似，只不过是在收集数据之后再执行动作。

因为 onStart() 和 onCompletion() 都用下游流套上游流的方式实现，只是收集数据和执行动作的顺序不同，就会产生下面这样有趣的效果：

```kotlin
GlobalScope.launch {
    flow {
3

            delay(1000)
            emit(it)
        }
    }.onStart { Log.v("test","start1") }
        .onStart { Log.v("test","start2") }
        .onCompletion { Log.v("test","complete1") }
        .onCompletion { Log.v("test","complete2") }
        .collect { Log.v("test", "$it") }
}
```

上述代码的输出结果如下：

```kotlin
start2
start1
1
2
3
complete1
complete2
```

链式调用中出现多个 `onStart { action }` 时，后出现的 action 会先执行，因为后续 onStart 构建的下游流包在了上游 onStart 的外面，并且 action 会在收集上游流数据之前执行。

而这个结论却不能沿用到 `onCompletion { action }` ，虽然 onCompletion 构建的下游流也包裹在上游 onCompletion 外面，但是 action 总是在收集上游流之后执行。

## 终端消费者

上面所有的扩展方法之所以称为“中间消费者”是因为它们都构建了一个新的下游流，并且只有当下游流被收集的时候，它们才会去收集上游流。也就是说，如果没有收集下游流，流中的数据就永远不会被发射，这个特性称为 **冷流** 。

看一个冷流的例子：

```kotlin
// 执行式的
suspend fun get(): List<String> =
    listof("a", "b", "c").onEach {
        delay(1000)
        print(it)
    }

// 声明式的
fun get(): Flow<String> =
    flowOf("a", "b", "c").onEach {
        delay(1000)
        print(it)
    }
```

分别调用这两个 get() 方法时，第一个 get 会立马打印出结果，而第二个什么也不会打印。因为第二个 get() 只是声明了如何构建一个冷流，它并没有被收集，所以也不会发射数据。

> Flow 是冷流，冷流不会发射数据，直到它被收集，所以冷流是“声明式的”。
>
> 所有能触发收集数据动作的消费者称为 **终端消费者** ，它就像点燃鞭炮的星火，使得被若干个中间消费者套娃了的流从外向内（从下游到上游）一个个的被收集，最终传导到原始流，触发数据的发射。

倒计时 demo 的 `reduce()` 就是一个终端消费者：

```kotlin
val mainScope = MainScope()
mainScope.launch {
    val ret = countdown(60_000, 2_000) { io(it) }
        .onStart { Log.v("test", "countdown start") }
        .onCompletion { Log.v("test", "countdown end") }
        .reduce { acc, value -> acc + value } // 终端消费者：计算所有异步结果的和
    // 因为 reduce() 是一个 suspend 方法，所以会挂起协程，直到倒计时完成才打印所有异步结果的和
    Log.v("test", "coutdown acc ret = $ret")
}
```

reduce() 的源码如下：

```kotlin
public suspend fun <S, T : S> Flow<T>.reduce(
    operation: suspend (accumulator: S, value: T) -> S // 累加算法
): S {
    var accumulator: Any? = NULL
    // 收集数据
    collect { value ->
        // 将收集的数据累加
        accumulator = if (accumulator !== NULL) {
            operation(accumulator as S, value)
        } else {
            value
        }
    }

    if (accumulator === NULL) throw NoSuchElementException("Empty flow can't be reduced")
    // 返回累加和
    return accumulator as S
}
```

reduce() 并没有构建新流，而是直接收集了数据，然后将所有数据进行累加并返回。

所有的终端消费者都是 suspend 方法，这意味着收集数据必须在协程中进行。demo 中使用 MainScope 启动协程，所以异步结果的和会在主线程中被打印。

## 线程切换

demo 中还剩下最后一个 `flowOn()` ，它是中间消费者，略复杂，限于篇幅原因，下次再分析。但这不影响先了解它的效果：它会切换所有上游代码执行的线程，但不改变下游代码执行的线程。

countdown() 方法通过 `flowOn(Dispatchers.Default)` ，实现了后台执行倒计时任务。而 reduce() 的调用发生在 flowOn() 之后，所以异步任务结果累加还是在主线程进行的。

`onStart()` 、 `onEach()` 、 `onCompletion()` 、 `map()` 、 `reduce()` ，这些消费者对数据的处理都被包装在用 `suspend` 修饰的 lambda 中。这意味着利用协程可以轻松地切换每个消费者运行的线程。也正是 `suspend` 的存在，运行在同一线程中的下游消费者不会发生背压，因为下游消费者的挂起方法会天然阻塞上游生产数据的速度。

## 总结

1. 异步数据流可以理解为一条时间轴上按序产生的数据，它可用于表达 **多个连续的异步过程** 。
2. 异步数据流也可以用“生产者/消费者”模型来理解，生产者和消费者之间就好像有一条管道，生产者从管道的一头插入数据，消费者从另一头取数据。因为管道的存在，数据是有序的，遵循先进先出的原则。
3. Kotlin 中的 `suspend` 方法用于表达一个异步过程，而 `Flow` 用于表达多连续个异步过程。 `Flow` 是冷流，冷流不会发射数据，直到它被收集的那一刻，所以冷流是“声明式的”。
4. 当 `Flow` 被收集的瞬间，数据开始生产并被发射出去，通过流收集器 `FlowCollector` 将其传递给消费者。流和流收集器是成对出现的概念。流是一组按序产生的数据，数据的产生表现为通过流收集器发射数据，在这里流收集器像是流数据容器（虽然它不持有任何一条数据），它定义了如何将数据传递给消费者。
5. 异步数据流中，生产者和消费者之间可以插入 **中间消费者** 。中间消费者建立了流上的 **拦截并转发** 机制：新建下游流，它生产数据的方式是通过收集上游数据，并转发到一个带有发射数据能力的 lambda 中。拥有多个中间消费者的流就像“套娃”一样，下游流套在上游流外面。中间消费者通过这种方式拦截了原始数据，就可以对其做任意变换再转发给下游消费者。因为 Flow 是冷流，所有的中间消费者只是定义了一连串待执行的调用链。
6. 所有能触发收集数据动作的消费者称为 **终端消费者** ，它就像点燃鞭炮的星火，使得被若干个中间消费者套娃的流从外向内（从下游到上游）一个个的被收集，最终传导到原始流，触发数据的发射。
7. 默认情况下，流中生产和消费数据是在同一个线程中进行的。但可以通过 `flowOn()` 改变上游流执行的线程，这并不影响下游流所执行的线程。
8. `Flow` 中生产和消费数据的操作都被包装在用 suspend 修饰的 lambda 中，用协程就可以轻松的实现异步生产，异步消费。

## 推荐阅读

- [Kotlin 基础 | 委托及其应用](https://juejin.cn/post/6947830195034259469 "https://juejin.cn/post/6947830195034259469")
- [Kotlin 基础 | 拒绝语法噪音](https://juejin.cn/post/6963054983067467790 "https://juejin.cn/post/6963054983067467790")
- [Kotlin 进阶 | 不变型、协变、逆变](https://juejin.cn/post/6921508377998671886 "https://juejin.cn/post/6921508377998671886")
- [Kotlin 实战 | 时隔一年，用 Kotlin 重构一个自定义控件](https://juejin.cn/post/6855129007798894599 "https://juejin.cn/post/6855129007798894599")
- [Kotlin 实战 | 用语法糖干掉形状 xml 文件](https://juejin.cn/post/6872136025624444941 "https://juejin.cn/post/6872136025624444941")
- [Kotlin 基础 | 望文生义的 Kotlin 集合操作](https://juejin.cn/post/6970853612352339981 "https://juejin.cn/post/6970853612352339981")
- [Kotlin 源码 | 降低代码复杂度的法宝](https://juejin.cn/post/6976033228771721247 "https://juejin.cn/post/6976033228771721247")
- [Kotlin 协程 | CoroutineContext 为什么要设计成 indexed set？（一）](https://juejin.cn/post/6978613779252641799 "https://juejin.cn/post/6978613779252641799")
- [Kotlin 进阶 | 异步数据流 Flow 的使用场景](https://juejin.cn/post/6989032238079803429 "https://juejin.cn/post/6989032238079803429")
- [Kotlin 异步 | Flow 限流的应用场景及原理](https://juejin.cn/post/6989782281191686180 "https://juejin.cn/post/6989782281191686180")

