# URLCredentialStorage

共享凭证缓存的管理员。

## 声明

```swift
class URLCredentialStorage : NSObject
```

## 概览

共享缓存存储和检索 URLCredential 的实例。你可以根据创建的 URLCredential.Persistence 永久存储基于密码的凭据。永远不会存储基于证书的凭据。

## 子类注释

URLCredentialStorage 类旨在按原样使用，但如果你有特定需求，则可以对其进行子类化，例如筛选存储的凭据。

覆盖此类的方法时，请注意采用任务参数的方法优先于不具有该方法的等效方法。因此，你应该在子类化时覆盖基于任务的方法，如下所示：

- 设置凭据 - 覆盖 set(_:for:task:) 而不是 set(_:for:)。
- 获取凭据 - 覆盖 getCredentials(for:task:completionHandler:) 而不是 credentials(for:)。
- 删除凭据 - 覆盖 remove(_:for:options:task:) 而不是 remove(_:for:options:) 和 remove(_:for:)。
- 设置默认凭据 - 覆盖 setDefaultCredential(_:for:task:) 而不是 setDefaultCredential(_:for:)。
- 获取默认凭据 - 覆盖 getDefaultCredential(for:task:completionHandler:) 而不是 defaultCredential(for:)。