# 图层样式属性动画

在渲染过程中，Core Animation 获取图层的不同属性并以特定顺序渲染它们。 该顺序决定了图层的最终外观。 本章通过设置不同的图层样式属性来说明渲染结果。

> 注意：Mac OS X 和 iOS 上可用的图层样式属性不同，并在本章中注明。

## 几何属性

图层的几何属性指定它相对于其父图层的显示方式。 几何图形还指定用于圆化图层角的半径以及应用于图层及其子图层的变换。 图A-1显示了示例图层的边界矩形。

图 A-1 图层几何图形
![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/visual-geometry_2x.png)

以下 CALayer 属性指定图层的几何图形：

- [bounds](https://developer.apple.com/documentation/quartzcore/calayer/1410915-bounds)
- [position](https://developer.apple.com/documentation/quartzcore/calayer/1410791-position)
- [frame](https://developer.apple.com/documentation/quartzcore/calayer/1410779-frame) (从 bounds 和 position 计算出来，并且不是可动画的)
- [anchorPoint](https://developer.apple.com/documentation/quartzcore/calayer/1410817-anchorpoint)
- [cornerRadius](https://developer.apple.com/documentation/quartzcore/calayer/1410818-cornerradius)
- [transform](https://developer.apple.com/documentation/quartzcore/calayer/1410836-transform)
- [zPosition](https://developer.apple.com/documentation/quartzcore/calayer/1410884-zposition)

> iOS 注意：cornerRadius 属性仅在 iOS 3.0及更高版本中受支持。

## 背景属性

Core Animation 渲染的第一件事是图层的背景。 你可以为背景指定一种颜色。 在 OS X 中，你还可以指定要应用于背景内容的 Core Image 滤镜。 图A-2显示了样本图层的两个版本。 左侧的图层设置了 backgroundColor 属性，而右侧的图层没有背景颜色，但边框的某些内容和指定给其 backgroundFilters 属性的缩放失真滤镜都有。

图 A-2 带背景色的图层
![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/visual-background_2x.png)

后台过滤器应用于位于图层后面的内容，该图层主要由父图层的内容组成。 你可以使用背景滤镜来使前景图层内容脱颖而出; 例如，通过应用模糊滤镜。

以下 CALayer 属性影响图层背景的显示：

- [backgroundColor](https://developer.apple.com/documentation/quartzcore/calayer/1410966-backgroundcolor)
- [backgroundFilters](https://developer.apple.com/documentation/quartzcore/calayer/1410827-backgroundfilters) (在 iOS 中不受支持)

> 平台注意：在 iOS 中，backgroundFilters 属性在 CALayer 类中公开，但你分配给此属性的滤镜将被忽略。

## 图层内容

如果图层具有任何内容，则该内容将呈现在背景颜色的顶部。 你可以通过直接设置位图来提供图层内容，通过使用委托来指定内容，或者通过对图层进行子类化并直接绘制内容。 你可以使用许多不同的绘图技术（包括 Quartz，Metal，OpenGL 和 Quartz Composer）来提供该内容。 图A-3显示了一个样本图层，其内容是一个直接设置的位图。 位图内容由右下角带有 Automator 图标的大部分透明空间组成。

图 A-3 显示位图图像的图层

![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/visual-contents_2x.png)

具有圆角半径的图层不会自动剪裁其内容; 然而，将图层的 masksToBounds 属性设置为 YES 确实会导致图层剪切到其角半径。

以下 [CALayer](https://developer.apple.com/documentation/quartzcore/calayer) 属性影响图层内容的显示：

- [contents](https://developer.apple.com/documentation/quartzcore/calayer/1410773-contents)
- [contentsGravity](https://developer.apple.com/documentation/quartzcore/calayer/1410872-contentsgravity)
- [masksToBounds](https://developer.apple.com/documentation/quartzcore/calayer/1410896-maskstobounds)

## Sublayers内容

任何图层都可能包含一个或多个子图层，称为子图层。 子图层递归呈现，并相对于父图层的边界矩形进行定位。 另外，Core Animation 将父层的 sublayerTransform 应用于相对于父层的锚点的每个子层。 你可以使用子图层转换将透视图和其他效果均等地应用于所有图层。 图A-4显示了具有两个子层的样本层。 左侧的版本包含背景色，而右侧的版本没有。

图 A-4 显示子图层内容的图层
![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/visual-sublayers_2x.png)

将图层的 masksToBounds 属性设置为 YES 会导致将任何子图素裁剪到图层的边界。

以下 CALayer 属性影响图层子图层的显示：

- [sublayers](https://developer.apple.com/documentation/quartzcore/calayer/1410802-sublayers)
- [masksToBounds](https://developer.apple.com/documentation/quartzcore/calayer/1410896-maskstobounds)
- [sublayerTransform](https://developer.apple.com/documentation/quartzcore/calayer/1410888-sublayertransform)

## 边框属性

图层可以使用指定的颜色和宽度显示可选边框。 边界遵循图层的边界矩形，并考虑到任何角半径值。 图A-5显示了应用边框后的样本图层。 请注意，超出图层边界的内容和子图层将呈现在边框下方。

图 A-5 显示边框属性内容的图层

![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/visual-borderwidth_2x.png)

以下 CALayer 属性影响图层边框的显示：

- [borderColor](https://developer.apple.com/documentation/quartzcore/calayer/1410903-bordercolor)
- [borderWidth](https://developer.apple.com/documentation/quartzcore/calayer/1410917-borderwidth)

> 平台注意：borderColor 和 borderWidth 属性仅在 iOS 3.0及更高版本中受支持。

## 滤镜属性

在 OS X 中，你可以将一个或多个滤镜应用于图层的内容，并使用自定义合成滤镜来指定图层的内容如何与其基础图层的内容混合。 图A-6显示了一个应用了 Core Image posterize 过滤器的示例图层。

图 A-6 显示过滤器属性的图层
![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/visual-filters_2x.png)

以下 CALayer 属性指定了图层内容滤镜：

- [filters](https://developer.apple.com/documentation/quartzcore/calayer/1410901-filters)
- [compositingFilter](https://developer.apple.com/documentation/quartzcore/calayer/1410748-compositingfilter)

> 平台注意：在 iOS 中，图层忽略你分配给它们的任何过滤器。

## 阴影属性

图层可以显示阴影效果并配置其形状，不透明度，颜色，偏移量和模糊半径。 如果你未指定自定义阴影形状，则阴影基于图层中不完全透明的部分。 图A-7显示了应用了红色阴影的同一样本层的几个不同版本。 左侧和中间版本包含背景颜色，因此阴影仅出现在图层的边界附近。 但是，右侧的版本不包含背景颜色。 在这种情况下，阴影应用于图层的内容，边框和子图层。

图 A-7 显示阴影属性的图层
![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/visual-shadow_2x.png)

以下 CALayer 属性影响图层阴影的显示：

- [shadowColor](https://developer.apple.com/documentation/quartzcore/calayer/1410829-shadowcolor)
- [shadowOffset](https://developer.apple.com/documentation/quartzcore/calayer/1410970-shadowoffset)
- [shadowOpacity](https://developer.apple.com/documentation/quartzcore/calayer/1410751-shadowopacity)
- [shadowRadius](https://developer.apple.com/documentation/quartzcore/calayer/1410819-shadowradius)
- [shadowPath](https://developer.apple.com/documentation/quartzcore/calayer/1410771-shadowpath)

> 平台注意：iOS 3.2及更高版本支持 shadowColor，shadowOffset，shadowOpacity 和 shadowRadius 属性。 shadowPath 属性在iOS 3.2及更高版本以及OS X v10.7和更高版本中受支持。

## 不透明属性

图层的不透明属性决定了背景内容通过图层显示的数量。 图A-8显示了一个不透明度设置为0.5的样本图层。 这允许背景图像的部分显示。

图 A-8 包含不透明属性的图层

![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/visual-opacity_2x.png)

以下 CALayer 属性指定图层的不透明度：

- [opacity](https://developer.apple.com/documentation/quartzcore/calayer/1410933-opacity)

## 遮罩属性

你可以使用遮罩来遮盖图层内容的全部或部分。 蒙版本身就是一个图层对象，其 alpha 通道用于确定什么是阻塞的以及传输的是什么。 遮罩层内容的不透明部分允许底层内容显示，而透明部分部分或完全遮蔽底层内容。 图A-9显示了一个样本图层与一个遮罩层和两个不同的背景。 在左侧版本中，该图层的不透明度设置为1.0。 在正确的版本中，图层的不透明度设置为0.5，从而增加了通过图层的蒙版部分传输的背景内容的数量。

图 A-9 与 mask 属性合成的图层
![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/visual-mask_2x.png)

以下 CALayer 属性指定图层的遮罩：

- [mask](https://developer.apple.com/documentation/quartzcore/calayer/1410861-mask)

> 平台注意：iOS 3.0 及更高版本支持 mask 属性。

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~