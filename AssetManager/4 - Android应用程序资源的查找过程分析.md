我们知道，在[Android](http://lib.csdn.net/base/android)系统中，每一个应用程序一般都会配置很多资源，用来适配不同密度、大小和方向的屏幕，以及适配不同的国家、地区和语言等等。这些资源是在应用程序运行时自动根据设备的当前配置信息进行适配的。这也就是说，给定一个相同的资源ID，在不同的设备配置之下，查找到的可能是不同的资源。这个资源查找过程对应用程序来说，是完全透明的。在本文中，我们就详细分析资源管理框架是如何根据ID来查找资源的。

《[android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

​        从前面[Android应用程序资源管理器（Asset Manager）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8791064)一文可以知道，Android资源管理框架实际就是由AssetManager和Resources两个类来实现的。其中，Resources类可以根据ID来查找资源，而AssetManager类根据文件名来查找资源。事实上，如果一个资源ID对应的是一个文件，那么Resources类是先根据ID来找到资源文件名称，然后再将该文件名称交给AssetManager类来打开对应的文件的，这个过程如图1所示。

![img](http://img.my.csdn.net/uploads/201304/16/1366046940_6832.jpg)

图1 应用程序查找资源的过程示意图

​        在图1中，Resources类根据资源ID来查到资源名称实际上也是要通过AssetManager类来实现的，这是因为资源ID与资源名称的对应关系是由打包在APK里面的resources.arsc文件中的。当Resources类查找的资源对应的是一个文件的时候，它就会再次将资源名称交给AssetManager，以便后者可以打开对应的文件，否则的话，上一步找到的资源名称就是最终的查找结果。

​        从前面[Android应用程序资源的编译和打包过程分析](http://blog.csdn.net/luoshengyang/article/details/8744683)一文可以知道，APK包里面的resources.arsc文件是在编译应用程序资源的时候生成的，然后连同其它被编译的以及原生的资源一起打包在一个APK包里面。

​        从前面[Android资源管理框架（Asset Manager）简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/8738877)一文又可以知道，Android应用程序资源是可以划分是很多类别的，但是从资源查找的过程来看，它们可以归结为两大类。第一类资源是不对应有文件的，而第二类资源是对应有文件的，例如，字符串资源是直接编译在resources.arsc文件中的，而界面布局资源是在APK包里面是对应的单独的文件的。如上所述，不对应文件的资源只需要执行从资源ID到资源名称的转换即可，而对应有文件的资源还需要根据资源名称来打开对应的文件。在本文中，我们就以界面布局资源的查找过程为例，来说明Android资源管理框架查找资源的过程。

​       我们知道，每一个Activity组件创建的时候，它的成员函数onCreate都会被调用，而在Activity组件的成员函数onCreate中，我们基本上都无一例外地调用setContentView来设置Activity组件的界面。在调用Activity组件的成员函数setContentView的时候，需要指定一个layout类型的资源ID，以便Android资源管理框架可以找到指定的Xml资源文件来填充（inflate）为Activity组件的界面。接下来，我们就从Activity类的成员函数setContentView开始，分析Android资源管理框架查找layout资源的过程，如图2所示。

![img](http://img.my.csdn.net/uploads/201304/20/1366398825_4498.jpg)

图2 类型为layout的资源的查找过程

​        这个过程可以分为22个步骤，接下来我们就详细分析每一个步骤。

​        Step 1. Activity.setContentView

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. public class Activity extends ContextThemeWrapper  
2. ​        implements LayoutInflater.Factory,  
3. ​        Window.Callback, KeyEvent.Callback,  
4. ​        OnCreateContextMenuListener, ComponentCallbacks {  
5. ​    ......  
6.   
7. ​    private Window mWindow;  
8. ​    ......  
9.   
10. ​    public Window getWindow() {  
11. ​        return mWindow;  
12. ​    }  
13.   
14. ​    .....  
15.   
16. ​    public void setContentView(int layoutResID) {  
17. ​        getWindow().setContentView(layoutResID);  
18. ​    }  
19.   
20. ​    ......  
21. }  

​        这个函数定义在文件frameworks/base/core/[Java](http://lib.csdn.net/base/java)/android/app/Activity.java中。

​        从前面[Android应用程序窗口（Activity）的窗口对象（Window）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8223770)一文可以知道，Activity类的成员变量mWindow指向的是一个PhoneWindow对象，因此，Activity类的成员函数setContentView实际上是调用PhoneWindow类的成员函数setContentView来进一步操作。

​        Step 2. PhoneWindow.setContentView

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. public class PhoneWindow extends Window implements MenuBuilder.Callback {  
2. ​    ......  
3.   
4. ​    // This is the view in which the window contents are placed. It is either  
5. ​    // mDecor itself, or a child of mDecor where the contents go.  
6. ​    private ViewGroup mContentParent;  
7. ​    ......  
8.   
9. ​    private LayoutInflater mLayoutInflater;  
10. ​    ......  
11.   
12. ​    @Override  
13. ​    public void setContentView(int layoutResID) {  
14. ​        if (mContentParent == null) {  
15. ​            installDecor();  
16. ​        } else {  
17. ​            mContentParent.removeAllViews();  
18. ​        }  
19. ​        mLayoutInflater.inflate(layoutResID, mContentParent);  
20. ​        final Callback cb = getCallback();  
21. ​        if (cb != null) {  
22. ​            cb.onContentChanged();  
23. ​        }  
24. ​    }  
25.   
26. ​    ......  
27. }  

​        这个函数定义在文件frameworks/base/policy/src/com/android/internal/policy/impl/PhoneWindow.java中。

​        PhoneWindow类的成员变量mContentParent用来描述一个类型为DecorView的视图对象，或者这个类型为DecorView的视图对象的一个子视图对象，用作UI容器。当它的值等于null的时候，就说明当前正在处理的Activity组件的视图对象还没有创建。在这种情况下，就会调用成员函数installDecor来创建当前正在处理的Activity组件的视图对象。否则的话，就说明是要重新设置当前正在处理的Activity组件的视图。在重新设置之前，首先调用成员变量mContentParent所描述的一个ViewGroup对象来移除原来的UI内容。

​        PhoneWindow类的成员变量mLayoutInflater指向的是一个PhoneLayoutInflater对象。PhoneLayoutInflater类是从LayoutInflater类继续下来的，同时它也继承了LayoutInflater类的成员函数inflate。通过调用PhoneWindow类的成员变量mLayoutInflater所指向的一个PhoneLayoutInflater对象的成员函数inflate，也就是从父类继承下来的成员函数inflate，就可以将参数layoutResID所描述的一个UI布局设置到mContentParent所描述的一个视图容器中去。这样就可以将当前正在处理的Activity组件的UI创建出来。

​        最后，PhoneWindow类的成员函数还会调用一个Callback接口的成员函数onContentChanged来通知当前正在处理的Activity组件，它的视图内容发生改变了。从前面[Android应用程序窗口（Activity）的窗口对象（Window）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8223770)一文可以知道，每一个Activity组件都实现了一个Callback接口，并且将这个Callback接口设置到了与它所关联的PhoneWindow的内部去，因此，最后调用的实际上是Activity类的成员函数onContentChanged。

​        接下来，我们就继续分析LayoutInflater类的成员函数inflate的实现，以便可以了解Android资源管理框架是如何找到参数layoutResID所描述的UI布局文件的。

​        Step 3. LayoutInflater.inflate

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. public abstract class LayoutInflater {  
2. ​    ......  
3.   
4. ​    public View inflate(int resource, ViewGroup root) {  
5. ​        return inflate(resource, root, root != null);  
6. ​    }  
7.   
8. ​    ......  
9.   
10. ​    public View inflate(int resource, ViewGroup root, boolean attachToRoot) {  
11. ​        ......  
12. ​        XmlResourceParser parser = getContext().getResources().getLayout(resource);  
13. ​        try {  
14. ​            return inflate(parser, root, attachToRoot);  
15. ​        } finally {  
16. ​            parser.close();  
17. ​        }  
18. ​    }  
19.   
20. ​    ......  
21. }  

​        这个函数定义在文件frameworks/base/core/java/android/view/LayoutInflater.java中。

​        LayoutInflater类两个参数版本的成员函数inflate通过调用三个参数版本的成员函数inflate来查找参数resource所描述的UI布局文件。

​        在LayoutInflater类三个参数版本的成员函数inflate中，首先是获得用来描述当前运行上下文环境的一个Resources对象，然后接调用这个Resources对象的成员函数getLayout来查找参数resource所描述的UI布局文件。

​        Resources类的成员函数getLayout找到了指定的UI布局文件之后，就会打开它。由于Android系统的UI布局文件是一个Xml文件，因此，Resources类的成员函数getLayout打开它之后，得到的是一个XmlResourceParser对象。有了这个XmlResourceParser对象之后，LayoutInflater类三个参数版本的成员函数inflate就将它传递给另外一个三个参数版本的成员函数inflate，以便后者可以通过它来创建一个UI界面。

​        接下来，我们就首先分析Resources类的成员函数getLayout的实现，然后再分析LayoutInflater类的另外一个三个参数版本的成员函数inflate的实现。

​        Step 4. Resources.getLayout

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. public class Resources {  
2. ​    ......  
3.   
4. ​    public XmlResourceParser getLayout(int id) throws NotFoundException {  
5. ​        return loadXmlResourceParser(id, "layout");  
6. ​    }  
7.   
8. ​    ......  
9. }  

​        这个函数定义在文件frameworks/base/core/java/android/content/res/Resources.java中。

​        Resources类的成员函数getLayout的实现很简单，它通过调用另外一个成员函数loadXmlResourceParser来查找并且打开由参数id所描述的一个UI布局文件。

​        Step 5. Resources.loadXmlResourceParser

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. public class Resources {  
2. ​    ......  
3.   
4. ​    /*package*/ XmlResourceParser loadXmlResourceParser(int id, String type)  
5. ​            throws NotFoundException {  
6. ​        synchronized (mTmpValue) {  
7. ​            TypedValue value = mTmpValue;  
8. ​            getValue(id, value, true);  
9. ​            if (value.type == TypedValue.TYPE_STRING) {  
10. ​                return loadXmlResourceParser(value.string.toString(), id,  
11. ​                        value.assetCookie, type);  
12. ​            }  
13. ​            throw new NotFoundException(  
14. ​                    "Resource ID #0x" + Integer.toHexString(id) + " type #0x"  
15. ​                    + Integer.toHexString(value.type) + " is not valid");  
16. ​        }  
17. ​    }  
18.   
19. ​    ......  
20. }  

​        这个函数定义在文件frameworks/base/core/java/android/content/res/Resources.java中。

​        参数id描述的是一个资源ID，Resources类的成员函数loadXmlResourceParser首先调用另外一个成员函数getValue来获得该资源ID所对应的资源值，并且保存在一个类型为TypedValue的变量value中。在我们这个情景中，参数id描述的是一个类型为layout的资源ID，从前面[Android应用程序资源的编译和打包过程分析](http://blog.csdn.net/luoshengyang/article/details/8744683)一文可以知道，类型为layout的资源ID对应的资源值即为一个UI布局文件名称。有了这个UI布局文件名称之后，Resources类的成员函数loadXmlResourceParser接着再调用另外一个四个参数版本的成员函数loadXmlResourceParser来加载对应的UI布局文件，并且得到一个XmlResourceParser对象返回给调用者。

​        注意，如果Resources类的成员函数getValue没有找到与参数id所描述的资源，或者找到的资源的值不是字符串类型的，那么Resources类的成员函数loadXmlResourceParser就会抛出一个类型为NotFoundException的异常。

​        接下来，我们就首先分析Resources类的成员函数getValue的实现，接着再分析Resources类四个参数版本的成员函数loadXmlResourceParser的实现。

​        Step 6. Resources.getValue

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. public class Resources {  
2. ​    ......  
3.   
4. ​    /*package*/ final AssetManager mAssets;  
5. ​    ......  
6.   
7. ​    public void getValue(int id, TypedValue outValue, boolean resolveRefs)  
8. ​            throws NotFoundException {  
9. ​        boolean found = mAssets.getResourceValue(id, outValue, resolveRefs);  
10. ​        if (found) {  
11. ​            return;  
12. ​        }  
13. ​        throw new NotFoundException("Resource ID #0x"  
14. ​                                    + Integer.toHexString(id));  
15. ​    }  
16.   
17. ​    ......  
18. }  

​        这个函数定义在文件frameworks/base/core/java/android/content/res/Resources.java中。

​        Resources类的成员变量mAssets指向的是一个AssetManager对象，Resources类的成员函数getValue通过调用它的成员函数getResourceValue来获得与参数id所对应的资源的值。注意，如果AssetManager类的成员函数getResourceValue查找不到与参数id所对应的资源，那么Resources类的成员函数getValue就会抛出一个类型为NotFoundException的异常。

​        接下来，我们就继续分析AssetManager类的成员函数getResourceValue的实现。

​        Step 7. AssetManager.getResourceValue

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. public final class AssetManager {  
2. ​    ......  
3.   
4. ​    private StringBlock mStringBlocks[] = null;  
5. ​    ......  
6.   
7. ​    /*package*/ final boolean getResourceValue(int ident,  
8. ​                                               TypedValue outValue,  
9. ​                                               boolean resolveRefs)  
10. ​    {  
11. ​        int block = loadResourceValue(ident, outValue, resolveRefs);  
12. ​        if (block >= 0) {  
13. ​            if (outValue.type != TypedValue.TYPE_STRING) {  
14. ​                return true;  
15. ​            }  
16. ​            outValue.string = mStringBlocks[block].get(outValue.data);  
17. ​            return true;  
18. ​        }  
19. ​        return false;  
20. ​    }  
21.   
22. ​    ......  
23. }  

​        这个函数定义在文件frameworks/base/core/java/android/content/res/AssetManager.java中。

​        AssetManager类的成员函数getResourceValue通过调用另外一个成员函数loadResourceValue来加载参数ident所描述的资源。如果加载成功，那么结果就会保存在参数outValue所描述的一个TypedValue对象中，并且AssetManager类的成员函数loadResourceValue的返回值block大于等于0。

​        从前面[Android应用程序资源管理器（Asset Manager）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8791064)一文可以知道，AssetManager类的成员变量mStringBlock描述的是一个StringBlock数组。这个StringBlock数组中的每一个StringBlock对象描述的都是当前应用程序使用的每一个资源索引表的资源项值字符串资源池。关于资源索引表的格式以及生成过程，可以参考前面[Android应用程序资源的编译和打包过程分析](http://blog.csdn.net/luoshengyang/article/details/8744683)一文。

​        了解了上述背景之后，我们就可以知道，当AssetManager类的成员函数loadResourceValue的返回值block大于等于0的时候，实际上就表示参数ident所描述的资源项在当前应用程序使用的第block个资源索引表中，而当参数ident所描述的资源项是一个字符串时，那么就可以在第block个资源索引表的资源项值字符串资源池中找到对应的字符串，并且保存在参数outValue所描述的一个TypedValue对象的成员变量string中，以便返回给调用者使用。注意，最终得到的字符串在第block个资源索引表的资源项值字符串资源池中的位置就保存在参数outValue所描述的一个TypedValue对象的成员变量data中。

​        接下来，我们就继续分析AssetManager类的成员函数loadResourceValue的实现。

​        Step 8. AssetManager.loadResourceValue

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. public final class AssetManager {  
2. ​    ......  
3.   
4. ​    /** Returns true if the resource was found, filling in mRetStringBlock and 
5. ​     *  mRetData. */  
6. ​    private native final int loadResourceValue(int ident, TypedValue outValue,  
7. ​                                               boolean resolve);  
8.   
9. ​    ......  
10. }  

​        这个函数定义在文件frameworks/base/core/java/android/content/res/AssetManager.java中。

​        AssetManager类的成员函数loadResourceValue是一个JNI方法，它是由C++层的函数android_content_AssetManager_loadResourceValue来实现的，如下所示：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. static jint android_content_AssetManager_loadResourceValue(JNIEnv* env, jobject clazz,  
2. ​                                                           jint ident,  
3. ​                                                           jobject outValue,  
4. ​                                                           jboolean resolve)  
5. {  
6. ​    AssetManager* am = assetManagerForJavaObject(env, clazz);  
7. ​    if (am == NULL) {  
8. ​        return 0;  
9. ​    }  
10. ​    const ResTable& res(am->getResources());  
11.   
12. ​    Res_value value;  
13. ​    ResTable_config config;  
14. ​    uint32_t typeSpecFlags;  
15. ​    ssize_t block = res.getResource(ident, &value, false, &typeSpecFlags, &config);  
16. ​    ......  
17.   
18. ​    uint32_t ref = ident;  
19. ​    if (resolve) {  
20. ​        block = res.resolveReference(&value, block, &ref);  
21. ​        ......  
22. ​    }  
23. ​    return block >= 0 ? copyValue(env, outValue, &res, value, ref, block, typeSpecFlags, &config) : block;  
24. }  

​        这个函数定义在文件frameworks/base/core/jni/android_util_AssetManager.cpp中。

​        函数android_content_AssetManager_loadResourceValue主要是执行以下五个操作：

​        1. 调用函数assetManagerForJavaObject来将参数clazz所描述的一个Java层的AssetManager对象的成员变量mObject转换为一个C++层的AssetManager对象。

​        2. 调用上述得到的C++层的AssetManager对象的成员函数getResources来获得一个ResTable对象，这个ResTable对象描述的是一个资源表。

​        3. 调用上述得到的ResTable对象的成员函数getResource来获得与参数ident所对应的资源项值及其配置信息，并且保存在类型为Res_value的变量value以及类型为ResTable_config的变量config中。

​        4. 如果参数resolve的值等于true，那么就继续调用上述得到的ResTable对象的成员函数resolveReference来解析前面所得到的资源项值。

​        5. 调用函数copyValue将上述得到的资源项值及其配置信息拷贝到参数outValue所描述的一个Java层的TypedValue对象中去，返回调用者可以获得与参数ident所对应的资源项内容。

​        接下来，我们就主要分析第2~4操作，即AssetManager对象的成员函数getResources以及ResTable类的成员函数getResource和resolveReference的实现，以便可以了解Android应用程序资源的查找过程。

​        Step 9. AssetManager.getResources

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. const ResTable& AssetManager::getResources(bool required) const  
2. {  
3. ​    const ResTable* rt = getResTable(required);  
4. ​    return *rt;  
5. }  

​        这个函数定义在文件frameworks/base/libs/utils/AssetManager.cpp中。

​        AssetManager类的成员函数getResources通过调用另外一个成员函数getResTable来获得当前应用程序所使用的资源表，后者的实现如下所示：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. const ResTable* AssetManager::getResTable(bool required) const  
2. {  
3. ​    ResTable* rt = mResources;  
4. ​    if (rt) {  
5. ​        return rt;  
6. ​    }  
7.   
8. ​    // Iterate through all asset packages, collecting resources from each.  
9.   
10. ​    AutoMutex _l(mLock);  
11.   
12. ​    if (mResources != NULL) {  
13. ​        return mResources;  
14. ​    }  
15.   
16. ​    ......  
17.   
18. ​    const size_t N = mAssetPaths.size();  
19. ​    for (size_t i=0; i<N; i++) {  
20. ​        Asset* ass = NULL;  
21. ​        ResTable* sharedRes = NULL;  
22. ​        bool shared = true;  
23. ​        const asset_path& ap = mAssetPaths.itemAt(i);  
24. ​        Asset* idmap = openIdmapLocked(ap);  
25. ​        ......  
26. ​        if (ap.type != kFileTypeDirectory) {  
27. ​            if (i == 0) {  
28. ​                // The first item is typically the framework resources,  
29. ​                // which we want to avoid parsing every time.  
30. ​                sharedRes = const_cast<AssetManager*>(this)->  
31. ​                    mZipSet.getZipResourceTable(ap.path);  
32. ​            }  
33. ​            if (sharedRes == NULL) {  
34. ​                ass = const_cast<AssetManager*>(this)->  
35. ​                    mZipSet.getZipResourceTableAsset(ap.path);  
36. ​                if (ass == NULL) {  
37. ​                    ......  
38. ​                    ass = const_cast<AssetManager*>(this)->  
39. ​                        openNonAssetInPathLocked("resources.arsc",  
40. ​                                                 Asset::ACCESS_BUFFER,  
41. ​                                                 ap);  
42. ​                    if (ass != NULL && ass != kExcludedAsset) {  
43. ​                        ass = const_cast<AssetManager*>(this)->  
44. ​                            mZipSet.setZipResourceTableAsset(ap.path, ass);  
45. ​                    }  
46. ​                }  
47.   
48. ​                if (i == 0 && ass != NULL) {  
49. ​                    // If this is the first resource table in the asset  
50. ​                    // manager, then we are going to cache it so that we  
51. ​                    // can quickly copy it out for others.  
52. ​                    LOGV("Creating shared resources for %s", ap.path.string());  
53. ​                    sharedRes = new ResTable();  
54. ​                    sharedRes->add(ass, (void*)(i+1), false, idmap);  
55. ​                    sharedRes = const_cast<AssetManager*>(this)->  
56. ​                        mZipSet.setZipResourceTable(ap.path, sharedRes);  
57. ​                }  
58. ​            }  
59. ​        } else {  
60. ​            ......  
61. ​            Asset* ass = const_cast<AssetManager*>(this)->  
62. ​                openNonAssetInPathLocked("resources.arsc",  
63. ​                                         Asset::ACCESS_BUFFER,  
64. ​                                         ap);  
65. ​            shared = false;  
66. ​        }  
67. ​        if ((ass != NULL || sharedRes != NULL) && ass != kExcludedAsset) {  
68. ​            if (rt == NULL) {  
69. ​                mResources = rt = new ResTable();  
70. ​                updateResourceParamsLocked();  
71. ​            }  
72. ​            ......  
73. ​            if (sharedRes != NULL) {  
74. ​                ......  
75. ​                rt->add(sharedRes);  
76. ​            } else {  
77. ​                ......  
78. ​                rt->add(ass, (void*)(i+1), !shared, idmap);  
79. ​            }  
80.   
81. ​            if (!shared) {  
82. ​                delete ass;  
83. ​            }  
84. ​        }  
85. ​        if (idmap != NULL) {  
86. ​            delete idmap;  
87. ​        }  
88. ​    }  
89.   
90. ​    ......  
91. ​    if (!rt) {  
92. ​        mResources = rt = new ResTable();  
93. ​    }  
94. ​    return rt;  
95. }  

​        这个函数定义在文件frameworks/base/libs/utils/AssetManager.cpp中。

​        AssetManager类的成员函数getResources的实现看起来比较复杂，但是它要做的事情就是解析当前应用程序所使用的资源包里面的resources.arsc文件。从前面[Android应用程序资源管理器（Asset Manager）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8791064)一文可以知道，当前应用程序所使用的资源包有两个，其中一个是系统资源包，即/system/framework/framework-res.apk，另外一个就是自己的APK文件。这些APK文件的路径都分别使用一个asset_path对象来描述，并且保存在AssetManager类的成员变量mAssetPaths中。

​        AssetManager类的成员变量mResources指向的是一个ResTable对象，如果它的值不等于NULL，那么就说明当前应用程序已经解析过它使用的资源包里面的resources.arsc文件，因此，这时候AssetManager类的成员函数getResources就可以直接将该ResTable对象返回给调用者。

​        如果当前应用程序还没有解析过它使用的资源包里面的resources.arsc文件，那么AssetManager类的成员函数getResources就会先获取由成员变量mLock所描述的一个互斥锁，避免多个线程同时去解析当前应用程序还没有解析过它使用的资源包里面的resources.arsc文件。注意，获取锁成功之后，有可能其它线程已经抢先一步解析了当前应用程序使用的资源包里面的resources.arsc文件了，因此，这时候就需要再次判断 AssetManager类的成员函数mResources是否等于NULL。如果不等于NULL，就可以将它所指向的ResTable对象返回给调用者了。

​        AssetManager类的成员函数getResources接下来按照以下步骤来解析当前应用程序所使用的每一个资源包里面的resources.arsc文件：

​        1. 检查资源包里面的resources.arsc文件已经提取出来。如果已经提取出来的话，那么以当前正在处理的资源包路径为参数，调用当前正在处理的AssetManager对象的成员变量mZipSet所指向的一个ZipSet对象的成员函数getZipResourceTableAsset就可以获得一个对应的Asset对象。

​        2. 如果资源包里面的resources.arsc文件还没有提取出来，那么就会调用当前正在处理的AssetManager对象的成员函数openNonAssetInPathLocked来将该resources.arsc文件提取出来。提取的结果就是获得一个对应的Asset对象，保存在变量ass中。注意，如果当前提取出来的Asset对象的地址值不等于全局变量kExcludedAsset的值，那么就将该Asset对象设置为当前正在处理的AssetManager对象的成员变量mZipSet所指向的一个ZipSet对象中去，这是通过调用该ZipSet对象的成员函数setZipResourceTableAsset来实现的。

​        3. 将上面获得的用来描述resources.arsc文件的Asset对象ass添加到变量rt所描述的一个ResTable对象中去，这是通过调用该ResTable对象的成员函数add来实现的。注意，如果该ResTable对象还没有创建，那么它就会首先被创建，并且同时保存在AssetManager类的成员变量mResources和变量rt中。另外一个地方需要注意的是，ResTable类的成员函数add在增加一个Asset对象时，会对该Asset对象所描述的resources.arsc文件的内容进行解析，结果就是得到一个系列的Package信息。每一个Package又包含了一个资源类型字符串资源池和一个资源项名称字符串资源池，以及一系列的资源类型规范数据块和一系列的资源项数据块。这些内容可以参考前面[Android应用程序资源的编译和打包过程分析](http://blog.csdn.net/luoshengyang/article/details/8744683)一文。还有第三个地方需要注意的是，每一个资源包里面的所有Pacakge形成一个PackageGroup，保存在变量rt所指向的一个ResTable对象的成员变量mPackageGroups中。总之，ResTable类的作用就类似于在前面[Android应用程序资源的编译和打包过程分析](http://blog.csdn.net/luoshengyang/article/details/8744683)一文所介绍的ResourceTable类。

​        一般来说，在AssetManager类的成员变量mAssetPaths中，第一个资源包路径指向的就是系统资源包，而系统资源包在当前正在处理的AssetManager对象创建的时候，可能就已经提取或者初始化过了，也就是它的resources.arsc文件已经提取或者解析过了。如果已经解析过，那么在代码中的for循环中，当i等于0时，调用当前正在处理的AssetManager对象的成员变量mZipSet所指向的一个ZipSet对象的成员函数getZipResourceTable就可以获得一个ResTable对象，并且保存在变量sharedRes中，最终就可以直接将该ResTable对象添加到变量rt所指向的一个ResTable对象中去，也就是添加到AssetManager类的成员变量mResources所指向的一个ResTable对象中去。如果只是提取过，但是还没有解析过，那么就会首先对它进行解析，并且将得到的ResTable对象保存在变量sharedRes中，最后再将该ResTable对象添加到变量rt所指向的一个ResTable对象中去。

​         此外，AssetManager类的成员变量mAssetPaths保存的资源包路径指向可能不是一个APK文件，而是一个目录文件。在这种情况下，AssetManager类的成员函数getResources就会直接调用另外一个成员函数openNonAssetInPathLocked来打开该目录下的resources.arsc文件，并且获得一个Asset对象，同样是保存在变量ass中。

​        在前面[Android应用程序资源管理器（Asset Manager）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8791064)一文还提到，Android系统的资源管理框架提供了一种idmap机制，用来个性化定制一个资源包里面已有的资源项，也就是说，每一个资源包都可能有一个对应的idmap文件，用来描述它所个性化定制的资源项。在提取和解析资源包的过程中，如果该资源包存在idmap文件，那么该idmap文件也会被解析，并且解析得到的一个Asset对象也会同时被增加到变量rt所指向的一个ResTable对象中去。

​        经过上述的一系列操作之后，AssetManager类的成员变量mResources所指向的一个ResTable对象就包含了当前应用程序所使用的资源包的所有信息，该ResTable对象最后就会返回给调用者来使用。

​        这一步执行完成之后，回到前面的Step 8中，即AssetManager类的成员函数loadResourceValue中，接下来它就会调用前面所获得的一个ResTable对象的成员函数getResource来获得指定的资源项内容。

​        Step 10. ResTable.getResource

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. ssize_t ResTable::getResource(uint32_t resID, Res_value* outValue, bool mayBeBag,  
2. ​        uint32_t* outSpecFlags, ResTable_config* outConfig) const  
3. {  
4. ​    ......  
5.   
6. ​    const ssize_t p = getResourcePackageIndex(resID);  
7. ​    const int t = Res_GETTYPE(resID);  
8. ​    const int e = Res_GETENTRY(resID);  
9.   
10. ​    ......  
11.   
12. ​    const Res_value* bestValue = NULL;  
13. ​    const Package* bestPackage = NULL;  
14. ​    ResTable_config bestItem;  
15. ​    memset(&bestItem, 0, sizeof(bestItem)); // make the compiler shut up  
16.   
17. ​    if (outSpecFlags != NULL) *outSpecFlags = 0;  
18.   
19. ​    // Look through all resource packages, starting with the most  
20. ​    // recently added.  
21. ​    const PackageGroup* const grp = mPackageGroups[p];  
22. ​    ......  
23.   
24. ​    size_t ip = grp->packages.size();  
25. ​    while (ip > 0) {  
26. ​        ip--;  
27. ​        int T = t;  
28. ​        int E = e;  
29.   
30. ​        const Package* const package = grp->packages[ip];  
31. ​        if (package->header->resourceIDMap) {  
32. ​            uint32_t overlayResID = 0x0;  
33. ​            status_t retval = idmapLookup(package->header->resourceIDMap,  
34. ​                                          package->header->resourceIDMapSize,  
35. ​                                          resID, &overlayResID);  
36. ​            if (retval == NO_ERROR && overlayResID != 0x0) {  
37. ​                // for this loop iteration, this is the type and entry we really want  
38. ​                ......  
39. ​                T = Res_GETTYPE(overlayResID);  
40. ​                E = Res_GETENTRY(overlayResID);  
41. ​            } else {  
42. ​                // resource not present in overlay package, continue with the next package  
43. ​                continue;  
44. ​            }  
45. ​        }  
46.   
47. ​        const ResTable_type* type;  
48. ​        const ResTable_entry* entry;  
49. ​        const Type* typeClass;  
50. ​        ssize_t offset = getEntry(package, T, E, &mParams, &type, &entry, &typeClass);  
51. ​        if (offset <= 0) {  
52. ​            // No {entry, appropriate config} pair found in package. If this  
53. ​            // package is an overlay package (ip != 0), this simply means the  
54. ​            // overlay package did not specify a default.  
55. ​            // Non-overlay packages are still required to provide a default.  
56. ​            if (offset < 0 && ip == 0) {  
57. ​                ......  
58. ​                return offset;  
59. ​            }  
60. ​            continue;  
61. ​        }  
62.   
63. ​        if ((dtohs(entry->flags)&entry->FLAG_COMPLEX) != 0) {  
64. ​            ......  
65. ​            continue;  
66. ​        }  
67.   
68. ​        ......  
69.   
70. ​        const Res_value* item =  
71. ​            (const Res_value*)(((const uint8_t*)type) + offset);  
72. ​        ResTable_config thisConfig;  
73. ​        thisConfig.copyFromDtoH(type->config);  
74.   
75. ​        if (outSpecFlags != NULL) {  
76. ​            if (typeClass->typeSpecFlags != NULL) {  
77. ​                *outSpecFlags |= dtohl(typeClass->typeSpecFlags[E]);  
78. ​            } else {  
79. ​                *outSpecFlags = -1;  
80. ​            }  
81. ​        }  
82.   
83. ​        if (bestPackage != NULL &&  
84. ​            (bestItem.isMoreSpecificThan(thisConfig) || bestItem.diff(thisConfig) == 0)) {  
85. ​            // Discard thisConfig not only if bestItem is more specific, but also if the two configs  
86. ​            // are identical (diff == 0), or overlay packages will not take effect.  
87. ​            continue;  
88. ​        }  
89.   
90. ​        bestItem = thisConfig;  
91. ​        bestValue = item;  
92. ​        bestPackage = package;  
93. ​    }  
94.   
95. ​    ......  
96.   
97. ​    if (bestValue) {  
98. ​        outValue->size = dtohs(bestValue->size);  
99. ​        outValue->res0 = bestValue->res0;  
100. ​        outValue->dataType = bestValue->dataType;  
101. ​        outValue->data = dtohl(bestValue->data);  
102. ​        if (outConfig != NULL) {  
103. ​            *outConfig = bestItem;  
104. ​        }  
105. ​        ......  
106. ​        return bestPackage->header->index;  
107. ​    }  
108.   
109. ​    return BAD_VALUE;  
110. }  

​        这个函数定义在文件frameworks/base/libs/utils/ResourceTypes.cpp中。

​        参数resID描述的是要查找的资源的ID，ResTable类的成员函数getResource分别获得它的Pakcage ID、Type ID以及Entry ID，保存在变量p、t以及e中。知道了Pakcage ID之后，就可以在ResTable类的成员变量mPackageGroups中找到对应的PakcageGroup。

​        注意，前面获得的PakcageGroup可能包含有多个Package，这些Package都保存在PakcageGroup的成员变量packages所描述的一个数组中，因此，ResTable类的成员函数getResource就通过一个while循环来在每一个Package中查找最符合条件的资源项。

​        如果当前正在处理的Package的成员变量header所描述的一个Header对象的成员变量resourceIDMap的值不等于NULL，那么它所指向的就是一个idmap，同时也说明当前正在处理的Package是一个Overlay Package，也就是说，它是用来覆盖已存在的资源项的。在这种情况下，ResTable类的成员函数getResource就会将参数resID所描述的资源ID映射为覆盖后的资源ID。注意，如果不能将参数resID所描述的资源ID映射为覆盖后的ID，那么当前正在处理的Package就会被跳过。

​        无论当前正在处理的Package是否是一个Overlay Package，最要要找到的资源项的Type ID和Entry ID都保存在变量T和E中，接下来ResTable类的成员函数getResource就会以这两个变量为参数来调用另外一个成员函数getEntry来在当前正在处理的Package中检查是否存在符合条件的资源项。如果存在的话，那么调用ResTable类的成员函数getEntry得到的返回值offset就会大于0，同时还会得到三个类型分别为ResTable_type、ResTable_entry和Type结构体，分别保存在变量type、entry和typeClass中。其中，ResTable_type用来描述一个资源类型，ResTable_entry用来描述一个资源项，而Type用来描述一个资源类型规范，关于这些结构的详细解释可以参考前面[Android应用程序资源的编译和打包过程分析](http://blog.csdn.net/luoshengyang/article/details/8744683)一文。

​        从前面[Android应用程序资源的编译和打包过程分析](http://blog.csdn.net/luoshengyang/article/details/8744683)一文还可以知道，对于一个普通的资源项来说，它在资源表文件resources.arsc中，由一个ResTable_entry和一个Res_value结构体前面连接在一起表示，其中，结构体ResTable_entry用来描述资源项的头部，而结构体Res_value用来描述资源项的值，并且这两个结构体是嵌入在一个ResTable_type结构体里面的。

​        理解了上述背景知道之后，我们就可以解释前面得到的返回值offset的含义了，它表示一个Res_value结构体在一个ResTable_type结构体中的偏移，也就是说，将前面获得的ResTable_type结构体type的开始地址，再加偏移量offset，就可以得到一个Res_value结构体item，而这个Res_value结构体就是表示资源ID等于resID的资源项的值。

​        注意，在调用ResTable类的成员函数getEntry来在当前正在处理的Package中查找与参数resID对应的资源项时，还会指定设备的当前配置信息。设备的当前配置信息是由ResTable类的成员变量mParams所指向的一个ResTable_config结构体来描述的。如果ResTable类的成员函数getEntry能成功找到一个匹配的资源项，那么它还会通过ResTable_type结构体type的成员变量config所指向的一个ResTable_config结构体来返回该资源项的实际配置信息，并且保存在另外一个ResTable_config结构体thisConfig中。

​        现在一切就准备就绪了，ResTable类的成员函数getResource要做的事情就是比较在前后两个Package中找到的两个资源项中，哪一个资源项更匹配设备的当前配置信息。注意，在前一个Package中找到的资源项的值及其所对应的Package和配置信息分别保存在Res_value结构体bestValue、Package结构体bestPackage和ResTable_config结构体bestItem中，而在后一个Package中找到的资源项对应的配置信息保存在ResTable_config结构体thisConfig中。

​        如果ResTable_config结构体bestItem描述的配置信息比ResTable_config结构体thisConfig描述的配置信息更具体，或者它们完全是一样的，那么ResTable类的成员函数getResource就会认为在前一个Package中找到的资源项更匹配设备的当前配置信息，于是就保持Res_value结构体bestValue、Package结构体bestPackage和ResTable_config结构体bestItem的值不变，否则的话，就会将在后一个Package中找到的资源项的值及其所对应的Package和配置信息分别保存在Res_value结构体bestValue、Package结构体bestPackage和ResTable_config结构体bestItem中。

​        ResTable类的成员函数getResource执行完成中间的while循环之后，最终得到的资源项的值以及配置信息就保存在Res_value结构体bestValue和ResTable_config结构体bestItem，最后就可以分别将它们的内容拷贝到输出参数outValue和outConfig中去，以便可以返回给调用者。同时，ResTable类的成员函数getResource还会将最终得到的资源项所在的Package的索引返回给调用者。通过这个Package的索引值，调用者就可以知道它所找到的资源项是在哪一个资源包中找到的，例如，是在系统资源包找到的，还是在应用程序本身的资源包找到的。

​        事实上，ResTable类的成员函数getResource返回给调用者的还有一个很重要的信息，那就是参数resID所描述的资源项的配置状况，也就是说，参数resID所描述的资源项的配置差异性信息。这个配置差异性信息就保存在前面得到的Type结构体typeClass的成员变量typeSpecFlags所描述的一个uint32_t数组中的第E个元素中，这是因为参数resID所描述的资源项的Entry ID等于E。关于资源项的配置差异性信息的详细描述，可以参考前面[Android应用程序资源的编译和打包过程分析](http://blog.csdn.net/luoshengyang/article/details/8744683)一文所提到的ResTable_typeSpec结构体。

​        在ResTable类的成员函数getResource查找资源的过程中，还有两个地方是需要注意的。

​        第一个地方是ResTable类的成员函数getResource是从后往前遍历Pakcage ID等于p的PackageGroup中的每一个Package的。在一个PackageGroup中，第一个Package是一个Base Package，其它的Package都是属于Overlay Package，其中，在Overlay Package中定义的资源是用来覆盖在Base Package中定义的资源的。如果ResTable类的成员函数getResource在调用另外一个成员函数getEntry来在某一个package中找不到对应的资源项时，即调用ResTable类的成员函数getEntry得到的返回值offset小于等于0的时候，需要进一步检查该package是一个Base Package还是一个Overlay Package。如果是一个Overlay Package，那么这种情况是允许发生的，因为一个Overlay Package只需要定义它需要覆盖的资源。另一方面，如果是一个Base Package，那么就是这种情况就是异常的，因为一个Base Package无论如何都要保证在给定一个合法的资源ID的前提下，一定可以找到一个对应的资源项，在最坏的情况下，这个资源项就是一个default类型的。

​        第二个地方是只有当调用ResTable类的成员函数getEntry得到资源项是一个普通资源项时，即得到的ResTable_entry结构体entry的成员变量flags的值的FLAG_COMPLEX位等于0时，才可以将得到的ResTable_type结构体type的偏移位置offset转换为一个Res_value结构体来访问。这是因为如果找到的资源项不是一个普通资源项，而是一个Bag资源项时，得到的ResTable_type结构体type的偏移位置offset是一个ResTable_map结构体数组，而不是一个Res_value结构体，这一点可以参考前面[Android应用程序资源的编译和打包过程分析](http://blog.csdn.net/luoshengyang/article/details/8744683)一文。

​        接下来，我们就继续分析ResTable类的成员函数getEntry的实现，以便可以了解它是如何在一个Package中找到指定Type ID、Entry ID以及配置信息的资源项的，如下所示：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. ssize_t ResTable::getEntry(  
2. ​    const Package* package, int typeIndex, int entryIndex,  
3. ​    const ResTable_config* config,  
4. ​    const ResTable_type** outType, const ResTable_entry** outEntry,  
5. ​    const Type** outTypeClass) const  
6. {  
7. ​    ......  
8. ​    const ResTable_package* const pkg = package->package;  
9.   
10. ​    const Type* allTypes = package->getType(typeIndex);  
11. ​    ......  
12.   
13. ​    const ResTable_type* type = NULL;  
14. ​    uint32_t offset = ResTable_type::NO_ENTRY;  
15. ​    ResTable_config bestConfig;  
16. ​    memset(&bestConfig, 0, sizeof(bestConfig)); // make the compiler shut up  
17.   
18. ​    const size_t NT = allTypes->configs.size();  
19. ​    for (size_t i=0; i<NT; i++) {  
20. ​        const ResTable_type* const thisType = allTypes->configs[i];  
21. ​        if (thisType == NULL) continue;  
22.   
23. ​        ResTable_config thisConfig;  
24. ​        thisConfig.copyFromDtoH(thisType->config);  
25. ​        ......  
26.   
27. ​        // Check to make sure this one is valid for the current parameters.  
28. ​        if (config && !thisConfig.match(*config)) {  
29. ​            ......  
30. ​            continue;  
31. ​        }  
32.   
33. ​        // Check if there is the desired entry in this type.  
34.   
35. ​        const uint8_t* const end = ((const uint8_t*)thisType)  
36. ​            + dtohl(thisType->header.size);  
37. ​        const uint32_t* const eindex = (const uint32_t*)  
38. ​            (((const uint8_t*)thisType) + dtohs(thisType->header.headerSize));  
39.   
40. ​        uint32_t thisOffset = dtohl(eindex[entryIndex]);  
41. ​        if (thisOffset == ResTable_type::NO_ENTRY) {  
42. ​            ......  
43. ​            continue;  
44. ​        }  
45.   
46. ​        if (type != NULL) {  
47. ​            // Check if this one is less specific than the last found.  If so,  
48. ​            // we will skip it.  We check starting with things we most care  
49. ​            // about to those we least care about.  
50. ​            if (!thisConfig.isBetterThan(bestConfig, config)) {  
51. ​                ......  
52. ​                continue;  
53. ​            }  
54. ​        }  
55.   
56. ​        type = thisType;  
57. ​        offset = thisOffset;  
58. ​        bestConfig = thisConfig;  
59. ​        ......  
60. ​        if (!config) break;  
61. ​    }  
62.   
63. ​    if (type == NULL) {  
64. ​        ......  
65. ​        return BAD_INDEX;  
66. ​    }  
67.   
68. ​    offset += dtohl(type->entriesStart);  
69. ​    ......  
70.   
71. ​    const ResTable_entry* const entry = (const ResTable_entry*)  
72. ​        (((const uint8_t*)type) + offset);  
73. ​    ......  
74.   
75. ​    *outType = type;  
76. ​    *outEntry = entry;  
77. ​    if (outTypeClass != NULL) {  
78. ​        *outTypeClass = allTypes;  
79. ​    }  
80. ​    return offset + dtohs(entry->size);  
81. }  

​        这个函数定义在文件frameworks/base/libs/utils/ResourceTypes.cpp中。

​        ResTable类的成员函数getEntry首先是在参数pakcage所描述的一个Package中找到与参数typeIndex所对应的一个Type结构体allTypes。这个Type结构体综合了我们在前面[Android应用程序资源的编译和打包过程分析](http://blog.csdn.net/luoshengyang/article/details/8744683)一文所描述的资源项的ResTable_typeSpec数据块和ResTable_type数据块，也就是，这个Type结构体的成员变量typeSpecFlags所指向的一个uint32_t数组描述了同一种类型的所有资源项的配置差异性性信息，即相当于一系列ResTable_typeSpec数据块，而成员变量configs所指向的一个ResTable_type数组描述的同一种类型的资源项的具体内容，即相当于一系列ResTable_type数据块。

​        结合前面[Android应用程序资源的编译和打包过程分析](http://blog.csdn.net/luoshengyang/article/details/8744683)一文对ResTable_typeSpec数据块和ResTable_type数据块的描述，我们就可以比较容易理解ResTable类的成员函数getEntry的实现了。由于Type结构体allTypes的成员变量configs所指向的一个ResTable_type数组描述的所有类型为typeIndex的资源项的具体内容，因此，ResTable类的成员函数getEntry只要遍历这个ResTable_type数组，并且找到与参数entryIdex所对应的最匹配资源项即可，也就是最能匹配参数config所描述的配置信息的资源项即可。

​        我们知道，在每一个ResTable_type结构体中，都包含有一个类型为uint32_t的偏移数组。偏移数组中的第entryIndex个元素的值就表示Entry ID等于entryIndex的资源项的具体内容相对于ResTable_type结构体的偏移值。有了这个偏移值之后，我们就得到Entry ID等于entryIndex的资源项的具体内容了，也就是得到一个对应的ResTable_entry结构体。

​        注意，每一个ResTable_type结构体都有一个成员变量config，它描述的是一个ResTable_config对象，表示该ResTable_type结构体的配置信息。如果该配置信息要求的配置信息不匹配，也就是与参数config所描述的设置配置信息不匹配，那么就不需要进一步检查该ResTable_type结构体是否存在Entry ID等于entryIndex的资源项了。

​        如果一个ResTable_type结构体的配置信息与参数config所描述的设置配置信息匹配，但是它不存在Entry ID等于entryIndex的资源项，即它的偏移数组中的第entryIndex个元素的值等于ResTable_type::NO_ENTRY，那么也不需要进一步对该ResTable_type结构体进行处理了。

​        如果一个ResTable_type结构体的配置信息与参数config所描述的设置配置信息匹配，并且它也存在Entry ID等于entryIndex的资源项，那么就需要比较前后两个ResTable_type结构体的配置信息与参数config所描述的设置配置信息相比，哪一个更具体一些。具有更具体的配置信息的资源项将作为最匹配的资源项返回给调用者。注意，前一个ResTable_type结构体及其配置信息分别保存在变量type和bestConfig中，并且在该ResTable_type结构体中Entry ID等于entryIndex的资源项相对它的偏移值保存在变量offset中。

​        遍历完成上面所述的ResTable_type数组之后，最终得到的最区配资源项的相关信息就保存在变量type、bestConfig和offset中。这时候将变量type所描述的ResTable_type结构体的起始位置，加上它的成员变量entriesStart的值，以及再加上变量offset的值，就可以得到一个对应的ResTable_entry结构体entry。这个ResTable_entry结构体entry就是用来描述最终在参数package所描述的Package中，找到与参数typeIndex、entryIndex和config最匹配的资源项的信息了。

​       最终，ResTable类的成员函数getEntry就将前面所得到的ResTable_entry结构体entry、ResTable_type结构体type以及Type结构体allTypes分别保存在输出参数outEntry、outType和outTypeClass返回给调用者了。同时，ResTable类的成员函数getEntry还会将变量offset的值加上ResTable_entry结构体entry的大小的结果返回给调用者。调用都得到这个结果之后，就可以将它作为输出参数outType所指向的ResTable_type结构体的偏移量，从而可以得到参数outEntry所指向的ResTable_entry结构体所描述的资源项的具体内容，也就是得到一个Res_value结构体，这个具体就可以参考前面对ResTable类的成员函数getResource的分析。

​        这一步执行完成之后，回到前面的Step 8中，即AssetManager类的成员函数loadResourceValue中，接下来它就会调用前面所获得的一个ResTable对象的成员函数resolveReference来解析前面所获得的资源项的内容了。

​         Step 11. ResTable.resolveReference

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. ssize_t ResTable::resolveReference(Res_value* value, ssize_t blockIndex,  
2. ​        uint32_t* outLastRef, uint32_t* inoutTypeSpecFlags,  
3. ​        ResTable_config* outConfig) const  
4. {  
5. ​    int count=0;  
6. ​    while (blockIndex >= 0 && value->dataType == value->TYPE_REFERENCE  
7. ​           && value->data != 0 && count < 20) {  
8. ​        if (outLastRef) *outLastRef = value->data;  
9. ​        uint32_t lastRef = value->data;  
10. ​        uint32_t newFlags = 0;  
11. ​        const ssize_t newIndex = getResource(value->data, value, true, &newFlags,  
12. ​                outConfig);  
13. ​        if (newIndex == BAD_INDEX) {  
14. ​            return BAD_INDEX;  
15. ​        }  
16. ​        ......  
17. ​        //printf("Getting reference 0x%08x: newIndex=%d\n", value->data, newIndex);  
18. ​        if (inoutTypeSpecFlags != NULL) *inoutTypeSpecFlags |= newFlags;  
19. ​        if (newIndex < 0) {  
20. ​            // This can fail if the resource being referenced is a style...  
21. ​            // in this case, just return the reference, and expect the  
22. ​            // caller to deal with.  
23. ​            return blockIndex;  
24. ​        }  
25. ​        blockIndex = newIndex;  
26. ​        count++;  
27. ​    }  
28. ​    return blockIndex;  
29. }  

​        这个函数定义在文件frameworks/base/libs/utils/ResourceTypes.cpp中。

​        ResTable类的成员函数resolveReference的实现其实很简单，它就是对参数value所描述的一个资源项值进行解析，前提是这个资源项值是一个引用，即value所指向的一个Res_value结构体的成员变量dataType的值等于TYPE_REFERENCE，因为如果不是引用类型的资源项值，就没有必要解析了。

​        注意，一个资源项的值有可能是嵌套引用的，也就是可能是引用的引用，因此，ResTable类的成员函数resolveReference需要使用一个while循环来不断地对参数value所描述的一个资源项值进行解析，直到最后一次解析出来的结果不是引用为止。但是，为了防止无限地解析下去，该while循环最多只允许执行20次。每一次解析都是调用ResTable类的成员函数getResource来实现的，这个成员函数我们在前面的Step 10中已经分析过了。

​        此外，上述while循环还有两个条件需要满足。第一个条件是每一次调用ResTable类的成员函数getResource来解析参数value所描述的资源项值之后，得到的返回值blockIndex都必须是大于等于0的，这是因为它表示该次解析是成功的。由于参数blockIndex最开始的值是由调用者传进来的，因此，也要保证它最开始的值大于等于0，才会执行代码中的while循环对参数value所描述的资源项值进行解析。第二个条件是参数value所描述的资源项的值不能等于0，即它所指向的一个Res_value结构体的成员变量data的值不能等于0，这是因为引用者都是不可能等于0的。

​        这一步执行完成之后，沿着调用路径最终返回到前面的Step 5中，即Resources类的成员函数loadXmlResourceParser中，我们就可以得到参数id所描述的资源项的值了。在我们这个情景中，参数id描述的是一个layout资源ID，它所对应的资源项的值是一个字符串，这个字符串描述的便是一个UI布局文件，即一个经过编译的、以二进制格式保存的Xml资源文件。有了这个Xml资源文件的路径之后，Resources类的另外一个四个参数版本的成员函数loadXmlResourceParser就会被调用来对该Xml资源文件进行解析，以便可以得到一个UI布局视图。

​        Step 12. Resources.loadXmlResourceParser

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. public class Resources {  
2. ​    ......  
3.   
4. ​    private int mLastCachedXmlBlockIndex = -1;  
5. ​    private final int[] mCachedXmlBlockIds = { 0, 0, 0, 0 };  
6. ​    private final XmlBlock[] mCachedXmlBlocks = new XmlBlock[4];  
7. ​    ......  
8.   
9. ​    /*package*/ XmlResourceParser loadXmlResourceParser(String file, int id,  
10. ​            int assetCookie, String type) throws NotFoundException {  
11. ​        if (id != 0) {  
12. ​            try {  
13. ​                // These may be compiled...  
14. ​                synchronized (mCachedXmlBlockIds) {  
15. ​                    // First see if this block is in our cache.  
16. ​                    final int num = mCachedXmlBlockIds.length;  
17. ​                    for (int i=0; i<num; i++) {  
18. ​                        if (mCachedXmlBlockIds[i] == id) {  
19. ​                            ......  
20. ​                            return mCachedXmlBlocks[i].newParser();  
21. ​                        }  
22. ​                    }  
23.   
24. ​                    // Not in the cache, create a new block and put it at  
25. ​                    // the next slot in the cache.  
26. ​                    XmlBlock block = mAssets.openXmlBlockAsset(  
27. ​                            assetCookie, file);  
28. ​                    if (block != null) {  
29. ​                        int pos = mLastCachedXmlBlockIndex+1;  
30. ​                        if (pos >= num) pos = 0;  
31. ​                        mLastCachedXmlBlockIndex = pos;  
32. ​                        XmlBlock oldBlock = mCachedXmlBlocks[pos];  
33. ​                        if (oldBlock != null) {  
34. ​                            oldBlock.close();  
35. ​                        }  
36. ​                        mCachedXmlBlockIds[pos] = id;  
37. ​                        mCachedXmlBlocks[pos] = block;  
38. ​                        ......  
39. ​                        return block.newParser();  
40. ​                    }  
41. ​                }  
42. ​            } catch (Exception e) {  
43. ​                NotFoundException rnf = new NotFoundException(  
44. ​                        "File " + file + " from xml type " + type + " resource ID #0x"  
45. ​                        + Integer.toHexString(id));  
46. ​                rnf.initCause(e);  
47. ​                throw rnf;  
48. ​            }  
49. ​        }  
50.   
51. ​        throw new NotFoundException(  
52. ​                "File " + file + " from xml type " + type + " resource ID #0x"  
53. ​                + Integer.toHexString(id));  
54. ​    }  
55.   
56. ​    ......  
57. }  

​        这个函数定义在文件frameworks/base/core/java/android/content/res/Resources.java中

​        Resources类有三个成员变量是与Xml资源文件缓存文件有关的，它们分别是mCachedXmlBlocks、mCachedXmlBlockIds和mLastCachedXmlBlockIndex。其中，mCachedXmlBlocks指向的是一个大小等于4的XmlBlock数组，mCachedXmlBlockIds指向的也是一个大小等于4的资源ID数组，而mLastCachedXmlBlockIndex表示上一次缓存的Xml资源文件在上述XmlBlock数组和资源ID数组中的索引。这就是说，Resources类最多可以缓存最近读取的四个Xml资源文件的内容，读取超过四个Xml资源文件之后 ，上述的XmlBlock数组和资源ID数组就会被循环利用。

​        理解了Resources类的上述三个成员变量的含义之后，Resources类的成员函数loadXmlResourceParser的实现就容易理解了。它首先是在成员变量mCachedXmlBlockIds所指向的资源ID数组中检查是否存在一个资源ID与参数id所描述的资源ID相等。如果存在的话，那么就会在成员变量mCachedXmlBlocks所指向的XmlBlock数组中找到一个对应的XmlBlock对象，并且调用这个XmlBlock对象的成员函数newParser来创建一个XmlResourceParser对象返回给调用者。

​        如果在Resources类的成员变量mCachedXmlBlockIds所指向的资源ID数组找到对应的资源ID的话，那么Resources类的成员函数loadXmlResourceParser就会调用成员变量mAssets所指向的一个Java层的AssetManager对象的成员函数openXmlBlockAsset来打开参数file所指定的Xml资源文件，从而获得一个XmlBlock对象block。这个XmlBlock对象block以及参数id所描述的资源ID同时也会被缓存在Resources类的成员变量mCachedXmlBlocks和mCachedXmlBlockIds所描述的XmlBlock数组和资源ID数组中。

​        最后，Resources类的成员函数loadXmlResourceParser就可以调用前面得到的XmlBlock对象block的成员函数newParser来创建一个XmlResourceParser对象返回给调用者。

​        接下来，我们就首先分析Java层的AssetManager类的成员函数openXmlBlockAsset的实现，接着再分析XmlBlock类的成员函数newParser的实现，以便可以了解一个Xml资源文件的打开过程。

​        Step 13. AssetManager.openXmlBlockAsset

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. public final class AssetManager {  
2. ​    ......  
3.   
4. ​    /*package*/ final XmlBlock openXmlBlockAsset(int cookie, String fileName)  
5. ​        throws IOException {  
6. ​        synchronized (this) {  
7. ​            if (!mOpen) {  
8. ​                throw new RuntimeException("Assetmanager has been closed");  
9. ​            }  
10. ​            int xmlBlock = openXmlAssetNative(cookie, fileName);  
11. ​            if (xmlBlock != 0) {  
12. ​                XmlBlock res = new XmlBlock(this, xmlBlock);  
13. ​                incRefsLocked(res.hashCode());  
14. ​                return res;  
15. ​            }  
16. ​        }  
17. ​        throw new FileNotFoundException("Asset XML file: " + fileName);  
18. ​    }  
19.   
20. ​    ......  
21. }  

​        这个函数定义在文件frameworks/base/core/java/android/content/res/AssetManager.java中。

​        AssetManager类的成员函数openXmlBlockAsset首先检查成员变量mOpen的值是否不等于true。如果不等于true的话，那么就说明当前正在处理的AssetManager对象还没有经过初始化，或者已经关闭了。

​        我们假设AssetManager类的成员变量mOpen的值等于true，那么接下来AssetManager类的成员函数openXmlBlockAsset就会调用另外一个成员函数openXmlAssetNative来打开参数fileName所指定的Xml资源文件。

​        成功打开参数fileName所指向的Xml资源文件之后，就会得到一个C++层的ResXMLTree对象的地址值xmlBlock，最后AssetManager类的成员函数openXmlBlockAsset就将该C++层的ResXMLTree对象的地址值封装在一个Java层的XmlBlock对象中，并且将该XmlBlock对象返回给调用者。

​        接下来，我们就继续分析AssetManager类的成员函数openXmlAssetNative的实现。

​        Step 14.  AssetManager.openXmlAssetNative

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. public final class AssetManager {  
2. ​    ......  
3.   
4. ​    private native final int openXmlAssetNative(int cookie, String fileName);  
5.   
6. ​    ......  
7. }  

​        这个函数定义在文件frameworks/base/core/java/android/content/res/AssetManager.java中。

​        AssetManager类的成员函数openXmlAssetNative是一个JNI方法，它是C++层的函数android_content_AssetManager_openXmlAssetNative来实现的，如下所示：

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. static jint android_content_AssetManager_openXmlAssetNative(JNIEnv* env, jobject clazz,  
2. ​                                                         jint cookie,  
3. ​                                                         jstring fileName)  
4. {  
5. ​    AssetManager* am = assetManagerForJavaObject(env, clazz);  
6. ​    ......  
7.   
8. ​    const char* fileName8 = env->GetStringUTFChars(fileName, NULL);  
9. ​    Asset* a = cookie  
10. ​        ? am->openNonAsset((void*)cookie, fileName8, Asset::ACCESS_BUFFER)  
11. ​        : am->openNonAsset(fileName8, Asset::ACCESS_BUFFER);  
12. ​    ......  
13.   
14. ​    env->ReleaseStringUTFChars(fileName, fileName8);  
15.   
16. ​    ResXMLTree* block = new ResXMLTree();  
17. ​    status_t err = block->setTo(a->getBuffer(true), a->getLength(), true);  
18. ​    a->close();  
19. ​    delete a;  
20. ​    ......  
21.   
22. ​    return (jint)block;  
23. }  

​        这个函数定义在文件frameworks/base/core/jni/android_util_AssetManager.cpp中。        

​        函数android_content_AssetManager_openXmlAssetNative首先调用另外一个函数assetManagerForJavaObject来将参数clazz所指向的一个Java层的AssetManager对象成员变量mObject转换为一个C++层的AssetManager对象。有了这个C++层的AssetManager对象之后，就可以调用它的成员函数openNonAsset来打开参数fileName所指定的Xml资源文件。

​        调用C++层的AssetManager对象的成员函数openNonAsset来成功地打开参数fileName所指定的Xml资源文件之后，函数android_content_AssetManager_openXmlAssetNative接下来就会创建一个ResXMLTree对象，并且将前面所打开的Xml资源文件的内容设置到该ResXMLTree对象中去，并且将该ResXMLTree对象的地址值返回给调用者。

​        假设参数cookie的值大于0，因此，函数android_content_AssetManager_openXmlAssetNative实际调用的是C++层的AssetManager类的三个参数版本的成员函数openNonAsset来打开参数fileName所指定的Xml资源文件，接下来，我们就分析这个函数的实现。

​        Step 15. AssetManager.openNonAsset

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. Asset* AssetManager::openNonAsset(void* cookie, const char* fileName, AccessMode mode)  
2. {  
3. ​    const size_t which = ((size_t)cookie)-1;  
4.   
5. ​    AutoMutex _l(mLock);  
6.   
7. ​    ......  
8.   
9. ​    if (which < mAssetPaths.size()) {  
10. ​        ......  
11. ​        Asset* pAsset = openNonAssetInPathLocked(  
12. ​            fileName, mode, mAssetPaths.itemAt(which));  
13. ​        if (pAsset != NULL) {  
14. ​            return pAsset != kExcludedAsset ? pAsset : NULL;  
15. ​        }  
16. ​    }  
17.   
18. ​    return NULL;  
19. }  

​        这个函数定义在文件frameworks/base/libs/utils/AssetManager.cpp中。

​        参数cookie是用来标识另外一个参数fileName所指向的文件是属于哪个APK文件的。从前面[Android应用程序资源管理器（Asset Manager）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8791064)一文可以知道，参数cookie实际上是一个整数，将这个整数减去1之后，得到的结果就是参数fileName所指向的文件所属于的APK文件在AssetManager类的成员变量mAssetPaths所描述的一个数组的索引。AssetManager类的成员变量mAssetPaths描述的是一个asset_path数组，数组中的每一个元素都是表示一个APK文件路径，这些APK文件就包含了当前应用程序所要使用的资源包。

​        AssetManager类的成员函数openNonAsset通过参数cookie知道了当前要打开的文件是位于哪个APK文件之后，接着就继续调用另外一个成员函数openNonAssetInPathLocked来打开该文件。

​        Step 16. AssetManager.openNonAssetInPathLocked

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. Asset* AssetManager::openNonAssetInPathLocked(const char* fileName, AccessMode mode,  
2. ​    const asset_path& ap)  
3. {  
4. ​    Asset* pAsset = NULL;  
5.   
6. ​    /* look at the filesystem on disk */  
7. ​    if (ap.type == kFileTypeDirectory) {  
8. ​        String8 path(ap.path);  
9. ​        path.appendPath(fileName);  
10.   
11. ​        pAsset = openAssetFromFileLocked(path, mode);  
12.   
13. ​        if (pAsset == NULL) {  
14. ​            /* try again, this time with ".gz" */  
15. ​            path.append(".gz");  
16. ​            pAsset = openAssetFromFileLocked(path, mode);  
17. ​        }  
18.   
19. ​        if (pAsset != NULL) {  
20. ​            //printf("FOUND NA '%s' on disk\n", fileName);  
21. ​            pAsset->setAssetSource(path);  
22. ​        }  
23.   
24. ​    /* look inside the zip file */  
25. ​    } else {  
26. ​        String8 path(fileName);  
27.   
28. ​        /* check the appropriate Zip file */  
29. ​        ZipFileRO* pZip;  
30. ​        ZipEntryRO entry;  
31.   
32. ​        pZip = getZipFileLocked(ap);  
33. ​        if (pZip != NULL) {  
34. ​            //printf("GOT zip, checking NA '%s'\n", (const char*) path);  
35. ​            entry = pZip->findEntryByName(path.string());  
36. ​            if (entry != NULL) {  
37. ​                //printf("FOUND NA in Zip file for %s\n", appName ? appName : kAppCommon);  
38. ​                pAsset = openAssetFromZipLocked(pZip, entry, mode, path);  
39. ​            }  
40. ​        }  
41.   
42. ​        if (pAsset != NULL) {  
43. ​            /* create a "source" name, for debug/display */  
44. ​            pAsset->setAssetSource(  
45. ​                    createZipSourceNameLocked(ZipSet::getPathName(ap.path.string()), String8(""),  
46. ​                                                String8(fileName)));  
47. ​        }  
48. ​    }  
49.   
50. ​    return pAsset;  
51. }  

​        这个函数定义在文件frameworks/base/libs/utils/AssetManager.cpp中。

​        AssetManager类的成员函数openNonAssetInPathLocked的实现是比较简单的，它按照以下两种方式来打开参数fileName所指向的文件。

​        如果参数ap描述的文件路径是一个目录，那么它就会直接将参数fileName所指向的文件名称附加到该目录后面去，然后直接调用另外一个成员函数openAssetFromFileLocked来打开参数fileName所指向的文件。如果打开失败，那么就再假定参数ap描述的文件路径是一个以“.gz“为后缀的压缩包，然后再次调用成员函数openAssetFromFileLocked来打开参数fileName所指向的文件。

​        如果参数ap描述的文件路径是一个普通文件，那么就意味着参数ap描述的是一个压缩文件，因此，它就会先调用成员函数getZipFileLocked来打开该压缩文件，然后再调用成员函数openAssetFromZipLocked来在该压缩包将参数fileName所指向的文件提取出来。

​        Android应用程序的资源一般都是打包在一个APK文件里面的，而APK文件就是一个Zip格式的压缩包，因此，AssetManager类的成员函数openNonAssetInPathLocked一般就是按照第二种方式来打开参数fileName所指向的文件。

​        接下来，我们就继续分析AssetManager类的成员函数openAssetFromZipLocked的实现。

​        Step 17. AssetManager.openAssetFromZipLocked

**[cpp]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. Asset* AssetManager::openAssetFromZipLocked(const ZipFileRO* pZipFile,  
2. ​    const ZipEntryRO entry, AccessMode mode, const String8& entryName)  
3. {  
4. ​    Asset* pAsset = NULL;  
5.   
6. ​    // TODO: look for previously-created shared memory slice?  
7. ​    int method;  
8. ​    size_t uncompressedLen;  
9. ​    ......  
10.   
11. ​    if (!pZipFile->getEntryInfo(entry, &method, &uncompressedLen, NULL, NULL,  
12. ​            NULL, NULL))  
13. ​    {  
14. ​        LOGW("getEntryInfo failed\n");  
15. ​        return NULL;  
16. ​    }  
17.   
18. ​    FileMap* dataMap = pZipFile->createEntryFileMap(entry);  
19. ​    if (dataMap == NULL) {  
20. ​        LOGW("create map from entry failed\n");  
21. ​        return NULL;  
22. ​    }  
23.   
24. ​    if (method == ZipFileRO::kCompressStored) {  
25. ​        pAsset = Asset::createFromUncompressedMap(dataMap, mode);  
26. ​        LOGV("Opened uncompressed entry %s in zip %s mode %d: %p", entryName.string(),  
27. ​                dataMap->getFileName(), mode, pAsset);  
28. ​    } else {  
29. ​        pAsset = Asset::createFromCompressedMap(dataMap, method,  
30. ​            uncompressedLen, mode);  
31. ​        LOGV("Opened compressed entry %s in zip %s mode %d: %p", entryName.string(),  
32. ​                dataMap->getFileName(), mode, pAsset);  
33. ​    }  
34. ​    if (pAsset == NULL) {  
35. ​        /* unexpected */  
36. ​        LOGW("create from segment failed\n");  
37. ​    }  
38.   
39. ​    return pAsset;  
40. }  

​        这个函数定义在文件frameworks/base/libs/utils/AssetManager.cpp中。

​        AssetManager类的成员函数openAssetFromZipLocked的实现也很简单，它无非就是从参数pZipFile所描述的压缩包中将参数entryName所描述的文件提取出来，并且根据提取出来的内容保存在一个Asset对象中返回给调用者。

​       注意，参数entryName所描述的文件有可能是经过是经过压缩后再打包到参数pZipFile所描述的压缩包去的。在这种情况下，AssetManager类的成员函数openAssetFromZipLocked就会调用Asset类的静态成员函数createFromCompressedMap来对它进行解压，然后再将解压完成后得到的内容保存在一个Asset对象中。否则的话，AssetManager类的成员函数openAssetFromZipLocked就会调用Asset类的静态成员函数createFromUncompressedMap来将它的内容保存在一个Asset对象中。

​        这一步执行完成之后，返回到前面的Step 12中，即Resources类的成员函数loadXmlResourceParser中，接下来就会调用Java层的XmlBlock类的成员函数newParser来创建一个XmlResourceParser对象，以便可以用来解析前面打开的Xml资源文件。

​       Step 18. XmlBlock.newParser

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. final class XmlBlock {  
2. ​    ......  
3.   
4. ​    public XmlResourceParser newParser() {  
5. ​        synchronized (this) {  
6. ​            if (mNative != 0) {  
7. ​                return new Parser(nativeCreateParseState(mNative), this);  
8. ​            }  
9. ​            return null;  
10. ​        }  
11. ​    }  
12.   
13. ​    ......  
14.   
15. ​    private final int mNative;  
16. ​    ......  
17.   
18. ​    private static final native int nativeCreateParseState(int obj);  
19. ​    ......  
20. }  

​        这个函数定义在文件frameworks/base/core/java/android/content/res/XmlBlock.java中。

​        XmlBlock类的成员函数mNative指向的是C++层的一个ResXMLTree对象。这个ResXMLTree对象是在前面的Step 14中创建的，用来描述前面所打开的一个Xml资源文件。

​        XmlBlock类的成员函数newParser首先是以成员变量mNative所描述的一个C++层的ResXMLTree对象的地址为参数，来调用JNI方法nativeCreateParseState，用来在C++层创建一个ResXMLParser对象，最后再将该C++层的ResXMLParser对象封装成Java层的一个Parser对象中，并且将该Parser对象返回给调用者。

​        这一步执行完成之后，返回到前面的Step 3中，即LayoutInflater类的成员函数inflate中，接下来它就会以前面所获得的一个Java层的Parser对象来参数，来调用另外一个重载版本的成员函数inflate，用来解析前面所打开的Xml资源文件，即一个UI布局文件，以便可以创建相应的UI布局出来。

​        Step 19. LayoutInflater.inflate

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. public abstract class LayoutInflater {  
2. ​    ......  
3.   
4. ​    public View inflate(XmlPullParser parser, ViewGroup root, boolean attachToRoot) {  
5. ​        synchronized (mConstructorArgs) {  
6. ​            final AttributeSet attrs = Xml.asAttributeSet(parser);  
7. ​            Context lastContext = (Context)mConstructorArgs[0];  
8. ​            mConstructorArgs[0] = mContext;  
9. ​            View result = root;  
10.   
11. ​            try {  
12. ​                // Look for the root node.  
13. ​                int type;  
14. ​                while ((type = parser.next()) != XmlPullParser.START_TAG &&  
15. ​                        type != XmlPullParser.END_DOCUMENT) {  
16. ​                    // Empty  
17. ​                }  
18.   
19. ​                if (type != XmlPullParser.START_TAG) {  
20. ​                    throw new InflateException(parser.getPositionDescription()  
21. ​                            + ": No start tag found!");  
22. ​                }  
23.   
24. ​                final String name = parser.getName();  
25.   
26. ​                ......  
27.   
28. ​                if (TAG_MERGE.equals(name)) {  
29. ​                    ......  
30.   
31. ​                    rInflate(parser, root, attrs);  
32. ​                } else {  
33. ​                    // Temp is the root view that was found in the xml  
34. ​                    View temp = createViewFromTag(name, attrs);  
35.   
36. ​                    ViewGroup.LayoutParams params = null;  
37.   
38. ​                    if (root != null) {  
39. ​                        ......  
40.   
41. ​                        // Create layout params that match root, if supplied  
42. ​                        params = root.generateLayoutParams(attrs);  
43. ​                        if (!attachToRoot) {  
44. ​                            // Set the layout params for temp if we are not  
45. ​                            // attaching. (If we are, we use addView, below)  
46. ​                            temp.setLayoutParams(params);  
47. ​                        }  
48. ​                    }  
49.   
50. ​                    ......  
51.   
52. ​                    // Inflate all children under temp  
53. ​                    rInflate(parser, temp, attrs);  
54. ​                    ......  
55.   
56. ​                    // We are supposed to attach all the views we found (int temp)  
57. ​                    // to root. Do that now.  
58. ​                    if (root != null && attachToRoot) {  
59. ​                        root.addView(temp, params);  
60. ​                    }  
61.   
62. ​                    // Decide whether to return the root that was passed in or the  
63. ​                    // top view found in xml.  
64. ​                    if (root == null || !attachToRoot) {  
65. ​                        result = temp;  
66. ​                    }  
67. ​                }  
68.   
69. ​            } catch (XmlPullParserException e) {  
70. ​                ......  
71. ​            } catch (IOException e) {  
72. ​                ......  
73. ​            } finally {  
74. ​                ......  
75. ​            }  
76.   
77. ​            return result;  
78. ​        }  
79. ​    }  
80.   
81. ​    ......  
82. }  

​        这个函数定义在文件frameworks/base/core/java/android/view/LayoutInflater.java中。

​        LayoutInflater类的成员函数inflate主要负责处理前面所打开的Xml资源文件的根节点，然后再调用另外一个成员函数rInflate来处理根节点的子节点。每一个节点都表示一个UI控件，这个UI控件是通过调用LayoutInflater类的成员函数createViewFromTag来创建的。

​        LayoutInflater类的成员函数createViewFromTag需要两个参数来创建一个UI控件。这两个参数分别对应于当前正在处理的Xml节点的名称以及属性集。注意，如果参数root的值不等于null，那么它所描述的一个ViewGroup就是调用LayoutInflater类的成员函数createViewFromTag获得的UI控件的父控件。因此，调用LayoutInflater类的成员函数createViewFromTag获得的UI控件及其对应的布局参数，最后都需要添加到参数root所描述的一个ViewGroup中去。

​        有一种特殊情况，如果当前正在处理的Xml节点的名称等于TAG_MERGE，即“merge”，那么就表示不用处理的当前正在处理的Xml节点，而是直接去处理当前正在处理的Xml节点的子节点。这是Android系统为提供的一种UI布局优化机制，实际上就是减少了一层UI嵌套，具体可以参考官方文档：[http://developer.android.com/training/improving-layouts/reusing-layouts.html](http://developer.android.com/training/improving-layouts/reusing-layouts.html)。

​         LayoutInflater类的成员函数rInflate在处理当前节点的子节点的时候，也是通过调用成员函数createViewFromTag来创建相应的UI控件的，因此，接下来我们就主要分析LayoutInflater类的成员函数createViewFromTag的实现。

​         Step 20. LayoutInflater.createViewFromTag

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. public abstract class LayoutInflater {  
2. ​    ......  
3.   
4. ​    private Factory mFactory;  
5. ​    ......  
6.   
7. ​    View createViewFromTag(String name, AttributeSet attrs) {  
8. ​        if (name.equals("view")) {  
9. ​            name = attrs.getAttributeValue(null, "class");  
10. ​        }  
11.   
12. ​        ......  
13.   
14. ​        try {  
15. ​            View view = (mFactory == null) ? null : mFactory.onCreateView(name,  
16. ​                    mContext, attrs);  
17.   
18. ​            if (view == null) {  
19. ​                if (-1 == name.indexOf('.')) {  
20. ​                    view = onCreateView(name, attrs);  
21. ​                } else {  
22. ​                    view = createView(name, null, attrs);  
23. ​                }  
24. ​            }  
25.   
26. ​            ......  
27. ​            return view;  
28.   
29. ​        } catch (InflateException e) {  
30. ​            ......  
31. ​        } catch (ClassNotFoundException e) {  
32. ​            ......  
33. ​        } catch (Exception e) {  
34. ​            ......  
35. ​        }  
36. ​    }  
37.   
38. ​    ......  
39. }  

​        这个函数定义在文件frameworks/base/core/java/android/view/LayoutInflater.java中。

​        参数name表示当前正在处理的Xml节点的名称。如果它的值等于“view”的话，那么真正要创建的UI控件的类名记录在参数attrs所描述的一个属性集中的一个名称为“class”的属性中。因此，当参数name的值等于“view”的时候，LayoutInflater类的成员函数createViewFromTag首先要做的便是从参数attrs所描述的一个属性集中获取接下来真正要创建的UI控件的类名。

​        LayoutInflater类的成员函数createViewFromTag接下来检查成员变量mFactory的值是否不等于null。如果不等于null的话，那么它就会指向一个Factory对象，该Factory对象描述的是一个UI控件创建工厂，专门用来负责创建UI控件。

​        如果LayoutInflater类的成员变量mFactory的值等于null，那么LayoutInflater类的成员函数createViewFromTag就会调用成员函数onCreateView或者createView来创建由参数name所指定的UI控件，取决于参数name是否包含了一个“.”字符。注意，如果参数name是否包含了一个“.”字符，那么就说明当前所创建的UI控件是一个用户自定义的UI控件，也就是不是Android提供的标准控件。

​        我们假设LayoutInflater类的成员变量mFactory的值等于null，并且参数name没有包含有“.”字符，那么LayoutInflater类的成员函数createViewFromTag最后就会调用成员函数onCreateView来创建由参数name所指定的UI控件。

​        LayoutInflater类的成员函数onCreateView是由其子类来重写的。从前面的Step 2可以知道，当前正在处理的实际上是一个PhoneLayoutInflater对象。PhoneLayoutInflater类继承了LayoutInflater类，并且重写了成员函数onCreateView。因此，接下来我们就继续分析PhoneLayoutInflater类的成员函数onCreateView的实现。

​        Step 21. PhoneLayoutInflater.onCreateView

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. public class PhoneLayoutInflater extends LayoutInflater {  
2. ​    private static final String[] sClassPrefixList = {  
3. ​        "android.widget.",  
4. ​        "android.webkit."  
5. ​    };  
6.   
7. ​    @Override protected View onCreateView(String name, AttributeSet attrs) throws ClassNotFoundException {  
8. ​        for (String prefix : sClassPrefixList) {  
9. ​            try {  
10. ​                View view = createView(name, prefix, attrs);  
11. ​                if (view != null) {  
12. ​                    return view;  
13. ​                }  
14. ​            } catch (ClassNotFoundException e) {  
15. ​                // In this case we want to let the base class take a crack  
16. ​                // at it.  
17. ​            }  
18. ​        }  
19.   
20. ​        return super.onCreateView(name, attrs);  
21. ​    }  
22.   
23. ​    ......  
24. }  

​        这个函数定义在文件frameworks/base/policy/src/com/android/internal/policy/impl/PhoneLayoutInflater.java中。

​        PhoneLayoutInflater类的成员函数onCreateView只负责创建两类标准的UI控件，一种是属于android.widget包的，另一种是属于android.webkit包的，其中，优先创建android.widget包的UI控件。

​        如果参数name所描述的UI控件既不属于android.widget包的，也不属于android.webkit包的，那么 PhoneLayoutInflater类的成员函数onCreateView就会将创建UI控件的操作交给父类来处理，即通过调用父类的成员函数onCreateView来创建。

​        如果参数name所描述的UI控件是属于android.widget包或者android.webkit包的，那么PhoneLayoutInflater类的成员函数onCreateView就会直接调用父类LayoutInflater的成员函数createView来创建参数name所描述的UI控件，因此，接下来我们就继续分析LayoutInflater类的成员函数createView的实现。

​        Step 22. LayoutInflater.createView

**[java]** [view plain](http://blog.csdn.net/luoshengyang/article/details/8806798#) [copy](http://blog.csdn.net/luoshengyang/article/details/8806798#)

1. public abstract class LayoutInflater {  
2. ​    ......  
3.   
4. ​    private Filter mFilter;  
5.   
6. ​    private final Object[] mConstructorArgs = new Object[2];  
7. ​    ......  
8.   
9. ​    private static final HashMap<String, Constructor> sConstructorMap =  
10. ​            new HashMap<String, Constructor>();  
11.   
12. ​    private HashMap<String, Boolean> mFilterMap;  
13. ​    ......  
14.   
15. ​    public final View createView(String name, String prefix, AttributeSet attrs)  
16. ​            throws ClassNotFoundException, InflateException {  
17. ​        Constructor constructor = sConstructorMap.get(name);  
18. ​        Class clazz = null;  
19.   
20. ​        try {  
21. ​            if (constructor == null) {  
22. ​                // Class not found in the cache, see if it's real, and try to add it  
23. ​                clazz = mContext.getClassLoader().loadClass(  
24. ​                        prefix != null ? (prefix + name) : name);  
25.   
26. ​                if (mFilter != null && clazz != null) {  
27. ​                    boolean allowed = mFilter.onLoadClass(clazz);  
28. ​                    if (!allowed) {  
29. ​                        failNotAllowed(name, prefix, attrs);  
30. ​                    }  
31. ​                }  
32. ​                constructor = clazz.getConstructor(mConstructorSignature);  
33. ​                sConstructorMap.put(name, constructor);  
34. ​            } else {  
35. ​                // If we have a filter, apply it to cached constructor  
36. ​                if (mFilter != null) {  
37. ​                    // Have we seen this name before?  
38. ​                    Boolean allowedState = mFilterMap.get(name);  
39. ​                    if (allowedState == null) {  
40. ​                        // New class -- remember whether it is allowed  
41. ​                        clazz = mContext.getClassLoader().loadClass(  
42. ​                                prefix != null ? (prefix + name) : name);  
43.   
44. ​                        boolean allowed = clazz != null && mFilter.onLoadClass(clazz);  
45. ​                        mFilterMap.put(name, allowed);  
46. ​                        if (!allowed) {  
47. ​                            failNotAllowed(name, prefix, attrs);  
48. ​                        }  
49. ​                    } else if (allowedState.equals(Boolean.FALSE)) {  
50. ​                        failNotAllowed(name, prefix, attrs);  
51. ​                    }  
52. ​                }  
53. ​            }  
54.   
55. ​            Object[] args = mConstructorArgs;  
56. ​            args[1] = attrs;  
57. ​            return (View) constructor.newInstance(args);  
58.   
59. ​        } catch (NoSuchMethodException e) {  
60. ​            ......  
61. ​        } catch (ClassNotFoundException e) {  
62. ​            ......  
63. ​        } catch (Exception e) {  
64. ​            ......  
65. ​        }  
66. ​    }  
67.   
68. ​    ......  
69. }  

​        这个函数定义在文件frameworks/base/core/java/android/view/LayoutInflater.java。

​        LayoutInflater类的静态成员变量sConstructorMap指向的是一个HashMap。这个HashMap缓存了当前应用程序使用过的每一种类型的UI控件类的构造函数。这些构造函数必须具有两个参数，其中第一个参数是一个Context，第二个参数是一个AttributeSet。LayoutInflater类的成员函数createView在创建一个UI控件的时候，就会将上述两个参数保存在LayoutInflater类的成员变量mConstructorArgs所描述的一个大小为2的数组中，并且以这个数组为参数，来调用对应的UI控件类的构造函数。

​        LayoutInflater类的mFilter指向的是一个Filter对象。这个Filter对象描述的是一个过滤器，LayoutInflater类的成员函数createView在创建一个UI控件之前，会先调用该过滤器的成员函数onLoadClass，来询问该过滤器是允许创建由参数name所描述的UI控件。如果不允许的话，LayoutInflater类的成员函数createView就会调用另外一个成员函数failNotAllowed来抛出一个异常。

​       为了避免每次创建一个UI控件时，都去询问过滤器是否允许创建，LayoutInflater类的成员函数createView只会在第一次创建一个名称为name的UI控件时，才会询问过滤器，并且将询问结果保存在成员变量mFilterMap所指向的一个HashMap。这样当LayoutInflater类的成员函数createView以后再创建同名的UI控件时，就可以直接通过成员变量mFilterMap所指向的一个HashMap来知道该UI控件是否是允许创建的。

​        理解了LayoutInflater类的静态成员变量sConstructorMap以及成员变量mConstructorArgs、mFilter和mFilterMap的含义之后，读者就可以自己去理解LayoutInflater类的成员函数createView的实现了。这里需要重复强调的一点就是，我们在自定义一个UI控件的时候，一定要提供一个具有两个参数类型分别为Context和AttributeSet的构造函数，否则的话，该自定义控件就不可以在UI布局文件中使用。

​        至此，我们就以layout资源为例，详细地分析了Android应用程序资源的查找过程了，并且也完整地分析完成Android系统的资源管理框架了。重新学习Android系统的资源管理框架，请参考前面[Android资源管理框架（Asset Manager）简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/8738877)一文。