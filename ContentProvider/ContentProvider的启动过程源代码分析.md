通过前面的学习，我们知道在[Android](http://lib.csdn.net/base/android)系统中，Content Provider可以为不同的应用程序访问相同的数据提供统一的入口。Content Provider一般是运行在独立的进程中的，每一个Content Provider在系统中只有一个实例存在，其它应用程序首先要找到这个实例，然后才能访问它的数据。那么，系统中的Content Provider实例是由谁来负责启动的呢？本文将回答这个问题。

《[android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

Content Provider和应用程序组件Activity、Service一样，需要在AndroidManifest.xml文件中配置之后才能使用。系统在安装包含Content Provider的应用程序的时候，会把这些Content Provider的描述信息保存起来，其中最重要的就是Content Provider的Authority信息，Android应用程序的安装过程具体可以参考[Android应用程序安装过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6766010)一文。注意，安装应用程序的时候，并不会把相应的Content Provider加载到内存中来，系统采取的是懒加载的机制，等到第一次要使用这个Content Provider的时候，系统才会把它加载到内存中来，下次再要使用这个Content Provider的时候，就可以直接返回了。

本文以前面一篇文章[Android应用程序组件Content Provider应用实例](http://blog.csdn.net/luoshengyang/article/details/6950440)中的例子来详细分析Content Provider的启动过程。在[Android应用程序组件Content Provider应用实例](http://blog.csdn.net/luoshengyang/article/details/6950440)这篇文章介绍的应用程序Article中，第一次使用ArticlesProvider这个Content Provider的地方是ArticlesAdapter类的getArticleCount函数，因为MainActivity要在ListView中显示文章信息列表时， 首先要知道ArticlesProvider中的文章信息的数量。从ArticlesAdapter类的getArticleCount函数调用开始，一直到ArticlesProvider类的onCreate函数被调用，就是ArticlesProvider的完整启动过程，下面我们就先看看这个过程的序列图，然后再详细分析每一个步骤：

![img](http://hi.csdn.net/attachment/201111/13/0_13211766049TP1.gif)

Step 1. ArticlesAdapter.getArticleCount

这个函数定义在前面一篇文章[Android应用程序组件Content Provider应用实例](http://blog.csdn.net/luoshengyang/article/details/6950440)介绍的应用程序Artilce源代码工程目录下，在文件为packages/experimental/Article/src/shy/luo/article/ArticlesAdapter.[Java](http://lib.csdn.net/base/java)中：

```java
public class ArticlesAdapter {  
    ......  
  
    private ContentResolver resolver = null;  
  
    public ArticlesAdapter(Context context) {  
resolver = context.getContentResolver();  
    }  
  
    ......  
  
    public int getArticleCount() {  
int count = 0;  
  
try {  
    IContentProvider provider = resolver.acquireProvider(Articles.CONTENT_URI);  
    Bundle bundle = provider.call(Articles.METHOD_GET_ITEM_COUNT, null, null);  
    count = bundle.getInt(Articles.KEY_ITEM_COUNT, 0);  
} catch(RemoteException e) {  
    e.printStackTrace();  
}  
  
return count;  
    }  
  
    ......  
}  
```
 这个函数通过应用程序上下文的ContentResolver接口resolver的acquireProvider函数来获得与Articles.CONTENT_URI对应的Content Provider对象的IContentProvider接口。常量Articles.CONTENT_URI是在应用程序ArticlesProvider中定义的，它的值为“content://shy.luo.providers.articles/item”，对应的Content Provider就是ArticlesProvider了。

 Step 2. ContentResolver.acqireProvider

 这个函数定义在frameworks/base/core/java/android/content/ContentResolver.java文件中：

```java
public abstract class ContentResolver {  
    ......  
  
    public final IContentProvider acquireProvider(Uri uri) {  
if (!SCHEME_CONTENT.equals(uri.getScheme())) {  
    return null;  
}  
String auth = uri.getAuthority();  
if (auth != null) {  
    return acquireProvider(mContext, uri.getAuthority());  
}  
return null;  
    }  
  
    ......  
}  
```
函数首先验证参数uri的scheme是否正确，即是否是以content://开头，然后取出它的authority部分，最后调用另外一个成员函数acquireProvider执行获取ContentProvider接口的操作。在我们这个情景中，参数uri的authority的内容便是“shy.luo.providers.articles”了。

从ContentResolver类的定义我们可以看出，它是一个抽象类，两个参数版本的acquireProvider函数是由它的子类来实现的。回到Step 1中，这个ContentResolver接口是通过应用程序上下文Context对象的getContentResolver函数来获得的，而应用程序上下文Context是由ContextImpl类来实现的，它定义在frameworks/base/core/java/android/app/ContextImpl.java文件中：

```java
class ContextImpl extends Context {  
    ......  
  
    private ApplicationContentResolver mContentResolver;  
  
    ......  
  
    final void init(LoadedApk packageInfo,  
    IBinder activityToken, ActivityThread mainThread,  
    Resources container) {  
......  
  
mContentResolver = new ApplicationContentResolver(this, mainThread);  
  
......  
    }  
  
    ......  
  
    @Override  
    public ContentResolver getContentResolver() {  
return mContentResolver;  
    }  
  
    ......  
}  
```
 ContextImpl类的init函数是在应用程序启动的时候调用的，具体可以参考

Android应用程序启动过程源代码分析

一文中的Step 34。

 因此，在上面的ContentResolver类的acquireProvider函数里面接下来要调用的ApplicationContentResolver类的acquireProvider函数。

 Step 3. ApplicationContentResolve.acquireProvider

 这个函数定义在frameworks/base/core/java/android/app/ContextImpl.java文件中：

```java
class ContextImpl extends Context {  
    ......  
  
    private static final class ApplicationContentResolver extends ContentResolver {  
......  
  
@Override  
protected IContentProvider acquireProvider(Context context, String name) {  
    return mMainThread.acquireProvider(context, name);  
}  
  
......  
  
private final ActivityThread mMainThread;  
    }  
  
    ......  
}  
```
 它调用ActivityThread类的acquireProvider函数进一步执行获取Content Provider接口的操作。

 Step 4. ActivityThread.acquireProvider

 这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

```java
public final class ActivityThread {  
    ......  
  
    public final IContentProvider acquireProvider(Context c, String name) {  
IContentProvider provider = getProvider(c, name);  
if(provider == null)  
    return null;  
......  
return provider;  
    }  
  
    ......  
}  
```
 它又是调用了另外一个成员函数getProvider来进一步执行获取Content Provider接口的操作。

 Step 5. ActivityThread.getProvider

 这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

```java
public final class ActivityThread {  
    ......  
  
    private final IContentProvider getExistingProvider(Context context, String name) {  
synchronized(mProviderMap) {  
    final ProviderClientRecord pr = mProviderMap.get(name);  
    if (pr != null) {  
        return pr.mProvider;  
    }  
    return null;  
}  
    }  
  
    ......  
  
    private final IContentProvider getProvider(Context context, String name) {  
IContentProvider existing = getExistingProvider(context, name);  
if (existing != null) {  
    return existing;  
}  
  
IActivityManager.ContentProviderHolder holder = null;  
try {  
    holder = ActivityManagerNative.getDefault().getContentProvider(  
        getApplicationThread(), name);  
} catch (RemoteException ex) {  
}  
  
IContentProvider prov = installProvider(context, holder.provider,  
    holder.info, true);  
  
......  
  
return prov;  
    }  
  
    ......  
}  
```
 这个函数首先会通过getExistingProvider函数来检查本地是否已经存在这个要获取的ContentProvider接口，如果存在，就直接返回了。本地已经存在的ContextProvider接口保存在ActivityThread类的mProviderMap成员变量中，以ContentProvider对应的URI的authority为键值保存。在我们这个情景中，因为是第一次调用ArticlesProvider接口，因此，这时候通过getExistingProvider函数得到的IContentProvider接口为null，于是下面就会调用ActivityManagerService服务的getContentProvider接口来获取一个ContentProviderHolder对象holder，这个对象就包含了我们所要获取的ArticlesProvider接口，在将这个接口返回给调用者之后，还会调用installProvider函数来把这个接口保存在本地中，以便下次要使用这个ContentProvider接口时，直接就可以通过getExistingProvider函数获取了。

我们先进入到ActivityManagerService服务的getContentProvider函数中看看它是如何获取我们所需要的ArticlesProvider接口的，然后再返回来看看installProvider函数的实现。

Step 6. ActivityManagerService.getContentProvider

这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：

```java
public final class ActivityManagerService extends ActivityManagerNative  
implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
    ......  
  
    public final ContentProviderHolder getContentProvider(  
    IApplicationThread caller, String name) {  
......  
  
return getContentProviderImpl(caller, name);  
    }  
  
    ......  
}  
```
它调用getContentProviderImpl函数来进一步执行操作。

Step 7. ActivityManagerService.getContentProviderImpl

这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：

```java
public final class ActivityManagerService extends ActivityManagerNative  
implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
    ......  
  
    private final ContentProviderHolder getContentProviderImpl(  
    IApplicationThread caller, String name) {  
ContentProviderRecord cpr;  
ProviderInfo cpi = null;  
  
synchronized(this) {  
    ProcessRecord r = null;  
    if (caller != null) {  
        r = getRecordForAppLocked(caller);  
        ......  
    }  
  
    // First check if this content provider has been published...  
    cpr = mProvidersByName.get(name);  
    if (cpr != null) {  
        ......  
    } else {  
        try {  
            cpi = AppGlobals.getPackageManager().  
                resolveContentProvider(name,  
                STOCK_PM_FLAGS | PackageManager.GET_URI_PERMISSION_PATTERNS);  
        } catch (RemoteException ex) {  
        }  
        ......  
    }  
  
    cpr = mProvidersByClass.get(cpi.name);  
    final boolean firstClass = cpr == null;  
    if (firstClass) {  
        try {  
            ApplicationInfo ai =  
                AppGlobals.getPackageManager().  
                getApplicationInfo(  
                cpi.applicationInfo.packageName,  
                STOCK_PM_FLAGS);  
            ......  
            cpr = new ContentProviderRecord(cpi, ai);  
        } catch (RemoteException ex) {  
            // pm is in same process, this will never happen.  
        }  
    }  
  
    if (r != null && cpr.canRunHere(r)) {  
        // If this is a multiprocess provider, then just return its  
        // info and allow the caller to instantiate it.  Only do  
        // this if the provider is the same user as the caller's  
        // process, or can run as root (so can be in any process).  
        return cpr;  
    }  
  
    ......  
  
    // This is single process, and our app is now connecting to it.  
    // See if we are already in the process of launching this  
    // provider.  
    final int N = mLaunchingProviders.size();  
    int i;  
    for (i=0; i<N; i++) {  
        if (mLaunchingProviders.get(i) == cpr) {  
            break;  
        }  
    }  
  
    // If the provider is not already being launched, then get it  
    // started.  
    if (i >= N) {  
        final long origId = Binder.clearCallingIdentity();  
        ProcessRecord proc = startProcessLocked(cpi.processName,  
            cpr.appInfo, false, 0, "content provider",  
            new ComponentName(cpi.applicationInfo.packageName,  
            cpi.name), false);  
        ......  
        mLaunchingProviders.add(cpr);  
        ......  
    }  
  
    // Make sure the provider is published (the same provider class  
    // may be published under multiple names).  
    if (firstClass) {  
        mProvidersByClass.put(cpi.name, cpr);  
    }  
    cpr.launchingApp = proc;  
    mProvidersByName.put(name, cpr);  
  
    ......  
}  
  
// Wait for the provider to be published...  
synchronized (cpr) {  
    while (cpr.provider == null) {  
        ......  
        try {  
            cpr.wait();  
        } catch (InterruptedException ex) {  
        }  
    }  
}  
  
return cpr;  
    }  
      
    ......  
}  
```
这个函数比较长，我们一步一步地分析。

函数首先是获取调用者的进程记录块信息：

```java
ProcessRecord r = null;  
if (caller != null) {  
    r = getRecordForAppLocked(caller);  
    ......  
}  
```
在我们这个情景中，要获取的就是应用程序Article的进程记录块信息了，后面会用到。

在ActivityManagerService中，有两个成员变量是用来保存系统中的Content Provider信息的，一个是mProvidersByName，一个是mProvidersByClass，前者是以Content Provider的authoriry值为键值来保存的，后者是以Content Provider的类名为键值来保存的。一个Content Provider可以有多个authority，而只有一个类来和它对应，因此，这里要用两个Map来保存，这里为了方便根据不同条件来快速查找而设计的。下面的代码就是用来检查要获取的Content Provider是否已经加存在的了：

```java
// First check if this content provider has been published...  
cpr = mProvidersByName.get(name);  
if (cpr != null) {  
    ......  
} else {  
    try {  
cpi = AppGlobals.getPackageManager().  
    resolveContentProvider(name,  
    STOCK_PM_FLAGS | PackageManager.GET_URI_PERMISSION_PATTERNS);  
    } catch (RemoteException ex) {  
    }  
    ......  
}  
  
cpr = mProvidersByClass.get(cpi.name);  
final boolean firstClass = cpr == null;  
if (firstClass) {  
    try {  
ApplicationInfo ai =  
    AppGlobals.getPackageManager().  
    getApplicationInfo(  
    cpi.applicationInfo.packageName,  
    STOCK_PM_FLAGS);  
......  
cpr = new ContentProviderRecord(cpi, ai);  
    } catch (RemoteException ex) {  
// pm is in same process, this will never happen.  
    }  
}  
```
在我们这个情景中，由于是第一次调用ArticlesProvider接口，因此，在mProvidersByName和mProvidersByClass两个Map中都不存在ArticlesProvider的相关信息，因此，这里会通过AppGlobals.getPackageManager函数来获得PackageManagerService服务接口，然后分别通过它的resolveContentProvider和getApplicationInfo函数来分别获取ArticlesProvider应用程序的相关信息，分别保存在cpi和cpr这两个本地变量中。这些信息都是在安装应用程序的过程中保存下来的，具体可以参考

Android应用程序安装过程源代码分析一文。

接下去这个代码判断当前要获取的Content Provider是否允许在客户进程中加载，即查看一个这个Content Provider否配置了multiprocess属性为true，如果允许在客户进程中加载，就直接返回了这个Content Provider的信息了：

```java
if (r != null && cpr.canRunHere(r)) {  
    // If this is a multiprocess provider, then just return its  
    // info and allow the caller to instantiate it.  Only do  
    // this if the provider is the same user as the caller's  
    // process, or can run as root (so can be in any process).  
    return cpr;  
}  
```
在我们这个情景中，要获取的ArticlesProvider设置了要在独立的进程中运行，因此，继续往下执行：

```java
// This is single process, and our app is now connecting to it.  
// See if we are already in the process of launching this  
// provider.  
final int N = mLaunchingProviders.size();  
int i;  
for (i=0; i<N; i++) {  
    if (mLaunchingProviders.get(i) == cpr) {  
break;  
    }  
}  
```
系统中所有正在加载的Content Provider都保存在mLaunchingProviders成员变量中。在加载相应的Content Provider之前，首先要判断一下它是可否正在被其它应用程序加载，如果是的话，就不用重复加载了。在我们这个情景中，没有其它应用程序也正在加载ArticlesProvider这个Content Provider，继续往前执行：

```java
// If the provider is not already being launched, then get it  
// started.  
if (i >= N) {  
    final long origId = Binder.clearCallingIdentity();  
    ProcessRecord proc = startProcessLocked(cpi.processName,  
cpr.appInfo, false, 0, "content provider",  
new ComponentName(cpi.applicationInfo.packageName,  
cpi.name), false);  
    ......  
    mLaunchingProviders.add(cpr);  
    ......  
}  
```
这里的条件i >= N为true，就表明没有其它应用程序正在加载这个Content Provider，因此，就要调用startProcessLocked函数来启动一个新的进程来加载这个Content Provider对应的类了，然后把这个正在加载的信息增加到mLaunchingProviders中去。我们先接着分析这个函数，然后再来看在新进程中加载Content Provider的过程，继续往下执行：

```java
// Make sure the provider is published (the same provider class  
// may be published under multiple names).  
if (firstClass) {  
    mProvidersByClass.put(cpi.name, cpr);  
}  
cpr.launchingApp = proc;  
mProvidersByName.put(name, cpr);  
```
这段代码把这个Content Provider的信息分别保存到mProvidersByName和mProviderByCalss两个Map中去，以方便后续查询。

因为我们需要获取的Content Provider是在新的进程中加载的，而getContentProviderImpl这个函数是在系统进程中执行的，它必须要等到要获取的Content Provider是在新的进程中加载完成后才能返回，这样就涉及到进程同步的问题了。这里使用的同步方法是不断地去检查变量cpr的provider域是否被设置了。当要获取的Content Provider在新的进程加载完成之后，它会通过Binder进程间通信机制调用到系统进程中，把这个cpr变量的provider域设置为已经加载好的Content Provider接口，这时候，函数getContentProviderImpl就可以返回了。下面的代码就是用来等待要获取的Content Provider是在新的进程中加载完成的：

```java
// Wait for the provider to be published...  
synchronized (cpr) {  
    while (cpr.provider == null) {  
......  
try {  
    cpr.wait();  
} catch (InterruptedException ex) {  
}  
    }  
}  
```
下面我们再分析在新进程中加载ArticlesProvider这个Content Provider的过程。

 Step 8. ActivityManagerService.startProcessLocked

 Step 9. Process.start

 Step 10. ActivityThread.main

 Step 11. ActivityThread.attach

 Step 12. ActivityManagerService.attachApplication

 这五步是标准的Android应用程序启动步骤，具体可以参考[Android应用程序启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6689748)一文中的Step 23到Step 27，或者[Android系统在新进程中启动自定义服务过程（startService）的原理分析](http://blog.csdn.net/luoshengyang/article/details/6677029)一文中的Step 4到Step 9，这里就不再详细描述了。

 Step 13. ActivityManagerService.attachApplicationLocked

 这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：

```java
public final class ActivityManagerService extends ActivityManagerNative  
implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
    ......  
  
    private final boolean attachApplicationLocked(IApplicationThread thread,  
    int pid) {  
// Find the application record that is being attached...  either via  
// the pid if we are running in multiple processes, or just pull the  
// next app record if we are emulating process with anonymous threads.  
ProcessRecord app;  
if (pid != MY_PID && pid >= 0) {  
    synchronized (mPidsSelfLocked) {  
        app = mPidsSelfLocked.get(pid);  
    }  
} else if (mStartingProcesses.size() > 0) {  
    ......  
} else {  
    ......  
}  
  
......  
  
app.thread = thread;  
app.curAdj = app.setAdj = -100;  
app.curSchedGroup = Process.THREAD_GROUP_DEFAULT;  
app.setSchedGroup = Process.THREAD_GROUP_BG_NONINTERACTIVE;  
app.forcingToForeground = null;  
app.foregroundServices = false;  
app.debugging = false;  
  
......  
  
boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);  
List providers = normalMode ? generateApplicationProvidersLocked(app) : null;  
  
try {  
    ......  
  
    thread.bindApplication(processName, app.instrumentationInfo != null  
        ? app.instrumentationInfo : app.info, providers,  
        app.instrumentationClass, app.instrumentationProfileFile,  
        app.instrumentationArguments, app.instrumentationWatcher, testMode,  
        isRestrictedBackupMode || !normalMode,  
        mConfiguration, getCommonServicesLocked());  
  
    ......  
} catch (Exception e) {  
    ......  
}  
  
......  
  
return true;  
    }  
  
    ......  
  
    private final List generateApplicationProvidersLocked(ProcessRecord app) {  
List providers = null;  
try {  
    providers = AppGlobals.getPackageManager().  
        queryContentProviders(app.processName, app.info.uid,  
        STOCK_PM_FLAGS | PackageManager.GET_URI_PERMISSION_PATTERNS);  
} catch (RemoteException ex) {  
}  
if (providers != null) {  
    final int N = providers.size();  
    for (int i=0; i<N; i++) {  
        ProviderInfo cpi =  
            (ProviderInfo)providers.get(i);  
        ContentProviderRecord cpr = mProvidersByClass.get(cpi.name);  
        if (cpr == null) {  
            cpr = new ContentProviderRecord(cpi, app.info);  
            mProvidersByClass.put(cpi.name, cpr);  
        }  
        app.pubProviders.put(cpi.name, cpr);  
        app.addPackage(cpi.applicationInfo.packageName);  
        ensurePackageDexOpt(cpi.applicationInfo.packageName);  
    }  
}  
return providers;  
    }  
  
    ......  
}  
```
这个函数首先是根据传进来的进程ID找到相应的进程记录块，注意，这个进程ID是应用程序ArticlesProvider的ID，然后对这个进程记录块做一些初倾始化的工作。再接下来通过调用generateApplicationProvidersLocked获得需要在这个过程中加载的Content Provider列表，在我们这个情景中，就只有ArticlesProvider这个Content Provider了。最后调用从参数传进来的IApplicationThread对象thread的bindApplication函数来执行一些应用程序初始化工作。从

Android应用程序启动过程源代码分析

一文中我们知道，在Android系统中，每一个应用程序进程都加载了一个ActivityThread实例，在这个ActivityThread实例里面，有一个成员变量mAppThread，它是一个Binder对象，类型为ApplicationThread，实现了IApplicationThread接口，它是专门用来和ActivityManagerService服务进行通信的。因此，调用下面语句：

```java
thread.bindApplication(processName, app.instrumentationInfo != null  
    ? app.instrumentationInfo : app.info, providers,  
    app.instrumentationClass, app.instrumentationProfileFile,  
    app.instrumentationArguments, app.instrumentationWatcher, testMode,  
    isRestrictedBackupMode || !normalMode,  
    mConfiguration, getCommonServicesLocked());  
```
就会进入到应用程序ArticlesProvider进程中的ApplicationThread对象的bindApplication函数中去。在我们这个情景场，这个函数调用中最重要的参数便是第三个参数providers了，它是我们要处理的对象。

Step 14. ApplicationThread.bindApplication

这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

```java
public final class ActivityThread {  
    ......  
  
    private final class ApplicationThread extends ApplicationThreadNative {  
......  
  
public final void bindApplication(String processName,  
        ApplicationInfo appInfo, List<ProviderInfo> providers,  
        ComponentName instrumentationName, String profileFile,  
        Bundle instrumentationArgs, IInstrumentationWatcher instrumentationWatcher,  
        int debugMode, boolean isRestrictedBackupMode, Configuration config,  
        Map<String, IBinder> services) {  
    if (services != null) {  
        // Setup the service cache in the ServiceManager  
        ServiceManager.initServiceCache(services);  
    }  
  
    AppBindData data = new AppBindData();  
    data.processName = processName;  
    data.appInfo = appInfo;  
    data.providers = providers;  
    data.instrumentationName = instrumentationName;  
    data.profileFile = profileFile;  
    data.instrumentationArgs = instrumentationArgs;  
    data.instrumentationWatcher = instrumentationWatcher;  
    data.debugMode = debugMode;  
    data.restrictedBackupMode = isRestrictedBackupMode;  
    data.config = config;  
    queueOrSendMessage(H.BIND_APPLICATION, data);  
}  
  
......  
    }  
  
    ......  
}  
```
 这个函数把相关的信息都封装成一个AppBindData对象，然后以一个消息的形式发送到主线程的消息队列中去等等待处理。这个消息最终是是在ActivityThread类的handleBindApplication函数中进行处理的。

 Step 15. ActivityThread.handleBindApplication

 这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

```java
public final class ActivityThread {  
    ......  
      
    private final void handleBindApplication(AppBindData data) {  
......  
  
List<ProviderInfo> providers = data.providers;  
if (providers != null) {  
    installContentProviders(app, providers);  
    ......  
}  
  
......  
    }  
  
    ......  
}  
```
 这个函数的内容比较多，我们忽略了其它无关的部分，只关注和Content Provider有关的逻辑，这里主要就是调用installContentProviders函数来在本地安装Content Providers信息。

 Step 16. ActivityThread.installContentProviders

 这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

```java
public final class ActivityThread {  
    ......  
      
    private final void installContentProviders(  
    Context context, List<ProviderInfo> providers) {  
final ArrayList<IActivityManager.ContentProviderHolder> results =  
    new ArrayList<IActivityManager.ContentProviderHolder>();  
  
Iterator<ProviderInfo> i = providers.iterator();  
while (i.hasNext()) {  
    ProviderInfo cpi = i.next();  
    StringBuilder buf = new StringBuilder(128);  
    buf.append("Pub ");  
    buf.append(cpi.authority);  
    buf.append(": ");  
    buf.append(cpi.name);  
    Log.i(TAG, buf.toString());  
    IContentProvider cp = installProvider(context, null, cpi, false);  
    if (cp != null) {  
        IActivityManager.ContentProviderHolder cph =  
            new IActivityManager.ContentProviderHolder(cpi);  
        cph.provider = cp;  
        results.add(cph);  
        // Don't ever unload this provider from the process.  
        synchronized(mProviderMap) {  
            mProviderRefCountMap.put(cp.asBinder(), new ProviderRefCount(10000));  
        }  
    }  
}  
  
try {  
    ActivityManagerNative.getDefault().publishContentProviders(  
        getApplicationThread(), results);  
} catch (RemoteException ex) {  
}  
    }  
  
    ......  
}  
```
 这个函数主要是做了两件事情，一是调用installProvider来在本地安装每一个Content Proivder的信息，并且为每一个Content Provider创建一个ContentProviderHolder对象来保存相关的信息。ContentProviderHolder对象是一个Binder对象，是用来把Content Provider的信息传递给ActivityManagerService服务的。当这些Content Provider都处理好了以后，还要调用ActivityManagerService服务的publishContentProviders函数来通知ActivityManagerService服务，这个进程中所要加载的Content Provider，都已经准备完毕了，而ActivityManagerService服务的publishContentProviders函数的作用就是用来唤醒在前面Step 7等待的线程的了。我们先来看installProvider的实现，然后再来看ActivityManagerService服务的publishContentProviders函数的实现。

Step 17. ActivityThread.installProvider

这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

```java
public final class ActivityThread {  
    ......  
      
    private final IContentProvider installProvider(Context context,  
    IContentProvider provider, ProviderInfo info, boolean noisy) {  
ContentProvider localProvider = null;  
if (provider == null) {  
    ......  
  
    Context c = null;  
    ApplicationInfo ai = info.applicationInfo;  
    if (context.getPackageName().equals(ai.packageName)) {  
        c = context;  
    } else if (mInitialApplication != null &&  
        mInitialApplication.getPackageName().equals(ai.packageName)) {  
            c = mInitialApplication;  
    } else {  
        try {  
            c = context.createPackageContext(ai.packageName,  
                Context.CONTEXT_INCLUDE_CODE);  
        } catch (PackageManager.NameNotFoundException e) {  
        }  
    }  
  
    ......  
  
    try {  
        final java.lang.ClassLoader cl = c.getClassLoader();  
        localProvider = (ContentProvider)cl.  
            loadClass(info.name).newInstance();  
        provider = localProvider.getIContentProvider();  
        ......  
  
        // XXX Need to create the correct context for this provider.  
        localProvider.attachInfo(c, info);  
    } catch (java.lang.Exception e) {  
        ......  
    }  
  
} else if (localLOGV) {  
    ......  
}  
  
synchronized (mProviderMap) {  
    // Cache the pointer for the remote provider.  
    String names[] = PATTERN_SEMICOLON.split(info.authority);  
    for (int i=0; i<names.length; i++) {  
        ProviderClientRecord pr = new ProviderClientRecord(names[i], provider,  
            localProvider);  
        try {  
            provider.asBinder().linkToDeath(pr, 0);  
            mProviderMap.put(names[i], pr);  
        } catch (RemoteException e) {  
            return null;  
        }  
    }  
    if (localProvider != null) {  
        mLocalProviders.put(provider.asBinder(),  
            new ProviderClientRecord(null, provider, localProvider));  
    }  
}  
  
return provider;  
    }  
  
    ......  
}  
```
 这个函数的作用主要就是在应用程序进程中把相应的Content Provider类加载进来了，在我们这个种情景中，就是要在ArticlesProvider这个应用程序中把ArticlesProvider这个Content Provider类加载到内存中来了：

```java
final java.lang.ClassLoader cl = c.getClassLoader();  
localProvider = (ContentProvider)cl.  
    loadClass(info.name).newInstance();  
```
 接着通过调用localProvider的getIContentProvider函数来获得一个Binder对象，这个Binder对象返回给installContentProviders函数之后，就会传到ActivityManagerService中去，后续其它应用程序就是通过获得这个Binder对象来和相应的Content Provider进行通信的了。我们先看一下这个函数的实现，然后再回到installProvider函数中继续分析。

 Step 18. ContentProvider.getIContentProvider

 这个函数定义在frameworks/base/core/java/android/content/ContentProvider.java文件中：

```java
public abstract class ContentProvider implements ComponentCallbacks {  
    ......  
  
    private Transport mTransport = new Transport();  
  
    ......  
  
    class Transport extends ContentProviderNative {  
......  
    }  
  
    public IContentProvider getIContentProvider() {  
return mTransport;  
    }  
  
    ......  
}  
```
从这里我们可以看出，ContentProvider类和Transport类的关系就类似于ActivityThread和ApplicationThread的关系，其它应用程序不是直接调用ContentProvider接口来访问它的数据，而是通过调用它的内部对象mTransport来间接调用ContentProvider的接口，这一点我们在下一篇文章中分析调用Content Provider接口来获取共享数据时将会看到。

回到前面的installProvider函数中，它接下来调用下面接口来初始化刚刚加载好的Content Provider：

```java
// XXX Need to create the correct context for this provider.  
localProvider.attachInfo(c, info);  
```
同样，我们先进入到ContentProvider类的attachInfo函数去看看它的实现，然后再回到installProvider函数来。

Step 19. ContentProvider.attachInfo

这个函数定义在frameworks/base/core/java/android/content/ContentProvider.java文件中：

```java
public abstract class ContentProvider implements ComponentCallbacks {  
    ......  
  
    public void attachInfo(Context context, ProviderInfo info) {  
/* 
* Only allow it to be set once, so after the content service gives 
* this to us clients can't change it. 
*/  
if (mContext == null) {  
    mContext = context;  
    mMyUid = Process.myUid();  
    if (info != null) {  
        setReadPermission(info.readPermission);  
        setWritePermission(info.writePermission);  
        setPathPermissions(info.pathPermissions);  
        mExported = info.exported;  
    }  
    ContentProvider.this.onCreate();  
}  
    }  
  
    ......  
}  
```
这个函数很简单，主要就是根据这个Content Provider的信息info来设置相应的读写权限，然后调用它的子类的onCreate函数来让子类执行一些初始化的工作。在我们这个情景中，这个子类就是ArticlesProvide应用程序中的ArticlesProvider类了。

Step 20. ArticlesProvider.onCreate

这个函数定义在前面一篇文章[Android应用程序组件Content Provider应用实例](http://blog.csdn.net/luoshengyang/article/details/6950440)介绍的应用程序ArtilcesProvider源代码工程目录下，在文件为packages/experimental/ArticlesProvider/src/shy/luo/providers/articles/ArticlesProvider.java中：

```java
public class ArticlesProvider extends ContentProvider {  
    ......  
  
    @Override  
    public boolean onCreate() {  
Context context = getContext();  
resolver = context.getContentResolver();  
dbHelper = new DBHelper(context, DB_NAME, null, DB_VERSION);  
  
return true;  
    }  
  
    ......  
}  
```
这个函数主要执行一些简单的工作，例如，获得应用程序上下文的ContentResolver接口和创建数据库操作辅助对象，具体可以参考前面一篇文章

## Android应用程序组件Content Provider应用实例

回到前面Step 17中的installProvider函数中，它接下来就是把这些在本地中加载的Content Provider信息保存下来了，以方便后面查询和使用：

```java
synchronized (mProviderMap) {  
    // Cache the pointer for the remote provider.  
    String names[] = PATTERN_SEMICOLON.split(info.authority);  
    for (int i=0; i<names.length; i++) {  
    ProviderClientRecord pr = new ProviderClientRecord(names[i], provider,  
localProvider);  
    try {  
provider.asBinder().linkToDeath(pr, 0);  
mProviderMap.put(names[i], pr);  
    } catch (RemoteException e) {  
return null;  
    }  
    }  
    if (localProvider != null) {  
    mLocalProviders.put(provider.asBinder(),  
new ProviderClientRecord(null, provider, localProvider));  
    }  
}  
```
和ActivityMangerService类似，在ActivityThread中，以Content Provider的authority为键值来把这个Content Provider的信息保存在mProviderMap成员变量中，因为一个Content Provider可以对应多个authority，因此这里用一个for循环来处理，同时又以这个Content Provider对应的Binder对象provider来键值来把这个Content Provider的信息保存在mLocalProviders成员变量中，表明这是一个在本地加载的Content Provider。

函数installProvider执行完成以后，返回到Step 16中的instalContentProviders函数中，执行下面语句：

```java
try {  
    ActivityManagerNative.getDefault().publishContentProviders(  
getApplicationThread(), results);  
} catch (RemoteException ex) {  
}  
```
前面已经提到，这个函数调用的作用就是通知ActivityMangerService，需要在这个进程中加载的Content Provider已经完加载完成了，参数results就包含了这些已经加载好的Content Provider接口。

Step 21. ActivityMangerService.publishContentProviders

这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：

```java
public final class ActivityManagerService extends ActivityManagerNative  
implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
    ......  
  
    public final void publishContentProviders(IApplicationThread caller,  
    List<ContentProviderHolder> providers) {  
......  
  
synchronized(this) {  
    final ProcessRecord r = getRecordForAppLocked(caller);  
    ......  
  
    final int N = providers.size();  
    for (int i=0; i<N; i++) {  
        ContentProviderHolder src = providers.get(i);  
        if (src == null || src.info == null || src.provider == null) {  
            continue;  
        }  
        ContentProviderRecord dst = r.pubProviders.get(src.info.name);  
        if (dst != null) {  
            mProvidersByClass.put(dst.info.name, dst);  
            String names[] = dst.info.authority.split(";");  
            for (int j = 0; j < names.length; j++) {  
                mProvidersByName.put(names[j], dst);  
            }  
  
            int NL = mLaunchingProviders.size();  
            int j;  
            for (j=0; j<NL; j++) {  
                if (mLaunchingProviders.get(j) == dst) {  
                    mLaunchingProviders.remove(j);  
                    j--;  
                    NL--;  
                }  
            }  
            synchronized (dst) {  
                dst.provider = src.provider;  
                dst.app = r;  
                dst.notifyAll();  
            }  
            ......  
        }  
    }  
}  
    }  
  
    ......  
}  
```
 在我们这个情景中，只有一个Content Provider，因此，这里的N等待1。在中间的for循环里面，最重要的是下面这个语句：

```java
ContentProviderRecord dst = r.pubProviders.get(src.info.name);  
```
 从这里得到的ContentProviderRecord对象dst，就是在前面Step 7中创建的ContentProviderRecord对象cpr了。在for循环中，首先是把这个Content Provider信息保存好在mProvidersByClass和mProvidersByName中：

```java
mProvidersByClass.put(dst.info.name, dst);  
String names[] = dst.info.authority.split(";");  
for (int j = 0; j < names.length; j++) {  
    mProvidersByName.put(names[j], dst);  
}  
```
前面已经说过，这两个Map中，一个是以类名为键值保存Content Provider信息，一个是以authority为键值保存Content Provider信息。

因为这个Content Provider已经加载好了，因此，把它从mLaunchingProviders列表中删除：

```java
int NL = mLaunchingProviders.size();  
int j;  
for (j=0; j<NL; j++) {  
    if (mLaunchingProviders.get(j) == dst) {  
mLaunchingProviders.remove(j);  
j--;  
NL--;  
    }  
}  
```
最后，设置这个ContentProviderRecord对象dst的provider域为从参数传进来的Content Provider远程接口：

```java
synchronized (dst) {  
    dst.provider = src.provider;  
    dst.app = r;  
    dst.notifyAll();  
}  
```
执行了dst.notiryAll语句后，在Step 7中等待要获取的Content Provider接口加载完毕的线程就被唤醒了。唤醒之后，它检查本地ContentProviderRecord变量cpr的provider域不为null，于是就返回了。它最终返回到Step 5中的ActivityThread类的getProvider函数中，继续往下执行：

```java
IContentProvider prov = installProvider(context, holder.provider,  
    holder.info, true);  
```
注意，这里是在Article应用程序中进程中执行installProvider函数的，而前面的Step 17的installProvider函数是在ArticlesProvider应用程序进程中执行的。

Step 22. ActivityThread.installProvider

这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

```java
public final class ActivityThread {  
    ......  
      
    private final IContentProvider installProvider(Context context,  
    IContentProvider provider, ProviderInfo info, boolean noisy) {  
......  
if (provider == null) {  
    ......  
} else if (localLOGV) {  
    ......  
}  
  
synchronized (mProviderMap) {  
    // Cache the pointer for the remote provider.  
    String names[] = PATTERN_SEMICOLON.split(info.authority);  
    for (int i=0; i<names.length; i++) {  
        ProviderClientRecord pr = new ProviderClientRecord(names[i], provider,  
            localProvider);  
        try {  
            provider.asBinder().linkToDeath(pr, 0);  
            mProviderMap.put(names[i], pr);  
        } catch (RemoteException e) {  
            return null;  
        }  
    }  
    ......  
}  
  
return provider;  
    }  
  
    ......  
}  
```
同样是执行installProvider函数，与Step 17不同，这里传进来的参数provider是不为null的，因此，它不需要执行在本地加载Content Provider的工作，只需要把从ActivityMangerService中获得的Content Provider接口保存在成员变量mProviderMap中就可以了。

这样，获取与"shy.luo.providers.artilces"这个uri对应的Content Provider（shy.luo.providers.articles.ArticlesProvider）就完成了，它同时也是启动Content Provider的完整过程。第三方应用程序获得了这个Content Provider的接口之后，就可以访问它里面的共享数据了。在下面一篇文章中，我们将重点分析Android应用程序组件Content Provider在不同进程中传输数据的过程，即Content Provider在不同应用程序中共享数据的原理，敬请关注。