---
title: Android AOP之字节码插桩
url: https://juejin.cn/post/6844903473599741965
publishedTime: 2017-04-11T15:26:11+08:00
---

---

title: Android AOP之字节码插桩  
author: 陶超  
description: 实现数据收集SDK时，为了实现非侵入的，全量的数据采集，采用了AOP的思想，探索和实现了一种Android上AOP的方式。本文基于数据收集SDK的AOP实现总结而成。  
categories: Android  
date: 2017/02/11  
tags:

- Android AOP
- 字节码
- java
- bytecode
- 数据收集

---

## 背景

  本篇文章基于《网易乐得无埋点数据收集SDK》总结而成，关于网易乐得无埋点数据采集SDK的功能介绍以及技术总结后续会有文章进行阐述，本篇单讲SDK中用到的Android端AOP的实现。

  随着流量红利时代过去，精细化运营时代的开始，网易乐得开始构建自己的大数据平台。其中，客户端数据采集是第一步。传统收集数据的方式是埋点，这种方式依赖开发，采集时效慢，数据采集代码与业务代码不解藕。

  为了实现非侵入的，全量的数据采集，AOP成了关键，数据收集SDK探索和实现了一种Android上AOP的方式。

## 目录

- [一、Android AOP](#1 "#1")  
   [1.1 什么是AOP](#1.1 "#1.1")  
   [1.2 Android AOP方式概述](#1.2 "#1.2")  
   [1.3 Android AOP方式对比选择](#1.3 "#1.3")
- [二、AOP应用情景](#2 "#2")  
   [2.1 Fragment生命周期](#2.1 "#2.1")  
   [2.2 用户点击事件](#2.2 "#2.2")  
   [2.3 弹窗事件](#2.3 "#2.3")
- [三、AOP实现概述](#3 "#3")
- [四、插桩入口](#4 "#4")  
   [4.1 Android打包流程说明](#4.1 "#4.1")  
   [4.2 插桩入口](#4.2 "#4.2")  
   [4.3 hook dx.jar获得插桩入口](#4.3 "#4.3")
- [五、bytecode manipulation](#5 "#5")  
   [5.1 ASM库简要介绍](#5.1 "#5.1")  
   [5.2 字节码基础](#5.2 "#5.2")  
   [5.3 bytecode manipulation实践](#5.3 "#5.3")
- [六、总结](#6 "#6")

## 一、Android AOP

## 1.1 什么是AOP

  面向切向编程（Aspect Oriented Programming），相对于面向对象编程（ObjectOriented Programming）而言。  
  OOP的精髓是把功能或问题模块化，每个模块处理自己的家务事。但在现实世界中，并不是所有问题都能完美得划分到模块中，有些功能是横跨并嵌入众多模块里的，比如下图所示的例子。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/4/11/64ddf269f57faf755052c66dfba57f9e~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

图1-1 AOP概念说明示例

  上图是一个APP模块结构示例，按照照OOP的思想划分为“视图交互”，“业务逻辑”，“网络”等三个模块，而现在假设想要对所有模块的每个方法耗时（性能监控模块）进行统计。这个性能监控模块的功能就是需要横跨并嵌入众多模块里的，这就是典型的AOP的应用场景。

  AOP的目标是把这些横跨并嵌入众多模块里的功能（如监控每个方法的性能） 集中起来，放到一个统一的地方来控制和管理。如果说，OOP如果是把问题划分到单个模块的话，那么AOP就是把涉及到众多模块的某一类问题进行统一管理。

  我们在开发无埋点数据收集是同样也遇到了很多需要横跨并嵌入众多模块里的场景，这些场景将在第二章（AOP应用情景）进行介绍。下面我们调研下Android AOP的实现方式。

## 1.2 Android AOP方式概述

  AOP从实现原理上可以分为运行时AOP和编译时AOP，对于Android来讲运行时AOP的实现主要是hook某些关键方法，编译时AOP主要是在Apk打包过程中对class文件的字节码进行扫描更改。Android主流的aop 框架有：

- Dexposed，Xposed等（运行时）
- aspactJ（编译时）

  除此之外，还有一些非框架的但是能帮助我们实现 AOP的工具类库：

- java的动态代理机制(对java接口有效)
- ASM,javassit等字节码操作类库
- (偏方)DexMaker:Dalvik 虚拟机上，在编译期或者运行时生成代码的 Java API。
- (偏方)ASMDEX(一个类似 ASM 的字节码操作库，运行在Android平台，操作Dex字节码)

## 1.3 Android AOP方式对比选择

  Dexposed，Xposed的缺陷很明显，xposed需要root权限，Dexposed只对部分系统版本有效。  
  与之相比aspactJ没有这些缺点，但是aspactJ作为一个AOP的框架来讲对于我们来讲太重了，不仅方法数大增，而且还有一堆aspactJ的依赖要引入项目中（这些代码定义了aspactJ框架诸如切点等概念）。更重要的是我们的目标仅仅是按照一些简单的切点（用户点击等）收集数据，而不是将整个项目开发从OOP过渡到AOP。  
  AspactJ对于我们想要实现的数据收集需求太重了，但是这种编译期操作class文件字节码实现AOP的方式对我们来说是合适的。  
  因此我们实现Android上AOP的方式确定为：

- 采用编译时的字节码操作的做法
- 自己hook Android编译打包流程并借助ASM库对项目字节码文件进行统一扫描，过滤以及修改。

  在具体讲解实现技术之前，先看一下无埋点数据收集需求遇到的三个需要AOP的场景。

## 二、AOP应用情景

  下面举出数据收集SDK通过修改字节码进行AOP的三个应用情景，其中情景一和二的字节码修改是方法级别的，情景三的字节码修改是指令级别的。

## 2.1 Fragment生命周期

### 说明

  收集页面数据时发现有些fragment是希望当作页面来看待，并且计算pv的(如首页用fragmen实现的tab)。而fragment的页面显示／隐藏事件需要根据：

```
 onResume()
onPause()
onHiddenChanged(boolean hidden)
setUserVisibleHint(boolean isVisibleToUser)
```

  这四个方法综合得出。  
  也就是说当项目中任一一个Fragment发生如上状态变化，我们都要拿到这个时机，并上报相关页面事件,也就是对Fragment的这几个方法进行AOP。  
  做法是：

- 对项目中所有代码进行扫描，筛选出所有Fragment的子类
- 对这些筛选出来的类的的onResumed，onPaused，onHiddenChanged，setFragmentUserVisibleHint这几个方法的字节码进行修改，添加上类似回调的逻辑
- 这样在项目中任何一个Fragment的这些回调触发的时候我们都可以得到通知，也即对Fragment的这几个切点进行了AOP。

### 示例

  假设我们有一个Fragment1（空类，内部什么代码也没有）

```javascript
 public class Fragment1 extends Fragment {}
```

  经过扫描修改字节码后变为：

```javascript
 public class Fragment1 extends Fragment {

    @TransformedDCSDK
    public void onResume() {
        super.onResume();
        Monitor.onFragmentResumed(this);
    }

    @TransformedDCSDK
    public void onPause() {
        super.onPause();
        Monitor.onFragmentPaused(this);
    }

    @TransformedDCSDK
    public void onHiddenChanged(boolean var1) {
        super.onHiddenChanged(var1);
        Monitor.onFragmentHiddenChanged(this, var1);
    }

    @TransformedDCSDK
    public void setUserVisibleHint(boolean var1) {
        super.setUserVisibleHint(var1);
        Monitor.setFragmentUserVisibleHint(this, var1);
    }
}
```

注：

1.  Monitor.onFragmentResumed等函数用于上报页面事件
2.  @TransformedDCSDK 注解标记方法被数据收集SDK进行了字节码修改

## 2.2 用户点击事件

### 说明

  点击事件是分析用户行为的一个重要事件，Android中的点击事件回调大多是View.OnClickListener的onClick方法（当然还有一部分是DialogInterface.OnClickListener或者重写OnTouchEvent自己封装的点击）。  
  也就是说当项目中任一一个控件被点击（触发了OnClickListener），我们都要拿到这个时机，并上报点击事件。也就是对View.OnClickListener的onClick方法进行AOP。做法是：

- 对项目中所有代码进行扫描，筛选出所有实现View.OnClickListener接口的类（匿名or不匿名）
- 对onClick方法的字节码进行修改，添加回调。
- 达到的效果就是当APP中任何一个View被点击时，我们都可以在捕捉到这个时机，并且上报相关点击事件。

### 示例

  假设有个实现接口的类

```javascript
 public class MyOnClickListener implements OnClickListener {
    public void onClick(View v) {
        //此处代表点击发生时的业务逻辑
    }
}
```

经过扫描修改字节码后变为：

```javascript
 public class MyOnClickListener implements OnClickListener {
    @TransformedDCSDK
    public void onClick(View v) {
        if (!Monitor.onViewClick(v)) {
           //此处代表点击发生时的业务逻辑
        }
    }
}
```

注：

1.  Monitor.onViewClick函数里面包含上报点击事件的逻辑
2.  可以通过Monitor.onViewClick的返回值控制原有业务逻辑是否执行，基本都是执行的，只有在特殊模式下（圈选）数据收集SDK才会忽略原有逻辑

## 2.3 弹窗事件

### 说明

  弹窗显示/关闭事件，当然弹窗的实现可以是Dialog，PopupWindow，View甚至Activity，这里仅以Dialog为例。  
  当项目中任意一个地方弹出/关闭Dialog，我们都要拿到这个时机，即对Dialog.show/dismiss/hide这几个方法进行AOP。做法是：

- 对项目中所有代码进行扫描，筛选出所有字节码指令中有调用Dialog.show/dismiss/hide的地方
- 字节码指令替换，替换成一段回调逻辑。
- 这样APP中所有Dialog的显示/关闭时，我们都可以在这时进行一些收集数据的操作。

### 示例

  假设项目中有一个代码（例如方法）块如下，其中某处调用了dialog.show()

```
 某个方法 {
    //其他代码
    dialog.show()
    //其他代码
}
```

经过扫描修改字节码后变为

```
 某个方法 {
    //其他代码
    Monitor.showDialog(dialog)
    //其他代码
}
```

注：Monitor.showDialog除了调用dialog.show()还进行一些数据收集逻辑

## 三、AOP实现概述

  第二章 （AOP应用情景）简单地列举了AOP在三种应用情景中达到的效果，下面介绍AOP的实现，实现的大致流程如下图所示：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/4/11/4516c88b8e7714c30ebb7f839735e0bb~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

图3-1 Android AOP实现流程

关键有以下几点：

A、字节码插桩入口（图3-1 中1，3两个环节）。  
  我们知道Android程序从Java源代码到可执行的Apk包，中间有（但不止有）两个环节：

- javac：将源文件编译成class格式的文件
- dex：将class格式的文件汇总到dex格式的文件中

  我们要想对字节码进行修改，只需要在javac之后，dex之前对class文件进行字节码扫描，并按照一定规则进行过滤及修改就可以了，这样修改过后的字节码就会在后续的dex打包环节被打到apk中，这就是我们的插桩入口（更具体的后面还会详述）。

B、bytecode manipulate（上图3-1 中第二个环节），这个环节主要做：

1.  字节码扫描，并按照一定规则进行过滤出哪些类的class文件需要进行字节码修改
2.  对筛选出来的类进行字节码修改操作

  最后B步骤修改过字节码的class文件，将连同资源文件，一起打入Apk中，得到最终可以在Android平台可以运行的APP。

  下面分别就插桩入口和ASM字节码操作两个方面进行详述。

## 四、插桩入口

  如 第三章（AOP实现概述）所述，我们在Android 打包流程的javac之后，dex之前获得字节码插桩入口。

## 4.1 Android打包流程说明

  完整的Android 打包流程如下图所示：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/4/11/8210696bc6c5cb8b8cca907b4fa19148~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

图4-1 Android打包流程

  说明：

- 图4-1中“dex”节点，表示将class文件打包到dex文件的过程，其输入包括1.项目java源文件经过javac后生成的class文件以及2.第三方依赖的class文件两种，这些class文件都是我们进行字节码扫描以及修改的目标。
- 具体来说，进行图4-1中dex任务是一个叫dx.jar的jar包，存在于Android SDK的sdk/build-tools/22.0.1/lib/dx.jar目录中，通过类似 :

  ```
   java dx.jar com.android.dx.command.Main --dex --num-threads=4 —-output output.jar input.jar
  ```

  的命令，进行将class文件打包为dex文件的步骤。

- 从上面的演示命令可以看出，dex任务是启动一个java进程，执行dx.jar中com.android.dx.command.Main类(当然对于multidex的项目入口可能不是这个类，这个再说)的main()方法进行dex任务，具体完成class到dex转化的是这个方法：

```
 private static boolean processClass(String name,byte[] bytes) {
      //内容省略
}
```

  方法processClass的第二个参数是一个byte\[\]，这就是class文件的二进制数据（class文件是一种紧凑的8位字节的二进制流文件， 各个数据项按顺序紧密的从前向后排列， 相邻的项［包括字节码指令］之间没有间隙），我们就是通过对这个二进制数据进行扫描，按照一定规则过滤以及字节码修改达到第二部分所描述的AOP情景。

## 4.2 插桩入口

  那么我们怎么获得插桩入口呢？

### 入口一：transform api

  对于Android Gradle Plugin 版本在1.5.0及以上的情况，Google官方提供了transformapi用作字节码插桩的入口。此处的Android Gradle Plugin 版本指的是build.gradle dependencies的如下配置：

```
 compile 'com.android.tools.build:gradle:1.5.0'
```

此处1.5.0即为Android Build Gradle Plugin 版本。

关于transform api如何使用就不详细介绍了，

1.  可自行查看[API](http://tools.android.com/tech-docs/new-build-system/transform-api "http://tools.android.com/tech-docs/new-build-system/transform-api")，
2.  参考热修复项目Nuwa的[gradle插桩插件](http://blog.csdn.net/sbsujjbcy/article/details/50839263 "http://blog.csdn.net/sbsujjbcy/article/details/50839263")（使用transfrom api实现）

### 入口二：hook dx.jar

  那么对于Android Build Gradle Plugin 版本在1.5.0以下的情况呢？  
  下面我们介绍一种不依赖transform api而获得插桩入口的方法，暂且称为 hook dx.jar吧。

> 提示：具体使用可以考虑综合这两种方式，首先检查build环境是否支持transform api（反射检查类com.android.build.gradle.BaseExtension是否有registerTransform这个方法即可）然后决定使用哪种方式的插桩入口。

## 4.3 hook dx.jar获得插桩入口

  hook dx.jar 即是在图4-1中的dex步骤进行hook，具体来讲就是hook 4.1节介绍的dx.jar中com.android.dx.command.Main.processClass方法，将这个方法的字节码更改为：

```
 private static boolean processClass(String name,byte[] bytes) {

  bytes＝扫描并修改（bytes）；// Hook点

  //原有逻辑省略

}
```

注：这种方式获得插桩入口也可参见博客[《APM之原理篇》](http://blog.csdn.net/sgwhp/article/details/50239747 "http://blog.csdn.net/sgwhp/article/details/50239747")

  如何在一个标准的java进程（记得么？dex任务是启动一个java进程，执行dx.jar中com.android.dx.command.Main类的main()方法进行dex任务）中对特定方法进行字节码插桩？

  这就需要运用Java1.5引入的Instrumentation机制。

### java Instrumentation

  java Instrumentation指的是可以用独立于应用程序之外的代理（agent）程序来监测和协助运行在JVM上的应用程序。这种监测和协助包括但不限于获取JVM运行时状态，替换和修改类定义等。  
  Instrumentation 的最大作用就是类定义的动态改变和操作。

##### Java Instrumentation两种使用方式：

- 方式一(java 1.5+)：  
  开发者可以在一个普通 Java 程序（带有 main 函数的 Java 类）运行时，通过 **– javaagent** 参数指定一个**特定的 jar 文件(agent.jar)**（包含 Instrumentation 代理）来启动 Instrumentation 的代理程序。例如:

  ```
   java -javaagent agent.jar  dex.jar  com.android.dx.command.Main  --dex …........
  ```

  如此，则在目标main函数执行之前，执行agent jar包指定类的 premain方法 ：

  ```
   premain(String args, Instrumentation inst)
  ```

- 方式二(java 1.6+)：

  ```
   VirtualMachine.loadAgent(agent.jar)
  VirtualMachine vm = VirtualMachine.attach(pid);
  vm.loadAgent(jarFilePath, args);
  ```

  此时，将执行agent jar包指定类的 agentmain方法：

  ```
   agentmain(String args, Instrumentation inst)
  ```

##### 说明：

- 关于上述代码中出现的agent.jar?  
    这里的agent就是一个包含一些指定信息的jar包，就像OSGI的插件jar包一样，在jar包的META-INF/MANIFEST.MF中添加如下信息：

  ```javascript
   Manifest-Version: 1.0
  Agent-Class: XXXXX
  Premain-Class: XXXXX
  Can-Redefine-Classes: true
  Can-Retransform-Classes: true
  ```

    这个jar包就成了agent jar包，其中Agent-Class指向具有agentmain(String args, Instrumentation inst)方法的类，Premain-Class指向具有premain(String args, Instrumentation inst)的类。

- 关于premain(String args, Instrumentation inst)？  
    第二个参数，Instumentation 类有个方法

  ```
   addTransformer(ClassFileTransformer transformer,boolean canRetransform)
  ```

    而一旦为Instrumentation inst添加了ClassFileTransformer：

  ```
   ClassFileTransformer c=new ClassFileTransformer()
  inst.addTransformer(c,true);
  ```

    那么以后这个jvm进程中再有**任何类的加载定义**，都会出发此ClassFileTransformer的transform方法

  ```javascript
   byte[] transform(  ClassLoader loader,String className,Class classBeingRedefined,ProtectionDomain protectionDomain,byte[] classfileBuffer)throwsIllegalClassFormatException;
  ```

    其中，参数byte\[\] classfileBuffer是类的class文件数据，对它进行修改就可以达到在一个标准的java进程中对特定方法进行字节码插桩的目的。

### hook dx.jar获得插桩入口的完整流程

完整流程如下图所示：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/4/11/9c8c92eb60e2a618610a4ab15990cb79~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

图4-2 hook dx.jar流程图

注：apply plugin: 'bytecodeplugin'中的bytecodeplugin是我们用于字节码插桩的gradle插件

A. 通过任意方式（as界面内点击／命令gradle build等）都会启动图4-2所描述的build流程。

B. 通过Java Instrumentation机制，为获得插桩入口，对于apk build过程进行了两处插桩（即hook），图4-2中标红部分：

- 在build进程，对ProcessBuilder.start()方法进行插桩  
  ProcessBuilder类是J2SE 1.5在java.lang中新添加的一个新类，此类用于创建操作系统进程，它提供一种启动和管理进程的方法，start方法就是开始创建一个进程,对它进行插桩，使得通过下面方式启动dx.jar进程执行dex任务时：

  ```
   java  dex.jar  com.android.dx.command.Main  --dex …........
  ```

  增加参数**\-javaagent agent.jar**，使得dex进程也可以使用Java Instrumentation机制进行字节码插桩

- 在dex进程  
  对我们的目标方法com.android.dx.command.Main.processClasses进行字节码插入，从而实现打入apk的每一个项目中的类都按照我们制定的规则进行过滤及字节码修改。

C. 图4-2左侧build进程使用Instrumentation的方式时之前叙述过的VirtualMachine.loadAgent方式（方式二），dex进程中的方式则是-javaagent agent.jar方式（方式一）。

  由此，我们获得了进行字节码插桩的入口，下面我们就使用ASM库的API，对项目中的每一个类进行扫描，过滤，及字节码修改。

## 五、bytecode manipulation

  在这一部分我们以第二部分描述的情景二的应用场景为例，对View.OnClickListener的onClick方法进行字节码修改。在实践bytecode manipulation时需要一些关于字节码以及ASM的基础知识需要了解。因此本部分组织结构如下：

- 首先介绍一下我们用来操纵字节码的类库ASM
- 然后介绍一些关于字节码的基本知识
- 最后实践对View.OnClickListener的onClick方法进行bytecode manipulation

## 5.1 ASM库简要介绍

### 简介

  ASM是一个java字节码操纵框架，它能被用来动态生成类或者增强既有类的功能。ASM 可以直接产生二进制 class 文件，也可以在类被加载入 Java 虚拟机之前动态改变类行为。类似功能的工具库还有javassist，BCEL等。  
  那么为什么选择ASM呢？  
  ASM与同类工具库（这里以javassist为例）相比：

A. 较难使用，API非常底层，贴近字节码层面，需要字节码知识及虚拟机相关知识  
B. ASM更快更高效，Javassist实现机制中包括了反射，所以更慢。下表是使用不同工具库生成同一个类的耗时比较

| Framework | First time | Later times |
| --------- | ---------- | ----------- |
| Javassist | 257        | 5.2         |
| BCEL      | 473        | 5.5         |
| ASM       | 62.4       | 1.1         |

C. ASM库更加强大灵活，比如可以感知细到字节码指令层次（第二部分情景三中的场景）

总结起来，ASM虽然不太容易使用，但是功能强大效率高值得挑战。

关于ASM库的使用可以[参考手册](http://download.forge.objectweb.org/asm/asm4-guide.pdf "http://download.forge.objectweb.org/asm/asm4-guide.pdf")，下面对其API进行简要介绍：

### ASM API简介

  ASM（core api） 按照**visitor模式**按照**class文件结构**依次访问class文件的每一部分，有如下几个重要的visitor。

##### ClassVisitor

按照class文件格式，按次序访问类文件每一部分，如下：

```javascript
 public abstract class ClassVisitor {
public ClassVisitor(int api);
public ClassVisitor(int api, ClassVisitor cv);
public void visit(int version, int access, String name,
String signature, String superName, String[] interfaces); public void visitSource(String source, String debug);
public void visitOuterClass(String owner, String name, String desc); AnnotationVisitor visitAnnotation(String desc, boolean visible); public void visitAttribute(Attribute attr);
public void visitInnerClass(String name, String outerName,
String innerName, int access);
public FieldVisitor visitField(int access, String name, String desc,
String signature, Object value);
public MethodVisitor visitMethod(int access, String name, String desc,
String signature, String[] exceptions); void visitEnd();
}
```

与之对应的class文件格式为：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/4/11/75e37e6574107cd932f7c9f5408ce95a~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

图5-1 class文件格式

重点看ClassVisitor的如下几个方法：

- visit：按照图5-1中描述的 **class文件格式**，读出“class类名”（this_class的指向），“父类名”（super_class的指向），“实现的接口（数组）”（interfaces的指向）等信息
- visitField：访问字段，即访问图5-1 **class文件格式**中的“field_info”,访问字断的逻辑委托给另外一种visitor（FieldVisitor）
- visitField：访问方法，即访问图5-1 **class文件格式**中的“method_info”,访问方法的逻辑委托给另外一种visitor（MethodVisitor）

其他方法可参考前面推荐的ASM手册，下面介绍一下负责访问方法的MethodVisitor。

##### MethodVisitor

按以下次序访问一个方法：

```
 visitAnnotationDefault?
( visitAnnotation | visitParameterAnnotation | visitAttribute )*
  ( visitCode
    ( visitTryCatchBlock | visitLabel | visitFrame | visitXxxInsn | visitLocalVariable | visitLineNumber )*
  visitMaxs )?
visitEnd
```

注：上述出现的“＊”表示出现“0+”次，“？”表示出现“0/1”次。 含义可类比正则式元字符。

下面说明几个比较关键的visit方法：

- visitCode()：开始访问方法体内的代码
- visitTryCatchBlock：访问方法的try catch block
- visitLocalVariable：指令，访问局部变量表里面的某个局部变量（关于局部变量表后面会有介绍）
- visitXxxInsn：指令，表示class文件方法体里面的字节码指令（如：IADD，ICONST_0，ARETURN等等字节码指令），完整的字节码指令表可参考[维基百科](https://en.wikipedia.org/wiki/Java_bytecode "https://en.wikipedia.org/wiki/Java_bytecode")。
- visitLabel(Label label)：如果方法体中有跳转指令，字节码指令中会出现label，所谓label可以近似看成行号的标记（并不是），指示跳转指令将要跳转到哪里
- visitFrame：记录当前栈帧（栈帧结构将在后面有介绍）状态，用于Class文件加载时的校验
- visitMaxs：指定当前方法的栈帧中，局部变量表和操作数栈的大小。（java栈大小是javac之后就确定了的）

简单介绍了asm库后，由于使用ASM还需要对字节码有一定的了解，故在实践之前再介绍一些关于字节码的基础知识：

## 5.2 字节码基础

### 概念

关于字节码，有以下概念定义比较重要：

- 全限定名（Internal names）:  
  全限定名即为全类名中的“.”,换为“/”，举例：

  ```
   类android.widget.AdapterView.OnItemClickListener的全限定名为：
  android/widget/AdapterView$OnItemClickListener
  ```

- 描述符(descriptors)：  
  1.类型描述符，如下图所示：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/4/11/911ec9e39c20a31d67f716f3054e2e0f~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

图5-2 java类型描述符

如图5-2所示，在class文件中类型 boolean用“Z”描述，数组用“［”描述（多维数组可叠加）,那么我们最常见的自定义引用类型呢？“L全限定名；”.例如:  
Android中的android.view.View类，描述符为“Landroid/view/View;”

2.方法描述符的组织结构为：

```
 （参数类型描述符）返回值描述符
```

其中无返回值void用“V”代替,举例：

```
 方法boolean onGroupClick(ExpandableListView parent, View v, int groupPosition, long id) 　的描述符如下：
(Landroid/widget/ExpandableListView;Landroid/view/View;IJ)Z
```

### 执行引擎

jvm执行引擎用于执行字节码，如下图

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/4/11/b66aef77e68a64a2501753a714ce432c~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

图5-3 字节码执行引擎栈帧结构

  如图5-3所示，纵向来看有三个线程，其中每一个线程内部都有一个栈结构（即通常所说的“堆栈”中的虚拟机栈），栈中的每一个元素（一帧）称为一个栈帧（stack frame）。栈帧与我们写的方法一一对应，每个方法的调用／return对应线程中的一个栈帧的入栈／出栈。

  方法体中各种字节码指令的执行都在栈帧中完成，下面介绍下栈帧中两个比较重要的部分：

- 局部变量表：  
  故名思义，存储当前方法中的局部变量，包括方法的入参。值得注意的是局部变量表的第一个槽位存放的是this。还拿方法onGroupClick举例：

  ```
   boolean onGroupClick(ExpandableListView parent, View v, int groupPosition, long id)
  ```

  刚进入此方法时，局部变量表的槽位状态如下：

| Slot Number | value                     |
| ----------- | ------------------------- |
| 0           | this                      |
| 1           | ExpandableListView parent |
| 2           | View v                    |
| 3           | int groupPosition         |
| 4           | long id                   |

- 操作数栈：  
  字节码指令执行的工作台。下面用指令iadd（int类型加）执行时操作数栈的变化进行举例：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/4/11/847e94580c5717701b575fb3200c1a94~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

图5-4 执行iadd指令时操作数栈的状态变化

例如，方法体中有语句如下：

```
 1+1
```

- 在执行iadd之前需要先压两个“1”到操作数栈（因为iadd指令需要两个操作数，执行后产生一个操作数）
- 从常量池中（“1”为int常量）经过两个iconst_1后操作数栈的状态如图5-4中所示“操作数栈状态1”
- 执行iadd，将两个“1”弹出，交给ALU相加，把结果“2”入栈，操作数栈的状态如图5-4中所示“操作数栈状态2”

##

## 5.3 bytecode manipulation实践

我们来实践第二部分情景二描述的AOP，即修改所有View.OnClickListener的OnClick方法的字节码。流程如下图所示：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/4/11/6cd7bc3cb20606d956adc10a76d183b0~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

图5-5 AOP 控件点击实现流程

对上图中三个步骤的详细说明：

##### 步骤一：

ASM的ClassVisitor对所有类的class文件进行扫描，在visit方法中得到当前类实现了哪些接口，判断这些接口中是否包含全限定名为“android/view/View$OnClickListener”的接口。如果有，证明当前类是View.OnClickListener，进行步骤二，否则终止扫描；

##### 步骤二：

ClassVisitor每扫描到一个方法时，在visitMethod中进行如下判定：

1.  此方法的名字是否为"onClick"
2.  此方法的描述符是否为"(Landroid/view/View;)V"

如果全部判定通过，则证明本次扫描到的方法是View.OnClickListener的onClick方法，然后将  
将扫描逻辑交给MethodVisitor，进行字节码的修改（步骤三）。

##### 步骤三：修改onClick方法的字节码

假设待修改的onClick方法如下：

```
 public void onClick(View v) {
        System.out.println("test");//代表方法中原有的代码（逻辑）
}
```

修改之后需要变成：

```
 public void onClick(View v) {
        if(!Monitor.onViewClick(v)) {
            System.out.println("test");//代表方法中原有的代码（逻辑）
        }
    }
```

即:  
  进入方法之后先执行Monitor.onViewClick(v)（里面是数据收集逻辑），然后根据返回值决定是执行原有onClick方法内的逻辑，还是说直接返回。下面是修改之后onClick方法的字节码：

```
 public onClick(Landroid/view/View;)V
    ALOAD 1//插入的字节码，将index为1的局部变量（入参v）压入操作数栈
    INVOKESTATIC com/netease/lede/bytecode/monitor/Monitor.onViewClick (Landroid/view/View;)Z//插入的字节码，调用方法Monitor.onViewClick(v)，将返回值（true/false）压入操作数栈
    IFEQ L0//插入的字节码,如果操作数栈栈顶为0（if条件为false），则跳转到lable L0，执行原有逻辑
    RETURN//插入的字节码，上条指令判断不满足（即操作数栈栈顶为1（true）），直接返回
   L0
    LINENUMBER 11 L0
   FRAME SAME
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    LDC "test"
    INVOKEVIRTUAL java/io/PrintStream.println (Ljava/lang/String;)V
   L1
    LINENUMBER 12 L1
    RETURN
   L2
    LOCALVARIABLE this Lcom/netease/caipiao/datacollection/bytecode/ViewOnclickListener; L0 L2 0
    LOCALVARIABLE v Landroid/view/View; L0 L2 1
    MAXSTACK = 2//操作数栈最大为2
    MAXLOCALS = 2//局部变量表最大为2
```

如上图所示，插入的字节码主要是**前面四行（图中已经用注释的形式做了标记）**，图中的字节码指令可以参照下表：

| 字节码指令    | 说明                                         | 指令入参                       |
| ------------- | -------------------------------------------- | ------------------------------ |
| ALOAD         | 将引用类型的对象从局部变量表load到操作数栈   | 局部变量表index                |
| INVOKESTATIC  | 调用类方法（即静态方法）                     | 1.类全限定名 2.方法描述符      |
| INVOKEVIRTUAL | 调用对象方法                                 | 1.类全限定名 2.方法描述符      |
| IFEQ          | 检查操作数栈栈定位置是否为０                 | 跳转Lable（栈顶为0时跳转）     |
| RETURN        | 无返回值返回（操作数栈无弹栈操作）           |                                |
| IRETURN       | 返回int值（操作数栈将栈顶int值弹栈）         |                                |
| GETSTATIC     | 获取类字段（静态成员变量）                   | 1.类全限定名，2.字段类型描述符 |
| LDC           | 从常量池取int,float,String等常量到操作数栈顶 | 常量值                         |
| MAXSTACK      | 操作数栈最大容量（javac编译时确定）          |                                |
| MAXLOCALS     | 局部变量表最大容量(javac编译时确定)          |

具体插入的代码是字节码代码的前四行，逻辑比较简单：

1.  进入方法之后先执行Monitor.onViewClick(v)  
    ALOAD 1：将index为1的局部变量（入参v）压入操作数栈  
    INVOKESTATIC com/netease/lede/bytecode/monitor/Monitor.onViewClick (Landroid/view/View;)Z：  
    调用方法Monitor.onViewClick(v)(消耗ALOAD 1压入的操作数)，并将返回值（true/false）压入操作数栈
2.  根据返回值决定跳转  
    IFEQ L0：  
    如果操作数栈栈顶为0（if条件为false），则跳转到lable L0，执行原有逻辑  
    RETURN：上条指令判断不满足（即操作数栈栈顶为1（true）），直接返回

注：值得注意的是MAXSTACK，MAXLOCALS 两个值在javac生成的class文件就已经固定，即，栈内存大小已经确定（有别于堆内存可以在运行时动态申请/释放）。

如此，经过上述三个步骤，我们完成了第二部分情景二描述的AOP实践。

## 六、总结

文章写的比较长，下面对主要的几点进行总结：

  首先介绍了AOP的概念，已及在Android平台的主流框架，面对无埋点数据收集的需求，这些现有的都不太合适因此需要自己动手实现，  
  然后，简单列举了无埋点数据收集SDK中需要AOP的应用情景  
  最后介绍了实现的技术细节，主要有两点：

1.  通过hook dx.jar的方式获得插桩入口（可以和transfrom api配合使用）
2.  使用ASM库修改字节码，此部分简要介绍了关于字节码的一些基本概念以及执行引擎，最后以View.OnClickListener为例进行了实践。

