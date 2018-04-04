---
layout: single
author_profile: true
title: iOS 和 Android 开发的 React Native 入门指南
---

## 前言

这一篇是给稍微有点原生（iOS 或者 Android）编程经验的人的一个系统性的 React Native 入门指南。主要总结的是我之前系统学习 React Native 的经验。

我在很早的时候就接触了 RN，但是刚开始那段时间基本处于一种瞎写的状态，不知道很多内在原理，导致我碰到问题各种谷歌、StackOverflow，搜代码，拷贝粘贴代码。然后又荒废了一段时间，今年开始又重拾 RN，有了之前的经验，这次是比较系统的学习了。

先介绍一下我的情况：我是原生 iOS 开发，Objective C 3年经验，Swift 2年经验。

## 路线

基本路线如下：

1. 详细了解 JavaScript
2. 详细了解 React
3. 详细了解 React Native
4. 其他（比如说 Redux）

下面详细介绍。

### JavaScript

关键是了解 JavaScript 的继承和原型链机制，同时包括 JavaScript 的对象模型。

参考资料如下：（参考资料如果包含浏览器相关的内容，可以忽略）

1. [MDN JavaScript](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript)
2. [ECMAScript 6 入门](http://es6.ruanyifeng.com/) 或者 [JavaScript 指南](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide)
3. [TypeScript 英文文档](http://www.typescriptlang.org/docs/handbook/basic-types.html)（[中文版文档](https://www.tslang.cn/docs/handbook/basic-types.html)）

先从 MDN 的 JavaScript 教程开始看，这是 JavaScript 的基础，有一定编程经验在初级部分可以看的快一点，但是关于 [JavaScript 对象介绍](https://developer.mozilla.org/zh-CN/docs/Learn/JavaScript/Objects)、[继承与原型链](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Inheritance_and_the_prototype_chain)、[对象模型的细节](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Details_of_the_Object_Model) 要详细的看。

如果没有头绪，或者觉得跳着看思路跟不上，可以从头到尾细看（我就是这么看的，但是我看的比较快，花了2天时间看完）

看完 MDN 的教程以后就可以看 ES6 相关内容了。我看的是 ruanyifeng 的 ES6 网上教程，不过也可以看 MDN 的 [JavaScript 指南](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide)，两者可以对比的看。我是看完 ruanyifeng 的教程之后又看了 MDN 的 JavaScript 指南，由于看过 ruanyifeng 的 ES6，所以 MDN 的 JavaScript 指南就可以快一点。我觉得这里比较关键的是理解 ES6 的模块（Module），也就是这两章：[Module 的语法](http://es6.ruanyifeng.com/#docs/module) 和 [Module 的加载实现
](http://es6.ruanyifeng.com/#docs/module-loader)，还有就是基本的 ES6 语法。

在看的过程中有些内容可能不好懂，完全可以跳过，等后面写代码或者看代码碰到的时候再回过头来看，只要记住有这么一个东西即可。有些内容比较好懂，或者理解一下就懂了，这时候可以对比一下和其他语言在这方面有哪些不同和相同（比如你是 Java 程序员，学习 JavaScript，就可以对比一下 Java 的继承和 JavaScript 的继承），这样子可以加深你对 JavaScript 的理解。

看完这些其实 JavaScript 已经入门了，不过我要特别提一下 TypeScript。由于 JavaScript 是弱类型语言，经常代码写着写着就不知道当前函数的参数定义是什么。无论是 debug 还是阅读代码、写代码，不知道一个东西确定是什么是很痛苦的（你可以想象一下你写的一个函数，有一个参数，但是却无法确定类型是什么，所以需要时刻在脑海中告诉自己这个参数的定义是这样的，但是一不小心疏忽了，编辑器也没有提醒你这里写的有问题，那只能在运行时发生问题才知道出了问题，然后又要花一段时间去调试）。TypeScript 是微软出的一个比较流行的解决这类问题的方案，就是给 JavaScript 加上 Type （类型）。你也可以在一开始不用 TypeScript，但是等到你碰到很多上面提到的类似问题时，TypeScript 就是你的解决方案。

### React

我刚开始学习 RN 的时候，对 React 一无所知，直接根据 React Native 的官方文档来学习。但是在越来越深入的写 React Native 相关代码的时候，我发现，了解 React 是很关键的。

对于喜欢立即上手的人来说，可以先不看 React 相关的内容，直接学习 RN。但是在你写了一定量的代码以后，你会碰到一些问题，比如如何复用组件，如何处理一些不是很常见的操作，如何处理数据（展示组件和容器组件）等等，这时候你就可以开始去了解 React 了。

了解 React 当然是官方文档：

1. [React 中文文档](https://doc.react-china.org/docs/hello-world.html)
2. [React 英文官方文档](https://reactjs.org/docs/hello-world.html)

上面两个内容是一样的，喜欢中文的可以看中文，喜欢看英文的可以看英文。

建议能够全部看完所有内容，不要忽略任何一个地方（如果是 Web 开发可以跳着看）。

当然其实看完这文档是不够的，我觉得有时间一定要理解一下 React 的实现原理。（我没有看过 React 源码，所以就不继续说了）

### React Native

说了那么多终于到 RN 这一部分了。还是要强调一下，学习 RN 可以先不了解 JavaScript 和 React，（前提是你要有一点编程经验，如果是大神的话，请随意）喜欢立即上手的可以直接根据官方文档上手。不过要深入了解 RN，则必须要了解 React 和 JavaScript。

学习 RN 的时候主要就是看 [官方英文文档](https://facebook.github.io/react-native/docs/getting-started.html)（[中文版](https://reactnative.cn/docs/0.51/getting-started.html)），一些环境依赖等根据官方文档来操作即可。

RN 需要关键了解使用的部分如下：

1. FlexBox，一个前端布局的方式，写界面的时候一定要了解这个，否则界面就写不出来
2. 各种 UI 组件的使用、属性等，可以一边写代码一边学习使用
3. 网络请求，大部分 app 都会有网络请求，所以了解这个也很关键。有一个框架 [axios](https://github.com/axios/axios) 可以尝试使用，这是一个 HTTP 客户端框架。
3. 如何与原生代码交互，iOS 相关文档地址 [iOS Native Modules](https://facebook.github.io/react-native/docs/native-modules-ios.html)、[iOS Native UI Components](https://facebook.github.io/react-native/docs/native-components-ios.html) ，Android 相关文档 [Android Native Modules](https://facebook.github.io/react-native/docs/native-modules-android.html)、[Android Native UI Components](https://facebook.github.io/react-native/docs/native-components-android.html)
4. 数据存储，使用 RealmJS 或者原生的 AsyncStorage，使用 RealmJS 的问题是，如果你的 iOS 原生项目已经使用了 RealmSwift 等的话，会导致符号冲突，无法通过编译。

一些坑：

1. 建议在原生项目上添加 RN 支持，文档地址[集成到现有原生应用](https://reactnative.cn/docs/0.51/integration-with-existing-apps.html#content)。如果使用 `create-react-native-app AwesomeProject` 命令来生成一个 RN 项目，会引入 Expo 这种东西，我个人不建议使用此类框架。
2. 不建议使用 `NavigatorIOS` 这类组件，推荐 [React Navigation](https://reactnavigation.org/)，但是这个开源组件坑也比较多，不过个人认为还是比 RN 提供的组件要好。


### 其他

Redux，Redux 是一个关于数据流的前端框架，也可以用在 RN 中。其他还有类似的比如 MobX（除此之外据说新版 React 有种 Context API 可以让我们告别使用 Redux，不太清楚，应该是比较新的东西）。

先说一下，一开始可以不使用 Redux，Redux 的官网也提示说如果不知道要不要使用 Redux，则不要使用 Redux。

学习自然要根据文档来：[Redux 英文官方文档](https://redux.js.org/)（[中文版](https://cn.redux.js.org/)）

理解 Redux 比较关键的是它的状态树，所有的操作最终会反应到状态树上，所有的操作也是围绕状态树展开来的。所以一开始做 app 的时候设计一个比较贴切的状态树很关键。

### 进阶

1. 打包
2. 热更新
3. 与原生交互
4. RN 原生实现原理

参考资料有如下：

1. [React Native拆包及热更新方案](http://solart.cc/2017/02/22/react-native-jsbundle-patch/)
1. [携程是如何做React Native优化的](https://zhuanlan.zhihu.com/p/23715716)
1. [微软的热更新方案 CodePush](https://microsoft.github.io/code-push/)
1. [ReactNative中文网推出的代码热更新服务](https://github.com/reactnativecn/react-native-pushy)
1. [iOS Native Modules](https://facebook.github.io/react-native/docs/native-modules-ios.html)
1. [iOS Native UI Components](https://facebook.github.io/react-native/docs/native-components-ios.html) 
1. [Android Native Modules](https://facebook.github.io/react-native/docs/native-modules-android.html)
1. [Android Native UI Components](https://facebook.github.io/react-native/docs/native-components-android.html)
1. [React Native通信机制详解](https://blog.cnbang.net/tech/2698/)


## 开发环境

我使用 Visual Studio Code，不建议使用 Atom（很卡）。并且配置了 TypeScript 环境，配置参考 demo 地址： https://github.com/Microsoft/TypeScript-React-Native-Starter

## 注意事项

个人觉得 RN 的要求其实很高，不仅要了解前端开发相关内容，还要了解 Android 和 iOS 的原生内容。一个纯 JavaScript 的 RN 项目不太现实。所以建议在原生项目上使用 RN，不要使用纯 RN 项目。这对于理解 RN 也有一定帮助。

不建议使用其他类似 RN 的解决方案，比如说 Weex，当然我并没有了解过 Weex。我想说的是 RN 的生态环境是很活跃的，GitHub 有很多开发者共享的开源代码，很多人也在 GitHub 和 stackoverflow 上讨论自己碰到的 RN 相关的问题。但是即使如此，RN 还是没有出 1.0 版本，而且在实际使用过程中经常要自己造轮子，也会碰到很多奇奇怪怪的问题，导致拖累开发速度，从学习角度不是问题，但是工作就不一样了。所以使用 Weex 情况可能更糟糕。


## 总结

学习 React Native 的过程其实更像是入门 React 前端开发，区别就是代码环境从浏览器变成了移动设备。

在原生移动开发势微的今天，从 React Native 切入来了解前端开发也很不错。在学习的同时，结合自己已有的原生开发经验，对比 React Native 开发，更能在巩固 React Native 相关知识的同时对加深原生开发的理解。