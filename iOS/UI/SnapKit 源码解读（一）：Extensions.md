# SnapKit 源码解读（一）：Extensions

## 前言

iOS 开发中的布局方式，总体而言经过了三个时代。混沌初开之时，世间只有3.5英寸（iPhone 4、iPhone 4S），那个时候屏幕适配对于大多数 iOS 开发者来说并不是什么难题，用 frame 就能精确高效的定位。这之后，苹果发布了4英寸机型（iPhone 5、iPhone 5C、iPhone 5S），与此同时苹果也推出了 AutoresizingMask，用来协调子视图与父视图之间的关系。再之后，各种各样的 iPhone 和 iPad 纷纷面世，不仅仅是屏幕尺寸方面的差异，更有异形屏（iPhone X）。在此期间，苹果提出了 AutoLayout 技术，供开发者进行屏幕适配。

使用 `AutoLayout` 的方法也有两种——通过 `Interface Builder` 或者纯代码。前者一直是苹果官方文档里所鼓励的，原因是苹果从最初到现在，对于 iOS 应用的想法都是小而美的，在他们的认知里，一个 APP 应该提供尽可能小的功能集，这也是为为何苹果迄今为止官方推荐的架构仍然是 MVC，官方推荐的开发方式仍是以 `StoryBoard（Size Classes）`。但是在一些项目较大的公司，`StoryBoard` 的某些特性（导致应用包过大，减缓启动速度，合并代码困难）又是不能为人所容忍的，便有了纯代码来实现 `View` 层的一群开发者（比如我）。

如果你曾经用代码来实现 `AutoLayout`，你会发现苹果提供的 `API` 的繁琐程度令人发指，这也是 `SnapKit` 这类框架被发明的原因。`SnapKit` 是一种使 `iOS` 和 `OS X` 上的自动布局更加简单的 `DSL`。与亲哥 `Masonry`（两个框架都出自同一个团队之手） 不一样的是，`SnapKit` 更好的利用了 `Swift` 的一些语言特性，如果你是一个 `Swift` 开发者，那么更应该优选 `SnapKit`。

接下来，我们从 `SnapKit` 划分好的各个模块来学习一下，不仅是学习如何更好的写出一个框架，更多的是如何的写好 `Swift`。本篇文章将带领大家了解一下 `Extensions` 这个模块。 

### ConstraintView+Extensions

#### 条件编译

`Swift` 中没有宏定义的概念，很多依赖宏来施展的小技巧都不能实现了。但是 `Swift` 还是保留了一些基础的功能，来让我们控制编译流程和内容。例如：

```
#if os(iOS) || os(tvOS)
    import UIKit
#else
    import AppKit
#endif
```

`#if`、`#elseif`、`#else` 和 `endif` 这套编译标记在 `Swift` 中同样可用，同时 `Swift` 还提供了 `os()` 这样检测系统平台的函数，在这里则是根据不同的操作系统选择导入不同的框架。更多内容可以参阅这篇文章：[条件编译](http://swifter.tips/condition-compile/)。

#### typealias

`ConstraintView` 这个类本身也是通过条件编译来声明的，同时用到了 `typealias`

```
#if os(iOS) || os(tvOS)
    public typealias ConstraintView = UIView
#else
    public typealias ConstraintView = NSView
#endif
```

`typealias` 是一种给类增加别名等方法，它的一个用途是实现跨平台的能力。`ConstraintView` 在 `iOS` 和 `tvOS` 平台上表示 `UIView`，在其他平台上则表示 `NSView`。

#### @available

`@available` 用于函数、方法、类或协议的前面，表明平台和操作系统适用性。

````
@available(*, deprecated:3.0, message:"Use newer snp.* syntax.")
````

`*` 代表全平台，同时还支持一些额外的参数：`deprecated=版本号`：从指定平台某个版本开始过期该声明，`message=信息内容`：给出一些附加信息。更多用法可以参考这篇文章：[每周 Swift 社区问答：@available 和 #available](http://swift.gg/2016/04/13/swift-qa-2016-04-13/)

#### 命名空间

扩展之于 `Swift` 就好比分类之于 `Objective-C`，所以同样可能遇到方法名冲突的可能。在 `Objective-C` 中，我们通过给方法名加前缀来规避这一问题。但在 `Swift` 中，我们有更好的方法，例如：

```
public var snp: ConstraintViewDSL {
    return ConstraintViewDSL(view: self)
}
```

通过计算属性就可以做到避免拓展方法名冲突了，调用起来就像 `view.snp.xxx` 这样，方便好用。

### ConstraintLayoutGuide+Extensions

`ConstraintLayoutGuide` 是在不同的平台上对 `LayoutGuide` 的不同表示：

```
#if os(iOS) || os(tvOS)
    @available(iOS 9.0, *)
    public typealias ConstraintLayoutGuide = UILayoutGuide
#else
    @available(OSX 10.11, *)
    public typealias ConstraintLayoutGuide = NSLayoutGuide
#endif
```

`UILayoutGuide` 可以为我们生成一个虚拟的占位对象，辅助我们来进行自动布局。关于 `UILayoutGuide` 的常见用法，可以参考这篇文章：[是时候了解一下UILayoutGuide了](https://juejin.im/entry/586a8ee1ac502e00614a72a6)

### UILayoutSupport+Extensions

`ConstraintLayoutSupport` 是在不同的平台上对 `UILayoutSupport` 的不同表示：

```
#if os(iOS) || os(tvOS)
    @available(iOS 8.0, *)
    public typealias ConstraintLayoutSupport = UILayoutSupport
#else
    public class ConstraintLayoutSupport {}
#endif
```

不过有所不同的是 `APPKit` 框架内并没有对应的概念，所以在 `#else` 分支里则定义了一个 `ConstraintLayoutSupport` 类，这也是跨平台时的一种做法。

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~