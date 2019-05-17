---
title: 'NSArray,NSDictionary,NSString 与 NSData的转换'
date: 2014-12-26 12:50:53
category: iOSTips
tags:
---
## NSArray,NSDictionary,NSString 与 NSData的转换
#### NSArray -> NSData
```
 NSArray *arr = @[@1,
                             @2,
                             @{
                                 @"name": @"foo",
                                 @"age" : @20
                                 } ,
                             @"hello word"];
            NSError *error;
            NSData *arrData = [NSJSONSerialization dataWithJSONObject:arr options:kNilOptions error:&error];
            if (error) {
                NSLog(@"NSArray -> NSData error:%@",error);
            }
```
#### NSData -> NSArray

```
//NSJSONReadingMutableContainers 会将节点转成可变容器对象,例如下面会生成的 arr 是NSMutableArray类型
                 NSError *error;
                NSArray *arr = [NSJSONSerialization JSONObjectWithData:arrData options:NSJSONReadingMutableContainers error:&error];
                if (error) {
                    NSLog(@"NSData -> NSArray error:%@",error);
                }else {
                    NSLog(@"NSData -> NSArray success:%@",arr);
                }
```
#### NSDictionary -> NSData
```
NSDictionary *dic = @{
                                  @"name": @"foo",
                                  @"age": @20,
                                  @"firends": @[
                                            @"jim",
                                            @"lili"
                                          ],
                                  };
            NSError *error;
            //NSJSONWritingPrettyPrinted 会在对象结尾加换行符,格式化更好看
            NSData *dicData = [NSJSONSerialization dataWithJSONObject:dic options:NSJSONWritingPrettyPrinted error:&error];
            if (error) {
                NSLog(@"NSDictionary -> NSData error:%@",error);
            }
            
```

#### NSData -> NSDictionary 
```
NSError *error;
                NSDictionary *dic = [NSJSONSerialization JSONObjectWithData:dicData options:NSJSONReadingMutableContainers error:&error];
                if (error) {
                    NSLog(@"NSData -> NSDictionary error:%@",error);
                }else {
                    NSLog(@"NSData -> NSDictionary success:%@",dic);
                }
```

#### NSString -> NSData

```
    NSString *str = @"my name is foo, this is just for test!";
    NSData *strData = [str dataUsingEncoding:NSUTF8StringEncoding];
```

#### NSData -> NSString

```
NSString *sttr = [[NSString alloc] initWithData:strData encoding:NSUTF8StringEncoding];
```

#### 关于NSKeyedArchiver
使用NSKeyedArchiver进行转换,将NSDictionary转换成NSData对象,这个对象与我们上面利用NSJSONSerialization转换成的data是两种data,NSKeyedArchiver在转换的时候会加入一些其他的数据结构



