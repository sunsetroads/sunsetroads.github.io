---
layout: post
title: OC 消息转发中的 NSMethodSignature 和 NSInvocation 解析
categories: iOS
description: NSMethodSignature 和 NSInvocation 解析
keywords: ios 
---

OC 向一个对象发送消息后，如果找不到对应的实现时就会进入消息转发。其中涉及到了 `NSMethodSignature` 和 `NSInvocation` 这两个不熟悉的类，这里总结一下它们的作用。

## 消息转发过程

 1. 调用 `resolveInstanceMethod:` 方法 (或 `resolveClassMethod:`)。允许用户在此时为该 Class 动态添加实现。如果有实现了，则调用并返回 YES，那么重新开始 `objc_msgSend` 流程。这一次对象会响应这个选择器，一般是因为它已经调用过 `class_addMethod`。如果仍没实现，继续下面的动作。

 2. 调用 `forwardingTargetForSelector:` 方法，尝试找到一个能响应该消息的对象。如果获取到，则直接把消息转发给它，返回非 nil 对象。否则返回 nil ，继续下面的动作。

 3. 调用 `methodSignatureForSelector:` 方法，尝试获得一个方法签名。如果获取不到，则直接调用 `doesNotRecognizeSelector` 抛出异常。如果能获取，则返回非 nil：创建一个 NSlnvocation 并传给 `forwardInvocation:`。

 4. 调用 `forwardInvocation:` 方法，将第 3 步获取到的方法签名包装成 Invocation 传入，如何处理就在这里面了，并返回非 nil。

 5. 调用 `doesNotRecognizeSelector:` ，默认的实现是抛出异常。如果第 3 步没能获得一个方法签名，执行该步骤。

## NSMethodSignature
官方文档的描述：
> A record of the type information for the return value and parameters of a method.

OC 的方法签名只是记录了方法的返回值和参数的类型，不包含方法名称，就像是一个方法格式的规定模板。开发者可以 override 消息转发过程前 4 个方法，由 runtime 来调用。最常见的实现消息转发就是重写方法 3 和 4。

生成一个方法签名：
```objc
NSMethodSignature *signature = [NSMethodSignature signatureWithObjCTypes:"@@:*"];
```
方法签名的 "@@:" 字符串怎么理解呢？ 第一个字符 @ 表明返回值是一个 id。对于消息传递系统来说，所有的 OC 对象都是 id 类型。 接下来的二个字符 @： 表明该方法接受一个 id 和一个 SEL 。其实每个 Objective-C 方法都把 id 和 SEL 作为头 2 个参数。最后一个字符 * 表示该方法的一个显式的参数是一个字符串（char *）。

那如何获取这些类型编码呢，可以参考官方文档 [Type Encodings](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html) ，也可以直接使用类型编码 @encode (type) 获取表示该类型的字符串，而不必硬编码。

## NSInvocation
官方文档的描述：
> An Objective-C message rendered as an object.

NSInvocation 用来包装方法和对应的对象，它可以存储方法的名称，调用对象和参数。

NSInvocation 对象需要使用一个方法签名 NSMethodSignature 来初始化。NSMethodSignature 只是表示了方法的返回值和参数的类型。所以在创建 NSInvocation 对象之后仍需指定消息的接收对象和 Selector。

原则上接收对象的 Selector 需要跟 NSMethodSignature 相匹配。但是根据实践来说，只要不造成 NSInvocation setArgument:atIndex 越界的异常，都是可以成功转发消息的，并且转发成功之后，未赋值的参数都将被赋值为 nil。


举个 🌰：
```objc
- (void) greetingWithInvocation {
    NSMethodSignature *methodSignature = [self methodSignatureForSelector:@selector (greetingWithName:)];
    
    NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:methodSignature];
    [invocation setSelector:@selector (greetingWithAge:name:)];
    
//    NSString *name = @"Tom";
//    [invocation setArgument:&name atIndex:3];
    NSUInteger age = 10;
    [invocation setArgument:&age atIndex:2];
    
    [invocation invokeWithTarget:self];
}

- (void) greetingWithName:(NSString *) name {
    NSLog (@"Hello World %@!",name);
}

- (void) greetingWithAge:(NSUInteger) age name:(NSString *) name {
    NSLog (@"Hello %@ % ld!", name, (long) age);
}
```
执行结果：
```
2017-05-03 16:16:29.815 NSInvocationDemo [50214:49610519] Hello (null) 10!
```

另外需要注意的是，`setArgument:atIndex:` 默认不会强引用它的 argument，如果 argument 在 NSInvocation 执行的时候之前被释放就会造成野指针异常（EXC_BAD_ACCESS）。调用 retainArguments 方法来强引用参数（包括 target 以及 selector）。