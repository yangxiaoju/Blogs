# URLAuthenticationChallenge

来自需要客户端身份验证的服务器的挑战。

## 声明

```swift
class URLAuthenticationChallenge : NSObject
```

## 概览

你的应用程序在各种 URLSession，NSURLConnection 和 NSURLDownload delegate 方法中收到身份验证挑战，例如 urlSession(_:task:didReceive:completionHandler:)。这些对象提供了在决定如何处理服务器的身份验证请求时所需的信息。

该身份验证挑战的核心是一个保护空间，它定义了所请求的身份验证类型，主机和端口号，网络协议以及（如果适用）身份验证领域（共享的同一服务器上的一组相关 URL） 一套凭证）。