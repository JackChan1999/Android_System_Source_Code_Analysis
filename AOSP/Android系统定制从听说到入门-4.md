### 本文配套视频：

[http://www.365yg.com/item/6434362415713878530/](http://www.365yg.com/item/6434362415713878530/)

### 4. AOSP源码工作环境

源码工作环境，就是安装所依赖的一些软件库，以及开发调试时需要进行的一些配置。
主要分为编译环境准备，AOSP源码下载，源码预编译等。这里主要介绍在Ubuntu14.04 LTS下开发Android6.0代码的工作环境准备。

#### 4.1编译环境搭建（Ubuntu14.04）

```
JDK和依赖包下载
$ sudo apt-get update #获取源的更新
$ sudo apt-get install openjdk-7-jdk #下载openjdk7
```

sunjdk 采用Java研究许可（Java Research License，简称JRL）许可证书，部分开源，仅限研究。openjdk采用GNU许可证书，完全开源。sunjdk中私有APIs用类似功能的开源代码替换/重新实现。Google为了解决与Oracal公司的版权纠纷，在Android 5.0以后JavaAPIS替换为openjdk。不同版本的AOSP代码，所要求的JDK版本也不一样，详附：

安装其他依赖

```
$ sudo apt-get install git-core gnupg flex bison gperf build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386  lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z-dev ccache libgl1-mesa-dev libxml2-utils xsltproc unzip
```

不同版本的Ubuntu系统所需要安装的依赖也是不一样的，在一些低版本即使按照官方文档做，编译的时候也会出现很多错误，所以建议使用Google推荐的Ubuntu系统版本。
USB设备权限配置

在 GNU/Linux系统下（比如Ubuntu），普通用户默认是没有权限去访问USB设备的，如果希望某个用户访问，可以以root身份在/etc/udev/rules.d/ 目录下创建一个rules规则文件，AOSP提供的rules里配置了常见Android设备。

#### 下载并配置AOSP的usb设备权限规则模板

```
$ wget -S -O - http://source.android.com/source/51-android.rules | sed "s/<username>/$USER/" | sudo tee >/dev/null /etc/udev/rules.d/51-android.rules; sudo     udevadm control --reload-rules
```

上述模板只配置了一些Nexus设备，如果是自己的Android设备，执行adb devices的时候可能会提示permission denied，原因就是因为自己的Android设备没有加入到这个配置文件当中。可以利用lsusb命令列出当前系统识别到的USB设备：
$lsusb #获取当前的usb设备，输出类似以下的信息

```
Bus 001 Device 007: ID 0403:cb48 Future Technology Devices International, Ltd
Bus 001 Device 006: ID 0461:4d15 Primax Electronics, Ltd Dell Optical Mouse
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

找出自己的手机设备，以第一行为例，idVendor是0403,idProduct是cb48。
可以在51-android.rules里新增一行：

```
SUBSYSTEM=="usb", ATTR{idVendor}=="0403", ATTR{idProduct}=="cb48", MODE="0600", OWNER="<username>"
```

记得要把<username>替换为自己的登录用户名。然后再执行：
$sudo udevadm control --reload-rules#重新加载USB访问规则
然后重新插拔USB设备即可。