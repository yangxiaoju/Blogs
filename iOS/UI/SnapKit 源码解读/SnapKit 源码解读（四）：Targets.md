# SnapKit 源码解读（四）：Targets

`Targets` 是一套协议，为基本数据类型扩充了一些方法，更方便我们进行 `AutoLayout`。

## ConstraintRelatableTarget

`ConstraintRelatableTarget` 是一个空的协议，如下：

```
public protocol ConstraintRelatableTarget {
}
``` 

按我个人理解，它是为了让所有遵守它的协议有了一个类似公共父类的作用，便于进行类型匹配。

```
extension Int: ConstraintRelatableTarget {
}

extension UInt: ConstraintRelatableTarget {
}

extension Float: ConstraintRelatableTarget {
}

extension Double: ConstraintRelatableTarget {
}

extension CGFloat: ConstraintRelatableTarget {
}

extension CGSize: ConstraintRelatableTarget {
}

extension CGPoint: ConstraintRelatableTarget {
}

extension ConstraintInsets: ConstraintRelatableTarget {
}

extension ConstraintItem: ConstraintRelatableTarget {
}

extension ConstraintView: ConstraintRelatableTarget {
}

@available(iOS 9.0, OSX 10.11, *)
extension ConstraintLayoutGuide: ConstraintRelatableTarget {
}
```

## ConstraintConstantTarget

`ConstraintConstantTarget` 也是一个空的协议，遵守它的有 `CGPoint`、`CGSize` 和 `ConstraintInsets` 这三种数据类型，并通过协议扩展为它们增加了一个方法 `constraintConstantTargetValueFor`，用于根据 `LayoutAttribute` 获得组合数据类型中正确的值。

```
extension ConstraintConstantTarget {
    
    internal func constraintConstantTargetValueFor(layoutAttribute: LayoutAttribute) -> CGFloat {
        if let value = self as? CGFloat {
            return value
        }
        
        if let value = self as? Float {
            return CGFloat(value)
        }
        
        if let value = self as? Double {
            return CGFloat(value)
        }
        
        if let value = self as? Int {
            return CGFloat(value)
        }
        
        if let value = self as? UInt {
            return CGFloat(value)
        }
        
        if let value = self as? CGSize {
            if layoutAttribute == .width {
                return value.width
            } else if layoutAttribute == .height {
                return value.height
            } else {
                return 0.0
            }
        }
        
        if let value = self as? CGPoint {
            #if os(iOS) || os(tvOS)
                switch layoutAttribute {
                case .left, .right, .leading, .trailing, .centerX, .leftMargin, .rightMargin, .leadingMargin, .trailingMargin, .centerXWithinMargins:
                    return value.x
                case .top, .bottom, .centerY, .topMargin, .bottomMargin, .centerYWithinMargins, .lastBaseline, .firstBaseline:
                    return value.y
                case .width, .height, .notAnAttribute:
                    return 0.0
                }
            #else
                switch layoutAttribute {
                case .left, .right, .leading, .trailing, .centerX:
                    return value.x
                case .top, .bottom, .centerY, .lastBaseline, .firstBaseline:
                    return value.y
                case .width, .height, .notAnAttribute:
                    return 0.0
                }
            #endif
        }
        
        if let value = self as? ConstraintInsets {
            #if os(iOS) || os(tvOS)
                switch layoutAttribute {
                case .left, .leftMargin, .centerX, .centerXWithinMargins:
                    return value.left
                case .top, .topMargin, .centerY, .centerYWithinMargins, .lastBaseline, .firstBaseline:
                    return value.top
                case .right, .rightMargin:
                    return -value.right
                case .bottom, .bottomMargin:
                    return -value.bottom
                case .leading, .leadingMargin:
                    return (ConstraintConfig.interfaceLayoutDirection == .leftToRight) ? value.left : value.right
                case .trailing, .trailingMargin:
                    return (ConstraintConfig.interfaceLayoutDirection == .leftToRight) ? -value.right : -value.left
                case .width:
                    return -(value.left + value.right)
                case .height:
                    return -(value.top + value.bottom)
                case .notAnAttribute:
                    return 0.0
                }
            #else
                switch layoutAttribute {
                case .left, .centerX:
                    return value.left
                case .top, .centerY, .lastBaseline, .firstBaseline:
                    return value.top
                case .right:
                    return -value.right
                case .bottom:
                    return -value.bottom
                case .leading:
                    return (ConstraintConfig.interfaceLayoutDirection == .leftToRight) ? value.left : value.right
                case .trailing:
                    return (ConstraintConfig.interfaceLayoutDirection == .leftToRight) ? -value.right : -value.left
                case .width:
                    return -(value.left + value.right)
                case .height:
                    return -(value.top + value.bottom)
                case .notAnAttribute:
                    return 0.0
                }
            #endif
        }
        
        return 0.0
    }
    
}
```

## ConstraintPriorityTarget

`ConstraintPriorityTarget` 是一个协议，有一个计算属性的声明：

```
public protocol ConstraintPriorityTarget {
    
    var constraintPriorityTargetValue: Float { get }
    
}
```

作用是把不同类型的值转换为 `Float` 类型，以便供 `priority` 使用。

```
extension Int: ConstraintPriorityTarget {
    
    public var constraintPriorityTargetValue: Float {
        return Float(self)
    }
    
}

extension UInt: ConstraintPriorityTarget {
    
    public var constraintPriorityTargetValue: Float {
        return Float(self)
    }
    
}

extension Float: ConstraintPriorityTarget {
    
    public var constraintPriorityTargetValue: Float {
        return self
    }
    
}

extension Double: ConstraintPriorityTarget {
    
    public var constraintPriorityTargetValue: Float {
        return Float(self)
    }
    
}

extension CGFloat: ConstraintPriorityTarget {
    
    public var constraintPriorityTargetValue: Float {
        return Float(self)
    }
    
}
```

## ConstraintMultiplierTarget

`ConstraintMultiplierTarget` 是一个协议，有一个计算属性的声明：

```
public protocol ConstraintMultiplierTarget {
    
    var constraintMultiplierTargetValue: CGFloat { get }
    
}
```

作用是把不同类型的值转换为 `CGFloat` 类型，以便供 `multiplier` 使用。

extension Int: ConstraintMultiplierTarget {
    
    public var constraintMultiplierTargetValue: CGFloat {
        return CGFloat(self)
    }
    
}

extension UInt: ConstraintMultiplierTarget {
    
    public var constraintMultiplierTargetValue: CGFloat {
        return CGFloat(self)
    }
    
}

extension Float: ConstraintMultiplierTarget {
    
    public var constraintMultiplierTargetValue: CGFloat {
        return CGFloat(self)
    }
    
}

extension Double: ConstraintMultiplierTarget {
    
    public var constraintMultiplierTargetValue: CGFloat {
        return CGFloat(self)
    }
    
}

extension CGFloat: ConstraintMultiplierTarget {
    
    public var constraintMultiplierTargetValue: CGFloat {
        return self
    }
    
}

## ConstraintOffsetTarget

`ConstraintOffsetTarget` 是一个协议，继承自 `ConstraintConstantTarget`：

```
public protocol ConstraintOffsetTarget: ConstraintConstantTarget {
}
```

通过协议扩展为遵守该协议的类型增加了新的方法，作用是把不同类型的值转换为 `CGFloat` 类型，以便供 `offset` 使用。

```
extension Int: ConstraintOffsetTarget {
}

extension UInt: ConstraintOffsetTarget {
}

extension Float: ConstraintOffsetTarget {
}

extension Double: ConstraintOffsetTarget {
}

extension CGFloat: ConstraintOffsetTarget {
}

extension ConstraintOffsetTarget {
    
    internal var constraintOffsetTargetValue: CGFloat {
        let offset: CGFloat
        if let amount = self as? Float {
            offset = CGFloat(amount)
        } else if let amount = self as? Double {
            offset = CGFloat(amount)
        } else if let amount = self as? CGFloat {
            offset = CGFloat(amount)
        } else if let amount = self as? Int {
            offset = CGFloat(amount)
        } else if let amount = self as? UInt {
            offset = CGFloat(amount)
        } else {
            offset = 0.0
        }
        return offset
    }
    
}
```

## ConstraintInsetTarget

`ConstraintOffsetTarget` 是一个协议，继承自 `ConstraintConstantTarget`：

```
public protocol ConstraintInsetTarget: ConstraintConstantTarget {
}
```

通过协议扩展为遵守该协议的类型增加了新的方法，作用是把不同类型的值转换为 `ConstraintInsets` 类型，以便供 `inset` 使用。

```
extension Int: ConstraintInsetTarget {
}

extension UInt: ConstraintInsetTarget {
}

extension Float: ConstraintInsetTarget {
}

extension Double: ConstraintInsetTarget {
}

extension CGFloat: ConstraintInsetTarget {
}

extension ConstraintInsets: ConstraintInsetTarget {
}

extension ConstraintInsetTarget {

    internal var constraintInsetTargetValue: ConstraintInsets {
        if let amount = self as? ConstraintInsets {
            return amount
        } else if let amount = self as? Float {
            return ConstraintInsets(top: CGFloat(amount), left: CGFloat(amount), bottom: CGFloat(amount), right: CGFloat(amount))
        } else if let amount = self as? Double {
            return ConstraintInsets(top: CGFloat(amount), left: CGFloat(amount), bottom: CGFloat(amount), right: CGFloat(amount))
        } else if let amount = self as? CGFloat {
            return ConstraintInsets(top: amount, left: amount, bottom: amount, right: amount)
        } else if let amount = self as? Int {
            return ConstraintInsets(top: CGFloat(amount), left: CGFloat(amount), bottom: CGFloat(amount), right: CGFloat(amount))
        } else if let amount = self as? UInt {
            return ConstraintInsets(top: CGFloat(amount), left: CGFloat(amount), bottom: CGFloat(amount), right: CGFloat(amount))
        } else {
            return ConstraintInsets(top: 0, left: 0, bottom: 0, right: 0)
        }
    }
    
}
```

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~