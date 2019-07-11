---
title: Xcode构建过程的后台工作
date: 2019-07-08 13:47:40
categories: [iOS]
tags: [iOS 高手]
---

探讨Xcode编译过程，编译系统：

1. Xcode的构建过程

   - Command + B发生了什么
   - 如何架构构建过程
   - Xcode如何用project文件的信息，决定构建过程的模型和流程

2. 编译器领域

   - Clang & Swift如何将源代码加入目标文件
   - 展示头文件和模块的运行
   - 编译器如何在代码中查找声明
   - Swift编译模型和C、C++ 和OC的不同

3. 连接器

   - 什么是符号及其与源代码的关系

   - 链接器是如何将编译器产生的目标文件整合成最终的可执行文件提供给App或者frameworks使用

# Xcode构建过程

## 构建系统

构建过程经过很多步骤，从源代码和项目资源开始，到最终提供给客户打包文件；需要编译和连接源代码、复制和处理资源，比如头文件资源目录和storyBoard、最后是代码签名和自定义shell脚本或者make文件，比如运行代码检查和验证工具。

![构建](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/Screen%20Shot%202019-07-09%20at%2010.11.42%20PM.png)

大多数任务在构建过程中都是由 **命令行工具运行**，比如 **Clang、LD、AC工具，IB工具，代码符号**等。***构建系统***就是将这些任务进行管理。

### 构建任务执行顺序

构建系统里面的任务按照特定的 **顺序进行**；那么这个顺序怎么决定和其重要性：

![构建顺序](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/Screen%20Shot%202019-07-09%20at%2010.33.31%20PM.png)

1. 构建任务的执行取决于信息的依赖关系；就是任务需要的输入和任务生成的输出；
   - 比如编译任务，需要源代码（.m文件）作为输入，以（.o）文件输出；
   - 比如链接器任务，需要几个目标文件，这些文件由编译器的上个任务中生成，再生成可执行或lib文件。

### 构建任务的依赖顺序

![依赖顺序](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/Screen%20Shot%202019-07-09%20at%2010.45.02%20PM.png)

构建系统通过构建任务的依赖关系决定任务执行的顺序，以及平行运行的任务。

## Build发生了什么？

### 构建系统获取构建描述

从Xcode项目文件，解析项目中的所有文件，目标app和依赖关系，构建设置，形成了下面的**构建树结构的定向图**：

![定向图](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/Screen%20Shot%202019-07-09%20at%2011.06.46%20PM.png)

上图显示了所由的依赖关系，项目中的输入和输出文件以及处理他们执行的任务；

然后低级执行引擎（是新的构建系统 **llbuild**, Github的开源项目）处理上面这张图，研究上面的依赖关系，决定执行哪个任务.

### 依赖关系例子Clang

因为我们无法获知过多的依赖关系信息，构建系统可能在任务的执行过程中找到更多，比如Clang编译OC文件时，会生成目标文件（.o 文件），但同时也生成另一个文件（.d）,这个文件其中有一个列表，列出源文件中的头文件；如果你更改了其中任何头文件，那么下次构建(Build)时，构建系统会使用这个文件中的信息（蓝色虚线），保证再次编译源文件。（下图的.h -> .d -> .m -> .o）



![clang](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/Screen%20Shot%202019-07-09%20at%2011.18.06%20PM.png)

### Incremental Builds累加构建

构建系统可以只构建部分依赖关系，因此确定合理的依赖关系十分重要。

## 影响构建系统的参数

### 构建系统自动检测是否需要重新运行任务

- 构建过程中的每个任务都有对应的签名，类似于Hash通过计算多个任务相关信息得出。任务的相关信息包括任务的输入的统计信息，比如文件路径和更改时间标签，运行命令的命令行指示，以及其他有关任务的元数据，比如编译器版本。
- 构建系统会追踪任务签名，包括当前和之前的构建，所以它知道每次构建时是否需要重新运行任务。如果某个任务签名发生变化，它就会重新运行这个任务。
- 上面的逻辑就是累加构建原理。

### 如何利用构建系统

- 不要担心任务执行的顺序，这是构建系统的工作
- 作为开发者需要考虑任务之间的依赖关系，让构建系统决定最佳的执行方法

## 任务依赖关系来源

### Build in

![build in](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/Screen%20Shot%202019-07-10%20at%209.36.32%20PM.png)

主要是来自构建系统自带的数据，比如编译器、连接器、资源目录、Storyboard等，这些规则定义了哪些是输入文件，哪些是输出文件。

### Target Dependencies 目标依赖关系

![target](https://github.com/HaviLee/Blog-Images/raw/master/高手/Screen%20Shot%202019-07-10%20at%209.36.40%20PM.png)

决定了构建的顺序，有时，构建系统（xcode）可以编译不同目标和平行文件，编译源阶段会提前开始，并且可以并行进行。

### Implicit(隐性) denpendencies

![implicit](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/Screen%20Shot%202019-07-10%20at%209.36.49%20PM.png)

在链接库里面列出一个目标，用二进制构建阶段，隐形依赖关系由scheme编辑器生成，默认是开启的，构建系统会为这个目标建立隐形依赖关系。即便不在目标依赖关系中。

### Build Phase Dependencies

![phase](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/Screen%20Shot%202019-07-10%20at%209.36.57%20PM.png)

构建阶段依赖,在这里你会看到几个构建阶段，复制头文件，编译源代码，复制bundle资源等。

### Scheme order dependencies

![scheme](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/Screen%20Shot%202019-07-10%20at%209.37.05%20PM.png)

如果开启了并行编译，这里的顺序完全不用考虑，如果关闭了并行编译，构建系统会按照你的排列顺序进行编译。

### 自定义shell脚本构建阶段或规则

![shell](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/11.png)

可以确定构建系统的输入和输出；避免重复运行不必要的脚本任务；

不要依赖项目里面目标依赖关系的自动连接，Clang编译器由自动关联功能；在构建设置中自动使用关联框架，让编译器自动连接框架对应导入的模块，不用再链接库构建阶段再明确表示。但是自动关联不会建立依赖关系，再构建系统层级，所以它不能保证依赖的目标在关联之前已经建好。所有clang这种自动关联框架只能用在平台STK的框架，比如UIKit,FoundationKit.因为这些在构建前已经存在了。

![yi](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/12.png)

### Create workspace  & Project reference

总结：创建明确的依赖关系可以帮助构建系统更好的并行构建任务，并保证每次构建的结果一致。

# Clang编译器

Clang是Apple的官方编译器。用于所有C语言，比如C、C++、OC。

## 头文件映射

![编译过程](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/Screen%20Shot%202019-07-11%20at%209.53.01%20PM.png)

编译器一次性编译多个的输入文件，仅仅生成一个输入文件，之后被链接器使用。如果想要从访问OS层的API，或者访问自己的代码实现，就需要一个叫做 **头文件** 的东西。头文件就是一种保证，像编译器保证在其他地方存在这个实现文件，通常头文件和实现是匹配的。但是如果你只更新了实现文件，而忘记了头文件，这时候，在编译阶段不会有问题，但是在运行时就会出现问题，因为编译器只检查声明文件，并不会检查实现。

问题出在连接过程，编译器通常包含不止一个头文件，并且所有的编译器都是这样被调用。下面看个实际的例子：

![image](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/Screen%20Shot%202019-07-11%20at%209.59.44%20PM.png)

上面的例子，app本身使用Swift编写，PetKit使用OC编写，还有一个support文件时C++语言写的，随着时间变化，app结构需要调整，这时候你调整文件的目录，不改变任何实现文件，app任然是可以的。为什么？？？

### Clang是如何找到头文件的？

```objective-c
#import <PetKit/Cat.h>

@Implementation Cat 

@end
```

比如上面的代码中，包含一个头文件命名为 **cat.h**,如何查看Clang做的工作呢？

**通过构建日志查看；在xcode的build中；**

打开命令行输入 **clang <list of arguments> -c Cat.mm -o Cat.o -v**;可以得到很多信息；我们只需要关注 **#include <…> search starts here**；这里的所搜路径不是指向源代码的搜索路径；而是看到的一个称为 **headermap**的东西；

![search](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/07111029.png)

Headermaps是xcode构建系统创建，来说明头文件的位置

#### 什么是Header Maps?

![header](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/07111036.png)

前两行只是给这个两个头文件插入了框架的名称，（个人建议不要依赖构建系统为你插入框架的名称，因为不写框架名称可能会对clang模块有影响），第三行是项目头文件，headermap是为了连接回源代码。公开头文件和私有头文件都是为了连接源代码。是为了 **这样做是为了 Clang 可以生成源目录中的文件的有用错误和警告消息,而不是位于生成目录中其他位置的潜在副本**

#### Header maps常见的问题

- 忘记将头文件添加到项目里面，它源目录里面但是不在你项目里
- 头文件命名重复，所有头文件需要区分；

#### Clang如何找到系统头文件



```objective-c
#import <Foundation/Foundation.h>
NS_ASSUME_NONNULL_BEGIN
@interface Pet : NSObject
- (instancetype)initWithName:(NSString *)name; @property (readonly) NSString *name; @property (readonly) BOOL isHungry;
- (void)eat;
 
```

上面的例子中，如何找到 **Foundation.h** 头文件？我们按照上面方式查看search path:

```objective-c
 #include "..." search starts here:  PetKit-generated-files.hmap (headermap)  PetKit-project-headers.hmap (headermap)   
#include <...> search starts here:  PetKit-own-target-headers.hmap (headermap)  PetKit-all-target-headers.hmap (headermap)  DerivedSources 
Build/Products/Debug (framework directory)  $(SDKROOT)/usr/include  $(SDKROOT)/System/Library/Frameworks.(framework directory)
```

头文件映射只用于我们自己的头文件；我们看下最后两行：关注导入路径，默认SDK有两个路径，一个是用户的usr/include；一个是系统库框架，



- **$(SDKROOT)/usr/include/Foundation/Foundation.h**

  这是正常的文件目录，我们只要输入搜索关键词，这里是Foundation/Foundation.h，但是头文件在这里没找到；找下一个；

- **$(SDKROOT)/System/Library/Frameworks/Foundation.framework/Headers/Foundation.h**

  这是框架目录，Clang查找方式有点不同，首先Clang确定这个框架（Foundation.framewrok）是否定义，然后在头文件目录中(Headers)查找头文件，找到就会停止。如果上面的没有找到呢？下面查找私有头文件目录，比如下面的例子：

- **$SDKROOT/System/Library/Frameworks/Foundation.framework/PrivateHeaders/Bogus.h**

  Apple的Framework不会有任何私有的头文件，但是你自己的框架会有，所以上面步骤之后也会检查你自己的私有头文件。如果私有的头文件目录中也没有，Clang所搜就会停止。

可以让xcode执行Product -> Perform Action -> Preprocess "*.m"，生成这样的headermap文件。

按照上面的方式clang需要处理800多个文件夹找到最后的 **Foundation.h**,每次进行编译都是如如此。如何解决这个问题呢？

### Clang Modules

Clang Modules只需要我们为Framework查找和解析头文件一次，然后存储到硬盘换成并可以再利用，这样可以提升构建速度。

- On-disk cached header representation
- Reuseable
- Faster build times

#### Clang Module实现



