# URLProtectionSpace

服务器或服务器上的区域，通常称为域，需要身份验证。

## 声明

```swift
class URLProtectionSpace : NSObject
```

## 概览

保护空间定义了一系列匹配约束，用于确定应提供哪个凭证。例如，如果请求为你的 delegate 提供了一个请求客户端用户名和密码的 URLAuthenticationChallenge 对象，那么你的应用程序应该为质询的保护空间中指定的特定主机，端口，协议和域提供正确的用户名和密码。

注意：这个类没有指定的初始化器; 它的 init 方法总是返回 nil。你必须通过调用创建保护空间中描述的初始化方法之一来初始化此类。