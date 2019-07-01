---
title: 类型移除（Swift Type Erasure）
date: 2019-06-28 13:47:40
categories: [Swift]
tags: [Advanced Swift]
---

## 为什么我们需要移除类型信息？Remove Type Information

Swift是一个多范式编程语言(muti-paradigm)；可以写出响应式编程、面向对象编程、泛型面向协议编程；面向协议的编程的基石是使用公共协议编写代码,并避免与继承相关的紧密耦合。

但是我们经常碰到下面的错误：

> error: protocol 'MyAwesomeProtocol' can only be used as a generic constraint because it has Self or associated type requirements

这个编译错误背后有很多内容。这个错误是同时引用了 Swift 协议的关联类型以及 Swift 的泛型约束。

## 引起这个错误的原因

**这个错误是由于将带有关联类型的协议作为函数的参数或者集合的对象引起的。**

下面的例子：

我们准备在一个tableview中展示files和folders.同时也会定义files和folders的详细View;首先定义一个Row的协议，来约定我们的view提供的内容和需要的参数：

```c++
import UIKit

protocol Row: class {
    associatedtype Model
    var sizeLabelText: String { get set }
    func configure(model: Model)
}
//Model
struct Folder {}
struct File {}
```

然后基于Row协议实现我们的View和detailView:

```c++
class FolderCell: Row {
    typealias Model = Folder
    var sizeLabelText: String = ""
    func configure(model: Folder) {
        print("Configured a \(type(of: self))")
    }
}
class FileCell: Row {
    typealias Model = File
    var sizeLabelText: String = ""
    func configure(model: File) {
        print("Configured a \(type(of: self))")
    }
}
class DetailFileCell: Row {
    typealias Model = File
    var sizeLabelText: String = ""
    func configure(model: File) {
        print("Configured a \(type(of: self))")
    }
}
```

目前为止很完美，符合我们的预期设想，下面我们来使用我们定义的协议：

1.作为Collection中的元素；2.作为函数的参数；3.作为函数的返回值；

```c++
// Collect an array of Row items
let cells: [Row] = [FileCell(), DetailFileCell()]

// Grab a random instance of Row
let randomFileCell: Row = (arc4random() % 2 == 0) ?
                            FileCell() : 
                            DetailFileCell()
                            
// Pass a Row as a function argument
func resize(row: Row) {
    …
}

// or return an instance of Row
func firstRow() -> Row {
    …
}
```

此时我们会遇到错误：

```
error: protocol 'Row' can only be used as a generic constraint because it has Self or associated type requirements.
协议"Row"只能用作泛型约束,因为它具有 Self 或关联的类型要求
```

## 如何解决这个错误？

乍一看,Swift 的泛型类型会为我们提供我们使用协议所需的功能。

```c++
// Try making our protocol generic
protocol Row<Model> {
    func configure(model: Model)
}

let cells: Row<File> = [FileCell(), DetailFileCell()]
```

但是此时你又会看到另一个错误：

>  `error: Protocols do not allow generic parameters; use associated types instead`.

我们首先来看下Swift是如何区分泛型类型（generic types）和协议的关联类型(protocol associated type);Swift 的结构体、类和枚举以及协议的关联类型都用于泛化功能的构造。它们允许我们编写一次代码,可以使用与多个类型中。

虽然在这方面类似,但泛型类型和相关类型以不同的方式完成其抽象,并附带各种权衡。

### 泛型参数：在编译器编译替换为具体类型

泛型类型参数是可在 Swift 结构、类或枚举中使用的类型的占位符；具体类型将在稍后提供,在编译时,具体类型将替换占位符。计划使用泛型类型的任何代码(无论是创建实例还是仅访问实例)都将知道占用占位符的现在具体类型。

比如：

```c++
struct MyRow<T> {
    var sizeLabelText: String = ""
    func configure(model: T) {
        print("Configured a \(type(of: self))")
    }
}
```

为了对 MyRow 实例的引用,我们必须明确泛型T的类型。否则会提示没有声明类型T

```
/// Must be explicit
let myFileRow: MyRow<File> = MyRow<File>() 

/// Cannot leave the placeholder unfilled
let myGenericRow = MyRow<T>() /// Error: Use of undeclared type T
```

**泛型占位符是我们类型公共 API 的一部分；可以理解为是其中的一个参数**

### 关联类型：都是协议的采用者提供类型

另一方面,关联的类型是不同类型的占位符。 Swift 类型协议是唯一可以使用 Swift 关联类型。与泛型类型一样,关联的类型可以在协议定义中的任何位置使用。**关键区别来自占位符的填充时间。**当一个类型使用协议的时候，同时需要提供具体类型来替换协议中的占位类型。

```c++
class FileCell: Row {
    typealias Model = File
    ...
    func configure(model: File) {
        ...
    }
}
```

**由于协议的采用者提供此类型,因此当我们将协议的实例用作方法中的参数、返回类型、var 类型或 let 等时,编译器不知道占位符的类型。**

(e.g., `let cells: [Row] = [FileCell(), DetailFileCell()]`) 上面的这个错误现在更加一目了然；编译器无法知道 Model 的类型是什么,因为它知道数组中每个单元格的只是它符合 Row 协议。该协议的采用者负责提供替代协议中的具体类型。

### 类型擦除模式

我们可以使用类型擦除模式来组合generic type和associated type,以满足编译器和灵活性目标。我们需要创建三个离散类型来满足这些约束。此模式包括抽象基类、私有容器类,最后还包括一个公共包装类。

#### 抽象基类

该过程的第一步是创建一个抽象基类。此基类将有三个要求:

1. 实现Row 协议
2. 定义泛型类型参数Model,作为Row的关联类型
3. 抽象。(每个函数必须由子类重写)

此类不会直接使用,而是需要子类化,以便将泛型类型约束绑定到协议的关联类型。我们已将此类标记为私有,并通过约定将其前缀为 _。

```c++
// Abstract generic base class that implements Row

// Generic parameter around the associated type

private class _AnyRowBase<Model>: Row {
    init() {
        guard type(of: self) != _AnyRowBase.self else {
            fatalError("_AnyRowBase<Model> instances can not be created; 
          create a subclass instance instead")
        }
    }

    func configure(model: Model) {
        fatalError("Must override")
    }

    var sizeLabelText: String {
        get { fatalError("Must override") }
        set { fatalError("Must override”) }
    }
}
```

#### 私有容器（Private Box）

接下来,我们将创建具有以下要求的私有 Box 类:

1. 继承泛型基类 _AnyRowBase
2. 定义一个实现了Row协议的泛型参数Concreate
3. 存储Concreate的实例，后面使用
4. 使用Concreate的实例来实现Row协议中的方法

此类称为 Box 类,因为它包含对协议的具体实现者的引用，Concreate实例在这个例子中就是实现了Row协议的FileCell`, `FolderCell` or `DetailFileCell。我们通过父类_AnyRowBase实现协议、通过子类重写父类的协议中的约束方法。最后，这个类提供了一个中介，用于将具体类的关联类型与基类的泛型类型参数连接起来。

```c++
// Box class

// final subclass of our abstract base

// Inherits the protocol conformance

// Links Concrete.Model (associated type) to _AnyRowBase.Model (generic parameter)

private final class _AnyRowBox<Concrete: Row>: _AnyRowBase<Concrete.Model> {
    // variable used since we're calling mutating functions
    var concrete: Concrete

    init(_ concrete: Concrete) {
        self.concrete = concrete
    }

    // Trampoline functions forward along to base
    override func configure(model: Concrete.Model) {
        concrete.configure(model: model)
    }

    // Trampoline property accessors along to base
    override var sizeLabelText: String {
        get {
            return concrete.sizeLabelText
        }
        set {
            concrete.sizeLabelText = newValue
        }
    }
}
```

### Public Wrapper公共封装

有了我们的私有实现细节，我们需要为类型擦除包装器创建一个公共接口；此模式使用的命名约定是在您要包装的协议前面加前缀Any，在我们的例子中是AnyRow.

这个类有下面的职责：

1. 实现Row协议
2. 定义泛型参数Model，作为Rows的关联类型
3. 在它的初始值设定项中，获取协议的具体实现者。
4. 将具体的实现者包装在一个私有框中Anyrowbase<model>。
5. 将每个函数调用都映射到Box中

```c++
// Public type erasing wrapper class

// Implements the Row protocol 

// Generic around the associated type

final class AnyRow<Model>: Row {
    private let box: _AnyRowBase<Model>

    // Initializer takes our concrete implementer of Row i.e. FileCell
    init<Concrete: Row>(_ concrete: Concrete) where Concrete.Model == Model {
        box = _AnyRowBox(concrete)
    }

    func configure(model: Model) {
        box.configure(model: model)
    }

    var sizeLabelText: String {
        get {
            return box.sizeLabelText
        }
        set {
            box.sizeLabelText = newValue
        }
    }
}
```

请注意，虽然我们的具体实现者是类型为“anyrowbase<model>的属性，但我们必须将其包装在“anyrowbox”的实例中。对anyRowBox的初始值话的唯一要求是具体类实现Row协议。这就是类型擦除实际发生的地方。

下面使用：

```c++
// Collect an array of Row items
let cells: [AnyRow<File>] = [AnyRow(FileCell()), 
AnyRow(DetailFileCell())]

// Grab a random instance of Row
let randomFileCell: Row = (arc4random() % 2 == 0) ?
                            AnyRow(FileCell()) : 
                            AnyRow(DetailFileCell())
                            
// Pass a Row as a function argument
func resize(row: AnyRow<File>) {
…
}

// Return an instance of Row
func firstRow() -> AnyRow<File> {
…
}

// Configure our collection of cells with a new File
cells.map() { 
    $0.configure(model: File())
}

// Mutate and access properties on a Cell
if let firstCell = cells.first {
    firstCell.sizeLabelText = "200KB"
    print(firstCell.sizeLabelText) // Prints 200KB
}
```

引用：[Type Erasure]([s-in-swift/](https://www.bignerdranch.com/blog/breaking-down-type-erasures-in-swift/))

