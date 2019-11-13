---
layout: post
title: 使用 Jenkins 从 Unity 工程中自动化导出 iOS 包
categories: Unity
description: Unity 自动化导出 iOS 包
keywords: Unity, ios, jenkins
---

Unity 打 iOS 平台的包过程很繁琐，有时候测试人员也需要不同类型的包来验证问题，出包需求较频繁。如果都由开发人员来出包会比较浪费时间，梳理整个流程后发现很多操作是重复的，不同类型的包区别在于版本号、游戏内容（可以用 git commitId 或 svn 版本号来表示）和一些业务参数，我们可以将这些参数由 Jenkins 配置，再结合 Shell、Python 脚本执行参数化构建，实现整个流程的自动化，让测试人员也可以打自己需要的包。

Unity 打 iOS 包的流程，可以看做下面三步，下面讲解每一步如何用脚本来执行，最后用 Jenkins 将整个流程串起来，实现在 Jenkins 网页上选择参数后一键打包。

1. Unity 工程导出 Xcode 工程
2. 配置 Xcode 工程，往 Xcode 里添加 iOS SDK 文件并修改 Xcode 编译配置
3. Xcode 工程导出 ipa 包

## Unity 工程导出 Xcode 工程
### 使用 UnityEditor 中 Xcode 相关的 Api
`UnityEditor` 提供了生成 Xcode 和配置 Xcode 的 Api。

`UnityEditor.BuildPipeline`提供了一个函数`BuildPlayer`用来导出成 Xcode 工程。

`UnityEditor.PlayerSettings`提供了各种修改 Xcode 工程基础配置的 Api，比如包名、版本号。

在 Unity 工程 Editor 目录下新建一个 iOSBuilder.cs，内容如下。
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

新建一个 build.sh，内容如下。
```
# Unity 程序路径
UNITY_PATH=/Applications/Unity/Unity.app/Contents/MacOS/Unity

# Unity 工程路径
PROJECT_PATH=/Users/sunsetroad/demo

# iOSBuilder 中 SetUnityParams 会读取这些参数，并作用于 PlaySetting
buildArgs="bundleIdentifier=test.com;bundleVersion=1.0;productName=test"

# 执行 iOSBuilder 方法
$UNITY_PATH -projectPath ${PROJECT_PATH} -executeMethod iOSBuilder.Build project-$buildArgs -quit
```
这里的参数都是可灵活配置的，为后面 Jenkins 一键打包做准备。

## 配置 Xcode 工程
Unity 应用中经常会用到一些 iOS 原生的 Api，我们需要在导出 Xcode 后，往 Xcode 中添加一些 OC 文件和依赖库以满足 C# 层的调用。

配置可能包括修改 Xcode 的 Info.plist、BuildSetting，添加 SDK文件和相关的依赖库，下面这些方法库可以帮助实现。

- [XUPorter](https://github.com/onevcat/XUPorter)


## Xcode 工程导出 ipa 包

## Jenkins 一键打包