---
title: CoreAnimation-CALayer(一)
date: 2016-01-03 00:55:20
category: iOSTips
tags: CoreAnimation
---
## CoreAnimation-CALayer(一)
### 初识CoreAnimation

我们一般知道[CoreAnimation](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/CoreAnimationBasics/CoreAnimationBasics.html#//apple_ref/doc/uid/TP40004514-CH2-SW3)是一种非常强大动画技术,其实CoreAnimation还是iOS和OSX中图形渲染的核心,我们在进行图形界面开发的时候无时无刻不是在使用它,使用CoreAnimation我们只需要进行简单的配置,系统就会帮我们进行每一帧的绘制从而达到流畅绚丽的动画效果.
![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/ca_architecture_2x.png)
上面是官方文档给出的iOS图形渲染结构图

* Craphics Hardware : GPU等负责渲染的硬件,操作系统封装了硬件层,提供了统一的绘制接口,这里会有一系列的针对不同硬件的封装接口.
* OpenGL ES 层统一封装了绘图硬件的接口,我们使用OpenGL的统一接口就可以控制任意的绘图硬件了,但是它是C语言实现的
* CoreAnimation 为了更友好的使用OpenGL引擎,在iOS/OSX中苹果针对OpenGL进行了封装,这就是CoreAnimation

上面是对iOS绘图系统做了一个简单的介绍,下面我们进行深入的学习
预计的CoreAnimation的系列文章包括:

* CALayer
* CoreAnimation类的结构
* 运用CoreAnimation进行动画

### CALayer
CoreAnimation本身是作用在layer上面并不是UIView上,所以我们有必要首先介绍一下CALayer.
 `CALayer`是什么?
  CALayer用来绘制我们能看到的一切!官网介绍: `Layers Provide the Basis for Drawing and Animations`（Layers是3D空间中的2维平面,是我们使用CoreAnimation进行绘图的核心,layer管理着视图的位置,内容,和其他可视化的属性,通过`bitmap`来管理这些信息,bitmap本身可以是视图绘制的结果也可以是你指定的镜像.基于这个原因,calayer可以看做是app中的model层,因为它就是来**管理绘图信息**的,明白这一点对理解以后的动画操作帮助很大.
  layer本身并不做实际的绘制工作,它只是收集app提供的内容,并且将这些内容缓存到bitmap中,当我们改变layer的属性时仅仅是改变了与layer相关的状态信息,如果这个属性是animateable的那么CoreAnimation系统会在合适的时机将layer的bitmap以及状态信息提交到硬件绘制系统(GPU).
![](http://img.blog.csdn.net/20150110094646249?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSGVsbG9fSHdj/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
之所以layer有一个与之相关的静态bitmap,所以基于layer的动画不同于基于view的动画,通过改变UIView的动画属性往往需要重复触发`- drawRect:`方法进行重绘,这个方法是比较昂贵的,因为它使用`CUP`计算,并且发生在主线程中!
如果使用coreAnimation,我们虽然进行大量的animated属性的改变,这些改变只会将layer的bitmap状态变化,当下一次屏幕刷新的时候才将bitmap中储存的最新的信息提交到CPU中,储存的过程是后台线程进行的.

#### CALayer的基本使用

#### 常见属性

##### animatable 属性
我们点进`CALayer`可以看到很多属性的注释后面都有一个`animatable`标示,这个标示代表修改这个属性附带有动画效果.具体一下两个特征

* 直接修改非`rootLayer` 的可动画属性可以实现隐式动画
* 在CoreAnimation中作为keyPath来进行修改产生动画效果.

利用CALayer我们可以实现一些常见的显示效果,例如阴影,圆角,背景颜色等

```c
ViewController.m
- (void)viewDidLoad {
    [super viewDidLoad];
    CALayer *redLayer = [CALayer layer];
    redLayer.frame = CGRectMake(10, 50, 100, 100);
    redLayer.backgroundColor= [UIColor redColor].CGColor;
    redLayer.cornerRadius = 20.0;//圆角
    redLayer.shadowColor = [UIColor blackColor].CGColor;//阴影颜色
    redLayer.shadowOpacity = 0.5;//阴影透明度
    redLayer.shadowOffset = CGSizeMake(3.0, 3.0);//阴影的偏移量
    redLayer.borderColor = [UIColor whiteColor].CGColor;//边界颜色
    redLayer.borderWidth = 2;//边界宽度
    [self.view.layer addSublayer:redLayer];//将layer添加到view的layer图层上.
}
```
![](http://o7vzr7y09.bkt.clouddn.com/Simulator%20Screen%20Shot%202017%E5%B9%B44%E6%9C%8828%E6%97%A5%20%E4%B8%8B%E5%8D%883.24.20.png)

##### layer的内容
layers 管理着app提供的显示内容,所有要展示的可视图形数据(成为内容contents)共同构成了layer的`bitmap`.
如果你手动初始化layer,你可以通过以下三种方式为bitmap提供内容content

* 将image对象赋值给`layer.contents`属性(适用于layer的内容很少或者从来不会改变的)
* 给layer设置一个代理`(id<CALayerDelegate>)`对象,用代理对象来进行绘制内容,适用于layer的内容可能周期发生变化或者需要使用另一个对象进行绘制工作的模式.(UIView就是这种模式.)
* 定义CALayer的子类,重写绘制相关的方法从而为layer提供内容

###### 设置contents
***contents***代表了layer呈现的内容对象,是一个id类型,但是我们需要将OC对象转换成`CoreGraphics`类型的值在设置才有效果.例如我们可以通过设置layer的`contents`属性来代替UIImageView来呈现图片.

```
 _bview = [[UIView alloc] initWithFrame:CGRectMake(100, 100, 100, 100)];
 //直接设置image给contents是不行的
    _bview.layer.contents = (__bridge id)image.CGImage;
    _bview.layer.contentsGravity = kCAGravityResizeAspectFill;
    [self.view addSubview:_bview];
```
![](http://o7vzr7y09.bkt.clouddn.com/Simulator%20Screen%20Shot%202017%E5%B9%B45%E6%9C%882%E6%97%A5%20%E4%B8%8B%E5%8D%885.59.21.png)

###### 设置delegate

```c
- (void)displayLayer:(CALayer *)layer {
    //在这个方法里面可以通过特定的条件变量或者提供第三方的类给layer设置内容
    if (self.displayYesImage) {
        // Display the Yes image
        theLayer.contents = [someHelperObject loadStateYesImage];
    }
    else {
        // Display the No image
        theLayer.contents = [someHelperObject loadStateNoImage];
    }
}

//如果上面的方法没有给layer设置内容,就会来到这个方法,在这个方法里面直接在ctx上下文中进行绘制工作

- (void)drawLayer:(CALayer *)layer inContext:(CGContextRef)ctx {
    CGMutablePathRef thePath = CGPathCreateMutable();
    
    CGPathMoveToPoint(thePath,NULL,15.0f,15.f);
    CGPathAddCurveToPoint(thePath,
                          NULL,
                          15.f,250.0f,
                          295.0f,250.0f,
                          295.0f,15.0f);
    
    CGContextBeginPath(theContext);
    CGContextAddPath(theContext, thePath);
    
    CGContextSetLineWidth(theContext, 5);
    CGContextStrokePath(theContext);
    
    // Release the path
    CFRelease(thePath);
}
```
###### 子类化layer
如果我们自定义了layer那么可以通过复写绘制方法进行绘制,通常我们不会这样做.但是这样layer的内容就是自己决定,例如`CATileLayer`类管理着一张图片,它可以将图片分割成若干小的部分,从而进行分开展示,因为只有layer知道那个碎片在特定的时间是否该展示,重写的步骤

* 重写 `display`方法直接给layer设置contents
* 重写`drawInContext:`根据上下文进行绘制.

选择具体重写哪个方法要看你想怎样控制绘制任务,重写`display`方法意味着你要 自己负责创建`CGImageRef`并且设置给contents属性,如果你仅仅是绘制的话那么应该重写`drawInContext:`.		

##### contentsGravity
`contentsGravity`是content在layer上的呈现方式,修改UIView的`contentMode`属性其实就是在操作`contentsGravity`
对应的枚举值为

```
CA_EXTERN NSString * const kCAGravityCenter
CA_EXTERN NSString * const kCAGravityTop
CA_EXTERN NSString * const kCAGravityBottom
CA_EXTERN NSString * const kCAGravityLeft
CA_EXTERN NSString * const kCAGravityRight
CA_EXTERN NSString * const kCAGravityTopLeft
CA_EXTERN NSString * const kCAGravityTopRight
CA_EXTERN NSString * const kCAGravityBottomLeft
CA_EXTERN NSString * const kCAGravityBottomRight
CA_EXTERN NSString * const kCAGravityResize
CA_EXTERN NSString * const kCAGravityResizeAspect -> `UIViewContentModeScaleAspectFit`
CA_EXTERN NSString * const kCAGravityResizeAspectFill -> `UIViewContentModeScaleAspectFill`
```
###### subLayers
CALayer 支持添加子layer,并且支持调整子layer的层级顺序,对应的api是

```c
- (void)addSublayer:(CALayer *)layer;
- (void)insertSublayer:(CALayer *)layer atIndex:(unsigned)idx;
- (void)insertSublayer:(CALayer *)layer below:(nullable CALayer *)sibling;
- (void)insertSublayer:(CALayer *)layer above:(nullable CALayer *)sibling;
- (void)replaceSublayer:(CALayer *)layer with:(CALayer *)layer2;
```
CALayer 的子类
![](http://o7vzr7y09.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-28%20%E4%B8%8B%E5%8D%883.29.51.png)
那么他们是做什么的呢,后面我们可能会用到其中的一个或几个来实现特殊的显示效果.

CAEmitterLayer|发散层，可以实现粒子效果
-------|-------
CAGradientLayer|梯度层，颜色渐变
CAEAGLayer|用OpenGL ES绘制的层
CAReplicationLayer|拷贝层可以用来复制sublayer
CAScrollLayer|可滑动的层
CAShapeLayer|绘制立体的贝塞尔曲线
CATextLayer|可以绘制AttributeString
CATiledLayer|用来管理一副可以被分割的大图
CATransformLayer|用来渲染3D layer的层次结构

###### frame,position,anchorPoint,bounds
frame 和 bounds
frame,bounds在一开始学习UI的时候就接触了这两个概念,但是可能很多人只知道frame是相对父视图坐标系统(0, 0)点的偏移量(这里不考虑size),bounds是相对于自身坐标系的(0, 0)点位置
![](http://img.my.csdn.net/uploads/201303/24/1364058232_8785.jpg)
修改frame的origin会产生位移,那么是否知道修改bounds会带来什么影响呢?其实坐标系统的关键就是要知道它的**原点**(0, 0)点在什么位置.

```c
	UIView *redView = [[UIView alloc] initWithFrame:CGRectMake(20, 20, 200, 200)];
    redView.backgroundColor = [UIColor redColor];
    redView.bounds = CGRectMake(- 20, - 20, 200, 200);
    [self.view addSubview:redView];
    
    UIView *subView = [[UIView alloc] initWithFrame:CGRectMake(20, 20, 100, 100)];
    subView.backgroundColor = [UIColor blueColor];
    [redView addSubview:subView];
```
![](http://o7vzr7y09.bkt.clouddn.com/Simulator%20Screen%20Shot%202017%E5%B9%B45%E6%9C%883%E6%97%A5%20%E4%B8%8A%E5%8D%8812.33.02.png)
这里将redView的bounds设置为(- 20, - 20, 200, 200)以后那么redView左上角的位置就是(-20,-20),那么它的原点(0,0)就是相对于左上角(10, 10)的位置,所以subView会显示在距离redView左上角(40, 40)的位置.
所以设置bounds会影响子视图的位置.

position: 位置点, 定义了锚点(anchorPoint)相对父layer上的位置.
anchorPoint: 锚点,定义了layer上的哪个点在position上.x,y取值都为0~1之间的数,默认(0.5-0.5)

```c
	_layer = [CALayer layer];
    _layer.frame = CGRectMake(10, 10, 100, 100);
    _layer.backgroundColor = [UIColor redColor].CGColor;
    [self.view.layer addSublayer:_layer];
    NSLog(@"%@",_layer);
    //position = CGPoint (60 60)
    //bounds = CGRect (0 0; 100 100)
    //anchorPoint = CGPoint (0.5, 0.5)
```
下面贴两张官方的图,下图展示了虽然位置是相同的,但是因为`anchorPoint`不同导致的position的不同
![](http://img.blog.csdn.net/20150110130534295?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSGVsbG9fSHdj/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  ![](http://img.blog.csdn.net/20150110130534295?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSGVsbG9fSHdj/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
注意思考为什么下面只是代码添加的顺序不同就会产生不同的效果

```c
	_layer = [CALayer layer];
    _layer.anchorPoint = CGPointMake(0, 0);
    _layer.frame = CGRectMake(10, 10, 100, 100);
    _layer.backgroundColor = [UIColor redColor].CGColor;
    [self.view.layer addSublayer:_layer];
```
![](http://o7vzr7y09.bkt.clouddn.com/Simulator%20Screen%20Shot%202017%E5%B9%B45%E6%9C%883%E6%97%A5%20%E4%B8%8A%E5%8D%8812.48.55.png)

```c
	_layer = [CALayer layer];
    _layer.frame = CGRectMake(10, 10, 100, 100);
    _layer.anchorPoint = CGPointMake(0, 0);
    _layer.backgroundColor = [UIColor redColor].CGColor;
    [self.view.layer addSublayer:_layer];
```
![](http://o7vzr7y09.bkt.clouddn.com/sss.png)

###### zPosition
在Z轴(垂直于屏幕坐标轴)上的值.默认是0,所以我们通过设置zPosition来调整子layer的显示,晚添加的layer也可能位于早添加的layer的下面

###### presentationLayer 和 modelLayer
在CALayer内部，它控制着两个属性：presentationLayer(以下称为P)和modelLayer（以下称为M）。P只负责显示，M只负责数据的存储和获取。我们对layer的各种属性赋值比如frame，实际上是直接对M的属性赋值，而P将在每一次屏幕刷新的时候回到M的状态。比如此时M的状态是1，P的状态也是1，然后我们把M的状态改为2，那么此时P还没有过去，也就是我们看到的状态P还是1，在下一次屏幕刷新的时候P才变为2。而我们几乎感知不到两次屏幕刷新之间的间隙，所以感觉就是我们一对M赋值，P就过去了.强烈建议看[这篇博文](http://blog.csdn.net/u013282174/article/details/50388546)来深入了解这两个属性.
##### CALayer 的隐式动画
任何`UIView`内部都有一个`CALayer`来进行渲染,这个layer就称为`UIView`的`根层`(rootLayer),在iOS中修改**非根层**(不是view.layer)的`Animatable`类型的属性时会产生动画,称之为`隐式动画`.
修改position的隐式动画

```c
	- (void)viewDidLoad {
	    [super viewDidLoad];
	    _layer = [CALayer layer];
	    _layer.anchorPoint = CGPointMake(0, 0);
	    _layer.frame = CGRectMake(10, 10, 100, 100);
	    _layer.backgroundColor = [UIColor redColor].CGColor;
	    [self.view.layer addSublayer:_layer];
	}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
  	  _layer.position = CGPointMake(50, 70);
	}
```
![](http://o7vzr7y09.bkt.clouddn.com/animation1.gif)
修改anchorPoint的隐式动画,跟上图一样的效果,原理是position没有变但是anchorPoint变了那么如果仍要保持原来的positon的话,layer就会向右下平移.

```c
	- (void)viewDidLoad {
		_layer.delegate = self;
	    _layer.bounds = CGRectMake(0, 0, 100, 100);
	    _layer.position = CGPointMake(60, 60);
	    _layer.backgroundColor = [UIColor redColor].CGColor;
	    [self.view.layer addSublayer:_layer];
    }
    - (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
 	   _layer.anchorPoint = CGPointMake(0, 0);   
	}
```
这里改变position属性产生位移动画效果,还可以设置

`backgroundColor`,`transform`,`anchorPoint`,`zPosition`,`anchorPointZ`,`frame`,`hidden`,`doubleSided`,`masksToBounds`,`contents`,`contentsRect`,`contentsScale`,`contentsCenter`,`minificationFilterBias`.如果想关闭隐式动画可以通过以下设置
```
	[CATransaction setDisableActions:YES];
```
下篇会继续讲解CALayer的一些使用.






