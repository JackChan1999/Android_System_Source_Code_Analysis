在[Android](http://lib.csdn.net/base/android)系统中，Content Provider作为应用程序四大组件之一，它起到在应用程序之间共享数据的作用，同时，它还是标准的数据访问接口。前面的一系列文章已经分析过[android](http://lib.csdn.net/base/android)应用程序的其它三大组件（Activity、Service和Broadcast Receiver）了，本文将简要介绍Content Provider组件在Android应用程序设计中的地位，为进一步学习打好基础。

《Android系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

我们知道，在Android系统上，每一个应用程序都有一个独立的用户ID。为什么要给每一个应用程序分配一个独立的用户ID呢？这是为了保护各个应用程序的数据不被其它应用程序恶意破坏而设计的。Android系统是基于[Linux](http://lib.csdn.net/base/linux)内核来开发的，而在[linux](http://lib.csdn.net/base/linux)系统中，每一个文件除了文件本身的数据之外，还具有一定的属性，其中最重要的就是文件权限了。所谓的文件权限，就是指对文件的读、写和执行权限。此外，Linux还是一个多用户的[操作系统](http://lib.csdn.net/base/operatingsystem)，因此，Linux将系统中的每一个文件都与一个用户以及用户组关联起来，基于不同的用户而赋予不同的文件权限。只有当一个用户对某个文件拥有相应的权限时，才能执行相应的操作，例如，只有当一个用户对某个文件拥有读权限时，这个用户才可以调用read系统调用来读取这个文件的内容。Android系统继承了Linux系统管理文件的方法，为每一个应用程序分配一个独立的用户ID和用户组ID，而由这个应用程序创建出来的数据文件就赋予相应的用户以及用户组读写的权限，其余用户则无权对该文件进行读写。

如果我们通过adb shell命令连上模拟器，切换到/data/data目录下，就可以看到很多以应用程序包（package）命名的文件夹，这些文件夹里面存放的就是各个应用程序的数据文件，例如，如果我们进入到Android系统日历应用程序数据目录com.android.providers.calendar下的databases文件中，会看到一个用来保存日历数据的[数据库](http://lib.csdn.net/base/mysql)文件calendar.db，它的权限设置如下所示：

```
root@android:/data/data/com.android.providers.calendar/databases # ls -l  
-rw-rw---- app_17   app_17      33792 2011-11-07 15:50 calendar.db  
```
在前面的十字符**-rw-rw----**中，最前面的符号**-**表示这是一个普通文件，接下来的三个字符**rw-**表示这个文件的所有者对这个文件可读可写不可执行，再接下来的三个字符**rw-**表示这个文件的所有者所在的用户组的用户对这个文件可读可写不可执行，而最后的三个字符---表示其它的用户对这个文件不可读写也不可执行，因为这是一个数据文件，所认所有用户都不可以执行它是正确的。在接下来的两个**app_17**字符串表示这个文件的所有者和这个所有者所在的用户组的名称均为**app_17**，这是应用程序在安装的时候系统分配的，在不同的系统上，这个字符串可能是不一样，不过它所表示的意义是一样的。这意味着只有用户ID为**app_17**或者用户组ID为**app_17**的那些进程才可以对这个calendar.db文件进行读写操作。我们通过执行终端上执行ps命令来查看一下哪个进程的用户ID为**app_17**：

```
root@android:/ # ps  
USER     PID   PPID  VSIZE  RSS     WCHAN    PC         NAME  
root      1     0     272    184   c009f230 0000875c S /init  
...      ...   ...    ...    ...      ...    ...         ...  
app_17   295   35    107468 21492 ffffffff afd0c38c  S com.android.providers.calendar  
..       ...   ...    ...    ...      ...    ...         ...  
root     556   527   892    332   00000000 afd0b24c  R ps  
```
这里我们看到，正是这个日历应用程序com.android.providers.calendar进程的用户ID为

*app_17。*

这样，就验证了我们前面的分析了：只有这个日历应用程序com.android.providers.calendar才可以对这个calendar.db文件进行读写操作。这个日历应用程序com.android.providers.calendar其实是由一个Content Provider组件来实现的，在下一篇文章中，我们将实现一个自己的Content Provider，然后再对Content Provider的实现原理进行详细分析。

Android系统对应用程序的数据文件作如此严格的保护会不会有点过头了呢？如果一个应用程序想要读写另外一个应用程序的数据文件时，应该怎么办呢？举一个典型的应用场景，我们在开发自己的应用程序时，有时候会希望读取通讯录里面某个联系人的手机号码或者电子邮件，以便可以对这个联系人打电话或者发送电子邮件，这时候就需要读取通讯录里面的联系人数据文件了。

现在在互联网里面，都流行平台的概念，各大公司都打着开放平台的口号，来吸引第三方来为自己的平台做应用，例如，国外最流行的Fackbook开放平台和Google+开放平台等，国内的有腾讯的Q+开放平台，还有新浪微博开放平台、360的开放平台等。这些开放平台的核心就是要开放用户数据给第三方来使使用，就像前面我们说的Android系统的通讯录，它需要把自己联系人数据开放出来给其它应用程序使用。但是，这些数据都是各个平台自己的核心数据和核心竞争力，它们需要有保护地进行开放。Android系统中的Content Provider应用程序组件正是结合上面分析的这种文件权限机制来秉承这种有保护地开放自己的数据给其它应用程序使用的理念。

从另外一个观点来看，即使我们不是在做平台，而只是在做一个应用程序软件，是不是就不需要这么使用到Content Provider机制了呢？非也，现在的应用程序软件，越着公司业务的成长，越来越庞大，越来越复杂。软件工程告诉我们，我们在设计这种大型的复杂的软件的时候，需要分模块和分层次来实现各个子功能组件，使得各个模块功能以松耦合的方式组织在一起完成整个应用程序功能。这样做的好处当然就是便于我们维护和扩展应用程序的代码和功能了，以及适应复杂的业务环境。在一个大型的应用程序软件[架构](http://lib.csdn.net/base/architecture)中，从垂直的方向来看，一般都会划分为数据层、数据访问接口层以及上面的业务层。数据层用来保存数据，这些数据可以用文件的方式来组织，也可以用数据库的方式来组织，甚至可以保存在网络中；数据访问层负责向上面的业务层提供数据，而向下管理好数据层的数据；最后业务层通过数据访问层来获取一些业务相关的数据来实现自己的业务逻辑。

基于这种开放平台建设或者复杂软件架构的理念，我们得出一个Android应用程序设计的一般方法，如下图所示：

![img](http://hi.csdn.net/attachment/201111/7/0_1320687133YFYN.gif)

在这个架构中， 数据层采用数据库、文件或者网络来保存数据，数据访问层使用Content Provider来实现，而业务层就通过一些APP来实现。为了降低各个功能模块间耦合性，我们可以把业务层的各个APP和数据访问层中的Content Provider，放在不同的应用程序进程中来实现，而数据库中的数据统一由Content Provider来管理，即Content Provider拥有对这些文件直接进行读写的权限，同时，它又根据需要来有保护地把这些数据开放出来给上层的APP来使用。

那么，Content Provider又是如何把数据开放给上面的APP使用呢？一方面是这些APP没有权限读取这些数据文件，另一外面是Content Provider和这些APP是在不同的进程空间里面。回忆一下，我们在前面[Android进程间通信（IPC）机制Binder简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6618363)这一系文章中学习的Android系统中Binder进程间通信机制，不同的应用程序之间可以通过Binder进程间调用来传输数据。因此，前面关于一个应用程序应该如何来读写另外一个应用程序的数据的问题的答案就是使用Binder进程间通信机制，虽然一个应用程序不能直接读取另一个应用程序的数据，但是它却可以通过进程间通信方式来请求另一个这个应用程序给它传输数据。这样我们就解决了文件权限限制所带来的问题了。

然而，事情还不是那么简单，一般Content  Provider管理的都是大量的数据，如果在进程间传输大量的数据，效率是不是会很低下呢？这时候，前面我们在[Android系统匿名共享内存Ashmem（Anonymous Shared Memory）简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6651971)这一系列文章中学习的匿名共享内存（Anonymous Shared Memory）就派上用场了。把需要在进程间传输的数据都写到共享内存中去，然后只能通过Binder进程间通信机制来传输一个共享内存的打开文件描述符给对方就好了，是不是很简单呢。在Android系统中，Binder进程间通信机制和匿名共享内存机制结合在一起使用，真是太完美了。

Content Provider如何在应用程序之间共享数据以及它在应用程序设计中的地位就简要介绍到这里了，在接下来的四篇文章中，我们将以一个 Content Provider的应用实例来详细分析它的启动过程、数据共享原理以及数据监控机制：

[Android应用程序组件Content Provider的应用实例](http://blog.csdn.net/luoshengyang/article/details/6950440)。
[Android应用程序组件Content Provider的启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6963418)。
[Android应用程序组件Content Provider在不同应用程序之间共享数据的原理分析](http://blog.csdn.net/luoshengyang/article/details/6967204)。
[Android应用程序组件Content Provider的数据监控机制分析](http://blog.csdn.net/luoshengyang/article/details/6985171)。