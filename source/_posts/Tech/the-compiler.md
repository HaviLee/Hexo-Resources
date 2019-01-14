---
title: The Compiler
date: 2017-03-10 13:47:40
categories: [Tech]
tags: [Compile,Xcode]
---

![clang](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/clang.jpeg)
粗略的来说：编译器有两个任务：将我们的`Objective-C`文件转换为底层代码，另一方面会分析我们的代码，保证我们的代码没有明显的错误。Xcode现在使用`clang`来作为编译器...

<!--more-->

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">编译器做了什么</h1>

在本文中，我们将要讨论下编译器到底是什么,如何利用它变成我们的有时。
**粗略的来说：编译器有两个任务：将我们的`Objective-C`文件转换为底层代码，另一方面会分析我们的代码，保证我们的代码没有明显的错误。**
最新的`Xcode`已经使用`clang`作为编译器。无论我们在哪里写编译器，我们都称之为`clang`.clang 是一个工具可以采用`Objective-C`代码，对代码分析，然后将代码转化为类似于程序指令集代码的更低级的表示形式：`LLVM Intermediate Representation`(LLVM IR). IR是一个偏底层的指令集代码且独立于操作系统的。LLVM 会处理这些IR指令并根据平台编译成对应的机器可读的二进制代码。这些可以一次完成也可以随着编译的时候进行。

使用 llvm IR指令的好处是, 您可以在llvm支持的任何平台上生成和运行它们.比如，如果你编写的是 iOS app,它默认是支持`Intel和ARM`两种架构，LLVM会负责将`IR`代码转化为对于平台的二进制代码。

LLVM(Low Level Virtual Machine) 得益于它的三层架构设计：
1. 它的第一层设计支持多种语言输入（C, C++, OC, Haskell）
2. 第二层中是共享优化器 (优化 llvm ir)
3. 不同的编译平台（Intel/ARM/PowerPC)

如果你想新开发一种语言，你只需要关注第一层；如果你想增加一个编译平台，你只需要负责第三层，而不用去关心输入的是什么编程语言。[LLVM Architecture](http://www.aosabook.org/en/llvm.html)有详细的LLVM架构的介绍。

**当编译一个源文件的时候，编译器会分几个步骤来处理源文件**,我们使用`clang`命令来打印出处理的一个`.m`源码文件的步骤：

```objectivec
% clang -ccc-print-phases hello.m
0: input, "hello.m", objective-c
1: preprocessor, {0}, objective-c-cpp-output
2: compiler, {1}, assembler
3: assembler, {2}, object
4: linker, {3}, image
5: bind-arch, "x86_64", {4}, image
```

本文只关注阶段`preprocess`和`compiler`,[Mach-O Executables](https://www.objc.io/issue-6/mach-o-executables.html)会详细介绍第三、四步。

## 头文件预处理

你编译源代码的第一步就是预处理。预处理器会将宏定义处理为编程语言，就是说会把使用宏的地方替换为定义的源码。比如：

```objectivec
#import <Foundation/Foundation.h>
```
预处理器会定位到该行，将其替换为该文件的内容。如果这个`header`包含其他宏定义，同样会在文件中进行替换。这就是为什么尽量不要在头文件中过多的`import`其他头文件，因为你引入一个头文件，编译器需要做更多的事情。比如在你的头文件中，不要这样写：
```objectivec
#import "MyClass.h"
```
而是使用`@class`:
```objectivec
@class MyClass;
```
这样做你像编译器保证存在这样一个类`MyClass`,然后你在实现文件`.m`中`import`这个头文件即可。

假如现在有个纯c文件，命名为`hello.c`:
```objectivec
#include <stdio.h>

int main() {
  printf("hello world\n");
  return 0;
}
```
我们使用预处理器对这个文件处理：
```objectivec
clang -E hello.c | less
```
此时我们可以看到预处理之后的文件有400多行，但是如果我们引用下面的头文件：

```objectivec
#import <Foundation/Foundation.h>
```
重新运行这个命令，发现文件有8万多行，几乎整个系统底层的代码都引入进来。幸运的是，现在有办法解决这个问题，这个新的功能成为 [Modules](http://clang.llvm.org/docs/Modules.html),可以使得这个处理过程简化。

### 宏替换 Macros

另外一个例子就是对你自定义的宏进行替换处理：

```objectivec
#define MY_CONSTANT 4
```
之后你在任何地方写的`MY_CONSTANT`，在这部分代码真正编译之前会被替换为`4`.你也可以自定义一个带参数的宏：

```objectivec
#define MY_MACRO(x) x
```
在这篇文章中就不展开讲解预处理，它是一个强大的工具。通常, 预处理器用于内联代码。我们强烈反对这样做。比如，你定义了如下的宏：

```objectivec
#define MAX(a,b) a > b ? a : b

int main() {
  printf("largest: %d\n", MAX(10,100));
  return 0;
}
```
上面的代码可以正常运行，但是下面的就出错了：
```objectivec
#define MAX(a,b) a > b ? a : b

int main() {
  int i = 200;
  printf("largest: %d\n", MAX(i++,100));
  printf("i: %d\n", i);
  return 0;
}
```
如果我们使用`clang -E max.c`会出现下面的错误：
```objectivec
largest: 201
i: 202
```
这很明显是在预处理编译器在展开宏的时候报的错误：

```objectivec
int main() {
  int i = 200;
  printf("largest: %d\n", i++ > 100 ? i++ : 100);
  printf("i: %d\n", i);
  return 0;
}
```

在这个例子中，宏的错误展开还是很明显的，但是还有很多的情况是无法预知并且很难debug.除了使用宏，你也可以使用`static inline`函数 ：

```objectivec
#include <stdio.h>

static const int MyConstant = 200;

static inline int max(int l, int r) {
   return l > r ? l : r;
}

int main() {
  int i = MyConstant;
  printf("largest: %d\n", max(i++,100));
  printf("i: %d\n", i);
  return 0;
}
```
上面的代码可以正确的打印结果：`(i: 201)`。因为代码是内联的，它的效果和宏变量是一样的，但是它更少出错。这样做你也可以设置断点，进行类型检测，避免无法预期的错误。

唯一合理的使用宏的一个场景是：进行日志输出，因为你可以使用`__FILE__`和`__LINE__`进行断言宏。

## 词法分析(Tokenization or Lexing)

## 生成抽象语法树(Parsing)

### 更多信息

## 静态分析

### 类型检测

### 其他分析

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">代码生成</h1>

## 代码优化

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">如何利用这个优势</h1>

## 使用`libclang`或者`clang`插件

## 写一个分析器

## 更多`clang`可能性

## 建议阅读

**************

> Reference: [Clang 8 Document](http://clang.llvm.org/docs/LibASTMatchersTutorial.html)
> [IBM Developer works](https://www.ibm.com/developerworks/cn/)
> [LLVM.org](http://llvm.org)
> 
