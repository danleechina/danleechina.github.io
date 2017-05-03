---
layout: single
author_profile: true
title: 一个实现极为简单的约束框架
---

# <a name="table-content"></a>目录

- [起因](#reason-for-this)
- [结果](#result-for-this)
- [如何封装 `autolayout`](#how-to-encapsulate-autolayout)
    - [使用原生约束 API 的问题](#problem-about-original-api)
    - [使用 `Masonry`](#using-masonry)
    - [使用 `DLSimpleAutolayout`](#using-dlsimpleautolayout)
    - [`DLSimpleAutolayout` 实现原理](#how-to-implement-DLSimpleAutolayout)
        - [基本原理](#principle-autolayout)
        - [`DLSimpleAutolayout` 的实现之 `DLSimpleViewAttribute`](#principle-DLSimpleViewAttribute)
        - [`DLSimpleAutolayout` 的实现之 `NSArray` 类别](#principle-NSArray-category)
        - [`DLSimpleAutolayout` 的实现之 `UIView` 类别](#principle-UIView-category)

	
# <a name="reason-for-this"></a> 起因

开发 iOS SDK 经常需要注意引入的开源代码或者定义的类名和方法名，可能会和用户代码冲突。所以一般的做法是类名，类别方法名加前缀。

最近在开发 SDK 的时候需要写一点界面，所以在思考用什么来布局。考虑使用 `Masonry`。但是引入 `Masonry` 需要对 `Masonry` 里面所有类、类别方法加前缀，成本太大，引入错误风险太高。同时 SDK 里面的界面通常比较简单，数量也比较少，引入一个 `Masonry`  反而会增大编译出来的二进制文件大小，不太值得。所以我就在思考如何设计一个代码精简的，类似 `Masonry` 的约束框架呢？

[↑目录](#table-content)

# <a name="result-for-this"></a> 结果

在阅读了 `Masonry` 的代码之后，我决定仿照它来重新写一个精简的约束框架，这里的精简是代码量极少。所以就有了后面的 `DLSimpleAutolayout`。这个约束框架代码总量大约只有300行。但是使用方式基本和 `Masonry` 类似，下面是 `DLSimpleAutolayout` 写出来的布局代码样式。

 ```
UIEdgeInsets padding = UIEdgeInsetsMake(10, 10, 10, 10);
 @[
  view1.dl_top.equalTo(superview).offset(padding.top),
  view1.dl_left.equalTo(superview).offset(padding.left),
  view1.dl_bottom.equalTo(superview).offset(-padding.bottom),
  view1.dl_right.equalTo(superview).offset(-padding.right),
].dl_constraint_install();
 
 ```
 
[↑目录](#table-content)

***
 
# <a name="how-to-encapsulate-autolayout"></a> 如何封装 `autolayout` 
 
下面我会循序渐进地讲解为什么，以及如何封装 `autolayout` 

## <a name="problem-about-original-api"></a> 使用原生约束 API 的问题

在 iOS 上面使用原生约束写代码是极为繁琐的一件事。比如下面这样的原生约束代码（来自 [Masonry README.md](https://github.com/SnapKit/Masonry/blob/master/README.md)）：

```
UIView *superview = self.view;

UIView *view1 = [[UIView alloc] init];
view1.translatesAutoresizingMaskIntoConstraints = NO;
view1.backgroundColor = [UIColor greenColor];
[superview addSubview:view1];

UIEdgeInsets padding = UIEdgeInsetsMake(10, 10, 10, 10);

[superview addConstraints:@[

    //view1 constraints
    [NSLayoutConstraint constraintWithItem:view1
                                 attribute:NSLayoutAttributeTop
                                 relatedBy:NSLayoutRelationEqual
                                    toItem:superview
                                 attribute:NSLayoutAttributeTop
                                multiplier:1.0
                                  constant:padding.top],

    [NSLayoutConstraint constraintWithItem:view1
                                 attribute:NSLayoutAttributeLeft
                                 relatedBy:NSLayoutRelationEqual
                                    toItem:superview
                                 attribute:NSLayoutAttributeLeft
                                multiplier:1.0
                                  constant:padding.left],

    [NSLayoutConstraint constraintWithItem:view1
                                 attribute:NSLayoutAttributeBottom
                                 relatedBy:NSLayoutRelationEqual
                                    toItem:superview
                                 attribute:NSLayoutAttributeBottom
                                multiplier:1.0
                                  constant:-padding.bottom],

    [NSLayoutConstraint constraintWithItem:view1
                                 attribute:NSLayoutAttributeRight
                                 relatedBy:NSLayoutRelationEqual
                                    toItem:superview
                                 attribute:NSLayoutAttributeRight
                                multiplier:1
                                  constant:-padding.right],

 ]];

```

问题出在苹果提供的那个特别长的函数。当你的视图多的时候，像上面这样的代码就会越来越庞大，可读性也越来越低。

[↑目录](#table-content)

## <a name="using-masonry"></a> 使用 `Masonry`

那么如何更加优雅的使用 `autolayout` 呢？绝大多数的 iOS 开发如果使用 `autolayout` 来布局视图的话，肯定或多或少的听说过 [`Masonry`](https://github.com/SnapKit/Masonry) （如果是 `Swift` 版本的话，官方推荐 [`SnapKit`](https://github.com/SnapKit/SnapKit)）。

利用 `Masonry`，我们可以将上面约束代码写成下面这样（来自 [Masonry README.md](https://github.com/SnapKit/Masonry/blob/master/README.md)）：

```

UIEdgeInsets padding = UIEdgeInsetsMake(10, 10, 10, 10);

[view1 mas_makeConstraints:^(MASConstraintMaker *make) {
    make.top.equalTo(superview.mas_top).with.offset(padding.top); //with is an optional semantic filler
    make.left.equalTo(superview.mas_left).with.offset(padding.left);
    make.bottom.equalTo(superview.mas_bottom).with.offset(-padding.bottom);
    make.right.equalTo(superview.mas_right).with.offset(-padding.right);
}];

```

 `Masonry` 会帮助我们将约束添加到适当的视图上（比如自身或者和另一个视图的公共祖先），同时也会帮助我们设置相应视图的 `translatesAutoresizingMaskIntoConstraints` 为 `NO`。
 
[↑目录](#table-content)
 
## <a name="using-dlsimpleautolayout"></a> 使用 `DLSimpleAutolayout`
 
 [`DLSimpleAutolayout`](https://github.com/danleechina/DLSimpleAutolayout) 是一个使用方式类似 `Masonry` 的约束框架，下面是 `DLSimpleAutolayout` 的约束代码（等价于上面的 `Masonry` 代码）：
 
 ```
UIEdgeInsets padding = UIEdgeInsetsMake(10, 10, 10, 10);
 @[
  view1.dl_top.equalTo(superview).offset(padding.top),
  view1.dl_left.equalTo(superview).offset(padding.left),
  view1.dl_bottom.equalTo(superview).offset(-padding.bottom),
  view1.dl_right.equalTo(superview).offset(-padding.right),
].dl_constraint_install();
 
 ```
 
 是不是看起来一摸一样？！！
 
 但是 `DLSimpleAutolayout` 的实现极为简单，只有一个头文件和一个实现文件，合计代码行数只有300行左右。
 
[↑目录](#table-content)
 
## <a name="how-to-implement-DLSimpleAutolayout"></a> `DLSimpleAutolayout` 实现原理

### <a name="principle-autolayout"></a> 基本原理

首先你需要了解布局的[基本原理](https://developer.apple.com/library/content/documentation/UserExperience/Conceptual/AutolayoutPG/AnatomyofaConstraint.html#//apple_ref/doc/uid/TP40010853-CH9-SW1)

简单的说每一个相等关系的约束最后会被 Apple 处理成一条类似下面的等式：

```
view1.left = 2.0 * view2.left + 8.0
```

上面这条公式的意思是 `view1` 的 `left` 约束等于 `view2` 的 `left` 值乘以 2 加 8。这里的 2 叫 multiplier，8 叫 constant，view1 叫 first item，view2 叫 second item，也就是分别对应

```
+(instancetype)constraintWithItem:(id)view1 attribute:(NSLayoutAttribute)attr1 relatedBy:(NSLayoutRelation)relation toItem:(nullable id)view2 attribute:(NSLayoutAttribute)attr2 multiplier:(CGFloat)multiplier constant:(CGFloat)c;
```
这个原生 API 的几个参数。无论是 `DLSimpleAutolayout` 还是 `Masonry` 的实现，基本上都是对这个 API 的封装，同时添加了一些其他代码，方便使用者使用约束。

[↑目录](#table-content)

### <a name="principle-DLSimpleViewAttribute"></a> `DLSimpleAutolayout` 的实现之 `DLSimpleViewAttribute`

`DLSimpleAutolayout` 的实现其实参考了 `Masonry`，比如 `DLSimpleViewAttribute` 类似与 `MASViewAttribute`。`DLSimpleViewAttribute` 类型接口如下：

```

@interface DLSimpleViewAttribute : NSObject

@property (nonatomic, strong, readonly) DLSimpleViewAttribute *(^equalTo)(id view);
@property (nonatomic, strong, readonly) DLSimpleViewAttribute *(^lessThanOrEqualTo)(id view);
@property (nonatomic, strong, readonly) DLSimpleViewAttribute *(^greaterThanOrEqualTo)(id view);
@property (nonatomic, strong, readonly) DLSimpleViewAttribute *(^multipliedBy)(CGFloat multipier);
@property (nonatomic, strong, readonly) DLSimpleViewAttribute *(^offset)(CGFloat offset);
@property (nonatomic, strong, readonly) NSLayoutConstraint *(^install)();

@end

```

也就是对应约束的3种大小关系，同时可以对 constant 进行 offset 操作（对应设置 constant），对 second item 有 multipliedBy 操作（对应设置 multiplier），最后添加约束的时候需要手动 `install()` 。比如下面这样：

 ```
UIEdgeInsets padding = UIEdgeInsetsMake(10, 10, 10, 10);
view1.dl_top.equalTo(superview).offset(padding.top).install();
view1.dl_left.equalTo(superview).offset(padding.left).install();
view1.dl_bottom.equalTo(superview).offset(-padding.bottom).install();
view1.dl_right.equalTo(superview).offset(-padding.right).install();
 
 ```

当调用 `install()` 的时候，`DLSimpleAutolayout` 会根据当前配置的信息，来生成一个约束，同时将约束添加到自身或者是和 second item 的最近的公共视图上。并设置自身的 `translatesAutoresizingMaskIntoConstraints` 为 `NO`。代码如下所示：

```
- (NSLayoutConstraint * (^)())install {
    return ^id(){
        // 设置为 NO
        ((UIView *)self.firstItem).translatesAutoresizingMaskIntoConstraints = NO;
        NSLayoutConstraint *constaraint = [NSLayoutConstraint constraintWithItem:self.firstItem
                                                                       attribute:self.firstAttribute
                                                                       relatedBy:self.relation
                                                                          toItem:self.secondItem
                                                                       attribute:self.secondAttribute
                                                                      multiplier:self.multiplier
                                                                        constant:self.constant];
        if (self.secondItem == nil) {
            [self.firstItem addConstraint:constaraint];
        } else {
            // 寻找最近的公共祖先
            UIView *closestCommonSuperview = [self dl_ClosestCommonSuperview:self.firstItem view2:self.secondItem];
            [closestCommonSuperview addConstraint:constaraint];
        }
        return constaraint;
    };
}

```

[↑目录](#table-content)

### <a name="principle-NSArray-category"></a> `DLSimpleAutolayout` 的实现之 NSArray 类别

考虑到方便在每个属性后面自动使用 `install()`，我对 NSArray 添加了一个类别，封装了这个操作。所以无需在每个 `DLSimpleViewAttribute` 对象后调用 `install()`，只需要将这些对象放到一个 NSArray 里面，对这个 array 调用 `dl_constraint_install()` 即可。这种类似语法糖的写法也方便了使用者更好的组织、调整自己的布局代码。 

该 NSArray 类别公开接口如下：

```
@interface NSArray (DLSimpleAutoLayout)

@property (nonatomic, strong, readonly) NSArray<NSLayoutConstraint *> * (^dl_constraint_install)();

@end
```

写出来的代码类似下面这样：


 ```
UIEdgeInsets padding = UIEdgeInsetsMake(10, 10, 10, 10);
 @[
  view1.dl_top.equalTo(superview).offset(padding.top),
  view1.dl_left.equalTo(superview).offset(padding.left),
  view1.dl_bottom.equalTo(superview).offset(-padding.bottom),
  view1.dl_right.equalTo(superview).offset(-padding.right),
].dl_constraint_install();
 
 ```
 
[↑目录](#table-content)

#### <a name="principle-UIView-category"></a> `DLSimpleAutolayout` 的实现之 `UIView` 类别

`DLSimpleAutolayout` 为 `UIView` 添加了类别，就是类似 `dl_xxx` 这样的只读属性。
并且每个 view 返回的 `dl_xxx` 都是一个 `DLSimpleViewAttribute` 类型。该类别的接口如下：

```

@interface UIView (DLSimpleAutoLayout)

@property (nonatomic, strong, readonly) DLSimpleViewAttribute *dl_left;
@property (nonatomic, strong, readonly) DLSimpleViewAttribute *dl_top;
@property (nonatomic, strong, readonly) DLSimpleViewAttribute *dl_right;
@property (nonatomic, strong, readonly) DLSimpleViewAttribute *dl_bottom;
@property (nonatomic, strong, readonly) DLSimpleViewAttribute *dl_leading;
@property (nonatomic, strong, readonly) DLSimpleViewAttribute *dl_trailing;
@property (nonatomic, strong, readonly) DLSimpleViewAttribute *dl_width;
@property (nonatomic, strong, readonly) DLSimpleViewAttribute *dl_height;
@property (nonatomic, strong, readonly) DLSimpleViewAttribute *dl_centerX;
@property (nonatomic, strong, readonly) DLSimpleViewAttribute *dl_centerY;
@property (nonatomic, strong, readonly) DLSimpleViewAttribute *dl_baseline;

#if TARGET_OS_IPHONE || TARGET_OS_TV

@property (nonatomic, strong, readonly) DLSimpleViewAttribute *dl_leftMargin;
@property (nonatomic, strong, readonly) DLSimpleViewAttribute *dl_rightMargin;
@property (nonatomic, strong, readonly) DLSimpleViewAttribute *dl_topMargin;
@property (nonatomic, strong, readonly) DLSimpleViewAttribute *dl_bottomMargin;
@property (nonatomic, strong, readonly) DLSimpleViewAttribute *dl_leadingMargin;
@property (nonatomic, strong, readonly) DLSimpleViewAttribute *dl_trailingMargin;
@property (nonatomic, strong, readonly) DLSimpleViewAttribute *dl_centerXWithinMargins;
@property (nonatomic, strong, readonly) DLSimpleViewAttribute *dl_centerYWithinMargins;

#endif
@end

```

[↑目录](#table-content)

到此为止，`DLSimpleAutolayout` 基本上就实现了。具体实现的话可以看代码 [`DLSimpleAutolayout`](https://github.com/danleechina/DLSimpleAutolayout/tree/master/DLSimpleAutolayoutDemo/DLSimpleAutolayout)

***