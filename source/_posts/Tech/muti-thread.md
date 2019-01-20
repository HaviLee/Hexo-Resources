---
title: 你理解的MVC对吗？
date: 2017-08-11 13:47:40
categories: 
  - Objective-C
tags:
  - MutiThread
---

![clang](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/thread.png)

[Concurrency](http://en.wikipedia.org/wiki/Concurrency_%28computer_science%29) 表示在同一时间运行多个任务的一个概念。这可以在单个 cpu 内核上以[时间共享](http://en.wikipedia.org/wiki/Preemption_%28computing%29)的方式发生, 也可以在多个 cpu 内核可用的情况下真正并行发生。

OSX和iOS提供了不同的API让我们来进行并发编程。不同的API具有不同的功能和局限性，可以操作不同的任务。他们是基于底层API的不同层次的封装。我们可以直接操作底层API来进行并发编程，但同时也需要负责更多的其他的工作。

并发编程是一个非常困难的主题, 有许多复杂的问题和陷阱,尤其在使用`GCD(Grand Central Dispatch)`和`NSOperationQueue`的时候。我们首先了解下OSX和iOS的不同的并发API,然后去探讨不同API的难点，这可以帮组你了解不同的API。

<!-- more -->

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">OSX和iOS的并发API</h1>





## Threads



## GCD

## Operation Queues

## Run Loops





<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">并发编程的挑战</h1>





<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">编程实践</h1>

## 共享资源

## 互斥

## 死锁

## 其他补充

## 优先级反转

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">结论</h1>


