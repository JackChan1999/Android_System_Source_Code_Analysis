摘要: 前些天为了在科室做培训，我基于Android 4.4重新整理了一份关于zygote的文档。从技术的角度看，这几年zygote并没有出现什么大的变化，所以如果有人以前研究过zygote，应该不会对本文写的内容感到陌生。 本篇文章和我的上一篇文章《Android4.4的init进程》可以算是姊妹篇啦。读完这两篇文章，我相信大家对Android的启动流程能有一些大面上的认识了。

# 1. 背景

前些天为了在科室做培训，我基于Android 4.4重新整理了一份关于zygote的文档。从技术的角度看，这几年zygote并没有出现什么大的变化，所以如果有人以前研究过zygote，应该不会对本文写的内容感到陌生。

# 2. zygote进程的描述

在Android中，zygote是整个系统创建新进程的核心装置。从字面上看，zygote是受精卵的意思，它的主要工作就是进行细胞分裂。

zygote进程在内部会先启动Dalvik虚拟机，继而加载一些必要的系统资源和系统类，最后进入一种监听状态。在后续的运作中，当其他系统模块（比如AMS）希望创建新进程时，只需向zygote进程发出请求，zygote进程监听到该请求后，会相应地“分裂”出新的进程，于是这个新进程在初生之时，就先天具有了自己的Dalvik虚拟机以及系统资源。

系统启动伊始，zygote进程就会被init进程启动起来，init进程的详情可参考我写的《Android4.4的init进程》一文，此处不再赘述。我们直接来看init.rc脚本里的相关描述吧。在这个脚本中是这样描述zygote的：

可以看到，zygote对应的可执行文件就是/system/bin/app_process，也就是说系统启动时会执行到这个可执行文件的main()函数里。

# 3. zygote进程的实现细节

zygote服务的main()函数位于frameworks\base\cmds\app_process\App_main.cpp文件中，其代码截选如下：

```c
int main(int argc, char* const argv[])
{
    . . . . . .
    AppRuntime runtime;
    const char* argv0 = argv[0];    // /system/bin/app_process
    argc--;
    argv++;
    . . . . . .
    int i = runtime.addVmArguments(argc, argv); // 会跳过-Xzygote，i的位置对应/system/bin
    . . . . . .
    while (i < argc) {
        const char* arg = argv[i++];		// 应该是/system/bin目录
        if (!parentDir) {
            parentDir = arg;
        } else if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = "zygote";
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } 
        . . . . . .
    }

    if (niceName && *niceName) {
        setArgv0(argv0, niceName);
        set_process_name(niceName);     // 一般改名为“zygote”
    }
    runtime.mParentDir = parentDir;
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit",
                startSystemServer ? "start-system-server" : "");
    } else if (className) {
        . . . . . .
    } else {
        . . . . . .
    }
}
```

## 3.1 AppRuntime的start()

main()函数里先构造了一个AppRuntime对象，即AppRuntime runtime；而后把进程名改成“zygote”，并利用runtime对象，把工作转交给java层的ZygoteInit类处理。

这个AppRuntime类继承于AndroidRuntime类，却没有重载其start(...)函数，所以main()函数中调用的runtime.start(...)其实走的是AndroidRuntime的start(...)，而且传入了类名参数，即字符串——“com.android.internal.os.ZygoteInit”。start()函数的主要代码截选如下：

```c
void AndroidRuntime::start(const char* className, const char* options)
{
    . . . . . .
    const char* rootDir = getenv("ANDROID_ROOT");
    . . . . . .
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);				// 初始化JNI接口
    JNIEnv* env;
    if (startVm(&mJavaVM, &env) != 0) {		// 启动虚拟机
        return;
    }
    onVmCreated(env);

    if (startReg(env) < 0) {				// 注册系统需要的jni函数
        ALOGE("Unable to register all android natives\n");
        return;
    }
    . . . . . .
    jclass startClass = env->FindClass(slashClassName);
    . . . . . .
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        . . . . . .
            env->CallStaticVoidMethod(startClass, startMeth, strArray);
    . . . . . .
}
```

抛开Java层和C++层的概念，上面的流程说白了就是，Zygote进程的main()函数在启动Dalvik虚拟机后，会调用另一个ZygoteInit类的main()静态函数。调用示意图如下：

![img](http://static.oschina.net/uploads/space/2015/0621/172847_tCDg_174429.png)

### 3.1.1 加载合适的虚拟机动态库

一开始需要初始化JNI接口。

```c
JniInvocation jni_invocation;
jni_invocation.Init(NULL);
```

jni_invocation的init()的代码如下：
【libnativehelper/JniInvocation.cpp】

```c
bool JniInvocation::Init(const char* library) 
{
#ifdef HAVE_ANDROID_OS
  char default_library[PROPERTY_VALUE_MAX];
  property_get(kLibrarySystemProperty, default_library, kLibraryFallback);
#else
  const char* default_library = kLibraryFallback;
#endif
  if (library == NULL) {
    library = default_library;
  }

  handle_ = dlopen(library, RTLD_NOW);
  . . . . . .
  . . . . . .
  if (!FindSymbol(reinterpret_cast<void**>(&JNI_GetDefaultJavaVMInitArgs_),
                  "JNI_GetDefaultJavaVMInitArgs")) {
    return false;
  }
  if (!FindSymbol(reinterpret_cast<void**>(&JNI_CreateJavaVM_),
                  "JNI_CreateJavaVM")) {
    return false;
  }
  if (!FindSymbol(reinterpret_cast<void**>(&JNI_GetCreatedJavaVMs_),
                  "JNI_GetCreatedJavaVMs")) {
    return false;
  }
  return true;
}
```

因为我们使用的是Android系统，所以已经定义了HAVE_ANDROID_OS宏，而且library参数为NULL，于是在JniInvocation的Init()函数中，会走到

```c
property_get(kLibrarySystemProperty, default_library, kLibraryFallback);
```

具体加载动态库的函数是dlopen()，代码如下：
【bionic/linker/Dlfcn.cpp】

```c
void* dlopen(const char* filename, int flags) {
  ScopedPthreadMutexLocker locker(&gDlMutex);
  soinfo* result = do_dlopen(filename, flags);
  if (result == NULL) {
    __bionic_format_dlerror("dlopen failed", linker_get_error_buffer());
    return NULL;
  }
  return result;
}
```

本文不需细究dlopen()的实现，大家只需知道，它是个强大的库函数，可以打开某个动态库，并将之装入内存。调用dlopen()时传入的第二个参数是RTLD_NOW，它表示加载器会立即计算库的依赖性，从而在dlopen()返回之前，解析出每个未定义变量的地址。

### 3.1.2 启动Dalvik虚拟机，startVm()

初始化JNI环境后，就可以启动Dalvik虚拟机了。下面是startVm()的代码截选：
【frameworks/base/core/jni/AndroidRuntime.cpp】

```c
int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv)
{
    . . . . . .
    JavaVMInitArgs initArgs;
    JavaVMOption opt;
    . . . . . .
    . . . . . .
    opt.extraInfo = (void*) runtime_exit;
    opt.optionString = "exit";
    mOptions.add(opt);
    . . . . . .
    // Increase the main thread's interpreter stack size for bug 6315322.
    opt.optionString = "-XX:mainThreadStackSize=24K";
    mOptions.add(opt);
    . . . . . .
    . . . . . .

    initArgs.version = JNI_VERSION_1_4;
    initArgs.options = mOptions.editArray();
    initArgs.nOptions = mOptions.size();
    initArgs.ignoreUnrecognized = JNI_FALSE;

    // 启动dalvik虚拟机
    if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {
        . . . . . .
    }
    . . . . . .
}
```

因为本文阐述的重点不是Dalvik虚拟机，所以就不再深究JNI_CreateJavaVM()了。我们大概知道该函数会调用到刚刚jni_invocation的init()中，FindSymbol()得到的动态库中相应的函数指针即可。

### 3.1.3 注册Android内部需要的函数，startReg()

当虚拟机成功启动后，JNI环境也就建立好了，现在可以把JNIEnv*传递给startReg()来注册一些重要的JNI接口了。startReg()的代码截选如下：
【frameworks/base/core/jni/AndroidRuntime.cpp】

```c
int AndroidRuntime::startReg(JNIEnv* env)
{
    androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);
    . . . . . .
    env->PushLocalFrame(200);
    if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0) {
        env->PopLocalFrame(NULL);
        return -1;
    }
    env->PopLocalFrame(NULL);
    return 0;
}
```

```c
static int register_jni_procs(const RegJNIRec array[], size_t count, JNIEnv* env)
{
    for (size_t i = 0; i < count; i++) {
        if (array[i].mProc(env) < 0) { // 回调每个RegJNIRec数组项的mProc
#ifndef NDEBUG
            ALOGD("----------!!! %s failed to load\n", array[i].mName);
#endif
            return -1;
        }
    }
    return 0;
}
```

注册jni函数的动作很简单，只是在遍历array数组，并尝试回调每个数组项的mProc回调函数。具体数组项类型的定义如下，而且struct定义的上方还顺带定义了REG_JNI宏。：

```c
#ifdef NDEBUG
    #define REG_JNI(name)      { name }
    struct RegJNIRec {
        int (*mProc)(JNIEnv*);
    };
#else
    #define REG_JNI(name)      { name, #name }
    struct RegJNIRec {
        int (*mProc)(JNIEnv*);
        const char* mName;
    };
#endif
```

gRegJNI的定义截选如下：

```c
static const RegJNIRec gRegJNI[] = {
    REG_JNI(register_android_debug_JNITest),
    REG_JNI(register_com_android_internal_os_RuntimeInit),   // 举例
    REG_JNI(register_android_os_SystemClock),
    REG_JNI(register_android_util_EventLog),
    REG_JNI(register_android_util_Log),
    REG_JNI(register_android_util_FloatMath),
    REG_JNI(register_android_text_format_Time),
REG_JNI(register_android_content_AssetManager),
    . . . . . .
    . . . . . .
```

这些注册动作内部，基本上就是为自己关心的类注册jni接口啦。比如上面的register_com_android_internal_os_RuntimeInit()函数，它的代码如下：
【frameworks/base/core/jni/AndroidRuntime.cpp】

```c
int register_com_android_internal_os_RuntimeInit(JNIEnv* env)
{
    return jniRegisterNativeMethods(env, "com/android/internal/os/RuntimeInit",
                                         gMethods, NELEM(gMethods));
}
```

它关心的就是RuntimeInit类，现在为这个类的native成员注册对应的实现函数，这些实现函数就记录在gMethods数组中：

```c
static JNINativeMethod gMethods[] = {
    { "nativeFinishInit", "()V",
        (void*) com_android_internal_os_RuntimeInit_nativeFinishInit },
    { "nativeZygoteInit", "()V",
        (void*) com_android_internal_os_RuntimeInit_nativeZygoteInit },
    { "nativeSetExitWithoutCleanup", "(Z)V",
        (void*) com_android_internal_os_RuntimeInit_nativeSetExitWithoutCleanup },
};
```

其他“注册动作”的格局大概也都是这样，我们就不赘述了。

### 3.1.4 加载ZygoteInit类

AppRuntime的start()最后会加载Java层次的ZygoteInit类，并利用JNI技术的CallStaticVoidMethod()调用其静态的main()函数。

```c
jclass startClass = env->FindClass(slashClassName);
    . . . . . .
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        . . . . . .
            env->CallStaticVoidMethod(startClass, startMeth, strArray);
```

这是很关键的一步，就在这一步，控制权就转移到Java层次了。

## 3.2 走入Java层——ZygoteInit.java

随着控制权传递到Java层次，ZygoteInit要做一些和Android平台紧密相关的重要动作，比如创建LocalServerSocket对象、预加载一些类以及资源、启动“Android系统服务”、进入核心循环等等。我们先画一张示意图：

![img](http://static.oschina.net/uploads/space/2015/0621/174529_MBKE_174429.png)
相应地，我们还可以把前文的调用关系也丰富一下，得到下图：

![img](http://static.oschina.net/uploads/space/2015/0621/174554_hfbb_174429.png)

### 3.2.1 registerZygoteSocket()

我们先看ZygoteInit的main()函数调用的那个registerZygoteSocket()。这个函数内部其实会利用一个叫作“ANDROID_SOCKET_zygote”的环境变量。可是这个环境变量又是从哪里来的呢？为了解答这个问题，我们需要先看一下init进程service_start()函数。

#### 3.2.1.1 先看一下init进程的service_start()

前文我们已经列出了在init.rc脚本中，zygote服务是如何声明的。现在我们只关心其中和socket相关的部分：

```c
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    . . . . . .
    socket zygote stream 660 root system
    . .  . . . .
```

这个服务的socket选项表明，它需要一个名为“zygote”的“流型（stream）”socket。

当init进程真的启动zygote服务时，会走到service_start()。我们现在只关心service_start()里和socket相关的动作。

```c
void service_start(struct service *svc, const char *dynamic_args)
{
    . . . . . .
    pid = fork();   // 先fork出新的service进程
    if (pid == 0) 
    {
        struct socketinfo *si;
        struct svcenvinfo *ei;
        . . . . . .
        // 再为service进程创建必须的socket接口
         for (si = svc->sockets; si; si = si->next) 
        {
            int socket_type = (
                    !strcmp(si->type, "stream") ? SOCK_STREAM :
                        (!strcmp(si->type, "dgram") ? SOCK_DGRAM : SOCK_SEQPACKET));
            int s = create_socket(si->name, socket_type,
                                      si->perm, si->uid, si->gid);
            if (s >= 0) {
                // 将socket接口记入ANDROID_SOCKET_zygote环境变量
                publish_socket(si->name, s);  
            }
        }
        . . . . . .
            if (execve(svc->args[0], (char**) svc->args, (char**) ENV) < 0) {
                ERROR("cannot execve('%s'): %s\n", svc->args[0], strerror(errno));
            }
     }
     . . . . . .
 }
```

每次为service创建新的子进程后，都会查看该service需要什么socket。比如zygote服务，明确提出需要一个“流型（stream）”的socket。

create_socket()会在/dev/socket目录中创建一个Unix范畴的socket，而后，publish_socket()会把新建的socket的文件描述符记录在以“ANDROID_SOCKET_”打头的环境变量中。比如zygote对应的socket选项中的socket名为“zygote”，那么该socket对应的环境变量名就是“ANDROID_SOCKET_zygote”。

create_socket()的代码截选如下：
【system/core/init/Util.c】

```c
int create_socket(const char *name, int type, mode_t perm, uid_t uid, gid_t gid)
{
    struct sockaddr_un addr;
    int fd, ret;
    . . . . . .
    fd = socket(PF_UNIX, type, 0);
. . . . . .
    // "/dev/socket/zygote"
    snprintf(addr.sun_path, sizeof(addr.sun_path), ANDROID_SOCKET_DIR"/%s", name);
    ret = unlink(addr.sun_path);
    . . . . . .
    ret = bind(fd, (struct sockaddr *) &addr, sizeof (addr));  // 给套接字命名
    . . . . . .
    chown(addr.sun_path, uid, gid);
    chmod(addr.sun_path, perm);
    . . . . . .
    return fd;
    . . . . . .
}
```

当用socket()函数创建套接字以后，使用bind()将“指定的地址”赋值给“用文件描述符代表的套接字”，一般来说，该操作被称为“给套接字命名”。通常情况下，在一个SOCK_STREAM套接字接收连接之前，必须通过bind()函数用本地地址为套接字命名。对于zygote，其套接字地址应该是“/dev/socket/zygote”。在调用bind()函数之后，socket()函数创建的套接字已经和指定的地址关联起来了，现在向这个地址发送的数据，就可以通过套接字读取出来了。

接下来，service_start()还调用了个publish_socket()函数，该函数的代码如下：
【system/core/init/Init.c】

```c
static void publish_socket(const char *name, int fd)
{
    char key[64] = ANDROID_SOCKET_ENV_PREFIX;
    char val[64];

    strlcpy(key + sizeof(ANDROID_SOCKET_ENV_PREFIX) - 1,
            name,
            sizeof(key) - sizeof(ANDROID_SOCKET_ENV_PREFIX));
    snprintf(val, sizeof(val), "%d", fd);  // 将文件描述符转为字符串
    add_environment(key, val);

    /* make sure we don't close-on-exec */
    fcntl(fd, F_SETFD, 0);
}
```

这么看来，所谓的“发布”（publish），主要是把socket的文件描述符记录进环境变量。
上面代码中的ANDROID_SOCKET_ENV_PREFIX的定义如下：

```c
#define ANDROID_SOCKET_ENV_PREFIX "ANDROID_SOCKET_"
```

那么对于zygote而言，就是在环境变量“ANDROID_SOCKET_zygote”里记录文件描述符对应的字符串。

add_environment()的代码如下：

```c
int add_environment(const char *key, const char *val)
{
    int n;
    for (n = 0; n < 31; n++) {
        if (!ENV[n]) {
            size_t len = strlen(key) + strlen(val) + 2;
            char *entry = malloc(len);
            snprintf(entry, len, "%s=%s", key, val);
            ENV[n] = entry;
            return 0;
        }
    }
    return 1;
}
```

无非是把字符串记入一个静态数组而已，在后续的代码里，service_start()会调用execve()，并把ENV环境变量传递给execve()。

#### 3.2.1.2 registerZygoteSocket()里创建LocalServerSocket

OK，我们已经看到init进程在新fork出的zygote进程里，是如何记录“ANDROID_SOCKET_zygote”环境变量的。现在我们可以回过头来看zygote中的registerZygoteSocket()了，此处会切实地用到这个环境变量。

registerZygoteSocket()的代码如下：
【frameworks/base/core/java/com/android/internal/os/ZygoteInit.java】

```c
private static void registerZygoteSocket() {
    if (sServerSocket == null) {
        int fileDesc;
        try {
            String env = System.getenv(ANDROID_SOCKET_ENV);
            fileDesc = Integer.parseInt(env);   // 从环境变量的字符串中解析出文件描述符
        } catch (RuntimeException ex) {
            throw new RuntimeException(
                    ANDROID_SOCKET_ENV + " unset or invalid", ex);
        }

        try {
            sServerSocket = new LocalServerSocket(createFileDescriptor(fileDesc));
        } catch (IOException ex) {
            throw new RuntimeException(
                    "Error binding to local socket '" + fileDesc + "'", ex);
        }
    }
}
```

先从环境变量里读出socket的文件描述符，然后创建LocalServerSocket对象，并记入静态变量sServerSocket中。以后zygote进程会循环监听这个socket，一旦accept到连接请求，就创建命令连接（Command Connection）。监听动作的细节是在runSelectLoop()中，我们会在后文阐述，这里先放下。

现在我们可以画一张创建zygote socket接口的示意图，如下：

![img](http://static.oschina.net/uploads/space/2015/0621/235824_J1XZ_174429.png)

请注意，图中明确画出了两个进程，一个add环境变量，另一个get环境变量。

### 3.2.2 预加载一些类——preloadClasses()

注册完socket接口，ZygoteInit会预加载一些类，这些类记录在frameworks/base/preloaded-classes文本文件里。下面是该文件的一部分截选：

```c
# Classes which are preloaded by com.android.internal.os.ZygoteInit.
# Automatically generated by frameworks/base/tools/preload/WritePreloadedClassFile.java.
# MIN_LOAD_TIME_MICROS=1250
# MIN_PROCESSES=10
android.R$styleable
android.accounts.Account
android.accounts.Account$1
android.accounts.AccountManager
android.accounts.AccountManager$12
android.accounts.AccountManager$13
android.accounts.AccountManager$6
android.accounts.AccountManager$AmsTask
android.accounts.AccountManager$AmsTask$1
android.accounts.AccountManager$AmsTask$Response
. . . . . .
. . . . . .
```

在Android4.4上，这个脚本文件已经长达两千七百多行了，它里面记录着加载时间超过1250微秒的类，ZygoteInit尝试在系统启动时就把它们预加载进来，从而省去后续频繁加载时带来的系统开销。

preloadClasses()的代码截选如下：

```c
private static void preloadClasses() 
{
    . . . . . .
    InputStream is = ClassLoader.getSystemClassLoader().getResourceAsStream(
                                            PRELOADED_CLASSES);  // 即"preloaded-classes"
    . . . . . .
    . . . . . .
        try {
            BufferedReader br = new BufferedReader(new InputStreamReader(is), 256);
            . . . . . .
            while ((line = br.readLine()) != null) 
            {
                line = line.trim();
                    . . . . . .
                    Class.forName(line);  // 使用加载当前类的类加载器来加载指定类
                    . . . . . .
                    count++;
                . . . . . .
            }
            . . . . . .
        } catch (IOException e) {
            . . . . . .
        } finally {
            . . . . . .
            runtime.preloadDexCaches();
            . . . . . .
        }
    . . . . . .
}
```

在一个while循环里，每次读取一行，然后调用Class.forName()方法来装载类。这个工作量可不小，毕竟有两千多行哩。

### 3.2.3 预加载一些系统资源——preloadResources()

除了预加载一些类，zygote进程还要预加载一些系统资源。

```c
private static void preloadResources() 
{
    . . . . . .
        mResources = Resources.getSystem();
        mResources.startPreloading();
        if (PRELOAD_RESOURCES) {
            . . . . . .
            TypedArray ar = mResources.obtainTypedArray(
                    com.android.internal.R.array.preloaded_drawables);
            int N = preloadDrawables(runtime, ar);
            ar.recycle();
            . . . . . .
            ar = mResources.obtainTypedArray(
                    com.android.internal.R.array.preloaded_color_state_lists);
            N = preloadColorStateLists(runtime, ar);
            ar.recycle();
            . . . . . .
        }
        mResources.finishPreloading();
    . . . . . .
}
```

首先，从preloaded_drawables数组资源中读取一个类型数组（TypedArray），具体的资源文件可参考frameworks/base/core/res/res/values/arrays.xml，截选如下：

基本上有两大类资源：

1）一类和图片有关（preloaed_drawables）
2）另一类和颜色有关（preloaded_color_state_lists）

加载第一类资源需要调用preloadDrawables()，逐个加载TypedArray里记录的图片资源：

```c
private static int preloadDrawables(VMRuntime runtime, TypedArray ar) 
{
    int N = ar.length();
    for (int i=0; i<N; i++) {
        . . . . . .
        int id = ar.getResourceId(i, 0);  // 获得i项对应的资源id
        . . . . . .
        if (id != 0) {
            if (mResources.getDrawable(id) == null) {
                throw new IllegalArgumentException(
                        "Unable to find preloaded drawable resource #0x"
                        + Integer.toHexString(id)
                        + " (" + ar.getString(i) + ")");
            }
        }
    }
    return N;
}
```

说起来之前mResources.obtainTypedArray()获取TypedArray时，其内部用的是AssetManager。得到TypedArray之后，我们就可以通过调用ar.getResourceId(i, 0)来得到数组项对应的资源id了。

其中的mResources是ZygoteInit的私有静态成员：

```c
private static Resources mResources;
```

mResources的getDrawable()函数内部，会调用loadDrawable()。这样，这些图片资源就都加载到ZygoteInit的mResources里了。

另一些资源是颜色资源，是用preloadColorStateLists()加载的：

```c
private static int preloadColorStateLists(VMRuntime runtime, TypedArray ar) {
    int N = ar.length();
    for (int i=0; i<N; i++) {
        . . . . . .
        int id = ar.getResourceId(i, 0);
        . . . . . .
        if (id != 0) {
            if (mResources.getColorStateList(id) == null) {
                throw new IllegalArgumentException(
                        "Unable to find preloaded color resource #0x"
                        + Integer.toHexString(id)
                        + " (" + ar.getString(i) + ")");
            }
        }
    }
    return N;
}
```

也是在一个for循环里逐个加载颜色集，比如arrays.xml里的
<item>@color/primary_text_dark</item>
这个颜色集的参考文件是frameworks/base/core/res/res/color/primary_text_dark.xml，

![img](http://static.oschina.net/uploads/space/2015/0622/000607_w56d_174429.png)

现在，我们画一张加载系统资源的调用关系图：

![img](http://static.oschina.net/uploads/space/2015/0622/000640_snAy_174429.png)

### 3.2.4 启动Android系统服务——startSystemServer()

接下来就是启动Android的重头戏了，此时ZygoteInit的main()函数会调用startSystemServer()，该函数用于启动整个Android系统的系统服务。其大体做法是先fork一个子进程，然后在子进程中做一些初始化动作，继而执行SystemServer类的main()静态函数。需要注意的是，startSystemServer()并不是在函数体内直接调用Java类的main()函数的，而是通过抛异常的方式，在startSystemServer()之外加以处理的。

startSystemServer()的代码如下：

```c
private static boolean startSystemServer()
        throws MethodAndArgsCaller, RuntimeException 
{
    . . . . . .
    /* Hardcoded command line to start the system server */
    String args[] = {
        "--setuid=1000",
        "--setgid=1000",
        "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1032,
                        3001,3002,3003,3006,3007",
        "--capabilities=" + capabilities + "," + capabilities,
        "--runtime-init",
        "--nice-name=system_server",
        "com.android.server.SystemServer",
    };
    ZygoteConnection.Arguments parsedArgs = null;
    int pid;
    try {
        parsedArgs = new ZygoteConnection.Arguments(args);
        ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
        ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

        // fork出系统服务对应的进程
        pid = Zygote.forkSystemServer(parsedArgs.uid, parsedArgs.gid,
                                            parsedArgs.gids, parsedArgs.debugFlags, null,
                                            parsedArgs.permittedCapabilities,
                                            parsedArgs.effectiveCapabilities);
    } catch (IllegalArgumentException ex) {
        throw new RuntimeException(ex);
    }

    // 对新fork出的系统进程，执行handleSystemServerProcess()
    if (pid == 0) {
        handleSystemServerProcess(parsedArgs);
    }
    return true;
}
```

| args[]中的字符串                         | 对应                              |
| ---------------------------------------- | ---------------------------------------- |
| "--setuid=1000"                          | parsedArgs.uid                           |
| "--setgid=1000"                          | parsedArgs.gid                           |
| "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1032,3001,3002,3003,3006,3007" | parsedArgs.gids                          |
| "--capabilities=" + capabilities + "," + capabilities | capabilitiesSpecified = true;permittedCapabilities = Long.decode(capStrings[0]);effectiveCapabilites = Long.decode(capString[1]); |
| "--runtime-init"                         | parsedArgs.runtimeInit设为true             |
| "--nice-name=system_server"              | parsedArgs.niceName                      |
| "com.android.server.SystemServer"        | parsedArgs.remainingArgs                 |

#### 3.2.4.1 Zygote.forkSystemServer()

Zygote.forkSystemServer()的代码如下：
【libcore/dalvik/src/main/java/dalvik/system/Zygote.java】

```c
public static int forkSystemServer(int uid, int gid, int[] gids, int debugFlags,
        int[][] rlimits, long permittedCapabilities, long effectiveCapabilities) 
{
    preFork();
    int pid = nativeForkSystemServer(uid, gid, gids, debugFlags, rlimits, 
                                     permittedCapabilities, effectiveCapabilities);
    postFork();
    return pid;
}
```

其中的nativeForkSystemServer()是个native成员函数，其对应的C++层函数为Dalvik_dalvik_system_Zygote_forkSystemServer()。
【dalvik/vm/native/dalvik_system_Zygote.cpp】

```c
const DalvikNativeMethod dvm_dalvik_system_Zygote[] = {
    { "nativeFork", "()I",
      Dalvik_dalvik_system_Zygote_fork },
    { "nativeForkAndSpecialize", "(II[II[[IILjava/lang/String;Ljava/lang/String;)I",
      Dalvik_dalvik_system_Zygote_forkAndSpecialize },
    { "nativeForkSystemServer", "(II[II[[IJJ)I",
      Dalvik_dalvik_system_Zygote_forkSystemServer },
    { NULL, NULL, NULL },
};
```

```c
static void Dalvik_dalvik_system_Zygote_forkSystemServer(
        const u4* args, JValue* pResult)
{
    pid_t pid;
    pid = forkAndSpecializeCommon(args, true);

    if (pid > 0) {
        int status;

        ALOGI("System server process %d has been created", pid);
        gDvm.systemServerPid = pid;
        if (waitpid(pid, &status, WNOHANG) == pid) {
            ALOGE("System server process %d has died. Restarting Zygote!", pid);
            kill(getpid(), SIGKILL);
        }
    }
    RETURN_INT(pid);
}
```

forkAndSpecializeCommon()内部其实会调用fork()，而后设置gid、uid等信息。

#### 3.2.4.2 SystemServer的handleSystemServerProgress()函数

接着，startSystemServer()会在新fork出的子进程中调用handleSystemServerProgress()，让这个新进程成为真正的系统进程（SystemServer进程）。

```c
// 对新fork出的系统进程，执行handleSystemServerProcess()
    if (pid == 0) {
        handleSystemServerProcess(parsedArgs);
    }
```

注意，调用handleSystemServerProcess()时，程序是运行在新fork出的进程中的。handleSystemServerProcess()的代码如下：
【frameworks/base/core/java/com/android/internal/os/ZygoteInit.java】

```c
private static void handleSystemServerProcess(ZygoteConnection.Arguments parsedArgs)
                                           throws ZygoteInit.MethodAndArgsCaller 
{
    closeServerSocket();
    Libcore.os.umask(S_IRWXG | S_IRWXO);

    if (parsedArgs.niceName != null) {
        Process.setArgV0(parsedArgs.niceName);  // niceName就是”system_server”
    }

    if (parsedArgs.invokeWith != null) {
        WrapperInit.execApplication(parsedArgs.invokeWith,
                parsedArgs.niceName, parsedArgs.targetSdkVersion,
                null, parsedArgs.remainingArgs);
} else {
        // 此时的remainingArgs就是”com.android.server.SystemServer”
        RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs);
    }
}
```

##### 3.2.4.2.1 closeServerSocket()

因为当前已经不是运行在zygote进程里了，所以zygote里的那个监听socket就应该关闭了。这就是closeServerSocket()的意义，其代码如下：

```c
static void closeServerSocket() 
{
    try {
        if (sServerSocket != null) {
            FileDescriptor fd = sServerSocket.getFileDescriptor();
            sServerSocket.close();
            if (fd != null) {
                Libcore.os.close(fd);
            }
        }
    } catch (IOException ex) {
        Log.e(TAG, "Zygote:  error closing sockets", ex);
    } catch (libcore.io.ErrnoException ex) {
        Log.e(TAG, "Zygote:  error closing descriptor", ex);
    }
    sServerSocket = null;
}
```

在handleSystemServerProcess()函数里，parsedArgs.niceName就是“system_server”，而且因为parsedArgs.invokeWith没有指定，所以其值为null，于是程序会走到RuntimeInit.zygoteInit()。

##### 3.2.4.2.2 RuntimeInit.zygoteInit()

RuntimeInit.zygoteInit()的代码如下：
【frameworks/base/core/java/com/android/internal/os/RuntimeInit.java】

```c
public static final void zygoteInit(int targetSdkVersion, String[] argv)
        throws ZygoteInit.MethodAndArgsCaller 
{
    if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application from zygote");
    redirectLogStreams();
    commonInit();
    nativeZygoteInit();
    applicationInit(targetSdkVersion, argv);
}
```

###### 3.2.4.2.2.1. 调用redirectLogStreams()

首先，在新fork出的系统进程里，需要重新定向系统输出流。

```c
public static void redirectLogStreams() 
{
    System.out.close();
    System.setOut(new AndroidPrintStream(Log.INFO, "System.out"));
    System.err.close();
    System.setErr(new AndroidPrintStream(Log.WARN, "System.err"));
}
```

###### 3.2.4.2.2.2.调用commonInit()

```c
private static final void commonInit() 
{
    . . . . . .
    Thread.setDefaultUncaughtExceptionHandler(new UncaughtHandler());


    TimezoneGetter.setInstance(new TimezoneGetter() 
    . . . . . .
    . . . . . .
    String trace = SystemProperties.get("ro.kernel.android.tracing");
    . . . . . .
    initialized = true;
}
```

当前正处于系统进程的主线程中，可以调用Thread.setDefaultUncaughtExceptionHandler()来设置一个默认的异常处理器，处理程序中的未捕获异常。其他的初始化动作，我们暂不深究。

###### 3.2.4.2.2.3. 调用nativeZygoteInit()

接下来调用的nativeZygoteInit()是个JNI函数，在AndroidRuntime.cpp文件中可以看到：
【frameworks/base/core/jni/AndroidRuntime.cpp】

```c
static JNINativeMethod gMethods[] = {
    { "nativeFinishInit", "()V",
        (void*) com_android_internal_os_RuntimeInit_nativeFinishInit },
    { "nativeZygoteInit", "()V",
        (void*) com_android_internal_os_RuntimeInit_nativeZygoteInit },
    { "nativeSetExitWithoutCleanup", "(Z)V",
        (void*) com_android_internal_os_RuntimeInit_nativeSetExitWithoutCleanup },
};
```

nativeZygoteInit()对应的本地函数为com_android_internal_os_RuntimeInit_nativeZygoteInit()。

```c
static void com_android_internal_os_RuntimeInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
{
    gCurRuntime->onZygoteInit();
}
```

gCurRuntime是C++层的AndroidRuntime类的静态变量。在AndroidRuntime构造之时，
gCurRuntime = this。不过实际调用的onZygoteInit()应该是AndroidRuntime的子类AppRuntime的：
【frameworks/base/cmds/app_process/App_main.cpp】

```c
class AppRuntime : public AndroidRuntime
{
    . . . . . .
    virtual void onZygoteInit()
    {
        // Re-enable tracing now that we're no longer in Zygote.
        atrace_set_tracing_enabled(true);


        sp<ProcessState> proc = ProcessState::self();
        ALOGV("App process: starting thread pool.\n");
        proc->startThreadPool();
    }
```

里面构造了进程的ProcessState全局对象，而且启动了线程池。

ProcessState对象是典型的单例模式，它的self()函数如下：

```c
sp<ProcessState> ProcessState::self()
{
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != NULL) {
        return gProcess;
    }
    gProcess = new ProcessState;
    return gProcess;
}
```

ProcessState对于Binder通信机制而言非常重要，现在system server进程的PrecessState算是初始化完毕了。

我们整理一下思路，画一张startSystemServer()的调用关系图：

![img](http://static.oschina.net/uploads/space/2015/0622/203214_ROdx_174429.png)

接下来我们来讲上图中zygoteInit()调用的最后一行：applicationInit()。

###### 3.2.4.2.2.4.调用applicationInit()

applicationInit()函数的代码如下：
【frameworks/base/core/java/com/android/internal/os/RuntimeInit.java】

```c
private static void applicationInit(int targetSdkVersion, String[] argv)
        throws ZygoteInit.MethodAndArgsCaller 
{
    nativeSetExitWithoutCleanup(true);
    VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);
    VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);

    final Arguments args;
    try {
        args = new Arguments(argv);
    } catch (IllegalArgumentException ex) {
        Slog.e(TAG, ex.getMessage());
        return;
    }
    invokeStaticMain(args.startClass, args.startArgs);
}
```

其中的invokeStaticMain()一句最为关键，它承担向外抛出“特殊异常”的作用。我们先画一张startSystemServer()的调用关系图：
![img](http://static.oschina.net/uploads/space/2015/0622/203613_9Noi_174429.png)

看到了吧，最后一步抛出了异常。这相当于一个“特殊的goto语句”！上面的cl = Class.forName(className)一句，其实加载的就是SystemServer类。这个类名是从前文的parsedArgs.remainingArgs得来的，其值就是“com.android.server.SystemServer”。此处抛出的异常，会被本进程的catch语句接住，在那里才会执行SystemServer类的main()函数。示意图如下：

![img](http://static.oschina.net/uploads/space/2015/0622/203639_0dbt_174429.png) 

如上图所示，新fork出的SystemServer子进程直接跳过了中间那句runSelectLoop()，径直跳转到caller.run()一步了。

当然，父进程Zygote在fork动作后，会退出startSystemServer()函数，并走到runSelectLoop()，从而进入一种循环监听状态，每当Activity Manager Service向它发出“启动新应用进程”的命令时，它又会fork一个子进程，并在子进程里抛出一个异常，这样子进程还是会跳转到catch一句。

我们可以把上面的示意图再丰富一下：

还有一点需要说明一下，fork出的SystemServer进程在跳转到catch语句后，会执行SystemServer类的main()函数，而其他情况下，fork出的应用进程在跳转的catch语句后，则会执行ActivityThread类的main()函数。这个ActivityThread对于应用程序而言非常重要，但因为和本篇主题关系不大，我们就不在这里展开讲了。

#### 3.2.4.3 SystemServer的main()函数

前文我们已经看到了，startSystemServer()创建的新进程在执行完applicationInit()之后，会抛出一个异常，并由新fork出的SystemServer子进程的catch语句接住，继而执行SystemServer类的main()函数。

那么SystemServer的main()函数又在做什么事情呢？其调用关系图如下：

![img](http://static.oschina.net/uploads/space/2015/0622/203851_LRSx_174429.png)

在Android4.4版本中，ServerThread已经不再继承于Thread了，它现在只是个辅助类，其命名还残留有旧代码的味道。在以前的Android版本中，ServerThread的确继承于Thread，而且在线程的run()成员函数里，做着类似addService、systemReady的工作。

因为本文主要是阐述zygote进程的，所以我们就不在这里继续细说system server进程啦，有兴趣的同学可以继续研究。我们还是回过头继续说zygote里的动作吧。

### 3.2.5监听zygote socket

#### 3.2.5.1runSelectLoop()

ZygoteInit的main()函数在调用完startSystemServer()之后，会进一步走到runSelectLoop()。runSelectInit()的代码如下：
【frameworks/base/core/java/com/android/internal/os/ZygoteInit.java】

```c
private static void runSelectLoop() throws MethodAndArgsCaller 
{
    ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
    ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
    FileDescriptor[] fdArray = new FileDescriptor[4];

    fds.add(sServerSocket.getFileDescriptor());
    peers.add(null);

    int loopCount = GC_LOOP_COUNT;
    while (true) {
        int index;

        if (loopCount <= 0) {
            gc();
            loopCount = GC_LOOP_COUNT;
        } else {
            loopCount--;
        }

        try {
            fdArray = fds.toArray(fdArray);
            index = selectReadable(fdArray);
        } catch (IOException ex) {
            throw new RuntimeException("Error in select()", ex);
        }

        if (index < 0) {
            throw new RuntimeException("Error in select()");
        } else if (index == 0) {
            ZygoteConnection newPeer = acceptCommandPeer();
            peers.add(newPeer);
            fds.add(newPeer.getFileDesciptor());
        } else {
            boolean done;
            done = peers.get(index).runOnce();
            if (done) {
                peers.remove(index);
                fds.remove(index);
            }
        }
    }
}
```

在一个while循环中，不断调用selectReadable()。该函数是个native函数，对应C++层的com_android_internal_os_ZygoteInit_selectReadable()。
【frameworks/base/core/jni/com_android_internal_os_ZygoteInit.cpp】

```c
static jint com_android_internal_os_ZygoteInit_selectReadable (JNIEnv *env, jobject clazz, jobjectArray fds)
{
    . . . . . .
    int err;
    do {
        err = select (nfds, &fdset, NULL, NULL, NULL);
    } while (err < 0 && errno == EINTR);
    . . . . . .
    for (jsize i = 0; i < length; i++) {
        jobject fdObj = env->GetObjectArrayElement(fds, i);
        . . . . . .
        int fd = jniGetFDFromFileDescriptor(env, fdObj);
        . . . . . .
        if (FD_ISSET(fd, &fdset)) {
            return (jint)i;
        }
    }
    return -1;
}
```

可以看到，主要就是调用select()而已。在Linux的socket编程中，select()负责监视若干文件描述符的变化情况，我们常见的变化情况有：读、写、异常等等。在zygote中，
err = select (nfds, &fdset, NULL, NULL, NULL);一句的最后三个参数都为NULL，表示该select()操作只打算监视文件描述符的“读变化”，而且如果没有可读的文件，select()就维持阻塞状态。

在被监视的文件描述符数组（fds）中，第一个文件描述符对应着“zygote接收其他进程连接申请的那个socket（及sServerSocket）”，一旦它发生了变化，我们就尝试建立一个ZygoteConnection。

```c
// (index == 0)的情况
ZygoteConnection newPeer = acceptCommandPeer();
peers.add(newPeer);
fds.add(newPeer.getFileDesciptor());
```

看到了吗，新创建的ZygoteConnection会被再次写入文件描述符数组（fds）。

如果select动作发现文件描述符数组（fds）的其他文件描述符有东西可读了，说明有其他进程通过某个已建立好的ZygoteConnection发来了命令，此时我们需要调用runOnce()。

```c
// (index > 0)的情况
boolean done;
done = peers.get(index).runOnce();
if (done) {
    peers.remove(index);
    fds.remove(index);
}
```

建立ZygoteConnection的acceptCommandPeer()的代码如下：

```c
private static ZygoteConnection acceptCommandPeer() {
    try {
        return new ZygoteConnection(sServerSocket.accept());
    } catch (IOException ex) {
        throw new RuntimeException(
                "IOException during accept()", ex);
    }
}
```

##### 3.2.5.1.1ZygoteConnection的runOnce()

ZygoteConnection的runOnce()代码截选如下：

```c
boolean runOnce() throws ZygoteInit.MethodAndArgsCaller {

    String args[];
    Arguments parsedArgs = null;
    FileDescriptor[] descriptors;

    . . . . . .
        args = readArgumentList();
        descriptors = mSocket.getAncillaryFileDescriptors();
    . . . . . . 
    int pid = -1;
    FileDescriptor childPipeFd = null;
    FileDescriptor serverPipeFd = null;

    try {
        parsedArgs = new Arguments(args);
        . . . . . .
        pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
                parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, 
                parsedArgs.seInfo, parsedArgs.niceName);
    } 
    . . . . . .
        if (pid == 0) {
            // in child
            IoUtils.closeQuietly(serverPipeFd);
            serverPipeFd = null;
            handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);
            return true;
        } else {
            // in parent...pid of < 0 means failure
            IoUtils.closeQuietly(childPipeFd);
            childPipeFd = null;
            return handleParentProc(pid, descriptors, serverPipeFd, parsedArgs);
        }
    . . . . . .
}
```

##### 3.2.5.1.2readArgumentList()

runOnce()中从socket中读取参数数据的动作是由readArgumentList()完成的，该函数的代码如下：

```c
private String[] readArgumentList()
        throws IOException 
{
    int argc;
    . . . . . .
        String s = mSocketReader.readLine();
    . . . . . .
        argc = Integer.parseInt(s);
    . . . . . .
    String[] result = new String[argc];
    for (int i = 0; i < argc; i++) {
        result[i] = mSocketReader.readLine();
        if (result[i] == null) {
            // We got an unexpected EOF.
            throw new IOException("truncated request");
        }
    }
    return result;
}
```

可是是谁在向这个socket写入参数的呢？当然是AMS啦。

我们知道，当AMS需要启动一个新进程时，会调用类似下面的句子：

```c
Process.ProcessStartResult startResult = Process.start("android.app.ActivityThread",
                    app.processName, uid, uid, gids, debugFlags, mountExternal,
                    app.info.targetSdkVersion, app.info.seinfo, null);
```

包括ActivityThread类名等重要信息的参数，最终就会通过socket传递给zygote。

##### 3.2.5.1.3handleChildProc()

runOnce()在读完参数之后，会进一步调用到handleChildProc()。正如前文所说，该函数会间接抛出特殊的MethodAndArgsCaller异常，只不过此时抛出的异常携带的类名为ActivityThread。

```c
private void handleChildProc(Arguments parsedArgs, FileDescriptor[] descriptors, 
                             FileDescriptor pipeFd, PrintStream newStderr)
                             throws ZygoteInit.MethodAndArgsCaller 
{
    closeSocket();
    ZygoteInit.closeServerSocket();
    . . . . . .
    if (parsedArgs.niceName != null) {
        Process.setArgV0(parsedArgs.niceName);
    }

    if (parsedArgs.runtimeInit) {
        if (parsedArgs.invokeWith != null) {
            WrapperInit.execApplication(parsedArgs.invokeWith,
                    parsedArgs.niceName, parsedArgs.targetSdkVersion,
                    pipeFd, parsedArgs.remainingArgs);
        } else {
            RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion,
                    parsedArgs.remainingArgs);
        }
    } else {
        String className;
        . . . . . .
            className = parsedArgs.remainingArgs[0];
        . . . . . .
        String[] mainArgs = new String[parsedArgs.remainingArgs.length - 1];
        System.arraycopy(parsedArgs.remainingArgs, 1,
                mainArgs, 0, mainArgs.length);

        if (parsedArgs.invokeWith != null) {
            WrapperInit.execStandalone(parsedArgs.invokeWith,
                    parsedArgs.classpath, className, mainArgs);
        } else {
            ClassLoader cloader;
            if (parsedArgs.classpath != null) {
                cloader = new PathClassLoader(parsedArgs.classpath,
                        ClassLoader.getSystemClassLoader());
            } else {
                cloader = ClassLoader.getSystemClassLoader();
            }

            try {
                ZygoteInit.invokeStaticMain(cloader, className, mainArgs);
            } catch (RuntimeException ex) {
                logAndPrintError(newStderr, "Error starting.", ex);
            }
        }
    }
}
```

# 4小结

至此，zygote进程就阐述完毕了。作为一个最原始的“受精卵”，它必须在合适的时机进行必要的细胞分裂。分裂动作也没什么大的花样，不过就是fork()新进程而已。如果fork()出的新进程是system server，那么其最终执行的就是SystemServer类的main()函数，而如果fork()出的新进程是普通的用户进程的话，那么其最终执行的就是ActivityThread类的main()函数。有关ActivityThread的细节，我们有时间再深入探讨，这里就不细说了。

本篇文章和我的上一篇文章《Android4.4的init进程》可以算是姊妹篇啦。读完这两篇文章，我相信大家对Android的启动流程能有一些大面上的认识了。