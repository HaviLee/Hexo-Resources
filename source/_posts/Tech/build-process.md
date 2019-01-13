---
title: The Build Process
date: 2017-01-10 13:47:40
categories: [Tech]
tags: [Compile,Xcode]
---

![](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/xcode.jpeg)
**Because he can take it, because he's not a hero. He's a silent guardian... a watchful protector. A dark knight!  --  The Dark Knight**
**我们这些天被宠坏了-- 我们只需要点击一下xcode上面的按钮，然后一首歌的时间，或者几秒，我们的app就可以运行。非常神奇，直到出现了错误。**
<!--more-->

在这篇文章中，我们会深入编译过程，然后去剖析xcode面板上的和项目有关的设定。更加深入的去研究每一步是如何工作的。

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">破译编译日志</h1>

我们深入了解Xcode编译过程内部工作原理的第一点就是查看完整的编译log。打开导航栏，选中一个编译，xcode会为你展示详细的log。
![image](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/precompile-header.png)

默认情况下，这个面板隐藏了很多的信息，你可以选中某行的信息，点击右侧的折叠按钮查看更多的信息。另外一个方式就是选中一个或者多个信息然后按下Cmd + C,这样会复制所有的细腻。至少你可以通过编辑menu查看所有的信息。

在我们的例子中，log大概有10000行，下面开始分析：
首先我们可以注意到日志根据我们项目的target分为了不同的模块：

```objectivec
Build target Pods-SSZipArchive
...
Build target Makefile-openssl
...
Build target Pods-AFNetworking
...
Build target crypto
...
Build target Pods
...
Build target ssl
...
Build target objcio
```

我们项目有依赖第三方库：使用Pods管理AFNetworking和SSZipArchive;使用subproject管理OpenSSL.
对于每一个target,Xcode会执行一系列的步骤，将源码最终转化为机器可识别的二进制文件。我们首先来看第一个target，SSZipeArchive.
在这个target的输出日志中，我们可以看到每个target日志里面详细。比如，第一个task是处理precompiled 头文件的：

```objectivec
(1) ProcessPCH /.../Pods-SSZipArchive-prefix.pch.pch Pods-SSZipArchive-prefix.pch normal armv7 objective-c com.apple.compilers.llvm.clang.1_0.compiler
    (2) cd /.../Dev/objcio/Pods
        setenv LANG en_US.US-ASCII
        setenv PATH "..."
    (3) /.../Xcode.app/.../clang 
            (4) -x objective-c-header 
            (5) -arch armv7 
            ... configuration and warning flags ...
            (6) -DDEBUG=1 -DCOCOAPODS=1 
            ... include paths and more ...
            (7) -c 
            (8) /.../Pods-SSZipArchive-prefix.pch 
            (9) -o /.../Pods-SSZipArchive-prefix.pch.pch
```

这些块代表了每个任务的编译过程，所以我们再详细的检查下。

1.每个以行数开头的块都描述了这个任务
2.第二是执行这个task的子描述，在这个例子中，操作的目录发生变化，重新设置 `Lang` 和 `PATH`环境变量。
3.这是最有意思的地方。为了处理`.pch`文件,`clang`会设定很多的设定。这一行展示了所有的参数，我们会查看其中一些...

- 参数`-x`指定编译语言，在这里就是`objective-c-header`.
- 目标架构指定为`armv7`
- 隐式的添加`#define`
- `-c`标签告诉`clang`应该做什么。`-c`意味着运行`preprocessor`、`parser`、`type-checking`, `LLVM 生成和优化`,还有指定`target`的集成代码生成阶段。最后使用`assembly`产生 `.o`文件。
- 输入文件
- 输出文件

还有很多的需要讨论，我们不在详细讨论其他task的过程。**关键的是你需要知道在整个编译阶段调用了什么工具以及使用哪些参数**
对于这个target，实际上有两个task需要处理`objective-c-header`文件，尽管只有一个`pch`文件，我们继续看下这个发生了什么：

```objectivec
ProcessPCH /.../Pods-SSZipArchive-prefix.pch.pch Pods-SSZipArchive-prefix.pch normal armv7 objective-c ...
ProcessPCH /.../Pods-SSZipArchive-prefix.pch.pch Pods-SSZipArchive-prefix.pch normal armv7s objective-c ...
```

这个target会编译两个平台`armv7`&`armv7s`，因此clang会处理这个pch文件两次。

接下在处理`pch`的过程中，我们发现了一些其他类型的task:

```objectivec
CompileC....
Libtool...
CreateUniversalBinary...
```

从这些名字我们可以猜到它的作用：

- {% label primary@CompileC %}: 就是编译`.m`和`.c`文件
- {% label primary@Libtool %}: 根据文件创建library
- {% label primary@CreateUniversalBinary %}: 会合并前面阶段产生的两个`.a`文件（分别运行在`armv7`和`armv7s`）

项目的其他Pod的依赖会执行同样的步骤。`AFNetworking`会像`SSZipArchives`一样编译然后`link`到项目中。



直到所有的`dependencies`编译成功，才开始编译我们app自己这个`target`。输入日志中包含了一些有意思的东西：

```objectivec
PhaseScriptExecution ...
DataModelVersionCompile ...
Ld ...
GenerateDSYMFile ...
CopyStringsFile ...
CpResource ...
CopyPNGFile ...
CompileAssetCatalog ...
ProcessInfoPlistFile ...
ProcessProductPackaging /.../some-hash.mobileprovision ...
ProcessProductPackaging objcio/objcio.entitlements ...
CodeSign ...
```

唯一一个无法从编译名称直到他的作用的是`Ld`，**这个是link tool的名字,类似于`libtool`。事实上，`libtool`经常简称为`ld`和`lipo`,经常用来产生可执行文件`executables`,而`libtool`用来产生`libraries`**。

上面每一步都会调用`command line tool`执行真正的工作，就向上面处理`pch`头文件一样。但是除了了解这些日志输出，我们还需要知道：**Xcode是如何知道哪些task需要执行？**

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">控制编译过程</h1>

## 编译阶段

### 自定义编译阶段

abd

## 编译规则

bac

## 编译设置

abd

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">工程文件</h1>

akladf;
---

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">总结</h1>

aldfa;
