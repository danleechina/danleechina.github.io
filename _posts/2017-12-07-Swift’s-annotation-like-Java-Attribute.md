---
layout: single
author_profile: true
title: Swift 中类似 Java 的注解：Attribute
---

## 前言

本文将描述 Java 中使用注解（`annotation`）的优势及原理（但是不会介绍 Java 注解的使用和自定义，你可以网上搜索相关资料），以及类似 Java 注解的 Swift Attribute，同时还会思考如果 Swift 支持自定义 Attribute 会有什么好处。

## Java 使用注解的优势

举例来说，Java 的 Web 框架 `SpringMVC`，这个框架的核心思想之一是使用 Java 的注解来简化 Web 工程配置。只需很少的代码就可以构造一个 Restful API。如下所示（代码引用来自 [Building a RESTful Web Service](https://spring.io/guides/gs/rest-service/) 官方 demo）：

```
package hello;

import java.util.concurrent.atomic.AtomicLong;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class GreetingController {

    private static final String template = "Hello, %s!";
    private final AtomicLong counter = new AtomicLong();

    @RequestMapping("/greeting")
    public Greeting greeting(@RequestParam(value="name", defaultValue="World") String name) {
        return new Greeting(counter.incrementAndGet(),
                            String.format(template, name));
    }
}
```

这里简单的几句代码，就完成了一个 `/greeting` API 的所有编码及配置（当然，这里忽然了 Model 类）。

`SpringMVC` 的优势之一是避免了像 Struts2 一样要配置和维护 XML 文件。所以用户可以不在过问 XML 文件，使用纯 Java 语言来完成所有开发及配置。

## Java 注解原理

Java 编译器在将源码编译成字节码的同时，会处理注解符号并附加到字节码的结构中。编译出来的文件大致包含类、类的方法、类的成员变量这三种信息，对于添加了注解的元素，编译器会同时在相应的元素上添加注解信息。

当 JVM 开始加载字节码文件的时候，会去读取类、类的方法、类的成员变量这些信息，当发现读取的信息中包含注解信息的时候则去处理这些信息。对待注解信息的逻辑有两种，一个是类加载完成以后丢弃，一个是类加载完成以后不丢弃。当然在编译阶段编译器也可以处理注解信息，根据注解信息对用户输出编译警告或者错误，所以注解的处理逻辑事实上有三种。

从这里我们可以发现注解的实现实际上需要两种支持：

1. 编译器的支持。这样我们才能将注解信息加入到编译输出的文件（对 Java 来说是字节码，对 c/c++/objC 来说是 object 文件）
2. 类加载支持。根据编译输出的文件结构加载类的时候，同时读取注解信息，根据注解信息进行相应的处理

## Swift 的注解： Attribute

事实上很多其他语言也有类似 Java 注解的这种东西，比如 c# 和 Swift 的 Attribute。

Swift 的 Attribute 使用在 Swift 的开发中其实也是非常普遍的，比如下面这个 AppDelegate：

```
import UIKit

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {

    var window: UIWindow?


    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {
        return true
    }
}
```

通过使用 `@UIApplicationMain` 编译器就会知道下面的 `AppDelegate` 这个类就是 app 启动的时候系统需要调用的 app 代理。联想一下 ObjC 中我们是这样使用 `AppDelegate`

```
int main(int argc, char *argv[])
{
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

所以在相应的类定义前面使用 `@UIApplicationMain`，我们就不需要自己告诉 iOS 系统该 app 的代理是哪个类。
并且如果你定义的 `AppDelegate` 没有实现 `UIApplicationDelegate` 的话，编译器在编译阶段就会展示一个编译错误给你，提示你应该去实现这个协议。

## Swift Attribute 的使用及缺点

Swift 语言本身定义了很多 Attribute 给开发人员使用，你可以查看[官方文档](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Attributes.html#//apple_ref/doc/uid/TP40014097-CH35-XID_460)了解使用详情。

网上有人也整理了一份 Swift 的所有 Attribute 以及说明，[swift attributes 符号](http://andelf.github.io/blog/2014/06/06/swift-attributes/)，当然随着 Swift 的发展可能会有越来越多的 Attribute 加入，原有的 attribute 也可能有些变化。


目前 Swift Attribute 的缺点是无法支持自定义，也就是说我们无法像 Java 一样使用注解。官方提供的 Attribute 的效果非常有限。有可能随着 Swift 的发展后面会加入对自定义 Attribute 的支持，毕竟 Swift 本身是已经支持 Attribute 的。关于 Swift 支持自定义 Attribute 你可以看 [Swift 关于自定义 Attribute 实现的讨论](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151221/004545.html)

## 如果 Swift 支持自定义 Attribute

让我们来想一想如果 Swift 可以支持自定义注解的话有什么好的？ Medium 上面有人已经这样设想了，你可以看 [Custom Attributes in Swift Language](https://medium.com/@azamsharp/custom-attributes-in-swift-language-5fa24030b3d0) 来了解。

但是对我来说，Swift 自定义 Attribute 的最大一个好处是，会出现类似 [`SpringMVC`](https://spring.io/) 这样的框架，用于支持 Swift 的后端开发，以及革新移动端开发方式。

一方面，很多人开始将 Swift 用于后端开发，类似 SpringMVC 这样的 Swift Web 框架也一定会出现。

另一方面，现在很多移动端开始支持 Router，也就是移动 app 里面页面的跳转是基于 URL 的，这样的开源代码很多比如 [JLRoutes](https://github.com/joeldev/JLRoutes)，但是基于 URL 的跳转的缺陷是需要配置 URL 跳转逻辑。我们是否可以想象一下，移动端的开发以后也可以像上面我举例的 SpringMVC demo 一样，通过简单的使用注解，当一个 URL 请求来的时候新的框架能够自动帮我们解析，并跳转到该界面，省去了配置 URL 和解析 URL？

## 总结

Java 注解的使用可以简化项目的配置，因为编译器和类加载机制可以根据注解来帮助我们完成这一部分配置，使得开发人员可以专注于语言本身。

另一方法 Swift 的 Attribute 作用则非常有限，但是由于 Swift 本身是支持 Attribute 的，如果未来 Swift 能支持自定义 Attribute 的话，想象空间还是很大的。

## 引用

1. [Building a RESTful Web Service](https://spring.io/guides/gs/rest-service/)
2. [Swfit 语言官方文档 - Attribute](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Attributes.html#//apple_ref/doc/uid/TP40014097-CH35-XID_460)
3. [Swift 关于自定义 Attribute 实现的讨论](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151221/004545.html)
4. [Java 注解机制及其原理](http://blog.csdn.net/wangyangzhizhou/article/details/51698638)
5. [Java中的注解是如何工作的？](http://www.importnew.com/10294.html)
6. [深入理解 Swift 派发机制](https://kemchenj.github.io/2016-12-25-1/)
7. [swift attributes 符号](http://andelf.github.io/blog/2014/06/06/swift-attributes/)
8. [Custom Attributes in Swift Language](https://medium.com/@azamsharp/custom-attributes-in-swift-language-5fa24030b3d0)
9. [SpringMVC](https://spring.io/)
10. [JLRoutes](https://github.com/joeldev/JLRoutes)