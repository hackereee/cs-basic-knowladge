# Surface

## SurfaceFlinger 连接过程
学习资料来自老罗的[《Android应用程序与SurfaceFlinger服务的连接过程分析》](https://blog.csdn.net/luoshengyang/article/details/7857163)一文
文章以BootAnmation开机动画的启动为例（为何可以看文中解释）。
* SurfaceComposerClient:
SurfaceComposerClient 是bootAnamation里的指针，以sp<SurfaceComposerClient>作为引用，在SurfaceComposerClient类内部，有一个类型为sp<ISurfaceComposerClient>的成员变量mClient， `mClient` 实际指向`BpSurfaceComposerClient`,看到Bp,显然是一个Binder Proxy，而在`BpSurfaceComposerClient`的另外那一头（SurfaceFlinger）引用了一个类型为Client的本地对象（BnBinder）。
书接上文，**Client类和BpSurfaceComposerClient类均实现了类型为ISurfaceComposerClient的Binder接口**，`BpSurfaceComposerClient`是在`SurfaceComposerClient`初始化时，通过`SurfaceFliger`的proxy的createConnection()方法实现的，也就是说呢，`SurfaceFlinger`的proxy通过binder调用了SurfaceFlinger的createConnection方法，然后创建了BpSurfaceComposerClient对象通道，又通过Binder将其返回到proxy上，而SurfaceFlinger也就保存了这个`BpSurfaceComposerClient`通道，从而实现了`SurfaceComposerClient`和`SurfaceFlinger`的连接
书接上文，**Client类和BpSurfaceComposerClient类均实现了类型为ISurfaceComposerClient的Binder接口**，`BpSurfaceComposerClient`是在`SurfaceComposerClient`初始化时，通过`SurfaceFliger`的proxy对象的createConnection()方法实现的，核心代码如下：

```C++
class BpSurfaceComposer : public BpInterface<ISurfaceComposer>
{
public:
    ......
 
    virtual sp<ISurfaceComposerClient> createConnection()
    {
        uint32_t n;
        Parcel data, reply;
        data.writeInterfaceToken(ISurfaceComposer::getInterfaceDescriptor());
        remote()->transact(BnSurfaceComposer::CREATE_CONNECTION, data, &reply);
        return interface_cast<ISurfaceComposerClient>(reply.readStrongBinder());
    }

```
为什么是BpSurfaceComposer,这个就不过多解释了，这个类呢呢就是就是SurfaceFlinger， 它们一个就是Bp, SurfaceFlinger侧可以看做Bn, 它们同时实现了ISurfaceComposer接口，然后由代理对象的createConnection方法通过Binder通道发送一个`CREATE_CONNECTION`指令，当SurfaceFlinger侧收到该指令则就解析并调用它本身的createConnection方法，而Bp（代理对象这边呢），也通过interface_cast<>模版函数创建了一个BpSurfaceComposerClient并返回给调用方(SurfaceComposerClient),此时SurfaceFlinger里已经存在了一个Client Native对象，这个对象其实就是BpSurfaceComposerClient的本地对象，这样连接就完成了。


## [Android应用程序与SurfaceFlinger服务之间的共享UI元数据（SharedClient）的创建过程分析](https://blog.csdn.net/luoshengyang/article/details/7867340)
