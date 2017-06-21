在前面一篇文章中，我们分析了[Android](http://lib.csdn.net/base/android)应用程序资源的编译和打包过程，最终得到的应用程序资源就与应用程序代码一起打包在一个APK文件中。[android](http://lib.csdn.net/base/android)应用程序在运行的过程中，是通过一个称为AssetManager的资源管理器来读取打包在APK文件里面的资源文件的。在本文中，我们就将详细分析Android应用程序资源管理器的创建以及初始化过程，为接下来的一篇文章分析应用程序资源的读取过程打下基础。

《Android系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

从前面[Android应用程序窗口（Activity）的运行上下文环境（Context）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8201936)一文可以知道，应用程序的每一个Activity组件都关联有一个ContextImpl对象，这个ContextImpl对象就是用来描述Activity组件的运行上下文环境的。Activity组件是从Context类继承下来的，而ContextImpl同样是从Context类继承下来的。我们在Activity组件调用的大部分成员函数都是转发给与它所关联的一个ContextImpl对象的对应的成员函数来处理的，其中就包括用来访问应用程序资源的两个成员函数getResources和getAssets。

ContextImpl类的成员函数getResources返回的是一个Resources对象，有了这个Resources对象之后，我们就可以通过资源ID来访问那些被编译过的应用程序资源了。ContextImpl类的成员函数getAssets返回的是一个AssetManager对象，有了这个AssetManager对象之后，我们就可以通过文件名来访问那些被编译过或者没有被编译过的应用程序资源文件了。事实上，Resources类也是通过AssetManager类来访问那些被编译过的应用程序资源文件的，不过在访问之前，它会先根据资源ID查找得到对应的资源文件名。

我们知道，在Android系统中，一个进程是可以同时加载多个应用程序的，也就是可以同时加载多个APK文件。每一个APK文件在进程中都对应有一个全局的Resourses对象以及一个全局的AssetManager对象。其中，这个全局的Resourses对象保存在一个对应的ContextImpl对象的成员变量mResources中，而这个全局的AssetManager对象保存在这个全局的Resourses对象的成员变量mAssets中。上述ContextImpl、Resourses和AssetManager的关系如图1所示：

![img](http://img.my.csdn.net/uploads/201304/12/1365697642_3078.jpg)

图1 ContextImpl、Resources和AssetManager的关系图

Resources类有一个成员函数getAssets，通过它就可以获得保存在Resources类的成员变量mAssets中的AssetManager，例如，ContextImpl类的成员函数getAssets就是通过调用其成员变量mResources所指向的一个Resources对象的成员函数getAssets来获得一个可以用来访问应用程序的非编译资源文件的AssetManager。

我们知道，Android应用程序除了要访问自己的资源之外，还需要访问系统的资源。系统的资源打包在/system/framework/framework-res.apk文件中，它在应用程序进程中是通过一个单独的Resources对象和一个单独的AssetManager对象来管理的。这个单独的Resources对象就保存在Resources类的静态成员变量mSystem中，我们可以通过Resources类的静态成员函数getSystem就可以获得这个Resources对象，而这个单独的AssetManager对象就保存在AssetManager类的静态成员变量sSystem中，我们可以通过AssetManager类的静态成员函数getSystem同样可以获得这个AssetManager对象。

AssetManager类除了在[Java](http://lib.csdn.net/base/java)层有一个实现之外，在 C++层也有一个对应的实现，而Java层的AssetManager类的功能就是通过C++层的AssetManager类来实现的。Java层的每一个AssetManager对象都有一个类型为int的成员变量mObject，它保存的便是在C++层对应的AssetManager对象的地址，因此，通过这个成员变量就可以将Java层的AssetManager对象与C++层的AssetManager对象关联起来。

C++层的AssetManager类有三个重要的成员变量mAssetPaths、mResources和mConfig。其中，mAssetPaths保存的是资源存放目录，mResources指向的是一个资源索引表，而mConfig保存的是设备的本地配置信息，例如屏幕密度和大小、国家地区和语言等等配置信息。有了这三个成员变量之后，C++层的AssetManager类就可以访问应用程序的资源了。

从前面[Android应用程序启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6689748)一文可以知道，每一个Activity组件在进程的加载过程中，都会创建一个对应的ContextImpl，并且调用这个ContextImpl对象的成员函数init来执行初始化Activity组件运行上下文环境的工作，其中就包括创建用来访问应用程序资源的Resources对象和AssetManager对象的工作，接下来，我们就从ContextImpl类的成员函数init开始分析Resources对象和AssetManager对象的创建以及初始化过程，如图2所示：

![img](http://img.my.csdn.net/uploads/201304/12/1365699210_3199.jpg)

图2 应用程序资源管理器的创建和初始化过程

这个过程可以分为14个步骤，接下来我们就详细分析每一个步骤。

### Step 1. ContextImpl.init

```java
class ContextImpl extends Context {  
    ......  

    /*package*/ LoadedApk mPackageInfo;  
    private Resources mResources;  
    ......  

    final void init(LoadedApk packageInfo,  
    IBinder activityToken, ActivityThread mainThread) {  
init(packageInfo, activityToken, mainThread, null);  
    }  

    final void init(LoadedApk packageInfo,  
        IBinder activityToken, ActivityThread mainThread,  
        Resources container) {  
mPackageInfo = packageInfo;  
mResources = mPackageInfo.getResources(mainThread);  

......  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/app/ContextImpl.java中。

参数packageInfo指向的是一个LoadedApk对象，这个LoadedApk对象描述的是当前正在启动的Activity组所属的Apk。三个参数版本的成员函数init调用了四个参数版本的成员函数init来初始化当前正在启动的Activity组件的运行上下文环境。其中，用来访问应用程序资源的Resources对象是通过调用参数packageInfo所指向的是一个LoadedApk对象的成员函数getResources来创建的。这个Resources对象创建完成之后，就会保存在ContextImpl类的成员变量mResources中。

接下来，我们就继续分析LoadedApk类的成员函数getResources的实现。

### Step 2. LoadedApk.getResources

```java
final class LoadedApk {  
    ......  

    private final String mResDir;  
    ......  

    Resources mResources;  
    ......  

    public Resources getResources(ActivityThread mainThread) {  
if (mResources == null) {  
    mResources = mainThread.getTopLevelResources(mResDir, this);  
}  
return mResources;  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/app/LoadedApk.java中。

参数mainThread指向了一个ActivityThread对象，这个ActivityThread对象描述的是当前正在运行的应用程序进程。

LoadedApk类的成员函数getResources首先检查其成员变量mResources的值是否等于null。如果不等于的话，那么就会将它所指向的是一个Resources对象返回给调用者，否则的话，就会调用参数mainThread所指向的一个ActivityThread对象的成员函数getTopLevelResources来获得这个Resources对象，然后再返回给调用者。

在调用ActivityThread类的成员函数getTopLevelResources来获得一个Resources对象的时候，需要指定要获取的Resources对象所对应的Apk文件路径，这个Apk文件路径就保存在LoadedApk类的成员变量mResDir中。例如，假设我们要获取的Resources对象是用来访问系统自带的音乐播放器的资源的，那么对应的Apk文件路径就为/system/app/Music.apk。

接下来，我们就继续分析ActivityThread类的成员函数getTopLevelResources的实现。

### Step 3. ActivityThread.getTopLevelResources

```java
public final class ActivityThread {  
    ......  

    final HashMap<ResourcesKey, WeakReference<Resources> > mActiveResources  
    = new HashMap<ResourcesKey, WeakReference<Resources> >();  
    ......  

    Resources getTopLevelResources(String resDir, CompatibilityInfo compInfo) {  
ResourcesKey key = new ResourcesKey(resDir, compInfo.applicationScale);  
Resources r;  
synchronized (mPackages) {  
    ......  

    WeakReference<Resources> wr = mActiveResources.get(key);  
    r = wr != null ? wr.get() : null;  
    ......  

    if (r != null && r.getAssets().isUpToDate()) {  
        ......  
        return r;  
    }  
}  

......  

AssetManager assets = new AssetManager();  
if (assets.addAssetPath(resDir) == 0) {  
    return null;  
}  
......  

r = new Resources(assets, metrics, getConfiguration(), compInfo);  
......  

synchronized (mPackages) {  
    WeakReference<Resources> wr = mActiveResources.get(key);  
    Resources existing = wr != null ? wr.get() : null;  
    if (existing != null && existing.getAssets().isUpToDate()) {  
        // Someone else already created the resources while we were  
        // unlocked; go ahead and use theirs.  
        r.getAssets().close();  
        return existing;  
    }  

    // XXX need to remove entries when weak references go away  
    mActiveResources.put(key, new WeakReference<Resources>(r));  
    return r;  
}  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/app/ActivityThread.java中。

ActivityThread类的成员变量mActiveResources指向的是一个HashMap。这个HashMap用来维护在当前应用程序进程中加载的每一个Apk文件及其对应的Resources对象的对应关系。也就是说，给定一个Apk文件路径，ActivityThread类的成员函数getTopLevelResources可以在成员变量mActiveResources中检查是否存在一个对应的Resources对象。如果不存在，那么就会新建一个，并且保存在ActivityThread类的成员变量mActiveResources中。

参数resDir即为要获取其对应的Resources对象的Apk文件路径，ActivityThread类的成员函数getTopLevelResources首先根据它来创建一个ResourcesKey对象，然后再以这个ResourcesKey对象在ActivityThread类的成员变量mActiveResources中检查是否存在一个Resources对象。如果存在，并且这个Resources对象里面包含的资源文件没有过时，即调用这个Resources对象的成员函数getAssets所获得一个AssetManager对象的成员函数isUpToDate的返回值等于true，那么ActivityThread类的成员函数getTopLevelResources就可以将该Resources对象返回给调用者了。

如果不存在与参数resDir对应的Resources对象，或者存在这个Resources对象，但是存在的这个Resources对象是过时的，那么ActivityThread类的成员函数getTopLevelResources就会新创建一个AssetManager对象，并且调用这个新创建的AssetManager对象的成员函数addAssetPath来将参数resDir所描述的Apk文件路径作为它的资源目录。

创建了一个新的AssetManager对象，ActivityThread类的成员函数getTopLevelResources还需要这个AssetManager对象来创建一个新的Resources对象。这个新创建的Resources对象需要以前面所创建的ResourcesKey对象为键值缓存在ActivityThread类的成员变量mActiveResources所描述的一个HashMap中，以便以后可以获取回来使用。不过，这个新创建的Resources对象在缓存到ActivityThread类的成员变量mActiveResources所描述的一个HashMap去之前，需要再次检查该HashMap是否已经存在一个对应的Resources对象了，这是因为当前线程在创建新的AssetManager对象和Resources对象的过程中，可能有其它线程抢先一步创建了与参数resDir对应的Resources对象，并且将该Resources对象保存到该HashMap中去了。

如果没有其它线程抢先创建一个与参数resDir对应的Resources对象，或者其它线程抢先创建出来的Resources对象是过时的，那么ActivityThread类的成员函数getTopLevelResources就会将前面创建的Resources对象缓存到成员变量mActiveResources所描述的一个HashMap中去，并且将前面创建的Resources对象返回给调用者，否则扩知，就会将其它线程抢先创建的Resources对象返回给调用者。

接下来，我们首先分析AssetManager类的构造函数和成员函数addAssetPath的实现，接着再分析Resources类的构造函数的实现，以便可以了解用来访问应用程序资源的AssetManager对象和Resources对象的创建以及初始化过程。

### Step 4. new AssetManager

```java
public final class AssetManager {  
    ......  

    private static AssetManager sSystem = null;  
    ......  

    public AssetManager() {  
synchronized (this) {  
    ......  
    init();  
    ......  
    ensureSystemAssets();  
}  
    }  

    private static void ensureSystemAssets() {  
synchronized (sSync) {  
    if (sSystem == null) {  
        AssetManager system = new AssetManager(true);  
        system.makeStringBlocks(false);  
        sSystem = system;  
    }  
}  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/content/res/AssetManager.java中。

AssetManager类的构造函数是通过调用另外一个成员函数init来执行初始化工作的。在初始化完成当前正在创建的AssetManager对象之后，AssetManager类的构造函数还会调用另外一个成员函数ensureSystemAssets来检查当前进程是否已经创建了一个用来访问系统资源的AssetManager对象。

如果用来访问系统资源的AssetManager对象还没有创建的话，那么AssetManager类的成员函数ensureSystemAssets就会创建并且初始化它，并且将它保存在AssetManager类的静态成员变量sSystem中。注意，创建用来访问系统资源和应用程序资源的AssetManager对象的过程是一样的，区别只在于它们所要访问的Apk文件不一样，因此，接下来我们就只分析用来访问应用资源的AssetManager对象的创建过程以及初始化过程。

### Step 5. AssetManager.init

```java
public final class AssetManager {  
    ......  

    private int mObject;  
    ......  

    private native final void init();  

    ......  
}  

       这个函数定义在文件frameworks/base/core/java/android/content/res/AssetManager.java中。

       AssetManager类的成员函数init是一个JNI函数，它是由C++层的函数android_content_AssetManager_init来实现的：

​```c
static void android_content_AssetManager_init(JNIEnv* env, jobject clazz)  
{  
    AssetManager* am = new AssetManager();  
    .....  

    am->addDefaultAssets();  

    ......  
    env->SetIntField(clazz, gAssetManagerOffsets.mObject, (jint)am);  
}  
```
这个函数定义在文件frameworks/base/core/jni/android_util_AssetManager.cpp中。

函数android_content_AssetManager_init首先创建一个C++层的AssetManager对象，接着调用这个C++层的AssetManager对象的成员函数addDefaultAssets来添加默认的资源路径，最后将这个这个C++层的AssetManager对象的地址保存在参数clazz所描述的一个Java层的AssetManager对象的成员变量mObject中。

### Step 6. AssetManager.addDefaultAssets

```c
static const char* kSystemAssets = "framework/framework-res.apk";  
......  

bool AssetManager::addDefaultAssets()  
{  
    const char* root = getenv("ANDROID_ROOT");  
    ......  

    String8 path(root);  
    path.appendPath(kSystemAssets);  

    return addAssetPath(path, NULL);  
}  
```
这个函数定义在文件frameworks/base/libs/utils/AssetManager.cpp中。

AssetManager类的成员函数addDefaultAssets首先通过环境变量ANDROID_ROOT来获得Android的系统路径，接着再将全局变量kSystemAssets所指向的字符串“framework/framework-res.apk”附加到这个系统路径的后面去，这样就可以得到系统资源文件framework-res.apk的绝对路径了。一般来说，环境变量ANDROID_ROOT所设置的Android系统路径就是“/system”，因此，最终得到的系统资源文件的绝对路径就为“/system/framework/framework-res.apk”。

得到了系统资源文件framework-res.apk的绝对路径之后，就调用AssetManager类的成员函数addAssetPath来将它添加到当前正在初始化的AssetManager对象中去。

### Step 7. AssetManager.addAssetPath

```c
static const char* kAppZipName = NULL; //"classes.jar";  
......  

bool AssetManager::addAssetPath(const String8& path, void** cookie)  
{  
    AutoMutex _l(mLock);  

    asset_path ap;  

    String8 realPath(path);  
    if (kAppZipName) {  
realPath.appendPath(kAppZipName);  
    }  
    ap.type = ::getFileType(realPath.string());  
    if (ap.type == kFileTypeRegular) {  
ap.path = realPath;  
    } else {  
ap.path = path;  
ap.type = ::getFileType(path.string());  
if (ap.type != kFileTypeDirectory && ap.type != kFileTypeRegular) {  
    ......  
    return false;  
}  
    }  

    // Skip if we have it already.  
    for (size_t i=0; i<mAssetPaths.size(); i++) {  
if (mAssetPaths[i].path == ap.path) {  
    if (cookie) {  
        *cookie = (void*)(i+1);  
    }  
    return true;  
}  
    }  

    ......  

    mAssetPaths.add(ap);  

    // new paths are always added at the end  
    if (cookie) {  
*cookie = (void*)mAssetPaths.size();  
    }  

    // add overlay packages for /system/framework; apps are handled by the  
    // (Java) package manager  
    if (strncmp(path.string(), "/system/framework/", 18) == 0) {  
// When there is an environment variable for /vendor, this  
// should be changed to something similar to how ANDROID_ROOT  
// and ANDROID_DATA are used in this file.  
String8 overlayPath("/vendor/overlay/framework/");  
overlayPath.append(path.getPathLeaf());  
if (TEMP_FAILURE_RETRY(access(overlayPath.string(), R_OK)) == 0) {  
    asset_path oap;  
    oap.path = overlayPath;  
    oap.type = ::getFileType(overlayPath.string());  
    bool addOverlay = (oap.type == kFileTypeRegular); // only .apks supported as overlay  
    if (addOverlay) {  
        oap.idmap = idmapPathForPackagePath(overlayPath);  

        if (isIdmapStaleLocked(ap.path, oap.path, oap.idmap)) {  
            addOverlay = createIdmapFileLocked(ap.path, oap.path, oap.idmap);  
        }  
    }  
    if (addOverlay) {  
        mAssetPaths.add(oap);  
    }   
    ......  
}  
    }  

    return true;  
}  
```
这个函数定义在文件frameworks/base/libs/utils/AssetManager.cpp中。

如果全局变量kAppZipName的值不等于NULL的话，那么它的值一般就是被设置为“classes.jar”，这时候就表示应用程序的资源文件是保存在参数path所描述的一个目录下的一个classes.jar文件中。全局变量kAppZipName的值一般被设置为NULL，并且参数path指向的是一个Apk文件，因此，接下来我们只考虑应用程序资源不是保存在一个classes.jar文件的情况。

AssetManager类的成员函数addAssetPath首先是要检查参数path指向的是一个文件或者目录，并且该文件或者目录存在，否则的话，它就会直接返回一个false值，而不会再继续往下处理了。

AssetManager类的成员函数addAssetPath接着再检查在其成员变量mAssetPaths所描述的一个类型为asset_path的Vector中是否已经添加过参数path所描述的一个Apk文件路径了。如果已经添加过了，那么AssetManager类的成员函数addAssetPath就不会再继续往下处理了，而是将与参数path所描述的一个Apk文件路径所对应的一个Cookie返回给调用者，即保存在输出参数cookie中，前提是参数cookie的值不等于NULL。一个Apk文件路径所对应的Cookie实际上只是一个整数，这个整数表示该Apk文件路径所对应的一个asset_path对象在成员变量mAssetPaths所描述的一个Vector中的索引再加上1。

经过上面的检查之后，AssetManager类的成员函数addAssetPath确保参数path所描述的一个Apk文件路径之前没有被添加过，于是接下来就会将与该Apk文件路径所对应的一个asset_path对象保存在成员变量mAssetPaths所描述的一个Vector的最末尾位置上，并且将这时候得到的Vector的大小作为一个Cookie值保存在输出参数cookie中返回给调用者。

AssetManager类的成员函数addAssetPath的最后一个工作是检查刚刚添加的Apk文件路径是否是保存在/system/framework/目录下面的。如果是的话，那么就会在/vendor/overlay/framework/目录下找到一个同名的Apk文件，并且也会将该Apk文件添加到成员变量mAssetPaths所描述的一个Vector中去。这是一种资源覆盖机制，手机厂商可以利用它来自定义的系统资源，即用自定义的系统资源来覆盖系统默认的系统资源，以达到个性化系统界面的目的。

如果手机厂商要利用上述的资源覆盖机制来自定义自己的系统资源，那么还需要提供一个idmap文件，用来说明它在/vendor/overlay/framework/目录提供的Apk文件要覆盖系统的哪些默认资源，使用资源ID来描述，因此，这个idmap文件实际上就是一个资源ID映射文件。这个idmap文件最终保存在/data/resource-cache/目录下，并且按照一定的格式来命令，例如，假设手机厂商提供的覆盖资源文件为/vendor/overlay/framework/framework-res.apk，那么对应的idmap文件就会以名称为@vendor@overlay@framework@framework-res.apk@idmap的形式保存在目录/data/resource-cache/下。

关于Android系统的资源覆盖（Overlay）机制，可以参考frameworks/base/libs/utils目录下的READ文件。

这一步执行完成之后，回到前面的Step 3中，即ActivityThread类的成员函数getTopLevelResources中，接下来它就会调用前面所创建的Java层的AssetManager对象的成员函数addAssetPath来添加指定的应用程序资源文件路径。

### Step 8. AssetManager.addAssetPath

```java
public final class AssetManager {  
    ......  

    /** 
     * Add an additional set of assets to the asset manager.  This can be 
     * either a directory or ZIP file.  Not for use by applications.  Returns 
     * the cookie of the added asset, or 0 on failure. 
     * {@hide} 
     */  
    public native final int addAssetPath(String path);  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/content/res/AssetManager.java中。

AssetManager类的成员函数addAssetPath是一个JNI方法，它是由C++层的函数android_content_AssetManager_addAssetPath来实现的，如下所示：

```c
static jint android_content_AssetManager_addAssetPath(JNIEnv* env, jobject clazz,  
                                               jstring path)  
{  
    ......  

    AssetManager* am = assetManagerForJavaObject(env, clazz);  
    ......  

    const char* path8 = env->GetStringUTFChars(path, NULL);  

    void* cookie;  
    bool res = am->addAssetPath(String8(path8), &cookie);  

    env->ReleaseStringUTFChars(path, path8);  

    return (res) ? (jint)cookie : 0;  
}  
```
这个函数定义在文件frameworks/base/core/jni/android_util_AssetManager.cpp中。

参数clazz指向的是Java层的一个AssetManager对象，函数android_content_AssetManager_addAssetPath首先调用另外一个函数assetManagerForJavaObject来将它的成员函数mObject转换为一个C++层的AssetManager对象。有了这个C++层的AssetManager对象之后，就可以调用它的成员函数addAssetPath来将参数path所描述的一个Apk文件路径添加到它里面去了，这个过程可以参考前面的Step 7。

这一步执行完成之后，回到前面的Step 3中，即ActivityThread类的成员函数getTopLevelResources中，接下来就会根据前面所创建的Java层的AssetManager对象来创建一个Resources对象。

### Step 9. new Resources

```java
public class Resources {  
    ......  

    /*package*/ final AssetManager mAssets;  
    ......  

    public Resources(AssetManager assets, DisplayMetrics metrics,  
    Configuration config, CompatibilityInfo compInfo) {  
mAssets = assets;  
......  

updateConfiguration(config, metrics);  
assets.ensureStringBlocks();  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/content/res/Resources.java中。

Resources类的构造函数首先将参数assets所指向的一个AssetManager对象保存在成员变量mAssets中，以便以后可以通过它来访问应用程序的资源，接下来调用另外一个成员函数updateConfiguration来设置设备配置信息，最后调用参数assets所指向的一个AssetManager对象的成员函数ensureStringBlocks来创建字符串资源池。

接下来，我们就首先分析Resources类的成员函数updateConfiguration的实现，接着再分析AssetManager类的成员函数ensureStringBlocks的实现。

### Step 10. Resources.updateConfiguration

```java
public class Resources {  
    ......  

    private final Configuration mConfiguration = new Configuration();  
    ......  

    public void updateConfiguration(Configuration config,  
    DisplayMetrics metrics) {  
synchronized (mTmpValue) {  
    int configChanges = 0xfffffff;  
    if (config != null) {  
        configChanges = mConfiguration.updateFrom(config);  
    }  
    if (mConfiguration.locale == null) {  
        mConfiguration.locale = Locale.getDefault();  
    }  
    if (metrics != null) {  
        mMetrics.setTo(metrics);  
        mMetrics.updateMetrics(mCompatibilityInfo,  
                mConfiguration.orientation, mConfiguration.screenLayout);  
    }  
    mMetrics.scaledDensity = mMetrics.density * mConfiguration.fontScale;  

    String locale = null;  
    if (mConfiguration.locale != null) {  
        locale = mConfiguration.locale.getLanguage();  
        if (mConfiguration.locale.getCountry() != null) {  
            locale += "-" + mConfiguration.locale.getCountry();  
        }  
    }  
    int width, height;  
    if (mMetrics.widthPixels >= mMetrics.heightPixels) {  
        width = mMetrics.widthPixels;  
        height = mMetrics.heightPixels;  
    } else {  
        //noinspection SuspiciousNameCombination  
        width = mMetrics.heightPixels;  
        //noinspection SuspiciousNameCombination  
        height = mMetrics.widthPixels;  
    }  
    int keyboardHidden = mConfiguration.keyboardHidden;  
    if (keyboardHidden == Configuration.KEYBOARDHIDDEN_NO  
            && mConfiguration.hardKeyboardHidden  
                    == Configuration.HARDKEYBOARDHIDDEN_YES) {  
        keyboardHidden = Configuration.KEYBOARDHIDDEN_SOFT;  
    }  
    mAssets.setConfiguration(mConfiguration.mcc, mConfiguration.mnc,  
            locale, mConfiguration.orientation,  
            mConfiguration.touchscreen,  
            (int)(mMetrics.density*160), mConfiguration.keyboard,  
            keyboardHidden, mConfiguration.navigation, width, height,  
            mConfiguration.screenLayout, mConfiguration.uiMode, sSdkVersion);  

    ......  
}  
 
......   
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/content/res/Resources.java中。

Resources类的成员变量mConfiguration指向的是一个Configuration对象，用来描述设备当前的配置信息，这些配置信息对应的就是在前面[Android资源管理框架（Asset Manager）简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/8738877)一文中提到18个资源维度。

Resources类的成员函数updateConfiguration首先是根据参数config和metrics来更新设备的当前配置信息，例如，屏幕大小和密码、国家地区和语言、键盘配置情况等等，接着再调用成员变量mAssets所指向的一个Java层的AssetManager对象的成员函数setConfiguration来将这些配置信息设置到与之关联的C++层的AssetManager对象中去。

接下来，我们就继续分析AssetManager类的成员函数setConfiguration的实现，以便可以了解设备配置信息的设置过程。

### Step 11. AssetManager.setConfiguration

```java
public final class AssetManager {  
    ......  

    /** 
     * Change the configuation used when retrieving resources.  Not for use by 
     * applications. 
     * {@hide} 
     */  
    public native final void setConfiguration(int mcc, int mnc, String locale,  
    int orientation, int touchscreen, int density, int keyboard,  
    int keyboardHidden, int navigation, int screenWidth, int screenHeight,  
    int screenLayout, int uiMode, int majorVersion);  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/content/res/AssetManager.java中。

AssetManager类的成员函数setConfiguration是一个JNI方法，它是由C++层的函数android_content_AssetManager_setConfiguration来实现的，如下所示：

```c
static void android_content_AssetManager_setConfiguration(JNIEnv* env, jobject clazz,  
                                                  jint mcc, jint mnc,  
                                                  jstring locale, jint orientation,  
                                                  jint touchscreen, jint density,  
                                                  jint keyboard, jint keyboardHidden,  
                                                  jint navigation,  
                                                  jint screenWidth, jint screenHeight,  
                                                  jint screenLayout, jint uiMode,  
                                                  jint sdkVersion)  
{  
    AssetManager* am = assetManagerForJavaObject(env, clazz);  
    if (am == NULL) {  
return;  
    }  

    ResTable_config config;  
    memset(&config, 0, sizeof(config));  

    const char* locale8 = locale != NULL ? env->GetStringUTFChars(locale, NULL) : NULL;  

    config.mcc = (uint16_t)mcc;  
    config.mnc = (uint16_t)mnc;  
    config.orientation = (uint8_t)orientation;  
    config.touchscreen = (uint8_t)touchscreen;  
    config.density = (uint16_t)density;  
    config.keyboard = (uint8_t)keyboard;  
    config.inputFlags = (uint8_t)keyboardHidden;  
    config.navigation = (uint8_t)navigation;  
    config.screenWidth = (uint16_t)screenWidth;  
    config.screenHeight = (uint16_t)screenHeight;  
    config.screenLayout = (uint8_t)screenLayout;  
    config.uiMode = (uint8_t)uiMode;  
    config.sdkVersion = (uint16_t)sdkVersion;  
    config.minorVersion = 0;  
    am->setConfiguration(config, locale8);  

    if (locale != NULL) env->ReleaseStringUTFChars(locale, locale8);  
}  
```
这个函数定义在文件frameworks/base/core/jni/android_util_AssetManager.cpp中。

参数clazz指向的是一个Java层的AssetManager对象，函数android_content_AssetManager_setConfiguration首先调用另外一个函数assetManagerForJavaObject将它的成员变量mObject转换为一个C++层的AssetManager对象。

函数android_content_AssetManager_setConfiguration接下来再根据其它参数来创建一个ResTable_config对象，这个ResTable_config对象就用来描述设备的当前配置信息。

函数android_content_AssetManager_setConfiguration最后调用前面获得C++层的AssetManager对象的成员函数setConfiguration来将前面创建的ResTable_config对象设置到它内部去，以便C++层的AssetManager对象可以根据设备的当前配置信息来找到最合适的资源。

### Step 12. AssetManager.setConfiguration

```c
void AssetManager::setConfiguration(const ResTable_config& config, const char* locale)  
{  
    AutoMutex _l(mLock);  
    *mConfig = config;  
    if (locale) {  
setLocaleLocked(locale);  
    } else if (config.language[0] != 0) {  
char spec[9];  
spec[0] = config.language[0];  
spec[1] = config.language[1];  
if (config.country[0] != 0) {  
    spec[2] = '_';  
    spec[3] = config.country[0];  
    spec[4] = config.country[1];  
    spec[5] = 0;  
} else {  
    spec[3] = 0;  
}  
setLocaleLocked(spec);  
    } else {  
updateResourceParamsLocked();  
    }  
}  
```
这个函数定义在文件frameworks/base/libs/utils/AssetManager.cpp中。

AssetManager类的成员变量mConfig指向的是一个ResTable_config对象，用来描述设备的当前配置信息，AssetManager类的成员函数setConfiguration首先将参数config所描述的设备配置信息拷贝到它里面去。

如果参数local的值不等于NULL，那么它指向的字符串就是用来描述设备的国家、地区和语言信息的，这时候AssetManager类的成员函数setConfiguration就会调用另外一个成员函数setLocalLocked来将它们设置到AssetManager类的另外一个成员变量mLocale中去。

如果参数local的值等于NULL，并且参数config指向的一个ResTable_config对象包含了设备的国家、地区和语言信息，那么AssetManager类的成员函数setConfiguration同样会调用另外一个成员函数setLocalLocked来将它们设置到AssetManager类的另外一个成员变量mLocale中去。

如果参数local的值等于NULL，并且参数config指向的一个ResTable_config对象没有包含设备的国家、地区和语言信息，那么就说明设备的国家、地区和语言等信息不需要更新，这时候AssetManager类的成员函数setConfiguration就会直接调用另外一个成员函数updateResourceParamsLocked来更新资源表中的设备配置信息。

注意，AssetManager类的成员函数setLocalLocked来更新了成员变量mLocale的内容之后，同样会调用另外一个成员函数updateResourceParamsLocked来更新资源表中的设备配置信息。

AssetManager类的成员函数updateResourceParamsLocked的实现如下所示：

```c
void AssetManager::updateResourceParamsLocked() const  
{  
    ResTable* res = mResources;  
    if (!res) {  
return;  
    }  

    size_t llen = mLocale ? strlen(mLocale) : 0;  
    mConfig->language[0] = 0;  
    mConfig->language[1] = 0;  
    mConfig->country[0] = 0;  
    mConfig->country[1] = 0;  
    if (llen >= 2) {  
mConfig->language[0] = mLocale[0];  
mConfig->language[1] = mLocale[1];  
    }  
    if (llen >= 5) {  
mConfig->country[0] = mLocale[3];  
mConfig->country[1] = mLocale[4];  
    }  
    mConfig->size = sizeof(*mConfig);  

    res->setParameters(mConfig);  
}  
```
这个函数定义在文件frameworks/base/libs/utils/AssetManager.cpp中。

AssetManager类的成员变量mResources指向的是一个ResTable对象，这个ResTable对象描述的就是一个资源索引表。Android应用程序的资源索引表的格式以及生成过程可以参考前面[Android应用程序资源的编译和打包过程分析](http://blog.csdn.net/luoshengyang/article/details/8744683)一文。

AssetManager类的成员函数updateResourceParamsLocked首先是将成员变量mLocale所描述的国家、地区和语言信息更新到另外一个成员变量mConfig中去，接着再将成员变量mConfig所包含的设备配置信息设置到成员变量mResources所描述的一个资源索引表中去，这是通过调用成员变量mResources所指向的一个ResTable对象的成员函数setParameters来实现的。

这一步执行完成之后，返回到前面的Step 9中，即Resources类的构造函数，接下来它就会调用AssetManager类的成员函数ensureStringBlocks来创建字符串资源池。

### Step 13. AssetManager.ensureStringBlocks

```java
public final class AssetManager {  
    ......  

    private StringBlock mStringBlocks[] = null;  
    ......  

    /*package*/ final void ensureStringBlocks() {  
if (mStringBlocks == null) {  
    synchronized (this) {  
        if (mStringBlocks == null) {  
            makeStringBlocks(true);  
        }  
    }  
}  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/content/res/AssetManager.java中。

AssetManager类的成员变量mStringBlocks指向的是一个StringBlock数组，其中，每一个StringBlock对象都是用来描述一个字符串资源池。从前面[Android应用程序资源的编译和打包过程分析](http://blog.csdn.net/luoshengyang/article/details/8744683)一文可以知道，每一个资源表都包含有一个资源项值字符串资源池，AssetManager类的成员变量mStringBlocks就是用来保存所有的资源表中的资源项值字符串资源池的。

AssetManager类的成员函数ensureStringBlocks首先检查成员变量mStringBlocks的值是否等于null。如果等于null的话，那么就说明当前应用程序使用的资源表中的资源项值字符串资源池还没有读取出来，这时候就会调用另外一个成员函数makeStringBlocks来进行读取。

### Step 14. AssetManager.makeStringBlocks

```java
public final class AssetManager {  
    ......  

    private final void makeStringBlocks(boolean copyFromSystem) {  
final int sysNum = copyFromSystem ? sSystem.mStringBlocks.length : 0;  
final int num = getStringBlockCount();  
mStringBlocks = new StringBlock[num];  
......  
for (int i=0; i<num; i++) {  
    if (i < sysNum) {  
        mStringBlocks[i] = sSystem.mStringBlocks[i];  
    } else {  
        mStringBlocks[i] = new StringBlock(getNativeStringBlock(i), true);  
    }  
}  
    }  

    ......  

    private native final int getStringBlockCount();  
    private native final int getNativeStringBlock(int block);  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/content/res/AssetManager.java中。

参数copyFromSystem表示是否要将系统资源表里面的资源项值字符串资源池也一起拷贝到成员变量mStringBlokcs所描述的一个数组中去。如果它的值等于true的时候，那么AssetManager就会首先获得makeStringBlocks首先获得系统资源表的个数sysNum，接着再获得总的资源表个数num，这是通过调用JNI方法getStringBlockCount来实现的。注意，总的资源表个数num是包含了系统资源表的个数sysNum的。

从前面的Step 4可以知道，用来访问系统资源包的AssetManager对象就保存在AssetManager类的静态成员变量sSystem中，并且这个AssetManager对象是最先被创建以及初始化的。也就是说，当执行到这一步的时候，所有系统资源表的资源项值字符串资源池已经读取出来，它们就保存在AssetManager类的静态成员变量sSystem所描述的一个AssetManager对象的成员变量mStringBlocks中，因此，只将它们拷贝到当前正在处理的AssetManager对象的成员变量mStringBlokcs的前sysNum个位置上去就可以了。

最后，AssetManager类的成员函数makeStringBlocks就调用另外一个JNI方法getNativeStringBlock来读取剩余的其它资源表的资源项值字符串资源池，并且分别将它们封装在一个StringBlock对象保存在成员变量mStringBlokcs所描述的一个数组中。

AssetManager类的JNI方法getNativeStringBlock实际上就是将每一个资源包里面的resources.arsc文件的资源项值字符串资源池数据块读取出来，并且封装在一个C++层的StringPool对象中，然后AssetManager类的成员函数makeStringBlocks再将该StringPool对象封装成一个Java层的StringBlock中。关于资源表中的资源项值字符串资源池的更多信息，可以参考前面[Android应用程序资源的编译和打包过程分析](http://blog.csdn.net/luoshengyang/article/details/8744683)一文。

至此，我们就分析完成Android应用程序资源管理器的创建的初始化过程了，主要就是创建和初始化用来访问应用程序资源的AssetManager对象和Resources对象，其中，初始化操作包括设置AssetManager对象的资源文件路径以及设备配置信息等。有了这两个初始化的AssetManager对象和Resources对象之后，在接下来的一篇文章中，我们就可以继续分析应用程序资源的查找过程了，敬请关注！