---
title: CoreAnimation-CALayer(四)
date: 2016-02-06 16:40:25
category: iOSTips
tags:
---
# CoreAnimation-CALayer(四)
## Animation的原理
我们在前面讲了在修改非根层的layer的animatable属性时会产生隐式动画,那为什么直接修改view.layer的animatable属性并不能产生动画,而是需要放在`+ (void)animateWithDuration:(NSTimeInterval)duration animations:(void (^)(void))animations`的animationBlock中才会产生动画呢?

前面文章讲了UIView和CALayer的关系,一个UIView内部默认有一个layer(rootLayer),layer负责绘制,view是layer的delegate.
其实我们在修改layer的animatable属性时,layer会调用它的delegate的`- (nullable id<CAAction>)actionForLayer:(CALayer *)layer forKey:(NSString *)event`方法,这个方法的返回值有三种情况
 
 * 返回nil
 * 返回(NSNull * )对象
 * 返回一个 (id<CAAction>)的对象.

 对这三种情况会产生三种效果:返回值为nil,就会产生**隐式动画**,返回值为(NSNull *)没有动画效果,第三种情况是使用返回的(id<CAAction>)对象产生一个CAAnimation,并且调用自己的addAnimation:方法进行动画.
直接修改非根层layer的animatable属性属于第一种,因为这个layer没有delegate对象,该方法返回nil,所以就有了隐式动画
直接修改根层layer的属性又分为两种情况,第一种在animationBlock外部直接修改,这里就返回NSNull *,第二种在animationBlock内部修改,这里会返回(id<CAAction>)产生动画,这里我们需要注意直接修改UIView的属性例如transform,其实是直接修改的它内部的layer的transform属性!
我们来验证一下

```c
//CustomLayer.m
@implementation CustomLayer
- (void)setPosition:(CGPoint)position {
    id action = nil;
    if ([self.delegate respondsToSelector:@selector(actionForLayer:forKey:)]) {
        action = [self.delegate actionForLayer:self forKey:@"positon"];
    }
    if (action == nil) {
        NSLog(@"action == nil");
    }else if ([action isKindOfClass:[NSNull class]]) {
        NSLog(@"action is null");
    }else if ([action conformsToProtocol:@protocol(CAAction)]) {
        NSLog(@"action conformsToProtocol:@protocol(CAAction)");
    }
        [super setPosition:position];
}
@end

//CustomView.m
@implementation CustomView
+ (Class)layerClass {
    return [CustomLayer class];
}
@end

//ViewController.m
- (void)viewDidLoad {
    [super viewDidLoad];
    _cV = [[CustomView alloc] initWithFrame:CGRectMake(50, 100, 100, 100)];
    _cV.backgroundColor = [UIColor redColor];
    [self.view addSubview:_cV];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    _cV.layer.position = CGPointMake(150, 200);
}
```

我们这里直接修改的是根层的position,输出台打印结果是: action is null

```c
- (void)viewDidLoad {
    [super viewDidLoad];
    _custLayer = [CustomLayer layer];
    _custLayer.bounds = CGRectMake(0, 0, 200, 200);
    _custLayer.position = CGPointMake(300, 300);
    _custLayer.backgroundColor = [UIColor redColor].CGColor;
    [self.view.layer addSublayer:_custLayer];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    _custLayer.position = CGPointMake(350, 350);
}
```
这里直接修改非根层layer的属性,打印结果是action == nil

#### CALayer的 presentationLayer 和 modelLayer
```c
- (void)viewDidLoad {
    [super viewDidLoad];
    _customView = [[UIView alloc] initWithFrame:CGRectMake(100, 100, 100, 100)];
    _customView.backgroundColor = [UIColor redColor];
    [self.view addSubview:_customView];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    CABasicAnimation *animation = [CABasicAnimation animation];
    animation.keyPath = @"position";
    animation.duration = 0.5;
    animation.toValue = [NSValue valueWithCGPoint:CGPointMake(200, 200)];
    [_customView.layer addAnimation:animation forKey:nil];
}
```
![](http://o7vzr7y09.bkt.clouddn.com/transform.gif)
我们看到动画完成后又回到了原来的位置,这是为什么呢?
在CALayer内部，它控制着两个属性：presentationLayer 展示层(以下称为P)和modelLayer（模型层以下称为M）。

P只负责显示，M只负责数据的存储和获取。我们对layer的各种属性赋值比如frame，**实际上是直接对M的属性赋值**，而P将在每一次屏幕刷新的时候回到M的状态。比如此时M的状态是1，P的状态也是1，然后我们把M的状态改为2，那么此时P还没有过去，也就是我们看到的状态P还是1，在下一次屏幕刷新的时候P才变为2。而我们几乎感知不到两次屏幕刷新之间的间隙，所以感觉就是我们一对M赋值，P就过去了。P就像是瞎子，M就像是瘸子，瞎子背着瘸子，瞎子每走一步（也就是每次屏幕刷新的时候）都要去问瘸子应该怎样走（这里的走路就是绘制内容到屏幕上），瘸子没法走，只能指挥瞎子背着自己走。可以简单的理解为：一般情况下，任意时刻P都会回到M的状态。

而当一个CAAnimation（以下称为A）加到了layer上面后，A就把M从P身上挤下去了。现在P背着的是A，P同样在每次屏幕刷新的时候去问他背着的那个家伙，A就指挥它从fromValue到toValue来改变值。而动画结束后，A会自动被移除，这时P没有了指挥，就只能大喊“M你在哪”，M说我还在原地没动呢，于是P就顺声回到M的位置了。这就是为什么动画结束后我们看到这个视图又回到了原来的位置，是因为我们看到在移动的是P，而指挥它移动的是A，M永远停在原来的位置没有动，动画结束后A被移除，P就回到了M的怀里。

我们不想动画完成以后再回到原来位置怎么办呢?`CAAniamtion`基类中提供了`removedOnCompletion`,这个属性默认是YES,我们需要将它置为NO,同时将`fillMode`设置为`kCAFillModeForwards`.

```c
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    CABasicAnimation *animation = [CABasicAnimation animation];
    animation.keyPath = @"position";
    animation.duration = 0.5;
    animation.removedOnCompletion = NO;
    animation.fillMode = kCAFillModeForwards;
    animation.toValue = [NSValue valueWithCGPoint:CGPointMake(200, 200)];
    [_customView.layer addAnimation:animation forKey:nil];
}
```
![](http://o7vzr7y09.bkt.clouddn.com/transform1.gif)
这两个方法其实是coreAnimation将presentationLayer属性一直保持在动画结束后的状态,而实际的modelLayer属性并没有真正发生改变. 我们如果把customView换成UIButton的话,在动画完成以后点击按钮是不能响应的.点击按钮动画前的位置却能响应!下面是解决办法.

```c
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    //注意: 关于toValue 和 fromValue 设置了这两个值以后presentationLayer的值(也就是动画轨迹)实际上是
    // presentationLayer原始值 -> fromValue -> toValue,动画完成后toValue -> modelLayer的值.
    NSLog(@"presentationLayer.position: %@",NSStringFromCGPoint(_customView.layer.presentationLayer.position));
    NSLog(@"modelLayer.position: %@",NSStringFromCGPoint(_customView.layer.modelLayer.position));
    _customView.layer.position = CGPointMake(200, 200);
    NSLog(@"presentationLayer.position: %@",NSStringFromCGPoint(_customView.layer.presentationLayer.position));
    NSLog(@"modelLayer.position: %@",NSStringFromCGPoint(_customView.layer.modelLayer.position));
    //presentationLayer.position 经历了 (150, 150) -> (150, 150) -> (200, 200) 的过程
    CABasicAnimation *animation = [CABasicAnimation animation];
    animation.keyPath = @"position";
    animation.duration = 0.5;
    animation.removedOnCompletion = NO;
    animation.fillMode = kCAFillModeForwards;
    animation.fromValue = [NSValue valueWithCGPoint:CGPointMake(150, 150)];
    [_customView.layer addAnimation:animation forKey:nil];
}
```
上面的例子实际上将`animation.fromValue = [NSValue valueWithCGPoint:CGPointMake(150, 150)];`这句注释掉也是没问题的,这样既能维持动画完成后状态,view的点击事件不受影响!具体的原理在代码里.
![](http://o7vzr7y09.bkt.clouddn.com/transform1.gif)


