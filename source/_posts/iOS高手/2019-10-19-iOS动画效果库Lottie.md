---
title: iOS动画效果库Lottie
date: 2019-10-19 20:47:40
keywords: iOS
description: 手动编写动画的代码也非常复杂，不容易维护，很多动画细节的调整还需要和动画设计师不断沟通打磨，尤其是千行以上的动画代码编写、维护、沟通的成本巨大。
categories: 
  - iOS高手
tags:
  - Lottie
comments: false



---

# Lottie

[Lottie 框架](http://airbnb.io/lottie/#/)就很好地解决了动画制作与开发隔离，以及多平台统一的问题。

- 动画设计师做好动画以后，可以使用[After Effects](https://www.adobe.com/products/aftereffects.html)将动画导出成JSON文件，然后由Lottie 加载和渲染这个JSON文件，并转换成对应的动画代码。
- 开发集成Lottie,由于是JSON格式，文件也会很小，可以减少 App 包大小。运行时还可以通过代码控制更改动画，比如更改颜色、位置以及任何关键值。
- Lottie 还支持页面切换的过场动画（UIViewController Transitions）。
- 然后使用 [Bodymovin](https://github.com/airbnb/lottie-web)进行导出。

# Bodymovin

- 需要先到[Adobe官网](https://www.adobeexchange.com/creativecloud.details.12557.html)下载Bodymovin插件，并在 After Effects 中安装。
- 使用 After Effects 制作完动画后，选择 Windows 菜单，找到 Extensions 的 Bodymovin 项，在菜单中选择 Render 按钮就可以输出JSON文件了。

`资源：`

[LottieFiles网站](https://lottiefiles.com/)还是一个动画设计师分享作品的平台，每个动画效果的JSON文件都可下载使用。

# 在iOS中使用Lottie

## 使用

- 通过 CocoaPods 集成 Lottie 框架到你工程中。Lottie iOS 框架的 GitHub 地址是https://github.com/airbnb/lottie-ios/，官方也提供了[可供学习的示例](https://github.com/airbnb/lottie-ios/tree/master/Example)。

- 然后，快速读取一个由Bodymovin 生成的JSON文件进行播放。具体代码如下所示：

  ```objective-c
  LOTAnimationView *animation = [LOTAnimationView animationNamed:@"Lottie"];
  [self.view addSubview:animation];
  [animation playWithCompletion:^(BOOL animationFinished) {
    // 动画完成后需要处理的事情
  }];
  ```

- 利用 Lottie 的动画进度控制能力，还可以完成手势与动效同步的问题。动画进度控制是 LOTAnimationView 的 animationProgress 属性，设置属性的示例代码如下：

  ```objective-c
  CGPoint translation = [gesture getTranslationInView:self.view];
  CGFloat progress = translation.y / self.view.bounds.size.height;
  animationView.animationProgress = progress;
  ```

- Lottie 还带有一个 UIViewController animation-controller，可以自定义页面切换的过场动画，示例代码如下：

  ```objective-c
  #pragma mark -- 定制转场动画
  
  // 代理返回推出控制器的动画
  - (id<UIViewControllerAnimatedTransitioning>)animationControllerForPresentedController:(UIViewController *)presented presentingController:(UIViewController *)presenting sourceController:(UIViewController *)source {
    LOTAnimationTransitionController *animationController = [[LOTAnimationTransitionController alloc] initWithAnimationNamed:@"vcTransition1" fromLayerNamed:@"outLayer" toLayerNamed:@"inLayer" applyAnimationTransform:NO];
    return animationController;
  }
  
  // 代理返回退出控制器的动画
  - (id<UIViewControllerAnimatedTransitioning>)animationControllerForDismissedController:(UIViewController *)dismissed {
    LOTAnimationTransitionController *animationController = [[LOTAnimationTransitionController alloc] initWithAnimationNamed:@"vcTransition2" fromLayerNamed:@"outLayer" toLayerNamed:@"inLayer" applyAnimationTransform:NO];
    return animationController;
  }
  ```

- Lottie 在运行期间提供接口和协议来更改动画，有动画数据搜索接口 LOTKeyPath，以及设置动画数据的协议 LOTValueDelegate。详细的说明和使用示例代码，你可以参看[官方 iOS 教程](http://airbnb.io/lottie/#/ios)。

## 多平台支持

Lottie 支持多平台，除了 支持[iOS](https://github.com/airbnb/lottie-ios)，还支持 [Android](https://github.com/airbnb/lottie-android) 、[React Native](https://github.com/react-native-community/lottie-react-native)和[Flutter](https://github.com/simolus3/fluttie)。除了官方维护的这些平台外，Lottie还支持[Windows](https://github.com/windows-toolkit/Lottie-Windows)、[Qt](https://blog.qt.io/blog/2019/03/08/announcing-qtlottie/)、[Skia](https://skia.org/user/modules/skottie) 。陈卿还实现了 [React](https://github.com/chenqingspring/react-lottie)、[Vue](https://github.com/chenqingspring/vue-lottie)和[Angular](https://github.com/chenqingspring/ng-lottie)对 Lottie的支持，并已将代码放到了GitHub上。

## 实现原理

**[Lottie iOS](https://github.com/airbnb/lottie-ios)在 iOS 内做的事情就是将 After Effects 编辑的动画内容，通过JSON文件这个中间媒介，一一映射到 iOS 的 LayerModel、Keyframe、ShapeItem、DashElement、Marker、Mask、Transform 这些类的属性中并保存了下来，接下来再通过 CoreAnimation 进行渲染。**

- Lottie iOS 使用系统自带的 Codable协议来解析JSON文件，这样就可以享受系统升级带来性能提升的便利，比如 ShapeItem 这个类设计如下：

  ```swift
  // Shape Layer
  class ShapeItem: Codable {
    
    /// shape 的名字
    let name: String
    
    /// shape 的类型
    let type: ShapeType
  
    // 和 json 中字符映射
    private enum CodingKeys : String, CodingKey {
      case name = "nm"
      case type = "ty"
    }
    // 初始化
    required init(from decoder: Decoder) throws {
      let container = try decoder.container(keyedBy: ShapeItem.CodingKeys.self)
      self.name = try container.decodeIfPresent(String.self, forKey: .name) ?? "Layer"
      self.type = try container.decode(ShapeType.self, forKey: .type)
    }
  
  }
  ```

  ShapeItem 有两个属性，映射到JSON的字符键值是 nm 和 ty，分别代表 shape 的名字和类型。

- Bodymovin 生成的JSON代码：

  ```json
  {
     "ty":"st",
     "fillEnabled":true,
     "c":{
        "k":[
           {
              "i":{
                 "x":[
                    0.833
                 ],
                 "y":[
                    0.833
                 ]
              },
              "o":{
                 "x":[
                    0.167
                 ],
                 "y":[
                    0.167
                 ]
              },
              "n":[
                 "0p833_0p833_0p167_0p167"
              ],
              "t":22,
              "s":[
                 0,
                 0.65,
                 0.6,
                 1
              ],
              "e":[
                 0.76,
                 0.76,
                 0.76,
                 1
              ]
           },
           {
              "t":36
           }
        ]
     },
     "o":{
        "k":100
     },
     "w":{
        "k":3
     },
     "lc":2,
     "lj":2,
     "nm":"Stroke 1",
     "mn":"ADBE Vector Graphic - Stroke"
  }
  ```

  在这段JSON代码中，nm 键对应的值是 Stroke 1，ty 键对应的值是 st。那我们再来看看，**st 是什么类型。**

- ShapeType 是个枚举类型，它的定义如下：

  ```swift
  enum ShapeType: String, Codable {
    case ellipse = "el"
    case fill = "fl"
    case gradientFill = "gf"
    case group = "gr"
    case gradientStroke = "gs"
    case merge = "mm"
    case rectangle = "rc"
    case repeater = "rp"
    case round = "rd"
    case shape = "sh"
    case star = "sr"
    case stroke = "st"
    case trim = "tm"
    case transform = "tr"
  }
  ```

  Lottie 就是通过这种方式，定义了一系列的类结构，可以将JSON数据全部映射过来。所有映射用的类都放在 Lottie 的 Model 目录下。使用 CoreAnimation 渲染的相关代码都在 NodeRenderSystem 目录下，比如前面举例的 Stoke。

- 在渲染前会生成一个节点，实现在 StrokeNode.swift 里，然后对 StokeNode 这个节点渲染的逻辑在 StrokeRenderer.swift 里。核心代码如下：

  ```swift
  // 设置 Context
  func setupForStroke(_ inContext: CGContext) {
    inContext.setLineWidth(width) // 行宽
    inContext.setMiterLimit(miterLimit)
    inContext.setLineCap(lineCap.cgLineCap) // 行间隔
    inContext.setLineJoin(lineJoin.cgLineJoin)
  	// 设置线条样式
    if let dashPhase = dashPhase, let lengths = dashLengths {
      inContext.setLineDash(phase: dashPhase, lengths: lengths)
    } else {
      inContext.setLineDash(phase: 0, lengths: [])
    }
  }
  
  // 渲染
  func render(_ inContext: CGContext) {
    guard inContext.path != nil && inContext.path!.isEmpty == false else {
      return
    }
    guard let color = color else { return }
    hasUpdate = false
    setupForStroke(inContext)
    inContext.setAlpha(opacity) // 设置透明度
    inContext.setStrokeColor(color) // 设置颜色
    inContext.strokePath()
  }
  ```

  