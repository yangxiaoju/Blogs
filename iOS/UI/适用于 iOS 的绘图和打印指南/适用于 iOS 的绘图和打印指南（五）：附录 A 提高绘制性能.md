# 提高绘制性能

在任何平台上绘图都是相对昂贵的操作，优化绘图代码应始终是开发过程中的重要一步。 表 A-1 列出了确保你的绘图代码尽可能优化的几个提示。 除了这些提示之外，你应该始终使用可用的性能工具来测试代码并消除热点和冗余。

表 A-1 提高绘图性能的提示
| Tip | Action |
|-|-|
| 最低限度绘制 | 在每个更新周期中，你只应更新实际更改的视图部分。 如果使用 UIView 的 [drawRect:](https://developer.apple.com/documentation/uikit/uiview/1622529-drawrect) 方法执行绘制，请使用传递给该方法的更新矩形来限制绘图的范围。 对于 OpenGL 绘图，你必须自己跟踪更新。 |
| 明智地调用 [setNeedsDisplay:](https://developer.apple.com/documentation/appkit/nsview/1483360-needsdisplay) | 如果你正在调用 [setNeedsDisplay:](https://developer.apple.com/documentation/appkit/nsview/1483360-needsdisplay) ,则应始终花时间计算实际需要重绘的区域。 不要只传递一个包含整个视图的矩形。 另外，不要调用 [setNeedsDisplay:](https://developer.apple.com/documentation/appkit/nsview/1483360-needsdisplay) 除非实际需要重绘内容。 如果内容没有真正改变，请不要重绘。 |
| 标记视图为不透明 | 合成一个视图的内容是不透明的，比合成一个部分透明的视图需要更少的工作。 要使视图不透明，视图的内容不得包含任何透明度，视图的 [opaqua](https://developer.apple.com/documentation/uikit/uiview/1622622-opaque) 属性必须设置为 YES。 |
| 在滚动期间重用表格单元格和视图 | 应该不惜一切代价避免在滚动期间创建新视图。 花时间创建新视图会减少可用于更新屏幕的时间，这会导致滚动行为不均匀。 |
| 通过修改当前变换矩阵来重用路径 | 通过修改当前转换矩阵，可以使用单个路径在屏幕的不同部分上绘制内容。 有关详细信息，请参阅[使用坐标变换提高绘图性能](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/GraphicsDrawingOverview/GraphicsDrawingOverview.html#//apple_ref/doc/uid/TP40010156-CH14-SW4)。 |
| 在滚动期间避免清除以前的内容 | 默认情况下，UIKit 在调用其 [drawRect:](https://developer.apple.com/documentation/uikit/uiview/1622529-drawrect) 方法来更新同一区域之前清除视图的当前上下文缓冲区。 如果你在视图中响应滚动事件，则在滚动更新期间重复清除此区域可能会很昂贵。 要禁用该行为，可以将 [clearsContextBeforeDrawing](https://developer.apple.com/documentation/uikit/uiview/1622449-clearscontextbeforedrawing) 属性中的值更改为 NO。 |
| 在绘图时最小化图形状态更改 | 更改图形状态需要底层图形子系统的工作。 如果你需要绘制使用类似状态信息的内容，请尝试一起绘制该内容以减少所需状态更改的数量。 |
| 使用工具来调试你的性能 | Core Animation instrument 可以帮助你发现应用程序中的绘图性能问题。 尤其是：通过 Flash 更新区域可以轻松查看视图的哪些部分实际上正在更新。 颜色错位图像可帮助你看到排列不良的图像，从而导致模糊图像和较差的性能。 有关更多信息，请参见[Instruments 用户指南](https://developer.apple.com/library/content/documentation/DeveloperTools/Conceptual/InstrumentsUserGuide/index.html#//apple_ref/doc/uid/TP40004652)中的[测量 iOS 设备中的图形性能](https://developer.apple.com/library/content/documentation/DeveloperTools/Conceptual/InstrumentsUserGuide/ExportingandImportingTraceData.html#//apple_ref/doc/uid/TP40004652-CH14)。|  