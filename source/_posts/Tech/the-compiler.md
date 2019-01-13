---
title: The Build Process
date: 2017-01-10 13:47:40
categories: [Tech]
tags: [Compile,Xcode]
---

![clang](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/clang.jpeg)
粗略的来说：编译器有两个任务：将我们的`Objective-C`文件转换为底层代码，另一方面会分析我们的代码，保证我们的代码没有明显的错误。Xcode现在使用`clang`来作为编译器...

<!--more-->

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">编译器做了什么</h1>

## 预处理

### 自定义宏 Macros

## 词法分析(Tokenization or Lexing)

## 解析成抽象语法树(Parsing)

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


