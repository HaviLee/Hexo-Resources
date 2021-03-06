---
title: Block本质探究一
keywords: Block
date: 2017-01-16 13:47:40
categories: 
  - Objective-C
tags:
  - Block
comments: false
---
![image](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Life/Lionel-Messi-Wallpapers-2.jpg)
**block本质也是一个OC对象，内部也有一个isa指针。block是封装了函数调用以及函数调用环境的OC对象。block原理是什么？本质是什么？__block作用是什么？有什么注意点？**
<!--more-->
<!-- description: block本质也是一个OC对象，内部也有一个isa指针。block是封装了函数调用以及函数调用环境的OC对象。block原理是什么？本质是什么？__block作用是什么？有什么注意点？ -->

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">面试题</h1>

**1.block原理是什么？本质是什么？**
2.__block作用是什么？有什么注意点？
3.block的属性修饰词为什么是copy？使用block有哪些使用注意？
4.block在修改NSMutableArray，需不需要使用__blcok?

>首先：block本质也是一个OC对象，内部也有一个isa指针。block是封装了函数调用以及函数调用环境的OC对象。


<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">探寻block的本质</h1>

```objectivec
- (void)createBlock
{
	int age = 10;
	void(^block)(int, int) = ^(int a, int b) {
		NSLog(@"this is an block, a = %d, b = %d",a,b);
		NSLog(@"this is an block, age = %d",age );
	};
	block(3,5);
}
```
使用下面的命令将.m文件转化为C++
```objectivec
xcrun -sdk iphonesimulator clang -rewrite-objc HaviBlock.m
```
下面是C++结构的block：

```objectivec
struct __HaviBlock__createBlock_block_impl_0 {//block的C++结构
  struct __block_impl impl;
  struct __HaviBlock__createBlock_block_desc_0* Desc;
  int age;
  __HaviBlock__createBlock_block_impl_0(void *fp, struct __HaviBlock__createBlock_block_desc_0 *desc, int _age, int flags=0) : age(_age) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __HaviBlock__createBlock_block_func_0(struct __HaviBlock__createBlock_block_impl_0 *__cself, int a, int b) {
  int age = __cself->age; // bound by copy//从这里可以看到是对外面的变量copy过来的
  __Block_byref_age_0 *age = __cself->age; // bound by ref//如果使用了__block，是对外面的变量创建了个引用

  NSLog((NSString *)&__NSConstantStringImpl__var_folders_82__00fdxvn217fjfl3my96zr0509801s_T_HaviBlock_1ee770_mi_0,a,b);
  NSLog((NSString *)&__NSConstantStringImpl__var_folders_82__00fdxvn217fjfl3my96zr0509801s_T_HaviBlock_1ee770_mi_1,age );
 }

static struct __HaviBlock__createBlock_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __HaviBlock__createBlock_block_desc_0_DATA = { 0, sizeof(struct __HaviBlock__createBlock_block_impl_0)};

static void _I_HaviBlock_createBlock(HaviBlock * self, SEL _cmd) {//这个就是block中的create函数
 int age = 10;
 void(*block)(int, int) = ((void (*)(int, int))&__HaviBlock__createBlock_block_impl_0((void *)__HaviBlock__createBlock_block_func_0, &__HaviBlock__createBlock_block_desc_0_DATA, age));
 ((void (*)(__block_impl *, int, int))((__block_impl *)block)->FuncPtr)((__block_impl *)block, 3, 5);
}

```
## 定义block变量
```objectivec
void(*block)(int, int) = ((void (*)(int, int))&__HaviBlock__createBlock_block_impl_0((void *)__HaviBlock__createBlock_block_func_0, &__HaviBlock__createBlock_block_desc_0_DATA, age));
```
从上面的定义，block中调用了__HaviBlock__createBlock_block_impl_0函数，并且将__HaviBlock__createBlock_block_impl_0函数的地址赋值给了blcok.下面来看下__HaviBlock__createBlock_block_impl_0内部结构：

## block_impl_0内部结构体

```objectivec
struct __HaviBlock__createBlock_block_impl_0

{
  struct __block_impl impl;
  struct __HaviBlock__createBlock_block_desc_0* Desc;
  int age;
  __HaviBlock__createBlock_block_impl_0(void *fp, struct __HaviBlock__createBlock_block_desc_0 *desc, int _age, int flags=0) : age(_age) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```
__HaviBlock__createBlock_block_impl_0 结构体内有一个同名的构造函数__HaviBlock__createBlock_block_impl_0，构造函数中对变量进行了赋值，并最终返回了一个结构体。<br>

也就是说最终将__HaviBlock__createBlock_block_impl_0结构体的地址赋值给了block变量！<br>

__HaviBlock__createBlock_block_impl_0构造函数有四个参数：
1.(void *)__HaviBlock__createBlock_block_func_0
2.&__HaviBlock__createBlock_block_desc_0_DATA
3.int _age,
4.int flags=0
其中flag是具有默认值，这里的age则是表示传入_age参数赋值给age成员；<br>
### block_func_0

```objectivec
static void __HaviBlock__createBlock_block_func_0(struct __HaviBlock__createBlock_block_impl_0 *__cself, int a, int b) {
  int age = __cself->age; // bound by copy

  NSLog((NSString *)&__NSConstantStringImpl__var_folders_82__00fdxvn217fjfl3my96zr0509801s_T_HaviBlock_1ee770_mi_0,a,b);
  NSLog((NSString *)&__NSConstantStringImpl__var_folders_82__00fdxvn217fjfl3my96zr0509801s_T_HaviBlock_1ee770_mi_1,age );
}
```
在这个函数中，首先取出age的值，紧接着可以看到两个熟悉的NSLog，这个就是我们再block中写下的代码。所以__HaviBlock__createBlock_block_func_0函数中其实保存着我们在block中写下的代码。__HaviBlock__createBlock_block_impl_0中传入的是__HaviBlock__createBlock_block_func_0，<strong> 就是说我们再block中写下的代码被封装成为__HaviBlock__createBlock_block_func_0</strong>并把__HaviBlock__createBlock_block_func_0函数的地址保存在__HaviBlock__createBlock_block_impl_0中。

### block_desc_0_DATA
```objectivec

static struct __HaviBlock__createBlock_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __HaviBlock__createBlock_block_impl_0*, struct __HaviBlock__createBlock_block_impl_0*);
  void (*dispose)(struct __HaviBlock__createBlock_block_impl_0*);
} __HaviBlock__createBlock_block_desc_0_DATA = { 0, sizeof(struct __HaviBlock__createBlock_block_impl_0), __HaviBlock__createBlock_block_copy_0, __HaviBlock__createBlock_block_dispose_0};

```
__HaviBlock__createBlock_block_desc_0中存储着两个参数：reserved 和 Block_size，并且reserved赋值为0，Block_size则存储着__HaviBlock__createBlock_block_impl_0的占用空间的大小。最后将__HaviBlock__createBlock_block_desc_0地址传入__HaviBlock__createBlock_block_impl_0中的Desc.

### age
age是我们定义的局部变量。因为在block中使用age局部变量，所以在block声明的时候会将age作为参数传入，<strong>也就是说block会捕获age变量 </strong>
如果在block中没有使用age，则只会传入__HaviBlock__createBlock_block_func_0 和__HaviBlock__createBlock_block_desc_0_DATA这两个参数。
<br>
**在这里可以思考：为什么在我们定义block之后，再改变age的值，在block调用的时候无效？**
```objectivec
int age = 10;
void(^block)(int ,int) = ^(int a, int b){
     NSLog(@"this is block,a = %d,b = %d",a,b);
     NSLog(@"this is block,age = %d",age);
};
age = 20;
block(3,5); 
// log: this is block,a = 3,b = 5
//      this is block,age = 10

```
A:因为在block定义的时候，已经将age的值传入__HaviBlock__createBlock_block_impl_0结构体中，并在调用的时候讲age从block中取出来使用，因此在block定义之后对局部变量进行改变无法被block捕获的。

## 重新探究_impl_0结构体
```objectivec
struct __HaviBlock__createBlock_block_impl_0 {
  struct __block_impl impl;
  struct __HaviBlock__createBlock_block_desc_0* Desc;
  __Block_byref_age_0 *age; // by ref
  __HaviBlock__createBlock_block_impl_0(void *fp, struct __HaviBlock__createBlock_block_desc_0 *desc, __Block_byref_age_0 *_age, int flags=0) : age(_age->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```
__HaviBlock__createBlock_block_impl_0第一个变量就是__block_impl结构体：

```objectivec
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};
```

从这里__HaviBlock__createBlock_block_impl_0内部有一个isa指针，因此说明block本质上是一个OC对象。而在
__HaviBlock__createBlock_block_impl_0 构造函数中传入的值存储在__HaviBlock__createBlock_block_impl_0结构体中，最后将改结构体的地址赋值给block。

根据__HaviBlock__createBlock_block_impl_0三个参数的分析得出结论：<br>
1.__block_impl结构体中的指针存储着&_NSConcreteStackBlock地址，可以暂时理解为类对象地址，block就是_NSConcreteStackBlock类型的。<br>
2.block代码中的代码被封装成为__HaviBlock__createBlock_block_func_0，FuncPtr则存储着__HaviBlock__createBlock_block_func_0的地址<br>
3.Desc指向__HaviBlock__createBlock_block_desc_0结构体对象，其中存储着__HaviBlock__createBlock_block_impl_0结构体占用的空间；

## 调用block执行内部函数
```objectivec

 ((void (*)(__block_impl *, int, int))((__block_impl *)block)->FuncPtr)((__block_impl *)block, 3, 5);

```

上面的代码可以看出block通过block找到FunPtr直接调用，通过上面的源代码我们知道block指向的是__HaviBlock__createBlock_block_impl_0类型的结构体，但是在__HaviBlock__createBlock_block_impl_0中并没有直接可以找到FunPtr，而FunPtr存储在__block_impl中，为什么block可以直接调用__block_impl中的FunPtr呢？<br>

是因为（__blcok_impl*）block将block强制转化为__block_impl类型的，因为__block_impl是__HaviBlock__createBlock_block_impl_0结构体的第一个成员，也就是说__block_impl的内存地址就是__HaviBlock__createBlock_block_impl_0结构体内存地址的开发。所以可以转化成功。（why?todo）<br>

FunPtr中存储着通过代码块封装的函数地址，那么调用这个函数，也就是执行代码快中的代码。回头看__HaviBlock__createBlock_block_func_0，可以发现第一个参数是_HaviBlock__createBlock_block_impl_0类型的指针，也就是说将block传入到了__HaviBlock__createBlock_block_func_0中，方便重中取出block捕获的值。

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">验证Block本质</h1>

**Block本质确实是__HaviBlock__createBlock_block_impl_0结构体**
方法：我们使用自定义和Block一致的结构体，并将block内部的结构体强制转化为我们自定义的结构体：
```objectivec
struct __main_block_desc_0 { 
    size_t reserved;
    size_t Block_size;
};
struct __block_impl {
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
};
// 模仿系统__main_block_impl_0结构体
struct __main_block_impl_0 { 
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    int age;
};
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int age = 10;
        void(^block)(int ,int) = ^(int a, int b){
            NSLog(@"this is block,a = %d,b = %d",a,b);
            NSLog(@"this is block,age = %d",age);
        };
// 将底层的结构体强制转化为我们自己写的结构体，通过我们自定义的结构体探寻block底层结构体
        struct __main_block_impl_0 *blockStruct = (__bridge struct __main_block_impl_0 *)block;
        block(3,5);
    }
    return 0;
}

```
通过打断点可以查看：
![duan1](https://media.githubusercontent.com/media/Interview-Skill/OC-Class-Analysis/master/Image/block2.png)
下面进入block内部，看一下堆栈信息中的函数调用地址。<strong>Debug workflow -> slways show Disassembly</strong>

![duan2](https://media.githubusercontent.com/media/Interview-Skill/OC-Class-Analysis/master/Image/block5.png)

## 总结
**到这里我们从源码查看了所有和block有关的结构体，下面通过一张图解释各个结构体的关系：**

![duan3](https://media.githubusercontent.com/media/Interview-Skill/OC-Class-Analysis/master/Image/Block.png)

**Block的底层数据结构：**

![duan3](https://media.githubusercontent.com/media/Interview-Skill/OC-Class-Analysis/master/Image/block7.png)

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">Block捕获变量</h1>

为了保证block能够正常访问外部变量，block有一个变量捕获机制：

## 局部变量
### auto变量
上面的代码我们已经了解了block对age变量的捕获。
auto变量离开作用域就会销毁，<strong>局部变量前面默认添加auto关键字</strong>。自定变量会捕获到block内部，也就是说在block内部会新增一个变量专门来存储变量值。auto变量只存在局部变量中，访问方式是值传递，通过对age源码的查看可以确认。

### static变量
static修饰的变量为指针传递，就是说他是通过传递该值的地址到block内部，看看下源码：
```objectivec
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        auto int age = 10;
        static int ageB = 11;
        void(^block)(void) = ^{
            NSLog(@"hello, a = %d, b = %d", age,ageB);
        };
        a = 1;
        b = 2;
        block();
    }
    return 0;
}
// log : block本质[57465:18555229] hello, a = 10, b = 2
// block中a的值没有被改变而b的值随外部变化而变化。

```
我们经过xcrun编译为C++：
```objectivec
struct __HaviNewBlock__verifyBlock_block_impl_0 {
  struct __block_impl impl;
  struct __HaviNewBlock__verifyBlock_block_desc_0* Desc;
  int age;
  int *ageB;	//这里可以看到使用static修饰的变量在编译为c++后是指针
  __HaviNewBlock__verifyBlock_block_impl_0(void *fp, struct __HaviNewBlock__verifyBlock_block_desc_0 *desc, int _age, int *_ageB, int flags=0) : age(_age), ageB(_ageB) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static void __HaviNewBlock__verifyBlock_block_func_0(struct __HaviNewBlock__verifyBlock_block_impl_0 *__cself, int a, int b) {
  int age = __cself->age; // bound by copy
  int *ageB = __cself->ageB; // bound by copy

  NSLog((NSString *)&__NSConstantStringImpl__var_folders_82__00fdxvn217fjfl3my96zr0509801s_T_HaviNewBlock_49c829_mi_2,a,b);
  NSLog((NSString *)&__NSConstantStringImpl__var_folders_82__00fdxvn217fjfl3my96zr0509801s_T_HaviNewBlock_49c829_mi_3,age );
  NSLog((NSString *)&__NSConstantStringImpl__var_folders_82__00fdxvn217fjfl3my96zr0509801s_T_HaviNewBlock_49c829_mi_4,(*ageB));
 }
 
 从这里我们可以看出ageB穿进去的是地址
 void(*block)(int, int) = ((void (*)(int, int))&__HaviNewBlock__verifyBlock_block_impl_0((void *)__HaviNewBlock__verifyBlock_block_func_0, &__HaviNewBlock__verifyBlock_block_desc_0_DATA, age, &ageB));


```

从源代码中看出，age和ageB两个变量都被捕获到了block内部，但是age是通过值传递的，ageB是传递的是地址。
为什么有这种差异？因为自动变量随时有可能被销毁，block在执行的时候有可能自动变量被销毁了，如果这个时候再去访问被销毁的地址就会找不到这个内存地址，因此自动变量一定是值传递而不是指针传递了。而静态变量是不会被销毁的，因此可以使用指针传递。因为传递的是值地址，在block调用前修改，会体现出来。

## 全局变量
我们看下全局的变量捕获情况：
```objectivec
int a = 10;
static int b = 11;
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        void(^block)(void) = ^{
            NSLog(@"hello, a = %d, b = %d", a,b);
        };
        a = 1;
        b = 2;
        block();
    }
    return 0;
}
// log hello, a = 1, b = 2

```
我们生成c++
![c++](https://media.githubusercontent.com/media/Interview-Skill/OC-Class-Analysis/master/Image/c%2B%2B.png)
通过上面的代码发现block_impl_0中并没有添加任何变量，因为block不需要捕获全局变量，因为全局变量在哪里都可以访问。<br>
<strong>因为局域变量需要跨函数访问所以需要捕获，全局变量在哪里都可以访问，所以不需要捕获</strong>
![c++](https://media.githubusercontent.com/media/Interview-Skill/OC-Class-Analysis/master/Image/auto-static.png)

** ⚠️局部变量都会被block捕获，自动变量是值捕获，静态变量为地址捕获。全局变量不会被捕获。**

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">block的类型</h1>


block是什么类型？在前面的源代码里看到isa指向了_NSConcreateStackBlock对象，那么block是不是就是_NSConcreateStackBlock类型？？？
```objectivec
int main(int argc, const char * argv[]) {
@autoreleasepool {
    // __NSGlobalBlock__ : __NSGlobalBlock : NSBlock : NSObject
    void (^block)(void) = ^{
        NSLog(@"Hello");
    };
    
    NSLog(@"%@", [block class]);
    NSLog(@"%@", [[block class] superclass]);
    NSLog(@"%@", [[[block class] superclass] superclass]);
    NSLog(@"%@", [[[[block class] superclass] superclass] superclass]);
}
return 0;
}
```
打印结果：
```objectivec
2018-12-08 18:30:03.304124+0800 iOS底层原理总结[56926:5718709] block -------__NSMallocBlock__
2018-12-08 18:30:03.304246+0800 iOS底层原理总结[56926:5718709] block -------__NSGlobalBlock
2018-12-08 18:30:03.304333+0800 iOS底层原理总结[56926:5718709] block -------NSBlock
2018-12-08 18:30:03.304401+0800 iOS底层原理总结[56926:5718709] block -------NSObject

```

从上面我们可以看到block都继承自NSBlock->NSObject:这也更加的验证了block是个对象

## block有三种类型
```objectivec
__NSGlobalBlock__ （ _NSConcreteGlobalBlock ）
__NSStackBlock__ （ _NSConcreteStackBlock ）
__NSMallocBlock__ （ _NSConcreteMallocBlock ）
```
我们通过代码验证这三种类型的不同：
```objectivec

- (void)blockType
{
	//1.内部没有调用外部任何变量的block
	void (^block1)(void) = ^{
		NSLog(@"hello");
	};
	
	//2.调用外部变量的block
	int a = 10;
	void (^block2)(void) = ^{
		NSLog(@"hello---%d",a);
	};
	
	//3.直接调用block
	
	NSLog(@"block-type:%@----%@----%@",[block1 class],[block2 class],[^{NSLog(@"%d",a);} class]);
}

2018-12-08 18:39:52.097893+0800 iOS底层原理总结[59479:5737537] 
block-type:__NSGlobalBlock__----__NSMallocBlock__----__NSStackBlock__

```
打印出来的类型和我们在源码中观察到的不一样？为什么：

## block在内存中的存储
block在内存是如何存储的？
![block-memmory](https://media.githubusercontent.com/media/Interview-Skill/OC-Class-Analysis/master/Image/block4.png)

>1.__NSGlobalBlock__直到程序结束后才会被收回，很少使用这样的block<br>
>2.__NSStackBlokc__存放在栈中，栈中的内存是由系统自动分配和释放，在作用执行完之后会立即释放，在相同的作用域中定义并调用block似乎多次一举<br>
>3.__NSMallocBlock__在平时编程中最常用的，存放在堆中的block需要程序员自己释放。

## block是如何定义其类型
![block-memmory](https://media.githubusercontent.com/media/Interview-Skill/OC-Class-Analysis/master/Image/block3.png)
我们验证上面的结论：
首先我们关闭ARC,因为ARC会自动帮我们进行很多处理：

```objectivec
// MRC环境！！！
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Global：没有访问auto变量：__NSGlobalBlock__
        void (^block1)(void) = ^{
            NSLog(@"block1---------");
        };   
        // Stack：访问了auto变量： __NSStackBlock__
        int a = 10;
        void (^block2)(void) = ^{
            NSLog(@"block2---------%d", a);
        };
        NSLog(@"%@ %@", [block1 class], [block2 class]);
        // __NSStackBlock__调用copy ： __NSMallocBlock__
        NSLog(@"%@", [[block2 copy] class]);
    }
    return 0;
}

打印结果：
__NSGlobalBlock__	__NSStackBlock__	__NSMallocBlock__

```
## 总结
1.没有访问auto变量的block是 __NSGlobalBlock__ 类型的，存放在数据段中。访问了 auto变量的block是 __NSStackBlock__ 存放在栈中（因为出了作用域就会销毁，block也是一个对象）。__NSStackBlock__ 进行copy之后变成了 __NSMallocBlock__ 类型，并被copy到了堆中。<br>
2.__NSGlobalBlock__ 类型的很少见，因为如果block不访问外界变量，直接通过函数实现就可以了，不需要block了。<br>
3.__NSStackBlock__ 访问了外部变量，并且存放在栈中，栈里面的代码在作用域结束后就会被销毁，那么就有可能在block内存销毁之后采取调用它，这样就会有问题。<br>

看下面的例子：
```objectivec
void (^block)(void);
void test()
{
    // __NSStackBlock__
    int a = 10;
    block = ^{
        NSLog(@"block---------%d", a);
    };
}
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        test();
        block();
    }
    return 0;
}

打印结果：block----------272632488找不到了

```

看到a时一个不可控的值，是因为创建的block是 __NSStackBlock__ (自己思考为什么)因此block是存在栈中的，当test函数执行完之后，栈内存中的block占用的内存会被收回，因此就找不到数据a了；

![stack](https://media.githubusercontent.com/media/Interview-Skill/OC-Class-Analysis/master/Image/nsstackblock.png)

为了避免这种情况发生，需要将block复制到堆中：
```objectivec

void (^block)(void);
void test()
{
    // __NSStackBlock__
    int a = 10;
    block = ^{
        NSLog(@"block---------%d", a);
    };
}
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        test();
        block();
    }
    return 0;
}

打印结果：block----------10
```
其他类型的block调用copy会有什么结果呢？
![stack](https://media.githubusercontent.com/media/Interview-Skill/OC-Class-Analysis/master/Image/block6.png)
>⚠️ 因此在MRC开始时期，我们经常使用copy来保存block，将栈上的block复制到堆中，即使栈中的block销毁，堆上的block也不会销毁，需要我们自己销毁,<strong>但是在ARC环境下，xcode会自动给我们进行copy操作，使得block不会被销毁。

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">ARC帮你做了什么</h1>

在ARC环境下，编译器会根据情况自动将栈上的block进行copy操作，将block复制到堆上。
**什么情况下ARC会自动的进行copy操作？**
以下代码是在ARC下进行的：

## block作为函数返回值

```objectivec
typedef void (^Block)(void);
Block myblock()
{
    int a = 10;
    // 上文提到过，block中访问了auto变量，此时block类型应为__NSStackBlock__
    Block block = ^{
        NSLog(@"---------%d", a);
    };
    return block;
}
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Block block = myblock();
        block();
       // 打印block类型为 __NSMallocBlock__
        NSLog(@"%@",[block class]);
    }
    return 0;
}

```
输出结果：
```objectivec
block -- type: __NSMallocBlock__
```
1). 上面提到，如果block访问auto变量，block的类型为 __NSStackBlock__，但是上面的block为 __NSMallocBlock__类型，并且可以打印出变量a的值，这说明了block并没有被销毁。<br>
2). block是经过copy操作可以变为__NSMallocBlock__类型,因此可以猜测ARC自动将我们的block进行copy操作，来保存block，并在适当的地方release.

## 将block赋值给__strong指针

block赋值强指针引用的时候，ARC也会自动对block进行一次copy操作。
```objectivec
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // block内没有访问auto变量
        Block block = ^{
            NSLog(@"block---------");
        };
        NSLog(@"%@",[block class]);
        int a = 10;
        // block内访问了auto变量，但没有赋值给__strong指针
        NSLog(@"%@",[^{
            NSLog(@"block1---------%d", a);
        } class]);
        // block赋值给__strong指针
        Block block2 = ^{
          NSLog(@"block2---------%d", a);
        };
        NSLog(@"%@",[block1 class]);
    }
    return 0;
}

```
打印结果：
```objectivec
__NSGlobalBlock__
__NSStackBlock__
__NSMallocBlock__
```

## block作为Cocoa API中的方法中含有usingBlock的时候
例如：遍历函数：
```objectivec
NSArray *array = @[];
[array enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
            
}];
```

## block作为GCD API的参数的时候
比如GCD延时操作：
```objectivec

static dispatch_once_t onceToken;
dispatch_once(&onceToken, ^{
            
});        
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            
});

```
 
<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">block声明写法</h1>

1. MRC环境下：
```objectivec
@property (nonmatic, copy) void (^block)(void);
```
2. ARC环境下：
```objectivec
@property (nonmatic, copy) void (^block)(void);
@property (nonmatic, strong) void (^block)(void);
```

> [探寻block的本质](https://www.jianshu.com/p/c99f4974ddb5)

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">附加题</h1>
 
下面的block是否会捕获变量呢？

```objectivec

#import "Person.h"
@implementation Person
- (void)test
{
    void(^block)(void) = ^{
        NSLog(@"%@",self);
    };
    block();
}
- (instancetype)initWithName:(NSString *)name
{
    if (self = [super init]) {
        self.name = name;
    }
    return self;
}
+ (void) test2
{
    NSLog(@"类方法test2");
}
@end

```

查看C++代码

```objectivec
struct __BlockSelfObject__test_block_impl_0 {
  struct __block_impl impl;
  struct __BlockSelfObject__test_block_desc_0* Desc;
  BlockSelfObject *self;	//self变量被捕获
  __BlockSelfObject__test_block_impl_0(void *fp, struct __BlockSelfObject__test_block_desc_0 *desc, BlockSelfObject *_self, int flags=0) : self(_self) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static void __BlockSelfObject__test_block_func_0(struct __BlockSelfObject__test_block_impl_0 *__cself) {
  BlockSelfObject *self = __cself->self; // bound by copy //从block中取出self

  NSLog((NSString *)&__NSConstantStringImpl__var_folders_82__00fdxvn217fjfl3my96zr0509801s_T_BlockSelfObject_2ee577_mi_0,self);
 }
 
 //默认情况下，方法传入两个参数cmd和self
 static void _I_BlockSelfObject_test(BlockSelfObject * self, SEL _cmd) {
 void(*block)(void) = ((void (*)())&__BlockSelfObject__test_block_impl_0((void *)__BlockSelfObject__test_block_func_0, &__BlockSelfObject__test_block_desc_0_DATA, self, 570425344));
 ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
}
```
从源码我们可以看到self同样被block捕获，同时我们看到test方法默认传递了两个参数cmd 和 self，而类方法也传递了两个参数cmd 和self。
```objectivec
static void _C_BlockSelfObject_test2(Class self, SEL _cmd) {
 NSLog((NSString *)&__NSConstantStringImpl__var_folders_82__00fdxvn217fjfl3my96zr0509801s_T_BlockSelfObject_5c7c2d_mi_1);
}

```
** ⚠️不论是对象方法还是类方法self都作为参数传递给方法内部，既然作为参数传入，那么self就是是局部变量。下面看看在block中使用成员变量和属性有什么不同？**
```objectivec

- (void)test
{
	void(^block)(void) = ^{
		NSLog(@"%@",self);
		NSLog(@"%@",self.name);
		NSLog(@"%@",_name);
	};
	block();
}
```

```objectivec
struct __BlockSelfObject__test_block_impl_0 {
  struct __block_impl impl;
  struct __BlockSelfObject__test_block_desc_0* Desc;
  BlockSelfObject *self; //同样只捕获了self变量
  __BlockSelfObject__test_block_impl_0(void *fp, struct __BlockSelfObject__test_block_desc_0 *desc, BlockSelfObject *_self, int flags=0) : self(_self) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static void __BlockSelfObject__test_block_func_0(struct __BlockSelfObject__test_block_impl_0 *__cself) {
  BlockSelfObject *self = __cself->self; // bound by copy

  NSLog((NSString *)&__NSConstantStringImpl__var_folders_82__00fdxvn217fjfl3my96zr0509801s_T_BlockSelfObject_a22808_mi_0,self);
  NSLog((NSString *)&__NSConstantStringImpl__var_folders_82__00fdxvn217fjfl3my96zr0509801s_T_BlockSelfObject_a22808_mi_1,((NSString *(*)(id, SEL))(void *)objc_msgSend)((id)self, sel_registerName("name")));
  NSLog((NSString *)&__NSConstantStringImpl__var_folders_82__00fdxvn217fjfl3my96zr0509801s_T_BlockSelfObject_a22808_mi_2,(*(NSString * _Nonnull *)((char *)self + OBJC_IVAR_$_BlockSelfObject$_name)));
 }
 
 属性调用get方法，通过方法选择器获取name
 成员变量直接通过地址获取
```
** ⚠️结论：<strong>即使block使用的是实例对象的属性，block捕获仍然是实例对象而非属性，并通过实例对象不同方法获取属性（属性调用get方法，通过方法选择器获取name，成员变量直接通过地址获取） </strong>**
























