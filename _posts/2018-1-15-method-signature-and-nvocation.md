---
layout: post
title: OC 消息转发中的 NSMethodSignature 和 NSInvocation 解析
categories: iOS
description: NSMethodSignature 和 NSInvocation 解析
keywords: ios 
---

OC 向一个对象发送消息后，如果找不到对应的实现时就会进入消息转发。其中涉及到了 `NSMethodSignature` 和 `NSInvocation` 这两个不熟悉的类，这里总结一下它们的作用。

## 消息转发过程

 1. 调用`resolveInstanceMethod:`方法 (或 `resolveClassMethod:`)。允许用户在此时为该 Class 动态添加实现。如果有实现了，则调用并返回YES，那么重新开始`objc_msgSend`流程。这一次对象会响应这个选择器，一般是因为它已经调用过`class_addMethod`。如果仍没实现，继续下面的动作。

 2. 调用`forwardingTargetForSelector:`方法，尝试找到一个能响应该消息的对象。如果获取到，则直接把消息转发给它，返回非 nil 对象。否则返回 nil ，继续下面的动作。

 3. 调用`methodSignatureForSelector:`方法，尝试获得一个方法签名。如果获取不到，则直接调用`doesNotRecognizeSelector`抛出异常。如果能获取，则返回非 nil：创建一个 NSlnvocation 并传给`forwardInvocation:`。

 4. 调用`forwardInvocation:`方法，将第3步获取到的方法签名包装成 Invocation 传入，如何处理就在这里面了，并返回非nil。

 5. 调用`doesNotRecognizeSelector:` ，默认的实现是抛出异常。如果第3步没能获得一个方法签名，执行该步骤。

## NSMethodSignature
先看下官方文档的描述：
> A record of the type information for the return value and parameters of a method.

OC 的方法签名只是记录了方法的返回值和参数的类型，不包含方法名称，就像是一个方法格式的规定模板。开发者可以 override 消息转发过程前 4 个方法，由 runtime 来调用。最常见的实现消息转发：就是重写方法3和4。可以这样生成一个方法签名：
```
NSMethodSignature *signature = [NSMethodSignature signatureWithObjCTypes:"@@:*"];
```
方法签名的 "@@:" 字符串怎么理解呢？ 第一个字符 @ 表明返回值是一个 id。对于消息传递系统来说，所有的 OC 对象都是 id 类型。 接下来的二个字符 @： 表明该方法接受一个 id 和一个 SEL 。其实每个 Objective-C 方法都把 id 和 SEL 作为头2个参数。最后一个字符 * 表示该方法的一个显式的参数是一个字符串（char *）。

那如何获取这些类型编码呢，可以参考官方文档 [Type Encodings](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html) ，也可以直接使用类型编码 @encode(type) 获取表示该类型的字符串，而不必硬编码。

## NSInvocation

