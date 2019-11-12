---
layout: post
title: 使用 Jenkins 从 Unity 工程中自动化导出 iOS 包
categories: Unity
description: Unity 自动化导出 iOS 包
keywords: untiy, ios, jenkins
---

Unity 打 iOS 平台的包过程很繁琐，有时候测试人员也需要不同类型的包来验证问题，出包需求较频繁。如果都由开发人员来出包会比较浪费时间，梳理整个流程后发现很多操作是重复的，不同类型的包区别在于版本号、渠道、客户端环境等参数，这里将这些参数由 Jenkins 指定来配置，再结合 Shell、Python 脚本，实现整个流程的自动化，让测试人员自己也可以打需要的包。


### Untiy打iOS包的流程，可以看做下面三步

1. untiy工程导出xcode工程
2. 往xcode里添加OC文件并修改xcode配置
3. xcode工程导出ipa包

### Untiy工程导出xcode工程

### 配置Xcode工程

### 自动化打包

