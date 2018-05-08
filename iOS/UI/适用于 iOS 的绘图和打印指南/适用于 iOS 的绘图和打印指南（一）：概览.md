# 适用于 iOS 的绘图和打印指南（一）：引言

## 关于 iOS 中的绘图和打印

本文档涉及三个相关主题：

- 绘制自定义 UI 视图。 自定义 UI 视图允许你绘制不易用标准 UI 元素绘制的内容。 例如，绘图程序可能会为用户的绘图使用自定义视图，或者街机游戏可能会使用自定义视图来绘制精灵。
- 绘制到屏幕外位图和 PDF 内容。 无论你计划稍后显示图像，将它们导出到文件，还是将图像打印到支持 AirPrint 的打印机，可以在不中断用户工作流程的情况下通过离屏绘图进行操作。
- 将 AirPrint 支持添加到你的应用程序。 iOS 打印系统可让你以不同的方式绘制内容以适应页面。

图 I-1 你可以将自定义视图与标准视图组合在一起，甚至可以在屏幕外画东西。
![](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/Art/UI_overlay_2x.png)

### 概览

iOS 本机图形系统结合了三种主要技术：UIKit，Core Graphics 和 Core Animation。 UIKit 在这些视图中提供了视图和一些高级绘图功能，Core Graphics 在 UIKit 视图内提供了额外的（低级别）绘图支持，Core Animation 可以将转换和动画应用到 UIKit 视图。 Core Animation 也负责视图合成。

#### 自定义 UI 视图允许更大的绘图灵活性

本文档介绍如何使用原生绘图技术绘制自定义用户界面视图。 这些技术包括 Core Graphics 和 UIKit 框架，支持 2D 绘图。

在考虑使用自定义 UI 视图之前，你应该确保你确实需要这样做。 原生绘图适合处理更复杂的 2D 布局需求。 但是，由于自定义视图是处理器密集型的，因此应该限制使用本机绘图技术执行的绘图数量。
 
作为自定义绘图的替代方案，iOS 应用程序可以通过其他几种方式在屏幕上绘制事物。

- 使用标准（内置）视图。 通过标准视图，你可以绘制常用的用户界面元素，包括 lists，collections，alerts，images，progress bars，tables 等，而无需自己明确绘制任何内容。 使用内置视图不仅可以确保 iOS 应用程序之间的一致用户体验，还可以节省编程工作量。 如果内置视图满足你的需求，则应阅读 [View Programming Guide for iOS](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/ViewPG_iPhoneOS/Introduction/Introduction.html#//apple_ref/doc/uid/TP40009503)。
- 使用 Core Animation 图层。 Core Animation 可让你使用动画和转换创建复杂的分层 2D 视图。 核心动画是标准视图动画的一个很好的选择，或者以复杂的方式组合视图来呈现深度错觉，并且可以与本文档中描述的自定义绘制视图相结合。 要了解有关 Core Animation 的更多信息，请阅读 Core Animation Overview。
- 在 GLKit 视图或自定义视图中使用 OpenGL ES。 OpenGL ES 框架提供了一套开放标准图形库，主要面向游戏开发或需要高帧速率的应用程序，如虚拟原型应用程序以及机械和建筑设计应用程序。 它符合 OpenGL ES 2.0 和 OpenGL ES v1.1 规范。 要了解更多关于 OpenGL 绘图的信息，请阅读 [OpenGL ES Programming Guide](https://developer.apple.com/library/content/documentation/3DDrawing/Conceptual/OpenGLES_ProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008793)。
- 使用网页内容。 [UIWebView](https://developer.apple.com/documentation/uikit/uiwebview) 类可让你在 iOS 应用程序中显示基于 Web 的用户界面。 要了解有关在 Web 视图中显示 Web 内容的更多信息，请阅读 [Using UIWebView to display select document types](https://developer.apple.com/library/content/qa/qa1630/_index.html#//apple_ref/doc/uid/DTS40008749) 和 [UIWebView Class Reference](https://developer.apple.com/documentation/uikit/uiwebview)。

根据你创建的应用程序类型，可能使用很少或不使用自定义绘图代码。 虽然沉浸式应用程序通常广泛使用自定义绘图代码，但实用程序和生产力应用程序通常可以使用标准视图和控件来显示其内容。

自定义绘图代码的使用应限于你显示的内容需要动态更改的情况。 例如，绘图应用程序通常需要使用自定义绘图代码来跟踪用户的绘图命令，而街机风格的游戏可能需要不断更新屏幕以反映不断变化的游戏环境。 在这些情况下，你应该选择适当的绘图技术并创建自定义视图类来处理事件并适当地更新显示。

另一方面，如果应用程序界面的大部分已修复，则可以预先将界面渲染为一个或多个图像文件，并在运行时使用 [UIImageView](https://developer.apple.com/documentation/uikit/uiimageview) 类显示这些图像。 你可以根据需要将 image views 与其他内容分层，以构建您的界面。 你也可以使用 [UILabel](https://developer.apple.com/documentation/uikit/uilabel) 类来显示可配置的文本，并包含按钮或其他控件以提供交互性。 例如，一个棋盘游戏的电子版通常可以用很少或没有定制的绘图代码来创建。

由于自定义视图通常处理器密集程度更高（GPU 的帮助较少），因此如果你可以使用标准视图执行所需操作，则应始终如此。 此外，你应该使自定义视图尽可能小，仅包含无法以其他方式绘制的内容，对其他所有内容使用标准视图。 如果你需要将标准 UI 元素与自定义绘图相结合，请考虑使用 Core Animation 图层将自定义视图与标准视图叠加，以便尽可能少地绘制。

#### 一些重要概念支撑着本机技术的绘制

当你使用 UIKit 和 Core Graphics 绘制内容时，除了视图绘制周期外，你还应该熟悉一些概念。

- 对于 [drawRect:](https://developer.apple.com/documentation/uikit/uiview/1622529-drawrect) 方法，UIKit 创建一个图形上下文用于渲染到显示器。 此图形上下文包含绘图系统执行绘图命令所需的信息，包括填充和笔触颜色，字体，剪辑区域和线宽等属性。 你还可以创建并绘制位图图像和 PDF 内容的自定义图形上下文。
- UIKit 有一个默认坐标系，其中绘图的原点位于视图的左上角; 正值向下和向右延伸。 你可以通过修改将视图的坐标空间映射到设备屏幕的当前转换矩阵来更改默认坐标系相对于基础视图或窗口的大小，方向和位置。
- 在 iOS 中，测量以点为单位的距离的逻辑坐标空间不等于以像素为单位的设备坐标空间。 为了更高的精度，点以浮点值表示。

> 相关章节：iOS 绘图概念

#### UIKit，Core Graphics 和 Core Animation 为你的应用程序提供许多绘图工具

UIKit 和 Core Graphics 有许多互补的图形功能，包括图形上下文，Bézier路径，图像，位图，透明图层，颜色，字体，PDF 内容以及绘图矩形和裁剪区域。 此外，Core Graphics 还具有与线条属性，颜色空间，图案颜色，渐变，阴影和图像蒙版相关的功能。 Core Animation 框架使你能够通过操纵和显示使用其他技术创建的内容来创建流畅的动画。

> 相关章节：iOS 绘图概念，使用 Bézier 路径绘制图形，绘制和创建图像，生成 PDF 内容

#### 应用可以绘制成离屏位图或 PDF

应用程序在屏幕上绘制内容通常很有用：

- 在缩放照片进行上传时，通常会使用离屏位图上下文，出于存储目的将内容呈现到图像文件中，或使用 Core Graphics 生成复杂图像进行显示。
- 在为了打印目的而绘制用户生成的内容时，经常使用离屏 PDF 上下文。

创建屏幕外上下文后，可以像绘制自定义视图的 [drawRect:](https://developer.apple.com/documentation/uikit/uiview/1622529-drawrect) 方法一样绘制屏幕上下文。

> 相关章节：绘制和创建图像，[生成 PDF 内容](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/GeneratingPDF/GeneratingPDF.html#//apple_ref/doc/uid/TP40010156-CH10-SW1)

#### 应用程序有一系列打印内容的选项

从 iOS 4.2 开始，应用程序可以使用 AirPrint 将内容无线打印到支持的打印机。 在组装一个打印作业时，他们有三种方式给 UIKit 打印内容：

- 他们可以为框架提供一个或多个可直接打印的对象; 这样的对象需要最少的应用程序参与。这些是包含或引用图像数据或 PDF 内容的 [NSData](https://developer.apple.com/documentation/foundation/nsdata)，[NSURL](https://developer.apple.com/documentation/foundation/nsurl)，[UIImage](https://developer.apple.com/documentation/uikit/uiimage) 或 [ALAsset](https://developer.apple.com/documentation/assetslibrary/alasset) 类的实例。
- 他们可以将打印格式化程序分配给打印作业。 打印格式化程序是可以将特定类型的内容（如纯文本或 HTML）布置在多个页面上的对象。
- 他们可以将页面渲染器分配给打印作业。 页面渲染器通常是 [UIPrintPageRenderer](https://developer.apple.com/documentation/uikit/uiprintpagerenderer) 的自定义子类的一个实例，用于绘制要部分或全部打印的内容。 页面渲染器可以使用一个或多个打印格式化程序来帮助其绘制和格式化其可打印内容。

> 相关章节：[打印](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/Printing/Printing.html#//apple_ref/doc/uid/TP40010156-CH12-SW1)

#### 很容易更新你的应用程序的高分辨率屏幕

某些 iOS 设备具有高分辨率屏幕，因此你的应用必须准备好在这些设备上以及屏幕分辨率较低的设备上运行。 iOS 处理处理不同分辨率所需的大部分工作，但您的应用必须完成剩下的工作。 你的任务包括提供特殊命名的高分辨率图像以及修改与图层和图像相关的代码，以考虑当前的比例因子。

> 相关附录：[在视图中支持高分辨率屏幕](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/SupportingHiResScreensInViews/SupportingHiResScreensInViews.html#//apple_ref/doc/uid/TP40010156-CH15-SW1)

### 参见

有关打印的完整示例，请参阅 [PrintPhoto: Using the Printing API with Photos](https://developer.apple.com/library/content/samplecode/PrintPhoto/Introduction/Intro.html#//apple_ref/doc/uid/DTS40010366)，[Sample Print Page Renderer](https://developer.apple.com/library/content/samplecode/Recipes_+_Printing/Introduction/Intro.html#//apple_ref/doc/uid/DTS40011098) 和 [UIKit Printing with UIPrintInteractionController and UIViewPrintFormatter](https://developer.apple.com/library/content/samplecode/PrintWebView/Introduction/Intro.html#//apple_ref/doc/uid/DTS40010311) 示例代码项目。

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~