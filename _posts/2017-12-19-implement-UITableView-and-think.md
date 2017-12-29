---
layout: single
author_profile: true
title: 实现 UITableView 以及思考
---

## 前言

一年前因为 `UITableView` 无法满足需求，我实现了类似 `UITableView` 的组件， `DLTableView`。

之所以实现一个自定义的 `UITableView`，是因为我需要一个能无限循环滚动的 TableView。

通常的做法是设置 dataSource 的 `numberOfRowsInSection:` 方法返回尽量多的行数，然后在对 row 取余以实现看起来是无限循环滚动的特效。`UIDatePicker` 就是这样子实现的。

这种方案有个问题，如果用户一直往下滚动，还是能够滚动到底的，所以行数要设置的尽量大。另一方面太大的行数会导致内存使用暴涨。 `UIDatePicker` 的做法是设置一个合理的行数，在用户停止滚动的时候去校正 `contentOffset`，由于没有动画效果，用户无法感知，在内存和效果之间取得了平衡。但是连续往下滚的话，你会发现还是能够滚到底部，等你重新进来的时候才会重置一下 `contentOffset`。所以这个方案并不完美。

因为诸多原因最终我决定实现一个 `UITableView`。好处有几个：

1. 从实践中思考苹果是如何实现的，有什么难点，它的 `UITabeleView` 有什么可以改进之处。这些不是单纯的看几篇博客或者直接看源码就可以获得的
2. 实现一个基本的 TableView 之后，可以添加诸多原生 TableView 不具备的功能，比如说循环滚动特性。项目实践可以更加灵活
3. 可以基于新的自定义的 TableView 实现一个比官方 `UIPickerView`/`UIDatePicker` 更好的 `DLPickerView`

这里先预告一下，下一篇我会讲解如何基于 `DLTableView` 实现 `DLPickerView`。这个 pickerView 有一下特性：

1. 类似 `UITableView` 的 delegate 和 dataSource，同时可以自定义 cell，用法完全类似 `UITableView`，灵活性更高。
2. 可以设置连续的可选区域，用户不能滚动到可选区域之外
3. 可以配置循环滚动或者不循环滚动
4. `DLPickerViewDelegate` 提供对每个 cell 基于当前位置的视图自定义

[`DLTableView`](https://github.com/danleechina/DLTableView) 的 GitHub 地址是 https://github.com/danleechina/DLTableView。

下面开始讲如何实现一个自定义的 TableView。

## 关键点

自定义 TableView 的实现有两个关键点：

1. Cell 复用
2. 找到实现的切入点

Cell 复用比较好理解，因为内存是有限的，不可能无限多的生成 Cell 实例。

Cell 复用的实现也比较简单，给每种 Cell 类设置一个 identifier，以此为键，值为一个包含该 Cell 类的 set 集合。Cell 滚出可视域时候加到 set 里面，Cell 出现在可视域时从 set 里面获取一个，如果 set 里面没有的话则生成一个。

那么切入点呢？

首先，我们的自定义 `DLTableView` 是基于 `UIScrollView` 的，`UIScrollView` 提供了很好的滚动效果，虽然在使用的时候发现还是有点不太满意的地方，比如以动画的方式设置 `contentOffset` 时没法做到设置一个完成动画的回调，还有就是比如说对滚动的阻力做调整。

现在请思考一个问题，在什么时间给 `DLTableView` 添加一个 Cell，也就是说将 Cell 作为 TableView 的子视图或者子孙视图？以及当 Cell 从用户视线中消失时将 Cell 从 `DLTableView` 中移除，并加入到复用的队列中？

首先想到的是 `scrollViewDidScroll:`，在 `UIScrollView` 滚动的时候添加 Cell。但是 `scrollViewDidScroll` 是代理实现的，作为继承自 `UIScrollView` 的 `DLTableView` 本身不应该实现这个代理。而且在 `contentSize` 未知的时候 `UIScrollView` 可能根本不能滚动。

当然我们可以 hook 掉 `UIScrollView` 的 `setDelegate:` 方法，然后在自定义 TableView 里面实现所有 `UIScrollViewDelegate` 的方法，在这些方法里面再回调给使用自定义 TableView 的 delegate。天猫的 [LazyScrollView](https://github.com/alibaba/LazyScrollView) 就是这样实现的 （不过它只复写了 `scrollViewDidScroll:` 方法，其他方法是通过动态转发来实现的）。这种方案的缺点是要手动设置 `contentSize`。

我这边的实现是使用 `layoutSubviews`。每次初始化的时候，UIView 都会调用该方法，在这个方法里面我们可以初始化最初的可见 Cell，以及 `contentSize`。灵感来自与苹果的官方 demo：[StreetScroller](https://developer.apple.com/library/content/samplecode/StreetScroller/Introduction/Intro.html)

另外，当 `UIScrollView` 滚动的时候，会频繁的调用该方法。这样我们就可以动态的将 Cell 加入或去掉。

>这里补充一点，有时候 `scrollViewDidScroll:` 的调用频率没我们想的那么多的时候，你也可以实现  `layoutSubviews`，`layoutSubviews` 的频率比 `scrollViewDidScroll:` 高，当然不要忘记调用 `[super layoutSubviews]`


## 实现细节

1. TableView 可能会有 header 或者 footer，所以在计算高度的时候每个 cell 的位置要加上 header 高度的偏移，计算 `contentSize` 的高度的时候要加上 footer 的高度。
2. TableView 可以不只有 row 还可以进一步区分每个 section，关键一点是每个 section 也可以有 header 和 footer，所以计算 cell 的位置的时候会有那么一点不直观，同时 section 的 header 和 footer 在 TableView 中的位置是滚动的，到顶的时候又是漂浮在 row 上面，这一点要注意。
3. UITableView 支持将某个 Cell 滚动到顶部，自定义的时候可以考虑将 Cell 滚到顶、中、底。
4. 点击。一般不会在每个 Cell 里面加一个 tap gesture recognizer。我的做法是在 TableView 层面加一个 Tap gesture recognizer，这样每次识别到点击的时候，可以具体分配到某个 cell。当然还有设置点击效果（比如 cell 点击变灰）。

## DLTableView

我这边实现了一个自定义 TabelView，`DLTableView`。

目前 `DLTableView` 实现的特性有：

1. 类似 `UITableViewDataSource` 的 `DLTableViewDataSource`，使用者可以实现 dataSource 来自定义行数和每一行的 cell
2. 类似 `UITableViewDelegate` 的 `DLTableViewDelegate`，目前包含点击逻辑、自定义高度和某个 Cell 滚出可视区域时候的回调
3. 可以循环滚动，同时内存使用不会出现暴涨（比如 `UITableView` 就会暴涨）
4. 点击某一行可以滚动到顶、中、底

你可以将 `DLTableView` 看做是一个简化版的 `UITableView`，它没有 section，只有 row，也没有实现每个行的分割线，更没有实现 `UITableView` 里面的左滑 cell 自定义菜单这些功能，而且目前还不支持自动计算行高。

不过由于 `DLTableView` 的实现是开源可见的，所以你可以基于此做进一步的自定义。

## 更好的 TableView

上面说了那么多，都是在讲如何实现一个类似 `UITableView` 的 TableView，主要是山寨。

那么如何基于山寨版的 TableView，提供一个更好的 TableView ？

首先什么叫做更好的 TableView ？我整理了如下：

1. 自动计算以及缓存 Cell 高度，市面上有很多相关开源代码，你可以查看。
2. 异步构建视图，以及加载数据，参考 AsyncDisplayKit(Texture) 的思想。

## 引用

1. [StreetScroller](https://developer.apple.com/library/content/samplecode/StreetScroller/Introduction/Intro.html)
2. [iOS 异构滚动视图 LazyScrollView 一些实现细节的深入解读](http://pingguohe.net/2017/12/21/iOS-LazyScroll-implemention.html)
3. [LazyScrollView](https://github.com/alibaba/LazyScrollView)