---
layout: post
title: Xcode 编译报错汇总
categories: Xcode
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

根据提示，说明 libyim.a 引用到了未定义的符号，所以先查找_CLLocationmanager 的来源，发现属于 CoreLocation.framwork，这是一个系统库。在 Build Phases 里添加依赖即可:

![](/images/xcode/undefined_sym.png)

---
### armv7 的编译错误

Untiy 开发人员反馈最近添加了 libuwa.a 库后出现下面的报错，但在 Xcode 工程中将此库删除并重新添加后又不报错了。

测试发现，在 Xcode 的 Build Phases 中将此库调整顺序后也不报错。

> B/bl/blx thumb2 branch out of range  (71414688 max is+/-16MB)

**解决办法**

最终在网上找到了原因，这是一个已知的 armv7 编译器链接错误，可以删除该库重新导入，或者将 Xcode 工程 BuildSetting 中的 Architectures 中的 armv7 删除掉：

![](/images/xcode/armv7_error.png)

参考：[https://groups.google.com/forum/#!topic/pdfnet-sdk/b4EoBiH_zjc](https://groups.google.com/forum/#!topic/pdfnet-sdk/b4EoBiH_zjc)

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

Xcode 在手机上运行后卡在启动页，Xcode 提示如下:

> iPhone has denied the launch request.
Internal launch error: process launch failed: failed to get the task for process 1304

**暂时的解决办法**

去 Edit Scheme 里将 Executable 改为 Ask on Lanuch。目前还没找到真正有效的解决方法，网上说是 Xcode 的 bug，暂时这样解决。

![](/images/xcode/edit_scheme.png)

![](/images/xcode/exec.png)

---
### C++ 混编使用 @import 错误

使用 FaceBook 的 SDK 时，它的有个头文件使用了`@import`的写法，编译时出现下面的报错
> use of @import when modules are disabled

**解决办法**

这是因为我们工程中存在`.mm`后缀的`C++`文件，需要在 Xcode 工程 Build Settings 中 将 Enable Modules 设置为 Yes。

![](/images/xcode/enable_module.png)


然后搜索`other c++ flags`，添加`-fcxx-modules`和`-fmodules`符号。

![](/images/xcode/module-error.png)

---

### 在 Xcode 中使用动态库

直接往 Xcode 工程中拖入一个库，编译虽然可以通过，但运行时会报一个 image not found 的错误，然后闪退。

> dyld: Library not loaded: @rpath/AdjustSdk.framework/AdjustSdk
  Referenced from: /private/var/containers/Bundle/Application/AD1BBE93-2E11-462B-AA2E-F5161C99345F/yjcq.app/Frameworks/npplaygamesdk.framework/npplaygamesdk
  Reason: image not found

**解决办法**

这是因为 Adjust.framework 是一个动态库，往 Xcode 添加动态库时需要去 General 中找到该库，将 Do not Emebd 改为 Embed & sign

![](/images/xcode/embed.png)

**区分动态库和静态库**

打开终端，使用 cd 命令进入 xxx.framework, 然后使用 file 命令查看该二进制文件，动态库会有**Mach-O dynamicallly**的标识。

![](/images/xcode/dym.png)

---

**Modules 报错**

导入一个 framework 后它的头文件出现报错
> Include of non-modular header inside framework module

**解决办法**

去 Xcode 的 Build Setting 中将此项改为 Yes。

![](/images/xcode/allow.png)

---