---
title: Mach-O Executables
date: 2017-04-10 13:47:40
categories: [Tech]
tags: [Compile,Xcode]
---

![clang](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/Mach-o.png)
当我们在Xcode上编译一个应用的时候，大部分工作都是将源代码(`.h`&`.m`)转换为可执行文件。这个可执行文件包含了在CPU上运行的二进制代码，保证可以在iOS设备上运行的ARM处理器，或在电脑上运行Intel处理器。我们下面会讨论编译器做了什么以及这个可执行文件里面是什么。
<!--more-->

让我们先把Xcode这个工具放到一边，我们下面使用command-line工具。因为我们在xcode上编译的时候，它的背后也是调用了一系列的工具。我们下面会避开xcode直接使用这些工具。希望这样可以让你更好的理解可执行文件（Mack-O executable）在iOS或者OSX上是如何工作和结合到一起的。

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">xcrun</h1>

先来普及一些内容：有一个我们经常使用的终端工具`xcrun`，看起来很古怪，但是功能很强大。这个工具经常被用来调用其他的工具：

```objectivec
clang -v
```

如果我们使用`xcrun`:

```objectivec
xcrun clang -v
```

`xcrun`的作用是找到`clang`的位置，然后调用`clang`.

为什么要这样做？这看起来没什么意义，原因是：

- `xcrun`可以让我们使用多个版本的xcode，在使用的时候指定Xcode的版本。
- 可以指定`SDK`

如果你的电脑同时有Xcode4.5和Xcode5。你可以使用`xcode-select`&`xcrun`指定使用Xcode5的 SDK,在其他平台这几乎是不可能的。你可以在`man page`查看`xcrun`&`xcrun`的详细信息。同时你也可以在不安装`command line tools`工具的前提下使用开发工具。

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">不适用IDE编译Hello World</h1>

返回到我们的终端，创建一个包含C文件的文件夹：

```objectivec
mkdir ~/Desktop/objcio-command-line
cd !$
touch helloworld.c
```

下面你可以编辑这个文件：

```objectivec
open -e helloworld.c
```

我们输入下面的内容：

```objectivec
#include <stdio.h>
int main(int argc, char *argv[])
{
    printf("Hello World!\n");
    return 0;
}
```

保存然后在终端输入：

```objectivec
xcrun clang helloworld.c
./a.out
```

你现在可以在终端中直接看到`hello world!`.上面的命令编译了C代码，并且运行，在没有IDE的前提下。

这里我们做了什么？我们把`helloworld.c`文件编译成为Mach-O文件`a.out`.这是编译器生成的默认名称，你也可以给这个文件设定名称。那么这个文件是如何生成的呢？我们下面可以看下具体的编译过程。

## 使用编译器编译Hello world

现在的xcode编译器前端使用的是`clang`，详情[about compiler](https://www.objc.io/issues/6-build-tools/compiler/).简单的来说，编译器会把输入文件`helloworld.c`处理成为可执行文件`a.out`.这个处理过程包过多个处理阶段，我们总结下：

**{% label danger@预处理 %}**

- 词法分解：将前端源码进行词法分解
- 宏展开
- 头文件展开

**{% label danger@解析和语法分析 %}**

- 将预处理程序标记转换为解析树
- 将语义分析应用于解析树
- 生成AST（Abstract Syntax Tree)语法抽象树

**{% label danger@代码生成和优化 %}**

- 将AST转换为低级中间代码（LLVM IR）
- 负责优化生成的中间码 IR
- 指定运行平台的代码生成
- 输出汇编代码(assembly)

**{% label danger@汇编(Assembler) %}**

- 将汇编代码转换为目标对象文件即`.o`文件。

**{% label danger@链接器 %}**

- 将多个目标文件合并为可执行文件(executable)（或动态库）

下面我们以一个例子来说明整个编译过程：

### 预处理

编译器做的第一件事就是预处理源代码文件。我们让clang输出预处理后的代码：

```objectivec
xcrun clang -E helloworld.c
```

得到大概有400多行，用下面的命令打开看看：

```objectivec
xcrun clang -E helloworld.c | open -f
```

你会看到输出文件底层有很多以`#`开头的内容，这些是行标记告诉我们的代码是来自哪一行。重新看下我们的`helloworld.c`你看到第一行是

```objectivec
#include <stdio.h>
```

我们之前都使用过`#include`&`#import`。这行命令告诉编译器在写这个命令的地方插入`stdio.h`文件的内容。这是一个递归的过程，因为`stdio.h`还有可能引用了其他头文件。

由于有太多的递归文件需要插入，因此我们需要追踪插入的代码在原始代码中的位置。为此，预处理器会在原始位置发生变化时插入以＃开头的行标记。`#`后面跟的是在原始文件中的行号，再后面是文件名。该行最后的数字是一些指示标识：（1）是指示新文件的开始，（2）返回某一个文件，（3）是来自系统头的标志，或者该文件将被处理的标志包裹在一个外部的“C”块中。

如果你滑到最后面，你可以看到`helloworld.c`的代码：

```objectivec
# 2 "helloworld.c" 2
int main(int argc, char *argv[])
{
 printf("Hello World!\n");
 return 0;
}
```

在Xcode中你可以通过设置`Product -> Perform Action -> Preprocess`.加载预处理文件需要一定的时间，生成的文件大概有万行。

### 编译阶段

下面就是对经过预处理的代码进行代码分析和生成。我们可以使用`clang`命令生成汇编`assembly`文件：

```objectivec
xcrun clang -S -o - helloworld.c | open -f

out put:
```

我们可以看下输出内容，我们注意到很多行都是以`.`开头，这些是汇编程序指令。这个实际上是`x86_64`汇编指令，还有就是标签，类似C语言中的。我们先介绍下面三行：

```objectivec
.section    __TEXT,__text,regular,pure_instructions
.globl  _main
.align  4, 0x90
```

这三行是汇编程序指令，而不是汇编代码。

- {% label success@.section %}:.section指令指定以下内容将进入哪个部分。后面会有section详细介绍。
- {% label success@.globl %}:指定了`main`是一个外部符号，这个就是我们的`main()`函数。它需要在我们的`library`之外对外暴露，因为我们的系统需要在运行可执行文件的时候调用它。
- {% label success@.align %}:指令指定以下内容的对齐方式.在我们的例子中，如果需要，下面的代码将是16（2^4）字节对齐并用0x90填充。

  下面是主函数的代码：

```objectivec
_main:                                  ## @main
    .cfi_startproc
## BB#0:
    pushq   %rbp
Ltmp2:
    .cfi_def_cfa_offset 16
Ltmp3:
    .cfi_offset %rbp, -16
    movq    %rsp, %rbp
Ltmp4:
    .cfi_def_cfa_register %rbp
    subq    $32, %rsp
```

这部分有很多类似于C代码中的标签。它们是对汇编代码某些部分的符号引用。首先是我们主函数`_main`的开始。这也是导出的符号。因此，二进制文件将引用此位置。

`.cfi_startproc`指令用于大多数函数的开头。CFI(Call Frame Information)是呼叫帧信息的缩写。一个帧信息对应于一个函数。当使用调试器进行调试时，实际上是单步执行/单步执行调用帧。在C函数中，函数有他们自己的调用帧，其他也会有。`.cfi_startproc`给函数提供了进入`.eh_frame`的入口，`eh_frame`包含了未展开的信息。这就是异常如何展开调用帧堆栈。该指令还将为CFI发出依赖于体系结构的指令。在我们输入文件中与相应的.cfi_endproc进行匹配，以标记main（）函数的结尾。

下面是标签`## BB#0:`,下面是我们的第一个汇编代码`pushq %rbp`.后面开始有意思了。在OS X上，我们有x86_64代码，对于这个体系结构，有一个所谓的应用程序二进制接口（ABI- application binary interface），它指定函数调用在汇编代码级别的工作方式。此ABI的一部分指定必须在函数调用之间保留`rbg`寄存器（基指针寄存器）.当函数返回时，确保`rbg`寄存器具有相同的值是主函数的责任。`pushq%rbp`将其值推送到堆栈上，以便稍后弹出。

`.cfi_def_cfa_offset 16 and .cfi_offset %rbp, -16`是另外两个CFI指令，同样，这些将输出与生成调用帧展开信息和调试信息相关的信息。我们通过改变栈和基指针，基本上告诉了调试器代码的位置，或者更确切地说，它们将导致信息被输出，调试器随后可以使用这些信息来找到它的方法。

下面`movq%rsp，%rbp`将允许我们将局部变量放入堆栈,`subq$32%rsp`将堆栈指针移动32个字节，然后函数可以使用这些字节,我们首先将旧堆栈指针存储在rbp中，并将其用作本地变量的基础，然后将堆栈指针更新为我们将使用的部分。

下面我们会调用`printf()`：

```objectivec
leaq    L_.str(%rip), %rax
movl    $0, -4(%rbp)
movl    %edi, -8(%rbp)
movq    %rsi, -16(%rbp)
movq    %rax, %rdi
movb    $0, %al
callq   _printf
```

首先, `leaq` 将指向 `L_.str` 的指针加载到 `rax` 寄存器中。请注意 `L_.str`标签是如何在汇编代码中进一步定义的。这是我们的c字符串 "helloworld!\n"。`edi` 和 `rsi` 寄存器保存函数的第一个和第二个参数。由于我们将调用另一个函数, 我们首先需要存储它们的当前值。这就是我们将使用基于我们刚才为其保留的 `rbp` 的32个字节的内容。首先是 32-bit 0, 然后是 `edi` 寄存器 (它保存 `argc`) 的32位值, 然后是 `rsi` 寄存器 (它保存 argc) 的64位值。我们以后不会使用这些值, 但由于编译器在没有优化的情况下运行, 因此无论如何它都会存储它们。

现在, 我们将函数printf()的第一个参数`rax`传递给寄存器函数`edi`的第一个参数。printf()函数是一个可变函数。abi调用约定指定用于保存参数的向量寄存器的数量需要存储在`al`寄存器中。在我们的例子中, 它是0。最后, `callq`调用 printf() 函数:

```objectivec
movl    $0, %ecx
movl    %eax, -20(%rbp)         ## 4-byte Spill
movl    %ecx, %eax
```

这将 ecx 寄存器设置为 0, 将 eax 寄存器保存 (溢出) 到堆栈上, 然后将 ecx 中的0值复制到 eax 中。abi 指定 eax 将保存函数的返回值, 而我们的 main () 函数返回 0:

```objectivec
addq    $32, %rsp
popq    %rbp
ret
.cfi_endproc
```

完成后, 我们将通过将堆栈指针 rsp 向后移动32个字节来恢复堆栈指针, 以撤消上面的 subq $32,% rsp 的效果。最后, 我们将弹出之前存储的 rbp 的值, 然后用 ret 返回到调用方, 它将读取堆栈上的返回地址。. cfi _ endproc 平衡. cfi _ startproc 指令。

接下来是我们的字符串文本`helloworld!`:

```objectivec
    .section    __TEXT,__cstring,cstring_literals
L_.str:                                 ## @.str
    .asciz   "Hello World!\n"
```

同样,`.`指令指定了以下需要进入哪个部分。`L_. str` 标签允许实际代码获取指向字符串文本的指针。`.asciz` 指令告诉汇编器输出一个0终止的字符串文本。
这将启动一个新的section `__TEXT __cstring`。这个section包含 c 字符串:

```objectivec
L_.str:                                 ## @.str
    .asciz     "Hello World!\n"
```

这两行创建一个空终止字符串。请注意 `L_.str` 是如何进一步用于访问字符串的名称。
`.subsections_via_symbols`指令用于静态链接编辑器。

更多汇编指令查看 [OS X Assembler Reference](https://developer.apple.com/library/mac/documentation/DeveloperTools/Reference/Assembler/),The AMD 64 website has documentation on the [application binary interface for x86_64](http://www.x86-64.org/documentation/abi.pdf). It also has a [Gentle Introduction to x86-64 Assembly](http://www.x86-64.org/documentation/assembly.html).

最后，你可以通过`Product -> Perform Action -> Assemble`来查看生成的汇编代码。

### 汇编器（Assembler）

简单地说, 汇编器将 (人类可读的) 汇编代码转换为机器代码。它创建一个目标对象文件, 通常被简单地称为对象文件。这些文件以`.o`结尾。如果你使用xcode编译你的app，你可以在你app的`drived data`文件夹里面的`Objects-normal`文件夹里面找到这些文件。

### 链接器

下面会详细的介绍连接器。但简单地说, 链接器会在对象文件和库之间建立符号链接。什么意思？回顾：

```objectivec
callq   _printf
```

`printf()`是`libc`库中的一个函数。有时候，最终的可执行文件需要能够知道 `printf ()` 在内存中的位置, 即 `_ printf` 符号的地址是什么。链接器获取所有对象文件 (在我们的示例中, 只有一个) 和库 (在我们的示例中, libc), 并链接任何未知的符号 (在我们的示例中, `_ printf`)。然后, 它将此符号编码到最终可执行文件中, 该符号可以在 `libc` 中找到, 链接器然后输出可以运行的最终可执行文件:`a.out`。

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">Sections 执行文件</h1>

正如我们上面提到的, 有个东西成为sections.一个可执行文件有多个section.可执行文件的不同section将分别进入自己的section, 每个section依次进入一个segment。这适用于我们的临时应用, 也适用于一个完整的二进制应用。

让我们来看看我们的`.out` 二进制文件的各个section。我们可以使用`size`工具来查看:

```objectivec
% xcrun size -x -l -m a.out 
Segment __PAGEZERO: 0x100000000 (vmaddr 0x0 fileoff 0)
Segment __TEXT: 0x1000 (vmaddr 0x100000000 fileoff 0)
    Section __text: 0x37 (addr 0x100000f30 offset 3888)
    Section __stubs: 0x6 (addr 0x100000f68 offset 3944)
    Section __stub_helper: 0x1a (addr 0x100000f70 offset 3952)
    Section __cstring: 0xe (addr 0x100000f8a offset 3978)
    Section __unwind_info: 0x48 (addr 0x100000f98 offset 3992)
    Section __eh_frame: 0x18 (addr 0x100000fe0 offset 4064)
    total 0xc5
Segment __DATA: 0x1000 (vmaddr 0x100001000 fileoff 4096)
    Section __nl_symbol_ptr: 0x10 (addr 0x100001000 offset 4096)
    Section __la_symbol_ptr: 0x8 (addr 0x100001010 offset 4112)
    total 0x18
Segment __LINKEDIT: 0x1000 (vmaddr 0x100002000 fileoff 8192)
total 0x100003000
```

我们的`a.out`具有四个segment,每个segment还有其他的section。

当我们运行可执行文件时, VM(Virtual memory) (虚拟内存) 系统会将section映射到进程的地址空间 (即内存) 中.映射在本质上是非常不同的, 但如果您不熟悉 vm系统, 只需假设 vm 将整个可执行文件加载到内存中, 虽然这不是真正发生的事情。vm 做了一些改进, 避免这样做。

当 vm 系统执行此映射时, segment和section将映射为不同的属性, 有不同的权限。

- {% label danger@_ _TEXT %}: 包含了我们的执行代码。它映射为只读和可执行。允许执行该Section代码, 但不允许对其进行修改。代码不能改变自身, 因此这些映射的页面永远不会变得肮脏。

- {% label danger@_ _DATA %}: 映射为可读写但是不可执行。它包含了需要更新的一些信息。

- {% label danger@_ _PAGEZERO %}:它的大小是4G.这个4G 并不是实际文件大小, 是指定进程地址空间的第一个映射为不可执行、不可写、不可读的文件。这就是为什么在读取或写入null 指针您将获得` exc_bad_access`, 或者其他一些 (相对) 较小的值。这是操作系统试图防止你造成严重[破坏](http://www.xkcd.com/371/)。

在segment还包含了section。它们包含可执行文件的不同部分。在`_ _TEXT`segment中，`_ _text`包含了编译过的二进制代码。`_ _stubs`&`_ _stub_helper`是用于动态连接器（dyld）。这允许在动态链接的代码中进行懒链接。`_ _const`(我们的例子中没有)是常量，`_cstring` 包含可执行文件的文本字符串常量 (源代码中引用的字符串)。

在`_ _DATA` segment中包含了可读写的数据。在我们例子中只有`_ _no_symbol_ptr`&`_ _la_symbol_ptr`，它们分别是非懒惰和懒惰的符号指针。懒惰符号指针用于可执行文件调用的所谓未定义函数，即不在可执行文件本身中的函数。他们可以被动态地解决。非懒惰指针时在可执行文件加载的时候被链接。

在`_ _DATA` segment其他的一些section是`_ _const`.其中将包含需要重新分配的常量数据。一个例子是 `char * const p = "foo"`; `p` 指向的数据不是恒定不变的。`_ bss`  section包含未初始化的静态变量, 如静态 `int a`; **ansi c** 标准指定静态变量必须设置为零。但它们可以在运行时更改。`_common`section包含未初始化的全局全局, 类似于静态变量。一个例子是 int a;在函数块之外。最后, `_dyld` 是一个占位符section, 由动态链接器使用。

Apple’s [OS X Assembler Reference](https://developer.apple.com/library/mac/documentation/DeveloperTools/Reference/Assembler/) 查看详细信息。

## Section内容

我们可以用 `otool` 来检查一个section的内容:

```objectivec
% xcrun otool -s __TEXT __text a.out 
a.out:
(__TEXT,__text) section
0000000100000f30 55 48 89 e5 48 83 ec 20 48 8d 05 4b 00 00 00 c7 
0000000100000f40 45 fc 00 00 00 00 89 7d f8 48 89 75 f0 48 89 c7 
0000000100000f50 b0 00 e8 11 00 00 00 b9 00 00 00 00 89 45 ec 89 
0000000100000f60 c8 48 83 c4 20 5d c3 
```

这个是我们app的代码。由于`-s __TEXT __text`非常常见，`otool` 有一个带有`-t `参数的快捷方式。我们甚至可以通过添加-v 来查看详细的暑促:

```objectivec
% xcrun otool -v -t a.out
a.out:
(__TEXT,__text) section
_main:
0000000100000f30    pushq   %rbp
0000000100000f31    movq    %rsp, %rbp
0000000100000f34    subq    $0x20, %rsp
0000000100000f38    leaq    0x4b(%rip), %rax
0000000100000f3f    movl    $0x0, 0xfffffffffffffffc(%rbp)
0000000100000f46    movl    %edi, 0xfffffffffffffff8(%rbp)
0000000100000f49    movq    %rsi, 0xfffffffffffffff0(%rbp)
0000000100000f4d    movq    %rax, %rdi
0000000100000f50    movb    $0x0, %al
0000000100000f52    callq   0x100000f68
0000000100000f57    movl    $0x0, %ecx
0000000100000f5c    movl    %eax, 0xffffffffffffffec(%rbp)
0000000100000f5f    movl    %ecx, %eax
0000000100000f61    addq    $0x20, %rsp
0000000100000f65    popq    %rbp
0000000100000f66    ret
```

它应该看起来很熟悉--这是我们在编译代码时看到的汇编代码。唯一不同的是, 我们在这里没有任何汇编器指令;这是纯粹二进制可执行文件。

使用相同的命令可以查看其他的section：

```objectivec
% xcrun otool -v -s __TEXT __cstring a.out
a.out:
Contents of (__TEXT,__cstring) section
0x0000000100000f8a  Hello World!\n
```

或者：

```objectivec
% xcrun otool -v -s __TEXT __eh_frame a.out 
a.out:
Contents of (__TEXT,__eh_frame) section
0000000100000fe0    14 00 00 00 00 00 00 00 01 7a 52 00 01 78 10 01 
0000000100000ff0    10 0c 07 08 90 01 00 00 
```

### 性能问题

`_DATA` 和 `_TEXT `段具有性能影响。如果您有一个非常大的二进制文件, 您可能需要查看 apple 关于[代码大小性能准则的文档](https://developer.apple.com/legacy/library/documentation/Performance/Conceptual/CodeFootprint/CodeFootprint.pdf)。将数据移动到 `_TEXT `是有益的, 因为这些页面永远不会被修改的。

### Arbitrary Sections

您可以使用`-sectcreate` 链接器标志将任意数据作为section添加到可执行文件中。这就是为什么可以将` info. plist` 添加到单个文件可执行文件中。`info. plist` 数据需要进入 `_ _TEXT` 段的 `_ _info_plist `部分。你可以通过命令`-sectcreate segname sectname file`调用链接器传递给clang.

```objectivec
-Wl,-sectcreate,__TEXT,__info_plist,path/to/Info.plist
```

同样的你可以使用`-sectalign`指定对齐。如果要添加一个全新的段, 请使用`-segprot`以指定segment的保护 (即读/写/可执行)。

你可以使用`/usr/include/mach-o/getsect.h`中的函数`getsectdata()`来获取sections,它将为您提供指向节数据的指针, 并通过引用返回其长度。

## Mach-O 文件

`os x `和 `ios `上的可执行文件是 [mach-o ](https://en.wikipedia.org/wiki/Mach-o)可执行文件:

```objectivec
file a.out 
a.out: Mach-O 64-bit executable x86_64
```

对于我们的GUI工具输出结果也是一样的：

```objectivec
file /Applications/Preview.app/Contents/MacOS/Preview 
/Applications/Preview.app/Contents/MacOS/Preview: Mach-O 64-bit executable x86_64
```

苹果关于 [Mach-O file format](https://developer.apple.com/library/mac/documentation/DeveloperTools/Conceptual/MachORuntime/index.html) 的详情。

我们可以使用 `otool(1)` 来查看可执行文件的Mach 头文件。它指定此文件是什么以及如何加载它。我们将使用`-h `输出头文件信息:

```objectivec
% otool -v -h a.out           a.out:
Mach header
      magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
MH_MAGIC_64  X86_64        ALL LIB64     EXECUTE    16       1296   NOUNDEFS DYLDLINK TWOLEVEL PIE
```

`cputype` 和` cpusubtype` 指定此可执行文件可以运行的cpu架构。`ncmds` 和`sizeofcmds` 是加载命令, 我们可以用`-l` 参数查看:

```objectivec
% otool -v -l a.out | open -f
a.out:
Load command 0
      cmd LC_SEGMENT_64
  cmdsize 72
  segname __PAGEZERO
   vmaddr 0x0000000000000000
   vmsize 0x0000000100000000
...
```

加载命令指定文件的逻辑结构及其在虚拟内存中的布局。打印出来的大部分信息都来自这些加载命令。在`load command 1`部分中, 我们找到 `initprot r-x`, 它指定了上面提到的保护: 只读 (无写入) 和可执行。

在segment中的每一个segment和section，load command 指定了他们的结束位置，以及保护内容等，下面是`_ _TEXT _ _text`的输出：

```objectivec
Section
  sectname __text
   segname __TEXT
      addr 0x0000000100000f30
      size 0x0000000000000037
    offset 3888
     align 2^4 (16)
    reloff 0
    nreloc 0
      type S_REGULAR
attributes PURE_INSTRUCTIONS SOME_INSTRUCTIONS
 reserved1 0
 reserved2 0
```

我们的代码在内存中的结尾地址是`0x100000f30`，它的偏移量是3888.如果你使用`xcrun otool -v -t a.out`查看详细信息，你会看到代码的实际地址是`0x100000f30`。

我们还可以查看可执行文件使用的动态库:

```objectivec
% otool -v -L a.out
a.out:
    /usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 169.3.0)
    time stamp 2 Thu Jan  1 01:00:02 1970
```

这是我们的可执行文件将找到它正在使用的 _ printf 符号的地方。

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">复杂的例子</h1>

我们看一个有三个文件的复杂例子：

{% label danger:@Foo.h %}:

```objectivec
#import <Foundation/Foundation.h>

@interface Foo : NSObject

- (void)run;

@end
```

{% label danger:@Foo.m %}:

```objectivec
#import "Foo.h"

@implementation Foo

- (void)run
{
    NSLog(@"%@", NSFullUserName());
}

@end
```

{% label danger:@helloworld.m %}:

```objectivec
#import "Foo.h"

int main(int argc, char *argv[])
{
    @autoreleasepool {
        Foo *foo = [[Foo alloc] init];
        [foo run];
        return 0;
    }
}
```

## 编译多个文件

在此示例中, 我们有多个文件。因此, 我们需要告诉 clang 首先为每个输入文件生成对象文件:

```objectivec
xcrun clang -c Foo.m
xcrun clang -c helloworld.m
```

我们永远不会编译头文件。头文件目的只是在编译的实现文件之间共享代码。foo.m 和 hellowd.m 通过 #import 声明引入了 foo.h 的内容。我们得到两个目标文件：

```objectivec
% file helloworld.o Foo.o
helloworld.o: Mach-O 64-bit object x86_64
Foo.o:        Mach-O 64-bit object x86_64
```

为了生成可执行文件, 我们需要将这两个对象文件和基础框架相互链接:

```objectivec
xcrun clang helloworld.o Foo.o -Wl,`xcrun --show-sdk-path`/System/Library/Frameworks/Foundation.framework/Foundation
```

运行我们的代码：

```objectivec
% ./a.out 
2013-11-03 18:03:03.386 a.out[8302:303] Daniel Eggert
```

## 符号化和链接

我们的小应用程序是从两个对象文件放在一起的,`foo.o`对象文件包含 foo 类的实现, `helloword.o` 对象文件包含 main () 函数, 并且调用foo 类。

此外, 这两者都使用了基础框架Foundation。`helloworld.o`将foundation用在自动释放池，并且通过`libobjc.dylib`间接的使用了runtime。因为它需要runtime发送消息。这个和Foo.o文件类似。

所有这些都被表示为所谓的符号。我们可以把一个符号看作是和程序运行有关的一个指针, 虽然它的性质稍有不同。

定义或使用的每个函数、全局变量、类等都会产生一个符号。当我们将对象文件链接到可执行文件时, 链接器 (ld(1)) 会根据需要在对象文件和动态库之间创建链接符号。

可执行文件和对象文件有一个符号表, 用于指定它们的符号。如果我们用 nm(1) 工具查看 helloworld. o 对象文件, 我们会得到这样的信息:

```objectivec
% xcrun nm -nm helloworld.o
                 (undefined) external _OBJC_CLASS_$_Foo
0000000000000000 (__TEXT,__text) external _main
                 (undefined) external _objc_autoreleasePoolPop
                 (undefined) external _objc_autoreleasePoolPush
                 (undefined) external _objc_msgSend
                 (undefined) external _objc_msgSend_fixup
0000000000000088 (__TEXT,__objc_methname) non-external L_OBJC_METH_VAR_NAME_
000000000000008e (__TEXT,__objc_methname) non-external L_OBJC_METH_VAR_NAME_1
0000000000000093 (__TEXT,__objc_methname) non-external L_OBJC_METH_VAR_NAME_2
00000000000000a0 (__DATA,__objc_msgrefs) weak private external l_objc_msgSend_fixup_alloc
00000000000000e8 (__TEXT,__eh_frame) non-external EH_frame0
0000000000000100 (__TEXT,__eh_frame) external _main.eh
```

这些都是helloworld文件的符号。`_OBJC_CLASS_$_Foo`是OC类Foo的符号。它是Foo类的未定义的外部符号。`External`表示这个符号不是这个对象文件的私有，`no-external`表示这个符号仅对对象文件可见。我们的`helloworld.o`对象文件引用了类Foo，但它没有实现它.因此在符号表最后标记为`undefined`.

接下来，`main()`函数的_main符号也是外部的，因为它需要是可见的才能被调用.但main函数在在helloworld中实现了，分配的内存地址是0，并且需要提供`__TEXT,__text` section的入口。下面是四个`runtime`运行时函数。这些也是没有定义的，需要通过链接器实现。

下面我们看下`Foo.o`的对象文件:
```objectivec
% xcrun nm -nm Foo.o
0000000000000000 (__TEXT,__text) non-external -[Foo run]
                 (undefined) external _NSFullUserName
                 (undefined) external _NSLog
                 (undefined) external _OBJC_CLASS_$_NSObject
                 (undefined) external _OBJC_METACLASS_$_NSObject
                 (undefined) external ___CFConstantStringClassReference
                 (undefined) external __objc_empty_cache
                 (undefined) external __objc_empty_vtable
000000000000002f (__TEXT,__cstring) non-external l_.str
0000000000000060 (__TEXT,__objc_classname) non-external L_OBJC_CLASS_NAME_
0000000000000068 (__DATA,__objc_const) non-external l_OBJC_METACLASS_RO_$_Foo
00000000000000b0 (__DATA,__objc_const) non-external l_OBJC_$_INSTANCE_METHODS_Foo
00000000000000d0 (__DATA,__objc_const) non-external l_OBJC_CLASS_RO_$_Foo
0000000000000118 (__DATA,__objc_data) external _OBJC_METACLASS_$_Foo
0000000000000140 (__DATA,__objc_data) external _OBJC_CLASS_$_Foo
0000000000000168 (__TEXT,__objc_methname) non-external L_OBJC_METH_VAR_NAME_
000000000000016c (__TEXT,__objc_methtype) non-external L_OBJC_METH_VAR_TYPE_
00000000000001a8 (__TEXT,__eh_frame) non-external EH_frame0
00000000000001c0 (__TEXT,__eh_frame) non-external -[Foo run].eh
```
倒数第五行显示`_OBJC_CLASS_$_Foo`已定义并且对Foo.o外部可见 - 因为这个类已实现。`Foo.o`也有未定义的符号，首先是我们正在使用的NSFullUserName()，NSLog()和NSObject()的符号.

**当我们把这两个对象文件和Foundation(一个动态库)进行链接的时候，动态链接器会帮助我们链接所有未定义的符号**,它可以通过这种方式解析`_OBJC_CLASS_$_Foo`因此它将需要使用Foundation框架。
当链接器通过动态库（在我们的例子中是Foundation框架）解析符号时，它将记录最终链接的images,它保存了使用该动态库解析的符号。链接器记录输出文件依赖的特定的动态库，以及它的路径。这就是我们的例子中`_NSFullUserName，_NSLog，_OBJC_CLASS _ $ _ NSObject，_objc_autoreleasePoolPop`等符号所发生的情况。

我们可以查看最终可执行文件a.out的符号表，并查看链接器如何解析所有符号：
```objectivec
% xcrun nm -nm a.out 
                 (undefined) external _NSFullUserName (from Foundation)
                 (undefined) external _NSLog (from Foundation)
                 (undefined) external _OBJC_CLASS_$_NSObject (from CoreFoundation)
                 (undefined) external _OBJC_METACLASS_$_NSObject (from CoreFoundation)
                 (undefined) external ___CFConstantStringClassReference (from CoreFoundation)
                 (undefined) external __objc_empty_cache (from libobjc)
                 (undefined) external __objc_empty_vtable (from libobjc)
                 (undefined) external _objc_autoreleasePoolPop (from libobjc)
                 (undefined) external _objc_autoreleasePoolPush (from libobjc)
                 (undefined) external _objc_msgSend (from libobjc)
                 (undefined) external _objc_msgSend_fixup (from libobjc)
                 (undefined) external dyld_stub_binder (from libSystem)
0000000100000000 (__TEXT,__text) [referenced dynamically] external __mh_execute_header
0000000100000e50 (__TEXT,__text) external _main
0000000100000ed0 (__TEXT,__text) non-external -[Foo run]
0000000100001128 (__DATA,__objc_data) external _OBJC_METACLASS_$_Foo
0000000100001150 (__DATA,__objc_data) external _OBJC_CLASS_$_Foo
```
我们看到所有的Foundation和Objective-C运行时符号仍未定义，但符号表现在有关于如何解决它们的信息，即它们将在哪个动态库中找到。

可执行文件还知道在哪里找到这些库：
```objectivec
% xcrun otool -L a.out
a.out:
    /System/Library/Frameworks/Foundation.framework/Versions/C/Foundation (compatibility version 300.0.0, current version 1056.0.0)
    /usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1197.1.1)
    /System/Library/Frameworks/CoreFoundation.framework/Versions/A/CoreFoundation (compatibility version 150.0.0, current version 855.11.0)
    /usr/lib/libobjc.A.dylib (compatibility version 1.0.0, current version 228.0.0)
```
这些未定义的符号在运行时由动态链接器dyld(1)解析。 当我们运行可执行文件时，dyld将确保_NSFullUserName等指向它们在Foundation中的实现等。

我们可以对Foundation运行nm（1）并检查这些符号的定义：

```objectivec
% xcrun nm -nm `xcrun --show-sdk-path`/System/Library/Frameworks/Foundation.framework/Foundation | grep NSFullUserName
0000000000007f3e (__TEXT,__text) external _NSFullUserName 
```

## 动态链接编辑器
有一些变量可用于查看dyld的用途。 首先，`DYLD_PRINT_LIBRARIES`可以让dyld将打印出加载的库：
```objectivec
% (export DYLD_PRINT_LIBRARIES=; ./a.out )
dyld: loaded: /Users/deggert/Desktop/command_line/./a.out
dyld: loaded: /System/Library/Frameworks/Foundation.framework/Versions/C/Foundation
dyld: loaded: /usr/lib/libSystem.B.dylib
dyld: loaded: /System/Library/Frameworks/CoreFoundation.framework/Versions/A/CoreFoundation
dyld: loaded: /usr/lib/libobjc.A.dylib
dyld: loaded: /usr/lib/libauto.dylib
[...]
```

这将向您展示作为加载Foundation的一部分的其他所有70个动态库。 那是因为Foundation依赖于其他动态库，而这些动态库依赖于其他动态库，依此类推：
```objectivec
% xcrun otool -L `xcrun --show-sdk-path`/System/Library/Frameworks/Foundation.framework/Foundation
```
查看Foundation使用的十五个动态库的列表
## 动态库共享缓存

当您构建一个真实世界的应用程序时，您将链接到各种Framework。 而这些反过来将使用无数其他Framework和dynamic libraries。需要加载的所有动态库的列表会很快变大。 相互依存的符号列表更是如此。 将有数千个符号要解决。 这项工作需要很长时间：几秒钟。

为了简化此过程，OS-X和iOS上的动态链接器使用位于`/var/db/dyld/`内的共享缓存.对于每个体系结构，操作系统都有一个文件，其中包含几乎所有已经链接在一起的动态库，并且它们的相互依赖的符号已经解析。加载Mach-O文件（可执行文件或库）时，动态链接器将首先检查它是否在此共享高速缓存映像中，如果是，则从共享高速缓存中使用它。每个进程都已将此dyld共享缓存映射到其地址空间。 此方法可显着缩短OS X和iOS上的启动时间。
