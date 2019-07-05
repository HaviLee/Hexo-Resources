---
title: Advanced Swift
date: 2017-01-17 13:47:40
keywords: Swift
description: Built In Types
categories: 
  - Swift
tags:
  - Collections
comments: false
---

#### Arrays 和可变性

1. Array是存储**相同**数据类型的**有序集合**。

2. Array是值类型。

3. let定义不可变数组，var定义可变数组

4. ```c++
   let fib = [0, 1, 2]//fib can't change
   var mutableFibs = [0, 1, 2, 3]// mutable
   mutableFibs.append(8)
   ```

5. let修饰值(Int/Array/Struce)类型：标识这个值不能被重新赋值；但是let修饰引用类型(Func/Class/Clouse):只是代表修饰的指针不能变，但是指针指向的对象是可以改变的；

6. 在NSArray中，虽然NSArray自身没有方法改变Array，但是通过下面方式可以：

7. ```c++
   let a = NSMutableArray(array: [1, 2, 3])
   let b: NSArray = a
   a.insert(4, at:3)
   b//(1,2,3,4)
   ```

8. 正确的做饭应该使用copy复制一份，因为在OC中，a和b指向的同一个内存；

9. Swift中只有Array类型；通过var和let声明是否可变；同时不存在a和b分享同一个引用；每进行一次赋值都会进行一次copy操作；Apple使用了 **copy-on-write**技术避免复制性能问题。

#### Array 和 Optional

- Swift Array提供了常用的操作：isEmpty / count
- Swift中尽量很少直接操作Index：Swift提供其它充足的函数满足你的需求
- 如果必须使用Index，请注意不要fore-unwrap
- First & last 方法返回optional值；first 详当于：isEmpty ? nil : self[0]

#### Arrays 变换函数

##### Map：对集合中的每个元素进行某种操作

```c++
//日常代码：
var squared: [Int] = []
```

