---
title: Runtime数据结构
keywords: iOS面试
date: 2019-04-27 18:47:40
categories: 
  - 面试
tags:
  - Runtime
comments: true
---

动态运行时(Runtime)

**<u>`编译语言和OC的动态语言的区别？`</u>**

**<u>`消息传递和函数调用的区别？`</u>**

**<u>`消息转发过程？`</u>**

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/5-1-1.png)

# 数据结构

- objc_object
- objc_class
- isa指针
- method_t

## objc_object

我们平时使用的对象类型都是id类型的，实际就是`objc_object`对象类型。objc_object是一个结构体，包括了isa_t这样一个共同体。

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/5-1-0.png)

## objc_class

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/5-1-2.png)

我们平时见的Class类就是objc_class类型。



## isa指针

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/5-1-3.png)

isa_t是一个共同体，共同体是64个0.isa分为指针型isa和非指针型的isa.

### isa指向

- 关于`对象`，isa指向`类对象`

![class](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/5-1-4.png)

- 关于`类对象`，其指向`元类对象`.

![metaclass](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/5-1-5.png)

## cache_t

- 用于`快速`查找方法执行函数
- 是可`增加扩展`的哈希表结构
- 是`局部性原理`的最佳应用：局部性原理，把调用频率最高的进行缓存。

![metaclass](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/5-1-6.png)

cache_t类似于一个数组，里面存放的是bucket_t结构体,它有两个成员变量key和IMP，key就是selector,IMP是无类型的函数指针。比如我们知道了这个key，我们可以根据hash算法定位这个bucket_t位置，然后获取IMP，进行执行。

## class_data_bits_t

- class_data_bits_t主要是class_rw_t的封装
- class_rw_t代表了类相关的读写信息、对class_ro_t的封装。比如给类添加分类的方法，属性等
- class_ro_t代表了类的只读信息

### class_rw_t

![metaclass](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/5-1-7.png)

比如分类A的所有方法列表数组，作为二维数组的第一个元素，等等；class_rw_t中方法列表一般都是分类中添加的。

### class_ro_t

![metaclass](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/5-1-8.png)

### method_t

![metaclass](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/5-1-9.png)

method_t是一个结构体：method_t实际就是对函数的一个封装。

### Type Encodings

- const char*types :是一个不可变的字符指针

![metaclass](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/5-1-10.png)

第一位是函数的返回值，因为函数的参数可以有多个，但返回值只有一个。具体的type encodings参见苹果。

![metaclass](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/5-1-11.png)



# 整体数据结构

![metaclass](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/5-1-12.png)