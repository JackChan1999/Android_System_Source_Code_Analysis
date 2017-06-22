与Dalvik虚拟机一样，ART运行时内部也有一个[Java](http://lib.csdn.net/base/java)堆，用来分配Java对象。当这些Java对象不再被使用时，ART运行时需要回收它们占用的内存。在前面一文中，我们简要介绍了ART运行时的垃圾收集机制，从中了解到ART运行时内部使用的Java堆是由四种Space以及各种辅助[数据结构](http://lib.csdn.net/base/datastructure)共同描述的。为了后面可以更好地分析ART运行时的垃圾收集机制，本文就对它内部使用的Java堆的创建过程进行分析。

老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)，欢迎关注！

《[Android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

ART运行时创建Java堆的过程，实际上就是创建在图1涉及到的各种数据结构：

![img](http://img.blog.csdn.net/20150104012422401?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图1 ART运行时堆的基本概念以及组成

从图1可以看到，ART运行时内部使用的Java堆的主要组成包括Image Space、Zygote Space、Allocation Space和Large Object Space四个Space，两个Mod Union Table，一个Card Table，两个Heap Bitmap，两个Object Map，以及三个Object Stack。这些数据结构的介绍和作用可以参考前面[ART运行时垃圾收集机制简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/42072975)一文，接下来我们主要是分析它们的创建过程。

从前面[Android运行时ART加载OAT文件的过程分析](http://blog.csdn.net/luoshengyang/article/details/39307813)一文可以知道，ART运行时内部使用的堆是在ART运行时初始化过程中创建的，即在Runtime类的成员函数Init中创建的，如下所示：

```c
bool Runtime::Init(const Options& raw_options, bool ignore_unrecognized) {    
......    
    
UniquePtr<ParsedOptions> options(ParsedOptions::Create(raw_options, ignore_unrecognized));    
......    
    
heap_ = new gc::Heap(options->heap_initial_size_,    
               options->heap_growth_limit_,    
               options->heap_min_free_,    
               options->heap_max_free_,    
               options->heap_target_utilization_,    
               options->heap_maximum_size_,    
               options->image_,    
               options->is_concurrent_gc_enabled_,    
               options->parallel_gc_threads_,    
               options->conc_gc_threads_,    
               options->low_memory_mode_,    
               options->long_pause_log_threshold_,    
               options->long_gc_log_threshold_,    
               options->ignore_max_footprint_);    

......    
}  
```
这个函数定义在文件art/runtime/runtime.cc中。

Runtime类的成员函数Init首先是调用ParsedOptions类的静态成员函数Create解析ART运行时的启动选项，并且保存在变量options指向的一个ParsedOptions对象的各个成员变量中，与堆相关的各个选项的含义如下所示：

1. options->heap_initial_size_: 堆的初始大小，通过选项-Xms指定。

2. options->heap_growth_limit_: 堆允许增长的上限值，这是堆的一个软上限值，通过选项-XX:HeapGrowthLimit指定。

3. options->heap_min_free_: 堆的最小空闲值，通过选项-XX:HeapMinFree指定。

4. options->heap_max_free_: 堆的最大空闲值，通过选项-XX:HeapMaxFree指定。

5. options->heap_target_utilization_: 堆的目标利用率，通过选项-XX:HeapTargetUtilization指定。

6. options->heap_maximum_size_: 堆的最大值，这是堆的一个硬上限值，通过选项-Xmx指定。

7. options->image_: 用来创建Image Space的Image文件，通过选项-Ximage指定。

8. options->is_concurrent_gc_enabled_: 是否支持并行GC，通过选项-Xgc指定。

9. options->parallel_gc_threads_: GC暂停阶段用于同时执行GC任务的线程数，通过选项-XX:ParallelGCThreads指定。

10. options->conc_gc_threads_: GC非暂停阶段用于同时执行GC任务的线程数，通过选项-XX:ConcGCThreads指定。

11. options->low_memory_mode_: 是否在低内存模式运行，通过选项XX:LowMemoryMode指定。

12. options->long_pause_log_threshold_: GC造成应用程序暂停的时间阀值，一旦超过该阀值，则输出警告日志，通过选项XX:LongPauseLogThreshold指定。

13. options->long_gc_log_threshold_: GC时间阀值，一旦超过该阀值，则输出警告日志，通过选项-XX:LongGCLogThreshold指定。

14. options->ignore_max_footprint_: 不对堆的大小进行限制标志，通过选项-XX:IgnoreMaxFootprint指定。

在上述14个选项中，除了后面的3个选项，其余的选项在前面[Dalvik虚拟机垃圾收集机制简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/41338251)这个系列的文章和[ART运行时垃圾收集机制简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/42072975)这篇文章均有详细解释过。

接下来我们就继续分析Heap类的构造函数，以便可以了解堆的创建过程，如下所示：

```c
static constexpr bool kGCALotMode = false;  
static constexpr size_t kGcAlotInterval = KB;  
......  

Heap::Heap(size_t initial_size, size_t growth_limit, size_t min_free, size_t max_free,  
   double target_utilization, size_t capacity, const std::string& original_image_file_name,  
   bool concurrent_gc, size_t parallel_gc_threads, size_t conc_gc_threads,  
   bool low_memory_mode, size_t long_pause_log_threshold, size_t long_gc_log_threshold,  
   bool ignore_max_footprint)  
    : ......,  
      concurrent_gc_(concurrent_gc),  
      parallel_gc_threads_(parallel_gc_threads),  
      conc_gc_threads_(conc_gc_threads),  
      low_memory_mode_(low_memory_mode),  
      long_pause_log_threshold_(long_pause_log_threshold),  
      long_gc_log_threshold_(long_gc_log_threshold),  
      ignore_max_footprint_(ignore_max_footprint),  
      ......,  
      capacity_(capacity),  
      growth_limit_(growth_limit),  
      max_allowed_footprint_(initial_size),  
      ......,  
      max_allocation_stack_size_(kGCALotMode ? kGcAlotInterval  
  : (kDesiredHeapVerification > kNoHeapVerification) ? KB : MB),  
      ......,  
      min_free_(min_free),  
      max_free_(max_free),  
      target_utilization_(target_utilization),  
      ...... {  
......  

live_bitmap_.reset(new accounting::HeapBitmap(this));  
mark_bitmap_.reset(new accounting::HeapBitmap(this));  

// Requested begin for the alloc space, to follow the mapped image and oat files  
byte* requested_alloc_space_begin = NULL;  
std::string image_file_name(original_image_file_name);  
if (!image_file_name.empty()) {  
    space::ImageSpace* image_space = space::ImageSpace::Create(image_file_name);  
    ......  
    AddContinuousSpace(image_space);  
    // Oat files referenced by image files immediately follow them in memory, ensure alloc space  
    // isn't going to get in the middle  
    byte* oat_file_end_addr = image_space->GetImageHeader().GetOatFileEnd();  
    ......  
    if (oat_file_end_addr > requested_alloc_space_begin) {  
      requested_alloc_space_begin =  
  reinterpret_cast<byte*>(RoundUp(reinterpret_cast<uintptr_t>(oat_file_end_addr),  
                                  kPageSize));  
    }  
}  

alloc_space_ = space::DlMallocSpace::Create(Runtime::Current()->IsZygote() ? "zygote space" : "alloc space",  
                                      initial_size,  
                                      growth_limit, capacity,  
                                      requested_alloc_space_begin);  
......  
alloc_space_->SetFootprintLimit(alloc_space_->Capacity());  
AddContinuousSpace(alloc_space_);  

// Allocate the large object space.  
const bool kUseFreeListSpaceForLOS = false;  
if (kUseFreeListSpaceForLOS) {  
    large_object_space_ = space::FreeListSpace::Create("large object space", NULL, capacity);  
} else {  
    large_object_space_ = space::LargeObjectMapSpace::Create("large object space");  
}  
......  
AddDiscontinuousSpace(large_object_space_);  

// Compute heap capacity. Continuous spaces are sorted in order of Begin().  
byte* heap_begin = continuous_spaces_.front()->Begin();  
size_t heap_capacity = continuous_spaces_.back()->End() - continuous_spaces_.front()->Begin();  
if (continuous_spaces_.back()->IsDlMallocSpace()) {  
    heap_capacity += continuous_spaces_.back()->AsDlMallocSpace()->NonGrowthLimitCapacity();  
}  

// Allocate the card table.  
card_table_.reset(accounting::CardTable::Create(heap_begin, heap_capacity));  
......  

image_mod_union_table_.reset(new accounting::ModUnionTableToZygoteAllocspace(this));  
......  

zygote_mod_union_table_.reset(new accounting::ModUnionTableCardCache(this));  
......  

// Default mark stack size in bytes.  
static const size_t default_mark_stack_size = 64 * KB;  
mark_stack_.reset(accounting::ObjectStack::Create("mark stack", default_mark_stack_size));  
allocation_stack_.reset(accounting::ObjectStack::Create("allocation stack",  
                                                  max_allocation_stack_size_));  
live_stack_.reset(accounting::ObjectStack::Create("live stack",  
                                            max_allocation_stack_size_));  

......  

if (ignore_max_footprint_) {  
    SetIdealFootprint(std::numeric_limits<size_t>::max());  
    concurrent_start_bytes_ = max_allowed_footprint_;  
}  

// Create our garbage collectors.  
for (size_t i = 0; i < 2; ++i) {  
    const bool concurrent = i != 0;  
    mark_sweep_collectors_.push_back(new collector::MarkSweep(this, concurrent));  
    mark_sweep_collectors_.push_back(new collector::PartialMarkSweep(this, concurrent));  
    mark_sweep_collectors_.push_back(new collector::StickyMarkSweep(this, concurrent));  
}  

......  
}  
```
这个函数定义在文件art/runtime/gc/heap.cc中。

Heap类的构造函数就负责创建上面图1所示的各种数据结构，创建过程如下所示：

创建Live Bitmap和Mark Bitmap，它们都是使用一个HeapBitmap对象来描述，并且分别保存在成员变量live_bitmap_和mark_bitmap_中。

如果指定了image文件，即参数 `original_image_file_name` 的值不等于空，则调用ImageSpace类的静态成员函数Create创建一个Image Space，并且调用Heap类的成员函数AddContinuousSpace将该Image Space添加到一个在地址空间上连续的Space列表中，即Image Space属于地址空间连续的Space。在Image文件的头部，指定了与Image文件关联的boot.art@classes.oat文件加载到内存的结束位置，我们需要将这个位置取出来，并且将它对齐到页面大小，保存变量 `requested_alloc_space_begin` 中，作为一会要创建的Zygote Space的开始地址。这样就可以使得Image Space和Zygote Space的布局与图1所示保持一致。

调用DlMallocSpace类的静态成员函数Create创建一个Zygote Space，注意最后一个参数 `requested_alloc_space_begin` ，指定了Zygote Space的起始地址，它刚刚好是紧跟在boot.art@classes.oat文件的后面。这时候代码是运行在Zygote进程中，而Zygote进程中的ART运行时刚开始的时候是没有Zygote Space和Allocation Space之分的。等到Zygote进程fork第一个子进程的时候，才会将这里创建的Zygote Space一分为二，得到一个Zygote Space和一个Allocation Space。这一点与前面[Dalvik虚拟机Java堆创建过程分析](http://blog.csdn.net/luoshengyang/article/details/41581063)一文分析Dalvik虚拟机堆的管理是一样的。由于这里创建的Zygote Space也是一个地址空间连续的Space，因此它也会被Heap类的成员函数AddContinuousSpace添加一个在地址空间上连续的Space列表中。

接下来是创建Large Object Space。正如前面[ART运行时垃圾收集机制简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/42072975)一文所述，ART运行时提供了两种Large Object Space，其中一种是Free List实现FreeListSpace，另外一种是由一组相互独立的内存块组成的LargeObjectMapSpace。这里由于kUseFreeListSpaceForLOS的值设置为false，因此Large Object Space使用的是后一种实现，通过LargeObjectMapSpace类的静态成员函数Create来创建。注意，这里创建的Large Object Space属于地址空间不连续的Space，因此需要调用Heap类的另外一个成员函数AddDiscontinuousSpace将它添加到内部一个在地址空间上不连续的Space列表中。

5.接下来是计算所有在地址空间上连续的Space占用的内存的大小。此时，地址空间上连续的Space有两个，分别是Image Space和Zygote Space，它们按照地址从小到大的顺序保存在Heap类的成员变量continuous_spaces_描述的一个向量中。由于Image Space的地址小于Zygote Space的地址，因此，保存在Heap类的成员变量continuous_spaces_描述的向量的第一个元素是Image Space，第二个元素是Zygote Space。注意变量heap_capacity的计算，它使用Zygote Space的结束地址减去Image Space的起始地址，得到的大小实际上是包含了图1所示的boot.art@classes.oat文件映射到内存的大小。此外，变量heap_capacity只是包含了Zygote Space的初始大小，即通过continuous_spaces_.back()->End()得到的大小并没有真实地反映Zygote Space的真实大小，因此这时候需要获得Zygote Space的真实大小，并且增加到变量heap_capacity去。首先，continuous_spaces_.back()返回的是一个ContinuousSpace指针，但是它实际上指向的是一个子类DlMallocSpace对象。其次，DlMallocSpace提供了函数NonGrowthLimitCapacity用来获得真实大小。因此，将continuous_spaces_.back()返回的是一个ContinuousSpace指针转换为一个DlMallocSpace指针，就可以调用其成员函数NonGrowthLimitCapacity来获得Zygote Space的真实大小，并且添加到变量heap_capacity中去。从这里我们也可以看到，变量heap_capacity描述的是并不是准确的Image Space和Zygote Space的大小之和，而是一个比它们的和要大的一个值。但是变量heap_capacity是用来创建Card Table的，因此它的值比真实的Space的大小大一些不会影响程序的逻辑正确性。

6.有了所有在地址空间上连续的Space的大小之和heap_capacity，以及地址最小的Space的起始地址heap_begin之后，就可以调用CardTable类的静态成员函数Create来创建一个Card Table了。这里的Card Table的创建过程与前面[Dalvik虚拟机Java堆创建过程分析](http://blog.csdn.net/luoshengyang/article/details/41581063)一文分析的Dalvik虚拟机内部使用的Card Table的创建过程是一样的。从这里我们也可以看出，只有地址空间连续的Space才具有Card Table。

7.接下来是创建用来记录在并行GC阶段，在Image Space上分配的对象对在Zygote Space和Allocation Space上分配的对象的引用的Mod Union Table，以及在Zygote Space上分配的对象对在Allocation Space上分配的对象的引用的Mod Union Table。前一个Mod Union Table使用ModUnionTableToZygoteAllocspace类来描述，后一个Mod Union Table使用ModUnionTableCardCache类来描述。关于ModUnionTableToZygoteAllocspace和ModUnionTableCardCache的关系和用途可以参考前面[ART运行时垃圾收集机制简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/42072975)一文。

8.接下来是创建Mark Stack、Allocation Stack和Live Stack，分别保存在成员变量mark_stack_、allocation_stack_和live_stack_中，它们均是使用一个ObjectStack对象来描述。Mark Stack的初始大小设置为 `default_mark_stack_size` ，即64KB。Allocation Stack和Live Stack的初始大小设置为 `max_allocation_stack_size_` 。 `max_allocation_stack_size_` 是Heap类的一个成员变量，它的值在构造函数初始化列表中设置，要么等于1KB，要么等于1MB。取决于kGCALotMode和kDesiredHeapVerification两个值。如果kGCALotMode的值等于true，那么 `max_allocation_stack_size_` 的值就等于kGcAlotInterval，即1KB。否则的话，如果kDesiredHeapVerification的值大于kNoHeapVerification的值，即ART运行时运行在堆验证模式下， `max_allocation_stack_size_` 的值就等于1KB。否则的话， `max_allocation_stack_size_` 的值就等于1MB。从最上面的代码可以看到，kGCALotMode的值等于false，因此 `max_allocation_stack_size_` 的值取决于kDesiredHeapVerification的值。

kDesiredHeapVerification是一个常量，它的定义如下所示：

```c
// How we want to sanity check the heap's correctness.  
enum HeapVerificationMode {  
kHeapVerificationNotPermitted,  // Too early in runtime start-up for heap to be verified.  
kNoHeapVerification,  // Production default.  
kVerifyAllFast,  // Sanity check all heap accesses with quick(er) tests.  
kVerifyAll  // Sanity check all heap accesses.  
};  
static constexpr HeapVerificationMode kDesiredHeapVerification = kNoHeapVerification;  
```
这个常量定义在文件rt/runtime/gc/heap.h 中。

常量kDesiredHeapVerification定义为kNoHeapVerification，即在GC过程不对堆进行验证，因此前面 `max_allocation_stack_size_` 的值会被设置为1MB，即Allocation Stack和Live Stack的初始大小被设置为1MB。

9.如果ART运行时启动时指定了-XX:IgnoreMaxFootprint选项，即Heap类的成员变量ignore_max_footprint_的值等于true，那么就需要调用Heap类的成员函数SetIdealFootprint将Heap类的成员变量max_allowed_footprint_的值设置为size_t类型的最大值，即不对堆的大小进行限制。同时也会将Heap类的成员变量concurrent_start_bytes_设置为成员变量max_allowed_footprint_的值，这意味着不会触发并行GC。

10.最后创建垃圾收集器，保存在Heap类的成员变量mark_sweep_collectors_描述的一个向量中。这些垃圾收集器一共有两组，其中一组用来执行并行GC，另外一组用来执行非并行GC。每一组都包含三个垃圾收集器，它们分别是MarkSweep、PartialMarkSweep和StickyMarkSweep。关于这三种垃圾收集器的关系和作用，可以参考前面[ART运行时垃圾收集机制简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/42072975)一文。

接下来我们就主要分析Image Space、Zygote Space、Allocation Space和Large Object Space的创建过程，其它的数据结构等到后面分析对象的分配过程和垃圾回收过程时再分析。

前面提到，Image Space是通过调用ImageSpace类的静态成员函数Create来创建的，它的实现如下所示：

```c
ImageSpace* ImageSpace::Create(const std::string& original_image_file_name) {    
if (OS::FileExists(original_image_file_name.c_str())) {    
    // If the /system file exists, it should be up-to-date, don't try to generate    
    return space::ImageSpace::Init(original_image_file_name, false);    
}    
// If the /system file didn't exist, we need to use one from the dalvik-cache.    
// If the cache file exists, try to open, but if it fails, regenerate.    
// If it does not exist, generate.    
std::string image_file_name(GetDalvikCacheFilenameOrDie(original_image_file_name));    
if (OS::FileExists(image_file_name.c_str())) {    
    space::ImageSpace* image_space = space::ImageSpace::Init(image_file_name, true);    
    if (image_space != NULL) {    
      return image_space;    
    }    
}    
CHECK(GenerateImage(image_file_name)) << "Failed to generate image: " << image_file_name;    
return space::ImageSpace::Init(image_file_name, true);    
}   
```
这个函数定义在文件art/runtime/gc/space/image_space.cc中。

ImageSpace类的静态成员函数Create我们在前面[Android运行时ART加载OAT文件的过程分析](http://blog.csdn.net/luoshengyang/article/details/39307813)一文已经分析过了， 它的执行逻辑如下所示：

判断参数original_image_file_name指定的Image文件是否存在。如果存在，就调用ImageSpace类的静态成员函数Init创建一个Image Space。否则的话，继续往下执行。

判断/data/dalvik-cache目录下是否存在一个system@framework@boot.art@classes.dex文件。如果存在，就调用ImageSpace类的静态成员函数Init创建一个Image Space。否则的话，继续往下执行。

调用ImageSpace类的静态成员函数GenerateImage在/data/dalvik-cache目录创建一个system@framework@boot.art@classes.dex，作为Image文件，并且调用ImageSpace类的静态成员函数Init创建一个Image Space。

Image文件的创建过程可以参考前面[Android运行时ART加载OAT文件的过程分析](http://blog.csdn.net/luoshengyang/article/details/39307813)一文，这里我们主要分析ImageSpace类的静态成员函数Init的实现，以便可以了解Image Space的创建过程，如下所示：

```c
ImageSpace* ImageSpace::Init(const std::string& image_file_name, bool validate_oat_file) {  
......  

UniquePtr<File> file(OS::OpenFileForReading(image_file_name.c_str()));  
......  

ImageHeader image_header;  
bool success = file->ReadFully(&image_header, sizeof(image_header));  
......  

// Note: The image header is part of the image due to mmap page alignment required of offset.  
UniquePtr<MemMap> map(MemMap::MapFileAtAddress(image_header.GetImageBegin(),  
                                         image_header.GetImageSize(),  
                                         PROT_READ | PROT_WRITE,  
                                         MAP_PRIVATE | MAP_FIXED,  
                                         file->Fd(),  
                                         0,  
                                         false));  
......  

UniquePtr<MemMap> image_map(MemMap::MapFileAtAddress(nullptr, image_header.GetImageBitmapSize(),  
                                               PROT_READ, MAP_PRIVATE,  
                                               file->Fd(), image_header.GetBitmapOffset(),  
                                               false));  
......  
size_t bitmap_index = bitmap_index_.fetch_add(1);  
std::string bitmap_name(StringPrintf("imagespace %s live-bitmap %u", image_file_name.c_str(),  
                               bitmap_index));  
UniquePtr<accounting::SpaceBitmap> bitmap(  
      accounting::SpaceBitmap::CreateFromMemMap(bitmap_name, image_map.release(),  
                                        reinterpret_cast<byte*>(map->Begin()),  
                                        map->Size()));  
......  

Runtime* runtime = Runtime::Current();  
mirror::Object* resolution_method = image_header.GetImageRoot(ImageHeader::kResolutionMethod);  
runtime->SetResolutionMethod(down_cast<mirror::ArtMethod*>(resolution_method));  

mirror::Object* callee_save_method = image_header.GetImageRoot(ImageHeader::kCalleeSaveMethod);  
runtime->SetCalleeSaveMethod(down_cast<mirror::ArtMethod*>(callee_save_method), Runtime::kSaveAll);  
callee_save_method = image_header.GetImageRoot(ImageHeader::kRefsOnlySaveMethod);  
runtime->SetCalleeSaveMethod(down_cast<mirror::ArtMethod*>(callee_save_method), Runtime::kRefsOnly);  
callee_save_method = image_header.GetImageRoot(ImageHeader::kRefsAndArgsSaveMethod);  
runtime->SetCalleeSaveMethod(down_cast<mirror::ArtMethod*>(callee_save_method), Runtime::kRefsAndArgs);  

UniquePtr<ImageSpace> space(new ImageSpace(image_file_name, map.release(), bitmap.release()));  
......  

space->oat_file_.reset(space->OpenOatFile());  
......  

return space.release();  
}  
```
这个函数定义在文件art/runtime/gc/space/image_space.cc中。

函数首先是调用OS::OpenFileForReading函数打开参数image_file_name指定的Image文件，目的是为了读取位于文件开始的一个Image文件头image_header。Image文件头使用类ImageHeader来描述，它的定义如下所示：

```c
class PACKED(4) ImageHeader {  
......  

private:  
......  

byte magic_[4];  
byte version_[4];  

// Required base address for mapping the image.  
uint32_t image_begin_;  

// Image size, not page aligned.  
uint32_t image_size_;  

// Image bitmap offset in the file.  
uint32_t image_bitmap_offset_;  

// Size of the image bitmap.  
uint32_t image_bitmap_size_;  

// Checksum of the oat file we link to for load time sanity check.  
uint32_t oat_checksum_;  

// Start address for oat file. Will be before oat_data_begin_ for .so files.  
uint32_t oat_file_begin_;  

// Required oat address expected by image Method::GetCode() pointers.  
uint32_t oat_data_begin_;  

// End of oat data address range for this image file.  
uint32_t oat_data_end_;  

// End of oat file address range. will be after oat_data_end_ for  
// .so files. Used for positioning a following alloc spaces.  
uint32_t oat_file_end_;  

// Absolute address of an Object[] of objects needed to reinitialize from an image.  
uint32_t image_roots_;  

......  
};  
```
这个类定义在文件art/runtime/image.h中。

ImageHeader类的各个成员变量的含义如下所示：

1. magic_: Image文件魔数，固定为'art\n'。

2. version_:Image文件版本号，目前设置为'005\0'。

3. image_begin_: 指定Image Space映射到内存的起始地址。

4. image_size_: Image Space的大小。

5. image_bitmap_offset_: 用来描述Image Space的对象的存活的Bitmap在Image文件的偏移位置。

6. image_bitmap_size_: 用来描述Image Space的对象的存活的Bitmap的大小。

7. oat_checksum_: 与Image文件关联的boot.art@classes.oat文件的检验值。

8. oat_file_begin_: 与Image文件关联的boot.art@classes.oat文件映射到内存的起始位置。

9. oat_data_begin_: 与Image文件关联的boot.art@classes.oat文件的OATDATA段映射到内存的起始位置。

10. oat_data_end_: 与Image文件关联的boot.art@classes.oat文件的OATDATA段映射到内存的结束位置。

11. oat_file_end_: 与Image文件关联的boot.art@classes.oat文件映射到内存的结束位置。

12. image_roots_: 一个Object对象地址数组地址，这些Object对象就是在Image Space上分配的对象。

上面第7到第11个与OAT文件相关的成员变量的具体含义可以进一步参考前面[Android运行时ART加载OAT文件的过程分析](http://blog.csdn.net/luoshengyang/article/details/39307813)一文。

理解了ImageHeader类的各个成员变量的含义之后，我们就可以知道Image文件头都包含有哪些信息了。回到ImageSpace类的静态成员函数Init中，接下来它就调用函数MemMap::MapFileAtAddress根据前面读出来的Image文件头的image_begin_和image_size_信息将参数image_file_name指定的Image文件里面的Image Space映射到内存来，并且得到一个MemMap对象map，用来描述映射的内存。

接下来再继续调用函数MemMap::MapFileAtAddress根据Image文件头的image_bitmap_size_和image_size_信息将参数image_file_name指定的Image文件里面的Bitmap也映射到内存来，并且得到一个MemMap对象image_map，用来描述映射的内存。注意，Image文件里面的Bitmap刚好是位于Space的后面，因此，在调用函数MemMap::MapFileAtAddress映射参数image_file_name指定的Image文件时，将文件偏移位置指定为image_size_对齐到页面大小的值，也就是调用image_header描述的ImageHeader对象的成员函数GetBitmapOffset得到的返回值。

有了前面的image_map之后，就可以调用SpaceBitmap类的静态成员函数CreateFromMemMap来创建一个SpaceBitmap对象bitmap了。这个SpaceBitmap对象bitmap描述的就是Image Space的Live Bitmap。

在Image文件头的成员变量image_roots_描述的对象数组中，有四个特殊的ArtMethod对象，用来描述四种特殊的ART运行时方法。ART运行时方法是一种用来描述其它ART方法的ART方法。它们具有特殊的用途，如下所示：

1. ImageHeader::kResolutionMethod: 用来描述一个还未进行解析和链接的ART方法。

2. ImageHeader::kCalleeSaveMethod: 用来描述一个由被调用者保存的r4-r11、lr和s0-s31寄存器的ART方法，即由被调用者保存非参数使用的通用寄存器以及所有的浮点数寄存器。

3. ImageHeader::kRefsOnlySaveMethod: 用来描述一个由被调用者保存的r5-r8、r10-r11和lr寄存器的ART方法，即由被调用者保存非参数使用的通用寄存器。

4. ImageHeader::kRefsAndArgsSaveMethod: 用来描述一个由被调用者保存的r1-r3、r5-r8、r10-r11和lr寄存器的ART方法，即由被调用者保存参数和非参数使用的通用寄存器。

注意，我们上面描述是针对ARM体系结构的ART方法调用约定的。其中，r0寄存器用来保存当前调用的ART方法，r1-r3寄存器用来传递前三个参数，其它参数通过栈来传递。栈顶由sp寄存器指定，r4寄存器用来保存一个在GC过程中使用到的线程挂起计数器，r5-r8和r10-r11寄存器用来分配给局部变量使用，r9寄存器用来保存当前调用线程对象，lr寄存器用来保存当前ART方法的返回地址，ip寄存器用来保存当前执行的指令地址。

上面四个特殊的ArtMethod对象从Image Space取出来之后，会通过调用Runtime类的成员函数SetResolutionMethod和SetCalleeSaveMethod保存在用来描述ART运行时的一个Runtime对象的内部，其中，第2、3和4个ArtMethod对象在ART运行时内部对应的类型分别为Runtime::kSaveAll、Runtime::kRefsOnly和Runtime::kRefsAndArgs。

这些ART运行时方法在调用还未进行解析和链接的ART方法时就会使用到。它们的具体用法可以参考前面[Android运行时ART执行类方法的过程分析](http://blog.csdn.net/luoshengyang/article/details/40289405)一文。

有了前面用来描述Image Space的MemMap对象map和描述Image Space的对象的存活的SpaceBitmap对象bitmap之后，就可以创建一个Image Space了，如下所示：

```c
ImageSpace::ImageSpace(const std::string& name, MemMap* mem_map,  
               accounting::SpaceBitmap* live_bitmap)  
    : MemMapSpace(name, mem_map, mem_map->Size(), kGcRetentionPolicyNeverCollect) {  
DCHECK(live_bitmap != NULL);  
live_bitmap_.reset(live_bitmap);  
}  
```
这个函数定义在文件art/runtime/gc/space/image_space.cc中。

从这里就可以看出：

1. Image Space的回收策略为kGcRetentionPolicyNeverCollect，即永远不会进行垃圾回收。

2. Image Space是一个在地址空间上连续的Space，因为它是通过父类MemMapSpace来管理Space的。

3. Image Space的对象的存活是通过Space Bitmap来标记的。注意，Image Space不像其它Space一样，有Live和Mark两个Bitmap。它们而是共用一个Bitmap，这是由于Image Space永远不会进行垃圾回收。

再回到ImageSpace类的静态成员函数Init中，在将前面创建的Image Space返回给调用者之前，还会调用ImageSpace类的成员函数OpenOatFile将与前面创建的Image Space关联的OAT文件也加载到内存来。OAT文件的加载过程可以参考前面[Android运行时ART加载OAT文件的过程分析](http://blog.csdn.net/luoshengyang/article/details/39307813)一文。

这样，Image Space就创建完成了。接下来我们继续分析Zygote Space的创建过程。从前面分析的Heap类构造函数可以知道，Zygote Space是一种DlMallocSpace，通过调用DlMallocSpace类的静态成员函数Create来创建，它的实现如下所示：

```c
DlMallocSpace* DlMallocSpace::Create(const std::string& name, size_t initial_size, size_t  
                             growth_limit, size_t capacity, byte* requested_begin) {  
......  
size_t starting_size = kPageSize;  
......  

// Page align growth limit and capacity which will be used to manage mmapped storage  
growth_limit = RoundUp(growth_limit, kPageSize);  
capacity = RoundUp(capacity, kPageSize);  

UniquePtr<MemMap> mem_map(MemMap::MapAnonymous(name.c_str(), requested_begin, capacity,  
                                         PROT_READ | PROT_WRITE));  
......  

void* mspace = CreateMallocSpace(mem_map->Begin(), starting_size, initial_size);  
......  

// Protect memory beyond the initial size.  
byte* end = mem_map->Begin() + starting_size;  
if (capacity - initial_size > 0) {  
    CHECK_MEMORY_CALL(mprotect, (end, capacity - initial_size, PROT_NONE), name);  
}  

// Everything is set so record in immutable structure and leave  
MemMap* mem_map_ptr = mem_map.release();  
DlMallocSpace* space;  
if (RUNNING_ON_VALGRIND > 0) {  
    space = new ValgrindDlMallocSpace(name, mem_map_ptr, mspace, mem_map_ptr->Begin(), end,  
                              growth_limit, initial_size);  
} else {  
    space = new DlMallocSpace(name, mem_map_ptr, mspace, mem_map_ptr->Begin(), end, growth_limit);  
}  
......  
return space;  
}  
```
这个函数定义在文件art/runtime/gc/space/dlmalloc_space.cc中。

DlMallocSpace类的静态成员函数Create创建的Zygote Space的底层是一块匿名共享内存，参数name就是用来指定这块匿名共享内存的名称的。此外，参数capacity指定的是要创建的匿名共享内存的大小，也就要创建的Zygote Space的大小。另外一个参数requested_begin也是和要创建的匿名共享内存相关的，用来指定在映射到进程地址空间的起始地址。剩下的两个参数initial_size和growth_limit用来指定要创建的Zygote Space的初始大小和增长上限值。

DlMallocSpace类的静态成员函数Create首先是调用函数MemMap::MapAnonymous创建一块匿名共享内存，并且得一个用来描述该匿名共享内存的MemMap对象mem_map，接下来再调用DlMallocSpace类的另外一个成员函数CreateMallocSpace来将前面创建好的匿名共享内存封装成一个mspace，以便后面可以通过C库提供的malloc和free等接口来分配和释放内存。这两步与前面[Dalvik虚拟机Java堆创建过程分析](http://blog.csdn.net/luoshengyang/article/details/41581063)一文分析的Dalvik虚拟机Zygote堆的创建过程是完全是一样的。

由于一开始并不是整个Zygote Space都是可以用来分配对象，而是只有参数initial_size指定的开始那一部分才可以分配，因此对于initial_size后面的那一部分内存，需要保护起来，方法是调用系统接口mprotect将后面的这部分内存的标志位设置为PROT_NONE。这意味着initial_size后面的那一部分内存是既不可以读，也不可以写的。

最后，判断宏RUNNING_ON_VALGRIND的值是否大于0。如果大于0，那么DlMallocSpace类的静态成员函数Create接下来创建的是一个ValgrindDlMallocSpace对象作为Zygote Space。否则的话，DlMallocSpace类的静态成员函数Create接下来创建的就是一个DlMallocSpace对象作为Zygote Space。ValgrindDlMallocSpace类是从DlMallocSpace类继承下来的，不过它会重写Alloc和Free等接口，目的是为了在为对象分配内存时，分配一块比请求大小大16个字节的内存，并且在将该内存块的开头和末尾的8个字节设置为不可访问。通过这种方法就可以检测分配出去的内存是否被应用程序非法访问。这就是我们平时经常听说的Valgrind内存非法访问检测原理。

RUNNING_ON_VALGRIND定义在文件external/valgrind/main/include/valgrind.h中，这里我们假设它的值等于0，因此接下来DlMallocSpace类的静态成员函数Create就会创建一个DlMallocSpace对象作为Zygote Space。DlMallocSpace对象的创建过程如下所示：

```c
size_t DlMallocSpace::bitmap_index_ = 0;  

DlMallocSpace::DlMallocSpace(const std::string& name, MemMap* mem_map, void* mspace, byte* begin,  
               byte* end, size_t growth_limit)  
    : MemMapSpace(name, mem_map, end - begin, kGcRetentionPolicyAlwaysCollect),  
      recent_free_pos_(0), num_bytes_allocated_(0), num_objects_allocated_(0),  
      total_bytes_allocated_(0), total_objects_allocated_(0),  
      lock_("allocation space lock", kAllocSpaceLock), mspace_(mspace),  
      growth_limit_(growth_limit) {  
......  

size_t bitmap_index = bitmap_index_++;  
......  

live_bitmap_.reset(accounting::SpaceBitmap::Create(  
      StringPrintf("allocspace %s live-bitmap %d", name.c_str(), static_cast<int>(bitmap_index)),  
      Begin(), Capacity()));  
......  

mark_bitmap_.reset(accounting::SpaceBitmap::Create(  
      StringPrintf("allocspace %s mark-bitmap %d", name.c_str(), static_cast<int>(bitmap_index)),  
      Begin(), Capacity()));  
......  

}  
```
这个函数定义在文件art/runtime/gc/space/dlmalloc_space.cc中。

从这里就可以看出：

Zygote Space的回收策略为kGcRetentionPolicyAlwaysCollect，即每次GC都会尝试对它进行垃圾回收。

Zygote Space是一个在地址空间上连续的Space，因为它是通过父类MemMapSpace来管理Space的。

Image Space的对象的存活是通过Space Bitmap来标记的，其中一个是Live Bitmap，用来标记上次GC后还存活的对象，另外一个是Mark Bitmap，用来标记当前GC后仍存活的对象。

在前面[Dalvik虚拟机Java堆创建过程分析]()一篇文章中，我们提到，Dalvik虚拟机最开始只有一个Zygote堆，不过在Zygote进程fork第一个子进程之前，就会将该Zygote堆一分为二，其中一个仍然是Zygote堆，另外一个为Active堆。ART运行时同样也是这样做的，只不过它是将Zygote Space一分为二，其中一个仍然是Zygote Space，另外一个为Allocation Space。

Zygote进程每次fork子进程的时候，都会调用Runtime类的成员函数PreZygoteFork，这样ART运行时就可以决定什么时候将Zygote Space一分为二。Runtime类的成员函数PreZygoteFork的实现如下所示：

```c
bool Runtime::PreZygoteFork() {  
heap_->PreZygoteFork();  
return true;  
}  
```
这个函数定义在文件rt/runtime/runtime.cc中。

Runtime类的成员变量heap_指向的是一个Heap对象，该Heap对象描述的便是ART运行时堆。Runtime类的成员函数PreZygoteFork的实现很简单，它通过调用Heap类的成员函数PreZygoteFork来执行拆分Zygote Space的功能。

Heap类的成员函数PreZygoteFork的实现如下所示：

```c
void Heap::PreZygoteFork() {  
static Mutex zygote_creation_lock_("zygote creation lock", kZygoteCreationLock);  
// Do this before acquiring the zygote creation lock so that we don't get lock order violations.  
CollectGarbage(false);  
Thread* self = Thread::Current();  
MutexLock mu(self, zygote_creation_lock_);  

// Try to see if we have any Zygote spaces.  
if (have_zygote_space_) {  
    return;  
}  

......  

{  
    // Flush the alloc stack.  
    WriterMutexLock mu(self, *Locks::heap_bitmap_lock_);  
    FlushAllocStack();  
}  

// Turns the current alloc space into a Zygote space and obtain the new alloc space composed  
// of the remaining available heap memory.  
space::DlMallocSpace* zygote_space = alloc_space_;  
alloc_space_ = zygote_space->CreateZygoteSpace("alloc space");  
alloc_space_->SetFootprintLimit(alloc_space_->Capacity());  

// Change the GC retention policy of the zygote space to only collect when full.  
zygote_space->SetGcRetentionPolicy(space::kGcRetentionPolicyFullCollect);  
AddContinuousSpace(alloc_space_);  
have_zygote_space_ = true;  

......  
}  
```
这个函数定义在文件art/runtime/gc/heap.cc中。

每次Heap类的成员函数PreZygoteFork被调用时，都会调用成员函数CollectGarbage执行一次垃圾收集，也就是Zygote进程每次fork子进程之前，都执行一次垃圾收集，这样就可以使得fork出来的子进程有一个紧凑的堆空间。

当Heap类的成员变量have_zygote_space_等于true的时候，就表明Zygote进程里面的Zygote Space已经被划分过了，因此，这时候Heap类的成员函数PreZygoteFork就可以直接返回了，因为接下来的所有操作都是与划分Zygote Space相关的。

在划分Zygote Space之前，先将保存在Allocation Stack里面的对象标记到对应的Space的Live Bitmap去，这是通过调用Heap类的成员函数FlushAllocStack来实现的。对于新分配的对象，ART运行时不像Dalvik虚拟机一样，马上就将它们标记对应的Space的Live Bitmap中去，而是将它们记录在Allocation Stack。这样做是为了可以执行Sticky Mark Sweep垃圾收集，后面分析ART运行时的垃圾收集过程时就可以看到这一点。

Heap类的成员函数FlushAllocStack的实现如下所示：

```c
void Heap::FlushAllocStack() {  
MarkAllocStack(alloc_space_->GetLiveBitmap(), large_object_space_->GetLiveObjects(),  
         allocation_stack_.get());  
allocation_stack_->Reset();  
}  
```
这个函数定义在文件art/runtime/gc/heap.cc中。

Heap类的成员函数FlushAllocStack首先调用另外一个成员函数MarkAllocStack将保存在Allocation Stack里面的对象标记到对应的Space的Live Bitmap去。注意，这时候Zygote进程只有三个Space，分别是Image Space、Zygote Space和Large Object Space。其中，Image Space是不能用来分配新对象的，因此，保存在Allocation Stack里面的对象要么是在Zygote Space上分配的，要么是Large Object Space上分配的。这时候就只需要将Zygote Space和Large Object Space的Live Bitmap取出来，连同Allocation Stack一起，传递给Heap类的成员函数MarkAllocStack，记它进行相应的标记。

一旦Heap类的成员函数MarkAllocStack将保存在Allocation Stack里面的对象都标记到对应的Space的Live Bitmap去之后，Heap类的成员函数FlushAllocStack就可以将Allocation Stack清空了。

Heap类的成员函数MarkAllocStack的实现如下所示：

```c
void Heap::MarkAllocStack(accounting::SpaceBitmap* bitmap, accounting::SpaceSetMap* large_objects,  
                  accounting::ObjectStack* stack) {  
mirror::Object** limit = stack->End();  
for (mirror::Object** it = stack->Begin(); it != limit; ++it) {  
    const mirror::Object* obj = *it;  
    DCHECK(obj != NULL);  
    if (LIKELY(bitmap->HasAddress(obj))) {  
      bitmap->Set(obj);  
    } else {  
      large_objects->Set(obj);  
    }  
}  
}  
```
这个函数定义在文件art/runtime/gc/heap.cc中。

Heap类的成员函数MarkAllocStack就是遍历Allocation Stack里面的每一个对象，并且判断它是属于Zygote Space还是Large Object Space。如果是属于Zygote Space，就将Zygote Space的Live Bitmap对应位设置为1。如果是属于Large Object Space，就将被遍历对象添加到Large Object Space的Live Space Set Map去，起到的也是将Large Object Space的Live Bitmap对应设置为1的效果。

回到Heap类的成员函数PreZygoteFork中，将保存在Allocation Stack里面的对象标记到对应的Space的Live Bitmap去之后，接下来就开始划分Zygote Space了。划分之前，我们首先明确，Heap类的成员变量alloc_space_指向的实际上就是Zygote Space，但是接下来我们要将它指向Allocation Space，因此就先把Heap类的成员变量alloc_space_保存在本地变量zygote_space中。从Zygote Space划分出一个Allocation Space的操作是通过调用DlMallocSpace类的成员函数CreateZygoteSpace来实现。DlMallocSpace类的成员函数CreateZygoteSpace返回来的是新划分出来的Allocation Space，因此这时候就将它保存在Heap类的成员变量alloc_space_中了。

最后，Heap类的成员函数PreZygoteFork还要做一些收尾工作，主要包括：

1. 设置新划分出来的Allocation Space底层使用的mspace的大小值，即将该mspace的大小设置为Allocation Space的大小，这是通过调用DlMallocSpace类的成员函数SetFootprintLimit实现的。

2. 将Zygote Space的回收策略修改为kGcRetentionPolicyFullCollect，这样只有执行类型为kGcRetentionPolicyFullCollect的GC时，才会对Zygote Space的垃圾进行回收。

3. 调用Heap类的成员函数AddContinuousSpace将新划分出来的Allocation Space添加到ART运行时内部的Continuous Space列表去。

4. 将Heap类的成员变量have_zygote_space_，表示Zygote Space已经被划分过了，以后不用再划分了。

接下来，我们就继续分析DlMallocSpace类的成员函数CreateZygoteSpace，以便可以了解Zygote Space是如何被一分为二的，它的实现如下所示：

```c
DlMallocSpace* DlMallocSpace::CreateZygoteSpace(const char* alloc_space_name) {  
end_ = reinterpret_cast<byte*>(RoundUp(reinterpret_cast<uintptr_t>(end_), kPageSize));  
......  

size_t size = RoundUp(Size(), kPageSize);  
// Trim the heap so that we minimize the size of the Zygote space.  
Trim();  
// Trim our mem-map to free unused pages.  
GetMemMap()->UnMapAtEnd(end_);  
// TODO: Not hardcode these in?  
const size_t starting_size = kPageSize;  
const size_t initial_size = 2 * MB;  
// Remaining size is for the new alloc space.  
const size_t growth_limit = growth_limit_ - size;  
const size_t capacity = Capacity() - size;  
......  
SetGrowthLimit(RoundUp(size, kPageSize));  
SetFootprintLimit(RoundUp(size, kPageSize));  
......  
UniquePtr<MemMap> mem_map(MemMap::MapAnonymous(alloc_space_name, End(), capacity, PROT_READ | PROT_WRITE));  
void* mspace = CreateMallocSpace(end_, starting_size, initial_size);  
// Protect memory beyond the initial size.  
byte* end = mem_map->Begin() + starting_size;  
if (capacity - initial_size > 0) {  
    CHECK_MEMORY_CALL(mprotect, (end, capacity - initial_size, PROT_NONE), alloc_space_name);  
}  
DlMallocSpace* alloc_space =  
      new DlMallocSpace(alloc_space_name, mem_map.release(), mspace, end_, end, growth_limit);  
live_bitmap_->SetHeapLimit(reinterpret_cast<uintptr_t>(End()));  
......  
mark_bitmap_->SetHeapLimit(reinterpret_cast<uintptr_t>(End()));  
......  
return alloc_space;  
}  
```
这个函数定义在文件runtime/gc/space/dlmalloc_space.cc中。

DlMallocSpace类的成员变量end_是从父类Continuous继承下来的，描述的是当前Zygote Space使用的最大值，而调用DlMallocSpace类的成员函数Size获得的是当前Zygote Space使用的大小size。

在前面[Dalvik虚拟机Java堆创建过程分析](http://blog.csdn.net/luoshengyang/article/details/41581063)一篇文章中，我们提到，Dalvik虚拟机在划分Zygote堆之前，会对Zygote堆底层使用的内存进行裁剪，主要就是将末尾没有使用到的内存占用的虚拟地址空间和物理页面归还给[操作系统](http://lib.csdn.net/base/operatingsystem)，以及将中间没有使用到的内存占用的物理页面归还给操作系统。这里在划分Zygote Space之前，也会执行相同的操作，这些操作通过调用DlMallocSpace类的成员函数Trim和MemMap类的成员函数UnMapAtEnd来实现的。

对Zygote Space进行裁剪之后，接下来就计算新的Allocation Space的大小，实际上就是等于原来Zygote Space的总大小减去已经已经使用的大小。原来Zygote Space的总大小可以调用DlMallocSpace类的成员函数Size获得，而原来Zygote Space已经使用的大小前面已经得到保存在变量size，因此我们很容易就可以计算得到新的Allocation Space的大小为capacity。

前面计算得到的capacity是新的Allocation Space的硬上限，我们还需要给它设置一个增长上限值，即一个软上限值。DlMallocSpace类的成员变量growth_limit_描述的是原来Zygote Space的增长上限值，用它的值减去原来Zygote Space已经使用的大小size，就可以得到新的Allocation Space的增长上限值。

我们还需要两个参数。 一个参数是新的Allocation Space的初始大小initial_size，这里设置为2MB。另一个参数是新的Allocation Space底层使用的mspace的初始大小starting_size，这里设置为kPageSize，即一页大小。

有了上述这些参数之后，就可以调用MemMap类的静态成员函数MapAnonymous来创建一块大小等于capacity的匿名共享内存，并且调用DlMallocSpace类的成员函数CreateMallocSpace将该匿名共享内存封装成一个mspace。注意，新创建的匿名共享内存映射到进程地址空间时，被指定为end_，即紧跟在Zygote Space的后面。

就像前面创建Zygote Space一样，对于新创建出来的Allocation Space的末尾部分内存，即不在初始大小的那一部分内存，需要调用系统接口mprotect将其标记为不可访问。

最后，DlMallocSpace类的成员函数CreateZygoteSpace就可以根据前面获得的信息创建一个DlMallocSpace对象alloc_space来描述新的Allocation Space了，并且该alloc_space返回给调用者。不过，在返回之前，DlMallocSpace类的成员函数CreateZygoteSpace还需要修改重新得到的Zygote Space使用的Live Bitmap和Mark Bitmap的上限值，即它们可以处理的范围，就等于原来Zygote Space已经使用的大小。

这样，Zygote Space和Allocation Space创建过程也分析完成了。我们再继续分析Large Object Space的创建过程。前面提到，Large Object Space是通过LargeObjectMapSpace类的静态成员函数Create创建的，它的实现如下所示：

```c
LargeObjectMapSpace* LargeObjectMapSpace::Create(const std::string& name) {  
return new LargeObjectMapSpace(name);  
}  
```
这个函数定义在文件art/runtime/gc/space/large_object_space.cc中。

LargeObjectMapSpace类的静态成员函数Create创建一个LargeObjectMapSpace对象来描述Large Object Space。LargeObjectMapSpace对象的创建过程如下所示：

```c
LargeObjectSpace::LargeObjectSpace(const std::string& name)  
    : DiscontinuousSpace(name, kGcRetentionPolicyAlwaysCollect),  
      num_bytes_allocated_(0), num_objects_allocated_(0), total_bytes_allocated_(0),  
      total_objects_allocated_(0) {  
}  

......  

LargeObjectMapSpace::LargeObjectMapSpace(const std::string& name)  
    : LargeObjectSpace(name),  
      lock_("large object map space lock", kAllocSpaceLock) {}  
```
这两个函数定义在文件art/runtime/gc/space/large_object_space.cc中。 

从这里就可以看出：

LargeObjectMapSpace继承于LargeObjectSpace。

Large Object Space的回收策略为kGcRetentionPolicyAlwaysCollect，即每次GC都会尝试对它进行垃圾回收。

Large Object Space是一个在地址空间上不连续的Space，因为它是通过父类DiscontinuousSpace来管理Space的。

DiscontinuousSpace类有两个类型为SpaceSetMap的成员变量live_objects_和mark_objects_，如下所示：

```c
class DiscontinuousSpace : public Space {  
public:  
......  

protected:  
......  

UniquePtr<accounting::SpaceSetMap> live_objects_;  
UniquePtr<accounting::SpaceSetMap> mark_objects_;  

private:  
......  
};  
```
这个类定义在文件art/runtime/gc/space/space.h中。

这两个成员变量分别用来描述Large Object Space的Live Bitmap和Mark Bitmap。注意，这里的Bitmap并不是一个真的Bitmap，而是一个Map。这是由于Large Object Space是一个在地址空间上不连续的Space，因此不能用一块连续的Bitmap来描述它的对象的存活性。

这样，Large Object Space的创建过程我们也分析完成了。最后，我们还需要分析Heap类的两个成员函数AddContinuousSpace和AddDiscontinuousSpace。前者用来在ART运行时内部添加一个Continuous Space，即Zygote Space和Allocation Space，而者用来在ART运行时内部添加一个Discontinuous Space，即Large Object Space。

Heap类的成员函数AddContinuousSpace的实现如下所示：

```c
void Heap::AddContinuousSpace(space::ContinuousSpace* space) {  
WriterMutexLock mu(Thread::Current(), *Locks::heap_bitmap_lock_);  
DCHECK(space != NULL);  
DCHECK(space->GetLiveBitmap() != NULL);  
live_bitmap_->AddContinuousSpaceBitmap(space->GetLiveBitmap());  
DCHECK(space->GetMarkBitmap() != NULL);  
mark_bitmap_->AddContinuousSpaceBitmap(space->GetMarkBitmap());  
continuous_spaces_.push_back(space);  
if (space->IsDlMallocSpace() && !space->IsLargeObjectSpace()) {  
    alloc_space_ = space->AsDlMallocSpace();  
}  

// Ensure that spaces remain sorted in increasing order of start address (required for CMS finger)  
std::sort(continuous_spaces_.begin(), continuous_spaces_.end(),  
    [](const space::ContinuousSpace* a, const space::ContinuousSpace* b) {  
      return a->Begin() < b->Begin();  
    });  

// Ensure that ImageSpaces < ZygoteSpaces < AllocSpaces so that we can do address based checks to  
// avoid redundant marking.  
bool seen_zygote = false, seen_alloc = false;  
for (const auto& space : continuous_spaces_) {  
    if (space->IsImageSpace()) {  
      DCHECK(!seen_zygote);  
      DCHECK(!seen_alloc);  
    } else if (space->IsZygoteSpace()) {  
      DCHECK(!seen_alloc);  
      seen_zygote = true;  
    } else if (space->IsDlMallocSpace()) {  
      seen_alloc = true;  
    }  
}  
}  
```
这个函数定义在文件art/runtime/gc/heap.cc中。

Heap类有两个成员变量live_bitmap_和mark_bitmap_，它们的类型为HeapBitmap。但它们其实并不是一个真正的Bitmap，而是一个Bitmap容器，分别用来保存ART运行时内部所有Continuous Space和Discontinuous Space的Live Bitmap和Mark Bitmap。其中，Continuous Space的Bitmap是一个SpaceBitmap，而Discontinuous Space的Bitmap是一个SpaceSetMap。

Heap类的成员函数AddContinuousSpace首先是将要添加的Continuous Space的Live Bitmap和Mark Bitmap分别添加到成员变量live_bitmap_和mark_bitmap_描述的HeapBitmap去，接着再将要添加的Continuous Space添加到成员变量continuous_spaces_描述的一个向量中去，最后还会对保存在成员变量continuous_spaces_描述的向量的Continuous Space进行排序，以确保它们是按照地址从小到大的顺序的排序的，即按照Image Space  <  Zygote Space  <  Allocation Space的顺序进行排序，就如图1所示。

Heap类的成员函数AddDiscontinuousSpace的实现如下所示：

```c
void Heap::AddDiscontinuousSpace(space::DiscontinuousSpace* space) {  
WriterMutexLock mu(Thread::Current(), *Locks::heap_bitmap_lock_);  
DCHECK(space != NULL);  
DCHECK(space->GetLiveObjects() != NULL);  
live_bitmap_->AddDiscontinuousObjectSet(space->GetLiveObjects());  
DCHECK(space->GetMarkObjects() != NULL);  
mark_bitmap_->AddDiscontinuousObjectSet(space->GetMarkObjects());  
discontinuous_spaces_.push_back(space);  
}  
```
这个函数定义在文件art/runtime/gc/heap.cc中。

Heap类的成员函数AddContinuousSpace首先是将要添加的Discontinuous Space的Live Bitmap和Mark Bitmap分别添加到成员变量live_bitmap_和mark_bitmap_描述的HeapBitmap去，接着再将要添加的Discontinuous Space添加到成员变量discontinuous_spaces_描述的一个向量中去。由于Discontinuous Space在地址空间上不能比较大小，因此它们就不像Continuous Space需要排序。

至此，我们就分析完成ART运行时堆的创建过程了，从中我们就可以对ART运行时堆的结构有一个基本的认识。接下来，我们就可以在这个基础上，分析ART运行时在堆上分配对象的过程了，敬请关注！更多信息也可以关注老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)。