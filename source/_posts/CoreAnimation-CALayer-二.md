---
title: CoreAnimation-CALayer(二)
date: 2016-01-10 17:33:11
category: iOSTips
tags:
---
## CoreAnimation-CALayer(二)
#### 修改view默认的layer
iOS中默认情况下UIView会创建一个layer,我们还可以为view指定其他的layer对象,大部分场景下我们不需要这样做,但是系统为我们提供了很多好用的layer子类,我们可以把这些layer指定为view相关的layer.比如`CATiledLayer`可以高效的管理大图片.
在view中重写`layerClass`方法来为view指定layer,再比如我们的view使用OpenGL ES 技术来进行绘制,我们可能会把view的layer换成`CAMetalLayer`或者`CAEAGLLayer`
替换很简单

```c
	CustomView.m
	+ (Class) layerClass {
		return [CAMetalLayer class];
	}
```
#### mask
`mask`遮盖层.本身也是一个layer

maskLayer类似于一个子图层，相对于父图层（即拥有该属性的图层，在这里就是layer）布局，但是它却不是一个普通的子图层。maskLayer并不会直接绘制在父图层之上，它只是定义了父图层的“可视部分”(可视形状)。mask属性就像是一个饼干切割机，mask图层实心的部分会被保留下来，其他的（透明的部分）则会被抛弃。如图
![](http://img.blog.csdn.net/20160812091842446)
通常我们使用`CASarpLayer`进行自定义的形状,然后设置为layer的mask,这样我们就可以得到任意形状的图片

```c
- (void)viewDidLoad {
    [super viewDidLoad];
    UIImage *image = [UIImage imageNamed:@"image"];
	_layer = [CALayer layer];
    _layer.anchorPoint = CGPointMake(0, 0);
    _layer.bounds = CGRectMake(0, 0, 200, 200);
    _layer.position = CGPointMake(50, 100);
    _layer.contents = (__bridge id)image.CGImage;
    _layer.masksToBounds = YES;
    _layer.contentsGravity = kCAGravityResizeAspectFill;
    [self.view.layer addSublayer:_layer];
  }
```
![](http://o7vzr7y09.bkt.clouddn.com/Simulator%20Screen%20Shot%202017%E5%B9%B45%E6%9C%883%E6%97%A5%20%E4%B8%8B%E5%8D%884.56.44.png)
我们添加下面的代码

```c
	- (void)viewDidLoad {
    [super viewDidLoad];
    
    UIImage *image = [UIImage imageNamed:@"image"];
    _layer = [CALayer layer];
    _layer.anchorPoint = CGPointMake(0, 0);
    _layer.bounds = CGRectMake(0, 0, 200, 200);
    _layer.position = CGPointMake(50, 100);
    _layer.contents = (__bridge id)image.CGImage;
    _layer.masksToBounds = YES;
    _layer.contentsGravity = kCAGravityResizeAspectFill;
    [self.view.layer addSublayer:_layer];
    
    CAShapeLayer *sharpLayer = [CAShapeLayer layer];
    UIBezierPath *path = [UIBezierPath bezierPath];
    //这里我制定了一个圆形,可以随意指定想要的图形.
    [path addArcWithCenter:CGPointMake(100, 100) radius:100 startAngle:0 endAngle:M_PI * 2 clockwise:YES];
    sharpLayer.path = path.CGPath;
    _layer.mask = sharpLayer;   
}
```
![](http://o7vzr7y09.bkt.clouddn.com/Simulator%20Screen%20Shot%202017%E5%B9%B45%E6%9C%883%E6%97%A5%20%E4%B8%8B%E5%8D%884.58.11.png)
下面我们再来添加几行代码来实现一个圆环,具体看注释

```c
	sharpLayer.lineWidth = 30;
    //指定描边颜色,随意设置不为clear的颜色就好,不设置的话不能达到效果!
    sharpLayer.strokeColor = [UIColor redColor].CGColor;
    //填充颜色设置为clear中间部分就不会显示出来
    sharpLayer.fillColor = [UIColor clearColor].CGColor;
```
![](http://o7vzr7y09.bkt.clouddn.com/Simulator%20Screen%20Shot%202017%E5%B9%B45%E6%9C%883%E6%97%A5%20%E4%B8%8B%E5%8D%885.11.33.png)
因为mask本身是个layer,所以它可以拥有动画效果
我们把代码稍作修改

```c
- (void)viewDidLoad {
    [super viewDidLoad];
    
    UIImage *image = [UIImage imageNamed:@"image"];
    
    _layer = [CALayer layer];
    _layer.anchorPoint = CGPointMake(0, 0);
    _layer.bounds = CGRectMake(0, 0, 200, 200);
    _layer.position = CGPointMake(50, 100);
    _layer.contents = (__bridge id)image.CGImage;
    _layer.masksToBounds = YES;
    _layer.contentsGravity = kCAGravityResizeAspectFill;
    [self.view.layer addSublayer:_layer];
    
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    CAShapeLayer *sharpLayer = [CAShapeLayer layer];
    UIBezierPath *path = [UIBezierPath bezierPath];
    [path addArcWithCenter:CGPointMake(100, 100) radius:100 -15 startAngle:0 endAngle:M_PI * 2 clockwise:YES];
    sharpLayer.path = path.CGPath;
    _layer.mask = sharpLayer;
    
    sharpLayer.lineWidth = 30;
    //指定描边颜色,随意设置不为clear的颜色就好,不设置的话不能达到效果!
    sharpLayer.strokeColor = [UIColor redColor].CGColor;
    //填充颜色设置为clear中间部分就不会显示出来
    sharpLayer.fillColor = [UIColor clearColor].CGColor;

    CABasicAnimation *animation = [CABasicAnimation animation];
    animation.keyPath = @"strokeEnd";
    animation.fromValue = @0;
    animation.duration = 3;
    [sharpLayer addAnimation:animation forKey:nil];
}
```
![](http://o7vzr7y09.bkt.clouddn.com/caca.gif)
#### CALayer 和  UIView的关系

`CALayer`作为一个跨平台框架（OS X和iOS）`QuatzCore`的类，负责MAC和iPhone（ipad等设备）上绘制所有的显示内容。而iOS系统为了处理用户交互事件（触屏操作）用UIView封装了一次CALayer，UIView本身负责处理交互事件，其持有一个Layer，用来负责绘制这个View的内容。而我们对UIView的和绘制相关的属性赋值和访问的时候（frame、backgroundColor等）UIView实际上是直接调用其Layer对应的属性（frame对应frame，center对应position等）的getter和setter。

layers并不是view的替代品,也就是说如果只使用CALayer是无法满足所有的界面要求的,layer与view协同合作,尤其是实现高效的动画或者渲染效果上面,但是layer不能识别用户交互事件,进行实际的绘制工作,不能参与事件传递链条,所以在app中必须要有部分view来接受用户的交互事件

在iOS中每一个View都有一个与之对应的layer.但是在OSX中我们必须指定view是否可以有自己的layer,在OSX v10.8以后的版本可以向iOS一样为每个view添加默认的layer,但是这并不是必须的,仍然可以在没必要使用view的时候来使用layer进行替换,layer在某种程度上会造成内存的增加,所以在使用之前最好做好利弊的权衡来决定是否使用这项技术.

> 这篇文章主要讨论了layer的mask层,知道了mask层我们就可以实现任意形状的自定义图层!
> 明白了UIView 和 CALayer的关系,我们就可以在适当的场景选择不同的技术进行实现.


