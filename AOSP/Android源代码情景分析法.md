> 视频地址:http://v.youku.com/v_show/id_XMjUyMDc5OTcxMg==.html

视频大纲

一. 工具

1.1 vim

模式切换 -- i、esc

搜索 -- /<关键字>

跳转到指定行 -- :<行号>

剪切、复制、和粘贴指定行 -- dd、yy、p

剪切、复制、和粘贴指定内容块 -- v、d、y、p

创建新行并进入编辑模式 -- o

Undo、Redo -- u、ctrl + r

保存、退出 -- w、wq、q!

1.2 find + xargs + grep

查找ActivityManagerNative类的子类：

```
find -name '*.java' | xargs grep 'extends[ \n\t]\+ActivityManagerNative'
```

二. 情景分析

一个工程会由很多个模块组成，每一个模块向外提供若干个调用接口。这些调用接口就是我们分析源码的切入点。从这些切入点出发，一步一步地跟踪每一个函数调用，直至终点。这个分析过程就是情景分析。情景分析实际上是把源码划分成一条又一条的线，每一条线都会把相关的功能点串在一起，形成一个上下文。因此，当我们选定了一条线进行分析的时候，就可以把无关的模块晾在一边，最大程度地减少干扰。

在做情景分析的时候，我们要适当地做笔记，主要记录函数的调用过程以及函数的位置。这样不仅能提高分析源码的效率，也方便日后反复查阅。

示例，Activity启动过程分析（基于5.0.0_r1版本源码）：

```
Activity.startActivity frameworks/base/core/java/android/app/Activity.java
  Activity.startActivityForResult frameworks/base/core/java/android/app/Activity.java
    Instrumentation.execStartActivity frameworks/base/core/java/android/app/Instrumentation.java
      ActivityManagerProxy.startActivity frameworks/base/core/java/android/app/ActivityManagerNative.java
 
ActivityManagerNative.onTransact frameworks/base/core/java/android/app/ActivityManagerNative.java
  ActivityManagerService.startActivity frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
    ......
```

详细过程，参考：[Android应用程序启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6689748)

Office Word: 流程图、架构图和示意图

MagicDraw UML: 类图、序列图