---
title: dispatch_group_t 使用详解
date: 2016-05-10 09:10:48
category: iOSTips
tags:
---

*  通常我们执行耗时操作会放到子线程中并发执行,这个过程中我们可能想知道各个任务全部执行完毕的时刻,当然我们可以通过记录一些标志位等手段来达到要求,但是可能步骤会比较复杂.强大的GCD中的`dispatch_group_t`可以轻松帮我们达到目的.

##### 我们先来看group中的api
 * dispatch_group_async
      
 ```c
 //将block(任务)提交到指定的队列中,并且将次任务放到(关联)指定的group,block将异步执行.
 void dispatch_group_async(dispatch_group_t group,dispatch_queue_t queue,dispatch_block_t block);		
 //上面方法的函数版本.接受一个函数指针.
void dispatch_group_async_f(dispatch_group_t group,dispatch_queue_t queue,void *_Nullable context,dispatch_function_t work);		
 ```
 * dispatch_group_wait
 ```c
 //这个方法会同步的等,接受一个timeout,group的任务全部完成或者超时以后会返回,返回值为0代表任务完成,非0代表超时.
 long dispatch_group_wait(dispatch_group_t group,dispatch_time_t timeout);
 ```
 * dispatch_group_notify
 ```c
 //当group中的任务都完成以后会执行block.注意这句代码要加到所有任务提交之后才管用.参数queue代表block会提交到哪个队列中.
 void dispatch_group_notify(dispatch_group_t group,dispatch_queue_t queue,dispatch_block_t block);
 //上面方法的函数版本
 void dispatch_group_notify_f(dispatch_group_t group,dispatch_queue_t queue,void *_Nullable context,dispatch_function_t work);
 ```
 * dispatch_group_enter
 ```c
 //用这个方法指定一个操作将要加到group中,用来替代`dispatch_group_async`,注意它只能和`dispatch_group_leave`配对使用.
 void dispatch_group_enter(dispatch_group_t group);
 ```
 * dispatch_group_leave
 ```c
 //指定一个任务完成了.只能和`dispatch_group_enter`配对使用.
 void dispatch_group_leave(dispatch_group_t group);
 ```
 
##### 使用案例 

###### dispatch_group_async
 
 ```c
 dispatch_queue_t concurrentQueue = dispatch_queue_create("com.GCDDemo.queue",DISPATCH_QUEUE_CONCURRENT);
    dispatch_group_t group = dispatch_group_create();
    
    dispatch_group_async(group, concurrentQueue, ^{
        [NSThread sleepForTimeInterval:2.0];
        NSLog(@"first download task success! %@",[NSThread currentThread]);
    });
    
    dispatch_group_async(group, concurrentQueue, ^{
        [NSThread sleepForTimeInterval:1.0];
        NSLog(@"second download task success! %@",[NSThread currentThread]);
    });
    
    dispatch_group_notify(group, concurrentQueue, ^{
        NSLog(@"begin task three! %@",[NSThread currentThread]);
    });
    输出台:
   second download task success! <NSThread: 0x600000279c80>{number = 4, name = (null)}
 first download task success! <NSThread: 0x6080002674c0>{number = 3, name = (null)}
begin task three! <NSThread: 0x6080002674c0>{number = 3, name = (null)}
 ```
###### dispatch_group_enter & dispatch_group_leave
 * 个人感觉这种方式比`dispatch_group_async`更加灵活.比如我们可以在任务的完成回调里面写dispatch_group_leave().
 
```c
dispatch_queue_t concurrentQueue = dispatch_queue_create("com.GCDDemo.concurrentQueue",DISPATCH_QUEUE_CONCURRENT);
    dispatch_queue_t serialQueue = dispatch_queue_create("com.GCDDemo.serialQueue", DISPATCH_QUEUE_SERIAL);
    dispatch_group_t group = dispatch_group_create();
    
    dispatch_group_enter(group);
    dispatch_async(concurrentQueue, ^{
        [NSThread sleepForTimeInterval:2.0];
        NSLog(@"first download task success! %@",[NSThread currentThread]);
        dispatch_group_leave(group);
    });
    
    dispatch_group_enter(group);
    dispatch_async(serialQueue, ^{
        [NSThread sleepForTimeInterval:2.0];
        NSLog(@"second download task success! %@",[NSThread currentThread]);
        dispatch_group_leave(group);
    });
    
    dispatch_group_enter(group);
    dispatch_async(serialQueue, ^{
        [NSThread sleepForTimeInterval:2.0];
        NSLog(@"three download task success! %@",[NSThread currentThread]);
        dispatch_group_leave(group);
    });
    
    //超时处理.
    long result = dispatch_group_wait(group, dispatch_time(DISPATCH_TIME_NOW, 1.0 * NSEC_PER_SEC));
    if (result == 0) {
        NSLog(@"success");
    }else {
        NSLog(@"time out!");
    }
    
    dispatch_group_notify(group, concurrentQueue, ^{
        NSLog(@"begin task three! %@",[NSThread currentThread]);
    });
//输出台:
time out!
 first download task success! <NSThread: 0x608000277200>{number = 4, name = (null)}
second download task success! <NSThread: 0x608000270c40>{number = 3, name = (null)}
 three download task success! <NSThread: 0x608000270c40>{number = 3, name = (null)}
begin task three! <NSThread: 0x608000277200>{number = 4, name = (null)}
注意我的任务1放在了并发队列中,任务2,和任务3放在了串行队列中,所以你看到的打印顺序是任务1和任务2几乎同时打印,任务3等2秒后打印,(在任务2之后).
```

