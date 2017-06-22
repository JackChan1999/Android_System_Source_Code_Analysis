[Android](http://lib.csdn.net/base/android)应用程序组件Service与Activity一样，既可以在新的进程中启动，也可以在应用程序进程内部启动；前面我们已经分析了在新的进程中启动Service的过程，本文将要介绍在应用程序内部绑定Service的过程，这是一种在应用程序进程内部启动Service的方法。

《[android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

在前面一篇文章[Android进程间通信（IPC）机制Binder简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6618363)中，我们就曾经提到，在Android系统中，每一个应用程序都是由一些Activity和Service组成的，一般Service运行在独立的进程中，而Activity有可能运行在同一个进程中，也有可能运行在不同的进程中；在接下来的文章中，[Android系统在新进程中启动自定义服务过程（startService）的原理分析](http://blog.csdn.net/luoshengyang/article/details/6677029)一文介绍了在新的进程中启动Service的过程，[Android应用程序启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6689748)一文介绍了在新的进程中启动Activity的过程，而[Android应用程序内部启动Activity过程（startActivity）的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6703247)一文则介绍了在应用程序进程内部启动Activity的过程；本文接过最后一棒，继续介绍在应用程序进程内部启动Service的过程，这种过程又可以称在应用程序进程内部绑定服务（bindService）的过程，这样，读者应该就可以对Android应用程序启动Activity和Service有一个充分的认识了。

这里仍然是按照老规矩，通过具体的例子来分析Android应用程序绑定Service的过程，而所使用的例子便是前面我们在介绍Android系统广播机制的一篇文章[Android系统中的广播（Broadcast）机制简要介绍和学习计划中](http://blog.csdn.net/luoshengyang/article/details/6730748)所开发的应用程序Broadcast了。

我们先简单回顾一下这个应用程序实例绑定Service的过程。在这个应用程序的MainActivity的onCreate函数中，会调用bindService来绑定一个计数器服务CounterService，这里绑定的意思其实就是在MainActivity内部获得CounterService的接口，所以，这个过程的第一步就是要把CounterService启动起来。当CounterService的onCreate函数被调用起来了，就说明CounterService已经启动起来了，接下来系统还要调用CounterService的onBind函数，跟CounterService要一个Binder对象，这个Binder对象是在CounterService内部自定义的CounterBinder类的一个实例，它继承于Binder类，里面实现一个getService函数，用来返回外部的CounterService接口。系统得到这个Binder对象之后，就会调用MainActivity在bindService函数里面传过来的ServiceConnection实例的onServiceConnected函数，并把这个Binder对象以参数的形式传到onServiceConnected函数里面，于是，MainActivity就可以调用这个Binder对象的getService函数来获得CounterService的接口了。

这个过程比较复杂，但总体来说，思路还是比较清晰的，整个调用过程为MainActivity.bindService->CounterService.onCreate->CounterService.onBind->MainActivity.ServiceConnection.onServiceConnection->CounterService.CounterBinder.getService。下面，我们就先用一个序列图来总体描述这个服务绑定的过程，然后就具体分析每一个步骤。

![img](http://hi.csdn.net/attachment/201109/3/0_1315064144wnUc.gif)

[点击查看大图](http://hi.csdn.net/attachment/201109/3/0_1315064144wnUc.gif)

### Step 1. ContextWrapper.bindService

这个函数定义在frameworks/base/core/[Java](http://lib.csdn.net/base/java)/android/content/ContextWrapper.java文件中：

```java
public class ContextWrapper extends Context {  
    Context mBase;  
    ......  

    @Override  
    public boolean bindService(Intent service, ServiceConnection conn,  
    int flags) {  
return mBase.bindService(service, conn, flags);  
    }  

    ......  
}  
```
这里的mBase是一个ContextImpl实例变量，于是就调用ContextImpl的bindService函数来进一步处理。

### Step 2. ContextImpl.bindService

这个函数定义在frameworks/base/core/java/android/app/ContextImpl.java文件中：

```java
class ContextImpl extends Context {  
    ......  

    @Override  
    public boolean bindService(Intent service, ServiceConnection conn,  
    int flags) {  
IServiceConnection sd;  
if (mPackageInfo != null) {  
    sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(),  
        mMainThread.getHandler(), flags);  
} else {  
    ......  
}  
try {  
    int res = ActivityManagerNative.getDefault().bindService(  
        mMainThread.getApplicationThread(), getActivityToken(),  
        service, service.resolveTypeIfNeeded(getContentResolver()),  
        sd, flags);  
    ......  
    return res != 0;  
} catch (RemoteException e) {  
    return false;  
}  
    }  

    ......  

}  
```
这里的mMainThread是一个ActivityThread实例，通过它的getHandler函数可以获得一个Handler对象，有了这个Handler对象后，就可以把消息分发到ActivityThread所在的线程消息队列中去了，后面我们将会看到这个用法，现在我们暂时不关注，只要知道这里从ActivityThread处获得了一个Handler并且保存在下面要介绍的ServiceDispatcher中去就可以了。

我们先看一下ActivityThread.getHandler的实现，然后再回到这里的bindService函数来。

### Step 3. ActivityThread.getHandler

这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

```java
public final class ActivityThread {  
    ......  

    final H mH = new H();  

    ......  

    private final class H extends Handler {  
......  

public void handleMessage(Message msg) {  
    ......  
}  

......  
    }  

    ......  

    final Handler getHandler() {  
return mH;  
    }  

    ......  
}  
```
这里返回的Handler是在ActivityThread类内部从Handler类继承下来的一个H类实例变量。

回到Step 2中的ContextImpl.bindService函数中，获得了这个Handler对象后，就调用mPackageInfo.getServiceDispatcher函数来获得一个IServiceConnection接口，这里的mPackageInfo的类型是LoadedApk，我们来看看它的getServiceDispatcher函数的实现，然后再回到ContextImpl.bindService函数来。

### Step 4. LoadedApk.getServiceDispatcher

这个函数定义在frameworks/base/core/java/android/app/LoadedApk.java文件中：

```java
final class LoadedApk {  
    ......  

    public final IServiceConnection getServiceDispatcher(ServiceConnection c,  
    Context context, Handler handler, int flags) {  
synchronized (mServices) {  
    LoadedApk.ServiceDispatcher sd = null;  
    HashMap<ServiceConnection, LoadedApk.ServiceDispatcher> map = mServices.get(context);  
    if (map != null) {  
        sd = map.get(c);  
    }  
    if (sd == null) {  
        sd = new ServiceDispatcher(c, context, handler, flags);  
        if (map == null) {  
            map = new HashMap<ServiceConnection, LoadedApk.ServiceDispatcher>();  
            mServices.put(context, map);  
        }  
        map.put(c, sd);  
    } else {  
        sd.validate(context, handler);  
    }  
    return sd.getIServiceConnection();  
}  
    }  

    ......  

    static final class ServiceDispatcher {  
private final ServiceDispatcher.InnerConnection mIServiceConnection;  
private final ServiceConnection mConnection;  
private final Handler mActivityThread;  
......  

private static class InnerConnection extends IServiceConnection.Stub {  
    final WeakReference<LoadedApk.ServiceDispatcher> mDispatcher;  
    ......  

    InnerConnection(LoadedApk.ServiceDispatcher sd) {  
        mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);  
    }  

    ......  
}  

......  

ServiceDispatcher(ServiceConnection conn,  
        Context context, Handler activityThread, int flags) {  
    mIServiceConnection = new InnerConnection(this);  
    mConnection = conn;  
    mActivityThread = activityThread;  
    ......  
}  

......  

IServiceConnection getIServiceConnection() {  
    return mIServiceConnection;  
}  

......  
    }  

    ......  
}  
```
在getServiceDispatcher函数中，传进来的参数context是一个MainActivity实例，先以它为Key值在mServices中查看一下，是不是已经存在相应的ServiceDispatcher实例，如果有了，就不用创建了，直接取出来。在我们这个情景中，需要创建一个新的ServiceDispatcher。在创建新的ServiceDispatcher实例的过程中，将上面传下来ServiceConnection参数c和Hanlder参数保存在了ServiceDispatcher实例的内部，并且创建了一个InnerConnection对象，这是一个Binder对象，一会是要传递给ActivityManagerService的，ActivityManagerServic后续就是要通过这个Binder对象和ServiceConnection通信的。

函数getServiceDispatcher最后就是返回了一个InnerConnection对象给ContextImpl.bindService函数。回到ContextImpl.bindService函数中，它接着就要调用ActivityManagerService的远程接口来进一步处理了。

### Step 5. ActivityManagerService.bindService

这个函数定义在frameworks/base/core/java/android/app/ActivityManagerNative.java文件中：


```java
class ActivityManagerProxy implements IActivityManager  
{  
    ......  

    public int bindService(IApplicationThread caller, IBinder token,  
    Intent service, String resolvedType, IServiceConnection connection,  
    int flags) throws RemoteException {  
Parcel data = Parcel.obtain();  
Parcel reply = Parcel.obtain();  
data.writeInterfaceToken(IActivityManager.descriptor);  
data.writeStrongBinder(caller != null ? caller.asBinder() : null);  
data.writeStrongBinder(token);  
service.writeToParcel(data, 0);  
data.writeString(resolvedType);  
data.writeStrongBinder(connection.asBinder());  
data.writeInt(flags);  
mRemote.transact(BIND_SERVICE_TRANSACTION, data, reply, 0);  
reply.readException();  
int res = reply.readInt();  
data.recycle();  
reply.recycle();  
return res;  
    }  

    ......  
}  
```
这个函数通过Binder驱动程序就进入到ActivityManagerService的bindService函数去了。

### Step 6. ActivityManagerService.bindService

这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：

```java
public final class ActivityManagerService extends ActivityManagerNative  
implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
    ......  

    public int bindService(IApplicationThread caller, IBinder token,  
    Intent service, String resolvedType,  
    IServiceConnection connection, int flags) {  
......  

synchronized(this) {  
    ......  
    final ProcessRecord callerApp = getRecordForAppLocked(caller);  
    ......  

    ActivityRecord activity = null;  
    if (token != null) {  
        int aindex = mMainStack.indexOfTokenLocked(token);  
        ......  
        activity = (ActivityRecord)mMainStack.mHistory.get(aindex);  
    }  
      
    ......  

    ServiceLookupResult res =  
        retrieveServiceLocked(service, resolvedType,  
        Binder.getCallingPid(), Binder.getCallingUid());  
      
    ......  

    ServiceRecord s = res.record;  

    final long origId = Binder.clearCallingIdentity();  

    ......  

    AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);  
    ConnectionRecord c = new ConnectionRecord(b, activity,  
        connection, flags, clientLabel, clientIntent);  

    IBinder binder = connection.asBinder();  
    ArrayList<ConnectionRecord> clist = s.connections.get(binder);  

    if (clist == null) {  
        clist = new ArrayList<ConnectionRecord>();  
        s.connections.put(binder, clist);  
    }  
    clist.add(c);  
    b.connections.add(c);  
    if (activity != null) {  
        if (activity.connections == null) {  
            activity.connections = new HashSet<ConnectionRecord>();  
        }  
        activity.connections.add(c);  
    }  
    b.client.connections.add(c);  
    clist = mServiceConnections.get(binder);  
    if (clist == null) {  
        clist = new ArrayList<ConnectionRecord>();  
        mServiceConnections.put(binder, clist);  
    }  
  
    clist.add(c);  

    if ((flags&Context.BIND_AUTO_CREATE) != 0) {  
        ......  
        if (!bringUpServiceLocked(s, service.getFlags(), false)) {  
            return 0;  
        }  
    }  

    ......  
}  

return 1;  
    }             

    ......  
}  
```
函数首先根据传进来的参数token是MainActivity在ActivityManagerService里面的一个令牌，通过这个令牌就可以将这个代表MainActivity的ActivityRecord取回来了。

接着通过retrieveServiceLocked函数，得到一个ServiceRecord，这个ServiceReocrd描述的是一个Service对象，这里就是CounterService了，这是根据传进来的参数service的内容获得的。回忆一下在MainActivity.onCreate函数绑定服务的语句：

```java
Intent bindIntent = new Intent(MainActivity.this, CounterService.class);  
bindService(bindIntent, serviceConnection, Context.BIND_AUTO_CREATE);  
```
这里的参数service，就是上面的bindIntent了，它里面设置了CounterService类的信息（CounterService.class），因此，这里可以通过它来把CounterService的信息取出来，并且保存在ServiceRecord对象s中。

接下来，就是把传进来的参数connection封装成一个ConnectionRecord对象。注意，这里的参数connection是一个Binder对象，它的类型是LoadedApk.ServiceDispatcher.InnerConnection，是在Step 4中创建的，后续ActivityManagerService就是要通过它来告诉MainActivity，CounterService已经启动起来了，因此，这里要把这个ConnectionRecord变量c保存下来，它保在在好几个地方，都是为了后面要用时方便地取回来的，这里就不仔细去研究了，只要知道ActivityManagerService要使用它时就可以方便地把它取出来就可以了，具体后面我们再分析。

最后，传进来的参数flags的位Context.BIND_AUTO_CREATE为1（参见上面MainActivity.onCreate函数调用bindService函数时设置的参数），因此，这里会调用bringUpServiceLocked函数进一步处理。

### Step 7. ActivityManagerService.bringUpServiceLocked

这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：

```java
public final class ActivityManagerService extends ActivityManagerNative  
implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
    ......  

    private final boolean bringUpServiceLocked(ServiceRecord r,  
    int intentFlags, boolean whileRestarting) {  
......  

final String appName = r.processName;  
ProcessRecord app = getProcessRecordLocked(appName, r.appInfo.uid);  

if (app != null && app.thread != null) {  
    try {  
        realStartServiceLocked(r, app);  
        return true;  
    } catch (RemoteException e) {  
        ......  
    }  
}  

// Not running -- get it started, and enqueue this service record  
// to be executed when the app comes up.  
if (startProcessLocked(appName, r.appInfo, true, intentFlags,  
    "service", r.name, false) == null) {  
        ......  
}  

......  
    }  

    ......  
}  
```
回忆在[Android系统中的广播（Broadcast）机制简要介绍和学习计划中](http://blog.csdn.net/luoshengyang/article/details/6730748)一文中，我们没有在程序的AndroidManifest.xml配置文件中设置CounterService的process属性值，因此，它默认就为application标签的process属性值，而application标签的process属性值也没有设置，于是，它们就默认为应用程序的包名了，即这里的appName的值为"shy.luo.broadcast"。接下来根据appName和应用程序的uid值获得一个ProcessRecord记录，由于之前在启动MainActivity的时候，已经根据这个appName和uid值创建了一个ProcessReocrd对象（具体可以参考[Android应用程序启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6689748)一文），因此，这里取回来的app和app.thread均不为null，于是，就执行realStartServiceLocked函数来执行下一步操作了。

如果这里得到的ProcessRecord变量app为null，又是什么情况呢？在这种情况下，就会执行后面的startProcessLocked函数来创建一个新的进程，然后在这个新的进程中启动这个Service了，具体可以参考前面一篇文章[Android系统在新进程中启动自定义服务过程（startService）的原理分析](http://blog.csdn.net/luoshengyang/article/details/6677029)。

### Step 8. ActivityManagerService.realStartServiceLocked

这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：

```java
public final class ActivityManagerService extends ActivityManagerNative  
implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
    ......  

    private final void realStartServiceLocked(ServiceRecord r,  
    ProcessRecord app) throws RemoteException {  
......  
r.app = app;  
......  

app.services.add(r);  
......  

try {  
    ......  
    app.thread.scheduleCreateService(r, r.serviceInfo);  
    ......  
} finally {  
    ......  
}  

requestServiceBindingsLocked(r);  

......  
    }  

    ......  
}  
```
这个函数执行了两个操作，一个是操作是调用app.thread.scheduleCreateService函数来在应用程序进程内部启动CounterService，这个操作会导致CounterService的onCreate函数被调用；另一个操作是调用requestServiceBindingsLocked函数来向CounterService要一个Binder对象，这个操作会导致CounterService的onBind函数被调用。

这里，我们先沿着app.thread.scheduleCreateService这个路径分析下去，然后再回过头来分析requestServiceBindingsLocked的调用过程。这里的app.thread是一个Binder对象的远程接口，类型为ApplicationThreadProxy。每一个Android应用程序进程里面都有一个ActivtyThread对象和一个ApplicationThread对象，其中是ApplicationThread对象是ActivityThread对象的一个成员变量，是ActivityThread与ActivityManagerService之间用来执行进程间通信的，具体可以参考[Android应用程序启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6689748)一文。

### Step 9. ApplicationThreadProxy.scheduleCreateService

这个函数定义在frameworks/base/core/java/android/app/ApplicationThreadNative.java文件中：

```java
class ApplicationThreadProxy implements IApplicationThread {  
    ......  

    public final void scheduleCreateService(IBinder token, ServiceInfo info)  
    throws RemoteException {  
Parcel data = Parcel.obtain();  
data.writeInterfaceToken(IApplicationThread.descriptor);  
data.writeStrongBinder(token);  
info.writeToParcel(data, 0);  
mRemote.transact(SCHEDULE_CREATE_SERVICE_TRANSACTION, data, null,  
    IBinder.FLAG_ONEWAY);  
data.recycle();  
    }  

    ......  
}  
```
这里通过Binder驱动程序就进入到ApplicationThread的scheduleCreateService函数去了。

### Step 10. ApplicationThread.scheduleCreateService

这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

```java
public final class ActivityThread {  
    ......  

    private final class ApplicationThread extends ApplicationThreadNative {  
......  

public final void scheduleCreateService(IBinder token,  
    ServiceInfo info) {  
    CreateServiceData s = new CreateServiceData();  
    s.token = token;  
    s.info = info;  

    queueOrSendMessage(H.CREATE_SERVICE, s);  
}  

......  
    }  

    ......  
}  
```

这里它执行的操作就是调用ActivityThread的queueOrSendMessage函数把一个H.CREATE_SERVICE类型的消息放到ActivityThread的消息队列中去。

### Step 11. ActivityThread.queueOrSendMessage

这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

```java
public final class ActivityThread {  
    ......  

    // if the thread hasn't started yet, we don't have the handler, so just  
    // save the messages until we're ready.  
    private final void queueOrSendMessage(int what, Object obj) {  
queueOrSendMessage(what, obj, 0, 0);  
    }  

    private final void queueOrSendMessage(int what, Object obj, int arg1) {  
queueOrSendMessage(what, obj, arg1, 0);  
    }  

    private final void queueOrSendMessage(int what, Object obj, int arg1, int arg2) {  
synchronized (this) {  
    ......  
    Message msg = Message.obtain();  
    msg.what = what;  
    msg.obj = obj;  
    msg.arg1 = arg1;  
    msg.arg2 = arg2;  
    mH.sendMessage(msg);  
}  
    }  

    ......  
}  
```
这个消息最终是通过mH.sendMessage发送出去的，这里的mH是一个在ActivityThread内部定义的一个类，继承于Hanlder类，用于处理消息的。

### Step 12. H.sendMessage

由于H类继承于Handler类，因此，这里实际执行的Handler.sendMessage函数，这个函数定义在frameworks/base/core/java/android/os/Handler.java文件，这里我们就不看了，有兴趣的读者可以自己研究一下，调用了这个函数之后，这个消息就真正地进入到ActivityThread的消息队列去了，最终这个消息由H.handleMessage函数来处理，这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

```java
public final class ActivityThread {  
    ......  

    private final class H extends Handler {  
......  

public void handleMessage(Message msg) {  
    ......  
    switch (msg.what) {  
    ......  
    case CREATE_SERVICE:  
        handleCreateService((CreateServiceData)msg.obj);  
        break;  
    ......  
    }  
}  
    }  

    ......  
}  
```
这个消息最终由ActivityThread的handleCreateService函数来处理。

### Step 13. ActivityThread.handleCreateService
这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

```java
public final class ActivityThread {  
    ......  

    private final void handleCreateService(CreateServiceData data) {  
......  

LoadedApk packageInfo = getPackageInfoNoCheck(  
data.info.applicationInfo);  
Service service = null;  
try {  
    java.lang.ClassLoader cl = packageInfo.getClassLoader();  
    service = (Service) cl.loadClass(data.info.name).newInstance();  
} catch (Exception e) {  
    ......  
}  

try {  
    ......  

    ContextImpl context = new ContextImpl();  
    context.init(packageInfo, null, this);  

    Application app = packageInfo.makeApplication(false, mInstrumentation);  
    context.setOuterContext(service);  
    service.attach(context, this, data.info.name, data.token, app,  
        ActivityManagerNative.getDefault());  

    service.onCreate();  
    mServices.put(data.token, service);  
    ......  
} catch (Exception e) {  
    ......  
}  
    }  

    ......  
}  
```
这个函数的工作就是把CounterService类加载到内存中来，然后调用它的onCreate函数。

### Step 14. CounterService.onCreate

这个函数定义在[Android系统中的广播（Broadcast）机制简要介绍和学习计划中](http://blog.csdn.net/luoshengyang/article/details/6730748)一文中所介绍的应用程序Broadcast的工程目录下的src/shy/luo/broadcast/CounterService.java文件中：

```java
public class CounterService extends Service implements ICounterService {  
    ......  

    @Override    
    public void onCreate() {    
super.onCreate();    

Log.i(LOG_TAG, "Counter Service Created.");    
    }   

    ......  
}  
```
这样，CounterService就启动起来了。

至此，应用程序绑定服务过程中的第一步MainActivity.bindService->CounterService.onCreate就完成了。

这一步完成之后，我们还要回到Step  8中去，执行下一个操作，即调用ActivityManagerService.requestServiceBindingsLocked函数，这个调用是用来执行CounterService的onBind函数的。

### Step 15. ActivityManagerService.requestServiceBindingsLocked

这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：

```java
public final class ActivityManagerService extends ActivityManagerNative  
implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
    ......  

    private final void requestServiceBindingsLocked(ServiceRecord r) {  
Iterator<IntentBindRecord> bindings = r.bindings.values().iterator();  
while (bindings.hasNext()) {  
    IntentBindRecord i = bindings.next();  
    if (!requestServiceBindingLocked(r, i, false)) {  
        break;  
    }  
}  
    }  

    private final boolean requestServiceBindingLocked(ServiceRecord r,  
    IntentBindRecord i, boolean rebind) {  
......  
if ((!i.requested || rebind) && i.apps.size() > 0) {  
  try {  
      ......  
      r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind);  
      ......  
  } catch (RemoteException e) {  
      ......  
  }  
}  
return true;  
    }  

    ......  
}  
```
这里的参数r就是我们在前面的Step 6中创建的ServiceRecord了，它代表刚才已经启动了的CounterService。函数requestServiceBindingsLocked调用了requestServiceBindingLocked函数来处理绑定服务的操作，而函数requestServiceBindingLocked又调用了app.thread.scheduleBindService函数执行操作，前面我们已经介绍过app.thread，它是一个Binder对象的远程接口，类型是ApplicationThreadProxy。

### Step 16. ApplicationThreadProxy.scheduleBindService

这个函数定义在frameworks/base/core/java/android/app/ApplicationThreadNative.java文件中：

```java
class ApplicationThreadProxy implements IApplicationThread {  
    ......  
      
    public final void scheduleBindService(IBinder token, Intent intent, boolean rebind)  
    throws RemoteException {  
Parcel data = Parcel.obtain();  
data.writeInterfaceToken(IApplicationThread.descriptor);  
data.writeStrongBinder(token);  
intent.writeToParcel(data, 0);  
data.writeInt(rebind ? 1 : 0);  
mRemote.transact(SCHEDULE_BIND_SERVICE_TRANSACTION, data, null,  
    IBinder.FLAG_ONEWAY);  
data.recycle();  
    }  

    ......  
}  
```
这里通过Binder驱动程序就进入到ApplicationThread的scheduleBindService函数去了。

### Step 17. ApplicationThread.scheduleBindService

这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

```java
public final class ActivityThread {  
    ......  

    public final void scheduleBindService(IBinder token, Intent intent,  
    boolean rebind) {  
BindServiceData s = new BindServiceData();  
s.token = token;  
s.intent = intent;  
s.rebind = rebind;  

queueOrSendMessage(H.BIND_SERVICE, s);  
    }  

    ......  
}  
```
这里像上面的Step 11一样，调用ActivityThread.queueOrSendMessage函数来发送消息。

### Step 18. ActivityThread.queueOrSendMessage

参考上面的Step 11，不过这里的消息类型是H.BIND_SERVICE。

### Step 19. H.sendMessage

参考上面的Step 12，不过这里最终在H.handleMessage函数中，要处理的消息类型是H.BIND_SERVICE：

```java
public final class ActivityThread {  
    ......  

    private final class H extends Handler {  
......  

public void handleMessage(Message msg) {  
    ......  
    switch (msg.what) {  
    ......  
    case BIND_SERVICE:  
        handleBindService((BindServiceData)msg.obj);  
        break;  
    ......  
    }  
}  
    }  

    ......  
}  
```
这里调用ActivityThread.handleBindService函数来进一步处理。

### Step 20. ActivityThread.handleBindService

这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

```java
public final class ActivityThread {  
    ......  

    private final void handleBindService(BindServiceData data) {  
Service s = mServices.get(data.token);  
if (s != null) {  
    try {  
        data.intent.setExtrasClassLoader(s.getClassLoader());  
        try {  
            if (!data.rebind) {  
                IBinder binder = s.onBind(data.intent);  
                ActivityManagerNative.getDefault().publishService(  
                    data.token, data.intent, binder);  
            } else {  
                ......  
            }  
            ......  
        } catch (RemoteException ex) {  
        }  
    } catch (Exception e) {  
        ......  
    }  
}  
    }  

    ......  
}  
```
在前面的Step 13执行ActivityThread.handleCreateService函数中，已经将这个CounterService实例保存在mServices中，因此，这里首先通过data.token值将它取回来，保存在本地变量s中，接着执行了两个操作，一个操作是调用s.onBind，即CounterService.onBind获得一个Binder对象，另一个操作就是把这个Binder对象传递给ActivityManagerService。

我们先看CounterService.onBind操作，然后再回到ActivityThread.handleBindService函数中来。

### Step 21. CounterService.onBind

这个函数定义在[Android系统中的广播（Broadcast）机制简要介绍和学习计划中](http://blog.csdn.net/luoshengyang/article/details/6730748)一文中所介绍的应用程序Broadcast的工程目录下的src/shy/luo/broadcast/CounterService.java文件中：

```java
public class CounterService extends Service implements ICounterService {  
    ......  

    private final IBinder binder = new CounterBinder();    

    public class CounterBinder extends Binder {    
public CounterService getService() {    
    return CounterService.this;    
}    
    }    

    @Override    
    public IBinder onBind(Intent intent) {    
return binder;    
    }    

    ......  
}  
```
这里的onBind函数返回一个是CounterBinder类型的Binder对象，它里面实现一个成员函数getService，用于返回CounterService接口。

至此，应用程序绑定服务过程中的第二步CounterService.onBind就完成了。

回到ActivityThread.handleBindService函数中，获得了这个CounterBinder对象后，就调用ActivityManagerProxy.publishService来通知MainActivity，CounterService已经连接好了。
### Step 22. ActivityManagerProxy.publishService

这个函数定义在frameworks/base/core/java/android/app/ActivityManagerNative.java文件中：

```java
class ActivityManagerProxy implements IActivityManager  
{  
    ......  

    public void publishService(IBinder token,  
    Intent intent, IBinder service) throws RemoteException {  
Parcel data = Parcel.obtain();  
Parcel reply = Parcel.obtain();  
data.writeInterfaceToken(IActivityManager.descriptor);  
data.writeStrongBinder(token);  
intent.writeToParcel(data, 0);  
data.writeStrongBinder(service);  
mRemote.transact(PUBLISH_SERVICE_TRANSACTION, data, reply, 0);  
reply.readException();  
data.recycle();  
reply.recycle();  
    }  

    ......  
}  
```
这里通过Binder驱动程序就进入到ActivityManagerService的publishService函数中去了。

### Step 23. ActivityManagerService.publishService

这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：

```java
public final class ActivityManagerService extends ActivityManagerNative  
implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
    ......  

    public void publishService(IBinder token, Intent intent, IBinder service) {  
......  
synchronized(this) {  
    ......  
    ServiceRecord r = (ServiceRecord)token;  
    ......  

    ......  
    if (r != null) {  
        Intent.FilterComparison filter  
            = new Intent.FilterComparison(intent);  
        IntentBindRecord b = r.bindings.get(filter);  
        if (b != null && !b.received) {  
            b.binder = service;  
            b.requested = true;  
            b.received = true;  
            if (r.connections.size() > 0) {  
                Iterator<ArrayList<ConnectionRecord>> it  
                    = r.connections.values().iterator();  
                while (it.hasNext()) {  
                    ArrayList<ConnectionRecord> clist = it.next();  
                    for (int i=0; i<clist.size(); i++) {  
                        ConnectionRecord c = clist.get(i);  
                        ......  
                        try {  
                            c.conn.connected(r.name, service);  
                        } catch (Exception e) {  
                            ......  
                        }  
                    }  
                }  
            }  
        }  

        ......  
    }  
}  
    }  

    ......  
}  
```
这里传进来的参数token是一个ServiceRecord对象，它是在上面的Step 6中创建的，代表CounterService这个Service。在Step 6中，我们曾经把一个ConnectionRecord放在ServiceRecord.connections列表中：

```java
   ServiceRecord s = res.record;  
   
   ......  
   
   ConnectionRecord c = new ConnectionRecord(b, activity,  
       connection, flags, clientLabel, clientIntent);  
   
   IBinder binder = connection.asBinder();  
   ArrayList<ConnectionRecord> clist = s.connections.get(binder);  
   
   if (clist == null) {  
   clist = new ArrayList<ConnectionRecord>();  
   s.connections.put(binder, clist);  
   }  
```
因此，这里可以从r.connections中将这个ConnectionRecord取出来：

```java
   Iterator<ArrayList<ConnectionRecord>> it  
   = r.connections.values().iterator();  
   while (it.hasNext()) {  
   ArrayList<ConnectionRecord> clist = it.next();  
   for (int i=0; i<clist.size(); i++) {  
   ConnectionRecord c = clist.get(i);  
       ......  
       try {  
   c.conn.connected(r.name, service);  
       } catch (Exception e) {  
   ......  
       }  
   }  
   }  
```
每一个ConnectionRecord里面都有一个成员变量conn，它的类型是IServiceConnection，是一个Binder对象的远程接口，这个Binder对象又是什么呢？这就是我们在Step 

4中创建的LoadedApk.ServiceDispatcher.InnerConnection对象了。因此，这里执行c.conn.connected函数后就会进入到LoadedApk.ServiceDispatcher.InnerConnection.connected函数中去了。

### Step 24. InnerConnection.connected

这个函数定义在frameworks/base/core/java/android/app/LoadedApk.java文件中：

```java
final class LoadedApk {  
    ......  

    static final class ServiceDispatcher {  
......  

private static class InnerConnection extends IServiceConnection.Stub {  
    ......  

    public void connected(ComponentName name, IBinder service) throws RemoteException {  
        LoadedApk.ServiceDispatcher sd = mDispatcher.get();  
        if (sd != null) {  
            sd.connected(name, service);  
        }  
    }  
    ......  
}  

......  
    }  

    ......  
}  
```
这里它将操作转发给ServiceDispatcher.connected函数。

### Step 25. ServiceDispatcher.connected

这个函数定义在frameworks/base/core/java/android/app/LoadedApk.java文件中：

```java
final class LoadedApk {  
    ......  

    static final class ServiceDispatcher {  
......  

public void connected(ComponentName name, IBinder service) {  
    if (mActivityThread != null) {  
        mActivityThread.post(new RunConnection(name, service, 0));  
    } else {  
        ......  
    }  
}  

......  
    }  

    ......  
}  
```
我们在前面Step 4中说到，这里的mActivityThread是一个Handler实例，它是通过ActivityThread.getHandler函数得到的，因此，调用它的post函数后，就会把一个消息放到ActivityThread的消息队列中去了。

### Step 26. H.post

由于H类继承于Handler类，因此，这里实际执行的Handler.post函数，这个函数定义在frameworks/base/core/java/android/os/Handler.java文件，这里我们就不看了，有兴趣的读者可以自己研究一下，调用了这个函数之后，这个消息就真正地进入到ActivityThread的消息队列去了，与sendMessage把消息放在消息队列不一样的地方是，post方式发送的消息不是由这个Handler的handleMessage函数来处理的，而是由post的参数Runnable的run函数来处理的。这里传给post的参数是一个RunConnection类型的参数，它继承了Runnable类，因此，最终会调用RunConnection.run函数来处理这个消息。

### Step 27. RunConnection.run

这个函数定义在frameworks/base/core/java/android/app/LoadedApk.java文件中：

```java
final class LoadedApk {  
    ......  

    static final class ServiceDispatcher {  
......  

private final class RunConnection implements Runnable {  
    ......  

    public void run() {  
        if (mCommand == 0) {  
            doConnected(mName, mService);  
        } else if (mCommand == 1) {  
            ......  
        }  
    }  

    ......  
}  

......  
    }  

    ......  
}  
```
这里的mCommand值为0，于是就执行ServiceDispatcher.doConnected函数来进一步操作了。

### Step 28. ServiceDispatcher.doConnected
这个函数定义在frameworks/base/core/java/android/app/LoadedApk.java文件中：

```java
final class LoadedApk {  
    ......  

    static final class ServiceDispatcher {  
......  

public void doConnected(ComponentName name, IBinder service) {  
    ......  

    // If there is a new service, it is now connected.  
    if (service != null) {  
        mConnection.onServiceConnected(name, service);  
    }  
}  


......  
    }  

    ......  
}  
```
这里主要就是执行成员变量mConnection的onServiceConnected函数，这里的mConnection变量的类型的ServiceConnection，它是在前面的Step 4中设置好的，这个ServiceConnection实例是MainActivity类内部创建的，在调用bindService函数时保存在LoadedApk.ServiceDispatcher类中，用它来换取一个IServiceConnection对象，传给ActivityManagerService。

### Step 29. ServiceConnection.onServiceConnected

这个函数定义在[Android系统中的广播（Broadcast）机制简要介绍和学习计划中](http://blog.csdn.net/luoshengyang/article/details/6730748)一文中所介绍的应用程序Broadcast的工程目录下的src/shy/luo/broadcast/MainActivity.java文件中：

```java
public class MainActivity extends Activity implements OnClickListener {    
    ......  

    private ServiceConnection serviceConnection = new ServiceConnection() {    
public void onServiceConnected(ComponentName className, IBinder service) {    
    counterService = ((CounterService.CounterBinder)service).getService();    

    Log.i(LOG_TAG, "Counter Service Connected");    
}    
......  
    };    
      
    ......  
}  
```
这里传进来的参数service是一个Binder对象，就是前面在Step 21中从CounterService那里得到的ConterBinder对象，因此，这里可以把它强制转换为CounterBinder引用，然后调用它的getService函数。

至此，应用程序绑定服务过程中的第三步MainActivity.ServiceConnection.onServiceConnection就完成了。

### Step 30. CounterBinder.getService

这个函数定义在[Android系统中的广播（Broadcast）机制简要介绍和学习计划中](http://blog.csdn.net/luoshengyang/article/details/6730748)一文中所介绍的应用程序Broadcast的工程目录下的src/shy/luo/broadcast/CounterService.java文件中：

```java
public class CounterService extends Service implements ICounterService {    
    ......  

    public class CounterBinder extends Binder {    
public CounterService getService() {    
    return CounterService.this;    
}    
    }    

    ......  
}  
```
这里就把CounterService接口返回给MainActivity了。

至此，应用程序绑定服务过程中的第四步CounterService.CounterBinder.getService就完成了。

这样，Android应用程序绑定服务（bindService）的过程的源代码分析就完成了，总结一下这个过程：

Step 1 -  Step 14，MainActivity调用bindService函数通知ActivityManagerService，它要启动CounterService这个服务，ActivityManagerService于是在MainActivity所在的进程内部把CounterService启动起来，并且调用它的onCreate函数；

Step 15 - Step 21，ActivityManagerService把CounterService启动起来后，继续调用CounterService的onBind函数，要求CounterService返回一个Binder对象给它；

Step 22 - Step 29，ActivityManagerService从CounterService处得到这个Binder对象后，就把它传给MainActivity，即把这个Binder对象作为参数传递给MainActivity内部定义的ServiceConnection对象的onServiceConnected函数；

Step 30，MainActivity内部定义的ServiceConnection对象的onServiceConnected函数在得到这个Binder对象后，就通过它的getService成同函数获得CounterService接口。