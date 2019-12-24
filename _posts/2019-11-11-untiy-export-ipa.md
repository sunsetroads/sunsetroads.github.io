---
layout: post
title: 使用 Jenkins 从 Unity 工程中自动化导出 iOS 包
categories: Unity
description: Unity 自动化导出 iOS 包
keywords: Unity, ios, jenkins
---

Unity 打 iOS 平台的包过程比较繁琐，手动操作会比较浪费时间还容易出错。这里把一些出包参数放在了 Jenkins 上配置，再结合 Shell、Python 脚本执行参数化构建，实现整个流程的自动化。

Unity 打 iOS 包的流程，可以看做下面三步，下面讲解每一步如何用脚本来执行，最后用 Jenkins 将整个流程串起来，实现在 Jenkins 上选择参数后一键打包。

1. Unity 工程导出 Xcode 工程
2. 配置 Xcode 工程，往 Xcode 里添加 iOS SDK 文件并修改 Xcode 编译配置
3. Xcode 工程导出 ipa 包

## Unity 工程导出 Xcode 工程
### UnityEditor
**UnityEditor** 提供了生成 Xcode 和修改 Xcode 工程配置的 Api。

在 Unity 工程 Editor 目录下新建一个 iOSBuilder.cs，内容如下：
```
using UnityEngine;
using System.Collections;
using System.Collections.Generic;
using UnityEditor;

public class iOSBuilder:Editor
{
	public static void Build()
	{
		SetUnityParams ();
		// UnityEditor.BuildPipeline 提供了一个函数`BuildPlayer`用来从 Untiy 工程导出成 Xcode 工程。
		BuildPipeline.BuildPlayer(ProjectBuilder.GetBuildScenes(), "/Users/sunsetroad/Desktop/test", BuildTarget.iOS, BuildOptions.None);
	}

	void SetUnityParams()
	{
		foreach(string arg in System.Environment.GetCommandLineArgs())
		{
			if(arg.StartsWith("project-", System.StringComparison.Ordinal))
			{
				ModifyPlayerSetting(arg.Replace("project-", ""), BuildTargetGroup.iOS);
			}
		}
	}

	//修改 Xcode 参数，
	void ModifyPlayerSetting(string arg, BuildTargetGroup targetGroup)
	{
		string[] argsArray = arg.Split(';');
		for (int i = 0; i < argsArray.Length; i++)
		{
			string[] argSprite = argsArray[i].Split('=');
			if (argSprite.Length != 2)
				continue;
			switch (argSprite[0].Trim())
			{
				// UnityEditor.PlayerSettings 提供了各种修改 Xcode 工程基础配置的 Api，比如包名、版本号等
				case "bundleIdentifier":
					PlayerSettings.applicationIdentifier = argSprite[1];
					break;
				case "bundleVersion":
					PlayerSettings.bundleVersion = argSprite[1];
					break;
				case "productName":
					PlayerSettings.productName = argSprite[1];
					break;
			}
		}
	}
}
```
当执行`iOSBuilder.Build`时，就会导出一个 Xcode 工程。

### Shell 中调用 C# 的方法
我们最终的目的是使用命令执行所有的操作，所以需要使用脚本调用`iOSBuilder.Build`。

新建一个 build.sh，内容如下：
```
# Unity 程序路径
UNITY_PATH=/Applications/Unity/Unity.app/Contents/MacOS/Unity

# Unity 工程路径
PROJECT_PATH=/Users/sunsetroad/demo

# iOSBuilder 中 SetUnityParams 会读取这些参数，并作用于 PlaySetting
buildArgs="bundleIdentifier=test.com;bundleVersion=1.0"

# 执行 iOSBuilder 方法
$UNITY_PATH -projectPath ${PROJECT_PATH} -executeMethod iOSBuilder.Build project-$buildArgs -quit
```
这里的参数都是可灵活配置的，为后面 Jenkins 一键打包做准备。

## 配置 Xcode 工程
Unity 应用有时候会需要调用一些 iOS 原生的 Api，这个交互流程这里不做详述，通常是在 C# 层定义好接口然后直接调用，然后在 OC 层去实现对应的接口。这时候就需要在导出 Xcode 后添加一些 OC 文件和依赖库，以及修改一些编译配置。

### 操作 Xcode 配置的几种方法
Xcode 是通过 pbxproj 文件来查找项目中的文件和工程的编译配置，pbxproj 通过 交叉 UUID 来索引，不方便直接读取和修改，下面这些库都是用来操作 pbxproj 文件的。

| 插件 | 问题 |
| ------ | ------ |
| [mod-pbxproj](https://github.com/kronenthaler/mod-pbxproj) | 无法修改 Xcode 的 Capabilities |
| [XUPorter](https://github.com/onevcat/XUPorter) | 作者不维护了，有一些 Bug 和功能缺陷 |
| [UnityEditor.iOS.Xcode](https://docs.unity3d.com/ScriptReference/iOS.Xcode.PBXProject.html) | 需要较高版本的 Unity |

XUPorter 和 UnityEditor.iOS.Xcode 都是使用 C# 开发。Untiy 的 [PostProcessBuild] 标签标注的函数会在导出 Xcode 后自动调用，我们可以在此函数中调用这两个插件来配置 Xcode，这样即使手动导出的 Xcode 也无需重复配置，是种不错的做法。

个人对 python 更熟悉一些，也基本没有手动出包的需求，就在 mod-pbxproj 的基础上开了一套 Xcode 相关工具 [Xcode-Tools](https://github.com/sunsetroads/Xcode-Tools)，添加了对 Xcode Capability 的修改，并提供了 ini 配置文件来表示 Xcode 中的各个选项，方便 Untiy 开发人员使用。

### 使用 Xcode-Tools

新建 start.py，添加以下内容：
```
from xcodetools import *

config_path ='/Users/sunsetroad/Desktop/config.ini'

project_path = '/Users/sunsetroad/Desktop/test'

Xcode.modify(project_path, config_path)
```

参考 [配置规则](https://github.com/sunsetroads/Xcode-Tools/blob/master/config.ini)，将 Xcode 配置写在一个 .ini 文件中，然后在 build.sh 中追加以下内容：
```
python3 /Users/sunsetroad/Desktop/Xcode-Tools/start.py
```

执行 build.sh，就会得到一个配置完善的 Xcode，剩下来就是打包的事了。

## Xcode 工程导出 ipa 包
Xcode 自动化打包网上教程已经太多了，我在 [Xcode-Tools](https://github.com/sunsetroads/Xcode-Tools) 中封装了 Package 模块，传入所需参数即可导出一个 ipa 包。

使用方法如下：
```
from xcodetools import Package

project_path = '/Users/sunsetroad/Desktop/test'

ipa_path = '/Users/sunsetroad/Desktop/IPA/test.ipa'

plist = '/Users/sunsetroad/Desktop/ExportOptions.plist'

# 开始自动打包
Package.build (project_path, ipa_path, plist)
```

为了将所有操作放在一个脚本里，我们再修改一下上一步中的 start.py 和 build.sh，并将每次打包时会变的参数改为从环境变量中获取。

**start.py**
```
from xcodetools import *

config_path ='/Users/sunsetroad/Desktop/config.ini'

project_path = '/Users/sunsetroad/Desktop/test'

ipa_path = '/Users/sunsetroad/Desktop/IPA/test.ipa'

plist = '/Users/sunsetroad/Desktop/ExportOptions.plist'

Xcode.modify(project_path, config_path)

Package.build(project_path, ipa_path, plist)

```

**build.sh**
```
bundleIdentifier=$1
bundleVersion=$2
commitId=$3

# Unity 程序路径
UNITY_PATH=/Applications/Unity/Unity.app/Contents/MacOS/Unity

# Unity 工程路径
PROJECT_PATH=/Users/sunsetroad/demo

# 更新工程
cd ${PROJECT_PATH}
git clean -f
git reset --hard ${commitId}

# iOSBuilder 中 SetUnityParams 会读取这些参数，并作用于 PlaySetting
buildArgs="bundleIdentifier=${bundleIdentifier};bundleVersion=${bundleVersion}"

# 执行 iOSBuilder 方法
$UNITY_PATH -projectPath ${PROJECT_PATH} -executeMethod iOSBuilder.Build project-$buildArgs -quit

# 配置 Xcode 并打包
python3 /Users/sunsetroad/Desktop/Xcode-Tools/start.py
```

## Jenkins 一键打包

现在只要这样执行 build.sh 就可以得到一个需要的 ipa 包了
```
./build.sh bundleIdentifier bundleVersion commitId
```

**使用 Jenkins 参数化构建功能**

在 Jenkins 配置中勾选参数化构建，并添加以下参数：

![](/images/jenkins_param.png)

添加执行 shell 指令：

![](/images/jenkins_shell.png)

启动该配置，设置相关参数后点击开始构建。

![](/images/jenkins_start.png)

## 总结

本文介绍了如何使用脚本来执行 Untiy 导出 iOS 包的每一步操作，并在最后将一些参数配置在 Jenkins 上，借助于 Jenkins 的参数化构建功能，让非开发人员也可以自由打包。

在例子中只要在 Jenkins 上选择一个包名，填一下版本号和 git commitId 即可打出一个包，这里的参数仅为示例，主要是提供了一种自动化构建的思路，使用时需要结合业务需求，在各个环节间新增脚本任务或参数。