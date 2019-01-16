---
title: CocoaPods 探究
date: 2017-05-11 13:47:40
categories: [Build-Tools]
tags: [CocoaPods]
---

![clang](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/cocoapods.jpeg)

Cocoapods是OSX和iOS应用的一个第三方依赖的管理工具。使用CocoaPods，你可以定义工程的依赖，称之为`pods`,你可以通过CocoaPods轻松的管理依赖的版本和开发环境。在这篇文章中，我们将会分析`pod install`的过程以及CocoaPods背后做了哪些工作。
CocoaPods背后的哲学有两层：首先，在您的项目中包含第三方代码涉及许多坑。对于初级开发者，`project`文件是令人生畏的。手动的配置`build phase`&`linker flag`很容易出现错误。CocoaPods简化了这个过程，帮助你自动配置编译设定。
<!--more-->
第二：CocoaPods让我们很容易找到一些第三方库。现在这意味着你不需要去构建一个app，它的每个部分都是有其他人负责并组装到一起。这意味着你可以找到一些比较好的库来减少开发时间和提高你的软件质量。

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px"> CocoaPods核心组成</h1>

CocoaPods是由Ruby写的并且由一些Ruby Gem构成。它核心的几个gem是：[CocoaPods/CocoaPods](https://github.com/CocoaPods/CocoaPods/),[CocoaPods/Core](https://github.com/CocoaPods/Core), and[CocoaPods/Xcodeproj](https://github.com/CocoaPods/Xcodeproj)。(CocoaPods是由另外一个包管理工具开发的依赖管理工具)。

## CocoaPods/CocoaPod

这是面向用户的组件, 每当您调用 pod 命令时都会被激活。它包括实际使用 cocoapod 所需的所有功能, 并利用所有它来执行其他gem任务。

## CocoaPods/Core

Core主要是处理和CocoaPods有关的文件，主要是`Podfile`&`podspecs`。

### Podfile

`Podfile`里面定义了项目的依赖。这个文件是可自定义的，你可以从[Podfile guide](http://guides.cocoapods.org/syntax/podfile.html) 中查看。

### Podspec

`.podsepec`文件指定了特定的pod是如何添加到项目中。它支持包括：`listing source files`/`frameworks`/`compiler flags`还有一些被依赖的库。这个是设置pod的一些参数。



## CocoaPods/Xcodeproj

这个`gem`会处理所有文件的交互。它有能力创建和修改`.xcodeproj`&`.xcworkspace`文件。它同时作为一个独立的gem也是很容易的，如果你想写一个脚本来修改project文件，这个`gem`很适合你。

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">运行Pod install</h1>

当你调用`pod install`后面有很多任务执行。你可以使用参数`--verbose`可以打印出这个命令运行的详细信息。比如：`pod install --verbose`:

```json
$ pod install --verbose

Analyzing dependencies

Updating spec repositories
Updating spec repo `master`
  $ /usr/bin/git pull
  Already up-to-date.


Finding Podfile changes
  - AFNetworking
  - HockeySDK

Resolving dependencies of `Podfile`
Resolving dependencies for target `Pods' (iOS 6.0)
  - AFNetworking (= 1.2.1)
  - SDWebImage (= 3.2)
    - SDWebImage/Core

Comparing resolved specification to the sandbox manifest
  - AFNetworking
  - HockeySDK

Downloading dependencies

-> Using AFNetworking (1.2.1)

-> Using HockeySDK (3.0.0)
  - Running pre install hooks
    - HockeySDK

Generating Pods project
  - Creating Pods project
  - Adding source files to Pods project
  - Adding frameworks to Pods project
  - Adding libraries to Pods project
  - Adding resources to Pods project
  - Linking headers
  - Installing libraries
    - Installing target `Pods-AFNetworking` iOS 6.0
      - Adding Build files
      - Adding resource bundles to Pods project
      - Generating public xcconfig file at `Pods/Pods-AFNetworking.xcconfig`
      - Generating private xcconfig file at `Pods/Pods-AFNetworking-Private.xcconfig`
      - Generating prefix header at `Pods/Pods-AFNetworking-prefix.pch`
      - Generating dummy source file at `Pods/Pods-AFNetworking-dummy.m`
    - Installing target `Pods-HockeySDK` iOS 6.0
      - Adding Build files
      - Adding resource bundles to Pods project
      - Generating public xcconfig file at `Pods/Pods-HockeySDK.xcconfig`
      - Generating private xcconfig file at `Pods/Pods-HockeySDK-Private.xcconfig`
      - Generating prefix header at `Pods/Pods-HockeySDK-prefix.pch`
      - Generating dummy source file at `Pods/Pods-HockeySDK-dummy.m`
    - Installing target `Pods` iOS 6.0
      - Generating xcconfig file at `Pods/Pods.xcconfig`
      - Generating target environment header at `Pods/Pods-environment.h`
      - Generating copy resources script at `Pods/Pods-resources.sh`
      - Generating acknowledgements at `Pods/Pods-acknowledgements.plist`
      - Generating acknowledgements at `Pods/Pods-acknowledgements.markdown`
      - Generating dummy source file at `Pods/Pods-dummy.m`
  - Running post install hooks
  - Writing Xcode project file to `Pods/Pods.xcodeproj`
  - Writing Lockfile in `Podfile.lock`
  - Writing Manifest in `Pods/Manifest.lock`

Integrating client project
```

下面我们仔细看下日志。

## 解析Podfile文件

如果你曾经想知道为什么 podfile 语法看起来有点奇怪, 那是因为你实际上是在写 ruby。与现在可用的其他格式相比, 它只是一个更简单的 DSL。

所以，安装依赖的第一步的时候就是指出所有的隐式或者显式的pod。`CocoaPods` 通过加载 podspecs 来生成所有依赖的Pod及其所有的版本列表。podspecs 本地存储在 `~/. cocoapods` 中。

![podspec](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/podspec.png)

### 版本控制与冲突解决

CocoaPods使用 [Semantic Versioning](http://semver.org/) 建立的约定来解决依赖的版本问题。这使得解决依赖关系更加容易，因为冲突解决系统可以依靠补丁版本之间的不间断更改。比如：有两个repo依赖`CocoaLumberJack`不同的版本。如果一个依赖版本`2.3.1`另一个依赖`2.3.3`，版本控制系统会使用最新的版本`2.3.3`，因为一般来说`2.3.3`是向后兼容`2.3.1`。

但是有时候会出现问题，因为有很多的第三方的库并不依赖这样的约定，这就导致解决版本冲突变的困难。

当然，还有一些手动解决版本冲突的方式。如果有一个repo依赖`CocoaLumberJack`版本是`1.2.5`,另一个repo依赖为`2.3.5`，这时候你只能手动的设定版本号了。



## 加载源代码

日志中下一步实际上是加载源代码。每一个`.podspec`都包含了对文件的引用，通常包含了远程git地址和tag。tag最后会处理成提交的`SHA`然后会储存在`~/Library/Caches/CocoaPods`.`Core Gem`的一个任务就是把源代码pull到这个缓存文件中。

然后`Core Gem`会根据`Podfile`/`.podspec`/`caches`把源代码加载到项目的`Pods`文件夹中。

![pod](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/podcache.png)

## 生成Pod.xcodeproj

每次运行`pod install`并且检测到依赖的变化，`Xcodeproj gem`都会更新`Pods.xcodeproj`。如果`Pods.xcodeproj`文件不存在，则会根据默认设置创建文件。否则，就会加载存在的文件。

## 集成依赖Repo

CocoaPods向项目集成repo的时候，它除了添加了源文件，还添加了很多其他设定。由于每个library都有自己的target，因此每个library都有下面的几个设定文件：

- `.xcconfig`:包含了build settings.

- `private .xcconfig`:将这些生成设置与默认cocoapods配置合并。

- `prefix.pch`:需要预编译的头文件

- `dummy.m`: 需要预编译的文件，

当所有依赖的target设置完成，这个`Pods`  target就会创建完成。这个也会添加一些其他文件。如果repo包含bundle资源，在`Pods-Resources.sh`中包含了如何将资源文件添加到你的app-target中。还有一个文件`Pods-environment.h`这个文件定义了一些宏帮助你来检测依赖repo是不是来自pod.最后会生成两个声明文件`plist`&`markdown`.


到目前为止，上面的这些操作都是在内存中处理完成的。为了保证这些工作不会重复执行，我们需要一个文件来记录这些东西。`Pods.xcodeproj`会被写入本地，同时产生的还有`Podfile.lock`&`Manifest.lock`。

### Podfile.lock

这是CocoaPods产生的一个很重要的文件。这个文件记录了所有依赖的pods最终的版本。如果你不确定pod安装的版本，你可以检查这个文件。如果将此文件加入到源代码管理中，这也有助于团队之间的一致性，建议这样做。

### Manifest.lock

这是每次执行`pod install`时创建的`Podfile.lock`的副本。如果你遇到提示错误：`The sandbox is not in sync with the Podfile.lock`，这是因为你`Podfile.lock`和`Manifest.lock`不一致导致的。由于Pods目录并不总是在版本控制下，因此确保开发人员在运行前更新pods，否则应用程序将崩溃，或者以其他不常见的方式失败。

## xcproj

如果你安装了[xcproj](https://github.com/0xced/xcproj) ，我们建议这样做，它可以打开`Pods.xcodeproj`文件并转化为`ASCII`格式文件。[ Xcode Project File Format](http://www.monobjc.net/getting-started.html).为什么？这些文件的不再接受修改，并且已经有一段时间没有修改，但是Xcode仍然依赖于它。如果没有xcproj，pods.xcodeproj将被写为一个XML plist，当您在xcode中打开它时，它将被重写，从而导致较大的文件差异。



# 结论

运行pod install的向项目中添加了许多文件，并由系统创建了许多文件。这个过程通常只需要几秒钟。当然，集成第三方库也可以不用CocoaPods，但这要比几秒钟长得多。

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">持续集成</h1>

CocoaPods在持续集成中发挥得非常好。而且，根据您的项目设置方式，构建这些项目仍然相当容易。

## 将Pods文件进行版本管理

如果您对`pods`文件夹及其内部的所有内容进行版本控制，那么您不需要为持续集成做任何特殊的事情。只需构建`.xcworkspace`的时候正确选择`scheme`，因为在编译`workspace`的时候需要制定`scheme`。

## 忽略Pods文件夹

当您不对您的pods文件夹进行版本管理时，您需要做更多的事情来使持续集成正常工作。至少，Podfile需要进行管理。建议生成的.xcworkspace和podfile.lock也受版本控制，以便于使用，并确保使用正确的pod版本。

一旦设置完成，在CI上运行cocoapods的关键是确保在每次构建之前运行`pod install`。在大多数集成工具上，如Jenkins或Travis，您可以将这些定义为构建步骤（事实上，Travis将自动执行）。随着[Xcode Bot](https://groups.google.com/d/msg/cocoapods/eYL8QB3XjyQ/10nmCRN8YxoJ)的发布，我们还没有很好地找到一个顺利的写作过程，但我们正在努力寻找一个解决方案，我们会确保一旦我们这样做了就共享它。

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">总结</h1>

Cocoapods简化了Objective-C开发流程，我们的目标是提高第三方开放源代码库的可发现性和参与度。了解`Pod install`幕后发生的事情只会帮助你开发出更好的应用程序。我们已经完成了整个过程，从加载`spec`和源代码，创建`.xcodeproj`及其所有组件，到将所有内容写入磁盘。所以下一次，运行`pod install --verbose`并观察魔法的发生。
