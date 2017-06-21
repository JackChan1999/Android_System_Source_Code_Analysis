在大家的支持和鼓励下，《[Android](http://lib.csdn.net/base/android)系统源代码情景分析》一书得以出版了，老罗在此首先谢过大家了。本书的内容来源于博客的文章，经过大半年的整理之后，形成了初稿。在正式出版之前，又经过了三次排版以及修订，最终得到终稿。然而，老罗深知，书中的内容并不尽完美，除了错误之外总还会有许多不尽人意的地方，因此，欢迎广大读者以及国内外的专家给老罗指出，以便改进。为了达到此目的，老罗特别在此列出该书有错误的地方。

《[android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

​        现在暂时将书中出现的错误划分为三类，第一类是笔误，第二类是表达问题，第三类是技术性错误，分别使用I、II和III来表示，错误所在的页码使用字母P来表示。

​        1. P3，倒数第2行（III）：sudo add-apt-repository ppa:ferramrobert/[Java](http://lib.csdn.net/base/java)。由于License问题（[http://askubuntu.com/questions/109209/sun-java6-plugin-has-no-installation-candidate](http://askubuntu.com/questions/109209/sun-java6-plugin-has-no-installation-candidate)），官方的Sun JDK6不能在Ubuntu上发布，因此，现在从下载源ferramrobert已经下载不到JDK6来安装了，可以通过修改安装源来解决这个问题，如下所示：

​       A. sudo add-apt-repository "deb http://us.archive.ubuntu.com/ubuntu/ hardy multiverse"

​       B. sudo apt-get update

​       C. sudo apt-get install sun-java6-jre sun-java6-plugin

​       D. sudo apt-get install sun-java6-jdk

​       如果在安装过程中，碰到有依赖包未安装，就先把依赖包安装上去就行了。感谢网友@偏左和@大桥++指出，2012-11-05。

​       如果这样还不能安装成功，那就只有自己手动安装了，官方JDK下载地址：[http://www.oracle.com/technetwork/java/javase/downloads/index.html](http://www.oracle.com/technetwork/java/javase/downloads/index.html)

​       更多的环境配置信息，可以参考官方文档：[http://source.android.com/source/initializing.html](http://source.android.com/source/initializing.html)

​       2. P12页，倒数第1个自然段（I）：重新生成的Android系统镜像文件ssystem.img位于out/target/product/generic目录中。这句话里面的ssystem.img应该改为system.img。感谢网友@lewisgre指出，2015-05-19。

​       3. P20，第278行代码（III）：temp = device_create(freg_class, NULL, dev, "%s", FREG_DEVICE_FILE_NAME)。这个函数调用的参数写错了，不过歪打正着，能正常编译以及工作，应该将第四个参数设置为NULL，即为：temp = device_create(freg_class, NULL, dev, NULL, "%s", FREG_DEVICE_FILE_NAME)。

​       函数device_create的原型为：struct device *device_create(struct class *cls, struct device *parent, dev_t devt, void *drvdata, const char *fmt, ...)

​       第四个参数drvdata表示一个私有数据，它可以为任意值或者NULL，第五个参数是一个格式化字符串，用来描述设备名称，最后是一个可变参数列表，是配合第五个参数使用的。在上述错误的调用中，参数drvdata的值等于“%s”，而参数fmt的值等于FREG_DEVICE_FILE_NAME，即“freg”。由于没有可变参数列表，并且参数fmt的值不带有%s或者%d之类的格式化符号，因此，这里设置的设备名称就等于“freg”。

​       感谢网友@insoonior的指出，2012-11-14。

​       4. P20，第297行代码：printk(KERN_ALERT"Succedded to initialize freg device.\n")。这行代码中的Succedded应改为Succeeded。感谢网友@Five_Cent_Nicol指出，2013-02-28。

​       5. P26，顺数第二段第2行（I）：这些动态链接库文件的命令需要符合一定的规范。这句话中的命令应该改为命名。感谢网友@迷死人的东东指出，2012-12-07。

​       6. P30，第一段第4行（I）：它的第一个成员变量的类型为freg_device_t。这里话里面的freg_device_t应改为hw_device_t。感谢网友@hustljh指出，2014-09-19。

​       7. P33，顺数第二段（I）：USER@MACHINE:~/Android$ mmm ./hardware/libhardware/freg。应改为：USER@MACHINE:~/Android$ mmm ./hardware/libhardware/modules/freg。感谢网友@jltxgcy指出，2014-01-06。

​       8. P36页，倒数第4个自然段（I）：如果不修改设备文件/def/freg的访问权限。这句话里面的/def/freg应该改为/dev/freg。感谢网友@Lasting泉指出，2015-03-05。

​       9. P53，lightpointer.cpp的第16行（I）：printf("Destory LightClass Object.")。单词Destory拼写错误，应为Destroy。感谢网友@hengbo12345指出，2012-10-26。

​       10. P69，第118行代码（I）：printf("\nTest Froever Class: \n")。应改为：Forever。相应地，P71页的输出：“Test Froever Class:” 也改为Forever。感谢网友@Coding人生指出，2014-05-05。

​       11. P78，倒数第一段第1行和第2行（I）：在分析这个函数之前，我们首先介绍三个结构体变量log_main、log_events和log_radio，它们的类型均为struct logger。这句话中的struct logger应改为struct logger_log。感谢网友@迷死人的东东指出，2012-12-11。

​       12. P146，顺数第三段第1行（I）：这些工作项有可能属于一个进程，也有会可能属于一个进程中的某一个线程。这句话中的也有会可能应改为也有可能。感谢网友@zhouaijia8指出，2013-04-07。 

​       13. P147，倒数第9行（I）：即将一个Binder实体对象的成员变量work的值设置为BINDER_WORKD_NODE。短语BINDER_WORKD_NODE中间的单词WORKD拼写错误，多了一个字母D，应为BINDER_WORK_NODE。感谢网友@brucechan1973指出，2012-10-30。

​       14. P155，最后四段（III）：对结构体binder_transaction的成员变量from_parent和to_parent的描述有偏差。应改为：

​        ---------------------------

​     *   成员变量from_parent和to_parent分别描述一个事务所依赖的另外一个事务，以及目标线程下一个需要处理的事务。假设线程A发起了一个事务T1，需要由线程B来处理；线程B在处理事务T1时，又需要线程C先处理事务T2；线程C在处理事务T2时，又需要线程A先处理事务T3。这样，事务T1就依赖于事务T2，而事务T2又依赖于事务T3，它们的关系如下：*

~~*T1->from_parent = T2;*~~* T2->from_parent = T1;*

~~*T2->from_parent = T3; *~~*T3->from_parent = T2;*

​         对于线程A来说，它需要处理的事务有两个，分别是T1和T3，它首先要处理事务T3，然后才能处理事务T1，因此，事务T1和T3的关系如下：

*T3->to_parent = T1;*

​        考虑这样一个情景：如果线程C在发起事务T3给线程A所属的进程来处理时，Binder驱动程序选择了该进程的另外一个线程D来处理该事务，这时候会出现什么情况呢？这时候线程A就会处于空闲等待状态，什么也不能做，因为它必须要等线程D处理完成事务T3后，它才可以继续执行事务T1。在这种情况下，与其让线程A闲着，还不如把事务T3交给它来处理，这样线程D就可以去处理其他事务，提高了进程的并发性。

*        现在，关键的问题又来了——Binder驱动程序在分发事务T3给目标进程处理时，它是如何知道线程A属于目标进程，并且正在等待事务T3的处理结果的？*~~*当线程B在处理事务T2时，就会将事务T2放在其事务堆栈transaction_stack的最前端。这样当线程B发起事务T3给线程C处理时，Binder驱动程序就可以沿着线程B的事务堆栈transaction_stack向下遍历，直到发现事务T3的目标进程等于事务T1的目标进程时，它就知道线程A正在等待事务T3的处理结果了。*~~*当线程C在处理事务T2时，就会将事务T2放在其事务堆栈transaction_stack的最前端。这样当线程C发起事务T3给线程A所属的进程处理时，Binder驱动程序就可以沿着线程C的事务堆栈transaction_stack向下遍历，即沿着事务T2的成员变量from_parent向下遍历，最后就会发现事务T3的目标进程等于事务T1的目标进程，并且事务T1是由线程A发起来的，这时候它就知道线程A正在等待事务T3的处理结果了。*

​        --------------------------

​        PS：结构体binder_transaction的成员变量from_parent和to_parent可以结合P256的第31行到第40行代码块以及P267的第77行到第79行的代码块来理解。这是个比较严重的技术性错误，由此造成读者的疑惑和费解，老罗先道歉了，同时，非常感谢网友@hongbog_cd指出，2012-11-14。

​        15. P155，最后一段第4行和第5行（I）：即沿着事务T2的成员变量from、parent向下遍历，最后就会发现事务T3的目标进程等于事务T1的目标进程。这里话里面的from、parent应改为from_parent，目标进程应改为源进程。感谢网友@albert1017diu指出，2014-08-06。 

​        16. P161页，第2个自然段（I）：其中，命令协议代码BR_INCREFS和BR_DECREFS分别用来增加和减少一个Service组件的弱引用计数；而命令协议代码BR_ACQUIRE和BR_RELEASE分别用来增加和减少一个Service组件的强引用计数。这句话里面的命令协议代码应该改为返回协议代码。感谢网友@zlp1992指出，2015-09-13。

​        17. P172，顺数第一段第2行和第3行（I）：在将内核缓冲区new_buffer加入到目标进程proc的空闲内缓冲区红黑树中之前。这句话中的空闲内缓冲区应改为空闲内核缓冲区。感谢网友@zhouaijia8指出，2013-04-07。

​        18. P225，顺数第二段文字后面的目录结构（I）：～/Android/frameworks/base/cmd。cmd后面少了一个s，应改为cmds。感谢网友@zhouaijia8指出，2013-04-09。

​        19. P240，倒数第二段，P696，倒数第五段（III）：这个内核缓冲区的大小被Binder库设置为1016

Kb

、第5行代码创建的匿名共享内存块的大小就为16

Kb

。这两句话要表达的单位是

千字节

，应该使用

KB

来表示，

Kb

里面的

b

是

bit

的意思，这里使用不当。感谢网友@hongbog_cd指出，2012-11-17。

​        20. P242，倒数第一段最后3行（III）：接下来第5行到第8行代码就会在列表mHandleToObject的第N到第（handle+1-N）个位置上分别插入一个handle_entry结构体，最后第11行就可以将与句柄值handle对应的handle_entry结构体返回给调用者。（handle+1-N）描述是要插入的handle_entry结构体的个数，因此，这句话要表达的意思其实是从第N个位置开始，插入（handle+1-N）个handle_entry结构体到列表mHandleToObject中，因此，这句话里面的第N到第（handle+1-N）个位置应该改为第N到第handle个位置。感谢网友@hongbog_cd指出，2012-11-19。

​        21. P243，顺数第一段第1行（I）：回到ProcessState类的成员函数getContextObject中。这句话中的getContextObject应改为getStrongProxyForHandle。感谢网友@zhouaijia8指出，2013-04-09。

​        22. P244，顺数第三段第1行和第2行（I）：Service进程在启动时，会首先将它里面的Service组件注册到Service Manager中。这句话开头的Service应改为Server。感谢网友@zhouaijia8指出，2013-04-09。

​        23. P252，图5-23（I）：binder: FregService->localBinder()；cookie: FregService->getWeakRefs()。写反了，应改为：binder:FregService->getWeakRefs()；cookie: FregService->localBinder()。感谢网友@jltxgcy指出，2014-05-07。

​        24. P270，顺数第一段第2行和第3行（I）：它等同于在前面5.1.1小节中介绍的结构体flat_binder_objec。flat_binder_objec后面少了一个t，应该改为flat_binder_object。感谢网友@hongbog_cd指出，2012-11-19。

​        25. P276，顺数第四段第1行（I）：回到函数svcmgr_handler中。这句中的svcmgr_handler应改为do_add_service。感谢网友@zhouaijia8指出，2013-04-10。

​        26. P279页，第4个自然段（I）：第7行将binder_write_read结构体bwr的输出缓冲区write_buffer设置为由参数data所描述的一块用户空间缓冲区。这句话里面的输出缓冲区应该改为输入缓冲区。感谢网友@jianghu1059指出，2015-09-18。

​        27. P329，顺数第四段第2行（I）：当这些小块的内存处理解锁状态时。这句话中的处理应该改为处于。感谢网友@nanfeng5651指出，2012-12-13。

​        28. P340，倒数第五段第2行和第3行（I）：如果是，那么就5行就调用函数lru_del将它从全局列表ashmem_lru_list中删除。这句话中的就5行应改为第5行。感谢网友@zhouaijia8指出，2013-04-11。

​        29. P350，顺数第二段和倒数第一段（I）：IMemoryBase类定义了MemoryHeapBase服务接口、并且实现了IMemoryBase接口的四个成员函数。这两句话中的IMemoryBase拼写错误，应改为IMemoryHeap。感谢网友@sulliy指出，2012-11-06。

​        30. P378，顺数第3行（I）：IMemoryFile接口定义了两个成员函数getFileDescriptor和setValue。短语IMemoryFile写错了，应改为IMemoryService。感谢网友@hongbog_cd指出，2012-11-14。

​        31. P399，顺数第2行和第4行（I）：ManActivity组件。短语ManActivity中的单词Man拼写错误，应改为MainActivity。感谢网友@herodie指出，2012-11-01。

​        32. P400，第二段和第三段（I）：action = "android.intent.action.Main"、要启动的Activity组件的Action名称和Category名称分别为"android.intent.action.Main"和"android.intent.category.LAUNCHER"。单词Main应全部大写MAIN。感谢网友@herodie指出，2012-11-01。

​        33. P418，Step 21（I）：ActivityStack.resumeTopActivityLokced。这里话里面的Lokced应改为Locked。感谢网友@SunShinXin指出，2014-12-31。

​        34. P420，第16行代码（III）：app = new ProcessRecordLocked(null, info, processName)。这里是调用成员函数newProcessRecordLocked来创建一个ProcessRecord对象，而不是直接创建一个ProcessRecordLocked对象。相应地，接下来的一段描述文字“第16行就会根据指定的名称以及用户ID来创建一个ProcessRecordLocked对象”中的ProcessRecordLocked应改为ProcessRecord。感谢网友@android迷指出，2012-12-05。

​        35. P497，顺数第二段第1行和第2行（I）：第8行到第28行代码在LoadedApk类的mReceivers中检查是否存在一个以广播接收者c为关键字的ReceiverDispatcher对象rd。这句话中的广播接收者c应改为为广播接收者r。感谢网友@nanfeng5651指出，2012-12-19。

​        36. P513，顺数第一段第2行和第3行（I）：那么第11行就将ActivityManagerService类的成员变量mBroadcastsScheduled的值设置为true。这句话中的true应改为false。感谢网友@zhouaijia8指出，2013-04-17。

​        37. P528，顺数第一段最后两行（I）：其中，前者用来描述一个vdn.shy.luo.article数据集合；即一个博客文章集合，后者用来描述一个vdn.shy.luo.article数据，即一个博客文章条目。这句话中的两个vdn.shy.luo.article应改为vnd.shy.luo.article，分号；改为逗号，。感谢网友@nanfeng5651指出，2012-12-20。

​        38. P528，倒数第一段前面两行（I）：其中，DB_TABLE和DB_VERSION用来描述这个SQLite[数据库](http://lib.csdn.net/base/mysql)的名称和版本号。这句话中的DB_TABLE应该改为DB_NAME。感谢网友@nanfeng5651指出，2012-12-20。

​        39. P539，倒数第一段（I）：ArticlesAdapter类的员函数getArticleById和getArticleByPos分别根据一个博客文章的ID值和位置值从ArticlesProvider组件获得指定的博客文章条目。这句话中的员函数应改为成员函数。感谢网友@zhouaijia8指出，2013-04-17。

​        40. P560，图10-9（I）：左上角的ApplicationThread。应改为：ActivityThread。感谢网友@hongbog_cd指出，2014-01-23。

​        41. P573，顺数第二段第3行（I）：最后就可以通过这个接口来访问ArtivlesProviders组件中的博客文章信息。这句话中的ArtivlesProviders应改为ArticlesProviders。感谢网友@zhouaijia8指出，2013-04-17。

​        42. P573，顺数第三段第1行和第2行（I）：我们继续分析MainActivity组件访问运行在另外一个应用程序进程中的ArtivlesProvider组件的博客文章信息的过程。这句话中的ArtivlesProviders应改为ArticlesProviders。感谢网友@zhouaijia8指出，2013-04-17。

​        43. P588，倒数第三段第1行（I）：参数memobj指向了一个Java层的Binder代理对象。这句话中的memobj应改为memObj。感谢网友@nanfeng5651指出，2012-12-24。

​        44. P693，倒数第二段倒数第1行和第2行（I）：将前面所创建的WindowState对象win保存在Window管理服务Window ManagerService的成员变量mWindows所描述的一个应用程序窗口列表中。这句话中的Window ManagerService有一个多余的空格，改为WindowManagerService。感谢网友@nanfeng5651指出，2012-12-27。

​        45. P713，倒数第三段第1行（I）：参数sacnKey和keyCode保存的分别是当前所发生的键盘事件所对应的扫描码和键盘码。这句话中的sacnKey应改为scanCode。感谢网友@nanfeng5651指出，2012-12-28。

​        46. P716，顺数第四段第1行（I）：第一种情况是在将一个新发生的键盘事件添加到待分键盘事件队列之前。这句话中的待分键盘事件队列应改为待分发键盘事件队列。感谢网友@nanfeng5651指出，2012-12-28。

​        47. P730，倒数第四段第2行和第3行（I）：第30行代码获得它的一个引用，并且保存在变量inputHandlerObjGlobal中。这句话中的inputHandlerObjGlobal应改为inputHandlerObjLocal。感谢网友@nanfeng5651指出，2013-01-03。

​        48. P768，倒数第一段（II）：HandlerThread类的成员函数quit的实现如下所示。这段话描述有误，改为：如前所示，HandlerThread类的成员函数quit首先获得前面在子线程中所创建的一个Looper对象，然后再调用这个Looper对象的成员函数quit来退出子线程。Looper类的成员函数quit的实现如下所示。感谢网友@nanfeng5651指出，2013-01-04。

​        49. P771，倒数第五段（I）：最后，参数workerQueue和threadFactory分别用来描述一个ThreadPoolExecutor线程池的工作任务队列和线程创建工厂。这句话中的workerQueue应改为workQueue。感谢网友@nanfeng5651指出，2013-01-04。

​        50. P807，顺数第二段第3行和第4行（I）：在这种情况下，参数pkg所描述的一个应用程序所获得的资源权访问权限就与它所共享的[Linux](http://lib.csdn.net/base/linux)用户所具有的资源权访问权限相同。这句话中的资源权访问权限应改为资源访问权限。感谢网友@nanfeng5651指出，2013-01-05。

​        51. P812，顺数第三段第2行和第3行（I）：如果不存在，那么第22行代码就会将文件/data/system/packages.xml重命令为/data/system/packages-backup.xml。这句话中的重命令应改为重命名。感谢网友@nanfeng5651指出，2013-01-05。

​        52. P828，倒数第二段第1行（I）：这一步执行完成之后，这回到前面的Step 10中。这句话中的这回应改为返回。感谢网友@nanfeng5651指出，2013-01-05。

**老罗的新浪微博：http://weibo.com/shengyangluo，欢迎关注！**

​     39. P539，倒数第一段（I）：ArticlesAdapter类的员函数getArticleById和getArticleByPos分别根据一个博客文章的ID值和位置值从ArticlesProvider组件获得指定的博客文章条目。这句话中的员函数应改为成员函数。感谢网友@zhouaijia8指出，2013-04-17。

​     52. P279页，第4个自然段（I）：第7行将binder_write_read结构体bwr的输出缓冲区write_buffer设置为由参数data所描述的一块用户空间缓冲区。这句话里面的输出缓冲区应该改为输入缓冲区。感谢网友@jianghu1059指出，2015-09-18。

​     53. P52，顺数第一段第一行（I）：sp类也是一个模块类。这句话中的模块类应改为模板类。感谢网友@码农小c指出，2016-05-25。

​     54. P63，顺数第一段第一行（I）：wp类是一个模块类。这句话中的模块类应改为模板类。感谢网友@码农小c指出，2016-05-25。