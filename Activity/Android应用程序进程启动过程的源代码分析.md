 [Android](http://lib.csdn.net/base/android)应用程序框架层创建的应用程序进程具有两个特点，一是进程的入口函数是ActivityThread.main，二是进程天然支持Binder进程间通信机制；这两个特点都是在进程的初始化过程中实现的，本文将详细分析[android](http://lib.csdn.net/base/android)应用程序进程创建过程中是如何实现这两个特点的。

《Android系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

​        Android应用程序框架层创建的应用程序进程的入口函数是ActivityThread.main比较好理解，即进程创建完成之后，Android应用程序框架层就会在这个进程中将ActivityThread类加载进来，然后执行它的main函数，这个main函数就是进程执行消息循环的地方了。Android应用程序框架层创建的应用程序进程天然支持Binder进程间通信机制这个特点应该怎么样理解呢？前面我们在学习[Android系统的Binder进程间通信机制](http://blog.csdn.net/luoshengyang/article/details/6618363)时说到，它具有四个组件，分别是驱动程序、守护进程、Client以及Server，其中Server组件在初始化时必须进入一个循环中不断地与Binder驱动程序进行到交互，以便获得Client组件发送的请求，具体可参考[Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6629298)一文，但是，当我们在Android应用程序中实现Server组件的时候，我们并没有让进程进入一个循环中去等待Client组件的请求，然而，当Client组件得到这个Server组件的远程接口时，却可以顺利地和Server组件进行进程间通信，这就是因为Android应用程序进程在创建的时候就已经启动了一个线程池来支持Server组件和Binder驱动程序之间的交互了，这样，极大地方便了在Android应用程序中创建Server组件。

​        在Android应用程序框架层中，是由ActivityManagerService组件负责为Android应用程序创建新的进程的，它本来也是运行在一个独立的进程之中，不过这个进程是在系统启动的过程中创建的。ActivityManagerService组件一般会在什么情况下会为应用程序创建一个新的进程呢？当系统决定要在一个新的进程中启动一个Activity或者Service时，它就会创建一个新的进程了，然后在这个新的进程中启动这个Activity或者Service，具体可以参考[Android系统在新进程中启动自定义服务过程（startService）的原理分析](http://blog.csdn.net/luoshengyang/article/details/6677029)、[Android应用程序启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6689748)和[Android应用程序在新的进程中启动新的Activity的方法和过程分析](http://blog.csdn.net/luoshengyang/article/details/6720261)这三篇文章。

​        ActivityManagerService启动新的进程是从其成员函数startProcessLocked开始的，在深入分析这个过程之前，我们先来看一下进程创建过程的序列图，然后再详细分析每一个步骤。

![img](http://hi.csdn.net/attachment/201109/5/0_1315236533f7n7.gif)

​       [点击查看大图](http://hi.csdn.net/attachment/201109/5/0_1315236533f7n7.gif)

​        Step 1. ActivityManagerService.startProcessLocked

​        这个函数定义在frameworks/base/services/[Java](http://lib.csdn.net/base/java)/com/android/server/am/ActivityManagerService.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. public final class ActivityManagerService extends ActivityManagerNative    
2. ​        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {    
3. ​    
4. ​    ......    
5. ​    
6. ​    private final void startProcessLocked(ProcessRecord app,    
7. ​                String hostingType, String hostingNameStr) {    
8. ​    
9. ​        ......    
10. ​    
11. ​        try {    
12. ​            int uid = app.info.uid;    
13. ​            int[] gids = null;    
14. ​            try {    
15. ​                gids = mContext.getPackageManager().getPackageGids(    
16. ​                    app.info.packageName);    
17. ​            } catch (PackageManager.NameNotFoundException e) {    
18. ​                ......    
19. ​            }    
20. ​                
21. ​            ......    
22. ​    
23. ​            int debugFlags = 0;    
24. ​                
25. ​            ......    
26. ​                
27. ​            int pid = Process.start("android.app.ActivityThread",    
28. ​                mSimpleProcessManagement ? app.processName : null, uid, uid,    
29. ​                gids, debugFlags, null);    
30. ​                
31. ​            ......    
32. ​    
33. ​        } catch (RuntimeException e) {    
34. ​                
35. ​            ......    
36. ​    
37. ​        }    
38. ​    }    
39. ​    
40. ​    ......    
41. ​    
42. }    

​        它调用了Process.start函数开始为应用程序创建新的进程，注意，它传入一个第一个参数为"android.app.ActivityThread"，这就是进程初始化时要加载的Java类了，把这个类加载到进程之后，就会把它里面的静态成员函数main作为进程的入口点，后面我们会看到。

​        Step 2. Process.start 

​        这个函数定义在frameworks/base/core/java/android/os/Process.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. public class Process {  
2. ​    ......  
3.   
4. ​    public static final int start(final String processClass,  
5. ​        final String niceName,  
6. ​        int uid, int gid, int[] gids,  
7. ​        int debugFlags,  
8. ​        String[] zygoteArgs)  
9. ​    {  
10. ​        if (supportsProcesses()) {  
11. ​            try {  
12. ​                return startViaZygote(processClass, niceName, uid, gid, gids,  
13. ​                    debugFlags, zygoteArgs);  
14. ​            } catch (ZygoteStartFailedEx ex) {  
15. ​                ......  
16. ​            }  
17. ​        } else {  
18. ​            ......  
19.   
20. ​            return 0;  
21. ​        }  
22. ​    }  
23.   
24. ​    ......  
25. }  

​       这里的supportsProcesses函数返回值为true，它是一个Native函数，实现在frameworks/base/core/jni/android_util_Process.cpp文件中：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. jboolean android_os_Process_supportsProcesses(JNIEnv* env, jobject clazz)  
2. {  
3. ​    return ProcessState::self()->supportsProcesses();  
4. }  

​       ProcessState::supportsProcesses函数定义在frameworks/base/libs/binder/ProcessState.cpp文件中：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. bool ProcessState::supportsProcesses() const  
2. {  
3. ​    return mDriverFD >= 0;  
4. }  

​       这里的mDriverFD是设备文件/dev/binder的打开描述符，如果成功打开了这个设备文件，那么它的值就会大于等于0，因此，它的返回值为true。

​       回到Process.start函数中，它调用startViaZygote函数进一步操作。

​       Step 3. Process.startViaZygote

​       这个函数定义在frameworks/base/core/java/android/os/Process.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. public class Process {  
2. ​    ......  
3.   
4. ​    private static int startViaZygote(final String processClass,  
5. ​            final String niceName,  
6. ​            final int uid, final int gid,  
7. ​            final int[] gids,  
8. ​            int debugFlags,  
9. ​            String[] extraArgs)  
10. ​            throws ZygoteStartFailedEx {  
11. ​        int pid;  
12.   
13. ​        synchronized(Process.class) {  
14. ​            ArrayList<String> argsForZygote = new ArrayList<String>();  
15.   
16. ​            // --runtime-init, --setuid=, --setgid=,  
17. ​            // and --setgroups= must go first  
18. ​            argsForZygote.add("--runtime-init");  
19. ​            argsForZygote.add("--setuid=" + uid);  
20. ​            argsForZygote.add("--setgid=" + gid);  
21. ​            if ((debugFlags & Zygote.DEBUG_ENABLE_SAFEMODE) != 0) {  
22. ​                argsForZygote.add("--enable-safemode");  
23. ​            }  
24. ​            if ((debugFlags & Zygote.DEBUG_ENABLE_DEBUGGER) != 0) {  
25. ​                argsForZygote.add("--enable-debugger");  
26. ​            }  
27. ​            if ((debugFlags & Zygote.DEBUG_ENABLE_CHECKJNI) != 0) {  
28. ​                argsForZygote.add("--enable-checkjni");  
29. ​            }  
30. ​            if ((debugFlags & Zygote.DEBUG_ENABLE_ASSERT) != 0) {  
31. ​                argsForZygote.add("--enable-assert");  
32. ​            }  
33.   
34. ​            //TODO optionally enable debuger  
35. ​            //argsForZygote.add("--enable-debugger");  
36.   
37. ​            // --setgroups is a comma-separated list  
38. ​            if (gids != null && gids.length > 0) {  
39. ​                StringBuilder sb = new StringBuilder();  
40. ​                sb.append("--setgroups=");  
41.   
42. ​                int sz = gids.length;  
43. ​                for (int i = 0; i < sz; i++) {  
44. ​                    if (i != 0) {  
45. ​                        sb.append(',');  
46. ​                    }  
47. ​                    sb.append(gids[i]);  
48. ​                }  
49.   
50. ​                argsForZygote.add(sb.toString());  
51. ​            }  
52.   
53. ​            if (niceName != null) {  
54. ​                argsForZygote.add("--nice-name=" + niceName);  
55. ​            }  
56.   
57. ​            argsForZygote.add(processClass);  
58.   
59. ​            if (extraArgs != null) {  
60. ​                for (String arg : extraArgs) {  
61. ​                    argsForZygote.add(arg);  
62. ​                }  
63. ​            }  
64.   
65. ​            pid = zygoteSendArgsAndGetPid(argsForZygote);  
66. ​        }  
67. ​    }  
68.   
69. ​    ......  
70. }  

​        这个函数将创建进程的参数放到argsForZygote列表中去，如参数"--runtime-init"表示要为新创建的进程初始化运行时库，然后调用zygoteSendAndGetPid函数进一步操作。

​        Step 4. Process.zygoteSendAndGetPid

​        这个函数定义在frameworks/base/core/java/android/os/Process.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. public class Process {  
2. ​    ......  
3.   
4. ​    private static int zygoteSendArgsAndGetPid(ArrayList<String> args)  
5. ​            throws ZygoteStartFailedEx {  
6. ​        int pid;  
7.   
8. ​        openZygoteSocketIfNeeded();  
9.   
10. ​        try {  
11. ​            /** 
12. ​            * See com.android.internal.os.ZygoteInit.readArgumentList() 
13. ​            * Presently the wire format to the zygote process is: 
14. ​            * a) a count of arguments (argc, in essence) 
15. ​            * b) a number of newline-separated argument strings equal to count 
16. ​            * 
17. ​            * After the zygote process reads these it will write the pid of 
18. ​            * the child or -1 on failure. 
19. ​            */  
20.   
21. ​            sZygoteWriter.write(Integer.toString(args.size()));  
22. ​            sZygoteWriter.newLine();  
23.   
24. ​            int sz = args.size();  
25. ​            for (int i = 0; i < sz; i++) {  
26. ​                String arg = args.get(i);  
27. ​                if (arg.indexOf('\n') >= 0) {  
28. ​                    throw new ZygoteStartFailedEx(  
29. ​                        "embedded newlines not allowed");  
30. ​                }  
31. ​                sZygoteWriter.write(arg);  
32. ​                sZygoteWriter.newLine();  
33. ​            }  
34.   
35. ​            sZygoteWriter.flush();  
36.   
37. ​            // Should there be a timeout on this?  
38. ​            pid = sZygoteInputStream.readInt();  
39.   
40. ​            if (pid < 0) {  
41. ​                throw new ZygoteStartFailedEx("fork() failed");  
42. ​            }  
43. ​        } catch (IOException ex) {  
44. ​            ......  
45. ​        }  
46.   
47. ​        return pid;  
48. ​    }  
49.   
50. ​    ......  
51. }  

​         这里的sZygoteWriter是一个Socket写入流，是由openZygoteSocketIfNeeded函数打开的：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. public class Process {  
2. ​    ......  
3.   
4. ​    /** 
5. ​    * Tries to open socket to Zygote process if not already open. If 
6. ​    * already open, does nothing.  May block and retry. 
7. ​    */  
8. ​    private static void openZygoteSocketIfNeeded()  
9. ​            throws ZygoteStartFailedEx {  
10.   
11. ​        int retryCount;  
12.   
13. ​        if (sPreviousZygoteOpenFailed) {  
14. ​            /* 
15. ​            * If we've failed before, expect that we'll fail again and 
16. ​            * don't pause for retries. 
17. ​            */  
18. ​            retryCount = 0;  
19. ​        } else {  
20. ​            retryCount = 10;  
21. ​        }  
22. ​      
23. ​        /* 
24. ​        * See bug #811181: Sometimes runtime can make it up before zygote. 
25. ​        * Really, we'd like to do something better to avoid this condition, 
26. ​        * but for now just wait a bit... 
27. ​        */  
28. ​        for (int retry = 0  
29. ​            ; (sZygoteSocket == null) && (retry < (retryCount + 1))  
30. ​            ; retry++ ) {  
31.   
32. ​                if (retry > 0) {  
33. ​                    try {  
34. ​                        Log.i("Zygote", "Zygote not up yet, sleeping...");  
35. ​                        Thread.sleep(ZYGOTE_RETRY_MILLIS);  
36. ​                    } catch (InterruptedException ex) {  
37. ​                        // should never happen  
38. ​                    }  
39. ​                }  
40.   
41. ​                try {  
42. ​                    sZygoteSocket = new LocalSocket();  
43. ​                    sZygoteSocket.connect(new LocalSocketAddress(ZYGOTE_SOCKET,  
44. ​                        LocalSocketAddress.Namespace.RESERVED));  
45.   
46. ​                    sZygoteInputStream  
47. ​                        = new DataInputStream(sZygoteSocket.getInputStream());  
48.   
49. ​                    sZygoteWriter =  
50. ​                        new BufferedWriter(  
51. ​                        new OutputStreamWriter(  
52. ​                        sZygoteSocket.getOutputStream()),  
53. ​                        256);  
54.   
55. ​                    Log.i("Zygote", "Process: zygote socket opened");  
56.   
57. ​                    sPreviousZygoteOpenFailed = false;  
58. ​                    break;  
59. ​                } catch (IOException ex) {  
60. ​                    ......  
61. ​                }  
62. ​        }  
63.   
64. ​        ......  
65. ​    }  
66.   
67. ​    ......  
68. }  

​        这个Socket由frameworks/base/core/java/com/android/internal/os/ZygoteInit.java文件中的ZygoteInit类在runSelectLoopMode函数侦听的。

​        Step 5. ZygoteInit.runSelectLoopMode

​        这个函数定义在frameworks/base/core/java/com/android/internal/os/ZygoteInit.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. public class ZygoteInit {  
2. ​    ......  
3.   
4. ​    /** 
5. ​    * Runs the zygote process's select loop. Accepts new connections as 
6. ​    * they happen, and reads commands from connections one spawn-request's 
7. ​    * worth at a time. 
8. ​    * 
9. ​    * @throws MethodAndArgsCaller in a child process when a main() should 
10. ​    * be executed. 
11. ​    */  
12. ​    private static void runSelectLoopMode() throws MethodAndArgsCaller {  
13. ​        ArrayList<FileDescriptor> fds = new ArrayList();  
14. ​        ArrayList<ZygoteConnection> peers = new ArrayList();  
15. ​        FileDescriptor[] fdArray = new FileDescriptor[4];  
16.   
17. ​        fds.add(sServerSocket.getFileDescriptor());  
18. ​        peers.add(null);  
19.   
20. ​        int loopCount = GC_LOOP_COUNT;  
21. ​        while (true) {  
22. ​            int index;  
23. ​            /* 
24. ​            * Call gc() before we block in select(). 
25. ​            * It's work that has to be done anyway, and it's better 
26. ​            * to avoid making every child do it.  It will also 
27. ​            * madvise() any free memory as a side-effect. 
28. ​            * 
29. ​            * Don't call it every time, because walking the entire 
30. ​            * heap is a lot of overhead to free a few hundred bytes. 
31. ​            */  
32. ​            if (loopCount <= 0) {  
33. ​                gc();  
34. ​                loopCount = GC_LOOP_COUNT;  
35. ​            } else {  
36. ​                loopCount--;  
37. ​            }  
38.   
39.   
40. ​            try {  
41. ​                fdArray = fds.toArray(fdArray);  
42. ​                index = selectReadable(fdArray);  
43. ​            } catch (IOException ex) {  
44. ​                throw new RuntimeException("Error in select()", ex);  
45. ​            }  
46.   
47. ​            if (index < 0) {  
48. ​                throw new RuntimeException("Error in select()");  
49. ​            } else if (index == 0) {  
50. ​                ZygoteConnection newPeer = acceptCommandPeer();  
51. ​                peers.add(newPeer);  
52. ​                fds.add(newPeer.getFileDesciptor());  
53. ​            } else {  
54. ​                boolean done;  
55. ​                done = peers.get(index).runOnce();  
56.   
57. ​                if (done) {  
58. ​                    peers.remove(index);  
59. ​                    fds.remove(index);  
60. ​                }  
61. ​            }  
62. ​        }  
63. ​    }  
64.   
65. ​    ......  
66. }  

​        当Step 4将数据通过Socket接口发送出去后，就会下面这个语句：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. done = peers.get(index).runOnce();  

​        这里从peers.get(index)得到的是一个ZygoteConnection对象，表示一个Socket连接，因此，接下来就是调用ZygoteConnection.runOnce函数进一步处理了。

​        Step 6. ZygoteConnection.runOnce

​        这个函数定义在frameworks/base/core/java/com/android/internal/os/ZygoteConnection.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. class ZygoteConnection {  
2. ​    ......  
3.   
4. ​    boolean runOnce() throws ZygoteInit.MethodAndArgsCaller {  
5. ​        String args[];  
6. ​        Arguments parsedArgs = null;  
7. ​        FileDescriptor[] descriptors;  
8.   
9. ​        try {  
10. ​            args = readArgumentList();  
11. ​            descriptors = mSocket.getAncillaryFileDescriptors();  
12. ​        } catch (IOException ex) {  
13. ​            ......  
14. ​            return true;  
15. ​        }  
16.   
17. ​        ......  
18.   
19. ​        /** the stderr of the most recent request, if avail */  
20. ​        PrintStream newStderr = null;  
21.   
22. ​        if (descriptors != null && descriptors.length >= 3) {  
23. ​            newStderr = new PrintStream(  
24. ​                new FileOutputStream(descriptors[2]));  
25. ​        }  
26.   
27. ​        int pid;  
28. ​          
29. ​        try {  
30. ​            parsedArgs = new Arguments(args);  
31.   
32. ​            applyUidSecurityPolicy(parsedArgs, peer);  
33. ​            applyDebuggerSecurityPolicy(parsedArgs);  
34. ​            applyRlimitSecurityPolicy(parsedArgs, peer);  
35. ​            applyCapabilitiesSecurityPolicy(parsedArgs, peer);  
36.   
37. ​            int[][] rlimits = null;  
38.   
39. ​            if (parsedArgs.rlimits != null) {  
40. ​                rlimits = parsedArgs.rlimits.toArray(intArray2d);  
41. ​            }  
42.   
43. ​            pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid,  
44. ​                parsedArgs.gids, parsedArgs.debugFlags, rlimits);  
45. ​        } catch (IllegalArgumentException ex) {  
46. ​            ......  
47. ​        } catch (ZygoteSecurityException ex) {  
48. ​            ......  
49. ​        }  
50.   
51. ​        if (pid == 0) {  
52. ​            // in child  
53. ​            handleChildProc(parsedArgs, descriptors, newStderr);  
54. ​            // should never happen  
55. ​            return true;  
56. ​        } else { /* pid != 0 */  
57. ​            // in parent...pid of < 0 means failure  
58. ​            return handleParentProc(pid, descriptors, parsedArgs);  
59. ​        }  
60. ​    }  
61.   
62. ​    ......  
63. }  

​         真正创建进程的地方就是在这里了：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid,  
2. ​    parsedArgs.gids, parsedArgs.debugFlags, rlimits);  

​        有Linux开发经验的读者很容易看懂这个函数调用，这个函数会创建一个进程，而且有两个返回值，一个是在当前进程中返回的，一个是在新创建的进程中返回，即在当前进程的子进程中返回，在当前进程中的返回值就是新创建的子进程的pid值，而在子进程中的返回值是0。因为我们只关心创建的新进程的情况，因此，我们沿着子进程的执行路径继续看下去：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1.    if (pid == 0) {  
2. // in child  
3. handleChildProc(parsedArgs, descriptors, newStderr);  
4. // should never happen  
5. return true;  
6.    } else { /* pid != 0 */  
7. ......  
8.    }  

​        这里就是调用handleChildProc函数了。

​        Step 7. ZygoteConnection.handleChildProc
​        这个函数定义在frameworks/base/core/java/com/android/internal/os/ZygoteConnection.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. class ZygoteConnection {  
2. ​    ......  
3.   
4. ​    private void handleChildProc(Arguments parsedArgs,  
5. ​            FileDescriptor[] descriptors, PrintStream newStderr)  
6. ​            throws ZygoteInit.MethodAndArgsCaller {  
7. ​        ......  
8.   
9. ​        if (parsedArgs.runtimeInit) {  
10. ​            RuntimeInit.zygoteInit(parsedArgs.remainingArgs);  
11. ​        } else {  
12. ​            ......  
13. ​        }  
14. ​    }  
15.   
16. ​    ......  
17. }  

​        由于在前面的Step 3中，指定了"--runtime-init"参数，表示要为新创建的进程初始化运行时库，因此，这里的parseArgs.runtimeInit值为true，于是就继续执行RuntimeInit.zygoteInit进一步处理了。

​        Step 8. RuntimeInit.zygoteInit

​        这个函数定义在frameworks/base/core/java/com/android/internal/os/RuntimeInit.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. public class RuntimeInit {  
2. ​    ......  
3.   
4. ​    public static final void zygoteInit(String[] argv)  
5. ​            throws ZygoteInit.MethodAndArgsCaller {  
6. ​        // TODO: Doing this here works, but it seems kind of arbitrary. Find  
7. ​        // a better place. The goal is to set it up for applications, but not  
8. ​        // tools like am.  
9. ​        System.setOut(new AndroidPrintStream(Log.INFO, "System.out"));  
10. ​        System.setErr(new AndroidPrintStream(Log.WARN, "System.err"));  
11.   
12. ​        commonInit();  
13. ​        zygoteInitNative();  
14.   
15. ​        int curArg = 0;  
16. ​        for ( /* curArg */ ; curArg < argv.length; curArg++) {  
17. ​            String arg = argv[curArg];  
18.   
19. ​            if (arg.equals("--")) {  
20. ​                curArg++;  
21. ​                break;  
22. ​            } else if (!arg.startsWith("--")) {  
23. ​                break;  
24. ​            } else if (arg.startsWith("--nice-name=")) {  
25. ​                String niceName = arg.substring(arg.indexOf('=') + 1);  
26. ​                Process.setArgV0(niceName);  
27. ​            }  
28. ​        }  
29.   
30. ​        if (curArg == argv.length) {  
31. ​            Slog.e(TAG, "Missing classname argument to RuntimeInit!");  
32. ​            // let the process exit  
33. ​            return;  
34. ​        }  
35.   
36. ​        // Remaining arguments are passed to the start class's static main  
37.   
38. ​        String startClass = argv[curArg++];  
39. ​        String[] startArgs = new String[argv.length - curArg];  
40.   
41. ​        System.arraycopy(argv, curArg, startArgs, 0, startArgs.length);  
42. ​        invokeStaticMain(startClass, startArgs);  
43. ​    }  
44.   
45. ​    ......  
46. }  

​        这里有两个关键的函数调用，一个是zygoteInitNative函数调用，一个是invokeStaticMain函数调用，前者就是执行Binder驱动程序初始化的相关工作了，正是由于执行了这个工作，才使得进程中的Binder对象能够顺利地进行Binder进程间通信，而后一个函数调用，就是执行进程的入口函数，这里就是执行startClass类的main函数了，而这个startClass即是我们在Step 1中传进来的"android.app.ActivityThread"值，表示要执行android.app.ActivityThread类的main函数。

​        我们先来看一下zygoteInitNative函数的调用过程，然后再回到RuntimeInit.zygoteInit函数中来，看看它是如何调用android.app.ActivityThread类的main函数的。

​        step 9. RuntimeInit.zygoteInitNative

​        这个函数定义在frameworks/base/core/java/com/android/internal/os/RuntimeInit.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. public class RuntimeInit {  
2. ​    ......  
3.   
4. ​    public static final native void zygoteInitNative();  
5.   
6. ​    ......  
7. }  

​        这里可以看出，函数zygoteInitNative是一个Native函数，实现在frameworks/base/core/jni/AndroidRuntime.cpp文件中：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. static void com_android_internal_os_RuntimeInit_zygoteInit(JNIEnv* env, jobject clazz)  
2. {  
3. ​    gCurRuntime->onZygoteInit();  
4. }  

​        这里它调用了全局变量gCurRuntime的onZygoteInit函数，这个全局变量的定义在frameworks/base/core/jni/AndroidRuntime.cpp文件开头的地方：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. static AndroidRuntime* gCurRuntime = NULL;  

​        这里可以看出，它的类型为AndroidRuntime，它是在AndroidRuntime类的构造函数中初始化的，AndroidRuntime类的构造函数也是定义在frameworks/base/core/jni/AndroidRuntime.cpp文件中：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. AndroidRuntime::AndroidRuntime()  
2. {  
3. ​    ......  
4.   
5. ​    assert(gCurRuntime == NULL);        // one per process  
6. ​    gCurRuntime = this;  
7. }  

​        那么这个AndroidRuntime类的构造函数又是什么时候被调用的呢？AndroidRuntime类的声明在frameworks/base/include/android_runtime/AndroidRuntime.h文件中，如果我们打开这个文件会看到，它是一个虚拟类，也就是我们不能直接创建一个AndroidRuntime对象，只能用一个AndroidRuntime类的指针来指向它的某一个子类，这个子类就是AppRuntime了，它定义在frameworks/base/cmds/app_process/app_main.cpp文件中：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. int main(int argc, const char* const argv[])  
2. {  
3. ​    ......  
4.   
5. ​    AppRuntime runtime;  
6. ​      
7. ​    ......  
8. }  

​        而AppRuntime类继续了AndroidRuntime类，它也是定义在frameworks/base/cmds/app_process/app_main.cpp文件中：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. class AppRuntime : public AndroidRuntime  
2. {  
3. ​    ......  
4.   
5. };  

​        因此，在前面的com_android_internal_os_RuntimeInit_zygoteInit函数，实际是执行了AppRuntime类的onZygoteInit函数。

​        Step 10. AppRuntime.onZygoteInit
​        这个函数定义在frameworks/base/cmds/app_process/app_main.cpp文件中：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. class AppRuntime : public AndroidRuntime  
2. {  
3. ​    ......  
4.   
5. ​    virtual void onZygoteInit()  
6. ​    {  
7. ​        sp<ProcessState> proc = ProcessState::self();  
8. ​        if (proc->supportsProcesses()) {  
9. ​            LOGV("App process: starting thread pool.\n");  
10. ​            proc->startThreadPool();  
11. ​        }  
12. ​    }  
13.   
14. ​    ......  
15. };  

​        这里它就是调用ProcessState::startThreadPool启动线程池了，这个线程池中的线程就是用来和Binder驱动程序进行交互的了。

​        Step 11. ProcessState.startThreadPool

​        这个函数定义在frameworks/base/libs/binder/ProcessState.cpp文件中：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. void ProcessState::startThreadPool()  
2. {  
3. ​    AutoMutex _l(mLock);  
4. ​    if (!mThreadPoolStarted) {  
5. ​        mThreadPoolStarted = true;  
6. ​        spawnPooledThread(true);  
7. ​    }  
8. }  

​        ProcessState类是Binder进程间通信机制的一个基础组件，它的作用可以参考

浅谈Android系统进程间通信（IPC）机制Binder中的Server和Client获得Service Manager接口之路

、

Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析

和

Android系统进程间通信（IPC）机制Binder中的Client获得Server远程接口过程源代码分析

这三篇文章。这里它调用spawnPooledThread函数进一步处理。

​        Step 12. ProcessState.spawnPooledThread

​        这个函数定义在frameworks/base/libs/binder/ProcessState.cpp文件中：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. void ProcessState::spawnPooledThread(bool isMain)  
2. {  
3. ​    if (mThreadPoolStarted) {  
4. ​        int32_t s = android_atomic_add(1, &mThreadPoolSeq);  
5. ​        char buf[32];  
6. ​        sprintf(buf, "Binder Thread #%d", s);  
7. ​        LOGV("Spawning new pooled thread, name=%s\n", buf);  
8. ​        sp<Thread> t = new PoolThread(isMain);  
9. ​        t->run(buf);  
10. ​    }  
11. }  

​        这里它会创建一个PoolThread线程类，然后执行它的run函数，最终就会执行PoolThread类的threadLoop函数了。

​        Step 13. PoolThread.threadLoop

​        这个函数定义在frameworks/base/libs/binder/ProcessState.cpp文件中：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. class PoolThread : public Thread  
2. {  
3. public:  
4. ​    PoolThread(bool isMain)  
5. ​        : mIsMain(isMain)  
6. ​    {  
7. ​    }  
8.   
9. protected:  
10. ​    virtual bool threadLoop()  
11. ​    {  
12. ​        IPCThreadState::self()->joinThreadPool(mIsMain);  
13. ​        return false;  
14. ​    }  
15.   
16. ​    const bool mIsMain;  
17. };  

​        这里它执行了IPCThreadState::joinThreadPool函数进一步处理。IPCThreadState也是Binder进程间通信机制的一个基础组件，它的作用可以参考

浅谈Android系统进程间通信（IPC）机制Binder中的Server和Client获得Service Manager接口之路

、

Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析

和

Android系统进程间通信（IPC）机制Binder中的Client获得Server远程接口过程源代码分析

这三篇文章。

​        Step 14. IPCThreadState.joinThreadPool

​        这个函数定义在frameworks/base/libs/binder/IPCThreadState.cpp文件中：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. void IPCThreadState::joinThreadPool(bool isMain)  
2. {  
3. ​    ......  
4.   
5. ​    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);  
6.   
7. ​    ......  
8.   
9. ​    status_t result;  
10. ​    do {  
11. ​        int32_t cmd;  
12.   
13. ​        ......  
14.   
15. ​        // now get the next command to be processed, waiting if necessary  
16. ​        result = talkWithDriver();  
17. ​        if (result >= NO_ERROR) {  
18. ​            size_t IN = mIn.dataAvail();  
19. ​            if (IN < sizeof(int32_t)) continue;  
20. ​            cmd = mIn.readInt32();  
21. ​            ......  
22.   
23. ​            result = executeCommand(cmd);  
24. ​        }  
25.   
26. ​        ......  
27. ​    } while (result != -ECONNREFUSED && result != -EBADF);  
28.   
29. ​    ......  
30. ​      
31. ​    mOut.writeInt32(BC_EXIT_LOOPER);  
32. ​    talkWithDriver(false);  
33. }  

​        这个函数首先告诉Binder驱动程序，这条线程要进入循环了：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);  

​        然后在中间的while循环中通过talkWithDriver不断与Binder驱动程序进行交互，以便获得Client端的进程间调用：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. result = talkWithDriver();  

​        获得了Client端的进程间调用后，就调用excuteCommand函数来处理这个请求：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. result = executeCommand(cmd);  

​        最后，线程退出时，也会告诉Binder驱动程序，它退出了，这样Binder驱动程序就不会再在Client端的进程间调用分发给它了：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. mOut.writeInt32(BC_EXIT_LOOPER);  
2. talkWithDriver(false);  

​        我们再来看看talkWithDriver函数的实现。

​        Step 15. talkWithDriver

​        这个函数定义在frameworks/base/libs/binder/IPCThreadState.cpp文件中：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. status_t IPCThreadState::talkWithDriver(bool doReceive)  
2. {  
3. ​    ......  
4.   
5. ​    binder_write_read bwr;  
6.   
7. ​    // Is the read buffer empty?  
8. ​    const bool needRead = mIn.dataPosition() >= mIn.dataSize();  
9.   
10. ​    // We don't want to write anything if we are still reading  
11. ​    // from data left in the input buffer and the caller  
12. ​    // has requested to read the next data.  
13. ​    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;  
14.   
15. ​    bwr.write_size = outAvail;  
16. ​    bwr.write_buffer = (long unsigned int)mOut.data();  
17.   
18. ​    // This is what we'll read.  
19. ​    if (doReceive && needRead) {  
20. ​        bwr.read_size = mIn.dataCapacity();  
21. ​        bwr.read_buffer = (long unsigned int)mIn.data();  
22. ​    } else {  
23. ​        bwr.read_size = 0;  
24. ​    }  
25.   
26. ​    ......  
27.   
28. ​    // Return immediately if there is nothing to do.  
29. ​    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;  
30.   
31. ​    bwr.write_consumed = 0;  
32. ​    bwr.read_consumed = 0;  
33. ​    status_t err;  
34. ​    do {  
35. ​        ......  
36. \#if defined(HAVE_ANDROID_OS)  
37. ​        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)  
38. ​            err = NO_ERROR;  
39. ​        else  
40. ​            err = -errno;  
41. \#else  
42. ​        err = INVALID_OPERATION;  
43. \#endif  
44. ​        ......  
45. ​        }  
46. ​    } while (err == -EINTR);  
47.   
48. ​    ....  
49.   
50. ​    if (err >= NO_ERROR) {  
51. ​        if (bwr..write_consumed > 0) {  
52. ​            if (bwr.write_consumed < (ssize_t)mOut.dataSize())  
53. ​                mOut.remove(0, bwr.write_consumed);  
54. ​            else  
55. ​                mOut.setDataSize(0);  
56. ​        }  
57. ​        if (bwr.read_consumed > 0) {  
58. ​            mIn.setDataSize(bwr.read_consumed);  
59. ​            mIn.setDataPosition(0);  
60. ​        }  
61. ​        ......  
62. ​        return NO_ERROR;  
63. ​    }  
64.   
65. ​    return err;  
66. }  

​        这个函数的具体作用可以参考

Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析

一文，它只要就是通过ioctl文件操作函数来和Binder驱动程序交互的了：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)  

​        有了这个线程池之后，我们在开发Android应用程序的时候，当我们要和其它进程中进行通信时，只要定义自己的Binder对象，然后把这个Binder对象的远程接口通过其它途径传给其它进程后，其它进程就可以通过这个Binder对象的远程接口来调用我们的应用程序进程的函数了，它不像我们在C++层实现Binder进程间通信机制的Server时，必须要手动调用IPCThreadState.joinThreadPool函数来进入一个无限循环中与Binder驱动程序交互以便获得Client端的请求，这样就实现了我们在文章开头处说的Android应用程序进程天然地支持Binder进程间通信机制。

​        细心的读者可能会发现，从Step 1到Step 9，都是在Android应用程序框架层运行的，而从Step 10到Step 15，都是在Android系统运行时库层运行的，这两个层次中的Binder进程间通信机制的接口一个是用Java来实现的，而别一个是用C++来实现的，这两者是如何协作的呢？这就是通过JNI层来实现的了，具体可以参考[Android系统进程间通信Binder机制在应用程序框架层的Java接口源代码分析](http://blog.csdn.net/luoshengyang/article/details/6642463)一文。

​        回到Step 8中的RuntimeInit.zygoteInit函数中，在初始化完成Binder进程间通信机制的基础设施后，它接着就要进入进程的入口函数了。

​        Step 16. RuntimeInit.invokeStaticMain

​        这个函数定义在frameworks/base/core/java/com/android/internal/os/RuntimeInit.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. public class ZygoteInit {  
2. ​    ......  
3.   
4. ​    static void invokeStaticMain(ClassLoader loader,  
5. ​            String className, String[] argv)  
6. ​            throws ZygoteInit.MethodAndArgsCaller {  
7. ​        Class<?> cl;  
8.   
9. ​        try {  
10. ​            cl = loader.loadClass(className);  
11. ​        } catch (ClassNotFoundException ex) {  
12. ​            ......  
13. ​        }  
14.   
15. ​        Method m;  
16. ​        try {  
17. ​            m = cl.getMethod("main", new Class[] { String[].class });  
18. ​        } catch (NoSuchMethodException ex) {  
19. ​            ......  
20. ​        } catch (SecurityException ex) {  
21. ​            ......  
22. ​        }  
23.   
24. ​        int modifiers = m.getModifiers();  
25. ​        ......  
26.   
27. ​        /* 
28. ​        * This throw gets caught in ZygoteInit.main(), which responds 
29. ​        * by invoking the exception's run() method. This arrangement 
30. ​        * clears up all the stack frames that were required in setting 
31. ​        * up the process. 
32. ​        */  
33. ​        throw new ZygoteInit.MethodAndArgsCaller(m, argv);  
34. ​    }  
35.   
36. ​    ......  
37. }  

​        前面我们说过，这里传进来的参数className字符串值为"android.app.ActivityThread"，这里就通ClassLoader.loadClass函数将它加载到进程中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. cl = loader.loadClass(className);  

​        然后获得它的静态成员函数main：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. m = cl.getMethod("main", new Class[] { String[].class });  

​        函数最后并没有直接调用这个静态成员函数main，而是通过抛出一个异常ZygoteInit.MethodAndArgsCaller，然后让ZygoteInit.main函数在捕获这个异常的时候再调用android.app.ActivityThread类的main函数。为什么要这样做呢？注释里面已经讲得很清楚了，它是为了清理堆栈的，这样就会让android.app.ActivityThread类的main函数觉得自己是进程的入口函数，而事实上，在执行android.app.ActivityThread类的main函数之前，已经做了大量的工作了。

​        我们看看ZygoteInit.main函数在捕获到这个异常的时候做了什么事：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. public class ZygoteInit {  
2. ​    ......  
3.   
4. ​    public static void main(String argv[]) {  
5. ​        try {  
6. ​            ......  
7. ​        } catch (MethodAndArgsCaller caller) {  
8. ​            caller.run();  
9. ​        } catch (RuntimeException ex) {  
10. ​            ......  
11. ​        }  
12. ​    }  
13.   
14. ​    ......  
15. }  

​        它执行MethodAndArgsCaller的run函数：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. public class ZygoteInit {  
2. ​    ......  
3.   
4. ​    public static class MethodAndArgsCaller extends Exception  
5. ​            implements Runnable {  
6. ​        /** method to call */  
7. ​        private final Method mMethod;  
8.   
9. ​        /** argument array */  
10. ​        private final String[] mArgs;  
11.   
12. ​        public MethodAndArgsCaller(Method method, String[] args) {  
13. ​            mMethod = method;  
14. ​            mArgs = args;  
15. ​        }  
16.   
17. ​        public void run() {  
18. ​            try {  
19. ​                mMethod.invoke(null, new Object[] { mArgs });  
20. ​            } catch (IllegalAccessException ex) {  
21. ​                ......  
22. ​            } catch (InvocationTargetException ex) {  
23. ​                ......  
24. ​            }  
25. ​        }  
26. ​    }  
27.   
28. ​    ......  
29. }  

​        这里的成员变量mMethod和mArgs都是在前面构造异常对象时传进来的，这里的mMethod就对应android.app.ActivityThread类的main函数了，于是最后就通过下面语句执行这个函数：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. mMethod.invoke(null, new Object[] { mArgs });  

​        这样，android.app.ActivityThread类的main函数就被执行了。

​        Step 17. ActivityThread.main

​        这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. public final class ActivityThread {  
2. ​    ......  
3.   
4. ​    public static final void main(String[] args) {  
5. ​        SamplingProfilerIntegration.start();  
6.   
7. ​        Process.setArgV0("<pre-initialized>");  
8.   
9. ​        Looper.prepareMainLooper();  
10. ​        if (sMainThreadHandler == null) {  
11. ​            sMainThreadHandler = new Handler();  
12. ​        }  
13.   
14. ​        ActivityThread thread = new ActivityThread();  
15. ​        thread.attach(false);  
16.   
17. ​        if (false) {  
18. ​            Looper.myLooper().setMessageLogging(new  
19. ​                LogPrinter(Log.DEBUG, "ActivityThread"));  
20. ​        }  
21. ​        Looper.loop();  
22.   
23. ​        if (Process.supportsProcesses()) {  
24. ​            throw new RuntimeException("Main thread loop unexpectedly exited");  
25. ​        }  
26.   
27. ​        thread.detach();  
28. ​        String name = (thread.mInitialApplication != null)  
29. ​            ? thread.mInitialApplication.getPackageName()  
30. ​            : "<unknown>";  
31. ​        Slog.i(TAG, "Main thread of " + name + " is now exiting");  
32. ​    }  
33.   
34. ​    ......  
35. }  

​        从这里我们可以看出，这个函数首先会在进程中创建一个ActivityThread对象：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. ActivityThread thread = new ActivityThread();  

​        然后进入消息循环中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6747696#) [copy](http://blog.csdn.net/luoshengyang/article/details/6747696#)

1. Looper.loop();  

​        这样，我们以后就可以在这个进程中启动Activity或者Service了。

​        至此，Android应用程序进程启动过程的源代码就分析完成了，它除了指定新的进程的入口函数是ActivityThread的main函数之外，还为进程内的Binder对象提供了Binder进程间通信机制的基础设施，由此可见，Binder进程间通信机制在Android系统中是何等的重要，而且是无处不在，想进一步学习Android系统的Binder进程间通信机制，请参考[Android进程间通信（IPC）机制Binder简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6618363)一文。