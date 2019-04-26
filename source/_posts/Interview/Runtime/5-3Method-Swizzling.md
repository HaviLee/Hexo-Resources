---
title: Method-Swizzling
keywords: iOS面试
date: 2019-04-28 10:47:40
categories: 
  - 面试
tags:
  - Runtime
comments: true
---

`什么是method-swizzling?`

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/5-3-1.png)

> 简单来说就是修改两个选择器的方法实现。

`代码实现`

```
#import <Foundation/Foundation.h>

@interface RuntimeObject : NSObject

- (void)test;

- (void)otherTest;

@end
```



```python
@implementation RuntimeObject

+ (void)load
{
    Method test = class_getInstanceMethod(self, @selector(test));
    Method otherTest = class_getInstanceMethod(self, @selector(otherTest));
    method_exchangeImplementations(test, otherTest);
}

- (void)test {
    NSLog(@"test");
}

- (void)otherTest
{
    //这个self 实际调用test
    [self otherTest];
    NSLog(@"otherTest");
}

@end
```

