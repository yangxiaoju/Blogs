# Swift 控制转移语句
控制转移语句通过将控制从一段代码转移到另一段代码来改变你的代码的执行顺序。 Swift 有五个控制转移语句：
- continue
- break
- fallthrough
- return
- throw

其中，continue 和 break 都与其他语言中的别无二致，于是简要介绍下 fallthrough 的用法。
## Fallthrough
在 Swift 中，switch 语句不会落在每个 case 的底部并进入下一个 case。 也就是说，只要第一个 switch case 完成，整个 switch 语句就会完成它的执行。 相比之下，C 要求您在每个 case 的末尾插入一个明确的 break 语句以防止漏掉。 避免默认的延迟意味着 Swift switch 语句比 C 中的对应语言更加简洁和可预测，因此避免了错误地执行多个 case 事件。

如果你需要 C 风格的贯穿行为，你可以使用 fallthrough 关键字逐个选择加入此行为。 下面的例子使用 fallthrough 创建一个数字的文本描述。
```
let integerToDescribe = 5
var description = "The number \(integerToDescribe) is"
switch integerToDescribe {
case 2, 3, 5, 7, 11, 13, 17, 19:
    description += " a prime number, and also"
    fallthrough
default:
    description += " an integer."
}
print(description)
// Prints "The number 5 is a prime number, and also an integer."
```
此示例声明一个名为 description 的新 String 变量并为其分配一个初始值。 该函数然后使用 switch 语句来考虑 integerToDescribe 的值。 如果
 integerToDescribe 的值是列表中的素数之一，则该函数会将文本附加到描述的末尾，以指出该数字是素数。 然后，它使用 fallthrough 关键字来“陷入”默认情况。 默认情况下会在描述的末尾添加一些额外的文本，并且 switch 语句已完成。

除非 integerToDescribe 的值在已知素数列表中，否则根本不会与第一个 switch case 相匹配。 由于没有其他特定情况，因此 integerToDescribe 与默认情况相匹配。

switch 语句执行完成后，使用 `print(_:separator:terminator:)` 函数打印数字的描述。 在这个例子中，数字5被正确识别为素数。

注意：fallthrough 关键字不检查导致执行陷入的 switch case 的情况。 fallthrough 关键字简单地使代码执行直接移动到下一个 case（或 default case）块内的语句，就像在 C 的标准 switch 语句行为中一样。
## Labeled Statements
在 Swift 中，可以在其他循环和条件语句中嵌套循环和条件语句，以创建复杂的控制流结构。但是，循环和条件语句都可以使用 break 语句来提前结束它们的执行。因此，明确你想要终止 break 语句的循环或条件语句有时很有用。同样，如果你有多个嵌套循环，则明确说明 continue 语句应该影响哪个循环会很有用。

为了实现这些目标，你可以用语句标签标记循环语句或条件语句。使用条件语句，可以使用带有 break 语句的语句标签来结束标记语句的执行。使用循环语句，可以使用带有 break 或 continue 语句的语句标签来结束或继续执行带标签的语句。

带标签的语句通过将标签放置在与语句的 introducer 关键字相同的行上，后跟冒号。下面是 while 循环语法的一个例子，虽然所有循环和 switch 语句的原理是相同的：
```
label name: while condition {
    statements
}
```
以下是官方文档中的代码示例：
```
gameLoop: while square != finalSquare {
    diceRoll += 1
    if diceRoll == 7 { diceRoll = 1 }
    switch square + diceRoll {
    case finalSquare:
        // diceRoll will move us to the final square, so the game is over
        break gameLoop
    case let newSquare where newSquare > finalSquare:
        // diceRoll will move us beyond the final square, so roll again
        continue gameLoop
    default:
        // this is a valid move, so find out its effect
        square += diceRoll
        square += board[square]
    }
}
print("Game over!")
```
注意：如果上面的 break 语句没有使用 gameLoop 标签，它将跳出 switch 语句，而不是 while 语句。 使用 gameLoop 标签可以清楚地知道应该终止哪个控制语句。

在调用继续 gameLoop 跳转到循环的下一次迭代时，使用 gameLoop 标签并不是必须的。 游戏中只有一个循环，因此对于 continue 语句将影响哪个循环没有任何含糊之处。 然而，在 continue 语句中使用 gameLoop 标签并没有什么坏处。 这样做符合标签与 break 语句一起使用，并有助于使游戏的逻辑更加清晰，以便阅读和理解。

如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~