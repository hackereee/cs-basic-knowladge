# mono.android 创建原生对象的步骤
1. 创建C#侧Object
2. JniEnv创建一个Java对象并获得对象句柄handle
3. 然后通过AndroidRuntime中的Addpeer方法将handle所指对象创建一个grefs(GlobalReference)


#  mono.android 原生对象的释放
释放有两种方式：
1. 主动Dispose, 该方法为用户主动Dispose对象，最终调用`JNIEnvInit.AndroidValueManager?.RemovePeer (this, key_handle);`方法，去释放对象并释放掉创建的grefs handle, 最后还要调用GC.SupressFinalize()方法阻止Mono去主动去析构对象（因为本质上Dipose所要完成的事情和析构是一致的，源码里的调用路径也没多大差异）
2. 析构：析构是C++的概念，实际上在C#中析构函数会被编译为 Finalize 方法，最终垃圾回收时会先调用这个方法对对象进行析构，去释放资源，如我们上面所说，析构函数是被动调用的，Dispose是主动调用的，但本质上在释放一个原生对象时，它们所要完成的事情是一样的，所以在析构函数执行之后这个对象的生命就已终结。


# 为何会出现grefs的解引用bug?
写这个笔记的原因实际上就是为了排查一个Xamarin Android应用的native crash，我们发现基本上都是由Native直接将代码Abort了，一般都是在获取某个Java环境的值(`CallStaticxxxMethodX`)， 或者在删除一个引用的时候(DeleteGlobalRef), Android Native 检查该引用对象堆的时候发现堆已经不存在，从而被ART主动终结了。
## 那么，为什么会这样？
实际在写文章的时候，我还没有找到真正的问题点，原因是其一该问题并不容易复现，基本上十多次才复现一次， 其二是我们项目其实已经很庞大了，需要定位一个没有明确调用栈的错误并不容易， 其三是我们并不能排除是不是三方框架的bug，只有一点可以确定，这一定是某个原生对象的调用时机（删除、解引用）不对而导致的bug。那我们有了以下几个信息：
1. mono android原生对象
2. 调用/删除一个已经被删除的Java侧引用
3. 偶发

那顺着这个思路基本可以确定如果不是mono本身的bug的话，肯定是一个多线程竞争的bug，那就先顺着这个思路排查吧。