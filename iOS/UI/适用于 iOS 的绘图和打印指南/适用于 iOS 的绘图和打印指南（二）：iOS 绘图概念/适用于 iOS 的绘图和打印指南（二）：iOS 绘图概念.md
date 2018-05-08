# iOS 绘图概念

高质量的图形是你应用用户界面的重要组成部分。 提供高质量的图形不仅使你的应用看起来不错，而且还使你的应用看起来像是系统其他部分的自然延伸。 iOS 提供了两个用于在系统中创建高质量图形的主要路径：OpenGL 或使用 Quartz，Core Animation 和 UIKit 的本机渲染。 本文档描述本地渲染。 （要了解 OpenGL 绘图，请参阅 [OpenGL ES Programming Guide](https://developer.apple.com/library/content/documentation/3DDrawing/Conceptual/OpenGLES_ProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008793)。

Quartz 是主要的绘图接口，为基于路径的绘图，反锯齿渲染，渐变填充图案，图像，颜色，坐标空间转换和 PDF 文档创建，显示和解析提供支持。 UIKit 为 line art，Quartz 图像和颜色操作提供 Objective-C 包装。 Core Animation 为许多 UIKit 视图属性中的动画变化提供了基础支持，也可用于实现自定义动画。

本章概述 iOS 应用程序的绘制过程，以及每种受支持绘图技术的特定绘图技术。 你还可以找到有关如何优化 iOS 平台的绘图代码的提示和指导。

> 重要提示：并非所有的 UIKit 类都是线程安全的。 请务必在除应用程序主线程以外的线程上执行绘图相关操作之前检查文档。

## UIKit 图形系统

在 iOS 中，无论是涉及 OpenGL，Quartz，UIKit 还是 Core Animation，都会在 UIView 类的实例或其子类的范围内进行绘制。 视图定义了绘图发生的部分屏幕。 如果你使用系统提供的视图，则自动为你处理此图。 但是，如果你定义自定义视图，则必须自己提供绘图代码。 如果你使用 Quartz，Core Animation 和 UIKit 进行绘制，则可以使用以下各节中描述的绘图概念。

除了直接绘制屏幕外，UIKit 还允许你绘制离屏位图和 PDF 图形上下文。 当你在离屏上下文中绘图时，你并未在视图中绘制，这意味着诸如视图绘制周期之类的概念不适用（除非你之后获取该图像并将其绘制在图像视图中或类似图像中）。

### 视图绘制周期

UIView 类的子类的基本绘图模型涉及按需更新内容。 [UIView](https://developer.apple.com/documentation/uikit/uiview) 类使更新过程更容易，更高效; 但是，通过收集你发起的更新请求并在最适当的时间将它们发送到你的绘图代码。

当第一次显示视图或需要重绘视图的一部分时，iOS 会通过调用视图的 [drawRect:](https://developer.apple.com/documentation/uikit/uiview/1622529-drawrect) 方法请求视图绘制其内容。

有几个动作可以触发视图更新：

- 移动或移除部分遮挡视图的其他视图
- 通过将其 [hidden](https://developer.apple.com/documentation/uikit/uiview/1622585-ishidden) 属性设置为 NO，使先前隐藏的视图再次可见
- 将视图从屏幕上滚动并返回到屏幕上
- 显式调用视图的 [setNeedsDisplay](https://developer.apple.com/documentation/uikit/uiview/1622437-setneedsdisplay) 或 [setNeedsDisplayInRect:](https://developer.apple.com/documentation/uikit/uiview/1622587-setneedsdisplayinrect) 方法

系统视图会自动重新绘制。 对于自定义视图，你必须重写 [drawRect:](https://developer.apple.com/documentation/uikit/uiview/1622529-drawrect) 方法并在其中执行所有绘图。 在 [drawRect:](https://developer.apple.com/documentation/uikit/uiview/1622529-drawrect) 方法内部，使用本机绘图技术来绘制形状，文本，图像，渐变或任何其他你想要的视觉内容。 当你的视图第一次可见时，iOS 将一个矩形传递给视图的 drawRect: 方法，该方法包含视图的整个可见区域。 在随后的调用中，矩形只包含实际需要重绘的部分视图。 为了获得最佳性能，你只应重新绘制受影响的内容。

在调用 [drawRect:](https://developer.apple.com/documentation/uikit/uiview/1622529-drawrect) 方法后，视图将自己标记为已更新，并等待新动作到达并触发另一个更新周期。 如果你的视图显示静态内容，那么你只需响应由滚动和其他视图引起的视图可见性更改。

但是，如果你想更改视图的内容，则必须告诉视图重新绘制其内容。 为此，请调用 [setNeedsDisplay](https://developer.apple.com/documentation/uikit/uiview/1622437-setneedsdisplay) 或 [setNeedsDisplayInRect:](https://developer.apple.com/documentation/uikit/uiview/1622587-setneedsdisplayinrect) 方法来触发更新。 例如，如果你每秒更新多次内容，则可能需要设置一个计时器来更新视图。 你也可以更新你的视图，以响应用户交互或在视图中创建新内容。

> 重要提示：不要自己调用视图的 drawRect：方法。 该方法应该仅在屏幕重绘期间由内置于 iOS 的代码调用。 在其他时候，不存在图形上下文，所以绘图是不可能的。 （图形上下文将在下一节中介绍。）

### 在 iOS 中协调系统和绘图

当应用程序在 iOS 中绘制某些东西时，它必须在由坐标系定义的二维空间中定位绘制的内容。 乍一看，这个概念可能看起来很简单，但事实并非如此。 在绘图时，iOS 中的应用程序有时必须处理不同的坐标系。

在 iOS 中，所有绘图都发生在图形上下文中。 从概念上讲，图形上下文是一个对象，描述应该在何处以及如何进行绘图，包括基本绘图属性，例如绘图时使用的颜色，剪贴区域，线宽和样式信息，字体信息，合成选项等等。

另外，如图1-1所示，每个图形上下文都有一个坐标系。 更确切地说，每个图形上下文都有三个坐标系：

- 绘图（用户）坐标系。 当你发出绘图命令时使用此坐标系。
- 视图坐标系（基本空间）。 该坐标系是相对于视图的固定坐标系。
- （物理）设备坐标系。 该坐标系表示物理屏幕上的像素。

图 1-1 绘图坐标，视图坐标和硬件坐标之间的关系
![](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/Art/coordinate_differences_2x.png)

iOS 的绘图框架创建用于绘制到特定目的地的图形上下文（屏幕，位图，PDF 内容等等），并且这些图形上下文为该目的地建立初始绘图坐标系。 该初始绘图坐标系被称为默认坐标系，并且是视图底层坐标系上的1：1映射。

每个视图还有一个当前转换矩阵（CTM），一个将当前绘图坐标系中的点映射到（固定）视图坐标系的数学矩阵。 应用程序可以修改此矩阵（如稍后所述）以更改未来绘图操作的行为。

iOS 的每个绘图框架都基于当前的图形上下文建立默认坐标系。 在 iOS 中，有两种主要类型的坐标系：

- 绘图操作的起点位于绘图区域的左上角的左上角坐标系（ULO），其正值向下和向右延伸。 UIKit 和 Core Animation 框架使用的默认坐标系统是基于 ULO 的。
- 左下原点坐标系（LLO），绘图操作的原点位于绘图区域的左下角，正值向上和向右延伸。 Core Graphics 框架使用的默认坐标系是基于 LLO 的。

这些坐标系如图1-2所示。

图 1-2 iOS 中的默认坐标系
![](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/Art/flipped_coordinates-2_2x.png)

注意：OS X 中的默认坐标系是基于 LLO 的。 虽然 Core Graphics 和 AppKit 框架的绘图函数和方法非常适合此默认坐标系，但 AppKit 为绘图坐标系提供程序化支持以使其具有左上角原点。

在调用视图的 [drawRect:](https://developer.apple.com/documentation/uikit/uiview/1622529-drawrect) 方法之前，UIKit 通过使图形上下文可用于绘制操作来建立用于绘制到屏幕的默认坐标系。 在视图的 drawRect：方法中，应用程序可以设置图形状态参数（如填充颜色）并绘制到当前图形上下文，而无需显式引用图形上下文。 这个隐含的图形上下文建立了 ULO 默认坐标系。

### 点与像素

在 iOS 中，你在绘图代码中指定的坐标与底层设备的像素之间存在区别。 当使用 Quartz，UIKit 和 Core Animation 等原生绘图技术时，绘图坐标空间和视图坐标空间都是逻辑坐标空间，距离以点为单位。 这些逻辑坐标系与系统框架使用的设备坐标空间分离，以管理屏幕上的像素。

系统会自动将视图坐标空间中的点映射到设备坐标空间中的像素，但该映射并不总是一对一。 这种行为导致一个重要的事实，你应该永远记住：

一点不一定对应于一个物理像素。

使用点（和逻辑坐标系）的目的是提供一个独立于设备的输出的一致大小。 对于大多数目的而言，点的实际大小是无关紧要的。 点的目标是提供一个相对一致的尺度，你可以在代码中使用该尺度来指定视图和渲染内容的大小和位置。 点如何实际映射到像素是由系统框架处理的细节。 例如，在具有高分辨率屏幕的设备上，宽度为一点的线条实际上可能会产生两条物理像素宽度的线条。 结果是，如果你在两台类似的设备上绘制相同的内容，并且其中只有一台具有高分辨率屏幕，则两台设备上的内容似乎大小相同。

> 注意：在 PDF 渲染和打印环境中，Core Graphics 使用行业标准的1点到1/72英寸的映射来定义“点”。

在 iOS 中，[UIScreen](https://developer.apple.com/documentation/uikit/uiscreen)，[UIView](https://developer.apple.com/documentation/uikit/uiview)，[UIImage](https://developer.apple.com/documentation/uikit/uiimage) 和 [CALayer](https://developer.apple.com/documentation/quartzcore/calayer) 类提供了一些属性来获取（以及在某些情况下设置）缩放因子，该因子描述了该特定对象的点和像素之间的关系。 例如，每个 UIKit 视图都有一个 [contentScaleFactor](https://developer.apple.com/documentation/uikit/uiview/1622657-contentscalefactor) 属性。 在标准分辨率屏幕上，比例因子通常为1.0。 在高分辨率屏幕上，比例因子通常为2.0。 未来，其他比例因子也是可能的。 （在版本4之前的iOS中，你应该假定比例因子为1.0。）

原生绘图技术（如 Core Graphics）将为你考虑当前的比例因子。 例如，如果其中一个视图实现了 [drawRect:](https://developer.apple.com/documentation/uikit/uiview/1622529-drawrect) 方法，则 UIKit 会自动将该视图的比例因子设置为屏幕的比例因子。 另外，UIKit 会自动修改绘图期间使用的任何图形上下文的当前变换矩阵，以考虑视图的比例因子。 因此，你在 drawRect：方法中绘制的任何内容都会针对底层设备的屏幕进行适当缩放。

由于这种自动映射，在编写绘图代码时，像素通常并不重要。 但是，有时你可能需要根据点映射到像素的方式更改应用的绘制行为 - 在具有高分辨率屏幕的设备上下载更高分辨率的图像，或避免在低分辨率屏幕上绘图时缩放人为因素 ， 例如。

在 iOS 中，当你在屏幕上绘制东西时，图形子系统会使用称为抗锯齿的技术来在较低分辨率的屏幕上逼近较高分辨率的图像。 解释这种技术的最好方法就是举例。 当你在纯白色背景上绘制黑色垂直线时，如果该线恰好落在像素上，则会显示为白色字段中的一系列黑色像素。 但是，如果它恰好出现在两个像素之间，它会并排显示为两个灰色像素，如图1-3所示。

图 1-3 以一个整数点值为中心的一点线
![](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/Art/pixel_alignment_2x.png)

由整数点定义的位置落在像素之间的中点。例如，如果从（1.0,1.0）到（1.0,10.0）绘制一个像素宽的垂直线，则会出现模糊的灰线。如果绘制一个两像素宽的线，则会得到一条纯黑线，因为它完全覆盖了两个像素（指定点的两侧各一个）。通常情况下，宽度为奇数个物理像素的行看起来比以偶数个物理像素测量宽度的行柔和，除非你调整其位置以使它们完全覆盖像素。

图 1-4 标准和视网膜显示器上单点宽线的外观

![](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/Art/regular_vs_retina_2x.png)

在高分辨率显示器上（比例因子为2.0），一点宽的线根本不是反锯齿，因为它占用了两个完整像素（从-0.5到+0.5）。 要绘制只包含单个物理像素的线条，你需要使其厚度为0.5个点，并将其位置偏移0.25个点。 图1-4显示了两种屏幕之间的比较。

当然，根据比例因子改变绘图特性可能会产生意想不到的后果。 一个1像素宽的线可能在某些设备上看起来不错，但是在高分辨率设备上可能非常薄，以至于难以清楚地看到。 决定是否做出这样的改变取决于你。

### 获取图形上下文

大多数情况下，图形上下文都是为你配置的。 每个视图对象都会自动创建一个图形上下文，以便你的代码可以在调用自定义 [drawRect:](https://developer.apple.com/documentation/uikit/uiview/1622529-drawrect) 方法时立即开始绘制。 作为此配置的一部分，底层 [UIView](https://developer.apple.com/documentation/uikit/uiview) 类为当前绘图环境创建图形上下文（[CGContextRef](https://developer.apple.com/documentation/coregraphics/cgcontext) 不透明类型）。

如果要绘制除视图之外的其他位置（例如，要在 PDF 或位图文件中捕获一系列绘图操作），或者需要调用需要上下文对象的 Core Graphics 函数，则必须执行其他步骤获取图形上下文对象。 下面的部分解释了如何。

有关图形上下文，修改图形状态信息以及使用图形上下文创建自定义内容的更多信息，请参见 [Quartz 2D Programming Guide](https://developer.apple.com/library/content/documentation/GraphicsImaging/Conceptual/drawingwithquartz2d/Introduction/Introduction.html#//apple_ref/doc/uid/TP30001066)。 有关与图形上下文一起使用的函数列表，请参阅 [CGContext Reference](https://developer.apple.com/documentation/coregraphics/cgcontext)，[CGBitmapContext Reference](https://developer.apple.com/documentation/coregraphics/cgbitmapcontext) 和 [CGPDFContext Reference](https://developer.apple.com/documentation/coregraphics/cgpdfcontext)。

#### 绘制到屏幕上

如果使用 Core Graphics 函数绘制视图，无论是在 [drawRect:](https://developer.apple.com/documentation/uikit/uiview/1622529-drawrect) 方法还是其他位置，都需要绘制图形上下文。 （许多这些函数的第一个参数必须是 [CGContextRef](https://developer.apple.com/documentation/coregraphics/cgcontext) 对象。）你可以调用函数 [UIGraphicsGetCurrentContext](https://developer.apple.com/documentation/uikit/1623918-uigraphicsgetcurrentcontext) 来获取在 `drawRect:` 中隐含的相同图形上下文的显式版本。 因为它是相同的图形上下文，所以绘图功能还应参考 ULO 默认坐标系。

如果要使用 Core Graphics 函数绘制 UIKit 视图，则应使用 UIKit 的 ULO 坐标系进行绘制操作。 或者，你可以对 CTM 应用翻转转换，然后使用 Core Graphics 原生 LLO 坐标系在 UIKit 视图中绘制对象。 [翻转默认坐标系](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/GraphicsDrawingOverview/GraphicsDrawingOverview.html#//apple_ref/doc/uid/TP40010156-CH14-SW26)详细讨论翻转变换。

`UIGraphicsGetCurrentContext` 函数总是返回当前有效的图形上下文。 例如，如果你创建一个 PDF 上下文，然后调用 [UIGraphicsGetCurrentContext](https://developer.apple.com/documentation/uikit/1623918-uigraphicsgetcurrentcontext)，则会收到该 PDF 上下文。 如果使用 Core Graphics 函数绘制视图，则必须使用 `UIGraphicsGetCurrentContext` 返回的图形上下文。

> 注意：`UIPrintPageRenderer` 类声明了几种绘制可打印内容的方法。 以类似于 `drawRect:` 的方式，UIKit 为这些方法的实现安装隐式图形上下文。 此图形上下文建立 ULO 默认坐标系。

#### 绘制到位图上下文和 PDF 上下文

UIKit 提供了在位图图形上下文中渲染图像以及通过在 PDF 图形上下文中绘制生成 PDF 内容的功能。 这两种方法都要求你首先分别调用一个创建图形上下文的函数 - 位图上下文或 PDF 上下文。 返回的对象用作后续绘制和状态设置调用的当前（和隐式）图形上下文。 当你在上下文中完成绘图时，你可以调用另一个函数来关闭上下文。

UIKit 提供的位图上下文和 PDF 上下文都建立了 ULO 默认坐标系。 核心图形具有相应的功能，用于在位图图形上下文中绘制并用于绘制 PDF 图形上下文。 但是，应用程序通过 Core Graphics 直接创建的上下文建立了 LLO 默认坐标系。

> 注意：在 iOS 中，建议你使用 UIKit 函数绘制位图上下文和 PDF 上下文。 但是，如果确实使用 Core Graphics 替代方法并打算显示渲染结果，则必须调整代码以补偿默认坐标系中的差异。 请参阅[翻转默认坐标系](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/GraphicsDrawingOverview/GraphicsDrawingOverview.html#//apple_ref/doc/uid/TP40010156-CH14-SW26)以获取更多信息。

有关详细信息，请参阅[绘制和创建图像](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/HandlingImages/Images.html#//apple_ref/doc/uid/TP40010156-CH13-SW1)（用于绘制位图上下文）和[生成 PDF 内容](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/GeneratingPDF/GeneratingPDF.html#//apple_ref/doc/uid/TP40010156-CH10-SW1)（用于绘制为 PDF 上下文）。

### 颜色和颜色空间

iOS 支持 Quartz 提供的全部色彩空间; 但是，大多数应用程序只需要 RGB 颜色空间。 由于 iOS 设计为在嵌入式硬件上运行并在屏幕上显示图形，因此 RGB 色彩空间是最适合使用的。

[UIColor](https://developer.apple.com/documentation/uikit/uicolor) 对象提供了使用 RGB，HSB 和灰度值指定颜色值的便捷方法。 以这种方式创建颜色时，你永远不需要指定颜色空间。 它由 `UIColor` 对象自动确定。

你还可以使用 Core Graphics 框架中的 [CGContextSetRGBStrokeColor](https://developer.apple.com/documentation/coregraphics/1456378-cgcontextsetrgbstrokecolor) 和 [CGContextSetRGBFillColor](https://developer.apple.com/documentation/coregraphics/1455624-cgcontextsetrgbfillcolor) 函数来创建和设置颜色。 尽管 Core Graphics 框架支持使用其他颜色空间创建颜色，并且可以创建自定义颜色空间，但不推荐在绘图代码中使用这些颜色。 你的绘图代码应该总是使用RGB颜色。

## 用 Quartz 和 UIKit 绘图

Quartz 是 iOS 中原生绘图技术的通用名称。 Core Graphics 框架是 Quartz 的核心，也是你用于绘制内容的主要界面。 该框架提供了用于处理以下内容的数据类型和函数：

- 图形上下文
- 路径
- 图像和位图
- 透明图层
- 颜色，图案颜色和颜色空间
- 渐变和阴影
- 字体
- PDF 内容

UIKit 基于 Quartz 的基本特性，为图形相关操作提供了一组重要的类。 UIKit 图形类并不是一套全面的绘图工具，Core Graphics 已经提供了这些。 相反，他们为其他 UIKit 类提供绘图支持。 UIKit 支持包括以下类和功能：

- [UIImage](https://developer.apple.com/documentation/uikit/uiimage)，它实现了用于显示图像的不可变类
- [UIColor](https://developer.apple.com/documentation/uikit/uiimage)，为设备颜色提供基本支持
- [UIFont](https://developer.apple.com/documentation/uikit/uifont)，为需要它的类提供字体信息
- [UIScreen](https://developer.apple.com/documentation/uikit/uiscreen)，提供有关屏幕的基本信息
- [UIBezierPath](https://developer.apple.com/documentation/uikit/uibezierpath)，它可以让你的应用绘制线条，弧形，椭圆形和其他形状。
- 用于生成 `UIImage` 对象的 JPEG 或 PNG 表示的函数
- 用于绘制位图图形上下文的函数
- 通过绘制到 PDF 图形上下文来生成 PDF 数据的功能
- 用于绘制矩形和剪切绘图区域的函数
- 用于更改和获取当前图形上下文的函数

有关构成 UIKit 的类和方法的信息，请参阅 [UIKit Framework Reference](https://developer.apple.com/documentation/uikit)。 有关构成 Core Graphics 框架的不透明类型和函数的更多信息，请参阅 Core Graphics Framework Reference。

### 配置图形上下文

在调用 drawRect: 方法之前，视图对象会创建一个图形上下文并将其设置为当前上下文。 此上下文仅存在于 drawRect: 调用的生命周期中。 你可以通过调用 UIGraphicsGetCurrentContext 函数来检索指向此图形上下文的指针。 该函数返回对 CGContextRef 类型的引用，该类型将传递给 Core Graphics 函数以修改当前的图形状态。 表1-1列出了用于设置图形状态不同方面的主要功能。 有关函数的完整列表，请参阅 CGContext 参考。 此表还列出了 UIKit 替代品的存在位置。

表1-1用于修改图形状态的核心图形函数
![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/UI/%E9%80%82%E7%94%A8%E4%BA%8E%20iOS%20%E7%9A%84%E7%BB%98%E5%9B%BE%E5%92%8C%E6%89%93%E5%8D%B0%E6%8C%87%E5%8D%97/%E9%80%82%E7%94%A8%E4%BA%8E%20iOS%20%E7%9A%84%E7%BB%98%E5%9B%BE%E5%92%8C%E6%89%93%E5%8D%B0%E6%8C%87%E5%8D%97%EF%BC%88%E4%BA%8C%EF%BC%89%EF%BC%9AiOS%20%E7%BB%98%E5%9B%BE%E6%A6%82%E5%BF%B5/%E8%A1%A81-1%E7%94%A8%E4%BA%8E%E4%BF%AE%E6%94%B9%E5%9B%BE%E5%BD%A2%E7%8A%B6%E6%80%81%E7%9A%84%E6%A0%B8%E5%BF%83%E5%9B%BE%E5%BD%A2%E5%87%BD%E6%95%B0.png?raw=true)

图形上下文包含一堆保存的图形状态。 当 Quartz 创建一个图形上下文时，堆栈是空的。 使用 [CGContextSaveGState](https://developer.apple.com/documentation/coregraphics/1456156-cgcontextsavegstate) 函数将当前图形状态的副本压入堆栈。 此后，对图形状态所做的修改会影响后续的绘制操作，但不会影响存储在堆栈上的副本。 完成修改后，可以使用 [CGContextRestoreGState](https://developer.apple.com/documentation/coregraphics/cgcontext/1455391-restoregstate) 函数从堆栈顶部弹出保存的状态，以返回到以前的图形状态。 以这种方式推动和弹出图形状态是返回到先前状态的快速方式，并且消除了逐个撤消每个状态改变的需要。 它也是将状态的某些方面（如剪切路径）恢复到其原始设置的唯一方法。

有关图形上下文的一般信息并使用它们来配置绘图环境，请参阅 [Quartz 2D 编程指南](https://developer.apple.com/library/content/documentation/GraphicsImaging/Conceptual/drawingwithquartz2d/Introduction/Introduction.html#//apple_ref/doc/uid/TP30001066)中的[图形上下文](https://developer.apple.com/library/content/documentation/GraphicsImaging/Conceptual/drawingwithquartz2d/dq_context/dq_context.html#//apple_ref/doc/uid/TP30001066-CH203)。

### 创建和绘制路径

路径是从一系列线和贝塞尔曲线创建的基于矢量的形状。 UIKit 包括 [UIRectFrame](https://developer.apple.com/documentation/uikit/1623926-uirectframe) 和 [UIRectFill](https://developer.apple.com/documentation/uikit/1623932-uirectfill) 函数（等等），用于在视图中绘制矩形等简单路径。核心图形还包括用于创建简单路径（如矩形和椭圆）的便利功能。

对于更复杂的路径，你必须使用 UIKit 的 [UIBezierPath](https://developer.apple.com/documentation/uikit/uibezierpath) 类自己创建路径，或者使用在 Core Graphics 框架中对 [CGPathRef](https://developer.apple.com/documentation/coregraphics/cgpath) 透明类型进行操作的函数。尽管你可以使用 API​ ​构建没有图形上下文的路径，但路径中的点仍然必须引用当前坐标系（其具有 ULO 或 LLO 方向），并且仍然需要图形上下文来实际渲染路径。

绘制路径时，你必须设置当前上下文。此上下文可以是自定义视图的上下文（在 drawRect:)中，位图上下文或 PDF 上下文中。坐标系决定路径的渲染方式。  [UIBezierPath](https://developer.apple.com/documentation/uikit/uibezierpath) 假定 ULO 坐标系统。因此，如果视图翻转（使用 LLO 坐标），则生成的形状可能与预期不同。为了获得最佳效果，你应始终指定相对于用于渲染的图形上下文的当前坐标系的原点的点。

> 注意：即使遵循此“规则”，弧也是需要额外工作的路径的一个方面。 如果使用在 ULO 坐标系中定位点的 Core Graphic 函数创建路径，然后在 UIKit 视图中渲染路径，则弧“指向”的方向会有所不同。 有关此主题的更多信息，请参见[使用不同坐标系绘制的副作用](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/GraphicsDrawingOverview/GraphicsDrawingOverview.html#//apple_ref/doc/uid/TP40010156-CH14-SW8)。

要在 iOS 中创建路径，建议你使用 [UIBezierPath](https://developer.apple.com/documentation/uikit/uibezierpath) 而不是 CGPath 函数，除非你需要某些仅 Core Graphics 提供的功能，例如向路径添加椭圆。 有关在 UIKit 中创建和渲染路径的更多信息，请参见[使用 Bézier 路径绘制图形](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/BezierPaths/BezierPaths.html#//apple_ref/doc/uid/TP40010156-CH11-SW1)。

有关使用 `UIBezierPath` 绘制路径的信息，请参阅[使用 Bézier 路径绘制图形](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/BezierPaths/BezierPaths.html#//apple_ref/doc/uid/TP40010156-CH11-SW1)。 有关如何使用 Core Graphics 绘制路径的信息，包括有关如何为复杂路径元素指定点的信息，请参阅 [Quartz 2D编程指南](https://developer.apple.com/library/content/documentation/GraphicsImaging/Conceptual/drawingwithquartz2d/Introduction/Introduction.html#//apple_ref/doc/uid/TP30001066)中的[路径](https://developer.apple.com/library/content/documentation/GraphicsImaging/Conceptual/drawingwithquartz2d/dq_paths/dq_paths.html#//apple_ref/doc/uid/TP30001066-CH211)。 有关用于创建路径的函数的信息，请参阅 [CGContext 参考](https://developer.apple.com/documentation/coregraphics/cgcontext)和 [CGPath 参考](https://developer.apple.com/documentation/coregraphics/cgpath)。

### 创建图案，渐变和阴影

Core Graphics 框架包括用于创建图案，渐变和阴影的附加功能。 你可以使用这些类型创建非单色颜色并使用它们填充你创建的路径。 模式是通过重复图像或内容创建的。 渐变和阴影提供了不同的方式来创建从颜色到颜色的平滑过渡。

[Quartz 2D 编程指南](https://developer.apple.com/library/content/documentation/GraphicsImaging/Conceptual/drawingwithquartz2d/Introduction/Introduction.html#//apple_ref/doc/uid/TP30001066)涵盖了创建和使用图案，渐变和阴影的细节。

### 自定义坐标空间

默认情况下，UIKit 创建一个直接的当前转换矩阵，将点映射到像素上。 虽然你可以在不修改矩阵的情况下完成所有绘图，但有时候可以方便。

当你首次调用视图的 `drawRect:` 方法时，将配置 CTM，以便坐标系的原点与视图的原点匹配，其正 X 轴向右延伸，而其正 Y 轴向下延伸。 但是，你可以通过向其添加缩放，旋转和平移因子来更改 CTM，从而更改默认坐标系相对于基础视图或窗口的大小，方向和位置。

#### 使用坐标变换提高绘图性能

修改CTM是用于在视图中绘制内容的标准技术，因为它允许你重复使用路径，这可能会减少绘制时所需的计算量。 例如，如果要从点（20,20）开始绘制一个正方形，则可以创建一个移动到（20,20）的路径，然后绘制所需的一组线以完成该正方形。 但是，如果你稍后决定将该方块移动到点（10,10），则必须重新创建具有新起点的路径。 因为创建路径是一个相对昂贵的操作，所以最好创建一个原点为（0，0）的正方形并修改 CTM，以便在所需的原点绘制正方形。

在 Core Graphics 框架中，有两种方法可以修改 CTM。 你可以使用 [CGContext 参考](https://developer.apple.com/documentation/coregraphics/cgcontext)中定义的 CTM 操作函数直接修改 CTM。 你还可以创建 [CGAffineTransform](https://developer.apple.com/documentation/coregraphics/cgaffinetransform) 结构，应用所需的任何转换，然后将该转换连接到 CTM 上。 使用仿射变换可以将变换分组，然后将它们一次应用到 CTM。 你还可以评估和反转仿射变换，并使用它们修改代码中的点，大小和矩形值。 有关使用仿射变换的更多信息，请参见 [Quartz 2D Programming Guide](https://developer.apple.com/library/content/documentation/GraphicsImaging/Conceptual/drawingwithquartz2d/Introduction/Introduction.html#//apple_ref/doc/uid/TP30001066) 和 [CGAffineTransform Reference](https://developer.apple.com/documentation/coregraphics/cgaffinetransform-rb5)。

#### 翻转默认坐标系

在 UIKit 绘图中翻转会修改背景 [CALayer](https://developer.apple.com/documentation/quartzcore/calayer)，使具有 LLO 坐标系的绘图环境与 UIKit 的默认坐标系对齐。 如果你只使用 UIKit 方法和函数进行绘制，则不需要翻转 CTM。 但是，如果将 Core Graphics 或 Image I/O 函数调用与 UIKit 调用混合，则可能需要翻转 CTM。

具体而言，如果你通过直接调用 Core Graphics 函数来绘制图像或 PDF 文档，则该对象将在视图的上下文中上下颠倒。 你必须翻转 CTM 才能正确显示图像和页面。

要将绘制到 Core Graphics 上下文的对象翻转以使其在 UIKit 视图中显示时正确显示，你必须分两步修改 CTM。 将原点转换为绘图区域的左上角，然后应用比例平移，将 y 坐标修改为 -1。 这样做的代码看起来类似于以下内容：

```
CGContextSaveGState(graphicsContext);
CGContextTranslateCTM(graphicsContext, 0.0, imageHeight);
CGContextScaleCTM(graphicsContext, 1.0, -1.0);
CGContextDrawImage(graphicsContext, image, CGRectMake(0, 0, imageWidth, imageHeight));
CGContextRestoreGState(graphicsContext);
```

如果你创建一个使用 Core Graphics 图像对象初始化的 [UIImage](https://developer.apple.com/documentation/uikit/uiimage) 对象，则 UIKit 会为你执行翻转转换。 每个 UIImage 对象都由一个不透明类型的 [CGImageRef](https://developer.apple.com/documentation/coregraphics/cgimageref) 支持。 你可以通过 CGImage 属性访问 Core Graphics 对象，并对图像执行一些操作。 （Core Graphics 在 UIKit 中没有提供与图像相关的工具。）完成后，可以从修改的 `CGImageRef` 对象中重新创建 UIImage 对象。

注意：你可以使用 Core Graphics 函数 [CGContextDrawImage](https://developer.apple.com/documentation/coregraphics/1454845-cgcontextdrawimage) 将图像绘制到任何渲染目标。 该函数有两个参数，第一个用于图形上下文，第二个用于定义图像大小及其在绘图表面（如视图）中的位置的矩形。 使用 [CGContextDrawImage](https://developer.apple.com/documentation/coregraphics/1454845-cgcontextdrawimage) 绘制图像时，如果你不将当前坐标系调整为 LLO 方向，则图像在 UIKit 视图中显示为反转。 此外，传递给此函数的矩形的原点与函数调用时的当前坐标系的原点相关。

#### 不同坐标系统绘制的副作用

当你绘制一个对象时，会参照一种绘图技术的默认坐标系，然后将其渲染到另一个绘图技术的图形上下文中，从而渲染某些渲染异常。 你可能需要调整代码以解决这些副作用。

##### 弧和旋转

如果使用 [CGContextAddArc](https://developer.apple.com/documentation/coregraphics/1455756-cgcontextaddarc) 和 [CGPathAddArc](https://developer.apple.com/documentation/coregraphics/1411147-cgpathaddarc) 等函数绘制路径并假定 LLO 坐标系，则需要翻转 CTM 以在 UIKit 视图中正确渲染弧线。 但是，如果你使用相同的函数来创建一个位于 ULO 坐标系中的点，然后在 UIKit 视图中渲染路径，那么你会注意到该弧是其原始的改变版本。 现在，弧的终止端点指向与端点将使用 UIBezierPath 类创建的弧相反的方向。 例如，向下指向的箭头现在朝上（如图1-5所示），并且弧线“弯曲”的方向也不同。 你必须更改 Core Graphics 绘制的弧的方向，以考虑基于 ULO 的坐标系; 这个方向由这些函数的 startAngle 和 endAngle 参数控制。

图 1-5 Core Graphics 与 UIKit 中的 Arc 渲染
![](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/Art/flipped_coordinates-1_2x.png)

如果旋转对象（例如，通过调用 [CGContextRotateCTM](https://developer.apple.com/documentation/coregraphics/1456228-cgcontextrotatectm)），则可以观察到相同类型的镜像效果。 如果使用引用 ULO 坐标系的核心图形调用来旋转对象，则在 UIKit 中渲染时对象的方向会颠倒过来。 你必须考虑代码中不同的旋转方向; 使用 `CGContextRotateCTM`，你可以通过反转角度参数的符号来做到这一点（例如，负值变为正值）。

##### 阴影

阴影从其对象落下的方向由偏移值指定，并且该偏移的含义是绘图框架的约定。 在 UIKit 中，正 x 和 y 偏移会使阴影向下并位于对象的右侧。 在 Core Graphics 中，x 和 y 偏移的正值会使阴影向上并位于对象的右侧。 翻转 CTM 以将对象与 UIKit 的默认坐标系对齐不会影响对象的阴影，因此阴影无法正确跟踪其对象。 为了正确追踪它，你必须适当修改当前坐标系的偏移值。

> 注意：在 iOS 3.2 之前，Core Graphics 和 UIKit 在阴影方向上共享相同的约定：正偏移值使阴影向下并位于对象的右侧。

## 应用 Core Animation 效果

Core Animation 是一个 Objective-C 框架，它提供了快速而轻松地创建流畅的实时动画的基础设施。 Core Animation 本身并不是绘图技术，因为它不提供用于创建形状，图像或其他类型内容的原始例程。 相反，它是一种用于操纵和显示使用其他技术创建的内容的技术。

大多数应用程序可以受益于在 iOS 中以某种形式使用 Core Animation。 动画向用户提供有关正在发生的事情的反馈。 例如，当用户浏览设置应用程序时，屏幕会根据用户是进一步浏览偏好层次还是备份到根节点而滑入和滑出视图。 这种反馈非常重要，为用户提供了上下文信息。 它还增强了应用程序的视觉风格。

在大多数情况下，你可以通过很少的努力获得 Core Animation 的好处。 例如，UIView 类的几个属性（包括视图的 frame，center，color 和 opacity 等）可以配置为在其值发生更改时触发动画。 你必须做一些工作才能让 UIKit 知道你想要这些动画，但是动画本身是为你自动创建和运行的。 有关如何触发内置视图动画的信息，请参阅 [UIView 类参考](https://developer.apple.com/documentation/uikit/uiview)中的 Animating Views。

当你超越基本动画时，你必须更直接地与 Core Animation 类和方法进行交互。 以下各节提供了有关 Core Animation 的信息，并向你展示如何使用其类和方法在 iOS 中创建典型的动画。 有关 Core Animation 以及如何使用它的更多信息，请参阅 [Core Animation Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40004514)。

### 关于 Layers

Core Animation 中的关键技术是 layer 对象。 layer 是与视图性质相似的轻量级对象，但它们实际上是封装了要显示的内容的几何图形，时间和视觉属性的模型对象。 内容本身以三种方式之一提供：

- 你可以将 `CGImageRef` 分配给 layer 对象的 `contents` 属性。
- 你可以将 delegate 分配给 layer，并让 delegate 处理绘制。
- 你可以继承 [CALayer](https://developer.apple.com/documentation/quartzcore/calayer) 并覆盖其中一种显示方法。

当你操作 layer 对象的属性时，实际操作的是模型级别的数据，它决定如何显示关联的内容。 该内容的实际呈现与你的代码分开处理，并进行了大量优化以确保其速度。 你只需设置图层内容，配置动画属性，然后让 Core Animation 接管。

有关 layer 及其使用方式的更多信息，请参阅 [Core Animation Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40004514)。

### 关于动画

当谈到动画层时，Core Animation 使用单独的动画对象来控制动画的时间和行为。 [CAAnimation](https://developer.apple.com/documentation/quartzcore/caanimation) 类及其子类提供了不同类型的动画行为，你可以在代码中使用它们。 你可以创建简单的动画，将属性从一个值迁移到另一个值，也可以创建复杂的关键帧动画，通过你提供的一组值和时序函数跟踪动画。

核心动画还可让你将多个动画组合到一个单位中，称为转场。 [CATransaction](https://developer.apple.com/documentation/quartzcore/catransaction) 对象将一组动画作为一个单元来管理。 你也可以使用此类的方法来设置动画的持续时间。

有关如何创建自定义动画的示例，请参阅 [Animation Types and Timing Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Animation_Types_Timing/Introduction/Introduction.html#//apple_ref/doc/uid/TP40006166)。

### Core Animation Layers 中的比例因子计算

直接使用 Core Animation 图层提供内容的应用程序可能需要调整其绘图代码以考虑缩放因子。 通常，当你绘制视图的 [drawRect:](https://developer.apple.com/documentation/uikit/uiview/1622529-drawrect) 方法或 layer delegate 的 [drawLayer:inContext:](https://developer.apple.com/documentation/quartzcore/calayerdelegate/2097262-drawlayer) 方法时，系统会自动调整图形上下文以考虑缩放因子。 但是，当你的视图执行下列操作之一时，了解或更改该比例因子可能仍然是必需的：

- 创建具有不同比例因子的其他 Core Animation layers，并将它们合并到它自己的内容中
- 直接设置 Core Animation layer 的 [contents](https://developer.apple.com/documentation/quartzcore/calayer/1410773-contents) 属性

Core Animation 的合成引擎会查看每个 layer 的 [contentsScale](https://developer.apple.com/documentation/quartzcore/calayer/1410746-contentsscale) 属性，以确定在合成期间是否需要缩放该图层的内容。 如果你的应用程序创建没有关联视图的 layer，则每个新 layer 对象的比例因子最初设置为1.0。 如果不改变比例因子，并且随后在高分辨率屏幕上绘制 layer，则 layer 的内容会自动缩放以补偿缩放因子的差异。 如果你不希望缩放内容，可以通过为 `contentsScale` 属性设置新值来将图层比例因子更改为2.0，但如果你在不提供高分辨率内容的情况下这样做，则你现有的内容可能会比期待的要小。 要解决该问题，你需要为你的图层提供更高分辨率的内容。

> 重要说明：layer 的 [contentsGravity](https://developer.apple.com/documentation/quartzcore/calayer/1410872-contentsgravity) 属性在确定是否在高分辨率屏幕上缩放标准分辨率图层内容方面发挥着重要作用。 此属性默认设置为值 [kCAGravityResize](https://developer.apple.com/documentation/quartzcore/kcagravityresize)，这会导致图层内容缩放以适合图层的边界。 将重力改变为非限制选项消除了否则会发生的自动缩放。 在这种情况下，你可能需要相应地调整你的内容或比例因子。

直接设置 layer 的 [contents](https://developer.apple.com/documentation/quartzcore/calayer/1410773-contents) 属性时，调整 layer 的内容以适应不同比例因子是最合适的。 Quartz 图像没有比例因子的概念，因此直接与像素一起工作。 因此，在创建你计划用于 layer 内容的 [CGImageRef](https://developer.apple.com/documentation/coregraphics/cgimageref) 对象之前，请检查比例因子并相应地调整图像的大小。 具体来说，从应用程序包中加载适当大小的图像，或者使用 [UIGraphicsBeginImageContextWithOptions](https://developer.apple.com/documentation/uikit/1623912-uigraphicsbeginimagecontextwitho) 函数来创建缩放因子与 layer 比例因子匹配的图像。 如果不创建高分辨率位图，则可以按照前面讨论的方式缩放现有的位图。

有关如何指定和加载高分辨率图像的信息，请参阅 [Loading Images into Your App](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/SupportingHiResScreensInViews/SupportingHiResScreensInViews.html#//apple_ref/doc/uid/TP40010156-CH15-SW3)。 有关如何创建高分辨率图像的信息，请参阅 [绘制位图上下文和 PDF 上下文](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/GraphicsDrawingOverview/GraphicsDrawingOverview.html#//apple_ref/doc/uid/TP40010156-CH14-SW22)。

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~