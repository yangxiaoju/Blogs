# 关于 iOS 中的绘图和打印

本文档涵盖三个相关主题：

- 绘制自定义 UI 视图。自定义 UI 视图允许你绘制无法使用标准 UI 元素轻松绘制的内容。例如，绘图程序可能会为用户的绘图使用自定义视图，或者街机游戏可能会使用自定义视图来绘制精灵。
- 绘制到屏幕外的位图和 PDF 内容。无论你是打算稍后显示图像，将它们导出到文件，还是将图像打印到启用 AirPrint 的打印机，屏幕外绘图都可以在不中断用户工作流程的情况下执行此操作。
- 为你的应用添加 AirPrint 支持。iOS 打印系统可让你以不同方式绘制内容以适合页面。

图 I-1 你可以将自定义视图与标准视图结合使用，甚至可以在屏幕外绘制内容。

![](https://developer.apple.com/library/archive/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/Art/UI_overlay_2x.png)

## 概览

iOS 原生图形系统结合了三种主要技术：UIKit，Core Graphics 和 Core Animation。UIKit 在这些视图中提供视图和一些高级绘图功能，Core Graphics 在 UIKit 视图中提供额外的（低级）绘图支持，Core Animation 提供了将变换和动画应用于 UIKit 视图的功能。Core Animation 还负责视图合成。

### 自定义 UI 视图允许更大的绘图灵活性

本文档介绍如何使用原生绘图技术绘制自定义 UI 视图。这些技术包括 Core Graphics 和 UIKit 框架，支持 2D 绘图。

在考虑使用自定义 UI 视图之前，你应确保确实需要这样做。原生绘图适用于处理更复杂的 2D 布局需求。但是，由于自定义视图是处理器密集型的，因此应限制使用原生绘图技术执行的绘制量。

作为自定义绘图的替代方案，iOS 应用程序可以通过其他几种方式在屏幕上绘制内容。

- 使用标准（内置）视图。标准视图允许你绘制常见的用户界面基元，包括列表，集合，alerts，图像，进度条，表格等，而无需自己明确绘制任何内容。使用内置视图不仅可以确保 iOS 应用程序之间的一致用户体验，还可以节省编程工作量。如果内置视图符合你的需求，你应该阅读 View Programming Guide for iOS。
- 使用 Core Animation layers。核心动画允许你使用动画和变换创建复杂的分层 2D 视图。Core Animation 是动画标准视图或以复杂方式组合视图以呈现深度幻觉的不错选择，并且可以与本文档中描述的自定义绘制视图结合使用。要了解有关 Core Animation 的更多信息，请阅读 Core Animation Overview。
- 在 GLKit 视图或自定义视图中使用 OpenGL ES。OpenGL ES 框架提供了一组开放标准图形库，主要面向游戏开发或需要高帧率的应用程序，如虚拟原型应用程序和机械和建筑设计应用程序。它符合 OpenGL ES 2.0 和 OpenGL ES v1.1 规范。要了解有关 OpenGL 绘图的更多信息，请阅读 OpenGL ES 编程指南。
- 使用网页内容。UIWebView 类允许你在 iOS 应用程序中显示基于 Web 的用户界面。要了解有关在 Web 视图中显示 Web 内容的更多信息，请阅读 Using UIWebView to display select document types 和 UIWebView Class Reference。

根据你创建的应用程序类型，可能会使用很少或不使用自定义绘图代码。虽然沉浸式应用程序通常广泛使用自定义绘图代码，但实用程序和生产力应用程序通常可以使用标准视图和控件来显示其内容。

自定义绘图代码的使用应限于你显示的内容需要动态更改的情况。例如，绘图应用程序通常需要使用自定义绘图代码来跟踪用户的绘图命令，并且街机风格的游戏可能需要不断更新屏幕以反映不断变化的游戏环境。在这些情况下，你应该选择适当的绘图技术并创建自定义视图类来处理事件并适当地更新显示。

另一方面，如果应用程序界面的大部分是固定的，你可以提前将界面渲染为一个或多个图像文件，并使用 UIImageView 类在运行时显示这些图像。你可以根据需要将图像视图与其他内容分层以构建界面。你还可以使用 UILabel 类显示可配置文本，并包括按钮或其他控件以提供交互性。例如，通常可以用很少或没有自定义绘图代码来创建棋盘游戏的电子版本。

因为自定义视图通常是处理器密集型的（GPU 的帮助较少），如果你可以使用标准视图执行所需操作，则应始终这样做。此外，你应该使自定义视图尽可能小，仅包含你无法以任何其他方式绘制的内容，使用标准视图用于其他所有内容。如果需要将标准 UI 元素与自定义绘图结合使用，请考虑使用 Core Animation 图层将自定义视图与标准视图叠加，以便尽可能少地绘制。

#### 一些关键概念以原生技术为基础

使用 UIKit 和 Core Graphics 绘制内容时，除了视图绘制周期外，你还应该熟悉一些概念。

- 对于 drawRect: 方法，UIKit 创建用于渲染到显示的图形上下文。此图形上下文包含绘图系统执行绘图命令所需的信息，包括填充和描边颜色，字体，剪切区域和线宽等属性。你还可以为位图图像和 PDF 内容创建和绘制自定义图形上下文。
- UIKit 有一个默认坐标系，其中绘图的原点位于视图的左上角; 正值向下延伸到该原点的右侧。你可以通过修改当前变换矩阵来更改默认坐标系相对于基础视图或窗口的大小，方向和位置，该矩阵将视图的坐标空间映射到设备屏幕。
- 在 iOS 中，测量点中距离的逻辑坐标空间不等于以像素为单位测量的设备坐标空间。为了获得更高的精度，点以浮点值表示。

> 相关章节：iOS Drawing Concepts

#### UIKit，Core Graphics 和 Core Animation 为你的应用程序提供了许多绘图工具

UIKit 和 Core Graphics 具有许多互补的图形功能，包括图形上下文，Bézier 路径，图像，位图，透明层，颜色，字体，PDF 内容以及绘图矩形和剪切区域。此外，Core Graphics 具有与线属性，颜色空间，图案颜色，渐变，阴影和图像蒙版相关的功能。Core Animation 框架使你可以通过操纵和显示使用其他技术创建的内容来创建流畅的动画。

> 相关章节：iOS Drawing Concepts, Drawing Shapes Using Bézier Paths, Drawing and Creating Images, Generating PDF Content

### 应用可以绘制到屏幕外位图或 PDF

应用程序在屏幕外绘制内容通常很有用：

- 在缩小照片以进行上传，将内容渲染到图像文件以用于存储目的时，或者使用 Core Graphics 生成用于显示的复杂图像时，通常使用屏幕外位图上下文。
- 在绘制用户生成的内容以进行打印时，通常会使用屏外 PDF 上下文。

创建屏幕外上下文后，你可以像在自定义视图的 drawRect: 方法中绘制一样绘制它。

> 相关章节：Drawing and Creating Images, Generating PDF Content

### 应用程序有一系列打印内容选项

从 iOS 4.2 开始，应用程序可以使用 AirPrint 以无线方式将内容打印到支持的打印机。在组装打印作业时，他们有三种方法可以为 UIKit 提供要打印的内容：

- 他们可以为框架提供一个或多个可直接打印的对象; 这些对象需要最少的 app 参与。这些是包含或引用图像数据或 PDF 内容的 NSData，NSURL，UIImage 或 ALAsset 类的实例。
- 他们可以将打印格式化程序分配给打印作业。打印格式化程序是可以在多个页面上布置特定类型（例如纯文本或 HTML）的内容的对象。
- 他们可以将页面渲染器分配给打印作业。页面渲染器通常是 UIPrintPageRenderer 的自定义子类的实例，用于绘制要部分或全部打印的内容。页面渲染器可以使用一个或多个打印格式化程序来帮助它绘制和格式化其可打印内容。

> 相关章节：Printin

#### 你可以轻松更新高分辨率屏幕的应用程序

某些 iOS 设备具有高分辨率屏幕，因此你的应用必须准备好在这些设备和具有较低分辨率屏幕的设备上运行。iOS 负责处理不同分辨率所需的大部分工作，但你的应用必须完成其余的工作。你的任务包括提供特别命名的高分辨率图像，并修改与图层和图像相关的代码，以考虑当前的比例因子。

> 相关附录：Supporting High-Resolution Screens In Views

## 相关阅读

有关打印的完整示例，请参阅 PrintPhoto: Using the Printing API with Photos，Sample Print Page Renderer, and UIKit Printing with UIPrintInteractionController 和 UIViewPrintFormatter sample code projects。