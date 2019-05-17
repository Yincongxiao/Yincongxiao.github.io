---
title: YYDiskCache study
date: 2016-11-15 10:16:23
category: iOS源码分析
tags:
---
## YYDiskCache
* 先来看指定构造器中都初始化了哪些变量.

```objc
- (instancetype)initWithPath:(NSString *)path
             inlineThreshold:(NSUInteger)threshold {
    self = [super init];
    if (!self) return nil;
//这里并不是直接初始化了一个存储对象的cache,
//而是从一个全局容器中取出与path对应的cache对象,    
    YYDiskCache *globalCache = _YYDiskCacheGetGlobal(path);
    if (globalCache) return globalCache;
    
    YYKVStorageType type;
    if (threshold == 0) {
        type = YYKVStorageTypeFile;
    } else if (threshold == NSUIntegerMax) {
        type = YYKVStorageTypeSQLite;
    } else {
        type = YYKVStorageTypeMixed;
    }
    
    YYKVStorage *kv = [[YYKVStorage alloc] initWithPath:path type:type];
    if (!kv) return nil;
    
    _kv = kv;
    _path = path;
    _lock = dispatch_semaphore_create(1);
    _queue = dispatch_queue_create("com.ibireme.cache.disk", DISPATCH_QUEUE_CONCURRENT);
    _inlineThreshold = threshold;
    _countLimit = NSUIntegerMax;
    _costLimit = NSUIntegerMax;
    _ageLimit = DBL_MAX;
    _freeDiskSpaceLimit = 0;
    _autoTrimInterval = 60;
    
    [self _trimRecursively];
    _YYDiskCacheSetGlobal(self);
    
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(_appWillBeTerminated) name:UIApplicationWillTerminateNotification object:nil];
    return self;
}
```
* 首相我们来分析第一行代码`YYDiskCache *globalCache = _YYDiskCacheGetGlobal(path);`做了什么,

```objc
static YYDiskCache *_YYDiskCacheGetGlobal(NSString *path) {
    if (path.length == 0) return nil;
    //这里进行全局的初始化操作,并且只做一次.
    _YYDiskCacheInitGlobal();
    //利用gcd的信号量机制加锁,保证线程安全.
    dispatch_semaphore_wait(_globalInstancesLock, DISPATCH_TIME_FOREVER);
    id cache = [_globalInstances objectForKey:path];
    dispatch_semaphore_signal(_globalInstancesLock);
    return cache;
}

static void _YYDiskCacheInitGlobal() {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
    //初始化锁
        _globalInstancesLock = dispatch_semaphore_create(1);
        //初始化保存chache对象的NSMapTable对象,
        _globalInstances = [[NSMapTable alloc] initWithKeyOptions:NSPointerFunctionsStrongMemory valueOptions:NSPointerFunctionsWeakMemory capacity:0];
    });
}
```
* 上面初始化中用来保存cache的容器使用到了NSMapTable对象,`NSMapTable`是在iOS6以后推出的类似NSMutableDictionary的容器类,它与NSDictionary的区别是`NSMapTable`可以指定使用什么策略来持有所存储对象的键和值,在这里选择的是key使用强引用,值使用若引用,也就是这个mapTable并不会使所存储的cache对象retainCount增加,并且当所存储的cache对象被释放后,mapTable会自动将cache移除.同时期推出的还有`NSHashTable`详细学习可以参看[这篇文章](http://www.jianshu.com/p/de71385930ba)


