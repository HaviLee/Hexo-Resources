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

- 和上下文无关

  ```objective-c
  #define ENABLE_CHINCHILLA 1 // ignored  
  #import <PetKit/PetKit.h>
  ***************************
  #define ENABLE_DOG 1 // ignored  
  #import <PetKit/PetKit.h>
  ```

  传统的导入方式，文本也会被导入；预编译器会遵从这个定义并应用到头文件夹，但是如果这样做，每个案例的模块就不一样，不能重复使用，**模块会忽略所有的文本信息**；这样就可以被所有实现文件重复使用了；

- mudule需要各自独立；要明确各自的依赖关系

  ```objective-c
  #import <Foundation/Foundation.h> // import not required before PetKit module  
  #import <PetKit/Cat.h>
  ```

  你只需要导入一个模块，而不需要导入其他头文件；

#### Clang如何知道创建Module

看下面的例子：

```objective-c
#import <Foundation/NSString.h>
 
```

我们知道了上面的头文件是如何找的，Foudation.framework的目录：

```objective-c
Foundation.framework/
	Headers/ 
		Foundation.h 
    NSString.h  ... 
	Modules/  
		module.modulemap
```

Clang编译器首先模块目录和模块映射（和头文件目录有关）；

**什么是modulemap?**

```objective-c
// Module Map - Foundation.framework/Modules/module.modulemap
 framework module Foundation [extern_c] [system] {
   umbrella header "Foundation.h"
   export *
   module * {
   	export *
 	 }
   explicit module NSDebug {
     header "NSDebug.h"
     export *
   }
 }

//umbrella header foundation.h
// Foundation.h
...
#import <Foundation/NSScanner.h>
#import <Foundation/NSSet.h>
#import <Foundation/NSSortDescriptor.h>
#import <Foundation/NSStream.h> #import <Foundation/NSString.h>
#import <Foundation/NSTextCheckingResult.h>
#import <Foundation/NSThread.h>
#import <Foundation/NSTimeZone.h>
...
```

modulemap描述了你的特定的一组头文件是如何映射到你的模块中；上面的就是Foundation的modulemap;

- 说明了哪个头文件属于该模块；这里只有Foundation.h一个，但这是一个特殊的头文件，这个是umbrella头文件，这就是说Clang要查找这个特殊的头文件，来确定NSString.h是不是这个模块的一部分；然后在这个umbrella header中可以找到；
- Clang确定NSSting.h属于Foundation模块后，我们可以替换文本输入到模块输入

#### 如何创建Foundation模块

- 首先为Clang单独创建位置；Clang位置里面包含了所有Foundation的头文件；
- 我们不会转移任何来自原始的编译器调用的现有的上下文
- 实际上转移的是传递给Clang命令行实参，随后继续传递

在创建Foundation模块的时候，framework本身会引入其他框架，也就是说我们有需要构建引用的那些模块，这个会不断向下寻找，但是我么看到一个好处，我们可以复用某些module,**所有的module都要序列化存储到模块缓存区**；

```
$ clang -fmodules —DENABLE_CHINCHILLA=1 ...
98XN8P5QH5OQ/  
  CoreFoundation-2A5I5R2968COJ.pcm  
  Security-1A229VWPAK67R.pcm 
  Foundation-1RDF848B47PF4.pcm
$ clang -fmodules —DENABLE_CAT=1 ...
1GYDULU5XJRF/ 
  CoreFoundation-2A5I5R2968COJ.pcm  
  Security-1A229VWPAK67R.pcm 
  Foundation-1RDF848B47PF4.pcm
```

在创建模块的时候，**命令行实参会向后传递**，就是说这些实参会影响module的内容，因此我们需要hash这些实参，再保存这些为特定编译器调用而创建的moudle到Hash匹配的目录中。

#### 如何为自己的框架创建Module

```objective-c
//
// NorwegianCat.h // PetKit
//
#import <PetKit/Cat.h>
NS_ASSUME_NONNULL_BEGIN @interface NorwegianCat: Cat ...
@end
NS_ASSUME_NONNULL_END
 
```

如果要使用modulemap,它会映射到源目录；下面是源文件目录：

```objective-c
PetWall/  
  PetKit/ 
    Cat/  
      Cat.h 
      Cat.mm 
      ... 
  Info.plist 
  Pet.h  
  Pet.m  
  PetKit.h
```

上面的源文件目录没有模块目录，看上去根本不是框架；clang不知道怎么做，这时有个叫做 **Clang 虚拟文件系统**，他会创建一个虚拟的抽象框架，方便Clang创建模块，**但是，抽象框架只是映射到目录文件的源文件,这样Clang就可以提示源文件的错误**

![dd](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/07171120.png)

**不指定Framework名字可能出错**

```
#import <PetKit/PetKit.h>  
#import “Cat.h”
```

第一行制定Petkit模块，第二行我们可以知道这个是Petkit中的，但是Clang不知道，因为你没有写明框架的名字，此时你有可能受到重复定义的报错，这样的错误常见于同一个头文件被重复导入；

```objective-c
#define ENABLE_CUTE_KITTENS 1
#import <PetKit/PetKit.h>  
#import “Cat.h”
```

修改下上下文，模块的导入不受任何影响，但是cat.h还是文本导入不变，但是此时提示就是矛盾定义，Clang无法为你解决；

**无论是导入公有还是私有的头文件，一定记得写明Framework的名字**



# Swift Builds

对于OC，clang单独编译每个文件；如果你要在另一个文件中引用一个类，你必须引用这个类的头文件；

![dd](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/07180827.png)

**Swift的设计不需要写入头文件**：

- 这对于新手很容易上手
- 避免了在不同的文件里重复一个声明
- 但是需要额外的记录来找打头文件

## Swift调用Swift代码

例子：

```objective-c
import UIKit
class PetViewController: UIViewController {
	var view = PetView(name: "Fido", frame: frame)
	... 
}
-----------------------------
#import "PetWall-Swift.h" 
@implementation AppDelegate 
	...
@end
----------------------------
@testable import PetWall
class TestPetViewController: XCTestCase { 
}
```

为了编译上面的这个PetViewController,编译器需要进行2步运算

1. 首先找到声明，包括Swift Target和OC的；
2. 第二步为类文件生成接口描述，以便声明可以被找到并用于OC和其他Swift目标；

### Swift Target中查找所有的声明

要编译PetViewController.swift编译器会首先查找PetView的初始化方法以便检查调用；但此时，需要先解析PetView.swift，并验证来保证初始化方法的声明是正确的。

编译器现在足够智能，不需要检查初始化方法的主体，但是它还需要做些额外的工作来处理类文件的接口部分；

这和Clang不同，为了编译一个swift文件，编译器会解析Target下所有的swift文件，来检查和编译的接口有关的部分。

![d](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/0718939.png)

#### Xcode9：上面的查找声明的方式会导致重复工作

![d](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/0718947.png)

在xcode9增量调试的构建中：

- 编译器会单独编译每个文件；
- 文件的编译可以并行进行；
- 强制编译器重复解析每个文件，解析一次作为实现文件生成.o文件，最终作为接口查找声明；

#### Xcode10将文件合并成组

![d](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/0718948.png)

- 将swift文件进行分组，尽可能多的分担工作；并保证做大限度的并行编译
- 在同组的文件进行复用处理，只有跨组查询的时候才会进行重新编译，由于分组数量少，因此可以大幅度提升增量调试构建的速度；

## Swift调用OC代码

```objc
import UIKit
class PetViewController: UIViewController {
	var view = PetView(name: "Fido", frame: frame)
	... 
}
```

比如上面的UIKit就是OC语言；这是系统框架，Swift和其他语言不同，它不需要外部功能接口（.h文件）；这就是你需要为每个OC-API编写swift声明；但是编译器内置了Clang的一大部分作为library；这就可以直接导入OC框架；

![d](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/07181031.png)

OC的声明来自哪里？importer会根据Target类型查找头文件；

- 任何target下(OC & Swift)如果你导入的是OC Framework；importer会在clang生成的这个framework的modulemap暴露的头文件中找到定义声明；
- 如果导入的是OC和Swift混编的framework；importer会在umbrella头文件中查找定义的声明；umbrella定义了公共接口；这样在这个framework里面的swift代码就可以调用这个框架里面的公共OC代码；
- 在App和单元测试中，可以导入Target的桥接头文件，bridge_header;允许其中的声明可以被Swift调用

##### importer会对函数进行重命名；

**在Xcode查看导入的Swift生成的接口**

点击可以看到根据swift生成接口.h;还可以根据不同版本的swift生成接口；

![d](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/07181101.png)



## OC调用Swift文件

Swift会生成一个头文件；可以自行导入，

```objective-c
Write in Swift:
class PetCollar: NSObject { 
  @objc
  var color: UIColor = .blue
  var name: String = "Fido" 
}

Use from Objective-C:
#import "PetWall-Swift.h"//编译器为Swift生成的.h文件
	- (void)resetCollarColor { self.collar.color = [UIColor redColor];
}
```

下面是转换过程：

编译器会对所有的继承自NSObject的swift类和标记为@objc的函数生成OC .h头文件；

![d](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/07181115.png)

对于单元测试和App,生成的头文件包含了公共和私有两种声明；就可以再app的OC部分使用内部的Swift代码；**但是，对于Framework,生成的头文件只包含Public声明，因为头文件包含在构建的产出中，是公共接口的一部分**

在上面的图中，

- 右侧编译器将OC类连接到一个包含了module PetWall的别名的Swift类；
- Module可以避免两个相同命名的类在运行时出错；
- 你可以让Swift重命名OC类，通过传递标识给objc 属性使用（@objc(name)）；但是你要保证两个name不重复，比如下面，这样就可以再OC类中使用PWLPetCollar;

![d](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/07181129.png)

## Swift基于Clang的模块概念进行构建

- 再使用声明之前，必须先导入module
- 可以导入Clang module（纯OC module）
- 编译器为每个Swift Target生成单独的module,包括 目标APP;所以单元测试的时候，主app也作为一个module使用；
- 在其他module中使用Swift module；编译器反序列化一个特殊的 swiftmodule文件；在使用时检查它的类型；这类似于编译器在目标里查找生命；最后编译器会生成一个总的module文件；而不是直接解析Swift文件；

下图右侧就是编译器生成的swiftmodule文件，很像OC的头文件，但是这个swiftmodule是二进制文件；它包含了内联函数的主题，很像OC的静态内联函数、C++的头文件实现；这个文件包含了私有声明的名称和类型，这使得你可以再调试器中引用他们；

![d](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/07190915.png)

### 累加构建过程

![d](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/07191009.png)

对于累加构建，编译器生成部分swiftmodule，然后合并成一个文件代表整个Module；合并的时候可能会生成一个OC头文件，很像一个连接器合并多个文件称为一个可执行文件。

# 连接器 Linker

- 连接器是用来做什么的？
- 什么是符号 symbols?
- What are object files? 
- What are libraries? 

## Linker做什么

Linker是构建一个可执行文件Mach-O的最后一个步骤；Linker会合并所有的.o文件称为一个可执行文件；它只是移动和打包代码，不会产生代码；

比如我们有两种类型的输入文件：

- 第一种是Object files（ .o)
- 第二种是各种库(.dylib, .tbd, .a)

## Symbols

- 符号是一个名称，代表代码或者数据片段；
- 代码片段可能会指向其他符号，比如：当一个函数调用另一个函数的时候；
- 符号具有属性可以影响Linker的行为；比如Weak symbols，指的是当你在系统上运行的时候它可能消失，比如某些API事ios11的，有些事iOS12的；
  - 连接器决定哪些符号必须出现和哪些符号可以再运行时处理；
- 编程语言通常将数据编码称为符号，C和C++都可以看到

## Object Files

- Object Files就是编译器的输出内容，.O文件；
- 是不可执行的Mach-O类型文件，这个文件是代码和数据的集合；因为是编译的代码还没有完成，还有缺失；
  - 每个文件的fragment是以符号表示，比如Print函数，就是用符号代替代码
  - fragment可能会引用没有定义的符号，比如如果一个.o文件引用另一个.o文件的函数，那么这个.o文件是未定义的。连接器会查找未定义符号进行link

## Libraries

- 库是定义符号的文件，但是这个文件不属于你的Target;

  - Dylibs: 动态库 frameworks
    - 动态库的Mach-O文件暴露的代码和数据片段可以被可执行文件是使用；他们是系统的一部分；
  - TBDs:基于文本的动态库文件
    - 是给iOS和MacOS创建SDK的时候，比如Mapkit,我们有你可能用到的所有这些动态库和函数；但是我们不想把这些数据和SDK一起加载，这样体积太大；编译器和连接器都不需要这个文件；它只要运行程序；我们创建了stub dylib，删除了所有的符号的主体，只保留了名字；然后转换为文本，这样使用起来比较容易；目前只用于发布SDK来减少体积；
  - Static archives 静态库
    - .a是之前使用AR工具创建的.o文件的集合，或者是lib的集合；
    - AR创建并维护文件组，将它合并称为一个.a库，听上去像是Tar或者zip文件；事实上确实是这样的，.a在更好的工具产生之前被UNIX所用，但是现在的编译器和连接器完全可以理解它并使用，它只是个archive文件；他们孕育了动态链接，在过去，所有的代码都被存档，因此不能涵盖所有的c库，因此如果.o文件含有符号，我们会把整个.o文件从库中提出来，但不会带入其他.o文件，如果在之间引用符号，只需要带入即可，如果是非符号行为，比如静态初始程序，或者将它们以个人dylib的形式重新导入，你需要明确的强制加载或者指定加载让连接器提取所有，即使这些文件没有关联。

  有了很多的.o,会将所有的.o输入到连接器，连接器会创建文件夹放置这些文件

  

![d](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/高手/07191521.png)

