# Core Animation 基础

Core Animation 为动画应用的视图和其他视觉元素提供了一个通用系统。 Core Animation 不能取代你的应用程序的视图。 相反，它是一种与视图集成的技术，可以为动画制作内容提供更好的性能和支持。 它通过将视图内容缓存到可由图形硬件直接操作的位图来实现此行为。 在某些情况下，这种缓存行为可能需要你重新思考如何呈现和管理你的应用内容，但大多数情况下，你使用 Core Animation 时并不知道它的存在。 除了缓存视图内容外，Core Animation 还定义了一种指定任意可视内容的方法，将该内容与视图集成，并与其他所有内容一起进行动画制作。

你可以使用 Core Animation 为你的应用的视图和可视对象设置动画效果。 大多数更改与修改视觉对象的属性有关。 例如，你可以使用 Core Animation 来为视图的位置，大小或不透明度设置动画效果。 当你进行这样的更改时，Core Animation 会在属性的当前值和你指定的新值之间进行动画处理。 你通常不会使用 Core Animation 以每秒60次的速度替换视图的内容，例如卡通内容。 相反，你使用 Core Animation 将视图的内容移动到屏幕上，淡入淡出内容，将任意图形转换应用于视图或更改视图的其他视觉属性。

## 图层为绘画和动画提供基础

图层对象是组织在 3D 空间中的 2D 平面，是 Core Animation 所做的一切的核心。 与视图一样，图层管理有关其曲面的几何图形，内容和视觉属性的信息。 与视图不同，图层不定义自己的外观。 一个层只管理位图周围的状态信息。 位图本身可以是视图绘制本身或你指定的固定图像的结果。 出于这个原因，你在应用程序中使用的主要图层被视为模型对象，因为它们主要管理数据。 这个概念很重要，因为它会影响动画的行为。

### 基于图层的绘图模型

大多数图层不会在你的应用中执行任何实际的绘图。 相反，图层会捕获应用程序提供的内容并将其缓存在位图中，该位图有时称为后备存储。 当你随后更改图层的属性时，你所做的只是更改与图层对象关联的状态信息。 当更改触发动画时，Core Animation 将图层的位图和状态信息传递给图形硬件，图形硬件完成使用新信息渲染位图的工作，如图1-1所示。 在硬件中操作位图会产生比在软件中更快的动画。

图 1-1 Core Animation 如何绘制内容
![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/basics_layer_rendering_2x.png)

由于它操纵静态位图，因此基于图层的绘图与传统的基于视图的绘图技术有很大不同。 使用基于视图的绘图时，对视图本身的更改通常会导致调用视图的drawRect: 方法以使用新参数重绘内容。 但以这种方式绘制是昂贵的，因为它使用主线程上的 CPU 来完成。 只要可能，通过在硬件中操作缓存的位图来实现相同或类似的效果，Core Animation 就可以避免这种开销。

尽管 Core Animation 尽可能使用缓存内容，但你的应用程序仍必须提供初始内容并不时进行更新。 有几种方法可以让你的应用程序提供包含内容的图层对象，详细信息请参阅[提供图层内容](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/SettingUpLayerObjects/SettingUpLayerObjects.html#//apple_ref/doc/uid/TP40004514-CH13-SW4)。

### 基于图层的动画

图层对象的数据和状态信息与屏幕上该图层内容的可视化表示分离。 这种解耦为 Core Animation 提供了一种插入自身的方法，并使从旧状态值到新状态值的变化动画化。 例如，更改图层的位置属性会使 Core Animation 将图层从其当前位置移动到新指定的位置。 对其他属性的类似更改会导致适当的动画。 图1-2说明了你可以在图层上执行的几种动画类型。 有关触发动画的图层属性的列表，请参阅[动画属性](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW1)。

图 1-2 你可以在图层上执行的动画示例
![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/basics_animation_types_2x.png)

在动画过程中，Core Animation 会以硬件完成所有逐帧绘制。 你只需指定动画的开始点和结束点，然后让 Core Animation 完成剩下的动作。 你还可以根据需要指定自定义时间信息和动画参数; 但是，如果你不这样做，Core Animation 会提供合适的默认值。

有关如何启动动画和配置动画参数的更多信息，请参阅[动画图层内容](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/CreatingBasicAnimations/CreatingBasicAnimations.html#//apple_ref/doc/uid/TP40004514-CH3-SW1)。

## 图层对象定义他们自己的几何图形

一个图层的任务之一是管理其内容的视觉几何图形。 视觉几何包含有关内容边界，屏幕位置以及图层是否以任何方式旋转，缩放或变形的信息。 与视图类似，图层具有可用于定位图层及其内容的 frame 和 bounds rectangles。 图层还具有视图不具有的其他属性，例如定义操作发生点的定位点。 你指定图层几何体的某些方面的方式也与你为视图指定该信息的方式不同。

## 图层使用两种类型的坐标系统

图层利用基于点的坐标系和单位坐标系来指定内容的位置。 使用哪个坐标系取决于传送的信息的类型。 指定直接映射到屏幕坐标的值时必须使用基于点的坐标，或者必须指定相对于其他图层的值（例如图层的位置属性）。 当值不应与屏幕坐标相关时使用单位坐标，因为它与其他值相关。 例如，图层的 [anchorPoint](https://developer.apple.com/documentation/quartzcore/calayer/1410817-anchorpoint) 属性指定了一个相对于图层本身边界的点，它可以改变。

基于点的坐标最常见的用途是指定图层的大小和位置，你可以使用图层的边界和位置属性进行确定。 bounds 定义了图层本身的坐标系，并包含了图层在屏幕上的大小。 position 属性定义了图层相对于其父坐标系的位置。 尽管图层具有 frame 属性，但该属性实际上是从边界和位置属性中的值导出的，并且使用频率较低。

图层 bounds 和 frame rectangles 的方向始终与底层平台的默认方向匹配。 图1-3显示了 iOS 和 OS X 上 bounds rectangle 的默认方向。在 iOS 中，默认情况下，bounds rectangle 的原点位于图层的左上角，而在 OS X 中，它位于底部左角。 如果你在 iOS 和 OS X 版本的应用程序之间共享 Core Animation 代码，则必须考虑这些差异。

图 1-3 iOS 和 OS X 的默认图层几何图形
![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/layer_coords_bounds_2x.png)

图 1-3 中要注意的一点是 position 属性位于图层的中间。 该属性是其中几个定义基于该图层的 [anchorPoint](https://developer.apple.com/documentation/quartzcore/calayer/1410817-anchorpoint) 属性中的值而更改的属性之一。 定位点表示某些坐标来自的点，并在[定位点影响几何操作](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/CoreAnimationBasics/CoreAnimationBasics.html#//apple_ref/doc/uid/TP40004514-CH2-SW17)中有更详细的描述。

定位点是你使用单位坐标系指定的几个属性之一。 Core Animation 使用单位坐标来表示当图层大小改变时其值可能改变的属性。 你可以将单位坐标视为指定总可能值的百分比。 单位坐标空间中的每个坐标的范围为0.0到1.0。 例如，沿着 x 轴，左边缘位于坐标0.0处，右边缘位于坐标1.0处。 沿着 y 轴，单位坐标值的方向根据平台而变化，如图1-4所示。

图 1-4 iOS 和 OS X 的默认单位坐标系
![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/layer_coords_unit_2x.png)

> 注意：在 OS X 10.8 之前，geometryFlipped 属性是一种在需要时更改图层 y 轴的默认方向的方法。 涉及翻转变换时，有时需要使用此属性来纠正图层的方向。 例如，如果父视图使用翻转变换，则其子视图（及其相应图层）的内容通常会被颠倒。 在这种情况下，将子图层的 geometryFlipped 属性设置为 YES 是纠正问题的简单方法。 在 OS X 10.8 及更高版本中，AppKit 为你管理这个属性，你不应该修改它。 对于 iOS 应用程序，建议你根本不要使用 geometryFlipped 属性。

所有坐标值，无论是点还是单位坐标都被指定为浮点数。 浮点数的使用允许你指定可能落在正常坐标值之间的精确位置。 使用浮点值很方便，特别是在打印过程中或在绘制视网膜显示时，其中一个点可能由多个像素表示。 浮点值允许你忽略基础设备分辨率，并以你需要的精度指定值。

### 锚点影响几何操作

与图层的几何相关的操作相对于该图层的锚点发生，你可以使用图层的 [anchorPoint](https://developer.apple.com/documentation/quartzcore/calayer/1410817-anchorpoint) 属性访问该锚点。 处理图层的 position 或[transform](https://developer.apple.com/documentation/quartzcore/calayer/1410836-transform)属性时，锚点的影响最为明显。 位置属性始终是相对于图层的锚点指定的，并且你应用于该图层的任何转换也会相对于锚点发生。

图1-5演示了如何将锚点从其默认值更改为不同的值影响图层的位置属性。 即使图层没有在其父母的范围内移动，将定位点从图层中心移动到图层的边界原点也会更改 position 属性中的值。

图 1-5 锚点如何影响图层的位置属性
![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/layer_coords_anchorpoint_position_2x.png)

图1-6显示了如何改变定位点影响应用于图层的转换。 当你对图层应用旋转变换时，旋转发生在锚点周围。 由于默认情况下将锚点设置为图层的中间位置，因此通常会创建你所期望的那种旋转行为。 但是，如果更改定位点，则旋转的结果会有所不同。

图 1-6 锚点如何影响图层转换
![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/layer_coords_anchorpoint_transform_2x.png)

### 图层可以在三个维度操纵

每个图层都有两个转换矩阵，你可以使用它们来操纵图层及其内容。 CALayer 的 [transform](https://developer.apple.com/documentation/quartzcore/calayer/1410836-transform) 属性指定要应用于图层及其嵌入子图层的变换。 通常当你想修改图层本身时使用这个属性。 例如，你可以使用该属性缩放或旋转图层或临时更改其位置。 [sublayerTransform](https://developer.apple.com/documentation/quartzcore/calayer/1410888-sublayertransform) 属性定义了仅应用于子图层的其他转换，并且最常用于将透视视觉效果添加到场景内容。

Transforms 的工作方式是将坐标值乘以数字矩阵以获得代表原始点变换版本的新坐标。 由于 Core Animation 值可以在三维中指定，每个坐标点有四个值，必须通过四乘四矩阵相乘，如图1-7所示。 在 Core Animation 中，图中的变换由 CATransform3D 类型表示。 幸运的是，你不必直接修改此结构的字段以执行标准转换。 Core Animation 提供了一套全面的功能，用于创建比例，平移和旋转矩阵以及进行矩阵比较。 除了使用函数来操作变换外，Core Animation 还扩展了键值编码支持，允许您使用 keyPath 修改变换。 有关可以修改的关键路径列表，请参阅 [CATransform3D Key Paths](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Key-ValueCodingExtensions/Key-ValueCodingExtensions.html#//apple_ref/doc/uid/TP40004514-CH12-SW1)。

图 1-7 使用矩阵运算来转换坐标

![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/transform_basic_math_2x.png)

图1-8显示了可以进行的一些更常见转换的矩阵配置。 通过标识变换乘以任何坐标返回完全相同的坐标。 对于其他转换，坐标如何修改完全取决于你更改哪个矩阵组件。 例如，要仅沿着 x 轴进行平移，你将为平移矩阵的 tx 分量提供非零值，并将 ty 和 tz 值保留为0。对于旋转，你将提供适当的正弦和余弦值目标旋转角度。

图 1-8 常见转换的矩阵配置

![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/transform_manipulations_2x.png)

有关用于创建和操作变换的函数的信息，请参阅核心动画函数参考。

## 图层树反映了动画状态的不同方面

使用 Core Animation 的应用程序有三组图层对象。 每一组图层对象在使你的应用程序的内容出现在屏幕上时具有不同的作用：

- 模型图层树中的对象（或简称为“图层树”）是你的应用与最多的对象交互的对象。 此树中的对象是存储任何动画的目标值的模型对象。 无论何时更改图层的属性，都可以使用这些对象之一。
- 表示树中的对象包含任何正在运行的动画的空中值。 尽管图层树对象包含动画的目标值，但表示树中的对象反映了屏幕上显示的当前值。 你绝不应该修改此树中的对象。 相反，你可以使用这些对象来读取当前的动画值，也许可以从这些值开始创建一个新的动画。
- 渲染树中的对象执行实际的动画，并且对于 Core Animation 是私有的。

每组图层对象都组织成一个层次结构，就像你的应用中的视图一样。 事实上，对于为所有视图启用图层的应用程序，每个树的初始结构都与视图层次结构完全匹配。 但是，应用程序可以根据需要将其他图层对象（即未与视图关联的图层）添加到图层分层结构中。 你可以在某些情况下执行此操作，以优化应用程序对不需要视图所有开销的内容的性能。 图1-9显示了在简单的 iOS 应用程序中找到的图层细分。 示例中的窗口包含一个内容视图，该视图本身包含一个按钮视图和两个独立的图层对象。 每个视图都有一个相应的图层对象，构成图层层次结构的一部分。

图 1-9 与窗口关联的图层
![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/sublayer_hierarchy_2x.png)

对于图层树中的每个对象，在演示文稿和渲染树中都有一个匹配的对象，如图1-10所示。 如前所述，应用程序主要处理图层树中的对象，但有时可能访问表示树中的对象。 具体而言，访问图层树中对象的 presentationLayer 属性将返回表示树中的相应对象。 你可能想要访问该对象以读取动画中间属性的当前值。

图 1-10 窗口的图层树
![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/sublayer_hierarchies_2x.png)

> 重要说明：只有在动画正在飞行时，才应访问演示树中的对象。 在动画进行过程中，表示树包含当前显示在屏幕上的图层值。 这种行为不同于图层树，它总是反映你的代码设置的最后一个值，它等同于动画的最终状态。

## 图层与视图的关系

图层不能代替你的应用视图 - 也就是说，你无法仅基于图层对象创建可视界面。图层为你的视图提供基础设施。具体而言，图层可以更轻松，更高效地绘制视图内容并进行动画处理，并在此过程中保持较高的帧速率。但是，有很多层没有做的事情。图层不处理事件，绘制内容，参与响应者链或做许多其他事情。因此，每个应用程序都必须有一个或多个视图来处理这些交互。

在 iOS 中，每个视图都由相应的图层对象支持，但在 OS X 中，你必须确定哪些视图应该具有图层。在 OS X v10.8 及更高版本中，向所有视图添加图层可能很有意义。但是，你不需要这样做，并且在开销不必要和不需要的情况下仍可以禁用图层。图层确实会增加应用程序的内存开销，但它们的好处往往超过了缺点，所以最好在禁用图层支持之前测试应用程序的性能。

当为视图启用图层支持时，你将创建被称为图层支持的视图。在分层视图中，系统负责创建底层图层对象并使该图层与视图保持同步。所有 iOS 视图都是分层支持的，OS X 中的大多数视图也是如此。但是，在 OS X 中，你还可以创建图层托管视图，该视图是你自己提供图层对象的视图。对于图层托管视图，AppKit 采取了一种管理图层的方法，并且不会响应视图更改而对其进行修改。

> 注意：对于图层支持的视图，建议尽可能操作视图而不是图层。 在 iOS 中，视图只是图层对象的薄包装，因此对图层进行的任何操作通常都可以正常工作。 但是在 iOS 和 OS X 中，操纵图层而不是视图可能不会产生所需的结果。 本文尽可能指出这些缺陷，并试图提供帮助您解决问题的方法。

除了与视图关联的图层外，还可以创建没有相应视图的图层对象。 你可以将这些独立图层对象嵌入到应用中的任何其他图层对象中，包括与视图关联的图层对象。 你通常使用独立的图层对象作为特定优化路径的一部分。 例如，如果你想在多个位置使用相同的图像，则可以加载图像一次，并将其与多个独立图层对象相关联，然后将这些对象添加到图层树中。 然后，每个图层都会引用源图像，而不是尝试在内存中创建该图像的自己的副本。

有关如何为应用视图启用图层支持的信息，请参阅[在应用中启用 Core Animation 支持](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/SettingUpLayerObjects/SettingUpLayerObjects.html#//apple_ref/doc/uid/TP40004514-CH13-SW5)。 有关如何创建图层对象层次结构的信息，以及关于何时可以这样做的提示，请参阅[构建图层层次结构](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/BuildingaLayerHierarchy/BuildingaLayerHierarchy.html#//apple_ref/doc/uid/TP40004514-CH6-SW2)。

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~