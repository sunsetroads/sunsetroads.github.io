---
layout: post
title: Untiy 导出的 Xcode 替换资源流程
categories: Untiy
description: Untiy 导出的 Xcode 替换资源流程
keywords: Untiy, Xcode
---

Untiy 工程导出 Xcode 后通常需要修改一些 iOS 相关的配置，如果每次导出 Xcode 都去修改会比较浪费时间，好的做法是将整个的流程写成脚本，使用脚本自动化操作。这里介绍一种简单快捷的做法。

先看下 Untiy 导出的 Xcode 工程的目录结构

![](/images/tree.png)

通过 git 比较发现，Untiy 工程内容通常会影响到 Xcode 的 Data、Classes、Libraries 下的内容，所以我们可以先配置好一个 Xcode 工程，需要重新打包时就重新导出一个 Xcode，然后把新 Xcode 下的 Data、Classes、Libraries 替换到旧的工程里。

**配置流程**

1. 打开配置好的 Xcode 工程，选中 Data Classes Libraries，右键点 Move to Trash 方式删除

2. 使用untiy导出新的 Xcode，将新导出的 Xcode 中的 Data Classes Libraries 复制到旧工程中相同的位置(在第一步中已删除)。

3. 在旧工程中添加 Classes Libraries 文件，以 create groups 方式添加

4. 在旧工程中添加 Data 文件，以 create folder references 方式添加 

**注意事项**

第 2 步可不是多余的哦 😓，如果不执行第二步，直接去新 Xcode 目录下添加 Data、Classes、Libraries，编译工程时有可能报错，主要是丢失一些文件引用。这应该是 Xcode 的一个 Bug，我们公司的 Classes 文件下有 2w 多个 c++ 文件，经过多次实验，只有按以上流程走才能正常编译。

