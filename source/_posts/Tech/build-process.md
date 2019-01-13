---
title: The Build Process
date: 2017-01-10 13:47:40
categories: [Tech]
tags: [Compile,Xcode]
---

![](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/xcode.jpeg)
**Because he can take it, because he's not a hero. He's a silent guardian... a watchful protector. A dark knight!  --  The Dark Knight**
**我们这些天被宠坏了-- 我们只需要点击一下xcode上面的按钮，然后一首歌的时间，或者几秒，我们的app就可以运行。非常神奇，直到出现了错误。**
<!--more-->

在这篇文章中，我们会深入编译过程，然后去剖析xcode面板上的和项目有关的设定。更加深入的去研究每一步是如何工作的。

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">破译构建日志</h1>

我们深入了解Xcode编译过程内部工作原理的第一点就是查看完整的编译log。打开导航栏，选中一个编译，xcode会为你展示详细的log。
![image](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/precompile-header.png)

默认情况下，这个面板隐藏了很多的信息，你可以选中某行的信息，点击右侧的折叠按钮查看更多的信息。另外一个方式就是选中一个或者多个信息然后按下Cmd + C,这样会复制所有的细腻。至少你可以通过编辑menu查看所有的信息。

在我们的例子中，log大概有10000行，下面开始分析：
首先我们可以注意到日志根据我们项目的target分为了不同的模块：

```objectivec
Build target Pods-SSZipArchive
...
Build target Makefile-openssl
...
Build target Pods-AFNetworking
...
Build target crypto
...
Build target Pods
...
Build target ssl
...
Build target objcio
```

我们项目有依赖第三方库：使用Pods管理AFNetworking和SSZipArchive;使用subproject管理OpenSSL.
对于每一个target,Xcode会执行一系列的步骤，将源码最终转化为机器可识别的二进制文件。我们首先来看第一个target，SSZipeArchive.
在这个target的输出日志中，我们可以看到每个target日志里面详细。比如，第一个task是处理precompiled 头文件的：

```objectivec
(1) ProcessPCH /.../Pods-SSZipArchive-prefix.pch.pch Pods-SSZipArchive-prefix.pch normal armv7 objective-c com.apple.compilers.llvm.clang.1_0.compiler
    (2) cd /.../Dev/objcio/Pods
        setenv LANG en_US.US-ASCII
        setenv PATH "..."
    (3) /.../Xcode.app/.../clang 
            (4) -x objective-c-header 
            (5) -arch armv7 
            ... configuration and warning flags ...
            (6) -DDEBUG=1 -DCOCOAPODS=1 
            ... include paths and more ...
            (7) -c 
            (8) /.../Pods-SSZipArchive-prefix.pch 
            (9) -o /.../Pods-SSZipArchive-prefix.pch.pch
```

这些块代表了每个任务的编译过程，所以我们再详细的检查下。

1.每个以行数开头的块都描述了这个任务
2.第二是执行这个task的子描述，在这个例子中，操作的目录发生变化，重新设置 `Lang` 和 `PATH`环境变量。
3.这是最有意思的地方。为了处理`.pch`文件,`clang`会设定很多的设定。这一行展示了所有的参数，我们会查看其中一些...

- 参数`-x`指定编译语言，在这里就是`objective-c-header`.
- 目标架构指定为`armv7`
- 隐式的添加`#define`
- `-c`标签告诉`clang`应该做什么。`-c`意味着运行`preprocessor`、`parser`、`type-checking`, `LLVM 生成和优化`,还有指定`target`的集成代码生成阶段。最后使用`assembly`产生 `.o`文件。
- 输入文件
- 输出文件

还有很多的需要讨论，我们不在详细讨论其他task的过程。**关键的是你需要知道在整个构建阶段调用了什么工具以及使用哪些参数**
对于这个target，实际上有两个task需要处理`objective-c-header`文件，尽管只有一个`pch`文件，我们继续看下这个发生了什么：

```objectivec
ProcessPCH /.../Pods-SSZipArchive-prefix.pch.pch Pods-SSZipArchive-prefix.pch normal armv7 objective-c ...
ProcessPCH /.../Pods-SSZipArchive-prefix.pch.pch Pods-SSZipArchive-prefix.pch normal armv7s objective-c ...
```

这个target会编译两个平台`armv7`&`armv7s`，因此clang会处理这个pch文件两次。

接下在处理`pch`的过程中，我们发现了一些其他类型的task:

```objectivec
CompileC....
Libtool...
CreateUniversalBinary...
```

从这些名字我们可以猜到它的作用：

- {% label primary@CompileC %}: 就是编译`.m`和`.c`文件
- {% label primary@Libtool %}: 根据文件创建library
- {% label primary@CreateUniversalBinary %}: 会合并前面阶段产生的两个`.a`文件（分别运行在`armv7`和`armv7s`）

项目的其他Pod的依赖会执行同样的步骤。`AFNetworking`会像`SSZipArchives`一样编译然后`link`到项目中。



直到所有的`dependencies`编译成功，才开始编译我们app自己这个`target`。输入日志中包含了一些有意思的东西：

```objectivec
PhaseScriptExecution ...
DataModelVersionCompile ...
Ld ...
GenerateDSYMFile ...
CopyStringsFile ...
CpResource ...
CopyPNGFile ...
CompileAssetCatalog ...
ProcessInfoPlistFile ...
ProcessProductPackaging /.../some-hash.mobileprovision ...
ProcessProductPackaging objcio/objcio.entitlements ...
CodeSign ...
```

唯一一个无法从编译名称直到他的作用的是`Ld`，**这个是link tool的名字,类似于`libtool`。事实上，`libtool`经常简称为`ld`和`lipo`,经常用来产生可执行文件`executables`,而`libtool`用来产生`libraries`**。

上面每一步都会调用`command line tool`执行真正的工作，就向上面处理`pch`头文件一样。但是除了了解这些日志输出，我们还需要知道：**Xcode是如何知道哪些task需要执行？**

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">控制构建过程</h1>

当你选中`Xcode`的时候，在项目的编辑区会有七个tab:`General`,`Capabilities`,`Resources Tags`,`Info`,`Build Settings`,`Build Phases`,`Build Rules`。

![build](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/build-setting.png)

我们主要关注最后三个功能。



## 构建任务

编译phase从顶层来控制如何生成`executable`文件。它定义了你的`project`编译过程中需要执行的`task`。

![build-phase](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/build-phase.png)

首先定义了项目的依赖。这个告诉我们的编译系统，在编译我们当前的`target`之前需要先编译的`target`.这个并不是真正的编译过程。Xcode 仅仅是提供了个`GUI`显示编译过程的一些设定。

在`CocoaPods`设定的脚本之后（可以查看 [Michele's article](https://www.objc.io/issues/6-build-tools/cocoapods-under-the-hood/) 获取更多关于CocoaPods的信息）。`Compile Sources`定义了所有需要编译的文件。我们会在`build settings`和`build rules`中深入的了解。在`Compile Sources`中的文件会根据这些`rules`和`settings`来进行处理。

当编译完成之后，下一步就是把所有的东西`link`到一起。这就是我们在 Xcode 中的另一个`Link Binary With Libraries`。这个设定列出了所有需要 link 到项目的`static`和`dynamic`库，这些库是上一步产生的。对于`static`和`dynamic`库的处理是不一样的，你可以查看 [Mach-O Executables](https://www.objc.io/issues/6-build-tools/mach-o-executables/) 深入研究。

当`Linking`操作完成之后，最后一个build阶段就是把静态资源复制到`bundle`中，比如`images`、`fonts`。对于`PNG`图片不仅仅是复制到他们的目的地，同时也会进行优化处理（如果你在build settings中开启`PNG Optimization`）。

尽管拷贝资源是编译的最后一个过程，但是编译过程并没有结束。比如，后面还有`code signing`,但是并不认为他是`build phase`，他是最后一个编译步骤`Packaging`.

### 自定义构建阶段

如果这些默认的构建达不到你的要求，你可以自己设定构建任务。比如，你可以添加构建任务来执行自定义脚本，[Cocoapods](https://www.objc.io/issues/6-build-tools/cocoapods-under-the-hood/) 就是这样添加的额外构建任务。你也可以添加其他构建任务来复制资源。这对于你希望把资源文件复制到指定的目录下很有帮助。

- 使用构建任务可以给你的`app icon`打上版本号和commit的水印。你只需要在构建任务中添加一个`Run Script`,在脚本中使用下面的命令获取版本号和提交记录,下面你就可以使用`ImageMagick`来修改你的App Icon.完整的例子查看 [Github Project](https://github.com/krzysztofzablocki/IconOverlaying):

  ```objectivec
  version=`/usr/libexec/PlistBuddy -c "Print CFBundleVersion" "${INFOPLIST_FILE}"`
  commit=`git rev-parse --short HEAD`
  ```

- 你也可以使用`Run script`来保证你和你成员写的源文件不至于过于肥大。在`Run Script`中输入下面的命令可以保证你的代码在超过设定的行数之后提出警告。

  ```objectivec
  find "${SRCROOT}" \( -name "*.h" -or -name "*.m" \) -print0 | xargs -0 wc -l | awk '$1 > 200 && $2 != "total" { print $2 ":1: warning: file more than 200 lines" }'
  ```

## 构建规则

构建规则定义了不同类型的文件应该如何编译。通常情况下你不需要修改这个设定选项。但有时候你需要为特定的文件类型添加自定义过程，你就可以添加一个新的`build rule`。

构建规则可以指定它作用的文件类型，这个文件应该如何处理，以及处理后文件的输出路径。下面我们创建了一个预处理，以`Objective-C`文件为输入文件，分析次文件中用于生成布局约束的语言的注释，输出文件中包含了生成的代码。因为没有办法在在build rules中定义一个build以.m作为输入文件和输出文件，这里我们使用`.mal`的扩展文件来处理这样的build rules:

![buile rules](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/build-rule.png)



这个规则应用于所有的`*.mal`格式的文件，并且这些后缀的文件使用自定义的脚本处理(这个脚本会以输入和输出路径作为参数调用预处理器)。最后，这个规则告诉编译系统在哪里找到这个编译规则生成的文件输出。

在本例中，输出只是一个普通的`.m`文件，所以它被编译`.m`文件的规则捕获，并且一切都将继续进行，就好像我们已经手动将预处理步骤的结果写入`.m`文件中一样。

在这个脚本中我们我们定义了变量来指定路径和文件名称。你可以从苹果 [Build Setting](https://developer.apple.com/library/mac/documentation/DeveloperTools/Reference/XcodeBuildSettingRef/1-Build_Setting_Reference/build_setting_ref.html#//apple_ref/doc/uid/TP40003931-CH3-SW105) 中查看到可用的变量定义。查看构建过程中所有已经存在的变量的值，你可以增加一个`Run Script`来在构建日志中显示构建变量值。

![show-log](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/build-log1.png)

构建日志中：

![log](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/build-export.png)

## 构建设置

到目前为止，我们已经看到`build phase`用来定义编译过程中的任务，`build rules`指定每种类型文件在编译过程中是如何处理的。在编译设定中，我们可以配置之前日志输出中的task的详细信息。

对于构建过程中的每个阶段，从编译到`link`再到代码签名和打包，你可以找到大量的可选项。注意，这些构建设置是如何划分是和构建阶段相关的，有时还与特定的编译文件类型相关。

其中许多的选项都有很好的文档，你可以在右侧的快速帮助查看或者[Build Setting](https://developer.apple.com/library/mac/documentation/DeveloperTools/Reference/XcodeBuildSettingRef/1-Build_Setting_Reference/build_setting_ref.html#//apple_ref/doc/uid/TP40003931-CH3-SW105)。

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">工程文件</h1>

上面我们讨论过的设定都保存在`project`文件中(`.pbxproj`)，还有其他和项目有关的信息（比如文件分组信息）。你很少会接触到这个文件内部，除非你需要解决这个文件的冲突。

我们鼓励大家使用自己喜欢的编译器打开这个文件从头到尾过一遍。它的可读性令人惊讶，你会认识到大多数构建任务的含义。认真阅读和理解完整的`project`文件对于你解决这个文件的冲突有很大的帮助。

## rootObject

首先来看下我们的入口文件，称为`rootObject`.在我们这个文件中，这个在下面的这行：

```objectivec
rootObject = 1793817C17A9421F0078255E /* Project object */;
```

我们可以根据这个project的ID**1793817C17A9421F0078255E**找到我们主项目的定义的地方：

```objectivec
/* Begin PBXProject section */
    1793817C17A9421F0078255E /* Project object */ = {
        isa = PBXProject;
...
```

这个部分定义了几个key-value,我们后面会理解这个文件是如何构造的。比如，`mainGroup`指出了根文件分组。如果你继续追踪这个id，你可以快速的看到项目文件是如何在`.pbxproj`表示的。但是我们首先来研究下和编译过程有关的。

## target

`target`关键字指出了编译目标的定义：

```objectivec
targets = (
    1793818317A9421F0078255E /* objcio */,
    170E83CE17ABF256006E716E /* objcio Tests */,
);
```

下面是我们根据`target`的ID找到`target`的定义：

```objectivec
1793818317A9421F0078255E /* objcio */ = {
    isa = PBXNativeTarget;
    buildConfigurationList = 179381B617A9421F0078255E /* Build configuration list for PBXNativeTarget "objcio" */;
    buildPhases = (
        F3EB8576A1C24900A8F9CBB6 /* Check Pods Manifest.lock */,
        1793818017A9421F0078255E /* Sources */,
        1793818117A9421F0078255E /* Frameworks */,
        1793818217A9421F0078255E /* Resources */,
        FF25BB7F4B7D4F87AC7A4265 /* Copy Pods Resources */,
    );
    buildRules = (
    );
    dependencies = (
        1769BED917CA8239008B6F5D /* PBXTargetDependency */,
        1769BED717CA8236008B6F5D /* PBXTargetDependency */,
    );
    name = objcio;
    productName = objcio;
    productReference = 1793818417A9421F0078255E /* objcio.app */;
    productType = "com.apple.product-type.application";
};
```

## buildConfigurationList

`buildConfigurationList`定义了所有可用的配置，通常是`Debug`和`Release`。下面是根据`Debug`的ID找到编译设定的存储位置：

```objectivec
179381B717A9421F0078255E /* Debug */ = {
    isa = XCBuildConfiguration;
    baseConfigurationReference = 05D234D6F5E146E9937E8997 /* Pods.xcconfig */;
    buildSettings = {
        ALWAYS_SEARCH_USER_PATHS = YES;
        ASSETCATALOG_COMPILER_LAUNCHIMAGE_NAME = LaunchImage;
        CODE_SIGN_ENTITLEMENTS = objcio/objcio.entitlements;
...
```

## buildPhases

`buildPhase`列出了我们在Xcode中定义的所有编译任务。这里非常容易识别，因为Xcode在build phase ID的后面做出了备注。

`buildRules`是空的，因为在这个项目中我们没有定义任何自定义编译规则。

`dependenices`列出了再Xcode 编译任务中定义的依赖。

以项目的ID作为线索。一旦你掌握了窍门并且理解了xcode中不同编译模块的之间的关系，在处理project文件冲突的过程中你可以更加容易的指出错误。你甚至可以在不打开xcode的情况下载github上直接查看project文件。

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">总结</h1>

现代软件是建立在其他软件的复杂堆栈之上的，比如`libraries`和`build tools`。反过来，软件本身就是建立在低层次的基础上。这就像一层一层的剥洋葱。虽然整个堆积可能很复杂，任何人都无法理解，但是你可以一层一层的剥离。





*************

Reference：[The Build Process](https://www.objc.io/issues/6-build-tools/build-process/)


