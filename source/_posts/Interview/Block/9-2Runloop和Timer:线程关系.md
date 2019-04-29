---
title: Runloop与Timer/线程关系
keywords: iOS面试
date: 2019-04-30 04:47:40
categories: 
  - 面试
tags:
  - Runloop
comments: true
---

# RunLoop与Timer关系

`滑动TableView我们的定时器还会生效吗？`

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/9-1-12.png)

当滑动tableview时，会将runloop从defaultMode切换到trackingMode.而runloop只能在一种mode下执行。

**解决：**

通过 `CFRunLoopAddTimer(runLoop, timer, commonMode)`中

**CFRunLoopAddTimer add timer代码实现原理:**

```c++
//
void CFRunLoopAddTimer(CFRunLoopRef rl, CFRunLoopTimerRef rlt, CFStringRef modeName) {    
    CHECK_FOR_FORK();
    if (__CFRunLoopIsDeallocating(rl)) return;
    if (!__CFIsValid(rlt) || (NULL != rlt->_runLoop && rlt->_runLoop != rl)) 
      return;
    __CFRunLoopLock(rl);
  	//如果是commonmodes的情况；
    if (modeName == kCFRunLoopCommonModes) {
			CFSetRef set = rl->_commonModes ? CFSetCreateCopy(kCFAllocatorSystemDefault, rl->_commonModes) : NULL;
		if (NULL == rl->_commonModeItems) {
	    rl->_commonModeItems = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
			}
	CFSetAddValue(rl->_commonModeItems, rlt);
	if (NULL != set) {
	    CFTypeRef context[2] = {rl, rlt};
	    /* add new item to all common-modes */
	    CFSetApplyFunction(set, (__CFRunLoopAddItemToCommonModes), (void *)context);
	    CFRelease(set);
	}
    } else {
	CFRunLoopModeRef rlm = __CFRunLoopFindMode(rl, modeName, true);
	if (NULL != rlm) {
            if (NULL == rlm->_timers) {
                CFArrayCallBacks cb = kCFTypeArrayCallBacks;
                cb.equal = NULL;
                rlm->_timers = CFArrayCreateMutable(kCFAllocatorSystemDefault, 0, &cb);
            }
	}
	if (NULL != rlm && !CFSetContainsValue(rlt->_rlModes, rlm->_name)) {
            __CFRunLoopTimerLock(rlt);
            if (NULL == rlt->_runLoop) {
		rlt->_runLoop = rl;
  	    } else if (rl != rlt->_runLoop) {
                __CFRunLoopTimerUnlock(rlt);
	        __CFRunLoopModeUnlock(rlm);
                __CFRunLoopUnlock(rl);
		return;
	    }
  	    CFSetAddValue(rlt->_rlModes, rlm->_name);
            __CFRunLoopTimerUnlock(rlt);
            __CFRunLoopTimerFireTSRLock();
            __CFRepositionTimerInMode(rlm, rlt, false);
            __CFRunLoopTimerFireTSRUnlock();
            if (!_CFExecutableLinkedOnOrAfter(CFSystemVersionLion)) {
                // Normally we don't do this on behalf of clients, but for
                // backwards compatibility due to the change in timer handling...
                if (rl != CFRunLoopGetCurrent()) CFRunLoopWakeUp(rl);
            }
	}
        if (NULL != rlm) {
	    __CFRunLoopModeUnlock(rlm);
	}
    }
    __CFRunLoopUnlock(rl);
}
```

```c++
static void __CFRunLoopAddItemToCommonModes(const void *value, void *ctx) {
    CFStringRef modeName = (CFStringRef)value;//这里是一个具体的mode了
    CFRunLoopRef rl = (CFRunLoopRef)(((CFTypeRef *)ctx)[0]);
    CFTypeRef item = (CFTypeRef)(((CFTypeRef *)ctx)[1]);
    if (CFGetTypeID(item) == __kCFRunLoopSourceTypeID) {
	CFRunLoopAddSource(rl, (CFRunLoopSourceRef)item, modeName);
    } else if (CFGetTypeID(item) == __kCFRunLoopObserverTypeID) {
	CFRunLoopAddObserver(rl, (CFRunLoopObserverRef)item, modeName);
    } else if (CFGetTypeID(item) == __kCFRunLoopTimerTypeID) {
	CFRunLoopAddTimer(rl, (CFRunLoopTimerRef)item, modeName);
    }
}
```



# RunLoop和多线程

`线程和runloop是怎样的关系？`

- 线程和runloop是一一对应的。
- 自己创建的线程默认是没有开启runloop的

`怎样实现一个常驻线程？`

- 未当前线程开启一个Runloop
- 向该runLoop中添加一个Port/Source等维持Runloop的事件的循环。
- 启动该runLoop

**代码实现：**

```c++
#import "MCObject.h"

@implementation MCObject

static NSThread *thread = nil;
// 标记是否要继续事件循环
static BOOL runAlways = YES;

+ (NSThread *)threadForDispatch{
    if (thread == nil) {
      //采用安全的方式创建线程
        @synchronized(self) {
            if (thread == nil) {
                // 线程的创建
                thread = [[NSThread alloc] initWithTarget:self selector:@selector(runRequest) object:nil];
                [thread setName:@"com.imooc.thread"];
                //启动
                [thread start];
            }
        }
    }
    return thread;
}
//线程入口
+ (void)runRequest
{
    // 创建一个Source
    CFRunLoopSourceContext context = {0, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL};
    CFRunLoopSourceRef source = CFRunLoopSourceCreate(kCFAllocatorDefault, 0, &context);
    
    // 创建RunLoop，同时向RunLoop的DefaultMode下面添加Source
    CFRunLoopAddSource(CFRunLoopGetCurrent(), source, kCFRunLoopDefaultMode);
    
    // 如果可以运行
    while (runAlways) {
    //达到循环运行一圈的时候，进行资源释放
        @autoreleasepool {
            // 令当前RunLoop运行在DefaultMode下面
            CFRunLoopRunInMode(kCFRunLoopDefaultMode, 1.0e10, true);
        }
    }
    
    // 某一时机 静态变量runAlways = NO时 可以保证跳出RunLoop，线程退出
    CFRunLoopRemoveSource(CFRunLoopGetCurrent(), source, kCFRunLoopDefaultMode);
    CFRelease(source);
}

@end
```

# Runloop面试总结

1. 什么是RunLoop,它是如何做到有事做事，没事休息？

2. runloop与线程的关系？

3. 实现一个常驻线程。

4. 怎样保证子线程数据回来刷新UI的时候不打断用户的滑动操作？

   滑动模式是tracking mode，需要我们把子线程的数据在主线程的default mode中。