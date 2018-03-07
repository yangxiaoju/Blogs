# 使用 guard 语句提升代码可读性和鲁棒性
guard 语句，就像 if 语句，根据表达式的布尔值执行语句。 你使用 guard 语句来要求条件必须为真，以便执行 guard 语句后的代码。 与 if 语句不同，guard 语句总是有一个 else 子句 - 如果条件不成立，则执行 else 子句中的代码。
```
func greet(person: [String: String]) {
    guard let name = person["name"] else {
        return
    }
    
    print("Hello \(name)!")
    
    guard let location = person["location"] else {
        print("I hope the weather is nice near you.")
        return
    }
    
    print("I hope the weather is nice in \(location).")
}
 
greet(person: ["name": "John"])
// Prints "Hello John!"
// Prints "I hope the weather is nice near you."
greet(person: ["name": "Jane", "location": "Cupertino"])
// Prints "Hello Jane!"
// Prints "I hope the weather is nice in Cupertino."
```
如果符合 guard 语句的条件，则在 guard 语句的大括号之后继续执行代码。 任何使用可选绑定作为条件的一部分赋值的变量或常量都可用于 guard 语句出现的代码块的其余部分。

如果不满足该条件，则执行 else 分支中的代码。 该分支必须转移控制以退出 guard 语句出现的代码块。 它可以使用控制传输语句（如 return，break，continue
 或 throw）来完成此操作，也可以调用不返回的函数或方法，例如 `fatalError(_:file:line:)`。

与使用 if 语句进行同样的检查相比，使用 guard 要求来提高代码的可读性。 它可以让你编写通常被执行的代码，而不用将其包装在一个 else 块中，并且它可以让代码处理违反要求的代码。

在实际项目中，guard 语句最常见的场景应该是配合闭包中的捕获列表，完成 swift 版本的 weak-strong-dance：
```
xxxView.closure = {[weak self] in
    guard let strongSelf = self else { return }
    strongSelf...
}
```
Perfect！

如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~
