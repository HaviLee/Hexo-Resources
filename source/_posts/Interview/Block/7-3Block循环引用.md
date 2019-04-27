---
title: Block循环引用
keywords: iOS面试
date: 2019-04-29 18:47:40
categories: 
  - 面试
tags:
  - block
comments: true
---

先看例子：

```c++
{
  _array = [NSMutableArray arrayWithObject:@"block"];
  _strBlk = ^NSString*(NSString * num) {//block一般使用copy属性修饰
    return [NSString stringWithFormat:@"helloc_%@",_array[0]];
  }
  _strBlk(@"hello");

}
```

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/7-2-11.png)

> 上面的问题造成自循环引用。由于当前的对象通过copy关键字对block进行了copy操作，所以当前对象对于block具有强引用。
>
> Block表达式中使用了array成员变量，而array是由strong修饰的，因此block也会捕获strong修饰符。因此block持有self

解决：

```python
{
  _array = [NSMutableArray arrayWithObject:@"block"];
  __weak NSArray* weakArr = _array;
  _strBlk = ^NSString*(NSString * num) {//block一般使用copy属性修饰
    return [NSString stringWithFormat:@"helloc_%@",weakArray[0]];
  }
  _strBlk(@"hello");

}
```

**__block引起的循环引用**

```c++
{
  __block MCBlock* blockSelf = self;
	_blk = ^int(int num) {
  	return num * blockSelf.var;
	};
  _blk(3);
}

```

- 在MRC情况下没有问题。
- 在ARC下会产生循环引用，引起循环引用。

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/7-2-12.png)

> 首先对象持有Block,而根据`__block`修饰的变量会截获到Block里面，block里面是会对这个变量进行强持有，而第一行的定义又使得`__block`变量强持有self对象，因此形成大环引用。

解决方案一:避免循环引用



```c++
{
  __block MCBlock* blockSelf = self;
	_blk = ^int(int num) {
    int result = num * blockSelf.var;
    blockSelf = nil;
  	return result;
	};
  _blk(3);
}
//弊端：如何block一直没有调用，就导致这个循环无法断开。
```

# 总结：

1. 什么是Block?

2. 为什么block会产生循环引用？

   A:1.如果当前Block对当前对象的某一成员变量截获的话，block会对当前对象的成员变量有一个强引用，而当前对象也对这个block进行强引用，因此造成了自循环引用。可以使用`__weak`进行消除自循环引用。2.如果使用`__block`造成了循环引用，在MRC下不会产生循环引用，在ARC下会产生循环引用。

3. 怎样理解Block截获变量的特性？

4. 你都遇到哪些循环引用？你又如何解决的？

   A:Block引起的循环引用有两方面的，一个是自循环引用，另一种是__block引起的循环引用。





