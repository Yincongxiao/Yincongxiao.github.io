---
title: YYMemeryCache-study
date: 2016-11-20 11:17:25
category: iOS源码分析
tags:
---

## YYCache  tips

> 之前YYKit刚开源的时候就粗略读过源码,当时真的是震惊,最近工作不忙,想细细读一遍,每次读作者的源码,膝盖都没有直起来过 - -~.  

### 简介

* 简单说一下YYCache的几个比较好的特性吧
  * LRU: 缓存支持 LRU \(least-recently-used\) 淘汰算法。
  * 缓存控制: 支持多种缓存控制方法：总数量、总大小、存活时间、空闲空间。
  * 兼容性: API 基本和 NSCache 保持一致, 所有方法都是线程安全的。
  * 内存缓存对象释放控制: 对象的释放\(release\) 可以配置为同步或异步进行，可以配置在主线程或后台线程进行。
  * 自动清空: 当收到内存警告或 App 进入后台时，缓存可以配置为自动清空。
  * 磁盘缓存可定制性: 磁盘缓存支持自定义的归档解档方法，以支持那些没有实现 NSCoding 协议的对象。
  * 存储类型控制: 磁盘缓存支持对每个对象的存储类型 \(SQLite/文件\) 进行自动或手动控制，以获得更高的存取性能

* 去[github](https://github.com/ibireme/YYCache)中看详细的介绍和使用方法,本文主要写自己在学习过程中的收货总结

## YYMemoryCache
* YYMemoryCache类的大体结构

![](/assets/屏幕快照 2017-03-09 下午5.14.40.png)

### 锁

* iOS中为了防止多线程对资源的抢夺,所有开发时使用锁来保证线程的安全,在[这篇文章](http://blog.ibireme.com/author/ibireme/)中作者详细介绍了iOS中几种锁的性能对比以及安全性的讨论

  ![iOS中的锁](http://blog.ibireme.com/wp-content/uploads/2016/01/lock_benchmark.png)  
  在YYCache中采用的是pthread\_mutex.

```objc
//创建一个`pthread_mutex`锁
pthread_mutex_init(&_lock, NULL);
//在锁中操作对象
pthread_mutex_lock(&_lock);
// do something safely...
pthread_mutex_unlock(&_lock)
```

### LRU淘汰算法

* **当LRU**的实现:_It uses LRU \(least-recently-used\) to remove objects; NSCache's eviction method_,在YYMemeryCache中使用了lru规则来进行缓存的淘汰,当发生内存警告或者缓存值达到上限,会优先淘汰哪些时间戳靠前的对象,最近使用的不被淘汰.YYMemoryCache中两个重要的内部对象`_YYLinkedMap`,`_YYLinkedMapNode`

##### YYLinkedMapNode
* \_YYLinkedMapNode 是缓存系统中的最小单元,是对被存储对象的一层包装,直接被`_YYLinkedMap`所持有,先来看看这个类的声明

```objc
@interface _YYLinkedMapNode : NSObject {
    @package
    __unsafe_unretained _YYLinkedMapNode *_prev; // retained by dic
    __unsafe_unretained _YYLinkedMapNode *_next; // retained by dic
    id _key; //锁存对象的key
    id _value; //具体存储的对象
    NSUInteger _cost; // 所存对象占用空间
    NSTimeInterval _time; // 最近一次使用该对象的时间戳
}
```

也就是这个对象中拥有了一个被存储对象全部的信息:key,元对象,以及在linkMap中的location,location的实现是通过持有前一个对象的指针以及后一个对象的指针来实现的

##### YYLinkedMap

* \_YYLinkedMap是实现lru的关键,它是\(\_YYLinkedMapNode \*\)的集合,通过记录集合内每个node对象的前后关系实现一个堆栈,本质是使用了CFMutableDictionaryRef来进行对象的存储,这个集合管理了对象的出栈,入栈以及排序,相关的方法依次有

```objc
// 将一个node对象插到队列最前面
- (void)insertNodeAtHead:(_YYLinkedMapNode *)node;

// 将一个node放到队列最前面
- (void)bringNodeToHead:(_YYLinkedMapNode *)node;

//移除掉指定node
- (void)removeNode:(_YYLinkedMapNode *)node;

//将最后一个个node移除
- (_YYLinkedMapNode *)removeTailNode;

//清除队列
- (void)removeAll
```
* 以上是YYMemoryCache中两个重要的类,在每次给memoryCache发送`setObject:forkey:`或者`objectForKey:`消息的时候都会更新对应的linkedMapNode对象的时间戳属性,并且把该对象放到队列的最前面,从而调整了缓存中对象的顺序.

### 内存缓存对象释放控制

```objc
     if (_releaseAsynchronously) {
            dispatch_queue_t queue = _releaseOnMainThread ? dispatch_get_main_queue() : YYMemoryCacheGetReleaseQueue();
            dispatch_async(queue, ^{
                CFRelease(holder); // hold and release in specified queue
            });
        } else if (_releaseOnMainThread && !pthread_main_np()) {
            dispatch_async(dispatch_get_main_queue(), ^{
                CFRelease(holder); // hold and release in specified queue
            });
        } else {
            CFRelease(holder);
        }
```



* 我们知道对象的创建需要分配内存空间,大量的创建对象会比较消耗性能,同样大量的对象的释放操作也是比较消耗性能的,所以在YYMemeryCache中提供了可以异步,并且选择子线程进行对象的释放的选项,这里释放操作比较巧妙我不是很理解,记录一下.   我暂时的理解是利用了block能够捕获外部变量,导致当执行到`dispatch_async(queue, ^{`虽然node已经被置为nil了,但是node对象并不会被马上释放\(被block所捕获\),等到切换到相应线程中以后对这个node对象发消息,编译器发现这个node已经被置空了,  才会马上释放该对象.

```objc
if (_lru->_totalCount > _countLimit) {
        _YYLinkedMapNode *node = [_lru removeTailNode];
        if (_lru->_releaseAsynchronously) {
            dispatch_queue_t queue = _lru->_releaseOnMainThread ? dispatch_get_main_queue() : YYMemoryCacheGetReleaseQueue();
            //node并不会马上释放,因为被block捕获了
            dispatch_async(queue, ^{
            //在这里可以实现在异步线程中释放对象?
                [node class]; //hold and release in queue
            });
        } else if (_lru->_releaseOnMainThread && !pthread_main_np()) {
            dispatch_async(dispatch_get_main_queue(), ^{
                [node class]; //hold and release in queue
            });
        }
    }
```

### 缓存上限的控制

* 在内存缓存中作者采用了轮询的方式来控制内存缓存中缓存上限,缓存个数以及过期时间,默认轮询时间是5秒,并且次轮训操作放到异步线程中,采用低优先级以获取较高的性能
* 作者定义了三个方法`- _trimToCost:`,`-_trimToCount:`,`-_trimToAge:`来分别限制最大缓存字节数,对象个数,缓存时间,我们拿其中一个来看其中的知识点

```objc
- (void)_trimToCost:(NSUInteger)costLimit {
    BOOL finish = NO;
    pthread_mutex_lock(&_lock);
    if (costLimit == 0) {
        [_lru removeAll];
        finish = YES;
    } else if (_lru->_totalCost <= costLimit) {
        finish = YES;
    }
    pthread_mutex_unlock(&_lock);
    if (finish) return;

    NSMutableArray *holder = [NSMutableArray new];
    while (!finish) {

    //pthread_mutex_trylock函数是pthread_mutex_lock函数的非阻塞版本,也可以用来加锁
    与pthread_mutex_lock的区别是:trylock如果没有获取到锁就会立刻返回不会阻塞当前线程,获取锁成功会返回0,否则返回其他值来说明锁的状态.
    但是lock如果没有获取到锁会一直等待从而发生阻塞.

    //获取锁成功后加锁
        if (pthread_mutex_trylock(&_lock) == 0) {
            if (_lru->_totalCost > costLimit) {
                _YYLinkedMapNode *node = [_lru removeTailNode];
                if (node) [holder addObject:node];
            } else {
                finish = YES;
            }
            pthread_mutex_unlock(&_lock);
        } else {
        //获取锁失败将当前线程挂起10ms
            usleep(10 * 1000); //10 ms
        }
    }
    //这里holder虽然是临时变量,超过函数{}范围后以后会被释放掉.
    //这里同样是利用了block的捕获变量能力来达到后台线程释放对象.
    if (holder.count) {
        dispatch_queue_t queue = _lru->_releaseOnMainThread ? dispatch_get_main_queue() : YYMemoryCacheGetReleaseQueue();
        dispatch_async(queue, ^{
            [holder count]; // release in queue
        });
    }
}
```

* 以上就是YYMemoryCache中的关键技术点和实现思路,从中学习到很多有用的知识,例如对象的释放选择性的放到子线程中,iru淘汰算法的实现,类之间的设计思路以及作者严谨的代码风格.以后还会分析YYDiskCache的具体实现.


