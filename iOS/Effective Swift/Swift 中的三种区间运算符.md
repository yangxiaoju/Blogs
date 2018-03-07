# Swift 中的三种区间运算符
Swift 包含几个区间运算符，它们是表示一系列值的捷径。
## 闭区间运算符
闭区间运算符（`a...b`）定义从 a 到 b 的范围，并包含值 a 和 b。a 的值不能大于 b。

在遍历希望使用所有值的范围（如使用 for-in 循环）时，闭区间运算符很有用。
```
for index in 1...5 {
    print("\(index) times 5 is \(index * 5)")
}
// 1 times 5 is 5
// 2 times 5 is 10
// 3 times 5 is 15
// 4 times 5 is 20
// 5 times 5 is 25
```
## 半开区间运算符
半开区间运算符（`a..<b`）定义了从 a 到 b 的范围，但不包括 b。它是半开放的，因为它包含了第一个值，而不包含最终值。与闭区间运算符一样，a 的值不得大于 b。如果 a 的值等于 b，则结果范围将为空：

当你使用基于零的列表（例如数组）时，半开区间特别有用，在该列表中，计算（但不包括）列表长度是有用的：
```
let names = ["Anna", "Alex", "Brian", "Jack"]
let count = names.count
for i in 0..<count {
    print("Person \(i + 1) is called \(names[i])")
}
// Person 1 is called Anna
// Person 2 is called Alex
// Person 3 is called Brian
// Person 4 is called Jack
```
请注意，该数组包含四个项目，但 `0..<count` 只计数至3（数组中最后一个项目的索引），因为它是半开区间。
## 单向区间运算符
闭区间运算符有一个替代形式，用于在一个方向上尽可能延续的范围——例如，范围包括从索引2到数组末尾的数组的所有元素。在这些情况下，可以省略区间运算符的一侧的值。这种区间被称为单向区间，因为运算符只有一个方向的范围。例如：
```
for name in names[2...] {
    print(name)
}
// Brian
// Jack
 
for name in names[...2] {
    print(name)
}
// Anna
// Alex
// Brian
```
半开区间运算符也有一个只写了最终值的片面形式。就像在双方都包含值一样，最终的值不是范围的一部分。例如：
```
for name in names[..<2] {
    print(name)
}
// Anna
// Alex
```
单边的区间可以用在其他上下文中，而不仅仅是下标。你不能遍历忽略第一个值的单项区间，因为不清楚遍历开始的位置。你可以遍历一个忽略了最终值的单向区间；但是，由于区间无限的延续，请确保为循环添加明确的结束条件。你还可以检查单边范围是否包含特定值，如下面的代码所示。
```
let range = ...5
range.contains(7)   // false
range.contains(4)   // true
range.contains(-1)  // true
```
如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~