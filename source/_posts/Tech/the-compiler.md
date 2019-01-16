---
title: The Compiler
date: 2017-03-10 13:47:40
categories: [Build-Tools]
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
pre
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

在预处理之后，现在每个`.m`文件都有一堆的定义。这时的文本是从字符串转化为词法流的。比如上面的C代码片段：

```objectivec
int main() {
  NSLog(@"hello, %@", @"world");
  return 0;
}
```

我们是使用`clang -Xclang -dump-tokens hello.m`把上面的代码进行词法分割：

```objectivec
int 'int'        [StartOfLine]  Loc=<hello.m:4:1>
identifier 'main'        [LeadingSpace] Loc=<hello.m:4:5>
l_paren '('             Loc=<hello.m:4:9>
r_paren ')'             Loc=<hello.m:4:10>
l_brace '{'      [LeadingSpace] Loc=<hello.m:4:12>
identifier 'NSLog'       [StartOfLine] [LeadingSpace]   Loc=<hello.m:5:3>
l_paren '('             Loc=<hello.m:5:8>
at '@'          Loc=<hello.m:5:9>
string_literal '"hello, %@"'            Loc=<hello.m:5:10>
comma ','               Loc=<hello.m:5:21>
at '@'   [LeadingSpace] Loc=<hello.m:5:23>
string_literal '"world"'                Loc=<hello.m:5:24>
r_paren ')'             Loc=<hello.m:5:31>
semi ';'                Loc=<hello.m:5:32>
return 'return'  [StartOfLine] [LeadingSpace]   Loc=<hello.m:6:3>
numeric_constant '0'     [LeadingSpace] Loc=<hello.m:6:10>
semi ';'                Loc=<hello.m:6:11>
r_brace '}'      [StartOfLine]  Loc=<hello.m:7:1>
eof ''          Loc=<hello.m:7:2>
```

我们可以看到每个词法都包含了一个代码片段和它在源码中的位置。这个源码的位置表示的是宏没有展开的位置，所以如果你的代码有问题，`clang`可以告诉你错误的地方。

## 生成抽象语法树(Parsing)

下面来到最有意思的部分:我们上一步生产的词法分割会解析成为抽象语法树（abstract syntax tree).由于OC是一个比较复杂的语言，解析比较困难。在解析之后，代码现在就是一个抽象语法树：一个代表了原始代码的结构。下面是例子：

```objectivec
#import <Foundation/Foundation.h>

@interface World
- (void)hello;
@end

@implementation World
- (void)hello {
  NSLog(@"hello, world");
}
@end

int main() {
   World* world = [World new];
   [world hello];
}
```

使用命令`clang -Xclang -ast-dump -fsyntax-only hello.m`得到下面的抽象语法树：

```objectivec
@interface World- (void) hello;
@end
@implementation World
- (void) hello (CompoundStmt 0x10372ded0 <hello.m:8:15, line:10:1>
  (CallExpr 0x10372dea0 <line:9:3, col:24> 'void'
    (ImplicitCastExpr 0x10372de88 <col:3> 'void (*)(NSString *, ...)' <FunctionToPointerDecay>
      (DeclRefExpr 0x10372ddd8 <col:3> 'void (NSString *, ...)' Function 0x1023510d0 'NSLog' 'void (NSString *, ...)'))
    (ObjCStringLiteral 0x10372de38 <col:9, col:10> 'NSString *'
      (StringLiteral 0x10372de00 <col:10> 'char [13]' lvalue "hello, world"))))


@end
int main() (CompoundStmt 0x10372e118 <hello.m:13:12, line:16:1>
  (DeclStmt 0x10372e090 <line:14:4, col:30>
    0x10372dfe0 "World *world =
      (ImplicitCastExpr 0x10372e078 <col:19, col:29> 'World *' <BitCast>
        (ObjCMessageExpr 0x10372e048 <col:19, col:29> 'id':'id' selector=new class='World'))")
  (ObjCMessageExpr 0x10372e0e8 <line:15:4, col:16> 'void' selector=hello
    (ImplicitCastExpr 0x10372e0d0 <col:5> 'World *' <LValueToRValue>
      (DeclRefExpr 0x10372e0a8 <col:5> 'World *' lvalue Var 0x10372dfe0 'world' 'World *'))))
```

抽象语法树的每一个节点都标注了原始代码的位置，所以后面如果你的代码有问题，clang可以指出你代码的错误位置。

### 更多信息

[Introduce to the clang AST](http://clang.llvm.org/docs/IntroductionToTheClangAST.html)

## 静态分析

一旦编译器生成了抽象语法树，编译器就可以对你的抽象语法树进行语法分析帮助你检查你的语法错误。比如类型检查，帮助你检查你的类型是不是正确的。比如你给一个对象发送消息，它会检查这个对象是不是实现这个方法。同时，`clang`会做一些高级的分析，保证你的代码不会出现其他怪异的行为。

### 类型检测

你任何时候写的代码，`clang`都会帮助你做一些检查保证你没有出现一些错误。其中一个比较明显的检查是保证你代码给正确的对象发送正确的消息，或者给函数传递正确的参数。如果你定义了一个`NSObject *`你是不允许给这个对象发送`hello`消息的，因为`clang`会提示你错粗。同样如果你创建一个子类：

```objectivec
@interface Test : NSObject
@end
```

如果你尝试把这个Test对象赋值给其他类型的对象，编译器会警告你所做的有肯能是错误的。
这里有两种类型检测：

- {% label primary@动态类型检测 %}：动态检测就是一个类型是在`runtime`阶段才进行检测，之前你可以在任何时间给任何对象发送任何消息,只有在`runtime`才可以确定这个对象会不会响应这个消息。这种在`runtime`才会进行检测的成为动态检测。

- {% label primary@静态类型检测 %}：静态检测就是类型检测是在编译期间。当你使用`ARC`的时候，编译器会在编译阶段检查很多的类型，因为它需要知道他正在处理的对象。比如，你不能写出下面的代码：

```objectivec
[myObject hello]
```

因为在我们的代码中没有定义`hello`方法。

### 其他分析

`clang`还会为做其他的很多的分析。如果把clang整个repo复制下来，然后在`lib/StaticAnalyzer/Checkers`可以看到静态检查器。比如：`ObjcUnusedIVarChecker.cpp`这个是检测`ivar`是不是没有被使用。还有一个`ObjcSelfInitCheck.cpp`这个是为了检测是否调用`[self initWith...]`或者`[super init]`在真正使用self之前。还有一些检测发生在编译的其他阶段。比如在`lib/Sema/SemaExprObjC.cpp`：

```objectivec
 Diag(SelLoc, diag::warn_arc_perform_selector_leaks);
```

这个会产生一个`performSelector may cause a leak because its selector is unknown`

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">代码生成</h1>

现在，你的代码经过词法分解，生成抽象树，然后经过clang 分析之后，就可以生成LLVM的IR代码。我们可以看下这个是如何发生的：

```objectivec
#include <stdio.h>

int main() {
  printf("hello world\n");
  return 0;
}
```

我们可以使用下面的命令生成`LLVM IR`中间代码：

```objectivec
clang -O3 -S -emit-llvm hello.c -o hello.ll
```

下面是生成的LLVM IR代码：

```objectivec
; ModuleID = 'hello.c'
target datalayout = "e-m:o-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-apple-macosx10.9.0"

@str = private unnamed_addr constant [12 x i8] c"hello world\00"

; Function Attrs: nounwind ssp uwtable
define i32 @main() #0 {
  %puts = tail call i32 @puts(i8* getelementptr inbounds ([12 x i8]* @str, i64 0, i64 0))
  ret i32 0
}

; Function Attrs: nounwind
declare i32 @puts(i8* nocapture readonly) #1

attributes #0 = { nounwind ssp uwtable "less-precise-fpmad"="false" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-nans-fp-math"="false" "stack-protector-buffer-size"="8" "unsafe-fp-math"="false" "use-soft-float"="false" }
attributes #1 = { nounwind }

!llvm.ident = !{!0}

!0 = metadata !{metadata !"Apple LLVM version 6.0 (clang-600.0.41.2) (based on LLVM 3.5svn)"}
```

我们可以看到`main`函数只有两行：一是打印String，一个是返回0.

对于OC代码也是同样的结果，我们使用命令`LLVM-dis < five.bc | less`:

```objectivec
#include <stdio.h>
#import <Foundation/Foundation.h>

int main() {
  NSLog(@"%@", [@5 description]);
  return 0;
}
```

下面是输出结果：

```objectivec
define i32 @main() #0 {
  %1 = load %struct._class_t** @"\01L_OBJC_CLASSLIST_REFERENCES_$_", align 8
  %2 = load i8** @"\01L_OBJC_SELECTOR_REFERENCES_", align 8, !invariant.load !5
  %3 = bitcast %struct._class_t* %1 to i8*
  %4 = tail call %0* bitcast (i8* (i8*, i8*, ...)* @objc_msgSend to %0* (i8*, i8*, i32)*)(i8* %3, i8* %2, i32 5)
  %5 = load i8** @"\01L_OBJC_SELECTOR_REFERENCES_2", align 8, !invariant.load !5
  %6 = bitcast %0* %4 to i8*
  %7 = tail call %1* bitcast (i8* (i8*, i8*, ...)* @objc_msgSend to %1* (i8*, i8*)*)(i8* %6, i8* %5)
  tail call void (i8*, ...)* @NSLog(i8* bitcast (%struct.NSConstantString* @_unnamed_cfstring_ to i8*), %1* %7)
  ret i32 0
}
```

最重要的是第四行，这里创建了一个`NSNumber`对象，第7行给number对象发送`descripton`消息，第8行打印`description`返回的string.

## 代码优化

为了能够看到`LLVM`和`clang`做了怎样的优化，我们需要一个更加复杂的C代码来演示，下面是一个递归函数：

```objectivec
#include <stdio.h>

int factorial(int x) {
   if (x > 1) return x * factorial(x-1);
   else return 1;
}

int main() {
  printf("factorial 10: %d\n", factorial(10));
}
```

使用下面的命令编译没有优化的LLVM IR:

```objectivec
clang -O0 -S -emit-llvm factorial.c -o factorial.ll
```

输出结果：

```objectivec
define i32 @factorial(i32 %x) #0 {
  %1 = alloca i32, align 4
  %2 = alloca i32, align 4
  store i32 %x, i32* %2, align 4
  %3 = load i32* %2, align 4
  %4 = icmp sgt i32 %3, 1
  br i1 %4, label %5, label %11

; <label>:5                                       ; preds = %0
  %6 = load i32* %2, align 4
  %7 = load i32* %2, align 4
  %8 = sub nsw i32 %7, 1
  %9 = call i32 @factorial(i32 %8)
  %10 = mul nsw i32 %6, %9
  store i32 %10, i32* %1
  br label %12

; <label>:11                                      ; preds = %0
  store i32 1, i32* %1
  br label %12

; <label>:12                                      ; preds = %11, %5
  %13 = load i32* %1
  ret i32 %13
}
```

你可以看到第9行`factorial`函数调用自己。这个是很低效的，因为调用堆栈会随着递归调用增加。我们使用`-O3`开关来打开优化：

```objectivec
clang -O3 -S -emit-llvm factorial.c -o factorial.ll
```

现在的输出结果是：

```objectivec
define i32 @factorial(i32 %x) #0 {
  %1 = icmp sgt i32 %x, 1
  br i1 %1, label %tailrecurse, label %tailrecurse._crit_edge

tailrecurse:                                      ; preds = %tailrecurse, %0
  %x.tr2 = phi i32 [ %2, %tailrecurse ], [ %x, %0 ]
  %accumulator.tr1 = phi i32 [ %3, %tailrecurse ], [ 1, %0 ]
  %2 = add nsw i32 %x.tr2, -1
  %3 = mul nsw i32 %x.tr2, %accumulator.tr1
  %4 = icmp sgt i32 %2, 1
  br i1 %4, label %tailrecurse, label %tailrecurse._crit_edge

tailrecurse._crit_edge:                           ; preds = %tailrecurse, %0
  %accumulator.tr.lcssa = phi i32 [ 1, %0 ], [ %3, %tailrecurse ]
  ret i32 %accumulator.tr.lcssa
}
```

尽管我们的函数的写法不是尾递归([Tail-recurse](http://en.wikipedia.org/wiki/Tail_call))，但是`clang`还是可以帮助我们优化，现在函数只有一个loop循环。还有很多`clang`的优化，GCC有一个比较好的示范 [ridiculousfish.com](http://ridiculousfish.com/blog/posts/will-it-optimize.html)

## 更多阅读

- [LLVM blog: posts tagged ‘optimization’](http://blog.llvm.org/search/label/optimization)
- [LLVM blog: vectorization improvements](http://blog.llvm.org/2013/05/llvm-33-vectorization-improvements.html)
- [LLVM blog: greedy register allocation](http://blog.llvm.org/2011/09/greedy-register-allocation-in-llvm-30.html)
- [The Polly project](http://polly.llvm.org/index.html)

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">如何利用这个优势</h1>

我们已经看到了整个编译过程，从词法分解到生成抽象树，再经过分析之后生成LLVM IR代码。我们会考虑：为什么要关系这个？

## 使用`libclang`或者`clang`插件

最让人兴奋的是`clang`是一个开源项目并且很容易编译：机会所有的东西都是库。这就意味着你可以你自己的`clang`,并且可以改变其中你需要改变的部分。比如，你可以改变`clang`生成`IR`方式，增加一个更好的类型检查，或者执行你自己的语法检测。

有很多的方式可以让你编译自己的`clang`,最方便的一种方式是使用 [libclang](http://clang.llvm.org/doxygen/group__CINDEX.html) ,`libclang`给`clang`提供了C版本的接口，你可以使用它来分析你所有的源代码。但是，以我的经验，如果你想要做出更加高级的`libclang`还是太有限了。还有一个库[ClangKit](https://github.com/macmade/ClangKit)，它是一个使用OC对clang进行封装的库。

另一种方法就是使用`clang`提供的C++库`LibTooling`.这需要更多的集成工作，这是因为涉及到C++.但是这个可以保证`clang`强大的功能。你可以做任何形式的代码分析，甚至重写你的代码。如果你想给`clang`增加你自己的分析器，希望写你自己的重构器，这需要些大量的底层代码，或者你希望根据你的项目生产文档或者图谱，`LibTooling`是你最好的选择。

## 写一个分析器

根据 [Tutorial for building tools using LibTooling](http://clang.llvm.org/docs/LibASTMatchersTutorial.html)的介绍来编译`LLVM`,`clang`和clang其他工具。保证你给编译预留了一些时间；尽管我有很快的机器，LLVM编译仍然消耗了大量的时间。

接下来，`cd ~/llvm/tools/clang/tools/`这个目录下，你可以创建你自己的clang工具。作为一例子，我们创建了一个简单的工具帮助我们正确的理解一个library的作用。下载[example repository](https://github.com/objcio/issue6-compiler-tool)然后进入这个文件夹中，输入`make`，这会帮助我们生成一个名为`example`的libray.

我们下面的使用方法：假设我们有一个类`Observer`:

```objectivec
@interface Observer
+ (instancetype)observerWithTarget:(id)target action:(SEL)selector;
@end
```

现在，每当使用此类时，我们都要检查该操作是否是存在于目标对象上的方法。我们可以写一个C++函数来做这个操作：

```objectivec
virtual bool VisitObjCMessageExpr(ObjCMessageExpr *E) {
  if (E->getReceiverKind() == ObjCMessageExpr::Class) {
    QualType ReceiverType = E->getClassReceiver();
    Selector Sel = E->getSelector();
    string TypeName = ReceiverType.getAsString();
    string SelName = Sel.getAsString();
    if (TypeName == "Observer" && SelName == "observerWithTarget:action:") {
      Expr *Receiver = E->getArg(0)->IgnoreParenCasts();
      ObjCSelectorExpr* SelExpr = cast<ObjCSelectorExpr>(E->getArg(1)->IgnoreParenCasts());
      Selector Sel = SelExpr->getSelector();
      if (const ObjCObjectPointerType *OT = Receiver->getType()->getAs<ObjCObjectPointerType>()) {
        ObjCInterfaceDecl *decl = OT->getInterfaceDecl();
        if (! decl->lookupInstanceMethod(Sel)) {
          errs() << "Warning: class " << TypeName << " does not implement selector " << Sel.getAsString() << "\n";
          SourceLocation Loc = E->getExprLoc();
          PresumedLoc PLoc = astContext->getSourceManager().getPresumedLoc(Loc);
          errs() << "in " << PLoc.getFilename() << " <" << PLoc.getLine() << ":" << PLoc.getColumn() << ">\n";
        }
      }
    }
  }
  return true;
}
```

该方法首先查找以`Observer`为接收器，以`observerWithTarget:action`为选择器的消息表达式，然后查看这个目标并检测该方法是否存在。当然，这只是一个特殊的例子，如果你希望使用AST在代码中机械的验证一些东西，上面就是你要做的东西。

## `clang`其他作用

我们还有很多其他的方式利用`clang`.比如，你可以写一个编译器插件然后可以动态加载到你的编译器中。如果你想给你的代码提供警告，你可以写一个clang插件。

如果你想大量的重构你代码，但是xcode或者appcode内置的工具没有办法满足你的要求，你可以使用clang写一个简单的重构工具。

最后你可以编译你自己的clang，然后指定xcode是用你的clang。

![iamge](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/compiler1.png)

## 建议阅读

---

[Clang 8 Document](http://clang.llvm.org/docs/LibASTMatchersTutorial.html)
[IBM Developer works](https://www.ibm.com/developerworks/cn/)
[LLVM.org](http://llvm.org)
[Clang Tutorial](https://github.com/loarabia/Clang-tutorial)
[X86_64 Assembly Language Tutorial](http://cocoafactory.com/blog/2012/11/23/x86-64-assembly-language-tutorial-part-1/)
[Custom clang Build with Xcode (I)](http://clang-analyzer.llvm.org/xcode.html)and[(II)](http://stackoverflow.com/questions/3297986/using-an-external-xcode-clang-static-analyzer-binary-with-additional-checks)
[Clang Tutorial (I)](http://kevinaboos.wordpress.com/2013/07/23/clang-tutorial-part-i-introduction/),[(II)](http://kevinaboos.wordpress.com/2013/07/23/clang-tutorial-part-ii-libtooling-example/)and[(III)](http://kevinaboos.wordpress.com/2013/07/29/clang-tutorial-part-iii-plugin-example/)
[Clang Plugin Tutorial](http://getoffmylawnentertainment.com/blog/2011/10/01/clang-plugin-development-tutorial/)
[LLVM blog: What every C programmer should know (I)](http://blog.llvm.org/2011/05/what-every-c-programmer-should-know.html),[(II)](http://blog.llvm.org/2011/05/what-every-c-programmer-should-know_14.html)and[(III)](http://blog.llvm.org/2011/05/what-every-c-programmer-should-know_21.html)

  
