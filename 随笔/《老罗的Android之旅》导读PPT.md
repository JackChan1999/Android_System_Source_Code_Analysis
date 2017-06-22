虽然好几个月没更新博客了，但是老罗一直有在准备可以分享的东西的。除了早前在微博分享Android4.2相关技术之外，这次还特意准备了13个PPT，总结之前所研究过的东西。内容从[Android](http://lib.csdn.net/base/Android)组件设计思想，到[Android](http://lib.csdn.net/base/android)源码开发和调试环境搭建，再到Android专用驱动和应用程序[架构](http://lib.csdn.net/base/architecture)等。可以作为《老罗的Android之旅》博客和《Android系统源代码情景分析》一书的导读，希望对大家有帮助。

《Android系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

废话就不多说了，直入主题，以下就是这个12个PPT的内容介绍以及下载地址。

### 1. Android组件设计思想

Android应用开发的哲学是把一切都看作是组件。把应用程序组件化的好处是降低模块间的耦合性，同时提高模块的复用性。Android的组件设计思想与传统的组件设计思想最大的区别在于，前者不依赖于进程。也就是说，进程即使由于内存紧张被强行杀掉了，但是运行在里面的组件还是存在的。这样就可以在组件再次需要使用时，原地满血复活，就像什么都没发生过一样。这种设计思想非常适合内存较小的移动设备。

理解Android组件设计思想，对Android应用程序架构会有更好的认识。这一节讲Android组件化设计的背景、理念、原则，以及Android在OS级别上提供的组件化支持，其中还会包含一个实验来验证这种组件化设计思想，可以对Android系统有一个高层次的抽象理解。

下载地址：http://download.csdn.net/detail/luoshengyang/6439629

### 2. Android源代码开发和调试环境搭建

Android源代码开发环境与SDK开发环境相比，优势是可以查看和调试系统源代码，包括[Java](http://lib.csdn.net/base/java)代码和C/C++代码。这对应用开发也是非常有用的，因为在开发中碰到疑难杂症时可以跟踪到系统内部去定位问题。对于涉及到C/C++代码的开发，例如JNI开发和安全相关开发，更加建议在Android源代码开发环境进行，这样就可以利用gdb以及gdbclient工具进行调试。

这个PPT主要讲Android源代码下载、编译和运行，以及C/C++、Java代码的调试。

下载地址：http://download.csdn.net/detail/luoshengyang/6439633

### 3. Android系统架构概述

Android系统 = [Linux](http://lib.csdn.net/base/linux)内核 + Android运行时。

Android系统使用的[linux](http://lib.csdn.net/base/linux)内核包含了一些专用驱动，例如Logger、Binder、Ashmem、Wakelock、Low-Memory Killer和Alarm等，这些Android专用驱动构成了Android运行时的基石。Android运行时从下到上又包括了HAL层、应用程序框架层和应用程序层。HAL层主要是为规避GPL而设计的，它将将硬件驱动分成内核空间和用户空间两部分，其中用户空间两部分采用的是商业友好的Apache License。应用程序框架层主要包括系统服务，例如组件管理服务、应用程序安装服务、窗口管理服务、多媒体服务和电信服务等。应用程序框架进一步又分为C/C++和Java两个层次，Java代码运行Dalvik虚拟机之上，并且通过JNI方法和C/C++交互。应用程序层主要就是由四大组件Activity、Service、Broadcast Receiver和Content Provider构成，它们是应用开发的基础。

这个PPT从一个通用的应用程序架构开始，概述Android系统的专用驱动、HAL、关键服务、Dalvik、窗口机制和四大组件等。这个PPT 作为前面第1个PPT的延续，帮助进一步了解Android系统的具体实现。

下载地址：[http://download.csdn.net/detail/luoshengyang/6439637](http://download.csdn.net/detail/luoshengyang/6439637)

### 4. Android硬件抽象层

Android硬件抽象层从开发到使用有一个清晰的层次。这个层次恰好对应了Android系统的架构层次，它向下涉及到Linux内核，向上涉及到应用程序框架层的服务，以及应用程序层对它的使用。Android硬件抽象层模块的开发本身也遵循一定的规范。有了这个规范之后，系统就可以对它进行自动加载，方便上层的使用。

这个PPT通过一个具体的实例来分析Android硬件抽象层的开发、[测试](http://lib.csdn.net/base/softwaretest)和使用，它在帮助我们理解Android系统架构的同时，也能教会我们如何在Android源代码环境中开发C/C++代码。

下载地址：[http://download.csdn.net/detail/luoshengyang/6440375](http://download.csdn.net/detail/luoshengyang/6440375)

### 5. Android专用驱动

Android专用驱动构成了Android运行时的基石。从技术上来讲，Android专用驱动也是整个Android系统的亮点，特别是Binder驱动。Binder是一种进程间通信机制(IPC)，它与传统的IPC机制对比，最大的特点是高效，因为通信数据在两个进程之间只需要执行一次拷贝即可。Binder在Android系统里面使用得非常广泛以及频繁。在涉及到比较大的通信数据时，Binder通常还结合另外一个驱动Ashmem来使用。Ashmem是一个共享内存驱动，它与传统的共享内存相比，最大的特点是它是通过文件描述符来描述的，并且可以动态地进行分块管理。动态分块管理的目的是可以将部分不再使用了的内存交回给系统，非常适合内存较小的移动设备使用。另外一个专用驱动Logger是一个日志驱动，它与传统的日志系统对比，特点是日志是记录在内核空间而非文件中，这样就可以提高日志的读写速度。

这个PPT讲Logger、Binder和Ashmem三个Android专用驱动的实现原理。由于这三个驱动在Android源代码里面用得非常广泛和频繁，因此理解它们的实现原理，就可以掌握Android的精华。这对以后阅读Android系统的其它代码，也是非常有帮助的。

下载地址：[http://download.csdn.net/detail/luoshengyang/6439643](http://download.csdn.net/detail/luoshengyang/6439643)

### 6. Android应用程序进程管理

Android系统里面的应用程序进程有一个特点，那就是它们是被系统托管的。也就是说，系统根据需要来创建进程以及回收进程。进程创建发生在组件启动时，它们是由Zygote进程负责创建。Zygote进程是由系统中的第一个进程init负责启动。此外，用来运行各种系统服务的System Server进程也是由Zygote进程创建的。进程回收发生在内存紧张时，由Low Memory Killer执行。此外，组件管理服务ActivityManagerService和窗口管理服务WindowManagerService也会在适当的时候主动进行进程回收。每一个应用程序进程根据运行情况被赋予优先级，当需要回收进程的时候，就按照优先级从低到高的顺序进行回收。

这个PPT讲Android应用程序进程的启动和回收，主要涉及到Zygote进程、System Server进程，以及组件管理服务ActivityManagerService、窗口服务WindowManagerService，还有专用驱动Low Memory Killer。通过了解Android系统对应用程序进程的管理，我们就能更清楚应用程序的运行机制。 

下载地址：[http://download.csdn.net/detail/luoshengyang/6439645](http://download.csdn.net/detail/luoshengyang/6439645)

### 7. Android应用程序消息机制

Android应用程序与传统的PC应用程序一样，都是消息驱动的。也就是说，在Android应用程序主线程中，所有函数都是在一个消息循环中执行的。Android应用程序其它线程，也可以像主线程一样，拥有消息循环。Android应用程序主线程是一个特殊的线程，因为它同时也是UI线程以及触摸屏、键盘等输入事件处理线程。主线程对消息循环很敏感，一旦发生阻塞，就会影响UI的流畅度，甚至发生ANR问题。

这个PPT讲Android应用程序线程消息循环原理，主要涉及到Handler和Looper两个类，以及根据消息循环的不同使用场景，总结出三种线程使用模型。掌握Android应用程序消息处理机制，有助于我们熟练地使用同步和异步编程，提高程序的运行性能。

下载地址：[http://download.csdn.net/detail/luoshengyang/6439647](http://download.csdn.net/detail/luoshengyang/6439647)

### 8. Android应用程序输入事件分发和处理机制

在Android应用程序中，有一类特殊的消息，是专门负责与用户进行交互的，它们就是触摸屏和键盘等输入事件。触摸屏和键盘事件是统一由系统输入管理器InputManager进行分发的。也就是说，InputManager负责从硬件接收输入事件，然后再将接收到的输入事件分发当前激活的窗口处理。此外，InputManager也能接收模拟的输入事件，用来模拟用户触摸和点击等事件。当前激活的窗口所运行在的线程接收到InputManager分发过来的输入事件之后，会将它们封装成输入消息，然后交给当前获得焦点的控件处理。

这个PPT讲Android应用程序输入事件的分发和处理过程，主要涉及到输入管理InputManager、输入事件监控线程InputReader、输入事件分发线程InputDispatcher，以及应用程序主线程消息循环。

下载地址：[http://download.csdn.net/detail/luoshengyang/6440247](http://download.csdn.net/detail/luoshengyang/6440247)

### 9. Android应用程序UI架构

Android系统采用一种称为Surface的UI架构为应用程序提供用户界面。在Android应用程序中，每一个Activity组件都关联有一个或者若干个窗口，每一个窗口都对应有一个Surface。有了这个Surface之后，应用程序就可以在上面渲染窗口的UI。最终这些已经绘制好了的Surface都会被统一提交给Surface管理服务SurfaceFlinger进行合成，最后显示在屏幕上面。无论是应用程序，还是SurfaceFlinger，都可以利用GPU等硬件来进行UI渲染，以便获得更流畅的UI。在Android应用程序UI架构中，还有一个重要的服务WindowManagerService，它负责统一管理协调系统中的所有窗口，例如管理窗口的大小、位置、打开和关闭等。

这个PPT讲Android应用程序的Surface机制，阐述Activity、Window和View的关系，以及应用程序、WindowManagerService和SurfaceFlinger协作完成UI渲染的过程。

下载地址：[http://download.csdn.net/detail/luoshengyang/6439651](http://download.csdn.net/detail/luoshengyang/6439651)

### 10. Android应用程序资源管理框架

Android应用程序主要由代码和资源组成。资源主要就是指那些与UI相关的东西，例如UI布局、字符串和图片等。代码和资源分开可以使得应用程序在运行时根据实际需要来组织UI。这样就可使得应用程序只需要编译一次，就可以支持不同的UI布局。这种特性使得应用程序在运行时可以适应不同的屏幕大小和密度，以及不同的国家和语言等。资源在Android应用程序编译的过程中，也会被编译成二进制格式。这是为了压缩资源存储空间，以及加快运行时的资源解析速度。Android应用程序在运行的时候，资源管理器AssetManager和Resources会根据当前的机器设置，即屏幕大小、密度、方向，以及国家、地区语言的信息，查找正确的资源，并且进行解析，最后将它们渲染在UI上。

这个PPT讲Android应用程序资源的编译、打包，以及它们在运行时的查找、解析过程。了解Android应用程序资源管理框架，有助于我们更好地开发出能够适配多种机型的应用程序。

下载地址：[http://download.csdn.net/detail/luoshengyang/6439653](http://download.csdn.net/detail/luoshengyang/6439653)

### 11. Dalvik虚拟机

Android应用程序是运行在Dalvik虚拟机里面的，并且每一个应用程序对应有一个单独的Dalvik虚拟机实例。Android应用程序中的Dalvik虚拟机实例实际上是从Zygote进程的地址空间拷贝而来的，这样就可以加快Android应用程序的启动速度。Dalvik虚拟机与Java虚拟机共享有差不多的特性，例如，它们都是解释执行，并且支持即时编译（JIT）、垃圾收集（GC）、Java本地方法调用（JNI）和Java远程调试协议（JDWP）等，差别在于两者执行的指令集是不一样的，并且前者的指令集是基本寄存器的，而后者的指令集是基于堆栈的。

这个PPT讲Dalvik虚拟机的内存管理、垃圾收集、即时编译、Java本地调用、进程和线程管理等。理解Dalvik虚拟机的上述实现细节，有助于在运行时修改程序的行为，例如，拦截Java函数的调用。

下载地址：[http://download.csdn.net/detail/luoshengyang/6439657](http://download.csdn.net/detail/luoshengyang/6439657)

### 12. Android安全机制

Android应用程序是运行在一个沙箱中。这个沙箱是基于Linux内核提供的用户ID（UID）和用户组ID（GID）来实现的。Android应用程序在安装的过程中，安装服务PackageManagerService会为它们分配一个唯一的UID和GID，以及根据应用程序所申请的权限，赋予其它的GID。有了这些UID和GID之后，应用程序就只能限访问特定的文件，一般就是只能访问自己创建的文件。此外，Android应用程序在调用敏感的API时，系统检查它在安装的时候会没有申请相应的权限。如果没有申请的话，那么访问也会被拒绝。对于有root权限的应用程序，则不受上述沙箱限制。此外，有root权限的应用程序，还可以通过Linux的ptrace注入到其它应用程序进程，以及系统进程，进行各种函数调用拦截。

这个PPT的计划是讲代码加壳、注入和拦截技术的，包括：

(1). SO注入。也就是从一个进程向另外一个进程注入一个SO文件，通过该注入的SO文件就可以实现函数拦截功能。

(2). SO加壳。加壳的目的自然就是加大别人对自己的C/C++代码进行静态逆向难度了，这个技术的关键是要实现一个能纯内存操作的Linker了。也就是说，解密后的SO文件内容是保存在一个内存缓冲区的，然后再针对该内存缓冲区进行解析和链接，最终形成一段可执行的代码。这个过程不会产生任何文件供别人做静态分析。

(3). C/C++函数GOT拦截。通过修改SO的GOT项来实现函数拦截。这个技术的特点是简单和稳定，但是不足之处于它是针对函数的调用方进行拦截的，而不是针对函数本身的实现来进行拦截的。这样当我们想对某一个函数进行拦截的时候，就必须要检查进程内所有的模块，然后对调用了目标函数的模块的相关GOT 项进行修改。此外，如果某一个模块是通过动态SO加载技术（dlopen、dlsym）来调用目标函数的话，GOT拦截就失效了，因为动态SO加载技术不会产生GOT项。

(4). C/C++函数INLINE拦截。这种方法是直接对目标函数的前面几条指令进行修改，用来实现拦截技术。INLINE拦截没有上述GOT拦截的缺点，但是它的实现会复杂很多。由于绝大部分Android设备都是基于ARM架构，因此这里只讨论ARM架构的C/C++函数INLINE拦截。ARMl架构主要分为ARM和THUMB两种指令集，也就是在Android设备上运行的C/C++函数分为ARM和THUMB两种类型。对于ARM指令集的函数，对它们进行拦截至少需要修改头8个字节；对于THUMB指令集，对它们进行拦截至少需要修改头12个字节。无论ARM指令还是THUMB指令函数，我们要修改的头8个字节或者12个字节都很容易碰到跳转或者PC相对寻址指令，这样就需要对指令进行重定位。这个重定位工作相当于繁重和麻烦，得实现一个ARM和THUMB指令解析库才行。不像X86的函数INLINE拦截，只需要函数的头5个字节即可，而且这5个字节几乎都是堆栈相关的操作，不会涉及到跳转或者PC相对寻址指令。

(5). DEX注入。在SO注入的基础上，要对目标进程进行DEX注入是相当简单的，通过DexClassLoader即可实现。

(6). DEX加壳。DEX加壳与SO加壳一样，都要求在解密之后，能够进行纯内存操作，中间不要产生任何和DEX或者ODEX文件，否则的话，就会给别提供静态分析的机会，这样就失去了加壳的目的。

(7). Java函数拦截。与C/C++函数拦截相对，Java函数拦截要优雅得多，因为所有的Java函数都是通过虚拟机来执行的。Dalvik虚拟机执行的函数分为Java和Native两种，它们都是使用Method结构体来描述。当一个Method结构体描述的是一个Java函数时，它有一个成员变量就指向该Java函数的方法区。而当一个Method结构体描述的是一个Native函数，它有一个成员变量指向该Native函数的地址。因此，主要我们能将一个用来描述Java函数的Method结构体修改为一个指向Native函数的Method结构体，就可以骗过Dalvik虚拟机来执行我们所指定的Native函数，从而实现拦截。

以上7个技术点涵盖了Android安全的攻与防基础。在这些基础上不仅可以保护我们自己的代码，还可以对别人的代码进行攻击。

下载地址：[http://download.csdn.net/detail/luoshengyang/6888251](http://download.csdn.net/detail/luoshengyang/6888251)

### 13. APK防反编译

我们的APK实际上就是一个ZIP压缩文件，里面包含有一个classes.dex，我们编译后生成的程序代码就全部在那里了，通过apktool等工具可以轻松地将它们反编译成smali代码。有了这些反编译出来的smali代码之后，我们就可以轻松地了解别人的APK使用的一些技术或者直接修改别人的APK。由于这些APK反编译工具的存在，我们迫切地希望能有方法去防止别人来反编译我们的APK，从而保护自己的商业机密和利益。

这个PPT讲了三种APK防反编译技术：1）添加非法指令；2）隐藏敏感代码；3）伪APK加密技术。此外，还探讨了更高级的Dex和Native加壳技术来防止别人反编译我们的APK。

下载地址：[http://download.csdn.net/detail/luoshengyang/6888253](http://download.csdn.net/detail/luoshengyang/6888253)