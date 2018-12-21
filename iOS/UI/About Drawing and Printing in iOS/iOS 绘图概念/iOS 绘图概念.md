# iOS 绘图概念

高质量的图形是应用程序用户界面的重要组成部分。提供高质量的图形不仅使你的应用程序看起来很好，而且还使你的应用程序看起来像是系统其余部分的自然扩展。iOS 提供了两种在系统中创建高质量图形的主要途径：使用 OpenGL 或 Quartz，Core Animation 和 UIKit 的原生渲染。本文档描述了原生渲染。（要了解 OpenGL 绘图，请参阅 OpenGL ES 编程指南。）

Quartz 是主要的绘图接口，支持基于路径的绘制，消除锯齿渲染，渐变填充图案，图像，颜色，坐标空间转换以及 PDF 文档创建，显示和解析。UIKit 为 line art，Quartz images 和颜色处理提供 Objective-C 包装。Core Animation 为许多 UIKit 视图属性中的更改动画提供了底层支持，还可用于实现自定义动画。

本章概述了 iOS 应用程序的绘图过程，以及每种支持的绘图技术的特定绘图技术。你还可以找到有关如何针对 iOS 平台优化绘图代码的提示和指导。

> 重要提示：并非所有 UIKit 类都是线程安全的。在应用程序主线程以外的线程上执行与绘图相关的操作之前，请务必查看文档。

## UIKit 图形系统

在 iOS 中，无论是否涉及 OpenGL，Quartz，UIKit 或 Core Animation，所有绘制到屏幕的内容都发生在 UIView 类的实例或其子类的范围内。视图定义了发生绘图的屏幕部分。如果使用系统提供的视图，则会自动为你处理此图形。但是，如果定义自定义视图，则必须自己提供绘图代码。如果使用 Quartz，Core Animation 和 UIKit 进行绘制，则使用以下各节中描述的绘图概念。

除了直接绘制到屏幕外，UIKit 还允许你绘制到屏幕外的位图和 PDF 图形上下文。在非屏幕上下文中绘制时，你没有在视图中绘制，这意味着视图绘制周期等概念不适用（除非你获取该图像并在图像视图中绘制它或类似图像）。

### 视图绘制周期

UIView 类的子类的基本绘图模型涉及按需更新内容。UIView 类使更新过程更容易，更有效; 但是，通过收集更新请求，你可以在最合适的时间将它们交付给你的绘图代码。首次显示视图或需要重绘视图的一部分时，iOS 会要求视图通过调用视图的 drawRect: 方法来绘制其内容。

有几个操作可以触发视图更新：

- 移动或删除部分遮挡视图的其他视图
- 通过将其 hidden 属性设置为 NO，可以再次显示先前隐藏的视图
- 滚动屏幕上的视图，然后返回到屏幕上
- 显式调用视图的 setNeedsDisplay 或 setNeedsDisplayInRect: 方法

系统视图会自动重绘。对于自定义视图，你必须覆盖 drawRect: 方法并在其中执行所有绘图。在 drawRect: 方法中，使用原生绘图技术绘制所需的形状，文本，图像，渐变或任何其他可视内容。第一次看到你的视图时，iOS 会将一个矩形传递给视图的 drawRect: 方法，该方法包含视图的整个可见区域。在后续调用期间，矩形仅包含实际需要重绘的视图部分。为获得最佳性能，你应仅重绘受影响的内容。

调用 drawRect: 方法后，视图将自身标记为已更新，并等待新操作到达并触发另一个更新周期。如果你的视图显示静态内容，那么你需要做的就是响应视图因滚动和其他视图的存在而导致的可见性变化。

但是，如果要更改视图的内容，则必须告诉视图重绘其内容。为此，请调用 setNeedsDisplay 或 setNeedsDisplayInRect: 方法以触发更新。例如，如果你每秒多次更新内容，则可能需要设置计时器以更新视图。你还可以更新视图以响应用户交互或在视图中创建新内容。

> 重要提示：请勿自行调用视图的 drawRect: 方法。只有在屏幕重绘期间内置于 iOS 中的代码才能调用该方法。在其他时候，不存在图形上下文，因此无法绘图。（图形上下文将在下一节中介绍。）

### iOS 中的坐标系和绘图

当应用程序在 iOS 中绘制内容时，它必须在由坐标系定义的二维空间中定位绘制的内容。这个概念乍一看似乎很简单，但事实并非如此。iOS 中的应用有时必须在绘制时处理不同的坐标系。

在 iOS 中，所有绘图都在图形上下文中进行。从概念上讲，图形上下文是描述绘图应在何处以及如何发生的对象，包括基本绘图属性，例如绘制时使用的颜色，剪切区域，线宽和样式信息，字体信息，合成选项等。

另外，如图1-1所示，每个图形上下文都有一个坐标系。更确切地说，每个图形上下文都有三个坐标系：

- 绘图（用户）坐标系。发出绘图命令时使用此坐标系。
- 视图坐标系（基本空间）。该坐标系是相对于视图的固定坐标系。
- （物理）设备坐标系。该坐标系表示物理屏幕上的像素。

图 1-1 绘图坐标，视图坐标和硬件坐标之间的关系
![](https://developer.apple.com/library/archive/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/Art/coordinate_differences_2x.png)

iOS 的绘图框架创建用于绘制到特定目的地的图形上下文 - 屏幕，位图，PDF 内容等 - 并且这些图形上下文为该目标建立初始绘图坐标系。此初始绘图坐标系称为默认坐标系，是视图底层坐标系上的1：1映射。

每个视图还具有当前变换矩阵（CTM），这是一种将当前绘图坐标系中的点映射到（固定）视图坐标系的数学矩阵。应用程序可以修改此矩阵（如稍后所述）以更改将来绘制操作的行为。

iOS 的每个绘图框架都基于当前图形上下文建立默认坐标系。在 iOS 中，有两种主要类型的坐标系：

- 左上方原点坐标系（ULO），其中绘制操作的原点位于绘图区域的左上角，正值向下和向右延伸。UIKit 和 Core Animation 框架使用的默认坐标系是基于 ULO 的。
- 左下原点坐标系（LLO），其中绘图操作的原点位于绘图区域的左下角，正值向上和向右延伸。Core Graphics 框架使用的默认坐标系是基于 LLO 的。

这些坐标系如图1-2所示。

图 1-2 iOS 中的默认坐标系
![](https://developer.apple.com/library/archive/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/Art/flipped_coordinates-2_2x.png)

> 注意：OS X 中的默认坐标系是基于 LLO 的。尽管 Core Graphics 和 AppKit 框架的绘图功能和方法非常适合此默认坐标系，但 AppKit 提供了编程支持，可以将绘图坐标系翻转为左上角原点。

在调用视图的 drawRect: 方法之前，UIKit 通过为绘图操作提供图形上下文来建立绘制到屏幕的默认坐标系。在视图的 drawRect: 方法中，应用程序可以设置图形状态参数（例如填充颜色）并绘制到当前图形上下文，而无需显式引用图形上下文。此隐式图形上下文建立 ULO 默认坐标系。

### 点与像素

在 iOS 中，你在绘图代码中指定的坐标与底层设备的像素之间存在区别。使用 Quartz，UIKit 和 Core Animation 等原生绘图技术时，绘图坐标空间和视图的坐标空间都是逻辑坐标空间，以点为单位测量距离。这些逻辑坐标系与系统框架用于管理屏幕上像素的设备坐标空间分离。

系统会自动将视图坐标空间中的点映射到设备坐标空间中的像素，但此映射并不总是一对一的。这种行为导致了一个重要的事实，你应该永远记住：

一点不一定对应于一个物理像素。

使用点（和逻辑坐标系）的目的是提供与设备无关的一致输出大小。在大多数情况下，点的实际大小是无关紧要的。点的目标是提供相对一致的比例，你可以在代码中使用它来指定视图和呈现内容的大小和位置。实际如何将点映射到像素是由系统框架处理的细节。例如，在具有高分辨率屏幕的设备上，一点宽的线实际上可能产生两个物理像素宽的线。结果是，如果你在两个类似的设备上绘制相同的内容，其中只有一个具有高分辨率屏幕，则两个设备上的内容大小大致相同。

> 注意：在 PDF 渲染和打印的上下文中，Core Graphics 使用一个点到 1/72 英寸的行业标准映射来定义“点”。

在 iOS 中，UIScreen，UIView，UIImage 和 CALayer 类提供了获取（并且在某些情况下，设置）比例因子的属性，该比例因子描述了该特定对象的点与像素之间的关系。例如，每个 UIKit 视图都有一个 contentScaleFactor 属性。在标准分辨率屏幕上，比例因子通常为1.0。在高分辨率屏幕上，比例因子通常为2.0。将来，其他比例因子也是可能的。（在版本4之前的iOS中，你应该假设比例因子为1.0。）

原生绘图技术（如 Core Graphics）会将当前比例因子考虑在内。例如，如果你的一个视图实现了 drawRect: 方法，UIKit 会自动将该视图的比例因子设置为屏幕的比例因子。此外，UIKit 会自动修改绘图过程中使用的任何图形上下文的当前变换矩阵，以考虑视图的比例因子。因此，你在 drawRect: 方法中绘制的任何内容都会针对底层设备的屏幕进行适当缩放。

由于这种自动映射，在编写绘图代码时，像素通常无关紧要。但是，有时你可能需要根据点映射到像素的方式更改应用程序的绘图行为 - 在具有高分辨率屏幕的设备上下载更高分辨率的图像，或者在低分辨率屏幕上绘图时避免缩放瑕疵，例如。

在 iOS 中，当你在屏幕上绘制内容时，图形子系统使用称为抗锯齿的技术来在较低分辨率的屏幕上逼近较高分辨率的图像。解释这种技术的最好方法是举例。当你在纯白色背景上绘制黑色垂直线时，如果该线恰好落在像素上，则它在白色区域中显示为一系列黑色像素。但是，如果它恰好出现在两个像素之间，则它会并排显示为两个灰色像素，如图1-3所示。

图 1-3 以整数点值为中心的单点线
![](https://developer.apple.com/library/archive/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/Art/pixel_alignment_2x.png)

由整数点定义的位置落在像素之间的中点处。例如，如果从（1.0,1.0）到（1.0,10.0）绘制一个像素宽的垂直线，则会得到模糊的灰线。如果绘制一条两像素宽的线，则会得到一条纯黑线，因为它完全覆盖两个像素（指定点两侧各一个）。通常，奇数个物理像素宽的线条比用偶数个物理像素测量宽度的线条更柔和，除非你调整它们的位置以使它们完全覆盖像素。

比例因子发挥作用的地方在于确定一点宽线覆盖多少像素。

在低分辨率显示器（比例因子为1.0）上，一点宽的线宽为一个像素。要在绘制一个点宽的水平或垂直线时避免抗锯齿，如果线宽为奇数个像素，则必须将位置偏移0.5个点到整个编号位置的任一侧。如果线宽是偶数个点，为了避免模糊线，你不能这样做。

图 1-4 标准和视网膜显示屏上一点宽线的外观
![](https://developer.apple.com/library/archive/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/Art/regular_vs_retina_2x.png)

在高分辨率显示器（比例因子为2.0）上，一个点宽的线根本没有抗锯齿，因为它占据两个完整像素（从-0.5到+0.5）。要绘制仅覆盖单个物理像素的线条，你需要将其设置为0.5个点，并将其位置偏移0.25个点。两种屏幕的比较如图1-4所示。

当然，基于比例因子改变绘图特性可能会产生意想不到的后果。1像素宽的线在某些设备上看起来不错，但在高分辨率设备上可能很薄，很难看清楚。由你决定是否进行此类更改。

### 获取图形上下文

大多数情况下，为你配置图形上下文。每个视图对象都会自动创建一个图形上下文，以便你的代码可以在调用自定义 drawRect: 方法后立即开始绘制。作为此配置的一部分，底层 UIView 类为当前绘图环境创建图形上下文（CGContextRef opaque 类型）。

如果要绘制视图以外的其他位置（例如，捕获 PDF 或位图文件中的一系列绘图操作），或者需要调用需要上下文对象的 Core Graphics 函数，则必须执行其他步骤获取图形上下文对象。以下部分解释了如何。

有关图形上下文，修改图形状态信息以及使用图形上下文创建自定义内容的更多信息，请参阅 Quartz 2D Programming Guide。有关与图形上下文结合使用的函数列表，请参阅 CGContext Reference，CGBitmapContext Reference 和 CGPDFContext Reference。

#### 绘制到屏幕

如果你使用 Core Graphics 函数绘制到视图，无论是在 drawRect: 方法还是其他地方，你都需要一个图形上下文来绘制。（许多这些函数的第一个参数必须是 CGContextRef 对象。）你可以调用函数 UIGraphicsGetCurrentContext 来获取 drawRect 中隐含的相同图形上下文的显式版本。因为它是相同的图形上下文，所以绘图函数也应该引用 ULO 默认坐标系。

如果要使用 Core Graphics 函数在 UIKit 视图中绘制，则应使用 UIKit 的 ULO 坐标系进行绘图操作。或者，你可以将翻转变换应用于 CTM，然后使用 Core Graphics 原生 LLO 坐标系在 UIKit 视图中绘制对象。Flipping the Default Coordinate System 详细讨论了翻转变换。

UIGraphicsGetCurrentContext 函数始终返回当前有效的图形上下文。例如，如果你创建 PDF 上下文然后调用 UIGraphicsGetCurrentContext，你将收到该 PDF 上下文。如果使用 Core Graphics 函数绘制视图，则必须使用 UIGraphicsGetCurrentContext 返回的图形上下文。

> 注意：UIPrintPageRenderer 类声明了几种绘制可打印内容的方法。以类似于 drawRect: 的方式，UIKit 为这些方法的实现安装隐式图形上下文。此图形上下文建立 ULO 默认坐标系。

#### 绘制到位图上下文和 PDF 上下文

UIKit 提供了在位图图形上下文中渲染图像的功能，以及通过绘制 PDF 图形上下文来生成 PDF 内容的功能。这两种方法都要求你首先分别调用创建图形上下文的函数 - 位图上下文或 PDF 上下文。返回的对象充当后续绘图和状态设置调用的当前（和隐式）图形上下文。在上下文中完成绘制后，可以调用另一个函数来关闭上下文。

UIKit 提供的位图上下文和 PDF 上下文都建立了 ULO 默认坐标系。Core Graphics 具有相应的功能，用于在位图图形上下文中进行渲染以及在 PDF 图形上下文中进行绘制。但是，应用程序直接通过 Core Graphics 创建的上下文建立了 LLO 默认坐标系。

> 注意：在 iOS 中，建议你使用 UIKit 函数绘制到位图上下文和 PDF 上下文。但是，如果你确实使用了 Core Graphics 替代方案并打算显示渲染结果，则必须调整代码以补偿默认坐标系中的差异。有关详细信息，请参阅翻转默认坐标系。

有关详细信息，请参阅绘制和创建图像（用于绘制到位图上下文）和生成 PDF 内容（用于绘制到 PDF 上下文）。

### 颜色和颜色空间

iOS 支持 Quartz 提供的全系列色彩空间; 但是，大多数应用程序应该只需要 RGB 颜色空间。由于 iOS 设计为在嵌入式硬件上运行并在屏幕上显示图形，因此 RGB 颜色空间是最合适的颜色空间。UIColor 对象提供了使用 RGB，HSB 和灰度值指定颜色值的便捷方法。以这种方式创建颜色时，你永远不需要指定颜色空间。它由 UIColor 对象自动确定。

你还可以使用 Core Graphics 框架中的 CGContextSetRGBStrokeColor 和 CGContextSetRGBFillColor 函数来创建和设置颜色。虽然 Core Graphics 框架包括支持使用其他颜色空间创建颜色，以及创建自定义颜色空间，但不建议在绘图代码中使用这些颜色。你的绘图代码应始终使用 RGB 颜色。

## 用 Quartz 和 UIKit 绘图

Quartz 是 iOS 中原生绘图技术的通用名称。Core Graphics 框架是 Quartz 的核心，也是你用于绘制内容的主要接口。该框架提供了用于操作以下内容的数据类型和函数：

- Graphics contexts
- Paths
- Images and bitmaps
- Transparency layers
- Colors, pattern colors, and color spaces
- Gradients and shadings
- Fonts
- PDF content

UIKit 通过为图形相关操作提供一组集中的类来构建 Quartz 的基本功能。UIKit 图形类并不是一套全面的绘图工具--Core Graphics 已经提供了这些工具。相反，它们为其他 UIKit 类提供绘图支持。UIKit 支持包括以下类和功能：

- UIImage, 它实现了一个用于显示图像的不可变类
- UIColor, 它为设备颜色提供基本支持
- UIFont, 它为需要它的类提供字体信息
- UIScreen, 它提供有关屏幕的基本信息
- UIBezierPath, 这使你的应用程序可以绘制线条，弧形，椭圆形和其他形状。
- 用于生成 UIImage 对象的 JPEG 或 PNG 表示的函数
- 绘制到位图图形上下文的函数
- 通过绘制到 PDF 图形上下文生成 PDF 数据的功能
- 绘制矩形和剪裁绘图区域的功能
- 用于更改和获取当前图形上下文的函数

有关组成 UIKit 的类和方法的信息，请参阅 UIKit Framework Reference。有关构成 Core Graphics 框架的 opaque 类型和函数的更多信息，请参阅 Core Graphics Framework Reference。

### 配置图形上下文

在调用 drawRect: 方法之前，视图对象会创建图形上下文并将其设置为当前上下文。此上下文仅存在于 drawRect: call 的生命周期中。你可以通过调用 UIGraphicsGetCurrentContext 函数来检索指向此图形上下文的指针。此函数返回对 CGContextRef 类型的引用，你将其传递给 Core Graphics 函数以修改当前图形状态。表1-1列出了用于设置图形状态不同方面的主要功能。有关函数的完整列表，请参阅 CGContext Reference。该表还列出了 UIKit 存在的替代方案。

表 1-1 用于修改图形状态的 Core graphics 函数
![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/UI/About%20Drawing%20and%20Printing%20in%20iOS/iOS%20%E7%BB%98%E5%9B%BE%E6%A6%82%E5%BF%B5/Table%201-1.png?raw=true)

图形上下文包含一个栈用于保存的图形状态。当 Quartz 创建图形上下文时，栈为空。使用 CGContextSaveGState 函数将当前图形状态的副本推送到堆栈。此后，对图形状态所做的修改会影响后续的绘图操作，但不会影响存储在栈中的副本。完成修改后，可以使用 CGContextRestoreGState 函数将保存的状态弹出栈顶部，从而返回到先前的图形状态。以这种方式 pushing 和 poping 图形状态是返回先前状态的快速方法，并且无需单独撤消每个状态更改。它也是将状态的某些方面（例如剪切路径）恢复到其原始设置的唯一方法。

有关图形上下文的一般信息以及使用它们配置绘图环境，请参阅 Quartz 2D Programming Guide 中的 Graphics Contexts。

### 创建和绘制路径

路径是从一系列线和贝塞尔曲线创建的基于矢量的形状。UIKit 包含 UIRectFrame 和 UIRectFill 函数（以及其他函数）用于在视图中绘制简单路径，例如矩形。Core Graphics 还包括用于创建简单路径（如矩形和椭圆）的便捷功能。

对于更复杂的路径，你必须使用 UIKit 的 UIBezierPath 类自己创建路径，或者使用在 Core Graphics 框架中对 CGPathRef opaque 类型进行操作的函数。虽然你可以使用任一 API 构建没有图形上下文的路径，但路径中的点仍然必须引用当前坐标系（具有 ULO 或 LLO 方向），并且你仍需要图形上下文来实际呈现路径。

绘制路径时，必须设置当前上下文。此上下文可以是自定义视图的上下文（在 drawRect:)中，位图上下文或 PDF 上下文。坐标系确定路径的呈现方式。UIBezierPath 假定 ULO 坐标系。因此，如果你的视图被翻转（使用 LLO 坐标），则生成的形状可能会呈现与预期不同的形状。为获得最佳结果，应始终指定相对于用于渲染的图形上下文的当前坐标系原点的点。

> 注意：即使遵循此“规则”，Arc 也是需要额外工作的路径的一个方面。如果使用 Core Graphic 函数创建路径，该函数定位 ULO 坐标系中的点，然后在 UIKit 视图中渲染路径，则弧“指向”的方向不同。有关此主题的更多信息，请参阅 Side Effects of Drawing with Different Coordinate Systems。

要在 iOS 中创建路径，建议你使用 UIBezierPath 而不是 CGPath 函数，除非你需要一些仅 Core Graphics 提供的功能，例如向路径添加省略号。有关在 UIKit 中创建和渲染路径的更多信息，请参阅 Drawing Shapes Using Bézier Paths。

有关使用 UIBezierPath 绘制路径的信息，请参阅 Drawing Shapes Using Bézier Paths。有关如何使用 Core Graphics 绘制路径的信息，包括有关如何为复杂路径元素指定点的信息，请参阅 Quartz 2D Programming Guide 中的 Paths。有关用于创建路径的函数的信息，请参阅 CGContext Reference 和 CGPath Reference。

### 创建模式，渐变和阴影

Core Graphics 框架包含用于创建模式，渐变和阴影的附加功能。你可以使用这些类型创建非单色颜色，并使用它们填充你创建的路径。模式是根据重复的图像或内容创建的。渐变和阴影提供了不同的方法来创建从颜色到颜色的平滑过渡。Quartz 2D Programming Guide 中包含了创建和使用模式，渐变和阴影的详细信息。

### 自定义坐标空间

默认情况下，UIKit 创建一个直接的当前变换矩阵，将点映射到像素上。虽然你可以在不修改矩阵的情况下完成所有绘图，但有时这样做很方便。首次调用视图的 drawRect: 方法时，将配置 CTM，使坐标系的原点与视图的原点匹配，正 X 轴向右延伸，正 Y 轴向下延伸。但是，你可以通过向其添加缩放，旋转和平移因子来更改 CTM，从而更改默认坐标系相对于基础视图或窗口的大小，方向和位置。

#### 使用坐标转换提高绘图性能

修改 CTM 是在视图中绘制内容的标准技术，因为它允许你重用路径，这可能会减少绘制时所需的计算量。例如，如果要从点（20,20）开始绘制一个正方形，则可以创建一个移动到（20,20）的路径，然后绘制所需的一组线以完成该正方形。但是，如果你稍后决定将该方块移动到该点（10,10），则必须使用新起点重新创建路径。因为创建路径是相对昂贵的操作，所以最好创建一个原点为（0,0）的正方形并修改CTM，以便在所需的原点绘制正方形。

在 Core Graphics 框架中，有两种方法可以修改 CTM。你可以使用 CGContext Reference 中定义的 CTM 操作函数直接修改 CTM。你还可以创建 CGAffineTransform 结构，应用所需的任何转换，然后将该转换连接到 CTM。使用仿射变换可以对变换进行分组，然后将它们一次性应用到 CTM。你还可以评估和反转仿射变换，并使用它们来修改代码中的点，大小和矩形值。有关使用仿射变换的更多信息，请参阅 Quartz 2D Programming Guide 和 CGAffineTransform Reference。

#### 翻转默认坐标系

在 UIKit 绘图中翻转会修改背景 CALayer，以将具有 LLO 坐标系的绘图环境与 UIKit 的默认坐标系对齐。如果你只使用 UIKit 方法和绘图功能，则不需要翻转 CTM。但是，如果将 Core Graphics 或 Image I/O 函数调用与 UIKit 调用混合使用，则可能需要翻转 CTM。

具体来说，如果你通过直接调用 Core Graphics 函数绘制图像或 PDF 文档，则该对象将在视图的上下文中呈现为倒置。你必须翻转 CTM 才能正确显示图像和页面。

要将绘制到 Core Graphics 上下文的对象翻转，以便在 UIKit 视图中显示时正确显示，你必须分两步修改 CTM。将原点转换为绘图区域的左上角，然后应用缩放平移，将 y 坐标修改为-1。执行此操作的代码类似于以下内容：

```objc
CGContextSaveGState(graphicsContext);
CGContextTranslateCTM(graphicsContext, 0.0, imageHeight);
CGContextScaleCTM(graphicsContext, 1.0, -1.0);
CGContextDrawImage(graphicsContext, image, CGRectMake(0, 0, imageWidth, imageHeight));
CGContextRestoreGState(graphicsContext);
```

如果你创建使用 Core Graphics 图像对象初始化的 UIImage 对象，UIKit 会为你执行翻转变换。每个 UIImage 对象都由 CGImageRef opaque 类型支持。你可以通过 CGImage 属性访问 Core Graphics 对象，并对图像进行一些操作。（Core Graphics 在 UIKit 中没有图像相关功能。）完成后，可以从修改后的 CGImageRef 对象重新创建 UIImage 对象。

> 注意：你可以使用 Core Graphics 函数 CGContextDrawImage 将图像绘制到任何渲染目标。此函数有两个参数，第一个用于图形上下文，第二个用于矩形，用于定义图像的大小及其在绘图表面（如视图）中的位置。使用 CGContextDrawImage 绘制图像时，如果不将当前坐标系调整为 LLO 方向，则图像在 UIKit 视图中显示为倒置。此外，传递给此函数的矩形的原点是相对于调用函数时当前坐标系的原点。

#### 不同坐标系下绘图的副作用

当你使用一个绘图技术的默认坐标系绘制对象然后在另一个绘图技术的图形上下文中渲染时，会显示一些渲染奇怪现象。你可能需要调整代码以解决这些副作用。

##### 弧和旋转

如果使用 CGContextAddArc 和 CGPathAddArc 等函数绘制路径并假设 LLO 坐标系，则需要翻转 CTM 以在 UIKit 视图中正确渲染弧。但是，如果使用相同的函数创建包含位于 ULO 坐标系中的点的弧，然后在 UIKit 视图中渲染路径，你将注意到弧是其原始的更改版本。现在，弧的终止端点指向与使用 UIBezierPath 类创建的弧所做的端点相反的方向。例如，向下箭头现在指向上方（如图1-5所示），弧“弯曲”的方向也不同。你必须更改 Core Graphics 绘制弧的方向以考虑基于 ULO 的坐标系; 此方向由这些函数的 startAngle 和 endAngle 参数控制。

图 1-5 Core Graphics 与 UIKit 中的弧形渲染
![](https://developer.apple.com/library/archive/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/Art/flipped_coordinates-1_2x.png)

如果旋转对象，则可以观察到相同类型的镜像效果（例如，通过调用 CGContextRotateCTM）。如果使用引用 ULO 坐标系的 Core Graphics 调用旋转对象，则在 UIKit 中渲染时对象的方向将反转。你必须在代码中考虑不同的轮换方向; 使用 CGContextRotateCTM，你可以通过反转角度参数的符号来执行此操作（例如，负值变为正值）。

##### 阴影

阴影从其对象落下的方向由偏移值指定，该偏移的含义是绘图框架的约定。在 UIKit 中，正 x 和 y 偏移使阴影向下并且在对象的右侧。在 Core Graphics 中，正 x 和 y 偏移会使阴影上升到对象的右侧。翻转 CTM 以使对象与 UIKit 的默认坐标系对齐不会影响对象的阴影，因此阴影无法正确跟踪其对象。要使其正确跟踪，必须适当修改当前坐标系的偏移值。

> 注意：在 iOS 3.2 之前，Core Graphics 和 UIKit 对阴影方向共享相同的约定：正偏移值使阴影向下和对象的右侧。

## 应用 Core Animation 效果

Core Animation 是一个 Objective-C 框架，为快速，轻松地创建流畅的实时动画提供基础设施。Core Animation 本身并不是绘图技术，因为它不提供用于创建形状，图像或其他类型内容的原始例程。相反，它是一种操纵和显示你使用其他技术创建的内容的技术。

大多数应用程序都可以从 iOS 中以某种形式使用 Core Animation 中受益。动画向用户提供有关正在发生的事情的反馈。例如，当用户浏览“设置”应用时，屏幕会根据用户是否在首选项层次结构中向下导航或返回到根节点而滑入和滑出视图。这种反馈很重要，并为用户提供上下文信息。它还增强了应用程序的视觉风格。

在大多数情况下，你可以轻松地获得 Core Animation 的好处。例如，UIView 类的几个属性（包括视图的 frame，center，color 和 opacity 等）可以配置为在其值发生变化时触发动画。你必须做一些工作才能让 UIKit 知道你希望执行这些动画，但动画本身会自动创建并运行。有关如何触发内置视图动画的信息，请参阅 UIView Class Reference 中的 Animating Views。

当你超越基本动画时，你必须更直接地与 Core Animation 类和方法进行交互。以下部分提供有关 Core Animation 的信息，并向你展示如何使用其类和方法在 iOS 中创建典型动画。有关 Core Animation 及其使用方法的其他信息，请参阅 Core Animation Programming Guide。

### 关于 Layers

Core Animation 中的关键技术是 layer 对象。Layers 是轻量级对象，在本质上与视图类似，但实际上是模型对象，它们封装了要显示的内容的几何，时间和视觉属性。内容本身以三种方式之一提供：

- 你可以将 CGImageRef 指定给 layer 对象的 contents 属性。
- 你可以将一个 delegate 分配给 layer，并让 delegate 处理绘制。
- 你可以子类化 CALayer 并覆盖其中一种显示方法。

操作 layer 对象的属性时，实际操作的是模型级数据，用于确定应如何显示关联内容。该内容的实际呈现与你的代码分开处理，并经过大量优化以确保其快速。你所要做的就是设置 layer 内容，配置动画属性，然后让 Core Animation 接管。

有关 layers 及其使用方式的更多信息，请参阅 Core Animation Programming Guide。

### 关于动画

在动画 layers 时，Core Animation 使用单独的动画对象来控制动画的时间和行为。CAAnimation 类及其子类提供了可在代码中使用的不同类型的动画行为。你可以创建将属性从一个值迁移到另一个值的简单动画，也可以创建复杂的关键帧动画，通过你提供的一组值和计时功能来跟踪动画。

Core Animation 还允许你将多个动画组合到一个单元中，称为事务。CATransaction 对象将动画组作为一个单元进行管理。你还可以使用此类的方法来设置动画的持续时间。

有关如何创建自定义动画的示例，请参阅 Animation Types and Timing Programming Guide。

### 考虑 Core Animation Layers 的比例因子

直接使用 Core Animation layers 提供内容的应用可能需要调整其绘图代码以考虑比例因子。通常，当你在视图的 drawRect: 方法中绘制时，或者在 layer delegate 的 drawLayer:inContext: 方法中绘制时，系统会自动调整图形上下文以考虑比例因子。但是，当你的视图执行以下操作之一时，可能仍然需要了解或更改该比例因子：

- 创建具有不同比例因子的其他 Core Animation layers，并将它们合成为自己的内容
- 直接设置 Core Animation layer 的 contents 属性

Core Animation 的合成引擎查看每个图层的 contentsScale 属性，以确定在合成期间是否需要缩放该图层的内容。如果你的应用创建没有关联视图的 layers，则每个新 layer 对象的比例因子最初设置为1.0。如果不更改该比例因子，并且随后在高分辨率屏幕上绘制图层，则会自动缩放 layer 的内容以补偿比例因子的差异。如果你不希望缩放内容，可以通过为 contentsScale 属性设置新值来将 layer 的比例因子更改为2.0，但如果你这样做而不提供高分辨率内容，则现有内容可能会比你期待的要大。要解决该问题，你需要为 layer 提供更高分辨率的内容。

> 重要说明：layer 的 contentsGravity 属性在确定标准分辨率图层内容是否在高分辨率屏幕上缩放时起作用。默认情况下，此属性设置为值 kCAGravityResize，这会导致 layer 内容缩放以适合 layer 的边界。将 gravity 更改为非尺寸选项可消除否则会发生的自动缩放。在这种情况下，你可能需要相应地调整内容或比例因子。

当你直接设置 layer 的 contents 属性时，调整 layer 的内容以适应不同的比例因子是最合适的。Quartz 图像没有比例因子的概念，因此直接与像素一起工作。因此，在创建计划用于 layer 内容的 CGImageRef 对象之前，请检查比例因子并相应地调整图像的大小。具体来说，从应用程序包中加载适当大小的图像，或使用 UIGraphicsBeginImageContextWithOptions 函数创建一个图像，其比例因子与 layer 的比例因子相匹配。如果你不创建高分辨率位图，则可以如前所述缩放现有位图。

有关如何指定和加载高分辨率图像的信息，请参阅 Loading Images into Your App。有关如何创建高分辨率图像的信息，请参阅 Drawing to Bitmap Contexts and PDF Contexts。