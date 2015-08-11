---
layout: post
title: "为什么不要在init和dealloc函数中使用accessor"
date: 2015-08-11 15:10:10 +0800
comments: true
author: 彭笑风
categories: 
---

### 前言

唐巧曾于2011年在Blog上发过一片文章--[不要在init和dealloc函数中使用accessor](http://www.devtang.com/blog/2011/08/10/do-not-use-accessor-in-init-and-dealloc-method/)，即调用get,set方法的点语法，相信绝大多数人都这么做过（包括我）。其中提到苹果的开发者文档中[讲解cocoa内存管理的文章](http://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/MemoryMgmt/MemoryMgmt.pdf)第16页有一节的题目为：Don’t Use Accessor Methods in Initializer Methods and dealloc。文章没有说原因，唐巧在搜寻了一些资料后觉得有可能是因为在 init 和 dealloc 中，对象的存在与否还不确定，所以给对象发消息可能不会成功。但事实上并非如此。

下为《[禅与 Objective-C 编程艺术](https://github.com/oa414/objc-zen-book-cn)》的原文引用

> objc创建对象的特性为两步创建：
> 
> - `alloc` 负责创建对象，这个过程包括分配足够的内存来保存对象，写入 `isa` 指针，初始化引用计数，以及重置所有实例变量。
> - `init` 负责初始化对象，这意味着使对象处于可用状态。这通常意味着为对象的实例变量赋予合理有用的值。
> 
> `alloc` 方法将返回一个有效的未初始化的对象实例。每一个对这个实例发送的消息会被转换成一次`objc_msgSend()` 函数的调用，形参 `self` 的实参是 `alloc` 返回的指针；这样 `self` 在所有方法的作用域内都能够被访问。 按照惯例，为了完成两步创建，新创建的实例第一个被调用的方法将是 `init`方法。注意，`NSObject` 在实现 `init` 时，只是简单的返回了 `self`。

可见alloc完后就可以直接对对象发送消息了，不存在可能不成功的情况。所以接下来想和大家一起探讨下为什么不能在init和dealloc中使用accessor。

### 原因

我认为，不能在init和dealloc中使用accessor的原因是由于面向对象的继承、多态特性与accessor可能造成的副作用联合导致的。继承和多态导致在父类的实现中调用accessor可能导致调用到子类重写的accessor，而此时子类部分并未完全初始化或已经销毁，导致原有的假设不成立，从而出现一系列的逻辑问题甚至崩溃。为了更清晰地阐述，以下分别从init和dealloc上举例说明。

### Init example

BaseClass:

``` 
@interface BaseClass : NSObject

@property(nonatomic) NSString* info;

@end

@implementation BaseClass

- (instancetype)init {
if ([super init]) {
self.info = @"baseInfo";
}
return self;
}
@end
```

SubClass:

``` 
@interface SubClass : BaseClass

@end

@interface SubClass ()

@property (nonatomic) NSString* subInfo;

@end

@implementation SubClass

- (instancetype)init {
if (self = [super init]) {
self.subInfo = @"subInfo";
}
return self;
}

- (void)setInfo:(NSString *)info {
[super setInfo:info];
NSString* copyString = [NSString stringWithString:self.subInfo];
NSLog(@"%@",copyString);
}
@end
```

当执行`[[SubClass alloc]init]`时会调用父类在Init方法。其中调用了accessor，去初始化父类部分的info属性。看起来十分正常，但一旦子类重写了该方法，那么由于多态此时调用的就是子类的accessor方法！子类的accessor实现中的代码都是以子类部分已初始化完全为前提编写，即子类部分已经初始化完毕，完全可用，而现实情况是其init方法并没有执行完，对此假设并不成立，从而可能造成崩溃。以上例子有人造的痕迹，现实中更多的是某个方法被少调用一次，出现逻辑错误。

### dealloc example

BaseClass:

``` 
@interface BaseClass : NSObject

@property(nonatomic) NSString* info;

@end

@implementation BaseClass

- (void)dealloc {
self.info = nil;
}

@end
```

SubClass:

``` 
@interface SubClass : BaseClass

@property (nonatomic) NSString* debugInfo;

@end

@implementation SubClass

- (instancetype)init {
if (self = [super init]) {
_debugInfo = @"This is SubClass";
}
return self;
}

- (void)setInfo:(NSString *)info {
NSLog(@"%@",[NSString stringWithString:self.debugInfo]);
}

- (void)dealloc {
_debugInfo = nil;
}

@end
```

在SubClass的实例对象销毁时，首先调用子类的dealloc，再调用父类的dealloc。如果父类在dealloc时调用了accessor 并且该accessor被子类重写，就会调用到子类的accessor。而此时子类的dealloc已经被调用了，基于其完整的假设已经不成立，那么再执行子类的代码会存在一定风险，如上例就会崩溃。

### 结语

在init和dealloc中使用accessor是存在风险的。即使现在代码没有问题，难保将来维护或扩展时会出现问题。只有将苹果所说的Don’t Use Accessor Methods in Initializer Methods and dealloc当作一条编程规范，才能从根本上规避这个问题。规矩立好了，代码欠的债就少，将来的生活就会更加美好。

------


### 参考链接

http://www.devtang.com/blog/2011/08/10/do-not-use-accessor-in-init-and-dealloc-method/
