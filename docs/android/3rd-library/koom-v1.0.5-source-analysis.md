---
title: 【开源库剖析】KOOM V1.0.5 源码解析
url: https://juejin.cn/post/6991374693403983886
publishedTime: 2021-08-01T16:24:52+08:00
---

快手在2020年中旬开源了一个线上OOM监控上报的框架：KOOM，这里简单研究下。

项目地址：[github.com/KwaiAppTeam…](https://github.com/KwaiAppTeam/KOOM/tree/v1.0.5 "https://github.com/KwaiAppTeam/KOOM/tree/v1.0.5")

### 一、官方项目介绍

#### 1.1 描述：

KOOM是快手性能优化团队在处理移动端OOM问题的过程中沉淀出的一套完整解决方案。其中Android Java内存部分在LeakCanary的基础上进行了大量优化，解决了线上内存监控的性能问题，在不影响用户体验的前提下线上采集内存镜像并解析。从 2020 年春节后在快手主APP上线至今解决了大量OOM问题，其性能和稳定性经受住了海量用户与设备的考验，因此决定开源以回馈社区。

#### 1.2 特点：

- 比leakCanary更丰富的泄漏场景检测；
- 比leakCanary更好的检测性能；
- 功能全面的支持线上大规模部署的闭环监控系统；

#### 1.3 KOOM框架

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/db0225724c48429cb7d5e1d1bd77bacc~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

#### 1.4 快手KOOM核心流程包括：

- 配置下发决策；
- 监控内存状态；
- 采集内存镜像；
- 解析镜像文件(以下简称hprof)生成报告并上传；
- 问题聚合报警与分配跟进。

#### 1.5 泄漏检测触发机制优化：

泄漏检测触发机制leakCanary做法是GC过后对象WeakReference一直不被加入 ReferenceQueue，它可能存在内存泄漏。这个过程会主动触发GC做确认，可能会造成用户可感知的卡顿，而KOOM采用内存阈值监控来触发镜像采集，将对象是否泄漏的判断延迟到了解析时，阈值监控只要在子线程定期获取关注的几个内存指标即可，性能损耗很低。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/764abe9fe1f045a5801bb79ed4d1721e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

#### 1.6 heap dump优化：

传统方案会冻结应用几秒，KOOM会fork新进程来执行dump操作，对父进程的正常执行没有影响。暂停虚拟机需要调用虚拟机的art::Dbg::SuspendVM函数，谷歌从Android 7.0开始对调用系统库做了限制，快手自研了kwai-linker组件，通过caller address替换和dl_iterate_phdr解析绕过了这一限制。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7b9190adb6de41ce961e0543d912b472~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

随机采集线上真实用户的内存镜像，普通dump和fork子进程dump阻塞用户使用的耗时如下：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/22c2943d0b904c27a5d99dd09441db25~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

而从官方给出的测试数据来看，效果似乎是非常好的。

### 二、官方demo演示

这里就直接跑下官方提供的koom-demo

点击按钮，经过dump heap -> heap analysis -> report cache/koom/report/三个流程(heap analysis时间会比较长，但是完全不影响应用的正常操作),最终在应用的cache/koom/report里生成json报告：

```
cepheus:/data/data/com.kwai.koom.demo/cache/koom/report # ls
2020-12-08_15-23-32.json
```

模拟一个最简单的单例CommonUtils持有LeakActivity实例的内存泄漏，看下json最终上报的内容是个啥：

```
{
   "analysisDone":true,
   "classInfos":[
       {
           "className":"android.app.Activity",
           "instanceCount":4,
           "leakInstanceCount":3
       },
       {
           "className":"android.app.Fragment",
           "instanceCount":4,
           "leakInstanceCount":3
       },
       {
           "className":"android.graphics.Bitmap",
           "instanceCount":115,
           "leakInstanceCount":0
       },
       {
           "className":"libcore.util.NativeAllocationRegistry",
           "instanceCount":1513,
           "leakInstanceCount":0
       },
       {
           "className":"android.view.Window",
           "instanceCount":4,
           "leakInstanceCount":0
       }
   ],
   "gcPaths":[
       {
           "gcRoot":"Local variable in native code",
           "instanceCount":1,
           "leakReason":"Activity Leak",
           "path":[
               {
                   "declaredClass":"java.lang.Thread",
                   "reference":"android.os.HandlerThread.contextClassLoader",
                   "referenceType":"INSTANCE_FIELD"
               },
               {
                   "declaredClass":"java.lang.ClassLoader",
                   "reference":"dalvik.system.PathClassLoader.runtimeInternalObjects",
                   "referenceType":"INSTANCE_FIELD"
               },
               {
                   "declaredClass":"",
                   "reference":"java.lang.Object[]",
                   "referenceType":"ARRAY_ENTRY"
               },
               {
                   "declaredClass":"com.kwai.koom.demo.CommonUtils",
                   "reference":"com.kwai.koom.demo.CommonUtils.context",
                   "referenceType":"STATIC_FIELD"
               },
               {
                   "reference":"com.kwai.koom.demo.LeakActivity",
                   "referenceType":"instance"
               }
           ],
           "signature":"378fc01daea06b6bb679bd61725affd163d026a8"
       }
   ],
   "runningInfo":{
       "analysisReason":"RIGHT_NOW",
       "appVersion":"1.0",
       "buildModel":"MI 9 Transparent Edition",
       "currentPage":"LeakActivity",
       "dumpReason":"MANUAL_TRIGGER",
       "jvmMax":512,
       "jvmUsed":2,
       "koomVersion":1,
       "manufacture":"Xiaomi",
       "nowTime":"2020-12-08_16-07-34",
       "pss":32,
       "rss":123,
       "sdkInt":29,
       "threadCount":17,
       "usageSeconds":40,
       "vss":5674
   }
}
```

这里主要分三个部分：类信息、gc引用路径、运行基本信息。这里从gcPaths中能看出LeakActivity被CommonUtils持有了引用。

框架使用这里参考官方接入文档即可，这里不赘述： [github.com/KwaiAppTeam…](https://github.com/KwaiAppTeam/KOOM/blob/master/README.zh-CN.md "https://github.com/KwaiAppTeam/KOOM/blob/master/README.zh-CN.md")

### 三、框架解析

#### 3.1 类图

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/84677b58d32a47cba33b9e9dcf2dbaa7~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

#### 3.2 时序图

KOOM初始化流程

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/76be2f74b90848e38947c40e50a3996e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

KOOM执行初始化方法，10秒延迟之后会在threadHandler子线程中先通过check状态判断是否开始工作，工作内容是先检查是不是有未完成分析的文件，如果有就就触发解析，没有则监控内存。

heap dump流程

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b49f0a1a0a854343b95f5d53f95db7dc~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

HeapDumpTrigger

- startTrack：监控自动触发dump hprof操作。开启内存监控，子线程5s触发一次检测，看当前是否满足触发heap dump的条件。条件是由一系列阀值组织，这部分后面详细分析。满足阀值后会通过监听回调给HeapDumpTrigger去执行trigger。
- trigger:主动触发dump hprof操作。这里是fork子进程来处理的，这部分也到后面详细分析。dump完成之后通过监听回调触发HeapAnalysisTrigger.startTrack触发heap分析流程。

heap analysis流程

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f9a1fc8183cd4d878b1ababada8181ce~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

HeapAnalysisTrigger

- startTrack 根据策略触发hprof文件分析。
- trigger 直接触发hprof文件分析。由单独起进程的service来处理，工作内容主要分内存泄漏检测（activity/fragment/bitmap/window）和泄漏数据整理缓存为json文件以供上报。

### 四、核心源码解析

经过前面的分析，基本上对框架的使用和结构有了一个宏观了解，这部分就打算对一些实现细节进行简单分析。

#### 4.1 内存监控触发dump规则

这里主要是研究HeapMonitor中isTrigger规则，每隔5S都会循环判断该触发条件。

```javascript
com/kwai/koom/javaoom/monitor/HeapMonitor.java
@Override
public boolean isTrigger() {
  if (!started) {
   return false;
  }
  HeapStatus heapStatus = currentHeapStatus();
  if (heapStatus.isOverThreshold) {
   if (heapThreshold.ascending()) {
     if (lastHeapStatus == null || heapStatus.used >= lastHeapStatus.used) {
       currentTimes++;
     } else {
       currentTimes = 0;
     }
   } else {
     currentTimes++;
   }
  } else {
   currentTimes = 0;
  }
  lastHeapStatus = heapStatus;
  return currentTimes >= heapThreshold.overTimes();
}
private HeapStatus lastHeapStatus;
private HeapStatus currentHeapStatus() {
  HeapStatus heapStatus = new HeapStatus();
  heapStatus.max = Runtime.getRuntime().maxMemory();
  heapStatus.used = Runtime.getRuntime().totalMemory() - Runtime.getRuntime().freeMemory();
  heapStatus.isOverThreshold = 100.0f * heapStatus.used / heapStatus.max > heapThreshold.value();
  return heapStatus;
}

com/kwai/koom/javaoom/common/KConstants.java

public static class HeapThreshold {
  public static int VM_512_DEVICE = 510;
  public static int VM_256_DEVICE = 250;
  public static int VM_128_DEVICE = 128;
  public static float PERCENT_RATIO_IN_512_DEVICE = 80;
  public static float PERCENT_RATIO_IN_256_DEVICE = 85;
  public static float PERCENT_RATIO_IN_128_DEVICE = 90;

  public static float getDefaultPercentRation() {
   int maxMem = (int) (Runtime.getRuntime().maxMemory() / MB);
   if (maxMem >= VM_512_DEVICE) {
     return KConstants.HeapThreshold.PERCENT_RATIO_IN_512_DEVICE;
   } else if (maxMem >= VM_256_DEVICE) {
     return KConstants.HeapThreshold.PERCENT_RATIO_IN_256_DEVICE;
   } else if (maxMem >= VM_128_DEVICE) {
     return KConstants.HeapThreshold.PERCENT_RATIO_IN_128_DEVICE;
   }
   return KConstants.HeapThreshold.PERCENT_RATIO_IN_512_DEVICE;
  }

  public static int OVER_TIMES = 3;
  public static int POLL_INTERVAL = 5000;
}
```

这里就是针对不同内存大小做了不同的阀值比例：

- 应用内存>512M 80%
- 应用内存>256M 85%
- 应用内存>128M 90%
- 低于128M的默认按80%

应用已使用内存/最大内存超过该比例则会触发heapStatus.isOverThreshold。连续满足3次触发heap dump，但是这个过程会考虑内存增长性，3次范围内出现了使用内存下降或者使用内存/最大内存低于对应阀值了则清零。

因此规则总结为：3次满足>阀值条件且内存一直处于上升期才触发。这样能减少无效的dump。

#### 4.2 fork进程执行dump操作实现

这里先对比下目前市面上三方框架的主流实现方案：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/054b2e1d336244708b7f7d169f9dce0b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

**LeakCanaray、Matrix、Probe采用的方案：**  
直接执行Debug.dumpHprofData()，它执行过程会先挂起当前进程的所有线程，然后执行dump操作，生成完成hprof文件之后再唤醒所有线程。整个过程非常耗时，会带来比较明显的卡顿，因此这个痛点严重影响该功能带到线上环境。

**KOOM采用的方案:**  
主进程fork子进程来处理hprof dump操作，主进程本身只有在fork 子进程过程会短暂的suspend VM, 之后耗时阻塞均发生在子进程内，对主进程完全不产生影响。suspend VM本身过程时间非常短，从测试结果来看完全可以忽略不计

接下来详细分析下fork dump方案的实现： 目前项目中默认使用ForkJvmHeapDumper来执行dump。

```
com/kwai/koom/javaoom/dump/ForkJvmHeapDumper.java
@Override
public boolean dump(String path) {
  boolean dumpRes = false;
  try {
   int pid = trySuspendVMThenFork();//暂停虚拟机，copy-on-write fork子进程
   if (pid == 0) {//子进程中
     Debug.dumpHprofData(path);//dump hprof
     exitProcess();//_exit(0) 退出进程
   } else {//父进程中
     resumeVM();//resume当前虚拟机
     dumpRes = waitDumping(pid);//waitpid异步等待pid进程结束
   }
  } catch (Exception e) {
   e.printStackTrace();
  }
  return dumpRes;
}
```

**为什么需要先suspendVM然后再fork?**  
起初我理解主要是让fork前后的内存镜像保存一致性，但是对于内存泄漏来说这个造成的影响并不大，demo直接fork好像也没有什么问题，何况这块做了大量工作绕过Android N的限制去suspendVM肯定是有其必要性的。  
最终才发现，单线程是没问题的，因为线程已经停了，demo加多线程dump会卡在suspendVM。因此需要先suspendVM，再fork，最后resumeVM。

好，确认工作流之后，来尝试实现：

native层：

这里fork、waitPid 、exitProcess都比较简单，直接忽略，重点看vm相关操作：

正常操作就应该是：

```
void *libHandle = dlopen("libart.so", RTLD_NOW);//打开libart.so, 拿到文件操作句柄
void *suspendVM = dlsym(libHandle, LIBART_DBG_SUSPEND);//获取suspendVM方法引用
void *resumeVM = dlsym(libHandle, LIBART_DBG_RESUME);//获取resumeVM方法引用
dlclose(libHandle);//关闭libart.so文件操作句柄
```

这在Android N以下的版本这么操作是OK了，但是谷歌从Android 7.0开始对调用系统库做了限制，基于此前提，快手自研了kwai-linker组件，通过caller address替换和dl_iterate_phdr解析绕过了这一限制，[官方文档对限制说明](https://developer.android.google.cn/about/versions/nougat/android-7.0-changes "https://developer.android.google.cn/about/versions/nougat/android-7.0-changes")，那么接下来就分析下KOOM是如何绕过此限制的。

源码参考：Android 9.0

```javascript
/bionic/libdl/libdl.cpp
02__attribute__((__weak__))
103void* dlopen(const char* filename, int flag) {
104  const void* caller_addr = __builtin_return_address(0);//得到当前函数返回地址
105  return __loader_dlopen(filename, flag, caller_addr);
106}

/bionic/linker/dlfcn.cpp
152void* __loader_dlopen(const char* filename, int flags, const void* caller_addr) {
153  return dlopen_ext(filename, flags, nullptr, caller_addr);
154}

131static void* dlopen_ext(const char* filename,
132                        int flags,
133                        const android_dlextinfo* extinfo,
134                        const void* caller_addr) {
135  ScopedPthreadMutexLocker locker(&g_dl_mutex);
136  g_linker_logger.ResetState();
137  void* result = do_dlopen(filename, flags, extinfo, caller_addr);//执行do_dlopen
138  if (result == nullptr) {
139    __bionic_format_dlerror("dlopen failed", linker_get_error_buffer());
140    return nullptr;
141  }
142  return result;
143}

/bionic/linker/linker.cpp
2049void* do_dlopen(const char* name, int flags,
2050                const android_dlextinfo* extinfo,
2051                const void* caller_addr) {
2052  std::string trace_prefix = std::string("dlopen: ") + (name == nullptr ? "(nullptr)" : name);
2053  ScopedTrace trace(trace_prefix.c_str());
2054  ScopedTrace loading_trace((trace_prefix + " - loading and linking").c_str());
2055  soinfo* const caller = find_containing_library(caller_addr);
2056  android_namespace_t* ns = get_caller_namespace(caller);
...
2141  return nullptr;
2142}
```

这里dlopen最终执行是通过\_\_loader_dlopen，只不过默认会传入当前函数地址，这个地址其实就是做了caller address校验，如果检测出是三方地址则校验不通过，这里传入系统函数地址则能通过校验，例如dlerror。

那么KOOM的做法是：

**大于N小于Q的Android版本**

```typescript
using __loader_dlopen_fn = void *(*)(const char *filename, int flag, void *address);
void *handle = ::dlopen("libdl.so", RTLD_NOW);//打开libel.so
//这里直接调用其__loader_dlopen方法，它与dlopen区别是可以传入caller address
auto __loader_dlopen = reinterpret_cast<__loader_dlopen_fn>(::dlsym(handle,"__loader_dlopen"));
__loader_dlopen(lib_name, flags, (void *) dlerror);//传入dlerror系统函数地址，保证caller address校验通过，绕过Android N限制。
```

**Android Q及其以上的版本**

因为Q引入了runtime namespace，因此\_\_loader_dlopen返回的handle为nullptr

这里通过dl_iterate_phdr在当前进程中查询已加载的符合条件的动态库基对象地址。

```html
int DlFcn::dl_iterate_callback(dl_phdr_info
*info, size_t size, void *data) { auto target = reinterpret_cast<dl_iterate_data
  *
  >(data); if (info->dlpi_addr != 0 && strstr(info->dlpi_name,
  target->info_.dlpi_name)) { target->info_.dlpi_addr = info->dlpi_addr;
  target->info_.dlpi_phdr = info->dlpi_phdr; target->info_.dlpi_phnum =
  info->dlpi_phnum; // break iterate return 1; } // continue iterate return 0; }
  dl_iterate_data data{}; data.info_.dlpi_name = "libart.so";
  dl_iterate_phdr(dl_iterate_callback, &data); CHECKP(data.info_.dlpi_addr > 0)
  handle = __loader_dlopen(lib_name, flags, (void *)
  data.info_.dlpi_addr);</dl_iterate_data
>
```

这里dl_iterate_callback会回调当前进程所装载的每一个动态库，这里过滤出libart.so对应的地址：data.info\_.dlpi_addr，再通过\_\_loader_dlopen尝试打开libart.so。

附：

```javascript
struct dl_phdr_info {
  ElfW(Addr) dlpi_addr;//基对象地址
  const ElfW(Phdr)* dlpi_phdr;//指针数组
  ElfW(Half) dlpi_phnum;//
...
};
```

这便是快手自研的kwai-linker组件通过caller address替换和dl_iterate_phdr解析绕过Android 7.0对调用系统库做的限制的具体实现。也是fork dump方案的核心技术点。

#### 4.3 内存泄漏检测实现

内存泄漏检测核心代码在于SuspicionLeaksFinder.find

```html
public Pair<List<ApplicationLeak>, List<LibraryLeak>> find() {
  boolean indexed = buildIndex();
  if (!indexed) {
   return null;
  }
  initLeakDetectors();
  findLeaks();
  return findPath();
}
```

##### 4.3.1 buildIndex()

```typescript
private boolean buildIndex() {
  Hprof hprof = Hprof.Companion.open(hprofFile.file());
  //选择可以作为gcroot的类类型
  KClass<GcRoot>[] gcRoots = new KClass[]{
     Reflection.getOrCreateKotlinClass(GcRoot.JniGlobal.class),
     //Reflection.getOrCreateKotlinClass(GcRoot.JavaFrame.class),
     Reflection.getOrCreateKotlinClass(GcRoot.JniLocal.class),
     //Reflection.getOrCreateKotlinClass(GcRoot.MonitorUsed.class),
     Reflection.getOrCreateKotlinClass(GcRoot.NativeStack.class),
     Reflection.getOrCreateKotlinClass(GcRoot.StickyClass.class),
     Reflection.getOrCreateKotlinClass(GcRoot.ThreadBlock.class),
     Reflection.getOrCreateKotlinClass(GcRoot.ThreadObject.class),
     Reflection.getOrCreateKotlinClass(GcRoot.JniMonitor.class)};

  //解析hprof文件为HeapGraph对象
  heapGraph = HprofHeapGraph.Companion.indexHprof(hprof, null,
     kotlin.collections.SetsKt.setOf(gcRoots));
  return true;
}

fun indexHprof(
   hprof: Hprof,
   proguardMapping: ProguardMapping? = null,
   indexedGcRootTypes: Set<KClass<out GcRoot>> = setOf(
       JniGlobal::class,
       JavaFrame::class,
       JniLocal::class,
       MonitorUsed::class,
       NativeStack::class,
       StickyClass::class,
       ThreadBlock::class,
       ThreadObject::class,
       JniMonitor::class
   )
): HeapGraph {
  //确认对应的record的index
  val index = HprofInMemoryIndex.createReadingHprof(hprof, proguardMapping, indexedGcRootTypes)
  //HprofHeapGraph是HeapGraph的实现类
  return HprofHeapGraph(hprof, index)
}
```

HprofInMemoryIndex.createReadingHprof核心逻辑：读取hprof文件，将不同内容封装为不同的record，然后将record转为索引化的index封装，之后查找内容可以通过index去索引到。

HprofReader.readHprofRecords() 封装record

- LoadClassRecord
- InstanceSkipContentRecord
- ObjectArraySkipContentRecord
- PrimitiveArraySkipContentRecord

HprofInMemoryIndex.onHprofRecord() 封装index：

- classIndex
- instanceIndex
- objectArrayIndex
- primitiveArrayIndex

```typescript
class HprofHeapGraph internal constructor(
   private val hprof: Hprof,
   private val index: HprofInMemoryIndex
) : HeapGraph {
...
  override val gcRoots: List<GcRoot>
   get() = index.gcRoots()
  override val objects: Sequence<HeapObject>
   get() {
     return index.indexedObjectSequence().map {wrapIndexedObject(it.second, it.first)}
   }
  override val classes: Sequence<HeapClass>
   get() {
     return index.indexedClassSequence().map {val objectId = it.first
           val indexedObject = it.second
           HeapClass(this, indexedObject, objectId)
         }
   }
  override val instances: Sequence<HeapInstance>
   get() {
     return index.indexedInstanceSequence().map {val objectId = it.first
           val indexedObject = it.second
           val isPrimitiveWrapper = index.primitiveWrapperTypes.contains(indexedObject.classId)
           HeapInstance(this, indexedObject, objectId, isPrimitiveWrapper)
         }
   }

  override val objectArrays: Sequence<HeapObjectArray>
   get() = index.indexedObjectArraySequence().map {val objectId = it.first
     val indexedObject = it.second
     val isPrimitiveWrapper = index.primitiveWrapperTypes.contains(indexedObject.arrayClassId)
     HeapObjectArray(this, indexedObject, objectId, isPrimitiveWrapper)
   }

  override val primitiveArrays: Sequence<HeapPrimitiveArray>
   get() = index.indexedPrimitiveArraySequence().map {val objectId = it.first
     val indexedObject = it.second
     HeapPrimitiveArray(this, indexedObject, objectId)
   }
```

Hprof 经过层层转换最终封装为HprofHeapGraph。

简而言之，这部分功能主要是将Hrpof文件按照扫描的格式解析为结构化的索引关系图，索引化后的内容封装为HprofHeapGraph，由它去通过对应的起始索引去定位每类数据。没有细抠这部分的实现细节，实现这个功能的库之前是squere的HAHA，现在改为shark,但是提供的功能大同小异。

##### 4.3.2 initLeakDetectors() 与findLeaks()

初始化泄漏检测者：

```
private void initLeakDetectors() {
  addDetector(new ActivityLeakDetector(heapGraph));
  addDetector(new FragmentLeakDetector(heapGraph));
  addDetector(new BitmapLeakDetector(heapGraph));
  addDetector(new NativeAllocationRegistryLeakDetector(heapGraph));
  addDetector(new WindowLeakDetector(heapGraph));
  ClassHierarchyFetcher.initComputeGenerations(computeGenerations);
  leakReasonTable = new HashMap<>();
}
```

初始化各类型泄漏的检测者，主要包含Activity、Fragment、Bitmap+NativeAllocationRegistry、window的泄漏检测。

其次是梳理以上几类对象类继承关系串，检测覆盖到他们的子类。

```javascript
public void findLeaks() {
  KLog.i(TAG, "start find leaks");
  //从HprofHeapGraph中获取所有instance
  Sequence<HeapObject.HeapInstance> instances = heapGraph.getInstances();
  Iterator<HeapObject.HeapInstance> instanceIterator = instances.iterator();
  while (instanceIterator.hasNext()) {
    HeapObject.HeapInstance instance = instanceIterator.next();
   if (instance.isPrimitiveWrapper()) {
      continue;
   }
    ClassHierarchyFetcher.process(instance.getInstanceClassId(),
       instance.getInstanceClass().getClassHierarchy());
   for (LeakDetector leakDetector : leakDetectors) {
      //是检测对象的子类&满足对应泄漏条件
      if (leakDetector.isSubClass(instance.getInstanceClassId())
          && leakDetector.isLeak(instance)) {
        ClassCounter classCounter = leakDetector.instanceCount();
       if (classCounter.leakInstancesCount <=
            SAME_CLASS_LEAK_OBJECT_GC_PATH_THRESHOLD) {
          leakingObjects.add(instance.getObjectId());
         leakReasonTable.put(instance.getObjectId(), leakDetector.leakReason());
       }
      }
    }
  }

  //关注class和对应instance数量，加入json
  HeapAnalyzeReporter.addClassInfo(leakDetectors);
  findPrimitiveArrayLeaks();
  findObjectArrayLeaks();
}
```

这里重点看看各类型对象是如何判断泄漏的：

```
ActivityLeakDetector:

private static final String ACTIVITY_CLASS_NAME = "android.app.Activity";
private static final String FINISHED_FIELD_NAME = "mFinished";
private static final String DESTROYED_FIELD_NAME = "mDestroyed";
public boolean isLeak(HeapObject.HeapInstance instance) {
  activityCounter.instancesCount++;
  HeapField destroyField = instance.get(ACTIVITY_CLASS_NAME, DESTROYED_FIELD_NAME);
  HeapField finishedField = instance.get(ACTIVITY_CLASS_NAME, FINISHED_FIELD_NAME);
  assert destroyField != null;
  assert finishedField != null;
  boolean abnormal = destroyField.getValue().getAsBoolean() == null
     || finishedField.getValue().getAsBoolean() == null;
  if (abnormal) {
   return false;
  }
  boolean leak = destroyField.getValue().getAsBoolean()
      || finishedField.getValue().getAsBoolean();
  if (leak) {
    activityCounter.leakInstancesCount++;
  }
  return leak;
}
```

mDestroyed和mFinish字段为true，但是实例还存在的Activity是疑似泄漏对象。

```javascript
FragmentLeakDetector:

  private static final String NATIVE_FRAGMENT_CLASS_NAME = "android.app.Fragment";
// native android Fragment, deprecated as of API 28.
  private static final String SUPPORT_FRAGMENT_CLASS_NAME = "android.support.v4.app.Fragment";
// pre-androidx, support library version of the Fragment implementation.
  private static final String ANDROIDX_FRAGMENT_CLASS_NAME = "androidx.fragment.app.Fragment";
// androidx version of the Fragment implementation
private static final String FRAGMENT_MANAGER_FIELD_NAME = "mFragmentManager”;
private static final String FRAGMENT_MCALLED_FIELD_NAME = "mCalled”;//Used to verify that subclasses call through to super class.

public FragmentLeakDetector(HeapGraph heapGraph) {
  HeapObject.HeapClass fragmentHeapClass =
      heapGraph.findClassByName(ANDROIDX_FRAGMENT_CLASS_NAME);
  fragmentClassName = ANDROIDX_FRAGMENT_CLASS_NAME;
  if (fragmentHeapClass == null) {
    fragmentHeapClass = heapGraph.findClassByName(NATIVE_FRAGMENT_CLASS_NAME);
   fragmentClassName = NATIVE_FRAGMENT_CLASS_NAME;
  }

  if (fragmentHeapClass == null) {
    fragmentHeapClass = heapGraph.findClassByName(SUPPORT_FRAGMENT_CLASS_NAME);
   fragmentClassName = SUPPORT_FRAGMENT_CLASS_NAME;
  }

  assert fragmentHeapClass != null;
  fragmentClassId = fragmentHeapClass.getObjectId();
  fragmentCounter = new ClassCounter();
}

public boolean isLeak(HeapObject.HeapInstance instance) {
  if (VERBOSE_LOG) {
    KLog.i(TAG, "run isLeak");
  }
  fragmentCounter.instancesCount++;
  boolean leak = false;
  HeapField fragmentManager = instance.get(fragmentClassName, FRAGMENT_MANAGER_FIELD_NAME);
  if (fragmentManager != null && fragmentManager.getValue().getAsObject() == null) {
    HeapField mCalledField = instance.get(fragmentClassName, FRAGMENT_MCALLED_FIELD_NAME);
   boolean abnormal = mCalledField == null || mCalledField.getValue().getAsBoolean() == null;
   if (abnormal) {
      KLog.e(TAG, "ABNORMAL mCalledField is null");
     return false;
   }
    leak = mCalledField.getValue().getAsBoolean();
   if (leak) {
      if (VERBOSE_LOG) {
        KLog.e(TAG, "fragment leak : " + instance.getInstanceClassName());
     }
      fragmentCounter.leakInstancesCount++;
   }
  }
  return leak;
}
```

这里分了三种fragment:

- android.app.Fragment
- android.support.v4.app.Fragment
- androidx.fragment.app.Fragment

对应的FragmentManager实例为null（这表示fragment被remove了）且满足对应的mCalled为true,即非perform状态，而是对应生命周期被回调状态（onDestroy），但是实例还存在的Fragment是疑似泄漏对象。

```
BitmapLeakDetector

private static final String BITMAP_CLASS_NAME = "android.graphics.Bitmap”;
public boolean isLeak(HeapObject.HeapInstance instance) {
  if (VERBOSE_LOG) {
    KLog.i(TAG, "run isLeak");
  }
  bitmapCounter.instancesCount++;
  HeapField fieldWidth = instance.get(BITMAP_CLASS_NAME, "mWidth");
  HeapField fieldHeight = instance.get(BITMAP_CLASS_NAME, "mHeight");
  assert fieldHeight != null;
  assert fieldWidth != null;
  boolean abnormal = fieldHeight.getValue().getAsInt() == null
     || fieldWidth.getValue().getAsInt() == null;
  if (abnormal) {
    KLog.e(TAG, "ABNORMAL fieldWidth or fieldHeight is null");
   return false;
  }

  int width = fieldWidth.getValue().getAsInt();
  int height = fieldHeight.getValue().getAsInt();
  boolean suspicionLeak = width * height >= KConstants.BitmapThreshold.DEFAULT_BIG_BITMAP;
  if (suspicionLeak) {
    KLog.e(TAG, "bitmap leak : " + instance.getInstanceClassName() + " " +
        "width:" + width + " height:" + height);
   bitmapCounter.leakInstancesCount++;
  }
  return suspicionLeak;
}

```

这里是针对Bitmap size做判断，超过768\*1366这个size的认为泄漏。

另外，NativeAllocationRegistryLeakDetector和WindowLeakDetector两类还没做具体泄漏判断规则，不参与对象泄漏检测，只是做了统计。

总结： 整体看下来，KOOM有两个值得借鉴的点：

- 1.触发内存泄漏检测，常规是watcher activity/fragment的onDestroy，而KOOM是定期轮询查看当前内存是否到达阀值；
- 2.dump hprof，常规是对应进程dump，而KOOM是fork进程dump。

参考：  
[快手自研OOM解决方案KOOM今日宣布开源](https://baijiahao.baidu.com/s?id=1674693121890730020&wfr=spider&for=pc "https://baijiahao.baidu.com/s?id=1674693121890730020&wfr=spider&for=pc")  
[Android9.0 hook dlopen问题/如何hook dlopen相关函数](https://links.jianshu.com/go?to=https%3A%2F%2Fbbs.pediy.com%2Fthread-257022.htm "https://links.jianshu.com/go?to=https%3A%2F%2Fbbs.pediy.com%2Fthread-257022.htm")  
[dl_iterate_phdr](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Flinuxheik%2Farticle%2Fdetails%2F17502433 "https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Flinuxheik%2Farticle%2Fdetails%2F17502433")

