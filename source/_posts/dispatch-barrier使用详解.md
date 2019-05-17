---
title: dispatch_barrier使用详解
date: 2016-05-02 11:08:47
category: iOSTips
tags:
---

* 多线程开发过程中,我们如果想在前面的任务完成后再去执行后面的某些操作,我们可以采用`dispatch_barrier_async`来轻松实现

* dispatch_barrier是跟`dipatch_async/dispatch_sync` 相似的api,用来将任务提交到指定队列中,它的使用注意点:queue**必须**是`DISPATCH_QUEUE_CONCURRENT`类型的.queue如果是global或者不是`DISPATCH_QUEUE_CONCURRENT`类型的其他queue,它的作用就和普通的`dipatch_async/dispatch_sync`一样.

```c
dispatch_queue_t concurrentQueue = dispatch_queue_create("com.GCDDemo.concurrentQueue",DISPATCH_QUEUE_CONCURRENT);
    for (int i = 0; i < 3; i ++) {
        dispatch_async(concurrentQueue, ^{
            [NSThread sleepForTimeInterval:1.0];
            NSLog(@"download task %d successed!",i);
        });
    }
    dispatch_barrier_async(concurrentQueue, ^{
        [NSThread sleepForTimeInterval:2.0];
        NSLog(@"barrier blcok success!");
    });
    dispatch_async(concurrentQueue, ^{
        [NSThread sleepForTimeInterval:1.0];
        NSLog(@"upload task successed!");
    });
    //输出台:
download task 2 successed!
download task 1 successed!
download task 0 successed!
barrier blcok success!
upload task successed!
```
* 假如queue是global queue

```c
dispatch_queue_t globalQueue = dispatch_get_global_queue(0, DISPATCH_QUEUE_PRIORITY_DEFAULT);
    for (int i = 0; i < 3; i ++) {
        dispatch_async(globalQueue, ^{
            [NSThread sleepForTimeInterval:1.0];
            NSLog(@"download task %d successed!",i);
        });
    }
    dispatch_barrier_async(globalQueue, ^{
        [NSThread sleepForTimeInterval:2.0];
        NSLog(@"barrier blcok success!");
    });
    dispatch_async(globalQueue, ^{
        [NSThread sleepForTimeInterval:1.0];
        NSLog(@"upload task successed!");
    });
    //输出台:
download task 1 successed!
download task 0 successed!
download task 2 successed!
upload task successed!
barrier blcok success!
```


