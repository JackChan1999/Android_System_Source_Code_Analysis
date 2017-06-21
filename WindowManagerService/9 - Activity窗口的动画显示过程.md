在前一文中，我们分析了Activity组件的切换过程。从这个过程可以知道，所有参与切换操作的窗口都会被设置切换动画。事实上，一个窗口在打开（关闭）的过程中，除了可能会设置切换动画之外，它本身也可能会设置有进入（退出）动画。再进一步地，如果一个窗口是附加在另外一个窗口之上的，那么被附加窗口所设置的动画也会同时传递给该窗口。本文就详细分析WindowManagerService服务显示窗口动画的原理。

《[Android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

在[android](http://lib.csdn.net/base/android)系统中，窗口动画的本质就是对原始窗口施加一个变换（Transformation）。在线性数学中，对物体的形状进行变换是通过乘以一个矩阵（Matrix）来实现，目的就是对物体进行偏移、旋转、缩放、切变、反射和投影等。因此，给窗口设置动画实际上就给窗口设置一个变换矩阵（Transformation Matrix）。

如前所述，一个窗口在打开（关闭）的过程，可能会被设置三个动画，它们分别是窗口本身所设置的进入（退出）动画（Self Transformation）、从被附加窗口传递过来的动画（Attached Transformation），以及宿主Activity组件传递过来的切换动画（App Transformation）。这三个Transformation组合在一起形成一个变换矩阵，以60fps的速度应用在窗口的原始形状之上，完成窗口的动画过程，如图1所示。

![img](http://img.my.csdn.net/uploads/201302/26/1361812199_5055.jpg)

图1 窗口的动画显示过程

从上面的分析可以知道，窗口的变换矩阵是应用在窗口的原始位置和大小之上的，因此，在显示窗口的动画之前，除了要给窗口设置变换矩阵之外，还要计算好窗口的原始位置和大小，以及布局和绘制好窗口的UI。在前面[Android窗口管理服务WindowManagerService计算Activity窗口大小的过程分析](http://blog.csdn.net/luoshengyang/article/details/8479101)和[Android应用程序窗口（Activity）的测量（Measure）、布局（Layout）和绘制（Draw）过程分析](http://blog.csdn.net/luoshengyang/article/details/8372924)这两篇文章中，我们已经分析过窗口的位置和大小计算过程以及窗口UI的布局和绘制过程了，本文主要关注窗口动画的设置、合成和显示过程。这三个过程通过以下四个部分的内容来描述：

1. 窗口动画的设置过程
2. 窗口动画的显示框架
3. 窗口动画的推进过程
4. 窗口动画的合成过程

其中，窗口动画的设置过程包括上述三个动画的设置过程，窗口动画的推进过程是指定动画的一步一步地迁移的过程，窗口动画的合成过程是指上述三个动画组合成一个变换矩阵的过程，后两个过程包含在了窗口动画的显示框架中。

## 窗口动画的设置过程

窗口被设置的动画虽然可以达到三个，但是这三个动画可以归结为两类，一类是普通动画，例如，窗口在打开过程中被设置的进入动画和在关闭过程中被设置的退出动画，另一类是切换动画。其中，Self Transformation和Attached Transformation都是属于普通动画，而App Transformation属于切换动画。接下来我们就分别分析这两种类型的动画的设置过程。

### 普通动画的设置过程

从前面[Android窗口管理服务WindowManagerService显示Activity组件的启动窗口（Starting Window）的过程分析](http://blog.csdn.net/luoshengyang/article/details/8577789)一文可以知道，窗口在打开的过程中，是通过调用WindowState类的成员函数performShowLocked来实现的，如下所示：

```java
public class WindowManagerService extends IWindowManager.Stub    
implements Watchdog.Monitor {    
    ......    
    
    private final class WindowState implements WindowManagerPolicy.WindowState {    
......    
    
boolean performShowLocked() {    
    ......    
    
    if (mReadyToShow && isReadyForDisplay()) {    
        ......    
    
        if (!showSurfaceRobustlyLocked(this)) {    
            return false;    
        }    

        ......    
    
        applyEnterAnimationLocked(this);    
    
        ......  
    }    
    return true;    
}    
    
......    
    }    
    
    ......    
}    
```
这个函数定义在文件frameworks/base/services/java/com/android/server/WindowManagerService.java中。

WindowState类的成员函数performShowLocked首先是调用WindowManagerService类的成员函数showSurfaceRobustlyLocked来通知SurfaceFlinger服务将当前正在处理的窗口设置为可见，接着再调用WindowManagerService类的成员函数applyEnterAnimationLocked来给当前正在处理的窗口设置一个进入动画，如下所示：

```java
public class WindowManagerService extends IWindowManager.Stub  
implements Watchdog.Monitor {  
    ......  

    private void applyEnterAnimationLocked(WindowState win) {  
int transit = WindowManagerPolicy.TRANSIT_SHOW;  
if (win.mEnterAnimationPending) {  
    win.mEnterAnimationPending = false;  
    transit = WindowManagerPolicy.TRANSIT_ENTER;  
}  

applyAnimationLocked(win, transit, true);  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/services/java/com/android/server/WindowManagerService.java中。

如果参数win所指向的一个WindowState对象的成员变量mEnterAnimationPending的值等于true，那么就说明它所描述的窗口正在等待显示，也就是正处于不可见到可见状态的过程中，那么WindowManagerService类的成员函数applyEnterAnimationLocked就会对该窗口设置一个类型为WindowManagerPolicy.TRANSIT_ENTER的动画，否则的话，就会对该窗口设置一个类型为WindowManagerPolicy.TRANSIT_SHOW的动画。

确定好窗口的动画类型之后，WindowManagerService类的成员函数applyEnterAnimationLocked就调用另外一个成员函数applyAnimationLocked来为窗口创建一个动画了。接下来我们先分析窗口在关闭的过程中所设置的动画类型，然后再来分析WindowManagerService类的成员函数applyAnimationLocked的实现。

从前面[Android窗口管理服务WindowManagerService计算Activity窗口大小的过程分析](http://blog.csdn.net/luoshengyang/article/details/8479101)一文可以知道，当应用程序进程请求WindowManagerService服务刷新一个窗口的时候，会调用到WindowManagerService类的成员函数relayoutWindow。WindowManagerService类的成员函数relayoutWindow在执行的过程中，如果发现需要将一个窗口从可见状态设置为不可见状态时，也就是发现需要关闭一个窗口时，就会对该窗口设置一个退出动出，如下所示：

```java
public class WindowManagerService extends IWindowManager.Stub    
implements Watchdog.Monitor {    
    ......    
    
    public int relayoutWindow(Session session, IWindow client,    
    WindowManager.LayoutParams attrs, int requestedWidth,    
    int requestedHeight, int viewVisibility, boolean insetsPending,    
    Rect outFrame, Rect outContentInsets, Rect outVisibleInsets,    
    Configuration outConfig, Surface outSurface) {    
......    
     
synchronized(mWindowMap) {    
    WindowState win = windowForClientLocked(session, client, false);  
    ......  

    if (viewVisibility == View.VISIBLE &&  
            (win.mAppToken == null || !win.mAppToken.clientHidden)) {  
        ......  
    } else {  
        ......  

        if (win.mSurface != null) {  
            ......  

            // If we are not currently running the exit animation, we  
            // need to see about starting one.  
            if (!win.mExiting || win.mSurfacePendingDestroy) {  
                // Try starting an animation; if there isn't one, we  
                // can destroy the surface right away.  
                int transit = WindowManagerPolicy.TRANSIT_EXIT;  
                ......  

                if (!win.mSurfacePendingDestroy && win.isWinVisibleLw() &&  
                      applyAnimationLocked(win, transit, false)) {  
                    ......  
                    win.mExiting = true;  
                }   

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

    ......  
}  
```
这个函数定义在文件frameworks/base/services/java/com/android/server/WindowManagerService.java中。

WindowState对象win描述的便是要刷新的窗口。当参数viewVisibility的值不等于View.VISIBLE时，就说明要将WindowState对象win所描述的窗口设置为不可见。另一方面，如果WindowState对象win的成员变量mAppToken的值不等于null，并且这个成员变量所指向的一个AppWindowToken对象的成员变量clientHidden的值等于true，那么就说明WindowState对象win所描述的窗口是一个与Activity组件相关的窗口，并且该Activity组件是处于不可见状态的。在这种情况下，也需要将WindowState对象win所描述的窗口设置为不可见。

一旦WindowState对象win所描述的窗口要设置为不可见，就需要考虑给它设置一个退出动画，不过有四个前提条件：

该窗口有一个绘图表面，即WindowState对象win的成员变量mSurface的值不等于null；

该窗口的绘图表面不是处于等待销毁的状态，即WindowState对象win的成员变量mSurfacePendingDestroy的值不等于true；

该窗口不是处于正在关闭的状态，即WindowState对象win的成员变量mExiting的值不等于true；

该窗口当前正在处于可见的状态，即WindowState对象win的成员isWinVisibleLw的返回值等于true。

在满足上述四个条件的情况下，就说明WindowState对象win所描述的窗口的状态要由可见变为不可见，因此，就需要给它设置一个退出动画，即一个类型为WindowManagerPolicy.TRANSIT_EXIT的动画，这同样是通过调用WindowManagerService类的成员函数applyAnimationLocked来实现的。

从上面的分析就可以知道，无论是窗口在打开时所需要的进入动画，还是窗口在关闭时所需要的退出动画，都是通过调用WindowManagerService类的成员函数applyAnimationLocked来设置的，它的实现如下所示：

```java
public class WindowManagerService extends IWindowManager.Stub    
implements Watchdog.Monitor {    
    ......    
    
    private boolean applyAnimationLocked(WindowState win,  
    int transit, boolean isEntrance) {  
if (win.mLocalAnimating && win.mAnimationIsEntrance == isEntrance) {  
    // If we are trying to apply an animation, but already running  
    // an animation of the same type, then just leave that one alone.  
    return true;  
}  

// Only apply an animation if the display isn't frozen.  If it is  
// frozen, there is no reason to animate and it can cause strange  
// artifacts when we unfreeze the display if some different animation  
// is running.  
if (!mDisplayFrozen && mPolicy.isScreenOn()) {  
    int anim = mPolicy.selectAnimationLw(win, transit);  
    int attr = -1;  
    Animation a = null;  
    if (anim != 0) {  
        a = AnimationUtils.loadAnimation(mContext, anim);  
    } else {  
        switch (transit) {  
            case WindowManagerPolicy.TRANSIT_ENTER:  
                attr = com.android.internal.R.styleable.WindowAnimation_windowEnterAnimation;  
                break;  
            case WindowManagerPolicy.TRANSIT_EXIT:  
                attr = com.android.internal.R.styleable.WindowAnimation_windowExitAnimation;  
                break;  
            case WindowManagerPolicy.TRANSIT_SHOW:  
                attr = com.android.internal.R.styleable.WindowAnimation_windowShowAnimation;  
                break;  
            case WindowManagerPolicy.TRANSIT_HIDE:  
                attr = com.android.internal.R.styleable.WindowAnimation_windowHideAnimation;  
                break;  
        }  
        if (attr >= 0) {  
            a = loadAnimation(win.mAttrs, attr);  
        }  
    }  
    ......  

    if (a != null) {  
        ......  

        win.setAnimation(a);  
        win.mAnimationIsEntrance = isEntrance;  
    }  
}  

......  

return win.mAnimation != null;  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/services/[Java](http://lib.csdn.net/base/java)/com/android/server/WindowManagerService.java中。

参数win描述的是要设置动画的窗口，参数transit描述的是要设置的动画的类型，而参数isEntrance描述的是要设置的动画是进入类型还是退出类型的。

如果参数win所指向的一个WindowState对象的成员变量mLocalAnimating的值等于true，那么就说明它所描述的窗口已经被设置过动画了，并且这个动画正在显示的过程中。在这种情况下，如果这个WindowState对象的成员变量mAnimationIsEntrance的值等于参数isEntrance的值，那么就说明该窗口正在显示的动画就是所要求设置的动画，这时候就不需要给窗口重新设置一个动画了，因此，WindowManagerService类的成员函数applyAnimationLocked就直接返回了。

我们假设需要给参数win所描述的窗口设置一个新的动画，这时候还需要继续判断屏幕当前是否是处于非冻结和点亮的状态的。只有在屏幕不是被冻结并且是点亮的情况下，WindowManagerService类的成员函数applyAnimationLocked才真正需要给参数win所描述的窗口设置一个动画，否则的话，设置了也是无法显示的。当WindowManagerService类的成员变量mDisplayFrozen的时候，就说明屏幕不是被冻结的，而当WindowManagerService类的成员变量mPolicy所指向的一个PhoneWindowManager对象的成员函数isScreenOn的返回值等于true的时候，就说明屏幕是点亮的。在满足上述两个条件的情况下，WindowManagerService类的成员函数applyAnimationLocked就开始给参数win所描述的窗口创建动画了。

WindowManagerService类的成员函数applyAnimationLocked首先是检查WindowManagerService类的成员变量mPolicy所指向的一个PhoneWindowManager对象是否可以为参数win所描述的窗口提供一个类型为transit的动画。如果可以的话，那么调用这个PhoneWindowManager对象的成员函数selectAnimationLw的返回值anim就不等于0，这时候WindowManagerService类的成员函数applyAnimationLocked就可以调用AnimationUtils类的静态成员函数loadAnimation来根据该返回值anim来创建一个动画，并且保存在变量a中。

如果WindowManagerService类的成员变量mPolicy所指向的一个PhoneWindowManager对象不可以为参数win所描述的窗口提供一个类型为transit的动画的话，那么WindowManagerService类的成员函数applyAnimationLocked就需要根据该窗口的布局参数来创建这个动画了。这个创建过程分为两步执行：

将参数transit的值转化为一个对应的动画样式名称；

调用WindowManagerService类的成员函数loadAnimation来在指定的窗口布局参数中创建前面第1步所指定样式名称的动画，并且保存在变量a中，其中，指定的窗口布局参数是由WindowState对象win的成员变量mAttrs所指向的一个WindowManager.LayoutParams对象来描述的。

最后，如果变量a的值不等于null，即前面成功地为参数win所描述的窗口创建了一个动画，那么接下来就会将该动画设置给参数win所描述的窗口。这是通过参数win所指向的一个WindowState对象的成员函数setAnimation来实现的，实际上就是将变量所指向的一个Animation对象保存在参数win所指向的一个WindowState对象的成员变量mAnimation中。同时，WindowManagerService类的成员函数applyAnimationLocked还会将参数isEntrance的值保存在参数win所指向的一个WindowState对象的成员变量mAnimationIsEntrance，以表明前面给它所设置的动画是属于进入类型还是退出类型的。

### 切换动画的设置过程 

从前面[Android窗口管理服务WindowManagerService切换Activity窗口（App Transition）的过程分析](http://blog.csdn.net/luoshengyang/article/details/8596449)一文可以知道，如果一个窗口属于一个Activity组件窗口，那么当该Activity组件被切换的时候，就会被设置一个切换动画，这是通过调用WindowManagerService类的成员函数setTokenVisibilityLocked来实现的，如下所示：

```java
public class WindowManagerService extends IWindowManager.Stub    
implements Watchdog.Monitor {    
    ......    
    
    boolean setTokenVisibilityLocked(AppWindowToken wtoken, WindowManager.LayoutParams lp,  
    boolean visible, int transit, boolean performLayout) {  
boolean delayed = false;  
......  

if (wtoken.hidden == visible) {  
    final int N = wtoken.allAppWindows.size();  
    ......  

    boolean runningAppAnimation = false;  

    if (transit != WindowManagerPolicy.TRANSIT_UNSET) {  
        if (wtoken.animation == sDummyAnimation) {  
            wtoken.animation = null;  
        }  
        applyAnimationLocked(wtoken, lp, transit, visible);  
        ......  

        if (wtoken.animation != null) {  
            delayed = runningAppAnimation = true;  
        }  
    }  

    for (int i=0; i<N; i++) {  
        WindowState win = wtoken.allAppWindows.get(i);  
        ......  

        if (visible) {  
            if (!win.isVisibleNow()) {  
                if (!runningAppAnimation) {  
                    applyAnimationLocked(win,  
                            WindowManagerPolicy.TRANSIT_ENTER, true);  
                }  
                ......  
            }  
        } else if (win.isVisibleNow()) {  
            if (!runningAppAnimation) {  
                applyAnimationLocked(win,  
                        WindowManagerPolicy.TRANSIT_EXIT, false);  
            }  
            ......  
        }  
    }  

    ......  
}  

if (wtoken.animation != null) {  
    delayed = true;  
}  

return delayed;  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/services/java/com/android/server/WindowManagerService.java中。

参数wtoken描述的是要切换的Activity组件，参数lp描述的是要用来创建切换动画的布局参数，参数transit描述的是要创建的切换动画的类型，而参数visible描述的是要切换的Activity组件接下来是否是可见的。

WindowManagerService类的成员函数setTokenVisibilityLocked首先判断要切换的Activity组件当前的可见性是否已经就是要设置的可见性，即参数wtoken所指向的一个AppWindowToken对象的成员变量hidden的值是否不等于参数visible的值。如果不等于的话，就说明要切换的Activity组件当前的可见性已经就是要设置的可见性了，这时候WindowManagerService类的成员函数setTokenVisibilityLocked就不用再为它设置切换动画了。

我们假设要切换的Activity组件当前的可见性不是要求设置的可见性，即参数wtoken所指向的一个AppWindowToken对象的成员变量hidden的值等于参数visible的值，那么WindowManagerService类的成员函数setTokenVisibilityLocked还会继续检查参数transit描述的是否是一个有效的动画类型，即它的值是否不等于WindowManagerPolicy.TRANSIT_UNSET。如果参数transit描述的是一个有效的动画类型的话，那么WindowManagerService类的成员函数setTokenVisibilityLocked接下来就会执行以下三个操作：

判断要切换的Activity组件当前是否被设置了一个哑动画，即参数wtoken所指向的一个AppWindowToken对象的成员变量animation是否与WindowManagerService类的成员变量sDummyAnimation指向了同一个Animation对象。如果是的话，那么就会将wtoken所指向的一个AppWindowToken对象的成员变量animation的值设置为null，因为接下来要重新为它设置一个新的Animation对象。从前面[Android窗口管理服务WindowManagerService切换Activity窗口（App Transition）的过程分析](http://blog.csdn.net/luoshengyang/article/details/8596449)一文可以知道，一个需要参与切换的Activity组件会设置可见性的时候，是会被设置一个哑动画的。

调用WindowManagerService类的四个参数版本的成员函数applyAnimationLocked根据参数lp、transit和visible的值来为要切换的Activity组件创建一个动画。

如果第2步可以成功地为要切换的Activity组件创建一个动画的话，那么这个动画就会保存在wtoken所指向的一个AppWindowToken对象的成员变量animation中，这时候就会将变量delayed和runningAppAnimation的值均设置为true。

变量runningAppAnimation的值等于true意味着与参数wtoken所描述的Activity组件所对应的窗口接下来要执行一个切换动画。在这种情况下，WindowManagerService类的成员函数setTokenVisibilityLocked就不需要为这些窗口单独设置一个进入或者退出类型的动画，否则的话，WindowManagerService类的成员函数setTokenVisibilityLocked就会根据这些窗口的当前可见性状态以及参数wtoken所描述的Activity组件被要求设置的可见性来单独设置一个进入或者退出类型的动画：

如果一个窗口当前是不可见的，即用来描述它的一个WindowState对象的成员函数isVisibleNow的返回值等于false，但是参数wtoken所描述的Activity组件被要求设置成可见的，即参数visible的值等于true，那么就需要给该窗口设置一个类型为WindowManagerPolicy.TRANSIT_ENTER的动画；

如果一个窗口当前是可见的，即用来描述它的一个WindowState对象的成员函数isVisibleNow的返回值等于true，但是参数wtoken所描述的Activity组件被要求设置成不可见的，即参数visible的值等于false，那么就需要给该窗口设置一个类型为WindowManagerPolicy.TRANSIT_EXIT的动画。

给与参数wtoken所描述的Activity组件所对应的窗口设置动画是通过调用WindowManagerService类的三个参数版本的成员函数applyAnimationLocked来实现的，这个成员函数在前面已经分析过了。另外，与参数wtoken所描述的Activity组件所对应的窗口是保存在参数wtoken所指向的一个AppWindowToken对象的成员变量allAppWindows所描述的一个ArrayList中的，因此，通过遍历这个ArrayList，就可以为与参数wtoken所描述的Activity组件所对应的每一个窗口设置一个动画。

最后，如果前面成功地为参数wtoken所描述的Activity组件创建了一个切换动画，即该参数所描述的一个AppWindowToken对象的成员变量animation的值不等于null，那么WindowManagerService类的成员函数setTokenVisibilityLocked的返回值delayed就会等于true，表示参数wtoken所描述的Activity组件要执行一个动换动画，同时也表明该Activity组件的窗口要延迟到切换动画显示结束后，才真正显示出来。

接下来，我们继续分析WindowManagerService类的四个参数版本的成员函数applyAnimationLocked的实现，以便可以了解Activity组件的切换动画的创建过程，如下所示：

```java
public class WindowManagerService extends IWindowManager.Stub    
implements Watchdog.Monitor {    
    ......    
    
    private boolean applyAnimationLocked(AppWindowToken wtoken,  
    WindowManager.LayoutParams lp, int transit, boolean enter) {  
// Only apply an animation if the display isn't frozen.  If it is  
// frozen, there is no reason to animate and it can cause strange  
// artifacts when we unfreeze the display if some different animation  
// is running.  
if (!mDisplayFrozen && mPolicy.isScreenOn()) {  
    Animation a;  
    if (lp != null && (lp.flags & FLAG_COMPATIBLE_WINDOW) != 0) {  
        a = new FadeInOutAnimation(enter);  
        ......  
    } else if (mNextAppTransitionPackage != null) {  
        a = loadAnimation(mNextAppTransitionPackage, enter ?  
                mNextAppTransitionEnter : mNextAppTransitionExit);  
    } else {  
        int animAttr = 0;  
        switch (transit) {  
            case WindowManagerPolicy.TRANSIT_ACTIVITY_OPEN:  
                animAttr = enter  
                        ? com.android.internal.R.styleable.WindowAnimation_activityOpenEnterAnimation  
                        : com.android.internal.R.styleable.WindowAnimation_activityOpenExitAnimation;  
                break;  
            case WindowManagerPolicy.TRANSIT_ACTIVITY_CLOSE:  
                animAttr = enter  
                        ? com.android.internal.R.styleable.WindowAnimation_activityCloseEnterAnimation  
                        : com.android.internal.R.styleable.WindowAnimation_activityCloseExitAnimation;  
                break;  
            case WindowManagerPolicy.TRANSIT_TASK_OPEN:  
                animAttr = enter  
                        ? com.android.internal.R.styleable.WindowAnimation_taskOpenEnterAnimation  
                        : com.android.internal.R.styleable.WindowAnimation_taskOpenExitAnimation;  
                break;  
            case WindowManagerPolicy.TRANSIT_TASK_CLOSE:  
                animAttr = enter  
                        ? com.android.internal.R.styleable.WindowAnimation_taskCloseEnterAnimation  
                        : com.android.internal.R.styleable.WindowAnimation_taskCloseExitAnimation;  
                break;  
            case WindowManagerPolicy.TRANSIT_TASK_TO_FRONT:  
                animAttr = enter  
                        ? com.android.internal.R.styleable.WindowAnimation_taskToFrontEnterAnimation  
                        : com.android.internal.R.styleable.WindowAnimation_taskToFrontExitAnimation;  
                break;  
            case WindowManagerPolicy.TRANSIT_TASK_TO_BACK:  
                animAttr = enter  
                        ? com.android.internal.R.styleable.WindowAnimation_taskToBackEnterAnimation  
                        : com.android.internal.R.styleable.WindowAnimation_taskToBackExitAnimation;  
                break;  
            case WindowManagerPolicy.TRANSIT_WALLPAPER_OPEN:  
                animAttr = enter  
                        ? com.android.internal.R.styleable.WindowAnimation_wallpaperOpenEnterAnimation  
                        : com.android.internal.R.styleable.WindowAnimation_wallpaperOpenExitAnimation;  
                break;  
            case WindowManagerPolicy.TRANSIT_WALLPAPER_CLOSE:  
                animAttr = enter  
                        ? com.android.internal.R.styleable.WindowAnimation_wallpaperCloseEnterAnimation  
                        : com.android.internal.R.styleable.WindowAnimation_wallpaperCloseExitAnimation;  
                break;  
            case WindowManagerPolicy.TRANSIT_WALLPAPER_INTRA_OPEN:  
                animAttr = enter  
                        ? com.android.internal.R.styleable.WindowAnimation_wallpaperIntraOpenEnterAnimation  
                        : com.android.internal.R.styleable.WindowAnimation_wallpaperIntraOpenExitAnimation;  
                break;  
            case WindowManagerPolicy.TRANSIT_WALLPAPER_INTRA_CLOSE:  
                animAttr = enter  
                        ? com.android.internal.R.styleable.WindowAnimation_wallpaperIntraCloseEnterAnimation  
                        : com.android.internal.R.styleable.WindowAnimation_wallpaperIntraCloseExitAnimation;  
                break;  
        }  
        a = animAttr != 0 ? loadAnimation(lp, animAttr) : null;  
        ......  
    }  
    if (a != null) {  
        ......  
        wtoken.setAnimation(a);  
    }  
}   
   
......  

return wtoken.animation != null;  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/services/java/com/android/server/WindowManagerService.java中。

WindowManagerService类的四个参数版本的成员函数applyAnimationLocked和三个参数版本的成员函数applyAnimationLocked的实现是类似的，不过它是为Activity组件创建动画，并且：

如果参数lp所描述的布局参数表明它是用来描述一个兼容窗口的，即它所指向的一个WindowManager.LayoutParams对象的成员变量flags的FLAG_COMPATIBLE_WINDOW位不等于0，那么创建的切换动画就固定为FadeInOutAnimation。

如果WindowManagerService类的成员变量mNextAppTransitionPackage的值不等于null，那么就说明Package名称为mNextAppTransitionPackage的Activity组件指定了一个自定义的切换动画，其中，指定的进入动画的类型由WindowManagerService类的成员变量mNextAppTransitionEnter来描述，指定的退出动画的类型由WindowManagerService类的成员变量mNextAppTransitionExit来描述。在这种情况下，WindowManagerService服务就会使用这个指定的切换动画，而不是使用默认的切换动画。一般来说，一个个Activity组件在调用成员函数startActivity来通知ActivityManagerService服务启动另外一个Activity之外，可以马上调用另外一个成员函数overridePendingTransition来指定自定久的切换动画。

切换动画的类型主要分三类：第一类是和Activity相关的；第二类是和Task相关的；第三类是和Wallpaper相关的。这一点可以参考前面[Android窗口管理服务WindowManagerService切换Activity窗口（App Transition）的过程分析](http://blog.csdn.net/luoshengyang/article/details/8596449)一文。

切换动画创建成功之后，就会调用参数wtoken所指向的一个AppWindowToken对象的成员函数setAnimation来保存在其成员变量animation中，这样就表示它所描述的Activity组件被设置了一个切换动画。

## 窗口动画的显示框架

窗口动画是在WindowManagerService服务刷新系统UI的时候显示的。从前面前面[Android窗口管理服务WindowManagerService切换Activity窗口（App Transition）的过程分析](http://blog.csdn.net/luoshengyang/article/details/8596449)一文可以知道，WindowManagerService服务刷新系统UI是通过调用WindowManagerService类的成员函数performLayoutAndPlaceSurfacesLockedInner来实现的，接下来我们就主要分析与窗口动画的显示框架相关的逻辑，如下所示：

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
      
        // 1. 计算各个窗口的大小    
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
    
        // 2. 推进各个Activity组件的切换动画    
        // Update animations of all applications, including those    
        // associated with exiting/removed apps    
        boolean tokensAnimating = false;  
        final int NAT = mAppTokens.size();  
        for (i=0; i<NAT; i++) {  
            if (mAppTokens.get(i).stepAnimationLocked(currentTime, dw, dh)) {  
                tokensAnimating = true;  
            }  
        }  
        final int NEAT = mExitingAppTokens.size();  
        for (i=0; i<NEAT; i++) {  
            if (mExitingAppTokens.get(i).stepAnimationLocked(currentTime, dw, dh)) {  
                tokensAnimating = true;  
            }  
        }    
      
        // 3. 推进各个窗口的动画    
        // SECOND LOOP: Execute animations and update visibility of windows.      
        ......    

        animating = tokensAnimating;   
        ......   

        mPolicy.beginAnimationLw(dw, dh);  

        final int N = mWindows.size();  

        for (i=N-1; i>=0; i--) {  
            WindowState w = mWindows.get(i);  
            ......  

            if (w.mSurface != null) {  
                // Execute animation.  
                ......  
                if (w.stepAnimationLocked(currentTime, dw, dh)) {  
                    animating = true;  
                    ......  
                }  

                ......  
            }  

            ......  

            mPolicy.animatingWindowLw(w, attrs);  
        }  
    
        ......    

        changes |= mPolicy.finishAnimationLw();  

        ......  
              
    } while (changes != 0);      
      
    // 4. 更新各个窗口的绘图表面    
    // THIRD LOOP: Update the surfaces of all windows.      
    ......      

    final int N = mWindows.size();  

    for (i=N-1; i>=0; i--) {  
        WindowState w = mWindows.get(i);  
        ......  

        if (w.mSurface != null) {  
            ......  
              
            //计算实际要显示的大小和位置  
            w.computeShownFrameLocked();  
            ......  

            //设置大小和位置等  
            ......  

            if (w.mAttachedHidden || !w.isReadyForDisplay()) {  
                ......  
            } else if (w.mLastLayer != w.mAnimLayer  
                    || w.mLastAlpha != w.mShownAlpha  
                    || w.mLastDsDx != w.mDsDx  
                    || w.mLastDtDx != w.mDtDx  
                    || w.mLastDsDy != w.mDsDy  
                    || w.mLastDtDy != w.mDtDy  
                    || w.mLastHScale != w.mHScale  
                    || w.mLastVScale != w.mVScale  
                    || w.mLastHidden) {  
                ......  

                //设置Z轴位置、Alpha通道和变换矩阵   
                ......  

                if (w.mLastHidden && !w.mDrawPending  
                        && !w.mCommitDrawPending  
                        && !w.mReadyToShow) {  
                    ......   

                    if (showSurfaceRobustlyLocked(w)) {  
                        w.mHasDrawn = true;  
                        w.mLastHidden = false;  
                    }   
                    
                    ......  
               }  

               ......  
            }  

            ......  
        }  

        ......  
    }  

    ......  
} catch (RuntimeException e) {      
    ......      
}      
      
......      
      
Surface.closeTransaction();      
      
......      
    
// 5. 检查是否需要再一次刷新系统UI    
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

WindowManagerService类的成员函数performLayoutAndPlaceSurfacesLockedInner按照以下步骤来显示窗口动画：

第一步是调用WindowManagerService类的成员函数performLayoutLockedInner来计算各个窗口的大小以及位置。只有知道了窗口的大小以及位置之后，我们才能它应用一个变换矩阵，然后得到窗口下一步要显示的大小以及位置。这一步可以参考前面[Android窗口管理服务WindowManagerService计算Activity窗口大小的过程分析](http://blog.csdn.net/luoshengyang/article/details/8479101)一文。

第二步是推进各个Activity组件的切换动画，也就是计算每一个Activity组件下一步所要执行的动画。注意，只有那些正在参与切换操作的Activity组件才设置有动画，而只有设置有切换动画的Activity组件才需要计算它们下一步所要执行的动画。从前面第一部分的内容可以知道，如果一个Activity组件参与了切换操作，那么用来描述它的一个AppWindowToken对象的成员变量animation就会指向一个Animation对象，用来描述一个切换动画。

用来描述系统当前的Activity组件的AppWindowToken对象有一部分保存在WindowManagerService类的成员变量mAppTokens所描述的一个ArrayList中，另外一部分保存在WindowManagerService类的成员变量mExitingAppTokens所描述的一个ArrayList中。其中，保存在成员变量mAppTokens中的AppWindowToken对象有可能描述的就是正在打开的Activity组件，而保存在成员变量mExitingAppTokens中的AppWindowToken对象描述的就是正在关闭的Activity组件，这两类Activity组件都是参与了切换操作的，因此，我们需要计算它们下一步所要执行的动画，这是通过调用用来描述它们的AppWindowToken对象的成员函数stepAnimationLocked来实现的。注意，对于那些不是正在打开或者正在关闭的Activity组件，调用用来描述它们的AppWindowToken对象的成员函数stepAnimationLocked不会有任何效果，因为这些AppWindowToken对象的成员变量animation的值等于null。

在这一步中，如果有任何一个正在打开或者正在关闭的Activity组件的动画还没有执行完成，那么调用用来描述它的AppWindowToken对象的成员函数stepAnimationLocked的返回值就会等于true，这时候变量tokensAnimating的值也会等于true，表示Activity组件的切换动画还在进行中。后面我们再详细分析AppWindowToken类的成员函数stepAnimationLocked的实现。

第三步是推进各个Activity组件的动画，也就是计算每一个窗口下一步所要执行的动画。注意，只有那些被设置有动画的窗口才需要计算它们下一下所要执行的动画。从前面第一部分的内容可以知道，如果一个窗口设置有动画，那么用来描述它的一个WindowState对象的成员变量mAnimation就会指向一个Animation对象，用来描述一个窗口动画。

用来描述系统当前的窗口的WindowState对象保存在WindowManagerService类的成员变量mWindows所描述的一个ArrayList中，只要调用这些WindowState对象的成员函数stepAnimationLocked，就可以计算它们所描述的窗口下一步所要执行的动画。同样，对于那些没有设置动画的窗口来说，调用用来描述它们的WindowState对象的成员函数stepAnimationLocked是不会有任何效果的，因为这些WindowState对象的成员变量mAnimation的值等于null。

注意，对于那些还没有创建绘图表面的窗口来说，即使它们设置有动画，在这一步里面，也是不需要计算它们下一步所要执行的动画的，这是因为一个没有绘图表面的窗口是无法对它应用动画的。

此外，在计算所有窗口下一步要执行的动画之前以及之后，会通知系统的窗口管理策略类窗口下一次要执行的动画就要开始计算了以及已经计算结束了，同时，每计算完成一个窗口下一次要执行的动画之后，也会通知系统的窗口管理策略类该窗口下一次要执行的动画已经计算完成了，这分别是通过调用WindowManagerService类的成员变量mPolicy所指向的一个PhoneWindowManager对象的成员函数beginAnimationLw、finishAnimationLw和animatingWindowLw来实现的。

我们知道，在Android系统中，有两个特殊的系统窗口，即状态栏窗口和锁屏窗口。如果当前激活的窗口是一个全屏窗口的时候，那么系统的状态栏窗口就会被隐藏。同时，在显示锁屏窗口的时候，一般系统中的所有其它窗口都会被它挡在后面。但是有些被设置了FLAG_SHOW_WHEN_LOCKED标志位的窗口，能够显示在锁屏窗口的上面。还有些被设置了FLAG_DISMISS_KEYGUARD位的窗口，它们能够解散系统当前正在显示的锁屏窗口。这些全屏窗口、能够显示在锁屏窗口上面的窗口以及能够解散锁屏窗口的窗口，会影响到系统的状态栏窗口和锁屏窗口是否需要显示的问题，而这是需要由窗口管理策略类来决定的。因此，在计算所有窗口下一步要执行的动画的前后，以及计算每一个窗口下一次要执行的动画的过程中，都要通知一下窗口管理策略类，以便它可以决定是否需要显示系统的状态栏窗口和锁屏窗口。

窗口管理策略类按照以下方法来决定是否需要显示系统的状态栏窗口和锁屏窗口：

在计算所有窗口下一步要执行的动画之前，假设状态栏窗口是不可见的，而锁屏窗口是可见的、不可解散的；

在计算一个窗口下一步要执行的动画的过程中，检查该窗口是否是可见的以及全屏显示的。如果该窗口是可见的，但是不是全屏的，那么就意味着状态栏窗口是可见的。同时，如果该窗口是可见的，并且也是全屏的，那么就会继续检查该窗口是否被设置了FLAG_SHOW_WHEN_LOCKED和FLAG_DISMISS_KEYGUARD标志位。如果这两个标志位分别被设置了，那么就意味着锁屏窗口是不可见的和需要解散的。

在计算完成所有窗口下一步要执行的动画之后，根据第2步收集到的信息来决定是否要显示状态栏窗口以及锁屏窗口。如果系统当前存在状态栏窗口，并且第2步收集到的信息表明它是需要显示的，那么就会将它的状态设置为可见的；另一方面，如果系统当前存在状态栏窗口，并且第2步收集到的信息表明系统当前有一个全屏的窗口，那么状态栏窗口无论如何都是需要隐藏的。如果系统当前存在锁屏窗口，并且第2步收集到的信息表明它是不需要显示的或者需要解散的，那么就会将它的状态设置不可见，否则的话，就会将它的状态设置为可见。

在计算完成所有窗口下一步要执行的动画之后，如果状态栏窗口或者锁屏窗口的状态由可见变为不可见，或者由不可见变为可见，那么PhoneWindowManager类的成员函数finishAnimationLw的返回值就会不等于0，这意味着WindowManagerService服务需要重新布局系统中各个窗口，即重新计算各个窗口的大小、位置以及下一步要执行的动画等操作，因为状态栏窗口或者锁屏窗口的可见性变化引发窗口堆栈发生变化。这就是上述的第一步、第二步和第三步要放在一个while循环来执行的原因。但是，这个while循环不能无限地执行下去，否则的话，系统的UI就永远刷新不出来。这个while循环最多允许执行7次，并且各个窗口的大小和位置的重复计算次数最多为4次。

第四步是更新各个窗口的绘图表面。既然是更新各个窗口的绘图表面，那么就意味着只有具有绘图表面的窗口才需要更新。一个窗口如果具有绘图表面，那么用来描述它的一个WindowState对象的成员变量mSurface的值就不等于null。对于具有绘图表面的窗口，这一步主要是执行以下操作：

调用用来描述它的WindowState对象的成员函数computeShownFrameLocked来计算它实际要显示的大小和位置，这是需要考虑它的原始大小和位置，以及它所被设置的变换矩阵。这个变换矩阵是通过组合它的动画来得到的，其中包括窗口本身所设置的及其所附加在的窗口所设置的动画，还有窗口所属的Activity组件所设置的切换动画。后面我们再分析WindowState类的成员函数computeShownFrameLocked的实现。

将第1步计算得到的窗口实际要显示的大小以及位置设置到SurfaceFlinger服务中去。这一点可以参考前面[Android窗口管理服务WindowManagerService计算窗口Z轴位置的过程分析](http://blog.csdn.net/luoshengyang/article/details/8570428)一文。

如果窗口当前已经就准备就绪显示，即用来描述它的WindowState对象的成员函数isReadyForDisplay的返回值等于true，并且它所附加在的窗口是可见的，即用来描述它的WindowState对象的成员变量mAttachedHidden的值等于false，那么只要满足以下条件之一，就需要考虑通知SurfaceFlinger服务将该窗口的状态设置为可见的：

A. 该窗口的Z轴位置发生了变化，即用来描述该窗口的WindowState对象的成员变量mLastLayer和mAnimLayer的值不相等；

B. 该窗口的Alpha通道值发生了变化，即用来描述该窗口的WindowState对象的成员变量mLastAlpha和mShownAlpha的值不相等；

C. 该窗口的变换矩阵发生了变化，即用来描述该窗口的WindowState对象的成员变量mLastDsDx、mLastDtDx、mLastDsDy和mLastDtDy的值不等于mDsDx、mDtDx、mDsDy和mDtDy的值；

D. 该窗口在宽度和高度上所设置的缩放因子发生了变化，即用来描述该窗口的WindowState对象的成员变量mLastHScale和mLastVScale的值不等于mHScale和mVScale的值；

E. 该窗口在上一次系统UI刷新时是不可见的，即用来描述该窗口的WindowState对象的成员变量mLastHidden的值等于true。

如果一个具有绘图表面的窗口满足上述条件，那么这一步会将该窗口的最新Z轴位置、Alpha通道值和变换矩阵设置到SurfaceFlinger服务中去，这一点可以参考前面[Android窗口管理服务WindowManagerService计算窗口Z轴位置的过程分析](http://blog.csdn.net/luoshengyang/article/details/8570428)一文。

此外，如果一个具有绘图表面的窗口满足的是上述的条件E，并且还满足以下条件：

F. 它的UI已经绘制完成，即用来描述它的WindowState对象的成员变量mDrawPending和mCommitDrawPending值等于false；

G. 它不是处于等待同一个窗口令牌的其它窗口的完成UI绘制的状态，即用来描述它的WindowState对象的成员变量mReadyToShow的值等于false。

那么这一步还会调用WindowManagerService类的成员函数showSurfaceRobustlyLocked来通知SurfaceFlinger服务将该窗口的状态设置为可见的。如果能够成功地通知SurfaceFlinger服务将该窗口的状态设置为可见，那么还会分别将用来描述该窗口的WindowState对象的成员变量mHasDrawn和mLastHidden的值设置为true和false，表示该窗口的UI已经绘制完成，并且状态已经设置为可见。

注意，上述四步对窗口的操作，例如设置窗口的大小、位置、Alpha通道值和变换矩阵等，都是在一个事务中执行的。这个事务从调用Surface类的静态成员函数openTransaction时开始，一直到调用Surface类的静态成员函数closeTransaction时结束。当事务结束的时候，前面所设置的窗口的状态才会批量地同步到SurfaceFlinger服务中去，这样就可以避免每修改一个窗口的一个状态，就触发SurfaceFlinger服务刷新一次系统的UI，造成屏幕闪烁。

第五步是检查是否需要再一次刷新系统UI。在三种情况下，是需要再一次刷新系统UI的。第一种情况是发现此时Activity组件的切换动画已经显示完成了；第二种情况是发现前面的操作会导致壁纸窗口的目标窗口被销毁了；第三种情况是发现此时还有窗口的动画未结束。

由于在Activity组件的切换过程中，很多操作都会被延迟执行，例如，窗口的Z轴位置的计算，因此，当出现第一种情况下，就需要重新执行这些延迟操作，主要就是重建窗口堆栈，以及计算每一个窗口的Z轴位置。在第二种情况下，由于当壁纸窗口的目标窗品被销毁之后，因此，就需要重新调整壁纸窗口在窗口堆栈中的位置。这两种情况都会导致变量needRelayout的值就等于true，表示需要马上重新刷新系统UI，这是通过以0为参数来调用WindowManagerService类的成员函数requestAnimationLocked来实现的。

在第三种情况中，变量animating的值会等于true，表示有窗口的动画还未结束，有可能是窗口本身的动画尚未结束，也有可能是Activity组件的切换动画尚未结束。在WindowManagerService服务中，窗口动画是以60帧每秒（fps）的速度来显示的，因此，这一步就会以约等于1000/60的参数来调用WindowManagerService类的成员函数requestAnimationLocked，表示要在1/60秒后再次刷新系统的UI，这样就可以把动画效果展现出来。

## 窗口动画的推进过程

从前面第一部分的内容可以知道，窗口动画有可能是是来自窗口本身所设置的动画，也有可能是来自于其宿主Activity组件所设置的切换动画，接下来我们就分别分析这两种类型的动画的推进过程。

### 窗口动画的推进过程

从前面第二部分的内容可以知道，窗口动画的推进过程是由WindowState类的成员函数stepAnimationLocked的实现的，如下所示：

```java
public class WindowManagerService extends IWindowManager.Stub  
implements Watchdog.Monitor {  
    ......  

    private final class WindowState implements WindowManagerPolicy.WindowState {  
......  

// Currently running animation.  
boolean mAnimating;  
boolean mLocalAnimating;  
Animation mAnimation;  
......  
boolean mHasLocalTransformation;  
final Transformation mTransformation = new Transformation();  
......  

// This must be called while inside a transaction.  Returns true if  
// there is more animation to run.  
boolean stepAnimationLocked(long currentTime, int dw, int dh) {  
    if (!mDisplayFrozen && mPolicy.isScreenOn()) {  
        // We will run animations as long as the display isn't frozen.  

        if (!mDrawPending && !mCommitDrawPending && mAnimation != null) {  
            ......  
            mHasLocalTransformation = true;  
            if (!mLocalAnimating) {  
                ......  
                mAnimation.initialize(mFrame.width(), mFrame.height(), dw, dh);  
                mAnimation.setStartTime(currentTime);  
                mLocalAnimating = true;  
                mAnimating = true;  
            }  
            mTransformation.clear();  
            final boolean more = mAnimation.getTransformation(  
                currentTime, mTransformation);  
            ......  

            if (more) {  
                // we're not done!  
                return true;  
            }  
            ......  
        }  
        ......  
    }   

    ......  

    mAnimating = false;  
    mLocalAnimating = false;  
    mAnimation = null;  
    ......  

    mHasLocalTransformation = false;  
    ......  

    mTransformation.clear();  
    ......  

    return false;  
}  

......  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/services/java/com/android/server/WindowManagerService.java中。

WindowState类有几个成员变量是用来描述窗口的动画状态的，其中：

| 动画状态                    | 说明                                       |
| :---------------------- | :--------------------------------------- |
| mAnimating              | 表示窗口是否处于正在显示动画的过程中                       |
| mLocalAnimating         | 表示窗口的动画是否已经初始化过了。一个动画只有经过初始化之后，才能开始执行    |
| mAnimation              | 表示窗口的动画对象                                |
| mHasLocalTransformation | 表示窗口的动画是否是一个本地动画，即这个动画是否是来自窗口本身的。有时候一个窗口虽然正在显示动画，但是这个动画有可能是其宿主Activity组件的切换动画。在这种情况下，mHasLocalTransformation的值就会等于false |
| mTransformation         | 表示一个变换矩阵，是根据窗口动画的当前推进状态来计算得到的，用来改变窗口的大小以及位置 |

窗口动画的推进只有以下四个条件均满足的情况下才会执行：

(1). 屏幕没有被冻结，即WindowManagerService类的成员变量mDisplayFrozen的值等于false；

(2). 屏幕是点亮的，即WindowManagerService类的成员变量mPolicy所指向的一个PhoneWindowManager对象的成员函数isScreenOn的返回值等于true；

(3). 窗口的UI已经绘制完成并且已经提交，即WindowState类的成员变量mDrawPending和mCommitDrawPending的值均等于false；

(4). 窗口已经被设置过动画，即WindowState类的成员变量mAnimation的值不等于null。

一旦满足上述四个条件，那么WindowState类的成员函数stepAnimationLocked就会执行以下操作：

(1). 将成员变量mHasLocalTransformation的值设置为true，表明窗口当前具有一个本地动画。

(2). 检查成员变量mLocalAnimating的值是否等于false。如果等于false的话，那么就说明窗口的动画还没有经过初始化，这时候就会对该动画进行初始化，这是通过调用成员变量mAnimation所指向的一个Animation对象的成员函数initialize和setStartTime来实现的。窗口动画初始化完成之后，还需要将成员变量mLocalAnimating和mAnimating的值均设置为true，表明窗口动画已经初始化过了，并且窗口当前正在执行动画的过程中。

(3). 将成员变量mTransformation所描述的变换矩阵的数据清空。

(4). 调用成员变量mAnimation所指向的一个Animation对象的成员函数getTransformation来计算窗口动画下一步所对应的变换矩阵，并且将这个变换矩阵的数据保存在成员变量mTransformation。

(5). 如果窗口的动画尚未结束显示，那么调用成员变量mAnimation所指向的一个Animation对象的成员函数getTransformation得到的返回值more就会等于true，这时候窗口动画的向前推进操作就完成了。

(6). 如果窗口的动画已经结束显示，那么调用成员变量mAnimation所指向的一个Animation对象的成员函数getTransformation得到的返回值more就会等于false，这时候就需要执行一些清理工作。这些清理工作包括将成员变量mAnimating、mLocalAnimating和mHasLocalTransformation的值设置为false，以及将成员变量mAnimation的值设置为null，还有将成员变量mTransformation所描述的变换矩阵的数据清空。

最后，如果窗口的动画尚未结束显示，那么WindowState类的成员函数stepAnimationLocked会返回一个true值给调用者，否则的话，就会返回一个false值给调用者。

### Activity组件切换动画的推进过程

从前面第二部分的内容可以知道，Activity组件切换动画的推进过程是由AppWindowToken类的成员函数stepAnimationLocked来实现的，如下所示：

```java
public class WindowManagerService extends IWindowManager.Stub  
implements Watchdog.Monitor {  
    ......  

    class AppWindowToken extends WindowToken {  
......  

boolean animating;  
Animation animation;  
boolean hasTransformation;  
final Transformation transformation = new Transformation();  
......  

// This must be called while inside a transaction.  
boolean stepAnimationLocked(long currentTime, int dw, int dh) {  
    if (!mDisplayFrozen && mPolicy.isScreenOn()) {  
        // We will run animations as long as the display isn't frozen.  

        if (animation == sDummyAnimation) {  
            // This guy is going to animate, but not yet.  For now count  
            // it as not animating for purposes of scheduling transactions;  
            // when it is really time to animate, this will be set to  
            // a real animation and the next call will execute normally.  
            return false;  
        }  

        if ((allDrawn || animating || startingDisplayed) && animation != null) {  
            if (!animating) {  
                ......  
                animation.initialize(dw, dh, dw, dh);  
                animation.setStartTime(currentTime);  
                animating = true;  
            }  
            transformation.clear();  
            final boolean more = animation.getTransformation(  
                currentTime, transformation);  
            ......  
            if (more) {  
                // we're done!  
                hasTransformation = true;  
                return true;  
            }  
            ......  
            animation = null;  
        }  
    }  

    ......  

    hasTransformation = false;  
    ......  

    animating = false;  
    ......  

    transformation.clear();  
    ......  

    return false;  
}  

......  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/services/java/com/android/server/WindowManagerService.java中。

AppWindowToken类有几个成员变量是用来描述Activity组件的切换动画状态的，其中：

| 动画状态               | 说明                                       |
| :----------------- | :--------------------------------------- |
| animating          | 表示Activity组件的切换动画是否已经初始化过了               |
| animation          | 表示Activity组件的切换动画对象                      |
| mHasTransformation | 表示Activity组件是否具有切换动画                     |
| transformation     | 表示一个变换矩阵，是根据Activity组件切换动画的当前推进状态来计算得到的，用来改变Activity组件窗口的大小以及位置 |

Activity组件切换动画的推进只有以下五个条件均满足的情况下才会执行：

(1). 屏幕没有被冻结，即WindowManagerService类的成员变量mDisplayFrozen的值等于false。

(2). 屏幕是点亮的，即WindowManagerService类的成员变量mPolicy所指向的一个PhoneWindowManager对象的成员函数isScreenOn的返回值等于true。

(3). Activity组件所设置的切换动画不是一个哑动画，即AppWindowToken类的成员变量animation与WindowManagerService类的静态成员变量sDummyAnimation不是指向同一个Animation对象。从前面[Android窗口管理服务WindowManagerService切换Activity窗口（App Transition）的过程分析](http://blog.csdn.net/luoshengyang/article/details/8596449)一文可以知道，一个Activity组件在进行切换之前，用来描述它的一个AppWindowToken对象的成员变量animation的值会被设置为sDummyAnimation，用来表示它准备要执行一个切换动画，但是这个切换动画还没有设置。

(4). Activity组件的所有窗口的UI都已经绘制完成，或者Activity组件的切换动画已经初始化过了，或者Activity组件的启动窗口已经结束显示，即AppWindowToken类的成员变量allDrawn、animating和startingDisplayed中的其中一个的值等于true。

(5). Activity组件已经被设置过切换动画，即AppWindowToken类的成员变量animation的值不等于null。

一旦满足上述五个条件，那么AppWindowToken类的成员函数stepAnimationLocked就会执行以下操作：

(1). 检查成员变量animating的值是否等于false。如果等于false的话，那么就说明Activity组件的切换动画还没有经过初始化，这时候就会对该动画进行初始化，这是通过调用成员变量animation所指向的一个Animation对象的成员函数initialize和setStartTime来实现的。窗口动画初始化完成之后，还需要将成员变量animating的值设置为true，表明窗口动画已经初始化过了。

(2). 将成员变量transformation所描述的变换矩阵的数据清空。

(3). 调用成员变量animation所指向的一个Animation对象的成员函数getTransformation来计算Activity组件切换动画下一步所对应的变换矩阵，并且将这个变换矩阵的数据保存在成员变量transformation。

(4). 如果Activity组件切换动画尚未结束显示，那么调用成员变量animation所指向的一个Animation对象的成员函数getTransformation得到的返回值more就会等于true，这时候Activity组件切换动画的向前推进操作就完成了。

(6). 如果Activity组件切换动画已经结束显示，那么调用成员变量animation所指向的一个Animation对象的成员函数getTransformation得到的返回值more就会等于false，这时候就需要执行一些清理工作。这些清理工作包括将成员变量animating和hasTransformation的值设置为false，以及将成员变量animation的值设置为null，还有将成员变量transformation所描述的变换矩阵的数据清空。

最后，如果窗口的动画尚未结束显示，那么AppWindowToken类的成员函数stepAnimationLocked就会将成员变量hasTransformation的值设置为true，并且返回一个true值给调用者，否则的话，就会返回一个false值给调用者。

## 窗口动画的合成过程

从前面第二部分的内容可以知道，WindowState类的成员函数computeShownFrameLocked负责合成窗口的动画，包括窗口本身所设置的进入（退出）动画、从被附加窗口传递过来的动画，以及宿主Activity组件传递过来的切换动画。窗口的这三个动画合成之后，就可以得到一个变换矩阵。将这个变换矩阵应用到窗口的原始大小和位置上去，就可以得到窗口经过动画变换后所得到的位置和大小。

WindowState类的成员函数computeShownFrameLocked的实现如下所示：

```java
public class WindowManagerService extends IWindowManager.Stub  
implements Watchdog.Monitor {  
    ......  

    private final class WindowState implements WindowManagerPolicy.WindowState {  
......  

// Actual frame shown on-screen (may be modified by animation)  
final Rect mShownFrame = new Rect();  
......  

// Current transformation being applied.  
float mDsDx=1, mDtDx=0, mDsDy=0, mDtDy=1;  
......  

// "Real" frame that the application sees.  
final Rect mFrame = new Rect();  
......  

void computeShownFrameLocked() {  
    final boolean selfTransformation = mHasLocalTransformation;  
    Transformation attachedTransformation =  
            (mAttachedWindow != null && mAttachedWindow.mHasLocalTransformation)  
            ? mAttachedWindow.mTransformation : null;  
    Transformation appTransformation =  
            (mAppToken != null && mAppToken.hasTransformation)  
            ? mAppToken.transformation : null;  

    // Wallpapers are animated based on the "real" window they  
    // are currently targeting.  
    if (mAttrs.type == TYPE_WALLPAPER && mLowerWallpaperTarget == null  
            && mWallpaperTarget != null) {  
        if (mWallpaperTarget.mHasLocalTransformation &&  
                mWallpaperTarget.mAnimation != null &&  
                !mWallpaperTarget.mAnimation.getDetachWallpaper()) {  
            attachedTransformation = mWallpaperTarget.mTransformation;  
            ......  
        }  
        if (mWallpaperTarget.mAppToken != null &&  
                mWallpaperTarget.mAppToken.hasTransformation &&  
                mWallpaperTarget.mAppToken.animation != null &&  
                !mWallpaperTarget.mAppToken.animation.getDetachWallpaper()) {  
            appTransformation = mWallpaperTarget.mAppToken.transformation;  
            ......  
        }  
    }  

    if (selfTransformation || attachedTransformation != null  
            || appTransformation != null) {  
        // cache often used attributes locally  
        final Rect frame = mFrame;  
        final float tmpFloats[] = mTmpFloats;  
        final Matrix tmpMatrix = mTmpMatrix;  

        // Compute the desired transformation.  
        tmpMatrix.setTranslate(0, 0);  
        if (selfTransformation) {  
            tmpMatrix.postConcat(mTransformation.getMatrix());  
        }  
        tmpMatrix.postTranslate(frame.left, frame.top);  
        if (attachedTransformation != null) {  
            tmpMatrix.postConcat(attachedTransformation.getMatrix());  
        }  
        if (appTransformation != null) {  
            tmpMatrix.postConcat(appTransformation.getMatrix());  
        }  

        // "convert" it into SurfaceFlinger's format  
        // (a 2x2 matrix + an offset)  
        // Here we must not transform the position of the surface  
        // since it is already included in the transformation.  
        //Slog.i(TAG, "Transform: " + matrix);  

        tmpMatrix.getValues(tmpFloats);  
        mDsDx = tmpFloats[Matrix.MSCALE_X];  
        mDtDx = tmpFloats[Matrix.MSKEW_X];  
        mDsDy = tmpFloats[Matrix.MSKEW_Y];  
        mDtDy = tmpFloats[Matrix.MSCALE_Y];  
        int x = (int)tmpFloats[Matrix.MTRANS_X] + mXOffset;  
        int y = (int)tmpFloats[Matrix.MTRANS_Y] + mYOffset;  
        int w = frame.width();  
        int h = frame.height();  
        mShownFrame.set(x, y, x+w, y+h);  
        ......  

        return;  
    }  

    mShownFrame.set(mFrame);  
    if (mXOffset != 0 || mYOffset != 0) {  
        mShownFrame.offset(mXOffset, mYOffset);  
    }  
    ......  
    mDsDx = 1;  
    mDtDx = 0;  
    mDsDy = 0;  
    mDtDy = 1;  
}  

......  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/services/java/com/android/server/WindowManagerService.java中。

在分析WindowState类的成员函数computeShownFrameLocked的实现之前，我们先来看几个相关的成员变量：

**--mFrame**，用来描述窗口的实际位置以及大小。它们的计算过程可以参考前面[Android窗口管理服务WindowManagerService计算Activity窗口大小的过程分析](http://blog.csdn.net/luoshengyang/article/details/8479101)一文。

**--mShownFrame**，用来描述窗口当前所要显示的位置以及大小。

**--mDsDx、mDtDx、mDsDy、mDtDy**，用来描述窗口的变换矩阵（二维）。

WindowState类的成员函数computeShownFrameLocked的目标就根据窗口的实际位置、大小，以及窗口的动画，来计算得到窗口当前所要显示的位置以及大小。

注意，在调用WindowState类的成员函数computeShownFrameLocked之前，窗口的实际位置和大小是已经计算好了的，并且窗口的动画也是已经向前推进好了的。窗口的实际位置和大小的计算过程可以参考前面[Android窗口管理服务WindowManagerService计算Activity窗口大小的过程分析](http://blog.csdn.net/luoshengyang/article/details/8479101)一文，而窗口动画向前推进的过程可以参考前面第三部分的内容。

WindowState类的成员函数computeShownFrameLocked首先检查当前正在处理的窗口有多少个动画是需要合成的，即：

检查成员变量mHasLocalTransformation的值是否等于true。如果等于true的话，那么就会将变量selfTransformation的值也设置为true，表示窗口本身设置有一个动画。这个动画要么是一个打开窗口类型的动画，要么是一个关闭窗口类型的动画。

检查成员变量mAttachedWindow的值是否等于null。如果不等于null的话，那么就说明当前正在处理的窗口是附加在另外一个窗口之上。如果这个被附加的窗口也设置有动画，那么成员变量mAttachedWindow所指向的一个WindowState对象的成员变量mHasLocalTransformation的值就会等于true，这时候用来描述被附加窗口当前所要执行的动画的一个变换矩阵就由该WindowState对象的成员变量mTransformation来描述。这个变换矩阵最终会被保存在变量attachedTransformation中。

检查成员变量mAppToken的值是否等于null。如果不等于null的话，那么就说明当前正在处理的窗口是一个Activity组件的窗口。如果这个Activity组件设置有切换动画，那么成员变量mAppToken所指向的一个AppWindowToken对象的成员变量hasTransformation的值就会等于true，这时候用来描述该Activity组件当前有所要执行的切换动画的一个变换矩阵就由该AppWindowToken对象的成员变量transformation来描述。这个变换矩阵最终会被保存在变量appTransformation中。

WindowState类的成员函数computeShownFrameLocked接着检查当前正在处理的窗口是否是一个壁纸窗口，即检查成员变量mAttrs所指向的一个WindowManager.LayoutParams对象的成员变量type的值的TYPE_WALLPAPER位是否等于1。如果当前正在处理的窗口是一个壁纸窗口，并且它当前有且仅有一个目标窗口，即WindowManagerService类的成员变量mWallpaperTarget和mLowerWallpaperTarget的值分别不等于null和等于null。在这种情况下，需要对壁纸窗口的动画进行特殊处理，即：

要把壁纸窗口所附加在的窗口的动画设置为壁纸窗口的目标窗口所附加在的窗口的动画，即将变量attachedTransformation指向用来描述壁纸窗口的目标窗口所附加在的窗口当前所要执行的动画的一个变换矩阵，前提是壁纸窗口的目标窗口设置有动画，并且这个目标窗口在结束动画过程后不会与壁纸窗口分离。

要把壁纸窗口的切换动画设置为壁纸窗口的目标窗口的切换动画，即将变量appTransformation指向用来描述壁纸窗口的目标窗口当前所要执行的切换动画的一个变换矩阵，前提是壁纸窗口的目标窗口设置有切换动画，并且这个目标窗口结束动画过程后不会与壁纸窗口分离。

通过上面的处理之后，如果变量selfTransformation的值等于true，或者变量attachedTransformation和appTransformation的值不等于null，那么就说明当前正在处理的窗口有动画需要显示，因此，接下来就要将这些动画组合成一个总的变换矩阵。这个总的变换矩阵就是通过WindowState类的成员变量mTmpMatrix来描述的，它是通过下面的步骤来获得的：

将该矩阵的偏移位置初始化为(0, 0)，这是通过调用变量tmpMatrix所描述的一个Matrix对象的成员函数setTranslate来实现的。

如果变量selfTransformation的值等于true，那么就说明当前正在处理的窗口本身设置有动画。这个动画是通过WindowState类的成员变量mTransformation来描述的，调用这个成员变量所指向的一个Transformation对象的成员函数getMatrix就可以获得一个对应的变换矩阵。将这个变换矩阵与变量tmpMatrix所描述的变换矩阵相乘，就可以得到一个中间变换矩阵，这是通过调用变量tmpMatrix所指向的一个Matrix对象的成员函数postConcat来实现的。

将上面第2步得到的中间变换矩阵的偏移位置设置为当前正在处理的窗口的实际位置，这是通过调用变量tmpMatrix所描述的一个Matrix对象的成员函数postTranslate来实现的，而当前正在处理的窗口的实际位置由变量frame所指向的一个Frame对象的成员变量left和top来描述。注意，变量frame和WindowState类的成员变量mFrame指向的是同一个Frame对象，因此它的成员变量left和top描述的是正在处理的窗口的实际位置。

如果变量attachedTransformation的值不等于null，那么就说明当前正在处理的窗口所附加在的窗口设置有动画。这个动画就是通过变量attachedTransformation所指向的一个Transformation对象来描述的，调用这个Transformation对象的成员函数getMatrix就可以获得一个对应的变换矩阵。将这个变换矩阵与变量tmpMatrix所描述的变换矩阵相乘，就可以得到一个中间变换矩阵，这是通过调用变量tmpMatrix所指向的一个Matrix对象的成员函数postConcat来实现的。

如果变量attachedTransformation的值不等于null，那么就说明当前正处理的窗口的宿主Activity组件设置有切换动画。这个切换动画就是通过变量appTransformation所指向的一个Transformation对象来描述的，调用这个Transformation对象的成员函数getMatrix就可以获得一个对应的变换矩阵。将这个变换矩阵与变量tmpMatrix所描述的变换矩阵相乘，就可以得到一个中间变换矩阵，这是通过调用变量tmpMatrix所指向的一个Matrix对象的成员函数postConcat来实现的。

经过上面的5个步骤之后，最终得到的变换矩阵就保存在变量tmpMatrix中。由于这个变换矩阵是要设置到SurfaceFlinger服务中去的，因此就需要将这个变换矩阵转换为SurfaceFlinger服务所要求的格式。SurfaceFlinger服务所要求的变换矩阵的格式是由窗口在X轴和Y轴上的切变值以及缩放值来表示的，它们可以按照以下的步骤来获得：

调用变量tmpMatrix所描述的一个Matrix对象的成员函数getValues来获得一个数组，并且保存在变量tmpFloats中。

数组tmpFloats的第Matrix.MSKEW_X和Matrix.MSKEW_Y个位置的值就分别表示窗口在X轴和Y轴上的切变值，分别保存在WindowState类的成员变量mDtDx和mDsDy中。

数组tmpFloats的第Matrix.MSCALE_X和Matrix.MSCALE_Y个位置的值就分别表示窗口在X轴和Y轴上的缩放值，分别保存在WindowState类的成员变量mDsDx和mDtDy中。

获得了当前正在处理的窗口的变换矩阵之后，接下来还要计算当前正在处理的窗口接下来要显示的位置以及大小。

前面得到的数组tmpFloats中的第Matrix.MTRANS_X和Matrix.MTRANS_Y个位置的值分别表示窗口在X轴和Y轴上的偏移值，它们实际就是窗口接下来要显示的位置。从前面[Android窗口管理服务WindowManagerService对壁纸窗口（Wallpaper Window）的管理分析](http://blog.csdn.net/luoshengyang/article/details/8550820)一文可以知道，如果当前正在处理的是一个壁纸窗口，那么WindowState类的成员变量mXOffset和mYOffset描述的就是壁纸窗口相对其目标窗口的偏移值。对于普通的窗口来说，WindowState类的成员变量mXOffset和mYOffset的值等于0。因此，需要将保在在数组tmpFloats中的第Matrix.MTRANS_X和Matrix.MTRANS_Y个位置的值与WindowState类的成员变量mXOffset和mYOffset的值分别相加，才能得到当前正在处理接下来要显示在的位置。

由于前面获得的变换矩阵已经包含了当前正在处理的窗口的大小缩放因子，因此，我们就将当前正在处理的窗口的大小设置为它的实际大小即可。通过调用变量frame所指向的一个Frame对象的成员函数width和height可以获得当前正在处理的窗口的实际大小。

经过上面两步之后，当前正在处理的窗口接下来要显示的位置以及大小就计算完成了，其中，位置值保存在变量x和y中，而大小值保存在变量w和h，最终就可以将它们保存在WindowState类的成员变量mShownFrame所指向的一个Frame对象中。

如果当前正在处理的窗口没有动画可以显示，即变量selfTransformation的值等于false，并且变量attachedTransformation和appTransformation的值均等于null，那么WindowState类的成员函数computeShownFrameLocked的实现就简单了，它只要简单地将成员变量mFrame的内容设置到成员变量mShownFrame中，并且将成员变量mDsDx、mDtDx、mDsDy和mDtDy分别设置为1、0、0和1即可，表示当前正在处理的窗口既没有切变变换，也没有缩放变换。另外，如果WindowState类的成员变量mXOffset或者mYOffset的值不等于0，那么就需要将它们作来偏移值设置到成员变量mShownFrame所描述的一个Frame对象去，以便可以正确地计算出当前正在处理的窗口的位置。

至此，我们就分析完成窗口动画的显示过程了，整个WindowManagerService服务的分析也到此结束了。WindowManagerService服务可以说是整个Android应用程序框架层最为复杂的模块了，它与SurfaceFlinger服务一起为整个Android系统提供了UI服务，理解它对理解Android系统有着重要的意义。不过，要理解WindowManagerService服务的实现，是必须下些功夫的，同时也希望这个系列的文章能够帮助到大家。重新学习WindowManagerService服务请参考[Android窗口管理服务WindowManagerService的简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/8462738)一文。