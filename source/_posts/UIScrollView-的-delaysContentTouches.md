---
title: UIScrollView 的 delaysContentTouches
date: 2015-11-23 13:15:33
category: iOSTips
tags:
---
# UIScrollView的`delaysContentTouches`

#### 问题描述: 
* 当UIScrollView上面有一个UIButton控件,在快速点击button的时候button没有高亮状态,却能响应点击事件,长按却有高亮,这样对用户很不友好
* 在iOS8下运行以前开发的项目时,如果触摸点是在子控件下，则会导致触摸事件被子控件接收后不能再次返回事件给UIScrollView，以至于不能滑动UIScrollView。

查阅资料后发现:
> UIScrollView的 touch 工作原理: 当手指touch的时候，UIScrollView会拦截该 touch 事件,会等待一段时间(150ms)，在这段时间内，如果没有手指没有移动，当事件结束时，UIScrollView会发送tracking events到子视图上。在事件结束前，手指发生了移动，那么UIScrollView就会进行移动，从而不会向子视图发送事件.

##### delaysContentTouches
`default is YES. if NO, we immediately call -touchesShouldBegin:withEvent:inContentView:. this has no effect on presses`

`delaysContentTouches` 默认值为YES，即UIScrollView会在接受到手势是延迟150ms来判断该手势是否能触发UIScrollView的滑动事件；
`delaysContentTouches` 为NO时，UIScrollView会立马将接受到的事件分发到子视图上

如果我们单纯地将这个属性设置为NO,那么当手指触摸点在子view上滑动scrollView时就不能正常滑动,应为事件直接被发送给了子view处理
我们还要做的是重写`- (BOOL)touchesShouldCancelInContentView:(UIView *)view`方法,这个方法的调用时机是:当手指在子view上开始滑动时系统会调用这个方法询问是否取消子view的touch事件,如果返回no,那么不取消,仍然由子view处理,如果返回yes那么子view取消touch事件交由scrollView接管

上面原理看完了那么我们需要做什么呢?
1. 自定义scrollView,将delaysContentTouches设置为NO
2. 重写`touchesShouldCancelInContentView`判断如果子View是我们的button返回yes,否则返回super操作.

```
@interface CustomScrollView : UIScrollView

@end

@implementation CustomScrollView

- (instancetype)initWithFrame:(CGRect)frame {
    if (self = [super initWithFrame:frame]) {
        self.delaysContentTouches = NO;
    }
    return self;
}

- (BOOL)touchesShouldCancelInContentView:(UIView *)view {
    if ([view isKindOfClass:[UIButton class]]) {
        return YES;
    }
     return [super touchesShouldCancelInContentView:view];
}

@end
```

#### UITableView
`UITableView`是`UIScrollView`的子类,当然也会遇到上面的问题,如果我们仅仅只是按照上面的解决方法自定义tableView,你会发现快速点击button还是不会高亮,原因是`UITableView`和cell之间还会有其他我们看不到的view层,这些层的`delaysContentTouches`默认也是YES

在iOS7与iOS8的表现还不一样

* iOS7下一个UIScrollView会有若干个`UITableVieCellScrollVIew`,他们存在于cell的contentView之间
* iOS8下一个UIScrollView在scrollView本身和cell之间会有一个`UITableViewWrapperView`,并没有iOS7下的`UITableVieCellScrollVIew`.

我们要做的就是找到这些view,然后将它们的`delaysContentTouches`设置为NO.

```
@interface CustomTableView : UITableView

@end

@implementation CustomTableView

- (instancetype)initWithFrame:(CGRect)frame {
    if (self = [super initWithFrame:frame]) {
        
        self.delaysContentTouches = NO;
        
        ///iOS 7
        for (UIView *subView in self.subviews) {
            if ([NSStringFromClass(subView.class) hasSuffix:@"CellScrollView"]) {
                UIScrollView *scrollView = (UIScrollView *)subView;
                scrollView.delaysContentTouches = NO;
                break;
            }
        }
        
        ///iOS 8
        UIView *wrapView = self.subviews.firstObject;
        if (wrapView && [NSStringFromClass(wrapView.class) hasSuffix:@"WrapperView"]) {
            for (UIGestureRecognizer *gesture in wrapView.gestureRecognizers) {
                // UIScrollViewDelayedTouchesBeganGestureRecognizer
                if ([NSStringFromClass(gesture.class) containsString:@"DelayedTouchesBegan"] ) {
                    gesture.enabled = NO;
                    break;
                }
            }
        }
    }
    return self;
}

- (BOOL)touchesShouldCancelInContentView:(UIView *)view {
    if ([view isKindOfClass:[UIButton class]]) {
        return YES;
    }
     return [super touchesShouldCancelInContentView:view];
}

@end
```



