---
title: APP架构如何设计
date: 2019-07-08 13:47:40
categories: [iOS]
tags: [iOS 高手]
---

# 大项目、多人、多团队架构思考

## 模块粒度划分参考

- 模块划分必须遵循一定的原则

  划分原则必须规范、清晰，否则会导致代码严重耦合，并加大架构重构的难度。

- 搞清楚模块粒度划分的标准

  - 单一功能原则：对象功能要单一，不要在一个对象里面添加过多功能。
  - 开闭原则：扩展是开放的，修改是关闭的
  - 里氏替换原则：子类对象可以替换基类对象
  - 接口隔离原则：接口的用途要单一，不要在一个接口上根据不同的入参实现多个功能
  - 依赖反转原则：方法依赖抽象，不要依赖实例。iOS就是高层业务依赖于协议。

- 选择合适的粒度。过大过小都不合适。

  - 组件可以任务是可组装的，独立的业务单元，具有高内聚，低耦合特性，因此组件是一种比较合适的模块划分的粒度。
  - **iOS的组件，应该是包含UI控件、相关多个小功能的合集，是一种粒度适中的模块。**
  - 可以先按照物理划分，也就是将多个相同功能的类移动到同一个文件夹下，然后使用cocoapods管理。

# 组件如何分层

组件解耦不是要求每个组件间都没有耦合，组件间需要有上线关系。只有组件间的上下关系分清楚了，才会容易维护和管理。一般思路是：

![组件分层](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/12-2-12.png)

- **独立与App的通用层**：客户端最底层，可以有时长统计、crash统计、网络第三方库、图片缓存、JSON处理，可以被任何App使用；
- **通用的业务层**：针对当下公司的一些通用基础组件，自定义控件、imageView封装；各个业务线对这个都有需求；
- **中间层**：主要是协调和解耦的作用；
- **业务层**：中间层就是为了实现业务A/B/C的解耦；

注意：不要过分的把所有的功能都做成组件，只有那些被多个业务或者团队使用的模块才做成组件，否则维护成本也是一个问题。

# 多团队之间的分工

如果公司有了多个产品线后，团队划分也是一个很重要的考量：

- 需要一个专门的基建团队，负责业务无关的基础功能组件和业务通用组件的开发
- 每个业务由一个专门的团队负责，业务可以按照功能耦合度来划分，耦合度搞的可以划分成单独的业务团队。
- 基建团队和业务团队轮岗。

# 组件化方案的标准

组件化是没有歧义的，关键在于 **组件间关系协调没有固定的标准，因此中间层的优劣称为了衡量一个架构优劣的好坏**；实际的工作中中间层一般分为 **协议式和中间者两种架构设计方案**。

## 协议式架构

- 协议式架构主要采用是协议式编程的思路。

  在编译层面使用协议定义规范，实现可以在不同的地方，从而达到分布管理和维护组件的方式。

- 缺点：

  - 由于协议编程缺少统一调度层，导致难以集中管理；特别是当项目变大、团队变多的情况下，架构管控就会越来越重要
  - 协议式编程接口定义过于规范，从而使得架构的灵活性不够高。

  

## 中间者架构

它采用中间者统一管理的方式，来控制App的整个生命周期中组件的调用关系。

![组件](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/Screen%20Shot%202019-07-08%20at%2022.15.53.png)

示例： [CTMediator](https://github.com/casatwy/CTMediator) 使用的是运行时解耦；下面是解耦的核心方法：

```objective-c
- (id)performTarget:(NSString *)targetName action:(NSString *)actionName params:(NSDictionary *)params shouldCacheTarget:(BOOL)shouldCacheTarget
{
    NSString *swiftModuleName = params[kCTMediatorParamsKeySwiftTargetModuleName];
    
    // generate target
    NSString *targetClassString = nil;
    if (swiftModuleName.length > 0) {
        targetClassString = [NSString stringWithFormat:@"%@.Target_%@", swiftModuleName, targetName];
    } else {
        targetClassString = [NSString stringWithFormat:@"Target_%@", targetName];
    }
    NSObject *target = self.cachedTarget[targetClassString];
    if (target == nil) {
        Class targetClass = NSClassFromString(targetClassString);
        target = [[targetClass alloc] init];
    }


    // generate action
    NSString *actionString = [NSString stringWithFormat:@"Action_%@:", actionName];
    SEL action = NSSelectorFromString(actionString);
    
    if (target == nil) {
        // 这里是处理无响应请求的地方之一，这个demo做得比较简单，如果没有可以响应的target，就直接return了。实际开发过程中是可以事先给一个固定的target专门用于在这个时候顶上，然后处理这种请求的
        [self NoTargetActionResponseWithTargetString:targetClassString selectorString:actionString originParams:params];
        return nil;
    }
    
    if (shouldCacheTarget) {
        self.cachedTarget[targetClassString] = target;
    }


    if ([target respondsToSelector:action]) {
        return [self safePerformAction:action target:target params:params];
    } else {
        // 这里是处理无响应请求的地方，如果无响应，则尝试调用对应target的notFound方法统一处理
        SEL action = NSSelectorFromString(@"notFound:");
        if ([target respondsToSelector:action]) {
            return [self safePerformAction:action target:target params:params];
        } else {
            // 这里也是处理无响应请求的地方，在notFound都没有的时候，这个demo是直接return了。实际开发过程中，可以用前面提到的固定的target顶上的。
            [self NoTargetActionResponseWithTargetString:targetClassString selectorString:actionString originParams:params];
            [self.cachedTarget removeObjectForKey:targetClassString];
            return nil;
        }
    }
```

performTarget:action:params:shouldCacheTarget:方法主要是对 targetName 和 actionName 进行容错处理，也就是对调用方法无响应的处理。这个方法封装了 safePerformAction:target:params 方法，入参 targetName 就是调用接口的对象，actionName 是调用的方法名，params 是参数。

从代码中同时还能看出只有满足 Target_ 前缀的对象和 Action 的方法才能被 CTMediator 使用。这时，我们可以看出中间者架构的优势，也就是利于统一管理，可以轻松管控制定的规则。

代码示例：

```objective-c
[self performTarget:kCTMediatorTargetA
                 action:kCTMediatorActionShowAlert
                 params:paramsToSend
      shouldCacheTarget:NO];
```

存在的问题：

1. 直接硬编码的调用方式，参数是以String的形式保存在内存中的，编码效率降低。
2. 由于只有在运行时才可以调用方法，在调用时缺少类型检查。

## 扩展CTMediator

[范例](https://github.com/ming1016/ArchitectureDemo)



[软件架构入门](http://www.ruanyifeng.com/blog/2016/09/software-architecture.html)

[Swift CTMediator](https://github.com/ModulizationDemo/SwfitDemo)

[[iOS应用架构谈 开篇](https://casatwy.com/iosying-yong-jia-gou-tan-kai-pian.html)]

