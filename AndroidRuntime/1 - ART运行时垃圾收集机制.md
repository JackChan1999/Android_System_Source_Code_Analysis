为了学习ART运行时的垃圾收集机制，我们先把Dalvik虚拟机的垃圾收集机制研究了一遍。这是因为两者都使用到了Mark-Sweep[算法](http://lib.csdn.net/base/datastructure)，因此它们在概念上有很多一致的地方。然而在实现上，Dalvik虚拟机的垃圾收集机制要简单一些。这样我们就可以先从简单的Dalvik虚拟机垃圾收集机制入手，然后再逐步深入地学习复杂的ART运行时垃圾收集机制。本文就对ART运行时垃圾收集机制进行简要介绍和制定学习计划。

老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)，欢迎关注！

《[Android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

与[Dalvik虚拟机垃圾收集机制简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/41338251)一文介绍的Dalvik虚拟机垃圾收集机制一样，ART运行时垃圾收集机制也涉及到类似于Zygote堆、Active堆、Card Table、Heap Bitmap和Mark Stack等概念，如图1所示：

![img](http://img.blog.csdn.net/20141224005343406?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图1 ART运行时垃圾收集机制的基本概念

从图1可以看到，ART运行时堆划分为四个空间，分别是Image Space、Zygote Space、Allocation Space和Large Object Space。其中，Image Space、Zygote Space、Allocation Space是在地址上连续的空间，称为Continuous Space，而Large Object Space是一些离散地址的集合，用来分配一些大对象，称为Discontinuous Space。

在Image Space和Zygote Space之间，隔着一段用来映射system@framework@boot.art@classes.oat文件的内存。从前面[Android运行时ART加载OAT文件的过程分析](http://blog.csdn.net/luoshengyang/article/details/39307813)一文可以知道，system@framework@boot.art@classes.oat是一个OAT文件，它是由在系统启动类路径中的所有DEX文件翻译得到的，而Image Space空间就包含了那些需要预加载的系统类对象。这意味着需要预加载的类对象是在生成system@framework@boot.art@classes.oat这个OAT文件的时候创建并且保存在文件system@framework@boot.art@classes.dex中，以后只要系统启动类路径中的DEX文件不发生变化（即不发生更新升级），那么以后每次系统启动只需要将文件system@framework@boot.art@classes.dex直接映射到内存即可，省去了创建各个类对象的时间。之前使用Dalvik虚拟机作为应用程序运行时时，每次系统启动时，都需要为那些预加载的类创建类对象。因此，虽然ART运行时第一次启动时会比较慢，但是以后启动实际上会更快。

由于system@framework@boot.art@classes.dex文件保存的是一些预先创建的对象，并且这些对象之间可能会互相引用，因此我们必须保证system@framework@boot.art@classes.dex文件每次加载到内存的地址都是固定的。这个固定的地址保存在system@framework@boot.art@classes.dex文件开头的一个Image Header中。此外，system@framework@boot.art@classes.dex文件也依赖于system@framework@boot.art@classes.oat文件，因此也会将后者固定加载到Image Space的末尾。

Zygote Space和Allocation Space与Dalvik虚拟机垃圾收集机制中的Zygote堆和Active堆的作用是一样的。Zygote Space在Zygote进程和应用程序进程之间共享的，而Allocation Space则是每个进程独占的。同样的，Zygote进程一开始只有一个Image Space和一个Zygote Space。在Zygote进程fork第一个子进程之前，就会把Zygote Space一分为二，原来的已经被使用的那部分堆还叫Zygote Space，而未使用的那部分堆就叫Allocation Space。以后的对象都在Allocation Space上分配。

通过上述这种方式，就可以使得Image Space和Zygote Space在Zygote进程和应用程序进程之间进行共享，而Allocation Space就每个进程都独立地拥有一份。注意，虽然Image Space和Zygote Space都是在Zygote进程和应用程序进程之间进行共享，但是前者的对象只创建一次，而后者的对象需要在系统每次启动时根据运行情况都重新创建一遍。

为了验证上面的分析，我们在某一次系统启动时，查看Settings.apk进程的地址空间，如图2所示：

![img](http://img.blog.csdn.net/20141224015019314)

图2 Settings.apk的地址空间

其中，地址空间0x60000000-0x60a94000对应的就是Image Space，它映射的是system@framework@boot.art@classes.dex文件，固定映射在地址0x60000000上。紧跟着的地址空间0x60a94000-0x64591000映射的是system@framework@boot.art@classes.oat文件。再接下来就是Zygote Space和Allocation Space，分别映射在地址空间0x64591000-0x646e7000和0x646e7000-0x6754e000上。并且都是匿名共享内存块。另外，通过翻译Settings.apk里面的classes.dex文件得到的OAT文件system@priv-app@Settings.apk@classes.dex映射在地址空间0xaf29b000-0xaf654000上。

与Dalivk虚拟机类似，ART运行时也使用一个Heap对象来描述堆，它的实现如图3所示：

![img](http://img.blog.csdn.net/20141229201435625?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图3 ART运行时堆描述类Heap的实现

Heap类包含了以下重要成员变量描述ART运行时的堆，它们的作用如下所述：

1. mark_sweep_collectors_: 一个std::vector&lt;collector::MarkSweep*>向量，保存了六种Mark-Sweep垃圾收集器。

2. continuous_spaces_: 一个std::vector&lt;space::ContinuousSpace*>向量，保存了图1所示的三个在地址空间上连续的Image Space、Zygote Space和Allocation Space。

3. concurrent_gc_: 一个bool变量，描述是否支持并行GC，可以通过ART运行时启动选项-Xgc来指定。

4. parallel_gc_threads_: 一个size_t变量，指定在GC暂停阶段用来同时执行GC任务的线程数，可以通过ART运行时启动选项-XX:ParallelGCThreads指定。如果没有指定，它的值就等于CPU核心数减1。这里之所以要减1是因为parallel_gc_threads_描述的实际上是除了当前GC线程之外的其它也用于GC任务的线程的个数。

5. conc_gc_threads_: 一个size_t变量，指定非GC暂停阶段用来同时执行GC任务的线程数，可以通过ART运行时启动选项-XX:ConcGCThreads来指定。

6. discontinuous_spaces_: 一个std::vector&lt;space::DiscontinuousSpace*>向量，保存了图1所示的在地址空间上不连续的Large Object Space。

7. alloc_space_: 一个space::DlMallocSpace指针，指向一个space::DlMallocSpace对象，该对象描述的是图1所示的Allocation Space。

8. large_object_space_: 一个space::LargeObjectSpace指针，指向一个space::LargeObjectSpace对象，该对象描述的是图1所示的Large Object Space。

9. card_table_: 一个UniquePtr&lt;accounting::CardTable>指针，指向一个accounting::CardTable对象，该对象描述的是图1所示的Card Table。

10. image_mod_union_table_: 一个UniquePtr&lt;accounting::ModUnionTable>指针，指向一个accounting::ModUnionTable对象，该对象描述的是图1所示的位于上方的Mod Union Table对象，用来记录在GC并行阶段在Image Space上分配的对象对在Zygote Space和Allocation Space上分配的对象的引用。

11. zygote_mod_union_table_: 一个UniquePtr&lt;accounting::ModUnionTable>指针，指向一个accounting::ModUnionTable对象，该对象描述的是图1所示的位于下方的Mod Union Table，用来记录在GC并行阶段在Zygote Space上分配的对象对在Allocation Space上分配的对象的引用。

12. mark_stack_: 一个UniquePtr&lt;accounting::ObjectStack>指针，指向一个accounting::ObjectStack对象，该对象描述的是图1所示的Mark Stack，用来在GC过程中实现递归对象标记。

13. allocation_stack_: 一个UniquePtr&lt;accounting::ObjectStack>指针，指向一个accounting::ObjectStack对象，该对象描述的是图1所示的Allocation Stack，用来记录上一次GC后分配的对象，用来实现类型为Sticky的Mark Sweep Collector。

14. live_stack_: 一个UniquePtr&lt;accounting::ObjectStack>指针，指向一个accounting::ObjectStack对象，该对象描述的是图1所示的Live Stack，配合allocation_stack_一起使用，用来实现类型为Sticky的Mark Sweep Collector。

15. mark_bitmap_: 一个UniquePtr&lt;accounting::HeapBitmap>指针，指向一个accounting::HeapBitmap对象，该对象描述的是图1所示的Mark Bitmap，与Dalvik虚拟机的Mark Bitmap作用是一样的，用来标记当前GC之后还存活的对象。

16. live_bitmap_: 一个UniquePtr&lt;accounting::HeapBitmap>指针，指向一个accounting::HeapBitmap对象，该对象描述的是图1所示的Live Bitmap，与Dalvik虚拟机的Live Bitmap作用是一样的，用来标记上一次GC之后还存活的对象。

除了上述的16个成员变量，Heap类还定义了以下三个垃圾收集接口：

1. CollectGarbage: 用来执行显式GC，例如用实现System.gc接口。

2. ConcurrentGC: 用来执行并行GC，只能被ART运行时内部的GC守护线程调用。

3. CollectGarbageInternal: ART运行时内部调用的GC接口，可以执行各种类型的GC。

上面我们提到，Heap类的成员变量mark_sweep_collectors_保存了ART运行时内部使用的六种垃圾收集器，这六种垃圾收集器分为两组。其中一组是支持并行GC的，另一组是不支持并行GC的。每一组都由MarkSweep、PartialMarkSweep和StickyMarkSweep三种类型的垃圾收集器组成。它们的类关系如图4所示：

![img](http://img.blog.csdn.net/20141229210014880?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图4 ART运行时的垃圾收集器

StickyMarkSweep继承于PartialMarkSweep，PartialMarkSweep又继承于MarkSweep、而MarkSweep又继承于GarbageCollector。因此，我们可以推断出，GarbageCollector定义了垃圾收集器接口，而MarkSweep、PartialMarkSweep和StickyMarkSweep通过重定某些接口来实现不同类型的垃圾收集器。

在GarbageCollector中，有一个成员变量heap_，指向的是ART运行时堆，同时定义了两个public成员函数IsConcurrent和GetGcType。前者用来获得一个垃圾收集器是否支持并行GC，后者用来获得垃圾收集器类型 。

垃圾收集器类型通过枚举类型GcType来描述，它的定义如下所示：

```c
 // The type of collection to be performed. The ordering of the enum matters, it is used to  
 // determine which GCs are run first.  
 enum GcType {  
   // Placeholder for when no GC has been performed.  
   kGcTypeNone,  
   // Sticky mark bits GC that attempts to only free objects allocated since the last GC.  
   kGcTypeSticky,  
   // Partial GC that marks the application heap but not the Zygote.  
   kGcTypePartial,  
   // Full GC that marks and frees in both the application and Zygote heap.  
   kGcTypeFull,  
   // Number of different GC types.  
   kGcTypeMax,  
 };  
```
这个枚举类型定义在文件t/runtime/gc/collector/gc_type.h中。 

其中，kGcTypeNone是一个占位符，kGcTypeMax描述的是垃圾收集器类型个数。kGcTypeSticky指的就是StickyMarkSweep类型的垃圾收集器，用来收集上次GC以来分配的对象。kGcTypePartial指的就是PartialMarkSweep类型的垃圾收集器，用来收集在Allocation Space上分配的对象。kGcTypeFull指的就是MarkSweep类型的垃圾收集器，用来收集在Zygote Space和Allocation Space上分配的对象。

此外，GarbageCollector通过定义以下五个虚函数描述GC的各个阶段：

1. InitializePhase: 用来实现GC的初始化阶段，用来初始化垃圾收集器内部的状态。

2. MarkingPhase: 用来实现GC的标记阶段，该阶段有可能是并行的，也有可能不是并行。

3. HandleDirtyObjectsPhase: 用来实现并行GC的Dirty Object标记，也就是递归标记那些在并行标记对象阶段中被修改的对象。

4. ReclaimPhase: 用来实现GC的回收阶段。

5. FinishPhase: 用来实现GC的结束阶段。

MarkSweep类通过重写上述五个虚函数实现自己的垃圾收集过程，同时，它又通过定义以下三个虚函数来让子类PartialMarkSweep和StickyMarkSweep实现特定的垃圾收集器：

1. MarkReachableObjects: 用来递归标记从根集对象引用的其它对象。

2. BindBitmap: 用来指定垃圾收集范围。

3. Sweep: 用来回收垃圾对象。 

其中，MarkSweep类通过自己实现的成员函数BindBitmap将垃圾收集范围指定为Zygote和Allocation空间，而PartialMarkSweep和StickyMarkSweep类通过重写成员函数BindBitmap将垃圾收集范围指定为Allocation空间和上次GC后所分配的对象。此外，StickyMarkSweep类还通过重定成员函数MarkReachableObjects和Sweep将对象标记和回收限制为上次GC后所分配的对象。

上面我们也提到了ART运行时将堆划分为连续空间和不连续空间两种类型，每一种类型又分别有不同的实现。这些实现的类关系如图5所示：

![img](http://img.blog.csdn.net/20141229212928765?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图5. ART运行时各种堆空间的关系图

首先看最下面的三个类ImageSpace、DlMallocSpace和LargeObjectMapSpace。其中，ImageSpace描述的是Image Space，DlMallocSpace描述的是Zygote Space和Allocation Space，LargeObjectMapSpace描述的是Large Object Space。它们都有一个共同的基类Space。

Space类有一个类型为GcRetentionPolicy的成员变量gc_retention_policy_，用来描述一个Space的回收策略。GcRetentionPolicy是一个枚举类型，它的定义如下所示：

```c
// See Space::GetGcRetentionPolicy.  
enum GcRetentionPolicy {  
  // Objects are retained forever with this policy for a space.  
  kGcRetentionPolicyNeverCollect,  
  // Every GC cycle will attempt to collect objects in this space.  
  kGcRetentionPolicyAlwaysCollect,  
  // Objects will be considered for collection only in "full" GC cycles, ie faster partial  
  // collections won't scan these areas such as the Zygote.  
  kGcRetentionPolicyFullCollect,  
};  
```
这个枚举类型定义在文件art/runtime/gc/space/space.h中。

kGcRetentionPolicyNeverCollect表示永远不会进行垃圾回收的Space，例如Image Space。kGcRetentionPolicyAlwaysCollect表示每一次GC都会尝试回收垃圾的Space，例如Allocation Space。kGcRetentionPolicyFullCollect表示只有执行类型为kGcTypeFull的GC才会进行垃圾回收的Space，例如Zygote Space。通过Space类的成员函数GetGcRetentionPolicy可以获得一个Space的回收策略。

一个Space除了具有回收策略之外，还具有Space Type，通过枚举类型SpaceType来描述。枚举类型SpaceType的定义如下所示：

```c
enum SpaceType {  
  kSpaceTypeImageSpace,  
  kSpaceTypeAllocSpace,  
  kSpaceTypeZygoteSpace,  
  kSpaceTypeLargeObjectSpace,  
};  
```
这个枚举类型定义在文件art/runtime/gc/space/space.h中。

kSpaceTypeImageSpace、kSpaceTypeAllocSpace、kSpaceTypeZygoteSpace和kSpaceTypeLargeObjectSpace描述的就分别是Image Space、Allocation Space、Zygote Space和Large Object Space。通过Space类的成员函数GetType可以获得一个Space的Type。此外，Space类还提供了IsImageSpace、IsDlMallocSpace、IsZygoteSpace和IsLargeObjectSpace四个辅助成员函数来方便判断一个Space的类型。

前面我们提到，Space划分Continuous和Discontinuous两种，其实还可以划分为Allocable和Non-Allocable两种。例如，Image Space是不能分配新对象的，而Zygote Space、Allocation Space和Large Object Space是可以分配对象。对于可分配新对象的Space，ART运行时定义了一个基类AllocSpace，它的定义以下几个重要的接口来描述一个可分配新对象的Space，如下所示：

1. GetBytesAllocated: 获得当前已经分配的字节数。

2. GetObjectsAllocated: 获得当前已经分配的对象数。

3. GetTotalBytesAllocated: 获得Space自创建以来所分配的字节数。

4. GetTotalObjectsAllocated: 获得Space自创建以来所分配的对象数。

5. Alloc: 在Space上分配一个指定大小的对象。

6. AllocationSize: 获得一个对象占据的内存块大小。

7. Free: 释放一个对象占据的内存块。

8. FreeList: 批量释放一系列对象占据的内存块。

Continuous Space和Discontinuous Space分别使用类ContinuousSpace和DiscontinuousSpace来描述，它们都是从类Space继承下来的。

ContinuousSpace类有两个成员变量begin_和end_，用来描述一个Continuous Space内部所使用的内存块的开始和结束地址。此外，ContinuousSpace类还定义了以下几个重要接口来获得一个Continuous Space的相关信息，如下所示：

1. Begin: 获得Space的起始地址。

2. End: 获得Space的结束地址。

3. Size: 获得Space的大小。

4. GetLiveBitmap: 获得Space的Live Bitmap。

5. GetMarkBitmap: 获得Space的Mark Bitmap。

由于Discontinuous Space在地址空间上是不连续的，因此它不像Continuous Space一样，可以使用类似begin_和end_的成员变量来确定Space内部使用的内存块。DiscontinuousSpace类在内部使用两个SpaceSetMap容器live_objects_和mark_objects_来描述已经分配对象集合和在GC过程中被标记的对象集合。此外，DiscontinuousSpace类还定义了以下两个接口来获得这两个对象集合，如下所示：

1. GetLiveObjects: 获得当前分配的对象集合。

2. GetMarkObjects: 获得在当前GC过程被标记的对象集合。

Continuous Space内部使用的内存块都是通过内存映射得到的，不过这块内存有可能是通过不同方式映射得到的。例如，Image Space内部使用的内存块是通过内存映射Image文件得到的，而Zygote Space和Allocation Space内部使用的内存块是通过内存映射匿名共享内存得到。通过内存映射得到的Space使用类MemMapSpace来描述，它继承于ContinousSpace。MemMapSpace类有一个成员变量mem_map_，类型是UniquePtr&lt;MemMap>，指向一块由一个MemMap对象描述的内存块，可以通过成员函数Capacity来获得该内存块的大小。

从图5可以看到，Image Space是通过ImageSpace类来描述，它继承于MemMapSpace类。前面我们提到，Image Space是从一个Image文件获得的，就是图1所示的boot.art@classes.dex文件，从[Android运行时ART加载OAT文件的过程分析一文](http://blog.csdn.net/luoshengyang/article/details/39307813)又可以知道，Image文件关联有一个OAT文件，就是图1所示的boot.art@classes.oat文件。因此，在ImageSpace类内部，有一个成员变量oat_file_，用来描述与Image Space关联的OAT文件。此外。在ImageSpace类内部，还有一个类型为UniquePtr&lt;accounting::SpaceBitmap>的成员变量live_bitmap_，用来标记Imape Space的存活对象。由于Image Space是不会进行新对象分配和垃圾回收的，因此它不像其它Space一样，还有另外一个Mark Bitmap。不过Space要求其子类要有一个Live Bitmap和一个Mark Bitmap，于是，ImageSpace就将内部的live_bitmap_同时作为Live Bitmap和Mark Bitmap来使用。

ImageSpace还通过定义以下几个接口来获得Image Space的相关信息，如下所示：

1. GetType: 重写了父类Space的成员函数GetType，返回固定为kSpaceTypeImageSpace，表示这是一个Image Space。

2. GetLiveBitmap: 获得Image Space的Live Bitmap。

3. GetMarkBitmap: 获得Image Space的Mark Bitmap。

4. Create: 一个静态成员函数，根据指定的Image文件创建一个Image Space。

Zygote Space和Allocation Space都是通过类DlMallocSpace来描述。虽然Zygote Space和Allocation Space内部使用的内存块都是通过内存映射得到的，不过在使用它们的时候，是通过C库提供的内存管理接口来使用的，因此这里就将它们的名字命名为DlMalloc，这也是DlMallocSpace的由来。由于DlMallocSpace还是可以分配新对象的，因此在图5中，我们看到它除了继承于MemMapSpace类之外，还继承于AllocSpace。

DlMallocSpace定义了以下几个重要成员变量来描述内部状态，如下所示：

1. mspace_: 和Dalvik虚拟机一样，ART运行时也是将Zygote Space和Allocation Space使用的匿名共享内存块封装成一个mspace对象，以便可以使用C库提供的内存管理接口来在上面分配和释放内存。

2. live_bitmap_: 指向一个SpaceBitmap，用来记录上次GC后存活对象。

3. mark_bitmap_: 指向一个SpaceBitmap，用来记录当前GC被标记的对象。

4. num_bytes_allocated_: 记录当前已经分配的内存字节数。

5. num_objects_allocated_: 记录当前已经分配的对象数。

6. total_bytes_allocated_: 记录从Space创建以来所分配的内存字节数。

7. total_objects_allocated_: 记录从Space创建以来所分配的对象数。

DlMallocSpace还新定义或者重写了父类的成员函数，如下所示：

1. GetType: 重写了父类Space的成员函数GetType，返回固定为SpaceTypeZygoteSpace或者kSpaceTypeAllocSpace，表示这是一个Zygote Space或者Allocation Space。

2. Create: 一个静态成员函数，用来创建一个Zygote Space或者Allocation Space。

3. GetBytesAllocated: 重写了父类AllocSpace的成员函数GetBytesAllocated。

4. GetObjectsAllocated: 重写了父类AllocSpace的成员函数GetObjectsAllocated。

5. GetTotalBytesAllocated: 重写了父类AllocSpace的成员函数GetTotalBytesAllocated。

6. GetTotalObjectsAllocated: 重写了父类AllocSpace的成员函数GetTotalObjectsAllocated。

7. Alloc: 重写了父类AllocSpace的成员函数Alloc。

8. AllocationSize: 重写了父类AllocSpace的成员函数AllocationSize。

9. Free: 重写了父类AllocSpace的成员函数Free。

10. FreeList: 重写了父类AllocSpace的成员函数FreeList。

11. Capacity: 重写了父类MemMapSpace的成员函数Capacity。

现在再来看最后一种Space -- Large Object Space。ART运行时提供了两种Large Object Space实现。其中一种实现和Continuous Space的实现类似，预先分配好一块大的内存空间，然后再在上面为对象分配内存块。不过这种方式实现的Large Object Space不像Continuous Space通过C库的内块管理接口来分配和释放内存，而是自己维护一个Free List。每次为对象分配内存时，都是从这个Free List找到合适的空闲的内存块来分配。释放内存的时候，也是将要释放的内存添加到该Free List去。

另外一种Large Object Space实现是每次为对象分配内存时，都单独为其映射一新的内存。也就是说，为每一个对象分配的内存块都是相互独立的。这种实现方式相比上面介绍的Free List实现方式，也更简单一些。在[android](http://lib.csdn.net/base/android) 4.4中，ART运行时使用的是后一种实现方式，因此我们这里也主要是介绍这种实现。

为每一对象映射一块独立的内存块的Large Object Space实现称为LargeObjectMapSpace，它与Free List方式的实现都是继承于类LargeObjectSpace。LargeObjectSpace又分别继承了DiscontinuousSpace和AllocSpace。因此，我们就可以知道，LargeObjectMapSpace描述的是一个在地址空间上不连续的Large Object Space。

LargeObjectSpace定义了以下重要成员变量描述内部状态，如下所示：

1. num_bytes_allocated_: 记录当前已经分配的内存字节数。

2. num_objects_allocated_: 记录当前已经分配的对象数。

3. total_bytes_allocated_: 记录从Space创建以来所分配的内存字节数。

4. total_objects_allocated_: 记录从Space创建以来所分配的对象数。

LargeObjectSpace还重写了父类AllocSpace的以下成员函数：

1. GetType: 重写了父类Space的成员函数GetType，返回固定为kSpaceTypeLargeObjectSpace，表示这是一个Large Object Space。

2. GetBytesAllocated: 重写了父类AllocSpace的成员函数GetBytesAllocated。

3. GetObjectsAllocated: 重写了父类AllocSpace的成员函数GetObjectsAllocated。

4. GetTotalBytesAllocated: 重写了父类AllocSpace的成员函数GetTotalBytesAllocated。

5. GetTotalObjectsAllocated: 重写了父类AllocSpace的成员函数GetTotalObjectsAllocated。

6. FreeList: 重写了父类AllocSpace的成员函数FreeList。

LargeObjectMapSpace继承于LargeObjectSpace，它内部有一个成员变量large_objects_，指向一个std::vector&lt;mirror::Object*, accounting::GCAllocator&lt;mirror::Object*> >向量，里面保存的就是为每一个对象独立映射的内存块。

此外，LargeObjectMapSpace还新定义或者重写了父类AllocSpace的以下成员变量：

1. Create: 一个静态成员函数，用来创建一个LargeObjectMapSpace。

2. Alloc: 重写了父类AllocSpace的成员函数Alloc。

3. AllocationSize: 重写了父类AllocSpace的成员函数AllocationSize。

4. Free: 重写了父类AllocSpace的成员函数Free。

除了Garbage Collector和Space，ART运行时垃圾收集机制比Dalvik垃圾收集机制还多了一个称Mod Union Table的概念。Mod Union Table是与Card Table配合使用的，用来记录在一次GC过程中，记录不会被回收的Space的对象对会被回收的Space的引用。例如，Image Space的对象对Zygote Space和Allocation Space的对象的引用，以及Zygote Space的对象对Allocation Space的对象的引用。

ART运行时定义了一个ModUnionTable基类以及一系列的子类，用来记录不同Space的对象对特定Space的对象的引用，它们的关系如图6所示：

![img](http://img.blog.csdn.net/20141230170357343?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图6 ART运行时的Mod Union Table实现

ModUnionTable类有一个成员变量heap_，指向ART运行时堆，可以通过成员函数GetHeap来获得。

在GC标记阶段，Card Table和Mod Union Table分三步处理。第一步是调用ModUnionTable类的成员函数ClearCards清理Card Table里面的Dirty Card，并且将这些Dirty Card记录在Mod Union Table中。第二步是调用ModUnionTable类的成员函数Update将遍历记录在Mod Union Table里面的Drity Card，并且找到对应的被修改对象，然后将被修改对象引用的其它对象记录起来。第三步是调用ModUnionTable类的成员函数MarkReferences标记前面第二步那些被被修改对象引用的其它对象。通过这种方式，就可以使用Card Table可以在标记阶段重复使用，即在执行第二步之前，重复执行第一步，最后通过Mod Union Table将所有被被修改对象引用的其它对象收集起来统一进行标记，避免对相同对象进行重复标记。

上面说到，Mod Union Table的作用就使得Card Table可以重复使用，前提是Mod Union Table将每一次清理Card Table之前，将之前已经是Dirty的Card缓存起来。因此，ModUnionTableCardCache类继承ModUnionTable类的时候，定义了一个类型为CardSet的成员变量cleared_cards_，就是用来缓存Dirty Card的，并且它重写了父类ModUnionTable的成员函数ClearCards、Update和MarkReferences来实现自己的DIrty Card缓存策略。

另外一个继承于ModUnionTable的子类ModUnionTableReferenceCache，不单会将Dirty Card缓存起来保存在类型同样为CardSet的成员变量cleared_cards_中，而且还将与这些Dirty Card对应的被引用对象也缓存起来保存在成员变量references_指向的一个SafeMap中。同样的，ModUnionTableReferenceCache重写了父类ModUnionTable的成员函数ClearCards、Update和MarkReferences来实现自己的Dirty Card缓存策略。此外，ModUnionTableReferenceCache还提供了一个AddReference接口，用来决定哪些被引用对象需要缓存。

ModUnionTableReferenceCache类并不是直接使用的，它也是作为基类使用。ART运行时提供了两个子类ModUnionTableToZygoteAllocspace和ModUnionTableToAllocspce，它们都继承于ModUnionTableReferenceCache类。从名字我们可以大致推断出，ModUnionTableToZygoteAllocspace类是用来记录位于Zygote Space和Allocation Space的被引用对象的，而ModUnionTableToAllocspace是用来记录位于Allocation Space的被引用对象的，也就是对应于前面图1所示的两个Mod Union Table。不过ART运行时实际上是使用ModUnionTableCardCache来记录位于Allocation Space的被引用对象的。在后面的文章中，我们就可以看到这一点。

至此，ART运行时垃圾收集机制涉及到的基础概念我们就介绍完毕。要做到完全理解这些概念，除了要熟悉[Dalvik虚拟机垃圾收集机制简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/41338251)这一系列文章分析Dalvik虚拟机垃圾收集机制之外，还需要继续从以下情景来进一步分析ART运行时垃圾机制：

1. [ART运行时堆的创建过程](http://blog.csdn.net/luoshengyang/article/details/42379729)。

2. [ART运行时的对象分配过程](http://blog.csdn.net/luoshengyang/article/details/42492621)。

3. [ART运行时的垃圾收集过程](http://blog.csdn.net/luoshengyang/article/details/42555483)。

按照这三个情景学习ART运行时的垃圾收集机制之后，我们就会对上面涉及的概念有一个清晰的认识了。敬请关注！想了解更多信息，也可以关注老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)。