---
layout: single
author_profile: true
title: 后台定期获取设备地理位置信息
---


背景：未越狱情况下，用户手动点击 `Home`键退出、双击 `Home`键强制退出，以及被系统终止、挂起的情况下仍需要上报地理位置。 

主要有一下几种方法，每种方法都有各自的特点和缺点。注意，如果用户在设置中关闭了定位服务，没有任何方法可以在后台获取地理位置信息；同时如果用户关闭了后台应用刷新，只有第二种方法可以短暂的运行。而且下面的有些方法使用的 `API` 要么是专门为定位服务设计的，要么官方用途是另外一些场景，只是被我们委婉的用于获取定位数据。

约束条件：`APP` 无法在任意时间，任何情况下去获取定位信息。后台获取定位信息的时间、频率都是不确定的。而且下面所有提到的时间其实都是最理想的情况下。

`iOS` 支持的所有能够在后台运行的场景以及苹果提供的对应方法在这个官方文档：[Background Execution](https://developer.apple.com/library/content/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/BackgroundExecution/BackgroundExecution.html)

获取定位信息的所有方式在这个官方文档：[Getting the User’s Location](https://developer.apple.com/library/content/documentation/UserExperience/Conceptual/LocationAwarenessPG/CoreLocation/CoreLocation.html)

关于如何节省定位时候的耗电在这个官方文档：[Energy Efficiency Guide for iOS Apps -LocationBestPractices](https://developer.apple.com/library/content/documentation/Performance/Conceptual/EnergyGuide-iOS/LocationBestPractices.html)


## `SCL`（significant change location） 

当定位变化比较大的时候，每15分钟唤醒，每次唤醒执行最多10秒钟

缺点：

1. 地理位置需要有所变化，否则系统永远不会唤醒应用
2. 状态栏会一直有定位指示器，直到关闭 `SCL` 服务，关闭之后没有办法在后台再次开启`SCL`

适用环境：用户手动点击 `Home`键退出、双击 `Home`键强制退出，以及被系统终止、挂起

不适用环境：地理位置没有变化

参考：

1. [Starting the Significant-Change Location Service](https://developer.apple.com/library/content/documentation/UserExperience/Conceptual/LocationAwarenessPG/CoreLocation/CoreLocation.html#//apple_ref/doc/uid/TP40009497-CH2-SW8)

2. [Getting Location Updates for iOS 7 and 8 when the App is Killed/Terminated/Suspended](http://mobileoop.com/getting-location-updates-for-ios-7-and-8-when-the-app-is-killedterminatedsuspended)

3. [Behaviour for significant change location API when terminated/suspended](http://stackoverflow.com/questions/3421242/behaviour-for-significant-change-location-api-when-terminated-suspended)


## `Background App Refresh` 后台应用刷新

可以设置最长每3分钟唤醒一次(超过3分钟下次就唤醒不了了，这个3分钟其实是每次唤醒能够执行的最长时间，在快结束的时候再次执行一个后台刷新任务，使得 `app` 能够在后台一直运行，也就是说 `app` 从退到后台开始就一直在运行，但是长时间在后台运行的 `app` 肯定会被系统检测到从而可能被系统强制终止。同时这个3分钟是实验得到的，不同设备不同系统版本可能会有区别)

缺点：频繁刷新，由于是一直运行，耗电量应该是极大的（暂未测试）

适用环境：用户点击 `Home`键退出

不适用环境：双击 `Home`键强制退出，以及被系统终止、挂起

参考：

1. [Tracking the User’s Location](https://developer.apple.com/library/content/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/BackgroundExecution/BackgroundExecution.html#//apple_ref/doc/uid/TP40007072-CH4-SW25)
2. [Background Location Update Programming for iOS 7 and iOS 8](http://mobileoop.com/background-location-update-programming-for-ios-7)


## `Background Fetch Mode` 定期刷新数据

可以配置 `app` 定期去网络下载数据。系统每20~30分钟调用一次。但是如果下载的数据花费的时间太长或者说并没有去下载任何数据。系统在下一次调用可能是1小时或者更久，甚至可能不会再次调用。

缺点：需要去做无谓的下载，既然是下载，肯定需要服务器支持

适用环境：用户手动点击 `Home`键退出、双击 `Home`键强制退出，以及被系统终止、挂起

不适用环境：暂时没发现（不确定在这里去频繁获取定位数据是否会被系统检测到从而不再被调用，毕竟这个 `API` 的使用场景是比如用户可能会频繁查看比赛实时数据）

参考：

1. [iOS periodic background location updates which depends not only on significant location change](http://stackoverflow.com/questions/32684584/ios-periodic-background-location-updates-which-depends-not-only-on-significant-l)
2. [Fetching Small Amounts of Content Opportunistically](https://developer.apple.com/library/content/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/BackgroundExecution/BackgroundExecution.html#//apple_ref/doc/uid/TP40007072-CH4-SW56)