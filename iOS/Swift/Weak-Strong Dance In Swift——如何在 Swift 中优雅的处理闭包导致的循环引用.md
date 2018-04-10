# Weak-Strong Dance In Swift——如何在 Swift 中优雅的处理闭包导致的循环引用

Objective-C 作为一门资历很老的语言，添加了 Block 这个特性后深受广大 iOS 开发者的喜爱。在 Swift 中，对应的概念叫做 Closure，即闭包。虽然更换了名字，但是概念和用法还是相似的——就算是副作用也一样，有可能导致循环引用。

```
class ClassA {
    var closure: (() -> Void)?
}

class ClassB {
    let classA = ClassA()
    
    init() {
        classA.closure = {
            self.method()
        }
        classA.closure?()
    }
    
    deinit {
        print("deinit")
    }
    
    private func method() {
        print("method")
    }
}

let classB = ClassB()
```
正是因为闭包会导致循环引用，所以在闭包中调用对象内部的方法时，都要显式的使用 `self`

与 Objective-C 不同的是，我们不需要在每一次使用闭包之前再繁琐的写上 `__weak typeof(self) weakSelf = self;` 了，取而代之的是捕获列表的概念：

```   
init() {
    classA.closure = {[weak self] in
        self?.method()
    }
    classA.closure?()
}
```

只要在捕获列表中声明了你想要用弱引用的方式捕获的对象，就可以及时的规避由闭包导致的循环引用了。但是同时可以看到，闭包中对于方法的调用从常规的 `self.method()` 变为了可选链的 `self?.method()`。这是因为假设闭包被放在子线程中执行，而且执行过程中 `self` 在主线程被释放了。由于 `self` 在闭包中成为了一个弱引用，因此会自动变为 `nil`。在 `Swift` 中，可选类型的概念让我们只能以可选链的方式来调用 `method`。

既然知道了 `self` 在闭包中成为了可选类型，那么除了可选链，还可以使用可选绑定来处理可选类型：

```
classA.closure = {[weak self] in
    if let strongSelf = self {
        strongSelf.method()
    } else {
        // 处理 self 被释放时的情况。
    }
}
```

但这样总是会让我们在闭包中的代码多出两句甚至更多，于是还有更优雅的方法，就是使用 `guard` 语句：

```
classA.closure = {[weak self] in
    guard let strongSelf = self else { return } 
    strongSelf.method()
}
```

一句代码搞定~

当然，对于是否每次在类似的场景下都需要进行 Weak-Strong Dance，很多开发者还有疑虑，我个人的建议是多线程下 weakSelf 意外被释放是确实有可能出现的（详情可以去看苹果关于内存管理的相关文档），为了让过自己手的代码健壮性更高，请不要偷懒。

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~