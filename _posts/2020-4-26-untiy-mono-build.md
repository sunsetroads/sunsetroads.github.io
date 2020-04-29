---
layout: post
title: Mac 下 Mono-Unity 编译 libmono.so 详解
categories: Unity
description: Mac 下 Mono-Unity 源码编译详解
keywords: mono, unity
---

Unity 打出的安卓包为了防止反编译，需要对 Assembly-CSharp.dll 加密处理。Assembly-CSharp.dll 是由 libmono.so 运行时读取然后在 mono 虚拟机上执行，所以需要修改 libmono.so 源码，在加载 Assembly-CSharp.dll 前解密处理，然后重新编译出 libmono.so。

libmono.so 是由 Unity 官方 Fork 了开源的 Mono 编译出来的，Unity 官方也将其开源了，源码在这里：
- [https://github.com/Unity-Technologies/mono/tree/unity-2018.4]()

这次我编译的是 Unity-2018.4 的，你也可以根据你的 Unity 版本下载对应分支的。

大多数人都经历过，照着别人的文档，甚至官方的，别人的操作成功了，自己的却一堆错

Mono-Unity 的编译环境比较复杂，依赖的工具链较多，不可避免也会有一些错误，这篇文章就用来讲述这个编译过程，附带我遇到的错误和解决办法。了解这个编译的过程，这样即使出现其他报错，也可以快速定位。

## 编译环境配置

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

### 使用 HomeBrew 安装依赖

Mono-Unity 依赖下面这些包

- autoconf

- automake

- libtool

- pkg-config

使用 HomeBrew 一个个安装就行了：
```
brew install autoconf
```

## 编译 libmono.so
这里以 mono-untiy-2018.4 为例，下载后在桌面新建文件夹 Test/T，将下载下来的源码放入，编译脚本运行后会在 mono-untiy-2018.4 上级目录安装依赖，这样建目录会方便查看依赖包。

### 直接编译一下

进入工程根目录 mono-untiy-2018.4，执行编译脚本
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
}

perl ${BUILDSCRIPTSDIR}/PrepareAndroidSDK.pl -ndk=r10e -env=envsetup.sh && source envsetup.sh

clean_build_krait_patch

clean_build "$CCFLAGS_ARMv7_VFP" "$LDFLAGS_ARMv7" "$OUTDIR/armv7a"
```

首先执行的`perl ${BUILDSCRIPTSDIR}/PrepareAndroidSDK.pl`，注意这里传入的参数 ndk-r10e。然后执行`clean_build_krait_patch`，先去下载了 krait-signal-handler 包，然后执行里面的 build.pl ：

```pl
sub BuildAndroid
{
	PrepareAndroidSDK::GetAndroidSDK(undef, undef, "r16b");
	system('$ANDROID_NDK_ROOT/ndk-build clean');
	system('$ANDROID_NDK_ROOT/ndk-build');
}
```
这里 krait-signal-handler 中 build.pl 传入的参数是 r16b，也就是说，构建脚本依赖的 NDK 版本和 krait-signal-handler 依赖的不一致，导致了重复下载，所以要去把 build.pl 中的 r16b 改为 r10e。

编译脚本安装了依赖的环境后，接着往下执行`clean_build`:
```sh
make clean && make distclean

./configure 

if [ "$?" -ne "0" ]; then 
	echo "Configure FAILED!"
	exit 1
fi

make && echo "Build SUCCESS!" || exit 1
```

./configure、make、make install 命令这些都是典型的使用 GNU 的 AUTOCONF 和 AUTOMAKE 产生的程序的安装步骤，这里需要了解下 configure 和 make 命令。

### configure 和 make

在 Linux 下安装一个应用程序时，一般先运行脚本 configure，然后用 make 来编译源程序，在运行 make install，最后运行 make clean 删除一些临时文件。使用上述三个自动工具，就可以生成 configure 脚本。运行configure 脚本，就可以生成 Makefile 文件，然后就可以运行 make、make install 和 make clean。

configure 是一个 shell 脚本，它可以自动设定源程序以符合各种不同平台上 Unix 系统的特性，并且根据系统叁数及环境产生合适的 Makefile 文件或是 C 的头文件 (header file)，让源程序可以很方便地在这些不同的平台上被编译链接。

运行 configure 脚本，就可产生出符合 GNU 规范的 Makefile 文件了，然后就可以运行 make 进行编译，再运行 make install 进行安装了，这里只需要编译。

引用自 [https://www.cnblogs.com/tinywan/p/7230039.html]()

### 错误查找过程
了解了编译脚本的执行过程后，可以开始根据执行的 log 找编译失败的原因了。

在上面编译失败的 log 中可以看到一句`make: *** No rule to make target 'clean'.  Stop`：

![](/images/mono/build_mis_makefile.png)

不熟悉 make 命令时还以为缺少什么环境，但其实编译失败和这个没关系，这是 configure 执行失败导致没有正常生成 Makefile，make 命令找不到 Makefile 文件后提示的，问题是出在 configure 脚本里。

中间的日志都可以忽略，直接看最后的报错：

![](/images/mono/build_error.png)

这里提示 C compiler cannot create executables，检查了系统的 gcc 和 clang，都是可以正常编译 C 程序的，网上也找不到解决办法。于是去 config.log 查看更详细的信息。

执行 configure 时会把执行过程的详细信息输出到 config.log，终端中输出的只是一份简要的，这两个文件都位于 mono-untiy-2018.4 根目录下。打开 config.log，在里面搜 C compiler cannot create executables，可以看到出现这个错误前发生了一些 error。

![](/images/mono/build_config_error.png)

来看这些 error，首先是 -V 和 -qvesion 问题，提示的是 arm-linux-androideabi-gcc 不支持这些参数，这个输出在 configure 的 4500 行，看下对应的代码：

![](/images/mono/complier_version_check.png)

这段代码是用来检查 arm-linux-androideabi-gcc 版本的，尝试了 --version -v -V -qversion 四个参数，使用 --veesion 和 -v 就可以获取到了，其他的参数不适用也无所谓，这个并不是真正的错误原因。

根据 error 发生的位置继续查看 configure 源码，从 4531 行到 4602 行，代码有点多，需要联系上下文才能理解。这段代码作用是来检查 NDK 中的 arm-linux-androideabi-gcc 是否正常，判断的标准是用它编译一段简单的 C 程序，然后查看是否生成了可执行文件。这里最终没有生成，所以抛出了个 C compiler cannot create executables。

config.log 里提到发生了一个 ld 链接器错误，cannot find -lkrait-signal-handler，忽略掉杂要信息后， configure:4553 这一行是这样的：
```
arm-linux-androideabi-gcc -L/Test/T/mono-unity-2018.4/../../android_krait_signal_handler/build/obj/local/armeabi -lkrait-signal-handler conftest.c
```
意思是使用 arm-linux-androideabi-gcc 编译 conftest.c，并链接库 krait-signal-handler。

GCC 会在 -L 选项后紧跟着的基本名称的基础上自动添加前缀 lib、后缀 .a，这里基本名称为 krait-signal-handler。现在去 -L 后面的路径下看下是否存在 libkrait-signal-handler.a：

![](/images/mono/gcc_link.png)

这时发现 libkrait-signal-handler.a 是存在的，只是前面的路径不对，configure 脚本以为 libkrait-signal-handler.a 位于 armabi 下，但实际编译出来的在 armabi-v7a，问题找到了，新建个 armabi 的目录将 krait-signal-handler.a 放入，再执行编译脚本。

等待终端刷屏了近 10 分钟，输出了下面的信息：

![](/images/mono/build_success.png)

编译成功了，在 builds/embedruntimes/android 下可以看到不同 CPU 架构下的 libmono.so。

### 编译选项优化

修改 ./external/buildscripts/build_runtime_android.sh 文件, 在这个脚本中修改 `-fpic -g -funwind-tables \` 为 `-fpic -O2 -funwind-tables \`， `-g` 打出来的 libmono.so 是 debug 版本的，文件会比较大。

修改 ./external/buildscripts/build_runtime_android_x86.sh，在这个脚本中把 `-fpic -g \` 修改为 `-fpic \`，这个修改据说是因为 x86 的编译选项和 arm 不一样，不去掉 `-g` 一些手机上进游戏会卡死。

如果只需要 armv7a 和 x86 的，可以在 build_runtime_android.sh 中注释掉下面两项：

`clean_build "$CCFLAGS_ARMv5_CPU" "$LDFLAGS_ARMv5" "$OUTDIR/armv5"` 

`clean_build "$CCFLAGS_ARMv6_VFP" "$LDFLAGS_ARMv5" "$OUTDIR/armv6_vfp"`
