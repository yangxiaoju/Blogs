# 在视图中支持高分辨率屏幕

针对 iOS SDK 4.0 及更高版本构建的应用程序需要准备好在具有不同屏幕分辨率的设备上运行。 幸运的是，iOS 可以轻松支持多种屏幕分辨率。 大部分处理不同类型屏幕的工作都是由系统框架为你完成的。 但是，你的应用仍需要做一些工作来更新基于栅格的图像，根据你的应用，你可能需要做额外的工作以充分利用可用的额外像素。

请参阅[点与像素](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/GraphicsDrawingOverview/GraphicsDrawingOverview.html#//apple_ref/doc/uid/TP40010156-CH14-SW7)以了解与此主题相关的重要背景信息。

## 支持高分辨率屏幕的清单

要为具有高分辨率屏幕的设备更新你的应用程序，你需要执行以下操作：

- 为应用程序包中的每个图像资源提供高分辨率图像，如[更新图像资源文件](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/SupportingHiResScreensInViews/SupportingHiResScreensInViews.html#//apple_ref/doc/uid/TP40010156-CH15-SW8)中所述。

- 提供高分辨率的应用程序和文档图标，如[更新应用程序的图标和启动图像](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/SupportingHiResScreensInViews/SupportingHiResScreensInViews.html#//apple_ref/doc/uid/TP40010156-CH15-SW2)中所述。

- 对于基于矢量的形状和内容，继续像以前一样使用定制的 Core Graphics 和 UIKit 绘图代码。 如果你想为绘制的内容添加额外的细节，请参阅[点与像素](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/GraphicsDrawingOverview/GraphicsDrawingOverview.html#//apple_ref/doc/uid/TP40010156-CH14-SW7)以获取有关如何操作的信息。

- 如果使用 OpenGL ES 进行绘图，请决定是否要选择高分辨率绘图并相应地设置图层的缩放比例，如[使用 OpenGL ES 或 GLKit 绘制高分辨率内容](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/SupportingHiResScreensInViews/SupportingHiResScreensInViews.html#//apple_ref/doc/uid/TP40010156-CH15-SW11)中所述。

- 对于你创建的自定义图像，修改图像创建代码以考虑当前比例因子，如[绘制位图上下文和 PDF 上下文](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/GraphicsDrawingOverview/GraphicsDrawingOverview.html#//apple_ref/doc/uid/TP40010156-CH14-SW22)中所述。

- 如果你的应用使用 Core Animation，请根据需要调整你的代码以补偿缩放因子，如 [Core Animation Layers 中的 Accounting for Scale Factors](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/GraphicsDrawingOverview/GraphicsDrawingOverview.html#//apple_ref/doc/uid/TP40010156-CH14-SW37) 中所述。

## 你可以免费获得绘制改进

iOS 中的绘图技术提供了大量的支持，无论底层屏幕的分辨率如何，都可以帮助你使渲染内容看起来更加美观：

- 标准 UIKit 视图（文本视图，按钮，表格视图等）在任何分辨率下自动呈现。
- 基于矢量的内容（[UIBezierPath](https://developer.apple.com/documentation/uikit/uibezierpath)，[CGPathRef](https://developer.apple.com/documentation/coregraphics/cgpath)，PDF）自动利用任何其他像素为形状渲染更清晰的线条。
- 文本会以更高的分辨率自动呈现。
- UIKit 支持自动加载图像的高分辨率变体（@2x）。 

如果你的应用仅使用原生绘图技术进行渲染，则你需要做的仅仅是支持更高分辨率的屏幕，这是你的图像的高分辨率版本。

## 更新你的图像资源文件

在 iOS 4 中运行的应用程序现在应该为每个图像资源包含两个单独的文件。 一个文件提供给定图像的标准分辨率版本，第二个文件提供同一图像的高分辨率版本。 每对图像文件的命名约定如下所示：

- 标准: `<ImageName><device_modifier>.<filename_extension>`
- 高分辨率: `<ImageName>@2x<device_modifier>.<filename_extension>`

每个名称的 `<ImageName>` 和 `<filename_extension>` 部分指定文件的常用名称和扩展名。 `<device_modifier>` 部分是可选的，并且包含字符串`〜ipad`或`〜iphone`。 如果要为 iPad 和 iPhone 指定不同版本的图像，可以包含这些修改器之一。 包含用于高分辨率图像的 `@2x` 修改器是新增功能，可让系统知道图像是标准图像的高分辨率变体。

> 重要提示：修饰符的顺序非常重要。 如果在设备修饰符后面错误地放置了 @2x，iOS 将无法找到该图像。

创建图像的高分辨率版本时，请将新版本放在应用程序包中与原始版本相同的位置。

### 将图像加载到你的应用程序

[UIImage](https://developer.apple.com/documentation/uikit/uiimage) 类处理将高分辨率图像加载到应用程序所需的所有工作。 在创建新图像对象时，你可以使用相同的名称来请求图像的标准和高分辨率版本。 例如，如果你有两个名为 Button.png 和 Button@2x.png 的图像文件，则可以使用以下代码来请求您的按钮图像：

```
UIImage *anImage = [UIImage imageNamed:@"Button"];
```

> 注意：在 iOS 4 及更高版本中，指定图像名称时可以省略文件扩展名。

在具有高分辨率屏幕的设备上，[imageNamed:](https://developer.apple.com/documentation/uikit/uiimage/1624146-imagenamed), [imageWithContentsOfFile:](https://developer.apple.com/documentation/uikit/uiimage/1624123-imagewithcontentsoffile) 和 [initWithContentsOfFile:](https://developer.apple.com/documentation/uikit/uiimage/1624112-init) 方法会自动在名称中使用 @2x 修饰符查找所请求图像的版本。 如果它找到一个，它会加载该图像。 如果你没有提供给定图像的高分辨率版本，则图像对象仍会加载标准分辨率图像（如果存在）并在绘制过程中对其进行缩放。

当它加载图像时，`UIImage` 对象根据图像文件的后缀自动将 [size](https://developer.apple.com/documentation/uikit/uiimage/1624105-size) 和 [scale](https://developer.apple.com/documentation/uikit/uiimage/1624110-scale) 属性设置为适当的值。 对于标准分辨率图像，它将 scale 属性设置为 1.0，并将图像的大小设置为图像的像素尺寸。 对于文件名中带有 @2x 后缀的图像，它将 scale 属性设置为 2.0，并将宽度和高度值减半以补偿缩放因子。 这些减半的值与你需要在逻辑坐标空间中使用以绘制图像的基于点的尺寸正确关联。

> 注意：如果使用 Core Graphics 创建图像，请记住 Quartz 图像没有明确的比例因子，因此其比例因子假定为 1.0。 如果要从 [CGImageRef](https://developer.apple.com/documentation/coregraphics/cgimageref) 数据类型创建 `UIImage` 对象，请使用 [initWithCGImage:scale:orientation:](https://developer.apple.com/documentation/uikit/uiimage/1624091-initwithcgimage) 来执行此操作。 该方法允许您将特定比例因子与Quartz图像数据相关联。

UIImage 对象在绘制过程中会自动考虑其比例因子。 因此，只要你在应用程序包中提供正确的图像资源，任何用于渲染图像的代码都应该一样。

### 使用图像视图显示多个图像

如果你的应用程序使用 [UIImageView](https://developer.apple.com/documentation/uikit/uiimageview) 类为高亮或动画呈现多个图像，则分配给该视图的所有图像必须使用相同的比例因子。 你可以使用图像视图来显示单个图像或多个图像的动画，还可以提供高光图像。 因此，如果你为其中一个图像提供高分辨率版本，则所有图像都必须具有高分辨率版本。

### 更新你的应用程序的图标并启动图像

除了更新应用程序的自定义图像资源之外，你还应该为应用程序的图标和启动图像提供新的高分辨率图标。 更新这些图像资源的过程与所有其他图像资源相同。 创建图像的新版本，将 @2x 修饰符字符串添加到相应的图像文件名，并像原始图像一样处理图像。 例如，对于应用程序图标，将高分辨率图像文件名添加到应用程序的 `Info.plist` 文件的 `CFBundleIconFiles` 项。

有关为你的应用程序指定图标和启动图像的信息，请参阅[适用于 iOS 的应用程序编程指南](https://developer.apple.com/library/content/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40007072)中的[应用程序相关资源](https://developer.apple.com/library/content/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/Inter-AppCommunication/Inter-AppCommunication.html#//apple_ref/doc/uid/TP40007072-CH6)。

## 使用 OpenGL ES 或 GLKit 绘制高分辨率内容

如果你的应用程序使用 OpenGL ES 或 GLKit 进行渲染，则你现有的绘图代码应该继续工作而不做任何更改。 然而，当在高分辨率屏幕上绘制时，你的内容会相应地缩放并显得更加块状。 块状外观的原因是，用于支持 OpenGL ES 渲染缓冲区（直接或间接）的 CAEAGLLayer 类的默认行为与其他 Core Animation 图层对象相同。 换句话说，它的缩放因子最初设置为1.0，这将导致 Core Animation 合成器在高分辨率屏幕上缩放图层的内容。 为了避免这种块状外观，你需要增加 OpenGL ES 渲染缓冲区的大小以匹配屏幕的大小。 （随着像素数量的增加，你可以增加为内容提供的细节数量。）因为向渲染缓冲区添加更多像素会有性能影响，但你必须明确选择支持高分辨率屏幕。

要启用高分辨率绘图，你必须更改用于呈现 OpenGL ES 或 GLKit 内容的视图的比例因子。 将视图的 contentScaleFactor 属性从1.0更改为2.0会触发底层 CAEAGLLayer 对象的比例因子的匹配更改。 renderbufferStorage:fromDrawable: 方法，用于将图层对象绑定到渲染缓冲区，通过将图层的边界乘以其比例因子来计算渲染缓冲区的大小。 因此，将缩放因子加倍会使得到的渲染缓冲区的宽度和高度加倍，从而为你的内容提供更多的像素。 之后，你可以为这些附加像素提供内容。

清单B-1显示了将图层对象绑定到渲染缓冲区并检索得到的大小信息的正确方法。 如果你使用 OpenGL ES 应用程序模板创建代码，那么此步骤已经为你完成，而你唯一需要做的就是适当地设置视图的比例因子。 如果你没有使用 OpenGL ES 应用程序模板，则应使用与此类似的代码来检索渲染缓冲区大小。 你绝对不应该认为渲染缓冲区大小对于给定类型的设备是固定的。

列表 B-1 初始化渲染缓冲区的存储并检索其实际尺寸
```
GLuint colorRenderbuffer;
glGenRenderbuffersOES(1, &colorRenderbuffer);
glBindRenderbufferOES(GL_RENDERBUFFER_OES, colorRenderbuffer);
[myContext renderbufferStorage:GL_RENDERBUFFER_OES fromDrawable:myEAGLLayer];
 
// Get the renderbuffer size.
GLint width;
GLint height;
glGetRenderbufferParameterivOES(GL_RENDERBUFFER_OES, GL_RENDERBUFFER_WIDTH_OES, &width);
glGetRenderbufferParameterivOES(GL_RENDERBUFFER_OES, GL_RENDERBUFFER_HEIGHT_OES, &height);
```

> 重要提示：由 [CAEAGLLayer](https://developer.apple.com/documentation/quartzcore/caeagllayer) 对象支持的视图不应实现自定义 [drawRect:](https://developer.apple.com/documentation/uikit/uiview/1622529-drawrect) 方法。实现 [drawRect:](https://developer.apple.com/documentation/uikit/uiview/1622529-drawrect) 方法会导致系统更改视图的默认比例因子，使其与屏幕的比例因子相匹配。如果你的绘图代码不期待这种行为，你的应用内容将无法正确呈现。

如果你选择使用高分辨率绘图，则还需要相应地调整应用的模型和纹理资源。例如，在 iPad 或高分辨率设备上运行时，你可能需要选择较大的模型和更详细的纹理以利用增加的像素数量。相反，在标准分辨率的 iPhone 上，你可以继续使用较小的模型和纹理。

确定是否支持高分辨率内容的一个重要因素是性能。当你将图层的比例因子从1.0更改为2.0时，会出现四倍的像素，这会给碎片处理器带来额外的压力。如果你的应用执行许多每片段计算，像素的增加可能会降低你的应用的帧速率。如果你发现应用在较高比例因子下运行速度显着较慢，请考虑以下选项之一：

- 使用 [OpenGL ES 编程指南](https://developer.apple.com/library/content/documentation/3DDrawing/Conceptual/OpenGLES_ProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008793) 中的性能调整指南优化片段着色器的性能。
- 选择一个更简单的算法在片段着色器中实现。通过这样做，你可以降低每个像素的质量，以更高的分辨率渲染整个图像。
- 使用1.0和2.0之间的分数比例因子。 1.5的缩放因子提供比1.0的缩放因子更好的质量，但需要填充比缩放为2.0的图像更少的像素。
- iOS 4及更高版本中的 OpenGL ES 提供多重采样作为选项。即使你的应用可以使用较小的比例因子（甚至1.0），仍然可以实施多重采样。另一个优点是，该技术还可以在不支持高分辨率显示的设备上提供更高的质量。

最好的解决方案取决于你的 OpenGL ES 应用程序的需求;你应该测试多个选项并选择在性能和图像质量之间提供最佳平衡的方法。

如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~