在[Android](http://lib.csdn.net/base/android)系统中，不同的应用程序是不能直接读写对方的数据文件的，如果它们想共享数据的话，只能通过Content Provider组件来实现。那么，Content Provider组件又是如何突破应用程序边界权限控制来实现在不同的应用程序之间共享数据的呢？在前面的文章中，我们已经简要介绍过它是通过Binder进程间通信机制以及匿名共享内存机制来实现的，在本文中，我们将详细分析它的数据共享原理。

《[android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

Android应用程序之间不能直接访问对方的数据文件的障碍在于每一个应用程序都有自己的用户ID，而每一个应用程序所创建的文件的读写权限都是只赋予给自己所属的用户，因此，就限制了应用程序之间相互读写数据的操作，关于Android应用程序的权限问题，具体可以参考前面一篇文章[Android应用程序组件Content Provider简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6946067)。通过前面[Android进程间通信（IPC）机制Binder简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6618363)等一系列文章的学习，我们知道，Binder进程间通信机制可以突破了以应用程序为边界的权限控制来实现在不同应用程序之间传输数据，而Content Provider组件在不同应用程序之间共享数据正是基于Binder进程间通信机制来实现的。虽然Binder进程间通信机制突破了以应用程序为边界的权限控制，但是它是安全可控的，因为数据的访问接口是由数据的所有者来提供的，换句话来说，就是数据提供方可以在接口层来实现安全控制，决定哪些数据是可以读，哪些数据可以写。虽然Content Provider组件本身也提供了读写权限控制，但是它的控制粒度是比较粗的，如果有需要，我们还是可以在接口访问层做更细粒度的权限控制以达到数据安全的目的。

Binder进程间通信机制虽然打通了应用程序之间共享数据的通道，但是还有一个问题需要解决，那就是数据要以什么来作来媒介来传输。我们知道，应用程序采用Binder进程间通信机制进行通信时，要传输的数据都是采用函数参数的形式进行的，对于一般的进程间调来来说，这是没有问题的，然而，对于应用程序之间的共享数据来说，它们的数据量可能是非常大的，如果还是简单的用函数参数的形式来传递，效率就会比较低下。通过前面[Android系统匿名共享内存Ashmem（Anonymous Shared Memory）简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6651971)等一系列文章的学习，我们知道，在应用程序进程之间以匿名共享内存的方式来传输数据效率是非常高的，因为它们之间只需要传递一个文件描述符就可以了。因此，Content Provider组件在不同应用程序之间传输数据正是基于匿名共享内存机制来实现的。

在继续分析Content Provider组件在不同应用程序之间共享数据的原理之前，我们假设应用程序之间需要共享的数据是保存在[数据库](http://lib.csdn.net/base/mysql)（SQLite）中的，因此，接下来的分析都是基于SQLite数据库游标（SQLiteCursor）来进行的。SQLiteCursor在共享数据的传输过程中发挥着重要的作用，因此，我们先来它和其它相关的类的关系图，如下所示：

![img](http://hi.csdn.net/attachment/201111/16/0_1321468014xnor.gif)

首先在第三方应用程序这一侧，当它需要访问Content Provider中的数据时，它会在本进程中创建一个CursorWindow对象，它在内部创建了一块匿名共享内存，同时，它实现了Parcel接口，因此它可以在进程间传输。接下来第三方应用程序把这个CursorWindow对象（连同它内部的匿名共享内存文件描述符）通过Binder进程间调用传输到Content Provider这一侧。这个匿名共享内存文件描述符传输到Binder驱动程序的时候，Binder驱动程序就会在目标进程（即Content Provider所在的进程）中创建另一个匿名共享文件描述符，指向前面已经创建好的匿名共享内存，因此，就实现了在两个进程中共享同一块匿名内存，这个过程具体可以参考[Android系统匿名共享内存Ashmem（Anonymous Shared Memory）在进程间共享的原理分析](http://blog.csdn.net/luoshengyang/article/details/6666491)一文。

在Content Provider这一侧，利用在Binder驱动程序为它创建好的这个匿名共享内存文件描述符，在本进程中创建了一个CursorWindow对象。现在，Content Provider开始要从本地中从数据库中查询第三方应用程序想要获取的数据了。Content Provider首先会创建一个SQLiteCursor对象，即SQLite数据库游标对象，它继承了AbstractWindowedCursor类，后者又继承了AbstractCursor类，而AbstractCursor类又实现了CrossProcessCursor和Cursor接口。其中，最重要的是在AbstractWindowedCursor类中，有一个成员变量mWindow，它的类型为CursorWindow，这个成员变量是通过AbstractWindowedCursor的子类SQLiteCursor的setWindow成员函数来设置的。这个SQLiteCursor对象设置好了父类AbstractWindowedCursor类的mWindow成员变量之后，它就具有传输数据的能力了，因为这个mWindow对象内部包含一块匿名共享内存。此外，这个SQLiteCursor对象的内部有两个成员变量，一个是SQLite数据库对象mDatabase，另外一个是SQLite数据库查询对象mQuery。SQLite数据库查询对象mQuery的类型为SQLiteQuery，它继承了SQLiteProgram类，后者又继承了SQLiteClosable类。SQLiteProgram类代表一个数据库存查询计划，它的成员变量mCompiledSql包含了一个已经编译好的SQL查询语句，SQLiteCursor对象就是利用这个编译好的SQL查询语句来获得数据的，但是它并不是马上就去获取数据的，而是等到需要时才去获取。

那么，要等到什么时候才会需要获取数据呢？一般来说，如果第三方应用程序在请求Content Provider返回数据时，如果指定了要返回关于这些数据的元信息时，例如数据条目的数量，那么Content Provider在把这个SQLiteCursor对象返回给第三方应用程序之前，就会去获取数据，因为只有获取了数据之后，才知道数据条目的数量是多少。SQLiteCursor对象通过调用成员变量mQuery的fillWindow成员函数来把从SQLite数据库中查询得到的数据保存其父类AbstractWindowedCursor的成员变量mWindow中去，即保存到第三方应用程序创建的这块匿名共享内存中去。如果第三方应用程序在请求Content Provider返回数据时，没有指定要返回关于这些数据的元信息，那么，就要等到第三方应用程序首次调用这个从Content Provider处返回的SQLiteCursor对象的数据获取方法时，才会真正执行从数据库存中查询数据的操作，例如调用了SQLiteCursor对象的getCount或者moveToFirst成员函数时。这是一种数据懒加载机制，需要的时候才去加载，这样就提高了数据传输过程中的效率。

上面说到，Content Provider向第三方应用程序返回的数据实际上是一个SQLiteCursor对象，那么，这个SQLiteCursor对象是如何传输到第三方应用程序的呢？因为它本身并不是一个Binder对象，我们需要对它进行适配一下。首先，Content Provider会根据这个SQLiteCursor对象来创建一个CursorToBulkCursorAdaptor适配器对象，这个适配器对象是一个Binder对象，因此，它可以在进程间传输，同时，它实现了IBulkCursor接口。Content Provider接着就通过Binder进程间通信机制把这个CursorToBulkCursorAdaptor对象返回给第三方应用程序，第三方应用程序得到了这个CursorToBulkCursorAdaptor之后，再在本地创建一个BulkCursorToCursorAdaptor对象，这个BulkCursorToCursorAdaptor对象的继承结构和SQLiteCursor对象是一样的，不过，它没有设置父类AbstractWindowedCursor的mWindow成员变量，因此，它只可以通过它内部的CursorToBulkCursorAdaptor对象引用来访问匿名共享内存中的数据，即通过访问Content Provider这一侧的SQLiteCursor对象来访问共享数据。

上面描述的数据共享模型还是比较复杂的，一下子理解不了也不要紧，下面我们还会结合第三方应用程序和Content Provider传输共享数据的完整过程来进一步分析Content Provider的数据共享原理，到时候再回过头来看这个数据共享模型就会清晰很多了。在接下来的内容中，我们就继续以[Android应用程序组件Content Provider应用实例](http://blog.csdn.net/luoshengyang/article/details/6950440)一文的例子来分析Content Provider在不同应用程序之间共享数据的原理。在[Android应用程序组件Content Provider应用实例](http://blog.csdn.net/luoshengyang/article/details/6950440)这篇文章介绍的应用程序Article中，它的主窗口MainActivity是通过调用它的内部ArticlesAdapter对象的getArticleByPos成员函数来把ArticlesProvider中的文章信息条目一条一条地取回来显示在ListView中的，在这篇文章中，我们就从ArticlesAdapter类的getArticleByPos函数开始，一步一步地分析第三方应用程序Article从ArticlesProvider这个Content Provider中获取数据的过程。同样，我们先来看看这个过程的序列图，然后再详细分析每一个步骤：

![img](http://hi.csdn.net/attachment/201111/16/0_1321468102VW5a.gif)

Step 1. ArticlesAdapter.getArticleByPos

这个函数定义在前面一篇文章[Android应用程序组件Content Provider应用实例](http://blog.csdn.net/luoshengyang/article/details/6950440)介绍的应用程序Artilce源代码工程目录下，在文件为packages/experimental/Article/src/shy/luo/article/ArticlesAdapter.[Java](http://lib.csdn.net/base/java)中：

```java
public class ArticlesAdapter {  
    ......  
  
    private ContentResolver resolver = null;  
  
    public ArticlesAdapter(Context context) {  
  resolver = context.getContentResolver();  
    }  
  
    ......  
  
    public Article getArticleByPos(int pos) {  
  Uri uri = ContentUris.withAppendedId(Articles.CONTENT_POS_URI, pos);  
  
  String[] projection = new String[] {  
      Articles.ID,  
      Articles.TITLE,  
      Articles.ABSTRACT,  
      Articles.URL  
  };  
  
  Cursor cursor = resolver.query(uri, projection, null, null, Articles.DEFAULT_SORT_ORDER);  
  if (!cursor.moveToFirst()) {  
      return null;  
  }  
  
  int id = cursor.getInt(0);  
  String title = cursor.getString(1);  
  String abs = cursor.getString(2);  
  String url = cursor.getString(3);  
  
  return new Article(id, title, abs, url);  
    }  
}  
```
这个函数通过应用程序上下文的ContentResolver接口resolver的query函数来获得与Articles.CONTENT_POS_URI这个URI对应的文章信息条目。常量Articles.CONTENT_POS_URI是在应用程序ArticlesProvider中定义的，它的值为“content://shy.luo.providers.articles/pos”，通过调用ContentUris.withAppendedId函数来在后面添加了一个整数，表示要获取指定位置的文章信息条目。这个位置是指

ArticlesProvider这个Content Provider中的所有文章信息条目按照Articles.DEFAULT_SORT_ORDER来排序后得到的位置的，常量Articles.DEFAULT_SORT_ORDER也是在应用程序ArticlesProvider中定义的，它的值为“_id asc”，即按照文章信息的ID值从小到大来排列。

Step 2. ContentResolver.query

这个函数定义在frameworks/base/core/java/android/content/ContentResolver.java文件中：

```java
public abstract class ContentResolver {  
    ......  
  
    public final Cursor query(Uri uri, String[] projection,  
      String selection, String[] selectionArgs, String sortOrder) {  
  IContentProvider provider = acquireProvider(uri);  
  if (provider == null) {  
      return null;  
  }  
  try {  
      ......  
      Cursor qCursor = provider.query(uri, projection, selection, selectionArgs, sortOrder);  
      ......  
        
      return new CursorWrapperInner(qCursor, provider);  
  } catch (RemoteException e) {  
      ......  
  } catch(RuntimeException e) {  
      ......  
  }  
    }  
  
    ......  
}  
```
这个函数首先通过调用acquireProvider函数来获得与参数uri对应的Content Provider接口，然后再通过这个接口的query函数来获得相应的数据。我们先来看看acquireProvider函数的实现，再回过头来分析这个Content Provider接口的query函数的实现。

Step 3. ContentResolver.acquireProvider

Step 4. ApplicationContentResolver.acquireProvider

Step 5. ActivityThread.acquireProvider

Step 6. ActivityThread.getProvider

从Step 3到Step 6是获取与上面Step 2中传进来的参数uri对应的Content Provider接口的过程。在前面一篇文章[Android应用程序组件Content Provider的启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6963418)中，我们已经详细介绍过这个过程了，这里不再详述。不过这里我们假设，这个Content Provider接口之前已经创建好了，因此，在Step 6的ActivityThread.getProvider函数中，通过getExistingProvider函数就把之前已经好的Content Provider接口返回来了。

回到Step 2中的ContentResolver.query函数中，它继续调用这个返回来的Content Provider接口来获取数据。从这篇文章[Android应用程序组件Content Provider的启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6963418)中，我们知道，这个Content Provider接口实际上是一个在ContentProvider类的内部所创建的一个Transport对象的远程接口。这个Transport类继承了ContentProviderNative类，是一个Binder对象的Stub类，因此，接下来就会进入到这个Binder对象的Proxy类ContentProviderProxy中执行query函数。

Step 7. ContentProviderProxy.query

这个函数定义在frameworks/base/core/java/android/content/ContentProviderNative.java文件中：

```java
final class ContentProviderProxy implements IContentProvider {  
    ......  
  
    public Cursor query(Uri url, String[] projection, String selection,  
      String[] selectionArgs, String sortOrder) throws RemoteException {  
  //TODO make a pool of windows so we can reuse memory dealers  
  CursorWindow window = new CursorWindow(false /* window will be used remotely */);  
  BulkCursorToCursorAdaptor adaptor = new BulkCursorToCursorAdaptor();  
  IBulkCursor bulkCursor = bulkQueryInternal(  
      url, projection, selection, selectionArgs, sortOrder,  
      adaptor.getObserver(), window,  
      adaptor);  
  if (bulkCursor == null) {  
      return null;  
  }  
  return adaptor;  
    }  
  
    ......  
}  
```
这个函数首先会创建一个CursorWindow对象，前面已经说过，这个CursorWindow对象包含了一块匿名共享内存，它的作用是把这块匿名共享内存通过Binder进程间通信机制传给Content Proivder，好让Content Proivder在里面返回所请求的数据。下面我们就先看看这个CursorWindow对象的创建过程，重点关注它是如何在内部创建匿名共享内存的，然后再回过头来看看它调用bulkQueryInternal函数来做了些什么事情。

CursorWindow类定义在frameworks/base/core/java/android/database/CursorWindow.java文件中，我们来看看它的构造函数的实现：

```java
public class CursorWindow extends SQLiteClosable implements Parcelable {  
    ......  
  
    private int nWindow;  
  
    ......  
  
    public CursorWindow(boolean localWindow) {  
  ......  
  
  native_init(localWindow);  
    }  
  
    ......  
}  
```
它主要调用本地方法native_init来执行初始化的工作，主要就是创建匿名共享内存了，传进来的参数localWindow为false，表示这个匿名共享内存只能通过远程调用来访问，即前面我们所说的，通过Content Proivder返回来的Cursor接口来访问这块匿名共享内存里面的数据。

Step 8. CursorWindow.native_init

这是一个JNI方法，它对应定义在frameworks/base/core/jni/android_database_CursorWindow.cpp文件中的native_init_empty函数：

```c

static JNINativeMethod sMethods[] =  
{  
     /* name, signature, funcPtr */  
    {"native_init", "(Z)V", (void *)native_init_empty},  
    ......  
};  

 函数native_init_empty的定义如下所示：

```c

static void native_init_empty(JNIEnv * env, jobject object, jboolean localOnly)  
{  
    ......  
  
    CursorWindow * window;  
  
    window = new CursorWindow(MAX_WINDOW_SIZE);  
    ......  
  
    if (!window->initBuffer(localOnly)) {  
  ......  
    }  
  
    ......  
    SET_WINDOW(env, object, window);  
}  
```
这个函数在C++层创建了一个CursorWindow对象，然后通过调用SET_WINDOW宏来把这个C++层的CursorWindow对象与Java层的CursorWindow对象关系起来：

```c

\#define SET_WINDOW(env, object, window) (env->SetIntField(object, gWindowField, (int)window))  
```
这里的gWindowField即定义为Java层的CursorWindow对象中的nWindow成员变量：

```c

static jfieldID gWindowField;  
  
......  
  
int register_android_database_CursorWindow(JNIEnv * env)  
{  
    jclass clazz;  
  
    clazz = env->FindClass("android/database/CursorWindow");  
    ......  
  
    gWindowField = env->GetFieldID(clazz, "nWindow", "I");  
  
    ......  
}  
```
这种用法在Android应用程序框架层中非常普遍。

下面我们重点关注C++层的CursorWindow对象的initBuffer函数的实现。

Step 9. CursorWindow.initBuffer

C++层的CursorWindow类定义在frameworks/base/core/jni/CursorWindow.cpp文件中：

```c

bool CursorWindow::initBuffer(bool localOnly)  
{  
    ......  
  
    sp<MemoryHeapBase> heap;  
    heap = new MemoryHeapBase(mMaxSize, 0, "CursorWindow");  
    if (heap != NULL) {  
  mMemory = new MemoryBase(heap, 0, mMaxSize);  
  if (mMemory != NULL) {  
      mData = (uint8_t *) mMemory->pointer();  
      if (mData) {  
          mHeader = (window_header_t *) mData;  
          mSize = mMaxSize;  
  
          ......  
      }  
  }  
  ......  
    } else {  
  ......  
    }  
}  
```
这里我们就可以很清楚地看到，在CursorWindow类的内部有一个成员变量mMemory，它的类型是MemoryBase。MemoryBase类为我们封装了匿名共享内存的访问以及在进程间的传输等问题，具体可以参考前面一篇文章

Android系统匿名共享内存（Anonymous Shared Memory）C++调用接口分析

，这里就不再详述了。

通过Step 8和Step 9两步，用来在第三方应用程序和Content Provider之间传输数据的媒介就准备好了，我们回到Step 7中，看看系统是如何把这个匿名共享存传递给Content Provider使用的。在Step 7中，最后调用bulkQueryInternal函数来进一步操作。

Step 10. ContentProviderProxy.bulkQueryInternal

这个函数定义在frameworks/base/core/java/android/content/ContentProviderNative.java文件中：

```java
final class ContentProviderProxy implements IContentProvider  
{  
    ......  
  
    private IBulkCursor bulkQueryInternal(  
      Uri url, String[] projection,  
      String selection, String[] selectionArgs, String sortOrder,  
      IContentObserver observer, CursorWindow window,  
      BulkCursorToCursorAdaptor adaptor) throws RemoteException {  
  Parcel data = Parcel.obtain();  
  Parcel reply = Parcel.obtain();  
  
  data.writeInterfaceToken(IContentProvider.descriptor);  
  
  url.writeToParcel(data, 0);  
  int length = 0;  
  if (projection != null) {  
      length = projection.length;  
  }  
  data.writeInt(length);  
  for (int i = 0; i < length; i++) {  
      data.writeString(projection[i]);  
  }  
  data.writeString(selection);  
  if (selectionArgs != null) {  
      length = selectionArgs.length;  
  } else {  
      length = 0;  
  }  
  data.writeInt(length);  
  for (int i = 0; i < length; i++) {  
      data.writeString(selectionArgs[i]);  
  }  
  data.writeString(sortOrder);  
  data.writeStrongBinder(observer.asBinder());  
  window.writeToParcel(data, 0);  
  
  // Flag for whether or not we want the number of rows in the  
  // cursor and the position of the "_id" column index (or -1 if  
  // non-existent).  Only to be returned if binder != null.  
  final boolean wantsCursorMetadata = (adaptor != null);  
  data.writeInt(wantsCursorMetadata ? 1 : 0);  
  
  mRemote.transact(IContentProvider.QUERY_TRANSACTION, data, reply, 0);  
  
  DatabaseUtils.readExceptionFromParcel(reply);  
  
  IBulkCursor bulkCursor = null;  
  IBinder bulkCursorBinder = reply.readStrongBinder();  
  if (bulkCursorBinder != null) {  
      bulkCursor = BulkCursorNative.asInterface(bulkCursorBinder);  
  
      if (wantsCursorMetadata) {  
          int rowCount = reply.readInt();  
          int idColumnPosition = reply.readInt();  
          if (bulkCursor != null) {  
              adaptor.set(bulkCursor, rowCount, idColumnPosition);  
          }  
      }  
  }  
  
  data.recycle();  
  reply.recycle();  
  
  return bulkCursor;  
    }  
  
    ......  
}  
```
这个函数有点长，不过它的逻辑很简单，就是把查询参数都写到一个Parcel对象data中去，然后通过下面Binder进程间通信机制把查询请求传给Content Provider处理：

```java
mRemote.transact(IContentProvider.QUERY_TRANSACTION, data, reply, 0);  
```
从这个Binder调用返回以后，就会得到一个IBulkCursor接口，它是一个Binder引用，实际是指向在Content Provider这一侧创建的一个CursorToBulkCursorAdaptor对象，后面我们将会看到。有了这个IBulkCursor接口之后，我们就可以通过Binder进程间调用来访问从Content Provider中查询得到的数据了。这个IBulkCursor接口最终最设置了上面Step 7中创建的BulkCursorToCursorAdaptor对象adaptor中去：

```java
adaptor.set(bulkCursor, rowCount, idColumnPosition);  
```

BulkCursorToCursorAdaptor类实现了Cursor接口，因此，我们可以通过Curosr接口来访问这些查询得到的共享数据。

在前面把查询参数写到Parcel对象data中去的过程中，有两个步骤是比较重要的，分别下面这段执行语句：

```java
window.writeToParcel(data, 0);  
  
// Flag for whether or not we want the number of rows in the  
// cursor and the position of the "_id" column index (or -1 if  
// non-existent).  Only to be returned if binder != null.  
final boolean wantsCursorMetadata = (adaptor != null);  
data.writeInt(wantsCursorMetadata ? 1 : 0);  
```
调用window.writeToParcel是把window对象内部的匿名共享内存块通过Binder进程间通信机制传输给Content Provider来使用；而当传进来的参数adaptor不为null时，就会往data中写入一个整数1，表示让Content Provider把查询得到数据的元信息一起返回来，例如数据的行数、数据行的ID列的索引位置等信息，这个整数值会促使Content Provider把前面说的IBulkCursor接口返回给第三方应用程序之前，真正执行一把数据库查询操作，后面我们将看到这个过程。

现在，我们重点来关注一下CursorWindow类的writeToParcel函数，看看它是如何把它内部的匿名共享内存对象写到数据流data中去的。

Step 11. CursorWindow.writeToParcel

这个函数定义在frameworks/base/core/java/android/database/CursorWindow.java文件中：

```java
public class CursorWindow extends SQLiteClosable implements Parcelable {  
    ......  
  
    public void writeToParcel(Parcel dest, int flags) {  
  ......  
  dest.writeStrongBinder(native_getBinder());  
  ......  
    }  
  
    ......  
}  
```
这个函数最主要的操作就是往数据流dest写入一个Binder对象，这个Binder对象是通过调用本地方法native_getBinder来得到的。

Step 12. CursorWindow.native_getBinder

这个函数定义在frameworks/base/core/jni/android_database_CursorWindow.cpp文件中：

```c

static jobject native_getBinder(JNIEnv * env, jobject object)  
{  
    CursorWindow * window = GET_WINDOW(env, object);  
    if (window) {  
  sp<IMemory> memory = window->getMemory();  
  if (memory != NULL) {  
      sp<IBinder> binder = memory->asBinder();  
      return javaObjectForIBinder(env, binder);  
  }  
    }  
    return NULL;  
}  
```
在前面的Step 8中，我们在C++层创建了一个CursorWindow对象，这个对象保存在Java层创建的CursorWindow对象的成员变量nWindow中，这里通过GET_WINDOW宏来把这个在C++层创建的CurosrWindow对象返回来：

```c

\#define GET_WINDOW(env, object) ((CursorWindow *)env->GetIntField(object, gWindowField))  
```
获得了这个CursorWindow对象以后，就调用它的getMemory函数来获得一个IMemory接口，这是一个Binder接口，具体可以参考前面一篇文章

Android系统匿名共享内存（Anonymous Shared Memory）C++调用接口分析

Step 13. CursorWindow.getMemory

这个函数定义在frameworks/base/core/jni/CursorWindow.h文件中：

```c

class CursorWindow  
{  
public:  
    ......  
  
    sp<IMemory>         getMemory() {return mMemory;}  
  
    ......  
}  
```
这个CursorWindow对象的成员变量mMemory就是在前面Step 9中创建的了。

这样，在第三方应用程序这一侧创建的匿名共享存对象就可以传递给Content Provider来使用了。回到前面的Step 10中，所有的参数都就准备就绪以后，就通过Binder进程间通信机制把数据查询请求发送给相应的Content Proivder了。这个请求是在ContentProviderNative类的onTransact函数中响应的。

Step 14. ContentProviderNative.onTransact

这个函数定义在frameworks/base/core/java/android/content/ContentProviderNative.java文件中：

```java
abstract public class ContentProviderNative extends Binder implements IContentProvider {  
    ......  
  
    @Override  
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags)  
    throws RemoteException {  
  try {  
      switch (code) {  
      case QUERY_TRANSACTION:  
          {  
              data.enforceInterface(IContentProvider.descriptor);  
  
              Uri url = Uri.CREATOR.createFromParcel(data);  
  
              // String[] projection  
              int num = data.readInt();  
              String[] projection = null;  
              if (num > 0) {  
                  projection = new String[num];  
                  for (int i = 0; i < num; i++) {  
                      projection[i] = data.readString();  
                  }  
              }  
  
              // String selection, String[] selectionArgs...  
              String selection = data.readString();  
              num = data.readInt();  
              String[] selectionArgs = null;  
              if (num > 0) {  
                  selectionArgs = new String[num];  
                  for (int i = 0; i < num; i++) {  
                      selectionArgs[i] = data.readString();  
                  }  
              }  
  
              String sortOrder = data.readString();  
              IContentObserver observer = IContentObserver.Stub.  
                  asInterface(data.readStrongBinder());  
              CursorWindow window = CursorWindow.CREATOR.createFromParcel(data);  
  
              // Flag for whether caller wants the number of  
              // rows in the cursor and the position of the  
              // "_id" column index (or -1 if non-existent)  
              // Only to be returned if binder != null.  
              boolean wantsCursorMetadata = data.readInt() != 0;  
  
              IBulkCursor bulkCursor = bulkQuery(url, projection, selection,  
                  selectionArgs, sortOrder, observer, window);  
              reply.writeNoException();  
              if (bulkCursor != null) {  
                  reply.writeStrongBinder(bulkCursor.asBinder());  
  
                  if (wantsCursorMetadata) {  
                      reply.writeInt(bulkCursor.count());  
                      reply.writeInt(BulkCursorToCursorAdaptor.findRowIdColumnIndex(  
                          bulkCursor.getColumnNames()));  
                  }  
              } else {  
                  reply.writeStrongBinder(null);  
              }  
  
              return true;  
          }  
      ......  
      }  
  } catch (Exception e) {  
      DatabaseUtils.writeExceptionToParcel(reply, e);  
      return true;  
  }  
  
  return super.onTransact(code, data, reply, flags);  
    }  
  
    ......  
}  
```
这一步其实就是前面Step 10的逆操作，把请求参数从数据流data中读取出来。这里我们同样是重点关注下面这两个参数读取的步骤：

```java
CursorWindow window = CursorWindow.CREATOR.createFromParcel(data);  
  
// Flag for whether caller wants the number of  
// rows in the cursor and the position of the  
// "_id" column index (or -1 if non-existent)  
// Only to be returned if binder != null.  
boolean wantsCursorMetadata = data.readInt() != 0;  
```
通过调用CursorWindow.CREATOR.createFromParcel函数来从数据流data中重建一个本地的CursorWindow对象；接着又将数据流data的下一个整数值读取出来，如果这个整数值不为0，变量wantsCursorMetadata的值就为true，说明Content Provider在返回IBulkCursor接口给第三方应用程序之前，要先实际执行一把数据库查询操作，以便把结果数据的元信息返回给第三方应用程序。

通过下面的代码我们可以看到，调用bulkQuery函数之后，就得到了一个IBulkCursor接口，这表示要返回的数据准备就绪了，但是这时候实际上还没有把结果数据从数据库中提取出来，而只是准备好了一个SQL查询计划，等到真正要使用这些结果数据时，系统才会真正执行查询数据库的操作:

```java
if (wantsCursorMetadata) {  
    reply.writeInt(bulkCursor.count());  
    ......  
}  
```
在将这个IBulkCursor接口返回给第三方应用程序之前，如果发现

wantsCursorMetadata的值就为true，就会调用它的count函数来获得结果数据的总行数，这样就会导致系统真正去执行数据库查询操作，并把结果数据保存到前面得到的CursorWindow对象中的匿名共享内存中去。

下面我们就重点关注CursorWindow.CREATOR.createFromParcel函数是如何从数据流data中在本地构造一个CursorWindow对象的。

Step 15. CursorWindow.CREATOR.createFromParcel

这个函数定义在frameworks/base/core/java/android/database/CursorWindow.java文件中：

```java
public class CursorWindow extends SQLiteClosable implements Parcelable {  
    ......  
  
    private CursorWindow(Parcel source) {  
  IBinder nativeBinder = source.readStrongBinder();  
  ......  
  
  native_init(nativeBinder);  
    }  
  
    ......  
  
    public static final Parcelable.Creator<CursorWindow> CREATOR  
      = new Parcelable.Creator<CursorWindow>() {  
  public CursorWindow createFromParcel(Parcel source) {  
      return new CursorWindow(source);  
  }  
  
  ......  
    };  
  
    ......  
}  
```
在创建CursorWindow对象的过程中，首先是从数据流source中将在前面Step 10中写入的Binder接口读取出来，然后使用这个Binder接口来初始化这个CursorWindow对象，通过前面的Step 13，我们知道，这个Binder接口的实际类型为IMemory，它封装了对匿名共享内存的访问操作。初始化这个匿名共享内存对象的操作是由本地方法native_init函数来实现的，下面我们就看看它的实现。

Step 16. CursorWindow.native_init

这个函数定义在frameworks/base/core/jni/android_database_CursorWindow.cpp文件中，对应的函数为native_init_memory函数：

```c
static JNINativeMethod sMethods[] =  
{  
    ......  
    {"native_init", "(Landroid/os/IBinder;)V", (void *)native_init_memory},  
};  
```
函数native_init_memory的实现如下所示：

```c
static void native_init_memory(JNIEnv * env, jobject object, jobject memObj)  
{  
    sp<IMemory> memory = interface_cast<IMemory>(ibinderForJavaObject(env, memObj));  
    ......  
  
    CursorWindow * window = new CursorWindow();  
    ......  
    if (!window->setMemory(memory)) {  
 ......  
    }  
  
    ......  
    SET_WINDOW(env, object, window);  
}  
```
函数首先是将前面Step 15中传进来的Binder接口转换为IMemory接口，接着创建一个C++层的CursorWindow对象，再接着用这个IMemory接口来初始化这个C++层的CursorWindow对象，最后像前面的Step 8一样，通过宏SET_WINDOW把这个C++层的CursorWindow对象和前面在Step 15中创建的Java层CursorWindow对象关联起来。

下面我们就重点关注CursorWindow类的setMemory函数的实现，看看它是如何使用这个IMemory接口来初始化其内部的匿名共享内存对象的。

Step 17. CursorWindow.setMemory

这个函数定义在frameworks/base/core/jni/CursorWindow.cpp文件中：

```c
bool CursorWindow::setMemory(const sp<IMemory>& memory)  
{  
    mMemory = memory;  
    mData = (uint8_t *) memory->pointer();  
    ......  
    mHeader = (window_header_t *) mData;  
  
    // Make the window read-only  
    ssize_t size = memory->size();  
    mSize = size;  
    mMaxSize = size;  
    mFreeOffset = size;  
    ......  
    return true;  
}  
```
从前面一篇文章

[Android系统匿名共享内存（Anonymous Shared Memory）C++调用接口分析](http://blog.csdn.net/luoshengyang/article/details/6939890)中，我们知道，这里得到的IMemory接口，实际上是一个Binder引用，它指向前面在Step 9中创建的MemoryBase对象，当我们第一次调用这个接口的pointer函数时，它便会通过Binder进程间通信机制去请求这个MemoryBase对象把它内部的匿名共享内存文件描述符返回来给它，而Binder驱动程序发现要传输的是一个文件描述符的时候，就会在目标进程中创建另外一个文件描述符，这个新建的文件描述符与要传输的文件描述符指向的是同一个文件，在我们这个情景中，这个文件就是我们前面创建的匿名共享内存文件了。因此，在目标进程中，即在Content Provider进程中，它可以通过这个新建的文件描述符来访问这块匿名共享内存，这也是匿名共享内存在进程间的共享原理，具体可以参考另外一篇文章[Android系统匿名共享内存Ashmem（Anonymous Shared Memory）在进程间共享的原理分析](http://blog.csdn.net/luoshengyang/article/details/6666491)。

这样，在Content Provider这一侧，就可以把第三方应用程序请求的数据保存在这个匿名共享内存中了，回到前面的Step 14中，下一步要执行的函数便是bulkQuery了，它的作用为请求的数据制定好一个SQL数据库查询计划。这个bulkQuery函数是由一个实现了IContentProvider接口的Binder对象来实现的，具体可以参考前面一篇文章[Android应用程序组件Content Provider的启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6963418)中，这个Binder对象的实际类型是定义在ContentProivder类内部的Transport类。

Step 18. Transport.bulkQuery

这个函数定义在frameworks/base/core/java/android/content/ContentProvider.java文件中：

```java
public abstract class ContentProvider implements ComponentCallbacks {  
    ......  
  
    class Transport extends ContentProviderNative {  
  ......  
  
  public IBulkCursor bulkQuery(Uri uri, String[] projection,  
          String selection, String[] selectionArgs, String sortOrder,  
          IContentObserver observer, CursorWindow window) {  
      ......  
  
      Cursor cursor = ContentProvider.this.query(uri, projection,  
          selection, selectionArgs, sortOrder);  
      ......  
  
      return new CursorToBulkCursorAdaptor(cursor, observer,  
          ContentProvider.this.getClass().getName(),  
          hasWritePermission(uri), window);  
  }  
  
  ......  
    }  
  
    ......  
}  
```
这个函数主要做了两件事情，一是调用ContentProvider的子类的query函数构造一个数据库查询计划，注意，从这个函数返回来的时候，还没有真正执行数据库查询的操作，而只是按照查询条件准备好了一个SQL语句，要等到第一次使用的时候才会去执行数据库查询操作；二是使用前面一步得到的Cursor接口以及传下来的参数window来创建一个CursorToBulkCursorAdaptor对象，这个对象实现了IBulkCursor接口，同时它也是一个Binder对象，是用来返回给第三方应用程序使用的，第三方应用程序必须通过这个接口来获取从ContentProvider中查询得到的数据，而这个CursorToBulkCursorAdaptor对象的功能就是利用前面获得的Cursor接口来执行数据库查询操作，然后把查询得到的结果保存在从参数传下来的window对象内部所引用的匿名共享内存中去。我们先来看ContentProvider的子类的query函数的实现，在我们这个情景中，这个子类就是ArticlesProvider了，然后再回过头来看看这个CursorToBulkCursorAdaptor对象是如何把数据库查询计划与匿名共享内存关联起来的。

Step 19. ArticlesProvider.query

这个函数定义在前面一篇文章[Android应用程序组件Content Provider应用实例](http://blog.csdn.net/luoshengyang/article/details/6950440)介绍的应用程序ArtilcesProvider源代码工程目录下，在文件packages/experimental/ArticlesProvider/src/shy/luo/providers/articles/ArticlesProvider.java 中：

```java
public class ArticlesProvider extends ContentProvider {  
    ......  
  
    @Override  
    public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {  
  SQLiteDatabase db = dbHelper.getReadableDatabase();  
  
  SQLiteQueryBuilder sqlBuilder = new SQLiteQueryBuilder();  
  String limit = null;  
  
  switch (uriMatcher.match(uri)) {  
      ......  
      case Articles.ITEM_POS: {  
          String pos = uri.getPathSegments().get(1);  
          sqlBuilder.setTables(DB_TABLE);  
          sqlBuilder.setProjectionMap(articleProjectionMap);  
          limit = pos + ", 1";  
          break;  
      }  
      ......  
  }  
  
  Cursor cursor = sqlBuilder.query(db, projection, selection, selectionArgs, null, null, TextUtils.isEmpty(sortOrder) ? Articles.DEFAULT_SORT_ORDER : sortOrder, limit);  
  ......  
  
  return cursor;  
    }  
  
    ......  
}  
```
从前面的Step 1中可以看到，传进来的参数uri的值为“content://shy.luo.providers.articles/pos”，通过uriMatcher的match函数来匹配这个uri的时候，得到的匹配码为Articles.ITEM_POS，这个知识点可以参考前面这篇文章

[Android应用程序组件Content Provider应用实例](http://blog.csdn.net/luoshengyang/article/details/6950440)。因为我们的数据是保存在SQLite数据库里面的，因此，必须要构造一个SQL语句来将所请求的数据查询出来。这里是通过SQLiteQueryBuilder类来构造这个SQL查询语句的，构造好了以后，就调用它的query函数来准备一个数据库查询计划。

Step 20. SQLiteQueryBuilder.query

这个函数定义在frameworks/base/core/java/android/database/sqlite/SQLiteQueryBuilder.java文件中：

```java
public class SQLiteQueryBuilder  
{  
    ......  
  
    public Cursor query(SQLiteDatabase db, String[] projectionIn,  
      String selection, String[] selectionArgs, String groupBy,  
      String having, String sortOrder, String limit) {  
  ......  
  
  String sql = buildQuery(  
      projectionIn, selection, groupBy, having,  
      sortOrder, limit);  
  
  ......  
  
  return db.rawQueryWithFactory(  
      mFactory, sql, selectionArgs,  
      SQLiteDatabase.findEditTable(mTables));  
    }  
  
    ......  
}  
```
这里首先是调用buildQuery函数来构造一个SQL语句，它无非就是根据从参数传来列名子句、select子句、where子句、group by子句、having子句、order子句以及limit子句来构造一个完整的SQL子句，这些都是SQL语法的基础知识了，这里我们就不关注了。构造好这个SQL查询语句之后，就调用从参数传下来的数据库对象db的rawQueryWithFactory函数来进一步操作了。

 Step 21. SQLiteDatabase.rawQueryWithFactory

 这个函数定义在frameworks/base/core/java/android/database/sqlite/SQLiteDatabase.java文件中：

```java
public class SQLiteDatabase extends SQLiteClosable {  
    ......  
  
    public Cursor rawQueryWithFactory(  
      CursorFactory cursorFactory, String sql, String[] selectionArgs,  
      String editTable) {  
  ......  
  
  SQLiteCursorDriver driver = new SQLiteDirectCursorDriver(this, sql, editTable);  
  
  Cursor cursor = null;  
  try {  
      cursor = driver.query(  
          cursorFactory != null ? cursorFactory : mFactory,  
          selectionArgs);  
  } finally {  
      ......  
  }  
  
  return cursor;  
    }  
  
    ......  
}  
```
这个函数会在内部创建一个SQLiteCursorDriver对象driver，然后调用它的query函数来创建一个Cursor对象，这个Cursor对象的实际类型是SQLiteCursor，下面我们将会看到，前面我们也已经看到，这个

SQLiteCursor的内部就包含了一个数据库查询计划。

Step 22. SQLiteCursorDriver.query

这个函数定义在frameworks/base/core/java/android/database/sqlite/SQLiteDirectCursorDriver.java文件中：

```java
public class SQLiteDirectCursorDriver implements SQLiteCursorDriver {  
    ......  
  
    public Cursor query(CursorFactory factory, String[] selectionArgs) {  
  // Compile the query  
  SQLiteQuery query = new SQLiteQuery(mDatabase, mSql, 0, selectionArgs);  
  
  try {  
      ......  
  
      // Create the cursor  
      if (factory == null) {  
          mCursor = new SQLiteCursor(mDatabase, this, mEditTable, query);  
      } else {  
          mCursor = factory.newCursor(mDatabase, this, mEditTable, query);  
      }  
  
      ......  
      return mCursor;  
  } finally {  
      ......  
  }  
    }  
  
    ......  
}  
```
这里我们就可以清楚地看到，这个函数首先会根据数据库对象mDatabase和原生SQL语句来构造一个SQLiteQuery对象，这个对象的创建的过程中，就会解析这个原生SQL语句，并且创建好数据库查询计划，这样做的好处是等到真正查询的时候就可以马上从数据库中获得取数据了，而不用去分析和理解这个SQL字符串语句，这个就是所谓的SQL语句编译了。有了这个SQLiteQuery对象之后，再把它和数据库对象mDatabase等待信息一起来创建一个SQLiteCursor对象，于是，这个

SQLiteCursor对象就圈定要将来要从数据库中获取的数据了。这一步执行完成之后，就把这个SQLiteCursor对象返回给上层，最终回到Step 18中的Transport类bulkQuery函数中。有了这个SQLiteCursor对象之后，就通过创建一个CursorToBulkCursorAdaptor对象来把它和匿名共享内存关联起来，这样，就为将来从数据库中查询得到的数据找到了归宿。

CursorToBulkCursorAdaptor类定义在frameworks/base/core/java/android/database/CursorToBulkCursorAdaptor.java文件中，我们来看看它的对象的构造过程，即它的构造函数的实现：

```java
public final class CursorToBulkCursorAdaptor extends BulkCursorNative  
  implements IBinder.DeathRecipient {  
    ......  
  
    public CursorToBulkCursorAdaptor(Cursor cursor, IContentObserver observer, String providerName,  
      boolean allowWrite, CursorWindow window) {  
  try {  
      mCursor = (CrossProcessCursor) cursor;  
      if (mCursor instanceof AbstractWindowedCursor) {  
          AbstractWindowedCursor windowedCursor = (AbstractWindowedCursor) cursor;  
          ......  
  
          windowedCursor.setWindow(window);  
      } else {  
          ......  
      }  
  } catch (ClassCastException e) {  
      ......  
  }  
    
  ......  
    }  
  
    ......  
}  
```
这里传进来的参数cursor的类型为SQLiteCursor，从上面的类图我们可以知道，

SQLiteCursor实现了CrossProcessCursor接口，并且继承了AbstractWindowedCursor类，因此，上面第一个if语句的条件成立，于是就会把这个SQLiteCurosr对象转换为一个AbstractWindowedCursor对象，目的是为了调用它的setWindow函数来把传进来的CursorWindow对象window保存起来，以便后面用来保存数据。

Step 23. AbstractWindowedCursor.setWindow

这个函数定义在frameworks/base/core/java/android/database/AbstractWindowedCursor.java文件中：

```java
public abstract class AbstractWindowedCursor extends AbstractCursor  
{  
    ......  
  
    public void setWindow(CursorWindow window) {  
  ......  
  
  mWindow = window;  
    }  
  
    ......  
  
    protected CursorWindow mWindow;  
}  
```
这个函数很简单，只是把参数window保存在

AbstractWindowedCursor类的成员变量mWindow中。注意，这个成员变量mWindow的访问权限为protected，即AbstractWindowedCursor的子类可以直接访问这个成员变量。

这一步完成以后，就返回到前面的Step 14中去了，执行下面语句：

```java
if (bulkCursor != null) {  
    reply.writeStrongBinder(bulkCursor.asBinder());  
  
    if (wantsCursorMetadata) {  
  reply.writeInt(bulkCursor.count());  
  reply.writeInt(BulkCursorToCursorAdaptor.findRowIdColumnIndex(  
      bulkCursor.getColumnNames()));  
    }  
} else {  
    ......  
}  
```
这里的bulkCursor不为null，于是，就会把这个

bulkCursor对象写入到数据流reply中，这个接口是要通过Binder进程间通信机制返回到第三方应用程序的，它的实际类型就是我们在前面Step 18中创建的CursorToBulkCursorAdaptor对象了。

从前面的Step 14的分析中，我们知道，这里的布尔变量wantsCursorMetadata为true，于是就会把请求数据的行数以及数据行的ID列索引号一起写入到数据流reply中去了。这里，我们重点分析IBulkCursor接口的count函数，因为这个调用使得这个Content Provider会真正去执行数据库查询的操作。至于是如何得到从数据库查询出来的数据行的ID列的位置呢？回忆前面这篇文章[Android应用程序组件Content Provider应用实例](http://blog.csdn.net/luoshengyang/article/details/6950440)，我们提到，如果我们想将数据库表中的某一列作为数据行的ID列的话，那么就必须把这个列的名称设置为"_id"，这里的BulkCursorToCursorAdaptor类的静态成员函数findRowIdColumnIndex函数就是根据这个列名"_id"来找到它是位于数据行的第几列的。

CursorToBulkCursorAdaptor类定义在frameworks/base/core/java/android/database/CursorToBulkCursorAdaptor.java文件中，它的count成员函数的实现如下所示：

```java
public final class CursorToBulkCursorAdaptor extends BulkCursorNative  
  implements IBinder.DeathRecipient {  
    ......  
  
    public int count() {  
  return mCursor.getCount();  
    }  
  
    ......  
}  
```
它的成员变量mCursor即为在前面

Step 22中创建的SQLiteCursor对象，于是，下面就会执行SQLiteCursor类的getCount成员函数。

Step 24. SQLiteCursor.getCount
这个函数定义在frameworks/base/core/java/android/database/sqlite/SQLiteCursor.java文件中：

```java
public class SQLiteCursor extends AbstractWindowedCursor {  
    ......  
  
    @Override  
    public int getCount() {  
  if (mCount == NO_COUNT) {  
      fillWindow(0);  
  }  
  return mCount;  
    }  
  
    ......  
}  
```
它里面的成员变量mCount的初始化为NO_COUNT，表示还没有去执行数据库查询操作，因此，还不知道它的值是多少，需要通过调用fillWindow函数来从数据据库中查询中，第三方应用程序所请求的数据一共有多少行。

Step 25. QLiteCursor.fillWindow
这个函数定义在frameworks/base/core/java/android/database/sqlite/SQLiteCursor.java文件中：

```java
public class SQLiteCursor extends AbstractWindowedCursor {  
    ......  
  
    private void fillWindow (int startPos) {  
  ......  
  
  mCount = mQuery.fillWindow(mWindow, mInitialRead, 0);  
    
  ......  
    }  
  
    ......  
}  
```
注意，这里的成员变量mWindow实际上是SQLiteCursor的父类

AbstractWindowedCursor的成员变量，是在Step 23中设置的，它的访问权限为protected，因此，SQLiteCursor类可以直接访问它。真正的数据库查询操作是由SQLiteCursor类的成员变量mQuery来执行的，它的类型是SQLiteCursor，是前面的Step 22中创建的，它知道如何去把第三方应用程序请求的数据从数据库中提取出来。

Step 26. SQLiteCursor.fillWindow

这个函数定义在frameworks/base/core/java/android/database/sqlite/SQLiteQuery.java文件中：

```java
public class SQLiteQuery extends SQLiteProgram {  
    ......  
  
    /* package */ int fillWindow(CursorWindow window,  
      int maxRead, int lastPos) {  
  ......  
  try {  
      ......  
      try {  
          ......  
          // if the start pos is not equal to 0, then most likely window is  
          // too small for the data set, loading by another thread  
          // is not safe in this situation. the native code will ignore maxRead  
          int numRows = native_fill_window(window, window.getStartPosition(), mOffsetIndex,  
              maxRead, lastPos);  
  
          ......  
          return numRows;  
      } catch (IllegalStateException e){  
          ......  
      } catch (SQLiteDatabaseCorruptException e) {  
          ......  
      } finally {  
          ......  
      }  
  } finally {  
      ......  
  }  
    }  
  
    ......  
}   
```
这里我们可以看到，真正的数据库查询操作是由本地方法native_fill_window来执行的，它最终也是调用了sqlite的库函数来执行数据库查询的操作，这里我们就不跟进去了，对sqlite有兴趣的读者可以自己研究一下。这个函数执行完成之后，就会把从数据库中查询得到的数据的行数返回来，这个行数最终返回到

Step 25中的SQLiteCursor.fillWindow函数，设置在SQLiteCursor类的成员变量mCount中，于是，下次再调用它的getCount函数时，就可以马上返回了。

这一步执行完成之后，就回到前面的Step 14中，最终就把从Content Provider中查询得到的数据通过匿名共享内存返回给第三方应用程序了。

至此，Android应用程序组件Content Provider在应用程序之间共享数据的原理就分析完成了，总的来说，它就是通过Binder进程间通信机制和匿名共享内存来实现的了。

关于应用程序间的数据共享还有另外的一个重要话题，就是数据更新通知机制了。因为数据是在多个应用程序中共享的，当其中一个应用程序改变了这些共享数据的时候，它有责任通知其它应用程序，让它们知道共享数据被修改了，这样它们就可以作相应的处理。在下一篇文章中，我们将分析Android应用程序组件Content Provider的数据更新通知机制，敬请关注。