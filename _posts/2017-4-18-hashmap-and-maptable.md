---
layout: post
title: NSHashtable 和 NSMaptable
categories: iOS
description: NSHashtable 和 NSMaptable 解析
keywords: ios 
---

NSSet，NSDictionary，NSArray 是 Foundation 框架关于集合操作的常用类，它们默认假定了其中对象的内存行为。对于 NSSet 来说，object 是强引用的，和 NSDictionary 中的 value 是一样的。而 NSDictionary 中的 key 则是 copy 的。

当开发者想要使 NSSet 的 objects 或者 NSDictionary 的 values 为 weak，或者 NSDictionary 使用没有实现协议的对象作为 key 时，就会比较麻烦（需要使用 NSValue 的方法 valueWithNonretainedObject）。还好苹果为我们提供了相对于 NSSet 和 NSDictionary 更通用的两个类 NSHashTable 和 NSMapTable。

## NSHashTable
NSHashTable 是更广泛意义的 NSSet，区别于 NSSet/NSMutableSet，NSHashTable 有如下特性：

* NSHashTable 是可变的；
* NSHashTable 可以持有 weak 类型的成员变量；
* NSHashTable 可以在添加成员变量的时候复制成员；
* NSHashTable 可以随意的存储指针并且利用指针的唯一性来进行 hash 同一性检查（检查成员变量是否有重复）和对比操作（equal）；

用法如下：
```objc
NSHashTable *hashTable = [NSHashTable hashTableWithOptions:NSPointerFunctionsCopyIn];
[hashTable addObject:@"foo"];
[hashTable addObject:@"bar"]; 
[hashTable addObject:@42];
[hashTable removeObject:@"bar"];
NSLog (@"Members: %@", [hashTable allObjects]);
```
NSHashTable 是根据一个 option 参数来进行初始化的，因为从 OSX 平台上移植到 iOS 平台上，原来 OSX 平台上使用的枚举类型被放弃了，从而用 option 来代替，命名也发生了一些变化：

* NSHashTableStrongMemory

等同于 NSPointerFunctionsStrongMemory。对成员变量进行强引用，这是一个默认值，如果采用这个默认值，NSHashTable 和 NSSet 就没什么区别了。

* NSHashTableWeakMemory

等同于 NSPointerFunctionsWeakMemory。对成员变量进行弱引用，使用 NSPointerFunctionsWeakMemory，object 引用在最后释放的时候会被指向 NULL。

* NSHashTableZeroingWeakMemory

已被抛弃，使用 NSHashTableWeakMemory 代替。

* NSHashTableCopyIn

在对象被加入集合之前进行复制（NSPointerFunction-acquireFunction），等同于 NSPointerFunctionsCopyIn。

* NSHashTableObjectPointerPersonality

用指针来等同代替实际的值，当打印这个指针的时候相当于调用 description 方法。和 NSPointerFunctionsObjectPointerPersonality 等同。

## NSMapTable

NSMapTable 是对更广泛意义的 NSDictionary。和 NSDictionary/NSMutableDictionary 相比具有如下特性：

* NSDictionary/NSMutableDictionary 会复制 keys 并且通过强引用 values 来实现存储；
* NSMapTable 是可变的；
* NSMapTable 可以通过弱引用来持有 keys 和 values，所以当 key 或者 value 被 deallocated 的时候，所存储的实体也会被移除；
* NSMapTable 可以在添加 value 的时候对 value 进行复制；

和 NSHashTable 类似，NSMapTable 可以随意的存储指针，并且利用指针的唯一性来进行对比和重复检查。

用法：假设用 NSMapTable 来存储不用被复制的 keys 和被弱引用的 value，这里的 value 就是某个 delegate 或者一种弱类型。

```objc
id delegate = ...;
NSMapTable *mapTable = [NSMapTable mapTableWithKeyOptions:NSMapTableStrongMemory
                                             valueOptions:NSMapTableWeakMemory];
[mapTable setObject:delegate forKey:@"foo"];
NSLog (@"Keys: %@", [[mapTable keyEnumerator] allObjects]);
```


通常，编程不是为了让人更聪明，而是最大化抽象一个问题的能力。NSSet 和 NSDictionary 是非常伟大的类，他们能解决 99% 的问题，也无疑是用来工作的正确工具。如果，然而你的问题牵扯到上述的内存问题时候，NSHashTable 和 NSMapTable 是值得一看的。