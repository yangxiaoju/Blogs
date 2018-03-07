# Swift 集合类型之数组
Swift 提供了三种主要的集合类型，称为数组，集合和字典。数组是有序的值集合。集合是唯一值的无序集合。字典是键值关联的无序集合。

![](http://upload-images.jianshu.io/upload_images/1694407-ceca23a17c3417a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Swift 的数组，集合和字典总是清除它们可以存储的值和键的类型。这意味着你不能将错误类型的值插入到集合中。这意味着你可以确信你将从集合中检索的值的类型。

注意：Swift 的数组，集合和字典类型被实现为泛型集合。
## 集合的可变性
如果你创建一个数组，一个集合或一个字典，并将其分配给一个变量，则创建的集合将是可变的。这意味着你可以通过添加，删除或更改集合中的项目来更改（或变更）该集合。如果将数组，集合和字典分配给常量，那么该集合是不可变的，其大小和内容不能更改。

注意：在集合不需要更改的所有情况下，创建一个不可变集合是一个很好的习惯。这样做会使你更容易推理代码，并使 Swift 编译器能够优化你创建的集合的性能。
## 数组
数组将相同类型的值存储在有序列表中。相同的值可以在不同的位置多次出现在数组中。
### 数组类型语法糖
Swift 数组的类型完全写成 `Array<Element>`，其中 `Element` 是数组允许存储的值的类型。你也可以用 `[Element]` 的简写形式写出一个数组的类型。尽管这两种形式在功能上是相同的，但是当指代数组的类型时，语法糖是优选的。
### 创建一个空数组
你可以使用初始化语法创建一个特定类型的空数组：
```
var someInts = [Int]()
print("someInts is of type [Int] with \(someInts.count) items.")
// Prints "someInts is of type [Int] with 0 items."
```
请注意，`someInts` 变量的类型从初始化方法的类型被推断为 `[Int]`。

或者，如果上下文已经提供了类型信息，比如函数参数或已经存在类型的变量或常量则可以创建一个空数组，其中包含一个空的数组文本，写成 `[]`（一对空括号）：
```
someInts.append(3)
// someInts now contains 1 value of type Int
someInts = []
// someInts is now an empty array, but is still of type [Int]
```
### 创建一个带有默认值的数组
Swift 的数组类型还提供了一个初始化方法，用于创建一个具有特定大小的数组，并将其所有值设置为相同的默认值。你将此初始化方法传递给适当类型的默认值（称为 `repeating`），以及在新数组中重复值的次数（称为 `count`）:
```
var threeDoubles = Array(repeating: 0.0, count: 3)
// threeDoubles is of type [Double], and equals [0.0, 0.0, 0.0]
```
### 通过一起添加两个数组来创建一个数组
你可以使用加法运算符（+）将两个具有兼容类型的现有数组加在一起来创建一个新数组。新数组的类型是从你添加在一起的两个数组的类型推断出来的：
```
var anotherThreeDoubles = Array(repeating: 2.5, count: 3)
// anotherThreeDoubles is of type [Double], and equals [2.5, 2.5, 2.5]
 
var sixDoubles = threeDoubles + anotherThreeDoubles
// sixDoubles is inferred as [Double], and equals [0.0, 0.0, 0.0, 2.5, 2.5, 2.5]
```
### 通过语法糖创建一个数组
你还可以使用数组语法糖初始化数组，这是将一个或多个值作为数组集合写入的简写方法。一个数组语法糖被写成一个值列表，用逗号分隔，并被一对方括号包围起来：
```
[value 1, value 2, value 3]
```
下面的例子创建一个名为 `shoppingList` 的数组来存储 `String` 值：
```
var shoppingList: [String] = ["Eggs", "Milk"]
// shoppingList has been initialized with two initial items
```
`shoppingList` 变量被声明为“`String` 数组”，写成 `[String]`。因为这个特定的数组已经指定了一个 `String` 类型的值类型，所以只允许存储 `String` 值。这里，`shoppingList` 数组使用两个字符串（“Eggs”和“Milk”）初始化的，写在一个数组语法糖中。

在这种情况下，数组文字包含两个字符串的值，没有别的。这与 `shoppingList` 变量的声明类型（一个只能包含 `String` 值的数组）是相匹配的，所以数组文本的赋值是允许用两个初始化项来初始化 `shoppingList` 的。

感谢 Swift 的类型推断，如果使用包含相同类型值的数组语法糖初始化数组，则不必编写数组的类型。`shoppingList` 的初始化本来可以写成一个简短的形式。
```
var shoppingList = ["Eggs", "Milk"]
```
用于数组文本中的所有值都是相同的类型，因此 Swift 可以推断出 `[String]` 是用于 shoppingList 变量的正确类型。
### 访问和修改数组
你可以通过其方法和属性或使用下标语法来访问和修改数组。

要找出数组中项目的数量，请使用其只读计数属性：
```
print("The shopping list contains \(shoppingList.count) items.")
// Prints "The shopping list contains 2 items."
```
使用布尔 `isEmpty` 属性作为检查 `count` 属性是否等于0的快捷方式：
```
if shoppingList.isEmpty {
    print("The shopping list is empty.")
} else {
    print("The shopping list is not empty.")
}
// Prints "The shopping list is not empty."
```
你可以通过调用数组的 `append(_:)` 方法将新项添加到数组的末尾：
```
shoppingList.append("Flour")
// shoppingList now contains 3 items, and someone is making pancakes
```
或者，使用加法赋值运算符（+=）追加一个或多个兼容项目的数组：
```
shoppingList += ["Baking Powder"]
// shoppingList now contains 4 items
shoppingList += ["Chocolate Spread", "Cheese", "Butter"]
// shoppingList now contains 7 items
```
通过使用下标语法从数组中检索一个值，在数组的名称之后立即传递要在方括号内检索的值的索引：
```
var firstItem = shoppingList[0]
// firstItem is equal to "Eggs"
```
注意：数组中的第一项索引为0，而不是1。Swift 中的数组始终为0索引。

你可以使用下标语法来更改给定索引处的现有值：
```
shoppingList[0] = "Six eggs"
// the first item in the list is now equal to "Six eggs" rather than "Eggs"
```
当使用下标语法时，你指定的索引需要有效。例如，在编写 `shoppingList[shoppingList.count] = "Salt"` 来尝试将一个项目追加到数组的末尾会导致运行时错误。

即使替换值的长度与要替换的范围不同，也可以使用下标语法一次更改一系列值。下面的例子用 `“Bananas”` 和 `“Apples”` 替代 `“Chocolate Spread”`，`“Cheese”` 和 `“Butter”`
```
shoppingList[4...6] = ["Bananas", "Apples"]
// shoppingList now contains 6 items
```
要在指定的索引处插入一个项目到数组，请调用数组的 `insert(_:at:)` 方法。
```
shoppingList.insert("Maple Syrup", at: 0)
// shoppingList now contains 7 items
// "Maple Syrup" is now the first item in the list
```
对 `insert(_:at:)` 方法在 `shoppingList` 最开始处插入具有值 `“Maple Syrup”` 的新项目，由索引0指示。

同样，你使用 `remove(at:)` 方法，从数组中删除一个项目。此方法删除指定索引处的项目并返回已删除的项目（但如果不需要，则可以忽略返回的值）：
```
let mapleSyrup = shoppingList.remove(at: 0)
// the item that was at index 0 has just been removed
// shoppingList now contains 6 items, and no Maple Syrup
// the mapleSyrup constant is now equal to the removed "Maple Syrup" string
```
注意：如果尝试访问或修改现有边界之外的索引的值，则会触发运行时错误。你可以通过将其与数组的 `count` 属性进行比较来检查索引是否有效。数组中最大的有效索引是 count-1，因为数组是从0开始索引的，但是当 `count` 为0（意味着数组为空）时，没有有效的索引。

当一个物品被移除时，数组中的任何间隙都会关闭，因此索引0处的值再次等于`“Six eggs”`：
```
firstItem = shoppingList[0]
// firstItem is now equal to "Six eggs"
```
如果要从数组中删除最后一项，请使用 `removeLast()` 方法而不是 `remove(at:)` 方法来避免查询数组的 `count` 属性。像 `remove(at:)` 方法一样，`removeLast()` 返回被删除的项目。
```
let apples = shoppingList.removeLast()
// the last item in the array has just been removed
// shoppingList now contains 5 items, and no apples
// the apples constant is now equal to the removed "Apples" string
```
### 遍历数组
你可以使用 for-in 循环遍历数组中的所有值：
```
for item in shoppingList {
    print(item)
}
// Six eggs
// Milk
// Flour
// Baking Powder
// Bananas
```
如果你需要每个项目的索引以及其值，请使用 `enumerated()` 方法遍历数组。对于数组中的每个项目，`enumerated()` 方法返回一个由整数和项目组成的元组。整数从零开始，每个项目加1；如果你遍历整个数组，这些整数匹配项目的索引。你可以将元组分解为临时常量或变量，作为迭代的一部分：
```
for (index, value) in shoppingList.enumerated() {
    print("Item \(index + 1): \(value)")
}
// Item 1: Six eggs
// Item 2: Milk
// Item 3: Flour
// Item 4: Baking Powder
// Item 5: Bananas
```
如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~