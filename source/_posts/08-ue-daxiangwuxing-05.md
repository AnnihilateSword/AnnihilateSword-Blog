---
title: 【UE大象无形总结】第三部分扩展虚幻引擎【终篇】
tags: [UE4]
date: 2025-01-11
updated: 2025-01-11
cover: /res/img/post/08-ue-daxiangwuxing-05/cover.png
top_img: /res/img/site/background.png
---

![](/res/img/post/08-ue-daxiangwuxing-05/cover.png)

<br>

# 前言

> **三千弱水，只取一瓢饮。**
> 
> 参考书籍：《大象无形：虚幻引擎程序设计浅析 (罗丁力 [罗丁力])》
> 补充说明：本系列是个人对书中内容的实践

<br>
<br>

# 十四、引擎独立应用程序

> 我该如何写出 `UHT` 这样，能够引用虚幻引擎的 `API`，又不需要依赖引擎运行的程序呢？

<br>

## 14.1 简介

什么是引擎独立应用程序？让我们首先看一下虚幻引擎的 `Launcher` 运行器。不知道大家是否曾经好奇，虚幻引擎的 `Launcher` 运行器是用什么写的呢？如果你曾经仔细观察过虚幻引擎的运行器的文件结构，你会惊讶地发现，这个文件结构非常类似于虚幻引擎本身的文件结构，而非虚幻引擎编译打包形成的游戏的文件结构。这意味着，虚幻引擎的运行器是一个微型虚幻引擎 ，而不是一个打包形成的游戏！

如何开发这样的东西呢？如果我们希望在一个小型的、不是游戏的应用程序中继续使用虚幻引擎的一部分API，那么我们就需要学习虚幻引擎运行器这样的技术。

<br>

## 14.2 如何开始

关于如何撰写这样的应用程序的介绍，虚幻引擎的官方文档语焉不详，几乎没有说明什么。但是虚幻引擎提供了几个案例性的程序，如 `BlankProgram` 和 `SlateViewer`。我们可以从这几个应用程序着手进行分析和学习。

简而言之，开发这样的程序需要遵循以下步骤：

1. 建立文件结构。
2. 配置 `.Target.cs` 和 `.build.cs`。
3. 撰写代码。
4. 剥离应用程序。

由于没有了虚幻引擎提供的打包机制，我们需要自己完成应用程序的剥离和发布。这个发布过程并不是一个简单地做加法的过程，而是一个做减法的过程。我们不是选择出哪些文件是我们需要的，并添加到程序中，而是选择出哪些文件是我们不需要的，从程序中删除。

那么，让我们首先把目光投向虚幻引擎源代码目录下的一个小小的文件夹，就是 `Engine/Source/Programs/BlankProgram`。

<br>

## 14.3 BlankProgram

这个文件夹包含了以下的文件结构：

```cpp
- Private
    - BlankProgram.h
    - BlankProgram.cpp
- Resources
    - Windows
        - BlankProgram.ico
- BlankProgram.Build.cs
- BlankProgram.Target.cs
```

如何编译呢？你只需要打开源码版本的虚幻引擎的工程文件，然后运行解决方案窗口中的 `BlankProgram` 项目，你会看到弹出了一个命令行窗口，显示出了一行 **“HelloWorld”**。

![](/res/img/post/08-ue-daxiangwuxing-05/14-3-1.png)

在对源码进行分析之前，我们必须先关注如何配置这个工程。怎么告诉 `UHT` 和 `UBT`，我们需要编译出一个可以运行的 `.exe` 文件，而不是虚幻引擎的插件、模块或者游戏？

答案是：我们通过 `.Target.cs` 指定。如果你忘记了 `UBT` 的相关内容，可以看第二部分中，对 `UBT` 虚幻引擎构建工具的描述。比较重要的函数主要有 `SetupBinaries` 和 `SetupGlobalEnvironment`。

> 下面代码截取自 UE4.27.2

```csharp
// Copyright Epic Games, Inc. All Rights Reserved.

using UnrealBuildTool;
using System.Collections.Generic;

[SupportedPlatforms(UnrealPlatformClass.Desktop)]
public class BlankProgramTarget : TargetRules
{
	public BlankProgramTarget(TargetInfo Target) : base(Target)
	{
		Type = TargetType.Program;
		LinkType = TargetLinkType.Monolithic;
		LaunchModuleName = "BlankProgram";

		// Lean and mean
		bBuildDeveloperTools = false;

		// Never use malloc profiling in Unreal Header Tool.  We set this because often UHT is compiled right before the engine
		// automatically by Unreal Build Tool, but if bUseMallocProfiler is defined, UHT can operate incorrectly.
		bUseMallocProfiler = false;

		// Editor-only data, however, is needed
		bBuildWithEditorOnlyData = true;

		// Currently this app is not linking against the engine, so we'll compile out references from Core to the rest of the engine
		bCompileAgainstEngine = false;
		bCompileAgainstCoreUObject = false;
		bCompileAgainstApplicationCore = false;

		// UnrealHeaderTool is a console application, not a Windows app (sets entry point to main(), instead of WinMain())
		bIsBuildingConsoleApplication = true;
	}
}
```

- 这里我们引入了模块名 `LaunchModuleName = "BlankProgram";`，指向的是 `.build.cs` 中定义的 `BlankProgram` 模块。
- 需要指定引入模块的编译类型，这里编译的类型是 `UEBuildBinaryType.Executable`。

![](/res/img/post/08-ue-daxiangwuxing-05/14-3-2.png)

编译类型指定非常重要，意味着我们的模块将不再被编译为二进制数据模块，用于编译链接，而是编译为可执行的 `exe` 应用程序。换句话说，对 `SetupBinaries` 函数的重载，完成了对编译为 `exe` 应用程序的指定。


继续看 `BlankProgram.Target.cs` 中的内容，我们能够看到 7个控制变量的设定，具体含义如下：

1. `bCompileLeanAndMeanUE`：这个控制变量表示编译为 “部分” 虚幻引擎，也就是非完整的虚幻引擎。
2. `bUseMallocProfile`：这个控制变量表示是否使用内存分配剖析器。`BlankProgram` 关闭了内存分配剖析。虚幻引擎给出的理由是，`UHT` 有可能会在虚幻引擎本身编译之前被编译（实际上经常就是这样）。如果先于引擎本身编译，同时又打开内存剖析，则 `UHT` 会出现异常工作。同样地，`BlankProgram` 也是一样。
3. `bBuildWithEditorOnlyData` 是 `BuildEditor` 控制。我们不需要一个编辑器，但是我们需要编辑器相关的数据。
4. `bCompiledAgainstEngine`、`bCompileAgainstCoreUObject` 和 `bCompileAgainstApplicationCore`：这三个标志控制了是直接静态链接还是动态链接的方式来与 `Engine` 模块、`CoreUObject` 模块 和 `ApplicationCore` 模块链接。这里 `BlankApp` 采用动态链接的方式以降低体积。
5. `bIsBuildingConsoleApplication`：是否编译为控制台命令程序。这实质是指定程序入口点。比如对于控制台应用程序，入口点将会是 `Main` 函数。

在此基础上，我们的 `HelloWorld` 程序就会显得比较简单，核心代码如下：

> 下面代码截取自 UE4.27.2

```cpp
// Copyright Epic Games, Inc. All Rights Reserved.


#include "BlankProgram.h"

#include "RequiredProgramMainCPPInclude.h"

DEFINE_LOG_CATEGORY_STATIC(LogBlankProgram, Log, All);

IMPLEMENT_APPLICATION(BlankProgram, "BlankProgram");

INT32_MAIN_INT32_ARGC_TCHAR_ARGV()
{
	GEngineLoop.PreInit(ArgC, ArgV);
	UE_LOG(LogBlankProgram, Display, TEXT("Hello World"));
	FEngineLoop::AppExit();
	return 0;
}
```

整个代码中包含的内容不多，相对来说，`Main` 函数中的内容理解起来比较简单，即预初始化引擎来完成日志系统初始化，从而支持 `UE_LOG` 宏，输出 `Hello World`。为了配合 `UE_LOG` 宏，我们需要定义 `logcategory`，所以需要在最前方书写 `DEFINE_LOG_CATEGORY_STATIC` 宏。

此时相对来说陌生的只有两个部分：

1. `RequiredProgramMainCPPInclude.h` 头文件的内容。
2. `IMPLEMENT_APPLICATION` 宏的含义。

对于前者，这个头文件包含了：

```cpp
// Copyright Epic Games, Inc. All Rights Reserved.

#pragma once

#include "CoreMinimal.h"
#include "LaunchEngineLoop.h"

// this is highly sketchy, but we need some stuff from launchengineloop.cpp
#include "Runtime/Launch/Private/LaunchEngineLoop.cpp"
```

- `CoreMinimal.h` 一些最小的核心模块
- `LaunchEngineLoop.h` 提供我们用的 `GEngineLoop` 定义。
- `LaunchEngineLoop.cpp` 官方注释：这是非常粗略的，但我们需要一些在`LaunchEngineLoop.cpp` 中的定义。

对于后者，实际上可以理解为和之前介绍的模块实现宏 `IMPLEMENT_MODULE` 一样，提供了模块的注册实现。

<br>

## 14.4 走得更远

整个程序就打印了一行日志输出，也许很多读者颇为怀疑笔者是不是在逗你玩。请勿慌张，先看看我们已经拥有了什么：

1. 可以引入虚幻引擎已有的运行时模块。
2. 可以包含虚幻引擎的类定义，从而使用虚幻引擎提供的公共 `API`。

在这个基础上，我们能走得更远：我们可以开发出一个引擎独立的 `Slate` 应用程序。若读者有兴趣，请按照以下步骤操作。

<br>

### 14.4.1 预先准备

请读者尽可能准备一个源码版的引擎进行试验。如果不知道如何获取 `Github` 上的虚幻引擎源码，请查看虚幻引擎官网上关于如何获取源码的资料。

如果读者不介意直接修改 `BlankProgram`，那就可以直接对 `BlankProgram` 代码进行操作（可以预先备份到别的文件夹）。否则，就需要复制一份 `BlankProgram`，然后重新命名文件夹名字，并对 `.Target.cs` 和 `.Build.cs` 中对应的名字进行修正。

下面我以后者做示例。

首先复制一份 `BlankProgram` 文件夹，我这里改名为 `TestProgram`，并对文件内容做以下修改：

`TestProgram.Target.cs`

```csharp
// Copyright Epic Games, Inc. All Rights Reserved.

using UnrealBuildTool;
using System.Collections.Generic;

[SupportedPlatforms(UnrealPlatformClass.Desktop)]
public class TestProgramTarget : TargetRules
{
	public TestProgramTarget(TargetInfo Target) : base(Target)
	{
		Type = TargetType.Program;
		LinkType = TargetLinkType.Monolithic;
		LaunchModuleName = "TestProgram";

		// Lean and mean
		bBuildDeveloperTools = false;

		// Never use malloc profiling in Unreal Header Tool.  We set this because often UHT is compiled right before the engine
		// automatically by Unreal Build Tool, but if bUseMallocProfiler is defined, UHT can operate incorrectly.
		bUseMallocProfiler = false;

		// Editor-only data, however, is needed
		bBuildWithEditorOnlyData = true;

		// Currently this app is not linking against the engine, so we'll compile out references from Core to the rest of the engine
		bCompileAgainstEngine = false;
		bCompileAgainstCoreUObject = false;
		bCompileAgainstApplicationCore = false;

		// UnrealHeaderTool is a console application, not a Windows app (sets entry point to main(), instead of WinMain())
		bIsBuildingConsoleApplication = true;
	}
}
```

`TestProgram.Build.cs`

```csharp
// Copyright Epic Games, Inc. All Rights Reserved.

using UnrealBuildTool;

public class TestProgram : ModuleRules
{
	public TestProgram(ReadOnlyTargetRules Target) : base(Target)
	{
		PublicIncludePaths.Add("Runtime/Launch/Public");

		PrivateIncludePaths.Add("Runtime/Launch/Private");		// For LaunchEngineLoop.cpp include

		PrivateDependencyModuleNames.Add("Core");
		PrivateDependencyModuleNames.Add("Projects");
	}
}
```

`TestProgram.h`

```cpp
// Copyright Epic Games, Inc. All Rights Reserved.

#pragma once

#include "CoreMinimal.h"
```

`TestProgram.cpp`

```cpp
// Copyright Epic Games, Inc. All Rights Reserved.


#include "TestProgram.h"

#include "RequiredProgramMainCPPInclude.h"

DEFINE_LOG_CATEGORY_STATIC(LogTestProgram, Log, All);

IMPLEMENT_APPLICATION(TestProgram, "TestProgram");

INT32_MAIN_INT32_ARGC_TCHAR_ARGV()
{
	GEngineLoop.PreInit(ArgC, ArgV);
	UE_LOG(LogTestProgram, Display, TEXT("TestProgram"));
	FEngineLoop::AppExit();
	return 0;
}
```

`Resources/Windows` 文件夹下的图标改成 `TestProgram.ico`

点击引擎目录下的 `GenerateProjectFiles.bat` 重新生成项目，运行效果如下：

![](/res/img/post/08-ue-daxiangwuxing-05/14-3-3.png)

### 14.4.2 增加模块引用

`TestProgram.Build.cs`

```csharp
// Copyright Epic Games, Inc. All Rights Reserved.

using UnrealBuildTool;

public class TestProgram : ModuleRules
{
	public TestProgram(ReadOnlyTargetRules Target) : base(Target)
	{
		PublicIncludePaths.Add("Runtime/Launch/Public");

		PrivateIncludePaths.Add("Runtime/Launch/Private");      // For LaunchEngineLoop.cpp include

		PrivateDependencyModuleNames.AddRange(new string[] { "Core", "Projects", "Slate", "SlateCore", "StandaloneRenderer" });
	}
}
```

<br>

### 14.4.3 设置入口点 WinMain 并添加 LOCTEXT_NAMESPACE 定义

为了能够创建一个 `Windows` 窗口，我们必须要让引擎的入口点设置为 `WinMain`。在 `BlankProgram.cpp` 最开头添加上一行 `LOCTEXT_NAMESPACE` 定义，内容随意，这是为了适配虚幻引擎的本地化机制。

`TestProgram.cpp`

```cpp
// Copyright Epic Games, Inc. All Rights Reserved.


#include "TestProgram.h"

#include "RequiredProgramMainCPPInclude.h"

#define LOCTEXT_NAMESPACE "TestProgram"

DEFINE_LOG_CATEGORY_STATIC(LogTestProgram, Log, All);

IMPLEMENT_APPLICATION(TestProgram, "TestProgram");

//INT32_MAIN_INT32_ARGC_TCHAR_ARGV()
int WINAPI WinMain(
	_In_ HINSTANCE hInInstance,
	_In_opt_ HINSTANCE hPrevInstance,
	_In_ LPSTR,
	_In_ int nCmdShow)
{
	GEngineLoop.PreInit(ArgC, ArgV);
	UE_LOG(LogTestProgram, Display, TEXT("TestProgram"));
	FEngineLoop::AppExit();
	return 0;
}
```

### 14.4.4 添加 SlateStandaloneApplication

此时 `GEngineLoop` 的初始化会缺少参数，因此需要略微修改，之后添加以下代码以启动 `SlateApplication`，并进入消息循环：

> 需要添加 `#include "StandaloneRenderer.h"`

`TestProgram.cpp`

```cpp
// Copyright Epic Games, Inc. All Rights Reserved.


#include "TestProgram.h"

#include "RequiredProgramMainCPPInclude.h"

#include "StandaloneRenderer.h"

#define LOCTEXT_NAMESPACE "TestProgram"

DEFINE_LOG_CATEGORY_STATIC(LogTestProgram, Log, All);

IMPLEMENT_APPLICATION(TestProgram, "TestProgram");

//INT32_MAIN_INT32_ARGC_TCHAR_ARGV()
int WINAPI WinMain(
	_In_ HINSTANCE hInInstance,
	_In_opt_ HINSTANCE hPrevInstance,
	_In_ LPSTR,
	_In_ int nCmdShow)
{
	//GEngineLoop.PreInit(ArgC, ArgV);
	//UE_LOG(LogTestProgram, Display, TEXT("TestProgram"));

	GEngineLoop.PreInit(GetCommandLineW());
    FSlateApplication::InitializeAsStandaloneApplication(GetStandardStandaloneRenderer());
	while (!GIsRequestingExit)
	{
		FSlateApplication::Get().Tick();
		FSlateApplication::Get().PumpMessages();
	}

	// 防止退出时空引用报错
    FSlateApplication::Shutdown();

	return 0;
}
```

<br>

### 14.4.5 链接 CoreUObject 和 ApplicationCore

由于 `Slate` 需要 `CoreUObject` 模块中的一些定义，因此需要在 `.Target.cs` 文件中把链接选项设置为 `true`，同时顺便也把编译为命令行应用程序设置为 `false`，即：

`TestProgram.Target.cs`

```csharp
bCompileAgainstCoreUObject = true;
bCompileAgainstApplicationCore = true;

// UnrealHeaderTool is a console application, not a Windows app (sets entry point to main(), instead of WinMain())
bIsBuildingConsoleApplication = false;
```

<br>

### 14.4.6 添加一个 Window

为了让我们的 `Slate` 系统打开一个窗口，我们需要手动增加一个 `Window`。在 `FSlateApplication` 初始化后，添加以下代码：

```cpp
TSharedPtr<SWindow> MainWindow = SNew(SWindow).ClientSize(FVector2D(800, 200));
FSlateApplication::Get().AddWindow(MainWindow.ToSharedRef());
```

这些操作做完后，点击引擎目录下的 `GenerateProjectFiles.bat` 重新生成项目

<br>

### 14.4.7 最终代码

如果一切顺利，最终完成的代码应该是这样：

`TestProgram.cpp`

```cpp
// Copyright Epic Games, Inc. All Rights Reserved.


#include "TestProgram.h"

#include "RequiredProgramMainCPPInclude.h"

#include "StandaloneRenderer.h"

#define LOCTEXT_NAMESPACE "TestProgram"

DEFINE_LOG_CATEGORY_STATIC(LogTestProgram, Log, All);

IMPLEMENT_APPLICATION(TestProgram, "TestProgram");

//INT32_MAIN_INT32_ARGC_TCHAR_ARGV()
int WINAPI WinMain(
	_In_ HINSTANCE hInInstance,
	_In_opt_ HINSTANCE hPrevInstance,
	_In_ LPSTR,
	_In_ int nCmdShow)
{
	GEngineLoop.PreInit(GetCommandLineW());
	FSlateApplication::InitializeAsStandaloneApplication(GetStandardStandaloneRenderer());

	auto MainWindow = SNew(SWindow).ClientSize(FVector2D(800, 200));
	FSlateApplication::Get().AddWindow(MainWindow);

	while (!GIsRequestingExit)
	{
		FSlateApplication::Get().Tick();
		FSlateApplication::Get().PumpMessages();
	}

	// 防止退出时空引用报错
	FSlateApplication::Shutdown();

	return 0;
}
```

<br>

### 14.4.8 最终效果

![](/res/img/post/08-ue-daxiangwuxing-05/14-4-1.png)

<br>
<br>

# 十五、插件开发

## 15.1 简介

插件系统作为虚幻引擎的重要组成部分，被赋予了非常多的重任。不严谨地说，**“插件”** 扛起了虚幻引擎的三分之一边天。在我们的日常使用中，几乎一直在跟插件打交道。比如用于版本控制的 Git、Subversion、P4 插件；用于 VR 的 GearVR、Oculus、SteamVR 插件；用于移动平台多媒体播放的各类 Media Player 插件等。截至目前为止，虚幻引擎官方自带的插件已经有许多！其中有些是虚幻官方开发的，而有些则是第三方公司、团队开发后被纳入官方主版本库的。

<br>

## 15.2 创建插件

### 15.2.1 引擎插件与项目插件

虚幻引擎4 的插件有两种：引擎插件和项目插件。引擎插件放在 `Engine\Plugins` 目录下，而项目插件则放在项目根目录 `( `.uproject` 同级目录）`下的 `Plugins` 中。那么引擎插件与项目插件有什么区别呢？用一种不太严谨的说法就是，把项目插件放到引擎插件目录下它就成了引擎插件。同理，引擎插件放到项目里它就成了项目插件。另外，引擎插件有一点特别的是它可以被加载到使用此引擎的所有项目里，而项目插件只能在当前项目里用。

引擎插件与项目插件除了存放的目录不相同外，其他差别不大。所以，在这里只介绍项目插件如何创建。

创建一个完整的插件需要创建插件配置文件、模块配置、创建源码目录等操作。所幸的是，虚幻引擎4 提供了插件创建工具，并提供了几种常见的插件模板，用于我们快速创建插件。

使用方法为：执行 `“Editor”` → `“Plugins”` 命令，在打开的窗口右下角单击 `“New Plugin”`，选择相应的插件模板。

<br>

### 15.2.2 插件结构

笔者使用 `Blank` `C++` 模板创建了一个名叫 `“MyBlank”` 的插件。插件工具随后生成了相应的文件，如下图所示。

![](/res/img/post/08-ue-daxiangwuxing-05/15-2-1.png)

![](/res/img/post/08-ue-daxiangwuxing-05/15-2-2.png)

下面我们来初步认识一下一个基础插件所包含的这些文件都有什么作用。

`.uplugin`

在插件目录最外层的是 `.uplugin` 文件。在 虚幻引擎4 启动的时候，会在 `Plugins` 目录里面搜索所有的 `.uplugin` 文件，每个 `.uplugin` 代表一个插件。

```cpp
{
	"FileVersion": 3,
	"Version": 1,
	"VersionName": "1.0",
	"FriendlyName": "MyBlank",
	"Description": "",
	"Category": "Other",
	"CreatedBy": "",
	"CreatedByURL": "",
	"DocsURL": "",
	"MarketplaceURL": "",
	"SupportURL": "",
	"CanContainContent": true,
	"IsBetaVersion": false,
	"IsExperimentalVersion": false,
	"Installed": false,
	"Modules": [
		{
			"Name": "MyBlank",
			"Type": "Runtime",
			"LoadingPhase": "Default"
		}
	]
}
```

以上为 `MyBlank.uplugin` 的内容。可以看出，它是一个 `.json` 格式的文件。前几条为插件的基本介绍，用于显示在插件面板。比如重要的是 “Modules” 的配置。“Modules” 即模块 ，一个插件可以包含多个模块，如果要添加新的模块，只需要在 `Modules` 里面添加即可。写法如下：

```cpp
"Modules": [
		{
			"Name": "MyBlank",
			"Type": "Runtime",
			"LoadingPhase": "Default"
		},
		{
			"Name": "another",
			"Type": "Developer",
			"LoadingPhase": "Default"
		}
	]
```

每个模块，需要配置名字、类型和加载阶段。根据需要的类型(Type)可以配置成以下类型：

1. `Runtime` 在任何情况下都会加载。
2. `RuntimeNoCommandlet` Runtime模式但不包含命令。
3. `Developer` 只在开发（Development）模式和编辑模式下加载，打包（Shipping）后不加载。
4. `Editor` 只在编辑器启动时加载。
5. `EditorNoCommandlet` Editor模式但不包含命令。
6. `Program` 独立的应用程序。

- **加载阶段 (LoadingPhase)** 用于控制此模块在什么时候加载和启动，默认为 **“Default”**，一般情况下此项不需要配置。可以用 `PreDefault`、`PostConfigIni`。`PostConfigIni` 将会让此模块在虚幻引擎关键模块加载后加载。而 `PreDefault` 则是让模块在一般模块前加载。通常情况下，这个配置只用于你的游戏模块直接引用了插件模块里的资源或者用到了插件里面的类型定义。

还有两个配置在模板里没有体现，分别是 `WhitelistPlatforms` 和 `BlacklistPlatforms`，即平台白名单和黑名单。因为 虚幻引擎4 是跨平台的，所以可以用这两个配置定义模块只能在哪些平台被加载或在哪些平台不加载。

- **Resources目录** 即资源目录，通常情况下其里面至少包含此插件的图标。

- **Source目录** 即此插件的源码目录，通常情况下，在 `Source` 目录里，每个目录代表一个模块。如刚才创建的插件，在 `Source` 目录下就只有一个名为 **“MyBlank”** 的目录，即 **“MyBlank”** 模块（模块名与插件名可以相同）。在 `MyBlank` 目录下，又包含 `MyBlank.Build.cs` 文件和 `Public`、`Private` 两个目录。通常头文件放在 `Public` 目录下，源文件放在 `Private` 目录下。

- **.Build.cs** 文件为模块的配置文件。此配置文件为 `.cs` 文件（即微软的 `C#` 语言），所以它可以支持包括 `C#` 的打印输出、变量定义、函数定义等特性。通常情况下，此配置文件在实际开发中，我们最常配置的是 `PrivateDependencyModuleNames` 和 `PublicDependencyModuleNames`，即私有依赖与公共依赖。简而言之就是，我们写的源码，如果需要引用其他模块，则需要对其进行依赖。如果源码在 `Private` 目录下，则需要添加相应的私有依赖，`Public` 亦然。

- **PCH** 文件为预编译头，通常在模块的 `Private` 目录下。关于预编译头的相关知识请自行查询。这里需要注意的是，在 虚幻引擎4 中，要求所有源文件（.cpp文件）都必须在第一行 `Include` 此模块的 `PCH` 文件中。

> **这里补充一下：UE4.27.2 创建的插件中默认已经没有 PCH 文件了
> UE官方采用了更好的方案：** 
> 👉[IWYU](https://dev.epicgames.com/documentation/en-us/unreal-engine/include-what-you-use-iwyu-for-unreal-engine-programming?application_version=5.3)

<br>

### 15.2.3 模块入口

虚幻引擎4 的模块继承自 `IModuleInterface`，在 `IModuleInterface` 中包含7个虚函数：`StartupModule`、`PreUnloadCallback`、`PostLoadCallback`、`ShutdownModule`、`SupportsDynamicReloading`、`SupportsAutomaticShutdown`、`IsGameModule`，其中 `StartupModule` 为模块入口。

修改项目 `.uproject` 文件，启用插件模块：

```txt
{
	"FileVersion": 3,
	"EngineAssociation": "{89006943-4124-9EEB-FB66-F38347A3DD1F}",
	"Category": "",
	"Description": "",
	"Modules": [
		{
			"Name": "UERenderTest",
			"Type": "Runtime",
			"LoadingPhase": "Default"
		}
	],
	"Plugins": [
		{
			"Name": "ShaderTest",
			"Enabled": true
		}
	]
}
```

还可以设置对不同平台的支持：

```txt
{
    "Name": "xxx",
    "Enabled": false,
    "SupportedTargetPlatforms": [
        "Lumin",
        "Mac",
        "Win64"
    ]
},
```

<br>
<br>

## 15.3 基于 Slate 的界面

### 15.3.1 Slate 简介

`Slate` 是虚幻引擎中的自定义用户界面系统的名称。目前的编辑器中的界面大部分都是使用 `Slate` 来构建的。同时 `Slate` 也可以作为我们游戏开发时的 `UI` 框架（除了 `UMG` 以外又多了一项选择）。以及虚幻引擎4 的 `UMG` 系统也是由 `Slate` 构建出来的。学会 `Slate` 有助于我们更深入地理解虚幻引擎4 的 `GUI` 架构。不管是用于项目开发，还是插件编写，都可以用到它。

<br>

### 15.3.2 Slate 基础概念

关于Slate的基础概念，在官方文档 👉[Slate UI框架](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/slate-user-interface-programming-framework-for-unreal-engine?application_version=5.3) 里已经有详细的解释。在这里，笔者将主要通过实例代码来讲解。希望读者在看完这些内容后，能把官方文档再详细地阅读一遍。

<br>

### 15.3.3 最基础的界面

创建最简单的 `Slate` 界面的方法是创建一个带界面的插件。因为不管如何，我们要使用 `Slate`，需要引用一些 `Slate` 模块、引用头文件等。而利用虚幻引擎4 编辑器创建插件模板，会帮我们完成这些初始化工作。在这里我们主要是关注 `Slate` 本身就可以了。

创建带界面的插件，如下图所示：

![](/res/img/post/08-ue-daxiangwuxing-05/15-3-1.png)

输入一个名字后，点击创建即可生成插件模板代码。然后切换到 `VS`，会提示解决方案有更新。点击 “全部覆盖” 即可。

编译项目后运行，在虚幻引擎4编辑器的工具栏上会多一个按钮（`即刚刚创建的插件名称`）。点击即会出现一个窗口，如下图所示：

![](/res/img/post/08-ue-daxiangwuxing-05/15-3-2.png)

> 注意文本内容：
> Add code to FStandalonePluginTestModule::OnSpawnPluginTab in StandalonePluginTest.cpp to override this window's contents
> 官方模板已经在教我们写了哈哈

至此，我们已经成功创建了一个点界面的插件。接下来，我们来简单分析一下，生成界面插件的一些过程。

首先，从目录结构上可以看出。在 `Public` 录下，生成了 4个文件。因为笔者创建的插件名叫 “StandalonePluginTest”。所以模板生成的4个文件名分别叫 `StandalonePluginTest.h`、`StandalonePluginTest.cpp`、`StandalonePluginTestCommands.h` 和 `StandalonePluginTestStyle.h`（插件名不同，这几个文件名会有相应的变化）。

- `StandalonePluginTest` 类为插件主类，插件入口与出口都在此类中。
- `StandalonePluginTestCommands` 类注册了一个 `OpenPluginWindow` 的命令。
- `StandalonePluginTestStyle` 类则是定义UI样式的类。

这里着重分析一下 `StartupModule` 入口函数。`StartupModule` 会在模块被载入时调用。

```cpp
void FStandalonePluginTestModule::StartupModule()
{
	// This code will execute after your module is loaded into memory; the exact timing is specified in the .uplugin file per-module
	
	// 初始化 UI 样式
	FStandalonePluginTestStyle::Initialize();
	FStandalonePluginTestStyle::ReloadTextures();
	// 注册命令
	FStandalonePluginTestCommands::Register();
	// 实例化一个命令，并进行行为绑定
	PluginCommands = MakeShareable(new FUICommandList);
	PluginCommands->MapAction(
		FStandalonePluginTestCommands::Get().OpenPluginWindow,
		FExecuteAction::CreateRaw(this, &FStandalonePluginTestModule::PluginButtonClicked),
		FCanExecuteAction());

	// 延迟菜单注册
	UToolMenus::RegisterStartupCallback(FSimpleMulticastDelegate::FDelegate::CreateRaw(this, &FStandalonePluginTestModule::RegisterMenus));
	// 注册一个 Tab 容器
	FGlobalTabmanager::Get()->RegisterNomadTabSpawner(StandalonePluginTestTabName, FOnSpawnTab::CreateRaw(this, &FStandalonePluginTestModule::OnSpawnPluginTab))
		.SetDisplayName(LOCTEXT("FStandalonePluginTestTabTitle", "StandalonePluginTest"))
		.SetMenuType(ETabSpawnerMenuType::Hidden);
}
```

在 `StartupModule` 里只是完成了初始化工作。那么具体流程是什么样的呢？

首先，我们在工具栏点击插件的按钮时，因为 `ToolbarExtender` 与 `PluginCommands` 关联，而 `PluginCommands` 的执行函数是 `FStandalonePluginTestModule::PluginButtonClicked`，所以，当点击工具栏上的按钮后，最终执行了 `PluginButtonClicked()` 函数。

在 `PluginButtomClicked` 里，调用了 `FGlobalTabmanager` 的 `TryInvokeTab` 函数。因为在 `StartupModule` 里，已经注册了这个 `Tab`，所以，`TryInvokeTab` 会启动这个 `Tab`。而在注册的时候，绑定了 `Tab` 生成的处理函数为 `&FStandalonePluginTestModule::OnSpawnPluginTab`。

在 `OnSpawnPluginTab` 函数内部，生成了具体的 `UI`。

```cpp
TSharedRef<SDockTab> FStandalonePluginTestModule::OnSpawnPluginTab(const FSpawnTabArgs& SpawnTabArgs)
{
	FText WidgetText = FText::Format(
		LOCTEXT("WindowWidgetText", "Add code to {0} in {1} to override this window's contents"),
		FText::FromString(TEXT("FStandalonePluginTestModule::OnSpawnPluginTab")),
		FText::FromString(TEXT("StandalonePluginTest.cpp"))
		);

	return SNew(SDockTab)
		.TabRole(ETabRole::NomadTab)
		[
			// Put your tab content here!
			SNew(SBox)
			.HAlign(HAlign_Center)
			.VAlign(VAlign_Center)
			[
				SNew(STextBlock)
				.Text(WidgetText)
			]
		];
}
```

接下来就这段生成 `UI` 的代码，我们开始正式进入 `Slate` 代码的学习。

<br>

### 15.3.4 SNew 与 SAssignNew

就像创建一个新的 `UObject` 对象是用 `NewObject()` 一样。在 `Slate` 中，创建一个新的 `UI`，有 `SNew` 与 `SAssignNew` 两种方式。

`SNew` 与 `SAssignNew` 的主要区别是返回值类型不一样。`SNew` 返回 `TSharedRef`，而 `SAssignNew` 返回 `TSharedPtr`。

> 这里书中写错了，本篇已修改，可以看下面官方注释

```cpp
/**
 * Slate widgets are constructed through SNew and SAssignNew.
 * e.g.
 *      
 *     TSharedRef<SButton> MyButton = SNew(SButton);
 *        or
 *     TSharedPtr<SButton> MyButton;
 *     SAssignNew( MyButton, SButton );
 *
 * Using SNew and SAssignNew ensures that widgets are populated
 */
```

一般我们需要存储一个 `UI` 对象，会在头文件里声明一个变量用于记录。如在 `StandalonePluginTest.h` 中添加以下一行代码：

```cpp
// 声明一个 Button 类型的变量
TSharedPtr<SButton> TestButtonPtr;
```

在 `cpp` 文件中实例化这个按钮的时候，有两种方法。

```cpp
SAssignNew(TestButtonPtr, SButton);
TestButtonPtr = SNew(SButton);
```

可能细心的读者会发现，`TestButtonPtr` 类型不是 `TSharedPtr` 吗？`SNew` 的返回值是 `TSharedRef`。类型不一样应该会报错的吧！但其实不会，在虚幻引擎4 中 `TSharedRef` 可以直接转换成 `TSharedPtr`，但这个转换不是双向的，`TSharedPtr` 转换成 `TSharedRef` 需要用到 `AsShared()` 函数。

下面来演示一下：

```cpp
SNew(SBox)
.HAlign(HAlign_Center)
.VAlign(VAlign_Center)
[
	//TestButtonPtr为TSharedRef所以要用AsShared进行转换
	TestButtonPtr->AsShared()
]
///////////////////////////////////
//////////////////////////////////
SNew(SBox)
.HAlign(HAlign_Center)
.VAlign(VAlign_Center)
[
	//SNew返回值是TSharedRef
	SNew(SButton)
]
```

<br>

### 15.3.5 Slate 控件的三种类型

在 `Slate` 中，控件分为三种类型：

1. `Leaf Widgets` 不带子槽的控件。如显示一块文本的 `STextBlock`。
2. `Panels` 子槽数量为动态的控件。如垂直排列任意数量子项，形成一些布局规则的 `SVerticalBox`。
3. `Compound Widgets` 子槽显式命名、数量固定的控件。如拥有一个名为 `Content` 的槽（包含按钮中所有控件）的 `SButton`。

在刚才的代码中，我们所演示的控件有 `SDockTab`、`SButton`、`SBox`。这三种控件都属于 `CompoundWidgets`。

观察以下这段代码的写法，会发现一个共性：这三种控制都是用中括号来指定具体的内容。

```cpp
SNew(SDockTab)
[
	SNew(SBox)
	[
		SNew(SButton)
		[
			SNew(STextBlock)
		]
	]
];
```

那么是否可以说，`Slate` 的容器添加内容都是以这种中括号的形式来添加呢？答案是不一定。我们先来看一下，这种中括号的方法是如何来实现添加子控件的。

以下内容涉及比较深入的内容，不理解也没关系。 以 `SButton` 为例，在 `SButton.h` 中：

> 以下代码截取自 UE4.27.2

```cpp
/**
* Slate's Buttons are clickable Widgets that can contain arbitrary
widgets as its Content().
*/
class SLATE_API SButton
	: public SBorder
{
public:
	SLATE_BEGIN_ARGS( SButton )
		: _Content()
		, _ButtonStyle( &FCoreStyle::Get().GetWidgetStyle< FButtonStyle >( "Button" ) )
	......
	......
	......
	{
	}

	/** Slot for this button's content (optional) */
	SLATE_DEFAULT_SLOT( FArguments, Content )
	....
	....
};
```

事实上，在 `Slate` 中，`SLATE_DEFAULT_SLOT` 指的就是那对中括号，而 `Content` 则是指中括号里面的具体内容。

在 `SButton` 构造函数中。先判断是否有设置 `_Text`，如果没设置的话就会取出 `Content` 的内容。因为 `SButton` 继承自 `SBorder`，所以调用了父类（`SBorder`）的构造函数，将中括号的内容放到了 `SBorder` 的中括号内。

`SBorder` 如何处理 `Content` 呢？答案是：

```cpp
ChildSlot
.HAlign(InArgs._HAlign)
.VAlign(InArgs._VAlign)
.Padding(InArgs._Padding)
[
	InArgs._Content.Widget
];
```

直接添加到 `ChildSlot` 里。

**解释这个过程只是为了说明一个道理：控件最终是要放到插槽里。** 在 `Slate` 系统里，容器并不是直接存储控件的，而是容器里的 `Slot`（插槽）来存放控件。插槽的数量决定这个容器是属于什么类型：没有 `Slot` 的叫 `Leaf Widgets`；`Slot` 数量不固定的叫 `Panels`；有明确的 `Slot` 的叫 `Compound Widgets`。

<br>

### 15.3.6 创建自定义控件

虽然 `Slate` 提供的控件非常多，但是很多时候我们还是需要自定义一些控件。而且不同类型的控件需要写在不同的文件里，方便管理。

自定义 `Slate` 控件需要继承 `SWidget` 或其子类。同时，建议每种控件都有独立的头文件与源文件。在虚幻引擎4 中，创建任何类都需要遵循一些规则，`Slate` 也不例外。

下面开始创建自定义控件：

![](/res/img/post/08-ue-daxiangwuxing-05/15-3-3.png)

![](/res/img/post/08-ue-daxiangwuxing-05/15-3-4.png)

![](/res/img/post/08-ue-daxiangwuxing-05/15-3-5.png)

> 以下代码以 UE4.27.2 为例

生成代码如下：

`SWidgetDemo.h`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "Widgets/SCompoundWidget.h"

/**
 * 
 */
class STANDALONEPLUGINTEST_API SWidgetDemo : public SCompoundWidget
{
public:
	SLATE_BEGIN_ARGS(SWidgetDemo)
	{}
	SLATE_END_ARGS()

	/** Constructs this widget with InArgs */
	void Construct(const FArguments& InArgs);
};
```

`SWidgetDemo.cpp`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.


#include "SWidgetDemo.h"
#include "SlateOptMacros.h"

BEGIN_SLATE_FUNCTION_BUILD_OPTIMIZATION
void SWidgetDemo::Construct(const FArguments& InArgs)
{
	/*
	ChildSlot
	[
		// Populate the widget
	];
	*/
}
END_SLATE_FUNCTION_BUILD_OPTIMIZATION

```

在这个最基础的 `Slate` 控件类中，我们选择的是继承 `SCompoundWidget`。`SCompoundWidget` 可以自定义 `Slot`。

`SLATE_BEGIN_ARGS` 和 `SLATE_END_ARGS` 为固定的控件格式。`Construct` 函数为控件构造函数，会在实例化此控件的时候执行。`SCompoundWidget` 自带一个名叫 `ChildSlot` 的子槽。我们如果要添加控件，需要添加到 `ChildSlot` 里。下面在源文件中，为 `ChildSlot` 添加一个按钮（`SButton`）试试：

`SWidgetDemo.cpp`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.


#include "SWidgetDemo.h"
#include "SlateOptMacros.h"

BEGIN_SLATE_FUNCTION_BUILD_OPTIMIZATION
void SWidgetDemo::Construct(const FArguments& InArgs)
{
	ChildSlot
	[
		// Populate the widget
		SNew(SButton)
	];
}
END_SLATE_FUNCTION_BUILD_OPTIMIZATION

```

接下来，我们把这个控件添加到插件的 `OnSpawnPluginTab` 函数中，让它显示出来，如下所示。

`StandalonePluginTest.cpp`

```cpp
#include "SWidgetDemo.h"

TSharedRef<SDockTab> FStandalonePluginTestModule::OnSpawnPluginTab(const FSpawnTabArgs& SpawnTabArgs)
{
	return SNew(SDockTab)
		.TabRole(ETabRole::NomadTab)
		[
			//实例化 SWidgetDemo 控件
			SNew(SWidgetDemo)
		];
}
```

运行后，最终效果将会是一个按钮充满整个窗口，如下图所示：

![](/res/img/post/08-ue-daxiangwuxing-05/15-3-6.png)

<br>

### 15.3.7 布局控件

`UI` 布局通常使用 `Panels` 类型的控件。常用的有 `SVerticalBox`、`SHorizontalBox`、`SOverlay`。接下来以 `SVerticalBox` 为例，演示一下垂直布局，如下图所示：

![](/res/img/post/08-ue-daxiangwuxing-05/15-3-7.png)

`SWidgetDemo.cpp`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.


#include "SWidgetDemo.h"
#include "SlateOptMacros.h"

BEGIN_SLATE_FUNCTION_BUILD_OPTIMIZATION
void SWidgetDemo::Construct(const FArguments& InArgs)
{
	ChildSlot
	[
		// Populate the widget
		SNew(SVerticalBox)
			+SVerticalBox::Slot()
			[
				SNew(SButton)
			]
			+SVerticalBox::Slot()
			[
				SNew(SButton)
			]
	];
}
END_SLATE_FUNCTION_BUILD_OPTIMIZATION

```

关于 `+` 操作符的重载可以看看这段代码：

```cpp
SLATE_SUPPORTS_SLOT(SVerticalBox::FSlot)

/**
 * Use this macro between SLATE_BEGIN_ARGS and SLATE_END_ARGS
 * in order to add support for slots.
 */
#define SLATE_SUPPORTS_SLOT( SlotType ) \
		TArray< SlotType* > Slots; \
		WidgetArgsType& operator + (SlotType& SlotToAdd) \
		{ \
			Slots.Add( &SlotToAdd ); \
			return *this; \
		}
```

在使用 `SVerticalBox` 控件时，添加新的子槽只需要 `+SVerticalBox::Slot()` 即可。

`Slot` 也可以进行嵌套，如图下所示：

![](/res/img/post/08-ue-daxiangwuxing-05/15-3-8.png)

`SWidgetDemo.cpp`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.


#include "SWidgetDemo.h"
#include "SlateOptMacros.h"

BEGIN_SLATE_FUNCTION_BUILD_OPTIMIZATION
void SWidgetDemo::Construct(const FArguments& InArgs)
{
	ChildSlot
	[
		// Populate the widget
		SNew(SVerticalBox)
			+SVerticalBox::Slot()
			[
				SNew(SButton)
			]
			+SVerticalBox::Slot()
			[
				SNew(SHorizontalBox)
				+SHorizontalBox::Slot()
				[
					SNew(SButton)
				]
				+SHorizontalBox::Slot()
				[
					SNew(SButton)
				]
				+ SHorizontalBox::Slot()
				[
					SNew(SButton)
				]
			]
	];
}
END_SLATE_FUNCTION_BUILD_OPTIMIZATION

```

<br>

### 15.3.8 控件参数与属性

每个控件都有自己的属性，不同属性影响了控件的表现形式。以 `SButton` 为例，我们可以设置它的 `Text` 属性。

```cpp
SNew(SButton).Text(LOCTEXT("Test", "测试文字"))
```

参数是如何定义及使用的呢？这需要使用 `Slate` 的 `SLATE_ATTRIBUTE` 宏。接下来的例子将演示如何使用，首先在 `SWidgetDemo.h` 中添加代码，如下所示：

```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "Widgets/SCompoundWidget.h"

/**
 * 
 */
class STANDALONEPLUGINTEST_API SWidgetDemo : public SCompoundWidget
{
public:
	SLATE_BEGIN_ARGS(SWidgetDemo)
	{}
	SLATE_ATTRIBUTE(FString, InText)
	SLATE_END_ARGS()

	/** Constructs this widget with InArgs */
	void Construct(const FArguments& InArgs);
};

```

通过 `SLATE_ATTRIBUTE` 宏，我们添加了一个 `FString` 类型的属性，名叫 `InText`。这里需要注意的是，`SLATE_ATTRIBUTE` 必须放在 `SLATE_BEGIN_ARGS` 和 `SLATE_END_ARGS` 中间。

然后，在源文件中，我们需要获取到这个属性：

`SWidgetDemo.cpp`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.


#include "SWidgetDemo.h"
#include "SlateOptMacros.h"

BEGIN_SLATE_FUNCTION_BUILD_OPTIMIZATION
void SWidgetDemo::Construct(const FArguments& InArgs)
{
	FString InString = InArgs._InText.Get();
	UE_LOG(LogTemp, Warning, TEXT("Attribute is %s"), *InString);

	ChildSlot
	[
		// Populate the widget
		SNew(SVerticalBox)
			+SVerticalBox::Slot()
			[
				SNew(SButton)
			]
			+SVerticalBox::Slot()
			[
				SNew(SHorizontalBox)
				+SHorizontalBox::Slot()
				[
					SNew(SButton)
				]
				+SHorizontalBox::Slot()
				[
					SNew(SButton)
				]
				+ SHorizontalBox::Slot()
				[
					SNew(SButton)
				]
			]
	];
}
END_SLATE_FUNCTION_BUILD_OPTIMIZATION

```

通过构造函数的 `InArgs` 参数获取到 `InText` 的值。在 `Slate` 中，所有传递的参数、属性、代理等都会存放到构造函数的 `InArgs` 里。

最后，进行测试。与一般控件设置属性一样，在 `StandalonePluginTest.cpp` 中，也就是我们创建控件的地方，修改代码为：

```cpp
SNew(SWidgetDemo).InText(FString("Hello Slate"))
```

如果一切正常，点击工具栏上的按钮后，将会在输出窗口打印 **“Attribute is Hello Slate”**。

<br>

### 15.3.9 Delegate

通常，一个控件除了显示外还需要与它进行交互。进行交互就需要有反馈，比如：点击一个登录按钮，需要产生登录的反馈（提示正在登录、执行登录代码等）。

以 `Button` 例，进行一个简单的演示

`SWidgetDemo.h`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "Widgets/SCompoundWidget.h"

/**
 * 
 */
class STANDALONEPLUGINTEST_API SWidgetDemo : public SCompoundWidget
{
public:
	SLATE_BEGIN_ARGS(SWidgetDemo)
	{}
	SLATE_ATTRIBUTE(FString, InText)
	SLATE_END_ARGS()

	/** Constructs this widget with InArgs */
	void Construct(const FArguments& InArgs);

private:
	FReply OnLoginButtonClicked();
};

```

在头文件中，只需要定义接收反馈的函数 `OnLoginButtonClicked`。需要注意的是，在 `Slate` 中，处理反馈的函数都要返回 `FReply`，告诉系统我们已经处理了这个点击事件。

`SWidgetDemo.cpp`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.


#include "SWidgetDemo.h"
#include "SlateOptMacros.h"

BEGIN_SLATE_FUNCTION_BUILD_OPTIMIZATION
void SWidgetDemo::Construct(const FArguments& InArgs)
{
	FString InString = InArgs._InText.Get();
	UE_LOG(LogTemp, Warning, TEXT("Attribute is %s"), *InString);

	ChildSlot
	[
		// Populate the widget
		SNew(SButton).OnClicked(this, &SWidgetDemo::OnLoginButtonClicked)
	];
}

FReply SWidgetDemo::OnLoginButtonClicked()
{
	UE_LOG(LogTemp, Warning, TEXT("On Login Button Clicked!!!"));
	return FReply::Handled();
}

END_SLATE_FUNCTION_BUILD_OPTIMIZATION

```

运行后点击按钮结果如下：

![](/res/img/post/08-ue-daxiangwuxing-05/15-3-9.png)


在构造函数中，实例化了一个 `SButton`。然后，设置它的 `OnClicked`，在  `OnClicked` 里传入 `OnLoginButtonClicked` 函数，这是一般 `C++` 回调函数的写法。`OnClicked` 是什么呢？在 `SButton.h` 里是这么定义的：

```cpp
/** Called when the button is clicked */
SLATE_EVENT( FOnClicked, OnClicked )
```

在 `Slate` 的宏，一般遵循( 类型名，变量名 )的规则。所以 `FOnClicked` 是一个类型，`OnClicked` 则是具体名字。

`SLATE_EVENT` 专门用于处理这种事件反馈（或者说 “回调” 或 “代理”）。接下来将以一个登录窗口的例子演示如何自定义 `SlateEvent`。

作为演示，先创建一个新的类 —— `SEventTest`

`SEventTest.h`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "Widgets/SCompoundWidget.h"

DECLARE_DELEGATE_TwoParams(FLoginDelegate, FString, FString)

/**
 * 
 */
class STANDALONEPLUGINTEST_API SEventTest : public SCompoundWidget
{
public:
	SLATE_BEGIN_ARGS(SEventTest)
	{}
	SLATE_EVENT(FLoginDelegate, OnStartLogin)
	SLATE_END_ARGS()

	/** Constructs this widget with InArgs */
	void Construct(const FArguments& InArgs);

private:
	FReply OnLoginButtonClicked();

	FLoginDelegate OnLoginDelegate;
	TSharedPtr<SButton> LoginButtonPtr;
	TSharedPtr<SEditableTextBox> UserNamePtr;
	TSharedPtr<SEditableTextBox> PasswordPtr;
};

```

`DECLARE_DELEGATE_TwoParams` 即自定义一个代理，并指明此代理将会有两个参数，类型为 `FString`。

`SLATE_EVENT(FLoginDelegate, OnStartLogin)` 即暴露一个接口，让 `SEventTest` 在实例化（`SNew` 或 `SAssignNew`）的时候，可以传入一个回调方法。

**FLoginDelegate**：`OnLoginDelegate` 是用于存储此回调方法。因为 `SLATE_EVENT` 将回调函数传入后，如果不进行存储，那么在构造函数结束后，将无法获取到此函数。因此，在构造函数里。我们就需要将 `OnLoginDelegate` 赋值。

`SEventTest.cpp`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.


#include "SEventTest.h"
#include "SlateOptMacros.h"

BEGIN_SLATE_FUNCTION_BUILD_OPTIMIZATION

// 文本本地化
#define LOCTEXT_NAMESPACE "SEventTest"

void SEventTest::Construct(const FArguments& InArgs)
{
	OnLoginDelegate = InArgs._OnStartLogin;

	ChildSlot.Padding(50, 50, 50, 50)
	[
		// Populate the widget
		SNew(SVerticalBox)
		+ SVerticalBox::Slot().AutoHeight()
		[
			SAssignNew(UserNamePtr, SEditableTextBox)
				.HintText(LOCTEXT("Username_Hint", "Please enter your account number"))
		]
		+ SVerticalBox::Slot().AutoHeight()
		[
			SAssignNew(PasswordPtr, SEditableTextBox)
				.IsPassword(true)
				.HintText(LOCTEXT("Password_Hint", "Please enter your password"))
		]
		+ SVerticalBox::Slot().AutoHeight()
		[
			SAssignNew(LoginButtonPtr, SButton)
				.OnClicked(this, &SEventTest::OnLoginButtonClicked)
				.Text(LOCTEXT("Login", "Login"))
		]
	];
}

FReply SEventTest::OnLoginButtonClicked()
{
	FString usn = UserNamePtr->GetText().ToString();
	FString pwd = PasswordPtr->GetText().ToString();
	OnLoginDelegate.ExecuteIfBound(usn, pwd);
	return FReply::Handled();
}

#undef LOCTEXT_NAMESPACE

END_SLATE_FUNCTION_BUILD_OPTIMIZATION

```

首先来看构造函数。第一行，从 `InArgs` 里面取出了 `OnStartLogin`，`OnStartLogin` 就是传入的回调函数。然后将它赋值给了 `OnLoginDelegate`。接下来，我们就可以在任何时候调用 `OnLoginDelegate`，因为它是一个类方法。我们可以在此类的任何地方调用它。

下面是界面布局。在一个 `SVerticalBox` 中放了 3个 `Slot`，前两个用于给用户输入账号密码，最后一个是登录按钮。然后绑定了点击登录按钮后，要执行函数。

`OnLoginButtonClicked` 函数将会在用户点击 “登录” 按钮后执行。在函数内部，我们首先获取到用户输入的账号与密码。然后，调用 `OnLoginDelegate` 的 `ExecuteIfBound` 函数将账号密码输出出去。这里需要大家知道的是，`Delegate` 的调用除了 `ExecuteIfBound` 还有 `Execute`。用 `ExecuteIfBound` 是因为有可能在 `SWidgetDemo` 实例化的时候，并没传入回调函数。这时候调用 `Execute` 就会发生错误。

那么，类的定义与实现就已经完成了。接下来，进行实际测试。

回到 `SWidgetDemo` 里。将 `SEventTest` 添加到构造函数里，让它显示出来。

`SWidgetDemo.h`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "Widgets/SCompoundWidget.h"

DECLARE_DELEGATE_TwoParams(FLoginDelegate, FString, FString)

/**
 * 
 */
class STANDALONEPLUGINTEST_API SWidgetDemo : public SCompoundWidget
{
public:
	SLATE_BEGIN_ARGS(SWidgetDemo)
	{}
	SLATE_END_ARGS()

	/** Constructs this widget with InArgs */
	void Construct(const FArguments& InArgs);

private:
	// 登录时的回调函数
	void OnLogin(FString usn, FString pwd);
};

```

`SWidgetDemo.cpp`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.


#include "SWidgetDemo.h"
#include "SlateOptMacros.h"
#include "SEventTest.h"

BEGIN_SLATE_FUNCTION_BUILD_OPTIMIZATION

// 文本本地化
#define LOCTEXT_NAMESPACE "SWidgetDemo"

void SWidgetDemo::Construct(const FArguments& InArgs)
{
	ChildSlot
	[
		// Populate the widget
		SNew(SEventTest).OnStartLogin(this, &SWidgetDemo::OnLogin)
	];
}

void SWidgetDemo::OnLogin(FString usn, FString pwd)
{
	UE_LOG(LogTemp, Warning, TEXT("Start Login %s - %s"), *usn, *pwd);
}

#undef LOCTEXT_NAMESPACE

END_SLATE_FUNCTION_BUILD_OPTIMIZATION

```

在 `SWidgetDemo.h` 中，只是添加了一个回调函数，用于进行测试。

在 `SWidgetDemo.cpp` 里的构造函数中，通过 `SNew` 的方式，实例化了 `SEventTest`。然后，通过 `.OnStartLogin()` 的方式，将OnLogin函数传入 `SEventTest` 内部。这样，就可以在 `SEventTest` 内部调用 `SWidgetDemo` 的 `OnLogin` 函数。

最后，记得 `include"SEventTest.h"`。登录界面如下图所示：

![](/res/img/post/08-ue-daxiangwuxing-05/15-3-10.png)

![](/res/img/post/08-ue-daxiangwuxing-05/15-3-11.png)

<br>
<br>

### 15.3.10 自定义皮肤

俗话说：“人靠衣装，佛靠金装”。同样的，做软件开发，`UI` 的样式也是非常重要的一环。在之前的内容中，我们都是使用虚幻引擎4 默认的 `UI` 样式。现在，我们将一起来研究自定义皮肤样式。

在本节中，将继续使用上一节的 “登录界面” 的代码。为它进行一些美化。在写代码前，我们先来看一下最终效果，如下图所示。

![](/res/img/post/08-ue-daxiangwuxing-05/15-3-13.png)

在上图中，首先是添加了一张图片 ，然后，按钮和输入框的背景图片被替换了，并且按钮中显示的字体及大小和颜色也进行了更改。

**资源准备**：通常情况下，插件所用到的资源都会放在 `Resources` 目录下。在本例中，需要准备一张大图、一个字体文件，以及三种按钮状态的图片，如下图所示。

![](/res/img/post/08-ue-daxiangwuxing-05/15-3-12.png)

**皮肤定义**：在插件模板中，提供了一个非常方便的类用于自定义皮肤样式 —— `StandalonePluginTestStyle`。要增加新的皮肤样式只需要在 `StandalonePluginTestStyle.cpp` 中的 `Create` 函数中添加相应代码即可。如下：

`StandalonePluginTestStyle.cpp`

```cpp
TSharedRef< FSlateStyleSet > FStandalonePluginTestStyle::Create()
{
	TSharedRef< FSlateStyleSet > Style = MakeShareable(new FSlateStyleSet("StandalonePluginTestStyle"));
	Style->SetContentRoot(IPluginManager::Get().FindPlugin("StandalonePluginTest")->GetBaseDir() / TEXT("Resources"));

	Style->Set("StandalonePluginTest.OpenPluginWindow", new IMAGE_BRUSH(TEXT("ButtonIcon_40x"), Icon40x40));
	Style->Set("UI.TopIcon", new IMAGE_BRUSH(TEXT("TopIcon"), FVector2D(256 * 3, 144 * 3)));

	FMargin Button1Margin(2.0 / 10.0f, 2.0 / 10.0f, 2.0 / 10.0f, 2.0 / 10.0f);
	Style->Set("UI.BUtton1", FButtonStyle()
		.SetNormal(BOX_BRUSH("Button_N", Button1Margin))
		.SetHovered(BOX_BRUSH("Button_H", Button1Margin))
		.SetPressed(BOX_BRUSH("Button_H", Button1Margin))
		.SetDisabled(BOX_BRUSH("Button_D", Button1Margin))
	);
	Style->Set("UI.InputLineText", FEditableTextBoxStyle()
		.SetBackgroundImageNormal(BOX_BRUSH("Button_N", Button1Margin))
		.SetBackgroundImageFocused(BOX_BRUSH("Button_H", Button1Margin))
	);
	Style->Set("UI.AdobeHS_12", FSlateFontInfo("D:/Dev/UE/SourceCode/Project/UE4.27.2/Blank/Plugins/StandalonePluginTest/Resources/AdobeHeitiStd-Regular.otf", 12));

	return Style;
}
```

创建一个新的皮肤样式使用 `Stype->Set` 的方式添加。其中第一个参数为样式的名称，第二个参数指定具体的样式。在图片创建中，我们使用了 `IMAGE_BRUSH` 宏。此宏在 `StandalonePluginTestStyle.cpp` 已经定义，除此之外，还有几个宏用于快速创建样式：

1. `IMAGE_BRUSH` 图片。
2. `BOX_BRUSH` 九宫格缩放样式。
3. `BORDER_BRUSH` 九宫格缩放样式，但不保留中间部分。
4. `TTF_FONTTTF` 字体。
5. `OTF_FONTOTF` 字体。

注：关于 “九宫格缩放” 请读者自行查找相关资料。

皮肤样式定义完成后，使用起来就非常简单了。引用 `StandalonePluginTestStyle.h` 后，只需要在对应的控件中，设置其相应的属性即可。

由于不同的控件设置皮肤样式不同，并且一个控件也可能有多个属性用于设置样式。所以，具体使用哪个属性来设置样式，需要打开对应控件的头文件，从头文件中的 `SLATE_ATTRIBUTE` 查找。因为虚幻引擎4 源码的注释非常完善，所以找起来也不难。

在本例中，使用皮肤样式，用到了两种方法。分别是 `StandalonePluginTestStyle::Get().GetBrush("style_name")` 与 `&StandalonePluginTestStyle::Get().GetWidgetStyle<FXXXStyle>("style_name")`。

`SEventTest.cpp`

```cpp
#include "StandalonePluginTestStyle.h"

void SEventTest::Construct(const FArguments& InArgs)
{
	OnLoginDelegate = InArgs._OnStartLogin;

	ChildSlot.Padding(20, 20, 20, 20)
	[
		// Populate the widget
		SNew(SVerticalBox)
		+ SVerticalBox::Slot().VAlign(VAlign_Top)
		[
			SNew(SHorizontalBox)
			+SHorizontalBox::Slot().HAlign(HAlign_Center)
			[
				SNew(SImage).Image(FStandalonePluginTestStyle::Get().GetBrush("UI.TopIcon"))
			]
		]
		+ SVerticalBox::Slot().AutoHeight().Padding(0, 30, 0, 10)
		[
			SAssignNew(UserNamePtr, SEditableTextBox)
				.Padding(FMargin(5, 3, 5, 3))
				.Style(&FStandalonePluginTestStyle::Get().GetWidgetStyle<FEditableTextBoxStyle>("UI.InputLineText"))
				.HintText(LOCTEXT("Username_Hint", "Please enter your account number"))
		]
		+ SVerticalBox::Slot().AutoHeight().Padding(0, 10, 0, 10)
		[
			SAssignNew(PasswordPtr, SEditableTextBox)
				.Padding(FMargin(5, 3, 5, 3))
				.Style(&FStandalonePluginTestStyle::Get().GetWidgetStyle<FEditableTextBoxStyle>("UI.InputLineText"))
				.IsPassword(true)
				.HintText(LOCTEXT("Password_Hint", "Please enter your password"))
		]
		+ SVerticalBox::Slot().AutoHeight().Padding(0, 10, 0, 10)
		[
			SAssignNew(LoginButtonPtr, SButton)
				.OnClicked(this, &SEventTest::OnLoginButtonClicked)
				.HAlign(HAlign_Center)
				.ContentPadding(FMargin(0, 10, 0, 10))
				.ButtonStyle(&FStandalonePluginTestStyle::Get().GetWidgetStyle<FButtonStyle>("UI.BUtton1"))
				[
					SNew(STextBlock)
					.ColorAndOpacity(FLinearColor(0.0f, 0.0f, 0.0f))
					.Font(FStandalonePluginTestStyle::Get().GetFontStyle("UI.AdobeHS_12"))
					.Text(LOCTEXT("Login", "Login"))
				]
		]
	];
}
```

> 注意图片是 png

```cpp
#define IMAGE_BRUSH( RelativePath, ... ) FSlateImageBrush( Style->RootToContentDir( RelativePath, TEXT(".png") ), __VA_ARGS__ )
```

最终效果：

![](/res/img/post/08-ue-daxiangwuxing-05/15-3-13.png)

![](/res/img/post/08-ue-daxiangwuxing-05/15-3-14.png)

<br>
<br>

### 15.3.11 图标字体

在虚幻引擎4 中，图标字体使用的是 👉[Font Awesome](https://fontawesome.com/icons)，一个完全免费的图标字体库。图标字体可能在没有设计师的配合或者没精力设计图标的情况下，用代码快速生成漂亮的图标。

具体使用方法如下：

1. 在 `Build.cs` 中添加模块引用（通常在 `PrivateDependencyModuleNames` 中添加 `EditorStyle`）。
2. 在需要使用图标字体的地方，引用相应头文件（ `EditorStyleSet.h` 和 `EditorFontGlyphs.h`）。
3. 设置字体 `Font` 为 `FontAwesome`。
4. 设置具体显示的图标。

以下示例代码将展示添加两个图标字体。

```cpp
#include "EditorStyleSet.h"
#include "EditorFontGlyphs.h"

void SWidgetDemo::Construct(const FArguments& InArgs)
{
	ChildSlot
	[
		// Populate the widget
		//SNew(SEventTest).OnStartLogin(this, &SWidgetDemo::OnLogin)
		SNew(SHorizontalBox)
		+ SHorizontalBox::Slot().AutoWidth()
		[
			SNew(STextBlock)
			.Font(FEditorStyle::Get().GetFontStyle("FontAwesome.16"))
			.Text(FEditorFontGlyphs::Android)
		]
		+ SHorizontalBox::Slot().AutoWidth()
		[
			SNew(STextBlock)
			.Font(FEditorStyle::Get().GetFontStyle("FontAwesome.16"))
			.Text(FEditorFontGlyphs::Apple)
		]
	];
}
```

`FEditorFontGlyphs` 指定具体的图标样式，因为样式繁多，读者可以从 👉[Font Awesome](https://fontawesome.com/icons) 中找到自己想要的图标。但因为版本更新速度不同，`Font Awesome` 中有些图标可能在虚幻引擎4 中没有，最终效果如下图所示：

![](/res/img/post/08-ue-daxiangwuxing-05/15-3-15.png)

<br>
<br>

### 15.3.12 组件继承

继承是实现代码重用、扩展功能的重要手段。在 `Slate` 的 `UI` 界面的实际开发中，如果用虚幻引擎4 默认的组件，则需要设置大量的属性，代码量偏多。而用组件继承的方式，可以有效降低代码量，让代码结构更清晰。同时，继承也大量用于扩展已有组件的功能。

接下来，笔者将用一个最简单的继承 `SButton` 的例子来讲解在 `Slate` 中如何实现组件继承。

首先 ，新建一个类名叫 SMyButton。具体代码如下：

`SMyButton.h`

```cpp
#pragma once

#include "CoreMinimal.h"
#include "Widgets/Input/SButton.h"

/**
 *
 */
class SMyButton : public SButton
{
public:
	SLATE_BEGIN_ARGS(SMyButton)
	{}
	SLATE_ATTRIBUTE(FText, Icon)
	SLATE_ATTRIBUTE(FText, Text)
	SLATE_END_ARGS()

	/** Constructs this widget with InArgs */
	void Construct(const FArguments& InArgs);
};
```

在 `SMyButton` 头文件中，定义此类继承 `SButton`。然后，通过 `SLATE_ATTRIBUTE` 暴露两个属性接口，这里想实现的结果是上一节学习的图标字体 。创建一个 `SMyButton` 的时候，只需要传入两个 `FText` 参数就能创建一个带小图标的按钮。

`SMyButton.cpp`

```cpp
#include "SMyButton.h"
#include "EditorStyleSet.h"
#include "EditorFontGlyphs.h"
#include "StandalonePluginTestStyle.h"

#define LOCTEXT_NAMESPACE "SMyButton"

void SMyButton::Construct(const FArguments& InArgs)
{
	SButton::Construct(SButton::FArguments()
		.ContentPadding(FMargin(10, 5, 10, 5))
		.ButtonStyle(&FStandalonePluginTestStyle::Get().GetWidgetStyle<FButtonStyle>("UI.Button1"))
		.Content()
		[
			SNew(SHorizontalBox)
			+ SHorizontalBox::Slot().AutoWidth().Padding(5, 0, 5, 0)
			[
				SNew(STextBlock)
				.Font(FEditorStyle::Get().GetFontStyle("FontAwesome.16"))
				.Text(InArgs._Icon)
			]
			+ SHorizontalBox::Slot().AutoWidth().Padding(5, 0, 5, 0)
			[
				SNew(STextBlock)
				.Font(FEditorStyle::Get().GetFontStyle("UI.AdobeHS_12"))
				.Text(InArgs._Text)
			]
		]
	);
}

#undef LOCTEXT_NAMESPACE
```

在 `SMyButton` 构建函数中，调用父类 (`SButton`) 构造函数即完成继承。在 `SButton` 构造函数中，我们传入一些参数来对 `SButton` 进行订制化。如 `ContentPadding` 指定 `Button` 内部元素的外边距，`ButtonStyle` 指定按钮样式。`Content` 指定按钮内部的内容是什么？在 `Content` 内部，实例化了一个 `SHorizontalBox` 用于进行横向排列元素。这里排列了两个 `STextBlock`，第一个 `STextBlock` 设置字体为 `FontAwesome`，就是用于创建图标字体。从 `InArgs` 里获取 `Text` 的内容，即头文件中 `SLATE_ATTRIBUTE` 的内容。

`SWidgetDemo.cpp`

```cpp
#include "SMyButton.h"

void SWidgetDemo::Construct(const FArguments& InArgs)
{
	ChildSlot
	[
		// Populate the widget
		SNew(SVerticalBox)
		+ SVerticalBox::Slot().VAlign(VAlign_Top)
		[
			SNew(SHorizontalBox)
			+ SHorizontalBox::Slot().HAlign(HAlign_Center)
			[
				SNew(SMyButton)
				.Icon(FEditorFontGlyphs::Android)
				.Text(LOCTEXT("Android", "Android"))
			]
		]
	];
}
```

最终，使用的时候，我们的代码将会变得非常简洁。最终效果如下图所示：

![](/res/img/post/08-ue-daxiangwuxing-05/15-3-16.png)

<br>
<br>

### 15.4.13 动态控制Slot

相信大家学习到现在，应该知道 `Slot` 的属性决定一个控件的显示位置、边距、对齐方式等表现形式。那么问题来了，如果想在运行时动态改变一个控件的显示怎么办？我们之前写的代码，`Slot` 的属性都是在代码里写好了的，后期没办法修改。我们知道，改变一个控件的方法是用 `SAssignNew`，存储控件的指针。那么 `Slot` 是否也可以用同样的方式呢？

答案是肯定的。但 `Slot` 不是用 `SAssignNew`，而是用 `Slot` 的 `Expose` 函数。具体使用如下所示：

`SWidgetDemo.h`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "Widgets/SCompoundWidget.h"

DECLARE_DELEGATE_TwoParams(FLoginDelegate, FString, FString)

/**
 * 
 */
class STANDALONEPLUGINTEST_API SWidgetDemo : public SCompoundWidget
{
public:
	SLATE_BEGIN_ARGS(SWidgetDemo)
	{}
	SLATE_END_ARGS()

	/** Constructs this widget with InArgs */
	void Construct(const FArguments& InArgs);

private:
	FReply ChangeSlotProperty(int32 type);

	SHorizontalBox::FSlot* ButtonSlot;
};

```

`SWidgetDemo.cpp`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.


#include "SWidgetDemo.h"
#include "SlateOptMacros.h"
#include "SEventTest.h"
#include "EditorStyleSet.h"
#include "EditorFontGlyphs.h"
#include "SMyButton.h"

BEGIN_SLATE_FUNCTION_BUILD_OPTIMIZATION

// 文本本地化
#define LOCTEXT_NAMESPACE "SWidgetDemo"

void SWidgetDemo::Construct(const FArguments& InArgs)
{
	ChildSlot
	[
		SNew(SVerticalBox)
			+ SVerticalBox::Slot().VAlign(VAlign_Top)
			[
				SNew(SHorizontalBox)
					+ SHorizontalBox::Slot().HAlign(HAlign_Center)
					.Expose(ButtonSlot) // 将 Slot 赋值给 ButtonSlot
					[
						SNew(SMyButton)
							.Icon(FEditorFontGlyphs::Android)
							.Text(LOCTEXT("Android", "Android"))
					]
			]
			+ SVerticalBox::Slot().AutoHeight()
			[
				SNew(SHorizontalBox)
					+ SHorizontalBox::Slot()
					[
						SNew(SButton)
							.OnClicked(this, &SWidgetDemo::ChangeSlotProperty, 0)
							.Text(LOCTEXT("0", "Left"))
					]
					+ SHorizontalBox::Slot()
					[
						SNew(SButton)
							.OnClicked(this, &SWidgetDemo::ChangeSlotProperty, 1)
							.Text(LOCTEXT("1", "Right"))
					]
					+ SHorizontalBox::Slot()
					[
						SNew(SButton)
							.OnClicked(this, &SWidgetDemo::ChangeSlotProperty, 2)
							.Text(LOCTEXT("2", "Center"))
					]
					+ SHorizontalBox::Slot()
					[
						SNew(SButton)
							.OnClicked(this, &SWidgetDemo::ChangeSlotProperty, 3)
							.Text(LOCTEXT("3", "Fill"))
					]
			]
	];

}

FReply SWidgetDemo::ChangeSlotProperty(int32 type)
{
	switch (type)
	{
	case 0:
		ButtonSlot->HAlign(HAlign_Left);
		break;
	case 1:
		ButtonSlot->HAlign(HAlign_Right);
		break;
	case 2:
		ButtonSlot->HAlign(HAlign_Center);
		break;
	case 3:
		ButtonSlot->HAlign(HAlign_Fill);
		break;
	}
	return FReply::Handled();
}

#undef LOCTEXT_NAMESPACE

END_SLATE_FUNCTION_BUILD_OPTIMIZATION

```

在上述代码中的构造函数里，首先用两个 `SVerticalBox::Slot` 将整个界面分为上下两个部分。上面部分有一个 `SHorizontalBox::Slot` 盛放一个按钮控件，并把这个 `Slot` 赋值给头文件中定义的 `ButtonSlot`。在下面部分的 `SVerticalBox::Slot` 中，放了 4个 `SHorizontalBox::Slot`，每个里面有一个按钮。点击按钮后会执行 `SWidgetDemoA::ChageSlotProperty` 函数，并传入不同的参数。通过判断参数的不同，来确定点击的是哪个按钮，如下图所示：

![](/res/img/post/08-ue-daxiangwuxing-05/15-3-17.png)

<br>
<br>

### 15.3.14 自定义容器布局

在之前的例子中，我们一般都是使用 `SCompoundWidget` 组件，通过设置属性来控制控件的布局。但除了 `SCompoundWidget` 外，还有一种容器组件（还记得之前说的三种控件类型吧）—— `Panel` 类型的组件。比如 `SVerticalBox` 就是一种垂直布局组件。但有些时候这些布局方式并不能满足我们的需求。这时候就要自己控制容器布局方式。

首先，我们来看一个自定义的容器布局演示，如下图所示：

![](/res/img/post/08-ue-daxiangwuxing-05/15-3-18.png)

![](/res/img/post/08-ue-daxiangwuxing-05/15-3-19.png)

**在上面两张图中，通过调整窗口的大小，容器中的元素布局会发生变化，简单来说，就是自适应窗口的宽度。**

在 `Slate` 中，容器布局是靠重写 `OnArrangeChildren` 函数 来完成的。

示例代码如下：

`SAutoLayout.h`

```cpp
#pragma once

#include "CoreMinimal.h"
#include "Widgets/SBoxPanel.h"

// 自动排列的容器组件。
class SAutoLayout :public SBoxPanel
{
public:
	class FSlot : public SBoxPanel::FSlot
	{
	public:
		FSlot()
			: SBoxPanel::FSlot()
		{}
		FSlot& AutoHeight()
		{
			SizeParam = FAuto();
			return *this;
		}
		FSlot& MaxHeight(const TAttribute<float>& InMaxHeight)
		{
			MaxSize = InMaxHeight;
			return *this;
		}
		FSlot& FillHeight(const TAttribute<float>& StretchCoefficient)
		{
			SizeParam = FStretch(StretchCoefficient);
			return *this;
		}
		FSlot& Padding(float Uniform)
		{
			SlotPadding = FMargin(Uniform);
			return *this;
		}
		FSlot& Padding(float Horizontal, float Vertical)
		{
			SlotPadding = FMargin(Horizontal, Vertical);
			return *this;
		}
		FSlot& Padding(float Left, float Top, float Right, float Bottom)
		{
			SlotPadding = FMargin(Left, Top, Right,
				Bottom);
			return *this;
		}
		FSlot& Padding(const TAttribute<FMargin>::FGetter& InDelegate)
		{
			SlotPadding.Bind(InDelegate);
			return *this;
		}
		FSlot& HAlign(EHorizontalAlignment InHAlignment)
		{
			HAlignment = InHAlignment;
			return *this;
		}
		FSlot& VAlign(EVerticalAlignment InVAlignment)
		{
			VAlignment = InVAlignment;
			return *this;
		}
		FSlot& Padding(TAttribute<FMargin> InPadding)
		{
			SlotPadding = InPadding;
			return *this;
		}
		FSlot& operator[](TSharedRef<SWidget> InWidget)
		{
			SBoxPanel::FSlot::operator[](InWidget);
			return *this;
		}
		FSlot& Expose(FSlot*& OutVarToInit)
		{
			OutVarToInit = this;
			return *this;
		}
	};

	static FSlot& Slot()
	{
		return *(new FSlot());
	}

public:
	SLATE_BEGIN_ARGS(SAutoLayout)
	{}
	SLATE_ATTRIBUTE(FMargin, ContentMargin)
	SLATE_SUPPORTS_SLOT(SAutoLayout::FSlot)
	SLATE_END_ARGS()

	FORCENOINLINE SAutoLayout()
		: SBoxPanel(Orient_Horizontal)
	{
	}

	virtual void OnArrangeChildren(const FGeometry& AllottedGeometry, FArrangedChildren& ArrangedChildren) const override;

public:
	void Construct(const FArguments& InArgs);

	FSlot& AddSlot()
	{
		SAutoLayout::FSlot& NewSlot = *new FSlot();
		this->Children.Add(&NewSlot);
		return NewSlot;
	}

private:
	FMargin ContentMargin;
};

```

首先，我们定义了一个继承自 `SBoxPanel` 的类，`SBoxPanel` 属于 `Panel` 类型的控件。接着在 `SAutolayout` 类中定义了一个 `FSlot` 并实现了一些常用的属性控制函数。从 `SLATE_BEGIN_ARGS` 宏开始定义 `SAutolayout` 类的属性，其中定义了一个 `ContentMargin`，用于设置子元素之间的间隙。重写 `OnArrangeChildren` 函数。

`AddSlot` 函数，用于动态向容器中添加子元素。其中，实现过程是向 `SAutoLayout` 类的 `Children`（可以理解为一个放元素的数组）添加 `Slot`。

`SAutoLayout.cpp`

```cpp
#include "SAutoLayout.h"

void SAutoLayout::Construct(const FArguments& InArgs)
{
	// 这里初始都是 0
	const int32 NumSlots = InArgs.Slots.Num();
	ContentMargin = InArgs._ContentMargin.Get();
	for (int32 SlotIndex = 0; SlotIndex < NumSlots; ++SlotIndex)
	{
		Children.Add(InArgs.Slots[SlotIndex]);
	}
}

void SAutoLayout::OnArrangeChildren(const FGeometry& AllottedGeometry, FArrangedChildren& ArrangedChildren) const
{
	// areaSize 即容器的尺寸
	FVector2D areaSize = AllottedGeometry.GetLocalSize();
	float startX = ContentMargin.Left;
	float startY = ContentMargin.Top;
	float currentMaxHeight = 0.0f;
	for (int32 ChildIndex = 0; ChildIndex < Children.Num(); ++ChildIndex)
	{
		const SBoxPanel::FSlot& CurChild = Children[ChildIndex];
		const EVisibility ChildVisibility = CurChild.GetWidget()->GetVisibility();
		// 获取元素的尺寸
		FVector2D size = CurChild.GetWidget()->GetDesiredSize();
		// Accepts 用于判断一个元素是否需要参与计算。
		if (ArrangedChildren.Accepts(ChildVisibility))
		{
			if (size.Y > currentMaxHeight)
			{
				currentMaxHeight = size.Y;
			}
			// 判断 StartX 是否超过容器的宽度。
			if (startX + size.X < areaSize.X)
			{
				ArrangedChildren.AddWidget(ChildVisibility, AllottedGeometry.MakeChild(CurChild.GetWidget(), FVector2D(startX, startY), FVector2D(size.X, size.Y)));
				startX += ContentMargin.Right;
				startX += size.X;
			}
			else
			{
				//超过宽度后，将 StartX 设置为最左边
				startX = ContentMargin.Left;
				//同时 StartY 增加一定的高度
				startY += currentMaxHeight + ContentMargin.Bottom;
				ArrangedChildren.AddWidget(ChildVisibility, AllottedGeometry.MakeChild(CurChild.GetWidget(), FVector2D(startX, startY), FVector2D(size.X, size.Y)));
				startX += size.X + ContentMargin.Right;
				currentMaxHeight = size.Y;
			}
		}
	}
}

```

在源文件中，首先在构造函数中，取出子元素数量。因为除了用 `AddSlot` 函数动态添加子元素外，也可以直接在实例化 `SAutoLayout` 的时候直接添加子元素（如 `SButton` 后面可以直接接中括号，把要添加的元素放进去），然后取出了传入的 `ContentMargin`。

`OnArrangeChildren` 会传入两个参数，其中 `AllottedGeometry` 为分配给此容器的尺寸，`ArrangedChildren` 则用于设定子元素的具体位置。

整个排列过程实现原理为：先获取容器的尺寸，然后定义 `StartX` 与 `StartY`，都默认为 0（表示左上角），然后循环获取所有的子元素，并得到子元素的尺寸。每添加一个子元素，则 `StartX` 会加上子元素的宽度，这样，`StartX` 会越来越大。当 `StartX` 超过容器的宽度后，将 `StartX` 归 0（这样，超始位置又回到了最左边）。然后，将 `StartY` 增加子元素的高度。通过这样的方式，实现一个从左到右，从上到下的子元素排列。

最后，使用此容器只需要实例化它，然后通过 `AddSlot` 的方式添加子元素即可，如下所示：

`SWidgetDemo.h`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "Widgets/SCompoundWidget.h"

class SAutoLayout;

DECLARE_DELEGATE_TwoParams(FLoginDelegate, FString, FString)

/**
 * 
 */
class STANDALONEPLUGINTEST_API SWidgetDemo : public SCompoundWidget
{
public:
	SLATE_BEGIN_ARGS(SWidgetDemo)
	{}
	SLATE_END_ARGS()

	/** Constructs this widget with InArgs */
	void Construct(const FArguments& InArgs);

private:
	TSharedPtr<SAutoLayout> LayoutPtr;
};

```

`SWidgetDemo.cpp`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.


#include "SWidgetDemo.h"
#include "SlateOptMacros.h"
#include "SAutoLayout.h"

BEGIN_SLATE_FUNCTION_BUILD_OPTIMIZATION

// 文本本地化
#define LOCTEXT_NAMESPACE "SWidgetDemo"

void SWidgetDemo::Construct(const FArguments& InArgs)
{
	ChildSlot
	[
		SAssignNew(LayoutPtr, SAutoLayout)
	];
	for (int32 i = 10; i < 80; i++)
	{
		LayoutPtr->AddSlot()
		[
			SNew(SButton)
			.Text(FText::FromString(FString::FromInt(i)))
		];
	}

}

#undef LOCTEXT_NAMESPACE

END_SLATE_FUNCTION_BUILD_OPTIMIZATION

```

> 上面的代码可以调式跟断点走走，不难理解

<br>
<br>

## 15.4 UMG 扩展

> 这块内容打算后面再另写文章，留个坑

<br>
<br>

## 15.5 蓝图扩展

### 15.5.1 蓝图函数库扩展

扩展蓝图节点，最简单的应该是基于 `UBlueprint Function Library` 的扩展。也就是我们一般称的 **“蓝图函数库”**。蓝图函数库的主要作用是把一些逻辑封装成一个单独的节点，增加节点的复用性。**且蓝图函数库里面的函数是 “全局” 函数，也就是说，我们可以在任何地方使用蓝图函数库里面的节点。**

在编辑器里，我们一般直接创建一个 `Blueprint Function Library`（如下图所示），然后打开蓝图节点编辑器，开发自己的蓝图函数库。

![](/res/img/post/08-ue-daxiangwuxing-05/15-5-1.png)

![](/res/img/post/08-ue-daxiangwuxing-05/15-5-2.png)

![](/res/img/post/08-ue-daxiangwuxing-05/15-5-3.png)

![](/res/img/post/08-ue-daxiangwuxing-05/15-5-4.png)

![](/res/img/post/08-ue-daxiangwuxing-05/15-5-5.png)

而在 `C++` 里，我们需要创建自己的类，并继承 `UBlueprintFunctionLibrary`。然后把你要封装的代码写成静态函数即可。

![](/res/img/post/08-ue-daxiangwuxing-05/15-5-6.png)

![](/res/img/post/08-ue-daxiangwuxing-05/15-5-7.png)

**这里注意，我们将这个类创建在插件 `StandalonePluginTest` 中，而这个插件的 `StandalonePluginTest.uplugin` 中设置的是 `"Type": "Editor"` 并不是运行时，所以下面创建的蓝图函数并不能被调用，可以试试。**

`MyTestFuncLib.h`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "Kismet/BlueprintFunctionLibrary.h"
#include "MyTestFuncLib.generated.h"

/**
 * 
 */
UCLASS()
class STANDALONEPLUGINTEST_API UMyTestFuncLib : public UBlueprintFunctionLibrary
{
	GENERATED_BODY()
	
public:
	UFUNCTION(BlueprintCallable, Category = "AnnihilateSword|TestFuncLib")
	static FString SayHello();
};

```

`MyTestFuncLib.cpp`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.


#include "MyTestFuncLib.h"

FString UMyTestFuncLib::SayHello()
{
    return FString("AnnihilateSword!!!");
}

```

**下面演示正确做法：**

![](/res/img/post/08-ue-daxiangwuxing-05/15-5-8.png)

![](/res/img/post/08-ue-daxiangwuxing-05/15-5-9.png)

**这里将继承的蓝图函数库类创建在 `MyBlank` 插件下，注意是 `Runtime` 类型：**

![](/res/img/post/08-ue-daxiangwuxing-05/15-5-10.png)

![](/res/img/post/08-ue-daxiangwuxing-05/15-5-11.png)

![](/res/img/post/08-ue-daxiangwuxing-05/15-5-12.png)

运行结果：

![](/res/img/post/08-ue-daxiangwuxing-05/15-5-13.png)

<br>

### 15.5.2 异步节点

> 这块内容打算后面再另写文章，留个坑

## 15.6 第三方库引用

### 15.6.1 lib 静态链接库的使用

首先先创建一个测试用的静态库：

![](/res/img/post/08-ue-daxiangwuxing-05/15-6-1.png)

![](/res/img/post/08-ue-daxiangwuxing-05/15-6-2.png)

![](/res/img/post/08-ue-daxiangwuxing-05/15-6-3.png)

![](/res/img/post/08-ue-daxiangwuxing-05/15-6-4.png)

`MyLib.h`

```cpp
#pragma once

//通过半径计算面积
float getCircleArea(float radius);

```

`MyLib.cpp`

```cpp
#include "MyLib.h"

float getCircleArea(float radius)
{
	return 3.1415f * (radius * radius);
}
```

点击生成（注意设置目标平台是 `x64`）会得到 `MyLib.h` 和 `LibTest.lib` 两个我们需要用到的文件

> 生成 Debug 或者 Release 版本都可以的。这里以 Release 版本 Lib 作为演示。

将这两个文件放到插件目录下，具体放在什么地方没有强制规定。为了更好地理解，现在将这两个文件放到了 `MyBlank/ThirdParty/x64` 目录下。

![](/res/img/post/08-ue-daxiangwuxing-05/15-6-5.png)

接下来，开始修改 `Build.cs`

```csharp
// Copyright Epic Games, Inc. All Rights Reserved.

using UnrealBuildTool;
using System.IO;

public class MyBlank : ModuleRules
{
    private string ThirdPartyPath
    {
        get
        {
            return Path.GetFullPath(Path.Combine(ModuleDirectory, "../../ThirdParty"));
        }
    }

    private string MyLibPath //第三方库 MyLib 的目录
    {
        get { return Path.GetFullPath(Path.Combine(ThirdPartyPath, "./x64/MyLib")); }
    }

    private bool LoadThirdPartyLib(ReadOnlyTargetRules Target)
    {
        bool isLibrarySupported = false;
        // 平台判断
        if ((Target.Platform == UnrealTargetPlatform.Win64) || (Target.Platform == UnrealTargetPlatform.Win32))
        {
            isLibrarySupported = true;
            System.Console.WriteLine("----- isLibrarySupported true");
            // string PlatformSubPath = (Target.Platform == UnrealTargetPlatform.Win64) ? "Win64" : "Win32";
            string LibrariesPath = Path.Combine(MyLibPath, "Lib");
            // 加载第三方静态库 .lib
            PublicAdditionalLibraries.Add(Path.Combine(LibrariesPath,/* PlatformSubPath,*/ "LibTest.lib"));
        }

        // 成功加载库的情况下，包含第三方库的头文件
        if (isLibrarySupported)
        {
            // Include path
            System.Console.WriteLine("----- PublicIncludePaths.Add true");
            PublicIncludePaths.Add(Path.Combine(MyLibPath, "Include"));
        }

        return isLibrarySupported;
    }
    
	public MyBlank(ReadOnlyTargetRules Target) : base(Target)
	{
		PCHUsage = ModuleRules.PCHUsageMode.UseExplicitOrSharedPCHs;

        // 调用加载静态链接库
        LoadThirdPartyLib(Target);

        PublicIncludePaths.AddRange(
			new string[] {
				// ... add public include paths required here ...
			}
			);
				
		
		...
		...
    }
}

```

至此，静态链接库的引入已经全部完成。我们来看看在 `Build.cs` 里都干了些什么？

首先，定义了一个获取到 `ThirdParty` 目录的属性，通过 `get` 方法获得 `ModuleDirectory` 上两级的 `ThirdParty` 目录。这里的 `ModuleDirectory` 指的是 `Build.cs` 所在的目录。

`LoadThirdPartyLib` 函数用于加载 `lib`，其中传入了 `ReadOnlyTargetRules`（即平台相关信息）。首先对平台进行判断，只有 `Windows` 平台的 `32位` 与 `64位` 才能加载（虽然我们只制作了 `64位` 的 `lib`，但还是留出了 `32位` 的接口）。然后，通过判断不同平台，选择不同的目录。即 `x64` 还是 `x86`。

找到了正确的 `lib` 的路径和头文件所在的目录后，将其分别添加到 `PublicAdditionalLibraries` 与 `PublicIncludePaths`。到此，引入完成。

**使用示例：**

`MyBlueprintFunctionLibrary.h`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "Kismet/BlueprintFunctionLibrary.h"
#include "MyBlueprintFunctionLibrary.generated.h"

/**
 * 
 */
UCLASS()
class MYBLANK_API UMyBlueprintFunctionLibrary : public UBlueprintFunctionLibrary
{
	GENERATED_BODY()

public:
	UFUNCTION(BlueprintCallable, Category = "AnnihilateSword|TestFuncLib")
	static FString SayHello_MyBlank();
};

```

`MyBlueprintFunctionLibrary.cpp`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.


#include "MyBlueprintFunctionLibrary.h"
#include "../ThirdParty/x64/MyLib/Include/MyLib.h"

FString UMyBlueprintFunctionLibrary::SayHello_MyBlank()
{
    float CircleArea = getCircleArea(10.0f);

    return FString::Printf(TEXT("Hello AnnihilateSword!!! -- Circle Area: %f"), CircleArea);
}

```

运行结果：

![](/res/img/post/08-ue-daxiangwuxing-05/15-6-6.png)

<br>
<br>

### 15.6.2 dll 动态链接库的使用

`MyDll.h`

```cpp
#pragma once

#define DLL_EXPORT __declspec(dllexport)

#ifdef __cplusplus
extern "C"
{
#endif
	float DLL_EXPORT getCircleArea(float radius);
#ifdef __cplusplus
}
#endif

```

`MyDll.cpp`

```cpp
#include "MyDll.h"

float DLL_EXPORT getCircleArea(float radius)
{
	return float(3.1415925 * (radius * radius));
}
```

> 这里我生成 Release 版本的 Dll 为例，放在该路径下：

![](/res/img/post/08-ue-daxiangwuxing-05/15-6-7.png)

还是以 `MyBlank` 中的蓝图函数库为示例：

`MyBlueprintFunctionLibrary.cpp`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.


#include "MyBlueprintFunctionLibrary.h"

// dll引入相关
typedef float(*_getCircleArea)(float radius);
_getCircleArea m_getCircleAreaFromDll;
void* dllHandle;


FString UMyBlueprintFunctionLibrary::SayHello_MyBlank()
{
    float area{ 0.0f };

    FString dllpath = FPaths::ProjectDir() / TEXT("Plugins/MyBlank/ThirdParty/x64/MyDll/Dll/MyDll.dll");
    UE_LOG(LogTemp, Warning, TEXT(">>> Dll Path: %s"), *dllpath);
    if (FPaths::FileExists(dllpath))
    {
        dllHandle = FPlatformProcess::GetDllHandle(*dllpath);
        if (dllHandle != NULL)
        {
            m_getCircleAreaFromDll = NULL;
            FString procName = "getCircleArea";
            m_getCircleAreaFromDll = (_getCircleArea)FPlatformProcess::GetDllExport(dllHandle, *procName);
            if (m_getCircleAreaFromDll == NULL)
            {
                UE_LOG(LogTemp, Warning, TEXT(">>> Dll Load Falied!!!"));
                return FString::Printf(TEXT(">>> Hello AnnihilateSword!!! -- Circle Area: %f"), -100.0f);
            }
            area = m_getCircleAreaFromDll(200.f);

            UE_LOG(LogTemp, Warning, TEXT(">>> Dll Load Successful!!!"));
            UE_LOG(LogTemp, Warning, TEXT(">>> Dll Calculate Area %f"), area);
            return FString::Printf(TEXT(">>> Hello AnnihilateSword!!! -- Circle Area: %f"), area);
        }
    }
    return FString::Printf(TEXT(">>>Hello AnnihilateSword!!! -- Circle Area: %f"), -200.0f);
}

```

![](/res/img/post/08-ue-daxiangwuxing-05/15-6-8.png)

> 👉[4.27.2 官方文档的一些说明](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/third-party-libraries?application_version=4.27)

<br>
<br>

# 十六、自定义资源和编辑器

> 这块内容打算后面再另写文章，留个坑

<br>
<br>

# 完结

马上过年了，提前祝大家：新年快乐，天天开心！

![](/res/img/post/08-ue-daxiangwuxing-05/end.png)

<br>
<br>

# 其他参考

[1] 👉[【UE4 C++】调用外部链接库：lib静态库](https://www.cnblogs.com/shiroe/p/14732889.html)
[2] 👉https://dev.epicgames.com/documentation/zh-cn/unreal-engine/third-party-libraries?application_version=4.27

<br>
<br>


The End.
