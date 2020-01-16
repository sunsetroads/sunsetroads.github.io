---
layout: post
title: 谨慎使用单例模式
categories: DesignPatterns
description: 谨慎使用单例模式
keywords: SingletonPattern
---

单例是一种很常见的设计模式，iOS 也提供了很多单例对象，比如 UIApplication、NSUserDefaults 等。但在我看过一些代码里，有很多本不需要使用单例的地方，从而带来一些不必要的麻烦，本文用来讲述一些单例的弊端和如何避免他们。

**单例模式**

> 保证一个类仅有一个实例，并提供一个访问它的全局访问点

## 单例带来的一些问题

### 单例让模块间潜在耦合

我们应该都用过单例，或许你见过类似这样的代码：

```
class Singleton {
    static let shared = Singleton()
    var state: Int = 0
    private init() {
    }
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

在这个例子中，A 和 B 本该是独立的模块。但是 B 可以通过修改单例对象的属性来影响 A 的行为，这让两个看起来完全不相关的模块之间建立了耦合，如果对 A 和 B 模块做单元测试，那么执行测试的顺序会影响到测试结果，这是不可控的。

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
```

这样看似增加了代码量，但明确的依赖关系会让程序运行过程变得清晰，也方便测试。但也有新的问题出现，当存在多级依赖`A --> B --> ... --> S`时，如果 A 和 S 需要用到 Config 对象，但中间的类却不需要，这时通过传递共享对象会显得臃肿。

如果这个 Config 对象是随着全局存在的，那将其作为单例对象也是无奈的选择。但如果本不需要全局存在却使用了单例，那就不只是存在潜在耦合了，更可能出现一些意想不到的错误。

### 非全局对象使用单例

在某些情况下，一个对象可能需要在整个进程中存在较长时间，但并非需要全局存在。

例如，一个多账号系统的 App，登陆后有一个 User 对象保存了用户的头像、昵称等信息，在多个页面可能需要用到这个 User 对象，这时把 User 对象做成单例可能就是个错误的做法，因为系统存在注销功能，登陆后应该是一个新的对象了。

把这些存在时期较长的对象做成单例的属性或许可以解决这个问题：

```
class Singleton {
    static let shared = Singleton()
    var currentUser:User?
    private init() {
    }
}

class User {

}
```
当用户注销后可以将`Singleton.shared.currentUser`置为`nil`，登陆后设置为新的`User`。

如何使用这个 User 对象呢？请不要像下面这样：
```
class Homepage {
    func doSomething() {
        DispatchQueue.global().async {
            // user Singleton.shared.currentUser
        }
    }
}
```
这样使用可能会有问题，如果一个页面在后台异步执行的任务用到了`Singleton.shared.currentUser`，而恰好该用户注销而导致`Singleton.shared.currentUser = nil`，这可能会引起一些错误。

使用依赖注入解决这个问题：
```
class Homepage {
    var user:User
    init(currentUser:User) {
        self.user = currentUser
        // show currentUser ui
    }
    
    func doSomething() {
        // 后台线程使用到 user 对象的一些属性
    }
}
```
这样即使`Singleton.shared.currentUser = nil`，但`Homepage`仍有一个对`User`对象的强引用，`User`对象是没有释放的，也不会影响到后台任务的执行。

依赖注入可以参考 [https://sunsetroads.github.io/2020/01/14/decoupling/](https://sunsetroads.github.io/2020/01/14/decoupling/)

## 不要轻易使用单例

如何处理全局共享的状态是一个难题，在面向对象编程中我们本应该最小化可变状态的作用域，但是单例却让可变的状态可以在程序中的任何地方访问，从而站在了对立面。

单例让状态变得混乱，程序难以测试，开发过程中应尽量避免使用。如果设计初期决定使用，那么请先思考这些问题：

1. 这个类表达的含义真的只能有一个实例么？还是只是为了好调用而已？
2. 这个单例持有的内存是否需要一直存在？
3. 是否能用类方法代替？
4. 这个单例对象是否能成为另一个单例对象的属性？如果是，应该作为属性

参考：[https://www.objc.io/issues/13-architecture/singletons/](https://www.objc.io/issues/13-architecture/singletons/)
