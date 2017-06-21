我们知道，[Android](http://lib.csdn.net/base/android)应用程序是通过消息来驱动的，即在应用程序的主线程（UI线程）中有一个消息循环，负责处理消息队列中的消息。我们也知道，[android](http://lib.csdn.net/base/android)应用程序是支持多线程的，即可以创建子线程来执行一些计算型的任务，那么，这些子线程能不能像应用程序的主线程一样具有消息循环呢？这些子线程又能不能往应用程序的主线程中发送消息呢？本文将分析Android应用程序线程消息处理模型，为读者解答这两个问题。

《Android系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

​        在开发Android应用程序中，有时候我们需要在应用程序中创建一些常驻的子线程来不定期地执行一些不需要与应用程序界面交互的计算型的任务。如果这些子线程具有消息循环，那么它们就能够常驻在应用程序中不定期的执行一些计算型任务了：当我们需要用这些子线程来执行任务时，就往这个子线程的消息队列中发送一个消息，然后就可以在子线程的消息循环中执行我们的计算型任务了。我们在前面一篇文章[Android系统默认Home应用程序（Launcher）的启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6767736)中，介绍Launcher的启动过程时，在Step 15（LauncherModel.startLoader）中，Launcher就是通过往一个子线程的消息队列中发送一个消息（sWorker.post(mLoaderTask)），然后子线程就会在它的消息循环中处理这个消息的时候执行从PackageManagerService中获取系统中已安装应用程序的信息列表的任务，即调用Step 16中的LoaderTask.run函数。

​        在开发Android应用程序中，有时候我们又需要在应用程序中创建一些子线程来执行一些需要与应用程序界面进交互的计算型任务。典型的应用场景是当我们要从网上下载文件时，为了不使主线程被阻塞，我们通常创建一个子线程来负责下载任务，同时，在下载的过程，将下载进度以百分比的形式在应用程序的界面上显示出来，这样就既不会阻塞主线程的运行，又能获得良好的用户体验。但是，我们知道，Android应用程序的子线程是不可以操作主线程的UI的，那么，这个负责下载任务的子线程应该如何在应用程序界面上显示下载的进度呢？如果我们能够在子线程中往主线程的消息队列中发送消息，那么问题就迎刃而解了，因为发往主线程消息队列的消息最终是由主线程来处理的，在处理这个消息的时候，我们就可以在应用程序界面上显示下载进度了。

​        上面提到的这两种情况，Android系统都为我们提供了完善的解决方案，前者可以通过使用HandlerThread类来实现，而后者可以使用AsyncTask类来实现，本文就详细这两个类是如何实现的。不过，为了更好地理解HandlerThread类和AsyncTask类的实现，我们先来看看应用程序的主线程的消息循环模型是如何实现的。

​        1. 应用程序主线程消息循环模型

​        在前面一篇文章[Android应用程序进程启动过程的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6747696)一文中，我们已经分析应用程序进程（主线程）的启动过程了，这里主要是针对它的消息循环模型作一个总结。当运行在Android应用程序框架层中的ActivityManagerService决定要为当前启动的应用程序创建一个主线程的时候，它会在ActivityManagerService中的startProcessLocked成员函数调用Process类的静态成员函数start为当前应用程序创建一个主线程：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. public final class ActivityManagerService extends ActivityManagerNative      
2. ​        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {      
3. ​      
4. ​    ......      
5. ​      
6. ​    private final void startProcessLocked(ProcessRecord app,      
7. ​                String hostingType, String hostingNameStr) {      
8. ​      
9. ​        ......      
10. ​      
11. ​        try {      
12. ​            int uid = app.info.uid;      
13. ​            int[] gids = null;      
14. ​            try {      
15. ​                gids = mContext.getPackageManager().getPackageGids(      
16. ​                    app.info.packageName);      
17. ​            } catch (PackageManager.NameNotFoundException e) {      
18. ​                ......      
19. ​            }      
20. ​                  
21. ​            ......      
22. ​      
23. ​            int debugFlags = 0;      
24. ​                  
25. ​            ......      
26. ​                  
27. ​            int pid = Process.start("android.app.ActivityThread",      
28. ​                mSimpleProcessManagement ? app.processName : null, uid, uid,      
29. ​                gids, debugFlags, null);      
30. ​                  
31. ​            ......      
32. ​      
33. ​        } catch (RuntimeException e) {      
34. ​                  
35. ​            ......      
36. ​      
37. ​        }      
38. ​    }      
39. ​      
40. ​    ......      
41. ​      
42. }      

​        这里我们主要关注Process.start函数的第一个参数“android.app.ActivityThread”，它表示要在当前新建的线程中加载android.app.ActivityThread类，并且调用这个类的静态成员函数main作为应用程序的入口点。ActivityThread类定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. public final class ActivityThread {    
2. ​    ......    
3. ​    
4. ​    public static final void main(String[] args) {    
5. ​        ......  
6. ​    
7. ​        Looper.prepareMainLooper();    
8. ​           
9. ​        ......    
10. ​    
11. ​        ActivityThread thread = new ActivityThread();    
12. ​        thread.attach(false);    
13. ​    
14. ​        ......   
15. ​        Looper.loop();    
16. ​    
17. ​        ......   
18. ​    
19. ​        thread.detach();    
20. ​        ......    
21. ​    }    
22. ​    
23. ​    ......    
24. }    

​        在这个main函数里面，除了创建一个ActivityThread实例外，就是在进行消息循环了。

​        在进行消息循环之前，首先会通过Looper类的静态成员函数prepareMainLooper为当前线程准备一个消息循环对象。Looper类定义在frameworks/base/core/[Java](http://lib.csdn.net/base/java)/android/os/Looper.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. public class Looper {  
2. ​    ......  
3.   
4. ​    // sThreadLocal.get() will return null unless you've called prepare().  
5. ​    private static final ThreadLocal sThreadLocal = new ThreadLocal();  
6.   
7. ​    ......  
8.   
9. ​    private static Looper mMainLooper = null;  
10.   
11. ​    ......  
12.   
13. ​    public static final void prepare() {  
14. ​        if (sThreadLocal.get() != null) {  
15. ​            throw new RuntimeException("Only one Looper may be created per thread");  
16. ​        }  
17. ​        sThreadLocal.set(new Looper());  
18. ​    }  
19.   
20. ​    ......  
21.   
22. ​    public static final void prepareMainLooper() {  
23. ​        prepare();  
24. ​        setMainLooper(myLooper());  
25. ​        ......  
26. ​    }  
27.   
28. ​    private synchronized static void setMainLooper(Looper looper) {  
29. ​        mMainLooper = looper;  
30. ​    }  
31.   
32. ​    public synchronized static final Looper getMainLooper() {  
33. ​        return mMainLooper;  
34. ​    }  
35.   
36. ​    ......  
37.   
38. ​    public static final Looper myLooper() {  
39. ​        return (Looper)sThreadLocal.get();  
40. ​    }  
41.   
42. ​    ......  
43. }  

​        Looper类的静态成员函数prepareMainLooper是专门应用程序的主线程调用的，应用程序的其它子线程都不应该调用这个函数来在本线程中创建消息循环对象，而应该调用prepare函数来在本线程中创建消息循环对象，下一节我们介绍一个线程类HandlerThread 时将会看到。

​        为什么要为应用程序的主线程专门准备一个创建消息循环对象的函数呢？这是为了让其它地方能够方便地通过Looper类的getMainLooper函数来获得应用程序主线程中的消息循环对象。获得应用程序主线程中的消息循环对象又有什么用呢？一般就是为了能够向应用程序主线程发送消息了。

​        在prepareMainLooper函数中，首先会调用prepare函数在本线程中创建一个消息循环对象，然后将这个消息循环对象放在线程局部变量sThreadLocal中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. sThreadLocal.set(new Looper());  

​        接着再将这个消息循环对象通过调用setMainLooper函数来保存在Looper类的静态成员变量mMainLooper中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. mMainLooper = looper;  

​       这样，其它地方才可以调用getMainLooper函数来获得应用程序主线程中的消息循环对象。

​       消息循环对象创建好之后，回到ActivityThread类的main函数中，接下来，就是要进入消息循环了：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. Looper.loop();   

​        Looper类具体是如何通过loop函数进入消息循环以及处理消息队列中的消息，可以参考前面一篇文章

Android应用程序消息处理机制（Looper、Handler）分析

，这里就不再分析了，我们只要知道ActivityThread类中的main函数执行了这一步之后，就为应用程序的主线程准备好消息循环就可以了。

​        2. 应用程序子线程消息循环模型

​        在Java框架中，如果我们想在当前应用程序中创建一个子线程，一般就是通过自己实现一个类，这个类继承于Thread类，然后重载Thread类的run函数，把我们想要在这个子线程执行的任务都放在这个run函数里面实现。最后实例这个自定义的类，并且调用它的start函数，这样一个子线程就创建好了，并且会调用这个自定义类的run函数。但是当这个run函数执行完成后，子线程也就结束了，它没有消息循环的概念。

​        前面说过，有时候我们需要在应用程序中创建一些常驻的子线程来不定期地执行一些计算型任务，这时候就可以考虑使用Android系统提供的HandlerThread类了，它具有创建具有消息循环功能的子线程的作用。

​        HandlerThread类实现在frameworks/base/core/java/android/os/HandlerThread.java文件中，这里我们通过使用情景来有重点的分析它的实现。

​        在前面一篇文章[Android系统默认Home应用程序（Launcher）的启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6767736)中，我们分析了Launcher的启动过程，其中在Step 15（LauncherModel.startLoader）和Step 16（LoaderTask.run）中，Launcher会通过创建一个HandlerThread类来实现在一个子线程加载系统中已经安装的应用程序的任务：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. public class LauncherModel extends BroadcastReceiver {  
2. ​    ......  
3.   
4. ​    private LoaderTask mLoaderTask;  
5.   
6. ​    private static final HandlerThread sWorkerThread = new HandlerThread("launcher-loader");  
7. ​    static {  
8. ​        sWorkerThread.start();  
9. ​    }  
10. ​    private static final Handler sWorker = new Handler(sWorkerThread.getLooper());  
11.   
12. ​    ......  
13.   
14. ​    public void startLoader(Context context, boolean isLaunching) {    
15. ​        ......    
16.   
17. ​        synchronized (mLock) {    
18. ​            ......    
19.   
20. ​            // Don't bother to start the thread if we know it's not going to do anything    
21. ​            if (mCallbacks != null && mCallbacks.get() != null) {    
22. ​                ......  
23.   
24. ​                mLoaderTask = new LoaderTask(context, isLaunching);    
25. ​                sWorker.post(mLoaderTask);    
26. ​            }    
27. ​        }    
28. ​    }    
29.   
30. ​    ......  
31.   
32. ​    private class LoaderTask implements Runnable {    
33. ​        ......    
34.   
35. ​        public void run() {    
36. ​            ......    
37.   
38. ​            keep_running: {    
39. ​                ......    
40.   
41. ​                // second step    
42. ​                if (loadWorkspaceFirst) {    
43. ​                    ......    
44. ​                    loadAndBindAllApps();    
45. ​                } else {    
46. ​                    ......    
47. ​                }    
48.   
49. ​                ......    
50. ​            }    
51.   
52. ​            ......    
53. ​        }    
54.   
55. ​        ......    
56. ​    }   
57.   
58. ​    ......  
59. }  

​        在这个LauncherModel类中，首先创建了一个HandlerThread对象：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. private static final HandlerThread sWorkerThread = new HandlerThread("launcher-loader");  

​        接着调用它的start成员函数来启动一个子线程：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. static {  
2. ​    sWorkerThread.start();  
3. }  

​        接着还通过这个HandlerThread对象的getLooper函数来获得这个子线程中的消息循环对象，并且使用这个消息循环创建对象来创建一个Handler：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. private static final Handler sWorker = new Handler(sWorkerThread.getLooper());  

​        有了这个Handler对象sWorker之后，我们就可以往这个子线程中发送消息，然后在处理这个消息的时候执行加载系统中已经安装的应用程序的任务了，在startLoader函数中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. mLoaderTask = new LoaderTask(context, isLaunching);    
2. sWorker.post(mLoaderTask);    

​        这里的mLoaderTask是一个LoaderTask对象，它实现了Runnable接口，因此，可以把这个LoaderTask对象作为参数传给sWorker.post函数。在sWorker.post函数里面，会把这个LoaderTask对象封装成一个消息，并且放入这个子线程的消息队列中去。当这个子线程的消息循环处理这个消息的时候，就会调用这个LoaderTask对象的run函数，因此，我们就可以在LoaderTask对象的run函数中通过调用loadAndBindAllApps来执行加载系统中已经安装的应用程序的任务了。

​        了解了HanderThread类的使用方法之后，我们就可以重点地来分析它的实现了：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. public class HandlerThread extends Thread {  
2. ​    ......  
3. ​    private Looper mLooper;  
4.   
5. ​    public HandlerThread(String name) {  
6. ​        super(name);  
7. ​        ......  
8. ​    }  
9.   
10. ​    ......  
11.   
12. ​    public void run() {  
13. ​        ......  
14. ​        Looper.prepare();  
15. ​        synchronized (this) {  
16. ​            mLooper = Looper.myLooper();  
17. ​            ......  
18. ​        }  
19. ​        ......  
20. ​        Looper.loop();  
21. ​        ......  
22. ​    }  
23.   
24. ​    public Looper getLooper() {  
25. ​        ......  
26. ​        return mLooper;  
27. ​    }  
28.   
29. ​    ......  
30. }  

​        首先我们看到的是，HandlerThread类继承了Thread类，因此，通过它可以在应用程序中创建一个子线程，其次我们看到在它的run函数中，会进入一个消息循环中，因此，这个子线程可以常驻在应用程序中，直到它接收收到一个退出消息为止。

​        在run函数中，首先是调用Looper类的静态成员函数prepare来准备一个消息循环对象：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. Looper.prepare();  

​        然后通过Looper类的myLooper成员函数将这个子线程中的消息循环对象保存在HandlerThread类中的成员变量mLooper中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. mLooper = Looper.myLooper();  

​        这样，其它地方就可以方便地通过它的getLooper函数来获得这个消息循环对象了，有了这个消息循环对象后，就可以往这个子线程的消息队列中发送消息，通知这个子线程执行特定的任务了。

​        最在这个run函数通过Looper类的loop函数进入消息循环中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. Looper.loop();  

​        这样，一个具有消息循环的应用程序子线程就准备就绪了。

​        HandlerThread类的实现虽然非常简单，当然这得益于Java提供的Thread类和Android自己本身提供的Looper类，但是它的想法却非常周到，为应用程序开发人员提供了很大的方便。
​        3. 需要与UI交互的应用程序子线程消息模型

​        前面说过，我们开发应用程序的时候，经常中需要创建一个子线程来在后台执行一个特定的计算任务，而在这个任务计算的过程中，需要不断地将计算进度或者计算结果展现在应用程序的界面中。典型的例子是从网上下载文件，为了不阻塞应用程序的主线程，我们开辟一个子线程来执行下载任务，子线程在下载的同时不断地将下载进度在应用程序界面上显示出来，这样做出来程序就非常友好。由于子线程不能直接操作应用程序的UI，因此，这时候，我们就可以通过往应用程序的主线程中发送消息来通知应用程序主线程更新界面上的下载进度。因为类似的这种情景在实际开发中经常碰到，Android系统为开发人员提供了一个异步任务类（AsyncTask）来实现上面所说的功能，即它会在一个子线程中执行计算任务，同时通过主线程的消息循环来获得更新应用程序界面的机会。

​        为了更好地分析AsyncTask的实现，我们先举一个例子来说明它的用法。在前面一篇文章[Android系统中的广播（Broadcast）机制简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6730748)中，我们开发了一个应用程序Broadcast，其中使用了AsyncTask来在一个线程在后台在执行计数任务，计数过程通过广播（Broadcast）来将中间结果在应用程序界面上显示出来。在这个例子中，使用广播来在应用程序主线程和子线程中传递数据不是最优的方法，当时只是为了分析Android系统的广播机制而有意为之的。在本节内容中，我们稍微这个例子作一个简单的修改，就可以通过消息的方式来将计数过程的中间结果在应用程序界面上显示出来。

​        为了区别[Android系统中的广播（Broadcast）机制简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6730748)一文中使用的应用程序Broadcast，我们将本节中使用的应用程序命名为Counter。首先在Android源代码工程中创建一个Android应用程序工程，名字就为Counter，放在packages/experimental目录下。关于如何获得Android源代码工程，请参考[在Ubuntu上下载、编译和安装Android最新源代码](http://blog.csdn.net/luoshengyang/article/details/6559955)一文；关于如何在Android源代码工程中创建应用程序工程，请参考[在Ubuntu上为Android系统内置Java应用程序测试Application Frameworks层的硬件服务](http://blog.csdn.net/luoshengyang/article/details/6580267)一文。这个应用程序工程定义了一个名为shy.luo.counter的package，这个例子的源代码主要就是实现在这个目录下的Counter.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. package shy.luo.counter;  
2.   
3. import android.app.Activity;  
4. import android.content.ComponentName;  
5. import android.content.Context;  
6. import android.content.Intent;  
7. import android.content.IntentFilter;  
8. import android.os.Bundle;  
9. import android.os.AsyncTask;  
10. import android.util.Log;  
11. import android.view.View;  
12. import android.view.View.OnClickListener;  
13. import android.widget.Button;  
14. import android.widget.TextView;  
15.   
16. public class Counter extends Activity implements OnClickListener {  
17. ​    private final static String LOG_TAG = "shy.luo.counter.Counter";  
18.   
19. ​    private Button startButton = null;  
20. ​    private Button stopButton = null;  
21. ​    private TextView counterText = null;  
22.   
23. ​    private AsyncTask<Integer, Integer, Integer> task = null;  
24. ​    private boolean stop = false;  
25.   
26. ​    @Override  
27. ​    public void onCreate(Bundle savedInstanceState) {  
28. ​        super.onCreate(savedInstanceState);  
29. ​        setContentView(R.layout.main);  
30.   
31. ​        startButton = (Button)findViewById(R.id.button_start);  
32. ​        stopButton = (Button)findViewById(R.id.button_stop);  
33. ​        counterText = (TextView)findViewById(R.id.textview_counter);  
34.   
35. ​        startButton.setOnClickListener(this);  
36. ​        stopButton.setOnClickListener(this);  
37.   
38. ​        startButton.setEnabled(true);  
39. ​        stopButton.setEnabled(false);  
40.   
41.   
42. ​        Log.i(LOG_TAG, "Main Activity Created.");  
43. ​    }  
44.   
45.   
46. ​    @Override  
47. ​    public void onClick(View v) {  
48. ​        if(v.equals(startButton)) {  
49. ​            if(task == null) {  
50. ​                task = new CounterTask();  
51. ​                task.execute(0);  
52.   
53. ​                startButton.setEnabled(false);  
54. ​                stopButton.setEnabled(true);  
55. ​            }  
56. ​        } else if(v.equals(stopButton)) {  
57. ​            if(task != null) {  
58. ​                stop = true;  
59. ​                task = null;  
60.   
61. ​                startButton.setEnabled(true);  
62. ​                stopButton.setEnabled(false);  
63. ​            }  
64. ​        }  
65. ​    }  
66.   
67. ​    class CounterTask extends AsyncTask<Integer, Integer, Integer> {  
68. ​        @Override  
69. ​        protected Integer doInBackground(Integer... vals) {  
70. ​            Integer initCounter = vals[0];  
71.   
72. ​            stop = false;  
73. ​            while(!stop) {  
74. ​                publishProgress(initCounter);  
75.   
76. ​                try {  
77. ​                    Thread.sleep(1000);  
78. ​                } catch (InterruptedException e) {  
79. ​                    e.printStackTrace();  
80. ​                }  
81.   
82. ​                initCounter++;  
83. ​            }  
84.   
85. ​            return initCounter;  
86. ​        }  
87.   
88. ​        @Override  
89. ​        protected void onProgressUpdate(Integer... values) {  
90. ​            super.onProgressUpdate(values);  
91.   
92. ​            String text = values[0].toString();  
93. ​            counterText.setText(text);  
94. ​        }  
95.   
96. ​        @Override  
97. ​        protected void onPostExecute(Integer val) {  
98. ​            String text = val.toString();  
99. ​            counterText.setText(text);  
100. ​        }  
101. ​    };  
102. }  

​        这个计数器程序很简单，它在界面上有两个按钮Start和Stop。点击Start按钮时，便会创建一个CounterTask实例task，然后调用它的execute函数就可以在应用程序中启动一个子线程，并且通过调用这个CounterTask类的doInBackground函数来执行计数任务。在计数的过程中，会通过调用publishProgress函数来将中间结果传递到onProgressUpdate函数中去，在onProgressUpdate函数中，就可以把中间结果显示在应用程序界面了。点击Stop按钮时，便会通过设置变量stop为true，这样，CounterTask类的doInBackground函数便会退出循环，然后将结果返回到onPostExecute函数中去，在onPostExecute函数，会把最终计数结果显示在用程序界面中。

​       在这个例子中，我们需要注意的是：

​       A. CounterTask类继承于AsyncTask类，因此它也是一个异步任务类；

​       B. CounterTask类的doInBackground函数是在后台的子线程中运行的，这时候它不可以操作应用程序的界面；

​       C. CounterTask类的onProgressUpdate和onPostExecute两个函数是应用程序的主线程中执行，它们可以操作应用程序的界面。

​       关于C这一点的实现原理，我们在后面会分析到，这里我们先完整地介绍这个例子，以便读者可以参考做一下实验。

​       接下来我们再看看应用程序的配置文件AndroidManifest.xml：

**[html]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. <?xml version="1.0" encoding="utf-8"?>  
2. <manifest xmlns:android="http://schemas.android.com/apk/res/android"  
3. ​      package="shy.luo.counter"  
4. ​      android:versionCode="1"  
5. ​      android:versionName="1.0">  
6. ​    <application android:icon="@drawable/icon" android:label="@string/app_name">  
7. ​        <activity android:name=".Counter"  
8. ​                  android:label="@string/app_name">  
9. ​            <intent-filter>  
10. ​                <action android:name="android.intent.action.MAIN" />  
11. ​                <category android:name="android.intent.category.LAUNCHER" />  
12. ​            </intent-filter>  
13. ​        </activity>  
14. ​    </application>  
15. </manifest>  

​       这个配置文件很简单，我们就不介绍了。

​       再来看应用程序的界面文件，它定义在res/layout/main.xml文件中：

**[html]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. <?xml version="1.0" encoding="utf-8"?>    
2. <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"    
3. ​    android:orientation="vertical"    
4. ​    android:layout_width="fill_parent"    
5. ​    android:layout_height="fill_parent"     
6. ​    android:gravity="center">    
7. ​    <LinearLayout    
8. ​        android:layout_width="fill_parent"    
9. ​        android:layout_height="wrap_content"    
10. ​        android:layout_marginBottom="10px"    
11. ​        android:orientation="horizontal"     
12. ​        android:gravity="center">    
13. ​        <TextView      
14. ​        android:layout_width="wrap_content"     
15. ​            android:layout_height="wrap_content"     
16. ​            android:layout_marginRight="4px"    
17. ​            android:gravity="center"    
18. ​            android:text="@string/counter">    
19. ​        </TextView>    
20. ​        <TextView      
21. ​            android:id="@+id/textview_counter"    
22. ​        android:layout_width="wrap_content"     
23. ​            android:layout_height="wrap_content"     
24. ​            android:gravity="center"    
25. ​            android:text="0">    
26. ​        </TextView>    
27. ​    </LinearLayout>    
28. ​    <LinearLayout    
29. ​        android:layout_width="fill_parent"    
30. ​        android:layout_height="wrap_content"    
31. ​        android:orientation="horizontal"     
32. ​        android:gravity="center">    
33. ​        <Button     
34. ​            android:id="@+id/button_start"    
35. ​            android:layout_width="wrap_content"    
36. ​            android:layout_height="wrap_content"    
37. ​            android:gravity="center"    
38. ​            android:text="@string/start">    
39. ​        </Button>    
40. ​        <Button     
41. ​            android:id="@+id/button_stop"    
42. ​            android:layout_width="wrap_content"    
43. ​            android:layout_height="wrap_content"    
44. ​            android:gravity="center"    
45. ​            android:text="@string/stop" >    
46. ​        </Button>    
47. ​     </LinearLayout>      
48. </LinearLayout>    

​       这个界面配置文件也很简单，等一下我们在模拟器把这个应用程序启动起来后，就可以看到它的截图了。

​       应用程序用到的字符串资源文件位于res/values/strings.xml文件中：

**[html]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. <?xml version="1.0" encoding="utf-8"?>    
2. <resources>    
3. ​    <string name="app_name">Counter</string>    
4. ​    <string name="counter">Counter: </string>    
5. ​    <string name="start">Start Counter</string>    
6. ​    <string name="stop">Stop Counter</string>    
7. </resources>   

​       最后，我们还要在工程目录下放置一个编译脚本文件Android.mk：

**[html]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. LOCAL_PATH:= $(call my-dir)          
2. include $(CLEAR_VARS)          
3. ​          
4. LOCAL_MODULE_TAGS := optional          
5. ​          
6. LOCAL_SRC_FILES := $(call all-subdir-java-files)          
7. ​          
8. LOCAL_PACKAGE_NAME := Counter          
9. ​          
10. include $(BUILD_PACKAGE)    

​       接下来就要编译了。有关如何单独编译Android源代码工程的模块，以及如何打包system.img，请参考

如何单独编译Android源代码中的模块

一文。

​       执行以下命令进行编译和打包：

**[html]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. USER-NAME@MACHINE-NAME:~/Android$ mmm packages/experimental/Counter            
2. USER-NAME@MACHINE-NAME:~/Android$ make snod  

​       这样，打包好的Android系统镜像文件system.img就包含我们前面创建的Counter应用程序了。

​       再接下来，就是运行模拟器来运行我们的例子了。关于如何在Android源代码工程中运行模拟器，请参考

在Ubuntu上下载、编译和安装Android最新源代码

一文。

​       执行以下命令启动模拟器：

**[html]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. USER-NAME@MACHINE-NAME:~/Android$ emulator   

​       最后我们就可以在Launcher中找到Counter应用程序图标，把它启动起来，点击Start按钮，就会看到应用程序界面上的计数器跑起来了：

![img](http://hi.csdn.net/attachment/201110/27/0_1319731144Ll99.gif)

​        这样，使用AsyncTask的例子就介绍完了，下面，我们就要根据上面对AsyncTask的使用情况来重点分析它的实现了。

​        AsyncTask类定义在frameworks/base/core/java/android/os/AsyncTask.java文件中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. public abstract class AsyncTask<Params, Progress, Result> {  
2. ​    ......  
3.   
4. ​    private static final BlockingQueue<Runnable> sWorkQueue =  
5. ​            new LinkedBlockingQueue<Runnable>(10);  
6.   
7. ​    private static final ThreadFactory sThreadFactory = new ThreadFactory() {  
8. ​        private final AtomicInteger mCount = new AtomicInteger(1);  
9.   
10. ​        public Thread newThread(Runnable r) {  
11. ​            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());  
12. ​        }  
13. ​    };  
14.   
15. ​    ......  
16.   
17. ​    private static final ThreadPoolExecutor sExecutor = new ThreadPoolExecutor(CORE_POOL_SIZE,  
18. ​        MAXIMUM_POOL_SIZE, KEEP_ALIVE, TimeUnit.SECONDS, sWorkQueue, sThreadFactory);  
19.   
20. ​    private static final int MESSAGE_POST_RESULT = 0x1;  
21. ​    private static final int MESSAGE_POST_PROGRESS = 0x2;  
22. ​    private static final int MESSAGE_POST_CANCEL = 0x3;  
23.   
24. ​    private static final InternalHandler sHandler = new InternalHandler();  
25.   
26. ​    private final WorkerRunnable<Params, Result> mWorker;  
27. ​    private final FutureTask<Result> mFuture;  
28.   
29. ​    ......  
30.   
31. ​    public AsyncTask() {  
32. ​        mWorker = new WorkerRunnable<Params, Result>() {  
33. ​            public Result call() throws Exception {  
34. ​                ......  
35. ​                return doInBackground(mParams);  
36. ​            }  
37. ​        };  
38.   
39. ​        mFuture = new FutureTask<Result>(mWorker) {  
40. ​            @Override  
41. ​            protected void done() {  
42. ​                Message message;  
43. ​                Result result = null;  
44.   
45. ​                try {  
46. ​                    result = get();  
47. ​                } catch (InterruptedException e) {  
48. ​                    android.util.Log.w(LOG_TAG, e);  
49. ​                } catch (ExecutionException e) {  
50. ​                    throw new RuntimeException("An error occured while executing doInBackground()",  
51. ​                        e.getCause());  
52. ​                } catch (CancellationException e) {  
53. ​                    message = sHandler.obtainMessage(MESSAGE_POST_CANCEL,  
54. ​                        new AsyncTaskResult<Result>(AsyncTask.this, (Result[]) null));  
55. ​                    message.sendToTarget();  
56. ​                    return;  
57. ​                } catch (Throwable t) {  
58. ​                    throw new RuntimeException("An error occured while executing "  
59. ​                        + "doInBackground()", t);  
60. ​                }  
61.   
62. ​                message = sHandler.obtainMessage(MESSAGE_POST_RESULT,  
63. ​                    new AsyncTaskResult<Result>(AsyncTask.this, result));  
64. ​                message.sendToTarget();  
65. ​            }  
66. ​        };  
67. ​    }  
68.   
69. ​    ......  
70.   
71. ​    public final Result get() throws InterruptedException, ExecutionException {  
72. ​        return mFuture.get();  
73. ​    }  
74.   
75. ​    ......  
76.   
77. ​    public final AsyncTask<Params, Progress, Result> execute(Params... params) {  
78. ​        ......  
79.   
80. ​        mWorker.mParams = params;  
81. ​        sExecutor.execute(mFuture);  
82.   
83. ​        return this;  
84. ​    }  
85.   
86. ​    ......  
87.   
88. ​    protected final void publishProgress(Progress... values) {  
89. ​        sHandler.obtainMessage(MESSAGE_POST_PROGRESS,  
90. ​            new AsyncTaskResult<Progress>(this, values)).sendToTarget();  
91. ​    }  
92.   
93. ​        private void finish(Result result) {  
94. ​                ......  
95. ​                onPostExecute(result);  
96. ​                ......  
97. ​        }  
98.   
99. ​    ......  
100.   
101. ​    private static class InternalHandler extends Handler {  
102. ​        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})  
103. ​        @Override  
104. ​        public void handleMessage(Message msg) {  
105. ​            AsyncTaskResult result = (AsyncTaskResult) msg.obj;  
106. ​            switch (msg.what) {  
107. ​                case MESSAGE_POST_RESULT:  
108. ​                 // There is only one result  
109. ​                 result.mTask.finish(result.mData[0]);  
110. ​                 break;  
111. ​                case MESSAGE_POST_PROGRESS:  
112. ​                 result.mTask.onProgressUpdate(result.mData);  
113. ​                 break;  
114. ​                case MESSAGE_POST_CANCEL:  
115. ​                 result.mTask.onCancelled();  
116. ​                 break;  
117. ​            }  
118. ​        }  
119. ​    }  
120.   
121. ​    private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {  
122. ​        Params[] mParams;  
123. ​    }  
124.   
125. ​    private static class AsyncTaskResult<Data> {  
126. ​        final AsyncTask mTask;  
127. ​        final Data[] mData;  
128.   
129. ​        AsyncTaskResult(AsyncTask task, Data... data) {  
130. ​            mTask = task;  
131. ​            mData = data;  
132. ​        }  
133. ​    }  
134. }  

​        从AsyncTask的实现可以看出，当我们第一次创建一个AsyncTask对象时，首先会执行下面静态初始化代码创建一个线程池sExecutor：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. private static final BlockingQueue<Runnable> sWorkQueue =  
2. ​    new LinkedBlockingQueue<Runnable>(10);  
3.   
4. private static final ThreadFactory sThreadFactory = new ThreadFactory() {  
5. ​    private final AtomicInteger mCount = new AtomicInteger(1);  
6.   
7. ​    public Thread newThread(Runnable r) {  
8. ​        return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());  
9. ​    }  
10. };  
11.   
12. ......  
13.   
14. private static final ThreadPoolExecutor sExecutor = new ThreadPoolExecutor(CORE_POOL_SIZE,  
15. ​    MAXIMUM_POOL_SIZE, KEEP_ALIVE, TimeUnit.SECONDS, sWorkQueue, sThreadFactory);  

​        这里的ThreadPoolExecutor是Java提供的多线程机制之一，这里用的构造函数原型为：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit,   
2. ​    BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory)  

​        各个参数的意义如下：

​        **corePoolSize -- 线程池的核心线程数量**

**        maximumPoolSize -- 线程池的最大线程数量**

**        keepAliveTime -- 若线程池的线程数数量大于核心线程数量，那么空闲时间超过keepAliveTime的线程将被回收**

**        unit -- 参数keepAliveTime使用的时间单位**

**        workerQueue -- 工作任务队列**

**        threadFactory -- 用来创建线程池中的线程**
​        简单来说，ThreadPoolExecutor的运行机制是这样的：每一个工作任务用一个Runnable对象来表示，当我们要把一个工作任务交给这个线程池来执行的时候，就通过调用ThreadPoolExecutor的execute函数来把这个工作任务加入到线程池中去。此时，如果线程池中的线程数量小于corePoolSize，那么就会调用threadFactory接口来创建一个新的线程并且加入到线程池中去，再执行这个工作任务；如果线程池中的线程数量等于corePoolSize，但是工作任务队列workerQueue未满，则把这个工作任务加入到工作任务队列中去等待执行；如果线程池中的线程数量大于corePoolSize，但是小于maximumPoolSize，并且工作任务队列workerQueue已经满了，那么就会调用threadFactory接口来创建一个新的线程并且加入到线程池中去，再执行这个工作任务；如果线程池中的线程量已经等于maximumPoolSize了，并且工作任务队列workerQueue也已经满了，这个工作任务就被拒绝执行了。

​        创建好了线程池后，再创建一个消息处理器：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. private static final InternalHandler sHandler = new InternalHandler();  

​        注意，这行代码是在应用程序的主线程中执行的，因此，这个消息处理器sHandler内部引用的消息循环对象looper是应用程序主线程的消息循环对象，消息处理器的实现机制具体可以参考前面一篇文章

Android应用程序消息处理机制（Looper、Handler）分析

。

​        AsyncTask类的静态初始化代码执行完成之后，才开始创建AsyncTask对象，即执行AsyncTask类的构造函数：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. public AsyncTask() {  
2. ​    mWorker = new WorkerRunnable<Params, Result>() {  
3. ​        public Result call() throws Exception {  
4. ​            ......  
5. ​            return doInBackground(mParams);  
6. ​        }  
7. ​    };  
8.   
9. ​    mFuture = new FutureTask<Result>(mWorker) {  
10. ​        @Override  
11. ​        protected void done() {  
12. ​            Message message;  
13. ​            Result result = null;  
14.   
15. ​            try {  
16. ​                result = get();  
17. ​            } catch (InterruptedException e) {  
18. ​                android.util.Log.w(LOG_TAG, e);  
19. ​            } catch (ExecutionException e) {  
20. ​                throw new RuntimeException("An error occured while executing doInBackground()",  
21. ​                    e.getCause());  
22. ​            } catch (CancellationException e) {  
23. ​                message = sHandler.obtainMessage(MESSAGE_POST_CANCEL,  
24. ​                    new AsyncTaskResult<Result>(AsyncTask.this, (Result[]) null));  
25. ​                message.sendToTarget();  
26. ​                return;  
27. ​            } catch (Throwable t) {  
28. ​                throw new RuntimeException("An error occured while executing "  
29. ​                    + "doInBackground()", t);  
30. ​            }  
31.   
32. ​            message = sHandler.obtainMessage(MESSAGE_POST_RESULT,  
33. ​                new AsyncTaskResult<Result>(AsyncTask.this, result));  
34. ​            message.sendToTarget();  
35. ​        }  
36. ​    };  
37. }  

​        在AsyncTask类的构造函数里面，主要是创建了两个对象，分别是一个WorkerRunnable对象mWorker和一个FutureTask对象mFuture。

​        WorkerRunnable类实现了Callable接口，此外，它的内部成员变量mParams用于保存从AsyncTask对象的execute函数传进来的参数列表：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {  
2. ​    Params[] mParams;  
3. }  

​        FutureTask类也实现了Runnable接口，所以它可以作为一个工作任务通过调用AsyncTask类的execute函数添加到sExecuto线程池中去：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. public final AsyncTask<Params, Progress, Result> execute(Params... params) {  
2. ​    ......  
3.   
4. ​    mWorker.mParams = params;  
5. ​    sExecutor.execute(mFuture);  
6.   
7. ​    return this;  
8. }  

​       这里的FutureTask对象mFuture是用来封装前面的WorkerRunnable对象mWorker。当mFuture加入到线程池中执行时，它调用的是mWorker对象的call函数：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. mWorker = new WorkerRunnable<Params, Result>() {  
2. ​    public Result call() throws Exception {  
3. ​           ......  
4. ​           return doInBackground(mParams);  
5. ​        }  
6. };  

​        在call函数里面，会调用AsyncTask类的doInBackground函数来执行真正的任务，这个函数是要由AsyncTask的子类来实现的，注意，这个函数是在应用程序的子线程中执行的，它不可以操作应用程序的界面。

​        我们可以通过mFuture对象来操作当前执行的任务，例如查询当前任务的状态，它是正在执行中，还是完成了，还是被取消了，如果是完成了，还可以通过它获得任务的执行结果，如果还没有完成，可以取消任务的执行。

​        当工作任务mWorker执行完成的时候，mFuture对象中的done函数就会被被调用，根据任务的完成状况，执行相应的操作，例如，如果是因为异常而完成时，就会抛异常，如果是正常完成，就会把任务执行结果封装成一个AsyncTaskResult对象：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. private static class AsyncTaskResult<Data> {  
2. ​    final AsyncTask mTask;  
3. ​    final Data[] mData;  
4.   
5. ​    AsyncTaskResult(AsyncTask task, Data... data) {  
6. ​        mTask = task;  
7. ​        mData = data;  
8. ​    }  
9. }  

​        其中，成员变量mData保存的是任务执行结果，而成员变量mTask指向前面我们创建的AsyncTask对象。

​        最后把这个AsyncTaskResult对象封装成一个消息，并且通过消息处理器sHandler加入到应用程序主线程的消息队列中：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. message = sHandler.obtainMessage(MESSAGE_POST_RESULT,  
2. ​    new AsyncTaskResult<Result>(AsyncTask.this, result));  
3. message.sendToTarget();  

​        这个消息最终就会在InternalHandler类的handleMessage函数中处理了：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. private static class InternalHandler extends Handler {  
2. ​    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})  
3. ​    @Override  
4. ​    public void handleMessage(Message msg) {  
5. ​        AsyncTaskResult result = (AsyncTaskResult) msg.obj;  
6. ​        switch (msg.what) {  
7. ​        case MESSAGE_POST_RESULT:  
8. ​            // There is only one result  
9. ​            result.mTask.finish(result.mData[0]);  
10. ​            break;  
11. ​        ......  
12. ​        }  
13. ​    }  
14. }  

​        在这个函数里面，最终会调用前面创建的这个AsyncTask对象的finish函数来进一步处理：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. private void finish(Result result) {  
2. ​       ......  
3. ​       onPostExecute(result);  
4. ​       ......  
5. }  

​        这个函数调用AsyncTask类的onPostExecute函数来进一步处理，AsyncTask类的onPostExecute函数一般是要由其子类来重载的，注意，这个函数是在应用程序的主线程中执行的，因此，它可以操作应用程序的界面。

​        在任务执行的过程当中，即执行doInBackground函数时候，可能通过调用publishProgress函数来将中间结果封装成一个消息发送到应用程序主线程中的消息队列中去：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. protected final void publishProgress(Progress... values) {  
2. ​    sHandler.obtainMessage(MESSAGE_POST_PROGRESS,  
3. ​        new AsyncTaskResult<Progress>(this, values)).sendToTarget();  
4. }  

​        这个消息最终也是由InternalHandler类的handleMessage函数来处理的：

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/6905587#) [copy](http://blog.csdn.net/luoshengyang/article/details/6905587#)

1. private static class InternalHandler extends Handler {  
2. ​    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})  
3. ​    @Override  
4. ​    public void handleMessage(Message msg) {  
5. ​        AsyncTaskResult result = (AsyncTaskResult) msg.obj;  
6. ​        switch (msg.what) {  
7. ​        ......  
8. ​        case MESSAGE_POST_PROGRESS:  
9. ​                 result.mTask.onProgressUpdate(result.mData);  
10. ​                 break;  
11. ​        ......  
12. ​        }  
13. ​    }  
14. }  

​        这里它调用前面创建的AsyncTask对象的onPorgressUpdate函数来进一步处理，这个函数一般是由AsyncTask的子类来实现的，注意，这个函数是在应用程序的主线程中执行的，因此，它和前面的onPostExecute函数一样，可以操作应用程序的界面。

​       这样，AsyncTask类的主要实现就介绍完了，结合前面开发的应用程序Counter来分析，会更好地理解它的实现原理。

​       至此，Android应用程序线程消息循环模型就分析完成了，理解它有利于我们在开发Android应用程序时，能够充分利用多线程的并发性来提高应用程序的性能以及获得良好的用户体验。