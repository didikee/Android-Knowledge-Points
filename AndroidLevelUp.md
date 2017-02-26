# Android 知识进阶

[TOC]

## 内存管理

> Java 使用的是 **有向图**来判断一个对象是否可以被回收.
> **有向图: **根顶点可达的对象都是有效对象，GC将不回收这些对象.如果某个对象 (连通子图)与这个根顶点不可达，那么我们认为这个(这些)对象不再被引用，可以被 GC 回收.

常见的内存管理方式:

- **有向图**:优点是管理内存的精度很高，但是效率较低.
- **计数器**:COM模型采用计数器方式管理构件，它与有向图相比，精度行低(很难处理循环引用的问题)，但执行效率很高


## 内存泄漏

#### Java内存泄漏引起的原因
内存泄漏是指无用对象（不再使用的对象）持续占有内存或无用对象的内存得不到及时释放，从而造成内存空间的浪费称为内存泄漏。

#### Java内存泄漏的根本原因是
长生命周期的对象持有短生命周期对象的引用就很可能发生内存泄漏，尽管短生命周期对象已经不再需要，但是因为长生命周期持有它的引用而导致不能被回收，这就是Java中内存泄漏的发生场景。

#### 内存泄漏的场景

1. 静态集合类引起内存泄漏
2. 当集合里面的对象属性被修改后，再调用remove()方法时不起作用.
3. 监听器:addXXXListener(),但没有移除,从而增加了内存泄漏的机会.
4. 各种连接:数据库cursor未关闭,IO流未关闭,自定义属性没有 `recycle()`等.
5. 内部类和外部模块的引用
6. 不正确的单例模式



## ART和Dalvik区别

> Art上应用启动快，运行快，但是耗费更多存储空间，安装时间长，总的来说ART的功效就是"空间换时间"。

ART代表Android Runtime,其处理应用程序执行的方式完全不同于Dalvik，Dalvik是依靠一个Just-In-Time(JIT)编译器去解释字节码。开发者编译后的应用代码需要通过一个解释器在用户的设备上运行，这一机制并不高效，但让应用能更容易在不同硬件和架构上运行。**ART则完全改变了这套做法，在应用安装的时候就预编译字节码到机器语言**，这一机制叫Ahead-Of-Time(AOT)编译。在移除解释代码这一过程后，应用程序执行将更有效率，启动更快。

ART优点：`系统性能的显著提升 应用启动更快、运行更快、体验更流畅、触感反馈更及时 更长的电池续航能力 支持更低的硬件`
ART缺点：`更大的存储空间占用，可能会增加10%-20% 更长的应用安装时间`

## 安卓手机开机过程

* **BootLoder引导,然后加载Linux内核.**
* 0号进程init启动.加载init.rc配置文件,配置文件有个命令启动了zygote进程
* zygote开始fork出SystemServer进程
* SystemServer加载各种JNI库,然后init1,init2方法,init2方法中开启了新线程ServerThread.
* 在SystemServer中会创建一个socket客户端，后续AMS（ActivityManagerService）会通过此客户端和zygote通信
* **ServerThread的run方法中开启了AMS,还孵化新进程ServiceManager,加载注册了一溜的服务,最后一句话进入loop 死循环**
* run方法的SystemReady调用resumeTopActivityLocked打开锁屏界面

> 人生有两件事,一件事是做自己想做的事,另一件事是把自己想做的事做好.

## App启动流程
## ActivityManagerService介绍
## WindowManagerService介绍
## ServiceManagerService介绍
## Activity 启动流程(非生命周期)


## 如何计算一张 w*h尺寸的图片在内存中的占用大小?
| 模式|描述|占用大小(每像素)|
|:--|:--|:---------:|
|ALPHA_8|此时图片只有alpha值，没有RGB值，一个像素占用一个字节|**1**|
|ARGB_4444|一个像素占用2个字节，alpha(A)值，Red（R）值，Green(G)值，Blue（B）值各占4个bites,共16bites,即2个字节|**2**|
|ARGB_8888|一个像素占用4个字节，alpha(A)值，Red（R）值，Green(G)值，Blue（B）值各占8个bites,共32bites,即4个字节.这是一种高质量的图片格式，电脑上普通采用的格式。它也是Android手机上一个BitMap的默认格式|**4**|
|RGB_565 |一个像素占用2个字节，没有alpha(A)值，即不支持透明和半透明，Red（R）值占5个bites ，Green(G)值占6个bites  ，Blue（B）值占5个bites,共16bites,即2个字节.对于没有透明和半透明颜色的图片来说，该格式的图片能够达到比较的呈现效果，相对于ARGB_8888来说也能减少一半的内存开销。因此它是一个不错的选择。另外我们通过android.content.res.Resources来取得一个张图片时，它也是以该格式来构建BitMap的.从Android4.0开始，该选项无效。即使设置为该值，系统任然会采用 ARGB_8888来构造图片|**2**|

**最终得出一张图片占用的内存大小约为:` w*h*[对应的格式]/1024`,`单位(KB)`**

**bitmap 的回收:**

虽然Bitmap在被回收时可以通过BitmapFinalizer来回收内存。但是调用recycle()是一个良好的习惯
**在Android4.0之前**，Bitmap的内存是分配在Native堆中，调用recycle()可以立即释放Native内存。
**从Android4.0开始**，Bitmap的内存就是分配在dalvik堆中，即JAVA堆中的，调用recycle()并不能立即释放Native内存。但是调用recycle()也是一个良好的习惯。
```
// 先判断是否已经回收
if(bitmap != null && !bitmap.isRecycled()){
    // 回收并且置为null
    bitmap.recycle();
    bitmap = null;
}
System.gc();
```
## 线程和进程的区别?


#### [拓展]Android 有几种进程?
1. 前台进程：即与用户正在交互的Activity或者Activity用到的Service等，如果系统内存不足时前台进程是最后被杀死的
2. 可见进程：可以是处于暂停状态(onPause)的Activity或者绑定在其上的Service，即被用户可见，但由于失去了焦点而不能与用户交互
3. 服务进程：其中运行着使用startService方法启动的Service，虽然不被用户可见，但是却是用户关心的，例如用户正在非音乐界面听的音乐或者正在非下载页面自己下载的文件等；当系统要空间运行前两者进程时才会被终止
4. 后台进程：其中运行着执行onStop方法而停止的程序，但是却不是用户当前关心的，例如后台挂着的QQ，这样的进程系统一旦没了有内存就首先被杀死
5. 空进程：不包含任何应用程序的程序组件的进程，这样的进程系统是一般不会让他存在的


#### 如何避免后台进程被杀死?

- 调用startForegound，让你的Service所在的线程成为前台进程
- Service的onStartCommond返回START_STICKY或START_REDELIVER_INTENT
- Service的onDestroy里面重新启动自己


## 设计模式[常见]

### 单例模式
1. 饿汉式：  private static Singleton uniqueInstance = new Singleton();
2. 懒汉式:  private static Singleton uniqueInstance = null;
3. 静态内部类 **[推荐]**
4. 单元素的枚举 **[推荐]**

**饿汉式 & 懒汉式**
- 懒汉式是典型的时间换空间
- 饿汉式是典型的空间换时间

不加同步的懒汉式是线程不安全的.加了又有损效率,所以 `饿汉式 & 懒汉式` 都不是单例模式的好选择.

### Builder模式
> 将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示。

极力推荐查看**`Android源码中的模式实现`**-->**`AlertDialog.Builder`**

**优点与缺点**
优点

- 良好的封装性， 使用建造者模式可以使客户端不必知道产品内部组成的细节；
- 建造者独立，容易扩展；
- 在对象创建过程中会使用到系统中的一些其它对象，这些对象在产品对象的创建过程中不易得到。

缺点

- 会产生多余的Builder对象以及Director对象，消耗内存；
- 对象的构建过程暴露。


### 原型模式

> 定义:**用原型实例指定创建对象的种类，并通过拷贝这些原型创建新的对象。**

**模式的使用场景**

- 类初始化需要消化非常多的资源，这个资源包括数据、硬件资源等，通过原型拷贝避免这些消耗；
- 通过 new 产生一个对象需要非常繁琐的数据准备或访问权限，则可以使用原型模式；
- 一个对象需要提供给其他对象访问，而且各个调用者可能都需要修改其值时，可以考虑使用原型模式拷贝多个对象供调用者使用，即保护性拷贝。
**`Android源码中的模式实现`**-->**`Intent`**

```
Uri uri = Uri.parse("smsto:0800000123");    
    Intent shareIntent = new Intent(Intent.ACTION_SENDTO, uri);    
    shareIntent.putExtra("sms_body", "The SMS text");    

    Intent intent = (Intent)shareIntent.clone() ;//通过clone,减少new对象后的赋值操作
    startActivity(intent);
```

**优点与缺点**

**优点**
原型模式是在内存二进制流的拷贝，要比直接 new 一个对象性能好很多，特别是要在一个循环体内产生大量的对象时，原型模式可以更好地体现其优点。
**缺点**
这既是它的优点也是缺点，直接在内存中拷贝，构造函数是不会执行的，在实际开发当中应该注意这个潜在的问题。优点就是减少了约束，缺点也是减少了约束，需要大家在实际应用时考虑。


### 工厂模式

**`Android源码中的模式实现`**-->**`BitmapFactory`**

**优点 & 缺点**
**优点**: 帮助封装,解耦
**缺点**: 可能增加客户端的复杂度

### 观察者模式

**`Android源码中的模式实现`**-->**`Listview && RecyclerView && Broadcast(广播)`**

首先在Android中，我们往ListView添加数据后，都会调用Adapter的notifyDataChanged()方法，其中使用了观察者模式。

当ListView的数据发生变化时，调用Adapter的notifyDataSetChanged函数，这个函数又会调用DataSetObservable的notifyChanged函数，这个函数会调用所有观察者(AdapterDataSetObserver)的onChanged方法，在onChanged函数中又会调用ListView重新布局的函数使得ListView刷新界面。

**Android中应用程序发送广播的过程：**

1. 通过sendBroadcast把一个广播通过Binder发送给ActivityManagerService，ActivityManagerService根据这个广播的Action类型找到相应的广播接收器，然后把这个广播放进自己的消息队列中，就完成第一阶段对这个广播的异步分发。
2. ActivityManagerService在消息循环中处理这个广播，并通过Binder机制把这个广播分发给注册的ReceiverDispatcher，ReceiverDispatcher把这个广播放进MainActivity所在线程的消息队列中，就完成第二阶段对这个广播的异步分发：
3. ReceiverDispatcher的内部类Args在MainActivity所在的线程消息循环中处理这个广播，最终是将这个广播分发给所注册的BroadcastReceiver实例的onReceive函数进行处理：

### 代理模式
**`Android源码中的模式实现`**-->**`Retrofit 包装 OkHttp`**
代理模式是对象的结构模式。代理模式给某一个对象提供一个代理对象，并由代理对象控制对原对象的引用.

**Android Xposed 的 `Hook`技术就是应用的 `代理模式`**

**角色介绍**

**抽象对象角色**：声明了目标对象和代理对象的共同接口，这样一来在任何可以使用目标对象的地方都可以使用代理对象。
**目标对象角色**：定义了代理对象所代表的目标对象。
**代理对象角色**：代理对象内部含有目标对象的引用，从而可以在任何时候操作目标对象；代理对象提供一个与目标对象相同的接口，以便可以在任何时候替代目标对象。代理对象通常在客户端调用传递给目标对象之前或之后，执行某个操作，而不是单纯地将调用传递给目标对象。

**优点:**给对象增加了本地化的扩展性，增加了存取操作控制
**缺点:**会产生多余的代理类

### 适配器模式
**`Android源码中的模式实现`**-->**`Adapter`**
**优点**：更好的复用性,更好的扩展性
**缺点**：过多的使用适配器，会让系统非常零乱，不容易整体进行把握。
