---
title: KVO和KVC
keywords: iOS面试
date: 2019-04-27 16:47:40
categories: 
  - 面试
tags:
  - KVO和KVC
comments: true
---

# KVO面试相关

## `什么是KVO？`

- KVO是Key-Value observing的缩写。
- KVO是Objective-C对`观察者设计模式`的实现。
- App使用了isa 混写(isa-swizzing)来实现KVO。

## `isa-swizzing是如何实现KVO的？`

当我们注册一个观察者的时候，调用addobserver的时候，系统会为我们动态的创建一个NSKVONotifying_A的类，同时将原来类的isa指针指向新创建的类

## `KVO的实现机制是怎样的？`

![4-4-4](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/4-4-1.png)

## KVO代码实现

```python
#import <Foundation/Foundation.h>

@interface MObject : NSObject

@property (nonatomic, assign) int value;

- (void)increase;

@end

____________________________

#import "MObject.h"

@implementation MObject

- (id)init
{
    self = [super init];
    if (self) {
        _value = 0;
    }
    return self;
}

- (void)increase
{
    //直接为成员变量赋值
    [self willChangeValueForKey:@"value"];
    _value += 1;
    [self didChangeValueForKey:@"value"];
}

@end

```

MObserver.m

```python
#import "MObserver.h"
#import "MObject.h"
@implementation MObserver

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context{
    
    if ([object isKindOfClass:[MObject class]] &&
         [keyPath isEqualToString:@"value"]) {
        
        // 获取value的新值
        NSNumber *valueNum = [change valueForKey:NSKeyValueChangeNewKey];
        NSLog(@"value is %@", valueNum);
    }
}


@end
```

delegate使用

```python
MObject *obj = [[MObject alloc] init];
    MObserver *observer = [[MObserver alloc] init];
    
    //调用kvo方法监听obj的value属性的变化
    [obj addObserver:observer forKeyPath:@"value" options:NSKeyValueObservingOptionNew context:NULL];
   
    //通过setter方法修改value
    obj.value = 1;
    
    // 1 通过kvc设置value能否生效？
    [obj setValue:@2 forKey:@"value"];
    
    // 2. 通过成员变量直接赋值value能否生效?
    [obj increase];
```

## `重写的setter添加的方法`

必须实现下面两个方法：

- `- (void)willChangeValueForKey:(NSString *)key;`
- `- (void)didChangeValueForKey:(NSString *)key;`

![4-4-4](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/4-4-2.png)

didChangeValue会触发我们的observer回调方法。

##### `通过KVC设置value能否生效？`

```python
[obj setValue:@2 forKey:@"value"];
```

是可以的，为什么可以生效呢，下面的KVC的实现原理。就是答案。其实setValue底层是可以调用属性的setter方法的。

##### `通过成员变量直接赋值value能否生效？`

```python
_value += 1;
```

对成员变量直接赋值无法调用，因为无法触发setter方法，可以按照下面的方法改：

```python
//直接为成员变量赋值
    [self willChangeValueForKey:@"value"];
    _value += 1;
    [self didChangeValueForKey:@"value"];
```

上面的代码也是手动触发KVO的方法。`didChangeValueForKey`会触发Observer回调。



## 总结：

- 使用setter方法改变值KVO才可以生效
- 使用KVC setValue:forKey:可以使KVO生效
- 直接修改成员变量无法触发KVO，需要`手动添加KVO`才会生效。

# KVC面试相关

## `什么是KVC?`

KVC是key-value coding的缩写，一种键值编码技术。

- `-(id)valueForKey:(NSString*)key`
- `-(void)setValue:(id)value forKey:(NSString *)key`

##### `通过键值编码技术会不会破坏面向对象编程？`

vauleForKey和setValue:ForKey中的key是没有任何限制的，也就是说在我们知道一个类或者实例它内部某一个私有成员变量的名称的情况下，在外界可以通过这个key对这个成员变量修改，因此会破会。

## `valueForKey系统实现流程？`

![4-4-4](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/4-4-3.png)

系统首先根据这个key判断是否有get(访问器)方法，有的话直接调用，没有的话会判断实例变量是否存在，存在的话，可以访问，没有的话，会抛出一个找不到key的异常。accessInstanceVariablesDirectly可以控制实例变量是不是可以访问到。

##### `访问器方法是否存在的判断规则？`

Accessor Method:在获取一个和key同名或者相似的实例变量的时候，

- `<getKey>`:如果有以get开头后面接key，valueForKey
- `<key>`:直接使用key
- `<isKey>`

##### `根据key获取的成员变量是哪个？`

- _key
- _isKey
- key
- isKey

都可以满足成员变量的获取。

## `setValue:ForKey的实现流程`

![4-4-4](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/4-4-4.png)



简单来说就是`如果没有找到Set<Key>方法的话，会按照_key，_iskey，key，iskey的顺序搜索成员并进行赋值操作`。

[KVO+KVC]: https://www.jianshu.com/p/b9f020a8b4c9



