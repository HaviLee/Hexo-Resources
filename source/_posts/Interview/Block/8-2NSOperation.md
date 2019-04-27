---
title: NSOperation&NSThread
keywords: iOS面试
date: 2019-04-30 02:47:40
categories: 
  - 面试
tags:
  - 多线程
comments: true
---

# NSOperation

需要结合NSOperationQueue配合使用来实现多线程方案。

`使用NSOperation这种技术方案有什么优点？`

- 添加任务依赖
- 任务执行状态的控制
- 最大并发量：maxConcurrent = 1,相当于串行队列

## 任务执行状态

`NSOperation有哪些可控制状态？`

- isReady：当前任务就绪状态
- isExecuting：当前任务是否正在执行
- isFinished：任务是否完成
- isCancelled：是否取消

## 状态控制

- 任务状态控制主要看我们是否重写了 `start` 和 `main`方法。
- 如果只重写了`main`方法，底层控制变更任务完成状态，以及任务退出;包括后续线程退出。这里所有的状态是由系统控制。
- 如果重写了`start`方法，自行控制任务状态。

```objective-c

- (void) start
{
  NSAutoreleasePool	*pool = [NSAutoreleasePool new];
  double		prio = [NSThread  threadPriority];

  AUTORELEASE(RETAIN(self));	// Make sure we exist while running.
  [internal->lock lock];
  NS_DURING
    {
      if (YES == [self isExecuting])
	{
	  [NSException raise: NSInvalidArgumentException
		      format: @"[%@-%@] called on executing operation",
	    NSStringFromClass([self class]), NSStringFromSelector(_cmd)];
	}
      if (YES == [self isFinished])
	{
	  [NSException raise: NSInvalidArgumentException
		      format: @"[%@-%@] called on finished operation",
	    NSStringFromClass([self class]), NSStringFromSelector(_cmd)];
	}
      if (NO == [self isReady])
	{
	  [NSException raise: NSInvalidArgumentException
		      format: @"[%@-%@] called on operation which is not ready",
	    NSStringFromClass([self class]), NSStringFromSelector(_cmd)];
	}
      if (NO == internal->executing)
	{
	  [self willChangeValueForKey: @"isExecuting"];
	  internal->executing = YES;
	  [self didChangeValueForKey: @"isExecuting"];
	}
    }
  NS_HANDLER
    {
      [internal->lock unlock];
      [localException raise];
    }
  NS_ENDHANDLER
  [internal->lock unlock];

  NS_DURING
    {
      if (NO == [self isCancelled])
	{
	  [NSThread setThreadPriority: internal->threadPriority];
     //调用了main函数
	  [self main];
	}
    }
  NS_HANDLER
    {
      [NSThread setThreadPriority:  prio];
      [localException raise];
    }
  NS_ENDHANDLER;

  [self _finish];
  [pool release];
}
```

```c++
- (void) _finish
{
  /* retain while finishing so that we don't get deallocated when our
   * queue removes and releases us.
   */
  [self retain];
  [internal->lock lock];
  if (NO == internal->finished)
    {
      if (YES == internal->executing)
        {
	  [self willChangeValueForKey: @"isExecuting"];
	  [self willChangeValueForKey: @"isFinished"];
	  internal->executing = NO;
	  internal->finished = YES;
	  [self didChangeValueForKey: @"isFinished"];
	  [self didChangeValueForKey: @"isExecuting"];
	}
      else
	{
	  [self willChangeValueForKey: @"isFinished"];
	  internal->finished = YES;
	  [self didChangeValueForKey: @"isFinished"];
	}
      if (NULL != internal->completionBlock)
	{
	  CALL_BLOCK_NO_ARGS(internal->completionBlock);
	}
    }
  [internal->lock unlock];
  [self release];
}
```

从上面的代码可以看出，只重写main函数，系统会帮助我们处理不同的状态。

`系统是如何移除一个isFinished = YES的NSOperation的？`

答案：通过KVO;



# NSThread

通常结合runloop考察。

## 启动流程

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/8-2-1.png)

> 首先创建一个NSThred线程，会调用它的start方法启动线程，在start内部会创建一个pthred线程，我们指定pthread的启动函数，在启动函数内部会调用NSThread定义的main函数，在main函数中会通过target performSelector形式执行在NSthread定义的执行体，之后关闭exit.

## Start 源码解读

面试题：

1. 结合runloop实现常驻线程？
2. NSThread内部实现机制？

```c++
- (void) start
{
  pthread_attr_t	attr;

  if (_active == YES)
    {
      [NSException raise: NSInternalInconsistencyException
                  format: @"[%@-%@] called on active thread",
        NSStringFromClass([self class]),
        NSStringFromSelector(_cmd)];
    }
  if (_cancelled == YES)
    {
      [NSException raise: NSInternalInconsistencyException
                  format: @"[%@-%@] called on cancelled thread",
        NSStringFromClass([self class]),
        NSStringFromSelector(_cmd)];
    }
  if (_finished == YES)
    {
      [NSException raise: NSInternalInconsistencyException
                  format: @"[%@-%@] called on finished thread",
        NSStringFromClass([self class]),
        NSStringFromSelector(_cmd)];
    }

  /* Make sure the notification is posted BEFORE the new thread starts.
   */
  gnustep_base_thread_callback();

  /* The thread must persist until it finishes executing.
   */
  RETAIN(self);

  /* Mark the thread as active while it's running.
   */
  _active = YES;

  errno = 0;
  pthread_attr_init(&attr);
  /* Create this thread detached, because we never use the return state from
   * threads.
   */
  pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
  /* Set the stack size when the thread is created.  Unlike the old setrlimit
   * code, this actually works.
   */
  if (_stackSize > 0)
    {
      pthread_attr_setstacksize(&attr, _stackSize);
    }
   //这里通过pthread创建一个线程
  if (pthread_create(&pthreadID, &attr, nsthreadLauncher, self))
    {
      DESTROY(self);
      [NSException raise: NSInternalInconsistencyException
                  format: @"Unable to detach thread (last error %@)",
                  [NSError _last]];
    }
}

```

启动线程函数：nsthreadLauncher

```c++
/**
 * Trampoline function called to launch the thread
 启动线程
 */
static void *
nsthreadLauncher(void *thread)
{
  //首先获取当前线程
  NSThread *t = (NSThread*)thread;

  setThreadForCurrentThread(t);

  /*
   * Let observers know a new thread is starting.
   */
  if (nc == nil)
    {
      nc = RETAIN([NSNotificationCenter defaultCenter]);
    }
  //发送通知，告诉观察者线程已经启动
  [nc postNotificationName: NSThreadDidStartNotification
		    object: t
		  userInfo: nil];
	//设置线程名字
  [t _setName: [t name]];
	//如果main函数什么也不做，线程就会退出，如果想保持线程不退出，需要在main中做一个runloop循环
  [t main];

  [NSThread exit];
  // Not reached
  return NULL;
}
```

main函数：

```c++
- (void) main
{
	//先做个异常判断
  if (_active == NO)
    {
      [NSException raise: NSInternalInconsistencyException
                  format: @"[%@-%@] called on inactive thread",
        NSStringFromClass([self class]),
        NSStringFromSelector(_cmd)];
    }
	//这里target和slector是如何传递进来的，是使用detachNewThread穿进来的。
  [_target performSelector: _selector withObject: _arg];
}
```

# 多线程和锁

`iOS中有哪些锁？或者日常开发中你用到哪些锁？`

- @synchronized
- atomic
- OSSpinLock
- NSSRecursiveLock
- NSLock
- dispatch_semaphore_t

## synchronized

一般都是在创建单例对象的时候使用，来保证在多线程环境下创建的对象的唯一性。

```c++
// @synchronized单例
+(instancetype)sharedLoadData
{
    static YYSingleton *singleton;
    @synchronized (self) {
    if (!singleton) {
        singleton = [[YYSingleton alloc] init];
    }
    }
    return singleton;
}

```

> 使用@synchronized虽然一定程度上解决了多线程的问题，但并不完美。因为只有在singleton未创建时，加锁才是必要的。如果singleton已经创建，这个时候还加锁的话，会影响性能。在iOS中，GCD为我们提供方便又高效的方法—dispatch_once;

```c++
 +(instancetype)sharedLoadData
{
    static YYSingleton *singleton = nil;

    static dispatch_once_t onceToken;
    // dispatch_once  无论使用多线程还是单线程，都只执行一次
    dispatch_once(&onceToken, ^{

        singleton = [[YYSingleton alloc] init];
    });
    return singleton;
}

```



## Atomic

- 修饰属性的关键字

- 对被修饰的对象进行原子操作(`不负责使用`)：

  > 比如：@property(atomic) NSMutableArray *array;
  >
  > self.array = [NSMutableArray array];这种赋值操作可以保证线程安全；
  >
  > 但是如果通过[self.array addObject: obj];是不能保证线程安全。

## OSSpinLock自旋锁

- `循环等待询问`，不释放当前资源。
- 用于轻量级数据访问，简单的int值 +1/-1操作

## NSLock

解决细粒度的线程保护问题。

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/8-2-2.png)

> 上面的代码造成死锁，使用NSLock对临近区加锁的时候，在锁里面又进行加锁，会由于重入的原因造成死锁。
>
> 这种场景使用NSRecursiveLock解决。

## NSRecursiveLock

递归锁可以方法重入，递归方法也可以使用

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/8-2-3.png)

## dispatch_semaphore_t

信号量型的锁

**相关函数：**

- dispatch_semaphore_create(1);
- dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER)//
- dispatch_semaphore_signal(semaphore)

****

**dispatch_semaphore_create(1)**

```c++
struct semaphore {
	int value;
	List<thread>;
}
```

> 两个成员变量：value，List：线程队列

```c++
dispatch_semaphore_wait()
{
	S.value = S.value - 1;
	if S.value < 0 then Block(S.List); ===> 阻塞是一个主动行为
}
```

> 先对value-1，如果信号量<0，然后会有个主动阻塞行为

```c++
dispatch_semaphore_singal()
{
	S.value = S.value + 1;
	if S.value <= 0 then wakeup(S.List); ===> 唤醒是个被动行为
}
```

> 先对value+1，如果信号量<=0，标识有队列在排队



# 总结：

1. 使用GCD实现多读单写？

2. iOS系统为我们提供了几种多线程技术，各自的特点是什么？

   A：主要是三种：GCD，NSOperation&NSOperationQueue 和NSThread;GCD用来执行一些简单的线程同步，子线程的分派和多度单写；NSOperation 例如第三方的AFNetworking，可以对任务状态控制，添加移除线程依赖；NSThread常用来实现常驻线程。

3. NSOperation对象在Finished之后是如何从queue中移除的？

   A：在内通过KVO，通知它的queue移除

4. 你用过哪些锁？怎么使用的？

   A：主要NSLock, @synchronized,递归锁，递归操作中使用递归锁。

