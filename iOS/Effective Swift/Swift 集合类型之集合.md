# Swift 集合类型之集合
Swift 提供了三种主要的集合类型，称为数组，集合和字典。数组是有序的值集合。集合是唯一值的无序集合。字典是键值关联的无序集合。

![](http://upload-images.jianshu.io/upload_images/1694407-ceca23a17c3417a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Swift 的数组，集合和字典总是清除它们可以存储的值和键的类型。这意味着你不能将错误类型的值插入到集合中。这意味着你可以确信你将从集合中检索的值的类型。

注意：Swift 的数组，集合和字典类型被实现为泛型集合。
## 集合的可变性
如果你创建一个数组，一个集合或一个字典，并将其分配给一个变量，则创建的集合将是可变的。这意味着你可以通过添加，删除或更改集合中的项目来更改（或变更）该集合。如果将数组，集合和字典分配给常量，那么该集合是不可变的，其大小和内容不能更改。

注意：在集合不需要更改的所有情况下，创建一个不可变集合是一个很好的习惯。这样做会使你更容易推理代码，并使 Swift 编译器能够优化你创建的集合的性能。
## 集合
一个集合在集合中存储不相同的相同类型的值，没有定义的顺序。当项目顺序不重要，或者需要确保一个项目只出现一次时，你可以使用一个集合而不是一个数组。
### 集合类型的哈希值
一个类型必须是可哈希的，才能被存储在一个集合中，也就是说，该类型必须提供一种为自己计算哈希值的方法。哈希值是一个 `Int` 值，对于所有比较相同的对象是相同的，所以如果 `a == b`，那么它跟随着 `a.hashValue == b.hashValue`。

所有 Swift 的基本类型，（如 `String`，`Int`，`Double` 和 `Bool`）都是可哈希的，可以用作集合值类型或字典类型。没有关联值的枚举 `case` 值默认情况下也是可哈希的。

注意：你可以使用自己的自定义类型作为集合值类型或字典键类型，使其符合 Swift 标准库中的 `Hashable` 协议。符合 `Hashable` 协议的类型必须提供一个名为 `hashValue` 的 gettable `Int` 属性。由类型的 `hashValue` 属性返回的值不需要在同一程序的不同执行中相同，也可以在不同的程序中相同。

因为 `Hashable` 协议符合 `Equatable`，所以符合类型还必须提供 equals 运算符(==) 的实现。`Equatable` 协议要求 == 的任何符合实现是等价关系。也就是说，对于所有值 a, b 和 c，== 的实现必须满足以下三个条件：
```
- a == a （自反性）
- a == b 意味着 b == a（对称性）
- a == b && b == c 意味着 a == c（传递性）
```
### 集合类型语法
Swift 集合的类型写为 `Set<Element>`，其中 `Element` 是该集允许存储的类型。与数组不同，集合不具有等价的语法糖。
### 创建和初始化一个空集合
你可以使用初始化语法创建一个特定类型的空集合：
```
var letters = Set<Character>()
print("letters is of type Set<Character> with \(letters.count) items.")
// Prints "letters is of type Set<Character> with 0 items."
```
注意：根据初始值设定项的类型，`letters` 的类型被推断为 `Set<Character>`

或者，如果上下文已经提供了类型信息，例如函数参数或已经类型化的变量或常量，则可以使用空数组文本创建一个空集：
```
letters.insert("a")
// letters now contains 1 value of type Character
letters = []
// letters is now an empty set, but is still of type Set<Character>
```
### 用数组语法糖创建一个集合
你也可以使用数组语法糖初始化一个集合，作为将一个或多个值作为集合写入的简写方法。

下面的例子创建一个名为 `favoriteGenres` 的组来存储 `String` 值：
```
var favoriteGenres: Set<String> = ["Rock", "Classical", "Hip hop"]
// favoriteGenres has been initialized with three initial items
```
集合类型不能单独从数组文本中推断出来，因此必须显式声明 `Set` 类型。但是，由于 Swift 的类型推断，如果使用包含相同类型的数组语法糖初始化，则不必编写该类型的集合。`favoriteGenres` 的初始化本来可以写成一个更短的形式：
```
var favoriteGenres: Set = ["Rock", "Classical", "Hip hop"]
```
由于数组文本中的所有值都是相同的类型，因此 Swift 可以推断出 `Set<String>` 是用于 `favoriteGenres` 变量的正确类型。
### 访问和修改一个集合
你可以通过其方法和属性来访问和修改一个集合。

要找出集合中的项目数量，请检查其只读属性：
```
print("I have \(favoriteGenres.count) favorite music genres.")
// Prints "I have 3 favorite music genres."
```
使用布尔 `isEmpty` 属性作为检查 `count` 属性是否等于0的快捷方式：
```
if favoriteGenres.isEmpty {
    print("As far as music goes, I'm not picky.")
} else {
    print("I have particular music preferences.")
}
// Prints "I have particular music preferences."
```
你可以通过调用集合的 `insert(_:)` 方法将一个新的项目添加到一个集合中：
```
favoriteGenres.insert("Jazz")
// favoriteGenres now contains 4 items
```
你可以通过调用集合的 `remove(_:)` 方法从集合中删除一个项目，该方法将删除该项目（如果该项目是该组的成员），并返回已删除的值，否则返回 `nil`（如果该项目不包含该项目）。或者，可以使用 `removeAll()` 方法删除集合中的所有项目。
```
if let removedGenre = favoriteGenres.remove("Rock") {
    print("\(removedGenre)? I'm over it.")
} else {
    print("I never much cared for that.")
}
// Prints "Rock? I'm over it."
```
要检查一个集合是否包含特定的项目，请使用 `contains(_:)` 方法。
```
if favoriteGenres.contains("Funk") {
    print("I get up on the good foot.")
} else {
    print("It's too funky in here.")
}
// Prints "It's too funky in here."
```
### 遍历集合
你可以使用 for-in 循环遍历集合中的值。
```
for genre in favoriteGenres {
    print("\(genre)")
}
// Jazz
// Hip hop
// Classical
```
Swift 的 `Set` 类型没有定义的顺序，要按特定的顺序迭代集合的值。请使用 `sorted()` 方法，该方法将集合的元素作为使用 < 运算符排序的数组。
```
for genre in favoriteGenres.sorted() {
    print("\(genre)")
}
// Classical
// Hip hop
// Jazz
```
## 执行集合操作
你可以高效地执行基本集合操作，例如将两个集合组合在一起，确定两个集合具有哪些值，或确定两个集合是包含全部，部分，还是不包含相同的值。
### 基本集合运算
下面的插图描述了两个集合—— a 和 b ——以阴影区域表示的各种集合操作的结果。
![](http://upload-images.jianshu.io/upload_images/1694407-73d974705d08c47f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 使用 `intersection(_:)` 方法创建一个只有两个集合通用的值的新集合。
- 使用 `symmetricDifference(_:)` 方法在集合中创建一个新集合，但不能同时包含两者。
- 使用 `union(_:)` 方法用两个集合中的所有值创建一个新集合。
- 使用 `subtracting(_:)` 方法创建一个新的集合，其值不在指定的集合中。
```
let oddDigits: Set = [1, 3, 5, 7, 9]
let evenDigits: Set = [0, 2, 4, 6, 8]
let singleDigitPrimeNumbers: Set = [2, 3, 5, 7]
 
oddDigits.union(evenDigits).sorted()
// [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
oddDigits.intersection(evenDigits).sorted()
// []
oddDigits.subtracting(singleDigitPrimeNumbers).sorted()
// [1, 9]
oddDigits.symmetricDifference(singleDigitPrimeNumbers).sorted()
// [1, 2, 9]
```
### 集合成员关系和相等性
下面的插图描述了三个集合 a, b 和 c，其中重叠区域表示在集合之间共享的元素。集合 a 是集合 b 的超集，因为 a 包含 b 中的所有元素。相反，集合 b 是 集合 a 的一个子集，因为 b 中的所有元素都包含在 a 中。集合 b 和集合 c 是彼此不相交的，因为它们不共享任何元素。
![](http://upload-images.jianshu.io/upload_images/1694407-03a345e1904cb46c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 使用相等运算符（`==`）来确定两个集合是否包含所有相同的值。
- 使用 `isSubset(of:)` 方法确定一个集合的所有值是否包含在指定的集合中。
- 使用 `isSuperset(of:)` 方法确定一个集合是否包含指定集合中的所有值。
- 使用 `sStrictSubset(of:)` 或 `isStrictSuperset(of:)` 方法来确定一个集合是一个子集还是超集，但不等于一个指定的集合。
- 使用 `isDisjoint(with:)` 方法来确定两个集合是否没有共同的值。
```
let houseAnimals: Set = ["🐶", "🐱"]
let farmAnimals: Set = ["🐮", "🐔", "🐑", "🐶", "🐱"]
let cityAnimals: Set = ["🐦", "🐭"]
 
houseAnimals.isSubset(of: farmAnimals)
// true
farmAnimals.isSuperset(of: houseAnimals)
// true
farmAnimals.isDisjoint(with: cityAnimals)
// true
```
如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~