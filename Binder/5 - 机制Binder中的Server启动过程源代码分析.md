在前面一篇文章[浅谈Android系统进程间通信（IPC）机制Binder中的Server和Client获得Service Manager接口之路](http://blog.csdn.net/luoshengyang/article/details/6627260)中，介绍了在[Android](http://lib.csdn.net/base/android)系统中Binder进程间通信机制中的Server角色是如何获得Service Manager远程接口的，即defaultServiceManager函数的实现。Server获得了Service Manager远程接口之后，就要把自己的Service添加到Service Manager中去，然后把自己启动起来，等待Client的请求。本文将通过分析源代码了解Server的启动过程是怎么样的。

《[android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

本文通过一个具体的例子来说明Binder机制中Server的启动过程。我们知道，在Android系统中，提供了多媒体播放的功能，这个功能是以服务的形式来提供的。这里，我们就通过分析MediaPlayerService的实现来了解Media Server的启动过程。

首先，看一下MediaPlayerService的类图，以便我们理解下面要描述的内容。

![img](http://hi.csdn.net/attachment/201107/24/0_1311479168os88.gif)

我们将要介绍的主角MediaPlayerService继承于BnMediaPlayerService类，熟悉Binder机制的同学应该知道BnMediaPlayerService是一个Binder Native类，用来处理Client请求的。BnMediaPlayerService继承于BnInterface<IMediaPlayerService>类，BnInterface是一个模板类，它定义在frameworks/base/include/binder/IInterface.h文件中：

```c
template<typename INTERFACE>  
class BnInterface : public INTERFACE, public BBinder  
{  
public:  
    virtual sp<IInterface>      queryLocalInterface(const String16& _descriptor);  
    virtual const String16&     getInterfaceDescriptor() const;  

protected:  
    virtual IBinder*            onAsBinder();  
};  
```
这里可以看出，BnMediaPlayerService实际是继承了IMediaPlayerService和BBinder类。IMediaPlayerService和BBinder类又分别继承了IInterface和IBinder类，IInterface和IBinder类又同时继承了RefBase类。

实际上，BnMediaPlayerService并不是直接接收到Client处发送过来的请求，而是使用了IPCThreadState接收Client处发送过来的请求，而IPCThreadState又借助了ProcessState类来与Binder驱动程序交互。有关IPCThreadState和ProcessState的关系，可以参考上一篇文章[浅谈Android系统进程间通信（IPC）机制Binder中的Server和Client获得Service Manager接口之路](http://blog.csdn.net/luoshengyang/article/details/6627260)，接下来也会有相应的描述。IPCThreadState接收到了Client处的请求后，就会调用BBinder类的transact函数，并传入相关参数，BBinder类的transact函数最终调用BnMediaPlayerService类的onTransact函数，于是，就开始真正地处理Client的请求了。

了解了MediaPlayerService类结构之后，就要开始进入到本文的主题了。

首先，看看MediaPlayerService是如何启动的。启动MediaPlayerService的代码位于frameworks/base/media/mediaserver/main_mediaserver.cpp文件中：

```c
int main(int argc, char** argv)  
{  
    sp<ProcessState> proc(ProcessState::self());  
    sp<IServiceManager> sm = defaultServiceManager();  
    LOGI("ServiceManager: %p", sm.get());  
    AudioFlinger::instantiate();  
    MediaPlayerService::instantiate();  
    CameraService::instantiate();  
    AudioPolicyService::instantiate();  
    ProcessState::self()->startThreadPool();  
    IPCThreadState::self()->joinThreadPool();  
}  
```
这里我们不关注AudioFlinger和CameraService相关的代码。先看下面这句代码：

```c
sp<ProcessState> proc(ProcessState::self());  
```
这句代码的作用是通过ProcessState::self()调用创建一个ProcessState实例。ProcessState::self()是ProcessState类的一个静态成员变量，定义在frameworks/base/libs/binder/ProcessState.cpp文件中：

```c
sp<ProcessState> ProcessState::self()  
{  
    if (gProcess != NULL) return gProcess;  
      
    AutoMutex _l(gProcessMutex);  
    if (gProcess == NULL) gProcess = new ProcessState;  
    return gProcess;  
}  
```
这里可以看出，这个函数作用是返回一个全局唯一的ProcessState实例gProcess。全局唯一实例变量gProcess定义在frameworks/base/libs/binder/Static.cpp文件中：

```c
Mutex gProcessMutex;  
sp<ProcessState> gProcess;  
```
再来看ProcessState的构造函数：

```c
ProcessState::ProcessState()  
    : mDriverFD(open_driver())  
    , mVMStart(MAP_FAILED)  
    , mManagesContexts(false)  
    , mBinderContextCheckFunc(NULL)  
    , mBinderContextUserData(NULL)  
    , mThreadPoolStarted(false)  
    , mThreadPoolSeq(1)  
{  
    if (mDriverFD >= 0) {  
// XXX Ideally, there should be a specific define for whether we  
// have mmap (or whether we could possibly have the kernel module  
// availabla).  
#if !defined(HAVE_WIN32_IPC)  
// mmap the binder, providing a chunk of virtual address space to receive transactions.  
mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);  
if (mVMStart == MAP_FAILED) {  
    // *sigh*  
    LOGE("Using /dev/binder failed: unable to mmap transaction memory.\n");  
    close(mDriverFD);  
    mDriverFD = -1;  
}  
#else  
mDriverFD = -1;  
#endif  
    }  
    if (mDriverFD < 0) {  
// Need to run without the driver, starting our own thread pool.  
    }  
}  
```
这个函数有两个关键地方，一是通过open_driver函数打开Binder设备文件/dev/binder，并将打开设备文件描述符保存在成员变量mDriverFD中；二是通过mmap来把设备文件/dev/binder映射到内存中。

先看open_driver函数的实现，这个函数同样位于frameworks/base/libs/binder/ProcessState.cpp文件中：

```c
static int open_driver()  
{  
    if (gSingleProcess) {  
return -1;  
    }  

    int fd = open("/dev/binder", O_RDWR);  
    if (fd >= 0) {  
fcntl(fd, F_SETFD, FD_CLOEXEC);  
int vers;  
#if defined(HAVE_ANDROID_OS)  
status_t result = ioctl(fd, BINDER_VERSION, &vers);  
#else  
status_t result = -1;  
errno = EPERM;  
#endif  
if (result == -1) {  
    LOGE("Binder ioctl to obtain version failed: %s", strerror(errno));  
    close(fd);  
    fd = -1;  
}  
if (result != 0 || vers != BINDER_CURRENT_PROTOCOL_VERSION) {  
    LOGE("Binder driver protocol does not match user space protocol!");  
    close(fd);  
    fd = -1;  
}  
#if defined(HAVE_ANDROID_OS)  
size_t maxThreads = 15;  
result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);  
if (result == -1) {  
    LOGE("Binder ioctl to set max threads failed: %s", strerror(errno));  
}  
#endif  
  
    } else {  
LOGW("Opening '/dev/binder' failed: %s\n", strerror(errno));  
    }  
    return fd;  
}  
```
这个函数的作用主要是通过open文件操作函数来打开/dev/binder设备文件，然后再调用ioctl文件控制函数来分别执行BINDER_VERSION和BINDER_SET_MAX_THREADS两个命令来和Binder驱动程序进行交互，前者用于获得当前Binder驱动程序的版本号，后者用于通知Binder驱动程序，MediaPlayerService最多可同时启动15个线程来处理Client端的请求。

open在Binder驱动程序中的具体实现，请参考前面一篇文章[浅谈Service Manager成为Android进程间通信（IPC）机制Binder守护进程之路](http://blog.csdn.net/luoshengyang/article/details/6621566)，这里不再重复描述。打开/dev/binder设备文件后，Binder驱动程序就为MediaPlayerService进程创建了一个struct binder_proc结构体实例来维护MediaPlayerService进程上下文相关信息。

我们来看一下ioctl文件操作函数执行BINDER_VERSION命令的过程：

```c
status_t result = ioctl(fd, BINDER_VERSION, &vers);  
```
这个函数调用最终进入到Binder驱动程序的binder_ioctl函数中，我们只关注BINDER_VERSION相关的部分逻辑：

```c
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)  
{  
    int ret;  
    struct binder_proc *proc = filp->private_data;  
    struct binder_thread *thread;  
    unsigned int size = _IOC_SIZE(cmd);  
    void __user *ubuf = (void __user *)arg;  

    /*printk(KERN_INFO "binder_ioctl: %d:%d %x %lx\n", proc->pid, current->pid, cmd, arg);*/  

    ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);  
    if (ret)  
return ret;  

    mutex_lock(&binder_lock);  
    thread = binder_get_thread(proc);  
    if (thread == NULL) {  
ret = -ENOMEM;  
goto err;  
    }  

    switch (cmd) {  
    ......  
    case BINDER_VERSION:  
if (size != sizeof(struct binder_version)) {  
    ret = -EINVAL;  
    goto err;  
}  
if (put_user(BINDER_CURRENT_PROTOCOL_VERSION, &((struct binder_version *)ubuf)->protocol_version)) {  
    ret = -EINVAL;  
    goto err;  
}  
break;  
    ......  
    }  
    ret = 0;  
err:  
......  
    return ret;  
}  
```
很简单，只是将BINDER_CURRENT_PROTOCOL_VERSION写入到传入的参数arg指向的用户缓冲区中去就返回了。BINDER_CURRENT_PROTOCOL_VERSION是一个宏，定义在kernel/common/drivers/staging/android/binder.h文件中：

```c
/* This is the current protocol version. */  
#define BINDER_CURRENT_PROTOCOL_VERSION 7  
```
这里为什么要把ubuf转换成struct binder_version之后，再通过其protocol_version成员变量再来写入呢，转了一圈，最终内容还是写入到ubuf中。我们看一下struct binder_version的定义就会明白，同样是在kernel/common/drivers/staging/android/binder.h文件中：

```c
/* Use with BINDER_VERSION, driver fills in fields. */  
struct binder_version {  
    /* driver protocol version -- increment with incompatible change */  
    signed long protocol_version;  
};  
```
从注释中可以看出来，这里是考虑到兼容性，因为以后很有可能不是用signed long来表示版本号。

这里有一个重要的地方要注意的是，由于这里是打开设备文件/dev/binder之后，第一次进入到binder_ioctl函数，因此，这里调用binder_get_thread的时候，就会为当前线程创建一个struct binder_thread结构体变量来维护线程上下文信息，具体可以参考[浅谈Service Manager成为Android进程间通信（IPC）机制Binder守护进程之路](http://blog.csdn.net/luoshengyang/article/details/6621566)一文。

接着我们再来看一下ioctl文件操作函数执行BINDER_SET_MAX_THREADS命令的过程：

```c
result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);  
```
这个函数调用最终进入到Binder驱动程序的binder_ioctl函数中，我们只关注BINDER_SET_MAX_THREADS相关的部分逻辑：

```c
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)  
{  
    int ret;  
    struct binder_proc *proc = filp->private_data;  
    struct binder_thread *thread;  
    unsigned int size = _IOC_SIZE(cmd);  
    void __user *ubuf = (void __user *)arg;  

    /*printk(KERN_INFO "binder_ioctl: %d:%d %x %lx\n", proc->pid, current->pid, cmd, arg);*/  

    ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);  
    if (ret)  
return ret;  

    mutex_lock(&binder_lock);  
    thread = binder_get_thread(proc);  
    if (thread == NULL) {  
ret = -ENOMEM;  
goto err;  
    }  

    switch (cmd) {  
    ......  
    case BINDER_SET_MAX_THREADS:  
if (copy_from_user(&proc->max_threads, ubuf, sizeof(proc->max_threads))) {  
    ret = -EINVAL;  
    goto err;  
}  
break;  
    ......  
    }  
    ret = 0;  
err:  
    ......  
    return ret;  
}  
```
这里实现也是非常简单，只是简单地把用户传进来的参数保存在proc->max_threads中就完毕了。注意，这里再调用binder_get_thread函数的时候，就可以在proc->threads中找到当前线程对应的struct binder_thread结构了，因为前面已经创建好并保存在proc->threads红黑树中。

回到ProcessState的构造函数中，这里还通过mmap函数来把设备文件/dev/binder映射到内存中，这个函数在[浅谈Service Manager成为Android进程间通信（IPC）机制Binder守护进程之路](http://blog.csdn.net/luoshengyang/article/details/6621566)一文也已经有详细介绍，这里不再重复描述。宏BINDER_VM_SIZE就定义在ProcessState.cpp文件中：

```c
#define BINDER_VM_SIZE ((1*1024*1024) - (4096 *2))  
```
mmap函数调用完成之后，Binder驱动程序就为当前进程预留了BINDER_VM_SIZE大小的内存空间了。

这样，ProcessState全局唯一变量gProcess就创建完毕了，回到frameworks/base/media/mediaserver/main_mediaserver.cpp文件中的main函数，下一步是调用defaultServiceManager函数来获得Service Manager的远程接口，这个已经在上一篇文章[浅谈Android系统进程间通信（IPC）机制Binder中的Server和Client获得Service Manager接口之路](http://blog.csdn.net/luoshengyang/article/details/6627260)有详细描述，读者可以回过头去参考一下。

再接下来，就进入到MediaPlayerService::instantiate函数把MediaPlayerService添加到Service Manger中去了。这个函数定义在frameworks/base/media/libmediaplayerservice/MediaPlayerService.cpp文件中：

```c
void MediaPlayerService::instantiate() {  
    defaultServiceManager()->addService(  
    String16("media.player"), new MediaPlayerService());  
}  
```
我们重点看一下IServiceManger::addService的过程，这有助于我们加深对Binder机制的理解。

在上一篇文章[浅谈Android系统进程间通信（IPC）机制Binder中的Server和Client获得Service Manager接口之路](http://blog.csdn.net/luoshengyang/article/details/6627260)中说到，defaultServiceManager返回的实际是一个BpServiceManger类实例，因此，我们看一下BpServiceManger::addService的实现，这个函数实现在frameworks/base/libs/binder/IServiceManager.cpp文件中：

```c
class BpServiceManager : public BpInterface<IServiceManager>  
{  
public:  
    BpServiceManager(const sp<IBinder>& impl)  
: BpInterface<IServiceManager>(impl)  
    {  
    }  

    ......  

    virtual status_t addService(const String16& name, const sp<IBinder>& service)  
    {  
Parcel data, reply;  
data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());  
data.writeString16(name);  
data.writeStrongBinder(service);  
status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);  
return err == NO_ERROR ? reply.readExceptionCode()   
    }  

    ......  

};  
```
这里的Parcel类是用来于序列化进程间通信数据用的。

先来看这一句的调用：

```c
data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());  
```
IServiceManager::getInterfaceDescriptor()返回来的是一个字符串，即"android.os.IServiceManager"，具体可以参考IServiceManger的实现。我们看一下Parcel::writeInterfaceToken的实现，位于frameworks/base/libs/binder/Parcel.cpp文件中：

```c
// Write RPC headers.  (previously just the interface token)  
status_t Parcel::writeInterfaceToken(const String16& interface)  
{  
    writeInt32(IPCThreadState::self()->getStrictModePolicy() |  
       STRICT_MODE_PENALTY_GATHER);  
    // currently the interface identification token is just its name as a string  
    return writeString16(interface);  
}  
```
它的作用是写入一个整数和一个字符串到Parcel中去。

再来看下面的调用：

```c
data.writeString16(name);  
```
这里又是写入一个字符串到Parcel中去，这里的name即是上面传进来的“media.player”字符串。

往下看：

```c
data.writeStrongBinder(service);  
```
这里定入一个Binder对象到Parcel去。我们重点看一下这个函数的实现，因为它涉及到进程间传输Binder实体的问题，比较复杂，需要重点关注，同时，也是理解Binder机制的一个重点所在。注意，这里的service参数是一个MediaPlayerService对象。

```c
status_t Parcel::writeStrongBinder(const sp<IBinder>& val)  
{  
    return flatten_binder(ProcessState::self(), val, this);  
}  
```
看到flatten_binder函数，是不是似曾相识的感觉？我们在前面一篇文章

浅谈Service Manager成为Android进程间通信（IPC）机制Binder守护进程之路

中，曾经提到在Binder驱动程序中，使用struct flat_binder_object来表示传输中的一个binder对象，它的定义如下所示：

```c
/* 
* This is the flattened representation of a Binder object for transfer 
* between processes.  The 'offsets' supplied as part of a binder transaction 
* contains offsets into the data where these structures occur.  The Binder 
* driver takes care of re-writing the structure type and data as it moves 
* between processes. 
*/  
struct flat_binder_object {  
    /* 8 bytes for large_flat_header. */  
    unsigned long       type;  
    unsigned long       flags;  

    /* 8 bytes of data. */  
    union {  
void        *binder;    /* local object */  
signed long handle;     /* remote object */  
    };  

    /* extra data associated with local object */  
    void            *cookie;  
};  
```
各个成员变量的含义请参考资料Android Binder设计与实现

我们进入到flatten_binder函数看看：

```c
status_t flatten_binder(const sp<ProcessState>& proc,  
    const sp<IBinder>& binder, Parcel* out)  
{  
    flat_binder_object obj;  
      
    obj.flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;  
    if (binder != NULL) {  
IBinder *local = binder->localBinder();  
if (!local) {  
    BpBinder *proxy = binder->remoteBinder();  
    if (proxy == NULL) {  
        LOGE("null proxy");  
    }  
    const int32_t handle = proxy ? proxy->handle() : 0;  
    obj.type = BINDER_TYPE_HANDLE;  
    obj.handle = handle;  
    obj.cookie = NULL;  
} else {  
    obj.type = BINDER_TYPE_BINDER;  
    obj.binder = local->getWeakRefs();  
    obj.cookie = local;  
}  
    } else {  
obj.type = BINDER_TYPE_BINDER;  
obj.binder = NULL;  
obj.cookie = NULL;  
    }  
      
    return finish_flatten_binder(binder, obj, out);  
}  
```
首先是初始化flat_binder_object的flags域：

```c
obj.flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;  
```
0x7f表示处理本Binder实体请求数据包的线程的最低优先级，FLAT_BINDER_FLAG_ACCEPTS_FDS表示这个Binder实体可以接受文件描述符，Binder实体在收到文件描述符时，就会在本进程中打开这个文件。

传进来的binder即为MediaPlayerService::instantiate函数中new出来的MediaPlayerService实例，因此，不为空。又由于MediaPlayerService继承自BBinder类，它是一个本地Binder实体，因此binder->localBinder返回一个BBinder指针，而且肯定不为空，于是执行下面语句：

```c
obj.type = BINDER_TYPE_BINDER;  
obj.binder = local->getWeakRefs();  
obj.cookie = local;  
```
设置了flat_binder_obj的其他成员变量，注意，指向这个Binder实体地址的指针local保存在flat_binder_obj的成员变量cookie中。

函数调用finish_flatten_binder来将这个flat_binder_obj写入到Parcel中去：

```c
inline static status_t finish_flatten_binder(  
    const sp<IBinder>& binder, const flat_binder_object& flat, Parcel* out)  
{  
    return out->writeObject(flat, false);  
}  
```
Parcel::writeObject的实现如下：

```c
status_t Parcel::writeObject(const flat_binder_object& val, bool nullMetaData)  
{  
    const bool enoughData = (mDataPos+sizeof(val)) <= mDataCapacity;  
    const bool enoughObjects = mObjectsSize < mObjectsCapacity;  
    if (enoughData && enoughObjects) {  
restart_write:  
*reinterpret_cast<flat_binder_object*>(mData+mDataPos) = val;  
  
// Need to write meta-data?  
if (nullMetaData || val.binder != NULL) {  
    mObjects[mObjectsSize] = mDataPos;  
    acquire_object(ProcessState::self(), val, this);  
    mObjectsSize++;  
}  
  
// remember if it's a file descriptor  
if (val.type == BINDER_TYPE_FD) {  
    mHasFds = mFdsKnown = true;  
}  

return finishWrite(sizeof(flat_binder_object));  
    }  

    if (!enoughData) {  
const status_t err = growData(sizeof(val));  
if (err != NO_ERROR) return err;  
    }  
    if (!enoughObjects) {  
size_t newSize = ((mObjectsSize+2)*3)/2;  
size_t* objects = (size_t*)realloc(mObjects, newSize*sizeof(size_t));  
if (objects == NULL) return NO_MEMORY;  
mObjects = objects;  
mObjectsCapacity = newSize;  
    }  
      
    goto restart_write;  
}  
```
这里除了把flat_binder_obj写到Parcel里面之内，还要记录这个flat_binder_obj在Parcel里面的偏移位置：

```c
mObjects[mObjectsSize] = mDataPos;  
```
这里因为，如果进程间传输的数据间带有Binder对象的时候，Binder驱动程序需要作进一步的处理，以维护各个Binder实体的一致性，下面我们将会看到Binder驱动程序是怎么处理这些Binder对象的。

再回到BpServiceManager::addService函数中，调用下面语句：

```c
status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);  
```
回到浅谈Android系统进程间通信（IPC）机制Binder中的Server和Client获得Service Manager接口之路

一文中的类图中去看一下，这里的remote成员函数来自于BpRefBase类，它返回一个BpBinder指针。因此，我们继续进入到BpBinder::transact函数中去看看：

```c
status_t BpBinder::transact(  
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)  
{  
    // Once a binder has died, it will never come back to life.  
    if (mAlive) {  
status_t status = IPCThreadState::self()->transact(  
    mHandle, code, data, reply, flags);  
if (status == DEAD_OBJECT) mAlive = 0;  
return status;  
    }  

    return DEAD_OBJECT;  
}  
```
这里又调用了IPCThreadState::transact进执行实际的操作。注意，这里的mHandle为0，code为ADD_SERVICE_TRANSACTION。ADD_SERVICE_TRANSACTION是上面以参数形式传进来的，那mHandle为什么是0呢？因为这里表示的是Service Manager远程接口，它的句柄值一定是0，具体请参考[浅谈Android系统进程间通信（IPC）机制Binder中的Server和Client获得Service Manager接口之路]()一文。

再进入到IPCThreadState::transact函数，看看做了些什么事情：

```c
status_t IPCThreadState::transact(int32_t handle,  
                          uint32_t code, const Parcel& data,  
                          Parcel* reply, uint32_t flags)  
{  
    status_t err = data.errorCheck();  

    flags |= TF_ACCEPT_FDS;  

    IF_LOG_TRANSACTIONS() {  
TextOutput::Bundle _b(alog);  
alog << "BC_TRANSACTION thr " << (void*)pthread_self() << " / hand "  
    << handle << " / code " << TypeCode(code) << ": "  
    << indent << data << dedent << endl;  
    }  
      
    if (err == NO_ERROR) {  
LOG_ONEWAY(">>>> SEND from pid %d uid %d %s", getpid(), getuid(),  
    (flags & TF_ONE_WAY) == 0 ? "READ REPLY" : "ONE WAY");  
err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);  
    }  
      
    if (err != NO_ERROR) {  
if (reply) reply->setError(err);  
return (mLastError = err);  
    }  
      
    if ((flags & TF_ONE_WAY) == 0) {  
#if 0  
if (code == 4) { // relayout  
    LOGI(">>>>>> CALLING transaction 4");  
} else {  
    LOGI(">>>>>> CALLING transaction %d", code);  
}  
#endif  
if (reply) {  
    err = waitForResponse(reply);  
} else {  
    Parcel fakeReply;  
    err = waitForResponse(&fakeReply);  
}  
#if 0  
if (code == 4) { // relayout  
    LOGI("<<<<<< RETURNING transaction 4");  
} else {  
    LOGI("<<<<<< RETURNING transaction %d", code);  
}  
#endif  
  
IF_LOG_TRANSACTIONS() {  
    TextOutput::Bundle _b(alog);  
    alog << "BR_REPLY thr " << (void*)pthread_self() << " / hand "  
        << handle << ": ";  
    if (reply) alog << indent << *reply << dedent << endl;  
    else alog << "(none requested)" << endl;  
}  
    } else {  
err = waitForResponse(NULL, NULL);  
    }  
      
    return err;  
}  
```
IPCThreadState::transact函数的参数flags是一个默认值为0的参数，上面没有传相应的实参进来，因此，这里就为0。

函数首先调用writeTransactionData函数准备好一个struct binder_transaction_data结构体变量，这个是等一下要传输给Binder驱动程序的。struct binder_transaction_data的定义我们在[浅谈Service Manager成为Android进程间通信（IPC）机制Binder守护进程之路](http://blog.csdn.net/luoshengyang/article/details/6621566)一文中有详细描述，读者不妨回过去读一下。这里为了方便描述，将struct binder_transaction_data的定义再次列出来：

```c
struct binder_transaction_data {  
    /* The first two are only used for bcTRANSACTION and brTRANSACTION, 
     * identifying the target and contents of the transaction. 
     */  
    union {  
size_t  handle; /* target descriptor of command transaction */  
void    *ptr;   /* target descriptor of return transaction */  
    } target;  
    void        *cookie;    /* target object cookie */  
    unsigned int    code;       /* transaction command */  

    /* General information about the transaction. */  
    unsigned int    flags;  
    pid_t       sender_pid;  
    uid_t       sender_euid;  
    size_t      data_size;  /* number of bytes of data */  
    size_t      offsets_size;   /* number of bytes of offsets */  

    /* If this transaction is inline, the data immediately 
     * follows here; otherwise, it ends with a pointer to 
     * the data buffer. 
     */  
    union {  
struct {  
    /* transaction data */  
    const void  *buffer;  
    /* offsets from buffer to flat_binder_object structs */  
    const void  *offsets;  
} ptr;  
uint8_t buf[8];  
    } data;  
};  
```
writeTransactionData函数的实现如下：

```c
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,  
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)  
{  
    binder_transaction_data tr;  

    tr.target.handle = handle;  
    tr.code = code;  
    tr.flags = binderFlags;  
      
    const status_t err = data.errorCheck();  
    if (err == NO_ERROR) {  
tr.data_size = data.ipcDataSize();  
tr.data.ptr.buffer = data.ipcData();  
tr.offsets_size = data.ipcObjectsCount()*sizeof(size_t);  
tr.data.ptr.offsets = data.ipcObjects();  
    } else if (statusBuffer) {  
tr.flags |= TF_STATUS_CODE;  
*statusBuffer = err;  
tr.data_size = sizeof(status_t);  
tr.data.ptr.buffer = statusBuffer;  
tr.offsets_size = 0;  
tr.data.ptr.offsets = NULL;  
    } else {  
return (mLastError = err);  
    }  
      
    mOut.writeInt32(cmd);  
    mOut.write(&tr, sizeof(tr));  
      
    return NO_ERROR;  
}  
```
注意，这里的cmd为BC_TRANSACTION。 这个函数很简单，在这个场景下，就是执行下面语句来初始化本地变量tr：

```c
tr.data_size = data.ipcDataSize();  
tr.data.ptr.buffer = data.ipcData();  
tr.offsets_size = data.ipcObjectsCount()*sizeof(size_t);  
tr.data.ptr.offsets = data.ipcObjects();  
```
回忆一下上面的内容，写入到tr.data.ptr.buffer的内容相当于下面的内容：

```c
writeInt32(IPCThreadState::self()->getStrictModePolicy() |  
       STRICT_MODE_PENALTY_GATHER);  
writeString16("android.os.IServiceManager");  
writeString16("media.player");  
writeStrongBinder(new MediaPlayerService());  
```
其中包含了一个Binder实体MediaPlayerService，因此需要设置tr.offsets_size就为1，tr.data.ptr.offsets就指向了这个MediaPlayerService的地址在tr.data.ptr.buffer中的偏移量。最后，将tr的内容保存在IPCThreadState的成员变量mOut中。

回到IPCThreadState::transact函数中，接下去看，(flags & TF_ONE_WAY) == 0为true，并且reply不为空，所以最终进入到waitForResponse(reply)这条路径来。我们看一下waitForResponse函数的实现：

```c
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)  
{  
    int32_t cmd;  
    int32_t err;  

    while (1) {  
if ((err=talkWithDriver()) < NO_ERROR) break;  
err = mIn.errorCheck();  
if (err < NO_ERROR) break;  
if (mIn.dataAvail() == 0) continue;  
  
cmd = mIn.readInt32();  
  
IF_LOG_COMMANDS() {  
    alog << "Processing waitForResponse Command: "  
        << getReturnString(cmd) << endl;  
}  

switch (cmd) {  
case BR_TRANSACTION_COMPLETE:  
    if (!reply && !acquireResult) goto finish;  
    break;  
  
case BR_DEAD_REPLY:  
    err = DEAD_OBJECT;  
    goto finish;  

case BR_FAILED_REPLY:  
    err = FAILED_TRANSACTION;  
    goto finish;  
  
case BR_ACQUIRE_RESULT:  
    {  
        LOG_ASSERT(acquireResult != NULL, "Unexpected brACQUIRE_RESULT");  
        const int32_t result = mIn.readInt32();  
        if (!acquireResult) continue;  
        *acquireResult = result ? NO_ERROR : INVALID_OPERATION;  
    }  
    goto finish;  
  
case BR_REPLY:  
    {  
        binder_transaction_data tr;  
        err = mIn.read(&tr, sizeof(tr));  
        LOG_ASSERT(err == NO_ERROR, "Not enough command data for brREPLY");  
        if (err != NO_ERROR) goto finish;  

        if (reply) {  
            if ((tr.flags & TF_STATUS_CODE) == 0) {  
                reply->ipcSetDataReference(  
                    reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),  
                    tr.data_size,  
                    reinterpret_cast<const size_t*>(tr.data.ptr.offsets),  
                    tr.offsets_size/sizeof(size_t),  
                    freeBuffer, this);  
            } else {  
                err = *static_cast<const status_t*>(tr.data.ptr.buffer);  
                freeBuffer(NULL,  
                    reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),  
                    tr.data_size,  
                    reinterpret_cast<const size_t*>(tr.data.ptr.offsets),  
                    tr.offsets_size/sizeof(size_t), this);  
            }  
        } else {  
            freeBuffer(NULL,  
                reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),  
                tr.data_size,  
                reinterpret_cast<const size_t*>(tr.data.ptr.offsets),  
                tr.offsets_size/sizeof(size_t), this);  
            continue;  
        }  
    }  
    goto finish;  

default:  
    err = executeCommand(cmd);  
    if (err != NO_ERROR) goto finish;  
    break;  
}  
    }  

finish:  
    if (err != NO_ERROR) {  
if (acquireResult) *acquireResult = err;  
if (reply) reply->setError(err);  
mLastError = err;  
    }  
      
    return err;  
}  
```
这个函数虽然很长，但是主要调用了talkWithDriver函数来与Binder驱动程序进行交互：

```c
status_t IPCThreadState::talkWithDriver(bool doReceive)  
{  
    LOG_ASSERT(mProcess->mDriverFD >= 0, "Binder driver is not opened");  
      
    binder_write_read bwr;  
      
    // Is the read buffer empty?  
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();  
      
    // We don't want to write anything if we are still reading  
    // from data left in the input buffer and the caller  
    // has requested to read the next data.  
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;  
      
    bwr.write_size = outAvail;  
    bwr.write_buffer = (long unsigned int)mOut.data();  

    // This is what we'll read.  
    if (doReceive && needRead) {  
bwr.read_size = mIn.dataCapacity();  
bwr.read_buffer = (long unsigned int)mIn.data();  
    } else {  
bwr.read_size = 0;  
    }  
      
    IF_LOG_COMMANDS() {  
TextOutput::Bundle _b(alog);  
if (outAvail != 0) {  
    alog << "Sending commands to driver: " << indent;  
    const void* cmds = (const void*)bwr.write_buffer;  
    const void* end = ((const uint8_t*)cmds)+bwr.write_size;  
    alog << HexDump(cmds, bwr.write_size) << endl;  
    while (cmds < end) cmds = printCommand(alog, cmds);  
    alog << dedent;  
}  
alog << "Size of receive buffer: " << bwr.read_size  
    << ", needRead: " << needRead << ", doReceive: " << doReceive << endl;  
    }  
      
    // Return immediately if there is nothing to do.  
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;  
      
    bwr.write_consumed = 0;  
    bwr.read_consumed = 0;  
    status_t err;  
    do {  
IF_LOG_COMMANDS() {  
    alog << "About to read/write, write size = " << mOut.dataSize() << endl;  
}  
#if defined(HAVE_ANDROID_OS)  
if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)  
    err = NO_ERROR;  
else  
    err = -errno;  
#else  
err = INVALID_OPERATION;  
#endif  
IF_LOG_COMMANDS() {  
    alog << "Finished read/write, write size = " << mOut.dataSize() << endl;  
}  
    } while (err == -EINTR);  
      
    IF_LOG_COMMANDS() {  
alog << "Our err: " << (void*)err << ", write consumed: "  
    << bwr.write_consumed << " (of " << mOut.dataSize()  
    << "), read consumed: " << bwr.read_consumed << endl;  
    }  

    if (err >= NO_ERROR) {  
if (bwr.write_consumed > 0) {  
    if (bwr.write_consumed < (ssize_t)mOut.dataSize())  
        mOut.remove(0, bwr.write_consumed);  
    else  
        mOut.setDataSize(0);  
}  
if (bwr.read_consumed > 0) {  
    mIn.setDataSize(bwr.read_consumed);  
    mIn.setDataPosition(0);  
}  
IF_LOG_COMMANDS() {  
    TextOutput::Bundle _b(alog);  
    alog << "Remaining data size: " << mOut.dataSize() << endl;  
    alog << "Received commands from driver: " << indent;  
    const void* cmds = mIn.data();  
    const void* end = mIn.data() + mIn.dataSize();  
    alog << HexDump(cmds, mIn.dataSize()) << endl;  
    while (cmds < end) cmds = printReturnCommand(alog, cmds);  
    alog << dedent;  
}  
return NO_ERROR;  
    }  
      
    return err;  
}  
```
这里doReceive和needRead均为1，有兴趣的读者可以自已分析一下。因此，这里告诉Binder驱动程序，先执行write操作，再执行read操作，下面我们将会看到。

最后，通过ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)进行到Binder驱动程序的binder_ioctl函数，我们只关注cmd为BINDER_WRITE_READ的逻辑：

```c
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)  
{  
    int ret;  
    struct binder_proc *proc = filp->private_data;  
    struct binder_thread *thread;  
    unsigned int size = _IOC_SIZE(cmd);  
    void __user *ubuf = (void __user *)arg;  

    /*printk(KERN_INFO "binder_ioctl: %d:%d %x %lx\n", proc->pid, current->pid, cmd, arg);*/  

    ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);  
    if (ret)  
return ret;  

    mutex_lock(&binder_lock);  
    thread = binder_get_thread(proc);  
    if (thread == NULL) {  
ret = -ENOMEM;  
goto err;  
    }  

    switch (cmd) {  
    case BINDER_WRITE_READ: {  
struct binder_write_read bwr;  
if (size != sizeof(struct binder_write_read)) {  
    ret = -EINVAL;  
    goto err;  
}  
if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {  
    ret = -EFAULT;  
    goto err;  
}  
if (binder_debug_mask & BINDER_DEBUG_READ_WRITE)  
    printk(KERN_INFO "binder: %d:%d write %ld at %08lx, read %ld at %08lx\n",  
    proc->pid, thread->pid, bwr.write_size, bwr.write_buffer, bwr.read_size, bwr.read_buffer);  
if (bwr.write_size > 0) {  
    ret = binder_thread_write(proc, thread, (void __user *)bwr.write_buffer, bwr.write_size, &bwr.write_consumed);  
    if (ret < 0) {  
        bwr.read_consumed = 0;  
        if (copy_to_user(ubuf, &bwr, sizeof(bwr)))  
            ret = -EFAULT;  
        goto err;  
    }  
}  
if (bwr.read_size > 0) {  
    ret = binder_thread_read(proc, thread, (void __user *)bwr.read_buffer, bwr.read_size, &bwr.read_consumed, filp->f_flags & O_NONBLOCK);  
    if (!list_empty(&proc->todo))  
        wake_up_interruptible(&proc->wait);  
    if (ret < 0) {  
        if (copy_to_user(ubuf, &bwr, sizeof(bwr)))  
            ret = -EFAULT;  
        goto err;  
    }  
}  
if (binder_debug_mask & BINDER_DEBUG_READ_WRITE)  
    printk(KERN_INFO "binder: %d:%d wrote %ld of %ld, read return %ld of %ld\n",  
    proc->pid, thread->pid, bwr.write_consumed, bwr.write_size, bwr.read_consumed, bwr.read_size);  
if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {  
    ret = -EFAULT;  
    goto err;  
}  
break;  
    }  
    ......  
    }  
    ret = 0;  
err:  
    ......  
    return ret;  
}  
```
函数首先是将用户传进来的参数拷贝到本地变量struct binder_write_read bwr中去。这里bwr.write_size > 0为true，因此，进入到binder_thread_write函数中，我们只关注BC_TRANSACTION部分的逻辑：

```c
binder_thread_write(struct binder_proc *proc, struct binder_thread *thread,  
            void __user *buffer, int size, signed long *consumed)  
{  
    uint32_t cmd;  
    void __user *ptr = buffer + *consumed;  
    void __user *end = buffer + size;  

    while (ptr < end && thread->return_error == BR_OK) {  
if (get_user(cmd, (uint32_t __user *)ptr))  
    return -EFAULT;  
ptr += sizeof(uint32_t);  
if (_IOC_NR(cmd) < ARRAY_SIZE(binder_stats.bc)) {  
    binder_stats.bc[_IOC_NR(cmd)]++;  
    proc->stats.bc[_IOC_NR(cmd)]++;  
    thread->stats.bc[_IOC_NR(cmd)]++;  
}  
switch (cmd) {  
    .....  
case BC_TRANSACTION:  
case BC_REPLY: {  
    struct binder_transaction_data tr;  

    if (copy_from_user(&tr, ptr, sizeof(tr)))  
        return -EFAULT;  
    ptr += sizeof(tr);  
    binder_transaction(proc, thread, &tr, cmd == BC_REPLY);  
    break;  
}  
......  
}  
*consumed = ptr - buffer;  
    }  
    return 0;  
}  
```
首先将用户传进来的transact参数拷贝在本地变量struct binder_transaction_data tr中去，接着调用binder_transaction函数进一步处理，这里我们忽略掉无关代码：

```c
static void  
binder_transaction(struct binder_proc *proc, struct binder_thread *thread,  
struct binder_transaction_data *tr, int reply)  
{  
    struct binder_transaction *t;  
    struct binder_work *tcomplete;  
    size_t *offp, *off_end;  
    struct binder_proc *target_proc;  
    struct binder_thread *target_thread = NULL;  
    struct binder_node *target_node = NULL;  
    struct list_head *target_list;  
    wait_queue_head_t *target_wait;  
    struct binder_transaction *in_reply_to = NULL;  
    struct binder_transaction_log_entry *e;  
    uint32_t return_error;  

......  

    if (reply) {  
 ......  
    } else {  
if (tr->target.handle) {  
    ......  
} else {  
    target_node = binder_context_mgr_node;  
    if (target_node == NULL) {  
        return_error = BR_DEAD_REPLY;  
        goto err_no_context_mgr_node;  
    }  
}  
......  
target_proc = target_node->proc;  
if (target_proc == NULL) {  
    return_error = BR_DEAD_REPLY;  
    goto err_dead_binder;  
}  
......  
    }  
    if (target_thread) {  
......  
    } else {  
target_list = &target_proc->todo;  
target_wait = &target_proc->wait;  
    }  
      
    ......  

    /* TODO: reuse incoming transaction for reply */  
    t = kzalloc(sizeof(*t), GFP_KERNEL);  
    if (t == NULL) {  
return_error = BR_FAILED_REPLY;  
goto err_alloc_t_failed;  
    }  
    ......  

    tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);  
    if (tcomplete == NULL) {  
return_error = BR_FAILED_REPLY;  
goto err_alloc_tcomplete_failed;  
    }  
      
    ......  

    if (!reply && !(tr->flags & TF_ONE_WAY))  
t->from = thread;  
    else  
t->from = NULL;  
    t->sender_euid = proc->tsk->cred->euid;  
    t->to_proc = target_proc;  
    t->to_thread = target_thread;  
    t->code = tr->code;  
    t->flags = tr->flags;  
    t->priority = task_nice(current);  
    t->buffer = binder_alloc_buf(target_proc, tr->data_size,  
tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));  
    if (t->buffer == NULL) {  
return_error = BR_FAILED_REPLY;  
goto err_binder_alloc_buf_failed;  
    }  
    t->buffer->allow_user_free = 0;  
    t->buffer->debug_id = t->debug_id;  
    t->buffer->transaction = t;  
    t->buffer->target_node = target_node;  
    if (target_node)  
binder_inc_node(target_node, 1, 0, NULL);  

    offp = (size_t *)(t->buffer->data + ALIGN(tr->data_size, sizeof(void *)));  

    if (copy_from_user(t->buffer->data, tr->data.ptr.buffer, tr->data_size)) {  
......  
return_error = BR_FAILED_REPLY;  
goto err_copy_data_failed;  
    }  
    if (copy_from_user(offp, tr->data.ptr.offsets, tr->offsets_size)) {  
......  
return_error = BR_FAILED_REPLY;  
goto err_copy_data_failed;  
    }  
    ......  

    off_end = (void *)offp + tr->offsets_size;  
    for (; offp < off_end; offp++) {  
struct flat_binder_object *fp;  
......  
fp = (struct flat_binder_object *)(t->buffer->data + *offp);  
switch (fp->type) {  
case BINDER_TYPE_BINDER:  
case BINDER_TYPE_WEAK_BINDER: {  
    struct binder_ref *ref;  
    struct binder_node *node = binder_get_node(proc, fp->binder);  
    if (node == NULL) {  
        node = binder_new_node(proc, fp->binder, fp->cookie);  
        if (node == NULL) {  
            return_error = BR_FAILED_REPLY;  
            goto err_binder_new_node_failed;  
        }  
        node->min_priority = fp->flags & FLAT_BINDER_FLAG_PRIORITY_MASK;  
        node->accept_fds = !!(fp->flags & FLAT_BINDER_FLAG_ACCEPTS_FDS);  
    }  
    if (fp->cookie != node->cookie) {  
        ......  
        goto err_binder_get_ref_for_node_failed;  
    }  
    ref = binder_get_ref_for_node(target_proc, node);  
    if (ref == NULL) {  
        return_error = BR_FAILED_REPLY;  
        goto err_binder_get_ref_for_node_failed;  
    }  
    if (fp->type == BINDER_TYPE_BINDER)  
        fp->type = BINDER_TYPE_HANDLE;  
    else  
        fp->type = BINDER_TYPE_WEAK_HANDLE;  
    fp->handle = ref->desc;  
    binder_inc_ref(ref, fp->type == BINDER_TYPE_HANDLE, &thread->todo);  
    ......  
                        
} break;  
......  
}  
    }  

    if (reply) {  
......  
    } else if (!(t->flags & TF_ONE_WAY)) {  
BUG_ON(t->buffer->async_transaction != 0);  
t->need_reply = 1;  
t->from_parent = thread->transaction_stack;  
thread->transaction_stack = t;  
    } else {  
......  
    }  
    t->work.type = BINDER_WORK_TRANSACTION;  
    list_add_tail(&t->work.entry, target_list);  
    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;  
    list_add_tail(&tcomplete->entry, &thread->todo);  
    if (target_wait)  
wake_up_interruptible(target_wait);  
    return;  
    ......  
}  
```
注意，这里传进来的参数reply为0，tr->target.handle也为0。因此，target_proc、target_thread、target_node、target_list和target_wait的值分别为：

```c
target_node = binder_context_mgr_node;  
target_proc = target_node->proc;  
target_list = &target_proc->todo;  
target_wait = &target_proc->wait;   
```
接着，分配了一个待处理事务t和一个待完成工作项tcomplete，并执行初始化工作：

```c
/* TODO: reuse incoming transaction for reply */  
t = kzalloc(sizeof(*t), GFP_KERNEL);  
if (t == NULL) {  
    return_error = BR_FAILED_REPLY;  
    goto err_alloc_t_failed;  
}  
......  

tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);  
if (tcomplete == NULL) {  
    return_error = BR_FAILED_REPLY;  
    goto err_alloc_tcomplete_failed;  
}  

......  

if (!reply && !(tr->flags & TF_ONE_WAY))  
    t->from = thread;  
else  
    t->from = NULL;  
t->sender_euid = proc->tsk->cred->euid;  
t->to_proc = target_proc;  
t->to_thread = target_thread;  
t->code = tr->code;  
t->flags = tr->flags;  
t->priority = task_nice(current);  
t->buffer = binder_alloc_buf(target_proc, tr->data_size,  
    tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));  
if (t->buffer == NULL) {  
    return_error = BR_FAILED_REPLY;  
    goto err_binder_alloc_buf_failed;  
}  
t->buffer->allow_user_free = 0;  
t->buffer->debug_id = t->debug_id;  
t->buffer->transaction = t;  
t->buffer->target_node = target_node;  
if (target_node)  
    binder_inc_node(target_node, 1, 0, NULL);  

offp = (size_t *)(t->buffer->data + ALIGN(tr->data_size, sizeof(void *)));  

if (copy_from_user(t->buffer->data, tr->data.ptr.buffer, tr->data_size)) {  
    ......  
    return_error = BR_FAILED_REPLY;  
    goto err_copy_data_failed;  
}  
if (copy_from_user(offp, tr->data.ptr.offsets, tr->offsets_size)) {  
    ......  
    return_error = BR_FAILED_REPLY;  
    goto err_copy_data_failed;  
}  
```
注意，这里的事务t是要交给target_proc处理的，在这个场景之下，就是Service Manager了。因此，下面的语句：

```c
t->buffer = binder_alloc_buf(target_proc, tr->data_size,  
tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));  
```
就是在Service Manager的进程空间中分配一块内存来保存用户传进入的参数了：

```c
if (copy_from_user(t->buffer->data, tr->data.ptr.buffer, tr->data_size)) {  
    ......  
    return_error = BR_FAILED_REPLY;  
    goto err_copy_data_failed;  
}  
if (copy_from_user(offp, tr->data.ptr.offsets, tr->offsets_size)) {  
    ......  
    return_error = BR_FAILED_REPLY;  
    goto err_copy_data_failed;  
}  
```
由于现在target_node要被使用了，增加它的引用计数：

```c
if (target_node)  
binder_inc_node(target_node, 1, 0, NULL);  
```
接下去的for循环，就是用来处理传输数据中的Binder对象了。在我们的场景中，有一个类型为BINDER_TYPE_BINDER的Binder实体MediaPlayerService：

```c
   switch (fp->type) {  
   case BINDER_TYPE_BINDER:  
   case BINDER_TYPE_WEAK_BINDER: {  
   struct binder_ref *ref;  
   struct binder_node *node = binder_get_node(proc, fp->binder);  
   if (node == NULL) {  
       node = binder_new_node(proc, fp->binder, fp->cookie);  
       if (node == NULL) {  
   return_error = BR_FAILED_REPLY;  
   goto err_binder_new_node_failed;  
       }  
       node->min_priority = fp->flags & FLAT_BINDER_FLAG_PRIORITY_MASK;  
       node->accept_fds = !!(fp->flags & FLAT_BINDER_FLAG_ACCEPTS_FDS);  
   }  
   if (fp->cookie != node->cookie) {  
       ......  
       goto err_binder_get_ref_for_node_failed;  
   }  
   ref = binder_get_ref_for_node(target_proc, node);  
   if (ref == NULL) {  
       return_error = BR_FAILED_REPLY;  
       goto err_binder_get_ref_for_node_failed;  
   }  
   if (fp->type == BINDER_TYPE_BINDER)  
       fp->type = BINDER_TYPE_HANDLE;  
   else  
       fp->type = BINDER_TYPE_WEAK_HANDLE;  
   fp->handle = ref->desc;  
   binder_inc_ref(ref, fp->type == BINDER_TYPE_HANDLE, &thread->todo);  
   ......  
                       
   } break;  
```
由于是第一次在Binder驱动程序中传输这个MediaPlayerService，调用binder_get_node函数查询这个Binder实体时，会返回空，于是binder_new_node在proc中新建一个，下次就可以直接使用了。

现在，由于要把这个Binder实体MediaPlayerService交给target_proc，也就是Service Manager来管理，也就是说Service Manager要引用这个MediaPlayerService了，于是通过binder_get_ref_for_node为MediaPlayerService创建一个引用，并且通过binder_inc_ref来增加这个引用计数，防止这个引用还在使用过程当中就被销毁。注意，到了这里的时候，t->buffer中的flat_binder_obj的type已经改为BINDER_TYPE_HANDLE，handle已经改为ref->desc，跟原来不一样了，因为这个flat_binder_obj是最终是要传给Service Manager的，而Service Manager只能够通过句柄值来引用这个Binder实体。

最后，把待处理事务加入到target_list列表中去：

```c
list_add_tail(&t->work.entry, target_list);  
```
并且把待完成工作项加入到本线程的todo等待执行列表中去：

```c
list_add_tail(&tcomplete->entry, &thread->todo);  
```
现在目标进程有事情可做了，于是唤醒它：

```c
if (target_wait)  
    wake_up_interruptible(target_wait);  
```
这里就是要唤醒Service Manager进程了。回忆一下前面

浅谈Service Manager成为Android进程间通信（IPC）机制Binder守护进程之路

这篇文章，此时， Service Manager正在binder_thread_read函数中调用wait_event_interruptible进入休眠状态。

这里我们先忽略一下Service Manager被唤醒之后的场景，继续MedaPlayerService的启动过程，然后再回来。

回到binder_ioctl函数，bwr.read_size > 0为true，于是进入binder_thread_read函数：

```c
static int  
binder_thread_read(struct binder_proc *proc, struct binder_thread *thread,  
           void  __user *buffer, int size, signed long *consumed, int non_block)  
{  
    void __user *ptr = buffer + *consumed;  
    void __user *end = buffer + size;  

    int ret = 0;  
    int wait_for_proc_work;  

    if (*consumed == 0) {  
if (put_user(BR_NOOP, (uint32_t __user *)ptr))  
    return -EFAULT;  
ptr += sizeof(uint32_t);  
    }  

retry:  
    wait_for_proc_work = thread->transaction_stack == NULL && list_empty(&thread->todo);  
      
    .......  

    if (wait_for_proc_work) {  
.......  
    } else {  
if (non_block) {  
    if (!binder_has_thread_work(thread))  
        ret = -EAGAIN;  
} else  
    ret = wait_event_interruptible(thread->wait, binder_has_thread_work(thread));  
    }  
      
    ......  

    while (1) {  
uint32_t cmd;  
struct binder_transaction_data tr;  
struct binder_work *w;  
struct binder_transaction *t = NULL;  

if (!list_empty(&thread->todo))  
    w = list_first_entry(&thread->todo, struct binder_work, entry);  
else if (!list_empty(&proc->todo) && wait_for_proc_work)  
    w = list_first_entry(&proc->todo, struct binder_work, entry);  
else {  
    if (ptr - buffer == 4 && !(thread->looper & BINDER_LOOPER_STATE_NEED_RETURN)) /* no data added */  
        goto retry;  
    break;  
}  

if (end - ptr < sizeof(tr) + 4)  
    break;  

switch (w->type) {  
......  
case BINDER_WORK_TRANSACTION_COMPLETE: {  
    cmd = BR_TRANSACTION_COMPLETE;  
    if (put_user(cmd, (uint32_t __user *)ptr))  
        return -EFAULT;  
    ptr += sizeof(uint32_t);  

    binder_stat_br(proc, thread, cmd);  
    if (binder_debug_mask & BINDER_DEBUG_TRANSACTION_COMPLETE)  
        printk(KERN_INFO "binder: %d:%d BR_TRANSACTION_COMPLETE\n",  
        proc->pid, thread->pid);  

    list_del(&w->entry);  
    kfree(w);  
    binder_stats.obj_deleted[BINDER_STAT_TRANSACTION_COMPLETE]++;  
                                       } break;  
......  
}  

if (!t)  
    continue;  

......  
    }  

done:  
    ......  
    return 0;  
}  
```
这里，thread->transaction_stack和thread->todo均不为空，于是wait_for_proc_work为false，由于binder_has_thread_work的时候，返回true，这里因为thread->todo不为空，因此，线程虽然调用了wait_event_interruptible，但是不会睡眠，于是继续往下执行。

由于thread->todo不为空，执行下列语句：

```c
if (!list_empty(&thread->todo))  
     w = list_first_entry(&thread->todo, struct binder_work, entry);  
```
w->type为BINDER_WORK_TRANSACTION_COMPLETE，这是在上面的binder_transaction函数设置的，于是执行：

```c
   switch (w->type) {  
   ......  
   case BINDER_WORK_TRANSACTION_COMPLETE: {  
   cmd = BR_TRANSACTION_COMPLETE;  
   if (put_user(cmd, (uint32_t __user *)ptr))  
       return -EFAULT;  
   ptr += sizeof(uint32_t);  
   
          ......  
   list_del(&w->entry);  
   kfree(w);  
     
   } break;  
   ......  
   }  
```
这里就将w从thread->todo删除了。由于这里t为空，重新执行while循环，这时由于已经没有事情可做了，最后就返回到binder_ioctl函数中。注间，这里一共往用户传进来的缓冲区buffer写入了两个整数，分别是BR_NOOP和BR_TRANSACTION_COMPLETE。

binder_ioctl函数返回到用户空间之前，把数据消耗情况拷贝回用户空间中：

```c
if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {  
    ret = -EFAULT;  
    goto err;  
}  
```
最后返回到IPCThreadState::talkWithDriver函数中，执行下面语句：

```c
    if (err >= NO_ERROR) {  
if (bwr.write_consumed > 0) {  
    if (bwr.write_consumed < (ssize_t)mOut.dataSize())  
        mOut.remove(0, bwr.write_consumed);  
    else  
        mOut.setDataSize(0);  
}  
if (bwr.read_consumed > 0) {  
<pre code_snippet_id="134056" snippet_file_name="blog_20131230_54_6706870" name="code" class="cpp">            mIn.setDataSize(bwr.read_consumed);  
    mIn.setDataPosition(0);</pre>        }        ......        return NO_ERROR;    }  
```
首先是把mOut的数据清空：

```c
mOut.setDataSize(0);  
```
然后设置已经读取的内容的大小：

```c
mIn.setDataSize(bwr.read_consumed);  
mIn.setDataPosition(0);  
```
然后返回到IPCThreadState::waitForResponse函数中。在IPCThreadState::waitForResponse函数，先是从mIn读出一个整数，这个便是BR_NOOP了，这是一个空操作，什么也不做。然后继续进入IPCThreadState::talkWithDriver函数中。

这时候，下面语句执行后：

```c
const bool needRead = mIn.dataPosition() >= mIn.dataSize();  
```
needRead为false，因为在mIn中，尚有一个整数BR_TRANSACTION_COMPLETE未读出。

这时候，下面语句执行后：

```c
const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;  
```
outAvail等于0。因此，最后bwr.write_size和bwr.read_size均为0，IPCThreadState::talkWithDriver函数什么也不做，直接返回到IPCThreadState::waitForResponse函数中。在IPCThreadState::waitForResponse函数，又继续从mIn读出一个整数，这个便是BR_TRANSACTION_COMPLETE：

```c
switch (cmd) {  
case BR_TRANSACTION_COMPLETE:  
       if (!reply && !acquireResult) goto finish;  
       break;  
......  
}  
```
reply不为NULL，因此，IPCThreadState::waitForResponse的循环没有结束，继续执行，又进入到IPCThreadState::talkWithDrive中。

这次，needRead就为true了，而outAvail仍为0，所以bwr.read_size不为0，bwr.write_size为0。于是通过：

```c
ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)  
```
进入到Binder驱动程序中的binder_ioctl函数中。由于bwr.write_size为0，bwr.read_size不为0，这次直接就进入到binder_thread_read函数中。这时候，thread->transaction_stack不等于0，但是thread->todo为空，于是线程就通过：

```c
wait_event_interruptible(thread->wait, binder_has_thread_work(thread));  
```
进入睡眠状态，等待Service Manager来唤醒了。

现在，我们可以回到Service Manager被唤醒的过程了。我们接着前面[浅谈Service Manager成为Android进程间通信（IPC）机制Binder守护进程之路](http://blog.csdn.net/luoshengyang/article/details/6621566)这篇文章的最后，继续描述。此时， Service Manager正在binder_thread_read函数中调用wait_event_interruptible_exclusive进入休眠状态。上面被MediaPlayerService启动后进程唤醒后，继续执行binder_thread_read函数：

```c
static int  
binder_thread_read(struct binder_proc *proc, struct binder_thread *thread,  
           void  __user *buffer, int size, signed long *consumed, int non_block)  
{  
    void __user *ptr = buffer + *consumed;  
    void __user *end = buffer + size;  

    int ret = 0;  
    int wait_for_proc_work;  

    if (*consumed == 0) {  
if (put_user(BR_NOOP, (uint32_t __user *)ptr))  
    return -EFAULT;  
ptr += sizeof(uint32_t);  
    }  

retry:  
    wait_for_proc_work = thread->transaction_stack == NULL && list_empty(&thread->todo);  

    ......  

    if (wait_for_proc_work) {  
......  
if (non_block) {  
    if (!binder_has_proc_work(proc, thread))  
        ret = -EAGAIN;  
} else  
    ret = wait_event_interruptible_exclusive(proc->wait, binder_has_proc_work(proc, thread));  
    } else {  
......  
    }  
      
    ......  

    while (1) {  
uint32_t cmd;  
struct binder_transaction_data tr;  
struct binder_work *w;  
struct binder_transaction *t = NULL;  

if (!list_empty(&thread->todo))  
    w = list_first_entry(&thread->todo, struct binder_work, entry);  
else if (!list_empty(&proc->todo) && wait_for_proc_work)  
    w = list_first_entry(&proc->todo, struct binder_work, entry);  
else {  
    if (ptr - buffer == 4 && !(thread->looper & BINDER_LOOPER_STATE_NEED_RETURN)) /* no data added */  
        goto retry;  
    break;  
}  

if (end - ptr < sizeof(tr) + 4)  
    break;  

switch (w->type) {  
case BINDER_WORK_TRANSACTION: {  
    t = container_of(w, struct binder_transaction, work);  
                              } break;  
......  
}  

if (!t)  
    continue;  

BUG_ON(t->buffer == NULL);  
if (t->buffer->target_node) {  
    struct binder_node *target_node = t->buffer->target_node;  
    tr.target.ptr = target_node->ptr;  
    tr.cookie =  target_node->cookie;  
    ......  
    cmd = BR_TRANSACTION;  
} else {  
    ......  
}  
tr.code = t->code;  
tr.flags = t->flags;  
tr.sender_euid = t->sender_euid;  

if (t->from) {  
    struct task_struct *sender = t->from->proc->tsk;  
    tr.sender_pid = task_tgid_nr_ns(sender, current->nsproxy->pid_ns);  
} else {  
    tr.sender_pid = 0;  
}  

tr.data_size = t->buffer->data_size;  
tr.offsets_size = t->buffer->offsets_size;  
tr.data.ptr.buffer = (void *)t->buffer->data + proc->user_buffer_offset;  
tr.data.ptr.offsets = tr.data.ptr.buffer + ALIGN(t->buffer->data_size, sizeof(void *));  

if (put_user(cmd, (uint32_t __user *)ptr))  
    return -EFAULT;  
ptr += sizeof(uint32_t);  
if (copy_to_user(ptr, &tr, sizeof(tr)))  
    return -EFAULT;  
ptr += sizeof(tr);  

......  

list_del(&t->work.entry);  
t->buffer->allow_user_free = 1;  
if (cmd == BR_TRANSACTION && !(t->flags & TF_ONE_WAY)) {  
    t->to_parent = thread->transaction_stack;  
    t->to_thread = thread;  
    thread->transaction_stack = t;  
} else {  
    t->buffer->transaction = NULL;  
    kfree(t);  
    binder_stats.obj_deleted[BINDER_STAT_TRANSACTION]++;  
}  
break;  
    }  

done:  

    ......  
    return 0;  
}  
```
Service Manager被唤醒之后，就进入while循环开始处理事务了。这里wait_for_proc_work等于1，并且proc->todo不为空，所以从proc->todo列表中得到第一个工作项：

```c
w = list_first_entry(&proc->todo, struct binder_work, entry);  
```
从上面的描述中，我们知道，这个工作项的类型为BINDER_WORK_TRANSACTION，于是通过下面语句得到事务项：

```c
t = container_of(w, struct binder_transaction, work);  
```
接着就是把事务项t中的数据拷贝到本地局部变量struct binder_transaction_data tr中去了：

```c
if (t->buffer->target_node) {  
    struct binder_node *target_node = t->buffer->target_node;  
    tr.target.ptr = target_node->ptr;  
    tr.cookie =  target_node->cookie;  
    ......  
    cmd = BR_TRANSACTION;  
} else {  
    ......  
}  
tr.code = t->code;  
tr.flags = t->flags;  
tr.sender_euid = t->sender_euid;  

if (t->from) {  
    struct task_struct *sender = t->from->proc->tsk;  
    tr.sender_pid = task_tgid_nr_ns(sender, current->nsproxy->pid_ns);  
} else {  
    tr.sender_pid = 0;  
}  

tr.data_size = t->buffer->data_size;  
tr.offsets_size = t->buffer->offsets_size;  
tr.data.ptr.buffer = (void *)t->buffer->data + proc->user_buffer_offset;  
tr.data.ptr.offsets = tr.data.ptr.buffer + ALIGN(t->buffer->data_size, sizeof(void *));  

这里有一个非常重要的地方，是Binder进程间通信机制的精髓所在：

​```c
tr.data.ptr.buffer = (void *)t->buffer->data + proc->user_buffer_offset;  
tr.data.ptr.offsets = tr.data.ptr.buffer + ALIGN(t->buffer->data_size, sizeof(void *));  
```
t->buffer->data所指向的地址是内核空间的，现在要把数据返回给Service Manager进程的用户空间，而Service Manager进程的用户空间是不能访问内核空间的数据的，所以这里要作一下处理。怎么处理呢？我们在学面向对象语言的时候，对象的拷贝有深拷贝和浅拷贝之分，深拷贝是把另外分配一块新内存，然后把原始对象的内容搬过去，浅拷贝是并没有为新对象分配一块新空间，而只是分配一个引用，而个引用指向原始对象。Binder机制用的是类似浅拷贝的方法，通过在用户空间分配一个虚拟地址，然后让这个用户空间虚拟地址与 t->buffer->data这个内核空间虚拟地址指向同一个物理地址，这样就可以实现浅拷贝了。怎么样用户空间和内核空间的虚拟地址同时指向同一个物理地址呢？请参考前面一篇文章[浅谈Service Manager成为Android进程间通信（IPC）机制Binder守护进程之路]()，那里有详细描述。这里只要将t->buffer->data加上一个偏移值proc->user_buffer_offset就可以得到t->buffer->data对应的用户空间虚拟地址了。调整了tr.data.ptr.buffer的值之后，不要忘记也要一起调整tr.data.ptr.offsets的值。

接着就是把tr的内容拷贝到用户传进来的缓冲区去了，指针ptr指向这个用户缓冲区的地址：

```c
if (put_user(cmd, (uint32_t __user *)ptr))  
    return -EFAULT;  
ptr += sizeof(uint32_t);  
if (copy_to_user(ptr, &tr, sizeof(tr)))  
    return -EFAULT;  
ptr += sizeof(tr);  
```
这里可以看出，这里只是对作tr.data.ptr.bufferr和tr.data.ptr.offsets的内容作了浅拷贝。

最后，由于已经处理了这个事务，要把它从todo列表中删除：

```c
list_del(&t->work.entry);  
t->buffer->allow_user_free = 1;  
if (cmd == BR_TRANSACTION && !(t->flags & TF_ONE_WAY)) {  
    t->to_parent = thread->transaction_stack;  
    t->to_thread = thread;  
    thread->transaction_stack = t;  
} else {  
    t->buffer->transaction = NULL;  
    kfree(t);  
    binder_stats.obj_deleted[BINDER_STAT_TRANSACTION]++;  
}  
```
注意，这里的cmd == BR_TRANSACTION && !(t->flags & TF_ONE_WAY)为true，表明这个事务虽然在驱动程序中已经处理完了，但是它仍然要等待Service Manager完成之后，给驱动程序一个确认，也就是需要等待回复，于是把当前事务t放在thread->transaction_stack队列的头部：

```c
t->to_parent = thread->transaction_stack;  
t->to_thread = thread;  
thread->transaction_stack = t;  
```
如果cmd == BR_TRANSACTION && !(t->flags & TF_ONE_WAY)为false，那就不需要等待回复了，直接把事务t删掉。

这个while最后通过一个break跳了出来，最后返回到binder_ioctl函数中：

```c
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)  
{  
    int ret;  
    struct binder_proc *proc = filp->private_data;  
    struct binder_thread *thread;  
    unsigned int size = _IOC_SIZE(cmd);  
    void __user *ubuf = (void __user *)arg;  

    ......  

    switch (cmd) {  
    case BINDER_WRITE_READ: {  
struct binder_write_read bwr;  
if (size != sizeof(struct binder_write_read)) {  
    ret = -EINVAL;  
    goto err;  
}  
if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {  
    ret = -EFAULT;  
    goto err;  
}  
......  
if (bwr.read_size > 0) {  
    ret = binder_thread_read(proc, thread, (void __user *)bwr.read_buffer, bwr.read_size, &bwr.read_consumed, filp->f_flags & O_NONBLOCK);  
    if (!list_empty(&proc->todo))  
        wake_up_interruptible(&proc->wait);  
    if (ret < 0) {  
        if (copy_to_user(ubuf, &bwr, sizeof(bwr)))  
            ret = -EFAULT;  
        goto err;  
    }  
}  
......  
if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {  
    ret = -EFAULT;  
    goto err;  
}  
break;  
}  
    ......  
    default:  
ret = -EINVAL;  
goto err;  
    }  
    ret = 0;  
err:  
    ......  
    return ret;  
}  
```
从binder_thread_read返回来后，再看看proc->todo是否还有事务等待处理，如果是，就把睡眠在proc->wait队列的线程唤醒来处理。最后，把本地变量struct binder_write_read bwr的内容拷贝回到用户传进来的缓冲区中，就返回了。

这里就是返回到frameworks/base/cmds/servicemanager/binder.c文件中的binder_loop函数了：

```c
void binder_loop(struct binder_state *bs, binder_handler func)  
{  
    int res;  
    struct binder_write_read bwr;  
    unsigned readbuf[32];  

    bwr.write_size = 0;  
    bwr.write_consumed = 0;  
    bwr.write_buffer = 0;  
      
    readbuf[0] = BC_ENTER_LOOPER;  
    binder_write(bs, readbuf, sizeof(unsigned));  

    for (;;) {  
bwr.read_size = sizeof(readbuf);  
bwr.read_consumed = 0;  
bwr.read_buffer = (unsigned) readbuf;  

res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);  

if (res < 0) {  
    LOGE("binder_loop: ioctl failed (%s)\n", strerror(errno));  
    break;  
}  

res = binder_parse(bs, 0, readbuf, bwr.read_consumed, func);  
if (res == 0) {  
    LOGE("binder_loop: unexpected reply?!\n");  
    break;  
}  
if (res < 0) {  
    LOGE("binder_loop: io error %d %s\n", res, strerror(errno));  
    break;  
}  
    }  
}  
```
返回来的数据都放在readbuf中，接着调用binder_parse进行解析：

```c
int binder_parse(struct binder_state *bs, struct binder_io *bio,  
         uint32_t *ptr, uint32_t size, binder_handler func)  
{  
    int r = 1;  
    uint32_t *end = ptr + (size / 4);  

    while (ptr < end) {  
uint32_t cmd = *ptr++;  
......  
case BR_TRANSACTION: {  
    struct binder_txn *txn = (void *) ptr;  
    if ((end - ptr) * sizeof(uint32_t) < sizeof(struct binder_txn)) {  
        LOGE("parse: txn too small!\n");  
        return -1;  
    }  
    binder_dump_txn(txn);  
    if (func) {  
        unsigned rdata[256/4];  
        struct binder_io msg;  
        struct binder_io reply;  
        int res;  

        bio_init(&reply, rdata, sizeof(rdata), 4);  
        bio_init_from_txn(&msg, txn);  
        res = func(bs, txn, &msg, &reply);  
        binder_send_reply(bs, &reply, txn->data, res);  
    }  
    ptr += sizeof(*txn) / sizeof(uint32_t);  
    break;  
                     }  
......  
default:  
    LOGE("parse: OOPS %d\n", cmd);  
    return -1;  
}  
    }  

    return r;  
```

首先把从Binder驱动程序读出来的数据转换为一个struct binder_txn结构体，保存在txn本地变量中，struct binder_txn定义在frameworks/base/cmds/servicemanager/binder.h文件中：

```c
struct binder_txn  
{  
    void *target;  
    void *cookie;  
    uint32_t code;  
    uint32_t flags;  

    uint32_t sender_pid;  
    uint32_t sender_euid;  

    uint32_t data_size;  
    uint32_t offs_size;  
    void *data;  
    void *offs;  
};  
```
函数中还用到了另外一个数据结构struct binder_io，也是定义在frameworks/base/cmds/servicemanager/binder.h文件中：

```c
struct binder_io  
{  
    char *data;            /* pointer to read/write from */  
    uint32_t *offs;        /* array of offsets */  
    uint32_t data_avail;   /* bytes available in data buffer */  
    uint32_t offs_avail;   /* entries available in offsets array */  

    char *data0;           /* start of data buffer */  
    uint32_t *offs0;       /* start of offsets buffer */  
    uint32_t flags;  
    uint32_t unused;  
};  
```
接着往下看，函数调bio_init来初始化reply变量：

```c
void bio_init(struct binder_io *bio, void *data,  
      uint32_t maxdata, uint32_t maxoffs)  
{  
    uint32_t n = maxoffs * sizeof(uint32_t);  

    if (n > maxdata) {  
bio->flags = BIO_F_OVERFLOW;  
bio->data_avail = 0;  
bio->offs_avail = 0;  
return;  
    }  

    bio->data = bio->data0 = data + n;  
    bio->offs = bio->offs0 = data;  
    bio->data_avail = maxdata - n;  
    bio->offs_avail = maxoffs;  
    bio->flags = 0;  
}  
```
接着又调用bio_init_from_txn来初始化msg变量：

```c
void bio_init_from_txn(struct binder_io *bio, struct binder_txn *txn)  
{  
    bio->data = bio->data0 = txn->data;  
    bio->offs = bio->offs0 = txn->offs;  
    bio->data_avail = txn->data_size;  
    bio->offs_avail = txn->offs_size / 4;  
    bio->flags = BIO_F_SHARED;  
}  
```
最后，真正进行处理的函数是从参数中传进来的函数指针func，这里就是定义在frameworks/base/cmds/servicemanager/service_manager.c文件中的svcmgr_handler函数：

```c
int svcmgr_handler(struct binder_state *bs,  
           struct binder_txn *txn,  
           struct binder_io *msg,  
           struct binder_io *reply)  
{  
    struct svcinfo *si;  
    uint16_t *s;  
    unsigned len;  
    void *ptr;  
    uint32_t strict_policy;  

    if (txn->target != svcmgr_handle)  
return -1;  

    // Equivalent to Parcel::enforceInterface(), reading the RPC  
    // header with the strict mode policy mask and the interface name.  
    // Note that we ignore the strict_policy and don't propagate it  
    // further (since we do no outbound RPCs anyway).  
    strict_policy = bio_get_uint32(msg);  
    s = bio_get_string16(msg, &len);  
    if ((len != (sizeof(svcmgr_id) / 2)) ||  
memcmp(svcmgr_id, s, sizeof(svcmgr_id))) {  
    fprintf(stderr,"invalid id %s\n", str8(s));  
    return -1;  
    }  

    switch(txn->code) {  
    ......  
    case SVC_MGR_ADD_SERVICE:  
s = bio_get_string16(msg, &len);  
ptr = bio_get_ref(msg);  
if (do_add_service(bs, s, len, ptr, txn->sender_euid))  
    return -1;  
break;  
    ......  
    }  

    bio_put_uint32(reply, 0);  
    return 0;  
}  
```
回忆一下，在BpServiceManager::addService时，传给Binder驱动程序的参数为：

```c
writeInt32(IPCThreadState::self()->getStrictModePolicy() | STRICT_MODE_PENALTY_GATHER);  
writeString16("android.os.IServiceManager");  
writeString16("media.player");  
writeStrongBinder(new MediaPlayerService());  
```
这里的语句：

```c
strict_policy = bio_get_uint32(msg);  
s = bio_get_string16(msg, &len);  
s = bio_get_string16(msg, &len);  
ptr = bio_get_ref(msg);  
```
就是依次把它们读取出来了，这里，我们只要看一下bio_get_ref的实现。先看一个数据结构struct binder_obj的定义：

```c
struct binder_object  
{  
    uint32_t type;  
    uint32_t flags;  
    void *pointer;  
    void *cookie;  
};  
```
这个结构体其实就是对应struct flat_binder_obj的。

接着看bio_get_ref实现：

```c
void *bio_get_ref(struct binder_io *bio)  
{  
    struct binder_object *obj;  

    obj = _bio_get_obj(bio);  
    if (!obj)  
return 0;  

    if (obj->type == BINDER_TYPE_HANDLE)  
return obj->pointer;  

    return 0;  
}  
```
_bio_get_obj这个函数就不跟进去看了，它的作用就是从binder_io中取得第一个还没取获取过的binder_object。在这个场景下，就是我们最开始传过来代表MediaPlayerService的flat_binder_obj了，这个原始的flat_binder_obj的type为BINDER_TYPE_BINDER，binder为指向MediaPlayerService的弱引用的地址。在前面我们说过，在Binder驱动驱动程序里面，会把这个flat_binder_obj的type改为BINDER_TYPE_HANDLE，handle改为一个句柄值。这里的handle值就等于obj->pointer的值。

回到svcmgr_handler函数，调用do_add_service进一步处理：

```c
int do_add_service(struct binder_state *bs,  
           uint16_t *s, unsigned len,  
           void *ptr, unsigned uid)  
{  
    struct svcinfo *si;  
//    LOGI("add_service('%s',%p) uid=%d\n", str8(s), ptr, uid);  

    if (!ptr || (len == 0) || (len > 127))  
return -1;  

    if (!svc_can_register(uid, s)) {  
LOGE("add_service('%s',%p) uid=%d - PERMISSION DENIED\n",  
     str8(s), ptr, uid);  
return -1;  
    }  

    si = find_svc(s, len);  
    if (si) {  
if (si->ptr) {  
    LOGE("add_service('%s',%p) uid=%d - ALREADY REGISTERED\n",  
         str8(s), ptr, uid);  
    return -1;  
}  
si->ptr = ptr;  
    } else {  
si = malloc(sizeof(*si) + (len + 1) * sizeof(uint16_t));  
if (!si) {  
    LOGE("add_service('%s',%p) uid=%d - OUT OF MEMORY\n",  
         str8(s), ptr, uid);  
    return -1;  
}  
si->ptr = ptr;  
si->len = len;  
memcpy(si->name, s, (len + 1) * sizeof(uint16_t));  
si->name[len] = '\0';  
si->death.func = svcinfo_death;  
si->death.ptr = si;  
si->next = svclist;  
svclist = si;  
    }  

    binder_acquire(bs, ptr);  
    binder_link_to_death(bs, ptr, &si->death);  
    return 0;  
}  
```
这个函数的实现很简单，就是把MediaPlayerService这个Binder实体的引用写到一个struct svcinfo结构体中，主要是它的名称和句柄值，然后插入到链接svclist的头部去。这样，Client来向Service Manager查询服务接口时，只要给定服务名称，Service Manger就可以返回相应的句柄值了。

这个函数执行完成后，返回到svcmgr_handler函数，函数的最后，将一个错误码0写到reply变量中去，表示一切正常：

```c
bio_put_uint32(reply, 0);  
```
svcmgr_handler函数执行完成后，返回到binder_parse函数，执行下面语句：

```c
binder_send_reply(bs, &reply, txn->data, res);  
```
我们看一下binder_send_reply的实现，从函数名就可以猜到它要做什么了，告诉Binder驱动程序，它完成了Binder驱动程序交给它的任务了。

```c
void binder_send_reply(struct binder_state *bs,  
               struct binder_io *reply,  
               void *buffer_to_free,  
               int status)  
{  
    struct {  
uint32_t cmd_free;  
void *buffer;  
uint32_t cmd_reply;  
struct binder_txn txn;  
    } __attribute__((packed)) data;  

    data.cmd_free = BC_FREE_BUFFER;  
    data.buffer = buffer_to_free;  
    data.cmd_reply = BC_REPLY;  
    data.txn.target = 0;  
    data.txn.cookie = 0;  
    data.txn.code = 0;  
    if (status) {  
data.txn.flags = TF_STATUS_CODE;  
data.txn.data_size = sizeof(int);  
data.txn.offs_size = 0;  
data.txn.data = &status;  
data.txn.offs = 0;  
    } else {  
data.txn.flags = 0;  
data.txn.data_size = reply->data - reply->data0;  
data.txn.offs_size = ((char*) reply->offs) - ((char*) reply->offs0);  
data.txn.data = reply->data0;  
data.txn.offs = reply->offs0;  
    }  
    binder_write(bs, &data, sizeof(data));  
}  
```
从这里可以看出，binder_send_reply告诉Binder驱动程序执行BC_FREE_BUFFER和BC_REPLY命令，前者释放之前在binder_transaction分配的空间，地址为buffer_to_free，buffer_to_free这个地址是Binder驱动程序把自己在内核空间用的地址转换成用户空间地址再传给Service Manager的，所以Binder驱动程序拿到这个地址后，知道怎么样释放这个空间；后者告诉MediaPlayerService，它的addService操作已经完成了，错误码是0，保存在data.txn.data中。

再来看binder_write函数：

```c
int binder_write(struct binder_state *bs, void *data, unsigned len)  
{  
    struct binder_write_read bwr;  
    int res;  
    bwr.write_size = len;  
    bwr.write_consumed = 0;  
    bwr.write_buffer = (unsigned) data;  
    bwr.read_size = 0;  
    bwr.read_consumed = 0;  
    bwr.read_buffer = 0;  
    res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);  
    if (res < 0) {  
fprintf(stderr,"binder_write: ioctl failed (%s)\n",  
        strerror(errno));  
    }  
    return res;  
}  
```
这里可以看出，只有写操作，没有读操作，即read_size为0。

这里又是一个ioctl的BINDER_WRITE_READ操作。直入到驱动程序的binder_ioctl函数后，执行BINDER_WRITE_READ命令，这里就不累述了。

最后，从binder_ioctl执行到binder_thread_write函数，我们首先看第一个命令BC_FREE_BUFFER：

```c
int  
binder_thread_write(struct binder_proc *proc, struct binder_thread *thread,  
            void __user *buffer, int size, signed long *consumed)  
{  
    uint32_t cmd;  
    void __user *ptr = buffer + *consumed;  
    void __user *end = buffer + size;  

    while (ptr < end && thread->return_error == BR_OK) {  
if (get_user(cmd, (uint32_t __user *)ptr))  
    return -EFAULT;  
ptr += sizeof(uint32_t);  
if (_IOC_NR(cmd) < ARRAY_SIZE(binder_stats.bc)) {  
    binder_stats.bc[_IOC_NR(cmd)]++;  
    proc->stats.bc[_IOC_NR(cmd)]++;  
    thread->stats.bc[_IOC_NR(cmd)]++;  
}  
switch (cmd) {  
......  
case BC_FREE_BUFFER: {  
    void __user *data_ptr;  
    struct binder_buffer *buffer;  

    if (get_user(data_ptr, (void * __user *)ptr))  
        return -EFAULT;  
    ptr += sizeof(void *);  

    buffer = binder_buffer_lookup(proc, data_ptr);  
    if (buffer == NULL) {  
        binder_user_error("binder: %d:%d "  
            "BC_FREE_BUFFER u%p no match\n",  
            proc->pid, thread->pid, data_ptr);  
        break;  
    }  
    if (!buffer->allow_user_free) {  
        binder_user_error("binder: %d:%d "  
            "BC_FREE_BUFFER u%p matched "  
            "unreturned buffer\n",  
            proc->pid, thread->pid, data_ptr);  
        break;  
    }  
    if (binder_debug_mask & BINDER_DEBUG_FREE_BUFFER)  
        printk(KERN_INFO "binder: %d:%d BC_FREE_BUFFER u%p found buffer %d for %s transaction\n",  
        proc->pid, thread->pid, data_ptr, buffer->debug_id,  
        buffer->transaction ? "active" : "finished");  

    if (buffer->transaction) {  
        buffer->transaction->buffer = NULL;  
        buffer->transaction = NULL;  
    }  
    if (buffer->async_transaction && buffer->target_node) {  
        BUG_ON(!buffer->target_node->has_async_transaction);  
        if (list_empty(&buffer->target_node->async_todo))  
            buffer->target_node->has_async_transaction = 0;  
        else  
            list_move_tail(buffer->target_node->async_todo.next, &thread->todo);  
    }  
    binder_transaction_buffer_release(proc, buffer, NULL);  
    binder_free_buf(proc, buffer);  
    break;  
                     }  

......  
*consumed = ptr - buffer;  
    }  
    return 0;  
}  
```
首先通过看这个语句：

```c
get_user(data_ptr, (void * __user *)ptr)  
```
这个是获得要删除的Buffer的用户空间地址，接着通过下面这个语句来找到这个地址对应的struct binder_buffer信息：

```c
buffer = binder_buffer_lookup(proc, data_ptr);  
```
因为这个空间是前面在binder_transaction里面分配的，所以这里一定能找到。

最后，就可以释放这块内存了：

```c
binder_transaction_buffer_release(proc, buffer, NULL);  
binder_free_buf(proc, buffer);  
```
再来看另外一个命令BC_REPLY：

```c
int  
binder_thread_write(struct binder_proc *proc, struct binder_thread *thread,  
            void __user *buffer, int size, signed long *consumed)  
{  
    uint32_t cmd;  
    void __user *ptr = buffer + *consumed;  
    void __user *end = buffer + size;  

    while (ptr < end && thread->return_error == BR_OK) {  
if (get_user(cmd, (uint32_t __user *)ptr))  
    return -EFAULT;  
ptr += sizeof(uint32_t);  
if (_IOC_NR(cmd) < ARRAY_SIZE(binder_stats.bc)) {  
    binder_stats.bc[_IOC_NR(cmd)]++;  
    proc->stats.bc[_IOC_NR(cmd)]++;  
    thread->stats.bc[_IOC_NR(cmd)]++;  
}  
switch (cmd) {  
......  
case BC_TRANSACTION:  
case BC_REPLY: {  
    struct binder_transaction_data tr;  

    if (copy_from_user(&tr, ptr, sizeof(tr)))  
        return -EFAULT;  
    ptr += sizeof(tr);  
    binder_transaction(proc, thread, &tr, cmd == BC_REPLY);  
    break;  
               }  

......  
*consumed = ptr - buffer;  
    }  
    return 0;  
}  
```
又再次进入到binder_transaction函数：

```c
static void  
binder_transaction(struct binder_proc *proc, struct binder_thread *thread,  
struct binder_transaction_data *tr, int reply)  
{  
    struct binder_transaction *t;  
    struct binder_work *tcomplete;  
    size_t *offp, *off_end;  
    struct binder_proc *target_proc;  
    struct binder_thread *target_thread = NULL;  
    struct binder_node *target_node = NULL;  
    struct list_head *target_list;  
    wait_queue_head_t *target_wait;  
    struct binder_transaction *in_reply_to = NULL;  
    struct binder_transaction_log_entry *e;  
    uint32_t return_error;  

    ......  

    if (reply) {  
in_reply_to = thread->transaction_stack;  
if (in_reply_to == NULL) {  
    ......  
    return_error = BR_FAILED_REPLY;  
    goto err_empty_call_stack;  
}  
binder_set_nice(in_reply_to->saved_priority);  
if (in_reply_to->to_thread != thread) {  
    .......  
    goto err_bad_call_stack;  
}  
thread->transaction_stack = in_reply_to->to_parent;  
target_thread = in_reply_to->from;  
if (target_thread == NULL) {  
    return_error = BR_DEAD_REPLY;  
    goto err_dead_binder;  
}  
if (target_thread->transaction_stack != in_reply_to) {  
    ......  
    return_error = BR_FAILED_REPLY;  
    in_reply_to = NULL;  
    target_thread = NULL;  
    goto err_dead_binder;  
}  
target_proc = target_thread->proc;  
    } else {  
......  
    }  
    if (target_thread) {  
e->to_thread = target_thread->pid;  
target_list = &target_thread->todo;  
target_wait = &target_thread->wait;  
    } else {  
......  
    }  


    /* TODO: reuse incoming transaction for reply */  
    t = kzalloc(sizeof(*t), GFP_KERNEL);  
    if (t == NULL) {  
return_error = BR_FAILED_REPLY;  
goto err_alloc_t_failed;  
    }  
      

    tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);  
    if (tcomplete == NULL) {  
return_error = BR_FAILED_REPLY;  
goto err_alloc_tcomplete_failed;  
    }  

    if (!reply && !(tr->flags & TF_ONE_WAY))  
t->from = thread;  
    else  
t->from = NULL;  
    t->sender_euid = proc->tsk->cred->euid;  
    t->to_proc = target_proc;  
    t->to_thread = target_thread;  
    t->code = tr->code;  
    t->flags = tr->flags;  
    t->priority = task_nice(current);  
    t->buffer = binder_alloc_buf(target_proc, tr->data_size,  
tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));  
    if (t->buffer == NULL) {  
return_error = BR_FAILED_REPLY;  
goto err_binder_alloc_buf_failed;  
    }  
    t->buffer->allow_user_free = 0;  
    t->buffer->debug_id = t->debug_id;  
    t->buffer->transaction = t;  
    t->buffer->target_node = target_node;  
    if (target_node)  
binder_inc_node(target_node, 1, 0, NULL);  

    offp = (size_t *)(t->buffer->data + ALIGN(tr->data_size, sizeof(void *)));  

    if (copy_from_user(t->buffer->data, tr->data.ptr.buffer, tr->data_size)) {  
binder_user_error("binder: %d:%d got transaction with invalid "  
    "data ptr\n", proc->pid, thread->pid);  
return_error = BR_FAILED_REPLY;  
goto err_copy_data_failed;  
    }  
    if (copy_from_user(offp, tr->data.ptr.offsets, tr->offsets_size)) {  
binder_user_error("binder: %d:%d got transaction with invalid "  
    "offsets ptr\n", proc->pid, thread->pid);  
return_error = BR_FAILED_REPLY;  
goto err_copy_data_failed;  
    }  
      
    ......  

    if (reply) {  
BUG_ON(t->buffer->async_transaction != 0);  
binder_pop_transaction(target_thread, in_reply_to);  
    } else if (!(t->flags & TF_ONE_WAY)) {  
......  
    } else {  
......  
    }  
    t->work.type = BINDER_WORK_TRANSACTION;  
    list_add_tail(&t->work.entry, target_list);  
    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;  
    list_add_tail(&tcomplete->entry, &thread->todo);  
    if (target_wait)  
wake_up_interruptible(target_wait);  
    return;  
    ......  
}  
```
注意，这里的reply为1，我们忽略掉其它无关代码。

前面Service Manager正在binder_thread_read函数中被MediaPlayerService启动后进程唤醒后，在最后会把当前处理完的事务放在thread->transaction_stack中：

```c
if (cmd == BR_TRANSACTION && !(t->flags & TF_ONE_WAY)) {  
    t->to_parent = thread->transaction_stack;  
    t->to_thread = thread;  
    thread->transaction_stack = t;  
}   
```
所以，这里，首先是把它这个binder_transaction取回来，并且放在本地变量in_reply_to中：

```c
in_reply_to = thread->transaction_stack;  
```
接着就可以通过in_reply_to得到最终发出这个事务请求的线程和进程：

```c
target_thread = in_reply_to->from;  
target_proc = target_thread->proc;  
```
然后得到target_list和target_wait：

```c
target_list = &target_thread->todo;  
target_wait = &target_thread->wait;  
```
下面这一段代码：

```c
/* TODO: reuse incoming transaction for reply */  
t = kzalloc(sizeof(*t), GFP_KERNEL);  
if (t == NULL) {  
    return_error = BR_FAILED_REPLY;  
    goto err_alloc_t_failed;  
}  


tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);  
if (tcomplete == NULL) {  
    return_error = BR_FAILED_REPLY;  
    goto err_alloc_tcomplete_failed;  
}  

if (!reply && !(tr->flags & TF_ONE_WAY))  
    t->from = thread;  
else  
    t->from = NULL;  
t->sender_euid = proc->tsk->cred->euid;  
t->to_proc = target_proc;  
t->to_thread = target_thread;  
t->code = tr->code;  
t->flags = tr->flags;  
t->priority = task_nice(current);  
t->buffer = binder_alloc_buf(target_proc, tr->data_size,  
    tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));  
if (t->buffer == NULL) {  
    return_error = BR_FAILED_REPLY;  
    goto err_binder_alloc_buf_failed;  
}  
t->buffer->allow_user_free = 0;  
t->buffer->debug_id = t->debug_id;  
t->buffer->transaction = t;  
t->buffer->target_node = target_node;  
if (target_node)  
    binder_inc_node(target_node, 1, 0, NULL);  

offp = (size_t *)(t->buffer->data + ALIGN(tr->data_size, sizeof(void *)));  

if (copy_from_user(t->buffer->data, tr->data.ptr.buffer, tr->data_size)) {  
    binder_user_error("binder: %d:%d got transaction with invalid "  
"data ptr\n", proc->pid, thread->pid);  
    return_error = BR_FAILED_REPLY;  
    goto err_copy_data_failed;  
}  
if (copy_from_user(offp, tr->data.ptr.offsets, tr->offsets_size)) {  
    binder_user_error("binder: %d:%d got transaction with invalid "  
"offsets ptr\n", proc->pid, thread->pid);  
    return_error = BR_FAILED_REPLY;  
    goto err_copy_data_failed;  
}  
```
我们在前面已经分析过了，这里不再重复。但是有一点要注意的是，这里target_node为NULL，因此，t->buffer->target_node也为NULL。

函数本来有一个for循环，用来处理数据中的Binder对象，这里由于没有Binder对象，所以就略过了。到了下面这句代码：

```c
binder_pop_transaction(target_thread, in_reply_to);  
```
我们看看做了什么事情：

```c
static void  
binder_pop_transaction(  
    struct binder_thread *target_thread, struct binder_transaction *t)  
{  
    if (target_thread) {  
BUG_ON(target_thread->transaction_stack != t);  
BUG_ON(target_thread->transaction_stack->from != target_thread);  
target_thread->transaction_stack =  
    target_thread->transaction_stack->from_parent;  
t->from = NULL;  
    }  
    t->need_reply = 0;  
    if (t->buffer)  
t->buffer->transaction = NULL;  
    kfree(t);  
    binder_stats.obj_deleted[BINDER_STAT_TRANSACTION]++;  
}  
```
由于到了这里，已经不需要in_reply_to这个transaction了，就把它删掉。

回到binder_transaction函数：

```c
t->work.type = BINDER_WORK_TRANSACTION;  
list_add_tail(&t->work.entry, target_list);  
tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;  
list_add_tail(&tcomplete->entry, &thread->todo);  
```
和前面一样，分别把t和tcomplete分别放在target_list和thread->todo队列中，这里的target_list指的就是最初调用IServiceManager::addService的MediaPlayerService的Server主线程的的thread->todo队列了，而thread->todo指的是Service Manager中用来回复IServiceManager::addService请求的线程。

最后，唤醒等待在target_wait队列上的线程了，就是最初调用IServiceManager::addService的MediaPlayerService的Server主线程了，它最后在binder_thread_read函数中睡眠在thread->wait上，就是这里的target_wait了：

```c
if (target_wait)  
    wake_up_interruptible(target_wait);  
```
这样，Service Manger回复调用IServiceManager::addService请求就算完成了，重新回到frameworks/base/cmds/servicemanager/binder.c文件中的binder_loop函数等待下一个Client请求的到来。事实上，Service Manger回到binder_loop函数再次执行ioctl函数时候，又会再次进入到binder_thread_read函数。这时个会发现thread->todo不为空，这是因为刚才我们调用了：

```c
list_add_tail(&tcomplete->entry, &thread->todo);  
```
把一个工作项tcompelete放在了在thread->todo中，这个tcompelete的type为BINDER_WORK_TRANSACTION_COMPLETE，因此，Binder驱动程序会执行下面操作：

```c
switch (w->type) {  
case BINDER_WORK_TRANSACTION_COMPLETE: {  
    cmd = BR_TRANSACTION_COMPLETE;  
    if (put_user(cmd, (uint32_t __user *)ptr))  
return -EFAULT;  
    ptr += sizeof(uint32_t);  

    list_del(&w->entry);  
    kfree(w);  
      
    } break;  
    ......  
}  
```
binder_loop函数执行完这个ioctl调用后，才会在下一次调用ioctl进入到Binder驱动程序进入休眠状态，等待下一次Client的请求。

上面讲到调用IServiceManager::addService的MediaPlayerService的Server主线程被唤醒了，于是，重新执行binder_thread_read函数：

```c
static int  
binder_thread_read(struct binder_proc *proc, struct binder_thread *thread,  
           void  __user *buffer, int size, signed long *consumed, int non_block)  
{  
    void __user *ptr = buffer + *consumed;  
    void __user *end = buffer + size;  

    int ret = 0;  
    int wait_for_proc_work;  

    if (*consumed == 0) {  
if (put_user(BR_NOOP, (uint32_t __user *)ptr))  
    return -EFAULT;  
ptr += sizeof(uint32_t);  
    }  

retry:  
    wait_for_proc_work = thread->transaction_stack == NULL && list_empty(&thread->todo);  

    ......  

    if (wait_for_proc_work) {  
......  
    } else {  
if (non_block) {  
    if (!binder_has_thread_work(thread))  
        ret = -EAGAIN;  
} else  
    ret = wait_event_interruptible(thread->wait, binder_has_thread_work(thread));  
    }  
      
    ......  

    while (1) {  
uint32_t cmd;  
struct binder_transaction_data tr;  
struct binder_work *w;  
struct binder_transaction *t = NULL;  

if (!list_empty(&thread->todo))  
    w = list_first_entry(&thread->todo, struct binder_work, entry);  
else if (!list_empty(&proc->todo) && wait_for_proc_work)  
    w = list_first_entry(&proc->todo, struct binder_work, entry);  
else {  
    if (ptr - buffer == 4 && !(thread->looper & BINDER_LOOPER_STATE_NEED_RETURN)) /* no data added */  
        goto retry;  
    break;  
}  

......  

switch (w->type) {  
case BINDER_WORK_TRANSACTION: {  
    t = container_of(w, struct binder_transaction, work);  
                              } break;  
......  
}  

if (!t)  
    continue;  

BUG_ON(t->buffer == NULL);  
if (t->buffer->target_node) {  
    ......  
} else {  
    tr.target.ptr = NULL;  
    tr.cookie = NULL;  
    cmd = BR_REPLY;  
}  
tr.code = t->code;  
tr.flags = t->flags;  
tr.sender_euid = t->sender_euid;  

if (t->from) {  
    ......  
} else {  
    tr.sender_pid = 0;  
}  

tr.data_size = t->buffer->data_size;  
tr.offsets_size = t->buffer->offsets_size;  
tr.data.ptr.buffer = (void *)t->buffer->data + proc->user_buffer_offset;  
tr.data.ptr.offsets = tr.data.ptr.buffer + ALIGN(t->buffer->data_size, sizeof(void *));  

if (put_user(cmd, (uint32_t __user *)ptr))  
    return -EFAULT;  
ptr += sizeof(uint32_t);  
if (copy_to_user(ptr, &tr, sizeof(tr)))  
    return -EFAULT;  
ptr += sizeof(tr);  

......  

list_del(&t->work.entry);  
t->buffer->allow_user_free = 1;  
if (cmd == BR_TRANSACTION && !(t->flags & TF_ONE_WAY)) {  
    ......  
} else {  
    t->buffer->transaction = NULL;  
    kfree(t);  
    binder_stats.obj_deleted[BINDER_STAT_TRANSACTION]++;  
}  
break;  
    }  

done:  
    ......  
    return 0;  
}  
```
在while循环中，从thread->todo得到w，w->type为BINDER_WORK_TRANSACTION，于是，得到t。从上面可以知道，Service Manager反回了一个0回来，写在t->buffer->data里面，现在把t->buffer->data加上proc->user_buffer_offset，得到用户空间地址，保存在tr.data.ptr.buffer里面，这样用户空间就可以访问这个返回码了。由于cmd不等于BR_TRANSACTION，这时就可以把t删除掉了，因为以后都不需要用了。

执行完这个函数后，就返回到binder_ioctl函数，执行下面语句，把数据返回给用户空间：

```c
if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {  
    ret = -EFAULT;  
    goto err;  
}  
```
接着返回到用户空间IPCThreadState::talkWithDriver函数，最后返回到IPCThreadState::waitForResponse函数，最终执行到下面语句：

```c
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)  
{  
    int32_t cmd;  
    int32_t err;  

    while (1) {  
if ((err=talkWithDriver()) < NO_ERROR) break;  
  
......  

cmd = mIn.readInt32();  

......  

switch (cmd) {  
......  
case BR_REPLY:  
    {  
        binder_transaction_data tr;  
        err = mIn.read(&tr, sizeof(tr));  
        LOG_ASSERT(err == NO_ERROR, "Not enough command data for brREPLY");  
        if (err != NO_ERROR) goto finish;  

        if (reply) {  
            if ((tr.flags & TF_STATUS_CODE) == 0) {  
                reply->ipcSetDataReference(  
                    reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),  
                    tr.data_size,  
                    reinterpret_cast<const size_t*>(tr.data.ptr.offsets),  
                    tr.offsets_size/sizeof(size_t),  
                    freeBuffer, this);  
            } else {  
                ......  
            }  
        } else {  
            ......  
        }  
    }  
    goto finish;  

......  
}  
    }  

finish:  
    ......  
    return err;  
}  
```
注意，这里的tr.flags等于0，这个是在上面的binder_send_reply函数里设置的。最终把结果保存在reply了：

```c
reply->ipcSetDataReference(  
       reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),  
       tr.data_size,  
       reinterpret_cast<const size_t*>(tr.data.ptr.offsets),  
       tr.offsets_size/sizeof(size_t),  
       freeBuffer, this);  
```
这个函数我们就不看了，有兴趣的读者可以研究一下。

从这里层层返回，最后回到MediaPlayerService::instantiate函数中。

至此，IServiceManager::addService终于执行完毕了。这个过程非常复杂，但是如果我们能够深刻地理解这一过程，将能很好地理解Binder机制的设计思想和实现过程。这里，对IServiceManager::addService过程中MediaPlayerService、ServiceManager和BinderDriver之间的交互作一个小结：

![img](http://hi.csdn.net/attachment/201107/24/0_1311530322F5jj.gif)

回到frameworks/base/media/mediaserver/main_mediaserver.cpp文件中的main函数，接下去还要执行下面两个函数：

```c
ProcessState::self()->startThreadPool();  
IPCThreadState::self()->joinThreadPool();  
```
首先看ProcessState::startThreadPool函数的实现：

```c
void ProcessState::startThreadPool()  
{  
    AutoMutex _l(mLock);  
    if (!mThreadPoolStarted) {  
mThreadPoolStarted = true;  
spawnPooledThread(true);  
    }  
}  
```
这里调用spwanPooledThread：

```c
void ProcessState::spawnPooledThread(bool isMain)  
{  
    if (mThreadPoolStarted) {  
int32_t s = android_atomic_add(1, &mThreadPoolSeq);  
char buf[32];  
sprintf(buf, "Binder Thread #%d", s);  
LOGV("Spawning new pooled thread, name=%s\n", buf);  
sp<Thread> t = new PoolThread(isMain);  
t->run(buf);  
    }  
}  
```
这里主要是创建一个线程，PoolThread继续Thread类，Thread类定义在frameworks/base/libs/utils/Threads.cpp文件中，其run函数最终调用子类的threadLoop函数，这里即为PoolThread::threadLoop函数：

```c
virtual bool threadLoop()  
{  
    IPCThreadState::self()->joinThreadPool(mIsMain);  
    return false;  
}  
```
这里和frameworks/base/media/mediaserver/main_mediaserver.cpp文件中的main函数一样，最终都是调用了IPCThreadState::joinThreadPool函数，它们的区别是，一个参数是true，一个是默认值false。我们来看一下这个函数的实现：

```c
void IPCThreadState::joinThreadPool(bool isMain)  
{  
    LOG_THREADPOOL("**** THREAD %p (PID %d) IS JOINING THE THREAD POOL\n", (void*)pthread_self(), getpid());  

    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);  

    ......  

    status_t result;  
    do {  
int32_t cmd;  

.......  

// now get the next command to be processed, waiting if necessary  
result = talkWithDriver();  
if (result >= NO_ERROR) {  
    size_t IN = mIn.dataAvail();  
    if (IN < sizeof(int32_t)) continue;  
    cmd = mIn.readInt32();  
    ......  
    }  

    result = executeCommand(cmd);  
}  

......  
    } while (result != -ECONNREFUSED && result != -EBADF);  

    .......  

    mOut.writeInt32(BC_EXIT_LOOPER);  
    talkWithDriver(false);  
```

这个函数最终是在一个无穷循环中，通过调用talkWithDriver函数来和Binder驱动程序进行交互，实际上就是调用talkWithDriver来等待Client的请求，然后再调用executeCommand来处理请求，而在executeCommand函数中，最终会调用BBinder::transact来真正处理Client的请求：

```c
status_t IPCThreadState::executeCommand(int32_t cmd)  
{  
    BBinder* obj;  
    RefBase::weakref_type* refs;  
    status_t result = NO_ERROR;  

    switch (cmd) {  
    ......  

    case BR_TRANSACTION:  
{  
    binder_transaction_data tr;  
    result = mIn.read(&tr, sizeof(tr));  
      
    ......  

    Parcel reply;  
      
    ......  

    if (tr.target.ptr) {  
        sp<BBinder> b((BBinder*)tr.cookie);  
        const status_t error = b->transact(tr.code, buffer, &reply, tr.flags);  
        if (error < NO_ERROR) reply.setError(error);  

    } else {  
        const status_t error = the_context_object->transact(tr.code, buffer, &reply, tr.flags);  
        if (error < NO_ERROR) reply.setError(error);  
    }  

    ......  
}  
break;  

    .......  
    }  

    if (result != NO_ERROR) {  
mLastError = result;  
    }  

    return result;  
}  
```
接下来再看一下BBinder::transact的实现：

```c
status_t BBinder::transact(  
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)  
{  
    data.setDataPosition(0);  

    status_t err = NO_ERROR;  
    switch (code) {  
case PING_TRANSACTION:  
    reply->writeInt32(pingBinder());  
    break;  
default:  
    err = onTransact(code, data, reply, flags);  
    break;  
    }  

    if (reply != NULL) {  
reply->setDataPosition(0);  
    }  

    return err;  
}  
```
最终会调用onTransact函数来处理。在这个场景中，BnMediaPlayerService继承了BBinder类，并且重载了onTransact函数，因此，这里实际上是调用了BnMediaPlayerService::onTransact函数，这个函数定义在frameworks/base/libs/media/libmedia/IMediaPlayerService.cpp文件中：

```c
status_t BnMediaPlayerService::onTransact(  
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)  
{  
    switch(code) {  
case CREATE_URL: {  
    ......  
                 } break;  
case CREATE_FD: {  
    ......  
                } break;  
case DECODE_URL: {  
    ......  
                 } break;  
case DECODE_FD: {  
    ......  
                } break;  
case CREATE_MEDIA_RECORDER: {  
    ......  
                            } break;  
case CREATE_METADATA_RETRIEVER: {  
    ......  
                                } break;  
case GET_OMX: {  
    ......  
              } break;  
default:  
    return BBinder::onTransact(code, data, reply, flags);  
    }  
}  
```
至此，我们就以MediaPlayerService为例，完整地介绍了Android系统进程间通信Binder机制中的Server启动过程。Server启动起来之后，就会在一个无穷循环中等待Client的请求了。在下一篇文章中，我们将介绍Client如何通过Service Manager远程接口来获得Server远程接口，进而调用Server远程接口来使用Server提供的服务，敬请关注