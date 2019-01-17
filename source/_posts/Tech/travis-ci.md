---
title: Travis CI
date: 2017-06-11 13:47:40
categories: [Build-Tools]
tags: [TravisCI]
---

![clang](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/travis.jpeg)

你有没有尝试过自己搭建 [CI](http://en.wikipedia.org/wiki/Continuous_integration)服务器？以个人观点来说，确实有点复杂。首先你需要有一台Mac,然后需要配置各种环境。你还需要管理你的账号和保证安全。你还要有获取远程仓库代码的权限，然后配置构建步骤和证书。在构建过程中，你还需要保证server的状态。
<!--more-->

最后你还需要花费大量的时间维护server.但是如果你的源代码是放在`github`上的，现在有一个方式：[Travis CI](https://travis-ci.org/).这个server可以提供可持续集成，这意味着它可以免去你大部分的配置。在Ruby世界中，Travis CI已经众所周知。自2013年4月起，该软件也支持iOS和Mac。

在这篇文章中，我们将展示如何在travis上配置你的项目。这包括构建和运行单元测试，还有把你的产品分发到设备。你可以查看 [Demo](https://github.com/objcio/issue-6-travis-ci).

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">Github集成</h1>

我喜欢`travis`的一个原因是它完美的和github进行融合。其中一个例子就是`PR`.Travis会自动编译每一个`PR`.下面是Travis编译成功的界面：
![success](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/github_ready_to_merge-4a9e8dc9.jpg)
下面是Travis build失败的结果：
![fail](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/github_merge_with_caution-bed39093.jpg)

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">Travis和Github绑定</h1>

我们首先来看下如何在Travis上面绑定你的github.使用你的github账号登录[Travis CI](https://travis-ci.org/),对于私有的repo，是需要付费的。
一旦登录成功，你就可以在travis上开启你需要进行CI的项目。在你的profile界面你可以看到你github下面的所有的repo。
{% note warning %}
如果你在github上创建了一个新的项目，这时候需要你点击`synd`按钮来同步你的信息。
{% endnote %}

![image](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/travis-repos.png)

通过上面的开发打开项目的集成功能，你可以在你的github项目上看到`travis hook`.下一步就是告诉Travis在收到pr之后应该做什么。
![image](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/webhook.png)

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">项目基本配置</h1>

`TravisCI`需要知道你项目的一些基本配置。首先在你的项目根目录下创建一个文件`.travis.yml`:
```ruby
language: objective-c
```

由于Travis运行在虚拟环境上，这个环境有预先[配置](https://docs.travis-ci.com/user/reference/osx/)了[Ruby](https://www.ruby-lang.org/en/),[Homebrew](http://brew.sh/),[CocoaPods](http://cocoapods.org/)，还有一些[default build scripts](https://github.com/jspahrsummers/objc-build-scripts). 上面一行代码可以保证编译你的项目。

预安装的一些脚本可以分析我们的Xcode 项目然后编译每一个target。如果编译成功且单元测试通过则认为构建成功。给你的github 推一下更新来观察是不是构建成功。

尽管开头很简单，但是这个并不能满足你项目的需求。对于如何配置默认编译行为并没有过多的文档说明，比如遇到 [code signing issues](https://github.com/travis-ci/travis-ci/issues/1322) 问题是因为并没有使用``iphonesimulator`SDK`。如果基本的设定不能满足你，我们可以做出更多的自定义脚本。

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">自定义构建命令</h1>





