## Android相关技术总结

### Java篇
##### Soft/Weak/Phantom Reference区别
* Soft reference：内存不足时回收。ReferenceQueue。就算是gc了，如果内存没有达到不足的状态，也不会回收。尽量回收长时间闲置不用的soft reference；
* Weak reference：比起软引用生命周期更短，一旦gc必被清理；
* Phantom reference：不影响对象的生命周期，仅用于判断对象是否将要回收；
* System.GC只是通知JVM回收，JVM何时臊面和回收是不确定的。gc线程的优先级较低；
* System.GC只是通知JVM回收，JVM何时臊面和回收是不确定的。gc线程的优先级较低；

##### Object.hashCode Object.equals
* equals: 默认情况下比较两个对象是否指向同一个内存地址；
* hashCode: 默认情况下对象对应的内存位置；
* toString: 默认情况下类名 + 16进制内存地址；
* Override equals的注意点：自反、对称、传递、一致；
* == 两边都是基本数据类型，判断值是否相等；
* == 两侧都是引用数据类型，判断内存地址是否相等；
* 没有override equals方法前，equals和==含义一致；
* hashCode功能之一：散列；
* override equals前通常要复写hashCode，equals相同的情况下，hashCode必须相同。反之，hashCode相同，不一定equals；
* hashCode多次调用，返回的结果必须相同；
* equals不相同的情况下，hashCode不一定要求不同；

##### ClassLoader
* 加载的目的：将字节码转换成JVM的Class类对象
* 两个重要特性：层次组织结构、代理模式
	* 层次组织结构：每个类的加载都有一个父类加载器。通过这种方式行程一个树状加载结构
	* 代理模式：类加载器既可以自己完成Java类定义，也可以代理给其他类加载器完成。启动类加载和最终定义这个类的类加载器可能不是同一个
* 为相同名称的类创建隔离空间，判断两个类是否相同，不仅判断类的二进制名称，还要判断两个类的定义类加载器，只有加载器一样，才能认为这两个类一样。即使是同样Java字节码，被两个不同的类加载器定义后会得到两个不同的类，相互赋值操作会抛出ClassCastException。
* 类加载过程
	* 装载：查找和导入.class文件
	* 链接（以下步骤可选）：
		* Verification：
			* 确保Java类的二进制表示在结构上是正确的（throw VerifyError）
			* 检查被引用的类型正确性和接入属性的正确性（public、private，final class继承，静态变量）
		* Preparation：
			* 创建Java类中的static域，并将域中的值设置成默认值
			* 对类的成员变量分配空间，但不会进行初始化
			* 可能会给一些有助于程序运行效率提高的数据结构分配空间
		* Resolution（Selected）：
			* 确保被引用的类能被正确的找到
			* 解析的过程可能会导致其他的Java类被加载
			* 为class、interface、method、parameter的符号引用定位到直接引用，完成内存结构布局
	* 初始化：
		* late initial
		* 初始化时主要操作是执行静态代码块和初始化静态域
		* 类初始化会使得其父类初始化
		* 接口的初始化不会引起父接口的初始化
		* 按照源代码的顺序从上到下依次执行
			* 步骤1：如果基类没有初始化，初始化基类
			* 步骤2：有类构造函数，执行构造函数
		* 构造函数将成员变量的初始化和static block提取出来放到一个clinit方法中（clinit不会调用父类的构造方法，因为默认步骤1已经执行了该操作）
		* 初始化过程由JVM保证线程安全
		* 类和接口初始化触发的几种方式
			* 创建类实例
			* 调用类中的静态方法
			* 给类或者接口中的静态域赋值
			* 访问类或接口中非常量的静态域（只初始化实际持有该域的类，不会初始化继承类，哪怕是对它invoke）
			* 在顶层Java类中执行assert
* 双亲委派（类装载器有载入类的需求时，会先请示其Parent使用其搜索路径帮忙载入，如果Parent 找不到,那么才由自己依照自己的搜索路径搜索类）
* 全盘负责：当一个ClassLoader装载一个类时，除非显式调用，否则该类依赖的其他类也是由这个ClassLoader加载的
* Jvm的ClassLoader有三个
	* Bootstrap（java.*, javax.*） -> * C++层的Loader，Java层没有对应类
	* Extension（java.ext.dirs） -> ExtClassLoader
	* System（java.class.path） -> AppClassLoader（应用程序一帮用这个加载）
* 需要给出类的二进制名称，ClassLoader尝试定位或者构造出类的组成定义
	* findLoadedClass: check if the class has already been loaded
	* loadClass: try to load class from parent class loader
	* findClass: if the parent is null, the class loader build-in to the virtual machine is used. Invoke the findClass method to find class. 
	* Use self class loader to loadClass
	* Resolve class（链接类）
* 类加载是线程安全的
	* Method getClassLoadingLock
	* ParallelLoaders：Encapsulates the set of parallel capable loader types（封装了并行的可装载的类型集合）
	* ParallelLoaders会决定一个ClassLoader有没有并行能力
	* 如果没有并行能力，ClassLoader自己作为锁来锁定loadClass方法
	* 如果有并行能力，根据类的名字去parallelLockMap（ConcurrentHashMap）或者创建一个独立的新锁，防止多线程并行加载类导致的线程安全问题
* 装载方式
	* 隐式：new等方式生成对象
	* 显式：通过class.forname等方法加载

##### Java堆新生代、老年代、永久代

### 数据结构篇
##### 基本数据结构大小
* byte：1字节，-128(-2 ^ 8) ~ 127(2 ^ 7 - 1)
* short：2字节，-32768（-2 ^ 15） ~ 32767（2 ^ 15 - 1）
* int: 4字节
* long: 8字节
* float: 4字节
* double: 8字节

##### Integer
* 出于节约内存的考虑，通过IntegerCache做了缓存逻辑，默认缓存[-128, 127]：
```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```
* 缓存通过一个for循环实现。从低到高并创建尽可能多的整数并存储在一个整数数组中。这个缓存会在Integer类第一次被使用的时候被初始化出来。以后，就可以使用缓存中包含的实例对象，而不是创建一个新的实例(在自动装箱的情况下)
* 如果一个变量p的值是：-128至127之间的整数、true 和 false的布尔值、‘\u0000’至 ‘\u007f’之间的字符中时，将p包装成a和b两个对象时，可以直接使用a==b判断a和b的值是否相等。
* 如果不是，则会出现判断错误的情况
```java
Integer integer3 = 300;
Integer integer4 = 300;
integer3 != integer4;
```

##### String
* 一旦一个String对象在内存(堆)中被创建出来，他就无法被修改。特别要注意的是，String类的所有方法都没有改变字符串本身的值，都是返回了一个新的对象。如果你需要一个可修改的字符串，应该使用StringBuffer 或者 StringBuilder。否则会有大量时间浪费在垃圾回收上，因为每次试图修改都有新的string对象被创建出来。

##### List、Set
* List：元素有放入顺序，元素可重复
* Set：元素放入无顺序，元素不可重复
	* TreeSet
		* 二叉树实现（TreeMap.keySet，红黑树，平衡二叉树）
		* 自动排序
		* 不允许放入null
		* 插入时TreeSet要调用compareTo()，所以TreeSet中的元素要实现Comparable
	* HashSet
		* hash表实现（HashMap）
		* 无序
		* 可以放入null，但只有一个

##### ArrayList、LinkedList、Vector
* ArrayList：可变大小数组，每次扩展时，以当前size的150%增长。初始容量比较小，如果可以预估的话可以分配一个较大的初始值，减少空间调整的次数。
* LinkedList：双向链表，add、remove时效率高，get、set时效率低。implement Queue的接口，实现了offer、peek、poll等功能
* Vector：强同步类，如果本身是thread-safe的环境，使用ArrayList性能更好。在扩展空间时增长原始长度的一倍。
* SynchronizedList和Vector都可以在多线程场景中保证数据的线程安全。两者都是List的子类
	* Vector使用同步方式（synchronized修饰的方法）。
	* SynchronizedList使用同步代码块（使用synchronized block 修饰调用的List方法）
	* 扩容方式的区别等同ArrayList和Vector的对比

##### HashTable
* 继承Dictionary
* 方法同步，无多线程问题
* Key、Value均不允许为null
* 初始大小11，增加方式old * 2 + 1
* hash直接使用对象的hashCode
* 遍历方式，Enumeration（只能读不能修改），不支持fail-fast

##### HashMap
* 继承AbstractMap
* 数据结构：数组+链表，Java 8增加红黑二叉树
* 方法非同步，有多线程问题。线程安全的实现方式：
	* Collections.synchronizedCollection
	* 装饰者模式对接口进行封装
	* ConcurrentHashMap（对桶数组做分段锁）
		* Java 8之前：分成16个桶，分段加锁
		* Java 8及以后：红黑树，CAS
* Key可以为null，但只有一个null key。value可以为null
* 默认初始大小16，以2的倍数扩展，且大小必须为2的指数。
* hash重新自己计算
* 遍历方式，iterator，支持fail-fast
* HashMap.get的实现

```java
public V get(Object key) {
    if (key == null)
        return getForNullKey();
    int hash = hash(key.hashCode());
    
    // 在"该hash值对应的链表"上查找"hash等于key"的元素
    for (Entry<K, V> e = table[indexFor(hash, table.length)]; e != null; e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            return e.value;
        }
    }
    
    return null;
}

// h & (length - 1) -> h % (length - 1)
static int indexFor(int h, int length) {
    return h & (length - 1);
}
```

* 为什么HashMap的容量要是2的幂次？
	* indexFor查找key时方便取模运算
	* 方便按位扩容
* 空间换时间，伪随机存取会造成一定的空间浪费，但是增加了查询和增删的速度
* 扩容
	* 默认长度16
	* 默认负载因子e = 0.75
	* 当HashMap内部元素超过 16 * e时扩容
	* 扩容为原先的2倍
* JDK 1.7前，链表结构可能会很长，时间复杂度最差O(n)。使用红黑树替换链表，时间复杂度固定在O(log(n))
* Java 8：数组 + 链表/红黑树（当链表长度超过8）
* 红黑树特征（TreeMap、TreeSet）：
	* 节点是红色或者黑色
	* 根节点、叶节点黑色
	* 红色节点的两个子节点都是黑色（从根节点到任意一个节点的路径上不能出现两个连续的红色节点，但是可以出现两个连续的黑色节点）
	* 从任一节点到其每个叶节点的所有路径包含的黑色节点数量都是一样的

##### 跳表
* 空间换时间
* 类似搜索树
* 随机决定新插入节点的索引层数
* 搜索、插入、删除时间复杂度都是O(log n)
* ConcurrentSkipListMap

### Android篇
##### SurfaceView、TextureView
* SurfaceView
	* 本质上是个View，同时自带一个Surface；
	* 一般只有DecorView（根视图）才是对WNS可见的，在WMS中有一个对应的WindowState，在SurfaceFlinger中有Layer；
	* SurfaceView因为有自己的Surface，也就在WMS有WindowState，在SurfaceFlinger中有Layer；
	* 在App端，SurfaceView是一体的；
	* 在Server端（WMS、SF），Surface的WindowState、Layer与宿主的WindowState、Layer是脱离的；
	* 好处是Surface的渲染是可以放到单独的子线程处理（甚至是Gl线程），不会卡主线程；
	* 缺点是Surface并没有存在于整个View hierarchy中，不受View属性的限制，不能进行类似View的平移、缩放、旋转等View的特性。不能存在于ViewGroup中；
	* 双缓冲策略：SurfaceView更新画面时用到两个Canvas，lockCanvas获取缓存，绘制完成后unlockCanvasAndPost将缓存内容显示；
	* 7.0及以后版本的SurfaceView平移、缩放不再产生黑边；
	* 通过SurfaceHolder与外界进行消息通信；
	![avatar](https://github.com/HayabusaJun/Learning/raw/master/ImageHosting/SurfaceView.png)
* TextureView
	*	不会像SurfaceView一样在WMS中创建单独的Window，而是作为View hierarchy中一个普通的View；
	* 支持所有的View操作和View属性；
	* 必须在硬件加速的窗口中。重载了onDraw方法，将SurfaceTexture中的图像数据作为纹理更新到对应的hardwareLayer；
	* SurfaceTexture.OnFrameAvailableListener用于通知TextureView内容有更新到；
	* SurfaceTextureListener用于告知SurfaceTexture已经准备好了，以便将SurfaceTexture交给内容源；
	* 缺点：必须在硬件加速窗口使用；内存占比比SurfaceView高；5.0以前在主线程渲染，5.0以后在单独的线程渲染；
	* 有1~3帧的延迟，内部缓冲队列使得比SurfaceView消耗更多的内存；
	* 总是使用Gl合成，而SurfaceView可以使用硬件overlay后端，减少内存和电量消耗；

##### 硬件加速
* View.setLayerType
	* LAYER_TYPE_HARDWARE——硬加速（OpenGL）
	* LAYER_TYPE_SOFTWARE——软加速（Bitmap）
	* LAYER_TYPE_NONE——无加速
* 离屏缓冲off-Screen Buffer（Cache），单独启用一块区域绘制View，并保存最近一次的绘制结果。绘制方式通过setType决定，可能是OpenGL、Bitmap，目的是加速重绘；
* 未开启硬件加速的时候，Canvas的绘制是由CPU进行的，Bitmap承载，绘制内容转为实际的像素点。界面重绘时，为了正确的绘制Bitmap，要将所有有变化的View及其相关的View（dirty rect）整体重新绘制；
* 开启硬件加速后，Canvas的绘制转义成GPU操作指令，由GPU实现，绘制内容通过displayList缓存下来。界面重绘时，仅需要改变displayList中有变化的View的display。对属性动画的支持更好（View本身没有变化，仅平移、缩放、旋转）；
* Canvas不同draw方法、Paint、Shader等在不同sdk version上对硬件加速的支持不一致；
* 仅对无invalidate的情况加速；

##### 页面渲染
* 页面渲染时，被绘制的元素最终要转换成像素点才能被显示
* CPU与GPU对比
	* CPU最多的是Cache、DRAM，其次是Control，最后是ALU（Arithmetic Logic Unit，算数逻辑单元）。串行结构
	* GPU最多的是ALU，其次是Cache、DRAM，最后是Controller。并行结构（级联结构）
* DisplayList
	* 基本绘制元素，包含绘制的原始属性（位置、尺寸、角度、Alpha）。对应Canvas的drawXXX()
	* Canvas -> OpenGL -> 驱动 -> GPU
* RenderNode
	* 一个RenderNode包含多个DisplayList
	* 通常一个RenderNode对应一个View，包含View自身及其所有子View的DisplayList
* Android绘制流程（6.0）
	* ViewRootImpl.performTraversals到PhoneWindow.DecorView.drawChild是每次遍历View树的固定过程，首先根据标志位判断是否需要布局并重新布局，然后进行Canvas的创建等操作开始绘制
	* 如果硬件加速不支持或者关闭，则使用软件绘制，生成的Canvas即Canvas.class。isHardwareAccelerated为false
	* 如果支持硬件加速，则生成DisplayListCanvas.class对象。isHardwareAccelerated为true
	* View通过isHardwareAccelerated区分是否硬件加速
	* View.draw -> onDraw -> dispatchDraw -> drawChild 递归路径（Draw路径）
		* 非硬件加速情况下，Canvas.drawXXX实际绘制
		* 硬件加速情况下，Canvas.drawXXX构建DisplayList
	* VIew.updateDisplayListDirty -> dispatchGetDisplayList -> recreateChildDisplay（DisplayList路径）
		* 仅在硬件加速情况下会经过
		* 用于在遍历View树绘制的过程中更新DisplayList属性
		* 快速跳过不需要重建DisplayList的View
	* 硬件加速的情况下，draw()执行结束后DisplayList构建完成，通过ThreadedRender.nSyncAndDrawFrame利用GPU绘制DisplayList到屏幕上
	* 软件绘制刷新流程
		* 默认情况下，View.clipChildren = true，即子View的绘制范围不能超过父View
		* 当View触发invalidate，且没有播放动画、没有触发layout时：
		* 对于全部透明的View，自身设置标记位Dirty，在draw(canvas)方法中，只有这个View被重绘
		* 有透明的View，自身和其父布局都会被标为Dirty。clipChild为true时，脏区被转换成ViewRoot.rect，刷新时层层向下判断，当View与脏区有重合时重绘，且父布局与脏区有重叠才重绘；clipChild为false时，只要View与脏区有重合时就重绘
* 总结
	* CPU擅长逻辑运算，GPU得益于大量的ALU和并行结构，擅长计算
	* 页面由基础元素DisplayList构成，渲染时需要大量的浮点运算
	* 硬件加速时，CPU控制复杂绘制逻辑、构建和更新DisplayList，GPU负责计算、渲染DisplayList
	* 硬件加速情况下，CPU只在重建等情况下更新必要的DisplayList，比如动画
	* 尽量使用简单的DisplayList达到效果，以实现最佳性能

##### onMeasure、onLayout、onDraw
* onMeasure：计算当前View的宽高
* onLayout：处理子View的布局
* onDraw：绘制当前View
* onMeasure -> onLayout -> onDraw
* requestLayout：View重新调用一次layout的过程
* Invalidate：View重新调用一次draw的过程。
* forceLayout：标注View在下一个sync周期重绘，强制进行layout
* View的内容变了，但是大小没变，调用invalidate；内容和大小都变了就先调用requestLayout在调用invalidate
* onMeasure细节
	* MeasureSpec.getMode：获取layout宽/高模式
		* UNSPECIFIED：任意大小
		* AT_MOST：wrap_content
		* EXACTLY：确定的dp或者match_parent
	* MeasureSpec.getSize：获取layout宽/高大小
	* setMeasureDimension：设置measure后确定下来的宽高
	* getSuggestedMinimumWidth、getSuggestedMinimumHeight：建议最小宽高
	* getMeasuredHeight、getHeight
		* getMeasuredHeight：测量高度，可能和真实高度不一样。如果没有requestLayout的话一直是之前的measure结果
		* getHeight：真实高度

##### View.TouchEvent
* onTouchListener优先级高于onTouchEvent，如果onTouchListener返回true，表示Touch已被消费，不会触发onTouchEvent回调
* onClick事件基于onTouchEvent实现，如果onTouchEvent返回true，表示TouchEvent已被消费，不会触发onClick
* TouchEvent
	* A.dispatchTouchEvent -> A.onInterceptTouchEvent -> B.dispatchTouchEvent -> B.onInterceptTouchEvent -> B.onTouchEvent -> A.onTouchEvent
	* ACTION_DOWN由父布局开始逐层向子布局进行dispatchTouchEvent -> onInterceptTouchEvent（只有ViewGroup有）
	* 子布局逐层向父布局回调onTouchEvent
	* 整个过程是自底层向顶层，再向底层的事件传递过程
	* 中间没有消费或者拦截事件（dispatchTouchEvent、onInterceptionTouchEvent、onTouchEvent都返回false），最终父布局将收到onTouchEvent
![avatar](https://github.com/HayabusaJun/Learning/raw/master/ImageHosting/TouchEvent1.png)
	* 在dispatchTouchEvent的过程中，如果有一层View return true，事件的传递将不再继续。同时整个层级中的任何View都不会受到onTouchEvent回调（包括自己）
![avatar](https://github.com/HayabusaJun/Learning/raw/master/ImageHosting/TouchEvent2.png)
	* 如果有一层onInterceptTouchEvent return true，事件将不再向子布局传递，而是从当前布局开始向父布局回调onTouchEvent。该层向下的子布局无任何View的回调
![avatar](https://github.com/HayabusaJun/Learning/raw/master/ImageHosting/TouchEvent3.png)
	* 如果有一层dispatchTouchEvent、onInterceptTouchEvent 都return true，事件的传递将截断，同时会回调本层的onTouchEvent
![avatar](https://github.com/HayabusaJun/Learning/raw/master/ImageHosting/TouchEvent4.png)
	* ViewGroup传递TouchEvent给View
		* 判断点击事件落在View的区域内
		* 子View没有在播放动画

### 设计模式篇
##### 策略模式
* 一个问题多种处理方法
* 封装多种同一类型的操作
* Android属性动画的时间差值器：线性、加速、减速
* 策略的增加导致类越来越多
* 与代理模式区别：
	* 代理模式有代理类，策略模式没有
	* 代理模式将模式的选择封装在代理类中，通过代理类向外提供服务；策略模式通过动态注入选择策略类型

##### 状态模式
* 行为取决于状态
* 运行时状态会改变
* e.g. 登录、未登录情况下用户操作的不同反应

##### 装饰模式（https://www.jianshu.com/p/1dc6e2cc5804）
* 给对象增加附加功能，透明动态的拓展功能
* Android ContextWrapper
* 过程
	* （已有）抽象基类A；
	* （已有）实现类B extends A
	* 装饰抽象类C，继承A，并在构造时持有A的一个实现类引用（比如B）；所有的抽象方法的调用都转发给B
	* 具体装饰类D，继承C，实现时在A的抽象方法实现中可以进行自由扩展
	* D就是对B的一个装饰类，对B的实现进行了拓展

