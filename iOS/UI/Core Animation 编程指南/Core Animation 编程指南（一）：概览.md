# 关于 Core Animation

Core Animation 是 iOS 和 OS X 上可用的图形渲染和动画基础结构，用于为应用的视图和其他视觉元素设置动画。 使用 Core Animation，绘制动画的每个帧所需的大部分工作都是为你完成的。 你所要做的就是配置一些动画参数（如开始点和结束点）并告诉 Core Animation 开始。 Core Animation 完成剩下的工作，将大部分实际绘图工作交给板载图形硬件来加速渲染。 这种自动图形加速可实现高帧率和流畅的动画效果，而不会增加 CPU 的负担并减慢应用程序的运行速度。

如果你正在编写 iOS 应用程序，则无论你是否了解，都使用 Core Animation。 如果你正在编写 OS X 应用程序，则可以非常轻松地利用 Core Animation。 Core Animation 位于 AppKit 和 UIKit 之下，并紧密集成到 Cocoa 和 Cocoa Touch 的视图工作流中。 当然，Core Animation 也具有接口，可以扩展应用视图所显示的功能，并让你更好地控制应用的动画。

![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/ca_architecture_2x.png)

## 概览

你可能永远都不需要直接使用 Core Animation，但是当你这样做时，你应该了解 Core Animation 作为应用程序基础结构的一部分所扮演的角色。

### Core Animation 管理你的应用程序的内容

Core Animation 本身不是一个绘图系统。 它是用于在硬件中合成和操纵应用内容的基础设施。 这个基础架构的核心是 layer 对象，你可以使用它来管理和操作你的内容。 layer 将你的内容捕获到位图中，该位图可由图形硬件轻松操纵。 在大多数应用程序中，layers 用作管理视图内容的方式，但你也可以根据需要创建独立图层。

> 相关章节：[Core Animation 基础](https://github.com/yangxiaoju/Blogs/blob/master/iOS/UI/Core%20Animation%20%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97/Core%20Animation%20%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97%EF%BC%88%E4%BA%8C%EF%BC%89%EF%BC%9ACore%20Animation%20%E5%9F%BA%E7%A1%80.md)，[设置 layer 对象](https://github.com/yangxiaoju/Blogs/blob/master/iOS/UI/Core%20Animation%20%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97/Core%20Animation%20%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97%EF%BC%88%E4%B8%89%EF%BC%89%EF%BC%9A%E8%AE%BE%E7%BD%AE%E5%9B%BE%E5%B1%82%E5%AF%B9%E8%B1%A1.md)

### Layer 修改触发动画

大多数使用 Core Animation 创建的动画都涉及修改 layer 的属性。 与视图类似，layer 对象具有 bounds rectangle，position onscreen，opacity，transform 以及可以修改的许多其他可视化属性。 对于大多数这些属性，更改属性的值会导致创建隐式动画，从而将图层从旧值动画到新值。 你还可以在需要更多控制最终动画行为的情况下，明确地动画这些属性。

> 相关章节：[动画 layer 内容](https://github.com/yangxiaoju/Blogs/blob/master/iOS/UI/Core%20Animation%20%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97/Core%20Animation%20%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97%EF%BC%88%E5%9B%9B%EF%BC%89%EF%BC%9A%E5%8A%A8%E7%94%BB%E5%9B%BE%E5%B1%82%E5%86%85%E5%AE%B9.md)，[高级动画技巧](https://github.com/yangxiaoju/Blogs/blob/master/iOS/UI/Core%20Animation%20%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97/Core%20Animation%20%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97%EF%BC%88%E5%85%AD%EF%BC%89%EF%BC%9A%E9%AB%98%E7%BA%A7%E5%8A%A8%E7%94%BB%E6%8A%80%E5%B7%A7.md)，[layer 样式属性动画](https://github.com/yangxiaoju/Blogs/blob/master/iOS/UI/Core%20Animation%20%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97/Core%20Animation%20%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97%EF%BC%88%E4%B9%9D%EF%BC%89%EF%BC%9A%E9%99%84%E5%BD%95%20A%20%E5%9B%BE%E5%B1%82%E6%A0%B7%E5%BC%8F%E5%B1%9E%E6%80%A7%E5%8A%A8%E7%94%BB.md)，[动画属性](https://github.com/yangxiaoju/Blogs/blob/master/iOS/UI/Core%20Animation%20%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97/Core%20Animation%20%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97%EF%BC%88%E5%8D%81%EF%BC%89%EF%BC%9A%E9%99%84%E5%BD%95%20B%20%E5%8A%A8%E7%94%BB%E5%B1%9E%E6%80%A7.md)

### Layer 可以组织成层次结构

可以分层排列 layers 以创建父子关系。 Layers 的排列会影响他们以类似于视图的方式管理的视觉内容。 附加到视图的一组 layers 的层次结构反映了相应的视图层次结构。 你还可以将独立图层添加到 layer 层次结构中，以扩展应用程序的可视内容，而不仅仅是视图。

> 相关章节：[构建层次结构](https://github.com/yangxiaoju/Blogs/blob/master/iOS/UI/Core%20Animation%20%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97/Core%20Animation%20%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97%EF%BC%88%E4%BA%94%EF%BC%89%EF%BC%9A%E6%9E%84%E5%BB%BA%E5%B1%82%E6%AC%A1%E7%BB%93%E6%9E%84.md)

### 操作让你更改 Layers 的默认行为

隐式层动画是使用动作对象实现的，动作对象是实现预定义接口的通用对象。 Core Animation 使用动作对象来实现通常与图层关联的默认动画集合。 你可以创建自己的动作对象来实现自定义动画或使用它们来实现其他类型的行为。 然后将你的操作对象分配给图层的某个属性。 当该属性更改时，Core Animation 将检索你的操作对象并告诉它执行其操作。

> 相关章节：[更改 layer 的默认行为](https://github.com/yangxiaoju/Blogs/blob/master/iOS/UI/Core%20Animation%20%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97/Core%20Animation%20%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97%EF%BC%88%E4%B8%83%EF%BC%89%EF%BC%9A%E6%9B%B4%E6%94%B9%E5%9B%BE%E5%B1%82%E7%9A%84%E9%BB%98%E8%AE%A4%E8%A1%8C%E4%B8%BA.md)

## 如何使用此文档

本文档面向需要更多控制其应用程序动画的开发人员，或者希望使用 layers 来提高应用程序绘图性能的开发人员。 本文档还提供了有关 iOS 和 OS X 的 layers 和视图之间集成的信息。iOS 和 OS X 中 layers 和视图之间的集成不同，并且了解这些差异对于创建高效动画非常重要。

## 先决条件

你应该已经了解目标平台的视图体系结构并熟悉如何创建基于视图的动画。 如果不是，你应该阅读以下文件之一：
- 对于 iOS 应用程序，你应该了解 [iOS 视图编程指南](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/ViewPG_iPhoneOS/Introduction/Introduction.html#//apple_ref/doc/uid/TP40009503)中描述的视图体系结构。
- 对于 OS X 应用程序，你应该了解[视图编程指南](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CocoaViewsGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40002978)中介绍的视图体系结构。

## 还应该看看

有关如何使用 Core Animation 实现特定类型的动画的示例，请参阅 Core Animation Cookbook。

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~