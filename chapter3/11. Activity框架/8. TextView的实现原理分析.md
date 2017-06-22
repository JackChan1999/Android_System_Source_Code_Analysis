在前面一个系列的文章中，我们以窗口为单位，分析了WindowManagerService服务的实现。同时，在再前面一个系列的文章中，我们又分析了窗口的组成。简单来说，窗口就是由一系列的视图按照一定的布局组织起来的。实际上，每一个视图都是一个控件，这些控制可以将自己的UI绘制在窗口的绘图表面上，同时还可以与用户进行交互，即获得用户的键盘或者触摸屏输入。在本文中，我们就详细分析窗口控件的上述实现原理。

《[Android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

由于[android](http://lib.csdn.net/base/android)系统提供的控件比较多，因此我们只能挑一个比较有代表的控件进行分析。这个比较有代表性的控件便是TextView，其它的一些基础控件，例如Button、EditText和CheckBox等，都是直接或者间接地以它为父类的。每一个控件的实现都是相当复杂的，不过基本上都是一些细节问题，而且不同的控件有不同的实现细节，因此，本文并不打算详细地分析TextView的具体实现，而是从所有控件为了实现自己的功能而需要的东西出发，去分析TextView的实现框架。

那么，控件为了实现自己的功能而需要的东西是什么呢？有两个材料是必不可少的。第一个材料是画布，第二个材料是用户输入。有画布才能绘制UI，而有用户输入才能与用户进行交互。因此，接下来我们主要分析TextView的绘制流程，以及它获得用户输入的过程。用户输入主要包括键盘输入以及触摸屏输入，本文主要关注的是键盘输入。触摸屏输入与键盘输入的获取过程是类似的，读者如果有兴趣的话，可以参照本文的内容来自己研究一下。

从前面[Android应用程序窗口（Activity）实现框架简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/8170307)这个系列的文章可以知道，应用程序窗口，即Activity窗口，是由一个PhoneWindow对象，一个DecorView对象，以及一个ViewRoot对象来描述的。其中，PhoneWindow对象用来描述窗口对象，DecorView对象用来描述窗口的顶层视图，ViewRoot对象除了用来与WindowManagerService服务通信之外，还用来接收用户输入。窗口控件本身也是一个视图，即一个View对象，它们是以树形结构组织在一起形成整个窗口的UI的。为了简单起见，本文假设要分析的TextView控件是直接以窗口的顶层视图为父视图的，即以DecorView为父视图，如图1所示：

![img](http://img.my.csdn.net/uploads/201303/06/1362504419_7420.jpg)

图1 窗口结构示意图以及DecorView、TextView的类关系图

图1显示的是一个包含了TextView控件的Activity窗口的结构示意图以及DecorView、TextView的简单类关系图，从中可以看出：

用户输入首先是由ViewRoot接收，然后再分发给TextView处理；

DecorView是一个视图容器，因此，它是从ViewGroup继承下来，而ViewGroup本身又是从View继承下来的；

TextView是一个简单视图，因此，它是直接继承了View。

接下来，我们就以图1所示的Activity窗口为例，来分析TextView控件的UI绘制框架及其获得键盘输入的过程。

## TextView控件的UI绘制框架

从前面[Android应用程序窗口（Activity）的测量（Measure）、布局（Layout）和绘制（Draw）过程分析](http://blog.csdn.net/luoshengyang/article/details/8372924)一文可以知道，Activity窗口的UI绘制操作分为三步来走，分别是测量、布局和绘制。

### 测量

为了能告诉父视图自己的所占据的空间的大小，所有控件都必须要重写父类View的成员函数onMeasure。

TextView类的成员函数onMeasure的实现如下所示：

```java
public class TextView extends View implements ViewTreeObserver.OnPreDrawListener {  
    ......  
  
    @Override  
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {  
int widthMode = MeasureSpec.getMode(widthMeasureSpec);  
int heightMode = MeasureSpec.getMode(heightMeasureSpec);  
int widthSize = MeasureSpec.getSize(widthMeasureSpec);  
int heightSize = MeasureSpec.getSize(heightMeasureSpec);  
  
int width;  
int height;  
  
//计算TextView控件的宽度和高度  
......  
  
setMeasuredDimension(width, height);  
    }  
  
    ......  
}  
```
这个函数定义在文件frameworks/base/core/[Java](http://lib.csdn.net/base/java)/android/widget/TextView.java中。

参数widthMeasureSpec和heightMeasureSpec分别用来描述宽度测量规范和高度测量规范。测量规范使用一个int值来表法，这个int值包含了两个分量。

第一个是mode分量，使用最高2位来表示。测量模式有三种，分别是MeasureSpec.UNSPECIFIED（0）、MeasureSpec.EXACTLY（1）、和MeasureSpec.AT_MOST（2）。

第二个是size分量，使用低30位来表示。当mode分量等于MeasureSpec.EXACTLY时，size分量的值就是父视图要求当前控件要设置的宽度或者高度；当mode分量等于MeasureSpec.AT_MOST时，size分量的值就是父视图限定当前控件可以设置的最大宽度或者高度；当mode分量等于MeasureSpec.UNSPECIFIED时，父视图不限定当前控件所设置的宽度或者高度，这时候当前控件一般就按照实际需求来设置自己的宽度和高度。

TextView类的成员函数onMeasure根据上述规则计算好自己的宽度wdith和高度height之后，必须要调用从父类View继承下来的成员函数setMeasuredDimension来通知父视图它所要设置的宽度和高度，否则的话，该函数调用结束之后，就会抛出一个类型为IllegalStateException的异常。

### 布局

前面的测量工作实际上是确定了控件的大小，但是控件的位置还未确定。控件的位置是通过布局这个操作来完成的。 

我们知道，控件是按照树形结构组织在一起的，其中，子控件的位置由父控件来设置，也就是说，只有容器类控件才需要执行布局操作，这是通过重写父类View的成员函数onLayout来实现的。从Activity窗口的结构可以知道，它的顶层视图是一个DecorView，这是一个容器类控件。Activity窗口的布局操作就是从其顶层视图开始执行的，每碰到一个容器类的子控件，就调用它的成员函数onLayout来让它有机会对自己的子控件的位置进行设置，依次类推。

我们常见的FrameLayout、LinearLayout、RelativeLayout、TableLayout和AbsoluteLayout，都是属于容器类控件，因此，它们都需要重写父类View的成员函数onLayout。由于TextView控件不是容器类控件，因此，它可以不重写父类View的成员函数onLayout。

### 绘制

有了前面两个操作之后，控件的位置的大小就确定下来了，接下来就可以对它们的UI进行绘制了。控件为了能够绘制自己的UI，必须要重写父类View的成员函数onDraw。

TextView类的成员函数onDraw的实现如下所示：

```java
public class TextView extends View implements ViewTreeObserver.OnPreDrawListener {  
    ......  
  
    @Override  
    protected void onDraw(Canvas canvas) {  
//在画布canvas上绘制UI  
......  
    }  
  
    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/widget/TextView.java中。

参数canvas描述的是一块画布，控件的UI就是绘制在这块画布上面的。画布提供了丰富的接口来绘制UI，例如画线（drawLine）、画圆（drawCircle）和贴图（drawBitmap）等等。有了这些UI画图接口之后，就可以随心所欲地绘制控件的UI了。

从前面[Android应用程序窗口（Activity）的测量（Measure）、布局（Layout）和绘制（Draw）过程分析](http://blog.csdn.net/luoshengyang/article/details/8372924)一文可以知道，Java层的Canvas实际上是封装了C++层的SkCanvas。C++层的SkCanvas内部有一块图形缓冲区，这块图形缓冲区就是窗口的绘图表面（Surface）里面的那块图形缓冲区。

从前面[Android应用程序与SurfaceFlinger服务的关系概述和学习计划](http://blog.csdn.net/luoshengyang/article/details/7846923)一文可以知道，窗口的绘图表面里面的那块图形缓冲区实际上是一块匿名共享内存，它是SurfaceFlinger服务负责创建的。SurfaceFlinger服务创建完成这块匿名共享内存之后，就会将其返回给窗口所运行在的进程。窗口所运行在的进程获得了这块匿名共享内存之后，就会映射到自己的进程空间来，因此，窗口的控件就可以在本进程内访问这块匿名共享内存了，实际上就是往这块匿名共享内存填入UI数据。注意，这个过程执行完成之后，控件的UI还没有反映到屏幕上来，因为这时候将控件的UI数据填入到图形缓冲区而已。

从前面[Android窗口管理服务WindowManagerService的简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/8462738)一文可以知道，窗口的UI的显示是WindowManagerService服务来控制的。因此，当窗口的所有控件都绘制完成自己的UI之后，窗口就会向WindowManagerService服务发送一个Binder进程间程通信请求。WindowManagerService服务接收到这个Binder进程间程通信请求之后，就会请求SurfaceFlinger服务刷新相应的窗口的UI。SurfaceFlinger服务刷新窗口UI的过程可以参考前面[Android系统Surface机制的SurfaceFlinger服务渲染应用程序UI的过程分析](http://blog.csdn.net/luoshengyang/article/details/8079456)一文。

从上面的描述就可以看出，控件的UI虽然是在一块简单的画布进行绘制，但是其中蕴含了丰富的知识点，并且需要应用程序进程、WindowManagerService服务和SurfaceFlinger服务三方紧密而有序的配合。

如果我们仔细阅读[Android应用程序窗口（Activity）的测量（Measure）、布局（Layout）和绘制（Draw）过程分析](http://blog.csdn.net/luoshengyang/article/details/8372924)一文，还可以得出以下两个结论：

(1). 一个窗口的所有控件的UI都是绘制在窗口的绘图表面上的，也就是说，一个窗口的所有控件的UI数据都是填写在同一块图形缓冲区中；

(2). 一个窗口的所有控件的UI的绘制操作是在主线程中执行的，事实上，所有与UI相关的操作都是必须是要在主线程中执行，否则的话，就会抛出一个类型为CalledFromWrongThreadException的异常来。

为什么要规定所有与UI相关的操作都必须在主线程中执行呢？我们知道，这些与UI相关的操作都涉及到大量的控件内部状态以及需要访问窗口的绘图表面，也就是说，要大量地访问控件类的成员变量以及窗口绘图表面里面的图形缓冲区，因此，如果不将这些与UI相关的操作限定在同一个线程中执行的话，那么就会涉及到线程同步问题。线程同步的开销是很大的，因此，就要保证那些与UI相关的操作都在同一个线程中执行。这个负责执行UI相关操作的线程便是应用程序进程的主线程，因此我们也将应用程序进程的主线程称为UI线程。

我们知道，应用程序进程的主线程除了负责执行与UI相关的操作之外，还负责响应用户的输入，因此，我们就要尽量地避免执行很耗时的UI操作，否则的话，系统就会由于应用程序进程的主线程无法及时响应用户输入而弹出ANR对话框。

那么，有没有办法让某一个控件的UI享有独立的图形缓冲区呢？也就是这个控件不将自己的UI数据填入到它的宿主窗口的绘图表面的图形缓冲区里面去。如果可以的话，那么我们就可以在另外一个独立的线程中绘制该控件的UI。这样做的好处是显而易见——可以在这个独立的线程执行相对比较耗时的UI绘制操作而不会导致主线程无法及时响应用户输入。答案是肯定的，在接下来的一篇文章中，我们就分析一个可以具有独立图形缓冲区的控件——SurfaceView。

## TextView控件获取键盘输入的过程分析

从前面[Android应用程序键盘（Keyboard）消息处理机制分析](http://blog.csdn.net/luoshengyang/article/details/6882903)一文可以知道，每一个窗口的创建的时候，都会与系统的输入管理器建立一个用户输入接收通道。输入管理器在启动两个线程，其中一个用来监控用户输入，即监控用户是否按下或者放开了键盘按键，或者是否触摸了屏幕，另外一个用来将监控到的用户输入事件分发给当前激活的窗口来处理，而这个分发过程就是通过前面建立的通道来进行的。

当前激活的窗口接收到输入管理器分发过来的用户输入事件之后，就会该事件封装成一个消息发送到当前激活的窗口所运行在的应用程序进程的主线程的消息队列中去。等到这个消息被处理的时候，就会调用与当前激活的窗口所关联的一个ViewRoot对象的成员函数deliverKeyEvent或者deliverPointerEvent来将前面接收到的用户输入分发给合适的控件。其中，ViewRoot类的成员函数deliverKeyEvent负责分发键盘输入事件，而ViewRoot类的成员函数deliverPointerEvent负责分发触摸屏输入事件。

接下来，我们就从ViewRoot类的成员函数deliverKeyEvent开始，分析一个TextView控件获得键盘输入的过程（获得触摸屏输入的过程是类似的），如图2所示：

![img](http://img.my.csdn.net/uploads/201303/09/1362766707_2559.jpg)

图2 TextView控件获得键盘输入的过程

这个过程可以分为14个步骤，接下来我们就详细分析每一个步骤。

### Step 1. ViewRoot.deliverKeyEvent

```java
public final class ViewRoot extends Handler implements ViewParent,  
View.AttachInfo.Callbacks {  
    ......  
  
    private void deliverKeyEvent(KeyEvent event, boolean sendDone) {  
// If mView is null, we just consume the key event because it doesn't  
// make sense to do anything else with it.  
boolean handled = mView != null  
        ? mView.dispatchKeyEventPreIme(event) : true;  
if (handled) {  
    if (sendDone) {  
        finishInputEvent();  
    }  
    return;  
}  
// If it is possible for this window to interact with the input  
// method window, then we want to first dispatch our key events  
// to the input method.  
if (mLastWasImTarget) {  
    InputMethodManager imm = InputMethodManager.peekInstance();  
    if (imm != null && mView != null) {  
        int seq = enqueuePendingEvent(event, sendDone);  
        ......  
  
        imm.dispatchKeyEvent(mView.getContext(), seq, event,  
                mInputMethodCallback);  
        return;  
    }  
}  
deliverKeyEventToViewHierarchy(event, sendDone);  
    }  
  
    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/ViewRoot.java中。

参数event描述的是窗口接收到的键盘事件，另外一个参数sendDone表示该键盘事件处理完成后，是否需要向系统的输入管理器发送一个通知。

ViewRoot类的成员变量mView描述的是窗口的顶层视图，即它指向的是一个DecorView对象，ViewRoot类的成员函数deliverKeyEvent首先是调用它的成员函数dispatchKeyEventPreIme来让它优先于输入法处理参数event所描述的键盘事件。如果这个DecorView对象的成员函数dispatchKeyEventPreIme的返回值handled等于true，那么就说明参数event所描述的键盘事件已经处理完毕，即ViewRoot类的成员函数deliverKeyEvent不用往下执行了。在这种情况下，如果参数sendDone的值等于true，那么ViewRoot类的成员函数deliverKeyEvent在返回之前，还会调用成员函数finishInputEvent来通知系统的输入管理器，当前激活的窗口已经处理完成刚刚发生的键盘事件了。在接下来的Step 2到Step 4中，我们再详细分析键盘事件优先于输入法分发给窗口处理的过程。

假设窗口不在输入法前面拦截参数event所描述的键盘事件，接下来ViewRoot类的成员函数deliverKeyEvent就会将该键盘事件分发给输入法处理，这个分发过程如下所示：

调用InputMethodManager类的静态成员函数peekInstance获得一个类型为InputMethodManager输入法管理器imm；

调用ViewRoot类的成员函数enqueuePendingEvent将参数event所描述的键盘事件缓存起来，等到输入法处理完成该键盘事件之后，再继续对它进行处理；

调用第1步获得的输入法管理器imm的成员函数dispatchKeyEvent来将参数event所描述的键盘事件分发给输入法处理。

这里有两个地方是需要注意的。第一个地方是只有当前窗口正在显示输入法的情况下，ViewRoot类的成员函数deliverKeyEvent才会将参数event所描述的键盘事件分发给输入法处理，这是通过检查ViewRoot类的成员变量mLastWasImTarget的值是否等于true来确定的。第二个地方是在将参数event所描述的键盘事件分发给输入法处理时，ViewRoot类的成员函数deliverKeyEvent会同时传递一个类型为InputMethodCallback的回调接口给输入法，以便输入法处理完成参数event所描述的键盘事件之后，可以调用这个回调接口的成员函数finishedEvent来向窗口发送一个键盘事件处理完成通知。这个类型为InputMethodCallback的回调接口就保存在ViewRoot类的成员变量mInputMethodCallback中，当它的成员函数finishedEvent被调用的时候，它就会调用ViewRoot类的成员函数deliverKeyEventToViewHierarchy来继续将参数event所描述的键盘事件分发给窗口处理。

 如果窗口当前不需要与输入法交互，即ViewRoot类的成员变量mLastWasImTarget的值等于false，那么ViewRoot类的成员函数deliverKeyEvent就会直接调用成员函数deliverKeyEventToViewHierarchy来将参数event所描述的键盘事件分发给窗口处理。

 接下来，我们就先析窗口在输入法之前处理键盘输入的过程，接着再分析窗口在输入法之后处理键盘输入的过程。

 从前面的分析可以知道，ViewRoot类的成员函数deliverKeyEvent是通过调用DecorView类的成员函数dispatchKeyEventPreIme来将获得的键盘输入优先于输入法分发给窗口处理的。DecorView类的成员函数dispatchKeyEventPreIme是从父类ViewGroup继承下来的，因此，接下来我们就继续分析ViewGroup类的成员函数dispatchKeyEventPreIme的实现。

###  Step 2. ViewGroup.dispatchKeyEventPreIme

```java
public abstract class ViewGroup extends View implements ViewParent, ViewManager {  
    ......  
  
    // The view contained within this ViewGroup that has or contains focus.  
    private View mFocused;  
    ......  
  
    @Override  
    public boolean dispatchKeyEventPreIme(KeyEvent event) {  
if ((mPrivateFlags & (FOCUSED | HAS_BOUNDS)) == (FOCUSED | HAS_BOUNDS)) {  
    return super.dispatchKeyEventPreIme(event);  
} else if (mFocused != null && (mFocused.mPrivateFlags & HAS_BOUNDS) == HAS_BOUNDS) {  
    return mFocused.dispatchKeyEventPreIme(event);  
}  
return false;  
    }  
  
    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/ViewGroup.java中。

ViewGroup类的成员函数dispatchKeyEventPreIme首先是检查当前正在处理的视图容器是否能够获得焦点。如果能够获得焦点的话，那么ViewGroup类的成员变量mPrivateFlags的FOCUSED位就会等于1。在当前正在处理的视图容器能够获得焦点的情况下，还要检查正在处理的视图容器是否已经计算过大小了，即检查ViewGroup类的成员变量mPrivateFlags的HAS_BOUNDS位是否等于1。只有在已经计算过大小并且能够获得焦点的情况下，那么正在处理的视图容器才有资格处理参数event所描述的键盘事件。注意，正在处理的视图容器是通过调用其父类View的成员函数dispatchKeyEventPreIme来处理参数event所描述的键盘事件的。

如果当前正在处理的视图容器没有资格处理参数event所描述的键盘事件，但是它有一个能够获得焦点的子视图，并且这个子视图的大小也是已经计算好了的，那么ViewGroup类的成员函数dispatchKeyEventPreIme就会将参数event所描述的键盘事件分发给该子视图处理。当前正在处理的视图容器能够获得焦点的子视图是通过ViewGroup类的成员变量mFocused来描述的，通过调用这个成员变量所描述的一个View对象的成员函数dispatchKeyEventPreIme即可将参数event所描述的键盘事件分发给够获得焦点的子视图处理。

一个视图容器是如何知道它的焦点子视图的呢？我们知道，当我们在屏幕上触摸一个窗口时，就会发生一个Pointer事件。这个Pointer事件关联有一个触摸点，通过检查这个触摸点当前是包含在窗口顶层视图的哪一个子视图里面，就可以知道哪一个子视图是焦点子视图了。

从上面的分析可以知道，无论是当前正在处理的视图容器获得焦点，还是它的子视图获得焦点，最终都是通过调用View类的成员函数dispatchKeyEventPreIme来在输入法之前处理参数event所描述的键盘事件，因此，接下来我们就继续分析View类的成员函数dispatchKeyEventPreIme的实现。

### Step 3. View.dispatchKeyEventPreIme

```java
public class View implements Drawable.Callback, KeyEvent.Callback, AccessibilityEventSource {  
    ......  
  
    public boolean dispatchKeyEventPreIme(KeyEvent event) {  
return onKeyPreIme(event.getKeyCode(), event);  
    }  
  
    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/View.java中。

View类的成员函数dispatchKeyEventPreIme的实现很简单，它只是通过调用另外一个成员函数onKeyPreIme来在输入法之前处理参数event所描述的键盘事件。

### Step 4. View.onKeyPreIme

```java
public class View implements Drawable.Callback, KeyEvent.Callback, AccessibilityEventSource {  
    ......  
  
    public boolean onKeyPreIme(int keyCode, KeyEvent event) {  
return false;  
    }  
  
    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/View.java中。

View类的成员函数onKeyPreIme默认是不会在输入法之前处理参数event所描述的键盘事件的，因此，我们在实现自己的控件的时候，如果需要在输入法之前处理键盘输入，那么就必须重写父类View的成员函数onKeyPreIme。在重写父类View的成员函数onKeyPreIme来处理一个键盘事件的时候，如果不希望这个键盘事件分发给输入法处理，那么就返回一个true值，否则的话，就返回一个false值。

我们假设当前获得焦点的是图1所示的TextView控件，但是由于TextView类没有重写其父类View的成员函数onKeyPreIme，因此，参数event所描述的键盘事件接下来就会继续分发给输入法或者当前激活的窗口处理。

这一步执行完成之后，回到前面的Step 1中，即ViewRoot类的成员函数deliverKeyEvent中，无论接下来是否需要先将一个键盘事件分发给输入法处理，最终都会调用到ViewRoot类的成员函数deliverKeyEventToViewHierarchy来继续将该键盘事件分发给当前激活的窗口处理。

### Step 5. ViewRoot.deliverKeyEventToViewHierarchy

```java
public final class ViewRoot extends Handler implements ViewParent,  
View.AttachInfo.Callbacks {  
    ......  
  
    private void deliverKeyEventToViewHierarchy(KeyEvent event, boolean sendDone) {  
try {  
    if (mView != null && mAdded) {  
        final int action = event.getAction();  
        boolean isDown = (action == KeyEvent.ACTION_DOWN);  
        ......  
  
        boolean keyHandled = mView.dispatchKeyEvent(event);  
  
        if (!keyHandled && isDown) {  
            int direction = 0;  
            switch (event.getKeyCode()) {  
            case KeyEvent.KEYCODE_DPAD_LEFT:  
                direction = View.FOCUS_LEFT;  
                break;  
            case KeyEvent.KEYCODE_DPAD_RIGHT:  
                direction = View.FOCUS_RIGHT;  
                break;  
            case KeyEvent.KEYCODE_DPAD_UP:  
                direction = View.FOCUS_UP;  
                break;  
            case KeyEvent.KEYCODE_DPAD_DOWN:  
                direction = View.FOCUS_DOWN;  
                break;  
            }  
  
            if (direction != 0) {  
  
                View focused = mView != null ? mView.findFocus() : null;  
                if (focused != null) {  
                    View v = focused.focusSearch(direction);  
                    ......  
  
                    if (v != null && v != focused) {  
                        ......  
  
                        focusPassed = v.requestFocus(direction, mTempRect);  
                    }  
  
                    ......  
                }  
            }  
        }  
    }  
  
} finally {  
    if (sendDone) {  
        finishInputEvent();  
    }  
    ......  
}  
    }  
  
    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/ViewRoot.java中。

ViewRoot类的成员函数deliverKeyEventToViewHierarchy首先将参数event所描述的键盘事件交给当前激活的窗口的顶层视图来处理，这是通过调用ViewRoot类的成员变量mView所描述的一个DecorView对象的成员函数dispatchKeyEvent来实现的。

如果当前激活的窗口的顶层视图在处理完成参数event所描述的键盘事件之后，希望该键盘事件还能继续被ViewRoot类的成员函数deliverKeyEventToViewHierarchy处理，那么前面调用DecorView类的成员函数dispatchKeyEvent得到的返回值keyHandled的值就会等于false。在这种情况下，如果参数event描述的是一个按下的键盘事件，即变量isDown的值等于true，那么ViewRoot类的成员函数deliverKeyEventToViewHierarchy就会继续检查参数event描述的是否是一个DPAD事件。如果是的话，那么就可能需要改变窗口当前的焦点子视图。

如果参数event描述的是一个DPAD事件，那么最终得到的变量direction的值就不会等于0，并且它描述的是当前按下的是哪一个方向的DPAD键。假设这时候窗口已经有一个焦点子视图，即调用ViewRoot类的成员变量mView所描述的一个DecorView对象的成员函数findFocus的返回值focused不等于null，那么接下来就要根据变量direction的值来决定下一个焦点子视图是谁。例如，假设变量direction的值等于View.FOCUS_LEFT，那么就表示在当前的焦点子视图focused的左边查找一个最靠近的子视图作为下一个焦点子视图，这是通过调用当前焦点子视图focused的成员函数focusSearch来实现的。

一旦找到了下一个焦点子视图v，并且该子视图不是当前的焦点子视图focused，那么ViewRoot类的成员函数deliverKeyEventToViewHierarchy就需要将子视图v设置为焦点子视图，这是通过调用变量v所描述的一个View对象的成员函数requestFocus来实现的。

通过前面的操作，参数event所描述的键盘事件就处理完成了。如果这时候参数sendDone的值等于true，那么就表示需要通知系统的输入管理器，参数event所描述的键盘事件已经处理完成了，这是通过调用ViewRoot类的成员函数finishInputEvent来实现的。

接下来，我们就继续分析DecorView类的成员函数dispatchKeyEvent的实现，以便可以了解窗口的顶层视图分发键盘事件的过程。

### Step 6. DecorView.dispatchKeyEvent

```java
public class PhoneWindow extends Window implements MenuBuilder.Callback {  
    ......  
  
    private final class DecorView extends FrameLayout implements RootViewSurfaceTaker {  
......  
  
@Override  
public boolean dispatchKeyEvent(KeyEvent event) {  
    final int keyCode = event.getKeyCode();  
    final boolean isDown = event.getAction() == KeyEvent.ACTION_DOWN;  
    ......  
  
    final Callback cb = getCallback();  
    final boolean handled = cb != null && mFeatureId < 0 ? cb.dispatchKeyEvent(event)  
            : super.dispatchKeyEvent(event);  
    if (handled) {  
        return true;  
    }  
    return isDown ? PhoneWindow.this.onKeyDown(mFeatureId, event.getKeyCode(), event)  
            : PhoneWindow.this.onKeyUp(mFeatureId, event.getKeyCode(), event);  
}  
  
......  
  
    }  
  
    ......  
}  
```
这个函数定义在文件frameworks/base/policy/src/com/android/internal/policy/impl/PhoneWindow.java中。

PhoneWindow类的成员函数getCallback是从父类Window继承下来的，它返回的是一个Window.Callback接口。每一个Activity组件都会实现一个Window.Callback接口，并且将这个Window.Callback接口设置到与它所关联的一个PhoneWindow对象的内部去，这样当该PhoneWindow对象接收到键盘事件的时候，就可以该键盘事件分发给与它所关联的Activity组件处理。

DecorView类的成员变量mFeatureId用来描述当前正在处理的DecorView对象的特征，当它的值小于0的时候，就表示当前正在处理的一个DecorView对象是用来描述一个Activity组件窗口的顶层视图的。

因此，当当前正在处理的DecorView对象描述的是一个Activity组件窗口的顶层视图，并且这个Activity组件实现有一个Window.Callback接口时，DecorView类的成员函数dispatchKeyEvent就会调用该Window.Callback接口的成员函数dispatchKeyEvent来通知对应的Activity组件，它接收到一个键盘事件了。否则的话，参数event所描述的键盘事件就会被分发给当前正在处理的DecorView对象的父对象来处理，这是通过调用DecorView类的父类View的成员函数dispatchKeyEvent来实现的。

我们假设当前正在处理的DecorView对象描述的是一个Activity组件窗口的顶层视图，并且这个Activity组件实现有一个Window.Callback接口，那么参数event所描述的键盘事件接下来就会分给该Activity组件处理。如果该Activity组件在处理完成这个键盘事件之后，希望该键盘事件还能继续分发下去给其它对象处理，那么它所实现的Window.Callback接口的成员函数dispatchKeyEvent的返回值handled就会等于false，这时候DecorView类的成员函数dispatchKeyEvent就会将该键盘事件分发给与当前正在处理的DecorView对象所关联的一个PhoneWindow对象的成员函数onKeyDown或者onKeyUp来处理，取决于变量isDown的值是true还是false，即当前发生的键盘事件是与按键按下有关，还是与按键松开有关。

PhoneWindow类的成员函数onKeyDown和onKeyUp主要是有来监控一些特殊按键事件，例如电话键和音量键，以便可以执行一些对应的逻辑。例如，当按下电话键时，就打开拨号程序；又如，当按下音量键时，就调节音量的大小。

接下来，我们就继续分析Activity类所实现的Window.Callback接口的成员函数dispatchKeyEvent的实现，以便可以了解键盘事件在Activity组件窗口的分发过程。

### Step 7. Activity.dispatchKeyEvent

```java
public class Activity extends ContextThemeWrapper  
implements LayoutInflater.Factory,  
Window.Callback, KeyEvent.Callback,  
OnCreateContextMenuListener, ComponentCallbacks {  
    ......  
  
    public boolean dispatchKeyEvent(KeyEvent event) {  
......  
  
Window win = getWindow();  
if (win.superDispatchKeyEvent(event)) {  
    return true;  
}  
View decor = mDecor;  
if (decor == null) decor = win.getDecorView();  
return event.dispatch(this, decor != null  
        ? decor.getKeyDispatcherState() : null, this);  
    }  
  
    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/app/Activity.java中。

Activity类的成员函数getWindow返回的是与当前正处理的Activity组件所关联的一个PhoneWindow对象，Activity类的成员函数dispatchKeyEvent获得了这个PhoneWindow对象之后，就会调用它的成员函数superDispatchKeyEvent，以便可以将参数event所描述的键盘事件分发给它处理。

这个PhoneWindow对象在处理完成参数event所描述的键盘事件之后，如果希望该键盘事件能继续往下分发，那么Activity类的成员函数dispatchKeyEvent就会将该键盘事件分发给当前正在处理的Activity组件处理，这是通过调用参数event所描述的一个KeyEvent对象的成员函数dispatch来实现的。

注意，在调用event所描述的一个KeyEvent对象的成员函数dispatch的时候，第一个参数指定为当前正在处理的Activity组件所实现的一个KeyEvent.Callback接口。参数event所指向的一个KeyEvent对象的成员函数dispatch的执行的过程中，就会相应地调用这个KeyEvent.Callback接口的成员函数onKeyDown、onKeyUp或者onKeyMultiple来处理它所描述的键盘事件，实际上就是调用Activity类的成员函数onKeyDown、onKeyUp或者onKeyMultiple来处理参数event所描述的键盘事件。因此，我们在自定义一个Activity组件时，如果需要处理分发给该Activity组件的键盘事件，那么就需要重写父类Activity的成员函数onKeyDown、onKeyUp或者onKeyMultiple。

接下来，我们就继续分析PhoneWindow类的成员函数superDispatchKeyEvent的实现，以便可以了解键盘事件在Activity组件窗口的分发过程。

### Step 8. PhoneWindow.superDispatchKeyEvent

```java
public class PhoneWindow extends Window implements MenuBuilder.Callback {  
    ......  
  
    // This is the top-level view of the window, containing the window decor.  
    private DecorView mDecor;  
    ......  
  
    @Override  
    public boolean superDispatchKeyEvent(KeyEvent event) {  
return mDecor.superDispatchKeyEvent(event);  
    }  
  
    ......  
}  
```
这个函数定义在文件frameworks/base/policy/src/com/android/internal/policy/impl/PhoneWindow.java中。

PhoneWindow类的成员变量mDecor描述的是当前正在处理的Activity组件窗口的顶层视图，PhoneWindow类的成员函数superDispatchKeyEvent通过调用它所指向的一个DecorView对象的成员函数superDispatchKeyEvent来处理参数event所描述的键盘事件。

### Step 9. DecorView.superDispatchKeyEvent

```java
public class PhoneWindow extends Window implements MenuBuilder.Callback {  
    ......  
  
    private final class DecorView extends FrameLayout implements RootViewSurfaceTaker {  
......  
  
public boolean superDispatchKeyEvent(KeyEvent event) {  
    return super.dispatchKeyEvent(event);  
}  

......  
    }  
  
    ......  
}  
```
这个函数定义在文件frameworks/base/policy/src/com/android/internal/policy/impl/PhoneWindow.java中。

DecorView类的成员函数superDispatchKeyEvent的实现很简单，它只是调用父类ViewGroup的成员函数dispatchKeyEvent来处理参数event所描述的键盘事件。

### Step 10. ViewGroup.dispatchKeyEvent

```java
public abstract class ViewGroup extends View implements ViewParent, ViewManager {  
    ......  
  
    @Override  
    public boolean dispatchKeyEvent(KeyEvent event) {  
if ((mPrivateFlags & (FOCUSED | HAS_BOUNDS)) == (FOCUSED | HAS_BOUNDS)) {  
    return super.dispatchKeyEvent(event);  
} else if (mFocused != null && (mFocused.mPrivateFlags & HAS_BOUNDS) == HAS_BOUNDS) {  
    return mFocused.dispatchKeyEvent(event);  
}  
return false;  
    }  
  
    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/ViewGroup.java中。

ViewGroup类的成员函数dispatchKeyEvent的实现与在前面的Step 3中所介绍的ViewGroup类的成员函数dispatchKeyEventPreIme的实现是类似的，即如果当前正在处理的视图容器能够获得焦点并且该视图容器的大小已经计算好了，那么就会将参数event所描述的键盘事件分发给它的父类View的成员函数dispatchKeyEvent来处理，否则的话，如果当前正在处理的视图容器有一个焦点子视图，并且这个焦点子视图的大小已经计算好了，那么就将参数event所描述的键盘事件分发给该焦点子视图的父类View的成员函数dispatchKeyEvent来处理。

从前面的调用过程可以知道，当前正在处理的视图容器即为Activity组件窗口的顶层视图。我们假设在该顶层视图中，获得焦点的是一个TextView控件，并且这个TextView控件的大小已经计算好了，那么接下来就会调用这个TextView控件的父类View的成员函数dispatchKeyEvent来处理参数event所描述的键盘事件。

### Step 11. View.dispatchKeyEvent

```java
public class View implements Drawable.Callback, KeyEvent.Callback, AccessibilityEventSource {  
    ......  
  
    private OnKeyListener mOnKeyListener;  
    ......  
  
    public boolean dispatchKeyEvent(KeyEvent event) {  
// If any attached key listener a first crack at the event.  
//noinspection SimplifiableIfStatement  
  
......  
  
if (mOnKeyListener != null && (mViewFlags & ENABLED_MASK) == ENABLED  
        && mOnKeyListener.onKey(this, event.getKeyCode(), event)) {  
    return true;  
}  
  
return event.dispatch(this, mAttachInfo != null  
        ? mAttachInfo.mKeyDispatchState : null, this);  
    }  
  
    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/View.java中。

当View类的成员变量mOnKeyListener的值不等于null时，它所指向的一个OnKeyListener对象描述的是注册到当前正在处理的视图的一个键盘事件监听器。在这种情况下，如果当前正在处理的视图是处于启用状态的，即它的成员变量mViewFlags的ENABLED位等于1，那么参数event所描述的键盘事件就先分给该键盘事件监听器处理，这是通过调用View类的成员变量mOnKeyListener所指向的一个OnKeyListener对象的成员函数onKey来实现的。

注册到当前正在处理的视图的键盘事件监听器在处理完成参数event所描述的键盘事件之后，如果希望该键盘事件还能继续往下处理，那么View类的成员函数dispatchKeyEvent就会继续调用参数event所指向的一个KeyEvent对象的成员函数dispatch来处理该键盘事件。

接下来，我们就继续分析KeyEvent类的成员函数dispatch的实现，以便可以了解键盘事件在Activity组件窗口的分发过程。

### Step 12. KeyEvent.dispatch

```java
public class KeyEvent extends InputEvent implements Parcelable {    
    ......    
    
    public final boolean dispatch(Callback receiver, DispatcherState state,    
    Object target) {    
switch (mAction) {    
case ACTION_DOWN: {    
    ......    
    boolean res = receiver.onKeyDown(mKeyCode, this);    
    ......    
    return res;    
}    
case ACTION_UP:    
    ......    
    return receiver.onKeyUp(mKeyCode, this);    
case ACTION_MULTIPLE:    
    final int count = mRepeatCount;    
    final int code = mKeyCode;    
    if (receiver.onKeyMultiple(code, count, this)) {    
        return true;    
    }    
    ......    
    return false;    
}    
return false;    
    }    
    
    ......    
}    
```
这个函数定义在文件frameworks/base/core/java/android/view/KeyEvent.java中。

从前面的调用过程可以知道，参数receiver指向的是一个View对象所实现的一个KeyEvent.Callback接口，这个KeyEvent.Callback接口是用来接收当前正在处理的键盘事件。

KeyEvent类的成员变量mAction描述的的是当前正在处理的键盘事件的类型，当它的值等于ACTION_DOWN、ACTION_UP和ACTION_MULTIPLE的时候，KeyEvent类的成员函数dispatch就会分别调用参数receiver所指向的一个View对象的成员函数onKeyDown、onKeyUp和onKeyMultiple来接收当前正在处理的键盘事件。

假设当前正在处理的键盘事件是与按键按下相关的，即KeyEvent类的成员变量mAction的值等于ACTION_DOWN，那么接下来就会调用参数receiver所指向的一个View对象的成员函数onKeyDown来接收当前正在处理的键盘事件。

由于前面我们已经假设了当前获得焦点的是一个TextView控件，因此，参数receiver指向的实际上是一个TextView对象。TextView类重写了父类View的成员函数onKeyDown，因此，接下来KeyEvent类的成员函数dispatch就会调用TextView类的成员函数onKeyDown来接收当前正在处理的键盘事件。

### Step 13. TextView.onKeyDown

```java
public class TextView extends View implements ViewTreeObserver.OnPreDrawListener {  
    ......  
  
    @Override  
    public boolean onKeyDown(int keyCode, KeyEvent event) {  
int which = doKeyDown(keyCode, event, null);  
if (which == 0) {  
    // Go through default dispatching.  
    return super.onKeyDown(keyCode, event);  
}  
  
return true;  
    }  
  
    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/widget/TextView.java中。

TextView类的成员函数onKeyDown调用另外一个成员函数doKeyDown来处理参数event所描述的键盘事件，以便可以相应地改变当前获得焦点的TextView控件的UI。当TextView类的成员函数doKeyDown的返回值which等于0的时候，就表示当前获得焦点的TextView控件希望参数event所描述的键盘事件可以继续分发给它的父类View处理，这是通过调用父类View的成员函数onKeyDown来实现的。

至此，我们就分析完成TextView控件获得键盘事件的过程了，整个TextView控件的实现框架也分析完成了。

在Android系统中，其它的Android控件与TextView控件的实现框架都是类似的，区别就在于实现细节和所表现的UI不一样，而且它们一般都有一个共同的特点，那就是都在宿主窗口的绘图表面上进行UI绘制。在接下来的一篇文章中，我们就分析另外一个种以SurfaceView为代表的特殊控件，它们的UI是绘制在一个专用的绘图表面上面的，即它们不与宿主窗口共享同一个绘图表面。敬请关注！