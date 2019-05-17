---
title: 初始化方法+ load 和+initialize方法
date: 2015-07-20 10:12:34
category: iOSTips
tags:
---

有时候我们希望类先执行某些一次性的初始化操作再使用,NSObject根类中有两个可以实现这种初始化操作的方法,这就是`+Load`和`+initailze`方法
#### +Load
##### 调用时机
对于加入运行期系统的每个类以及它的分类来说,必定会调用此方法,而且只会被调用一次,通常是在应用程序启动的时候,执行时机在main函数之前!并且先调用父类的+load再调用子类的.

```c
@implementation FatherClass
+(void)load {
    NSLog(@"%s",__func__);
}
@end

@interface SonClass : FatherClass
@end
@implementation SonClass
+(void)load {
    NSLog(@"%s",__func__);
}
@end
//输出台:
+[FatherClass load]
+[SonClass load]
```
如果分类中也实现了该方法,那么先调用本类的再调用分类的
```c
@implementation FatherClass
+(void)load {
     NSLog(@"%s,%@",__func__,self);
}
@end
@implementation FatherClass (category)
+(void)load {
    NSLog(@"%s,%@",__func__,self);
}
@end
输出台:
//+[FatherClass load],FatherClass
//+[FatherClass(category) load],FatherClass
```
如果两个没有继承关系的类都实现了+load方法,那么它的调用顺序取决于谁先被加到运行期环境中
![](/assets/屏幕快照 2017-04-06 上午9.13.34.png)
上图中的AnyObject类与FatherClass类都继承自NSObject,但是FatherClass先被加入进运行期环境,所以它的+load方法会先被执行.
```c
输出台:
+[FatherClass load]
+[SonClass load]
+[AnyObject load]
```
##### 使用注意点:
* **在+load的调用时机,系统还处于"脆弱"状态**,虽然系统的库已经被加载进运行期系统,但是我们自己编写的类,或者引用的其他的类库中的类不一定已经可以使用,所以在+load中要尽量避免初始化其他的对象. 比如下面的代码就是不安全的
```c
@implementation FatherClass
+(void)load {
    NSLog(@"%s",__func__);
    AnyObject *anyObject = [AnyObject new];
//    use anyObject...
}
@end
```
当然AnyObject这个类使我们自己写的,我们可能通过Complie Sources知道它加载的顺序(这不是一个好办法),但是是用其他类库我们就不得而知.如果恰好在AnyObject中使用了+load方法来进行某些初始化操作来赋予这个类某些特性,并且这个类被载入的晚,那么这就有问题了.
* **+load方法不像普通的方法那样遵循继承规则**,如果一个类本身没有实现+load方法,那么无论其各级超类是否实现此方法系统都不会调动.这句话应该这样理解:正常我们给一个对象或者类发消息,如果这个对象(或类)本身没有实现该方法,那么系统会通过isa指针找到父类的实现.但是+load方法不同,子类如果没有实现该方法那么也不会去父类中找.也就是说你实现了系统就调用,你没实现就算了.但是如果在+load中显式的调用[super load];那么就会去调用父类方法了.

```c
//普通方法,子类实现
@implementation FatherClass
- (void)eat {
     NSLog(@"%s,%@",__func__,self);
}
@end
@implementation SonClass
- (void)eat {
  NSLog(@"%s,%@",__func__,self);
}
@end
SonClass *son = [SonClass new];
[son eat];
输出台:
//-[SonClass eat],<SonClass: 0x600000017970>

//子类未实现
@implementation FatherClass
- (void)eat {
     NSLog(@"%s,%@",__func__,self);
}
@en
@implementation SonClass
@end
SonClass *son = [SonClass new];
[son eat];
输出台:
//-[FatherClass eat],<SonClass: 0x600000001600>

//+load方法,子类实现
@implementation FatherClass
+(void)load {
     NSLog(@"%s,%@",__func__,self);
}
@end
@implementation SonClass
+(void)load {
    NSLog(@"%s,%@",__func__,self);

@end
输出台:
//+[FatherClass load],FatherClass
//+[SonClass load],SonClass

//子类未实现
@implementation FatherClass
+(void)load {
     NSLog(@"%s,%@",__func__,self);
}
@end
@implementation SonClass
@end
输出台:
//+[FatherClass load],FatherClass
```
* **在+load方法中的实现务必精简**,尽量减少里面所执行的操作,因为整个应用在执行+load方法时都会阻塞,如果在+load中进行繁杂的代码,那么应用程序在执行期间就会变得无响应,不要调用可能会加锁的方法.实际上但凡是通过+load方法实现的某些任务,基本上都做得不对,真正的用途仅在于调试程序,比如可以再分类中实现+load来看该分类是否已经正确载入系统中.
 ####+initialize
 #####调用时机
对于每个类来说,该方法会在程序第一次使用该类或者该类的子类时被调用,并且只会调用一次.如果子类没有实现,那么会调用父类的该方法

```c
//子类实现
@implementation FatherClass
+ (void)initialize {
    NSLog(@"%s,%@",__func__,self);
}
- (void)eat {
     NSLog(@"%s,%@",__func__,self);
}
@end
@implementation SonClass
+ (void)initialize {
    NSLog(@"%s,%@",__func__,self);
}
@end

SonClass *son = [SonClass new];
[son eat];
输出台:
//+[FatherClass initialize],FatherClass
//+[SonClass initialize],SonClass

//子类不实现
@implementation FatherClass
+ (void)initialize {
    NSLog(@"%s,%@",__func__,self);
}
- (void)eat {
     NSLog(@"%s,%@",__func__,self);
}
@end
@implementation SonClass
@end

SonClass *son = [SonClass new];
[son eat];
输出台:
//+[FatherClass initialize],FatherClass
//+[FatherClass initialize],SonClass
```
我们发现子类如果实现了就走子类的方法,子类没有实现就走父类的方法.这与普通的方法是相同的,都遵循集成规则,这个与+load不同.
那我们如果不想因为子类而调用到父类的方法该怎么办呢?
```c
@implementation FatherClass
+ (void)initialize {
    if (self == [FatherClass class]) {
        NSLog(@"%s,%@",__func__,self);
    }
}
@end
输出台
//+[FatherClass initialize],FatherClass
```
##### +load与+initalize方法的区别
* +initalize 是惰性调用,只有当给该类或者该类的派生类被使用时才会被调用.
* +load方法,应用会阻塞并等待所有类的+load执行完才会继续执行.
* +initalize方法是线程安全的.所以不用担心对该类第一次发消息的线程问题.
* +load不遵循继承规则
* +load方法运行环境不是安全的,但是+initalize方法运行时可以调用任何类的任何方法;

