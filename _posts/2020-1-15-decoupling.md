---
layout: post
title: 解耦套路之依赖注入
categories: DesignPatterns
description: 解耦套路之依赖注入
keywords: keyword1, keyword2
---

 Spring 是 Java 最火的框架了，控制反转（IoC）是 Spring 框架的核心，依赖注入（DI）是它的实现方式，本文用来学习理解这种编程思想，具体的实现要根据使用的语言来选择对应的容器框架。

### 举个例子

`People`类的`eat()`函数依赖于`Food`类，可能会这样实现：
```
class People {
    init() {
    }
    
    func eat() {
        let food = Food()
        print("I am eating \(food.name) ")
    }
}

class Food {
    var name:String
    init() {
        name = "炸鸡"
    }
}

let w = People()
w.eat()
```


但现在假如要动态获取食物种类，将`name`作为初始化参数动态配置，就需要做下面的修改：

```
class People {
    var food:Food
    init(foodName:String) {
        food = Food(name: foodName)
    }
    
    func eat() {
        print("I am eating \(food.name)")
    }
}

class Food {
    var name:String
    init(name:String) {
        self.name = name
    }
}

let w = People(foodName: "炸鸡")
w.eat()
```

问题出现了，我们要修改底层类`Food`，却不得不对它的上层类`People`也进行修改，在真实开发环境中，底层的类可能被数十个上层类引用到，这样的设计是难以维护的。

所以我们需要控制反转，让底层类不再控制上层类，而依赖注入是控制反转的一种实现，就是把底层类作为参数传入上层类，实现上层类对下层类的控制。

### 依赖注入
这里我们用构造方法传递的依赖注入方式重写`People`类：

```
class People {
    var food:Food
    init(food:Food) {
        self.food = food
    }
    
    func eat() {
        print("I am eating \(food.name)")
    }
}

class Food {
    var name:String
    init(name:String) {
        self.name = name
    }
}

let food = Food(name: "炸鸡")
let w = People(food: food)
w.eat()
```

当我们修改`Food`类时，就不需要再去修改它的上层类`People`了

依赖注入还有基于属性、set 方法、注解等实现方式，但目的都是一样的，把依赖的对象在外部生成，然后传入。

### 容器（IoC Container）

依赖注入带来了新的问题，我们在初始化一个对象时，同时要去初始化它依赖的类，依赖链长时需要写很多初始化代码，IoC 容器解决了这个问题。

其实这段代码就可以看作一个容器
```
let food = Food(name: "炸鸡")
let w = People(food: food)
```

不同的是在使用了容器框架后，我们就不需要手动写这些代码了，而是提供一个配置，比如 XML 文件、注解等，当需要`People`对象时，容器框架会根据它的配置来生成对应的对象，Java 甚至可以通过注解，直接在代码上写这些配置。

### 容器的实现
不同语言会有对应的容器框架，框架内部会读取配置文件、注解等信息，在程序预编译、编译期或者运行期间读取配置来生成需要的对象，有些会直接插入代码，有些会在运行时利用反射技术来生成。

iOS 中可以参考的容器框架:
* [GitHub - appsquickly/Typhoon: Powerful dependency injection for iOS & OSX (Objective-C & Swift)](https://github.com/appsquickly/Typhoon)

* [GitHub - atomicobject/objection: A lightweight dependency injection framework for Objective-C](https://github.com/atomicobject/objection)


### 依赖注入的好处
* 声明式的依赖让依赖关系更直观，也方便写测试用例
* 下层类的一些修改不再影响上层类
* 依赖链过长时不需要记每个类的构造函数，提供配置给容器框架即可

### 依赖注入的设计思想

其实就是加了个中介，资源再不由使用资源的双方管理，而由不使用资源的容器管理。

这可以带来很多好处，资源集中管理，实现资源的可配置和易管理。同时也降低了使用资源双方的依赖程度，也就是我们说的耦合度。

### 和工厂模式的比较

工厂模式只是将依赖从具体的类转化到工厂类，并且新增一个实现时，通常需要去工厂类中添加一个分支，IoC 容器则不需要这一步，它使用固定的框架代码读取配置动态生成对象，进一步实现了解耦。

### 不要和依赖倒转搞混了

> 依赖倒转原则

> A.高层模块不应该依赖于底层模块，二者都应该依赖于抽象。

> B.抽象不应该依赖于细节，细节应该依赖于抽象。

很多介绍控制反转的，都喜欢说控制反转是依赖倒转的实现，其实不然，依赖倒转的本质是依赖抽象，而依赖注入的本质是依赖容器。

依赖抽象通常是利用面向对象的继承、多态来实现，但即使没有继承、多态的特性，依然可以实现依赖注入。


