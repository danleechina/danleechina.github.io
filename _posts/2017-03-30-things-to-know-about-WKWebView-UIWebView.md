---
layout: single
author_profile: true
title: UIWebView 和 WKWebView
---

公司上一个版本 x.x.x 的 app 发布之后，OOM（Out of Memory） free seesion 的值直线下降。目前维持在 73% 左右。一直在思考为什么这
个值下降的这么严重。直到发现这篇文章，[资料1]((https://code.facebook.com/posts/1146930688654547/reducing-fooms-in-the-facebook-ios-app/))。意识到是 UIWebView 的内存泄漏导致的。

先说一下为什么之前 UIWebView 内存泄漏但是 fabric 显示的 OOM free session 值却并不低。

iOS 8 设备有两个奇怪的 bug

1. [JavaScriptCore bug](https://bugs.webkit.org/show_bug.cgi?id=139654)
2. [UIWebView bug](http://stackoverflow.com/questions/12757237/websafeforwarder-forwardinvocation-crashes)

第一个 bug 已经在 iOS 9 上面解决了（在 iOS 8 设备上，如果您使用了 documentView.webView.mainFrame.javaScriptContext 获取了网页的 JSContext，通过频繁的进入和退出网页界面，这个 crash 是毕现的）。但是第二 bug iOS 10 也还存在。

关于这两个 bug 网上的资料都不多。从搜到的仅有的资料来看，后者很可能和 UIWebView 的内存泄漏有关。至少从 crash 堆栈来看，OC 运行时找不到转发的类。

所以我的想法就很简单了。直接干涉 UIWebView delegate 的方法转发。基于 NSProxy 可以很好的实现一个代理 delegate。具体实现可以看这个 [Smart Proxy Delegation](http://petersteinberger.com/blog/2013/smart-proxy-delegation/)，或者有兴趣可以直接看 AsyncDisplay 和 YYKit 的源代码，里面都有具体的实现，AsyncDisplay 的源代码中注释很丰富，强烈推荐。

换了这种方式之后，这两个 bug 导致的 crash 就再也没出现了（意外的也修复好了在 iOS 8 上的 JavaScriptCore crash）。

但是 OOM free session 值却开始直线下滑了，到现在稳定在 73% 左右。联想到之前这两个 crash 占比达到 80% 左右，我猜想之前的 crash 都转换到了 OOM 上了。

由于 UIWebView 内存泄漏实在太过严重，我决定切换到 WKWebView。

# 如何兼容 iOS 7

切换到 WKWebView 遇到的第一个问题是怎么同时也支持 iOS 7，这是我们的产品经理要求的。

先说一点题外话，苹果设备默认是支持老版本的系统安装老版本的 app 的。也就是说如果您的 app 之前是支持 iOS 7 的，但是下一个版本只支持最低 iOS 8了，那么一个 iOS 7 的用户去 App Store 下载您的 app 的时候，苹果会自动弹窗提醒用户将会安装该 app 的支持 iOS 7的老版本。

也就是说只支持最低 iOS 8设备并不会让你损失 iOS 7 的用户，如果您之前是支持 iOS 7的话。

但是万一真的想支持 iOS 7同时又能实现新的 iOS 特性怎么办？两个步骤：

1. 在项目配置的 build phases 中添加你需要的 framework（比如  WebKit.framework），如果该 framework 不支持您的最低 iOS 版本要求，设置 framework 的 status 为 weak
2. 当你需要用到该 framework 的类的时候（比如 WKWebView），首先在实现文件中导入 framework 的主头文件

    ```
    #import <WebKit/WebKit.h>
    ```
    
    当你要生成 WKWebView 对象的时候，先判断类是否存在，再决定是用 WKWebView 还是 UIWebView
    
    ```
    id webView = nil;
    if([WKWebView class]) {
        webView = [WKWebView new];
    } else {
        webView = [UIWebView new];
    }
    ```
    同时在实现文件中实现 WKWebView 和 UIWebView 的 delegate。实际运行的时候，运行时会调用正确的 delegate 方法（WKWebView 或者 UIWebView）。

# 如何使用 WebView

这个比较简单，我说一下一些有用的实现方式：

1. KVO 监听 `estimatedProgress` 来设置进度条的值
2. KVO 监听 `title` 来设置比如说导航栏标题
3. 通常设置 webView（无论是 UIWebView 还是 WKWebView） 的 delegate 的时候我会用 proxy 进行封装，见资料2
4. 在 UIWebView 里面，我会是用 JavaScriptCore 来让页面的 JS 调用 OC 本地方法
5. 在 WKWebView 里面，我会用 WKWebViewConfiguration 添加一个脚本处理对象给 WKUserContentController，页面 JS 可以通过调用 `window.webkit.messageHandlers.<your handler name>.postMessage({data: data, id: handle});`将数据传给客户端本地。代码如下：

    ```
    WKWebViewConfiguration *config = [WKWebViewConfiguration new];
    config.userContentController = [WKUserContentController new];
    [config.userContentController addScriptMessageHandler:<Your Handler Object> name:@"<your handler Name>"];
    WKWebView *wkWebView = [[WKWebView alloc] initWithFrame:CGRectMake(0,0,0,0) configuration:config];
    ```
    然后在<Your Handler Object>上实现 WKScriptMessageHandler ，处理 js 传过来的数据。参见资料3

# 处理 Cookie

UIWebView 的 cookie 和 NSHTTPCookieStorage 的单例对象是互通的。所以 UIWebView 的 cookie 基本上处理比较简单。NSHTTPCookieStorage 的单例对象会自动将 cookie 添加到相应的 UIWebView 的 request 上。每次加载完页面以后 UIWebView 上生成的 cookie 也会自动保存到 NSHTTPCookieStorage 单例对象。

WKWebView 的 cookie 处理起来就很麻烦了。为了将 NSHTTPCookieStorage 单例对象的 cookie 设置到 WKWebView，需要在 loadRequest 的时候将 cookie 设置到 request 的 http 头上。代码如下：

```
- (void)loadRequest:(NSURLRequest *)req {
    NSMutableURLRequest *request = req.mutableCopy;
    NSString *urlString = request.URL.absoluteString;
    if (urlString && [urlString rangeOfString:@"www.xxx.com"].location != NSNotFound) {
        NSHTTPCookie *cookie = // specific cookie to be set;
        NSArray* cookies = [NSArray arrayWithObjects: cookie, nil];
        NSDictionary * headers = [NSHTTPCookie requestHeaderFieldsWithCookies:cookies];
        [request setAllHTTPHeaderFields:headers];
    }
    [(WKWebView *)self.wkWebView loadRequest:request];
}
```

如果有 AJAX 的 request 的话需要设置脚本（原理就是将本地 cookie 通过 js 设置到页面的 document.cookie）：

```
WKUserContentController* userContentController = WKUserContentController.new;
WKUserScript * cookieScript = [[WKUserScript alloc] 
    initWithSource: @"document.cookie = 'TeskCookieKey1=TeskCookieValue1';document.cookie = 'TeskCookieKey2=TeskCookieValue2';"
    injectionTime:WKUserScriptInjectionTimeAtDocumentStart forMainFrameOnly:NO];
// again, use stringWithFormat: in the above line to inject your values programmatically
[userContentController addUserScript:cookieScript];
WKWebViewConfiguration* webViewConfig = WKWebViewConfiguration.new;
webViewConfig.userContentController = userContentController;
WKWebView * webView = [[WKWebView alloc] initWithFrame:CGRectMake(/*set your values*/) configuration:webViewConfig];
```

由于采用了新的实现机制，WKWebView 获取的 cookie 不会被设置到 NSHTTPCookieStorage 单例对象上。如果想要获取 WKWebView 上面的 cookie，同样可以是用 document.cookie。但是这种方式无法获取到 httpOnly 的 cookie。为了获取到所有的 WKWebView 上的 cookie。可以使用 NSURLProtocol，在 NSURLProtocol 得到调用的时候去取所有的相关 cookie。但是 WKWebView 默认并不走 NSURLProtocol，需要使用私有 API：

```
    Class cls = NSClassFromString(@"WKBrowsingContextController");
    SEL sel = NSSelectorFromString(@"registerSchemeForCustomProtocol:");
    if ([(id)cls respondsToSelector:sel]) {
        // 把 http 和 https 请求交给 NSURLProtocol 处理
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
        [(id)cls performSelector:sel withObject:@"http"];
        [(id)cls performSelector:sel withObject:@"https"];
#pragma clang diagnostic pop
    }

    [NSURLProtocol registerClass:[CustomURLProtocol class]];
```

原理请看[让 WKWebView 支持 NSURLProtocol](https://blog.yeatse.com/2016/10/26/support-nsurlprotocol-in-wkwebview/)

# 参考资料

1. [OOM Crash 分析](https://code.facebook.com/posts/1146930688654547/reducing-fooms-in-the-facebook-ios-app/)
2. [Smart Proxy Delegation](http://petersteinberger.com/blog/2013/smart-proxy-delegation/)
3. [在 WKWebView 上 js 传递数据给 iOS ](http://stackoverflow.com/questions/29249132/wkwebview-complex-communication-between-javascript-native-code)
4. [Set cookie to WKWebView](http://stackoverflow.com/questions/26573137/can-i-set-the-cookies-to-be-used-by-a-wkwebview)
5. [让 WKWebView 支持 NSURLProtocol](https://blog.yeatse.com/2016/10/26/support-nsurlprotocol-in-wkwebview/)
