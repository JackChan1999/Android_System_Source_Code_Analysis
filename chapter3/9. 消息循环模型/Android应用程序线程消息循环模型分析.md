我们知道，[Android](http://lib.csdn.net/base/android)应用程序是通过消息来驱动的，即在应用程序的主线程（UI线程）中有一个消息循环，负责处理消息队列中的消息。我们也知道，[android](http://lib.csdn.net/base/android)应用程序是支持多线程的，即可以创建子线程来执行一些计算型的任务，那么，这些子线程能不能像应用程序的主线程一样具有消息循环呢？这些子线程又能不能往应用程序的主线程中发送消息呢？本文将分析Android应用程序线程消息处理模型，为读者解答这两个问题。

《Android系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

在开发Android应用程序中，有时候我们需要在应用程序中创建一些常驻的子线程来不定期地执行一些不需要与应用程序界面交互的计算型的任务。如果这些子线程具有消息循环，那么它们就能够常驻在应用程序中不定期的执行一些计算型任务了：当我们需要用这些子线程来执行任务时，就往这个子线程的消息队列中发送一个消息，然后就可以在子线程的消息循环中执行我们的计算型任务了。我们在前面一篇文章[Android系统默认Home应用程序（Launcher）的启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6767736)中，介绍Launcher的启动过程时，在Step 15（LauncherModel.startLoader）中，Launcher就是通过往一个子线程的消息队列中发送一个消息（sWorker.post(mLoaderTask)），然后子线程就会在它的消息循环中处理这个消息的时候执行从PackageManagerService中获取系统中已安装应用程序的信息列表的任务，即调用Step 16中的LoaderTask.run函数。

在开发Android应用程序中，有时候我们又需要在应用程序中创建一些子线程来执行一些需要与应用程序界面进交互的计算型任务。典型的应用场景是当我们要从网上下载文件时，为了不使主线程被阻塞，我们通常创建一个子线程来负责下载任务，同时，在下载的过程，将下载进度以百分比的形式在应用程序的界面上显示出来，这样就既不会阻塞主线程的运行，又能获得良好的用户体验。但是，我们知道，Android应用程序的子线程是不可以操作主线程的UI的，那么，这个负责下载任务的子线程应该如何在应用程序界面上显示下载的进度呢？如果我们能够在子线程中往主线程的消息队列中发送消息，那么问题就迎刃而解了，因为发往主线程消息队列的消息最终是由主线程来处理的，在处理这个消息的时候，我们就可以在应用程序界面上显示下载进度了。

上面提到的这两种情况，Android系统都为我们提供了完善的解决方案，前者可以通过使用HandlerThread类来实现，而后者可以使用AsyncTask类来实现，本文就详细这两个类是如何实现的。不过，为了更好地理解HandlerThread类和AsyncTask类的实现，我们先来看看应用程序的主线程的消息循环模型是如何实现的。

## 应用程序主线程消息循环模型

在前面一篇文章[Android应用程序进程启动过程的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6747696)一文中，我们已经分析应用程序进程（主线程）的启动过程了，这里主要是针对它的消息循环模型作一个总结。当运行在Android应用程序框架层中的ActivityManagerService决定要为当前启动的应用程序创建一个主线程的时候，它会在ActivityManagerService中的startProcessLocked成员函数调用Process类的静态成员函数start为当前应用程序创建一个主线程：

```java
public final class ActivityManagerService extends ActivityManagerNative      
implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {      
      
    ......      
      
    private final void startProcessLocked(ProcessRecord app,      
        String hostingType, String hostingNameStr) {      
      
......      
      
try {      
    int uid = app.info.uid;      
    int[] gids = null;      
    try {      
        gids = mContext.getPackageManager().getPackageGids(      
            app.info.packageName);      
    } catch (PackageManager.NameNotFoundException e) {      
        ......      
    }      
          
    ......      
      
    int debugFlags = 0;      
          
    ......      
          
    int pid = Process.start("android.app.ActivityThread",      
        mSimpleProcessManagement ? app.processName : null, uid, uid,      
        gids, debugFlags, null);      
          
    ......      
      
} catch (RuntimeException e) {      
          
    ......      
      
}      
    }      
      
    ......      
      
}      
```
这里我们主要关注Process.start函数的第一个参数“android.app.ActivityThread”，它表示要在当前新建的线程中加载android.app.ActivityThread类，并且调用这个类的静态成员函数main作为应用程序的入口点。ActivityThread类定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

```java
public final class ActivityThread {    
    ......    
    
    public static final void main(String[] args) {    
......  
    
Looper.prepareMainLooper();    
   
......    
    
ActivityThread thread = new ActivityThread();    
thread.attach(false);    
    
......   
Looper.loop();    
    
......   
    
thread.detach();    
......    
    }    
    
    ......    
}    
```
在这个main函数里面，除了创建一个ActivityThread实例外，就是在进行消息循环了。

在进行消息循环之前，首先会通过Looper类的静态成员函数prepareMainLooper为当前线程准备一个消息循环对象。Looper类定义在frameworks/base/core/[Java](http://lib.csdn.net/base/java)/android/os/Looper.java文件中：

```java
public class Looper {  
    ......  

    // sThreadLocal.get() will return null unless you've called prepare().  
    private static final ThreadLocal sThreadLocal = new ThreadLocal();  

    ......  

    private static Looper mMainLooper = null;  

    ......  

    public static final void prepare() {  
if (sThreadLocal.get() != null) {  
    throw new RuntimeException("Only one Looper may be created per thread");  
}  
sThreadLocal.set(new Looper());  
    }  

    ......  

    public static final void prepareMainLooper() {  
prepare();  
setMainLooper(myLooper());  
......  
    }  

    private synchronized static void setMainLooper(Looper looper) {  
mMainLooper = looper;  
    }  

    public synchronized static final Looper getMainLooper() {  
return mMainLooper;  
    }  

    ......  

    public static final Looper myLooper() {  
return (Looper)sThreadLocal.get();  
    }  

    ......  
}  
```
Looper类的静态成员函数prepareMainLooper是专门应用程序的主线程调用的，应用程序的其它子线程都不应该调用这个函数来在本线程中创建消息循环对象，而应该调用prepare函数来在本线程中创建消息循环对象，下一节我们介绍一个线程类HandlerThread 时将会看到。

为什么要为应用程序的主线程专门准备一个创建消息循环对象的函数呢？这是为了让其它地方能够方便地通过Looper类的getMainLooper函数来获得应用程序主线程中的消息循环对象。获得应用程序主线程中的消息循环对象又有什么用呢？一般就是为了能够向应用程序主线程发送消息了。

在prepareMainLooper函数中，首先会调用prepare函数在本线程中创建一个消息循环对象，然后将这个消息循环对象放在线程局部变量sThreadLocal中：

```java
sThreadLocal.set(new Looper());  
```
接着再将这个消息循环对象通过调用setMainLooper函数来保存在Looper类的静态成员变量mMainLooper中：

```java
mMainLooper = looper;  
```
这样，其它地方才可以调用getMainLooper函数来获得应用程序主线程中的消息循环对象。

消息循环对象创建好之后，回到ActivityThread类的main函数中，接下来，就是要进入消息循环了：

```java
Looper.loop();   
```
Looper类具体是如何通过loop函数进入消息循环以及处理消息队列中的消息，可以参考前面一篇文章

Android应用程序消息处理机制（Looper、Handler）分析

，这里就不再分析了，我们只要知道ActivityThread类中的main函数执行了这一步之后，就为应用程序的主线程准备好消息循环就可以了。

## 应用程序子线程消息循环模型

在Java框架中，如果我们想在当前应用程序中创建一个子线程，一般就是通过自己实现一个类，这个类继承于Thread类，然后重载Thread类的run函数，把我们想要在这个子线程执行的任务都放在这个run函数里面实现。最后实例这个自定义的类，并且调用它的start函数，这样一个子线程就创建好了，并且会调用这个自定义类的run函数。但是当这个run函数执行完成后，子线程也就结束了，它没有消息循环的概念。

前面说过，有时候我们需要在应用程序中创建一些常驻的子线程来不定期地执行一些计算型任务，这时候就可以考虑使用Android系统提供的HandlerThread类了，它具有创建具有消息循环功能的子线程的作用。

HandlerThread类实现在frameworks/base/core/java/android/os/HandlerThread.java文件中，这里我们通过使用情景来有重点的分析它的实现。

在前面一篇文章[Android系统默认Home应用程序（Launcher）的启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6767736)中，我们分析了Launcher的启动过程，其中在Step 15（LauncherModel.startLoader）和Step 16（LoaderTask.run）中，Launcher会通过创建一个HandlerThread类来实现在一个子线程加载系统中已经安装的应用程序的任务：

```java
public class LauncherModel extends BroadcastReceiver {  
    ......  

    private LoaderTask mLoaderTask;  

    private static final HandlerThread sWorkerThread = new HandlerThread("launcher-loader");  
    static {  
sWorkerThread.start();  
    }  
    private static final Handler sWorker = new Handler(sWorkerThread.getLooper());  

    ......  

    public void startLoader(Context context, boolean isLaunching) {    
......    

synchronized (mLock) {    
    ......    

    // Don't bother to start the thread if we know it's not going to do anything    
    if (mCallbacks != null && mCallbacks.get() != null) {    
        ......  

        mLoaderTask = new LoaderTask(context, isLaunching);    
        sWorker.post(mLoaderTask);    
    }    
}    
    }    

    ......  

    private class LoaderTask implements Runnable {    
......    

public void run() {    
    ......    

    keep_running: {    
        ......    

        // second step    
        if (loadWorkspaceFirst) {    
            ......    
            loadAndBindAllApps();    
        } else {    
            ......    
        }    

        ......    
    }    

    ......    
}    

......    
    }   

    ......  
}  
```
在这个LauncherModel类中，首先创建了一个HandlerThread对象：

```java
private static final HandlerThread sWorkerThread = new HandlerThread("launcher-loader");  
```
接着调用它的start成员函数来启动一个子线程：

```java
static {  
    sWorkerThread.start();  
}  
```
接着还通过这个HandlerThread对象的getLooper函数来获得这个子线程中的消息循环对象，并且使用这个消息循环创建对象来创建一个Handler：

```java
private static final Handler sWorker = new Handler(sWorkerThread.getLooper()); 
```
有了这个Handler对象sWorker之后，我们就可以往这个子线程中发送消息，然后在处理这个消息的时候执行加载系统中已经安装的应用程序的任务了，在startLoader函数中：

```java
mLoaderTask = new LoaderTask(context, isLaunching);    
sWorker.post(mLoaderTask);    
```
这里的mLoaderTask是一个LoaderTask对象，它实现了Runnable接口，因此，可以把这个LoaderTask对象作为参数传给sWorker.post函数。在sWorker.post函数里面，会把这个LoaderTask对象封装成一个消息，并且放入这个子线程的消息队列中去。当这个子线程的消息循环处理这个消息的时候，就会调用这个LoaderTask对象的run函数，因此，我们就可以在LoaderTask对象的run函数中通过调用loadAndBindAllApps来执行加载系统中已经安装的应用程序的任务了。

了解了HanderThread类的使用方法之后，我们就可以重点地来分析它的实现了：

```java
public class HandlerThread extends Thread {  
    ......  
    private Looper mLooper;  

    public HandlerThread(String name) {  
super(name);  
......  
    }  

    ......  

    public void run() {  
......  
Looper.prepare();  
synchronized (this) {  
    mLooper = Looper.myLooper();  
    ......  
}  
......  
Looper.loop();  
......  
    }  

    public Looper getLooper() {  
......  
return mLooper;  
    }  

    ......  
}  
```
首先我们看到的是，HandlerThread类继承了Thread类，因此，通过它可以在应用程序中创建一个子线程，其次我们看到在它的run函数中，会进入一个消息循环中，因此，这个子线程可以常驻在应用程序中，直到它接收收到一个退出消息为止。

在run函数中，首先是调用Looper类的静态成员函数prepare来准备一个消息循环对象：

```java
Looper.prepare();  
```
然后通过Looper类的myLooper成员函数将这个子线程中的消息循环对象保存在HandlerThread类中的成员变量mLooper中：

```java
mLooper = Looper.myLooper();  
```
这样，其它地方就可以方便地通过它的getLooper函数来获得这个消息循环对象了，有了这个消息循环对象后，就可以往这个子线程的消息队列中发送消息，通知这个子线程执行特定的任务了。

最在这个run函数通过Looper类的loop函数进入消息循环中：

```java
Looper.loop();  
```
这样，一个具有消息循环的应用程序子线程就准备就绪了。

HandlerThread类的实现虽然非常简单，当然这得益于Java提供的Thread类和Android自己本身提供的Looper类，但是它的想法却非常周到，为应用程序开发人员提供了很大的方便。
需要与UI交互的应用程序子线程消息模型

前面说过，我们开发应用程序的时候，经常中需要创建一个子线程来在后台执行一个特定的计算任务，而在这个任务计算的过程中，需要不断地将计算进度或者计算结果展现在应用程序的界面中。典型的例子是从网上下载文件，为了不阻塞应用程序的主线程，我们开辟一个子线程来执行下载任务，子线程在下载的同时不断地将下载进度在应用程序界面上显示出来，这样做出来程序就非常友好。由于子线程不能直接操作应用程序的UI，因此，这时候，我们就可以通过往应用程序的主线程中发送消息来通知应用程序主线程更新界面上的下载进度。因为类似的这种情景在实际开发中经常碰到，Android系统为开发人员提供了一个异步任务类（AsyncTask）来实现上面所说的功能，即它会在一个子线程中执行计算任务，同时通过主线程的消息循环来获得更新应用程序界面的机会。

为了更好地分析AsyncTask的实现，我们先举一个例子来说明它的用法。在前面一篇文章[Android系统中的广播（Broadcast）机制简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6730748)中，我们开发了一个应用程序Broadcast，其中使用了AsyncTask来在一个线程在后台在执行计数任务，计数过程通过广播（Broadcast）来将中间结果在应用程序界面上显示出来。在这个例子中，使用广播来在应用程序主线程和子线程中传递数据不是最优的方法，当时只是为了分析Android系统的广播机制而有意为之的。在本节内容中，我们稍微这个例子作一个简单的修改，就可以通过消息的方式来将计数过程的中间结果在应用程序界面上显示出来。

为了区别[Android系统中的广播（Broadcast）机制简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6730748)一文中使用的应用程序Broadcast，我们将本节中使用的应用程序命名为Counter。首先在Android源代码工程中创建一个Android应用程序工程，名字就为Counter，放在packages/experimental目录下。关于如何获得Android源代码工程，请参考[在Ubuntu上下载、编译和安装Android最新源代码](http://blog.csdn.net/luoshengyang/article/details/6559955)一文；关于如何在Android源代码工程中创建应用程序工程，请参考[在Ubuntu上为Android系统内置Java应用程序测试Application Frameworks层的硬件服务](http://blog.csdn.net/luoshengyang/article/details/6580267)一文。这个应用程序工程定义了一个名为shy.luo.counter的package，这个例子的源代码主要就是实现在这个目录下的Counter.java文件中：

```java
package shy.luo.counter;  

import android.app.Activity;  
import android.content.ComponentName;  
import android.content.Context;  
import android.content.Intent;  
import android.content.IntentFilter;  
import android.os.Bundle;  
import android.os.AsyncTask;  
import android.util.Log;  
import android.view.View;  
import android.view.View.OnClickListener;  
import android.widget.Button;  
import android.widget.TextView;  

public class Counter extends Activity implements OnClickListener {  
    private final static String LOG_TAG = "shy.luo.counter.Counter";  

    private Button startButton = null;  
    private Button stopButton = null;  
    private TextView counterText = null;  

    private AsyncTask<Integer, Integer, Integer> task = null;  
    private boolean stop = false;  

    @Override  
    public void onCreate(Bundle savedInstanceState) {  
super.onCreate(savedInstanceState);  
setContentView(R.layout.main);  

startButton = (Button)findViewById(R.id.button_start);  
stopButton = (Button)findViewById(R.id.button_stop);  
counterText = (TextView)findViewById(R.id.textview_counter);  

startButton.setOnClickListener(this);  
stopButton.setOnClickListener(this);  

startButton.setEnabled(true);  
stopButton.setEnabled(false);  


Log.i(LOG_TAG, "Main Activity Created.");  
    }  


    @Override  
    public void onClick(View v) {  
if(v.equals(startButton)) {  
    if(task == null) {  
        task = new CounterTask();  
        task.execute(0);  

        startButton.setEnabled(false);  
        stopButton.setEnabled(true);  
    }  
} else if(v.equals(stopButton)) {  
    if(task != null) {  
        stop = true;  
        task = null;  

        startButton.setEnabled(true);  
        stopButton.setEnabled(false);  
    }  
}  
    }  

    class CounterTask extends AsyncTask<Integer, Integer, Integer> {  
@Override  
protected Integer doInBackground(Integer... vals) {  
    Integer initCounter = vals[0];  

    stop = false;  
    while(!stop) {  
        publishProgress(initCounter);  

        try {  
            Thread.sleep(1000);  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        }  

        initCounter++;  
    }  

    return initCounter;  
}  

@Override  
protected void onProgressUpdate(Integer... values) {  
    super.onProgressUpdate(values);  

    String text = values[0].toString();  
    counterText.setText(text);  
}  

@Override  
protected void onPostExecute(Integer val) {  
    String text = val.toString();  
    counterText.setText(text);  
}  
    };  
}  
```
这个计数器程序很简单，它在界面上有两个按钮Start和Stop。点击Start按钮时，便会创建一个CounterTask实例task，然后调用它的execute函数就可以在应用程序中启动一个子线程，并且通过调用这个CounterTask类的doInBackground函数来执行计数任务。在计数的过程中，会通过调用publishProgress函数来将中间结果传递到onProgressUpdate函数中去，在onProgressUpdate函数中，就可以把中间结果显示在应用程序界面了。点击Stop按钮时，便会通过设置变量stop为true，这样，CounterTask类的doInBackground函数便会退出循环，然后将结果返回到onPostExecute函数中去，在onPostExecute函数，会把最终计数结果显示在用程序界面中。

在这个例子中，我们需要注意的是：

A. CounterTask类继承于AsyncTask类，因此它也是一个异步任务类；

B. CounterTask类的doInBackground函数是在后台的子线程中运行的，这时候它不可以操作应用程序的界面；

C. CounterTask类的onProgressUpdate和onPostExecute两个函数是应用程序的主线程中执行，它们可以操作应用程序的界面。

关于C这一点的实现原理，我们在后面会分析到，这里我们先完整地介绍这个例子，以便读者可以参考做一下实验。

接下来我们再看看应用程序的配置文件AndroidManifest.xml：

```xml
<?xml version="1.0" encoding="utf-8"?>  
<manifest xmlns:android="http://schemas.android.com/apk/res/android"  
      package="shy.luo.counter"  
      android:versionCode="1"  
      android:versionName="1.0">  
    <application android:icon="@drawable/icon" android:label="@string/app_name">  
<activity android:name=".Counter"  
          android:label="@string/app_name">  
    <intent-filter>  
        <action android:name="android.intent.action.MAIN" />  
        <category android:name="android.intent.category.LAUNCHER" />  
    </intent-filter>  
</activity>  
    </application>  
</manifest>  
```
这个配置文件很简单，我们就不介绍了。

再来看应用程序的界面文件，它定义在res/layout/main.xml文件中：

```xml
<?xml version="1.0" encoding="utf-8"?>    
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"    
    android:orientation="vertical"    
    android:layout_width="fill_parent"    
    android:layout_height="fill_parent"     
    android:gravity="center">    
    <LinearLayout    
android:layout_width="fill_parent"    
android:layout_height="wrap_content"    
android:layout_marginBottom="10px"    
android:orientation="horizontal"     
android:gravity="center">    
<TextView      
android:layout_width="wrap_content"     
    android:layout_height="wrap_content"     
    android:layout_marginRight="4px"    
    android:gravity="center"    
    android:text="@string/counter">    
</TextView>    
<TextView      
    android:id="@+id/textview_counter"    
android:layout_width="wrap_content"     
    android:layout_height="wrap_content"     
    android:gravity="center"    
    android:text="0">    
</TextView>    
    </LinearLayout>    
    <LinearLayout    
android:layout_width="fill_parent"    
android:layout_height="wrap_content"    
android:orientation="horizontal"     
android:gravity="center">    
<Button     
    android:id="@+id/button_start"    
    android:layout_width="wrap_content"    
    android:layout_height="wrap_content"    
    android:gravity="center"    
    android:text="@string/start">    
</Button>    
<Button     
    android:id="@+id/button_stop"    
    android:layout_width="wrap_content"    
    android:layout_height="wrap_content"    
    android:gravity="center"    
    android:text="@string/stop" >    
</Button>    
     </LinearLayout>      
</LinearLayout>    
```
这个界面配置文件也很简单，等一下我们在模拟器把这个应用程序启动起来后，就可以看到它的截图了。

应用程序用到的字符串资源文件位于res/values/strings.xml文件中：

```xml
<?xml version="1.0" encoding="utf-8"?>    
<resources>    
    <string name="app_name">Counter</string>    
    <string name="counter">Counter: </string>    
    <string name="start">Start Counter</string>    
    <string name="stop">Stop Counter</string>    
</resources>   
```
最后，我们还要在工程目录下放置一个编译脚本文件Android.mk：

```
LOCAL_PATH:= $(call my-dir)          
include $(CLEAR_VARS)          
LOCAL_MODULE_TAGS := optional          
LOCAL_SRC_FILES := $(call all-subdir-java-files)          
LOCAL_PACKAGE_NAME := Counter          
include $(BUILD_PACKAGE)    
```
接下来就要编译了。有关如何单独编译Android源代码工程的模块，以及如何打包system.img，请参考如何单独编译Android源代码中的模块一文。

执行以下命令进行编译和打包：

```
USER-NAME@MACHINE-NAME:~/Android$ mmm packages/experimental/Counter            
USER-NAME@MACHINE-NAME:~/Android$ make snod  
```
这样，打包好的Android系统镜像文件system.img就包含我们前面创建的Counter应用程序了。

再接下来，就是运行模拟器来运行我们的例子了。关于如何在Android源代码工程中运行模拟器，请参考在Ubuntu上下载、编译和安装Android最新源代码一文。

执行以下命令启动模拟器：

```
USER-NAME@MACHINE-NAME:~/Android$ emulator   
```
最后我们就可以在Launcher中找到Counter应用程序图标，把它启动起来，点击Start按钮，就会看到应用程序界面上的计数器跑起来了：

![img](http://hi.csdn.net/attachment/201110/27/0_1319731144Ll99.gif)

这样，使用AsyncTask的例子就介绍完了，下面，我们就要根据上面对AsyncTask的使用情况来重点分析它的实现了。

AsyncTask类定义在frameworks/base/core/java/android/os/AsyncTask.java文件中：

```java
public abstract class AsyncTask<Params, Progress, Result> {  
    ......  

    private static final BlockingQueue<Runnable> sWorkQueue =  
    new LinkedBlockingQueue<Runnable>(10);  

    private static final ThreadFactory sThreadFactory = new ThreadFactory() {  
private final AtomicInteger mCount = new AtomicInteger(1);  

public Thread newThread(Runnable r) {  
    return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());  
}  
    };  

    ......  

    private static final ThreadPoolExecutor sExecutor = new ThreadPoolExecutor(CORE_POOL_SIZE,  
MAXIMUM_POOL_SIZE, KEEP_ALIVE, TimeUnit.SECONDS, sWorkQueue, sThreadFactory);  

    private static final int MESSAGE_POST_RESULT = 0x1;  
    private static final int MESSAGE_POST_PROGRESS = 0x2;  
    private static final int MESSAGE_POST_CANCEL = 0x3;  

    private static final InternalHandler sHandler = new InternalHandler();  

    private final WorkerRunnable<Params, Result> mWorker;  
    private final FutureTask<Result> mFuture;  

    ......  

    public AsyncTask() {  
mWorker = new WorkerRunnable<Params, Result>() {  
    public Result call() throws Exception {  
        ......  
        return doInBackground(mParams);  
    }  
};  

mFuture = new FutureTask<Result>(mWorker) {  
    @Override  
    protected void done() {  
        Message message;  
        Result result = null;  

        try {  
            result = get();  
        } catch (InterruptedException e) {  
            android.util.Log.w(LOG_TAG, e);  
        } catch (ExecutionException e) {  
            throw new RuntimeException("An error occured while executing doInBackground()",  
                e.getCause());  
        } catch (CancellationException e) {  
            message = sHandler.obtainMessage(MESSAGE_POST_CANCEL,  
                new AsyncTaskResult<Result>(AsyncTask.this, (Result[]) null));  
            message.sendToTarget();  
            return;  
        } catch (Throwable t) {  
            throw new RuntimeException("An error occured while executing "  
                + "doInBackground()", t);  
        }  

        message = sHandler.obtainMessage(MESSAGE_POST_RESULT,  
            new AsyncTaskResult<Result>(AsyncTask.this, result));  
        message.sendToTarget();  
    }  
};  
    }  

    ......  

    public final Result get() throws InterruptedException, ExecutionException {  
return mFuture.get();  
    }  

    ......  

    public final AsyncTask<Params, Progress, Result> execute(Params... params) {  
......  

mWorker.mParams = params;  
sExecutor.execute(mFuture);  

return this;  
    }  

    ......  

    protected final void publishProgress(Progress... values) {  
sHandler.obtainMessage(MESSAGE_POST_PROGRESS,  
    new AsyncTaskResult<Progress>(this, values)).sendToTarget();  
    }  

private void finish(Result result) {  
        ......  
        onPostExecute(result);  
        ......  
}  

    ......  

    private static class InternalHandler extends Handler {  
@SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})  
@Override  
public void handleMessage(Message msg) {  
    AsyncTaskResult result = (AsyncTaskResult) msg.obj;  
    switch (msg.what) {  
        case MESSAGE_POST_RESULT:  
         // There is only one result  
         result.mTask.finish(result.mData[0]);  
         break;  
        case MESSAGE_POST_PROGRESS:  
         result.mTask.onProgressUpdate(result.mData);  
         break;  
        case MESSAGE_POST_CANCEL:  
         result.mTask.onCancelled();  
         break;  
    }  
}  
    }  

    private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {  
Params[] mParams;  
    }  

    private static class AsyncTaskResult<Data> {  
final AsyncTask mTask;  
final Data[] mData;  

AsyncTaskResult(AsyncTask task, Data... data) {  
    mTask = task;  
    mData = data;  
}  
    }  
}  
```
从AsyncTask的实现可以看出，当我们第一次创建一个AsyncTask对象时，首先会执行下面静态初始化代码创建一个线程池sExecutor：

```java
private static final BlockingQueue<Runnable> sWorkQueue =  
    new LinkedBlockingQueue<Runnable>(10);  

private static final ThreadFactory sThreadFactory = new ThreadFactory() {  
    private final AtomicInteger mCount = new AtomicInteger(1);  

    public Thread newThread(Runnable r) {  
return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());  
    }  
};  

......  

private static final ThreadPoolExecutor sExecutor = new ThreadPoolExecutor(CORE_POOL_SIZE,  
    MAXIMUM_POOL_SIZE, KEEP_ALIVE, TimeUnit.SECONDS, sWorkQueue, sThreadFactory);  
```
这里的ThreadPoolExecutor是Java提供的多线程机制之一，这里用的构造函数原型为：

```java
ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit,   
    BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory)  
```
各个参数的意义如下：

| 方法声明            | 功能说明                                     |
| :-------------- | :--------------------------------------- |
| corePoolSize    | 线程池的核心线程数量                               |
| maximumPoolSize | 线程池的最大线程数量                               |
| keepAliveTime   | 若线程池的线程数数量大于核心线程数量，那么空闲时间超过keepAliveTime的线程将被回收 |
| unit            | 参数keepAliveTime使用的时间单位                   |
| workerQueue     | 工作任务队列                                   |
| threadFactory   | 用来创建线程池中的线程                              |
简单来说，ThreadPoolExecutor的运行机制是这样的：每一个工作任务用一个Runnable对象来表示，当我们要把一个工作任务交给这个线程池来执行的时候，就通过调用ThreadPoolExecutor的execute函数来把这个工作任务加入到线程池中去。此时，如果线程池中的线程数量小于corePoolSize，那么就会调用threadFactory接口来创建一个新的线程并且加入到线程池中去，再执行这个工作任务；如果线程池中的线程数量等于corePoolSize，但是工作任务队列workerQueue未满，则把这个工作任务加入到工作任务队列中去等待执行；如果线程池中的线程数量大于corePoolSize，但是小于maximumPoolSize，并且工作任务队列workerQueue已经满了，那么就会调用threadFactory接口来创建一个新的线程并且加入到线程池中去，再执行这个工作任务；如果线程池中的线程量已经等于maximumPoolSize了，并且工作任务队列workerQueue也已经满了，这个工作任务就被拒绝执行了。

创建好了线程池后，再创建一个消息处理器：

```java
private static final InternalHandler sHandler = new InternalHandler();  
```
注意，这行代码是在应用程序的主线程中执行的，因此，这个消息处理器sHandler内部引用的消息循环对象looper是应用程序主线程的消息循环对象，消息处理器的实现机制具体可以参考前面一篇文章

## Android应用程序消息处理机制（Looper、Handler）分析

AsyncTask类的静态初始化代码执行完成之后，才开始创建AsyncTask对象，即执行AsyncTask类的构造函数：

```java
public AsyncTask() {  
    mWorker = new WorkerRunnable<Params, Result>() {  
public Result call() throws Exception {  
    ......  
    return doInBackground(mParams);  
}  
    };  

    mFuture = new FutureTask<Result>(mWorker) {  
@Override  
protected void done() {  
    Message message;  
    Result result = null;  

    try {  
        result = get();  
    } catch (InterruptedException e) {  
        android.util.Log.w(LOG_TAG, e);  
    } catch (ExecutionException e) {  
        throw new RuntimeException("An error occured while executing doInBackground()",  
            e.getCause());  
    } catch (CancellationException e) {  
        message = sHandler.obtainMessage(MESSAGE_POST_CANCEL,  
            new AsyncTaskResult<Result>(AsyncTask.this, (Result[]) null));  
        message.sendToTarget();  
        return;  
    } catch (Throwable t) {  
        throw new RuntimeException("An error occured while executing "  
            + "doInBackground()", t);  
    }  

    message = sHandler.obtainMessage(MESSAGE_POST_RESULT,  
        new AsyncTaskResult<Result>(AsyncTask.this, result));  
    message.sendToTarget();  
}  
    };  
}  
```
在AsyncTask类的构造函数里面，主要是创建了两个对象，分别是一个WorkerRunnable对象mWorker和一个FutureTask对象mFuture。

WorkerRunnable类实现了Callable接口，此外，它的内部成员变量mParams用于保存从AsyncTask对象的execute函数传进来的参数列表：

```java
private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {  
    Params[] mParams;  
}  
```
FutureTask类也实现了Runnable接口，所以它可以作为一个工作任务通过调用AsyncTask类的execute函数添加到sExecuto线程池中去：

```java
public final AsyncTask<Params, Progress, Result> execute(Params... params) {  
    ......  

    mWorker.mParams = params;  
    sExecutor.execute(mFuture);  

    return this;  
}  
```
这里的FutureTask对象mFuture是用来封装前面的WorkerRunnable对象mWorker。当mFuture加入到线程池中执行时，它调用的是mWorker对象的call函数：

```java
mWorker = new WorkerRunnable<Params, Result>() {  
    public Result call() throws Exception {  
   ......  
   return doInBackground(mParams);  
}  
};  
```
在call函数里面，会调用AsyncTask类的doInBackground函数来执行真正的任务，这个函数是要由AsyncTask的子类来实现的，注意，这个函数是在应用程序的子线程中执行的，它不可以操作应用程序的界面。

我们可以通过mFuture对象来操作当前执行的任务，例如查询当前任务的状态，它是正在执行中，还是完成了，还是被取消了，如果是完成了，还可以通过它获得任务的执行结果，如果还没有完成，可以取消任务的执行。

当工作任务mWorker执行完成的时候，mFuture对象中的done函数就会被被调用，根据任务的完成状况，执行相应的操作，例如，如果是因为异常而完成时，就会抛异常，如果是正常完成，就会把任务执行结果封装成一个AsyncTaskResult对象：

```java
private static class AsyncTaskResult<Data> {  
    final AsyncTask mTask;  
    final Data[] mData;  

    AsyncTaskResult(AsyncTask task, Data... data) {  
mTask = task;  
mData = data;  
    }  
}  
```
其中，成员变量mData保存的是任务执行结果，而成员变量mTask指向前面我们创建的AsyncTask对象。

最后把这个AsyncTaskResult对象封装成一个消息，并且通过消息处理器sHandler加入到应用程序主线程的消息队列中：

```java
message = sHandler.obtainMessage(MESSAGE_POST_RESULT,  
    new AsyncTaskResult<Result>(AsyncTask.this, result));  
message.sendToTarget();  
```
这个消息最终就会在InternalHandler类的handleMessage函数中处理了：

```java
private static class InternalHandler extends Handler {  
    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})  
    @Override  
    public void handleMessage(Message msg) {  
AsyncTaskResult result = (AsyncTaskResult) msg.obj;  
switch (msg.what) {  
case MESSAGE_POST_RESULT:  
    // There is only one result  
    result.mTask.finish(result.mData[0]);  
    break;  
......  
}  
    }  
}  
```
在这个函数里面，最终会调用前面创建的这个AsyncTask对象的finish函数来进一步处理：

```java
private void finish(Result result) {  
       ......  
       onPostExecute(result);  
       ......  
}  
```
这个函数调用AsyncTask类的onPostExecute函数来进一步处理，AsyncTask类的onPostExecute函数一般是要由其子类来重载的，注意，这个函数是在应用程序的主线程中执行的，因此，它可以操作应用程序的界面。

在任务执行的过程当中，即执行doInBackground函数时候，可能通过调用publishProgress函数来将中间结果封装成一个消息发送到应用程序主线程中的消息队列中去：

```java
protected final void publishProgress(Progress... values) {  
    sHandler.obtainMessage(MESSAGE_POST_PROGRESS,  
new AsyncTaskResult<Progress>(this, values)).sendToTarget();  
}  
```
这个消息最终也是由InternalHandler类的handleMessage函数来处理的：

```java
private static class InternalHandler extends Handler {  
    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})  
    @Override  
    public void handleMessage(Message msg) {  
AsyncTaskResult result = (AsyncTaskResult) msg.obj;  
switch (msg.what) {  
......  
case MESSAGE_POST_PROGRESS:  
         result.mTask.onProgressUpdate(result.mData);  
         break;  
......  
}  
    }  
}  
```
这里它调用前面创建的AsyncTask对象的onPorgressUpdate函数来进一步处理，这个函数一般是由AsyncTask的子类来实现的，注意，这个函数是在应用程序的主线程中执行的，因此，它和前面的onPostExecute函数一样，可以操作应用程序的界面。

这样，AsyncTask类的主要实现就介绍完了，结合前面开发的应用程序Counter来分析，会更好地理解它的实现原理。

至此，Android应用程序线程消息循环模型就分析完成了，理解它有利于我们在开发Android应用程序时，能够充分利用多线程的并发性来提高应用程序的性能以及获得良好的用户体验。