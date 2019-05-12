---
title: Block本质
keywords: iOS面试
date: 2019-04-29 16:47:40
categories: 
  - 面试
tags:
  - block
comments: true
---

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/7-1-0.png)

# Block的本质

`Q：什么是Block?`

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/7-1-1.png)

A：Block是将`函数`及其`执行上下文`封装起来的`对象`。

```python
{
  int multiplier = 6;
  int(^Block)(int) = ^int(int num) {
    return num * multiplier;
  };
  Block(2);
}
```

编译器是如何处理Block呢？

**源码解析：**

<u>使用【`clang -rewrite-objc file.m`】查看编译后的文件内容</u>

来查看反编译的代码：

```python
//I标识是instance方法，MCBlock类名，method方法名
static void _I_MCBlock_method(MCBlock * self, SEL _cmd) {
    int multiplier = 6;
  	/*
  	int(^Block)(int) = ^int(int num) {
      return num * multiplier;
  	};
  	*/
  	/*
  	__MCBlock__method_block_impl_0是一个结构体
  	__MCBlock__method_block_func_0函数指针
 		__MCBlock__method_block_desc_0_DATAblock信息
  	局部变量
  	*/
    int(*Block)(int) = ((int (*)(int))&__MCBlock__method_block_impl_0((void *)__MCBlock__method_block_func_0, &__MCBlock__method_block_desc_0_DATA, multiplier));
  	/*
  	Block(2);
  	*/
  	//通过函数指针取到了函数体
    ((int (*)(__block_impl *, int))((__block_impl *)Block)->FuncPtr)((__block_impl *)Block, 2);

}
```

**__MCBlockmethod_block_impl_0**

```python
struct __block_impl {
  void *isa;//isa指针，这个是对象的标志
  int Flags;
  int Reserved;
  void *FuncPtr;//函数指针
};

////////////////////

struct __MCBlock__method_block_impl_0 {
  struct __block_impl impl;
  struct __MCBlock__method_block_desc_0* Desc;
  int multiplier;
  //c++结构体的一个构造函数，
  __MCBlock__method_block_impl_0(void *fp, struct __MCBlock__method_block_desc_0 *desc, int _multiplier, int flags=0) : multiplier(_multiplier) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

////////////////////
static int __MCBlock__method_block_func_0(struct __MCBlock__method_block_impl_0 *__cself, int num) {
  int multiplier = __cself->multiplier; // bound by copy


        return num * multiplier;
    }

```

`什么是`Block调用？

A：Block调用即是函数调用。

# Block截获变量

看代码：

```python
{
    int multiplier = 6;
    int(^Block)(int) = ^int(int num)
    {
//        __block
        return num * multiplier;
    };
    multiplier = 4;
    NSLog(@"result is %d", Block(2)); result = 12;
}
```

**被截获变量的类型**

- 局部变量	
  1. 基本数据类型
  2. 对象类型
- 静态局部变量
- 全局变量
- 静态全局变量

`Q:关于Block的截获特性你是否有了解，或者Block的变量截获是什么？`

这道题应该从被截获变量类型为入口。

- 对于`基本数据`类型的`局部变量`截获其值。
- 对于`对象类型的局部变量`，连通`所有权修饰符`一起截获。
- 以`指针形式`截获局部静态变量。
- `不截获`全局变量、静态全局变量。

**使用命令【clang -rewrite-objc -fobjc-arc file.m】**

-fobjc-arc:编译的结果有变化。当变量在block中被使用到的时候，就成为变量被block截获。

```python
@implementation MCBlock

// 全局变量
int global_var = 4;
// 静态全局变量
static int static_global_var = 5;

- (void)method
{
    //基本数据类型的局部变量
    int var = 1;
    //对象类型的局部变量
    __unsafe_unretained id unsafe_obj = nil;
    __strong id strong_obj = nil;
    //局部静态变量
    static int static_var = 3;

    void(^Block)(void) = ^
    {
        NSLog(@"局部变量<基本数据类型> var=%d",var);
        NSLog(@"局部变量<__unsafe_unretained 对象类型> var=%@",unsafe_obj);
        NSLog(@"局部变量<__strong 对象类型> var=%@",strong_obj);
        NSLog(@"静态变量 %d",static_var);
        NSLog(@"全局变量 %d",global_var);
        NSLog(@"静态全局变量 %d",static_global_var);
    };
    Block();
}

@end

2019-04-27 08:11:06.060816+0800 Block[72728:10526959] 局部变量<基本数据类型> var=1
2019-04-27 08:11:06.061044+0800 Block[72728:10526959] 局部变量<__unsafe_unretained 对象类型> var=(null)
2019-04-27 08:11:06.061179+0800 Block[72728:10526959] 局部变量<__strong 对象类型> var=(null)
2019-04-27 08:11:06.061303+0800 Block[72728:10526959] 静态变量 3
2019-04-27 08:11:06.061416+0800 Block[72728:10526959] 全局变量 4
2019-04-27 08:11:06.061533+0800 Block[72728:10526959] 静态全局变量 5
```

我们查看下使用clang编译成c++的MCBlock代码：

```c++
// @implementation MCBlock
int global_var = 4;

static int static_global_var = 5;

//block表达式
struct __MCBlock__method_block_impl_0 {
  struct __block_impl impl;
  struct __MCBlock__method_block_desc_0* Desc;
  //截获的局部变量
  int var;
  //连同所有权修饰符一起截获
  __unsafe_unretained id unsafe_obj;
  __strong id strong_obj;
  //以指针形式截获静态局部变量
  int *static_var;
  //对全局变量，静态全局变量不截获
  
  /*
  看下面的构造函数：var(_var), unsafe_obj(_unsafe_obj), strong_obj(_strong_obj), static_var(_static_var)，var是在直接传递给的
  主要是这个block的构造函数的参数是以上面的格式。
  */
  __MCBlock__method_block_impl_0(void *fp, struct __MCBlock__method_block_desc_0 *desc, int _var, __unsafe_unretained id _unsafe_obj, __strong id _strong_obj, int *_static_var, int flags=0) : var(_var), unsafe_obj(_unsafe_obj), strong_obj(_strong_obj), static_var(_static_var) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

//Block的结构体的构造实现

```c++
static void _I_MCBlock_method(MCBlock * self, SEL _cmd) {

    int var = 1;

    __attribute__((objc_ownership(none))) id unsafe_obj = __null;
    __attribute__((objc_ownership(strong))) id strong_obj = __null;

    static int static_var = 3;

  	//在这里创建了block.可以看到局部变量是直接传递(截获)；对象类型截获所有权修饰符，静态局部变量以指针形式。
    void(*Block)(void) = ((void (*)())&__MCBlock__method_block_impl_0((void *)__MCBlock__method_block_func_0, &__MCBlock__method_block_desc_0_DATA, var, unsafe_obj, strong_obj, &static_var, 570425344));
    ((void (*)(__block_impl *))((__block_impl *)Block)->FuncPtr)((__block_impl *)Block);
}
```

[Block本质]: https://www.jianshu.com/nb/16449096	"block"

