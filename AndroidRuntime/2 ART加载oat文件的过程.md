在前面一文中，我们介绍了[Android](http://lib.csdn.net/base/android)运行时ART，它的核心是OAT文件。OAT文件是一种[android](http://lib.csdn.net/base/android)私有ELF文件格式，它不仅包含有从DEX文件翻译而来的本地机器指令，还包含有原来的DEX文件内容。这使得我们无需重新编译原有的APK就可以让它正常地在ART里面运行，也就是我们不需要改变原来的APK编程接口。本文我们通过OAT文件的加载过程分析OAT文件的结构，为后面分析ART的工作原理打基础。

老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)，欢迎关注！

《Android系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

OAT文件的结构如图1所示：

![img](http://img.blog.csdn.net/20140914023348415)

图1 OAT文件结构

由于OAT文件本质上是一个ELF文件，因此在最外层它具有一般ELF文件的结构，例如它有标准的ELF文件头以及通过段（Section）来描述文件内容。关于ELF文件的更多知识，可以参考维基百科：[http://en.wikipedia.org/wiki/Executable_and_Linkable_Format](http://en.wikipedia.org/wiki/Executable_and_Linkable_Format)。

作为Android私有的一种ELF文件，OAT文件包含有两个特殊的段oatdata和oatexec，前者包含有用来生成本地机器指令的dex文件内容，后者包含有生成的本地机器指令，它们之间的关系通过储存在oatdata段前面的oat头部描述。此外，在OAT文件的dynamic段，导出了三个符号oatdata、oatexec和oatlastword，它们的值就是用来界定oatdata段和oatexec段的起止位置的。其中，[oatdata, oatexec - 1]描述的是oatdata段的起止位置，而[oatexec, oatlastword + 3]描述的是oatexec的起止位置。要完全理解OAT的文件格式，除了要理解本文即将要分析的OAT加载过程之外，还需要掌握接下来文章分析的类和方法查找过程。

在分析OAT文件的加载过程之前，我们需要简单介绍一下OAT是如何产生的。如前面[Android ART运行时无缝替换Dalvik虚拟机的过程分析](http://blog.csdn.net/luoshengyang/article/details/18006645)一文所示，APK在安装的过程中，会通过dex2oat工具生成一个OAT文件：

```c
static void run_dex2oat(int zip_fd, int oat_fd, const char* input_file_name,    
    const char* output_file_name, const char* dexopt_flags)    
{    
    static const char* DEX2OAT_BIN = "/system/bin/dex2oat";    
    static const int MAX_INT_LEN = 12;      // '-'+10dig+'\0' -OR- 0x+8dig    
    char zip_fd_arg[strlen("--zip-fd=") + MAX_INT_LEN];    
    char zip_location_arg[strlen("--zip-location=") + PKG_PATH_MAX];    
    char oat_fd_arg[strlen("--oat-fd=") + MAX_INT_LEN];    
    char oat_location_arg[strlen("--oat-name=") + PKG_PATH_MAX];    
    
    sprintf(zip_fd_arg, "--zip-fd=%d", zip_fd);    
    sprintf(zip_location_arg, "--zip-location=%s", input_file_name);    
    sprintf(oat_fd_arg, "--oat-fd=%d", oat_fd);    
    sprintf(oat_location_arg, "--oat-location=%s", output_file_name);    
    
    ALOGV("Running %s in=%s out=%s\n", DEX2OAT_BIN, input_file_name, output_file_name);    
    execl(DEX2OAT_BIN, DEX2OAT_BIN,    
  zip_fd_arg, zip_location_arg,    
  oat_fd_arg, oat_location_arg,    
  (char*) NULL);    
    ALOGE("execl(%s) failed: %s\n", DEX2OAT_BIN, strerror(errno));    
}    
```
这个函数定义在文件frameworks/native/cmds/installd/commands.c中。

其中，参数zip_fd和oat_fd都是打开文件描述符，指向的分别是正在安装的APK文件和要生成的OAT文件。OAT文件的生成过程主要就是涉及到将包含在APK里面的classes.dex文件的DEX字节码翻译成本地机器指令。这相当于是编写一个输入文件为DEX、输出文件为OAT的编译器。这个编译器是基于LLVM编译框架开发的。编译器的工作原理比较高大上，所幸的是它不会影响到我们接下来的分析，因此我们就略过DEX字节码翻译成本地机器指令的过程，假设它很愉快地完成了。

APK安装过程中生成的OAT文件的输入只有一个DEX文件，也就是来自于打包在要安装的APK文件里面的classes.dex文件。实际上，一个OAT文件是可以由若干个DEX生成的。这意味着在生成的OAT文件的oatdata段中，包含有多个DEX文件。那么，在什么情况下，会生成包含多个DEX文件的OAT文件呢？

从前面[Android ART运行时无缝替换Dalvik虚拟机的过程分析](http://blog.csdn.net/luoshengyang/article/details/18006645)一文可以知道，当我们选择了ART运行时时，Zygote进程在启动的过程中，会调用libart.so里面的函数JNI_CreateJavaVM来创建一个ART虚拟机。函数JNI_CreateJavaVM的实现如下所示：

```c
extern "C" jint JNI_CreateJavaVM(JavaVM** p_vm, JNIEnv** p_env, void* vm_args) {  
const JavaVMInitArgs* args = static_cast<JavaVMInitArgs*>(vm_args);  
if (IsBadJniVersion(args->version)) {  
    LOG(ERROR) << "Bad JNI version passed to CreateJavaVM: " << args->version;  
    return JNI_EVERSION;  
}  
Runtime::Options options;  
for (int i = 0; i < args->nOptions; ++i) {  
    JavaVMOption* option = &args->options[i];  
    options.push_back(std::make_pair(std::string(option->optionString), option->extraInfo));  
}  
bool ignore_unrecognized = args->ignoreUnrecognized;  
if (!Runtime::Create(options, ignore_unrecognized)) {  
    return JNI_ERR;  
}  
Runtime* runtime = Runtime::Current();  
bool started = runtime->Start();  
if (!started) {  
    delete Thread::Current()->GetJniEnv();  
    delete runtime->GetJavaVM();  
    LOG(WARNING) << "CreateJavaVM failed";  
    return JNI_ERR;  
}  
*p_env = Thread::Current()->GetJniEnv();  
*p_vm = runtime->GetJavaVM();  
return JNI_OK;  
}  
```
这个函数定义在文件art/runtime/jni_internal.cc中。

参数vm_args用作ART虚拟机的启动参数，它被转换为一个JavaVMInitArgs对象后，再按照Key-Value的组织形式保存一个Options向量中，并且以该向量作为参数传递给Runtime类的静态成员函数Create。

Runtime类的静态成员函数Create负责在进程中创建一个ART虚拟机。创建成功后，就调用Runtime类的另外一个静态成员函数Start启动该ART虚拟机。注意，这个创建ART虚拟的动作只会在Zygote进程中执行，SystemServer系统进程以及Android应用程序进程的ART虚拟机都是直接从Zygote进程fork出来共享的。这与Dalvik虚拟机的创建方式是完全一样的。

接下来我们就重点分析Runtime类的静态成员函数Create，它的实现如下所示：

```c
bool Runtime::Create(const Options& options, bool ignore_unrecognized) {  
// TODO: acquire a static mutex on Runtime to avoid racing.  
if (Runtime::instance_ != NULL) {  
    return false;  
}  
InitLogging(NULL);  // Calls Locks::Init() as a side effect.  
instance_ = new Runtime;  
if (!instance_->Init(options, ignore_unrecognized)) {  
    delete instance_;  
    instance_ = NULL;  
    return false;  
}  
return true;  
}  
```
这个函数定义在文件art/runtime/runtime.cc中。

instance_是Runtime类的静态成员变量，它指向进程中的一个Runtime单例。这个Runtime单例描述的就是当前进程的ART虚拟机实例。

函数首先判断当前进程是否已经创建有一个ART虚拟机实例了。如果有的话，函数就立即返回。否则的话，就创建一个ART虚拟机实例，并且保存在Runtime类的静态成员变量instance_中，最后调用Runtime类的成员函数Init对该新创建的ART虚拟机进行初始化。

Runtime类的成员函数Init的实现如下所示：

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

java_vm_ = new JavaVMExt(this, options.get());  
......  

Thread* self = Thread::Attach("main", false, NULL, false);  
......  

if (GetHeap()->GetContinuousSpaces()[0]->IsImageSpace()) {  
    class_linker_ = ClassLinker::CreateFromImage(intern_table_);  
} else {  
    ......  
    class_linker_ = ClassLinker::CreateFromCompiler(*options->boot_class_path_, intern_table_);  
}  
......  

return true;  
}  
```
这个函数定义在文件art/runtime/runtime.cc中。

Runtime类的成员函数Init首先调用ParsedOptions类的静态成员函数Create对ART虚拟机的启动参数raw_options进行解析。解析后得到的参数保存在一个ParsedOptions对象中，接下来就根据这些参数一个ART虚拟机堆。ART虚拟机堆使用一个Heap对象来描述。

创建好ART虚拟机堆后，Runtime类的成员函数Init接着又创建了一个JavaVMExt实例。这个JavaVMExt实例最终是要返回给调用者的，使得调用者可以通过该JavaVMExt实例来和ART虚拟机交互。再接下来，Runtime类的成员函数Init通过Thread类的成员函数Attach将当前线程作为ART虚拟机的主线程，使得当前线程可以调用ART虚拟机提供的JNI接口。

Runtime类的成员函数GetHeap返回的便是当前ART虚拟机的堆，也就是前面创建的ART虚拟机堆。通过调用Heap类的成员函数GetContinuousSpaces可以获得堆里面的连续空间列表。如果这个列表的第一个连续空间是一个Image空间，那么就调用ClassLinker类的静态成员函数CreateFromImage来创建一个ClassLinker对象。否则的话，上述ClassLinker对象就要通过ClassLinker类的另外一个静态成员函数CreateFromCompiler来创建。创建出来的ClassLinker对象是后面ART虚拟机加载加载[Java](http://lib.csdn.net/base/java)类时要用到的。

后面我们分析ART虚拟机的垃圾收集机制时会看到，ART虚拟机的堆包含有三个连续空间和一个不连续空间。三个连续空间分别用来分配不同的对象。当第一个连续空间不是Image空间时，就表明当前进程不是Zygote进程，而是安装应用程序时启动的一个dex2oat进程。安装应用程序时启动的dex2oat进程也会在内部创建一个ART虚拟机，不过这个ART虚拟机是用来将DEX字节码编译成本地机器指令的，而Zygote进程创建的ART虚拟机是用来运行应用程序的。

接下来我们主要分析ParsedOptions类的静态成员函数Create和ART虚拟机堆Heap的构造函数，以便可以了解ART虚拟机的启动参数解析过程和ART虚拟机的堆创建过程。

ParsedOptions类的静态成员函数Create的实现如下所示：

```c
Runtime::ParsedOptions* Runtime::ParsedOptions::Create(const Options& options, bool ignore_unrecognized) {  
UniquePtr<ParsedOptions> parsed(new ParsedOptions());  
const char* boot_class_path_string = getenv("BOOTCLASSPATH");  
if (boot_class_path_string != NULL) {  
    parsed->boot_class_path_string_ = boot_class_path_string;  
}  
......  

parsed->is_compiler_ = false;  
......  

for (size_t i = 0; i < options.size(); ++i) {  
    const std::string option(options[i].first);  
    ......  

    if (StartsWith(option, "-Xbootclasspath:")) {  
      parsed->boot_class_path_string_ = option.substr(strlen("-Xbootclasspath:")).data();  
    } else if (option == "bootclasspath") {  
      parsed->boot_class_path_  
  = reinterpret_cast<const std::vector<const DexFile*>*>(options[i].second);  
    } else if (StartsWith(option, "-Ximage:")) {  
      parsed->image_ = option.substr(strlen("-Ximage:")).data();  
    } else if (......) {      
      ......  
    } else if (option == "compiler") {  
      parsed->is_compiler_ = true;  
    } else {  
      ......  
    }  
}  
    
......  

if (!parsed->is_compiler_ && parsed->image_.empty()) {  
    parsed->image_ += GetAndroidRoot();  
    parsed->image_ += "/framework/boot.art";  
}  

......  

return parsed.release();  
}  
```
这个函数定义在文件art/runtime/runtime.cc中。

ART虚拟机的启动参数比较多，这里我们只关注两个：-Xbootclasspath、-Ximage和compiler。

参数-Xbootclasspath用来指定启动类路径。如果没有指定启动类路径，那么默认的启动类路径就通过环境变量BOOTCLASSPATH来获得。

参数-Ximage用来指定ART虚拟机所使用的Image文件。这个Image是用来启动ART虚拟机的。

参数compiler用来指定当前要创建的ART虚拟机是用来将DEX字节码编译成本地机器指令的。

如果没有指定Image文件，并且当前创建的ART虚拟机又不是用来编译DEX字节码的，那么就将该Image文件指定为设备上的/system/framework/boot.art文件。我们知道，system分区的文件都是在制作ROM时打包进去的。这样上述代码的逻辑就是说，如果没有指定Image文件，那么将system分区预先准备好的framework/boot.art文件作为Image文件来启动ART虚拟机。不过，/system/framework/boot.art文件可能是不存在的。在这种情况下，就需要生成一个新的Image文件。这个Image文件就是一个包含了多个DEX文件的OAT文件。接下来通过分析ART虚拟机堆的创建过程就会清楚地看到这一点。

Heap类的构造函数的实现如下所示：

```c
Heap::Heap(size_t initial_size, size_t growth_limit, size_t min_free, size_t max_free,  
   double target_utilization, size_t capacity, const std::string& original_image_file_name,  
   bool concurrent_gc, size_t parallel_gc_threads, size_t conc_gc_threads,  
   bool low_memory_mode, size_t long_pause_log_threshold, size_t long_gc_log_threshold,  
   bool ignore_max_footprint)  
    : ...... {  
......  

std::string image_file_name(original_image_file_name);  
if (!image_file_name.empty()) {  
    space::ImageSpace* image_space = space::ImageSpace::Create(image_file_name);  
    ......  
    AddContinuousSpace(image_space);  
    ......  
}  

......  
}  
```
这个函数定义在文件art/runtime/gc/heap.cc中。

ART虚拟机堆的详细创建过程我们在后面分析ART虚拟机的垃圾收集机制时再分析，这里只关注与Image文件相关的逻辑。

参数original_image_file_name描述的就是前面提到的Image文件的路径。如果它的值不等于空的话，那么就以它为参数，调用ImageSpace类的静态成员函数Create创建一个Image空间，并且调用Heap类的成员函数AddContinuousSpace将该Image空间作为本进程的ART虚拟机堆的第一个连续空间。

接下来我们继续分析ImageSpace类的静态成员函数Create，它的实现如下所示：

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

ImageSpace类的静态成员函数Create首先是检查参数original_image_file_name指定的Image文件是否存在。如果存在的话，就以它为参数，调用ImageSpace类的另外一个静态成员函数Init来创建一个Image空间。否则的话，再调用函数GetDalvikCacheFilenameOrDie根据参数original_image_file_name构造另外一个在/data/dalvik-cache目录下的文件路径，然后再检查这个文件是否存在。如果存在的话，就同样是以它为参数，调用ImageSpace类的静态成员函数Init来创建一个Image空间。否则的话，就要调用ImageSpace类的另外一个静态成员函数GenerateImage来生成一个新的Image文件，接着再调用ImageSpace类的静态成员函数Init来创建一个Image空间了。

我们假设参数original_image_file_name的值等于“/system/framework/boot.art”，那么ImageSpace类的静态成员函数Create的执行逻辑实际上就是：

检查文件/system/framework/boot.art是否存在。如果存在，那么就以它为参数，创建一个Image空间。否则的话，执行下一步。

检查文件/data/dalvik-cache/system@framework@boot.art@classes.dex是否存在。如果存在，那么就以它为参数，创建一个Image空间。否则的话，执行下一步。

调用ImageSpace类的静态成员函数GenerateImage在/data/dalvik-cache目录下生成一个system@framework@boot.art@classes.dex，然后再以该文件为参数，创建一个Image空间。

接下来我们再来看看ImageSpace类的静态成员函数GenerateImage的实现，如下所示：

```c
static bool GenerateImage(const std::string& image_file_name) {  
const std::string boot_class_path_string(Runtime::Current()->GetBootClassPathString());  
std::vector<std::string> boot_class_path;  
Split(boot_class_path_string, ':', boot_class_path);  
......  

std::vector<std::string> arg_vector;  

std::string dex2oat(GetAndroidRoot());  
dex2oat += (kIsDebugBuild ? "/bin/dex2oatd" : "/bin/dex2oat");  
arg_vector.push_back(dex2oat);  

std::string image_option_string("--image=");  
image_option_string += image_file_name;  
arg_vector.push_back(image_option_string);  
......  

for (size_t i = 0; i < boot_class_path.size(); i++) {  
    arg_vector.push_back(std::string("--dex-file=") + boot_class_path[i]);  
}  

std::string oat_file_option_string("--oat-file=");  
oat_file_option_string += image_file_name;  
oat_file_option_string.erase(oat_file_option_string.size() - 3);  
oat_file_option_string += "oat";  
arg_vector.push_back(oat_file_option_string);  
......  

if (kIsTargetBuild) {  
    arg_vector.push_back("--image-classes-zip=/system/framework/framework.jar");  
    arg_vector.push_back("--image-classes=preloaded-classes");  
}   
......  

// Convert the args to char pointers.  
std::vector<char*> char_args;  
for (std::vector<std::string>::iterator it = arg_vector.begin(); it != arg_vector.end();  
      ++it) {  
    char_args.push_back(const_cast<char*>(it->c_str()));  
}  
char_args.push_back(NULL);  

// fork and exec dex2oat  
pid_t pid = fork();  
if (pid == 0) {  
    ......  

    execv(dex2oat.c_str(), &char_args[0]);  

    ......  
    return false;  
} else {  
    ......  

    // wait for dex2oat to finish  
    int status;  
    pid_t got_pid = TEMP_FAILURE_RETRY(waitpid(pid, &status, 0));  
    .......  
}  
return true;  
}  
```
这个函数定义在文件art/runtime/gc/space/image_space.cc中。

ImageSpace类的静态成员函数GenerateImage实际上就调用dex2oat工具在/data/dalvik-cache目录下生成两个文件：system@framework@boot.art@classes.dex和system@framework@boot.art@classes.oat。

system@framework@boot.art@classes.dex是一个Image文件，通过--image选项传递给dex2oat工具，里面包含了一些需要在Zygote进程启动时预加载的类。这些需要预加载的类由/system/framework/framework.jar文件里面的preloaded-classes文件指定。

system@framework@boot.art@classes.oat是一个OAT文件，通过--oat-file选项传递给dex2oat工具，它是由系统启动路径中指定的jar文件生成的。每一个jar文件都通过一个--dex-file选项传递给dex2oat工具。这样dex2oat工具就可以将它们所包含的classes.dex文件里面的DEX字节码翻译成本地机器指令。

这样，我们就得到了一个包含有多个DEX文件的OAT文件system@framework@boot.art@classes.oat。

通过上面的分析，我们就清楚地看到了ART运行时所需要的OAT文件是如何产生的了。其中，由系统启动类路径指定的DEX文件生成的OAT文件称为类型为BOOT的OAT文件，即boot.art文件。有了这个背景知识之后，接下来我们就继续分析ART运行时是如何加载OAT文件的。

ART运行时提供了一个OatFile类，通过调用它的静态成员函数Open可以在本进程中加载OAT文件，它的实现如下所示：

```c
OatFile* OatFile::Open(const std::string& filename,  
               const std::string& location,  
               byte* requested_base,  
               bool executable) {  
CHECK(!filename.empty()) << location;  
CheckLocation(filename);  
\#ifdef ART_USE_PORTABLE_COMPILER  
// If we are using PORTABLE, use dlopen to deal with relocations.  
//  
// We use our own ELF loader for Quick to deal with legacy apps that  
// open a generated dex file by name, remove the file, then open  
// another generated dex file with the same name. http://b/10614658  
if (executable) {  
    return OpenDlopen(filename, location, requested_base);  
}  
\#endif  
// If we aren't trying to execute, we just use our own ElfFile loader for a couple reasons:  
//  
// On target, dlopen may fail when compiling due to selinux restrictions on installd.  
//  
// On host, dlopen is expected to fail when cross compiling, so fall back to OpenElfFile.  
// This won't work for portable runtime execution because it doesn't process relocations.  
UniquePtr<File> file(OS::OpenFileForReading(filename.c_str()));  
if (file.get() == NULL) {  
    return NULL;  
}  
return OpenElfFile(file.get(), location, requested_base, false, executable);  
}  
```
这个函数定义在文件art/runtime/oat_file.cc中。

参数filename和location实际上是一样的，指向要加载的OAT文件。参数requested_base是一个可选参数，用来描述要加载的OAT文件里面的oatdata段要加载在的位置。参数executable表示要加载的OAT是不是应用程序的主执行文件。一般来说，一个应用程序只有一个classes.dex文件， 这个classes.dex文件经过编译后，就得到一个OAT主执行文件。不过，应用程序也可以在运行时动态加载DEX文件。这些动态加载的DEX文件在加载的时候同样会被翻译成OAT再运行，它们相应打包在应用程序的classes.dex文件来说，就不属于主执行文件了。

OatFile类的静态成员函数Open的实现虽然只有寥寥几行代码，但是要理解它还得先理解宏ART_USE_PORTABLE_COMPILER的的作用。在前面[Android运行时ART简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/39256813)一文中提到，ART运行时利用LLVM编译框架来将DEX字节码翻译成本地机器指令，其中要通过一个称为Backend的模块来生成本地机器指令。这些生成的机器指令就保存在ELF文件格式的OAT文件的oatexec段中。

ART运行时会为每一个类方法都生成一系列的本地机器指令。这些本地机器指令不是孤立存在的，因为它们可能需要其它的函数来完成自己的功能。例如，它们可能需要调用ART运行时的堆管理系统提供的接口来为对象分配内存空间。这样就会涉及到一个模块依赖性问题，就好像我们在编写程序时，需要依赖C库提供的接口一样。这要求Backend为类方法生成本地机器指令时，要处理调用其它模块提供的函数的问题。

ART运行时支持两种类型的Backend：Portable和Quick。Portable类型的Backend通过集成在LLVM编译框架里面的一个称为MCLinker的链接器来生成本地机器指令。关于MCLinker的更多知识，可以参考[https://code.google.com/p/mclinker](https://code.google.com/p/mclinker)。简单来说，假设我们有一个模块A，它依赖于模块B、C和D，那么在为模块A生成本地机器指令时，指出它依赖于模块B、C和D就行了。在生成的OAT文件中会记录好这些依赖关系，这是ELF文件格式本来就支持的特性。这些OAT文件要通过系统的动态链接器提供的dlopen函数来加载。函数dlopen在加载OAT文件的时候，会通过重定位技术来处理好它与其它模块的依赖关系，使得它能够调用其它模块提供的接口。这个实际上就通用的编译器、静态连接器以及动态链接器合作在一起干的事情，MCLinker扮演的就是静态链接器的角色。既然是通用的技术，因为就称能产生这种OAT文件的Backend为Portable类型的。

另一方面，Quick类型的Backend生成的本地机器指令用另外一种方式来处理依赖模块之间的依赖关系。简单来说，就是ART运行时会在每一个线程的TLS（线程本地区域）提供一个函数表。有了这个函数表之后，Quick类型的Backend生成的本地机器指令就可以通过它来调用其它模块的函数。也就是说，Quick类型的Backend生成的本地机器指令要依赖于ART运行时提供的函数表。这使得Quick类型的Backend生成的OAT文件在加载时不需要再处理模式之间的依赖关系。再通俗一点说的就是Quick类型的Backend生成的OAT文件在加载时不需要重定位，因此就不需要通过系统的动态链接器提供的dlopen函数来加载。由于省去重定位这个操作，Quick类型的Backend生成的OAT文件在加载时就会更快，这也是称为Quick的缘由。

关于ART运行时类型为Portable和Quick两种类型的Backend，我们就暂时讲解到这里，后面分析ART运行时执行类方法的时候，我们再详细分析。现在我们需要知道的就是，如果在编译ART运行时时，定义了宏ART_USE_PORTABLE_COMPILER，那么就表示要使用Portable类型的Backend来生成OAT文件，否则就使用Quick类型的Backend来生成OAT文件。默认情况下，使用的是Quick类型的Backend。

接下就可以很好地理解OatFile类的静态成员函数Open的实现了：

如果编译时指定了ART_USE_PORTABLE_COMPILER宏，并且参数executable为true，那么就通过OatFile类的静态成员函数OpenDlopen来加载指定的OAT文件。OatFile类的静态成员函数OpenDlopen直接通过动态链接器提供的dlopen函数来加载OAT文件。

其余情况下，通过OatFile类的静态成员函数OpenElfFile来手动加载指定的OAT文件。这种方式是按照ELF文件格式来解析要加载的OAT文件的，并且根据解析获得的信息将OAT里面相应的段加载到内存中来。

接下来我们就分别看看OatFile类的静态成员函数OpenDlopen和OpenElfFile的实现，以便可以对OAT文件有更清楚的认识。

OatFile类的静态成员函数OpenDlopen的实现如下所示：

```c
OatFile* OatFile::OpenDlopen(const std::string& elf_filename,  
                     const std::string& location,  
                     byte* requested_base) {  
UniquePtr<OatFile> oat_file(new OatFile(location));  
bool success = oat_file->Dlopen(elf_filename, requested_base);  
if (!success) {  
    return NULL;  
}  
return oat_file.release();  
}  
```
这个函数定义在文件art/runtime/oat_file.cc中。

OatFile类的静态成员函数OpenDlopen首先是创建一个OatFile对象，接着再调用该OatFile对象的成员函数Dlopen加载参数elf_filename指定的OAT文件。

OatFile类的成员函数Dlopen的实现如下所示：

```c
bool OatFile::Dlopen(const std::string& elf_filename, byte* requested_base) {  
char* absolute_path = realpath(elf_filename.c_str(), NULL);  
......  

dlopen_handle_ = dlopen(absolute_path, RTLD_NOW);  
......  

begin_ = reinterpret_cast<byte*>(dlsym(dlopen_handle_, "oatdata"));  
......  

if (requested_base != NULL && begin_ != requested_base) {  
    ......  
    return false;  
}  

end_ = reinterpret_cast<byte*>(dlsym(dlopen_handle_, "oatlastword"));  
......  

// Readjust to be non-inclusive upper bound.  
end_ += sizeof(uint32_t);  
return Setup();  
}  
```
这个函数定义在文件art/runtime/oat_file.cc中。

OatFile类的成员函数Dlopen首先是通过动态链接器提供的dlopen函数将参数elf_filename指定的OAT文件加载到内存中来，接着同样是通过动态链接器提供的dlsym函数从加载进来的OAT文件获得两个导出符号oatdata和oatlastword的地址，分别保存在当前正在处理的OatFile对象的成员变量begin_和end_中。根据图1所示，符号oatdata的地址即为OAT文件里面的oatdata段加载到内存中的开始地址，而符号oatlastword的地址即为OAT文件里面的oatexec加载到内存中的结束地址。符号oatlastword本身也是属于oatexec段的，它自己占用了一个地址，也就是sizeof(uint32_t)个字节，于是将前面得到的end_值加上sizeof(uint32_t)，得到的才是oatexec段的结束地址。

实际上，上面得到的`begin_`值指向的是加载内存中的oatdata段的头部，即OAT头。这个OAT头描述了OAT文件所包含的DEX文件的信息，以及定义在这些DEX文件里面的类方法所对应的本地机器指令在内存的位置。另外，上面得到的`end_`是用来在解析OAT头时验证数据的正确性的。此外，如果参数requested_base的值不等于0，那么就要求oatdata段必须要加载到requested_base指定的位置去，也就是上面得到的begin_值与requested_base值相等，否则的话就会出错返回。

最后，OatFile类的成员函数Dlopen通过调用另外一个成员函数Setup来解析已经加载内存中的oatdata段，以获得ART运行时所需要的更多信息。我们分析完成OatFile类的静态成员函数OpenElfFile之后，再来看OatFile类的成员函数Setup的实现。

OatFile类的静态成员函数OpenElfFile的实现如下所示：

```c
OatFile* OatFile::OpenElfFile(File* file,  
                      const std::string& location,  
                      byte* requested_base,  
                      bool writable,  
                      bool executable) {  
UniquePtr<OatFile> oat_file(new OatFile(location));  
bool success = oat_file->ElfFileOpen(file, requested_base, writable, executable);  
if (!success) {  
    return NULL;  
}  
return oat_file.release();  
}  
```
这个函数定义在文件art/runtime/oat_file.cc中。

OatFile类的静态成员函数OpenElfFile创建了一个OatFile对象后，就调用它的成员函数ElfFileOpen来执行加载OAT文件的工作，它的实现如下所示：

```c
bool OatFile::ElfFileOpen(File* file, byte* requested_base, bool writable, bool executable) {  
elf_file_.reset(ElfFile::Open(file, writable, true));  
......  
bool loaded = elf_file_->Load(executable);  
......  
begin_ = elf_file_->FindDynamicSymbolAddress("oatdata");  
......  
if (requested_base != NULL && begin_ != requested_base) {  
    ......  
    return false;  
}  
end_ = elf_file_->FindDynamicSymbolAddress("oatlastword");  
......  
// Readjust to be non-inclusive upper bound.  
end_ += sizeof(uint32_t);  
return Setup();  
}  
```
这个函数定义在文件art/runtime/oat_file.cc中。

OatFile类的静态成员函数OpenElfFile的实现与前面分析的成员函数Dlopen是很类似的，唯一不同的是前者通过ElfFile类来手动加载参数file指定的OAT文件，实际上就是按照ELF文件格式来解析参数file指定的OAT文件，并且将文件里面的oatdata段和oatexec段加载到内存中来。我们可以将ElfFile类看作是ART运行时自己实现的OAT文件动态链接器。一旦参数file指定的OAT文件指定的文件加载完成之后，我们同样是通过两个导出符号oatdata和oatlastword来获得oatdata段和oatexec段的起止位置。同样，如果参数requested_base的值不等于0，那么就要求oatdata段必须要加载到requested_base指定的位置去。

将参数file指定的OAT文件加载到内存之后，OatFile类的静态成员函数OpenElfFile最后也是调用OatFile类的成员函数Setup来解析其中的oatdata段。OatFile类的成员函数Setup定义在文件art/runtime/oat_file.cc中，我们分三部分来阅读，以便可以更好地理解OAT文件的格式。

OatFile类的成员函数Setup的第一部分实现如下所示：

```c
bool OatFile::Setup() {  
if (!GetOatHeader().IsValid()) {  
    LOG(WARNING) << "Invalid oat magic for " << GetLocation();  
    return false;  
}  
const byte* oat = Begin();  
oat += sizeof(OatHeader);  
if (oat > End()) {  
    LOG(ERROR) << "In oat file " << GetLocation() << " found truncated OatHeader";  
    return false;  
}  
```
我们先来看OatFile类的三个成员函数GetOatHeader、Begin和End的实现，如下所示：

```c
const OatHeader& OatFile::GetOatHeader() const {  
return *reinterpret_cast<const OatHeader*>(Begin());  
}  

const byte* OatFile::Begin() const {  
CHECK(begin_ != NULL);  
return begin_;  
}  

const byte* OatFile::End() const {  
CHECK(end_ != NULL);  
return end_;  
}  
```
这三个函数主要是涉及到了OatFile类的两个成员变量begin_和end_，它们分别是OAT文件里面的oatdata段开始地址和oatexec段的结束地址。

通过OatFile类的成员函数GetOatHeader可以清楚地看到，OAT文件里面的oatdata段的开始储存着一个OAT头，这个OAT头通过类OatHeader描述，定义在文件art/runtime/oat.h中，如下所示：

```c
class PACKED(4) OatHeader {  
public:  
......  
private:  
uint8_t magic_[4];  
uint8_t version_[4];  
uint32_t adler32_checksum_;  

InstructionSet instruction_set_;  
uint32_t dex_file_count_;  
uint32_t executable_offset_;  
uint32_t interpreter_to_interpreter_bridge_offset_;  
uint32_t interpreter_to_compiled_code_bridge_offset_;  
uint32_t jni_dlsym_lookup_offset_;  
uint32_t portable_resolution_trampoline_offset_;  
uint32_t portable_to_interpreter_bridge_offset_;  
uint32_t quick_resolution_trampoline_offset_;  
uint32_t quick_to_interpreter_bridge_offset_;  

uint32_t image_file_location_oat_checksum_;  
uint32_t image_file_location_oat_data_begin_;  
uint32_t image_file_location_size_;  
uint8_t image_file_location_data_[0];  // note variable width data at end  

......  
};  
```
类OatHeader的各个成员变量的含义如下所示：

- magic: 标志OAT文件的一个魔数，等于‘oat\n’
- version: OAT文件版本号，目前的值等于‘007、0’
- `adler32_checksum_`: OAT头部检验和
- `instruction_set_`: 本地机指令集，有四种取值，分别为  kArm(1)、kThumb2(2)、kX86(3)和kMips(4)
- `dex_file_count_`: OAT文件包含的DEX文件个数
- `executable_offset_`: oatexec段开始位置与oatdata段开始位置的偏移值
- `interpreter_to_interpreter_bridge_offset_`和`interpreter_to_compiled_code_bridge_offset_`: 

ART运行时在启动的时候，可以通过-Xint选项指定所有类的方法都是解释执行的，这与传统的虚拟机使用解释器来执行类方法差不多。同时，有些类方法可能没有被翻译成本地机器指令，这时候也要求对它们进行解释执行。这意味着解释执行的类方法在执行的过程中，可能会调用到另外一个也是解释执行的类方法，也可能调用到另外一个按本地机器指令执行的类方法中。OAT文件在内部提供有两段trampoline代码，分别用来从解释器调用另外一个也是通过解释器来执行的类方法和从解释器调用另外一个按照本地机器执行的类方法。这两段trampoline代码的偏移位置就保存在成员变量 `interpreter_to_interpreter_bridge_offset_`和`interpreter_to_compiled_code_bridge_offset_`。

- `jni_dlsym_lookup_offset_`: 

类方法在执行的过程中，如果要调用另外一个方法是一个JNI函数，那么就要通过存在放置`jni_dlsym_lookup_offset_`的一段trampoline代码来调用。

- `portable_resolution_trampoline_offset_`和`quick_resolution_trampoline_offset_`: 

用来在运行时解析还未链接的类方法的两段trampoline代码。其中，`portable_resolution_trampoline_offset_`指向的trampoline代码用于Portable类型的Backend生成的本地机器指令，而`quick_resolution_trampoline_offset_`用于Quick类型的Backend生成的本地机器指令。

`portable_to_interpreter_bridge_offset_`和`quick_to_interpreter_bridge_offset_: 与`interpreter_to_interpreter_bridge_offset_`和`interpreter_to_compiled_code_bridge_offset_`的作用刚好相反，用来在按照本地机器指令执行的类方法中调用解释执行的类方法的两段trampoline代码。其中，`portable_to_interpreter_bridge_offset_`用于Portable类型的Backend生成的本地机器指令，而`quick_to_interpreter_bridge_offset_`用于Quick类型的Backend生成的本地机器指令。

由于每一个应用程序都会依赖于boot.art文件，因此为了节省由打包在应用程序里面的classes.dex生成的OAT文件的体积，上述`interpreter_to_interpreter_bridge_offset_`、`interpreter_to_compiled_code_bridge_offset_`、`jni_dlsym_lookup_offset_`、`portable_resolution_trampoline_offset_`、`portable_to_interpreter_bridge_offset_`、`quick_resolution_trampoline_offset_`和`quick_to_interpreter_bridge_offset_`七个成员变量指向的trampoline代码段只存在于boot.art文件中。换句话说，在由打包在应用程序里面的classes.dex生成的OAT文件的oatdata段头部中，上述七个成员变量的值均等于0。

- `image_file_location_data_`: 用来创建Image空间的文件的路径的在内存中的地址
- `image_file_location_size_`: 用来创建Image空间的文件的路径的大小
- `image_file_location_oat_data_begin_`: 用来创建Image空间的OAT文件的oatdata段在内存的位置。
- `image_file_location_oat_checksum_`:用来创建Image空间的OAT文件的检验和

上述四个成员变量记录了一个OAT文件所依赖的用来创建Image空间文件以及创建这个Image空间文件所使用的OAT文件的相关信息。

通过OatFile类的成员函数Setup的第一部分代码的分析，我们就知道了，OAT文件的oatdata段在最开始保存着一个OAT头，如图2所示：

![img](http://img.blog.csdn.net/20140923010150806?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图2 OAT头部

我们接着再看OatFile类的成员函数Setup的第二部分代码：

```c
oat += GetOatHeader().GetImageFileLocationSize();  
if (oat > End()) {  
LOG(ERROR) << "In oat file " << GetLocation() << " found truncated image file location: "  
     << reinterpret_cast<const void*>(Begin())  
     << "+" << sizeof(OatHeader)  
     << "+" << GetOatHeader().GetImageFileLocationSize()  
     << "<=" << reinterpret_cast<const void*>(End());  
return false;  
}  
```
调用OatFile类的成员函数GetOatHeader获得的是正在打开的OAT文件的头部OatHeader，通过调用它的成员函数GetImageFileLocationSize获得的是正在打开的OAT依赖的Image空间文件的路径大小。变量oat最开始的时候指向oatdata段的开始位置。读出OAT头之后，变量oat就跳过了OAT头。由于正在打开的OAT文件引用的Image空间文件路径保存在紧接着OAT头的地方。因此，将Image空间文件的路径大小增加到变量oat去后，就相当于是跳过了保存Image空间文件路径的位置。

通过OatFile类的成员函数Setup的第二部分代码的分析，我们就知道了，紧接着在OAT头后面的是Image空间文件路径，如图3所示：

![img](http://img.blog.csdn.net/20140923011120468?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图3 OAT头和Image空间文件路径

我们接着再看OatFile类的成员函数Setup的第三部分代码：       

```c
  for (size_t i = 0; i < GetOatHeader().GetDexFileCount(); i++) {  
      size_t dex_file_location_size = *reinterpret_cast<const uint32_t*>(oat);  
      ......  
  
      oat += sizeof(dex_file_location_size);  
      ......  
  
      const char* dex_file_location_data = reinterpret_cast<const char*>(oat);  
      oat += dex_file_location_size;  
      ......  
  
      std::string dex_file_location(dex_file_location_data, dex_file_location_size);  
  
      uint32_t dex_file_checksum = *reinterpret_cast<const uint32_t*>(oat);  
      oat += sizeof(dex_file_checksum);  
      ......  
  
      uint32_t dex_file_offset = *reinterpret_cast<const uint32_t*>(oat);  
      ......  
        
      oat += sizeof(dex_file_offset);  
      ......  
  
      const uint8_t* dex_file_pointer = Begin() + dex_file_offset;  
      if (!DexFile::IsMagicValid(dex_file_pointer)) {  
        ......  
        return false;  
      }  
      if (!DexFile::IsVersionValid(dex_file_pointer)) {  
        ......  
        return false;  
      }  
  
      const DexFile::Header* header = reinterpret_cast<const DexFile::Header*>(dex_file_pointer);  
      const uint32_t* methods_offsets_pointer = reinterpret_cast<const uint32_t*>(oat);  
  
      oat += (sizeof(*methods_offsets_pointer) * header->class_defs_size_);  
      ......  
  
      oat_dex_files_.Put(dex_file_location, new OatDexFile(this,  
                                                   dex_file_location,  
                                                   dex_file_checksum,  
                                                   dex_file_pointer,  
                                                   methods_offsets_pointer));  
  }  
  return true;  
}  
```

这部分代码用来获得包含在oatdata段的DEX文件描述信息。每一个DEX文件记录在oatdata段的描述信息包括：

DEX文件路径大小，保存在变量dex_file_location_size中；

DEX文件路径，保存在变量dex_file_location_data中；

DEX文件检验和，保存在变量dex_file_checksum中；

DEX文件内容在oatdata段的偏移，保存在变量dex_file_offset中；

DEX文件包含的类的本地机器指令信息偏移数组，保存在变量methods_offsets_pointer中；

在上述五个信息中，最重要的就是第4个和第5个信息了。

通过第4个信息，我们可以在oatdata段中找到对应的DEX文件的内容。DEX文件最开始部分是一个DEX文件头，上述代码通过检查DEX文件头的魔数和版本号来确保变量dex_file_offset指向的位置确实是一个DEX文件。

通过第5个信息我们可以找到DEX文件里面的每一个类方法对应的本地机器指令。这个数组的大小等于header->class_defs_size_，即DEX文件里面的每一个类在数组中都对应有一个偏移值。这里的header指向的是DEX文件头，它的class_defs_size_描述了DEX文件包含的类的个数。在DEX文件中，每一个类都是有一个从0开始的编号，该编号就是用来索引到上述数组的，从而获得对应的类所有方法的本地机器指令信息。

最后，上述得到的每一个DEX文件的信息都被封装在一个OatDexFile对象中，以便以后可以直接访问。如果我们使用OatDexFile来描述每一个DEX文件的描述信息，那么就可以通过图4看到这些描述信息在oatdata段的位置：

![img](http://img.blog.csdn.net/20140924005112468?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图4 OAT头、Image空间文件路径、DEX文件描述信息

为了进一步理解包含在oatdata段的DEX文件描述信息，我们继续看OatDexFile类的构造函数的实现，如下所示：

```c
OatFile::OatDexFile::OatDexFile(const OatFile* oat_file,  
                        const std::string& dex_file_location,  
                        uint32_t dex_file_location_checksum,  
                        const byte* dex_file_pointer,  
                        const uint32_t* oat_class_offsets_pointer)  
    : oat_file_(oat_file),  
      dex_file_location_(dex_file_location),  
      dex_file_location_checksum_(dex_file_location_checksum),  
      dex_file_pointer_(dex_file_pointer),  
      oat_class_offsets_pointer_(oat_class_offsets_pointer) {}  
```
这个函数定义在文件art/runtime/oat_file.cc中。

OatDexFile类的构造函数的实现很简单，它将我们在上面得到的DEX文件描述息保存在相应的成员变量中。通过这些信息，我们就可以获得包含在该DEX文件里面的类的所有方法的本地机器指令信息。

例如，通过调用OatDexFile类的成员函数GetOatClass可以获得指定类的所有方法的本地机器指令信息：

```c
const OatFile::OatClass* OatFile::OatDexFile::GetOatClass(uint16_t class_def_index) const {  
uint32_t oat_class_offset = oat_class_offsets_pointer_[class_def_index];  

const byte* oat_class_pointer = oat_file_->Begin() + oat_class_offset;  
CHECK_LT(oat_class_pointer, oat_file_->End()) << oat_file_->GetLocation();  
mirror::Class::Status status = *reinterpret_cast<const mirror::Class::Status*>(oat_class_pointer);  

const byte* methods_pointer = oat_class_pointer + sizeof(status);  
CHECK_LT(methods_pointer, oat_file_->End()) << oat_file_->GetLocation();  

return new OatClass(oat_file_,  
              status,  
              reinterpret_cast<const OatMethodOffsets*>(methods_pointer));  
}  
```
这个函数定义在文件art/runtime/oat_file.cc中。

参数`class_def_index`表示要查找的目标类的编号。这个编号用作数组`oat_class_offsets_pointer_`（即前面描述的methods_offsets_pointer数组）的索引，就可以得到一个偏移位置oat_class_offset。这个偏移位置是相对于OAT文件的oatdata段的，因此将该偏移值加上OAT文件的oatdata段的开始位置后，就可以得到目标类的所有方法的本地机器指令信息。这些信息的布局如图5所示：

![img](http://img.blog.csdn.net/20140924011349861?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图5 DEX文件里面的类描述信息

在OAT文件中，每一个DEX文件包含的每一个类的描述信息都通过一个OatClass对象来描述。为了方便描述，我们称之为OAT类。我们通过OatClass类的构造函数来理解它的作用，如下所示：

```c
OatFile::OatClass::OatClass(const OatFile* oat_file,  
                    mirror::Class::Status status,  
                    const OatMethodOffsets* methods_pointer)  
    : oat_file_(oat_file), status_(status), methods_pointer_(methods_pointer) {}  
```
这个函数定义在文件art/runtime/oat_file.cc中。

参数oat_file描述的是宿主OAT文件，参数status描述的是OAT类状态，参数methods_pointer是一个数组，描述的是OAT类的各个方法的信息，它们被分别保存在OatClass类的相应成员变量中。通过这些信息，我们就可以获得包含在该DEX文件里面的类的所有方法的本地机器指令信息。

例如，通过调用OatClass类的成员函数GetOatMethod可以获得指定类方法的本地机器指令信息：

```c
const OatFile::OatMethod OatFile::OatClass::GetOatMethod(uint32_t method_index) const {  
const OatMethodOffsets& oat_method_offsets = methods_pointer_[method_index];  
return OatMethod(  
      oat_file_->Begin(),  
      oat_method_offsets.code_offset_,  
      oat_method_offsets.frame_size_in_bytes_,  
      oat_method_offsets.core_spill_mask_,  
      oat_method_offsets.fp_spill_mask_,  
      oat_method_offsets.mapping_table_offset_,  
      oat_method_offsets.vmap_table_offset_,  
      oat_method_offsets.gc_map_offset_);  
}  
```
这个函数定义在文件art/runtime/oat_file.cc中。

参数method_index描述的目标方法在类中的编号，用这个编号作为索引，就可以在OatClass类的成员变量methods_pointer_指向的一个数组中找到目标方法的本地机器指令信息。这些本地机器指令信息封装在一个OatMethod对象，它们在OAT文件的布局如图6下所示：

![img](http://img.blog.csdn.net/20140924013404573?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图6 DEX文件里面的类(OatClass)描述信息

为了进一步理解OatMethod的作用，我们继续看它的构造函数的实现，如下所示：

```c
OatFile::OatMethod::OatMethod(const byte* base,  
                      const uint32_t code_offset,  
                      const size_t frame_size_in_bytes,  
                      const uint32_t core_spill_mask,  
                      const uint32_t fp_spill_mask,  
                      const uint32_t mapping_table_offset,  
                      const uint32_t vmap_table_offset,  
                      const uint32_t gc_map_offset)  
: begin_(base),  
    code_offset_(code_offset),  
    frame_size_in_bytes_(frame_size_in_bytes),  
    core_spill_mask_(core_spill_mask),  
    fp_spill_mask_(fp_spill_mask),  
    mapping_table_offset_(mapping_table_offset),  
    vmap_table_offset_(vmap_table_offset),  
    native_gc_map_offset_(gc_map_offset) {  
    ......  
}  
```
这个函数定义在文件art/runtime/oat_file.cc中。

OatMethod类包含了很多对应类方法的本地机器指令执行时要用到的信息，其中，最重要的就是参数base和code_offset描述的信息。

参数base描述的是OAT文件的OAT头在内存的位置，而参数code_offset描述的是类方法的本地机器指令相对OAT头的偏移位置。将这两者相加，就可以得到一个类方法的本地机器指令在内存的位置。我们可以通过调用OatMethod类的成员函数GetCode来获得这个结果。

OatMethod类的成员函数GetCode的实现如下所示：

```c
const void* OatFile::OatMethod::GetCode() const {  
return GetOatPointer<const void*>(code_offset_);  
}  
```
这个函数定义在文件art/runtime/oat_file.cc中。

OatMethod类的成员函数调用另外一个成员函数GetOatPointer来获得一个类方法的本地机器指令在内存的位置。

OatMethod类的成员函数GetOatPointer的实现如下所示：

```c
class OatFile {  
......  

class OatMethod {  
......  

private:  
    template<class T>  
    T GetOatPointer(uint32_t offset) const {  
      if (offset == 0) {  
return NULL;  
      }  
      return reinterpret_cast<T>(begin_ + offset);  
    }  
    
......  
};  

......  
};  
```
这个函数定义在文件art/runtime/oat_file.h中。

通过上面对OAT文件加载过程的分析，我们就可以清楚地看到OAT文件的格式，以及如何在OAT文件中找到一个类方法的本地机器指令。我们通过图7来总结在OAT文件中找到一个类方法的本地机器指令的过程：

![img](http://img.blog.csdn.net/20140924020914024?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图7 在OAT文件中查找类方法的本地机器指令的过程

我们从左往右来看图7。首先是根据类签名信息从包含在OAT文件里面的DEX文件中查找目标Class的编号，然后再根据这个编号找到在OAT文件中找到对应的OatClass。接下来再根据方法签名从包含在OAT文件里面的DEX文件中查找目标方法的编号，然后再根据这个编号在前面找到的OatClass中找到对应的OatMethod。有了这个OatMethod之后，我们就根据它的成员变量begin_和code_offset_找到目标类方法的本地机器指令了。其中，从DEX文件中根据签名找到类和方法的编号要求对DEX文件进行解析，这就需要利用Dalvik虚拟机的知识了。

至此，我们就通过OAT文件的加载过程分析完成OAT文件的格式了。为了加深对OAT文件格式的理解，有接下来的一篇文章中，我们再详细分析上面描述的类方法的本地机器指令的查找过程。敬请关注！更多信息也可以关注老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)。