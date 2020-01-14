---
layout: post
title: 解耦套路之依赖注入
categories: Design Patterns
description: 解耦套路之依赖注入
keywords: keyword1, keyword2
---

 Spring 是 Java 最火的框架了，控制反转（IoC）是 Spring 框架的核心，依赖注入（DL）是它的实现方式，本文用来学习理解这种编程思想，具体的实现要根据使用的语言来选择对应的容器框架。

先看一个例子
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


现在假如要动态获取食物种类，将`name`作为初始化参数动态配置，就需要做下面这样的修改

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

我们要修改底层类`Food`，却不得不对它的上层类`People`也进行修改，在真实开发环境中，底层的类可能被数十个上层类引用到，这样的设计是难以维护的。

所以我们需要控制反转，让底层类不再控制上层类，而依赖注入是控制反转的一种实现，就是把底层类作为参数传入上层类，实现上层类对下层类的控制。

这里我们用构造方法传递的依赖注入方式重写`People`类的定义：

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

当我们修改`Food`类时，就不需要再去修改`People`了

依赖注入还有基于熟悉、set 方法、注解等实现方式，但目的都是一样的，把依赖的对象在外部生成，然后传入。

**什么是容器（Ioc Container）呢？**

其实就是这段代码了
```
let food = Food(name: "炸鸡")
let w = People(food: food)
```

在使用一些容器框架后，我们就不需要手动写这些代码，而是提供一个配置（xml文件、注解等）

当需要`People`对象时，容器框架会根据它的配置来生成对应的对象，Java 甚至可以通过注解，直接在代码上写这些配置。

不同语言会有对应的容器框架，框架内部会读取配置文件、注解等信息，在程序预编译、编译期或者运行期间读取配置来生成需要的对象。

**这样做的好处？**
* 依赖关系更直观，也方便写测试用例
* 依赖链过长时不需要记每个类的构造函数，提供配置即可

**设计思想**

其实就是添加了个中介，资源再不由使用资源的双方管理，而由不使用资源的容器管理。

这可以带来很多好处，资源集中管理，实现资源的可配置和易管理。同时也降低了使用资源双方的依赖程度，也就是我们说的耦合度。

