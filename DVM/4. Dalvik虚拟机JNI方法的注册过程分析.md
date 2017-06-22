在前面一文中，我们分析了Dalvik虚拟机的运行过程。从中可以知道，Dalvik虚拟机在调用一个成员函数的时候，如果发现该成员函数是一个JNI方法，那么就会直接跳到它的地址去执行。也就是说，JNI方法是直接在本地[操作系统](http://lib.csdn.net/base/operatingsystem)上执行的，而不是由Dalvik虚拟机解释器执行。由此也可看出，JNI方法是[Android](http://lib.csdn.net/base/android)应用程序与本地操作系统直接进行通信的一个手段。在本文中，我们就详细分析JNI方法的注册过程。

**老罗的新浪微博：http://weibo.com/shengyangluo，欢迎关注！**

《[android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

在Android系统中，JNI方法是以C/C++语言来实现的，然后编译在一个so文件里面。这个JNI方法在能够被调用之前，首先要加载到当前应用程序进程的地址空间来，如下所示：

```java
package shy.luo.jni;  

public class ClassWithJni {  
    ......  
      
    static {  
System.loadLibrary("nanosleep");    
    }  
      
    ......  
      
    private native int nanosleep(long seconds, long nanoseconds);  
      
    ......  
}  
```
上述代码假设ClassWithJni类有一个JNI方法nanosleep，它实现在一个名称为libnanosleep.so的文件中，因此，在该JNI方法能够被调用之前，我们首先要将它加载到当前应用程序进程来，这是通过调用System类的静态成员函数loadLibrary来实现的。

JNI方法nanosleep的实现如下所示：

```c
#include "jni.h"  
#include "JNIHelp.h"  

#include <time.h>  

static jint shy_luo_jni_ClassWithJni_nanosleep(JNIEnv* env, jobject clazz, jlong seconds, jlong nanoseconds)   
{  
    struct timespec req;  
    req.tv_sec  = seconds;  
    req.tv_nsec = nanoseconds;  
      
    return nanosleep(&req, NULL);  
}  

static const JNINativeMethod method_table[] = {  
    {"nanosleep", "(JJ)I", (void*)shy_luo_jni_ClassWithJni_nanosleep},  
};  

extern "C" jint JNI_OnLoad(JavaVM* vm, void* reserved)   
{  
      JNIEnv* env = NULL;  
    jint result = -1;  

    if (vm->GetEnv((void**) &env, JNI_VERSION_1_4) != JNI_OK) {  
return result;  
    }  

    jniRegisterNativeMethods(env, "shy/luo/jni/ClassWithJni", method_table, NELEM(method_table));  
      
    return JNI_VERSION_1_4;  
}  
```
假设上述函数经过编译之后，就位于一个名称为libnanosleep.so的文件。

当libnanosleep.so文件被加载的时候，函数JNI_OnLoad就会被调用。在函数JNI_OnLoad中，参数vm描述的是当前进程中的Dalvik虚拟机，通过调用它的成员函数GetEnv就可以获得一个JNIEnv对象。有了这个JNIEnv对象之后，我们就可以调用另外一个函数jniRegisterNativeMethods来向当前进程中的Dalvik虚拟机注册一个JNI方法shy_luo_jni_ClassWithJni_nanosleep。这个JNI方法即为shy.luo.jni.ClassWithJni类的成员函数nanasleep的实现。

JNI方法shy_luo_jni_ClassWithJni_nanosleep要做的事情实际上就是通过系统调用nanosleep来使得当前进程进入睡眠状态，直至seconds秒nanoseconds纳秒之后再唤醒。使用系统调用nanosleep来使得当前进程进入睡眠状态的好处它的时间精度可以达到纳秒级，但是这个系统调用有两个地方是需要注意的：

如果进程在睡眠的过程中接收到信号，那么它就会提前被唤醒，这时候系统调用nanosleep的返回值为-1，并且错误代码errno被设置为EINTR。

如果CPU的时钟中断精度达不到纳秒级别，那么nanosleep的睡眠精度也达不到纳秒级，也就是说，当前进程不一定能在指定的纳秒之后被唤醒，会有一定的延时。

不过，JNI方法shy_luo_jni_ClassWithJni_nanosleep的实现不是我们的重点，我们的重点是分析它注册到Dalvik虚拟机的过程。

前面提到，JNI方法shy_luo_jni_ClassWithJni_nanosleep是libnanosleep.so文件加载的时候被注册到Dalvik虚拟机的，因此，我们就从libnanosleep.so文件的加载开始，分析JNI方法shy_luo_jni_ClassWithJni_nanosleep注册到Dalvik虚拟机的过程，也就是从System类的静态成员函数loadLibrary开始分析一个JNI方法注册到Dalvik虚拟机的过程，如图1所示：

![img](http://img.blog.csdn.net/20130516013514637)

图1 JNI方法注册到Dalvik虚拟机的过程

这个过程可以分为12个步骤，接下来我们就详细分析每一个步骤。

### Step 1. System.loadLibrary

```java
public final class System {  
    ......  
      
    public static void loadLibrary(String libName) {  
SecurityManager smngr = System.getSecurityManager();  
if (smngr != null) {  
    smngr.checkLink(libName);  
}  
Runtime.getRuntime().loadLibrary(libName, VMStack.getCallingClassLoader());  
    }  

    ......  
}  
```
这个函数定义在文件libcore/luni/src/main/[Java](http://lib.csdn.net/base/java)/java/lang/System.java中。

System类的成员函数loadLibrary首先调用SecurityManager类的成员函数checkLink来进行安全检查，即检查名称为libName的so文件是否允许加载。注意，这是Java的安全代码检查机制，而不是Android系统的安全检查机制，而且Android系统没有使用它来进行安全检查。因此，这个检查总是能通过的。

System类的成员函数loadLibrary接下来就再通过运行时类Runtime的成员函数loadLibrary来加载名称为libName的so文件，接下来我们就继续分析它的实现。

### Step 2. Runtime.loadLibrary

```java
public class Runtime {  
    ......  

    void loadLibrary(String libraryName, ClassLoader loader) {  
if (loader != null) {  
    String filename = loader.findLibrary(libraryName);  
    if (filename == null) {  
        throw new UnsatisfiedLinkError("Couldn't load " + libraryName + ": " +  
                "findLibrary returned null");  
    }  
    String error = nativeLoad(filename, loader);  
    if (error != null) {  
        throw new UnsatisfiedLinkError(error);  
    }  
    return;  
}  

String filename = System.mapLibraryName(libraryName);  
List<String> candidates = new ArrayList<String>();  
String lastError = null;  
for (String directory : mLibPaths) {  
    String candidate = directory + filename;  
    candidates.add(candidate);  
    if (new File(candidate).exists()) {  
        String error = nativeLoad(candidate, loader);  
        if (error == null) {  
            return; // We successfully loaded the library. Job done.  
        }  
        lastError = error;  
    }  
}  

if (lastError != null) {  
    throw new UnsatisfiedLinkError(lastError);  
}  
throw new UnsatisfiedLinkError("Library " + libraryName + " not found; tried " + candidates);  
    }  

    ......  
}  
```
这个函数定义在文件libcore/luni/src/main/java/java/lang/Runtime.java中。

在Runtime类的成员函数loadLibrary中，参数libraryName表示要加载的so文件，而参数loader表示与要加载的so文件所关联的类的一个类加载器。例如，在我们这个情景中，libraryName等于“nanosleep”，与它所关联的类为shy.luo.jni.ClassWithJni。每一类有一个关联的类加载器，用来负责加载该类。在Dalvik虚拟机中，类加载器除了知道它要加载的类所在的文件路径之外，还知道该类所属的APK用来保存so文件的路径。因此，给定一个so文件名称，一个类加载器可以判断它是否存在自己的so文件目录中。

参数libraryName只是描述要加载的so文件的部分名称，它的完整名称需要根据本地操作系统的特证来确定。由于目前Android系统都是属于[Linux](http://lib.csdn.net/base/linux)系统，而在[linux](http://lib.csdn.net/base/linux)系统中，so文件的命名规范通常就是lib<name>.so的形式，因此，在我们这个情景中，名称为“nanosleep”的so文件的完整名称就为“libnanosleep.so”，这是通过调用System类的静态成员函数mapLibraryName来获得的。

上面所获得的libnanosleep.so文件的名称仍然还不够完整，因为它没有包含绝对路径。在这种情况下，我们是无法将它加载到Dalvik虚拟机中去的。当参数loader的值不等于null的时候，Runtime类的成员函数loadLibrary就会调用它的成员函数findLibrary来它的so文件目录中寻找是否有一外名称为“libnanosleep.so”。如果存在的话，那么就会返回该libnanosleep.so文件的绝对路径。有了libnanosleep.so文件的绝对路径之后，就可以调用Runtime类的另外一个成员函数nativeLoad来将它加载到当前进程的Dalvik虚拟机中。注意，将参数libraryName转换为lib<name>.so的完整形式，以及获得该so文件的绝对路径，都是由参数loader所描述的一个类加载器的成员函数findLibrary来完成的。

另一方面，如果参数loader的值等于null，那么就表示当前要加载的so文件要在系统范围的so文件目录查找。这些系统范围的so文件目录保存在Runtime类的成员变量mLibPaths所描述的一个String数组中。通过依次检查这些目录是否存在与参数libraryName对应的so文件，就可以确定参数libraryName所指定加载的so文件是否是一个合法的so文件。如果合法的话，那么同样会调用Runtime类的另外一个成员函数nativeLoad来将它加载到当前进程的Dalvik虚拟机中。注意，这里在检查参数libraryName所表示的so文件是否存在于系统范围的so文件目录之前，同样要将它转换为lib<name>.so的形式，这同样也是通过调用System类的静态成员函数mapLibraryName来完成的。

如果最后无法在指定的APK或者系统范围的so文件目录中找到由参数libraryName所描述的so文件，或者找到了该so文件，但是在加载该so文件的过程中出现错误，那么Runtime类的成员函数loadLibrary都会抛出一个类型为UnsatisfiedLinkError的异常。

由于加载参数libraryName所描述的so文件是由Runtime类的成员函数nativeLoad来实现的，因此，接下来我们继续分析它的实现。

### Step 3. Runtime.nativeLoad

```java
public class Runtime {  
    ......  

    private static native String nativeLoad(String filename, ClassLoader loader);  

    ......  
}  
```
这个函数定义在文件libcore/luni/src/main/java/java/lang/Runtime.java中。

Runtime类的成员函数nativeLoad是一个JNI方法。由于该JNI方法是属于Java核心类Runtime的，也就是说，它在Dalvik虚拟机启动的时候就已经在内部注册过了，因此，这时候我们可以直接调用它注册其它的JNI方法，也就是so文件filename里面所指定的JNI方法。Dalvik虚拟机在启动过程中注册Java核心类的操作，具体可以参考前面[Dalvik虚拟机的启动过程分析](http://blog.csdn.net/luoshengyang/article/details/8885792)一文。

Runtime类的成员函数nativeLoad在C++层对应的函数为Dalvik_java_lang_Runtime_nativeLoad，如下所示：

```c
static void Dalvik_java_lang_Runtime_nativeLoad(const u4* args,  
    JValue* pResult)  
{  
    StringObject* fileNameObj = (StringObject*) args[0];  
    Object* classLoader = (Object*) args[1];  
    char* fileName = NULL;  
    StringObject* result = NULL;  
    char* reason = NULL;  
    bool success;  

    assert(fileNameObj != NULL);  
    fileName = dvmCreateCstrFromString(fileNameObj);  

    success = dvmLoadNativeCode(fileName, classLoader, &reason);  
    if (!success) {  
const char* msg = (reason != NULL) ? reason : "unknown failure";  
result = dvmCreateStringFromCstr(msg);  
dvmReleaseTrackedAlloc((Object*) result, NULL);  
    }  

    free(reason);  
    free(fileName);  
    RETURN_PTR(result);  
}  
```
这个函数定义在文件dalvik/vm/native/java_lang_Runtime.c中。

参数args[0]保存的是一个Java层的String对象，这个String对象描述的就是要加载的so文件，函数Dalvik_java_lang_Runtime_nativeLoad首先是调用函数dvmCreateCstrFromString来将它转换成一个C++层的字符串fileName，然后再调用函数dvmLoadNativeCode来执行加载so文件的操作。

接下来，我们就继续分析函数dvmLoadNativeCode的实现，以便可以了解一个so文件的加载过程。

### Step 4. dvmLoadNativeCode

```c
bool dvmLoadNativeCode(const char* pathName, Object* classLoader,  
char** detail)  
{  
    SharedLib* pEntry;  
    void* handle;  
    ......  

    pEntry = findSharedLibEntry(pathName);  
    if (pEntry != NULL) {  
if (pEntry->classLoader != classLoader) {  
    ......  
    return false;  
}  
......  

if (!checkOnLoadResult(pEntry))  
    return false;  
return true;  
    }  
    ......  

    handle = dlopen(pathName, RTLD_LAZY);  
    ......  

    /* create a new entry */  
    SharedLib* pNewEntry;  
    pNewEntry = (SharedLib*) calloc(1, sizeof(SharedLib));  
    pNewEntry->pathName = strdup(pathName);  
    pNewEntry->handle = handle;  
    pNewEntry->classLoader = classLoader;  
    ......  

    /* try to add it to the list */  
    SharedLib* pActualEntry = addSharedLibEntry(pNewEntry);  

    if (pNewEntry != pActualEntry) {  
......  
freeSharedLibEntry(pNewEntry);  
return checkOnLoadResult(pActualEntry);  
    } else {  
......  

bool result = true;  
void* vonLoad;  
int version;  

vonLoad = dlsym(handle, "JNI_OnLoad");  
if (vonLoad == NULL) {  
    LOGD("No JNI_OnLoad found in %s %p, skipping init\n",  
        pathName, classLoader);  
} else {  
    ......  

    OnLoadFunc func = vonLoad;  
    ......  

    version = (*func)(gDvm.vmList, NULL);  
    ......  

    if (version != JNI_VERSION_1_2 && version != JNI_VERSION_1_4 &&  
        version != JNI_VERSION_1_6)  
    {  
        .......  
        result = false;  
    } else {  
        LOGV("+++ finished JNI_OnLoad %s\n", pathName);  
    }  

}  

......  

if (result)  
    pNewEntry->onLoadResult = kOnLoadOkay;  
else  
    pNewEntry->onLoadResult = kOnLoadFailed;  

......  

return result;  
    }  
}   
```
这个函数定义在文件dalvik/vm/Native.c中。

函数dvmLoadNativeCode首先是检查参数pathName所指定的so文件是否已经加载过了，这是通过调用函数findSharedLibEntry来实现的。如果已经加载过，那么就可以获得一个SharedLib对象pEntry。这个SharedLib对象pEntry描述了有关参数pathName所指定的so文件的加载信息，例如，上次用来加载它的类加载器和上次的加载结果。如果上次用来加载它的类加载器不等于当前所使用的类加载器，或者上次没有加载成功，那么函数dvmLoadNativeCode就回直接返回false给调用者，表示不能在当前进程中加载参数pathName所描述的so文件。

我们假设参数pathName所指定的so文件还没有被加载过，这时候函数dvmLoadNativeCode就会先调用dlopen来在当前进程中加载它，并且将获得的句柄保存在变量handle中，接着再创建一个SharedLib对象pNewEntry来描述它的加载信息。这个SharedLib对象pNewEntry还会通过函数addSharedLibEntry被缓存起来，以便可以知道当前进程都加载了哪些so文件。

注意，在调用函数addSharedLibEntry来缓存新创建的SharedLib对象pNewEntry的时候，如果得到的返回值pActualEntry指向的不是SharedLib对象pNewEntry，那么就表示另外一个线程也正在加载参数pathName所指定的so文件，并且比当前线程提前加载完成。在这种情况下，函数addSharedLibEntry就什么也不用做而直接返回了。否则的话，函数addSharedLibEntry就要继续负责调用前面所加载的so文件中的一个指定的函数来注册它里面的JNI方法。

这个指定的函数的名称为“JNI_OnLoad”，也就是说，每一个用来实现JNI方法的so文件都应该定义有一个名称为“JNI_OnLoad”的函数，并且这个函数的原型为：

```c
jint JNI_OnLoad(JavaVM* vm, void* reserved);  
```
函数dvmLoadNativeCode通过调用函数dlsym就可以获得在前面加载的so中名称为“JNI_OnLoad”的函数的地址，最终保存在函数指针func中。有了这个函数指针之后，我们就可以直接调用它来执行注册JNI方法的操作了。注意，在调用该JNI_OnLoad函数时，第一个要传递进行的参数是一个JavaVM对象，这个JavaVM对象描述的是在当前进程中运行的Dalvik虚拟机，第二个要传递的参数可以设置为NULL，这是保留给以后使用的。

从前面[Dalvik虚拟机的启动过程分析](http://blog.csdn.net/luoshengyang/article/details/8885792)一文可以知道，在当前进程所运行的Dalvik虚拟机实例是通过全局变量gDvm所描述的一个DvmGlobals结构体的成员变量vmList来描述的，因此，我们就可以将它传递在前面加载的so中名称中定义的JNI_OnLoad函数。注意，定义在该so文件中的JNI_OnLoad函数一旦执行成功，它的返回值就必须等于JNI_VERSION_1_2、JNI_VERSION_1_4或者JNI_VERSION_1_6，用来表示所注册的JNI方法的版本。

最后， 函数dvmLoadNativeCode根据上述的JNI_OnLoad函数的执行成功与否，将前面所创建的一个SharedLib对象pNewEntry的成员变量onLoadResult设置为kOnLoadOkay或者kOnLoadFailed，这样就可以记录参数pathName所指定的so文件是否是加载成功的，也就是它是否成功地注册了其内部的JNI方法。

在我们这个情景中，参数pathName所指定的so文件为libnanosleep.so，接下来我们就继续分析它的函数JNI_OnLoad的实现，以便可以发解定义在它里面的JNI方法的注册过程。

### Step 5. JNI_OnLoad

定义在libnanosleep.so文件中的函数JNI_OnLoad的实现可以参考文章开始的部分。从它的实现可以知道，它所注册的JNI方法shy_luo_jni_ClassWithJni_nanosleep是与shy.luo.jni.ClassWithJni类的成员函数nanosleep对应的，并且是通过调用函数jniRegisterNativeMethods来实现的。因此，接下来我们就继续分析函数jniRegisterNativeMethods的实现。

### Step 6. jniRegisterNativeMethods

```c
int jniRegisterNativeMethods(JNIEnv* env, const char* className,  
    const JNINativeMethod* gMethods, int numMethods)  
{  
    jclass clazz;  

    LOGV("Registering %s natives\n", className);  
    clazz = (*env)->FindClass(env, className);  
    if (clazz == NULL) {  
LOGE("Native registration unable to find class '%s'\n", className);  
return -1;  
    }  

    int result = 0;  
    if ((*env)->RegisterNatives(env, clazz, gMethods, numMethods) < 0) {  
LOGE("RegisterNatives failed for '%s'\n", className);  
result = -1;  
    }  

    (*env)->DeleteLocalRef(env, clazz);  
    return result;  
}  
```
这个函数定义在文件dalvik/libnativehelper/JNIHelp.c中。

参数env所指向的一个JNIEnv结构体，通过调用这个JNIEnv结构体可以获得参数className所描述的一个类。这个类就是要注册JNI的类，而它所要注册的JNI就是由参数gMethods来描述的。

注册参数gMethods所描述的JNI方法是通过调用env所指向的一个JNIEnv结构体的成员函数RegisterNatives来实现的，因此，接下来我们就继续分析它的实现。

### Step 7. JNIEnv.RegisterNatives

```c
typedef _JNIEnv JNIEnv;  
......  

struct _JNIEnv {  
    /* do not rename this; it does not seem to be entirely opaque */  
    const struct JNINativeInterface* functions;  
    ......  

    jint RegisterNatives(jclass clazz, const JNINativeMethod* methods,  
jint nMethods)  
    { return functions->RegisterNatives(this, clazz, methods, nMethods); }  

    ......  
}  
```
这个函数定义在文件dalvik/libnativehelper/include/nativehelper/jni.h中。

从前面[Dalvik虚拟机的运行过程分析](http://blog.csdn.net/luoshengyang/article/details/8914953)一文可以知道，结构体JNIEnv的成员变量functions指向的是一个函数表，这个函数表又包含了一系列的函数指针，指向了在当前进程中运行的Dalvik虚拟机中定义的函数。对于结构体JNIEnv的成员函数RegisterNatives来说，它就是通过调用这个函数表中名称为RegisterNatives的函数指针来注册参数gMethods所描述的JNI方法的。

从前面[Dalvik虚拟机的启动过程分析](http://blog.csdn.net/luoshengyang/article/details/8914953)一文可以知道，上述函数表中名称为RegisterNatives的函数指针指向的是在Dalvik虚拟机内部定义的函数RegisterNatives，因此，接下来我们就继续分析它的实现。

### Step 8. RegisterNatives

```c
static jint RegisterNatives(JNIEnv* env, jclass jclazz,  
    const JNINativeMethod* methods, jint nMethods)  
{  
    JNI_ENTER();  

    ClassObject* clazz = (ClassObject*) dvmDecodeIndirectRef(env, jclazz);  
    jint retval = JNI_OK;  
    int i;  

    ......  

    for (i = 0; i < nMethods; i++) {  
if (!dvmRegisterJNIMethod(clazz, methods[i].name,  
        methods[i].signature, methods[i].fnPtr))  
{  
    retval = JNI_ERR;  
}  
    }  

    JNI_EXIT();  
    return retval;  
}  
```
这个函数定义在文件dalvik/vm/Jni.c中。

参数jclazz描述的是要注册JNI方法的类，而参数methods描述的是要注册的一组JNI方法，这个组JNI方法的个数由参数nMethods来描述。

函数RegisterNatives首先是调用函数dvmDecodeIndirectRef来获得要注册JNI方法的类对象，接着再通过一个for循环来依次调用函数dvmRegisterJNIMethod注册参数methods描述所描述的每一个JNI方法。注意，每一个JNI方法都由名称、签名和地址来描述。

接下来，我们就继续分析函数dvmRegisterJNIMethod的实现。

### Step 9. dvmRegisterJNIMethod

```c
static bool dvmRegisterJNIMethod(ClassObject* clazz, const char* methodName,  
    const char* signature, void* fnPtr)  
{  
    Method* method;  
    bool result = false;  

    if (fnPtr == NULL)  
goto bail;  

    method = dvmFindDirectMethodByDescriptor(clazz, methodName, signature);  
    if (method == NULL)  
method = dvmFindVirtualMethodByDescriptor(clazz, methodName, signature);  
    if (method == NULL) {  
LOGW("ERROR: Unable to find decl for native %s.%s:%s\n",  
    clazz->descriptor, methodName, signature);  
goto bail;  
    }  

    if (!dvmIsNativeMethod(method)) {  
LOGW("Unable to register: not native: %s.%s:%s\n",  
    clazz->descriptor, methodName, signature);  
goto bail;  
    }  

    if (method->nativeFunc != dvmResolveNativeMethod) {  
/* this is allowed, but unusual */  
LOGV("Note: %s.%s:%s was already registered\n",  
    clazz->descriptor, methodName, signature);  
    }  

    dvmUseJNIBridge(method, fnPtr);  

    LOGV("JNI-registered %s.%s:%s\n", clazz->descriptor, methodName,  
signature);  
    result = true;  

bail:  
    return result;  
}  
```
这个函数定义在文件dalvik/vm/Jni.c中。

函数dvmRegisterJNIMethod在注册参数methodName所描述的JNI方法之前，首先会进行一系列的检查，包括：

确保参数clazz所描述的类有一个名称为methodName的成员函数。首先是调用函数dvmFindDirectMethodByDescriptor来检查methodName是否是clazz的一个非虚成员函数，然后再调用函数dvmFindVirtualMethodByDescriptor来检查methodName是否是clazz的一个虚成员函数。

确保类clazz的成员函数methodName确实是声明为JNI方法，即带有native修饰符，这是通过调用函数dvmIsNativeMethod来实现的。

通过了前面的第1个检查之后，就可以获得一个Method对象method，用来描述要注册的JNI方法所对应的Java类成员函数。当一个Method对象method描述的是一个JNI方法的时候，它的成员变量nativeFunc保存的就是该JNI方法的地址，但是在对应的JNI方法注册进来之前，该成员变量的值被统一设置为dvmResolveNativeMethod。因此，当我们调用了一个未注册的JNI方法时，实际上执行的是函数dvmResolveNativeMethod。函数dvmResolveNativeMethod此时会在Dalvik虚拟内部以及当前所有已经加载的共享库中检查是否存在对应的JNI方法。如果不存在，那么它就会抛出一个类型为java.lang.UnsatisfiedLinkError的异常。

注意，一个JNI方法是可以重复注册的，无论如何，函数dvmRegisterJNIMethod都是调用另外一个函数dvmUseJNIBridge来继续执行注册JNI的操作。

### Step 10. dvmUseJNIBridge

```c
/** 
* Returns true if the -Xjnitrace setting implies we should trace 'method'. 
*/  
static bool shouldTrace(Method* method)  
{  
    return gDvm.jniTrace && strstr(method->clazz->descriptor, gDvm.jniTrace);  
}  

/* 
* Point "method->nativeFunc" at the JNI bridge, and overload "method->insns" 
* to point at the actual function. 
*/  
void dvmUseJNIBridge(Method* method, void* func)  
{  
    DalvikBridgeFunc bridge = shouldTrace(method)  
? dvmTraceCallJNIMethod  
: dvmSelectJNIBridge(method);  
    dvmSetNativeFunc(method, bridge, func);  
}  
```
这个函数定义在文件dalvik/vm/Jni.c中。

一个JNI方法并不是直接被调用的，而是通过由Dalvik虚拟机间接地调用，这个用来间接调用JNI方法的函数就称为一个Bridge。这些Bridage函数在真正调用JNI方法之前，会执行一些通用的初始化工作。例如，会将当前线程的状态设置为NATIVE，因为它即将要执行一个Native函数。又如，会为即将要被调用的JNI方法准备好前面两个参数，第一个参数是一个JNIEnv对象，用来描述当前线程的Java环境，通过它可以访问反过来访问Java代码和Java对象，第二个参数是一个jobject对象，用来描述当前正在执行JNI方法的Java对象。

这些Bridage函数实际上仍然不是直接调用地调用JNI方法的，这是因为Dalvik虚拟机是可以运行在各种不同的平台之上，而每一种平台可能都定义有自己的一套函数调用规范，也就是所谓的ABI（Application Binary Interface），这是一个API（Application Programming Interface）不同的概念。ABI是在二进制级别上定义的一套函数调用规范，例如参数是通过寄存器来传递还是堆栈来传递，而API定义是一个应用程序编程接口规范。换句话说，API定义了源代码和库之间的接口，因此同样的代码可以在支持这个API的任何系统中编译 ，而ABI允许编译好的目标代码在使用兼容ABI的系统中无需改动就能运行。 

为了使得运行在不同平台上的Dalvik虚拟机能够以统一的方法来调用JNI方法，这些Bridage函数使用了一个libffi库，它的源代码位于external/libffi目录中。Libffi是一个开源项目，用于高级语言之间的相互调用的处理，它的实现机制可以进一步参考[http://www.sourceware.org/libffi/](http://www.sourceware.org/libffi/)。

回到函数dvmUseJNIBridge中，它主要就是根据Dalvik虚拟机的启动选项来为即将要注册的JNI选择一个合适的Bridge函数。如果我们在Dalvik虚拟机启动的时候，通过-Xjnitrace选项来指定了要跟踪参数method所描述的JNI方法，那么函数dvmUseJNIBridge为该JNI方法选择的Bridge函数就为dvmTraceCallJNIMethod，否则的话，就再通过另外一个函数dvmSelectJNIBridge来进一步选择一个合适的Bridge函数。选择好Bridge函数之后，函数dvmUseJNIBridge最终就调用函数dvmSetNativeFunc来执行真正的JNI方法注册操作。

我们假设参数method所描述的JNI方法没有设置为跟踪，因此，接下来，我们就首先分析函数dvmSelectJNIBridge的实现，接着再分析函数dvmSetNativeFunc的实现。

### Step 11. dvmSelectJNIBridge

```c
/* 
* Returns the appropriate JNI bridge for 'method', also taking into account 
* the -Xcheck:jni setting. 
*/  
static DalvikBridgeFunc dvmSelectJNIBridge(const Method* method)  
{  
    enum {  
kJNIGeneral = 0,  
kJNISync = 1,  
kJNIVirtualNoRef = 2,  
kJNIStaticNoRef = 3,  
    } kind;  
    static const DalvikBridgeFunc stdFunc[] = {  
dvmCallJNIMethod_general,  
dvmCallJNIMethod_synchronized,  
dvmCallJNIMethod_virtualNoRef,  
dvmCallJNIMethod_staticNoRef  
    };  
    static const DalvikBridgeFunc checkFunc[] = {  
dvmCheckCallJNIMethod_general,  
dvmCheckCallJNIMethod_synchronized,  
dvmCheckCallJNIMethod_virtualNoRef,  
dvmCheckCallJNIMethod_staticNoRef  
    };  

    bool hasRefArg = false;  

    if (dvmIsSynchronizedMethod(method)) {  
/* use version with synchronization; calls into general handler */  
kind = kJNISync;  
    } else {  
/* 
 * Do a quick scan through the "shorty" signature to see if the method 
 * takes any reference arguments. 
 */  
const char* cp = method->shorty;  
while (*++cp != '\0') {     /* pre-incr to skip return type */  
    if (*cp == 'L') {  
        /* 'L' used for both object and array references */  
        hasRefArg = true;  
        break;  
    }  
}  

if (hasRefArg) {  
    /* use general handler to slurp up reference args */  
    kind = kJNIGeneral;  
} else {  
    /* virtual methods have a ref in args[0] (not in signature) */  
    if (dvmIsStaticMethod(method))  
        kind = kJNIStaticNoRef;  
    else  
        kind = kJNIVirtualNoRef;  
}  
    }  

    return dvmIsCheckJNIEnabled() ? checkFunc[kind] : stdFunc[kind];  
}  
```
这个函数定义在文件dalvik/vm/Jni.c中。

Dalvik虚拟机提供的Bridge函数主要是分为两类。第一类Bridge函数在调用完成JNI方法之后，会检查该JNI方法的返回结果是否与声明的一致，这是因为一个声明返回String的JNI方法在执行时返回的可能会是一个Byte Array。如果不一致，取决于Dalvik虚拟机的启动选项，它可能会停机。第二类Bridge函数不对JNI方法的返回结果进行上述检查。选择哪一类Bridge函数可以通过-Xcheck:jni选项来决定。不过由于检查一个JNI方法的返回结果是否与声明的一致是很耗时的，因此，我们一般都不会使用第一类Bridge函数。

此外，每一类Bridge函数又分为四个子类：Genernal、Sync、VirtualNoRef和StaticNoRef，它们的选择规则为：

一个JNI方法的参数列表中如果包含有引用类型的参数，那么对应的Bridge函数就是Genernal类型的，即为dvmCallJNIMethod_general或者dvmCheckCallJNIMethod_general。

一个JNI方法如果声明为同步方法，即带有synchronized修饰符，那么对应的Bridge函数就是Sync类型的，即为dvmCallJNIMethod_synchronized或者dvmCheckCallJNIMethod_synchronized。

一个JNI方法的参数列表中如果不包含有引用类型的参数，并且它是一个虚成员函数，那么对应的Bridge函数就是kJNIVirtualNoRef类型的，即为dvmCallJNIMethod_virtualNoRef或者dvmCheckCallJNIMethod_virtualNoRef。

一个JNI方法的参数列表中如果不包含有引用类型的参数，并且它是一个静态成员函数，那么对应的Bridge函数就是StaticNoRef类型的，即为dvmCallJNIMethod_staticNoRef或者dvmCheckCallJNIMethod_staticNoRef。

每一类Bridge函数之所以要划分为上述四个子类，是因为每一个子类的Bridge函数在调用真正的JNI方法之前，所要进行的准备工作是不一样的。例如，Genernal类型的Bridge函数需要为引用类型的参数增加一个本地引用，避免它在JNI方法执行的过程中被回收。又如，Sync类型的Bridge函数在调用JNI方法之前，需要执行同步原始，以避免多线程访问的竞争问题。

这一步执行完成之后，返回到前面的Step 10中，即函数dvmUseJNIBridge中，这时候它就获得了一个Bridge函数，因此，接下来它就可以调用函数dvmSetNativeFunc来执行真正的JNI方法注册操作了。

### Step 12. dvmSetNativeFunc

```c
void dvmSetNativeFunc(Method* method, DalvikBridgeFunc func,  
    const u2* insns)  
{  
    ......  

    if (insns != NULL) {  
/* update both, ensuring that "insns" is observed first */  
method->insns = insns;  
android_atomic_release_store((int32_t) func,  
    (void*) &method->nativeFunc);  
    } else {  
/* only update nativeFunc */  
method->nativeFunc = func;  
    }  

    ......  
}  
```
这个函数定义在文件dalvik/vm/oo/Class.c中。

参数method表示要注册JNI方法的Java类成员函数，参数func表示JNI方法的Bridge函数，参数insns表示要注册的JNI方法的函数地址。

当参数insns的值不等于NULL的时候，函数dvmSetNativeFunc就分别将参数insns和func的值分别保存在参数method所指向的一个Method对象的成员变量insns和nativeFunc中，而当insns的值等于NULL的时候，函数dvmSetNativeFunc就只将参数func的值保存在参数method所指向的一个Method对象成员变量nativeFunc中。

假设在前面的Step 11中选择的Bridge函数为dvmCallJNIMethod_general，并且结合前面[Dalvik虚拟机的运行过程分析](http://blog.csdn.net/luoshengyang/article/details/8914953)一文，我们就可以得到Dalvik虚拟机在运行过程中调用JNI方法的过程：

调用函数dvmCallJNIMethod_general，执行一些必要的准备工作；

函数dvmCallJNIMethod_general再调用函数dvmPlatformInvoke来以统一的方式来调用对应的JNI方法；

函数dvmPlatformInvoke通过libffi库来调用对应的JNI方法，以屏蔽Dalvik虚拟机运行在不同目标平台的细节。

至此，我们就分析完成Dalvik虚拟机JNI方法的注册过程了。这样，我们就打通了Java代码和Native代码之间的道路。实际上，很多Java和Android核心类的功能都是通过本地操作系统提供的系统调用来完成的，例如，Zygote类的成员函数forkAndSpecialize最终是通过Linux系统调用fork来创建一个Android应用程序进程的，又如，Thread类的成员函数start最终是通过pthread线程库函数pthread_create来创建一个Android应用程序线程的。

在接下来的一篇文章中，我们就在Java代码和Native代码打通了的基础上，分析Android应用程序进程和线程与本地操作系统进程的线程的关系，也就是Dalvik虚拟机进程和线程与本地操作系统进程的线程的关系，敬请关注！