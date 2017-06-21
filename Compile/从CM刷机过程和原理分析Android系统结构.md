 前面101篇文章都是分析[Android](http://lib.csdn.net/base/android)系统源码，似乎不够接地气。如果能让[android](http://lib.csdn.net/base/android)系统源码在真实设备上跑跑看效果，那该多好。这不就是传说中的刷ROM吗？刷ROM这个话题是老罗以前一直避免谈的，因为觉得没有全面了解Android系统前就谈ROM是不完整的。写完了101篇文章后，老罗觉得第102篇文章该谈谈这个话题了，并且选择CM这个有代表性的ROM来谈，目标是加深大家对Android系统的了解。

老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)，欢迎关注！

《Android系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

​       说起刷ROM的动机，除了上面说的用来看Android系统源码在真实设备上的运行效果，还有很多值得说的。回想起PC时代，我们对我们自己拥有的设备（电脑），基本上能做的就是在上面重装系统。这个系统是厂商做好给我们的，里面有什么我们就用什么，不能随心所欲地定制。当然，如果你用的是[Linux](http://lib.csdn.net/base/linux)系统，你是可以随心所欲地对它进行定制的。不过可惜的是，我们的女神用的都是Windows系统。你和女神说你想要什么样的[linux](http://lib.csdn.net/base/linux)系统，我给你定制一个，她会不知道你说的是什么——她需要的是一个不会中毒的又跑得快的Windows系统而已。现如今虽然很多女神用的是仍然是我们不能随心所欲定制的[iOS](http://lib.csdn.net/base/ios)系统，但是在移动设备上，[ios](http://lib.csdn.net/base/ios)系统毕竟不能做到Windows在PC那样的一家独大——我们还有不少女神是用Android系统的。所以，如果你现在和女神说，我可以帮你刷一个专属的精简Android系统，里面没有一堆你不需要的预装软件，会让你的手机跑得很快，那女神得有多崇拜你啊。

​       当然，刷ROM的动机不能只是为了让女神崇拜，作为一个程序猿，我们的首要任务是维护宇宙和平。怎么维护呢？至少程序有BUG不能不改吧。你不改的话，老板是不会放过你的。但是，碰到那些很棘手的BUG，怎么办呢？例如，你是一个Android应用开发者，调用一个API接口的时候，总是抛出一个异常，而这个异常不跟到API内部实现去看看实在是不知道什么原因造成的。这时候，如果你手头上有这个手机的系统源代码，找这个API内部实现的地方，加一两行调试代码，再编译回手机上去跑，是不是就很容易定位问题了呢？

​        所以说，会刷ROM，不只是可以获得女神崇拜，还可以维护世界和平，作为一个Android开发者，你还有什么理由不去学习刷ROM呢？既然你决定学习刷ROM了，那你就得先搞清楚两个问题：1. 什么是刷ROM；2. 怎么学习刷ROM。

​        在回答第一个问题之前，我们先来看看Android设备从硬件到系统的结构，如图1所示：

![img](http://img.blog.csdn.net/20140611004823062?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图1 Android系统[架构](http://lib.csdn.net/base/architecture)

​       最底层的是各种硬件设备，往上一层是Bootloader。Bootloader是什么概念呢？我们都知道，PC主板上有一小段程序叫做BIOS，主板加电时它是第一个跑起来的程序，负责初始化硬件，以及将OS启动起来。在[嵌入式](http://lib.csdn.net/base/embeddeddevelopment)世界里（手机也是属于嵌入式设备），也有一小段类似BIOS的程序，不过它不叫BIOS，而是叫Bootloader。使用最广泛的Bootloader是一个叫uboot的程序，它支持非常多的体系结构。经过编译后，uboot会生成一个uboot.bin镜像，将这个镜像烧到设备上的一个特定分区去，就可以作为Bootloader使用了。

​        Bootloader支持交互式启动，也就是我们可以让Bootloader初始化完成硬件之后，不是马上去启动OS，而是停留在当前状态，等待用户输入命令告诉它接下来该干什么。这种启动模块就称为Fastboot模式。对于Android设备来说，我们可以通过adb reboot bootloader命令来让它重新启动并且进入到Fastboot模式中去。

​        在讨论Fastboot模式之前，我们先了解一下嵌入式设备的ROM结构（NAND flash之类的芯片）。通常，一个能够正常启动的嵌入式设备的ROM包含有以下四个分区：

​        1. Bootloader分区，也就是存放uboot.bin的分区

​        2. Bootloader用来保存环境变量的分区

​        3. Kernel分区，也就是存放OS内核的分区

​        4. Rootfs分区，也就是存入系统第一个进程init对应的程序的分区

​        当设备处于Fastboot模式时，我们可以通过另外一个工具fastboot来让设备执行指定的命令。对搞机者来说，最常用的命令就是刷入各种镜像文件了，例如，往Kernel分区和Rootfs分区刷入指定的镜像。

​        对于Android设备来说，当它处于Fastboot模式时，我们可以将一个包含有Kernel和Rootfs的Recovery.img镜像通过fastboot工具刷入到一个称为设备上一个称为Recovery的分区去。这个过程就是刷Recovery了，它也是属于刷ROM的一种。由于Recovery分区包含有Kernel和Rootfs，因此将Recovery.img刷入到设备后，我们就可以让设备正常地启动起来了。这种启动方式就称为Recovery模式。 对于Android设备来说，我们可以通过adb reboot recovery命令来让它进入到Recovery模式中去。

​        当设备处于Recovery模式时，我们可以做些什么呢？答案是取决于刷入的Recovery.img所包含的Rootfs所包含的程序。更确切地说，是取决于Rootfs镜像里面的init程序都做了些什么事情。不过顾名思义，Recovery就是用来恢复系统的意思，也包含有更新系统的意思。这里所说的系统，是用户正常使用的系统，里面包含有Android运行时框架，使得我们可以在上面安装和使用各种APP。

​       用户正常使用Android设备时的系统，主要是包含有两个分区：System分区和Boot分区。System分区包含有Android运行时框架、系统APP以及预装的第三方APP等，而Boot分区包含有Kernel和Rootfs。刷入到System分区和Boot分区的两个镜像称为system.img和boot.img，我们通常将它们打包和压缩为一个zip文件，例如update.zip，并且将它上传到Android设备上的sdcard上去。这样当我们进入到Recovery模式时，就可以在Recovery界面上用我们之前上传到sdcard的zip包来更新用户正常使用Android设备时所用的系统了。这个过程就是通常所说的刷ROM了。

​       不知道大家看明白了没有？广义上的刷ROM，实际上包含更新Recovery和更新用户正常使用的系统两个意思；而狭义上的刷ROM，只是更新用户正常使用的那个系统。更新Recovery需要进入到Fastboot模式中，而更新用户正常使用的那个系统需要进入到Recovery模式中。Android设备在启动的过程中，在默认情况下，一旦Bootloader启动完成，就会直接启动用户正常使用的那个系统，而不会进入到Recovery模式，或者停留在Bootloader中，也就是停留在Fastboot模式中。只有通过特定的命令，例如adb reboot recovery和adb reboot bootloader，或者特定的按键，例如在设备启动过程中同时按住音量减小键和电源开关键，才能让设备进入到Recovery模式或者Fastboot模式中。

​       因此，一个完整的刷ROM过程，包含以下两个步骤：

​       1. 让设备进入到Fastboot模式，刷入一个recovery.img镜像

​       2. 让设备进入到Recovery模式，刷入一个包含system.img镜像和boot.img镜像的zip包

​       不过需要注意的是，system.img镜像和boot.img镜像不一定是只有在Recovery模式才能刷入，在Fastboot模式下也是可以刷入的，就像在Fastboot模式中刷入recovery.img镜像一样，只不过在Recovery模式下刷入它们更友好一些。说到这里，就不得不说另外一个概念，就是所谓的Bootloader锁。在锁定Bootloader的情况下，我们是无法刷入非官方的recovery.img、system.img和boot.img镜像的。这是跟厂商实现的Bootloader相关的，它们可以通过一定的[算法](http://lib.csdn.net/base/datastructure)（例如签名）来验证要刷入的镜像是否是官方发布的。在这种情况下，必须要对Bootloader进行解锁，我们才可以刷入非官方的镜像。

​       好了，以上就回答了什么是刷ROM这个问题，接下来我们要回答的是如常学习刷ROM这个问题。

​       前面我们提到了刷ROM的两个步骤，实际上我们还少了一个重要的步骤，那就是先要制作recovery.img、system.img和boot.img。有人可能会说，网上不是很多现成的刷机包吗？直接拿过来用不就是行了吗？但是别忘了前面我们所说的刷ROM动机：随心所欲地定制自己的系统。去拿别人制好的刷机包就失去了随心所欲定制的能力。那就只能自己去编译AOSP源码生成刷机包了。

​       然而，从零开始从AOSP源码中编译出能在自己使用的手机上运行的系统，可不是一件容易的事情。不过，好在有很多现成的基于AOSP的第三方开源项目，可以编译出来在目前市场上大部分的手机上运行。其中，最著名的就是[CyanogenMod](http://www.cyanogenmod.org/)了，简称CM。国内大部分的第三方Android系统，都是基于CM来开发的，包括MIUI和锤子。

​       选择CM来讲解刷ROM的过程是本文的主题。不过单就刷ROM这个过程来说，CM官网上已经有很详细的过程，包括从CM源码编译出适用特定机型的刷机包，以及将编译出来的刷机包刷到手机里面去的过程。因此，本文不是单纯地讲解刷ROM过程，而是要结合原理来讲解刷ROM过程的关键步骤，来达到帮助大家更进一步地理解Android的目的。

​        要真正做到理解CM ROM的刷机原理，除了要对Android系统本身有一定的认识之外，还熟练掌握Android的源码管理系统和编译系统。因此，在继续阅读下面的内容之前，希望可以先阅读前面[Android源代码仓库及其管理工具Repo分析](http://blog.csdn.net/luoshengyang/article/details/18195205)和[Android编译系统简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/18466779)这两个系列的文章。接下来我们就开始讲解CM ROM的刷机过程和原理。

​        我们首先是要准备好环境以及手机。这是本次操作所用的环境以及手机：

​        1. Ubuntu 13.04

​        2. CM-10.1

​        3. OPPO Find 5

​        也就是说，我们将在Ubuntu 13.04上为OPPO Find 5制作CM-10.1的Recovery和ROM。

​        不过先别急着制作自己的ROM。为了保证接下来的步骤可以顺利执行，我们首先尝试刷一下CM官方对应版本的Recovery和ROM到我们的OPPO Find 5手机上。如果一切正常，就说明我们使用CM的源码来制作的Recovery和ROM也是可以运行在OPPO Find 5上的。

​        适合OPPO Find 5的CM官方Recovery下载地址：[http://download2.clockworkmod.com/recoveries/recovery-clockwork-6.0.4.6-find5.img](http://download2.clockworkmod.com/recoveries/recovery-clockwork-6.0.4.6-find5.img)。假设我们下载好之后，保存在本地的路径为$CM/recovery-clockwork-6.0.4.6-find5.img。

​        适合OPPO Find 5的CM官方10.1.3版本ROM下载地址：[http://download.cyanogenmod.org/get/jenkins/42498/cm-10.1.3-find5.zip](http://download.cyanogenmod.org/get/jenkins/42498/cm-10.1.3-find5.zip)。假设我们下载好之后，保存在本地的路径为$CM/cm-10.1.3.find5.zip。

​        注意，由于我们计划用CM-10.1源码来制作自己的ROM，所以我们在下载CM官方ROM，也要下载对应10.1版本的。

​        在刷Recovery和ROM的过程中，我们需要借助于Android SDK里面的fastboot和adb工具，因此，为了方便执行这些命令，我们先将这些工具的目录加入到PATH环境变量去。假设我们下载的Android SDK保存在目录$ASDK中，那么打开一个终端，执行以下命令即可：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. $ export PATH=$ASDK/platform-tools:$PATH  

​        先刷Recovery，步骤如下所示：

​        1. 保持OPPO Find 5在正常开机状态，并且通USB连接到将有Ubuntu 13.04的电脑上。

​        2. 还是在刚才打开的终端上，并且进入到保存recovery-clockwork-6.0.4.6-find5.img的目录$CM。

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. $ cd $CM  

​        3. 执行以下命令让OPPO Find 5重启，并且进入Fastboot模式。

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. $ adb reboot bootloader  

​        4. 可以看到OPPO Find 5停留在Fastboot界面上，执行以下命令确保fastboot工具能够连接到OPPO Find 5。

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. $ fastboot devices  

​       如果能够连接，那么上述命令将会输出一串标识OPPO Find 5的ID。

​       5. 刷入我们刚才下载的Recovery。

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. $ fastboot flash recovery recovery-clockwork-6.0.4.6-find5.img  

​       6. 提示刷入成功后，执行以下命令正常重启手机。

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. $ fastboot reboot  

​       如果一切正常，手机将进入到原来的系统中。

​       继续在上述打开的终端上，刷CM-10.1.3 ROM，步骤如下所示：

​       1. 将下载好的cm-10.1.3.find5.zip上传至OPPO Find 5的sdcard上

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. $ adb push cm-10.1.3.find5.zip /sdcard/cm-10.1.3.find5.zip  

​       2. 执行以下命令让OPPO Find 5重启，并且进入Recovery模式。

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. $ adb reboot recovery  

​       进入到Recovery模式后，我们将看到显示的Recovery版本号为6.0.4.6，这表明我们现在进入的就是刚才我们刷入的Recovery。

​       3. 在刷入新的ROM前，我们先备份一下当前的ROM，以防万一刷机失败，可以进行恢复。在Recovery界面中，通过音量增大/减小键，选中“backup and restore”选项，按下电源键，进入下一个界面，同样是通过音量增大/减小键，选中“backup”，按下电源键，就可以对当前系统进行备份了。

​       4. 备份完成之后，我们还要清除手机上的数据，恢复至出厂设置。回到Recovery界面中，通过音量增大/减小键，选中"wipe data/factory reset"，按下电源键，确认后即可进行清除数据，并且恢复至出厂设置。

​       5. 清除数据完成之后，再回到Recovery界面上，通过音量增大/减小键，选中“install zip”选项，按下电源键，进入下一个界面，同样是通过音量增大/减小键，选中“choose zip from sdcard”，按下电源键，找到前面我们上传至sdcard的cm-10.1.3.find5.zip，确认之后就可以进行刷机了。

​       6. 刷机完成后，再回到Recovery界面上，通过音量增大/减小键，选中“reboot system now”选项，按下电源键，正常启动系统。

​       如果一切正常，手机将进入到刚才刷入的CM-10.1.3系统中。

​       现在我们就可以确定OPPO Find 5可以正常运行CM-10.1.3的系统了。接下来激动人心的时刻就要开始了，我们将要自己下载和编译CM-10.1源码，并且将编译出来的Recovery和ROM刷入到OPPO Find 5去。同时，在接下来的步骤中，我们会将相关的原理讲清楚，以便我们可以更好地理解Android系统的结构，这也是本文的重点之一。以下假设我们将CM-10.1源码保存在目录$CMSOURCE中，并且已经按照Android官网文档的要求初始化好Android的源码编译环境，即在我们的Ubuntu机器上安装了要求的软件，详情请参考：[http://source.android.com/source/initializing.html](http://source.android.com/source/initializing.html)。

​        1. 进入到$CMSOURCE目录中。

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. $ cd $CMSOURCE  

​        2. 将当前目录初始为CM-10.1分支源码的Git仓库。

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. $ repo init -u git://github.com/CyanogenMod/android.git -b cm-10.1  

​        3. 下载CM-10.1分支源码。

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. $ repo sync  

​        以上两步都是关于Android源码仓库的知识，可以参考

Android源代码仓库及其管理工具Repo分析

一文，这里不再详述。

​        4. 进入到$CMSOURCE目录下的vendor/cm子目录中，并且执行里面的get-prebuilts脚本，用来获得一些预编译的APP。

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. $ ./get-prebuilts  

​        打开$CMSOURCE/vendor/cm/get-prebuilts文件，它的内容如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. BASEDIR=`dirname $0`  
2.   
3. mkdir -p $BASEDIR/proprietary  
4.   
5. \# Get Android Terminal Emulator (we use a prebuilt so it can update from the Market)  
6. curl -L -o $BASEDIR/proprietary/Term.apk -O -L http://jackpal.github.com/Android-Terminal-Emulator/downloads/Term.apk  
7. unzip -o -d $BASEDIR/proprietary $BASEDIR/proprietary/Term.apk lib/*  

​        我们可以发现，实际上这里只是去下载一个叫做Android Terminal Emulator的APP，地址是http://jackpal.github.com/Android-Terminal-Emulator/downloads/Term.apk。这个APP最终会包含在我们自己编译出来的ROM。它用来Android手机上模拟出一个终端来，然后我们就可以像在Linux主机上一样执行一些常用的Linux命令。是不是很酷呢？原来Android手机不单止可以运行我们常见的APP，还可以运运我们常用的Linux命令。关于这个Android Terminal Emulator的安装和介绍，参可以这里：

https://github.com/jackpal/Android-Terminal-Emulator

。此外，这个Android Terminal Emulator还可以配合另外一个封装了busybox的kbox工具，用来在Android手机上获得更多的Linux常用命令，kbox的安装和介绍，可以参考这里：

http://kevinboone.net/kbox2_install.html

。

​        5. 回到$CMSOURCE目录中，将build子目录下的envsetup.sh脚本加载到当前终端来。

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. $ source build/envsetup.sh  

​        参考

Android编译系统环境初始化过程分析

一文，envsetup.sh脚本加载到当前终端后，我们就可以获得一系列与Android编译系统相关的命令，例如lunch/m/mm/mmm，以及下一步要执行的breakfast命令。

​        6. 为OPPO Find 5下载相关的源码。

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. $ breakfast find5  

​        在

Android编译系统简要介绍和学习计划

这个系列的文章中，我们提到，在编译Android的官方源码之前，我们需要执行一个lunch命令来为我们的目标设备初始化编译环境。这个lunch命令是由Android官方源码的envsetup.sh脚本提供的。CM修改envsetup.sh脚本，额外提供了一个breakfast命令，用来从网上寻找指定的设备相关的源码，以便我们可以为该设备编译出能运行的ROM来。

​        打开envsetup.sh文件，查看breakfast的实现：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. function breakfast()  
2. {  
3. ​    target=$1  
4. ​    CM_DEVICES_ONLY="true"  
5. ​    unset LUNCH_MENU_CHOICES  
6. ​    add_lunch_combo full-eng  
7. ​    for f in `/bin/ls vendor/cm/vendorsetup.sh 2> /dev/null`  
8. ​        do  
9. ​            echo "including $f"  
10. ​            . $f  
11. ​        done  
12. ​    unset f  
13.   
14. ​    if [ $# -eq 0 ]; then  
15. ​        # No arguments, so let's have the full menu  
16. ​        lunch  
17. ​    else  
18. ​        echo "z$target" | grep -q "-"  
19. ​        if [ $? -eq 0 ]; then  
20. ​            # A buildtype was specified, assume a full device name  
21. ​            lunch $target  
22. ​        else  
23. ​            # This is probably just the CM model name  
24. ​            lunch cm_$target-userdebug  
25. ​        fi  
26. ​    fi  
27. ​    return $?  
28. }  

​        函数breakfast主要是做了以下两件事情。

​        第一件事情是检查vendor/cm目录下是否存在一个vendorsetup.sh文件。如果存在的话，就将它加载到当前终端来。注意，这里是通过ls命令来检查文件endor/cm/vendorsetup.sh是否存在的。如果不存在的话，标准输出就为空，而错误信息会重定向至/dev/null。如果存在的话，字符串“endor/cm/vendorsetup.sh”就会输出到标准输出来，也就是变量f的值会等于“endor/cm/vendorsetup.sh”。

​        接下来我们就看看文件endor/cm/vendorsetup.sh的内容：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. for combo in $(curl -s https://raw.github.com/CyanogenMod/hudson/master/cm-build-targets | sed -e 's/#.*$//' | grep cm-10.1 | awk {'print $1'})  
2. do  
3. ​    add_lunch_combo $combo  
4. done  

​        它所做的工作就是将

https://raw.github.com/CyanogenMod/hudson/master/cm-build-targets

的内容下载回来，并且去掉其中的空行，最后将含有"cm-10.1"的行的第1列取出来，并且通过add_lunch_combo命令将其加入到Android编译系统的lunch菜单去。

​        从[https://raw.github.com/CyanogenMod/hudson/master/cm-build-targets](https://raw.github.com/CyanogenMod/hudson/master/cm-build-targets)下载回来的是官方CM所支持的机型列表，格式为cm_<product>-<variant> <version>，以下列出的是部分内容：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. \# CM build target list  
2. \# <lunchcombo> <branch> [period: "D"aily, "W"eekly or "M"onthly]  
3. \# Absence of a period indicates Daily (the default)  
4.   
5. cm_a700-userdebug cm-11.0  
6. cm_acclaim-userdebug cm-11.0  
7. cm_amami-userdebug cm-11.0  
8. cm_anzu-userdebug jellybean M  
9. cm_apexqtmo-userdebug cm-11.0  
10. cm_aries-userdebug cm-11.0  
11. cm_captivatemtd-userdebug cm-11.0  
12. ......  

​        由此可见，执行脚本endor/cm/vendorsetup.sh之后，cm-10.1所支持的机型就会增加到lunch菜单中去。

​        回到函数breakfast中，它所做的第二件事情是检查执行函数是否带有参数，即变量target的值是否等于空。如果变量target的值不等于空，并且它的值是<product>-<variant>的形式，那么就直接以它为参数，调用lunch函数。否则的话，就以cm_$target-userdebug为参数，调用lunch函数。

​        在这一步中，我们调用breakfast函数时，传进来的参数为find5，不是<product>-<variant>的形式，因此，函数breakfast最后会以cm_find5-userdebug为参数，调用lunch函数。

​        函数lunch的实现我们在前面一篇文章[Android编译系统环境初始化过程分析](http://blog.csdn.net/luoshengyang/article/details/18928789)已经分析过了，不过当时分析的是AOSP官方版本的实现，CM对其进行了一些修改，增加了一些CM自有的逻辑，下面我们就看看这些修改：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. function lunch()  
2. {  
3. ​    ......  
4.   
5. ​    local product=$(echo -n $selection | sed -e "s/-.*$//")  
6. ​    check_product $product  
7. ​    if [ $? -ne 0 ]  
8. ​    then  
9. ​        # if we can't find a product, try to grab it off the CM github  
10. ​        T=$(gettop)  
11. ​        pushd $T > /dev/null  
12. ​        build/tools/roomservice.py $product  
13. ​        popd > /dev/null  
14. ​        check_product $product  
15. ​    else  
16. ​        build/tools/roomservice.py $product true  
17. ​    fi  
18. ​      
19. ​    ......  
20. }  

​       这里的变量selection的值就等于我们传进来的参数"cm_find5-userdebug"，通过sed命令将"cm_find5"提取出来，并且赋值给变量product。接下来调用check_product函数来检查当前的CM源码中是否支持find5这个设备。

​       如果不支持的话，那么它的返回值就不等于0，即#?不等于0，那么接下来就会通过build/tools/roomservice.py到CM源码服务器去检查是否支持find5这个设备。如果CM源码服务器支持find5这个设备的话，那么build/tools/roomservice.py就会将与find5相关的源码下载回来。这时候我们就会发现本地CM源码目录中多了一个device/oppo/find5目录。里面存放的都是编译find5的ROM时所要用的文件。

​       另一方面，如果当前的CM源码中已经支持find5这个设备，那么函数lunch也会调用build/tools/roomservice.py去CM源码服务器检查当前CM源码目录中find5设备依赖的其它源码是否有更新，或者是否有新的依赖。如果有的话，就将这些依赖更新下载回来。

​       脚本build/tools/roomservice.py的详细内容这里就不分析了，下面主要是解释一下与Android的源码仓库管理工具Repo相关的逻辑。关于Android的源码仓库管理工具Repo的详细分析，可以参考[Android源代码仓库及其管理工具Repo分析](http://blog.csdn.net/luoshengyang/article/details/18195205)一文。

​       CM源码服务器放在github上，地址为[http://github.com/CyanogenMod](http://github.com/CyanogenMod)，上面保存的是CM修改过的AOSP工程、CM支持的设备相关源码工程（下载回来放在device/<manufacturer>/<device>目录中），以及CM支持的设备对应的内核源码工程（下载回来放在kernel/<manufacturer>/<device>目录中）。

​       脚本build/tools/roomservice.py会根据传进来的第一个参数，到CM源码服务上检查是否存在相应的工程。在我们这个场景中，传给build/tools/roomservice.py的第一个参数为cm_find5。这时候前面的cm_会被去掉，然后到CM源码服务上检查是否存在一个android_device_<manufacturer>_find5的工程。如果存在的话，那么就会将它下载回来，保存在device/<manufacturer>/find5目录中。这里的<manufacturer>对应的就是oppo了。

​       下载回来的设备相关源码实际上是作为是一个[Git](http://lib.csdn.net/base/git)仓库来管理的，因此，脚本build/tools/roomservice.py还需要将该[git](http://lib.csdn.net/base/git)仓库纳入到Repo仓库去管理，以便以后执行repo sync命令时，可以同时对这些设备相关的源码进行更新。

​        从[Android源代码仓库及其管理工具Repo分析](http://blog.csdn.net/luoshengyang/article/details/18195205)一文可以知道，Repo仓库保存在.repo目录中，而它所管理的Git仓库由.repo/manifest.xml文件描述。文件.repo/manifest.xml实际上只是一个符号链接，它链接至.repo/manifests/default.xml文件。目录.repo/manifests实际上也是一个Git仓库，用来描述当前的CM源码目录都是由哪些工程构成的，并且这些工程是来自于哪些Git远程仓库的。

​       如果按照标准的Repo仓库管理方法，从CM源码服务器上下载回来设备相关源码之后，应该往.repo/manifests/default.xml文件增加相应的描述，以后repo工具可以对这些设备相关的源码进行管理。但是，由于Repo仓库是由官方维护的，当我们在本地往.repo/manifests/default.xml增加了新的内容之后，下次执行repo sync命令时，.repo/manifests/default.xml的内容又会被恢复至修改前的样子，因此，修改.repo/manifests/default.xml文件是不适合的。CM采用另外一个办法，那就是在.repo目录下另外创建一个local_manifests目录，在里面可以随意增加任意命名的xml文件，只要这些xml文件的规范与.repo/manifests/default.xml文件的规范一致即可。执行repo sync命令时，它就会同时从.repo/manifests/default.xml和.repo/local_manifests目录下的xml文件中读取当前都有哪些源码工程需要更新。

​       实际上，在.repo/local_manifests目录下的xml文件，除了可以描述新增的工程之外，还可以描述要删除的工程。例如，如果我们不想将某一个系统功能或者系统APP编译到我们自己制作的ROM去，那么就可以在.repo/local_manifests目录下增加一个xml文件，里面描述我们需要删除对应的工程。这样，当我们从服务器下载回来相应的工程之后，它们就会在本地中被删除。这样就做到了很好的定制化编译，而且又不会与官方的源码结构产生冲突。关于CM的Local Manifests机制，可以参考官方文档：[http://wiki.cyanogenmod.org/w/Doc:_Using_manifests](http://wiki.cyanogenmod.org/w/Doc:_Using_manifests)。

​       脚本build/tools/roomservice.py将下载回来的设备相关源码纳入到Repo仓库管理的办法就是在.repo/local_manifests目录下创建一个roomservice.xml文件。例如，当我们从CM源码服务器下载回来find5相关的设备源码之后，就可以看到在roomservice.xml文件中看到相应的一行内容：

**[html]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. <?xml version="1.0" encoding="UTF-8"?>  
2. <manifest>  
3.   ......  
4.   <project name="CyanogenMod/android_device_oppo_find5" path="device/oppo/find5" remote="github" />  
5.   ......  
6. </manifest>  

​       这表明本地的device/oppo/find5目录是来自于远程仓库github的，并且相对路径为CyanogenMod/android_device_oppo_find5。

​       好了，现在我们终于将OPPO Find 5相关的设备源码下载回来了，但是在编译之前。需要从OPPO Find 5上提取一些设备相关的私有文件。

​       7. 保持OPPO Find 5开机状态，并且通过USB连接到Ubuntu 13.04上，进行到$CMSOURCE/device/oppo/find5目录中,执行以下命令提取设备私有文件。

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. $ ./extract-files.sh  

​       脚本extract-files.sh的内容如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. \#!/bin/sh  
2.   
3. VENDOR=oppo  
4. DEVICE=find5  
5.   
6. BASE=../../../vendor/$VENDOR/$DEVICE/proprietary  
7. rm -rf $BASE/*  
8.   
9. for FILE in `cat proprietary-blobs.txt | grep -v ^# | grep -v ^$ | sed -e 's#^/system/##g'`; do  
10. ​    DIR=`dirname $FILE`  
11. ​    if [ ! -d $BASE/$DIR ]; then  
12. ​        mkdir -p $BASE/$DIR  
13. ​    fi  
14. ​    adb pull /system/$FILE $BASE/$FILE  
15. done  
16.   
17. ./setup-makefiles.sh  

​       首先是创建一个vendor/oppo/find5/proprietary目录，接着是读取文件proprietary-blobs.txt中的每一行，并且将每一行所描述的文件从设备上的/system目录中获取出来，保存在vendor/oppo/find5/proprietary对应的子目录下面，最后再执行另外一个脚本setup-makefiles.sh。

​       文件device/oppo/find5/proprietary-blobs.txt部分的内容如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. /bin/btnvtool  
2. /bin/ds_fmc_appd  
3. /bin/efsks  
4. /bin/hci_qcomm_init  
5. /bin/ks  
6. /bin/mm-qcamera-daemon  
7. /bin/mpdecision  
8. /bin/netmgrd  
9. /bin/nv_tee  
10. /bin/qcks  
11. /bin/qmuxd  
12. ......  

​       这里列出的文件路径都是相对于设备上的/system目录的，并且都是设备特定的、不公开源码的，因此，我们需要从设备上获取出来。

​       再来看脚本setup-makefiles.sh的内容：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. \#!/bin/sh  
2.   
3. VENDOR=oppo  
4. DEVICE=find5  
5. OUTDIR=vendor/$VENDOR/$DEVICE  
6. MAKEFILE=../../../$OUTDIR/$DEVICE-vendor-blobs.mk  
7.   
8. (cat << EOF) > $MAKEFILE  
9. \# Copyright (C) 2013 The CyanogenMod Project  
10. \#  
11. \# Licensed under the Apache License, Version 2.0 (the "License");  
12. \# you may not use this file except in compliance with the License.  
13. \# You may obtain a copy of the License at  
14. \#  
15. \#      http://www.apache.org/licenses/LICENSE-2.0  
16. \#  
17. \# Unless required by applicable law or agreed to in writing, software  
18. \# distributed under the License is distributed on an "AS IS" BASIS,  
19. \# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.  
20. \# See the License for the specific language governing permissions and  
21. \# limitations under the License.  
22.   
23. \# This file is generated by device/$VENDOR/$DEVICE/setup-makefiles.sh  
24.   
25. PRODUCT_COPY_FILES += \\  
26. EOF  
27.   
28. LINEEND=" \\"  
29. COUNT=`cat proprietary-blobs.txt | grep -v ^# | grep -v ^$ | wc -l | awk {'print $1'}`  
30. for FILE in `cat proprietary-blobs.txt | grep -v ^# | grep -v ^$ | sed -e 's#^/system##g' -e 's#^/##g'`; do  
31. ​    COUNT=`expr $COUNT - 1`  
32. ​    if [ $COUNT = "0" ]; then  
33. ​        LINEEND=""  
34. ​    fi  
35. ​    echo "    $OUTDIR/proprietary/$FILE:system/$FILE$LINEEND" >> $MAKEFILE  
36. done  
37.   
38. (cat << EOF) > ../../../$OUTDIR/$DEVICE-vendor.mk  
39. \# Copyright (C) 2013 The CyanogenMod Project  
40. \#  
41. \# Licensed under the Apache License, Version 2.0 (the "License");  
42. \# you may not use this file except in compliance with the License.  
43. \# You may obtain a copy of the License at  
44. \#  
45. \#      http://www.apache.org/licenses/LICENSE-2.0  
46. \#  
47. \# Unless required by applicable law or agreed to in writing, software  
48. \# distributed under the License is distributed on an "AS IS" BASIS,  
49. \# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.  
50. \# See the License for the specific language governing permissions and  
51. \# limitations under the License.  
52.   
53. \# This file is generated by device/$VENDOR/$DEVICE/setup-makefiles.sh  
54.   
55. \# Pick up overlay for features that depend on non-open-source files  
56. DEVICE_PACKAGE_OVERLAYS := vendor/$VENDOR/$DEVICE/overlay  
57.   
58. \$(call inherit-product, vendor/$VENDOR/$DEVICE/$DEVICE-vendor-blobs.mk)  
59. EOF  
60.   
61. (cat << EOF) > ../../../$OUTDIR/BoardConfigVendor.mk  
62. \# Copyright (C) 2013 The CyanogenMod Project  
63. \#  
64. \# Licensed under the Apache License, Version 2.0 (the "License");  
65. \# you may not use this file except in compliance with the License.  
66. \# You may obtain a copy of the License at  
67. \#  
68. \#      http://www.apache.org/licenses/LICENSE-2.0  
69. \#  
70. \# Unless required by applicable law or agreed to in writing, software  
71. \# distributed under the License is distributed on an "AS IS" BASIS,  
72. \# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.  
73. \# See the License for the specific language governing permissions and  
74. \# limitations under the License.  
75.   
76. \# This file is generated by device/$VENDOR/$DEVICE/setup-makefiles.sh  
77.   
78. USE_CAMERA_STUB := false  
79. EOF  

​       这个脚本主要就是用来在vendor/oppo/find5目录下生成两个文件：find5-vendor-blobs.mk和BoardConfigVendor.mk文件。这两个文件都是接下来为OPPO Find 5编译ROM时要用到的。

​       生成的find5-vendor-blobs.mk的部分内容如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. PRODUCT_COPY_FILES += \  
2. ​    vendor/oppo/find5/proprietary/bin/btnvtool:system/bin/btnvtool \  
3. ​    vendor/oppo/find5/proprietary/bin/ds_fmc_appd:system/bin/ds_fmc_appd \  
4. ​    vendor/oppo/find5/proprietary/bin/efsks:system/bin/efsks \  
5. ​    vendor/oppo/find5/proprietary/bin/hci_qcomm_init:system/bin/hci_qcomm_init \  
6. ​    vendor/oppo/find5/proprietary/bin/ks:system/bin/ks \  
7. ......  

​       实际上就是通过PRODUCT_COPY_FILES变量告诉编译系统，要将刚才从设备上获取回来的文件打包到编译出来的ROM的system分区里面去。

​       生成的文件BoardConfigVendor.mk只有1行，如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. USE_CAMERA_STUB := false  

​       在编译摄像头相关的库文件时，就会到这里定义的USE_CAMERA_STUB变量。

​       到目前为止，我们就分别从CM源码服务器和目标设备上获取到了编译OPPO Find 5的ROM所需要的设备相关的文件了，接下来就可以开始编译CM源码了。

​       8. 回到$CMSOUCE目录中，为OPPO Find 5编译ROM。

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. $ brunch find5  

​       命令brunch也是由build/envsetup.sh脚本提供的，它的实现如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. function brunch()  
2. {  
3. ​    breakfast $*  
4. ​    if [ $? -eq 0 ]; then  
5. ​        mka bacon  
6. ​    else  
7. ​        echo "No such item in brunch menu. Try 'breakfast'"  
8. ​        return 1  
9. ​    fi  
10. ​    return $?  
11. }  

​       函数brunch做了两件事情：执行breakfast命令和执行mka bacon命令。

​       前面我们不是已经执行过breakfast命令了吗？这里为什么又要再执行一次呢？从前面的分析可以知道，函数breakfast在执行的过程中，会调用函数lunch，而函数lunch又会调用函数check_product来检查目标设备是否存在。因为在前一步中，我们已经将目标设备相关的源码下载回来了，因此这时候目标设备是肯定存在的。当目标设备存在的时候，函数lunch会通过build/tools/roomservice.py脚本检查目标设备相关的源码是否依赖有其它工程，并且这些工程是否已经下载回来了。如果有依赖的工程，并且这些工程还没有下载回来，那么就需要将它们下载回来。

​       当目标设备存在的时候，函数lunch执行build/tools/roomservice.py脚本的形式为：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. build/tools/roomservice.py $product true  

​       传递给build/tools/roomservice.py第二件参数为true，表示要处理的是目标设备$product依赖的工程。

​       脚本build/tools/roomservice.py是如何知道目标设备有没有依赖的工程的呢？原来，在下载回来的设备源码中，有一个cm.dependencies文件，里面描述了自己所依赖的工程。例如，device/oppo/find5/cm.dependencies的内容如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. [  
2.  {  
3.    "repository": "android_kernel_oppo_find5",  
4.    "target_path": "kernel/oppo/find5",  
5.    "branch": "cm-10.1"  
6.  }  
7. ]  

​       上面描述的是OPPO Find 5所使用的内核源码，它来自于CM源码服务器上的android_kernel_oppo_find5仓库的cm-10.1分支，下载回来后保存在kernel/oppo/find5目录中。这意味着我们通过CM源码编译出来的ROM所使用的内核也是由我们自己编译出来的。

​       为了以后执行repo sync命令同步本地源码时，也可以将设备源码依赖的工程也同时同步回来，我们需要将这些依赖工程也纳入到Repo仓库中去，因此，当再次执行过breakfast使命之后。我们就可以在.repo/local_manifests/roomservice.xml文件中发现以下两行内容：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. <?xml version="1.0" encoding="UTF-8"?>  
2. <manifest>  
3.   ......  
4.   <project name="CyanogenMod/android_device_oppo_find5" path="device/oppo/find5" remote="github" />  
5.   <project name="CyanogenMod/android_kernel_oppo_find5" path="kernel/oppo/find5" remote="github" revision="cm-10.1" />  
6.   ......  
7. </manifest>  

​      回到函数brunch中，它接下来执行的命令mka也是由build/envsetup.sh脚本提供的，如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. function mka() {  
2. ​    case `uname -s` in  
3. ​        Darwin)  
4. ​            make -j `sysctl hw.ncpu|cut -d" " -f2` "$@"  
5. ​            ;;  
6. ​        *)  
7. ​            schedtool -B -n 1 -e ionice -n 1 make -j$(cat /proc/cpuinfo | grep "^processor" | wc -l) "$@"  
8. ​            ;;  
9. ​    esac  
10. }  

​       它实际上是通过一个叫做schedtool的工具来调用make工具对源码进行编译。我们知道，编码Android源码是一个漫长的过程，而现在的机器都是多核的，为了加快这个编译过程，需要将机器的所有核以都充分利用起来。工具schedtool所做的事情就是充分地利机器的多核特性来执行make命令，使得我们可以尽快结束编译过程。

​       关于Android源码的编译详细过程，可以参考[Android编译系统简要介绍和学习计划这个系列](http://blog.csdn.net/luoshengyang/article/details/18466779)的文章，这里只进行简要的说明。

​       从[Android编译系统简要介绍和学习计划这个系列](http://blog.csdn.net/luoshengyang/article/details/18466779)一文可以知道，Android的编译系统是由很多的mk文件组成的，每一个mk文件都是一个Makefile脚本片段。在编译开始之前，这些mk文件会组合在一起，形成一个很大的Makefile文件，然后再根据这个很大的Makefile文件的指令对源码进行编译。

​       由这些mk文件组成的Makefile文件内容可以抽象为四个层次，如图2所示：

![img](http://img.blog.csdn.net/20140614152648031?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图2 Android编译系统层次

​       最下面一层描述的设备CPU的体系架构（Architecture），Android设备支持arm、x86和mips三种CPU体系架构。再往上一层描述的是设备所使用的芯片（Board），例如用的是高通的芯片，还是三星的芯片，等等，与这些芯片相关的源文件存放在hardware目录下。接下来再往上的一层是设备（Device），描述的是具体的硬件设备。最上面的一层是产品（Product），描述的是在硬件设备上运行的软件模块。

​       在这一步中，我们通过brunch命令编译CM源码时，指定的唯一参数是find5，那么Android编译系统是如何根据这个参数来找包含上述四个层次的mk文件为OPPO Find 5编译出能正常运行的ROM的呢？

​       在我们执行breakfast或者brunch命令的过程中，会调用另外一个函数check_product。根据[Android编译系统环境初始化过程分析](http://blog.csdn.net/luoshengyang/article/details/18928789)一文，函数check_product会通过另外一个函数get_build_var加载build/core/config.mk文件。文件build/core/config.mk又会继续加载另外一个文件build/core/envsetup.mk。最后，文件build/core/envsetup.mk又会加载另外一个文件build/core/product_config.mk。

​       在build/core/product_config.mk文件的加载过程中，有以下的一段逻辑：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. ifneq ($(strip $(TARGET_BUILD_APPS)),)  
2. \# An unbundled app build needs only the core product makefiles.  
3. all_product_configs := $(call get-product-makefiles,\  
4. ​    $(SRC_TARGET_DIR)/product/AndroidProducts.mk)  
5. else  
6.   ifneq ($(CM_BUILD),)  
7. ​    all_product_configs := $(shell ls device/*/$(CM_BUILD)/cm.mk)  
8.   else  
9. ​    # Read in all of the product definitions specified by the AndroidProducts.mk  
10. ​    # files in the tree.  
11. ​    all_product_configs := $(get-all-product-makefiles)  
12.   endif # CM_BUILD  
13. endif  

​       当我们编译的是整个Android源码时，变量TARGET_BUILD_APPS的值等于空。这时候就会判断是否设置了一个名称为CM_BUILD的环境变量。如果设置了的话，那么接下来就会加载device/*/$(CM_BUILD)目录下的cm.mk文件来获得目标产品配置文件。否则，首先会通过调用另外一个函数get-all-product-makefiles来获得/device/*/*目录下的所有名称为AndroidProducts.mk的文件，接着再在这些AndroidProducts.mk文件中定义的PRODUCT_MAKEFILES变量来获得目标产品配置文件。

​       环境变量CM_BUILD的值是在函数check_product中设置的，并且只有在CM源码中编译时才会设置。在AOSP源码编译时，是不会设置环境变量CM_BUILD的。CM版本的check_product函数实现如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. function check_product()  
2. {  
3. ​    T=$(gettop)  
4. ​    if [ ! "$T" ]; then  
5. ​        echo "Couldn't locate the top of the tree.  Try setting TOP." >&2  
6. ​        return  
7. ​    fi  
8.   
9. ​    if (echo -n $1 | grep -q -e "^cm_") ; then  
10. ​       CM_BUILD=$(echo -n $1 | sed -e 's/^cm_//g')  
11. ​       if [ `uname` == "Darwin" ]; then  
12. ​           export BUILD_NUMBER=$((date +%s%N ; echo $CM_BUILD; hostname) | openssl sha1 | cut -c1-10)  
13. ​       else  
14. ​           export BUILD_NUMBER=$((date +%s%N ; echo $CM_BUILD; hostname) | sha1sum | cut -c1-10)  
15. ​       fi  
16. ​    else  
17. ​       CM_BUILD=  
18. ​    fi  
19. ​    export CM_BUILD  
20.   
21. ​    CALLED_FROM_SETUP=true BUILD_SYSTEM=build/core \  
22. ​        TARGET_PRODUCT=$1 \  
23. ​        TARGET_BUILD_VARIANT= \  
24. ​        TARGET_BUILD_TYPE= \  
25. ​        TARGET_BUILD_APPS= \  
26. ​        get_build_var TARGET_DEVICE > /dev/null  
27. ​    # hide successful answers, but allow the errors to show  
28. }  

​        从这里就可以看出，环境变量CM_BUILD的值实际上就是目标设备的名称。例如，前面我们执行breakfast命令时，通过函数lunch调用check_product函数时，传进来的参数为为cm_find5，函数check_product会将find5提取出来，并且赋值给环境变量CM_BUILD。这样在加载build/core/product_config.mk文件时，找到的产品配置文件就为device/*/find5/cm.mk文件。在device目录中，只有oppo子目录包含有find5这个目录，因此最终加载的实际上是device/oppo/find5/cm.mk文件。

​        无论是通过环境变量CM_BUILD来直接获得的目标产品配置文件，还是通过AndroidProducts.mk文件间接获得的目标产品配置文件，获得的目标产品配置文件都会直接或者间接地通过变量PRODUCT_COPY_FILES和PRODUCT_PACKAGES来设备要包含的软件模块。

​        以device/oppo/find5/cm.mk为例，它的部分内容如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. ......  
2. \# Inherit device configuration  
3. $(call inherit-product, device/oppo/find5/full_find5.mk)  
4.   
5. \## Device identifier. This must come after all inclusions  
6. PRODUCT_DEVICE := find5  
7. PRODUCT_NAME := cm_find5  
8. PRODUCT_BRAND := Oppo  
9. PRODUCT_MODEL := Find 5  
10. PRODUCT_MANUFACTURER := Oppo  
11. ......  

​        除定了一些设备相关的变量之外，它还会加载另外一个文件device/oppo/find5/full_find5.mk，后者的部分内容如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. \# Inherit from hardware-specific part of the product configuration  
2. $(call inherit-product, device/oppo/find5/device.mk)  
3. $(call inherit-product-if-exists, vendor/oppo/find5/find5-vendor.mk)  

​       文件device/oppo/find5/full_find5.mk又会继续加载另外两个文件device/oppo/find5/device.mk和vendor/oppo/find5/find5-vendor.mk。

​       在文件device/oppo/find5/device.mk中，我们就可以看到PRODUCT_COPY_FILES和PRODUCT_PACKAGES的定义：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. PRODUCT_PACKAGES := \  
2. ​    lights.msm8960  
3.   
4. PRODUCT_PACKAGES += \  
5. ​    charger_res_images \  
6. ​    charger  
7.   
8. \# Vold and Storage  
9. PRODUCT_COPY_FILES += \  
10. ​        device/oppo/find5/configs/vold.fstab:system/etc/vold.fstab  
11.   
12. \# Live Wallpapers  
13. PRODUCT_PACKAGES += \  
14. ​        LiveWallpapers \  
15. ​        LiveWallpapersPicker \  
16. ​        VisualizationWallpapers \  
17. ​        librs_jni  
18. ......  

​       而文件vendor/oppo/find5/find5-vendor.mk会加载另外一个文件vendor/oppo/find5/find5-vendor-blobs.mk，如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. ......  
2. $(call inherit-product, vendor/oppo/find5/find5-vendor-blobs.mk)  
3. ......  

​        回忆前面的第7步，文件文件vendor/oppo/find5/find5-vendor-blobs.mk是我们执行extract-files.sh脚本的时候生成的，里面通过变量PRODUCT_COPY_FILES告诉编译系统将从设备上获得的私有文件打包到要制作的ROM去。

​        以上就是Product级别的配置信息的获取过程，接下来我们再看Device、Board和Architecture级别的配置信息的获取过程。

​        函数check_product会通过另外一个函数get_build_var来加载build/core/config.mk文件的过程中，会在目标产品对应的设备目录中找到一个名称为BoardConfig.mk文件，如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. ......  
2. board_config_mk := \  
3. ​    $(strip $(wildcard \  
4. ​        $(SRC_TARGET_DIR)/board/$(TARGET_DEVICE)/BoardConfig.mk \  
5. ​        device/*/$(TARGET_DEVICE)/BoardConfig.mk \  
6. ​        vendor/*/$(TARGET_DEVICE)/BoardConfig.mk \  
7. ​    ))  
8. ......  

​        上述Makefile脚本片段会在build/target/board、device/*、vendor/*目录中寻找一个名称为$(TARGET_DEVICE)的子目录，并且在找到的子目录里面加载一个名称为BoardConfig.mk文件，来获得Device、Board和Architecture级别的配置信息。

​        变量TARGET_DEVICE指向的便是目标设备的名称。在我们这个场景中，它的值就等于find5，因此，最终获得的Device、Board和Architecture级别的配置信息就是来自于device/oppo/find5/BoardConfig.mk文件，它的部分内容如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. TARGET_CPU_ABI := armeabi-v7a  
2. TARGET_CPU_ABI2 := armeabi  
3. ......  
4. TARGET_ARCH := arm  
5. ......  
6.   
7. \# Bluetooth  
8. BOARD_HAVE_BLUETOOTH := true  
9. BOARD_HAVE_BLUETOOTH_QCOM := true  
10. BLUETOOTH_HCI_USE_MCT := true  
11. ......  
12.   
13. TARGET_BOARD_PLATFORM := msm8960  
14. ......  
15.   
16. \# Wifi  
17. BOARD_HAS_QCOM_WLAN              := true  
18. BOARD_WLAN_DEVICE                := qcwcn  
19. ......  
20.   
21. \# Display  
22. TARGET_QCOM_DISPLAY_VARIANT := caf  
23. USE_OPENGL_RENDERER := true  
24. ......  
25.   
26. \# GPS  
27. BOARD_HAVE_NEW_QC_GPS := true  
28. ......  
29.   
30. \# Camera  
31. COMMON_GLOBAL_CFLAGS += -DDISABLE_HW_ID_MATCH_CHECK -DQCOM_BSP_CAMERA_ABI_HACK -DNEW_ION_API  
32. ......  
33.   
34. \# Audio  
35. BOARD_USES_ALSA_AUDIO:= true  
36. TARGET_QCOM_AUDIO_VARIANT := caf  
37. TARGET_USES_QCOM_MM_AUDIO := true  
38. TARGET_USES_QCOM_COMPRESSED_AUDIO := true  
39. BOARD_AUDIO_CAF_LEGACY_INPUT_BUFFERSIZE := true  
40. ......  

​        从这里就可以看到各种Device、Board和Architecture级别的配置信息。例如，CPU体系架构由TARGET_ARCH、TARGET_CPU_ABI和TARGET_CPU_ABI2来描述。芯片类型由TARGET_BOARD_PLATFORM来描述，而设备信息的描述则包括Bluetooth、Wifi、Display、GPS、Camera和Audio等。

​        在BoardConfig.mk文件中配置的编译信息是在什么时候用到的呢？以下我们就以TARGET_BOARD_PLATFORM为例，说明这些配置信息在编译的过程中是如何使用的。在Android源码中，hardware目录包含各种芯片相关的模块源码。在这些模块的编译脚本中，就会根据TARGET_BOARD_PLATFORM的值来为指定的芯片编译出正确的模块来。

​        例如，假设我们使用的是高通芯片，在它的Camera HAL2模块源码目录hardware/qcom/camera/QCamera/HAL2/core中定义的Android.mk文件有如下的内容：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/29688041#) [copy](http://blog.csdn.net/luoshengyang/article/details/29688041#)

1. ......  
2. LOCAL_CFLAGS += -DCAMERA_ION_HEAP_ID=ION_CP_MM_HEAP_ID # 8660=SMI, Rest=EBI  
3. LOCAL_CFLAGS += -DCAMERA_ZSL_ION_HEAP_ID=ION_CP_MM_HEAP_ID  
4.   
5. LOCAL_CFLAGS+= -DHW_ENCODE  
6. LOCAL_CFLAGS+= -DUSE_NEON_CONVERSION  
7.   
8. ifeq ($(TARGET_BOARD_PLATFORM),msm8960)  
9. ​        LOCAL_CFLAGS += -DCAMERA_GRALLOC_HEAP_ID=GRALLOC_USAGE_PRIVATE_MM_HEAP  
10. ​        LOCAL_CFLAGS += -DCAMERA_GRALLOC_FALLBACK_HEAP_ID=GRALLOC_USAGE_PRIVATE_IOMMU_HEAP  
11. ​        LOCAL_CFLAGS += -DCAMERA_ION_FALLBACK_HEAP_ID=ION_IOMMU_HEAP_ID  
12. ​        LOCAL_CFLAGS += -DCAMERA_ZSL_ION_FALLBACK_HEAP_ID=ION_IOMMU_HEAP_ID  
13. ​        LOCAL_CFLAGS += -DCAMERA_GRALLOC_CACHING_ID=0  
14. else ifeq ($(TARGET_BOARD_PLATFORM),msm8660)  
15. ​        LOCAL_CFLAGS += -DCAMERA_GRALLOC_HEAP_ID=GRALLOC_USAGE_PRIVATE_CAMERA_HEAP  
16. ​        LOCAL_CFLAGS += -DCAMERA_GRALLOC_FALLBACK_HEAP_ID=GRALLOC_USAGE_PRIVATE_CAMERA_HEAP # Don't Care  
17. ​        LOCAL_CFLAGS += -DCAMERA_ION_FALLBACK_HEAP_ID=ION_CAMERA_HEAP_ID # EBI  
18. ​        LOCAL_CFLAGS += -DCAMERA_ZSL_ION_FALLBACK_HEAP_ID=ION_CAMERA_HEAP_ID  
19. ​        LOCAL_CFLAGS += -DCAMERA_GRALLOC_CACHING_ID=0  
20. else  
21. ​        LOCAL_CFLAGS += -DCAMERA_GRALLOC_HEAP_ID=GRALLOC_USAGE_PRIVATE_ADSP_HEAP  
22. ​        LOCAL_CFLAGS += -DCAMERA_GRALLOC_FALLBACK_HEAP_ID=GRALLOC_USAGE_PRIVATE_ADSP_HEAP # Don't Care  
23. ​        LOCAL_CFLAGS += -DCAMERA_GRALLOC_CACHING_ID=GRALLOC_USAGE_PRIVATE_UNCACHED #uncached  
24. endif  
25. ......  

​       上述Makefile脚本片段会根据TARGET_BOARD_PLATFORM的值不同而设置不同的LOCAL_CFLAGS值，以便可以为目标设备编译出正确的HAL模块来。

​       这一步执行完成之后，也就是编译完成之后，我们就可以在out/target/product/find5目录中看到两个文件：recovery.img和cm-10.1-xxxxxxxx-UNOFFICIAL-find5.zip。其中，recovery.img是Recovery文件，而cm-10.1-xxxxxxxx-UNOFFICIAL-find5.zip是ROM文件。有了这两个文件之后，我们就可以参照前面的介绍，将它们刷入我们的OPPO Find 5手机上去了。当刷入完成，并且可以正常运行之后，现在我们的手机上面运行的Recovery和ROM都是我们自己亲手打造的了！

​       至此，我们就介绍完成CM的刷机过程和原理了，是不是觉得自己亲手去编译一个ROM是一件很容易的事呢？其实不然。我们上面编译的ROM之所以这么轻松，是因为CM已经为我们做了很多的工作，那就是已经在CM源码服务器上准备好了所有设备相关的源码，也就是我们下载回来之后存放在device目录下的源码。如果我们手头上有一部CM官方不支持的手机，那么CM源码服务器上是没有对应的设备源码存在的，这时候我们就需要自己去开发对应的设备源码了。这是一个艰苦的过程，需要不停的调试、调试、再调试，也许要花上几周，甚至一个月来完成这个工作。当然，这个过程也是有一些经验和技巧可以参考的，具体可以参考CM文档：[http://wiki.cyanogenmod.org/w/Doc:_porting_intro](http://wiki.cyanogenmod.org/w/Doc:_porting_intro)。这里就不再详述。

​       最后，本文之所以选择CM这个第三方ROM和源码来讲解，是因为CM官方提供了很全面的资料供我们去学习如何基于AOSP源码来制作ROM，这样可以使我们少走很多弯路！另外，上面所述的ROM过程都是参考了CM官方文档的。更多的CM刷ROM教程，可以参考：[http://wiki.cyanogenmod.org/w/Development](http://wiki.cyanogenmod.org/w/Development)。更多信息也可以关注老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)。