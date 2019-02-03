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

有个重要的问题就是你没有办法控制你的代码何时何地进行执行，并且也无法预知什么时候会因为其他任务的执行而被终止。这样的线程调度是很强大的技术。但是也带来了巨大的复杂性，我们后面会遇到。

先不考虑线程调度的复杂性，你既可以使用 [POSIX thread](http://en.wikipedia.org/wiki/POSIX_Threads) 底层API，也可以使用OC对他的封装 `NSThread`来创建你的线程。下面是使用`pthread`来从一亿的数据中找到最大值和最小值。它产生了4个并行运行的线程，从这个例子你可以看出为什么不直接使用`pthread`:

```objectivec
#import <pthread.h>

struct threadInfo {
    uint32_t * inputValues;
    size_t count;
};

struct threadResult {
    uint32_t min;
    uint32_t max;
};

void * findMinAndMax(void *arg)
{
    struct threadInfo const * const info = (struct threadInfo *) arg;
    uint32_t min = UINT32_MAX;
    uint32_t max = 0;
    for (size_t i = 0; i < info->count; ++i) {
        uint32_t v = info->inputValues[i];
        min = MIN(min, v);
        max = MAX(max, v);
    }
    free(arg);
    struct threadResult * const result = (struct threadResult *) malloc(sizeof(*result));
    result->min = min;
    result->max = max;
    return result;
}

int main(int argc, const char * argv[])
{
    size_t const count = 1000000;
    uint32_t inputValues[count];

    // Fill input values with random numbers:
    for (size_t i = 0; i < count; ++i) {
        inputValues[i] = arc4random();
    }

    // Spawn 4 threads to find the minimum and maximum:
    size_t const threadCount = 4;
    pthread_t tid[threadCount];
    for (size_t i = 0; i < threadCount; ++i) {
        struct threadInfo * const info = (struct threadInfo *) malloc(sizeof(*info));
        size_t offset = (count / threadCount) * i;
        info->inputValues = inputValues + offset;
        info->count = MIN(count - offset, count / threadCount);
        int err = pthread_create(tid + i, NULL, &findMinAndMax, info);
        NSCAssert(err == 0, @"pthread_create() failed: %d", err);
    }
    // Wait for the threads to exit:
    struct threadResult * results[threadCount];
    for (size_t i = 0; i < threadCount; ++i) {
        int err = pthread_join(tid[i], (void **) &(results[i]));
        NSCAssert(err == 0, @"pthread_join() failed: %d", err);
    }
    // Find the min and max:
    uint32_t min = UINT32_MAX;
    uint32_t max = 0;
    for (size_t i = 0; i < threadCount; ++i) {
        min = MIN(min, results[i]->min);
        max = MAX(max, results[i]->max);
        free(results[i]);
        results[i] = NULL;
    }

    NSLog(@"min = %u", min);
    NSLog(@"max = %u", max);
    return 0;
}
```

`NSThread`是对`pthread`的一个比较简单的封装。这让我们的代码看起来更像cocoa环境。比如，你可以定义自己的一个类作为 `NSThread`的子类，在这个类里面你可以封装你在后台执行的代码。对于上面的例子，我们可以定义下面的子类：

```objective-c
@interface FindMinMaxThread : NSThread
@property (nonatomic) NSUInteger min;
@property (nonatomic) NSUInteger max;
- (instancetype)initWithNumbers:(NSArray *)numbers;
@end

@implementation FindMinMaxThread {
    NSArray *_numbers;
}

- (instancetype)initWithNumbers:(NSArray *)numbers 
{
    self = [super init];
    if (self) {
        _numbers = numbers;
    }
    return self;
}

- (void)main
{
    NSUInteger min;
    NSUInteger max;
    // process the data
    self.min = min;
    self.max = max;
}
@end
```

为了开启一个新的线程，我们需要创建一个新的线程对象并且调用他们的 `start`方法：

```objective-c
NSMutableSet *threads = [NSMutableSet set];
NSUInteger numberCount = self.numbers.count;
NSUInteger threadCount = 4;
for (NSUInteger i = 0; i < threadCount; i++) {
    NSUInteger offset = (numberCount / threadCount) * i;
    NSUInteger count = MIN(numberCount - offset, numberCount / threadCount);
    NSRange range = NSMakeRange(offset, count);
    NSArray *subset = [self.numbers subarrayWithRange:range];
    FindMinMaxThread *thread = [[FindMinMaxThread alloc] initWithNumbers:subset];
    [threads addObject:thread];
    [thread start];
}
```

现在我们可以观测线程的属性 `isFinished` 判断一个线程是不是执行完毕，这样我们可以在计算结果之前判断所有的线程是不是运行完成。我们将这个留给读者自己实现。直接使用线程比如 `pthread`或者 `NSThread`,这是一个相对笨重的体验，而且不能很好的体现我们的面向对象编程。

直接使用线程可能产生一个问题是，**如果你的代码和底层框架都生成了自己的线程，则活跃的线程的数量将会呈指数级增长**。这在大的项目中是一个常见的问题。比如，你创建了8个线程以利用8个CPU内核，并且从这些线程调用的框架代码执行相同的操作，因为它不知道您已经创建了线程，那么就会很快产生几十个线程。每个线程都应该负责自己的代码部分，至少，最后的结果是没有问题的。线程不是免费的资源。每个线程都会浪费内存和内核资源。

下面我们会讨论下两个并发API: `Grand Central Dispatch` 和 `Operation Queue`。这两个API通过我们日常使用的[线程池管理](http://en.wikipedia.org/wiki/Thread_pool_pattern)来解决上面的问题。

## GCD-Grand Central Dispatch

GCD是在OSX 10.6和iOS4引入的新的API,目的是为了帮助开发者更好的利用设备的多个CPU内核。我们会在下面的文章[article about low-level concurrency APIs](https://www.objc.io/issues/2-concurrency/low-level-concurrency-apis/)中详细介绍一些GCD的底层。

使用GCD你不需要在直接和线程打交道。你不需要把你的代码片段放到一个队列中，GCD在底层会帮助我们管理一个[线程池](http://en.wikipedia.org/wiki/Thread_pool_pattern)。GCD决定你的代码片段会在哪个线程上执行，并且会根据可用的系统资源管理这些线程。这可以解决在开发过程中会创建过多的线程问题，因为使用GCD这些线程是由线程池管理。

使用GCD另外一个重要的改变就是：你考虑的是队列中任务而不需要考虑线程中的任务。这种新的处理并发的模型很容易操作。

GCD暴露了5个不同的队列：**{% label danger@在主线程运行的main 队列,具有不同优先级的三个后台队列，和一个具有更低优先级的后台队列—I/O操作 %}**.当然你也可以创建自定义队列，可以是串行队列也可以是并行队列。虽然自定义队列是一个强大的抽象, 但您在它们上安排的所有块最终都会细流到系统的全局队列及其线程池中。

![image](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/thread.png)

对不同的队列使用不同的优先级听起来很前卫。但是，建议不要使用优先级，使用默认的优先级就好了。在具有不同优先级的队列上安排任务可能会很快导致意外行为, 特别这些任务访问共享资源。这可能会导致整个程序停顿, 因为一些低优先级的任务阻碍了高优先级任务的执行。你可以在下面阅读更多关于这种现象的信息, 称为[优先级倒置](https://www.objc.io/issues/2-concurrency/concurrency-apis-and-pitfalls/#priority-inversion)。

尽管GCD是一个底层C 的API,但是很方便使用。这使得人们很容易忘记, 在将代码块调度到 gcd 队列时, 并发编程的所有警告和陷阱仍然适用。建议阅读下面的[challenges of concurrent programming](https://www.objc.io/issues/2-concurrency/concurrency-apis-and-pitfalls/#challenges-of-concurrent-programming)来避免潜在的问题。后面我们会对GCD API做一个详细的演示，这些演示包含了一些有价值的解释。

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


