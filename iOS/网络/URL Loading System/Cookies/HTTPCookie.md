# HTTPCookie

HTTP cookie 的表示。

## 声明

```swift
class HTTPCookie : NSObject
```

## 概览

HTTPCookie 对象是不可变的，从包含 cookie 属性的字典初始化。该类支持两种不同的 cookie 版本：

- 版本0：Netscape.Most cookie 定义的原始 cookie 格式采用此格式。
- 版本1：RFC 6265 中定义的 cookie 格式，HTTP 状态管理机制。