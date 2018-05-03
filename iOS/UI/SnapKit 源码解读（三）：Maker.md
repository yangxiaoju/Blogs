# SnapKit 源码解读（三）：Maker

`Maker` 是 `SnapKit` 中最核心的概念，所有关于约束的操作都是通过 `Maker` 来进行管理和操作的。

## ConstraintMaker

`ConstraintMaker` 是施加、更新和重置约束三个功能的核心类，与之对应的也实现了相关的类方法。同时其还声明了很多计算属性，用来配置约束使用。

### 初始化方法

`ConstraintMaker` 是通过 `LayoutConstraintItem` 类型的 `item` 参数进行初始化的，`LayoutConstraintItem` 是一个模型类，保存了所有的约束。
 
```
private let item: LayoutConstraintItem
    
internal init(item: LayoutConstraintItem) {
    self.item = item
    self.item.prepare()
}
```

### 计算属性

`ConstraintMaker` 内部有很多的计算属性，用以辅助描述约束。

```
public var left: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.left)
}
    
public var top: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.top)
}
    
public var bottom: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.bottom)
}
    
public var right: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.right)
}
    
public var leading: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.leading)
}
    
public var trailing: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.trailing)
}
    
public var width: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.width)
}
    
public var height: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.height)
}
  
public var centerX: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.centerX)
}
   
public var centerY: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.centerY)
}
  
@available(*, deprecated:3.0, message:"Use lastBaseline instead")
public var baseline: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.lastBaseline)
}
    
public var lastBaseline: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.lastBaseline)
}
    
@available(iOS 8.0, OSX 10.11, *)
public var firstBaseline: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.firstBaseline)
}
    
@available(iOS 8.0, *)
public var leftMargin: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.leftMargin)
}
    
@available(iOS 8.0, *)
public var rightMargin: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.rightMargin)
}
    
@available(iOS 8.0, *)
public var topMargin: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.topMargin)
}
    
@available(iOS 8.0, *)
public var bottomMargin: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.bottomMargin)
}
    
@available(iOS 8.0, *)
public var leadingMargin: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.leadingMargin)
}
    
@available(iOS 8.0, *)
public var trailingMargin: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.trailingMargin)
}
    
@available(iOS 8.0, *)
public var centerXWithinMargins: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.centerXWithinMargins)
}
    
@available(iOS 8.0, *)
public var centerYWithinMargins: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.centerYWithinMargins)
}
    
public var edges: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.edges)
}
public var size: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.size)
}
public var center: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.center)
}
    
@available(iOS 8.0, *)
public var margins: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.margins)
}
    
@available(iOS 8.0, *)
public var centerWithinMargins: ConstraintMakerExtendable {
    return self.makeExtendableWithAttributes(.centerWithinMargins)
}
```

所有的这些计算属性都会通过内部的一个 `makeExtendableWithAttributes` 方法初始化一个 `ConstraintMakerExtendable` 类型的对象，并返回之，`makeExtendableWithAttributes` 方法如下所示：

```
private var descriptions = [ConstraintDescription]()

internal func makeExtendableWithAttributes(_ attributes: ConstraintAttributes) -> ConstraintMakerExtendable {
    let description = ConstraintDescription(item: self.item, attributes: attributes)
    self.descriptions.append(description)
    return ConstraintMakerExtendable(description)
}
```

该方法会根据 `self.item` 初始化一个 `ConstraintDescription` 对象，并将该对象存储在 `descriptions` 数组中，再创造一个 `ConstraintMakerExtendable` 类型的对象并返回之。

### prepareConstraints

`prepareConstraints` 方法会先生成一个 `ConstraintMaker` 类型的对象 `maker`，然后将 `maker` 通过 `closure` 传递给外界做一些属性的配置，接着遍历 `maker.descriptions` 取出其 `constraint` 属性，并返回之。

```
internal static func prepareConstraints(item: LayoutConstraintItem, closure: (_ make: ConstraintMaker) -> Void) -> [Constraint] {
    let maker = ConstraintMaker(item: item)
    closure(maker)
    var constraints: [Constraint] = []
    for description in maker.descriptions {
        guard let constraint = description.constraint else {
            continue
        }
        constraints.append(constraint)
    }
    return constraints
}
```

### makeConstraints

`makeConstraints` 方法与 `prepareConstraints` 方法大同小异，区别在于，`makeConstraints` 方法的最后不是 `return constraints`，而是挨个 `constraint.activateIfNeeded(updatingExisting: false)`。

```
internal static func makeConstraints(item: LayoutConstraintItem, closure: (_ make: ConstraintMaker) -> Void) {
    let maker = ConstraintMaker(item: item)
    closure(maker)
    var constraints: [Constraint] = []
    for description in maker.descriptions {
        guard let constraint = description.constraint else {
            continue
        }
        constraints.append(constraint)
    }
    for constraint in constraints {
        constraint.activateIfNeeded(updatingExisting: false)
    }
}
```

### remakeConstraints

移除约束的操作分为两步，第一步：移除现有约束，第二步：重新添加约束。

```
internal static func remakeConstraints(item: LayoutConstraintItem, closure: (_ make: ConstraintMaker) -> Void) {
    self.removeConstraints(item: item)
    self.makeConstraints(item: item, closure: closure)
}
```

移除约束操作就是从 `item` 的 `constraints` 属性里所有 `constraint` 挨个失效。

```
internal static func removeConstraints(item: LayoutConstraintItem) {
    let constraints = item.constraints
    for constraint in constraints {
        constraint.deactivateIfNeeded()
    }
}
```

### updateConstraints

更新约束的操作也分为两部分：首先会判断当前是否已经施加过约束，如果没有，则直接转入 `makeConstraints` 流程。其余操作与 `makeConstraints` 类似，区别就是 `updatingExisting` 参数为 `true`。

```
internal static func updateConstraints(item: LayoutConstraintItem, closure: (_ make: ConstraintMaker) -> Void) {
    guard item.constraints.count > 0 else {
        self.makeConstraints(item: item, closure: closure)
        return
    }
        
    let maker = ConstraintMaker(item: item)
    closure(maker)
    var constraints: [Constraint] = []
    for description in maker.descriptions {
        guard let constraint = description.constraint else {
            continue
        }
        constraints.append(constraint)
    }
    for constraint in constraints {
        constraint.activateIfNeeded(updatingExisting: true)
    }
}
```

## ConstraintMakerFinalizable

`ConstraintMakerFinalizable` 是对 `ConstraintDescription` 的一层包装，并提供了一些方便方法供我们设置和获取一些值。

```
public class ConstraintMakerFinalizable {
    
    internal let description: ConstraintDescription
    
    internal init(_ description: ConstraintDescription) {
        self.description = description
    }
    
    @discardableResult
    public func labeled(_ label: String) -> ConstraintMakerFinalizable {
        self.description.label = label
        return self
    }
    
    public var constraint: Constraint {
        return self.description.constraint!
    }
    
}
```

其中，`labeled` 方法的最后 `return self`，是为了可以实现链式调用。

## ConstraintMakerPriortizable

`ConstraintMakerPriortizable` 是 `ConstraintMakerFinalizable` 的子类，扩充了关于优先级的一些方法。

```
public class ConstraintMakerPriortizable: ConstraintMakerFinalizable {
    
    @discardableResult
    public func priority(_ amount: ConstraintPriority) -> ConstraintMakerFinalizable {
        self.description.priority = amount.value
        return self
    }
    
    @discardableResult
    public func priority(_ amount: ConstraintPriorityTarget) -> ConstraintMakerFinalizable {
        self.description.priority = amount
        return self
    }

}
```

## ConstraintMakerEditable

`ConstraintMakerEditable` 是 `ConstraintMakerPriortizable` 的子类，扩充了关于设置 `multiplier` 和 `constant` 的方法。

``` 
public class ConstraintMakerEditable: ConstraintMakerPriortizable {

    @discardableResult
    public func multipliedBy(_ amount: ConstraintMultiplierTarget) -> ConstraintMakerEditable {
        self.description.multiplier = amount
        return self
    }
    
    @discardableResult
    public func dividedBy(_ amount: ConstraintMultiplierTarget) -> ConstraintMakerEditable {
        return self.multipliedBy(1.0 / amount.constraintMultiplierTargetValue)
    }
    
    @discardableResult
    public func offset(_ amount: ConstraintOffsetTarget) -> ConstraintMakerEditable {
        self.description.constant = amount.constraintOffsetTargetValue
        return self
    }
    
    @discardableResult
    public func inset(_ amount: ConstraintInsetTarget) -> ConstraintMakerEditable {
        self.description.constant = amount.constraintInsetTargetValue
        return self
    }
    
}
```

## ConstraintMakerRelatable

`ConstraintMakerRelatable` 是对 `ConstraintDescription` 的一层包装，作用是为 `ConstraintDescription` 设置 `relation` 提供方便方法。有 `equalTo`、`equalToSuperview`、`lessThanOrEqualTo`、`lessThanOrEqualToSuperview`、`greaterThanOrEqualTo` 和 `greaterThanOrEqualToSuperview` 等一系列方法：

```
@discardableResult
public func equalTo(_ other: ConstraintRelatableTarget, _ file: String = #file, _ line: UInt = #line) -> ConstraintMakerEditable {
    return self.relatedTo(other, relation: .equal, file: file, line: line)
}
    
@discardableResult
public func equalToSuperview(_ file: String = #file, _ line: UInt = #line) -> ConstraintMakerEditable {
    guard let other = self.description.item.superview else {
        fatalError("Expected superview but found nil when attempting make constraint `equalToSuperview`.")
    }
    return self.relatedTo(other, relation: .equal, file: file, line: line)
}
    
@discardableResult
public func lessThanOrEqualTo(_ other: ConstraintRelatableTarget, _ file: String = #file, _ line: UInt = #line) -> ConstraintMakerEditable {
    return self.relatedTo(other, relation: .lessThanOrEqual, file: file, line: line)
}
    
@discardableResult
public func lessThanOrEqualToSuperview(_ file: String = #file, _ line: UInt = #line) -> ConstraintMakerEditable {
    guard let other = self.description.item.superview else {
        fatalError("Expected superview but found nil when attempting make constraint `lessThanOrEqualToSuperview`.")
    }
    return self.relatedTo(other, relation: .lessThanOrEqual, file: file, line: line)
}
    
@discardableResult
public func greaterThanOrEqualTo(_ other: ConstraintRelatableTarget, _ file: String = #file, line: UInt = #line) -> ConstraintMakerEditable {
    return self.relatedTo(other, relation: .greaterThanOrEqual, file: file, line: line)
}
    
@discardableResult
public func greaterThanOrEqualToSuperview(_ file: String = #file, line: UInt = #line) -> ConstraintMakerEditable {
    guard let other = self.description.item.superview else {
        fatalError("Expected superview but found nil when attempting make constraint `greaterThanOrEqualToSuperview`.")
    }
    return self.relatedTo(other, relation: .greaterThanOrEqual, file: file, line: line)
}
```

其中有两个点值得学习。

### #file 和 #line

`Swift` 为我们提供了设置方法参数默认值的方法，就是在方法声明处直接为参数赋值：

```
..._ file: String = #file, _ line: UInt = #line...
```

而 `#file` 和 `#line` 则是系统为我们提供的编译符号，供我们获取文件名和行号，更多细节可以参考这篇博客：[LOG 输出](http://swifter.tips/log/)

### fatalError()

有的时候，该崩还是得崩，`fatalError()` 助你一崩到底，还能添加附加信息，哈啤。

除了以上两点，所有这些方法均调用了内部的 `relatedTo` 方法：

```
internal func relatedTo(_ other: ConstraintRelatableTarget, relation: ConstraintRelation, file: String, line: UInt) -> ConstraintMakerEditable {
    let related: ConstraintItem
    let constant: ConstraintConstantTarget
        
    if let other = other as? ConstraintItem {
        guard other.attributes == ConstraintAttributes.none ||
                other.attributes.layoutAttributes.count <= 1 ||
                other.attributes.layoutAttributes == self.description.attributes.layoutAttributes ||
                other.attributes == .edges && self.description.attributes == .margins ||
                other.attributes == .margins && self.description.attributes == .edges else {
            fatalError("Cannot constraint to multiple non identical attributes. (\(file), \(line))");
        }
            
        related = other
        constant = 0.0
    } else if let other = other as? ConstraintView {
        related = ConstraintItem(target: other, attributes: ConstraintAttributes.none)
        constant = 0.0
    } else if let other = other as? ConstraintConstantTarget {
        related = ConstraintItem(target: nil, attributes: ConstraintAttributes.none)
        constant = other
    } else if #available(iOS 9.0, OSX 10.11, *), let other = other as? ConstraintLayoutGuide {
        related = ConstraintItem(target: other, attributes: ConstraintAttributes.none)
        constant = 0.0
    } else {
        fatalError("Invalid constraint. (\(file), \(line))")
    }
        
    let editable = ConstraintMakerEditable(self.description)
    editable.description.sourceLocation = (file, line)
    editable.description.relation = relation
    editable.description.related = related
    editable.description.constant = constant
    return editable
}
```

该方法会对约束信息的合法性进行一些判断，进而生成一个 `ConstraintMakerEditable` 类型的对象并返回之。

## ConstraintMakerExtendable

`ConstraintMakerExtendable` 是 `ConstraintMakerRelatable` 的父类，封装了一系列的计算属性，并有 `side-effect` 每次调用时对 `description.attributes` 的会有一些额外效果：

```
public class ConstraintMakerExtendable: ConstraintMakerRelatable {
    
    public var left: ConstraintMakerExtendable {
        self.description.attributes += .left
        return self
    }
    
    public var top: ConstraintMakerExtendable {
        self.description.attributes += .top
        return self
    }
    
    public var bottom: ConstraintMakerExtendable {
        self.description.attributes += .bottom
        return self
    }
    
    public var right: ConstraintMakerExtendable {
        self.description.attributes += .right
        return self
    }
    
    public var leading: ConstraintMakerExtendable {
        self.description.attributes += .leading
        return self
    }
    
    public var trailing: ConstraintMakerExtendable {
        self.description.attributes += .trailing
        return self
    }
    
    public var width: ConstraintMakerExtendable {
        self.description.attributes += .width
        return self
    }
    
    public var height: ConstraintMakerExtendable {
        self.description.attributes += .height
        return self
    }
    
    public var centerX: ConstraintMakerExtendable {
        self.description.attributes += .centerX
        return self
    }
    
    public var centerY: ConstraintMakerExtendable {
        self.description.attributes += .centerY
        return self
    }
    
    @available(*, deprecated:3.0, message:"Use lastBaseline instead")
    public var baseline: ConstraintMakerExtendable {
        self.description.attributes += .lastBaseline
        return self
    }
    
    public var lastBaseline: ConstraintMakerExtendable {
        self.description.attributes += .lastBaseline
        return self
    }
    
    @available(iOS 8.0, OSX 10.11, *)
    public var firstBaseline: ConstraintMakerExtendable {
        self.description.attributes += .firstBaseline
        return self
    }
    
    @available(iOS 8.0, *)
    public var leftMargin: ConstraintMakerExtendable {
        self.description.attributes += .leftMargin
        return self
    }
    
    @available(iOS 8.0, *)
    public var rightMargin: ConstraintMakerExtendable {
        self.description.attributes += .rightMargin
        return self
    }
    
    @available(iOS 8.0, *)
    public var topMargin: ConstraintMakerExtendable {
        self.description.attributes += .topMargin
        return self
    }
    
    @available(iOS 8.0, *)
    public var bottomMargin: ConstraintMakerExtendable {
        self.description.attributes += .bottomMargin
        return self
    }
    
    @available(iOS 8.0, *)
    public var leadingMargin: ConstraintMakerExtendable {
        self.description.attributes += .leadingMargin
        return self
    }
    
    @available(iOS 8.0, *)
    public var trailingMargin: ConstraintMakerExtendable {
        self.description.attributes += .trailingMargin
        return self
    }
    
    @available(iOS 8.0, *)
    public var centerXWithinMargins: ConstraintMakerExtendable {
        self.description.attributes += .centerXWithinMargins
        return self
    }
    
    @available(iOS 8.0, *)
    public var centerYWithinMargins: ConstraintMakerExtendable {
        self.description.attributes += .centerYWithinMargins
        return self
    }
    
    public var edges: ConstraintMakerExtendable {
        self.description.attributes += .edges
        return self
    }
    public var size: ConstraintMakerExtendable {
        self.description.attributes += .size
        return self
    }
    
    @available(iOS 8.0, *)
    public var margins: ConstraintMakerExtendable {
        self.description.attributes += .margins
        return self
    }
    
    @available(iOS 8.0, *)
    public var centerWithinMargins: ConstraintMakerExtendable {
        self.description.attributes += .centerWithinMargins
        return self
    }
    
}
```

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~