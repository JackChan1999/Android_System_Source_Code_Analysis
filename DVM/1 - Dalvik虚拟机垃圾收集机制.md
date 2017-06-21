伴随着“Dalvik is dead，long live Dalvik“这行AOSP代码提交日志，在Android5.0中，ART运行时取代了Dalvik虚拟机。虽然Dalvik虚拟机不再使用，但是它曾经的作用是不可磨灭的。因此，在研究ART运行时的垃圾收集机制之前，先理解Dalvik虚拟机的垃圾收集机制也是很重要和有帮助的。因此，本文就对Dalvik虚拟机的垃圾收集机进行简要介绍和制定学习计划。

老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)，欢迎关注！

《[Android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

之所以说理解Dalvik虚拟机的垃圾收集机制对学习ART运行时的垃圾收集机制有帮助，是因为两者都使用到了一些共同或者相通的技术，并且前者的实现相对简单一些。这样我们就可以从简单的学起。等到有了一定的基础之后，再学习复杂的就会容易理解很多。

好了，废话不多说，我们开始介绍Dalvik虚拟机的垃圾收集机制涉及到的基本概念或者说术语，如图1所示：

![img](http://img.blog.csdn.net/20141123013605828?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图1 Dalvik虚拟机垃圾收集机制的基本概念

Dalvik虚拟机用来分配对象的堆划分为两部分，一部分叫做Active Heap，另一部分叫做Zygote Heap。从前面[Dalvik虚拟机的启动过程分析](http://blog.csdn.net/luoshengyang/article/details/8885792)这篇文章可以知道，[android](http://lib.csdn.net/base/android)系统的第一个Dalvik虚拟机是由Zygote进程创建的。再结合[Android应用程序进程启动过程的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6747696)这篇文章，我们可以知道，应用程序进程是由Zygote进程fork出来的。也就是说，应用程序进程使用了一种写时拷贝技术（COW）来复制了Zygote进程的地址空间。这意味着一开始的时候，应用程序进程和Zygote进程共享了同一个用来分配对象的堆。然而，当Zygote进程或者应用程序进程对该堆进行写操作时，内核就会执行真正的拷贝操作，使得Zygote进程和应用程序进程分别拥有自己的一份拷贝。

拷贝是一件费时费力的事情。因此，为了尽量地避免拷贝，Dalvik虚拟机将自己的堆划分为两部分。事实上，Dalvik虚拟机的堆最初是只有一个的。也就是Zygote进程在启动过程中创建Dalvik虚拟机的时候，只有一个堆。但是当Zygote进程在fork第一个应用程序进程之前，会将已经使用了的那部分堆内存划分为一部分，还没有使用的堆内存划分为另外一部分。前者就称为Zygote堆，后者就称为Active堆。以后无论是Zygote进程，还是应用程序进程，当它们需要分配对象的时候，都在Active堆上进行。这样就可以使得Zygote堆尽可能少地被执行写操作，因而就可以减少执行写时拷贝的操作。在Zygote堆里面分配的对象其实主要就是Zygote进程在启动过程中预加载的类、资源和对象了。这意味着这些预加载的类、资源和对象可以在Zygote进程和应用程序进程中做到长期共享。这样既能减少拷贝操作，还能减少对内存的需求。

明白了Dalvik虚拟机为什么要把用来分配对象的堆划分为Active堆和Zygote堆之后，我们再看到底堆是个什么东西，请看图2：

![img](http://img.blog.csdn.net/20141123020828718?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图2 Dalvik虚拟机的堆

在Dalvik虚拟机中，堆实际上就是一块匿名共享内存。关于Android系统的匿名共享内存，可以参考[Android系统匿名共享内存Ashmem（Anonymous Shared Memory）简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6651971)这个系列的文章。Dalvik虚拟机并不是直接管理这块匿名共享内存，而是将它封装成一个mspace，交给C库来管理。mspace是libc中的概念，我们可以通过libc提供的函数create_mspace_with_base创建一个mspace，然后再通过mspace_开头的函数管理该mspace。例如，我们可以通过mspace_malloc和mspace_bulk_free来在指定的mspace中分配和释放内存。实际上，我们在使用libc提供的函数malloc和free分配和释放内存时，也是在一个mspace进行的，只不过这个mspace是由libc默认创建的。

Dalvik虚拟机除了要给应用层分配对象之外，最重要的还是要对这些已经分配出去的对象进行管理，也就是要在对象不再被使用的时候，对其进行自动回收。没吃过猪肉，也见过猪跑，自动回收对象（也就是垃圾收集）的[算法](http://lib.csdn.net/base/datastructure)不用多说，就是耳熟能详的Mark-Sweep算法。

顾名思义，Mark-Sweep垃圾收集算法主要分为两个阶段：Mark和Sweep。Mark阶段从对象的根集开始标记被引用的对象。标记完成后，就进入到Sweep阶段，而Sweep阶段所做的事情就是回收没有被标记的对象占用的内存。这里涉及到的一个核心概念就是我们怎么标记对象有没有被引用的，换句说就是通过什么[数据结构](http://lib.csdn.net/base/datastructure)来描述对象有没有被引用。是的，就是图1中的Heap Bitmap了。Heap Bitmap的结构如图3所示：

![img](http://img.blog.csdn.net/20141123023350259?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图3 Heap Bitmap

从名字就可以推断出，Heap Bitmap使用位图来标记对象是否被使用。如果一个对象被引用，那么在Bitmap中与它对应的那一位就会被设置为1。否则的话，就设置为0。在Dalvik虚拟机中，使用一个unsigned long数组来描述一个Heap Bitmap。我们假设堆的大小为Max Heap Size。我们使用libc提供的函数mspace_malloc来从堆里面分配内存时，得到的内存的地址总是对齐到HB_OBJECT_ALIGNMENT的。HB_OBJECT_ALIGNMENT的值等于8，也就是说，我们分配的对象的地址的最低三位总是0。这意味着我们在考虑Bitmap中的位与对象的对应关系时，可以不考虑这最低三位的值。这样可以大大地减少Bitmap的大小。例如，在32位设备上，每一个对象的地址都有32位，除去最低的三位，我们在考虑Bitmap与对象的对应关系时，只需要考虑高29位就可以了。因此，在计算所需要的Bitmap的大小时，就可以将堆的大小值除以HB_OBJECT_ALIGNMENT，即除以8。

假设一个unsigned long数占HB_BITS_PER_WORD个位，那么，我们就需要一个大小为（Max Heap Size /  HB_OBJECT_ALIGNMENT / HB_BITS_PER_WORD）的unsigned long数组来描述一个大小为Max Heap Size的堆。在32位设备上，一个unsigned long数占用32位，即HB_BITS_PER_WORD的值等于32。综合上面的描述，我们就可以知道，一个大小为Max Heap Size的堆需要一个大小为（Max Heap Size / 8 / 32）的unsigned long数组来描述，即需要一块字节数等于（Max Heap Size / 8 / 32）× 4的内存来描述。

在图1中，我们使用了两个Bitmap来描述堆的对象，一个称为Live Bitmap，另一个称为Mark Bitmap。Live Bitmap用来标记上一次GC时被引用的对象，也就是没有被回收的对象，而Mark Bitmap用来标记当前GC有被引用的对象。有了这两个信息之后，我们就可以很容易地知道哪些对象是需要被回收的，即在Live Bitmap在标记为1，但是在Mark Bitmap中标记为0的对象。

在垃圾收集的Mark阶段，要求除了垃圾收集线程之外，其它的线程都停止，否则的话，就会可能导致不能正确地标记每一个对象。这种现象在垃圾收集算法中称为Stop The World，会导致程序中止执行，造成停顿的现象。为了尽可能地减少停顿，我们必须要允许在Mark阶段有条件地允许程序的其它线程执行。这种垃圾收集算法称为并行垃圾收集算法（Concurrent GC）。

为了实现Concurrent GC，Mark阶段又划分两个子阶段。第一个子阶段只负责标记根集对象。所谓的根集对象，就是指在GC开始的瞬间，被全局变量、栈变量和寄存器等引用的对象。有了这些根集变量之后，我们就可以顺着它们找到其余的被引用变量。例如，一个栈变量引了一个对象，而这个对象又通过成员变量引用了另外一个对象，那该被引用的对象也会同时标记为正在使用。这个标记被根集对象引用的对象的过程就是第二个子阶段。在Concurrent GC，第一个子阶段是不允许垃圾收集线程之外的线程运行的，但是第二个子阶段是允许的。不过，在第二个子阶段执行的过程中，如果一个线程修改了一个对象，那么该对象必须要记录起来，因为它很有可能引用了新的对象，或者引用了之前未引用过的对象。如果不这样做的话，那么就会导致被引用对象还在使用然而却被回收。这种情况出现在只进行部分垃圾收集的情况，这时候Card Table的作用就是用来记录非垃圾收集堆对象对垃圾收集堆对象的引用。Dalvik虚拟机进行部分垃圾收集时，实际上就是只收集在Active堆上分配的对象。因此对Dalvik虚拟机来说，Card Table就是用来记录在Zygote堆上分配的对象在部收垃圾收集执行过程中对在Active堆上分配的对象的引用。

我们是不是想到再用一个Bitmap在描述上述第二个子阶段被修改的对象呢？虽然我们尽大努力减少了用来标记对象的Bitmap的大小，不过还是比较可观的。因此，为了减少内存的消耗，我们使用另外一种技术来标记Mark第二子阶段被修改的对象。这种技术使用到了一种称为Card Table的数据结构，如图4所示：

![img](http://img.blog.csdn.net/20141123032427912?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图4 Card Table

从名字可以看出，Card Table由Card组成，一个Card实际上就是一个字节，它的值要么是CLEAN，要么是DIRTY。如果一个Card的值是CLEAN，就表示与它对应的对象在Mark第二子阶段没有被程序修改过。否则的话，就意味着被程序修改过，对于这些被修改过的对象。需要在Mark第二子阶段结束之后，再次禁止垃圾收集线程之外的其它线程执行，以便垃圾收集线程再次根据Card Table记录的信息对被修改过的对象引用的其它对象进行重新标记。由于Mark第二子阶段执行的时间不会太长，因此在该阶段被修改的对象不会很多，这样就可以保证第二次子阶段结束后，再次执行标记对象的过程是很快的，因而此时对程序造成的停顿非常小。

在Card Table中，在连续GC_CARD_SIZE地址中的对象共用一个Card。Dalvik虚拟机将GC_CARD_SIZE的值设置为128。因此，假设堆的大小为Max Heap Size，那么我们只需要一块字节数为（Max Heap Size / 128）的Card Table。相比大小为（Max Heap Size / 8 / 32）× 4的Bitmap，减少了一半的内存需求。

在Mark阶段，Dalvik虚拟机能过递归方式来标记对象。但是，这不是通过函数的递归调用来实现的，而是借助一个称为Mark Stack的栈来实现的。具体来说，当我们标记完成根集对象之后，就按照它们的地址从小到大的顺序标记它们所引用的其它对象。假设有A、B、C和D四个对象，它的地址大小关系为A < B < C < D，其中，B和D是根集对象，A被D引用，C没有被B和D引用。那么我们将依次遍历B和D。当遍历到B的时候，没有发现它引用其它对象，然后就继续向前遍历D对象。发现它引用了A对象。按照递归的算法，这时候除了标记A对象是正在使用之外，还应该去检查A对象有没有引用其它对象，然后又再检查它引用的对象有没有又引用其它的对象，一直这样遍历下去。这样就跟函数递归一样。更好的做法是将对象A记录在一个Mark Stack中，然后继续检查地址值比对象D大的其它对象。对于地址值比对象D大的其它对象，如果它们引用了一个地址值比它们小的其它对象，那么这些其它对象同样要记录在Mark Stack中。等到该轮检查结束之后，再回过头来检查记录在Mark Stack里面的对象。然后又重复上述过程，直到Mark Stack等于空为止。

这就是我们在图1中显示的Mark Stack的作用，它的具体结构如图5所示：

![img](http://img.blog.csdn.net/20141123040034475?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图5 Mark Stack

在Dalvik虚拟机中，每一个对象都是从Object类继承下来的，也就是说，每一个对象占用的内存大小都至少等于sizeof(Object)。此外，我们通过libc提供的函数mspace_malloc为对象分配内存时，libc需要额外的内存来记录被分配出去的内存的信息。例如，需要记录被分配出去的内存的大小。每一块分配出去的内存需要额外的HEAP_SOURCE_CHUNK_OVERHEAD内存来记录上述的管理信息。因此，在Dalvik虚拟机中，每一个对象的大小都至少为sizeof(Object) + HEAP_SOURCE_CHUNK_OVERHEAD。这就意味着对于一个大小为Max Heap Size的堆来说，最多可以分配Max Heap Size / (sizeof(Object) + HEAP_SOURCE_CHUNK_OVERHEAD)个对象。于是，在最坏情况下，我们就需要一个大小为（Max Heap Size / (sizeof(Object) + HEAP_SOURCE_CHUNK_OVERHEAD)）的Object*数组来描述Mark Stack，以便可以实现上述的非递归函数调用的递归标记算法。

至此，我们就对Dalvik虚拟机的垃圾收集机制中涉及到的基础概念分析完成了。没有结合代码来分析，可能这些概念一时还难以理解通透。不过不要紧，接下来我们将按照以下三个情景来结合源码深入分析上述的概念：

1. [Dalvik虚拟机堆的创建过程](http://blog.csdn.net/luoshengyang/article/details/41581063)

2. [Dalvik虚拟机的对象分配过程](http://blog.csdn.net/luoshengyang/article/details/41688319)

3. [Dalvik虚拟机的垃圾收集过程](http://blog.csdn.net/luoshengyang/article/details/41822747)

按照这三个情景学习Davlik虚拟机的垃圾收集机制之后，我们就会对上面涉及的概念有一个清晰的认识了，同时也会我们后面学习ART运行时的垃圾收集机集打下坚实的基础。敬请关注！想了解更多信息，也可以关注老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)