# 使用流布局

你可以使用具体布局对象 UICollectionViewFlowLayout 类来排列集合视图中的 items。 流布局实现了基于行的中断布局，这意味着布局对象将 cells 放置在线性路径上，并尽可能多地沿着该线安装 cells。 当布局对象在当前行的空间不足时，会创建一个新行并继续布局过程。 图3-1显示了垂直滚动的流布局的外观。 在这种情况下，水平放置线条，每条新线条位于上一行的下方。 单个 section 中的 cell 可以选择性地包围部分标题和部分页脚视图。

图 3-1 使用流布局布置 sections 和 cells
![](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/Art/flow_horiz_headers_2x.png)

你可以使用流布局来实现网格，但你也可以更多地使用它。 线性布局的想法可以应用于许多不同的设计。 例如，你可以调整间距以沿着滚动维度创建一行 items，而不是具有网格 items。 items 也可以是不同的大小，这产生了比传统网格更不对称的东西，但仍然具有线性流动。 有很多可能性。

你可以通过编程或在 Xcode 中使用 Interface Builder 来配置流布局。 配置流布局的步骤如下所示：

1、创建一个流布局对象并将其分配给你的集合视图。
2、配置 cells 的宽度和高度。
3、为 lines 和 items 设置间距选项（根据需要）。
4、如果你想要 section headers 或 section footers，请指定它们的大小。
5、设置布局的滚动方向。

> 重要说明：至少必须指定 cells 的宽度和高度。 如果你不这样做，你的 items 被分配一个宽度和高度为0，并永远不可见。

## 自定义流布局属性

流布局对象公开了几个用于配置内容外观的属性。 设置后，这些属性将同样适用于布局中的所有 items。 例如，使用流布局对象的 itemSize 属性设置 cell 大小会导致所有 cells 具有相同的大小。

如果要动态改变 items 的间距或大小，可以使用 UICollectionViewDelegateFlowLayout 协议的方法来完成。 你可以在分配给集合视图本身的同一个委托对象上实现这些方法。 如果存在给定方法，则流布局对象将调用该方法，而不是使用它的固定值。 然后，你的实现必须为集合视图中的所有 items 返回适当的值。

### 指定流布局中 Items 的大小

如果集合视图中的所有 items 都具有相同的大小，则将适当的宽度和高度值分配给流布局对象的 itemSize 属性。 （始终以点为单位指定 items 的大小。）这是为大小不变的内容配置布局对象的最快方法。

如果要为 cells 指定不同的大小，则必须在集合视图委托中实现 collectionView:layout:sizeForItemAtIndexPath: 方法。 你可以使用提供的 index path 信息返回相应 items 的大小。 在布局过程中，流布局对象垂直放置在同一条线上，如图3-2所示。 然后该线的总高度或宽度由该维度中的最大项确定。

图 3-2 流布局中不同尺寸的项目
![](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/Art/flow_horiz_layout_uneven_2x.png)

> 注意：为 cells 指定不同的大小时，单行上的 items 数可能因行而异。

### 指定 Items 和 Lines 之间的空间

使用流布局，你可以指定同一行上的 items 之间的最小间距和连续行之间的最小间距。请记住，你提供的间距仅为最小间距。由于它布置内容的方式，流布局对象可能会将项目之间的间距增加到大于指定值的值。布局对象可能会类似地增加布局的项目大小不同时的实际行间距。

在布局过程中，流布局对象将 items 添加到当前行，直到没有足够的空间来放置整个项目。如果线条的大小足以适合整数个物品而没有额外的空间，则 items 之间的间距将等于最小间距。如果线条末端有额外的空间，则布局对象将增加 items 间距，直到项目均匀地适合线条边界，如图3-3所示。增加间距可以改善物品的整体外观，并防止每条线末尾出现大的间隙。

图 3-3 项目之间的实际间距可能会大于最小值
![](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/Art/flow_item_spacing_2x.png)

对于行间距，流布局对象使用与 items 间距相同的技术。如果所有 items 的大小相同，则流布局能够绝对地遵守最小行间距值，并且一行中的所有项目似乎与下一行中的项目均匀隔开。如果 items 的尺寸不同，单个 items 之间的实际间距可能会有所不同。

图3-4演示了当 items 大小不同时，最小行间距会发生什么。使用不同大小的 items 时，流布局对象会从滚动方向上尺寸最大的每一行中选取项目。例如，在垂直滚动布局中，它会查找每个高度最大的行中的 items。然后将这些 items 之间的间距设置为最小值。如果物品位于线条的不同部分，如图所示，实际的线条间距似乎大于最小值。

图 3-4 如果项目大小不同，行间距会有所不同
![](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/Art/flow_line_spacing_2x.png)

与其他流布局属性一样，你可以使用固定间距值或动态更改值。 Lines 和 items 间距是在逐个部分的基础上处理的。 因此，给定部分中的所有 items 的 lines 和 items 间距均相同，但各部分之间可能会有所不同。 你可以使用流布局对象的 [minimumLineSpacing](https://developer.apple.com/documentation/uikit/uicollectionviewflowlayout/1617717-minimumlinespacing) 和 [minimumInteritemSpacing](https://developer.apple.com/documentation/uikit/uicollectionviewflowlayout/1617706-minimuminteritemspacing) 属性或使用集合视图委托的 [collectionView:layout:minimumLineSpacingForSectionAtIndex:](https://developer.apple.com/documentation/uikit/uicollectionviewdelegateflowlayout/1617705-collectionview) 和 [collectionView:layout:minimumInteritemSpacingForSectionAtIndex:](https://developer.apple.com/documentation/uikit/uicollectionviewdelegateflowlayout/1617696-collectionview) 方法静态设置间距。

### 使用 Section Insets 调整内容的边距

Section insets 是调整可用于布置 cells 的空间的一种方法。 你可以使用 insets 在 section 的标题视图之后和其页脚视图之前插入空格。 你还可以使用 insets 在内容的边上插入空格。 图3-5演示了 insets 如何影响垂直滚动流布局中的某些内容。

图 3-5 Section insets 更改布局 cells 的可用空间
![](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/Art/flow_section_insets_2x.png)

由于 insets 减少了布局单元格的可用空间量，因此可以使用它们来限制给定行中 cells 的数量。 指定非滚动方向的插入是缩小每行的空间的一种方法。 如果将这些信息与适当的 cells 大小相结合，则可以控制每行上的 cells 数量。

### 知道何时对流布局进行子类化

虽然你可以非常有效地使用流布局而无需进行子类化，但仍然有时你可能需要继承以获取所需的行为。 表3-1列出了为实现所需效果而需要继承 [UICollectionViewFlowLayout](https://developer.apple.com/documentation/uikit/uicollectionviewflowlayout) 的一些场景。

表 3-1 UICollectionViewFlowLayout 的子类化方案

| Scenario | Subclassing tips |
| - | - |
| 你想添加新的补充或装饰视图到你的布局 | 标准流布局类仅支持 section 标题和 section 页脚视图，并且不支持装饰视图。 为了支持附加的补充和装饰视图，你至少需要覆盖以下方法： [layoutAttributesForElementsInRect:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617769-layoutattributesforelementsinrec) (required)、[layoutAttributesForItemAtIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617797-layoutattributesforitematindexpa) (required)、[layoutAttributesForSupplementaryViewOfKind:atIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617792-layoutattributesforsupplementary) (to support new supplementary views)、 [layoutAttributesForDecorationViewOfKind:atIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617809-layoutattributesfordecorationvie) (to support new decoration views) 在layoutAttributesForElementsInRect：方法中，可以调用super来获取单元格的布局属性，然后为指定矩形中的任何新补充或装饰视图添加属性。 使用其他方法按需提供属性。有关在布局过程中为视图提供属性的信息，请参阅为给定矩形中的项目[创建布局属性](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/CreatingCustomLayouts/CreatingCustomLayouts.html#//apple_ref/doc/uid/TP40012334-CH5-SW21)并[提供布局属性](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/CreatingCustomLayouts/CreatingCustomLayouts.html#//apple_ref/doc/uid/TP40012334-CH5-SW6)。|
| 你想调整流布局返回的布局属性 | 覆盖 [layoutAttributesForElementsInRect:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617769-layoutattributesforelementsinrec) 方法和任何返回布局属性的方法。 你的方法的实现应该调用 super，修改父类提供的属性，然后返回它们。有关这些方法所需内容的深入讨论，请参阅为给定矩形中的项目[创建布局属性](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/CreatingCustomLayouts/CreatingCustomLayouts.html#//apple_ref/doc/uid/TP40012334-CH5-SW21)并[提供布局属性](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/CreatingCustomLayouts/CreatingCustomLayouts.html#//apple_ref/doc/uid/TP40012334-CH5-SW6)。 |
| 你想为你的单元格和视图添加新的布局属性 | 创建一个 [UICollectionViewLayoutAttributes](https://developer.apple.com/documentation/uikit/uicollectionviewlayoutattributes) 的自定义子类并添加表示你的自定义布局信息所需的任何属性。 子类 [UICollectionViewFlowLayout](https://developer.apple.com/documentation/uikit/uicollectionviewflowlayout) 并重写 [layoutAttributesClass](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617774-layoutattributesclass) 方法。 在你实现该方法时，返回你的自定义子类。 你还应该覆盖 [layoutAttributesForElementsInRect:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617769-layoutattributesforelementsinrec) 方法，[layoutAttributesForItemAtIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617797-layoutattributesforitematindexpa) 方法以及任何其他返回布局属性的方法。 在您的自定义实现中，你应该为你定义的任何自定义属性设置值。 |
| 你想为正在插入或删除的项目指定初始或最终位置 | 默认情况下，为插入或删除的项目创建一个简单的淡入淡出动画。要创建自定义动画，你必须重写部分或全部以下方法：[initialLayoutAttributesForAppearingItemAtIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617789-initiallayoutattributesforappear)、[initialLayoutAttributesForAppearingSupplementaryElementOfKind:atIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617737-initiallayoutattributesforappear)、[initialLayoutAttributesForAppearingDecorationElementOfKind:atIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617726-initiallayoutattributesforappear)、[finalLayoutAttributesForDisappearingItemAtIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617740-finallayoutattributesfordisappea)、[finalLayoutAttributesForDisappearingSupplementaryElementOfKind:atIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617742-finallayoutattributesfordisappea)、[finalLayoutAttributesForDisappearingDecorationElementOfKind:atIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617762-finallayoutattributesfordisappea)在这些方法的实现中，指定每个视图在插入之前或在删除之后所需的属性。流布局对象使用你提供的属性来动画插入和删除。如果你重写这些方法，则还建议你重写[prepareForCollectionViewUpdates:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617784-prepareforcollectionviewupdates) 和 [finalizeCollectionViewUpdates](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617787-finalizecollectionviewupdates) 方法。你可以使用这些方法来跟踪当前周期中正在插入或删除的项目。有关插入和删除操作的更多信息，请参阅[使插入和删除动画更有趣](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/CreatingCustomLayouts/CreatingCustomLayouts.html#//apple_ref/doc/uid/TP40012334-CH5-SW13)。 |

还有一些情况下，正确的做法是从头开始创建自定义布局。在决定这样做之前，请花时间考虑是否真的有必要。流布局提供了许多适用于许多不同类型布局的可定制行为，并且由于它提供给你，它易于使用并包含大量优化以使其更高效。然而，所有这些并不是说你永远不应该创建一个自定义布局，因为在某些情况下这样做是绝对有意义的。流布局将滚动方向限制为一个方向，因此如果你的布局包含的内容在两个方向上都比屏幕边界延伸得更远，则自定义布局更有意义。如果你的布局不是网格或基于行的中断布局（如上所述），或者布局中的项目移动频繁以至于对流布局的子类化比创建自己的布局更为精确，那么创建自定义布局是正确的决策。

有关创建自定义布局的更多信息，请参阅[创建自定义布局](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/CreatingCustomLayouts/CreatingCustomLayouts.html#//apple_ref/doc/uid/TP40012334-CH5-SW1)。

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~