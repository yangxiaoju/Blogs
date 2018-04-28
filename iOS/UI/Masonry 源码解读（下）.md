# Masonry 源码解读（下）

## 前言

书接上文，我们在上一篇文章中已经解解读了 Masonry 框架中最核心的功能是如何实现的，接下来再看一下另外的一些点。

## 设置约束不相等性

`Masonry` 中为我们准备了设置约束不相等时的方法：

``` 
- (MASConstraint * (^)(id attr))greaterThanOrEqualTo;

- (MASConstraint * (^)(id attr))lessThanOrEqualTo;
```

以 `greaterThanOrEqualTo` 为例，看一下它和 `equalTo` 的区别：

```
- (MASConstraint * (^)(id))greaterThanOrEqualTo {
    return ^id(id attribute) {
        return self.equalToWithRelation(attribute, NSLayoutRelationGreaterThanOrEqual);
    };
}
```

同 `equalTo` 别无二致，同样是调用了 `equalToWithRelation` 方法，只不过 `relation` 参数不同。

```
- (MASConstraint * (^)(id, NSLayoutRelation))equalToWithRelation {
    return ^id(id attribute, NSLayoutRelation relation) {
        if ([attribute isKindOfClass:NSArray.class]) {
            ...
        } else {
            NSAssert(!self.hasLayoutRelation || self.layoutRelation == relation && [attribute isKindOfClass:NSValue.class], @"Redefinition of constraint relation");
            self.layoutRelation = relation; // 在这里设置的
            self.secondViewAttribute = attribute;
            return self;
        }
    };
}
```

### autoboxing

由上面的代码我们可以看到，传递给 `equalToWithRelation` 方法的 `attribute` 参数是一个 `id` 类型的对象，这意味每次调用 `.equalTo` 方法时，需要对纯数字的参数进行包装。若是简单的数字约束还好，但是针对 `size` 等需要传递结构体才能解决的约束，就显得很繁琐了。`Masonry` 为我们提供了一些宏用来解决这个问题：

```
#define mas_equalTo(...)                 equalTo(MASBoxValue((__VA_ARGS__)))
#define mas_greaterThanOrEqualTo(...)    greaterThanOrEqualTo(MASBoxValue((__VA_ARGS__)))
#define mas_lessThanOrEqualTo(...)       lessThanOrEqualTo(MASBoxValue((__VA_ARGS__)))

#define mas_offset(...)                  valueOffset(MASBoxValue((__VA_ARGS__)))


#ifdef MAS_SHORTHAND_GLOBALS

#define equalTo(...)                     mas_equalTo(__VA_ARGS__)
#define greaterThanOrEqualTo(...)        mas_greaterThanOrEqualTo(__VA_ARGS__)
#define lessThanOrEqualTo(...)           mas_lessThanOrEqualTo(__VA_ARGS__)

#define offset(...)                      mas_offset(__VA_ARGS__)

#endif
```

以其中一种为例：

```
#define mas_equalTo(...)                 equalTo(MASBoxValue((__VA_ARGS__)))
```

可以看出，`mas_equalTo` 是通过每次调用 `equalTo` 方法时，对参数调用 `MASBoxValue()` 这个宏来解决 autoboxing 问题的。而 `MASBoxValue()` 宏的定义是：

```
#define MASBoxValue(value) _MASBoxValue(@encode(__typeof__((value))), (value))
```

转而调用了一个名为 `_MASBoxValue` 的函数：

```
static inline id _MASBoxValue(const char *type, ...) {
    va_list v;
    va_start(v, type);
    id obj = nil;
    if (strcmp(type, @encode(id)) == 0) {
        id actual = va_arg(v, id);
        obj = actual;
    } else if (strcmp(type, @encode(CGPoint)) == 0) {
        CGPoint actual = (CGPoint)va_arg(v, CGPoint);
        obj = [NSValue value:&actual withObjCType:type];
    } else if (strcmp(type, @encode(CGSize)) == 0) {
        CGSize actual = (CGSize)va_arg(v, CGSize);
        obj = [NSValue value:&actual withObjCType:type];
    } else if (strcmp(type, @encode(MASEdgeInsets)) == 0) {
        MASEdgeInsets actual = (MASEdgeInsets)va_arg(v, MASEdgeInsets);
        obj = [NSValue value:&actual withObjCType:type];
    } else if (strcmp(type, @encode(double)) == 0) {
        double actual = (double)va_arg(v, double);
        obj = [NSNumber numberWithDouble:actual];
    } else if (strcmp(type, @encode(float)) == 0) {
        float actual = (float)va_arg(v, double);
        obj = [NSNumber numberWithFloat:actual];
    } else if (strcmp(type, @encode(int)) == 0) {
        int actual = (int)va_arg(v, int);
        obj = [NSNumber numberWithInt:actual];
    } else if (strcmp(type, @encode(long)) == 0) {
        long actual = (long)va_arg(v, long);
        obj = [NSNumber numberWithLong:actual];
    } else if (strcmp(type, @encode(long long)) == 0) {
        long long actual = (long long)va_arg(v, long long);
        obj = [NSNumber numberWithLongLong:actual];
    } else if (strcmp(type, @encode(short)) == 0) {
        short actual = (short)va_arg(v, int);
        obj = [NSNumber numberWithShort:actual];
    } else if (strcmp(type, @encode(char)) == 0) {
        char actual = (char)va_arg(v, int);
        obj = [NSNumber numberWithChar:actual];
    } else if (strcmp(type, @encode(bool)) == 0) {
        bool actual = (bool)va_arg(v, int);
        obj = [NSNumber numberWithBool:actual];
    } else if (strcmp(type, @encode(unsigned char)) == 0) {
        unsigned char actual = (unsigned char)va_arg(v, unsigned int);
        obj = [NSNumber numberWithUnsignedChar:actual];
    } else if (strcmp(type, @encode(unsigned int)) == 0) {
        unsigned int actual = (unsigned int)va_arg(v, unsigned int);
        obj = [NSNumber numberWithUnsignedInt:actual];
    } else if (strcmp(type, @encode(unsigned long)) == 0) {
        unsigned long actual = (unsigned long)va_arg(v, unsigned long);
        obj = [NSNumber numberWithUnsignedLong:actual];
    } else if (strcmp(type, @encode(unsigned long long)) == 0) {
        unsigned long long actual = (unsigned long long)va_arg(v, unsigned long long);
        obj = [NSNumber numberWithUnsignedLongLong:actual];
    } else if (strcmp(type, @encode(unsigned short)) == 0) {
        unsigned short actual = (unsigned short)va_arg(v, unsigned int);
        obj = [NSNumber numberWithUnsignedShort:actual];
    }
    va_end(v);
    return obj;
}
```

在函数定义的最开始使用了 `static inline`, 这是内联函数的定义。引入内联函数的目的是为了解决程序中函数调用的效率问题。 

函数调用会带来降低效率的问题，因为调用函数实际上将程序执行顺序转移到函数所存放在内存中某个地址，将函数的程序内容执行完后，再返回到转去执行该函数前的地方。这种转移操作要求在转去前要保护现场并记忆执行的地址，转回后先要恢复现场，并按原来保存地址继续执行。因此，函数调用要有一定的时间和空间方面的开销，于是将影响其效率。特别是对于一些函数体代码不是很大，但又频繁地被调用的函数来讲，解决其效率问题更为重要。引入内联函数实际上就是为了解决这一问题。 

在程序编译时，编译器将程序中出现的内联函数的调用表达式用内联函数的函数体来进行替换。显然，这种做法不会产生转去转回的问题，但是由于在编译时将函数休中的代码被替代到程序中，因此会增加目标程序代码量，进而增加空间开销，而在时间代销上不象函数调用时那么大，可见它是以目标代码的增加为代价来换取时间的节省。

因为每一个调用 `mas_` 前缀开头的宏都会调用 `_MASBoxValue` 函数，所以在这里进行了 `static inline` 的处理。

`_MASBoxValue` 函数的参数声明为 `const char *type, ...` 说明其接受的是可变参数。在使用 `MASBoxValue(value)` 对 `_MASBoxValue` 函数进行调用时，传入的参数只有两个：值的类型编码（`@encode(__typeof__((value)))`）和值（`value`）。

`@encode` ，`@编译器指令`之一，返回一个给定类型编码为一种内部表示的字符串（例如，`@encode(int) → i`），类似于 `ANSI C` 的 `typeof` 操作。苹果的 `Objective-C` 运行时库内部利用类型编码帮助加快消息分发。详细内容可以参考 `NSHipster` 这篇博客：[Type Encodings](http://nshipster.cn/type-encodings/)

`Objective-C` 中对于可变参数的处理是依赖一组宏来实现的，通过代码来看更直观一点：

```
static inline id _MASBoxValue(const char *type, ...) {
    va_list v; // 指向变参的指针
    va_start(v, type); // 使用第一个参数来初使化 v 指针
    id obj = nil; // 声明一个 id 类型的指针（用于存储返回值）
    if (strcmp(type, @encode(id)) == 0) { // strcmp() 函数是用来比较字符串的，如果相同则返回 0
        id actual = va_arg(v, id); // 返回可变参数，va_arg 第二个参数为可变参数类型，如果有多个可变参数，依次调用可获取各个参数
        obj = actual; // 由于传入的本身就是 id 类型，所以不需要类型转换
    } else if (strcmp(type, @encode(CGPoint)) == 0) { // 如果匹配 CGPoint 类型
        CGPoint actual = (CGPoint)va_arg(v, CGPoint); // 取出可变参数
        obj = [NSValue value:&actual withObjCType:type]; // 通过 NSValue 对基本数据类型做一次包装
    } ... // 之后的分支同理。
    ...
    va_end(v); // 结束可变参数的获取
    return obj; // 返回转换后的结果
}
```

### NSArray

传入的参数不仅可以是单个值，也可以是数组：

```
make.height.equalTo(@[view1.mas_height, view2.mas_height]);
```

内部实现为：

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
            ...
        }
    };
}
```

对于数组类型的参数 `attribute` 参数，会落入第一个分支，将数据中的参数一一拆出来，分别生成 `MASViewConstraint` 类型的对象，再通过这些对象来初始化一个 `MASCompositeConstraint` 类型的对象（`compositeConstraint`），接下来我们看一下 `MASCompositeConstraint` 的初始化方法：

```
- (id)initWithChildren:(NSArray *)children;
```

```
- (id)initWithChildren:(NSArray *)children {
    self = [super init];
    if (!self) return nil;

    _childConstraints = [children mutableCopy];
    for (MASConstraint *constraint in _childConstraints) {
        constraint.delegate = self;
    }

    return self;
}
```

与 `MASViewConstraint` 类型类似，都是继承与 `MASConstraint` 类的模型类，用 `@property (nonatomic, strong) NSMutableArray *childConstraints;` 属性来保存一组约束，这其中的每个约束的 `delegate` 都是 `self`。

再之后，调用 `[self.delegate constraint:self shouldBeReplacedWithConstraint:compositeConstraint];` 方法：

```
- (void)constraint:(MASConstraint *)constraint shouldBeReplacedWithConstraint:(MASConstraint *)replacementConstraint {
    NSUInteger index = [self.constraints indexOfObject:constraint];
    NSAssert(index != NSNotFound, @"Could not find constraint %@", constraint);
    [self.constraints replaceObjectAtIndex:index withObject:replacementConstraint];
}
```

去替换数组中对应存储的约束。

在对视图施加约束的时候也有所不同，`MASCompositeConstraint` 是通过遍历 `childConstraints` 中所有的实例挨个 `install`：

```
- (void)install {
    for (MASConstraint *constraint in self.childConstraints) {
        constraint.updateExisting = self.updateExisting;
        [constraint install];
    }
}
```

### 优先级

约束是可以设置优先级的，从 0-1000，不过通常情况下也不需要这么多个等级，让我们先来看一下 `Masonry` 中是如何实现这一功能的：

```
make.left.greaterThanOrEqualTo(label.mas_left).with.priorityLow();
```

通过调用 `.priorityLow()` 方法，就为这条约束设置了低优先级，下面是该方法的声明和实现：

```
- (MASConstraint * (^)(void))priorityLow;
```

```
- (MASConstraint * (^)(void))priorityLow {
    return ^id{
        self.priority(MASLayoutPriorityDefaultLow);
        return self;
    };
}
```

`MASLayoutPriorityDefaultLow` 是预先定义好的一组常量之一，定义如下，是通过预编译宏对跨平台的优先级做了一层封装：

```
#if TARGET_OS_IPHONE || TARGET_OS_TV

    #import <UIKit/UIKit.h>
    #define MAS_VIEW UIView
    #define MAS_VIEW_CONTROLLER UIViewController
    #define MASEdgeInsets UIEdgeInsets

    typedef UILayoutPriority MASLayoutPriority;
    static const MASLayoutPriority MASLayoutPriorityRequired = UILayoutPriorityRequired;
    static const MASLayoutPriority MASLayoutPriorityDefaultHigh = UILayoutPriorityDefaultHigh;
    static const MASLayoutPriority MASLayoutPriorityDefaultMedium = 500;
    static const MASLayoutPriority MASLayoutPriorityDefaultLow = UILayoutPriorityDefaultLow;
    static const MASLayoutPriority MASLayoutPriorityFittingSizeLevel = UILayoutPriorityFittingSizeLevel;

#elif TARGET_OS_MAC

    #import <AppKit/AppKit.h>
    #define MAS_VIEW NSView
    #define MASEdgeInsets NSEdgeInsets

    typedef NSLayoutPriority MASLayoutPriority;
    static const MASLayoutPriority MASLayoutPriorityRequired = NSLayoutPriorityRequired;
    static const MASLayoutPriority MASLayoutPriorityDefaultHigh = NSLayoutPriorityDefaultHigh;
    static const MASLayoutPriority MASLayoutPriorityDragThatCanResizeWindow = NSLayoutPriorityDragThatCanResizeWindow;
    static const MASLayoutPriority MASLayoutPriorityDefaultMedium = 501;
    static const MASLayoutPriority MASLayoutPriorityWindowSizeStayPut = NSLayoutPriorityWindowSizeStayPut;
    static const MASLayoutPriority MASLayoutPriorityDragThatCannotResizeWindow = NSLayoutPriorityDragThatCannotResizeWindow;
    static const MASLayoutPriority MASLayoutPriorityDefaultLow = NSLayoutPriorityDefaultLow;
    static const MASLayoutPriority MASLayoutPriorityFittingSizeCompression = NSLayoutPriorityFittingSizeCompression;

#endif
```

而 `self.priority()` 就是简单的赋值：

```
- (MASConstraint * (^)(MASLayoutPriority))priority {
    return ^id(MASLayoutPriority priority) {
        NSAssert(!self.hasBeenInstalled,
                 @"Cannot modify constraint priority after it has been installed");
        
        self.layoutPriority = priority;
        return self;
    };
}
```

当然我们也可以为每一条约束设置我们想要的优先级，如下所示，原理相同就不在赘述了：

```
make.top.equalTo(label.mas_top).with.priority(600);
```

## 通过另一个视图为当前视图设置约束

设置约束不一定每次都要传入数值，也可以依据别的视图来设置约束：

```
make.edges.equalTo(view2);
```

由于传入的并非 `NSArray` 类型的参数，于是同样会落入 `equalToWithRelation` 的 `else` 分支：

```
- (MASConstraint * (^)(id, NSLayoutRelation))equalToWithRelation {
    return ^id(id attribute, NSLayoutRelation relation) {
        if ([attribute isKindOfClass:NSArray.class]) {
            ...
        } else {
            NSAssert(!self.hasLayoutRelation || self.layoutRelation == relation && [attribute isKindOfClass:NSValue.class], @"Redefinition of constraint relation");
            self.layoutRelation = relation;
            self.secondViewAttribute = attribute; // 在这里将 view2 传入
            return self;
        }
    };
}
```

不同类型的 `attribute` 是在 `set` 方法里处理的，让我们看一下 `self.secondViewAttribute` 属性的 `set` 方法：

```
- (void)setSecondViewAttribute:(id)secondViewAttribute {
    if ([secondViewAttribute isKindOfClass:NSValue.class]) {
        [self setLayoutConstantWithValue:secondViewAttribute];
    } else if ([secondViewAttribute isKindOfClass:MAS_VIEW.class]) {
        _secondViewAttribute = [[MASViewAttribute alloc] initWithView:secondViewAttribute layoutAttribute:self.firstViewAttribute.layoutAttribute]; // 重点在这行
    } else if ([secondViewAttribute isKindOfClass:MASViewAttribute.class]) {
        MASViewAttribute *attr = secondViewAttribute;
        if (attr.layoutAttribute == NSLayoutAttributeNotAnAttribute) {
            _secondViewAttribute = [[MASViewAttribute alloc] initWithView:attr.view item:attr.item layoutAttribute:self.firstViewAttribute.layoutAttribute];;
        } else {
            _secondViewAttribute = secondViewAttribute;
        }
    } else {
        NSAssert(NO, @"attempting to add unsupported attribute: %@", secondViewAttribute);
    }
}
```

在传入的 `secondViewAttribute` 是 `view` 的情形下，会通过 `self.firstViewAttribute.layoutAttribute` 来从 `view2` 中提取出对应的约束数值，创建新的 `MASViewAttribute` 赋值给 `_secondViewAttribute`。

## 更新约束

在我看来 `Masonry` 相对于原生和其他大多数 `AutoLayout` 框架最大的优点在于，当你想更新约束的时候，不需要持有对应约束的引用，而是调用 `mas_updateConstraints`(用于更新约束) 或 `mas_remakeConstraints`(用于重设约束)即可，下面看一下这两个方法的声明和实现：

### mas_updateConstraints

```
- (NSArray *)mas_updateConstraints:(void(NS_NOESCAPE ^)(MASConstraintMaker *make))block;
```

```
- (NSArray *)mas_updateConstraints:(void(^)(MASConstraintMaker *))block {
    self.translatesAutoresizingMaskIntoConstraints = NO;
    MASConstraintMaker *constraintMaker = [[MASConstraintMaker alloc] initWithView:self];
    constraintMaker.updateExisting = YES;
    block(constraintMaker);
    return [constraintMaker install];
}
```

相较于 `mas_makeConstraints` 区别在于 `constraintMaker.updateExisting = YES;` ，下面看一下 `updateExisting` 这个标志位对于施加约束的影响：

```
- (NSArray *)install {
    if (self.removeExisting) {
        ...
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

在 `install` 方法中，会对 `constraints` 中的每一个 `constraint` 的 `updateExisting` 的标志位都设为 `YES`，再来看一下 `MASConstraint` 的 `install` 方法：

```
- (void)install {
    ...

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

如果 `updateExisting` 为 `YES`，就通过 `layoutConstraintSimilarTo` 方法来尝试拿到之前已经 `install` 过的类似约束，如果能拿到，那么就只是更改了原有约束的 `constant`，如果没有，就正常添加约束。下面是 `layoutConstraintSimilarTo` 方法的实现：

```
- (MASLayoutConstraint *)layoutConstraintSimilarTo:(MASLayoutConstraint *)layoutConstraint {
    // check if any constraints are the same apart from the only mutable property constant

    // go through constraints in reverse as we do not want to match auto-resizing or interface builder constraints
    // and they are likely to be added first.
    for (NSLayoutConstraint *existingConstraint in self.installedView.constraints.reverseObjectEnumerator) {
        if (![existingConstraint isKindOfClass:MASLayoutConstraint.class]) continue;
        if (existingConstraint.firstItem != layoutConstraint.firstItem) continue;
        if (existingConstraint.secondItem != layoutConstraint.secondItem) continue;
        if (existingConstraint.firstAttribute != layoutConstraint.firstAttribute) continue;
        if (existingConstraint.secondAttribute != layoutConstraint.secondAttribute) continue;
        if (existingConstraint.relation != layoutConstraint.relation) continue;
        if (existingConstraint.multiplier != layoutConstraint.multiplier) continue;
        if (existingConstraint.priority != layoutConstraint.priority) continue;

        return (id)existingConstraint;
    }
    return nil;
}
```

简单的遍历 `constraints` 数组里的所有约束，并比较属性是否相同。

### mas_remakeConstraints

```
- (NSArray *)mas_remakeConstraints:(void(NS_NOESCAPE ^)(MASConstraintMaker *make))block;
```

```
- (NSArray *)mas_remakeConstraints:(void(^)(MASConstraintMaker *make))block {
    self.translatesAutoresizingMaskIntoConstraints = NO;
    MASConstraintMaker *constraintMaker = [[MASConstraintMaker alloc] initWithView:self];
    constraintMaker.removeExisting = YES;
    block(constraintMaker);
    return [constraintMaker install];
}
```

相较于 `mas_makeConstraints` 区别在于会简单粗暴的把所有视图上试驾过的约束挨个拿出来 `uninstall`，由此可见这个方法对于性能的影响是蛮大的，要慎用：

```
- (NSArray *)install {
    if (self.removeExisting) {
        NSArray *installedConstraints = [MASViewConstraint installedConstraintsForView:self.view];
        for (MASConstraint *constraint in installedConstraints) {
            [constraint uninstall];
        }
    }
    ... 
    return constraints;
}
```

`installedConstraintsForView` 方法就是简单的返回 `mas_installedConstraints` 中的所有对象，在此不再赘述：

```
+ (NSArray *)installedConstraintsForView:(MAS_VIEW *)view {
    return [view.mas_installedConstraints allObjects];
}
```

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~