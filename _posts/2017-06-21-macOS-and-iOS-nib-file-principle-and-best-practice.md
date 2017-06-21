---
layout: single
author_profile: true
title: macOS 和 iOS 中 Nib 文件实现原理以及最佳实践
---


> 译者注：本文是对 Apple 官方文档的翻译，原文地址为：https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/LoadingResources/CocoaNibs/CocoaNibs.html

> iOS 和 macOS 中的 xib 文件以及 storyboard 面板最终会被 Xcode 打包系统处理生成相应的 nib 文件并放置到 ipa 包中。所以可以说本文是对这两种文件底层原理的解析。同时该文档也是实践指南。

# <a name="table-content"></a>目录

- [Nib 文件](#Nib-Files)
- [解剖 Nib 文件](#Anatomy-of-a-Nib-File)
    - [关于您的界面对象](#About-Your-Interface-Objects)
    - [关于文件的所有者](#About-the-File’s-Owner)
    - [关于第一响应者](#About-the-First-Responder)
    - [关于顶级对象](#About-the-Top-Level-Objects)
    - [关于图像和声音资源](#About-Image-and-Sound-Resources)
- [Nib 文件设计指南](#Nib-File-Design-Guidelines)
- [Nib 对象生命周期](#The-Nib-Object-Life-Cycle)
    - [对象加载过程](#The-Object-Loading-Process)
- [管理 Nib 文件中对象的生命周期](#Managing-the-Lifetimes-of-Objects-from-Nib-Files)
    - [OS X 中的顶级对象可能需要特殊处理](#Top-level-Objects-in-OS-X-May-Need-Special-Handling)
- [Action Methods](#Action-Methods)
- [Nib 文件的内置支持](#Built-In-Support-For-Nib-Files)
    - [应用程序加载主 Nib 文件](#The-Application-Loads-the-Main-Nib-File)
    - [每个视图控制器管理其自己的 Nib 文件](#Each-View-Controller-Manages-its-Own-Nib-File)
    - [文档和窗口控制器加载相关的 Nib 文件](#Document-and-Window-Controllers-Load-Their-Associated-Nib-File)
- [以编程方式加载 Nib 文件](#Loading-Nib-Files-Programmatically)
    - [使用NSBundle加载Nib文件](#Loading-Nib-Files-Using-NSBundle)
    - [使用 UINib 和 NSNib 加载 Nib 文件](#Loading-Nib-Files-Using-UINib-and-NSNib)
    - [在加载时替换代理对象](#Replacing-Proxy-Objects-at-Load-Time)
    - [访问 Nib 文件的内容](#Accessing-the-Contents-of-a-Nib-File)
- [连接 Nib 文件中的菜单项](#Connecting-Menu-Items-Across-Nib-Files)


# <a name="Nib-Files"></a> Nib 文件

在 OS X 和 iOS 创建应用程序中起着重要的作用。使用 nib 文件，您可以使用 Xcode 以图形方式创建和操作用户界面（而不是以编程方式）。因为您可以立即查看更改的结果，您可以快速尝试不同的布局和配置。您也可以稍后更改用户界面的许多方面，而无需重写任何代码。

对于使用 AppKit 或 UIKit 框架构建的应用程序，nib 文件具有重要意义。这两个框架都支持使用 nib 文件来布局窗口、视图和控件，并将这些项目与应用程序的事件处理代码集成在一起。Xcode 与这些框架配合使用，可帮助您将用户界面的控件连接到项目中，设置为您自定义的对象。该集成显着减少了加载 nib  文件后所需的设置量，并且在以后可以轻松更改代码和用户界面之间的关系。

> 注意：虽然可以在不使用 nib 文件的情况下创建 Objective-C 应用程序，但这样做非常罕见，不推荐。 根据您的应用程序，避免 nib 文件可能需要您编程大量的框架行为才能实现与使用 nib 文件相同的结果。

[↑目录](#table-content)

---

## <a name="Anatomy-of-a-Nib-File"></a> 解剖 Nib 文件

一个 nib 文件描述了应用程序的用户界面的视觉元素，包括窗口，视图，控件等等。 它还可以描述非可视元素，例如应用程序中管理窗口和视图的对象。 更重要的是，在 Xcode 中配置的对象最后会一一映射到 nib 文件描述的对象。 在运行时，这些描述用于在应用程序中重新创建对象及其配置。 当您在运行时加载 nib 文件时，您将获得 Xcode 文档中对象的精确副本。 nib 加载代码实例化对象，配置它们，并重新建立您在 nib 文件中创建的任何对象间的连接。 

以下部分描述了如何组织与 AppKit 和 UIKit 框架一起使用的 nib 文件，在其中找到的对象类型以及如何有效地使用这些对象。

### <a name="About-Your-Interface-Objects"></a> 关于您的界面对象

界面对象是添加到 nib 文件中来实现用户界面的对象。当在运行时加载一个 nib 时，界面对象是由 nib 加载代码实际实例化的对象。大多数新的 nib 文件默认情况下至少有一个界面对象，通常是窗口或菜单资源，并且可以在界面设计中添加更多的界面对象到 nib 文件。这是 nib 文件中最常见的对象类型，通常也是您为什么要使用 nib 文件的原因。

除了表示视觉对象，如窗口，视图，控件和菜单之外，界面对象还可以表示非视觉对象。在几乎所有情况下，添加到 nib 文件的非可视对象都是您的应用程序用于管理可视对象的额外控制器对象。虽然您可以在应用程序中创建这些对象，但将其添加到 nib 文件并将其配置在那里通常更为方便。 Xcode 提供了一个通用对象，特别地，您可以在将控制器和其他非可视对象添加到 nib 文件时使用。它还提供了通常用于管理 Cocoa 绑定的控制器对象。

### <a name="About-the-File’s-Owner"></a> 关于文件的所有者 

nib 文件中最重要的对象是 File's Owner 对象。 与界面对象不同，File's Owner 对象是一个占位符对象，在加载 nib 文件时不会创建。 相反，您可以在代码中创建此对象，并将其传递给 nib 加载代码。 这个对象如此重要的原因是它是应用程序代码和 nib 文件内容之间的主要链接。 更具体地说，它是负责 nib 文件内容的控制器对象。

在 Xcode 中，您可以在文件的所有者和 nib 文件中的其他接口对象之间创建连接。 加载 nib 文件时，nib 加载代码将使用您指定的替换对象重新创建这些连接。 这允许您的对象引用 nib 文件中的对象并自动从界面对象接收消息。

### <a name="About-the-First-Responder"></a> 关于第一响应者

在 nib 文件中，First Responder 是一个占位符对象，表示应用程序动态确定的响应链中的第一个对象。因为应用程序的响应链无法在设计时确定，所以第一响应者占位符充当需要针对应用程序的响应链的任何操作消息的备用目标。菜单项通常充当第一响应者占位符。例如，“窗口”菜单中的“最小化”菜单项隐藏应用程序中的最前面的窗口，而不仅仅是特定的窗口，“复制”菜单项应该复制当前选择，而不仅仅是单个控件或视图的选择。应用程序中的其他对象也可以成为第一响应者。

当您将 nib 文件加载到内存中时，您无需管理或替换第一响应者占位符对象。 AppKit 和 UIKit 框架根据应用程序的当前配置自动设置和维护第一响应者。

有关响应链的更多信息以及如何用于在基于 AppKit 的应用程序中分派事件，请参阅 [*Event Architecture*](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/EventOverview/EventArchitecture/EventArchitecture.html#//apple_ref/doc/uid/10000060i-CH3)。有关 iPhone 应用程序中的响应链和处理操作的信息，请参阅 [*Event Handling Guide for UIKit Apps*](https://developer.apple.com/library/content/documentation/EventHandling/Conceptual/EventHandlingiPhoneOS/index.html#//apple_ref/doc/uid/TP40009541)。

### <a name="About-the-Top-Level-Objects"></a> 关于顶级对象

当您的程序加载 nib 文件时，Cocoa 会重新创建 Xcode 中创建的整个对象图。该对象图包括在 nib 文件中找到的所有窗口、视图、控件、单元格、菜单和自定义对象。顶级对象是不具有父对象的这些对象的子集。顶级对象通常仅包含添加到 nib 文件中的窗口、菜单栏和自定义控制器对象。 （诸如 File's Owner，第一响应者和应用程序的对象是占位符对象，并不被视为顶级对象。）

通常，您使用 File's Owner 对象中的 outlet 来存储对 nib 文件的顶级对象的引用。如果不使用 outlet，则可以直接从 nib 加载例程中检索顶层对象。您应该始终保持一个指向这些对象的指针，因为您的应用程序负责使用完之后释放它们。有关加载时的 nib 对象行为的更多信息，请参阅 [*管理 Nib 文件中对象的生命周期*](#Managing-the-Lifetimes-of-Objects-from-Nib-Files)。

### <a name="About-Image-and-Sound-Resources"></a> 关于图像和声音资源

在 Xcode 中，您可以从 nib 文件的内容中引用外部图像和声音资源。一些控件和视图能够显示图像或播放声音作为默认配置的一部分。 Xcode 库提供对 Xcode 项目中的图像和声音资源的访问支持，以便您可以将 nib 文件链接到这些资源。 nib 文件不直接存储这些资源。相反，它存储资源文件的名称，以便 nib 加载代码稍后可以找到。

加载包含图像或声音资源引用的 nib 文件时，nib 加载代码将实际的图像或声音文件读入内存并缓存。在 OS X 中，图像和声音资源存储在命名的缓存中，以便以后可以在需要时访问它们。在 iOS 中，只有图像资源存储在命名的缓存中。要访问图像，请使用 NSImage 或 UIImage 的 `imageNamed:` 方法，具体取决于您的平台。要在OS X 中访问缓存的声音，请使用 NSSound 的 `soundNamed:` 方法。

[↑目录](#table-content)

---

## <a name="Nib-File-Design-Guidelines"></a> Nib 文件设计指南

创建 nib 文件时，请仔细考虑如何打算使用该文件中的对象。一个非常简单的应用程序可能将所有用户界面组件存储在单个 nib 文件中，但对于大多数应用程序，最好将组件分布在多个 nib 文件中。创建较小的 nib 文件可让您立即加载您需要的界面部分。它们还可以更容易地调试您可能遇到的任何问题，因为需要检查的地方较少。

创建 nib 文件时，请牢记以下准则：

- 用懒加载设计你的 nib 文件。计划加载仅包含您需要的对象的 nib 文件。
- 在 OS X 应用程序的主要 nib 文件中，请考虑仅将应用程序菜单栏和可选应用程序委托对象存储在 nib 文件中。避免在应用程序启动后包括任何不会使用的窗口或用户界面元素。相反，将这些资源放在单独的 nib 文件中，并在启动后根据需要加载它们。
- 将重复的用户界面组件（如文档窗口）存储在单独的 nib 文件中。
- 对于仅偶尔使用的窗口或菜单，将其存储在单独的 nib 文件中。通过将其存储在单独的 nib 文件中，只有在实际使用时才将资源加载到内存中。
- 使 File's Owener 对象成为 nib 文件外的任何单一联系人。请参阅 [*访问 Nib 文件的内容*](#Accessing-the-Contents-of-a-Nib-File)

[↑目录](#table-content)

---

## <a name="The-Nib-Object-Life-Cycle"></a>  Nib 对象生命周期

当 nib 文件加载到内存中时，nib 加载代码需要几个步骤来确保 nib 文件中的对象被正确创建和初始化。 了解这些步骤可以帮助您编写更好的控制器代码来管理用户界面。

### <a name="The-Object-Loading-Process"></a> 对象加载过程

当您使用 NSNib 或 NSBundle 的方法加载和实例化 nib 文件中的对象时，底层的
 nib 加载代码执行以下操作：

1. 它将 nib 文件和任何引用的资源文件的内容加载到内存中：
    - 整个 nib 对象图的原始数据被加载到内存中，但没有解档。
    - nib 文件相关联的任何自定义图像资源都将被加载并添加到 Cocoa 图像缓存中。请参阅[*关于图像和声音资源*](#About-Image-and-Sound-Resources)。
    - nib 文件相关联的任何自定义声音资源都将被加载并添加到 Cocoa 声音缓存;请参阅 [*关于图像和声音资源*](#About-Image-and-Sound-Resources)。
 
2. 它解档 nib 对象图数据并实例化对象。如何初始化每个新对象取决于对象的类型及其在归档中的编码方式。 nib 加载代码使用以下规则（按顺序）来确定要使用的初始化方法。
    
    - 默认情况下，对象会收到一个 `initWithCoder:` 消息。
        
        - 在 OS X 中，标准对象列表包括系统提供的视图，单元格，菜单和视图控制器，并且在默认的 Xcode 库中可用。它还包括使用自定义插件添加到库的任何第三方对象。即使您更改了这样一个对象的类，Xcode 将标准对象编码到 nib 文件中，然后在对象被解档时通知归档器使用改自定义类。
        
        - 在 iOS 中，使用 `initWithCoder:` 方法初始化符合 NSCoding 协议的任何对象。这包括 UIView 和 UIViewController 的所有子类，无论它们是默认的 Xcode 库或定义的自定义类的一部分。

    - OS X 中自定义视图会收到一条 `initWithFrame:` 消息。
        
        - 自定义视图是 NSView 的子类，Xcode 没有可用的实现。通常，这些是您在应用程序中定义并用于提供自定义可视内容的视图。自定义视图不包括作为默认库或集成第三方插件的一部分的标准系统视图（如 NSSlider）。
        
        - 当它遇到自定义视图时，Xcode 将一个特殊的 NSCustomView 对象编码到你的 nib 文件中。自定义视图对象包括构建您指定的真实视图子类所需的信息。在加载时，NSCustomView 对象将一个 `alloc` 和 `initWithFrame:` 消息发送到真实的视图类，然后交换自己生成的视图对象。净效果是真正的视图对象处理在 nib 加载过程中的后续交互。
        
        - iOS 中的自定义视图不会使用 `initWithFrame:`方法进行初始化。
    - 除了上述步骤中描述的以外的自定义对象接收 `init` 初始消息。
    
3. 重新建立 nib 文件中对象之间的所有连接（action、outlet 和绑定）。 这包括与 File's Owner 和其他占位符对象的连接。 建立连接的方法因平台而异：

    - Outlet 连接
        - 在 OS X 中，nib 加载代码首先尝试使用对象自己的方法重新连接 outlet。对于每个 outlet ，Cocoa 查找一个形式为 setOutletName 的方法，如果存在这样的方法，则调用它。如果找不到这样的方法，Cocoa 会在对象中搜索具有相应 outlet 名称的实例变量，并尝试直接设置值。如果找不到实例变量，则不会创建任何连接。
            设置 outlet 还会为任何注册的观察者生成键值观察（KVO）通知。这些通知可能在所有对象间连接重新建立之前发生，并且在调用任何对象的 `awakeFromNib` 方法之前肯定会发生这些通知。
        - 在 iOS 中，nib 加载代码使用 `setValue:forKey:`方法重新连接每个 outlet。该方法同样寻找适当的访问器方法，并且在失败时使用其他方式。有关此方法如何设置值的更多信息，请参阅其在 [*NSKeyValueCoding Protocol Reference*](https://developer.apple.com/documentation/foundation/object_runtime/nskeyvaluecoding) 的描述。
        - 在 iOS 中设置 outlet 还会为任何注册的观察者生成 KVO 通知。这些通知可能在所有对象间连接重新建立之前发生，并且在调用任何对象的 `awakeFromNib` 方法之前肯定会发生这些通知。
    - 动作连接
        - 在 OS X 中，nib 加载代码使用源对象的 `setTarget:` 和 `setAction:`方法来建立与目标对象的连接。如果目标对象没有响应 action 方法，则不会创建任何连接。如果目标对象为空，则该动作由响应链处理。
        - 在 iOS 中，nib 加载代码使用 UIControl 对象的 `addTarget:action:forControlEvents:` 方法来配置操作。如果目标为空，则动作由响应链处理。
    - 绑定
        - 在 OS X 中，Cocoa 使用源对象的 `bind:toObject:withKeyPath:options:` 方法来创建它与其目标对象之间的连接。
        - iOS 中不支持绑定。
    
4. 将一个 `awakeFromNib` 消息发送到 nib 文件中定义匹配选择器的相应对象：
    - 在 OS X 中，该消息被发送到定义该方法的任何界面对象。它也被发送到 File's Owner 和定义的任何占位符对象。
    - 在 iOS 中，此消息仅发送到由 nib 加载代码实例化的界面对象。它不发送到 File's Owner、第一响应者或任何其他占位符对象。
5. 显示在 nib 文件中启用了“Visible at launch time”属性的任何窗口。

nib 加载代码调用对象的 awakeFromNib 方法的顺序是不能保证的。在 OS X 中，Cocoa尝试最后调用 File‘Owner 的 awakeFromNib 方法，但不能保证该行为。如果您需要在加载时进一步配置 nib 文件中的对象，则最适合的时间是在您的 nib 加载调用返回后。这时，所有的对象都被创建，初始化并准备好使用。

[↑目录](#table-content)

---

## <a name="Managing-the-Lifetimes-of-Objects-from-Nib-Files"></a> 管理 Nib 文件中对象的生命周期

每次您要求 NSBundle 或 NSNib 类加载 nib 文件时，底层代码会创建该文件中的对象的新副本并将其返回给您。（nib 加载代码不会从以前的加载尝试中回收 nib 文件对象。）您需要确保在必要时保持新的对象图，并在完成之后将其取消。通常需要对顶级对象的强引用以确保它们不被释放。您不需要强引用对象图中较低的对象，因为它们由父母结点拥有，您应该尽可能减少引用循环的风险。

从实际的角度来看，iOS 和 OS X 的 outlet 应该被定义为声明的属性。 Outlets 通常应该是 weak 的，除了 nib 文件对应的 File's Owner 、和 nib 文件拥有的顶级对象（或 iOS 的故事板场景中）应该是强的。因此，您创建的 outlet 通常应该是 weak 的，因为：
- 例如，您创建的从视图控制器 view 或窗口控制器窗口的子视图的 outlet 是不暗示所有权的对象之间的任意引用。
- 强大的 outlet 经常由框架类指定（例如，UIViewController 的 view outlet 或 NSWindowController 的窗口 outlet）。
    
    
```
@property (weak) IBOutlet MyView *viewContainerSubview;
@property (strong) IBOutlet MyOtherClass *topLevelObject;
```

> 注意：在 OS X 中，并非所有类都支持弱引用 - 请参阅 [*Transitioning to ARC Release Notes*](https://developer.apple.com/library/content/releasenotes/ObjectiveC/RN-TransitioningToARC/Introduction/Introduction.html#//apple_ref/doc/uid/TP40011226)。 在不能指定的情况下，您应该使用 `assign` ：

 ```
 @property (assign) IBOutlet NSTextView *textView;
 ```

Outlet 通常被认为是定义类别的私有; 除非有理由公开揭露属性，否则隐藏属性声明类扩展名。 例如：


```
// MyClass.h
 
@interface MyClass : MySuperclass
@end
 
// MyClass.m
 
@interface MyClass ()
@property (weak) IBOutlet MyView *viewContainerSubview;
@property (strong) IBOutlet MyOtherClass *topLevelObject;
@end
```

这些模式扩展到从容器视图到其子视图的引用，您必须考虑对象图的内部一致性。 例如，在表视图单元格的情况下，特定子视图的 outlet 通常是 weak。 如果表视图包含图像视图和文本视图，那么这些视图仍然有效，只要它们是表视图单元本身的子视图。 

当 outlet 被认为拥有参考对象时，outlet 应变为 strong：

- 如前所述，通常情况下，top level 的对象在 nib 文件中经常被认为是由 File's Owner 拥有的。
- 在某些情况下，您可能需要一个来自 nib 文件的对象存在其原始容器之外。 例如，您可能有一个视图的 outlet，可以临时从其初始视图层次结构中删除，因此必须独立进行维护。
    
您希望被子类化的类（特别是抽象类）公开暴露 outlet，以便它们可以被子类（例如UIViewController 的视图 outlet）适当地使用。 如果使用者需要访问属性，则 outlet 也可能会暴露出来;.例如，表视图单元格可能会暴露子视图。 在后一种情况下，可以适合公开一个以私有定义可读的只读公共 outlet ，例如：


```
// MyClass.h
 
@interface MyClass : UITableViewCell
@property (weak, readonly) MyType *outletName;
@end
 
// MyClass.m
 
@interface MyClass ()
@property (weak, readwrite) IBOutlet MyType *outletName;
@end

```

### <a name="Top-level-Objects-in-OS-X-May-Need-Special-Handling"></a> OS X 中的顶级对象可能需要特殊处理

由于历史原因，在 OS X 中，将创建一个 nib 文件的顶级对象，并附加一个引用计数。应用开发框架提供了几个功能，可帮助确保正确释放 nib 对象：

- NSWindow 对象（包括面板）有一个 isReleasedWhenClosed 属性，如果设置为YES，则会在窗口关闭时指示窗口释放自身（以及其视图层次结构中的所有相关对象）。在 nib 文件中，您可以通过 Xcode inspector 的“Attribute”窗格中的“Release when closed ”复选框来设置此选项。

- 如果 nib 文件的 File's Owner 是 NSWindowController 对象（在基于文档的应用程序中的文档 nib 中的默认值，请记住 NSDocument 管理 NSWindowController 的一个实例）或 NSViewController 对象，它会自动处理其管理的窗口。

如果 File's Owner 不是 NSWindowController 或 NSViewController 的实例，那么您需要自己递减顶级对象的引用计数。您必须将顶级对象的引用转换为 Core Foundation 类型并使用 CFRelease。 （如果不希望有所有顶级对象的 outlet ，可以使用 NSNib 类的 `instantiateNibWithOwner:topLevelObjects:`方法来获取一个 nib文件顶层对象的数组。）

[↑目录](#table-content)

---


## <a name="Action-Methods"></a> Action Methods

一般来说，action method（参见 OS X 中的 Target-Action 或 iOS 中的 Target-Action）是通常由 nib 文件中另一个对象调用的方法。 Action method 使用类型限定符 IBAction(void类型的替代)，将声明的方法标记为一个 action method，以便Xcode 知道它。


```
@interface MyClass
- (IBAction)myActionMethod:(id)sender;
@end
```

您可以选择将 action method 视为您的类私有的，因此不会在 public @interface 中声明它们。 （因为 Xcode 解析实现文件，所以不需要在头文件中声明它们。）


```
// MyClass.h
 
@interface MyClass
@end
 
// MyClass.m
 
@implementation MyClass
- (IBAction)myActionMethod:(id)sender {
    // Implementation.
}
@end
```

您通常不应以编程方式调用 action method。 如果您的类需要执行与 action method相关联的工作，那么您应该将实现应用到另一种由 action mehtod 调用的方法中。


```
// MyClass.h
 
@interface MyClass
@end
 
// MyClass.m
 
@interface MyClass (PrivateMethods)
- (void)doSomething;
- (void)doWorkThatRequiresMeToDoSomething;
@end
 
@implementation MyClass
- (IBAction)myActionMethod:(id)sender {
    [self doSomething];
}
 
- (void)doSomething {
    // Implementation.
}
 
- (void)doWorkThatRequiresMeToDoSomething {
    // Pre-processing.
    [self doSomething];
    // Post-processing.
}
 
@end

```

[↑目录](#table-content)

---

## <a name="Built-In-Support-For-Nib-Files"></a> Nib 文件的内置支持

AppKit 和 UIKit 框架都提供了一定量的自动化行为来加载和管理应用程序中的 nib 文件。 这两个框架都提供了用于加载应用程序主要 nib 文件的基础设施。 此外，AppKit 框架提供了通过 NSDocument 和 NSWindowController 类加载其他 nib 文件的支持。 以下部分介绍了 nib 文件的内置支持，如何利用它们，以及在自己的应用程序中修改该支持的方法。

### <a name="The-Application-Loads-the-Main-Nib-File"></a> 应用程序加载主 Nib 文件

大多数 Xcode 项目模板都已预先为应用程序配置了一个主要的 nib 文件。所有您需要做的是修改默认 nib 文件并构建您的应用程序。在启动时，应用程序的默认配置数据告诉应用程序对象找到该 nib 文件，以便加载它。在基于 AppKit 和 UIKit 的应用程序中，此配置数据位于应用程序的 Info.plist 文件中。首次加载应用程序时，应用程序启动代码默认会在 Info.plist 文件中查找 NSMainNibFile 键。如果找到它，它会在应用程序包中查找一个 nib 文件，其名称（带或不带文件扩展名）与该键的值匹配，然后加载它。

### <a name="Each-View-Controller-Manages-its-Own-Nib-File"></a> 每个视图控制器管理其自己的 Nib 文件

UIViewController（iOS）和NSViewController（OS X）类支持自动加载其关联的 nib文件。如果在创建视图控制器时指定 nib 文件，则当您尝试访问视图控制器的视图时，该 nib 文件会自动加载。视图控制器和 nib 文件对象之间的任何连接都将自动创建，在 iOS 中，当视图最终加载并显示在屏幕上时，UIViewController 对象还会收到其他通知。为了更好地管理内存，UIViewController 类还可以在低内存条件下处理卸载其 nib 文件（如适用）。

有关如何使用 UIViewController 类及其配置方式的更多信息，请参阅 [*View Controller Programming Guide for iOS*](https://developer.apple.com/library/content/featuredarticles/ViewControllerPGforiPhoneOS/index.html#//apple_ref/doc/uid/TP40007457)。


### <a name="Document-and-Window-Controllers-Load-Their-Associated-Nib-File"></a> 文档和窗口控制器加载相关的 Nib 文件

在 AppKit 框架中，NSDocument 类与默认窗口控制器一起加载包含文档窗口的 nib 文件。 NSDocument的 windowNibName 方法是一种方便的方法，您可以使用它来指定包含相应文档窗口的 nib 文件。创建新文档时，文档对象将您指定的 nib 文件名传递给默认的窗口控制器对象，该对象加载并管理 nib 文件的内容。如果您使用Xcode 提供的标准模板，您唯一需要做的是将文档窗口的内容添加到 nib 文件中。

NSWindowController 类还提供自动支持加载 nib 文件。如果以编程方式创建自定义窗口控件，则可以选择使用 NSWindow 对象或 nib 文件的名称初始化它们。如果选择后一个选项，NSWindowController 类会在客户端首次尝试访问窗口时自动加载指定的 nib 文件。之后，窗口控制器将窗口保持在内存中。即使窗口的“Release when closed”属性被设置，它也不会从 nib 文件重新加载它。

> 重要：当使用 NSWindowController 或 NSDocument 自动加载窗口时，重要的是您的 nib 文件配置正确。 这两个类都包括一个窗口 outlet，您必须连接到您要管理的窗口。 如果不将此 outlet 连接到窗口对象，则 nib 文件已加载，但文档或窗口控制器不显示窗口。 有关 Cocoa 文档体系结构的更多信息，请参阅 [*Document-Based App Programming Guide for Mac*](https://developer.apple.com/library/content/documentation/DataManagement/Conceptual/DocBasedAppProgrammingGuideForOSX/Introduction/Introduction.html#//apple_ref/doc/uid/TP40011179) 。

[↑目录](#table-content)

---

## <a name="Loading-Nib-Files-Programmatically"></a> 以编程方式加载 Nib 文件 

OS X 和 iOS 都提供了将 nib 文件加载到应用程序中的便利方法。 AppKit 和 UIKit 框架都在 NSBundle 类上定义了支持加载 nib 文件的附加方法。 此外，AppKit 框架还提供了 NSNib 类，它提供与 NSBundle 类似的 nib 加载行为，但提供了在特定情况下可能有用的一些其他优点。 

在设计应用程序时，请确保手动加载的任何 nib 文件都以简化加载过程的方式进行配置。 为 File's Owner 选择一个适当的对象并保持你的 nib 文件很小可以大大提高它们的易用性和内存效率。 有关配置 nib 文件的更多提示，请参阅 [*Nib 文件设计指南*](#Nib-File-Design-Guidelines)。

### <a name="Loading-Nib-Files-Using-NSBundle"></a> 使用NSBundle加载Nib文件 

AppKit 和 UIKit 框架在 NSBundle 类（使用 Objective-C 类别）上定义了其他方法来支持加载 nib 文件资源。 两平台之间使用方法的语义与方法的语法不同。 在 AppKit 中，通常有更多的选项可以访问 bundle，因此还有更多的方法可以从这些 bundle 加载 nib 文件。 在 UIKit 中，应用程序只能从 main bundle 装载 nib 文件，因此需要较少的选项。 两种平台上可用的方法如下：

- AppKit:
    - `loadNibNamed:owner:` 类方法
    - `loadNibFile:externalNameTable:withZone:` 类方法
    - `loadNibFile:externalNameTable:withZone:` 对象方法

- UIKit
    - `loadNibNamed:owner:options:` instance方法

每当加载 nib 文件时，都应该始终指定一个对象作为该 nib 文件的 File's Owner。File's Owner 的作用很重要的。它是运行代码和即将在内存中创建的新对象之间的主要接口。所有的 nib 加载方法提供了一种方法来直接指定 Files' Owner，或者作为选项字典中的参数。

AppKit 和 UIKit 框架处理 nib 加载的方式之间的语义差异之一就是顶层的 nib 对象返回到应用程序的方式。在 AppKit 框架中，您必须使用 `loadNibFile:externalNameTable:withZone:methods` 显式请求它们。在 UIKit 中，`loadNibNamed:owner:options:` 方法直接返回这些对象的数组。在这两种情况下避免担心顶层对象的最简单的方法是将它们存储在 File's Owner 的 outlets 中（请参阅 [*管理 Nib 文件中对象的生命周期*](#Managing-the-Lifetimes-of-Objects-from-Nib-Files)）。

清单1-1 显示了一个简单的例子，说明如何使用基于 AppKit 的应用程序中的 NSBundle 类加载 nib 文件。 `loadNibNamed:owner:`方法返回后，可以使用任何引用 nib 文件对象的 outlet。换句话说，整个 nib 加载过程发生在该单个调用的限制内。 AppKit 框架中的 nib 加载方法返回一个布尔值，以指示加载操作是否成功。


清单 1-1  Loading a nib file from the current bundle

```
- (BOOL)loadMyNibFile
{
    // The myNib file must be in the bundle that defines self's class.
    if (![NSBundle loadNibNamed:@"myNib" owner:self])
    {
        NSLog(@"Warning! Could not load myNib file.\n");
        return NO;
    }
    return YES;

```

清单 1-2 显示了如何在基于 UIKit 的应用程序中加载 nib 文件的示例。 在这种情况下，该方法将检查返回的数组以查看 nib 对象是否已成功加载。 （每个 nib 文件应至少有一个表示 nib 文件内容的顶级对象。）此示例显示了当文件所有者对象不包含占位符对象时的简单情况。 有关如何指定其他占位符对象的示例，请参阅[*在加载时替换代理对象*](#Replacing-Proxy-Objects-at-Load-Time)。

清单 1-2  Loading a nib in an iPhone application

```
- (BOOL)loadMyNibFile
{
    NSArray*    topLevelObjs = nil;
 
    topLevelObjs = [[NSBundle mainBundle] loadNibNamed:@"myNib" owner:self options:nil];
    if (topLevelObjs == nil)
    {
        NSLog(@"Error! Could not load myNib file.\n");
        return NO;
    }
    return YES;
}
```

> 注意：如果您正在开发适用于 iOS 的通用应用程序，则可以使用特定于设备的命名约定自动为底层设备加载正确的 nib 文件。 有关如何命名 nib 文件的更多信息，请参阅[*iOS Supports Device-Specific Resources*](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/LoadingResources/Introduction/Introduction.html#//apple_ref/doc/uid/10000051i-CH1-SW2) 。

### <a name="Getting-a-Nib-File’s-Top-Level-Objects"></a> 获取 Nib 文件的顶级对象 

获取 nib 文件的顶级对象的最简单方法是在 File's Owner 对象中定义 outlets 以及用于访问这些对象的 setter 方法（或属性）。 此方法可确保顶层对象由对象保留，并始终对其进行引用。

清单 1-3 显示了使用 outlet 保留 nib 文件唯一顶级对象的简化 Cocoa 类的接口和实现。 在这种情况下，nib 文件中唯一的顶级对象是 NSWindow对象。 因为 Cocoa 中的顶级对象的初始引用计数为1，因此会包含一个额外的释放消息。 这很好，因为在发出调用的时候，该属性已被保留在窗口中。 您不会希望以这种方式在 iPhone 应用程序中释放顶级对象。

清单 1-3  Using outlets to get the top-level objects


```
// Class interface.
@interface MyController : NSObject
- (void)loadMyWindow;
@end
 
// Private class extension.
@interface MyController ()
@property (strong) IBOutlet NSWindow *window;
@end
 
 
// Class implementation
@implementation MyController
 
- (void)loadMyWindow {
    [NSBundle loadNibNamed:@"myNib" owner:self];
 
    // The window starts off with a retain count of 1
    // and is then retained by the property, so add an extra release.
    NSWindow *window = self.window;
    CFRelease(__bridge window);
}
@end
```

如果您不想使用 outlet 来存储对 nib 文件的顶级对象的引用，则必须在代码中手动检索这些对象。 获取顶级对象的技术因目标平台而异。 在 OS X 中，您必须明确要求对象，而在 iOS 中，它们将自动返回给您。

清单 1-4 显示了在 OS X 中获取 nib 文件的顶级对象的过程。此方法将可变数组放入 nameTable 字典中，并将其与 NSNibTopLevelObjects 关键字相关联。 nib 加载代码查找此数组对象，如果存在，将顶级对象放在其中。 因为每个对象在添加到数组之前以保留计数为1开始，所以简单地释放数组也不足以释放数组中的对象。 因此，此方法向每个对象发送发送消息，以确保数组是唯一持有对它们的引用的实体。

清单 1-4  Getting the top-level objects from a nib file at runtime


```
- (NSArray*)loadMyNibFile
{
    NSBundle*            aBundle = [NSBundle mainBundle];
    NSMutableArray*      topLevelObjs = [NSMutableArray array];
    NSDictionary*        nameTable = [NSDictionary dictionaryWithObjectsAndKeys:
                                            self, NSNibOwner,
                                            topLevelObjs, NSNibTopLevelObjects,
                                            nil];
 
    if (![aBundle loadNibFile:@"myNib" externalNameTable:nameTable withZone:nil])
    {
        NSLog(@"Warning! Could not load myNib file.\n");
        return nil;
    }
 
    // Release the objects so that they are just owned by the array.
    [topLevelObjs makeObjectsPerformSelector:@selector(release)];
    return topLevelObjs;
}
```

获取 iPhone 应用程序中的顶级对象要简单得多，如清单 1-2 所示。在 UIKit 框架中，NSBundle 的 loadNibNamed:owner:options: 方法自动返回一个包含顶级对象的数组。另外，在返回数组之前，对对象的保留计数进行调整，以便不需要向每个对象发送额外的释放消息。返回的数组是对象的唯一所有者。

### <a name="Loading-Nib-Files-Using-UINib-and-NSNib"></a> 使用 UINib 和 NSNib 加载 Nib 文件

在要创建 nib 文件内容的多个副本的情况下，UINib（iOS）和 NSNib（OS X）类提供更好的性能。正常的 nib 加载过程涉及从磁盘读取 nib 文件，然后实例化其包含的对象。然而，使用 UINib 和 NSNib 类，从磁盘读取 nib 文件只要一次，并将内容存储在内存中。因为它们在内存中，创建连续的对象集需要更少的时间，因为它不需要访问磁盘。

使用 UINib 和 NSNib 类始终是一个两步的过程。首先，您创建一个类的实例，并使用 nib 文件的位置信息进行初始化。其次，您将实例化 nib 文件的内容以将对象加载到内存中。每次实例化 nib 文件时，都需要指定一个不同的 File's Owner 对象并接收一组新的顶级对象。

清单 1-5 显示了使用 OS X 中的 NSNib 类加载 nib 文件的内容的一种方法。由instantiateNibWithOwner ：topLevelObjects：方法返回给您的数组已经自动释放。如果您打算使用该数组任何一段时间，您应该复制一份。

清单 1-5 Loading a nib file using NSNib


```
- (NSArray*)loadMyNibFile
{
    NSNib*      aNib = [[NSNib alloc] initWithNibNamed:@"MyPanel" bundle:nil];
    NSArray*    topLevelObjs = nil;
 
    if (![aNib instantiateNibWithOwner:self topLevelObjects:&topLevelObjs])
    {
        NSLog(@"Warning! Could not load nib file.\n");
        return nil;
    }
    // Release the raw nib data.
    [aNib release];
 
    // Release the top-level objects so that they are just owned by the array.
    [topLevelObjs makeObjectsPerformSelector:@selector(release)];
 
    // Do not autorelease topLevelObjs.
    return topLevelObjs;
}
```

### <a name="Replacing-Proxy-Objects-at-Load-Time"></a> 在加载时替换代理对象 

在 iOS 中，可以创建除 File's Owner 之外的包括占位符对象的 nib 文件。 代理对象表示在 nib 文件外部创建但与 nib 文件内容有某种连接的对象。 代理通常用于支持 iPhone 应用程序中的导航控制器。 当使用导航控制器时，您通常将文件的所有者对象连接到某些常见对象（如应用程序委托）。 因此，代理对象表示导航控制器对象层次结构中已经加载到内存中的部分，因为它们是以编程方式创建的(也可以从不同的 nib 文件加载)。

> 注意：OS X nib 文件不支持自定义占位符对象（File's Owner 除外）。

您添加到 nib 文件的每个占位符对象必须具有唯一的名称。要为对象分配名称，请选择 Xcode 中的对象并打开 insepector 窗口。insepector 的“Attributes”窗格包含一个“Name”字段，用于指定占位符对象的名称。您分配的名称应描述对象的行为或类型，但实际上它可以是任何您想要的。

当您准备加载包含占位符对象的 nib 文件时，必须在调用 `loadNibNamed:owner:options:method` 时指定任何代理的替换对象。此方法的options 参数接受附加信息的字典。您可以使用此字典传递有关占位符对象的信息。字典必须包含 UINibExternalObjects 键，其值是另一个包含每个占位符替换的名称和对象的字典。

清单 1-6 显示了一个 applicationDidFinishLaunching: 方法的示例版本，用于手动加载应用程序的主 nib 文件。因为应用程序的委托对象是由 UIApplicationMain 函数创建的，所以该方法在主 nib 文件中使用占位符（名称为“AppDelegate”）来表示该对象。代理字典存储占位符对象信息，并且选项字典包含该字典。

清单 1-6  Replacing placeholder objects in a nib file


```
- (void)applicationDidFinishLaunching:(UIApplication *)application
{
    NSArray*    topLevelObjs = nil;
    NSDictionary*    proxies = [NSDictionary dictionaryWithObject:self forKey:@"AppDelegate"];
    NSDictionary*    options = [NSDictionary dictionaryWithObject:proxies forKey:UINibExternalObjects];
 
    topLevelObjs = [[NSBundle mainBundle] loadNibNamed:@"Main" owner:self options:options];
    if ([topLevelObjs count] == 0)
    {
        NSLog(@"Warning! Could not load myNib file.\n");
        return;
    }
 
    // Show window
    [window makeKeyAndVisible];
}
```

有关 `loadNibNamed:owner:options:` 方法中选项字典的更多信息，请参阅 *NSBundle UIKit Additions Reference*。

### <a name="Accessing-the-Contents-of-a-Nib-File"></a> 访问 Nib 文件的内容

成功加载 nib 文件后，其内容就可以立即使用。如果您在 File's Owner 中配置了 outlet 来指向 nib 文件中的对象，那么现在可以使用这些 outlet。如果您没有使用任何 outlet 配置 File's Owner ，则应确保以某种方式获取对顶级对象的引用，以便稍后释放它们。

因为 outlets 在加载 nib 文件时填充实际对象，所以随后可以像您以编程方式创建的任何其他对象一样使用 outlet。例如，如果您有一个指向窗口的 outlet，则可以将窗口发送一个 `makeKeyAndOrderFront:` 消息，以在用户屏幕上显示该消息。完成使用
nib 文件中的对象后，您必须像任何其他对象一样释放它们。

> 重要提示：使用这些对象后，您将负责释放加载的任何 nib 文件的顶级对象。不这样做是许多应用程序中内存泄漏的原因。释放顶级对象后，将 nib 文件中指向对象的任何 outlet 都清除为 nil （weak 属性的不需要手动处理），这是一个好主意。您应该清除与所有 nib 文件对象相关联的 outlet，而不仅仅是顶级对象。

[↑目录](#table-content)

---

## <a name="Connecting-Menu-Items-Across-Nib-Files"></a> 连接 Nib 文件中的菜单项

OS X 应用程序菜单栏中的项目通常需要与许多不同的对象进行交互，包括应用程序的文档和窗口。问题是许多这些对象不能（或不应该）直接从主 nib 文件访问。主 nib 文件的 File's Owner 始终设置为 NSApplication 类的一个实例。虽然您可能能够在主 nib 文件中实例化一些自定义对象，但这样做是不切实际或不必要的。在文档对象的情况下，直接连接到特定文档对象是不可能的，因为文档对象的数量可以动态地更改，甚至可以为 nil。

大多数菜单项将动作消息发送到以下之一：

- 一个总是处理命令的固定对象
- 动态对象，如文档或窗口

消息传递给固定对象是一个相对简单的过程，通常最好通过应用程序委托来处理。应用程序委托对象在运行应用程序时协助 NSApplication 对象，并且是少数几个包括在主 nib 文件的。如果菜单项是指应用程序级命令，则可以直接在应用程序委托中实现该命令，或者只需让代理将消息转发到应用程序其他位置的相应对象。

如果您有一个菜单项作用于最前面的窗口的内容，则需要将菜单项链接到第一个响应者占位符对象。如果与菜单项相关联的操作方法特定于您的一个对象（而不是由Cocoa 定义），则必须在创建连接之前将该操作添加到第一响应程序。

创建连接后，您需要在自定义类中实现操作方法。该对象还应实现validateMenuItem: 方法，以在适当的时间启用菜单项。有关响应链如何处理命令的更多信息，请参阅 [*Cocoa Event Handling Guide*](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/EventOverview/Introduction/Introduction.html#//apple_ref/doc/uid/10000060i)。

[↑目录](#table-content)

---