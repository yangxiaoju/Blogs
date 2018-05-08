# 绘制和创建图像

大多数情况下，使用标准视图显示图像相当简单。但是，有两种情况需要你做额外的工作：

- 如果要将图像显示为自定义视图的一部分，则必须自己在视图的 [drawRect:](https://developer.apple.com/documentation/uikit/uiview/1622529-drawrect) 方法中绘制图像。[绘制图像](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/HandlingImages/Images.html#//apple_ref/doc/uid/TP40010156-CH13-SW10)解释了怎么做。
- 如果要将图像渲染到屏幕外（以便稍后绘制或保存到文件中），则必须创建位图图像上下文。要了解更多信息，请阅读[使用位图图形上下文创建新图像](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/HandlingImages/Images.html#//apple_ref/doc/uid/TP40010156-CH13-SW8)。

## 绘制图像

为了获得最佳性能，如果使用 `UIImageView` 类可以满足图像绘制需求，则应该使用此图像对象来初始化 `UIImageView` 对象。但是，如果你需要明确绘制图像，则可以存储图像并稍后在视图的 `drawRect:` 方法中使用它。

以下示例显示如何从应用程序包中加载图像。

```
NSString *imagePath = [[NSBundle mainBundle] pathForResource:@"myImage" ofType:@"png"];
UIImage *myImageObj = [[UIImage alloc] initWithContentsOfFile:imagePath];
 
// Store the image into a property of type UIImage *
// for use later in the class's drawRect: method.
self.anImage = myImageObj;
```

要在视图的 drawRect: 方法中显式地绘制生成的图像，你可以使用 UIImage 中提供的任何绘图方法。这些方法允许你指定要在图像中绘制图像的位置，因此不需要在绘制之前创建和应用单独的转换。

下面的代码片段将在视图中的点（10,10）处绘制上面加载的图像。

```
- (void)drawRect:(CGRect)rect
{
    ...
 
    // Draw the image.
    [self.anImage drawAtPoint:CGPointMake(10, 10)];
}
```

> 重要提示：如果你使用 [CGContextDrawImage](https://developer.apple.com/documentation/coregraphics/1454845-cgcontextdrawimage) 函数直接绘制位图图像，默认情况下图像数据会沿着 y 轴反转。这是因为 Quartz 图像假设一个坐标系统，其左下角原点和正坐标轴向上和向右延伸。虽然你可以在绘制之前应用变换，但更简单（推荐）的绘制 Quartz 图像的方法是将它们包裹在 UIImage 对象中，这会自动补偿坐标空间中的这种差异。有关使用 Core Graphics 创建和绘制图像的更多信息，请参阅 [Quartz 2D Programming Guide](https://developer.apple.com/library/content/documentation/GraphicsImaging/Conceptual/drawingwithquartz2d/Introduction/Introduction.html#//apple_ref/doc/uid/TP30001066)。

## 使用位图图形上下文创建新图像

大多数情况下，绘图时，你的目标是在屏幕上显示某些内容。但是，将某些内容绘制到屏幕外缓冲区有时很有用。例如，你可能希望创建现有图像的缩略图，将其绘制到缓冲区中，以便将其保存到文件中，等等。为了支持这些需求，你可以创建位图图像上下文，使用 UIKit 框架或 Core Graphics 函数来绘制它，然后从上下文中获取图像对象。

在 UIKit 中，过程如下：

1、调用 [UIGraphicsBeginImageContextWithOptions](https://developer.apple.com/documentation/uikit/1623912-uigraphicsbeginimagecontextwitho) 来创建位图上下文并将其推送到图形堆栈上。

对于第一个参数（`size`），传递一个 [CGSize](https://developer.apple.com/documentation/coregraphics/cgsize) 值来指定位图上下文的维度（以点为单位）。

对于第二个参数（`opaque`），如果图像包含透明度（alpha 通道），则传递NO。否则，传递YES以最大化性能。

对于最后一个参数（`scale`），为设备主屏幕适当缩放的位图传递0.0，或传递你选择的比例因子。

例如，下面的代码片段会创建一个200 x 200像素的位图。 （像素数量是通过将图像尺寸乘以比例因子确定的。）

```
UIGraphicsBeginImageContextWithOptions(CGSizeMake(100.0,100.0), NO, 2.0);
```

注意：通常应避免调用类似名称的 `UIGraphicsBeginImageContext` 函数（除了作为向后兼容的后备），因为它总是会创建比例因子为1.0的图像。如果底层设备具有高分辨率屏幕，则使用 `UIGraphicsBeginImageContext` 创建的图像在渲染时可能不会显示为平滑。

2、使用 UIKit 或 Core Graphics 例程将图像的内容绘制到新创建的图形上下文中。

3、调用 [UIGraphicsGetImageFromCurrentImageContext](https://developer.apple.com/documentation/uikit/1623924-uigraphicsgetimagefromcurrentima) 函数以根据你绘制的内容生成并返回 [UIImage](https://developer.apple.com/documentation/uikit/uiimage) 对象。如果需要，你可以继续绘制并再次调用此方法以生成其他图像。

4、调用 [UIGraphicsEndImageContext](https://developer.apple.com/documentation/uikit/1623933-uigraphicsendimagecontext) 从图形栈中弹出上下文。

清单3-1中的方法获取通过 Internet 下载的图像，并将其绘制到基于图像的上下文中，缩小为应用程序图标的大小。然后它获得从位图数据创建的 UIImage 对象并将其分配给实例变量。请注意，位图的大小（UIGraphicsBeginImageContextWithOptions 的第一个参数）和绘制内容的大小（imageRect 的大小）应该匹配。如果内容大于位图，则内容的一部分将被裁剪并且不会出现在结果图像中。

清单 3-1 将缩小的图像绘制到位图上下文并获取生成的图像
```
- (void)connectionDidFinishLoading:(NSURLConnection *)connection {
    UIImage *image = [[UIImage alloc] initWithData:self.activeDownload];
    if (image != nil && image.size.width != kAppIconHeight && image.size.height != kAppIconHeight) {
        CGRect imageRect = CGRectMake(0.0, 0.0, kAppIconHeight, kAppIconHeight);
        UIGraphicsBeginImageContextWithOptions(itemSize, NO, [UIScreen mainScreen].scale);
        [image drawInRect:imageRect];
        self.appRecord.appIcon = UIGraphicsGetImageFromCurrentImageContext();  // UIImage returned.
        UIGraphicsEndImageContext();
    } else {
        self.appRecord.appIcon = image;
    }
    self.activeDownload = nil;
    [image release];
    self.imageConnection = nil;
    [delegate appImageDidLoad:self.indexPathInTableView];
}
```


你也可以调用 Core Graphics 函数来绘制生成的位图图像的内容;清单3-2中的代码段描绘了一个 PDF 页面的缩小图像，给出了一个例子。请注意，在调用 CGContextDrawPDFPage 以将绘制的图像与 UIKit 的默认坐标系对齐之前，代码将翻转图形上下文。

清单 3-2 使用 Core Graphics 函数绘制位图上下文
```
// Other code precedes...
 
CGRect pageRect = CGPDFPageGetBoxRect(page, kCGPDFMediaBox);
pdfScale = self.frame.size.width/pageRect.size.width;
pageRect.size = CGSizeMake(pageRect.size.width * pdfScale, pageRect.size.height * pdfScale);
UIGraphicsBeginImageContextWithOptions(pageRect.size, YES, pdfScale);
CGContextRef context = UIGraphicsGetCurrentContext();
 
// First fill the background with white.
CGContextSetRGBFillColor(context, 1.0,1.0,1.0,1.0);
CGContextFillRect(context,pageRect);
CGContextSaveGState(context);
 
// Flip the context so that the PDF page is rendered right side up
CGContextTranslateCTM(context, 0.0, pageRect.size.height);
CGContextScaleCTM(context, 1.0, -1.0);
 
// Scale the context so that the PDF page is rendered at the
// correct size for the zoom level.
CGContextScaleCTM(context, pdfScale,pdfScale);
CGContextDrawPDFPage(context, page);
CGContextRestoreGState(context);
UIImage *backgroundImage = UIGraphicsGetImageFromCurrentImageContext();
UIGraphicsEndImageContext();
backgroundImageView = [[UIImageView alloc] initWithImage:backgroundImage];
 
// Other code follows...
```

如果你更喜欢完全使用 Core Graphics 来绘制位图图形上下文，则可以使用 CGBitmapContextCreate 函数来创建上下文并在其中绘制图像内容。 完成绘图时，调用 CGBitmapContextCreateImage 函数以从位图上下文中获取 CGImageRef 对象。 你可以直接绘制核心图形图像或使用它来初始化 UIImage 对象。 完成后，在图形上下文中调用 CGContextRelease 函数。