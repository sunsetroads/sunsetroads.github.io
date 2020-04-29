---
layout: post
title: Mac 下 Mono-Unity 编译 libmono.so 过程详解
categories: Unity
description: Mac 下 Mono-Unity 源码编译过程详解
keywords: mono, unity
---

Unity 打出的安卓包为了防止反编译，需要对 Assembly-CSharp.dll 加密处理。Assembly-CSharp.dll 是由 libmono.so 运行时读取然后在 mono 虚拟机上执行，所以需要修改 libmono.so 源码，在加载 Assembly-CSharp.dll 前解密处理，然后重新编译出 libmono.so。

libmono.so 是由 Unity 官方 Fork 了开源的 Mono 编译出来的，Unity 官方也将其开源了，我们要编译的源码在这里：
- [https://github.com/Unity-Technologies/mono/tree/unity-2018.4]()

这次我编译的是 Unity-2018.4 的，你也可以根据你的 Unity 版本下载对应分支的。

大多数人都经历过，照着别人的文档，甚至官方的，别人的操作成功了，自己的却一堆错 ...

Mono-Unity 的编译环境比较复杂，依赖的工具链较多，不可避免也会有一些错误，这篇文章就用来讲述这个编译过程，附带我遇到的错误和解决办法。了解这个编译的过程，这样即使出现其他报错，也可以快速定位。

## Mono-Unity 编译环境配置

编译 Untiy-Mono 需要安装一些编译脚本依赖的包。HomeBrew 是 MacOS 上的包管理工具，使用它安装这些依赖会很方便。

### 安装 HomeBrew
安装 HomeBrew，有时候不太顺利，这里提供 3 种安装方式，安装失败时可以切换试试。

#### 按官网教程安装

官网中介绍的安装方式，执行下面的命令即可
```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```
我自己的 Mac 上最早就是这样安装的，但最近给另一台 Mac 配置环境时碰到了下图的错误，用 VPN 也不行。

![](/images/mono/brew-error.png)

#### 使用国内源安装
在知乎上找到的一个国内源：
```
/bin/zsh -c "$(curl -fsSL https://gitee.com/cunkai/HomebrewCN/raw/master/Homebrew.sh)"
```

#### 使用 Ruby 脚本安装

将 [https://gist.github.com/sunsetroads/11c35fb3caef2980041b1fcb07ab9a31]() 的内容复制保存为 homebrew.rb，然后执行命令：
```
ruby homebrew.rb
```

#### 检查是否安装成功
执行`brew --help`检查是否安装成功

![](/images/mono/brew-success.png)

出现上图内容就说明 HomeBrew 安装成功了。

### 使用 HomeBrew 安装 Mono-Unity 依赖

Mono-Unity 依赖下面这些包

- autoconf

- automake

- libtool

- pkg-config

使用 HomeBrew 一个个安装就行了：
```
brew install autoconf
```

## Mono-Unity 编译流程
这里以 mono-untiy-2018.4 为例，下载后在桌面新建文件夹 Test/T，将下载下来的源码放入，编译脚本运行后会在源码工程上级目录安装依赖，这样建目录方便查看。

### 直接编译一下

进入工程根目录 mono-untiy-2018.4，执行命令
`./external/buildscripts/build_runtime_android.sh`开始编译:

![](/images/mono/build_start.png)

等了挺久，然后编译失败了：

![](/images/mono/build_error.png)

查看日志还发现了时间久的原因，下载了 NDK-r10e 后，又去下载了 NDK-r16，总共 1G 多的文件，比较浪费时间了：

![](/images/mono/build_ndk.png)

在网上搜了一会，没有好的解决办法，决定看下编译脚本的执行过程，来查找报错的根本原因。

*PS*：这一步还可能提示缺少什么包，按提示执行`brew install 包名`即可。

### 编译脚本执行过程
build_runtime_android.sh 就是入口脚本，先忽略掉杂要信息，看下它的关键内容：
```sh
function clean_build_krait_patch
{
	# 检查是否有下载 krait-signal-handler，并执行 build.pl
	KRAIT_PATCH_PATH="${CWD}/../../android_krait_signal_handler/build"
	local KRAIT_PATCH_REPO="git://github.com/Unity-Technologies/krait-signal-handler.git"
	git clone --branch "master" "$KRAIT_PATCH_REPO" "$KRAIT_PATCH_PATH"
	(cd "$KRAIT_PATCH_PATH" && ./build.pl)
}

function clean_build
{
	make clean && make distclean

	./configure 

	if [ "$?" -ne "0" ]; then 
		echo "Configure FAILED!"
		exit 1
	fi

	make && echo "Build SUCCESS!" || exit 1
	...
}

perl ${BUILDSCRIPTSDIR}/PrepareAndroidSDK.pl -ndk=r10e -env=envsetup.sh && source envsetup.sh

clean_build_krait_patch

clean_build "$CCFLAGS_ARMv7_VFP" "$LDFLAGS_ARMv7" "$OUTDIR/armv7a"
```

首先执行的`perl ${BUILDSCRIPTSDIR}/PrepareAndroidSDK.pl`，注意这里传入的参数 ndk-r10e，再看下还后执行的`clean_build_krait_patch`，先去下载了 krait-signal-handler包，然后执行了它的 build.pl，看下 build.pl 内容：

```pl
sub BuildAndroid
{
	PrepareAndroidSDK::GetAndroidSDK(undef, undef, "r16b");
	system('$ANDROID_NDK_ROOT/ndk-build clean');
	system('$ANDROID_NDK_ROOT/ndk-build');
}
```
这里传入的参数是 r16b，也就是说，构建脚本依赖的 NDK 版本和 krait-signal-handler 依赖的不一致，导致了重复下载，所以将 build.pl 中的 r16b 改为 r10e。


