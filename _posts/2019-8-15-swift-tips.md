---
layout: post
title: Swift 之奇淫巧技
categories: Swift
description: Swift
keywords: ios, swift
---

在读一些 Swift 的框架源码时，会发现作者一些简洁特殊的写法，本文用来记录这些将来可能用到的代码片段。


#### 判断是否可以导入某个库
```swift
#if canImport(SpriteKit)
   // this will be true for iOS, macOS, tvOS, and watchOS
#else
   // this will be true for other platforms, such as Linux
#endif
```

#### 判断是否为模拟器
```swift
#if targetEnvironment(simulator)
   // code for the simulator here
#else
   // code for real devices here
#endif
```

#### 判断运行的系统
```swift
#if !os(Linux)
   // Matches macOS, iOS, watchOS, tvOS, and any other future platforms
#endif

#if os(macOS) || os(iOS) || os(tvOS) || os(watchOS)
   // Matches only Apple platforms, but needs to be kept up to date as new platforms are added
#endif
```

#### 让 Xcode 在编译时候作出提示
```swift
func encrypt(_ string: String, with password: String) -> String {
    #warning("This is terrible method of encryption")
    return password + String(string.reversed()) + password
}

struct Configuration {
    var apiKey: String {
        #error("Please enter your API key below then delete this line.")
        return "Enter your key here"
    }
}

#if os(macOS)
#error("MyLibrary is not supported on macOS.")
#endif
```