在[Android](http://lib.csdn.net/base/android)系统中，应用程序进程都是由Zygote进程孵化出来的，而Zygote进程是由Init进程启动的。Zygote进程在启动时会创建一个Dalvik虚拟机实例，每当它孵化一个新的应用程序进程时，都会将这个Dalvik虚拟机实例复制到新的应用程序进程里面去，从而使得每一个应用程序进程都有一个独立的Dalvik虚拟机实例。在本文中，我们就分析Dalvik虚拟机在Zygote进程中的启动过程。

**老罗的新浪微博：http://weibo.com/shengyangluo，欢迎关注！**

《[android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

​        Zygote进程在启动的过程中，除了会创建一个Dalvik虚拟机实例之外，还会将[Java](http://lib.csdn.net/base/java)运行时库加载到进程中来，以及注册一些Android核心类的JNI方法来前面创建的Dalvik虚拟机实例中去。注意，一个应用程序进程被Zygote进程孵化出来的时候，不仅会获得Zygote进程中的Dalvik虚拟机实例拷贝，还会与Zygote一起共享Java运行时库，这完全得益于[Linux](http://lib.csdn.net/base/linux)内核的进程创建机制（fork）。这种Zygote孵化机制的优点是不仅可以快速地启动一个应用程序进程，还可以节省整体的内存消耗，缺点是会影响开机速度，毕竟Zygote是在开机过程中启动的。不过，总体来说，是利大于弊的，毕竟整个系统只有一个Zygote进程，而可能有无数个应用程序进程，而且我们不会经常去关闭手机，大多数情况下只是让它进入休眠状态。

​        从前面[Android系统进程Zygote启动过程的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6768304)一文可以知道，Zygote进程在启动的过程中，会调用到AndroidRuntime类的成员函数start，接下来我们就这个函数开始分析Dalvik虚拟机启动相关的过程，如图1所示：

![img](http://img.blog.csdn.net/20130506004759886)

图1 Dalvik虚拟机的启动过程

​        这个过程可以分为8个步骤，接下来我们就详细分析每一个步骤。

​        Step 1. AndroidRuntime.start

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8885792#) [copy](http://blog.csdn.net/luoshengyang/article/details/8885792#)

1. void AndroidRuntime::start(const char* className, const bool startSystemServer)  
2. {  
3. ​    ......  
4.   
5. ​    /* start the virtual machine */  
6. ​    if (startVm(&mJavaVM, &env) != 0)  
7. ​        goto bail;  
8.   
9. ​    /* 
10. ​     * Register android functions. 
11. ​     */  
12. ​    if (startReg(env) < 0) {  
13. ​        LOGE("Unable to register all android natives\n");  
14. ​        goto bail;  
15. ​    }  
16.   
17. ​    ......  
18.   
19. ​    /* 
20. ​     * Start VM.  This thread becomes the main thread of the VM, and will 
21. ​     * not return until the VM exits. 
22. ​     */  
23. ​    jclass startClass;  
24. ​    jmethodID startMeth;  
25.   
26. ​    slashClassName = strdup(className);  
27. ​    for (cp = slashClassName; *cp != '\0'; cp++)  
28. ​        if (*cp == '.')  
29. ​            *cp = '/';  
30.   
31. ​    startClass = env->FindClass(slashClassName);  
32. ​    if (startClass == NULL) {  
33. ​        LOGE("JavaVM unable to locate class '%s'\n", slashClassName);  
34. ​        /* keep going */  
35. ​    } else {  
36. ​        startMeth = env->GetStaticMethodID(startClass, "main",  
37. ​            "([Ljava/lang/String;)V");  
38. ​        if (startMeth == NULL) {  
39. ​            LOGE("JavaVM unable to find main() in '%s'\n", className);  
40. ​            /* keep going */  
41. ​        } else {  
42. ​            env->CallStaticVoidMethod(startClass, startMeth, strArray);  
43. ​            ......  
44. ​        }  
45. ​    }  
46.   
47. ​    LOGD("Shutting down VM\n");  
48. ​    if (mJavaVM->DetachCurrentThread() != JNI_OK)  
49. ​        LOGW("Warning: unable to detach main thread\n");  
50. ​    if (mJavaVM->DestroyJavaVM() != 0)  
51. ​        LOGW("Warning: VM did not shut down cleanly\n");  
52.   
53. ​    ......  
54. }  

​        这个函数定义在文件frameworks/base/core/jni/AndroidRuntime.cpp中。

​        AndroidRuntime类的成员函数start主要做了以下四件事情：

​        1. 调用成员函数startVm来创建一个Dalvik虚拟机实例，并且保存在成员变量mJavaVM中。

​        2. 调用成员函数startReg来注册一些Android核心类的JNI方法。

​        3. 调用参数className所描述的一个Java类的静态成员函数main，来作为Zygote进程的Java层入口。从前面[Android系统进程Zygote启动过程的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6768304)一文可以知道，这个入口类就为com.android.internal.os.ZygoteInit。执行这一步的时候，Zygote进程中的Dalvik虚拟机实例就开始正式运作了。注意，在这一步中，也就是在com.android.internal.os.ZygoteInit类的静态成员函数main，会进行大量的Android核心类和系统资源文件预加载。其中，预加载的Android核心类可以参考frameworks/base/preloaded-classes这个文件，而预加载的系统资源就是包含在/system/framework/framework-res.apk中。

​        4. 从com.android.internal.os.ZygoteInit类的静态成员函数main返回来的时候，就说明Zygote进程准备要退出来了。在退出之前，会调用前面创建的Dalvik虚拟机实例的成员函数DetachCurrentThread和DestroyJavaVM。其中，前者用来将Zygote进程的主线程脱离前面创建的Dalvik虚拟机实例，后者是用来销毁前面创建的Dalvik虚拟机实例。

​        接下来，我们就主要关注Dalvik虚拟机实例的创建过程，以及Android核心类JNI方法的注册过程，即AndroidRuntime类的成员函数startVm和startReg的实现。

​        Step 2. AndroidRuntime.startVm

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8885792#) [copy](http://blog.csdn.net/luoshengyang/article/details/8885792#)

1. int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv)  
2. {  
3. ​    int result = -1;  
4. ​    JavaVMInitArgs initArgs;  
5. ​    JavaVMOption opt;  
6. ​    ......  
7.   
8. ​    property_get("dalvik.vm.checkjni", propBuf, "");  
9. ​    if (strcmp(propBuf, "true") == 0) {  
10. ​        checkJni = true;  
11. ​    } else if (strcmp(propBuf, "false") != 0) {  
12. ​        /* property is neither true nor false; fall back on kernel parameter */  
13. ​        property_get("ro.kernel.android.checkjni", propBuf, "");  
14. ​        if (propBuf[0] == '1') {  
15. ​            checkJni = true;  
16. ​        }  
17. ​    }  
18. ​    ......  
19.   
20. ​    property_get("dalvik.vm.execution-mode", propBuf, "");  
21. ​    if (strcmp(propBuf, "int:portable") == 0) {  
22. ​        executionMode = kEMIntPortable;  
23. ​    } else if (strcmp(propBuf, "int:fast") == 0) {  
24. ​        executionMode = kEMIntFast;  
25. \#if defined(WITH_JIT)  
26. ​    } else if (strcmp(propBuf, "int:jit") == 0) {  
27. ​        executionMode = kEMJitCompiler;  
28. \#endif  
29. ​    }  
30.   
31. ​    property_get("dalvik.vm.stack-trace-file", stackTraceFileBuf, "");  
32. ​    ......  
33.   
34. ​    strcpy(heapsizeOptsBuf, "-Xmx");  
35. ​    property_get("dalvik.vm.heapsize", heapsizeOptsBuf+4, "16m");  
36. ​    //LOGI("Heap size: %s", heapsizeOptsBuf);  
37. ​    opt.optionString = heapsizeOptsBuf;  
38. ​    mOptions.add(opt);  
39. ​    ......  
40.   
41. ​    if (checkJni) {  
42. ​        /* extended JNI checking */  
43. ​        opt.optionString = "-Xcheck:jni";  
44. ​        mOptions.add(opt);  
45.   
46. ​        ......  
47. ​    }  
48. ​    ......  
49.   
50. ​    if (executionMode == kEMIntPortable) {  
51. ​        opt.optionString = "-Xint:portable";  
52. ​        mOptions.add(opt);  
53. ​    } else if (executionMode == kEMIntFast) {  
54. ​        opt.optionString = "-Xint:fast";  
55. ​        mOptions.add(opt);  
56. \#if defined(WITH_JIT)  
57. ​    } else if (executionMode == kEMJitCompiler) {  
58. ​        opt.optionString = "-Xint:jit";  
59. ​        mOptions.add(opt);  
60. \#endif  
61. ​    }  
62. ​    ......  
63.   
64. ​    if (stackTraceFileBuf[0] != '\0') {  
65. ​        static const char* stfOptName = "-Xstacktracefile:";  
66.   
67. ​        stackTraceFile = (char*) malloc(strlen(stfOptName) +  
68. ​            strlen(stackTraceFileBuf) +1);  
69. ​        strcpy(stackTraceFile, stfOptName);  
70. ​        strcat(stackTraceFile, stackTraceFileBuf);  
71. ​        opt.optionString = stackTraceFile;  
72. ​        mOptions.add(opt);  
73. ​    }  
74. ​    ......  
75.   
76. ​    initArgs.options = mOptions.editArray();  
77. ​    initArgs.nOptions = mOptions.size();  
78. ​    ......  
79.   
80. ​    /* 
81. ​     * Initialize the VM. 
82. ​     * 
83. ​     * The JavaVM* is essentially per-process, and the JNIEnv* is per-thread. 
84. ​     * If this call succeeds, the VM is ready, and we can start issuing 
85. ​     * JNI calls. 
86. ​     */  
87. ​    if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {  
88. ​        LOGE("JNI_CreateJavaVM failed\n");  
89. ​        goto bail;  
90. ​    }  
91.   
92. ​    result = 0;  
93.   
94. bail:  
95. ​    free(stackTraceFile);  
96. ​    return result;  
97. }  

​        这个函数定义在文件frameworks/base/core/jni/AndroidRuntime.cpp中。

​        在启动Dalvik虚拟机的时候，可以指定一系列的选项，这些选项可以通过特定的系统属性来指定。下面我们就简单了解几个可能有用的选项。

​        1. -Xcheck:jni：用来启动JNI方法检查。我们在C/C++代码中，可以修改Java对象的成员变量或者调用Java对象的成员函数。加了-Xcheck:jni选项之后，就可以对要访问的Java对象的成员变量或者成员函数进行合法性检查，例如，检查类型是否匹配。我们可以通过dalvik.vm.checkjni或者ro.kernel.android.checkjni这两个系统属性来指定是否要启用-Xcheck:jni选项。注意，加了-Xcheck:jni选项之后，会使用得JNI方法执行变慢。

​        2. -Xint:portable，-Xint:fast，-Xint:jit：用来指定Dalvik虚拟机的执行模式。Dalvik虚拟机支持三种运行模式，分别是Portable、Fast和Jit。Portable是指Dalvik虚拟机以可移植的方式来进行编译，也就是说，编译出来的虚拟机可以在任意平台上运行。Fast是针对当前平台对Dalvik虚拟机进行编译，这样编译出来的Dalvik虚拟机可以进行特殊的优化，从而使得它能更快地运行程序。Jit不是解释执行代码，而是将代码动态编译成本地语言后再执行。我们可以通过dalvik.vm.execution-mode系统属笥来指定Dalvik虚拟机的解释模式。

​        3. -Xstacktracefile：用来指定调用堆栈输出文件。Dalvik虚拟机接收到SIGQUIT（Ctrl-\或者kill -3）信号之后，会将所有线程的调用堆栈输出来，默认是输出到日志里面。指定了-Xstacktracefile选项之后，就可以将线程的调用堆栈输出到指定的文件中去。我们可以通过dalvik.vm.stack-trace-file系统属性来指定调用堆栈输出文件。

​        4. -Xmx：用来指定Java对象堆的最大值。Dalvik虚拟机的Java对象堆的默认最大值是16M，不过我们可以通过dalvik.vm.heapsize系统属性来指定为其它值。

​        更多的Dalvik虚拟机启动选项，可以参考[Controlling the Embedded VM](http://www.netmite.com/android/mydroid/2.0/dalvik/docs/embedded-vm-control.html)一文。

​        设置好Dalvik虚拟机的启动选项之后，AndroidRuntime的成员函数startVm就会调用另外一个函数JNI_CreateJavaVM来创建以及初始化一个Dalvik虚拟机实例。

​        Step 3. JNI_CreateJavaVM

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8885792#) [copy](http://blog.csdn.net/luoshengyang/article/details/8885792#)

1. /* 
2.  * Create a new VM instance. 
3.  * 
4.  * The current thread becomes the main VM thread.  We return immediately, 
5.  * which effectively means the caller is executing in a native method. 
6.  */  
7. jint JNI_CreateJavaVM(JavaVM** p_vm, JNIEnv** p_env, void* vm_args)  
8. {  
9. ​    const JavaVMInitArgs* args = (JavaVMInitArgs*) vm_args;  
10. ​    JNIEnvExt* pEnv = NULL;  
11. ​    JavaVMExt* pVM = NULL;  
12. ​    const char** argv;  
13. ​    int argc = 0;  
14. ​    ......  
15.   
16. ​    /* zero globals; not strictly necessary the first time a VM is started */  
17. ​    memset(&gDvm, 0, sizeof(gDvm));  
18.   
19. ​    /* 
20. ​     * Set up structures for JNIEnv and VM. 
21. ​     */  
22. ​    //pEnv = (JNIEnvExt*) malloc(sizeof(JNIEnvExt));  
23. ​    pVM = (JavaVMExt*) malloc(sizeof(JavaVMExt));  
24.   
25. ​    memset(pVM, 0, sizeof(JavaVMExt));  
26. ​    pVM->funcTable = &gInvokeInterface;  
27. ​    pVM->envList = pEnv;  
28. ​    ......  
29.   
30. ​    argv = (const char**) malloc(sizeof(char*) * (args->nOptions));  
31. ​    memset(argv, 0, sizeof(char*) * (args->nOptions));  
32. ​    ......  
33.   
34. ​    /* 
35. ​     * Convert JNI args to argv. 
36. ​     * 
37. ​     * We have to pull out vfprintf/exit/abort, because they use the 
38. ​     * "extraInfo" field to pass function pointer "hooks" in.  We also 
39. ​     * look for the -Xcheck:jni stuff here. 
40. ​     */  
41. ​    for (i = 0; i < args->nOptions; i++) {  
42. ​        ......  
43. ​    }  
44.   
45. ​    ......  
46.   
47. ​    /* set this up before initializing VM, so it can create some JNIEnvs */  
48. ​    gDvm.vmList = (JavaVM*) pVM;  
49.   
50. ​    /* 
51. ​     * Create an env for main thread.  We need to have something set up 
52. ​     * here because some of the class initialization we do when starting 
53. ​     * up the VM will call into native code. 
54. ​     */  
55. ​    pEnv = (JNIEnvExt*) dvmCreateJNIEnv(NULL);  
56.   
57. ​    /* initialize VM */  
58. ​    gDvm.initializing = true;  
59. ​    if (dvmStartup(argc, argv, args->ignoreUnrecognized, (JNIEnv*)pEnv) != 0) {  
60. ​        free(pEnv);  
61. ​        free(pVM);  
62. ​        goto bail;  
63. ​    }  
64.   
65. ​    /* 
66. ​     * Success!  Return stuff to caller. 
67. ​     */  
68. ​    dvmChangeStatus(NULL, THREAD_NATIVE);  
69. ​    *p_env = (JNIEnv*) pEnv;  
70. ​    *p_vm = (JavaVM*) pVM;  
71. ​    result = JNI_OK;  
72.   
73. bail:  
74. ​    gDvm.initializing = false;  
75. ​    if (result == JNI_OK)  
76. ​        LOGV("JNI_CreateJavaVM succeeded\n");  
77. ​    else  
78. ​        LOGW("JNI_CreateJavaVM failed\n");  
79. ​    free(argv);  
80. ​    return result;  
81. }  

​        这个函数定义在文件dalvik/vm/Jni.c中。

​        JNI_CreateJavaVM主要完成以下四件事情。

​        1. 为当前进程创建一个Dalvik虚拟机实例，即一个JavaVMExt对象。

​        2. 为当前线程创建和初始化一个JNI环境，即一个JNIEnvExt对象，这是通过调用函数dvmCreateJNIEnv来完成的。

​        3. 将参数vm_args所描述的Dalvik虚拟机启动选项拷贝到变量argv所描述的一个字符串数组中去，并且调用函数dvmStartup来初始化前面所创建的Dalvik虚拟机实例。

​        4. 调用函数dvmChangeStatus将当前线程的状态设置为正在执行NATIVE代码，并且将面所创建和初始化好的JavaVMExt对象和JNIEnvExt对象通过输出参数p_vm和p_env返回给调用者。

​        gDvm是一个类型为DvmGlobals的全局变量，用来收集当前进程所有虚拟机相关的信息，其中，它的成员变量vmList指向的就是当前进程中的Dalvik虚拟机实例，即一个JavaVMExt对象。以后每当需要访问当前进程中的Dalvik虚拟机实例时，就可以通过全局变量gDvm的成员变量vmList来获得，避免了在函数之间传递该Dalvik虚拟机实例。

​        每一个Dalvik虚拟机实例都有一个函数表，保存在对应的JavaVMExt对象的成员变量funcTable中，而这个函数表又被指定为gInvokeInterface。gInvokeInterface是一个类型为JNIInvokeInterface的结构体，它定义在文件dalvik/vm/Jni.c中，如下所示：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8885792#) [copy](http://blog.csdn.net/luoshengyang/article/details/8885792#)

1. static const struct JNIInvokeInterface gInvokeInterface = {  
2. ​    NULL,  
3. ​    NULL,  
4. ​    NULL,  
5.   
6. ​    DestroyJavaVM,  
7. ​    AttachCurrentThread,  
8. ​    DetachCurrentThread,  
9.   
10. ​    GetEnv,  
11.   
12. ​    AttachCurrentThreadAsDaemon,  
13. };  

​        有了这个Dalvik虚拟机函数表之后，我们就可以将当前线程Attach或者Detach到Dalvik虚拟机中去，或者销毁当前进程的Dalvik虚拟机等。

​        每一个Dalvik虚拟机实例还有一个JNI环境列表，保存在对应的JavaVMExt对象的成员变量envList中。注意，JavaVMExt对象的成员变量envList描述的是一个JNIEnvExt列表，其中，每一个Attach到Dalvik虚拟机中去的线程都有一个对应的JNIEnvExt，用来描述它的JNI环境。有了这个JNI环境之后，我们才可以在Java函数和C/C++函数之间互相调用。

​        每一个JNIEnvExt对象都有两个成员变量prev和next，它们均是一个JNIEnvExt指针，分别指向前一个JNIEnvExt对象和后一个JNIEnvExt对象，也就是说，每一个Dalvik虚拟机实例的成员变量envList描述的是一个双向JNIEnvExt列表，其中，列表中的第一个JNIEnvExt对象描述的是主线程的JNI环境。

​        上述提到的DvmGlobals结构体定义文件dalvik/vm/Globals.h中，JNIInvokeInterface结构体定义在文件dalvik/libnativehelper/include/nativehelper/jni.h中，JavaVMExt和JNIEnvExt结构体定义在文件dalvik/vm/JniInternal.h中。

​        接下来，我们接下来就继续分析函数dvmCreateJNIEnv和函数dvmStartup的实现，以便可以了解JNI环境的创建和初始化过程，以及Dalvik虚拟机的虚拟机初始化过程。

​        Step 4. dvmCreateJNIEnv

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8885792#) [copy](http://blog.csdn.net/luoshengyang/article/details/8885792#)

1. /* 
2.  * Create a new JNIEnv struct and add it to the VM's list. 
3.  * 
4.  * "self" will be NULL for the main thread, since the VM hasn't started 
5.  * yet; the value will be filled in later. 
6.  */  
7. JNIEnv* dvmCreateJNIEnv(Thread* self)  
8. {  
9. ​    JavaVMExt* vm = (JavaVMExt*) gDvm.vmList;  
10. ​    JNIEnvExt* newEnv;  
11.   
12. ​    ......  
13.   
14. ​    newEnv = (JNIEnvExt*) calloc(1, sizeof(JNIEnvExt));  
15. ​    newEnv->funcTable = &gNativeInterface;  
16. ​    newEnv->vm = vm;  
17.   
18. ​    ......  
19.   
20. ​    if (self != NULL) {  
21. ​        dvmSetJniEnvThreadId((JNIEnv*) newEnv, self);  
22. ​        assert(newEnv->envThreadId != 0);  
23. ​    } else {  
24. ​        /* make it obvious if we fail to initialize these later */  
25. ​        newEnv->envThreadId = 0x77777775;  
26. ​        newEnv->self = (Thread*) 0x77777779;  
27. ​    }  
28.   
29. ​    ......  
30.   
31. ​    /* insert at head of list */  
32. ​    newEnv->next = vm->envList;  
33. ​    assert(newEnv->prev == NULL);  
34. ​    if (vm->envList == NULL)            // rare, but possible  
35. ​        vm->envList = newEnv;  
36. ​    else  
37. ​        vm->envList->prev = newEnv;  
38. ​    vm->envList = newEnv;  
39.   
40. ​    ......  
41.   
42. ​    return (JNIEnv*) newEnv;  
43. }  

​        这个函数定义在文件dalvik/vm/Jni.c中。

​        函数dvmCreateJNIEnv主要是执行了以下三个操作：

​        1. 创建一个JNIEnvExt对象，用来描述一个JNI环境，并且设置这个JNIEnvExt对象的宿主Dalvik虚拟机，以及所使用的本地接口表，即设置这个JNIEnvExt对象的成员变量funcTable和vm。这里的宿主Dalvik虚拟机即为当前进程的Dalvik虚拟机，它保存在全局变量gDvm的成员变量vmList中。本地接口表由全局变量gNativeInterface来描述。

​        2. 参数self描述的是前面创建的JNIEnvExt对象要关联的线程，可以通过调用函数dvmSetJniEnvThreadId来将它们关联起来。注意，当参数self的值等于NULL的时候，就表示前面的JNIEnvExt对象是要与主线程关联的，但是要等到后面再关联，因为现在用来描述主线程的Thread对象还没有准备好。通过将一个JNIEnvExt对象的成员变量envThreadId和self的值分别设置为0x77777775和0x77777779来表示它还没有与线程关联。

​        3. 在一个Dalvik虚拟机里面，可以运行多个线程。所有关联有JNI环境的线程都有一个对应的JNIEnvExt对象，这些JNIEnvExt对象相互连接在一起保存在用来描述其宿主Dalvik虚拟机的一个JavaVMExt对象的成员变量envList中。因此，前面创建的JNIEnvExt对象需要连接到其宿主Dalvik虚拟机的JavaVMExt链表中去。

​        gNativeInterface是一个类型为JNINativeInterface的结构体，它定义在文件dalvik/vm/Jni.c，如下所示：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8885792#) [copy](http://blog.csdn.net/luoshengyang/article/details/8885792#)

1. static const struct JNINativeInterface gNativeInterface = {  
2. ​    ......  
3. ​      
4. ​    FindClass,  
5.   
6. ​    ......  
7.   
8. ​    GetMethodID,  
9. ​      
10. ​    ......  
11.   
12. ​    CallObjectMethod,  
13. ​      
14. ​    ......  
15.   
16. ​    GetFieldID,  
17.   
18. ​    ......  
19. ​      
20. ​    SetIntField,  
21. ​      
22. ​    ......  
23.   
24. ​    RegisterNatives,  
25. ​    UnregisterNatives,  
26.   
27. ​    ......  
28.   
29. ​    GetJavaVM,  
30.   
31. ​    ......  
32. };  

​         从gNativeInterface的定义就可以看出，结构体JNINativeInterface用来描述一个本地接口表。当我们需要在C/C++代码在中调用Java函数，就要用到这个本地接口表，例如：

​         1. 调用函数FindClass可以找到指定的Java类；

​         2. 调用函数GetMethodID可以获得一个Java类的成员函数，并且可以通过类似CallObjectMethod函数来间接调用它；

​         3. 调用函数GetFieldID可以获得一个Java类的成员变量，并且可以通过类似SetIntField的函数来设置它的值；

​         4. 调用函数RegisterNatives和UnregisterNatives可以注册和反注册JNI方法到一个Java类中去，以便可以在Java函数中调用；

​         5. 调用函数GetJavaVM可以获得当前进程中的Dalvik虚拟机实例。

​         事实上，结构体JNINativeInterface定义的可以在C/C++代码中调用的函数非常多，具体可以参考它在dalvik\libnativehelper\include\nativehelper\jni.h文件中的定义。

​         这一步执行完成之后，返回到前面的Step 3中，即函数JNI_CreateJavaVM中，接下来就会继续调用函数dvmStartup来初始化前面所创建的Dalvik虚拟机实例。

​         Step 5. dvmStartup

​         这个函数定义在文件dalvik/vm/Init.c中，用来初始化Dalvik虚拟机，我们分段来阅读：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8885792#) [copy](http://blog.csdn.net/luoshengyang/article/details/8885792#)

1. /* 
2.  * VM initialization.  Pass in any options provided on the command line. 
3.  * Do not pass in the class name or the options for the class. 
4.  * 
5.  * Returns 0 on success. 
6.  */  
7. int dvmStartup(int argc, const char* const argv[], bool ignoreUnrecognized,  
8. ​    JNIEnv* pEnv)  
9. {  
10. ​    int i, cc;  
11.   
12. ​    ......  
13.   
14. ​    setCommandLineDefaults();  
15.   
16. ​    /* prep properties storage */  
17. ​    if (!dvmPropertiesStartup(argc))  
18. ​        goto fail;  
19.   
20. ​    /* 
21. ​     * Process the option flags (if any). 
22. ​     */  
23. ​    cc = dvmProcessOptions(argc, argv, ignoreUnrecognized);  
24. ​    if (cc != 0) {  
25. ​        ......  
26. ​        goto fail;  
27. ​    }  

​        这段代码用来处理Dalvik虚拟机的启动选项，这些启动选项保存在参数argv中，并且个数等于argc。在处理这些启动选项之前，还会执行以下两个操作：

​        1. 调用函数setCommandLineDefaults来给Dalvik虚拟机设置默认参数，因为启动选项不一定会指定Dalvik虚拟机的所有属性。

​        2. 调用函数dvmPropertiesStartup来分配足够的内存空间来容纳由参数argv和argc所描述的启动选项。

​        完成以上两个操作之后，就可以调用函数dvmProcessOptions来处理参数argv和argc所描述的启动选项了，也就是根据这些选项值来设置Dalvik虚拟机的属性，例如，设置Dalvik虚拟机的Java对象堆的最大值。

​        在上述代码中，函数setCommandLineDefaults和dvmPropertiesStartup定义在文件dalvik/vm/Init.c中，函数dvmPropertiesStartup定义在文件dalvik/vm/Properties.c中。

​         我们继续往下阅读代码：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8885792#) [copy](http://blog.csdn.net/luoshengyang/article/details/8885792#)

1. /* configure signal handling */  
2. if (!gDvm.reduceSignals)  
3. ​    blockSignals();  

​         如果我们没有在Dalvik虚拟机的启动选项中指定-Xrs，那么gDvm.reduceSignals的值就会被设置为false，表示要在当前线程中屏蔽掉SIGQUIT信号。在这种情况下，会有一个线程专门用来处理SIGQUIT信号。这个线程在接收到SIGQUIT信号的时候，就会将各个线程的调用堆栈打印出来，因此，这个线程又称为dump-stack-trace线程。

​         屏蔽当前线程的SIGQUIT信号是通过调用函数blockSignals来实现的，这个函数定义在文件dalvik/vm/Init.c中。

​         我们继续往下阅读代码：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8885792#) [copy](http://blog.csdn.net/luoshengyang/article/details/8885792#)

1. /* 
2.  * Initialize components. 
3.  */  
4. if (!dvmAllocTrackerStartup())  
5. ​    goto fail;  
6. if (!dvmGcStartup())  
7. ​    goto fail;  
8. if (!dvmThreadStartup())  
9. ​    goto fail;  
10. if (!dvmInlineNativeStartup())  
11. ​    goto fail;  
12. if (!dvmVerificationStartup())  
13. ​    goto fail;  
14. if (!dvmRegisterMapStartup())  
15. ​    goto fail;  
16. if (!dvmInstanceofStartup())  
17. ​    goto fail;  
18. if (!dvmClassStartup())  
19. ​    goto fail;  
20. if (!dvmThreadObjStartup())  
21. ​    goto fail;  
22. if (!dvmExceptionStartup())  
23. ​    goto fail;  
24. if (!dvmStringInternStartup())  
25. ​    goto fail;  
26. if (!dvmNativeStartup())  
27. ​    goto fail;  
28. if (!dvmInternalNativeStartup())  
29. ​    goto fail;  
30. if (!dvmJniStartup())  
31. ​    goto fail;  
32. if (!dvmReflectStartup())  
33. ​    goto fail;  
34. if (!dvmProfilingStartup())  
35. ​    goto fail;  

​        这段代码用来初始化Dalvik虚拟机的各个子模块，接下来我们就分别描述。

​        1. dvmAllocTrackerStartup

​        这个函数定义在文件dalvik/vm/AllocTracker.c中，用来初始化Davlik虚拟机的对象分配记录子模块，这样我们就可以通过DDMS工具来查看Davlik虚拟机的对象分配情况。

​        2. dvmGcStartup

​        这个函数定义在文件dalvik/vm/alloc/Alloc.c中，用来初始化Davlik虚拟机的垃圾收集（ GC）子模块。

​        3. dvmThreadStartup

​        这个函数定义在文件dalvik/vm/Thread.c中，用来初始化Davlik虚拟机的线程列表、为主线程创建一个Thread对象以及为主线程初始化执行环境。Davlik虚拟机中的所有线程均是本地[操作系统](http://lib.csdn.net/base/operatingsystem)线程。在[linux](http://lib.csdn.net/base/linux)系统中，一般都是使用pthread库来创建和管理线程的，Android系统也不例外，也就是说，Davlik虚拟机中的每一个线程均是一个pthread线程。注意，Davlik虚拟机中的每一个线程均用一个Thread结构体来描述，这些Thread结构体组织在一个列表中，因此，这里要先对它进行初始化。

​        4. dvmInlineNativeStartup

​        这个函数定义在文件dalvik/vm/InlineNative.c中，用来初始化Davlik虚拟机的内建Native函数表。这些内建Native函数主要是针对java.Lang.String、java.Lang.Math、java.Lang.Float和java.Lang.Double类的，用来替换这些类的某些成员函数原来的实现（包括Java实现和Native实现）。例如，当我们调用java.Lang.String类的成员函数compareTo来比较两个字符串的大小时，实际执行的是由Davlik虚拟机提供的内建函数javaLangString_compareTo（同样是定义在文件dalvik/vm/InlineNative.c中）。在提供有__memcmp16函数的系统中，函数javaLangString_compareTo会利用它来直接比较两个字符串的大小。由于函数__memcmp16是用优化过的汇编语言的来实现的，它的效率会更高。

​        5. dvmVerificationStartup

​        这个函数定义在文件dalvik/vm/analysis/DexVerify.c中，用来初始化Dex文件验证器。Davlik虚拟机与Java虚拟机一样，在加载一个类文件的时候，一般需要验证它的合法性，也就是验证文件中有没有非法的指令或者操作等。

​        6. dvmRegisterMapStartup

​        这个函数定义在文件dalvik/vm/analysis/RegisterMap.c中，用来初始化寄存器映射集（Register Map）子模块。Davlik虚拟机支持精确垃圾收集（Exact GC或者Precise GC），也就是说，在进行垃圾收集的时候，Davlik虚拟机可以准确地判断当前正在使用的每一个寄存器里面保存的是对象引用还是非对象引用。对于对象引用，意味被引用的对象现在还不可以回收，因此，就可以进行精确的垃圾收集。

​        为了帮助垃圾收集器准备地判断寄存器保存的是对象引用还是非对象引用，Davlik虚拟机在验证了一个类之后，还会为它的每一个成员函数生成一个寄存器映射集。寄存器映射集记录了类成员函数在每一个GC安全点（Safe Point）中的寄存器使用情况，也就是记录每一个寄存器里面保存的是对象引用还是非对象引用。由于垃圾收集器一定是在GC安全点进行垃圾收集的，因此，根据每一个GC安全点的寄存器映射集，就可以准确地知道对象的引用情况，从而可以确定哪些可以回收，哪些对象还不可以回收。

​        7. dvmInstanceofStartup

​        这个函数定义在文件dalvik/vm/oo/TypeCheck.c中，用来初始化instanceof操作符子模块。在使用instanceof操作符来判断一个对象A是否是一个类B的实例时，Davlik虚拟机需要检查类B是否是从对象A的声明类继承下来的。由于这个检查的过程比较耗时，Davlik虚拟机在内部使用一个缓冲，用来记录第一次两个类之间的instanceof操作结果，这样后面再碰到相同的instanceof操作时，就可以快速地得到结果。

​        8. dvmClassStartup

​        这个函数定义在文件dalvik/vm/oo/Class.c中，用来初始化启动类加载器（Bootstrap Class Loader），同时还会初始化java.lang.Class类。启动类加载器是用来加载Java核心类的，用来保证安全性，即保证加载的Java核心类是合法的。

​        9. dvmThreadObjStartup

​        这个函数定义在文件dalvik/vm/Thread.c中，用来加载与线程相关的类，即java.lang.Thread、java.lang.VMThread和java.lang.ThreadGroup。

​        10. dvmExceptionStartup

​        这个函数定义在文件dalvik/vm/Exception.c中，用来加载与异常相关的类，即java.lang.Throwable、java.lang.RuntimeException、java.lang.StackOverflowError、java.lang.Error、java.lang.StackTraceElement和java.lang.StackTraceElement类。

​        11. dvmStringInternStartup

​        这个函数定义在文件dalvik/vm/Intern.c中，用来初始化java.lang.String类内部私有一个字符串池，这样当Dalvik虚拟机运行起来之后，我们就可以调用java.lang.String类的成员函数intern来访问这个字符串池里面的字符串。

​        12. dvmNativeStartup

​        这个函数定义在文件dalvik/vm/Native.c中，用来初始化Native Shared Object库加载表，也就是SO库加载表。这个加载表是用来描述当前进程有哪些SO文件已经被加载过了。

​        13. dvmInternalNativeStartup

​        这个函数定义在文件dalvik/vm/native/InternalNative.c中，用来初始化一个内部Native函数表。所有需要直接访问Dalvik虚拟机内部函数或者[数据结构](http://lib.csdn.net/base/datastructure)的Native函数都定义在这张表中，因为它们如果定义在外部的其它SO文件中，就无法直接访问Dalvik虚拟机的内部函数或者数据结构。例如，前面提到的java.lang.String类的成员函数intent，由于它要访问Dalvik虚拟机内部的一个私有字符串池，因此，它所对应的Native函数就要在Dalvik虚拟机内部实现。

​        14. dvmJniStartup

​        这个函数定义在文件dalvik/vm/Jni.c中，用来初始化全局引用表，以及加载一些与Direct Buffer相关的类，如DirectBuffer、PhantomReference和ReferenceQueue等。 

​        我们在一个JNI方法中，可能会需要访问一些Java对象，这样就需要通知GC，这些Java对象现在正在被Native Code引用，不能回收。这些被Native Code引用的Java对象就会被记录在一个全局引用表中，具体的做法就是调用JNI环境对象（JNIEnv）的成员函数NewLocalRef/DeleteLocalRef和NewGlobalRef/DeleteGlobalRef等来显式地引用或者释放Java对象。

​        有时候我们需要在Java代码中，直接在Native层分配内存，也就直接使用malloc来分配内存。这些Native内存不同于在Java堆中分配的内存，区别在于前者需要不接受GC管理，而后者接受GC管理。这些直接在Native层分配的内存有什么用呢？考虑一个场景，我们需要在Java代码中从一个IO设备中读取数据。从IO设备读取数据意味着要调用由本地操作系统提供的read接口来实现。这样我们就有两种做法。第一种做法在Native层临时分配一个缓冲区，用来保存从IO设备read回来的数据，然后再将这个数据拷贝到Java层中去，也就是拷贝到Java堆去使用。第二种做法是在Java层创建一个对象，这个对象在Native层直接关联有一块内存，从IO设备read回来的数据就直接保存这块内存中。第二种方法和第一种方法相比，减少了一次内存拷贝，因而可以提高性能。

​        我们将这种能够直接在Native层中分配内存的Java对象就称为DirectBuffer。由于DirectBuffer使用的内存是不接受GC管理的，因此，我们就需要通过其它的方式来管理它们。具体做法就是为每一个DirectBuffer对象创建一个PhantomReference引用。注意，DirectBuffer对象本身是一个Java对象，它是接受GC管理的。当GC准备回收一个DirectBuffer对象时，如果发现它还有PhantomReference引用，那就会在回收它之前，把相应的PhantomReference引用加入到与之关联的一个ReferenceQueue队列中去。这样我们就可以通过判断一个DirectBuffer对象的PhantomReference引用是否已经加入到一个相关的ReferenceQueue队列中。如果已经加入了的话，那么就可以在该DirectBuffer对象被回收之前，释放掉之前为它在Native层分配的内存。

​        15. dvmReflectStartup

​         这个函数定义在文件dalvik/vm/reflect/Reflect.c中，用来加载反射相关的类，如java.lang.reflect.AccessibleObject、java.lang.reflect.Constructor、java.lang.reflect.Field、java.lang.reflect.Method和java.lang.reflect.Proxy等。

​         16. dvmProfilingStartup

​         这个函数定义在文件dalvik/vm/Profile.c，用来初始化Dalvik虚拟机的性能分析子模块，以及加载dalvik.system.VMDebug类等。

​         Dalvik虚拟机的各个子模块初始化完成之后，我们继续往下阅读代码：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8885792#) [copy](http://blog.csdn.net/luoshengyang/article/details/8885792#)

1. /* make sure we got these [can this go away?] */  
2. assert(gDvm.classJavaLangClass != NULL);  
3. assert(gDvm.classJavaLangObject != NULL);  
4. //assert(gDvm.classJavaLangString != NULL);  
5. assert(gDvm.classJavaLangThread != NULL);  
6. assert(gDvm.classJavaLangVMThread != NULL);  
7. assert(gDvm.classJavaLangThreadGroup != NULL);  
8.   
9. /* 
10.  * Make sure these exist.  If they don't, we can return a failure out 
11.  * of main and nip the whole thing in the bud. 
12.  */  
13. static const char* earlyClasses[] = {  
14. ​    "Ljava/lang/InternalError;",  
15. ​    "Ljava/lang/StackOverflowError;",  
16. ​    "Ljava/lang/UnsatisfiedLinkError;",  
17. ​    "Ljava/lang/NoClassDefFoundError;",  
18. ​    NULL  
19. };  
20. const char** pClassName;  
21. for (pClassName = earlyClasses; *pClassName != NULL; pClassName++) {  
22. ​    if (dvmFindSystemClassNoInit(*pClassName) == NULL)  
23. ​        goto fail;  
24. }  

​        这段代码检查java.lang.Class、java.lang.Object、java.lang.Thread、java.lang.VMThread和java.lang.ThreadGroup这五个核心类经过前面的初始化操作后已经得到加载，并且确保系统中存在java.lang.InternalError、java.lang.StackOverflowError、java.lang.UnsatisfiedLinkError和java.lang.NoClassDefFoundError这四个核心类。

​         我们继续往下阅读代码：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8885792#) [copy](http://blog.csdn.net/luoshengyang/article/details/8885792#)

1. /* 
2.  * Miscellaneous class library validation. 
3.  */  
4. if (!dvmValidateBoxClasses())  
5. ​    goto fail;  
6.   
7. /* 
8.  * Do the last bits of Thread struct initialization we need to allow 
9.  * JNI calls to work. 
10.  */  
11. if (!dvmPrepMainForJni(pEnv))  
12. ​    goto fail;  
13.   
14. /* 
15.  * Register the system native methods, which are registered through JNI. 
16.  */  
17. if (!registerSystemNatives(pEnv))  
18. ​    goto fail;  
19.   
20. /* 
21.  * Do some "late" initialization for the memory allocator.  This may 
22.  * allocate storage and initialize classes. 
23.  */  
24. if (!dvmCreateStockExceptions())  
25. ​    goto fail;  
26.   
27. /* 
28.  * At this point, the VM is in a pretty good state.  Finish prep on 
29.  * the main thread (specifically, create a java.lang.Thread object to go 
30.  * along with our Thread struct).  Note we will probably be executing 
31.  * some interpreted class initializer code in here. 
32.  */  
33. if (!dvmPrepMainThread())  
34. ​    goto fail;  
35.   
36. /* 
37.  * Make sure we haven't accumulated any tracked references.  The main 
38.  * thread should be starting with a clean slate. 
39.  */  
40. if (dvmReferenceTableEntries(&dvmThreadSelf()->internalLocalRefTable) != 0)  
41. {  
42. ​    LOGW("Warning: tracked references remain post-initialization\n");  
43. ​    dvmDumpReferenceTable(&dvmThreadSelf()->internalLocalRefTable, "MAIN");  
44. }  
45.   
46. /* general debugging setup */  
47. if (!dvmDebuggerStartup())  
48. ​    goto fail;  

​        这段代码继续执行其它函数来执行其它的初始化和检查工作，如下所示：

​        1. dvmValidateBoxClasses

​        这个函数定义在文件dalvik/vm/reflect/Reflect.c中，用来验证Dalvik虚拟机中存在相应的装箱类，并且这些装箱类有且仅有一个成员变量，这个成员变量是用来描述对应的数字值的。这些装箱类包括java.lang.Boolean、java.lang.Character、java.lang.Float、java.lang.Double、java.lang.Byte、java.lang.Short、java.lang.Integer和java.lang.Long。

​        所谓装箱，就是可以自动将一个数值转换一个对象，例如，将数字1自动转换为一个java.lang.Integer对象。相应地，也要求能将一个装箱类对象转换成一个数字，例如，将一个值等于1的java.lang.Integer对象转换为数字1。

​        2. dvmPrepMainForJni

​        这个函数定义在文件dalvik/vm/Thread.c中，用来准备主线程的JNI环境，即将在前面的Step 5中为主线程创建的Thread对象与在前面Step 4中创建的JNI环境关联起来。回忆在前面的Step 4中，虽然我们已经为当前线程创建好一个JNI环境了，但是还没有将该JNI环境与主线程关联，也就是还没有将主线程的ID设置到该JNI环境中去。

​        3. registerSystemNatives

​        这个函数定义在文件dalvik/vm/Init.c中，它调用另外一个函数jniRegisterSystemMethods，后者接着又调用了函数registerCoreLibrariesJni来为Java核心类注册JNI方法。函数registerCoreLibrariesJni定义在文件libcore/luni/src/main/native/Register.cpp中。

​        4. dvmCreateStockExceptions

​        这个函数定义在文件dalvik/vm/alloc/Alloc.c中，用来预创建一些与内存分配有关的异常对象，并且将它们缓存起来，以便以后可以快速使用。这些异常对象包括java.lang.OutOfMemoryError、java.lang.InternalError和java.lang.NoClassDefFoundError。

​        5. dvmPrepMainThread

​        这个函数定义在文件dalvik/vm/Thread.c中，用来为主线程创建一个java.lang.ThreadGroup对象、一个java.lang.Thread对角和java.lang.VMThread对象。这些Java对象和在前面Step 5中创建的C++层Thread对象关联一起，共同用来描述Dalvik虚拟机的主线程。

​        6. dvmReferenceTableEntries

​        这个函数定义在文件dalvik/vm/ReferenceTable.h中，用来确保主线程当前不引用有任何Java对象，这是为了保证主线程接下来以干净的方式来执行程序入口。

​        7. dvmDebuggerStartup

​        这个函数定义在文件dalvik/vm/Debugger.c中，用来初始化Dalvik虚拟机的调试环境。注意，Dalvik虚拟机与Java虚拟机一样，都是通过JDWP协议来支持远程调试的。

​        上述初始化和检查操作执行完成之后，我们再来看最后一段代码：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8885792#) [copy](http://blog.csdn.net/luoshengyang/article/details/8885792#)

1. ​    /* 
2. ​     * Init for either zygote mode or non-zygote mode.  The key difference 
3. ​     * is that we don't start any additional threads in Zygote mode. 
4. ​     */  
5. ​    if (gDvm.zygote) {  
6. ​        if (!dvmInitZygote())  
7. ​            goto fail;  
8. ​    } else {  
9. ​        if (!dvmInitAfterZygote())  
10. ​            goto fail;  
11. ​    }  
12.   
13. ​    ......  
14.   
15. ​    return 0;  
16.   
17. fail:  
18. ​    dvmShutdown();  
19. ​    return 1;  
20. }  

​        这段代码完成Dalvik虚拟机的最后一步初始化工作。它检查Dalvik虚拟机是否指定了-Xzygote启动选项。如果指定了的话，那么就说明当前是在Zygote进程中启动Dalvik虚拟机，因此，接下来就会调用函数dvmInitZygote来执行最后一步初始化工作。否则的话，就会调用另外一个函数dvmInitAfterZygote来执行最后一步初始化工作。

​        由于当前是在Zygote进程中启动Dalvik虚拟机的，因此，接下来我们就继续分析函数dvmInitZygote的实现。在接下来的文章中分析Android应用程序进程的创建过程时，我们再分析函数dvmInitAfterZygote的实现。

​        Step 6. dvmInitZygote

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8885792#) [copy](http://blog.csdn.net/luoshengyang/article/details/8885792#)

1. /* 
2.  * Do zygote-mode-only initialization. 
3.  */  
4. static bool dvmInitZygote(void)  
5. {  
6. ​    /* zygote goes into its own process group */  
7. ​    setpgid(0,0);  
8.   
9. ​    return true;  
10. }  

​        这个函数定义在文件dalvik/vm/Init.c中。

​        函数dvmInitZygote的实现很简单，它只是调用了系统调用setpgid来设置当前进程，即Zygote进程的进程组ID。注意，在调用setpgid的时候，传递进去的两个参数均为0，这意味着Zygote进程的进程组ID与进程ID是相同的，也就是说，Zygote进程运行在一个单独的进程组里面。

​        这一步执行完成之后，Dalvik虚拟机的创建和初始化工作就完成了，回到前面的Step 1中，即AndroidRuntime类的成员函数start中，接下来就会调用AndroidRuntime类的另外一个成员函数startReg来注册Android核心类的JNI方法。

​        Step 7. AndroidRuntime.startReg

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8885792#) [copy](http://blog.csdn.net/luoshengyang/article/details/8885792#)

1. /* 
2.  * Register android native functions with the VM. 
3.  */  
4. /*static*/ int AndroidRuntime::startReg(JNIEnv* env)  
5. {  
6. ​    /* 
7. ​     * This hook causes all future threads created in this process to be 
8. ​     * attached to the JavaVM.  (This needs to go away in favor of JNI 
9. ​     * Attach calls.) 
10. ​     */  
11. ​    androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);  
12.   
13. ​    LOGV("--- registering native functions ---\n");  
14.   
15. ​    /* 
16. ​     * Every "register" function calls one or more things that return 
17. ​     * a local reference (e.g. FindClass).  Because we haven't really 
18. ​     * started the VM yet, they're all getting stored in the base frame 
19. ​     * and never released.  Use Push/Pop to manage the storage. 
20. ​     */  
21. ​    env->PushLocalFrame(200);  
22.   
23. ​    if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0) {  
24. ​        env->PopLocalFrame(NULL);  
25. ​        return -1;  
26. ​    }  
27. ​    env->PopLocalFrame(NULL);  
28.   
29. ​    //createJavaThread("fubar", quickTest, (void*) "hello");  
30.   
31. ​    return 0;  
32. }  

​        这个函数定义在文件frameworks/base/core/jni/AndroidRuntime.cpp中。

​        AndroidRuntime类的成员函数startReg首先调用函数androidSetCreateThreadFunc来设置一个线程创建钩子javaCreateThreadEtc。这个线程创建钩子是用来初始化一个Native线程的JNI环境的，也就是说，当我们在C++代码中创建一个Native线程的时候，函数javaCreateThreadEtc会被调用来初始化该Native线程的JNI环境。后面在分析Dalvik虚拟机线程的创建过程时，我们再详细分析函数javaCreateThreadEtc的实现。

​        AndroidRuntime类的成员函数startReg接着调用函数register_jni_procs来注册Android核心类的JNI方法。在注册JNI方法的过程中，需要在Native代码中引用到一些Java对象，这些Java对象引用需要记录在当前线程的一个Native堆栈中。但是此时Dalvik虚拟机还没有真正运行起来，也就是当前线程的Native堆栈还没有准备就绪。在这种情况下，就需要在注册JNI方法之前，手动地将在当前线程的Native堆栈中压入一个帧（Frame），并且在注册JNI方法之后，手动地将该帧弹出来。

​        当前线程的JNI环境是由参数env所指向的一个JNIEnv对象来描述的，通过调用它的成员函数PushLocalFrame和PopLocalFrame就可以手动地往当前线程的Native堆栈压入和弹出一个帧。注意，这个帧是一个本地帧，只可以用来保存Java对象在Native代码中的本地引用。

​        函数register_jni_procs的实现如下所示：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8885792#) [copy](http://blog.csdn.net/luoshengyang/article/details/8885792#)

1. static int register_jni_procs(const RegJNIRec array[], size_t count, JNIEnv* env)  
2. {  
3. ​    for (size_t i = 0; i < count; i++) {  
4. ​        if (array[i].mProc(env) < 0) {  
5. ​            return -1;  
6. ​        }  
7. ​    }  
8. ​    return 0;  
9. }  

​        这个函数定义在文件frameworks/base/core/jni/AndroidRuntime.cpp中。

​        从前面的调用过程可以知道，参数array指向的是全局变量gRegJNI所描述的一个JNI方法注册函数表，其中，每一个表项都用一个RegJNIRec对象来描述，而每一个RegJNIRec对象都有一个成员变量mProc，指向一个JNI方法注册函数。通过依次调用这些注册函数，就可以将Android核心类的JNI方法注册到前面的所创建的Dalvik虚拟机中去。

​        通过观察全局变量gRegJNI所描述的JNI方法注册函数表，我们就可以看出注册了哪些Android核心类的JNI方法，如下所示：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8885792#) [copy](http://blog.csdn.net/luoshengyang/article/details/8885792#)

1. static const RegJNIRec gRegJNI[] = {  
2. ​    REG_JNI(register_android_debug_JNITest),  
3. ​    REG_JNI(register_com_android_internal_os_RuntimeInit),  
4. ​    REG_JNI(register_android_os_SystemClock),  
5. ​    REG_JNI(register_android_util_EventLog),  
6. ​    REG_JNI(register_android_util_Log),  
7. ​    REG_JNI(register_android_util_FloatMath),  
8. ​    REG_JNI(register_android_text_format_Time),  
9. ​    REG_JNI(register_android_pim_EventRecurrence),  
10. ​    REG_JNI(register_android_content_AssetManager),  
11. ​    REG_JNI(register_android_content_StringBlock),  
12. ​    REG_JNI(register_android_content_XmlBlock),  
13. ​    REG_JNI(register_android_emoji_EmojiFactory),  
14. ​    REG_JNI(register_android_security_Md5MessageDigest),  
15. ​    REG_JNI(register_android_text_AndroidCharacter),  
16. ​    REG_JNI(register_android_text_AndroidBidi),  
17. ​    REG_JNI(register_android_text_KeyCharacterMap),  
18. ​    REG_JNI(register_android_os_Process),  
19. ​    REG_JNI(register_android_os_Binder),  
20. ​    REG_JNI(register_android_view_Display),  
21. ​    REG_JNI(register_android_nio_utils),  
22. ​    REG_JNI(register_android_graphics_PixelFormat),  
23. ​    REG_JNI(register_android_graphics_Graphics),  
24. ​    REG_JNI(register_android_view_Surface),  
25. ​    REG_JNI(register_android_view_ViewRoot),  
26. ​    REG_JNI(register_com_google_android_gles_jni_EGLImpl),  
27. ​    REG_JNI(register_com_google_android_gles_jni_GLImpl),  
28. ​    REG_JNI(register_android_opengl_jni_GLES10),  
29. ​    REG_JNI(register_android_opengl_jni_GLES10Ext),  
30. ​    REG_JNI(register_android_opengl_jni_GLES11),  
31. ​    REG_JNI(register_android_opengl_jni_GLES11Ext),  
32. ​    REG_JNI(register_android_opengl_jni_GLES20),  
33.   
34. ​    REG_JNI(register_android_graphics_Bitmap),  
35. ​    REG_JNI(register_android_graphics_BitmapFactory),  
36. ​    REG_JNI(register_android_graphics_BitmapRegionDecoder),  
37. ​    REG_JNI(register_android_graphics_Camera),  
38. ​    REG_JNI(register_android_graphics_Canvas),  
39. ​    REG_JNI(register_android_graphics_ColorFilter),  
40. ​    REG_JNI(register_android_graphics_DrawFilter),  
41. ​    REG_JNI(register_android_graphics_Interpolator),  
42. ​    REG_JNI(register_android_graphics_LayerRasterizer),  
43. ​    REG_JNI(register_android_graphics_MaskFilter),  
44. ​    REG_JNI(register_android_graphics_Matrix),  
45. ​    REG_JNI(register_android_graphics_Movie),  
46. ​    REG_JNI(register_android_graphics_NinePatch),  
47. ​    REG_JNI(register_android_graphics_Paint),  
48. ​    REG_JNI(register_android_graphics_Path),  
49. ​    REG_JNI(register_android_graphics_PathMeasure),  
50. ​    REG_JNI(register_android_graphics_PathEffect),  
51. ​    REG_JNI(register_android_graphics_Picture),  
52. ​    REG_JNI(register_android_graphics_PorterDuff),  
53. ​    REG_JNI(register_android_graphics_Rasterizer),  
54. ​    REG_JNI(register_android_graphics_Region),  
55. ​    REG_JNI(register_android_graphics_Shader),  
56. ​    REG_JNI(register_android_graphics_Typeface),  
57. ​    REG_JNI(register_android_graphics_Xfermode),  
58. ​    REG_JNI(register_android_graphics_YuvImage),  
59. ​    REG_JNI(register_com_android_internal_graphics_NativeUtils),  
60. ​    
61. ​    REG_JNI(register_android_database_CursorWindow),  
62. ​    REG_JNI(register_android_database_SQLiteCompiledSql),  
63. ​    REG_JNI(register_android_database_SQLiteDatabase),  
64. ​    REG_JNI(register_android_database_SQLiteDebug),  
65. ​    REG_JNI(register_android_database_SQLiteProgram),  
66. ​    REG_JNI(register_android_database_SQLiteQuery),  
67. ​    REG_JNI(register_android_database_SQLiteStatement),  
68. ​    REG_JNI(register_android_os_Debug),  
69. ​    REG_JNI(register_android_os_FileObserver),  
70. ​    REG_JNI(register_android_os_FileUtils),  
71. ​    REG_JNI(register_android_os_MessageQueue),  
72. ​    REG_JNI(register_android_os_ParcelFileDescriptor),  
73. ​    REG_JNI(register_android_os_Power),  
74. ​    REG_JNI(register_android_os_StatFs),  
75. ​    REG_JNI(register_android_os_SystemProperties),  
76. ​    REG_JNI(register_android_os_UEventObserver),  
77. ​    REG_JNI(register_android_net_LocalSocketImpl),  
78. ​    REG_JNI(register_android_net_NetworkUtils),  
79. ​    REG_JNI(register_android_net_TrafficStats),  
80. ​    REG_JNI(register_android_net_wifi_WifiManager),  
81. ​    REG_JNI(register_android_nfc_NdefMessage),  
82. ​    REG_JNI(register_android_nfc_NdefRecord),  
83. ​    REG_JNI(register_android_os_MemoryFile),  
84. ​    REG_JNI(register_com_android_internal_os_ZygoteInit),  
85. ​    REG_JNI(register_android_hardware_Camera),  
86. ​    REG_JNI(register_android_hardware_SensorManager),  
87. ​    REG_JNI(register_android_media_AudioRecord),  
88. ​    REG_JNI(register_android_media_AudioSystem),  
89. ​    REG_JNI(register_android_media_AudioTrack),  
90. ​    REG_JNI(register_android_media_JetPlayer),  
91. ​    REG_JNI(register_android_media_ToneGenerator),  
92.   
93. ​    REG_JNI(register_android_opengl_classes),  
94. ​    REG_JNI(register_android_bluetooth_HeadsetBase),  
95. ​    REG_JNI(register_android_bluetooth_BluetoothAudioGateway),  
96. ​    REG_JNI(register_android_bluetooth_BluetoothSocket),  
97. ​    REG_JNI(register_android_bluetooth_ScoSocket),  
98. ​    REG_JNI(register_android_server_BluetoothService),  
99. ​    REG_JNI(register_android_server_BluetoothEventLoop),  
100. ​    REG_JNI(register_android_server_BluetoothA2dpService),  
101. ​    REG_JNI(register_android_server_Watchdog),  
102. ​    REG_JNI(register_android_message_digest_sha1),  
103. ​    REG_JNI(register_android_ddm_DdmHandleNativeHeap),  
104. ​    REG_JNI(register_android_backup_BackupDataInput),  
105. ​    REG_JNI(register_android_backup_BackupDataOutput),  
106. ​    REG_JNI(register_android_backup_FileBackupHelperBase),  
107. ​    REG_JNI(register_android_backup_BackupHelperDispatcher),  
108.   
109. ​    REG_JNI(register_android_app_NativeActivity),  
110. ​    REG_JNI(register_android_view_InputChannel),  
111. ​    REG_JNI(register_android_view_InputQueue),  
112. ​    REG_JNI(register_android_view_KeyEvent),  
113. ​    REG_JNI(register_android_view_MotionEvent),  
114.   
115. ​    REG_JNI(register_android_content_res_ObbScanner),  
116. ​    REG_JNI(register_android_content_res_Configuration),  
117. };  

​        上述函数表同样是定义在文件frameworks/base/core/jni/AndroidRuntime.cpp中。

​        回到AndroidRuntime类的成员函数startReg中，接下来我们就继续分析函数androidSetCreateThreadFunc的实现，以便可以了解线程创建钩子javaCreateThreadEtc的注册过程。

​        Step 8. androidSetCreateThreadFunc

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8885792#) [copy](http://blog.csdn.net/luoshengyang/article/details/8885792#)

1. static android_create_thread_fn gCreateThreadFn = androidCreateRawThreadEtc;  
2.   
3. ......  
4.   
5. void androidSetCreateThreadFunc(android_create_thread_fn func)  
6. {  
7. ​    gCreateThreadFn = func;  
8. }  

​        这个函数定义在文件frameworks/base/libs/utils/Threads.cpp中。

​        从这里就可以看到，线程创建钩子javaCreateThreadEtc被保存在一个函数指针gCreateThreadFn中。注意，函数指针gCreateThreadFn默认是指向函数androidCreateRawThreadEtc的，也就是说，如果我们不设置线程创建钩子的话，函数androidCreateRawThreadEtc就是默认使用的线程创建函数。后面在分析Dalvik虚拟机线程的创建过程时，我们再详细分析函数指针gCreateThreadFn是如何使用的。

​        至此，我们就分析完成Dalvik虚拟机在Zygote进程中的启动过程，这个启动过程主要就是完成了以下四个事情：

​        1. 创建了一个Dalvik虚拟机实例；

​        2. 加载了Java核心类及其JNI方法；

​        3. 为主线程的设置了一个JNI环境；

​        4. 注册了Android核心类的JNI方法。

​        换句话说，就是Zygote进程为Android系统准备好了一个Dalvik虚拟机实例，以后Zygote进程在创建Android应用程序进程的时候，就可以将它自身的Dalvik虚拟机实例复制到新创建Android应用程序进程中去，从而加快了Android应用程序进程的启动过程。此外，Java核心类和Android核心类（位于dex文件中），以及它们的JNI方法（位于so文件中），都是以内存映射的方式来读取的，因此，Zygote进程在创建Android应用程序进程的时候，除了可以将自身的Dalvik虚拟机实例复制到新创建的Android应用程序进程之外，还可以与新创建的Android应用程序进程共享Java核心类和Android核心类，以及它们的JNI方法，这样就可以节省内存消耗。

​        同时，我们也应该看到，Zygote进程为了加快Android应用程序进程的启动过程，牺牲了自己的启动速度，因为它需要加载大量的Java核心类，以及注册大量的Android核心类JNI方法。Dalvik虚拟机在加载Java核心类的时候，还需要对它们进行验证以及优化，这些通常都是比较耗时的。又由于Zygote进程是由init进程启动的，也就是说Zygote进程在是开机的时候进行启动的，因此，Zygote进程的牺牲是比较大的。不过毕竟我们在玩手机的时候，很少会关机，也就是很少开机，因此，牺牲Zygote进程的启动速度是值得的，换来的是Android应用程序的快速启动。而且，Android系统为了加快Java类的加载速度，还会想方设法地提前对Dex文件进行验证和优化，这些措施具体参考[Dalvik Optimization and Verification With dexopt](http://www.netmite.com/android/mydroid/dalvik/docs/dexopt.html)一文。

​        学习了Dalvik虚拟机的启动过程之后，在接下来的一篇文章中，我们就继续分析Dalvik虚拟机的运行机制，敬请关注！