我们知道，在[Android](http://lib.csdn.net/base/android)系统中，Dalvik虚拟机是运行[Linux](http://lib.csdn.net/base/linux)内核之上的。如果我们把Dalvik虚拟机看作是一台机器，那么它也有进程和线程的概念。事实上，我们的确是可以在[Java](http://lib.csdn.net/base/java)代码中创建进程和线程，也就是Dalvik虚拟机进程和线程。那么，这些Dalvik虚拟机所创建的进程和线程与其宿主[linux](http://lib.csdn.net/base/linux)内核的进程和线程有什么关系呢？本文将通过Dalvik虚拟机进程和线程的创建过程来回答这个问题。

**老罗的新浪微博：http://weibo.com/shengyangluo，欢迎关注！**

《[android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

​        此外，从前面[Dalvik虚拟机的运行过程分析](http://blog.csdn.net/luoshengyang/article/details/8914953)一文可以知道，Dalvik虚拟机除了可以执行Java代码之外，还可以执行Native代码，也就是C/C++函数。这些C/C++函数在执行的过程中，又可以通过本地[操作系统](http://lib.csdn.net/base/operatingsystem)提供的系统调用来创建本地操作系统进程或者线程，也就是Linux进程和线程。如果在Native代码中创建出来的进程又加载有Dalvik虚拟机，那么它实际上又可以看作是一个Dalvik虚拟机进程。另一方面，如果在Native代码中创建出来的线程能够执行Java代码，那么它实际上又可以看作是一个Dalvik虚拟机线程。

​        这样事情看起来似乎很复杂，因为既有Dalvik虚拟机进程和线程，又有Native操作系统进程和线程，而且它们又可以同时执行Java代码和Native代码。为了理清它们之间的关系，我们将按照以下四个情景来组织本文：

​        1. Dalvik虚拟机进程的创建过程；

​        2. Dalvik虚拟机线程的创建过程；

​        3. 只执行C/C++代码的Native线程的创建过程；

​        4. 能同时执行C/C++代码和Java代码的Native线程的创建过程。

​        对于上述进程和线程，Android系统都分别提供有接口来创建：

​        1. Dalvik虚拟机进程可以通过android.os.Process类的静态成员函数start来创建；

​        2. Dalvik虚拟机线程可以通过java.lang.Thread类的成员函数start来创建；

​        3. 只执行C/C++代码的Native线程可以通过C++类Thread的成员函数run来创建；

​        4. 能同时执行C/C++代码和Java代码的Native线程也可以通过C++类Thread的成员函数run来创建；

​        接下来，我们就按照上述四个情况来分析Dalvik虚拟机进程和线程和Native操作系统进程和线程的关系。

​        一. Dalvik虚拟机进程的创建过程

​        Dalvik虚拟机进程实际上就是通常我们所说的Android应用程序进程。从前面[Android应用程序进程启动过程的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6747696)一文可以知道，Android应用程序进程是由ActivityManagerService服务通过android.os.Process类的静态成员函数start来请求Zygote进程创建的，而Zyogte进程最终又是通过dalvik.system.Zygote类的静态成员函数forkAndSpecialize来创建该Android应用程序进程的。因此，接下来我们就从dalvik.system.Zygote类的静态成员函数forkAndSpecialize开始分析Dalvik虚拟机进程的创建过程，如图1所示：

![img](http://img.blog.csdn.net/20130521020307352)

图1 Dalvik虚拟机进程的创建过程

​       这个过程可以分为3个步骤，接下来我们就详细分析每一个步骤。

​       Step 1. Zygote.forkAndSpecialize

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. public class Zygote {  
2. ​    ......  
3.   
4. ​    native public static int forkAndSpecialize(int uid, int gid, int[] gids,  
5. ​            int debugFlags, int[][] rlimits);  
6.   
7. ​    ......  
8. }  

​        这个函数定义在文件libcore/dalvik/src/main/java/dalvik/system/Zygote.java中。

​        Zygote类的静态成员函数forkAndSpecialize是一个JNI方法，它是由C++层的函数Dalvik_dalvik_system_Zygote_forkAndSpecialize来实现的，如下所示：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. /* native public static int forkAndSpecialize(int uid, int gid, 
2.  *     int[] gids, int debugFlags); 
3.  */  
4. static void Dalvik_dalvik_system_Zygote_forkAndSpecialize(const u4* args,  
5. ​    JValue* pResult)  
6. {  
7. ​    pid_t pid;  
8.   
9. ​    pid = forkAndSpecializeCommon(args, false);  
10.   
11. ​    RETURN_INT(pid);  
12. }  

​        这个函数定义在文件dalvik/vm/native/dalvik_system_Zygote.c中。

​        注意，参数args指向的是一个u4数组，它里面包含了所有从Java层传递进来的参数，这是由Dalvik虚拟机封装的。另外一个参数pResult用来保存JNI方法调用结果，这是通过宏RETURN_INT来实现的。

​        函数Dalvik_dalvik_system_Zygote_forkAndSpecialize的实现很简单，它通过调用另外一个函数forkAndSpecializeCommon来创建一个Dalvik虚拟机进程。

​        Step 2. forkAndSpecializeCommon

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. /* 
2.  * Utility routine to fork zygote and specialize the child process. 
3.  */  
4. static pid_t forkAndSpecializeCommon(const u4* args, bool isSystemServer)  
5. {  
6. ​    pid_t pid;  
7.   
8. ​    uid_t uid = (uid_t) args[0];  
9. ​    gid_t gid = (gid_t) args[1];  
10. ​    ArrayObject* gids = (ArrayObject *)args[2];  
11. ​    u4 debugFlags = args[3];  
12. ​    ArrayObject *rlimits = (ArrayObject *)args[4];  
13. ​    int64_t permittedCapabilities, effectiveCapabilities;  
14.   
15. ​    if (isSystemServer) {  
16. ​        /* 
17. ​         * Don't use GET_ARG_LONG here for now.  gcc is generating code 
18. ​         * that uses register d8 as a temporary, and that's coming out 
19. ​         * scrambled in the child process.  b/3138621 
20. ​         */  
21. ​        //permittedCapabilities = GET_ARG_LONG(args, 5);  
22. ​        //effectiveCapabilities = GET_ARG_LONG(args, 7);  
23. ​        permittedCapabilities = args[5] | (int64_t) args[6] << 32;  
24. ​        effectiveCapabilities = args[7] | (int64_t) args[8] << 32;  
25. ​    } else {  
26. ​        permittedCapabilities = effectiveCapabilities = 0;  
27. ​    }  
28.   
29. ​    if (!gDvm.zygote) {  
30. ​        ......  
31. ​        return -1;  
32. ​    }  
33.   
34. ​    ......  
35.   
36. ​    pid = fork();  
37.   
38. ​    if (pid == 0) {  
39. ​        int err;  
40. ​        ......  
41.   
42. ​        err = setgroupsIntarray(gids);  
43. ​        ......  
44.   
45. ​        err = setrlimitsFromArray(rlimits);   
46. ​        ......  
47.   
48. ​        err = setgid(gid);  
49. ​        ......  
50.   
51. ​        err = setuid(uid);  
52. ​        ......  
53.   
54. ​        err = setCapabilities(permittedCapabilities, effectiveCapabilities);  
55. ​        ......  
56.   
57. ​        enableDebugFeatures(debugFlags);  
58. ​        ......  
59.   
60. ​        gDvm.zygote = false;  
61. ​        if (!dvmInitAfterZygote()) {  
62. ​            ......  
63. ​            dvmAbort();  
64. ​        }  
65. ​    } else if (pid > 0) {  
66. ​        /* the parent process */  
67. ​    }  
68.   
69. ​    return pid;  
70. }  

​        这个函数定义在文件dalvik/vm/native/dalvik_system_Zygote.c中。

​        函数forkAndSpecializeCommon除了可以用来创建普通的Android应用程序进程之外，还用来创建System进程。Android系统中的System进程和普通的Android应用程序进程一样，也是由Zygote进程负责创建的，具体可以参考前面[Android系统进程Zygote启动过程的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6768304)一文。

​        当函数forkAndSpecializeCommon是调用来创建System进程的时候，参数isSystemServer的值就等于true，这时候在参数列表args就会包含两个额外的参数permittedCapabilities和effectiveCapabilities。其中，permittedCapabilities表示System进程允许的特权，而effectiveCapabilities表示System进程当前的有效特权，这两个参数的关系就类似于进程的uid和euid的关系一样。

​        进程的特权是什么概念呢？从Linux 2.2开始，Root用户的权限被划分成了一系列的子权限，每一个子权限都通过一个bit来表示，这些bit的定义可以参考[CAPABILITIES(7)](http://man7.org/linux/man-pages/man7/capabilities.7.html)。每一个进程都关联有一个u32整数，用来描述它所具有的Root用户子权限，我们可以通过系统调用capset来进行设置。

​        参考前面[Android系统进程Zygote启动过程的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6768304)一篇文章可以知道，Zygote进程在创建System进程的时候，给它指定的permittedCapabilities和effectiveCapabilities均为130104352，因此，System进程在运行的时候，就可以获得一些Root用户特权。

​        当参数isSystemServer的值等于false的时候，变量permittedCapabilities和effectiveCapabilities的值被设置为0，也就是说，由Zygote进程创建出来的Android应用程序进程是不具有任何的Root用户特权的。

​        除了上述的permittedCapabilities和effectiveCapabilities之外，参数列表args还包含了其它的参数：

​        **--uid**：要创建的进程的用户ID。一般来说，每一个应用程序进程都有一个唯一的用户ID，用来将应用程序进程封闭在一个沙箱里面运行。

​        **--gid**：要创建的进程的用户组ID。一般来说，每一个应用程序进程都有一个唯一的用户组ID，也是用来将应用程序进程封闭在一个沙箱里面运行。

​        **--gids**：要创建的进程的额外用户组ID。这个额外的用户组ID实际上对应的就是应用程序所申请的资源访权限。

​        **--debugFlags**：要创建的进程在运行时的调试选项。例如，我们可以将debugFlags的DEBUG_ENABLE_CHECKJNI位设置为1，从而打开该进程中的Dalvik虚拟机的JNI检查选项。

​        **--rlimits**：要创建的进程所受到的资源限制。例如，该进程所能打开文件的个数。

​        了解上述参数的含义之后，函数forkAndSpecializeCommon的实现就容易理解了，它主要就是调用系统调用fork来创建一个进程。我们知道系统调用fork执行完成之后，会有两次返回，其中一次是返回当前进程中，另外一次是返回到新创建的进程中。当系统调用fork返回到新创建的进程的时候，它的返回值pid就会等于0，这时候就可以调用相应的函数来设置新创建的进程的uid、gid、gids、debugFlags和rlimits，从而将限制了新创建的进程的权限。

​        对于函数forkAndSpecializeCommon的实现，还有两个地方是需要注意的。

​        第一个地方是只有Zygote进程才有权限创建System进程和Android应用程序进程。从前面[Dalvik虚拟机的启动过程分析](http://blog.csdn.net/luoshengyang/article/details/8885792)一文可以知道，Zygote进程在启动运行在它里面的Dalvik虚拟机的时候，gDvm.zygote的值会等于true，这时候函数forkAndSpecializeCommon才可以使用系统调用fork来创建一个新的进程。

​        第二个地方是在新创建出来的进程中，gDvm.zygote的值会被设置为false，以表示它不是Zygote进程。我们知道，当一个进程使用系统调用fork来创建一个新进程的时候，前者就称为父进程，后者就称为子进程。这时候父进程和子进程共享的地址空间是一样的，但是只要某个地址被父进程或者子进程进行写入操作的时候，这块被写入的地址空间才会在父进程和子进程之间独立开来，这种机制就称为COW（copy on write）。因此，当函数forkAndSpecializeCommon将新创建的进程中的gDvm.zygote的值设置为false的时候， Zygote进程的gDvm.zygote的值仍然保持为true。

​        从上述的第二点的描述还可以进一步看出，由Zygote进程创建出来的System进程和Android应用程序进程实际上是共享了很多东西，而且只要这些东西都是只读的时候，它们就会一直被共享着。从前面[Dalvik虚拟机的启动过程分析](http://blog.csdn.net/luoshengyang/article/details/8885792)一文可以知道，Zygote进程在启动的过程中，加载了很多东西，例如，Java和Android核心类库（dex文件）及其JNI方法（so文件）。这些dex文件和so文件的只读段，例如代码段，都会一直在Zygote进程、System进程和Android应用程序进程中进行共享。这样，我们在Zygote进程中进行的大量预加载行为就获得了价值，一方面是可以加快System进程和Android应用程序进程的启动过程中，另外一方面也使得系统的整体内存消耗减少。

​        此外，运行在Zygote进程中的Dalvik虚拟机开始的时候也会与System进程和Android应用程序进程一起共享，但是由于上述的COW机制，在必要的时候，System进程和Android应用程序进程还是会复制一份出来的，从而使得它们都具有独立的Dalvik虚拟机实例。

​        最后，函数forkAndSpecializeCommon还会调用函数dvmInitAfterZygote来进一步对在新创建的进程中运行的Dalvik虚拟机进行初始化，接下来我们就继续分析它的实现。

​        Step 3. dvmInitAfterZygote

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. /* 
2.  * Do non-zygote-mode initialization.  This is done during VM init for 
3.  * standard startup, or after a "zygote fork" when creating a new process. 
4.  */  
5. bool dvmInitAfterZygote(void)  
6. {  
7. ​    ......  
8.   
9. ​    /* 
10. ​     * Post-zygote heap initialization, including starting 
11. ​     * the HeapWorker thread. 
12. ​     */  
13. ​    if (!dvmGcStartupAfterZygote())  
14. ​        return false;  
15.   
16. ​    ......  
17.   
18. ​    /* start signal catcher thread that dumps stacks on SIGQUIT */  
19. ​    if (!gDvm.reduceSignals && !gDvm.noQuitHandler) {  
20. ​        if (!dvmSignalCatcherStartup())  
21. ​            return false;  
22. ​    }  
23.   
24. ​    /* start stdout/stderr copier, if requested */  
25. ​    if (gDvm.logStdio) {  
26. ​        if (!dvmStdioConverterStartup())  
27. ​            return false;  
28. ​    }  
29.   
30. ​    ......  
31.   
32. ​    /* 
33. ​     * Start JDWP thread.  If the command-line debugger flags specified 
34. ​     * "suspend=y", this will pause the VM.  We probably want this to 
35. ​     * come last. 
36. ​     */  
37. ​    if (!dvmInitJDWP()) {  
38. ​        LOGD("JDWP init failed; continuing anyway\n");  
39. ​    }  
40.   
41. ​    ......  
42.   
43. \#ifdef WITH_JIT  
44. ​    if (gDvm.executionMode == kExecutionModeJit) {  
45. ​        if (!dvmCompilerStartup())  
46. ​            return false;  
47. ​    }  
48. \#endif  
49.   
50. ​    return true;  
51. }  

​        这个函数定义在文件dalvik/vm/Init.c中。

​        函数dvmInitAfterZygote执行的Dalvik虚拟机初始化操作包括：

​        1. 调用函数dvmGcStartupAfterZygote来进行一次GC。

​        2. 调用函数dvmSignalCatcherStartup来启动一个Linux信号收集线程，主要是用来捕捉SIGQUIT信号，以便可以在进程退出前将各个线程的堆栈DUMP出来。

​        3. 调用函数dvmStdioConverterStartup来启动一个标准输出重定向线程，该线程负责将当前进程的标准输出（stdout和stderr）重定向到日志输出系统中去，前提是设置了Dalvik虚拟机的启动选项-Xlog-stdio。

​        4. 调用函数dvmInitJDWP来启动一个JDWP线程，以便我们可以用DDMS工具来调试进程中的Dalvik虚拟机。

​        5. 调用函数dvmCompilerStartup来启动JIT，前提是当前使用的Dalvik虚拟机在编译时支持JIT，并且该Dalvik虚拟机在启动时指定了-Xint:jit选项。

​        这一步执先完成之后，一个Dalvik虚拟机进程就创建完成了，从中我们就可以得出结论：一个Dalvik虚拟机进程实际上就是一个Linux进程。

​         二. Dalvik虚拟机线程的创建过程

​         在Java代码中，我们可以通过java.lang.Thread类的成员函数start来创建一个Dalvik虚拟机线程，因此，接下来我们就从这个函数开始分析Dalvik虚拟机线程的创建过程，如图2所示：

![img](http://img.blog.csdn.net/20130522234252727)

图2 Dalvik虚拟机线程的创建过程

​       这个过程可以分为10个步骤，接下来我们就详细分析每一个步骤。

​       Step 1. Thread.start

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. public class Thread implements Runnable {  
2. ​    ......  
3.   
4. ​    public synchronized void start() {  
5. ​        if (hasBeenStarted) {  
6. ​            throw new IllegalThreadStateException("Thread already started."); // TODO Externalize?  
7. ​        }  
8.   
9. ​        hasBeenStarted = true;  
10.   
11. ​        VMThread.create(this, stackSize);  
12. ​    }  
13.   
14. ​    ......  
15. }  

​        这个函数定义在文件libcore/luni/src/main/java/java/lang/Thread.java中。

​        Thread类的成员函数start首先检查成员变量hasBeenStarted的值是否等于true。如果等于true的话，那么就说明当前正在处理的Thread对象所描述的Java线程已经启动起来了。一个Java线程是不能重复启动的，否则的话，Thread类的成员函数start就会抛出一个类型为IllegalThreadStateException的异常。

​        通过了上面的检查之后，Thread类的成员函数start接下来就继续调用VMThread类的静态成员函数create来创建一个线程。

​        Step 2. VMThread.create

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. class VMThread  
2. {  
3. ​    ......  
4.   
5. ​    native static void create(Thread t, long stacksize);  
6.   
7. ​    ......  
8. }  

​       这个函数定义在文件luni/src/main/java/java/lang/VMThread.java中。

​       VMThread类的静态成员函数create是一个JNI方法，它是C++层的函数Dalvik_java_lang_VMThread_create来实现的，如下所示：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. /* 
2.  * static void create(Thread t, long stacksize) 
3.  * 
4.  * This is eventually called as a result of Thread.start(). 
5.  * 
6.  * Throws an exception on failure. 
7.  */  
8. static void Dalvik_java_lang_VMThread_create(const u4* args, JValue* pResult)  
9. {  
10. ​    Object* threadObj = (Object*) args[0];  
11. ​    s8 stackSize = GET_ARG_LONG(args, 1);  
12.   
13. ​    /* copying collector will pin threadObj for us since it was an argument */  
14. ​    dvmCreateInterpThread(threadObj, (int) stackSize);  
15. ​    RETURN_VOID();  
16. }  

​        这个函数定义在文件dalvik/vm/native/java_lang_VMThread.c中。

​        函数Dalvik_java_lang_VMThread_create的实现很简单，它将Java层传递过来的参数获取出来之后，就调用另外一个函数dvmCreateInterpThread来执行创建线程的工作。

​        Step 3. dvmCreateInterpThread

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. bool dvmCreateInterpThread(Object* threadObj, int reqStackSize)  
2. {  
3. ​    pthread_attr_t threadAttr;  
4. ​    pthread_t threadHandle;  
5. ​    ......  
6. ​    Thread* newThread = NULL;  
7. ​    ......  
8. ​    int stackSize;  
9. ​    ......  
10.   
11. ​    if (reqStackSize == 0)  
12. ​        stackSize = gDvm.stackSize;  
13. ​    else if (reqStackSize < kMinStackSize)  
14. ​        stackSize = kMinStackSize;  
15. ​    else if (reqStackSize > kMaxStackSize)  
16. ​        stackSize = kMaxStackSize;  
17. ​    else  
18. ​        stackSize = reqStackSize;  
19.   
20. ​    pthread_attr_init(&threadAttr);  
21. ​    pthread_attr_setdetachstate(&threadAttr, PTHREAD_CREATE_DETACHED);  
22. ​    ......  
23.   
24. ​    newThread = allocThread(stackSize);  
25. ​    ......  
26.   
27. ​    newThread->threadObj = threadObj;  
28. ​    ......  
29.   
30. ​    int cc = pthread_create(&threadHandle, &threadAttr, interpThreadStart,  
31. ​            newThread);  
32. ​    ......  
33.   
34. ​    while (newThread->status != THREAD_STARTING)  
35. ​        pthread_cond_wait(&gDvm.threadStartCond, &gDvm.threadListLock);  
36. ​    ......  
37.   
38. ​    newThread->next = gDvm.threadList->next;  
39. ​    if (newThread->next != NULL)  
40. ​        newThread->next->prev = newThread;  
41. ​    newThread->prev = gDvm.threadList;  
42. ​    gDvm.threadList->next = newThread;  
43. ​    ......  
44.   
45. ​    newThread->status = THREAD_VMWAIT;  
46. ​    pthread_cond_broadcast(&gDvm.threadStartCond);  
47. ​    ......  
48.   
49. ​    return true;  
50. }  

​        这个函数定义在文件dalvik/vm/Thread.c中。

​        参数reqStackSize表示要创建的Dalvik虚拟机线程的Java栈大小。Dalvik虚拟机线程实际上具有两个栈，一个是Java栈，另一个是Native栈。Native栈是在调用Native代码时使用的，它是由操作系统来管理的，而Java栈是由Dalvik虚拟机来管理的。从前面[Dalvik虚拟机的运行过程分析](http://blog.csdn.net/luoshengyang/article/details/8914953)一文可以知道，Dalvik虚拟机解释器在执行Java代码，每当遇到函数调用指令，就会在当前线程的Java栈上创建一个帧，用来保存当前函数的执行状态，以便要调用的函数执行完成后可以返回到当前函数来。

​        当参数reqStackSize的值等于0的时候，那么就会使用保存在gDvm.stackSize中的值作为接下来要创建的线程的Java栈大小。我们可以通过Dalvik虚拟机的启动选项-Xss来指定gDvm.stackSize的值。如果没有指定，那么gDvm.stackSize的值就设置为kDefaultStackSize (12*1024) 个字节。

​        另外，如果参数reqStackSize的值不等于0，那么它必须大于等于kMinStackSize（512+768），并且小于等于kMaxStackSize（256*1024+768），否则的话，它就会被修正。

​        参数threadObj描述的是Java层的一个Thread对象，它在Dalvik虚拟机中对应有一个Native层的Thread对象。这个Native层的Thread对象是通函数allocThread来分配的，并且与它对应的Java层的Thread对象会保存在它的成员变量threadObj中。

​        函数dvmCreateInterpThread是通过函数pthread_create来创建一个线程。函数pthread_create在创建一个线程的时候，需要知道该线程的属性。这些属性是通过一个pthread_attr_t结构体来描述的，而这个pthread_attr_t结构体可以调用函数pthread_attr_init来初始化，以及调用函数pthread_attr_setdetachstate来设置它的分离状态。

​        函数pthread_create实际上是由pthread库提供的一个函数，它最终是通过系统调用clone来请求内核创建一个线程的。由此就可以看出，Dalvik虚拟机线程实际上就是本地操作系统线程。

​        新创建的Dalvik虚拟机线程启动完成之后，就会将自己的状态设置为THREAD_STARTING，而当前线程会通过一个while循环来等待新创建的Dalvik虚拟机线程的状态被设置为THREAD_STARTING之后再继续往前执行，主要就是：

​        1. 将用来描述新创建的Dalvik虚拟机线程的Native层的Thread对象保存在gDvm.threadList所描述的一个线程列表中，这是因为当前所有Dalvik虚拟机线程都保存在这个列表中。

​        2. 将新创建的Dalvik虚拟机线程的状态设置为THREAD_VMWAIT，使得新创建的Dalvik虚拟机线程继续往前执行，这是因为新创建的Dalvik虚拟机线程将自己的状态设置为THREAD_STARTING唤醒创建它的线程之后，又会等待创建它的线程通知它继续往前执行。

​        注意，前面在调用函数pthread_create来创建新的Dalvik虚拟机线程时，指定该Dalvik虚拟机线程的入口点函数为interpThreadStart，因此，接下来我们继续分析函数interpThreadStart的实现，以便可以了解新创建的Dalvik虚拟机线程的启动过程。

​        Step 4. interpThreadStart

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. static void* interpThreadStart(void* arg)  
2. {  
3. ​    Thread* self = (Thread*) arg;  
4. ​    ......  
5.   
6. ​    prepareThread(self);  
7. ​    ......  
8.   
9. ​    self->status = THREAD_STARTING;  
10. ​    pthread_cond_broadcast(&gDvm.threadStartCond);  
11. ​    ......  
12.   
13. ​    while (self->status != THREAD_VMWAIT)  
14. ​        pthread_cond_wait(&gDvm.threadStartCond, &gDvm.threadListLock);  
15. ​    ......  
16.   
17. ​    self->jniEnv = dvmCreateJNIEnv(self);  
18. ​    ......  
19.   
20. ​    dvmChangeStatus(self, THREAD_RUNNING);  
21. ​    ......  
22.   
23. ​    if (gDvm.debuggerConnected)  
24. ​        dvmDbgPostThreadStart(self);  
25. ​    ......  
26.   
27. ​    int priority = dvmGetFieldInt(self->threadObj,  
28. ​                        gDvm.offJavaLangThread_priority);  
29. ​    dvmChangeThreadPriority(self, priority);  
30. ​    ......  
31.   
32. ​    Method* run = self->threadObj->clazz->vtable[gDvm.voffJavaLangThread_run];  
33. ​    JValue unused;  
34. ​    ......  
35.   
36. ​    dvmCallMethod(self, run, self->threadObj, &unused);  
37. ​    ......  
38.   
39. ​    dvmDetachCurrentThread();  
40.   
41. ​    return NULL;  
42. }  

​        这个函数定义在文件dalvik/vm/Thread.c中。

​        参数arg指向的是一个Native层的Thread对象，这个Thread对象最后会被保存在变量self，用来描述新创建的Dalvik虚拟机线程，也就是当前执行的线程。

​        函数interpThreadStart一方面是对新创建的Dalvik虚拟机线程进行初始化，另一方面是执行新创建的Dalvik虚拟机线程的主体函数，这些工作包括：

​        1. 调用函数prepareThread来初始化新创建的Dalvik虚拟机线程。

​        2. 将新创建的Dalvik虚拟机线程的状态设置为THREAD_STARTING，以便其父线程，也就是创建它的线程可以继续往前执行。

​        3. 通过一个while循环来等待父线程通知自己继续往前执行，也就是等待父线程将自己的状态设置为THREAD_VMWAIT。

​        4. 调用函数dvmCreateJNIEnv来为新创建的Dalvik虚拟机线程创建一个JNI环境。

​        5. 调用函数dvmChangeStatus将新创建的Dalvik虚拟机线程的状态设置为THREAD_RUNNING，表示它正式进入运行状态。

​        6. 如果此时gDvm.debuggerConnected的值等于true，那么就说明有调试器连接到当前Dalvik虚拟机来了，这时候就调用函数dvmDbgPostThreadStart来通知调试器新创建了一个线程。

​        7. 调用函数dvmChangeThreadPriority来设置新创建的Dalvik虚拟机线程的优先级，这个优先级值保存在用来描述新创建的Dalvik虚拟机线程的一个Java层Thread对象的成员变量priority中。

​        8. 找到Java层的java.lang.Thread类的成员函数run，并且通过函数dvmCallMethod来交给Dalvik虚拟机解释器执行，这个java.lang.Thread类的成员函数run即为Dalvik虚拟机线程的Java代码入口点函数。

​        9. 从函数dvmCallMethod返回来之后，新创建的Dalvik虚拟机线程就完成自己的使命了，这时候就可以调用函数dvmDetachCurrentThread来执行清理工作。

​        接下来，我们主要分析第1、4、8和第9个操作，也就是函数prepareThread、dvmCreateJNIEnv、dvmCallMethod和dvmDetachCurrentThread的实现，以便可以了解一个新创建的Dalvik虚拟机线程的初始化、运行和退出过程。

​        Step 5. prepareThread

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. static bool prepareThread(Thread* thread)  
2. {  
3. ​    ......  
4.   
5. ​    /* 
6. ​     * Initialize our reference tracking tables. 
7. ​     * 
8. ​     * Most threads won't use jniMonitorRefTable, so we clear out the 
9. ​     * structure but don't call the init function (which allocs storage). 
10. ​     */  
11. \#ifdef USE_INDIRECT_REF  
12. ​    if (!dvmInitIndirectRefTable(&thread->jniLocalRefTable,  
13. ​            kJniLocalRefMin, kJniLocalRefMax, kIndirectKindLocal))  
14. ​        return false;  
15. \#else  
16. ​    /* 
17. ​     * The JNI local ref table *must* be fixed-size because we keep pointers 
18. ​     * into the table in our stack frames. 
19. ​     */  
20. ​    if (!dvmInitReferenceTable(&thread->jniLocalRefTable,  
21. ​            kJniLocalRefMax, kJniLocalRefMax))  
22. ​        return false;  
23. \#endif  
24. ​    if (!dvmInitReferenceTable(&thread->internalLocalRefTable,  
25. ​            kInternalRefDefault, kInternalRefMax))  
26. ​        return false;  
27.   
28. ​    memset(&thread->jniMonitorRefTable, 0, sizeof(thread->jniMonitorRefTable));  
29. ​    ......  
30.   
31. ​    return true;  
32. }  

​        这个函数定义在文件dalvik/vm/Thread.c中。

​        函数prepareThread最主要的就是对参数thread所描述的一个Dalvik虚拟机线程的三个引用表进行初始化。

​        第一个是JNI本地引用表。一个Dalvik虚拟机线程在执行JNI方法的时候，可能会需要访问Java层的对象。这些Java层对象在被JNI方法访问之前，需要往当前Dalvik虚拟机线程的JNI方法本地引用表添加一个引用，以便它们不会被GC回收。JNI方法本地引用表有两种实现方式，一种是添加到它里面的是间接引用，另一种是直接引用。两者的区别在于，在进行GC的时候，间接引用的Java对象可以移动，而直接引用的Java对象不可以移动。在编译Dalvik虚拟机的时候，可以通过宏USE_INDIRECT_REF来决定使用直接引用表，还是间接引用表。

​        第二个是Dalvik虚拟机内部引用表。有时候，我们需要在Dalvik虚拟机内部为线程创建一些对象，这些对象需要添加到一个Dalvik虚拟机内部引用表中去，以便在线程退出时，可以对它们进行清理。

​        第三个是Monitor引用表。一个Dalvik虚拟机线程在执行JNI方法的时候，除了可能需要访问Java层的对象之外，还可能需要进行一些同步操作，也就是进行MonitorEnter和MonitorExit操作。注意，MonitorEnter和MonitorExit是Java代码中的同步指令，用来实现synchronized代码块或者函数的。Dalvik虚拟机需要跟踪在JNI方法中所执行的MonitorEnter操作，也就是将被Monitor的对象添加到Monitor引用表中去，以便在JNI方法退出时，可以隐式地进行MonitorExit操作，避免出现不对称的MonitorEnter和MonitorExit操作。

​        注意，Monitor引用表实际没有被执行初始化， 函数prepareThread只是对它进行清0，这是因为我们一般很少在JNI方法中执行同步操作的，因此，就最好在使用的时候再进行初始化。

​         这一步执行完成之后，回到前面的Step 4中，即函数interpThreadStart中，接下来它就会调用另外一个函数dvmCreateJNIEnv来为新创建的Dalvik虚拟机线程创建一个JNI环境。

​         Step 6. dvmCreateJNIEnv

​         函数定义在文件dalvik/vm/Jni.c中，它的实现可以参考前面[Dalvik虚拟机的启动过程分析](http://blog.csdn.net/luoshengyang/article/details/8885792)一文，主要就是用来为新创建的Dalvik虚拟机线程创建一个JNI环境，也就是创建一个JNIEnvExt对象，并且为该JNIEnvExt对象创建一个Java代码访问函数表。有了这个Java代码访问函数表之后，我们才可以在JNI方法中访问Java对象，以及执行Java代码。

​         这一步执行完成之后，回到前面的Step 4中，即函数interpThreadStart中，接下来它就会调用另外一个函数dvmCallMethod来通知Dalvik虚拟机解释器执行java.lang.Thread类的成员函数run。

​         Step 7. dvmCallMethod

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. void dvmCallMethod(Thread* self, const Method* method, Object* obj,  
2. ​    JValue* pResult, ...)  
3. {  
4. ​    va_list args;  
5. ​    va_start(args, pResult);  
6. ​    dvmCallMethodV(self, method, obj, false, pResult, args);  
7. ​    va_end(args);  
8. }  

​        这个函数定义在文件dalvik/vm/interp/Stack.c中。

​        函数dvmCallMethod的实现很简单，它通过调用另外一个函数dvmCallMethodV来通知Dalvik虚拟机解释器执行参数method所描述的一个Java函数。在我们这个场景中，实际上就是执行java.lang.Thread类的成员函数run。

​        Step 8. dvmCallMethodV

​        这个函数定义在文件dalvik/vm/interp/Stack.c中。在前面[Dalvik虚拟机的运行过程分析](http://blog.csdn.net/luoshengyang/article/details/8914953)一文中，我们已经分析过函数dvmCallMethodV的实现了，它主要就是调用另外一个函数dvmInterpret来启动Dalvik虚拟机解释器，并且解释执行指定的Java代码。

​        如前所述，在我们这个场景中，接下来要执行的Java代码便是java.lang.Thread类的成员函数run。接下来，我们就继续分析java.lang.Thread类的成员函数run的实现，以便可以了解Dalvik虚拟机线程的Java代码入口点。

​        Step 9. Thread.run

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. public class Thread implements Runnable {  
2. ​    ......  
3.   
4. ​    Runnable target;  
5. ​    ......  
6.   
7. ​    public void run() {  
8. ​        if (target != null) {  
9. ​            target.run();  
10. ​        }  
11. ​    }  
12.   
13. ​    ......  
14. }  

​        这个函数定义在文件libcore/luni/src/main/java/java/lang/Thread.java中。

​        Thread类的成员函数run首先检查成员变量target的值。如果不等于null的话，那么就会调用它所指向的一个Runnable对象的成员函数run来作为新创建的Dalvik虚拟机线程的执行主体，否则的话，就什么也不做。

​        一般来说，我们都是通过从Thread继承下来一个子类来创建一个Dalvik虚拟机线程的。在继承下来的Thread子类中，重写父类Thread的成员函数run，这时候这一步所执行的函数就是子类的成员函数run了。

​        这一步执行完成之后，回到前面的Step 4中，即函数interpThreadStart中，这时候新创建的Dalvik虚拟机线程就执行完成主体函数了，因此，接下来它就即将要退出。不过在退出之前，函数interpThreadStart会调用另外一个函数dvmDetachCurrentThread来执行清理工作。

​        接下来，我们就继续分析函数dvmDetachCurrentThread的实现。

​        Step 10. dvmDetachCurrentThread

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. void dvmDetachCurrentThread(void)  
2. {  
3. ​    Thread* self = dvmThreadSelf();  
4. ​    ......  
5.   
6. ​    /* 
7. ​     * Release any held monitors.  Since there are no interpreted stack 
8. ​     * frames, the only thing left are the monitors held by JNI MonitorEnter 
9. ​     * calls. 
10. ​     */  
11. ​    dvmReleaseJniMonitors(self);  
12. ​    ......  
13.   
14. ​    /* 
15. ​     * Do some thread-exit uncaught exception processing if necessary. 
16. ​     */  
17. ​    if (dvmCheckException(self))  
18. ​        threadExitUncaughtException(self, group);  
19. ​    ......  
20.   
21. ​    vmThread = dvmGetFieldObject(self->threadObj,  
22. ​                    gDvm.offJavaLangThread_vmThread);  
23. ​    ......  
24.   
25. ​    /* 
26. ​     * Tell the debugger & DDM.  This may cause the current thread or all 
27. ​     * threads to suspend. 
28. ​     * 
29. ​     * The JDWP spec is somewhat vague about when this happens, other than 
30. ​     * that it's issued by the dying thread, which may still appear in 
31. ​     * an "all threads" listing. 
32. ​     */  
33. ​    if (gDvm.debuggerConnected)  
34. ​        dvmDbgPostThreadDeath(self);  
35. ​    ......  
36.   
37. ​    dvmObjectNotifyAll(self, vmThread);  
38. ​    ......  
39.   
40. ​    /* 
41. ​     * Lose the JNI context. 
42. ​     */  
43. ​    dvmDestroyJNIEnv(self->jniEnv);  
44. ​    self->jniEnv = NULL;  
45.   
46. ​    self->status = THREAD_ZOMBIE;  
47.   
48. ​    /* 
49. ​     * Remove ourselves from the internal thread list. 
50. ​     */  
51. ​    unlinkThread(self);  
52. ​    ......  
53.   
54. ​    freeThread(self);  
55. }  

​        这个函数定义在文件dalvik/vm/Thread.c中。

​        函数dvmDetachCurrentThread首先是调用函数dvmThreadSelf来获得用来描述当前即将要退出的Dalvik虚拟机线程的Native层的Thread对象self。有了这个Native层的Thread对象self之后，就可以开始执行相应的清理工作了。这些清理工作主要包括：

​        1. 调用函数dvmReleaseJniMonitors来释放那些在JNI方法中持有的Monitor，也就是MonitorExit那些被MonitorEnter了的对象。

​        2. 调用函数dvmCheckException来检查线程是否有未处理异常。如果有的话，那么就调用函数threadExitUncaughtException将它们交给thread-exit-uncaught-exception handler处理。

​        3. 检查当前Dalvik虚拟机是否被调试器连接了，即检查gDvm.debuggerConnected的值是否等于true。如果被调试器连接了的话，那么就调用函数dvmDbgPostThreadDeath通知调试器当前线程要退出了。

​        4. 调用函数dvmObjectNotifyAll来向那些调用了Thread.join等待当前线程结束的线程发送通知。

​        5. 调用函数dvmDestroyJNIEnv来销毁用来描述当前JNI上下文环境的一个JNIEnvExt对象。

​        6. 将当前线程的状态设置为僵尸状态（THREAD_ZOMBIE），并且调用函数unlinkThread将当前线程从Dalvik虚拟机的线程列表中移除。

​        7. 调用函数freeThread来释放当前线程的Java栈和各个引用表（即前面Step 5所描述的三个引用表）所占用的内存，以及释放Thread对象self所占用的内存。

​        这一步执行完成之后，回到前面的Step 4中，即函数interpThreadStart中，这时候新创建的Dalvik虚拟机线程就结束了，整个Dalvik虚拟机线程的创建、运行和结束过程也分析完成了，从中我们就可以得出结论：一个Dalvik虚拟机线程实际上就是一个Linux线程。

​        三. 只执行C/C++代码的Native线程的创建过程

​        我们这里所说的Native线程就是指本地操作系统线程，它们是Dalvik虚拟机在执行C/C++代码的过程中创建的，很显然它们就是Linux线程。例如，在C/C++代码中，我们可以通过C++类Thread的成员函数run来创建一个Native线程。接下来我们就从C++类Thread的成员函数run开始分析Dalvik虚拟机中的Native线程的创建过程，如图3所示：

![img](http://img.blog.csdn.net/20130524025304569)

图3 只执行C/C++代码的Native线程的创建过程

​       这个过程可以分为3个步骤，接下来我们就详细分析每一个步骤。

​       Step 1. Thread.run

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. status_t Thread::run(const char* name, int32_t priority, size_t stack)  
2. {  
3. ​    Mutex::Autolock _l(mLock);  
4. ​    ......  
5.   
6. ​    bool res;  
7. ​    if (mCanCallJava) {  
8. ​        res = createThreadEtc(_threadLoop,  
9. ​                this, name, priority, stack, &mThread);  
10. ​    } else {  
11. ​        res = androidCreateRawThreadEtc(_threadLoop,  
12. ​                this, name, priority, stack, &mThread);  
13. ​    }  
14. ​    ......  
15.   
16. ​    return NO_ERROR;  
17. }  

​        这个函数定义在文件frameworks/base/libs/utils/Threads.cpp中。

​        我们在创建一个Thread对象的时候，可以指定它所描述的Native线程是否可以调用Java代码，它的构造函数原型如下所示：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. class Thread : virtual public RefBase  
2. {  
3. public:  
4. ​    ......  
5.   
6. ​    Thread(bool canCallJava = true);  
7. ​    
8. ​    ......  
9. };  

​        这个函数定义在文件frameworks/base/include/utils/threads.h中。

​        回到Thread类的成员函数run中，如果当前正在处理的Thread对象所描述的Native线程可以调用Java代码，那么它的成员变量mCanCallJava的值就等于false，这时候就会调用另外一个函数androidCreateRawThreadEtc来创建这个Native线程。

​        Step 2. androidCreateRawThreadEtc

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. int androidCreateRawThreadEtc(android_thread_func_t entryFunction,  
2. ​                               void *userData,  
3. ​                               const char* threadName,  
4. ​                               int32_t threadPriority,  
5. ​                               size_t threadStackSize,  
6. ​                               android_thread_id_t *threadId)  
7. {  
8. ​    pthread_attr_t attr;  
9. ​    pthread_attr_init(&attr);  
10. ​    pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);  
11. ​    ......  
12.   
13. ​    if (threadStackSize) {  
14. ​        pthread_attr_setstacksize(&attr, threadStackSize);  
15. ​    }  
16.   
17. ​    errno = 0;  
18. ​    pthread_t thread;  
19. ​    int result = pthread_create(&thread, &attr,  
20. ​                    (android_pthread_entry)entryFunction, userData);  
21. ​    ......  
22.   
23. ​    if (threadId != NULL) {  
24. ​        *threadId = (android_thread_id_t)thread; // XXX: this is not portable  
25. ​    }  
26. ​    return 1;  
27. }  

​        这个函数定义在文件frameworks/base/libs/utils/Threads.cpp中。

​        从函数androidCreateRawThreadEtc的定义就可以看出，在Android系统中，Native线程和Dalvik虚拟机线程一样，都是通过pthread库提供的函数pthread_create来创建的，其中，参数entryFunction指向的就是新创建的Native线程的入口点。

​        参数entryFunction是从前面的Step 1传进来的，它指向的是Thread类的静态成员函数_threadLoop，接下来我们就继续分析它的实现，以便可以了解一个Native线程的运行过程。

​        Step 3. Thread._threadLoop

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. int Thread::_threadLoop(void* user)  
2. {  
3. ​    Thread* const self = static_cast<Thread*>(user);  
4. ​    sp<Thread> strong(self->mHoldSelf);  
5. ​    wp<Thread> weak(strong);  
6. ​    ......  
7.   
8. ​    bool first = true;  
9.   
10. ​    do {  
11. ​        bool result;  
12. ​        if (first) {  
13. ​            first = false;  
14. ​            self->mStatus = self->readyToRun();  
15. ​            result = (self->mStatus == NO_ERROR);  
16.   
17. ​            if (result && !self->mExitPending) {  
18. ​                ......  
19.   
20. ​                result = self->threadLoop();  
21. ​            }  
22. ​        } else {  
23. ​            result = self->threadLoop();  
24. ​        }  
25.   
26. ​        if (result == false || self->mExitPending) {  
27. ​            ......  
28. ​            break;  
29. ​        }  
30.   
31. ​        // Release our strong reference, to let a chance to the thread  
32. ​        // to die a peaceful death.  
33. ​        strong.clear();  
34. ​        // And immediately, re-acquire a strong reference for the next loop  
35. ​        strong = weak.promote();  
36. ​    } while(strong != 0);  
37.   
38. ​    return 0;  
39. }  

​        这个函数定义在文件frameworks/base/libs/utils/Threads.cpp中。

​        参数user指向的是一个Thread对象。这个Thread对象最终保存变量selft，用来描述前面所创建的Native线程。

​        Thread对象self的成员变量mHoldSelf是一个类型为sp<Thread>的[智能](http://lib.csdn.net/base/aiplanning)指针，它引用的就是Thread对象self本身。因此，Thread类的静态成员函数_threadLoop一开始就首先获得Thread对象self的一个强引用strong和弱引用weak，目的是为了避免该Thread对象在线程运行的过程中被销毁。关于Android系统中的智能指针，可以参考前面[Android系统的智能指针（轻量级指针、强指针和弱指针）的实现原理分析](http://blog.csdn.net/luoshengyang/article/details/6786239)一文。

​         Thread类的静态成员函数_threadLoop的主体是一个while循环，这个while循环不断地调用Thread对象self的成员函数threadLoop，来作为前面所创建的Native线程的执行主体，直到满足以下三个条件之一：

​        1. Thread对象self的成员函数threadLoop的返回值等于false。

​        2. Thread对象self的成员函数threadLoop在执行的过程中，调用了另外一个成员函数requestExit请求前面所创建的Native线程退出，这时候Thread对象self的成员变量mExitPending的值就会等于true。

​        3. Thread对象self被销毁了，即强指针strong释放了对Thread对象self的强引用之后，弱指针weak不能成功地提升成强指针。

​        注意，上述while循环第一次执行的时候，在调用Thread对象self的成员函数threadLoop之前，会首先调用另外一个成员函数readyToRun，以便前面所创建的Native线程在正式运行前有机会执行一些自定义的初始化工作。

​        我们一般都是使用Thread子类来创建Native线程的，这时候通过重写父类Thread的成员函数readyToRun和threadLoop，就可以使得新创建的Native线程主动地执行自定义的任务。

​        至此，我们就分析完成只执行C/C++代码的Native线程的创建和运行过程了，从中我们就可以得出结论：Native线程与Dalvik虚拟机线程一样，也是一个Linux线程。

​        四. 能同时执行C/C++代码和Java代码的Native线程的创建过程

​        与只能执行C/C++代码的Native线程一样，能同时执行C/C++代码和Java代码的Native线程也是可以通过C++类Thread的成员函数run来创建的，因此，接下来我们就从这个函数开始分析能同时执行C/C++代码和Java代码的Native线程的创建过程，如图4所示：

![img](http://img.blog.csdn.net/20130524033150318)

图4 能同时执行C/C++代码和Java代码的Native线程的创建过程

​       这个过程可以分为14个步骤，接下来我们就详细分析每一个步骤。

​       Step 1. Thread.run

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. status_t Thread::run(const char* name, int32_t priority, size_t stack)  
2. {  
3. ​    Mutex::Autolock _l(mLock);  
4. ​    ......  
5.   
6. ​    bool res;  
7. ​    if (mCanCallJava) {  
8. ​        res = createThreadEtc(_threadLoop,  
9. ​                this, name, priority, stack, &mThread);  
10. ​    } else {  
11. ​        res = androidCreateRawThreadEtc(_threadLoop,  
12. ​                this, name, priority, stack, &mThread);  
13. ​    }  
14. ​    ......  
15.   
16. ​    return NO_ERROR;  
17. }  

​        这个函数定义在文件frameworks/base/libs/utils/Threads.cpp中。

​        Thread类的成员函数run的实现可以参考前面只执行C/C++代码的Native线程的创建过程的Step 1。不过，在我们这个场景中，当前正在处理的Thread对象的成员变量mCanCallJava的值就等于true，这时候Thread类的成员函数run就会调用另外一个函数createThreadEtc来执行创建线程的工作。因此，接下来我们就继续分析函数createThreadEtc的实现。

​        Step 2. createThreadEtc

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. inline bool createThreadEtc(thread_func_t entryFunction,  
2. ​                            void *userData,  
3. ​                            const char* threadName = "android:unnamed_thread",  
4. ​                            int32_t threadPriority = PRIORITY_DEFAULT,  
5. ​                            size_t threadStackSize = 0,  
6. ​                            thread_id_t *threadId = 0)  
7. {  
8. ​    return androidCreateThreadEtc(entryFunction, userData, threadName,  
9. ​        threadPriority, threadStackSize, threadId) ? true : false;  
10. }  

​        这个函数定义在文件frameworks/base/include/utils/threads.h中。

​        函数createThreadEtc的实现很简单，它通过调用函数androidCreateThreadEtc来执行创建线程的操作。

​        Step 3. androidCreateThreadEtc

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. static android_create_thread_fn gCreateThreadFn = androidCreateRawThreadEtc;  
2.   
3. int androidCreateThreadEtc(android_thread_func_t entryFunction,  
4. ​                            void *userData,  
5. ​                            const char* threadName,  
6. ​                            int32_t threadPriority,  
7. ​                            size_t threadStackSize,  
8. ​                            android_thread_id_t *threadId)  
9. {  
10. ​    return gCreateThreadFn(entryFunction, userData, threadName,  
11. ​        threadPriority, threadStackSize, threadId);  
12. }  

​        这个函数定义在文件frameworks/base/libs/utils/Threads.cpp中。

​        函数createThreadEtc的实现也很简单，它通调用函数指针gCreateThreadFn所指向的函数来执行创建线程的操作。注意，虽然函数指针gCreateThreadFn开始的时候指向的是函数androidCreateRawThreadEtc，但是在前面[Dalvik虚拟机的启动过程分析](http://blog.csdn.net/luoshengyang/article/details/8885792)一文中提到，Zygote进程在启动的过程中，会通过调用函数androidSetCreateThreadFunc将它重新指向函数javaCreateThreadEtc。因此，接下来实际上是调用的函数javaCreateThreadEtc来创建线程。

​        Step 4. javaCreateThreadEtc

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. /*static*/ int AndroidRuntime::javaCreateThreadEtc(  
2. ​                                android_thread_func_t entryFunction,  
3. ​                                void* userData,  
4. ​                                const char* threadName,  
5. ​                                int32_t threadPriority,  
6. ​                                size_t threadStackSize,  
7. ​                                android_thread_id_t* threadId)  
8. {  
9. ​    void** args = (void**) malloc(3 * sizeof(void*));   // javaThreadShell must free  
10. ​    int result;  
11.   
12. ​    assert(threadName != NULL);  
13.   
14. ​    args[0] = (void*) entryFunction;  
15. ​    args[1] = userData;  
16. ​    args[2] = (void*) strdup(threadName);   // javaThreadShell must free  
17.   
18. ​    result = androidCreateRawThreadEtc(AndroidRuntime::javaThreadShell, args,  
19. ​        threadName, threadPriority, threadStackSize, threadId);  
20. ​    return result;  
21. }  

​        这个函数定义在文件frameworks/base/core/jni/AndroidRuntime.cpp中。

​        函数javaCreateThreadEtc首先是将entryFunction、userData和threadName三个参数封装在一个void*数组args中，然后再以该数组为参数，调用另外一个函数androidCreateRawThreadEtc来执行创建线程的工作。

​        Step 5. androidCreateRawThreadEtc

​        这个函数定义在文件frameworks/base/libs/utils/Threads.cpp中，它的具体实现可以参考前面只执行C/C++代码的Native线程的创建过程中的Step 2。总体来说，它就是过pthread库提供的函数pthread_create来创建一个线程，并且将从前面Step 4传递过来的函数AndroidRuntime类的静态成员函数javaThreadShell作为该新创建的线程的入口点函数。因此，接下来我们就继续分析AndroidRuntime类的静态成员函数javaThreadShell的实现。

​        Step 6. AndroidRuntime.javaThreadShell

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. /*static*/ int AndroidRuntime::javaThreadShell(void* args) {  
2. ​    void* start = ((void**)args)[0];  
3. ​    void* userData = ((void **)args)[1];  
4. ​    char* name = (char*) ((void **)args)[2];        // we own this storage  
5. ​    free(args);  
6. ​    JNIEnv* env;  
7. ​    int result;  
8.   
9. ​    /* hook us into the VM */  
10. ​    if (javaAttachThread(name, &env) != JNI_OK)  
11. ​        return -1;  
12.   
13. ​    /* start the thread running */  
14. ​    result = (*(android_thread_func_t)start)(userData);  
15.   
16. ​    /* unhook us */  
17. ​    javaDetachThread();  
18. ​    free(name);  
19.   
20. ​    return result;  
21. }  

​        这个函数定义在文件frameworks/base/core/jni/AndroidRuntime.cpp中。

​        AndroidRuntime类的静态成员函数javaThreadShell主要是执行以三个操作：

​        1. 调用函数javaAttachThread来将当前线程附加到在当前进程中运行的Dalvik虚拟机中去，使得当前线程不仅能够执行 C/C++代码，还可以执行Java代码。

​        2. 调用函数指针start所指向的函数来作为当前线程的执行主体。函数指针start指向的函数是在前面Step 4中封装，而被封装的函数是从前面的Step 1传递过来的，也就是Thread类的静态成员函数_threadLoop。因此，当前新创建的线程是以Thread类的静态成员函数_threadLoop作为执行主体的。

​        3. 从Thread类的静态成员函数_threadLoop返回来之后，当前线程就准备要退出了。在退出之前，需要调用函数javaDetachThread来将当前线程从当前进程中运行的Dalvik虚拟机中移除。

​        接下来，我们就分别分析上述三个函数的实现，以便可以了解一个能同时执行C/C++代码和Java代码的Native线程的初始化和运行过程。

​        Step 7. javaAttachThread

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. static int javaAttachThread(const char* threadName, JNIEnv** pEnv)  
2. {  
3. ​    JavaVMAttachArgs args;  
4. ​    JavaVM* vm;  
5. ​    jint result;  
6.   
7. ​    vm = AndroidRuntime::getJavaVM();  
8. ​    assert(vm != NULL);  
9.   
10. ​    args.version = JNI_VERSION_1_4;  
11. ​    args.name = (char*) threadName;  
12. ​    args.group = NULL;  
13.   
14. ​    result = vm->AttachCurrentThread(pEnv, (void*) &args);  
15. ​    if (result != JNI_OK)  
16. ​        LOGI("NOTE: attach of thread '%s' failed\n", threadName);  
17.   
18. ​    return result;  
19. }  

​        这个函数定义在文件frameworks/base/core/jni/AndroidRuntime.cpp中。

​        函数javaAttachThread首先调用AndroidRuntime类的静态成员函数getJavaVM来获得在当前进程中运行的Dalvik虚拟机实例。这个Dalvik虚拟机实例是在Zygote进程启动的过程中创建的，具体可以参考前面[Dalvik虚拟机的启动过程分析](http://blog.csdn.net/luoshengyang/article/details/8885792)一文。

​        获得了运行在当前进程中的Dalvik虚拟机实例实例之后，函数javaAttachThread接下来就可以调用它的成员函数AttachCurrentThread来将当前线程附加到它里去。同样是从前面[Dalvik虚拟机的启动过程分析](http://blog.csdn.net/luoshengyang/article/details/8885792)一文可以知道，这里的变量vm指向的实际上是一个JavaVMExt对象，这个JavaVMExt对象的成员函数AttachCurrentThread实际上是一个函数指针，它指向的函数为在Dalvik虚拟机内部定义的一个全局函数AttachCurrentThread。因此，接下来我们就继续分析在Dalvik虚拟机内部定义的全局函数AttachCurrentThread的实现。

​        Step 8. AttachCurrentThread

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. /* 
2.  * Attach the current thread to the VM.  If the thread is already attached, 
3.  * this is a no-op. 
4.  */  
5. static jint AttachCurrentThread(JavaVM* vm, JNIEnv** p_env, void* thr_args)  
6. {  
7. ​    return attachThread(vm, p_env, thr_args, false);  
8. }  

​        这个函数定义在文件dalvik/vm/Jni.c中。

​        函数AttachCurrentThread的实现很简单，它通过调用另外一个函数attachThread来将当前线程附加到在当前进程中运行的Dalvik虚拟机中去。

​       Step 9. attachThread

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. static jint attachThread(JavaVM* vm, JNIEnv** p_env, void* thr_args,  
2. ​    bool isDaemon)  
3. {  
4. ​    JavaVMAttachArgs* args = (JavaVMAttachArgs*) thr_args;  
5. ​    Thread* self;  
6. ​    bool result = false;  
7.   
8. ​    /* 
9. ​     * Return immediately if we're already one with the VM. 
10. ​     */  
11. ​    self = dvmThreadSelf();  
12. ​    if (self != NULL) {  
13. ​        *p_env = self->jniEnv;  
14. ​        return JNI_OK;  
15. ​    }  
16.   
17. ​    /* 
18. ​     * No threads allowed in zygote mode. 
19. ​     */  
20. ​    if (gDvm.zygote) {  
21. ​        return JNI_ERR;  
22. ​    }  
23.   
24. ​    /* tweak the JavaVMAttachArgs as needed */  
25. ​    JavaVMAttachArgs argsCopy;  
26. ​    if (args == NULL) {  
27. ​        /* allow the v1.1 calling convention */  
28. ​        argsCopy.version = JNI_VERSION_1_2;  
29. ​        argsCopy.name = NULL;  
30. ​        argsCopy.group = dvmGetMainThreadGroup();  
31. ​    } else {  
32. ​        assert(args->version >= JNI_VERSION_1_2);  
33.   
34. ​        argsCopy.version = args->version;  
35. ​        argsCopy.name = args->name;  
36. ​        if (args->group != NULL)  
37. ​            argsCopy.group = args->group;  
38. ​        else  
39. ​            argsCopy.group = dvmGetMainThreadGroup();  
40. ​    }  
41.   
42. ​    result = dvmAttachCurrentThread(&argsCopy, isDaemon);  
43. ​    ......  
44.   
45. ​    if (result) {  
46. ​        self = dvmThreadSelf();  
47. ​        assert(self != NULL);  
48. ​        dvmChangeStatus(self, THREAD_NATIVE);  
49. ​        *p_env = self->jniEnv;  
50. ​        return JNI_OK;  
51. ​    } else {  
52. ​        return JNI_ERR;  
53. ​    }  
54. }  

​        这个函数定义在文件dalvik/vm/Jni.c中。

​        函数attachThread首先是调用函数dvmThreadSelf来检查在当前进程中运行的Dalvik虚拟机是否已经为当前线程创建过一个Thread对象了。如果已经创建过的话，那么就说明当前线程之前已经附加到当前进程中运行的Dalvik虚拟机去了。因此，这时候函数attachThread就什么也不用做就返回了。

​        我们假设当前线程还没有附加到当前进程中运行的Dalvik虚拟机中，接下来函数attachThread就继续检查gDvm.zygote的值是否等于true。如果等于true的话，那么就说明当前进程的Zygote进程，这时候是不允许创建附加新的线程到Dalvik虚拟机中来的。因此，这时候函数attachThread就直接返回一个错误码给调用者。

​        通过了前面的检查之后，函数attachThread再将根据参数thr_args的值来初始化一个JavaVMAttachArgs结构体。这个JavaVMAttachArgs结构体用来描述当前线程附加到Dalvik虚拟机时的属性，包括它所使用的JNI版本号、线程名称和所属的线程组。注意，当参数thr_args的值等于NULL的时候，当前线程所使用的JNI版本号默认为JNI_VERSION_1_2，并且它的线程组被设置为当前进程的主线程组。另一方面，如果参数thr_args的值不等于NULL，那么就要求它所指定的JNI版本号大于等于JNI_VERSION_1_2，并且当它没有指定线程组属性性，将当前线程的线程组设置为当前进程的主线程组。

​        初始化好上述的JavaVMAttachArgs结构体之后，函数attachThread就调用另外一个函数dvmAttachCurrentThread来为当前线程创建一个JNI上下文环境。如果函数dvmAttachCurrentThread能成功为当前线程创建一个JNI上下文环境，那么它的返回值result就会等于ture。在这种情况下，函数dvmAttachCurrentThread同时还会为当前线程一个Thread对象。因此，这时候调用函数dvmThreadSelf的返回值就不等于NULL，并且它的成员变量jniEnv指向了一个JNIEnvExt对象，用来描述当前线程的JNI上下文环境。

​        最后，函数attachThread就将当前线程的状态设置为THREAD_NATIVE，以表示它接下来都是在执行C/C++代码，并且将前面所创建的JNI上下文环境保存在输出参数p_env中返回给调用者使用。

​        接下来，我们就继续分析函数dvmAttachCurrentThread的实现，以便可以了解Dalvik虚拟机为当前线程创建JNI上下文环境的过程。

​        Step 10. dvmAttachCurrentThread

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. bool dvmAttachCurrentThread(const JavaVMAttachArgs* pArgs, bool isDaemon)  
2. {  
3. ​    Thread* self = NULL;  
4. ​    ......  
5. ​    bool ok, ret;  
6.   
7. ​    self = allocThread(gDvm.stackSize);  
8. ​    ......  
9. ​    setThreadSelf(self);  
10. ​    ......  
11.   
12. ​    ok = prepareThread(self);  
13. ​    ......  
14. ​      
15. ​    self->jniEnv = dvmCreateJNIEnv(self);  
16. ​    ......  
17.   
18. ​    self->next = gDvm.threadList->next;  
19. ​    if (self->next != NULL)  
20. ​        self->next->prev = self;  
21. ​    self->prev = gDvm.threadList;  
22. ​    gDvm.threadList->next = self;  
23. ​    ......  
24.   
25. ​    /* tell the debugger & DDM */  
26. ​    if (gDvm.debuggerConnected)  
27. ​        dvmDbgPostThreadStart(self);  
28.   
29. ​    return ret;  
30.   
31. ​    ......  
32. }  

​        这个函数定义在文件dalvik/vm/Thread.c中。

​        函数dvmAttachCurrentThread主要是执行以下五个操作：

​        1. 调用函数allocThread和setThreadSelf为当前线程创建一个Thread对象。

​        2. 调用函数prepareThread来初始化当前线程所要使用到的引用表，它的具体实现可以参考前面Dalvik虚拟机线程的创建过程中的Step 5。

​        3. 调用函数dvmCreateJNIEnv来当前线程创建一个JNI上下文环境，它的具体实现可以参考前面[Dalvik虚拟机的启动过程分析](http://blog.csdn.net/luoshengyang/article/details/8885792)一文。这个创建出来的JNI上下文环境使用一个JNIEnvExt结构体来描述。JNIEnvExt结构体有一个重要的成员变量funcTable，它指向的是一系列的Dalvik虚拟机回调函数。正是因为有了这个Dalvik虚拟机回调函数表，当前线程才可以访问Dalvik虚拟机中的Java对象或者执行Dalvik虚拟机中的Java代码。

​        4. 将当前线程添加到gDvm.threadList所描述的一个Dalvik虚拟机线程列表中去。从这里就可以看出，可以执行Java代码的Native线程和Dalvik虚拟机线程一样，都是保存在gDvm.threadList所描述的一个列表中。

​        5. 如果在当前进程中运行的Dalvik虚拟机有调试器连接，即gDvm.debuggerConnected的值等于true，那么就调用函数dvmDbgPostThreadStart来通知调试器在当前进程中运行的Dalvik虚拟机新增了一个线程。

​        这一步执行完成之后，当前线程就具有一个JNI上下文环境了。返回到前面的Step 6中，即AndroidRuntime类的静态成员函数javaThreadShell中，接下来它就会调用Thread类的静态成员函数_threadLoop来作为当前线程的执行主体。

​        Step 11. Thread._threadLoop

​        这个函数定义在文件frameworks/base/libs/utils/Threads.cpp中，它的具体实现可以参考前面只执行C/C++代码的Native线程的创建过程的Step 3。总的来说，Thread类的静态成员函数_threadLoop就是在一个无限循环中不断地调用用来描述当前线程的一个Thread对象的成员函数threadLoop，直到当前线程请求退出为止。

​        这一步执行完成之后，当前线程就准备要退出了。返回前面的Step 6中，即AndroidRuntime类的静态成员函数javaThreadShell中，接下来它就会调用函数javaDetachThread来将当前线程从在当前进程中运行的Dalvik虚拟机中移除。因此，接下来我们就继续分析函数javaDetachThread的实现。

​        Step 12. javaDetachThread

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. /* 
2.  * Detach the current thread from the set visible to the VM. 
3.  */  
4. static int javaDetachThread(void)  
5. {  
6. ​    JavaVM* vm;  
7. ​    jint result;  
8.   
9. ​    vm = AndroidRuntime::getJavaVM();  
10. ​    assert(vm != NULL);  
11.   
12. ​    result = vm->DetachCurrentThread();  
13. ​    if (result != JNI_OK)  
14. ​        LOGE("ERROR: thread detach failed\n");  
15. ​    return result;  
16. }  

​        这个函数定义在frameworks/base/core/jni/AndroidRuntime.cpp中。

​        函数javaDetachThread首先是调用AndroidRuntime类的静态成员函数getJavaVM来获得在当前进程中运行的Dalvik虚拟机实例，接着再调用这个Dalvik虚拟机实例的成员函数DetachCurrentThread来将当前线程将Dalvik虚拟机中移除。

​        在前面的Step 7中提到，AndroidRuntime类的静态成员函数getJavaVM返回的实际上是一个JavaVMExt对象，这个JavaVMExt对象的成员函数DetachCurrentThread实际上与成员函数AttachCurrentThread一样，也是一个函数指针，并且通过前面[Dalvik虚拟机的启动过程分析](http://blog.csdn.net/luoshengyang/article/details/8885792)一文可以知道，这个函数指针指向的是在Dalvik虚拟机内部定义的函数DetachCurrentThread。

​        因此，接下来我们就继续分在Dalvik虚拟机内部定义的函数DetachCurrentThread的实现，以便可以了解一个能同时执行C/C++代码和Java代码的Native线程脱离Dalvik虚拟机的过程。

​        Step 13. DetachCurrentThread

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8923484#) [copy](http://blog.csdn.net/luoshengyang/article/details/8923484#)

1. /* 
2.  * Dissociate the current thread from the VM. 
3.  */  
4. static jint DetachCurrentThread(JavaVM* vm)  
5. {  
6. ​    ......  
7.   
8. ​    /* detach the thread */  
9. ​    dvmDetachCurrentThread();  
10.   
11. ​    /* (no need to change status back -- we have no status) */  
12. ​    return JNI_OK;  
13. }  

​        这个函数定义在文件dalvik/vm/Jni.c中。

​        函数DetachCurrentThread主要是能通过调用另外一个函数dvmDetachCurrentThread来将当前线程从Dalvik虚拟机中移除，因此，接下来我们就继续分析函数dvmDetachCurrentThread的实现。

​        Step 14. dvmDetachCurrentThread

​        这个函数定义在文件dalvik/vm/Thread.c中，它的具体实现可以参考前面Dalvik虚拟机线程的创建过程中的Step 10。总的来说，函数dvmDetachCurrentThread主要就是用来清理当前线程的JNI上下文环境，例如，清理当前线程还在引用的对象，以及清理当前线程所占用的内存等。

​        至此，我们就分析完成能同时执行C/C++代码和Java代码的Native线程的创建和运行过程了，从中我们就可以得出结论：能同时执行C/C++代码和Java代码的Native线程与只能执行C/C++代码的Native线程一样，都是一个Linux线程，不过区别就在于前者会被附加到Dalvik虚拟机中去，并且具有一个JNI上下文环境，因而可以执行Java代码。

​        这样，Dalvik虚拟机进程和线程的创建过程分析就分析完成了，从中我们就可以得到它们与本地操作系统的进程和线程的关系：

​        1. Dalvik虚拟机进程就是本地操作系统进程，也就是Linux进程，区别在于前者运行有一个Dalvik虚拟机实例。

​        2. Dalvik虚拟机线程就是本地操作系统进程，也就是Linux线程，区别在于前者在创建的时候会自动附加到Dalvik虚拟机中去，而后者在需要执行Java代码的时候才会附加到Dalvik虚拟机中去。

​        我们可以思考一下：为什么Dalvik虚拟机要将自己的进程和线程使用本地操作系统的进程和线程来实现呢？我们知道，进程调度是一个很复杂的问题，特别是在多核的情况下，它要求高效地利用CPU资源，并且公平地调试各个进程，以达到系统吞吐量最大的目标。Linux内核本身已经实现了这种高效和公平的进程调度机制，因此，就完全没有必要在Dalvik虚拟机内部再实现一套，这样就要求Dalvik虚拟机使用本地操作系统的进程来作为自己的进程。此外，Linux内核没有线程的概念，不过它可以使用一种称为轻量级进程的概念来实现线程，这样Dalvik虚拟机使用本地操作系统的线程来作为自己的线程，就同样可以利用Linux内核的进程调度机制来调度它的线程，从而也能实现高效和公平的原则。

​        至此，我们也完成了Dalvik虚拟机的学习了，重新学习请参考[Dalvik虚拟机简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/8852432)一文。