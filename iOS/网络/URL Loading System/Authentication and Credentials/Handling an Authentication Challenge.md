# 处理身份验证挑战

当服务器要求对 URL 请求进行身份验证时，适当地做出响应。

## 概览

当你的应用程序使用 URLSessionTask 发出请求时，服务器可能会在继续之前响应一个或多个凭据要求。会话任务尝试为你处理此问题。如果不能，则会调用会话的 delegate 来处理挑战。

实现本文中描述的 delegate 方法，以回答应用程序连接到的服务器发出的质询。如果你没有实现委托，服务器可能会拒绝你的请求，并且你收到 HTTP 状态代码401（禁止）的响应，而不是你期望的数据。

## 确定适当的 Delegate 方法

根据你收到的质询的性质，实施一种或两种 delegate 身份验证方法。

实现 URLSessionTaskDelegate 的 urlSession(_:task:didReceive:completionHandler:) 方法来处理特定于任务的挑战。这些都是对用户名/密码验证要求的挑战。从给定会话创建的每个任务都可能发出自己的挑战。

> 注意：请参阅 NSURLProtectionSpace 身份验证方法常量，以获取会话范围或特定于任务的身份验证方法的指南。

举个简单的例子，考虑一下当你请求受 HTTP 基本身份验证保护的 http URL 时会发生什么，如 RFC 7617 中所定义。因为这是一个特定于任务的挑战，你可以通过实现 urlSession(_:task:didReceive:completionHandler:) 来处理这个问题。

> 注意：如果你通过 https 连接，你还会收到服务器信任质询。有关处理此类会话范围质询的信息，请参阅 Performing Manual Server Trust Authentication。

图 1 概述了响应 HTTP Basic 挑战的策略。

![](https://docs-assets.developer.apple.com/published/00ef01d197/759d2099-d938-415f-ac8a-1a0cac9dea4b.png)

以下部分实施此策略。

## 确定身份验证质询的类型

收到身份验证质询后，请使用你的 delegate 方法确定质询类型。Delegate 方法接收 URLAuthenticationChallenge 实例，该实例描述正在发出的质询。此实例包含一个 protectionSpace 属性，其 authenticationMethod 属性指示要发出的质询类型（例如对用户名和密码的请求，或客户端证书）。你可以使用此值来确定是否可以处理挑战。

你通过直接调用传递给挑战的 completion handler 来响应挑战，传递 URLSession.AuthChallengeDisposition，指示你对挑战的响应。你可以使用 disposition 参数提供凭据，取消请求或允许默认处理继续进行，以适当的方式进行。

清单1测试身份验证方法以查看它是否是预期的类型 HTTP Basic。如果 authenticationMethod 属性指示其他类型的质询，则它使用 URLSession.AuthChallengeDisposition.performDefaultHandling 处置调用完成处理程序。告诉任务使用其默认处理可以满足挑战; 否则，任务将转到响应中的下一个挑战并再次调用此委托。此过程将继续，直到任务到达你希望处理的 HTTP Basic 挑战。

清单 1 检查身份验证质询的身份验证方法

```swift
let authMethod = challenge.protectionSpace.authenticationMethod
guard authMethod == NSURLAuthenticationMethodHTTPBasic else {
    completionHandler(.performDefaultHandling, nil)
    return
}
```

## 创建凭据实例

要成功回答此挑战，你需要提交与你收到的挑战类型相对应的凭据。对于 HTTP Basic 和 HTTP Digest 问题，你需要提供用户名和密码。清单2显示了一个帮助器方法，它尝试从用户界面字段创建 URLCredential 实例（如果已填入）。

清单 2 从用户界面值创建 URLCredential

```swift
func credentialsFromUI() -> URLCredential? {
    guard let username = usernameField.text, !username.isEmpty,
        let password = passwordField.text, !password.isEmpty else {
            return nil
    }
    return URLCredential(user: username, password: password,
                         persistence: .forSession)
}
```

在此示例中，返回的 URLCredential 具有 URLCredential.Persistence.forSession 持久性，因此它仅由创建任务的 URLSession 实例存储。你需要为其他会话实例创建的任务以及应用程序的未来运行提供新的 URLCredential 实例。

## 调用 Completion Handler

一旦尝试创建凭证实例，就必须调用 completion handler 来回答挑战。

- 如果你无法创建凭证，或者用户明确取消，请调用完成处理程序并传递 URLSession.AuthChallengeDisposition.cancelAuthenticationChallenge 处置。
- 如果你可以创建凭证实例，请使用 URLSession.AuthChallengeDisposition.useCredential 处置将其传递给 completion handler。

清单 3 显示了这两个选项。

清单 3 调用身份验证质询 completion Handler

```swift
guard let credential = credentialOrNil else {
    completionHandler(.cancelAuthenticationChallenge, nil)
    return
}
completionHandler(.useCredential, credential)
```

如果你提供服务器接受的凭据，则任务开始上传或下载数据。

> 重要：对于等待用户完成用户名/密码对话框的情况，你可以将完成处理程序传递给其他方法或临时将其存储在属性中。但最终你必须调用完成处理程序来完成挑战并允许任务继续进行，即使你选择取消，如清单3的失败案例所示。

## 优雅的处理失败

如果凭据被拒绝，系统将再次调用你的 delegate 方法。发生这种情况时，回调会将你拒绝的凭据作为 URLAuthenticationChallenge 参数的 proposedCredential 属性提供。挑战实例还包括 previousFailureCount 属性，该属性指示凭据被拒绝的次数。你可以使用这些属性来确定下一步操作。例如，如果 previousFailureCount 大于零，则可以使用 proposedCredential 的用户字符串来填充用户/密码重新进入 UI。