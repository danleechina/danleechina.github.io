---
layout: single
author_profile: true
title: iOS 生成 SecKeyRef 的正规方式
---

## 前言

针对 macOS 的开发，多年以前苹果就弃用了 OpenSSL，转而推荐自有框架 Security 和 CommonCrypto。当然你仍然可以使用 OpenSSL，比如说在 iOS 上使用开源库 [OpenSSL for iPhone](https://github.com/x2on/OpenSSL-for-iPhone)。

苹果有一套自己的方式来生成各种密钥（对称加密、非对称加密），你可以查看苹果的 sample code [CryptoExercise](https://developer.apple.com/library/content/samplecode/CryptoExercise/Introduction/Intro.html)，来了解如何在苹果自有平台（macOS、iOS、 tvOS 等等）上使用这一套机制。

OpenSSL 是被广泛使用的生成公私钥对以及各类证书等文件的方式，比如生成 PEM 或者 DER 后缀的文件（前者是 Base64 编码，或者则是 DER 编码的内容，如何包含的只是公钥或者私钥的话，本质上就没有区别）。但是在 iOS 并没有原生支持读取只包含公钥或者私钥的方法（iOS 10 之后可以使用 SecKeyCreateWithData 来生成） 。

在 iOS 上， SecKeyRef 对象是一个密码学角度的抽象的密钥对象（也就是说它可以代表一个公钥、私钥或者某种对称加密的密钥）。所以如何生成这样一个对象就显得格外重要，因为无论是加解密还是签名，都会需要这个对象

## 原生生成公私钥对象的一种通用方式 （仅限 iOS 10 及以上）

苹果从 iOS 10 开始支持直接从公私钥数据来生成 SecKeyRef。步骤如下：

1. 对于 PEM 编码的数据，需要先将多余的信息给剔除，主要是头尾两行 （begin 和 end ）以及去掉换行。
2. 构造一个 attribute 属性字典，指定密钥算法（比如 RSA），密钥格式（公钥还是私钥），还有密钥大小
3. 调用 SecKeyCreateWithData，返回一个 SecKeyRef

下面是具体代码：

```
SecKeyRef getPrivateKeyFromPem() {
    // 下面是对于 PEM 格式的密钥文件的密钥多余信息的处理，通常 DER 不需要这一步
    NSString *key = @"PEM 格式的密钥文件";
    NSRange spos;
    NSRange epos;
    spos = [key rangeOfString:@"-----BEGIN RSA PRIVATE KEY-----"];
    if(spos.length > 0){
        epos = [key rangeOfString:@"-----END RSA PRIVATE KEY-----"];
    }else{
        spos = [key rangeOfString:@"-----BEGIN PRIVATE KEY-----"];
        epos = [key rangeOfString:@"-----END PRIVATE KEY-----"];
    }
    if(spos.location != NSNotFound && epos.location != NSNotFound){
        NSUInteger s = spos.location + spos.length;
        NSUInteger e = epos.location;
        NSRange range = NSMakeRange(s, e-s);
        key = [key substringWithRange:range];
    }
    key = [key stringByReplacingOccurrencesOfString:@"\r" withString:@""];
    key = [key stringByReplacingOccurrencesOfString:@"\n" withString:@""];
    key = [key stringByReplacingOccurrencesOfString:@"\t" withString:@""];
    key = [key stringByReplacingOccurrencesOfString:@" "  withString:@""];
    
    // This will be base64 encoded, decode it.
    NSData *data = base64_decode(key);
    if(!data){
        return nil;
    }
    
    // 设置属性字典
    NSMutableDictionary *options = [NSMutableDictionary dictionary];
    options[(__bridge id)kSecAttrKeyType] = (__bridge id) kSecAttrKeyTypeRSA;
    options[(__bridge id)kSecAttrKeyClass] = (__bridge id) kSecAttrKeyClassPrivate;
    NSNumber *size = @2048;
    options[(__bridge id)kSecAttrKeySizeInBits] = size;
    NSError *error = nil;
    CFErrorRef ee = (__bridge CFErrorRef)error;
    
    // 调用接口获取密钥对象
    SecKeyRef ret = SecKeyCreateWithData((__bridge CFDataRef)data, (__bridge CFDictionaryRef)options, &ee);
    if (error) {
        return nil;
    }
    return ret;
}

```

## 原生生成公私钥对象的一种通用方式 （iOS 9 及以前）

针对 iOS 10 以前的版本，想要获取私钥的正规途径是通过 P12（亦即 PKCS #12） 文件获取（P12 是同时包含公私钥的文件，同时需要一个对称密码来使用 p12 文件），步骤也很简单：

1. 读取 p12 文件，当然我不推荐你直接将 p12 文件放在 app bundle 中。你可以硬编码在代码中，会安全一丢丢。
2. 设置参数字典，主要是设置你在导出 p12 文件时候设置的密码。
3. 调用 SecPKCS12Import 导出 p12 文件包含的 item 数组
4. 获取 item 数组第一个元素的字典，其中 kSecImportItemIdentity 键对应的是值也就是 SecIdentityRef 对象
5. 从 SecIdentityRef 中拷出私钥对象

如果要拷出公钥，稍微有点不一样：

1. 上面步骤中获得 item 数组第一个元素的字典，其中 kSecImportItemTrust 键对应的是值也就是一个 Trust 对象。
2. 调用 SecTrustCopyPublicKey 获取公钥对象

下面是代码解释：

```
NSString *resourcePath = [[NSBundle mainBundle] pathForResource:@"rsaPrivate" ofType:@"p12"];
NSData *p12Data = [NSData dataWithContentsOfFile:resourcePath];

NSMutableDictionary * options = [[NSMutableDictionary alloc] init];

SecKeyRef privateKeyRef = NULL;
id publicKey = NULL;

// 改成你设置的密码
[options setObject:@"" forKey:(id)kSecImportExportPassphrase];
CFArrayRef items = CFArrayCreate(NULL, 0, 0, NULL);
OSStatus securityError = SecPKCS12Import((CFDataRef) p12Data, (CFDictionaryRef)options, &items);

if (securityError == noErr && CFArrayGetCount(items) > 0) {
    // 获取一个 Identity 对象
    CFDictionaryRef identityDict = CFArrayGetValueAtIndex(items, 0);
    
    // 获取私钥
    SecIdentityRef identityApp = (SecIdentityRef)CFDictionaryGetValue(identityDict, kSecImportItemIdentity);
    securityError = SecIdentityCopyPrivateKey(identityApp, &privateKeyRef);
    if (securityError != noErr) {
        privateKeyRef = NULL;
    }
    
    // 获取一个 Trust 对象
    SecTrustRef trustRef = (SecTrustRef)CFDictionaryGetValue(identityDict, kSecImportItemTrust);
    // 获取公钥
    publicKey = (__bridge_transfer id)SecTrustCopyPublicKey(trustRef);
}
CFRelease(items);
```

## 从证书文件读取公钥对象

从证书文件读取公钥对象步骤如下：

1. 读取证书文件生成一个 Certificate 对象（SecCertificateRef 类型）
2. 从 Certificate 对象获取一个 Trust 对象 （SecTrustRef 类型）
3. 从 Trust 对象拷贝出公钥 （这一步可以先根据 Trust 对象来判断证书是否可信）

代码解释如下：

```
id publicKey = nil;
SecCertificateRef certificate;
SecCertificateRef certificates[1];
CFArrayRef tempCertificates = nil;
SecPolicyRef policy = nil;
SecTrustRef trust = nil;
SecTrustResultType result;

certificate = SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certificateDate);
if (certificate) {
    certificates[0] = certificate;
    tempCertificates = CFArrayCreate(NULL, (const void **)certificates, 1, NULL);
    policy = SecPolicyCreateBasicX509();
    SecTrustCreateWithCertificates(tempCertificates, policy, &trust);
    SecTrustEvaluate(trust, &result);
    // 获得公钥对象
    publicKey = (__bridge_transfer id)SecTrustCopyPublicKey(trust);
}

if (trust) {
    CFRelease(trust);
}

if (policy) {
    CFRelease(policy);
}

if (tempCertificates) {
    CFRelease(tempCertificates);
}

if (certificate) {
    CFRelease(certificate);
}
```

## 如何从密钥文件生成 P12 和证书等

你可以参考这个 SO [How can I get SecKeyRef from DER/PEM file](https://stackoverflow.com/questions/10579985/how-can-i-get-seckeyref-from-der-pem-file)

## 开源库

[Objective-C-RSA](https://github.com/ideawu/Objective-C-RSA) 这个开源库源码解释了如何自己处理 PEM 格式密钥文件的头，但是由于解析力不够强，经常会返回一个空的密钥对象。所以必要时候可以参考一下。但是不太推荐。笔者对 ASN.1 等概念不太熟悉，这里不过多讨论了。

## 使用 OpenSSL

使用 OpenSSL 无法生成 SecKeyRef 密钥对象，但是 OpenSSL 提供了完整的密码学各类操作支持（加密，加签，解密，验签等），所以你完全可以不需要苹果的 Security 框架。你可以参考 [支付宝的 Demo](https://doc.open.alipay.com/docs/doc.htm?spm=a219a.7629140.0.0.ycHYn0&treeId=54&articleId=104509&docType=1)，了解如何使用该库。开源代码地址是 [OpenSSL for iPhone](https://github.com/x2on/OpenSSL-for-iPhone)


## iOS 上关于加密等密码学操作的建议

苹果的官方文档 [苹果 Security 框架文档](https://developer.apple.com/documentation/security) 完整的描述了如何在苹果自有平台使用 Security 框架。你可以参考它。

主要是理解几个对象：（文档地址 [Certificate, Key, and Trust Services](https://developer.apple.com/documentation/security/certificate_key_and_trust_services)）

1. certificate 对象
2. identity 对象
3. trust 对象
4. key 对象
5. policy 对象

## 引用

1. [OpenSSL for iPhone](https://github.com/x2on/OpenSSL-for-iPhone)
2. [苹果 sample code CryptoExercise](https://developer.apple.com/library/content/samplecode/CryptoExercise/Introduction/Intro.html#//apple_ref/doc/uid/DTS40008019-Intro-DontLinkElementID_2)
3. [Swift 对称密码使用法](https://digitalleaves.com/blog/2015/08/commoncrypto-in-swift/)
4. [Swift 非对称密码使用法](https://digitalleaves.com/blog/2015/10/asymmetric-cryptography-in-swift/)
5. [iOS 上的公钥 VS OpenSSL 上的公钥对比](https://digitalleaves.com/blog/2015/10/sharing-public-keys-between-ios-and-the-rest-of-the-world/)
6. [密钥文件等的区别和转换](https://support.ssl.com/Knowledgebase/Article/View/19/0/der-vs-crt-vs-cer-vs-pem-certificates-and-how-to-convert-them)
7. [iOS 上的 SHA256 with RSA VS JAVA 平台](https://stackoverflow.com/questions/42689108/rsa-sha256-signing-in-ios-and-verification-on-java)
8. [P12、证书文件的生成](https://stackoverflow.com/questions/10579985/how-can-i-get-seckeyref-from-der-pem-file)
9. [苹果 Security 框架文档](https://developer.apple.com/documentation/security)
10. [Objective-C-RSA 开源库](https://github.com/ideawu/Objective-C-RSA)
