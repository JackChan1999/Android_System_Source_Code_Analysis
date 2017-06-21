 经过两年的时间，终于完成对[Android](http://lib.csdn.net/base/android)系统的研究了。[android](http://lib.csdn.net/base/android)是一个博大精深的系统，老罗不敢说自己精通了（事实上最讨厌的就是说自己精通神马神马的了，或者说企业说要招聘精通神马神马的人才），但是至少可以说打通了整个Android系统，从最上面的应用层，一直到最下面的[Linux](http://lib.csdn.net/base/linux)内核，炼就的是一种内功修养。这篇文章和大家一起分享这两年研究Android系统的历程，以此感谢大家一直以来的支持和鼓励。

《Android系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

​        以下是本文的提纲：

​        **1. 理念**

​        **2. 里程碑**

​        **3. 看过的书**

​        **4. 研究过的内容**

​        **5. 将来要做的事情**

​        它们涵盖了老罗这两年一直想要和大家分享的内容。好了，不说废话了，直入主题。

​        一. 理念

​        这里说的理念是说应该带什么样的心态去研究一个系统。古人说书中自的颜如玉，书中自有黄金屋，我想说代码里也有颜如玉和黄金屋，所以老罗希望大家都能“Read The Fucking Source Code”。再者，对于优秀的开源项目来说，不去读一下它的源代码，简直就是暴殄天物啊。那么，读代码有什么好处呢？太多了，除了可以学到别人的优秀代码、[架构](http://lib.csdn.net/base/architecture)之外，最重要的是，我们能从中找到答案，从而可以解决自己项目上的燃眉之急。

​        我们在项目中碰到问题的时候，通常第一反应都是到网上去搜索答案。但是有时候有些问题，网络并不能给出满意的答案。这时候就千万不要忘了你所拥有的一个大招——从代码中找答案！当然，从代码中找答案说起来是轻松，但是等到直正去找时，可能就会发现云里雾里，根本不知道那些代码在说什么东东，甚至连自己想要看的源代码文件都不知道在哪里。这就要求平时就要养成读代码的习惯，不要临时抱佛脚。有时候临时抱佛脚是能解决问题，但是千万不能抱着这种侥幸心里，掌握一门技术还是需要踏踏实实地一步一步走。

​        胡克其实在牛顿之前，就发现了万有引力定律，并且推导出了正确的公式。但由于数学不好，他只能勉强解释行星绕日的圆周运动，而没有认识到支配天体运行的力量是“万有”的。后来数学狂人牛顿用微积分圆满地解决了胡克的问题，并且把他提出的力学三条基本定律推广到了星系空间，改变了自从亚里士多德以来公认的天地不一的旧观点，被科学界奉为伟大的发现。胡克大怒，指责牛顿剽窃了他的成果。牛顿尖酸刻薄的回敬：是啊，我他妈还真是站在巨人的肩膀上呢！

​        我们有理由相信像牛顿、乔布斯之类的狂人，不用站在巨人的肩膀上也能取得瞩目的成就。但是，我们不是牛顿，也不是乔布斯，所以在看代码之前，还是找一些前人总结的资料来看看吧。拿Android系统来说，你在至少得懂点[linux](http://lib.csdn.net/base/linux)内核基础吧！所以在看Android源代码之前，先找些Linux内核的经典书籍来看看吧，骚年！后面老罗会推荐一些书籍给大家。

​        另外，我们知道，现在的互联网产品，讲究的是快速迭代。Android系统自第一个版本发布以来，到现在已经经历了很多版本呢？那么我们应该如何去选择版本来阅读呢？一般来说，就是选择最新的版本来阅读了。不过随着又有新版本的源代码的发布，我们所看的源代码就会变成旧版本。这时候心里就会比较纠结：是应该继续看旧的代码，还是去追新版本的代码呢？就当是看连续剧，一下子跳到前面去，可能就不知道讲什么了。其实版本就算更新得再快，基础的东西也是不会轻易变化的。我们看代码时，要抱着的一个目的就是弄懂它的骨架和脉络。毕竟对于一个系统来说，它是有很多细节的，我们无法在短时间把它们都完全吃透。但是主要我们掌握了它的骨架和脉络，以后无论是要了解它的什么细节，都可以很轻轻地找到相关的源文件，并且可以很容易进入主题。

​        坦白说，对于Android系统，很多细节我也不了解。所以有时候你们可以看到，在博客文章后面的评论上，有些同学问的一些比较具体的问题，我是没有回复的。一来是我不懂，二来是我也没有时间去帮这些同学去扒代码来看。这也是在文章一开头，我就说自己没有精通Android系统的原因。但是请相信，主要你熟悉Android系统的代码，并且有出现问题的现场，顺藤摸瓜跟着代码走下去，并且多一点耐心和细心，是可以解决问题的！

​        关于Android版本的问题，相信大家都知道我现在的文章都是基于2.3来写的。很多同学都说我out了，现在都4.2了，甚至4.3或者5.0都要出来了，还在看2.3。我想说的是，主要你掌握了它的骨架和脉络，无论版本上怎么变化，原理都是一样的，这就是以不变应万变之道。因此，我就一直坚持研究2.3，这样可以使得前前后后研究的东西更连贯一致，避免分散了自己的精力。如果还有疑问的话，后面我讲到Android的UI架构时，就会简单对比一下4.2和2.3的不同，其实就会发现，基本原理还是一样的！

​        说到Android系统的骨架和脉络，也有同学抱怨我的文章里面太多代码细节了，他们希望我可以抽象一下，用高度概括的语言或者图像来勾勒出每一个模块的轮廓。我想说的是，如果你不看代码，不了解细节，即使我能够用概括的语言或者图像来勾勒出这样的轮廓出来，也许这个轮廓只有我才能看得懂。

​        我在真正开始看Android系统的源代码之前，也是有这样的想法，希望能有一张图来清楚地告诉我Android系统的轮廓，例如，HAL为什么要将驱动划分成用户空间和内核空间两部分，为什么说Binder是所有IPC机制效率最高的。我确实是从网上得到抽象的资料来解释这两个问题，但是这些资料对我来说，还是太抽象了，以至于我有似懂非懂的感觉，实际上就是不懂！就是因为这样，激发了我要从代码中找答案的念头！现在当我回过头来这些所谓抽象的轮廓时，我就清楚地知道它讲的是什么了。

​        所以古人云“天将降大任于斯人也，必先苦其心志，劳其筋骨，饿其体肤”是有道理的，因为只有亲身经历过一些磨难后得到的东西才是真实的！

​        好了，关于理念的问题，就完了，这里再做一下总结：

​        **1. 从代码中找答案——Read The Fucking Source Code。**

​        **2. 以不变应万变——坚持看一个版本的代码直至理清它的骨架和脉络。**

​        二. 里程碑

​        研究Android 2.3期间，主要是经历了以下五个时间点，如图1所示：

![img](http://img.blog.csdn.net/20130528005735616)

图1 研究Android 2.3的里程碑

​         从2011年06月21日第一篇博客文章开始，到2013年06月03日结束对Android 2.3的研究，一共是差不多两年的时间，一个从无到有的过程。其中，最痛苦的莫过于是2011年12月下旬到2012年06月12日这6个多月的时间里面，整理了2011年12月下旬前的所有博客文章，形成了《Android系统源代码情景分析》一书，并且最终在2012年10月下旬正式上市。

​        总的来说，就是在两年的时间里面，获得了以下的两个产出： 

​        **1. 《老罗的Android之旅》博客专栏93篇文章，1857224次访问，4156条评论，13440积分，排名154。**

​        **2. 《Android系统源代码情景分析》一书3大篇16章，830页，1570000字。**

​        以上产出除了能帮助到广大的网友之外，也让自己理清了Android系统的骨架和脉络。这些骨架和脉络接下来再总结。2013年06月03日之后，将何去何从？接下来老罗也会简单说明。

​        三. 看过的书 

​        在2011年06月21日开始写博客之前，其实已经看过不少的书。在2011年06月21日之后，也一边写博客一边看过不少的书。这个书单很长，下面我主要分类列出一些主要的和经典的。

​        **语言：**

​        《深度探索C++对象模型》，对应的英文版是《Inside C+++ Object Model》

​        **程序编译、链接、加载：**

​        《链接器和加载器》，对应的英文版是《Linker and Loader》

​        《程序员的自我修养：链接、装载和库》

​        **操作系统：**

​        《Linux内核设计与实现》，对应的英文版是《Linux Kernel Development》

​        《深入理解Linux内核》，对应的英文版是《Understanding the Linux Kernel》

​        《深入Linux内核架构》，对应的英文版是《Professional Linux Kernel Architecture》

​        《Linux内核源代码情景分析》

​         **网络：**

​        《Linux网络体系结构：Linux内核中网络协议的设计与实现》，对应的英文版是《The Linux Networking Architecture: Design and Implementation of Network Protocols in the Linux Kernel》

​        《深入理解LINUX网络技术内幕》，对应的英文版是《 Understanding Linux Network Internals》

​        **设备驱动：**

​        《Linux设备驱动程序》，对应的英文版是《Linux Device Drivers》

​        《精通Linux设备驱动程序开发》，对应的英文版是《Essential Linux Device Drivers》

​        **虚拟机：**

​        《[Java ](http://lib.csdn.net/base/java)SE 7虚拟机规范》

​        《深入[Java](http://lib.csdn.net/base/java)虚拟机》，对应的英文版是《Inside the [java ](http://lib.csdn.net/base/java)Virtual Machine》

​        《[Oracle](http://lib.csdn.net/base/oracle) JRockit: The Definitive Guide》

​        **嵌入式：**

​        《嵌入式Linux开发》，对应的英文版是《Embedded Linux Primer》

​        《构建嵌入式Linux系统》，对应的英文版是《Building Embedded Linux Systems》

​        **ARM体系架构：**

​        《ARM嵌入式系统开发：软件设计与优化》，对应的英文版是《ARM System Developer's Guide: Designing and Optimizing System Software》

​        **综合：**

​       《深入理解计算机系统》，对应的英文版是《Computer Systems: A Programmer's Perspective》

​        上面介绍的这些书，都是属于进阶级别的，所以要求要有一定的语言基础以及操作系统基础。此外，对于看书，老罗有一些观点，供大家参考：

​        **1. 书不是要用的时候才去看的，要养成经常看书、终身学习的习惯。**

​        **2. 不要只看与目前自己工作相关的书，IT技术日新月异，三五年河东，三五年河西。**

​        **3. 书看得多了，就会越看越快，学习新的东西时也越容易进入状态。**

​        **对于Android应用开发，力推官方文档：**

​        [http://developer.android.com/training/index.html](http://developer.android.com/training/index.html)

​        [http://developer.android.com/guide/components/index.html](http://developer.android.com/guide/components/index.html)

​        [http://developer.android.com/tools/index.html](http://developer.android.com/tools/index.html)

​        四. 研究过的内容

​        整个博客的内容看似松散，实际上都是有组织有计划的，目标是打通整个Android系统，从最上面的应用层，到最下面的Linux内核层。简单来说，博客的所有文章可以划分为“**三横三纵**”，如图2所示：

![img](http://img.blog.csdn.net/20130528234751506)

图2 Android系统研究之“三横三纵”

​        接下来，老罗就分别描述这三条横线和纵线，并且给出对应的博客文章链接。

​        1. 准备 -- Preparation -- 横线

​        主要就是：

​       （1）通过阅读相关的书籍来了解Linux内核和Android应用基础知识

​         [Android学习启动篇](http://blog.csdn.net/luoshengyang/article/details/6557518)

​       （2）搭建好Android源代码环境

​         [在Ubuntu上下载、编译和安装Android最新源代码](http://blog.csdn.net/luoshengyang/article/details/6559955)

​         [在Ubuntu上下载、编译和安装Android最新内核源代码（Linux Kernel）](http://blog.csdn.net/luoshengyang/article/details/6564592)

​         [如何单独编译Android源代码中的模块](http://blog.csdn.net/luoshengyang/article/details/6566662)

​         [制作可独立分发的Android模拟器](http://blog.csdn.net/luoshengyang/article/details/6586759)

​       （3）Android系统有很多C++代码，这些C++代码用到了很多[智能](http://lib.csdn.net/base/aiplanning)指针，因此有必要了解一下Android系统在C/C++ Runtime Framework中提供的智能指针

​         [Android系统的智能指针（轻量级指针、强指针和弱指针）的实现原理分析](http://blog.csdn.net/luoshengyang/article/details/6786239)

​         2. 专用驱动 -- Proprietary Drivers -- 横线

​         这些专用驱动就是指Logger、Binder和Ashmem，它们整个Android系统的基石：

​        （1）Logger

​         [ 浅谈Android系统开发中LOG的使用](http://blog.csdn.net/luoshengyang/article/details/6581828)

​         [Android日志系统驱动程序Logger源代码分析](http://blog.csdn.net/luoshengyang/article/details/6595744)

​         [Android应用程序框架层和系统运行库层日志系统源代码分析](http://blog.csdn.net/luoshengyang/article/details/6598703)

​         [Android日志系统Logcat源代码简要分析](http://blog.csdn.net/luoshengyang/article/details/6606957)

​        （2）Binder

​          [Android进程间通信（IPC）机制Binder简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6618363)

​         [浅谈Service Manager成为Android进程间通信（IPC）机制Binder守护进程之路](http://blog.csdn.net/luoshengyang/article/details/6621566)

​         [浅谈Android系统进程间通信（IPC）机制Binder中的Server和Client获得Service Manager接口之路](http://blog.csdn.net/luoshengyang/article/details/6627260)

​         [Android系统进程间通信（IPC）机制Binder中的Server启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6629298)

​         [Android系统进程间通信（IPC）机制Binder中的Client获得Server远程接口过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6633311)

​         [Android系统进程间通信Binder机制在应用程序框架层的Java接口源代码分析](http://blog.csdn.net/luoshengyang/article/details/6642463)

​        （3）Ashmem

​          [Android系统匿名共享内存Ashmem（Anonymous Shared Memory）简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6651971)

​          [Android系统匿名共享内存Ashmem（Anonymous Shared Memory）驱动程序源代码分析](http://blog.csdn.net/luoshengyang/article/details/6664554)

​          [Android系统匿名共享内存Ashmem（Anonymous Shared Memory）在进程间共享的原理分析](http://blog.csdn.net/luoshengyang/article/details/6666491)

​          [Android系统匿名共享内存（Anonymous Shared Memory）C++调用接口分析](http://blog.csdn.net/luoshengyang/article/details/6939890)

​        3. 硬件抽象层 -- HAL -- 纵线

​        硬件抽层象最适合用作Android系统的学习入口，它从下到上涉及到了Android系统的各个层次：

​         [Android硬件抽象层（HAL）概要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6567257)

​         [在Ubuntu上为Android系统编写Linux内核驱动程序](http://blog.csdn.net/luoshengyang/article/details/6568411)

​         [在Ubuntu上为Android系统内置C可执行程序测试Linux内核驱动程序](http://blog.csdn.net/luoshengyang/article/details/6571210)

​         [在Ubuntu上为Android增加硬件抽象层（HAL）模块访问Linux内核驱动程序](http://blog.csdn.net/luoshengyang/article/details/6573809)

​         [在Ubuntu为Android硬件抽象层（HAL）模块编写JNI方法提供Java访问硬件服务接口](http://blog.csdn.net/luoshengyang/article/details/6575988)

​         [在Ubuntu上为Android系统的Application Frameworks层增加硬件访问服务](http://blog.csdn.net/luoshengyang/article/details/6578352)

​         [在Ubuntu上为Android系统内置Java应用程序测试Application Frameworks层的硬件服务](http://blog.csdn.net/luoshengyang/article/details/6580267)

​        4. 应用程序组件 -- Application Component -- 纵线

​        应用程序组件是Android系统的核心，为开发者提供了贴心的服务。应用程序组件有四种，分别是Activity、Service、Broadcast Receiver和Content Provider。围绕应用程序组件，又有应用程序进程、消息循环和安装三个相关模块。

​       （1）Activity

​         [Android应用程序的Activity启动过程简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6685853)

​         [Android应用程序启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6689748)

​         [Android应用程序内部启动Activity过程（startActivity）的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6703247)

​         [Android应用程序在新的进程中启动新的Activity的方法和过程分析](http://blog.csdn.net/luoshengyang/article/details/6720261)

​         [解开Android应用程序组件Activity的"singleTask"之谜](http://blog.csdn.net/luoshengyang/article/details/6714543)

​       （2）Service

​         [Android系统在新进程中启动自定义服务过程（startService）的原理分析](http://blog.csdn.net/luoshengyang/article/details/6677029)

​         [Android应用程序绑定服务（bindService）的过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6745181)

​       （3）Broadcast Receiver

​         [Android系统中的广播（Broadcast）机制简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6730748)

​         [Android应用程序注册广播接收器（registerReceiver）的过程分析](http://blog.csdn.net/luoshengyang/article/details/6737352)

​         [Android应用程序发送广播（sendBroadcast）的过程分析](http://blog.csdn.net/luoshengyang/article/details/6744448)

​       （4）Content Provider

​         [Android应用程序组件Content Provider简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6946067)

​         [Android应用程序组件Content Provider应用实例](http://blog.csdn.net/luoshengyang/article/details/6950440)

​         [Android应用程序组件Content Provider的启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6963418)

​         [Android应用程序组件Content Provider在应用程序之间共享数据的原理分析](http://blog.csdn.net/luoshengyang/article/details/6967204)

​         [Android应用程序组件Content Provider的共享数据更新通知机制分析](http://blog.csdn.net/luoshengyang/article/details/6985171)

​       （5）进程

​         [Android系统进程Zygote启动过程的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6768304)

​         [Android应用程序进程启动过程的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6747696)

​       （6）消息循环

​         [Android应用程序消息处理机制（Looper、Handler）分析](http://blog.csdn.net/luoshengyang/article/details/6817933)

​         [Android应用程序键盘（Keyboard）消息处理机制分析](http://blog.csdn.net/luoshengyang/article/details/6882903)

​         [Android应用程序线程消息循环模型分析](http://blog.csdn.net/luoshengyang/article/details/6905587)

​       （7）安装

​         [Android应用程序安装过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6766010)

​         [Android系统默认Home应用程序（Launcher）的启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6767736)

​        5. 用户界面架构 -- UI -- 纵线

​        大家对老罗现在还在写Android 2.3的UI架构意见最大，认为已经过时了。老罗认为持有这种观点的人，都是没有经过认真思考的。老罗承认，从Android 4.0开始，UI部分发生了比较大的变化。但是请注意，这些变化都是在Android  2.3的UI架构基础之上进行的，也就是说，Android  2.3的UI架构并没有过时。你不能说Android 4.0在Android  2.3之上增加了一些feature，就说Android  2.3过时了。

​        下面这张是从Android官网拿过来的最新UI渲染流程图，也就是[4.2的UI渲染流程图](http://source.android.com/devices/graphics.html)：

![img](http://img.blog.csdn.net/20130529004204159)

图2 Android 4.2的UI渲染流程

​        从这张图可以看出关于Android的UI架构的三条主线：

​      **（1）每一个Window的Surface都怎样渲染的？不管怎么样，最后渲染出来的都是一个Buffer，交给SurfaceFlinger合成到Display上。**

​      **（2）SurfaceFlinger是怎样合成每一个Window的Surface的？**

​      **（3）WindowManamgerService是怎么样管理Window的？** 

​        第（1）和第（2）两个点在2.3和4.2之间有变化，主要是因为增加了GPU的支持，具体就表现为Window的Surface在渲染的时候使用了GPU，而SurfaceFlinger在合成每一个Window的Surface的时候，也使用了GPU或者Overlay和Blitter这些硬件加速，但是主体流程都没有变，也就是说，Window的Surface渲染好之后，最终依然是交给SurfaceFlinger来合成。此外，虽然我还没有开始看4.2的代码，但是可以看得出，4.2里面的HWComposer，只不过是封装和抽象了2.3就有的Overlay和Blitter，而SurfaceTexture的作用与2.3的SurfaceComposerClient、SurfaceControl也是类似的。第（3）点基本上就没有什么变化，除非以后要支持多窗口。

​        通过上述对比，只想强调一点：Android 2.3的UI架构并没有过时，是值得去研究的，并且在2.3的基础上去研究4.2的UI架构，会更有帮助。

​        仁者见仁，智者见智，Android 2.3的UI架构的说明就到此为止，接下来它的分析路线，都是围绕上述三个点来进行的。

​        首先是以开机动画为切入点，了解Linux内核里面的驱动：

​        [Android系统的开机画面显示过程分析](http://blog.csdn.net/luoshengyang/article/details/7691321)

​        FB驱动抽象了显卡，上面的用户空间程序就是通过它来显示UI的。

​        HAL层的Gralloc模块对FB驱动进行了封装，以方便SurfaceFlinger对它进行访问：

​        [Android帧缓冲区（Frame Buffer）硬件抽象层（HAL）模块Gralloc的实现原理分析](http://blog.csdn.net/luoshengyang/article/details/7747932)

​        SurfaceFlinger负责合成各个应用程序窗口的UI，也就是将各个窗口的UI合成，并且通过FB显示在屏幕上。在对SurfaceFlinger进行分析之前，我们首先了解应用程序是如何使用的它的：

​        [Android应用程序与SurfaceFlinger服务的关系概述和学习计划](http://blog.csdn.net/luoshengyang/article/details/7846923)

​        [Android应用程序与SurfaceFlinger服务的连接过程分析](http://blog.csdn.net/luoshengyang/article/details/7857163)

​        [Android应用程序与SurfaceFlinger服务之间的共享UI元数据（SharedClient）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/7867340)

​        [Android应用程序请求SurfaceFlinger服务创建Surface的过程分析](http://blog.csdn.net/luoshengyang/article/details/7884628)

​        [Android应用程序请求SurfaceFlinger服务渲染Surface的过程分析](http://blog.csdn.net/luoshengyang/article/details/7932268)

​        万事俱备，可以开始分析SurfaceFlinger了：

​        [Android系统Surface机制的SurfaceFlinger服务简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/8010977)

​        [Android系统Surface机制的SurfaceFlinger服务的启动过程分析](http://blog.csdn.net/luoshengyang/article/details/8022957)

​        [Android系统Surface机制的SurfaceFlinger服务对帧缓冲区（Frame Buffer）的管理分析](http://blog.csdn.net/luoshengyang/article/details/8046659)

​        [Android系统Surface机制的SurfaceFlinger服务的线程模型分析](http://blog.csdn.net/luoshengyang/article/details/8062945)

​        [Android系统Surface机制的SurfaceFlinger服务渲染应用程序UI的过程分析](http://blog.csdn.net/luoshengyang/article/details/8079456)

​        SurfaceFlinger操作的对象是应用程序窗口，因此，我们要掌握应用程序窗口的组成：

​        [Android应用程序窗口（Activity）实现框架简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/8170307)

​        [Android应用程序窗口（Activity）的运行上下文环境（Context）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8201936)

​        [Android应用程序窗口（Activity）的窗口对象（Window）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8223770)

​        [Android应用程序窗口（Activity）的视图对象（View）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8245546)

​        [Android应用程序窗口（Activity）与WindowManagerService服务的连接过程分析](http://blog.csdn.net/luoshengyang/article/details/8275938)

​        [Android应用程序窗口（Activity）的绘图表面（Surface）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8303098)

​        [Android应用程序窗口（Activity）的测量（Measure）、布局（Layout）和绘制（Draw）过程分析](http://blog.csdn.net/luoshengyang/article/details/8372924)

​        应用程序窗口是由WindowManagerService进行管理的，并且也是WindowManagerService负责提供窗口信息给SurfaceFlinger的，因此，我们最后分析WindowManagerService：

​        [Android窗口管理服务WindowManagerService的简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/8462738)

​        [Android窗口管理服务WindowManagerService计算Activity窗口大小的过程分析](http://blog.csdn.net/luoshengyang/article/details/8479101)

​        [Android窗口管理服务WindowManagerService对窗口的组织方式分析](http://blog.csdn.net/luoshengyang/article/details/8498908)

​        [Android窗口管理服务WindowManagerService对输入法窗口（Input Method Window）的管理分析](http://blog.csdn.net/luoshengyang/article/details/8526644)

​        [Android窗口管理服务WindowManagerService对壁纸窗口（Wallpaper Window）的管理分析](http://blog.csdn.net/luoshengyang/article/details/8550820)

​        [Android窗口管理服务WindowManagerService计算窗口Z轴位置的过程分析](http://blog.csdn.net/luoshengyang/article/details/8570428)

​        [Android窗口管理服务WindowManagerService显示Activity组件的启动窗口（Starting Window）的过程分析](http://blog.csdn.net/luoshengyang/article/details/8577789)

​        [Android窗口管理服务WindowManagerService切换Activity窗口（App Transition）的过程分析](http://blog.csdn.net/luoshengyang/article/details/8596449)

​        [Android窗口管理服务WindowManagerService显示窗口动画的原理分析](http://blog.csdn.net/luoshengyang/article/details/8611754)

​        上述内容都研究清楚之后，Android系统的UI架构的骨架就清晰了。但是前面所研究的应用程序窗口还是太抽象了，我们有必要研究一下那些组成应用程序窗口内容的UI控件是怎么实现的，以TextView和SurfaceView为代表：

​        [Android控件TextView的实现原理分析](http://blog.csdn.net/luoshengyang/article/details/8636153)

​        [Android视图SurfaceView的实现原理分析](http://blog.csdn.net/luoshengyang/article/details/8661317)

​        最后，分析Android系统的UI架构，怎能不提它的资源管理框架？它有效地分离了代码和UI：

​        [Android资源管理框架（Asset Manager）简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/8738877)

​        [Android应用程序资源的编译和打包过程分析](http://blog.csdn.net/luoshengyang/article/details/8744683)

​        [Android应用程序资源管理器（Asset Manager）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8791064)

​        [Android应用程序资源的查找过程分析](http://blog.csdn.net/luoshengyang/article/details/8806798)

​        分析这里，Android系统的UI架构就分析完成了，看出什么门道来没有？是的，我们以开机动画为切入点，从Linux内核空间的FB驱动，一直分析到用户空间中HAL层模块Gralloc、C/C++ Runtime Framework层的SurfaceFlinger、Java Runtime Framework层的WindowMangerService、Window、Widget，以及资源管理框架，从下到上，披荆斩棘。

​        6. Dalvik虚拟机 -- 横线

​        Android系统的应用程序及部分应用程序框架是使用Java语言开发的，它们运行在Dalvik虚拟机之上，还有另外一部分应用唾弃框架在使用C/C++语言开发的。使用Java语言开发的应用程序框架老罗称之为Java Runtime Framework，而使用C/C++语言开发的应用程序框架老罗称之为C/C++ Runtime Framework，它们被Dalvik虚拟机一分为二。通过前面的学习，其实我们都已经了解Android系统的Java Runtime Framework和C/C++ Runtime Framework，因此，我们最后将注意力集中在Dalvik虚拟机上：

​        [Dalvik虚拟机简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/8852432)

​        [Dalvik虚拟机的启动过程分析](http://blog.csdn.net/luoshengyang/article/details/8885792)

​        [Dalvik虚拟机的运行过程分析](http://blog.csdn.net/luoshengyang/article/details/8914953)

​        [Dalvik虚拟机JNI方法的注册过程分析](http://blog.csdn.net/luoshengyang/article/details/8923483)

​        [Dalvik虚拟机进程和线程的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8923484)

​        学习完成“三横三纵”这六条主线之后，我们就可以自豪地说，从上到下地把Android系统打通，并且将它的骨架和脉络也理清了！

​        **对于“准备”、“专用驱动”、“HAL”、“应用程序组件”这四条主线，老罗极力推荐大家看《Android系统源代码情况分析》一书，内容比博客文章要系统、详细很多，不说其它的，就单单是讲Binder进程间通信机制的那一章，就物超所值。在《Android系统源代码情景分析》一书中，老罗最引以为豪的就是讲Binder进程间通信机制的第5章，网上或者其它书上绝对是找不到这么详尽的分析资料。**

​        五. 将来要做的事情

​        接下来要做的主要是三件事情：

​        **1. 继续研究Android系统**

​        本来以为前段时间的Google I/O会发布Android 4.3或者5.0，然后老罗就以最新发布的版本为蓝本来进行研究。既然没有发布新版本，那么就只有以现在的最新发布版本4.2为蓝本进行研究了。如前所述，4.2与2.3相比，最大的不同之处是默认增加了GPU支持，因此，老罗在接下来的一段时间里，将着重研究4.2的GPU支持。

​        **2. 停止博客更新**

​        这两年投入在博客上的精力太多了，博客上的文章基本上熬夜熬出来的。大多数时候，一个话题要花一个星期左右的时间来看代码，然后再花四个星期左右的时间将文章写出来。本来是这样计划的，依靠《Android系统源代码情景分析》一书的销量，可以在经济上得到一定的回报，然后可以继续在博客上投入，直至把4.x版本的GPU支持写完，最后再整理出一本关于Android系统UI架构的书。但是最近询问了一下书的销量，差强人意，达不到预期目标。由于没有形成良性循环，因此没有办法，只好停止博客更新。老罗需要把精力投入在其它事情上，希望大家谅解！

​        **3. 仍然会持续地进行一些小分享**

​        主要是一些随笔分享，这些分享主要会发布在微博或者QQ群上面，那里也方便一些和大家进行讨论。此外，老罗也乐意和大家进行一些线下分享，主要针对企业或者单位组织的沙龙、活动和会议等，同时也可以单独地针对企业内部进行分享。不过前提当然是举办方对《老罗的Android之旅》专栏或者《Android系统源代码情景分析》一书的内容感兴趣，并且邀请老罗去参加。

​        如果需要邀请老罗去参加分享，可以通过微博或者邮箱和老罗联系，非常感谢大家两年以来的支持！

​        **邮箱：shyluo@gmail.com**

​        **微博：http://weibo.com/shengyangluo**