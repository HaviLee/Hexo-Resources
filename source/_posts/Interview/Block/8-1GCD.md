---
title: GCD
keywords: iOS面试
date: 2019-04-30 01:47:40
categories: 
  - 面试
tags:
  - 多线程
comments: true
---

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/8-1-0.png)

# GCD

- 涉及同步/异步和串行/并发
- dispatch_barrier_async:异步栅栏调用
- dispatch_group

# 同步/异步和串行/并发

- `dispatch_sync(serial_queue, ^{//任务})`
- `dispatch_async(serial_queue, ^{//任务})`
- `dispatch_sync(concurrent_queue, ^{//任务})`
- `dispatch_async(concurrent_queue, ^{//任务})`

## 同步串行

#### 造成死锁

```c++
- (void)viewDidLoad 
{
	dispatch_sync(dispatch_get_main_queue(), ^{
		[self doSomething];
	});
}
```

> 结果会造成`死锁`。

`死锁是队列引起的循环等待`

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/8-1-1.png)

> `解释：`这里我们以黑色框作为viewDidLoad任务，红色作为Block的任务，首先是viewDidLoad会被添加到主队列中，然后添加Block任务，这两个任务最终都要分配到主线程中处理，viewDidLoad分配到主线程中执行，而它的执行过程中需要调用Block，只有当block同步调用结束后，viewDidLoad才可以继续执行，所以viewDidLoad任务是否结束需要Block任务执行完成；Block任务要想执行，根据主队列的一个性质，先进先出，需要等到ViewDidLoad处理完成，因此这两个任务产生了相互等待。

#### 变形：

```c++
- (void)viewDidLoad 
{
	dispatch_sync(serialQueue, ^{
		[self doSomething];
	});
}
//这里任务添加到了自定义的队列中。
//没有问题
```

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/8-1-2.png)

> 解释：这里不同的是将block分配到了一个串行队列中，这里ViewDidLoad在主队列执行，执行到任务中需要执行block任务，而block任务被分配到了另外的串行队列中，因此这个串行队列中的Block可以执行，等这个串行队列中的block任务执行完了，可以继续执行主队列的viewDidLoad任务。

## 同步并发

```c++
- (void)viewDidLoad 
{
	NSLog(@"1");
	dispatch_sync(global_queue, ^{
		NSLog(@"2");
		dispatch_sync(global_queue, ^{
			NSLog(@"3");
		});
		NSLog(@"4");
	});
	NSLog(@"5");
}

//打印顺序：1.2.3.4.5
```

> 只要是同步提交，都是在当前线程进行执行。
>
> 首先viewDidLoad在主线程的队列中执行，所以先打印1;然后给主线程添加一个并发队列任务，这时候执行2，又往global这个并发队列中添加了任务，因为是并发，所以不会产生相互等待的结果，所以打印3，接着后面的。

## 异步串行

```c++
//或者
- (void)viewDidLoad 
{
	dispatch_async(dispatch_get_main_queue(), ^{
		[self doSomething];
	});
}
```

这个产生的结果和同步串行是一样的，造成死锁

## 异步并发

```c++
- (void)viewDidLoad 
{
	dispatch_async(global_queue, ^{
		NSLog(@"1")；
		[self performSelector:@selector(printLog)
						  withObject:nil
						  afterDelay:0];
		NSLog(@"3");
	});
}

- (void)printLog
{
	NSLog(@"2");
}

//结果：1和3，2是不打印的
```

> 首先是异步分派到队列的，因此GCD底层会开启一个新的现场执行GCD中的任务，这些线程默认是没有开启runloop的，因此这个方法即时延迟0s，performSelector这个方法是依赖runloop的，因此这个方法是失效的。

# dispatch_barrier_async()

**Q：怎样利用GCD实现多读单写？**

**Q：如果要你实现多读单写的模型，你该怎么实现？**

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/8-1-3.png)



问题描述：假如我们在内存中维护一个字典或者DB文件，有多个读者和写者需要操作这个共享文件，我们为了能够实现这个多读单写的目的，就需要考虑多线程对于共享数据的访问问题，首先读者和读者是可以并发的，读者和写者是互斥的；写者和写者也是互斥的，多个写者会导致程序出问题。

## 多读单写实现流程

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/8-1-4.png)

> 有多个读处理并发操作，关于写处理需要和 读处理进行互斥操作，在写处理完成之后，才可以再进行读处理；GCD中的dispatch_barrier_async提供了这样的一个模型，当有读者读的时候，其它的读操作也可以进行，但是不可以进行写处理，没有读的时候才可以进行写处理；当在进行写处理的时候，如果读处理提交，需要等写完成后，再进行读。这个很形象的成为栅栏。

## 多读单写方案

dispatch_barrier_async(concurrent_queue, ^{//写操作})

```objective-c
#import <Foundation/Foundation.h>

@interface UserCenter : NSObject

@end
```

```objective-c
#import "UserCenter.h"

@interface UserCenter()
{
    // 定义一个并发队列
    dispatch_queue_t concurrent_queue;
    
    // 用户数据中心, 可能多个线程需要数据访问
    NSMutableDictionary *userCenterDic;
}

@end

// 多读单写模型
@implementation UserCenter

- (id)init
{
    self = [super init];
    if (self) {
        // 通过宏定义 DISPATCH_QUEUE_CONCURRENT 创建一个并发队列
        concurrent_queue = dispatch_queue_create("read_write_queue", DISPATCH_QUEUE_CONCURRENT);
        // 创建数据容器
        userCenterDic = [NSMutableDictionary dictionary];
    }
    
    return self;
}

- (id)objectForKey:(NSString *)key
{
    __block id obj;
    // 同步读取指定数据
  	//同步到并发队列中，因此可以允许多个线程中读取，A和B线程同时读取
  	//同时是一个同步调用，可以立即获取值。
    dispatch_sync(concurrent_queue, ^{
        obj = [userCenterDic objectForKey:key];
    });
    
    return obj;
}
//认为是一个写者
- (void)setObject:(id)obj forKey:(NSString *)key
{
    // 异步栅栏调用设置数据
    dispatch_barrier_async(concurrent_queue, ^{
        [userCenterDic setObject:obj forKey:key];
    });
}

@end
```

```c++
dispatch_queue_t concurrentQueue1= dispatch_queue_create("www.goole.com", DISPATCH_QUEUE_CONCURRENT);
    
    /********************模拟读取操作********************/
     dispatch_async(concurrentQueue1, ^{
         for (int i=0; i<1000; i++) {
             int index=arc4random()%12;
             NSLog(@"读取任务-%@",self.array[index]);
         }
     });
     dispatch_async(concurrentQueue1, ^{
         for (int i=0; i<1000; i++) {
             int index=arc4random()%12;
             NSLog(@"读取任务二%@",self.array[index]);
         }
     });
     dispatch_async(concurrentQueue1, ^{
         for (int i=0; i<1000; i++) {
             int index=arc4random()%12;
             NSLog(@"读取任务三%@",self.array[index]);
         }
    });
    
    
    /********************模拟写入操作********************/
    dispatch_barrier_async(concurrentQueue1, ^{
        [self.array removeAllObjects];
        [self.array addObjectsFromArray:@[@"aaa",@"bbb",@"ccc",@"ddd",@"eee",@"qqq",@"ccc",@"fff",@"ggg",@"rrr",@"bbb",@"jjj"]];
    });

    
    
    
    dispatch_async(concurrentQueue1, ^{
        for (int i=0; i<100; i++) {
            int index=arc4random()%12;
            NSLog(@"读取任务四%@",self.array[index]);
        }
    });
    dispatch_async(concurrentQueue1, ^{
        for (int i=0; i<100; i++) {
            int index=arc4random()%12;
            NSLog(@"读取任务五%@",self.array[index]);
        }
    });
    dispatch_async(concurrentQueue1, ^{
        for (int i=0; i<100; i++) {
            int index=arc4random()%12;
            NSLog(@"读取任务六%@",self.array[index]);
        }
    });
    
//结论:打印结果会是 读取任务一二三 执行完之后 执行写入操作,写入操作执行完之后最后执行 读取任务四五六
```



# Dispatch_group_async()

`使用GCD实现这个需求：A、B、C三个任务并发，完成之后执行任务D?`

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/8-1-5.png)

代码实现：

```c++
#import "GroupObject.h"

@interface GroupObject()
{
    dispatch_queue_t concurrent_queue;
    NSMutableArray <NSURL *> *arrayURLs;
}

@end

@implementation GroupObject

- (id)init
{
    self = [super init];
    if (self) {
        // 创建并发队列
        concurrent_queue = dispatch_queue_create("concurrent_queue", DISPATCH_QUEUE_CONCURRENT);
        arrayURLs = [NSMutableArray array];
    }

    return self;
}

- (void)handle
{
    // 创建一个group
    dispatch_group_t group = dispatch_group_create();
    
    // for循环遍历各个元素执行操作
    for (NSURL *url in arrayURLs) {
        
        // 异步组分派到并发队列当中
        dispatch_group_async(group, concurrent_queue, ^{
            
            //根据url去下载图片
            
            NSLog(@"url is %@", url);
        });
    }
    
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        // 当添加到组中的所有任务执行完成之后会调用该Block
        NSLog(@"所有图片已全部下载完成");
    });
}

```

[队列]: https://juejin.im/post/5b28ca5de51d4558e03cc847	"队列"
[栅栏应用]: https://juejin.im/post/5c4a9ed6e51d45599635c38e	"栅栏"

