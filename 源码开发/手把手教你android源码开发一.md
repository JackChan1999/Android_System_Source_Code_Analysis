- [手把手教你android源码开发一](http://www.jianshu.com/p/c01ce375e77a)
- [手把手教你android源码开发二](http://www.jianshu.com/p/6b2de1c4a1bc)

本文配套视频：
[https://v.qq.com/x/page/j0515fs0k2e.html](https://v.qq.com/x/page/j0515fs0k2e.html)

android源码之前分为四层，如下图：

![img](http://upload-images.jianshu.io/upload_images/4037105-ffccdac6439d8283.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

新的android源码分为五层，增加了HAL层，如下图：

![img](https://alleniverson.gitbooks.io/android-interview/content/Android/img/framework.png)

### 1.源码级开发（系统开发）

源码级开发就是基于AOSP（Android Open Source Project）环境的开发工作，主要开发场景为Android系统定制，比如手机设备的MIUI,Flyme，Smartisan OS,基于电视的LetvUI，甚至还有针对投影仪，路由器等传统物联设备的Android系统定制开发。

#### 1.1 Android系统分层

HAL层：（Hardware Abstract Layer）硬件抽象层。Android系统里封装内核驱动程序的接口层。对上层提供接口，屏蔽底层驱动实现细节.

本来Linux内核可以负责驱动接口定义和驱动实现，但是受限于GNU License（开源感染性），如果厂商选择驱动接口和实现都在内核空间完成，就必须开放自己的驱动源代码。这是不符合厂商利益的（驱动包含核心硬件参数，与其他厂家竞争的法宝）。所以Google将Linux内核中跟底层硬件操作相关的逻辑封装成HAL层接口，厂商基于接口去实现，不直接在内核空间实现驱动。因为Android系统遵循Apache License，不强制开源。

#### 1.2三方应用开发与源码级开发的区别

三方应用开发是基于Android SDK开发。主要技术方向为APP及混合APP开发，数据库，网络协议，应用架构等，服务于商业APP需求。

源码级开发是基于AOSP环境开发，主要技术方向为系统应用开发，Framework开发，底层浏览器内核开发，音视频编解码开发，虚拟机开发，底层驱动开发等。服务于系统定制需求。

#### 1.3 AOSP官网

AOSP官网提供系统开发相关指导，比如源码的环境搭建，下载，编译，维护，更新版本，开放驱动的下载等。

![img](http://upload-images.jianshu.io/upload_images/4037105-56d4bbe84bed421e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 欢迎关注微信公众号、长期为您推荐优秀博文、开源项目、视频
- 微信公众号名称：Android干货程序员

![img](http://upload-images.jianshu.io/upload_images/4037105-8f737b5104dd0b5d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)