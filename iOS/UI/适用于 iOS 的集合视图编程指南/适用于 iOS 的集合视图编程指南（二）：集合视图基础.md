# 集合视图基础

为了在屏幕上显示其内容，集合视图与许多不同的对象协作。 一些对象是自定义的，并且必须由你的应用程序提供。 例如，你的应用程序必须提供一个数据源对象，告诉集合视图有多少项要显示。 其他对象由 UIKit 提供，并且是基本集合视图设计的一部分。

像表一样，集合视图是面向数据的对象，其实现涉及与应用程序对象的协作。 了解你在代码中必须做什么需要有关集合视图如何完成它的一些背景信息。

## 集合视图是对象的协作

集合视图的设计将所呈现的数据与数据在屏幕上排列和呈现的方式分开。 虽然你的应用程序严格负责管理要呈现的数据，但其视觉呈现由许多不同的对象管理。 表1-1列出了 UIKit 中的集合视图类，并通过它们在实现集合视图界面中扮演的角色对它们进行组织。 大多数类都是按照原样使用的，不需要任何子类化，所以通常可以用很少的代码实现一个集合视图。 当你想超越提供的行为时，你可以继承并提供这种行为。

表 1-1 用于实现集合视图的类和协议

| Purpose | Classes/Protocols | Description |
| - | - | - |
| Top-level containment and management | [UICollectionView](https://developer.apple.com/documentation/uikit/uicollectionview) [UICollectionViewController](https://developer.apple.com/documentation/uikit/uicollectionviewcontroller) | UICollectionView 对象为集合视图的内容定义可见区域。 这个类从 [UIScrollView](https://developer.apple.com/documentation/uikit/uiscrollview) 继承，并可根据需要包含一个大的可滚动区域。 该类还可以根据从其布局对象接收到的布局信息来简化数据的表示。 UICollectionViewController 对象为集合视图提供视图控制器级管理支持。 它的使用是可选的。 |
| Content management | [UICollectionViewDataSource](https://developer.apple.com/documentation/uikit/uicollectionviewdatasource) protocol [UICollectionViewDelegate](https://developer.apple.com/documentation/uikit/uicollectionviewdelegate) protocol | 数据源对象是与集合视图关联的最重要的对象，并且是你必须提供的对象。 数据源管理集合视图的内容并创建呈现该内容所需的视图。 要实现数据源对象，你必须创建一个遵守 UICollectionViewDataSource 协议的对象。 集合视图委托对象允许你从集合视图中截取有趣的消息并自定义视图的行为。 例如，你可以使用委托对象跟踪集合视图中项目的选择和突出显示。 与数据源对象不同，委托对象是可选的。 有关如何实现数据源和委托对象的信息，请参阅[设计数据源和委托](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/CreatingCellsandViews/CreatingCellsandViews.html#//apple_ref/doc/uid/TP40012334-CH7-SW1)。 |
| Presentation | [UICollectionReusableView](https://developer.apple.com/documentation/uikit/uicollectionreusableview) [UICollectionViewCell](https://developer.apple.com/documentation/uikit/uicollectionviewcell) | 显示在集合视图中的所有视图必须是 UICollectionReusableView 类的实例。 该类支持集合视图使用的回收机制。 重用视图（而不是创建新视图）可以提高整体性能，尤其是在滚动期间可以提高性能。 UICollectionViewCell 对象是用于主数据项的特定类型的可重用视图。 |
| Layout | [UICollectionViewLayout](https://developer.apple.com/documentation/uikit/uicollectionviewlayout) [UICollectionViewLayoutAttributes](https://developer.apple.com/documentation/uikit/uicollectionviewlayoutattributes) [UICollectionViewUpdateItem](https://developer.apple.com/documentation/uikit/uicollectionviewupdateitem) | UICollectionViewLayout 的子类称为布局对象，负责定义集合视图内单元和可重用视图的位置，大小和可视属性。在布局过程中，布局对象会创建布局属性对象（UICollectionViewLayoutAttributes 类的实例），告诉集合视图在何处以及如何显示单元格和可重用视图。每当在集合视图中插入，删除或移动数据项时，布局对象都会接收 UICollectionViewUpdateItem 类的实例。 你永远不需要自己创建这个类的实例。有关布局对象的更多信息，请参阅 [The Layout Object Controls the Visual Presentation](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/CollectionViewBasics/CollectionViewBasics.html#//apple_ref/doc/uid/TP40012334-CH2-SW14)。 |
| Flow layout | [UICollectionViewFlowLayout](https://developer.apple.com/documentation/uikit/uicollectionviewflowlayout) [UICollectionViewDelegateFlowLayout](https://developer.apple.com/documentation/uikit/uicollectionviewdelegateflowlayout) protocol | UICollectionViewFlowLayout 类是用于实现网格或其他基于行的布局的具体布局对象。 你可以按原样使用该类，或者与流代理对象结合使用，从而允许你动态地自定义布局信息。 | 

图1-1显示了与集合视图关联的核心对象之间的关系。 集合视图获取有关要从其数据源显示的单元格的信息。 数据源和委托对象是你的应用程序提供的用于管理内容的自定义对象，包括 cells 的选择和突出显示。 布局对象负责决定这些单元格所属的位置，并以一个或多个布局属性对象的形式将该信息发送到集合视图。 集合视图然后将布局信息与实际 cells（和其他视图）合并以创建最终的视觉呈现。

图 1-1 合并内容和布局以创建最终视觉效果
![](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/Art/cv_objects_2x.png)

创建集合视图界面时，首先将 UICollectionView 对象添加到 storyboard 或 nib 文件。 将集合视图视为中心枢纽，所有其他物体都从中心枢纽发出。 添加该对象后，你可以开始配置任何相关对象，例如数据源或委托。 所有配置都以集合视图本身为中心。 例如，不创建集合视图对象也就从不创建布局对象。

## 可重用视图提高性能

集合视图采用视图重用来提高效率。当视图移出屏幕时，它们将从视图中移除并置于重用队列中，而不是被删除。随着新内容在屏幕上滚动，视图将从队列中出列并用新内容改变用途。为了便于回收和重用，集合视图显示的所有视图必须从 [UICollectionReusableView](https://developer.apple.com/documentation/uikit/uicollectionreusableview) 类继承。

集合视图支持三种不同类型的可重用视图，每种视图都有特定的预期用法：

- Cells 显示是集合视图的主要内容。Cell 的工作是呈现数据源对象中单个 item 的内容。每个 cell 都必须是 [UICollectionViewCell](https://developer.apple.com/documentation/uikit/uicollectionviewcell) 类的一个实例，你可以根据需要对其进行子类化以显示你的内容。Cell 对象为管理自己的选择和突出显示状态提供了固有的支持。要将高亮实际应用于 cell，你必须编写一些自定义代码。有关实现 cell 突出显示/选择的信息，请参阅[管理选择和突出显示的视觉状态](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/CreatingCellsandViews/CreatingCellsandViews.html#//apple_ref/doc/uid/TP40012334-CH7-SW8)。
- 补充视图显示关于部分的信息。像 cell 一样，补充视图是数据驱动的。与 cell 不同，补充视图不是必需的，它们的使用和放置由正在使用的布局对象控制。例如，流布局支持页眉和页脚作为可选补充视图。
- 装饰视图是由布局对象完全拥有的可视装饰，并且不受任何数据源对象中的任何数据束缚。例如，布局对象可能使用装饰视图来实现自定义背景外观。

与表视图不同，集​​合视图不会在数据源提供的单元格和补充视图上施加特定样式。相反，基本的可重用视图类是空白的画布供你修改。例如，你可以使用它们构建小视图层次结构，显示图像，甚至动态地绘制内容。

你的数据源对象负责提供与其关联的集合视图使用的单元格和补充视图。但是，数据源从不直接创建视图。当被问及一个视图时，数据源使用集合视图的方法将所需类型的视图出列。出列方法始终返回有效视图，方法是从重用队列中检索一个视图，或使用你提供的类，nib 文件或 storyboard 创建新视图。

有关如何从数据源创建和配置视图的信息，请参阅[配置 Cells 和补充视图](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/CreatingCellsandViews/CreatingCellsandViews.html#//apple_ref/doc/uid/TP40012334-CH7-SW6)。

## 布局对象控制可视化表示

布局对象全权负责确定集合视图中项目的放置位置和视觉样式。虽然你的数据源对象提供了视图和实际内容，但布局对象将确定这些视图的大小，位置以及其他与外观相关的属性。这种责任分离使得动态改变布局成为可能，而不需要改变由你的应用管理的任何数据对象。

集合视图使用的布局过程与应用程序其余视图使用的布局过程相关，但与之不同。换句话说，不要混淆布局对象与用于在父视图内重新定位子视图的 layoutSubviews 方法。布局对象从不触及其直接管理的视图，因为它实际上并不拥有任何这些视图。相反，它会生成描述集合视图中单元格，补充视图和装饰视图的位置，大小和视觉外观的属性。这就是集合视图的工作，将这些属性应用于实际的视图对象。

布局对象如何影响集合视图中的视图没有限制。布局对象可以移动一些视图而不是其他视图。它只能移动一点点视图，或者可以随意在屏幕上移动它们。它甚至可以重新定位视图而不考虑周围的视图。例如，如果需要，布局对象可以将视图堆叠在彼此之上。唯一真正的限制是布局对象如何影响你希望应用具有的视觉风格。

图1-2显示了垂直滚动流布局如何排列单元格和补充视图。在垂直滚动流布局中，内容区域的宽度保持不变并且高度增长以适应内容。为了计算面积，布局对象一次放置一个视图和单元格，为每个布局对象选择最合适的位置。在流布局的情况下，单元格和补充视图的大小被指定为布局对象上的属性或使用委托。计算布局只是使用这些属性来放置每个视图。

图 1-2 布局对象提供了布局度量
![](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/Art/cv_layout_basics_2x.png)

布局对象不仅控制其视图的大小和位置。 布局对象可以指定其他视图相关属性，例如其透明度，3D 空间中的变换以及其他视图上方或下方的可见性（如果有）。 这些属性可让你创建更有趣的布局。 例如，你可以通过将视图置于另一个视图上并更改它们的 z 顺序来创建单元格堆栈，也可以使用变换在任何轴上旋转它们。

有关布局对象如何履行其对集合视图的职责的详细信息，请参阅[创建自定义布局](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/CreatingCustomLayouts/CreatingCustomLayouts.html#//apple_ref/doc/uid/TP40012334-CH5-SW1)。

## 集合视图自动启动动画

集合视图支持基础级别的动画。插入（或删除）项目或部分时，集合视图会自动为受该更改影响的所有视图生成动画。例如，插入项目时，插入点后的项目通常会移动以为新项目腾出空间。集合视图可以创建这些动画，因为它可以检测项目的当前位置并在插入后计算它们的最终位置。因此，它可以将每个项目从其初始位置动画到其最终位置。

除了动画化插入，删除和移动操作之外，你可以随时使布局无效并强制重新计算其布局属性。使布局失效不会直接动画项目;当你使布局无效时，集合视图会在其新计算的位置显示这些项目，而不会对它们进行动画处理。相反，在自定义布局中，你可以使用此行为来定期定位单元格并创建动画效果。

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~