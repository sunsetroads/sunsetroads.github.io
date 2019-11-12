---
layout: post
title: 使用 Jenkins 从 Unity 工程中自动化导出 iOS 包
categories: Unity
description: Unity 自动化导出 iOS 包
keywords: untiy, ios, jenkins
---

Unity 打 iOS 平台的包过程很繁琐，有时候测试人员也需要不同类型的包来验证问题，出包需求较频繁。如果都由开发人员来出包会比较浪费时间，梳理整个流程后发现很多操作是重复的，不同类型的包区别在于版本号、游戏内容（可以用 git commitId 或 svn 版本号来表示）和一些业务参数，我们可以将这些参数由 Jenkins 配置，再结合 Shell、Python 脚本执行参数化构建，实现整个流程的自动化，让测试人员也可以打自己需要的包。

Untiy 打 iOS 包的流程，可以看做下面三步，下面讲解每一步如何用脚本来执行，最后用 Jenkins 将整个流程串起来，实现在 Jenkins 网页上选择参数后一键打包。

1. Untiy 工程导出 Xcode 工程
2. 配置 Xcode 工程，往 Xcode 里添加 iOS SDK 文件并修改 Xcode 编译配置
3. Xcode 工程导出 ipa 包

## Untiy 工程导出 Xcode 工程

## 配置 Xcode 工程

## Xcode 工程导出 ipa 包

## Jenkins 一键打包