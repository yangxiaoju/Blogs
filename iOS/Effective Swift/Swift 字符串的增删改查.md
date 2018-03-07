# Swift 字符串的增删改查
Swift 中的 `String` 类型是整个 Swift 标准库中最重要的类型之一， 熟练地掌握 `String` 的基本用法有助于减少翻看文档的时间。（可以有更多的时间用来吃鸡~）
## 字符串可变性
你可以通过将某个特定的字符串分配给一个变量（在这种情况下可以修改）或者指定一个常量（在这种情况下，它不能被修改）来指示是否可以修改（或改动）一个特定的字符串：
```
var variableString = "Horse"
variableString += " and carriage"
// variableString is now "Horse and carriage"
 
let constantString = "Highlander"
constantString += " and another Highlander"
// this reports a compile-time error - a constant string cannot be modified
```
## 字符串是值类型
Swift 的 `String` 类型是一个值类型。如果你创建一个新的字符串，该字符串被传递给一个函数或方法，或者当它被分配给一个常量或变量时被复制。在每种情况下，创建现有字符串的值副本，并且新副本被传递或分配，而不是原始版本。

Swift 的默认复制 `String` 行为可以确保当一个函数或方法传递给你一个 `String` 值时，显然你拥有这个确切的 `String` 值，而不管它来自哪里。你可以确信，你传递的字符串不会被修改，除非你自己修改它。

在幕后，Swift 的编译器优化了字符串的值复制，所以只有在绝对必要时才会进行必要的复制，这意味着在使用字符串作为值类型时，你总能获得优异的性能。
## 使用字符
你可以通过使用 for-in 循环遍历字符串来访问字符串的单个 `Character` 值：
```
for character in "Dog!🐶" {
    print(character)
}
// D
// o
// g
// !
// 🐶
```
或者，你可以通过提供字符类型标注来从单字符字符串字面量语法创建独立的字符常量或变量：
```
let exclamationMark: Character = "!"
```
字符串值可以通过将一个 `Character` 值数组作为参数传递给它的初始化方法来构造：
```
let catCharacters: [Character] = ["C", "a", "t", "!", "🐱"]
let catString = String(catCharacters)
print(catString)
// Prints "Cat!🐱"
```
## 连接字符串和字符
字符串值可以与加法运算符（+）一起添加（或连接）以创建新的字符串值：
```
let string1 = "hello"
let string2 = " there"
var welcome = string1 + string2
// welcome now equals "hello there"
```
你还可以使用复合加法运算符（+=）将字符串值追加到现有的字符串变量中：
```
var instruction = "look over"
instruction += string2
// instruction now equals "look over there"
```
你可以使用 `String` 类型的 `append()` 方法将 `Character` 值附加到 `String` 变量：
```
let exclamationMark: Character = "!"
welcome.append(exclamationMark)
// welcome now equals "hello there!"
```
如果使用多行字符串文字构建较长字符串的行，则需要字符串中的每一行以换行符结束，包括最后一行，例如：
```
let badStart = """
one
two
"""
let end = """
three
"""
print(badStart + end)
// Prints two lines:
// one
// twothree
 
let goodStart = """
one
two
 
"""
print(goodStart + end)
// Prints three lines:
// one
// two
// three
```
在上面的代码中，连接 `badStart` 和 `end` 会产生一个双行字符串，这不是所需的结果。由于 `badStart` 的最后一行不以换行符结束，因此该行与第一行的结尾相结合。相反，两行 `goodStart` 以换行符结束，所以它与结尾结合时，结果有三行，如预期的那样。
## 字符串插值
字符串插值是一种通过将常量，变量，文字和表达式的值包含在字符串文字中来构造新的字符串值的方法。你可以在单行和多行字符串文字中使用字符串插值。插入到字符串文字中的每个项目都包含在一堆括号中，并以反斜杠（\）作为前缀：
```
let multiplier = 3
let message = "\(multiplier) times 2.5 is \(Double(multiplier) * 2.5)"
// message is "3 times 2.5 is 7.5"
```
在上面的例子中，`multiplier` 的值被插入字符串文字中作为 `\(multiplier)` 。当计算字符串插值以创建实际字符串时，此占位符将替换为 `multiplier` 的实际值。

乘数的值也是稍后在字符串中较大表达式的一部分。该表达式计算 `Double(multiplier) * 2.5` 的值，并将结果（7.5）插入到字符串中。在这种情况下，当它被包含在字符串文字中时，表达式被写为 `\(Double(multiplier) * 2.5)`。

注意：在字符串插值中括号内写入的表达式不能包含未转义的反斜杠（\）,回车符或换行符。但是，它们可以包含其他字符串文字。
## 字符计数
要检索字符串中字符值的计数，请使用字符串的 `count` 属性：
```
let unusualMenagerie = "Koala 🐨, Snail 🐌, Penguin 🐧, Dromedary 🐪"
print("unusualMenagerie has \(unusualMenagerie.count) characters")
// Prints "unusualMenagerie has 40 characters"
```
请注意，Swift 对 `Character` 值使用扩展字形组意味着字符串连接和修改不总是影响字符串的字符数。

例如，如果使用四字符 `cafe` 初始化一个新字符串，然后在字符串末尾附加一个 
COMBINING ACUTE ACCENT (U+0301) ，则结果字符串的字符数仍然为 4，
`é` 是第四个字符，而不是 `e`：
```
var word = "cafe"
print("the number of characters in \(word) is \(word.count)")
// Prints "the number of characters in cafe is 4"
 
word += "\u{301}"    // COMBINING ACUTE ACCENT, U+0301
 
print("the number of characters in \(word) is \(word.count)")
// Prints "the number of characters in café is 4"
```
注意：扩展字符集群可以由多个 Unicode 标量组成。这意味着不同的字符和同一个字符的不同表示可能需要不同数量的内存。因此，如果不迭代字符串以确定其扩展的字符集群边界，则无法计算字符串中的字符数。如果你使用特别长的字符串值，请注意 `count` 属性必须遍历这个字符串中的 Unicode 标量以确定该字符串的字符。

`count` 属性返回的字符数不总是与包含相同字符的 `NSString` 的 `length` 属性相同。`NSString` 的长度基于字符串的 `UTF-16` 表示中的 16 位代码单元的数量，而不是字符串中的 Unicode 扩展字符集群的数量。
## 访问和修改字符串
你可以通过其方法和属性或使用下标语法来访问和修改字符串。
### 字符串索引
每个 `String` 值都有一个关联的索引类型 `String.Index`，它对应于字符串中每个 `Character` 的位置。

如上所示，不同的字符可能需要不同的内存量来存储，所以为了确定哪个字符位于特定的位置，必须从该字符串的开头或结尾遍历每个 Unicode 标量。由于这个原因，Swift 字符串不能被整数值索引。

使用 `startIndex` 属性来访问字符串的第一个字符的位置。`endIndex` 属性是字符串中最后一个字符之后的位置。因此，`endIndex` 属性不是字符串下标的有效参数。如果一个字符串是空的，`startIndex` 和 `endIndex` 是相等的。

你可以使用 `String` 的 `index(before:)` 和 `index(after:)` 方法访问之前和之后的索引。要访问距离给定索引较远的索引，可以使用 `index(_:offsetBy:)` 方法而不是多次调用其中一个方法。

你可以使用下标语法来访问特定字符串索引处的字符。
```
let greeting = "Guten Tag!"
greeting[greeting.startIndex]
// G
greeting[greeting.index(before: greeting.endIndex)]
// !
greeting[greeting.index(after: greeting.startIndex)]
// u
let index = greeting.index(greeting.startIndex, offsetBy: 7)
greeting[index]
// a
```
尝试访问字符串范围之外的索引或字符串范围之外的索引处的字符将触发运行时错误。
```
greeting[greeting.endIndex] // Error
greeting.index(after: greeting.endIndex) // Error
```
使用 `indice` 属性可以访问字符串中所有单个字符的索引。
```
for index in greeting.indices {
    print("\(greeting[index]) ", terminator: "")
}
// Prints "G u t e n   T a g ! "
```
注意：你可以使用符合 `Collection` 协议的任何类型的 `startIndex` 和 `endIndex` 属性和 `index(before:)`, `index(after:)` 和 `index(_: offset:)` 方法。这包括字符串（如此处所示）以及集合类型（如数组，字典和集合）。
### 插入和删除
要将单个字符插到指定索引处的字符串中，请使用 `insert(_:at:)` 方法，并将另一个字符串的内容插入到指定索引处，请使用 `insert(contentsOf:at:)` 方法。
```
var welcome = "hello"
welcome.insert("!", at: welcome.endIndex)
// welcome now equals "hello!"
 
welcome.insert(contentsOf: " there", at: welcome.index(before: welcome.endIndex))
// welcome now equals "hello there!"
```
要从指定索引处的字符串中删除单个字符，请使用 `remove(at:)` 方法，并删除指定范围的子字符串，请使用 `removeSubrange(_:)` 方法。
```
welcome.remove(at: welcome.index(before: welcome.endIndex))
// welcome now equals "hello there"
 
let range = welcome.index(welcome.endIndex, offsetBy: -6)..<welcome.endIndex
welcome.removeSubrange(range)
// welcome now equals "hello"
```
注意：你可以使用符合 `RangeReplaceableCollection` 协议的任何类型的 `insert(_:at:)`，`insert(contentOf:at:)`，`remove(at:)` 和 `removeSubrange(_:) `方法。这包括字符串（如此出所示）以及集合类型（如数组，字典和集合）。
## Substrings
当你从一个字符串中得到一个子字符串，例如使用下标或像 `prefix(_:)` 这样的方法时，结果就是一个 `Substring` 的实例，而不是另一个字符串。Swift 中的子字符串与字符串的大部分方法相同，这意味着你可以像处理字符串一样使用子字符串。但是，与字符串不同，在对字符串执行操作时，只需要很短的时间就可以使用子字符串。当你准备好将结果存储较长时间时，可以将子字符串转换为字符串的一个实例。例如：
```
let greeting = "Hello, world!"
let index = greeting.index(of: ",") ?? greeting.endIndex
let beginning = greeting[..<index]
// beginning is "Hello"
 
// Convert the result to a String for long-term storage.
let newString = String(beginning)
```
像字符串一样，每个子字符串都有一个内存区域，其中构成子字符串的字符被存储。字符串和子字符串之间的区别在于，作为性能优化，子字符串可以重用用于存储原始字符串的部分内存，或者用于存储另一个子字符串的部分内存。（字符串有一个类似的优化，但如果两个字符串共享内存，它们是相等的。）这种性能优化意味着，你不必花费内存的成本，知道你修改字符串或子字符串。如上所述，子字符串不适合长期存储——因为它们重用原始字符串的存储空间，只要使用任何子字符串，整个原始字符串就必须保存在内存中。

在上面的例子中，`greeting` 是一个字符串，这意味着它有一个内存区域，构成字符串的字符被存储。因为开始是一个 `greeting` 的子字符串，所以它重用了 `greeting` 使用的内存。相反，`newString` 是一个字符串，当它从字符串创建时，它有自己的存储空间。下图显示了这些关系：
![](http://upload-images.jianshu.io/upload_images/1694407-6223eea165a66ee5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
注意：`String` 和 `Substring` 都符合 `StringProtocol` 协议，这意味着字符串操作函数接受 `StringProtocol` 值通常很方便。你可以使用 `String` 或 `Substring` 值来调用这些函数。
## 比较字符串
Swift 提供了三种比较文本值的方法：字符串和字符相等，前缀相等，和后缀相等
### 字符串和字符相等
如比较运算符中所述，使用“等于”运算符（==）和不等于运算符（!=）检查字符串和字符的相等性：
```
let quotation = "We're a lot alike, you and I."
let sameQuotation = "We're a lot alike, you and I."
if quotation == sameQuotation {
    print("These two strings are considered equal")
}
// Prints "These two strings are considered equal"
```
两个字符串值（或两个字符值）被认为是相等的，如果他们的扩展字形群是正则等价的。如果扩展字形集群具有相同的语言含义和外观，即使它们是由不同的 Unicode 标量在幕后组成的，它们也是相同的。

例如， LATIN SMALL LETTER E WITH ACUTE (U+00E9) 与 LATIN SMALL LETTER E (U+0065) 正好想等，随后是COMBINING ACUTE ACCENT (U+0301)。这两个扩展的字形组合都是表示字符的有效方式，所以它们被认为是正则等价的：
```
// "Voulez-vous un café?" using LATIN SMALL LETTER E WITH ACUTE
let eAcuteQuestion = "Voulez-vous un caf\u{E9}?"
 
// "Voulez-vous un café?" using LATIN SMALL LETTER E and COMBINING ACUTE ACCENT
let combinedEAcuteQuestion = "Voulez-vous un caf\u{65}\u{301}?"
 
if eAcuteQuestion == combinedEAcuteQuestion {
    print("These two strings are considered equal")
}
// Prints "These two strings are considered equal"
```
例如，英文中使用的 LATIN CAPITAL LETTER A (U+0041, or "A") 与俄文中使用的 CYRILLIC CAPITAL LETTER A (U+0410, or "А") 并不相同。这些字符在外观上相似，但不具有相同的语言含义：
```
let latinCapitalLetterA: Character = "\u{41}"
 
let cyrillicCapitalLetterA: Character = "\u{0410}"
 
if latinCapitalLetterA != cyrillicCapitalLetterA {
    print("These two characters are not equivalent.")
}
// Prints "These two characters are not equivalent."
```
注意：Swift 中的字符串和字符比较不是区域设置敏感的。
### 前缀和后缀相等性
要检查一个字符串是否具有特定的字符串前缀或后缀，请调用字符串的 `hasPrefix(_:)` 和 `hasSuffix(_:)` 方法，这两个方法都采用 `String` 类型的单个参数，并返回一个布尔值。

下面的例子考虑了一组代表莎士比亚的罗密欧与朱丽叶的前两幕的场景位置：
```
let romeoAndJuliet = [
    "Act 1 Scene 1: Verona, A public place",
    "Act 1 Scene 2: Capulet's mansion",
    "Act 1 Scene 3: A room in Capulet's mansion",
    "Act 1 Scene 4: A street outside Capulet's mansion",
    "Act 1 Scene 5: The Great Hall in Capulet's mansion",
    "Act 2 Scene 1: Outside Capulet's mansion",
    "Act 2 Scene 2: Capulet's orchard",
    "Act 2 Scene 3: Outside Friar Lawrence's cell",
    "Act 2 Scene 4: A street in Verona",
    "Act 2 Scene 5: Capulet's mansion",
    "Act 2 Scene 6: Friar Lawrence's cell"
]
```
你可以使用 `hasPrefix(_:)` 方法与 `romeoAndJuliet` 数组来计算剧本的第1部分中的场景量：
```
var act1SceneCount = 0
for scene in romeoAndJuliet {
    if scene.hasPrefix("Act 1 ") {
        act1SceneCount += 1
    }
}
print("There are \(act1SceneCount) scenes in Act 1")
// Prints "There are 5 scenes in Act 1"
```
同样，使用 `hasSuffix(_:)` 方法来计算 Capulet 大厦和 Friar Lawrence 小区或周围发生的场景数量：
```
var mansionCount = 0
var cellCount = 0
for scene in romeoAndJuliet {
    if scene.hasSuffix("Capulet's mansion") {
        mansionCount += 1
    } else if scene.hasSuffix("Friar Lawrence's cell") {
        cellCount += 1
    }
}
print("\(mansionCount) mansion scenes; \(cellCount) cell scenes")
// Prints "6 mansion scenes; 2 cell scenes"
```
注意：`hasPrefix(_:)` 和 `hasSuffix(_:)` 方法在每个字符串中的扩展字形集群之间执行逐字符规范等价比较。

如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~