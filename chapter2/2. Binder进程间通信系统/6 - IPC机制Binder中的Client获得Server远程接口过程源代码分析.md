在上一篇文章中，我们分析了[Android](http://lib.csdn.net/base/android)系统进程间通信机制Binder中的Server在启动过程使用Service Manager的addService接口把自己添加到Service Manager守护过程中接受管理。在这一篇文章中，我们将深入到Binder驱动程序源代码去分析Client是如何通过Service Manager的getService接口中来获得Server远程接口的。Client只有获得了Server的远程接口之后，才能进一步调用Server提供的服务。

《[android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

这里，我们仍然是通过Android系统中自带的多媒体播放器为例子来说明Client是如何通过IServiceManager::getService接口来获得MediaPlayerService这个Server的远程接口的。假设计读者已经阅读过前面三篇文章[浅谈Service Manager成为Android进程间通信（IPC）机制Binder守护进程之路](http://blog.csdn.net/luoshengyang/article/details/6621566)、[浅谈Android系统进程间通信（IPC）机制Binder中的Server和Client获得Service Manager接口之路](http://blog.csdn.net/luoshengyang/article/details/6627260)和[Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6629298)，即假设Service Manager和MediaPlayerService已经启动完毕，Service Manager现在等待Client的请求。

这里，我们要举例子说明的Client便是MediaPlayer了，它声明和实现在frameworks/base/include/media/mediaplayer.h和frameworks/base/media/libmedia/mediaplayer.cpp文件中。MediaPlayer继承于IMediaDeathNotifier类，这个类声明和实现在frameworks/base/include/media/IMediaDeathNotifier.h和frameworks/base/media/libmedia//IMediaDeathNotifier.cpp文件中，里面有一个静态成员函数getMeidaPlayerService，它通过IServiceManager::getService接口来获得MediaPlayerService的远程接口。

在介绍IMediaDeathNotifier::getMeidaPlayerService函数之前，我们先了解一下这个函数的目标。看来前面[浅谈Android系统进程间通信（IPC）机制Binder中的Server和Client获得Service Manager接口之路](http://blog.csdn.net/luoshengyang/article/details/6627260)这篇文章的读者知道，我们在获取Service Manager远程接口时，最终是获得了一个BpServiceManager对象的IServiceManager接口。类似地，我们要获得MediaPlayerService的远程接口，实际上就是要获得一个称为BpMediaPlayerService对象的IMediaPlayerService接口。现在，我们就先来看一下BpMediaPlayerService的类图：

![img](http://hi.csdn.net/attachment/201107/26/0_13117045717gMi.gif)

从这个类图可以看到，BpMediaPlayerService继承于BpInterface<IMediaPlayerService>类，即BpMediaPlayerService继承了IMediaPlayerService类和BpRefBase类，这两个类又分别继续了RefBase类。BpRefBase类有一个成员变量mRemote，它的类型为IBinder，实际是一个BpBinder对象。BpBinder类使用了IPCThreadState类来与Binder驱动程序进行交互，而IPCThreadState类有一个成员变量mProcess，它的类型为ProcessState，IPCThreadState类借助ProcessState类来打开Binder设备文件/dev/binder，因此，它可以和Binder驱动程序进行交互。

BpMediaPlayerService的构造函数有一个参数impl，它的类型为const sp<IBinder>&，从上面的描述中，这个实际上就是一个BpBinder对象。这样，要创建一个BpMediaPlayerService对象，首先就要有一个BpBinder对象。再来看BpBinder类的构造函数，它有一个参数handle，类型为int32_t，这个参数的意义就是请求MediaPlayerService这个远程接口的进程对MediaPlayerService这个Binder实体的引用了。因此，获取MediaPlayerService这个远程接口的本质问题就变为从Service Manager中获得MediaPlayerService的一个句柄了。

现在，我们就来看一下IMediaDeathNotifier::getMeidaPlayerService的实现：

```c
// establish binder interface to MediaPlayerService  
/*static*/const sp<IMediaPlayerService>&  
IMediaDeathNotifier::getMediaPlayerService()  
{  
    LOGV("getMediaPlayerService");  
    Mutex::Autolock _l(sServiceLock);  
    if (sMediaPlayerService.get() == 0) {  
sp<IServiceManager> sm = defaultServiceManager();  
sp<IBinder> binder;  
do {  
    binder = sm->getService(String16("media.player"));  
    if (binder != 0) {  
        break;  
     }  
     LOGW("Media player service not published, waiting...");  
     usleep(500000); // 0.5 s  
} while(true);  

if (sDeathNotifier == NULL) {  
sDeathNotifier = new DeathNotifier();  
    }  
    binder->linkToDeath(sDeathNotifier);  
    sMediaPlayerService = interface_cast<IMediaPlayerService>(binder);  
    }  
    LOGE_IF(sMediaPlayerService == 0, "no media player service!?");  
    return sMediaPlayerService;  
}  
```
函数首先通过defaultServiceManager函数来获得Service Manager的远程接口，实际上就是获得BpServiceManager的IServiceManager接口，具体可以参考[浅谈Android系统进程间通信（IPC）机制Binder中的Server和Client获得Service Manager接口之路]()一文。总的来说，这里的语句：

```c
sp<IServiceManager> sm = defaultServiceManager();  
```
相当于是：

```c
sp<IServiceManager> sm = new BpServiceManager(new BpBinder(0));   
```
这里的0表示Service Manager的远程接口的句柄值是0。

接下去的while循环是通过sm->getService接口来不断尝试获得名称为“media.player”的Service，即MediaPlayerService。为什么要通过这无穷循环来得MediaPlayerService呢？因为这时候MediaPlayerService可能还没有启动起来，所以这里如果发现取回来的binder接口为NULL，就睡眠0.5秒，然后再尝试获取，这是获取Service接口的标准做法。
我们来看一下BpServiceManager::getService的实现：

```c
class BpServiceManager : public BpInterface<IServiceManager>  
{  
    ......  

    virtual sp<IBinder> getService(const String16& name) const  
    {  
unsigned n;  
for (n = 0; n < 5; n++){  
    sp<IBinder> svc = checkService(name);  
    if (svc != NULL) return svc;  
    LOGI("Waiting for service %s...\n", String8(name).string());  
    sleep(1);  
}  
return NULL;  
    }  

    virtual sp<IBinder> checkService( const String16& name) const  
    {  
Parcel data, reply;  
data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());  
data.writeString16(name);  
remote()->transact(CHECK_SERVICE_TRANSACTION, data, &reply);  
return reply.readStrongBinder();  
    }  

    ......  
};  
```
BpServiceManager::getService通过BpServiceManager::checkService执行操作。

在BpServiceManager::checkService中，首先是通过Parcel::writeInterfaceToken往data写入一个RPC头，这个我们在[Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6629298)一文已经介绍过了，就是写往data里面写入了一个整数和一个字符串“android.os.IServiceManager”， Service Manager来处理CHECK_SERVICE_TRANSACTION请求之前，会先验证一下这个RPC头，看看是否正确。接着再往data写入一个字符串name，这里就是“media.player”了。回忆一下[Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6629298)这篇文章，那里已经往Service Manager中注册了一个名字为“media.player”的MediaPlayerService。

这里的remote()返回的是一个BpBinder，具体可以参考[浅谈Android系统进程间通信（IPC）机制Binder中的Server和Client获得Service Manager接口之路](http://blog.csdn.net/luoshengyang/article/details/6627260)一文，于是，就进行到BpBinder::transact函数了：

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
这里的mHandle = 0，code = CHECK_SERVICE_TRANSACTION，flags = 0。

这里再进入到IPCThread::transact函数中：

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
首先是调用函数writeTransactionData写入将要传输的数据到IPCThreadState的成员变量mOut中去：

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
结构体binder_transaction_data在上一篇文章[Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析]()已经介绍过，这里不再累述，这个结构体是用来描述要传输的参数的内容的。这里着重描述一下将要传输的参数tr里面的内容，handle = 0，code =  CHECK_SERVICE_TRANSACTION，cmd = BC_TRANSACTION，data里面的数据分别为：

```c
writeInt32(IPCThreadState::self()->getStrictModePolicy() | STRICT_MODE_PENALTY_GATHER);  
writeString16("android.os.IServiceManager");  
writeString16("media.player");  
```
这是在BpServiceManager::checkService函数里面写进去的，其中前两个是RPC头，Service Manager在收到这个请求时会验证这两个参数是否正确，这点前面也提到了。IPCThread->getStrictModePolicy默认返回0，STRICT_MODE_PENALTY_GATHER定义为：

```c
// Note: must be kept in sync with android/os/StrictMode.java's PENALTY_GATHER  
#define STRICT_MODE_PENALTY_GATHER 0x100  
```
我们不关心这个参数的含义，这不会影响我们分析下面的源代码，有兴趣的读者可以研究一下。这里要注意的是，要传输的参数不包含有Binder对象，因此tr.offsets_size = 0。要传输的参数最后写入到IPCThreadState的成员变量mOut中，包括cmd和tr两个数据。

回到IPCThread::transact函数中，由于(flags & TF_ONE_WAY) == 0为true，即这是一个同步请求，并且reply  != NULL，最终调用：

```c
err = waitForResponse(reply);  
```
进入到waitForResponse函数中：

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
这个函数通过IPCThreadState::talkWithDriver与驱动程序进行交互：

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

    ......  

    // Return immediately if there is nothing to do.  
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;  

    bwr.write_consumed = 0;  
    bwr.read_consumed = 0;  
    status_t err;  
    do {  
......  
#if defined(HAVE_ANDROID_OS)  
if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)  
    err = NO_ERROR;  
else  
    err = -errno;  
#else  
err = INVALID_OPERATION;  
#endif  
......  
    } while (err == -EINTR);  

    ......  

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

......  

return NO_ERROR;  
    }  

    return err;  
}  
```
这里的needRead为true，因此，bwr.read_size大于0；outAvail也大于0，因此，bwr.write_size也大于0。函数最后通过：

```c
ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)  
```
进入到Binder驱动程序的binder_ioctl函数中。注意，这里的mProcess->mDriverFD是在我们前面调用defaultServiceManager函数获得Service Manager远程接口时，打开的设备文件/dev/binder的文件描述符，mProcess是IPCSThreadState的成员变量。

Binder驱动程序的binder_ioctl函数中，我们只关注BINDER_WRITE_READ命令相关的逻辑：

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
这里的filp->private_data的值是在defaultServiceManager函数创建ProcessState对象时，在ProcessState构造函数通过open文件操作函数打开设备文件/dev/binder时设置好的，它表示的是调用open函数打开设备文件/dev/binder的进程上下文信息，这里将它取出来保存在proc本地变量中。

这里的thread本地变量表示当前线程上下文信息，通过binder_get_thread函数获得。在前面执行ProcessState构造函数时，也会通过ioctl文件操作函数进入到这个函数，那是第一次进入到binder_ioctl这里，因此，调用binder_get_thread时，表示当前进程上下文信息的proc变量还没有关于当前线程的上下文信息，因此，会为proc创建一个表示当前线程上下文信息的thread，会保存在proc->threads表示的红黑树结构中。这里调用binder_get_thread就可以直接从proc找到并返回了。

进入到BINDER_WRITE_READ相关的逻辑。先看看BINDER_WRITE_READ的定义：

```c
#define BINDER_WRITE_READ           _IOWR('b', 1, struct binder_write_read)  
```
这里可以看出，BINDER_WRITE_READ命令的参数类型为struct binder_write_read：

```c
struct binder_write_read {  
    signed long write_size; /* bytes to write */  
    signed long write_consumed; /* bytes consumed by driver */  
    unsigned long   write_buffer;  
    signed long read_size;  /* bytes to read */  
    signed long read_consumed;  /* bytes consumed by driver */  
    unsigned long   read_buffer;  
};  
```
这个结构体的含义可以参考[浅谈Service Manager成为Android进程间通信（IPC）机制Binder守护进程之路]()一文。这里首先是通过copy_from_user函数把用户传进来的参数的内容拷贝到本地变量bwr中。

从上面的调用过程，我们知道，这里bwr.write_size是大于0的，因此进入到binder_thread_write函数中，我们只关注BC_TRANSACTION相关的逻辑：

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
default:  
    printk(KERN_ERR "binder: %d:%d unknown command %d\n", proc->pid, thread->pid, cmd);  
    return -EINVAL;  
}  
*consumed = ptr - buffer;  
    }  
    return 0;  
}  
```
这里再次把用户传出来的参数拷贝到本地变量tr中，tr的类型为struct binder_transaction_data，这个就是前面我们在IPCThreadState::writeTransactionData写入的内容了。

接着进入到binder_transaction函数中，不相关的代码我们忽略掉：

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

    .......  

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
if (!(tr->flags & TF_ONE_WAY) && thread->transaction_stack) {  
    ......  
}  
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
    binder_stats.obj_created[BINDER_STAT_TRANSACTION]++;  

    tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);  
    if (tcomplete == NULL) {  
return_error = BR_FAILED_REPLY;  
goto err_alloc_tcomplete_failed;  
    }  
    binder_stats.obj_created[BINDER_STAT_TRANSACTION_COMPLETE]++;  

    t->debug_id = ++binder_last_id;  

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

    ......  

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
注意，这里的参数reply = 0，表示这是一个BC_TRANSACTION命令。

前面我们提到，传给驱动程序的handle值为0，即这里的tr->target.handle = 0，表示请求的目标Binder对象是Service Manager，因此有：

```c
target_node = binder_context_mgr_node;  
target_proc = target_node->proc;  
target_list = &target_proc->todo;  
target_wait = &target_proc->wait;  
```
其中binder_context_mgr_node是在Service Manager通知Binder驱动程序它是守护过程时创建的。

接着创建一个待完成事项tcomplete，它的类型为struct binder_work，这是等一会要保存在当前线程的todo队列去的，表示当前线程有一个待完成的事务。紧跟着创建一个待处理事务t，它的类型为struct binder_transaction，这是等一会要存在到Service Manager的todo队列去的，表示Service Manager当前有一个事务需要处理。同时，这个待处理事务t也要存放在当前线程的待完成事务transaction_stack列表中去：

```c
t->from_parent = thread->transaction_stack;  
thread->transaction_stack = t;  
```
这样表明当前线程还有事务要处理。

继续往下看，就是分别把tcomplete和t放在当前线程thread和Service Manager进程的todo队列去了：

```c
t->work.type = BINDER_WORK_TRANSACTION;  
list_add_tail(&t->work.entry, target_list);  
tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;  
list_add_tail(&tcomplete->entry, &thread->todo);  
```
最后，Service Manager有事情可做了，就要唤醒它了：

```c
wake_up_interruptible(target_wait);  
```
前面我们提到，此时Service Manager正在等待Client的请求，也就是Service Manager此时正在进入到Binder驱动程序的binder_thread_read函数中，并且休眠在target->wait上，具体参考[浅谈Service Manager成为Android进程间通信（IPC）机制Binder守护进程之路]()一文。

这里，我们暂时忽略Service Manager被唤醒之后的情景，继续看当前线程的执行。

函数binder_transaction执行完成之后，就一路返回到binder_ioctl函数里去了。函数binder_ioctl从binder_thread_write函数调用处返回后，发现bwr.read_size大于0，于是就进入到binder_thread_read函数去了：

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
函数首先是写入一个操作码BR_NOOP到用户传进来的缓冲区中去。

回忆一下上面的binder_transaction函数，这里的thread->transaction_stack != NULL，并且thread->todo也不为空，所以线程不会进入休眠状态。

进入while循环中，首先是从thread->todo队列中取回待处理事项w，w的类型为BINDER_WORK_TRANSACTION_COMPLETE，这也是在binder_transaction函数里面设置的。对BINDER_WORK_TRANSACTION_COMPLETE的处理也很简单，只是把一个操作码BR_TRANSACTION_COMPLETE写回到用户传进来的缓冲区中去。这时候，用户传进来的缓冲区就包含两个操作码了，分别是BR_NOOP和BINDER_WORK_TRANSACTION_COMPLETE。

binder_thread_read执行完之后，返回到binder_ioctl函数中，将操作结果写回到用户空间中去：

```c
if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {  
    ret = -EFAULT;  
    goto err;  
}  
```
最后就返回到IPCThreadState::talkWithDriver函数中了。

IPCThreadState::talkWithDriver函数从下面语句：

```c
ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)  
```
返回后，首先是清空之前写入Binder驱动程序的内容：

```c
if (bwr.write_consumed > 0) {  
     if (bwr.write_consumed < (ssize_t)mOut.dataSize())  
  mOut.remove(0, bwr.write_consumed);  
     else  
  mOut.setDataSize(0);  
}  

 接着是设置从Binder驱动程序读取的内容：

​```c
if (bwr.read_consumed > 0) {  
     mIn.setDataSize(bwr.read_consumed);  
     mIn.setDataPosition(0);  
}  
```
然后就返回到IPCThreadState::waitForResponse去了。IPCThreadState::waitForResponse函数的处理也很简单，就是处理刚才从Binder驱动程序读入内容了。从前面的分析中，我们知道，从Binder驱动程序读入的内容就是两个整数了，分别是BR_NOOP和BR_TRANSACTION_COMPLETE。对BR_NOOP的处理很简单，正如它的名字所示，什么也不做；而对BR_TRANSACTION_COMPLETE的处理，就分情况了，如果这个请求是异步的，那个整个BC_TRANSACTION操作就完成了，如果这个请求是同步的，即要等待回复的，也就是reply不为空，那么还要继续通过IPCThreadState::talkWithDriver进入到Binder驱动程序中去等待BC_TRANSACTION操作的处理结果。

这里属于后一种情况，于是再次通过IPCThreadState::talkWithDriver进入到Binder驱动程序的binder_ioctl函数中。不过这一次在binder_ioctl函数中，bwr.write_size等于0，而bwr.read_size大于0，于是再次进入到binder_thread_read函数中。这时候thread->transaction_stack仍然不为NULL，不过thread->todo队列已经为空了，因为前面我们已经处理过thread->todo队列的内容了，于是就通过下面语句：

```c
ret = wait_event_interruptible(thread->wait, binder_has_thread_work(thread));  
```
进入休眠状态了，等待Service Manager的唤醒。

现在，我们终于可以回到Service Manager被唤醒之后的过程了。前面我们说过，Service Manager此时正在binder_thread_read函数中休眠中：

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
    t->saved_priority = task_nice(current);  
    if (t->priority < target_node->min_priority &&  
        !(t->flags & TF_ONE_WAY))  
        binder_set_nice(t->priority);  
    else if (!(t->flags & TF_ONE_WAY) ||  
        t->saved_priority > target_node->min_priority)  
        binder_set_nice(target_node->min_priority);  
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
    ......  
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
    ......  
}  
break;  
    }  

done:  

    *consumed = ptr - buffer;  
    ......  
    return 0;  
}  
```
这里就是从语句中唤醒了：

```c
ret = wait_event_interruptible_exclusive(proc->wait, binder_has_proc_work(proc, thread));  
```
Service Manager唤醒过来看，继续往下执行，进入到while循环中。首先是从proc->todo中取回待处理事项w。这个事项w的类型是BINDER_WORK_TRANSACTION，这是上面调用binder_transaction的时候设置的，于是通过w得到待处理事务t：

```c
t = container_of(w, struct binder_transaction, work);  
```
接下来的内容，就把cmd和t->buffer的内容拷贝到用户传进来的缓冲区去了，这里就是Service Manager从用户空间传进来的缓冲区了：

```c
if (put_user(cmd, (uint32_t __user *)ptr))  
    return -EFAULT;  
ptr += sizeof(uint32_t);  
if (copy_to_user(ptr, &tr, sizeof(tr)))  
    return -EFAULT;  
ptr += sizeof(tr);  
```
注意，这里先是把t->buffer的内容拷贝到本地变量tr中，再拷贝到用户空间缓冲区去。关于t->buffer内容的拷贝，请参考[Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析]()一文，它的一个关键地方是Binder驱动程序和Service Manager守护进程共享了同一个物理内存的内容，拷贝的只是这个物理内存在用户空间的虚拟地址回去：

```c
tr.data.ptr.buffer = (void *)t->buffer->data + proc->user_buffer_offset;  
tr.data.ptr.offsets = tr.data.ptr.buffer + ALIGN(t->buffer->data_size, sizeof(void *));  
```
对于Binder驱动程序这次操作来说，这个事项就算是处理完了，就要从todo队列中删除了：

```c
list_del(&t->work.entry);  
```
紧接着，还不放删除这个事务，因为它还要等待Service Manager处理完成后，再进一步处理，因此，放在thread->transaction_stack队列中：

```c
t->to_parent = thread->transaction_stack;  
t->to_thread = thread;  
thread->transaction_stack = t;  
```
还要注意的一个地方是，上面写入的cmd = BR_TRANSACTION，告诉Service Manager守护进程，它要做什么事情，后面我们会看到相应的分析。

这样，binder_thread_read函数就处理完了，回到binder_ioctl函数中，同样是操作结果写回到用户空间的缓冲区中去：

```c
if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {  
    ret = -EFAULT;  
    goto err;  
}  
```
最后，就返回到frameworks/base/cmds/servicemanager/binder.c文件中的binder_loop函数去了：

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
这里就是从下面的语句：

```c
res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);  
```
返回来了。接着就进入binder_parse函数处理从Binder驱动程序里面读取出来的数据：

```c
int binder_parse(struct binder_state *bs, struct binder_io *bio,  
         uint32_t *ptr, uint32_t size, binder_handler func)  
{  
    int r = 1;  
    uint32_t *end = ptr + (size / 4);  

    while (ptr < end) {  
uint32_t cmd = *ptr++;  
switch(cmd) {  
......  
case BR_TRANSACTION: {  
    struct binder_txn *txn = (void *) ptr;  
    ......  
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
}  
```
前面我们说过，Binder驱动程序写入到用户空间的缓冲区中的cmd为BR_TRANSACTION，因此，这里我们只关注BR_TRANSACTION相关的逻辑。

这里用到的两个[数据结构](http://lib.csdn.net/base/datastructure)struct binder_txn和struct binder_io可以参考前面一篇文章[Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6629298)，这里就不复述了。

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

//    LOGI("target=%p code=%d pid=%d uid=%d\n",  
//         txn->target, txn->code, txn->sender_pid, txn->sender_euid);  

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
    case SVC_MGR_GET_SERVICE:  
    case SVC_MGR_CHECK_SERVICE:  
s = bio_get_string16(msg, &len);  
ptr = do_find_service(bs, s, len);  
if (!ptr)  
    break;  
bio_put_ref(reply, ptr);  
return 0;  

    ......  
    }  
    default:  
LOGE("unknown code %d\n", txn->code);  
return -1;  
    }  

    bio_put_uint32(reply, 0);  
    return 0;  
```

这里， Service Manager要处理的code是SVC_MGR_CHECK_SERVICE，这是在前面的BpServiceManager::checkService函数里面设置的。

回忆一下，在BpServiceManager::checkService时，传给Binder驱动程序的参数为：

```c
writeInt32(IPCThreadState::self()->getStrictModePolicy() | STRICT_MODE_PENALTY_GATHER);    
writeString16("android.os.IServiceManager");    
writeString16("media.player");    
```
这里的语句：

```c
strict_policy = bio_get_uint32(msg);    
s = bio_get_string16(msg, &len);    
s = bio_get_string16(msg, &len);   
```
其中，会验证一下传进来的第二个参数，即"android.os.IServiceManager"是否正确，这个是验证RPC头，注释已经说得很清楚了。

最后，就是调用do_find_service函数查找是存在名称为"media.player"的服务了。回忆一下前面一篇文章[Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6629298)，MediaPlayerService已经把一个名称为"media.player"的服务注册到Service Manager中，所以这里一定能找到。我们看看do_find_service这个函数：

```c
void *do_find_service(struct binder_state *bs, uint16_t *s, unsigned len)  
{  
    struct svcinfo *si;  
    si = find_svc(s, len);  

//    LOGI("check_service('%s') ptr = %p\n", str8(s), si ? si->ptr : 0);  
    if (si && si->ptr) {  
return si->ptr;  
    } else {  
return 0;  
    }  
}  
```
这里又调用了find_svc函数：

```c
struct svcinfo *find_svc(uint16_t *s16, unsigned len)  
{  
    struct svcinfo *si;  

    for (si = svclist; si; si = si->next) {  
if ((len == si->len) &&  
    !memcmp(s16, si->name, len * sizeof(uint16_t))) {  
    return si;  
}  
    }  
    return 0;  
}  
```
就是在svclist列表中查找对应名称的svcinfo了。

然后返回到do_find_service函数中。回忆一下前面一篇文章[Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6629298)，这里的si->ptr就是指MediaPlayerService这个Binder实体在Service Manager进程中的句柄值了。

回到svcmgr_handler函数中，调用bio_put_ref函数将这个Binder引用写回到reply参数。我们看看bio_put_ref的实现：

```c
void bio_put_ref(struct binder_io *bio, void *ptr)  
{  
    struct binder_object *obj;  

    if (ptr)  
obj = bio_alloc_obj(bio);  
    else  
obj = bio_alloc(bio, sizeof(*obj));  

    if (!obj)  
return;  

    obj->flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;  
    obj->type = BINDER_TYPE_HANDLE;  
    obj->pointer = ptr;  
    obj->cookie = 0;  
}  
```
这里很简单，就是把一个类型为BINDER_TYPE_HANDLE的binder_object写入到reply缓冲区中去。这里的binder_object就是相当于是flat_binder_obj了，具体可以参考[Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析]()一文。

再回到svcmgr_handler函数中，最后，还写入一个0值到reply缓冲区中，表示操作结果码：

```c
bio_put_uint32(reply, 0);  
```
最后返回到binder_parse函数中，调用binder_send_reply函数将操作结果反馈给Binder驱动程序：

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
注意，这里的status参数为0。从这里可以看出，binder_send_reply告诉Binder驱动程序执行BC_FREE_BUFFER和BC_REPLY命令，前者释放之前在binder_transaction分配的空间，地址为buffer_to_free，buffer_to_free这个地址是Binder驱动程序把自己在内核空间用的地址转换成用户空间地址再传给Service Manager的，所以Binder驱动程序拿到这个地址后，知道怎么样释放这个空间；后者告诉Binder驱动程序，它的SVC_MGR_CHECK_SERVICE操作已经完成了,要查询的服务的句柄值也是保存在data.txn.data，操作结果码是0，也是保存在data.txn.data中。

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

最后，从binder_ioctl执行到binder_thread_write函数，首先是执行BC_FREE_BUFFER命令，这个命令的执行在前面一篇文章[Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析]()已经介绍过了，这里就不再累述了。

我们重点关注BC_REPLY命令的执行：

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
......  
thread->transaction_stack = in_reply_to->to_parent;  
target_thread = in_reply_to->from;  
......  
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
    binder_stats.obj_created[BINDER_STAT_TRANSACTION]++;  

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

    off_end = (void *)offp + tr->offsets_size;  
    for (; offp < off_end; offp++) {  
struct flat_binder_object *fp;  
......  
fp = (struct flat_binder_object *)(t->buffer->data + *offp);  
switch (fp->type) {  
......  
case BINDER_TYPE_HANDLE:  
case BINDER_TYPE_WEAK_HANDLE: {  
    struct binder_ref *ref = binder_get_ref(proc, fp->handle);  
    if (ref == NULL) {  
        ......  
        return_error = BR_FAILED_REPLY;  
        goto err_binder_get_ref_failed;  
    }  
    if (ref->node->proc == target_proc) {  
        ......  
    } else {  
        struct binder_ref *new_ref;  
        new_ref = binder_get_ref_for_node(target_proc, ref->node);  
        if (new_ref == NULL) {  
            return_error = BR_FAILED_REPLY;  
            goto err_binder_get_ref_for_node_failed;  
        }  
        fp->handle = new_ref->desc;  
        binder_inc_ref(new_ref, fp->type == BINDER_TYPE_HANDLE, NULL);  
        ......  
    }  
} break;  

......  
}  
    }  

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
这次进入binder_transaction函数的情形和上面介绍的binder_transaction函数的情形基本一致，只是这里的proc、thread和target_proc、target_thread调换了角色，这里的proc和thread指的是Service Manager进程，而target_proc和target_thread指的是刚才请求SVC_MGR_CHECK_SERVICE的进程。

那么，这次是如何找到target_proc和target_thread呢。首先，我们注意到，这里的reply等于1，其次，上面我们提到，Binder驱动程序在唤醒Service Manager，告诉它有一个事务t要处理时，事务t虽然从Service Manager的todo队列中删除了，但是仍然保留在transaction_stack中。因此，这里可以从thread->transaction_stack找回这个等待回复的事务t，然后通过它找回target_proc和target_thread：

```c
in_reply_to = thread->transaction_stack;  
target_thread = in_reply_to->from;  
target_list = &target_thread->todo;  
target_wait = &target_thread->wait;  
```
再接着往下看，由于Service Manager返回来了一个Binder引用，所以这里要处理一下，就是中间的for循环了。这是一个BINDER_TYPE_HANDLE类型的Binder引用，这是前面设置的。先把t->buffer->data的内容转换为一个struct flat_binder_object对象fp，这里的fp->handle值就是这个Service在Service Manager进程里面的引用值了。接通过调用binder_get_ref函数得到Binder引用对象struct binder_ref类型的对象ref：

```c
struct binder_ref *ref = binder_get_ref(proc, fp->handle);  
```
这里一定能找到，因为前面MediaPlayerService执行IServiceManager::addService的时候把自己添加到Service Manager的时候，会在Service Manager进程中创建这个Binder引用，然后把这个Binder引用的句柄值返回给Service Manager用户空间。

这里面的ref->node->proc不等于target_proc，因为这个Binder实体是属于创建MediaPlayerService的进程的，而不是请求这个服务的远程接口的进程的，因此，这里调用binder_get_ref_for_node函数为这个Binder实体在target_proc创建一个引用：

```c
struct binder_ref *new_ref;  
new_ref = binder_get_ref_for_node(target_proc, ref->node);  
```
然后增加引用计数：

```c
binder_inc_ref(new_ref, fp->type == BINDER_TYPE_HANDLE, NULL);  
```
这样，返回数据中的Binder对象就处理完成了。注意，这里会把fp->handle的值改为在target_proc中的引用值：

```c
fp->handle = new_ref->desc;  
```
这里就相当于是把t->buffer->data里面的Binder对象的句柄值改写了。因为这是在另外一个不同的进程里面的Binder引用，所以句柄值当然要用新的了。这个值最终是要拷贝回target_proc进程的用户空间去的。

再往下看：

```c
if (reply) {  
     BUG_ON(t->buffer->async_transaction != 0);  
     binder_pop_transaction(target_thread, in_reply_to);  
} else if (!(t->flags & TF_ONE_WAY)) {  
     ......  
} else {  
     ......  
}  
```
这里reply等于1，执行binder_pop_transaction函数把当前事务in_reply_to从target_thread->transaction_stack队列中删掉，这是上次调用binder_transaction函数的时候设置的，现在不需要了，所以把它删掉。

再往后的逻辑就跟前面执行binder_transaction函数时候一样了，这里不再介绍。最后的结果就是唤醒请求SVC_MGR_CHECK_SERVICE操作的线程：

```c
if (target_wait)  
     wake_up_interruptible(target_wait);  
```
这样，Service Manger回复调用SVC_MGR_CHECK_SERVICE请求就算完成了，重新回到frameworks/base/cmds/servicemanager/binder.c文件中的binder_loop函数等待下一个Client请求的到来。事实上，Service Manger回到binder_loop函数再次执行ioctl函数时候，又会再次进入到binder_thread_read函数。这时个会发现thread->todo不为空，这是因为刚才我们调用了：

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

上面讲到调用请求SVC_MGR_CHECK_SERVICE操作的线程被唤醒了，于是，重新执行binder_thread_read函数：

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
就是从下面这个调用：

```c
ret = wait_event_interruptible(thread->wait, binder_has_thread_work(thread));  
```
被唤醒过来了。在while循环中，从thread->todo得到w，w->type为BINDER_WORK_TRANSACTION，于是，得到t。从上面可以知道，Service Manager返回来了一个Binder引用和一个结果码0回来，写在t->buffer->data里面，现在把t->buffer->data加上proc->user_buffer_offset，得到用户空间地址，保存在tr.data.ptr.buffer里面，这样用户空间就可以访问这个数据了。由于cmd不等于BR_TRANSACTION，这时就可以把t删除掉了，因为以后都不需要用了。

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
注意，这里的tr.flags等于0，这个是在上面的binder_send_reply函数里设置的。接着就把结果保存在reply了：

```c
reply->ipcSetDataReference(    
 reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),    
 tr.data_size,    
 reinterpret_cast<const size_t*>(tr.data.ptr.offsets),    
 tr.offsets_size/sizeof(size_t),    
 freeBuffer, this);    
```
我们简单看一下Parcel::ipcSetDataReference函数的实现：

```c
void Parcel::ipcSetDataReference(const uint8_t* data, size_t dataSize,  
    const size_t* objects, size_t objectsCount, release_func relFunc, void* relCookie)  
{  
    freeDataNoInit();  
    mError = NO_ERROR;  
    mData = const_cast<uint8_t*>(data);  
    mDataSize = mDataCapacity = dataSize;  
    //LOGI("setDataReference Setting data size of %p to %lu (pid=%d)\n", this, mDataSize, getpid());  
    mDataPos = 0;  
    LOGV("setDataReference Setting data pos of %p to %d\n", this, mDataPos);  
    mObjects = const_cast<size_t*>(objects);  
    mObjectsSize = mObjectsCapacity = objectsCount;  
    mNextObjectHint = 0;  
    mOwner = relFunc;  
    mOwnerCookie = relCookie;  
    scanForFds();  
}  
```
上面提到，返回来的数据中有一个Binder引用，因此，这里的mObjectSize等于1，这个Binder引用对应的位置记录在mObjects成员变量中。

从这里层层返回，最后回到BpServiceManager::checkService函数中：

```c
virtual sp<IBinder> BpServiceManager::checkService( const String16& name) const  
{  
    Parcel data, reply;  
    data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());  
    data.writeString16(name);  
    remote()->transact(CHECK_SERVICE_TRANSACTION, data, &reply);  
    return reply.readStrongBinder();  
}  
```
这里就是从：

```c
remote()->transact(CHECK_SERVICE_TRANSACTION, data, &reply);  
```
返回来了。我们接着看一下reply.readStrongBinder函数的实现：

```c
sp<IBinder> Parcel::readStrongBinder() const  
{  
    sp<IBinder> val;  
    unflatten_binder(ProcessState::self(), *this, &val);  
    return val;  
}  
```
这里调用了unflatten_binder函数来构造一个Binder对象：

```c
status_t unflatten_binder(const sp<ProcessState>& proc,  
    const Parcel& in, sp<IBinder>* out)  
{  
    const flat_binder_object* flat = in.readObject(false);  

    if (flat) {  
switch (flat->type) {  
    case BINDER_TYPE_BINDER:  
        *out = static_cast<IBinder*>(flat->cookie);  
        return finish_unflatten_binder(NULL, *flat, in);  
    case BINDER_TYPE_HANDLE:  
        *out = proc->getStrongProxyForHandle(flat->handle);  
        return finish_unflatten_binder(  
            static_cast<BpBinder*>(out->get()), *flat, in);  
}          
    }  
    return BAD_TYPE;  
}  
```
这里的flat->type是BINDER_TYPE_HANDLE，因此调用ProcessState::getStrongProxyForHandle函数：

```c
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)  
{  
    sp<IBinder> result;  

    AutoMutex _l(mLock);  

    handle_entry* e = lookupHandleLocked(handle);  

    if (e != NULL) {  
// We need to create a new BpBinder if there isn't currently one, OR we  
// are unable to acquire a weak reference on this current one.  See comment  
// in getWeakProxyForHandle() for more info about this.  
IBinder* b = e->binder;  
if (b == NULL || !e->refs->attemptIncWeak(this)) {  
    b = new BpBinder(handle);   
    e->binder = b;  
    if (b) e->refs = b->getWeakRefs();  
    result = b;  
} else {  
    // This little bit of nastyness is to allow us to add a primary  
    // reference to the remote proxy when this team doesn't have one  
    // but another team is sending the handle to us.  
    result.force_set(b);  
    e->refs->decWeak(this);  
}  
    }  

    return result;  
}  
```
这里我们可以看到，ProcessState会把使用过的Binder远程接口（BpBinder）缓存起来，这样下次从Service Manager那里请求得到相同的句柄（Handle）时就可以直接返回这个Binder远程接口了，不用再创建一个出来。这里是第一次使用，因此，e->binder为空，于是创建了一个BpBinder对象：

```c
b = new BpBinder(handle);   
e->binder = b;  
if (b) e->refs = b->getWeakRefs();  
result = b;  
```
最后，函数返回到IMediaDeathNotifier::getMediaPlayerService这里，从这个语句返回：

```c
binder = sm->getService(String16("media.player"));  
```
这里，就相当于是：

```c
binder = new BpBinder(handle);  
```
最后，函数调用：

```c
sMediaPlayerService = interface_cast<IMediaPlayerService>(binder);  
```
到了这里，我们可以参考一下前面一篇文章[浅谈Android系统进程间通信（IPC）机制Binder中的Server和Client获得Service Manager]()，就会知道，这里的interface_cast实际上最终调用了IMediaPlayerService::asInterface函数：

```c
android::sp<IMediaPlayerService> IMediaPlayerService::asInterface(const android::sp<android::IBinder>& obj)  
{  
    android::sp<IServiceManager> intr;  
    if (obj != NULL) {               
intr = static_cast<IMediaPlayerService*>(   
    obj->queryLocalInterface(IMediaPlayerService::descriptor).get());  
if (intr == NULL) {  
    intr = new BpMediaPlayerService(obj);  
}  
    }  
    return intr;   
}  
```
这里的obj就是BpBinder，而BpBinder::queryLocalInterface返回NULL，因此就创建了一个BpMediaPlayerService对象：

```c
intr = new BpMediaPlayerService(new BpBinder(handle));  
```
因此，我们最终就得到了一个BpMediaPlayerService对象，达到我们最初的目标。

有了这个BpMediaPlayerService这个远程接口之后，MediaPlayer就可以调用MediaPlayerService的服务了。

至此，Android系统进程间通信（IPC）机制Binder中的Client如何通过Service Manager的getService函数获得Server远程接口的过程就分析完了，Binder机制的学习就暂告一段落了。

不过，细心的读者可能会发现，我们这里介绍的Binder机制都是基于C/C++语言实现的，但是我们在编写应用程序都是基于[Java](http://lib.csdn.net/base/java)语言的，那么，我们如何使用Java语言来使用系统的Binder机制来进行进程间通信呢？这就是下一篇文章要介绍的内容了，敬请关注。