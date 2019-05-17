---
title: reactivecocoa深入学习
date: 2015-11-15 20:13:55
category: iOSTips
tags:
---

## ReactiveCocoa

### 前言

* 在 **iOS** 编程中我们需要处理各种事件,例如响应按钮的点击,监听键盘的输入,监听网络回包等...我们通常使用`Cocoa`推荐的例如`target-action`、`delegate`、`key-value observing`、`callback`等。
* **ReactiveCocoa**为我们提供了一种统一化的解决此类问题的方式,使用RAC解决问题，就不需要考虑调用顺序，直接考虑结果，把每一次操作都写成一系列嵌套的方法中，使代码高聚合，方便管理。
* **ReactiveCocoa**将所有`Cocoa`中的事件都定义为了信号\(single\)，从而可以使用一些基本工具来更容易的连接、过滤和组合.

### RAC中涉及到的编程思想:
 
* **函数式编程**（[functional programming](https://en.wikipedia.org/wiki/Functional_programming)）：使用高阶函数，例如函数用其他函数作为参数。

* **响应式编程**（[reactive programming](https://en.wikipedia.org/wiki/Reactive_programming)）：关注于数据流和变化传播。不需要考虑事件的调用过程,只需要关注数据的流入和输出.


>  所以，你可能听说过reactivecocoa被描述为[函数响应式编程](https://en.wikipedia.org/wiki/Functional_reactive_programming)[FRP](https://en.wikipedia.org/wiki/Functional_reactive_programming)框架。
> 其他平台上也有类似的框架例如java的 **RXJava**  swift中的**ReactiveSwif**

* **链式编程** : 是将多个操作（多行代码）通过点号.链接在一起成为一句代码,使代码可读性好。注意点:要想达到链式编程方法的返回值必须是一个(返回值是本身对象的)`block`),典型代表就是[Masonry](https://github.com/SnapKit/Masonry)框架

---

### RAC框架的结构\(直接略过\)

![](http://blog.leichunfeng.com/images/ReactiveCocoa%20v2.5.png)
### RAC中重要的类
#### RAC最重要的类是**RACSingle**\(信号类\)

* 它本身不具备发送信号的能力，而是交给内部一个订阅者\(`id <RACSubscriber>`\)去发出。
* 默认一个信号创建出来都是冷信号，即使是值改变了，也不会触发，只有订阅了这个信号，这个信号才会变为热信号，值改变了就会触发
* RACSignal的每个操作都会返回一个RACsignal，这在术语上叫做连贯接口（fluent interface）,从而实现 `链式编程`

---

> 所以这里着重介绍一下RACSingle的创建和订阅的实现,从而理解信号的创建以及数据的传递过程


```objc
//这里用到一个工厂方法将子类实例返回回去
+ (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe {
 return [RACDynamicSignal createSignal:didSubscribe];
}
//我们再来看看子类`RACDynamicSignal`中的具体实现,其实就是吧传递过来的didSubscribe这个block保存起来
+ (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe {
 RACDynamicSignal *signal = [[self alloc] init];
 signal->_didSubscribe = [didSubscribe copy];
 return [signal setNameWithFormat:@"+createSignal:"];
}
```

方法的参数`didSubscribe`这个block 可以理解为是对信号的描述,
上面只是信号的创建过程,上面提到了默认信号被创建出来以后只是冷信号,也就是**didSubscribe**这个block只有当RACSingle调用subscribeNext:方法是才会调用,方法里的subscriber有三种方法,也可以理解为可以发送三种信号分别为:**next**、**error**、**completed**,
其中`sendNext()`是发送我们需要传递的对象,`sendError`,和`sendComplete`都会中断信号的订阅.不同的是`sendError`会传递一个错误值`error`.一个`signal`在因`error`终止或者`sendComplete`前，可以发送任意数量的`next`事件


```
  - (RACDisposable *)subscribeNext:(void (^)(id x))nextBlock;
```

下面我们再来看看:subscribeNext:方法的实现过程:

```c
- (RACDisposable *)subscribeNext:(void (^)(id x))nextBlock {
 NSCParameterAssert(nextBlock != NULL);
 RACSubscriber *o = [RACSubscriber subscriberWithNext:nextBlock 
                                                error:NULL 
                                            completed:NULL];
 return [self subscribe:o];
}
```

这里首先创建一个RACSubscriber对象,这个是遵守了RACSubscriber协议的**RACSubscriber \***类型的对象,这个对象会将传递进来的nextBlock保存起来\(也是保存成成员变量\),当创建对象时候的didSubscribe中的"subscriber"调用sendNext:的时候nextBlock就会被调用! 有点绕,这里只是介绍了其中大概的工作原理,其中还有RACScheduler的参与,这里就不做介绍了..

 #### RAC为cocoa中的很多类通过分类的形式快速生成信号以及常见的宏使用:

**@weakify\(self\) & @strongify\(self\)**: _Weak & Strong dance_

**RACObserve\(TARGET, KEYPATH\) & RAC\(TARGET, ...\)** :这对宏简直是绝配类似**KVC**的用法,能够快速地实现对象属性的映射

```
//e.g.
RAC(self.titleLabel,text) = [RACObserve(self.viewModel, title);

```

**rac\_signalForSelector** : 执行某一个方法就会生成信号

**rac\_valuesAndChangesForKeyPath**：用于监听某个对象的属性改变。

**rac\_signalForControlEvents**:替代_UIControl_中的_Target_模式

**rac\_addObserverForName**用于监听某个通知。

**RACTuplePack\(...\)**快速包装成元祖类,成对出现的有RACTupleUnpack\(...\)

```
 RACTuple *tuple = RACTuplePack(@"xmg",@20); 
 RACTupleUnpack(NSString *name,NSNumber *age) = tuple;

```

**rac\_liftSelector:withSignalsFromArray:Signals:**

> 应用场景: 当界面有多个请求的时候,当所有的请求都回包时才出发某个操作
> 
> ```
>  RACSignal *request1 = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
>      // 发送请求1 ,并且获取到数据以后:
>     [subscriber sendNext:@"发送请求1"];
>      return nil; 
> }];
>  RACSignal *request2 = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
>      // 发送请求2 ,并且获取到数据以后:
>     [subscriber sendNext:@"发送请求2"];
>      return nil; 
> }]; 
> // 使用注意：updateUIWithR1::中的参数和信号所传递过来的数据一一对应
>  [self rac_liftSelector:@selector(updateUIWithR1:r2:) withSignalsFromArray:@[request1,request2]];
> ```

#### RACSubject类

* _RACSubject_既是信号\(继承自_RACSingle_类\),又是订阅者\(遵循`<RACSubscriber>`\),通常使用_RACSubject_类来代替代理,其实我感觉代替通知都可以

  ```
  RACSubject *subject = [RACSubject subject];
  [subject subscribeNext:^(id x) { 
     NSLog(@"第一个订阅者%@",x); 
  }];
  [subject subscribeNext:^(id x) {
     NSLog(@"第二个订阅者%@",x); 
  }];
  [subject sendNext:@"1"];
  ```

* _RACReplaySubject_:重复提供信号类，RACSubject的子类。


**_RACReplaySubject与RACSubject区别_**: RACReplaySubject可以先发送信号，在订阅信号，RACSubject就不可以。如果一个信号每被订阅一次，就需要把之前的值重复发送一遍，使用重复提供信号类。

#### RAC中的集合类RACSequence

**RACSequence**是RAC中的集合类,可以实现OC对象与信号中传递值之间的转换,RAC类库中提供了NSArray,NSDictionary等集合类的分类供其转换

```
//遍历数组
NSArray *numbers = @[@1,@2,@3,@4]; 
[numbers.rac_sequence.signal subscribeNext:^(id x) {
     NSLog(@"%@",x);
 }];

//遍历字典
NSDictionary *dict = @{@"name":@"qdaily",@"age":@3};
 [dict.rac_sequence.signal subscribeNext:^(RACTuple *x) { 
       RACTupleUnpack(NSString *key,NSString *value) = x; 
       NSLog(@"%@ %@",key,value); 
}];

```

**RACTupleUnpack**将RACTuple进行解包

```
//使用map方法可以快速进行字典转模型
NSArray *flags = [[dictArr.rac_sequence map:^id(id value) {
     return [FlagItem flagWithDict:value]; 
}] array];
```

#### RACCommand:RAC中用于处理事件的类

> 可以理解为命令类,是对信号和事件的封装

使用场景:监听按钮点击，网络请求
_RACCommand_使用注意点:

1. signalBlock必须要返回一个信号，不能传nil. 
2. 如果不想要传递信号，直接创建空的信号\[RACSignal empty\]; 
3. RACCommand中信号如果数据传递完，必须调用\[subscriber sendCompleted\]，这时命令才会执行完毕，否则永远处于执行中。 
4. RACCommand需要被强引用，否则接收不到RACCommand中的信号，因此RACCommand中的信号是延迟发送的。

RACCommand简单使用

```
 RACCommand *requestCommand = [[RACCommand alloc] initWithSignalBlock:^RACSignal *(id input) {
     NSLog(@"请求数据ing..."); 
    //这里即使不需要传递值也要创建空的信号而不能返回nil
    //    return [RACSignal empty]; 
     return [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        [subscriber sendNext:@"请求数据"];
         // 注意：数据传递完，最好调用sendCompleted，这时命令才执行完毕。 
        [subscriber sendCompleted]; return nil; 
    }];
 }];
 // 强引用命令，不要被销毁，否则接收不到数据,这个command属性一般暴露给外界
 _conmmand = command;

//下面通常是外面调用
 [command.executionSignals subscribeNext:^(id x) { 
    [x subscribeNext:^(id x) {
         NSLog(@"网络获取的值是:%@",x);
     }];
 }];

```

```
// RACCommand高级用法 
// switchToLatest:用于signal of signals，获取signal of signals发出的最新信号,也就是可以直接拿到RACCommand中的信号 [command.executionSignals.switchToLatest subscribeNext:^(id x) {                          
    NSLog(@"%@",x);
 }];

 //executing 监听命令是否执行完毕,默认会来一次，可以直接跳过，skip表示跳过第一次信号。
 [[command.executing skip:1] subscribeNext:^(id x) { 
    if ([x boolValue] == YES) {
         // 正在执行 NSLog(@"正在执行"); 
    }else{ 
        // 执行完成 NSLog(@"执行完成");
     }
 }];

```

#### RACMuticastConnection

> RACMulticastConnection:用于当一个信号，被多次订阅时，为了保证创建信号时，避免多次调用创建信号中的block，造成副作用，可以使用这个类处理。

* 普通的订阅信号过程:

  ```
  RACSignal *signal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
     NSLog(@"发送请求");
     return nil;
  }];
  // 第一次订阅信号 
  [signal subscribeNext:^(id x) {
     NSLog(@"接收数据"); 
  }]; 
  //  第二次订阅信号 
  [signal subscribeNext:^(id x) {
     NSLog(@"接收数据");
  }]; 
  // 3.运行结果:打印两次@"发送请求"，也就是每次订阅都会发送一次请求
  ```

* RACMulticastConnection:解决重复请求问题

  ```
  RACSignal *signal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
     NSLog(@"发送请求"); 
    [subscriber sendNext:@1];
         return nil; 
    }]; 

  RACMulticastConnection *connect = [signal publish];

  // 注意：订阅信号，也不能激活信号，只是保存订阅者到数组，必须通过连接,当调用连接，就会一次性调用所有订阅者的sendNext: 
  [connect.signal subscribeNext:^(id x) { 
    NSLog(@"订阅者一信号");
  }];
  [connect.signal subscribeNext:^(id x) {
     NSLog(@"订阅者二信号"); 
  }]; 
  [connect connect];
  //运行结果:只会打印一次@"发送请求"
  ```

### RAC中信号处理的常用方法

#### RAC 核心方法绑定`bind`


对比之前的开发方式是赋值，而用RAC开发，应该把重心放在绑定，也就是在创建一个对象的时候，就绑定好以后想要做的事情，而不是等赋值之后在去做事情。
列如：把数据展示到控件上，之前都是重写控件的setModel方法，用RAC就可以在一开始创建控件的时候，就绑定好数据。

> **注意:**在开发中很少使用`bind`方法，`bind`属于RAC中的底层方法，RAC已经封装了很多好用的其他方法，底层都是调用bind，用法比bind简单.

下面例子中要实现当 _testField_ 的文字改变以后每次输出"输出:\*\*",下面的信号订阅方法底层会转换成bind方法

```objc
//使用 bind 方法
[[_textField.rac_textSignal bind:^RACStreamBindBlock{
        return ^RACStream *(id value, BOOL *stop){
            return [RACReturnSignal return:[NSString stringWithFormat:@"输出:%@",value]];
        };

    }] subscribeNext:^(id x) {
        NSLog(@"%@",x);
    }];
```

```
//直接订阅信号
[_textField.rac_textSignal subscribeNext:^(id x) { NSLog(@"输出:%@",x); }];

```

### 过滤信号的操作

#### fliter : 过滤信号\(条件过滤\)

> * 通过返回**bool**的方式控制是否接受信号传递过来的值\(控制是否调用`subscribeNext`这个block\)
> * fliter会将接受的信号通过返回的条件进行筛选,并且生成新的信号供订阅者订阅.
> * 从fliter的使用中可以看出`RAC`中对信号的操作都会生成新的信号,以便达到链式编程的目的
> * 从这点建议在对信号的操作的代码书写规范:每一次操作都应该折行,这样阅读起来更加清晰

本例中要实现当用户名长度大于3的时候输出log

```objc
[[self.usernameTextField.rac_textSignal
     filter:^BOOL(id value){
        NSString*text = value;
        return text.length > 3;
}] subscribeNext:^(id x){
        NSLog(@"%@", x);
  }];
```

#### ignore:

> 忽略某些信号的值

```
// 内部调用filter过滤，忽略掉ignore的值 [[_textField.rac_textSignal ignore:@"1"]     
    subscribeNext:^(id x) {
         NSLog(@"%@",x); 
}];
```

#### distinctUntilChanged:

> 当上一次的值和当前的值有明显的变化就会发出信号, 在开发中，刷新UI经常使用，只有两次数据不一样才需要刷新,提高性能,减少不必要的操作

```
[[_textField.rac_textSignal distinctUntilChanged] subscribeNext:^(id x) {
   NSLog(@"%@",x);
}];
```

#### take

> 从信号发出的值中取前n个

```
RACSubject *signal = [RACSubject subject]; 
[[signal take:1] subscribeNext:^(id x) {
     NSLog(@"%@",x); 
}];
[signal sendNext:@1]; 
[signal sendNext:@2];
```

#### takeLast:

> 取最后N次的信号,前提条件，由于是从信号的后面取,所以订阅者必须调用完成，因为只有完成，就知道总共有多少信号.

```
RACSubject *signal = [RACSubject subject];
[[signal takeLast:1] subscribeNext:^(id x) 
   NSLog(@"%@",x);
}];
[signal sendNext:@1];
[signal sendNext:@2];
//必须有下一步!
[signal sendCompleted];
```

#### takeUntil:\(RACSignal \*\):

> 获取信号直到某个信号执行完成

```
[_textField.rac_textSignal 
    takeUntil:self.rac_willDeallocSignal];
```

#### skip:\(NSUInteger\):

> 跳过几个信号,不接受。

```
// 表示输入第一次，不会被监听到，跳过第一次发出的信号
[[_textField.rac_textSignal skip:1] subscribeNext:^(id x) {
       NSLog(@"%@",x);
}];
```

#### switchToLatest:

> 用于signalOfSignals（信号的信号），有时候信号也会发出信号，会在signalOfSignals中，获取signalOfSignals发送的最新信号。

```
RACSubject *signalOfSignals = [RACSubject subject];
RACSubject *signal = [RACSubject subject];

// 获取信号中信号最近发出信号，订阅最近发出的信号。
// 注意switchToLatest：只能用于信号中的信号
[signalOfSignals.switchToLatest subscribeNext:^(id x) {

   NSLog(@"%@",x);
}];
[signalOfSignals sendNext:signal];
[signal sendNext:@1];
```

#### Map:

> \(映射\) map能够"加工信号传递的值”
> 
> 这里的加工信号指的是将上一个next事件传递过来的信号值x进行**加工**或者**转换**成任意的OC对象,通过block返回值返回回去,这个例子中就是将用户名加工成NSNumber值返回回去,所以下一个next事件接收到的值就是NSNumber对象.

```objc
[[[self.usernameTextField.rac_textSignal
  map:^id(NSString*text){
    return @(text.length);
  }]
  filter:^BOOL(NSNumber*length){
    return[length integerValue] > 3;
  }]
  subscribeNext:^(id x){
    NSLog(@"%@", x):
  }];

//test2
[[validPasswordSignal
map:^id(NSNumber *passwordValid){
    return[passwordValid boolValue] ? [UIColor clearColor]:[UIColor yellowColor];
}]
subscribeNext:^(UIColor *color){
    self.passwordTextField.backgroundColor = color;
}];

```

#### flatten map: 信号的映射

> flatten可以对传递过来的信号进行加工,会重新新建一个信号,block中的返回值是一个信号

```
 [[_textField.rac_textSignal flattenMap:^RACStream *(id value) {
     return [RACReturnSignal return:[NSString stringWithFormat:@"输出:%@",value]]; }]
              subscribeNext:^(id x) {
}];

```

flattenMap 和 Map 方法的区别:

* flattenMap方法中的block返回值是信号,所以是对信号的加工
* Map 方法中的block的返回值是对象,是对信号传递的变量的加工
* 一般如果传递的是对象,使用map,信号传递的是信号就用flattenMap

#### Reduce
 
> 聚合: 将多个信号发出的值进行聚合

常用用法:通常使用

```
+ (RACSignal *)combineLatest:(id<NSFastEnumeration>)signals reduce:(id (^)())reduceBlock;
```

将多个信号进行聚合,在`reduceBlock`中将信号传递的`值`进行整合.这个方法会创建一个新的信号携带整合好的值返回,每次这两个源信号的任何一个产生新值时，reduce block都会执行，block的返回值会发给下一个信号。常见应用场景就是登录界面是否满足登录条件从而控制登录按钮的状态

> 注意这个方法中的reduceBlock: 这个block是一个返回值为id类型的,也就是说我们将需要聚合的信号携带的值进行加工组合以后一定要返回一个新的对象,combineLatest::这个方法返回的信号会携带这个新对象! 我们再注意看`reduceBlock`中的参数,参数个数是不确定的,他们和信号中携带的值一一对应,为了实现这个功能RAC中专门有个`RACBlockTrampoline`类来处理这个逻辑

```

RACSignal *signalA = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {

   [subscriber sendNext:@1];

   return nil;
}];

RACSignal *signalB = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {

   [subscriber sendNext:@2];

   return nil;
}];

RACSignal *reduceSignal = [RACSignal combineLatest:@[signalA,signalB] reduce:^id(NSNumber *num1 ,NSNumber *num2){

  return [NSString stringWithFormat:@"%@ %@",num1,num2];

}];

[reduceSignal subscribeNext:^(id x) {

   NSLog(@"%@",x);
}];
```

### ReactiveCocoa操作方法之控制流操作

#### doNext:

> 执行Next之前，会先执行这个Block

 #### doCompleted:

> 执行sendCompleted之前，会先执行这个Block

```
[[[[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) { 
    [subscriber sendNext:@1];
     [subscriber sendCompleted]; 
    return nil;
 }] doNext:^(id x) { 
    // 执行[subscriber sendNext:@1];之前会调用这个Block         
    NSLog(@"doNext");; 
}] doCompleted:^{
 // 执行[subscriber sendCompleted];之前会调用这个Block 
    NSLog(@"doCompleted");; }] subscribeNext:^(id x) {     
    NSLog(@"%@",x);
 }];
```

### ReactiveCocoa中的线程操作

#### deliverOn:

> 内容传递切换到指定线程中，副作用在原来线程中,把在创建信号时block中的代码称之为副作用。

#### subscribeOn:

> 内容传递和副作用都会切换到指定线程中。

### ReactiveCocoa操作方法之**时间控制**。

#### timeout：

> 超时，可以让一个信号在一定的时间后，自动报错。

```
RACSignal *signal = [[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
     return nil; 
}] timeout:1 onScheduler:[RACScheduler currentScheduler]];

[signal subscribeNext:^(id x) {
     NSLog(@"%@",x); 
} error:^(NSError *error) { 
    // 1秒后会自动调用
     NSLog(@"%@",error); 
}];
```

#### interval

> 定时：每隔一段时间发出信号

```
[[RACSignal interval:1 onScheduler:[RACScheduler currentScheduler]] subscribeNext:^(id x) {
     NSLog(@"%@",x);
 }];
```

#### delay

> 延迟发送next。

```
 RACSignal *signal = [[[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) { 
    [subscriber sendNext:@1]; 
    return nil;
 }] delay:2] subscribeNext:^(id x) { 
    NSLog(@"%@",x);
 }];
```

### ReactiveCocoa操作方法之重复。

#### retry

> 重试 ：只要失败，就会重新执行创建信号中的block,直到成功.

#### throttle

> 节流:当某个信号发送比较频繁时，可以使用节流，在某一段时间不发送信号内容，过了一段时间获取信号的最新内容发出。

```
 RACSubject *signal = [RACSubject subject]; 
 _signal = signal; 
    // 节流，在一定时间（1秒）内，不接收任何信号内容，过了这个时间（1秒）获取最后发送的信号内容发出。
 [[signal throttle:1] subscribeNext:^(id x) {             
        NSLog(@"%@",x); 
}];
```

> References:
> 
> [MVVM+RAC英文教程](:https://www.raywenderlich.com/74106/mvvm-tutorial-with-reactivecocoa-part-1)
> 
> [http:\/\/www.jianshu.com\/p\/87ef6720a096]()
> 
> [http:\/\/www.cocoachina.com\/ios\/20150123\/10994.html]()


