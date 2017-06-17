### AMS

如果站在四大组件的角度来看，AMS就是Binder中的Server。

AMS全称是ActivityManagerService，看字面意思是管理Activity的，但其实四大组件都归它管。估计是Android底层开发人员先写了ActivityManagerService用来管理Activity，后来写Service、Receiver、CP的时候发现代码都差不多，于是就全都用ActivityManagerService，但是却忘记改名字了——我也是猜的，纯属八卦。

由此而说到了插件化，我记得16年和Lody、张勇、林光亮一起吃夜宵的时候，我当时问了困惑已久的两个问题：

1）App的安装过程，为什么不把apk解压缩到本地，这样读取图片就不用每次从apk包中读取了——这个问题，我们放到PMS那一节再详细说。

2）为什么Hook永远是在Binder Client端，也就是四大组件这边，而不是在AMS那一侧进行Hook。

这里要说清楚第二个问题。就拿Android剪切板举例吧。前面说过，这也是个Binder服务。

AMS要负责和所有App的四大组件进行通信，也真够他忙的。如果在一个App中，在AMS层面把剪切板功能给篡改了，那会导致Android系统所有的剪切板功能被篡改——这就是病毒了，如果是这样的话，Android系统早就死翘翘了。所以Android系统不允许我们这么做。

我们只能在AMS的另一侧，Client端，也就是四大组件这边做篡改，这样即使我们把剪切板功能篡改了，也只影响篡改代码所在的App，在别的App中，剪切板功能还是正常的。

关于AMS我们就说这么多，下面介绍四大组件时，会反复提到四大组件和AMS的跨进程通信。

### Activity 第1讲

对于做App的开发人员而言，Activity是四大组件中用的最多的，也是最复杂的，我这里只讲Activity的启动和通信原理。还有一些相关的概念，比如说View、Looper、Intent、Resource，我以后另起章节来介绍。

注：我对四大组件的分析，都是基于罗升阳的那本分析Android底层的书，我把其中罗列的大部分代码都删掉了，只保留那些对App开发人员有用的一些代码片段和一些关键类名，并融入了我对四大组件的理解。

1）首先要搞清，App是怎么启动的。

在手机屏幕上点击某个App的Icon，假设就是斗鱼App吧，这个App的首页（或引导页）就出现在我们面前了。这个看似简单的操作，背后经历了Activity和AMS的反反复复的通信过程。

首先要搞清楚，在手机屏幕上点击App的icon快捷图标，此时手机屏幕就是一个Activity，而这个Activity所在的App，业界称之为Launcher。Launcher是手机系统厂商提供的，类似小米华为这样的手机，比拼的就是谁的Launcher绚丽和人性化。

Launcher这个App，其实和我们做的各类应用类App没有什么不同，我们大家用过华为、小米之类的手机，预装App以及我们下载的各种App，都显示在Launcher上，每个App表现为一个Icon。Icon多了可以分页，可以分组，此外，Launcher也会发起网络请求，调用天气的数据，显示在屏幕上，所谓的人性化界面。

还记得我们在开发一款App时，在Manifest文件中是怎么定义默认启动Activity的么？如下所示：

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170519224918166-1866694860.png)

而Launcher中为每个App的icon提供了启动这个App所需要的Intent信息，如下所示（比如说斗鱼的包名是）：

```
action：android.intent.action.MAIN
category: android.intent.category.LAUNCHER
cmp: 斗鱼的包名+ 首页Activity名
```

这些信息是App安装（或Android系统启动）的时候，PackageManagerService从斗鱼的apk包的manifest文件中读取到的。

所以点击icon就启动了斗鱼App中的首页。

2）启动App哪有那么简单

前面介绍的只是App启动的一个最简单的概述。

仔细看，我们会发现，Launcher和斗鱼是两个不同的App，他们位于不同的进程中，它们之间的通信是通过Binder完成的——这时候AMS出场了。

仍然以启动斗鱼App为例子，整体流程是：

1. Launcher通知AMS，要启动斗鱼App，而且指定要启动斗鱼的哪个页面（也就是首页）。
2. AMS通知Launcher，好了我知道了，没你什么事了，同时，把要启动的首页记下来。
3. Launcher当前页面进入Paused状态，然后通知AMS，我睡了，你可以去找斗鱼App了。
4. AMS检查斗鱼App是否已经启动了。是，则唤起斗鱼App即可。否，就要启动一个新的进程。AMS在新进程中创建一个ActivityThread对象，启动其中的main函数。
5. 斗鱼App启动后，通知AMS，说我启动好了。
6. AMS翻出之前在第二步存的值，告诉斗鱼App，启动哪个页面。
7. 斗鱼App启动首页，创建Context并与首页Activity关联。然后调用首页Activity的onCreate函数。

至此启动流程完成，分成两部分，1-3步，Launcher和AMS相互通信，而后面几步，斗鱼App和AMS相互通信。

这会牵扯一堆类进来，列举如下，在接下来的分析中，我们都会遇到：

- Instrumentation
- ActivityThread
- H
- LoadedApk
- AMS
- ActivityManagerNative和ActivityManagerProxy
- ApplicationThread和ApplicationThreadProxy

第1阶段 Launcher通知AMS

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170519224933760-1702392298.png)

这是我根据老罗那本书中对Activity的分析，自己手绘的UML图，一共七张，也就是Activity启动所经历的七个阶段。建议各位读者也亲自手绘一遍，从而就加深理解。

-    第1步、第2步

从上图中我们看到，点击Launcher上的斗鱼App的icon快捷图标，这时会调用Launcher的startActivitySafely方法，其实还是会调用Activity的startActivity方法，intent中带着要启动斗鱼App所需要的关键信息，如下所示：

```
action = “android.intent.action.MAIN”
category = “android.intent.category.LAUNCHER”
cmp = “com.douyu.activity.MainActivity”
```

第3行代码是我猜的，就是斗鱼App在Mainfest文件中指定为首页的那个Activity。这样，我们终于明白，为什么在Mainfest中，给首页指定action和category了。在app的安装过程中，会把这个信息“记录”在Launcher的斗鱼启动快捷图标中。关于App的安装过程，我会在后面的文章详细介绍。

startActivity这个方法，如果我们看它的实现，会发现它调来调去，经过一系列startActivity的重载方法，最后会走到startActivityForResult方法。

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170519224954541-1243107283.png)

我们知道startActivityForResult需要两个参数，一个是intent，另一个是code，这里code是-1，表示Launcher才不关心斗鱼的App是否启动成功了呢。

第3步： startActivityForResult

Activity内部会保持一个对Instrumentation的引用，但凡是做过App单元测试的同学，对这个类都很熟悉，称之为仪表盘。

在startActivityForResult方法的实现中，会调用Instrumentation的execStartActivity方法。

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170519225012385-1869709294.png)

看到这里，我们发现有个mMainThread变量，这是一个ActivityThread类型的变量。

这个家伙的来头可不小。

ActivityThread，就是主线程，也就是UI线程，它是在App启动时创建的，它代表了App应用程序。

啥？ActivityThread代表了App应用程序，那Application类岂不是被架空了？其实，Application对我们App开发人员来说也许很重要，但是在Android系统中还真的没那么重要，他就是个上下文。Activity不是有个Context上下文吗？Application就是整个ActivityThread的上下文。

ActivityThread则没有那么简单了。它里面有main函数。

我们知道大部分程序都有main函数，比如java、C#，远了不说，iPhone App用到的Objective-C，也有main函数，那么Android的main函数藏在哪里？就在ActivityThread中，如下所示，代码太多，我只截取了一部分

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170519225039353-1677547683.png)

又有人会问？不是说谁写的程序，谁就要提供main函数，作为入口吗？但Android App却不是这样的。Android App的main函数，在ActivityThread里面，而这个类是Android系统提供的底层类，不是我们提供的。

所以这就是Andoid有趣的地方。Android App的入口是Mainifest中定义默认启动Activity。这是由Android AMS与四大组件的通信机制决定的。

最近在看吴越版的西游记，就发现这个西天取经为啥用了十几年啊？因为这师徒四个取经路上爱管闲事，所以耽搁了很久。我这篇文章也是如此，经常讲着讲着就跑题了，再这么写下去，不知道要写到啥时候，所以我们一路向西，径直往前走，再遇到奇怪的类，先不要理它。

在回到代码来，这里要传递2个很重要的参数：

- 通过ActivityThread的getApplicationThread方法取到一个Binder对象，它的类型为ApplicationThread，它代表着Launcher所在的App进程。
- mToken，这也是个Binder对象，它代表了Launcher这个Activity，这里也通过Instrumentation传给AMS，AMS一查电话簿，就知道是谁向AMS发起请求了。

这两个参数是伏笔，传递给AMS，以后AMS想反过来通知Launcher，就能通过这两个参数，找到Launcher。

第4步，Instrumentation的execStartActivity方法

Instrumentation绝对是Adnroid测试团队的最爱，因为它可以帮我们启动Activity。

回到我们的App启动过程来，在Instrumentation的execStartActivity方法中，

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170519225106307-741280835.png)

我理解这就是一个透传，Activity把数据借助Instrumentation，传递给ActivityManagerNative，没太多有趣的内容，就不多讲了。

第5步：AMN的getDefault方法

ActivityManagerNative，简称AMN。这个类后面会反复用到。

AMN通过getDefault方法，从ServiceManager中取得一个名为activity的对象，然后把它包装成一个ActivityManagerProxy对象（简称AMP），AMP就是AMS的代理对象。

备注1：ServiceManager是一个容器类。

备注2:  AMN的getDefault方法返回类型为IActivityManager，而不是AMP。IActivityManager是一个实现了IInterface的接口，里面定义了四大组件所有的生命周期。

AMN和AMP都实现了IActivityManager接口，AMS继承自AMN（好乱），那么对照着前面AIDL的UML，就不难理解了：

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170519225130822-867987839.png)

第6步，AMP的startActivity方法

看到这里，你会发现AMP的startActivity方法，和AIDL的Proxy方法，是一模一样的，写入数据到另一个进程，也就是AMS，然后等待AMS返回结果。

至此，第一阶段的工作就做完了。

后续流程请参加下一篇文章。