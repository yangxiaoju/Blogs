# 加载图片

由于功能和美学原因，图像是应用程序用户界面的普遍元素。 它们可以成为应用程序的关键差异化因素。

应用程序使用的许多图像（包括启动图像和应用程序图标）以文件形式存储在应用程序的主包中。 你可以拥有特定于设备类型（iPad 与 iPhone 和 iPod touch）并且针对高分辨率显示器进行了优化的启动图像和图标。 你可以在 [iOS 应用程序编程指南](https://developer.apple.com/library/content/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40007072)的[高级应用程序技巧](https://developer.apple.com/library/content/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/PerformanceTips/PerformanceTips.html#//apple_ref/doc/uid/TP40007072-CH7)和[应用程序相关资源](https://developer.apple.com/library/content/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/Inter-AppCommunication/Inter-AppCommunication.html#//apple_ref/doc/uid/TP40007072-CH6)中找到这些 bundle 图像文件的完整说明。 [更新你的图像资源文件](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/SupportingHiResScreensInViews/SupportingHiResScreensInViews.html#//apple_ref/doc/uid/TP40010156-CH15-SW8)讨论了使图像文件与高分辨率屏幕兼容的调整。

另外，iOS 支持使用 UIKit 和 Core Graphics 框架来加载和显示图像。 你如何确定用于绘制图像的类和函数取决于你打算如何使用它们。 但是，只要有可能，建议你使用 UIKit 类在代码中表示图像。 表C-1列出了一些使用场景以及处理它们的推荐选项。

表 C-1 图像的使用场景
| Scenario | Recommended usage |
|-|-|
| 显示图像作为视图的内容 | 使用 [UIImageView](https://developer.apple.com/documentation/uikit/uiimageview) 类来显示图像。 此选项假定你的视图的唯一内容是图片。 你仍然可以在图像视图上叠加其他视图以绘制其他控件或内容。 |
| 将图像显示为部分视图的装饰 | 使用 [UIImage](https://developer.apple.com/documentation/uikit/uiimage) 类加载和绘制图像。 |
| 将一些位图数据保存到图像对象中 | 你可以使用[使用位图图形上下文创建新图像](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/HandlingImages/Images.html#//apple_ref/doc/uid/TP40010156-CH13-SW8)中所述的 UIKit 函数或 Core Graphics 函数来执行此操作。 |
| 将图像保存为 JPEG 或 PNG 文件 | 从原始图像数据创建一个 `UIImage` 对象。 调用 [UIImageJPEGRepresentation](https://developer.apple.com/documentation/uikit/1624115-uiimagejpegrepresentation) 或 [UIImagePNGRepresentation](https://developer.apple.com/documentation/uikit/1624096-uiimagepngrepresentation) 函数以获取 NSData 对象，并使用该对象的方法将数据保存到文件。 |

## 系统支持图像

UIKit 框架以及 iOS 的底层系统框架为你提供了广泛的创建，访问，绘图，书写和操作图像的可能性。

### UIKit 图像类和函数

UIKit 框架有三个类和一个协议，它们以某种方式与图像相关联： 

[UIImage](https://developer.apple.com/documentation/uikit/uiimage)

这个类的对象表示 UIKit 框架中的图像。 你可以从几个不同的来源创建它们，包括文件和 Quartz 图像对象。 该类的方法使您可以使用不同的混合模式和不透明度值将图像绘制到当前图形上下文中。

UIImage 类会自动处理任何所需的转换，例如应用适当的比例因子（考虑到高分辨率显示），并且在给定 Quartz 图像时，修改图像的坐标系以使其与默认坐标系 UIKit（y 原点位于左上角）。

[UIImageView](https://developer.apple.com/documentation/uikit/uiimageview)

此类的对象是显示单个图像或动画显示一系列图像的视图。 如果图像是视图的唯一内容，请使用 UIImageView 类而不是绘制图像。

[UIImagePickerController](https://developer.apple.com/documentation/uikit/uiimagepickercontroller) 和 [UIImagePickerControllerDelegate](https://developer.apple.com/documentation/uikit/uiimagepickercontrollerdelegate)

这些类和协议为你的应用程序提供了获取用户提供的图像（照片）和电影的方法。 该类提供并管理用户界面，供你选择和拍摄照片和电影。 当用户选择一张照片时，它将所选的 UIImage 对象传递给 delegate，delegate 必须实现协议方法。

除了这些类之外，UIKit 还声明了可以调用以执行各种图像任务的函数：

- 绘制到图像支持的图形上下文中。 [UIGraphicsBeginImageContext](https://developer.apple.com/documentation/uikit/1623922-uigraphicsbeginimagecontext) 函数创建一个离屏位图图形上下文。 你可以在此图形上下文中绘制，然后从中提取 `UIImage` 对象。 （有关更多信息，请参见[绘制图像](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/HandlingImages/Images.html#//apple_ref/doc/uid/TP40010156-CH13-SW10)。）
- 获取或缓存图像数据。 每个 [UIImage](https://developer.apple.com/documentation/uikit/uiimage) 对象都有一个可直接访问的支持 Core Graphics 图像对象（CGImageRef）。 然后，你可以将 Core Graphics 对象传递给 Image I/O 框架以保存数据。 你还可以通过调用 [UIImagePNGRepresentation](https://developer.apple.com/documentation/uikit/1624096-uiimagepngrepresentation) 或 [UIImageJPEGRepresentation](https://developer.apple.com/documentation/uikit/1624115-uiimagejpegrepresentation) 函数将 UIImage 对象中的图像数据转换为 PNG 或 JPEG 格式。 然后可以访问数据对象中的字节，并且可以将图像数据写入文件。
- 将图像写入设备上的相册。 调用 UIImageWriteToSavedPhotosAlbum 函数，传入 UIImage 对象，将该图像放入设备上的相册中。

当你使用这些 UIKit 类和函数时，[Drawing Images](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/HandlingImages/Images.html#//apple_ref/doc/uid/TP40010156-CH13-SW10) 可以识别场景。

### 其他与图像相关的框架

你可以使用 UIKit 以外的多个系统框架来创建，访问，修改和编写图像。 如果你发现无法使用 UIKit 方法或函数完成特定的与图像相关的任务，则这些较低级别框架之一的功能可能可以做到你想要的功能。 其中一些功能可能需要 Core Graphics 图像对象（CGImageRef）。 你可以通过 CGImage 属性访问支持 UIImage 对象的 CGImageRef 对象。

> 注意：如果 UIKit 方法或函数存在以完成给定的与图像相关的任务，则应该使用它来代替任何相应的低级函数。

Quartz 的 Core Graphics 框架是最重要的底层系统框架。 它的几个函数对应于 UIKit 的函数和方法; 例如，一些 Core Graphics 函数允许你创建和绘制位图图形上下文，而另一些则允许你从各种来源创建图像。 但是，Core Graphics 为处理图像提供了更多选项。 借助 Core Graphics，你可以创建和应用图像蒙版，从现有图像的各个部分创建图像，应用色彩空间以及访问许多其他图像属性，包括每行的字节，每像素的位数和呈现意图。

Image I/O 框架与 Core Graphics 密切相关。 它允许应用程序读取和写入大多数图像文件格式，包括标准网页格式，高动态范围图像和原始相机数据。 它具有快速图像编码和解码，图像元数据和图像缓存。

Assets Library 是一个框架，允许应用访问由 Photos 应用管理的资源。 你可以通过表示（例如，PNG 或 JPEG）或 URL 来获取资产。 从表示或 URL 中，你可以获取 Core Graphics 图像对象或原始图像数据。 该框架还允许你将图像写入“保存的照片相册”。

### 支持的图像格式

表C-2列出了 iOS 直接支持的图像格式。 在这些格式中，PNG 格式是你的应用程序中最推荐使用的格式。 通常，UIKit 支持的图像格式与 Image I/O 框架支持的格式相同。

表 C-2 支持的图像格式
![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/UI/%E9%80%82%E7%94%A8%E4%BA%8E%20iOS%20%E7%9A%84%E7%BB%98%E5%9B%BE%E5%92%8C%E6%89%93%E5%8D%B0%E6%8C%87%E5%8D%97/%E9%80%82%E7%94%A8%E4%BA%8E%20iOS%20%E7%9A%84%E7%BB%98%E5%9B%BE%E5%92%8C%E6%89%93%E5%8D%B0%E6%8C%87%E5%8D%97%EF%BC%88%E4%B8%83%EF%BC%89%EF%BC%9A%E9%99%84%E5%BD%95%20C%20%E5%8A%A0%E8%BD%BD%E5%9B%BE%E7%89%87/%E8%A1%A8%20C-2%20%E6%94%AF%E6%8C%81%E7%9A%84%E5%9B%BE%E5%83%8F%E6%A0%BC%E5%BC%8F.png?raw=true)

## 保持图像质量

为你的用户界面提供高质量图像应该是你设计中的优先事项。 图像提供了一种合理有效的方式来显示复杂的图形，并应在适当的地方使用。 为你的应用程序创建图像时，请记住以下准则：

- 为图像使用 PNG 格式。 PNG 格式提供无损图像内容，这意味着将图像数据保存为 PNG 格式，然后再读取它们会得到完全相同的像素值。 PNG 还具有优化的存储格式，可以更快地读取图像数据。 这是 iOS 的首选图像格式。
- 创建图像，以便它们不需要调整大小。 如果你打算使用特定尺寸的图像，请确保以该尺寸创建相应的图像资源。 请勿创建较大的图像并将其缩小以适合尺寸，因为缩放需要额外的 CPU 周期并需要插值。 如果你需要以可变尺寸显示图像，请将图像的多个版本包含在不同的尺寸中，并从距离目标尺寸较近的图像缩小。
- 从不透明的 PNG 文件中删除 Alpha 通道。 如果 PNG 图像的每个像素都不透明，则删除 Alpha 通道可避免混合包含该图像的图层。 这大大简化了图像的合成并提高了绘图性能。

