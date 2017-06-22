Android系统在启动的过程中，会扫描系统中的特定目录，以便对保存在里面的应用程序进行安装。这是通过PackageManagerService来实现的。PackageManagerService在安装一个应用程序的过程中，主要完成两件事情：

- 第一件事情是解析这个应用程序的配置文件AndroidManifest.xml，以便可以获得它的安装信息
- 第二件事情是为这个应用程序分配Linux用户ID和Linux用户组ID，以便它可以在系统中获得合适的运行权限。

Android应用程序启动过程的最后一个是启动一个Home应用程序，用来显示系统中已经安装了的应用程序。Android系统提供了一个默认的Home应用程序———Launcher。应用程序Launcher在启动的过程中，首先会请求PackageManagerService返回系统中已经安装了的应用程序的信息，接着再将这些应用程序信息封装成一个快捷图标显示在系统的屏幕中，以便用户可以通过点击这些快捷图标来启动相应的应用程序。

## 应用程序的安装工程

PackageManagerService在安装一个应用程序的过程中，会对这个应用的配置文件AndroidManifest.xml进行解析，以便获得它的安装信息。一个应用程序可以配置的安装信息有很多，其中，最重要的就是它的组件信息。一个应用程序组件只有在配置文件中被准确配置之后，才可以被ActivityManagerService正常的启动起来。

Android系统中的每一个应用程序都有一个Linux用户ID。这些Linux用户ID就是PackageManagerService来分配的。

一个应用程序除了拥有一个Linux用户ID之外，还可以拥有若干个Linux用户组ID，以便在系统中获得更多的资源访问权限，例如，读取联系人信息、使用摄像头、发送短信，以及拨打电话等权限。这些Linux用户组ID也是由PackageManagerService来分配的。

## PackageManagerService

| 方法声明                 | 功能描述                              |
| :------------------- | :-------------------------------- |
| updatePermissionLP() | 为申请了特定的资源访问权限的应用程序分配相应的Linux用户组ID |
| scanDirLI()          | 安装特定目录下的应用程序                      |
| scanPackageLI()      |                                   |
| readPermissions()    |                                   |

Linux用户ID、Linux用户组ID

## PackageParser

- parseApplication()

## Settings

- readLP()
- readPackageLP()
- readShareUserLP()
- writeLP()
- /data/system/package.xml 保存应用程序安装信息
- /data/system/package-backup.xml

## Environment

| 方法声明               | 功能描述    |
| :----------------- | :------ |
| getDataDirectory() | /data   |
| getRootDirectory() | /system |

- /data/app-private：受DRM保护的私有应用程序
- /data/app：用户自己安装的应用程序
- /system/framework：保护的应用程序是资源型的
- /system/app：保护的是系统自带的应用程序
- /vendor/app：保存的是设备厂商提供的应用程序

## 应用程序的显示过程

Android系统提供了一个默认的Home应用程序———Launcher，用来显示系统中已经安装了的应用程序，它是由System进程负责启动的，同时它也是系统中第一个启动的应用程序。

将系统中已经安装了的应用程序显示出来的目的是为了让用户有一个统一的入口来启动它们。一个根Activity组件的启动过程就代表了一个应用程序的启动过程。因此，应用程序Launcher在显示一个应用程序时，只需要获得它的根Activity组件信息即可。

一个应用程序的根Activity组件的类型被定义为CATEGORY_LAUNCHER，同时它的Action名称被定义为ACTION_MAIN，这样应用程序Launcher就可以通过这两个条件来请求PackageManagerService返回系统中所有已经安装了的应用程序的根Activity组件的信息。

获得了系统中已经安装了的应用程序的根Activity组件信息之后，应用程序Launcher就会分别将它们封装成一个快捷图标，显示在系统的屏幕中，这样用户就可以通过点击这些快捷图标来启动相应的应用程序了。

System进程是由Zygote进程负责启动的，而System进程在启动过程中，又会创建一个ServerThread线程来启动系统中的关键服务。当系统中的关键服务都启动起来之后，这个ServerThread线程就会通知ActivityManagerService将应用程序Launcher启动起来。