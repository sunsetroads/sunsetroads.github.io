---
layout: post
title: Xcode 编译报错汇总
categories: Git
description: Xcode 编译报错汇总
keywords: xcode, error
---

公司主要做 Untiy 开发，Untiy 导出的 Xcode 工程经常会遇到一些编译错误，这里记录下遇到的报错和解决办法。其中问题描述尽量使用文字，以后可以复制 Xcode 中的错误日志来搜索。解决办法则尽可能添加图片说明，方便不熟悉 Xcode 的人使用。

---
### 未定义符号

这是出现过很多此的问题，一些 SDK 需要添加一些依赖库，但没有添加上：
 > Undefined symbols for architecture arm64:
 "0BJC_CLASS_$_CLLocationmanager", referenced 	from: objc-class-ref in libyim.a (Geographylocation.o)

**解决办法**

根据提示，说明 libyim.a 引用到了未定义的符合，所以先查找_CLLocationmanager 的来源，发现属于 CoreLocation.framwork，这是一个系统库。在 Build Phases 里添加依赖即可:

![](/images/xcode/undefined_sym.png)

---
### armv7 的编译错误

Untiy 开发人员反馈最近添加了 libuwa.a 库后出现下面的报错，但在 xcode 工程中将此库删除并重新添加后又不报错了。

测试发现，在 Xcode Build Phases 将 此库调整顺序后也不报错。

> B/bl/blx thumb2 branch out of range  (71414688 max is+/-16MB)

**解决办法**

最终在网上找到了原因，这是一个已知的 armv7 编译器链接错误，可以删除该库重新导入，或者将 Xcode 工程 BuildSetting 中的 Architectures 中的 armv7 删除掉：

![](/images/xcode/undefined_sym.png)

参考：https://groups.google.com/forum/#!topic/pdfnet-sdk/b4EoBiH_zjc

---
### Xcode 控制台 po text 报错

Xcode 调试时明明有值却显示 nil，po 命令报下面错误：

>error: Couldnt materialize: couldnt get the value of variable text: variable not available 
>error: errored out in Doexecute, couldn't Preparetoexecutejit Expression

**解决办法**

这是工程编译策略的问题，先找到运行的 Build Configuration，这里是 Debug

![](/images/xcode/build_configtion.png)

然后去 Build Settings 里搜索 Optimization Level，将 Debug 选项中的 fastest, Smallest[-Os] 改为 None。

![](/images/xcode/optimal_none.png)

---

### 真机调试报错

证书设置都没有问题，但无法在手机上运行，提示如下:

>  iPhone has denied the launch request.
Internal launch error: process launch failed: failed to get the task for process 1304

**解决办法**

去 Edit Scheme 里将 Executable 改为 Ask on Lanuch

![](/images/xcode/edit_scheme.png)

![](/images/xcode/exec.png)

---