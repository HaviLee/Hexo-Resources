---
title: The Build Process
date: 2017-01-10 13:47:40
categories: 
  - Tech
tags:
  - Xcode Compiler 
---

>  Because he can take it,because he's not a hero.He's a silent guardian... a watchful protector.A dark knight!  --  <The Dark Knight>

我们这些天被宠坏了-- 我们只需要点击一下xcode上面的按钮，然后一首歌的时间，或者几秒，我们的app就可以运行。非常神奇，直到出现了错误。

在这篇文章中，我们会深入编译过程，然后去剖析xcode面板上的和项目有关的设定。更加深入的去研究每一步是如何工作的。

## 破译编译日志

我们深入了解Xcode编译过程内部工作原理的第一点就是查看完整的编译log。打开导航栏，选中一个编译，xcode会为你展示详细的log。
![image](https://raw.githubusercontent.com/Interview-Skill/2019-Blog/Images/xcode.png)

默认情况下，这个面板隐藏了很多的信息，你可以选中某行的信息，点击右侧的折叠按钮查看更多的信息。另外一个方式就是选中一个或者多个信息然后按下Cmd + C,这样会复制所有的细腻。至少你可以通过编辑menu查看所有的信息。

在我们的例子中，log大概有10000行，下面开始分析：
首先我们可以注意到日志根据我们项目的target分为了不同的模块：

```php
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

```php
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

1. 每个以行数开头的块都描述了这个任务
2. 第二是执行这个task的子描述，在这个例子中，操作的目录发生变化，重新设置 `Lang` 和 `PATH`环境变量。
3. 这是最有意思的地方。为了处理`.pch`文件

## 控制编译过程

### 编译阶段

abd

### 编译规则

bac

### 编译设置

abd

## 工程文件

akladf;

## 总结

aldfa;
