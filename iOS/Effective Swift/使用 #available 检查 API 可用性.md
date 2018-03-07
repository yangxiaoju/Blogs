# 使用 #available 检查 API 可用性
Swift 内置了对 API 可用性检查的支持，确保你不会意外使用给定部署 target 上不可用的 API。

编译器使用 SDK 中的可用性信息来验证代码中使用的所有 API 在项目指定的部署 target 上是否可用。 如果你尝试使用不可用的 API，Swift 会在编译时报告错误。

在 if 或 guard 语句中使用可用性条件来有条件地执行代码块，具体取决于你要使用的 API 在运行时是否可用。 编译器在验证该代码块中的 API 可用时使用可用性条件中的信息。
```
if #available(iOS 10, macOS 10.12, *) {
    // Use iOS 10 APIs on iOS, and use macOS 10.12 APIs on macOS
} else {
    // Fall back to earlier iOS and macOS APIs
}
```
上面的可用性条件指定在 iOS 中，if 语句的主体仅在 iOS 10 及更高版本中执行; 在 macOS 中，仅在 macOS 10.12 及更高版本中。 最后一个参数 * 是必需的，并指定在任何其他平台上，if 的主体在你的目标指定的最小部署 target 上执行。

通用形式中，可用性条件采用平台名称和版本的列表。  除了指定主要版本号（如 iOS 8 或 MacOS 10.10）之外，你还可以指定次要版本号，如 iOS 8.3 和
 MacOS 10.10.3。
```
if #available(platform name version, ..., *) {
    statements to execute if the APIs are available
} else {
    fallback statements to execute if the APIs are unavailable
}
```
如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~