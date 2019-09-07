---
title: 码哥底层探究-KVO
date: 2018-07-18 13:47:40
categories: [iOS]
tags: [iOS 底层原理]
---

# KVO

## 定义

- KVO的全称是Key-Value Observing，俗称“键值监听”，可以用于监听某个对象属性值的改变。

![image](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/08292140.png)

## 本质

### 未使用KVO的方法调用

![image](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/09012120.png)

### 使用KVO的方法调用

![image](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/09012203.png)

NSKVONotifiying_MJPerson中的setAge方法会调用系统Foundation的_NSSetIntValueAndNotify方法；

#### _NSSetIntValueAndNotify

**函数内部函数逻辑：**

- 调用willChangeValueForKey:
- 调用原来的setter实现
- 调用didChangeValueForKey:

- didChangeValueForKey:内部会调用observer的observeValueForKeyPath:ofObject:change:context:方法

函数实现的伪代码：

```objc
NSKVONotifying_MJPerson中的setAge方法：
- (void)setAge:(int)age {
  _NSSetIntValueAndNotify();
}
//方法的伪代码
void _NSSetIntValueAndNotify()
{
  [self willChangeValueForKey:@"age"];
  [super setAge:age];
  [self didChangeValueForKey:@"age"];
}
//
- (void)didChangeValueForKey:(NSSring*)key
{
  //通知监听器，某某值发生变化
  [observer observeValueForKeyPath:key ofObject:self change:nil context:nil];
}
```

### 代码验证

- KVO前后类对象的变化

  添加前后实例对象的isa指向发生了变化

  ```objc
  Interview01[43003:6563529] person1添加KVO监听之前 - MJPerson MJPerson
  Interview01[43003:6563529] 类对象 - NSKVONotifying_MJPerson MJPerson
  ```

- KVO前后set方法调用的变化

  添加KVO后setAge方法在_NSSetIntValueAndNotify中调用

  ```objc
  2019-09-01 22:55:09.662039+0800 Interview01[41890:6553632] person1添加KVO监听之前 - 0x1040ce630 0x1040ce630
  2019-09-01 22:55:15.984746+0800 Interview01[41890:6553632] person1添加KVO监听之后 - 0x104427cf2 0x1040ce630
  (lldb) p (IMP)0x1040ce630
  (IMP) $0 = 0x00000001040ce630 (Interview01`-[MJPerson setAge:] at MJPerson.m:13)
  (lldb) p (IMP)0x104427cf2
  (IMP) $1 = 0x0000000104427cf2 (Foundation`_NSSetIntValueAndNotify)
  (lldb) 
  ```

- KVO前后元类对象变化

  ```objc
  NSLog(@"元类对象 - %@ %@",
            object_getClass(object_getClass(self.person1)), // self.person1.isa.isa
            object_getClass(object_getClass(self.person2))); // self.person2.isa.isa
  
  ////////
  Interview01[43421:6566871] 元类对象 - NSKVONotifying_MJPerson MJPerson
  
  ```

  就是说NSKVONotifying_MJPerson类对象的isa指向自己的元类对象。

### 重写class方法

NSKVONotifying_MJPerson重写class方法，导致使用KVO的实例对象直接调用class方法返回的就是MJPerson。

**苹果为了屏蔽了KVO的内部实现，隐藏了NSKVONotifying_MJPerson的重载。** **可以使用runtime方法 `object_getClass(id *)`获得真正的类对象。**

验证NSKVONotifying_MJPerson中的方法：

```objective-c
- (void)printMethodNamesOfClass:(Class)cls
{
    unsigned int count;
    // 获得方法数组
    Method *methodList = class_copyMethodList(cls, &count);
    
    // 存储方法名
    NSMutableString *methodNames = [NSMutableString string];
    
    // 遍历所有的方法
    for (int i = 0; i < count; i++) {
        // 获得方法
        Method method = methodList[i];
        // 获得方法名
        NSString *methodName = NSStringFromSelector(method_getName(method));
        // 拼接方法名
        [methodNames appendString:methodName];
        [methodNames appendString:@", "];
    }
    
    // 释放
    free(methodList);
    
    // 打印方法名
    NSLog(@"%@ %@", cls, methodNames);
}
```

**使用person->name**对成员变量直接赋值无法触发KVO，改进：

```objc
[self.person1 willChangeValueForKey:@"age"];
self.person1->_age = 2;
[self.person1 didChangeValueForKey:@"age"];
```

# KVC

## 定义

- KVC的全称是Key-Value Coding，俗称“键值编码”，可以通过一个key来访问某个属性；

```objc
常见的API有
- (void)setValue:(id)value forKeyPath:(NSString *)keyPath;
- (void)setValue:(id)value forKey:(NSString *)key;
- (id)valueForKeyPath:(NSString *)keyPath;
- (id)valueForKey:(NSString *)key; 
```

## setValue:forKey:的原理

![image](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/09022245.png)



## valueForKey:的原理

![image](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/09022246.png)

