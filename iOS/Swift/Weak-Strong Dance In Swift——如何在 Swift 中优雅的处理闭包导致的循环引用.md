# Weak-Strong Dance In Swift——如何在 Swift 中优雅的处理闭包导致的循环引用

Objective-C 作为一门资历很老的语言，添加了 Block 这个特性后深受广大 iOS 开发者的喜爱。在 Swift 中，对应的概念叫做 Closure，即闭包。虽然更换了名字，但是概念和用法还是相似的，就算是副作用也一样，有可能导致循环引用。

下面我们用一个例子看一下，首先我们需要第一个控制器（`FirstViewController`），它所做的就是简单的推出第二个控制器（`SecondViewController`）。

```
class FirstViewController: UIViewController {
    
    private let button: UIButton = {
        let button = UIButton()
        button.setTitleColor(UIColor.black, for: .normal)
        button.setTitle("跳转到 SecondViewController", for: .normal)
        button.sizeToFit()
        return button
    }()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        button.center = view.center
        view.addSubview(button)
        button.addTarget(self, action: #selector(buttonClick), for: .touchUpInside)
    }
    
    @objc private func buttonClick() {
        let secondViewController = SecondViewController()        
        navigationController?.pushViewController(secondViewController, animated: true)
    }
}
```

下面是 `SecondViewController` 的代码。`SecondViewController` 所做的事情是推出第三个控制器（`ThirdViewController`），不同的是，`thirdViewController` 是作为一个属性存在的，同时它还有一个闭包 `closure` ，这是我们用来测试循环引用问题的。还实现了 `deinit` 方法，用来打印一条语句，看该控制器是否被释放了。

```
class SecondViewController: UIViewController {
    
    private let thirdViewController = ThirdViewController()
    private let button: UIButton = {
        let button = UIButton()
        button.setTitleColor(UIColor.black, for: .normal)
        button.setTitle("跳转到 ThirdViewController", for: .normal)
        button.sizeToFit()
        return button
    }()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        button.center = view.center
        view.addSubview(button)
        button.addTarget(self, action: #selector(buttonClick), for: .touchUpInside)
    }
    
    deinit {
        print("SecondViewController-被释放了")
    }
    
    @objc private func buttonClick() {
        thirdViewController.closure = {
            self.test()
        }
        navigationController?.pushViewController(thirdViewController, animated: true)
    }
    
    private func test() {
        print("调用 test 方法")
    }
}

```

接下来我们看一下 `ThirdViewController` 的代码。在 `ThirdViewController` 中有一个按钮，点击一下就会触发闭包。同时我们还实现了 `deinit` 方法，用来打印一条语句，看该控制器是否被释放了。

```
class ThirdViewController: UIViewController {
    
    private let button: UIButton = {
        let button = UIButton()
        button.setTitleColor(UIColor.black, for: .normal)
        button.setTitle("点击按钮", for: .normal)
        button.sizeToFit()
        return button
    }()
    
    var closure: (() -> Void)?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        button.center = view.center
        view.addSubview(button)
        button.addTarget(self, action: #selector(buttonClick), for: .touchUpInside)
    }
    
    deinit {
        print("ThirdViewController-被释放了")
    }
    
    @objc private func buttonClick() {
        closure?()
    }
}

```

当我们连续推到第三个控制器，点击按钮（触发闭包）后，再回到第一个控制器，看一下三个控制器的生命周期。当流程走完后，发现控制台只有一条语句：

```
调用 test 方法
```

这说明闭包已经引起了循环引用问题，导致第二个控制器没能被释放（内存泄漏）。正是因为闭包会导致循环引用，所以在闭包中调用对象内部的方法时，都要显式的使用 `self`，提醒我们要注意可能引起的内存泄漏问题。与 `Objective-C` 不同的是，我们不需要在每一次使用闭包之前再繁琐的写上 `__weak typeof(self) weakSelf = self;` 了，取而代之的是捕获列表的概念：

```   
@objc private func buttonClick() { 
    thirdViewController.closure = { [weak self] in 
        self?.test()
    }
    navigationController?.pushViewController(thirdViewController, animated: true)
}
```

再重复一次上面的流程，可以看到控制台多了两条语句：

```
调用 test 方法
SecondViewController-被释放了
ThirdViewController-被释放了
```

只要在捕获列表中声明了你想要用弱引用的方式捕获的对象，就可以及时的规避由闭包导致的循环引用了。但是同时可以看到，闭包中对于方法的调用从常规的 `self.test()` 变为了可选链的 `self?.test()`。这是因为假设闭包在子线程中执行，执行过程中 `self` 在主线程随时有可能被释放。由于 `self` 在闭包中成为了一个弱引用，因此会自动变为 `nil`。在 `Swift` 中，可选类型的概念让我们只能以可选链的方式来调用 `test`。下面修改一下 `ThirdViewController` 中的代码：

```
@objc private func buttonClick() {
    // 模拟网络请求
    DispatchQueue.global().asyncAfter(deadline: DispatchTime.now() + 5) {
        self.closure?()
    }
}
```

再次执行相同的操作步骤，这次我们发现 `test` 方法没能正确的得到调用：

```
SecondViewController-被释放了
ThirdViewController-被释放了
```

在实际的项目中，这可能会导致一些问题，闭包中捕获的 `self` 是 `weak` 的，有可能在闭包执行的过程中就被释放了，导致闭包中的一部分方法被执行了而一部分没有，应用的状态因此变得不一致。于是这个时候就要用到 `Weak-Strong Dance` 了。

既然知道了 `self` 在闭包中成为了可选类型，那么除了可选链，还可以使用可选绑定来处理可选类型：

```
@objc private func buttonClick() { 
    thirdViewController.closure = { [weak self] in 
        if let strongSelf = self {
            strongSelf.test()
        } else {
            // 处理 self 被释放时的情况。
        }
    }
    navigationController?.pushViewController(thirdViewController, animated: true)
}
```

但这样总是会让我们在闭包中的代码多出两句甚至更多，于是还有更优雅的方法，就是使用 `guard` 语句：

```
@objc private func buttonClick() { 
    thirdViewController.closure = { [weak self] in 
        guard let strongSelf = self else { return } 
        strongSelf.test()
    }
    navigationController?.pushViewController(thirdViewController, animated: true)
}
```

一句代码搞定~

当然，有人看到这里会说，每次都要使用 `strongSelf` 来调用 `self` 的方法，好烦啊……那么这一点还是可以进一步被优化的，`Swift` 与 `Objective-C` 不同，是可以使用部分关键字来声明变量的，于是我们可以：

```
@objc private func buttonClick() { 
    thirdViewController.closure = { [weak self] in 
        guard let `self` = self else { return } 
        self.test()
    }
    navigationController?.pushViewController(thirdViewController, animated: true)
}
```

这样就可以避免每次书写 `strongSelf` 的烦躁感了~

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~