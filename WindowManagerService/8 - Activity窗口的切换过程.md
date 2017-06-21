在[Android](http://lib.csdn.net/base/android)系统中，同一时刻只有一个Activity组件是处于激活状态的，因此，当ActivityManagerService服务激活了一个新的Activity组件时，它就需要通知WindowManagerService服务将该Activity组件的窗口显示出来，这会涉及到将焦点和屏幕等资源从前一个激活的Activity组件切换到后一个激活的Activity组件的过程，本文就详细分析这个过程。

《[android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

Activity窗口的切换操作是在新激活的Activity组件的启动过程进行的。具体来说，就是在前一个激活的Activity组件进入到Paused状态并且新激活的Activity组件进之到Resumed 状态之后，将前一个激活的Activity组件的窗口设置为不可见，以及将新激活的Activity组件的窗口设置为可见。整个切换过程是需要在ActivityManagerService服务和WindowManagerService服务的协作之下进行的，如图1所示。

![img](http://img.my.csdn.net/uploads/201302/21/1361378601_3774.jpg)

图1 Activity窗口的切换操作示意图

WindowManagerService服务在执行Activity窗口的切换操作的时候，会给参与切换操作的Activity组件的设置一个动画，以便可以向用户展现一个Activity组件切换效果，从而提高用户体验。事实上，一个Activity窗口在由不可见状态切换到可见状态的过程中，除了会被设置一个Activity组件切换动画之外，还有被设置一个窗口进入动画，此外，如果该Activity窗口是附加在另外一个窗口之上的，并且这个被附加的窗口正在显示一个动画，那么这个动画也同时会被设置给到该Activity窗口的显示过程中去。本文主要是关注Activity窗口的切换操作，在接下来的一篇文章中分析窗口的动画框架时，我们再详细分析上述三种动画是如何作用在窗口的显示过程中的。

从前面[Android应用程序启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6689748)一文可以知道，ActivityManagerService服务在启动一个Activity组件的过程中，会调用到ActivityStack类的成员函数startActivityLocked。ActivityStack类的成员函数startActivityLocked首先会给正在启动的Activity组件准备一个切换操作，接着再调用其它的成员函数来通知前一个激活的Activity组件进入到Paused状态。等到前一个激活的Activity组件进入到Paused状态之后，ActivityManagerService服务就会检查用来运行正在启动的Activity组件的进程启动起来了没有。如果这个进程还没有启动，那么ActivityManagerService服务就会将该进程启动起来，然后再调用ActivityStack类的成员函数realStartActivityLocked来将正在启动的Activity组件加载起来，并且将它的状态设置为Resumed，最后通知WindowManagerService服务执行前面所准备的切换操作。

接下来，我们就从ActivityStack类的成员函数startActivityLocked开始分析Activity窗口的切换过程，如图2所示。

![img](http://img.my.csdn.net/uploads/201302/21/1361379970_6984.jpg)

图2 Activity窗口的切换过程

这个过程可以分为9个步骤，接下来我们就详细分析每一个步骤。

### Step 1. ActivityStack.startActivityLocked

```java
public class ActivityStack {  
    ......  

    private final void startActivityLocked(ActivityRecord r, boolean newTask,  
   boolean doResume) {  
final int NH = mHistory.size();  

int addPos = -1;  

if (!newTask) {  
   // If starting in an existing task, find where that is...  
   ......  
}  

// Place a new activity at top of stack, so it is next to interact  
// with the user.  
if (addPos < 0) {  
   addPos = NH;  
}  
......  

// Slot the activity into the history stack and proceed  
mHistory.add(addPos, r);  
......  

if (NH > 0) {  
   ......  

   if ((r.intent.getFlags()&Intent.FLAG_ACTIVITY_NO_ANIMATION) != 0) {  
       mService.mWindowManager.prepareAppTransition(WindowManagerPolicy.TRANSIT_NONE);  
       ......  
   } else if ((r.intent.getFlags()&Intent.FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET) != 0) {  
       mService.mWindowManager.prepareAppTransition(  
               WindowManagerPolicy.TRANSIT_TASK_OPEN);  
       ......  
   } else {  
       mService.mWindowManager.prepareAppTransition(newTask  
               ? WindowManagerPolicy.TRANSIT_TASK_OPEN  
               : WindowManagerPolicy.TRANSIT_ACTIVITY_OPEN);  
       ......  
   }  

   ......  
}  

......  

if (doResume) {  
   resumeTopActivityLocked(null);  
}  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/services/[Java](http://lib.csdn.net/base/java)/com/android/server/am/ActivityStack.java中。

参数r描述的是正在启动的Activity组件，ActivityStack类的成员函数startActivityLocked首先找到它在Activity组件堆栈中的位置addPos，即找到它在ActivityStack类的成员变量mHistory所描述的一个ArrayList中的位置，然后再将正在启动的Activity组件保存在该位置中。

变量NH描述的是将参数r描述的是正在启动的Activity组件保存在Activity组件堆栈前系统已经启动了的Activity组件的个数。只有在这个变量的值大于0的情况下，系统才需要执行一个Activity组件切换操作。也就是说，如果参数r描述的是正在启动的Activity组件是系统中第一个启动的Activity组件，那么就不需要执行一个Activity组件切换操作了。

注意，即使参数r描述的是正在启动的Activity组件不是系统中第一个启动的Activity组件，那么系统也可能不需要执行一个Activity组件切换操作，因为用来启动参数r所描述的一个Activity组件的一个Intent对象的成员函数getFlags返回的一个标志值的Intent.FLAG_ACTIVITY_NO_ANIMATION位可能会不等于0，这意味着正在启动的Activity组件不需要显示切换动画。在这种情况下，ActivityManagerSerivce服务就会通知WindowManagerService服务不需要准备一个Activity组件切换操作，这是通过以WindowManagerPolicy.TRANSIT_NONE为参数来调用ActivityStack类的成员变量mService所指向的一个ActivityManagerService对象的成员变量mWindowManager所描述的一个WindowManagerService对象的成员函数prepareAppTransition来实现的。

另一方面，如果参数r描述的是正在启动的Activity组件不是系统中第一个启动的Activity组件，并且系统需要执行一个Activity组件切换操作，即需要WindowManagerService服务在显示正在启动的Activity组件的窗口时应用一个切换动画，那么这个动画的类型也是有讲究的。具体来说，如果参数r描述的是Activity组件是需要在在一个新的任务中启动的，即参数newTask的值等于true，那么切换动画的类型就指定为WindowManagerPolicy.TRANSIT_TASK_OPEN，否则的话，切换动画的类型就指定为WindowManagerPolicy.TRANSIT_ACTIVITY_OPEN。

此外，如果用来启动参数r所描述的一个Activity组件的一个Intent对象的成员函数getFlags返回的一个标志值的Intent.FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET位不等于0，那么也会将切换动画的类型设置为WindowManagerPolicy.TRANSIT_TASK_OPEN。Intent.FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET这个标志位等于1意味着当参数r所描述的一个Activity组件所在的任务下次作为前台任务来运行时，参数r所描述的Activity组件以及它上面的并且属于同一个任务的其它Activity组件都会被结束掉，以使得位于参数r所描述的Activity组件的前面一个Activity组件可以显示出来。例如，如果我们正在使用一个Email Activity组件来查看Email，这时候又需要启动另外一个Pictrue Activity来查看该Email附件的一张图片。这时候Email Activity和Pictrue Activity就是在同一个任务中的。突然间，我们因为其它原历，需要按下Home键回到Launcher中去完成其它任务。完成其它任务之后，再点击Launcher上面的Email Activity图标来想重新浏览之前正在查看的Email，但是又不想看到的是该Email附件的那张图片。在这种情况下，之前在启动Pictrue Activity来查看Email的附件图片时，就可以将用来启动Pictrue Activity的一个Intent对象的标志值的Intent.FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET位设置为1。

无论如何，最终指定的切换动画的类型都是通过调用ActivityStack类的成员变量mService所指向的一个ActivityManagerService对象的成员变量mWindowManager所描述的一个WindowManagerService对象的成员函数prepareAppTransition来通知WindowManagerService服务的。

ActivityStack类的成员函数startActivityLocked通知WindowManagerService服务准备好一个Activity组件切换操作之后，如果参数doResume的值等于true，那么它就会继续调用另外一个成员函数resumeTopActivityLocked来继续执行启动参数r所描述的一个Activity组件的操作。

接下来，我们就首先分析WindowManagerService类的成员函数prepareAppTransition的实现，以便可以了解WindowManagerService服务是如何准备一个Activity组件切换操作的，然后再回过头来分析ActivityStack类的成员函数resumeTopActivityLocked是如何继续执行启动参数r所描述的一个Activity组件的操作的。

### Step 2. WindowManagerService.prepareAppTransition

```java
public class WindowManagerService extends IWindowManager.Stub  
implements Watchdog.Monitor {  
    ......  

    // State management of app transitions.  When we are preparing for a  
    // transition, mNextAppTransition will be the kind of transition to  
    // perform or TRANSIT_NONE if we are not waiting.  If we are waiting,  
    // mOpeningApps and mClosingApps are the lists of tokens that will be  
    // made visible or hidden at the next transition.  
    int mNextAppTransition = WindowManagerPolicy.TRANSIT_UNSET;  
    ......  
    boolean mAppTransitionReady = false;  
    ......  
    boolean mAppTransitionTimeout = false;  
    boolean mStartingIconInTransition = false;  
    boolean mSkipAppTransitionAnimation = false;  


    public void prepareAppTransition(int transit) {  
if (!checkCallingPermission(android.Manifest.permission.MANAGE_APP_TOKENS,  
       "prepareAppTransition()")) {  
   throw new SecurityException("Requires MANAGE_APP_TOKENS permission");  
}  

synchronized(mWindowMap) {  
   ......  
   if (!mDisplayFrozen && mPolicy.isScreenOn()) {  
       if (mNextAppTransition == WindowManagerPolicy.TRANSIT_UNSET  
               || mNextAppTransition == WindowManagerPolicy.TRANSIT_NONE) {  
           mNextAppTransition = transit;  
       } else if (transit == WindowManagerPolicy.TRANSIT_TASK_OPEN  
               && mNextAppTransition == WindowManagerPolicy.TRANSIT_TASK_CLOSE) {  
           // Opening a new task always supersedes a close for the anim.  
           mNextAppTransition = transit;  
       } else if (transit == WindowManagerPolicy.TRANSIT_ACTIVITY_OPEN  
               && mNextAppTransition == WindowManagerPolicy.TRANSIT_ACTIVITY_CLOSE) {  
           // Opening a new activity always supersedes a close for the anim.  
           mNextAppTransition = transit;  
       }  
       mAppTransitionReady = false;  
       mAppTransitionTimeout = false;  
       mStartingIconInTransition = false;  
       mSkipAppTransitionAnimation = false;  
       mH.removeMessages(H.APP_TRANSITION_TIMEOUT);  
       mH.sendMessageDelayed(mH.obtainMessage(H.APP_TRANSITION_TIMEOUT),  
               5000);  
   }  
}  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/services/java/com/android/server/WindowManagerService.java中。

调用WindowManagerService类的成员函数prepareAppTransition来通知WindowManagerService服务准备一个Activity组件切换操作是需要具有android.Manifest.permission.MANAGE_APP_TOKENS，否则的话，WindowManagerService类的成员函数prepareAppTransition就会抛出一个类型为SecurityException的异常。

WindowManagerService类的成员变量mNextAppTransition描述的就是WindowManagerService服务接下来要执行一个Activity组件切换操作的类型，也就是要执行的一个Activity组件切换动画的类型。WindowManagerService类的成员函数prepareAppTransition按照以下规则来设置WindowManagerService服务接下来要执行的Activity组件切换动画的类型：

当 WindowManagerService类的成员变量mNextAppTransition的值等于WindowManagerPolicy.TRANSIT_UNSET或者WindowManagerPolicy.TRANSIT_NONE的时候，就说明WindowManagerService服务接下来没有Activity组件切换动画等待执行的，这时候参数transit所描述的Activity组件切换动画就可以作为WindowManagerService服务接下来要执行的Activity组件切换动画。

当WindowManagerService类的成员变量mNextAppTransition的值等于WindowManagerPolicy.TRANSIT_TASK_CLOSE，那么就说明WindowManagerService服务接下来要执行一个关闭Activity组件任务的切换动画等待执行的。在这种情况下，如果参数transit所描述的是一个打开Activity组件任务的切换动画，即它的值等于WindowManagerPolicy.TRANSIT_TASK_OPEN，那么就需要将WindowManagerService服务接下来要执行的Activity组件切换动画为打开Activity组件任务类型的。这是因为打开Activity组件任务的切换动画的优先级高于关闭Activity组件任务的切换动画。

当WindowManagerService类的成员变量mNextAppTransition的值等于WindowManagerPolicy.TRANSIT_ACTIVITY_CLOSE，那么就说明WindowManagerService服务接下来要执行一个关闭Activity组件的切换动画等待执行的。在这种情况下，如果参数transit所描述的是一个打开Activity组件的切换动画，即它的值等于WindowManagerPolicy.TRANSIT_ACTIVITY_OPEN，那么就需要将WindowManagerService服务接下来要执行的Activity组件切换动画为打开Activity组件类型的。这是因为打开Activity组件的切换动画的优先级高于关闭Activity组件的切换动画。

设置好WindowManagerService服务接下来要执行的Activity组件切换动画的类型之后，WindowManagerService类的成员函数prepareAppTransition还会将其余四个成员变量mAppTransitionReady、mAppTransitionTimeout、mStartingIconInTransition和mSkipAppTransitionAnimation的值设置为false，其中：

mAppTransitionReady表示WindowManagerService服务可以开始执行一个Activity组件的切换动画了没有？

mAppTransitionTimeout表示WindowManagerService服务正在执行的Activity组件切换动画是否已经超时？

mStartingIconInTransition表示WindowManagerService服务开始显示正在启动的Activity组件的启动窗口了没有？

mSkipAppTransitionAnimation表示WindowManagerService服务是否需要不执行Activity组件的切换动画？

最后，WindowManagerService类的成员函数prepareAppTransition还会调用成员变量mH所描述的一个H对象的成员函数sendMessageDelayed来向WindowManagerService服务所运行在的线程发送一个类型为APP_TRANSITION_TIMEOUT的消息。这个消息将在5秒后被执行，是用来强制前面所设置的Activity组件切换动画要在5秒之内执行完成的，否则的话，WindowManagerService服务就会认为该切换动画执行超时了。

这一步执行完成之后，WindowManagerService服务接下来要执行的Activity组件切换操作或者说切换动画就准备完成了。注意，这时候只是准备好Activity组件切换动画，但是这个切换动画还不能执行，要等到前一个激活的Activity组件进入到Paused状态并且接下来正在启动的Activity组件进入到Resumed状态之后才能执行。

回到前面的Step 1中，即ActivityStack类的成员函数startActivityLocked，接下来它就会调用另外一个成员函数resumeTopActivityLocked来继续启动指定的Activity组件。从前面[Android应用程序启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6689748)一文可以知道，ActivityStack类的成员函数resumeTopActivityLocked以及接下来要调用的其它成员函数就是执行以下两个操作：

通知当前处于激活状态的Activity组件所运行在的应用程序进程，它所运行的一个Activity组件要由Resumed状态进入到Paused状态了。

检查用来运行当前正在启动的Activity组件的应用程序进程是否已经启动起来了。如果已经启动起来，那么就会直接通知该应用程序进程将正在启动的Activity组件加载起来，否则的话，就会先将该应用程序进程启动起来，然后再通知它将正在启动的Activity组件加载起来。

在第2步中，通知相应的应用程序进程将正在启动的Activity组件加载起来是通过调用ActivityStack类的成员函数realStartActivityLocked来实现的，接下来我们就继续分析这个成员函数的实现，以便可以继续了解Activity组件的切换操作的执行过程。

### Step 3. ActivityStack.realStartActivityLocked

```java
public class ActivityStack {  
    ......  

    final boolean realStartActivityLocked(ActivityRecord r,  
   ProcessRecord app, boolean andResume, boolean checkConfig)  
   throws RemoteException {  
......  

mService.mWindowManager.setAppVisibility(r, true);  
......  

try {  
   ......  

   app.thread.scheduleLaunchActivity(new Intent(r.intent), r,  
           System.identityHashCode(r),  
           r.info, r.icicle, results, newIntents, !andResume,  
           mService.isNextTransitionForward());  
   ......  

} catch (RemoteException e) {  
   ......  
}  

......  

if (andResume) {  
   // As part of the process of launching, ActivityThread also performs  
   // a resume.  
   r.state = ActivityState.RESUMED;  
   ......  
   mResumedActivity = r;  
   ......  
   completeResumeLocked(r);  
   ......  
}   

......  

return true;  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/services/java/com/android/server/am/ActivityStack.java中。

ActivityStack类的成员函数realStartActivityLocked主要执行以下三个操作：

通知WindowManagerService服务将参数r所描述的Activity组件的可见性设置为true，这是通过调用ActivityStack类的成员变量mService所指向的一个ActivityManagerService对象的成员变量mWindowManager所描述的一个WindowManagerService对象的成员函数setAppVisibility来实现的。

通知参数r所描述的Activity组件所运行在的应用程序进程将它加载起来，这是通过调用参数app所指向的一个ProcessRecord对象的成员变量thread所描述的一个类型为ApplictionThread的Binder代理对象的成员函数scheduleLaunchActivity来实现的。

当参数andResume的值等于true的时候，就表示在执行第2步时，参数r所描述的Activity组件所运行在的应用程序进程已经将它的状态设置为Resumed了，即已经调用过它的成员函数onResume了。在这种情况，ActivityManagerService服务也需要将该Activity组件的状态设置为Resumed了，即将r所指向的一个ActivityRecord对象的成员变量state的值设置为ActivityState.RESUMED，并且将ActivityStack类的成员变量mResumedActivity的值设置为r，以便表示当前激活的Activity组件为参数r所描述的Activity组件。最后，ActivityManagerService服务还需要调用ActivityStack类的成员函数completeResumeLocked来通知WindowManagerService服务执行在前面Step 2所准备好的Activity组件切换操作。

接下来，我们首先分析WindowManagerService类的成员函数setAppVisibility的实现，以便可以了解WindowManagerService服务是如何设置一个Activity组件的可见性的，接着再分析ActivityStack类的成员函数completeResumeLocked的实现，以便可以了解ActivityManagerService服务是如何通知WindowManagerService服务执行前面所准备好的一个Activity组件切换操作的。

### Step 4. WindowManagerService.setAppVisibility

```java
public class WindowManagerService extends IWindowManager.Stub  
implements Watchdog.Monitor {  
    ......  

    public void setAppVisibility(IBinder token, boolean visible) {  
if (!checkCallingPermission(android.Manifest.permission.MANAGE_APP_TOKENS,  
       "setAppVisibility()")) {  
   throw new SecurityException("Requires MANAGE_APP_TOKENS permission");  
}  

AppWindowToken wtoken;  

synchronized(mWindowMap) {  
   wtoken = findAppWindowToken(token);  
   ......  

   // If we are preparing an app transition, then delay changing  
   // the visibility of this token until we execute that transition.  
   if (!mDisplayFrozen && mPolicy.isScreenOn()  
           && mNextAppTransition != WindowManagerPolicy.TRANSIT_UNSET) {  
       // Already in requested state, don't do anything more.  
       if (wtoken.hiddenRequested != visible) {  
           return;  
       }  
       wtoken.hiddenRequested = !visible;  
       ......  

       wtoken.setDummyAnimation();  
       mOpeningApps.remove(wtoken);  
       mClosingApps.remove(wtoken);  
       wtoken.waitingToShow = wtoken.waitingToHide = false;  
       wtoken.inPendingTransaction = true;  
       if (visible) {  
           mOpeningApps.add(wtoken);  
           wtoken.startingDisplayed = false;  
           wtoken.startingMoved = false;  

           // If the token is currently hidden (should be the  
           // common case), then we need to set up to wait for  
           // its windows to be ready.  
           if (wtoken.hidden) {  
               wtoken.allDrawn = false;  
               wtoken.waitingToShow = true;  

               if (wtoken.clientHidden) {  
                   // In the case where we are making an app visible  
                   // but holding off for a transition, we still need  
                   // to tell the client to make its windows visible so  
                   // they get drawn.  Otherwise, we will wait on  
                   // performing the transition until all windows have  
                   // been drawn, they never will be, and we are sad.  
                   wtoken.clientHidden = false;  
                   wtoken.sendAppVisibilityToClients();  
               }  
           }  
       } else {  
           mClosingApps.add(wtoken);  

           // If the token is currently visible (should be the  
           // common case), then set up to wait for it to be hidden.  
           if (!wtoken.hidden) {  
               wtoken.waitingToHide = true;  
           }  
       }  
       return;  
   }  

   ......  
   setTokenVisibilityLocked(wtoken, null, visible, WindowManagerPolicy.TRANSIT_UNSET, true);  
   wtoken.updateReportedVisibilityLocked();  
   ......  
}  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/services/java/com/android/server/WindowManagerService.java中。

调用WindowManagerService类的成员函数setAppVisibility来设置Activity组件的可见性需要具有android.Manifest.permission.MANAGE_APP_TOKENS权限，否则的话，WindowManagerService类的成员函数setAppVisibility就会抛出一个类型为SecurityException的异常。

从前面[Android窗口管理服务WindowManagerService对窗口的组织方式分析](http://blog.csdn.net/luoshengyang/article/details/8498908)一文可以知道，每一个Activity组件在WindowManagerService服务内部都对应有一个AppWindowToken对象，用来描述该Activity组件的窗口在WindowManagerService服务中的状态。因此，WindowManagerService类的成员函数setAppVisibility就先通过调用成员函数findAppWindowToken来找到与参数token所描述的一个Activity组件所对应的一个AppWindowToken对象wtoken，以便接下来可以设置它的状态。

注意，WindowManagerService类的成员函数setAppVisibility在修改参数token所描述的一个Activity组件的可见性的时候，需要考虑WindowManagerService服务接下来是否需要执行一个Activity组件操作，即WindowManagerService类的成员变量mNextAppTransition的值是否等于WindowManagerPolicy.TRANSIT_UNSET。如果等于的话，那么参数token所描述的Activity组件就是正在等待执行切换操作的Activity组件，这时候修改的可见性就会复杂一些，否则的话，只要简单地执行以下两个操作即可：

调用WindowManagerService类的成员函数setTokenVisibilityLocked来将参数token所描述的Activity组件的可见性设置为参数visible所描述的值；

调用AppWindowToken对象wtoken的成员函数updateReportedVisibilityLocked来向ActivityManagerService服务报告参数token所描述的Activity组件的可见性。

我们假设WindowManagerService服务接下来需要执行一个Activity组件操作，即WindowManagerService类的成员变量mNextAppTransition的值等于WindowManagerPolicy.TRANSIT_UNSET，这时候还需要满足三个额外的条件，即：

屏幕当前不是处于冻结状态，即WindowManagerService类的成员变量mDisplayFrozen的值等于false；

屏幕当前是点亮的，即WindowManagerService类的成员变量mPolicy所指向的一个PhoneWindowManager对象的成员函数isScreenOn的返回值等于true；

参数token所描述的一个Activity组件的可见性已经不等于所要设置的可见性，即前面所找到的AppWindowToken对象wtoken的成员变量hiddenRequested的值不等于参数visible的值。

那么WindowManagerService类的成员函数setAppVisibility接下来才会开始修改参数token所描述的一个Activity组件的可见性，即：

将AppWindowToken对象wtoken的成员变量hiddenRequested的值设置为参数visible的相反值。也就是说，如果参数token所描述的Activity组件是可见的，那么就将AppWindowToken对象wtoken的成员变量hiddenRequested的值设置为false，否则的话，就设置为true。

调用AppWindowToken对象wtoken的成员函数setDummyAnimation来给参数token所描述的Activity组件设置一个哑动画。注意，要等到执行参数token所描述的Activity组件的切换操作时，WindowManagerService服务才会给该Activity组件设置一个合适的切换动画。

分别将AppWindowToken对象wtoken从WindowManagerService类的成员变量mOpeningApps和mClosingApps所描述的两个ArrayList中删除。注意，WindowManagerService类的成员变量mOpeningApps和mClosingApps保存的分别是系统当前正在打开和关闭的Activity组件，后面会根据参数visible的值来决定参数token所描述的Activity组件是正在打开的还是正在关闭的，以便可以将它放在WindowManagerService类的成员变量mOpeningApps或者mClosingApps中。

将AppWindowToken对象wtoken的成员变量waitingToShow和waitingToHide的值都初始化为false，表示参数token所描述的Activity组件既不是正在等待显示的，也不是正在等待隐藏的，这两个成员变量的值同样是要根据参数visible的值来设置的。

将AppWindowToken对象wtoken的成员变量inPendingTransaction的值设置为true，表示参数token所描述的Activity组件正在等待切换。

接下来的操作取决于参数visible的值是true还是false。

假设参数visible的值等于true，那么就表示要将参数token所描述的Activity组件设置可见，这时候WindowManagerService类的成员函数setAppVisibility会继续执行以下操作：

将AppWindowToken对象wtoken添加WindowManagerService类的成员变量mOpeningApps所描述的一个ArrayList中，表示参数token所描述的Activity组件是正在打开的。

将AppWindowToken对象wtoken的成员变量startingDisplayed和startingMoved的值都设置为false，表示参数token所描述的Activity组件的启动窗口还没有显示出来，以及也没有被转移给其它的Activity组件。

如果AppWindowToken对象wtoken的成员变量hidden的值等于true，那么就意味着参数token所描述的Activity组件当前是不可见的。由于在这种情况下，参数token所描述的Activity组件正在等待打开，因此，该Activity组件的窗口一定是还没有绘制出来，并且正在等待绘制以及显示出来，这时候就需要将AppWindowToken对象wtoken的成员变量allDrawn和waitingToShow的值分别设置为false和true。

如果AppWindowToken对象wtoken的成员变量hidden的值等于true，并且另外一个成员变量clientHidden的值也等于true，那么就说明在应用程序进程这一侧看来，参数token所描述的Activity组件是不可见的，这时候就需要让该应用程序进程认为参数token所描述的Activity组件是可见的，以便该应用程序进程可以将参数token所描述的Activity组件的窗口绘制出来，这样WindowManagerService服务接下来才可以将该Activity组件的窗口显示出来。通知应用程序进程将参数token所描述的Activity组件设置为true是通过调用AppWindowToken对象wtoken的成员函数sendAppVisibilityToClients来实现的，同时在通知之前，也会将AppWindowToken对象wtoken的成员变量clientHidden设置为false。

假设参数visible的值等于false，那么就表示要将参数token所描述的Activity组件设置不可见，这时候WindowManagerService类的成员函数setAppVisibility会继续执行以下操作：

将AppWindowToken对象wtoken添加WindowManagerService类的成员变量mClosingApps所描述的一个ArrayList中，表示参数token所描述的Activity组件是正在关闭的。

如果AppWindowToken对象wtoken的成员变量hidden的值等于false，那么就意味着参数token所描述的Activity组件当前是可见的。由于在这种情况下，参数token所描述的Activity组件正在等待关闭，因此，该Activity组件的窗口接下来的状态应该等待隐藏不见的，这时候就需要将AppWindowToken对象wtoken的成员变量waitingToHide的值设置为true。

这一步执行完成之后，参数token所描述的Activity组件的可见性就设置好了，回到前面的Step 3中，即ActivityStack类的成员函数realStartActivityLocked中，接下来就会通知参数token所描述的Activity组件所运行在的应用程序进程将它加载起来，并且最后调用ActivityStack类的成员函数completeResumeLocked来通知WindowManagerService服务执行在前面Step 2所准备好的Activity组件切换操作。

接下来，我们就继续分析ActivityStack类的成员函数completeResumeLocked的实现。

### Step 5. ActivityStack.completeResumeLocked

```java
public class ActivityStack {  
    ......  

    private final void completeResumeLocked(ActivityRecord next) {  
......  

// schedule an idle timeout in case the app doesn't do it for us.  
Message msg = mHandler.obtainMessage(IDLE_TIMEOUT_MSG);  
msg.obj = next;  
mHandler.sendMessageDelayed(msg, IDLE_TIMEOUT);  
......  

if (mMainStack) {  
   mService.setFocusedActivityLocked(next);  
}  
......  

ensureActivitiesVisibleLocked(null, 0);  
mService.mWindowManager.executeAppTransition();  

......  

    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/services/java/com/android/server/am/ActivityStack.java中。

ActivityStack类的成员函数completeResumeLocked主要是执行以下三个操作：

向ActivityManagerService服务所运行在的线程发送一个类型为IDLE_TIMEOUT_MSG的消息，这个消息将在IDLE_TIMEOUT毫秒后处理。这个类型为IDLE_TIMEOUT_MSG实际是用来监控WindowManagerService服务能否在IDLE_TIMEOUT毫秒之内，完成参数next所描述的Activity组件的切换操作，并且将它的窗口显示出来。如果能够到的话，WindowManagerService服务就会通知ActivityManagerService服务，然后ActivityManagerService服务就会执行一些清理工作，例如，将那些已经处于Stopped状态的Activity组件清理掉。如果不能够做到的话，那么ActivityStack类的成员函数completeResumeLocked也需要保证在IDLE_TIMEOUT毫秒之后，ActivityManagerService服务能够执行上述的清理工作。

如果当前正在处理的ActivityStack对象描述的是系统当前所使用的Activity组件堆栈，即ActivityStack类的成员变量mMainStack的值等于true，那么就会调用成员变量mService所指向的一个ActivityManagerService对象的成员函数setFocusedActivityLocked来将参数next所描述的Activity组件设置为系统当前获得焦点的Activity组件。

调用ActivityStack类的成员函数ensureActivitiesVisibleLocked来从上下到检查哪些Activity组件是需要设置为可见的，哪些Activity组件是需要设置为不可见的。

调用成员变量mService所指向的一个ActivityManagerService对象的成员变量mWindowManager所描述的一个WindowManagerService对象的成员函数executeAppTransition来通知WindowManagerService服务执行在前面Step 2所准备好的Activity组件切换操作。

我们主要关注第3点和第4点的操作，因此，接下来我们先分析ActivityStack类的成员函数ensureActivitiesVisibleLocked的实现，然后再分析WindowManagerService类的成员函数executeAppTransition的实现。

### Step 6. ActivityStack.ensureActivitiesVisibleLocked

```java
public class ActivityStack {  
    ......  

    final void ensureActivitiesVisibleLocked(ActivityRecord starting,  
   int configChanges) {  
ActivityRecord r = topRunningActivityLocked(null);  
if (r != null) {  
   ensureActivitiesVisibleLocked(r, starting, null, configChanges);  
}  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/services/java/com/android/server/am/ActivityStack.java。

ActivityStack类有两个重载版本的成员函数ensureActivitiesVisibleLocked，其中，两个参数版本的成员函数是通过调用四个版本的成员函数来实现的，因此，接下来我们主要分析ActivityStack类四个参数版本的成员函数ensureActivitiesVisibleLocked的实现。

ActivityStack类四个参数版本的成员函数ensureActivitiesVisibleLocked主要就是从上下到检查哪些Activity组件是需要设置为可见的，哪些Activity组件是需要设置为不可见的，我们分段来阅读：

```java
public class ActivityStack {  
    ......  

    final void ensureActivitiesVisibleLocked(ActivityRecord top,  
   ActivityRecord starting, String onlyThisProcess, int configChanges) {  
......  

// If the top activity is not fullscreen, then we need to  
// make sure any activities under it are now visible.  
final int count = mHistory.size();  
int i = count-1;  
while (mHistory.get(i) != top) {  
   i--;  
}  
```
参数top描述的是位于Activity组件堆栈顶端的Activity组件，而参数starting描述的是ActivityManagerService服务当前正在启动的Activity组件。另外，参数onlyThisProcess描述的是是否只对运行在进程名称等于它的Activity组件进行处理，如果它的值等于null，那么就表示对所有的Activity组件都进行处理。

这段代码首先找到参数top所描述的一个Activity组件在Activity组件堆栈的位置i，以便接下来可以从这个位置开始向下检查哪些Activity组件是需要设置为可见的，哪些Activity组件是需要设置为不可见的。

我们继续往下阅读代码：

```java
ActivityRecord r;  
boolean behindFullscreen = false;  
for (; i>=0; i--) {  
    r = (ActivityRecord)mHistory.get(i);  
    ......  
    if (r.finishing) {  
continue;  
    }  

    ......  

    if (r.app == null || r.app.thread == null) {  
if (onlyThisProcess == null  
       || onlyThisProcess.equals(r.processName)) {  
   // This activity needs to be visible, but isn't even  
   // running...  get it started, but don't resume it  
   // at this point.  
   ......  
   if (!r.visible) {  
       ......  
       mService.mWindowManager.setAppVisibility(r, true);  
   }  
   if (r != starting) {  
       startSpecificActivityLocked(r, false, false);  
   }  
}  

    } else if (r.visible) {  
// If this activity is already visible, then there is nothing  
// else to do here.  
......  

    } else if (onlyThisProcess == null) {  
// This activity is not currently visible, but is running.  
// Tell it to become visible.  
r.visible = true;  
if (r.state != ActivityState.RESUMED && r != starting) {  
   // If this activity is paused, tell it  
   // to now show its window.  
   ......  
   try {  
       mService.mWindowManager.setAppVisibility(r, true);  
       r.app.thread.scheduleWindowVisibility(r, true);  
       ......  
   } catch (Exception e) {  
       ......  
   }  
}  
    }  

    ......  

    if (r.fullscreen) {  
// At this point, nothing else needs to be shown  
......  
behindFullscreen = true;  
i--;  
break;  
    }  
}  
```
这段码的功能是从上到下找到第一个全屏显示的Activity组件，并且将该Activity组件以及位于该Activity组件上面的其它Activity组件的可见性设置为true。对这些Activity组件的可见性设置逻辑如下所示：

如果一个Activity组件处于正在结束的状态，即用来描述该Activity组件的一个ActivityRecord对象r的成员变量finishing的值等于true，那么跳过对该Activity组件的处理。

如果一个Activity组件所运行在的进程还没有启动起来，即用来描述该Activity组件的一个ActivityRecord对象r的成员变量app的值等于null，或者该成员变量app的值不等于null，但是它所指向的一个ProcessRecord对象的成员变量thread的值等于null，并且该Activity组件是需要处理的，即参数onlyThisProcess的值等于null，或者它的值不等于null，但是等于该Activity组件所运行在的进程的名称，那么就会做两个检查。第一个检查是看该Activity组件是否已经处于可见状态，即ActivityRecord对象r的成员变量visible的值是否等于true。如果不等于的话，那么就会通知WindowManagerService服务将该Activity组件的可见性设置为true。第二个检查是看该Activity组件是否就是参数sarting所描述的Activity组件。如果不是的话，那么就需要调用ActivityStack类的成员函数startSpecificActivityLocked来将该Activity组件所运行的进程启动起来。

如果一个Activity组件已经是可见的，即用来描述该Activity组件的一个ActivityRecord对象r的成员变量visible的值等于true，那么就什么也不用做。

如果一个Activity组件是不可见的，即用来描述该Activity组件的一个ActivityRecord对象r的成员变量visible的值等于true，并且该Activity组件是需要处理的，即参数onlyThisProcess的值等于null，那么就需要做两件事情。第一件事情就将ActivityRecord对象r的成员变量visible的值设置为true，以便表示该Activity组件是可见的。第二件事情是检查该Activity组件是否是不处于Resumed状态，并且它不是参数starting所描述的Activity组件。如果检查通过的话，那么就会先通知WindowManagerService服务将该Activity组件的可见性设置为true，然后再向该Activity组件所运在的进程发送一个通知，该Activity组件可见变为可见了。注意，参数starting所描述的Activity组件在前面的Step 3中已经被设置为可见的了，所以这里不需要重复将它设置为可见。

这段代码是如何知道一个Activity组件是不是全屏显示的呢？这是通过检查一个对应的ActivityRecord对象的成员变量fullscreen来判断的。如果这个成员变量的值等于true，那么就说明对应的Activity组件是全屏显示的。

我们继续往下阅读最后一段代码：

```java
// Now for any activities that aren't visible to the user, make  
// sure they no longer are keeping the screen frozen.  
while (i >= 0) {  
   r = (ActivityRecord)mHistory.get(i);  
   ......  
   if (!r.finishing) {  
       if (behindFullscreen) {  
           if (r.visible) {  
               ......  
               r.visible = false;  
               try {  
                   mService.mWindowManager.setAppVisibility(r, false);  
                   if ((r.state == ActivityState.STOPPING  
                           || r.state == ActivityState.STOPPED)  
                           && r.app != null && r.app.thread != null) {  
                       ......  
                       r.app.thread.scheduleWindowVisibility(r, false);  
                   }  
               } catch (Exception e) {  
                   ......  
               }  
           }   
           ......  
       } else if (r.fullscreen) {  
           ......  
           behindFullscreen = true;  
       }  
   }  
   i--;  
}  
    }  

    ......  
}  
```
这段代码是将在前面找到的第一个全屏显示的Activity组件的下面的所有Activity组件的可见性设置为false。注意，只有那些当前不是处于正在结束状态的、并且是处于可见状态的Activity组件才需要处理，即那些对应的ActivityRecord对象的成员变量finishing和visible的值分别等于false和true的值Activity组件才进行处理。处理的逻辑如下所示：

将对应的ActivityRecord对象的成员变量visible的值设置为false。

通知WindowManagerService服务将该Activity组件的可见性设置为false。

向相应的进程发送一个通知，该Activity组件可见变为不可见了。

这一步执行完成之后，回到前面的Step 5中，即ActivityStack类的成员函数completeResumeLocked中，接下来它就会继续调用WindowManagerService类的成员函数executeAppTransition来执行在前面Step 2所准备好的Activity组件切换操作。

### Step 7. WindowManagerService.executeAppTransition

```java
public class WindowManagerService extends IWindowManager.Stub  
implements Watchdog.Monitor {  
    ......  

    public void executeAppTransition() {  
if (!checkCallingPermission(android.Manifest.permission.MANAGE_APP_TOKENS,  
       "executeAppTransition()")) {  
   throw new SecurityException("Requires MANAGE_APP_TOKENS permission");  
}  

synchronized(mWindowMap) {  
   ......  
   if (mNextAppTransition != WindowManagerPolicy.TRANSIT_UNSET) {  
       mAppTransitionReady = true;  
       final long origId = Binder.clearCallingIdentity();  
       performLayoutAndPlaceSurfacesLocked();  
       Binder.restoreCallingIdentity(origId);  
   }  
}  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/services/java/com/android/server/WindowManagerService.java中。

调用WindowManagerService类的成员函数executeAppTransition来执行之前准备好的Activity组件切换操作需要具有android.Manifest.permission.MANAGE_APP_TOKENS权限，否则的话，WindowManagerService类的成员函数setAppVisibility就会抛出一个类型为SecurityException的异常。

注意，只有当之前已经准备好了一个Activity组件切换操作的情况下，WindowManagerService类的成员函数executeAppTransition才可以调用另外一个成员函数performLayoutAndPlaceSurfacesLocked来执行该Activity组件切换操作。从前面的Step 2可以知道，如果WindowManagerService类的成员变量mNextAppTransition的值不等于WindowManagerPolicy.TRANSIT_UNSET，那么就说明之前已经准备好了一个Activity组件切换操作。

在调用WindowManagerService类的成员函数performLayoutAndPlaceSurfacesLocked来执行一个Activity组件切换操作之前，WindowManagerService类的成员函数executeAppTransition还会将成员变量mAppTransitionReady的值设置为true，以便可以表示WindowManagerService服务已经就准备就绪执行一个Activity组件切换操作了。

接下来，我们就继续分析WindowManagerService类的成员函数performLayoutAndPlaceSurfacesLocked的实现，以便可以了解WindowManagerService服务是如何执行一个Activity组件切换操作的。

### Step 8. WindowManagerService.performLayoutAndPlaceSurfacesLocked

这一步可以参考前面[Android窗口管理服务WindowManagerService计算Activity窗口大小的过程分析](http://blog.csdn.net/luoshengyang/article/details/8479101)一文的Step 5，它主要就是做三件事情：

检查是否有窗口资源等待回收。如果有的话，那么就调用WindowManagerService类的成员函数removeWindowInnerLocked来将它们从系统中移除，以便可以回收它们所占用的资源。

调用WindowManagerService类的成员函数performLayoutAndPlaceSurfacesLockedInner来刷新系统UI，其中就包含了执行在前面Step 2中所准备的Activity组件切换操作。

在第2步的执行过程中，如果又有一些窗口要被移除，那么又会继续调用WindowManagerService类的成员函数removeWindowInnerLocked来将它们从系统中移除。移除完成之后，又会再次递归调用WindowManagerService类的成员函数performLayoutAndPlaceSurfacesLocked来刷新系统UI。

我们主要关注第2步，即WindowManagerService类的成员函数performLayoutAndPlaceSurfacesLockedInner的实现，以便可以了解Activity组件的切换过程。

### Step 9. WindowManagerService.performLayoutAndPlaceSurfacesLockedInner

```java
public class WindowManagerService extends IWindowManager.Stub    
implements Watchdog.Monitor {    
    ......    
    
    private final void performLayoutAndPlaceSurfacesLockedInner(    
   boolean recoveringMemory) {    
......    
    
Surface.openTransaction();    
......    
    
try {    
   ......    
   int repeats = 0;    
   int changes = 0;    
       
   do {    
       repeats++;    
       if (repeats > 6) {    
           ......    
           break;    
       }    
    
       // 1. 计算各个窗口的大小，以便让各个窗口可以对自己的UI元素进行布局  
       // FIRST LOOP: Perform a layout, if needed.    
       if (repeats < 4) {    
           changes = performLayoutLockedInner();    
           if (changes != 0) {    
               continue;    
           }    
       } else {    
           Slog.w(TAG, "Layout repeat skipped after too many iterations");    
           changes = 0;    
       }    

       // 2. 计算各个窗口接下来要执行的动画  
       // Update animations of all applications, including those  
       // associated with exiting/removed apps  
       ......  
    
       // 3. 执行各个窗口的动画  
       // SECOND LOOP: Execute animations and update visibility of windows.    
       ......    

       // 4. 检查当前是否需要执行Activity组件切换操作  
       // If we are ready to perform an app transition, check through  
       // all of the app tokens to be shown and see if they are ready  
       // to go.  
       if (mAppTransitionReady) {  
           ......  
       }  

       ......  
           
   } while (changes != 0);    
    
   // 5. 更新各个窗口的绘图表面  
   // THIRD LOOP: Update the surfaces of all windows.    
   ......    
} catch (RuntimeException e) {    
   ......    
}    
    
......    
    
Surface.closeTransaction();    
    
......    
    
// 6. 销毁那些需要销毁的绘图表面  
// Destroy the surface of any windows that are no longer visible.    
......    
    
// 7. 删除那些正在退出的类型为WindowToken的窗口令牌  
// Time to remove any exiting tokens?    
......    
    
// 8. 删除那些正在退出的类型为AppWindowToken的窗口令牌  
// Time to remove any exiting applications?    
......    

// 9. 检查是否需要再一次刷新系统UI  
if (needRelayout) {  
   requestAnimationLocked(0);  
} else if (animating) {  
   requestAnimationLocked(currentTime+(1000/60)-SystemClock.uptimeMillis());  
}  

......  
    }    
    
    ......    
}    
```
这个函数定义在文件frameworks/base/services/java/com/android/server/WindowManagerService.java中。

WindowManagerService类的成员函数performLayoutAndPlaceSurfacesLockedInner是整个WindowManagerService服务的核心，它的主要任务就是刷新系统的UI，执行的操作包括：

计算各个窗口的大小，以便让各个窗口可以对自己的UI元素进行布局。这一步主要是通过调用WindowManagerService类的成员函数performLayoutLockedInner来实现的，具体可以参考前面[Android窗口管理服务WindowManagerService计算Activity窗口大小的过程分析](http://blog.csdn.net/luoshengyang/article/details/8479101)一文。

计算各个窗口接下来要执行的动画。

执行各个窗口的动画。

检查当前是否需要执行Activity组件切换操作，即检查WindowManagerService类的成员变量mAppTransitionReady的值是否等于true。如果等于的话，那么接下来就会执行这个Activity组件切换操作，实际上就是给正在切换的Activity组件应用一个切换动画。

更新各个窗口的绘图表面。

销毁那些需要销毁的绘图表面。

删除那些正在退出的类型为WindowToken的窗口令牌。

删除那些正在退出的类型为AppWindowToken的窗口令牌。

检查是否需要再一次刷新系统UI。前面的操作可能会导致窗口堆栈或者窗口布局发生变化，例如，前面第6步销毁的绘图表面可能是当前正在显示的输入法窗口或者壁纸窗口的绘图表面，或者前面第3步表明窗口的动画还没有结束，这时候都会要求WindowManagerService服务马上再次刷新系统UI，以便可以反映出窗口堆栈或者窗口布局的变化，例如，可以持续地将窗口的动画显示完毕。如果需要再一次刷新系统UI，那么有可能是需要马上执行的，也有可能是过一会再执行的。如果是窗口堆栈或者窗口布局发生了变化的情况，那么变量needRelayout的值就会等于true，这时候就会以0为参数来调用WindowManagerService类的成员函数requestAnimationLocked，以便可以马上再次刷新系统UI。如果是窗口的动画还没有结束，那么变量animating的值就会等于true，这时候就会以一个非0参数来调用WindowManagerService类的成员函数requestAnimationLocked，请求过一段时间后再刷新系统UI。这个非0参数的值大约就等于1000/60，其中，分子表示1000毫秒，也就是说，在1/60秒后再刷新系统UI，意思就是以60帧每秒（fps）的速度来显示窗口的动画。

注意，在上述9个操作中，第1步到第4步是放在一个do...while循环中执行的，这是因为第2步到第4步的操作会导致导致窗口堆栈或者窗口布局发生变化，例如，有些窗口本来是不可见的，现在变成可见了，这时候就会要求重新计算各个窗口的大小，以及让各个窗口重新布局自己的UI元素，即重新执行第1步的操作。然而，这些操作不能无休止地执行，否则的话，系统的UI就会死在那里不动了，因此，这个do...while循环最多只能执行7次。此外，在这最多7次的循环中，只有前4次循环才会重新各个窗口的大小。这些都是为了尽早地结束上述的do...while的，以便系统的UI可以尽快地刷新出来。

另外一个还有一个地方是需要注意的，即第1步到第5步的操作是放在一个事务中执行，即在Surface.openTransaction和Surface.closeTransaction之间执行，这意味着当Surface.closeTransaction执行完成之后，WindowManagerService服务才会通知SurfaceFlinger服务将系统的UI渲染到帧缓冲区（FB）中去，也就是说，在Surface.closeTransaction执行之后，我们才会看到系统的新UI。实际上，前面第1步到第5步的操作都只是修改了各个窗口的绘制表面的状态，这些修改都只是反映在图形缓冲区中，而不是反映在帧缓冲区中的。

在本文中，我们主要关注第4步的操作，即与Activity组件切换相关的操作，我们分段来阅读这部分代码：

```java
// If we are ready to perform an app transition, check through  
// all of the app tokens to be shown and see if they are ready  
// to go.  
if (mAppTransitionReady) {  
    int NN = mOpeningApps.size();  
    boolean goodToGo = true;  
    ......  
    if (!mDisplayFrozen && !mAppTransitionTimeout) {  
// If the display isn't frozen, wait to do anything until  
// all of the apps are ready.  Otherwise just go because  
// we'll unfreeze the display when everyone is ready.  
for (i=0; i<NN && goodToGo; i++) {  
   AppWindowToken wtoken = mOpeningApps.get(i);  
   ......  
   if (!wtoken.allDrawn && !wtoken.startingDisplayed  
           && !wtoken.startingMoved) {  
       goodToGo = false;  
   }  
}  
    }  
```
从前面的Step 7可以知道，只有当WindowManagerService类的成员变量mAppTransitionReady的值等于true的时候，才说明WindowManagerService服务可以执行一个Activity组件切换操作。不过在执行这个切换操作之前，要求所有正在打开的Activity组件的UI都已经绘制完毕。因为如果这些正在打开的Activity组件的UI还没有绘制完成的话，就无法给它们应用一个切换动画。

从前面的Step 4可以知道，系统当前所有正在打开的Activity组件都保存在WindowManagerService类的成员变量mOpeningApps所描述的一个ArrayList中，因此，这段代码就通过遍历这个ArrayList来检查每一个正在打开的Activity组件的UI是否已经绘制完成，即检查对应的AppWindowToken对象的成员变量allDraw的值是否不等于true。如果不等于true的话，就说明还没有绘制完成。此外，还要求这些正在打开的Activity组件的启动窗口已经显示结束，或者已经转移给其它的Activity组件，即要求查对应的AppWindowToken对象的成员变量startingDisplayed和startingMoved的值均等于true。只要其中的一个正在打开的Activity组件不能满足上述条件，那么变量goodToGo的值就会等于false，表示这时候还不能执行Activity组件操作。

注意，上述检查是在屏幕不是处于冻结状态或者前面准备好的Activity组件切换操作尚示超时的情况下进行，即在WindowManagerService类的成员变量mDisplayFrozen和mAppTransitionTimeout的值均等于false的情况下进行的，这意味着：

如果屏幕处于冻结状态，就要马上执行前面准备好的Activity组件切换操作，因为等到屏幕解冻的时候，上述的三个条件都会得到满足。

如果者前面准备好的Activity组件切换操作已经超时，那么就说明不需要再等待了，而应该立即执行该Activity组件切换操作。

我们假设上述检查都能通过，即最终得到的变量goodToGo的值等于true，我们继续往下阅读代码：

```java
if (goodToGo) {  
    ......  
    int transit = mNextAppTransition;  
    if (mSkipAppTransitionAnimation) {  
transit = WindowManagerPolicy.TRANSIT_UNSET;  
    }  
    mNextAppTransition = WindowManagerPolicy.TRANSIT_UNSET;  
    mAppTransitionReady = false;  
    mAppTransitionRunning = true;  
    mAppTransitionTimeout = false;  
    mStartingIconInTransition = false;  
    mSkipAppTransitionAnimation = false;  

    mH.removeMessages(H.APP_TRANSITION_TIMEOUT);  
```
由于接下来就要执行Activity组件切换操作了，因此，这段代码就先修改与WindowManagerService服务的Activity组件切换操作相关的状态。

首先是将WindowManagerService类的成员变量mNextAppTransition的值保存在变量transit中，并且将该成员变量的值重置为WindowManagerPolicy.TRANSIT_UNSET，以便WindowManagerService服务接下来可以准备一个新的Activity组件切换操作。注意，如果WindowManagerService类的成员变量mSkipAppTransitionAnimation的值等于true，那么就意味着要跳过此次Activity组件切换操作，即将前面得到的变量transit的值设置为WindowManagerPolicy.TRANSIT_UNSET。一般来说，如果一个Activity组件还在显示启动窗口的过程中，又有另外一个Activity组件被启动，并且这个Activity组件也要求显示启动窗口，那么当前正在显示启动窗口的Activity组件就会跳过之前为它所准备的切换操作，这是为了让后面那个启动的Activity组件尽快地显示出来。

接着还会分别设置WindowManagerService类的以下五个成员变量的值，其中：

mAppTransitionReady的值被重置为false，表示WindowManagerService服务接下来可以接受执行下一个Activity组件切换操作；

mAppTransitionRunning的值被设置为true，表示WindowManagerService服务目前正处于显示Activity组件切换动画的过程中；

mAppTransitionTimeout的值被设置为false，这是由于前面准备好的Activity组件切换操作已经得到执行了，所以就不存在超时问题了，同时后面还会通过调用WindowManagerService类的成同变量mH所指向一个H对象的成员函数removeMessages来删除之前发送到WindowManagerService服务所运行在的线程的消息队列中的一个类型为APP_TRANSITION_TIMEOUT的消息；

mStartingIconInTransition的值被重置为false，因为此时那些正在打开的Activity组件的启动窗口都已经结束显示；

mSkipAppTransitionAnimation的值被重置为false，因为前面已经对跳过切换动画的情况进行过处理了。

我们继续向下阅读代码：

```java
// If there are applications waiting to come to the  
// top of the stack, now is the time to move their windows.  
// (Note that we don't do apps going to the bottom  
// here -- we want to keep their windows in the old  
// Z-order until the animation completes.)  
if (mToTopApps.size() > 0) {  
    NN = mAppTokens.size();  
    for (i=0; i<NN; i++) {  
AppWindowToken wtoken = mAppTokens.get(i);  
if (wtoken.sendingToTop) {  
   wtoken.sendingToTop = false;  
   moveAppWindowsLocked(wtoken, NN, false);  
}  
    }  
    mToTopApps.clear();  
}  
```
这段代码对那些被请求移动至窗口堆栈顶端的Activity组件进行处理。这些被请求移动至窗口堆栈顶端的Activity组件会被保存在WindowManagerService类的成员变量mToTopApps所描述的一个ArrayList中。同时，用来描述这些被请求移动至窗口堆栈顶端的Activity组件的AppWindowToken对象的成员变量sendingToTop的值也会等于true。对于这些Activity组件，是通过调用WindowManagerService类的成员函数moveAppWindowsLocked来将它们移动到窗口堆栈顶端的。注意，在移动之前，还会将对应的AppWindowToken对象的成员变量sendingToTop的值设置为false。

除了可以请求WindowManagerService服务将指定的Activity组件移动到窗口堆栈顶端之外，还可以请求WindowManagerService服务将指定的Activity组件移动到窗口堆栈底端。这些被请求移动至窗口堆栈底端的Activity组件是保存在WindowManagerService类的成员变量mToBottomApps所描述的一个ArrayList中。不过，这些Activity组件要等到切换动画显示完成之后，才会被移动至窗口堆栈底端。

将那些被请求移动至窗口堆栈顶端的Activity组件都移动到窗口堆栈顶端之后，就可以将WindowManagerService类的成员变量mToTopApps所描述的一个ArrayList清空了。

我们继续往下阅读代码：

```java
WindowState oldWallpaper = mWallpaperTarget;  

adjustWallpaperWindowsLocked();  
......  

// The top-most window will supply the layout params,  
// and we will determine it below.  
LayoutParams animLp = null;  
AppWindowToken animToken = null;  
int bestAnimLayer = -1;  

......  
int foundWallpapers = 0;  
// Do a first pass through the tokens for two  
// things:  
// (1) Determine if both the closing and opening  
// app token sets are wallpaper targets, in which  
// case special animations are needed  
// (since the wallpaper needs to stay static  
// behind them).  
// (2) Find the layout params of the top-most  
// application window in the tokens, which is  
// what will control the animation theme.  
final int NC = mClosingApps.size();  
NN = NC + mOpeningApps.size();  
for (i=0; i<NN; i++) {  
    AppWindowToken wtoken;  
    int mode;  
    if (i < NC) {  
wtoken = mClosingApps.get(i);  
mode = 1;  
    } else {  
wtoken = mOpeningApps.get(i-NC);  
mode = 2;  
    }  
    if (mLowerWallpaperTarget != null) {  
if (mLowerWallpaperTarget.mAppToken == wtoken  
       || mUpperWallpaperTarget.mAppToken == wtoken) {  
   foundWallpapers |= mode;  
}  
    }  
    if (wtoken.appFullscreen) {  
WindowState ws = wtoken.findMainWindow();  
if (ws != null) {  
   // If this is a compatibility mode  
   // window, we will always use its anim.  
   if ((ws.mAttrs.flags&FLAG_COMPATIBLE_WINDOW) != 0) {  
       animLp = ws.mAttrs;  
       animToken = ws.mAppToken;  
       bestAnimLayer = Integer.MAX_VALUE;  
   } else if (ws.mLayer > bestAnimLayer) {  
       animLp = ws.mAttrs;  
       animToken = ws.mAppToken;  
       bestAnimLayer = ws.mLayer;  
   }  
}  
    }  
}  
```
这段代码用来检查那些参与切换操作的Activity组件的窗口是否与壁纸窗口有关。如果有关的话，接下来要执行的切换动画就需要修改为与壁纸相关的类型。

要检查那些参与切换操作的Activity组件的窗口是否与壁纸窗口有关，首先需要确保壁纸窗口目前是位于那些需要显示壁纸的窗口的下面的，这是通过调用WindowManagerService类的成员函数adjustWallpaperWindowsLocked来实现的。在调整壁纸窗口在窗口堆栈的位置之前，会首先将壁纸窗口的当前目标窗口保存在变量oldWallpaper中，这是因为接下来要执行的切换动画的类型与壁纸窗口当前是否具有目标窗口有关。这一点我们后面再分析。

参与切换操作的Activity组件可以划分为两类，一类是需要打开的，一类是需要关闭的，它们分别保存在WindowManagerService类的成员变量mOpeningApps和mClosingApps所描述的ArrayList中。因此，通过检查保存在这两个ArrayList中的Activity组件的窗口是否是需要显示壁纸的，就可以知道那些参与切换操作的Activity组件的窗口是否与壁纸窗口有关。

从前面[Android窗口管理服务WindowManagerService对壁纸窗口（Wallpaper Window）的管理分析](http://blog.csdn.net/luoshengyang/article/details/8550820)一文可以知道，在调整壁纸窗口在窗口堆栈的位置的时候，如果刚好碰到系统在执行两个Activity组件的切换操作，并且这两个Activity组件都需要显示壁纸，那么Z轴位置较低的窗口就会保存在WindowManagerService类的成员变量mLowerWallpaperTarget中，而Z轴位置较高的窗口就会保存在WindowManagerService类的成员变量mUpperWallpaperTarget中。因此，这段代码就可以通过一个for循环来检查保存在WindowManagerService类的成员变量mOpeningApps和mClosingApps中那些AppWindowToken对象，如果它们刚好被WindowManagerService类的成员变量mLowerWallpaperTarget或者mUpperWallpaperTarget所描述的WindowState对象的成员变量mAppToken所引用，那么就说明那些参与切换操作的Activity组件的窗口是与壁纸窗口有关的。

最终得到的变量foundWallpapers的值就反映了那些参与切换操作的Activity组件的窗口是否与壁纸窗口有关，其中：

变量foundWallpapers的值等于0表示与壁纸窗口无关；

变量foundWallpapers的值等于1表示只有那些需要关闭的Activity组件与壁纸窗口无关；

变量foundWallpapers的值等于2表示只有那些需要打开的Activity组件与壁纸窗口无关；

变量foundWallpapers的值等于2表示需要打开和关闭的Activity组件均与壁纸窗口无关。

这段代码的for循环除了用来检查那些参与切换操作的Activity组件的窗口是否与壁纸窗口有关之外，还有另外一个重要的任务，那就是找到用来创建Activity组件切换动画的参数。用来创建Activity组件切换动画的参数是保存在一个类型为WindowManager.LayoutParams的对象中的，而这个类型为WindowManager.LayoutParams的对象是需要来自那些参与切换操作的Activity组件的窗口的。

我们知道，每一个窗口都具有一个WindowManager.LayoutParams对象，用来描述它的属性，因此，这段代码的for循环就是要从参与切换操作的Activity组件的窗口的WindowManager.LayoutParams对象中挑选出一个来创建切换动画，然后再将这个切换动画应用到每一个参与切换操作的Activity组件的窗口中去。这个被挑选中的窗口应用具有以下属性：

它所属的Activity组件是全屏显示的，即用来描述该Activity组件的AppWindowToken对象的成员变量appFullscreen的值等于true。

它是它所属的Activity组件的主窗口。一个Activity组件的主窗口可以通过调用对应的AppWindowToken对象的成员函数findMainWindow来获得。

它是所有候选窗口中Z轴位置最高的，即用来描述它的一个WindowState对象的成员变量mLayer的值是最大的，或者它是一个兼容窗口，即用来描述它的一个WindowState对象的成员变量mAttrs所指向的一个WindowManager.LayoutParams对象的成员变量flags的值的FLAG_COMPATIBLE_WINDOW位等于1。

一旦找到了这样的窗口，这段代码中的for循环就会将用来描述它的属性的一个WindowManager.LayoutParams对象保存在变量animLp中，并且会将用来描述它所属的Activity组件的一个AppWindowToken对象保存在变量animToken中。

我们继续往下阅读代码：

```java
if (foundWallpapers == 3) {  
    ......  
    switch (transit) {  
case WindowManagerPolicy.TRANSIT_ACTIVITY_OPEN:  
case WindowManagerPolicy.TRANSIT_TASK_OPEN:  
case WindowManagerPolicy.TRANSIT_TASK_TO_FRONT:  
   transit = WindowManagerPolicy.TRANSIT_WALLPAPER_INTRA_OPEN;  
   break;  
case WindowManagerPolicy.TRANSIT_ACTIVITY_CLOSE:  
case WindowManagerPolicy.TRANSIT_TASK_CLOSE:  
case WindowManagerPolicy.TRANSIT_TASK_TO_BACK:  
   transit = WindowManagerPolicy.TRANSIT_WALLPAPER_INTRA_CLOSE;  
   break;  
    }  
    ......  
} else if (oldWallpaper != null) {  
    // We are transitioning from an activity with  
    // a wallpaper to one without.  
    transit = WindowManagerPolicy.TRANSIT_WALLPAPER_CLOSE;  
    ......  
} else if (mWallpaperTarget != null) {  
    // We are transitioning from an activity without  
    // a wallpaper to now showing the wallpaper  
    transit = WindowManagerPolicy.TRANSIT_WALLPAPER_OPEN;  
    ......  
}  

if ((transit&WindowManagerPolicy.TRANSIT_ENTER_MASK) != 0) {  
    mLastEnterAnimToken = animToken;  
    mLastEnterAnimParams = animLp;  
} else if (mLastEnterAnimParams != null) {  
    animLp = mLastEnterAnimParams;  
    mLastEnterAnimToken = null;  
    mLastEnterAnimParams = null;  
}  

// If all closing windows are obscured, then there is  
// no need to do an animation.  This is the case, for  
// example, when this transition is being done behind  
// the lock screen.  
if (!mPolicy.allowAppAnimationsLw()) {  
    animLp = null;  
}  
```
这段代码用来最终确定接下来要设置的切换动画的类型。

如果变量foundWallpapers的值等于3，那么就说明正在打开和正在关闭的Activity组件的窗口均与壁纸窗口有关，这时候就会按照以下规则来修改之前所设置的切换动画的类型：

如果之前所设置的切换动画是与打开操作相关的，即变量transit的值等于WindowManagerPolicy.TRANSIT_ACTIVITY_OPEN、WindowManagerPolicy.TRANSIT_TASK_OPEN或者WindowManagerPolicy.TRANSIT_TASK_TO_FRONT，那么就会将实际的切换动画的类型设置为WindowManagerPolicy.TRANSIT_WALLPAPER_INTRA_OPEN。这种类型的切换动画是与打开壁纸窗口相关的。

如果之前所设置的切换动画是与关闭操作相关的，即变量transit的值等于WindowManagerPolicy.TRANSIT_ACTIVITY_CLOSE、WindowManagerPolicy.TRANSIT_TASK_CLOSE或者WindowManagerPolicy.TRANSIT_TASK_TO_BACK，那么就会将实际的切换动画的类型设置为WindowManagerPolicy.TRANSIT_WALLPAPER_INTRA_CLOSE。这种类型的切换动画是与关闭壁纸窗口相关的。

如果变量foundWallpapers的值不等于3，但是变量oldWallpaper的值不等于null，那么就说明当前正在执行的Activity组件切换操作是从一个需要显示壁纸的Activity组件切换到一个不需要显示壁纸的Activity组件上去，这时候就会将实际的切换动画的类型设置为WindowManagerPolicy.TRANSIT_WALLPAPER_CLOSE。这种类型的切换动画是与关闭壁纸窗口相关的。

如果变量foundWallpapers的值不等于3，并且变量oldWallpaper的值等于null，但是WindowManagerService类的成员变量mWallpaperTarget的值不等于null，那么就说明当前正在执行的Activity组件切换操作是从一个不需要显示壁纸的Activity组件切换到一个需要显示壁纸的Activity组件上去，这时候就会将实际的切换动画的类型设置为WindowManagerPolicy.TRANSIT_WALLPAPER_OPEN。这种类型的切换动画是与打开壁纸窗口相关的。

最终得到的切换动画的类型就保存在变量transit中。从上面的逻辑可以知道，最终得到的切换动画的类型可能是与关闭操作相关的，但是我们需要的是一个与打开操作相关的切换动画。因此，如果最终得到的切换动画的类型是与关闭操作相关的，那么我们就使用上一次使用的Activity组件切换动画。用来创建上一次使用的Activity组件切换动画的WindowManager.LayoutParams对象和AppWindowToken分别保存在WindowManagerService类的成员变量mLastEnterAnimParams和mLastEnterAnimToken中。

通过检查变量transit的值的TRANSIT_ENTER_MASK位是否不等于0，就可以知道最终得到的切换动画的类型是否是与打开操作相关的。如果是与打开操作相关的，那么就会将前面获得的变量animToken和animLp的值保存在WindowManagerService类的成员变量mLastEnterAnimToken和mLastEnterAnimParams中，以便以后可以重复使用。如果是与关闭操作相关的，那么就会将WindowManagerService类的成员变量mLastEnterAnimParams的值保存在变量animLp中，以及将WindowManagerService类的成员变量mLastEnterAnimToken和mLastEnterAnimParams重置为null，以便表示上一次使用的Activity组件切换动画不是与打开操作相关的。

此时，如果我们所获得的变量animLp的值不等于null，那么它所指向的一个WindowManager.LayoutParams对象就可以用来创建一个Activity组件切换动画，但是如果窗口管理策略类表明此时不需要显示Activity组件切换动画，即WindowManagerService类的成员变量mPolicy所指向的一个PhoneWindowManager对象的成员函数allowAppAnimationsLw的返回值等于false，那么前面得的变量animLp的值就会被重置为null。当正在执行的Activity组件切换操作是发生在锁屏窗口的后面时，就会出现这种情况。

我们继续往下阅读代码：

```java
NN = mOpeningApps.size();  
for (i=0; i<NN; i++) {  
    AppWindowToken wtoken = mOpeningApps.get(i);  
    ......  
    wtoken.reportedVisible = false;  
    wtoken.inPendingTransaction = false;  
    wtoken.animation = null;  
    setTokenVisibilityLocked(wtoken, animLp, true, transit, false);  
    wtoken.updateReportedVisibilityLocked();  
    wtoken.waitingToShow = false;  
    wtoken.showAllWindowsLocked();  
}  
NN = mClosingApps.size();  
for (i=0; i<NN; i++) {  
    AppWindowToken wtoken = mClosingApps.get(i);  
    ......  
    wtoken.inPendingTransaction = false;  
    wtoken.animation = null;  
    setTokenVisibilityLocked(wtoken, animLp, false, transit, false);  
    wtoken.updateReportedVisibilityLocked();  
    wtoken.waitingToHide = false;  
    // Force the allDrawn flag, because we want to start  
    // this guy's animations regardless of whether it's  
    // gotten drawn.  
    wtoken.allDrawn = true;  
}  
......  

mOpeningApps.clear();  
mClosingApps.clear();  
```
这段代码将给所有参与了切换操作的Activity组件设置一个切换动画，而这个动画就来自前面所获得的变量animLp所指向的一个WindowManager.LayoutParams对象。由于参与了切换操作的Activity组件可以划分为两类，即一类是正在打开的，一类是正在打闭的，因此，我们就分别讨论这两种类型的Activity组件的切换动画的设置过程。

对于正在打开的Activity组件，它们的切换动画的设置过程如下所示：

找到对应的AppWindowToken对象；

将对应的AppWindowToken对象的成员变量reportedVisible的值设置为false，表示还没有向ActivityManagerService服务报告过正在打开的Activity组件的可见性；

将对应的AppWindowToken对象的成员变量inPendingTransaction的值设置为false，表示正在打开的Activity组件不是处于等待执行切换操作的状态了；

将对应的AppWindowToken对象的成员变量animation的值设置为null，因为接下来要重新这个成员变量的值来描述正在打开的Activity组件的切换动画；

调用WindowManagerService类的成员函数setTokenVisibilityLocked将正在打开的Activity组件的可见性设置为true，并且给正在打开的Activity组件设置一个切换动画，这个切换动画会保存在对应的AppWindowToken对象的成员变量animation中；

调用对应的AppWindowToken对象的成员函数updateReportedVisibilityLocked向ActivityManagerService服务报告正在打开的Activity组件的可见性；

将对应的AppWindowToken对象的成员变量waitingToShow的值设置为false，表示正在打开的Activity组件的窗口不是处于等待显示的状态了；

调用对应的AppWindowToken对象的成员函数showAllWindowsLocked通知SurfaceFlinger服务将正在打开的Activity组件的窗口设置为可见的。

对于正在关闭的Activity组件，它们的切换动画的设置过程如下所示：

找到对应的AppWindowToken对象；

将对应的AppWindowToken对象的成员变量inPendingTransaction的值设置为false，表示正在关闭的Activity组件不是处于等待执行切换操作的状态了；

将对应的AppWindowToken对象的成员变量animation的值设置为null，因为接下来要重新这个成员变量的值来描述正在关闭的Activity组件的切换动画；

调用WindowManagerService类的成员函数setTokenVisibilityLocked将正在关闭的Activity组件的可见性设置为true，并且给正在关闭的Activity组件设置一个切换动画，这个切换动画会保存在对应的AppWindowToken对象的成员变量animation中；

调用对应的AppWindowToken对象的成员函数updateReportedVisibilityLocked向ActivityManagerService服务报告正在关闭的Activity组件的可见性；

将对应的AppWindowToken对象的成员变量waitingToHide的值设置为false，表示正在关闭的Activity组件的窗口不是处于等待隐藏的状态了；

将对应的AppWindowToken对象的成员变量allDrawn的值设置为true，这样就可以使得前面所设置的切换动画得以执行。

给所有参与了切换操作的Activity组件都设置了一个切换动画之后，接下来就可以将WindowManagerService类的成员变量mOpeningApps和mClosingApps所描述的两个ArrayList清空了。

我们继续向下阅读最后一段代码：

```java
    // This has changed the visibility of windows, so perform  
    // a new layout to get them all up-to-date.  
    changes |= PhoneWindowManager.FINISH_LAYOUT_REDO_LAYOUT;  
    ......  
}  
```
由于前面的操作已经导致那些正在打开的Activity组件的窗口由不可见变为可见，即相当于是导致窗口堆栈发生了变化，这时候就需要重新计算各个窗口的大小，以便让各个窗口对自己的UI元素进行重新布局，这是通过将变量changes的值的PhoneWindowManager.FINISH_LAYOUT_REDO_LAYOUT位设置为1来实现的。

从前面的分析可以知道，一旦变量changes的值不等于0，WindowManagerService类的成员函数performLayoutAndPlaceSurfacesLockedInner开始的那个do...while循环就会重复执行，也就是会重复执行以下三个操作：

计算各个窗口的大小，以便让各个窗口可以对自己的UI元素进行布局。

计算各个窗口接下来要执行的动画。

执行各个窗口的动画。

