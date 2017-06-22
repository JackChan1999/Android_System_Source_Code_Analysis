前面我们分析了Dalvik虚拟机堆的创建过程，以及[Java](http://lib.csdn.net/base/java)对象在堆上的分配过程。这些知识都是理解Dalvik虚拟机垃圾收集过程的基础。垃圾收集是一个复杂的过程，它要将那些不再被引用的对象进行回收。一方面要求Dalvik虚拟机能够标记出哪些对象是不再被引用的。另一方面要求Dalvik虚拟机尽快地回收内存，避免应用程序长时间停顿。本文就将详细分析Dalvik虚拟机是如何解决上述问题完成垃圾收集过程的。

老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)，欢迎关注！

《[Android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

Dalvik虚拟机使用Mark-Sweep[算法](http://lib.csdn.net/base/datastructure)来进行垃圾收集。顾名思义，Mark-Sweep算法就是为Mark和Sweep两个阶段进行垃圾回收。其中，Mark阶段从根集（Root Set）开始，递归地标记出当前所有被引用的对象，而Sweep阶段负责回收那些没有被引用的对象。在分析Dalvik虚拟机使用的Mark-Sweep算法之前，我们先来了解一下什么情况下会触发GC。

Dalvik虚拟机在三种情况下会触发四种类型的GC。每一种类型GC使用一个GcSpec结构体来描述，它的定义如下所示：

```c
struct GcSpec {  
  /* If true, only the application heap is threatened. */  
  bool isPartial;  
  /* If true, the trace is run concurrently with the mutator. */  
  bool isConcurrent;  
  /* Toggles for the soft reference clearing policy. */  
  bool doPreserve;  
  /* A name for this garbage collection mode. */  
  const char *reason;  
};  
```
这个结构体定义在文件dalvik/vm/alloc/Heap.h中。

GcSpec结构体的各个成员变量的含义如下所示：

| 成员变量         | 描述                                       |
| :----------- | :--------------------------------------- |
| isPartial    | 为true时，表示仅仅回收Active堆的垃圾；为false时，表示同时回收Active堆和Zygote堆的垃圾。 |
| isConcurrent | 为true时，表示执行并行GC；为false时，表示执行非并行GC。       |
| doPreserve   | 为true时，表示在执行GC的过程中，不回收软引用引用的对象；为false时，表示在执行GC的过程中，回收软引用引用的对象。 |
| reason       | 一个描述性的字符串。                               |

Davlik虚拟机定义了四种类的GC，如下所示：

```c
/* Not enough space for an "ordinary" Object to be allocated. */  
extern const GcSpec *GC_FOR_MALLOC;  
  
/* Automatic GC triggered by exceeding a heap occupancy threshold. */  
extern const GcSpec *GC_CONCURRENT;  
  
/* Explicit GC via Runtime.gc(), VMRuntime.gc(), or SIGUSR1. */  
extern const GcSpec *GC_EXPLICIT;  
  
/* Final attempt to reclaim memory before throwing an OOM. */  
extern const GcSpec *GC_BEFORE_OOM;  
```
这四个全局变量声明在文件dalvik/vm/alloc/Heap.h中。

它们的含义如下所示：

| 全局变量          | 描述                                       |
| :------------ | :--------------------------------------- |
| GC_FOR_MALLOC | 表示是在堆上分配对象时内存不足触发的GC。                    |
| GC_CONCURRENT | 表示是在已分配内存达到一定量之后触发的GC。                   |
| GC_EXPLICIT   | 表示是应用程序调用System.gc、VMRuntime.gc接口或者收到SIGUSR1信号时触发的GC。 |
| GC_BEFORE_OOM | 表示是在准备抛OOM异常之前进行的最后努力而触发的GC。             |

实际上，GC_FOR_MALLOC、GC_CONCURRENT和GC_BEFORE_OOM三种类型的GC都是在分配对象的过程触发的。

在前面[Dalvik虚拟机为新创建对象分配内存的过程](http://blog.csdn.net/luoshengyang/article/details/41688319)分析一文，我们提到，Dalvik虚拟机在Java堆上分配对象的时候，在碰到分配失败的情况，会尝试调用函数gcForMalloc进行垃圾回收。

函数gcForMalloc的实现如下所示：

```c
static void gcForMalloc(bool clearSoftReferences)  
{  
    ......  
  
    const GcSpec *spec = clearSoftReferences ? GC_BEFORE_OOM : GC_FOR_MALLOC;  
    dvmCollectGarbageInternal(spec);  
}  
```
这个函数定义在文件dalvik/vm/alloc/Heap.cpp中。

参数clearSOftRefereces表示是否要对软引用引用的对象进行回收。如果要对软引用引用的对象进行回收，那么就表明当前内存是非常紧张的了，因此，这时候执行的就是GC_BEFORE_OOM类型的GC。否则的话，执行的就是GC_FOR_MALLOC类型的GC。它们都是通过调用函数dvmCollectGarbageInternal来执行的。

在前面[Dalvik虚拟机为新创建对象分配内存的过程分析](http://blog.csdn.net/luoshengyang/article/details/41688319)一文，我们也提到，当Dalvik虚拟机成功地在堆上分配一个对象之后，会检查一下当前分配的内存是否超出一个阀值，如下所示：

```c
void* dvmHeapSourceAlloc(size_t n)    
{    
    ......  
  
    HeapSource *hs = gHs;    
    Heap* heap = hs2heap(hs);    
    if (heap->bytesAllocated + n > hs->softLimit) {    
......    
return NULL;    
    }    
    void* ptr;    
    if (gDvm.lowMemoryMode) {    
......    
ptr = mspace_malloc(heap->msp, n);    
......  
    } else {    
ptr = mspace_calloc(heap->msp, 1, n);    
......   
    }    
  
    countAllocation(heap, ptr);    
    ......    
  
    if (heap->bytesAllocated > heap->concurrentStartBytes) {    
......    
dvmSignalCond(&gHs->gcThreadCond);    
    }    
    return ptr;    
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapSource.cpp中。

函数dvmHeapSourceAlloc成功地在Active堆上分配到一个对象之后，就会检查Active堆当前已经分配的内存（heap->bytesAllocated）是否大于预设的阀值（heap->concurrentStartBytes）。如果大于，那么就会通过条件变量gHs->gcThreadCond唤醒GC线程进行垃圾回收。预设的阀值（heap->concurrentStartBytes）是一个比指定的堆最小空闲内存小128K的数值。也就是说，当堆的空闲内不足时，就会触发GC_CONCURRENT类型的GC。

GC线程是Dalvik虚拟机启动的过程中创建的，它的执行体函数是gcDaemonThread，实现如下所示：

```c
static void *gcDaemonThread(void* arg)  
{  
    dvmChangeStatus(NULL, THREAD_VMWAIT);  
    dvmLockMutex(&gHs->gcThreadMutex);  
    while (gHs->gcThreadShutdown != true) {  
bool trim = false;  
if (gHs->gcThreadTrimNeeded) {  
    int result = dvmRelativeCondWait(&gHs->gcThreadCond, &gHs->gcThreadMutex,  
            HEAP_TRIM_IDLE_TIME_MS, 0);  
    if (result == ETIMEDOUT) {  
        /* Timed out waiting for a GC request, schedule a heap trim. */  
        trim = true;  
    }  
} else {  
    dvmWaitCond(&gHs->gcThreadCond, &gHs->gcThreadMutex);  
}  
  
......  
  
dvmLockHeap();  
  
if (!gDvm.gcHeap->gcRunning) {  
    dvmChangeStatus(NULL, THREAD_RUNNING);  
    if (trim) {  
        trimHeaps();  
        gHs->gcThreadTrimNeeded = false;  
    } else {  
        dvmCollectGarbageInternal(GC_CONCURRENT);  
        gHs->gcThreadTrimNeeded = true;  
    }  
    dvmChangeStatus(NULL, THREAD_VMWAIT);  
}  
dvmUnlockHeap();  
    }  
    dvmChangeStatus(NULL, THREAD_RUNNING);  
    return NULL;  
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapSource.cpp中。

GC线程平时没事的时候，就在条件变量gHs->gcThreadCond上进行等待HEAP_TRIM_IDLE_TIME_MS毫秒（5000毫秒）。如果在HEAP_TRIM_IDLE_TIME_MS毫秒内，都没有得到执行GC的通知，那么它就调用函数trimHeaps对Java堆进行裁剪，以便可以将堆上的一些没有使用到的内存交还给内核。函数trimHeaps的实现可以参考前面[Dalvik虚拟机Java堆创建过程分析](http://blog.csdn.net/luoshengyang/article/details/41581063)一文。否则的话，就会调用函数dvmCollectGarbageInternal进行类型为GC_CONCURRENT的GC。

注意，函数gcDaemonThread在调用函数dvmCollectGarbageInternal进行类型为GC_CONCURRENT的GC之前，会先调用函数dvmLockHeap来锁定堆的。等到GC执行完毕，再调用函数dvmUnlockHeap来解除对堆的锁定。这与函数gcForMalloc调用函数dvmCollectGarbageInternal进行类型为GC_FOR_MALLOC和GC_CONCURRENT的GC是一样的。只不过，对堆进行锁定和解锁的操作是在调用堆栈上的函数dvmMalloc进行的，具体可以参考前面[Dalvik虚拟机为新创建对象分配内存的过程分析](http://blog.csdn.net/luoshengyang/article/details/41688319)一文。

当应用程序调用System.gc、VMRuntime.gc接口，或者接收到SIGUSR1信号时，最终会调用到函数dvmCollectGarbage，它的实现如下所示：

```c
void dvmCollectGarbage()  
{  
    if (gDvm.disableExplicitGc) {  
return;  
    }  
    dvmLockHeap();  
    dvmWaitForConcurrentGcToComplete();  
    dvmCollectGarbageInternal(GC_EXPLICIT);  
    dvmUnlockHeap();  
}  
```
这个函数定义在文件dalvik/vm/alloc/Alloc.cpp中。

如果Davik虚拟机在启动的时候，通过-XX:+DisableExplicitGC选项禁用了显式GC，那么函数dvmCollectGarbage什么也不做就返回了。这意味着Dalvik虚拟机可能会不支持应用程序显式的GC请求。

一旦Dalvik虚拟机支持显式GC，那么函数dvmCollectGarbage就会先锁定堆，并且等待有可能正在执行的GC_CONCURRENT类型的GC完成之后，再调用函数dvmCollectGarbageInternal进行类型为GC_EXPLICIT的GC。

以上就是三种情况下会触发四种类型的GC。有了这个背景知识之后 ，我们接下来就开始分析Dalvik虚拟机使用的Mark-Sweep算法，整个过程如图1所示：

![img](http://img.blog.csdn.net/20141210011257140?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图1 Dalvik虚拟机垃圾收集过程

Dalvik虚拟机支持非并行和并行两种GC。在图1中，左边是非并行GC的执行过程，而右边是并行GC的执行过程。它们的总体流程是相似的，主要差别在于前者在执行的过程中一直是挂起非GC线程的，而后者是有条件地挂起非GC线程。

由前面的分析可以知道，无论是并行GC，还是非并行GC，它们都是通过函数dvmCollectGarbageInternal来执行的。函数dvmCollectGarbageInternal的实现如下所示：

```c
void dvmCollectGarbageInternal(const GcSpec* spec)  
{  
    ......  
  
    if (gcHeap->gcRunning) {  
......  
return;  
    }  
    ......  
  
    gcHeap->gcRunning = true;  
    ......  
  
    dvmSuspendAllThreads(SUSPEND_FOR_GC);   
    ......  
  
    if (!dvmHeapBeginMarkStep(spec->isPartial)) {  
......  
dvmAbort();  
    }  
  
    dvmHeapMarkRootSet();  
    ......  
  
    if (spec->isConcurrent) {  
......  
dvmClearCardTable();  
dvmUnlockHeap();  
dvmResumeAllThreads(SUSPEND_FOR_GC);  
......  
    }  
  
    dvmHeapScanMarkedObjects();  
  
    if (spec->isConcurrent) {  
......  
dvmLockHeap();  
......  
dvmSuspendAllThreads(SUSPEND_FOR_GC);  
......  
dvmHeapReMarkRootSet();  
......  
dvmHeapReScanMarkedObjects();  
    }  
  
    dvmHeapProcessReferences(&gcHeap->softReferences,  
                     spec->doPreserve == false,  
                     &gcHeap->weakReferences,  
                     &gcHeap->finalizerReferences,  
                     &gcHeap->phantomReferences);  
    ......  
  
    dvmHeapSweepSystemWeaks();  
    ......  
     
    dvmHeapSourceSwapBitmaps();  
    .......  
  
    if (spec->isConcurrent) {  
dvmUnlockHeap();  
dvmResumeAllThreads(SUSPEND_FOR_GC);  
......  
    }  
    dvmHeapSweepUnmarkedObjects(spec->isPartial, spec->isConcurrent,  
                        &numObjectsFreed, &numBytesFreed);  
    ......  
    dvmHeapFinishMarkStep();  
    if (spec->isConcurrent) {  
dvmLockHeap();  
    }  
    ......  
  
    dvmHeapSourceGrowForUtilization();  
    ......  
  
    gcHeap->gcRunning = false;  
    ......  
  
    if (spec->isConcurrent) {  
......  
dvmBroadcastCond(&gDvm.gcHeapCond);  
    }  
  
    if (!spec->isConcurrent) {  
dvmResumeAllThreads(SUSPEND_FOR_GC);  
......  
    }  
  
    .......  
    dvmEnqueueClearedReferences(&gDvm.gcHeap->clearedReferences);  
   
    ......  
}  
```
这个函数定义在文件dalvik/vm/alloc/Heap.cpp中。

前面提到，在调用函数dvmCollectGarbageInternal之前，堆是已经被锁定了的，因此这时候任何需要堆上分配对象的线程都会被挂起。注意，这不会影响到那些不需要在堆上分配对象的线程。

在图1中显示的GC流程中，除了第一步的Lock Heap和最后一步的Unlock Heap，都对应于函数dvmCollectGarbageInternal的实现，它的执行逻辑如下所示：

第1步到第3步用于并行和非并行GC：

调用函数dvmSuspendAllThreads挂起所有的线程，以免它们干扰GC。

调用函数dvmHeapBeginMarkStep初始化Mark Stack，并且设定好GC范围。关于Mark Stack的介绍，可以参考前面[Dalvik虚拟机垃圾收集机制简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/41338251)一文。

调用函数dvmHeapMarkRootSet标记根集对象。

第4到第6步用于并行GC：

调用函数dvmClearCardTable清理Card Table。关于Card Table的介绍，可以参考前面[Dalvik虚拟机垃圾收集机制简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/41338251)一文。因为接下来我们将会唤醒第1步挂起的线程。并且使用这个Card Table来记录那些在GC过程中被修改的对象。

调用函数dvmUnlock解锁堆。这个是针对调用函数dvmCollectGarbageInternal执行GC前的堆锁定操作。

调用函数dvmResumeAllThreads唤醒第1步挂起的线程。

第7步用于并行和非并行GC：

调用函数dvmHeapScanMarkedObjects从第3步获得的根集对象开始，归递标记所有被根集对象引用的对象。

第8步到第11步用于并行GC：

调用函数dvmLockHeap重新锁定堆。这个是针对前面第5步的操作。

调用函数dvmSuspendAllThreads重新挂起所有的线程。这个是针对前面第6步的操作。

10. 调用函数dvmHeapReMarkRootSet更新根集对象。因为有可能在第4步到第6步的执行过程中，有线程创建了新的根集对象。

11. 调用函数dvmHeapReScanMarkedObjects归递标记那些在第4步到第6步的执行过程中被修改的对象。这些对象记录在Card Table中。

第12步到第14步用于并行和非并行GC：

12. 调用函数dvmHeapProcessReferences处理那些被软引用（Soft Reference）、弱引用（Weak Reference）和影子引用（Phantom Reference）引用的对象，以及重写了finalize方法的对象。这些对象都是需要特殊处理的。

13. 调用函数dvmHeapSweepSystemWeaks回收系统内部使用的那些被弱引用引用的对象。

14. 调用函数dvmHeapSourceSwapBitmaps交换Live Bitmap和Mark Bitmap。执行了前面的13步之后，所有还被引用的对象在Mark Bitmap中的bit都被设置为1。而Live Bitmap记录的是当前GC前还被引用着的对象。通过交换这两个Bitmap，就可以使得当前GC完成之后，使得Live Bitmap记录的是下次GC前还被引用着的对象。

第15步和第16步用于并行GC：

15. 调用函数dvmUnlock解锁堆。这个是针对前面第8步的操作。

16. 调用函数dvmResumeAllThreads唤醒第9步挂起的线程。

第17步和第18步用于并行和非并行GC：

17. 调用函数dvmHeapSweepUnmarkedObjects回收那些没有被引用的对象。没有被引用的对象就是那些在执行第14步之前，在Live Bitmap中的bit设置为1，但是在Mark Bitmap中的bit设置为0的对象。

18. 调用函数dvmHeapFinishMarkStep重置Mark Bitmap以及Mark Stack。这个是针对前面第2步的操作。

第19步用于并行GC：

19. 调用函数dvmLockHeap重新锁定堆。这个是针对前面第15步的操作。

第20步用于并行和非并行GC：

20. 调用函数dvmHeapSourceGrowForUtilization根据设置的堆目标利用率调整堆的大小。

第21步用于并行GC：      

21. 调用函数dvmBroadcastCond唤醒那些等待GC执行完成再在堆上分配对象的线程。

第22步用于非并行GC：

22. 调用函数dvmResumeAllThreads唤醒第1步挂起的线程。

第23步用到并行和非并行GC：

23. 调用函数dvmEnqueueClearedReferences将那些目标对象已经被回收了的引用对象增加到相应的Java队列中去，以便应用程序可以知道哪些引用引用的对象已经被回收了。

以上就是并行和非并行GC的执行总体流程，它们的主要区别在于，前者在GC过程中，有条件地挂起和唤醒非GC线程，而后者在执行GC的过程中，一直都是挂起非GC线程的。并行GC通过有条件地挂起和唤醒非GC线程，就可以使得应用程序获得更好的响应性。但是我们也应该看到，并行GC需要多执行一次标记根集对象以及递归标记那些在GC过程被访问了的对象的操作。也就是说，并行GC在使用得应用程序获得更好的响应性的同时，也需要花费更多的CPU资源。

为了更深入地理解GC的执行过程，接下来我们再详细分析在上述的23步中涉及到的重要函数。

1. dvmSuspendAllThreads

函数dvmSuspendAllThreads用来挂起Dalvik虚拟机中除当前线程之外的其它线程，它的实现如下所示：

```c
void dvmSuspendAllThreads(SuspendCause why)  
{  
    Thread* self = dvmThreadSelf();  
    Thread* thread;  
    ......  
  
    lockThreadSuspend("susp-all", why);  
    ......  
  
    dvmLockThreadList(self);  
  
    lockThreadSuspendCount();  
    for (thread = gDvm.threadList; thread != NULL; thread = thread->next) {  
if (thread == self)  
    continue;  
  
/* debugger events don't suspend JDWP thread */  
if ((why == SUSPEND_FOR_DEBUG || why == SUSPEND_FOR_DEBUG_EVENT) &&  
    thread->handle == dvmJdwpGetDebugThread(gDvm.jdwpState))  
    continue;  
  
dvmAddToSuspendCounts(thread, 1,  
                      (why == SUSPEND_FOR_DEBUG ||  
                      why == SUSPEND_FOR_DEBUG_EVENT)  
                      ? 1 : 0);  
    }  
    unlockThreadSuspendCount();  
  
    for (thread = gDvm.threadList; thread != NULL; thread = thread->next) {  
if (thread == self)  
    continue;  
  
/* debugger events don't suspend JDWP thread */  
if ((why == SUSPEND_FOR_DEBUG || why == SUSPEND_FOR_DEBUG_EVENT) &&  
    thread->handle == dvmJdwpGetDebugThread(gDvm.jdwpState))  
    continue;  
  
/* wait for the other thread to see the pending suspend */  
waitForThreadSuspend(self, thread);  
......  
    }  
  
    dvmUnlockThreadList();  
    unlockThreadSuspend();  
  
    ......  
}  
```
这个函数定义在文件dalvik/vm/Thread.cpp中。

在以下三种情况下，当前线程需要挂起其它线程：

执行GC。

调试器请求。

发生调试事件，如断点命中。

而且，上述三种情况有可能是同时发生的。例如，一个线程在分配对象的过程中发生GC的同时，调试器也刚好请求挂起所有线程。这时候就需要保证每一个挂起操作都是顺序执行的，即要对挂起线程的操作进行加锁。这是通过调用函数函数lockThreadSuspend来实现的。

函数lockThreadSuspend尝试获取gDvm._threadSuspendLock锁。如果获取失败，就表明有其它线程也正在请求挂起Dalvik虚拟机中的线程，包括当前线程。每一个Dalvik虚拟机线程都有一个Suspend Count计数，每当它们都挂起的时候，对应的Suspend Count计数就会加1，而当被唤醒时，对应的Suspend Count计数就会减1。在获取gDvm._threadSuspendLock锁失败的情况下，当前线程按照一定的时间间隔检查自己的Suspend Count，直到自己的Suspend Count等于0，并且能成功获取gDvm._threadSuspendLock锁为止。这样就可以保证每一个挂起Dalvik虚拟机线程的请求都能够得到顺序执行。

从函数lockThreadSuspend返回之后，就表明当前线程可以执行挂起其它线程的操作了。它首先要做的第一件事情是遍历Dalvik虚拟机线程列表，并且调用函数dvmAddToSuspendCounts将列表里面的每一个线程对应的Suspend Count都增加1，但是除了当前线程之外。注意，当挂起线程的请求是调试器发出或者是由调试事件触发的时候，Dalvik虚拟机中的JDWP线程是不可以挂起的，因为JDWP线程挂起来之后，就没法调试了。在这种情况下，也不能将JDWP线程对应的Suspend Count都增加1。

通过调用函数dvmJdwpGetDebugThread可以获得JDWP线程句柄，用这个句柄和Dalvik虚拟机线程列表中的线程句柄相比，就可以知道当前遍历的线程是否是JDWP线程。同时，通过参数why可以知道当挂起线程的请求是否是由调试器发出的或者由调试事件触发的，

注意，在增加被挂起线程的Suspend Count计数之前，必须要获取gDvm.threadSuspendCountLock锁。这个锁的获取和释放可以通过函数lockThreadSuspendCount和unlockThreadSuspendCount完成。

将被挂起线程的Suspend Count计数都增加1之后，接下来就是等待被挂起线程自愿将自己挂起来了。这是通过函数waitForThreadSuspend来实现。当一个线程自愿将自己挂起来的时候，会将自己的状态设置为非运行状态（THREAD_RUNNING），这样函数waitForThreadSuspend通过不断地检查一个线程的状态是否处于非运行状态就可以知道它是否已经挂起来了。

那么，一个线程在什么情况才会自愿将自己挂起来呢？一个线程在执行的过程中，会在合适的时候检查自己的Suspend Count计数。一旦该计数值不等于0，那么它就知道有线程请求挂起自己，因此它就会很配合地将自己的状态设置为非运行的，并且将自己挂起来。例如，当一个线程通过解释器执行代码时，就会周期性地检查自己的Suspend Count是否等于0。这里说的周期性，实际上就是碰到IF指令、GOTO指令、SWITCH指令、RETURN指令和THROW指令等时。

2. dvmHeapBeginMarkStep

函数dvmHeapBeginMarkStep用来初始化堆的标记范围和Mark Stack，它的实现如下所示：

```c
bool dvmHeapBeginMarkStep(bool isPartial)  
{  
    GcMarkContext *ctx = &gDvm.gcHeap->markContext;  
  
    if (!createMarkStack(&ctx->stack)) {  
return false;  
    }  
    ctx->finger = NULL;  
    ctx->immuneLimit = (char*)dvmHeapSourceGetImmuneLimit(isPartial);  
    return true;  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

在标记过程中用到的各种信息都保存一个GcMarkContext结构体描述的GC标记上下文ctx中。其中，ctx->stack描述的是Mark Stack，ctx->finger描述的是一个标记指纹，在递归标记对象时会用到，ctx->immuneLimit用来限定堆的标记范围。

函数dvmHeapBeginMarkStep调用另外一个函数createMarkStack来初始化Mark Stack，它的实现如下所示：

```c
static bool createMarkStack(GcMarkStack *stack)  
{  
    assert(stack != NULL);  
    size_t length = dvmHeapSourceGetIdealFootprint() * sizeof(Object*) /  
(sizeof(Object) + HEAP_SOURCE_CHUNK_OVERHEAD);  
    madvise(stack->base, length, MADV_NORMAL);  
    stack->top = stack->base;  
    return true;  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

函数createMarkStack根据最坏情况来设置Mark Stack的长度，也就是用当前堆的大小除以对象占用的最小内存得到的结果。当前堆的大小可以通过函数dvmHeapSourceGetIdealFootprint来获得。在堆上分配的对象，都是从Object类继承下来的，因此，每一个对象占用的最小内存是sizeof(Object)。由于在为对象分配内存时，还需要额外的内存来保存管理信息，例如实际分配给对象的内存字节数。这些额外的管理信息占用的内存字节数通过宏HEAP_SOURCE_CHUNK_OVERHEAD来描述。

由于Mark Stack实际上是一个Object指针数组，因此有了上面的信息之后，我们就可以计算出Mark Stack的长度length。Mark Stack使用的内存块在Dalvik虚拟机启动的时候就创建好的，具体参考前面[Dalvik虚拟机Java堆创建过程分析](http://blog.csdn.net/luoshengyang/article/details/41581063)一文，起始地址位于stack->base中。由于这块内存开始的时候被函数madvice设置为MADV_DONTNEED，因此这里要将它重新设置为MADV_NORMAL，以便可以通知内核，这块内存要开始使用了。

Mark Stack的栈顶stack->top指向的是Mark Stack内存块的起始位置，以后就会从这个位置开始由小到大增长。

回到函数dvmHeapBeginMarkStep中，ctx->immuneLimit记录的实际上是堆的起始标记位置。在此位置之前的对象，都不会被GC。这个位置是通过函数dvmHeapSourceGetImmuneLimit来确定的，它的实现如下所示：

```c
void *dvmHeapSourceGetImmuneLimit(bool isPartial)  
{  
    if (isPartial) {  
return hs2heap(gHs)->base;  
    } else {  
return NULL;  
    }  
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapSource.cpp中。

当参数isPartial等于true时，函数dvmHeapSourceGetImmuneLimit返回的是Active堆的起始位置，否则的话就返回NULL值。也就是说，如果当前执行的GC只要求回收部分垃圾，那么就只回收Active堆的垃圾，否则的话，就同时回收Active堆和Zygote堆的垃圾。

dvmHeapMarkRootSet

函数dvmHeapMarkRootSet用来的标记根集对象，它的实现如下所示：

```c
/* Mark the set of root objects. 
 * 
 * Things we need to scan: 
 * - System classes defined by root classloader 
 * - For each thread: 
 *   - Interpreted stack, from top to "curFrame" 
 *     - Dalvik registers (args + local vars) 
 *   - JNI local references 
 *   - Automatic VM local references (TrackedAlloc) 
 *   - Associated Thread/VMThread object 
 *   - ThreadGroups (could track & start with these instead of working 
 *     upward from Threads) 
 *   - Exception currently being thrown, if present 
 * - JNI global references 
 * - Interned string table 
 * - Primitive classes 
 * - Special objects 
 *   - gDvm.outOfMemoryObj 
 * - Objects in debugger object registry 
 * 
 * Don't need: 
 * - Native stack (for in-progress stuff in the VM) 
 *   - The TrackedAlloc stuff watches all native VM references. 
 */  
void dvmHeapMarkRootSet()  
{  
    GcHeap *gcHeap = gDvm.gcHeap;  
    dvmMarkImmuneObjects(gcHeap->markContext.immuneLimit);  
    dvmVisitRoots(rootMarkObjectVisitor, &gcHeap->markContext);  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

通过注释我们可以看到根集对象都包含了哪些对象。总的来说，就是包含两大类对象。一类是Dalvik虚拟机内部使用的全局对象，另一类是应用程序正在使用的对象。前者会维护在内部的一些[数据结构](http://lib.csdn.net/base/datastructure)中，例如Hash表，后者维护在调用栈中。对这些根集对象进行标记都是通过函数dvmVisitRoots和回调函数rootMarkObjectVisitor进行的。

此外，我们还需要将不在堆回收范围内的对象也看作是根集对象，以便后面可以使用统一的方法来遍历这两类对象所引用的其它对象。标记不在堆回收范围内的对象是通过函数dvmMarkImmuneObjects来实现的，如下所示：

```c
void dvmMarkImmuneObjects(const char *immuneLimit)  
{  
    ......  
  
    for (size_t i = 1; i < gHs->numHeaps; ++i) {  
if (gHs->heaps[i].base < immuneLimit) {  
    assert(gHs->heaps[i].limit <= immuneLimit);  
    /* Compute the number of words to copy in the bitmap. */  
    size_t index = HB_OFFSET_TO_INDEX(  
        (uintptr_t)gHs->heaps[i].base - gHs->liveBits.base);  
    /* Compute the starting offset in the live and mark bits. */  
    char *src = (char *)(gHs->liveBits.bits + index);  
    char *dst = (char *)(gHs->markBits.bits + index);  
    /* Compute the number of bytes of the live bitmap to copy. */  
    size_t length = HB_OFFSET_TO_BYTE_INDEX(  
        gHs->heaps[i].limit - gHs->heaps[i].base);  
    /* Do the copy. */  
    memcpy(dst, src, length);  
    /* Make sure max points to the address of the highest set bit. */  
    if (gHs->markBits.max < (uintptr_t)gHs->heaps[i].limit) {  
        gHs->markBits.max = (uintptr_t)gHs->heaps[i].limit;  
    }  
}  
    }  
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapSource.cpp中。

从前面分析的函数dvmHeapSourceGetImmuneLimit可以知道，参数immuneList要么是等于Zygote堆的最大地址值，要么是等于NULL。这取决于当前GC要执行的是全部垃圾回收还是部分垃圾回收。

函数dvmMarkImmuneObjects是在当前GC执行之前调用的，这意味着当前存活的对象都标记在Live Bitmap中。现在函数dvmMarkImmuneObjects要做的就是将不在回收范围内的对象的标记位从Live Bitmap拷贝到Mark Bitmap去。具体做法就是分别遍历Active堆和Zygote堆，如果它们处于不回范围中，那么就对里面的对象在Live Bitmap中对应的内存块拷贝到Mark Bitmap的对应位置去。

计算一个对象在一个Bitmap的标记位所在的内存块是通过宏HB_OFFSET_TO_INDEX和HB_OFFSET_TO_BYTE_INDEX来实现的。这两个宏的具体实现可以参考前面[Dalvik虚拟机Java堆创建过程分析](http://blog.csdn.net/luoshengyang/article/details/41581063)一文。

回到函数dvmHeapMarkRootSet中，我们继续分析函数dvmVisitRoots的实现，如下所示：

```c
void dvmVisitRoots(RootVisitor *visitor, void *arg)  
{  
    assert(visitor != NULL);  
    visitHashTable(visitor, gDvm.loadedClasses, ROOT_STICKY_CLASS, arg);  
    visitPrimitiveTypes(visitor, arg);  
    if (gDvm.dbgRegistry != NULL) {  
visitHashTable(visitor, gDvm.dbgRegistry, ROOT_DEBUGGER, arg);  
    }  
    if (gDvm.literalStrings != NULL) {  
visitHashTable(visitor, gDvm.literalStrings, ROOT_INTERNED_STRING, arg);  
    }  
    dvmLockMutex(&gDvm.jniGlobalRefLock);  
    visitIndirectRefTable(visitor, &gDvm.jniGlobalRefTable, 0, ROOT_JNI_GLOBAL, arg);  
    dvmUnlockMutex(&gDvm.jniGlobalRefLock);  
    dvmLockMutex(&gDvm.jniPinRefLock);  
    visitReferenceTable(visitor, &gDvm.jniPinRefTable, 0, ROOT_VM_INTERNAL, arg);  
    dvmUnlockMutex(&gDvm.jniPinRefLock);  
    visitThreads(visitor, arg);  
    (*visitor)(&gDvm.outOfMemoryObj, 0, ROOT_VM_INTERNAL, arg);  
    (*visitor)(&gDvm.internalErrorObj, 0, ROOT_VM_INTERNAL, arg);  
    (*visitor)(&gDvm.noClassDefFoundErrorObj, 0, ROOT_VM_INTERNAL, arg);  
}  
```
这个函数定义在文件dalvik/vm/alloc/Visit.cpp。

参数visitor指向的是函数rootMarkObjectVisitor，它负责将根集对象在Mark Bitmap中的位设置为1。我们后面再分析这个函数的实现。

以下对象将被视为根集对象：

1. 被加载到Dalvik虚拟机的类对象。这些类对象缓存在gDvm.loadedClasses指向的一个Hash表中，通过函数visitHashTable来遍历和标记。

2. 在Dalvik虚拟机内部创建的原子类，例如Double、Boolean类等，通过函数visitPrimitiveTypes来遍历和标记。

3. 注册在调试器的对象。这些对象保存在gDvm.dbgRegistry指向的一个Hash表中，通过函数visitHashTable来遍历和标记。

4. 字符串常量池中的String对象。这些对象保存保存在gDvm.literalStrings指向的一个Hash表中，通过函数visitHashTable来遍历和标记。

5. 在JNI中创建的全局引用对象所引用的对象。这些被引用对象保存在gDvm.jniGlobalRefTable指向的一个间接引用表中，通过函数visitIndirectRefTable来遍历和标记。

6. 在JNI中通过GetStringUTFChars、GetByteArrayElements等接口访问字符串或者数组时被Pin住的对象。这些对象保存在gDvm.jniPinRefTable指向的一个引用表中，通过函数visitReferenceTable来遍历和标记。

7. 当前位于Davlik虚拟机线程的调用栈的对象。这些对象记录在栈帧中，通过函数visitThreads来遍历和标记。

8. 在Dalvik虚拟机内部创建的OutOfMemory异常对象，这个对象保存在gDvm.outOfMemoryObj中。

9. 在Dalvik虚拟机内部创建的InternalError异常对象，这个对象保存在gDvm.internalErrorObj中。

10. 在Dalvik虚拟机内部创建的N oClassDefFoundError异常对象，这个对象保存在gDvm.noClassDefFoundErrorObj中。

在上述这些根集对象中，最复杂和最难遍历和标记的就是位于Dalvik虚拟机线程栈中的对象，因此接下来我们就继续分析函数发isitThreads的实现，如下所示：

```c
static void visitThreads(RootVisitor *visitor, void *arg)  
{  
    Thread *thread;  
  
    assert(visitor != NULL);  
    dvmLockThreadList(dvmThreadSelf());  
    thread = gDvm.threadList;  
    while (thread) {  
visitThread(visitor, thread, arg);  
thread = thread->next;  
    }  
    dvmUnlockThreadList();  
}  
```
这个函数定义在文件dalvik/vm/alloc/Visit.cpp。

所有的Dalvik虚拟机线程都保存在gDvm.threadList指向的一个列表中，函数visitThreads通过调用另外一个函数函数visitThread来遍历和标记每一个Dalvik虚拟机栈上对象。后者的实现如下所示：

```c
static void visitThread(RootVisitor *visitor, Thread *thread, void *arg)  
{  
    u4 threadId;  
  
    assert(visitor != NULL);  
    assert(thread != NULL);  
    threadId = thread->threadId;  
    (*visitor)(&thread->threadObj, threadId, ROOT_THREAD_OBJECT, arg);  
    (*visitor)(&thread->exception, threadId, ROOT_NATIVE_STACK, arg);  
    visitReferenceTable(visitor, &thread->internalLocalRefTable, threadId, ROOT_NATIVE_STACK, arg);  
    visitIndirectRefTable(visitor, &thread->jniLocalRefTable, threadId, ROOT_JNI_LOCAL, arg);  
    if (thread->jniMonitorRefTable.table != NULL) {  
visitReferenceTable(visitor, &thread->jniMonitorRefTable, threadId, ROOT_JNI_MONITOR, arg);  
    }  
    visitThreadStack(visitor, thread, arg);  
}  
```
这个函数定义在文件dalvik/vm/alloc/Visit.cpp。

对于每一个Dalvik虚拟机线程来说，除了它当前栈上的对象需要标记为根集对象之外，还有以下对象需要标记为根集对象：

用来描述Dalvik虚拟机线程的线程对象。这个对象保存在thread->threadObj中。

当前Davik虚拟机线程的未处理异常对象。这个对象保在在thread->expception中。

Dalvik虚拟机线程内部创建的对象，例如执行过程中抛出的异常对象。这些对象保存在thread->internalLocalRefTable指向的一个引用表中。通过函数visitReferenceTable来遍历和标记。

Dalvik虚拟机线程在JNI中创建的局部引用对象。这些对象保存在thread->jniLocalRefTable指向的一个间接引用表中，通过函数visitIndirectRefTable来遍历和标记。

Dalvik虚拟机线程在JNI中创建的用于同步的Monitor对象。这些对象保存在thread->jniMonitorRefTable指向的一个引用表中，通过函数visitReferenceTable来遍历和标记。

当前栈上的对象通过函数visitThreadStack来遍历和标记，它的实现如下所示：

```c
static void visitThreadStack(RootVisitor *visitor, Thread *thread, void *arg)  
{  
    ......  
    u4 threadId = thread->threadId;  
    const StackSaveArea *saveArea;  
    for (u4 *fp = (u4 *)thread->interpSave.curFrame;  
 fp != NULL;  
 fp = (u4 *)saveArea->prevFrame) {  
Method *method;  
saveArea = SAVEAREA_FROM_FP(fp);  
method = (Method *)saveArea->method;  
if (method != NULL && !dvmIsNativeMethod(method)) {  
    const RegisterMap* pMap = dvmGetExpandedRegisterMap(method);  
    const u1* regVector = NULL;  
    if (pMap != NULL) {  
        ......  
        int addr = saveArea->xtra.currentPc - method->insns;  
        regVector = dvmRegisterMapGetLine(pMap, addr);  
    }  
    if (regVector == NULL) {  
        ......  
        for (size_t i = 0; i < method->registersSize; ++i) {  
            if (dvmIsValidObject((Object *)fp[i])) {  
                (*visitor)(&fp[i], threadId, ROOT_JAVA_FRAME, arg);  
            }  
        }  
    } else {  
        ......  
        u2 bits = 1 << 1;  
        for (size_t i = 0; i < method->registersSize; ++i) {  
            bits >>= 1;  
            if (bits == 1) {  
                .......  
                bits = *regVector++ | 0x0100;  
            }  
            if ((bits & 0x1) != 0) {  
                ......  
                (*visitor)(&fp[i], threadId, ROOT_JAVA_FRAME, arg);  
            }  
        }  
        dvmReleaseRegisterMapLine(pMap, regVector);  
    }  
}  
......  
    }  
}  
```
这个函数定义在文件dalvik/vm/alloc/Visit.cpp。

在Dalvik虚拟机中，每一个Dalvik虚拟机线程都用一个Thread结构体来描述。在Thread结构体中，有一个成员变量interpSave，它指向的是一个InterpSaveState结构体。在InterpSaveState结构体中，又有一个成员变量curFrame，它指向线程的当前调用栈帧，如图2所示：

![img](http://img.blog.csdn.net/20141216023518485?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图2 Dalvik虚拟机线程的调用栈

在图2中，Dalvik虚拟机调用栈由高地址往低地址增长，即栈顶位于低地址。在每一个调用帧前面，有一个类型为StackSaveArea的结构体。在StackSaveArea结构体中，有四个重要的成员变量prevFrame、savedPc、method和currentPc。其中，prevFrame指向前一个调用帧，savedPc指向当前函数调用完成后的返回地址，method指向当前调用的函数，currentPc指向当前执行的指令。通过成员变量prevFrame，就可以从当前调用栈帧开始，遍历整个调用栈。此外，通过成员变量method，可以知道一个调用栈帧对应的调用函数是一个Java函数还是一个JNI函数。如果是一个JNI函数，那么是不需要检查它的栈帧的。因为JNI函数不会在里面保存Java对象。

那么在JNI函数里面创建或者访问的对象是如何管理的呢？我们在JNI函数中需要创建或者访问Java对象时，都必须要通过JNI接口来进行。JNI接口会把JNI函数创建或者访问的Java对象保存线程相关的一个JNI Local Reference Table中。这些被JNI Local Reference Table引用的Java对象，在JNI函数调用结束的时候，会相应地被自动释放引用。但是有些情况下，我们需要在JNI函数中手动释放被JNI Local Reference Table引用的Java对象，因为JNI Local Reference Table的大小是有限的。一旦一个JNI函数引用的Java对象数大于JNI Local Reference Table的容量，那么就会发生错误。这种情况特别容易发生循环语句中。

由于JNI函数里面创建或者访问的对象保存在JNI Local Reference Table中，因此，在前面分析的函数visitThread中，需要对每一个Dalvik虚拟机线程的JNI Local Reference Table中的Java对象进行遍历和标记，避免它们在GC过程中被回收。

好了，我们现在将目光返回到Dalvik虚拟机线程的调用栈中，我们需要做的是遍历那些非JNI函数调用，并且对位于栈帧里面的Java对象进行标记，以免它们在GC过程中被回收。我们知道，Dalvik虚拟机指令是基于寄存器的。也就是说，无论函数里面的参数，还是局部变量，都是保存在寄存器中。但是，我们需要注意的是。这些寄存器不是真实的硬件寄存器，而是虚拟出来的。那么是怎么虚拟出来的呢？其实就是将寄存器映射到调用栈帧对就的内存块上。因此，我们只需要遍历调用栈帧对应的内存块，就可以知道线程的调用栈都引用了哪些Java对象，从而可以对它们进行标记。

现在，新的问题又出现了。调用栈帧的数据不一定都是Java对象引用，还有可能是一个原子类型，例如整数、布尔变量或者浮点数。在遍历调用栈帧的时候，我们是怎么知道里面的一个数据到底是Java对象引用还是一个原子类型呢？我们可以想到的一个办法是判断它们值。如果它们的值是位于Java堆的范围内，那么就认为它们是Java对象引用。但是这样做是不准确的，因为有可能这是一个整数值，并且这个整数值刚好落在Java堆的地址范围之内。不过，我们宁愿将一个整数值误认为一个Java对象引用，也比漏掉一个Java对象引用好。因为将一个整数值误认为一个Java对象引用，导致的后果仅仅是垃圾不能及时回收，而漏掉一个Java对象引用，则意味着一个正在使用Java对象还在被引用的时候被回收。

上面描述的在调用栈帧中，大小值只要是落在Java堆的地址范围之内就认为是Java对象引用的做法称为保守GC算法，与此相对的称为准确GC。在准确GC中，用一个称为Register Map的数据结构来辅助GC。Register Map记录了一个Java函数所有可能的GC执行点所对应的寄存器使用情况。如果在某一个GC执行点，某一个寄存器对应的是一个Java对象引用，那么在对应的Register Map中，就有一位被标记为1。

从前面我们分析的函数dvmSuspendAllThreads可以知道，每当一个Dalvik虚拟机线程被挂起等待GC时，它们总是挂起在IF、GOTO、SWITCH、RETURN和THROW等跳转指令中。这些指令点对应的实际上就GC执行点。因此，我们可以在一个函数中针对出现上述指令的地方，均记录函数当前的寄存器使用情况，从而可以实现准确GC。那么，这个工作是由谁去做的呢？我们知道，APK在安装的时候，安装服务会它们的DEX字节码进行优化，得到一个odex文件。实际上，APK在安装的时候，它们的DEX字节码除了会被优化之外，还会被验证和分析。验证是为了保证不包含非法指令，而分析就是为了得到指令的执行状况，其中就包括得到每一个GC执行点的寄存器使用情况，最终形成一个Register Map，并且保存在odex文件中。这样，当一个odex文件被加载使用的时候，我们就可以直接获得一个函数在某一个GC执行点的寄存器使用情况。

函数visitThreadStack在遍历调用栈帧的过程中，通过调用函数dvmGetExpandedRegisterMap可以获得当前调用函数对应的Register Map。有了这个Register Map之后，再通过在当前StackSaveArea结构体的成员变量currentPc的值，就可以获得一个用来描述调用函数当前寄存器使用情况的一个寄存器向量regVector。在获得的寄存器向量regVector中，如果某一个位的值等于1，那么就说明对应的寄存器保存的是一个Java对象引用。在这种情况下，就需要将该Java对象标记为活的。

注意，函数visitThreadStack按照字节来访问寄存器向量regVector。每次读出1个字节的数据到变量bits的0～7位，然后再将变量bits的第8位设置为1。此后从低位开始，每访问一位就将变量bits向右移一位。当向左移够8位后，就能保证变量bits的值变为1，这时候就再从寄存器向量regVector读取下一字节进行遍历。

由于Register Map不是强制的，因此有可能某些函数不存在对应的Register Map，在这种情况下，就需要使用前面我们所分析的保守GC算法，遍历调用栈帧的所有数据，只要它们的值在Java堆的范围之内，均认为它们是Java对象引用，并且对它们进行标记。

在前面分析的一系列函数，都是通过调用函数rootMarkObjectVisitor来对对象进行标记的，它的实现如下所示：

```c
static void rootMarkObjectVisitor(void *addr, u4 thread, RootType type,  
                          void *arg)  
{  
    ......  
    Object *obj = *(Object **)addr;  
    GcMarkContext *ctx = (GcMarkContext *)arg;  
    if (obj != NULL) {  
markObjectNonNull(obj, ctx, false);  
    }  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

参数addr指向的是Java对象地址，arg指向一个描述当前GC标记上下文的GcMarkContext结构体。函数rootMarkObjectVisitor通过调用另外一个函数markObjectNonNull来标记参数addr所描述的Java对象。

函数markObjectNonNull的实现如下所示：

```c
static void markObjectNonNull(const Object *obj, GcMarkContext *ctx,  
                      bool checkFinger)  
{  
    ......  
    if (obj < (Object *)ctx->immuneLimit) {  
assert(isMarked(obj, ctx));  
return;  
    }  
    if (!setAndReturnMarkBit(ctx, obj)) {  
/* This object was not previously marked. 
 */  
if (checkFinger && (void *)obj < ctx->finger) {  
    /* This object will need to go on the mark stack. 
     */  
    markStackPush(&ctx->stack, obj);  
}  
    }  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

函数markObjectNonNull不仅在标记根集对象的时候会调用到，在递归标记被根集对象所引用的对象的时候也会调用到。在标记根集对象调用时，参数checkFlinger的值等于false，而在递归标记被根集对象所引用的对象时，参数checkFlinger的值等于true，并且参数ctx指向的GcMarkContext结构体的成员变量finger被设置为当前遍历过的Java对象的最大地址值。

前面提到，对于在非回收范围内的Java对象，它们一开始的时候就已经标记为存活的了，因此，函数markObjectNonNull一开始就判断一个参数obj描述的Java对象是否在非回收堆范围内。如果是的话，那么就不需要重复对其它进行标记了。

对于在回收范围内的Java对象，则需要调用函数setAndReturnMarkBit将其在Mark Bitmap中对应的位设置为1。函数setAndReturnMarkBit在将一个位设置为1之前，要返回该位的旧值。也就是说，当函数setAndReturnMarkBit的返回值等于0时，就表示一个Java对象是第一次被标记。在这种情况下，如果参数checkFlinger的值等于true，并且被遍历的Java对象的地址值小于参数ctx指向的GcMarkContext结构体的成员变量finger，那么就需要调用函数markStackPush将该对象压入Mark Stack中，以便后面可以继续对该对象进行递归遍历。

函数setAndReturnMarkBit的实现如下所示：

```c
static long setAndReturnMarkBit(GcMarkContext *ctx, const void *obj)  
{  
    return dvmHeapBitmapSetAndReturnObjectBit(ctx->bitmap, obj);  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

从前面[Dalvik虚拟机Java堆创建过程分析](http://blog.csdn.net/luoshengyang/article/details/41581063)一文可以知道，ctx->bitmap指向的是Mark Bitmap，因此函数setAndReturnMarkBit是将参数obj描述的Java对象在Mark Bitmap中对应的位设置为1。

函数markStackPush的实现如下所示：

```c
static void markStackPush(GcMarkStack *stack, const Object *obj)  
{  
    ......  
    *stack->top = obj;  
    ++stack->top;  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

从这里就可以看出，函数markStackPush将参数obj描述的Java对象压入到Mark Stack中。凡是在Mark Stack中的Java对象，都是需要递归遍历它们所引用的Java对象的。

这样，当函数dvmHeapMarkRootSet调用完毕，所有的根集对象在Mark Bitmap中对应的位就都被设置为1了。

dvmClearCardTable

函数dvmClearCardTable用来清零Card Table，它的实现如下所示：

```c
void dvmClearCardTable()  
{  
    ......  
  
    if (gDvm.lowMemoryMode) {  
      // zero out cards with madvise(), discarding all pages in the card table  
      madvise(gDvm.gcHeap->cardTableBase, gDvm.gcHeap->cardTableLength, MADV_DONTNEED);  
    } else {  
      // zero out cards with memset(), using liveBits as an estimate  
      const HeapBitmap* liveBits = dvmHeapSourceGetLiveBits();  
      size_t maxLiveCard = (liveBits->max - liveBits->base) / GC_CARD_SIZE;  
      maxLiveCard = ALIGN_UP_TO_PAGE_SIZE(maxLiveCard);  
      if (maxLiveCard > gDvm.gcHeap->cardTableLength) {  
  maxLiveCard = gDvm.gcHeap->cardTableLength;  
      }  
  
      memset(gDvm.gcHeap->cardTableBase, GC_CARD_CLEAN, maxLiveCard);  
    }  
}  
```
这个函数定义在文件dalvik/vm/alloc/CardTable.cpp中。

在低内存模式下，函数dvmClearCardTable调用另外一个函数madvice将Card Table对应的虚拟内存块标记为MADV_DONTNEED，以便这块虚拟内存对应的物理内存页可以被内核回收。但是当Card Table对应的虚拟内存块被访问的时候，内核会重新给它映射物理页，并且会将被映射的物理页的值初始化为0。通过这种巧妙的方式，不仅可以将Card Table清零，而且还可以释放暂时不使用的内存，符合低内存模式运行的要求。不过我们也应该注意到，这种做法实际上是通过牺牲程序性能来换取空间的，因为重新给一块虚拟内存映射物理页，以及对该物理页进行初始化都是需要花费时间的。在计算机的世界时，根据实际需要，以时间换空间，或者以空间换时间，都是经典的做法。

在非低内存模式下，函数dvmClearCardTable并没有对整个Card Table进行清零。从前面[Dalvik虚拟机垃圾收集机制简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/41338251)一文可以知道，Card Table虽然只有Heap Bitmap一半的大小，但是当堆较大时，Card Table的绝对值仍然是不小的。而且每次执行GC时，整个堆并没有完全用完。因此，函数dvmClearCardTable就根据当前存活的地址值最大的Java对象来评估应该清零的Card Table大小。一来是因为只有当前存活的Java对象才可能在并行GC期间访问其引用的其它Java对象，二来是不清零部分不会对程序产生影响。试想未清零部分的Card Table，如果它的值本来就于等于0，那么就已经是我们想要的结果了。另一方面，如果未清零部分的Card Table的值等于1，那么产生的后果仅仅就是造成一些垃圾对象没有及时回收而已，但是不会影响程序的正常逻辑。计算出应该清零的部分Card Table大小之后，就调用函数memset来将它们的值设置为GC_CARD_CLEAN，即设置为0。

dvmResumeAllThreads

函数dvmResumeAllThreads用来唤醒之前被挂起的线程，它的实现如下所示：

```c
void dvmResumeAllThreads(SuspendCause why)  
{  
    Thread* self = dvmThreadSelf();  
    Thread* thread;  
  
    lockThreadSuspend("res-all", why);  /* one suspend/resume at a time */  
    ......  
  
    dvmLockThreadList(self);  
    lockThreadSuspendCount();  
    for (thread = gDvm.threadList; thread != NULL; thread = thread->next) {  
if (thread == self)  
    continue;  
  
/* debugger events don't suspend JDWP thread */  
if ((why == SUSPEND_FOR_DEBUG || why == SUSPEND_FOR_DEBUG_EVENT) &&  
    thread->handle == dvmJdwpGetDebugThread(gDvm.jdwpState))  
{  
    continue;  
}  
  
if (thread->suspendCount > 0) {  
    dvmAddToSuspendCounts(thread, -1,  
                          (why == SUSPEND_FOR_DEBUG ||  
                          why == SUSPEND_FOR_DEBUG_EVENT)  
                          ? -1 : 0);  
} else {  
    LOG_THREAD("threadid=%d:  suspendCount already zero",  
        thread->threadId);  
}  
    }  
    unlockThreadSuspendCount();  
    dvmUnlockThreadList();  
    ......  
  
    unlockThreadSuspend();  
    ......  
      
    lockThreadSuspendCount();  
    int cc = pthread_cond_broadcast(&gDvm.threadSuspendCountCond);  
    ......  
  
    unlockThreadSuspendCount();  
    ......  
}   
```
这个函数定义在文件alvik/vm/Thread.cpp中。

对照我们前面分析的函数dvmSuspendAllThreads的实现，就可以比较容易理解函数dvmResumeAllThreads的实现了。在GC期间，GC线程需要挂起其它的Dalvik虚拟机线程时，就将它们的Suspend Count增加1。各个Dalvik虚拟机线程在执行的过程中，会周期性地检查自己的Suspend Count。一旦发现自己的Suspend Count大于0，那么就会将自己挂起，等待GC线程唤醒。

我们以GOTO指令的执行过程说明一个Dalvik虚拟机自愿挂起自己的过程。通过前面[Dalvik虚拟机的运行过程分析](http://blog.csdn.net/luoshengyang/article/details/8914953)一文的学习，我们很容易找到GOTO指令的解释执行过程，如下所示：

```c
HANDLE_OPCODE(OP_GOTO /*+AA*/)  
    vdst = INST_AA(inst);  
    if ((s1)vdst < 0)  
ILOGV("|goto -0x%02x", -((s1)vdst));  
    else  
ILOGV("|goto +0x%02x", ((s1)vdst));  
    ILOGV("> branch taken");  
    if ((s1)vdst < 0)  
PERIODIC_CHECKS((s1)vdst);  
    FINISH((s1)vdst);  
OP_END  

       这段代码定义在文件dalvik/vm/mterp/out/InterpC-portable.cpp中。

       如果是向后跳转，那么就会调用宏PERIODIC_CHECKS来检查是否需要挂起当前线程。

       宏PERIODIC_CHECKS的定义如下所示：

​```c
\#define PERIODIC_CHECKS(_pcadj) {                              \  
if (dvmCheckSuspendQuick(self)) {                                   \  
    EXPORT_PC();  /* need for precise GC */                         \  
    dvmCheckSuspendPending(self);                                   \  
}                                                                   \  
    }  
```
这个宏定义在文件dalvik/vm/mterp/out/InterpC-portable.cpp中。

宏PERIODIC_CHECKS首先是调用函数dvmCheckSuspendQuick来检查保存在当前线程对象里面的一个kSubModeSuspendPending标志位是否等于1。当这个kSubModeSuspendPending等于1时候，就说明线程的Suspend Count值大于0。换句话说，每当一个线程的Suspend Count增加1之后的值大于0，那么都会相应地将对应的线程对对象的kSubModeSuspendPending标志位设置为1。以便后面可以用来快速检测线程是否有挂起请求。

在当前线程的Suspend Count大于0的情况下，宏PERIODIC_CHECKS接下来做两件事情。第一件事情是调用宏EXPORT_PC将当前的PC值记录在当前调用栈帧的StackSaveArea结构体中的成员变量currentPc中，以便可以执行准确GC。这一点是我们前面分析过的。第二件事情就是调用函数dvmCheckSuspendPending来将自己挂起。

函数dvmCheckSuspendPending的实现如下所示：

```c
bool dvmCheckSuspendPending(Thread* self)  
{  
    assert(self != NULL);  
    if (self->suspendCount == 0) {  
return false;  
    } else {  
return fullSuspendCheck(self);  
    }  
}  
```
这个函数定义在文件dalvik/vm/Thread.cpp中。

在我们这个情景中，当前线程的Suspend Count肯定是不等于0的，因此，接下来就会调用函数fullSuspendCheck。

函数fullSuspendCheck的实现如下所示：

```c
static bool fullSuspendCheck(Thread* self)  
{  
    ......  
    lockThreadSuspendCount();   /* grab gDvm.threadSuspendCountLock */  
  
    bool needSuspend = (self->suspendCount != 0);  
    if (needSuspend) {  
......  
ThreadStatus oldStatus = self->status;      /* should be RUNNING */  
self->status = THREAD_SUSPENDED;  
......  
  
while (self->suspendCount != 0) {  
    ......  
    dvmWaitCond(&gDvm.threadSuspendCountCond,  
            &gDvm.threadSuspendCountLock);  
}  
......  
self->status = oldStatus;  
......  
    }  
  
    unlockThreadSuspendCount();  
  
    return needSuspend;  
}  
```
这个函数定义在文件dalvik/vm/Thread.cpp中。

从这里就可以清楚地看到，函数fullSuspendCheck在当前线程的Supend Count不等于0的情况下，首先是将自己的状态设置为THREAD_SUSPENDED，接着挂起在条件变量gDvm.threadSuspendCountCond上，直到被唤醒为止，然后再恢复为之前的状态。

有了这些背景知识之后，回到函数dvmResumeAllThreads中，我们就更容易理解它的实现了。它所要做的就是将每一个被挂起的Dalvik虚拟机线程的Suspend Count减少1，并且在最后将挂起在条件变量gDvm.threadSuspendCountCond上的所有线程都唤醒。

dvmHeapScanMarkedObjects

函数dvmHeapScanMarkedObjects用来递归标记根集对象所引用的其它对象，它的实现如下所示：

```c
void dvmHeapScanMarkedObjects(void)  
{  
    GcMarkContext *ctx = &gDvm.gcHeap->markContext;  
  
    ......  
    dvmHeapBitmapScanWalk(ctx->bitmap, scanBitmapCallback, ctx);  
  
    ctx->finger = (void *)ULONG_MAX;  
  
    ......  
    processMarkStack(ctx);  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

前面已经将根集对象标记在ctx->bitmap指向的Mark Bitmap了，现在就开始调用函数dvmHeapBitmapScanWalk来标记它们直接引用的对象。

函数dvmHeapBitmapScanWalk的实现如下所示：

```c
void dvmHeapBitmapScanWalk(HeapBitmap *bitmap,  
                   BitmapScanCallback *callback, void *arg)  
{  
    ......  
    uintptr_t end = HB_OFFSET_TO_INDEX(bitmap->max - bitmap->base);  
    uintptr_t i;  
    for (i = 0; i <= end; ++i) {  
unsigned long word = bitmap->bits[i];  
if (UNLIKELY(word != 0)) {  
    unsigned long highBit = 1 << (HB_BITS_PER_WORD - 1);  
    uintptr_t ptrBase = HB_INDEX_TO_OFFSET(i) + bitmap->base;  
    void *finger = (void *)(HB_INDEX_TO_OFFSET(i + 1) + bitmap->base);  
    while (word != 0) {  
        const int shift = CLZ(word);  
        Object *obj = (Object *)(ptrBase + shift * HB_OBJECT_ALIGNMENT);  
        (*callback)(obj, finger, arg);  
        word &= ~(highBit >> shift);  
    }  
    end = HB_OFFSET_TO_INDEX(bitmap->max - bitmap->base);  
}  
    }  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

函数dvmHeapBitmapScanWalk遍历Mark Bitmap中值等于1的位，并且调用参烽callback指向的函数来标记它们引用的对象。在遍历的过程中，主要用到了HB_BITS_PER_WORD、HB_INDEX_TO_OFFSET、HB_OBJECT_ALIGNMENT和HB_OFFSET_TO_INDEX这四个宏，它们的具体实现可以参考前面[Dalvik虚拟机Java堆创建过程分析](http://blog.csdn.net/luoshengyang/article/details/41581063)一文。

参烽callback指向的函数指向的函数为scanBitmapCallback，它的实现如下所示：

```c
static void scanBitmapCallback(Object *obj, void *finger, void *arg)  
{  
    GcMarkContext *ctx = (GcMarkContext *)arg;  
    ctx->finger = (void *)finger;  
    scanObject(obj, ctx);  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

参数obj指向要遍历的对象，finger指向当前遍历的Java对象的最大地址，arg指向一个GcMarkContext结构体。Java对象是按照地址值从小到大的顺序遍历的，因此这里传进来的参数finger的值也是递增的。

函数scanBitmapCallback将参数finger的值保存在ctx->finger之后，就调用函数scanObject来标记参数obj指向的对象所引用的对象，它的实现如下所示：

```c
static void scanObject(const Object *obj, GcMarkContext *ctx)  
{  
    ......  
    if (obj->clazz == gDvm.classJavaLangClass) {  
scanClassObject(obj, ctx);  
    } else if (IS_CLASS_FLAG_SET(obj->clazz, CLASS_ISARRAY)) {  
scanArrayObject(obj, ctx);  
    } else {  
scanDataObject(obj, ctx);  
    }  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

不同类型的对象调用不同的函数来遍历。其中，类对象使用函数scanClassObject来遍历，数组对象使用函数scanArrayObject来遍历，普通对象使用函数scanDataObject来遍历。

接下来，我们主要分析普通对象的标记过程，即函数scanDataObject的实现，如下所示：

```c
static void scanDataObject(const Object *obj, GcMarkContext *ctx)  
{  
    ......  
    markObject((const Object *)obj->clazz, ctx);  
    scanFields(obj, ctx);  
    if (IS_CLASS_FLAG_SET(obj->clazz, CLASS_ISREFERENCE)) {  
delayReferenceReferent((Object *)obj, ctx);  
    }  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

注意，参数obj指向的对象是已经标记过了的，因此，函数scanDataObject只标记它所引用的对象，其中就包括它所引用的类对象obj->clazz，以及它的引用类型的成员变量所引用的对象。前者直接通过函数markObject来标记，后者通过函数scanField依次遍历标记，不过最终也是通过调用函数markObject来标记每一个被引用对象。因此，接下来我就主要分析函数markObject的实现，如下所示：

```c
static void markObject(const Object *obj, GcMarkContext *ctx)  
{  
    if (obj != NULL) {  
markObjectNonNull(obj, ctx, true);  
    }  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

这里我们又看到前面已经分析过的函数markObjectNonNull，不同的是这里指定的第三个参数为true，表示要将那些地址值小于ctx->finger的并且是第一次被标记的Java对象压入到Mark Stack去，以便后面可以继续对它们进行递归遍历。

回到函数scanDataObject中，对于引用类型的对象，需要调用函数delayReferenceReferent对它们所引用的目标对象进行特殊处理，如下所示：

```c
static void delayReferenceReferent(Object *obj, GcMarkContext *ctx)  
{  
    ......  
    GcHeap *gcHeap = gDvm.gcHeap;  
    size_t pendingNextOffset = gDvm.offJavaLangRefReference_pendingNext;  
    size_t referentOffset = gDvm.offJavaLangRefReference_referent;  
    Object *pending = dvmGetFieldObject(obj, pendingNextOffset);  
    Object *referent = dvmGetFieldObject(obj, referentOffset);  
    if (pending == NULL && referent != NULL && !isMarked(referent, ctx)) {  
Object **list = NULL;  
if (isSoftReference(obj)) {  
    list = &gcHeap->softReferences;  
} else if (isWeakReference(obj)) {  
    list = &gcHeap->weakReferences;  
} else if (isFinalizerReference(obj)) {  
    list = &gcHeap->finalizerReferences;  
} else if (isPhantomReference(obj)) {  
    list = &gcHeap->phantomReferences;  
}  
......  
enqueuePendingReference(obj, list);  
    }  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

引用类型的对象，例如SoftReference、WeakReference、PhantomReference和FinalizerReference，都是从父类Reference继承下来的。在父类Reference中，有两个成员变量pendingNext和referent。前者用来将相同类型的引用对象链接起来形成一个链表，等待GC处理，只在虚拟机内部使用，后者指向目标对象。

如果一个引用对象还没有加入到Dalvik虚拟机内部的链表中，即其成员变量pendingNext的值等于NULL，并且它引用了目标对象，即其成员变量referent的值不等于NULL，并且它还没有被标记过，那么就要将它加入到相应的列表去等待GC处理。

不同类型的引用对象被加入到不同的列表等待处理，其中：

软引用对象加入到gcHeap->softReferences列表；

弱引用对象加入到gcHeap->weakReferences列表；

finalizer引用对象，即重写了成员函数finalize的对象，加入到gcHeap->finalizerReferences列表；

影子引用对象加入到gcHeap->phantomReferences列表。

将一个引用对象加入到指定的列表是通过函数enqueuePendingReference来实现的，实际上就是通过引用对象的成员变量pendingNext链入到列表中。

执行完成上述的函数后，被根集对象引用的一部分对象就也被递归标记了。回到函数dvmHeapScanMarkedObjects中，接下来需要继续调用函数processMarkStack来递归标记还没有被标记的另外一部分对象。

在分析函数processMarkStack的实现之前，我们首先要明确哪些是该标记又未标记的对象。我们首先要明确两点：

在遍历Mark Bitmap之前，根集对象已经被标记；

遍历Mark Bitmap是按地址值从小到大的顺序遍历的。

这意味着，遍历完成Mark Bitmap之后，以下对象都是已经被标记的：

根集对象；

被根集对象直接或者间接引用并且地址值比当前遍历的对象还要大的对象。

这时候该标记还没有标记的对象就剩下那些被根集对象直接或者间接引用并且地址值比当前遍历的对象还要小的对象。这些对象统统被压入到了Mark Stack中，因此，函数processMarkStack只要继续递归遍历Mark Stack中的对象就可以对那些该标记还没有被标记的对象进行标记。

举个例子说，有四个对象A、B、C、D、E和F，其中，C和D是根集对象，它们的地址址依次递增，C引用了B，D引用了E。B又引用了A和F。遍历Mark Bitmap的时候，依次发生以下事情：

C被遍历；

C引用了B，但是由于其地址值比B大，因此B被标记并且被压入Mark Stack中；

D被遍历；

D引用了E，但是由于其地址值比D大，因此E被标记；

由于E被标记了，因此E也会被遍历。

遍历完成Mark Bitmap之后，Mark Stack被压入了B，因此要从Mark Stack中弹出B，并且对B进行遍历。遍历B的过程如下所示：

B引用了A，因此A被标记同时也会被压入到Mark Stack中；

B引用了F，因此F也会被标标记以及压入到Mark Stack中；

现在Mark Stack又被压入了A和F，因此又要继续将它们弹出进行遍历：

遍历A，它没有引用其它对象，因此没有对象被压入Mark Stack中；

遍历F。它也没有引用其它对象，因此也没有对象被村入Mark Stack中。

此轮遍历结束后，Mark Stack为空，因此所有被根集对象直接或者间接引用均被标记。

有了上述背景知识之后，函数processMarkStack的实现就一目了然了，它的实现如下所示：

```c
static void processMarkStack(GcMarkContext *ctx)  
{  
    assert(ctx != NULL);  
    assert(ctx->finger == (void *)ULONG_MAX);  
    assert(ctx->stack.top >= ctx->stack.base);  
    GcMarkStack *stack = &ctx->stack;  
    while (stack->top > stack->base) {  
const Object *obj = markStackPop(stack);  
scanObject(obj, ctx);  
    }  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

在调用函数processMarkStack之前，ctx->finger的值已经被设置为ULONG_MAX，因此，在接下来的对象标记过程中，只要被引用对象是第一次被标记，都会被压入到Mark Stack中进行递归遍历，直到Mark Stack变成空为止。

函数processMarkStack是通过调用函数scanObject来标记被Mark Stack中的对象所引用的对象的。这个函数前面已经分析过了，这里就不再复述。

这样，当函数dvmHeapScanMarkedObjects调用完毕，所有的被根集对象直接或者间接引用的对象都被递归地在Mark Bitmap中标记了。

dvmHeapReMarkRootSet

函数dvmHeapReMarkRootSet用来重新标记根集对象，在并行GC中调用，它的实现如下所示：

```c
void dvmHeapReMarkRootSet()  
{  
    GcMarkContext *ctx = &gDvm.gcHeap->markContext;  
    assert(ctx->finger == (void *)ULONG_MAX);  
    dvmVisitRoots(rootReMarkObjectVisitor, ctx);  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

与前面分析的函数dvmHeapMarkRootSet类似，函数dvmHeapReMarkRootSet也是调用函数dvmVisitRoots来重新遍历根集对象，不过使用的是另外一个函数rootReMarkObjectVisitor来标记根集对象。

函数rootReMarkObjectVisitor的实现如下所示：

```c
static void rootReMarkObjectVisitor(void *addr, u4 thread, RootType type,  
                            void *arg)  
{  
    ......  
    Object *obj = *(Object **)addr;  
    GcMarkContext *ctx = (GcMarkContext *)arg;  
    if (obj != NULL) {  
markObjectNonNull(obj, ctx, true);  
    }  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

与前面分析的函数rootMarkObjectVisitor类似，函数rootReMarkObjectVisitor也是使用函数markObjectNonNull来标记根集对象，不同的是后者在调用函数markObjectNonNull时，第三个参数指定为true，并且在此之前，ctx->finger的值已经被设置为ULONG_MAX，因此，被根集对象直接引用的对象都会被压入到Mark Stack中。

这些被压入到Mark Stack中的对象后面会进一步得到处理。

dvmHeapReScanMarkedObjects

函数dvmHeapReScanMarkedObjects用来标记那些在并行GC执行过程被引用的对象，它的实现如下所示：

```c
void dvmHeapReScanMarkedObjects()  
{  
    GcMarkContext *ctx = &gDvm.gcHeap->markContext;  
  
    /* 
     * The finger must have been set to the maximum value to ensure 
     * that gray objects will be pushed onto the mark stack. 
     */  
    assert(ctx->finger == (void *)ULONG_MAX);  
    scanGrayObjects(ctx);  
    processMarkStack(ctx);  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

函数dvmHeapReScanMarkedObjects首先是调用函数scanGrayObjects标记那些在并行GC过程中被修改的对象，这些对象称为灰色（Gray）对象。更通俗地说，灰色对象就是那些自己已经被标记但是自己引用的对象还没有被标记的对象，它们以后需要再次遍历和标记引用的对象。相应地，也存在黑色（Black）对象和白色（White）对象。黑色对象是指那些自己已经被标记并且自己引用的对象也已经被标记的对象，而白色对象就是那些没有被标记的对象，它们是要被回收的。

在前面[Dalvik虚拟机垃圾收集机制简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/41338251)一文中，我们提到，Card Table主要是用来在并行执行的、并且只进行部分垃圾回收的GC。这一点从这里就可以得到体现。首先，在并行GC的标记阶段，GC线程和其它线程是同时运行的，这样其它线程就有机会去操作对象，这样会造成GC线程标错对象或者漏标对象。标错对象不会对程序的正确性造成影响，顶多就是造成标错对象不能及时回收。但是漏标对象问题就大了，这会造成还被引用的对象被GC回收掉。因此，并行GC就需要执行两遍标记阶段。第一遍是GC线程和其它线程同时运行的，而第二遍是只有GC线程运行的。由于大部分对象在第一遍标记阶段已经被标记，因此第二遍真正需要标记的对象不会太多，这样就可以保证第二遍标记阶段可以快速完成。

现在我们分两种情况进行考虑。第一种情况是当前执行的是全部垃圾收集，即在Zygote堆和Active堆上分配对象都进行回收。这时候实际上我们通过函数processMarkStack递归标记函数dvmHeapReMarkRootSet重新标记的根集对象所引用的其它对象，就可以保证不会漏标对象。第二种情况下是当前执行的部分垃圾收集，即只回收在Active堆上分配的对象。这时候如果在GC线程和其它线程运行阶段，其它线程修改了在Zygote堆上分配的对象，造成在Zygote堆上分配的对象引用了一个在Active堆上分配的、原来没有被标记的、并且也不在新的根集内的对象，那么就一定会造成这个在Active堆分配的对象被错误地回收。不过这时好在Card Table记录了这个在Zygote堆上分配的、被修改了的对象，因此就可以调用函数scanGaryObjects来递归标记它所引用的在Active堆上分配的、原来没有被标记的、并且也不在新的根集内的对象，保证这些被重新引用的对象不会被错误回收。

注意，函数scanGrayObjects来遍历和标记灰色对象的过程中，也会将那些被灰色对象引用的其它对象压入到Mark Stack中。这样在Mark Stack中，就汇集了之前由新的根集对象和灰色对象所引用的其它对象，再通过调用前面分析的函数processMarkStack，就可以递归地将所有这些被引用的对象进行标记。

接下来，我们就重点分析函数scanGrayObjects的实现，如下所示：

```c
static void scanGrayObjects(GcMarkContext *ctx)  
{  
    GcHeap *h = gDvm.gcHeap;  
    const u1 *base, *limit, *ptr, *dirty;  
  
    base = &h->cardTableBase[0];  
    // The limit is the card one after the last accessible card.  
    limit = dvmCardFromAddr((u1 *)dvmHeapSourceGetLimit() - GC_CARD_SIZE) + 1;  
    assert(limit <= &base[h->cardTableOffset + h->cardTableLength]);  
  
    ptr = base;  
    for (;;) {  
dirty = (const u1 *)memchr(ptr, GC_CARD_DIRTY, limit - ptr);  
if (dirty == NULL) {  
    break;  
}  
assert((dirty > ptr) && (dirty < limit));  
ptr = scanDirtyCards(dirty, limit, ctx);  
if (ptr == NULL) {  
    break;  
}  
assert((ptr > dirty) && (ptr < limit));  
    }  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

在前面[Dalvik虚拟机Java堆创建过程分析](http://blog.csdn.net/luoshengyang/article/details/41581063)一文提到，在并行GC执行的过程中，如果一个对象被修改了，例如通过函数dvmSetFieldObject修改了一个对象的引用类型的成员变量，那么就会调用函数dvmMarkCard将Card Table中对应的字节的设置为GC_CARD_DIRTY。

函数scanGrayObjects通过函数memchr依次找到Card Table中包含GC_CARD_DIRTY值的内存块，然后再使用函数scanDirtyCards该内存块进行遍历，以便找到那些有可能将该内存块设置为GC_CARD_DIRTY的对象，并且对它们进行遍历。

函数scanDirtyCards的实现如下所示：

```c
const u1 *scanDirtyCards(const u1 *start, const u1 *end,  
                 GcMarkContext *ctx)  
{  
    const HeapBitmap *markBits = ctx->bitmap;  
    const u1 *card = start, *prevAddr = NULL;  
    while (card < end) {  
if (*card != GC_CARD_DIRTY) {  
    return card;  
}  
const u1 *ptr = prevAddr ? prevAddr : (u1*)dvmAddrFromCard(card);  
const u1 *limit = ptr + GC_CARD_SIZE;  
while (ptr < limit) {  
    Object *obj = nextGrayObject(ptr, limit, markBits);  
    if (obj == NULL) {  
        break;  
    }  
    scanObject(obj, ctx);  
    ptr = (u1*)obj + ALIGN_UP(objectSize(obj), HB_OBJECT_ALIGNMENT);  
}  
if (ptr < limit) {  
    /* Ended within the current card, advance to the next card. */  
    ++card;  
    prevAddr = NULL;  
} else {  
    /* Ended past the current card, skip ahead. */  
    card = dvmCardFromAddr(ptr);  
    prevAddr = ptr;  
}  
    }  
    return NULL;  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

函数scanDirtyCards最外层的while循环依次遍历[start, end)范围内的Dirty Card。从前面[Dalvik虚拟机垃圾收集机制简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/41338251)一文可以知道，每GC_CARD_SIZE个对象共用一个Dirty Card，因此，对于每一个Dirty Card，函数scanDirtyCards内层的while循环都必须遍历可能存在的GC_CARD_SIZE个对象，并且调用我们前面分析过的函数scanObject对它们引用的其它对象进行标记。

注意，在函数scanDirtyCards中，我们最开始知道的只有Dirty Card的地址，但是我们遍历对象的时候需要的是对象地址，因此需要从Dirty Card地址计算出对象地址，这是通过函数dvmAddrFromCard来实现的。此外，我们还可以通过函数dvmCardFromAddr根据对象地址获得其对应的Card地址。关于函数dvmAddrFromCard和dvmCardFromAddr的实现，可以参考前面[Dalvik虚拟机Java堆创建过程分析](http://blog.csdn.net/luoshengyang/article/details/41581063)一文。

取出一个Dirty Card对应的下一个对象可以通过函数nextGrayObject来实现。由于一个Card对应的GC_CARD_SIZE个对象在堆空间上都是连续的，因此，当我们从Dirty Card中取得前一个对象之后，就可以根据该对象的大小估算出下一个对象的可能地址。通过这种方式，可以加快从Dirty Card中取出对象的效率。

函数nextGrayObject的实现如下所示：

```c
static Object *nextGrayObject(const u1 *base, const u1 *limit,  
                      const HeapBitmap *markBits)  
{  
    const u1 *ptr;  
  
    ......  
    for (ptr = base; ptr < limit; ptr += HB_OBJECT_ALIGNMENT) {  
if (dvmHeapBitmapIsObjectBitSet(markBits, ptr))  
    return (Object *)ptr;  
    }  
    return NULL;  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

虽然每GC_CARD_SIZE个对象共用一个Dirty Card，但是这些对象不一定会存在。对于不存在的对象，我们是不需要标记它们所引用的对象的。因此，从Dirty Card取出下一个对象的时候，只需要调用函数dvmHeapBitmapIsObjectBitSet判断它在Mark Bitmap中对应的位是否等于1，就可以知道该对象是否存在了。之所以能够这样做是因为，一个灰色对象要么是一个在并行GC过程中新创建或者新访问的对象，要么是一个原来已经存在的对象，即在前一轮标记中已经在Mark Bitmap中标记为存活。无论是哪一种对象，如果它们在并行GC运行之后，仍然是存活的，那么要么是在新的根集中，要么是要原来已经被标记为存活的对象集合中。因此，不管是哪一种情况，都可以保证在并GC运行之后仍然存活的对象在Mark Bitmap中是被标记过的。现在我们遍历灰色对象，仅仅是为将它们所引用的对象进行标记。换句话说，通过函数nextGrayObject的过滤，可以将那些虽然在并行GC过程中被修改过的但是在并行GC执行过后不再存活了的对象过滤掉，这样就可以加快后面归递标记对象的过程。

这样，当函数dvmHeapReScanMarkedObjects执行完成之后，那些在并行GC执行过程中产生的新根集对象和被重新引用的对象都会得到标记。

dvmHeapProcessReferences

函数dvmHeapProcessReferences用来处理引用类型的对象，它的实现如下所示：

```c
void dvmHeapProcessReferences(Object **softReferences, bool clearSoftRefs,  
                      Object **weakReferences,  
                      Object **finalizerReferences,  
                      Object **phantomReferences)  
{  
    assert(softReferences != NULL);  
    assert(weakReferences != NULL);  
    assert(finalizerReferences != NULL);  
    assert(phantomReferences != NULL);  
    /* 
     * Unless we are in the zygote or required to clear soft 
     * references with white references, preserve some white 
     * referents. 
     */  
    if (!gDvm.zygote && !clearSoftRefs) {  
preserveSomeSoftReferences(softReferences);  
    }  
    /* 
     * Clear all remaining soft and weak references with white 
     * referents. 
     */  
    clearWhiteReferences(softReferences);  
    clearWhiteReferences(weakReferences);  
    /* 
     * Preserve all white objects with finalize methods and schedule 
     * them for finalization. 
     */  
    enqueueFinalizerReferences(finalizerReferences);  
    /* 
     * Clear all f-reachable soft and weak references with white 
     * referents. 
     */  
    clearWhiteReferences(softReferences);  
    clearWhiteReferences(weakReferences);  
    /* 
     * Clear all phantom references with white referents. 
     */  
    clearWhiteReferences(phantomReferences);  
    /* 
     * At this point all reference lists should be empty. 
     */  
    assert(*softReferences == NULL);  
    assert(*weakReferences == NULL);  
    assert(*finalizerReferences == NULL);  
    assert(*phantomReferences == NULL);  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

前面在标记对象的过程中，SoftReference、WeakReference、FinalizerReference和PhantomReference类型的引用对象分别被保存在了gcHeap->softReferences、gcHeap->weakReferences、gcHeap->finalizerReferences和gcHeap->phantomReferences四个列表中，函数dvmHeapProcessReferences的参数softReferences、weakReferences、finalizerReferences和phantomReferences分别指向这四个表中。

另外一个参数clearSoftRefs表示是否要全部回收SoftReference引用的对象。只有堆内存很紧张的时候，SoftReference引用的对象才会被回收。从前面[Dalvik虚拟机为新创建对象分配内存的过程分析](http://blog.csdn.net/luoshengyang/article/details/41688319)一文可以知道，在堆上分配对象的过程中遇到内存不足时，首先尝试在不全部回收SoftReference引用的对象情况下进行GC。如果GC之后仍然是不能成功分配对象，最后才会再次进行GC，并且指定要回收SoftReference引用的对象。

如果不要求全部回收SoftReference引用的对象，即参数clearSoftRefs的值等于false，并且当前进程是应用程序进程，即gDvm.zygote的值等于false，那么就会保留部分SoftReference引用的对象。换句话说，如果当前进程是Zygote进程，或者要求全部回收SoftReference引用的对象，那么被SoftReference引用的对象就一个都不能保留。

保留部分SoftReference引用的对象是通过函数preserveSomeSoftReferences来实现的，如下所示：

```c
static void preserveSomeSoftReferences(Object **list)  
{  
    assert(list != NULL);  
    GcMarkContext *ctx = &gDvm.gcHeap->markContext;  
    size_t referentOffset = gDvm.offJavaLangRefReference_referent;  
    Object *clear = NULL;  
    size_t counter = 0;  
    while (*list != NULL) {  
Object *ref = dequeuePendingReference(list);  
Object *referent = dvmGetFieldObject(ref, referentOffset);  
if (referent == NULL) {  
    /* Referent was cleared by the user during marking. */  
    continue;  
}  
bool marked = isMarked(referent, ctx);  
if (!marked && ((++counter) & 1)) {  
    /* Referent is white and biased toward saving, mark it. */  
    markObject(referent, ctx);  
    marked = true;  
}  
if (!marked) {  
    /* Referent is white, queue it for clearing. */  
    enqueuePendingReference(ref, &clear);  
}  
    }  
    *list = clear;  
    /* 
     * Restart the mark with the newly black references added to the 
     * root set. 
     */  
    processMarkStack(ctx);  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

函数preserveSomeSoftReferences依次遍历保存在参数list描述的SoftReference引用对象列表，并且检查它们所引用的对象。如果被引用对象存在，并且之前没有被标记过，那么就隔一个就调用函数markObject来进行标记。而对于引用对象没有被标记的引用，则重新被链入到参数list描述的SoftReference引用对象列表，以便后面可以进行处理。换句话说，Dalvik虚拟机在GC过程中保留SoftReference引用对象引用的对象的策略是每隔一个就保留一个，保留总数的一半。

由于被保留的对象可能又会引用其它对象，因此我们需要递归地标记被保留对象。在调用函数markObject标记被保留对象的过程中，被保留对象会被压入到Mark Stack中，因此，这里我们只要调用前面分析过的函数processMarkStack就可以递归地标记被保留对象，直到它们直接或者间接引用的对象全部都被标记为止。

回到函数dvmHeapProcessReferences中，保留了部分SoftReference引用的对象之后，接下来就开始调用函数clearWhiteReferences清理剩余的被SoftReference引用的对象以及所有被WeakReference引用的对象。这些对象除了被SoftReference和WeakReference引用之外，再没有被其它对象引用，也就是说它们接下来是要被回收的，因此我们将引用了它们的SoftReference和WeakReference称为White Reference。

函数clearWhiteReferences的实现如下所示：

```c
static void clearWhiteReferences(Object **list)  
{  
    assert(list != NULL);  
    GcMarkContext *ctx = &gDvm.gcHeap->markContext;  
    size_t referentOffset = gDvm.offJavaLangRefReference_referent;  
    while (*list != NULL) {  
Object *ref = dequeuePendingReference(list);  
Object *referent = dvmGetFieldObject(ref, referentOffset);  
if (referent != NULL && !isMarked(referent, ctx)) {  
    /* Referent is white, clear it. */  
    clearReference(ref);  
    if (isEnqueuable(ref)) {  
        enqueueReference(ref);  
    }  
}  
    }  
    assert(*list == NULL);  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

只有那些之前没有被标记过的，并且被SoftReference和WeakReference引用的对象才会被处理。处理过程如下所示：

调用函数clearReference函数切断引用对象和被引用对象的关系，也就是将引用对象的成员变量referent设置为NULL。

如果引用对象在创建的时候关联有队列，那么就调用函数enqueueReference将其加入到关联队列中去，以便应用程序可以知道哪些引用对象引用的对象被GC回收了。

再回到函数dvmHeapProcessReferences中，处理完成SoftReference和WeakReference引用之后。接着再调用函数enqueueFinalizerReferences处理FinalizerReference引用。

函数enqueueFinalizerReferences的实现如下所示：

```c
static void enqueueFinalizerReferences(Object **list)  
{  
    assert(list != NULL);  
    GcMarkContext *ctx = &gDvm.gcHeap->markContext;  
    size_t referentOffset = gDvm.offJavaLangRefReference_referent;  
    size_t zombieOffset = gDvm.offJavaLangRefFinalizerReference_zombie;  
    bool hasEnqueued = false;  
    while (*list != NULL) {  
Object *ref = dequeuePendingReference(list);  
Object *referent = dvmGetFieldObject(ref, referentOffset);  
if (referent != NULL && !isMarked(referent, ctx)) {  
    markObject(referent, ctx);  
    /* If the referent is non-null the reference must queuable. */  
    assert(isEnqueuable(ref));  
    dvmSetFieldObject(ref, zombieOffset, referent);  
    clearReference(ref);  
    enqueueReference(ref);  
    hasEnqueued = true;  
}  
    }  
    if (hasEnqueued) {  
processMarkStack(ctx);  
    }  
    assert(*list == NULL);  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

函数enqueueFinalizerReferences依次遍历参数list描述的FinalizerReference列表，对于被FinalizerReference引用的，并且之前没有被标记过的对象，执行以下操作：

1. 调用函数markObject对其进行标记。

2. 调用函数dvmSetFieldObject将被引用对象从FinalizerReference引用的成员变量referent转移到成员变量zombie。

3. 调用函数clearReference将FinalizerReference引用的成员变量referent设置为NULL。

4. 调用函数enqueueReferece将FinalizerReference引用加入到关联的列表。

执行完成上述四个步骤后，Dalvik虚拟机里面的Finalizer线程就可以调度执行被FinalizerReference引用的对象的成员函数finalize了。也正是由于被FinalizerReference引用的对象需要执行成员函数finalize，所以它们在没有被回收之前，需要再标记一次。

与前面调用函数preserveSomeSoftReferences来保留部分被SoftReference引用的对象类似，在函数enqueueFinalizerReferences里被重新标记的对象，有可能引用了其它对象。因此，在函数enqueueFinalizerReferences的最后，需要调用函数processMarkStack来进行递归标记。

再回到函数dvmHeapProcessReferences中，这时候被FinalizerReference引用的对象就递归标记完成了。但是在递归中被标记的这些对象中，有些可能是SoftReference对象，有些也可能是WeakReference对象。对于这两类对象，它所引用的对象是要回收的，因此，函数dvmHeapProcessReferences接下来要调用函数clearWhiteReferences对它们进行清理。

最后，函数dvmHeapProcessReferences调用函数clearWhiteReferences来清理被PhantomReference引用的对象。

从函数dvmHeapProcessReferences的执行过程就可以看出，在一次GC中，被SoftReference引用的对象有可能被回收，也有可能不被回收，定义有成员函数finalize的对象有一次机会不被回收，被WeakReference和PhantomReference引用的对象全部会被回收。

dvmHeapSweepSystemWeaks

函数dvmHeapSweepSystemWeaks用来回收在Dalvik虚拟机内部使用的类似被WeakReference引用的对象，它的实现如下所示：

```c
void dvmHeapSweepSystemWeaks()  
{  
    dvmGcDetachDeadInternedStrings(isUnmarkedObject);  
    dvmSweepMonitorList(&gDvm.monitorList, isUnmarkedObject);  
    sweepWeakJniGlobals();  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

函数dvmHeapSweepSystemWeaks回收的对象包括：

A. 在字符串常量池中不再被引用的字符串。它们通过函数dvmGcDetachDeadInternedStrings来回收。

B. 在gDvm.monitorList列表不再被引用的Monitor对象。它们通过函数dvmSweepMonitorList来回收。

C. 在JNI中被WeakReference引用的全局对象，它们通过函数sweepWeakJniGlobals来回收。

11. dvmHeapSourceSwapBitmaps

函数dvmHeapSourceSwapBitmaps用来交换Live Bitmap和Mark Bitmap，它的实现如下所示：

```c
void dvmHeapSourceSwapBitmaps()  
{  
    HeapBitmap tmp = gHs->liveBits;  
    gHs->liveBits = gHs->markBits;  
    gHs->markBits = tmp;  
}  
```
这个函数定义在文件alvik/vm/alloc/HeapSource.cpp中。

在执行函数dvmHeapSourceSwapBitmaps的时候，在当前GC后还存活的对象在Mark Bitmap中对应的位都已经设置为1，而在上一次GC后还存活的对象在Live Bitmap中对应的位都已经被设置为1。由于Live Bitmap总是用来标记上一次GC后还存活的对象，因此，这里就将Mark Bitmap和Live Bitmap进行交换。

dvmHeapSweepUnmarkedObjects

函数dvmHeapSweepUnmarkedObjects用来清除不再被引用的对象，它的实现如下所示：

```c
void dvmHeapSweepUnmarkedObjects(bool isPartial, bool isConcurrent,  
                         size_t *numObjects, size_t *numBytes)  
{  
    uintptr_t base[HEAP_SOURCE_MAX_HEAP_COUNT];  
    uintptr_t max[HEAP_SOURCE_MAX_HEAP_COUNT];  
    SweepContext ctx;  
    HeapBitmap *prevLive, *prevMark;  
    size_t numHeaps, numSweepHeaps;  
  
    numHeaps = dvmHeapSourceGetNumHeaps();  
    dvmHeapSourceGetRegions(base, max, numHeaps);  
    if (isPartial) {  
assert((uintptr_t)gDvm.gcHeap->markContext.immuneLimit == base[0]);  
numSweepHeaps = 1;  
    } else {  
numSweepHeaps = numHeaps;  
    }  
    ctx.numObjects = ctx.numBytes = 0;  
    ctx.isConcurrent = isConcurrent;  
    prevLive = dvmHeapSourceGetMarkBits();  
    prevMark = dvmHeapSourceGetLiveBits();  
    for (size_t i = 0; i < numSweepHeaps; ++i) {  
dvmHeapBitmapSweepWalk(prevLive, prevMark, base[i], max[i],  
                       sweepBitmapCallback, &ctx);  
    }  
    *numObjects = ctx.numObjects;  
    *numBytes = ctx.numBytes;  
    if (gDvm.allocProf.enabled) {  
gDvm.allocProf.freeCount += ctx.numObjects;  
gDvm.allocProf.freeSize += ctx.numBytes;  
    }  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

前面提到，当当前GC执行的是部分垃圾回收时，即参数isPartial等于true时，只有Active堆的垃圾会被回收。由于Active堆总是在base[0]，所以函数dvmHeapSweepUnmarkedObjects在只执行部分垃圾回收时，将变量numSweepHeaps的值设置为1，就可以保证在后面的for循环只会遍历和清除Active堆的垃圾。

此外，变量prevLive和preMark指向的分别是当前Mark Bitmap和Live Bitmap，但是由于在执行函数dvmHeapSweepUnmarkedObjects之前，Mark Bitmap和Live Bitmap已经被交换，因此，变量prevLive和preMark实际上指向的Bitmap描述的分别上次GC后仍然存活的对象和当前GC后仍然存活的对象。

准备工作完成之后，函数dvmHeapSweepUnmarkedObjects调用函数dvmHeapBitmapSweepWalk来遍历上次GC后仍存活但是当前GC后不再存活的对象。这些对象正是要被回收的对象，它们通过函数sweepBitmapCallback来回收。

函数dvmHeapSweepUnmarkedObjects的实现如下所示：

```c
void dvmHeapBitmapSweepWalk(const HeapBitmap *liveHb, const HeapBitmap *markHb,  
                    uintptr_t base, uintptr_t max,  
                    BitmapSweepCallback *callback, void *callbackArg)  
{  
    ......  
    void *pointerBuf[4 * HB_BITS_PER_WORD];  
    void **pb = pointerBuf;  
    size_t start = HB_OFFSET_TO_INDEX(base - liveHb->base);  
    size_t end = HB_OFFSET_TO_INDEX(max - liveHb->base);  
    unsigned long *live = liveHb->bits;  
    unsigned long *mark = markHb->bits;  
    for (size_t i = start; i <= end; i++) {  
unsigned long garbage = live[i] & ~mark[i];  
if (UNLIKELY(garbage != 0)) {  
    unsigned long highBit = 1 << (HB_BITS_PER_WORD - 1);  
    uintptr_t ptrBase = HB_INDEX_TO_OFFSET(i) + liveHb->base;  
    while (garbage != 0) {  
        int shift = CLZ(garbage);  
        garbage &= ~(highBit >> shift);  
        *pb++ = (void *)(ptrBase + shift * HB_OBJECT_ALIGNMENT);  
    }  
    /* Make sure that there are always enough slots available */  
    /* for an entire word of 1s. */  
    if (pb >= &pointerBuf[NELEM(pointerBuf) - HB_BITS_PER_WORD]) {  
        (*callback)(pb - pointerBuf, pointerBuf, callbackArg);  
        pb = pointerBuf;  
    }  
}  
    }  
    if (pb > pointerBuf) {  
(*callback)(pb - pointerBuf, pointerBuf, callbackArg);  
    }  
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapBitmap.cpp中。

函数dvmHeapSweepUnmarkedObjects的实现逻辑很简单，它在在参数liveGB描述的Bitmap中位等于1但是在参数markHb描述的Bitmap中位却等于0的对象，也就是在上次GC后仍然存活的但是在当前GC后却不存活的对象，并且将这些对象的地址收集在变量pointerBuf描述的数组中，最后调用参数callback指向的回调函数批量回收保存在该数组中的对象。

参数callback指向的回调函数为sweepBitmapCallback，它的实现如下所示：

```c
static void sweepBitmapCallback(size_t numPtrs, void **ptrs, void *arg)  
{  
    assert(arg != NULL);  
    SweepContext *ctx = (SweepContext *)arg;  
    if (ctx->isConcurrent) {  
dvmLockHeap();  
    }  
    ctx->numBytes += dvmHeapSourceFreeList(numPtrs, ptrs);  
    ctx->numObjects += numPtrs;  
    if (ctx->isConcurrent) {  
dvmUnlockHeap();  
    }  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

如果当前执行的并行GC，参照前面的图1，此时Java堆是没有被锁定的，因此，函数sweepBitmapCallback在调用另外一个函数dvmHeapSourceFreeList来回收参数ptrs描述的Java堆内存时，需要先锁定堆，并且在回收完成后解锁堆。

函数dvmHeapSourceFreeList是真正进行内存回收的地方，它的实现如下所示：

```c
size_t dvmHeapSourceFreeList(size_t numPtrs, void **ptrs)  
{  
    ......  
    Heap* heap = ptr2heap(gHs, *ptrs);  
    size_t numBytes = 0;  
    if (heap != NULL) {  
mspace msp = heap->msp;  
......  
if (heap == gHs->heaps) {  
    // Count freed objects.  
    for (size_t i = 0; i < numPtrs; i++) {  
        ......  
        countFree(heap, ptrs[i], &numBytes);  
    }  
    // Bulk free ptrs.  
    mspace_bulk_free(msp, ptrs, numPtrs);  
} else {  
    // This is not an 'active heap'. Only do the accounting.  
    for (size_t i = 0; i < numPtrs; i++) {  
        ......  
        countFree(heap, ptrs[i], &numBytes);  
    }  
}  
    }  
    return numBytes;  
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapSource.cpp中。

函数dvmHeapSourceFreeList首先调用函数计算要释放的内存所属的堆heap，即Active堆还是Zygote堆。Active堆保存在gHs->heaps指向的Heap数组的第一个元素，根据这个信息就可以知道堆heap是Active堆还是Zygote堆。

对于Active堆，函数dvmHeapSourceFreeList首先是对每一块要释放的内存调用函数countFree进行计账，然后再调用C库提供的接口mspace_build_free批量释放参数ptrs指向的一系列内存块。

对于Zygote堆，函数dvmHeapSourceFreeList仅仅是对每一块要释放的内存调用函数countFree进行计账，但是并没有真正释放对应的内存块。这是因为Zygote堆是Zygote进程和所有的应用程序进程共享。一旦某一个进程对它进行了写类型的操作，那么就会导致对应的内存块不再在Zygote进程和应用程序进程之间进行共享。这样对Zygote堆的内存块进行释放反而会增加物理内存的开销。

dvmHeapFinishMarkStep

函数dvmHeapFinishMarkStep用来执行GC清理工作，它的实现如下所示：

```c
void dvmHeapFinishMarkStep()  
{  
    GcMarkContext *ctx = &gDvm.gcHeap->markContext;  
  
    /* The mark bits are now not needed. 
     */  
    dvmHeapSourceZeroMarkBitmap();  
  
    /* Clean up everything else associated with the marking process. 
     */  
    destroyMarkStack(&ctx->stack);  
  
    ctx->finger = NULL;  
}  
```
这个函数定义在文件dalvik/vm/alloc/MarkSweep.cpp中。

函数dvmHeapFinishMarkStep做三个清理工作。一是调用函数dvmHeapSourceZeroMarkBitmap重置Mark Bitmap；二是调用函数destroyMarkStack销毁Mark Stack；三是将递归标记对象过程中使用到的finger值设置为NULL。

dvmHeapSourceGrowForUtilization

函数dvmHeapSourceGrowForUtilization用来将堆的大小设置为理想值，它的实现如下所示：

```c
/* 
 * Given the current contents of the active heap, increase the allowed 
 * heap footprint to match the target utilization ratio.  This 
 * should only be called immediately after a full mark/sweep. 
 */  
void dvmHeapSourceGrowForUtilization()  
{  
    HS_BOILERPLATE();  
  
    HeapSource *hs = gHs;  
    Heap* heap = hs2heap(hs);  
  
    /* Use the current target utilization ratio to determine the 
     * ideal heap size based on the size of the live set. 
     * Note that only the active heap plays any part in this. 
     * 
     * Avoid letting the old heaps influence the target free size, 
     * because they may be full of objects that aren't actually 
     * in the working set.  Just look at the allocated size of 
     * the current heap. 
     */  
    size_t currentHeapUsed = heap->bytesAllocated;  
    size_t targetHeapSize = getUtilizationTarget(hs, currentHeapUsed);  
  
    /* The ideal size includes the old heaps; add overhead so that 
     * it can be immediately subtracted again in setIdealFootprint(). 
     * If the target heap size would exceed the max, setIdealFootprint() 
     * will clamp it to a legal value. 
     */  
    size_t overhead = getSoftFootprint(false);  
    setIdealFootprint(targetHeapSize + overhead);  
  
    size_t freeBytes = getAllocLimit(hs);  
    if (freeBytes < CONCURRENT_MIN_FREE) {  
/* Not enough free memory to allow a concurrent GC. */  
heap->concurrentStartBytes = SIZE_MAX;  
    } else {  
heap->concurrentStartBytes = freeBytes - CONCURRENT_START;  
    }  
  
    /* Mark that we need to run finalizers and update the native watermarks 
     * next time we attempt to register a native allocation. 
     */  
    gHs->nativeNeedToRunFinalization = true;  
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapSource.cpp中。

每次GC执行完成之后，都需要根据预先设置的目标堆利用率和已经分配出去的内存字节数计算得到理想的堆大小。注意，已经分配出去的内存字节数只考虑在Active堆上分配出去的字节数。得到了Active堆已经分配出去的字节数currentHeapUsed之后，就可以调用函数getUtilizationTarget来计算Active堆的理想大小targetHeapSize了。

函数getUtilizationTarget的实现如下所示：

```c
/* 
 * Given the size of a live set, returns the ideal heap size given 
 * the current target utilization and MIN/MAX values. 
 */  
static size_t getUtilizationTarget(const HeapSource* hs, size_t liveSize)  
{  
    /* Use the current target utilization ratio to determine the 
     * ideal heap size based on the size of the live set. 
     */  
    size_t targetSize = (liveSize / hs->targetUtilization) * HEAP_UTILIZATION_MAX;  
  
    /* Cap the amount of free space, though, so we don't end up 
     * with, e.g., 8MB of free space when the live set size hits 8MB. 
     */  
    if (targetSize > liveSize + hs->maxFree) {  
targetSize = liveSize + hs->maxFree;  
    } else if (targetSize < liveSize + hs->minFree) {  
targetSize = liveSize + hs->minFree;  
    }  
    return targetSize;  
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapSource.cpp中。

堆的目标利用率保存在hs->targetUtilization，它是以HEAP_UTILIZATION_MAX为基数计算出来得到的一个百分比，因此这里计算堆的理想大小时，需要乘以HEAP_UTILIZATION_MAX。

计算出来的堆理想大小targetSize要满足空闲内存不能大于预先设定的最大值（hs->maxFree）以及不能小于预先设定的最小值（hs->minFree）。否则的话，就要进行相应的调整。

回到函数dvmHeapSourceGrowForUtilization中，得到了Active堆的理想大小targetHeapSize之后，还要以参数false调用函数getSoftFootprint得到Zygote堆的大小overhead。将targetHeapSize和overhead相加，将得到的结果调用函数setIdealFootprint设置Java堆的大小。由此我们就可以知道，函数setIdealFootprint设置的整个Java堆的大小，而不是Active堆的大小，因此前面需要得到Zygote堆的大小。

函数dvmHeapSourceGrowForUtilization接下来调用函数getAllocLimit得到堆当前允许分配的最大值freeBytes，然后计算并行GC触发的条件。从前面[Dalvik虚拟机为新创建对象分配内存的过程分析](http://blog.csdn.net/luoshengyang/article/details/41688319)一文可以知道，当Active堆heap当前已经分配的大小超过heap->concurrentStartBytes时，就会触发并行GC。

计算并行GC触发条件时，需要用到CONCURRENT_MIN_FREE和CONCURRENT_START两个值，它们的定义如下所示：

```c
/* Start a concurrent collection when free memory falls under this 
 * many bytes. 
 */  
\#define CONCURRENT_START (128 << 10)  
  
/* The next GC will not be concurrent when free memory after a GC is 
 * under this many bytes. 
 */  
#define CONCURRENT_MIN_FREE (CONCURRENT_START + (128 << 10))  
```
这两个宏定义在文件dalvik/vm/alloc/HeapSource.cpp中。

也就是说，CONCURRENT_MIN_FREE定义为256K，而CONCURRENT_START定义为128K。

由此我们就可以知道，当Active堆允许分配的内存小于256K时，禁止执行并行GC，而当Active堆允许分配的内存大于等于256K，并且剩余的空闲内存小于128K，就会触发并行GC。

函数dvmHeapSourceGrowForUtilization最后是将gHs->nativeNeedToRunFinalization的值设置为true，这样当以后我们调用dalvik.system.VMRuntime类的成员函数registerNativeAllocation注册Native内存分配数时，就会触发java.lang.System类的静态成员函数runFinalization被调用，从而使得那些定义了成员函数finalize又即将被回收的对象执行成员函数finalize。

至此，我们就分析完成Dalvik虚拟机的垃圾收集过程了。[android](http://lib.csdn.net/base/android) 5.0之后，Dalvik虚拟机已经被ART运行时替换了。我们这里分析Dalvik虚拟机垃圾收集过程的目的，是为了能够更好地理解ART运行时的垃圾收集机制，因为两者有很多相通的东西。同时，Dalvik虚拟机的垃圾收集过程要比ART运行时的垃圾收集过程简单。因此，先学习Dalvik虚拟机的垃圾收集过程，可以使得我们循序渐进地更好学习ART运行时的垃圾收集过程。在接下来的一系列文章，我们就将开始分析ART运行时的垃圾收集机制，敬请关注！更多的信息也可以关注老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)。