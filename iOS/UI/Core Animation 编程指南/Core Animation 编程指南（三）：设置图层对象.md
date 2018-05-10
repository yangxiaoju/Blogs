# 设置图层对象

图层对象是 Core Animation 的核心。 图层管理你的应用的视觉内容，并提供修改该内容的风格和视觉外观的选项。 虽然 iOS 应用程序已自动启用图层支持，但 OS X 应用程序的开发人员必须先明确启用它，才能利用性能优势。 启用后，你需要了解如何配置和操作应用的图层以获得所需的效果。

## 在你的应用程序中启用 Core Animation 支持

在 iOS 应用程序中，Core Animation 始终处于启用状态，并且每个视图都由一个图层支持。 在 OS X 中，应用程序必须通过执行以下操作来显式启用 Core Animation 支持：
- 链接到 QuartzCore 框架。 （只有在明确使用 Core Animation 接口的情况下，iOS 应用程序才必须与此框架链接。）
- 通过执行以下任一操作，为一个或多个 [NSView](https://developer.apple.com/documentation/appkit/nsview) 对象启用图层支持：
    - 在你的 nib 文件中，使用 View Effects 检查器为你的视图启用图层支持。 检查员显示所选视图及其子视图的复选框。 建议你尽可能在窗口的内容视图中启用图层支持。
    - 对于以编程方式创建的视图，请调用视图的 [setWantsLayer:](https://developer.apple.com/documentation/appkit/nsview/1483695-wantslayer) 方法并传递值 YES 以指示视图应使用图层。

以上述方式之一启用图层支持可创建图层支持的视图。 通过分层视图，系统负责创建底层对象并保持该层的更新。 在 OS X 中，也可以创建一个图层托管视图，其中你的应用程序实际上创建并管理底层图层对象。 （你不能在 iOS 中创建图层托管视图。）有关如何创建图层托管视图的更多信息，请参阅[图层托管允许你在 OS X 中更改图层对象](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/SettingUpLayerObjects/SettingUpLayerObjects.html#//apple_ref/doc/uid/TP40004514-CH13-SW2)。

## 更改与视图关联的图层对象

默认情况下，支持图层的视图会创建 CALayer 类的实例，并且在大多数情况下，你可能不需要不同类型的图层对象。 但是，Core Animation 提供了不同的图层类，每个图层类都提供了你可能会觉得有用的专用功能。 选择不同的图层类可能使你能够以简单的方式提高性能或支持特定类型的内容。 例如，CATiledLayer 类针对以有效方式显示大图像进行了优化。

### 更改 UIView 使用的图层类

你可以通过覆盖视图的 [layerClass](https://developer.apple.com/documentation/uikit/uiview/1622626-layerclass) 方法并返回不同的类对象来更改 iOS 视图使用的图层类型。 大多数 iOS 视图创建一个 [CALayer](https://developer.apple.com/documentation/quartzcore/calayer) 对象，并将该图层用作其内容的后备存储。 对于你自己的大部分视图，这个默认选择是一个很好的选择，你不需要改变它。 但是你可能会发现在某些情况下，不同的图层类更合适。 例如，你可能想在以下情况下更改图层类：

- 你的视图使用 Metal 或 OpenGL ES 绘制内容，在这种情况下，你将使用 [CAMetalLayer](https://developer.apple.com/documentation/quartzcore/cametallayer) 或 [CAEAGLLayer](https://developer.apple.com/documentation/quartzcore/caeagllayer) 对象。
- 有一个专门的图层类可以提供更好的性能。
- 你想要利用一些专门的核心动画层类，如粒子发射器或复制器。

更改视图的图层类非常简单; 清单2-1显示了一个例子。 你所要做的就是重写 layerClass 方法，并返回你想要使用的类对象。 在显示之前，视图调用 layerClass 方法并使用返回的类为其自身创建一个新的图层对象。 一旦创建，视图的图层对象就不能改变。

清单 2-1 指定 iOS 视图的图层类
```
+ (Class) layerClass {
   return [CAMetalLayer class];
}
```

有关图层类的列表以及如何使用它们，请参阅[不同的图层类提供专门的行为](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/SettingUpLayerObjects/SettingUpLayerObjects.html#//apple_ref/doc/uid/TP40004514-CH13-SW25)。

### 更改 NSView 使用的图层类

你可以通过重写 [makeBackingLayer](https://developer.apple.com/documentation/appkit/nsview/1483687-makebackinglayer) 方法来更改 NSView 对象使用的默认图层类。 在实现此方法时，创建并返回你想要 AppKit 使用的图层对象来备份你的自定义视图。 在你想要使用自定义图层（如滚动或平铺图层）的情况下，你可以重写此方法。

有关图层类的列表以及如何使用它们，请参阅[不同的图层类提供专门的行为](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/SettingUpLayerObjects/SettingUpLayerObjects.html#//apple_ref/doc/uid/TP40004514-CH13-SW25)。

### 图层托管允许你更改 OS X 中的图层对象

图层托管视图是你自己创建和管理基础图层对象的 [NSView](https://developer.apple.com/documentation/appkit/nsview) 对象。 你可以在要控制与视图关联的图层对象类型的情况下使用图层托管。 例如，你可以创建一个图层托管视图，以便你可以分配默认 CALayer 类以外的图层类。 你也可以在需要使用单个视图来管理独立层的层次结构的情况下使用它。

当你调用你的视图的 [setLayer:](https://developer.apple.com/documentation/appkit/nsview/1483298-layer) 方法并提供一个图层对象时，AppKit 会采取一种不干涉的方法来处理该图层。 通常，AppKit 更新视图的图层对象，但在图层托管情况下，它不适用于大多数属性。

要创建图层托管视图，请在屏幕上显示视图之前创建图层对象并将其与视图相关联，如清单2-2所示。 除了设置图层对象之外，还必须调用 [setWantsLayer:](https://developer.apple.com/documentation/appkit/nsview/1483695-wantslayer) 方法来让视图知道它应该使用图层。

清单 2-2 创建一个图层托管视图
```
// Create myView...
 
[myView setWantsLayer:YES];
CATiledLayer* hostedLayer = [CATiledLayer layer];
[myView setLayer:hostedLayer];
 
// Add myView to a view hierarchy.
```

如果你选择自己托管图层，则必须自行设置 [contentsScale](https://developer.apple.com/documentation/quartzcore/calayer/1410746-contentsscale) 属性并在适当的时间提供高分辨率内容。 有关高分辨率内容和缩放因子的更多信息，请参阅[使用高分辨率图像](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/SettingUpLayerObjects/SettingUpLayerObjects.html#//apple_ref/doc/uid/TP40004514-CH13-SW6)。

### 不同的图层类提供专门的行为

Core Animation 定义了许多标准的图层类，每个类都是为特定用例而设计的。 CALayer 类是所有图层对象的根类。 它定义了所有图层对象必须支持的行为，并且是层级视图所使用的默认类型。 但是，你也可以在表2-1中指定其中一个图层类。

表 2-1 CALayer 子类及其用法
| Class | Usage |
|-|-|
| [CAEmitterLayer](https://developer.apple.com/documentation/quartzcore/caemitterlayer) | 用于实现基于 Core Animation 的粒子发射器系统。 发射器层对象控制粒子的生成及其原点。 |
| [CAGradientLayer](https://developer.apple.com/documentation/quartzcore/cagradientlayer) | 用于绘制填充图层形状的颜色渐变（在任何圆角的范围内）。 |
| [CAMetalLayer](https://developer.apple.com/documentation/quartzcore/cametallayer) | 用于使用 Metal 设置和提供用于渲染图层内容的可绘制纹理。 |
| [CAEAGLLayer](https://developer.apple.com/documentation/quartzcore/caeagllayer)/[CAOpenGLLayer](https://developer.apple.com/documentation/quartzcore/caopengllayer) | 用于设置使用 OpenGL ES（iOS）或 OpenGL（OS X）呈现图层内容的后备存储和上下文。 |
| [CAReplicatorLayer](https://developer.apple.com/documentation/quartzcore/careplicatorlayer) | 当你想自动复制一个或多个子图层时使用。 复制器为你制作副本，并使用你指定的属性来更改副本的外观或属性。 |
| [CAScrollLayer](https://developer.apple.com/documentation/quartzcore/cascrolllayer) | 用于管理由多个子层组成的大型可滚动区域。 |
| [CAShapeLayer](https://developer.apple.com/documentation/quartzcore/cashapelayer) | 用于绘制一个三次贝塞尔样条曲线。 形状图层对于绘制基于路径的形状是有利的，因为它们总是会形成一条清晰的路径，与绘制到图层后台存储区的路径相反，在缩放时看起来不太好。 但是，清晰的结果确实涉及渲染主线程上的形状并缓存结果。 |
| [CATextLayer](https://developer.apple.com/documentation/quartzcore/catextlayer) | 用于呈现纯文本或属性字符串。 |
| [CATiledLayer](https://developer.apple.com/documentation/quartzcore/catiledlayer) | 用于管理大图像，该图像可以分成更小的图块并单独渲染，支持放大和缩小内容。 |
| [CATransformLayer](https://developer.apple.com/documentation/quartzcore/catransformlayer) | 用于渲染真正的 3D 图层层次结构，而不是其他图层类实现的拼合图层层次结构。 |
| [QCCompositionLayer](https://developer.apple.com/documentation/quartz/qccompositionlayer) | 用于渲染 Quartz Composer 组合。 （仅限 OS X） |

## 提供图层的内容

图层是管理你的应用提供的内容的数据对象。 图层的内容由包含要显示的可视化数据的位图组成。 你可以通过以下三种方式之一提供该位图的内容：

- 直接将图像对象分配给图层对象的 [contents](https://developer.apple.com/documentation/quartzcore/calayer/1410773-contents) 属性。 （这种技术最适用于从未或很少改变的图层内容。）
- 将 delegate 对象分配给图层，并让 delegate 绘制图层的内容。 （这种技术最适合可能定期更改的图层内容，并且可以由外部对象（如视图）提供。）
- 定义一个图层子类并重写其中一个绘图方法以自行提供图层内容。 （如果你必须创建自定义图层子类，或者要更改图层的基本绘制行为，则此技术是适当的。）

唯一需要担心为图层提供内容的时间是你自己创建图层对象时。如果你的应用程序仅包含层次支持的视图，则不必担心使用上述任何技术提供图层内容。图层支持的视图以尽可能高效的方式自动为其关联图层提供内容。

### 使用图像作为图层的内容

由于图层只是用于管理位图图像的容器，因此可以将图像直接分配给图层的 [contents](https://developer.apple.com/documentation/quartzcore/calayer/1410773-contents) 属性。 为图层分配图像非常简单，你可以指定想要在屏幕上显示的确切图像。 该图层使用你直接提供的图像对象，并且不会尝试创建该图像的自己的副本。 如果你的应用在多个位置使用相同的图像，此行为可以节省内存。

你分配给图层的图像必须是 [CGImageRef](https://developer.apple.com/documentation/coregraphics/cgimageref) 类型。 （在OS X v10.6及更高版本中，你也可以分配 [NSImage](https://developer.apple.com/documentation/appkit/nsimage) 对象。）分配图像时，请记住提供分辨率与本机设备分辨率相匹配的图像。 对于具有 Retina 显示器的设备，这可能还需要你调整图像的 [contentsScale](https://developer.apple.com/documentation/quartzcore/calayer/1410746-contentsscale) 属性。 有关在图层中使用高分辨率内容的信息，请参阅[使用高分辨率图像](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/SettingUpLayerObjects/SettingUpLayerObjects.html#//apple_ref/doc/uid/TP40004514-CH13-SW6)。

### 使用 Delegate 来提供图层的内容

如果图层的内容动态更改，则可以使用 delegate 对象在需要时提供和更新该内容。 在显示时间，图层会调用 delegate 的方法来提供所需的内容：

- 如果 delegate 实现 [displayLayer:](https://developer.apple.com/documentation/quartzcore/calayerdelegate/2097261-display) 方法，则该实现负责创建位图并将其分配给图层的 [contents](https://developer.apple.com/documentation/quartzcore/calayer/1410773-contents) 属性。
- 如果你的 delegate 实现了 [drawLayer:inContext:](https://developer.apple.com/documentation/quartzcore/calayerdelegate/2097262-drawlayer) 方法，Core Animation 将创建一个位图，创建一个图形上下文以绘制该位图，然后调用 delegate 方法来填充位图。 你所有的委托方法所要做的就是绘制到提供的图形上下文中。

Delegate 对象必须实现 `displayLayer:` 或 `drawLayer:inContext:` 方法。 如果委托实现 `displayLayer:` 和 `drawLayer:inContext:` 方法，则该层仅调用 `displayLayer:` 方法。

覆盖 `displayLayer:` 方法最适用于应用程序偏好加载或创建要显示的位图的情况。 清单2-3显示了 `displayLayer:` delegate 方法的示例实现。 在这个例子中，delegate 使用一个辅助对象来加载和显示它所需要的图像。 delegate 方法根据自己的内部状态选择要显示哪个图像，在本例中是一个名为 `displayYesImage` 的自定义属性。

清单 2-3 直接设置图层内容
```
- (void)displayLayer:(CALayer *)theLayer {
    // Check the value of some state property
    if (self.displayYesImage) {
        // Display the Yes image
        theLayer.contents = [someHelperObject loadStateYesImage];
    }
    else {
        // Display the No image
        theLayer.contents = [someHelperObject loadStateNoImage];
    }
}
```

如果你没有预渲染的图像或助手对象为你创建位图，则你的委托可以使用 drawLayer:inContext: 方法动态地绘制内容。 清单2-4显示了 drawLayer:inContext: 方法的示例实现。 在此示例中，delegate 使用固定宽度和当前渲染颜色绘制简单的曲线路径。

清单 2-4 绘制一个图层的内容
```
- (void)drawLayer:(CALayer *)theLayer inContext:(CGContextRef)theContext {
    CGMutablePathRef thePath = CGPathCreateMutable();
 
    CGPathMoveToPoint(thePath,NULL,15.0f,15.f);
    CGPathAddCurveToPoint(thePath,
                          NULL,
                          15.f,250.0f,
                          295.0f,250.0f,
                          295.0f,15.0f);
 
    CGContextBeginPath(theContext);
    CGContextAddPath(theContext, thePath);
 
    CGContextSetLineWidth(theContext, 5);
    CGContextStrokePath(theContext);
 
    // Release the path
    CFRelease(thePath);
}
```

对于具有自定义内容的图层支持的视图，你应该继续覆盖视图的方法来执行绘制。 层次支持的视图自动将其自己作为其图层的 delegate 并实现所需的 delegate 方法，并且不应更改该配置。 相反，你应该实现你的视图的 drawRect: 方法来绘制你的内容。

在 OS X v10.8 及更高版本中，替代绘图的方法是通过覆盖视图的 [wantsUpdateLayer](https://developer.apple.com/documentation/appkit/nsview/1483461-wantsupdatelayer) 和 [updateLayer](https://developer.apple.com/documentation/appkit/nsview/1483580-updatelayer) 方法来提供位图。 重写 wantsUpdateLayer 并返回 YES 会导致 NSView 类遵循备用渲染路径。 视图不会调用 drawRect:，而是调用 updateLayer 方法，该方法的实现必须将位图直接分配给图层的 [contents](https://developer.apple.com/documentation/quartzcore/calayer/1410773-contents) 属性。 这是 AppKit 希望直接设置视图的图层对象内容的场景之一。

### 通过子类提供图层内容

如果你正在实现自定义图层类，则可以覆盖图层类的绘图方法以执行任何绘图。 图层对象本身生成自定义内容的情况并不常见，但图层当然可以管理内容的显示。 例如，CATiledLayer 类通过将较大的图像拆分为可以单独管理和渲染的较小图块来管理大图像。 因为只有图层具有关于在任何给定时间需要渲染哪些图块的信息，所以它直接管理图形行为。

在继承子类时，可以使用以下任一技术来绘制图层的内容：

- 覆盖图层的 [display](https://developer.apple.com/documentation/quartzcore/calayer/1410926-display) 方法，并使用它直接设置图层的 [contents](https://developer.apple.com/documentation/quartzcore/calayer/1410773-contents) 属性。
- 覆盖图层的 [drawInContext:](https://developer.apple.com/documentation/quartzcore/calayer/1410757-drawincontext) 方法并使用它绘制到提供的图形上下文中。

你重写的方法取决于你在绘图过程中需要多少控制。 显示方法是更新图层内容的主要入口点，因此覆盖该方法可使你完全控制过程。 覆盖显示方法也意味着你负责创建要分配给 content 属性的 [CGImageRef](https://developer.apple.com/documentation/coregraphics/cgimageref)。 如果你只想绘制内容（或者让图层管理绘图操作），则可以改为重写 drawInContext: 方法，并让图层为你创建后备存储。

### 调整你提供的内容

将图像分配给图层的 [contents](https://developer.apple.com/documentation/quartzcore/calayer/1410773-contents) 属性时，图层的 [contentsGravity](https://developer.apple.com/documentation/quartzcore/calayer/1410872-contentsgravity) 属性决定如何处理该图像以适应当前边界。 默认情况下，如果图像大于或小于当前边界，图层对象将缩放图像以适合可用空间。 如果图层边界的高宽比与图像的高宽比不同，则会导致图像失真。 你可以使用 contentsGravity 属性来确保以最佳方式呈现你的内容。

- 基于 position 的 gravity 常量允许你将图像固定到图层边界矩形的特定边或角，而不缩放图像。
- 基于 scaling 的 gravity 常量可让你使用多种选项中的一种来拉伸图像，其中一些选项可保留高宽比，其中一些选项不会。

图2-1显示了基于 position 的 gravity 常量如何影响图像。 除 kCAGravityCenter 常量外，每个常量都会将图像固定到图层边界矩形的特定边或角点。 kCAGravityCenter 常数将图层置于图层的中心。 这些常量都不会导致图像以任何方式缩放，因此图像总是以原始大小呈现。 如果图像大于图层边界，则可能会导致图像部分被裁剪，如果图像较小，图层未覆盖的图层部分会显示图层的背景颜色（如果已设置）。

图 2-1 图层基于 position 的 gravity 常量
![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/layer_contentsgravity1_2x.png)

图2-2显示了基于 scaling 的 gravity 常量如何影响图像。 所有这些常量都会缩放图像，如果它不完全适合图层的边界矩形。 模式之间的区别在于它们如何处理图像的原始高宽比。 一些模式保留它，而其他模式不保留。 默认情况下，图层的 contentsGravity 属性设置为 [kCAGravityResize](https://developer.apple.com/documentation/quartzcore/kcagravityresize) 常量，这是唯一不保留图像宽高比的模式。

图 2-2 图层的基于 scaling 的 gravity 常量
![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/positioningmask_2x.png)

### 处理高分辨率图像

图层对底层设备的屏幕分辨率没有任何固有的知识。一个图层只是存储一个指向你的位图的指针，并以给定可用像素的最佳方式显示它。如果将图像分配给图层的 [contents](https://developer.apple.com/documentation/quartzcore/calayer/1410773-contents) 属性，则必须通过将图层的 [contentsScale](https://developer.apple.com/documentation/quartzcore/calayer/1410746-contentsscale) 属性设置为适当的值来告诉 Core Animation 关于图像的分辨率。该属性的默认值为1.0，这适用于打算在标准分辨率屏幕上显示的图像。如果你的图像用于 Retina 显示，请将此属性的值设置为2.0。

只有在你直接为你的图层分配位图时，才需要更改 contentsScale 属性的值。 UIKit 和 AppKit 中的图层支持视图根据屏幕分辨率和视图管理的内容自动将图层的比例因子设置为适当的值。例如，如果你将 `NSImage`(https://developer.apple.com/documentation/appkit/nsimage) 对象分配给 OS X 中图层的内容属性，则 AppKit 会查看是否存在图像的标准和高分辨率变体。如果有，AppKit 使用当前分辨率的正确变体并设置 contentScale 属性的值以匹配。

在 OS X 中，基于基于 position 的 gravity 常量影响从分配给图层的 NSImage 对象中选择图像表示的方式。由于这些常量不会导致缩放图像，因此 Core Animation 依赖 contentsScale 属性来选择具有最适合像素密度的图像表示。

在 OS X 中，图层的 delegate 可以实现图层：shouldInheritContentsScale:fromWindow: 方法并使用它来响应比例因子的变化。只要给定窗口的分辨率发生变化，AppKit 就会自动调用该方法，这可能是因为窗口在标准分辨率和高分辨率屏幕之间移动。如果代表支持更改图层图像的分辨率，则此方法的实现应返回 YES。该方法应该根据需要更新图层的内容以反映新的分辨率。

## 调整图层的视觉风格和外观

图层对象内置了可以用来补充图层主要内容的可视化装饰，例如边框和背景颜色。 由于这些视觉装饰并不需要你的任何渲染，所以它们可以在某些情况下将图层作为独立实体使用。 你所要做的就是在图层上设置一个属性，图层处理必要的图形，包括任何动画。 有关这些视觉装饰物如何影响图层外观的其他说明，请参见[图层样式属性动画](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/LayerStyleProperties/LayerStyleProperties.html#//apple_ref/doc/uid/TP40004514-CH10-SW1)。

### 图层有他们自己的背景和边界

除了基于图像的内容之外，图层还可以显示填充背景和描边边框。 背景颜色在图层内容图像后面呈现，边框呈现在图像顶部，如图2-3所示。 如果图层包含子图层，则图层也会出现在边框下方。 由于背景颜色位于图像后面，因此该颜色会透过图像的任何透明部分。

图 2-3 为图层添加边框和背景

![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/layer_border_background_2x.png)

清单 2-5 显示了设置图层的背景颜色和边框所需的代码。 所有这些属性都是可以动画的。

清单 2-5 设置图层的背景颜色和边框
```
myLayer.backgroundColor = [NSColor greenColor].CGColor;
myLayer.borderColor = [NSColor blackColor].CGColor;
myLayer.borderWidth = 3.0;
```

> 注意：你可以将任何类型的颜色用于图层的背景，包括具有透明度的颜色或使用图案图像。 但是，在使用图案图像时，请注意 Core Graphics 处理图案图像的渲染，并使用其标准坐标系进行处理，这与 iOS 中的默认坐标系不同。 因此，除非你翻转坐标，否则 iOS 上呈现的图像默认情况下会颠倒。

如果你将图层的背景色设置为不透明颜色，请考虑将图层的 opaqua 属性设置为 YES。 这样做可以在合成屏幕上的图层时提高性能，并且不需要图层的后台存储来管理 Alpha 通道。 不过，如果图层也具有非零的圆角半径，则不得将图层标记为不透明。

### 图层支持圆角半径

你可以通过为其添加圆角半径来为你的图层创建一个圆角矩形效果。 圆角半径是一种视觉装饰，它掩盖了图层边界矩形的一部分角落，以允许底层内容显示，如图 2-4 所示。 因为它涉及应用透明度蒙版，所以除非 masksToBounds 属性设置为YES，否则圆角半径不会影响图层的内容属性中的图像。 但是，圆角半径总是会影响图层背景颜色和边框的绘制方式。

图 2-4 图层上的圆角半径
![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/layer_corner_radius_2x.png)

要将圆角半径应用于图层，请指定图层的 cornerRadius 属性的值。 你指定的半径值以点为单位并在显示之前应用于图层的所有四个角。

### 图层支持内置阴影

CALayer 类包含几个用于配置阴影效果的属性。 阴影通过使其看起来好像在其底层内容上方浮动而增加了图层的深度。 这是另一种视觉装饰，你可能会发现它在特定情况下对你的应用有用。 通过图层，你可以控制阴影的颜色，相对于图层内容的位置，不透明度和形状。

默认情况下，图层阴影的不透明度值设置为0，这有效隐藏了阴影。 将不透明度更改为非零值会导致 Core Animation 绘制阴影。 由于默认情况下阴影直接位于图层下，因此你可能还需要更改阴影的偏移量，然后才能看到阴影。 不过，需要记住的是，你为阴影指定的偏移量是使用图层的本地坐标系来应用的，这在 iOS 和 OS X 上是不同的。图2-5显示了一个具有向下延伸的阴影的图层， 图层右侧。 在 iOS 中，这需要为 y 轴指定正值，但在 OS X 中，该值需要为负值。

图 2-5 对图层应用阴影

![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/layer_shadows_2x.png)

向图层添加阴影时，阴影是图层内容的一部分，但实际上是延伸到图层的边界矩形之外。 因此，如果为该图层启用了 masksToBounds 属性，则阴影效果会在边缘被剪切。 如果图层包含任何透明内容，则可能会导致一种奇怪的效果，即直接位于图层下方的阴影部分仍然可见，但延伸到图层之外的部分不会。 如果你想要一个阴影，但也想使用边界遮罩，你可以使用两个图层而不是一个图层。 将蒙版应用于包含内容的图层，然后将该图层嵌入到启用了阴影效果的大小完全相同的第二个图层中。

有关如何将阴影应用于图层的示例，请参阅阴影属性。

### 滤镜将视觉效果添加到 OS X 视图

在 OS X 应用程序中，可以将 Core Image 滤镜直接应用于图层的内容。 你可能会这样做，以模糊或锐化图层的内容，更改颜色，扭曲内容或执行许多其他类型的操作。 例如，图像处理程序可能会使用这些滤镜非破坏性地修改图像，而视频编辑程序可能会使用它们来实现不同类型的视频转换效果。 而且由于滤镜应用于硬件中的图层内容，因此渲染速度很快且平滑。

> 注意：你无法将滤镜添加到 iOS 中的图层对象。

对于给定的图层，可以将滤镜应用于图层的前景和背景内容。 前景内容由图层本身包含的所有内容组成，包括图像的 [contents](https://developer.apple.com/documentation/quartzcore/calayer/1410773-contents) 属性，背景颜色，边框和子图层的内容。 背景内容是直接在图层下方的内容，但实际上并不是图层本身的一部分。 大多数图层的背景内容是其直接超图层的内容，可能完全或部分被图层遮挡。 例如，当你希望用户专注于图层的前景内容时，你可以将模糊滤镜应用于背景内容。

你可以通过将 [CIFilter](https://developer.apple.com/documentation/coreimage/cifilter) 对象添加到图层的以下属性来指定滤镜：
- [filters](https://developer.apple.com/documentation/quartzcore/calayer/1410901-filters) 属性包含一个只影响图层前景内容的滤镜数组。
- [backgroundFilters](https://developer.apple.com/documentation/quartzcore/calayer/1410827-backgroundfilters) 属性包含仅影响图层背景内容的滤镜数组。
- [compositingFilter](https://developer.apple.com/documentation/quartzcore/calayer/1410748-compositingfilter) 属性定义图层的前景和背景内容如何合成在一起。

要为图层添加滤镜，你必须首先找到并创建 CIFilter 对象，然后在将其添加到图层之前对其进行配置。 CIFilter 类包含几个用于查找可用 Core Image 的类方法，例如 [filterWithName:](https://developer.apple.com/documentation/coreimage/cifilter/1438255-filterwithname) 方法。不过，创建滤镜只是第一步。许多滤镜具有定义滤镜如何修改图像的参数。例如，框模糊滤镜具有影响所应用的模糊量的输入半径参数。你应始终为这些参数提供值作为滤镜配置过程的一部分。但是，你不需要指定的一个常见参数是由图层本身提供的输入图像。

将滤镜添加到图层时，最好在将滤镜添加到图层之前配置滤镜参数。这样做的主要原因是，一旦添加到图层，你无法修改 CIFilter 对象本身。但是，你可以使用图层的 [setValue:forKeyPath:](https://developer.apple.com/documentation/objectivec/nsobject/1418139-setvalue) 方法在事实之后更改过滤器值。

清单2-6显示了如何创建并向图层对象应用捏合失真过滤器。此滤镜将内层的源像素向内捏住，使距离指定中心点最近的那些像素变形。请注意，在示例中，您不需要为过滤器指定输入图像，因为图层的图像是自动使用的。

清单 2-6 将滤镜应用于图层

```
CIFilter* aFilter = [CIFilter filterWithName:@"CIPinchDistortion"];
[aFilter setValue:[NSNumber numberWithFloat:500.0] forKey:@"inputRadius"];
[aFilter setValue:[NSNumber numberWithFloat:1.25] forKey:@"inputScale"];
[aFilter setValue:[CIVector vectorWithX:250.0 Y:150.0] forKey:@"inputCenter"];
 
myLayer.filters = [NSArray arrayWithObject:aFilter];
```

有关可用 Core Image 滤镜的信息，请参阅 [Core Image 参考](https://developer.apple.com/library/content/documentation/GraphicsImaging/Reference/CoreImageFilterReference/index.html#//apple_ref/doc/uid/TP40004346)。

## OS X 视图的图层重绘策略影响性能

在 OS X 中，支持图层的视图支持几种不同的策略，以确定何时更新底层的内容。 由于本机 AppKit 绘图模型与 Core Animation 引入的绘图模型之间存在差异，因此这些策略可以更轻松地将旧代码迁移到 Core Animation。 你可以在逐个视图的基础上配置这些策略，以确保每个视图的最佳性能。

每个视图定义一个 layerContentsRedrawPolicy 方法，该方法返回视图图层的重画策略。 你可以使用 setLayerContentsRedrawPolicy: 方法设置策略。 为了保持与传统绘图模型的兼容性，AppKit 默认将重绘策略设置为 NSViewLayerContentsRedrawDuringViewResize。 但是，你可以将策略更改为表2-2中的任何值。 请注意，建议的重绘策略不是默认策略。

表 2-2 OS X 视图的图层重绘策略

| Policy | Usage | 
|-|-|
| [NSViewLayerContentsRedrawOnSetNeedsDisplay](https://developer.apple.com/documentation/appkit/nsviewlayercontentsredrawpolicy/nsviewlayercontentsredrawonsetneedsdisplay) | 这是推荐的策略。 使用此策略，视图几何体更改不会自动使视图更新其图层的内容。 相反，该图层的现有内容被拉伸和操纵以促进几何变化。 要强制视图重绘自己并更新图层的内容，你必须显式调用视图的 [setNeedsDisplay:](https://developer.apple.com/documentation/appkit/nsview/1483360-needsdisplay) 方法。 此策略最接近地表示 Core Animation 图层的标准行为。 但是，它不是默认策略，必须明确设置。 |
| [NSViewLayerContentsRedrawDuringViewResize](https://developer.apple.com/documentation/appkit/nsviewlayercontentsredrawpolicy/nsviewlayercontentsredrawduringviewresize) | 这是默认的重绘策略。 此策略通过在视图的几何更改时回顾图层的内容来保持与传统 AppKit 绘图的最大兼容性。 这种行为导致视图的 drawRect: 方法在调整大小操作期间在应用程序主线程上多次调用。 |
| [NSViewLayerContentsRedrawBeforeViewResize](https://developer.apple.com/documentation/appkit/nsviewlayercontentsredrawpolicy/nsviewlayercontentsredrawbeforeviewresize) | 通过这个策略，AppKit 在任何调整大小操作并缓存该位图之前将其绘制为最终大小。 调整大小操作使用缓存的位图作为起始图像，将其缩放以适合旧的边界矩形。 然后将位图动画化为其最终大小。 这种行为可能导致视图的内容在动画开始时出现拉伸或扭曲，并且在初始外观不重要或不明显的情况下更好。 |
| [NSViewLayerContentsRedrawNever](https://developer.apple.com/documentation/appkit/nsview.layercontentsredrawpolicy/1483763-never) | 有了这个策略，即使调用 setNeedsDisplay: 方法，AppKit 也不会更新图层。 此策略对于内容不会更改以及视图大小偶尔更改的视图来说最适合。 例如，你可以将此用于显示固定大小内容或背景元素的视图。 |

查看重绘策略可以减少使用独立子图层来提高绘图性能的需求。 在引入视图重绘策略之前，有一些层次支持的视图比需要的更频繁地绘制，从而导致性能问题。 这些性能问题的解决方案是使用子图层来呈现视图内容中不需要定期重绘的部分。 通过在 OS X v10.6 中引入重绘策略，现在建议你将图层支持的视图的重绘策略设置为适当的值，而不是创建显式的子图层次结构。

## 将自定义属性添加到图层

[CAAnimation](https://developer.apple.com/documentation/quartzcore/caanimation) 和 [CALayer](https://developer.apple.com/documentation/quartzcore/calayer) 类扩展了键值编码惯例以支持自定义属性。 你可以使用此行为将数据添加到图层，并使用你定义的自定义键检索它。 你甚至可以将操作与你的自定义属性相关联，以便在更改属性时执行相应的动画。

有关如何设置和获取自定义属性的信息，请参阅[键值编码兼容容器类](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Key-ValueCodingExtensions/Key-ValueCodingExtensions.html#//apple_ref/doc/uid/TP40004514-CH12-SW3)。 有关将操作添加到图层对象的信息，请参阅[更改图层的默认行为](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/ReactingtoLayerChanges/ReactingtoLayerChanges.html#//apple_ref/doc/uid/TP40004514-CH7-SW1)。

## 打印层次支持视图的内容

在打印过程中，图层会根据需要重新绘制其内容以适应打印环境。 Core Image 在渲染到屏幕时通常依赖于缓存的位图，而在打印时会重新绘制该内容。 特别是，如果一个图层背景视图使用 drawRect: 方法提供图层内容，Core Animation 将在打印期间再次调用 drawRect：以生成打印图层内容。

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~