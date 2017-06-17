### ContentProvider

#### ContentProvider是什么？

ContentProvider，简称CP。

做App开发的同学，尤其是电商类App，对CP并不熟悉，对这个概念的最大程度的了解，也仅仅是建立在书本上，它是Android四大组件中的一个。

做系统管理类的App，比如说手机助手这种，有机会频繁使用CP。

而对于应用类App，数据通常存在服务器端，其它应用类App也想使用的时候，一般都是从服务器取数据，所以没机会使用到CP。

有时候我们会在自己的App中读取通信录或者短信的数据，这时候就需要用到CP了。通信录或者短信的数据，是以CP的形式提供的，我们在App这边，是使用方。

对于做应用类App的同学，很少有机会自定义CP供其它App使用。

我们快速回顾一下在App中怎么使用CP。

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170526222000935-1998749550.png)

1）定义CP的App1：

在App1中定义一个CP的子类MyContentProvider，并在Manifest中声明，为此要在MyContentProvider中实现CP的增删改查四个方法：

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170526222020138-495218737.png)

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170526222029575-1530590653.png)

2）使用CP的App2：

在App2访问App1中定义的CP，为此，要使用到ContentResolver，它也提供了增删改查4个方法，用于访问App1中定义的CP：

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170526222040325-1806998309.png)

首先我们看一下ContentResolver的增删改查这4个方法的底层实现，其实都是和AMS通信，最终调用App1的CP的增删改查4个方法，后面我们会讲到这个流程是怎么样的。

其次，URI是CP的身份证，唯一标识。

我们在App1中为CP声明URI，也就是authorities的值为baobao，那么在App2中想使用它，就在ContentResolver的增删改查4个方法中指定URI，格式为：

uri = Uri.parse("content://baobao/");

接下来把两个App都进入debug模式，就可以从App2调试进入App1了，比如说，query操作。

#### CP的本质

CP的本质是把数据存储在SQLite数据库中。

各种数据源，有各种格式，比如短信、通信录，它们在SQLite中就是不同的数据表，但是对外界的使用者而言，就需要封装成统一的访问方式，比如说对于数据集合而言，必须要提供增删改查四个方法，于是我们在SQLite之上封装了一层，也就是CP。

#### 匿名共享内存（ASM）

CP读取数据使用到了匿名共享内存，英文简称ASM，所以你看上面CP和AMS通信忙的不亦乐乎，其实下面别有一番风景。

关于ASM的概念，它其实也是个Binder通信，我画个图哦，你们就明白了：

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170526222051654-2107606435.png)

什么？还没看懂？那我再画一个类的交互关系图：

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170526222101591-2145340358.png)

这里的CursorWindow就是匿名共享内存。

这个流程，简单来说是这样的：

1）Client内部有一个CursorWindow对象，发送请求的时候，把这个CursorWindow类型的对象传过去，这个对象暂时为空。

2）Server收到请求，搜集数据，填充到这个CursorWindow对象。

3）Client读取内部的这个CursorWindow对象，获取到数据。

由此可见，这个CursorWindow对象，就是匿名共享内存，这是同一块匿名内存。   

举个生活中的例子就是，你定牛奶，在你家门口放个箱子，送牛奶的人每天早上往这个箱子放一袋牛奶，你睡醒了去箱子里取牛奶。这个牛奶箱就是匿名共享内存。

#### CP与AMS的通信流程

接下来我们看一下CP是怎么和AMS通信的。

能坚持看到这里的人，都不容易。我努力多贴图，不贴代码，即使有代码，也是App开发人员能看懂的代码。

还是拿App2想访问App1中定义的CP为例子。我们就看CP的insert方法。

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170526222112216-35003614.png)

上面这5行代码，包括了启动CP和执行CP方法两部分，分水岭在insert方法，insert方法的实现，前半部分仍然是在启动CP，当CP启动后获取到CP的代理对象，后半部分是通过代理对象，调用insert方法。

整体的流程如下图所示：

![img](http://images2015.cnblogs.com/blog/13430/201705/13430-20170526222122529-365974622.png)

1）App2发送消息给AMS，想要访问App1中的CP。

2）AMS检查发现，App1中的CP没启动过，为此新开一个进程，启动App1，然后获取到App1启动的CP，把CP的代理对象返回给App2。

3）App2拿到CP的代理对象，也就是IContentProvider，就调用它的增删改查4个方法了，接下来就是使用ASM来传输数据或者修改数据了，也就是上面提到的CursorWindow这个类，取得数据或者操作结果即可，作为App的开发人员，不需要知道太多底层的详细信息，用不上。

至此，关于CP的介绍就结束了。下一篇文章，我们看一下App的安装流程，也就PMS。