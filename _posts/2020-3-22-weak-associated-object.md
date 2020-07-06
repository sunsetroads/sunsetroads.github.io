---
layout: post
title:  Weak Associated Object
categories: iOS
description: OC 实现一个 Weak Associated Object
keywords: oc, weak
--- 

OC 类和对象，在 runtime 层都是用 struct 表示的，category 也不例外，由于 category 的结构体中没有 ivar_list_t 这一字段，所以无法直接在一个类别中添加成员变量。如果需要，通常是借助 runtime 来关联对象，但关联对象的策略里没有提供对弱引用对象的支持，本文用来探讨如何关联一个弱引用对象。

## Associated Object 介绍

先看下类别在 runtime 中的结构体定义

```c++
typedef struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
} category_t;
```

由于没有`ivar_list_t`，在类别中添加成员变量通常需要借助 runtime 的函数`objc_getAssociatedObject`和`objc_setAssociatedObject`：
```objc
@interface NSObject (Weak)

@property(nonatomic)NSString *myObject;;

@end

@implementation NSObject (Weak)

-(NSString *) myObject {
    return objc_getAssociatedObject (self, @selector (myObject));
}

-(void) setMyObject:(NSString *) myObject {
    objc_setAssociatedObject (self, @selector (myObject), myObject, OBJC_ASSOCIATION_ASSIGN);
}
@end
```

`objc_setAssociatedObject`设置关联对象时需要指定关联策略 objc_AssociationPolicy：
```objc
typedef OBJC_ENUM (uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0,         
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1,
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,
    OBJC_ASSOCIATION_RETAIN = 01401,    
    OBJC_ASSOCIATION_COPY = 01403
};
```
可以看到这里有一个`OBJC_ASSOCIATION_ASSIGN`，但并没有 `OBJC_ASSOCIATION_WEAK`，所以无法直接持有一个 weak 对象，下面会讲如何通过添加中间层来间接持有一个 weak 对象。


### 关联对象实现原理
与普通的属性或实例变量不同，Objective-C 在底层使用 AssociationsManager 统一管理各个对象的 associated objects，里面是一个嵌套的字典，数据结构大概是这样：
```
{
    被关联的对象 ：{
        关联对象时指定的 key ： 关联的对象
    }
}
```

在对象 dealloc 时经过如下调用栈解除对这些关联对象的引用：
```
dealloc
    object_dispose
        objc_destructInstance
            _object_remove_assocations  // 移除必要的 associated objects
```
### Weak 和 Assign 的区别
weak 变量相对于 assign 变量的最大魅力在于，若指向的对象被销毁了，前者会被 Objective-C 设置为 nil；后者却没有这样的待遇，利用这个特性，可以避免访问到悬垂指针。这个逻辑是如何实现的呢？简单来说，Runtime 在底层维护一个 weak 表，每每分配一个 weak 指针并赋值有效对象的地址时，会将对象地址和 weak 指针地址注册到 weak 表中，其中对象地址作为 key；当对象被废弃时，可根据对象地址快速寻找到指向它的所有 weak 指针，这些 weak 指针会被赋值 0（即 nil）并移出 weak 表。

## 定义 Weak Associated Object
有时候，为了安全起见，我们需要在对象和 associated object 之间建立 weak 关系，而非 assign 关系，如何实现呢？其实也简单，计算机领域有句名言： 

>“计算机科学领域的任何问题都可以通过增加一个间接的中间层来解决”


### 最佳方案

这是一种简洁优雅的实现方式，__weak 本身就会把指针指向 nil，那直接利用就是了。使用 OBJC_ASSOCIATION_COPY 关联策略将 block copy 到堆上，利用 block 把持有的 weak 对象返回，如果对象不存在了，返回的便是空值。

```objc
@implementation NSObject (Weak)

-(NSString *) myObject {
    id (^block)(void) = objc_getAssociatedObject (self, @selector (myObject));
    return (block ? block () : nil);
}

-(void) setMyObject:(NSString *) myObject {
    id __weak weakObject = myObject;
    id (^block)(void) = ^{ return weakObject; };
    objc_setAssociatedObject (self, @selector (myObject), block, OBJC_ASSOCIATION_COPY);
}
@end
```
### 包装类
这种方式是通过包装一个对象实现的，先定义一个这样的类，拥有一个 weak 修饰的属性：

```objc
@interface WeakAssociatedObjectWrapper : NSObject

@property (nonatomic, weak) id object;

@end

@implementation WeakAssociatedObjectWrapper

@end
```

需要定义 weak associated object 时，使用该类对象作为中转：
```objc
@implementation NSObject (Weak)

-(NSString *) myObject {
    WeakAssociatedObjectWrapper *wrapper = objc_getAssociatedObject (self,  @selector (myObject));
    return wrapper.object;
}

-(void) setMyObject:(NSString *) myObject {
    WeakAssociatedObjectWrapper *wrapper = [WeakAssociatedObjectWrapper new];
    wrapper.object = myObject;
    objc_setAssociatedObject (self, @selector (myObject), wrapper, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
@end
```
这种方式稍微麻烦了一点。其实也可以借助一些可以存储弱引用对象的容器来做中间层，比如 NSHashtable 和 NSMaptable。

NSHashtable 和 NSMaptable 的使用可以参考：[https://sunsetroads.github.io/2017/04/18/hashmap-and-maptable/](https://sunsetroads.github.io/2017/04/18/hashmap-and-maptable/)