# SnapKit 源码解读（二）：DSLs

与 `Masonry` 不同，`SnapKit` 充分利用了 `Swift` 的语言特性，用更优雅的方式实现了一套 `DSL`。而这一切的开始，源于 `ConstraintDSL`。

## ConstraintDSL

`ConstraintDSL` 是一个协议：

```
public protocol ConstraintDSL {
    
    var target: AnyObject? { get }
    
    func setLabel(_ value: String?)
    func label() -> String?
    
}
```

### public

`public` 的作用是，即使两部分代码处于两个不同的 `module`，也能使用这个协议。在寻常的项目中，默认的权限级别是 `internal`，即在同一个 `module` 内可以使用。如果是写三方库，则要用 `public` 来标记提供给外界的 `API` 了。

### 协议扩展

与 `Objective-C` 中的协议不同，`Swift` 中的协议可以通过扩展为协议添加默认的实现，如下所示，为 `setLabel` ,`label` 协议方法提供了默认的实现：

```
extension ConstraintDSL {
    
    public func setLabel(_ value: String?) {
        objc_setAssociatedObject(self.target as Any, &labelKey, value, .OBJC_ASSOCIATION_COPY_NONATOMIC)
    }
    public func label() -> String? {
        return objc_getAssociatedObject(self.target as Any, &labelKey) as? String
    }
    
}
```

### 关联对象

提供给 `ConstraintDSL` 协议的默认实现其实是实现了关联对象的效果，关联对象是通过扩展实现属性的一种方式，`set` 方法使用的 `API` 为：

```
@available(iOS 3.1, *)
public func objc_setAssociatedObject(_ object: Any, _ key: UnsafeRawPointer, _ value: Any?, _ policy: objc_AssociationPolicy)
```

`object` 参数表示关联的源对象，在这里是 `self.target as Any`，因为 `Swift` 是一门强类型的语言，所以需要做一个类型转换。

`key` 参数表示关联对象的键，`SnapKit` 是声明了一个 `private var labelKey: UInt8 = 0` 来作为 `key`，使用的时候需要配合 `&` 操作符一起使用。

`value` 参数表示与对象的关键键关联的值。

`policy` 参数表示关联的策略，是一个枚举，取值如下：

```
public enum objc_AssociationPolicy : UInt {

    
    /**< Specifies a weak reference to the associated object. */
    case OBJC_ASSOCIATION_ASSIGN

    /**< Specifies a strong reference to the associated object. 
     *   The association is not made atomically. */
    case OBJC_ASSOCIATION_RETAIN_NONATOMIC

    
    /**< Specifies that the associated object is copied. 
     *   The association is not made atomically. */
    case OBJC_ASSOCIATION_COPY_NONATOMIC

    
    /**< Specifies a strong reference to the associated object.
     *   The association is made atomically. */
    case OBJC_ASSOCIATION_RETAIN

    
    /**< Specifies that the associated object is copied.
     *   The association is made atomically. */
    case OBJC_ASSOCIATION_COPY
}
```

与 `Objective-C` 属性的对应关系是一致的，在此不再赘述。

关联引用的 `get` 的 `API` 如下：

```
@available(iOS 3.1, *)
public func objc_getAssociatedObject(_ object: Any, _ key: UnsafeRawPointer) -> Any?
```

所需的两个参数与 `set` 中的一致。

## ConstraintBasicAttributesDSL

`ConstraintBasicAttributesDSL` 是一个继承于 `ConstraintDSL` 的协议，如下所示：

```
public protocol ConstraintBasicAttributesDSL : ConstraintDSL {
}
```

进而又通过扩展为其添加了一些计算属性，会返回对象和其相关的布局属性的模型对象，属于 `ConstraintItem` 类型，如下所示：

```
extension ConstraintBasicAttributesDSL {
    
    // MARK: Basics
    
    public var left: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.left)
    }
    
    public var top: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.top)
    }
    
    public var right: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.right)
    }
    
    public var bottom: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.bottom)
    }
    
    public var leading: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.leading)
    }
    
    public var trailing: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.trailing)
    }
    
    public var width: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.width)
    }
    
    public var height: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.height)
    }
    
    public var centerX: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.centerX)
    }
    
    public var centerY: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.centerY)
    }
    
    public var edges: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.edges)
    }
    
    public var size: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.size)
    }
    
    public var center: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.center)
    }
    
}
```

## ConstraintAttributesDSL

`ConstraintAttributesDSL` 是一个继承于 `ConstraintBasicAttributesDSL` 的协议，如下所示：

```
public protocol ConstraintAttributesDSL : ConstraintBasicAttributesDSL {
}
```

进而又通过扩展为其添加了一些计算属性，会返回对象和其相关的布局属性（`baseline` 和 `margin` 相关）的模型对象，属于 `ConstraintItem` 类型，如下所示：

```
extension ConstraintAttributesDSL {
    
    // MARK: Baselines
    
    @available(*, deprecated:3.0, message:"Use .lastBaseline instead")
    public var baseline: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.lastBaseline)
    }
    
    @available(iOS 8.0, OSX 10.11, *)
    public var lastBaseline: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.lastBaseline)
    }
    
    @available(iOS 8.0, OSX 10.11, *)
    public var firstBaseline: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.firstBaseline)
    }
    
    // MARK: Margins
    
    @available(iOS 8.0, *)
    public var leftMargin: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.leftMargin)
    }
    
    @available(iOS 8.0, *)
    public var topMargin: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.topMargin)
    }
    
    @available(iOS 8.0, *)
    public var rightMargin: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.rightMargin)
    }
    
    @available(iOS 8.0, *)
    public var bottomMargin: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.bottomMargin)
    }
    
    @available(iOS 8.0, *)
    public var leadingMargin: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.leadingMargin)
    }
    
    @available(iOS 8.0, *)
    public var trailingMargin: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.trailingMargin)
    }
    
    @available(iOS 8.0, *)
    public var centerXWithinMargins: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.centerXWithinMargins)
    }
    
    @available(iOS 8.0, *)
    public var centerYWithinMargins: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.centerYWithinMargins)
    }
    
    @available(iOS 8.0, *)
    public var margins: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.margins)
    }
    
    @available(iOS 8.0, *)
    public var centerWithinMargins: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.centerWithinMargins)
    }
    
}
```

## ConstraintViewDSL

`ConstraintViewDSL` 是一个遵守了 `ConstraintAttributesDSL` 协议的结构体，定义如下：

```
public struct ConstraintViewDSL: ConstraintAttributesDSL {
    
    ...
    
}
```

`ConstraintViewDSL` 初始化方法只需要一个参数，配合内部的一个属性 `view` 存储需要布局的视图：

```
internal let view: ConstraintView
    
internal init(view: ConstraintView) {
    self.view = view
        
}
```

`ConstraintViewDSL` 真正需要自己实现的代理中的属性只有一个，就是 `target`，这是一个用来返回布局对应的视图的计算属性，相关实现如下：

```
public var target: AnyObject? {
    return self.view
}
```

`ConstraintViewDSL` 声明了一些关键方法，内部都是通过调用 `ConstraintMaker` 类的相关方法实现的：

```
@discardableResult
public func prepareConstraints(_ closure: (_ make: ConstraintMaker) -> Void) -> [Constraint] {
    return ConstraintMaker.prepareConstraints(item: self.view, closure: closure)
}
    
public func makeConstraints(_ closure: (_ make: ConstraintMaker) -> Void) {
    ConstraintMaker.makeConstraints(item: self.view, closure: closure)
}
    
public func remakeConstraints(_ closure: (_ make: ConstraintMaker) -> Void) {
    ConstraintMaker.remakeConstraints(item: self.view, closure: closure)
}
    
public func updateConstraints(_ closure: (_ make: ConstraintMaker) -> Void) {
    ConstraintMaker.updateConstraints(item: self.view, closure: closure)
}
    
public func removeConstraints() {
    ConstraintMaker.removeConstraints(item: self.view)
}
```

### @discardableResult

有的时候，方法有一个返回值，但我们经常性的会忽视这个返回值，这时可以在方法前加上 `@discardableResult` 标记，可以在这种情况下不出现警告。

最后，声明了一些计算属性，作为一些设置 `view` 相关属性的 `shortcut`：

```
public var contentHuggingHorizontalPriority: Float {
    get {
        return self.view.contentHuggingPriority(for: .horizontal).rawValue
    }
    set {
        self.view.setContentHuggingPriority(LayoutPriority(rawValue: newValue), for: .horizontal)
    }
}
    
public var contentHuggingVerticalPriority: Float {
    get {
        return self.view.contentHuggingPriority(for: .vertical).rawValue
    }
    set {
        self.view.setContentHuggingPriority(LayoutPriority(rawValue: newValue), for: .vertical)
    }
}
    
public var contentCompressionResistanceHorizontalPriority: Float {
    get {
        return self.view.contentCompressionResistancePriority(for: .horizontal).rawValue
    }
    set {
        self.view.setContentCompressionResistancePriority(LayoutPriority(rawValue: newValue), for: .horizontal)
    }
}
    
public var contentCompressionResistanceVerticalPriority: Float {
    get {
        return self.view.contentCompressionResistancePriority(for: .vertical).rawValue
    }
    set {
        self.view.setContentCompressionResistancePriority(LayoutPriority(rawValue: newValue), for: .vertical)
    }
}
``` 

## ConstraintLayoutGuideDSL

`ConstraintLayoutGuideDSL` 是一个遵守了 `ConstraintAttributesDSL` 协议的结构体，作用是替 `UILayoutGuide` 提供了一些 `DSLs`，与 `ConstraintViewDSL` 唯一的区别在于，`target` 属性返回的对象不同：

```
public struct ConstraintLayoutGuideDSL: ConstraintAttributesDSL {
    
    ...
    
    public var target: AnyObject? {
        return self.guide
    }
    
    internal let guide: ConstraintLayoutGuide
    
    internal init(guide: ConstraintLayoutGuide) {
        self.guide = guide
        
    }
    
}
```

## ConstraintLayoutSupportDSL

`ConstraintLayoutSupportDSL` 是一个遵守了 `ConstraintDSL` 协议的结构体，作用是替 `ConstraintLayoutSupport` 提供了一些 `DSLs`：

```
@available(iOS 8.0, *)
public struct ConstraintLayoutSupportDSL: ConstraintDSL {
    
    public var target: AnyObject? {
        return self.support
    }
    
    internal let support: ConstraintLayoutSupport
    
    internal init(support: ConstraintLayoutSupport) {
        self.support = support
        
    }
    
    public var top: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.top)
    }
    
    public var bottom: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.bottom)
    }
    
    public var height: ConstraintItem {
        return ConstraintItem(target: self.target, attributes: ConstraintAttributes.height)
    }
}
```

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~