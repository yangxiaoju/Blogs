# 提高动画性能

Core Animation 是改善基于应用的动画帧速率的好方法，但它的使用并不能保证提高性能。 特别是在 OS X 中，你仍然必须选择使用 Core Animation 行为的最有效方法。 就所有与性能相关的问题而言，你应该使用工具来衡量和跟踪应用程序的性能，以确保性能得到改善而不会退化。

## 为你的 OS X 视图选择最佳重绘策略

即使视图是层次支持的，[NSView](https://developer.apple.com/documentation/appkit/nsview) 类的默认重绘策略也会保留该类的原始绘图行为。 如果你在应用中使用图层支持的视图，则应检查重绘策略选项并选择能够为你的应用提供最佳性能的视图。 在大多数情况下，默认策略不是可能提供最佳性能的策略。 相反，[NSViewLayerContentsRedrawOnSetNeedsDisplayDisplay](https://developer.apple.com/documentation/appkit/nsviewlayercontentsredrawpolicy/nsviewlayercontentsredrawonsetneedsdisplay) 策略更有可能减少应用程序的绘制量并提高性能。 其他策略也可能为特定类型的视图提供更好的表现。

有关视图重绘策略的更多信息，请参阅 [OS X 视图的图层重绘策略影响性能](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/SettingUpLayerObjects/SettingUpLayerObjects.html#//apple_ref/doc/uid/TP40004514-CH13-SW22)。

## 更新 OS X 中的图层以优化渲染路径

在 OS X v10.8及更高版本中，视图有两个用于更新底层图层内容的选项。 当你更新 OS X v10.7及更早版本中的图层后备视图时，图层会将视图 drawRect: 方法中的绘图命令捕获到后台位图图像中。 缓存绘图命令是有效的，但在所有情况下都不是最有效的选项。 如果你知道如何直接提供图层的内容而不实际渲染它们，则可以使用 [updateLayer](https://developer.apple.com/documentation/appkit/nsview/1483580-updatelayer) 方法来实现。

有关渲染的不同路径（包括涉及 updateLayer 方法的路径）的信息，请参阅[使用委托来提供图层的内容](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/SettingUpLayerObjects/SettingUpLayerObjects.html#//apple_ref/doc/uid/TP40004514-CH13-SW14)。

## 一般提示和技巧

有几种方法可以使你的图层实现更高效。 然而，与任何这样的优化一样，在尝试优化之前，你应该始终测量代码的当前性能。 这为你提供了一个可以用来确定优化是否正常工作的基线。

### 尽可能使用不透明层

将图层的 opaqua 属性设置为 YES 可让 Core Animation 知道它不需要维护图层的 Alpha 通道。 没有 Alpha 通道意味着合成器不需要将图层的内容与背景内容混合，这可以节省渲染过程中的时间。 但是，此属性主要与作为层支持视图一部分的图层或 Core Animation 创建底层图位图的情况有关。 如果将图像直接分配给图层的内容属性，那么不管 opaque 属性中的值如何，该图像的 Alpha 通道都会保留。

## 为 CAShapeLayer 对象使用更简单的路径

CAShapeLayer 类通过在复合时间将你提供的路径渲染到位图图像中来创建其内容。优点是该层始终以尽可能最好的分辨率绘制路径，但这种优势需要花费额外的渲染时间。如果你提供的路径非常复杂，则对该路径进行栅格化可能会过于昂贵。如果图层大小频繁变化（因此必须频繁重绘），则绘图花费的时间可能会增加并成为性能瓶颈。

减少形状图层绘制时间的一种方法是将复杂的形状分解成更简单的形状。在合成器中使用更简单的路径并将多个 CAShapeLayer 对象叠加层叠可以比绘制一条大型复杂路径快得多。这是因为绘图操作发生在 CPU 上，而合成发生在 GPU 上。然而，与这种性质的任何简化一样，潜在的性能收益取决于你的内容。因此，在优化之前测量代码的性能尤为重要，以便你有用于比较的基准。

## 为相同图层明确设置图层内容

如果你在多个图层对象中使用相同的图像，请自行加载图像并将其直接分配给这些图层对象的内容属性。 将内容分配给 content 属性可防止该层为内存分配内存。 相反，该图层使用你提供的图像作为其后备存储。 当几个图层使用相同的图像时，这意味着所有这些图层共享相同的内存，而不是为自己分配图像副本。

## 始终将图层大小设置为积分值

为获得最佳效果，请始终将图层对象的宽度和高度设置为整数值。 虽然使用浮点数指定图层边界的宽度和高度，但最终将使用图层边界来创建位图图像。 指定宽度和高度的整数值简化了 Core Animation 必须执行的工作，以创建和管理后备存储和其他图层信息。

## 根据需要使用异步图层呈现

你在委托的 drawLayer 中执行的任何 [drawLayer:inContext:](https://developer.apple.com/documentation/quartzcore/calayerdelegate/2097262-drawlayer) 方法或你的视图的 drawRect: 方法通常在应用程序的主线程中同步发生。 但是，在某些情况下，同步绘制内容可能无法提供最佳性能。 如果你注意到你的动画效果不佳，则可以尝试在图层上启用 [drawsAsynchronously](https://developer.apple.com/documentation/quartzcore/calayer/1410974-drawsasynchronously) 属性，将这些操作移至后台线程。 如果你这样做，确保你的绘图代码是线程安全的。 与往常一样，在将其放入生产代码之前，应始终测量绘图的性能。

## 向图层添加阴影时指定阴影路径

让 Core Animation 确定阴影的形状可能会很昂贵，并会影响应用程序的性能。 不是让 Core Animation 确定阴影的形状，而是使用 CALayer 的 [shadowPath](https://developer.apple.com/documentation/quartzcore/calayer/1410771-shadowpath) 属性明确指定阴影形状。 当你为此属性指定路径对象时，Core Animation 将使用该形状绘制和缓存阴影效果。 对于形状永远不会改变或很少改变的图层，这可以通过减少 Core Animation 完成的渲染量来大大提高性能。

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~