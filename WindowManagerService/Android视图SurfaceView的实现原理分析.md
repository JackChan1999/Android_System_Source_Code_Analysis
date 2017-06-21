在[Android](http://lib.csdn.net/base/android)系统中，有一种特殊的视图，称为SurfaceView，它拥有独立的绘图表面，即它不与其宿主窗口共享同一个绘图表面。由于拥有独立的绘图表面，因此SurfaceView的UI就可以在一个独立的线程中进行绘制。又由于不会占用主线程资源，SurfaceView一方面可以实现复杂而高效的UI，另一方面又不会导致用户输入得不到及时响应。在本文中，我们就详细分析SurfaceView的实现原理。

《[android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

在前面[Android控件TextView的实现原理分析](http://blog.csdn.net/luoshengyang/article/details/8636153)一文中提到，普通的Android控件，例如TextView、Button和CheckBox等，它们都是将自己的UI绘制在宿主窗口的绘图表面之上，这意味着它们的UI是在应用程序的主线程中进行绘制的。由于应用程序的主线程除了要绘制UI之外，还需要及时地响应用户输入，否则的话，系统就会认为应用程序没有响应了，因此就会弹出一个ANR对话框出来。对于一些游戏画面，或者摄像头预览、视频播放来说，它们的UI都比较复杂，而且要求能够进行高效的绘制，因此，它们的UI就不适合在应用程序的主线程中进行绘制。这时候就必须要给那些需要复杂而高效UI的视图生成一个独立的绘图表面，以及使用一个独立的线程来绘制这些视图的UI。

在前面[Android应用程序与SurfaceFlinger服务的关系概述和学习计划](http://blog.csdn.net/luoshengyang/article/details/7846923)和[Android系统Surface机制的SurfaceFlinger服务简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/8010977)这两个系统的文章中，我们主要分析了Android应用程序窗口是如何通过SurfaceFlinger服务来绘制自己的UI的。一般来说，每一个窗口在SurfaceFlinger服务中都对应有一个Layer，用来描述它的绘图表面。对于那些具有SurfaceView的窗口来说，每一个SurfaceView在SurfaceFlinger服务中还对应有一个独立的Layer或者LayerBuffer，用来单独描述它的绘图表面，以区别于它的宿主窗口的绘图表面。

无论是LayerBuffer，还是Layer，它们都是以LayerBase为基类的，也就是说，SurfaceFlinger服务把所有的LayerBuffer和Layer都抽象为LayerBase，因此就可以用统一的流程来绘制和合成它们的UI。由于LayerBuffer的绘制和合成与Layer的绘制和合成是类似的，因此本文不打算对LayerBuffer的绘制和合成操作进行分析。需要深入理解LayerBuffer的绘制和合成操作的，可以参考[Android应用程序与SurfaceFlinger服务的关系概述和学习计划](http://blog.csdn.net/luoshengyang/article/details/7846923)和[Android系统Surface机制的SurfaceFlinger服务简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/8010977)这两个系统的文章。

为了接下来可以方便地描述SurfaceView的实现原理分析，我们假设在一个Activity窗口的视图结构中，除了有一个DecorView顶层视图之外，还有两个TextView控件，以及一个SurfaceView视图，这样该Activity窗口在SurfaceFlinger服务中就对应有两个Layer或者一个Layer的一个LayerBuffer，如图1所示：

![img](http://img.my.csdn.net/uploads/201303/11/1363016714_1787.jpg)

图1 SurfaceView及其宿主Activity窗口的绘图表面示意图

在图1中，Activity窗口的顶层视图DecorView及其两个TextView控件的UI都是绘制在SurfaceFlinger服务中的同一个Layer上面的，而SurfaceView的UI是绘制在SurfaceFlinger服务中的另外一个Layer或者LayerBuffer上的。

注意，用来描述SurfaceView的Layer或者LayerBuffer的Z轴位置是小于用来其宿主Activity窗口的Layer的Z轴位置的，但是前者会在后者的上面挖一个“洞”出来，以便它的UI可以对用户可见。实际上，SurfaceView在其宿主Activity窗口上所挖的“洞”只不过是在其宿主Activity窗口上设置了一块透明区域。

从总体上描述了SurfaceView的大致实现原理之后，接下来我们就详细分析它的具体实现过程，包括它的绘图表面的创建过程、在宿主窗口上面进行挖洞的过程，以及绘制过程。

## SurfaceView的绘图表面的创建过程

由于SurfaceView具有独立的绘图表面，因此，在它的UI内容可以绘制之前，我们首先要将它的绘图表面创建出来。尽管SurfaceView不与它的宿主窗口共享同一个绘图表面，但是它仍然是属于宿主窗口的视图结构的一个结点的，也就是说，SurfaceView仍然是会参与到宿主窗口的某些执行流程中去。

从前面[Android应用程序窗口（Activity）的绘图表面（Surface）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8303098)一文可以知道，每当一个窗口需要刷新UI时，就会调用ViewRoot类的成员函数performTraversals。ViewRoot类的成员函数performTraversals在执行的过程中，如果发现当前窗口的绘图表面还没有创建，或者发现当前窗口的绘图表面已经失效了，那么就会请求WindowManagerService服务创建一个新的绘图表面，同时，它还会通过一系列的回调函数来让嵌入在窗口里面的SurfaceView有机会创建自己的绘图表面。

接下来，我们就从ViewRoot类的成员函数performTraversals开始，分析SurfaceView的绘图表面的创建过程，如图2所示：

![img](http://img.my.csdn.net/uploads/201303/13/1363104461_5280.jpg)

图2 SurfaceView的绘图表面的创建过程

这个过程可以分为8个步骤，接下来我们就详细分析每一个步骤。

### Step 1. ViewRoot.performTraversals

```java
public final class ViewRoot extends Handler implements ViewParent,  
View.AttachInfo.Callbacks {  
    ......  

    private void performTraversals() {  
// cache mView since it is used so much below...  
final View host = mView;  
......  

final View.AttachInfo attachInfo = mAttachInfo;  

final int viewVisibility = getHostVisibility();  
boolean viewVisibilityChanged = mViewVisibility != viewVisibility  
        || mNewSurfaceNeeded;  
......  


if (mFirst) {  
    ......  

    if (!mAttached) {  
        host.dispatchAttachedToWindow(attachInfo, 0);  
        mAttached = true;  
    }  
      
    ......  
}  

......  

if (viewVisibilityChanged) {  
    ......  

    host.dispatchWindowVisibilityChanged(viewVisibility);  

    ......  
}  

......  

mFirst = false;  

......  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/[Java](http://lib.csdn.net/base/java)/android/view/ViewRoot.java中。

ViewRoot类的成员函数performTraversals的详细实现可以参考[Android应用程序窗口（Activity）的绘图表面（Surface）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8303098)和[Android窗口管理服务WindowManagerService计算Activity窗口大小的过程分析](http://blog.csdn.net/luoshengyang/article/details/8479101)这两篇文章，这里我们只关注与SurfaceView的绘图表面的创建相关的逻辑。

我们首先分析在ViewRoot类的成员函数performTraversals中四个相关的变量host、attachInfo、viewVisibility和viewVisibilityChanged。

变量host与ViewRoot类的成员变量mView指向的是同一个DecorView对象，这个DecorView对象描述的当前窗口的顶层视图。

变量attachInfo与ViewRoot类的成员变量mAttachInfo指向的是同一个AttachInfo对象。在Android系统中，每一个视图附加到它的宿主窗口的时候，都会获得一个AttachInfo对象，用来描述被附加的窗口的信息。

变量viewVisibility描述的是当前窗口的可见性。

变量viewVisibilityChanged描述的是当前窗口的可见性是否发生了变化。

ViewRoot类的成员变量mFirst表示当前窗口是否是第一次被刷新UI。如果是的话，那么它的值就会等于true，说明当前窗口的绘图表面还未创建。在这种情况下，如果ViewRoot类的另外一个成员变量mAttached的值也等于true，那么就表示当前窗口还没有将它的各个子视图附加到它的上面来。这时候ViewRoot类的成员函数performTraversals就会从当前窗口的顶层视图开始，通知每一个子视图它要被附加到宿主窗口上去了，这是通过调用变量host所指向的一个DecorView对象的成员函数dispatchAttachedToWindow来实现的。DecorView类的成员函数dispatchAttachedToWindow是从父类ViewGroup继承下来的，在后面的Step 2中，我们再详细分析ViewGroup类的成员数dispatchAttachedToWindow的实现。

接下来， ViewRoot类的成员函数performTraversals判断当前窗口的可见性是否发生了变化，即检查变量viewVisibilityChanged的值是否等于true。如果发生了变化，那么就会从当前窗口的顶层视图开始，通知每一个子视图它的宿主窗口的可见发生变化了，这是通过调用变量host所指向的一个DecorView对象的成员函数dispatchWindowVisibilityChanged来实现的。DecorView类的成员函数dispatchWindowVisibilityChanged是从父类ViewGroup继承下来的，在后面的Step 5中，我们再详细分析ViewGroup类的成员数dispatchWindowVisibilityChanged的实现。

我们假设当前窗口有一个SurfaceView，那么当该SurfaceView接收到它被附加到宿主窗口以及它的宿主窗口的可见性发生变化的通知时，就会相应地将自己的绘图表面创建出来。接下来，我们就分别分析ViewGroup类的成员数dispatchAttachedToWindow和dispatchWindowVisibilityChanged的实现，以便可以了解SurfaceView的绘图表面的创建过程。 

### Step 2. ViewGroup.dispatchAttachedToWindow

```java
public abstract class ViewGroup extends View implements ViewParent, ViewManager {  
    ......  

    // Child views of this ViewGroup  
    private View[] mChildren;  
    // Number of valid children in the mChildren array, the rest should be null or not  
    // considered as children  
    private int mChildrenCount;  
    ......  

    @Override  
    void dispatchAttachedToWindow(AttachInfo info, int visibility) {  
super.dispatchAttachedToWindow(info, visibility);  
visibility |= mViewFlags & VISIBILITY_MASK;  
final int count = mChildrenCount;  
final View[] children = mChildren;  
for (int i = 0; i < count; i++) {  
    children[i].dispatchAttachedToWindow(info, visibility);  
}  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/ViewGroup.java中。

ViewGroup类的成员变量mChildren保存的是当前正在处理的视图容器的子视图，而另外一个成员变量mChildrenCount保存的是这些子视图的数量。

ViewGroup类的成员函数dispatchAttachedToWindow的实现很简单，它只是简单地调用当前正在处理的视图容器的每一个子视图的成员函数dispatchAttachedToWindow，以便可以通知这些子视图，它们被附加到宿主窗口上去了。

当前正在处理的视图容器即为当前正在处理的窗口的顶层视图，由于前面我们当前正在处理的窗口有一个SurfaceView，因此这一步就会调用到该SurfaceView的成员函数dispatchAttachedToWindow。

由于SurfaceView类的成员函数dispatchAttachedToWindow是从父类View继承下来的，因此，接下来我们就继续分析View类的成员函数dispatchAttachedToWindow的实现。

### Step 3. View.dispatchAttachedToWindow

```java
public class View implements Drawable.Callback, KeyEvent.Callback, AccessibilityEventSource {  
    ......  

    AttachInfo mAttachInfo;  
    ......  

    void dispatchAttachedToWindow(AttachInfo info, int visibility) {  
//System.out.println("Attached! " + this);  
mAttachInfo = info;  
......  

onAttachedToWindow();  

......  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/View.java中。

View类的成员函数dispatchAttachedToWindow首先将参数info所指向的一个AttachInfo对象保存在自己的成员变量mAttachInfo中，以便当前视图可以获得其所附加在的窗口的相关信息，接下来再调用另外一个成员函数onAttachedToWindow来让子类有机会处理它被附加到宿主窗口的事件。

前面我们已经假设了当前处理的是一个SurfaceView。SurfaceView类重写了父类View的成员函数onAttachedToWindow，接下来我们就继续分析SurfaceView的成员函数onAttachedToWindow的实现，以便可以了解SurfaceView的绘图表面的创建过程。

### Step 4. SurfaceView.onAttachedToWindow

```java
public class SurfaceView extends View {  
    ......  

    IWindowSession mSession;  
    ......  

    @Override  
    protected void onAttachedToWindow() {  
super.onAttachedToWindow();  
mParent.requestTransparentRegion(this);  
mSession = getWindowSession();  
......  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/SurfaceView.java中。

SurfaceView类的成员函数onAttachedToWindow做了两件重要的事。

第一件事情是通知父视图，当前正在处理的SurfaceView需要在宿主窗口的绘图表面上挖一个洞，即需要在宿主窗口的绘图表面上设置一块透明区域。当前正在处理的SurfaceView的父视图保存在父类View的成员变量mParent中，通过调用这个成员变量mParent所指向的一个ViewGroup对象的成员函数requestTransparentRegion，就可以通知到当前正在处理的SurfaceView的父视图，当前正在处理的SurfaceView需要在宿主窗口的绘图表面上设置一块透明区域。在后面第2部分的内容中，我们再详细分析SurfaceView在宿主窗口的绘图表面的挖洞过程。

第二件事情是调用从父类View继承下来的成员函数getWindowSession来获得一个实现了IWindowSession接口的Binder代理对象，并且将该Binder代理对象保存在SurfaceView类的成员变量mSession中。从前面[Android应用程序窗口（Activity）实现框架简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/8170307)这个系列的文章可以知道，在Android系统中，每一个应用程序进程都有一个实现了IWindowSession接口的Binder代理对象，这个Binder代理对象是用来与WindowManagerService服务进行通信的，View类的成员函数getWindowSession返回的就是该Binder代理对象。在接下来的Step 8中，我们就可以看到，SurfaceView就可以通过这个实现了IWindowSession接口的Binder代理对象来请求WindowManagerService服务为自己创建绘图表面的。

这一步执行完成之后，返回到前面的Step 1中，即ViewRoot类的成员函数performTraversals中，我们假设当前窗口的可见性发生了变化，那么接下来就会调用顶层视图的成员函数dispatchWindowVisibilityChanged，以便可以通知各个子视图，它的宿主窗口的可见性发生化了。

窗口的顶层视图是使用DecorView类来描述的，而DecroView类的成员函数dispatchWindowVisibilityChanged是从父类ViewGroup类继承下来的，因此，接下来我们就继续分析GroupView类的成员函数dispatchWindowVisibilityChanged的实现，以便可以了解包含在当前窗口里面的一个SurfaceView的绘图表面的创建过程。

### Step 5. ViewGroup.dispatchWindowVisibilityChanged

```java
public abstract class ViewGroup extends View implements ViewParent, ViewManager {   
    ......  

    @Override  
    public void dispatchWindowVisibilityChanged(int visibility) {  
super.dispatchWindowVisibilityChanged(visibility);  
final int count = mChildrenCount;  
final View[] children = mChildren;  
for (int i = 0; i < count; i++) {  
    children[i].dispatchWindowVisibilityChanged(visibility);  
}  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/ViewGroup.java中。

ViewGroup类的成员函数dispatchWindowVisibilityChanged的实现很简单，它只是简单地调用当前正在处理的视图容器的每一个子视图的成员函数dispatchWindowVisibilityChanged，以便可以通知这些子视图，它们所附加在的宿主窗口的可见性发生变化了。

当前正在处理的视图容器即为当前正在处理的窗口的顶层视图，由于前面我们当前正在处理的窗口有一个SurfaceView，因此这一步就会调用到该SurfaceView的成员函数dispatchWindowVisibilityChanged。

由于SurfaceView类的成员函数dispatchWindowVisibilityChanged是从父类View继承下来的，因此，接下来我们就继续分析View类的成员函数dispatchWindowVisibilityChanged的实现。

### Step 6. View.dispatchWindowVisibilityChanged

```java
public class View implements Drawable.Callback, KeyEvent.Callback, AccessibilityEventSource {  
    ......  

    public void dispatchWindowVisibilityChanged(int visibility) {  
onWindowVisibilityChanged(visibility);  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/View.java中。

View类的成员函数dispatchWindowVisibilityChanged的实现很简单，它只是调用另外一个成员函数onWindowVisibilityChanged来让子类有机会处理它所附加在的宿主窗口的可见性变化事件。

前面我们已经假设了当前处理的是一个SurfaceView。SurfaceView类重写了父类View的成员函数onWindowVisibilityChanged，接下来我们就继续分析SurfaceView的成员函数onWindowVisibilityChanged的实现，以便可以了解SurfaceView的绘图表面的创建过程。

### Step 7. SurfaceView.onWindowVisibilityChanged

```java
public class SurfaceView extends View {  
    ......  

    boolean mRequestedVisible = false;  
    boolean mWindowVisibility = false;  
    boolean mViewVisibility = false;  
    .....  

    @Override  
    protected void onWindowVisibilityChanged(int visibility) {  
super.onWindowVisibilityChanged(visibility);  
mWindowVisibility = visibility == VISIBLE;  
mRequestedVisible = mWindowVisibility && mViewVisibility;  
updateWindow(false, false);  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/SurfaceView.java中。

SurfaceView类有三个用来描述可见性的成员变量mRequestedVisible、mWindowVisibility和mViewVisibility。其中，mWindowVisibility表示SurfaceView的宿主窗口的可见性，mViewVisibility表示SurfaceView自身的可见性。只有当mWindowVisibility和mViewVisibility的值均等于true的时候，mRequestedVisible的值才为true，表示SurfaceView是可见的。

参数visibility描述的便是当前正在处理的SurfaceView的宿主窗口的可见性，因此，SurfaceView类的成员函数onWindowVisibilityChanged首先将它记录在成员变量mWindowVisibility，接着再综合另外一个成员变量mViewVisibility来判断当前正在处理的SurfaceView是否是可见的，并且记录在成员变量mRequestedVisible中。

最后，SurfaceView类的成员函数onWindowVisibilityChanged就会调用另外一个成员函数updateWindow来更新当前正在处理的SurfaceView。在更新的过程中，如果发现当前正在处理的SurfaceView还没有创建绘图表面，那么就地请求WindowManagerService服务为它创建一个。

接下来，我们就继续分析SurfaceView类的成员函数updateWindow的实现，以便可以了解SurfaceView的绘图表面的创建过程。

### Step 8. SurfaceView.updateWindow

```java
public class SurfaceView extends View {  
    ......  

    final Surface mSurface = new Surface();  
    ......  

    MyWindow mWindow;  
    .....  

    int mWindowType = WindowManager.LayoutParams.TYPE_APPLICATION_MEDIA;  
    ......  

    int mRequestedType = -1;  
    ......  

    private void updateWindow(boolean force, boolean redrawNeeded) {  
if (!mHaveFrame) {  
    return;  
}  
......  

int myWidth = mRequestedWidth;  
if (myWidth <= 0) myWidth = getWidth();  
int myHeight = mRequestedHeight;  
if (myHeight <= 0) myHeight = getHeight();  

getLocationInWindow(mLocation);  
final boolean creating = mWindow == null;  
final boolean formatChanged = mFormat != mRequestedFormat;  
final boolean sizeChanged = mWidth != myWidth || mHeight != myHeight;  
final boolean visibleChanged = mVisible != mRequestedVisible  
        || mNewSurfaceNeeded;  
final boolean typeChanged = mType != mRequestedType;  
if (force || creating || formatChanged || sizeChanged || visibleChanged  
    || typeChanged || mLeft != mLocation[0] || mTop != mLocation[1]  
    || mUpdateWindowNeeded || mReportDrawNeeded || redrawNeeded) {  
    ......  

    try {  
        final boolean visible = mVisible = mRequestedVisible;  
        mLeft = mLocation[0];  
        mTop = mLocation[1];  
        mWidth = myWidth;  
        mHeight = myHeight;  
        mFormat = mRequestedFormat;  
        mType = mRequestedType;  
        ......  

        // Places the window relative  
        mLayout.x = mLeft;  
        mLayout.y = mTop;  
        mLayout.width = getWidth();  
        mLayout.height = getHeight();  
        ......  

        mLayout.memoryType = mRequestedType;  

        if (mWindow == null) {  
            mWindow = new MyWindow(this);  
            mLayout.type = mWindowType;  
            ......  
            mSession.addWithoutInputChannel(mWindow, mLayout,  
                    mVisible ? VISIBLE : GONE, mContentInsets);  
        }  
        ......  

        mSurfaceLock.lock();  
        try {  
            ......  

            final int relayoutResult = mSession.relayout(  
                mWindow, mLayout, mWidth, mHeight,  
                    visible ? VISIBLE : GONE, false, mWinFrame, mContentInsets,  
                    mVisibleInsets, mConfiguration, mSurface);  
            ......  

        } finally {  
            mSurfaceLock.unlock();  
        }  

        ......  
    } catch (RemoteException ex) {  
    }  

    .....  
}  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/SurfaceView.java中。

在分析SurfaceView类的成员函数updateWindow的实现之前，我们首先介绍一些相关的成员变量的含义，其中，mSurface、mWindow、mWindowType和mRequestedType这四个成员变量是最重要的。

SurfaceView类的成员变量mSurface指向的是一个Surface对象，这个Surface对象描述的便是SurfaceView专有的绘图表面。对于一般的视图来说，例如，TextView或者Button，它们是没有专有的绘图表面的，而是与专宿主窗口共享同一个绘图表面，因此，它们就不会像SurfaceView一样，有一个专门的类型为Surface的成员变量来描述自己的绘图表面。

在前面[Android应用程序窗口（Activity）与WindowManagerService服务的连接过程分析](http://blog.csdn.net/luoshengyang/article/details/8275938)一文提到，每一个Activity窗口都关联有一个W对象。这个W对象是一个实现了IWindow接口的Binder本地对象，它是用来传递给WindowManagerService服务的，以便WindowManagerService服务可以通过它来和它所关联的Activity窗口通信。例如，WindowManagerService服务通过这个W对象来通知它所关联的Activity窗口的大小或者可见性发生变化了。同时，这个W对象还用来在WindowManagerService服务这一侧唯一地标志一个窗口，也就是说，WindowManagerService服务会为这个W对象创建一个WindowState对象。

SurfaceView类的成员变量mWindow指向的是一个MyWindow对象。MyWindow类是从BaseIWindow类继承下来的，后者与W类一样，实现了IWindow接口。也就是说，每一个SurfaceView都关联有一个实现了IWindow接口的Binder本地对象，就如第一个Activity窗口都关联有一个实现了IWindow接口的W对象一样。从这里我们就可以推断出，每一个SurfaceView在WindowManagerService服务这一侧都对应有一个WindowState对象。从这一点来看，WindowManagerService服务认为Activity窗口和SurfaceView的地位是一样的，即认为它们都是一个窗口，并且具有绘图表面。接下来我们就会通过SurfaceView类的成员函数updateWindow的实现来证实这个推断。

SurfaceView类的成员变量mWindowType描述的是SurfaceView的窗口类型，它的默认值等于TYPE_APPLICATION_MEDIA。也就是说，我们在创建一个SurfaceView的时候，默认是用来显示多媒体的，例如，用来显示视频。SurfaceView还有另外一个窗口类型TYPE_APPLICATION_MEDIA_OVERLAY，它是用来在视频上面显示一个Overlay的，这个Overlay可以用来显示视字幕等信息。

我们假设一个Activity窗口嵌入有两个SurfaceView，其中一个SurfaceView的窗口类型为TYPE_APPLICATION_MEDIA，另外一个SurfaceView的窗口类型为TYPE_APPLICATION_MEDIA_OVERLAY，那么在WindowManagerService服务这一侧就会对应有三个WindowState对象，其中，用来描述SurfaceView的WindowState对象是附加在用来描述Activity窗口的WindowState对象上的。从前面[Android窗口管理服务WindowManagerService计算窗口Z轴位置的过程分析](http://blog.csdn.net/luoshengyang/article/details/8570428)一文可以知道，如果一个WindowState对象所描述的窗口的类型为TYPE_APPLICATION_MEDIA或者TYPE_APPLICATION_MEDIA_OVERLAY，那么它就会位于它所附加在的窗口的下面。也就是说，类型为TYPE_APPLICATION_MEDIA或者TYPE_APPLICATION_MEDIA_OVERLAY的窗口的Z轴位置是小于它所附加在的窗口的Z轴位置的。同时，如果一个窗口同时附加有类型为TYPE_APPLICATION_MEDIA和TYPE_APPLICATION_MEDIA_OVERLAY的两个窗口，那么类型为TYPE_APPLICATION_MEDIA_OVERLAY的窗口的Z轴大于类型为TYPE_APPLICATION_MEDIA的窗口的Z轴位置。

从上面的描述就可以得出一个结论：如果一个Activity窗口嵌入有两个类型分别为TYPE_APPLICATION_MEDIA和TYPE_APPLICATION_MEDIA_OVERLAY的SurfaceView，那么该Activity窗口的Z轴位置大于类型为TYPE_APPLICATION_MEDIA_OVERLAY的SurfaceView的Z轴位置，而类型为TYPE_APPLICATION_MEDIA_OVERLAY的SurfaceView的Z轴位置又大于类型为TYPE_APPLICATION_MEDIA的窗口的Z轴位置。

注意，我们在创建了一个SurfaceView之后，可以调用它的成员函数setZOrderMediaOverlay、setZOrderOnTop或者setWindowType来修改该SurfaceView的窗口类型，也就是修改该SurfaceView的成员变量mWindowType的值。

SurfaceView类的成员变量mRequestedType描述的是SurfaceView的绘图表面的类型，一般来说，它的值可能等于SURFACE_TYPE_NORMAL，也可能等于SURFACE_TYPE_PUSH_BUFFERS。

当一个SurfaceView的绘图表面的类型等于SURFACE_TYPE_NORMAL的时候，就表示该SurfaceView的绘图表面所使用的内存是一块普通的内存。一般来说，这块内存是由SurfaceFlinger服务来分配的，我们可以在应用程序内部自由地访问它，即可以在它上面填充任意的UI数据，然后交给SurfaceFlinger服务来合成，并且显示在屏幕上。在这种情况下，SurfaceFlinger服务使用一个Layer对象来描述该SurfaceView的绘图表面。

当一个SurfaceView的绘图表面的类型等于SURFACE_TYPE_PUSH_BUFFERS的时候，就表示该SurfaceView的绘图表面所使用的内存不是由SurfaceFlinger服务分配的，因而我们不能够在应用程序内部对它进行操作。例如，当一个SurfaceView是用来显示摄像头预览或者视频播放的时候，我们就会将它的绘图表面的类型设置为SURFACE_TYPE_PUSH_BUFFERS，这样摄像头服务或者视频播放服务就会为该SurfaceView绘图表面创建一块内存，并且将采集的预览图像数据或者视频帧数据源源不断地填充到该内存中去。注意，这块内存有可能是来自专用的硬件的，例如，它可能是来自视频卡的。在这种情况下，SurfaceFlinger服务使用一个LayerBuffer对象来描述该SurfaceView的绘图表面。

从上面的描述就得到一个重要的结论：绘图表面类型为SURFACE_TYPE_PUSH_BUFFERS的SurfaceView的UI是不能由应用程序来控制的，而是由专门的服务来控制的，例如，摄像头服务或者视频播放服务，同时，SurfaceFlinger服务会使用一种特殊的LayerBuffer来描述这种绘图表面。使用LayerBuffer来描述的绘图表面在进行渲染的时候，可以使用硬件加速，例如，使用copybit或者overlay来加快渲染速度，从而可以获得更流畅的摄像头预览或者视频播放。

注意，我们在创建了一个SurfaceView之后，可以调用它的成员函数getHolder获得一个SurfaceHolder对象，然后再调用该SurfaceHolder对象的成员函数setType来修改该SurfaceView的绘图表面的类型，即修改该SurfaceView的成员变量mRequestedType的值。

介绍完成SurfaceView类的成员变量mSurface、mWindow、mWindowType和mRequestedType的含义之后，我们再介绍其它几个接下来要用到的其它成员变量的含义：

| 成员变量                      | 含义                                       |
| :------------------------ | :--------------------------------------- |
| mHaveFrame                | 用来描述SurfaceView的宿主窗口的大小是否已经计算好了。只有当宿主窗口的大小计算之后，SurfaceView才可以更新自己的窗口。 |
| mRequestedWidth           | 用来描述SurfaceView最后一次被请求的宽度。               |
| mRequestedHeight          | 用来描述SurfaceView最后一次被请求的高度。               |
| mRequestedFormat          | 用来描述SurfaceView最后一次被请求的绘图表面的像素格式。        |
| mNewSurfaceNeeded         | 用来描述SurfaceView是否需要新创建一个绘图表面。            |
| mLeft、mTop、mWidth、mHeight | 用来描述SurfaceView上一次所在的位置以及大小。             |
| mFormat                   | 用来描述SurfaceView的绘图表面上一次所设置的格式。           |
| mVisible                  | 用来描述SurfaceView上一次被设置的可见性。               |
| mType                     | 用来描述SurfaceView的绘图表面上一次所设置的类型。           |
| mUpdateWindowNeeded       | 用来描述SurfaceView是否被WindowManagerService服务通知执行一次UI更新操作。 |
| mReportDrawNeeded         | 用来描述SurfaceView是否被WindowManagerService服务通知执行一次UI绘制操作。 |
| mLayout                   | 指向的是一个WindowManager.LayoutParams对象，用来传递SurfaceView的布局参数以及属性值给WindowManagerService服务，以便WindowManagerService服务可以正确地维护它的状态。 |

理解了上述成员变量的含义的之后，接下来我们就可以分析SurfaceView类的成员函数updateWindow创建绘图表面的过程了，如下所示：

(1). 判断成员变量mHaveFrame的值是否等于false。如果是的话，那么就说明现在还不是时候为SurfaceView创建绘图表面，因为它的宿主窗口还没有准备就绪。

(2). 获得SurfaceView当前要使用的宽度和高度，并且保存在变量myWidth和myHeight中。注意，如果SurfaceView没有被请求设置宽度或者高度，那么就通过调用父类View的成员函数getWidth和getHeight来获得它默认所使用的宽度和高度。

(3). 调用父类View的成员函数getLocationInWindow来获得SurfaceView的左上角位置，并且保存在成员变量mLocation所描述的一个数组中。

(4). 判断以下条件之一是否成立：

- SurfaceView的绘图表面是否还未创建，即成员变量mWindow的值是否等于null；

- SurfaceView的绘图表面的像素格式是否发生了变化，即成员变量mFormat和mRequestedFormat的值是否不相等；

- SurfaceView的大小是否发生了变化，即变量myWidth和myHeight是否与成员变量mWidth和mHeight的值不相等；

- SurfaceView的可见性是否发生了变化，即成员变量mVisible和mRequestedVisible的值是否不相等，或者成员变量NewSurfaceNeeded的值是否等于true；

- SurfaceView的绘图表面的类型是否发生了变化，即成员变量mType和mRequestedType的值是否不相等；

- SurfaceView的位置是否发生了变化，即成员变量mLeft和mTop的值是否不等于前面计算得到的mLocation[0]和mLocation[1]的值；

- SurfaceView是否被WindowManagerService服务通知执行一次UI更新操作，即成员变量mUpdateWindowNeeded的值是否等于true；

- SurfaceView是否被WindowManagerService服务通知执行一次UI绘制操作，即成员变量mReportDrawNeeded的值是否等于true；

- SurfaceView类的成员函数updateWindow是否被调用者强制要求刷新或者绘制SurfaceView，即参数force或者redrawNeeded的值是否等于true。

只要上述条件之一成立，那么SurfaceView类的成员函数updateWindow就需要对SurfaceView的各种信息进行更新，即执行以下第5步至第7步操作。

(5). 将SurfaceView接下来要设置的可见性、位置、大小、绘图表面像素格式和类型分别记录在成员变量mVisible、mLeft、mTop、mWidth、mHeight、mFormat和mType，同时还会将这些信息整合到成员变量mLayout所指向的一个WindowManager.LayoutParams对象中去，以便接下来可以传递给WindowManagerService服务。

(6). 检查成员变量mWindow的值是否等于null。如果等于null的话，那么就说明该SurfaceView还没有增加到WindowManagerService服务中去。在这种情况下，就会创建一个MyWindow对象保存在该成员变量中，并且调用成员变量mSession所描述的一个Binder代理对象的成员函数addWithoutInputChannel来将该MyWindow对象传递给WindowManagerService服务。在前面的Step 4中提到，SurfaceView类的成员变量mSession指向的是一个实现了IWindowSession接口的Binder代理对象，该Binder代理对象引用的是运行在WindowManagerService服务这一侧的一个Session对象。Session类的成员函数addWithoutInputChannel与另外一个成员函数add的实现是类似的，它们都是用来在WindowManagerService服务内部为指定的窗口增加一个WindowState对象，具体可以参考前面[Android应用程序窗口（Activity）与WindowManagerService服务的连接过程分析](http://blog.csdn.net/luoshengyang/article/details/8275938)一文。不过，Session类的成员函数addWithoutInputChannel只是在WindowManagerService服务内部为指定的窗口增加一个WindowState对象，而Session类的成员函数add除了会在WindowManagerService服务内部为指定的窗口增加一个WindowState对象之外，还会为该窗口创建一个用来接收用户输入的通道，具体可以参考[Android应用程序键盘（Keyboard）消息处理机制分析](http://blog.csdn.net/luoshengyang/article/details/6882903)一文。

(7). 调用成员变量mSession所描述的一个Binder代理对象的成员函数relayout来请求WindowManagerService服务对SurfaceView的UI进行布局。从前面[Android应用程序窗口（Activity）的绘图表面（Surface）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8303098)一文可以知道，WindowManagerService服务在对一个窗口进行布局的时候，如果发现该窗口的绘制表面还未创建，或者需要需要重新创建，那么就会为请求SurfaceFlinger服务为该窗口创建一个新的绘图表面，并且将该绘图表面返回来给调用者。在我们这个情景中，WindowManagerService服务返回来的绘图表面就会保存在成员变量mSurface。注意，这一步由于可能会修改SurfaceView的绘图表面，即修改成员变量mSurface的指向的一个Surface对象的内容，因此，就需要在获得成员变量mSurfaceLock所描述的一个锁的情况下执行，避免其它线程同时修改该绘图表面的内容，这是因为我们可能会使用一个独立的线程来来绘制SurfaceView的UI。

执行完成上述步骤之后，SurfaceView的绘图表面的创建操作就执行完成了，而当SurfaceView有了绘图表面之后，我们就可以使用独立的线程来绘制它的UI了，不过，在绘制之前，我们还需要在SurfaceView的宿主窗口上挖一个洞，以便绘制出来的UI不会被挡住。

## SurfaceView的挖洞过程

SurfaceView的窗口类型一般都是TYPE_APPLICATION_MEDIA或者TYPE_APPLICATION_MEDIA_OVERLAY，也就是说，它的Z轴位置是小于其宿主窗口的Z位置的。为了保证SurfaceView的UI是可见的，SurfaceView就需要在其宿主窗口的上面挖一个洞出来，实际上就是在其宿主窗口的绘图表面上设置一块透明区域，以便可以将自己显示出来。

从SurfaceView的绘图表面的创建过程可以知道，SurfaceView在被附加到宿主窗口之上的时候，会请求在宿主窗口上设置透明区域，而每当其宿主窗口刷新自己的UI的时候，就会将所有嵌入在它里面的SurfaceView所设置的透明区域收集起来，然后再通知WindowManagerService服务为其设置一个总的透明区域。

从SurfaceView的绘图表面的创建过程可以知道，SurfaceView在被附加到宿主窗口之上的时候，SurfaceView类的成员函数onAttachedToWindow就会被调用。SurfaceView类的成员函数onAttachedToWindow在被调用的期间，就会请求在宿主窗口上设置透明区域。接下来，我们就从SurfaceView类的成员函数onAttachedToWindow开始，分析SurfaceView的挖洞过程，如图3所示：

![img](http://img.my.csdn.net/uploads/201303/15/1363284472_6794.jpg)

图3 SurfaceView的挖洞过程

这个过程可以分为6个步骤，接下来我们就详细分析每一个步骤。

### Step 1. SurfaceView.onAttachedToWindow

```java
public class SurfaceView extends View {  
    ......  

    @Override  
    protected void onAttachedToWindow() {  
super.onAttachedToWindow();  
mParent.requestTransparentRegion(this);  
......  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/SurfaceView.java中。

SurfaceView类的成员变量mParent是从父类View继承下来的，用来描述当前正在处理的SurfaceView的父视图。我们假设当前正在处理的SurfaceView的父视图就为其宿主窗口的顶层视图，因此，接下来SurfaceView类的成员函数onAttachedToWindow就会调用DecorView类的成员函数requestTransparentRegion来请求在宿主窗口之上挖一个洞。

DecorView类的成员函数requestTransparentRegion是从父类ViewGroup继承下来的，因此，接下来我们就继续分析ViewGroup类的成员函数requestTransparentRegion的实现。

### Step 2. ViewGroup.requestTransparentRegion

```java
public abstract class ViewGroup extends View implements ViewParent, ViewManager {  
    ......  

    public void requestTransparentRegion(View child) {  
if (child != null) {  
    child.mPrivateFlags |= View.REQUEST_TRANSPARENT_REGIONS;  
    if (mParent != null) {  
        mParent.requestTransparentRegion(this);  
    }  
}  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/ViewGroup.java中。

参数child描述的便是要在宿主窗口设置透明区域的SurfaceView，ViewGroup类的成员函数requestTransparentRegion首先将它的成员变量mPrivateFlags的值的View.REQUEST_TRANSPARENT_REGIONS位设置为1，表示它要在宿主窗口上设置透明区域，接着再调用从父类View继承下来的成员变量mParent所指向的一个视图容器的成员函数requestTransparentRegion来继续向上请求设置透明区域，这个过程会一直持续到当前正在处理的视图容器为窗口的顶层视图为止。

前面我们已经假设了参数child所描述的SurfaceView是直接嵌入在宿主窗口的顶层视图中的，而窗口的顶层视图的父视图是使用一个ViewRoot对象来描述的，也就是说，当前正在处理的视图容器的成员变量mParent指向的是一个ViewRoot对象，因此，接下来我们就继续分析ViewRoot类的成员函数requestTransparentRegion的实现，以便可以继续了解SurfaceView的挖洞过程。

### Step 3. ViewRoot.requestTransparentRegion

```java
public final class ViewRoot extends Handler implements ViewParent,  
View.AttachInfo.Callbacks {  
    ......  

    public void requestTransparentRegion(View child) {  
// the test below should not fail unless someone is messing with us  
checkThread();  
if (mView == child) {  
    mView.mPrivateFlags |= View.REQUEST_TRANSPARENT_REGIONS;  
    // Need to make sure we re-evaluate the window attributes next  
    // time around, to ensure the window has the correct format.  
    mWindowAttributesChanged = true;  
    requestLayout();  
}  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/ViewRoot.java中。

ViewRoot类的成员函数requestTransparentRegion首先调用另外一个成员函数checkThread来检查当前执行的线程是否是应用程序的主线程，如果不是的话，那么就会抛出一个类型为CalledFromWrongThreadException的异常。

通过了上面的检查之后，ViewRoot类的成员函数requestTransparentRegion再检查参数child所描述的视图是否就是当前正在处理的ViewRoot对象所关联的窗口的顶层视图，即检查它与ViewRoot类的成员变量mView是否是指向同一个View对象。由于一个ViewRoot对象有且仅有一个子视图，因此，如果上述检查不通过的话，那么就说明调用者正在非法调用ViewRoot类的成员函数requestTransparentRegion来设置透明区域。

通过了上述两个检查之后，ViewRoot类的成员函数requestTransparentRegion就将成员变量mView所描述的一个窗口的顶层视图的成员变量mPrivateFlags的值的View.REQUEST_TRANSPARENT_REGIONS位设置为1，表示该窗口被设置了一块透明区域。

当一个窗口被请求设置了一块透明区域之后，它的窗口属性就发生变化了，因此，这时候除了要将与它所关联的一个ViewRoot对象的成员变量mWindowAttributesChanged的值设置为true之外，还要调用该ViewRoot对象的成员函数requestLayout来请求刷新一下窗口的UI，即请求对窗口的UI进行重新布局和绘制。

从前面[Android应用程序窗口（Activity）的绘图表面（Surface）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8303098)一文可以知道，ViewRoot类的成员函数requestLayout最终会调用到另外一个成员函数performTraversals来实际执行刷新窗口UI的操作。ViewRoot类的成员函数performTraversals在刷新窗口UI的过程中，就会将嵌入在它里面的SurfaceView所要设置的透明区域收集起来，以便可以请求WindowManagerService将这块透明区域设置到它的绘图表面上去。

接下来，我们就继续分析ViewRoot类的成员函数performTraversals的实现，以便可以继续了解SurfaceView的挖洞过程。

### Step 4. ViewRoot.performTraversals

```java
public final class ViewRoot extends Handler implements ViewParent,  
View.AttachInfo.Callbacks {  
    ......  

    private void performTraversals() {  
......  

// cache mView since it is used so much below...  
final View host = mView;  
......  

final boolean didLayout = mLayoutRequested;  
......  

if (didLayout) {  
    ......  

    host.layout(0, 0, host.mMeasuredWidth, host.mMeasuredHeight);  

    ......  

    if ((host.mPrivateFlags & View.REQUEST_TRANSPARENT_REGIONS) != 0) {  
        // start out transparent  
        // TODO: AVOID THAT CALL BY CACHING THE RESULT?  
        host.getLocationInWindow(mTmpLocation);  
        mTransparentRegion.set(mTmpLocation[0], mTmpLocation[1],  
                mTmpLocation[0] + host.mRight - host.mLeft,  
                mTmpLocation[1] + host.mBottom - host.mTop);  

        host.gatherTransparentRegion(mTransparentRegion);  
        ......  

        if (!mTransparentRegion.equals(mPreviousTransparentRegion)) {  
            mPreviousTransparentRegion.set(mTransparentRegion);  
            // reconfigure window manager  
            try {  
                sWindowSession.setTransparentRegion(mWindow, mTransparentRegion);  
            } catch (RemoteException e) {  
            }  
        }  
    }  

    ......  
}  

......  

boolean cancelDraw = attachInfo.mTreeObserver.dispatchOnPreDraw();  

if (!cancelDraw && !newSurface) {  
    ......  

    draw(fullRedrawNeeded);  

    ......  
}   

......  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/ViewRoot.java中。

ViewRoot类的成员函数performTraversals的具体实现可以参考前面[Android应用程序窗口（Activity）实现框架简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/8170307)这个系列的文章以及[Android窗口管理服务WindowManagerService计算Activity窗口大小的过程分析](http://blog.csdn.net/luoshengyang/article/details/8479101)一文，这里我们只关注窗口收集透明区域的逻辑。

ViewRoot类的成员函数performTraversals是在窗口的UI布局完成之后，并且在窗口的UI绘制之前，收集嵌入在它里面的SurfaceView所设置的透明区域的，这是因为窗口的UI布局完成之后，各个子视图的大小和位置才能确定下来，这样SurfaceView才知道自己要设置的透明区域的位置和大小。

变量host与ViewRoot类的成员变量mView指向的是同一个DecorView对象，这个DecorView对象描述的便是当前正在处理的窗口的顶层视图。从前面的Step 3可以知道，如果当前正在处理的窗口的顶层视图内嵌有SurfaceView，那么用来描述它的一个DecorView对象的成员变量mPrivateFlags的值的View.REQUEST_TRANSPARENT_REGIONS位就会等于1。在这种情况下，ViewRoot类的成员函数performTraversals就知道需要在当前正在处理的窗口的上面设置一块透明区域了。这块透明区域的收集过程如下所示：

(1). 计算顶层视图的位置和大小，即计算顶层视图所占据的区域。

(2). 将顶层视图所占据的区域作为窗口的初始化透明区域，保存在ViewRoot类的成员变量mTransparentRegion中。

(3). 从顶层视图开始，从上到下收集每一个子视图所要设置的区域，最终收集到的总透明区域也是保存在ViewRoot类的成员变量mTransparentRegion中。

(4). 检查ViewRoot类的成员变量mTransparentRegion和mPreviousTransparentRegion所描述的区域是否相等。如果不相等的话，那么就说明窗口的透明区域发生了变化，这时候就需要调用ViewRoot类的的静态成员变量sWindowSession所描述的一个Binder代理对象的成员函数setTransparentRegion通知WindowManagerService为窗口设置由成员变量mTransparentRegion所指定的透明区域。

其中，第(3)步是通过调用变量host所描述的一个DecorView对象的成员函数gatherTransparentRegion来实现的。 DecorView类的成员函数gatherTransparentRegion是从父类ViewGroup继承下来的，因此，接下来我们就继续分析ViewGroup类的成员函数gatherTransparentRegion的实现，以便可以了解SurfaceView的挖洞过程。

### Step 5. ViewGroup.gatherTransparentRegion

```java
public abstract class ViewGroup extends View implements ViewParent, ViewManager {  
    ......  

    @Override  
    public boolean gatherTransparentRegion(Region region) {  
// If no transparent regions requested, we are always opaque.  
final boolean meOpaque = (mPrivateFlags & View.REQUEST_TRANSPARENT_REGIONS) == 0;  
if (meOpaque && region == null) {  
    // The caller doesn't care about the region, so stop now.  
    return true;  
}  
super.gatherTransparentRegion(region);  
final View[] children = mChildren;  
final int count = mChildrenCount;  
boolean noneOfTheChildrenAreTransparent = true;  
for (int i = 0; i < count; i++) {  
    final View child = children[i];  
    if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {  
        if (!child.gatherTransparentRegion(region)) {  
            noneOfTheChildrenAreTransparent = false;  
        }  
    }  
}  
return meOpaque || noneOfTheChildrenAreTransparent;  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/ViewGroup.java中。

ViewGroup类的成员函数gatherTransparentRegion首先是检查当前正在处理的视图容器是否被请求设置透明区域，即检查成员变量mPrivateFlags的值的 View.REQUEST_TRANSPARENT_REGIONS位是否等于1。如果不等于1，那么就说明不用往下继续收集窗口的透明区域了，因为在这种情况下，当前正在处理的视图容器及其子视图都不可能设置有透明区域。另一方面，如果参数region的值等于null，那么就说明调用者不关心当前正在处理的视图容器的透明区域，而是关心它是透明的，还是不透明的。在上述两种情况下，ViewGroup类的成员函数gatherTransparentRegion都不用进一步处理了。

假设当前正在处理的视图容器被请求设置有透明区域，并且参数region的值不等于null，那么接下来ViewGroup类的成员函数gatherTransparentRegion就执行以下两个操作：

(1). 调用父类View的成员函数gatherTransparentRegion来检查当前正在处理的视图容器是否需要绘制。如果需要绘制的话，那么就会将它所占据的区域从参数region所占据的区域移除，这是因为参数region所描述的区域开始的时候是等于窗口的顶层视图的大小的，也就是等于窗口的整个大小的。

(2). 调用当前正在处理的视图容器的每一个子视图的成员函数gatherTransparentRegion来继续往下收集透明区域。

在接下来的Step 6中，我们再详细分析当前正在处理的视图容器的每一个子视图的透明区域的收集过程，现在我们主要分析View类的成员函数gatherTransparentRegion的实现，如下所示：

```java
public class View implements Drawable.Callback, KeyEvent.Callback, AccessibilityEventSource {  
    ......  

    public boolean gatherTransparentRegion(Region region) {  
final AttachInfo attachInfo = mAttachInfo;  
if (region != null && attachInfo != null) {  
    final int pflags = mPrivateFlags;  
    if ((pflags & SKIP_DRAW) == 0) {  
        // The SKIP_DRAW flag IS NOT set, so this view draws. We need to  
        // remove it from the transparent region.  
        final int[] location = attachInfo.mTransparentLocation;  
        getLocationInWindow(location);  
        region.op(location[0], location[1], location[0] + mRight - mLeft,  
                location[1] + mBottom - mTop, Region.Op.DIFFERENCE);  
    } else if ((pflags & ONLY_DRAWS_BACKGROUND) != 0 && mBGDrawable != null) {  
        // The ONLY_DRAWS_BACKGROUND flag IS set and the background drawable  
        // exists, so we remove the background drawable's non-transparent  
        // parts from this transparent region.  
        applyDrawableToTransparentRegion(mBGDrawable, region);  
    }  
}  
return true;  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/View.java中。

View类的成员函数gatherTransparentRegion首先是检查当前正在处理的视图的前景是否需要绘制，即检查成员变量mPrivateFlags的值的SKIP_DRAW位是否等于0。如果等于0的话，那么就说明当前正在处理的视图的前景是需要绘制的。在这种情况下，View类的成员函数gatherTransparentRegion就会将当前正在处理的视图所占据的区域从参数region所描述的区域中移除，以便当前正在处理的视图的前景可以显示出来。

另一方面，如果当前正在处理的视图的前景不需要绘制，但是该视图的背景需要绘制，并且该视图是设置有的，即成员变量mPrivateFlags的值的SKIP_DRAW位不等于0，并且成员变量mBGDrawable的值不等于null，这时候View类的成员函数gatherTransparentRegion就会调用另外一个成员函数applyDrawableToTransparentRegion来将该背景中的不透明区域从参数region所描述的区域中移除，以便当前正在处理的视图的背景可以显示出来。

回到ViewGroup类的成员函数gatherTransparentRegion中，当前正在处理的视图容器即为当前正在处理的窗口的顶层视图，前面我们已经假设它里面嵌入有一个SurfaceView子视图，因此，接下来就会收集该SurfaceView子视图所设置的透明区域，这是通过调用SurfaceView类的成员函数gatherTransparentRegion来实现的。

接下来，我们就继续分析SurfaceView类的成员函数gatherTransparentRegion的实现，以便可以继续了解SurfaceView的挖洞过程。

### Step 6. SurfaceView.gatherTransparentRegion

```java
public class SurfaceView extends View {  
    ......  

    @Override  
    public boolean gatherTransparentRegion(Region region) {  
if (mWindowType == WindowManager.LayoutParams.TYPE_APPLICATION_PANEL) {  
    return super.gatherTransparentRegion(region);  
}  

boolean opaque = true;  
if ((mPrivateFlags & SKIP_DRAW) == 0) {  
    // this view draws, remove it from the transparent region  
    opaque = super.gatherTransparentRegion(region);  
} else if (region != null) {  
    int w = getWidth();  
    int h = getHeight();  
    if (w>0 && h>0) {  
        getLocationInWindow(mLocation);  
        // otherwise, punch a hole in the whole hierarchy  
        int l = mLocation[0];  
        int t = mLocation[1];  
        region.op(l, t, l+w, t+h, Region.Op.UNION);  
    }  
}  
if (PixelFormat.formatHasAlpha(mRequestedFormat)) {  
    opaque = false;  
}  
return opaque;  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/SurfaceView.java中。

SurfaceView类的成员函数gatherTransparentRegion首先是检查当前正在处理的SurfaceView是否是用作窗口面板的，即它的成员变量mWindowType的值是否等于WindowManager.LayoutParams.TYPE_APPLICATION_PANEL。如果等于的话，那么就会调用父类View的成员函数gatherTransparentRegion来检查该面板是否需要绘制。如果需要绘制，那么就会将它所占据的区域从参数region所描述的区域移除。

假设当前正在处理的SurfaceView不是用作窗口面板的，那么SurfaceView类的成员函数gatherTransparentRegion接下来就会直接检查当前正在处理的SurfaceView是否是需要在宿主窗口的绘图表面上进行绘制，即检查成员变量mPrivateFlags的值的SKIP_DRAW位是否等于1。如果需要的话，那么也会调用父类View的成员函数gatherTransparentRegion来将它所占据的区域从参数region所描述的区域移除。

假设当前正在处理的SurfaceView不是用作窗口面板，并且也是不需要在宿主窗口的绘图表面上进行绘制的，而参数region的值又不等于null，那么SurfaceView类的成员函数gatherTransparentRegion就会先计算好当前正在处理的SurfaceView所占据的区域，然后再将该区域添加到参数region所描述的区域中去，这样就可以得到窗口的一个新的透明区域。

最后，SurfaceView类的成员函数gatherTransparentRegion判断当前正在处理的SurfaceView的绘图表面的像素格式是否设置有透明值。如果有的话，那么就会将变量opaque的值设置为false，否则的话，变量opaque的值就保持为true。变量opaque的值最终会返回给调用者，这样调用者就可以知道当前正在处理的SurfaceView的绘图表面是否是半透明的了。

至此，我们就分析完成SurfaceView的挖洞过程了，接下来我们继续分析SurfaceView的绘制过程。

## SurfaceView的绘制过程

SurfaceView虽然具有独立的绘图表面，不过它仍然是宿主窗口的视图结构中的一个结点，因此，它仍然是可以参与到宿主窗口的绘制流程中去的。从前面[Android应用程序窗口（Activity）的测量（Measure）、布局（Layout）和绘制（Draw）过程分析](http://blog.csdn.net/luoshengyang/article/details/8372924)一文可以知道，窗口在绘制的过程中，每一个子视图的成员函数draw或者dispatchDraw都会被调用到，以便它们可以绘制自己的UI。

SurfaceView类的成员函数draw和dispatchDraw的实现如下所示：

```java
public class SurfaceView extends View {  
    ......  

    @Override  
    public void draw(Canvas canvas) {  
if (mWindowType != WindowManager.LayoutParams.TYPE_APPLICATION_PANEL) {  
    // draw() is not called when SKIP_DRAW is set  
    if ((mPrivateFlags & SKIP_DRAW) == 0) {  
        // punch a whole in the view-hierarchy below us  
        canvas.drawColor(0, PorterDuff.Mode.CLEAR);  
    }  
}  
super.draw(canvas);  
    }  

    @Override  
    protected void dispatchDraw(Canvas canvas) {  
if (mWindowType != WindowManager.LayoutParams.TYPE_APPLICATION_PANEL) {  
    // if SKIP_DRAW is cleared, draw() has already punched a hole  
    if ((mPrivateFlags & SKIP_DRAW) == SKIP_DRAW) {  
        // punch a whole in the view-hierarchy below us  
        canvas.drawColor(0, PorterDuff.Mode.CLEAR);  
    }  
}  
// reposition ourselves where the surface is   
mHaveFrame = true;  
updateWindow(false, false);  
super.dispatchDraw(canvas);  
    }  

    ......  
}  
```
这两个函数定义在文件frameworks/base/core/java/android/view/SurfaceView.java中。

SurfaceView类的成员函数draw和dispatchDraw的参数canvas所描述的都是建立在宿主窗口的绘图表面上的画布，因此，在这块画布上绘制的任何UI都是出现在宿主窗口的绘图表面上的。

本来SurfaceView类的成员函数draw是用来将自己的UI绘制在宿主窗口的绘图表面上的，但是这里我们可以看到，如果当前正在处理的SurfaceView不是用作宿主窗口面板的时候，即其成员变量mWindowType的值不等于WindowManager.LayoutParams.TYPE_APPLICATION_PANEL的时候，SurfaceView类的成员函数draw只是简单地将它所占据的区域绘制为黑色。

本来SurfaceView类的成员函数dispatchDraw是用来绘制SurfaceView的子视图的，但是这里我们同样看到，如果当前正在处理的SurfaceView不是用作宿主窗口面板的时候，那么SurfaceView类的成员函数dispatchDraw只是简单地将它所占据的区域绘制为黑色，同时，它还会通过调用另外一个成员函数updateWindow更新自己的UI，实际上就是请求WindowManagerService服务对自己的UI进行布局，以及创建绘图表面，具体可以参考前面第1部分的内容。

从SurfaceView类的成员函数draw和dispatchDraw的实现就可以看出，SurfaceView在其宿主窗口的绘图表面上面所做的操作就是将自己所占据的区域绘为黑色，除此之外，就没有其它更多的操作了，这是因为SurfaceView的UI是要展现在它自己的绘图表面上面的。接下来我们就分析如何在SurfaceView的绘图表面上面进行UI绘制。

从前面[Android应用程序窗口（Activity）的测量（Measure）、布局（Layout）和绘制（Draw）过程分析](http://blog.csdn.net/luoshengyang/article/details/8372924)一文可以知道，如果要在一个绘图表面进行UI绘制，那么就顺序执行以下的操作：

(1). 在绘图表面的基础上建立一块画布，即获得一个Canvas对象。

(2). 利用Canvas类提供的绘图接口在前面获得的画布上绘制任意的UI。

(3). 将已经填充好了UI数据的画布缓冲区提交给SurfaceFlinger服务，以便SurfaceFlinger服务可以将它合成到屏幕上去。

SurfaceView提供了一个SurfaceHolder接口，通过这个SurfaceHolder接口就可以执行上述的第(1)和引(3)个操作，示例代码如下所示：

```java
SurfaceView sv = (SurfaceView )findViewById(R.id.surface_view);  

SurfaceHolder sh = sv.getHolder();  

Cavas canvas = sh.lockCanvas()  

//Draw something on canvas  
......  

sh.unlockCanvasAndPost(canvas);  
```
注意，只有在一个SurfaceView的绘图表面的类型不是SURFACE_TYPE_PUSH_BUFFERS的时候，我们才可以自由地在上面绘制UI。我们使用SurfaceView来显示摄像头预览或者播放视频时，一般就是会将它的绘图表面的类型设置为SURFACE_TYPE_PUSH_BUFFERS。在这种情况下，SurfaceView的绘图表面所使用的图形缓冲区是完全由摄像头服务或者视频播放服务来提供的，因此，我们就不可以随意地去访问该图形缓冲区，而是要由摄像头服务或者视频播放服务来访问，因为该图形缓冲区有可能是在专门的硬件里面分配的。

另外还有一个地方需要注意的是，上述代码既可以在应用程序的主线程中执行，也可以是在一个独立的线程中执行。如果上述代码是在应用程序的主线程中执行，那么就需要保证它们不会占用过多的时间，否则的话，就会导致应用程序的主线程不能及时地响应用户输入，从而导致ANR问题。

为了方便起见，我们假设一个SurfaceView的绘图表面的类型不是SURFACE_TYPE_PUSH_BUFFERS，接下来，我们就从SurfaceView的成员函数getHolder开始，分析这个SurfaceView的绘制过程，如下所示：

![img](http://img.my.csdn.net/uploads/201303/16/1363444666_7268.jpg)

图4 SurfaceView的绘制过程

这个过程可以分为5个步骤，接下来我们就详细分析每一个步骤。

### Step 1. SurfaceView.getHolder

```java
public class SurfaceView extends View {  
    ......  

    public SurfaceHolder getHolder() {  
return mSurfaceHolder;  
    }  

    ......  

    private SurfaceHolder mSurfaceHolder = new SurfaceHolder() {  
......  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/SurfaceView.java中。

SurfaceView类的成员函数getHolder的实现很简单，它只是将成员变量mSurfaceHolder所指向的一个SurfaceHolder对象返回给调用者。

### Step 2. SurfaceHolder.lockCanvas

```java
public class SurfaceView extends View {  
    ......  

    final ReentrantLock mSurfaceLock = new ReentrantLock();  
    final Surface mSurface = new Surface();  
    ......  

    private SurfaceHolder mSurfaceHolder = new SurfaceHolder() {  
......  

public Canvas lockCanvas() {  
    return internalLockCanvas(null);  
}  

......  

private final Canvas internalLockCanvas(Rect dirty) {  
    if (mType == SURFACE_TYPE_PUSH_BUFFERS) {  
        throw new BadSurfaceTypeException(  
                "Surface type is SURFACE_TYPE_PUSH_BUFFERS");  
    }  
    mSurfaceLock.lock();  
    ......  

    Canvas c = null;  
    if (!mDrawingStopped && mWindow != null) {  
        Rect frame = dirty != null ? dirty : mSurfaceFrame;  
        try {  
            c = mSurface.lockCanvas(frame);  
        } catch (Exception e) {  
            Log.e(LOG_TAG, "Exception locking surface", e);  
        }  
    }  
    ......  

    if (c != null) {  
        mLastLockTime = SystemClock.uptimeMillis();  
        return c;  
    }  
    ......  

    mSurfaceLock.unlock();  

    return null;    
}   

......           
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/SurfaceView.java中。

SurfaceHolder类的成员函数lockCanvas通过调用另外一个成员函数internalLockCanvas来在当前正在处理的SurfaceView的绘图表面上建立一块画布返回给调用者。

SurfaceHolder类的成员函数internalLockCanvas首先是判断当前正在处理的SurfaceView的绘图表面的类型是否是SURFACE_TYPE_PUSH_BUFFERS，如果是的话，那么就会抛出一个类型为BadSurfaceTypeException的异常，原因如前面所述。

由于接下来SurfaceHolder类的成员函数internalLockCanvas要在当前正在处理的SurfaceView的绘图表面上建立一块画布，并且返回给调用者访问，而这块画布不是线程安全的，也就是说它不能同时被多个线程访问，因此，就需要对当前正在处理的SurfaceView的绘图表面进行锁保护，这是通过它的锁定它的成员变量mSurfaceLock所指向的一个ReentrantLock对象来实现的。

注意，如果当前正在处理的SurfaceView的成员变量mWindow的值等于null，那么就说明它的绘图表面还没有创建好，这时候就无法创建一块画布返回给调用者。同时，如果当前正在处理的SurfaceView的绘图表面已经创建好，但是该SurfaceView当前是处于停止绘制的状态，即它的成员变量mDrawingStopped的值等于true，那么也是无法创建一块画布返回给调用者的。

假设当前正在处理的SurfaceView的绘制表面已经创建好，并且它不是处于停止绘制的状态，那么SurfaceHolder类的成员函数internalLockCanvas就会通过调用该SurfaceView的成员变量mSurface所指向的一个Surface对象的成员函数lockCanvas来创建一块画布，并且返回给调用者。注意，在这种情况下，当前正在处理的SurfaceView的绘制表面还是处于锁定状态的。

另一方面，如果SurfaceHolder类的成员函数internalLockCanvas不能成功地在当前正在处理的SurfaceView的绘制表面上创建一块画布，即变量c的值等于null，那么SurfaceHolder类的成员函数internalLockCanvas在返回一个null值调用者之前，还会将该SurfaceView的绘制表面就会解锁。

从前面第1部分的内容可以知道，SurfaceView类的成员变量mSurface描述的是就是SurfaceView的专有绘图表面，接下来我们就继续分析它所指向的一个Surface对象的成员函数lockCanvas的实现，以便可以了解SurfaceView的画布的创建过程。

### Step 3. Surface.lockCanvas

Surface类的成员函数lockCanvas的具体实现可以参考前面[Android应用程序窗口（Activity）的测量（Measure）、布局（Layout）和绘制（Draw）过程分析](http://blog.csdn.net/luoshengyang/article/details/8372924)一文，它大致就是通过JNI方法来在当前正在处理的绘图表面上获得一个图形缓冲区，并且将这个图形绘冲区封装在一块类型为Canvas的画布中返回给调用者使用。

调用者获得了一块类型为Canvas的画布之后，就可以调用Canvas类所提供的绘图函数来绘制任意的UI了，例如，调用Canvas类的成员函数drawLine、drawRect和drawCircle可以分别用来画直线、矩形和圆。

调用者在画布上绘制完成所需要的UI之后，就可以将这块画布的图形绘冲区的UI数据提交给SurfaceFlinger服务来处理了，这是通过调用SurfaceHolder类的成员函数unlockCanvasAndPost来实现的。

### Step 4. SurfaceHolder.unlockCanvasAndPost

```java
public class SurfaceView extends View {  
    ......  

    final ReentrantLock mSurfaceLock = new ReentrantLock();  
    final Surface mSurface = new Surface();  
    ......  

    private SurfaceHolder mSurfaceHolder = new SurfaceHolder() {  
......  

public void unlockCanvasAndPost(Canvas canvas) {  
    mSurface.unlockCanvasAndPost(canvas);  
    mSurfaceLock.unlock();  
}   

......           
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/SurfaceView.java中。

SurfaceHolder类的成员函数unlockCanvasAndPost是通过调用当前正在处理的SurfaceView的成员变量mSurface所指向的一个Surface对象的成员函数unlockCanvasAndPost来将参数canvas所描述的一块画布的图形缓冲区提交给SurfaceFlinger服务处理的。

提交完成参数canvas所描述的一块画布的图形缓冲区给SurfaceFlinger服务之后，SurfaceHolder类的成员函数unlockCanvasAndPost再调用当前正在处理的SurfaceView的成员变量mSurfaceLock所指向的一个ReentrantLock对象的成员函数unlock来解锁当前正在处理的SurfaceView的绘图表面，因为在前面的Step 2中，我们曾经将该绘图表面锁住了。

接下来，我们就继续分析Surface类的成员函数unlockCanvasAndPost的实现，以便可以了解SurfaceView的绘制过程。

### Step 5. Surface.unlockCanvasAndPost