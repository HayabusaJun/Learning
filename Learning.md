## Android相关技术总结

### Java篇
##### Soft/Weak/PhantomReference区别
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
