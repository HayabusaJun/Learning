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

##### Java对象在内存中的结构
* 包含三部分：对象头、实例数据、对齐填充
* Java对象头包含两部分：
	* Class Metadata Address（类型指针）。存储类的元数据的指针，JVM通过该指针找到是哪个类的实例
	* Mark Word（标记字段）。存储对象自身运行时的数据，比如Hash、GC分代年龄、锁状态
* Monitor
	* Mark Word有一个字段指向Monitor对象，Monitor中记录锁持有的线程、等待线程队列信息等。
	* _owner：记录当前持有锁的线程
	* _EntryList：记录所有阻塞等待锁的线程
	* _WaitSet：记录调用了wait()但没有被通知的线程

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

![avatar](https://github.com/HayabusaJun/Learning/raw/master/ImageHosting/JvmObjectLifeGeneration.png)

* 新生代：存放刚new出来的对象，一般占堆空间的1/3。由于频繁创建对象，所以会频繁触发MinorGC进行垃圾回收
	* Eden（默认8 /10）：新对象的出生地。如果创建对象的内存很大则直接分配到老年代。Eden区内存不够就会触发MinorGC，对新生代进行一次GC
	* Survivor From、Survivor To（各自默认1 / 10）复制算法操作区
	* 复制算法：正常使用时只使用Survivor两块中的一块。进行GC时，内存中正在使用的存活对象被复制到其中一块，然后清除正在使用的内存区域。保存存活对象的Survivor就成为了下个阶段的使用区域。下次GC时交换两个Survivor的角色。
	* 为什么这么搞：YoungGen. GC频繁，对象存活概率低，复制算法回收效率高，不容易产生碎片（重点）
	* 缺点：浪费了一块Survivor区域的内存
	* 对象在Survivor区每经历过一次MinorGC，就增长一岁，增长到一定年龄后（默认15岁）就会进入老年代
	* 虽然会触发stop-the-world，但回收速度快

![avatar](https://github.com/HayabusaJun/Learning/raw/master/ImageHosting/JvmObjectOldGeneration.png)

* 老年代：比较稳定，MajorGC不会频繁执行
	* 清理Tenured区
	* MajorGC前通常至少已经有过一次MinorGC
	* 老年代空间不足时会触发MajorGC
	* 无法找到足够大的连续空间分配给新创建的较大对象时，也会触发MajorGC来腾出空间
	* 标记-清除算法：分为标记和清除两个阶段，首先标记所有要回收的对象，标记完成后统一回收所有被标记的对象。为了减少浪费，一般会标记出碎片大小，方面后面直接分配
	* 标记-清除算法缺点
		* 效率不高，标记和清除都是
		* 清除完成后产生大量碎片空间
	* 标记-整理算法：标记出所有需要回收的对象，让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存。需要花费一段时间
	* 老年代被存满后会抛出OOM
* 永久代：主要存放Class和Meta（元数据）
	* 类加载后元数据被放入永久区
	* GC不会在主程序运行的时候清除永久代
	* 永久代也会触发OOM
	* Java 8中永久代被元数据区替代，元数据区不再像永久代一样存在JVM中，而是存在本地内存中，内存区域增大
* Full GC
	* 标记-清除算法
	* Cleaning the entire heap - both Young and Tenurned space

##### Jvm内存区域划分

![avatar](https://github.com/HayabusaJun/Learning/raw/master/ImageHosting/JvmMemory.png)

* 程序计数器（PC寄存器）：记录处理器下一条指令的地址。为了在线程切换后能够恢复到正确的执行位置，每个线程都有一个独立的程序计数器，因此是线程私有的。
* Java虚拟机栈：每个线程都有一个，生命周期与线程相同，存储线程中Java方法调用的状态：局部变量、参数、返回值、运算中间结果。
一个JVM包含多个栈帧，一个栈帧用来存储局部变量表、操作数栈、动态链接、方法出口等信息。当线程调用一个Java方法时，虚拟机压入一个新的栈帧到该线程的Java栈中，当该方法执行完成，这个栈帧就从Java栈中弹出。我们平常所说的栈内存（Stack）指的就是Java虚拟机栈。
	* 如果线程请求分配的栈容量超过Java虚拟机所允许的的最大容量，Java虚拟机会抛出StackOverflowError
	* 如果Java虚拟机栈可以动态扩展（大部分Java虚拟机都可以动态扩展），但是扩展时无法申请到足够的内存，或者在创建新的线程时没有足够的内存去创建对应的Java虚拟机栈，则会抛出OutOfMemoryError异常
* 本地方法栈：用来支持Native方法服务
* Java堆：所有线程共享的运行时内存区域
	* 几乎所有的对象实例都在这里分配内存
	* 堆存储的对象被GC管理
	* 容量可固定可扩展，所使用的内存在物理上不需要连续
	* 没有足够的内存完成实例分配时抛出OOM
* 方法区：所有线程共享的运行时内存区域，存储已经被JVM加载的类的结构信息
	* 运行时常量、字段、方法、静态变量
	* Java堆的逻辑组成部分
	* 不需要连续的物理内存
	* 可选不实现GC
	* 一样可能抛出OOM
* 运行时常量池
	* 方法区的一部分
	* 存放编译器生成的字面量和符号引用

##### ThreadPoolExecutor
* 当ThreadPool.size < coreThreads.size时，即使之前的线程已经出现空闲了，executor也会创建一个core thread
* 当coreThreads.size达到ThreadPool.size，新任务会优先去往WorkQueue
* 当WorkQueue.isFull，但没有到达max值时，创建一个non-core thread
* 当WorkQueue达到max值时，任务被交给RejectedExecutionHandler
* 当non-core thread空闲时间超过keep alive time时，自行销毁
* 当core-thread空闲时间超过allowCoreThreadTimeout时自行销毁

### 锁篇
##### 锁的状态
* 重量级锁：依赖于Mutex锁；
* 无锁→偏锁→轻锁→重锁。锁可升级不可降级；
* 锁的状态切换：
	* 无锁（01）
	* 当有一个线程执行到synchronized block时，升级为偏锁（01）；
	* 当有第二个线程wait时，偏锁升级为轻锁（00）；
	* 等待线程spin several times for unlock；当spin count累计到阈值时，升级为重锁（10），内核态和用户态出现交替。
	* GC（11）；
* 偏向锁：只有一个线程执行synchronized block。线程进入自己已经获得锁的synchronized block时无需CAS的加锁和解锁。无多线程竞争锁；
* 只有多线程开始竞争锁的时候，偏向锁才会被释放。线程不会主动释放偏向锁；
* 轻量级锁：两个线程近乎交替执行synchronized block； 
* 线程间修改堆中数据的几个操作
	* lock/unlock：线程独占状态
	* read：变量从堆读取到线程栈；load：read的变量存为副本；use：副本传递给引擎
	* assign：引擎修改变量；store：修改变量传递给堆；write：写入堆
* lock会清空工作内存（线程栈）上的对象副本，执行前会重新read
* unlock前要有store和write，不允许线程丢弃assign，不允许不assign的情况下store

##### 原子性、可见性、有序性
* Volatile：可见、有序，无原子性。保证每次都从驻村中读取，如果变量被修改，一定会被立刻写回主存。禁止指令重排，不存在阻塞和性能问题
* final：可见性，无需同步，所有内存可以直接访问到final对象
* synchronized（原子、可见、有序）：lock、unlock，串行
* 编译器和处理器不会改变有数据依赖的两个指令的顺序
* long、double：64位数据，JVM为了兼容32位，天然具有读写的原子性

##### 可重入、不可重入
* 可重入：如果一个线程已经获取到一个锁，那么它可以访问所有被这个锁锁住的block
* 不可重入锁（自旋锁）：synchronized
* 可重入锁：ReentrantLock，一个线程可以进入任何一个该线程已经拥有的锁的同步代码块

##### 显式锁、隐式锁
* 显式锁：ReentrantLock、ReentrantReadWriteLock（AQS：AbstractQueuedSynchronizer）
* 隐式锁：sychronized

##### 悲观锁、乐观锁
* 悲观锁（独占锁）：假设一定会发生冲突，因此获取到锁之后会阻塞其他线程使其等待。
	* 优点：简单
	* 缺点：挂起和恢复线程需要转入内核态进行，性能开销大
	* 代表：synchronized
* 乐观锁：假设不会产生冲突，先去尝试执行，失败了再进行其他处理（spin, wait, etc.）
	* 不会阻塞其他线程，也不涉及内核态和用户态的切换
	* 代表：CAS

##### 公平锁、非公平锁
* 公平锁：各线程在加锁前检查有无排队的线程，按照排队的顺序去获取锁
* 非公平锁：加锁前不考虑队列，先去尝试获取锁，获取不到的情况下再去排队

##### 可重入锁、不可重入锁
* 如果一个线程已经获取到一个锁，那么它可以访问所有被这个锁锁住的block

##### synchronized关键字
* 独占锁
* 修饰静态方法时，锁类对象（Object.class）。修饰非静态方法时，锁对象（this）
* 每个对象有一个锁和一个等待队列
* 锁只能被一个对象持有，其他需要锁的线程需要阻塞等待
* 锁被释放后，对象会从队列中取一个线程并唤醒，唤醒的线程是不确定的，所以是非公平锁
* 锁对象，非锁代码。只要是不同的锁对象，相同的synchronize block依然可以并行执行
* synchronized方法不能防止非synchronized方法被同时执行，所以一般在保护变量时，需要在所有访问该变量的方法上加上synchronized
* Java对象Monitor中记录锁持有的线程、等待线程队列信息等。
	* _owner：记录当前持有锁的线程
	* _EntryList：记录所有阻塞等待锁的线程
	* _WaitSet：记录调用了wait()但没有被通知的线程
	* 操作机制
		1. 多个线程竞争锁时，先进入_EntryList，竞争成功的被标记位_owner。其他线程继续在_EntryList中阻塞等待
		2. 如果Owner线程调用了wait()，则Owner释放对象锁，并进入_WaitSet等待被唤醒。Owner被置空，_EntryList的线程再次竞争锁
		3. 如果Owner线程执行完了，便释放锁，Owner被置空，_EntryList的线程再次竞争锁
* JVM对synchronized的优化
	* 自旋锁：可以让等待线程执行一定次数的循环，在循环中获取锁。通过CPU的占用来节省线程切换带来的消耗
	* 锁消除：如果虚拟机运行时发现一段锁住的block中不存在可以被其他线程共享的数据时，就将锁消除
	* 锁粗化：当虚拟机检测到有多个零散的代码块都是用同一个对象加锁时，会将锁拓展到整个操作序列外部。比如StringBuffer.append
	* 轻量级锁：如果整个synchronized的期间没有线程竞争，轻量级锁就不会升级为重量锁，避免互斥量的开销
	* 偏向锁：如果线程获得了一个锁，那么该锁就成为偏锁。获得偏锁的线程无需再次请求该锁

##### CAS（Compare and Swap）
* 从内存位置获取到值，先与旧的预期值比较。如果相等，则将内存位置的值修改为新值。如果不等，说明与其他线程有冲突，不做任何修改。在不阻塞线程的情况下避免并发的冲突，性能好。
* 重试机制：使用一个死循环，CAS成功结束循环。失败就从内存中重新获取旧值，并计算新值，再次尝试CAS。
* AtomicInteger.incrementAndGet
```java
public final int incrementAndGet() {
    for(;;) {
        int current = get();
        int next = current + 1;
        if (compareAndSet(current, next))
            return next;
    }
}
```
* 读取、比较、修改，需要保证比较和修改的原子性，才能保证整个CAS操作的原子性
* CAS只保证操作的原子性，不保证可见性
* 存才ABA问题，解决方式是增加版本号

##### ReentrantLock
* 和synchronized类似（可重入、可见性、解决竞态条件等）
* 支持非阻塞方式获取锁
* 支持响应中断
* 支持限时
* 支持公平锁和非公平锁。内部类通过FairSync和NoFairSync实现，两者均基础Sync，Sync又继承AQS

##### AQS（AbstractQueuedSynchronizer）
* State：使用该锁的数量，volatile修饰，并通过CAS操作
* FIFO双向链表，记录线程信息，每个链代表一个等待线程
* （非公平锁）请求锁时的三种表现
	* 没有其他线程持有锁，直接获取到锁
	* 当前线程已经获取到该锁了，state +1，表示重入申请到锁了。释放锁时 -1。可重入的表现
	* 其他线程已经持有锁了，将自己添加到等待队列（即使是非公平锁，线程进入等待队列后还是得按照顺序来）
* （公平锁）在竞争锁前判断等待队列中有没有其他队列在等待锁
* 挂起等待：
	* 如果该节点的前序节点是HEAD，尝试获取锁，获取到了就移除。获取不到进入下一步
	* 判断前一个节点的waitStatus，如果是SINGAL，将自己挂起。如果是CANCEL，移除前序节点。如果是其他值，标记位SINGAL，进入下个循环（代表一个线程有两次竞争机会，竞争不到就挂起）

##### ReentrantReadWriteLock
* 一个读锁一个写锁，分别对应读操作和写操作
* 只有一个线程可以获得写锁，只有没有任何线程获得任何锁时才能获取到写锁
* 如果有现场持有写锁，任何线程都获得不到任何锁
* 没有线程获取到写锁时，读锁可以被多个线程获得
* 读写锁虽然有两个锁，但实际上只有一个等待队列
	* 获取写锁时，要保证没有任何线程持有锁；
	* 写锁释放后，会唤醒队列第一个线程，可能是读锁和写锁；
	* 获取读锁时，先判断写锁有没有被持有，没有就可以获取成功；
	* 获取读锁成功后，会将队列中等待读锁的线程挨个唤醒，直到遇到等待写锁的线程位置；
	* 释放读锁时，要检查读锁数，如果为0，则唤醒队列中的下一个线程，否则不进行操作。

##### Fail-Fast and Fail-Safe
* Fail-Fast：
	* instantly throw Concurrent Modify Exception。
	* single thread env：structure is modified at any time by any method other than iterator's own remove method
	* Multi thread env: one thread is modifying the structure of the collection while other thread is iterating over it
	* How iterator come to know that internal structure had been changed? Iterator checks “mods” flag whenever it gets next value, the “mods” flag will be changed whenever there’s an structural modification.
* Fail-Safe:
	* Makes copy of the internal data structure and the iterator over copied data structure. Any structure modification done to the iterator affects the copied data structure, origin data structure remains unchanged, no Concurrent Modify Exception throw
	* Overhead of maintaining the copied data structure i.e memory.
	* Fail safe iterator does not guarantee that the data being read is the data currently in the original data structure.
	* Costly maybe, but preclude interference among concurrent threads.
	* Element-changing operations on snapshot iterators (remove(), set(), and add()) are not supported. These methods throw UnsupportedOperationException.

![avatar](https://github.com/HayabusaJun/Learning/raw/master/ImageHosting/FailFastFailSafe.png)

##### CopyOnWrite(COW)
* 当我们往一个容器添加元素的时候，不直接往当前容器添加，而是先将当前容器进行Copy，复制出一个新的容器，然后新的容器里添加元素，添加完元素之后，再将原容器的引用指向新的容器。这样做的好处是我们可以对CopyOnWrite容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。所以CopyOnWrite容器也是一种读写分离的思想，读和写不同的容器。
* 读的时候不需要加锁，如果读的时候有多个线程正在向ArrayList添加数据，读还是会读到旧的数据，因为写的时候不会锁住旧的ArrayList。
* 适合读多写少的场景
* 注意扩容带来的开销，减少添加的次数，多使用addAll。大容量的CopyOnWrite容易造成频繁的Young GC和Full GC
* 数据量较大的时候，建议使用ConcurrentXXX
* 不保证数据一致性

##### 应用：单例模式
* 不使用锁和volatile的懒加载线程安全单例
	* 利用classloader的特性，类加载是线程安全的
	* 饿汉，隐藏了Holder类，避免Holder类加载导致提前被初始化
	* 为什么instance要声明为final？类初始化时，所有的final变量必须初始化完成后才会返回实例。保证其原子性
```java
public class Singleton {
    public static Singleton getInstance() {
        return Holder.instance;
    }
    
    static class Holder {
        public static final Singleton instance = new Singleton();
    }
}
```
* 使用锁和Volatile的懒加载线程安全单例（DCL）
如果不加volatile谁会有问题？instance = new Singleton()，三个操作：
	* 1. allocate
	* 2. construct
	* 3. 使pointer指向刚刚构建好的object

	如果不加volatile，这三个操作可能被指令重排，比如使得3.提前到2.之前执行，而其他线程在此期间判断instance为null，也去初始化Singleton
```java
public class Singleton {
    private volatile static Singleton instance;
    
    public final getSingleton() {
       if (null == instance) {
           synchronized (Singleton.class) {
               if (null == instance) {
                   instance = new Singleton();
               }
           } 
       }
       
       return instance;
    }
}
```
* 使用枚举
	* enum继承Enum类。INSTANCE反编译后会变成一个static final对象，于是当Singleton类被加载的时候，INSTANCE已经被初始化出来了。同时类加载和初始化都是线程安全的
	* 可以避免反序列化，为什么？枚举的反序列化不是通过反射实现。
```java
public enum Singleton {
    Instance;
    public Singleton getInstance() {
        return Instance;
    }
}
```
* 防止Singleton通过序列化的方式构建
```java
public class Singleton implements java.io.Serializable {
    public static Singleton Instance = new Singleton();
    
    protected Singleton() {}
    
    private Object readResolve() {
        return Instance;
    }
}
```


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

### Linux篇
##### 线程调度(http://ohmerhe.com/2018/03/21/android_thread_scheduling/)
* 进程是资源管理最小单位，管理诸如CPU、内存、文件等。并将线程分配到某个CPU上执行
* 线程是执行的最小单位。一个进程至少需要一个线程作为它指令的执行体
	* 核心级线程：更利于并发使用多处理器的资源
	* 用户级线程：更多的考虑上下文切换开销
* Linux中内核线程、轻量级线程、用户线程的区别和联系
	*	内核线程只运行在内核态，不受用户态上下文影响
	*	用户线程是完全建立在用户空间的线程库，用户线程的创建、调度、同步和销毁全部由库函数在用户空间完成，不需要内核的帮助。因此低消耗且高效
	*	轻量级进程（LWP）建立在内核之上并由内核支持的用户线程。每一个轻量级线程都与一个特定的内核线程相关联。由内核管理并像普通进程一样被调度。Android的线程即对应着LWP。
	*	LinuxThreads是用户空间的线程库，一个用户线程对应一个轻量级进程（线程进程1:1模型），一个轻量级进程对应一个特定的内核线程。将线程调度等同于进程调度，调度交由内核完成。线程的创建、同步、销毁由核外线程库完成
* Linux内核不存在真正意义上的线程。所有的执行实体都是任务（task），每一个任务在Linux上都类似于一个单线程的进程，具有内存空间、任务实体、文件资源etc.。但Linux下不同任务间可以选择共用内存空间，这些共用内存空间的任务就成为一个进程下的线程。
* ps | grep $[package name]查找某个应用下的所有线程

* 进程调度：决定哪个进程执行，哪个进程等待；每个进程运行多长时间
* 进程优先级
	* 普通优先级，调度策略SCHED_NORMAL
	* 实时优先级，调度策略SCHED_FIFO（一次机会做完）、SCHED_RR（多次轮转）
	* 实时进程优先级始终高于普通进程。实时进程只会被更高优先级的实时进程抢占
	* 通过设置调度策略可以实现两种优先级（普通/实时优先级）的相互切换
	* 度量方式：
		* 普通进程：nice值
		* 实时进程：RTPRI
* nice值
	* 设定一个普通进程的优先级
	* -20 ~ 19，越大优先级越低，获得CPU的调用机会越少
	* 父进程fork出的子进程nice值继承父进程，父进程renice，子进程不会同时renice
	* 进程nice值被改变后，底层会改变进程所在的cGroup（进程组）
|Java Priority|nice值|
|-|-|
|1|19|
|2|16|
|3|13|
|4|10|
|5|0|
|6|-2|
|7|-4|
|8|-5|
|9|-6|
|10|-8|

* RTPRI
	* 设定一个实时进程的优先级
	* 0 ~ 99，越大优先级越高
	* 实时进程对响应时间要求较高，会抢占一般进程的运行时间

【nice值与RTPRI的优先级关系】

![avatar](https://github.com/HayabusaJun/Learning/raw/master/ImageHosting/prioNiceRtpri.png)

* 动态优先级
	* 在执行阶段调度程序增加或减少进程静态优先级的值
	* 奖励IO消耗进程
	* 惩罚CPU消耗进程
* 时间片：一个进程被抢占前能持续运行的时间。桌面级一般1ms，移动设备一般10ms
* 完全公平调度算法（CFS）：一个进程的优先级越高，而且该进程到当前时刻执行的总时间越小（vruntime），则该进程被调度的可能性越高。
	* 理想状态：所有进程都近似于运行在同一个CPU上
	* vruntime：一个可执行进程到当前时刻执行的总时间。需要通过进程数n归一化，并根据线程优先级加权。vruntime越大，说明进程运行的越久
	* 内核通过红黑树得到vruntime最小的进程
	* vruntime += delta（当前进程运行时间）× NICE_0_LOAD（系统进程默认权重，定值）/ se.weight（当前进程权重，优先级越高，权重越大）
* 进程组（cgroup）
	* 按照某种标准划分的进程。资源获取和限制都是以cgroup为单位实现的
	* 记录、限制进程组可以使用的资源数量。e.g：限制进程组的总内存申请；记录某个进程使用CPU的时间
	* 控制进程组的优先级。e.g：为某个进程组设定特殊的cgroup.share
	* 挂起、恢复进程组。
	* 隔离进程组的进程、网络、文件系统、内存空间等。e.g：不同的进程组有不同的namespace，同时拥有不同的挂载空间
* 层级（hiecarchy）：cgroups可以组织成hierarchical形式——cgroups树。子节点继承父节点的特定属性
* 子系统（subsystem）：资源控制器，必须attach到一个层级上才能生效。层级上所有的cgroups都受这个子系统控制
* Android中的子系统
	* cpu：控制对CPU cgroup的访问
	* cpuset：控制对多核系统中独立CPU和内存节点的访问
	* schedtune：控制进程调度及boost触发
* cgroup.share：cGroup获得CPU时间的相对值。CPU获得时间片比例前台进程：后台进程（/dev/cpuctl/ : /dev/cpuctl/bg_non_interactive） ≈ 95 : 5
* SchedPolicy进程组
```c
/* Keep in sync with THREAD_GROUP_* in frameworks/base/core/java/android/os/Process.java */
typedef enum {
    SP_DEFAULT    = -1,
    SP_BACKGROUND = 0,
    SP_FOREGROUND = 1,
    SP_SYSTEM     = 2,  // can't be used with set_sched_policy()
    SP_AUDIO_APP  = 3,
    SP_AUDIO_SYS  = 4,
    SP_TOP_APP    = 5,
    SP_CNT,
    SP_MAX        = SP_CNT - 1,
    SP_SYSTEM_DEFAULT = SP_FOREGROUND,
} SchedPolicy;
```
* Android进程组（frameworks/base/core/java/android/os/Process.java）与SchedPolicy进程组对应关系

|Android进程组|SchedPolicy进程组|
|-|-|
|THREAD_GROUP_DEFAULT|SP_DEFAULT|
|THREAD_GROUP_BG_NONINTERACTIVE|SP_BACKGROUND|
|THREAD_GROUP_FOREGROUND|SP_FOREGROUND|
|THREAD_GROUP_SYSTEM|SP_SYSTEM|
|THREAD_GROUP_AUDIO_APP|SP_AUDIO_APP|
|THREAD_GROUP_AUDIO_SYS|SP_AUDIO_SYS|
|THREAD_GROUP_TOP_APP|SP_TOP_APP|

* Linux进程优先级oom_score_adj（after v2.6.36）
	* -1000 ~ + 1001，值越小进程优先级越高，内存紧张时越不容易被回收
	* 每个进程的优先级保存在/proc/[pid]/oom_score_adj
	* LowMemoryKiller根据当前内存情况由低到高依次释放进程：
		*	CACHED_APP_MAX_ADJ
		*	CACHED_APP_MIN_ADJ
		*	BACKUP_APP_ADJ
		*	PERCEPTIBLE_APP_ADJ
		*	VISIBLE_APP_ADJ
		*	FOREGROUND_APP_ADJ

![avatar](https://github.com/HayabusaJun/Learning/raw/master/ImageHosting/oomScoreAdj.png)

* Android进程划分
	* 前台进程（Foreground Process）
	* 可见进程（Visible Process）
	* 服务进程（Service Process）
	* 后台进程（Background Process）
	* 空进程（Empty Process）
* Android Process State：ActivityManager重新定义了process_state的划分，并与oom_score_adj做对应

![avater](https://github.com/HayabusaJun/Learning/raw/master/ImageHosting/AndroidProcessState.png)

* Android进程优先级与Android应用组件生命周期变化相关，涉及：Activity、Service、Broadcast、ContentProvider、Process的相关事件。这些事件都过直接或者间接的调用到ActivityManagerService.java中的updateOomAdjLocked方法来更新进程优先级。updateOomAdjLocked方法先通过computeOomAdjLocked计算进程优先级，再通过applyOomAdjLocked应用进程优先级
* Android应用状态发生变化后，会导致进程的oom_score_adj、procState、schedGroup等进程状态的重新计算和设置，从而改变进程的优先级和调度策略，帮助系统更合理的进行资源分配和回收
* Android的线程对应Linux内核中的轻量级进程，主线程不建议手动设置，由系统根据应用状态的变化来调整。但子线程可以自行配置优先级。
* computeOomAdjLocked大致经过以下过程：
	* 空进程判断
	* app.maxAdj <= ProcessList.FOREGROUND_APP_ADJ 的情况
	* 是否有前台优先级
	* 是否有前台服务
	* 是否特殊进程
	* 遍历所有Service的所有连接的Client，根据连接的关系确认客户端进程的优先级
	* 遍历所有ContentProvider的所有连接的client
* Android几种异步线程方式：new Thread()、AsyncTask、HandlerThread、ThreadPoolExecutor、IntentService，除了AsyncTask默认（THREAD_PRIORITY_BACKGROUND）优先级外，其他的均是继承当前线程的priority
* Android线程除了手动设置的优先级外，所在进程因应用状态产生变化，线程优先级也会随之变化

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

##### Activity

![avatar](https://github.com/HayabusaJun/Learning/raw/master/ImageHosting/ActivityFragmentLifeCycle.png)

* Activity启动模式
	* standard：允许相同Activity叠加
	* singleTop：栈顶不允许相同的Activity叠加，启动与栈顶相同的Activity将会触发栈顶Activity的onNewIntent
	* singleTask：如果Activity栈中没有该类型的Activity，启动；如果有，将栈中位于它之上的Activity都弹出，并调用自己的onNewIntent
	* singleInstance：只有一个Activity实例，且该实例运行在一个独立的Activity栈中，这个栈也不会存放其他Activity
* ActivityManagerService
	* 管理Activity运行状态的系统进程
	* 也兼任管理其他组件的运行状态
	* init进程是Android的初始化进程，init进程生成Zygote进程，Android大部分的系统进程和应用进程是通过Zygote进程生成
	* AMS是一个实名Binder（Context.ACTIVITY_SERVICE）
	* 同时还注册meminfo、cpuinfo等
	* AMS构造步骤
		* 初始化必要的Context和Handler
		* 定义广播队列，说明不仅管理Activity，也管理其他组件
		* 管理Service和Provider的对象数组
		* 初始化system下面的一些列文件目录
		* ActivityStackSupervisor：管理Activity Stack和Activity的状态信息
		* ActivityStarter：启动处理类
* Activity何时可见？
	* onResume()？
	* onResume -> handleResumeActivity -> Activity.makeVisible()
* App启动的入口
	* ActivityThread.main()

![avatar](https://github.com/HayabusaJun/Learning/raw/master/ImageHosting/ActivityStartFlow.png)

##### Context
* Activity extends ContextThemeWrapper extends ContextWrapper extends Context
* Application / Service extends ContextWrapper extends Context
* 作用
	* 启动Activity
	* 启动、停止Service
	* 发送广播Intent
	* 注册广播监听
	* 访问Apk资源（Resources、AssetManager）
	* 创建View
	* 访问Package相关信息
	* 权限申请管理
* Context个数：Activity.count + Service.count + 1 (Application)
* 不同Context的功能区别

||Application|Activity|Service|
|-|-|-|-|
|Show Dialogs||√||
|Start Activities||√||
|Layout Inflation||√||
|Start/Bind Services|√|√|√|
|Register BroadcastReceiver|√|√|√|
|Load APK Resources|√|√|√|

##### Handler
* 几个相关对象：Looper、MessageQueue、Message
* 为什么主线程不需要Looper.prepare和Looper.loop（先prepare后loop，不能搞反顺序）
	* ActivityThread.main已经做了这件事情
* sendMessage的不同方法最终都会走到sendMessageAtTime，调用MessageQueue.enqueueMessage，将Message加入到Message链表中
* Looper.loop后，Looper循环从MessageQueue中取Message
	* 如果是Message，回调handleMessage
	* 如果设置了callback（Runnable.run），回调callback
* 延迟是如何实现的
	* MessageQueue.enqueueMessage(Message, long)，根据Message.when将Message按升序插入到MessageQueue中
	* enqueue后，如果链表首部的Message.when > 0，计算delay time，Looper阻塞。等待下一个Message enqueue或者delay时间到的时候唤醒

##### Binder IPC
https://www.jianshu.com/p/429a1ff3560c
https://blog.csdn.net/universus/article/details/6211589
* 基于安全、性能、稳定性的考量
* 性能
	* Socket作为通用接口，传输效率太低，开销大，主要用于网络，本地低速通信。两次拷贝；
	* 消息队列基于”存储-转发”，数据先从发送方缓冲区拷贝到内核缓冲区，再从内核缓冲区拷贝到接收方缓冲区。两次拷贝；
	* 共享内存无数据拷贝，但控制较复杂；
	* Binder仅需一次数据拷贝，性能上仅次于共享内存；
* 稳定性
	* 基于C/S架构，职责清晰；
* 安全性
	Linux IPC安全问题
	* 没有底层安全措施；
	* 无法获取到对方可信的UID/PID，无法鉴别对方身份；Binder内核层分配UID；
	* 接入点开发，理论上任意程序都可以建立连接；
* Linux IPC概念
	* 进程间隔离，无法直接访问
	* 进程空间划分：用户空间、内核空间。32位Linux系统中，内核空间1G，用户空间3G
	* 系统调用：用户态（3级）、内核态（0级）。用户空间想要访问内核资源（网络、文件系统），需要借助系统调用实现，用户空间访问系统空间的唯一方法，保证所系统有资源的访问都是在内核态下完成的，避免越权访问。
	* 发送方并通过系统调用进入内核态。内核程序在内核空间分配内存，开辟发送缓冲区，并将数据从用户空间拷贝到内核的缓冲区（copy_from_user）。接收方在用户空间分配接收缓冲区，内核通过copy_to_user将缓冲区的数据拷贝到接收方的接收缓冲区。
		* 两次拷贝数据
		* 缓冲区大小难以计算，不是浪费时间就是浪费空间
* Binder
	* Binder不是Linux内核的一部分
	* 动态内核可加载模块（Loadable Kernel Module）：可单独编译，不可单独运行，需要被链接到内核，成为内核的一部分才能运行。
	* Android系统通过这个动态加载的模块运行在内核空间，用户空间通过这个空间作为桥梁进行通信。
	* Binder Driver
* 一次拷贝的实现：mmap 内存映射
	* 用户空间的一片内存区域与内核层的一片内存区域相互映射，双方对各自区域的写操作都能被对方感知
	* 通常是作用在有物理介质的文件系统上的
		* 常规文件读取操作需要经历：磁盘 -> 内核空间 -> 用户空间。通过mmap将磁盘空间与用户空间做映射，减少拷贝次数
* 一次Binder IPC过程
	* Binder Driver在内核创建一块数据接收缓冲区
	* 内核空间开辟一块内核缓存区，并与Binder的数据接收缓冲区建立mmap
	* 数据接收缓冲区与接收进程用户空间建立mmap
	* 发送方的数据被copy_from_user()映射到内核缓冲区后，因为两个mmap的作用，直接作用到了接收进程的用户空间
* C/S/ServiceManager/Binder Driver
	* C/S/ServiceManager均在用户空间，Binder Driver在内核空间
	* C/S/ServiceManager均通过系统调用open、mmap、ioctl访问设备文件/dev/binder，从而实现与Binder Driver的交互，实现Binder IPC

![avatar](https://github.com/HayabusaJun/Learning/raw/master/ImageHosting/BinderStructure.png)

* Binder Driver
	* 负责IPC的建立，进程间传递，引用计数管理，数据包的传递交互等
* ServiceManager
	* 将字符形式的Binder名字转换成Client对Binder的引用，使得Client通过Binder名字获得Binder的引用。
	* 注册了名字的Binder叫实体Binder
	* S创建Binder，并连同自己的名字发送给ServiceManager，向ServiceManager注册自己。Binder Driver为S的Binder创建内核中的实体节点以及ServiceManager对实体的引用，打包发送给ServiceManager。ServiceManager将名字和打包信息填入索引表
	* ServiceManager与其他进程通信时一样使用Binder IPC，充当S。SM提供的Binder较特殊，没有名字也无需注册，进程使用BINDER_SET_CONTEXT_MGR将自己注册为SM。SM的Binder的引用也始终为0。
	* C向SM请求访问S名字的Binder，SM将查找到的S引用返回
	* 此时S的Binder引用有两个，一个在ServiceManager中，一个在Client中

![avatar](https://github.com/HayabusaJun/Learning/blob/master/ImageHosting/BinderProcedure.png)

* A、B进程间对象的引用——代理模式
	* A跨进程获取到B的并不是对象本身，而是B对象的代理，只有接口，没有具体实现。A这些方法调用后会将参数传递给Binder Driver
	* 当Binder Driver接收到A调用信息后，通过proxyObject查询到对象实际存在的B进程和对象方法，通知B进程的对象调用实际的方法，并要求B进程将调用结果发送给自己
* AIDL各文件的意义
	* IBinder：接口，代表能实现IPC的能力，继承了IBinder接口并实现就能进行IPC
	* IInterface：S端能提供什么服务，与.aidl中定义的接口一致
	* Binder：Java层的Binder类，代表Binder本地对象。BinderProxy是Binder的内部类，代表Binder的本地代理。两者均基础IBinder
	* Stub：S端，静态内部类
		* 继承IBinder和IInterface。在IInterface中完成承诺的实现
		* asInterface：判断C/S是否处于同一进程，属于返回本地Binder对象，不属于创建Proxy对象
		* onTransact：反序列化调用方法和入参，进行实际的调用并返回ServiceManager调用结果
	* Proxy：C端，
		* 实现IInterface接口，序列化调用方法和入参
		* 调用remote.transact()后，通过系统调用进入内核态。执行方法的C端线程挂起，等待返回值唤醒

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

