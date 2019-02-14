# AFNetworking 源码解读（二）：AFSecurityPolicy

AFSecurityPolicy 通过安全连接评估服务器对固定的 X.509 证书和公钥的信任。将固定的 SSL 证书添加到你的应用程序有助于防止中间人攻击和其他漏洞。强烈建议处理敏感客户数据或财务信息的应用程序通过 HTTPS 连接路由所有通信，并配置并启用 SSL 固定。

## 初始化方法

先来看无需任何参数的默认初始化方法：

```objc
+ (instancetype)defaultPolicy {
    AFSecurityPolicy *securityPolicy = [[self alloc] init];
    securityPolicy.SSLPinningMode = AFSSLPinningModeNone;

    return securityPolicy;
}
```

该方法简单的调用了默认初始化方法创建了 AFSecurityPolicy 对象，并设置其 SSLPinningMode 为 AFSSLPinningModeNone。

SSLPinningMode 是 AFSSLPinningMode 类型的枚举，声明如下：

```objc
typedef NS_ENUM(NSUInteger, AFSSLPinningMode) {
    AFSSLPinningModeNone,
    AFSSLPinningModePublicKey,
    AFSSLPinningModeCertificate,
};
```

1. AFSSLPinningModeNone 不要使用固定证书来验证服务器。
2. AFSSLPinningModePublicKey 根据固定证书的公钥验证主机证书。
3. AFSSLPinningModeCertificate 针对固定证书验证主机证书。

init 方法的实现如下：

```objc
- (instancetype)init {
    self = [super init];
    if (!self) {
        return nil;
    }

    self.validatesDomainName = YES;

    return self;
}
```

AFSecurityPolicy 还提供了可以配置 pinningMode 的初始化方法：

```objc
+ (instancetype)policyWithPinningMode:(AFSSLPinningMode)pinningMode {
    return [self policyWithPinningMode:pinningMode withPinnedCertificates:[self defaultPinnedCertificates]];
}
```

defaultPinnedCertificates 方法的实现如下：

```objc
+ (NSSet *)defaultPinnedCertificates {
    static NSSet *_defaultPinnedCertificates = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        NSBundle *bundle = [NSBundle bundleForClass:[self class]];
        _defaultPinnedCertificates = [self certificatesInBundle:bundle];
    });

    return _defaultPinnedCertificates;
}
```

该方法只会执行一次，用于从当前类所在的 bundle 中取出证书，取出证书的方法如下：

```objc
+ (NSSet *)certificatesInBundle:(NSBundle *)bundle {
    NSArray *paths = [bundle pathsForResourcesOfType:@"cer" inDirectory:@"."];

    NSMutableSet *certificates = [NSMutableSet setWithCapacity:[paths count]];
    for (NSString *path in paths) {
        NSData *certificateData = [NSData dataWithContentsOfFile:path];
        [certificates addObject:certificateData];
    }

    return [NSSet setWithSet:certificates];
}
```

这个方法会获取所有的 .cer 文件并将其转为 NSData 类型放入集合中并返回。

同时，AFSecurityPolicy 还支持自己传入证书的初始化方法：

```objc
+ (instancetype)policyWithPinningMode:(AFSSLPinningMode)pinningMode withPinnedCertificates:(NSSet *)pinnedCertificates {
    AFSecurityPolicy *securityPolicy = [[self alloc] init];
    securityPolicy.SSLPinningMode = pinningMode;

    [securityPolicy setPinnedCertificates:pinnedCertificates];

    return securityPolicy;
}
```

### setPinnedCertificates

setPinnedCertificates 被重写，用于在设置 pinnedCertificates 属性时造成 sideEffect。

```objc
- (void)setPinnedCertificates:(NSSet *)pinnedCertificates {
    _pinnedCertificates = pinnedCertificates;

    if (self.pinnedCertificates) {
        NSMutableSet *mutablePinnedPublicKeys = [NSMutableSet setWithCapacity:[self.pinnedCertificates count]];
        for (NSData *certificate in self.pinnedCertificates) {
            id publicKey = AFPublicKeyForCertificate(certificate);
            if (!publicKey) {
                continue;
            }
            [mutablePinnedPublicKeys addObject:publicKey];
        }
        self.pinnedPublicKeys = [NSSet setWithSet:mutablePinnedPublicKeys];
    } else {
        self.pinnedPublicKeys = nil;
    }
}
```

该方法会遍历 pinnedCertificates 属性，并针对每一个 certificate 通过 AFPublicKeyForCertificate 函数生成一个 publicKey。AFPublicKeyForCertificate 函数实现如下：

```objc
static id AFPublicKeyForCertificate(NSData *certificate) {
    id allowedPublicKey = nil;
    SecCertificateRef allowedCertificate;
    SecPolicyRef policy = nil;
    SecTrustRef allowedTrust = nil;
    SecTrustResultType result;

    allowedCertificate = SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certificate);
    __Require_Quiet(allowedCertificate != NULL, _out);

    policy = SecPolicyCreateBasicX509();
    __Require_noErr_Quiet(SecTrustCreateWithCertificates(allowedCertificate, policy, &allowedTrust), _out);
    __Require_noErr_Quiet(SecTrustEvaluate(allowedTrust, &result), _out);

    allowedPublicKey = (__bridge_transfer id)SecTrustCopyPublicKey(allowedTrust);

_out:
    if (allowedTrust) {
        CFRelease(allowedTrust);
    }

    if (policy) {
        CFRelease(policy);
    }

    if (allowedCertificate) {
        CFRelease(allowedCertificate);
    }

    return allowedPublicKey;
}
```

该函数做了如下几件事情：

1. 通过 SecCertificateCreateWithData 函数创建一个 SecCertificateRef 类型的对象。
2. __Require_Quiet 会在传入的条件为 false 的时候，调用 _out 标签后的代码。
3. 通过 SecPolicyCreateBasicX509 函数创建了一个 SecPolicyRef 类型的对象。
4. __Require_noErr_Quiet 当条件抛出异常时，执行标记以后的代码。SecTrustCreateWithCertificates 会根据 allowedCertificate 和 policy 生成 SecTrustRef 对象。SecTrustEvaluate 会根据 allowedTrust 创建 SecTrustResultType 对象。
5. 最后，SecTrustCopyPublicKey 函数会使用 allowedTrust 作为参数创建 allowedPublicKey 并返回。

### 校验证书

evaluateServerTrust:forDomain: 是用来校验证书用的方法，实现如下：

```objc
- (BOOL)evaluateServerTrust:(SecTrustRef)serverTrust
                  forDomain:(NSString *)domain
{
    if (domain && self.allowInvalidCertificates && self.validatesDomainName && (self.SSLPinningMode == AFSSLPinningModeNone || [self.pinnedCertificates count] == 0)) {
        // https://developer.apple.com/library/mac/documentation/NetworkingInternet/Conceptual/NetworkingTopics/Articles/OverridingSSLChainValidationCorrectly.html
        //  According to the docs, you should only trust your provided certs for evaluation.
        //  Pinned certificates are added to the trust. Without pinned certificates,
        //  there is nothing to evaluate against.
        //
        //  From Apple Docs:
        //          "Do not implicitly trust self-signed certificates as anchors (kSecTrustOptionImplicitAnchors).
        //           Instead, add your own (self-signed) CA certificate to the list of trusted anchors."
        NSLog(@"In order to validate a domain name for self signed certificates, you MUST use pinning.");
        return NO;
    }

    NSMutableArray *policies = [NSMutableArray array];
    if (self.validatesDomainName) {
        [policies addObject:(__bridge_transfer id)SecPolicyCreateSSL(true, (__bridge CFStringRef)domain)];
    } else {
        [policies addObject:(__bridge_transfer id)SecPolicyCreateBasicX509()];
    }

    SecTrustSetPolicies(serverTrust, (__bridge CFArrayRef)policies);

    if (self.SSLPinningMode == AFSSLPinningModeNone) {
        return self.allowInvalidCertificates || AFServerTrustIsValid(serverTrust);
    } else if (!AFServerTrustIsValid(serverTrust) && !self.allowInvalidCertificates) {
        return NO;
    }

    switch (self.SSLPinningMode) {
        case AFSSLPinningModeNone:
        default:
            return NO;
        case AFSSLPinningModeCertificate: {
            NSMutableArray *pinnedCertificates = [NSMutableArray array];
            for (NSData *certificateData in self.pinnedCertificates) {
                [pinnedCertificates addObject:(__bridge_transfer id)SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certificateData)];
            }
            SecTrustSetAnchorCertificates(serverTrust, (__bridge CFArrayRef)pinnedCertificates);

            if (!AFServerTrustIsValid(serverTrust)) {
                return NO;
            }

            // obtain the chain after being validated, which *should* contain the pinned certificate in the last position (if it's the Root CA)
            NSArray *serverCertificates = AFCertificateTrustChainForServerTrust(serverTrust);
            
            for (NSData *trustChainCertificate in [serverCertificates reverseObjectEnumerator]) {
                if ([self.pinnedCertificates containsObject:trustChainCertificate]) {
                    return YES;
                }
            }
            
            return NO;
        }
        case AFSSLPinningModePublicKey: {
            NSUInteger trustedPublicKeyCount = 0;
            NSArray *publicKeys = AFPublicKeyTrustChainForServerTrust(serverTrust);

            for (id trustChainPublicKey in publicKeys) {
                for (id pinnedPublicKey in self.pinnedPublicKeys) {
                    if (AFSecKeyIsEqualToKey((__bridge SecKeyRef)trustChainPublicKey, (__bridge SecKeyRef)pinnedPublicKey)) {
                        trustedPublicKeyCount += 1;
                    }
                }
            }
            return trustedPublicKeyCount > 0;
        }
    }
    
    return NO;
}
```

改校验方法分为如下几部分来看：

1. 如果需要交验证书中的域名，则使用 SecPolicyCreateSSL 创建策略，如果不需要，则使用 SecPolicyCreateBasicX509 创建策略。
2. 通过 SecTrustSetPolicies 设置 serverTrust 和 policies。
3. 如果 SSLPinningMode 是 AFSSLPinningModeNone，则使用 AFServerTrustIsValid 来判断证书是否有效，如下所示：

```objc
static BOOL AFServerTrustIsValid(SecTrustRef serverTrust) {
    BOOL isValid = NO;
    SecTrustResultType result;
    __Require_noErr_Quiet(SecTrustEvaluate(serverTrust, &result), _out);

    isValid = (result == kSecTrustResultUnspecified || result == kSecTrustResultProceed);

_out:
    return isValid;
}
```

4. 如果 AFServerTrustIsValid(serverTrust) 和 allowInvalidCertificates 均为 false，则返回 NO。
5. 如果 SSLPinningMode 类型为 AFSSLPinningModeCertificate，则获取 pinnedCertificates 中所有的证书数据，并将其设置为参与校验锚点证书，如果此时判断证书已然不合法，则返回 NO。否则，使用 AFCertificateTrustChainForServerTrust 从 serverTrust 中取出证书链中所有证书，如果本地证书中包含其中任一，则返回 YES。AFCertificateTrustChainForServerTrust 函数实现如下：

```
static NSArray * AFCertificateTrustChainForServerTrust(SecTrustRef serverTrust) {
    CFIndex certificateCount = SecTrustGetCertificateCount(serverTrust);
    NSMutableArray *trustChain = [NSMutableArray arrayWithCapacity:(NSUInteger)certificateCount];

    for (CFIndex i = 0; i < certificateCount; i++) {
        SecCertificateRef certificate = SecTrustGetCertificateAtIndex(serverTrust, i);
        [trustChain addObject:(__bridge_transfer NSData *)SecCertificateCopyData(certificate)];
    }

    return [NSArray arrayWithArray:trustChain];
}
```

6. 如果 SSLPinningMode 类型为 AFSSLPinningModeCertificate，则使用 AFCertificateTrustChainForServerTrust 从 serverTrust 中取出证书链中所有证书，并与 pinnedPublicKeys 中的所有公钥通过 AFSecKeyIsEqualToKey 一一对比）。如果有一个相同则返回 YES。

```objc
static BOOL AFSecKeyIsEqualToKey(SecKeyRef key1, SecKeyRef key2) {
#if TARGET_OS_IOS || TARGET_OS_WATCH || TARGET_OS_TV
    return [(__bridge id)key1 isEqual:(__bridge id)key2];
#else
    return [AFSecKeyGetData(key1) isEqual:AFSecKeyGetData(key2)];
#endif
}
```