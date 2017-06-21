使用C/C++开发应用程序最令头痛的问题就是内存管理。慎不留神，要么内存泄漏，要么内存破坏。虚拟机要解决的问题之一就是帮助应用程序自动分配和释放内存。为了达到这个目的，虚拟机在启动的时候向[操作系统](http://lib.csdn.net/base/operatingsystem)申请一大块内存当作对象堆。之后当应用程序创建对象时，虚拟机就会在堆上分配合适的内存块。而当对象不再使用时，虚拟机就会将它占用的内存块归还给堆。Dalvik虚拟机也不例外，本文就分析它的[Java](http://lib.csdn.net/base/java)堆创建过程。

老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)，欢迎关注！

《[Android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

从前面[Dalvik虚拟机垃圾收集机制简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/41338251)一文可以知道，在Dalvik虚拟机中，Java堆实际上是由一个Active堆和一个Zygote堆组成的，如图1所示：

![img](http://img.blog.csdn.net/20141129010759802?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图1 Dalvik虚拟机的Java堆

其中，Zygote堆用来管理Zygote进程在启动过程中预加载和创建的各种对象，而Active堆是在Zygote进程fork第一个子进程之前创建的。之后无论是Zygote进程还是其子进程，都在Active堆上进行对象分配和释放。这样做的目的是使得Zygote进程和其子进程最大限度地共享Zygote堆所占用的内存。

为了管理Java堆，Dalvik虚拟机需要一些辅助[数据结构](http://lib.csdn.net/base/datastructure)，包括一个Card Table、两个Heap Bitmap和一个Mark Stack。Card Table是为了记录在垃圾收集过程中对象的引用情况的，以便可以实现Concurrent G。图1的两个Heap Bitmap，一个称为Live Heap Bitmap，用来记录上次GC之后，还存活的对象，另一个称为Mark Heap Bitmap，用来记录当前GC中还存活的对象。这样，上次GC后存活的但是当前GC不存活的对象，就是需要释放的对象。Davlk虚拟机使用标记-清除（Mark-Sweep）[算法](http://lib.csdn.net/base/datastructure)进行GC。在标记阶段，通过一个Mark Stack来实现递归检查被引用的对象，即在当前GC中存活的对象。有了这个Mark Stack，就可以通过循环来模拟函数递归调用。

Dalvik虚拟机Java堆的创建过程实际上就是上面分析的各种数据结构的创建过程，它们是在Dalvik虚拟机启动的过程中创建的。接下来，我们就详细分析这个过程。

从前面[Dalvik虚拟机的启动过程分析](http://blog.csdn.net/luoshengyang/article/details/8885792)一文可以知道，Dalvik虚拟机在启动的过程中，会通过调用函数dvmGcStartup来创建Java堆，它的实现如下所示：

```c
bool dvmGcStartup()  
{  
    dvmInitMutex(&gDvm.gcHeapLock);  
    pthread_cond_init(&gDvm.gcHeapCond, NULL);  
    return dvmHeapStartup();  
}  
```
这个函数定义在文件dalvik/vm/alloc/Alloc.cpp中。

函数dvmGcStartup首先是分别初始化一个锁和一个条件变量，它们都是用来保护堆的并行访问的，接着再调用另外一个函数dvmHeapStartup来创建Java堆。

函数dvmHeapStartup的实现如下所示：

```c
bool dvmHeapStartup()  
{  
    GcHeap *gcHeap;  
  
    if (gDvm.heapGrowthLimit == 0) {  
gDvm.heapGrowthLimit = gDvm.heapMaximumSize;  
    }  
  
    gcHeap = dvmHeapSourceStartup(gDvm.heapStartingSize,  
                          gDvm.heapMaximumSize,  
                          gDvm.heapGrowthLimit);  
    ......  
  
    gDvm.gcHeap = gcHeap;  
    ......  
  
    if (!dvmCardTableStartup(gDvm.heapMaximumSize, gDvm.heapGrowthLimit)) {  
LOGE_HEAP("card table startup failed.");  
return false;  
    }  
  
    return true;  
}  
```
这个函数定义在文件dalvik/vm/alloc/Heap.cpp中。

gDvm是一个类型为DvmGlobals的全局变量，它通过各个成员变量记录了Dalvik虚拟机的各种信息。这里涉及到三个重要与Java堆相关的信息，分别是Java堆的起始大小（Starting Size）、最大值（Maximum Size）和增长上限值（Growth Limit）。在启动Dalvik虚拟机的时候，我们可以分别通过-Xms、-Xmx和-XX:HeapGrowthLimit三个选项来指定上述三个值。

Java堆的起始大小（Starting Size）指定了Davlik虚拟机在启动的时候向系统申请的物理内存的大小。后面再根据需要逐渐向系统申请更多的物理内存，直到达到最大值（Maximum Size）为止。这是一种按需要分配策略，可以避免内存浪费。在默认情况下，Java堆的起始大小（Starting Size）和最大值（Maximum Size）等于4M和16M。但是厂商会通过dalvik.vm.heapstartsize和dalvik.vm.heapsize这两个属性将它们设置为合适设备的值的。

注意，虽然Java堆使用的物理内存是按需要分配的，但是它使用的虚拟内存的总大小却是需要在Dalvik启动的时候就确定的。这个虚拟内存的大小就等于Java堆的最大值（Maximum Size）。想象一下，如果不这样做的话，会出现什么情况。假设开始时创建的虚拟内存小于Java堆的最大值（Maximum Size），由于实际情况是允许虚拟内存的大小是达到Java堆的最大值（Maximum Size）的，因此，当开始时创建的虚拟内存无法满足需求时，那么就需要重新创建另外一块更大的虚拟内存。这样就需要将之前的虚拟内存的内容拷贝到新创建的更大的虚拟内存去，并且还要相应地修改各种辅助数据结构。这样太麻烦了，而且效率也太低了。因此就在一开始的时候，就创建一块与Java堆的最大值（Maximum Size）相等的虚拟内存。

但是，Dalvik虚拟机又希望能够动态地调整Java堆的可用最大值，于是就出现了一个称为增长上限的值（Growth Limit）。这个增长上限值（Growth Limit），我们可以认为它是Java堆大小的软限制，而前面所描述的最大值（Maximum Size），是Java堆大小的硬限制。通过动态地调整增长上限值（Growth Limit），就可以实现动态调整Java堆的可用最大值，但是这个增长上限值必须要小于等于最大值（Maximum Size）。从函数dvmHeapStartup的实现可以知道，如果没有指定Java堆的增长上限的值（Growth Limit），那么它的值就等于Java堆的最大值（Maximum Size）。

事实上，在全局变量gDvm中，除了上面提到的三个信息之外，还有三种信息是与Java堆相关的，它们分别是堆最小空闲值（Min Free）、堆最大空闲值（Max Free）和堆目标利用率（Target Utilization）。这三个值可以分别通过Dalvik虚拟机的启动选项-XX:HeapMinFree、-XX:HeapMaxFree和-XX:HeapTargetUtilization来指定。它们用来确保每次GC之后，Java堆已经使用和空闲的内存有一个合适的比例，这样可以尽量地减少GC的次数。举个例子说，堆的利用率为U，最小空闲值为MinFree字节，最大空闲值为MaxFree字节。假设在某一次GC之后，存活对象占用内存的大小为LiveSize。那么这时候堆的理想大小应该为(LiveSize / U)。但是(LiveSize / U)必须大于等于(LiveSize + MinFree)并且小于等于(LiveSize + MaxFree)。

了解了这些与Java堆大小相关的信息之后，我们回到函数dvmGcStartup中，可以清楚看到，它先是调用函数dvmHeapSourceStartup来创建一个Java堆，接着再调用函数dvmCardTableStartup来为该Java堆创建一个Card Table。接下来我们先分析函数dvmHeapSourceStartup的实现，接着再分析函数dvmCardTableStartup的实现。

函数dvmHeapSourceStartup的实现如下所示：

```c
GcHeap* dvmHeapSourceStartup(size_t startSize, size_t maximumSize,  
                     size_t growthLimit)  
{  
    GcHeap *gcHeap;  
    HeapSource *hs;  
    mspace msp;  
    size_t length;  
    void *base;  
    ......  
  
    /* 
     * Allocate a contiguous region of virtual memory to subdivided 
     * among the heaps managed by the garbage collector. 
     */  
    length = ALIGN_UP_TO_PAGE_SIZE(maximumSize);  
    base = dvmAllocRegion(length, PROT_NONE, gDvm.zygote ? "dalvik-zygote" : "dalvik-heap");  
    ......  
  
    /* Create an unlocked dlmalloc mspace to use as 
     * a heap source. 
     */  
    msp = createMspace(base, kInitialMorecoreStart, startSize);  
    ......  
  
    gcHeap = (GcHeap *)calloc(1, sizeof(*gcHeap));  
    ......  
  
    hs = (HeapSource *)calloc(1, sizeof(*hs));  
    ......  
  
    hs->targetUtilization = gDvm.heapTargetUtilization * HEAP_UTILIZATION_MAX;  
    hs->minFree = gDvm.heapMinFree;  
    hs->maxFree = gDvm.heapMaxFree;  
    hs->startSize = startSize;  
    hs->maximumSize = maximumSize;  
    hs->growthLimit = growthLimit;  
    ......   
    hs->numHeaps = 0;  
    ......  
    hs->heapBase = (char *)base;  
    hs->heapLength = length;  
    ......  
  
    if (!addInitialHeap(hs, msp, growthLimit)) {  
......  
    }  
    if (!dvmHeapBitmapInit(&hs->liveBits, base, length, "dalvik-bitmap-1")) {  
......  
    }  
    if (!dvmHeapBitmapInit(&hs->markBits, base, length, "dalvik-bitmap-2")) {  
......  
    }  
    if (!allocMarkStack(&gcHeap->markContext.stack, hs->maximumSize)) {  
......  
    }  
    gcHeap->markContext.bitmap = &hs->markBits;  
    gcHeap->heapSource = hs;  
  
    gHs = hs;  
    return gcHeap;  
  
    ......  
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapSource.cpp中。

函数dvmHeapSourceStartup的执行过程如下所示：

1.将参数maximum指定的最大堆大小对齐到内存页边界，得到结果为length，并且调用函数dvmAllocRegion分配一块大小等于length的匿名共享内存块，起始地址为base。这块匿名共享内存即作为Dalvik虚拟机的Java堆。

2.调用函数createMspace将前面得到的匿名共享内存块封装为一个mspace，以便后面可以通过C库得供的mspace_malloc和mspace_bulk_free等函数来管理Java堆。这个mspace的起始大小为Java堆的起始大小，这意味着一开始在该mspace上能够分配的内存不能超过Java堆的起始大小。不过后面我们动态地调整这个mspace的大小，使得它可以使用更多的内存，但是不能超过Java堆的最大值。

3.分配一个GcHeap结构体gcHeap和一个HeapSource结构体hs，用来维护Java堆的信息，包括Java堆的目标利用率、最小空闲值、最大空闲值、起始大小、最大值、增长上限值、堆个数、起始地址和大小等信信息。

4.调用函数addInitialHeap在前面得到的匿名共享内存上创建一个Active堆。这个Active堆的最大值被设置为Java堆的起始大小。

5.调用函数dvmHeapBitmapInit创建和初始化一个Live Bitmap和一个Mark Bitmap，它们在GC时会用得到。

6.调用函数allockMarkStack创建和初始化一个Mark Stack，它在GC时也会用到。

7.将前面创建和初始化好的Mark Bitmap和HeapSource结构体hs保存在前面创建的GcHeap结构体gcHeap中，并且将该GcHeap结构体gcHeap返回给调用者。同时，HeapSource结构体hs也会保存在全局变量gHs中。 

为了更好地对照图2来理解函数dvmHeapSourceStartup所做的事情，接下来我们详细分析上述提到的关键函数dvmAllocRegion、createMspace、addInitialHeap、dvmHeapBitmapInit和allockMarkStack的实现。

函数dvmAllocRegion的实现如下所示：

```c
void *dvmAllocRegion(size_t byteCount, int prot, const char *name) {  
    void *base;  
    int fd, ret;  
  
    byteCount = ALIGN_UP_TO_PAGE_SIZE(byteCount);  
    fd = ashmem_create_region(name, byteCount);  
    if (fd == -1) {  
return NULL;  
    }  
    base = mmap(NULL, byteCount, prot, MAP_PRIVATE, fd, 0);  
    ret = close(fd);  
    if (base == MAP_FAILED) {  
return NULL;  
    }  
    if (ret == -1) {  
munmap(base, byteCount);  
return NULL;  
    }  
    return base;  
}  
```
这个函数定义在文件dalvik/vm/Misc.cpp中。

从这里就可以清楚地看出，函数dvmAllocRegion所做的事情就是调用函数ashmem_create_region来创建一块匿名共享内存。关于[android](http://lib.csdn.net/base/android)系统的匿名共享内存，可以参考前面[Android系统匿名共享内存Ashmem（Anonymous Shared Memory）简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6651971)一文。

函数createMspace的实现如下所示：

```c
static mspace createMspace(void* begin, size_t morecoreStart, size_t startingSize)  
{  
    // Clear errno to allow strerror on error.  
    errno = 0;  
    // Allow access to inital pages that will hold mspace.  
    mprotect(begin, morecoreStart, PROT_READ | PROT_WRITE);  
    // Create mspace using our backing storage starting at begin and with a footprint of  
    // morecoreStart. Don't use an internal dlmalloc lock. When morecoreStart bytes of memory are  
    // exhausted morecore will be called.  
    mspace msp = create_mspace_with_base(begin, morecoreStart, false /*locked*/);  
    if (msp != NULL) {  
// Do not allow morecore requests to succeed beyond the starting size of the heap.  
mspace_set_footprint_limit(msp, startingSize);  
    } else {  
ALOGE("create_mspace_with_base failed %s", strerror(errno));  
    }  
    return msp;  
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapSource.cpp中。

参数begin指向前面创建的一块匿名共享内存的起始地址，也就是Java堆的起始地址，函数createMspace通过C库提供的函数create_mspace_with_base将该块匿名共享内存封装成一个mspace，并且通过调用C库提供的函数mspace_set_footprint_limit设置该mspace的大小为Java堆的起始大小。

函数addInitialHeap的实现如下所示：

```c
static bool addInitialHeap(HeapSource *hs, mspace msp, size_t maximumSize)  
{  
    assert(hs != NULL);  
    assert(msp != NULL);  
    if (hs->numHeaps != 0) {  
return false;  
    }  
    hs->heaps[0].msp = msp;  
    hs->heaps[0].maximumSize = maximumSize;  
    hs->heaps[0].concurrentStartBytes = SIZE_MAX;  
    hs->heaps[0].base = hs->heapBase;  
    hs->heaps[0].limit = hs->heapBase + maximumSize;  
    hs->heaps[0].brk = hs->heapBase + kInitialMorecoreStart;  
    hs->numHeaps = 1;  
    return true;  
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapSource.cpp中。

在分析函数addInitialHeap的实现之前，我们先解释一下两个数据结构HeapSource和Heap。

在结构体HeapSource中，有一个类型为Heap的数组heaps，如下所示：

```c
struct HeapSource {  
    ......  
  
    /* The heaps; heaps[0] is always the active heap, 
     * which new objects should be allocated from. 
     */  
    Heap heaps[HEAP_SOURCE_MAX_HEAP_COUNT];  
  
    /* The current number of heaps. 
     */  
    size_t numHeaps;  
    ......  
  
};  
```
这个结构体定义在文件dalvik/vm/alloc/HeapSource.cpp中。

这个Heap数组最多有HEAP_SOURCE_MAX_HEAP_COUNT个Heap，并且当前拥有的Heap个数记录在numpHeaps中。

HEAP_SOURCE_MAX_HEAP_COUNT是一个宏，定义为2，如下所示：

```c
/* The largest number of separate heaps we can handle. 
 */  
\#define HEAP_SOURCE_MAX_HEAP_COUNT 2  

这个宏定义在文件dalvik/vm/alloc/HeapSource.h中。

这意味着Dalvik虚拟机的Java堆最多可以划分为两个Heap，就是图1所示的Active堆和Zygote堆。

结构Heap的定义如下所示：

​```c
struct Heap {  
    /* The mspace to allocate from. 
     */  
    mspace msp;  
  
    /* The largest size that this heap is allowed to grow to. 
     */  
    size_t maximumSize;  
  
    /* Number of bytes allocated from this mspace for objects, 
     * including any overhead.  This value is NOT exact, and 
     * should only be used as an input for certain heuristics. 
     */  
    size_t bytesAllocated;  
  
    /* Number of bytes allocated from this mspace at which a 
     * concurrent garbage collection will be started. 
     */  
    size_t concurrentStartBytes;  
  
    /* Number of objects currently allocated from this mspace. 
     */  
    size_t objectsAllocated;  
  
    /* 
     * The lowest address of this heap, inclusive. 
     */  
    char *base;  
    /* 
     * The highest address of this heap, exclusive. 
     */  
    char *limit;  
  
    /* 
     * If the heap has an mspace, the current high water mark in 
     * allocations requested via dvmHeapSourceMorecore. 
     */  
    char *brk;  
};  
```
这个结构体定义在文件dalvik/vm/alloc/HeapSource.cpp中。

结构体Heap用来描述一个堆，它的各个成员变量的含义如下所示：

| 成员变量                 | 说明                      |
| :------------------- | :---------------------- |
| msp                  | 描述堆所使用内存块               |
| maximumSize          | 描述堆可以使用的最大内存值           |
| bytesAllocated       | 描述堆已经分配的字节数             |
| concurrentStartBytes | 描述堆已经分配的内存达到指定值就要触发并行GC |
| objectsAllocated     | 描述已经分配的对象数              |
| base                 | 描述堆所使用的内存块的起始地址         |
| limit                | 描述堆所使用的内存块的结束地址         |
| brk                  | 描述当前堆所分配的最大内存值          |

回到函数addInitialHeap中，参数hs和msp指向的是在函数dvmHeapSourceStartup中创建的HeapSource结构体和mspace内存对象，而参数maximumSize描述的Java堆的增长上限值。

通过函数addInitialHeap的实现就可以看出，Dalvik虚拟机在启动的时候，实际上只创建了一个Heap。这个Heap就是我们在图1中所说的Active堆，它开始的时候管理的是整个Java堆。但是在图1中，我们说Java堆实际上还包含有一个Zygote堆的，那么这个Zygote堆是怎么来的呢？

从前面[Dalvik虚拟机进程和线程的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8923484)一文可以知道，Zygote进程会通过调用函数forkAndSpecializeCommon来fork子进程，其中与Dalvik虚拟机Java堆相关的逻辑如下所示：

```c
static pid_t forkAndSpecializeCommon(const u4* args, bool isSystemServer)  
{  
    pid_t pid;  
    ......  
  
    if (!dvmGcPreZygoteFork()) {  
......  
    }  
    ......  
  
    pid = fork();  
  
    if (pid == 0) {  
......  
    } else {  
......  
    }  
  
    return pid;  
}  
```
这个函数定义在文件dalvik/vm/native/dalvik_system_Zygote.cpp中。

从这里就可以看出，Zygote进程在fork子进程之前，会调用函数dvmGcPreZygoteFork来处理一下Dalvik虚拟机Java堆。接下来我们就看看函数dvmGcPreZygoteFork都做了哪些事情。

函数dvmGcPreZygoteFork的实现如下所示：

```c
bool dvmGcPreZygoteFork()  
{  
    return dvmHeapSourceStartupBeforeFork();  
}  

这个函数定义在文件dalvik/vm/alloc/Alloc.cpp中。

函数dvmGcPreZygoteFork只是简单地封装了对另外一个函数dvmHeapSourceStartupBeforeFork的调用，后者的实现如下所示：

​```c
bool dvmHeapSourceStartupBeforeFork()  
{  
    HeapSource *hs = gHs; // use a local to avoid the implicit "volatile"  
  
    HS_BOILERPLATE();  
  
    assert(gDvm.zygote);  
  
    if (!gDvm.newZygoteHeapAllocated) {  
/* Ensure heaps are trimmed to minimize footprint pre-fork. 
*/  
trimHeaps();  
/* Create a new heap for post-fork zygote allocations.  We only 
 * try once, even if it fails. 
 */  
ALOGV("Splitting out new zygote heap");  
gDvm.newZygoteHeapAllocated = true;  
return addNewHeap(hs);  
    }  
    return true;  
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapSource.cpp中。

前面我们在分析函数dvmHeapSourceStartup的实现时提到，全局变量gHs指向的是一个HeapSource结构体，它描述了Dalvik虚拟机Java堆的信息。同时，gDvm也是一个全局变量，它的类型为DvmGlobals。gDvm指向的DvmGlobals结构体的成员变量newZygoteHeapAllocated的值被初始化为false。因此，当函数dvmHeapSourceStartupBeforeFork第一次被调用时，它会先调用函数trimHeaps来将Java堆中没有使用到的内存归还给系统，接着再调用函数addNewHeap来创建一个新的Heap。这个新的Heap就是图1所说的Zygote堆了。

由于函数dvmHeapSourceStartupBeforeFork第一次被调用之后，gDvm指向的DvmGlobals结构体的成员变量newZygoteHeapAllocated的值就会被修改为true，因此起到的效果就是以后Zygote进程对函数dvmHeapSourceStartupBeforeFork的调用都是无用功。这也意味着Zygote进程只会在fork第一个子进程的时候，才会将Java堆划一分为二来管理。

接下来我们就继续分析函数trimHeaps和addNewHeap的实现，以便更好地理解Dalvik虚拟机是如何管理Java堆的。

函数trimHeaps的实现如下所示：

```c
/* 
 * Return unused memory to the system if possible. 
 */  
static void trimHeaps()  
{  
    HS_BOILERPLATE();  
  
    HeapSource *hs = gHs;  
    size_t heapBytes = 0;  
    for (size_t i = 0; i < hs->numHeaps; i++) {  
Heap *heap = &hs->heaps[i];  
  
/* Return the wilderness chunk to the system. */  
mspace_trim(heap->msp, 0);  
  
/* Return any whole free pages to the system. */  
mspace_inspect_all(heap->msp, releasePagesInRange, &heapBytes);  
    }  
  
    /* Same for the native heap. */  
    dlmalloc_trim(0);  
    size_t nativeBytes = 0;  
    dlmalloc_inspect_all(releasePagesInRange, &nativeBytes);  
  
    LOGD_HEAP("madvised %zd (GC) + %zd (native) = %zd total bytes",  
    heapBytes, nativeBytes, heapBytes + nativeBytes);  
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapSource.cpp中。

函数trimHeaps对Dalvik虚拟机使用的Java堆和默认Native堆做了同样的两件事情。

第一件事情是调用C库提供的函数mspace_trim/dlmalloc_trim来将没有使用到的虚拟内存和物理内存归还给系统，这是通过系统调用mremap来实现的。

第二件事情是调用C库提供的函数mspace_inspect_all/dlmalloc_inspect_all将不能使用的内存碎片对应的物理内存归还给系统，这是通过系统调用madvise来实现的。注意，在此种情况下，只能归还无用的物理内存，而不能归还无用的虚拟内存。因为归还内存碎片对应的虚拟内存会使得堆的整体虚拟地址不连续。

函数addNewHeap的实现如下所示：

```c
static bool addNewHeap(HeapSource *hs)  
{  
    Heap heap;  
  
    assert(hs != NULL);  
    if (hs->numHeaps >= HEAP_SOURCE_MAX_HEAP_COUNT) {  
......  
return false;  
    }  
  
    memset(&heap, 0, sizeof(heap));  
  
    /* 
     * Heap storage comes from a common virtual memory reservation. 
     * The new heap will start on the page after the old heap. 
     */  
    char *base = hs->heaps[0].brk;  
    size_t overhead = base - hs->heaps[0].base;  
    assert(((size_t)hs->heaps[0].base & (SYSTEM_PAGE_SIZE - 1)) == 0);  
  
    if (overhead + hs->minFree >= hs->maximumSize) {  
......  
return false;  
    }  
    size_t morecoreStart = SYSTEM_PAGE_SIZE;  
    heap.maximumSize = hs->growthLimit - overhead;  
    heap.concurrentStartBytes = hs->minFree - CONCURRENT_START;  
    heap.base = base;  
    heap.limit = heap.base + heap.maximumSize;  
    heap.brk = heap.base + morecoreStart;  
    if (!remapNewHeap(hs, &heap)) {  
      return false;  
    }  
    heap.msp = createMspace(base, morecoreStart, hs->minFree);  
    if (heap.msp == NULL) {  
return false;  
    }  
  
    /* Don't let the soon-to-be-old heap grow any further. 
     */  
    hs->heaps[0].maximumSize = overhead;  
    hs->heaps[0].limit = base;  
    mspace_set_footprint_limit(hs->heaps[0].msp, overhead);  
  
    /* Put the new heap in the list, at heaps[0]. 
     * Shift existing heaps down. 
     */  
    memmove(&hs->heaps[1], &hs->heaps[0], hs->numHeaps * sizeof(hs->heaps[0]));  
    hs->heaps[0] = heap;  
    hs->numHeaps++;  
  
    return true;  
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapSource.cpp中。

函数addNewHeap所做的事情实际上就是将前面创建的Dalvik虚拟机Java堆一分为二，得到两个Heap。

在划分之前，HeadSource结构体hs只有一个Heap，如图2所示：

![img](http://img.blog.csdn.net/20141130035810421?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图2 Dalvik虚拟机Java堆一分为二之前

接下来在未使用的Dalvik虚拟机Java堆中创建另外一个Heap，如图3所示：

![img](http://img.blog.csdn.net/20141130035902922?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图3 在未使用的Dalvik虚拟机Java堆中创建一个新的Heap

最后调整HeadSource结构体hs的heaps数组，即交heaps[0]和heaps[1]的值，结果如图4所示：

![img](http://img.blog.csdn.net/20141130040136046?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图4 Dalvik虚拟机Java堆一分为二之后

其中，heaps[1]就是我们在图1中所说的Zygote堆，而heaps[0]就是我们在图1中所说的Active堆。以后无论是Zygote进程，还是Zygote子进程，需要分配对象时，都在Active堆上进行。这样就可以使得Zygote堆最大限度地在Zygote进程及其子进程中共享。

这样我们就分析完了函数addInitialHeap及其相关函数的实现，接下来我们继续分析函数dvmHeapBitmapInit和allocMarkStack的实现。

函数dvmHeapBitmapInit的实现如下所示：

```c
bool dvmHeapBitmapInit(HeapBitmap *hb, const void *base, size_t maxSize,  
               const char *name)  
{  
    void *bits;  
    size_t bitsLen;  
  
    assert(hb != NULL);  
    assert(name != NULL);  
    bitsLen = HB_OFFSET_TO_INDEX(maxSize) * sizeof(*hb->bits);  
    bits = dvmAllocRegion(bitsLen, PROT_READ | PROT_WRITE, name);  
    if (bits == NULL) {  
ALOGE("Could not mmap %zd-byte ashmem region '%s'", bitsLen, name);  
return false;  
    }  
    hb->bits = (unsigned long *)bits;  
    hb->bitsLen = hb->allocLen = bitsLen;  
    hb->base = (uintptr_t)base;  
    hb->max = hb->base - 1;  
    return true;  
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapBitmap.cpp中。

参数hb指向一个HeapBitmap结构体，这个结构体正是函数dvmHeapBitmapInit要进行初始化的。参数base和maxSize描述的是Java堆的起始地址和大小。另外一个参数name描述的是参数hb指向的HeapBitmap结构体的名称。

在分析函数dvmHeapBitmapInit的实现之前，我们先来了解一下结构体HeapBitmap的定义，如下所示：

```c
struct HeapBitmap {  
    /* The bitmap data, which points to an mmap()ed area of zeroed 
     * anonymous memory. 
     */  
    unsigned long *bits;  
  
    /* The size of the used memory pointed to by bits, in bytes.  This 
     * value changes when the bitmap is shrunk. 
     */  
    size_t bitsLen;  
  
    /* The real size of the memory pointed to by bits.  This is the 
     * number of bytes we requested from the allocator and does not 
     * change. 
     */  
    size_t allocLen;  
  
    /* The base address, which corresponds to the first bit in 
     * the bitmap. 
     */  
    uintptr_t base;  
  
    /* The highest pointer value ever returned by an allocation 
     * from this heap.  I.e., the highest address that may correspond 
     * to a set bit.  If there are no bits set, (max < base). 
     */  
    uintptr_t max;  
};  
```
这个结构体定义在文件dalvik/vm/alloc/HeapBitmap.h。

代码对HeapBitmap结构体的各个成员变量的含义已经有很详细的注释，其中最重要的就是成员变量bits指向的一个类型为unsigned long的数组，这个数组的每一个bit都用来标记一个对象是否存活。

回到函数dvmHeapBitmapInit中，Java堆的起始地址为base，大小为maxSize，由此我们就知道，在Java堆上创建的对象的地址范围为[base, maxSize)。但是通过C库提供的mspace_malloc来在Java堆分配内存时，得到的内存地址是以8字节对齐的。这意味着我们只需要(maxSize / 8)个bit来描述Java堆的对象。结构体HeapBitmap的成员变量bits是一个类型为unsigned long的数组，也就是说，数组中的每一个元素都可以描述sizeof(unsigned long)个对象的存活。在32位设备上，一个unsigned long占用32个bit，这意味着需要一个大小为(maxSize / 8 / 32)的unsigned long数组来描述Java堆对象的存活。如果换成字节数来描述的话，就是说我们需要一块大小为(maxSize / 8 / 32) × 4的内存块来描述一个大小为maxSize的Java堆对象。

Dalvik虚拟机提供了一些宏来描述对象地址与HeapBitmap结构体的成员变量bits所描述的unsigned long数组的关系，如下所示：

```c
#define HB_OBJECT_ALIGNMENT 8  
#define HB_BITS_PER_WORD (sizeof(unsigned long) * CHAR_BIT)  
  
/* <offset> is the difference from .base to a pointer address. 
 * <index> is the index of .bits that contains the bit representing 
 *         <offset>. 
 */  
#define HB_OFFSET_TO_INDEX(offset_) \  
    ((uintptr_t)(offset_) / HB_OBJECT_ALIGNMENT / HB_BITS_PER_WORD)  
#define HB_INDEX_TO_OFFSET(index_) \  
    ((uintptr_t)(index_) * HB_OBJECT_ALIGNMENT * HB_BITS_PER_WORD)  
  
#define HB_OFFSET_TO_BYTE_INDEX(offset_) \  
  (HB_OFFSET_TO_INDEX(offset_) * sizeof(*((HeapBitmap *)0)->bits))   
```
这些宏定义在文件dalvik/vm/alloc/HeapBitmap.h中。

假设我们知道了一个对象的地址为ptr，Java堆的起始地址为base，那么就可以计算得到一个偏移值offset。有了这个偏移值之后，就可以通过宏HB_OFFSET_TO_INDEX计算得到用来描述该对象存活的bit位于HeapBitmap结构体的成员变量bits所描述的unsigned long数组的索引index。有了这个index之后，我们就可以得到一个unsigned long值。接着再通过对象地址ptr的第4到第8位表示的数值为索引，在前面找到的unsigned long值取出相应的位，就可以得到该对象是否存活了。

相反，给出一个HeapBitmap结构体的成员变量bits所描述的unsigned long数组的索引index，我们可以通过宏HB_INDEX_TO_OFFSET找到一个偏移值offset，将这个偏移值加上Java堆的起始地址base，就可以得到一个Java对象的地址ptr。

第三个宏HB_OFFSET_TO_BYTE_INDEX借助宏HB_OFFSET_TO_INDEX来找出用来描述对象存活的bit在HeapBitmap结构体的成员变量bits所描述的内存块的字节索引。

有了上述的基础知识之后，函数dvmHeapBitmapInit的实现就一目了然了。

接下来我们再来看函数allocMarkStack的实现，如下所示：

```c
static bool allocMarkStack(GcMarkStack *stack, size_t maximumSize)  
{  
    const char *name = "dalvik-mark-stack";  
    void *addr;  
  
    assert(stack != NULL);  
    stack->length = maximumSize * sizeof(Object*) /  
(sizeof(Object) + HEAP_SOURCE_CHUNK_OVERHEAD);  
    addr = dvmAllocRegion(stack->length, PROT_READ | PROT_WRITE, name);  
    if (addr == NULL) {  
return false;  
    }  
    stack->base = (const Object **)addr;  
    stack->limit = (const Object **)((char *)addr + stack->length);  
    stack->top = NULL;  
    madvise(stack->base, stack->length, MADV_DONTNEED);  
    return true;  
}  
```
这个函数定义在文件vm/alloc/HeapSource.cpp中。

参数stack指向的是一个GcMarkStack结构体，这个结构体正是函数allocMarkStack要进行初始化的。参数maximumSize描述的是Java堆的大小。

同样是在分析函数allocMarkStack的实现之前，我们先来了解一下结构体GcMarkStack的定义，如下所示：

```c
struct GcMarkStack {  
    /* Highest address (exclusive) 
     */  
    const Object **limit;  
  
    /* Current top of the stack (exclusive) 
     */  
    const Object **top;  
  
    /* Lowest address (inclusive) 
     */  
    const Object **base;  
  
    /* Maximum stack size, in bytes. 
     */  
    size_t length;  
};  
```
这个结构体定义在文件dalvik/vm/alloc/MarkSweep.h中。

代码对HeapBitmap结构体的各个成员变量的含义已经有很详细的注释。总结来说，GcMarkStack通过一个Object数组来描述一个栈。这个Object数组的大小通过成员变量length来描述。成员变量base和limit分别描述栈的最低地址和最高地址，另外一个成员变量top指向栈顶。

回到函数allocMarkStack中，我们分析一下需要一个多大的栈来描述Java堆的所有对象。首先，每一个Java对象都是必须要从Object结构体继承下来的，这意味着每一个Java对象占用的内存都至少为sizeof(Object)。其次，通过C库提供的接口mspace_malloc在Java堆上为对象分配内存时，C库自己需要一些额外的内存来管理该块内存，例如用额外的4个字节来记录分配出去的内存块的大小。额外需要的内存大小通过宏HEAP_SOURCE_CHUNK_OVERHEAD来描述。最后，我们就可以知道，一个大小为maximumSize的Java堆，在最坏情况下，存在(maximumSize / (sizeof(Object) + HEAP_SOURCE_CHUNK_OVERHEAD))个对象。也就是说，GcMarkStack通过一个大小为(maximumSize / (sizeof(Object) + HEAP_SOURCE_CHUNK_OVERHEAD))的Object*数组来描述一个栈。如果换成字节数来描述的话，就是说我们需要一块大小为(maximumSize * sizeof(Object*) / (sizeof(Object) + HEAP_SOURCE_CHUNK_OVERHEAD))的内存块来描述一个GcMarkStack栈。

有了上述的基础知识之后，函数allocMarkStack的实现同样也一目了然了。

这样，函数dvmHeapSourceStartup及其相关的函数dvmAllocRegion、createMspace、addInitialHeap、dvmHeapBitmapInit和allockMarkStack的实现我们就分析完了，回到前面的函数dvmHeapStartup中，它调用函数dvmHeapSourceStartup创建完成Java堆及其相关的Heap Bitmap和Mark Stack之后，还需要继续调用函数dvmCardTableStartup来创建一个Card Table。这个Card Table在执行Concurrent GC时要使用到。

函数dvmCardTableStartup的实现如下所示：

```c
/* 
 * Maintain a card table from the the write barrier. All writes of 
 * non-NULL values to heap addresses should go through an entry in 
 * WriteBarrier, and from there to here. 
 * 
 * The heap is divided into "cards" of GC_CARD_SIZE bytes, as 
 * determined by GC_CARD_SHIFT. The card table contains one byte of 
 * data per card, to be used by the GC. The value of the byte will be 
 * one of GC_CARD_CLEAN or GC_CARD_DIRTY. 
 * 
 * After any store of a non-NULL object pointer into a heap object, 
 * code is obliged to mark the card dirty. The setters in 
 * ObjectInlines.h [such as dvmSetFieldObject] do this for you. The 
 * JIT and fast interpreters also contain code to mark cards as dirty. 
 * 
 * The card table's base [the "biased card table"] gets set to a 
 * rather strange value.  In order to keep the JIT from having to 
 * fabricate or load GC_DIRTY_CARD to store into the card table, 
 * biased base is within the mmap allocation at a point where it's low 
 * byte is equal to GC_DIRTY_CARD. See dvmCardTableStartup for details. 
 */  
  
/* 
 * Initializes the card table; must be called before any other 
 * dvmCardTable*() functions. 
 */  
bool dvmCardTableStartup(size_t heapMaximumSize, size_t growthLimit)  
{  
    size_t length;  
    void *allocBase;  
    u1 *biasedBase;  
    GcHeap *gcHeap = gDvm.gcHeap;  
    int offset;  
    void *heapBase = dvmHeapSourceGetBase();  
    assert(gcHeap != NULL);  
    assert(heapBase != NULL);  
    /* All zeros is the correct initial value; all clean. */  
    assert(GC_CARD_CLEAN == 0);  
  
    /* Set up the card table */  
    length = heapMaximumSize / GC_CARD_SIZE;  
    /* Allocate an extra 256 bytes to allow fixed low-byte of base */  
    allocBase = dvmAllocRegion(length + 0x100, PROT_READ | PROT_WRITE,  
                    "dalvik-card-table");  
    if (allocBase == NULL) {  
return false;  
    }  
    gcHeap->cardTableBase = (u1*)allocBase;  
    gcHeap->cardTableLength = growthLimit / GC_CARD_SIZE;  
    gcHeap->cardTableMaxLength = length;  
    biasedBase = (u1 *)((uintptr_t)allocBase -  
               ((uintptr_t)heapBase >> GC_CARD_SHIFT));  
    offset = GC_CARD_DIRTY - ((uintptr_t)biasedBase & 0xff);  
    gcHeap->cardTableOffset = offset + (offset < 0 ? 0x100 : 0);  
    biasedBase += gcHeap->cardTableOffset;  
    assert(((uintptr_t)biasedBase & 0xff) == GC_CARD_DIRTY);  
    gDvm.biasedCardTableBase = biasedBase;  
  
    return true;  
}  
```
这个函数定主在文件dalvik/vm/alloc/CardTable.cpp中。

参数heapMaximumSize和growthLimit描述的是Java堆的最大值和增长上限值。

在Dalvik虚拟机中，Card Table和Heap Bitmap的作用是类似的。区别在于：

Card Table不是使用一个bit来描述一个对象，而是用一个byte来描述GC_CARD_SIZE个对象；

Card Table不是用来描述对象的存活，而是用来描述在Concurrent GC的过程中被修改的对象，这些对象需要进行特殊处理。

全局变量gDvm的成员变量gcHeap指向了一个GcHeap结构体。在GcHeap结构体，通过cardTableBase、cardTableLength、cardTableMaxLength和cardTableOffset这四个成员变量来描述一个Card Table。它们的定义如下所示：

```c
struct GcHeap {  
    ......  
  
    /* GC's card table */  
    u1* cardTableBase;  
    size_t cardTableLength;  
    size_t cardTableMaxLength;  
    size_t cardTableOffset;  
    ......  
  
};  
```
这个结构体定义在文件dalvik/vm/alloc/HeapInternal.h中。

其中，成员变量cardTableBase和cardTableMaxLength描述的是创建的Card Table和起始地址和大小。成员变量cardTableLength描述的当前Card Table使用的大小。成员变量cardTableMaxLength和cardTableLength的关系就对应于Java堆的最大值(Maximum Size)和增长上限值(Growth Limit)的关系。

Card Table在真正使用的时候，并不是从成员变量cardTableBase描述的起始地址开始的，而是从一个相对起始地址有一定偏移的位置开始的。这个偏移量记录在成员变量cardTableOffset中。相应地，Java堆的起始地址和Card Table的偏移地址的差值记录在全局变量gDvm指向的结构体DvmGlobals的成员变量biasedCardTableBase。按照函数dvmCardTableStartup前面的注释，之所以要这样做，是为了避免JIT在Card Table伪造假值。至于JIT会在Card Table伪造假值的原因，就不得而知，因为还没有研究JIT。在此也希望了解的同学可以告诉一下老罗:)

前面我们提到，在Card Table中，用一个byte来描述GC_CARD_SIZE个对象。GC_CARD_SIZE是一个宏，它的定义如下所示：

```c
\#define GC_CARD_SHIFT 7  
\#define GC_CARD_SIZE (1 << GC_CARD_SHIFT)  
```
这两个宏定义在文件dalvik/vm/alloc/CardTable.h中。

也就是说，在Card Table中，用一个byte来描述128个对象。每当一个对象在Concurrent GC的过程中被修改时，典型的情景就是我们通过函数dvmSetFieldObje修改了该对象的引用类型的成员变量。在这种情况下，该对象在Card Table中对应的字节会被设置为GC_CARD_DIRTY。相反，如果一个对象在Concurrent GC的过程中没有被修改，那么它在Card Table中对应的字节会保持为GC_CARD_CLEAN。

GC_CARD_DIRTY和GC_CARD_CLEAN是两个宏，它们的定义在如下所示：

```c
#define GC_CARD_CLEAN 0  
#define GC_CARD_DIRTY 0x70  
```
这两个宏定义在文件dalvik/vm/alloc/CardTable.h中。

接下来我们再通过四个函数dvmIsValidCard、dvmCardFromAddr、dvmAddrFromCard和dvmMarkCard来进一步理解Card Table和对象地址的关系。

函数dvmIsValidCard的实现如下所示：

```c
/* 
 * Returns true iff the address is within the bounds of the card table. 
 */  
bool dvmIsValidCard(const u1 *cardAddr)  
{  
    GcHeap *h = gDvm.gcHeap;  
    u1* begin = h->cardTableBase + h->cardTableOffset;  
    u1* end = &begin[h->cardTableLength];  
    return cardAddr >= begin && cardAddr < end;  
}  
```
这个函数定义在文件dalvik/vm/alloc/CardTable.cpp中。

参数cardAddr描述的是一个Card Table内部的地址。由于上述的偏移地址的存在，并不是所有的Card Table内部地址都是正确的Card Table地址。只有大于等于偏移地址并且小于当前使用的地址的地址才是正确的地址。

Card Table的起始地址记录在GcHeap结构体的成员变量cardTableBase中，而偏移量记录在另外一个成员变量cardTableOffset中，因此将这两个值相加即可得到Card Table的偏移地址。另外，当前Card Table使用的大小记录在GcHeap结构体的成员变量cardTableLength中，因此，通过这些信息我们就可以判断参数cardAddr描述的是否是一个正确的Card Table地址。

函数dvmCardFromAddr的实现如下所示：

```c
/* 
 * Returns the address of the relevant byte in the card table, given 
 * an address on the heap. 
 */  
u1 *dvmCardFromAddr(const void *addr)  
{  
    u1 *biasedBase = gDvm.biasedCardTableBase;  
    u1 *cardAddr = biasedBase + ((uintptr_t)addr >> GC_CARD_SHIFT);  
    assert(dvmIsValidCard(cardAddr));  
    return cardAddr;  
}  
```
这个函数定义在文件dalvik/vm/alloc/CardTable.cpp中。

参数addr描述的是一个对象地址，函数dvmCardFromAddr返回它在Card Table中对应的字节的地址。全局变量gDvm指向的结构体DvmGlobals的成员变量biasedCardTableBase记录的是Java堆的起始地址与Card Table的偏移地址的差值。将参数addr的值左移GC_CARD_SHIFT位，相当于是得到对象addr在Card Table的字节索引值。将这个索引值加上Java堆的起始地址与Card Table的偏移地址的差值，即可得到对象addr在Card Table中对应的字节的地址。

函数dvmAddrFromCard的实现如下所示：

```c
/* 
 * Returns the first address in the heap which maps to this card. 
 */  
void *dvmAddrFromCard(const u1 *cardAddr)  
{  
    assert(dvmIsValidCard(cardAddr));  
    uintptr_t offset = cardAddr - gDvm.biasedCardTableBase;  
    return (void *)(offset << GC_CARD_SHIFT);  
}  
```
这个函数定义在文件dalvik/vm/alloc/CardTable.cpp中。

参数cardAddr描述的是一个Card Table地址，函数dvmAddrFromCard返回它对应的对象的地址，它所执行的操作刚刚是和上面分析的函数dvmCardFromAddr相反。在此这里不再多述，同学会自己体会一下。

函数dvmMarkCard的实现如下所示：

```c
/* 
 * Dirties the card for the given address. 
 */  
void dvmMarkCard(const void *addr)  
{  
    u1 *cardAddr = dvmCardFromAddr(addr);  
    *cardAddr = GC_CARD_DIRTY;  
}  
```
这个函数定义在文件dalvik/vm/alloc/CardTable.cpp中。

在Concurrent GC执行的过程中，如果修改了一个对象的类型为引用的成员变量，那么就需要调用函数dvmMarkCard来将该对象在Card Table中对应的字节设置为GC_CARD_DIRTY，以便后面可以对这个对象进行特殊的处理。这个特殊的处理我们后面分析Dalvik虚拟机的垃圾收集过程时再分析。

函数dvmMarkCard的实现很简单，它首先是通过函数dvmCardFromAddr找到对象在Card Table中对应的字节的地址，然后再将访字节的值设置为GC_CARD_DIRTY。

有了这些基础知识之后，回到函数dvmCardTableStartup中，我们需要知道要创建的Card Table的大小以及该Card Table使用的偏移量。正常来说，我们需要的Card Table的大小为(heapMaximumSize / GC_CARD_SIZE)，其中，heapMaximumSize为Java堆的大小。但是前面分析的偏移量的存在，我们需要额外的一些内存。额外的内存大小为0x100，即256个字节。因此，我们最终需要的Card Table的大小length就为：
```
(heapMaximumSize / GC_CARD_SIZE) + 0x100
```
我们通过调用函数dvmAllocRegion来创建Card Table，得到其起始地址为allocBase。接下来就可以计算Card Table使用的偏移地址。

首先是计算一个偏移地址biasedBase：
```
(u1 \*)((uintptr_t)allocBase - ((uintptr_t)heapBase >> GC_CARD_SHIFT))
```
其中，heapBase是Java堆的起始地址。

用GC_CARD_DIRTY的值减去biasedBase地址的低8位，就可以得到一个初始偏移量offset：
```
GC_CARD_DIRTY - ((uintptr_t)biasedBase & 0xff)
```
GC_CARD_DIRTY的值定义为0x70，biasedBase地址的低8位描述的值界于0和0xff之间，因此，上面计算得到的offset可能为负数。在这种情况下，需要将它的值加上256，这是因为我们需要保证Card Table使用的偏移量是正数。最终得到的偏移量如下所示：
```
offset + (offset < 0 ? 0x100 : 0)
```
这里之所以是加上256，是因为我们在创建Card Table的时候，额外地增加了256个字节，因此这里不仅可以保证偏移量是正数，还可以保证最终使用的Card Table不会超出前面通过调用函数dvmAllocRegion创建的内存块范围。     

上述计算得到的偏移量保存在gcHeap->cardTableOffset中。相应地，Java堆的起始地址和Card Table使用的偏移地址的差值需要调整为：
```
biasedBase + gcHeap->cardTableOffset
```
得到的结果保存在gDvm.biasedCardTableBase中。

这里之所以要采取这么奇怪的算法来给Card Table设置一个偏移量，就是为了前面说的，避免JIT在Card Table伪造假值。

至此，我们就分析完成Dalvik虚拟机在启动的过程中创建Java堆及其相关的Mark Heap Bitmap、Live Heap Bitmap、Mark Stack和Card Table数据结构了。有了这些基础知识，接下来我们就可以继续分析Java对象的分配过程和垃圾收集过程了，敬请关注！更多的信息也可以关注老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)。