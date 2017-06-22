ART运行时和Dalvik虚拟机一样，在堆上为对象分配内存时都要解决内存碎片和内存不足问题。内存碎片问题可以使用dlmalloc技术解决。内存不足问题则通过垃圾回收和在允许范围内增长堆大小解决。由于垃圾回收会影响程序，因此ART运行时采用力度从小到大的进垃圾回收策略。一旦力度小的垃圾回收执行过后能满足分配要求，那就不需要进行力度大的垃圾回收了。本文就详细分析ART运行时在堆上为对象分配内存的过程。

老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)，欢迎关注！

《[Android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

从前面[ART运行时Java堆创建过程分析](http://blog.csdn.net/luoshengyang/article/details/42379729)一文可以知道，在ART运行时中，主要用来分配对象的堆空间Zygote Space和Allocation Space的底层使用的都是匿名共享内存，并且通过C库提供的malloc和free接口来分进行管理。这样就可以通过dlmalloc技术来尽量解决碎片问题。这一点与我们在前面[Dalvik虚拟机为新创建对象分配内存的过程分析](http://blog.csdn.net/luoshengyang/article/details/41688319)一文提到的Dalvik虚拟机解决堆内存碎片问题的方法是一样的。因此，接下来在分析ART运行时为新创建对象分配的过程中，主要会分析它是如何解决内存不足的问题的。

ART运行时为新创建对象分配的过程如图1所示：

![img](http://img.blog.csdn.net/20150108105944855?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图1 ART运行时为新创建对象分配内存的过程

对比[Dalvik虚拟机为新创建对象分配内存的过程分析](http://blog.csdn.net/luoshengyang/article/details/41688319)一文的图2，可以发现，ART运行时和Dalvik虚拟机为新创建对象分配内存的过程几乎是一模一样的，它们的区别仅仅是在于垃圾收集的方式和策略不同。

从前面[Android运行时ART执行类方法的过程分析](http://blog.csdn.net/luoshengyang/article/details/40289405)一文可以知道，ART运行时为从DEX字节码翻译得到的Native代码提供的一个函数调用表中，有一个pAllocObject接口，是用来分配对象的。当ART运行时以Quick模式运行在ARM体系结构时，上述提到的pAllocObject接口由函数 `art_quick_alloc_object` 来实现。因此，接下来我们就从函数 `art_quick_alloc_object` 的实现开始分析ART运行时为新创建对象分配内存的过程。

函数 `art_quick_alloc_object` 的实现如下所示：

```c
    /* 
     * Called by managed code to allocate an object 
     */  
    .extern artAllocObjectFromCode  
ENTRY art_quick_alloc_object  
    SETUP_REF_ONLY_CALLEE_SAVE_FRAME  @ save callee saves in case of GC  
    mov    r2, r9                     @ pass Thread::Current  
    mov    r3, sp                     @ pass SP  
    bl     artAllocObjectFromCode     @ (uint32_t type_idx, Method* method, Thread*, SP)  
    RESTORE_REF_ONLY_CALLEE_SAVE_FRAME  
    RETURN_IF_RESULT_IS_NON_ZERO  
    DELIVER_PENDING_EXCEPTION  
END art_quick_alloc_object  
```
这个函数定义在文件art/runtime/arch/arm/quick_entrypoints_arm.S中。

这是一段ARM汇编，我们需要注意的一点是Native代码调用ART运行时提供的对象分配接口的参数传递方式。其中，参数 `type_idx` 描述的是要分配的对象的类型，通过寄存器r0传递，参数method描述的是当前调用的类方法，通过寄存器r1传递。

函数 `art_quick_alloc_object` 是通过调用另外一个函数artAllocObjectFromCode来分配对象的。函数 `art_quick_alloc_object` 除了传递前面描述的参数 `type_idx` 和method给函数artAllocObjectFromCode之外，还会传递另外的两个参数。其中一个是描述当前线程的一个Thread对象，该对象总是保存在寄存器r9中，现在由于要通过参数的形式传递给另外一个函数，因此就将它放在寄存器r2。另外一个是栈指针sp，也是由于要通过参数的形式的传递另外一个函数，这里也会将它放在寄存器r3中。

函数artAllocObjectFromCode的实现如下所示：

```c
extern "C" mirror::Object* artAllocObjectFromCode(uint32_t type_idx, mirror::ArtMethod* method,  
                                           Thread* self, mirror::ArtMethod** sp)  
    SHARED_LOCKS_REQUIRED(Locks::mutator_lock_) {  
FinishCalleeSaveFrameSetup(self, sp, Runtime::kRefsOnly);  
return AllocObjectFromCode(type_idx, method, self, false);  
}  
```
这个函数定义在文件art/runtime/entrypoints/quick/quick_alloc_entrypoints.cc中。

函数artAllocObjectFromCode又是通过调用另外一个函数AllocObjectFromCode来分配对象的。不过，在调用函数AllocObjectFromCode之前，函数artAllocObjectFromCode会先调用另外一个函数FinishCalleeSaveFrameSetup在当前调用栈帧中保存一个运行时信息。这个运行时信息描述的是接下来要调用的方法的类型为Runtime::kRefsOnly，也就是由被调用者保存那些不是用来传递参数的通用寄存器，即除了r0-r3的其它通用寄存器。

函数AllocObjectFromCode的实现如下所示：

```c
// Given the context of a calling Method, use its DexCache to resolve a type to a Class. If it  
// cannot be resolved, throw an error. If it can, use it to create an instance.  
// When verification/compiler hasn't been able to verify access, optionally perform an access  
// check.  
static inline mirror::Object* AllocObjectFromCode(uint32_t type_idx, mirror::ArtMethod* method,  
                                           Thread* self,  
                                           bool access_check)  
    SHARED_LOCKS_REQUIRED(Locks::mutator_lock_) {  
mirror::Class* klass = method->GetDexCacheResolvedTypes()->Get(type_idx);  
Runtime* runtime = Runtime::Current();  
if (UNLIKELY(klass == NULL)) {  
    klass = runtime->GetClassLinker()->ResolveType(type_idx, method);  
    if (klass == NULL) {  
      DCHECK(self->IsExceptionPending());  
      return NULL;  // Failure  
    }  
}  
if (access_check) {  
    if (UNLIKELY(!klass->IsInstantiable())) {  
      ThrowLocation throw_location = self->GetCurrentLocationForThrow();  
      self->ThrowNewException(throw_location, "Ljava/lang/InstantiationError;",  
                       PrettyDescriptor(klass).c_str());  
      return NULL;  // Failure  
    }  
    mirror::Class* referrer = method->GetDeclaringClass();  
    if (UNLIKELY(!referrer->CanAccess(klass))) {  
      ThrowIllegalAccessErrorClass(referrer, klass);  
      return NULL;  // Failure  
    }  
}  
if (!klass->IsInitialized() &&  
      !runtime->GetClassLinker()->EnsureInitialized(klass, true, true)) {  
    DCHECK(self->IsExceptionPending());  
    return NULL;  // Failure  
}  
return klass->AllocObject(self);  
}  
```
这个函数定义在文件art/runtime/entrypoints/entrypoint_utils.h中。

参数 `type_idx` 描述的是要分配的对象的类型，函数AllocObjectFromCode需要将它解析为一个Class对象，以便可以获得更多的信息进行内存分配。

函数AllocObjectFromCode首先是在当前调用类方法method的Dex Cache中检查是否已经存在一个与参数 `type_idx` 对应的Class对象。如果已经存在，那么就说明参数 `type_idx` 描述的对象类型已经被加载和解析过了，因此这时候就可以直接拿来使用。否则的话，就通过调用保存在当前运行时对象内部的一个ClassLinker对象的成员函数ResolveType来对参数 `type_idx` 描述的对象类型进行加载和解析。关于Dex Cache的知识，可以参数前面[Android运行时ART执行类方法的过程分析](http://blog.csdn.net/luoshengyang/article/details/40289405)一文，而对象类型（即类）的加载和解析过程可以参考前面[Android运行时ART加载类和方法的过程分析](http://blog.csdn.net/luoshengyang/article/details/39533503)一文。

得到了要分配的对象的类型klass之后，如果参数 `access_check` 的值等于true，那么就对该类型进行检查，即检查它是否可以实例化以及是否可以访问。如果检查通过，或者不需要检查，那么接下来还要确保类型klass是已经初始化过了的。前面的检查都没有问题之后，最后函数AllocObjectFromCode就调用Class类的成员函数AllocObject来分配一个类型为klass的对象。

Class类的成员函数AllocObject的实现如下所示：

```c
Object* Class::AllocObject(Thread* self) {  
......  
return Runtime::Current()->GetHeap()->AllocObject(self, this, this->object_size_);  
}  
```
这个函数定义在文件art/runtime/mirror/class.cc中。

这里我们就终于看到调用ART运行时内部的Heap对象的成员函数AllocObject在堆上分配对象了，其中，要分配的大小保存在当前Class对象的成员变量`object_size_`中。

Heap类的成员函数AllocObject的实现如下所示：

```c
mirror::Object* Heap::AllocObject(Thread* self, mirror::Class* c, size_t byte_count) {  
......  

mirror::Object* obj = NULL;  
size_t bytes_allocated = 0;  
......  

bool large_object_allocation =  
      byte_count >= large_object_threshold_ && have_zygote_space_ && c->IsPrimitiveArray();  
if (UNLIKELY(large_object_allocation)) {  
    obj = Allocate(self, large_object_space_, byte_count, &bytes_allocated);  
    ......  
} else {  
    obj = Allocate(self, alloc_space_, byte_count, &bytes_allocated);  
    ......  
}  

if (LIKELY(obj != NULL)) {  
    obj->SetClass(c);  
    ......  

    RecordAllocation(bytes_allocated, obj);  
    ......  

    if (UNLIKELY(static_cast<size_t>(num_bytes_allocated_) >= concurrent_start_bytes_)) {  
      ......  
      SirtRef<mirror::Object> ref(self, obj);  
      RequestConcurrentGC(self);  
    }  
    ......  

    return obj;  
} else {  
    ......  
    self->ThrowOutOfMemoryError(oss.str().c_str());  
    return NULL;  
}  
}  
```
这个函数定义在文件art/runtime/gc/heap.cc中。

Heap类的成员函数AllocObject首先是要确定要在哪个Space上分配内存。可以分配内存的Space有三个，分别Zygote Space、Allocation Space和Large Object Space。不过，Zygote Space在还没有划分出Allocation Space之前，就在Zygote Space上分配，而当Zygote Space划分出Allocation Space之后，就只能在Allocation Space上分配。从前面[ART运行时Java堆创建过程分析](http://blog.csdn.net/luoshengyang/article/details/42379729)一文可以知道，Heap类的成员变量`alloc_space_`在Zygote Space在还没有划分出Allocation Space之前指向Zygote Space，划分之后就指向Allocation Space。Large Object Space则始终由Heap类的成员变量`large_object_space_`指向。

只要满足以下三个条件，就在Large Object Space上分配，否则就在Zygote Space或者Allocation Space上分配：

1. 请求分配的内存大于等于Heap类的成员变量`large_object_threshold_`指定的值。这个值等于3 * kPageSize，即3个页面的大小。

2. 已经从Zygote Space划分出Allocation Space，即Heap类的成员变量`have_zygote_space_`的值等于true。

3. 被分配的对象是一个原子类型数组，即byte数组、int数组和boolean数组等。

确定好要在哪个Space上分配内存之后，就可以调用Heap类的成员函数Allocate进行分配了。如果分配成功，Heap类的成员函数Allocate就返回新分配的对象，保存在变量obj中。接下来再做三件事情：

1. 调用Object类的成员函数SetClass设置新分配对象obj的类型。

2. 调用Heap类的成员函数RecordAllocation记录当前的内存分配状况。

3. 检查当前已经分配出去的内存是否已经达到由Heap类的成员变量`concurrent_start_bytes_`设定的阀值。如果达到，那么就调用Heap类的成员函数RequestConcurrentGC通知GC执行一次并行GC。关于执行并行GC的阀值，接下来分析ART运行时的垃圾收集过程中再详细分析。

另一方面，如果Heap类的成员函数Allocate分配内存失败，则Heap类的成员函数AllocObject抛出一个OOM异常。

接下来，我们先分析Heap类的成员函数RecordAllocation的实现，接着再分析Heap类的成员函数Allocate的实现。因为后者的执行流程比较复杂，而前者的执行流程比较简单。我们先分析容易的，以免打断后面的分析。

Heap类的成员函数RecordAllocation的实现如下所示：

```c
inline void Heap::RecordAllocation(size_t size, mirror::Object* obj) {  
DCHECK(obj != NULL);  
DCHECK_GT(size, 0u);  
num_bytes_allocated_.fetch_add(size);  

if (Runtime::Current()->HasStatsEnabled()) {  
    RuntimeStats* thread_stats = Thread::Current()->GetStats();  
    ++thread_stats->allocated_objects;  
    thread_stats->allocated_bytes += size;  

    // TODO: Update these atomically.  
    RuntimeStats* global_stats = Runtime::Current()->GetStats();  
    ++global_stats->allocated_objects;  
    global_stats->allocated_bytes += size;  
}  

// This is safe to do since the GC will never free objects which are neither in the allocation  
// stack or the live bitmap.  
while (!allocation_stack_->AtomicPushBack(obj)) {  
    CollectGarbageInternal(collector::kGcTypeSticky, kGcCauseForAlloc, false);  
}  
}  
```
这个函数定义在文件art/runtime/gc/heap.cc中。

Heap类的成员函数RecordAllocation首先是记录当前已经分配的内存字节数以及对象数，接着再将新分配的对象压入到Heap类的成员变量`allocation_stack_`描述的Allocation Stack中去。后面这一点与Dalvik虚拟机的做法是不一样的。Dalvik虚拟机直接将新分配出来的对象记录在Live Bitmap中，具体可以参考前面[Dalvik虚拟机为新创建对象分配内存的过程分析](http://blog.csdn.net/luoshengyang/article/details/41688319)一文。ART运行时之所以要将新分配的对象压入到Allocation Stack中去，是为了以后可以执行Sticky GC。

注意，如果不能成功将新分配的对角压入到Allocation Stack中，就说明上次GC以来，新分配的对象太多了，因此这时候就需要执行一个Sticky GC，将Allocation Stack里面的垃圾进行回收，然后再尝试将新分配的对象压入到Allocation Stack中，直到成功为止。

接下来我们就重点分析Heap类的成员函数Allocate的实现，以便可以了解新创建对象在堆上分配的具体过程，如下所示：

```c
template <class T>  
inline mirror::Object* Heap::Allocate(Thread* self, T* space, size_t alloc_size,  
                               size_t* bytes_allocated) {  
......  

mirror::Object* ptr = TryToAllocate(self, space, alloc_size, false, bytes_allocated);  
if (ptr != NULL) {  
    return ptr;  
}  
return AllocateInternalWithGc(self, space, alloc_size, bytes_allocated);  
}  
```
这个函数定义在文件art/runtime/gc/heap.cc中。

Heap类的成员函数Allocate首先调用成员函数TryToAllocate尝试在不执行GC的情况下进行内存分配。如果分配失败，再调用成员函数AllocateInternalWithGc进行带GC的内存分配。

Heap类的成员函数Allocate是一个模板函数，不同类型的Space会导致调用不同重载的成员函数TryToAllocate进行不带GC的内存分配。虽然可以用来分配内存的Space有Zygote Space、Allocation Space和Large Object Space三个，但是前两者的类型是相同的，因此实际上只有两个不同重载版本的成员函数TryToAllocate，它们的实现如下所示：

```c
inline mirror::Object* Heap::TryToAllocate(Thread* self, space::AllocSpace* space, size_t alloc_size,  
                                    bool grow, size_t* bytes_allocated) {  
if (UNLIKELY(IsOutOfMemoryOnAllocation(alloc_size, grow))) {  
    return NULL;  
}  
return space->Alloc(self, alloc_size, bytes_allocated);  
}  

// DlMallocSpace-specific version.  
inline mirror::Object* Heap::TryToAllocate(Thread* self, space::DlMallocSpace* space, size_t alloc_size,  
                                    bool grow, size_t* bytes_allocated) {  
if (UNLIKELY(IsOutOfMemoryOnAllocation(alloc_size, grow))) {  
    return NULL;  
}  
if (LIKELY(!running_on_valgrind_)) {  
    return space->AllocNonvirtual(self, alloc_size, bytes_allocated);  
} else {  
    return space->Alloc(self, alloc_size, bytes_allocated);  
}  
}  
```
这两个函数定义在文件art/runtime/gc/heap.cc中。

Heap类两个重载版本的成员函数TryToAllocate的实现逻辑都几乎是相同的，首先是调用另外一个成员函数IsOutOfMemoryOnAllocation判断分配请求的内存后是否会超过堆的大小限制。如果超过，则分配失败；否则的话再在指定的Space进行内存分配。

Heap类的成员函数IsOutOfMemoryOnAllocation的实现如下所示：

```c
inline bool Heap::IsOutOfMemoryOnAllocation(size_t alloc_size, bool grow) {  
size_t new_footprint = num_bytes_allocated_ + alloc_size;  
if (UNLIKELY(new_footprint > max_allowed_footprint_)) {  
    if (UNLIKELY(new_footprint > growth_limit_)) {  
      return true;  
    }  
    if (!concurrent_gc_) {  
      if (!grow) {  
 return true;  
      } else {  
 max_allowed_footprint_ = new_footprint;  
      }  
    }  
}  
return false;  
}  
```
这个函数定义在文件art/runtime/gc/heap.cc中。

Heap类的成员变量`num_bytes_allocated_`描述的是目前已经分配出去的内存字节数，成员变量max_allowed_footprint_描述的是目前堆可分配的最大内存字节数，成员变量`growth_limit_`描述的是目前堆允许增长到的最大内存字节数。这里需要注意的一点是，`max_allowed_footprint_`是Heap类施加的一个限制，不会对各个Space实际可分配的最大内存字节数产生影响，并且各个Space在创建的时候，已经把自己可分配的最大内存数设置为允许使用的最大内存字节数的。

如果目前堆已经分配出去的内存字节数再加上请求分配的内存字节数`new_footprint`小于等于目前堆可分配的最大内存字节数`max_allowed_footprint_`，那么分配出请求的内存字节数之后不会造成OOM，因此Heap类的成员函数IsOutOfMemoryOnAllocation就返回false。

另一方面，如果目前堆已经分配出去的内存字节数再加上请求分配的内存字节数`new_footprint`大于目前堆可分配的最大内存字节数`max_allowed_footprint_`，并且也大于目前堆允许增长到的最大内存字节数`growth_limit_`，那么分配出请求的内存字节数之后造成OOM，因此Heap类的成员函数IsOutOfMemoryOnAllocation就返回true。

剩下另外一种情况，目前堆已经分配出去的内存字节数再加上请求分配的内存字节数 `new_footprint` 大于目前堆可分配的最大内存字节数 `max_allowed_footprint_` ，但是小于等于目前堆允许增长到的最大内存字节数growth_limit_，这时候就要看情况会不会出现OOM了。如果ART运行时运行在非并行GC的模式中，即Heap类的成员变量concurrent_gc_等于false，那么取决于允不允许增长堆的大小，即参数grow的值。如果不允许，那么Heap类的成员函数IsOutOfMemoryOnAllocation就返回true，表示当前请求的分配会造成OOM。如果允许，那么Heap类的成员函数IsOutOfMemoryOnAllocation就会修改目前堆可分配的最大内存字节数 `max_allowed_footprint_` ，并且返回false，表示允许当前请求的分配。这意味着，在非并行GC运行模式中，在分配内存过程中遇到内存不足，并且当前可分配内存还未达到增长上限时，要等到执行完成一次非并行GC后，才能成功分配到内存，因为每次执行完成GC之后，都会按照预先设置的堆目标利用率来增长堆的大小。

另一方面，如果ART运行时运行在并行GC的模式中，那么只要前堆已经分配出去的内存字节数再加上请求分配的内存字节数new_footprint不地超过目前堆允许增长到的最大内存字节数growth_limit_，那么就不管允不允许增长堆的大小，都认为不会发生OOM，因此Heap类的成员函数IsOutOfMemoryOnAllocation就返回false。这意味着，在并行GC运行模式中，在分配内存过程中遇到内存不足，并且当前可分配内存还未达到增长上限时，无非等到执行并行GC后，就有可能成功分配到内存，因为实际执行内存分配的Space可分配的最大内存字节数是足够的。

回到前面Heap类的成员函数TryToAllocate中，从前面[ART运行时Java堆创建过程分析](http://blog.csdn.net/luoshengyang/article/details/42379729)一文可以知道，对于Large Object Space版本的成员函数TryToAllocate，调用的是LargeObjectMapSpace类的成员函数Alloc进行内存分配，而对于Zygote Space或者Allocation Space版本的成员函数TryToAllocate，如果成员变量 `running_on_valgrind_` 的值等于true，就调用ValgrindDlMallocSpace类的成员函数AllocNonvirtual进行内存分配，否则就调用DlMallocSpace类的成员函数Alloc进行内存分配。我们假设Heap类的成员变量 `running_on_valgrind_` 的值等于false，因此接下来我们主要分析LargeObjectMapSpace类的成员函数Alloc和DlMallocSpace类的成员函数Alloc的实现。

LargeObjectMapSpace类的成员函数Alloc的实现如下所示：

```c
mirror::Object* LargeObjectMapSpace::Alloc(Thread* self, size_t num_bytes, size_t* bytes_allocated) {  
MemMap* mem_map = MemMap::MapAnonymous("large object space allocation", NULL, num_bytes,  
                                  PROT_READ | PROT_WRITE);  
if (mem_map == NULL) {  
    return NULL;  
}  
MutexLock mu(self, lock_);  
mirror::Object* obj = reinterpret_cast<mirror::Object*>(mem_map->Begin());  
large_objects_.push_back(obj);  
mem_maps_.Put(obj, mem_map);  
size_t allocation_size = mem_map->Size();  
DCHECK(bytes_allocated != NULL);  
*bytes_allocated = allocation_size;  
num_bytes_allocated_ += allocation_size;  
total_bytes_allocated_ += allocation_size;  
++num_objects_allocated_;  
++total_objects_allocated_;  
return obj;  
}  
```
这个函数定义在文件art/runtime/gc/space/large_object_space.cc中。

从这里就可以看到，Large Object Map Space分配内存的逻辑是很简单的，直接就是调用MemMap类的静态成员函数MapAnonymous创建一块指定大小的匿名内存，然后再将该匿名共享内存添加到成员变量large_objects_描述的一个向量中去，最后更新内部的各个统计数据。

DlMallocSpace类的成员函数Alloc的实现如下所示：

```c
mirror::Object* DlMallocSpace::Alloc(Thread* self, size_t num_bytes, size_t* bytes_allocated) {  
return AllocNonvirtual(self, num_bytes, bytes_allocated);  
}  
```
这个函数定义在文件art/runtime/gc/space/dlmalloc_space.cc中。

DlMallocSpace类的成员函数Alloc调用另外一个成员函数AllocNonvirtual来进行内存分配，后者的实现如下所示：

```c
inline mirror::Object* DlMallocSpace::AllocNonvirtual(Thread* self, size_t num_bytes,  
                                               size_t* bytes_allocated) {  
mirror::Object* obj;  
{  
    MutexLock mu(self, lock_);  
    obj = AllocWithoutGrowthLocked(num_bytes, bytes_allocated);  
}  
if (obj != NULL) {  
    // Zero freshly allocated memory, done while not holding the space's lock.  
    memset(obj, 0, num_bytes);  
}  
return obj;  
}  
```
这个函数定义在文件art/runtime/gc/space/dlmalloc_space-inl.h中。

DlMallocSpace类的成员函数AllocNonvirtual首先是调用另外一个成员函数AllocWithoutGrowthLocked在不增长Space的大小的前提下进行内存分配，分配成功之后再调用函数memset对分配出来的内存进行清空，最后将分配出来的内存返回给调用者。

DlMallocSpace类的成员函数AllocWithoutGrowthLocked的实现如下所示：

```c
inline mirror::Object* DlMallocSpace::AllocWithoutGrowthLocked(size_t num_bytes, size_t* bytes_allocated) {  
mirror::Object* result = reinterpret_cast<mirror::Object*>(mspace_malloc(mspace_, num_bytes));  
if (result != NULL) {  
    if (kDebugSpaces) {  
      CHECK(Contains(result)) << "Allocation (" << reinterpret_cast<void*>(result)  
     << ") not in bounds of allocation space " << *this;  
    }  
    size_t allocation_size = AllocationSizeNonvirtual(result);  
    DCHECK(bytes_allocated != NULL);  
    *bytes_allocated = allocation_size;  
    num_bytes_allocated_ += allocation_size;  
    total_bytes_allocated_ += allocation_size;  
    ++total_objects_allocated_;  
    ++num_objects_allocated_;  
}  
return result;  
}  
```
这个函数定义在文件art/runtime/gc/space/dlmalloc_space-inl.h中。

从前面[ART运行时Java堆创建过程分析](http://blog.csdn.net/luoshengyang/article/details/42379729)一文可以知道，DlMallocSpace底层使用的匿名共享内存块被封装成一个mspace对象，并且保存在成员变量mspace_中，因此这里就可以直接调用C库提供的mspace_malloc接口进行内存分配。使用mspace_malloc分配的内存会自动被清空，因此这里不用再手动清空。DlMallocSpace类的成员函数AllocWithoutGrowthLocked在将分配出来的内存返回给调用者之前，同样是会更新内部的各个统计数据。

回到前面Heap类的成员函数Allocate中，在调用成员函数TryToAllocate不能成功分配指定大小的内存块之后，接下来就继续调用成员函数AllocateInternalWithGc执行带GC的内存分配，它的实现如下所示：

```c
mirror::Object* Heap::AllocateInternalWithGc(Thread* self, space::AllocSpace* space,  
                                      size_t alloc_size, size_t* bytes_allocated) {  
mirror::Object* ptr;  

// The allocation failed. If the GC is running, block until it completes, and then retry the  
// allocation.  
collector::GcType last_gc = WaitForConcurrentGcToComplete(self);  
if (last_gc != collector::kGcTypeNone) {  
    // A GC was in progress and we blocked, retry allocation now that memory has been freed.  
    ptr = TryToAllocate(self, space, alloc_size, false, bytes_allocated);  
    if (ptr != NULL) {  
      return ptr;  
    }  
}  

// Loop through our different Gc types and try to Gc until we get enough free memory.  
for (size_t i = static_cast<size_t>(last_gc) + 1;  
      i < static_cast<size_t>(collector::kGcTypeMax); ++i) {  
    bool run_gc = false;  
    collector::GcType gc_type = static_cast<collector::GcType>(i);  
    switch (gc_type) {  
      case collector::kGcTypeSticky: {  
   const size_t alloc_space_size = alloc_space_->Size();  
   run_gc = alloc_space_size > min_alloc_space_size_for_sticky_gc_ &&  
       alloc_space_->Capacity() - alloc_space_size >= min_remaining_space_for_sticky_gc_;  
   break;  
 }  
      case collector::kGcTypePartial:  
 run_gc = have_zygote_space_;  
 break;  
      case collector::kGcTypeFull:  
 run_gc = true;  
 break;  
      default:  
 break;  
    }  

    if (run_gc) {  
      // If we actually ran a different type of Gc than requested, we can skip the index forwards.  
      collector::GcType gc_type_ran = CollectGarbageInternal(gc_type, kGcCauseForAlloc, false);  
      DCHECK_GE(static_cast<size_t>(gc_type_ran), i);  
      i = static_cast<size_t>(gc_type_ran);  

      // Did we free sufficient memory for the allocation to succeed?  
      ptr = TryToAllocate(self, space, alloc_size, false, bytes_allocated);  
      if (ptr != NULL) {  
 return ptr;  
      }  
    }  
}  

// Allocations have failed after GCs;  this is an exceptional state.  
// Try harder, growing the heap if necessary.  
ptr = TryToAllocate(self, space, alloc_size, true, bytes_allocated);  
if (ptr != NULL) {  
    return ptr;  
}  

......  

// We don't need a WaitForConcurrentGcToComplete here either.  
CollectGarbageInternal(collector::kGcTypeFull, kGcCauseForAlloc, true);  
return TryToAllocate(self, space, alloc_size, true, bytes_allocated);  
}  
```
这个函数定义在文件art/runtime/gc/heap.cc中。

Heap类的成员函数AllocateInternalWithGc主要是通过垃圾回收来满足请求分配的内存，它的执行逻辑如下所示：

1. 调用Heap类的成员函数WaitForConcurrentGcToComplete检查是否有并行GC正在执行。如果有的话，就等待其执行完成，并且得到它的类型last_gc。如果last_gc如果不等于collector::kGcTypeNone，就表示有并行GC并且已经执行完成，因此就可以调用Heap类的成员函数TryToAllocate在不增长当前堆大小的前提下再次尝试分配请求的内存了。如果分配成功，则返回得到的内存起始地址给调用者。否则的话，继续往下执行。

2.  依次执行kGcTypeSticky、kGcTypePartial和kGcTypeFull三种类型的GC。每次GC执行完毕，都尝试调用Heap类的成员函数TryToAllocate在不增长当前堆大小的前提下再次尝试分配请求的内存。如果分配内存成功，则返回得到的内存起始地址给调用者，并且不再执行下一种类型的GC。

这里需要注意的一点是，kGcTypeSticky、kGcTypePartial和kGcTypeFull三种类型的GC的垃圾回收力度是依次加强：kGcTypeSticky只回收上次GC后在Allocation Space中新分配的垃圾对象；kGcTypePartial只回收Allocation Space的垃圾对象；kGcTypeFull同时回收Zygote Space和Allocation Space的垃圾对象。通过这种策略，就有可能以最小代价解决分配对象时遇到的内在不足问题。不过，对于类型为kGcTypeSticky和kGcTypePartial的GC，它们的执行有前提条件的。

类型为kGcTypeSticky的GC的执行代码虽然是最小的，但是它能够回收的垃圾也是最小的。如果回收的垃圾不足于满足请求分配的内存，那就相当于做了一次无用功了。因此，执行类型为kGcTypeSticky的GC需要满足两个条件。第一个条件是上次GC后在Allocation Space上分配的内存要达到一定的阀值，这样才有比较大的概率回收到较多的内存。第二个条件Allocation Space剩余的未分配内存要达到一定的阀值，这样可以保证在回收得到较少内存时，也有比较大的概率满足请求分配的内存。前一个阀值定义在Heap类的成员变量`min_alloc_space_size_for_sticky_gc_`中，它的值设置为2M，而上次GC以来分配的内存通过当前Allocation Space的大小估算得到，即通过调用Heap类的成员变量alloc_space_指向的一个DlMallocSpace对象的成员函数Size获得。后一个阀值定义在Heap类的成员变量`min_remaining_space_for_sticky_gc_`中，它的值设置为1M，而Allocation Space剩余的未分配内存可以用Allocation Space的总大小减去当前Allocation Space的大小得到。通过调用Heap类的成员变量alloc_space_指向的一个DlMallocSpace对象的成员函数Capacity获得其总大小。

类型为kGcTypePartial的GC的执行前提是已经从Zygote Space中划分出Allocation Space。从前面[ART运行时Java堆创建过程分析](http://blog.csdn.net/luoshengyang/article/details/42379729)一文可以知道，当Heap类的成员变量 `have_zygote_space_` 的值等于true时，就表明已经从Zygote Space中划分出Allocation Space了。因此，在这种情况下，就可以执行类型为kGcTypePartial的GC了。

每一种类型的GC都是通过调用Heap类的成员函数CollectGarbageInternal来执行。注意这时候调用Heap类的成员函数CollectGarbageInternal传递的第三个参数为false，表示不对那些只被软引用对象引用的对象进行回收。如果上述的三种类型的GC执行完毕，还是不能满足分配请求的内存，则继续往下执行。

3. 经过前面三种类型的GC后还是不能成功分配到内存，那就说明能够回收的内存还是太小了，因此，这时候只能通过在允许范围内增长堆的大小来满足内存分配请求了。前面分析Heap类的成员函数TryToAllocate时，将第四个参数设置为true，即可在允许范围内增长堆大小的前提下进行内存分配。如果在允许范围内增长了堆的大小还是不能成功分配到请求的内存，那就只能出最后的一个大招了。

4.  最后的大招是首先执行一个类型为kGcTypeFull的、要求回收那些只被软引用对象引用的对象的GC，接着再在允许范围内增长堆大小的前提下尝试分配内存。这一次如果还是失败，那就真的是内存不足了。

至此，我们就对ART运行时在堆上为新创建对象分配内存的过程分析完成了。从中我们就可以看到，ART运行时面临的最大挑战就是内存不足问题，它要通过在允许范围内增长堆大小以及垃圾回收两个手段来解决。其中，垃圾回收会对程序造成影响，因此在执行垃圾回收时，使用的力度要从小到大。在接下来的一篇文章中，我们就将详细分析这些力度不同的垃圾回收是如何实现的，敬请关注！更多的信息也可以关注老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)。