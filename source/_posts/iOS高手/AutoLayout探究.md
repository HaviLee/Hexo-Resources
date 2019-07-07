---
title: AutoLayout探究
date: 2019-07-07 13:47:40
categories: [iOS]
tags: [iOS 高手]
---

# Auto Layout是如何自动布局的？





# Auto Layout的来历

Auto Layout是苹果自己开发的布局引擎，背后使用的**布局算法是** [Cassowary](https://overconstrained.io/) 。

> Cassowary 可以解析线性等式系统和线性不等式系统，用来表示用户界面中的那些相等和不等的关系。因此，Cassowary 开发了一中规则系统，通过约束来描述视图间的关系。约束就是规则，这个能够表示出一个视图相对于另一个视图的位置。
>
> Cassowary 算法让视图位置按照一种简单的布局思路来写，这些简单的相对位置描述可以在运行时动态的计算出视图的具体的位置。视图位置写法简化了，界面相关代码也就更容易维护。

# Auto Layout的生命周期

Auto Layout的布局引擎系统称为 **Layout Engine** ,在每个视图得到自己的布局之前，Layout Engine会将视图、约束、优先级、固定大小通过计算转换为最终的大小和位置。在layout Engine里面，每当约束发生变化，就会触发Deffered Layout Pass,完成后进入进入监听约会变化的状态。当再次监听到约束变化，就会进入下一轮中。

![lifeCiycle](https://github.com/HaviLee/Blog-Images/raw/master/高手/Screen%20Shot%202019-07-07%20at%2021.39.21.png)

> 上图中，当Constranints 发生约束变化，添加、删除视图会触发约束变化。Activating或者Deactivating，设置Constraints或者Priority时也会触发约束变化。Layout Engine在约束变化后会重新计算布局，获得布局后调用 **superview.setNeedLayout**.
>
> 接下来进入 **Deferrend Layout Pass**,主要是做容错处理，如果有些视图在更新约束时没有确定或者缺失的话，会在这里进行容错处理。
>
> 下面Layout Engine会从上到下调用 LayoutSubviews()，通过Cassowary算法计算各个子视图的位置，算出后将子视图的 **frame**  从layout engine拷贝出来。
>
> 接下来的，就和手写布局的绘制、渲染一样了。因此使用Auto Layout和手写的区别是多了一步布局的计算过程。

# Auto Layout 性能问题

ios12之前 Auto Layout在计算兄弟视图之间有关系的场景的时候，在遍历视图的时候不断的处理和兄弟视图间的关系，这时会修改重新计算，导致计算呈指数增加。

ios12之后，Auto Layout使用了Cassowary算法的高效修改更新的特性。

[WWDC 2018 AutoLayout](https://developer.apple.com/videos/play/wwdc2018/202/)        [Simplex算法](https://en.wikipedia.org/wiki/Simplex_algorithm)

# Auto Layout的易用性

Auto Layout本身原生写法不易用；Apple提供了VFL(Visual Format Language)这种DSL(Domain Specific Language-领域特定语言)来简化 Auto Layout的写法。

Auto Layout只是一种基础的布局思路。前端之前出现了 FlexBox高级的响应式布局，Apple提供了基于Auto Layout封装出一个类似Felxbox的 **UIStackView**来提高易用性。

UIStackView会在父视图中设置子视图的排列方式，比如Fill,Leading/Center,而不用在每个子视图中设置自己和兄弟视图的关系。



[自定义的DSL](https://github.com/ming1016/STMAssembleView)

