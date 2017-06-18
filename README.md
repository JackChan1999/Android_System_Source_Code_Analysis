虽然Android系统自2008年9月发布第一个版本1.0以来，截至2011年10月发布最新版本4.0，一共存在十多个版本，但是据官方统计，截至2012年3月5日，占据首位的是Android 2.3，市场占有率达到66.5%；其次是Android 2.2，市场占有率为25.3%；第三位是Android 2.1，市场占有率为6.6%；而最新发布的Android 3.2和Android 4.0的市场占有率只有3.3%和2%。因此，在本书中，我们选择了Android 2.3的源代码来分析Android系统的实现，一是因为从目前来说，它的基础架构是最稳定的；二是因为它是使用最广泛的。

### 本书内容

全书分为初识Android系统篇、Android专用驱动系统篇和Android应用程序框架篇三个部分。

初识Android系统篇包含三个章节的内容，主要介绍Android系统的基础知识。第1章介绍与Android系统有关的参考书籍，以及Android源代码工程环境的搭建方法;第2章介绍Android系统的硬件抽象层;第3章介绍Android系统的智能指针。读者可能会觉得奇怪，为什么一开始就介绍Android系统的硬件抽象层呢?因为涉及硬件，它似乎是一个深奥的知识点。其实不然，Android系统的硬件抽象层无论是从实现上，还是从使用上，它的层次都是非常清晰的，而且从下到上涵盖了整个Android 系统，包括Android系统在用户空间和内核空间的实现。内核空间主要涉及硬件驱动程序的编写方法，而用户空间涉及运行时库层、应用程序框架层及应用程序层。因此，尽早学习Android系统的硬件抽象层，有助于我们从整体上去认识Android系统，以便后面可以更好地分析它的源代码。在分析Android系统源代码的过程中，经常会碰到智能指针，第3章我们就重点分析Android系统智能指针的实现原理，也是为了后面可以更好地分析Android系统源代码。

Android专用驱动系统篇包含三个章节的内容。我们知道，Android系统是基于Linux内核来开发的，但是由于移动设备的CPU和内存配置都要比PC低，因此，Android系统并不是完全在Linux内核上开发的，而是在Linux内核里面添加了一些专用的驱动模块来使它更适合于移动设备。这些专用的驱动模块同时也形成了Android系统的坚实基础，尤其是Logger日志驱动程序、Binder进程间通信驱动程序，以及Ashmem匿名共享内存驱动程序，它们在Android系统中被广泛地使用。在此篇中，我们分别在第4章、第5章和第6章分析Logger日志系统、Binder进程间通信系统和Ashmem共享内存系统的实现原理，为后面深入分析Android应用程序的框架打下良好的基础。

Android应用程序框架篇包含十个章节的内容。我们知道，在移动平台中，Android系统、iOS系统和Windows Phone系统正在形成三足鼎立之势，谁的应用程序更丰富、质量更高、用户体验更好，谁就能取得最终的胜利。因此，每个平台都在尽最大努力吸引第三方开发者来为其开发应用程序。这就要求平台必须提供良好的应用程序架构，以便第三方开发者可以将更多的精力集中在应用程序的业务逻辑上，从而开发出数量更多、质量更高和用户体验更好的应用程序。在此篇中，我们将从组件、进程、消息和安装四个维度来分析Android应用程序的实现框架。第7章到第10章分析Android应用程序四大组件Activity、Service、Broadcast Receiver和Content Provider的实现原理;第11章和第12章分析Android应用程序进程的启动过程；第13章到第15章分析Android应用程序的消息处理机制;第16章分析Android应用程序的安装和显示过程。学习了这些知识之后，我们就可以掌握Android系统的精髓了。

### 本书特点

本书从初学者的角度出发，结合具体的使用情景，在纵向和横向上对Android系统的源代码进行了全面、深入、细致的分析。在纵向上，采用从下到上的方式，分析的源代码涉及了Android系统的内核层（Linux Kernel）、硬件抽象层（HAL）、运行时库层（Runtime）、应用程序框架层（Application Framework）以及应用程序层（Application），这有利于读者从整体上掌握Android系统的架构。在横向上，从Android应用程序的组件、进程、消息以及安装四个角度出发，全面地剖析了Android系统的应用程序框架层，这有利于读者深入地理解Android应用程序的架构以及运行原理。

[《Android系统源代码情景分析》一书勘误](http://blog.csdn.net/luoshengyang/article/details/8116866)

### 目录

[第一篇 初识Android系统](http://0xcc0xcd.com/p/books/978-7-121-18108-5/s1.php)

[第一章 准备知识](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c1.php)

[1.1 Linux内核参考书籍](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c11.php)

[1.2 Android应用程序参考书籍](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c12.php)

[1.3 下载、编译和运行Android源代码](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c13.php)

[1.3.1 下载Android源代码](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c131.php)

[1.3.2 编译Android源代码](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c132.php)

[1.3.3 运行Android模拟器](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c133.php)

[1.4 下载、编译和运行Android内核源代码](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c14.php)

[1.4.1 下载Android内核源代码](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c141.php)

[1.4.2 编译Android内核源代码](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c142.php)

[1.4.3 运行Android模拟器](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c143.php)

[1.5 开发第一个Android应用程序](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c15.php)

[1.6 单独编译和打包Android应用程序模块](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c16.php)

[1.6.1 导入单独编译模块的mmm命令](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c161.php)

[1.6.2 单独编译Android应用程序模块](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c162.php)

[1.6.3 重新打包Android系统镜像文件](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c163.php)

[第二章 硬件抽象层](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c2.php)

[2.1 开发Android硬件驱动程序](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c21.php)

[2.1.1 实现内核驱动程序模块](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c211.php)

[2.1.2 修改内核Kconfig文件](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c212.php)

[2.1.3 修改内核Makefile文件](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c213.php)

[2.1.4 编译内核驱动程序模块](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c214.php)

[2.1.5 验证内核驱动程序模块](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c215.php)

[2.2 开发C可执行程序验证Android硬件驱动程序](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c22.php)

[2.3 开发Android硬件抽象层模块](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c23.php)

[2.3.1 硬件抽象层模块编写规范](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c231.php)

[2.3.1.1 硬件抽象层模块文件命名规范](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c2311.php)

[2.3.1.2 硬件抽象层模块结构体定义规范](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c2312.php)

[2.3.2 编写硬件抽象层模块接口](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c232.php)

[2.3.3 硬件抽象层模块的加载过程](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c233.php)

[2.3.4 处理硬件设备访问权限问题](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c234.php)

[2.4 开发Android硬件访问服务](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c24.php)

[2.4.1 定义硬件访问服务接口](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c241.php)

[2.4.2 实现硬件访问服务](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c242.php)

[2.4.3 实现硬件访问服务的JNI方法](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c243.php)

[2.4.4 启动硬件访问服务](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c244.php)

[2.5 开发Android应用程序来使用硬件访问服务](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c25.php)

[第三章 智能指针](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c3.php)

[3.1 轻量级指针](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c31.php)

[3.1.1 实现原理分析](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c311.php)

[3.1.2 使用实例分析](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c312.php)

[3.2 强指针和弱指针](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c32.php)

[3.2.1 强指针的实现原理分析](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c321.php)

[3.2.2 弱指针的实现原理分析](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c322.php)

[3.2.3 应用实例分析](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c323.php)

[第二篇 Android专用驱动系统](http://0xcc0xcd.com/p/books/978-7-121-18108-5/s2.php)

[第四章 Logger日志系统](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c4.php)

[4.1 Logger日志格式](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c41.php)

[4.2 Logger日志驱动程序](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c42.php)

[4.2.1 基础数据结构](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c421.php)

[4.2.2 日志设备的初始化过程](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c422.php)

[4.2.3 日志设备文件的打开过程](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c423.php)

[4.2.4 日志记录的读取过程](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c424.php)

[4.2.5 日志记录的写入过程](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c425.php)

[4.3 运行时库层日志库](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c43.php)

[4.4 C/C++日志写入接口](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c44.php)

[4.5 Java日志写入接口](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c45.php)

[4.6 Logcat工具分析](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c46.php)

[4.6.1 基础数据结构](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c461.php)

[4.6.2 初始化过程](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c462.php)

[4.6.3 日志记录的读取过程](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c463.php)

[4.6.4 日志记录的输出过程](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c464.php)

[第五章 Binder进程间通信系统](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c5.php)

[5.1 Binder驱动程序](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c51.php)

[5.1.1 基础数据结构](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c511.php)

[5.1.2 Binder设备的初始化过程](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c512.php)

[5.1.3 Binder设备文件的打开过程](http://0xcc0xcd.com/p/books/978-7-121-18108-5/c513.php)

5.1.4 设备文件内存映射过程

5.1.5 内核缓冲区管理

5.1.5.1 分配内核缓冲区

5.1.5.2 释放内核缓冲区

5.1.5.3 查询内核缓冲区

5.2 Binder进程间通信库

5.3 Binder进程间通信应用实例

5.4 Binder对象引用计数技术

5.4.1 Binder本地对象的生命周期

5.4.2 Binder实体对象的生命周期

5.4.3 Binder引用对象的生命周期

5.4.4 Binder代理对象的生命周期

5.5 Binder对象死亡通知机制

5.5.1 注册死亡接收通知

5.5.2 发送死亡接收通知

5.5.3 注销死亡接收通知

5.6 Service Manager的启动过程

5.6.1 打开和映射Binder设备文件

5.6.2 注册成为Binder上下文管理者

5.6.3 循环等待Client进程请求

5.7 Service Manager代理对象接口的获取过程

5.8 Service的启动过程

5.8.1 注册Service组件

5.8.1.1 封装通信数据为Parcel对象

5.8.1.2 发送和处理BC_TRANSACTION命令协议

5.8.1.3 发送和处理BR_TRANSACTION返回协议

5.8.1.4 发送和处理BC_REPLY命令协议

5.8.1.5 发送和处理BR_REPLY返回协议

5.8.2 循环等待Client进程请求

5.9 Service代理对象接口的获取过程

5.10 Binder进程间通信机制的Java实现接口

5.10.1 获取Service Manager的Java代理对象接口

5.10.2 AIDL服务接口解析

5.10.3 Java服务的启动过程

5.10.4 获取Java服务的代理对象接口

5.10.5 Java服务的调用过程

第六章 Ashmem匿名共享内存系统

6.1 Ashmem驱动程序

6.1.1 相关数据结构

6.1.2 设备初始化过程

6.1.3 设备文件打开过程

6.1.4 设备文件内存映射过程

6.1.5 内存块的锁定和解锁过程

6.1.6 解锁状态内存块的回收过程

6.2 运行时库cutils的匿名共享内存接口

6.3 匿名共享内存的C++访问接口

6.3.1 MemoryHeapBase

6.3.1.1 Server端的实现

6.3.1.2 Client端的实现

6.3.2 MemoryBase

6.3.2.1 Server端的实现

6.3.2.2 Client端的实现

6.3.3 应用实例

6.4 匿名共享内存的Java访问接口

6.4.1 MemoryFile

6.4.2 应用实例

6.5 匿名共享内存的共享原理分析

第三篇 Android应用程序框架篇

第七章 Activity组件的启动过程

7.1 Activity组件应用实例

7.1 Activity组件应用实例

7.2 根Activity的启动过程

7.3 Activity在进程内的启动过程

7.4 Activity在新进程中的启动过程

第八章 Service组件的启动过程

8.1 Service组件应用实例

8.2 Service在新进程中的启动过程

8.3 Service在进程内的绑定过程

第九章 Android系统广播机制

9.1 广播应用实例

9.2 广播接收者的注册过程

9.3 广播的发送过程

第十章 Content Provider组件的实现原理

10.1 Content Provider组件应用实例

10.1.1 ArticlesProvider

10.1.2 Article

10.2 Content Provider组件的启动过程

10.3 Content Provider组件的数据共享原理

10.4 Content Provider组件的数据更新通知机制

10.4.1 内容观察者的注册过程

10.4.2 数据更新的通知过程

第十一章 Zygote和System进程的启动过程

11.1 Zygote进程的启动脚本

11.2 Zygote进程的启动过程

11.3 System进程的启动过程

第十二章 Android应用程序进程的启动过程

12.1 应用程序进程的创建过程

12.2 Binder线程池的启动过程

12.3 消息循环的创建过程

第十三章 Android应用程序的消息处理机制

13.1 创建线程消息队列

13.2 线程消息循环过程

13.3 线程消息发送过程

13.4 线程消息处理过程

第十四章 Android应用程序的键盘消息处理机制

14.1 InputManager的启动过程

14.1.1 创建InputManager

14.1.2 启动InputManager

14.1.3 启动InputDispatcher

14.1.4 启动InputReader

14.2 InputChannel的注册过程

14.2.1 创建InputChannel

14.2.2 注册Server端InputChannel

14.2.3 注册当前激活窗口

14.2.4 注册Client端InputChannel

14.3 键盘消息的分发过程

14.3.1 InputReader处理键盘事件

14.3.2 InputDispatcher分发键盘事件

14.3.3 当前激活的窗口获得键盘消息

14.3.4 InputDispatcher获得键盘事件处理完成通知

14.4 InputChannel的注销过程

14.4.1 销毁应用程序窗口

14.4.2 注销Client端InputChannel

14.4.3 注销Server端InputChannel

第十五章 Android应用程序线程的消息循环模型

15.1 应用程序主线程消息循环模型

15.2 界面无关的应用程序子线程消息循环模型

15.3 界面相关的应用程序子线程消息循环模型

第十六章 Android应用程序的安装和显示过程

16.1 应用程序的安装过程

16.2 应用程序的显示过程