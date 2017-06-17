### App启动流程第2篇

书接上文，App启动一共有七个阶段，上篇文章篇幅所限，我们只看了第一阶段，接下来讲剩余的六个阶段，仍然是拿斗鱼App举例子。

简单回顾一下第一阶段的流程，就是Launcher向AMS发送一个跨进程通信，通过AMN/AMP，告诉AMS，我要启动斗鱼App。

画一个图，描述一下启动App所经历的7个阶段：

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170519225853353-1638311589.png)

第2阶段 AMS处理Launcher传过来的信息

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170519225909588-1361872026.png)

这个阶段主要是Binder的Server端在做事情。因为我们是没有机会修改Binder的Server端逻辑的，所以这个阶段看起来非常枯燥，我尽量说的简单些。

1. 首先Binder，也就是AMN/AMP，和AMS通信，肯定每次是做不同的事情，就比如说这次Launcher要启动斗鱼App，那么会发送类型为START_ACTIVITY——TRANSACTION的请求给AMS，同时会告诉AMS要启动哪个Activity。
2. AMS说，好，我知道了，然后它会干一件很有趣的事情，就是检查斗鱼App中的Manifest文件，是否存在要启动的Activity。如果不存在，就抛出Activity not found的错误，各位做App的同学对这个异常应该再熟悉不过了，经常写了个Activity而忘记在Manifest中声明了，就报这个错，就是因为AMS在这里做检查。不管是新启动一个App的首页，还是在App内部跳转到另一个Activity，都会做这个检查。
3. 但是Launcher还活着啊，所以接下来AMS会通知Launcher，哥们儿没你什么事了，你“停薪留职”吧。那么AMS是通过什么途径告诉Launcher的呢？

前面讲过，Binder的双方进行通信是平等的，谁发消息，谁就是Client，接收的一方就是Server。Client这边会调用Server的代理对象。

对于从Launcher发来的消息，通过AMS的代理对象AMP，发送给AMS。

那么当AMS想给Launcher发消息，又该怎么办呢？前面不是把Launcher以及它所在的进程给传过来了吗？它在AMS这边保存为一个ActivityRecord对象，这个对象里面有一个ApplicationThreadProxy，单单从名字看就出卖了它，这就是一个Binder代理对象。它的Binder真身，也就是ApplicationThread。

站在AIDL的角度，来画这张图，是这样的：

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170519225921916-810851677.png)

所以结论是，AMS通过ApplicationThreadProxy发送消息，而App端则是通过ApplicationThread来接收这个消息。

第3阶段 Launcher去休眠，然后通知AMS，我真的已经“停薪留职”了，没有吃空饷

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170519225928432-1563080021.png)

ApplicationThread（简称APT），它和ApplicationThreadProxy（简称ATP）的关系，我们在第三阶段已经介绍过了。

APT接收到来自AMS的消息后，就调用ActivityThread的sendMessage方法，向Launcher的主线程消息队列发送一个PAUSE_ACTIVITY消息。

前面说过，ActivityThread就是主线程（UI线程）

看到下面的代码是不是很亲切？

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170519225946869-1315230205.png)

发送消息是通过一个名为H的Handler类的完成的，这个H类的名字真特么有个性，想记不住都难。

做App的同学都知道，继承自Handler类的子类，就要实现handleMessage方法，这里是一个switch…case语句，处理各种各样的消息，PAUSE_ACTIVITY消息只是其中一种，由此也能预见到，AMS给Activity发送的所有消息，以及给其它三大组件发送的所有消息，都从H这里经过，为什么要强调这一点呢，既然四大组件都走这条路，那么这里就可以做点手脚，从而做插件化技术，这个我们以后介绍插件化技术的时候会讲到。

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170519225957400-731726279.png)

H对于PAUSE_ACTIVITY消息的处理，如上面的代码，是调用ActivityThread的handlePauseActivity方法。这个方法干两件事：

-    ActivityThread里面有一个mActivities集合，保存当前App也就是Launcher中所有打开的Activity，把它找出来，让它休眠。
-    通过AMP通知AMS，我真的休眠了。

你可能会找不到H和APT这两个类文件，那是因为它们都是ActivityThread的内嵌类。

至此，Launcher的工作完成了。你可以看到在这个过程中，各个类都起到了什么作用。芸芸众生，粉墨登场：

-    APT
-    ActivityThread
-    H

第4阶段 AMS启动新的进程

接下来又轮到AMS做事了，你们会发现我不太喜欢讲解AMS的流程，甚至都不画UML图，因为这部分逻辑和App开发人员关系不是很大，我尽量说的简单一些，把流程说清楚就好。

AMS接下来要启动斗鱼App的首页，因为斗鱼App不在后台进程中，所以要启动一个新的进程。这里调用的是Process.start方法，并且指定了ActivityThread的main函数为入口函数。

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170519230007713-577694475.png)

第5阶段 新的进程启动，以ActivityThread的main函数作为入口

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170519230017916-541390106.png)

启动新进程，其实就是启动一个新的App。

在启动新进程的时候，为这个进程创建ActivityThread对象，这就是我们耳熟能详的主线程（UI线程）。

创建好UI线程后，立刻进入ActivityThread的main函数，接下来要做2件具有重大意义的事情：

1）创建一个主线程Looper，也就是MainLooper。看见没，MainLooper就是在这里创建的。

2）创建Application。记住，Application是在这里生成的。

App开发人员对Application非常熟悉，因为我们可以在其中写代码，进行一些全局的控制，所以我们通常认为Application是掌控全局的，其实Application的地位在App中并没有那么重要，它就是一个Context上下文，仅此而已。

App中的灵魂是ActivityThread，也就是主线程，只是这个类对于App开发人员是访问不到的——使用反射倒是可以修改这个类的一些行为。

创建新App的最后，就是告诉AMS，我启动好了，同时把自己的ActivityThread对象发送给AMS，从此以后，AMS的电话簿中就多了这个新的App的登记信息，AMS以后向这个App发送消息，就通过这个ActivityThread对象。

第6阶段 AMS告诉新App启动哪个Activity

AMS把传入的ActivityThread对象，转为一个ApplicationThread对象，用于以后和这个App跨进程通信。还记得APT和ATP的关系吗？这就又回到第2阶段的那张关系图了。

还记得第1阶段，Launcher发送给AMS要启动斗鱼App的哪个Activity吗？这个信息被AMS存下来了。

那么在第6阶段，AMS从过去的记录中翻出来要启动哪个Activity，然后通过ATP告诉App。

第7阶段 启动斗鱼首页Activity

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170519230029494-413774325.png)

毕其功于一役，尽在第7阶段。这是最后一步。

App，这个Binder的另一端，通过APT接收到AMS的消息，仍然是在H的handleMessage方法的switch语句中处理，只不过，这次消息的类型是LAUNCH_ACTIVITY：

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170519230047322-2090810570.png)

ActivityClientRecord是什么？这是AMS传递过来的要启动的Activity。

还是这里，我们仔细看那个getPackageInfoNoCheck方法，这个方法会提取Apk中的所有资源，然后设置给r的packageInfo属性。这个属性的类型很有名，叫做LoadedApk。各位记住这里，这个地方可以干坏事，也是插件化技术渗入的一个点。

在H的这个分支中，又反过来回调ActivityThread的handleLaunchActivity方法，你要是觉得很绕那就对了。其实我一直觉得，ActivityThread和H合并成一个类也没问题。

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170519230101088-869045714.png)

重新看一下这个过程，每次都是APT执行ActivityThread的sendMessage方法，在这个方法中，把消息拼装一下，然后扔个H的swicth语句去分析，来决定要执行ActivityThread的那个方法。每次都是这样，习惯就好了。

handleLaunchActivity方法都做哪些事呢？

1）通过Instrumentation的newActivity方法，创建出来要启动的Activity实例。

2）为这个Activity创建一个上下文Context对象，并与Activity进行关联。

3）通过Instrumentation的callActivityOnCreate方法，执行Activity的onCreate方法，从而启动Activity。看到这里是不是很熟悉很亲切？

至此，App启动完毕。这个流程是经过了很多次握手， App和ASM，频繁的向对方发送消息，而发送消息的机制，是建立在Binder的基础之上的。

下一篇文章，我们讲Context家族。