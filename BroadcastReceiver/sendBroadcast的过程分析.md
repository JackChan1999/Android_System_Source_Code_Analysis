前面我们分析了[Android](http://lib.csdn.net/base/android)应用程序注册广播接收器的过程，这个过程只完成了万里长征的第一步，接下来它还要等待ActivityManagerService将广播分发过来。ActivityManagerService是如何得到广播并把它分发出去的呢？这就是本文要介绍的广播发送过程了。

《[android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

广播的发送过程比广播接收器的注册过程要复杂得多了，不过这个过程仍然是以ActivityManagerService为中心。广播的发送者将广播发送到ActivityManagerService，ActivityManagerService接收到这个广播以后，就会在自己的注册中心查看有哪些广播接收器订阅了该广播，然后把这个广播逐一发送到这些广播接收器中，但是ActivityManagerService并不等待广播接收器处理这些广播就返回了，因此，广播的发送和处理是异步的。概括来说，广播的发送路径就是从发送者到ActivityManagerService，再从ActivityManagerService到接收者，这中间的两个过程都是通过Binder进程间通信机制来完成的，因此，希望读者在继续阅读本文之前，对Android系统的Binder进程间通信机制有所了解，具体可以参考[Android进程间通信（IPC）机制Binder简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6618363)一文。

本文继续以[Android系统中的广播（Broadcast）机制简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6730748)一文中所开发的应用程序为例子，并且结合上文[Android应用程序注册广播接收器（registerReceiver）的过程分析](http://blog.csdn.net/luoshengyang/article/details/6737352)的内容，一起来分析Android应用程序发送广播的过程。

回顾一下[Android系统中的广播（Broadcast）机制简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6730748)一文中所开发的应用程序的组织[架构](http://lib.csdn.net/base/architecture)，MainActivity向ActivityManagerService注册了一个CounterService.BROADCAST_COUNTER_ACTION类型的计数器服务广播接收器，计数器服务CounterService在后台线程中启动了一个异步任务（AsyncTask），这个异步任务负责不断地增加计数，并且不断地将当前计数值通过广播的形式发送出去，以便MainActivity可以将当前计数值在应用程序的界面线程中显示出来。

计数器服务CounterService发送广播的代码如下所示：

```java
public class CounterService extends Service implements ICounterService {    
    ......   
  
    public void startCounter(int initVal) {    
AsyncTask<Integer, Integer, Integer> task = new AsyncTask<Integer, Integer, Integer>() {        
    @Override    
    protected Integer doInBackground(Integer... vals) {    
        ......    
    }    
  
    @Override     
    protected void onProgressUpdate(Integer... values) {    
        super.onProgressUpdate(values);    
  
        int counter = values[0];    
  
        Intent intent = new Intent(BROADCAST_COUNTER_ACTION);    
        intent.putExtra(COUNTER_VALUE, counter);    
  
        sendBroadcast(intent);    
    }    
  
    @Override    
    protected void onPostExecute(Integer val) {    
        ......   
    }    
  
};    
  
task.execute(0);        
    }    
  
    ......  
}  
```
在onProgressUpdate函数中，创建了一个BROADCAST_COUNTER_ACTION类型的Intent，并且在这里个Intent中附加上当前的计数器值，然后通过CounterService类的成员函数sendBroadcast将这个Intent发送出去。CounterService类继承了Service类，Service类又继承了ContextWrapper类，成员函数sendBroadcast就是从ContextWrapper类继承下来的，因此，我们就从ContextWrapper类的sendBroadcast函数开始，分析广播发送的过程。

在继承分析广播的发送过程前，我们先来看一下广播发送过程的序列图，然后按照这个序图中的步骤来一步一步分析整个过程。

![img](http://hi.csdn.net/attachment/201109/2/0_13149880404Zo8.gif)

[点击查看大图](http://hi.csdn.net/attachment/201109/2/0_13149880404Zo8.gif)

Step 1. ContextWrapper.sendBroadcast

这个函数定义在frameworks/base/core/[Java](http://lib.csdn.net/base/java)/android/content/ContextWrapper.java文件中：

```java
public class ContextWrapper extends Context {  
    Context mBase;  
  
    ......  
  
    @Override  
    public void sendBroadcast(Intent intent) {  
mBase.sendBroadcast(intent);  
    }  
  
    ......  
  
}  
```
 这里的成员变量mBase是一个ContextImpl实例，这里只简单地调用ContextImpl.sendBroadcast进一行操作。

 Step 2. ContextImpl.sendBroadcast

 这个函数定义在frameworks/base/core/java/android/app/ContextImpl.java文件中：

```java
class ContextImpl extends Context {  
    ......  
  
    @Override  
    public void sendBroadcast(Intent intent) {  
String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());  
try {  
    ActivityManagerNative.getDefault().broadcastIntent(  
        mMainThread.getApplicationThread(), intent, resolvedType, null,  
        Activity.RESULT_OK, null, null, null, false, false);  
} catch (RemoteException e) {  
}  
    }  
  
    ......  
  
}  
```
这里的resolvedType表示这个Intent的MIME类型，我们没有设置这个Intent的MIME类型，因此，这里的resolvedType为null。接下来就调用ActivityManagerService的远程接口ActivityManagerProxy把这个广播发送给ActivityManagerService了。

Step 3. ActivityManagerProxy.broadcastIntent

这个函数定义在frameworks/base/core/java/android/app/ActivityManagerNative.java文件中：

```java
class ActivityManagerProxy implements IActivityManager  
{  
    ......  
  
    public int broadcastIntent(IApplicationThread caller,  
Intent intent, String resolvedType,  IIntentReceiver resultTo,  
int resultCode, String resultData, Bundle map,  
String requiredPermission, boolean serialized,  
boolean sticky) throws RemoteException  
    {  
Parcel data = Parcel.obtain();  
Parcel reply = Parcel.obtain();  
data.writeInterfaceToken(IActivityManager.descriptor);  
data.writeStrongBinder(caller != null ? caller.asBinder() : null);  
intent.writeToParcel(data, 0);  
data.writeString(resolvedType);  
data.writeStrongBinder(resultTo != null ? resultTo.asBinder() : null);  
data.writeInt(resultCode);  
data.writeString(resultData);  
data.writeBundle(map);  
data.writeString(requiredPermission);  
data.writeInt(serialized ? 1 : 0);  
data.writeInt(sticky ? 1 : 0);  
mRemote.transact(BROADCAST_INTENT_TRANSACTION, data, reply, 0);  
reply.readException();  
int res = reply.readInt();  
reply.recycle();  
data.recycle();  
return res;  
    }  
  
    ......  
  
}  
```
 这里的实现比较简单，把要传递的参数封装好，然后通过Binder驱动程序进入到ActivityManagerService的broadcastIntent函数中。

 Step 4. ctivityManagerService.broadcastIntent

 这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：

```java
public final class ActivityManagerService extends ActivityManagerNative  
implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
    ......  
  
    public final int broadcastIntent(IApplicationThread caller,  
    Intent intent, String resolvedType, IIntentReceiver resultTo,  
    int resultCode, String resultData, Bundle map,  
    String requiredPermission, boolean serialized, boolean sticky) {  
synchronized(this) {  
    intent = verifyBroadcastLocked(intent);  
  
    final ProcessRecord callerApp = getRecordForAppLocked(caller);  
    final int callingPid = Binder.getCallingPid();  
    final int callingUid = Binder.getCallingUid();  
    final long origId = Binder.clearCallingIdentity();  
    int res = broadcastIntentLocked(callerApp,  
        callerApp != null ? callerApp.info.packageName : null,  
        intent, resolvedType, resultTo,  
        resultCode, resultData, map, requiredPermission, serialized,  
        sticky, callingPid, callingUid);  
    Binder.restoreCallingIdentity(origId);  
    return res;  
}  
    }  
  
    ......  
}  
```
 这里调用broadcastIntentLocked函数来进一步处理。

 Step 5. ActivityManagerService.broadcastIntentLocked

 这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：

```java
public final class ActivityManagerService extends ActivityManagerNative  
implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
    ......  
  
    private final int broadcastIntentLocked(ProcessRecord callerApp,  
    String callerPackage, Intent intent, String resolvedType,  
    IIntentReceiver resultTo, int resultCode, String resultData,  
    Bundle map, String requiredPermission,  
    boolean ordered, boolean sticky, int callingPid, int callingUid) {  
intent = new Intent(intent);  
  
......  
  
// Figure out who all will receive this broadcast.  
List receivers = null;  
List<BroadcastFilter> registeredReceivers = null;  
try {  
    if (intent.getComponent() != null) {  
        ......  
    } else {  
        ......  
        registeredReceivers = mReceiverResolver.queryIntent(intent, resolvedType, false);  
    }  
} catch (RemoteException ex) {  
    ......  
}  
  
final boolean replacePending =  
    (intent.getFlags()&Intent.FLAG_RECEIVER_REPLACE_PENDING) != 0;  
  
int NR = registeredReceivers != null ? registeredReceivers.size() : 0;  
if (!ordered && NR > 0) {  
    // If we are not serializing this broadcast, then send the  
    // registered receivers separately so they don't wait for the  
    // components to be launched.  
    BroadcastRecord r = new BroadcastRecord(intent, callerApp,  
        callerPackage, callingPid, callingUid, requiredPermission,  
        registeredReceivers, resultTo, resultCode, resultData, map,  
        ordered, sticky, false);  
    ......  
    boolean replaced = false;  
    if (replacePending) {  
        for (int i=mParallelBroadcasts.size()-1; i>=0; i--) {  
            if (intent.filterEquals(mParallelBroadcasts.get(i).intent)) {  
                ......  
                mParallelBroadcasts.set(i, r);  
                replaced = true;  
                break;  
            }  
        }  
    }  
  
    if (!replaced) {  
        mParallelBroadcasts.add(r);  
  
        scheduleBroadcastsLocked();  
    }  
  
    registeredReceivers = null;  
    NR = 0;  
}  
  
......  
  
    }  
  
    ......  
}  
```
 这个函数首先是根据intent找出相应的广播接收器：

```java
   // Figure out who all will receive this broadcast.  
   List receivers = null;  
   List<BroadcastFilter> registeredReceivers = null;  
   try {  
if (intent.getComponent() != null) {  
......  
} else {  
    ......  
    registeredReceivers = mReceiverResolver.queryIntent(intent, resolvedType, false);  
}  
   } catch (RemoteException ex) {  
......  
   }  
```
回忆一下前面一篇文章

Android应用程序注册广播接收器（registerReceiver）的过程分析

中的Step 6（ActivityManagerService.registerReceiver）中，我们将一个filter类型为BROADCAST_COUNTER_ACTION类型的BroadcastFilter实例保存在了ActivityManagerService的成员变量mReceiverResolver中，这个BroadcastFilter实例包含了我们所注册的广播接收器，这里就通过mReceiverResolver.queryIntent函数将这个BroadcastFilter实例取回来。由于注册一个广播类型的接收器可能有多个，所以这里把所有符合条件的的BroadcastFilter实例放在一个List中，然后返回来。在我们这个场景中，这个List就只有一个BroadcastFilter实例了，就是MainActivity注册的那个广播接收器。

       继续往下看：

```java
final boolean replacePending =  
  (intent.getFlags()&Intent.FLAG_RECEIVER_REPLACE_PENDING) != 0;  
```
       这里是查看一下这个intent的Intent.FLAG_RECEIVER_REPLACE_PENDING位有没有设置，如果设置了的话，ActivityManagerService就会在当前的系统中查看有没有相同的intent还未被处理，如果有的话，就有当前这个新的intent来替换旧的intent。这里，我们没有设置intent的Intent.FLAG_RECEIVER_REPLACE_PENDING位，因此，这里的replacePending变量为false。

       再接着往下看：

```java
  int NR = registeredReceivers != null ? registeredReceivers.size() : 0;  
  if (!ordered && NR > 0) {  
// If we are not serializing this broadcast, then send the  
// registered receivers separately so they don't wait for the  
// components to be launched.  
BroadcastRecord r = new BroadcastRecord(intent, callerApp,  
    callerPackage, callingPid, callingUid, requiredPermission,  
    registeredReceivers, resultTo, resultCode, resultData, map,  
    ordered, sticky, false);  
......  
boolean replaced = false;  
if (replacePending) {  
    for (int i=mParallelBroadcasts.size()-1; i>=0; i--) {  
if (intent.filterEquals(mParallelBroadcasts.get(i).intent)) {  
    ......  
    mParallelBroadcasts.set(i, r);  
    replaced = true;  
    break;  
}  
    }  
}  
  
if (!replaced) {  
    mParallelBroadcasts.add(r);  
  
    scheduleBroadcastsLocked();  
}  
  
registeredReceivers = null;  
NR = 0;  
   }  
```
前面我们说到，这里得到的列表registeredReceivers的大小为1，且传进来的参数ordered为false，表示要将这个广播发送给所有注册了BROADCAST_COUNTER_ACTION类型广播的接收器，因此，会执行下面的if语句。这个if语句首先创建一个广播记录块BroadcastRecord，里面记录了这个广播是由谁发出的以及要发给谁等相关信息。由于前面得到的replacePending变量为false，因此，不会执行接下来的if语句，即不会检查系统中是否有相同类型的未处理的广播。

这样，这里得到的replaced变量的值也为false，于是，就会把这个广播记录块r放在ActivityManagerService的成员变量mParcelBroadcasts中，等待进一步处理；进一步处理的操作由函数scheduleBroadcastsLocked进行。

Step 6. ActivityManagerService.scheduleBroadcastsLocked

这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：

```java
public final class ActivityManagerService extends ActivityManagerNative  
implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
    ......  
  
    private final void scheduleBroadcastsLocked() {  
......  
  
if (mBroadcastsScheduled) {  
    return;  
}  
  
mHandler.sendEmptyMessage(BROADCAST_INTENT_MSG);  
mBroadcastsScheduled = true;  
    }  
  
    ......  
}  
```
这里的mBroadcastsScheduled表示ActivityManagerService当前是不是正在处理其它广播，如果是的话，这里就先不处理直接返回了，保证所有广播串行处理。

注意这里处理广播的方式，它是通过消息循环来处理，每当ActivityManagerService接收到一个广播时，它就把这个广播放进自己的消息队列去就完事了，根本不管这个广播后续是处理的，因此，这里我们可以看出广播的发送和处理是异步的。

这里的成员变量mHandler是一个在ActivityManagerService内部定义的Handler类变量，通过它的sendEmptyMessage函数把一个类型为BROADCAST_INTENT_MSG的空消息放进ActivityManagerService的消息队列中去。这里的空消息是指这个消息除了有类型信息之外，没有任何其它额外的信息，因为前面已经把要处理的广播信息都保存在mParcelBroadcasts中了，等处理这个消息时，从mParcelBroadcasts就可以读回相关的广播信息了，因此，这里不需要把广播信息再放在消息内容中。

Step 7. Handler.sendEmptyMessage

这个自定义的Handler类实现在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中，它是ActivityManagerService的内部类，调用了它的sendEmptyMessage函数来把一个消息放到消息队列后，一会就会调用它的handleMessage函数来真正处理这个消息：

```java
public final class ActivityManagerService extends ActivityManagerNative  
implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
    ......  
  
    final Handler mHandler = new Handler() {  
public void handleMessage(Message msg) {  
    switch (msg.what) {  
    ......  
    case BROADCAST_INTENT_MSG: {  
        ......  
        processNextBroadcast(true);  
    } break;  
    ......  
    }  
}  
    }  
  
    ......  
}   
```
这里又调用了ActivityManagerService的processNextBroadcast函数来处理下一个未处理的广播。

Step 8. ActivityManagerService.processNextBroadcast

这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：

```java
public final class ActivityManagerService extends ActivityManagerNative  
implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
    ......  
  
    private final void processNextBroadcast(boolean fromMsg) {  
synchronized(this) {  
    BroadcastRecord r;  
  
    ......  
  
    if (fromMsg) {  
        mBroadcastsScheduled = false;  
    }  
  
    // First, deliver any non-serialized broadcasts right away.  
    while (mParallelBroadcasts.size() > 0) {  
        r = mParallelBroadcasts.remove(0);  
        ......  
        final int N = r.receivers.size();  
        ......  
        for (int i=0; i<N; i++) {  
            Object target = r.receivers.get(i);  
            ......  
  
            deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false);  
        }  
        addBroadcastToHistoryLocked(r);  
        ......  
    }  
  
    ......  
  
}  
    }  
  
    ......  
}  
```
这里传进来的参数fromMsg为true，于是把mBroadcastScheduled重新设为false，这样，下一个广播就能进入到消息队列中进行处理了。前面我们在Step 5中，把一个广播记录块BroadcastRecord放在了mParallelBroadcasts中，因此，这里就把它取出来进行处理了。广播记录块BroadcastRecord的receivers列表中包含了要接收这个广播的目标列表，即前面我们注册的广播接收器，用BroadcastFilter来表示，这里while循环中的for循环就是把这个广播发送给每一个订阅了该广播的接收器了，通过deliverToRegisteredReceiverLocked函数执行。

Step 9. ActivityManagerService.deliverToRegisteredReceiverLocked

这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：

```java
public final class ActivityManagerService extends ActivityManagerNative  
implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
    ......  
  
    private final void deliverToRegisteredReceiverLocked(BroadcastRecord r,  
    BroadcastFilter filter, boolean ordered) {  
boolean skip = false;  
if (filter.requiredPermission != null) {  
    ......  
}  
if (r.requiredPermission != null) {  
    ......  
}  
  
if (!skip) {  
    // If this is not being sent as an ordered broadcast, then we  
    // don't want to touch the fields that keep track of the current  
    // state of ordered broadcasts.  
    if (ordered) {  
        ......  
    }  
  
    try {  
        ......  
        performReceiveLocked(filter.receiverList.app, filter.receiverList.receiver,  
            new Intent(r.intent), r.resultCode,  
            r.resultData, r.resultExtras, r.ordered, r.initialSticky);  
        ......  
    } catch (RemoteException e) {  
        ......  
    }  
}  
  
    }  
  
    ......  
}  
```
 函数首先是检查一下广播发送和接收的权限，在我们分析的这个场景中，没有设置权限，因此，这个权限检查就跳过了，这里得到的skip为false，于是进入下面的if语句中。由于上面传时来的ordered参数为false，因此，直接就调用performReceiveLocked函数来进一步执行广播发送的操作了。

Step 10. ActivityManagerService.performReceiveLocked

这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：

```java
public final class ActivityManagerService extends ActivityManagerNative  
implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
    ......  
  
    static void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,  
    Intent intent, int resultCode, String data, Bundle extras,  
    boolean ordered, boolean sticky) throws RemoteException {  
// Send the intent to the receiver asynchronously using one-way binder calls.  
if (app != null && app.thread != null) {  
    // If we have an app thread, do the call through that so it is  
    // correctly ordered with other one-way calls.  
    app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,  
            data, extras, ordered, sticky);  
} else {  
    ......  
}  
    }  
  
    ......  
}  
```
注意，这里传进来的参数app是注册广播接收器的Activity所在的进程记录块，在我们分析的这个场景中，由于是MainActivity调用registerReceiver函数来注册这个广播接收器的，因此，参数app所代表的ProcessRecord就是MainActivity所在的进程记录块了；而参数receiver也是注册广播接收器时传给ActivityManagerService的一个Binder对象，它的类型是IIntentReceiver，具体可以参考上一篇文章

Android应用程序注册广播接收器（registerReceiver）的过程分析

中的Step 2。

       MainActivity在注册广播接收器时，已经把自己的ProcessRecord记录下来了，所以这里的参数app和app.thread均不为null，于是，ActivityManagerService就调用app.thread.scheduleRegisteredReceiver函数来把这个广播分发给MainActivity了。这里的app.thread是一个Binder远程对象，它的类型是ApplicationThreadProxy，我们在前面介绍应用程序的Activity启动过程时，已经多次看到了，具体可以参考主题[Android应用程序的Activity启动过程简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6685853)。

       Step 11. ApplicationThreadProxy.scheduleRegisteredReceiver
       这个函数定义在frameworks/base/core/java/android/app/ApplicationThreadNative.java文件中：

```java
class ApplicationThreadProxy implements IApplicationThread {  
    ......  
  
    public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,  
    int resultCode, String dataStr, Bundle extras, boolean ordered, boolean sticky)  
    throws RemoteException {  
Parcel data = Parcel.obtain();  
data.writeInterfaceToken(IApplicationThread.descriptor);  
data.writeStrongBinder(receiver.asBinder());  
intent.writeToParcel(data, 0);  
data.writeInt(resultCode);  
data.writeString(dataStr);  
data.writeBundle(extras);  
data.writeInt(ordered ? 1 : 0);  
data.writeInt(sticky ? 1 : 0);  
mRemote.transact(SCHEDULE_REGISTERED_RECEIVER_TRANSACTION, data, null,  
    IBinder.FLAG_ONEWAY);  
data.recycle();  
    }  
  
    ......  
}  
```
这里通过Binder驱动程序就进入到ApplicationThread.scheduleRegisteredReceiver函数去了。ApplicationThread是ActivityThread的一个内部类，具体可以参考Activity启动主题

Android应用程序的Activity启动过程简要介绍和学习计划

。

Step 12. ApplicaitonThread.scheduleRegisteredReceiver
这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

```java
public final class ActivityThread {  
    ......  
  
    private final class ApplicationThread extends ApplicationThreadNative {  
......  
  
// This function exists to make sure all receiver dispatching is  
// correctly ordered, since these are one-way calls and the binder driver  
// applies transaction ordering per object for such calls.  
public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,  
        int resultCode, String dataStr, Bundle extras, boolean ordered,  
        boolean sticky) throws RemoteException {  
    receiver.performReceive(intent, resultCode, dataStr, extras, ordered, sticky);  
}  
  
......  
    }  
  
    ......  
  
}  
```
这里的receiver是在前面一篇文章

Android应用程序注册广播接收器（registerReceiver）的过程分析

中的Step 4中创建的，它的具体类型是LoadedApk.ReceiverDispatcher.InnerReceiver，即定义在LoadedApk类的内部类ReceiverDispatcher里面的一个内部类InnerReceiver，这里调用它的performReceive函数。

Step 13. InnerReceiver.performReceive

这个函数定义在frameworks/base/core/java/android/app/LoadedApk.java文件中：

```java
final class LoadedApk {    
    ......   
  
    static final class ReceiverDispatcher {    
  
final static class InnerReceiver extends IIntentReceiver.Stub {   
    ......  
  
    public void performReceive(Intent intent, int resultCode,  
            String data, Bundle extras, boolean ordered, boolean sticky) {  
      
        LoadedApk.ReceiverDispatcher rd = mDispatcher.get();  
        ......  
        if (rd != null) {  
            rd.performReceive(intent, resultCode, data, extras,  
                    ordered, sticky);  
        } else {  
            ......  
        }  
    }  
}  
  
......  
    }  
  
    ......  
}  
```
 这里，它只是简单地调用ReceiverDispatcher的performReceive函数来进一步处理，这里的ReceiverDispatcher类是LoadedApk类里面的一个内部类。

 Step 14. ReceiverDispatcher.performReceive

 这个函数定义在frameworks/base/core/java/android/app/LoadedApk.java文件中：

```java
final class LoadedApk {    
    ......   
  
    static final class ReceiverDispatcher {    
......  
  
public void performReceive(Intent intent, int resultCode,  
        String data, Bundle extras, boolean ordered, boolean sticky) {  
    ......  
  
    Args args = new Args();  
    args.mCurIntent = intent;  
    args.mCurCode = resultCode;  
    args.mCurData = data;  
    args.mCurMap = extras;  
    args.mCurOrdered = ordered;  
    args.mCurSticky = sticky;  
    if (!mActivityThread.post(args)) {  
        ......  
    }   
}  
  
......  
    }  
  
    ......  
}  
```
这里mActivityThread成员变量的类型为Handler，它是前面MainActivity注册广播接收器时，从ActivityThread取得的，具体可以参考前面一篇文章

Android应用程序注册广播接收器（registerReceiver）的过程分析

中的Step 3。这里ReceiverDispatcher借助这个Handler，把这个广播以消息的形式放到MainActivity所在的这个ActivityThread的消息队列中去，因此，ReceiverDispatcher不等这个广播被MainActivity处理就返回了，这里也体现了广播的发送和处理是异步进行的。

注意这里处理消息的方式是通过Handler.post函数进行的，post函数的参数是Runnable类型的，这个消息最终会调用这个这个参数的run成员函数来处理。这里的Args类是LoadedApk类的内部类ReceiverDispatcher的一个内部类，它继承于Runnable类，因此，可以作为mActivityThread.post的参数传进去，代表这个广播的intent也保存在这个Args实例中。

Step 15. Hanlder.post

这个函数定义在frameworks/base/core/java/android/os/Handler.java文件中，这个函数我们就不看了，有兴趣的读者可以自己研究一下，它的作用就是把消息放在消息队列中，然后就返回了，这个消息最终会在传进来的Runnable类型的参数的run成员函数中进行处理。

Step 16. Args.run

这个函数定义在frameworks/base/core/java/android/app/LoadedApk.java文件中：

```java
final class LoadedApk {    
    ......   
  
    static final class ReceiverDispatcher {  
......  
  
final class Args implements Runnable {  
    ......  
  
    public void run() {  
        BroadcastReceiver receiver = mReceiver;  
  
        ......  
  
        Intent intent = mCurIntent;  
          
        ......  
  
        try {  
            ClassLoader cl =  mReceiver.getClass().getClassLoader();  
            intent.setExtrasClassLoader(cl);  
            if (mCurMap != null) {  
                mCurMap.setClassLoader(cl);  
            }  
            receiver.setOrderedHint(true);  
            receiver.setResult(mCurCode, mCurData, mCurMap);  
            receiver.clearAbortBroadcast();  
            receiver.setOrderedHint(mCurOrdered);  
            receiver.setInitialStickyHint(mCurSticky);  
            receiver.onReceive(mContext, intent);  
        } catch (Exception e) {  
            ......  
        }  
  
        ......  
    }  
  
    ......  
}  
  
......  
    }  
  
    ......  
}  
```
这里的mReceiver是ReceiverDispatcher类的成员变量，它的类型是BroadcastReceiver，这里它就是MainActivity注册广播接收器时创建的BroadcastReceiver实例了，具体可以参考前面一篇文章

Android应用程序注册广播接收器（registerReceiver）的过程分析

中的Step 2。

有了这个ReceiverDispatcher实例之后，就可以调用它的onReceive函数把这个广播分发给它处理了。

Step 17. BroadcastReceiver.onReceive

这个函数定义[Android系统中的广播（Broadcast）机制简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6730748)一文中所介绍的Android应用程序Broadcast的工程目录下的src/shy/luo/broadcast/MainActivity.java文件中：

```java
public class MainActivity extends Activity implements OnClickListener {      
    ......    
  
    private BroadcastReceiver counterActionReceiver = new BroadcastReceiver(){    
public void onReceive(Context context, Intent intent) {    
    int counter = intent.getIntExtra(CounterService.COUNTER_VALUE, 0);    
    String text = String.valueOf(counter);    
    counterText.setText(text);    
  
    Log.i(LOG_TAG, "Receive counter event");    
}      
    }  
  
    ......    
  
}  
```
这样，MainActivity里面的定义的BroadcastReceiver实例counterActionReceiver就收到这个广播并进行处理了。

至此，Android应用程序发送广播的过程就分析完成了，结合前面这篇分析广播接收器注册过程的文章

Android应用程序注册广播接收器（registerReceiver）的过程分析

，就会对Android系统的广播机制且个更深刻的认识和理解了。

最后，我们总结一下这个Android应用程序发送广播的过程：

Step 1 - Step 7，计数器服务CounterService通过sendBroadcast把一个广播通过Binder进程间通信机制发送给ActivityManagerService，ActivityManagerService根据这个广播的Action类型找到相应的广播接收器，然后把这个广播放进自己的消息队列中去，就完成第一阶段对这个广播的异步分发了；

Step 8 - Step 15，ActivityManagerService在消息循环中处理这个广播，并通过Binder进程间通信机制把这个广播分发给注册的广播接收分发器ReceiverDispatcher，ReceiverDispatcher把这个广播放进MainActivity所在的线程的消息队列中去，就完成第二阶段对这个广播的异步分发了；

Step 16 - Step 17， ReceiverDispatcher的内部类Args在MainActivity所在的线程消息循环中处理这个广播，最终是将这个广播分发给所注册的BroadcastReceiver实例的onReceive函数进行处理。

这样，Android系统广播机制就学习完成了，希望对读者有所帮助。重新学习Android系统的广播机制，请回到[Android系统中的广播（Broadcast）机制简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6730748)一文中。