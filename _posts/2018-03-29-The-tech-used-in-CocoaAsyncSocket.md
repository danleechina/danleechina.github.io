---
layout: single
author_profile: true
title: CocoaAsyncSocket 实现时用到的技术
---

## 前言

最近在阅读 [CocoaAsyncSocket](https://github.com/robbiehanson/CocoaAsyncSocket) 的源码，整理里一下其中用到的一些技术点。

## GCD 相关

### 目标队列（Target Queue）

#### 概念 

目标队列的基本概念是：你创建的所有队列，如果没有指定其目标队列，那么它的目标队列是优先级为 `DISPATCH_QUEUE_PRIORITY_DEFAULT` 的全局并发队列。每次在你队列中的一个 block 开始执行的时候，GCD 会重新将其放入目标队列执行。详细介绍可以查看引用1。


#### 目标队列可能引起死锁

这里说一下使用目标队列可能导致的死锁问题：

比如说有两个队列：
```
dispatch_queue_t queueOne;
dispatch_queue_t targetQueueOne;
queueOne.targetQueue = targetQueueOne;
```
那么所有在 queue 上的操作最终会在 targetQueue 上执行。一切看起来都很好，现在有设想下面这样的函数：

```
 - (BOOL)doSomething
  {
      __block BOOL result = NO;
      dispatch_block_t block = ^{
          result = [self someInternalMethodToBeRunOnlyOnQueueOne];
      }
      if (is_executing_on_queue(queueOne))
          block();
      else
          dispatch_sync(queueOne, block);
      
      return result;
  }
```
如果你在 targetQueueOne 上调用这个方法会怎样？答案是死锁。这是因为 GCD 的 API 没有提供一个机制去发现队列的目标队列，所以我们不知道 queueOne 的目标队列。（死锁的原因是在当前串型队列上又同步派发了一个操作，导致两者相互等待）

#### 目标队列死锁问题解决

一句话解释：使用 `dispatch_queue_set_specific()` 和 `dispatch_get_specific()`。

先给 queueOne 设置 sepcific

```
IsOnQueueOneOrTargetQueueKey = &IsOnQueueOneOrTargetQueueKey;

void *nonNullUnusedPointer = (__bridge void *)self;
dispatch_queue_set_specific(queue, IsOnQueueOneOrTargetQueueKey, nonNullUnusedPointer, NULL);
```
每次要判断是否是在当前队列或者目标队列的时候如下判断：
```
if (dispatch_get_specific(IsOnQueueOneOrTargetQueueKey)) {
	block();
} else {
    dispatch_sync(queueOne, block);
}

```


### GCD Dispatch 源

#### 概念 

当要处理系统底层相关任务的时候，我们必须准备好要等待一段时间。当你的应用程序陷入系统内核或者其他系统层面的时候，比起在进程内部的函数调用，我们要面临切换上下文导致的巨大时间开销。结果就是，许多系统库会提供异步接口使你的代码提交一个请求给系统，然后继续自己的工作，当系统完成你的请求，在回调你的请求结果处理 block。GCD 的 dispatch source 就是这样一个基本数据类型。具体介绍可以查看官方文档，引用2。

#### 使用 Dispatch 源来做一个定时器

在 iOS 中想要实现一个定时器可以使用系统原生的类 `NSTimer`，`NSTimer` 的问题是它的设计模式是 target-action 模式。也就是要在当前类中定义一个方法给定时器使用。比起多定义一个方法我们更喜欢用 block。但是 `NSTimer` 的 API 在 iOS 11 才支持回调的方式。有很多方式来自定义 `NSTimer` 使用回调的方式。这里不做介绍，可以查看引用3。

实现一个回调的定时器的一种方式是使用 Dispatch 源，代码如下所示：

```

dispatch_source_t CreateDispatchTimer(uint64_t interval,
              uint64_t leeway,
              dispatch_queue_t queue,
              dispatch_block_t block)
{
   dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER,
                                                     0, 0, queue);
   if (timer)
   {
      dispatch_source_set_timer(timer, dispatch_walltime(NULL, 0), interval, leeway);
      dispatch_source_set_event_handler(timer, block);
      dispatch_resume(timer);
   }
   return timer;
}
 
void MyCreateTimer()
{
   dispatch_source_t aTimer = CreateDispatchTimer(30ull * NSEC_PER_SEC,
                               1ull * NSEC_PER_SEC,
                               dispatch_get_main_queue(),
                               ^{ MyPeriodicTask(); });
 
   // Store it somewhere for later use.
    if (aTimer)
    {
        MyStoreTimer(aTimer);
    }
}
```


#### 使用 Dispatch 源来处理 Socket 相关操作

无论是服务器端还是客户端socket 都需要某种机制使得其能够和对方进行持续的通信。通常的解决办法是进行一个永久循环，遇到某种条件时再终止循环。这种方式缺陷较多，一是不够“优雅”，二是无限循环非常浪费 CPU 时间。使用 Dispatch 源来做处理会更好。


 CocoaAsyncSocket 这个框架的核心思想之一就是使用源来处理 Socket 的读和写。下面代码来自 CocoaAsyncSocket：

```
accept4Source = dispatch_source_create(DISPATCH_SOURCE_TYPE_READ, socket4FD, 0, socketQueue);
int socketFD = socket4FD;
dispatch_source_t acceptSource = accept4Source;
__weak GCDAsyncSocket *weakSelf = self;
dispatch_source_set_event_handler(accept4Source, ^{ @autoreleasepool {
	__strong GCDAsyncSocket *strongSelf = weakSelf;
	if (strongSelf == nil) return_from_block;
	unsigned long i = 0;
	unsigned long numPendingConnections = dispatch_source_get_data(acceptSource);
	while ([strongSelf doAccept:socketFD] && (++i < numPendingConnections));
}});
dispatch_source_set_cancel_handler(accept4Source, ^{
	close(socketFD);
});

dispatch_resume(accept4Source);
```

使用 `dispatch_source_create` 来创建源，参数 `DISPATCH_SOURCE_TYPE_READ` 指的是这是一个读类型的源，读事件来自与 socket4FD，事件回调发生在 socketQueue 上。

`dispatch_source_set_event_handler` 指定了每次事件发生时的回调，
`dispatch_source_set_cancel_handler` 指定了取消一个源的操作，取消一个源可以使用 `dispatch_source_cancel` 完成。默认情况下源是不会开始执行的，所以要用 `dispatch_resume` 显示的启动一个源。

dispatch 源还有很多作用，比如监视一个文件夹中的文件变化等。详细介绍可以看官方文档或者引用4


### SSL/TLS 握手

我们知道每一个 TCP 连接都会进行3次握手，然后才会开始通信，具体原理可以查看引用5。HTTP 连接是建立在 TCP 连接的基础之上的。HTTPS 则更进一步，其连接是建立在 SSL/TLS 连接之上的。而 SSL/TLS 连接又是建立在 TCP 3次握手之后。这里不描述 SSL/TLS 的握手过程，详细可以查看参考资料6。

如何在 iOS 或者 macOS 系统上实现 SSL/TLS 握手呢？先说一下为什么要自己实现 SSL/TLS 握手，毕竟我们使用 HTTPS 的时候好像并没有处理这个？

了解客户端证书绑定的会知道，要做客户端证书绑定我们需要实现 NSURLSessionDelegate 的 `URLSession:didReceiveChallenge:completionHandler` 协议。其实在这一步发生的时候，NSURLSession 就已经正在为我们进行 SSL/TLS 握手，只是由于我们实现了这个协议，所以 NSURLSession 需要我们告诉它是否要信任这次握手，默认情况下 NSURLSession 有自己的逻辑来决定是否信任。NSURLSession 为我们隐藏了很多其他握手细节。

当我们想自定义 SSL/TLS 的握手细节的时候就需要自定义了。（SSL/TLS 握手中每一个步骤请看资料6）。

iOS 这边实现 SSL/TLS 握手 有两种方式

1. 使用苹果提供的 SecureTransport 框架
2. 使用 CFStream 相关接口

使用 SecureTransport 的优势是可以结合 dispatch 源，并且性能高，可控性强，但是 SecureTransport 没有开源。

使用 CFStream 的优势是使用 dispatch source 苹果没有一种机制告诉我们当前是否要进入后台或者正在后台。CFStream 的 API 就可以，通过设置 steam 的 kCFStreamNetworkServiceType 为 kCFStreamNetworkServiceTypeVoIP 即可。

注意 CocoaAsyncSocket 中貌似无法用 CFSteam 来自定义证书认证。

具体如何使用 CFStream 请看资料 [CFSocketStream](https://developer.apple.com/library/content/technotes/tn2232/_index.html#//apple_ref/doc/uid/DTS40012884-CH1-SECCFSOCKETSTREAM)

具体如何使用 Secure Transport 请看资料 [Secure Transport](https://developer.apple.com/library/content/technotes/tn2232/_index.html#//apple_ref/doc/uid/DTS40012884-CH1-SECSECURETRANSPORT)


### SOCKET

CocoaAsyncSocket 本来就是对原生的 Socket 的封装，所以要看懂源码需要了解原生的 Socket 的写法。主要分成两个部分：

1. 服务器端 Socket
2. 客户端 Socket

同时 Socket 本身还有很多细节内容需要了解。建议在阅读或者使用 CocoaAsyncSocket 之前了解这方面的知识。

英文资料可以看： [Introduction to Sockets Programming in C using TCP/IP](http://www.csd.uoc.gr/~hy556/material/tutorials/cs556-3rd-tutorial.pdf)

中文资料可以看： [Socket](https://github.com/xuelangZF/CS_Offer/blob/master/Network/Socket.md)

## 引用

1. [【翻译】GCD Target Queues](https://www.jianshu.com/p/bd2609cac26b)
2. [Dispatch Sources 官方文档](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/GCDWorkQueues/GCDWorkQueues.html)
3. [NSTimer-Blocks](https://github.com/jivadevoe/NSTimer-Blocks)
4. [细说GCD（Grand Central Dispatch）如何用](https://github.com/ming1016/study/wiki/%E7%BB%86%E8%AF%B4GCD%EF%BC%88Grand-Central-Dispatch%EF%BC%89%E5%A6%82%E4%BD%95%E7%94%A8)
5. [通俗大白话来理解TCP协议的三次握手和四次分手](https://github.com/jawil/blog/issues/14)
6. [SSL/TLS协议运行机制的概述](http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html)