---
layout: post
title: iOS 中 Load 和 Initialize 的区别
categories: iOS
description: load 和 initialize 的区别
keywords: OC, iOS
---

+load 和 +initialize 是 Objective-C runtime 会自动调用的两个类方法，但是它们被调用的时机却是有差别的。

+load 方法是在类被加载的时候调用的，而 +initialize 方法是在类或它的子类收到第一条消息之前被调用的，这里所指的消息包括实例方法和类方法的调用。也就是说 +initialize 方法是以懒加载的方式被调用的，如果程序一直没有给某个类或它的子类发送消息，那么这个类的 +initialize 方法是永远不会被调用的

```
@implementation Person
// load方法会在当前类被加载到内存的时候调用, 有且仅会调用一次
+ (void)load
{
    NSLog(@"Person类被加载到内存了");
}
 
// initialize用于对某一个类进行一次性的初始化
+ (void)initialize
{
    NSLog(@"Person initialize");
}

@end
```

### 总结一下：

|  | +load | +initialize |
| ------ | ------ | ------ |
| 执行的时机 | 在程序运行后立即执行 | 在类的方法第一次被调时执行 |
| 类别中定义 | 全都执行，但后于类中的方法 | 覆盖类中的方法，只执行一个|
| 存在继承关系 | 先执行父类的load方法，再调用子类的initialize方法 | 先执行父类的initialize方法，再调用子类的load方法 |
| 若自身未定义，是否沿用父类的方法？ | 否 | 是(所以该方法有可能被调用多次)|


### load的实际应用实例
#### 1.完成模块的初始化，瘦身AppDelegate

> 很多写在- application:didFinishLaunchingWithOptions:中的代码都只是为了在程序启动时获得一次调用机会，多为某些模块的初始化工作，如：

```
- (BOOL)application:(UIApplication *)application
didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // ...
    [FooModule setup];
    // ...
    return YES;
}
```

> 利用Notification的方式在自己的模块内部搞定

```
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
#### 2.完成类的的Hook,Patch等等
> 结合Method Swizzling替换viewWillAppear方法

```
@interface UIViewController (MethodModified)

@end

@implementation UIViewController (MethodModified)

+ (void)load {
    static dispatch_once_t onceToken;
    //防止程序员对 +load 方法的手动调用
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


 