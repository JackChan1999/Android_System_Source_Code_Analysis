### App内部的页面跳转

在介绍完App的启动流程后，我们发现，其实就是启动一个App的首页。

接下来我们看App内部页面的跳转。

从ActivityA跳转到ActivityB，其实可以把ActivityA看作是Launcher，那么这个跳转过程，和App的启动过程就很像了。

有了前面的分析基础，会发现，这个过程不需要重新启动一个新的进程，所以可以省略App启动过程中的一些步骤，流程简化为：

1）ActivityA向AMS发送一个启动ActivityB的消息。

2）AMS保存ActivityB的信息，然后通知App，你可以休眠了（onPaused）。

3）ActivityA进入休眠，然后通知AMS，我休眠了。

4）AMS发现ActivityB所在的进程就是ActivityA所在的进程，所以不需要重新启动新的进程，所以它就会通知App，启动ActivityB。

5）App启动ActivityB。

不想看上述文字的，看我画的这个图：

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170519230735275-563343566.png)

整体流程我就不多说了，和上一篇文章介绍的App启动流程是基本一致的。

以上的分析，仅限于ActivityA和ActivityB在相同的进程中，如果在Manifest中指定这两个Activity不在同一个进程中，那么就又是另一套流程了，但是整体流程大同小异。

### Context家族史

Activity和Service都有Context，这三个类，还有Application，其实是亲戚一家子。

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170519230746869-193964502.png)

Activity因为有了一层Theme，所以中间有个ContextThemeWrapper，相当于它是Service和Application的侄子。

ContextWrapper只是一个包装类，没有任何具体的实现，真正的逻辑都在ContextImpl里面。

一个应用包含的Context个数：Service个数+Activity个数+1(Application类本身对应一个Context对象)。

应用程序中包含多个ContextImpl对象，而其内部变量mPackageInfo指向同一个PackageInfo对象。

我们就拿Activity举例子，看看Activity和Context的联系和区别。

我们知道，跳转到一个新的Activity要这么写：

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170519230800588-1214836011.png)

我们还知道，也可以在Activity中使用getApplicationContext方法获取Context上下文信息，然后使用Context 的startActivity方法，启动一个新的Activity：

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170519230811869-1855137329.png)

这二者的区别是什么？我们画个图，就看明白了：

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170519230823603-908308741.png)

因为Context的startActivity方法，我看了在ContextImpl中的源码实现，仍然是从ActivityThread中取出Instrumentation，然后执行execStartActivity方法，这和使用Activity的startActivity方法的流程是一样的。

还记得我们前面分析的App启动流程么？在第五阶段，创建App进程的时候，先创建的ActivityThread，再创建的Application。Application的生命周期是跟着整个App走的。

而getApplicationContext得到的Context，就是从ActivityThread中取出来的Application对象，所以这个Context上下文，使用时要当心，容易引起内存泄露。

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170519230840885-574938888.png)

下一篇文章，我们就要迈进Service的世界了。