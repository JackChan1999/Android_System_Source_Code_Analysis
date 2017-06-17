### Service

Service有两套流程，一套是启动流程，另一套是绑定流程。我们做App开发的同学都应该知道。

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170520231111588-262034472.png)

#### 在新进程启动Service

  我们先看Service启动过程，假设要启动的Service是在一个新的进程中，分为5个阶段：

  1）App向AMS发送一个启动Service的消息。

  2）AMS检查启动Service的进程是否存在，如果不存在，先把Service信息存下来，然后创建一个新的进程。

  3）新进程启动后，通知AMS说我可以啦。

  4）AMS把刚才保存的Service信息发送给新进程

  5）新进程启动Service

我们仔细看一下这5个阶段：

第1阶段

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170520231119900-1492923568.png)

和Activity非常像，仍然是通过AMM/AMP把要启动的Service信息发送给AMS。

第2阶段

AMS检查Service是否在Manifest中声明了，没声明会直接报错。

AMS检查启动Service的进程是否存在，如果不存在，先把Service信息存下来，然后创建一个新的进程。

在AMS中，每个Service，都使用ServiceRecord对象来保存。

第3阶段

Service所在的新进程启动的过程，就和前面介绍App启动时的过程差不多。

新进程启动后，也会创建新的ActivityThread，然后把ActivityThread对象通过AMP传递给AMS，告诉AMS，新进程启动成功了。

第4阶段

AMS把传进来的ActivityThread对象改造为ApplicationThreadProxy，也就是ATP，通过ATP，把要启动的Service信息发送给新进程。

第5阶段

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170520231131010-228401732.png)

新进程通过ApplicationThread接收到AMS的信息，和前面介绍的启动Activity的最后一步相同，借助于ActivityThread和H，执行Service的onCreate方法。在此期间，为Service创建了Context上下文对象，并与Service相关联。

需要重点关注的是ActivityThread的handleCreateService方法，

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170520231142197-538428403.png)

你会发现，这段代码和前面介绍的handleLaunchActivity差不多，都是从PMS中取出包的信息packageInfo，这是一个LoadedApk对象，然后获取它的classloader，反射出来一个类的对象，在这里反射的是Service。

四大组件的逻辑都是如此，所以我们要做插件化，可以在这里做文章，换成插件的classloader，加载插件中的四大组件。

至此，我们在一个新的进程中启动了一个Service。

#### 启动统一进程的Service

如果是在当前进程启动这个Service，那么上面的步骤就简化为：

  1）App向AMS发送一个启动Service的消息。

  2）AMS例行检查，比如Service是否声明了，把Service在AMS这边注册。AMS发现要启动的Service就是App所在的Service，就通知App启动这个Service。

  3）App启动Service。

我们看到，没有了启动新进程的过程。

#### 在同一进程绑定Service

如果是在当前进程绑定这个Service呢？过程是这样的：

  1）App向AMS发送一个绑定Service的消息。

  2）AMS例行检查，比如Service是否声明了，把Service在AMS这边注册。AMS发现要启动的Service就是App所在的Service，就先通知App启动这个Service，然后再通知App，对Service进行绑定操作。

  3）App收到AMS第1个消息，启动Service，

  4）App收到AMS第2个消息，绑定Service，并把一个Binder对象传给AMS

  5）AMS把接收到的Binder对象，发送给App

  6）App收到Binder对象，就可以使用了。

你也许会问，都在一个进程，App内部直接使用Binder对象不就好了，其实吧，要考虑不在一个进程的场景，代码又不能写两份，两套逻辑，所以就都放在一起了，即使在同一个进程，也要绕着AMS走一圈。

第1阶段：App向AMS发送一个绑定Service的消息。

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170520231153400-1053122975.png)

第4阶段：处理第2个消息

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170520231200791-687448931.png)

第5阶段和第6阶段：

这一步是要仔细说的，因为AMS把Binder对象传给App，这里没用ATP和APT，而是用到了AIDL来实现，这个AIDL的名字是IServiceConnection。

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170520231209541-1531293878.png)

ServiceDispatcher的connect方法，最终会调用ServiceConneciont的onServiceConnected方法，这个方法我们就很熟悉了。App开发人员在这个方法中拿到connection，就可以做自己的事情了。

好了，关于Service的底层知识，我们就全都介绍完了。当你再去编写一个Service时，是否感觉对这个组件理解的更透彻了呢？

下一篇我们聊一聊BroadcastReceiver。