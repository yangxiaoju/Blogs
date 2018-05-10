# 更改图层的默认行为

Core Animation 使用动作对象为图层实现其隐式动画行为。 动作对象是符合 CAAction 协议并定义要在图层上执行的一些相关行为的对象。 所有 CAAnimation 对象都实现该协议，并且这些对象通常分配为在图层属性更改时执行。

动画属性是一种动作类型，但你可以使用几乎任何你想要的动作来定义动作。 但是，要做到这一点，你必须定义你的操作对象并将它们与应用程序的图层对象关联。

## 自定义操作对象遵守 CAAction 协议

要创建你自己的操作对象，请从你的某个类遵守 CAAction 协议并实现 [runActionForKey:object:arguments:](https://developer.apple.com/documentation/quartzcore/caaction/1410806-run) 方法。在该方法中，使用可用的信息执行你想要在图层上执行的任何操作。你可以使用该方法将动画对象添加到图层，或者可以使用它来执行其他任务。

当你定义一个动作对象时，你必须决定你想如何触发该动作。动作的触发器定义了你稍后用于注册该动作的密钥。操作对象可以由以下任何情况触发：

- 其中一个图层属性的值已更改。这可以是图层的任何属性，而不仅仅是可动画的属性。 （你还可以将操作与添加到图层的自定义属性相关联。）标识此操作的键是该属性的名称。
- 图层变得可见或被添加到图层层次结构中。识别此操作的关键是 [kCAOnOrderIn](https://developer.apple.com/documentation/quartzcore/kcaonorderin)。
- 图层已从图层分层结构中移除。识别此操作的关键是 [kCAOnOrderOut](https://developer.apple.com/documentation/quartzcore/kcaonorderout)。
- 该层即将参与过渡动画。识别这个动作的关键是 [kCATransition](https://developer.apple.com/documentation/quartzcore/kcatransition)。

## 动作对象必须安装在图层上以产生效果

在可以执行动作之前，该层需要找到要执行的相应动作对象。 与层相关的操作的关键是要修改的属性的名称或标识操作的特殊字符串。 当在图层上发生适当的事件时，图层会调用其 [actionForKey:](https://developer.apple.com/documentation/quartzcore/calayer/1410844-actionforkey) 方法来搜索与该关键字关联的操作对象。 在此搜索过程中，你的应用可以在多个位置插入自身，并为该密钥提供相关操作对象。

Core Animation 按以下顺序查找操作对象：

1、如果图层具有委托并且该委托实现了 [actionForLayer:forKey:](https://developer.apple.com/documentation/quartzcore/calayerdelegate/2097264-action) 方法，那么图层将调用该方法。 代表必须执行以下任一操作：
- 返回给定键的操作对象。
- 如果它不处理动作，则返回 nil，在这种情况下继续搜索。
- 返回 [NSNull](https://developer.apple.com/documentation/foundation/nsnull) 对象，在这种情况下，搜索立即结束。

2、该图层在层的 [actions](https://developer.apple.com/documentation/quartzcore/calayer/1410789-actions) 字典中查找给定的键。

3、该图层在 [style](https://developer.apple.com/documentation/quartzcore/calayer/1410875-style) 字典中查找包含该键的动作字典。 （换句话说，样式字典包含一个动作键，其值也是一个字典，该层在第二个字典中查找给定的键。）

4、该图层调用其 [defaultActionForKey:](https://developer.apple.com/documentation/quartzcore/calayer/1410954-defaultaction) 类方法。

5、该层执行由 Core Animation 定义的隐式操作（如果有的话）。

如果你在任何适当的搜索点提供了一个操作对象，该层会停止搜索并执行返回的操作对象。当它找到一个动作对象时，该层会调用该对象的 runActionForKey:object:arguments: 方法来执行动作。如果为给定键定义的动作已经是 CAAnimation 类的实例，则可以使用该方法的默认实现来执行动画。如果你正在定义符合 CAAction 协议的自定义对象，则必须使用对象的该方法实现来执行适当的操作。

你在哪里安装你的动作对象取决于你打算如何修改图层。

- 对于只适用于特定情况的操作，或者已经使用委托对象的图层，请提供委托并实现其 actionForLayer:forKey: 方法。
- 对于通常不使用委托的图层对象，将动作添加到图层的动作字典中。
- 对于与你在图层对象上定义的自定义属性相关的操作，请将操作包括在图层的样式字典中。
- 对于图层行为的基本操作，对图层进行子类化并覆盖 defaultActionForKey: 方法。

清单 6-1 显示了用于提供操作对象的委托方法的实现。在这种情况下，委托会查找图层内容属性的更改，并使用过渡动画将新内容交换到位。

清单 6-1 使用图层委托对象提供动作
```
- (id<CAAction>)actionForLayer:(CALayer *)theLayer
                        forKey:(NSString *)theKey {
    CATransition *theAnimation=nil;
 
    if ([theKey isEqualToString:@"contents"]) {
 
        theAnimation = [[CATransition alloc] init];
        theAnimation.duration = 1.0;
        theAnimation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseIn];
        theAnimation.type = kCATransitionPush;
        theAnimation.subtype = kCATransitionFromRight;
    }
    return theAnimation;
}
```

## 暂时禁用使用 CATransaction 类的操作

你可以使用 [CATransaction](https://developer.apple.com/documentation/quartzcore/catransaction) 类临时禁用图层操作。 当你更改图层的属性时，Core Animation 通常会创建一个隐式事务对象来为更改设置动画。 如果你不想动画更改，则可以通过创建显式事务并将其 [kCATransactionDisableActions](https://developer.apple.com/documentation/quartzcore/kcatransactiondisableactions) 属性设置为 true 来禁用隐式动画。 清单6-2展示了一个代码片段，它在从图层树中移除指定图层时禁用动画。

清单 6-2 暂时禁用图层的动作
```
[CATransaction begin];
[CATransaction setValue:(id)kCFBooleanTrue
                 forKey:kCATransactionDisableActions];
[aLayer removeFromSuperlayer];
[CATransaction commit];
```

有关使用事务对象管理动画行为的更多信息，请参阅[显式事务让您更改动画参数](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AdvancedAnimationTricks/AdvancedAnimationTricks.html#//apple_ref/doc/uid/TP40004514-CH8-SW3)。

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~