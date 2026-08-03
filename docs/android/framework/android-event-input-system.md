---
title: 图解Android - Android GUI 系统 (5) - Android的Event Input System
url: https://www.cnblogs.com/samchen2009/p/3368158.html
publishedTime: 2013-11-12T19:27:00.0000000+08:00
---

## Android的用户输入处理

Android的用户输入系统获取用户按键（或模拟按键）输入，分发给特定的模块（Framework或应用程序）进行处理，它涉及到以下一些模块：

+   Input Reader: 负责从硬件获取输入，转换成事件（Event), 并分发给Input Dispatcher.
+   Input Dispatcher: 将Input Reader传送过来的Events 分发给合适的窗口，并监控ANR。
+   Input Manager Service： 负责Input Reader 和 Input Dispatchor的创建，并提供Policy 用于Events的预处理。
+   Window Manager Service：管理Input Manager 与 View（Window) 以及 ActivityManager 之间的通信。
+   View and Activity：接收按键并处理。
+   ActivityManager Service：ANR 处理。

它们之间的关系如下图所示（黑色箭头代表控制信号传递方向，而红色箭头代表用户输入数据的传递方向）。

![](https://images0.cnblogs.com/blog/563439/201311/06230834-a8145552ca56482c92b6179dff6bdb5c.png)

这块代码很多，但相对来说不难理解，按照惯例，我们先用一张大图（点击看大图）鸟瞰一下全貌先。

[![](https://images0.cnblogs.com/blog/563439/201311/06231217-1c6eb385295b44d29ef7064e7788b731.png)](https://images0.cnblogs.com/blog/563439/201311/06231217-1c6eb385295b44d29ef7064e7788b731.png)

四种不同颜色代表了四个不同的线程, InputReader Thread，InputDispatch Thread 和 Server Thread 存在于SystemServer进程里。UI Thread则存在于Activity所在进程。颜色较深部分是比较重要，需要重点分析的模块。

**初始化**

整个输入系统的初始化可以划分为Java 和 Native两个部分，可以用两张时序图分别描述，首先看Java端，  

![](https://images0.cnblogs.com/blog/563439/201311/07000238-d1c984cbaf4d4c7abeca35ea460d0223.png)

1.  在SystemServer的初始化过程中，InputManagerService 被创建出来，它做的第一件事情就是初始化Native层，包括EventHub, InputReader 和 InputDispatcher，这一部分我们将在后面详细介绍。
2.  当InputManager Service 以及其他的System Service 初始化完成之后，应用程序就开始启动。如果一个应用程序有Activity（只有Activit能够接受用户输入），它要将自己的Window(ViewRoot)通过setView()注册到Window Manager Service 中。（详见[图解Android - Android GUI 系统 (2) - 窗口管理 (View, Canvas, Window Manager)](http://www.cnblogs.com/samchen2009/p/3367496.html)）。
3.  用户输入的捕捉和处理发生在不同的进程里（生产者：Input Reader 和 Input Dispatcher 在System Server 进程里，而消耗者，应用程序运行在自己的进程里），因此用户输入事件（Event)的传递需要跨进程。在这里，Android使用了Socket 而不是 Binder来完成。OpenInputChannelPair 生成了两个Socket的FD， 代表一个双向通道的两端，向一端写入数据，另外一端便可以读出，反之依然，如果一端没有写入数据，另外一端去读，则陷入阻塞等待。OpenInputChannelPair() 发生在WindowManager Service 内部。为什么不用binder？ 个人的分析是，Socket可以实现异步的通知，且只需要两个线程参与（Pipe两端各一个），假设系统有N个应用程序，跟输入处理相关的线程数目是 n+1 (1是发送（Input Dispatcher）线程）。然而，如果用Binder实现的话，为了实现异步接收，每个应用程序需要两个线程，一个Binder线程，一个后台处理线程，（不能在Binder线程里处理输入，因为这样太耗时，将会堵塞住发送端的调用线程）。在发送端，同样需要两个线程，一个发送线程，一个接收线程来接收应用的完成通知，所以，N个应用程序需要 2（N+1)个线程。相比之下，Socket还是高效多了。
4.  通过RegisterInputChannel, Window Manager Service 将刚刚创建的一个Socket FD，封装在InputWindowHandle(代表一个WindowState) 里传给InputManagerService。
5.  InputManagerService 通过JNI（NativeInputManager）最终调用到了InputDispatchor 的 RegisterInputChannel()方法，这里，一个Connection 对象被创建出来，代表与远端某个窗口(InputWindowHandle)的一条用户输入数据通道。一个Dispatcher可能有多个Connection（多个Window）同时存在。为了监听来自于Window的消息，InputDispator 通过AddFd 将这些个FD 加入到Looper中，这样，只要某个Window在Socket的另一端写入数据，Looper就会马上从睡眠中醒来，进行处理。
6.  到这里，ViewRootImpl 的 AddWindow 返回，WMS 将SocketPair的另外一个FD 放在返回参数 OutputChannel 里。
7.  接着ViewRootImpl 创建了WindowInputEventReceiver 用于接受InputDispatchor 传过来的事件，后者同样通过AddFd() 将读端的Socket FD 加入到Looper中，这样一旦InputDispatchor发送Event，Looper就会立即醒来处理。

 接下来看刚才没有讲完的NativeInit。

![](https://images0.cnblogs.com/blog/563439/201311/07010833-7e69a96f53df4c489549024edbd57f4e.png)

1.  NativeInit 是 NativeInputManager类的一个方法，在InputManagerService的构造函数中被调用。代码在 frameworks/base/services/jni/com\_android\_server\_input\_inputManagerService.cpp.
2.  首先创建一个EventHub, 用来监听所有的event输入。
3.  创建一个InputDispatchor对象。
4.  创建一个InputReader对象，他的输入是EventHub， 输出是InputDispatchor。
5.  然后分别为InputReader 和 InputDispatchor 创建各自的线程。注意，当前运行在System Server 的 WMThread线程里。
6.  接着，InputManagerService 调用NativeStart 通知InputReader 和 InputDispatchor 开始工作。
7.  InputDispatchor是InputReader的消费者，它的线程首先启动，进入Looper等待状态。
8.  接着 InputReader 线程启动，等待用户输入的发生。

至此，一切准备工作就绪，万事具备，之欠用户一击了。

## Eventhub 和 Input Reader

Android设备可以同时连接多个输入设备，比如说触摸屏，键盘，鼠标等等。用户在任何一个设备上的输入就会产生一个中断，经由Linux内核的中断处理以及设备驱动转换成一个Event，并传递给用户空间的应用程序进行处理。每个输入设备都有自己的驱动程序，数据接口也不尽相同，如何在一个线程里（上面说过只有一个InputReader Thread)把所有的用户输入都给捕捉到? 这首先要归功于Linux 内核的输入子系统（Input Subsystem), 它在各种各样的设备驱动程序上加了一个抽象层，只要底层的设备驱动程序按照这层抽象接口来实现，上层应用就可以通过统一的接口来访问所有的输入设备。这个抽象层有三个重要的概念，input handler, input handle 和 input\_dev,它们的关系如下图所示：

    ![](https://images0.cnblogs.com/blog/563439/201311/07195811-678e3594c3fa4dcead6d99f2b9f65267.png)

+   input\_dev 代表底层的设备，比如图中的“USB keyboard" 或 "Power Button" (PC的电源键），所有设备的input\_dev 对象保存在一个全局的input\_dev 队列里。
+   input\_handler 代表某类输入设备的处理方法，比如说 evdev就是专门处理输入设备产成的Event（事件），而“sysrq" 是专门处理键盘上“sysrq"与其他按键组合产生的系统请求，比如“ALT+SysRq+p"（先Ctrl+ALT+F1切换到虚拟终端)可以打印当前CPU的寄存器值。所有的input\_handler 存放在 input\_handler队列里。
+   一个input\_dev 可以有多个input\_handler, 比如下图中“USB Mouse" 设备可以由”evdev" 和 “mousedev" 来分别处理它产生的输入。
+   同样，一个input\_handler 可以用于多种输入设备，比如“USB Keyboard", "Power Button" 都可以产成Event，所以，这些Event都可以交由evdev进行处理。
+   Input handle 用来关联某个input\_dev 和 某个 input\_handler, 它对应于下图中的紫色的原点。每个input handle 都会生成一个文件节点，比如图中4个 evdev的handle就对应与 /dev/input/下的四个文件"event0~3". 通过input handle, 可以找到对应的input\_handler 和 input\_dev.

简单说来，input\_dev对应于底层驱动，而input\_handler是个上层驱动，而input\_handle 提供给应用程序标准的文件访问接口来打通这条上下通道。通过Linux input system获取用户输入的流程简单如下：

1.  设备通过input\_register\_dev 将自己的驱动注册到Input 系统。
2.  各种Handler 通过 input\_register\_handler将自己注册到Input系统中。
3.  每一个注册进来的input\_dev 或 Input\_handler 都会通过input\_connect() 寻找对方，生成对应的 input\_handle，并在/dev/input/下产成一个设备节点文件. 
4.  应用程序通过打开(Open)Input\_handle对应的文件节点，打开其对应的input\_dev 和 input\_handler的驱动。这样，当用户按键时，底层驱动就能捕捉到，并交给对应的上次驱动（handler)进行处理，然后返回给应用程序，流程如下图中红色箭头所示。

上图中的深色点就是 Input Handle， 左边垂直方向是Input Handler， 而水平方向是Input Dev。 下面是更为详细的一个流程图，感兴趣的同学可以点击大图看看。

[![](https://images0.cnblogs.com/blog/563439/201311/07225317-0771ad2449024d3eb90d1690710a676e.png)](https://images0.cnblogs.com/blog/563439/201311/07225317-0771ad2449024d3eb90d1690710a676e.png)

所以，只要打开 /dev/input/ 下的所有 event\* 设备文件，我们就可以有办法获取所有输入设备的输入事件，不管它是触摸屏，还是一个USB 设备，还是一个红外遥控器。Android中完成这个工作的就是EventHub。

EventHub实现在 framework/base/services/input/EventHub.cpp, 它和InputReader 的工作流程如下图所示：

![](https://images0.cnblogs.com/blog/563439/201311/08002356-d9405c2dfd9c4ebbacbf68ef958500a4.png) 

1.   NativeInputManager的构造函数里第一件事情就是创建一个EventHub对象，它的构造函数里主要生成并初始化几个控制的FD：  
    1.  mINotifyFd: 用来监控""/dev/input"目录下是否有文件生成，有的话说明有新的输入设备接入，EventHub将从epool\_wait中唤醒，来打开新加入的设备。
    2.  mWakeReaderFD， mWakeWriterFD： 一个Pipe的两端，当往mWakeWriteFD 写入数据的时候，等待在mWakeReaderFD的线程被唤醒，这里用来给上层应用提供唤醒等待线程，比如说，当上层应用改变输入属性需要EventHub进行相应更新时。
    3.  mEpollFD，用于epoll\_wait()的阻塞等待，这里通过epoll\_ctrl(EPOLL\_ADD\_FD, fd) 可以等待多个fd的事件，包括上面提到的mINotifyFD, mWakeReaderFD, 以及输入设备的FD。
2.  紧接着，InputManagerService启动InputReader 线程，进入无限的循环，每次循环调用loopOnce(). 第一次循环，会主动扫描 "/dev/input/" 目录，并打开下面的所有文件，通过ioctl()从底层驱动获取设备信息，并判断它的设备类型。这里处理的设备类型有：INPUT\_DEVICE\_CLASS\_KEYBOARD， INPUT\_DEVICE\_CLASS\_TOUCH， INPUT\_DEVICE\_CLASS\_DPAD，INPUT\_DEVICE\_CLASS\_JOYSTICK 等。
3.  找到每个设备对应的键值映射文件，读取并生产一个KeyMap 对象。一般来说，设备对应的键值映射文件是 "/system/usr/keylayout/Vendor\_%04x\_Product\_%04x".
4.  将刚才扫描到的/dev/input 下所有文件的FD 加到epool等待队列中，调用epool\_wait() 开始等待事件的发生。
5.  某个时间发生，可能是用户按键输入，也可能是某个设备插入，亦或用户调整了设备属性，epoll\_wait() 返回，将发生的Event 存放在mPendingEventItems 里。如果这是一个用户输入，系统调用Read() 从驱动读到这个按键的信息，存放在rawEvents里。
6.  getEvents() 返回，进入InputReader的processEventLocked函数。
7.  通过rawEvent 找到产生时间的Device，再找到这个Device对应的InputMapper对象，最终生成一个NotifyArgs对象，将其放到NotifyArgs的队列中。
8.  第一次循环，或者后面发生设备变化的时候（比如说设备拔插），调用 NativeInputManager 提供的回调，通过JNI通知Java 层的Input Manager Service 做设备变化的相应处理，比如弹出一个提示框提示新设备插入。这部分细节会在后面介绍。
9.  调用NotifyArgs里面的Notify()方法，最终调用到InputDispatchor 对应的Notify接口（比如NotifyKey) 将接下来的处理交给InputDispatchor，EventHub 和 InputReader 工作结束，但马上又开始新的一轮等待，重复6～9的循环。

## Input Dispatcher

接下来看看目前为止最长一张时序图，通过下面18个步骤，事件将发送到应用程序进行处理。

[![](https://images0.cnblogs.com/blog/563439/201311/09182433-546a06b6bc6449469821ceb144fa36a1.png)](https://images0.cnblogs.com/blog/563439/201311/09182433-546a06b6bc6449469821ceb144fa36a1.png)

1.  接上节的最后一步，NotifyKey() 的实现在Input Dispatcher 内部，他首先做简单的校验，对于按键事件，只有Action 是 AKEY\_EVENT\_ACTION\_DOWN 和 AKEY\_EVENT\_ACTION\_UP，即按下和弹起这两个Event别接受。
2.  Input Reader 传给Input Dispather的数据类型是 NotifyKeyArgs， 后者在这里将其转换为 KeyEvent, 然后交由 Policy 来进行第一步的解析和过滤，interceptKeyBeforeQueuing, 对于手机产品，这个工作是在PhoneWindowManager 里完成，（不同类型的产品可以定义不同的WindowManager, 比如GoogleTV 里用到的是TVWindowManager)。KeyEvent 在这里将会被分为三类：
    1.  System Key: 比如说 音量键，Power键，电话键，以及一些特殊的组合键，如用于截屏的音量+Power，等等。部分System Key 会在这里立即处理，比如说电话键，但有一些会放到后面去做处理，比如说音量键，但不管怎样，这些键不会传给应用程序，所以称为系统键。
    2.  Global Key：最终产品中可能会有一些特殊的按键，它不属于某个特定的应用，在所有应用中的行为都是一样，但也不包含在Andrioid的系统键中，比如说GoogleTV 里会有一个“TV” 按键，按它会直接呼起“TV”应用然后收看电视直播，这类按键在Android定义为Global Key.
    3.  User Key：除此之外的按键就是User Key, 它最终会传递到当前的应用窗口。
3.  phoneWindowManager的interceptKeyBeforeQueuing() 最后返回了wmActiions，里面包含若干个flags，NativeInputManager在handleInterceptActions()， 假如用户按了Power键，这里会通知Android睡眠或唤醒。最后，返回一个 policyFlags，结束第一次的intercept 过程。
4.  接下来，按键马上进入第二轮处理。如果用户在Setting->Accessibility 中选择打开某些功能，比如说手势识别，Android的AccessbilityManagerService（辅助功能服务) 会创建一个 InputFilter 对象，它会检查输入的事件，根据需要可能会转换成新的Event，比如说两根手指头捏动的手势最终会变成ZOOM的event. 目前，InputManagerService 只支持一个InputFilter, 新注册的InputFilter会把老的覆盖。InputFilter 运行在SystemServer 的 ServerThread 线程里（除了绘制，窗口管理和Binder调用外，大部分的System Service 都运行在这个线程里）。而filterInput() 的调用是发生在Input Reader线程里，通过InputManagerService 里的 InputFilterHost 对象通知另外一个线程里的InputFilter 开始真正的解析工作。所以，InputReader 线程从这里结束一轮的工作，重新进入epoll\_wait() 等待新的用户输入。InputFilter 的工作也分为两个步骤，首先由InputEventConsistencyVerifier 对象（InputEventConsistencyVerifier.java）对输入事件的完整性做一个检查，检查事件的ACTION\_DOWN 和 ACTION\_UP 是否一一配对。很多同学可能在Android Logcat 里看到过以下一些类似的打印："ACTION\_UP but key was not down." 就出自此处。接下来，进入到AccessibilityInputFilter 的 onInputEvent()，这里将把输入事件（主要是MotionEvent)进行处理，根据需要变成另外一个Event，然后通过sendInputEvent（）将事件发回给InputDispatcher。最终调用到injectInputEvent() 将这个事件送入 mInBoundQueue.
5.  这个时候，InputDispather 还在Looper中睡眠等待，injectInputEvent（）通过wake() 将其唤醒。这是进入Input Dispatcher 线程。
6.  InputDispatcher 大部分的工作在 dispatcherOnce 里完成。首先从mInBoundQueue 中读出队列头部的事件 mPendingEvent, 然后调用 pokeUserActivity(). poke的英文意思是"搓一下, 捅一下“， 这个函数的目的也就是”捅一下“PowerManagerService 提醒它”别睡眠啊，我还活着呢“，最终调用到PowerManagerService 的 updatePowerStateLocked()，防止手机进入休眠状态。需要注意的是，上述动作不会马上执行，而是存储在命令队列，mCommandQueue里，这里面的命令会在后面依次被执行。
7.  接下来是dispatchKeyLocked(), 第一次进去这个函数的时候，先检查Event是否已经过处理（interceptBeforeDispatching), 如果没有，则生成一个命令，同样放入mCommandQueue里。
8.  runCommandsLockedInterruptible() 依次执行mCommandQueue 里的命令，前面说过，pokeUserActivity 会调用PowerManagerService 的 updatePowerStateLocked(), 而 interceptKeyBeforeDispatching() 则最终调用到PhoneWindowManager的同名函数。我们在interceptBeforeQueuing 里面提到的一些系统按键在这个被执行，比如 HOME/MENU/SEARCH 等。
9.  接下来，处理前面提过GlobalKey，GlobalKeyManager 通过broadcast将这些全局的Event发送给感兴趣的应用。最终，interceptKeyBeforeDispatching 将返回一个Int值，-1 代表Skip，这个Event将不会发送给应用程序。0 代表 Continue, 将进入下一步的处理。1 则表明还需要后续的Event才能做出决定。
10.  命令运行完之后，退出 dispatchOnce， 然后调用pollOnce 进入下一轮等待。但这里不会被阻塞，因为timeout值被设成了0.
11.  第二次进入dispatchKeyLocked(), 这是Event的状态已经设为”已处理“，这时候才真正进入了发射阶段。
12.  接下来调用 findFocusedWindowTargetLocked() 获取当前的焦点窗口，这里面会做一件非常重要的事情，就是检测目标应用是否有ANR发生，如果下诉条件满足，则说明可能发生了ANR：
    1.  目标应用不会空，而目标窗口为空。说明应用程序在启动过程中出现了问题。
    2.  目标 Activity 的状态是Pause，即不再是Focused的应用。
    3.  目标窗口还在处理上一个事件。这个我们下面会说到。
13.  如果目标窗口处于正常状态，调用dispatchEventLocked() 进入真正的发送程序。
14.  这里，事件又换了一件马甲，从EventEntry 变成 DispatchEntry, 并送人mOutBoundQueue。然后调用startDispatchCycle() 开始发送。
15.  最终的发送发生在InputPublish的sendMessage()。这里就用到了我们前面提到的SocketPair, 一旦sendMessage() 执行，目标窗口所在进程的Looper线程就会被唤醒，然后读取键值并进行处理，这个过程我们下面马上就会谈到。
16.  乖乖，还没走完啊？是的，工作还差最后一步，Input Dispatcher给这个窗口发送下一个命令之前，必须等待该窗口的回复，如果超过5s没有收到，就会通过Input Manager Service 向Activity Manager 汇报，后者会弹出我们熟知的 "Application No Response" 窗口。所以，事件会放入mWaitQueue进行暂存。如果窗口一切正常，完成按键处理后它会调用InputConsumer的sendFinishedSignal() 往SocketPair 里写入完成信号，Input Dispatcher 从 Loop中醒来，并从Socket中读取该信号，然后从mWaitQueue 里清除该事件标志其处理完毕。
17.  并非所有的事件应用程序都会处理，如果没有处理，窗口程序返回的完成消息里的 msg.body.finished.handled 会等于false，InputDispatcher 会调用dispatchKeyUnhandled() 将其交给PhoneWindowManager。Android 在这里提供了一个Fallback机制，如果在 /system/usr/keychars/ 下面的kcm文件里定义了 fallback关键字，Android就识别它为一个Fallback Keycode。当它的Parent Keycode没有被应用程序处理，InputDispatcher 会把 Fallback Keycode 当成一个新的Event，重新发给应用程序。下面是一个定义Fallback Key 的例子。如果按了小键盘的0且应用程序不受理它，InputDispatcher 会再发送一个'INSERT' event 给应用程序。
    ```
    #/system/usr/keychars/generic.kcm
    ...
    key NUMPAD_0 {
        label: '0'             //打印字符
        base: fallback INSERT  //behavior
        numlock: '0'        //在一个textView里输出的字符
    }
    ```
18.  经历了重重关卡，一个按键发送的流程终于完成了，不管有没有Fallback Key存在，调用startDispatcherCycle() 开始下一轮征程。。。

史上最长的流程图终于介绍完了，有点迷糊了？好吧，再看看下面这张图总结一下：

+   InputDispatcher 是一个异步系统，里面用到3个Queue（队列）来保存中间任务和事件，分别是 mInBoundQueue, mOutBoundQueue，mWaitQueue不同队列的进出划分了按键的不同处理阶段。
+   InputReader 采集的输入实现首先经过InterceptBeforeQueuing处理，Android 系统会将这些按键分类（System/Global/User)， 这个过程是在InputReader线程里完成。
+   如果是Motion Event, filterEvent()可能会将其转换成其他的Event。然后通过InjectKeyEvent 将这个按键发给InputDispatcher。这个过程是在System Process的ServerThread里完成。
+   在进入mOutBoundQueue 之前，首先要经过 interceptBeforeDispatching() 的处理，System 和 Global 事件会在这个处理，而不会发送给用户程序。
+   通过之前生成的Socket Pair, InputPublish 将 Event发送给当前焦点窗口，然后InputDispatcher将Event放入mWaitQueue 等待窗口的回复。
+   如果窗口回复，该对象被移出mWaitQueue， 一轮事件处理结束。如果窗口没有处理该事件，从kcm文件里搜寻Fallback 按键，如果有，则重新发送一个新的事件给用户。
+   如果超过5s没有收到用户回复，则说明用户窗口出现阻塞，InputDispather 会通过Input Manager Service发送ANR给ActivityManager。

##  [![](https://cacoo.com/diagrams/AAtJbtBiDIos7Uex-52A09.png)](https://cacoo.com/diagrams/AAtJbtBiDIos7Uex-52A09.png)

## Key processing

前面我们说过，NativeInputEventReceiver() 通过addFd() 将SocketPair的一个FD 加入到UI线程的loop里，这样，当Input Dispatcher在Socket的另外一端写入Event数据，应用程序的UI线程就会从睡眠中醒来，开始事件的处理流程。时序图如下所示：

 ![](https://images0.cnblogs.com/blog/563439/201311/12182030-2e67f0b4f93d44278fabfe7e8332c21d.png)

1.   收到的时间首先会送到队列中，ViewRootImpl 通过 deliverInputEvent() 向InputStage传递消息。
2.  InputStage 是 Android 4.3 新推出的实现，它将输入事件的处理分成若干个阶段（Stage）, 如果当前有输入法窗口，则事件处理从 NativePreIme 开始，否则的话，从EarlyPostIme 开始。事件会依次经过每个Stage，如果该事件没有被标识为 “Finished”， 该Stage就会处理它，然后返回处理结果，Forward 或 Finish， Forward 运行下一Stage继续处理，而Finished事件将会简单的Forward到下一级，直到最后一级 Synthetic InputStage。流程图和每个阶段完成的事情如下图所示。
    
    [![](https://cacoo.com/diagrams/I6JISipb1F5tzN5d-C4027.png)](https://cacoo.com/diagrams/I6JISipb1F5tzN5d-C4027.png)
    
3.  最后 通过finishInputEvent() 回复InputDispatcher。
