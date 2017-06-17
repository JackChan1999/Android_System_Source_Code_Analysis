### BroadcastReceiver

BroadcastReceiver，也就是广播，简称Receiver。

很多App开发人员表示，从来没用过Receiver。其实吧，对于音乐播放类App，用Service和Receiver还是蛮多的，如果你用过QQ音乐，App退到后台，音乐照样播放不会停止，这就是你写的Service在后台起作用。

在前台的Activity，点击停止按钮，就会给后台Service发送一个Receiver，通知它停止播放音乐；点击播放按钮，仍然是发送这个Receiver，只是携带的值变了，所以Service收到请求后播放音乐。

反过来，后台Service播放每播放完一首音乐，接下来准备播放下一首音乐的时候，就会给前台Activity发Receiver，让Activity显示下一首音乐的名称。

所以音乐播放器的原理，就是一个前后台Activity和Service互相发送和接收Receiver的过程。

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170528113922032-677268609.png)

### Receiver分静态广播和动态广播两种。

在Manifest中声明的Receiver，是静态广播：

 ![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170520230609166-2016809804.png)

在程序中手动写注册代码的，是动态广播：

 ![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170520230622088-1079718301.png)

二者具有相同的功能，只是写法不同。既然如此，我们就可以把所有静态广播，都改为动态广播，这就省的在Manifest文件中声明了，避免了AMS检查。接下来你想到什么？对，Receiver的插件化解决方案，就是这个思路。

接下来我们看Receiver是怎么和AMS打交道的，分为两部分，一是注册，二是发送广播。

你只有注册了这个广播，发送这个广播时，才能通知到你，执行onReceive方法。

我们就拿音乐播放器来举例，在Activity注册Receiver，在Service发送广播。Service播放下一首音乐时，会通知Activity修改当前正在播放的音乐名称。

注册过程如下所示：

 1）在Activity中，注册Receiver，并通知AMS。

 ![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170520230632572-1837816098.png)

这里Activity使用了Context提供的registerReceiver方法，然后通过AMN/AMP，把一个receiver传给AMS。

在创建这个Receiver对象的时候，需要为receiver指定IntentFilter，这个filter就是Receiver的身份证，用来描述receiver。

在Context的registerReceiver方法中，它会使用PMS获取到包的信息，也就是LoadedApk对象。

就是这个LoadedApk对象，它的getReceiverDispatcher方法，将Receiver封装成一个实现了IIntentReceiver接口的Binder对象。

我们就是将这个Binder对象和filter传递给AMS。

只传递Receiver给AMS是不够的，当发送广播时，AMS不知道该发给谁啊？所以Activity所在的进程还要把自身对象也发送给AMS。

2）AMS收到消息后，就会把上面这些信息，存在一个列表中，这个列表中保存了所有的Receiver。

注意了，这里忙活半天，都是在注册动态receiver。

静态receiver什么时候注册到AMS的呢？是在App安装的时候。PMS会解析Manifest中的四大组件信息，把其中的receiver存起来。

动态receiver和静态receiver分别存在AMS不同的变量中，在发送广播的时候，会把两种receiver合并到一起，然后以此发送。其中动态的排在静态的前面，所以动态receiver永远优先于静态receiver收到消息。

此外，Android系统每次启动的时候，也会把静态广播接收者注册到AMS。因为Android系统每次启动时，都会重新安装所有的apk，详细流程，我们会在后面PMS的相关章节看到。

发送广播的流程如下：

1）在Service中，通过AMM/AMP，发送广播给AMS，广播中携带着Filter。

2）AMS收到这个广播后，在receiver列表中，根据filter找到对应的receiver，可能是多个，把它们都放到一个广播队列中。最后向AMS的消息队列发送一个消息。

当消息队列中的这个消息被处理时，AMS就从广播队列中找到合适的receiver，向广播接收者所在的进程发送广播。

3）receiver所在的进程收到广播，并没有把广播直接发给receiver，而是将广播封装成一个消息，发送到主线程的消息队列中，当这个消息被处理时，才会把这个消息中的广播发送给receiver。

我们下面通过图，仔细看一下这3个阶段：

第1步，Service发送广播给AMS

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170520230643682-530204934.png)

发送广播，是通过Intent这个参数，携带了Filter，从而告诉AMS，什么样的receiver能接受这个广播。

第2步，AMS接收广播，发送广播。

接收广播和发送广播是不同步的。AMS每接收到一个广播，就把它扔到广播发送队列中，至于发送是否成功，它就不管了。

因为receiver分为无序receiver和有序receiver，所以广播发送队列也分为两个，分别发送这两种广播。

AMS发送广播给客户端，这又是一个跨进程通信，还是通过ATP，把消息发给APT。因为要传递Receiver这个对象，所以它也是一个Binder对象，才可以传过去。我们前面说过，在把Receiver注册到AMS的时候，会把Receiver封装为一个IIntentReceiver接口的Binder对象。那么接下来，AMS就是把这个IIntentReceiver接口对象传回来。

第3步，App处理广播

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170520230652791-1362873866.png)

这个流程描述如下：

 1）消息从AMS传到客户端，把AMS中的IIntentReceiver接口对象转为InnerReceiver对象，这就是receiver，这是一个AIDL跨进程通信。

 2）然后在ReceiverDispatcher中封装一个Args对象（这是一个Runnable对象，要实现run方法），包括广播接收者所需要的所有信息，交给ActivtyThread来发送

 3）接下来要走的路就是我们所熟悉的了，ActivtyThread把Args消息扔到H这个Hanlder中，向主线程消息队列发送消息。等到执行Args消息的时候，自然是执行Args的run方法。

 4）在Args的run方法中，实例化一个Receiver对象，调用它的onReceiver方法。

 5）最后，在Args的run方法中，随着Receiver的onReceiver方法调用结束，会通过AMN/AMP发送一个消息个AMS，告诉AMS，广播发送成功了。AMS得到通知后，就发送广播给下一个Receiver。

 注意：InnerReceiver是IIntentReceiver的stub，是Binder对象的接收端。

### 广播的种类

 Android广播按发送方式分类有三种：无序广播、有序广播（OrderedBroadcast）和粘性广播（StickyBroadcast）。

 1）无序广播是最普通的广播。

 2）有序广播区别于无序广播，就在于它可以指定优先级。

这两种receiver存在AMS不同的变量中，可以认为是两个receiver集合，发送不同类别的广播。

 3）粘性广播是无序广播的一种。

粘性广播，我们平常见的不多，但我说一个场景你就明白了，那就是电池电量。当电量小于20%的时候，就会提示用户。

而获取电池的电量信息，就是通过广播来实现的。

但是一般的广播，发完就完了。我们需要有这样一种广播，发出后，还能一直存在，未来的注册者也能收到这个广播，这种广播就是粘性广播。

由于动态receiver只能在Activity的onCreate()方法调用时才能注册再接收广播，所以当程序没有运行就不能接受到广播；但是静态注册的则不依赖于程序是否处于运行状态。

至此，关于广播的所有概念，就全都介绍完了，就连我都比较惊讶的是，我居然没贴什么代码。希望上述这两千多字，能引导App开发人员进入一个神奇的世界。

下一篇文章，我们去看看ContentProvider。