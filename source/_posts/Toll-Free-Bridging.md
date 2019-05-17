---
title: Toll-Free Bridging
date: 2017-02-23 21:15:07
category: iOSTips
tags:
---


### Toll-Free Bridging

翻译自[Apple](https://developer.apple.com/library/content/documentation/General/Conceptual/CocoaEncyclopedia/Toll-FreeBridgin/Toll-FreeBridgin.html#//apple_ref/doc/uid/TP40010810-CH2).
#### 关于Toll-Free Bridging
我们日常开发中一般使用Fundation的类,其中有一些是跟Core Fundation 框架中的类是可以进行内部转换的,这个特性就被称之为`Toll-Free Bridging`,意味着你可以使用同一种数据结构作为Core Fundation 中函数的参数,或者作为Fundation中方法的参数.例如[NSLocal](https://developer.apple.com/reference/foundation/nslocale)与[CFLocal](https://developer.apple.com/reference/corefoundation/cflocale)就是Toll-Free Bridging 的.所以,如果你在使用一个需要传递(NSLocal *)类型的参数的OC方法的时候,你可以传递一个CFLocalRef类型的值进去.反过来也一样.你可以只创建以一种类型的对象s来避免编译器的警告.
例如以下的例子

```c
NSLocale *gbNSLocale = [[NSLocale alloc] initWithLocaleIdentifier:@"en_GB"];
CFLocaleRef gbCFLocale = (CFLocaleRef) gbNSLocale;
CFStringRef cfIdentifier = CFLocaleGetIdentifier (gbCFLocale);
NSLog(@"cfIdentifier: %@", (NSString *)cfIdentifier);
// logs: "cfIdentifier: en_GB"
CFRelease((CFLocaleRef) gbNSLocale);
CFLocaleRef myCFLocale = CFLocaleCopyCurrent();
NSLocale * myNSLocale = (NSLocale *) myCFLocale;
[myNSLocale autorelease];
NSString *nsIdentifier = [myNSLocale localeIdentifier];
CFShow((CFStringRef) [@"nsIdentifier: " stringByAppendingString:nsIdentifier]);
// logs identifier for current locale
```
这里需要注意的是,我们用到的内存管理语句同样是Toll-Free Bridging的,所以你可以使用CFRelease来释放Cocoa对象,也可以吧release和autorelease用在Core Fundation的对象上
但是如果使用garbage collectin 进行内存管理的话,Core Fundation 和Cocoa对象会有很大的差异.
下面有一张对照表,列出了在Core Fundation和在Fundation中是Toll-Free Bridging转换的

| Core Fundation Type | Fundation class | Availability |
| ---------  |  --------  | ------- |
|CFArrayRef | NSArray| OS X 10.0|
|CFAttributedStringRef| NSAttributedString|OS X 10.4|
|CFBooleanRef|NSNumber|OS X 10.0|
|CFCalendarRef|NSCalendar|OS X 10.4|
|CFCharacterSetRef|NSCharacterSet|OS X 10.0|
|CFDataRef|NSData|OS X 10.0|
|CFDateRef|NSDate|OS X 10.0|
|CFDictionaryRef|NSDictionary|OS X 10.0|
|CFErrorRef|NSError|OS X 10.5|
|CFLocaleRef|NSLocale|OS X 10.4|
|CFMutableArrayRef|NSMutableArray|OS X 10.0|
|CFMutableAttributedStringRef|NSMutableAttributedString|OS X 10.4|
|CFMutableCharacterSetRef|NSMutableCharacterSet|OS X 10.0|
|CFMutableDataRef|NSMutableData|OS X 10.0|
|CFMutableDictionaryRef|NSMutableDictionary|OS X 10.0|
|CFMutableSetRef|NSMutableSet|OS X 10.0|
|CFMutableStringRef|NSMutableString|OS X 10.0|
|CFNullRef|NSNull|OS X 10.2|
|CFNumberRef|NSNumber|OS X 10.0|
|CFReadStreamRef|NSInputStream|OS X 10.0|
|CFRunLoopTimerRef|NSTimer|OS X 10.0|
|CFSetRef|NSSet|OS X 10.0|
|CFStringRef|NSString|OS X 10.0|
|CFTimeZoneRef|NSTimeZone|OS X 10.0|
|CFURLRef|NSURL|OS X 10.0|
|CFWriteStreamRef|NSOutputStream|OS X 10.0|
* 注意:并不是所有的类都是Toll-Free Bridging 的,例如NSRunLoop和CFRunLoopRef就不是,NSDateFormate和CFDateFormateRef也不是.

如果foundation某一个类是Toll-Free Bridging 的,那么在该类的头文件中按住option点击类名字,就会看到对应的core foundation 的类
![](http://o7vzr7y09.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-05-19%20%E4%B8%8B%E5%8D%881.27.46.png)

#### __bridge, __bridge_transfer,__bridge_retained关键字
我们知道，ARC环境下，编译器不会自动管理CF对象的内存，我们需要手动管理。这就是我们在创建一个CF对象以后需要我们使用CFRelease将其手动释放。

```
NSLocale *gbNSLocale = [[NSLocale alloc] initWithLocaleIdentifier:@"en_GB"];
CFLocaleRef gbCFLocale = (CFLocaleRef) gbNSLocale;
CFStringRef cfIdentifier = CFLocaleGetIdentifier (gbCFLocale);
NSLog(@"cfIdentifier: %@", (NSString *)cfIdentifier);
// logs: "cfIdentifier: en_GB"
CFRelease((CFLocaleRef) gbNSLocale);
```
例如上面的例子,我们将`NSLocale *`类型的对象转换成`CFLocaleRef`的对象以后默认情况下`gbNSLocale`的内存管理任务也就从ARC交给了我们,我们需要在后面手动释放`CFRelease((CFLocaleRef) gbNSLocale);`
##### __bridge
CF和OC对象转化时只涉及对象类型不涉及对象所有权的转化

```c
NSArray *anNSArray = @[@1,@2,@3];
CFArrayRef anCFArray = (__bridge CFArrayRef)anNSArray;
NSLog(@"size of array is %li",CFArrayGetCount(anCFArray));
输出台:
//size of array is = 3
```

这里用到了`__bridge`关键字,它的意思是,虽然将NSArray类型的对象转成了CFArrayRef类型的结构,但是ARC仍然对该对象具有内存管理权限,所以我们不需要负责anCFArray的释放,也就不用添加`CFRelease(anCFArray);`

##### __bridge_transfer
常用在CF对象转化成OC对象时，将CF对象的所有权交给OC对象，此时ARC就能自动管理该内存,作用同CFBridgingRelease()

##### __bridge_retained
__bridge_retained（也可以使用CFBridgingRetain）将Objective-C的对象转换为Core Foundation的对象，同时将对象（内存）的管理权交给我们，后续需要使用CFRelease或者相关方法来释放对象；

```c
NSArray *anNSArray = @[@1,@2,@3];
CFArrayRef anCFArray = (__bridge_retained CFArrayRef)anNSArray;
NSLog(@"size of array is %li",CFArrayGetCount(anCFArray));
CFRelease(anCFArray);
输出台:
//size of array is = 3
```




