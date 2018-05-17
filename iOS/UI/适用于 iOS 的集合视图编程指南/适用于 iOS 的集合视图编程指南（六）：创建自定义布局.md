# 创建自定义布局

在开始构建自定义布局之前，请考虑是否真的需要这样做。 UICollectionViewFlowLayout 类提供了大量已经针对效率进行了优化的行为，并且可以通过多种方式进行调整，以实现许多不同类型的标准布局。 考虑实施自定义布局的唯一时间是在以下情况：

- 你想要的布局看起来不像一个网格或基于行的中断布局（布局中的项目被放置到一行中直到它已满，然后继续到下一行，直到放置所有项目）或者需要滚动多于 一个方向。
- 你想要经常更改所有单元格位置，以便修改现有流布局比创建自定义布局更有帮助。

好消息是，从 API 的角度来看，实现自定义布局并不困难。 最难的部分是执行确定布局中 items 位置所需的计算。 当你知道这些项目的位置时，将该信息提供给集合视图非常简单。

## 子类化 UICollectionViewLayout

对于自定义布局，你希望子类 UICollectionViewLayout，它为你的设计提供了一个全新的起点。只有少数方法为你的布局对象提供了核心行为，并且在你的实现中是必需的。剩下的方法可以让你根据需要重写来调整布局行为。核心方法处理以下关键任务：

- 指定可滚动内容区域的大小。
- 为组成布局的 cells 和视图提供属性对象，以便集合视图可以定位每个 cell 和视图。

虽然你可以创建一个实现了核心方法的函数布局对象，但是如果你实现了几个可选的方法，你的布局可能会更有吸引力。

布局对象使用其数据源提供的信息来创建集合视图的布局。你的布局通过调用 collectionView 属性的方法与数据源进行通信，这可以在所有布局的方法中访问。请记住你的集合视图在布局过程中知道和不知道的内容。由于布局过程正在进行，集合视图无法跟踪视图的布局或定位。因此，即使布局对象不会限制你调用任何集合视图的方法，也不要依赖集合视图来获取计算布局所需的数据以外的任何内容。

### 了解核心布局过程

集合视图直接与你的自定义布局对象一起工作来管理整个布局过程。 当集合视图确定它需要布局信息时，它会要求你的布局对象提供它。 例如，集合视图首次显示或调整大小时会请求布局信息。 你还可以通过调用布局对象的 [invalidateLayout](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617728-invalidatelayout) 方法来指示集合视图来显式更新其布局。 该方法抛弃现有布局信息并强制布局对象生成新的布局信息。

> 注意：注意不要将布局对象的 [invalidateLayout](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617728-invalidatelayout) 方法与集合视图的 [reloadData](https://developer.apple.com/documentation/uikit/uicollectionview/1618078-reloaddata) 方法混淆。 调用 [invalidateLayout](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617728-invalidatelayout) 方法不一定会导致集合视图抛出现有的单元格和子视图。 而是强制布局对象在移动和添加或删除项目时重新计算其所有布局属性。 如果数据源中的数据已更改，则 reloadData 方法是适当的。 无论你如何启动布局更新，实际的布局过程都是相同的。

在布局过程中，集合视图会调用布局对象的特定方法。 这些方法是你计算 item 位置并为集合视图提供所需主要信息的机会。 也可以调用其他方法，但这些方法总是在布局过程中按以下顺序调用：

1、使用 [prepareLayout](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617752-preparelayout) 方法执行提供布局信息所需的前期计算。
2、根据你的初始计算，使用 [collectionViewContentSize](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617796-collectionviewcontentsize) 方法返回整个内容区域的整体大小。
3、使用 [layoutAttributesForElementsInRect:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617769-layoutattributesforelementsinrec) 方法返回指定矩形中的 cells 和视图的属性。

图 5-1 说明了如何使用上述方法来生成布局信息。

图 5-1 布置你的自定义内容
![](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/Art/cv_layout_process_2x.png)

prepareLayout 方法是你执行任何计算以确定布局中 cells 和视图位置的机会。至少，你应该在此方法中计算足够的信息，以便能够返回内容区域的总体大小，并返回到步骤2中的集合视图。

集合视图使用内容大小来适当地配置其滚动视图。例如，如果你计算的内容大小在当前设备屏幕的边界上纵向和横向扩展，则滚动视图将进行调整以允许同时在两个方向上滚动。与 UICollectionViewFlowLayout 不同，默认情况下，它不会调整内容的布局以仅向一个方向滚动。

根据当前的滚动位置，集合视图然后调用 layoutAttributesForElementsInRect: 方法来请求特定矩形中的 cells 和视图的属性，该矩形可能与可见矩形相同也可能不相同。在返回这些信息后，核心布局过程就会有效完成。

布局完成后，cells 和视图的属性保持不变，直到你或集合视图使布局无效。调用布局对象的 invalidateLayout 方法会导致布局过程再次开始，从对 prepareLayout 方法的新调用开始。集合视图也可以在滚动期间自动使布局无效。如果用户滚动其内容，则集合视图将调用布局对象的 [shouldInvalidateLayoutForBoundsChange:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617781-shouldinvalidatelayoutforboundsc) 方法，并在该方法返回 YES 时使布局无效。

> 注意：记住调用 invalidateLayout 方法不会立即开始布局更新过程很有用。 该方法仅将布局标记为与数据不一致并且需要更新。 在下一个视图更新周期中，集合视图检查其布局是否脏，如果是，则更新它。 实际上，你可以快速连续多次调用 invalidateLayout 方法，而不必每次都触发立即布局更新。

### 创建布局属性

你的布局负责的属性对象是 [UICollectionViewLayoutAttributes](https://developer.apple.com/documentation/uikit/uicollectionviewlayoutattributes) 类的实例。 这些实例可以在你的应用中以各种不同的方法创建。 当你的应用程序不处理数千个项目时，在准备布局时创建这些实例是有意义的，因为布局信息可以被缓存和引用，而不是实时计算。 如果先前计算所有属性的成本超过了应用中缓存的好处，那么在请求时就可以很容易地创建属性。

无论如何，当创建 UICollectionViewLayoutAttributes 类的新实例时，请使用以下类方法之一：

- [layoutAttributesForCellWithIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewlayoutattributes/1617759-layoutattributesforcellwithindex)
- [layoutAttributesForSupplementaryViewOfKind:withIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewlayoutattributes/1617801-layoutattributesforsupplementary)
- [layoutAttributesForDecorationViewOfKind:withIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewlayoutattributes/1617786-layoutattributesfordecorationvie)

你必须根据所显示视图的类型使用正确的类方法，因为集合视图使用该信息从数据源对象请求适当类型的视图。使用不正确的方法会导致集合视图在错误的位置创建错误的视图，并且你的布局未按预期显示。

创建每个属性对象后，为相应视图设置相关属性。至少，在布局中设置视图的大小和位置。在布局视图重叠的情况下，将值分配给 [zIndex](https://developer.apple.com/documentation/uikit/uicollectionviewlayoutattributes/1617768-zindex) 属性以确保重叠视图的一致排序。其他属性可让你控制 cells 或视图的可见性或外观，并可根据需要进行更改。如果标准属性类不适合你的应用程序的需求，你可以继承并扩展它以存储有关每个视图的其他信息。在对布局属性进行子类化时，需要实现用于比较自定义属性的 isEqual: 方法，因为集合视图对其某些操作使用此方法。

有关布局属性的更多信息，请参阅 [UICollectionViewLayoutAttributes](https://developer.apple.com/documentation/uikit/uicollectionviewlayoutattributes) 类参考。

### 准备布局

在布局周期开始时，布局对象在开始布局过程之前调用 prepareLayout。 此方法是你计算稍后通知布局的信息的机会。 prepareLayout 方法不需要实现自定义布局，而是作为提供初始计算的机会（如有必要）。 调用此方法后，你的布局必须具有足够的信息来计算集合视图的内容大小，这是布局过程中的下一步。 然而，这些信息的范围可以从这个最低要求到创建和存储布局将使用的所有布局属性对象。 prepareLayout 方法的使用取决于你的应用程序的基础结构，以及根据请求计算事先计算的内容与计算的内容。 有关 prepareLayout 方法的示例，请参阅[准备布局](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/AWorkedExample/AWorkedExample.html#//apple_ref/doc/uid/TP40012334-CH8-SW1)。

### 为给定矩形中的 Items 提供布局属性

在布局过程的最后一步中，集合视图会调用布局对象的 layoutAttributesForElementsInRect: 方法。 此方法的目的是为每个 cell 以及与指定矩形相交的每个补充或装饰视图提供布局属性。 对于大型可滚动内容区域，集合视图可能只是要求该内容区域当前可见部分中的项目属性。 在图5-2中，布局对象需要创建属性对象的当前可见内容是单元格6至20以及第二个页眉视图。 你必须准备好为集合视图内容区域的任何部分提供布局属性。 这些属性可能用于促进插入或删除项目的动画。

图 5-2 仅布置可见视图

![](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/Art/cv_visible_elements_2x.png)

因为 layoutAttributesForElementsInRect: 方法是在布局对象的 prepareLayout 方法之后调用的，所以你应该已经拥有了大部分需要返回或创建所需属性的信息。 layoutAttributesForElementsInRect: 方法的实现遵循以下步骤：

1、迭代 prepareLayout 方法生成的数据，以访问缓存的属性或创建新的属性。

2、检查每个 item 的 frame 以查看它是否与传递给 layoutAttributesForElementsInRect: 方法的矩形相交。

3、对于每个相交 item，将对应的 UICollectionViewLayoutAttributes 对象添加到数组中。

4、将布局属性数组返回到集合视图。

根据管理布局信息的方式，你可以在你的 prepareLayout 方法中创建 UICollectionViewLayoutAttributes 对象，或者等待并在 layoutAttributesForElementsInRect: 方法中执行。在形成符合应用程序需求的实现时，请记住缓存布局信息的好处。为单元重复计算新的布局属性是一项昂贵的操作，可能会对应用程序的性能产生显着的不利影响。也就是说，当你的集合视图管理的 items 数量很大时，在请求时创建布局属性可能更有意义（用于性能）。这只是确定哪种策略对你的应用最有意义。

> 注意：布局对象还需要能够根据需要为单个 item 提供布局属性。 由于多种原因，集合视图可能会在正常布局过程之外请求该信息，包括创建适当的动画。 有关按需提供布局属性的更多信息，请参阅[按需提供布局属性](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/CreatingCustomLayouts/CreatingCustomLayouts.html#//apple_ref/doc/uid/TP40012334-CH5-SW2)。

有关如何实现 layoutAttributesForElementsInRect 的具体示例，请参阅[提供布局属性](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/AWorkedExample/AWorkedExample.html#//apple_ref/doc/uid/TP40012334-CH8-SW9)。

### 按需提供布局属性

集合视图会定期要求你的布局对象为正式布局过程之外的各个 items 提供属性。 例如，配置视图在配置 items 的插入和删除动画时要求提供此信息。 你的布局对象必须准备好为其支持的每个 cell，补充视图和装饰视图提供布局属性。 你可以通过覆盖以下方法来执行此操作：

- [layoutAttributesForItemAtIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617797-layoutattributesforitematindexpa)
- [layoutAttributesForSupplementaryViewOfKind:atIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617792-layoutattributesforsupplementary)
- [layoutAttributesForDecorationViewOfKind:atIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617809-layoutattributesfordecorationvie)

你的这些方法的实现应该检索给定 cell 或视图的当前布局属性。 每个自定义布局对象都需要实现 layoutAttributesForItemAtIndexPath: 方法。 如果你的布局不包含任何补充视图，则无需重写 layoutAttributesForSupplementaryViewOfKind:atIndexPath: 方法。 同样，如果它不包含修饰视图，则不需要重写 layoutAttributesForDecorationViewOfKind:atIndexPath: 方法。 返回属性时，不应更新布局属性。 如果你需要更改布局信息，请使布局对象无效并在随后的布局周期中让其更新该数据。

### 连接你的自定义布局以供使用

有两种方法可以将自定义布局链接到集合视图：以编程方式或通过 storyboards。 集合视图通过可写属性 collectionViewLayout 链接到其布局。 要将布局设置为自定义实现，请将集合视图的布局属性设置为自定义布局对象的实例。 清单5-1显示了所需的代码行。

清单 5-1 链接你的自定义布局
```
self.collectionView.collectionViewLayout = [[MyCustomLayout alloc] init];
```

另外，从 storyboard 中打开“文档大纲”面板并选择你的集合视图（它在控制器的下拉菜单中列出）。 选择集合视图后，打开“实用程序”窗格中的“属性”检查器，并在标记为集合视图的部分下方将“流布局”选项从“流”更改为“自定义”。 它下面的选项从滚动方向更改为类，你现在可以选择你的自定义布局类。

## 让你的自定义布局更具吸引力

在布局过程中为每个单元格和视图提供布局属性是必需的，但是还有其他行为可以改善用户对自定义布局的体验。 实施这些行为是可选的，但建议实现。

### 通过补充视图提升内容

补充视图与集合视图的 cells 分开，并具有自己的一组布局属性。 与 cells 类似，这些视图由数据源对象提供，但其目的是增强应用程序的主要内容。 例如，UICollectionViewFlowLayout 为 section header 和 section footer 使用补充视图。 另一个应用程序可以使用补充视图为每个 cell 提供自己的文本标签以显示有关该单元格的信息。 与收集视图单元格一样，补充视图也会经历一个回收过程，以优化收集视图使用的资源数量。 因此，应用程序中使用的所有补充视图都应该从 [UICollectionReusableView](https://developer.apple.com/documentation/uikit/uicollectionreusableview) 类中继承。

为布局添加补充视图的步骤如下所示：

1、使用 [registerClass:forSupplementaryViewOfKind:withReuseIdentifier:](https://developer.apple.com/documentation/uikit/uicollectionview/1618103-register) 或 [registerNib:forSupplementaryViewOfKind:withReuseIdentifier:](https://developer.apple.com/documentation/uikit/uicollectionview/1618101-registernib) 方法将你的补充视图注册到集合视图的布局对象。

2、在你的数据源中，实现 [collectionView:viewForSupplementaryElementOfKind:atIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewdatasource/1618037-collectionview)。 由于这些视图是可重用的，因此请调用 [dequeueReusableSupplementaryViewOfKind:withReuseIdentifier:forIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionview/1618068-dequeuereusablesupplementaryview) 在返回它之前出列或创建一个新的可重用视图并设置所有必需的数据。

3、像为 cells 一样创建补充视图的布局属性对象。

4、将这些布局属性对象包含在由 layoutAttributesForElementsInRect: 方法返回的属性数组中。

5、实现 layoutAttributesForSupplementaryViewOfKind:atIndexPath: 方法在查询时返回指定补充视图的属性对象。

在自定义布局中为补充视图创建属性对象的过程与 cells 的过程几乎完全相同，但不同之处在于，自定义布局可以具有多种类型的补充视图，但仅限于一种类型的 cell。 这是因为补充视图是为了增强主要内容，因此与其分开。 可以通过多种方式补充应用程序的内容，因此每个补充视图的方法都会指定将哪种视图与其他视图区分开来，并允许你的布局根据其类型正确计算其属性。 注册补充视图以供使用时，布局对象使用你提供的字符串来区分视图与其他视图。 有关将补充视图合并到自定义布局的示例，请参阅合并补充视图。

### 在自定义布局中包含装饰视图

装饰视图是可增强集合视图布局外观的视觉装饰。 与 cells 和补充视图不同，装饰视图仅提供可视内容，因此独立于数据源。 你可以使用它们来提供自定义背景，填充 cells 周围的空间，或者根据需要填充 cells。 装饰视图仅由布局对象定义和管理，不与集合视图的数据源对象交互。

要将装饰视图添加到布局，请执行以下操作：

1、使用 [registerClass:forDecorationViewOfKind:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617739-register) 或 [registerNib:forDecorationViewOfKind:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617732-register) 方法将布局对象注册到布局对象。尽管此方法与注册 cell 和补充视图类似，但请记住，在布局对象内注册修饰视图，而不是在数据源内注册。

2、在布局对象的 layoutAttributesForElementsInRect: 方法中，为装饰视图创建属性，就像对 cells 和补充视图所做的一样。

3、在布局对象中实现 [layoutAttributesForDecorationViewOfKind:atIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617809-layoutattributesfordecorationvie) 方法，并在请求时返回装饰视图的属性。

4、或者，实现 [initialLayoutAttributesForAppearingDecorationElementOfKind:atIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617726-initiallayoutattributesforappear) 和 [finalLayoutAttributesForDisappearingDecorationElementOfKind:atIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617762-finallayoutattributesfordisappea) 方法来处理装饰视图外观和消失的动画。 有关更多信息，请参阅[使插入和删除动画更有趣](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/CreatingCustomLayouts/CreatingCustomLayouts.html#//apple_ref/doc/uid/TP40012334-CH5-SW13)。

装饰视图的创建过程与 cells 和补充视图的过程不同。 只需注册类或 nib 文件即可确保装饰视图在需要时创建。 由于它们纯粹是可视化的，装饰视图不需要超出已提供的 nib 文件或对象的 [initWithFrame:](https://developer.apple.com/documentation/uikit/uiview/1622488-initwithframe) 方法所做的任何配置。 因此，当需要装饰视图时，集合视图会为你创建并应用由布局对象提供的属性。 任何装饰视图都应该仍然是 UICollectionReusableView 的子类，因为布局对象使用装饰视图的回收机制。

> 注意：为装饰视图创建属性时，请不要忘记考虑 zIndex 属性。 你可以使用 zIndex 属性将装饰视图分层放置在显示的单元格和补充视图之后（或者，如果你愿意的话）。

### 插入和删除动画更有趣

插入和删除单元格和视图在布局过程中会带来一个有趣的挑战。插入 cell 可能导致其他 cells 和视图的布局更改。即使布局对象知道如何将现有 cells 和视图从其当前位置动画到新位置，但它当前没有插入 cells 的位置。集合视图不是在没有动画的情况下插入新 cell，而是要求布局对象提供一组用于动画的初始属性。同样，当 cell 被删除时，集合视图会要求布局对象提供一组用于任何动画端点的最终属性。

要了解初始属性的工作原理，有助于看到一个示例。起始布局（图5-3）显示了一个最初只包含三个 cells 的集合视图。插入新 cell 时，集合视图会要求布局对象为要插入的单元格提供初始属性。在这种情况下，布局对象将 cell 的初始位置设置为集合视图的中间位置，并将其 alpha 值设置为 0 以隐藏它。在动画制作过程中，这个新单元似乎淡入，并从收集视图的中心移动到其右下角的最终位置。

图 5-3 指定屏幕上出现的项目的初始属性
![](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/Art/custom_insert_animations_2x.png)

清单 5-2 显示了你可能用于指定图5-3中插入单元的初始属性的代码。 此方法将 cell 的位置设置为集合视图的中心并使其透明。 布局对象然后将为 cell 提供最终位置和 alpha 作为正常布局过程的一部分。

清单 5-2 指定插入单元格的初始属性
```
- (UICollectionViewLayoutAttributes *)initialLayoutAttributesForAppearingItemAtIndexPath:(NSIndexPath *)itemIndexPath {
   UICollectionViewLayoutAttributes* attributes = [self layoutAttributesForItemAtIndexPath:itemIndexPath];
   attributes.alpha = 0.0;
 
   CGSize size = [self collectionView].frame.size;
   attributes.center = CGPointMake(size.width / 2.0, size.height / 2.0);
   return attributes;
}
```

> 注意：当插入一个 cell 时，清单5-2将为所有 cells 设置动画，因此插入第四个单元格之前已存在的三个 cells 也会从集合视图的中心弹出。 要仅为要插入的 cells 创建动画，请检查该项目的 index path 是否与传递给 [prepareForCollectionViewUpdates:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617784-prepareforcollectionviewupdates) 方法的项目的 index path 匹配，并且仅在找到匹配项时才执行动画。 否则，返回通过调用 super 的 [initialLayoutAttributesForAppearingItemAtIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617789-initiallayoutattributesforappear) 方法返回的属性。

处理删除的过程与插入过程相同，除了你指定最终属性而不是初始属性。 在前面的例子中，如果你使用了插入 cells 时使用的相同属性，那么删除 cell 会导致它在移动到集合视图中心时淡出。 在 UICollectionViewLayout 类中有六种可用的方法 - 对 item，补充视图和装饰视图的两种单独方法（对于初始属性和最终属性）。

### 改善布局的滚动体验

你的自定义布局对象可以影响集合视图的滚动行为，以创建更好的用户体验。当滚动相关的触摸事件结束时，滚动视图基于当前实际的速度和减速率来确定滚动内容的最终放置位置。当集合视图知道该位置时，如果通过调用其 [targetContentOffsetForProposedContentOffset:withScrollingVelocity:](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617729-targetcontentoffsetforproposedco) 方法来修改位置，它会询问其布局对象。因为它在底层内容仍在移动时调用此方法，所以你的自定义布局会影响滚动内容的最终静止点。

图 5-4 演示了如何使用布局对象来更改集合视图的滚动行为。假设集合视图偏移量从（0，0）开始并且用户向左滑动。集合视图计算滚动自然停止的位置，并将该值作为“提议的”内容偏移值提供。你的布局对象可能会更改建议的值以确保滚动停止时，某个项目正好位于集合视图的可见范围内。这个新值将成为目标内容偏移量，并且是你从 targetContentOffsetForProposedContentOffset:withScrollingVelocity: 方法返回的值。

图 5-4 将建议的内容偏移量更改为更适当的值
![](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/Art/custom_target_scroll_offset_2x.png)

## 实现自定义布局的提示

以下是实现自定义布局对象的一些技巧和建议：

- 考虑使用 prepareLayout 方法来创建并存储稍后需要的 UICollectionViewLayoutAttributes 对象。 集合视图将在某个时刻要求布局属性对象，因此在某些情况下，先创建并存储它们是有意义的。 如果你的 items 数量较少（几百个），或者这些项目的实际布局属性很少发生更改，则情况尤其如此。

    但是，如果你的布局需要管理数千个 items，则需要权衡缓存与重新计算的好处。 对于布局不经常变化的可变大小的项目，缓存通常不需要定期重新计算复杂的布局信息。 对于大量固定大小的项目，按需计算属性可能会更简单。 对于属性频繁变化的项目，无论如何你可能会重新计算，因此缓存可能会占用额外的内存空间。

- 避免子类化 UICollectionView。 集合视图几乎没有或没有它自己的外观。 相反，它会从你的数据源对象以及布局对象中的所有与布局相关的信息中提取其所有视图。 如果你尝试以三维布局项目，则正确的方法是实施自定义布局，以便适当地设置每个 cell 和视图的3D变换。

- 切勿从自定义布局对象的 layoutAttributesForElementsInRect: 方法中调用 UICollectionView的visibleCells 方法。 集合视图不知道 items 的位置，除了布局对象告诉它的位置。 所以要求可见的 cells 就是将请求转发给你的布局对象。

    你的布局对象应该始终知道内容区域中项目的位置，并且应该能够随时返回这些项目的属性。 在大多数情况下，它应该自己做。 在数量有限的情况下，布局对象可能依赖数据源中的信息来定位项目。 例如，显示地图上项目的布局可能会从数据源中检索每个项目的地图位置。

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~