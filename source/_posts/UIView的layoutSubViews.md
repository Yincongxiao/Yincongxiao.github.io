---
title: UIView的layoutSubViews
date: 2015-01-25 15:05:09
category: iOSTips
tags:
---
本文包含了`UIView`中`layoutSubviews`,`setNeedsLayout`,`layoutIfNeeded`三个常用方法的调用时机.以后还会有其他方法的研究补充.

### layoutSubviews

UIView中的`layoutSubviews`方法本身实际上并没有做什么事情,而是需要交给子view去复写,顾名思义我们应该在这个方法中调整(布局)子view,我们只有充分了解该方法的调用时机,才能知道某些调整是不是应该写到这个方法中.
经过实例验证`layoutSubviews`的调用时机有以下几个情况:

* 初始化的时候是不会调用`layoutSubviews`,view被添加到父view上的时候,并且view的frame不是`CGRectZero`才会调用`layoutSubviews`!
* view的`size`(注意是size)发生改变的时候.如果只改变位置而size没发生改变是不会调用`layoutSubviews` 方法的,想想这也完全合理.
* 屏幕旋转的时候会调用,`window`->`rootVC`->`rootVC.SubViews`
* 调用`addSubView:`方法
* `UIScrollView`在滑动的时候会频繁调用`layoutSubviews`

下面我们就逐个验证一下,加深一下记忆.往下看会有意想不到的收货哦.
首先我们创建一个view取名为`ParentView`继承自`UIView`
仅仅是打印一下方法的执行.
```c
- (void)layoutSubviews {
[super layoutSubviews];
NSLog(@"%s",__func__);
}
```
在vc中

##### view被add到父view上

```c
- (void)viewDidLoad {
[super viewDidLoad];
ParentView *pv = [[ParentView alloc] initWithFrame:CGRectMake(100, 100, 100, 50)];
pv.backgroundColor = [UIColor redColor];
[self.view addSubview:pv];
}
//-[ParentView layoutSubviews]
```
有打印,说明view被加到父view上确实调用了`layoutSubviews`方法,但是我们把初始化的frame改为`CGRectZero`,那么没有打印,说明**view的frame不为CGRectZero才会调用layoutSubviews**
我们把`[self.view addSubview:pv];`注释掉以后就没有打印,所以**初始化不会调用layoutSubviews!**

##### setNeedsLayout
`setNeedsLayout`方法:给view加上需要刷新的标记,异步的去调用`layoutIfNeeded`,从而去调用`layoutSubviews`(关于`layoutIfNeeded`方法下文会具体分析)
```c
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
[_pv setNeedsLayout];
NSLog(@"%s",__func__);
}
//-[ViewController touchesBegan:withEvent:]
//-[ParentView layoutSubviews]
```
发现每次点击都会调用,说明`layoutSubviews`确实会触发view的`layoutSubviews`,注意这里是**异步**调用,所以并不是在调用`setNeedsLayout`的时刻去调用`layoutSubviews`,而是在下一个runloop循环处理该事件,所以打印顺序是这样的!

注意: addSubView其实是隐式得调用了`setNeedsLayout`方法,我们来验证一下
在`addSubview`后加一句打印

```c
[self.view addSubview:_pv];
NSLog(@"addSubview");
// addSubview
//-[ParentView layoutSubviews]
```
先调用的addSubview再异步调用layoutSubviews.

##### layoutIfNeeded
```c
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
[_pv layoutIfNeeded];
}
```
发现并不会打印,`layoutIfNeeded`:调用该方法会首先查看该view是不是具有*需要重新布局的标志*(是否调用过`setNeedsLayout`方法).如果有标记那么就去调用`layoutSubviews`,如果没有那么什么都不做!
tip:我们如果想立马重新排版那么可以组合使用这两个方法.通常我们只需要调用`setNeedsLayout`就能达到要求.
```c
[_pv setNeedsLayout];
[_pv layoutIfNeeded];
```
```c
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
[_pv setNeedsLayout];
[_pv layoutIfNeeded];
NSLog(@"%s",__func__);
}
//-[ParentView layoutSubviews]
//-[ViewController touchesBegan:withEvent:]
```
看打印顺序发生了改变;



##### 改变 view 的frame
```c
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
_pv.frame = CGRectMake(100, 100, 100, 50);
}
```
我们直接设置为原来的frame,发现也不会打印`layoutSubviews`,后来验证得出结论: **view的`size`(注意是size)发生改变的时候才会调用`layoutSubviews`.如果只改变位置而size没发生改变是不会调用`layoutSubviews` 方法的,想想这也完全合理**


##### 旋转屏幕
通过真机测试,我这里得到这样一个结论(很费解):
* UIView没有子view,旋转屏幕并不会调用它的`layoutSubviews`方法
* UIView有子view并且子view不是`UIView`类型的(UIView的子类例如UIButton,UILabel,UIImageView,UIScrollView等)会调用两次`layoutSubviews`方法.

##### UIScrollView的layoutSubviews
在这里我们只是将`ParentView`父类改变为`UIScrollView`,我发现`addSubView:`调用的时候会调用两次`layoutSubviews`,而普通的UIView只会调用一次.并且给他设置一个contentSize,当滑动的时候确实会多次调用`layoutSubviews`.


