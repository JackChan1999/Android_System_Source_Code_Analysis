摘要: 我们在编写Android程序时，常常会用到广播（Broadcast）机制。从易用性的角度来说，使用广播是非常简单的。不过，这个不是本文关心的重点，我们希望探索得再深入一点儿。我想，许多人也不想仅仅停留在使用广播的阶段，而是希望了解一些广播机制的内部机理。如果是这样的话，请容我斟一杯红茶，慢慢道来。

## 1. 概述

我们在编写Android程序时，常常会用到广播（Broadcast）机制。从易用性的角度来说，使用广播是非常简单的。不过，这个不是本文关心的重点，我们希望探索得再深入一点儿。我想，许多人也不想仅仅停留在使用广播的阶段，而是希望了解一些广播机制的内部机理。如果是这样的话，请容我斟一杯红茶，慢慢道来。

简单地说，Android广播机制的主要工作是为了实现一处发生事情，多处得到通知的效果。这种通知工作常常要牵涉跨进程通讯，所以需要由AMS（Activity Manager Service）集中管理。

![img](http://static.oschina.net/uploads/space/2014/0424/201441_jyH9_174429.png)

在Android系统中，接收广播的组件叫作receiver，而且receiver还分为动态和静态的。动态receiver是在运行期通过调用registerReceiver()注册的，而静态receiver则是在AndroidManifest.xml中声明的。动态receiver比较简单，静态的就麻烦一些了，因为在广播递送之时，静态receiver所从属的进程可能还没有启动呢，这就需要先启动新的进程，费时费力。另一方面，有些时候用户希望广播能够按照一定顺序递送，为此，Android又搞出了ordered broadcast的概念。

细节如此繁杂，非一言可以说清。我们先从receiver这一侧入手吧。

## 2. 两种receiver

Android中的receiver，分为“动态receiver”和“静态receiver”。

### 2.1 动态receiver

动态receiver必须在运行期动态注册，其实际的注册动作由ContextImpl对象完成：

```java
@Override
public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) 
{    
    return registerReceiver(receiver, filter, null, null);
}
@Override
public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
                               String broadcastPermission, Handler scheduler) 
{   
    return registerReceiverInternal(receiver, filter, broadcastPermission,
                                    scheduler, getOuterContext());
}
```

注册之时，用户会把一个自定义的receiver对象作为第一个参数传入。当然，用户的receiver都是继承于BroadcastReceiver的。使用过广播机制的程序员，对这个BroadcastReceiver应该都不陌生，这里就不多说了。我们需要关心的是，这个registerReceiverInternal()内部还包含了什么重要的细节。

registerReceiverInternal()代码的截选如下：

```java
private Intent registerReceiverInternal(BroadcastReceiver receiver,
                                        IntentFilter filter, String broadcastPermission,
                                        Handler scheduler, Context context) 
{
    IIntentReceiver rd = null;    
    if (receiver != null) 
    {        
        if (mPackageInfo != null && context != null) 
        {            
            if (scheduler == null) 
            {
                scheduler = mMainThread.getHandler();
            }            
            // 查找和context对应的“子哈希表”里的ReceiverDispatcher，如果找不到，就重新new一个
            rd = mPackageInfo.getReceiverDispatcher(receiver, context, scheduler,
                                                    mMainThread.getInstrumentation(), true);
        } 
        . . . . . .
    }    
    try 
    {        
        return ActivityManagerNative.getDefault().registerReceiver(
                mMainThread.getApplicationThread(), mBasePackageName,
                rd, filter, broadcastPermission);
    } 
    catch (RemoteException e) 
    {        
        return null;
    }
}
```

请大家注意那个rd对象（IIntentReceiver rd）。我们知道，在Android架构中，广播动作最终其实都是由AMS递送出来的。AMS利用binder机制，将语义传递给各个应用进程，应用进程再辗转调用到receiver的onReceive()，完成这次广播。而此处的rd对象正是承担“语义传递工作“的binder实体。

为了管理这个重要的binder实体，Android搞出了一个叫做ReceiveDispatcher的类。该类的定义截选如下：

【frameworks/base/core/java/android/app/LoadedApk.java】

```java
static final class ReceiverDispatcher 
{
    final static class InnerReceiver extends IIntentReceiver.Stub {
        . . . . . .
        . . . . . .
    }
    final IIntentReceiver.Stub mIIntentReceiver;   // 请注意这个域！它就是传到外面的rd。
    final BroadcastReceiver mReceiver;
    final Context mContext;
    final Handler mActivityThread;
    final Instrumentation mInstrumentation;
    final boolean mRegistered;
    final IntentReceiverLeaked mLocation;
    RuntimeException mUnregisterLocation;
    boolean mForgotten;
    . . . . . .
```

这样看来，“动态注册的BroadcastReceiver”和“ReceiverDispatcher节点”具有一一对应的关系。示意图如下：

![img](http://static.oschina.net/uploads/space/2014/0424/210455_ny00_174429.png)

一个应用里可能会注册多个动态receiver，所以这种一一对应关系最好整理成表，这个表就位于LoadedApk中。前文mPackageInfo.getReceiverDispatcher()一句中的mPackageInfo就是LoadedApk对象。

在Android的架构里，应用进程里是用LoadedApk来对应一个apk的，进程里加载了多少个apk，就会有多少LoadedApk。每个LoadedApk里会有一张“关于本apk动态注册的所有receiver”的哈希表（mReceivers）。当然，在LoadedApk初创之时，这张表只是个空表。

mReceivers表的定义如下：

```java
private final 
HashMap<Context, HashMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>> mReceivers
    = new HashMap<Context, HashMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>>();
```

该表的key项是我们比较熟悉的Context，也就是说可以是Activity、Service或Application。而value项则是另一张“子哈希表”。这是个“表中表”的形式。言下之意就是，每个Context（比如一个activity），是可以注册多个receiver的，这个很好理解。mReceivers里的“子哈希表”的key值为BroadcastReceiver，value项为ReceiverDispatcher，示意图如下：

![img](http://static.oschina.net/uploads/space/2014/0424/210601_MymB_174429.png)

图：客户进程中的mReceivers表

接下来我们继续看registerReceiverInternal()，它最终调用到

```java
ActivityManagerNative.getDefault().registerReceiver(
                    mMainThread.getApplicationThread(), mBasePackageName,
                    rd, filter, broadcastPermission);
```

registerReceiver()函数的filter参数指明了用户对哪些intent感兴趣。对同一个BroadcastReceiver对象来说，可以注册多个感兴趣的filter，就好像声明静态receiver时，也可以为一个receiver编写多个<intent-filter>一样。这些IntentFilter信息会汇总到AMS的mRegisteredReceivers表中。在AMS端，我们可以这样访问相应的汇总表：

```java
ReceiverList rl = (ReceiverList)mRegisteredReceivers.get(receiver.asBinder());
```

其中的receiver参数为IIntentReceiver型，正对应着ReceiverDispatcher中那个binder实体。也就是说，每个客户端的ReceiverDispatcher，会对应AMS端的一个ReceiverList。

ReceiverList的定义截选如下：

```java
class ReceiverList extends ArrayList<BroadcastFilter>
        implements IBinder.DeathRecipient 
{
    final ActivityManagerService owner; 
    public final IIntentReceiver receiver;    
    public final ProcessRecord app;    
    public final int pid;    
    public final int uid;
    BroadcastRecord curBroadcast = null;
    boolean linkedToDeath = false;
    String stringName;
    . . . . . .
```

ReceiverList继承于ArrayList<BroadcastFilter>，而BroadcastFilter又继承于IntentFilter，所以ReceiverList可以被理解为一个IntentFilter数组列表。

```java
class BroadcastFilter extends IntentFilter {
    final ReceiverList receiverList;
    final String packageName;
    final String requiredPermission;
    . . . . . .
```

现在，我们可以绘制一张完整一点儿的图：

![img](http://static.oschina.net/uploads/space/2014/0424/210642_v7QA_174429.png)

这张图只画了一个用户进程，在实际的系统里当然会有很多用户进程了，不过其关系是大致统一的，所以我们不再重复绘制。关于动态receiver的注册，我们就先说这么多。至于激发广播时，又会做什么动作，我们会在后文阐述，现在我们先接着说明和动态receiver相对的静态receiver。

### 2.2 静态receiver

静态receiver是指那些在AndroidManifest.xml文件中声明的receiver，它们的信息会在系统启动时，由Package Manager Service（PKMS）解析并记录下来。以后，当AMS调用PKMS的接口来查询“和intent匹配的组件”时，PKMS内部就会去查询当初记录下来的数据，并把结果返回AMS。有的同学认为静态receiver是常驻内存的，这种说法并不准确。因为常驻内存的只是静态receiver的描述性信息，并不是receiver实体本身。

在PKMS内部，会有一个针对receiver而设置的Resolver（决策器），其示意图如下：

![img](http://static.oschina.net/uploads/space/2014/0424/210720_Tdr7_174429.png)

关于PKMS的查询动作的细节，可参考PKMS的相关文档。目前我们只需知道，PKMS向外界提供了queryIntentReceivers()函数，该函数可以返回一个List<ResolveInfo>列表。

我们举个实际的例子：

```java
Intent intent = new Intent(Intent.ACTION_PRE_BOOT_COMPLETED);
List<ResolveInfo> ris = null;try {
    ris = AppGlobals.getPackageManager().queryIntentReceivers(intent, null, 0, 0);
} catch (RemoteException e) {}
```

这是AMS的systemReady()函数里的一段代码，意思是查找有多少receiver对ACTION_PRE_BOOT_COMPLETED感兴趣。

ResolveInfo的定义截选如下：

```java
public class ResolveInfo implements Parcelable 
{    
    public ActivityInfo activityInfo;    
    public ServiceInfo serviceInfo;    
    public IntentFilter filter;    
    public int priority;    
    public int preferredOrder;    
    public int match;
    . . . . . .
    . . . . . .
```

总之，当系统希望发出一个广播时，PKMS必须能够决策出，有多少静态receiver对这个广播感兴趣，而且这些receiver的信息分别又是什么。

关于receiver的注册动作，我们就先说这么多。下面我们来看看激发广播时的动作。

## 3. 激发广播

大家常见的激发广播的函数有哪些呢？从ContextImpl.java文件中，我们可以看到一系列发送广播的接口，列举如下：

- public void sendBroadcast(Intent intent)
- public void sendBroadcast(Intent intent, int userId)
- public void sendBroadcast(Intent intent, String receiverPermission)
- public void sendOrderedBroadcast(Intent intent, String receiverPermission)
- public void sendOrderedBroadcast(Intent intent, String receiverPermission, BroadcastReceiver resultReceiver, Handler scheduler, int initialCode, String initialData, Bundle initialExtras)
- public void sendStickyBroadcast(Intent intent)
- public void sendStickyOrderedBroadcast(Intent intent, BroadcastReceiver resultReceiver, Handler scheduler, int initialCode, String initialData, Bundle initialExtras)

其中sendBroadcast()是最简单的发送广播的动作。而sendOrderedBroadcast()，则是用来向系统发出有序广播(Ordered broadcast)的。这种有序广播对应的所有接收器只能按照一定的优先级顺序，依次接收intent。这些优先级一般记录在AndroidManifest.xml文件中，具体位置在<intent-filter>元素的android:priority属性中，其数值越大表示优先级越高，取值范围为-1000到1000。另外，有时候我们也可以调用IntentFilter对象的setPriority()方法来设置优先级。

对于有序广播而言，前面的接收者可以对接收到的广播intent进行处理，并将处理结果放置到广播intent中，然后传递给下一个接收者。需要注意的是，前面的接收者有权终止广播的进一步传播。也就是说，如果广播被前面的接收者终止了，那么后面的接收器就再也无法接收到广播了。

还有一个怪东西，叫做sticky广播，它又是什么呢？简单地说，sticky广播可以保证“在广播递送时尚未注册的receiver”，一旦日后注册进系统，就能够马上接到“错过”的sticky广播。有关它的细节，我们在后文再说。

在发送方，我们熟悉的调用sendBroadcast()的代码片段如下：

```java
mContext = getApplicationContext(); 
Intent intent = new Intent();  
intent.setAction("com.android.xxxxx");  
intent.setFlags(1);  
mContext.sendBroadcast(intent);
```

上面的mContext的内部其实是在调用一个ContextImpl对象的同名函数，所以我们继续查看ContextImpl.java文件。

【frameworks/base/core/java/android/app/ContextImpl.java】

```java
@Override
public void sendBroadcast(Intent intent) 
{
    String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());    
    try 
    {
        intent.setAllowFds(false);
        ActivityManagerNative.getDefault().broadcastIntent(
            mMainThread.getApplicationThread(), intent, resolvedType, null,
            Activity.RESULT_OK, null, null, null, false, false,
            Binder.getOrigCallingUser());
    } catch (RemoteException e) {
    }
}
```

简单地调用broadcastIntent()向AMS发出请求了。

### 3.1 AMS一侧的broadcastIntentLocked()

用户进程把发送广播的语义传递到AMS之后，最终会由AMS的broadcastIntentLocked()处理。其原型如下：

```java
private final int broadcastIntentLocked(ProcessRecord callerApp,
                                        String callerPackage, 
                                        Intent intent, String resolvedType,
                                        IIntentReceiver resultTo, int resultCode, 
                                        String resultData,
                                        Bundle map, String requiredPermission,
                                        boolean ordered, boolean sticky, 
                                        int callingPid, int callingUid,                   
                                        int userId)
```

broadcastIntentLocked()需要考虑以下方面的技术细节。

首先，有些广播intent只能由具有特定权限的进程发送，而有些广播intent在发送之前需要做一些其他动作。当然，如果发送方进程是系统进程、phone进程、shell进程，或者具有root权限的进程，那么必然有权发出广播。

另外，有时候用户希望发送sticky广播，以便日后注册的receiver可以收到“错过”的sticky广播。要达到这个目的，系统必须在内部维护一张sticky广播表，在具体的实现中，AMS会把广播intent加入mStickyBroadcasts映射表中。mStickyBroadcasts是一张哈希映射表，其key值为intent的action字符串，value值为“与这个action对应的intent数组列表”的引用。当我们发送sticky广播时，新的广播intent要么替换掉intent数组列表中的某项，要么作为一个新项被添加进数组列表，以备日后使用。

发送广播时，还需要考虑所发送的广播是否需要有序（ordered）递送。而且，receiver本身又分为动态注册和静态声明的，这让我们面对的情况更加复杂。从目前的代码来看，静态receiver一直是按照有序方式递送的，而动态receiver则需要根据ordered参数的值，做不同的处理。当我们需要有序递送时，AMS会把动态receivers和静态receivers合并到一张表中，这样才能依照receiver的优先级，做出正确的处理，此时动态receivers和静态receivers可能呈现一种交错顺序。

另一方面，有些广播是需要发给特定目标组件的，这个也要加以考虑。

现在我们来分析broadcastIntentLocked()函数。说得难听点儿，这个函数的实现代码颇有些裹脚布的味道，我们必须耐下性子解读这部分代码。经过一番努力，我们可以将其逻辑大致整理成以下几步：

1） 为intent添加FLAG_EXCLUDE_STOPPED_PACKAGES标记；
2） 处理和package相关的广播；
3） 处理其他一些系统广播；
4） 判断当前是否有权力发出广播；
5） 如果要发出sticky广播，那么要更新一下系统中的sticky广播列表；
6） 查询和intent匹配的静态receivers；
7） 查询和intent匹配的动态receivers；
8） 尝试向并行receivers递送广播；
9） 整合（剩下的）并行receivers，以及静态receivers，形成一个串行receivers表；
10） 尝试逐个向串行receivers递送广播。

下面我们来详细说这几个部分。

#### 3.1.1 为intent添加FLAG_EXCLUDE_STOPPED_PACKAGES标记

对应的代码为：

```java
intent = new Intent(intent);// By default broadcasts do not go to stopped apps.intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);
```

为什么intent要添加FLAG_EXCLUDE_STOPPED_PACKAGES标记呢？原因是这样的，在Android 3.1之后，PKMS加强了对“处于停止状态的”应用的管理。如果一个应用在安装后从来没有启动过，或者已经被用户强制停止了，那么这个应用就处于停止状态（stopped state）。为了达到精细调整的目的，Android增加了2个flag：FLAG_INCLUDE_STOPPED_PACKAGES和FLAG_EXCLUDE_STOPPED_PACKAGES，以此来表示intent是否要激活“处于停止状态的”应用。

```java
/**
 * If set, this intent will not match any components in packages that
 * are currently stopped.  If this is not set, then the default behavior
 * is to include such applications in the result.
 */
public static final int FLAG_EXCLUDE_STOPPED_PACKAGES = 0x00000010;
/**
 * If set, this intent will always match any components in packages that
 * are currently stopped.  This is the default behavior when
 * {@link #FLAG_EXCLUDE_STOPPED_PACKAGES} is not set.  If both of these
 * flags are set, this one wins (it allows overriding of exclude for
 * places where the framework may automatically set the exclude flag).
 */
public static final int FLAG_INCLUDE_STOPPED_PACKAGES = 0x00000020;
```

从上面的broadcastIntentLocked()函数可以看到，在默认情况下，AMS是不会把intent广播发给“处于停止状态的”应用的。据说Google这样做是为了防止一些流氓软件或病毒干坏事。当然，如果广播的发起者认为自己的确需要广播到“处于停止状态的”应用的话，它可以让intent携带FLAG_INCLUDE_STOPPED_PACKAGES标记，从这个标记的注释可以了解到，如果这两个标记同时设置的话，那么FLAG_INCLUDE_STOPPED_PACKAGES标记会“取胜”，它会覆盖掉framework自动添加的FLAG_EXCLUDE_STOPPED_PACKAGES标记。

#### 3.1.2 处理和package相关的广播

接下来需要处理一些系统级的“Package广播”，这些主要从PKMS（Package Manager Service）处发来。比如，当PKMS处理APK的添加、删除或改动时，一般会发出类似下面的广播：ACTION_PACKAGE_ADDED、ACTION_PACKAGE_REMOVED、ACTION_PACKAGE_CHANGED、ACTION_EXTERNAL_APPLICATIONS_UNAVAILABLE、ACTION_UID_REMOVED。

AMS必须确保发送“包广播”的发起方具有BROADCAST_PACKAGE_REMOVED权限，如果没有，那么AMS会抛出异常（SecurityException）。接着，AMS判断如果是某个用户id被删除了的话（Intent.ACTION_UID_REMOVED），那么必须把这件事通知给“电池状态服务”（Battery Stats Service）。另外，如果是SD卡等外部设备上的应用不可用了，这常常是因为卡被unmount了，此时PKMS会发出Intent.ACTION_EXTERNAL_APPLICATIONS_UNAVAILABLE，而AMS则需要把SD卡上的所有包都强制停止（forceStopPackageLocked()），并立即发出另一个“Package广播”——EXTERNAL_STORAGE_UNAVAILABLE。

如果只是某个外部包被删除或改动了，则要进一步判断intent里是否携带了EXTRA_DONT_KILL_APP额外数据，如果没有携带，说明需要立即强制结束package，否则，不强制结束package。看来有些应用即使在删除或改动了包后，还会在系统（内存）中保留下来并继续运行。另外，如果是删除包的话，此时要发出PACKAGE_REMOVED广播。

#### 3.1.3 处理其他一些系统广播

broadcastIntentLocked()不但要对“Package广播”进行处理，还要关心其他一些系统广播。比如ACTION_TIMEZONE_CHANGED、ACTION_CLEAR_DNS_CACHE、PROXY_CHANGE_ACTION等等，感兴趣的同学可以自行研究这些广播的意义。

#### 3.1.4 判断当前是否有权力发出广播

接着，broadcastIntentLocked()会判断当前是否有权力发出广播，代码截选如下：

```java
/*
 * Prevent non-system code (defined here to be non-persistent
 * processes) from sending protected broadcasts.
 */
if (callingUid == Process.SYSTEM_UID || callingUid == Process.PHONE_UID
        || callingUid == Process.SHELL_UID || callingUid == 0) 
{
    // Always okay.
} 
else if (callerApp == null || !callerApp.persistent) 
{
    try 
    {
        if (AppGlobals.getPackageManager().isProtectedBroadcast(intent.getAction())) 
        {
            String msg = "Permission Denial: not allowed to send broadcast "
                    + intent.getAction() + " from pid="
                    + callingPid + ", uid=" + callingUid;
            Slog.w(TAG, msg);
            throw new SecurityException(msg);
        }
    } 
    catch (RemoteException e) 
    {
        Slog.w(TAG, "Remote exception", e);
        return ActivityManager.BROADCAST_SUCCESS;
    }
}
```

如果发起方的Uid为SYSTEM_UID、PHONE_UID或SHELL_UID，或者发起方具有root权限，那么它一定有权力发送广播。

另外，还有一个“保护性广播”的概念，也要考虑进来。网上有一些人询问AndroidManifest.xml中的一级标记<protected-broadcast>是什么意思。简单地说，Google认为有一些广播是只能由系统发送的，如果某个系统级AndroidManifest.xml中写了这个标记，那么在PKMS解析该文件时，就会把“保护性广播”标记中的名字（一般是Action字符串）记录下来。在系统运作起来之后，如果某个不具有系统权限的应用试图发送系统中的“保护性广播”，那么到AMS的broadcastIntentLocked()处就会被拦住，AMS会抛出异常，提示"Permission Denial: not allowed to send broadcast"。

我们在frameworks/base/core/res/AndroidManifest.xml文件中，可以看到<protected-broadcast>标记的具体写法，截选如下：

![img](http://static.oschina.net/uploads/space/2014/0424/211117_Ute5_174429.jpg)

#### 3.1.5 必要时更新一下系统中的sticky广播列表

接着，broadcastIntentLocked()中会判断当前是否在发出sticky广播，如果是的话，必须把广播intent记录下来。

一开始会判断一下发起方是否具有发出sticky广播的能力，比如说要拥有android.Manifest.permission.BROADCAST_STICKY权限等等。判断合格后，broadcastIntentLocked()会更新AMS里的一张表——mStickyBroadcasts，其大致代码如下：

```java
ArrayList<Intent> list = mStickyBroadcasts.get(intent.getAction());
if (list == null) 
{
    list = new ArrayList<Intent>();
    mStickyBroadcasts.put(intent.getAction(), list);
}
int N = list.size();
int i;
for (i=0; i<N; i++) 
{
    if (intent.filterEquals(list.get(i))) 
    {
        // This sticky already exists, replace it.
        list.set(i, new Intent(intent));
        break;
    }
}
if (i >= N) 
{
    list.add(new Intent(intent));
}
```

mStickyBroadcasts的定义是这样的：

```java
final HashMap<String, ArrayList<Intent>> mStickyBroadcasts =
        new HashMap<String, ArrayList<Intent>>();
```

上面代码的filterEquals()函数会比较两个intent的action、data、type、class以及categories等信息，但不会比较extra数据。如果两个intent的action是一样的，但其他信息不同，那么它们在ArrayList<Intent>中会被记成两个不同的intent。而如果发现新发送的intent在ArrayList中已经有个“相等的”旧intent时，则会用新的替掉旧的。

以后，每当注册新的动态receiver时，注册动作中都会遍历一下mStickyBroadcast表，看哪些intent可以和新receiver的filter匹配，只有匹配的intent才会递送给新receiver，示意图如下：

![img](http://static.oschina.net/uploads/space/2014/0424/211259_PixD_174429.png)

图中新receiver的filter只对a1和a3这两个action感兴趣，所以遍历时就不会考虑mStickyBroadcast表中的a2表项对应的子表，而a1、a3子表所对应的若干intent中又只有一部分可以和filter匹配，比如a1的intent1以及a3的intent2，所以图中只选择了这两个intent递送给新receiver。

除了记入mStickyBoradcast表的动作以外，sticky广播和普通广播在broadcastIntentLocked()中的代码是一致的，并没有其他什么不同了。

#### 3.1.6 尝试向并行receivers递送广播

然后broadcastIntentLocked()会尝试向并行receivers递送广播。此时会调用到queue.scheduleBroadcastsLocked()。相关代码截选如下：

```java
int NR = registeredReceivers != null ? registeredReceivers.size() : 0;
if (!ordered && NR > 0) 
{
    // If we are not serializing this broadcast, then send the
    // registered receivers separately so they don't wait for the
    // components to be launched.
    final BroadcastQueue queue = broadcastQueueForIntent(intent);
    BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
            callerPackage, callingPid, callingUid, requiredPermission,
            registeredReceivers, resultTo, resultCode, resultData, map,
            ordered, sticky, false);
    if (DEBUG_BROADCAST) Slog.v(
            TAG, "Enqueueing parallel broadcast " + r);
    final boolean replaced = replacePending && queue.replaceParallelBroadcastLocked(r);
    if (!replaced) {
        queue.enqueueParallelBroadcastLocked(r);
        queue.scheduleBroadcastsLocked();    // 注意这句。。。
    }
    registeredReceivers = null;
    NR = 0;
}
```

简单地说就是，new一个BroadcastRecord节点，并插入BroadcastQueue内的并行处理队列，最后发起实际的广播调度（scheduleBroadcastsLocked()）。关于上面代码中的registeredReceivers列表，我们会在后文说明，这里先跳过。

其实不光并行处理部分需要一个BroadcastRecord节点，串行处理部分也需要BroadcastRecord节点。也就是说，要激发一次广播，AMS必须构造一个或两个BroadcastRecord节点，并将之插入合适的广播队列（mFgBroadcastQueue或mBgBroadcastQueue）。插入成功后，再执行队列的scheduleBroadcastsLocked()动作，进行实际的派发调度。示意图如下：

![img](http://static.oschina.net/uploads/space/2014/0424/211428_pjfy_174429.png)

请注意图中BroadcastRecord节点所携带的节点链。在mParallelBroadcasts表中，每个BroadcastRecord只可能携带BroadcastFilter，因为平行处理的节点只会对应动态receiver，而所有静态receiver只能是串行处理的。另一方面，在mOrderedBroadcasts表中，BroadcastRecord中则既可能携带BroadcastFilter，也可能携带ResolveInfo。这个其实很容易理解，首先，ResolveInfo对应静态receiver，放到这里自不待言，其次，如果用户在发送广播时明确指定要按ordered方式发送的话，那么即使目标方的receiver是动态注册的，它对应的BroadcastFilter也会被强制放到这里。

好，现在让我们再整合一下思路。BroadcastRecord节点内部的receivers列表，记录着和这个广播动作相关的目标receiver信息，该列表内部的子节点可能是ResolveInfo类型的，也可能是BroadcastFilter类型的。ResolveInfo是从PKMS处查到的静态receiver的描述信息，它的源头是PKMS分析的那些AndroidManifest.xml文件。而BroadcastFilter事实上来自于本文一开始阐述动态receiver时，提到的AMS端的mRegisteredReceivers哈希映射表。现在，我们再画一张示意图：

![img](http://static.oschina.net/uploads/space/2014/0424/211512_dpBK_174429.png)

因为BroadcastRecord里的BroadcastFilter，和AMS的mRegisteredReceivers表中（间接）所指的对应BroadcastFilter是同一个对象，所以我是用虚线将它们连起来的。

Ok，我们接着看scheduleBroadcastsLocked()动作。scheduleBroadcastsLocked()的代码如下：

【frameworks/base/services/java/com/android/server/am/BroadcastQueue.java】

```java
public void scheduleBroadcastsLocked() 
{
    . . . . . .
    if (mBroadcastsScheduled) 
    {
        return;
    }
    mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
    mBroadcastsScheduled = true;
}
```

发出BROADCAST_INTENT_MSG消息。

上面用到的mHandler是这样创建的：

```java
final Handler mHandler = new Handler() 
{
    public void handleMessage(Message msg) 
    {
        switch (msg.what) 
        {
            case BROADCAST_INTENT_MSG: 
            {
                if (DEBUG_BROADCAST) 
                    Slog.v(TAG, "Received BROADCAST_INTENT_MSG");
                processNextBroadcast(true);
            } 
            break;
            
            case BROADCAST_TIMEOUT_MSG: 
            {
                synchronized (mService) 
                {
                    broadcastTimeoutLocked(true);
                }
            } 
            break;
        }
    }
};
```

​也就是说，AMS端会在BroadcastQueue.java中的processNextBroadcast()具体处理广播。

#### 3.1.7 整理两个receiver列表

我们前文已经说过，有些广播是需要有序递送的。为了合理处理“有序递送”和“平行递送”，broadcastIntentLocked()函数内部搞出了两个list：

```java
List receivers = null;
List<BroadcastFilter> registeredReceivers = null;
```

其中，receivers主要用于记录“有序递送”的receiver，而registeredReceivers则用于记录与intent相匹配的动态注册的receiver。

关于这两个list的大致运作是这样的，我们先利用包管理器的queryIntentReceivers()接口，查询出和intent匹配的所有静态receivers，此时所返回的查询结果本身已经排好序了，因此，该返回值被直接赋值给了receivers变量，代码如下：

```java
receivers = AppGlobals.getPackageManager().queryIntentReceivers(intent, resolvedType, STOCK_PM_FLAGS, userId);
```

而对于动态注册的receiver信息，就不是从包管理器获取了，这些信息本来就记录在AMS之中，此时只需调用：

```java
registeredReceivers = mReceiverResolver.queryIntent(intent, resolvedType, false, userId);
```

就可以了。注意，此时返回的registeredReceivers中的子项是没有经过排序的。而关于PKMS的queryIntentReceivers()，我们可以参考PKMS的专题文档，此处不再赘述。

如果我们要“并行递送”广播， registeredReceivers中的各个receiver会在随后的queue.scheduleBroadcastsLocked()动作中被并行处理掉。如果大家折回头看看向并行receivers递送广播的代码，会发现在调用完queue.scheduleBroadcastsLocked()后，registeredReceivers会被强制赋值成null值。

如果我们要“串行递送”广播，那么必须考虑把registeredReceivers表合并到receivers表中去。我们知道，一开始receivers列表中只记录了一些静态receiver，这些receiver将会被“有序递送”。现在我们只需再遍历一下registeredReceivers列表，并将其中的每个子项插入到receivers列表的合适地方，就可以合并出一条顺序列表了。当然，如果registeredReceivers已经被设为null了，就无所谓合并了。

为什么静态声明的receiver只会“有序递送”呢？我想也许和这种receiver的复杂性有关系，因为在需要递送广播时，receiver所属的进程可能还没有启动呢，所以也许会涉及到启动进程的流程，这些都是比较复杂的流程。

当然，上面所说的是没有明确指定目标组件的情况，如果intent里含有明确的目标信息，那么就不需要调用包管理器的queryIntentReceivers()了，只需new一个ArrayList，并赋值给receivers，然后把目标组件对应的ResolveInfo信息添加进receivers数组列表即可。

#### 3.1.8 尝试逐个向串行receivers递送广播

当receivers列表整理完毕之后，现在要开始尝试逐个向串行receivers递送广播了。正如前文所说，这里要重新new一个新的BroadcastRecord节点：

```java
if ((receivers != null && receivers.size() > 0)
    || resultTo != null) 
{
    BroadcastQueue queue = broadcastQueueForIntent(intent);
    BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
            callerPackage, callingPid, callingUid, requiredPermission,
            receivers, resultTo, resultCode, resultData, map, ordered,
            sticky, false);
    . . . . . .
    boolean replaced = replacePending && queue.replaceOrderedBroadcastLocked(r); 
    if (!replaced) {
        queue.enqueueOrderedBroadcastLocked(r);
        queue.scheduleBroadcastsLocked();
    }
}
```

而scheduleBroadcastsLocked()最终会间接导致走到 BroadcastQueue.java中的processNextBroadcast()。这一点和前文所说的“向并行receivers递送广播”的动作基本一致。

下面我们来看，递送广播动作中最重要的processNextBroadcast()。

### 3.2 最重要的processNextBroadcast()

从processNextBroadcast()的代码，我们就可以看清楚前面说的“平行广播”、“有序广播”和“动态receiver”、“静态receiver”之间的关系了。

我们在前文已经说过，所有的静态receiver都是串行处理的，而动态receiver则会按照发广播时指定的方式，进行“并行”或“串行”处理。能够并行处理的广播，其对应的若干receiver一定都已经存在了，不会牵扯到启动新进程的操作，所以可以在一个while循环中，一次性全部deliver。而有序广播，则需要一个一个地处理，其滚动处理的手段是发送事件，也就是说，在一个receiver处理完毕后，会利用广播队列（BroadcastQueue）的mHandler，发送一个BROADCAST_INTENT_MSG事件，从而执行下一次的processNextBroadcast()。

processNextBroadcast()的代码逻辑大体是这样的：先尝试处理BroadcastQueue中的“平行广播”部分。这需要遍历并行列表（mParallelBroadcasts）的每一个BroadcastRecord以及其中的receivers列表。对于平行广播而言，receivers列表中的每个子节点是个BroadcastFilter。我们直接将广播递送出去即可：

```java
while (mParallelBroadcasts.size() > 0) 
{
    r = mParallelBroadcasts.remove(0);
    r.dispatchTime = SystemClock.uptimeMillis();
    r.dispatchClockTime = System.currentTimeMillis();
    final int N = r.receivers.size();
    . . . . . . 
    for (int i=0; i<N; i++) 
    {
        Object target = r.receivers.get(i);
        . . . . . .
        deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false);
    }
    . . . . . .
}
```

#### 3.2.1 用deliverToRegisteredReceiverLocked()递送到平行动态receiver

deliverToRegisteredReceiverLocked()的代码截选如下：

【frameworks/base/services/java/com/android/server/am/BroadcastQueue.java】

```java
private final void deliverToRegisteredReceiverLocked(BroadcastRecord r,
                                                     BroadcastFilter filter, 
                                                     boolean ordered) 
{
    . . . . . .
    . . . . . .
    if (!skip) 
    {
        if (ordered) 
        {
            r.receiver = filter.receiverList.receiver.asBinder();
            r.curFilter = filter;
            filter.receiverList.curBroadcast = r;
            r.state = BroadcastRecord.CALL_IN_RECEIVE;
            if (filter.receiverList.app != null) 
            {
                r.curApp = filter.receiverList.app;
                filter.receiverList.app.curReceiver = r;
                mService.updateOomAdjLocked();
            }
        }
        
            . . . . . .
            performReceiveLocked(filter.receiverList.app, 
                                 filter.receiverList.receiver,
                                 new Intent(r.intent), r.resultCode,
                                 r.resultData, r.resultExtras, 
                                 r.ordered, r.initialSticky);
            if (ordered) 
            {
                r.state = BroadcastRecord.CALL_DONE_RECEIVE;
            }
        
        . . . . . .
}
```

```java
private static void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,
        Intent intent, int resultCode, String data, Bundle extras,
        boolean ordered, boolean sticky) throws RemoteException 
{
    // Send the intent to the receiver asynchronously using one-way binder calls.
    if (app != null && app.thread != null) 
    {
        // If we have an app thread, do the call through that so it is
        // correctly ordered with other one-way calls.
        app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
                data, extras, ordered, sticky);
    } 
    else 
    {
        receiver.performReceive(intent, resultCode, data, extras, ordered, sticky);
    }
}
```

终于通过app.thread向用户进程传递语义了。注意scheduleRegisteredReceiver()的receiver参数，它对应的就是前文所说的ReceiverDispatcher的Binder实体——InnerReceiver了。

总之，当语义传递到用户进程的ApplicationThread以后，走到：

【frameworks/base/core/java/android/app/ActivityThread.java】

```java
// This function exists to make sure all receiver dispatching is
// correctly ordered, since these are one-way calls and the binder driver
// applies transaction ordering per object for such calls.
public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
        int resultCode, String dataStr, Bundle extras, boolean ordered,
        boolean sticky) throws RemoteException 
{
    receiver.performReceive(intent, resultCode, dataStr, extras, ordered, sticky);
}
```

终于走到ReceiverDispatcher的InnerReceiver了：

【frameworks/base/core/java/android/app/LoadedApk.java】

```java
static final class ReceiverDispatcher 
{
    final static class InnerReceiver extends IIntentReceiver.Stub 
    {
        . . . . . .
        . . . . . .
        public void performReceive(Intent intent, int resultCode,
                                   String data, Bundle extras, 
                                   boolean ordered, boolean sticky) 
        {
            LoadedApk.ReceiverDispatcher rd = mDispatcher.get();
            . . . . . .
            if (rd != null) {
                rd.performReceive(intent, resultCode, data, extras,
                                  ordered, sticky);
            } 
            . . . . . .
        }
    }
    . . . . . .
    public void performReceive(Intent intent, int resultCode,
                               String data, Bundle extras, 
                               boolean ordered, boolean sticky) 
    {
        . . . . . .
        Args args = new Args(intent, resultCode, data, extras, ordered, sticky);
        if (!mActivityThread.post(args)) // 请注意这一句！
        {
            if (mRegistered && ordered) 
            {
                IActivityManager mgr = ActivityManagerNative.getDefault();
                if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                        "Finishing sync broadcast to " + mReceiver);
                args.sendFinished(mgr);
            }
        }
    }
}
```

请注意mActivityThread.post(args)一句，这样，事件泵最终会回调Args参数的run()成员函数：

```java
final class Args extends BroadcastReceiver.PendingResult implements Runnable 
{
    . . . . . .
    . . . . . .
    public void run() 
    {
        final BroadcastReceiver receiver = mReceiver;
        . . . . . .
        try {
            ClassLoader cl =  mReceiver.getClass().getClassLoader();
            intent.setExtrasClassLoader(cl);
            setExtrasClassLoader(cl);
            receiver.setPendingResult(this);
            receiver.onReceive(mContext, intent);  // 回调具体receiver的onReceive()
        } catch (Exception e) {
            . . . . . .
        }
        
        if (receiver.getPendingResult() != null) {
            finish();
        }
        . . . . . .
    }
}
```

其中的那句receiver.onReceive(this)，正是回调我们具体receiver的onReceive()成员函数的地方。噢，终于看到应用程序员熟悉的onReceive()了。这部分的示意图如下：

![img](http://static.oschina.net/uploads/space/2014/0424/212025_RpoR_174429.png)

#### 3.2.2 静态receiver的递送

说完动态递送，我们再来看静态递送。对于静态receiver，情况会复杂很多，因为静态receiver所从属的进程有可能还没有运行起来呢。此时BroadcastRecord节点中记录的子列表的节点是ResolveInfo对象。

```java
ResolveInfo info = (ResolveInfo)nextReceiver;
. . . . . .
r.state = BroadcastRecord.APP_RECEIVE;
String targetProcess = info.activityInfo.processName;
```

那么我们要先找到receiver所从属的进程的进程名。

 processNextBroadcast()中启动进程的代码如下：

```java
ProcessRecord app = mService.getProcessRecordLocked(targetProcess, 
info.activityInfo.applicationInfo.uid);
. . . . . .
if (app != null && app.thread != null) 
{
    . . . . . .
    app.addPackage(info.activityInfo.packageName);
    processCurBroadcastLocked(r, app);
    return;
    . . . . . .
}
r.curApp = mService.startProcessLocked(targetProcess,
                               info.activityInfo.applicationInfo, true,
                               r.intent.getFlags() | Intent.FLAG_FROM_BACKGROUND,
                               "broadcast", r.curComponent,              
                               (r.intent.getFlags()&Intent.FLAG_RECEIVER_BOOT_UPGRADE) != 0, 
                               false)
. . . . . .
mPendingBroadcast = r;
mPendingBroadcastRecvIndex = recIdx;
```

如果目标进程已经存在了，那么app.thread肯定不为null，直接调用processCurBroadcastLocked()即可，否则就需要启动新进程了。启动的过程是异步的，可能很耗时，所以要把BroadcastRecord节点记入mPendingBroadcast。

###### 3.2.2.1 processCurBroadcastLocked()

我们先说processCurBroadcastLocked()。

```java
private final void processCurBroadcastLocked(BroadcastRecord r,
                        ProcessRecord app) throws RemoteException 
{
    . . . . . .
    r.receiver = app.thread.asBinder();
    r.curApp = app;
    app.curReceiver = r;
    . . . . . .
    . . . . . .
        app.thread.scheduleReceiver(new Intent(r.intent), r.curReceiver,
              mService.compatibilityInfoForPackageLocked(r.curReceiver.applicationInfo),
              r.resultCode, r.resultData, r.resultExtras, r.ordered);
        . . . . . .
        started = true;
    . . . . . .
}
```

其中最重要的是调用app.thread.scheduleReceiver()的那句。

这个函数对应于ActivityThread一侧的同名函数，代码如下：

```java
void scheduleReceiver(Intent intent, ActivityInfo info, 
                      CompatibilityInfo compatInfo,                      
                      int resultCode, String data, 
                      Bundle extras, boolean sync) 
                      throws RemoteException;
```

其中ActivityInfo info参数，记录着目标receiver的信息。可以看到，递送静态receiver时，是不会用到RecevierDispatcher的。

用户进程里handleMessage()这样处理RECEIVER消息：

```java
case RECEIVER:
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "broadcastReceiveComp");
    handleReceiver((ReceiverData)msg.obj);
    maybeSnapshot();
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);    
    break;
```

ActivityThread中，会运用反射机制，创建出BroadcastReceiver对象，而后回调该对象的onReceive()成员函数。

```java
private void handleReceiver(ReceiverData data) 
{
    . . . . . .
    IActivityManager mgr = ActivityManagerNative.getDefault();
    BroadcastReceiver receiver;
    try {
        java.lang.ClassLoader cl = packageInfo.getClassLoader();
        data.intent.setExtrasClassLoader(cl);
        data.setExtrasClassLoader(cl);
        receiver = (BroadcastReceiver)cl.loadClass(component).newInstance();
    } catch (Exception e) {
        . . . . . .
    }
    try {
        . . . . . .
        receiver.setPendingResult(data);
        receiver.onReceive(context.getReceiverRestrictedContext(), data.intent);
    } catch (Exception e) {
        . . . . . .
    } finally {
        sCurrentBroadcastIntent.set(null);
    }
    if (receiver.getPendingResult() != null) {
        data.finish();
    }
}
```

###### 3.2.2.2 必要时启动新进程

现在我们回过头来看，在目标进程尚未启动的情况下，是如何完成递送的。刚刚我们已经看到调用startProcessLocked()的句子了，只要不出问题，目标进程成功启动后就会调用AMS的attachApplication()。

有关attachApplication()的详情，请参考其他关于AMS的文档，此处我们只需知道它里面又会调用attachApplicationLocked()函数。

```java
private final boolean attachApplicationLocked(IApplicationThread thread, int pid)
```

attachApplicationLocked()内有这么几句：

```java
// Check if a next-broadcast receiver is in this process...
if (!badApp && isPendingBroadcastProcessLocked(pid)) {
    try {
        didSomething = sendPendingBroadcastsLocked(app);
    } catch (Exception e) {
        // If the app died trying to launch the receiver we declare it 'bad'
        badApp = true;
    }
}
```

它们的意思是，如果新启动的进程就是刚刚mPendingBroadcast所记录的进程的话，此时AMS就会执行sendPendingBroadcastsLocked(app)一句。

```java
// The app just attached; send any pending broadcasts that it should receive
boolean sendPendingBroadcastsLocked(ProcessRecord app) {
    boolean didSomething = false;
    for (BroadcastQueue queue : mBroadcastQueues) {
        didSomething |= queue.sendPendingBroadcastsLocked(app);
    }
    return didSomething;
}
```

BroadcastQueue的sendPendingBroadcastsLocked()函数如下：

```java
public boolean sendPendingBroadcastsLocked(ProcessRecord app) {
    boolean didSomething = false;
    final BroadcastRecord br = mPendingBroadcast;
    if (br != null && br.curApp.pid == app.pid) {
        try {
            mPendingBroadcast = null;
            processCurBroadcastLocked(br, app);
            didSomething = true;
        } catch (Exception e) {
            . . . . . .
        }
    }
    return didSomething;
}
```

可以看到，既然目标进程已经成功启动了，那么mPendingBroadcast就可以赋值为null了。接着，sendPendingBroadcastsLocked()会调用前文刚刚阐述的processCurBroadcastLocked()，其内再通过app.thread.scheduleReceiver()，将语义发送到用户进程，完成真正的广播递送。这部分在上一小节已有阐述，这里就不多说了。

#### 3.2.3 说说有序广播是如何循环起来的？

我们知道，平行广播的循环很简单，只是在一个while循环里对每个动态receiver执行deliverToRegisteredReceiverLocked()即可。而对有序广播来说，原则上每次processNextBroadcast()只会处理一个BroadcastRecord的一个receiver而已。当然，此时摘下的receiver既有可能是动态注册的，也有可能是静态的。

对于动态注册的receiver，目标进程处理完广播之后，会间接调用am.finishReceiver()向AMS发出反馈，关于这一步，前文在讲Args.run()时已有涉及，只是当时主要关心的是里面的receiver.onReceive()一句，现在我们要关心其中的receiver.setPendingResult()一句：

```java
final class Args extends BroadcastReceiver.PendingResult implements Runnable 
{
    . . . . . .
    . . . . . .
    public void run() 
    {
        final BroadcastReceiver receiver = mReceiver;
        . . . . . .
        try {
            ClassLoader cl =  mReceiver.getClass().getClassLoader();
            intent.setExtrasClassLoader(cl);
            setExtrasClassLoader(cl);

            // 注意，把自己设到BroadcastReceiver的mPendingResult域里了
            receiver.setPendingResult(this);    
            receiver.onReceive(mContext, intent);  // 回调具体receiver的onReceive()
        } catch (Exception e) {
            . . . . . .
        }
        
        if (receiver.getPendingResult() != null) {
            finish();
        }
        . . . . . .
    }
}
```

因为receiver通过调用setPendingResult(this)将自己设到mPendingResult域里了，那么在执行完receiver.onReceive()一句后，就会马上走入if (receiver.getPendingResult()!= null)一句，从而执行到finish()函数。

对于动态receiver而言，当初注册它时，已经在对应的ReceiverDispatcher对象里将mRegistered成员记为true了。我们摘一句前文创建ReceiverDispatcher对象的句子，如下：

```java
rd = new LoadedApk.ReceiverDispatcher(
                receiver, context, scheduler, null, true).getIIntentReceiver();
```

请注意ReceiverDispatcher构造函数的最后那个true值参数，它会给ReceiverDispatcher的mRegistered成员变量赋值。再后来，当ReceiverDispatcher构造Args对象时，就会用到这个mRegistered域。Args的构造函数如下：

```java
public Args(Intent intent, int resultCode, String resultData, Bundle resultExtras,
        boolean ordered, boolean sticky) {
    super(resultCode, resultData, resultExtras,
            mRegistered ? TYPE_REGISTERED : TYPE_UNREGISTERED,
            ordered, sticky, mIIntentReceiver.asBinder());
    mCurIntent = intent;
    mOrdered = ordered;
}
```

看到那个mRegistered了吗？此时mRegistered为true，所以用到的是TYPE_REGISTERED。这个值会记入Args的mType域（其实是其父类PendingResult的mType域）。另外，当我们要发出有序广播时，从AMS一侧传来的ordered标记最终也会传递到Args的构造函数，即上面那个ordered参数。这个参数会记入Args对象的两个成员里，即mOrdered和mOrderedHint，后者其实是Args父类PendingResult的成员。

我们绕这一小圈是为了说明什么呢？就是为了说明刚刚处理完onReceive()函数后，所调用的finish()函数会有什么行为。

finish()函数的代码截选如下：
【frameworks/base/core/java/android/content/BroadcastReceiver.java】

```java
public final void finish() {
    if (mType == TYPE_COMPONENT) {
        . . . . . .
    } else if (mOrderedHint && mType != TYPE_UNREGISTERED) {
        if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                "Finishing broadcast to " + mToken);
        final IActivityManager mgr = ActivityManagerNative.getDefault();
        sendFinished(mgr);
    }
}
```

因为此时mType的值为TYPE_REGISTERED，而且mOrderedHint为true，所以会调用到sendFinished(mgr)。

Args继承于BroadcastReceiver.PendingResult，它调用的sendFinished()就是PendingResult的sendFinished()：
【frameworks/base/core/java/android/content/BroadcastReceiver.java】

```java
public void sendFinished(IActivityManager am) 
{
    synchronized (this) {
        if (mFinished) {
            throw new IllegalStateException("Broadcast already finished");
        }
        mFinished = true;
    
        try {
            if (mResultExtras != null) {
                mResultExtras.setAllowFds(false);
            }
            if (mOrderedHint) {
                am.finishReceiver(mToken, mResultCode, mResultData, mResultExtras,
                        mAbortBroadcast);
            } else {
                // This broadcast was sent to a component; it is not ordered,
                // but we still need to tell the activity manager we are done.
                am.finishReceiver(mToken, 0, null, null, false);
            }
        } catch (RemoteException ex) {
        }
    }
}
```

代码中的am.finishReceiver()会通知AMS，表示用户侧receiver已经处理好了，或者至少告一段落了，请AMS进行下一步动作。对于有序广播而言，向AMS传递的语义中携带有mResultCode、mResultData、mResultExtras以及mAbortBroadcast。而对于普通广播而言，此时mOrderedHint为false，是不会把mAbortBroadcast语义传递给AMS的。

而对于静态注册的receiver，情况是类似的，最终也是调用am.finishReceiver()向AMS发出回馈的，只不过发起的动作是在ActivityThread的handleReceiver()动作中。还记得前文我们阐述AMS向用户进程发来语义，要求静态receiver处理广播时的情景吗？我们再列一下当时的scheduleReceiver()函数：

```java
public final void scheduleReceiver(Intent intent, ActivityInfo info,
        CompatibilityInfo compatInfo, int resultCode, String data, Bundle extras,
        boolean sync) {
    ReceiverData r = new ReceiverData(intent, resultCode, data, extras,
            sync, false, mAppThread.asBinder());
```

其中ReceiverData类的定义截选如下：

```java
static final class ReceiverData extends BroadcastReceiver.PendingResult {
public ReceiverData(Intent intent, int resultCode, 
                        String resultData, Bundle resultExtras,
                            boolean ordered, boolean sticky, IBinder token) {
        super(resultCode, resultData, resultExtras, TYPE_COMPONENT, 
               ordered, sticky, token);
        this.intent = intent;
    }
```

请大家注意两个地方，一个是ordered参数（即scheduleReceiver的sync参数），它会记入ReceiverData的mOrderedHint成员（其实是它的父类BroadcastReceiver.PendingResult的成员）中。另一个是构造ReceiverData时，为其父类构造器传入的TYPE_COMPONENT参数，该参数会记入mType域中。

scheduleReceiver()动作最终导致走到用户进程的handleReceiver()处：

【frameworks/base/core/java/android/app/ActivityThread.java】

```java
private void handleReceiver(ReceiverData data) 
{
        . . . . . .
        receiver.setPendingResult(data);
        receiver.onReceive(context.getReceiverRestrictedContext(), data.intent);
        . . . . . .
    if (receiver.getPendingResult() != null) {
        data.finish();
    }
}
```

此处调用的finish()是PendingResult类的finish()：

```java
public final void finish() 
{
    if (mType == TYPE_COMPONENT) {
        final IActivityManager mgr = ActivityManagerNative.getDefault();
        if (QueuedWork.hasPendingWork()) {
            QueuedWork.singleThreadExecutor().execute( new Runnable() {
                @Override public void run() {
                    . . . . . .
                    sendFinished(mgr);
                }
            });
        } else {
            if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                    "Finishing broadcast to component " + mToken);
            sendFinished(mgr);
        }
} else if (mOrderedHint && mType != TYPE_UNREGISTERED) {
        . . . . . .
    }
}
```

此时mType为TYPE_COMPONENT，最终会调用到sendFinished(mgr)。此处的sendFinished()内部最终也会调用到am.finishReceiver()，向AMS通告receiver已经处理好了。sendFinished()的代码前面已经列过，此处不再重列。

AMS侧在收到finishReceiver语义后，执行：

```java
public void finishReceiver(IBinder who, int resultCode, String resultData,
        Bundle resultExtras, boolean resultAbort) 
{
    . . . . . .
    try {
        boolean doNext = false;
        BroadcastRecord r = null;

        synchronized(this) {
            r = broadcastRecordForReceiverLocked(who);
            if (r != null) {
                doNext = r.queue.finishReceiverLocked(r, resultCode,
                    resultData, resultExtras, resultAbort, true);
            }
        }

        if (doNext) {
            r.queue.processNextBroadcast(false);
        }
        trimApplications();
    } finally {
        Binder.restoreCallingIdentity(origId);
    }
}
```

可以看到，如有必要，会继续调用processNextBroadcast()，从而完成有序广播的循环处理。

#### 3.2.4说说有序广播的拦截处理

对于那些处理有序广播的receivers而言，它们具有阻止其后receivers继续处理广播的权利。在BroadcastReceiver中提供有一个abortBroadcast()函数，只要我们在receiver的onReceive()中调用这个函数，那么其后面的receivers就不会再执行onReceive()了。

abortBroadcast()函数的相关代码如下：
【frameworks/base/core/java/android/content/BroadcastReceiver.java】

```java
public abstract class BroadcastReceiver {
    private PendingResult mPendingResult;
    . . . . . .
    public final void abortBroadcast() {
        checkSynchronousHint();
        mPendingResult.mAbortBroadcast = true;
    }
```

说起这段代码里的mPendingResult，请大家回忆一下前文阐述调用receiver.onReceive()的地方，是不是在调用之前会有个receiver.setPendingResult()？为了方便阅读，我们以前文所说的Args类为例：

```java
final class Args extends BroadcastReceiver.PendingResult implements Runnable 
{
    . . . . . .
    public void run() 
    {
        final BroadcastReceiver receiver = mReceiver;
        . . . . . .
        try {
            . . . . . .
            // 注意，把自己设到BroadcastReceiver的mPendingResult域里了
            receiver.setPendingResult(this);    
            receiver.onReceive(mContext, intent);  // 回调具体receiver的onReceive()
        } catch (Exception e) {
            . . . . . .
        }
        
        if (receiver.getPendingResult() != null) {
            finish();
        }
        . . . . . .
    }
}
```

Args把自己设到BroadcastReceiver的mPendingResult域里了，当我们在onReceive()中又调用abortBroadcast()时，事实上就是把Args的mAbortBroadcast设为true了。这个域会在调用finish()时传递给AMS，让AMS去完成拦截动作。finish()调用am.finishReceiver()时，最后一个参数就是mAbortBroadcast。

```java
am.finishReceiver(mToken, mResultCode, mResultData, mResultExtras,
        mAbortBroadcast);
```

前文在讲有序广播是如何循环的时，已经列过AMS一侧的finishReceiver()函数了，我们再摘录几句：

```java
        doNext = r.queue.finishReceiverLocked(r, resultCode,
            resultData, resultExtras, resultAbort, true);
. . . . . .
if (doNext) {
    r.queue.processNextBroadcast(false);
}
```
finishReceiverLocked()函数的代码截选如下：

【frameworks/base/services/java/com/android/server/am/BroadcastQueue.java】

```java
public boolean finishReceiverLocked(BroadcastRecord r, int resultCode,
        String resultData, Bundle resultExtras, boolean resultAbort,
        boolean explicit) {
    int state = r.state;
    r.state = BroadcastRecord.IDLE;
    . . . . . .
    r.curFilter = null;
    r.curApp = null;
    r.curComponent = null;
    r.curReceiver = null;
    mPendingBroadcast = null;

    r.resultCode = resultCode;
    r.resultData = resultData;
    r.resultExtras = resultExtras;
    r.resultAbort = resultAbort;

    return state == BroadcastRecord.APP_RECEIVE
            || state == BroadcastRecord.CALL_DONE_RECEIVE;
}
```

主要是清理BroadcastRecord里的成员变量，此时会把“拦截”标志记入BroadcastRecord的resultAbort成员变量。

接着程序走入r.queue.processNextBroadcast()，该函数内部会根据resultAbort来判断要不要继续将广播传递给下一个receiver处理。我们只摘录processNextBroadcast()里相关的几句：

【frameworks/base/services/java/com/android/server/am/BroadcastQueue.java】

```java
final void processNextBroadcast(boolean fromMsg) {
    . . . . . .
        do {
            if (mOrderedBroadcasts.size() == 0) {
                // No more broadcasts pending, so all done!
                . . . . . .
                return;
            }
            r = mOrderedBroadcasts.get(0);
            . . . . . .
            if (r.receivers == null || r.nextReceiver >= numReceivers
                    || r.resultAbort || forceReceive) {
                . . . . . .
                addBroadcastToHistoryLocked(r);
                mOrderedBroadcasts.remove(0);
                r = null;
                looped = true;
                continue;
            }
        } while (r == null);
    . . . . . .
}
```

可以看到，当r.resultAbort为true时，当前处理的BroadcastRecord节点就会从mOrderedBroadcasts里删除了，也就是说，和这个广播相关的后续receiver们都不会被回调了。这就是拦截动作。

#### 3.2.5 说说有序广播的timeout处理

因为AMS很难知道一次广播究竟能不能完全成功递送出去，所以它必须实现一种“时限机制”。前文在阐述broadcastIntentLocked()时，提到过new一个BroadcastRecord节点，并插入一个BroadcastQueue里的“平行列表”或者“有序列表”。不过当时我们没有太细说那个BroadcastQueue，现在我们多加一点儿说明。

实际上系统中有两个BroadcastQueue，一个叫做“前台广播队列”，另一个叫“后台广播队列”，在AMS里是这样定义的：

```java
BroadcastQueue mFgBroadcastQueue;
BroadcastQueue mBgBroadcastQueue;
```

为什么要搞出两个队列呢？我认为这是因为系统对“广播时限”的要求不同导致的。对于前台广播队列而言，它里面的每个广播必须在10秒之内把广播递送给receiver，而后台广播队列的时限比较宽，只需60秒之内递送到就可以了。具体时限值请看BroadcastQueue的mTimeoutPeriod域。注意，这个10秒或60秒限制是针对一个receiver而言的。比方说“前台广播队列”的某个BroadcastRecord节点对应了3个receiver，那么在处理这个广播节点时，只要能在30秒（3 x 10）之内搞定就可以了。事实上，AMS系统考虑了更多东西，所以它给一个BroadcastRecord的总时限是其所有receiver时限之和的2倍，在此例中就是60秒（2 x 3 x 10）。

对于平行receiver而言，时限的作用小一点儿，因为动态receiver是直接递送到目标进程的，它不考虑目标端是什么时候处理完这个广播的。

然而对于有序receiver来说，时限就比较重要了。因为receiver之间必须是串行处理的，也就是说上一个receiver在没处理完时，系统是不会让下一个receiver进行处理的。从processNextBroadcast()的代码来看，在处理有序receiver时，BroadcastRecord里的nextReceiver域会记录“下一个应该处理的receiver”的标号。只有在BroadcastRecord的所有receiver都处理完后，或者BroadcastRecord的处理时间超过了总时限的情况下，系统才会把这个BroadcastRecord节点从队列里删除。因此我们在processNextBroadcast()里看到的获取当前BroadcastRecord的句子是写死为r = mOrderedBroadcasts.get(0)的。

![img](http://static.oschina.net/uploads/space/2014/0424/212658_8nBR_174429.png)

在拿到当前BroadcastRecord之后，利用nextReceiver值拿到当前该处理的receiver信息：

```java
int recIdx = r.nextReceiver++;
. . . . . .
Object nextReceiver = r.receivers.get(recIdx);
```

当然，一开始，nextReceiver的值只会是0，表示第一个receiver有待处理，此时会给BroadcastRecord的dispatchTime域赋值。

```java
int recIdx = r.nextReceiver++;
r.receiverTime = SystemClock.uptimeMillis();if (recIdx == 0) {
    r.dispatchTime = r.receiverTime;
    r.dispatchClockTime = System.currentTimeMillis();
    . . . . . .
}
```

也就是说，dispatchTime的意义是标记实际处理BroadcastRecord的起始时间，那么这个BroadcastRecord所能允许的最大时限值就是：

dispatchTime + 2 * mTimeoutPeriod * 其receiver总数

一旦超过这个时限，而BroadcastRecord又没有处理完，那么就强制结束这个BroadcastRecord节点：

```java
if ((numReceivers > 0) &&
        (now > r.dispatchTime + (2*mTimeoutPeriod*numReceivers))) 
{
    . . . . . .
    broadcastTimeoutLocked(false); // forcibly finish this broadcast
    forceReceive = true;
    r.state = BroadcastRecord.IDLE;
}
```

此处调用的broadcastTimeoutLocked()的参数是boolean fromMsg，表示这个函数是否是在处理“时限消息”的地方调用的，因为当前是在processNextBroadcast()函数里调用broadcastTimeoutLocked()的，所以这个参数为false。从这个参数也可以看出，另一处判断“处理已经超时”的地方是在消息处理机制里，在那个地方，fromMsg参数应该设为true。

大体上说，每当processNextBroadcast()准备递送receiver时，会调用setBroadcastTimeoutLocked()设置一个延迟消息：

```java
long timeoutTime = r.receiverTime + mTimeoutPeriod;
. . . . . .
setBroadcastTimeoutLocked(timeoutTime);
```

setBroadcastTimeoutLocked()的代码如下：

```java
final void setBroadcastTimeoutLocked(long timeoutTime) 
{    if (! mPendingBroadcastTimeoutMessage) 
    {
        Message msg = mHandler.obtainMessage(BROADCAST_TIMEOUT_MSG, this);
        mHandler.sendMessageAtTime(msg, timeoutTime);
        mPendingBroadcastTimeoutMessage = true;
    }
}
```

只要我们的receiver能及时处理广播，系统就会cancel上面的延迟消息。这也就是说，但凡事件泵的handleMessage()开始处理这个消息，就说明receiver处理超时了。此时，系统会放弃处理这个receiver，并接着尝试处理下一个receiver。

```java
final Handler mHandler = new Handler() 
{
    public void handleMessage(Message msg) {
        switch (msg.what) 
        {
            . . . . . .
            case BROADCAST_TIMEOUT_MSG: 
            {
                synchronized (mService) 
                {
                    broadcastTimeoutLocked(true);
                }
            } break;
        }
    }
};
```

broadcastTimeoutLocked()的代码截选如下：

```java
final void broadcastTimeoutLocked(boolean fromMsg) 
{
    if (fromMsg) {
        mPendingBroadcastTimeoutMessage = false;
    }
    if (mOrderedBroadcasts.size() == 0) {
        return;
    }
    long now = SystemClock.uptimeMillis();
    BroadcastRecord r = mOrderedBroadcasts.get(0);
    . . . . . .
    . . . . . .
    finishReceiverLocked(r, r.resultCode, r.resultData,
            r.resultExtras, r.resultAbort, true);
    scheduleBroadcastsLocked();
    . . . . . .
}
```

可以看到，当一个receiver超时后，系统会放弃继续处理它，并再次调用scheduleBroadcastsLocked()，尝试处理下一个receiver。

## 4. 尾声

有关Android的广播机制，我们就先说这么多吧。品一杯红茶，说一段代码，管他云山雾罩，任那琐碎冗长，我自冷眼看安卓，瞧他修短随化。