​    [Android](http://lib.csdn.net/base/android)应用程序组件Service与Activity一样，既可以在新的进程中启动，也可以在应用程序进程内部启动；前面我们已经分析了在新的进程中启动Service的过程，本文将要介绍在应用程序内部绑定Service的过程，这是一种在应用程序进程内部启动Service的方法。

《[android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

​        在前面一篇文章[Android进程间通信（IPC）机制Binder简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6618363)中，我们就曾经提到，在Android系统中，每一个应用程序都是由一些Activity和Service组成的，一般Service运行在独立的进程中，而Activity有可能运行在同一个进程中，也有可能运行在不同的进程中；在接下来的文章中，[Android系统在新进程中启动自定义服务过程（startService）的原理分析](http://blog.csdn.net/luoshengyang/article/details/6677029)一文介绍了在新的进程中启动Service的过程，[Android应用程序启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6689748)一文介绍了在新的进程中启动Activity的过程，而[Android应用程序内部启动Activity过程（startActivity）的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6703247)一文则介绍了在应用程序进程内部启动Activity的过程；本文接过最后一棒，继续介绍在应用程序进程内部启动Service的过程，这种过程又可以称在应用程序进程内部绑定服务（bindService）的过程，这样，读者应该就可以对Android应用程序启动Activity和Service有一个充分的认识了。

​        这里仍然是按照老规矩，通过具体的例子来分析Android应用程序绑定Service的过程，而所使用的例子便是前面我们在介绍Android系统广播机制的一篇文章[Android系统中的广播（Broadcast）机制简要介绍和学习计划中](http://blog.csdn.net/luoshengyang/article/details/6730748)所开发的应用程序Broadcast了。

​        我们先简单回顾一下这个应用程序实例绑定Service的过程。在这个应用程序的MainActivity的onCreate函数中，会调用bindService来绑定一个计数器服务CounterService，这里绑定的意思其实就是在MainActivity内部获得CounterService的接口，所以，这个过程的第一步就是要把CounterService启动起来。当CounterService的onCreate函数被调用起来了，就说明CounterService已经启动起来了，接下来系统还要调用CounterService的onBind函数，跟CounterService要一个Binder对象，这个Binder对象是在CounterService内部自定义的CounterBinder类的一个实例，它继承于Binder类，里面实现一个getService函数，用来返回外部的CounterService接口。系统得到这个Binder对象之后，就会调用MainActivity在bindService函数里面传过来的ServiceConnection实例的onServiceConnected函数，并把这个Binder对象以参数的形式传到onServiceConnected函数里面，于是，MainActivity就可以调用这个Binder对象的getService函数来获得CounterService的接口了。

​        这个过程比较复杂，但总体来说，思路还是比较清晰的，整个调用过程为MainActivity.bindService->CounterService.onCreate->CounterService.onBind->MainActivity.ServiceConnection.onServiceConnection->CounterService.CounterBinder.getService。下面，我们就先用一个序列图来总体描述这个服务绑定的过程，然后就具体分析每一个步骤。

![img](http://hi.csdn.net/attachment/201109/3/0_1315064144wnUc.gif)

[点击查看大图](http://hi.csdn.net/attachment/201109/3/0_1315064144wnUc.gif)

​        Step 1. ContextWrapper.bindService

​        这个函数定义在frameworks/base/core/[Java](http://lib.csdn.net/base/java)/android/content/ContextWrapper.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. public class ContextWrapper extends Context {  
2. ​    Context mBase;  
3. ​    ......  
4.   
5. ​    @Override  
6. ​    public boolean bindService(Intent service, ServiceConnection conn,  
7. ​            int flags) {  
8. ​        return mBase.bindService(service, conn, flags);  
9. ​    }  
10.   
11. ​    ......  
12. }  

​        这里的mBase是一个ContextImpl实例变量，于是就调用ContextImpl的bindService函数来进一步处理。

​        Step 2. ContextImpl.bindService

​        这个函数定义在frameworks/base/core/java/android/app/ContextImpl.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. class ContextImpl extends Context {  
2. ​    ......  
3.   
4. ​    @Override  
5. ​    public boolean bindService(Intent service, ServiceConnection conn,  
6. ​            int flags) {  
7. ​        IServiceConnection sd;  
8. ​        if (mPackageInfo != null) {  
9. ​            sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(),  
10. ​                mMainThread.getHandler(), flags);  
11. ​        } else {  
12. ​            ......  
13. ​        }  
14. ​        try {  
15. ​            int res = ActivityManagerNative.getDefault().bindService(  
16. ​                mMainThread.getApplicationThread(), getActivityToken(),  
17. ​                service, service.resolveTypeIfNeeded(getContentResolver()),  
18. ​                sd, flags);  
19. ​            ......  
20. ​            return res != 0;  
21. ​        } catch (RemoteException e) {  
22. ​            return false;  
23. ​        }  
24. ​    }  
25.   
26. ​    ......  
27.   
28. }  

​        这里的mMainThread是一个ActivityThread实例，通过它的getHandler函数可以获得一个Handler对象，有了这个Handler对象后，就可以把消息分发到ActivityThread所在的线程消息队列中去了，后面我们将会看到这个用法，现在我们暂时不关注，只要知道这里从ActivityThread处获得了一个Handler并且保存在下面要介绍的ServiceDispatcher中去就可以了。

​        我们先看一下ActivityThread.getHandler的实现，然后再回到这里的bindService函数来。

​        Step 3. ActivityThread.getHandler

​        这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. public final class ActivityThread {  
2. ​    ......  
3.   
4. ​    final H mH = new H();  
5.   
6. ​    ......  
7.   
8. ​    private final class H extends Handler {  
9. ​        ......  
10.   
11. ​        public void handleMessage(Message msg) {  
12. ​            ......  
13. ​        }  
14.   
15. ​        ......  
16. ​    }  
17.   
18. ​    ......  
19.   
20. ​    final Handler getHandler() {  
21. ​        return mH;  
22. ​    }  
23.   
24. ​    ......  
25. }  

​        这里返回的Handler是在ActivityThread类内部从Handler类继承下来的一个H类实例变量。

​        回到Step 2中的ContextImpl.bindService函数中，获得了这个Handler对象后，就调用mPackageInfo.getServiceDispatcher函数来获得一个IServiceConnection接口，这里的mPackageInfo的类型是LoadedApk，我们来看看它的getServiceDispatcher函数的实现，然后再回到ContextImpl.bindService函数来。

​        Step 4. LoadedApk.getServiceDispatcher

​        这个函数定义在frameworks/base/core/java/android/app/LoadedApk.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. final class LoadedApk {  
2. ​    ......  
3.   
4. ​    public final IServiceConnection getServiceDispatcher(ServiceConnection c,  
5. ​            Context context, Handler handler, int flags) {  
6. ​        synchronized (mServices) {  
7. ​            LoadedApk.ServiceDispatcher sd = null;  
8. ​            HashMap<ServiceConnection, LoadedApk.ServiceDispatcher> map = mServices.get(context);  
9. ​            if (map != null) {  
10. ​                sd = map.get(c);  
11. ​            }  
12. ​            if (sd == null) {  
13. ​                sd = new ServiceDispatcher(c, context, handler, flags);  
14. ​                if (map == null) {  
15. ​                    map = new HashMap<ServiceConnection, LoadedApk.ServiceDispatcher>();  
16. ​                    mServices.put(context, map);  
17. ​                }  
18. ​                map.put(c, sd);  
19. ​            } else {  
20. ​                sd.validate(context, handler);  
21. ​            }  
22. ​            return sd.getIServiceConnection();  
23. ​        }  
24. ​    }  
25.   
26. ​    ......  
27.   
28. ​    static final class ServiceDispatcher {  
29. ​        private final ServiceDispatcher.InnerConnection mIServiceConnection;  
30. ​        private final ServiceConnection mConnection;  
31. ​        private final Handler mActivityThread;  
32. ​        ......  
33.   
34. ​        private static class InnerConnection extends IServiceConnection.Stub {  
35. ​            final WeakReference<LoadedApk.ServiceDispatcher> mDispatcher;  
36. ​            ......  
37.   
38. ​            InnerConnection(LoadedApk.ServiceDispatcher sd) {  
39. ​                mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);  
40. ​            }  
41.   
42. ​            ......  
43. ​        }  
44.   
45. ​        ......  
46.   
47. ​        ServiceDispatcher(ServiceConnection conn,  
48. ​                Context context, Handler activityThread, int flags) {  
49. ​            mIServiceConnection = new InnerConnection(this);  
50. ​            mConnection = conn;  
51. ​            mActivityThread = activityThread;  
52. ​            ......  
53. ​        }  
54.   
55. ​        ......  
56.   
57. ​        IServiceConnection getIServiceConnection() {  
58. ​            return mIServiceConnection;  
59. ​        }  
60.   
61. ​        ......  
62. ​    }  
63.   
64. ​    ......  
65. }  

​         在getServiceDispatcher函数中，传进来的参数context是一个MainActivity实例，先以它为Key值在mServices中查看一下，是不是已经存在相应的ServiceDispatcher实例，如果有了，就不用创建了，直接取出来。在我们这个情景中，需要创建一个新的ServiceDispatcher。在创建新的ServiceDispatcher实例的过程中，将上面传下来ServiceConnection参数c和Hanlder参数保存在了ServiceDispatcher实例的内部，并且创建了一个InnerConnection对象，这是一个Binder对象，一会是要传递给ActivityManagerService的，ActivityManagerServic后续就是要通过这个Binder对象和ServiceConnection通信的。

​        函数getServiceDispatcher最后就是返回了一个InnerConnection对象给ContextImpl.bindService函数。回到ContextImpl.bindService函数中，它接着就要调用ActivityManagerService的远程接口来进一步处理了。

​       Step 5. ActivityManagerService.bindService

​       这个函数定义在frameworks/base/core/java/android/app/ActivityManagerNative.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. class ActivityManagerProxy implements IActivityManager  
2. {  
3. ​    ......  
4.   
5. ​    public int bindService(IApplicationThread caller, IBinder token,  
6. ​            Intent service, String resolvedType, IServiceConnection connection,  
7. ​            int flags) throws RemoteException {  
8. ​        Parcel data = Parcel.obtain();  
9. ​        Parcel reply = Parcel.obtain();  
10. ​        data.writeInterfaceToken(IActivityManager.descriptor);  
11. ​        data.writeStrongBinder(caller != null ? caller.asBinder() : null);  
12. ​        data.writeStrongBinder(token);  
13. ​        service.writeToParcel(data, 0);  
14. ​        data.writeString(resolvedType);  
15. ​        data.writeStrongBinder(connection.asBinder());  
16. ​        data.writeInt(flags);  
17. ​        mRemote.transact(BIND_SERVICE_TRANSACTION, data, reply, 0);  
18. ​        reply.readException();  
19. ​        int res = reply.readInt();  
20. ​        data.recycle();  
21. ​        reply.recycle();  
22. ​        return res;  
23. ​    }  
24.   
25. ​    ......  
26. }  

​         这个函数通过Binder驱动程序就进入到ActivityManagerService的bindService函数去了。

​         Step 6. ActivityManagerService.bindService

​         这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. public final class ActivityManagerService extends ActivityManagerNative  
2. ​        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
3. ​    ......  
4.   
5. ​    public int bindService(IApplicationThread caller, IBinder token,  
6. ​            Intent service, String resolvedType,  
7. ​            IServiceConnection connection, int flags) {  
8. ​        ......  
9.   
10. ​        synchronized(this) {  
11. ​            ......  
12. ​            final ProcessRecord callerApp = getRecordForAppLocked(caller);  
13. ​            ......  
14.   
15. ​            ActivityRecord activity = null;  
16. ​            if (token != null) {  
17. ​                int aindex = mMainStack.indexOfTokenLocked(token);  
18. ​                ......  
19. ​                activity = (ActivityRecord)mMainStack.mHistory.get(aindex);  
20. ​            }  
21. ​              
22. ​            ......  
23.   
24. ​            ServiceLookupResult res =  
25. ​                retrieveServiceLocked(service, resolvedType,  
26. ​                Binder.getCallingPid(), Binder.getCallingUid());  
27. ​              
28. ​            ......  
29.   
30. ​            ServiceRecord s = res.record;  
31.   
32. ​            final long origId = Binder.clearCallingIdentity();  
33.   
34. ​            ......  
35.   
36. ​            AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);  
37. ​            ConnectionRecord c = new ConnectionRecord(b, activity,  
38. ​                connection, flags, clientLabel, clientIntent);  
39.   
40. ​            IBinder binder = connection.asBinder();  
41. ​            ArrayList<ConnectionRecord> clist = s.connections.get(binder);  
42.   
43. ​            if (clist == null) {  
44. ​                clist = new ArrayList<ConnectionRecord>();  
45. ​                s.connections.put(binder, clist);  
46. ​            }  
47. ​            clist.add(c);  
48. ​            b.connections.add(c);  
49. ​            if (activity != null) {  
50. ​                if (activity.connections == null) {  
51. ​                    activity.connections = new HashSet<ConnectionRecord>();  
52. ​                }  
53. ​                activity.connections.add(c);  
54. ​            }  
55. ​            b.client.connections.add(c);  
56. ​            clist = mServiceConnections.get(binder);  
57. ​            if (clist == null) {  
58. ​                clist = new ArrayList<ConnectionRecord>();  
59. ​                mServiceConnections.put(binder, clist);  
60. ​            }  
61. ​          
62. ​            clist.add(c);  
63.   
64. ​            if ((flags&Context.BIND_AUTO_CREATE) != 0) {  
65. ​                ......  
66. ​                if (!bringUpServiceLocked(s, service.getFlags(), false)) {  
67. ​                    return 0;  
68. ​                }  
69. ​            }  
70.   
71. ​            ......  
72. ​        }  
73.   
74. ​        return 1;  
75. ​    }             
76.   
77. ​    ......  
78. }  

​         函数首先根据传进来的参数token是MainActivity在ActivityManagerService里面的一个令牌，通过这个令牌就可以将这个代表MainActivity的ActivityRecord取回来了。

​        接着通过retrieveServiceLocked函数，得到一个ServiceRecord，这个ServiceReocrd描述的是一个Service对象，这里就是CounterService了，这是根据传进来的参数service的内容获得的。回忆一下在MainActivity.onCreate函数绑定服务的语句：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. Intent bindIntent = new Intent(MainActivity.this, CounterService.class);  
2. bindService(bindIntent, serviceConnection, Context.BIND_AUTO_CREATE);  

​        这里的参数service，就是上面的bindIntent了，它里面设置了CounterService类的信息（CounterService.class），因此，这里可以通过它来把CounterService的信息取出来，并且保存在ServiceRecord对象s中。

​        接下来，就是把传进来的参数connection封装成一个ConnectionRecord对象。注意，这里的参数connection是一个Binder对象，它的类型是LoadedApk.ServiceDispatcher.InnerConnection，是在Step 4中创建的，后续ActivityManagerService就是要通过它来告诉MainActivity，CounterService已经启动起来了，因此，这里要把这个ConnectionRecord变量c保存下来，它保在在好几个地方，都是为了后面要用时方便地取回来的，这里就不仔细去研究了，只要知道ActivityManagerService要使用它时就可以方便地把它取出来就可以了，具体后面我们再分析。

​        最后，传进来的参数flags的位Context.BIND_AUTO_CREATE为1（参见上面MainActivity.onCreate函数调用bindService函数时设置的参数），因此，这里会调用bringUpServiceLocked函数进一步处理。

​        Step 7. ActivityManagerService.bringUpServiceLocked

​        这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. public final class ActivityManagerService extends ActivityManagerNative  
2. ​        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
3. ​    ......  
4.   
5. ​    private final boolean bringUpServiceLocked(ServiceRecord r,  
6. ​            int intentFlags, boolean whileRestarting) {  
7. ​        ......  
8.   
9. ​        final String appName = r.processName;  
10. ​        ProcessRecord app = getProcessRecordLocked(appName, r.appInfo.uid);  
11.   
12. ​        if (app != null && app.thread != null) {  
13. ​            try {  
14. ​                realStartServiceLocked(r, app);  
15. ​                return true;  
16. ​            } catch (RemoteException e) {  
17. ​                ......  
18. ​            }  
19. ​        }  
20.   
21. ​        // Not running -- get it started, and enqueue this service record  
22. ​        // to be executed when the app comes up.  
23. ​        if (startProcessLocked(appName, r.appInfo, true, intentFlags,  
24. ​            "service", r.name, false) == null) {  
25. ​                ......  
26. ​        }  
27.   
28. ​        ......  
29. ​    }  
30.   
31. ​    ......  
32. }  

​        回忆在[Android系统中的广播（Broadcast）机制简要介绍和学习计划中](http://blog.csdn.net/luoshengyang/article/details/6730748)一文中，我们没有在程序的AndroidManifest.xml配置文件中设置CounterService的process属性值，因此，它默认就为application标签的process属性值，而application标签的process属性值也没有设置，于是，它们就默认为应用程序的包名了，即这里的appName的值为"shy.luo.broadcast"。接下来根据appName和应用程序的uid值获得一个ProcessRecord记录，由于之前在启动MainActivity的时候，已经根据这个appName和uid值创建了一个ProcessReocrd对象（具体可以参考[Android应用程序启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6689748)一文），因此，这里取回来的app和app.thread均不为null，于是，就执行realStartServiceLocked函数来执行下一步操作了。

​        如果这里得到的ProcessRecord变量app为null，又是什么情况呢？在这种情况下，就会执行后面的startProcessLocked函数来创建一个新的进程，然后在这个新的进程中启动这个Service了，具体可以参考前面一篇文章[Android系统在新进程中启动自定义服务过程（startService）的原理分析](http://blog.csdn.net/luoshengyang/article/details/6677029)。

​        Step 8. ActivityManagerService.realStartServiceLocked

​        这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. public final class ActivityManagerService extends ActivityManagerNative  
2. ​        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
3. ​    ......  
4.   
5. ​    private final void realStartServiceLocked(ServiceRecord r,  
6. ​            ProcessRecord app) throws RemoteException {  
7. ​        ......  
8. ​        r.app = app;  
9. ​        ......  
10.   
11. ​        app.services.add(r);  
12. ​        ......  
13.   
14. ​        try {  
15. ​            ......  
16. ​            app.thread.scheduleCreateService(r, r.serviceInfo);  
17. ​            ......  
18. ​        } finally {  
19. ​            ......  
20. ​        }  
21.   
22. ​        requestServiceBindingsLocked(r);  
23.   
24. ​        ......  
25. ​    }  
26.   
27. ​    ......  
28. }  

​        这个函数执行了两个操作，一个是操作是调用app.thread.scheduleCreateService函数来在应用程序进程内部启动CounterService，这个操作会导致CounterService的onCreate函数被调用；另一个操作是调用requestServiceBindingsLocked函数来向CounterService要一个Binder对象，这个操作会导致CounterService的onBind函数被调用。

​        这里，我们先沿着app.thread.scheduleCreateService这个路径分析下去，然后再回过头来分析requestServiceBindingsLocked的调用过程。这里的app.thread是一个Binder对象的远程接口，类型为ApplicationThreadProxy。每一个Android应用程序进程里面都有一个ActivtyThread对象和一个ApplicationThread对象，其中是ApplicationThread对象是ActivityThread对象的一个成员变量，是ActivityThread与ActivityManagerService之间用来执行进程间通信的，具体可以参考[Android应用程序启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6689748)一文。

​        Step 9. ApplicationThreadProxy.scheduleCreateService

​        这个函数定义在frameworks/base/core/java/android/app/ApplicationThreadNative.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. class ApplicationThreadProxy implements IApplicationThread {  
2. ​    ......  
3.   
4. ​    public final void scheduleCreateService(IBinder token, ServiceInfo info)  
5. ​            throws RemoteException {  
6. ​        Parcel data = Parcel.obtain();  
7. ​        data.writeInterfaceToken(IApplicationThread.descriptor);  
8. ​        data.writeStrongBinder(token);  
9. ​        info.writeToParcel(data, 0);  
10. ​        mRemote.transact(SCHEDULE_CREATE_SERVICE_TRANSACTION, data, null,  
11. ​            IBinder.FLAG_ONEWAY);  
12. ​        data.recycle();  
13. ​    }  
14.   
15. ​    ......  
16. }  

​         这里通过Binder驱动程序就进入到ApplicationThread的scheduleCreateService函数去了。

​         Step 10. ApplicationThread.scheduleCreateService
​         这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. public final class ActivityThread {  
2. ​    ......  
3.   
4. ​    private final class ApplicationThread extends ApplicationThreadNative {  
5. ​        ......  
6.   
7. ​        public final void scheduleCreateService(IBinder token,  
8. ​            ServiceInfo info) {  
9. ​            CreateServiceData s = new CreateServiceData();  
10. ​            s.token = token;  
11. ​            s.info = info;  
12.   
13. ​            queueOrSendMessage(H.CREATE_SERVICE, s);  
14. ​        }  
15.   
16. ​        ......  
17. ​    }  
18.   
19. ​    ......  
20. }  

​         这里它执行的操作就是调用ActivityThread的queueOrSendMessage函数把一个H.CREATE_SERVICE类型的消息放到ActivityThread的消息队列中去。

​         Step 11. ActivityThread.queueOrSendMessage
​         这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. public final class ActivityThread {  
2. ​    ......  
3.   
4. ​    // if the thread hasn't started yet, we don't have the handler, so just  
5. ​    // save the messages until we're ready.  
6. ​    private final void queueOrSendMessage(int what, Object obj) {  
7. ​        queueOrSendMessage(what, obj, 0, 0);  
8. ​    }  
9.   
10. ​    private final void queueOrSendMessage(int what, Object obj, int arg1) {  
11. ​        queueOrSendMessage(what, obj, arg1, 0);  
12. ​    }  
13.   
14. ​    private final void queueOrSendMessage(int what, Object obj, int arg1, int arg2) {  
15. ​        synchronized (this) {  
16. ​            ......  
17. ​            Message msg = Message.obtain();  
18. ​            msg.what = what;  
19. ​            msg.obj = obj;  
20. ​            msg.arg1 = arg1;  
21. ​            msg.arg2 = arg2;  
22. ​            mH.sendMessage(msg);  
23. ​        }  
24. ​    }  
25.   
26. ​    ......  
27. }  

​        这个消息最终是通过mH.sendMessage发送出去的，这里的mH是一个在ActivityThread内部定义的一个类，继承于Hanlder类，用于处理消息的。

​        Step 12. H.sendMessage

​        由于H类继承于Handler类，因此，这里实际执行的Handler.sendMessage函数，这个函数定义在frameworks/base/core/java/android/os/Handler.java文件，这里我们就不看了，有兴趣的读者可以自己研究一下，调用了这个函数之后，这个消息就真正地进入到ActivityThread的消息队列去了，最终这个消息由H.handleMessage函数来处理，这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. public final class ActivityThread {  
2. ​    ......  
3.   
4. ​    private final class H extends Handler {  
5. ​        ......  
6.   
7. ​        public void handleMessage(Message msg) {  
8. ​            ......  
9. ​            switch (msg.what) {  
10. ​            ......  
11. ​            case CREATE_SERVICE:  
12. ​                handleCreateService((CreateServiceData)msg.obj);  
13. ​                break;  
14. ​            ......  
15. ​            }  
16. ​        }  
17. ​    }  
18.   
19. ​    ......  
20. }  

​        这个消息最终由ActivityThread的handleCreateService函数来处理。

​        Step 13. ActivityThread.handleCreateService
​        这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. public final class ActivityThread {  
2. ​    ......  
3.   
4. ​    private final void handleCreateService(CreateServiceData data) {  
5. ​        ......  
6.   
7. ​        LoadedApk packageInfo = getPackageInfoNoCheck(  
8. ​        data.info.applicationInfo);  
9. ​        Service service = null;  
10. ​        try {  
11. ​            java.lang.ClassLoader cl = packageInfo.getClassLoader();  
12. ​            service = (Service) cl.loadClass(data.info.name).newInstance();  
13. ​        } catch (Exception e) {  
14. ​            ......  
15. ​        }  
16.   
17. ​        try {  
18. ​            ......  
19.   
20. ​            ContextImpl context = new ContextImpl();  
21. ​            context.init(packageInfo, null, this);  
22.   
23. ​            Application app = packageInfo.makeApplication(false, mInstrumentation);  
24. ​            context.setOuterContext(service);  
25. ​            service.attach(context, this, data.info.name, data.token, app,  
26. ​                ActivityManagerNative.getDefault());  
27.   
28. ​            service.onCreate();  
29. ​            mServices.put(data.token, service);  
30. ​            ......  
31. ​        } catch (Exception e) {  
32. ​            ......  
33. ​        }  
34. ​    }  
35.   
36. ​    ......  
37. }  

​         这个函数的工作就是把CounterService类加载到内存中来，然后调用它的onCreate函数。

​         Step 14. CounterService.onCreate

​         这个函数定义在[Android系统中的广播（Broadcast）机制简要介绍和学习计划中](http://blog.csdn.net/luoshengyang/article/details/6730748)一文中所介绍的应用程序Broadcast的工程目录下的src/shy/luo/broadcast/CounterService.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. public class CounterService extends Service implements ICounterService {  
2. ​    ......  
3.   
4. ​    @Override    
5. ​    public void onCreate() {    
6. ​        super.onCreate();    
7.   
8. ​        Log.i(LOG_TAG, "Counter Service Created.");    
9. ​    }   
10.   
11. ​    ......  
12. }  

​        这样，CounterService就启动起来了。

​        至此，应用程序绑定服务过程中的第一步MainActivity.bindService->CounterService.onCreate就完成了。

​        这一步完成之后，我们还要回到Step  8中去，执行下一个操作，即调用ActivityManagerService.requestServiceBindingsLocked函数，这个调用是用来执行CounterService的onBind函数的。

​        Step 15. ActivityManagerService.requestServiceBindingsLocked

​        这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. public final class ActivityManagerService extends ActivityManagerNative  
2. ​        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
3. ​    ......  
4.   
5. ​    private final void requestServiceBindingsLocked(ServiceRecord r) {  
6. ​        Iterator<IntentBindRecord> bindings = r.bindings.values().iterator();  
7. ​        while (bindings.hasNext()) {  
8. ​            IntentBindRecord i = bindings.next();  
9. ​            if (!requestServiceBindingLocked(r, i, false)) {  
10. ​                break;  
11. ​            }  
12. ​        }  
13. ​    }  
14.   
15. ​    private final boolean requestServiceBindingLocked(ServiceRecord r,  
16. ​            IntentBindRecord i, boolean rebind) {  
17. ​        ......  
18. ​        if ((!i.requested || rebind) && i.apps.size() > 0) {  
19. ​          try {  
20. ​              ......  
21. ​              r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind);  
22. ​              ......  
23. ​          } catch (RemoteException e) {  
24. ​              ......  
25. ​          }  
26. ​        }  
27. ​        return true;  
28. ​    }  
29.   
30. ​    ......  
31. }  

​        这里的参数r就是我们在前面的Step 6中创建的ServiceRecord了，它代表刚才已经启动了的CounterService。函数requestServiceBindingsLocked调用了requestServiceBindingLocked函数来处理绑定服务的操作，而函数requestServiceBindingLocked又调用了app.thread.scheduleBindService函数执行操作，前面我们已经介绍过app.thread，它是一个Binder对象的远程接口，类型是ApplicationThreadProxy。

​        Step 16. ApplicationThreadProxy.scheduleBindService

​        这个函数定义在frameworks/base/core/java/android/app/ApplicationThreadNative.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. class ApplicationThreadProxy implements IApplicationThread {  
2. ​    ......  
3. ​      
4. ​    public final void scheduleBindService(IBinder token, Intent intent, boolean rebind)  
5. ​            throws RemoteException {  
6. ​        Parcel data = Parcel.obtain();  
7. ​        data.writeInterfaceToken(IApplicationThread.descriptor);  
8. ​        data.writeStrongBinder(token);  
9. ​        intent.writeToParcel(data, 0);  
10. ​        data.writeInt(rebind ? 1 : 0);  
11. ​        mRemote.transact(SCHEDULE_BIND_SERVICE_TRANSACTION, data, null,  
12. ​            IBinder.FLAG_ONEWAY);  
13. ​        data.recycle();  
14. ​    }  
15.   
16. ​    ......  
17. }  

​         这里通过Binder驱动程序就进入到ApplicationThread的scheduleBindService函数去了。

​         Step 17. ApplicationThread.scheduleBindService
​         这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. public final class ActivityThread {  
2. ​    ......  
3.   
4. ​    public final void scheduleBindService(IBinder token, Intent intent,  
5. ​            boolean rebind) {  
6. ​        BindServiceData s = new BindServiceData();  
7. ​        s.token = token;  
8. ​        s.intent = intent;  
9. ​        s.rebind = rebind;  
10.   
11. ​        queueOrSendMessage(H.BIND_SERVICE, s);  
12. ​    }  
13.   
14. ​    ......  
15. }  

​        这里像上面的Step 11一样，调用ActivityThread.queueOrSendMessage函数来发送消息。

​        Step 18. ActivityThread.queueOrSendMessage

​        参考上面的Step 11，不过这里的消息类型是H.BIND_SERVICE。

​        Step 19. H.sendMessage

​        参考上面的Step 12，不过这里最终在H.handleMessage函数中，要处理的消息类型是H.BIND_SERVICE：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. public final class ActivityThread {  
2. ​    ......  
3.   
4. ​    private final class H extends Handler {  
5. ​        ......  
6.   
7. ​        public void handleMessage(Message msg) {  
8. ​            ......  
9. ​            switch (msg.what) {  
10. ​            ......  
11. ​            case BIND_SERVICE:  
12. ​                handleBindService((BindServiceData)msg.obj);  
13. ​                break;  
14. ​            ......  
15. ​            }  
16. ​        }  
17. ​    }  
18.   
19. ​    ......  
20. }  

​         这里调用ActivityThread.handleBindService函数来进一步处理。

​         Step 20. ActivityThread.handleBindService

​         这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. public final class ActivityThread {  
2. ​    ......  
3.   
4. ​    private final void handleBindService(BindServiceData data) {  
5. ​        Service s = mServices.get(data.token);  
6. ​        if (s != null) {  
7. ​            try {  
8. ​                data.intent.setExtrasClassLoader(s.getClassLoader());  
9. ​                try {  
10. ​                    if (!data.rebind) {  
11. ​                        IBinder binder = s.onBind(data.intent);  
12. ​                        ActivityManagerNative.getDefault().publishService(  
13. ​                            data.token, data.intent, binder);  
14. ​                    } else {  
15. ​                        ......  
16. ​                    }  
17. ​                    ......  
18. ​                } catch (RemoteException ex) {  
19. ​                }  
20. ​            } catch (Exception e) {  
21. ​                ......  
22. ​            }  
23. ​        }  
24. ​    }  
25.   
26. ​    ......  
27. }  

​        在前面的Step 13执行ActivityThread.handleCreateService函数中，已经将这个CounterService实例保存在mServices中，因此，这里首先通过data.token值将它取回来，保存在本地变量s中，接着执行了两个操作，一个操作是调用s.onBind，即CounterService.onBind获得一个Binder对象，另一个操作就是把这个Binder对象传递给ActivityManagerService。

​        我们先看CounterService.onBind操作，然后再回到ActivityThread.handleBindService函数中来。

​        Step 21. CounterService.onBind

​        这个函数定义在[Android系统中的广播（Broadcast）机制简要介绍和学习计划中](http://blog.csdn.net/luoshengyang/article/details/6730748)一文中所介绍的应用程序Broadcast的工程目录下的src/shy/luo/broadcast/CounterService.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. public class CounterService extends Service implements ICounterService {  
2. ​    ......  
3.   
4. ​    private final IBinder binder = new CounterBinder();    
5.   
6. ​    public class CounterBinder extends Binder {    
7. ​        public CounterService getService() {    
8. ​            return CounterService.this;    
9. ​        }    
10. ​    }    
11.   
12. ​    @Override    
13. ​    public IBinder onBind(Intent intent) {    
14. ​        return binder;    
15. ​    }    
16.   
17. ​    ......  
18. }  

​        这里的onBind函数返回一个是CounterBinder类型的Binder对象，它里面实现一个成员函数getService，用于返回CounterService接口。

​        至此，应用程序绑定服务过程中的第二步CounterService.onBind就完成了。

​        回到ActivityThread.handleBindService函数中，获得了这个CounterBinder对象后，就调用ActivityManagerProxy.publishService来通知MainActivity，CounterService已经连接好了。
​        Step 22. ActivityManagerProxy.publishService

​        这个函数定义在frameworks/base/core/java/android/app/ActivityManagerNative.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. class ActivityManagerProxy implements IActivityManager  
2. {  
3. ​    ......  
4.   
5. ​    public void publishService(IBinder token,  
6. ​    Intent intent, IBinder service) throws RemoteException {  
7. ​        Parcel data = Parcel.obtain();  
8. ​        Parcel reply = Parcel.obtain();  
9. ​        data.writeInterfaceToken(IActivityManager.descriptor);  
10. ​        data.writeStrongBinder(token);  
11. ​        intent.writeToParcel(data, 0);  
12. ​        data.writeStrongBinder(service);  
13. ​        mRemote.transact(PUBLISH_SERVICE_TRANSACTION, data, reply, 0);  
14. ​        reply.readException();  
15. ​        data.recycle();  
16. ​        reply.recycle();  
17. ​    }  
18.   
19. ​    ......  
20. }  

​         这里通过Binder驱动程序就进入到ActivityManagerService的publishService函数中去了。

​         Step 23. ActivityManagerService.publishService

​         这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. public final class ActivityManagerService extends ActivityManagerNative  
2. ​        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
3. ​    ......  
4.   
5. ​    public void publishService(IBinder token, Intent intent, IBinder service) {  
6. ​        ......  
7. ​        synchronized(this) {  
8. ​            ......  
9. ​            ServiceRecord r = (ServiceRecord)token;  
10. ​            ......  
11.   
12. ​            ......  
13. ​            if (r != null) {  
14. ​                Intent.FilterComparison filter  
15. ​                    = new Intent.FilterComparison(intent);  
16. ​                IntentBindRecord b = r.bindings.get(filter);  
17. ​                if (b != null && !b.received) {  
18. ​                    b.binder = service;  
19. ​                    b.requested = true;  
20. ​                    b.received = true;  
21. ​                    if (r.connections.size() > 0) {  
22. ​                        Iterator<ArrayList<ConnectionRecord>> it  
23. ​                            = r.connections.values().iterator();  
24. ​                        while (it.hasNext()) {  
25. ​                            ArrayList<ConnectionRecord> clist = it.next();  
26. ​                            for (int i=0; i<clist.size(); i++) {  
27. ​                                ConnectionRecord c = clist.get(i);  
28. ​                                ......  
29. ​                                try {  
30. ​                                    c.conn.connected(r.name, service);  
31. ​                                } catch (Exception e) {  
32. ​                                    ......  
33. ​                                }  
34. ​                            }  
35. ​                        }  
36. ​                    }  
37. ​                }  
38.   
39. ​                ......  
40. ​            }  
41. ​        }  
42. ​    }  
43.   
44. ​    ......  
45. }  

​        这里传进来的参数token是一个ServiceRecord对象，它是在上面的Step 6中创建的，代表CounterService这个Service。在Step 6中，我们曾经把一个ConnectionRecord放在ServiceRecord.connections列表中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1.    ServiceRecord s = res.record;  
2.   
3.    ......  
4.   
5.    ConnectionRecord c = new ConnectionRecord(b, activity,  
6. ​    connection, flags, clientLabel, clientIntent);  
7.   
8.    IBinder binder = connection.asBinder();  
9.    ArrayList<ConnectionRecord> clist = s.connections.get(binder);  
10.   
11.    if (clist == null) {  
12. clist = new ArrayList<ConnectionRecord>();  
13. s.connections.put(binder, clist);  
14.    }  

​        因此，这里可以从r.connections中将这个ConnectionRecord取出来：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1.    Iterator<ArrayList<ConnectionRecord>> it  
2. = r.connections.values().iterator();  
3.    while (it.hasNext()) {  
4. ArrayList<ConnectionRecord> clist = it.next();  
5. for (int i=0; i<clist.size(); i++) {  
6. ​        ConnectionRecord c = clist.get(i);  
7. ​    ......  
8. ​    try {  
9. ​        c.conn.connected(r.name, service);  
10. ​    } catch (Exception e) {  
11. ​        ......  
12. ​    }  
13. }  
14.    }  

​        每一个ConnectionRecord里面都有一个成员变量conn，它的类型是IServiceConnection，是一个Binder对象的远程接口，这个Binder对象又是什么呢？这就是我们在Step 

4中创建的LoadedApk.ServiceDispatcher.InnerConnection对象了。因此，这里执行c.conn.connected函数后就会进入到LoadedApk.ServiceDispatcher.InnerConnection.connected函数中去了。

​        Step 24. InnerConnection.connected

​        这个函数定义在frameworks/base/core/java/android/app/LoadedApk.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. final class LoadedApk {  
2. ​    ......  
3.   
4. ​    static final class ServiceDispatcher {  
5. ​        ......  
6.   
7. ​        private static class InnerConnection extends IServiceConnection.Stub {  
8. ​            ......  
9.   
10. ​            public void connected(ComponentName name, IBinder service) throws RemoteException {  
11. ​                LoadedApk.ServiceDispatcher sd = mDispatcher.get();  
12. ​                if (sd != null) {  
13. ​                    sd.connected(name, service);  
14. ​                }  
15. ​            }  
16. ​            ......  
17. ​        }  
18.   
19. ​        ......  
20. ​    }  
21.   
22. ​    ......  
23. }  

​         这里它将操作转发给ServiceDispatcher.connected函数。

​         Step 25. ServiceDispatcher.connected

​         这个函数定义在frameworks/base/core/java/android/app/LoadedApk.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. final class LoadedApk {  
2. ​    ......  
3.   
4. ​    static final class ServiceDispatcher {  
5. ​        ......  
6.   
7. ​        public void connected(ComponentName name, IBinder service) {  
8. ​            if (mActivityThread != null) {  
9. ​                mActivityThread.post(new RunConnection(name, service, 0));  
10. ​            } else {  
11. ​                ......  
12. ​            }  
13. ​        }  
14.   
15. ​        ......  
16. ​    }  
17.   
18. ​    ......  
19. }  

​        我们在前面Step 4中说到，这里的mActivityThread是一个Handler实例，它是通过ActivityThread.getHandler函数得到的，因此，调用它的post函数后，就会把一个消息放到ActivityThread的消息队列中去了。

​        Step 26. H.post

​        由于H类继承于Handler类，因此，这里实际执行的Handler.post函数，这个函数定义在frameworks/base/core/java/android/os/Handler.java文件，这里我们就不看了，有兴趣的读者可以自己研究一下，调用了这个函数之后，这个消息就真正地进入到ActivityThread的消息队列去了，与sendMessage把消息放在消息队列不一样的地方是，post方式发送的消息不是由这个Handler的handleMessage函数来处理的，而是由post的参数Runnable的run函数来处理的。这里传给post的参数是一个RunConnection类型的参数，它继承了Runnable类，因此，最终会调用RunConnection.run函数来处理这个消息。

​       Step 27. RunConnection.run

​       这个函数定义在frameworks/base/core/java/android/app/LoadedApk.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. final class LoadedApk {  
2. ​    ......  
3.   
4. ​    static final class ServiceDispatcher {  
5. ​        ......  
6.   
7. ​        private final class RunConnection implements Runnable {  
8. ​            ......  
9.   
10. ​            public void run() {  
11. ​                if (mCommand == 0) {  
12. ​                    doConnected(mName, mService);  
13. ​                } else if (mCommand == 1) {  
14. ​                    ......  
15. ​                }  
16. ​            }  
17.   
18. ​            ......  
19. ​        }  
20.   
21. ​        ......  
22. ​    }  
23.   
24. ​    ......  
25. }  

​        这里的mCommand值为0，于是就执行ServiceDispatcher.doConnected函数来进一步操作了。

​        Step 28. ServiceDispatcher.doConnected
​        这个函数定义在frameworks/base/core/java/android/app/LoadedApk.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. final class LoadedApk {  
2. ​    ......  
3.   
4. ​    static final class ServiceDispatcher {  
5. ​        ......  
6.   
7. ​        public void doConnected(ComponentName name, IBinder service) {  
8. ​            ......  
9.   
10. ​            // If there is a new service, it is now connected.  
11. ​            if (service != null) {  
12. ​                mConnection.onServiceConnected(name, service);  
13. ​            }  
14. ​        }  
15.   
16.   
17. ​        ......  
18. ​    }  
19.   
20. ​    ......  
21. }  

​        这里主要就是执行成员变量mConnection的onServiceConnected函数，这里的mConnection变量的类型的ServiceConnection，它是在前面的Step 4中设置好的，这个ServiceConnection实例是MainActivity类内部创建的，在调用bindService函数时保存在LoadedApk.ServiceDispatcher类中，用它来换取一个IServiceConnection对象，传给ActivityManagerService。

​        Step 29. ServiceConnection.onServiceConnected

​        这个函数定义在[Android系统中的广播（Broadcast）机制简要介绍和学习计划中](http://blog.csdn.net/luoshengyang/article/details/6730748)一文中所介绍的应用程序Broadcast的工程目录下的src/shy/luo/broadcast/MainActivity.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. public class MainActivity extends Activity implements OnClickListener {    
2. ​    ......  
3.   
4. ​    private ServiceConnection serviceConnection = new ServiceConnection() {    
5. ​        public void onServiceConnected(ComponentName className, IBinder service) {    
6. ​            counterService = ((CounterService.CounterBinder)service).getService();    
7.   
8. ​            Log.i(LOG_TAG, "Counter Service Connected");    
9. ​        }    
10. ​        ......  
11. ​    };    
12. ​      
13. ​    ......  
14. }  

​        这里传进来的参数service是一个Binder对象，就是前面在Step 21中从CounterService那里得到的ConterBinder对象，因此，这里可以把它强制转换为CounterBinder引用，然后调用它的getService函数。

​        至此，应用程序绑定服务过程中的第三步MainActivity.ServiceConnection.onServiceConnection就完成了。

​        Step 30. CounterBinder.getService

​        这个函数定义在[Android系统中的广播（Broadcast）机制简要介绍和学习计划中](http://blog.csdn.net/luoshengyang/article/details/6730748)一文中所介绍的应用程序Broadcast的工程目录下的src/shy/luo/broadcast/CounterService.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6745181#) [copy](http://blog.csdn.net/luoshengyang/article/details/6745181#)

1. public class CounterService extends Service implements ICounterService {    
2. ​    ......  
3.   
4. ​    public class CounterBinder extends Binder {    
5. ​        public CounterService getService() {    
6. ​            return CounterService.this;    
7. ​        }    
8. ​    }    
9.   
10. ​    ......  
11. }  

​        这里就把CounterService接口返回给MainActivity了。

​        至此，应用程序绑定服务过程中的第四步CounterService.CounterBinder.getService就完成了。

​        这样，Android应用程序绑定服务（bindService）的过程的源代码分析就完成了，总结一下这个过程：

​        1. Step 1 -  Step 14，MainActivity调用bindService函数通知ActivityManagerService，它要启动CounterService这个服务，ActivityManagerService于是在MainActivity所在的进程内部把CounterService启动起来，并且调用它的onCreate函数；

​        2. Step 15 - Step 21，ActivityManagerService把CounterService启动起来后，继续调用CounterService的onBind函数，要求CounterService返回一个Binder对象给它；

​        3. Step 22 - Step 29，ActivityManagerService从CounterService处得到这个Binder对象后，就把它传给MainActivity，即把这个Binder对象作为参数传递给MainActivity内部定义的ServiceConnection对象的onServiceConnected函数；

​        4. Step 30，MainActivity内部定义的ServiceConnection对象的onServiceConnected函数在得到这个Binder对象后，就通过它的getService成同函数获得CounterService接口。