在前面一文中，我们分析了Dalvik虚拟机创建[Java](http://lib.csdn.net/base/java)堆的过程。有了Java堆之后，Dalvik虚拟机就可以在上面为对象分配内存了。在Java堆为对象分配内存需要解决内存碎片和内存不足两个问题。要解决内存碎片问题，就要找到一块大小最合适的空闲内存分配给对象使用。而内存不足有可能是内存配额用完引起的，也有可能是垃圾没有及时回收引起的，要区别对待。本文就详细分析Dalvik虚拟机是如何解决这些问题的。

老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)，欢迎关注！

《[Android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

内存碎片问题其实是一个通用的问题，不单止Dalvik虚拟机在Java堆为对象分配内存时会遇到，C库的malloc函数在分配内存时也会遇到。[android](http://lib.csdn.net/base/android)系统使用的C库bionic使用了[Doug Lea](http://gee.cs.oswego.edu/)写的dlmalloc内存分配器。也就是说，我们调用函数malloc的时候，使用的是dlmalloc内存分配器来分配内存。这是一个成熟的内存分配器，可以很好地解决内存碎片问题。关于dlmalloc内存分配器的设计，可以参考这篇文章：[A Memory Allocator](http://gee.cs.oswego.edu/dl/html/malloc.html)。

前面[Dalvik虚拟机垃圾收集机制简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/41338251)一文提到，Dalvik虚拟机的Java堆的底层实现是一块匿名共享内存，并且将其抽象为C库的一个mspace，如图1所示：

![img](http://img.blog.csdn.net/20141203015154422)

图1 Dalvik虚拟机Java堆

于是，Dalvik虚拟机就很机智地利用C库里面的dlmalloc内存分配器来解决内存碎片问题！

为了应对可能面临的内存不足问题，Dalvik虚拟机采用一种渐进的方法来为对象分配内存，直到尽了最大努力，如图2所示：

![img](http://img.blog.csdn.net/20141203015902705?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图2 Dalvik虚拟机为对象分配内存的过程

接下来，我们就详细分析这个过程，以便可以了解Dalvik虚拟机是如何解决内存不足问题的，以及分配出来的内存是如何管理的。

Dalvik虚拟机实现了一个dvmAllocObject函数。每当Dalvik虚拟机需要为对象分配内存时，就会调用函数dvmAllocObject。例如，当Dalvik虚拟机的解释器遇到一个new指令时，它就会调用函数dvmAllocObject，如下所示：

```c
HANDLE_OPCODE(OP_NEW_INSTANCE /*vAA, class@BBBB*/)  
    {  
  ClassObject* clazz;  
  Object* newObj;  
  
  EXPORT_PC();  
  
  vdst = INST_AA(inst);  
  ref = FETCH(1);  
  ......  
  clazz = dvmDexGetResolvedClass(methodClassDex, ref);  
  if (clazz == NULL) {  
      clazz = dvmResolveClass(curMethod->clazz, ref, false);  
      ......  
  }  
  
  ......  
  
  newObj = dvmAllocObject(clazz, ALLOC_DONT_TRACK);  
  ......  
  
  SET_REGISTER(vdst, (u4) newObj);  
    }  
    FINISH(2);  
OP_END  
```
这个代码段定义在文件dalvik/vm/mterp/out/InterpC-portable.cpp中。

关于Dalvik虚拟机的解释器的实现，可以参考[Dalvik虚拟机的运行过程分析](http://blog.csdn.net/luoshengyang/article/details/8914953)一文，上面这段代码首先是找到要创建的对象的类型clazz，接着以其作为参数调用函数dvmAllocObject为要创建的对象分配内存。另外一个参数ALLOC_DONT_TRACK是告诉Dalvik虚拟机的堆管理器，要分配的对象是一个根集对象，不需要对它进行跟踪。因为根集对象在GC时是会自动被追踪处理的。

函数dvmAllocObject的实现如下所示：

```c
Object* dvmAllocObject(ClassObject* clazz, int flags)  
{  
    Object* newObj;  
  
    assert(clazz != NULL);  
    assert(dvmIsClassInitialized(clazz) || dvmIsClassInitializing(clazz));  
  
    /* allocate on GC heap; memory is zeroed out */  
    newObj = (Object*)dvmMalloc(clazz->objectSize, flags);  
    if (newObj != NULL) {  
  DVM_OBJECT_INIT(newObj, clazz);  
  dvmTrackAllocation(clazz, clazz->objectSize);   /* notify DDMS */  
    }  
  
    return newObj;  
}  
```
这个函数定义在文件dalvik/vm/alloc/Alloc.cpp中。

函数dvmAllocObject调用函数dvmMalloc从Java堆中分配一块指定大小的内存给新创建的对象使用。如果分配成功，那么接下来就先使用宏DVM_OBJECT_INIT来初始化新创建对对象的成员变量clazz，使得新创建的对象可以与某个特定的类关联起来，接着再调用函数dvmTrackAllocation记录当前的内存分配信息，以便通知DDMS。

函数dvmMalloc返回的只是一块内存地址，这是没有类型的。但是由于每一个Java对象都是从Object类继承下来的，因此，函数dvmAllocObject可以将获得的没有类型的内存块强制转换为一个Object对象。

Object类的定义如下所示：

```c
struct Object {  
    /* ptr to class object */  
    ClassObject*    clazz;  
  
    /* 
     * A word containing either a "thin" lock or a "fat" monitor.  See 
     * the comments in Sync.c for a description of its layout. 
     */  
    u4              lock;  
};  
```
这个类定义在文件dalvik/vm/oo/Object.h中。

Object类有两个成员变量：clazz和lock。其中，成员变量clazz的类型为ClassObject，它对应于Java层的java.lang.Class类，用来描述对象所属的类。成员变量lock是一个锁，正是因为有了这个成员变量，在Java层中，每一个对象都可以当锁使用。

理解了Object类的定义之后，我们继续分析函数dvmMalloc的实现，如下所示：

```c
void* dvmMalloc(size_t size, int flags)  
{  
    void *ptr;  
  
    dvmLockHeap();  
  
    /* Try as hard as possible to allocate some memory. 
     */  
    ptr = tryMalloc(size);  
    if (ptr != NULL) {  
  /* We've got the memory. 
   */  
  if (gDvm.allocProf.enabled) {  
      Thread* self = dvmThreadSelf();  
      gDvm.allocProf.allocCount++;  
      gDvm.allocProf.allocSize += size;  
      if (self != NULL) {  
          self->allocProf.allocCount++;  
          self->allocProf.allocSize += size;  
      }  
  }  
    } else {  
  /* The allocation failed. 
   */  
  
  if (gDvm.allocProf.enabled) {  
      Thread* self = dvmThreadSelf();  
      gDvm.allocProf.failedAllocCount++;  
      gDvm.allocProf.failedAllocSize += size;  
      if (self != NULL) {  
          self->allocProf.failedAllocCount++;  
          self->allocProf.failedAllocSize += size;  
      }  
  }  
    }  
  
    dvmUnlockHeap();  
  
    if (ptr != NULL) {  
  /* 
   * If caller hasn't asked us not to track it, add it to the 
   * internal tracking list. 
   */  
  if ((flags & ALLOC_DONT_TRACK) == 0) {  
      dvmAddTrackedAlloc((Object*)ptr, NULL);  
  }  
    } else {  
  /* 
   * The allocation failed; throw an OutOfMemoryError. 
   */  
  throwOOME();  
    }  
  
    return ptr;  
}  
```
这个函数定义在文件dalvik/vm/alloc/Heap.cpp中。

在Java堆分配内存前后，要对Java堆进行加锁和解锁，避免多个线程同时对Java堆进行操作。这分别是通过函数dvmLockHeap和dvmunlockHeap来实现的。真正执行内存分配的操作是通过调用另外一个函数tryMalloc来完成的。如果分配成功，则记录当前线程成功分配的内存字节数和对象数等信息。否则的话，就记录当前线程失败分配的内存字节数和对象等信息。有了这些信息之后，我们就可以通过DDMS等工具来对应用程序的内存使用信息进行统计了。

最后，如果分配内存成功，并且参数flags的ALLOC_DONT_TRACK位设置为0，那么需要将新创建的对象增加到Dalvik虚拟机内部的一个引用表去。保存在这个内部引用表的对象在执行GC时，会添加到根集去，以便可以正确地判断对象的存活。

另一方面，如果分配内存失败，那么就是时候调用函数throwOOME抛出一个OOM异常了。

我们接下来继续分析函数tryMalloc的实现，如下所示：

```c
static void *tryMalloc(size_t size)  
{  
    void *ptr;  
    ......  
  
    ptr = dvmHeapSourceAlloc(size);  
    if (ptr != NULL) {  
  return ptr;  
    }  
  
    if (gDvm.gcHeap->gcRunning) {  
  ......  
  dvmWaitForConcurrentGcToComplete();  
    } else {  
  ......  
  gcForMalloc(false);  
    }  
  
    ptr = dvmHeapSourceAlloc(size);  
    if (ptr != NULL) {  
  return ptr;  
    }  
  
    ptr = dvmHeapSourceAllocAndGrow(size);  
    if (ptr != NULL) {  
  ......  
  return ptr;  
    }  
  
    gcForMalloc(true);  
    ptr = dvmHeapSourceAllocAndGrow(size);  
    if (ptr != NULL) {  
  return ptr;  
    }  
     
    ......  
  
    return NULL;  
}  
```
这个函数定义在文件dalvik/vm/alloc/Heap.cpp中。

函数tryMalloc的执行流程就如图2所示：

1. 调用函数dvmHeapSourceAlloc在Java堆上分配指定大小的内存。如果分配成功，那么就将分配得到的地址直接返回给调用者了。函数dvmHeapSourceAlloc在不改变Java堆当前大小的前提下进行内存分配，这是属于轻量级的内存分配动作。

2. 如果上一步内存分配失败，这时候就需要执行一次GC了。不过如果GC线程已经在运行中，即gDvm.gcHeap->gcRunning的值等于true，那么就直接调用函数dvmWaitForConcurrentGcToComplete等到GC执行完成就是了。否则的话，就需要调用函数gcForMalloc来执行一次GC了，参数false表示不要回收软引用对象引用的对象。

3. GC执行完毕后，再次调用函数dvmHeapSourceAlloc尝试轻量级的内存分配操作。如果分配成功，那么就将分配得到的地址直接返回给调用者了。

4. 如果上一步内存分配失败，这时候就得考虑先将Java堆的当前大小设置为Dalvik虚拟机启动时指定的Java堆最大值，再进行内存分配了。这是通过调用函数dvmHeapSourceAllocAndGrow来实现的。 

5. 如果调用函数dvmHeapSourceAllocAndGrow分配内存成功，则直接将分配得到的地址直接返回给调用者了。

6. 如果上一步内存分配还是失败，这时候就得出狠招了。再次调用函数gcForMalloc来执行GC。参数true表示要回收软引用对象引用的对象。

7. GC执行完毕，再次调用函数dvmHeapSourceAllocAndGrow进行内存分配。这是最后一次努力了，成功与事都到此为止。

这里涉及到的关键函数有三个，分别是dvmHeapSourceAlloc、dvmHeapSourceAllocAndGrow和gcForMalloc。后面一个我们在接下来一篇文章分析Dalvik虚拟机的垃圾收集过程时再分析。现在重点分析前面两个函数。

函数dvmHeapSourceAlloc的实现如下所示：

```c
void* dvmHeapSourceAlloc(size_t n)  
{  
    HS_BOILERPLATE();  
  
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
  if (ptr == NULL) {  
      return NULL;  
  }  
  uintptr_t zero_begin = (uintptr_t)ptr;  
  uintptr_t zero_end = (uintptr_t)ptr + n;  
  ......  
  uintptr_t begin = ALIGN_UP_TO_PAGE_SIZE(zero_begin);  
  uintptr_t end = zero_end & ~(uintptr_t)(SYSTEM_PAGE_SIZE - 1);  
  ......  
  if (begin < end) {  
      ......  
      madvise((void*)begin, end - begin, MADV_DONTNEED);  
      ......  
      memset((void*)end, 0, zero_end - end);  
      ......  
      zero_end = begin;  
  }  
  memset((void*)zero_begin, 0, zero_end - zero_begin);  
    } else {  
  ptr = mspace_calloc(heap->msp, 1, n);  
  if (ptr == NULL) {  
      return NULL;  
  }  
    }  
  
    countAllocation(heap, ptr);  
    ......  
    if (gDvm.gcHeap->gcRunning || !hs->hasGcThread) {  
  ......  
  return ptr;  
    }  
    if (heap->bytesAllocated > heap->concurrentStartBytes) {  
  ......  
  dvmSignalCond(&gHs->gcThreadCond);  
    }  
    return ptr;  
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapSource.cpp中。

从前面[Dalvik虚拟机Java堆创建过程分析](http://blog.csdn.net/luoshengyang/article/details/41581063)一文可知，gHs是一个全局变量，它指向一个HeapSource结构。在这个HeapSource结构中，有一个heaps数组，其中第一个元素描述的是Active堆，第二个元素描述的是Zygote堆。

通过宏hs2heap可以获得HeapSource结构中的Active堆，保存在本地变量heap中。宏hs2heap的实现如下所示：

```c
#define hs2heap(hs_) (&((hs_)->heaps[0]))  
```
这个宏定义在文件dalvik/vm/alloc/HeapSource.cpp中。

在前面[Dalvik虚拟机Java堆创建过程分析](http://blog.csdn.net/luoshengyang/article/details/41581063)一文中，我们解释了Java堆有起始大小、最大值、增长上限值、最小空闲值、最大空闲值和目标利用率等参数。在Dalvik虚拟机内部，还有一个称为软限制（Soft Limit）的参数。堆软限制是一个与堆目标利率相关的参数。

Java堆的Soft Limit开始的时候设置为最大允许的整数值。但是每一次GC之后，Dalvik虚拟机会根据Active堆已经分配的内存字节数、设定的堆目标利用率和Zygote堆的大小，重新计算Soft Limit，以及别外一个称为理想大小（Ideal Size）的值。如果此时只有一个堆，即只有Active堆没有Zygote堆，那么Soft Limit就等于Ideal Size。如果此时有两个堆，那么Ideal Size就等于Zygote堆的大小再加上Soft Limit值，其中Soft Limit值就是此时Active堆的大小，它是根据Active堆已经分配的内存字节数和设定的堆目标利用率计算得到的。

这个Soft Limit值到底有什么用呢？它主要是用来限制Active堆无节制地增长到最大值的，而是要根据预先设定的堆目标利用率来控制Active有节奏地增长到最大值。这样可以更有效地使用堆内存。想象一下，如果我们一开始Active堆的大小设置为最大值，那么就很有可能造成已分配的内存分布在一个很大的范围。这样随着Dalvik虚拟机不断地运行，Active堆的内存碎片就会越来越来重。相反，如果我们施加一个Soft Limit，那可以尽量地控制已分配的内存都位于较紧凑的范围内。这样就可以有效地减少碎片。

回到函数dvmHeapSourceAlloc中，参数n描述的是要分配的内存大小，而heap->bytesAllocated描述的是Active堆已经的内存大小。由于函数dvmHeapSourceAlloc是不允许增长Active堆的大小的，因此当(heap->bytesAllocated + n)的值大于Active堆的Soft Limit时，就直接返回一个NULL值表示分配内存失败。

如果要分配的内存不会超过Active堆的Soft Limit，那么就要考虑Dalivk虚拟机在启动时是否指定了低内存模式。我们可以通过-XX:LowMemoryMode选项来让Dalvik虚拟机运行低内存模式下。在低内存模式和非低内存模块中，对象内存的分配方式有所不同。

在低内存模式中，Dalvik虚拟机假设对象不会马上就使用分配到的内存，因此，它就通过系统接口madvice和MADV_DONTNEED标志告诉内核，刚刚分配出去的内存在近期内不会使用，内核可以该内存对应的物理页回收。当分配出去的内存被使用时，内核就会重新给它映射物理页，这样就可以做按需分配物理内存，适合在内存小的设备上运行。这里有三点需要注意。 

第一点是Dalvik虚拟机要求分配给对象的内存初始化为0，但是在低内存模式中，是使用函数mspace_malloc来分配内存，该函数不会将分配的内存初始化为0，因此我们需要自己去初始化这块内存。

第二点是对于被系统接口madvice标记为MADV_DONTNEED的内存，是不需要我们将它初始化为0的，一来是因为这是无用功（对应的物理而可能会被内核回收），二来是因为当这些内存在真正使用时，内核在为它们映射物理页的同时，也会同时映射的物理页初始为0。

第三点是在调用系统接口madvice时，指定的内存地址以及内存大小都必须以页大小为边界的，但是函数mspace_malloc分配出来的内存的地址只能保证对齐到8个字节，因此，我们是有可能不能将所有分配出来的内存都通过系统接口madvice标记为MADV_DONTNEED的。这时候对于不能标记为MADV_DONTNEED的内存，就需要调用memset来将它们初始化为0。

在非低内存模式中，处理的逻辑就简单很多了，直接使用函数mspace_calloc在Active堆上分配指定的内存大小即可，同时该函数还会将分配的内存初始化为0，正好是可以满足Dalvik虚拟机的要求。

注意，由于内存碎片的存在，即使是要分配的内存没有超出Active堆的Soft Limit，在调用函数mspace_malloc和函数mspace_calloc的时候，仍然有可能出现无法成功分配内存的情况。在这种情况下，都直接返回一个NULL值给调用者。

在分配成功的情况下，函数dvmHeapSourceAlloc还需要做两件事情。

第一件事情是调用函数countAllocation来计账，它的实现如下所示：

```c
static void countAllocation(Heap *heap, const void *ptr)  
{  
    assert(heap->bytesAllocated < mspace_footprint(heap->msp));  
  
    heap->bytesAllocated += mspace_usable_size(ptr) +  
      HEAP_SOURCE_CHUNK_OVERHEAD;  
    heap->objectsAllocated++;  
    HeapSource* hs = gDvm.gcHeap->heapSource;  
    dvmHeapBitmapSetObjectBit(&hs->liveBits, ptr);  
  
    assert(heap->bytesAllocated < mspace_footprint(heap->msp));  
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapSource.cpp中。

函数countAllocation要计的账有三个：

1. 记录Active堆当前已经分配的字节数。

2. 记录Active堆当前已经分配的对象数。

3. 调用函数dvmHeapBitmapSetObjectBit将新分配的对象在Live Heap Bitmap上对应的位设置为1，也就是说将新创建的对象标记为是存活的。关于Live Heap Bitmap，可以参考前面[Dalvik虚拟机Java堆创建过程分析](http://blog.csdn.net/luoshengyang/article/details/41581063)一文。

回到函数dvmHeapSourceAlloc中，它需要做的第二件事情是检查当前Active堆已经分配的字节数是否已经大于预先设定的Concurrent GC阀值heap->concurrentStartBytes。如果大于的话，那么就需要通知GC线程执行一次Concurrent GC。当然，如果当前GC线程已经在进行垃圾回收，那么就不用通知了。当gDvm.gcHeap->gcRunning的值等于true时，就表示GC线程正在进行垃圾回收。

这样，函数dvmHeapSourceAlloc的实现就分析完成了，接下来我们继续分析另外一个函数dvmHeapSourceAllocAndGrow的实现，如下所示：

```c
void* dvmHeapSourceAllocAndGrow(size_t n)  
{  
    ......  
  
    HeapSource *hs = gHs;  
    Heap* heap = hs2heap(hs);  
    void* ptr = dvmHeapSourceAlloc(n);  
    if (ptr != NULL) {  
  return ptr;  
    }  
  
    size_t oldIdealSize = hs->idealSize;  
    if (isSoftLimited(hs)) {  
  ......  
  hs->softLimit = SIZE_MAX;  
  ptr = dvmHeapSourceAlloc(n);  
  if (ptr != NULL) {  
      ......  
      snapIdealFootprint();  
      return ptr;  
  }  
    }  
  
    ptr = heapAllocAndGrow(hs, heap, n);  
    if (ptr != NULL) {  
  ......  
  snapIdealFootprint();  
    } else {  
  ......  
  setIdealFootprint(oldIdealSize);  
    }  
    return ptr;  
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapSource.cpp中。

函数dvmHeapSourceAllocAndGrow首先是在不增加Active堆的前提下，调用我们前面分析的函数dvmHeapSourceAlloc来分配大小为n的内存。如果分配成功，那么就可以直接返回了。否则的话，继续往前处理。

在继续往前处理之前，先记录一下当前Zygote堆和Active堆的大小之和oldIdealSize。这是因为后面我们可能会修改Active堆的大小。当修改了Active堆的大小，但是仍然不能成功分配大小为n的内存，那么就需要恢复之前Zygote堆和Active堆的大小。

如果Active堆设置有Soft Limit，那么函数isSoftLimited的返回值等于true。在这种情况下，先将Soft Limit去掉，再调用函数dvmHeapSourceAlloc来分配大小为n的内存。如果分配成功，那么在将分配得到的地址返回给调用者之前，需要调用函数snapIdealFootprint来修改Active堆的大小。也就是说，在去掉Active堆的Soft Limit之后，可以成功地分配到大小为n的内存，这时候就需要相应的增加Soft Limit的大小。

如果Active堆没有设置Soft Limit，或者去掉Soft Limit之后，仍然不能成功地在Active堆上分配在大小为n的内存，那么这时候就得出大招了，它会调用函数heapAllocAndGrow将Java堆的大小设置为允许的最大值，然后再在Active堆上分配大小为n的内存。

最后，如果能成功分配到大小为n的内存，那么就调用函数snapIdealFootprint来重新设置Active堆的当前大小。否则的话，就调用函数setIdealFootprint来恢复之前Active堆的大小。这是因为虽然分配失败，但是前面仍然做了修改Active堆大小的操作。

为了更好地理解函数dvmHeapSourceAllocAndGrow的实现，我们继续分析一下涉及到的函数isSoftLimited、setIdealFootprint、snapIdealFootprint和heapAllocAndGrow的实现。

函数isSoftLimited的实现如下所示：

```c
static bool isSoftLimited(const HeapSource *hs)  
{  
    /* softLimit will be either SIZE_MAX or the limit for the 
     * active mspace.  idealSize can be greater than softLimit 
     * if there is more than one heap.  If there is only one 
     * heap, a non-SIZE_MAX softLimit should always be the same 
     * as idealSize. 
     */  
    return hs->softLimit <= hs->idealSize;  
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapSource.cpp中。

根据我们前面的分析，hs->softLimit描述的是Active堆的大小，而hs->idealSize描述的是Zygote堆和Active堆的大小之和。

当只有一个堆时，即只有Active堆时，如果设置了Soft Limit，那么它的大小总是等于Active堆的大小，即这时候hs->softLimit总是等于hs->idealSize。如果没有设置Soft Limit，那么它的值会被设置为SIZE_MAX值，这会就会保证hs->softLimit大于hs->idealSize。也就是说，当只有一个堆时，函数isSoftLimited能正确的反映Active堆是否设置有Soft Limit。

当有两个堆时，即Zygote堆和Active堆同时存在，那么如果设置有Soft Limit，那么它的值就总是等于Active堆的大小。由于hs->idealSize描述的是Zygote堆和Active堆的大小之和，因此就一定可以保证hs->softLimit小于等于hs->idealSize。如果没有设置Soft Limit，即hs->softLimit的值等于SIZE_MAX，那么就一定可以保证hs->softLimit的值大于hs->idealSize的值。也就是说，当有两个堆时，函数isSoftLimited也能正确的反映Active堆是否设置有Soft Limit。

函数setIdealFootprint的实现如下所示：

```c
static void setIdealFootprint(size_t max)  
{  
    HS_BOILERPLATE();  
  
    HeapSource *hs = gHs;  
    size_t maximumSize = getMaximumSize(hs);  
    if (max > maximumSize) {  
  LOGI_HEAP("Clamp target GC heap from %zd.%03zdMB to %u.%03uMB",  
          FRACTIONAL_MB(max),  
          FRACTIONAL_MB(maximumSize));  
  max = maximumSize;  
    }  
  
    /* Convert max into a size that applies to the active heap. 
     * Old heaps will count against the ideal size. 
     */  
    size_t overhead = getSoftFootprint(false);  
    size_t activeMax;  
    if (overhead < max) {  
  activeMax = max - overhead;  
    } else {  
  activeMax = 0;  
    }  
  
    setSoftLimit(hs, activeMax);  
    hs->idealSize = max;  
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapSource.cpp中。

函数setIdealFootprint的作用是要将Zygote堆和Active堆的大小之和设置为max。在设置之前，先检查max值是否大于Java堆允许的最大值maximum。如果大于的话，那么就属于将max的值修改为maximum。

接下为是以参数false来调用函数getSoftFootprint来获得Zygote堆的大小overhead。如果max的值大于Zygote堆的大小overhead，那么从max中减去overhead，就可以得到Active堆的大小activeMax。如果max的值小于等于Zygote堆的大小overhead，那么就说明要将Active堆的大小activeMax设置为0。

最后，函数setIdealFootprint调用函数setSoftLimit设置Active堆的当前大小，并且将Zygote堆和Active堆的大小之和记录在hs->idealSize中。

这里又涉及到两个函数getSoftFootprint和setSoftLimit，我们同样对它们进行分析。

函数getSoftFootprint的实现如下所示：

```c
static size_t getSoftFootprint(bool includeActive)  
{  
    HS_BOILERPLATE();  
  
    HeapSource *hs = gHs;  
    size_t ret = oldHeapOverhead(hs, false);  
    if (includeActive) {  
  ret += hs->heaps[0].bytesAllocated;  
    }  
  
    return ret;  
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapSource.cpp中。

函数getSoftFootprint首先调用函数oldHeapOverhead获得Zygote堆的大小ret。当参数includeActive等于true时，就表示要返回的是Zygote堆的大小再加上Active堆当前已经分配的内存字节数的值。而当参数includeActive等于false时，要返回的仅仅是Zygote堆的大小。

函数oldHeapOverhead的实现如下所示：

```c
static size_t oldHeapOverhead(const HeapSource *hs, bool includeActive)  
{  
    size_t footprint = 0;  
    size_t i;  
  
    if (includeActive) {  
  i = 0;  
    } else {  
  i = 1;  
    }  
    for (/* i = i */; i < hs->numHeaps; i++) {  
//TODO: include size of bitmaps?  If so, don't use bitsLen, listen to .max  
  footprint += mspace_footprint(hs->heaps[i].msp);  
    }  
    return footprint;  
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapSource.cpp中。

从这里就可以看出，当参数includeActive等于true时，函数oldHeapOverhead返回的是Zygote堆和Active堆的大小之和，而当参数includeActive等于false时，函数oldHeapOverhead仅仅返回Zygote堆的大小。注意，hs->heaps[0]指向的是Active堆，而hs->heaps[1]指向的是Zygote堆。这一点可以参考前面[Dalvik虚拟机Java堆创建过程分析](http://blog.csdn.net/luoshengyang/article/details/41581063)一文。

回到函数setIdealFootprint中，我们继续分析函数setSoftLimit的实现，如下所示：

```c
static void setSoftLimit(HeapSource *hs, size_t softLimit)  
{  
    ......  
    mspace msp = hs->heaps[0].msp;  
    size_t currentHeapSize = mspace_footprint(msp);  
    if (softLimit < currentHeapSize) {  
  ......  
  mspace_set_footprint_limit(msp, currentHeapSize);  
  hs->softLimit = softLimit;  
    } else {  
  ......  
  mspace_set_footprint_limit(msp, softLimit);  
  hs->softLimit = SIZE_MAX;  
    }  
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapSource.cpp中。

函数setSoftLimit首先是获得Active堆的当前大小currentHeapSize。如果参数softLimit的值小于Active堆的当前大小currentHeapSize，那么就意味着要给Active堆设置一个Soft Limit，这时候主要就是将参数softLimit的保存在hs->softLimit中。另一方面，如果参数softLimit的值大于等于Active堆的当前大小currentHeapSize，那么就意味着要去掉Active堆的Soft Limit，并且将Active堆的大小设置为参数softLimit的值。

回到函数dvmHeapSourceAlloc中，我们继续分析最后两个函数snapIdealFootprint和heapAllocAndGrow的实现，

函数snapIdealFootprint的实同如下所示：

```c
static void snapIdealFootprint()  
{  
    HS_BOILERPLATE();  
  
    setIdealFootprint(getSoftFootprint(true));  
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapSource.cpp中。

函数snapIdealFootprint通过调用前面分析的函数getSoftFootprint和setIdealFootprint来调整Active堆的大小以及Soft Limit值。回忆一下函数dvmHeapSourceAlloc调用snapIdealFootprint的情景，分别是在修改了Active堆的Soft Limit或者将Active堆的大小设置为允许的最大值，并且成功在Active堆分配了指定大小的内存之后进行的。这样就需要调用函数snapIdealFootprint将Active堆的大小设置为实际使用的大小（而不是允许的最大值），以及重新设置Soft Limit值。

函数heapAllocAndGrow的实现如下所示：

```c
static void* heapAllocAndGrow(HeapSource *hs, Heap *heap, size_t n)  
{  
    ......  
    size_t max = heap->maximumSize;  
  
    mspace_set_footprint_limit(heap->msp, max);  
    void* ptr = dvmHeapSourceAlloc(n);  
  
    ......  
    mspace_set_footprint_limit(heap->msp,  
                         mspace_footprint(heap->msp));  
    return ptr;  
}  
```
这个函数定义在文件dalvik/vm/alloc/HeapSource.cpp中。

函数heapAllocAndGrow使用最激进的办法来在参数heap描述的堆上分配内存。在我们这个情景中，参数heap描述的就是Active堆上。它首先将Active堆的大小设置为允许的最大值，接着再调用函数dvmHeapSourceAlloc在上面分配大小为n的内存。接着再通过函数mspace_footprint获得分配了n个字节之后Active堆的大小，并且将该值设置为Active堆的当前大小限制。这就相当于是将Active堆的当前大小限制值从允许设置的最大值减少为一个刚刚合适的值。

至此，我们就分析完成了Dalvik虚拟机为新创建的对象分配内存的过程。只有充分理解了对象内存的分配过程之后，我们才能够更好地理解对象内存的释放过程，也就是Dalvik虚拟机的垃圾收集过程。在接下来的一篇文章中，我们就将详细分析Dalvik虚拟机的垃圾收集过程，敬请关注！更多的信息也可以关注老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)。