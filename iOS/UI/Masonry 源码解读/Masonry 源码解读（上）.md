# Masonry 源码解读（上）

## 前言

iOS 开发中的布局方式，总体而言经过了三个时代。混沌初开之时，世间只有3.5英寸（iPhone 4、iPhone 4S），那个时候屏幕适配对于大多数 iOS 开发者来说并不是什么难题，用 frame 就能精确高效的定位。这之后，苹果发布了4英寸机型（iPhone 5、iPhone 5C、iPhone 5S），与此同时苹果也推出了 AutoresizingMask，用来协调子视图与父视图之间的关系。再之后，各种各样的 iPhone 和 iPad 纷纷面世，不仅仅是屏幕尺寸方面的差异，更有异形屏（iPhone X）。在此期间，苹果提出了 AutoLayout 技术，供开发者进行屏幕适配。

使用 `AutoLayout` 的方法也有两种——通过 `Interface Builder` 或者纯代码。前者一直是苹果官方文档里所鼓励的，原因是苹果从最初到现在，对于 iOS 应用的想法都是小而美的，在他们的认知里，一个 APP 应该提供尽可能小的功能集，这也是为为何苹果迄今为止官方推荐的架构仍然是 MVC，官方推荐的开发方式仍是以 `StoryBoard（Size Classes）`。但是在一些项目较大的公司，`StoryBoard` 的某些特性（导致应用包过大，减缓启动速度，合并代码困难）又是不能为人所容忍的，便有了纯代码来实现 `View` 层的一群开发者（比如我）。

如果你曾经用代码来实现 `AutoLayout`，你会发现苹果提供的 `API` 的繁琐程度令人发指，这也是 `Masonry` 这类框架被发明的原因。`Masonry` 是一个轻量级的布局框架，它使用更好的语法来封装 `AutoLayout`。`Masonry` 有自己的布局 `DSL`，它提供了一种链式的方式来描述你的 `NSLayoutConstraints`，从而得到更简洁和可读的布局代码。

接下来，我们就从 `Masonry` 中 `README` 提供的代码着手，看一看 `Masonry` 是如何帮助我们简化繁琐的 `AutoLayout` 代码的。

## mas_makeConstraints:

举一个简单的例子，你想要一个视图填充它的父视图，但是在每一边间隔10个点。

```
UIView *superview = self.view;

UIView *view1 = [[UIView alloc] init];
view1.backgroundColor = [UIColor greenColor];
[superview addSubview:view1];

UIEdgeInsets padding = UIEdgeInsetsMake(10, 10, 10, 10);

[view1 mas_makeConstraints:^(MASConstraintMaker *make) {
    make.top.equalTo(superview.mas_top).with.offset(padding.top); //with is an optional semantic filler
    make.left.equalTo(superview.mas_left).with.offset(padding.left);
    make.bottom.equalTo(superview.mas_bottom).with.offset(-padding.bottom);
    make.right.equalTo(superview.mas_right).with.offset(-padding.right);
}];
```

我们想要实现约束的效果，是通过 `mas_makeConstraints:` 这个方法来实现的，这个方法可以在任意 `UIView` 类及其子类上调用，说明其是一个分类方法，这也是这个方法加了 `mas_` 前缀的原因。该方法声明在 `UIView+MASAdditions.h` 文件中，先来看一下这个方法的完整声明：

```
- (NSArray *)mas_makeConstraints:(void(NS_NOESCAPE ^)(MASConstraintMaker *make))block;
```

这个方法传递的参数是一个参数为 `MASConstraintMaker` 类型的无返回值的 `block`，而该方法的返回值则是一个数组。方法声明中我们看到了一个叫做 `NS_NOESCAPE` 的宏，`NS_NOESCAPE` 用于修饰方法中的 `block` 类型参数，作用是告诉编译器，该 `block` 在方法返回之前就会执行完毕，而不是被保存起来在之后的某个时候再执行。编译器被告知后，就会相应的进行一些优化。更详细的内容请参考 [Add @noescape to public library API](https://github.com/apple/swift-evolution/blob/master/proposals/0012-add-noescape-to-public-library-api.md)

接下来是方法的实现：

```
- (NSArray *)mas_makeConstraints:(void(^)(MASConstraintMaker *))block {
    self.translatesAutoresizingMaskIntoConstraints = NO;
    MASConstraintMaker *constraintMaker = [[MASConstraintMaker alloc] initWithView:self];
    block(constraintMaker);
    return [constraintMaker install];
}
```

### self.translatesAutoresizingMaskIntoConstraints = NO;

默认情况下，`view` 上的 `autoresizing mask` 会产生约束条件，以完全确定视图的位置。这允许 `AutoLayout` 系统跟踪其布局被手动控制的 `view` 的 `frame`（例如通过 `setFrame:`）。当你选择通过添加自己的约束来使用 `AutoLayout` 来定位视图时，必须 `self.translatesAutoresizingMaskIntoConstraints = NO;`。IB 会自动为你做这件事。

### constraintMaker

在这之后，创造了一个 `MASConstraintMaker` 类型的对象 `constraintMaker`，`MASConstraintMaker` 的初始化方法为：

```
- (id)initWithView:(MAS_VIEW *)view;
```

可以看到，传入的 `view` 参数是由一个叫做 `MAS_VIEW` 的宏来作为参数声明的，`MAS_VIEW` 的定义是：

```
#if TARGET_OS_IPHONE || TARGET_OS_TV

    #import <UIKit/UIKit.h>
    #define MAS_VIEW UIView
    ...

#elif TARGET_OS_MAC

    #import <AppKit/AppKit.h>
    #define MAS_VIEW NSView
    ...

#endif
```

因为 `Masonry` 是一个跨平台的框架，于是通过预编译宏来让在不同的平台上，`MAS_VIEW` 代表的意义不同。接下来看初始化方法的实现：

```
- (id)initWithView:(MAS_VIEW *)view {
    self = [super init];
    if (!self) return nil;
    
    self.view = view;
    self.constraints = NSMutableArray.new;
    
    return self;
}
```

在初始化方法中，将传入的 `view` 通过一个弱指针(`@property (nonatomic, weak) MAS_VIEW *view;`)保留在了 `constraintMaker` 中，同时初始化了一个名为 `constraints` 的 `NSMutableArray`，用来保存约束。

### block(constraintMaker);

接着，`constraintMaker` 通过 `block(constraintMaker);` 传递给了我们，而我们对它做了什么呢？

```
make.top.equalTo(superview.mas_top).with.offset(padding.top); 
make.left.equalTo(superview.mas_left).with.offset(padding.left);
make.bottom.equalTo(superview.mas_bottom).with.offset(-padding.bottom);
make.right.equalTo(superview.mas_right).with.offset(-padding.right);
```

我们将对视图约束的描述，以一种迥异于创建 `NSLayoutConstraints` 对象的方式描述了我们对视图的约束，我们以其中的一句作为例子来看看 `Masonry` 是如何实现链式 `DSL` 的。

```
make.top.equalTo(superview.mas_top).with.offset(padding.top);
```

#### make.top

`make` 是 `MASConstraintMaker` 类型的对象，这个类型封装了一系列只读 `MASConstraint` 属性，`top` 就是其中之一，声明和实现如下：

```
@property (nonatomic, strong, readonly) MASConstraint *top;
```

```
- (MASConstraint *)top {
    return [self addConstraintWithLayoutAttribute:NSLayoutAttributeTop];
}
```

`addConstraintWithLayoutAttribute:` 方法的实现为：

```
- (MASConstraint *)addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
    return [self constraint:nil addConstraintWithLayoutAttribute:layoutAttribute];
}

间接调用了 `constraint:addConstraintWithLayoutAttribute:` 方法：

- (MASConstraint *)constraint:(MASConstraint *)constraint addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
    MASViewAttribute *viewAttribute = [[MASViewAttribute alloc] initWithView:self.view layoutAttribute:layoutAttribute];
    MASViewConstraint *newConstraint = [[MASViewConstraint alloc] initWithFirstViewAttribute:viewAttribute];
    if ([constraint isKindOfClass:MASViewConstraint.class]) {
        //replace with composite constraint
        NSArray *children = @[constraint, newConstraint];
        MASCompositeConstraint *compositeConstraint = [[MASCompositeConstraint alloc] initWithChildren:children];
        compositeConstraint.delegate = self;
        [self constraint:constraint shouldBeReplacedWithConstraint:compositeConstraint];
        return compositeConstraint;
    }
    if (!constraint) {
        newConstraint.delegate = self;
        [self.constraints addObject:newConstraint];
    }
    return newConstraint;
}
```

让我们一句一句来看：

```
MASViewAttribute *viewAttribute = [[MASViewAttribute alloc] initWithView:self.view layoutAttribute:layoutAttribute];
```

首先，会初始化一个 `MASViewAttribute` 类型的对象（`viewAttribute`）,该类型的初始化方法是：

```
- (id)initWithView:(MAS_VIEW *)view layoutAttribute:(NSLayoutAttribute)layoutAttribute;
```

实现为：

```
- (id)initWithView:(MAS_VIEW *)view layoutAttribute:(NSLayoutAttribute)layoutAttribute {
    self = [self initWithView:view item:view layoutAttribute:layoutAttribute];
    return self;
}

- (id)initWithView:(MAS_VIEW *)view item:(id)item layoutAttribute:(NSLayoutAttribute)layoutAttribute {
    self = [super init];
    if (!self) return nil;
    
    _view = view;
    _item = item;
    _layoutAttribute = layoutAttribute;
    
    return self;
}
```

`MASViewAttribute` 是一个模型类，用于存储视图和视图对应的 `NSLayoutAttribute`。

接下来会初始化一个 `MASViewConstraint` 类型的对象（`newConstraint`）：

```
MASViewConstraint *newConstraint = [[MASViewConstraint alloc] initWithFirstViewAttribute:viewAttribute];
```

`MASViewConstraint`的初始化方法是：

```
- (id)initWithFirstViewAttribute:(MASViewAttribute *)firstViewAttribute;
```

```
- (id)initWithFirstViewAttribute:(MASViewAttribute *)firstViewAttribute {
    self = [super init];
    if (!self) return nil;
    
    _firstViewAttribute = firstViewAttribute;
    self.layoutPriority = MASLayoutPriorityRequired;
    self.layoutMultiplier = 1;
    
    return self;
}
```

`MASViewConstraint` 也是一个模型类，会通过刚才初始化的 `viewAttribute` 作为初始化参数，并存储在 `_firstViewAttribute` 的实例变量中。

接下来，由于 `constraint` 参数为 nil， 所以直接走到这里：

```
if (!constraint) {
    newConstraint.delegate = self;
    [self.constraints addObject:newConstraint];
}
```

将 `newConstraint` 对象的代理设为 `self` (`make`)，同时将其放置到 `constraints` 数组中。

简而言之就是，`make` 中的 `top` 方法会初始化一个 `MASViewAttribute` 类型的对象 `viewAttribute`，并通过该对象初始化一个 `MASViewConstraint` 类型的对象 `newConstraint`，让 `make` 作为 `newConstraint` 对象的 `delegate`，并存储在 `make` 的 `constraints` 属性中。接下来，`return newConstraint;`

`return newConstraint;` 看似简简单单的一句代码，却是 `Masonry` 这套链式 `DSL` 能够生效的核心。

#### .equalTo

`newConstraint` 是 `MASViewConstraint` 类型的对象，而 `MASViewConstraint` 又是 `MASConstraint` 的子类。在 `MASConstraint` 中，声明了一系列的方法，例如：

```
/**
 *	Sets the constraint relation to NSLayoutRelationEqual
 *  returns a block which accepts one of the following:
 *    MASViewAttribute, UIView, NSValue, NSArray
 *  see readme for more details.
 */
- (MASConstraint * (^)(id attr))equalTo;
```
这个方法的返回值是一个接受 `id` 类型的 `attr` 参数，返回 `MASConstraint` 类型的 `block`，这样写的意义是，既可以做到传递参数，返回 `self`，同时又确保了可以实现链式调用的 `DSL`。 

该方法的实现为：

```
- (MASConstraint * (^)(id))equalTo {
    return ^id(id attribute) {
        return self.equalToWithRelation(attribute, NSLayoutRelationEqual);
    };
}
```

在方法的实现中，调用了一个名为 `equalToWithRelation` 的内部方法，方法的实现为：

```
- (MASConstraint * (^)(id, NSLayoutRelation))equalToWithRelation { MASMethodNotImplemented(); }
```

`MASConstraint` 其实是一个基类，其中 `equalToWithRelation` 方法本身的实现里只有一个名为 `MASMethodNotImplemented();` 的宏，这个宏的实现仅仅是抛出一个异常：

```
#define MASMethodNotImplemented() \
    @throw [NSException exceptionWithName:NSInternalInconsistencyException \
                                   reason:[NSString stringWithFormat:@"You must override %@ in a subclass.", NSStringFromSelector(_cmd)] \
                                 userInfo:nil]
```

而我们在 `makeConstraints` 的时候，实际调用的是 `MASViewConstraint` 这个 `MASConstraint` 子类中的实现：

```
- (MASConstraint * (^)(id, NSLayoutRelation))equalToWithRelation {
    return ^id(id attribute, NSLayoutRelation relation) {
        if ([attribute isKindOfClass:NSArray.class]) {
            NSAssert(!self.hasLayoutRelation, @"Redefinition of constraint relation");
            NSMutableArray *children = NSMutableArray.new;
            for (id attr in attribute) {
                MASViewConstraint *viewConstraint = [self copy];
                viewConstraint.layoutRelation = relation;
                viewConstraint.secondViewAttribute = attr;
                [children addObject:viewConstraint];
            }
            MASCompositeConstraint *compositeConstraint = [[MASCompositeConstraint alloc] initWithChildren:children];
            compositeConstraint.delegate = self.delegate;
            [self.delegate constraint:self shouldBeReplacedWithConstraint:compositeConstraint];
            return compositeConstraint;
        } else {
            NSAssert(!self.hasLayoutRelation || self.layoutRelation == relation && [attribute isKindOfClass:NSValue.class], @"Redefinition of constraint relation");
            self.layoutRelation = relation;
            self.secondViewAttribute = attribute;
            return self;
        }
    };
}
```

该方法接收两个参数，一个表示了对应的属性（`mas_top`），一个表示了相等关系（`NSLayoutRelationEqual`），进入方法后会先对我们传入的属性做一个类型判断，我们传入的是一个单个的属性，所以会落入 `else` 分支，同样是依赖断言做了一系列保护性的判断，并将相等关系和视图属性分别赋值给 `layoutRelation` 和 `secondViewAttribute` 属性，并返回 `self`。

返回 `self`，看似简简单单的一个操作，却是 `Masonry` 能够实现链式 DSL 最重要的基石。（重要的事情说 n 遍）

#### superview.mas_top

再来看看我们传入的 `mas_top`，这是一个声明在 `View+MASAdditions.h` 当中的只读属性：

```
@property (nonatomic, strong, readonly) MASViewAttribute *mas_top;
```

简单生成并返回了一个 `MASViewAttribute` 属性的对象：

```
- (MASViewAttribute *)mas_top {
    return [[MASViewAttribute alloc] initWithView:self layoutAttribute:NSLayoutAttributeTop];
}
```

实现为：

```
- (id)initWithView:(MAS_VIEW *)view layoutAttribute:(NSLayoutAttribute)layoutAttribute;
```

```
- (id)initWithView:(MAS_VIEW *)view layoutAttribute:(NSLayoutAttribute)layoutAttribute {
    self = [self initWithView:view item:view layoutAttribute:layoutAttribute];
    return self;
}

- (id)initWithView:(MAS_VIEW *)view item:(id)item layoutAttribute:(NSLayoutAttribute)layoutAttribute {
    self = [super init];
    if (!self) return nil;
    
    _view = view;
    _item = item;
    _layoutAttribute = layoutAttribute;
    
    return self;
}
```

#### .with

我们继续往下看的时候，`.with` 同样是实现在 `MASViewConstraint` 中的一个方法：

```
- (MASConstraint *)with {
    return self;
}
```

同样是简简单单的返回了 `self`，而且是仅仅做了这一件事情。所以这个方法仅仅是一个语法……好吧都不能叫做语法糖，就叫语气助词吧，是为了让我们写出的 `DSL` 可读性更高而存在的。当然了，你要是觉得多余，也是可以不写的。

#### .offset

接下来是 `offset`：

```
- (MASConstraint * (^)(CGFloat offset))offset;
```

就是简简单单的一个赋值操作罢了，写成这么复杂的原因就是实现可以传递参数的链式调用。

```
- (MASConstraint * (^)(CGFloat))offset {
    return ^id(CGFloat offset){
        self.offset = offset;
        return self;
    };
}
```

以上就是 `make.top.equalTo(superview.mas_top).with.offset(padding.top);` 在 `Masonry` 内部到底做了些什么，其余几句也是类似的，总而言之就是：

> 对 `MASConstraintMaker` 类型的对象属性(`MASConstraint`的子类) `top`（或其他任何你想要去布局的属性），进行了初始化，并通过返回 `MASViewConstraint` 类型的对象（`newConstraint`），不断地调用 `newConstraint` 的对象方法，对 `newConstraint` 中的属性做了赋值，以确保其可以完整的表达一个 `NSLayoutConstraints`。

### install

在配置好我们想要的约束后，我们还需要对视图施加约束：

```
return [constraintMaker install];
```

先来看一下 `install` 方法：

```
- (NSArray *)install;
```

```
- (NSArray *)install {
    if (self.removeExisting) {
        NSArray *installedConstraints = [MASViewConstraint installedConstraintsForView:self.view];
        for (MASConstraint *constraint in installedConstraints) {
            [constraint uninstall];
        }
    }
    NSArray *constraints = self.constraints.copy;
    for (MASConstraint *constraint in constraints) {
        constraint.updateExisting = self.updateExisting;
        [constraint install];
    }
    [self.constraints removeAllObjects];
    return constraints;
}
```

首先，会对 `constraints` 属性做一份 `copy`，之后遍历 `constraints` 中的所有 `MASConstraint` 及其子类型的属性，并调用其 `install` 方法：

```
- (void)install;
```

实现为：

```
- (void)install {
    if (self.hasBeenInstalled) {
        return;
    }
    
    if ([self supportsActiveProperty] && self.layoutConstraint) {
        self.layoutConstraint.active = YES;
        [self.firstViewAttribute.view.mas_installedConstraints addObject:self];
        return;
    }
    
    MAS_VIEW *firstLayoutItem = self.firstViewAttribute.item;
    NSLayoutAttribute firstLayoutAttribute = self.firstViewAttribute.layoutAttribute;
    MAS_VIEW *secondLayoutItem = self.secondViewAttribute.item;
    NSLayoutAttribute secondLayoutAttribute = self.secondViewAttribute.layoutAttribute;

    // alignment attributes must have a secondViewAttribute
    // therefore we assume that is refering to superview
    // eg make.left.equalTo(@10)
    if (!self.firstViewAttribute.isSizeAttribute && !self.secondViewAttribute) {
        secondLayoutItem = self.firstViewAttribute.view.superview;
        secondLayoutAttribute = firstLayoutAttribute;
    }
    
    MASLayoutConstraint *layoutConstraint
        = [MASLayoutConstraint constraintWithItem:firstLayoutItem
                                        attribute:firstLayoutAttribute
                                        relatedBy:self.layoutRelation
                                           toItem:secondLayoutItem
                                        attribute:secondLayoutAttribute
                                       multiplier:self.layoutMultiplier
                                         constant:self.layoutConstant];
    
    layoutConstraint.priority = self.layoutPriority;
    layoutConstraint.mas_key = self.mas_key;
    
    if (self.secondViewAttribute.view) {
        MAS_VIEW *closestCommonSuperview = [self.firstViewAttribute.view mas_closestCommonSuperview:self.secondViewAttribute.view];
        NSAssert(closestCommonSuperview,
                 @"couldn't find a common superview for %@ and %@",
                 self.firstViewAttribute.view, self.secondViewAttribute.view);
        self.installedView = closestCommonSuperview;
    } else if (self.firstViewAttribute.isSizeAttribute) {
        self.installedView = self.firstViewAttribute.view;
    } else {
        self.installedView = self.firstViewAttribute.view.superview;
    }


    MASLayoutConstraint *existingConstraint = nil;
    if (self.updateExisting) {
        existingConstraint = [self layoutConstraintSimilarTo:layoutConstraint];
    }
    if (existingConstraint) {
        // just update the constant
        existingConstraint.constant = layoutConstraint.constant;
        self.layoutConstraint = existingConstraint;
    } else {
        [self.installedView addConstraint:layoutConstraint];
        self.layoutConstraint = layoutConstraint;
        [firstLayoutItem.mas_installedConstraints addObject:self];
    }
}
```

接下来，我们来一段一段分析：

#### hasBeenInstalled

首先，在 `install` 之前，会做一次判断，看是否已经被 `install` 过：

```
if (self.hasBeenInstalled) {
    return;
}
```

判断依据则是：

```
- (BOOL)hasBeenInstalled {
    return (self.layoutConstraint != nil) && [self isActive];
}
```

`layoutConstraint` 是一个 `MASLayoutConstraint` 类型的 `weak` 属性，`MASLayoutConstraint` 是 `NSLayoutConstraint` 的子类，只是为了增加一个属性（`mas_key`）。

```
@property (nonatomic, weak) MASLayoutConstraint *layoutConstraint;
``` 

`isActive` 则是通过判断 `layoutConstraint` 是否响应 `isActive` 以及 `isActive` 方法返回的结果，来综合决定。

```
- (BOOL)isActive {
    BOOL active = YES;
    if ([self supportsActiveProperty]) {
        active = [self.layoutConstraint isActive];
    }

    return active;
}
```

```
- (BOOL)supportsActiveProperty {
    return [self.layoutConstraint respondsToSelector:@selector(isActive)];
}
```

#### mas_installedConstraints

```
if ([self supportsActiveProperty] && self.layoutConstraint) {
    self.layoutConstraint.active = YES;
    [self.firstViewAttribute.view.mas_installedConstraints addObject:self];
    return;
}
```

如果 `supportsActiveProperty` 且 `layoutConstraint` 不为空，则将 `layoutConstraint.active` 设为 `YES`，并将其添加到 `firstViewAttribute.view` 的 `mas_installedConstraints` 只读属性中去。

```
@property (nonatomic, readonly) NSMutableSet *mas_installedConstraints;
```

`mas_installedConstraints` 是一个可变集合，是通过分类给 MAS_View 类添加的关联对象，用来保存已经 `active` 的对象。

```
static char kInstalledConstraintsKey;

- (NSMutableSet *)mas_installedConstraints {
    NSMutableSet *constraints = objc_getAssociatedObject(self, &kInstalledConstraintsKey);
    if (!constraints) {
        constraints = [NSMutableSet set];
        objc_setAssociatedObject(self, &kInstalledConstraintsKey, constraints, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    return constraints;
}
```

#### 生成约束

```
MAS_VIEW *firstLayoutItem = self.firstViewAttribute.item;
NSLayoutAttribute firstLayoutAttribute = self.firstViewAttribute.layoutAttribute;
MAS_VIEW *secondLayoutItem = self.secondViewAttribute.item;
NSLayoutAttribute secondLayoutAttribute = self.secondViewAttribute.layoutAttribute;

// alignment attributes must have a secondViewAttribute
// therefore we assume that is refering to superview
// eg make.left.equalTo(@10)
if (!self.firstViewAttribute.isSizeAttribute && !self.secondViewAttribute) {
    secondLayoutItem = self.firstViewAttribute.view.superview;
    secondLayoutAttribute = firstLayoutAttribute;
}
    
MASLayoutConstraint *layoutConstraint
    = [MASLayoutConstraint constraintWithItem:firstLayoutItem
                                    attribute:firstLayoutAttribute
                                    relatedBy:self.layoutRelation
                                       toItem:secondLayoutItem
                                    attribute:secondLayoutAttribute
                                   multiplier:self.layoutMultiplier
                                     constant:self.layoutConstant];
    
layoutConstraint.priority = self.layoutPriority;
layoutConstraint.mas_key = self.mas_key;
```

其实就是把我们再 `block` 中通过诸如 `make.top.equalTo(superview.mas_top).with.offset(padding.top);` 这样的语句配置的属性作为 `MASLayoutConstraint` 初始化方法的参数，生成一个约束。唯一需要注意的是，如果你设置的是一个 `isSizeAttribute`，并且 `secondViewAttribute` 为 `nil`，会做一些额外的参数调整。

#### 施加约束

```
if (self.secondViewAttribute.view) {
    MAS_VIEW *closestCommonSuperview = [self.firstViewAttribute.view mas_closestCommonSuperview:self.secondViewAttribute.view];
    NSAssert(closestCommonSuperview,
             @"couldn't find a common superview for %@ and %@",
             self.firstViewAttribute.view, self.secondViewAttribute.view);
    self.installedView = closestCommonSuperview;
} else if (self.firstViewAttribute.isSizeAttribute) {
    self.installedView = self.firstViewAttribute.view;
} else {
    self.installedView = self.firstViewAttribute.view.superview;
}
```

如果 `secondViewAttribute.view` 中的存在，就通过 `mas_closestCommonSuperview` 方法寻找最近的公共子视图：

```
/**
 *	Finds the closest common superview between this view and another view
 *
 *	@param	view	other view
 *
 *	@return	returns nil if common superview could not be found
 */
- (instancetype)mas_closestCommonSuperview:(MAS_VIEW *)view;
```

```
- (instancetype)mas_closestCommonSuperview:(MAS_VIEW *)view {
    MAS_VIEW *closestCommonSuperview = nil;

    MAS_VIEW *secondViewSuperview = view;
    while (!closestCommonSuperview && secondViewSuperview) {
        MAS_VIEW *firstViewSuperview = self;
        while (!closestCommonSuperview && firstViewSuperview) {
            if (secondViewSuperview == firstViewSuperview) {
                closestCommonSuperview = secondViewSuperview;
            }
            firstViewSuperview = firstViewSuperview.superview;
        }
        secondViewSuperview = secondViewSuperview.superview;
    }
    return closestCommonSuperview;
}
```

递归求解。

如果设置的是一个尺寸约束（`firstViewAttribute.isSizeAttribute`），则施加在 `firstViewAttribute.view` 上。否则施加在 `firstViewAttribute.view.superView` 上。

```
MASLayoutConstraint *existingConstraint = nil;
if (self.updateExisting) {
    existingConstraint = [self layoutConstraintSimilarTo:layoutConstraint];
}
if (existingConstraint) {
    // just update the constant
    existingConstraint.constant = layoutConstraint.constant;
    self.layoutConstraint = existingConstraint;
} else {
    [self.installedView addConstraint:layoutConstraint];
    self.layoutConstraint = layoutConstraint;
    [firstLayoutItem.mas_installedConstraints addObject:self];
}
```

对视图施加约束，并将 `layoutConstraint` 存储在 `self.layoutConstraint` 属性中，同时把 `self` 存储到之前提到过的叫做 `mas_installedConstraints` 的关联对象中。至此，文章开始提到的例子业已完成。

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~