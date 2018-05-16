# 纳入手势支持

你可以通过使用手势识别器为你的集合视图添加更大的交互性。 将手势识别器添加到集合视图中，并在发生这些手势时使用它来触发操作。 对于集合视图，你可能想要实现两种类型的操作：

- 你想要触发集合视图布局信息的更改。
- 你想直接操纵 cells 和视图。

你应该始终将你的手势识别器附加到集合视图本身 - 而不是特定的 cell 或视图。 UICollectionView 类是 UIScrollView 的子类，因此将你的手势识别器附加到集合视图不太可能干扰必须跟踪的其他手势。 另外，由于集合视图可以访问数据源和布局对象，因此你仍然可以访问所需的所有信息，以适当地操作 cells 和视图。

## 使用手势识别器修改布局信息

手势识别器提供了一种动态修改布局参数的简单方法。例如，你可以使用捏合手势识别器来更改自定义布局中项目之间的间距。配置这种手势识别器的过程相对简单。

1、创建手势识别器。

2、将手势识别器附加到集合视图。

3、使用手势识别器的处理程序方法更新布局参数并使布局对象失效。

你使用与所有对象相同的 alloc/init 过程创建手势识别器。在初始化期间，你指定手势触发时要调用的目标对象和操作方法。然后调用集合视图的 [addGestureRecognizer:](https://developer.apple.com/documentation/uikit/uiview/1622496-addgesturerecognizer) 方法将其附加到视图。大部分实际工作发生在你在初始化时指定的操作方法中。

清单4-1显示了一个由连接到集合视图的捏手势识别器调用的动作方法的示例。在此示例中，pinch 数据用于更改自定义布局中 cells 之间的距离。布局对象实现自定义 updateSpreadDistance 方法，该方法验证新的距离值并将其存储起来，以便以后在布局过程中使用。操作方法会使布局无效，并强制它根据新值更新 items 的位置。

清单 4-1 使用手势识别器来更改布局值
```
- (void)handlePinchGesture:(UIPinchGestureRecognizer *)sender {
    if ([sender numberOfTouches] != 2)
        return;
 
   // Get the pinch points.
   CGPoint p1 = [sender locationOfTouch:0 inView:[self collectionView]];
   CGPoint p2 = [sender locationOfTouch:1 inView:[self collectionView]];
 
   // Compute the new spread distance.
    CGFloat xd = p1.x - p2.x;
    CGFloat yd = p1.y - p2.y;
    CGFloat distance = sqrt(xd*xd + yd*yd);
 
   // Update the custom layout parameter and invalidate.
   MyCustomLayout* myLayout = (MyCustomLayout*)[[self collectionView] collectionViewLayout];
   [myLayout updateSpreadDistance:distance];
   [myLayout invalidateLayout];
}
```

有关创建手势识别器并将其附加到视图的更多信息，请参阅 iOS 事件处理指南。

## 使用默认手势行为

UICollectionView 类监听单击以启动其突出显示和选择的委托方法。如果要将自定义轻击或长按手势添加到集合视图，请将手势识别器的值配置为与集合视图已使用的值不同。例如，你可以配置轻击手势识别器以仅响应双击。

清单4-2显示了如何让集合视图响应你的手势，而不是监听 cell 选择/高亮显示。由于集合视图不使用手势识别器来启动其委托方法，因此你的自定义手势识别器优先于默认选择监听器，方法是将手势识别器的 delaysTouchesBegan 属性设置为 YES 或取消触摸事件，从而延迟其他触摸事件的注册通过将你的手势识别器的 cancelsTouchesInView 属性设置为 YES。每次 tap 注册时，都会首先检查你的手势识别器是否应具有优先权。如果输入对你的手势识别器无效，则委托方法将按正常方式调用。

清单 4-2 优先考虑你的手势识别器
```
UITapGestureRecognizer* tapGesture = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(handleTapGesture:)];
tapGesture.delaysTouchesBegan = YES;
tapGesture.numberOfTapsRequired = 2;
[self.collectionView addGestureRecognizer:tapGesture];
```

## 操纵单元格和视图

如何使用手势识别器来操作 cells 和视图取决于你计划制作的操作类型。可以在标准手势识别器的操作方法内执行简单的插入和删除操作。但是，如果你计划更复杂的操作，则可能需要定义一个自定义手势识别器来自己跟踪触摸事件。

一种类型的操作需要自定义手势识别器将集合视图中的 cell 从一个位置移动到另一个位置。移动 cell 最直接的方法是从集合视图中（暂时）删除它，使用手势识别器拖动该单元格的视觉表示，然后在触摸事件结束时将 cell 插入新位置。所有这些都需要自己管理触摸事件，与布局对象紧密合作以确定新的插入位置，操纵数据源更改，然后将该项目插入新位置。

有关创建自定义手势识别器的更多信息，请参阅 iOS 事件处理指南。

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~