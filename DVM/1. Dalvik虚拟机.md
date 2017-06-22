我们知道，[Android](http://lib.csdn.net/base/android)应用程序是运行在Dalvik虚拟机里面的，并且每一个应用程序对应有一个单独的Dalvik虚拟机实例。除了指令集和类文件格式不同，Dalvik虚拟机与[Java](http://lib.csdn.net/base/java)虚拟机共享有差不多的特性，例如，它们都是解释执行，并且支持即时编译（JIT）、垃圾收集（GC）、Java本地方法调用（JNI）和Java远程调试协议（JDWP）等。本文对Dalvik虚拟机进行简要介绍，以及制定学习计划。

**老罗的新浪微博：http://weibo.com/shengyangluo，欢迎关注！**

**《[android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！**

Dalvik虚拟机是由[Dan Bornstein](http://www.milk.com/home/danfuzz/)开发的，名字来源于他的祖先曾经居住过的位于冰岛的同名小渔村。Dalvik虚拟机起源于[Apache Harmony](http://zh.wikipedia.org/wiki/Apache_Harmony)项目，后者是由Apache软件基金会主导的，目标是实现一个独立的、兼容JDK 5的虚拟机，并根据Apache License v2发布。由此可见，Dalvik虚拟机从诞生的那一天开始，就和Java有说不清理不断的关系。

Dalvik虚拟机与Java虚拟机的最显著区别是它们分别具有不同的类文件格式以及指令集。Dalvik虚拟机使用的是dex（Dalvik Executable）格式的类文件，而Java虚拟机使用的是class格式的类文件。一个dex文件可以包含若干个类，而一个class文件只包括一个类。由于一个dex文件可以包含若干个类，因此它就可以将各个类中重复的字符串和其它常数只保存一次，从而节省了空间，这样就适合在内存和处理器速度有限的手机系统中使用。一般来说，包含有相同类的未压缩dex文件稍小于一个已经压缩的jar文件。

Dalvik虚拟机使用的指令是基于寄存器的，而Java虚拟机使用的指令集是基于堆栈的。基于堆栈的指令很紧凑，例如，Java虚拟机使用的指令只占一个字节，因而称为字节码。基于寄存器的指令由于需要指定源地址和目标地址，因此需要占用更多的指令空间，例如，Dalvik虚拟机的某些指令需要占用两个字节。基于堆栈和基于寄存器的指令集各有优劣，一般而言，执行同样的功能，前者需要更多的指令（主要是load和store指令），而后者需要更多的指令空间。需要更多指令意味着要多占用CPU时间，而需要更多指令空间意味着数据缓冲（d-cache）更易失效。

此外，还有一种观点认为，基于堆栈的指令更具可移植性，因为它不对目标机器的寄存器进行任何假设。然而，基于寄存器的指令由于对目标机器的寄存器进行了假设，因此，它更有利于进行[AOT](http://en.wikipedia.org/wiki/AOT_compiler)（ahead-of-time）优化。 所谓AOT，就是在解释语言程序运行之前，就先将它编译成本地机器语言程序。AOT本质上是一种静态编译，它是相对于JIT而言的，也就是说，前者是在程序运行前进行编译，而后者是在程序运行时进行编译。运行时编译意味着可以利用运行时信息来得到比较静态编译更优化的代码，同时也意味不能进行某些高级优化，因为优化过程太耗时了。另一方面，运行前编译由于不占用程序运行时间，因此，它就可以不计时间成本来优化代码。无论AOT，还是JIT，最终的目标都是将解释语言编译为本地机器语言，而本地机器语言都是基于寄存器来执行的，因此，在某种程度来讲，基于寄存器的指令更有利于进行AOT编译以及优化。

事实上，基于寄存器和基于堆栈的指令集之争，就如精简指令集（RISC）和复杂指令集（CISC）之争，谁优谁劣，至今是没有定论的。例如，上面提到完成相同的功能，基于堆栈的Java虚拟机需要更多的指令，因此就会比基于寄存器的Dalvik虚拟机慢，然而，在2010年，[Oracle](http://lib.csdn.net/base/oracle)在一个ARM设备上使用一个non-graphical [Java ](http://lib.csdn.net/base/java)benchmarks来对比[java ](http://lib.csdn.net/base/java)SE Embedded和Android 2.2的性能，发现后者比前者慢了2~3倍。上述性能比较结论以及数据可以参考以下两篇文章：

- [Virtual Machine Showdown: Stack Versus Registers](http://static.usenix.org/events/vee05/full_papers/p153-yunhe.pdf)
- [Java SE Embedded Performance Versus Android 2.2](https://blogs.oracle.com/javaseembedded/entry/how_does_android_22s_performance_stack_up_against_java_se_embedded)

基于寄存器的Dalvik虚拟机和基于堆栈的Java虚拟机的更多比较和分析，还可以参考以下文章：

- [http://en.wikipedia.org/wiki/Dalvik_(software)](http://en.wikipedia.org/wiki/Dalvik_(software))
- [http://www.infoq.com/news/2007/11/dalvik](http://www.infoq.com/news/2007/11/dalvik)
- [http://www.zhihu.com/question/20207106](http://www.zhihu.com/question/20207106)

不管结论如何，Dalvik虚拟机都在尽最大的努力来优化自身，这些措施包括：

将多个类文件收集到同一个dex文件中，以便节省空间；

使用只读的内存映射方式加载dex文件，以便可以多进程共享dex文件，节省程序加载时间；

提前调整好字节序（byte order）和字对齐（word alignment）方式，使得它们更适合于本地机器，以便提高指令执行速度；

尽量提前进行字节码验证（bytecode verification），提高程序的加载速度；

需要重写字节码的优化要提前进行。

这些优化措施的更具体描述可以参考[Dalvik Optimization and Verification With dexopt](http://www.netmite.com/android/mydroid/dalvik/docs/dexopt.html)一文。

分析完Dalvik虚拟机和Java虚拟机的区别之后，接下来我们再简要分析一下Dalvik虚拟机的其它特性，包括内存管理、垃圾收集、JIT、JNI以及进程和线程管理。

## 内存管理

Dalvik虚拟机的内存大体上可以分为Java Object Heap、Bitmap Memory和Native Heap三种。

Java Object Heap是用来分配Java对象的，也就是我们在代码new出来的对象都是位于Java Object Heap上的。Dalvik虚拟机在启动的时候，可以通过-Xms和-Xmx选项来指定Java Object Heap的最小值和最大值。为了避免Dalvik虚拟机在运行的过程中对Java Object Heap的大小进行调整而影响性能，我们可以通过-Xms和-Xmx选项来将它的最小值和最大值设置为相等。

Java Object Heap的最小和最大默认值为2M和16M，但是手机在出厂时，厂商会根据手机的配置情况来对其进行调整，例如，G1、Droid、Nexus One和Xoom的Java Object Heap的最大值分别为16M、24M、32M 和48M。我们可以通过ActivityManager类的成员函数getMemoryClass来获得Dalvik虚拟机的Java Object Heap的最大值。

ActivityManager类的成员函数getMemoryClass的实现如下所示：

```java
public class ActivityManager {  
    ......  
  
    /** 
     * Return the approximate per-application memory class of the current 
     * device.  This gives you an idea of how hard a memory limit you should 
     * impose on your application to let the overall system work best.  The 
     * returned value is in megabytes; the baseline Android memory class is 
     * 16 (which happens to be the Java heap limit of those devices); some 
     * device with more memory may return 24 or even higher numbers. 
     */  
    public int getMemoryClass() {  
return staticGetMemoryClass();  
    }  
  
    /** @hide */  
    static public int staticGetMemoryClass() {  
// Really brain dead right now -- just take this from the configured  
// vm heap size, and assume it is in megabytes and thus ends with "m".  
String vmHeapSize = SystemProperties.get("dalvik.vm.heapsize", "16m");  
return Integer.parseInt(vmHeapSize.substring(0, vmHeapSize.length()-1));  
    }  
  
    ......  
}  
```
这个函数定义在文件frameworks/base/core/java/android/app/ActivityManager.java中。

Dalvik虚拟机在启动的时候，就是通过读取系统属性dalvik.vm.heapsize的值来获得Java Object Heap的最大值的，而ActivityManager类的成员函数getMemoryClass最终也通过读取这个系统属性的值来获得Java Object Heap的最大值。

这个Java Object Heap的最大值也就是我们平时所说的Android应用程序进程能够使用的最大内存。这里必须要注意的是，Android应用程序进程能够使用的最大内存指的是能够用来分配Java Object的堆。

Bitmap Memory也称为External Memory，它是用来处理图像的。在HoneyComb之前，Bitmap Memory是在Native Heap中分配的，但是这部分内存同样计入Java Object Heap中，也就是说，Bitmap占用的内存和Java Object占用的内存加起来不能超过Java Object Heap的最大值。这就是为什么我们在调用BitmapFactory相关的接口来处理大图像时，会抛出一个OutOfMemoryError异常的原因：

```
java.lang.OutOfMemoryError: bitmap size exceeds VM budget 
```
在HoneyComb以及更高的版本中，Bitmap Memory就直接是在Java Object Heap中分配了，这样就可以直接接受GC的管理。

Native Heap就是在Native Code中使用malloc等分配出来的内存，这部分内存是不受Java Object Heap的大小限制的，也就是它可以自由使用，当然它是会受到系统的限制。但是有一点需要注意的是，不要因为Native Heap可以自由使用就滥用，因为滥用Native Heap会导致系统可用内存急剧减少，从而引发系统采取激进的措施来Kill掉某些进程，用来补充可用内存，这样会影响系统体验。

此外，在HoneyComb以及更高的版本中，我们可以在AndroidManifest.xml的application标签中增加一个值等于“true”的android:largeHeap属性来通知Dalvik虚拟机应用程序需要使用较大的Java Object Heap。事实上，在内存受限的手机上，即使我们将一个应用程序的android:largeHeap属性设置为“true”，也是不能增加它可用的Java Object Heap的大小的，而即便是可以通过这个属性来增大Java Object Heap的大小，一般情况也不应该使用该属性。为了提高系统的整体体验，我们需要做的是致力于降低应用程序的内存需求，而不是增加增加应用程序的Java Object Heap的大小，毕竟系统总共可用的内存是固定的，一个应用程序用得多了，就意味意其它应用程序用得少了。

## 垃圾收集(GC)

Dalvik虚拟机可以自动回收那些不再使用了的Java Object，也就是那些不再被引用了的Java Object。垃圾自动收集机制将开发者从内存问题中解放出来，极大地提高了开发效率，以及提高了程序的可维护性。

我们知道，在C或者C++中，开发者需要手动地管理在堆中分配的内存，但是这往往导致很多问题。例如，内存分配之后忘记释放，造成内存泄漏。又如，非法访问那些已经释放了的内存，引发程序崩溃。如果没有一个好的C或者C++应用程序开发框架，一般的开发者根本无法驾驭内存问题，因为程序大了之后，很容易造成失控。最要命的是，内存被破坏的时候，并不一定就是程序崩溃的时候，它就是一颗不定时炸弹，说不准什么时候会被引爆，因此，查找原因是非常困难的。

从这里我们也可以推断出，Android为什么会选择Java而不是C/C++来作为应用程序开发语言，就是为了能够让开发远离内存问题，而将精力集中在业务上，开发出更多更好的APP来，从而迎头赶超[iOS](http://lib.csdn.net/base/ios)。当然，Android系统内存也存在大量的C/C++代码，这只要考虑性能问题，毕竟C/C++程序的运行性能整体上还是优于运行在虚拟机之上的Java程序的。不过，为了避免出现内存问题，在Android系统内部的C++代码，大量地使用了[智能指针](http://blog.csdn.net/luoshengyang/article/details/6786239)来自动管理对象的生命周期。选择Java来作为Android应用程序的开发语言，可以说是技术与商业之间一个折衷，事实证明，这种折衷是成功的。

回到正题，在GingerBread之前，Dalvik虚拟使用的垃圾收集机制有以下特点：

Stop-the-world，也就是垃圾收集线程在执行的时候，其它的线程都停止；

Full heap collection，也就是一次收集完全部的垃圾；

一次垃圾收集造成的程序中止时间通常都大于100ms。

在GingerBread以及更高的版本中，Dalvik虚拟使用的垃圾收集机制得到了改进，如下所示：

Cocurrent，也就是大多数情况下，垃圾收集线程与其它线程是并发执行的；

Partial collection，也就是一次可能只收集一部分垃圾；

一次垃圾收集造成的程序中止时间通常都小于5ms。

Dalvik虚拟机执行完成一次垃圾收集之后，我们通常可以看到类似以下的日志输出：

```
D/dalvikvm(9050): GC_CONCURRENT freed 2049K, 65% free 3571K/9991K, external 4703K/5261K, paused 2ms+2ms  
```

在这一行日志中，GC_CONCURRENT表示GC原因，2049K表示总共回收的内存，3571K/9991K表示Java Object Heap统计，即在9991K的Java Object Heap中，有3571K是正在使用的，4703K/5261K表示External Memory统计，即在5261K的External Memory中，有4703K是正在使用的，2ms+2ms表示垃圾收集造成的程序中止时间。

## 即时编译(JIT)

前面提到，JIT是相对AOT而言的，即JIT是在程序运行的过程中进行编译的，而AOT是在程序运行前进行编译的。在程序运行的过程中进行编译既有好处，也有坏处。好处在于可以利用程序的运行时信息来对编译出来的代码进行优化，而坏处在于占用程序的运行时间，也就是说不能花太多时间在代码编译和优化之上。

为了解决时间问题，JIT可能只会选择那些热点代码进行编译或者优化。根据2-8原则，一个程序80%的时间可能都是在重复执行20%的代码。因此，JIT就可以选择这20%经常执行的代码来进行编译和优化。

为了充分地利用好运行时信息来优化代码，JIT采用一种激进的方法。JIT在编译代码的时候，会对程序的运行情况进行假设，并且按照这种假设来对代码进行优化。随着程序的代码，如果前面的假设一直保持成立，那么JIT就什么也不用做，因此就可以提高程序的运行性能。一旦前面的假设不再成立了，那么JIT就需要对前面编译优化的代码进行调整，以便适应新的情况。这种调整成本可能是很昂贵的，但是只要假设不成立的情况很少或者几乎不会发生，那么获得的好处还是大于坏处的。由于JIT在编译和优化代码的时候，对程序的运行情况进行了假设，因此，它所采取的激进优化措施又称为赌博，即Gambling。

我们以一个例子来说明这种Gambling。我们知道，Java的同步原语涉及到Lock和Unlock操作。Lock和Unlock操作是非常耗时的，而且它们只有在多线程环境中才真的需要。但是一些同步函数或者同步代码，有程序运行的时候，有可能始终都是被单线程执行，也就是说，这些同步函数或者同步代码不会被多线程同时执行。这时候JIT就可以采取一种Lazy Unlocking机制。

当一个线程T1进入到一个同步代码C时，它还是按照正常的流程来获取一个轻量级锁L1，并且线程T1的ID会记录在轻量锁L1上。当经程T1离开同步函数或者同步代码时，它并不会释放前面获得的轻量级锁L1。当线程T1再次进入同步代码C时，它就会发现轻量级锁L的所有者正是自己，因此，它就可以直接执行同步代码C。这时候如果另外一个线程T2也要进入同步代码C，它就会发现轻量级锁L已经被线程T1获取。在这种情况下，JIT就需要检查线程T1的调用堆栈，看看它是否还在执行同步代码C。如果是的话，那么就需要将轻量级锁L1转换成一个重量级锁L2，并且将重量级锁L2的状态设置为锁定，然后再让线程T2在重量级锁L2上睡眠。等线程T1执行完成同步代码C之后，它就会按照正常的流程来释放重量级锁L2，从而唤醒线程T2来执行同步代码C。另一方面，如果线程T2在进入同步代码C的时候，JIT通过检查线程T1的调用堆栈，发现它已经离开同步代码C了，那么它就直接将轻量级锁L1的所有者记录为线程T2，并且让线程T2执行同步代码C。

通过上述的Lazy Unlocking机制，我们就可以充分地利用程序的运行时信息来提高程序的执行性能，这种优化对于静态编译的语言来说，是无法做到的。从这个角度来看，我们就可以说，静态编译语言（如C++）并不一定比在虚拟机上执行的语言（如Java）快，这是因为后者可以有一种强大的武器叫做JIT。

Dalvik虚拟机从Android 2.2版本开始，才支持JIT，而且是可选的。在编译Dalvik虚拟机的时候，可以通过WITH_JIT宏来将JIT也编译进去，而在启动Dalvik虚拟机的时候，可以通过-Xint:jit选项来开启JIT功能。

关于虚拟机JIT的实现原理的简要介绍，可以进一步参考这篇文章：[http://blog.reverberate.org/2012/12/hello-jit-world-joy-of-simple-jits.html](http://blog.reverberate.org/2012/12/hello-jit-world-joy-of-simple-jits.html)。

## Java本地调用(JNI)

无论如何，虚拟机最终都是运行在目标机器之上的，也就是说，它需要将自己的指令翻译成目标机器指令来执行，并且有些功能，需要通过调用目标机器运行的[操作系统](http://lib.csdn.net/base/operatingsystem)接口来完成。这样就需要有一个机制，使得函数调用可以从Java层穿越到Native层，也就是C/C++层。这种机制就称为Java本地调用，即JNI。当然，我们在执行Native代码的时候，有时候也是需要调用到Java函数的，这同样是可以通过JNI机制来实现。也就是说，JNI机制既支持在Java函数中调用C/C++函数，也支持在C/C++函数中调用Java函数。

事实上，Dalvik虚拟机提供的Java运行时库，大部分都是通过调用目标机器操作系统接口来实现的，也就是通过调用[Linux](http://lib.csdn.net/base/linux)系统接口来实现的。例如，当我们调用android.os.Process类的成员函数start来创建一个进程的时候，最终会调用到[linux](http://lib.csdn.net/base/linux)系统提供的fork系统调用来创建一个进程。

同时，为了方便开发者使用C/C++语言来开发应用程序，Android官方提供了NDK。通过NDK，我们就可以使用JNI机制来在Java函数中调用到C/C++函数。不过Android官方是不提倡使用NDK来开发应用程序的，这从它对NDK的支持远远不如SDK的支持就可以看得出来。

## 进程和线程管理

一般来说，虚拟机的进程和线程都是与目标机器本地操作系统的进程和线程一一对应的，这样做的好处是可以使本地操作系统来调度进程和线程。进程和线程调度是操作系统的核心模块，它的实现是非常复杂的，特别是考虑到多核的情况，因此，就完全没有必要在虚拟机中提供一个进程和线程库。

Dalvik虚拟机运行在Linux操作系统之上。我们知道，Linux操作系统并没有纯粹的线程概念，只要两个进程共享同一个地址空间，那么就可以认为它们同一个进程的两个线程。Linux操作系统提供了两个fork和clone两个调用，其中，前者就是用来创建进程的，而后者就是用来创建线程的。关于Linux操作系统的进程和线程的实现，可以参考在前面[Android学习启动篇](http://blog.csdn.net/luoshengyang/article/details/6557518)一文中提到的经典Linux内核书籍。

关于Android应用程序进程，它有两个很大的特点，下面我们就简要介绍一下。

第一个特点是每一个Android应用程序进程都有一个Dalvik虚拟机实例。这样做的好处是Android应用程序进程之间不会相互影响，也就是说，一个Android应用程序进程的意外中止，不会影响到其它的Android应用程序进程的正常运行。

第二个特点是每一个Android应用程序进程都是由一种称为Zygote的进程fork出来的。Zygote进程是由init进程启动起来的，也就是在系统启动的时候启动的。Zygote进程在启动的时候，会创建一个虚拟机实例，并且在这个虚拟机实例将所有的Java核心库都加载起来。每当Zygote进程需要创建一个Android应用程序进程的时候，它就通过复制自身来实现，也就是通过fork系统调用来实现。这些被fork出来的Android应用程序进程，一方面是复制了Zygote进程中的虚拟机实例，另一方面是与Zygote进程共享了同一套Java核心库。这样不仅Android应用程序进程的创建过程很快，而且由于所有的Android应用程序进程都共享同一套Java核心库而节省了内存空间。

关于Dalvik虚拟机的特性，我们就简要介绍到这里。事实上，Dalvik虚拟机和Java虚拟机的实现是类似的，例如，Dalvik虚拟机也支持JDWP（Java Debug Wire Protocol）协议，这样我们就可以使用DDMS来调试运行在Dalvik虚拟机中的进程。对Dalvik虚拟机的其它特性或者实现原理有兴趣的，建议都可以参考Java虚拟机的实现，这里提供三本参考书：

- Java Virtual Machine Specification ([Java SE](http://lib.csdn.net/base/javase) 7) 
- Inside the Java Virtual Machine, Second Edition
- [oracle](http://lib.csdn.net/base/oracle) JRockit: The Definitive Guide

另外，关于Dalvik虚拟机的指令集和dex文件格式的介绍，可以参考官方文档：[http://source.android.com/tech/dalvik/index.html](http://source.android.com/tech/dalvik/index.html)。如果对虚拟机的实现原理有兴趣的，还可以参考这个链接：[http://www.weibo.com/1595248757/zvdusrg15](http://www.weibo.com/1595248757/zvdusrg15)。

在这里，我们学习Dalvik虚拟机的目标是打通Java层到C/C++层之间的函数调用，从而可以更好地理解Android应用程序是如何在Linux内核上面运行的。为了达到这个目的，在接下来的文章中，我们将关注以下四个情景：

1. [Dalvik虚拟机的启动过程](http://blog.csdn.net/luoshengyang/article/details/8885792)
2. [ Dalvik虚拟机的运行过程](http://blog.csdn.net/luoshengyang/article/details/8914953)
3. [JNI函数的注册过程](http://blog.csdn.net/luoshengyang/article/details/8923483)
4. [Java进程和线程的创建过程](http://blog.csdn.net/luoshengyang/article/details/8923484)

掌握了这四个情景之后，再结合前面的所有文章，我们就可以从上到下地打通整个Android系统了，敬请关注！