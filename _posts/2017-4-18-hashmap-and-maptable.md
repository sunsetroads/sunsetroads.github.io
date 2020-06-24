---
layout: post
title: NSHashtable 和 NSMaptable
categories: iOS
description: NSHashtable 和 NSMaptable 解析
keywords: ios 
---

NSSet，NSDictionary，NSArray 是 Foundation 框架关于集合操作的常用类，和其他标准的集合操作库不同，他们的实现方法对开发者进行隐藏，只允许开发者写一些简单的代码，让他们相信这些代码有理由正常的工作，然而这样的话最好的代码抽象风格就会被打破，苹果的本意也被曲解了。在这种情况下，开发者寻求更好的抽象方式来使用集合，或者说寻找一种更通用的方式。