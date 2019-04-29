---
title: Runloop本质
keywords: iOS面试
date: 2019-04-30 03:47:40
categories: 
  - 面试
tags:
  - Runloop
comments: true
---

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/9-1-1.png)

`什么是RunLoop?`

A：Runloop是通过内部维护的`事件循环`来对`事件/消息进行管理`的一个对象。

`事件循环是什么？`

A：1.在没有消息需要处理的时候，休眠以避免资源占用。2.有消息需要处理时候，可以立刻唤醒。

# Event Loop

没有消息需要处理的时候，休眠以避免资源占用:

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/9-1-2.png)

> 从用户态由系统切换为内核态，就是没有任务的时候，我们的进程或者线程进行休眠。

有消息需要处理时候，可以立刻唤醒：

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/9-1-3.png)

用户态：一般是我们用户层面的调用，API，当我们进行底层API的时候，计算机的状态变化，主要是可以合理的安排资源调用。

### 事件循环的机制

维护的事件循环可以不断的处理消息或者事件，对他们进行管理，同时对于没有消息处理的时候，会从用户态到内核态的一个切换，休眠以避免资源占用；同时有消息处理的时候，会从内核态到用户态的一个切换；当前的用户线程会被唤醒。

# 什么是RunLoop?

`我们的App的main函数为什么保持不退出？`

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/9-1-4.png)

> 解释：在main函数中，会调用uiapplecation函数，在这个函数内部会启动一个运行循环，也就是runloop.他会不断的接收消息，接收消息后，会进行处理，这个runloop循环不是一个简单的for或者while循环；等待不是死循环。

标准答案：在main函数内部，有调用UIApplicationMain函数，在这个函数内部会启动主线程的runloop,runloop是对事件循环的维护机制，可以做到有事的时候做事，没有事情的做的时候，可以做到从用户态到内核态的切换，避免资源的占用，当前线程处于休眠状态。

# RunLoop数据结构

- NSRunLoop是对CFRunLoop的封装，提供了面向对象的API
- NSRunLoop处于Foundation,CFRunLoop位于CoreFoundation.

主要数据结构：

1. CFRunLoop
2. CFRunLoopMode
3. Source/Timer/Observer

## CFRunLoop

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/9-1-5.png)



> CFRunloop主要包括：pthread，currentMode:当前所处的mode; modes; commonModes

❓这里为什么commonModes是一个包含String的Set?

## CFRunLoopMode

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/9-1-6.png)



## Source/Timer/Observer

### CFRunLoopSource

`source0和source1的区别：`

- source0：需要手动唤醒线程
- source1：具备唤醒线程的能力

### CFRunLoopTimer

基于事件的定时器；这个和NSTimer是toll-free bridged的。

### CFRunLoopObserver

观测时间点：

`我们可以观察runloop哪些时间点？`

1. kCFRunLoopEntry: runloop的一个入口时机
2. kCFRunLoopBeforeTimers: 将做对timer做一些处理
3. kCFRunLoopBeforeSources：将要处理source
4. kCFRunLoopBeforeWaiting：这里即将要从用户态到内核态
5. kCFRunLoopAfterWaiting: 从内核态切到用户态不久后
6. kCFRunLoopExit:

## 各个数据结构的关系：一对多

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/9-1-7.png)



### RunLoop 的Mode

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/9-1-8.png)

❓<u>**如果一个Timer想要添加到多个mode怎么办？**</u>

A：这就是CommonModes的存在.

> 一个runloop可以对应多个mode,一个mode又可以有多个source、timer、observers；当runloop运行在某一个mode上的时候，加入另一个mode的timer触发了回调，这个时候我们没有办法接收到。我们只能处理mode1中的timer/source/observer;多个mode起到了隔离的作用。

### CommonMode的特殊性

- CommonMode`不是实际存在`的一种mode
- 是`同步source/Timer/Observer到多个mode`中的一种技术方案。

# Runloop事件循环机制

## 实现机制

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/9-1-10.png)

> 在runloop启动之后，会首先发送一个通知，告知观察者runloop即将启动，之后runloop发送即将处理source0的通知，后面处理source0；如果有source1要处理，会跳转处理唤醒时收到的消息；如果没有source1，线程就会休会，同时发送即将休眠通知；后面线程正式休眠，等待唤醒。唤醒条件：1.Source1事件2.Timer事件3.外部手动唤醒。唤醒后又到处理Timer/source0事件。

❓`当一个处于休眠的Runloop可以通过哪些方式唤醒？`

Q：app从点击程序启动到运行退出系统都发生了什么？？

A：我们调用了main函数之后，有调用UIApplicationMain函数，在这个函数内部会启动主线程的runloop,经过一系列的处理之后，我么的runloop处于休眠状态，此时比如点击屏幕，会产生一个machport，machport会转化为source1,可以唤醒我们主线程，当我们app杀死的时候，会退出runloop，runloop退出后，线程也就结束了。



## Runloop的核心

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/9-1-11.png)

用户态向核心态的转化：main函数启动后，经过处理后，内部会调用mach_msg()函数，这个函数会引起系统调用，经过系统调用，当前用户线程就把控制权交给核心态，mach_msg()会在在一定条件下会返回触发方；这个条件就是唤醒线程的条件。

