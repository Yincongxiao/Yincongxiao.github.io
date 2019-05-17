---
title: CoreAnimation-CALayer(三)
date: 2016-02-01 00:06:02
category: iOSTips
tags:
---

# CoreAnimation-CALayer(三)
## CASharpLayer
`CASharpLayer`是一个通过矢量图形而不是bitmap（位图）来绘制的CALayer子类.我们可以指定`CGPathRef`来快速地创建各种形状的layer,CASharpLayer里面有各种`Animatable`的属性,通过这些属性我们可以实现一些很有意思的动画效果,例如[前文](http://asnail.xyz/2016/01/10/CoreAnimation-CALayer-二/)的圆环进度条,还有下面的浮动动画.

如果我们使用普通的CALayer通过设置绘制路径也能实现不同形状的layer,但是是用CASharpLayer有以下好处:

* 渲染快速。CAShapeLayer使用了硬件加速，绘制同一图形会比用Core Graphics快很多。
* 高效使用内存。一个CAShapeLayer不需要像普通CALayer一样创建一个寄宿图形（backing image），所以无论有多大，都不会占用太多的内存。
* 不会被图层边界剪裁掉。一个CAShapeLayer可以在边界之外绘制。你的图层路径不会像在使用Core Graphics的普通CALayer一样被剪裁掉。
* 不会出现像素化。当你给CAShapeLayer做3D变换时，它不像一个有寄宿图的普通图层一样变得像素化。

##### CASharpLayer 的属性
凡是标记`Animatable`都是可动画属性,可以再CoreAnimation使用来实现动画.
###### path `Animatable`
 sharpLayer的路径,指定路径就能生成相应形状的sharpLayer
###### fillColor; `Animatable`
 填充色,默认是不透明的黑色
###### fillRule
填充规则,有`kCAFillRuleNonZero`(默认),`kCAFillRuleEvenOdd`
###### strokeColor `Animatable`
 描边颜色
###### strokeStart; `Animatable`
表示绘画开始的地方占总绘画过程的百分比,取值范围[0~1]
###### strokeEnd; `Animatable`
表示绘画结束的地方占总绘画过程的百分比,[0~1]

下面是通过改变path来实现路径的动画效果.

```c
- (void)viewDidLoad {
    [super viewDidLoad];
    
    UIImage *image = [UIImage imageNamed:@"image"];
    
    CALayer *waterLayer = [CALayer layer];
    waterLayer.bounds = CGRectMake(0, 0, 300, 200);
    waterLayer.anchorPoint = CGPointMake(0, 0);
    waterLayer.position = CGPointMake(50, 200);
    waterLayer.contents = (__bridge id)image.CGImage;
    [self.view.layer addSublayer:waterLayer];
    
    UIBezierPath *bezierPath = [UIBezierPath bezierPath];
    [bezierPath moveToPoint:CGPointMake(0, 25)];
    [bezierPath addQuadCurveToPoint:CGPointMake(300, 25) controlPoint:CGPointMake(150, -30)];
    [bezierPath addLineToPoint:CGPointMake(300, 200)];
    [bezierPath addLineToPoint:CGPointMake(0, 200)];
    [bezierPath closePath];
    
    UIBezierPath *bezierPath2 = [UIBezierPath bezierPath];
    [bezierPath2 moveToPoint:CGPointMake(0, 50)];
    [bezierPath2 addQuadCurveToPoint:CGPointMake(300, 50) controlPoint:CGPointMake(150, 80)];
    [bezierPath2 addLineToPoint:CGPointMake(300, 200)];
    [bezierPath2 addLineToPoint:CGPointMake(0, 200)];
    [bezierPath2 closePath];
    
    _layer = [CAShapeLayer layer];
    _layer.path = bezierPath.CGPath;
    
    waterLayer.mask = _layer;
    
    NSTimer *timer = [NSTimer timerWithTimeInterval:3 repeats:YES block:^(NSTimer * _Nonnull timer) {
        NSValue *toValue;
        if (_path1) {
            _path1 = NO;
            toValue = (__bridge NSValue *)bezierPath.CGPath;
        }else {
            _path1 = YES;
            toValue = (__bridge NSValue *)bezierPath2.CGPath;
        }
        CABasicAnimation *animation = [CABasicAnimation animation];
        animation.duration = 3;
        animation.removedOnCompletion = NO;
        animation.fillMode = kCAFillModeForwards;
        animation.keyPath = @"path";
        animation.toValue = toValue;
        [_layer addAnimation:animation forKey:nil];
    }];
    [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
    [timer fire];
}
```
![](http://o7vzr7y09.bkt.clouddn.com/waterAnimation.gif)
#### 矢量图和bitmap
##### 位图
位图是通过排列像素点来构造的，像素点的信息包括颜色+透明度(ARGB)，颜色通过RGB来表示，所以一个像素一共有4个信息(透明度、R、G、B)，每个信息的取值范围是0-255，也就是一共256个数，刚好可以用8位二进制来表示，所以每个像素点的信息通常通过32位（4字节）编码来表示，这种位图叫做32位位图，而一些位图没有Alpha通道，这样的位图每个像素点只有RGB信息，只需要24位就可以表示一个像素点的信息。

位图在进行变形（缩放、3D旋转等）时会重新绘制每个像素点的信息，所以会造成图形的模糊。

值得一提的是，对于GPU而言，它绘制位图的效率是相当高的，所以如果你要提高绘制效率，可以想办法把复杂的绘制内容转换成位图数据，然后丢给GPU进行渲染，比如使用CoreText来绘制文字。
##### 矢量图

矢量图是通过对多个点进行布局然后按照一定规则进行连线后形成的图形。矢量图的信息总共只有两个：点属性和线属性。点属性包括点的坐标、连线顺序等；线属性包括线宽、描线颜色等。

每当矢量图进行变形的时候，只会把所有的点进行重新布局，然后重新按点属性和线属性进行连线。  __所以每次变形都不会影响线宽，也不会让图变得模糊__。

如何重新布局是通过把所有点坐标转换成矩阵信息，然后通过矩阵乘法重新计算新的矩阵，再把矩阵转换回点信息。比如要对一个矢量图进行旋转，就先把这个矢量图所有的点转换成一个矩阵（x,y,0），然后乘以旋转矩阵：
    
    (    cosa  sina   0
         -sina  cosa  0
    0      0  1)

得到新的矩阵（x·cosa-y·sina, x·sina+y·cosa, 0） 
然后把这个矩阵转换成点坐标（x·cosa-y·sina, x·sina+y·cosa）这就是新的点了。对矢量图所有的点进行这样的操作后，然后重新连线，出现的新的图形就是旋转后的矢量图了。



