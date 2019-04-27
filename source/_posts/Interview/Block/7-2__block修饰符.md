---
title: __block修饰符
keywords: iOS面试
date: 2019-04-29 17:47:40
categories: 
  - 面试
tags:
  - block
comments: true
---

# __block修饰符

`Q：在什么场景下需要使用__block修饰符？`

A：`一般情况下`，对被截获的变量进行`赋值操作`的时候需要添加`__block修饰符`。

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/7-2-1.png)

> 上面的重点是，赋值操作，而不是使用操作；
>
> 看下面的坑：
>
> ```c++
> {
>   NSMutableArray *array = [NSMutableArray array];
>   void (^Block) (void) = ^{
>     [array addObject:@123];
>   };
>   Block();
> }
> /*
> 上面是一个典型的坑，这里array只是被使用了，而没有进行赋值操作，因此不需要使用__block修饰
> */
> ```
>
> 对比：
>
> ```c++
> {
>   NSMutableArray *array = nil;
>   void (^Block) (void) = ^{
>     array = [NSMutableArray array];
>   };
>   Block();
> }
> /*
> 这种情况下就需要添加__block修饰符。
> */
> ```

**`Q：使用__block修饰后，对变量进行赋值时，它的结构特点`：**

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/7-2-2.png)

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/7-2-3.png)

> - 对于全局变量&静态全局变量，block不会捕获他们。
> - 对于静态局部变量，是以指针的形式捕获。

`Q：笔试题：`

```c++
{
  __block int multipiler = 6;
  int(^Block)(int) = ^int(int num) {
    return num * multipiler;
  };
  multipiler = 4;
  NSLog(@"result is %d",Block(2));
}
//result is 8;
```

> `__block修饰的变量变成了对象`；
>
> ![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/7-2-4.png)
>
> 经过__block修饰之后，int类型转换为一个结构体(对象，有isa指针)；
>
> ![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/7-2-5.png)
>
> 而在后面程序block后面中使用multipiler=4的时候，multipiler此时已经是一个对象，实际背后是mulitipiler.__forwarding->mulitiper = 4进行赋值。

`__forwarding指针是用来做什么？`

# Block内存管理

`Block的类型？`

- _NSConcrete`Global`Block:全局
- _NSConcrete`Stack`Block：栈
- _NSConcrete`Malloc`Block：堆

`不同的Block在内存的分配情况`

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/7-2-6.png)

## Block的copy操作

`对Block进行copy有什么效果？`

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/7-2-7.png)

> - 在栈上声明对象的一个成员变量为block，@property(assign) Block;如果后面栈退出，但是通过成员变量访问这个block就会造成crash.

`栈上Block的销毁？`

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/7-2-8.png)

> - 在变量的作用域或者函数的作用域退出后，__block变量和block都会销毁。

`栈上的block copy后效果？`

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/7-2-9.png)

> - 堆上的block和__block变量在变量作用域结束后仍存在。

`Q：当我们对栈上的Block进行copy之后，在MRC情况是否会产生内存泄露？`

A:进行copy后，堆上的block没有其它对象引用，和你alloc一个对象，没有release一样的。

`栈上__block变量的copy的结果？`

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/7-2-10.png)

> 栈上的__forwarding指针发生了变化，如图；
>
> 比如在栈上的`__block`的变量做修改，如果block做了copy操作，此时我们对 `__block`变量进行修改，实际上我们修改的不是栈上的`__block`变量，实际是根据`__forwarding`指针修改的是堆上的__block变量。
>
> 只要栈上的block经过copy后，栈上和堆上的__block修饰的变量修改的话，都是发生在堆上。

## __forwarding总结

```c++
{
  __block int multipiler = 10;
  _blk = ^int(int num) {
    //_blk是一个成员变量，对_blk进行赋值，实际进行对赋值的block进行了copy操作，就会复制到堆上，
    return num * multipiler;
  };//在栈上修改，
  multipiler = 6;//不是对上面的mulitipiler赋值，而是对block里面的mulitipiler.__forwarding->mulitiper = 6；即修改了block捕获的值。//进行copy操作后，修改的其实是堆上的__forwarding中的multipiler值。
  [self execteBlock];
}

- (void)execteBlock
{
	int result = _blk(4);
	NSLog(@"result is %d",result);
}
//result is 24.
```

`__forwarding存在的意义`

- 不论在任何内存位置，都可以顺利的访问同一个__block变量。
- 栈上的block变量经过copy后，在栈上和堆上访问__block变量都是访问的堆上的。