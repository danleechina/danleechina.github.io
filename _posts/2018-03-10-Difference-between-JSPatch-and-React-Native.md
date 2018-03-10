---
layout: single
author_profile: true
title: JSPatch 和 React Native 的区别
---

## 前言

去年的3月份，很多苹果开发者可能都收到了苹果对于热更新的警告邮件。收到邮件的很可能代码中使用了类似 JSPatch 这样的热修复框架。

同时很多没有使用 JSPatch 等类似框架的开发者可能也收到了这份警告邮件，这很可能是苹果的代码检测机制的误伤。不过这也引起了很多使用 React Native 这类框架的开发者的担忧，是不是说以后也不能使用 React Native 这类框架了？

答案当然是可以继续使用 React Native 了。我这里会详细说明一下为什么，主要从两者的原理和能力来阐述。

## JSPatch 的原理

JSPatch 的代码非常精炼。核心文件就两个，一个是 `JSPatch.js`，另外一个是 `JSEngine.m`。实现主要依赖两个：

1. Objective C 语言的动态属性
2. JavaScriptCore 通过 JS 调用 Native 的能力

JSPatch 的核心文件也分别对应了这两者。`JSEngine.m` 主要是定义一套原生接口给 `JSPatch.js` 使用，这套原生接口使用了 OBjective C 语言的动态能力。`JSPatch.js` 则是定义了一套 JS 接口给开发者使用，开发者可以基于这套接口写热更新代码。`JSPatch.js` 定义的这套接口相当于一层 DSL。JSPatch 的 GitHub 上的文档详细介绍了这套 [DSL JSPatch-基础用法](https://github.com/bang590/JSPatch/wiki/JSPatch-%E5%9F%BA%E7%A1%80%E7%94%A8%E6%B3%95)

结合两者使得 JSPatch 的能力十分强大，可以通过 JS 任意调用原生 API。

## React Native 的能力

React Native 是一个非常庞大的项目，我这边暂时还没有去详细阅读过源码。我主要讲一下 React Native 是如何和原生交互的。关于更详细的描述可能查看 [官方文档](http://facebook.github.io/react-native/docs/native-modules-ios.html)

React Native 主要有两种与原生代码交互的途径。官方文档将其描述成 `Native Modules` 和 `Native UI Components`。

1. `Native Modules`，说明如何将原生的代码（与 UI 无关）暴露给 RN 的 JS 代码
2. `Native UI Components`，说明如何将原生的 UI 组件暴露给 RN 的 JS 代码

注意，这两者都需要写原生代码，同时重新打包运行，这样暴露出来接口才能被 JS 代码调用执行。也就是说这里没有热更新。

与原生交互，我们比较关注

1. JS 如何调用原生代码并取得返回值
2. 原生代码如何调用 JS 代码并获得返回值

对于前者，官方文档已经详细的阐述了。主要是 Callback、Promises。

对于后者，官方文档中说的并不详细，需要将文档中的手段结合起来使用。总的来说比较复杂。

总结一下 RN，本质上来说 RN 是自成一体的，我们只能在 RN 的能力范围内具备热更新能力。RN 的能力主要来自两方面：

1. RN 的原生代码库，这套代码库使得我们可以基于此编写 JS 的 UI，进行网络请求以及其他一些原生服务，比如地理位置服务、推送等。这些方面都能进行热更新。RN 的官方文档也几乎全是关于这方面的内容
2. 我们自己编写的并暴露给 RN 的原生代码，可以基于此编写相应地  JS 代码，RN 的官方文档详细的描述了这一方面的规范。在我们自己的 RN 原生模块范围内能够进行热更新


## 总结

通过上面的描述，我们知道了：

1. JSPatch 很强大，可以通过 JS 任意调用原生 API。所以 JSPatch 的热更新能力是全面的。
2. React Native 也很强大，但是其强大是**克制**的。其提供的热更新能力基本上只是写界面，编写一些通常的业务逻辑。

## 参考资料

1. [JSPatch 关于苹果热更新警告邮件的 issue](https://github.com/bang590/JSPatch/issues/746)
2. [React Native 关于苹果热更新警告邮件的 issue](https://github.com/facebook/react-native/issues/13011)
3. [关于苹果警告 JSPatch 作者回复](http://blog.cnbang.net/internet/3374/)
4. [接入 JSPatch FAQ](https://jspatch.com/Docs/appleFAQ)
5. [JSPatch实现原理详解](http://blog.cnbang.net/tech/2808/)
6. [iOS 动态更新方案对比:JSPatch vs React Native](http://blog.cnbang.net/tech/3237/)
7. [React Native通信机制详解](http://blog.cnbang.net/tech/2698/)
8. [Native Modules](http://facebook.github.io/react-native/docs/native-modules-ios.html)