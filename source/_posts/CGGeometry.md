---
title: CGGeometry
date: 2014-09-21 14:37:56
category: iOSTips
tags:
---

#### CGGeometry
> CGGeometry可以对CGRect数据进行快速地操作,返回我们想要的结果,下面是对这个头文件的一些方法进行的使用总结:

##### 结构体
```c
//代表位置坐标
struct CGPoint {
CGFloat x;
CGFloat y;
};
typedef struct CGPoint CGPoint;
//代表大小
struct CGSize {
CGFloat width;
CGFloat height;
};
typedef struct CGSize CGSize;
//代表二维空间向量
struct CGVector {
CGFloat dx;
CGFloat dy;
};
typedef struct CGVector CGVector;
//代表一个矩形
struct CGRect {
CGPoint origin;
CGSize size;
};
typedef struct CGRect CGRect;
```
##### 常量

//CGPointZero == CGPointMake(0, 0) == (x = 0, y = 0)
CG_EXTERN const CGPoint CGPointZero
//CGSizeZero == CGSizeMake(0, 0, 0, 0) == (width = 0, height = 0)
CG_EXTERN const CGSize CGSizeZero
//CGRectZero == CGRectMake(0, 0, 0, 0) == (origin = (x = 0, y = 0), size = (width = 0, height = 0))
CG_EXTERN const CGRect CGRectZero
//空的Rect,注意它与CGRectZero的区别:当使用 CGRectIntersection(rect1, rect2);来计算两个rect的交集的时候,如果没有交集会返回CGRectNull.
CGRectNull
//无限大的矩形, 打印值为(origin = (x = -8.9884656743115785E+307, y = -8.9884656743115785E+307), size = (width = 1.7976931348623157E+308, height = 1.7976931348623157E+308))
CGRectInfinite

##### 创建方法

CG_INLINE CGPoint CGPointMake(CGFloat x, CGFloat y);
CG_INLINE CGSize CGSizeMake(CGFloat width, CGFloat height);
CG_INLINE CGVector CGVectorMake(CGFloat dx, CGFloat dy);
CG_INLINE CGRect CGRectMake(CGFloat x, CGFloat y, CGFloat width, CGFloat height);

##### 从CGRect中获取具体值
```c
CGRect rect1 = CGRectMake(5, 5, 20, 20);
CGRectGetMinX(rect1);//5
CGRectGetMidX(rect1);//15
CGRectGetMaxX(rect1);//25
CGRectGetMinY(rect1);//5
CGRectGetMidY(rect1);//15
CGRectGetMaxY(rect1);//25
CGRectGetWidth(rect1);//20
CGRectGetHeight(rect1);//20
```
##### 判断
在判断这种结构体时不能使用 == 来判断否则会报错,必须使用下列系列函数
bool CGPointEqualToPoint(CGPoint point1, CGPoint point2)
bool CGSizeEqualToSize(CGSize size1, CGSize size2)
bool CGRectEqualToRect(CGRect rect1, CGRect rect2)
//是否为空CGRectIsEmpty(CGRectZero);//YES
bool CGRectIsEmpty(CGRect rect)
//BOOL equal = CGRectEqualToRect(CGRectZero, CGRectNull);//NO
bool CGRectIsNull(CGRect rect)
//判断rect是否是无限大的.
bool CGRectIsInfinite(CGRect rect)

//用来判断两个矩形是否有交集
bool CGRectIntersectsRect(CGRect rect1, CGRect rect2)
//等同于下面:
````c
		CGRect intersection = CGRectIntersection(rect1, rect2);
		if (!CGRectIsNull(intersection)) {
		//有交集
		}else {
		//没有交集
		}
```
//判断一个点是否在矩形范围内
bool CGRectContainsPoint(CGRect rect, CGPoint point)
//判断rect1是否包含rect2: rect1和rect2的交集是否等于rect2.
bool CGRectContainsRect(CGRect rect1, CGRect rect2)
##### 操作
* void **CGRectDivide**(CGRect rect, CGRect * slice,
CGRect * remainder, CGFloat amount, CGRectEdge edge)

用来分割矩形的函数
rect:被分割的矩形,slice是"被切下来的部分",remainder是"剩余的部分",amount是在某一个轴上切割时的偏移量,edge设置垂直于哪个轴进行切割,并且从哪个方向开始切割
```c
	typedef CF_ENUM(uint32_t, CGRectEdge) {
	CGRectMinXEdge, //x轴最小值开始(从左往右)
	CGRectMinYEdge, //y轴最小值开始(从上往下)
	CGRectMaxXEdge, //x轴最大值开始(从右往左)
	CGRectMaxYEdge //y轴最大值开始(从下往上)
	};

	CGRect rect = CGRectMake(10, 10, 100, 100);
	CGRect slice, remainder;
	CGRectDivide(rect, &slice, &remainder, 10, CGRectMinYEdge);
	//sloceslice = (origin = (x = 10, y = 10), size = (width = 100, height = 10))
	//remainder = (origin = (x = 10, y = 20), size = (width = 100, height = 90))
```
* CGRect **CGRectInset**(CGRect rect, CGFloat dx, CGFloat dy)
以rect的中心为中心,宽度减少 2 * dx , 高度减少 2 * dy
```c
	CGRect rect = CGRectMake(10, 50, 100, 100);//灰色
	CGRect rect2 = CGRectInset(rect, 20, 20);//红色
```
![](/assets/屏幕快照 2017-04-21 下午2.09.15.png)

* CGRect **CGRectIntegral**(CGRect rect)
当rect的四个值有不是整数的值时,返回一个能装得下该rect的最小的整数矩形

	```c
		CGRect rect = CGRectMake(10.8, 50.8, 100.7, 100.5);
		CGRect CGRectIntegral(CGRect rect)
		CGRect rect2 = CGRectIntegral(rect);
		//rect2 = (origin = (x = 10, y = 50), size = (width = 102, height = 102))
	```
* CGRect **CGRectUnion**(CGRect r1, CGRect r2)
	r1,r2的并集.
![](/assets/屏幕快照 2017-04-21 下午2.22.13.png)

* CGRect **CGRectIntersection**(CGRect r1, CGRect r2)
r1,r2 交集,如果没有重叠部分返回null rect
* CGRect **CGRectOffset**(CGRect rect, CGFloat dx, CGFloat dy)
rect按照dx,dy平移

* CGRectCreateDictionaryRepresentation(rect);
返回一个包含rect信息的字典

```c
		CGRect rect = CGRectMake(10, 50, 100, 100);
		CFDictionaryRef dic = CGRectCreateDictionaryRepresentation(rect);
		{
		Height = 100;
		Width = 100;
		X = 10;
		Y = 50;
		}
```



