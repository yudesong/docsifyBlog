## Android-Stability【Fdleak】: Android Fd泄漏问题分析

[![](https://cdn2.jianshu.io/assets/default_avatar/5-33d2da32c552b8be9a0548c7a4576607.jpg)](https://www.jianshu.com/u/4311af2fe8b5)

[马小藤](https://www.jianshu.com/u/4311af2fe8b5)IP属地: 湖北

0.9162018.11.06 21:03:09字数 3,852阅读 13,835

## Android-Stability【Fdleak】: Android Fd泄漏问题分析

## 1\. Fd leak问题概述

### 1.1 fd limit

如图可以查看某个进成的fd信息：

![](https://upload-images.jianshu.io/upload_images/807220-0dacd6c4b74ca1e7?imageMogr2/auto-orient/strip|imageView2/2/w/874/format/webp)

在这里插入图片描述

系统对进程使用系统资源有相关限制cat /proc/pid/limits 即可看到：

![](https://upload-images.jianshu.io/upload_images/807220-8544fb4994bb4d96?imageMogr2/auto-orient/strip|imageView2/2/w/673/format/webp)

在这里插入图片描述

soft Limit 软限制 ： 系统资源的使用的上限值  
hard Limit 硬限制 ：但不能超出hard limits值

如上看到系统对3207进程的的资源限制情况，其中对于FD资源限制为max open file一行，上限制是1024，即使用的fd如果已满1024了，就会没有fd资源提供，再需要fd的时候就会出现异常，即fd泄漏问题。

### 1.2 更改limit方法

进程可以通过如下方法获取fd限制信息，也可以通过setrlimit方法对本进程fd限制进行调整，但最大不能超过hardlimit限制：  
**a. Native 方法：**  
getrlimit(RLIMIT_NOFILE, &rlim);  
setrlimit(RLIMIT_NOFILE, &rlim);  
**b. Java 方法：**  
android.system.Os.getrlimit(OsConstants.RLIMIT_NOFILE);  
java方法未实现setrlimit方法，如果是手机厂商的话可以尝试添加一个setrlimit方法的java接口。  
**c. 终端：**  
这里要注意的是只有security组的成员可使更改rlimit永久生效，普通用户的更改的进程退出侯失效，重新进入该进程后将恢复使用默认的1024.

### 1.3 Fd泄漏问题

由上可以看出，fd资源不足时，再需要fd的线程可能就会产生异常，而需要fd资源的场景很多，即在fd资源不足时，这些需要fd资源的很多场景，代码都有可能出现异常从而进程crash或功能异常，因此这些场景可能只是躺枪导致进程FC，真正罪魁祸首应该是大量占用fd资源的地方。  
因此，Fd泄漏的原因一般都有如下特点：  
1\. 同一个问题可能出现不同堆栈  
2\. 比较隐晦

## 2\. 需要open Fd的场景

同一个问题可能有不同的堆栈，但都一般都是需要fd时的堆栈，因此，如果遇到类似需要fd的堆栈都可以怀疑是否是发生了fd泄漏了，当然日志中伴随者大量“Too many open files”的字眼。  
如下Java层的Error Msg均有fd泄漏的嫌疑：  
"Too many open files"  
"Could not allocate JNI Env"  
"Could not allocate dup blob fd"  
"Could not read input channel file descriptors from parcel"  
"pthread_create \* "  
"InputChannel is not initialized"  
"Could not open input channel pair"  
...  
当然还有许多其他类型的Error Msg，后续补充。  
大致看下比较常见需要fd的场景：

### 2.1 Resource相关

![](https://upload-images.jianshu.io/upload_images/807220-642e2a829ac766f6.png)

image.png

使用BufferedReader去读取文件内容是常见的方式，bufferedReadernew出来需要及时关闭，注意如上br.close在try里面，如过close之前发生异常未走到close则这里便有fd泄漏的风险，因此br.close应当在finally块中操作，保证一定能close掉。  
详细流程（虽然基于M的code来看，但基本如下最终调用系统函数open，会生成一个int的fd值）：

在这里插入图片描述

简化过程：

![](https://upload-images.jianshu.io/upload_images/807220-7084a98fcea3db26.png)

image.png

### 2.2 HandlerThread

HandlerThread是自带looper的thread，使用时可以使用该looper构造handler来使用handler的功能，一个HandlerThread对应一个Looper成员变量

![](https://upload-images.jianshu.io/upload_images/807220-a93403f40f03f6ce.png)

image.png

而Looper对象初始化时Looper.prepare() 需要fd资源，而且是一个HandlerThread起来会消耗一对fd(eventFd和epollFd)，这两个fd的目的也很明确，就是用来实现线程间通信的。

详细过程如下：

![](https://upload-images.jianshu.io/upload_images/807220-aa35b00d2fce7dbc.png)

image.png

简化过成如下：

![](https://upload-images.jianshu.io/upload_images/807220-b11a958450744897.png)

image.png

如过是HandlerThread异常导致的fd泄漏，我们在该进程的Fd信息中，下面是解问提时遇到的systemui的fd泄漏案例：  
HandlerThread开启过多时的fd信息，ls -la proc/{pid}/fd/ 获取：

```jsx
fd 775: anon_inode:[eventfd]
fd 776: anon_inode:[eventpoll]
fd 777: anon_inode:[eventpoll]
fd 778: anon_inode:[eventfd]
fd 779: anon_inode:[eventfd]
fd 780: anon_inode:[eventpoll]
fd 781: anon_inode:[eventpoll]
fd 782: anon_inode:[eventpoll]
...
fd 808: anon_inode:[eventpoll]
fd 809: anon_inode:[eventfd]
fd 810: /dev/ashmem
fd 811: anon_inode:[eventpoll]
fd 812: anon_inode:[eventfd]
fd 813: anon_inode:dmabuf
fd 815: anon_inode:[eventfd]
fd 816: anon_inode:[eventpoll]
fd 817: anon_inode:sync_file
...
fd 832: anon_inode:[eventpoll]
fd 833: anon_inode:[eventfd]
fd 834: anon_inode:[eventpoll]
fd 835: anon_inode:[eventpoll]
fd 836: anon_inode:[eventfd]
fd 837: anon_inode:[eventpoll]
fd 838: /dev/ashmem
fd 839: anon_inode:[eventpoll]
fd 840: anon_inode:[eventfd]
fd 841: anon_inode:[eventfd]
...
fd 860: anon_inode:[eventpoll]
fd 861: anon_inode:[eventfd]
fd 862: anon_inode:[eventfd]
fd 863: anon_inode:[eventpoll]
fd 864: anon_inode:[eventfd]
fd 865: anon_inode:[eventfd]
fd 866: anon_inode:[eventpoll]
fd 867: anon_inode:[eventpoll]
fd 868: anon_inode:[eventfd]
fd 869: anon_inode:[eventpoll]
```

可以看到打开的eventfd，eventpoll类型问提异常的多。  
再看下该进程的进程trace或ps信息中的线程情况：

```css
pid: 11019, tid: 7441, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 7532, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 7534, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 7632, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 7806, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 7856, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 8116, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 8304, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 8388, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 8467, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 8565, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 8697, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 8760, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 8853, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 8940, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 9107, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 9215, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 9311, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 9331, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 9596, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 9930, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 10036, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 10069, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 10127, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 10231, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 10331, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 10449, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 10568, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 10674, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 10772, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 10822, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 10920, name: async_sensor >>> com.android.systemui <<<
pid: 11019, tid: 11113, name: async_sensor >>> com.android.systemui <<<
```

可以看到async_sensor这个线程异常的多，很明显是该线程异常起的过多导致了fd泄漏，可以继续排查该线程的异常起这么多的原因，从而解决该fd泄漏问提。

### 2.3 Java Thread Start

Java在起线程的时候也会需要开Fd资源，如果线程生命周期操作不当，当起的线程过多时也会导致Fd泄漏的问提，当然这里也是会经常躺枪。  
可以先阅读该篇文章进行详细的学习：[https://www.jianshu.com/p/e574f0ffdb42](https://www.jianshu.com/p/e574f0ffdb42)  
详细过成：

![](https://upload-images.jianshu.io/upload_images/807220-5577654f0db8c1bb.png)

image.png

简化过成：

在这里插入图片描述

**对于1的地方**，如果Fd资源不足了，会报出类似如下调用栈错误，应该是已经有Fd泄漏了导致再起线程时这里创建JNIENV需要打开Fd失败，当然这里有可能只是躺枪：

```bash
   java.lang.OutOfMemoryError: Could not allocate JNI Env
   at java.lang.Thread.nativeCreate(Native Method)
   at java.lang.Thread.start(Thread.java:729)
   at com.android.server.wifi.WifiNative.startHal(WifiNative.java:1639)
   at com.android.server.wifi.WifiStateMachine.setupDriverForSoftAp(WifiStateMachine.java:3970)
   at com.android.server.wifi.WifiStateMachine.-wrap9(WifiStateMachine.java)
   at com.android.server.wifi.WifiStateMachine$InitialState.processMessage(WifiStateMachine.java:4480)
   at com.android.internal.util.StateMachine$SmHandler.processMsg(StateMachine.java:980)
   at com.android.internal.util.StateMachine$SmHandler.handleMessage(StateMachine.java:799)
   at android.os.Handler.dispatchMessage(Handler.java:102)
   at android.os.Looper.loop(Looper.java:163)
   at android.os.HandlerThread.run(HandlerThread.java:61)
```

**对于2的地方**，如果内存资源不足了，或者没有连续虚拟地址可用会抛出如下类似错误调用栈，这个时候应该去调查为何虚拟地址不足，这也是OOM的问提了：

```bash
java.lang.OutOfMemoryError: pthread_create (1040KB stack) failed: Try again
at java.lang.Thread.nativeCreate(Native Method)
at java.lang.Thread.start(Thread.java:733)
at com.tencent.mm.sdk.f.b$a.start(SourceFile:61)
at com.tencent.mm.am.a.bU(SourceFile:60)
at com.tencent.mm.ui.MMAppMgr$8.tC(SourceFile:315)
at com.tencent.mm.sdk.platformtools.am.handleMessage(SourceFile:69)
at com.tencent.mm.sdk.platformtools.aj.handleMessage(SourceFile:173)
at com.tencent.mm.sdk.platformtools.aj.dispatchMessage(SourceFile:128)
at android.os.Looper.loop(Looper.java:176)
at android.app.ActivityThread.main(ActivityThread.java:6701)
at java.lang.reflect.Method.invoke(Native Method)
at com.android.internal.os.Zygote$MethodAndArgsCaller.run(Zygote.java:246)
at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:783)
```

复现该类问提之需要在fd不足时不断起java线程，或内存紧张时起线程就比较容易复现类似的调用栈。  
所以该类调用栈容易是躺枪，fd leak问提需要调查谁消耗的fd过多，oom问提需要找到是谁导致内存不足。

### 2.4 InputChannel

fd leak中一种比较常见的错误调用桟：  
log1：

```bash
06-22 20:34:43.035 10037 20556 20556 E AndroidRuntime: FATAL EXCEPTION: main
06-22 20:34:43.035 10037 20556 20556 E AndroidRuntime: Process: com.miui.weather2, PID: 20556
06-22 20:34:43.035 10037 20556 20556 E AndroidRuntime: java.lang.RuntimeException: Could not read input channel file descriptors from parcel.
06-22 20:34:43.035 10037 20556 20556 E AndroidRuntime:     at android.view.InputChannel.nativeReadFromParcel(Native Method)
06-22 20:34:43.035 10037 20556 20556 E AndroidRuntime:     at android.view.InputChannel.readFromParcel(InputChannel.java:148)
06-22 20:34:43.035 10037 20556 20556 E AndroidRuntime:     at android.view.InputChannel$1.createFromParcel(InputChannel.java:39)
06-22 20:34:43.035 10037 20556 20556 E AndroidRuntime:     at android.view.InputChannel$1.createFromParcel(InputChannel.java:37)
06-22 20:34:43.035 10037 20556 20556 E AndroidRuntime:     at com.android.internal.view.InputBindResult.<init>(InputBindResult.java:68)
06-22 20:34:43.035 10037 20556 20556 E AndroidRuntime:     at com.android.internal.view.InputBindResult$1.createFromParcel(InputBindResult.java:112)
06-22 20:34:43.035 10037 20556 20556 E AndroidRuntime:     at com.android.internal.view.InputBindResult$1.createFromParcel(InputBindResult.java:110)
06-22 20:34:43.035 10037 20556 20556 E AndroidRuntime:     at com.android.internal.view.IInputMethodManager$Stub$Proxy.startInputOrWindowGainedFocus(IInputMethodManager.java:723)
06-22 20:34:43.035 10037 20556 20556 E AndroidRuntime:     at android.view.inputmethod.InputMethodManager.startInputInner(InputMethodManager.java:1295)
06-22 20:34:43.035 10037 20556 20556 E AndroidRuntime:     at android.view.inputmethod.InputMethodManager.onPostWindowFocus(InputMethodManager.java:1543)
06-22 20:34:43.035 10037 20556 20556 E AndroidRuntime:     at android.view.ViewRootImpl$ViewRootHandler.handleMessage(ViewRootImpl.java:4069)
06-22 20:34:43.035 10037 20556 20556 E AndroidRuntime:     at android.os.Handler.dispatchMessage(Handler.java:106)
06-22 20:34:43.035 10037 20556 20556 E AndroidRuntime:     at android.os.Looper.loop(Looper.java:171)
06-22 20:34:43.035 10037 20556 20556 E AndroidRuntime:     at android.app.ActivityThread.main(ActivityThread.java:6642)
06-22 20:34:43.035 10037 20556 20556 E AndroidRuntime:     at java.lang.reflect.Method.invoke(Native Method)
06-22 20:34:43.035 10037 20556 20556 E AndroidRuntime:     at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:518)
06-22 20:34:43.035 10037 20556 20556 E AndroidRuntime:     at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:873)
```

log2：

```php
07-22 11:06:05.644  1526  1645 E AndroidRuntime: *** FATAL EXCEPTION IN SYSTEM PROCESS: android.ui
07-22 11:06:05.644  1526  1645 E AndroidRuntime: java.lang.RuntimeException: Could not open input channel pair.  status=-24
07-22 11:06:05.644  1526  1645 E AndroidRuntime:   at android.view.InputChannel.nativeOpenInputChannelPair(Native Method)
07-22 11:06:05.644  1526  1645 E AndroidRuntime:   at android.view.InputChannel.openInputChannelPair(InputChannel.java:94)
07-22 11:06:05.644  1526  1645 E AndroidRuntime:   at com.android.server.wm.WindowState.openInputChannel(WindowState.java:2011)
07-22 11:06:05.644  1526  1645 E AndroidRuntime:   at com.android.server.wm.WindowManagerService.addWindow(WindowManagerService.java:1406)
07-22 11:06:05.644  1526  1645 E AndroidRuntime:   at com.android.server.wm.Session.addToDisplay(Session.java:197)
07-22 11:06:05.644  1526  1645 E AndroidRuntime:   at android.view.ViewRootImpl.setView(ViewRootImpl.java:750)
07-22 11:06:05.644  1526  1645 E AndroidRuntime:   at android.view.WindowManagerGlobal.addView(WindowManagerGlobal.java:356)
07-22 11:06:05.644  1526  1645 E AndroidRuntime:   at android.view.WindowManagerImpl.addView(WindowManagerImpl.java:93)
07-22 11:06:05.644  1526  1645 E AndroidRuntime:   at android.app.Dialog.show(Dialog.java:330)
07-22 11:06:05.644  1526  1645 E AndroidRuntime:   at com.android.server.am.AppErrors.handleShowAppErrorUi(AppErrors.java:755)
07-22 11:06:05.644  1526  1645 E AndroidRuntime:   at com.android.server.am.ActivityManagerService$UiHandler.handleMessage(ActivityManagerService.java:1849)
07-22 11:06:05.644  1526  1645 E AndroidRuntime:   at android.os.Handler.dispatchMessage(Handler.java:105)
07-22 11:06:05.644  1526  1645 E AndroidRuntime:   at android.os.Looper.loop(Looper.java:171)
07-22 11:06:05.644  1526  1645 E AndroidRuntime:   at android.os.HandlerThread.run(HandlerThread.java:65)
07-22 11:06:05.644  1526  1645 E AndroidRuntime:   at com.android.server.ServiceThread.run(ServiceThread.java:46)
07-22 11:06:05.644  1526  1645 E AndroidRuntime:   at com.android.server.UiThread.run(UiThread.java:42)
```

当然类似inputchannel的异常调用栈不止如上两个，还有  
这里inputchannel也是需要fd资源，AddWindow的时候需要初始化inputchannel去和InputManagerService进行跨进程通信来监控Input事件，本质上是初始化了一对socket文件进行通信：  
详细过程：

![](https://upload-images.jianshu.io/upload_images/807220-afc0a406ae906aff.png)

image.png

可以通过该博客进行学习InpuntChannnel相关的知识：[https://blog.csdn.net/hongzg1982/article/details/54812359](https://blog.csdn.net/hongzg1982/article/details/54812359)

简化过程：

![](https://upload-images.jianshu.io/upload_images/807220-49d5b82099a7ccdc.png)

image.png

如上知道addWindow或创建出一对socket文件用于构造inputchannel，从而实现app从InputDispatcher那里得到input事件，因此，如果此时Fd资源紧张，则很容易在addWindow时发生异常。  
不过如过异常log显示inputchannel相关的居多的话也有必要此时window存在过多，或异常添加window为销毁导致socket增加而fd泄漏。  
因此对于该类的异常，可以看看当前的window信息，adb shell dumpsys window或得，即可以比较明确的看出是哪个window有异常，然后找原因解决。

**类似的写个该类型demo试下**：  
写一个demo，点击按钮让其不断的弹AlertDialog：  
结果出现异常：

```bash
10-28 20:49:07.547 14246 14246 E AndroidRuntime: FATAL EXCEPTION: main
10-28 20:49:07.547 14246 14246 E AndroidRuntime: Process: com.example.chengang.fdtest, PID: 14246
10-28 20:49:07.547 14246 14246 E AndroidRuntime: java.lang.RuntimeException: Could not read input channel file descriptors from parcel.
10-28 20:49:07.547 14246 14246 E AndroidRuntime: at android.view.InputChannel.nativeReadFromParcel(Native Method)
10-28 20:49:07.547 14246 14246 E AndroidRuntime: at android.view.InputChannel.readFromParcel(InputChannel.java:148)
10-28 20:49:07.547 14246 14246 E AndroidRuntime: at android.view.IWindowSession$Stub$Proxy.addToDisplay(IWindowSession.java:759)
10-28 20:49:07.547 14246 14246 E AndroidRuntime: at android.view.ViewRootImpl.setView(ViewRootImpl.java:669)
10-28 20:49:07.547 14246 14246 E AndroidRuntime: at android.view.WindowManagerGlobal.addView(WindowManagerGlobal.java:319)
10-28 20:49:07.547 14246 14246 E AndroidRuntime: at android.view.WindowManagerImpl.addView(WindowManagerImpl.java:85)
10-28 20:49:07.547 14246 14246 E AndroidRuntime: at android.app.Dialog.show(Dialog.java:325)
10-28 20:49:07.547 14246 14246 E AndroidRuntime: at android.app.AlertDialog$Builder.show(AlertDialog.java:1112)
10-28 20:49:07.547 14246 14246 E AndroidRuntime: at com.example.chengang.fdtest.MainActivity.showNormalDialog(MainActivity.java:70)
10-28 20:49:07.547 14246 14246 E AndroidRuntime: at com.example.chengang.fdtest.MainActivity.access$000(MainActivity.java:13)
10-28 20:49:07.547 14246 14246 E AndroidRuntime: at com.example.chengang.fdtest.MainActivity$1.onClick(MainActivity.java:31)
10-28 20:49:07.547 14246 14246 E AndroidRuntime: at android.view.View.performClick(View.java:5275)
10-28 20:49:07.547 14246 14246 E AndroidRuntime: at android.view.View$PerformClick.run(View.java:21559)
10-28 20:49:07.547 14246 14246 E AndroidRuntime: at android.os.Handler.handleCallback(Handler.java:815)
10-28 20:49:07.547 14246 14246 E AndroidRuntime: at android.os.Handler.dispatchMessage(Handler.java:104)
10-28 20:49:07.547 14246 14246 E AndroidRuntime: at android.os.Looper.loop(Looper.java:207)
10-28 20:49:07.547 14246 14246 E AndroidRuntime: at android.app.ActivityThread.main(ActivityThread.java:5845)
10-28 20:49:07.547 14246 14246 E AndroidRuntime: at java.lang.reflect.Method.invoke(Native Method)
10-28 20:49:07.547 14246 14246 E AndroidRuntime: at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:907)
10-28 20:49:07.547 14246 14246 E AndroidRuntime: at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:768)


10-28 20:49:07.849 9259 9286 E AndroidRuntime: *** FATAL EXCEPTION IN SYSTEM PROCESS: android.ui
10-28 20:49:07.849 9259 9286 E AndroidRuntime: java.lang.RuntimeException: Failed to initialize display event receiver. status=-2147483648
10-28 20:49:07.849 9259 9286 E AndroidRuntime: at android.view.DisplayEventReceiver.nativeInit(Native Method)
10-28 20:49:07.849 9259 9286 E AndroidRuntime: at android.view.DisplayEventReceiver.<init>(DisplayEventReceiver.java:71)
10-28 20:49:07.849 9259 9286 E AndroidRuntime: at android.view.Choreographer$FrameDisplayEventReceiver.<init>(Choreographer.java:824)
10-28 20:49:07.849 9259 9286 E AndroidRuntime: at android.view.Choreographer.<init>(Choreographer.java:219)
10-28 20:49:07.849 9259 9286 E AndroidRuntime: at android.view.Choreographer.<init>(Choreographer.java:79)
10-28 20:49:07.849 9259 9286 E AndroidRuntime: at android.view.Choreographer$1.initialValue(Choreographer.java:116)
10-28 20:49:07.849 9259 9286 E AndroidRuntime: at android.view.Choreographer$1.initialValue(Choreographer.java:109)
10-28 20:49:07.849 9259 9286 E AndroidRuntime: at java.lang.ThreadLocal$Values.getAfterMiss(ThreadLocal.java:430)
10-28 20:49:07.849 9259 9286 E AndroidRuntime: at java.lang.ThreadLocal.get(ThreadLocal.java:65)
10-28 20:49:07.849 9259 9286 E AndroidRuntime: at android.view.Choreographer.getInstance(Choreographer.java:249)
10-28 20:49:07.849 9259 9286 E AndroidRuntime: at android.view.ViewRootImpl.<init>(ViewRootImpl.java:501)
10-28 20:49:07.849 9259 9286 E AndroidRuntime: at android.view.WindowManagerGlobal.addView(WindowManagerGlobal.java:306)
10-28 20:49:07.849 9259 9286 E AndroidRuntime: at android.view.WindowManagerImpl.addView(WindowManagerImpl.java:85)
10-28 20:49:07.849 9259 9286 E AndroidRuntime: at android.app.Dialog.show(Dialog.java:325)
10-28 20:49:07.849 9259 9286 E AndroidRuntime: at com.android.server.am.ActivityManagerService$UiHandler.handleMessage(ActivityManagerService.java:1694)
10-28 20:49:07.849 9259 9286 E AndroidRuntime: at android.os.Handler.dispatchMessage(Handler.java:111)
10-28 20:49:07.849 9259 9286 E AndroidRuntime: at android.os.Looper.loop(Looper.java:207)
10-28 20:49:07.849 9259 9286 E AndroidRuntime: at android.os.HandlerThread.run(HandlerThread.java:61)
10-28 20:49:07.849 9259 9286 E AndroidRuntime: at com.android.server.ServiceThread.run(ServiceThread.java:46)
```

**看到不仅demo app crash了，而且system_server也出现了异常crash，手机重启了，可怕**！！！  
足见fd泄漏问题的严重性，也了解到app异常也会影响到system_server的稳定性。

为了不让系统重启，简单的看log，下面日日志是只弹50个dialog时抓取到的信息（其实也可以复写UnCatchExceptionHandler,在异常退出前抓取所需的fd，hprof，dumpsys window信息）：  
**看下该异常時的fd信息：**

```rust
socket 文件类型变多：
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 100 -> socket:[275165]
...
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 112 -> socket:[275177]
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 113 -> socket:[275179]
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 114 -> socket:[275181]
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 115 -> socket:[275183]
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 13 -> socket:[268102]
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 20 -> socket:[244415]
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 38 -> socket:[264306]
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 4 -> socket:[159952]
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 44 -> socket:[258994]
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 46 -> socket:[262529]
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 49 -> socket:[249721]
...
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 86 -> socket:[275153]
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 87 -> socket:[275155]
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 88 -> socket:[275157]
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 89 -> socket:[270815]
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 90 -> socket:[266061]
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 91 -> socket:[270817]
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 92 -> socket:[275159]
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 93 -> socket:[275161]
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 94 -> socket:[266063]
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 95 -> socket:[270819]
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 96 -> socket:[272449]
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 97 -> socket:[266065]
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 98 -> socket:[272451]
lrwx------ u0_a158 u0_a158 2018-10-28 21:17 99 -> socket...

ashmem类型也变多，可能跟surface创建surfaceflinger需要通过ashmem类型文件传递显示数据导致（暂未证实，后续调查）
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 520 -> /dev/ashmem
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 521 -> /dev/ashmem
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 522 -> anon_inode:dmabuf
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 523 -> /dev/ashmem
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 525 -> anon_inode:dmabuf
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 526 -> /dev/ashmem
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 527 -> /dev/ashmem
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 528 -> anon_inode:dmabuf
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 529 -> /dev/ashmem
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 53 -> /dev/ion
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 530 -> anon_inode:dmabuf
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 531 -> /dev/ashmem
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 532 -> anon_inode:dmabuf
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 533 -> anon_inode:dmabuf
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 534 -> /dev/ashmem
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 535 -> /dev/ashmem
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 536 -> anon_inode:dmabuf
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 537 -> /dev/ashmem
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 538 -> anon_inode:dmabuf
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 539 -> anon_inode:dmabuf
lr-x------ u0_a158 u0_a158 2018-10-28 21:04 54 -> /proc/ged
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 540 -> /dev/ashmem
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 541 -> anon_inode:dmabuf
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 542 -> anon_inode:dmabuf
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 543 -> /dev/ashmem
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 544 -> anon_inode:dmabuf
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 545 -> anon_inode:dmabuf
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 546 -> /dev/ashmem
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 547 -> /dev/ashmem
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 548 -> /dev/ashmem
lrwx------ u0_a158 u0_a158 2018-10-28 21:04 549 -> /dev/ashmem
```

可明显看到socket类型的文件变多，正是inputchannel增多导致，另外值得注意的是ashmem，dmabuf也明显增多（后续调查原因）;  
**看下该case的hprof对比：**  
弹dialog之前：  
\[图片上传失败...(image-a2582c-1541509372236)\]

弹50个dialog之后：

![](https://upload-images.jianshu.io/upload_images/807220-1e9d56082fd48849.png)

image.png

明显看出inputchannel增多。  
**看下该case下window的情况：**

```bash
dumpsys window:
WINDOW MANAGER TOKENS (dumpsys window tokens)
WINDOW MANAGER WINDOWS (dumpsys window windows)
Window #56 Window{db0f6a6 u0 AOD}:
Window #55 Window{a450d9c u0 StatusBar}:
Window #54 Window{5261571 u0 KeyguardScrim}:
Window #53 Window{7eadf3a u0 AssistPreviewPanel}:
Window #52 Window{b58497e u0 DockedStackDivider}:
Window #51 Window{3aab4ea u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #50 Window{de9258c u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #49 Window{bce8de u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #48 Window{5843360 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #47 Window{7a80592 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #46 Window{6744bf4 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #45 Window{feaff06 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #44 Window{f5a4348 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #43 Window{b3d893a u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #42 Window{b1ad5c u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #41 Window{f84182e u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #40 Window{a4de30 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #39 Window{983dfe2 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #38 Window{9a0e9c4 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #37 Window{c56d456 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #36 Window{7a9a418 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #35 Window{19fa98a u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #34 Window{86da12c u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #33 Window{e7dd37e u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #32 Window{21a3500 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #31 Window{1418632 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #30 Window{4ef7394 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #29 Window{bdfb5a6 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #28 Window{b9430e8 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #27 Window{f2615da u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #26 Window{e2a00fc u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #25 Window{aaf1ace u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #24 Window{c2137d0 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #23 Window{595f882 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #22 Window{cce964 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #21 Window{3eaa2f6 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #20 Window{6b6e9b8 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #19 Window{ce5ce2a u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #18 Window{db3cccc u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #17 Window{5dcee1e u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #16 Window{7b6e6a0 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #15 Window{5f636d2 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #14 Window{664b34 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #13 Window{69c9c46 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #12 Window{36ece88 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #11 Window{4b3d27a u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #10 Window{598049c u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #9 Window{68c4d6e u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #8 Window{4984170 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #7 Window{a974122 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #6 Window{1a89904 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #5 Window{2daa196 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #4 Window{ad8df58 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #3 Window{12522ca u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #2 Window{ff6eac7 u0 com.example.chengang.fdtest/com.example.chengang.fdtest.MainActivity}:
Window #1 Window{2624127 u0 com.android.systemui/com.android.systemui.recents.RecentsActivity}:
Window #0 Window{df85d5b u0 com.android.systemui.ImageWallpaper}:
```

这一看window信息就很明显了com.example.chengang.fdtest.MainActivity有异常addWindow，  
如此就可以确定异常code深入分析了。

### 2.5 Sqlite Cursor

类似的案例在该博客中记录的比较详细：[https://blog.csdn.net/jk198310/article/details/43765263](https://blog.csdn.net/jk198310/article/details/43765263)  
意思是遇到  
"E CursorWindow: Could not allocate CursorWindow "  
"Can’t open DB due to too many open files "  
等错误Msg时可以怀疑可能有Fd 泄漏，但肯定不全是。  
**这里只看下cursor查询数据时也是需要打开fd的，值得注意的是CursorWindow默认大小是2MB，如过使用不当可能不仅导致fd泄漏风险，还有内存泄漏风险。**

通过Uri查询数据库所得到的数据集，保存在native层的CursorWindow中。CursorWindow的实质是共享内存的抽象，以实现跨进程数据共享。共享内存所採用的实现方式是文件映射。  
在ContentProvider端透过SQLiteDatabase的封装查询到的数据集保存在CursorWindow所指向的共享内存中。然后通过Binder把这片共享内存传递到ContentResolver端，即查询端。  
这样客户就能够通过Cursor来訪问这块共享内存中的数据集了。

详细打开fd过程：

![](https://upload-images.jianshu.io/upload_images/807220-0cacafd7197213c2.png)

image.png

### 2.6 Bitmap IPC

BitMap在IPC传递时需要打开fd进行传递，比如如下情况：  
一个应用使用包含一个bitmap的RemoteView去更新通知栏通知，更新一个通知实际上涉及到3个进程，app将通知IPC給system_server，system_server将通知转给注册收通知的进程，比如systemui 通知栏。  
而如果通知中还有Bitmap，bitmap在进行IPC时实际上是通过传递指向bitmap所在的匿名共享内存的fd进行传递的，即大致如下：

![](https://upload-images.jianshu.io/upload_images/807220-71177026a393a37f.png)

image.png

详细打开fd的过程：

![](https://upload-images.jianshu.io/upload_images/807220-a7f171d8911ca28a.png)

image.png

IPC时传递该Fd即可而不是传递整个Bitmap对象。  
即大致如下：  
a. app 中得bitmap存到ashmem中，IPC给system_Server时将指向该ashmem文件的fd writetoparcel成parcel对象传递給system_server  
b. system_server得到parcel对象再createFromParcel由该parcel中的fd得到bitmap所在的ashmem，即system_server中已IPC得到了app 传来的bitmap对象。  
c. system_server将bitmap传地给注册监听的进程比如systemui通知栏使用同样的方式，开辟ashmem存储文件保存bitmap，使用封装指向该ashmem的fd的parcel对象传地给有关进程。  
d. systemui同样使用createFromParcel获取到传递过来的fd从而等到该fd指向的ashmem，最后构造出最终的bitmap对象。

在这里插入图片描述

由以上可知，如过app断更新带bitmap的通知太过频繁，可能会不仅导致本进程fd泄漏，还有可能影响到其他进程，甚至影响到system_Server导致重启，毕竟system_Server作为中转需要更多的fd资源。

**类似case，log：**

```php
10-17 12:13:02.007 2096 2096 E AndroidRuntime: *** FATAL EXCEPTION IN SYSTEM PROCESS: main
10-17 12:13:02.007 2096 2096 E AndroidRuntime: java.lang.RuntimeException: Could not copy bitmap to parcel blob.
10-17 12:13:02.007 2096 2096 E AndroidRuntime: at android.graphics.Bitmap.nativeWriteToParcel(Native Method)
10-17 12:13:02.007 2096 2096 E AndroidRuntime: at android.graphics.Bitmap.writeToParcel(Bitmap.java:1553)
10-17 12:13:02.007 2096 2096 E AndroidRuntime: at android.widget.RemoteViews$BitmapCache.writeBitmapsToParcel(RemoteViews.java:984)
10-17 12:13:02.007 2096 2096 E AndroidRuntime: at android.widget.RemoteViews.writeToParcel(RemoteViews.java:2854)
10-17 12:13:02.007 2096 2096 E AndroidRuntime: at android.widget.RemoteViews.clone(RemoteViews.java:1903)
10-17 12:13:02.007 2096 2096 E AndroidRuntime: at android.app.Notification.cloneInto(Notification.java:1521)
10-17 12:13:02.007 2096 2096 E AndroidRuntime: at android.app.Notification.clone(Notification.java:1495)
10-17 12:13:02.007 2096 2096 E AndroidRuntime: at android.service.notification.StatusBarNotification.clone(StatusBarNotification.java:161)
10-17 12:13:02.007 2096 2096 E AndroidRuntime: at com.android.server.notification.NotificationManagerService$NotificationListeners.notifyPostedLocked(NotificationManagerService.java:3398)
10-17 12:13:02.007 2096 2096 E AndroidRuntime: at com.android.server.notification.NotificationManagerService$8.run(NotificationManagerService.java:2228)
10-17 12:13:02.007 2096 2096 E AndroidRuntime: at android.os.Handler.handleCallback(Handler.java:742)
10-17 12:13:02.007 2096 2096 E AndroidRuntime: at android.os.Handler.dispatchMessage(Handler.java:95)
10-17 12:13:02.007 2096 2096 E AndroidRuntime: at android.os.Looper.loop(Looper.java:157)
10-17 12:13:02.007 2096 2096 E AndroidRuntime: at com.android.server.SystemServer.run(SystemServer.java:302)
10-17 12:13:02.007 2096 2096 E AndroidRuntime: at com.android.server.SystemServer.main(SystemServer.java:176)
10-17 12:13:02.007 2096 2096 E AndroidRuntime: at java.lang.reflect.Method.invoke(Native Method)
10-17 12:13:02.007 2096 2096 E AndroidRuntime: at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:746)
10-17 12:13:02.007 2096 2096 E AndroidRuntime: at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:636)
```

**看下该case的fd信息：**

```rust
09-11 14:13:34.144 2546 2671 W FdInfoManager: MIUI_FD /proc/self/fd/396 -----> /dev/ashmem
09-11 14:13:34.144 2546 2671 W FdInfoManager: MIUI_FD /proc/self/fd/397 -----> /dev/ashmem
09-11 14:13:34.144 2546 2671 W FdInfoManager: MIUI_FD /proc/self/fd/398 -----> /dev/ashmem
09-11 14:13:34.144 2546 2671 W FdInfoManager: MIUI_FD /proc/self/fd/399 -----> /dev/ashmem
...
09-11 14:13:34.145 2546 2671 W FdInfoManager: MIUI_FD /proc/self/fd/412 -----> /dev/ashmem
09-11 14:13:34.146 2546 2671 W FdInfoManager: MIUI_FD /proc/self/fd/413 -----> /dev/ashmem
09-11 14:13:34.146 2546 2671 W FdInfoManager: MIUI_FD /proc/self/fd/414 -----> socket:[9368579]
09-11 14:13:34.146 2546 2671 W FdInfoManager: MIUI_FD /proc/self/fd/415 -----> /dev/ashmem
...
09-11 14:13:34.157 2546 2671 W FdInfoManager: MIUI_FD /proc/self/fd/524 -----> /dev/ashmem
09-11 14:13:34.157 2546 2671 W FdInfoManager: MIUI_FD /proc/self/fd/526 -----> /dev/ashmem
09-11 14:13:34.157 2546 2671 W FdInfoManager: MIUI_FD /proc/self/fd/527 -----> /dev/ashmem
09-11 14:13:34.158 2546 2671 W FdInfoManager: MIUI_FD /proc/self/fd/529 -----> /dev/ashmem
...
09-11 14:13:34.161 2546 2671 W FdInfoManager: MIUI_FD /proc/self/fd/552 -----> /dev/ashmem
09-11 14:13:34.161 2546 2671 W FdInfoManager: MIUI_FD /proc/self/fd/553 -----> /dev/ashmem
09-11 14:13:34.161 2546 2671 W FdInfoManager: MIUI_FD /proc/self/fd/555 -----> /dev/ashmem
```

fd信息中ashmem信息明显很多，单通常只根据该异常调用栈，和ashmem较多的fd信息还不足以断定就是包含bitmap的通知弹的过多导致，这时需要查看logcat，需要看下此时通知是否真得事很多，比如该case的logcat中搜notification相关的日志出现大量更新notification的信息，且很多和notification相关的异常，即可以朝这个方向去找是谁发的notification有大量的bitmap传递。  
一般这样的app如阅读类app，计步类app会频繁更新通知栏的应用嫌疑比较大。  
关于bitmap的remoteview更新notification的问题事实上是aosp的原生bug，具体不在此讨论。

**当然，不仅bitmap，类似的需要使用到fd进行IPC传递的对象都有可能异常传地而导致fd泄漏问题，或躺枪，==。**

## 3\. Fd泄漏问题分析需要什么信息

#### 3.1 logcat 查看异常栈情况，什么信息在频繁的打印，涉及到fd open的日志频繁打印是最直接怀疑的地方。

#### 3.2 fdinfo 查看进程fd的打开情况，ls -la /proc/{pid}/fd/ 即可，或者 lsof命令打印指定进程或所有进程的fd信息，根据fd的打开类型推测是什么类型文件的泄漏，多了fd类型线索然后根据日志去推测。

#### 3.3 trace or ps信息 跟线程相关的fd泄漏通过进程的信息可以很快定位出异常线程。

#### 3.4 dumpsys window 通过window信息可以很快定位出异常window用于解决inputchannel相关的fd泄漏问提。

## 4\. Fd泄漏问题信息获取

当然知道怎么分析fd泄漏问题还是远远不够的，对于fd泄漏的问提最重要的应该就是相关信息的日志获取了，没有米怎么做饭，没有日志怎么分析，因此下面讨论下关于fd泄漏相关的日志获取。  
一. 容易复现，fd泄漏时可能会伴随卡顿异常等现象，可以在进程复现未挂的时候抓取想要的信息：  
1.查看fd信息adb shell ls -a -l /proc/<pid>/fd ，lsof  
2.查看进程线程信息：ps -t <pid>,或者抓进程trace, kill -3 <pid>  
3.抓取hprof定位资源使用情况  
二.难复现，通常fd泄漏都不会是那么容易复现的，那就需要特殊的日志信息获取方式了：  
1.对于应用自身fd泄漏发生JE时可以在复写UncatchHandlerException在应用crash的时候通过readlink的方式读取/proc/self/fd的信息，以获取fd信息，抓取进程的ps信息或者trace信息，抓取window情况，dumpsys window等;

2.O之后NE的Tombstone文件中有open files，可以查看打开的fd信息;

3.针对rom厂商，可以在RuntimeInit.java中LoggingHandler 中对于Error Msg进行判断，如怀疑有fd泄漏即读取/proc/self/fd/中的fd信息，记得使用readlink方法，当然当fd满了的时候再读的时候可能会读不成功，尝试使用i前面提供的方法提高进程的fd rlimit再读即可（我厂手机就是这么干的）:  
这种方法可以针对所有java进程的fd泄漏异常时的fd信息获取，RuntimeInit.java跑在各个应用的进程中;  
如过再做个daemon进程进行判断是否需要抓取fd信息则可以根据情况进行打印，比如在跑稳定性测试时默认打印所有fd泄漏的进程的fd信息，方便debug分析。

4.如过自动化测试可以复现，也可以尝试写个脚本，一定时间获取下指定进程的fd信息，通过lsof，或ls -la /proc/<pid>/fd/方式获取到fd数量，当fd数量高于指定值比如900后执行抓取需要的log信息脚本，比如logcat，trace，hprof，dumpsys window等;

**总之就是根据打开的文件类型，以及上下文的log来推断重复打开为关闭的文件句柄泄漏的代码位置。**

## 5\. 总结

以上时序图的uml文件及简化图片的draw.io的xml文件已分享至百度云，如有修改可以下载自行修改：  
[https://pan.baidu.com/s/1I9GlkVmeSCKAS8JEqCbSaA](https://pan.baidu.com/s/1I9GlkVmeSCKAS8JEqCbSaA)

©

著作权归作者所有,转载或内容合作请联系作者  
【社区内容提示】社区部分内容疑似由AI辅助生成，浏览时请结合常识与多方信息审慎甄别。  
平台声明：文章内容（如有图片或视频亦包括在内）由作者上传并发布，文章内容仅代表作者本人观点，简书系信息发布平台，仅提供信息存储服务。

17人点赞

更多精彩内容，就在简书APP

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![  ](https://cdn2.jianshu.io/assets/default_avatar/5-33d2da32c552b8be9a0548c7a4576607.jpg)](https://www.jianshu.com/u/4311af2fe8b5)

总资产6共写了4.5W字获得127个赞共178个粉丝

### 相关阅读[更多精彩内容](https://www.jianshu.com/)

- 用两张图告诉你，为什么你的 App 会卡顿? - Android - 掘金 Cover 有什么料？ 从这篇文章中你...
- Android 自定义View的各种姿势1 Activity的显示之ViewRootImpl详解 Activity...
- 每个地方都有其独特的魅力，内蒙响沙湾的落日让我印象深刻 【照片拍摄于国庆去内蒙响沙湾旅游期间】 落日时整片天空被晚...

  [![](https://cdn2.jianshu.io/assets/default_avatar/5-33d2da32c552b8be9a0548c7a4576607.jpg)江酱\_](https://www.jianshu.com/u/099f95510d4e)阅读 3,073评论 1赞 2

- 总想打翻时光的杯子 你进的来 或者，我出的去 再重复一场仅有的 相遇 而我，也绝不会急着寻求未来 苛刻明日 看着你...

  [![](https://cdn2.jianshu.io/assets/default_avatar/9-cceda3cf5072bcdd77e8ca4f21c40998.jpg)道夫123](https://www.jianshu.com/u/41ed32e069a6)阅读 2,839评论 12赞 39

