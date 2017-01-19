---
layout: single
author_profile: true
title: iOS app 逆向分析
---


- [砸壳](#crack)
- [class-dump](#class-dump)
- [搭建越狱开发环境 Theos](#theos_pc)
- [配置越狱开发调试的 iOS 设备 Theos 环境](#theos_ios)
- [动态分析－Logify](#logify)
- [Theos 创建 tweak](#tweak)
- [动态分析－lldb](#lldb)
- [参考资料](#references)

## <a name="crack"></a>砸壳

为了砸壳，我们需要使用到 dumpdecrypted，这个工具已经开源并且托管在了 GitHub 上面，我们需要进行手动编译。步骤如下：

1. 从 GitHub 上 clone 源码：

	```
	$ cd ~/iOSReverse	
	$ git clone git://github.com/stefanesser/dumpdecrypted/ 
	```
	
2. 编译 dumpdecrypted.dylib：

	```
	$ cd dumpdecrypted/
	$ make
	```
执行完 `make` 命令之后，在当前目录下就会生成一个 `dumpdecrypted.dylib`，这个就是我们等下要使用到的砸壳工具。（注意可能需要更改 `makefile` 的配置，以适应不同架构的处理器和不同版本的 `iPhone sdk`）。

有了砸壳工具之后，下一步就需要找到待砸的猎物了，这里就以微信为例子，来说明如何砸掉微信的壳，并导出它的头文件。

1. 使用 ssh 连上你的 iOS 系统（需要在越狱机器上安装 `OpenSSH`）

	```
	# 注意将 IP 换成你自己 iOS 系统的 IP
	$ ssh root@192.168.1.107
	```

2. 连上之后，使用 `ps` 配合 `grep` 命令来找到微信的可执行文件

	```
	  $ ps -e | grep WeChat
	  4557 ??    5:35.98 /var/mobile/Containers/Bundle/Application/944128A6-C840-434C-AAE6-AE9A5128BE5B/WeChat.app/WeChat
	  4838 ttys000    0:00.01 grep WeChat
	```

	在这里，因为我们已经知道微信的应用名就叫 WeChat，所以可以直接进行关键字搜索。那如果我们不知道想要找到应用名称叫什么该怎么办呢？可以在 App Store 下载该 app 的 IPA 包，解压查看 Payload 下扩展名为 `.app` 的文件名，这就是我们需要查找的进程

	> 注意：如果输入 ps 之后，系统提示没有这个命令，那就需要到 cydia 当中安装 adv-cmds 包。

3. 使用 Cycript 找到目标应用的 Documents 目录路径。

	> Cycript 是一款脚本语言，可以看成是 Objective-JavaScript，它最好用的地方在于，可以直接附加到进程上，然后直接测试函数的效果。关于 cycript 的更详细介绍可以看它的官网.
	
	安装 cycript 也很简单，直接在 cydia 上搜索 cycript 并安装就可以了。装完之后就可以来查找目标应用的 Documents 目录了：
	
	```
	$ cycript -p WeChat
	cy# NSHomeDirectory()
	@"/var/mobile/Containers/Data/Application/13727BC5-F7AA-4ABE-8527-CEDDA5A1DADD"
	```
	可以看到，我们打印出了应用的 Home 目录，而 Documents 目录就是在 Home 的下一层，所以在我这里 Documents 的全路径就为 `/var/mobile/Containers/Data/Application/13727BC5-F7AA-4ABE-8527-CEDDA5A1DADD/Documents`
	
	> 注意：可以使用 Ctrl + D 来退出 cycript

4. 将 dumpdecrypted.dylib 拷到 Documents 目录下：

	```
	# 注意将 IP 和路径换成你自己 iOS 系统上的
	# 注意本步是在电脑端进行的操作
	$ scp ~/iOSReverse/dumpdecrypted/dumpdecrypted.dylib root@192.168.1.107:/var/mobile/Containers/Data/Application/13727BC5-F7AA-4ABE-8527-CEDDA5A1DADD/Documents
	
	```

5. 砸壳

	```
	$ cd /var/mobile/Containers/Data/Application/13727BC5-F7AA-4ABE-8527-CEDDA5A1DADD/Documents
	# 后面的路径即为一开始使用 ps 命令找到的目标应用可执行文件的路径 
	$ DYLD_INSERT_LIBRARIES=dumpdecrypted.dylib /var/mobile/Containers/Bundle/Application/944128A6-C840-434C-AAE6-AE9A5128BE5B/WeChat.app/WeChat
	```

	完成后会在当前目录生成一个 WeChat.decrypted 文件，这就是砸壳后的文件。之后就是将它拷贝到 OS X 用 class-dump 来导出头文件啦。（可以使用 scp 命令拷贝到 OS X 下）
	
	```
	$ mkdir ~/iOSReverse/WeChat
	# 注意替换 IP和路径
	$ scp root@192.168.1.107:/var/mobile/Containers/Data/Application/13727BC5-F7AA-4ABE-8527-CEDDA5A1DADD/Documents/WeChat.decrypted ~/iOSReverse/WeChat/
	```

## <a name="class-dump"></a>class-dump

class-dump 是一个工具，它利用了 Objective-C 语言的运行时特性，将存储在 Mach-O 文件中的头文件信息提取出来，并生成对应的 .h 文件。可以在其 [class-dump 官网](http://stevenygard.com/projects/class-dump/)下载，在个人目录下新建一个 bin 目录，并将其添加到 PATH 路径中，然后将下载后的 class-dump-3.5.dmg 里面的 class-dump 可执行文件复制到该 bin 目录下，赋予可执行权限：

```
$ mkdir ~/bin
$ vim ~/.bash_profile
# 编辑 ~/.bash_profile 文件，并添加如下一行
export PATH=$HOME/bin/:$PATH
# 将 class-dump 拷贝到 bin 目录后执行下面命令
$ chmod +x ~/bin/class-dump
```

### 使用 class-dump

使用 class-dump 来将微信的头文件导出：

```
$ mkdir ~/iOSReverse/WeChat/Headers
$ cd ~/iOSReverse/WeChat/
$ class-dump -S -s -H WeChat.decrypted -o Headers/
```
> 注意使用 class-dump 仍然只导出 CDStructures.h 一个文件（或者没有文件），则可能架构选择错误，或者使用的`dumpcrypted` 编译 `sdk` 不对
> 
> 考虑下面类似的命令来指定砸壳文件的架构
> `class-dump --arch armv7 dajietiao.decrypted -H -o ./heads/`


## <a name="theos_pc"></a>搭建越狱开发环境 Theos

1. 安装 Xcode 和 Command Line Tools
2. 安装 dpkg

	`$ brew install dpkg`

2. 从 Github 下载 Theos

	打开命令行，进行如下操作：
	
	```
	$ export THEOS=/opt/theos
	# 如果之前已经安装过 theos，请先删除，然后下载最新版
	$ rm -rf $THEOS
	$ sudo git clone --recursive https://github.com/theos/theos.git $THEOS
	```

3. 配置ldid

	```
	$ git clone git://git.saurik.com/ldid.git
	$ cd ldid
	$ git submodule update --init
	$ ./make.sh
	$ cp -f ./ldid $THEOS/bin/ldid
	```
	或者直接从这两个地址下载

	1. http://joedj.net/ldid
	2. http://jontelang.com/guide/chapter2/ldid.html

4. 配置 Theos 环境

	编辑 `~/.bash_profile` 文件，在末尾添加下面这几句，注意替换 example.local 为与本地计算机同处一个 Wi-Fi 环境的 iOS 设备的 IP 地址。编辑完以后重启命令行终端。
	
	```
	export THEOS=/opt/theos
	export PATH=$THEOS/bin:$PATH
	export THEOS_DEVICE_IP=example.local THEOS_DEVICE_PORT=22
	```
	
5. 添加 substrate 库到 Theos 中

	```
	# 可以通过 brew install wget 安装 wget
	$ wget http://apt.saurik.com/debs/mobilesubstrate_0.9.6011_iphoneos-arm.deb
	$ mkdir substrate
	$ dpkg-deb -x mobilesubstrate_*_iphoneos-arm.deb substrate
	$ mv substrate/Library/Frameworks/CydiaSubstrate.framework/CydiaSubstrate $THEOS/lib/libsubstrate.dylib
	$ mv substrate/Library/Frameworks/CydiaSubstrate.framework/Headers/CydiaSubstrate.h $THEOS/include/substrate.h
	```

## <a name="theos_ios"></a>配置越狱开发调试的 iOS 设备 Theos 环境

安装 Theos 及其依赖。（详情查看参考资料 3）

1. 添加下列源到 cydia source 中。

	1. http://coolstar.org/publicrepo
	2. http://nix.howett.net/theos

2. 安装 `Perl`, `Theos`, `iOS ToolChain` 

## <a name="logify"></a>动态分析－Logify

配置越狱开发环境 OK 之后，经常在分析 app 的时候会用到 logify。
神器 Logify ，它是 theos 的一个模块，作用就是根据头文件自动生成 tweak，生成的 tweak 会在头文件的所有方法中注入 NSLog 来打印方法的入参和出参，非常适合追踪方法的调用和数据传递.

根据此前砸壳后 class_dump 出来的头文件，找到我们感兴趣的头文件比如说 BaseMsgContentViewController 在终端执行如下命令：

	$ logify.pl /path/to/BaseMsgContentViewController.h > /out/to/Tweak.xm
	
输出的tweak文件中带百分号的关键字，例如 %hook、%log、%orig 都是 mobilesubstrate 的 MobileHooker 模块提供的宏，其实也就是把 method swizzling 相关的方法封装成了各种宏标记，使用起来更简单，大家想要更深入了解各种标记，可以google一下logos语言

## <a name="tweak"></a>Theos 创建 tweak

上面我们用 logify 生成了一个 tweak 代码，我们要把它安装到手机上，首先需要使用 theos 进行编译，安装了 theos 之后，在终端输入 nic.pl，会让我们配置一些信息。

比如首先是选择项目模版当然是 tweak 啦，然后是项目名称、作者，后面两个选项要注意：

1. 首先是 bundle filter，这个需要填你需要注入的目标 app 的 bundle id，MobileLoader 模块会根据它来寻找你的 tweak 的注入目标
2. 最后是 list id applications to terminate upon installation，这里指定当 tweak 安装成功之后，需要 kill 的进程，比如我们要 hook 微信，这里就填微信的二进制文件名就可以了，为什么要 kill ？ 因为我们的插件是需要在 app 启动时加载进去的，如果不重启app，插件是不会生效的。

把上面  logify 生成的 tweak 文件覆盖到当前目录，并用文本编辑器打开 makefile 文件
最后在终端进入项目目录，输入 `make package install` 命令
期间会让你输入设备的 ssh 密码，越狱机器的默认 ssh 密码是 alpine。

然后打开 Xcode 的 Windows->Devices 选中要调试的 iOS 设备，将隐藏的控制台打开。这样我们就可以看到 app 运行时输出的日志了。（不知道如何操作的查看 参考资料 7）
由于我们使用 logify 生成的 tweak 可以打印每个函数的入参和出参，所以运行到相应的函数的时候，可以在控制台看到相应的日志。

## <a name="lldb"></a>动态分析－lldb

通过将我们砸壳的文件比如 WeChat.decrypted 抛给 hopper 解析，我们可以方便的反编译代码。查看反编译代码的时候我们会想对对应的代码进行调试。当需要在 app 中设置断点进行调试的时候，就需要远程连接到 iOS 设备，使用 lldb 进行调试。具体配置参看 参考资料 4

在 iOS 设备上使用下面这句命令开启 WeChat 的调试
	`$ debugserver *:19999 -a WeChat`
	
在计算机终端上运行下面的命令，连接到远程 iOS 设备调试

```
$ lldb 
$ process connect connect://deviceIP:19999
```

调试的时候一些有用的命令

```
# 查看 app 运行的内存基地址
$ image list -o -f
# 基地址加上函数的偏移地址（可以通过 hopper 查看，注意 hopper 刚开始让我们配置的架构选项）就是相应代码段的内存实际地址
$ br s -a '0X00000000000E800+0x00000001017d7c6c'
```

## <a name="references"></a>参考资料
1. [iOS 逆向手把手教程之一：砸壳与class-dump](http://www.swiftyper.com/2016/05/02/iOS-reverse-step-by-step-part-1-class-dump/)
2. [配置越狱开发环境](http://iphonedevwiki.net/index.php/Theos/Setup#For_Mac_OS_X)
3. [配置越狱开发调试的 iOS 设备](http://iphonedevwiki.net/index.php/Theos/Setup/iOS)
4. [使用 lldb debug 越狱设备上的 app](http://codedigging.com/blog/2016-04-27-debugging-ios-binaries-with-lldb/)
5. [移动App入侵与逆向破解技术－iOS篇](http://dev.qq.com/topic/577e0acc896e9ebb6865f321)
6. [使用 Reveal 查看越狱设备 app 界面](http://blog.csdn.net/cuibo1123/article/details/45694657)
7. [打开 Xcode 查看设备运行时日志](http://stackoverflow.com/questions/24484817/how-to-get-device-console-in-xcode6)