---
layout: post
title: Swift 中值类型的性能优化
categories: iOS
description: Swift 中的 Copy-On-Write
keywords: Copy On Write
---

Swift 中的 Array Dictionary 类型都是作为值类型传递的，但值类型的传递可能会带来性能问题，这里模仿一下 Copy-On-Write 机制来进行优化。

举个例子，想象一下有1000个元素的数组。如果将该数组复制到另一个变量中，则即使两个数组最终相同，Swift也必须复制所有1000个元素。

### Copy-On-Write 机制

Copy-On-Write 解决了此问题，参照下面的例子，当您将两个变量指向同一数组时，它们都指向相同的基础数据。当您对第二个变量进行修改时，Swift 会在此时进行完整复制，以便仅修改第二个变量而第一个不变。

```
func address(of object: UnsafeRawPointer) -> String {
    let addr = Int(bitPattern: object)
    return String(format: "%p", addr)
}

var array1 = [1, 2, 3, 4]

address(of: array1)     // 0x60000200cd60

var array2 = array1

address(of: array2)     // 0x60000200cd60

array1.append(2)

address(of: array1)     // 0x60000112cec0
address(of: array2)     // 0x60000200cd60

```


需要注意的是，Copy-On-Write 是专门添加到 Swift 数组和字典的功能，并非所有值类型都具有这种行为。这意味着，如果我们拥有一个包含大量信息的结构体类型，则可能需要 Copy-On-Write 来避免无用的复制，提高应用程序的性能。因此，我们必须手动创建此行为。

### 手动实现 Copy-On-Write

先创建一个用于 Copy-On-Write 的结构体
```
struct User {
    var identifier = 1
}

```

`class` 是一种引用类型，当我们将引用类型分配给另一个引用类型时，这两个变量将共享同一实例，而不是像值类型一样复制它。这里创建一个 Ref 类，用于包装我们的值类型

```
final class Ref<T> {
    var value: T
    init(value: T) {
        self.value = value
    }
}

```

然后，我们再创建一个struct包装Ref：
```
struct Box<T> {
    private var ref: Ref<T>
    init(value: T) {
        ref = Ref(value: value)
    }

    var value: T {
        get { return ref.value }
        set {
            guard isKnownUniquelyReferenced(&ref) else {
                ref = Ref(value: newValue)
                return
            }
            ref.value = newValue
        }
    }
}

```
由于struct是值类型，因此当我们将其分配给另一个变量时，将复制它的值，而属性实例 ref 仍然是两个副本共享的，因为它是引用类型。

然后，当我们第一次更改两个`Box`变量中的一个时，我们创建一个`ref`属性的新实例：

```
guard isKnownUniquelyReferenced(&ref) else {
    ref = Ref(value: newValue)
    return
}
```

这样，两个`Box`变量不再共享相同的`ref`实例。

`isKnownUniquelyReferenced`返回一个布尔值，该布尔值表示给定对象是否具有单个强引用。

这里是整个代码：
```
final class Ref<T> {
    var value: T
    init(value: T) {
        self.value = value
    }
}

struct Box<T> {
    private var ref: Ref<T>
    init(value: T) {
        ref = Ref(value: value)
    }

    var value: T {
        get { return ref.value }
        set {
            guard isKnownUniquelyReferenced(&ref) else {
                ref = Ref(value: newValue)
                return
            }
            ref.value = newValue
        }
    }
}
```
我们可以像这样使用这种包装：
```
let user = User()

let box = Box(value: user)
var box2 = box                  //box2 共享 box1.ref

box2.value.identifier = 2       //创建一个新的对象给 box2.ref
```

可以在这里查看更多更详细的 [Swift Optimization Tips](https://github.com/apple/swift/blob/master/docs/OptimizationTips.rst).


