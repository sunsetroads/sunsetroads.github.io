---
layout: post
title: iOS 中 Load 和 Initialize 的区别
categories: iOS
description: load 和 initialize 的区别
keywords: OC, iOS
---

load 和 initialize 是 Objective-C runtime 会自动调用的两个类方法，但是它们被调用的时机却容易混淆，通过测试来总结一下各种情况。

### load 调用时机
- load 方法是在 main() 执行前加载类时调用的，仅会自动调用一次。

- 类、父类、类别中同时存在 load 时的调用顺序：父类 -> 类 -> 类别。

- 这里发现类别中的 load 没有覆盖掉类的 load 方法，说明没有走 objc_msgSend 那套查找方法的流程，应该是直接用函数指针执行了这个方法，也是算是对 iOS 启动的一种优化吧。

### initialize 调用时机
- initialize 方法是在类或它的子类收到第一条消息之前被调用的，类似于懒加载的方式。

- 如果子类没有实现 initialize，而父类实现了。那么父类的 initialize 将会调用多次。

- 如果程序一直没有给某个类或它的子类发送消息，那么这个类的 initialize 方法是永远不会被调用的。

- 如果类和它的类别都实现 initialize，那么只会调用类别的，如果父类和子类同时实现 initialize，会先调用父类的。说明这里查找执行 initialize 还是使用的 objc_msgSend。

### load 的实际应用实例
#### 1. 完成模块的初始化，瘦身 AppDelegate

很多写在 - application:didFinishLaunchingWithOptions: 中的代码都只是为了在程序启动时获得一次调用机会，多为某些模块的初始化工作，如：

```objc
- (BOOL)application:(UIApplication *)application
didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // ...
    [FooModule setup];
    // ...
    return YES;
}
```

利用 Notification 的方式在自己的模块内部搞定

```objc
/// FooModule.m
+ (void)load
{
    __block id observer =
    [[NSNotificationCenter defaultCenter]
     addObserverForName:UIApplicationDidFinishLaunchingNotification
     object:nil
     queue:nil
     usingBlock:^(NSNotification *note) {
         [self setup]; // Do whatever you want
         [[NSNotificationCenter defaultCenter] removeObserver:observer];
     }];
}
```
#### 2. 完成类的的 Hook,Patch 等等
结合 Method Swizzling 替换 viewWillAppear 方法

```objc
@interface UIViewController (MethodModified)

@end

@implementation UIViewController (MethodModified)

+ (void)load {
    static dispatch_once_t onceToken;
    //防止手动调用 load 的骚操作
    dispatch_once(&onceToken, ^{
        Class class = [self class];

        SEL originalSelector = @selector(viewWillAppear:);
        SEL swizzledSelector = @selector(zn_viewWillAppear:);

        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);

        BOOL success = class_addMethod(class, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod));
        if (success) {
            class_replaceMethod(class, swizzledSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

#pragma mark - Method Swizzling

- (void)zn_viewWillAppear:(BOOL)animated {
    [self mrc_viewWillAppear:animated];
    // ...
}

@end

```


 