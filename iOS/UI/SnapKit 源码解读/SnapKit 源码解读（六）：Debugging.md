# SnapKit 源码解读（六）：Debugging

在开发过程中使用纯代码布局，可能经常性的会遇到一些约束上的问题，有的时候是约束不足，有的时候是多了。这种情况下，`SnapKit` 会为你打印一些信息，来辅助我们排查问题，而实现这些打印信息的功能就在 `Debugging.swift` 中。

## Debugging

想要打印出对象的描述，方法就是 `override` 其 `description` 计算属性。

```
public extension LayoutConstraint {
    
    override public var description: String {
        var description = "<"
        
        description += descriptionForObject(self)
        
        if let firstItem = conditionalOptional(from: self.firstItem) {
            description += " \(descriptionForObject(firstItem))"
        }
        
        if self.firstAttribute != .notAnAttribute {
            description += ".\(descriptionForAttribute(self.firstAttribute))"
        }
        
        description += " \(descriptionForRelation(self.relation))"
        
        if let secondItem = self.secondItem {
            description += " \(descriptionForObject(secondItem))"
        }
        
        if self.secondAttribute != .notAnAttribute {
            description += ".\(descriptionForAttribute(self.secondAttribute))"
        }
        
        if self.multiplier != 1.0 {
            description += " * \(self.multiplier)"
        }
        
        if self.secondAttribute == .notAnAttribute {
            description += " \(self.constant)"
        } else {
            if self.constant > 0.0 {
                description += " + \(self.constant)"
            } else if self.constant < 0.0 {
                description += " - \(abs(self.constant))"
            }
        }
        
        if self.priority.rawValue != 1000.0 {
            description += " ^\(self.priority)"
        }
        
        description += ">"
        
        return description
    }
    
}
```

虽然从代码上看就是简单的不断拼接字符串，但细节实现还是有一个点可以学习一下的。

### 方法重载和泛型

`Swift` 不同于 `Objective-C`，方法可以只有参数不同。与此同时，泛型为我们增加了方法的可扩展性——不再局限于某一种类型。如下两个方法就是用来把可选和非可选类型统一转成可选类型使用的（然而我并不知道为啥要这么做……逃）

```
private func conditionalOptional<T>(from object: Optional<T>) -> Optional<T> {
    return object
}

private func conditionalOptional<T>(from object: T) -> Optional<T> {
    return Optional.some(object)
}
```

就这一个点，完。

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~