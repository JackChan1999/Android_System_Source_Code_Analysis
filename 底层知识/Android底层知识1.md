这个系列的文章一共8篇，我酝酿了很多年，参考了很多资源，查看了很多源码，直到今天把它写出来，也是战战兢兢，生怕什么地方写错了，贻笑大方。

### 引言

早在我还是Android菜鸟的时候，有很多技术我都不太明白，也都找不到答案，比如apk是怎么安装的，比如资源是怎么加载的。

再比如说，每本书都会讲AIDL，但我却从来没用过。四大组件也是这个问题，我只用过Activity，其它三个组件，不但没用过，甚至连它们是做什么的，都不是很清楚。

之所以这样，是因为我一直从事的是电商类App开发工作，对于这类App，基本就是由列表页和详情页组成的，所以我们每天面对的是Activity，会写这两类页面，把网络底层封装的足够强大就够了。

绝大多数App开发人员，都是如此。

但直到接触Android的插件化编程和热修复技术，才发现只掌握上述这些技术是远远不够的。

### 还是引言

市场上有很多介绍Android底层的书籍，网上也有很多文章，但大都是给ROM开发人员看的，动辄贴出几页代码，不适合App开发人员去阅读学习。

我曾经在微信中问过老罗和老邓，你们写的书为什么我们App开发人员看不懂啊，他们就呵呵了，跟我说，他们的书就是写给ROM开发人员看的。

于是，这几年来，我一直在寻找这样一类知识，App开发人员看了能有助于他们更好的编写App程序，而又不需要知道太多这门技术底层的代码实现。

这类知识分为两种。

一种是知道概念即可，就比如说Zygote，其实App开发人员是不需要了解Zygote的，知道有这么个东西是“孕育天地”的就够了，类似的还有SurfaceFlinger、WMS这些概念。

还有一种是需要知道内部原理，就比如说Binder。关于Binder的介绍铺天盖地，但对于我们App开发人员，需要了解的是它的架构模型，只要有Client和Server，以及SM就足够了。

四大组件的底层通信机制都是基于Binder的，我们需要知道每个组件中，分别是哪些类扮演了Binder Client，哪些类扮演了Binder Server。知道这些概念，有助于我们App开发人员进行插件化编程。

### 目录

我这个系列的文章，已经写好了下面的内容，会在接下来的每天发布一篇，共计8篇，看了这8篇文章，就可以迈进Android插件化的大门了。

-    Binder
-    AIDL
-    AMS
-    Activity
-    Service
-    ContentProvider
-    匿名共享内存
-    BroadcastReceiver
-    PMS及App安装过程

Android底层知识，还应该包括以下内容，但是和插件化关系不大，也不是我擅长的领域，所以我只列出了大纲，没有继续写下去：

-    View和ViewGroup
-    Message、Looper和Handler
-    权限管理
-    Android SDK工具内部原理

有兴趣的同学，可以按照我这个思路继续写下去，记得，一，少贴代码。多画图，二，一定要有趣。

接下来就详细讲那些App开发人员需要知道的Android底层知识。

###  Binder

Binder是为了解决跨进程通信。

关于Binder的文章实在是太多了，每篇文章都能从Java层讲到C++层，App开发人员其实是没必要了解这么多内容的。我们看对App开发有用的几个点：

1）首先，Binder分为Client和Server两个进程。

注意，Client和Server是相对的。谁发消息，谁就是Client，谁接收消息，谁就是Server。

举个例子，两个进程A和B之间使用Binder通信，进程A发消息给进程B，那么这时候A是Binder Client，B是Binder Server；进程B发消息给进程A，那么这时候B是Binder Client，A是Binder Server——其实这么说虽然简单了，但还是不太严谨，我们先这么理解着。

2）其次，我们看下面这个图（摘自田维术的博客），基本说明白了Binder的组成解构：

 ![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170516223325025-1448613892.png)

图中的IPC就是进程间通信的意思。

图中的ServiceManager，负责把Binder Server注册到一个容器中。

有人把ServiceManager比喻成电话局，存储着每个住宅的座机电话，还是很恰当的。张三给李四打电话，拨打电话号码，会先转接到电话局，电话局的接线员查到这个电话号码的地址，因为李四的电话号码之前在电话局注册过，所以就能拨通；没注册，就会提示该号码不存在。

对照着Android Binder机制，对着上面这图，张三就是Binder Client，李四就是Binder Server，电话局就是ServiceManager，电话局的接线员在这个过程中做了很多事情，对应着图中的Binder驱动

3）接下来我们看Binder通信的过程，还是摘自田维术博客的一张图：

 ![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170516223354650-984999229.png)



 注：图中的SM也就是ServiceManager。

 我们看到，Client想要直接调用Server的add方法，是不可以的，因为它们在不同的进程中，这时候就需要Binder来帮忙了。

1. 首先是Server在SM这个容器中注册。
2. 其次，Client想要调用Server的add方法，就需要先获取Server对象， 但是SM不会把真正的Server对象返回给Client，而是把Server的一个代理对象返回给Client，也就是Proxy。
3. 然后，Client调用Proxy的add方法，SM会帮他去调用Server的add方法，并把结果返回给Client。

以上这3步，Binder驱动出了很多力，但我们不需要知道Binder驱动的底层实现，涉及到C++的代码了——把有限的时间去做更有意义的事情。

App开发人员对Binder的掌握，这些内容就足够了。

是时候总结一波了：

1. 学习Binder，是为了更好的理解AIDL，基于AIDL模型，进而了解四大组件的原理。
2. 理解了Binder，再看AMS和四大组件的关系，就像是Binder的两个进程Server和Client通信。

### AIDL

 AIDL是Binder的延伸。一定要先看懂我前面介绍的Binder，再来看AIDL。要按顺序阅读。

Android系统中很多系统服务都是aidl，比如说剪切板。举这个例子，是为了让App开发人员知道AIDL无处不在，和我们距离非常近。

 AIDL中需要知道下面几个类：

-    IBinder
-    IInterface
-    Binder
-    Proxy
-    Stub

当我们自定义一个aidl文件时（比如MyAidl.aidl，里面有一个sum方法），Android Studio会帮我们生成一个类文件MyAidl.java，如下图所示：

 ![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170516223432791-1010721944.png)

MyAidl.java这个生成文件中，包括MyAidl接口，以及Stub和Proxy两个实现了MyAidl接口的类，其中Stub是定义在MyAidl接口中的，而Proxy则定义在Stub类中。

我曾经很不理解，为什么不是生成3个文件，一个接口，两个类，清晰明了。都放在一个文件中，这是导致很多人看不懂AIDL的一个门槛。其实Android这么设计是有道理的。当有多个AIDL类的时候，Stub和Proxy类就会重名，把它们放在各自的AIDL接口中，就必须MyAidl.Stub这样去使用，就区分开了。

对照这张图，我们继续来分析，Stub的sum方法是怎么调用到Proxy的sum方法？然后又调用另一个进程的sum方法的？

 起决定意义的是Stub的asInterface方法和onTransact方法。其实这个图没有画全，把完整的Binder Server也画上，就应该是这样：

 ![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170516223504650-228437964.png)

 1）先从Client看起，对于AIDL的使用者，我们这么写程序：

```java
MyAidl.Stub.asInterface(某IBinder对象).sum(1, 2);   //最好在执行sum方法前判空。
```

asInterface方法的作用是判断参数——也就是IBinder对象，和自己是否在同一个进程：

-    是，则直接转换、直接使用，接下来就跟Binder跨进程通信无关啦；
-    否，则把这个IBinder参数包装成一个Proxy对象，这时调用Stub的sum方法，间接调用Proxy的sum方法。

return new MyAidl.Stub.Proxy(obj);



 2）Proxy在自己的sum方法中，会使用Parcelable来准备数据，把函数名称、函数参数都写入 `_data`，让`_reply`接收函数返回值。最后使用IBinder的transact方法，把数据就传给Binder的Server端了。

```java
mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0); //这里的mRemote就是asInterface方法传过来的obj参数
```

 3）Server则是通过onTransact方法接收Client进程传过来的数据，包括函数名称、函数参数，找到对应的函数，这里是sum，把参数喂进去，得到结果，返回。

 所以onTransact函数经历了读数据-->执行要调用的函数-->把执行结果再写数据的过程。

下一篇文章要介绍的四大组件的原理，我们都可以对照着AIDL的这张图来看，比如说，四大组件的启动和后续流程，都是在和ActivityManagerService（简称AMS）来来回回的通信，四大组件给AMS发消息，四大组件就是Binder Client，而AMS就是Binder Server；AMS发消息通知四大组件，那么角色就互换。



那么四大组件中，比如说Activity，又是哪个类扮演了Stub的角色，哪个类扮演了Proxy的角色呢？这也是我下一篇文章要介绍的，包括AMS、四大组件各自的运行原理。

好戏即将开始。