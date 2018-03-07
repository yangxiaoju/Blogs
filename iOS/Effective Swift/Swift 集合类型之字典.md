# Swift 集合类型之字典
Swift 提供了三种主要的集合类型，称为数组，集合和字典。数组是有序的值集合。集合是唯一值的无序集合。字典是键值关联的无序集合。

![](http://upload-images.jianshu.io/upload_images/1694407-ceca23a17c3417a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Swift 的数组，集合和字典总是清除它们可以存储的值和键的类型。这意味着你不能将错误类型的值插入到集合中。这意味着你可以确信你将从集合中检索的值的类型。

注意：Swift 的数组，集合和字典类型被实现为泛型集合。
## 集合的可变性
如果你创建一个数组，一个集合或一个字典，并将其分配给一个变量，则创建的集合将是可变的。这意味着你可以通过添加，删除或更改集合中的项目来更改（或变更）该集合。如果将数组，集合和字典分配给常量，那么该集合是不可变的，其大小和内容不能更改。

注意：在集合不需要更改的所有情况下，创建一个不可变集合是一个很好的习惯。这样做会使你更容易推理代码，并使 Swift 编译器能够优化你创建的集合的性能。
## 字典
字典存储相同类型的键和集合中相同类型的值之间的关联，没有定义的顺序。每个值都与一个唯一的键相关联，该键作为字典中该值的标识符。与数组中的项目不同，字典中的项目没有指定的顺序。当你需要根据标识符查找值时，你可以使用字典，就像使用真实世界的字典查找特定字词的定义一样。
### 字典类型语法糖
Swift 字典的类型完全写成 `Dictionary<Key, Value>`，其中 `Key` 是可以用作字典键的值的类型，`Value` 是字典为这些键存储的值的类型。

注意：字典的 `Key` 的类型必须符合 `Hashable` 协议，就像集合的值类型一样。

你还可以用简写形式将字典类型写为 `[Key: Value]`。虽然这两种形式在功能上是相同的，但是当指代字典的类型时，语法糖是优选的。
### 创建一个空字典
与数组一样，你可以使用初始化语法创建一个特定类型的空字典：
```
var namesOfIntegers = [Int: String]()
// namesOfIntegers is an empty [Int: String] dictionary
```
本示例创建一个类型为 `[Int: String]` 的空字典来存储可读的整数值名称。它的键是 `Int` 类型的，它的值是 `String` 类型的。

如果上下文已经提供了类型信息，那么可以创建一个空的字典，其中有一个空的字典语法糖，写成 `[:]`（一对方括号内的冒号）:
```
namesOfIntegers[16] = "sixteen"
// namesOfIntegers now contains 1 key-value pair
namesOfIntegers = [:]
// namesOfIntegers is once again an empty dictionary of type [Int: String]
```
### 用语法糖创建一个字典
你也可以用一个语法糖来初始化一个字典，这个字典的语法与前面看到的数组语法糖相似。字典语法糖是将一个或多个键值对作为字典集合的简写形式。

键值对是键和值的组合。在字典语法糖中，每个键值对中的键和值由冒号分隔。键值对写成一个列表，用逗号分隔，并用一对方括号括起来：
```
[key 1: value 1, key 2: value 2, key 3: value 3]
```
下面的例子创建一个字典来存储国际机场的名称。在这个字典中，`key` 是三个字母的国际航空运输协会代码，其值是机场名称：
```
var airports: [String: String] = ["YYZ": "Toronto Pearson", "DUB": "Dublin"]
```
`airports` 字典被声明为具有 `[String: String]` 类型，这意味着“字典的键类型为 `String`，其值也为 `String` 类型”。

airports 字典用包含两个键值对的字典语法糖初始化。第一对有一个 `“YYZ”` 的 `key` 和一个 `“Toronto Pearson”` 的值。第二对有一个 `“DUB”` 的 `key` 和一个 `“Dublin”` 的值。

这个字典语法糖包含两个字符串对。这个键值类型匹配 `airports` 变量声明的类型（一个只有字符串键和值的字典），所以允许字典语法糖的赋值作为初始化两个初始项目的 `airports` 字典的方法。

与数组一样，如果使用键和值具有一致类型的字典语法糖进行初始化，则不必编写字典的类型。`airports` 的初始化本来可以写成一个较短的形式：
```
var airports = ["YYZ": "Toronto Pearson", "DUB": "Dublin"]
```
由于语法糖中所有键都是相同的类型，同样所有的值都是相同的类型，所以 Swift 可以推断 `[String: String]` 是用于 `airports` 字典的正确类型。
### 访问和修改字典
你可以通过其方法和属性或使用下标语法来访问和修改字典。

与数组一样，你可以通过检查 `Dictionary` 的只读属性 `count` 来找出 `Dictionary` 中的项目数量：
```
print("The airports dictionary contains \(airports.count) items.")
// Prints "The airports dictionary contains 2 items."
```
使用布尔 `isEmpty` 属性作为检查 `count` 属性是否等于0的快捷方式：
```
if airports.isEmpty {
    print("The airports dictionary is empty.")
} else {
    print("The airports dictionary is not empty.")
}
// Prints "The airports dictionary is not empty."
```
你可以使用下标语法将新项目添加到字典中。使用适当类型的新键作为下标索引，并分配适当类型的新值：
```
airports["LHR"] = "London"
// the airports dictionary now contains 3 items
```
你还可以使用下标语法来更改与特定键关联的值：
```
airports["LHR"] = "London Heathrow"
// the value for "LHR" has been changed to "London Heathrow"
```
作为下标的替代方法，使用字典的 `updateValue(_:forKey:)` 方法来设置或更新特定键的值。像上面的代码例子一样，`updateValue(_:forKey:)` 方法为一个 `key` 设置一个值，如果这个 `key` 已经存在的话。然而，与下标不同，`updateValue(_:forKey:)` 方法在执行更新后返回旧值。这使你可以检查是否发生更新。

`updateValue(_:forKey:)` 方法返回字典值类型的可选值。例如，对于存错字符串值的字典，该方法返回 `String?` 类型或“可选 `String`”的值。如果更新前存在该值，则此可选值包含该值的旧值；如果值不存在，则值为 `nil`：
```
if let oldValue = airports.updateValue("Dublin Airport", forKey: "DUB") {
    print("The old value for DUB was \(oldValue).")
}
// Prints "The old value for DUB was Dublin."
```
你也可以使用下标语法从字典中检索特定键的值。因为可以请求没有值的键，所以字典的下标返回字典值类型的可选值。如果字典包含请求键的值，则下标返回包含该键的现有值的可选值。否则，下标返回 `nil`：
```
if let airportName = airports["DUB"] {
    print("The name of the airport is \(airportName).")
} else {
    print("That airport is not in the airports dictionary.")
}
// Prints "The name of the airport is Dublin Airport."
```
你可以使用下标语法通过为该键分配值为 `nil` 来从字典中删除键值对：
```
airports["APL"] = "Apple International"
// "Apple International" is not the real airport for APL, so delete it
airports["APL"] = nil
// APL has now been removed from the dictionary
```
或者，使用 `removeValue(forKey:)` 方法从字典中移除键值对。如果键值对存在并且返回被移除的值，则该方法将移除键值对，如果没有值，则返回 `nil`：
```
if let removedValue = airports.removeValue(forKey: "DUB") {
    print("The removed airport's name is \(removedValue).")
} else {
    print("The airports dictionary does not contain a value for DUB.")
}
// Prints "The removed airport's name is Dublin Airport."
```
### 遍历字典
你可以使用 for-in 循环遍历字典中的键值对。字典中的每一项都作为 (`key`, `value`) 元组返回，并且可以将元组的成员分解为临时常量或变量，作为迭代的一部分：
```
for (airportCode, airportName) in airports {
    print("\(airportCode): \(airportName)")
}
// YYZ: Toronto Pearson
// LHR: London Heathrow
```
你还可以通过访问其键和值属性来检索字典的键或值的可遍历集合。
```
for airportCode in airports.keys {
    print("Airport code: \(airportCode)")
}
// Airport code: YYZ
// Airport code: LHR
 
for airportName in airports.values {
    print("Airport name: \(airportName)")
}
// Airport name: Toronto Pearson
// Airport name: London Heathrow
```
如果需要使用带有数组示例的 `API` 的字典键或值，请使用键或值属性初始化新数组：
```
let airportCodes = [String](airports.keys)
// airportCodes is ["YYZ", "LHR"]
 
let airportNames = [String](airports.values)
// airportNames is ["Toronto Pearson", "London Heathrow"]
```
Swift 的字典类型没有定义的顺序。要按特定顺序遍历字典的键或值，请在键或值属性上使用 `sorted()` 方法。

如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~