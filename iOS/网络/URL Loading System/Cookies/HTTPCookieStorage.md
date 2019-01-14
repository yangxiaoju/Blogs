# HTTPCookieStorage

管理 cookie 存储的容器。

## 声明

```swift
class HTTPCookieStorage : NSObject
```

## 概览

每个存储的 cookie 由 HTTPCookie 类的实例表示。

## 共享 Cookie 存储

共享返回的持久性 cookie 存储可能适用于应用扩展程序或其他应用，但需遵循以下准则：

- iOS - 每个应用和应用扩展程序都有一个唯一的数据容器，这意味着它们具有单独的 Cookie 存储区。你可以使用 sharedCookieStorage(forGroupContainerIdentifier:) 方法获取公共 cookie 存储。
- macOS（非沙盒）- 从 macOS 10.11 开始，每个应用程序都有自己的 cookie 存储空间。在 macOS 10.11 之前，在用户的应用程序之间共享一个共同的 cookie 存储。
- macOS（沙盒） - 与 iOS相同。
- UIWebView  - 应用程序中的 UIWebView 实例继承父应用程序的共享 cookie 存储。
- WKWebView  - 每个 WKWebView 实例都有自己的 cookie 存储。有关更多信息，请参阅 WKHTTPCookieStore 类。

会话 cookie（cookie 对象的 isSessionOnly 属性为 true）是单个进程的本地 cookie，不共享。 

注意：如果在进程之间共享 cookie 存储，则对 cookie 接受策略所做的更改会影响使用 cookie 存储的所有当前正在运行的应用程序。

## 子类注释

HTTPCookieStorage 类可以原样使用，但你可以将其子类化。例如，你可以覆盖 storeCookies(_:for:)，getCookiesFor(_:completionHandler:) 等存储方法，以筛选存储的 Cookie，或出于安全性或其他原因重新实现存储机制。

覆盖此类的方法时，请注意系统首选采用任务参数的方法，而不是等效方法。因此，你应该在子类化时覆盖基于任务的方法，如下所示：

- 检索 cookie - 覆盖 getCookiesFor(_:completionHandler:)，而不是 cookies(for:)。
- 添加 cookie - 覆盖 storeCookies(_:for:)，而不是 setCookies(_:for:mainDocumentURL:)。