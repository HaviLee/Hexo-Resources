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

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">一. xcrun</h1>





<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">二. 不适用IDE编译Hello World</h1>

## 1. 使用编译器编译Hello world

### a. 预处理

### b. 编译代码

### c. 汇编程序

### d. 链接器





<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">三. Sections 执行文件</h1>


## 1. Section内容

### a. 性能备注

### b. Arbitrary Sections


## 2. Mach-O 文件






<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">四. 复杂的例子</h1>

## 1. 编译多个文件


## 2. 符号化和链接

## 3. 动态链接编辑器

## 4. 动态库共享缓存
