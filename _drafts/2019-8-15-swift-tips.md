---
layout: post
title: Swift 之奇淫巧技
categories: Swift
description: Swift
keywords: ios, swift
---

在读一些 Swift 的框架源码时，会发现一些简洁特殊的 Api，本文用来记录这些代码片段，以后开发中可能会用到。


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

#### 条件编译
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
#### 使用 Raw 字符串
使用`#`来包裹的 Raw 字符串，里面的字符不会做处理，特别是一些转义字符。对于正则的特别好用:
```swift
let regex1 = "\\\\[A-Z]+[A-Za-z]+\\.[a-z]+"
let regex2 = #"\\[A-Z]+[A-Za-z]+\.[a-z]+"#
```
插值需要这样做:
```swift
let answer = 42
let dontpanic = #"The answer is \#(answer)."#
```
使用多个 # 号来消除歧义，前后 # 号数量要一致：
```swift
let a = ##"aaa"#"##
print(a) // aaa"#
```

跨越多行的字符串可以使用`"""`来包裹。
```swift
let quotation = 
"""
The White Rabbit put on his spectacles. "Where shall I begin,
please your Majesty?" he asked.

"Begin at the beginning," the King said gravely, "and go on
till you come to the end; then stop."
"""
```