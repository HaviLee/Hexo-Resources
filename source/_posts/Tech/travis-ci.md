---
title: Travis CI
date: 2017-06-11 13:47:40
categories: [Build-Tools]
tags: [TravisCI]
---

![clang](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/travis.jpeg)

Cocoapods是OSX和iOS应用的一个第三方依赖的管理工具。使用CocoaPods，你可以定义工程的依赖，称之为`pods`,你可以通过CocoaPods轻松的管理依赖的版本和开发环境。在这篇文章中，我们将会分析`pod install`的过程以及CocoaPods背后做了哪些工作。
CocoaPods背后的哲学有两层：首先，在您的项目中包含第三方代码涉及许多坑。对于初级开发者，`project`文件是令人生畏的。手动的配置`build phase`&`linker flag`很容易出现错误。CocoaPods简化了这个过程，帮助你自动配置编译设定。
<!--more-->