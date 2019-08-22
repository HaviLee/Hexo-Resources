---
title: 数据结构之链表
date: 2019-08-05 19:47:40
keywords: 知识小集
description: 主要是日常生活的一些小坑
categories: 
  - Swift
  - Objective-C
tags:
  - iOS
comments: false
---

# Xcode

## Xcode 每次都需要Clean，修改的代码才会起作用？

原因是Xcode10导致的，Xcode新增了编译选项，新建的项目默认使用 **New Build System**;解决办法就是使用 **Legacy Build System**。

当我使用了CocoaPods develop Pods的时候，修改的代码想要生效，需要每次都Clean下项目。

![image](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/08081102.png)



![image](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/08081103.png)

[Apple Developer]: https://developer.apple.com/documentation/xcode_release_notes/xcode_10_release_notes/build_system_release_notes_for_xcode_10



# Swift

## swift 中 self.navigationController = nil

在swift中，如果在 `viewDidLoad`中对VC的导航栏做操作，此时 `self.navigationController = nil`；但是在

viewWillAppear中此时的 `self.navigationController ！= nil`。