---
layout: single
author_profile: true
title: 关于 `-canOpenURL:` 和 `-openURL:` 的那些事
---

最近工作中需要知道用户的 iPhone 上都安装了哪些应用，所以查阅了一些资料。总的来说主要还是通过使用 Apple 提供的 `- canOpenURL:` API 来测试相应 APP 的 scheme 是否有效来实现。

## 如何知道某个 APP 支持的所有 scheme
很显然，为了知道对应的 APP 是否在设备上，我们需要知道这个 APP 支持的所有 scheme（如果 APP 没有定义任何自己的 scheme，请看[使用私有 API ](#private-api)）。这里我总结了一下大概有两种方法。

* 使用别人已经整理好的scheme。比如这个[wiki](http://wiki.akosma.com/IPhone_URL_Schemes)页面，里面收录了一些比较有名的 APP 的 scheme。当然还有这个[网站](http://handleopenurl.com/)，你可以在这里查询 scheme 对应的 APP，你还可以帮助将你知道的 APP 的 scheme 添加进去，方便后面的人搜索。
* 自己动手查看 APP 的 scheme。主要分下面几步：
	1. 获取 IPA 包。这里你可以使用 iTunes 下载 APP。 然后在`~/Music/iTunes/iTunes Media/Mobile Applications`这个目录下可以找到 iTunes 为我们下载的 IPA 包。
	2. 将 IPA 包的扩展名改成 zip，这样我们就可以解压了。
	3. 解压之后打开`Payload`文件夹，右键扩展名为 app 的文件夹，选择 `显示包内容` 就可以查看到内部的文件了
	4. 找到 `Info.plist` 文件，搜索 `CFBundleURLSchemes` 就可以看到这个 APP 给自己定义的 scheme 了。

## 如何使用 `-canOpenURL:`
iOS9 上面出于对用户隐私的保护，开发人员不能像以前一样随便 `-canOpenURL:` 了。 通常在 iOS9 及以后版本的系统中，使用 `-canOpenURL:` 分为如下几步。

* 点击工程文件，选择相应的 `Target`，点击 `Info` 选项，展开 `Custom iOS Target Properties`
* 找到或者添加 `LSApplicationQueriesSchemes` 到属性列表中。同时在这个数组下面添加相应的 scheme

当然你也可以直接编辑项目的 `Info.plist` 文件，在 `LSApplicationQueriesSchemes` 键下面添加你在 APP 中会使用到的 scheme。

通过将 scheme 添加到白名单中，我们就可以在代码中使用 `-canOpenURL:` 来测试对应 scheme 的 APP 是否被安装。如果你没有将 scheme 添加到白名单中却在 `-canOpenURL:` 中使用了，你的查询会失败，同时控制台会打印类似下面这样的信息：

```
-canOpenURL: failed for URL: "fb://" - error: "This app is not allowed to query for scheme fb"
-canOpenURL: failed for URL: "twitter://" - error: "This app is not allowed to query for scheme twitter"
```
正常的输出是这样的（如何 `-canOpenURL:` 测试 scheme 失败的话，成功的话控制台默认不会有任何输出
)：

```
-canOpenURL: failed for URL: "fb://" - error: "(null)"
```

这里或许有个疑问就是为什么这样就能保护用户隐私呢？ 毕竟我们可以添加很多的我们会测试的 scheme 到白名单中。 如果你无限制的添加 scheme 到白名单中， app store 审核时候可能会觉得你滥用了这个机制，从而不让你的 APP 审核通过。

要注意的是使用 iOS8 sdk 编译的代码跑在运行 iOS9 系统的设备上是没有问题的，但是一个 APP 可以测试的 scheme 有数量的限制，大概是50条。

## 关于 iOS9 上的 `-openURL:` 
我测试了一下，在没有添加 scheme 到白名单的情况下，使用 `-openURL:` 会默认打开相应的应用，同时该函数返回 `true`（如果设备安装了），否则函数返回 `false`(也不会有任何提醒)。 而在添加了 scheme 到白名单的情况下，使用 `-openURL:` 会弹窗提示(有时候又没有弹窗😭，难道是第一次选择是跳转以后后面都默认跳转吗？ 从这点可以看出 Apple 这个 API 做的用户体验并不好)用户是否需要打开相应的应用，选择是则打开，否则不打开，同时函数也会正确的返回 `true` 或者 `false`

## <a name="private-api"></a>使用私有 API
私有 API 也就是 Apple 提供的 iOS SDK 中没有暴露给外部的接口类和方法。我们知道 OC 函数调用是运行时决定的，利用这个特性，通过一些手段获取到 Apple 的一些私有 API 的类名和方法名。我们就可以很容易的使用这些 API。对于本文主题，GitHub 上有一个很好的例子，大家可以下载这个 [demo](https://github.com/wujianguo/iOSAppsInfo)，了解如何使用私有 API 来获取设备上支持的所有 scheme，以及所有安装的 APP 信息。

需要注意的是， 如果你在项目中使用了私有 API，那么你可能需要做一些函数调用方式上的处理，以避免在 APP 审核的时候被发现导致被拒。

关于私有 API 更多的介绍可以看这个 StackOverflow 回答 [What exactly is a private api and why will apple reject an ios app if one is us](http://stackoverflow.com/questions/17580251/what-exactly-is-a-private-api-and-why-will-apple-reject-an-ios-app-if-one-is-us)

关于 Apple 是如果检测应用是否使用了私有 API可以看这里 [How does apple know you are using private api](http://stackoverflow.com/questions/2842357/how-does-apple-know-you-are-using-private-api)

## 参考资料

* [App Programming Guide for iOS(Inter-App Communication)](https://developer.apple.com/library/ios/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/Inter-AppCommunication/Inter-AppCommunication.html#//apple_ref/doc/uid/TP40007072-CH6-SW2)
* [ UIApplication Class Reference](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UIApplication_Class/#//apple_ref/occ/instm/UIApplication/canOpenURL:)
* [Querying URL Schemes with canOpenURL](http://useyourloaf.com/blog/querying-url-schemes-with-canopenurl/)
* [私有 API 相关文档](https://github.com/nst/iOS-Runtime-Headers)
