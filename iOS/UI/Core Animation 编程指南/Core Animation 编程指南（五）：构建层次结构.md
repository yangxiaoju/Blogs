# 构建层次结构

大多数情况下，在你的应用中使用图层的最佳方式是将它们与视图对象结合使用。 但是，有时你可能需要通过向其添加附加图层对象来增强视图层次结构。 这样做时可以使用使用图层提供更好的性能，或者让你实现仅凭视图难以处理的功能。 在这些情况下，你需要知道如何管理你创建的图层层次结构。

> 重要提示：在 OS X v10.8 及更高版本中，建议你尽量减少对层次结构的使用，并使用层次支持的视图。 在该版本的 OS X 中引入的图层重绘策略允许你自定义层支持的视图的行为，并仍然获得以前使用独立层可能获得的性能。

## 将图层排列成图层层次结构

层次结构在许多方面与查看层次结构相似。 将一个图层嵌入另一个图层中，以在要嵌入的图层（称为子图层）和父图层（称为超级图层）之间创建父子关系。 这种亲子关系会影响子图层的各个方面。 例如，其内容位于父级内容的上方，其位置是相对于其父级的坐标系指定的，并且受到应用于父级的任何变换的影响。

### 添加，插入和删除子图层

每个图层对象都有添加，插入和删除子图层的方法。 表4-1总结了这些方法及其行为。

表 4-1 修改图层分层结构的方法 
| Behavior | Methods | Description |
| - | - | - |
| 添加图层 | [addSublayer:](https://developer.apple.com/documentation/quartzcore/calayer/1410833-addsublayer) | 将新的子图层对象添加到当前图层。 子图层被添加到图层的子图层列表的末尾。 这会导致子图层出现在其 zPosition 属性中具有相同值的任何兄弟的顶部。 |
| 插入图层 | [insertSublayer:above:](https://developer.apple.com/documentation/quartzcore/calayer/1410798-insertsublayer) [insertSublayer:atIndex:](https://developer.apple.com/documentation/quartzcore/calayer/1410944-insertsublayer) [insertSublayer:below:](https://developer.apple.com/documentation/quartzcore/calayer/1410840-insertsublayer) | 将子图层插入指定索引处的子图层或与其他子图层相关的位置。 当插入到另一个子图层的上方或下方时，只需指定子图层数组中的子图层的位置。 层的实际可见性主要由它们的zPosition属性中的值决定，其次由它们在子层数组中的位置决定。 |
| 删除图层 | [removeFromSuperlayer](https://developer.apple.com/documentation/quartzcore/calayer/1410767-removefromsuperlayer) | 从其父层删除子图层。 |
| 交换图层 | [replaceSublayer:with:]()  | 为另一个交换一个子图层。 如果你要插入的子图层已经在另一个图层层次结构中，则首先从该层次结构中删除它。 |

在使用自己创建的图层对象时，可以使用上述方法。 你不会使用这些方法来排列属于层次支持视图的图层。 但是，层次支持的视图可以作为你自己创建的独立层的父层。

### 定位和调整子图层的大小

添加和插入子图层时，必须在子图层出现在屏幕上之前设置子图层的大小和位置。 你可以在将子图层添加到图层层次结构后修改其大小和位置，但应该习惯于在创建图层时设置这些值。

你可以使用 [bounds](https://developer.apple.com/documentation/quartzcore/calayer/1410915-bounds) 属性设置子图层的大小，并使用 [position](https://developer.apple.com/documentation/quartzcore/calayer/1410791-position) 属性在其父图层中设置其位置。 边界矩形的原点几乎总是（0，0），并且大小与点中指定的图层的大小无关。 位置属性中的值相对于图层的锚点进行解释，该锚点默认位于图层的中心。 如果不为这些属性赋值，Core Animation 会将图层的初始宽度和高度设置为0，并将位置设置为（0，0）。

```
myLayer.bounds = CGRectMake(0, 0, 100, 100);
myLayer.position = CGPointMake(200, 200);
```

重要提示：始终对图层的宽度和高度使用整数。

### 图层层次结构如何影响动画

某些父图层属性可能会影响应用于其子图层的任何动画的行为。其中一个属性是速度属性，它是动画速度的乘数。该属性的值默认设置为1.0，但将其更改为2.0会使动画以原始速度的两倍运行，从而完成一半时间。该属性不仅影响其设置的图层，还影响该图层的子图层。这种变化也是可乘的。如果子图层和其父图层的速度均为2.0，则子图层上的动画将以其原始速度的四倍运行。

大多数其他层更改以可预测的方式影响任何包含的子层。例如，将旋转变换应用到图层会旋转该图层及其所有子图层。同样，更改图层的不透明度会更改其子图层的不透明度。对[图层大小的更改遵循调整图层层次结构的布局中](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/BuildingaLayerHierarchy/BuildingaLayerHierarchy.html#//apple_ref/doc/uid/TP40004514-CH6-SW7)所述的布局规则。

## 调整图层层次结构的布局

Core Animation 支持多种选项来调整子层的大小和位置以响应其父层次的变化。 在 iOS 中，普遍使用层次支持的视图使得层次层次的创建不那么重要; 只支持手动布局更新。 对于 OS X，可以使用其他几个选项来更容易地管理层次结构。

只有在使用你创建的独立图层对象构建图层分层结构时，图层层级布局才相关。 如果你应用的图层都与视图相关联，则使用基于视图的布局支持来更新视图的大小和位置以响应更改。

### 使用约束来管理 OS X 中的图层层次结构

约束使你可以使用图层及其父图层或同层图层之间的一组详细关系指定图层的位置和大小。定义约束需要以下步骤：

- 创建一个或多个 [CAConstraints](https://developer.apple.com/documentation/quartzcore/caconstraint) 对象。使用这些对象来定义约束参数。
- 将你的约束对象添加到其修改属性的图层。
- 检索共享的 [CAConstraintLayoutManager](https://developer.apple.com/documentation/quartzcore/caconstraintlayoutmanager) 对象并分配给直接父图层。

图4-1显示了可用于定义约束的属性以及它们影响的图层的方面。你可以使用约束根据相对于其他图层的中点边缘位置来更改图层的位置。你也可以使用它们来更改图层的大小。你所做的更改可以与父图层或与其他图层相关的比例成比例。你甚至可以为所产生的更改添加比例因子或常数。这种额外的灵活性可以使用一套简单的规则非常精确地控制图层的大小和位置。

图 4-1 约束布局管理器属性
![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/ca_constraint_2x.png)

每个约束对象封装沿同一轴的两个层之间的几何关系。 每个轴最多可以分配两个约束对象，而这两个约束可以决定哪个属性可以更改。 例如，如果你为图层的左侧和右侧边缘指定约束，则图层的大小会发生变化。 如果你为图层的左边缘和宽度指定约束，则图层右边缘的位置会更改。 如果你为其中一个图层的边指定单个约束，则 Core Animation 会创建一个隐式约束，以便将该图层的大小保持在给定维中。

创建约束时，你必须始终指定三条信息：

- 你想要约束的图层的方面
- 要用作参考的图层
- 用于比较的参考图层的方面

程序清单 4-1 显示了一个简单的约束，它将一个图层的垂直中点固定到其父图层的垂直中点。 引用父图层时，请使用字符串父图层。 这个字符串是保留用于引用父层的特殊名称。 使用它可以避免需要指向图层的指针或知道图层的名称。 它还允许您更改父图层并将约束自动应用于新父级。 （创建与兄弟图层相关的约束时，必须使用其名称属性标识兄弟图层。）

清单 4-1 定义一个简单的约束
```
[myLayer addConstraint:[CAConstraint constraintWithAttribute:kCAConstraintMidY
                                                 relativeTo:@"superlayer"
                                                  attribute:kCAConstraintMidY]];
```

要在运行时应用约束，你必须将共享的 CAConstraintLayoutManager 对象附加到直接父图层。每层负责管理其子层的布局。将布局管理器分配给父级会告诉 Core Animation 应用其子级定义的约束。布局管理器对象自动应用约束。将它分配给父层后，你不必告诉它更新布局。

要查看更具体情况下的约束如何工作，请考虑图4-2。在这个例子中，设计要求 layerA 的宽度和高度保持不变，并且 layerA 保持居中在它的父图层中。另外，层B的宽度必须与层A的宽度相同，层B的顶部边缘必须保持在层A的底部边缘以下10个点，并且层B的底部边缘必须保持超过层的底部边缘10个点。清单 4-2 显示了你将用于为此示例创建子层和约束的代码。

图 4-2 基于约束的布局示例
![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/constraintsManagerExample_2x.png)

清单 4-2 为你的图层设置约束
```
// Create and set a constraint layout manager for the parent layer.
theLayer.layoutManager=[CAConstraintLayoutManager layoutManager];
 
// Create the first sublayer.
CALayer *layerA = [CALayer layer];
layerA.name = @"layerA";
layerA.bounds = CGRectMake(0.0,0.0,100.0,25.0);
layerA.borderWidth = 2.0;
 
// Keep layerA centered by pinning its midpoint to its parent's midpoint.
[layerA addConstraint:[CAConstraint constraintWithAttribute:kCAConstraintMidY
                                                 relativeTo:@"superlayer"
                                                  attribute:kCAConstraintMidY]];
[layerA addConstraint:[CAConstraint constraintWithAttribute:kCAConstraintMidX
                                                 relativeTo:@"superlayer"
                                                  attribute:kCAConstraintMidX]];
[theLayer addSublayer:layerA];
 
// Create the second sublayer
CALayer *layerB = [CALayer layer];
layerB.name = @"layerB";
layerB.borderWidth = 2.0;
 
// Make the width of layerB match the width of layerA.
[layerB addConstraint:[CAConstraint constraintWithAttribute:kCAConstraintWidth
                                                 relativeTo:@"layerA"
                                                  attribute:kCAConstraintWidth]];
 
// Make the horizontal midpoint of layerB match that of layerA
[layerB addConstraint:[CAConstraint constraintWithAttribute:kCAConstraintMidX
                                                 relativeTo:@"layerA"
                                                  attribute:kCAConstraintMidX]];
 
// Position the top edge of layerB 10 points from the bottom edge of layerA.
[layerB addConstraint:[CAConstraint constraintWithAttribute:kCAConstraintMaxY
                                                 relativeTo:@"layerA"
                                                  attribute:kCAConstraintMinY
                                                     offset:-10.0]];
 
// Position the bottom edge of layerB 10 points
//  from the bottom edge of the parent layer.
[layerB addConstraint:[CAConstraint constraintWithAttribute:kCAConstraintMinY
                                                 relativeTo:@"superlayer"
                                                  attribute:kCAConstraintMinY
                                                     offset:+10.0]];
 
[theLayer addSublayer:layerB];
```

清单4-2中有一件有趣的事情是代码从不明确地设置 layerB 的大小。 由于已定义约束条件，每次布局更新时都会自动设置 layerB 的宽度和高度。 因此，使用 bounds rectangle 设置大小是不必要的。

警告：创建约束时，请勿在约束中创建循环引用。 循环约束使得不可能计算出所需的布局信息。 遇到此类循环引用时，布局行为未定义。

### 为你的 OS X 图层分层结构设置自动调整规则

自动调整规则是调整 OS X 中图层的大小和位置的另一种方法。使用自动调整规则，你可以指定图层的边缘是否应与父图层的相应边缘保持固定或可变的距离。你可以同样指定图层的宽度或高度是固定的还是可变的。关系总是在图层和它的父图层之间。你不能使用自动调整规则来指定兄弟图层之间的关系。

要为图层设置自动调整规则，你必须将适当的常量分配给图层的 [autoresizingMask](https://developer.apple.com/documentation/quartzcore/calayer/1410877-autoresizingmask) 属性。默认情况下，图层被配置为具有固定的宽度和高度。在布局过程中，Core Animation 会自动为你计算图层的精确大小和位置，并且会涉及基于许多因素的复杂计算集。 Core Animation 在请求代理执行任何手动布局更新之前应用自动调整行为，因此你可以根据需要使用委托调整自动调整布局的结果。

### 手动布局你的图层层次结构

在 iOS 和 OS X 上，你可以通过在父图层的委托对象上实现 [layoutSublayersOfLayer:](https://developer.apple.com/reference/objectivec/nsobject/1410941) 方法来手动处理布局。 你可以使用该方法调整当前嵌入到图层中的任何子图层的大小和位置。 在进行手动布局更新时，你需要执行必要的计算来定位每个子图层。

如果你正在实现自定义图层子类，那么你的子类可以覆盖 [layoutSublayers](https://developer.apple.com/documentation/quartzcore/calayer/1410935-layoutsublayers) 方法并使用该方法（而不是委托）来处理任何布局任务。 如果需要完全控制自定义图层类中子图层的位置，则应该只覆盖此方法。 替换默认实现可防止 Core Animation 在 OS X 上应用约束或自动调整规则。

## 子层和剪切

与视图不同，父图层不会自动剪裁位于边框矩形外的子图层的内容。 相反，父图层允许其子图层默认全部显示。 但是，可以通过将图层的 masksToBounds 属性设置为 YES 来重新启用裁剪。

如果指定了图层的剪切蒙版，则其形状包含图层的角半径。 图4-3显示了一个图层，演示了 masksToBounds 属性如何影响具有圆角的图层。 当属性设置为 NO 时，即使子层延伸到其父层的边界之外，子层也会全部显示。 将该属性更改为 YES 会导致其内容被裁剪。

图 4-3 将子图层裁剪到父级边界
![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/clipping_2x.png)

## 转换图层之间的坐标值

有时，你可能需要将一个图层中的坐标值转换为不同图层中同一屏幕位置处的坐标值。 CALayer 类提供了一组可以用于此目的的简单转换例程：

- [convertPoint:fromLayer:](https://developer.apple.com/documentation/quartzcore/calayer/1410825-convert)
- [convertPoint:toLayer:](https://developer.apple.com/documentation/quartzcore/calayer/1410881-convertpoint)
- [convertRect:fromLayer:](https://developer.apple.com/documentation/quartzcore/calayer/1410948-convertrect)
- [convertRect:toLayer:](https://developer.apple.com/documentation/quartzcore/calayer/1410742-convertrect)

除了转换点和矩形值之外，还可以使用 convertTime:fromLayer: 和 convertTime:toLayer: 方法在图层之间转换时间值。 每个图层都定义了自己的本地时间空间，并使用该时间空间将动画的开始和结束与系统的其余部分同步。 这些时间空间默认同步; 但是，如果更改一组图层的动画速度，那么这些图层的时间空间会相应更改。 你可以使用时间转换方法来解决任何此类因素，并确保两个图层的时间同步。

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~