# 自定义布局：一个工作示例

创建自定义集合视图布局非常简单，只需简单的需求，但该过程的实现细节可能会有所不同。 你的布局必须为你的集合视图包含的每个视图生成布局属性对象。 这些属性的创建顺序取决于应用程序的性质。 使用包含数千个项目的集合视图，预先计算和缓存布局属性是一个耗时的过程，因此只有在请求特定项目时才创建属性更有意义。 对于包含更少项目的应用程序，只需计算一次布局信息并将其缓存到引用即可，这样可以节省应用程序大量不必要的重新计算。 本章中的工作示例属于第二类。

请记住，提供的示例代码决不是创建自定义布局的明确方式。 在开始创建自定义布局之前，请花点时间设计一个实施结构，使你的应用最适合获得最佳性能。 有关布局自定义过程的概念性概述，请参阅[创建自定义布局](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/CreatingCustomLayouts/CreatingCustomLayouts.html#//apple_ref/doc/uid/TP40012334-CH5-SW1)。

由于本章按照特定顺序提供了自定义布局实现，因此请从头到尾按照具体实现目标的示例进行操作。 本章重点介绍创建自定义布局，而不是实现完整的应用程序。 因此，不提供用于创建最终产品的视图和控制器的实现。 布局使用自定义集合视图 cell 作为其 cell，并使用自定义视图创建将 cells 彼此连接的线条。 为集合视图创建自定义 cells 和视图以及使用集合视图的要求在前面的章节中介绍。 要查看此信息，请参阅[集合视图基础](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/CollectionViewBasics/CollectionViewBasics.html#//apple_ref/doc/uid/TP40012334-CH2-SW1)和[设计数据源和委托](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/CreatingCellsandViews/CreatingCellsandViews.html#//apple_ref/doc/uid/TP40012334-CH7-SW1)。

## 概念

此工作示例的目的是实现用于显示信息的分层树状结构的自定义布局，如图6-1所示。 该示例提供了代码片段，随后是代码的解释，以及你所达到的定制过程中的要点。 集合视图的每个部分构成树的一个深度级别：部分0仅包含 NSObject 单元。 第1部分包含 NSObject 的所有子单元。 第2部分包含这些孩子的所有孩子，等等。 每个 cell 都是一个自定义 cell，具有相关类名称的标签，cells 之间的连接是补充视图。 由于连接器视图类必须确定要绘制多少个连接，因此需要访问数据源中的数据。 因此，将这些连接作为补充视图而非装饰视图来实现是有意义的。

图 6-1 类层次结构
![](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/Art/example_final_screen_2x.png)

## 初始化

创建自定义布局的第一步是对 UICollectionViewLayout 类进行子类化。 这样做为你提供构建自定义布局所需的基础。

对于这个例子，自定义协议是必要的，以通知某些 items 之间的布局的间距。 如果特定 item 的属性需要来自数据源的额外信息，则最好为自定义布局实施协议，而不是直接连接到数据源。 你的结果布局更加稳健且可重用; 它不会附加到特定的数据源，而是会响应实现其协议的任何对象。

清单6-1显示了自定义布局的头文件的必要代码。 现在，任何实现 MyCustomProtocol 协议的类都可以使用自定义布局，并且布局可以查询该类以获取所需的信息。

清单 6-1 连接到自定义协议
```
@interface MyCustomLayout : UICollectionViewLayout
@property (nonatomic, weak) id<MyCustomProtocol> customDataSource;
@end
```

接下来，因为集合视图将管理的项目数量相对较少，所以自定义布局利用缓存系统来存储它在准备布局时生成的布局属性，然后在集合视图请求它们时检索这些存储的值。清单6-2显示了我们的布局需要维护的三个私有属性以及 init 方法。 layoutInformation 字典包含我们的集合视图中所有类型视图的所有布局属性，maxNumRows 属性跟踪需要多少行来填充我们树的最高列。 insets 对象控制 cells 之间的间距，并用于设置视图和内容大小的帧。前两个属性的值在准备布局时设置，但应使用 init 方法设置 insets 对象。在这种情况下，INSET_TOP，INSET_LEFT，INSET_BOTTOM 和 INSET_RIGHT 指的是你为每个参数定义的常量。

清单 6-2 初始化变量
```
@interface MyCustomLayout()
 
@property (nonatomic) NSDictionary *layoutInformation;
@property (nonatomic) NSInteger maxNumRows;
@property (nonatomic) UIEdgeInsets insets;
 
@end
 
-(id)init {
    if(self = [super init]) {
        self.insets = UIEdgeInsetsMake(INSET_TOP, INSET_LEFT, INSET_BOTTOM, INSET_RIGHT);
    }
    return self;
}
```

此自定义布局的最后一步是创建自定义布局属性。 虽然这一步并不总是必要的，但在这种情况下，当 cells 被放置时，代码需要访问当前 cells 的子元素的index path，以便它可以调整子 index path 的框架以匹配其父元素的框架。 因此，通过继承 [UICollectionViewLayoutAttributes](https://developer.apple.com/documentation/uikit/uicollectionviewlayoutattributes) 来存储 cells 的子元素的数组即可提供该信息。 你继承 UICollectionViewLayoutAttributes，并在头文件中添加如下代码：

```
@property (nonatomic) NSArray *children;
```

正如 UICollectionViewLayoutAttributes 类引用中所解释的，子类化布局属性要求你覆盖 iOS 7 及更高版本中继承的 [isEqual:](https://developer.apple.com/documentation/objectivec/1418956-nsobject/1418795-isequal) 方法。 有关更多信息，请参阅 [UICollectionViewLayoutAttributes 类参考](https://developer.apple.com/documentation/uikit/uicollectionviewlayoutattributes)。

在这种情况下，isEqual: 方法的实现很简单，因为只有一个字段可以比较 - children 数组的内容。 如果两个布局属性对象的数组都匹配，那么它们必须相等，因为孩子只能属于一个类。 清单 6-3 显示了 isEqual: 方法的实现。

清单 6-3 满足子类布局属性的需求
```
-(BOOL)isEqual:(id)object {
    MyCustomAttributes *otherAttributes = (MyCustomAttributes *)object;
    if ([self.children isEqualToArray:otherAttributes.children]) {
        return [super isEqual:object];
    }
    return NO;
}
```

请记住在自定义布局文件中包含自定义布局属性的头文件。

在此过程中，你已准备好开始实施具有你奠定的基础的自定义布局的主要部分。

## 准备布局

现在所有必需的组件都已经初始化，你可以准备布局。 集合视图首先在布局过程中调用 [prepareLayout](https://developer.apple.com/documentation/uikit/uicollectionviewlayout/1617752-preparelayout) 方法。 在这个例子中，prepareLayout 方法用于实例化集合视图中每个视图的所有布局属性对象，然后将这些属性缓存在 layoutInformation 字典中供以后使用。 有关 prepareLayout 方法的更多信息，请参阅[准备布局](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/CreatingCustomLayouts/CreatingCustomLayouts.html#//apple_ref/doc/uid/TP40012334-CH5-SW19)。

### 创建布局属性

prepareLayout 方法的示例实现分为两部分。 图6-2显示了该方法前半部分的目标。 代码遍历每个 cell，如果该 cell 有子元素，则将这些子元素与父元素相关联。 正如你在图中看到的，这个过程是针对每个 cell 完成的，包括其他父 cell 的子 cell。

图 6-2 连接父索引路径和子索引路径
![](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/Art/layout_process_2x.png)

清单6-4显示了 prepareLayout 方法实现的前半部分。在代码开头初始化的两个可变字典构成了缓存机制的基础。第一个 layoutInformation 是 layoutInformation 属性的本地等价物。创建一个局部可变副本允许实例变量是不可变的，这在自定义布局的实现中是有意义的，因为在 prepareLayout 方法结束运行后布局属性不应该被修改。然后代码按递增顺序遍历每个部分，然后遍历每个部分中的每个项目以为每个单元格创建属性。自定义方法 attributesWithChildrenForIndexPath: 返回自定义布局属性的一个实例，其中子属性填充了当前 index path 中项目的子项的索引路径。然后将属性对象存储在本地 cellInformation 字典中，其 index path 作为关键字。通过对所有项目的初始传递，代码可以在设置项目的框架之前为每个项目设置子项。

清单 6-4 创建布局属性
```
- (void)prepareLayout {
    NSMutableDictionary *layoutInformation = [NSMutableDictionary dictionary];
    NSMutableDictionary *cellInformation = [NSMutableDictionary dictionary];
    NSIndexPath *indexPath;
    NSInteger numSections = [self.collectionView numberOfSections;]
    for(NSInteger section = 0; section < numSections; section++){
        NSInteger numItems = [self.collectionView numberOfItemsInSection:section];
        for(NSInteger item = 0; item < numItems; item++){
            indexPath = [NSIndexPath indexPathForItem:item inSection:section];
            MyCustomAttributes *attributes =
            [self attributesWithChildrenAtIndexPath:indexPath];
            [cellInformation setObject:attributes forKey:indexPath];
        }
    }
    //end of first section
```

### 存储布局属性 

图6-3描述了在 prepareLayout 方法的后半部分发生的过程，其中从最后一行到第一行构建树层次结构。这种方法最初看起来很奇特，但它是消除调整 children cell’s frames 所涉及的复杂性的巧妙方法。因为 children cells 的 frame 需要与他们父母的 frame 相匹配，并且因为 cells 之间的行间距的大小取决于 cell 有多少 children（包括每个 child cell 有多少 children 等等），你想在设置父母的之前设置孩子的 frame。通过这种方式，可以调整 children cell 及其所有 children cell 以匹配其整个父母的 cells。

在第1步中，最后一列的 cells 按顺序排列。在步骤2中，布局确定第二列的帧。在这一列中， cells 可以按顺序排列，因为没有 cell 有多于一个孩子。但是，必须调整绿色 cells 的 frame 以匹配其父 cell 的 frame，因此它会向下移动一个空格。在最后一步中，第一列的 cell 将被放置。第二列的前三个 cell 是第一列中第一个单元格的子元素，因此第一列中第一个 cells 之后的 cells 向下移动。在这种情况下，实际上并不需要这么做，因为两个 cells 之后的 cell 没有自己的孩子，但布局对象不够聪明，无法知道这一点。相反，它总是调整空间，以防跟随有孩子的小孩有自己的孩子。同样，绿色 cell 现在都向下移动，以匹配他们的父母。

图 6-3 组帧过程
![](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/Art/worked_example_2x.png)

清单 6-5 显示了 prepareLayout 方法的后半部分，其中设置了每个 item 的 frame。 在一些代码行之后的注释数字对应于在代码之后找到的编号解释。

清单 6-5 存储布局属性
```
//continuation of prepareLayout implementation
    for(NSInteger section = numSections - 1; section >= 0; section—-){
        NSInteger numItems = [self.collectionView numberOfItemsInSection:section];
        NSInteger totalHeight = 0;
        for(NSInteger item = 0; item < numItems; item++){
            indexPath = [NSIndexPath indexPathForItem:item inSection:section];
            MyCustomAttributes *attributes = [cellInfo objectForKey:indexPath]; // 1
            attributes.frame = [self frameForCellAtIndexPath:indexPath
                                withHeight:totalHeight];
            [self adjustFramesOfChildrenAndConnectorsForClassAtIndexPath:indexPath]; // 2
            cellInfo[indexPath] = attributes;
            totalHeight += [self.customDataSource
                            numRowsForClassAndChildrenAtIndexPath:indexPath]; // 3
        }
        if(section == 0){
            self.maxNumRows = totalHeight; // 4
        }
    }
    [layoutInformation setObject:cellInformation forKey:@"MyCellKind"]; // 5
    self.layoutInformation = layoutInformation
}
```

在清单6-5中，代码按降序遍历这些部分，从后向前构建树。 totalHeight 变量跟踪当前 item 需要的行数。 这个实现并没有巧妙地跟踪间距，只是留下带有子元素的 cells 之下的空白区域，以便两个 cells 的子元素不会重叠，并且 totalHeight 变量可帮助完成此操作。 代码按以下顺序完成此操作：

1、在设置单元格框架之前，从本地字典中检索在第一遍数据中创建的布局属性。

2、自定义的 adjustFramesOfChildrenAndConnectorsForClassAtIndexPath: 方法递归调整所有 cells 的子和孙等的 frame 以匹配 cell 的 frame。

3、将调整后的属性放回字典后，将调整 totalHeight 变量以反映下一个项目帧的位置。这是代码利用自定义协议的地方。无论实现该协议需要实现 numRowsForClassAndChildrenAtIndexPath: 方法的对象，该方法返回每个类需要占用多少行，给定了它有多少个子节点。

4、maxNumRows 属性（稍后需要设置内容大小）设置为节0的总高度。具有最长高度的列始终为第0节，该节的高度已针对树中的所有子级进行了调整，因为此实现不包含智能空间调整。

5、该方法通过将具有所有单元属性的字典插入到具有唯一字符串标识符作为其关键字的本地 layoutInformation 字典中来结束。

用于在最后一步中插入字典的字符串标识符将用于自定义布局的其余部分，以检索 cells 的正确属性。 当补充视图进一步发挥作用时，这变得更加重要。

## 提供内容大小

在准备布局时，代码将 maxNumRows 的值设置为布局中最大部分的行数。 这些信息可以用来设置适当的内容大小，这是布局过程的下一步。 清单6-6显示了 collectionViewContentSize 的实现。 它依赖于常量 ITEM_WIDTH 和 ITEM_HEIGHT，这些常量大概是应用程序的全局参数（例如，它们在自定义单元实现中需要正确调整单元格的大小）。

清单 6-6 调整内容区域的大小
```
- (CGSize)collectionViewContentSize {
    CGFloat width = self.collectionView.numberOfSections * (ITEM_WIDTH + self.insets.left + self.insets.right);
    CGFloat height = self.maxNumRows * (ITEM_HEIGHT + _insets.top + _insets.bottom);
    return CGSizeMake(width, height);
}
```

## 提供布局属性

对所有布局属性对象进行初始化和高速缓存，代码已经完全准备好，以提供 layoutAttributesForElementsInRect: 方法中请求的所有布局信息。此方法是布局过程中的第二步，与 prepareLayout 方法不同，它是必需的。该方法提供了一个矩形，并期望所提供的矩形中包含的任何视图的布局属性对象数组。在某些情况下，包含数千个项目的集合视图可能会等待，直到调用此方法才初始化布局属性对象，以仅为包含在所提供的矩形内的元素，但此实现依赖于缓存。因此，layoutAttributesForElementsInRect: 方法只需要遍历所有存储的属性并将它们收集到一个返回给调用者的数组中。

清单6-7显示了 layoutAttributesForElementsInRect: 方法的实现。该代码遍历所有包含主字典 _layoutInformation 中特定类型视图的布局属性对象的子元素。如果在子字典中检查的属性包含在给定的矩形中，则它们被添加到存储矩形内的所有属性的数组中，所有属性在检查完所有存储的属性后返回。

清单 6-7 收集和处理存储的属性
```
- (NSArray*)layoutAttributesForElementsInRect:(CGRect)rect {
    NSMutableArray *myAttributes [NSMutableArray arrayWithCapacity:self.layoutInformation.count];
    for(NSString *key in self.layoutInformation){
        NSDictionary *attributesDict = [self.layoutInformation objectForKey:key];
        for(NSIndexPath *key in attributesDict){
            UICollectionViewLayoutAttributes *attributes =
            [attributesDict objectForKey:key];
            if(CGRectIntersectsRect(rect, attributes.frame)){
                [attributes addObject:attributes];
            }
        }
    }
    return myAttributes;
}
```

注意：layoutAttributesForElementsInRect 的实现：永远不会引用给定属性的视图是否可见。 请记住，此方法提供的矩形不一定是可见的矩形，不管你的实现是什么，它都不应该假定它返回的属性是可见视图。 有关 layoutAttributesForElementsInRect: 方法的更详细讨论，请参阅[为给定矩形中的项目提供布局属性](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/CreatingCustomLayouts/CreatingCustomLayouts.html#//apple_ref/doc/uid/TP40012334-CH5-SW6)。

## 请求时提供个别属性

正如在按需提供布局属性部分所讨论的那样，布局对象必须准备好在布局过程完成后随时为你的集合视图内任何类型的任何单一项目返回布局属性。 对于所有三种视图都有方法 - cell，补充视图和装饰视图 - 但该应用程序当前仅使用 cells，所以目前需要实现的唯一方法是 layoutAttributesForItemAtIndexPath: 方法。

清单6-8显示了这个方法的实现。 它打开存储的 cells 字典，并在该子字典中返回存储有指定索引路径的属性对象作为其键。

清单 6-8 提供特定项目的属性
```
- (UICollectionViewLayoutAttributes *)layoutAttributesForItemAtIndexPath:(NSIndexPath *)indexPath {
    return self.layoutInfo[@"MyCellKind"][indexPath];
}
```

图6-4显示了代码中此时布局的样子。 所有的 cells 都被正确放置和调整以匹配他们的父母，但是连接它们的线尚未被绘制。

图 6-4 目前的布局
![](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/Art/example_mid_screen_2x.png)

## 纳入补充视图

在当前状态下，应用程序以分层的方式正确显示其所有 cells，但由于没有将父级连接到子级的行，所以类图很难解释。要绘制将 cells 连接到子元素的线条，此应用程序实现依赖于可作为补充视图并入布局的自定义视图。有关设计补充视图的更多信息，请参阅通过补充视图提升内容。

清单6-9显示了可以合并到 prepareLayout 实现中以包含补充视图的代码行。为 cells 和补充视图创建属性对象之间的细微区别在于补充视图的方法需要一个字符串标识符来告诉属性对象适用于哪种补充视图。这是因为自定义布局可以有多种不同类型的补充视图，而每个布局只能有一种类型的单元格。

清单 6-9 为补充视图创建属性对象
```
// create another dictionary to specifically house the attributes for the supplementary view
NSMutableDictionary *supplementaryInfo = [NSMutableDictionary dictionary];
…
// within the initial pass over the data, create a set of attributes for the supplementary views as well
UICollectionViewLayoutAttributes *supplementaryAttributes = [UICollectionViewLayoutAttributes layoutAttributesForSupplementaryViewOfKind:@"ConnectionViewKind" withIndexPath:indexPath];
[supplementaryInfo setObject: supplementaryAttributes forKey:indexPath];
…
// in the second pass over the data, set the frame for the supplementary views just as you did for the cells
UICollectionViewLayoutAttributes *supplementaryAttributes = [supplementaryInfo objectForKey:indexPath];
supplementaryAttributes.frame = [self frameForSupplementaryViewOfKind:@"ConnectionViewKind" AtIndexPath:indexPath];
[supplementaryInfo setObject:supplementaryAttributes ForKey:indexPath];
...
// before setting the instance version of _layoutInformation, insert the local supplementaryInfo dictionary into the local layoutInformation dictionary
[layoutInformation setObject:supplementaryInfo forKey:@"ConnectionViewKind"];
```

由于补充视图的代码与 cells 的代码类似，因此将此代码并入 prepareLayout 方法很简单。该代码对补充视图使用相同的缓存机制，因为它使用另一个字典专门用于 ConnectionViewKind 补充视图。如果你要添加多种补充视图，则需要为该视图创建另一个字典，并为这种视图添加以下代码行。但在这种情况下，布局只需要一种补充视图。与初始化 cells 布局属性的代码一样，此实现使用自定义的 frameForSupplementaryViewOfKind:AtIndexPath: 方法，根据它是哪种视图来确定补充视图的框架。请记住，prepareLayout 方法的实现中显示的自定义 adjustFramesOfChildrenAndConnectorsForClassAtIndexPath: 需要包含与类层次布局相关的任何补充视图的调整。

在示例代码的情况下，layoutAttributesForElementsInRect: 实现中无需修改任何内容，因为它旨在循环存储在主字典中的所有属性。只要将补充视图属性添加到主字典中，则提供的 layoutAttributesForElementsInRect: 实现按预期工作。

最后，与 cells 的情况一样，集合视图可以随时请求特定视图的补充视图属性。因此，layoutAttributesForSupplementaryElementOfKind:atIndexPath: 的实现是必需的。

清单6-10显示了该方法的实现，它与 layoutAttributesForItemAtIndexPath 几乎相同。作为一个例外，使用提供的种类字符串而不是将一种类型的视图硬编码到返回值中允许你在自定义布局中使用多个补充视图。

清单 6-10 按需提供补充视图属性
```
- (UICollectionViewLayoutAttributes *) layoutAttributesForSupplementaryViewOfKind:(NSString *)kind atIndexPath:(NSIndexPath *)indexPath {
    return self.layoutInfo[kind][indexPath];
}
```

## 概括

通过包含补充视图，你现在可以有一个布局对象，可以充分地再现类分层结构图。 在最终实施中，你可能希望将调整结合到自定义布局中以节省空间。 本示例探讨了自定义集合视图布局的实际基础实现可能如何。 集合视图非常强大，并提供比此处更多的功能。 在移动，插入或删除时突出显示和选择（甚至动画）cells 都是简单的增强功能，可以合并到你的应用程序中。 要将你的自定义布局提升到下一个级别，请查看[创建自定义布局](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/CreatingCustomLayouts/CreatingCustomLayouts.html#//apple_ref/doc/uid/TP40012334-CH5-SW1)的最后几个部分。

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~