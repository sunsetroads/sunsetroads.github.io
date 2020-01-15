---
layout: post
title: 避免滥用单例模式
categories: DesignPatterns
description: 避免滥用单例
keywords: SingletonPattern
---

单例是一种很常见的设计模式，iOS 也提供了很多单例对象，比如 UIApplication、NSUserDefaults 等。但在我看过一些代码里，有很多本不需要使用单例的地方，从而带来一些不必要的麻烦，本文用来讲述一些单例的弊端和如何避免他们。


## 单例的弊端

我们应该都用过单例，或许你见过类似这样的代码：
```
class Singleton {
    static let shared = Singleton()
    var state: Int = 0
}

class A {
    func doSomething() {
        if Singleton.shared.state == 0 {
            // do something
        }
    }
}

class B {
    func doSomething() {
        Singleton.shared.state = 0
    }
}
```

在这个例子中，A 和 B 本该是独立的模块。但是 B 可以通过修改单例对象的属性来影响 A 的行为，两个看起来完全不相关的模块之间建立了耦合。

## 避免使用单例

全局状态让程序难以调试，既然 B 会影响到 A，那就应该明确他们的依赖关系，可以使用依赖注入的方式进行改造:
```
struct Config {
    var state: Int = 0
}

class A {
    var config:Config
    init(config:Config) {
        self.config = config
    }
    func doSomething() {
        if config.state == 1 {
            // do something
        }
    }
}

class B {
    var config:Config
    init(config:Config) {
        self.config = config
    }
    func doSomething() {
        config.state = 1
    }
}

let config = Config()
let b = B(config: config)
b.doSomething()

...

let a = A(config: b.config)
a.doSomething()
```
这样看似增加了代码量，但明确的依赖关系会让程序运行过程变得清晰，也方便测试。

**在面向对象编程中我们本应该最小化可变状态的作用域，但是单例却让可变的状态可以在程序中的任何地方访问，从而站在了对立面。**

## 使用单例时先思考

有时候我们不得不使用单例，但使用前应该先思考这些问题：

* 这个类表达的含义真的只能有一个实例么？（如UIApplication）还是只是为了好调用而已？
* 这个单例持有的内存是否需要一直存在？
* 是否能用类方法代替？
* 这个单例对象是否能成为另一个单例对象的属性？如果是，应该作为属性