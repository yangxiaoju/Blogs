# 使用自动闭包（autoclosures）简化 API 语法并推迟代码执行时间
autoclosure 是一个自动创建的闭包，用于封装作为参数传递给函数的表达式。 它不需要任何参数，当它被调用时，它会返回包装在其中的表达式的值。 这种语法上的便利可以让你通过写一个普通的表达式而不是显式的闭包来省略函数参数的大括号。

通常调用采用自动闭包的函数，但实现这种功能并不常见。 例如，`assert(condition:message:file:line:)` 函数为其条件和消息参数采用 autoclosure; 其条件参数仅在 debug 版本中评估，并且仅当条件为假时才执行其消息参数。

autoclosure 让你延迟调用，因为在你调用闭包之前，里面的代码不会运行。 延迟调用对于有 side-effect 或计算成本较高的代码非常有用，因为它可以让你控制代码的执行时间。 下面的代码显示了闭包延迟执行的方式。
```
var customersInLine = ["Chris", "Alex", "Ewa", "Barry", "Daniella"]
print(customersInLine.count)
// Prints "5"
 
let customerProvider = { customersInLine.remove(at: 0) }
print(customersInLine.count)
// Prints "5"
 
print("Now serving \(customerProvider())!")
// Prints "Now serving Chris!"
print(customersInLine.count)
// Prints "4"
```
尽管 `customersInLine` 数组的第一个元素被闭包中的代码移除，但在实际调用闭包之前，数组元素不会被删除。 如果闭包永远不会被调用，闭包内的表达式永远不会被计算，这意味着数组元素永远不会被移除。 请注意，`customerProvider` 的类型不是`String`，而是 `() -> String` - 不带参数返回字符串的函数。

当你将闭包作为参数传递给函数时，你会得到延迟计算的相同行为。
```
// customersInLine is ["Alex", "Ewa", "Barry", "Daniella"]
func serve(customer customerProvider: () -> String) {
    print("Now serving \(customerProvider())!")
}
serve(customer: { customersInLine.remove(at: 0) } )
// Prints "Now serving Alex!"
```
上面列表中的 `serve(customer:)` 函数使用显式闭包来返回客户的名字。 下面的 `serve(customer:)` 版本执行相同的操作，但不是采用显式闭包，而是通过使用 `@autoclosure` 属性标记其参数类型来采用 autoclosure。 现在你可以调用函数，就好像它使用 `String` 参数而不是闭包。 该参数会自动转换为闭包，因为 `customerProvider` 参数的类型使用 `@autoclosure` 属性标记。
```
// customersInLine is ["Ewa", "Barry", "Daniella"]
func serve(customer customerProvider: @autoclosure () -> String) {
    print("Now serving \(customerProvider())!")
}
serve(customer: customersInLine.remove(at: 0))
// Prints "Now serving Ewa!"
```
注意：过度使用自动闭包会使你的代码难以理解。 上下文和函数名称应该明确表示调用被延迟。

如果你想要允许逃逸的自动闭包，请同时使用 `@autoclosure` 和 `@escaping`
 属性。 
```
// customersInLine is ["Barry", "Daniella"]
var customerProviders: [() -> String] = []
func collectCustomerProviders(_ customerProvider: @autoclosure @escaping () -> String) {
    customerProviders.append(customerProvider)
}
collectCustomerProviders(customersInLine.remove(at: 0))
collectCustomerProviders(customersInLine.remove(at: 0))
 
print("Collected \(customerProviders.count) closures.")
// Prints "Collected 2 closures."
for customerProvider in customerProviders {
    print("Now serving \(customerProvider())!")
}
// Prints "Now serving Barry!"
// Prints "Now serving Daniella!"
```
在上面的代码中，`collectCustomerProviders(_:)` 函数不是将传递给它的闭包作为其 `customerProvider` 参数进行调用，而是将闭包附加到
 `customerProviders` 数组。 数组声明在函数范围之外，这意味着数组中的闭包可以在函数返回后执行。 因此，必须允许 `customerProvider` 参数的值转义该函数的作用域。

如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~