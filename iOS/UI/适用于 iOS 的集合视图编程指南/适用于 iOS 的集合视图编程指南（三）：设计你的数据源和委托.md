# 设计你的数据源和委托

每个集合视图都必须有一个数据源对象。 数据源对象是你的应用显示的内容。 它可以是应用程序数据模型中的对象，也可以是管理集合视图的视图控制器。 数据源的唯一要求是必须能够提供收集视图所需的信息，例如显示这些项目时有多少项目以及要使用哪些视图。

委托对象是一个可选（但推荐）的对象，用于管理与内容呈现和交互相关的方面。 虽然委托的主要工作是管理 cell 突出显示和选择，但它可以扩展为提供其他信息。 例如，流布局扩展了基本委托行为以自定义布局度量，例如 cells 的大小以及它们之间的间距。

## 数据源管理你的内容

数据源对象是负责管理使用集合视图呈现的内容的对象。 数据源对象必须遵守 [UICollectionViewDataSource](https://developer.apple.com/documentation/uikit/uicollectionviewdatasource) 协议，该协议定义了你必须支持的基本行为和方法。 数据源的工作是为集合视图提供以下问题的答案：

- 集合视图包含多少个 sections？
- 对于给定的 section，一个部分包含多少 items？
- 对于给定的 section 或 item，应该使用哪些视图来显示相应的内容？

section 和 item 是集合视图内容的基本组织原则。 集合视图通常至少包含一个 section，可能包含更多 sections。 每个 section 又包含零个或多个 item。 item 表示要呈现的主要内容，而 section 将这些项目组织为逻辑组。 例如，照片应用可能会使用 section 来表示单张相册或同一天拍摄的一组照片。

集合视图引用它包含的使用 [NSIndexPath](https://developer.apple.com/documentation/foundation/nsindexpath) 对象的数据。 当试图找到一个 item 时，集合视图使用由布局对象提供给它的 index path 信息。 对于 item，index path 包含一个 section 号和一个 item 号。 对于补充和装饰视图，index path 包含由布局对象提供的任何值。 附加到补充和装饰视图的 index path 的含义取决于你的应用程序，但第一个索引对应于数据源中的特定部分。 这些视图的 index path 更多是关于识别而不是意义，以确定哪种视图正在被考虑。 因此，例如，如果你在流布局中看到了为段创建页眉和页脚的补充视图，则 index path 提供的相关信息就是所引用的部分。

> 注：虽然标准 index paths 支持多个级别，但集合视图的单元格仅支持具有“section”和“item”参数的2级深度索引路径，这与 UITableView 类的 index path 非常相似。 如有必要，补充视图和装饰视图可以具有更复杂的 index path。 其 index path > 1的元素被解释为对应于路径中第一个索引指定的部分。 传统上，只需要第二个索引，但补充和装饰视图不限于两个。 设计数据源时请记住这一点。

无论你如何安排数据对象中的 sections 和 items，这些 sections 和 items 的视觉表示仍然由布局对象决定。 不同的布局对象可以非常不同地显示 sections 和 items 数据，如图2-1所示。 在该图中，流布局对象将每个连续部分垂直放置在前一个部分之下。 自定义布局可以将部分放置在非线性布置中，再次展示布局与实际数据的分离。

图 2-1 根据布局对象排列排列的部分
![](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/Art/cv_layout_sections_2x.png)

## 设计你的数据对象

高效的数据源使用 sections 和 items 来帮助组织其底层数据对象。将数据组织成 sections 和 items 使得以后更容易实现数据源方法。由于你的数据源方法经常被调用，因此你需要确保你的这些方法的实现能够尽快检索数据。

一个简单的解决方案（但当然不是唯一的解决方案）是让数据源使用一组嵌套数组，如图2-2所示。在此配置中，顶级数组包含一个或多个数组，代表数据源的各个 sections。每个 section 数组然后包含该 items 的数据项。在 section 中查找 item 是检索其部分数组，然后从该数组中检索 items 的问题。这种安排可以轻松管理中等规模的 items 集合并按需检索单个 item。

图 2-2 使用嵌套数组排列数据对象
![](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/Art/ds_data_object_layout_2x.png)

在设计数据结构时，你总是可以从一组简单的数组开始，并根据需要移至更高效的结构。 一般来说，你的数据对象不应该是性能瓶颈。 集合视图通常只访问你的数据源来计算总共有多少个对象，并获取当前屏幕上的元素的视图。 如果布局对象只依赖数据对象中的数据，那么当数据源包含数千个对象时，性能可能会受到严重影响。

### 告诉集合视图关于你的内容

通过集合视图询问数据源的问题包括其包含的 sections 数以及每 section 包含的 items 数。 集合视图会要求你的数据源在发生以下任何操作时提供此信息：

- 集合视图首次显示。
- 你将一个不同的数据源对象分配给集合视图。
- 你显式调用集合视图的 [reloadData](https://developer.apple.com/documentation/uikit/uicollectionview/1618078-reloaddata) 方法。
- 集合视图委托使用 [performBatchUpdates:completion:](https://developer.apple.com/documentation/uikit/uicollectionview/1618045-performbatchupdates) 或任何移动，插入或删除方法来执行块。

使用 [numberOfSectionsInCollectionView:](https://developer.apple.com/documentation/uikit/uicollectionviewdatasource/1618023-numberofsections) 方法提供 sections 的数量，并使用 [collectionView:numberOfItemsInSection:](https://developer.apple.com/documentation/uikit/uicollectionviewdatasource/1618058-collectionview) 方法提供每 section 中的 items 数。你必须实现 collectionView:numberOfItemsInSection: 方法，但是如果你的集合视图只有一个 section，则实现 numberOfSectionsInCollectionView:  方法是可选的。这两种方法都会用适当的信息返回整数值。

如果你如图2-2所示实现了你的数据源，那么数据源方法的实现可能与清单2-1中所示的一样简单。 在此代码中，_data 变量是存储顶级数组部分的数据源的自定义成员变量。 获取该数组的数量会得出 sections 的数量。 获取其中一个子阵列的计数可得出该部分中的 items 数量。 （当然，你自己的代码应该做任何错误检查，以确保返回的值是有效的。）

清单 2-1 提供 section 和 item 数
```
- (NSInteger)numberOfSectionsInCollectionView:(UICollectionView*)collectionView {
    // _data is a class member variable that contains one array per section.
    return [_data count];
}
 
- (NSInteger)collectionView:(UICollectionView*)collectionView numberOfItemsInSection:(NSInteger)section {
    NSArray* sectionArray = [_data objectAtIndex:section];
    return [sectionArray count];
}
```

## 配置 Cells 和补充视图

数据源的另一个重要任务是提供集合视图用于显示内容的视图。集合视图不会跟踪你应用的内容。它只是获取你给它的视图，并将当前的布局信息应用到它们。因此，视图显示的所有内容都是你的责任。

在你的数据源报告其管理的 sections 数和 items 之后，集合视图会要求布局对象为集合视图的内容提供布局属性。在某些时候，集合视图会要求布局对象提供特定矩形中的元素列表（通常这是可见的矩形）。集合视图使用该列表向你的数据源请求相应的 cells 和补充视图。为了提供这些 cells 和补充视图，你的代码必须执行以下操作：

1、将你的模板 cells 和视图嵌入到 storyboard 文件中。 （或者，为每种支持的 cell 或视图注册类或 nib 文件。）

2、在你的数据源中，请求时将出列并配置相应的 cell 或视图。

为了确保以最有效的方式使用 cell 和补充视图，集合视图承担为你创建这些对象的责任。每个集合视图维护当前未使用的 cells 和补充视图的内部队列。你可以不要自己创建对象，只需询问集合视图即可为你提供所需的视图。如果有人正在等待重复使用队列，则集合视图会对其进行准备并将其快速返回给你。如果没有在等待，集合视图使用已注册的类或 nib 文件创建一个新的并将其返回给你。因此，每次你将某个 cell 或视图取出时，都会获得一个随时可用的对象。

重复使用标识符可以注册多种类型的 cells 和多种类型的补充视图。重用标识符是一个字符串，用于区分注册的 cell 和视图类型。该字符串的内容仅与你的数据源对象相关。但是，当询问视图或 cell 时，可以使用提供的 index path 来确定你可能需要的视图或 cell 类型，然后将适当的重用标识符传递给出队方法。

### 注册 Cells 和补充视图

你可以编程或在应用程序的 storyboard 文件中配置你的集合视图的 cells 和视图。

在 storyboard 中配置 cells 和视图。 在 storyboard 中配置 cells 和补充视图时，可以通过将 item 拖放到你的集合视图并在其中进行配置。 这会在集合视图和相应的 cell 或视图之间建立关系。

- 对于 cells，从对象库中拖出一个 Collection View Cell 并将其放到你的集合视图中。 将你的单元格的自定义类和集合可重用视图标识符设置为适当的值。
- 对于补充视图，从对象库中拖出一个 Collection Reusable View 并将其放到你的集合视图中。 将视图的自定义类和集合可重用视图标识符设置为适当的值。

通过编程配置 cells。 使用 [registerClass:forCellWithReuseIdentifier:](https://developer.apple.com/documentation/uikit/uicollectionview/1618089-registerclass) 或 [registerNib:forCellWithReuseIdentifier:](https://developer.apple.com/documentation/uikit/uicollectionview/1618083-registernib) 方法将你的 cell 与重用标识符相关联。 你可以将这些方法作为父视图控制器初始化过程的一部分。

以编程方式配置补充视图。 使用 [registerClass:forSupplementaryViewOfKind:withReuseIdentifier:](https://developer.apple.com/documentation/uikit/uicollectionview/1618103-register) 或 [registerNib:forSupplementaryViewOfKind:withReuseIdentifier:](https://developer.apple.com/documentation/uikit/uicollectionview/1618101-registernib) 方法将每种视图与重用标识符相关联。 你可以将这些方法作为父视图控制器初始化过程的一部分。

尽管只使用重用标识符注册 cells，但补充视图要求你指定一个称为类别字符串的附加标识符。 每个布局对象都负责定义它支持的补充视图的种类。 例如，UICollectionViewFlowLayout 类支持两种补充视图：节标题视图（section header view）和节脚注视图（section footer view）。 为了识别这两种类型的视图，它定义了字符串常量 [UICollectionElementKindSectionHeader](https://developer.apple.com/documentation/uikit/uicollectionelementkindsectionheader) 和 [UICollectionElementKindSectionFooter](https://developer.apple.com/documentation/uikit/uicollectionelementkindsectionfooter)。 在布局过程中，布局对象包含类型字符串以及该视图类型的其他布局属性。 集合视图然后将信息传递给你的数据源。 你的数据源然后使用种类字符串和重用标识符来决定哪个视图对象出列并返回。

> 注意：如果你实现自己的自定义布局，则需要负责定义布局支持的补充视图的种类。 布局可以支持任意数量的补充视图，每个补充视图都有其自己的类型字符串。 有关定义自定义布局的更多信息，请参阅[创建自定义布局](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/CreatingCustomLayouts/CreatingCustomLayouts.html#//apple_ref/doc/uid/TP40012334-CH5-SW1)。

注册是一次性事件，必须在尝试将任何 cells 或视图出列之前进行。 注册后，你可以根据需要出列尽可能多的 cells 或视图，而无需重新注册它们。 建议你在将一个或多个项目出列后更改注册信息。 最好注册一次你的单元格和视图并完成它。

### 出列和配置 Cells 和视图

数据源对象负责在集合视图中提供 cells 和补充视图。 [UICollectionViewDataSource](https://developer.apple.com/documentation/uikit/uicollectionviewdatasource) 协议包含两个用于此目的的方法：[collectionView:cellForItemAtIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewdatasource/1618029-collectionview) 和 [collectionView:viewForSupplementaryElementOfKind:atIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewdatasource/1618037-collectionview)。 因为 cells 是集合视图的必需元素，所以你的数据源必须实现 collectionView:cellForItemAtIndexPath: 方法，但 collectionView:viewForSupplementaryElementOfKind:atIndexPath: 方法是可选的，并且取决于所用布局的类型。 在这两种情况下，这些方法的实现都遵循一个非常简单的模式：

1、使用 [dequeueReusableCellWithReuseIdentifier:forIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionview/1618063-dequeuereusablecellwithreuseiden) 或 [dequeueReusableSupplementaryViewOfKind:withReuseIdentifier:forIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionview/1618068-dequeuereusablesupplementaryview) 取出一个 cell 或适当类型的视图。
2、使用指定 index path 的数据配置视图。
3、返回视图。

出队过程旨在减轻你必须自己创建 cell 或视图的责任。 只要你以前注册了一个 cell 或视图，就可以保证出列方法永远不会返回 nil。 如果在重用队列中没有给定类型的单元格或视图，则使用 storyboard 或使用你注册的类或 nib 文件，出列方法会简单地创建一个 cell 或视图。

从出队过程返回给你的 cell 应处于原始状态，并准备好用新数据进行配置。 对于必须创建的 cell 或视图，出列进程使用常规进程创建并初始化它 - 也就是通过从 storyboard 或 nib 文件加载视图或创建新实例并使用 [initWithFrame:](https://developer.apple.com/documentation/uikit/uiview/1622488-initwithframe) 方法初始化它。 相比之下，不是从头开始创建的项目，而是从重用队列中检索的 item 可能已包含以前用法的数据。 在这种情况下，出列方法调用 item 的 [prepareForReuse](https://developer.apple.com/documentation/uikit/uicollectionreusableview/1620141-prepareforreuse) 方法，使其有机会返回到原始状态。 当你实现自定义 cell 或视图类时，可以重写此方法以将属性重置为默认值并执行任何其他清理。

数据源将该视图出列后，它将使用其新数据配置该视图。 你可以使用传递给你的数据源方法的 index path 来找到适当的数据对象，然后将该对象的数据应用于视图。 配置视图后，将其从方法中返回并完成。 清单2-2显示了如何配置 cell 的简单示例。 在将该 cell 出列后，该方法使用关于该 cell 位置的信息设置该 cell 的自定义标签，然后返回该 cell。

清单 2-2 配置一个自定义单元
```
- (UICollectionViewCell *)collectionView:(UICollectionView *)collectionView
                  cellForItemAtIndexPath:(NSIndexPath *)indexPath {
   MyCustomCell* newCell = [self.collectionView dequeueReusableCellWithReuseIdentifier:MyCellID
                                                                          forIndexPath:indexPath];
 
   newCell.cellLabel.text = [NSString stringWithFormat:@"Section:%d, Item:%d", indexPath.section, indexPath.item];
   return newCell;
}
```

> 注意：从数据源返回视图时，始终返回有效视图。 返回 nil，即使由于某种原因被请求的视图不应该显示，导致断言，并且你的应用程序终止，因为布局对象期望这些方法返回有效的视图。

## 插入，删除和移动 Sections 和 Items

要插入，删除或移动单个 section 或 item，请按照下列步骤操作：

1、更新数据源对象中的数据。
2、调用集合视图的相应方法来插入或删除 section 或 item。

在通知任何更改的集合视图之前更新数据源至关重要。 集合视图方法假定你的数据源包含当前正确的数据。 如果没有，集合视图可能会从你的数据源收到错误的一组项目，或者请求那些不在那里的项目并导致你的应用程序崩溃。

以编程方式添加，删除或移动单个项目时，集合视图的方法会自动创建动画以反映更改。 但是，如果要一起动画多个更改，则必须在块内执行所有插入，删除或移动调用，并将该块传递给 [performBatchUpdates:completion:](https://developer.apple.com/documentation/uikit/uicollectionview/1618045-performbatchupdates) 方法。 批量更新过程然后同时为所有更改设置动画，你可以自由混合调用以在同一个 block 内插入，删除或移动项目。

清单2-3显示了如何执行批量更新以删除当前选定项目的简单示例。 传递给 performBatchUpdates:completion: 方法的 block 首先调用一个自定义方法来更新数据源。 然后它会通知集合视图删除这些项目。 你提供的更新 block 和完成 block 都是同步执行的。

清单 2-3 删除选定的 items 
```
[self.collectionView performBatchUpdates:^{
   NSArray* itemPaths = [self.collectionView indexPathsForSelectedItems];
 
   // Delete the items from the data source.
   [self deleteItemsFromDataSourceAtIndexPaths:itemPaths];
 
   // Now delete the items from the collection view.
   [self.collectionView deleteItemsAtIndexPaths:itemPaths];
} completion:nil];
```

## 管理选择和高亮的视觉状态

集合视图默认支持单项选择，并可配置为支持多项选择或完全禁用选择。 集合视图检测其边界内的分接点，并相应地突出显示或选择相应的单元格。 大多数情况下，集合视图仅修改 cell 的属性以指示它被选中或高亮; 它不会改变你的 cell 的视觉外观，只有一个例外。 如果 cell 的 [selectedBackgroundView](https://developer.apple.com/documentation/uikit/uicollectionviewcell/1620138-selectedbackgroundview) 属性包含有效视图，则该 cell 高亮或选中时，集合视图将显示该视图。

清单2-4显示了可以合并到自定义集合视图单元的实现中的代码，以便高亮和选定状态的外观变化。 cell 的 [backgroundView](https://developer.apple.com/documentation/uikit/uicollectionviewcell/1620131-backgroundview) 属性将始终是 cell 首次加载时以及 cell 未高亮或未选中时的默认视图。 [selectedBackgroundView](https://developer.apple.com/documentation/uikit/uicollectionviewcell/1620138-selectedbackgroundview) 属性将替换默认背景视图，只要某个单元格被高亮或选中。 在这种情况下，选中或高亮时，cell 的背景颜色将从红色变为白色。

清单 2-4 设置背景视图以指示更改的状态
```
UIView* backgroundView = [[UIView alloc] initWithFrame:self.bounds];
backgroundView.backgroundColor = [UIColor redColor];
self.backgroundView = backgroundView;
 
UIView* selectedBGView = [[UIView alloc] initWithFrame:self.bounds];
selectedBGView.backgroundColor = [UIColor whiteColor];
self.selectedBackgroundView = selectedBGView;
```

集合视图的委托为集合视图提供以下方法以便高亮和选择：

- [collectionView:shouldSelectItemAtIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewdelegate/1618095-collectionview)
- [collectionView:shouldDeselectItemAtIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewdelegate/1618067-collectionview)
- [collectionView:didSelectItemAtIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewdelegate/1618032-collectionview)
- [collectionView:didDeselectItemAtIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewdelegate/1618035-collectionview)
- [collectionView:shouldHighlightItemAtIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewdelegate/1618070-collectionview)
- [collectionView:didHighlightItemAtIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewdelegate/1618049-collectionview)
- [collectionView:didUnhighlightItemAtIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewdelegate/1618027-collectionview)

这些方法为你提供了很多机会来调整集合视图的高亮/选择行为，以达到确切的所需规格。

例如，如果你更喜欢自己绘制单元格的选择状态，则可以将 selectedBackgroundView 属性设置为 nil，并使用委托对象将任何可视化更改应用于单元格。 你可以在 [collectionView:didSelectItemAtIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewdelegate/1618032-collectionview) 方法中应用可视化更改，并在 collectionView:didDeselectItemAtIndexPath: 方法中将其删除。

如果你希望自己绘制高亮状态，则可以覆盖 [collectionView:didHighlightItemAtIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewdelegate/1618049-collectionview) 和 [collectionView:didUnhighlightItemAtIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewdelegate/1618027-collectionview) 委托方法，并使用它们应用高亮。 如果你还在 selectedBackgroundView 属性中指定了视图，则应该更改 cell 的内容视图以确保你的更改可见。 清单2-5显示了一种使用内容视图的背景色更改高亮的简单方法。

清单 2-5 将临时高亮显示应用于 cell
```
- (void)collectionView:(UICollectionView *)colView didHighlightItemAtIndexPath:(NSIndexPath *)indexPath {
    UICollectionViewCell* cell = [colView cellForItemAtIndexPath:indexPath];
    cell.contentView.backgroundColor = [UIColor blueColor];
}
 
- (void)collectionView:(UICollectionView *)colView didUnhighlightItemAtIndexPath:(NSIndexPath *)indexPath {
    UICollectionViewCell* cell = [colView cellForItemAtIndexPath:indexPath];
    cell.contentView.backgroundColor = nil;
}
```

cell 高亮的状态与其选中状态之间存在细微但重要的区别。高亮的状态是一种过渡状态，你可以使用该状态在用户的手指仍在触摸设备时将可见高光应用于 cell。此状态仅在集合视图正在跟踪 cell 上的触摸事件时设置为 YES。当触摸事件停止时，高亮显示的状态返回值 NO。相比之下，选中状态仅在一系列触摸事件结束后才发生改变 - 具体地，当那些触摸事件指示用户试图选择该 cell 时。

图2-3显示了用户触摸未选中 cell 时发生的一系列步骤。初始触摸事件会导致集合视图将 cell 的高亮显示状态更改为 YES，但这样做不会自动更改 cell 的外观。如果在 cell 中发生最终修饰事件，高亮显示的状态将返回到 NO，并且集合视图将所选状态更改为 YES。当用户更改选中状态时，集合视图将在 cell 的 selectedBackgroundView 属性中显示视图，但这是集合视图对 cell 进行的唯一可视化更改。任何其他视觉更改必须由你的委托对象进行。

图 2-3 跟踪单元格中的触摸
![](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/Art/cell_selection_semantics_2x.png)

无论用户是选择还是取消选择 cell，cell 的选定状态总是最后要改变的。 单元格中的点击总是会导致首先更改单元格高亮显示的状态。 只有在 tap 序列结束并且在该序列期间应用了任何高亮部分后，才会更改所选 cell 的状态。 在设计 cell 时，应确保高亮显示和选中状态的视觉外观不会以非预期方式发生冲突。

## 显示 Cell 的编辑菜单

当用户在 cell 上执行长按手势时，集合视图将尝试显示该 cell 的编辑菜单。 编辑菜单可用于剪切，复制和粘贴集合视图中的 cell。 在显示编辑菜单之前必须满足几个条件：

- delegate 必须实现与处理操作相关的所有三种方法：

    [collectionView:shouldShowMenuForItemAtIndexPath:](https://developer.apple.com/documentation/uikit/uicollectionviewdelegate/1618010-collectionview)

    [collectionView:canPerformAction:forItemAtIndexPath:withSender:](https://developer.apple.com/documentation/uikit/uicollectionviewdelegate/1618051-collectionview)

    [collectionView:performAction:forItemAtIndexPath:withSender:](https://developer.apple.com/documentation/uikit/uicollectionviewdelegate/1618073-collectionview)

- collectionView:shouldShowMenuForItemAtIndexPath: 方法必须为指定的 cell 返回 YES。
- collectionView:canPerformAction:forItemAtIndexPath:withSender: 方法必须为至少一个所需操作返回 YES。 集合视图支持以下操作：

    cut:

    copy:

    paste:

如果满足这些条件并且用户从菜单中选择操作，集合视图会调用委托的 collectionView:performAction:forItemAtIndexPath:withSender: 方法对指定的项目执行操作。

清单2-6显示了如何防止出现其中一个菜单项。 在此示例中，[collectionView:canPerformAction:forItemAtIndexPath:withSender:](https://developer.apple.com/documentation/uikit/uicollectionviewdelegate/1618051-collectionview) 方法可防止“剪切”菜单项出现在“编辑”菜单中。 它启用复制和粘贴项目，以便用户可以插入内容。

清单 2-6 在编辑菜单中选择性地禁用动作
```
- (BOOL)collectionView:(UICollectionView *)collectionView
        canPerformAction:(SEL)action
        forItemAtIndexPath:(NSIndexPath *)indexPath
        withSender:(id)sender {
   // Support only copying and pasting of cells.
   if ([NSStringFromSelector(action) isEqualToString:@"copy:"]
      || [NSStringFromSelector(action) isEqualToString:@"paste:"])
      return YES;
 
   // Prevent all other actions.
   return NO;
}
```

有关使用粘贴板命令的更多信息，请参阅[适用于 iOS 的文本编程指南](https://developer.apple.com/library/content/documentation/StringsTextFonts/Conceptual/TextAndWebiPhoneOS/Introduction/Introduction.html#//apple_ref/doc/uid/TP40009542)。

## 在布局之间切换

在布局之间转换最简单的方法是使用 [setCollectionViewLayout:animated:](https://developer.apple.com/documentation/uikit/uicollectionview/1618086-setcollectionviewlayout) 方法。 但是，如果你需要控制转换或希望它是交互式的，请使用 [UICollectionViewTransitionLayout](https://developer.apple.com/documentation/uikit/uicollectionviewtransitionlayout) 对象。

UICollectionViewTransitionLayout 类是一种特殊类型的布局，当转换为新布局时，它将作为集合视图的布局对象进行安装。 通过转换布局对象，你可以让对象沿着非线性路径，使用不同的时序算法，或根据传入的触摸事件进行移动。 标准类提供到新布局的线性转换，但与 [UICollectionViewLayout](https://developer.apple.com/documentation/uikit/uicollectionviewlayout) 类类似，UICollectionViewTransitionLayout 类可以被分类以创建任何所需的效果。 这样做时，你需要实现与创建自定义布局时相同的方法，并允许你的实现适应来自用户的输入，通常来自手势识别器。 有关创建自定义布局对象的更多信息，请参阅[创建自定义布局](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/CreatingCustomLayouts/CreatingCustomLayouts.html#//apple_ref/doc/uid/TP40012334-CH5-SW1)。

UICollectionViewLayout 类提供了几种跟踪布局之间转换的方法。 UICollectionViewTransitionLayout 对象通过 [transitionProgress](https://developer.apple.com/documentation/uikit/uicollectionviewtransitionlayout/1622191-transitionprogress) 属性跟踪转换的完成情况。随着转换的发生，你的代码会定期更新此属性以指示转换的完成百分比。例如，将 UICollectionViewTransitionLayout 类与可用于在布局之间切换的对象（如手势识别器）结合使用，可以创建交互式切换。另外，如果实现自定义转换布局对象，则 UICollectionViewTransitionLayout 类提供了两种跟踪与布局相关的值的方法：[updateValue:forAnimatedKey:](https://developer.apple.com/documentation/uikit/uicollectionviewtransitionlayout/1622194-updatevalue) 和 [valueForAnimatedKey:](https://developer.apple.com/documentation/uikit/uicollectionviewtransitionlayout/1622193-valueforanimatedkey) 方法。这些方法会跟踪在转换期间可以设置和更改的特殊浮点值，以便与布局重要信息进行通信。例如，如果你使用捏合手势在布局之间进行转换，则可以使用这些方法来告诉转换布局对象视图需要彼此偏移的偏移量。

在你的应用程序中包含 UICollectionViewTransitionLayout 对象的步骤如下所示：

1、使用 [initWithCurrentLayout:nextLayout:](https://developer.apple.com/documentation/uikit/uicollectionviewtransitionlayout/1622189-init) 方法创建标准类或自定义类的实例。
2、通过定期修改 [transitionProgress](https://developer.apple.com/documentation/uikit/uicollectionviewtransitionlayout/1622191-transitionprogress) 属性来沟通转换的进度。 在更改转换的进度后，不要忘记使用集合视图的 [invalidateLayout](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617728-invalidatelayout) 方法使布局无效。
3、在集合视图的委托中实现 [collectionView:transitionLayoutForOldLayout:newLayout:](https://developer.apple.com/documentation/uikit/uicollectionviewdelegate/1618100-collectionview) 方法并返回你的转换布局对象。
4、也可以使用 [updateValue:forAnimatedKey:](https://developer.apple.com/documentation/uikit/uicollectionviewtransitionlayout/1622194-updatevalue) 方法为布局修改值，以指示与布局对象相关的已更改值。 这种情况下的稳定值是0。

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~