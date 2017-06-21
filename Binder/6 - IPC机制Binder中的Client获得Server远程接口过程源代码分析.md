在上一篇文章中，我们分析了[Android](http://lib.csdn.net/base/android)系统进程间通信机制Binder中的Server在启动过程使用Service Manager的addService接口把自己添加到Service Manager守护过程中接受管理。在这一篇文章中，我们将深入到Binder驱动程序源代码去分析Client是如何通过Service Manager的getService接口中来获得Server远程接口的。Client只有获得了Server的远程接口之后，才能进一步调用Server提供的服务。

《[android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

​        这里，我们仍然是通过Android系统中自带的多媒体播放器为例子来说明Client是如何通过IServiceManager::getService接口来获得MediaPlayerService这个Server的远程接口的。假设计读者已经阅读过前面三篇文章[浅谈Service Manager成为Android进程间通信（IPC）机制Binder守护进程之路](http://blog.csdn.net/luoshengyang/article/details/6621566)、[浅谈Android系统进程间通信（IPC）机制Binder中的Server和Client获得Service Manager接口之路](http://blog.csdn.net/luoshengyang/article/details/6627260)和[Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6629298)，即假设Service Manager和MediaPlayerService已经启动完毕，Service Manager现在等待Client的请求。

​        这里，我们要举例子说明的Client便是MediaPlayer了，它声明和实现在frameworks/base/include/media/mediaplayer.h和frameworks/base/media/libmedia/mediaplayer.cpp文件中。MediaPlayer继承于IMediaDeathNotifier类，这个类声明和实现在frameworks/base/include/media/IMediaDeathNotifier.h和frameworks/base/media/libmedia//IMediaDeathNotifier.cpp文件中，里面有一个静态成员函数getMeidaPlayerService，它通过IServiceManager::getService接口来获得MediaPlayerService的远程接口。

​        在介绍IMediaDeathNotifier::getMeidaPlayerService函数之前，我们先了解一下这个函数的目标。看来前面[浅谈Android系统进程间通信（IPC）机制Binder中的Server和Client获得Service Manager接口之路](http://blog.csdn.net/luoshengyang/article/details/6627260)这篇文章的读者知道，我们在获取Service Manager远程接口时，最终是获得了一个BpServiceManager对象的IServiceManager接口。类似地，我们要获得MediaPlayerService的远程接口，实际上就是要获得一个称为BpMediaPlayerService对象的IMediaPlayerService接口。现在，我们就先来看一下BpMediaPlayerService的类图：

![img](http://hi.csdn.net/attachment/201107/26/0_13117045717gMi.gif)

​        从这个类图可以看到，BpMediaPlayerService继承于BpInterface<IMediaPlayerService>类，即BpMediaPlayerService继承了IMediaPlayerService类和BpRefBase类，这两个类又分别继续了RefBase类。BpRefBase类有一个成员变量mRemote，它的类型为IBinder，实际是一个BpBinder对象。BpBinder类使用了IPCThreadState类来与Binder驱动程序进行交互，而IPCThreadState类有一个成员变量mProcess，它的类型为ProcessState，IPCThreadState类借助ProcessState类来打开Binder设备文件/dev/binder，因此，它可以和Binder驱动程序进行交互。

​       BpMediaPlayerService的构造函数有一个参数impl，它的类型为const sp<IBinder>&，从上面的描述中，这个实际上就是一个BpBinder对象。这样，要创建一个BpMediaPlayerService对象，首先就要有一个BpBinder对象。再来看BpBinder类的构造函数，它有一个参数handle，类型为int32_t，这个参数的意义就是请求MediaPlayerService这个远程接口的进程对MediaPlayerService这个Binder实体的引用了。因此，获取MediaPlayerService这个远程接口的本质问题就变为从Service Manager中获得MediaPlayerService的一个句柄了。

​       现在，我们就来看一下IMediaDeathNotifier::getMeidaPlayerService的实现：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. // establish binder interface to MediaPlayerService  
2. /*static*/const sp<IMediaPlayerService>&  
3. IMediaDeathNotifier::getMediaPlayerService()  
4. {  
5. ​    LOGV("getMediaPlayerService");  
6. ​    Mutex::Autolock _l(sServiceLock);  
7. ​    if (sMediaPlayerService.get() == 0) {  
8. ​        sp<IServiceManager> sm = defaultServiceManager();  
9. ​        sp<IBinder> binder;  
10. ​        do {  
11. ​            binder = sm->getService(String16("media.player"));  
12. ​            if (binder != 0) {  
13. ​                break;  
14. ​             }  
15. ​             LOGW("Media player service not published, waiting...");  
16. ​             usleep(500000); // 0.5 s  
17. ​        } while(true);  
18.   
19. ​        if (sDeathNotifier == NULL) {  
20. ​        sDeathNotifier = new DeathNotifier();  
21. ​    }  
22. ​    binder->linkToDeath(sDeathNotifier);  
23. ​    sMediaPlayerService = interface_cast<IMediaPlayerService>(binder);  
24. ​    }  
25. ​    LOGE_IF(sMediaPlayerService == 0, "no media player service!?");  
26. ​    return sMediaPlayerService;  
27. }  

​        函数首先通过defaultServiceManager函数来获得Service Manager的远程接口，实际上就是获得BpServiceManager的IServiceManager接口，具体可以参考

浅谈Android系统进程间通信（IPC）机制Binder中的Server和Client获得Service Manager接口之路

一文。总的来说，这里的语句：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. sp<IServiceManager> sm = defaultServiceManager();  

​        相当于是：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. sp<IServiceManager> sm = new BpServiceManager(new BpBinder(0));   

​        这里的0表示Service Manager的远程接口的句柄值是0。

​        接下去的while循环是通过sm->getService接口来不断尝试获得名称为“media.player”的Service，即MediaPlayerService。为什么要通过这无穷循环来得MediaPlayerService呢？因为这时候MediaPlayerService可能还没有启动起来，所以这里如果发现取回来的binder接口为NULL，就睡眠0.5秒，然后再尝试获取，这是获取Service接口的标准做法。
​        我们来看一下BpServiceManager::getService的实现：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. class BpServiceManager : public BpInterface<IServiceManager>  
2. {  
3. ​    ......  
4.   
5. ​    virtual sp<IBinder> getService(const String16& name) const  
6. ​    {  
7. ​        unsigned n;  
8. ​        for (n = 0; n < 5; n++){  
9. ​            sp<IBinder> svc = checkService(name);  
10. ​            if (svc != NULL) return svc;  
11. ​            LOGI("Waiting for service %s...\n", String8(name).string());  
12. ​            sleep(1);  
13. ​        }  
14. ​        return NULL;  
15. ​    }  
16.   
17. ​    virtual sp<IBinder> checkService( const String16& name) const  
18. ​    {  
19. ​        Parcel data, reply;  
20. ​        data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());  
21. ​        data.writeString16(name);  
22. ​        remote()->transact(CHECK_SERVICE_TRANSACTION, data, &reply);  
23. ​        return reply.readStrongBinder();  
24. ​    }  
25.   
26. ​    ......  
27. };  

​         BpServiceManager::getService通过BpServiceManager::checkService执行操作。

​         在BpServiceManager::checkService中，首先是通过Parcel::writeInterfaceToken往data写入一个RPC头，这个我们在[Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6629298)一文已经介绍过了，就是写往data里面写入了一个整数和一个字符串“android.os.IServiceManager”， Service Manager来处理CHECK_SERVICE_TRANSACTION请求之前，会先验证一下这个RPC头，看看是否正确。接着再往data写入一个字符串name，这里就是“media.player”了。回忆一下[Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6629298)这篇文章，那里已经往Service Manager中注册了一个名字为“media.player”的MediaPlayerService。

​        这里的remote()返回的是一个BpBinder，具体可以参考[浅谈Android系统进程间通信（IPC）机制Binder中的Server和Client获得Service Manager接口之路](http://blog.csdn.net/luoshengyang/article/details/6627260)一文，于是，就进行到BpBinder::transact函数了：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. status_t BpBinder::transact(  
2. ​    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)  
3. {  
4. ​    // Once a binder has died, it will never come back to life.  
5. ​    if (mAlive) {  
6. ​        status_t status = IPCThreadState::self()->transact(  
7. ​            mHandle, code, data, reply, flags);  
8. ​        if (status == DEAD_OBJECT) mAlive = 0;  
9. ​        return status;  
10. ​    }  
11.   
12. ​    return DEAD_OBJECT;  
13. }  

​        这里的mHandle = 0，code = CHECK_SERVICE_TRANSACTION，flags = 0。

​        这里再进入到IPCThread::transact函数中：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. status_t IPCThreadState::transact(int32_t handle,  
2. ​                                  uint32_t code, const Parcel& data,  
3. ​                                  Parcel* reply, uint32_t flags)  
4. {  
5. ​    status_t err = data.errorCheck();  
6.   
7. ​    flags |= TF_ACCEPT_FDS;  
8.   
9. ​    IF_LOG_TRANSACTIONS() {  
10. ​        TextOutput::Bundle _b(alog);  
11. ​        alog << "BC_TRANSACTION thr " << (void*)pthread_self() << " / hand "  
12. ​            << handle << " / code " << TypeCode(code) << ": "  
13. ​            << indent << data << dedent << endl;  
14. ​    }  
15. ​      
16. ​    if (err == NO_ERROR) {  
17. ​        LOG_ONEWAY(">>>> SEND from pid %d uid %d %s", getpid(), getuid(),  
18. ​            (flags & TF_ONE_WAY) == 0 ? "READ REPLY" : "ONE WAY");  
19. ​        err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);  
20. ​    }  
21. ​      
22. ​    if (err != NO_ERROR) {  
23. ​        if (reply) reply->setError(err);  
24. ​        return (mLastError = err);  
25. ​    }  
26. ​      
27. ​    if ((flags & TF_ONE_WAY) == 0) {  
28. ​        #if 0  
29. ​        if (code == 4) { // relayout  
30. ​            LOGI(">>>>>> CALLING transaction 4");  
31. ​        } else {  
32. ​            LOGI(">>>>>> CALLING transaction %d", code);  
33. ​        }  
34. ​        #endif  
35. ​        if (reply) {  
36. ​            err = waitForResponse(reply);  
37. ​        } else {  
38. ​            Parcel fakeReply;  
39. ​            err = waitForResponse(&fakeReply);  
40. ​        }  
41. ​        #if 0  
42. ​        if (code == 4) { // relayout  
43. ​            LOGI("<<<<<< RETURNING transaction 4");  
44. ​        } else {  
45. ​            LOGI("<<<<<< RETURNING transaction %d", code);  
46. ​        }  
47. ​        #endif  
48. ​          
49. ​        IF_LOG_TRANSACTIONS() {  
50. ​            TextOutput::Bundle _b(alog);  
51. ​            alog << "BR_REPLY thr " << (void*)pthread_self() << " / hand "  
52. ​                << handle << ": ";  
53. ​            if (reply) alog << indent << *reply << dedent << endl;  
54. ​            else alog << "(none requested)" << endl;  
55. ​        }  
56. ​    } else {  
57. ​        err = waitForResponse(NULL, NULL);  
58. ​    }  
59. ​      
60. ​    return err;  
61. }  

​         首先是调用函数writeTransactionData写入将要传输的数据到IPCThreadState的成员变量mOut中去：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,  
2. ​    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)  
3. {  
4. ​    binder_transaction_data tr;  
5.   
6. ​    tr.target.handle = handle;  
7. ​    tr.code = code;  
8. ​    tr.flags = binderFlags;  
9. ​      
10. ​    const status_t err = data.errorCheck();  
11. ​    if (err == NO_ERROR) {  
12. ​        tr.data_size = data.ipcDataSize();  
13. ​        tr.data.ptr.buffer = data.ipcData();  
14. ​        tr.offsets_size = data.ipcObjectsCount()*sizeof(size_t);  
15. ​        tr.data.ptr.offsets = data.ipcObjects();  
16. ​    } else if (statusBuffer) {  
17. ​        tr.flags |= TF_STATUS_CODE;  
18. ​        *statusBuffer = err;  
19. ​        tr.data_size = sizeof(status_t);  
20. ​        tr.data.ptr.buffer = statusBuffer;  
21. ​        tr.offsets_size = 0;  
22. ​        tr.data.ptr.offsets = NULL;  
23. ​    } else {  
24. ​        return (mLastError = err);  
25. ​    }  
26. ​      
27. ​    mOut.writeInt32(cmd);  
28. ​    mOut.write(&tr, sizeof(tr));  
29. ​      
30. ​    return NO_ERROR;  
31. }  

​        结构体binder_transaction_data在上一篇文章

Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析

已经介绍过，这里不再累述，这个结构体是用来描述要传输的参数的内容的。这里着重描述一下将要传输的参数tr里面的内容，handle = 0，code =  CHECK_SERVICE_TRANSACTION，cmd = BC_TRANSACTION，data里面的数据分别为：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. writeInt32(IPCThreadState::self()->getStrictModePolicy() | STRICT_MODE_PENALTY_GATHER);  
2. writeString16("android.os.IServiceManager");  
3. writeString16("media.player");  

​       这是在BpServiceManager::checkService函数里面写进去的，其中前两个是RPC头，Service Manager在收到这个请求时会验证这两个参数是否正确，这点前面也提到了。IPCThread->getStrictModePolicy默认返回0，STRICT_MODE_PENALTY_GATHER定义为：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. // Note: must be kept in sync with android/os/StrictMode.java's PENALTY_GATHER  
2. \#define STRICT_MODE_PENALTY_GATHER 0x100  

​       我们不关心这个参数的含义，这不会影响我们分析下面的源代码，有兴趣的读者可以研究一下。这里要注意的是，要传输的参数不包含有Binder对象，因此tr.offsets_size = 0。要传输的参数最后写入到IPCThreadState的成员变量mOut中，包括cmd和tr两个数据。

​       回到IPCThread::transact函数中，由于(flags & TF_ONE_WAY) == 0为true，即这是一个同步请求，并且reply  != NULL，最终调用：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. err = waitForResponse(reply);  

​       进入到waitForResponse函数中：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)  
2. {  
3. ​    int32_t cmd;  
4. ​    int32_t err;  
5.   
6. ​    while (1) {  
7. ​        if ((err=talkWithDriver()) < NO_ERROR) break;  
8. ​        err = mIn.errorCheck();  
9. ​        if (err < NO_ERROR) break;  
10. ​        if (mIn.dataAvail() == 0) continue;  
11. ​          
12. ​        cmd = mIn.readInt32();  
13. ​          
14. ​        IF_LOG_COMMANDS() {  
15. ​            alog << "Processing waitForResponse Command: "  
16. ​                << getReturnString(cmd) << endl;  
17. ​        }  
18.   
19. ​        switch (cmd) {  
20. ​        case BR_TRANSACTION_COMPLETE:  
21. ​            if (!reply && !acquireResult) goto finish;  
22. ​            break;  
23. ​          
24. ​        case BR_DEAD_REPLY:  
25. ​            err = DEAD_OBJECT;  
26. ​            goto finish;  
27.   
28. ​        case BR_FAILED_REPLY:  
29. ​            err = FAILED_TRANSACTION;  
30. ​            goto finish;  
31. ​          
32. ​        case BR_ACQUIRE_RESULT:  
33. ​            {  
34. ​                LOG_ASSERT(acquireResult != NULL, "Unexpected brACQUIRE_RESULT");  
35. ​                const int32_t result = mIn.readInt32();  
36. ​                if (!acquireResult) continue;  
37. ​                *acquireResult = result ? NO_ERROR : INVALID_OPERATION;  
38. ​            }  
39. ​            goto finish;  
40. ​          
41. ​        case BR_REPLY:  
42. ​            {  
43. ​                binder_transaction_data tr;  
44. ​                err = mIn.read(&tr, sizeof(tr));  
45. ​                LOG_ASSERT(err == NO_ERROR, "Not enough command data for brREPLY");  
46. ​                if (err != NO_ERROR) goto finish;  
47.   
48. ​                if (reply) {  
49. ​                    if ((tr.flags & TF_STATUS_CODE) == 0) {  
50. ​                        reply->ipcSetDataReference(  
51. ​                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),  
52. ​                            tr.data_size,  
53. ​                            reinterpret_cast<const size_t*>(tr.data.ptr.offsets),  
54. ​                            tr.offsets_size/sizeof(size_t),  
55. ​                            freeBuffer, this);  
56. ​                    } else {  
57. ​                        err = *static_cast<const status_t*>(tr.data.ptr.buffer);  
58. ​                        freeBuffer(NULL,  
59. ​                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),  
60. ​                            tr.data_size,  
61. ​                            reinterpret_cast<const size_t*>(tr.data.ptr.offsets),  
62. ​                            tr.offsets_size/sizeof(size_t), this);  
63. ​                    }  
64. ​                } else {  
65. ​                    freeBuffer(NULL,  
66. ​                        reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),  
67. ​                        tr.data_size,  
68. ​                        reinterpret_cast<const size_t*>(tr.data.ptr.offsets),  
69. ​                        tr.offsets_size/sizeof(size_t), this);  
70. ​                    continue;  
71. ​                }  
72. ​            }  
73. ​            goto finish;  
74.   
75. ​        default:  
76. ​            err = executeCommand(cmd);  
77. ​            if (err != NO_ERROR) goto finish;  
78. ​            break;  
79. ​        }  
80. ​    }  
81.   
82. finish:  
83. ​    if (err != NO_ERROR) {  
84. ​        if (acquireResult) *acquireResult = err;  
85. ​        if (reply) reply->setError(err);  
86. ​        mLastError = err;  
87. ​    }  
88. ​      
89. ​    return err;  
90. }  

​        这个函数通过IPCThreadState::talkWithDriver与驱动程序进行交互：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. status_t IPCThreadState::talkWithDriver(bool doReceive)  
2. {  
3. ​    LOG_ASSERT(mProcess->mDriverFD >= 0, "Binder driver is not opened");  
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
45. ​    } while (err == -EINTR);  
46.   
47. ​    ......  
48.   
49. ​    if (err >= NO_ERROR) {  
50. ​        if (bwr.write_consumed > 0) {  
51. ​            if (bwr.write_consumed < (ssize_t)mOut.dataSize())  
52. ​                mOut.remove(0, bwr.write_consumed);  
53. ​            else  
54. ​                mOut.setDataSize(0);  
55. ​        }  
56. ​        if (bwr.read_consumed > 0) {  
57. ​            mIn.setDataSize(bwr.read_consumed);  
58. ​            mIn.setDataPosition(0);  
59. ​        }  
60.   
61. ​        ......  
62.   
63. ​        return NO_ERROR;  
64. ​    }  
65.   
66. ​    return err;  
67. }  

​        这里的needRead为true，因此，bwr.read_size大于0；outAvail也大于0，因此，bwr.write_size也大于0。函数最后通过：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)  

​        进入到Binder驱动程序的binder_ioctl函数中。注意，这里的mProcess->mDriverFD是在我们前面调用defaultServiceManager函数获得Service Manager远程接口时，打开的设备文件/dev/binder的文件描述符，mProcess是IPCSThreadState的成员变量。

​        Binder驱动程序的binder_ioctl函数中，我们只关注BINDER_WRITE_READ命令相关的逻辑：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)  
2. {  
3. ​    int ret;  
4. ​    struct binder_proc *proc = filp->private_data;  
5. ​    struct binder_thread *thread;  
6. ​    unsigned int size = _IOC_SIZE(cmd);  
7. ​    void __user *ubuf = (void __user *)arg;  
8.   
9. ​    /*printk(KERN_INFO "binder_ioctl: %d:%d %x %lx\n", proc->pid, current->pid, cmd, arg);*/  
10.   
11. ​    ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);  
12. ​    if (ret)  
13. ​        return ret;  
14.   
15. ​    mutex_lock(&binder_lock);  
16. ​    thread = binder_get_thread(proc);  
17. ​    if (thread == NULL) {  
18. ​        ret = -ENOMEM;  
19. ​        goto err;  
20. ​    }  
21.   
22. ​    switch (cmd) {  
23. ​    case BINDER_WRITE_READ: {  
24. ​        struct binder_write_read bwr;  
25. ​        if (size != sizeof(struct binder_write_read)) {  
26. ​            ret = -EINVAL;  
27. ​            goto err;  
28. ​        }  
29. ​        if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {  
30. ​            ret = -EFAULT;  
31. ​            goto err;  
32. ​        }  
33. ​        if (binder_debug_mask & BINDER_DEBUG_READ_WRITE)  
34. ​            printk(KERN_INFO "binder: %d:%d write %ld at %08lx, read %ld at %08lx\n",  
35. ​            proc->pid, thread->pid, bwr.write_size, bwr.write_buffer, bwr.read_size, bwr.read_buffer);  
36. ​        if (bwr.write_size > 0) {  
37. ​            ret = binder_thread_write(proc, thread, (void __user *)bwr.write_buffer, bwr.write_size, &bwr.write_consumed);  
38. ​            if (ret < 0) {  
39. ​                bwr.read_consumed = 0;  
40. ​                if (copy_to_user(ubuf, &bwr, sizeof(bwr)))  
41. ​                    ret = -EFAULT;  
42. ​                goto err;  
43. ​            }  
44. ​        }  
45. ​        if (bwr.read_size > 0) {  
46. ​            ret = binder_thread_read(proc, thread, (void __user *)bwr.read_buffer, bwr.read_size, &bwr.read_consumed, filp->f_flags & O_NONBLOCK);  
47. ​            if (!list_empty(&proc->todo))  
48. ​                wake_up_interruptible(&proc->wait);  
49. ​            if (ret < 0) {  
50. ​                if (copy_to_user(ubuf, &bwr, sizeof(bwr)))  
51. ​                    ret = -EFAULT;  
52. ​                goto err;  
53. ​            }  
54. ​        }  
55. ​        if (binder_debug_mask & BINDER_DEBUG_READ_WRITE)  
56. ​            printk(KERN_INFO "binder: %d:%d wrote %ld of %ld, read return %ld of %ld\n",  
57. ​            proc->pid, thread->pid, bwr.write_consumed, bwr.write_size, bwr.read_consumed, bwr.read_size);  
58. ​        if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {  
59. ​            ret = -EFAULT;  
60. ​            goto err;  
61. ​        }  
62. ​        break;  
63. ​                            }  
64. ​    ......  
65. ​    default:  
66. ​        ret = -EINVAL;  
67. ​        goto err;  
68. ​    }  
69. ​    ret = 0;  
70. err:  
71. ​    ......  
72. ​    return ret;  
73. }  

​        这里的filp->private_data的值是在defaultServiceManager函数创建ProcessState对象时，在ProcessState构造函数通过open文件操作函数打开设备文件/dev/binder时设置好的，它表示的是调用open函数打开设备文件/dev/binder的进程上下文信息，这里将它取出来保存在proc本地变量中。

​        这里的thread本地变量表示当前线程上下文信息，通过binder_get_thread函数获得。在前面执行ProcessState构造函数时，也会通过ioctl文件操作函数进入到这个函数，那是第一次进入到binder_ioctl这里，因此，调用binder_get_thread时，表示当前进程上下文信息的proc变量还没有关于当前线程的上下文信息，因此，会为proc创建一个表示当前线程上下文信息的thread，会保存在proc->threads表示的红黑树结构中。这里调用binder_get_thread就可以直接从proc找到并返回了。

​        进入到BINDER_WRITE_READ相关的逻辑。先看看BINDER_WRITE_READ的定义：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. \#define BINDER_WRITE_READ           _IOWR('b', 1, struct binder_write_read)  

​        这里可以看出，BINDER_WRITE_READ命令的参数类型为struct binder_write_read：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. struct binder_write_read {  
2. ​    signed long write_size; /* bytes to write */  
3. ​    signed long write_consumed; /* bytes consumed by driver */  
4. ​    unsigned long   write_buffer;  
5. ​    signed long read_size;  /* bytes to read */  
6. ​    signed long read_consumed;  /* bytes consumed by driver */  
7. ​    unsigned long   read_buffer;  
8. };  

​        这个结构体的含义可以参考

浅谈Service Manager成为Android进程间通信（IPC）机制Binder守护进程之路

一文。这里首先是通过copy_from_user函数把用户传进来的参数的内容拷贝到本地变量bwr中。

​        从上面的调用过程，我们知道，这里bwr.write_size是大于0的，因此进入到binder_thread_write函数中，我们只关注BC_TRANSACTION相关的逻辑：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. int  
2. binder_thread_write(struct binder_proc *proc, struct binder_thread *thread,  
3. ​                    void __user *buffer, int size, signed long *consumed)  
4. {  
5. ​    uint32_t cmd;  
6. ​    void __user *ptr = buffer + *consumed;  
7. ​    void __user *end = buffer + size;  
8.   
9. ​    while (ptr < end && thread->return_error == BR_OK) {  
10. ​        if (get_user(cmd, (uint32_t __user *)ptr))  
11. ​            return -EFAULT;  
12. ​        ptr += sizeof(uint32_t);  
13. ​        if (_IOC_NR(cmd) < ARRAY_SIZE(binder_stats.bc)) {  
14. ​            binder_stats.bc[_IOC_NR(cmd)]++;  
15. ​            proc->stats.bc[_IOC_NR(cmd)]++;  
16. ​            thread->stats.bc[_IOC_NR(cmd)]++;  
17. ​        }  
18. ​        switch (cmd) {  
19. ​        ......  
20. ​        case BC_TRANSACTION:  
21. ​        case BC_REPLY: {  
22. ​            struct binder_transaction_data tr;  
23.   
24. ​            if (copy_from_user(&tr, ptr, sizeof(tr)))  
25. ​                return -EFAULT;  
26. ​            ptr += sizeof(tr);  
27. ​            binder_transaction(proc, thread, &tr, cmd == BC_REPLY);  
28. ​            break;  
29. ​                       }  
30. ​        ......  
31. ​        default:  
32. ​            printk(KERN_ERR "binder: %d:%d unknown command %d\n", proc->pid, thread->pid, cmd);  
33. ​            return -EINVAL;  
34. ​        }  
35. ​        *consumed = ptr - buffer;  
36. ​    }  
37. ​    return 0;  
38. }  

​        这里再次把用户传出来的参数拷贝到本地变量tr中，tr的类型为struct binder_transaction_data，这个就是前面我们在IPCThreadState::writeTransactionData写入的内容了。

​        接着进入到binder_transaction函数中，不相关的代码我们忽略掉：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. static void  
2. binder_transaction(struct binder_proc *proc, struct binder_thread *thread,  
3. struct binder_transaction_data *tr, int reply)  
4. {  
5. ​    struct binder_transaction *t;  
6. ​    struct binder_work *tcomplete;  
7. ​    size_t *offp, *off_end;  
8. ​    struct binder_proc *target_proc;  
9. ​    struct binder_thread *target_thread = NULL;  
10. ​    struct binder_node *target_node = NULL;  
11. ​    struct list_head *target_list;  
12. ​    wait_queue_head_t *target_wait;  
13. ​    struct binder_transaction *in_reply_to = NULL;  
14. ​    struct binder_transaction_log_entry *e;  
15. ​    uint32_t return_error;  
16.   
17. ​    .......  
18.   
19. ​    if (reply) {  
20. ​        ......  
21. ​    } else {  
22. ​        if (tr->target.handle) {  
23. ​            ......  
24. ​        } else {  
25. ​            target_node = binder_context_mgr_node;  
26. ​            if (target_node == NULL) {  
27. ​                return_error = BR_DEAD_REPLY;  
28. ​                goto err_no_context_mgr_node;  
29. ​            }  
30. ​        }  
31. ​        ......  
32. ​        target_proc = target_node->proc;  
33. ​        if (target_proc == NULL) {  
34. ​            return_error = BR_DEAD_REPLY;  
35. ​            goto err_dead_binder;  
36. ​        }  
37. ​        if (!(tr->flags & TF_ONE_WAY) && thread->transaction_stack) {  
38. ​            ......  
39. ​        }  
40. ​    }  
41. ​    if (target_thread) {  
42. ​        ......  
43. ​    } else {  
44. ​        target_list = &target_proc->todo;  
45. ​        target_wait = &target_proc->wait;  
46. ​    }  
47. ​    ......  
48.   
49. ​    /* TODO: reuse incoming transaction for reply */  
50. ​    t = kzalloc(sizeof(*t), GFP_KERNEL);  
51. ​    if (t == NULL) {  
52. ​        return_error = BR_FAILED_REPLY;  
53. ​        goto err_alloc_t_failed;  
54. ​    }  
55. ​    binder_stats.obj_created[BINDER_STAT_TRANSACTION]++;  
56.   
57. ​    tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);  
58. ​    if (tcomplete == NULL) {  
59. ​        return_error = BR_FAILED_REPLY;  
60. ​        goto err_alloc_tcomplete_failed;  
61. ​    }  
62. ​    binder_stats.obj_created[BINDER_STAT_TRANSACTION_COMPLETE]++;  
63.   
64. ​    t->debug_id = ++binder_last_id;  
65. ​      
66. ​    ......  
67.   
68.   
69. ​    if (!reply && !(tr->flags & TF_ONE_WAY))  
70. ​        t->from = thread;  
71. ​    else  
72. ​        t->from = NULL;  
73. ​    t->sender_euid = proc->tsk->cred->euid;  
74. ​    t->to_proc = target_proc;  
75. ​    t->to_thread = target_thread;  
76. ​    t->code = tr->code;  
77. ​    t->flags = tr->flags;  
78. ​    t->priority = task_nice(current);  
79. ​    t->buffer = binder_alloc_buf(target_proc, tr->data_size,  
80. ​        tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));  
81. ​    if (t->buffer == NULL) {  
82. ​        return_error = BR_FAILED_REPLY;  
83. ​        goto err_binder_alloc_buf_failed;  
84. ​    }  
85. ​    t->buffer->allow_user_free = 0;  
86. ​    t->buffer->debug_id = t->debug_id;  
87. ​    t->buffer->transaction = t;  
88. ​    t->buffer->target_node = target_node;  
89. ​    if (target_node)  
90. ​        binder_inc_node(target_node, 1, 0, NULL);  
91.   
92. ​    offp = (size_t *)(t->buffer->data + ALIGN(tr->data_size, sizeof(void *)));  
93.   
94. ​    if (copy_from_user(t->buffer->data, tr->data.ptr.buffer, tr->data_size)) {  
95. ​        ......  
96. ​        return_error = BR_FAILED_REPLY;  
97. ​        goto err_copy_data_failed;  
98. ​    }  
99.   
100. ​    ......  
101.   
102. ​    if (reply) {  
103. ​        ......  
104. ​    } else if (!(t->flags & TF_ONE_WAY)) {  
105. ​        BUG_ON(t->buffer->async_transaction != 0);  
106. ​        t->need_reply = 1;  
107. ​        t->from_parent = thread->transaction_stack;  
108. ​        thread->transaction_stack = t;  
109. ​    } else {  
110. ​        ......  
111. ​    }  
112.   
113. ​    t->work.type = BINDER_WORK_TRANSACTION;  
114. ​    list_add_tail(&t->work.entry, target_list);  
115. ​    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;  
116. ​    list_add_tail(&tcomplete->entry, &thread->todo);  
117. ​    if (target_wait)  
118. ​        wake_up_interruptible(target_wait);  
119. ​    return;  
120.   
121. ​    ......  
122. }  

​        注意，这里的参数reply = 0，表示这是一个BC_TRANSACTION命令。

​        前面我们提到，传给驱动程序的handle值为0，即这里的tr->target.handle = 0，表示请求的目标Binder对象是Service Manager，因此有：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. target_node = binder_context_mgr_node;  
2. target_proc = target_node->proc;  
3. target_list = &target_proc->todo;  
4. target_wait = &target_proc->wait;  

​        其中binder_context_mgr_node是在Service Manager通知Binder驱动程序它是守护过程时创建的。

​        接着创建一个待完成事项tcomplete，它的类型为struct binder_work，这是等一会要保存在当前线程的todo队列去的，表示当前线程有一个待完成的事务。紧跟着创建一个待处理事务t，它的类型为struct binder_transaction，这是等一会要存在到Service Manager的todo队列去的，表示Service Manager当前有一个事务需要处理。同时，这个待处理事务t也要存放在当前线程的待完成事务transaction_stack列表中去：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. t->from_parent = thread->transaction_stack;  
2. thread->transaction_stack = t;  

​        这样表明当前线程还有事务要处理。

​        继续往下看，就是分别把tcomplete和t放在当前线程thread和Service Manager进程的todo队列去了：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. t->work.type = BINDER_WORK_TRANSACTION;  
2. list_add_tail(&t->work.entry, target_list);  
3. tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;  
4. list_add_tail(&tcomplete->entry, &thread->todo);  

​        最后，Service Manager有事情可做了，就要唤醒它了：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. wake_up_interruptible(target_wait);  

​        前面我们提到，此时Service Manager正在等待Client的请求，也就是Service Manager此时正在进入到Binder驱动程序的binder_thread_read函数中，并且休眠在target->wait上，具体参考

浅谈Service Manager成为Android进程间通信（IPC）机制Binder守护进程之路

一文。

​        这里，我们暂时忽略Service Manager被唤醒之后的情景，继续看当前线程的执行。

​        函数binder_transaction执行完成之后，就一路返回到binder_ioctl函数里去了。函数binder_ioctl从binder_thread_write函数调用处返回后，发现bwr.read_size大于0，于是就进入到binder_thread_read函数去了：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. static int  
2. binder_thread_read(struct binder_proc *proc, struct binder_thread *thread,  
3. ​                   void  __user *buffer, int size, signed long *consumed, int non_block)  
4. {  
5. ​    void __user *ptr = buffer + *consumed;  
6. ​    void __user *end = buffer + size;  
7.   
8. ​    int ret = 0;  
9. ​    int wait_for_proc_work;  
10.   
11. ​    if (*consumed == 0) {  
12. ​        if (put_user(BR_NOOP, (uint32_t __user *)ptr))  
13. ​            return -EFAULT;  
14. ​        ptr += sizeof(uint32_t);  
15. ​    }  
16.   
17. retry:  
18. ​    wait_for_proc_work = thread->transaction_stack == NULL && list_empty(&thread->todo);  
19.   
20. ​    ......  
21. ​      
22. ​    if (wait_for_proc_work) {  
23. ​        ......  
24. ​    } else {  
25. ​        if (non_block) {  
26. ​            if (!binder_has_thread_work(thread))  
27. ​                ret = -EAGAIN;  
28. ​        } else  
29. ​            ret = wait_event_interruptible(thread->wait, binder_has_thread_work(thread));  
30. ​    }  
31.   
32. ​    ......  
33.   
34. ​    while (1) {  
35. ​        uint32_t cmd;  
36. ​        struct binder_transaction_data tr;  
37. ​        struct binder_work *w;  
38. ​        struct binder_transaction *t = NULL;  
39.   
40. ​        if (!list_empty(&thread->todo))  
41. ​            w = list_first_entry(&thread->todo, struct binder_work, entry);  
42. ​        else if (!list_empty(&proc->todo) && wait_for_proc_work)  
43. ​            w = list_first_entry(&proc->todo, struct binder_work, entry);  
44. ​        else {  
45. ​            if (ptr - buffer == 4 && !(thread->looper & BINDER_LOOPER_STATE_NEED_RETURN)) /* no data added */  
46. ​                goto retry;  
47. ​            break;  
48. ​        }  
49.   
50. ​        if (end - ptr < sizeof(tr) + 4)  
51. ​            break;  
52.   
53. ​        switch (w->type) {  
54. ​        ......  
55. ​        case BINDER_WORK_TRANSACTION_COMPLETE: {  
56. ​            cmd = BR_TRANSACTION_COMPLETE;  
57. ​            if (put_user(cmd, (uint32_t __user *)ptr))  
58. ​                return -EFAULT;  
59. ​            ptr += sizeof(uint32_t);  
60.   
61. ​            binder_stat_br(proc, thread, cmd);  
62. ​            if (binder_debug_mask & BINDER_DEBUG_TRANSACTION_COMPLETE)  
63. ​                printk(KERN_INFO "binder: %d:%d BR_TRANSACTION_COMPLETE\n",  
64. ​                proc->pid, thread->pid);  
65.   
66. ​            list_del(&w->entry);  
67. ​            kfree(w);  
68. ​            binder_stats.obj_deleted[BINDER_STAT_TRANSACTION_COMPLETE]++;  
69. ​                                               } break;  
70. ​        ......  
71. ​        }  
72.   
73. ​        if (!t)  
74. ​            continue;  
75.   
76. ​        ......  
77. ​    }  
78.   
79. done:  
80. ​    ......  
81. ​    return 0;  
82. }  

​       函数首先是写入一个操作码BR_NOOP到用户传进来的缓冲区中去。

​      回忆一下上面的binder_transaction函数，这里的thread->transaction_stack != NULL，并且thread->todo也不为空，所以线程不会进入休眠状态。

​      进入while循环中，首先是从thread->todo队列中取回待处理事项w，w的类型为BINDER_WORK_TRANSACTION_COMPLETE，这也是在binder_transaction函数里面设置的。对BINDER_WORK_TRANSACTION_COMPLETE的处理也很简单，只是把一个操作码BR_TRANSACTION_COMPLETE写回到用户传进来的缓冲区中去。这时候，用户传进来的缓冲区就包含两个操作码了，分别是BR_NOOP和BINDER_WORK_TRANSACTION_COMPLETE。

​      binder_thread_read执行完之后，返回到binder_ioctl函数中，将操作结果写回到用户空间中去：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {  
2. ​    ret = -EFAULT;  
3. ​    goto err;  
4. }  

​       最后就返回到IPCThreadState::talkWithDriver函数中了。

​       IPCThreadState::talkWithDriver函数从下面语句：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)  

​       返回后，首先是清空之前写入Binder驱动程序的内容：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. if (bwr.write_consumed > 0) {  
2. ​     if (bwr.write_consumed < (ssize_t)mOut.dataSize())  
3. ​          mOut.remove(0, bwr.write_consumed);  
4. ​     else  
5. ​          mOut.setDataSize(0);  
6. }  

​       接着是设置从Binder驱动程序读取的内容：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. if (bwr.read_consumed > 0) {  
2. ​     mIn.setDataSize(bwr.read_consumed);  
3. ​     mIn.setDataPosition(0);  
4. }  

​       然后就返回到IPCThreadState::waitForResponse去了。IPCThreadState::waitForResponse函数的处理也很简单，就是处理刚才从Binder驱动程序读入内容了。从前面的分析中，我们知道，从Binder驱动程序读入的内容就是两个整数了，分别是BR_NOOP和BR_TRANSACTION_COMPLETE。对BR_NOOP的处理很简单，正如它的名字所示，什么也不做；而对BR_TRANSACTION_COMPLETE的处理，就分情况了，如果这个请求是异步的，那个整个BC_TRANSACTION操作就完成了，如果这个请求是同步的，即要等待回复的，也就是reply不为空，那么还要继续通过IPCThreadState::talkWithDriver进入到Binder驱动程序中去等待BC_TRANSACTION操作的处理结果。

​      这里属于后一种情况，于是再次通过IPCThreadState::talkWithDriver进入到Binder驱动程序的binder_ioctl函数中。不过这一次在binder_ioctl函数中，bwr.write_size等于0，而bwr.read_size大于0，于是再次进入到binder_thread_read函数中。这时候thread->transaction_stack仍然不为NULL，不过thread->todo队列已经为空了，因为前面我们已经处理过thread->todo队列的内容了，于是就通过下面语句：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. ret = wait_event_interruptible(thread->wait, binder_has_thread_work(thread));  

​      进入休眠状态了，等待Service Manager的唤醒。

​      现在，我们终于可以回到Service Manager被唤醒之后的过程了。前面我们说过，Service Manager此时正在binder_thread_read函数中休眠中：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. static int  
2. binder_thread_read(struct binder_proc *proc, struct binder_thread *thread,  
3. ​                   void  __user *buffer, int size, signed long *consumed, int non_block)  
4. {  
5. ​    void __user *ptr = buffer + *consumed;  
6. ​    void __user *end = buffer + size;  
7.   
8. ​    int ret = 0;  
9. ​    int wait_for_proc_work;  
10.   
11. ​    if (*consumed == 0) {  
12. ​        if (put_user(BR_NOOP, (uint32_t __user *)ptr))  
13. ​            return -EFAULT;  
14. ​        ptr += sizeof(uint32_t);  
15. ​    }  
16.   
17. retry:  
18. ​    wait_for_proc_work = thread->transaction_stack == NULL && list_empty(&thread->todo);  
19.   
20. ​    ......  
21.   
22. ​    if (wait_for_proc_work) {  
23. ​        ......  
24. ​        if (non_block) {  
25. ​            if (!binder_has_proc_work(proc, thread))  
26. ​                ret = -EAGAIN;  
27. ​        } else  
28. ​            ret = wait_event_interruptible_exclusive(proc->wait, binder_has_proc_work(proc, thread));  
29. ​    } else {  
30. ​        ......  
31. ​    }  
32. ​      
33. ​    ......  
34.   
35. ​    while (1) {  
36. ​        uint32_t cmd;  
37. ​        struct binder_transaction_data tr;  
38. ​        struct binder_work *w;  
39. ​        struct binder_transaction *t = NULL;  
40.   
41. ​        if (!list_empty(&thread->todo))  
42. ​            w = list_first_entry(&thread->todo, struct binder_work, entry);  
43. ​        else if (!list_empty(&proc->todo) && wait_for_proc_work)  
44. ​            w = list_first_entry(&proc->todo, struct binder_work, entry);  
45. ​        else {  
46. ​            if (ptr - buffer == 4 && !(thread->looper & BINDER_LOOPER_STATE_NEED_RETURN)) /* no data added */  
47. ​                goto retry;  
48. ​            break;  
49. ​        }  
50.   
51. ​        if (end - ptr < sizeof(tr) + 4)  
52. ​            break;  
53.   
54. ​        switch (w->type) {  
55. ​        case BINDER_WORK_TRANSACTION: {  
56. ​            t = container_of(w, struct binder_transaction, work);  
57. ​                                      } break;  
58. ​        ......  
59. ​        }  
60.   
61. ​        if (!t)  
62. ​            continue;  
63.   
64. ​        BUG_ON(t->buffer == NULL);  
65. ​        if (t->buffer->target_node) {  
66. ​            struct binder_node *target_node = t->buffer->target_node;  
67. ​            tr.target.ptr = target_node->ptr;  
68. ​            tr.cookie =  target_node->cookie;  
69. ​            t->saved_priority = task_nice(current);  
70. ​            if (t->priority < target_node->min_priority &&  
71. ​                !(t->flags & TF_ONE_WAY))  
72. ​                binder_set_nice(t->priority);  
73. ​            else if (!(t->flags & TF_ONE_WAY) ||  
74. ​                t->saved_priority > target_node->min_priority)  
75. ​                binder_set_nice(target_node->min_priority);  
76. ​            cmd = BR_TRANSACTION;  
77. ​        } else {  
78. ​            ......  
79. ​        }  
80. ​        tr.code = t->code;  
81. ​        tr.flags = t->flags;  
82. ​        tr.sender_euid = t->sender_euid;  
83.   
84. ​        if (t->from) {  
85. ​            struct task_struct *sender = t->from->proc->tsk;  
86. ​            tr.sender_pid = task_tgid_nr_ns(sender, current->nsproxy->pid_ns);  
87. ​        } else {  
88. ​            ......  
89. ​        }  
90.   
91. ​        tr.data_size = t->buffer->data_size;  
92. ​        tr.offsets_size = t->buffer->offsets_size;  
93. ​        tr.data.ptr.buffer = (void *)t->buffer->data + proc->user_buffer_offset;  
94. ​        tr.data.ptr.offsets = tr.data.ptr.buffer + ALIGN(t->buffer->data_size, sizeof(void *));  
95.   
96. ​        if (put_user(cmd, (uint32_t __user *)ptr))  
97. ​            return -EFAULT;  
98. ​        ptr += sizeof(uint32_t);  
99. ​        if (copy_to_user(ptr, &tr, sizeof(tr)))  
100. ​            return -EFAULT;  
101. ​        ptr += sizeof(tr);  
102.   
103. ​        ......  
104.   
105. ​        list_del(&t->work.entry);  
106. ​        t->buffer->allow_user_free = 1;  
107. ​        if (cmd == BR_TRANSACTION && !(t->flags & TF_ONE_WAY)) {  
108. ​            t->to_parent = thread->transaction_stack;  
109. ​            t->to_thread = thread;  
110. ​            thread->transaction_stack = t;  
111. ​        } else {  
112. ​            ......  
113. ​        }  
114. ​        break;  
115. ​    }  
116.   
117. done:  
118.   
119. ​    *consumed = ptr - buffer;  
120. ​    ......  
121. ​    return 0;  
122. }  

​        这里就是从语句中唤醒了：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. ret = wait_event_interruptible_exclusive(proc->wait, binder_has_proc_work(proc, thread));  

​        Service Manager唤醒过来看，继续往下执行，进入到while循环中。首先是从proc->todo中取回待处理事项w。这个事项w的类型是BINDER_WORK_TRANSACTION，这是上面调用binder_transaction的时候设置的，于是通过w得到待处理事务t：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. t = container_of(w, struct binder_transaction, work);  

​        接下来的内容，就把cmd和t->buffer的内容拷贝到用户传进来的缓冲区去了，这里就是Service Manager从用户空间传进来的缓冲区了：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. if (put_user(cmd, (uint32_t __user *)ptr))  
2. ​    return -EFAULT;  
3. ptr += sizeof(uint32_t);  
4. if (copy_to_user(ptr, &tr, sizeof(tr)))  
5. ​    return -EFAULT;  
6. ptr += sizeof(tr);  

​        注意，这里先是把t->buffer的内容拷贝到本地变量tr中，再拷贝到用户空间缓冲区去。关于t->buffer内容的拷贝，请参考

Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析

一文，它的一个关键地方是Binder驱动程序和Service Manager守护进程共享了同一个物理内存的内容，拷贝的只是这个物理内存在用户空间的虚拟地址回去：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. tr.data.ptr.buffer = (void *)t->buffer->data + proc->user_buffer_offset;  
2. tr.data.ptr.offsets = tr.data.ptr.buffer + ALIGN(t->buffer->data_size, sizeof(void *));  

​       对于Binder驱动程序这次操作来说，这个事项就算是处理完了，就要从todo队列中删除了：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. list_del(&t->work.entry);  

​       紧接着，还不放删除这个事务，因为它还要等待Service Manager处理完成后，再进一步处理，因此，放在thread->transaction_stack队列中：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. t->to_parent = thread->transaction_stack;  
2. t->to_thread = thread;  
3. thread->transaction_stack = t;  

​       还要注意的一个地方是，上面写入的cmd = BR_TRANSACTION，告诉Service Manager守护进程，它要做什么事情，后面我们会看到相应的分析。

​       这样，binder_thread_read函数就处理完了，回到binder_ioctl函数中，同样是操作结果写回到用户空间的缓冲区中去：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {  
2. ​    ret = -EFAULT;  
3. ​    goto err;  
4. }  

​       最后，就返回到frameworks/base/cmds/servicemanager/binder.c文件中的binder_loop函数去了：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. void binder_loop(struct binder_state *bs, binder_handler func)  
2. {  
3. ​    int res;  
4. ​    struct binder_write_read bwr;  
5. ​    unsigned readbuf[32];  
6.   
7. ​    bwr.write_size = 0;  
8. ​    bwr.write_consumed = 0;  
9. ​    bwr.write_buffer = 0;  
10. ​      
11. ​    readbuf[0] = BC_ENTER_LOOPER;  
12. ​    binder_write(bs, readbuf, sizeof(unsigned));  
13.   
14. ​    for (;;) {  
15. ​        bwr.read_size = sizeof(readbuf);  
16. ​        bwr.read_consumed = 0;  
17. ​        bwr.read_buffer = (unsigned) readbuf;  
18.   
19. ​        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);  
20.   
21. ​        if (res < 0) {  
22. ​            LOGE("binder_loop: ioctl failed (%s)\n", strerror(errno));  
23. ​            break;  
24. ​        }  
25.   
26. ​        res = binder_parse(bs, 0, readbuf, bwr.read_consumed, func);  
27. ​        if (res == 0) {  
28. ​            LOGE("binder_loop: unexpected reply?!\n");  
29. ​            break;  
30. ​        }  
31. ​        if (res < 0) {  
32. ​            LOGE("binder_loop: io error %d %s\n", res, strerror(errno));  
33. ​            break;  
34. ​        }  
35. ​    }  
36. }  

​        这里就是从下面的语句：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);  

​        返回来了。接着就进入binder_parse函数处理从Binder驱动程序里面读取出来的数据：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. int binder_parse(struct binder_state *bs, struct binder_io *bio,  
2. ​                 uint32_t *ptr, uint32_t size, binder_handler func)  
3. {  
4. ​    int r = 1;  
5. ​    uint32_t *end = ptr + (size / 4);  
6.   
7. ​    while (ptr < end) {  
8. ​        uint32_t cmd = *ptr++;  
9. ​        switch(cmd) {  
10. ​        ......  
11. ​        case BR_TRANSACTION: {  
12. ​            struct binder_txn *txn = (void *) ptr;  
13. ​            ......  
14. ​            if (func) {  
15. ​                unsigned rdata[256/4];  
16. ​                struct binder_io msg;  
17. ​                struct binder_io reply;  
18. ​                int res;  
19.   
20. ​                bio_init(&reply, rdata, sizeof(rdata), 4);  
21. ​                bio_init_from_txn(&msg, txn);  
22. ​                res = func(bs, txn, &msg, &reply);  
23. ​                binder_send_reply(bs, &reply, txn->data, res);  
24. ​            }  
25. ​            ptr += sizeof(*txn) / sizeof(uint32_t);  
26. ​            break;  
27. ​                             }  
28. ​        ......  
29. ​        default:  
30. ​            LOGE("parse: OOPS %d\n", cmd);  
31. ​            return -1;  
32. ​        }  
33. ​    }  
34.   
35. ​    return r;  
36. }  

​         前面我们说过，Binder驱动程序写入到用户空间的缓冲区中的cmd为BR_TRANSACTION，因此，这里我们只关注BR_TRANSACTION相关的逻辑。

​         这里用到的两个[数据结构](http://lib.csdn.net/base/datastructure)struct binder_txn和struct binder_io可以参考前面一篇文章[Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6629298)，这里就不复述了。

​         接着往下看，函数调bio_init来初始化reply变量：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. void bio_init(struct binder_io *bio, void *data,  
2. ​              uint32_t maxdata, uint32_t maxoffs)  
3. {  
4. ​    uint32_t n = maxoffs * sizeof(uint32_t);  
5.   
6. ​    if (n > maxdata) {  
7. ​        bio->flags = BIO_F_OVERFLOW;  
8. ​        bio->data_avail = 0;  
9. ​        bio->offs_avail = 0;  
10. ​        return;  
11. ​    }  
12.   
13. ​    bio->data = bio->data0 = data + n;  
14. ​    bio->offs = bio->offs0 = data;  
15. ​    bio->data_avail = maxdata - n;  
16. ​    bio->offs_avail = maxoffs;  
17. ​    bio->flags = 0;  
18. }  

​        接着又调用bio_init_from_txn来初始化msg变量：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. void bio_init_from_txn(struct binder_io *bio, struct binder_txn *txn)  
2. {  
3. ​    bio->data = bio->data0 = txn->data;  
4. ​    bio->offs = bio->offs0 = txn->offs;  
5. ​    bio->data_avail = txn->data_size;  
6. ​    bio->offs_avail = txn->offs_size / 4;  
7. ​    bio->flags = BIO_F_SHARED;  
8. }  

​       最后，真正进行处理的函数是从参数中传进来的函数指针func，这里就是定义在frameworks/base/cmds/servicemanager/service_manager.c文件中的svcmgr_handler函数：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. int svcmgr_handler(struct binder_state *bs,  
2. ​                   struct binder_txn *txn,  
3. ​                   struct binder_io *msg,  
4. ​                   struct binder_io *reply)  
5. {  
6. ​    struct svcinfo *si;  
7. ​    uint16_t *s;  
8. ​    unsigned len;  
9. ​    void *ptr;  
10. ​    uint32_t strict_policy;  
11.   
12. //    LOGI("target=%p code=%d pid=%d uid=%d\n",  
13. //         txn->target, txn->code, txn->sender_pid, txn->sender_euid);  
14.   
15. ​    if (txn->target != svcmgr_handle)  
16. ​        return -1;  
17.   
18. ​    // Equivalent to Parcel::enforceInterface(), reading the RPC  
19. ​    // header with the strict mode policy mask and the interface name.  
20. ​    // Note that we ignore the strict_policy and don't propagate it  
21. ​    // further (since we do no outbound RPCs anyway).  
22. ​    strict_policy = bio_get_uint32(msg);  
23. ​    s = bio_get_string16(msg, &len);  
24. ​    if ((len != (sizeof(svcmgr_id) / 2)) ||  
25. ​        memcmp(svcmgr_id, s, sizeof(svcmgr_id))) {  
26. ​        fprintf(stderr,"invalid id %s\n", str8(s));  
27. ​        return -1;  
28. ​    }  
29.   
30. ​    switch(txn->code) {  
31. ​    case SVC_MGR_GET_SERVICE:  
32. ​    case SVC_MGR_CHECK_SERVICE:  
33. ​        s = bio_get_string16(msg, &len);  
34. ​        ptr = do_find_service(bs, s, len);  
35. ​        if (!ptr)  
36. ​            break;  
37. ​        bio_put_ref(reply, ptr);  
38. ​        return 0;  
39.   
40. ​    ......  
41. ​    }  
42. ​    default:  
43. ​        LOGE("unknown code %d\n", txn->code);  
44. ​        return -1;  
45. ​    }  
46.   
47. ​    bio_put_uint32(reply, 0);  
48. ​    return 0;  
49. }  

​        这里， Service Manager要处理的code是SVC_MGR_CHECK_SERVICE，这是在前面的BpServiceManager::checkService函数里面设置的。

​        回忆一下，在BpServiceManager::checkService时，传给Binder驱动程序的参数为：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. writeInt32(IPCThreadState::self()->getStrictModePolicy() | STRICT_MODE_PENALTY_GATHER);    
2. writeString16("android.os.IServiceManager");    
3. writeString16("media.player");    

​       这里的语句：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. strict_policy = bio_get_uint32(msg);    
2. s = bio_get_string16(msg, &len);    
3. s = bio_get_string16(msg, &len);   

​       其中，会验证一下传进来的第二个参数，即"android.os.IServiceManager"是否正确，这个是验证RPC头，注释已经说得很清楚了。

​       最后，就是调用do_find_service函数查找是存在名称为"media.player"的服务了。回忆一下前面一篇文章[Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6629298)，MediaPlayerService已经把一个名称为"media.player"的服务注册到Service Manager中，所以这里一定能找到。我们看看do_find_service这个函数：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. void *do_find_service(struct binder_state *bs, uint16_t *s, unsigned len)  
2. {  
3. ​    struct svcinfo *si;  
4. ​    si = find_svc(s, len);  
5.   
6. //    LOGI("check_service('%s') ptr = %p\n", str8(s), si ? si->ptr : 0);  
7. ​    if (si && si->ptr) {  
8. ​        return si->ptr;  
9. ​    } else {  
10. ​        return 0;  
11. ​    }  
12. }  

​       这里又调用了find_svc函数：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. struct svcinfo *find_svc(uint16_t *s16, unsigned len)  
2. {  
3. ​    struct svcinfo *si;  
4.   
5. ​    for (si = svclist; si; si = si->next) {  
6. ​        if ((len == si->len) &&  
7. ​            !memcmp(s16, si->name, len * sizeof(uint16_t))) {  
8. ​            return si;  
9. ​        }  
10. ​    }  
11. ​    return 0;  
12. }  

​       就是在svclist列表中查找对应名称的svcinfo了。

​       然后返回到do_find_service函数中。回忆一下前面一篇文章[Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6629298)，这里的si->ptr就是指MediaPlayerService这个Binder实体在Service Manager进程中的句柄值了。

​       回到svcmgr_handler函数中，调用bio_put_ref函数将这个Binder引用写回到reply参数。我们看看bio_put_ref的实现：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. void bio_put_ref(struct binder_io *bio, void *ptr)  
2. {  
3. ​    struct binder_object *obj;  
4.   
5. ​    if (ptr)  
6. ​        obj = bio_alloc_obj(bio);  
7. ​    else  
8. ​        obj = bio_alloc(bio, sizeof(*obj));  
9.   
10. ​    if (!obj)  
11. ​        return;  
12.   
13. ​    obj->flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;  
14. ​    obj->type = BINDER_TYPE_HANDLE;  
15. ​    obj->pointer = ptr;  
16. ​    obj->cookie = 0;  
17. }  

​        这里很简单，就是把一个类型为BINDER_TYPE_HANDLE的binder_object写入到reply缓冲区中去。这里的binder_object就是相当于是flat_binder_obj了，具体可以参考

Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析

一文。

​        再回到svcmgr_handler函数中，最后，还写入一个0值到reply缓冲区中，表示操作结果码：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. bio_put_uint32(reply, 0);  

​        最后返回到binder_parse函数中，调用binder_send_reply函数将操作结果反馈给Binder驱动程序：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. void binder_send_reply(struct binder_state *bs,  
2. ​                       struct binder_io *reply,  
3. ​                       void *buffer_to_free,  
4. ​                       int status)  
5. {  
6. ​    struct {  
7. ​        uint32_t cmd_free;  
8. ​        void *buffer;  
9. ​        uint32_t cmd_reply;  
10. ​        struct binder_txn txn;  
11. ​    } __attribute__((packed)) data;  
12.   
13. ​    data.cmd_free = BC_FREE_BUFFER;  
14. ​    data.buffer = buffer_to_free;  
15. ​    data.cmd_reply = BC_REPLY;  
16. ​    data.txn.target = 0;  
17. ​    data.txn.cookie = 0;  
18. ​    data.txn.code = 0;  
19. ​    if (status) {  
20. ​        data.txn.flags = TF_STATUS_CODE;  
21. ​        data.txn.data_size = sizeof(int);  
22. ​        data.txn.offs_size = 0;  
23. ​        data.txn.data = &status;  
24. ​        data.txn.offs = 0;  
25. ​    } else {  
26. ​        data.txn.flags = 0;  
27. ​        data.txn.data_size = reply->data - reply->data0;  
28. ​        data.txn.offs_size = ((char*) reply->offs) - ((char*) reply->offs0);  
29. ​        data.txn.data = reply->data0;  
30. ​        data.txn.offs = reply->offs0;  
31. ​    }  
32. ​    binder_write(bs, &data, sizeof(data));  
33. }  

​        注意，这里的status参数为0。从这里可以看出，binder_send_reply告诉Binder驱动程序执行BC_FREE_BUFFER和BC_REPLY命令，前者释放之前在binder_transaction分配的空间，地址为buffer_to_free，buffer_to_free这个地址是Binder驱动程序把自己在内核空间用的地址转换成用户空间地址再传给Service Manager的，所以Binder驱动程序拿到这个地址后，知道怎么样释放这个空间；后者告诉Binder驱动程序，它的SVC_MGR_CHECK_SERVICE操作已经完成了,要查询的服务的句柄值也是保存在data.txn.data，操作结果码是0，也是保存在data.txn.data中。

​        再来看binder_write函数：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. int binder_write(struct binder_state *bs, void *data, unsigned len)  
2. {  
3. ​    struct binder_write_read bwr;  
4. ​    int res;  
5. ​    bwr.write_size = len;  
6. ​    bwr.write_consumed = 0;  
7. ​    bwr.write_buffer = (unsigned) data;  
8. ​    bwr.read_size = 0;  
9. ​    bwr.read_consumed = 0;  
10. ​    bwr.read_buffer = 0;  
11. ​    res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);  
12. ​    if (res < 0) {  
13. ​        fprintf(stderr,"binder_write: ioctl failed (%s)\n",  
14. ​                strerror(errno));  
15. ​    }  
16. ​    return res;  
17. }  

​        这里可以看出，只有写操作，没有读操作，即read_size为0。

​        这里又是一个ioctl的BINDER_WRITE_READ操作。直入到驱动程序的binder_ioctl函数后，执行BINDER_WRITE_READ命令，这里就不累述了。

​        最后，从binder_ioctl执行到binder_thread_write函数，首先是执行BC_FREE_BUFFER命令，这个命令的执行在前面一篇文章

Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析

已经介绍过了，这里就不再累述了。

​        我们重点关注BC_REPLY命令的执行：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. int    
2. binder_thread_write(struct binder_proc *proc, struct binder_thread *thread,    
3. ​                    void __user *buffer, int size, signed long *consumed)    
4. {    
5. ​    uint32_t cmd;    
6. ​    void __user *ptr = buffer + *consumed;    
7. ​    void __user *end = buffer + size;    
8. ​    
9. ​    while (ptr < end && thread->return_error == BR_OK) {    
10. ​        if (get_user(cmd, (uint32_t __user *)ptr))    
11. ​            return -EFAULT;    
12. ​        ptr += sizeof(uint32_t);    
13. ​        if (_IOC_NR(cmd) < ARRAY_SIZE(binder_stats.bc)) {    
14. ​            binder_stats.bc[_IOC_NR(cmd)]++;    
15. ​            proc->stats.bc[_IOC_NR(cmd)]++;    
16. ​            thread->stats.bc[_IOC_NR(cmd)]++;    
17. ​        }    
18. ​        switch (cmd) {    
19. ​        ......    
20. ​        case BC_TRANSACTION:    
21. ​        case BC_REPLY: {    
22. ​            struct binder_transaction_data tr;    
23. ​    
24. ​            if (copy_from_user(&tr, ptr, sizeof(tr)))    
25. ​                return -EFAULT;    
26. ​            ptr += sizeof(tr);    
27. ​            binder_transaction(proc, thread, &tr, cmd == BC_REPLY);    
28. ​            break;    
29. ​                       }    
30. ​    
31. ​        ......    
32. ​        *consumed = ptr - buffer;    
33. ​    }    
34. ​    return 0;    
35. }   

​        又再次进入到binder_transaction函数：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. static void  
2. binder_transaction(struct binder_proc *proc, struct binder_thread *thread,  
3. struct binder_transaction_data *tr, int reply)  
4. {  
5. ​    struct binder_transaction *t;  
6. ​    struct binder_work *tcomplete;  
7. ​    size_t *offp, *off_end;  
8. ​    struct binder_proc *target_proc;  
9. ​    struct binder_thread *target_thread = NULL;  
10. ​    struct binder_node *target_node = NULL;  
11. ​    struct list_head *target_list;  
12. ​    wait_queue_head_t *target_wait;  
13. ​    struct binder_transaction *in_reply_to = NULL;  
14. ​    struct binder_transaction_log_entry *e;  
15. ​    uint32_t return_error;  
16.   
17. ​    ......  
18.   
19. ​    if (reply) {  
20. ​        in_reply_to = thread->transaction_stack;  
21. ​        if (in_reply_to == NULL) {  
22. ​            ......  
23. ​            return_error = BR_FAILED_REPLY;  
24. ​            goto err_empty_call_stack;  
25. ​        }  
26. ​        ......  
27. ​        thread->transaction_stack = in_reply_to->to_parent;  
28. ​        target_thread = in_reply_to->from;  
29. ​        ......  
30. ​        target_proc = target_thread->proc;  
31. ​    } else {  
32. ​        ......  
33. ​    }  
34. ​    if (target_thread) {  
35. ​        e->to_thread = target_thread->pid;  
36. ​        target_list = &target_thread->todo;  
37. ​        target_wait = &target_thread->wait;  
38. ​    } else {  
39. ​        ......  
40. ​    }  
41. ​      
42.   
43. ​    /* TODO: reuse incoming transaction for reply */  
44. ​    t = kzalloc(sizeof(*t), GFP_KERNEL);  
45. ​    if (t == NULL) {  
46. ​        return_error = BR_FAILED_REPLY;  
47. ​        goto err_alloc_t_failed;  
48. ​    }  
49. ​    binder_stats.obj_created[BINDER_STAT_TRANSACTION]++;  
50.   
51. ​    tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);  
52. ​    if (tcomplete == NULL) {  
53. ​        return_error = BR_FAILED_REPLY;  
54. ​        goto err_alloc_tcomplete_failed;  
55. ​    }  
56. ​    ......  
57.   
58. ​    if (!reply && !(tr->flags & TF_ONE_WAY))  
59. ​        t->from = thread;  
60. ​    else  
61. ​        t->from = NULL;  
62. ​    t->sender_euid = proc->tsk->cred->euid;  
63. ​    t->to_proc = target_proc;  
64. ​    t->to_thread = target_thread;  
65. ​    t->code = tr->code;  
66. ​    t->flags = tr->flags;  
67. ​    t->priority = task_nice(current);  
68. ​    t->buffer = binder_alloc_buf(target_proc, tr->data_size,  
69. ​        tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));  
70. ​    if (t->buffer == NULL) {  
71. ​        return_error = BR_FAILED_REPLY;  
72. ​        goto err_binder_alloc_buf_failed;  
73. ​    }  
74. ​    t->buffer->allow_user_free = 0;  
75. ​    t->buffer->debug_id = t->debug_id;  
76. ​    t->buffer->transaction = t;  
77. ​    t->buffer->target_node = target_node;  
78. ​    if (target_node)  
79. ​        binder_inc_node(target_node, 1, 0, NULL);  
80.   
81. ​    offp = (size_t *)(t->buffer->data + ALIGN(tr->data_size, sizeof(void *)));  
82.   
83. ​    if (copy_from_user(t->buffer->data, tr->data.ptr.buffer, tr->data_size)) {  
84. ​        binder_user_error("binder: %d:%d got transaction with invalid "  
85. ​            "data ptr\n", proc->pid, thread->pid);  
86. ​        return_error = BR_FAILED_REPLY;  
87. ​        goto err_copy_data_failed;  
88. ​    }  
89. ​    if (copy_from_user(offp, tr->data.ptr.offsets, tr->offsets_size)) {  
90. ​        binder_user_error("binder: %d:%d got transaction with invalid "  
91. ​            "offsets ptr\n", proc->pid, thread->pid);  
92. ​        return_error = BR_FAILED_REPLY;  
93. ​        goto err_copy_data_failed;  
94. ​    }  
95. ​    ......  
96.   
97. ​    off_end = (void *)offp + tr->offsets_size;  
98. ​    for (; offp < off_end; offp++) {  
99. ​        struct flat_binder_object *fp;  
100. ​        ......  
101. ​        fp = (struct flat_binder_object *)(t->buffer->data + *offp);  
102. ​        switch (fp->type) {  
103. ​        ......  
104. ​        case BINDER_TYPE_HANDLE:  
105. ​        case BINDER_TYPE_WEAK_HANDLE: {  
106. ​            struct binder_ref *ref = binder_get_ref(proc, fp->handle);  
107. ​            if (ref == NULL) {  
108. ​                ......  
109. ​                return_error = BR_FAILED_REPLY;  
110. ​                goto err_binder_get_ref_failed;  
111. ​            }  
112. ​            if (ref->node->proc == target_proc) {  
113. ​                ......  
114. ​            } else {  
115. ​                struct binder_ref *new_ref;  
116. ​                new_ref = binder_get_ref_for_node(target_proc, ref->node);  
117. ​                if (new_ref == NULL) {  
118. ​                    return_error = BR_FAILED_REPLY;  
119. ​                    goto err_binder_get_ref_for_node_failed;  
120. ​                }  
121. ​                fp->handle = new_ref->desc;  
122. ​                binder_inc_ref(new_ref, fp->type == BINDER_TYPE_HANDLE, NULL);  
123. ​                ......  
124. ​            }  
125. ​        } break;  
126.   
127. ​        ......  
128. ​        }  
129. ​    }  
130.   
131. ​    if (reply) {  
132. ​        BUG_ON(t->buffer->async_transaction != 0);  
133. ​        binder_pop_transaction(target_thread, in_reply_to);  
134. ​    } else if (!(t->flags & TF_ONE_WAY)) {  
135. ​        ......  
136. ​    } else {  
137. ​        ......  
138. ​    }  
139.   
140. ​    t->work.type = BINDER_WORK_TRANSACTION;  
141. ​    list_add_tail(&t->work.entry, target_list);  
142. ​    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;  
143. ​    list_add_tail(&tcomplete->entry, &thread->todo);  
144. ​    if (target_wait)  
145. ​        wake_up_interruptible(target_wait);  
146. ​    return;  
147.   
148. ​    ......  
149. }  

​        这次进入binder_transaction函数的情形和上面介绍的binder_transaction函数的情形基本一致，只是这里的proc、thread和target_proc、target_thread调换了角色，这里的proc和thread指的是Service Manager进程，而target_proc和target_thread指的是刚才请求SVC_MGR_CHECK_SERVICE的进程。

​        那么，这次是如何找到target_proc和target_thread呢。首先，我们注意到，这里的reply等于1，其次，上面我们提到，Binder驱动程序在唤醒Service Manager，告诉它有一个事务t要处理时，事务t虽然从Service Manager的todo队列中删除了，但是仍然保留在transaction_stack中。因此，这里可以从thread->transaction_stack找回这个等待回复的事务t，然后通过它找回target_proc和target_thread：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. in_reply_to = thread->transaction_stack;  
2. target_thread = in_reply_to->from;  
3. target_list = &target_thread->todo;  
4. target_wait = &target_thread->wait;  

​       再接着往下看，由于Service Manager返回来了一个Binder引用，所以这里要处理一下，就是中间的for循环了。这是一个BINDER_TYPE_HANDLE类型的Binder引用，这是前面设置的。先把t->buffer->data的内容转换为一个struct flat_binder_object对象fp，这里的fp->handle值就是这个Service在Service Manager进程里面的引用值了。接通过调用binder_get_ref函数得到Binder引用对象struct binder_ref类型的对象ref：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. struct binder_ref *ref = binder_get_ref(proc, fp->handle);  

​       这里一定能找到，因为前面MediaPlayerService执行IServiceManager::addService的时候把自己添加到Service Manager的时候，会在Service Manager进程中创建这个Binder引用，然后把这个Binder引用的句柄值返回给Service Manager用户空间。

​       这里面的ref->node->proc不等于target_proc，因为这个Binder实体是属于创建MediaPlayerService的进程的，而不是请求这个服务的远程接口的进程的，因此，这里调用binder_get_ref_for_node函数为这个Binder实体在target_proc创建一个引用：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. struct binder_ref *new_ref;  
2. new_ref = binder_get_ref_for_node(target_proc, ref->node);  

​       然后增加引用计数：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. binder_inc_ref(new_ref, fp->type == BINDER_TYPE_HANDLE, NULL);  

​      这样，返回数据中的Binder对象就处理完成了。注意，这里会把fp->handle的值改为在target_proc中的引用值：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. fp->handle = new_ref->desc;  

​     这里就相当于是把t->buffer->data里面的Binder对象的句柄值改写了。因为这是在另外一个不同的进程里面的Binder引用，所以句柄值当然要用新的了。这个值最终是要拷贝回target_proc进程的用户空间去的。

​      再往下看：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. if (reply) {  
2. ​     BUG_ON(t->buffer->async_transaction != 0);  
3. ​     binder_pop_transaction(target_thread, in_reply_to);  
4. } else if (!(t->flags & TF_ONE_WAY)) {  
5. ​     ......  
6. } else {  
7. ​     ......  
8. }  

​       这里reply等于1，执行binder_pop_transaction函数把当前事务in_reply_to从target_thread->transaction_stack队列中删掉，这是上次调用binder_transaction函数的时候设置的，现在不需要了，所以把它删掉。

​       再往后的逻辑就跟前面执行binder_transaction函数时候一样了，这里不再介绍。最后的结果就是唤醒请求SVC_MGR_CHECK_SERVICE操作的线程：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. if (target_wait)  
2. ​     wake_up_interruptible(target_wait);  

​       这样，Service Manger回复调用SVC_MGR_CHECK_SERVICE请求就算完成了，重新回到frameworks/base/cmds/servicemanager/binder.c文件中的binder_loop函数等待下一个Client请求的到来。事实上，Service Manger回到binder_loop函数再次执行ioctl函数时候，又会再次进入到binder_thread_read函数。这时个会发现thread->todo不为空，这是因为刚才我们调用了：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. list_add_tail(&tcomplete->entry, &thread->todo);  

​       把一个工作项tcompelete放在了在thread->todo中，这个tcompelete的type为BINDER_WORK_TRANSACTION_COMPLETE，因此，Binder驱动程序会执行下面操作：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. switch (w->type) {    
2. case BINDER_WORK_TRANSACTION_COMPLETE: {    
3. ​    cmd = BR_TRANSACTION_COMPLETE;    
4. ​    if (put_user(cmd, (uint32_t __user *)ptr))    
5. ​        return -EFAULT;    
6. ​    ptr += sizeof(uint32_t);    
7. ​    
8. ​    list_del(&w->entry);    
9. ​    kfree(w);    
10. ​        
11. ​    } break;    
12. ​    ......    
13. }    

​       binder_loop函数执行完这个ioctl调用后，才会在下一次调用ioctl进入到Binder驱动程序进入休眠状态，等待下一次Client的请求。

​      上面讲到调用请求SVC_MGR_CHECK_SERVICE操作的线程被唤醒了，于是，重新执行binder_thread_read函数：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. static int    
2. binder_thread_read(struct binder_proc *proc, struct binder_thread *thread,    
3. ​                   void  __user *buffer, int size, signed long *consumed, int non_block)    
4. {    
5. ​    void __user *ptr = buffer + *consumed;    
6. ​    void __user *end = buffer + size;    
7. ​    
8. ​    int ret = 0;    
9. ​    int wait_for_proc_work;    
10. ​    
11. ​    if (*consumed == 0) {    
12. ​        if (put_user(BR_NOOP, (uint32_t __user *)ptr))    
13. ​            return -EFAULT;    
14. ​        ptr += sizeof(uint32_t);    
15. ​    }    
16. ​    
17. retry:    
18. ​    wait_for_proc_work = thread->transaction_stack == NULL && list_empty(&thread->todo);    
19. ​    
20. ​    ......    
21. ​    
22. ​    if (wait_for_proc_work) {    
23. ​        ......    
24. ​    } else {    
25. ​        if (non_block) {    
26. ​            if (!binder_has_thread_work(thread))    
27. ​                ret = -EAGAIN;    
28. ​        } else    
29. ​            ret = wait_event_interruptible(thread->wait, binder_has_thread_work(thread));    
30. ​    }    
31. ​        
32. ​    ......    
33. ​    
34. ​    while (1) {    
35. ​        uint32_t cmd;    
36. ​        struct binder_transaction_data tr;    
37. ​        struct binder_work *w;    
38. ​        struct binder_transaction *t = NULL;    
39. ​    
40. ​        if (!list_empty(&thread->todo))    
41. ​            w = list_first_entry(&thread->todo, struct binder_work, entry);    
42. ​        else if (!list_empty(&proc->todo) && wait_for_proc_work)    
43. ​            w = list_first_entry(&proc->todo, struct binder_work, entry);    
44. ​        else {    
45. ​            if (ptr - buffer == 4 && !(thread->looper & BINDER_LOOPER_STATE_NEED_RETURN)) /* no data added */    
46. ​                goto retry;    
47. ​            break;    
48. ​        }    
49. ​    
50. ​        ......    
51. ​    
52. ​        switch (w->type) {    
53. ​        case BINDER_WORK_TRANSACTION: {    
54. ​            t = container_of(w, struct binder_transaction, work);    
55. ​                                      } break;    
56. ​        ......    
57. ​        }    
58. ​    
59. ​        if (!t)    
60. ​            continue;    
61. ​    
62. ​        BUG_ON(t->buffer == NULL);    
63. ​        if (t->buffer->target_node) {    
64. ​            ......    
65. ​        } else {    
66. ​            tr.target.ptr = NULL;    
67. ​            tr.cookie = NULL;    
68. ​            cmd = BR_REPLY;    
69. ​        }    
70. ​        tr.code = t->code;    
71. ​        tr.flags = t->flags;    
72. ​        tr.sender_euid = t->sender_euid;    
73. ​    
74. ​        if (t->from) {    
75. ​            ......    
76. ​        } else {    
77. ​            tr.sender_pid = 0;    
78. ​        }    
79. ​    
80. ​        tr.data_size = t->buffer->data_size;    
81. ​        tr.offsets_size = t->buffer->offsets_size;    
82. ​        tr.data.ptr.buffer = (void *)t->buffer->data + proc->user_buffer_offset;    
83. ​        tr.data.ptr.offsets = tr.data.ptr.buffer + ALIGN(t->buffer->data_size, sizeof(void *));    
84. ​    
85. ​        if (put_user(cmd, (uint32_t __user *)ptr))    
86. ​            return -EFAULT;    
87. ​        ptr += sizeof(uint32_t);    
88. ​        if (copy_to_user(ptr, &tr, sizeof(tr)))    
89. ​            return -EFAULT;    
90. ​        ptr += sizeof(tr);    
91. ​    
92. ​        ......    
93. ​    
94. ​        list_del(&t->work.entry);    
95. ​        t->buffer->allow_user_free = 1;    
96. ​        if (cmd == BR_TRANSACTION && !(t->flags & TF_ONE_WAY)) {    
97. ​            ......    
98. ​        } else {    
99. ​            t->buffer->transaction = NULL;    
100. ​            kfree(t);    
101. ​            binder_stats.obj_deleted[BINDER_STAT_TRANSACTION]++;    
102. ​        }    
103. ​        break;    
104. ​    }    
105. ​    
106. done:    
107. ​    ......    
108. ​    return 0;    
109. }    

​        就是从下面这个调用：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. ret = wait_event_interruptible(thread->wait, binder_has_thread_work(thread));  

​       被唤醒过来了。在while循环中，从thread->todo得到w，w->type为BINDER_WORK_TRANSACTION，于是，得到t。从上面可以知道，Service Manager返回来了一个Binder引用和一个结果码0回来，写在t->buffer->data里面，现在把t->buffer->data加上proc->user_buffer_offset，得到用户空间地址，保存在tr.data.ptr.buffer里面，这样用户空间就可以访问这个数据了。由于cmd不等于BR_TRANSACTION，这时就可以把t删除掉了，因为以后都不需要用了。

​       执行完这个函数后，就返回到binder_ioctl函数，执行下面语句，把数据返回给用户空间：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {    
2. ​    ret = -EFAULT;    
3. ​    goto err;    
4. }    

​       接着返回到用户空间IPCThreadState::talkWithDriver函数，最后返回到IPCThreadState::waitForResponse函数，最终执行到下面语句：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)    
2. {    
3. ​    int32_t cmd;    
4. ​    int32_t err;    
5. ​    
6. ​    while (1) {    
7. ​        if ((err=talkWithDriver()) < NO_ERROR) break;    
8. ​            
9. ​        ......    
10. ​    
11. ​        cmd = mIn.readInt32();    
12. ​    
13. ​        ......    
14. ​    
15. ​        switch (cmd) {    
16. ​        ......    
17. ​        case BR_REPLY:    
18. ​            {    
19. ​                binder_transaction_data tr;    
20. ​                err = mIn.read(&tr, sizeof(tr));    
21. ​                LOG_ASSERT(err == NO_ERROR, "Not enough command data for brREPLY");    
22. ​                if (err != NO_ERROR) goto finish;    
23. ​    
24. ​                if (reply) {    
25. ​                    if ((tr.flags & TF_STATUS_CODE) == 0) {    
26. ​                        reply->ipcSetDataReference(    
27. ​                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),    
28. ​                            tr.data_size,    
29. ​                            reinterpret_cast<const size_t*>(tr.data.ptr.offsets),    
30. ​                            tr.offsets_size/sizeof(size_t),    
31. ​                            freeBuffer, this);    
32. ​                    } else {    
33. ​                        ......  
34. ​                    }    
35. ​                } else {    
36. ​                    ......   
37. ​                }    
38. ​            }    
39. ​            goto finish;    
40. ​    
41. ​        ......    
42. ​        }    
43. ​    }    
44. ​    
45. finish:    
46. ​    ......    
47. ​    return err;    
48. }    

​       注意，这里的tr.flags等于0，这个是在上面的binder_send_reply函数里设置的。接着就把结果保存在reply了：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. reply->ipcSetDataReference(    
2. ​       reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),    
3. ​       tr.data_size,    
4. ​       reinterpret_cast<const size_t*>(tr.data.ptr.offsets),    
5. ​       tr.offsets_size/sizeof(size_t),    
6. ​       freeBuffer, this);    

​       我们简单看一下Parcel::ipcSetDataReference函数的实现：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. void Parcel::ipcSetDataReference(const uint8_t* data, size_t dataSize,  
2. ​    const size_t* objects, size_t objectsCount, release_func relFunc, void* relCookie)  
3. {  
4. ​    freeDataNoInit();  
5. ​    mError = NO_ERROR;  
6. ​    mData = const_cast<uint8_t*>(data);  
7. ​    mDataSize = mDataCapacity = dataSize;  
8. ​    //LOGI("setDataReference Setting data size of %p to %lu (pid=%d)\n", this, mDataSize, getpid());  
9. ​    mDataPos = 0;  
10. ​    LOGV("setDataReference Setting data pos of %p to %d\n", this, mDataPos);  
11. ​    mObjects = const_cast<size_t*>(objects);  
12. ​    mObjectsSize = mObjectsCapacity = objectsCount;  
13. ​    mNextObjectHint = 0;  
14. ​    mOwner = relFunc;  
15. ​    mOwnerCookie = relCookie;  
16. ​    scanForFds();  
17. }  

​        上面提到，返回来的数据中有一个Binder引用，因此，这里的mObjectSize等于1，这个Binder引用对应的位置记录在mObjects成员变量中。

​        从这里层层返回，最后回到BpServiceManager::checkService函数中：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. virtual sp<IBinder> BpServiceManager::checkService( const String16& name) const  
2. {  
3. ​    Parcel data, reply;  
4. ​    data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());  
5. ​    data.writeString16(name);  
6. ​    remote()->transact(CHECK_SERVICE_TRANSACTION, data, &reply);  
7. ​    return reply.readStrongBinder();  
8. }  

​        这里就是从：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. remote()->transact(CHECK_SERVICE_TRANSACTION, data, &reply);  

​        返回来了。我们接着看一下reply.readStrongBinder函数的实现：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. sp<IBinder> Parcel::readStrongBinder() const  
2. {  
3. ​    sp<IBinder> val;  
4. ​    unflatten_binder(ProcessState::self(), *this, &val);  
5. ​    return val;  
6. }  

​        这里调用了unflatten_binder函数来构造一个Binder对象：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. status_t unflatten_binder(const sp<ProcessState>& proc,  
2. ​    const Parcel& in, sp<IBinder>* out)  
3. {  
4. ​    const flat_binder_object* flat = in.readObject(false);  
5. ​      
6. ​    if (flat) {  
7. ​        switch (flat->type) {  
8. ​            case BINDER_TYPE_BINDER:  
9. ​                *out = static_cast<IBinder*>(flat->cookie);  
10. ​                return finish_unflatten_binder(NULL, *flat, in);  
11. ​            case BINDER_TYPE_HANDLE:  
12. ​                *out = proc->getStrongProxyForHandle(flat->handle);  
13. ​                return finish_unflatten_binder(  
14. ​                    static_cast<BpBinder*>(out->get()), *flat, in);  
15. ​        }          
16. ​    }  
17. ​    return BAD_TYPE;  
18. }  

​        这里的flat->type是BINDER_TYPE_HANDLE，因此调用ProcessState::getStrongProxyForHandle函数：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)  
2. {  
3. ​    sp<IBinder> result;  
4.   
5. ​    AutoMutex _l(mLock);  
6.   
7. ​    handle_entry* e = lookupHandleLocked(handle);  
8.   
9. ​    if (e != NULL) {  
10. ​        // We need to create a new BpBinder if there isn't currently one, OR we  
11. ​        // are unable to acquire a weak reference on this current one.  See comment  
12. ​        // in getWeakProxyForHandle() for more info about this.  
13. ​        IBinder* b = e->binder;  
14. ​        if (b == NULL || !e->refs->attemptIncWeak(this)) {  
15. ​            b = new BpBinder(handle);   
16. ​            e->binder = b;  
17. ​            if (b) e->refs = b->getWeakRefs();  
18. ​            result = b;  
19. ​        } else {  
20. ​            // This little bit of nastyness is to allow us to add a primary  
21. ​            // reference to the remote proxy when this team doesn't have one  
22. ​            // but another team is sending the handle to us.  
23. ​            result.force_set(b);  
24. ​            e->refs->decWeak(this);  
25. ​        }  
26. ​    }  
27.   
28. ​    return result;  
29. }  

​       这里我们可以看到，ProcessState会把使用过的Binder远程接口（BpBinder）缓存起来，这样下次从Service Manager那里请求得到相同的句柄（Handle）时就可以直接返回这个Binder远程接口了，不用再创建一个出来。这里是第一次使用，因此，e->binder为空，于是创建了一个BpBinder对象：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. b = new BpBinder(handle);   
2. e->binder = b;  
3. if (b) e->refs = b->getWeakRefs();  
4. result = b;  

​       最后，函数返回到IMediaDeathNotifier::getMediaPlayerService这里，从这个语句返回：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. binder = sm->getService(String16("media.player"));  

​        这里，就相当于是：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. binder = new BpBinder(handle);  

​        最后，函数调用：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. sMediaPlayerService = interface_cast<IMediaPlayerService>(binder);  

​        到了这里，我们可以参考一下前面一篇文章

浅谈Android系统进程间通信（IPC）机制Binder中的Server和Client获得Service Manager

，就会知道，这里的interface_cast实际上最终调用了IMediaPlayerService::asInterface函数：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. android::sp<IMediaPlayerService> IMediaPlayerService::asInterface(const android::sp<android::IBinder>& obj)  
2. {  
3. ​    android::sp<IServiceManager> intr;  
4. ​    if (obj != NULL) {               
5. ​        intr = static_cast<IMediaPlayerService*>(   
6. ​            obj->queryLocalInterface(IMediaPlayerService::descriptor).get());  
7. ​        if (intr == NULL) {  
8. ​            intr = new BpMediaPlayerService(obj);  
9. ​        }  
10. ​    }  
11. ​    return intr;   
12. }  

​        这里的obj就是BpBinder，而BpBinder::queryLocalInterface返回NULL，因此就创建了一个BpMediaPlayerService对象：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6633311#) [copy](http://blog.csdn.net/luoshengyang/article/details/6633311#)

1. intr = new BpMediaPlayerService(new BpBinder(handle));  

​        因此，我们最终就得到了一个BpMediaPlayerService对象，达到我们最初的目标。

​        有了这个BpMediaPlayerService这个远程接口之后，MediaPlayer就可以调用MediaPlayerService的服务了。

​        至此，Android系统进程间通信（IPC）机制Binder中的Client如何通过Service Manager的getService函数获得Server远程接口的过程就分析完了，Binder机制的学习就暂告一段落了。

​        不过，细心的读者可能会发现，我们这里介绍的Binder机制都是基于C/C++语言实现的，但是我们在编写应用程序都是基于[Java](http://lib.csdn.net/base/java)语言的，那么，我们如何使用Java语言来使用系统的Binder机制来进行进程间通信呢？这就是下一篇文章要介绍的内容了，敬请关注。