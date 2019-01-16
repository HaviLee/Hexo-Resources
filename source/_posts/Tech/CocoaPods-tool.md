---
title: CocoaPods 探究
date: 2017-05-11 13:47:40
categories: [Tech]
tags: [CocoaPods,Compile]
---

![clang](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/cocoapods.jpeg)

Cocoapods是OSX和iOS应用的一个第三方依赖的管理工具。使用CocoaPods，你可以定义工程的依赖，称之为`pods`,你可以通过CocoaPods轻松的管理依赖的版本和开发环境。在这篇文章中，我们将会分析`pod install`的过程以及CocoaPods背后做了哪些工作。
CocoaPods背后的哲学有两层：首先，在您的项目中包含第三方代码涉及许多坑。对于初级开发者，`project`文件是令人生畏的。手动的配置`build phase`&`linker flag`很容易出现错误。CocoaPods简化了这个过程，帮助你自动配置编译设定。
<!--more-->
第二：CocoaPods让我们很容易找到一些第三方库。现在这意味着你不需要去构建一个app，它的每个部分都是有其他人负责并组装到一起。这意味着你可以找到一些比较好的库来减少开发时间和提高你的软件质量。

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">CocoaPods核心组成</h1>
C


<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">运行`pod install`</h1>


<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">Pod install 结果</h1>


<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">持续集成</h1>


<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">总结</h1>


