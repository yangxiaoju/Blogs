# 动画属性

CALayer 和 CIFilter 中的许多属性都可以生成动画。 本附录列出了这些属性以及默认使用的动画。

## CALayer 动画属性

表B-1列出了你可能考虑动画的 CALayer 类的属性。 对于每个属性，该表还列出了为执行隐式动画而创建的默认动画对象的类型。

表 B-1 图层属性及其默认动画

| Property | Default animation |
| - | - |
| [anchorPoint](https://developer.apple.com/documentation/quartzcore/calayer/1410817-anchorpoint) | 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 [CABasicAnimation](https://developer.apple.com/documentation/quartzcore/cabasicanimation) 对象。 |
| [backgroundColor](https://developer.apple.com/documentation/quartzcore/calayer/1410966-backgroundcolor) | 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 [CABasicAnimation](https://developer.apple.com/documentation/quartzcore/cabasicanimation) 对象。 |
| [backgroundFilters](https://developer.apple.com/documentation/quartzcore/calayer/1410827-backgroundfilters) | 使用默认的默认 [CATransition](https://developer.apple.com/documentation/quartzcore/catransition) 对象，如[表B-3](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW3)所述。 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 CABasicAnimation 对象为过滤器的子属性设置动画。 |
| [borderColor](https://developer.apple.com/documentation/quartzcore/calayer/1410903-bordercolor) | 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 [CABasicAnimation](https://developer.apple.com/documentation/quartzcore/cabasicanimation) 对象。 |
| [borderWidth](https://developer.apple.com/documentation/quartzcore/calayer/1410917-borderwidth) | 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 [CABasicAnimation](https://developer.apple.com/documentation/quartzcore/cabasicanimation) 对象。 |
| [bounds](https://developer.apple.com/documentation/quartzcore/calayer/1410915-bounds) | 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 [CABasicAnimation](https://developer.apple.com/documentation/quartzcore/cabasicanimation) 对象。 |
| [compositingFilter](https://developer.apple.com/documentation/quartzcore/calayer/1410748-compositingfilter) | 使用默认的默认 [CATransition](https://developer.apple.com/documentation/quartzcore/catransition) 对象，如[表B-3](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW3)所述。 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 CABasicAnimation 对象为过滤器的子属性设置动画。 |
| [contents](https://developer.apple.com/documentation/quartzcore/calayer/1410773-contents) | 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 [CABasicAnimation](https://developer.apple.com/documentation/quartzcore/cabasicanimation) 对象。 |
| [contentsRect](https://developer.apple.com/documentation/quartzcore/calayer/1410866-contentsrect) | 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 [CABasicAnimation](https://developer.apple.com/documentation/quartzcore/cabasicanimation) 对象。 |
| [cornerRadius](https://developer.apple.com/documentation/quartzcore/calayer/1410818-cornerradius) | 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 [CABasicAnimation](https://developer.apple.com/documentation/quartzcore/cabasicanimation) 对象。 |
| [doubleSided](https://developer.apple.com/documentation/quartzcore/calayer/1410924-doublesided) | 没有默认的隐式动画。 |
| [filters](https://developer.apple.com/documentation/quartzcore/calayer/1410901-filters) | 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 [CABasicAnimation](https://developer.apple.com/documentation/quartzcore/cabasicanimation) 对象。 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 CABasicAnimation 对象为过滤器的子属性设置动画。 |
| [frame](https://developer.apple.com/documentation/quartzcore/calayer/1410779-frame) | 这个属性不是可以动画的。 通过动画 [bounds](https://developer.apple.com/documentation/quartzcore/calayer/1410915-bounds) 和 [position]() 属性可以获得相同的结果。 |
| [hidden](https://developer.apple.com/documentation/quartzcore/calayer/1410838-hidden) | 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 [CABasicAnimation](https://developer.apple.com/documentation/quartzcore/cabasicanimation) 对象。 |
| [mask](https://developer.apple.com/documentation/quartzcore/calayer/1410861-mask) | 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 [CABasicAnimation](https://developer.apple.com/documentation/quartzcore/cabasicanimation) 对象。 |
| [maskToBounds](https://developer.apple.com/documentation/quartzcore/calayer/1410896-maskstobounds) | 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 [CABasicAnimation](https://developer.apple.com/documentation/quartzcore/cabasicanimation) 对象。 |
| [opacity](https://developer.apple.com/documentation/quartzcore/calayer/1410933-opacity) | 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 [CABasicAnimation](https://developer.apple.com/documentation/quartzcore/cabasicanimation) 对象。 |
| [position](https://developer.apple.com/documentation/quartzcore/calayer/1410791-position) | 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 [CABasicAnimation](https://developer.apple.com/documentation/quartzcore/cabasicanimation) 对象。 |
| [shadowColor](https://developer.apple.com/documentation/quartzcore/calayer/1410829-shadowcolor) | 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 [CABasicAnimation](https://developer.apple.com/documentation/quartzcore/cabasicanimation) 对象。 |
| [shadowOffset](https://developer.apple.com/documentation/quartzcore/calayer/1410970-shadowoffset) | 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 [CABasicAnimation](https://developer.apple.com/documentation/quartzcore/cabasicanimation) 对象。 |
| [shadowOpacity](https://developer.apple.com/documentation/quartzcore/calayer/1410751-shadowopacity) | 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 [CABasicAnimation](https://developer.apple.com/documentation/quartzcore/cabasicanimation) 对象。 |
| [shadowPath](https://developer.apple.com/documentation/quartzcore/calayer/1410771-shadowpath) | 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 [CABasicAnimation](https://developer.apple.com/documentation/quartzcore/cabasicanimation) 对象。 |
| [shadowRadius](https://developer.apple.com/documentation/quartzcore/calayer/1410819-shadowradius) | 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 [CABasicAnimation](https://developer.apple.com/documentation/quartzcore/cabasicanimation) 对象。 |
| [sublayers](https://developer.apple.com/documentation/quartzcore/calayer/1410802-sublayers) | 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 [CABasicAnimation](https://developer.apple.com/documentation/quartzcore/cabasicanimation) 对象。 |
| [sublayerTransform](https://developer.apple.com/documentation/quartzcore/calayer/1410888-sublayertransform) | 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 [CABasicAnimation](https://developer.apple.com/documentation/quartzcore/cabasicanimation) 对象。 |
| [transform](https://developer.apple.com/documentation/quartzcore/calayer/1410836-transform) | 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 [CABasicAnimation](https://developer.apple.com/documentation/quartzcore/cabasicanimation) 对象。 |
| [zPosition](https://developer.apple.com/documentation/quartzcore/calayer/1410884-zposition) | 使用[表B-2](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW2)中所述的默认隐含 [CABasicAnimation](https://developer.apple.com/documentation/quartzcore/cabasicanimation) 对象。 |

表B-2列出了默认的基于属性的动画的动画属性。

表 B-2 默认隐含的基本动画

| Description | Value |
| - | - |
| Class | [CABasicAnimation](https://developer.apple.com/documentation/quartzcore/cabasicanimation) |
| Duration | 0.25秒，或当前 transaction 的持续时间 |
| Key path | 设置为图层的属性名称。 |

表B-3列出了默认基于过渡动画的动画对象配置。

表 B-3 默认隐含转换

| Description | Value |
| - | - |
| Class | [CATransition](https://developer.apple.com/documentation/quartzcore/catransition) |
| Duration | 0.25秒，或当前 transaction 的持续时间 |
| Type | Fade (kCATransitionFade) |
| Start progress | 0.0 |
| End progress | 1.0 |

## CIFilter 动画特性

Core Animation 为 Core Image 的 [CIFilter](https://developer.apple.com/documentation/coreimage/cifilter) 类添加了以下动画属性。 这些属性仅在 OS X 上可用。

- [name](https://developer.apple.com/documentation/coreimage/cifilter/1437997-setname)
- [enabled](https://developer.apple.com/documentation/coreimage/cifilter/1438276-enabled)

有关这些补充的更多信息，请参阅 CIFilter Core Animation Additions。

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~