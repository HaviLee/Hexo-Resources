---
title: 多线程编程：APIS和挑战
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

Apple为OSX和iOS提供了并发编程相同的API接口。下面我们会介绍`pthread`,`NSThread`,`GCD`,`NSOperationQueue`和`Runloop`。从技术上来说，runloop本身不在我们的讨论范围内，因为并不能实现并发。但是他和我们讨论的多线程有很大的关系。

我们从底层API开始了解，然后再研究高级API。这是因为高层次的API是基于底层API进行封装的。但是，当你根据你的场景选择不同的API的时候，你应该优先考虑高层抽象API：选择顶层抽象API可以让你工作更加的简单容易。

如果您想知道为什么我们如此坚持不懈地推荐高级抽象和非常简单的并发代码, 您应该阅读本文的第二部分，[challenges of concurrent programming](https://www.objc.io/issues/2-concurrency/concurrency-apis-and-pitfalls/#challenges-of-concurrent-programming), 还有这个[Peter Steinberger’s thread safety article](https://www.objc.io/issues/2-concurrency/thread-safe-class-design/)。

## Threads

[Threads](http://en.wikipedia.org/wiki/Thread_%28computing%29)是进程的子单元，可由操作系统调度程序独立调度。实际上所有的并发API都是基于底层的--对于GCD和Operation Queue都成立。

多线程可以同一时间在单个CPU上执行，或者被认为是同一时间。操作系统会给每个线程分配一些时间，所以从用户角度来看貌似所有的任务都是同事进行的。如果有多个CPU可以使用，此时多线程才是真正的多线程，从而减少特定工作负载所需的总时间。

你可以使用[CPU strategy view](http://developer.apple.com/library/mac/#documentation/DeveloperTools/Conceptual/InstrumentsUserGuide/AnalysingCPUUsageinYourOSXApp/AnalysingCPUUsageinYourOSXApp.html)来探究你的代码或者framework在多个CPU上是如何被调度的。

有个重要的问题就是你没有办法控制你的代码何时何地进行执行，并且也无法预知什么时候会因为其他任务的执行而被终止。









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


