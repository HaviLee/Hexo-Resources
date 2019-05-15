---
title: 第三方库
keywords: iOS面试
date: 2019-05-01 11:47:40
categories: 
  - 面试
tags:
  - 第三方库
comments: true
---

# AFNetworking

`整体框架图？`

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/14-1-1.png)

> 底层是基于session，

**主要类关系图**

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/14-1-2.png)

**主要类的功能：**

AFURLSessionManager:

- 创建和管理NSURLSession、NSURLSessionTask
- 实现NSURLSessionDelegate等协议的代理方法
- 引入AFSecurityPolicy保证请求安全
- 引入AFNetworkingReachabilityManager监控网络变化



AFURLConnectionOperation 这个类是基于 NSURLConnection 构建的，其希望能在后台线程接收 Delegate 回调。为此 AFNetworking 单独创建了一个线程，并在这个线程中启动了一个 RunLoop：

+ ```c++
    - (void)networkRequestThreadEntryPoint:(id)__unused object {
        @autoreleasepool {
            [[NSThread currentThread] setName:@"AFNetworking"];
            NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
            [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
            [runLoop run];
    }
  }
    - (NSThread *)networkRequestThread {
        static NSThread *_networkRequestThread = nil;
        static dispatch_once_t oncePredicate;
        dispatch_once(&oncePredicate, ^{
            _networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
            [_networkRequestThread start];
        });
        return _networkRequestThread;
      }
    ```
    
    RunLoop 启动前内部必须要有至少一个 Timer/Observer/Source，所以 AFNetworking 在 [runLoop run] 之前先创建了一个新的 NSMachPort 添加进去了。通常情况下，调用者需要持有这个 NSMachPort (mach_port) 并在外部线程通过这个 port 发送消息到 loop 内；但此处添加 port 只是为了让 RunLoop 不至于退出，并没有用于实际的发送消息。

- ```c++
    -(void)start {
      [self.lock lock];
      if ([self isCancelled]) {
          [self performSelector:@selector(cancelConnection) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
      } else if ([self isReady]) {
          self.state = AFOperationExecutingState;
          [self performSelector:@selector(operationDidStart) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
      }
  [self.lock unlock];
}
```

当需要这个后台线程执行任务时，AFNetworking 通过调用 [NSObject performSelector:onThread:..] 将这个任务扔到了后台线程的 RunLoop 中。
---------------------


# SDWebImage

**架构简图**

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/14-1-3.png)

> 1. UIImageView + WebCache通过setImageUrl可以进行图片下载或者缓存提取
> 2. 支持上面的工作的是SDWebImageManager.
> 3. 底层是SDImageCache只要是图片缓存，一方面是内存缓存，一方面是磁盘缓存
> 4. SDWebImageDownloader:下载类；

**加载图片的流程**

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/14-1-4.png)

# ReactiveCocoa

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/14-1-5.png)

> RAC是一个函数式响应式编程；
>
> 主要涉及的概念是信号&订阅；我们可以订阅一个信号；

## 信号

RAC中的核心类RACSignal;下图是和信号有关的所有类；

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/14-1-6.png)

信号的父类RACStream提供了几个方法：

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/14-1-7.png)

> 1. empty/return/bind/concat/zipWith这些都是抽象方法，不能直接使用
> 2. 在这些函数上层，提供了具体的实现方法：map/take/skip/ignore/filter方法；

**什么是信号？**

信号代表一连串的状态：

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/14-1-8.png)

**信号量介绍**

- RACReturnSignal
- RACDynamicSignal

## 订阅

RACSubscriber订阅流程

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/14-1-9.png)



> 开始订阅一个信号；

# AsyncDisplayKit

主要是为了减轻主线程的压力，把一些任务放到子线程来做；

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/14-1-10.png)



主要解决的问题：

- 解决布局耗时运算：文本宽高、视图布局计算
- 渲染：文本渲染、图片解码、图形绘制
- UIkit 对象：对象创建、调整、销毁

## 基本原理

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/14-1-12.png)

> 正常情况：UIView作为CALayer的delegate,CALayer作为UIView的一个成员变量，负责UI视图的展示工作；
>
> AsyncDisplay在UIView上面封装了一个类ASNode类，它有个成员变量.view可以生成UIView,同时每个UIView有个.node成员变量；
>
> ASNode可以在后台线程使用，

1. **针对ASNode的修改和提交，会对其封装并提交到一个全局容器当中**
2. **ASDK也在Runloop中注册了一个observer**
3. **当RunLoop进入休眠前，ASDK执行loop内提交的所有任务**

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/14-1-13.png)

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/14-1-14.png)

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/14-1-15.png)

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/14-1-16.png)

