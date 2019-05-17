---
title: runtime黑魔法-Method Swizzling.md
date: 2015-07-25 15:16:36
category: iOSTips
tags:
---
#
### Method Swizzling
> 场景:我们需要在DEBUG模式下通过打印台知道我们点击跳转的控制器的名字,当然我们可以在自己的`baseViewController`的`viewWillAppear:`中进行操作,但是这样我们的所有类都必须继承自这个`baseViewController`才能得到这个能力,有没有更加面向切面的方式来达到这个效果呢?`Method Swizzling`!

我们需要做的就是给UIViewController写一个分类可以命名为UIViewController (MethodSwizzling)


```c
@implementation UIViewController (MethodSwizzling)

+ (void)load {
static dispatch_once_t onceToken;
dispatch_once(&onceToken, ^{
Class class = [self class];
//get selector of `viewWillAppear`
SEL originalSelector = @selector(viewWillAppear:);
//get selector of `swizzed_viewWillAppear`
SEL swizzledSelector = @selector(swizzed_viewWillAppear:);
//get Method of `viewWillAppear`
Method originalMethod = class_getInstanceMethod(class, originalSelector);
Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
// exchange the IMP of the method `viewWillAppear` and `swizzed_viewWillAppear`
method_exchangeImplementations(originalMethod, swizzledMethod);
});
}

- (void)swizzed_viewWillAppear:(BOOL)animated {
[self swizzed_viewWillAppear:animated];
NSLog(@"viewWillAppear-%@",self.class);
}

//viewWillAppear-ViewController
@end
```
#### 原理:
想理解方法交换首先要理解以下几个概念:`Method`,`SEL`,`IMP`
##### Method

* 在+load方法中获取两个方法(Method)
* 交换他们的实现(IMP)

#### 注意点:
* method swizzling的实现要在+ load中而不是在+ initialize方法中,它俩的区别可以看[这篇文章](http://asnail.xyz/2015/07/20/初始化方法-load-和-initialize方法/)
* 注意我们的swizzled的方法一定要调用自己,而不是调用原方法,而且这个方法我们基本一定要调用!是因为方法的调用实际上是寻找方法实现的过程,正常通过方法名`viewWillAppear:`去class的方法列表(dispatch_table)中寻找到的是`viewWillAppear`的IMP实现,但是我们通过`method_exchangeImplementations`将这两个方法的IMP交换了,所以这里调用`swizzed_viewWillAppear:`其实是找到了`viewWillAppear`的实现.
* 在类簇中是无法通过method swizzling 的方式进行方法交换的.


