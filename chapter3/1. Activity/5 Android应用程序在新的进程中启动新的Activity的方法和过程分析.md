Android应用程序在新的进程中启动新的Activity的方法和过程分析

---

前面我们在分析Activity启动过程的时候，看到同一个应用程序的Activity一般都是在同一个进程中启动，事实上，Activity也可以像Service一样在新的进程中启动，这样，一个应用程序就可以跨越好几个进程了，本文就分析一下在新的进程中启动Activity的方法和过程。

《[Android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

在前面[Android进程间通信（IPC）机制Binder简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6618363)一文中，我们提到，在[android](http://lib.csdn.net/base/android)系统中，每一个应用程序都是由一些Activity和Service组成的，一般Service运行在独立的进程中，而Activity有可能运行在同一个进程中，也有可能运行在不同的进程中。在前面[Android系统在新进程中启动自定义服务过程（startService）的原理分析](http://blog.csdn.net/luoshengyang/article/details/6677029)一文中，我们已经介绍了使用Activity.startService接口来在新进程中启动Service的过程，然后又在前面[Android应用程序内部启动Activity过程（startActivity）的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6703247)一文中介绍了使用Activity.startActivity接口来在原来的进程中启动Activity的过程，现在，我们就来看一下同一个Android应用程序如何在新的进程中启动新的Activity。

老规矩，我们通过例子来介绍Android应用程序在新的进程中启动新的Activity的方法以及分析其过程。首先在Android源代码工程中创建一个Android应用程序工程，名字就称为Process吧。关于如何获得Android源代码工程，请参考[在Ubuntu上下载、编译和安装Android最新源代码](http://blog.csdn.net/luoshengyang/article/details/6559955)一文；关于如何在Android源代码工程中创建应用程序工程，请参考[在Ubuntu上为Android系统内置Java应用程序测试Application Frameworks层的硬件服务](http://blog.csdn.net/luoshengyang/article/details/6580267)一文。这个应用程序工程定义了一个名为shy.luo.process的package，这个例子的源代码主要就是实现在这里了。下面，将会逐一介绍这个package里面的文件。

应用程序的默认Activity定义在src/shy/luo/process/MainActivity.[Java](http://lib.csdn.net/base/java)文件中：

```java
package shy.luo.process;       
      
import android.app.Activity;      
import android.content.Intent;      
import android.os.Bundle;      
import android.util.Log;      
import android.view.View;      
import android.view.View.OnClickListener;      
import android.widget.Button;      
      
public class MainActivity extends Activity  implements OnClickListener {      
    private final static String LOG_TAG = "shy.luo.process.MainActivity";      
      
    private Button startButton = null;      
      
    @Override      
    public void onCreate(Bundle savedInstanceState) {      
super.onCreate(savedInstanceState);      
setContentView(R.layout.main);      
      
startButton = (Button)findViewById(R.id.button_start);      
startButton.setOnClickListener(this);      
      
Log.i(LOG_TAG, "Main Activity Created.");      
    }      
      
    @Override      
    public void onClick(View v) {      
if(v.equals(startButton)) {      
    Intent intent = new Intent("shy.luo.process.subactivity");      
    startActivity(intent);      
}      
    }      
}      
```
和前面文章的例子一样，它的实现很简单，当点击它上面的一个按钮的时候，就会启动另外一个名字为“shy.luo.process.subactivity”的Actvity。

名字为“shy.luo.process.subactivity”的Actvity实现在src/shy/luo/process/SubActivity.java文件中：

```java
package shy.luo.process;      
      
import android.app.Activity;      
import android.os.Bundle;      
import android.util.Log;      
import android.view.View;      
import android.view.View.OnClickListener;      
import android.widget.Button;      
      
public class SubActivity extends Activity implements OnClickListener {      
    private final static String LOG_TAG = "shy.luo.process.SubActivity";      
      
    private Button finishButton = null;      
      
    @Override      
    public void onCreate(Bundle savedInstanceState) {      
super.onCreate(savedInstanceState);      
setContentView(R.layout.sub);      
      
finishButton = (Button)findViewById(R.id.button_finish);      
finishButton.setOnClickListener(this);      
      
Log.i(LOG_TAG, "Sub Activity Created.");      
    }      
      
    @Override      
    public void onClick(View v) {      
if(v.equals(finishButton)) {      
    finish();      
}      
    }      
}      
```
它的实现也很简单，当点击上面的一个铵钮的时候，就结束自己，回到前面一个Activity中去。

再来重点看一下应用程序的配置文件AndroidManifest.xml：

```xml
<?xml version="1.0" encoding="utf-8"?>      
<manifest xmlns:android="http://schemas.android.com/apk/res/android"      
    package="shy.luo.task"      
    android:versionCode="1"      
    android:versionName="1.0">      
    <application android:icon="@drawable/icon" android:label="@string/app_name">      
<activity android:name=".MainActivity"      
          android:label="@string/app_name">    
          android:process=":shy.luo.process.main"    
    <intent-filter>      
        <action android:name="android.intent.action.MAIN" />      
        <category android:name="android.intent.category.LAUNCHER" />      
    </intent-filter>      
</activity>      
<activity android:name=".SubActivity"      
          android:label="@string/sub_activity"    
          android:process=":shy.luo.process.sub">      
    <intent-filter>      
        <action android:name="shy.luo.task.subactivity"/>      
        <category android:name="android.intent.category.DEFAULT"/>      
    </intent-filter>      
</activity>      
    </application>      
</manifest>     
```
为了使MainActivity和SubActivity在不同的进程中启动，我们分别配置了这两个Activity的android:process属性。在[官方文档](http://developer.android.com/guide/topics/manifest/activity-element.html)中，是这样介绍Activity的android:process属性的：

The name of the process in which the activity should run. Normally, all components of an application run in the default process created for the application. It has the same name as the application package. The[ ](http://developer.android.com/guide/topics/manifest/application-element.html) element's [process](http://developer.android.com/guide/topics/manifest/application-element.html#proc) attribute can set a different default for all components. But each component can override the default, allowing you to spread your application across multiple processes.
If the name assigned to this attribute begins with a colon (':'), a new process, private to the application, is created when it's needed and the activity runs in that process. If the process name begins with a lowercase character, the activity will run in a global process of that name, provided that it has permission to do so. This allows components in different applications to share a process, reducing resource usage.

大意为，一般情况下，同一个应用程序的Activity组件都是运行在同一个进程中，但是，如果Activity配置了android:process这个属性，那么，它就会运行在自己的进程中。如果android:process属性的值以":"开头，则表示这个进程是私有的；如果android:process属性的值以小写字母开头，则表示这是一个全局进程，允许其它应用程序组件也在这个进程中运行。

因此，这里我们以":"开头，表示创建的是私有的进程。事实上，这里我们不要前面的":"也是可以的，但是必须保证这个属性性字符串内至少有一个"."字符，具体可以看一下解析AndroidManiefst.xml文件的源代码，在frameworks/base/core/java/android/content/pm/PackageParser.java文件中：

```java
public class PackageParser {  
  
    ......  
  
    private boolean parseApplication(Package owner, Resources res,  
    XmlPullParser parser, AttributeSet attrs, int flags, String[] outError)  
    throws XmlPullParserException, IOException {  
final ApplicationInfo ai = owner.applicationInfo;  
final String pkgName = owner.applicationInfo.packageName;  
  
TypedArray sa = res.obtainAttributes(attrs,  
    com.android.internal.R.styleable.AndroidManifestApplication);  
  
......  
  
if (outError[0] == null) {  
    CharSequence pname;  
    if (owner.applicationInfo.targetSdkVersion >= Build.VERSION_CODES.FROYO) {  
        pname = sa.getNonConfigurationString(  
            com.android.internal.R.styleable.AndroidManifestApplication_process, 0);  
    } else {  
        // Some older apps have been seen to use a resource reference  
        // here that on older builds was ignored (with a warning).  We  
        // need to continue to do this for them so they don't break.  
        pname = sa.getNonResourceString(  
            com.android.internal.R.styleable.AndroidManifestApplication_process);  
    }  
    ai.processName = buildProcessName(ai.packageName, null, pname,  
        flags, mSeparateProcesses, outError);  
  
    ......  
}  
  
......  
  
    }  
  
    private static String buildProcessName(String pkg, String defProc,  
    CharSequence procSeq, int flags, String[] separateProcesses,  
    String[] outError) {  
if ((flags&PARSE_IGNORE_PROCESSES) != 0 && !"system".equals(procSeq)) {  
    return defProc != null ? defProc : pkg;  
}  
if (separateProcesses != null) {  
    for (int i=separateProcesses.length-1; i>=0; i--) {  
        String sp = separateProcesses[i];  
        if (sp.equals(pkg) || sp.equals(defProc) || sp.equals(procSeq)) {  
            return pkg;  
        }  
    }  
}  
if (procSeq == null || procSeq.length() <= 0) {  
    return defProc;  
}  
return buildCompoundName(pkg, procSeq, "process", outError);  
    }  
  
    private static String buildCompoundName(String pkg,  
    CharSequence procSeq, String type, String[] outError) {  
String proc = procSeq.toString();  
char c = proc.charAt(0);  
if (pkg != null && c == ':') {  
    if (proc.length() < 2) {  
        outError[0] = "Bad " + type + " name " + proc + " in package " + pkg  
            + ": must be at least two characters";  
        return null;  
    }  
    String subName = proc.substring(1);  
    String nameError = validateName(subName, false);  
    if (nameError != null) {  
        outError[0] = "Invalid " + type + " name " + proc + " in package "  
            + pkg + ": " + nameError;  
        return null;  
    }  
    return (pkg + proc).intern();  
}  
String nameError = validateName(proc, true);  
if (nameError != null && !"system".equals(proc)) {  
    outError[0] = "Invalid " + type + " name " + proc + " in package "  
        + pkg + ": " + nameError;  
    return null;  
}  
return proc.intern();  
    }  
  
  
    private static String validateName(String name, boolean requiresSeparator) {  
final int N = name.length();  
boolean hasSep = false;  
boolean front = true;  
for (int i=0; i<N; i++) {  
    final char c = name.charAt(i);  
    if ((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z')) {  
        front = false;  
        continue;  
    }  
    if (!front) {  
        if ((c >= '0' && c <= '9') || c == '_') {  
            continue;  
        }  
    }  
    if (c == '.') {  
        hasSep = true;  
        front = true;  
        continue;  
    }  
    return "bad character '" + c + "'";  
}  
  
return hasSep || !requiresSeparator  
    ? null : "must have at least one '.' separator";  
    }  
  
    ......  
  
}  
```

从调用parseApplication函数解析application标签开始，通过调用buildProcessName函数对android:process属性进解析，接着又会调用buildCompoundName进一步解析，这里传进来的参数pkg就为"shy.luo.process"，参数procSeq为MainActivity的属性android:process的值":shy.luo.process.main"，进一步将这个字符串保存在本地变量proc中。如果proc的第一个字符是":"，则只需要调用validateName函数来验证proc字符串里面的字符都是合法组成就可以了，即以大小写字母或者"."开头，后面可以跟数字或者"_"字符；如果proc的第一个字符不是":"，除了保证proc字符里面的字符都是合法组成外，还要求至少有一个"."字符。

MainActivity和SubActivity的android:process属性配置就介绍到这里了，其它更多的信息读者可以参考[官方文档](http://developer.android.com/guide/topics/manifest/activity-element.html)或者源代码文件frameworks/base/core/java/android/content/pm/PackageParser.java。

再来看界面配置文件，它们定义在res/layout目录中，main.xml文件对应MainActivity的界面：  

```xml
<?xml version="1.0" encoding="utf-8"?>      
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"      
    android:orientation="vertical"      
    android:layout_width="fill_parent"      
    android:layout_height="fill_parent"       
    android:gravity="center">      
<Button       
    android:id="@+id/button_start"      
    android:layout_width="wrap_content"      
    android:layout_height="wrap_content"      
    android:gravity="center"      
    android:text="@string/start" >      
</Button>      
</LinearLayout>      
```
而sub.xml对应SubActivity的界面：

```xml
<?xml version="1.0" encoding="utf-8"?>      
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"      
    android:orientation="vertical"      
    android:layout_width="fill_parent"      
    android:layout_height="fill_parent"       
    android:gravity="center">      
<Button       
    android:id="@+id/button_finish"      
    android:layout_width="wrap_content"      
    android:layout_height="wrap_content"      
    android:gravity="center"      
    android:text="@string/finish" >      
</Button>      
</LinearLayout>    
```
字符串文件位于res/values/strings.xml文件中：

```xml
<?xml version="1.0" encoding="utf-8"?>      
<resources>      
    <string name="app_name">Process</string>      
    <string name="sub_activity">Sub Activity</string>      
    <string name="start">Start activity in new process</string>      
    <string name="finish">Finish activity</string>      
</resources>     
```
最后，我们还要在工程目录下放置一个编译脚本文件Android.mk：

```
LOCAL_PATH:= $(call my-dir)      
include $(CLEAR_VARS)      
      
LOCAL_MODULE_TAGS := optional      
      
LOCAL_SRC_FILES := $(call all-subdir-java-files)      
      
LOCAL_PACKAGE_NAME := Process      
      
include $(BUILD_PACKAGE)  
```
接下来就要编译了。有关如何单独编译Android源代码工程的模块，以及如何打包system.img，请参考[如何单独编译Android源代码中的模块]()一文。

执行以下命令进行编译和打包：

```
USER-NAME@MACHINE-NAME:~/Android$ mmm packages/experimental/Process        
USER-NAME@MACHINE-NAME:~/Android$ make snod     
```
这样，打包好的Android系统镜像文件system.img就包含我们前面创建的Process应用程序了。

再接下来，就是运行模拟器来运行我们的例子了。关于如何在Android源代码工程中运行模拟器，请参考在Ubuntu上下载、编译和安装Android最新源代码一文。

执行以下命令启动模拟器：

```
USER-NAME@MACHINE-NAME:~/Android$ emulator    
```
模拟器启动起，就可以App Launcher中找到Process应用程序图标，接着把它启动起来：

![img](http://hi.csdn.net/attachment/201108/27/0_1314433747FSdJ.gif)

点击中间的按钮，就会在新的进程中启动SubActivity：

![img](http://hi.csdn.net/attachment/201108/27/0_1314433831zXPc.gif)

现在，我们如何来确认SubActivity是不是在新的进程中启动呢？Android源代码工程为我们准备了adb工具，可以查看模拟器上系统运行的状况，执行下面的命令查看：

```
USER-NAME@MACHINE-NAME:~/Android$ adb shell dumpsys activity  
```
这个命令输出的内容比较多，这里我们只关心系统中的任务和进程部分：

```xml
......  
  
Running activities (most recent first):  
    TaskRecord{40770440 #3 A shy.luo.process}  
      Run #2: HistoryRecord{406d4b20 shy.luo.process/.SubActivity}  
      Run #1: HistoryRecord{40662bd8 shy.luo.process/.MainActivity}  
    TaskRecord{40679eb8 #2 A com.android.launcher}  
      Run #0: HistoryRecord{40677570 com.android.launcher/com.android.launcher2.Launcher}  
  
......  
  
PID mappings:  
    ......  
  
    PID #416: ProcessRecord{4064b720 416:shy.luo.process:shy.luo.process.main/10037}  
    PID #425: ProcessRecord{406ddc30 425:shy.luo.process:shy.luo.process.sub/10037}  
  
......  
```
这里我们看到，虽然MainActivity和SubActivity都是在同一个应用程序并且运行在同一个任务中，然而，它们却是运行在两个不同的进程中，这就可以看到Android系统中任务这个概念的强大之处了，它使得我们在开发应用程序的时候，可以把相对独立的模块放在独立的进程中运行，以降低模块之间的耦合性，同时，我们又不必去考虑一个应用程序在两个进程中运行的细节的问题，Android系统中的任务会为我们打点好一切。

在启动Activity的时候，系统是如何做到在新的进程中来启动这个Activity的呢？在前面两篇文章[Android应用程序启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6689748)和[Android应用程序内部启动Activity过程（startActivity）的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6703247)中，分别在Step 22和Step 21中分析了Activity在启动过程中与进程相关的函数ActivityStack.startSpecificActivityLocked函数中，它定义在frameworks/base/services/java/com/android/server/am/ActivityStack.java文件中：

```java
public class ActivityStack {    
    
    ......    
    
    private final void startSpecificActivityLocked(ActivityRecord r,    
    boolean andResume, boolean checkConfig) {    
// Is this activity's application already running?    
ProcessRecord app = mService.getProcessRecordLocked(r.processName,    
    r.info.applicationInfo.uid);    
    
......    
    
if (app != null && app.thread != null) {    
    try {    
        realStartActivityLocked(r, app, andResume, checkConfig);    
        return;    
    } catch (RemoteException e) {    
        ......    
    }    
}    
    
mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,    
    "activity", r.intent.getComponent(), false);    
    }    
    
    
    ......    
    
}    
```
从这个函数可以看出，决定一个Activity是在新的进程中启动还是在原有的进程中启动的因素有两个，一个是看这个Activity的process属性的值，另一个是这个Activity所在的应用程序的uid。应用程序的UID是由系统分配的，而Activity的process属性值，如前所述，是可以在AndroidManifest.xml文件中进行配置的，如果没有配置，它默认就为application标签的process属性值，如果application标签的process属性值也没有配置，那么，它们就默认为应用程序的package名。这里就是根据processName和uid在系统查找是否已有相应的进程存在，如果已经有了，就会调用realStartActivityLocked来直接启动Activity，否则的话，就要通过调用ActivityManagerService.startProcessLocked函数来创建一个新的进程，然后在新进程中启动这个Activity了。对于前者，可以参考[Android应用程序内部启动Activity过程（startActivity）的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6703247)一文，而后者，可以参考[Android应用程序启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6689748)一文。

至此，Android应用程序在新的进程中启动新的Activity的方法和过程分析就结束了。在实际开发中，一个应用程序一般很少会在一个新的进程中启动另外一个Activity，如果真的需要这样做，还要考虑如何与应用程序中其它进程的Activity进行通信，这时候不妨考虑使用[Binder进程间通信机制](http://blog.csdn.net/luoshengyang/article/details/6618363)。写这篇文章的目的，更多是让我们去了解Android应用程序的[架构](http://lib.csdn.net/base/architecture)，这种架构可以使得应用程序组件以松耦合的方式组合在一起，便于后续的扩展和维护，这是非常值得我们学习的。