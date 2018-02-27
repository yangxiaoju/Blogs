# 使用 print() 函数打印应用日志
## 打印常量和变量
你可以使用 `print(_:separator:terminator:)` 函数打印常量或变量的当前值：
```
print("Hello, world!")
// Prints "Hello, world!"
```
`print(_:separator:terminator:)` 函数是一个全局函数，它将一个或多个值打印到适当的输出。例如，在 Xcode 中，`print(_:separator:terminator:)` 函数将其输出打印在 Xcode 的“控制台”窗格中。

`print(_:separator:terminator:)` 函数的完整声明为：
```
func print(_ items: Any..., separator: String = default, terminator: String = default)
```
其参数的含义是：
- items
零个或多个要打印的项目。
- separator
每个项目之间打印的字符串。 默认值是一个空格(" ")。
- terminator
所有项目打印完成后打印的字符串。 默认值是换行符("\n")。

第一个参数 items 为可变参数，这意味着你可以将零个或多个 item 传递给
 `print(_:separator:terminator:)` 函数。 每个项目的文本表示与通过调用 `String(item)` 获得的相同。例如：
```
print("One two three four five")
// Prints "One two three four five"

print(1...5)
// Prints "1...5"

print(1.0, 2.0, 3.0, 4.0, 5.0)
// Prints "1.0 2.0 3.0 4.0 5.0"
```
要打印由空格以外的内容分隔的 item，请传递一个 String 作为 separator。
```
print(1.0, 2.0, 3.0, 4.0, 5.0, separator: " ... ")
// Prints "1.0 ... 2.0 ... 3.0 ... 4.0 ... 5.0"
```
每次调用 `print(_:separator:terminator:)` 的输出默认包含一个 terminator。 要打印没有尾随换行符的 item，请传递一个空字符串作为终止符。
```
for n in 1...5 {
    print(n, terminator: "")
}
// Prints "12345"
```
separator 和 terminator 具有默认值，因此在调用此函数时可以忽略它们。
## 为 print() 添加更多信息
在项目中 print 的内容有可能是打印在控制台上，有可能会写入到日志文件并在用户发送反馈给我们时获取到，以便获取项目在运行时的更多信息。而一个完备的日志系统不可能仅仅是简单的输出网络请求的状况或者是数据库存取的情况，更多的时候需要携带当前 print() 函数所在的文件名，行，列，方法名等。在 Swift 中有几个为此而生的 Literal Expression，用来解决这些问题。

| Literal | Type | Value |
|:--------:|:-------:|:-------:|
| #file | String | 它出现的文件的名称。 |
| #line | Int | 它出现的行号。 |
| #column | Int | 它开始的列号。 |
| #function | String | 它出现的声明的名称。 |

在函数内部，#function 的值是该函数的名称，在方法中它是该方法的名称，在属性 getter 或 setter 中，它是该属性的名称，在 init 或 subscript 等特殊成员内部，它是该关键字的名称，在文件的顶层，它是当前模块的名称。

当用作函数或方法参数的默认值时，special literal 的值在调用处计算默认值表达式时确定。
```
func logFunctionName(string: String = #function) {
    print(string)
}
func myFunction() {
    logFunctionName() // Prints "myFunction()".
}
```
这样一来，就可以为 print() 的每次输出添加一些必要的辅助信息了。
```
func log(_ items: Any..., file: String = #file, line: Int = #line, function: String = #function) {
    ...
}
```
如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~