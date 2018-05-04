# SnapKit 源码解读（五）：Models

`Models` 里面的所有文件，都是用来对约束建模使用的。

## Typealiases

`Typealiases` 为跨平台能力定义了一套公用的类。

```
#if os(iOS) || os(tvOS)
    import UIKit
    typealias LayoutRelation = NSLayoutRelation
    typealias LayoutAttribute = NSLayoutAttribute
    typealias LayoutPriority = UILayoutPriority
#else
    import AppKit
    typealias LayoutRelation = NSLayoutConstraint.Relation
    typealias LayoutAttribute = NSLayoutConstraint.Attribute
    typealias LayoutPriority = NSLayoutConstraint.Priority
#endif
```

## Constraint

`Constraint` 类是对 `NSLayoutConstraint` 的建模，其中包含了构建应用所需要的所有必备参数，更新和激活约束的相关方法。

### 初始化方法

```
public final class Constraint {

    internal let sourceLocation: (String, UInt)
    internal let label: String?

    private let from: ConstraintItem
    private let to: ConstraintItem
    private let relation: ConstraintRelation
    private let multiplier: ConstraintMultiplierTarget
    private var constant: ConstraintConstantTarget {
        didSet {
            self.updateConstantAndPriorityIfNeeded()
        }
    }
    private var priority: ConstraintPriorityTarget {
        didSet {
          self.updateConstantAndPriorityIfNeeded()
        }
    }
    public var layoutConstraints: [LayoutConstraint]
    
    public var isActive: Bool {
        for layoutConstraint in self.layoutConstraints {
            if layoutConstraint.isActive {
                return true
            }
        }
        return false
    }
    
    // MARK: Initialization

    internal init(from: ConstraintItem,
                  to: ConstraintItem,
                  relation: ConstraintRelation,
                  sourceLocation: (String, UInt),
                  label: String?,
                  multiplier: ConstraintMultiplierTarget,
                  constant: ConstraintConstantTarget,
                  priority: ConstraintPriorityTarget) {
        self.from = from
        self.to = to
        self.relation = relation
        self.sourceLocation = sourceLocation
        self.label = label
        self.multiplier = multiplier
        self.constant = constant
        self.priority = priority
        self.layoutConstraints = []

        // get attributes
        let layoutFromAttributes = self.from.attributes.layoutAttributes
        let layoutToAttributes = self.to.attributes.layoutAttributes

        // get layout from
        let layoutFrom = self.from.layoutConstraintItem!

        // get relation
        let layoutRelation = self.relation.layoutRelation

        for layoutFromAttribute in layoutFromAttributes {
            // get layout to attribute
            let layoutToAttribute: LayoutAttribute
            #if os(iOS) || os(tvOS)
                if layoutToAttributes.count > 0 {
                    if self.from.attributes == .edges && self.to.attributes == .margins {
                        switch layoutFromAttribute {
                        case .left:
                            layoutToAttribute = .leftMargin
                        case .right:
                            layoutToAttribute = .rightMargin
                        case .top:
                            layoutToAttribute = .topMargin
                        case .bottom:
                            layoutToAttribute = .bottomMargin
                        default:
                            fatalError()
                        }
                    } else if self.from.attributes == .margins && self.to.attributes == .edges {
                        switch layoutFromAttribute {
                        case .leftMargin:
                            layoutToAttribute = .left
                        case .rightMargin:
                            layoutToAttribute = .right
                        case .topMargin:
                            layoutToAttribute = .top
                        case .bottomMargin:
                            layoutToAttribute = .bottom
                        default:
                            fatalError()
                        }
                    } else if self.from.attributes == self.to.attributes {
                        layoutToAttribute = layoutFromAttribute
                    } else {
                        layoutToAttribute = layoutToAttributes[0]
                    }
                } else {
                    if self.to.target == nil && (layoutFromAttribute == .centerX || layoutFromAttribute == .centerY) {
                        layoutToAttribute = layoutFromAttribute == .centerX ? .left : .top
                    } else {
                        layoutToAttribute = layoutFromAttribute
                    }
                }
            #else
                if self.from.attributes == self.to.attributes {
                    layoutToAttribute = layoutFromAttribute
                } else if layoutToAttributes.count > 0 {
                    layoutToAttribute = layoutToAttributes[0]
                } else {
                    layoutToAttribute = layoutFromAttribute
                }
            #endif

            // get layout constant
            let layoutConstant: CGFloat = self.constant.constraintConstantTargetValueFor(layoutAttribute: layoutToAttribute)

            // get layout to
            var layoutTo: AnyObject? = self.to.target

            // use superview if possible
            if layoutTo == nil && layoutToAttribute != .width && layoutToAttribute != .height {
                layoutTo = layoutFrom.superview
            }

            // create layout constraint
            let layoutConstraint = LayoutConstraint(
                item: layoutFrom,
                attribute: layoutFromAttribute,
                relatedBy: layoutRelation,
                toItem: layoutTo,
                attribute: layoutToAttribute,
                multiplier: self.multiplier.constraintMultiplierTargetValue,
                constant: layoutConstant
            )

            // set label
            layoutConstraint.label = self.label

            // set priority
            layoutConstraint.priority = LayoutPriority(rawValue: self.priority.constraintPriorityTargetValue)

            // set constraint
            layoutConstraint.constraint = self

            // append
            self.layoutConstraints.append(layoutConstraint)
        }
    }

    ...

}
```

初始化方法接受所有创建一个 `NSLayoutConstraint` 所需要的参数，最终生成一个 `layoutConstraint` 并放入自身的数组里。

### final

如果你想要一个类不能被其他的类继承，请加上 `final` 标记

### 公共方法

`Constraint` 类提供了一些 `public` 的方法，用来供我们激活/取消激活约束，修改 `constant`。

```
public final class Constraint {

    ...

    public func activate() {
        self.activateIfNeeded()
    }

    public func deactivate() {
        self.deactivateIfNeeded()
    }

    @discardableResult
    public func update(offset: ConstraintOffsetTarget) -> Constraint {
        self.constant = offset.constraintOffsetTargetValue
        return self
    }

    @discardableResult
    public func update(inset: ConstraintInsetTarget) -> Constraint {
        self.constant = inset.constraintInsetTargetValue
        return self
    }

    @discardableResult
    public func update(priority: ConstraintPriorityTarget) -> Constraint {
        self.priority = priority.constraintPriorityTargetValue
        return self
    }

    ...

}
```

### 内部方法

`Constraint` 提供了三个内部的方法，分别用来更新优先级，激活约束，取消激活约束。

```
public final class Constraint {

    ...

    // MARK: Internal

    internal func updateConstantAndPriorityIfNeeded() {
        for layoutConstraint in self.layoutConstraints {
            let attribute = (layoutConstraint.secondAttribute == .notAnAttribute) ? layoutConstraint.firstAttribute : layoutConstraint.secondAttribute
            layoutConstraint.constant = self.constant.constraintConstantTargetValueFor(layoutAttribute: attribute)

            let requiredPriority = ConstraintPriority.required.value
            if (layoutConstraint.priority.rawValue < requiredPriority), (self.priority.constraintPriorityTargetValue != requiredPriority) {
                layoutConstraint.priority = LayoutPriority(rawValue: self.priority.constraintPriorityTargetValue)
            }
        }
    }

    internal func activateIfNeeded(updatingExisting: Bool = false) {
        guard let item = self.from.layoutConstraintItem else {
            print("WARNING: SnapKit failed to get from item from constraint. Activate will be a no-op.")
            return
        }
        let layoutConstraints = self.layoutConstraints

        if updatingExisting {
            var existingLayoutConstraints: [LayoutConstraint] = []
            for constraint in item.constraints {
                existingLayoutConstraints += constraint.layoutConstraints
            }

            for layoutConstraint in layoutConstraints {
                let existingLayoutConstraint = existingLayoutConstraints.first { $0 == layoutConstraint }
                guard let updateLayoutConstraint = existingLayoutConstraint else {
                    fatalError("Updated constraint could not find existing matching constraint to update: \(layoutConstraint)")
                }

                let updateLayoutAttribute = (updateLayoutConstraint.secondAttribute == .notAnAttribute) ? updateLayoutConstraint.firstAttribute : updateLayoutConstraint.secondAttribute
                updateLayoutConstraint.constant = self.constant.constraintConstantTargetValueFor(layoutAttribute: updateLayoutAttribute)
            }
        } else {
            NSLayoutConstraint.activate(layoutConstraints)
            item.add(constraints: [self])
        }
    }

    internal func deactivateIfNeeded() {
        guard let item = self.from.layoutConstraintItem else {
            print("WARNING: SnapKit failed to get from item from constraint. Deactivate will be a no-op.")
            return
        }
        let layoutConstraints = self.layoutConstraints
        NSLayoutConstraint.deactivate(layoutConstraints)
        item.remove(constraints: [self])
    }
}
```

值得一看的是 `activateIfNeeded` 方法，其实就是找出现有的约束，并更新对应的 `constant`。

## ConstraintDescription

`ConstraintDescription` 是内部用来建模 `Constraint` 类型的模型，其中有一个懒加载属性 `constraint`，第一次使用时才会创建真正的 `Constraint` 对象，避免了额外的性能消耗。

```
public class ConstraintDescription {
    
    internal let item: LayoutConstraintItem
    internal var attributes: ConstraintAttributes
    internal var relation: ConstraintRelation? = nil
    internal var sourceLocation: (String, UInt)? = nil
    internal var label: String? = nil
    internal var related: ConstraintItem? = nil
    internal var multiplier: ConstraintMultiplierTarget = 1.0
    internal var constant: ConstraintConstantTarget = 0.0
    internal var priority: ConstraintPriorityTarget = 1000.0
    internal lazy var constraint: Constraint? = {
        guard let relation = self.relation,
              let related = self.related,
              let sourceLocation = self.sourceLocation else {
            return nil
        }
        let from = ConstraintItem(target: self.item, attributes: self.attributes)
        
        return Constraint(
            from: from,
            to: related,
            relation: relation,
            sourceLocation: sourceLocation,
            label: self.label,
            multiplier: self.multiplier,
            constant: self.constant,
            priority: self.priority
        )
    }()
    
    // MARK: Initialization
    
    internal init(item: LayoutConstraintItem, attributes: ConstraintAttributes) {
        self.item = item
        self.attributes = attributes
    }
    
}
```

## ConstraintInsets、ConstraintConfig、ConstraintView、ConstraintLayoutGuide 和 ConstraintLayoutSupport

这五个 `swift` 文件中都是对现有类型的 `typealias`，没有什么特别的。

## ConstraintRelation

`ConstraintRelation` 是一个枚举类型，用来在框架内部表示等于、大于和小于的关系。

```
internal enum ConstraintRelation : Int {
    case equal = 1
    case lessThanOrEqual
    case greaterThanOrEqual
    
    internal var layoutRelation: LayoutRelation {
        get {
            switch(self) {
            case .equal:
                return .equal
            case .lessThanOrEqual:
                return .lessThanOrEqual
            case .greaterThanOrEqual:
                return .greaterThanOrEqual
            }
        }
    }
}
```

## ConstraintAttributes

`ConstraintAttributes` 也是枚举，不过与寻常用到的枚举不同，它是可以组合而非非此即彼的枚举，即位掩码。

### OptionSet

`Swift` 中给出的位掩码解决方案，是通过 `struct` 遵守 `OptionSet` 协议来实现的，为了与枚举类似，`ConstraintAttributes` 也定义了一个内部只读属性来作为 `rawValue`。

```
internal struct ConstraintAttributes : OptionSet {
    
    internal init(rawValue: UInt) {
        self.rawValue = rawValue
    }
    internal init(_ rawValue: UInt) {
        self.init(rawValue: rawValue)
    }
    internal init(nilLiteral: ()) {
        self.rawValue = 0
    }
    
    internal private(set) var rawValue: UInt
    ...
}

```

同时，为了方便使用，还定义了一批计算属性供我们快速创建和使用对应的枚举：

```
internal struct ConstraintAttributes : OptionSet {
    
    ...
    // normal
    
    internal static var none: ConstraintAttributes { return self.init(0) }
    internal static var left: ConstraintAttributes { return self.init(1) }
    internal static var top: ConstraintAttributes {  return self.init(2) }
    internal static var right: ConstraintAttributes { return self.init(4) }
    internal static var bottom: ConstraintAttributes { return self.init(8) }
    internal static var leading: ConstraintAttributes { return self.init(16) }
    internal static var trailing: ConstraintAttributes { return self.init(32) }
    internal static var width: ConstraintAttributes { return self.init(64) }
    internal static var height: ConstraintAttributes { return self.init(128) }
    internal static var centerX: ConstraintAttributes { return self.init(256) }
    internal static var centerY: ConstraintAttributes { return self.init(512) }
    internal static var lastBaseline: ConstraintAttributes { return self.init(1024) }
    
    @available(iOS 8.0, OSX 10.11, *)
    internal static var firstBaseline: ConstraintAttributes { return self.init(2048) }
    
    @available(iOS 8.0, *)
    internal static var leftMargin: ConstraintAttributes { return self.init(4096) }
    
    @available(iOS 8.0, *)
    internal static var rightMargin: ConstraintAttributes { return self.init(8192) }
    
    @available(iOS 8.0, *)
    internal static var topMargin: ConstraintAttributes { return self.init(16384) }
    
    @available(iOS 8.0, *)
    internal static var bottomMargin: ConstraintAttributes { return self.init(32768) }
    
    @available(iOS 8.0, *)
    internal static var leadingMargin: ConstraintAttributes { return self.init(65536) }
    
    @available(iOS 8.0, *)
    internal static var trailingMargin: ConstraintAttributes { return self.init(131072) }
    
    @available(iOS 8.0, *)
    internal static var centerXWithinMargins: ConstraintAttributes { return self.init(262144) }
    
    @available(iOS 8.0, *)
    internal static var centerYWithinMargins: ConstraintAttributes { return self.init(524288) }
    
    // aggregates
    
    internal static var edges: ConstraintAttributes { return self.init(15) }
    internal static var size: ConstraintAttributes { return self.init(192) }
    internal static var center: ConstraintAttributes { return self.init(768) }
    
    @available(iOS 8.0, *)
    internal static var margins: ConstraintAttributes { return self.init(61440) }
    
    @available(iOS 8.0, *)
    internal static var centerWithinMargins: ConstraintAttributes { return self.init(786432) }

    ...

}
```

每一个计算属性初始化对应枚举的时候，`rawValue` 都是 2 的 n 次方，也是为了避免运算时出现冲突。

### 自定义运算符

`Swift` 可以自定义运算符，来简化复杂的方法调用。

```
internal func + (left: ConstraintAttributes, right: ConstraintAttributes) -> ConstraintAttributes {
    return left.union(right)
}

internal func +=(left: inout ConstraintAttributes, right: ConstraintAttributes) {
    left.formUnion(right)
}

internal func -=(left: inout ConstraintAttributes, right: ConstraintAttributes) {
    left.subtract(right)
}

internal func ==(left: ConstraintAttributes, right: ConstraintAttributes) -> Bool {
    return left.rawValue == right.rawValue
}
```

## ConstraintItem

`ConstraintItem` 建模了 `target` 和 `attributes` 之间的关系，同样定义了自定义运算符，用来判断相等性。

```
public final class ConstraintItem {
    
    internal weak var target: AnyObject?
    internal let attributes: ConstraintAttributes
    
    internal init(target: AnyObject?, attributes: ConstraintAttributes) {
        self.target = target
        self.attributes = attributes
    }
    
    internal var layoutConstraintItem: LayoutConstraintItem? {
        return self.target as? LayoutConstraintItem
    }
    
}

public func ==(lhs: ConstraintItem, rhs: ConstraintItem) -> Bool {
    // pointer equality
    guard lhs !== rhs else {
        return true
    }
    
    // must both have valid targets and identical attributes
    guard let target1 = lhs.target,
          let target2 = rhs.target,
          target1 === target2 && lhs.attributes == rhs.attributes else {
            return false
    }
    
    return true
}
```

## LayoutConstraint

`LayoutConstraint` 是 `NSLayoutConstraint` 的子类，做了两件事情：1、定义了新的计算属性 `label`，其实是对 `identifier` 的别称。2、自定义了预算符，用来判断约束之间的相等性。

```
public class LayoutConstraint : NSLayoutConstraint {
    
    public var label: String? {
        get {
            return self.identifier
        }
        set {
            self.identifier = newValue
        }
    }
    
    internal weak var constraint: Constraint? = nil
    
}

internal func ==(lhs: LayoutConstraint, rhs: LayoutConstraint) -> Bool {
    guard lhs.firstItem === rhs.firstItem &&
          lhs.secondItem === rhs.secondItem &&
          lhs.firstAttribute == rhs.firstAttribute &&
          lhs.secondAttribute == rhs.secondAttribute &&
          lhs.relation == rhs.relation &&
          lhs.priority == rhs.priority &&
          lhs.multiplier == rhs.multiplier else {
        return false
    }
    return true
}
```

## LayoutConstraintItem

`LayoutConstraintItem` 是一个仅供 `class` 使用的协议，继承这个协议的有 `ConstraintLayoutGuide` 和 `ConstraintView`，目的在于为这两者通过协议扩展提供一些方法和关联对象。

```
public protocol LayoutConstraintItem: class {
}

@available(iOS 9.0, OSX 10.11, *)
extension ConstraintLayoutGuide : LayoutConstraintItem {
}

extension ConstraintView : LayoutConstraintItem {
}


extension LayoutConstraintItem {
    
    internal func prepare() {
        if let view = self as? ConstraintView {
            view.translatesAutoresizingMaskIntoConstraints = false
        }
    }
    
    internal var superview: ConstraintView? {
        if let view = self as? ConstraintView {
            return view.superview
        }
        
        if #available(iOS 9.0, OSX 10.11, *), let guide = self as? ConstraintLayoutGuide {
            return guide.owningView
        }
        
        return nil
    }
    internal var constraints: [Constraint] {
        return self.constraintsSet.allObjects as! [Constraint]
    }
    
    internal func add(constraints: [Constraint]) {
        let constraintsSet = self.constraintsSet
        for constraint in constraints {
            constraintsSet.add(constraint)
        }
    }
    
    internal func remove(constraints: [Constraint]) {
        let constraintsSet = self.constraintsSet
        for constraint in constraints {
            constraintsSet.remove(constraint)
        }
    }
    
    private var constraintsSet: NSMutableSet {
        let constraintsSet: NSMutableSet
        
        if let existing = objc_getAssociatedObject(self, &constraintsKey) as? NSMutableSet {
            constraintsSet = existing
        } else {
            constraintsSet = NSMutableSet()
            objc_setAssociatedObject(self, &constraintsKey, constraintsSet, .OBJC_ASSOCIATION_RETAIN_NONATOMIC)
        }
        return constraintsSet
        
    }
    
}
private var constraintsKey: UInt8 = 0
``` 
 
其中 `constraintsSet` 是管理施加在 `ConstraintLayoutGuide` 和 `ConstraintView` 上的约束，便于更新约束。

## ConstraintPriority

`ConstraintPriority` 是对约束优先级进行定义的一个结构体并提供了判断相等性和差距的方便方法，唯一需要在意的是在 `OS X` 平台上 `medium` 的定义是 501，不知道为啥~

```
public struct ConstraintPriority : ExpressibleByFloatLiteral, Equatable, Strideable {
    public typealias FloatLiteralType = Float
    
    public let value: Float
    
    public init(floatLiteral value: Float) {
        self.value = value
    }
    
    public init(_ value: Float) {
        self.value = value
    }
    
    public static var required: ConstraintPriority {
        return 1000.0
    }
    
    public static var high: ConstraintPriority {
        return 750.0
    }
    
    public static var medium: ConstraintPriority {
        #if os(OSX)
            return 501.0
        #else
            return 500.0
        #endif
        
    }
    
    public static var low: ConstraintPriority {
        return 250.0
    }
    
    public static func ==(lhs: ConstraintPriority, rhs: ConstraintPriority) -> Bool {
        return lhs.value == rhs.value
    }

    // MARK: Strideable

    public func advanced(by n: FloatLiteralType) -> ConstraintPriority {
        return ConstraintPriority(floatLiteral: value + n)
    }

    public func distance(to other: ConstraintPriority) -> FloatLiteralType {
        return other.value - value
    }
}
```

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~