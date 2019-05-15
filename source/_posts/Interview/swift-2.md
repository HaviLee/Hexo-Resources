---
title: Swift
keywords: iOS面试
date: 2019-05-01 16:47:40
categories: 
  - 面试
tags:
  - Swift
comments: true
---

**Question #1**

**请用更好的方式写这个for循环：**



```
`for var i = 0; i < 5; i++ {   print("Hello!") }`
```



**Answer：**



```
for _ in 0...4 {
  print("Hello!")
}
```

**Swift实现了两个范围操作符，闭合操作符和半开操作符。 第一个包括范围中的所有值。 例如，以下包括从0到4的所有整数：0 ... 4半开操作符不包括最后一个元素。 以下产生相同的0到4结果：0 .. <5**







***\*Question #2\****

**考虑下面的代码：**



```
struct Tutorial {
  var difficulty: Int = 1
}
 
var tutorial1 = Tutorial()
var tutorial2 = tutorial1
tutorial2.difficulty = 2
```



**tutorial1.difficulty和tutorial2.difficulty的值是什么？ 如果Tutorial是一个类，这将是什么不同？ 为什么？**





**Answer:**

**tutorial1.difficulty是1，而tutorial2.difficulty是2。**
**Swift中的结构是值类型，它们通过值而不是引用来复制。 以下行创建tutorial1的copy并将其分配给tutorial2：**

```objc

var tutorial2 = tutorial1

从这一行开始，对tutorial2的任何更改都不会反映在tutorial1中。
如果Tutorial是一个类，tutorial1.difficulty和tutorial2.difficulty将是2.在Swift中的类是引用类型。 对tutorial1的属性的任何更改都将反映到tutorial2中，反之亦然。
```




 


 

**Question  #3**
 

**view1用var声明，view2用let声明。  这里有什么区别，最后一行会编译？** 



```
import UIKit
 
var view1 = UIView()
view1.alpha = 0.5
 
let view2 = UIView()
view2.alpha = 0.5 // Will this line compile?
```




 


 

**Answer:**
 

**view1是一个变量，可以重新分配给一个新的UIView实例。  通过让你只能赋值一次，所以下面的代码不编译：**
 



```
view2 = view1 // Error: view2 is immutable
但是，UIView是一个具有引用语义的类，所以你可以改变view2的属性（这意味着最后一行将编译）：
```



```
`let view2 = UIView() view2.alpha = 0.5 *// Yes!*`
```




 

**Question  #4**
 

**此代码按字母顺序对名称数组进行排序，看起来很复杂。  尽可能简化它和关闭。** 

 





 

**Answer:** 

**第一个简化与参数有关。类型推理系统可以计算闭包中的参数的类型，所以你可以摆脱它们：**
  



**返回类型也可以推断，所以放弃它：**
 



**$ i表示法可以替换参数名称：**
 



```
let sortedAnimals = animals.sort { return $0 < $1 }
```

**在单语句闭包中，可以省略return关键字。最后一条语句的返回值成为闭包的返回值：**
 

```
let sortedAnimals = animals.sort { $0 < $1 }
```

**这已经更简单，但现在不要停止！ 对于字符串，有一个比较函数定义如下：**
 

```
func <(lhs: String, rhs: String) -> Bool
```

**这个整洁的小函数使你的代码像下面这样容易：**
 

```
let sortedAnimals = animals.sort(<)
```

**请注意，此渐进的每个步骤都会编译并输出相同的结果，并且您创建了一个字符闭包！**
 



 

 

***\*Question  #5\**** 

***\*此代码创建两个类，Address和Person，它创建两个实例来表示Ray和Brian。\**** 

 



**假设Brian移动到街对面的新建筑，所以你更新代码如下：**





```
brian.address.fullAddress = "148 Tutorial Street"
```




 

**问题是，这样做有什么问题吗？**

 

 

**Answer:** 

**ray也搬到了新大楼！ Address是一个类，有引用语义，所以\**headquarters\**是相同的实例，无论你通过ray或brian访问它。  改变headquarters的地址将改变它们。 你能想象如果brian得到ray的邮件会发生什么，反之亦然？  解决方案是创建一个新的地址分配给Brian，或者将Address声明为结构体而不是类。**
 

 

 

**中场：**

**现在要加大难度了。 你准备好了吗？** 

 

***\*Question  #1\**** 

***\*考虑下面的代码：\****



```
var optional1: String? = nil
var optional2: String? = .None
```



***\*nil和.None之间有什么区别？  optional1和optional2变量如何不同？\**** 

 

 

***\*Answer:\**** 

**没有什么区别。  Optional 、None（.None）是初始化缺少值的可选变量的正确方法，而nil仅仅是.None的语法糖。 事实上，参考下面代码：**
 



```
nil == .None // On Swift 1.x this doesn't compile. You need Optional<Int>.None
```



**记住，在hood下可选是一个枚举**







 

**Question  #2** 

**这里有一个Thermometer(温度计)的模型作为类和结构：** 



```
public class ThermometerClass {
  private(set) var temperature: Double = 0.0
  public func registerTemperature(temperature: Double) {
    self.temperature = temperature
  }
}
 
let thermometerClass = ThermometerClass()
thermometerClass.registerTemperature(56.0)
 
public struct ThermometerStruct {
  private(set) var temperature: Double = 0.0
  public mutating func registerTemperature(temperature: Double) {
    self.temperature = temperature
  }
}
 
let thermometerStruct = ThermometerStruct()
thermometerStruct.registerTemperature(56.0)
```



**这段代码错在哪里？为什么会错？**

 

 

**Answer:** 

**编译器会在最后一行报错。 ThermometerStruct被正确地声明了一个变化函数来改变其内部变量的温度，但编译器报错，因为registerTemperature被调用通过let创建的实例，因此它是不可变的。 在结构中，更改内部状态的方法必须标记为突变，但不允许从不可变变量调用它们。** 

 

 

***\*Question  #3\****  

***\*下面这段代码会打印出什么？为什么？\****

 





 

 

**Answer:**
 

***\*它会打印I  love cars。 捕获列表在声明闭包时创建事物的副本，因此捕获的值不会更改，即使您为thing分配了一个新值。 如果省略闭包中的捕获列表，编译器将使用引用而不是副本。 在这种情况下，当调用闭包时，会反映对变量的任何更改，如下面的代码所示：\**** 

 





 

***\*Question  #4\**** 

***\*这是一个全局函数，用于计算数组中唯一值的数量：\**** 

 





**它使用<和==运算符，因此它将T限制为实现可比较协议的类型。 你可以这样使用它：**



```
countUniques([1, 2, 3, 3]) // result is 3
```





将这个函数重写为Array上的扩展方法，实现以下调用：



```
[1, 2, 3, 3].countUniques() // should print 3
```




 


 

**Answer:**

**在Swift 2.0中，可以通过强制类型约束使用条件扩展泛型类型。 如果通用类型不满足约束，则扩展既不可见也不可访问。 因此，全局countUniques函数可以重写为Array扩展：**
 



```
extension Array where Element: Comparable {
  func countUniques() -> Int {
    let sorted = sort(<)
    let initial: (Element?, Int) = (.None, 0)
    let reduced = sorted.reduce(initial) { ($1, $0.0 == $1 ? $0.1 : $0.1 + 1) }
    return reduced.1
  }
}
```

注意，只有当通用Element类型实现了Comparable协议时，新方法才可用。 例如，如果你在UIViews数组上调用它，编译器会报错，如下所示：



 





 

 

**Question  #5** 

**这里有一个函数来计算给予两个（可选）双精度的除法。  在执行实际划分之前有三个前提条件要验证： - 被除数必须包含非零值 - 除数必须包含非零值 - 除数不能为零** 



```
func divide(dividend: Double?, by divisor: Double?) -> Double? {
  if dividend == .None {
    return .None
  }
 
  if divisor == .None {
    return .None
  }
 
  if divisor == 0 {
    return .None
  }
 
  return dividend! / divisor!
}
```

**此代码按预期工作，但有两个问题： 前提条件可以利用guard语句 它使用强制解包 使用guard语句改进此函数，并避免使用强制解包。**
 



 

 

**Answer:**
 



```

```





 



###  高级

**Question #1**
 

**考虑下面的代码：**





首先，创建一个实例：



```
var t: Thermometer = Thermometer(temperature:56.8)
```

**但是这样更好地初始化它：**
 

```
var thermometer: Thermometer = 56.8
```



**问题：如何实现它？**

 

 

**Answer：**

**Swift定义了以下协议，通过使用赋值运算符，可以使用文字值初始化类型：** 



-  `NilLiteralConvertible`
-  `BooleanLiteralConvertible`
-  `IntegerLiteralConvertible`
-  `FloatLiteralConvertible`
-  `UnicodeScalarLiteralConvertible`
-  `ExtendedGraphemeClusterLiteralConvertible`
-  `StringLiteralConvertible`
-  `ArrayLiteralConvertible`
-  `DictionaryLiteralConvertible`

 采用相应的协议并提供公共初始化器允许特定类型的文本初始化。 在温度计类型的情况下，您实现FloatLiteralConvertible协议如下：





```
extension Thermometer : FloatLiteralConvertible {
  public init(floatLiteral value: FloatLiteralType) {
    self.init(temperature: value)
  }
}
```




 


 

**Question  #2** 
 

**Swift有一组预定义的操作符，执行不同类型的操作，如算术或逻辑。  它还允许创建自定义运算符，一元或二进制。 定义并实现具有以下规格的定制^^电源操作员： - 取两个Ints作为参数 - 返回第二个参数的第一个参数 - 忽略潜在的溢出错误** 


 


 

**Answer：**

**在两个步骤中创建新的自定义运算符：声明和实现。 声明使用operator关键字来指定类型（一元或二进制），组成运算符的字符序列，其关联性和优先级。 在这种情况下，运算符为^^，类型为中缀（二进制）。 关联性是正确的，优先级设置为155，因为乘法和除法的值为150.这里是声明：** 



```
infix operator ^^ { associativity right precedence 155 }
```

实现如下：





```
func ^^(lhs: Int, rhs: Int) -> Int {
  let l = Double(lhs)
  let r = Double(rhs)
  let p = pow(l, r)
  return Int(p)
}
```

注意，它不考虑溢出; 如果操作产生Int不能表示的结果，例如。 大于Int.max，则发生运行时错误。



 

 

**Question #3**
 

**你能用这样的原始值定义一个枚举吗？ 为什么？** 

 





 

 

**Answer：**

**不能。 原始值类型必须： - 符合Equatable协议 - 可以是以下任何类型的字面转换： Int String Character 在上面的代码中，原始类型是一个元组，并且不兼容 - 即使它的各个值。** 

 

 

**Question  #4** 

**考虑下面的代码，将Pizza定义为结构体，Pizzeria定义为协议，扩展名包含方法makeMargherita（）的默认实现：** 



```
struct Pizza {
  let ingredients: [String]
}
 
protocol Pizzeria {
  func makePizza(ingredients: [String]) -> Pizza
  func makeMargherita() -> Pizza
}
 
extension Pizzeria {
  func makeMargherita() -> Pizza {
    return makePizza(["tomato", "mozzarella"])
  }
}
```

现在创建一个Lombardis的结构体：





```
struct Lombardis: Pizzeria {
  func makePizza(ingredients: [String]) -> Pizza {
    return Pizza(ingredients: ingredients)
  }
  func makeMargherita() -> Pizza {
    return makePizza(["tomato", "basil", "mozzarella"])
  }
}
```

以下代码创建Lombardi的两个实例。 这两个中的哪一个将包含basil？



 





 

**Answer:**

**都包含。  Pizzeria协议声明了makeMargherita（）方法并提供了一个默认实现。 该方法在Lombardis实现中被覆盖。 由于在两种情况下在协议中声明该方法，所以在运行时调用正确的实现。 如果协议没有声明makeMargherita（）方法，但扩展仍然提供这样的默认实现怎么办？** 



```
protocol Pizzeria {
  func makePizza(ingredients: [String]) -> Pizza
}
 
extension Pizzeria {
  func makeMargherita() -> Pizza {
    return makePizza(["tomato", "mozzarella"])
  }
}
```

 在这种情况下，只有lombardis2会包含，而lombardis1将没有它，因为它将使用在扩展中定义的方法。



 

 

**Question  #5** 

**以下代码具有编译时错误，找到它！**



```
struct Kitten {
}
 
func showKitten(kitten: Kitten?) {
  guard let k = kitten else {
    print("There is no kitten")
  }
 
  print(k)
}
```

提示：有三处错误。



 

 

**Answer:**

**任何guard的else体需要一个退出路径，通过使用return，抛出一个异常或调用一个@noreturn。  最简单的解决方案是添加一个return语句。** 



```
func showKitten(kitten: Kitten?) {
  guard let k = kitten else {
    print("There is no kitten")
    return
  }
  print(k)
}
```

这里是一个抛出异常的版本。



 



 最后，这里有一个实现调用fatalError（），它是一个@noreturn函数。



 





 

 

**一些口述问题**

 

**初学者**

**Question  #1** 

**什么是可选的，可选的什么问题解决？** 

 

**Answer：**

**可选项用于让任何类型的变量表示缺少值。  在Objective-C中，缺少值仅在引用类型中可用，并且它使用nil特殊值。 值类型，例如int或float，没有这样的能力。 Swift通过可选项将缺少值概念扩展到引用和值类型。 可选变量可以在任何时候保持值或零。** 

 

 

***\*Question  #2\**** 

***\*什么时候应该使用结构，什么时候应该使用类？\**** 

 

***\*Answer：\****

***\*关于在结构上使用类是一个好的还是坏的做法，一直存在争论。  函数式编程往往倾向于值类型，而面向对象的编程更喜欢类。 在Swift中，类和结构的特征有很多不同。 您可以将差异总结如下： 类支持继承，结构不支持 类是引用类型，结构是值类型 没有普遍的规则决定什么是最好使用。 一般的建议是使用所需的最小工具来完成你的目标，但一个好的经验规则是使用结构，除非你需要继承或引用语义。 有关详细信息，请查看此详细的帖子。\**** 

 

 

***\*Question  #3\**** 

***\*什么是泛型和他们解决什么问题？\**** 

 

***\*Answer：\****

***\*泛型用于使算法安全地处理类型。  在Swift中，泛型可以在函数和数据类型中使用，例如。 在类，结构或枚举中。 泛型解决了代码重复的问题。 一个常见的情况是当你有一个方法接受一种类型的参数，你必须复制它只是为了适应另一个类型的参数。 例如，在下面的代码中，第二个函数是第一个函数的“clone” - 它只接受字符串而不是整数。\**** 



```
func areIntEqual(x: Int, _ y: Int) -> Bool {
  return x == y
}
 
func areStringsEqual(x: String, _ y: String) -> Bool {
  return x == y
}
 
areStringsEqual("ray", "ray") // true
areIntEqual(1, 1) // true
```



**在OC中可以用NSObject来解决问题：**



```
import Foundation
 
func areTheyEqual(x: NSObject, _ y: NSObject) -> Bool {
  return x == y
}
 
areTheyEqual("ray", "ray") // true
areTheyEqual(1, 1) // true
```

而在swift你可以利用泛型来很好的解决问题：





```
func areTheyEqual<T: Equatable>(x: T, _ y: T) -> Bool {
  return x == y
}
 
areTheyEqual("ray", "ray")
areTheyEqual(1, 1)
```




 

**Question  #4** 
 

**有一些情况下，你不能避免使用隐式解包可选。  什么时候？ 为什么？** 

 

**Answer：**

**使用隐式解包可选的最常见原因是： 当不是自然的nil属性不能在实例化时初始化。 一个典型的例子是Interface Builder插件，它在所有者初始化后总是初始化。 在这种特定情况下 - 假设它在Interface Builder中正确配置 - 插座在使用之前保证为非零。 解决强参考周期问题，这是两个实例彼此引用并且需要对其他实例的非空参考。 在这种情况下，引用的一侧可以标记为未知，而另一侧使用隐式解包的可选。** 

**重要提示：不要使用隐式展开的可选，除非你必须。  使用它们不当会增加运行时崩溃的机会。 在某些情况下，崩溃可能是预期的行为，但是有更好的方法来实现相同的结果，例如，通过使用fatalError（）。** 

 

 

**Question  #5** 

**解包可选的各种方法是什么？  他们如何评价安全性？ 提示：至少有七种方式。** 

 

**Answer:**



-  **forced unwrapping** `!` operator  -- unsafe
-  **implicitly unwrapped** variable  declaration -- unsafe in many cases
-  **optional binding** --  safe
-  **optional chaining** --  safe
-  **nil coalescing** operator  -- safe
-  new Swift 2.0 `guard` statement  -- safe
-  new Swift 2.0 **optional  pattern** -- safe (suggested by [@Kametrixom](https://twitter.com/Kametrixom))



 

**中场**

**到这里你的表现都非常出色，下面来看看这些问题。**

 

**Question  #1** 

**Swift是一种面向对象的语言还是功能语言？** 

 

**Answer:**

**Swift是一种支持这两种范例的混合语言。 它实现了OOP的三个基本原则： - 封装 - 继承 - 多态性 至于Swift是一个功能语言，有不同但等效的方法来定义它。 维基百科上更常见的一种：“...一种将计算视为数学函数的评估，避免改变状态和可变数据的编程范式[...]。 很难说Swift是一个功能齐全的语言，但它有基础。** 

 

 

***\*Question  #2\**** 

***\*Swift中包含以下哪些功能？ 1.通用类 2.通用结构 3.通用协议\**** 

 

***\*Answer:\****

**1和2。  泛型可以在类，结构，枚举，全局函数和方法中使用。 3通过类型部分实现。 它不是一个通用类型本身，它是一个占位符名称。 它通常被称为关联类型，并且当协议被类型采用时被定义。**
 

 

 

**Question  #3** 

**在OC中，常数一般这样声明：**



```
const int number = 0;
```

这是swift的做法：





```
let number = 0
```

它们之间有什么区别吗？ 如果是，你能解释它们有什么不同吗？



 

**Answer:**

**const是使用编译时值或在编译时解析的表达式初始化的变量。 通过let创建的不可变是在运行时确定的常量，可以使用静态或动态表达式的结果初始化它。 请注意，其值只能分配一次。** 

 

 

***\*Question  #4\**** 

***\*要声明静态属性或函数，请在值类型上使用静态修饰符。  这里有一个结构的例子：\**** 



```
struct Sun {
  static func illuminate() {}
}
```

对于类，可以使用静态或类修饰符。 他们实现同样的目的，但在实际上他们是不同的。 你能解释它们有什么不同吗？



 

***\*Answer:\****

***\*static使一个属性或一个函数静态和不可重写。  通过使用类，可以覆盖属性或函数。 当应用于类时，static将成为类final的别名。 例如，在此代码中，编译器会在尝试覆盖illuminate（）时提示：\**** 



```
class Star {
  class func spin() {}
  static func illuminate() {}
}
 
class Sun : Star {
  override class func spin() {
    super.spin()
  }
  override static func illuminate() { // error: class method overrides a 'final' class method
    super.illuminate()
  }
}
```




 


 

**Question  #5**
 

**可以使用扩展添加存储属性吗？  说明。** 

 

**Answer:**

**不，这是不可能的。  扩展可用于向现有类型添加新行为，但不能更改类型本身或其接口。 如果添加了存储属性，则需要额外的内存来存储新值。 扩展无法管理此类任务。(表示不服，runtime，大家自己体会)**
 

 

 

**高级**

**朋友，你是一个聪明人，下面准备好难度升级了吗？**

 

**Question  #1**  

**在Swift  1.2中，你能解释用通用类型声明枚举的问题吗？ 例如，具有两个通用类型T和V的Either枚举，T用作左侧案例的关联值类型，V用于右侧案例：** 



```
enum Either<T, V> {
  case Left(T)
  case Right(V)
}
```



 

**Answer:**

**编译会失败，正确做法如下：**



```
class Box<T> {
  let value: T
  init(_ value: T) {
    self.value = value
  }
}
 
enum Either<T, V> {
  case Left(Box<T>)
  case Right(Box<V>)
}
```



 

 

**Question  #2** 

**闭包是值类型还是引用类型？**

 

**Answer：**

**闭包是引用类型。  如果一个闭包被分配给一个变量，并且该变量被复制到另一个变量中，那么也将复制对同一闭包及其捕获列表的引用。** 

 

 

***\*Question  #3\**** 

***\*UInt类型用于存储无符号整数。  它实现以下初始化器，用于从有符号整数转换：\**** 



 但是，如果提供负值，以下代码会生成编译时错误异常：





```
let myNegative = UInt(-1)
```

知道一个负数是内部表示的，使用二的补码作为一个正数，你怎么能将一个负数转换为一个UInt，同时保持其内存表示？




 

***\*Answer：\****

**使用下面的方式:**



```
UInt(bitPattern: Int)
```



 

***\*Question  #4\**** 

***\*你能描述一个你可能在Swift中获得循环引用的情况，以及你将如何解决它？\**** 

 

***\*Answer:\****

***\*循环引用发生在两个实例彼此之间存在强引用时，从而导致内存泄漏，因为两个实例都不会被释放。  原因是实例不能被释放，只要有一个强的引用，但每个实例保持另一个活着，因为它的强引用。 你可以通过强弱循环引用来解决这个问题，用弱引用(weak)或者无主引用(unowned)替换其中一个强引用。\****