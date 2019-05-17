---
title: GCD-dispatch_source
date: 2016-07-02 11:08:47
category: iOSTips
tags:
---
# GCD-dispatch_source
### 简单介绍
 `dispatch_source` 并不像`queue`,`async`等常见, 它是GCD中一种强大的事件 api 
它是BSD系内核惯有功能[kqueue](https://en.wikipedia.org/wiki/Kqueue)的包装,它可以用来监听系统或者我们自定义的各种事件,它的强大之处是无论在任何线程上都可以通过函数`dispatch_source_merge_data()`来执行之前设置的回调句柄(理解为block)
### 使用
### 创建事件源
先看代码:

```
- (void)viewDidLoad {
    [super viewDidLoad];
    _source = dispatch_source_create(DISPATCH_SOURCE_TYPE_DATA_ADD, 0, 0, dispatch_queue_create("myqueue", DISPATCH_QUEUE_CONCURRENT));
    dispatch_resume(_source);
    dispatch_source_set_event_handler(_source, ^{
        unsigned long receive = dispatch_source_get_data(_source);
        NSLog(@"%zd",receive);
    });
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        for (int i = 0; i < 10; i ++) {
            dispatch_source_merge_data(_source, 2);
        }
    });
}
```
这里我们通过`dispatch_queue_create()`创建了一个`source`这个函数接受四个参数

* dispatch_source_type_t type //事件类型
* uintptr_t handle, //可以理解为句柄、索引或id，假如要监听进程，需要传入进程的ID
* unsigned long mask //可以理解为描述，提供更详细的描述，让它知道具体要监听什么

* dispatch_queue_t _Nullable queue //自定义源需要的一个队列，用来处理所有的响应句柄（block）

chat: dispatch_source的事件类型.

|type类型  |用途  |  
| --- | --- | 
| DISPATCH_SOURCE_TYPE_DATA_ADD | 自定义的事件，变量增加 |
| DISPATCH_SOURCE_TYPE_DATA_OR | 自定义的事件，变量OR |
| DISPATCH_SOURCE_TYPE_MACH_SEND | MACH端口发送 |
| DISPATCH_SOURCE_TYPE_MACH_RECV | MACH端口接收 |
| DISPATCH_SOURCE_TYPE_PROC |进程监听,如进程的退出、创建一个或更多的子线程、进程收到UNIX信号  |
| DISPATCH_SOURCE_TYPE_READ | IO操作，如对文件的操作、socket操作的读响应 |
|DISPATCH_SOURCE_TYPE_SIGNAL| 接收到UNIX信号时响应 |
| DISPATCH_SOURCE_TYPE_TIMER |  定时器
 |
| DISPATCH_SOURCE_TYPE_VNODE |文件状态监听，文件被删除、移动、重命名|
| DISPATCH_SOURCE_TYPE_WRITE | IO操作，如对文件的操作、socket操作的写响应 |
还有一点要注意的是,默认我们创建的source是被suspend的,所以我们需要手动的调用`dispatch_resume(source)`恢复它.

`dispatch_source_set_event_handler()`函数用来设置souce事件触发的回调,它接受两个参数,我们设置的source和一个会掉的block

`dispatch_source_merge_data(_source, 2);`来手动触发source的事件源,第二个参数不能是 <= 0 的数, 否则不能得到回调.

`dispatch_source_get_data()`得到事件源传递来的数据.

如果我们使用`DISPATCH_SOURCE_TYPE_DATA_ADD`类型的事件源,并且我们*在短时间内*频繁的调用`dispatch_source_merge_data()`函数,之前注册的事件源回调并不会相应的调用多次,而只是调用一次,并且`dispatch_source_get_data()`得到的结果是多次的参数和20.
但是如果中间间隔时间长的话就会多次调用,每次传递值是2.

其他有用的函数还有:

| func | 用途|
| --- | --- |
|voird dispatch_source_set_cancel_handler(dispatch_source_t source,dispatch_block_t _Nullable handler) | dispatch源取消时调用的block，一般用于关闭文件或socket等，释放相关资源 |
|long dispatch_source_testcancel(dispatch_source_t source) | 检测是否dispatch源被取消，如果返回非0值则表明dispatch源已经被取消|
|void dispatch_source_set_registration_handler(dispatch_source_t source, dispatch_block_t registration_handler);|可用于设置dispatch源启动时调用block，调用完成后即释放这个block。也可在dispatch源运行当中随时调用这个函数|
| dispatch_suspend(queue) |  挂起queue|
|dispatch_resume(source)|分派源创建时默认处于暂停状态，在分派源分派处理程序之前必须先恢复|
|uintptr_t dispatch_source_get_handle(dispatch_source_t source)| 得到dispatch源创建，即调用dispatch_source_create的第二个参数 |
| unsigned long dispatch_source_get_mask(dispatch_source_t source) | 得到dispatch源创建，即调用dispatch_source_create的第三个参数 |
|得到dispatch源创建，即调用dispatch_source_create的第三个参数| 取消dispatch源的事件处理--即不再调用block。如果调用dispatch_suspend只是暂停dispatch源。|

### 实践
倒计时label

```
@implementation TimerLabel {
    dispatch_source_t _timer;
}

- (instancetype)initWithTimeInterval:(NSUInteger)interval {
    if (self = [super init]) {
        self.numberOfLines = 1;
        __weak CountLabel *weakSelf = self;
        __block NSUInteger timeout = interval;
        dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
        _timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0,queue);
        dispatch_source_set_timer(_timer,dispatch_walltime(NULL, 0),1.0*NSEC_PER_SEC, 0);
        dispatch_source_set_event_handler(_timer, ^{
            if(timeout<=0){
                dispatch_source_cancel(_timer);
                dispatch_async(dispatch_get_main_queue(), ^{
                    weakSelf.text = @"ok now";
                });
            }else{
                NSUInteger seconds = timeout % (timeout + 1);
                NSString *strTime = [NSString stringWithFormat:@"剩余:%.2lu s", seconds];
                dispatch_async(dispatch_get_main_queue(), ^{
                    [UIView beginAnimations:nil context:nil];
                    [UIView setAnimationDuration:1];
                    weakSelf.text = strTime;
                    [weakSelf sizeToFit];
                    [UIView commitAnimations];
                });
                timeout--;
            }
        });
    }
    return self;
}

- (void)start {
    dispatch_resume(_timer);
}

@end

```



