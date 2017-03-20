---
layout: single
author_profile: true
title: 对 iOS app 进行安全加固
---

总所周知，运行在越狱设备上的 iOS app，非常容易遭到破解分析，这里我列举一些可以加大破解难度的方法，希望有所帮助。

# 一些实用手段

## 防止 tweak 依附

通常来说，我们要分析一个 app，最开始一般是砸壳，

```
$ DYLD_INSERT_LIBRARIES=dumpdecrypted.dylib /path/to/XXX.app/XXX
```

然后将解密之后的二进制文件扔给类似 hopper 这样的反编译器处理。直接将没有砸壳的二进制文件扔个 hopper 反编译出来的内容是无法阅读的（被苹果加密了）。所以说砸壳是破解分析 app 的第一步。对于这一步的防范，有两种方式。

1. 限制二进制文件头内的段
    通过在 Xcode 里面工程配置 build setting 选项中将

    ```
    -Wl,-sectcreate,__RESTRICT,__restrict,/dev/null
    ``` 
    
    添加到 "Other Linker Flags"（注意这里我在项目中碰到了一个 问题，在 iPod touch iOS 9.3 的设备上，使用了 swift 的项目会导致莫名奇妙的 swift 标准库无法找到，而在 iOS 10 的设备上没有这个问题。之前并没有以为是因为添加了这个的原因，直到网上搜了所有解决方案，比如这个 [SO Post](http://stackoverflow.com/questions/26024100/dyld-library-not-loaded-rpath-libswiftcore-dylib) 都没有效果的时候，我才发现是这个设置的原因）

2. setuid 和 setgid （Apple 不接受调用这两个函数的 app，因为它可以通过查看符号表来判断您的二进制运行文件是否包含这两个函数）

具体原理可以查看参考资料1，2

## 检测越狱设备上是否有针对性 tweak

一般来说在越狱手机上，我们会使用 TheOS 创建 tweak 类型的工程。然后针对我们要分析的类，使用提供的 logify.pl 命令生成的 mk 文件来打印该类所有方法的入参和出参。这对分析 app 的运行方式有很大的帮助。当然，我们也可以自己创建某个类的 mk，来 hook 某个函数，让它以我们想要的方式运行，比如说对于一些做了证书绑定的 app，如果它用的框架是 `AFNetWorking` 的话，那么我们可以创建一个 mk 文件，hook `AFSecurityPolicy` 类的下列方法：

```
- (BOOL)evaluateServerTrust:(SecTrustRef)serverTrust forDomain:(NSString *)domain
```

让这个方法永远返回 `YES`，那么大多数的应用所做的证书绑定也就失效了。用过 TheOS 的 tweak 模版的话，你会发现这种方式相当简单快速。

对于这一步的防范，可以在工程的 main 函数里面加入一层判断，首先读取 `/Library/MobileSubstrate/DynamicLibraries` 下所有的 plist 文件的内容，查看是否某个 plist 含有你的 app 的 bundle id，是的话，可以判定有人想利用 tweak 攻击你的 app，这时候你可以采取比如说将 app 给 crash 掉，或者限制某些功能等方式来应对。

具体原理可以查看参考资料4，简单来说，就是 MobileSubstrate 在 app 加载到内存的时候会先去检查 `/Library/MobileSubstrate/DynamicLibraries` 下面是否有需要加载的 tweak，有的话就加载，怎么判断有没有？就是根据 plist 里面的 bundle ID 判断的。

代码参考如下：

```
static __inline__ __attribute__((always_inline)) int anti_tweak()
{
    uint8_t lmb[] = {'S', 'u', 'b', 's', 't', 'r', 'a', 't', 'e', '/', 'D', 'y', 'n', 'a', 'm', 'i', 'c', 0, };
    NSString *dir = [NSString stringWithFormat:@"/%@/%@%s%@", @"Library", @"Mobile", lmb, @"Libraries"];
    NSArray *dirFiles = [[NSFileManager defaultManager] contentsOfDirectoryAtPath:dir error:nil];
    NSArray *plistFiles = [dirFiles filteredArrayUsingPredicate:
                           [NSPredicate predicateWithFormat:
                            [NSString stringWithFormat:@"%@ %@%@ '.%@%@'",@"self", @"EN", @"DSWITH", @"pli", @"st"]]];
    int cnt = 0;
    for (NSString *file in plistFiles) {
        NSString *filePath = [dir stringByAppendingPathComponent:file];
        NSString *fileContent = [NSString stringWithContentsOfFile:filePath encoding:NSUTF8StringEncoding error:nil];
        if (fileContent && [fileContent rangeOfString:[[NSBundle mainBundle] bundleIdentifier]].location != NSNotFound) {
            cnt ++;
        }
    }
    // 返回有针对本 app 的 tweak 数量，为 0 说明没有
    return cnt;
}

```

## 防 http 抓包

通常破解一个 app，我们会抓包。这样的话，我们的 app 所有接口，接口数据都会暴露在逆向人员的眼皮底下。这时候，我们可以限制 http 抓包。方式很简单，就是将 NSURLSessionConfiguration 的 connection​Proxy​Dictionary 设置成空的字典，因为这个属性就是用来控制会话的可用代理的。可用参见官方文档，也就是参考资料5。下面是对于 `AFNetWorking` 的使用方法：

```
// 继承 AFHTTPSessionManager，重写下列方法
- (instancetype)initWithServerHost:(PDLServerHost*)serverHost {
#ifdef DEBUG
    // debug 版本的包仍然能够正常抓包
    self = [super initWithBaseURL:serverHost.baseURL];
#else
    NSURLSessionConfiguration *conf = [NSURLSessionConfiguration ephemeralSessionConfiguration];
    conf.connectionProxyDictionary = @{};
    self = [super initWithBaseURL:serverHost.baseURL sessionConfiguration:conf];
#endif
    return self;
}
```

但是由于 OC 方法很容易被 hook，避免抓包是不可能的，所以，个人认为最好的方式是对请求参数进行加密（最好是非对称加密，比如 RSA）

## 混淆（或者加密）硬编码的明文字符串

对于被砸壳的二进制文件，逆向分析人员分析代码有一条重要线索，也就是被硬编码的明文字符串。比如说，你的 app 被人抓包了，某些数据请求接口也被人发现了，那么很简单，逆向人员可以直接拷贝特征比较明显的字符串到 hopper 中搜索，通过查看该字符串被引用的地方，可以很快的找到相应的逻辑代码。

对于这一步的防范，需要做的就是对硬编码的明文进行加密或混淆。 有个开源代码可以用，[UAObfuscatedString](https://github.com/UrbanApps/UAObfuscatedString)，但是这个开源混淆代码写出来的字符串是相当长的（也就是麻烦），同时不支持加密。最近我写了一个工具，可以在编译期间加密所有代码中的明文字符串，在 app 运行的时候解密字符串。这个工具的特点如下：

1. 简单，开发人员可以硬编码明文字符串，所有的加密会在编译开始时自动处理
2. 可以自定义加密或者混淆方式，（为了不影响 app 运行效率，需要提供一个简单快速的加密或混淆方式）提高解密难度

项目地址 [MixPlainText](https://github.com/danleechina/mixplaintext)

## 使用 Swift 开发

Swift 是目前比较新的开发 iOS 语言，由于 Swift 目前还不是很稳定，越狱开源社区对这个的支持也不是很即时，比如说 class-dump 工具目前就不支持含有 Swift 的二进制文件。 TheOS 也是最近才开始支持 Swift，但是还没有加到主分支上（可以参见 [Features](https://github.com/theos/theos/wiki/Features)）。所以目前来看，至少 Swift 可能比纯 OC 的工程要安全一点点。当然，等 Swift 日趋稳定，以及越狱开源社区的逐渐支持，这一点优势可能就不明显了。

## 使用静态内连 C 函数

由于 OC 语言的动态性，导致 OC 的代码是最容易被破解分析的。在安全性上，更推荐使用 C 语言写成的函数。但是 C 语言的函数也是可以被 hook 的，主要有3种方式：

1. 使用 Facebook 开源的 [fishhook](https://github.com/facebook/fishhook)
2. 使用 MobileSubstrate 提供的 hook C 语言函数的方法

    ```
    void MSHookFunction(void* function, void* replacement, void** p_original);
    ```
    
3. 使用 mach_override，关于 mach_override 和 fishhook 的区别请看 [mach_override  和 fishhook 区别](http://stackoverflow.com/questions/17832031/technical-differences-between-mach-override-and-fishhook)

由于上面这三种方式可以 hook C 函数。要想不被 hook 解决方法是使用静态内联函数，这样的话需要被 hook 的函数没有统一的入口，逆向人员想要破解只能去理解该函数的逻辑。

## 使用 block

严格来说使用 block 并不能很大程度提高安全性，因为逆向人员只要找到使用该 block 的方法，一般来说在其附近就会有 block 内代码的逻辑。具体查找方法和原理可以看参考资料6。

但是个人认为使用 block 的安全性是比直接使用 oc 方法是要高的。在我的逆向分析 app 的经验中，对于使用了 block 的方法，目前我还不知道到怎么 hook （有知道的话，可以在 blog 上提个 issue 告诉我，先谢过 🙏），同时对于含有嵌套的 block 或者是作为参数传递的 block，处理起来就更加复杂了。所以，如果能将内敛 C 函数，嵌套 block ， block 类型参数组合起来的话，安全性应该是会有一定提升。

## 代码混淆

代码混淆的方式有几种：

1. 添加无用又不影响逻辑的代码片段，迷糊逆向人员
2. 对关键的类、方法，命名成与真实意图无关的名称

对于第二种，目前有一些自动化工具，比如念茜提到的一个工具参见参考资料7。

个人认为最好的一个加密混淆工具是 [ios-class-guard](https://github.com/Polidea/ios-class-guard)，不过目前这个项目已经停止维护了。但是这种方式的混淆我觉得才是最终极的方案。

## 其他方法

比如 ptrace 反调试等（不过据说已经可以很容易被绕过）

```
// see http://iphonedevwiki.net/index.php/Crack_prevention for detail
static force_inline void disable_gdb() {
#ifndef DEBUG
    typedef int (*ptrace_ptr_t)(int _request, pid_t _pid, caddr_t _addr, int _data);
#ifndef PT_DENY_ATTACH
#define PT_DENY_ATTACH 31
#endif
    // this trick can be worked around,
    // see http://stackoverflow.com/questions/7034321/implementing-the-pt-deny-attach-anti-piracy-code
    void* handle = dlopen(0, RTLD_GLOBAL | RTLD_NOW);
    ptrace_ptr_t ptrace_ptr = dlsym(handle, [@"".p.t.r.a.c.e UTF8String]);
    ptrace_ptr(PT_DENY_ATTACH, 0, 0, 0);
    dlclose(handle);
#endif
}
```


## 注意事项

通过了解 OC 的运行时特性和 mach-o 二进制文件的结构，借助现有的工具，你会发现 hook 方法 是很简单就能完成的。虽然上面我提到了一些提高安全性的几个方案，但是，所有这些方式只是增加了逆向人员的逆向难度，并不能让 app 变的坚不可摧。不过采取一定的措施肯定比什么措施都不采取来的安全，你说呢？

# 参考资料

1. [防止 tweak 依附](http://bbs.iosre.com/t/tweak-app-app-tweak/438)
2. [Blocking Code Injection on iOS and OS X](https://pewpewthespells.com/blog/blocking_code_injection_on_ios_and_os_x.html)
3. [Crack prevention](http://iphonedevwiki.net/index.php/Crack_prevention)
4. [Cydia Substrate](http://iphonedevwiki.net/index.php/Cydia_Substrate)
5. [connection​Proxy​Dictionary](https://developer.apple.com/reference/foundation/nsurlsessionconfiguration/1411499-connectionproxydictionary?language=objc)
6. [iOS符号表恢复&逆向支付宝](http://blog.imjun.net/2016/08/25/iOS%E7%AC%A6%E5%8F%B7%E8%A1%A8%E6%81%A2%E5%A4%8D-%E9%80%86%E5%90%91%E6%94%AF%E4%BB%98%E5%AE%9D/)
7. [iOS安全攻防（二十三）：Objective-C代码混淆](http://blog.csdn.net/yiyaaixuexi/article/details/29201699)
8. [iOS安全攻防（二十四）：敏感逻辑的保护方案（1）](http://blog.csdn.net/yiyaaixuexi/article/details/29210413)