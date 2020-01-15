---
layout: post
title: 解耦套路之避免滥用单例
categories: DesignPatterns
description: 解耦套路之避免滥用单例
keywords: SingletonPattern
---

单例是整个 Cocoa 中被广泛使用的核心设计模式之一。事实上，苹果开发者库把单例作为 "Cocoa 核心竞争力" 之一。作为一个iOS开发者，我们经常和单例打交道，比如 UIApplication 和 NSFileManager 等等。我们在开源项目、苹果示例代码和 StackOverflow 中见过了无数使用单例的例子。