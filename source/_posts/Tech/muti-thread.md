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

现在我们可以观测线程的属性 `isFinished` 判断一个线程是不是执行完毕，这样我们可以在计算结果之前判断所有的线程是不是运行完成。



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


