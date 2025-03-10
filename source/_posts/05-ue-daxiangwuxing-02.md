---
title: 【UE大象无形总结】第二部分虚幻引擎浅析【上卷】
tags: [UE4]
date: 2025-01-04
updated: 2025-01-04
cover: /res/img/post/05-ue-daxiangwuxing-02/cover.png
top_img: /res/img/site/background.png
---

![](/res/img/post/05-ue-daxiangwuxing-02/cover.png)

<br>

# 前言

> **幸得识君桃花面，从此阡陌多暖春。**
> 
> 最近一口气看完了不良人1~6季，目前最喜欢的国漫了，期待第七季！
> 参考书籍：《大象无形：虚幻引擎程序设计浅析 (罗丁力 [罗丁力])》
> 补充说明：本系列是个人对书中内容的实践

考虑到篇幅，这里将虚幻引擎浅析分成上中下三卷记录。

> 对于 Windows 平台而言，你可以在 `Launch` 模块下的 `LaunchWindows.cpp` 中找到你熟悉的 `WinMain` 函数。

<br>
<br>

# 八、模块机制

## 8.1 模块简介

> 虚幻引擎为什么需要引入模块机制？

任何一名 `C++` 程序员，一定被 `C++` 的项目配置所困扰过。你需要添加头文件目录、`lib` 目录等。还需要对 `Debug` 模式、`Release` 模式下不同的包含做出控制。这是相当复杂的一个问题。更不用说对于虚幻引擎来说，需要的编译模式远远不止这么几种。在编辑器模式下，有些类会被引入，在最终发布的游戏中，这些类又完全不再需要，如何处理这样的问题呢？

在 `UE3` 的时代，使用 `MakeFile` 来模拟模块，而到了 `UE4` 的时代，为了彻底解决这样的问题，虚幻引擎借助 `UnrealBuildTool` 引入了模块机制。

仔细观察虚幻引擎的源码目录，会发现其按照四大部分：`Runtime`、`Development`、`Editor`、`Plugin`来进行规划，而每个部分内部，则包含了一个一个的小文件夹。每个文件夹即对应一个模块。一个模块文件夹中应该包含这些内容。

- `Public` 文件夹
- `Private` 文件夹
- `.build.cs` 文件

模块的划分是一门艺术。你需要知道的是，不同模块之间的互相调用并不是很方便。只有通过 `XXXX_API` 宏暴露的类和成员函数才能够被其他模块访问。因此，模块系统也让你从一开始就对自己的类进行精心的设计，以免出现复杂的类与类依赖。

<br>
<br>

## 8.2 创建自己的模块【重要】

> 写这篇文章的时候，我使用的是 UE4.27.2 版本，这部分参考的其他资料，书上的已经行不通了

### 8.2.1 前置准备

为了方便演示，这里我从新建项目一步步来

1. 首先新建一个空白 C++ 项目，这里我命名为 MMORPG；
2. 这里我整理了一下目录结构：
   ```txt
   MMORPG
        Private
            MMORPG.cpp
            MMORPGGameModeBase.cpp
        Public
            MMORPG.h
            MMORPGGameModeBase.h
        MMORPG.Build.cs
   ```
3. 删除工程目录下的 `Binaries` 和 `Intermediate` 文件；
4. 右键 `.uproject` 文件重新生成 VS 项目文件；

启动编辑器，新建一个 `ActorTest` 类继承自 `AActor`，重新构建项目，然后创建一个 ActorTest 蓝图实例在 Level 中用作测试。

<br>

### 8.2.2 修改 .uproject 文件

修改 `.uproject `文件，这里我添加一个 `CommonTool` 模块作为示例：

```txt
"Modules": [
	{
		"Name": "MMORPG",
		"Type": "Runtime",
		"LoadingPhase": "Default",
		"AdditionalDependencies": [
			"Engine"
		]
	},
	{
		"Name": "CommonTool",
		"Type": "Runtime",
		"LoadingPhase": "Default"
	}
]
```

在 Source 目录下添加以下文件：

```txt
CommonTool
    Private
        CommonTool.cpp
    Public
        CommonTool.h
    CommonTool.Build.cs
```

`CommonTool.Build.cs`，这个可以直接复制其他模块的 `.Build.cs` 稍作修改：

```csharp
// Copyright Epic Games, Inc. All Rights Reserved.

using UnrealBuildTool;

public class CommonTool : ModuleRules
{
	public CommonTool(ReadOnlyTargetRules Target) : base(Target)
	{
		PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;
	
		PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "InputCore" });

		PrivateDependencyModuleNames.AddRange(new string[] {  });

		// Uncomment if you are using Slate UI
		// PrivateDependencyModuleNames.AddRange(new string[] { "Slate", "SlateCore" });
		
		// Uncomment if you are using online features
		// PrivateDependencyModuleNames.Add("OnlineSubsystem");

		// To include OnlineSubsystemSteam, add it to the plugins section in your uproject file with the Enabled attribute set to true
	}
}
```

`CommonTool.h`

```cpp
#pragma once

#include "Modules/ModuleManager.h"

DECLARE_LOG_CATEGORY_EXTERN(CommonToolModule, All, All);

class FCommonToolModule : public IModuleInterface
{
public:
	/* This will get called when the editor loads the module */
	virtual void StartupModule() override;
	
	/* This will get called when the editor unloads the module */
	virtual void ShutdownModule() override;
}
```

`CommonTool.cpp`

```cpp
#include "CommonTool.h"

DEFINE_LOG_CATEGORY(CommonToolModule);

void FCommonToolModule::StartupModule()
{
	UE_LOG(CommonToolModule, Warning, TEXT("CommonTool module has started!"));
}

void FCommonToolModule::ShutdownModule()
{
	UE_LOG(CommonToolModule, Warning, TEXT("CommonTool module has shut down"));
}

IMPLEMENT_MODULE(FCommonToolModule, CommonTool)
```

**然后还需要在项目，我这里是 `MMORPG.Target.cs` 和 `MMORPGEditor.Target.cs` 文件里面添加该模块**：

```csharp
// ----------------- MMORPG.Target.cs
using UnrealBuildTool;
using System.Collections.Generic;

public class MMORPGTarget : TargetRules
{
	public MMORPGTarget( TargetInfo Target) : base(Target)
	{
		Type = TargetType.Game;
		DefaultBuildSettings = BuildSettingsVersion.V2;
		ExtraModuleNames.AddRange( new string[] { "MMORPG", "CommonTool" } );
	}
}

// ----------------- MMORPGEditor.Target.cs
using UnrealBuildTool;
using System.Collections.Generic;

public class MMORPGEditorTarget : TargetRules
{
	public MMORPGEditorTarget( TargetInfo Target) : base(Target)
	{
		Type = TargetType.Editor;
		DefaultBuildSettings = BuildSettingsVersion.V2;
		ExtraModuleNames.AddRange( new string[] { "MMORPG", "CommonTool" } );
	}
}
```

最后删除 `Binaries` 和 `Intermediate` 文件夹，右键 `.uproject` 文件重新生成 VS 项目文件，构建项目。

到这里就成功新建我们自定义的模块了。

![](/res/img/post/05-ue-daxiangwuxing-02/8-2-1.png)

<br>

### 8.2.3 引入模块

![](/res/img/post/05-ue-daxiangwuxing-02/8-2-2.png)

这里新建一个 UObject 类用来测试，是之前写过的图片转换工具函数

> UE4.27.2 中 ImageWrapper 好像自己默认包含在引用的模块了，这里没有引入模块也能使用

`ImageConvertManager.h`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "UObject/NoExportTypes.h"
#include "ImageConvertManager.generated.h"

/**
 * 
 */
UCLASS()
class COMMONTOOL_API UImageConvertManager : public UObject
{
	GENERATED_BODY()
	
public:
	static UImageConvertManager* GetInstance();

	bool CovertPNG2JPG(const FString& SourceName, const FString& TargetName);

private:
	static UImageConvertManager* Instance;
};
```

`ImageConvertManager.cpp`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.


#include "ImageConvertManager.h"
#include "IImageWrapperModule.h"
#include "IImageWrapper.h"
#include "HAL/FileManagerGeneric.h"

UImageConvertManager* UImageConvertManager::Instance = nullptr;

UImageConvertManager* UImageConvertManager::GetInstance()
{
    if (Instance == nullptr)
	{
		Instance = NewObject<UImageConvertManager>();
	}
    return Instance;
}

bool UImageConvertManager::CovertPNG2JPG(const FString& SourceName, const FString& TargetName)
{
	check(SourceName.EndsWith(TEXT(".png")) && TargetName.EndsWith(TEXT(".jpg")));

	// FModuleManager::LoadModuleChecked 用于加载并获取一个特定模块的实例。
	// 它的作用是确保指定的模块已经被加载，并返回该模块的实例引用。
	// 如果模块尚未加载，会触发一个断言错误（assertion error）。
	IImageWrapperModule& ImageWrapperModule = FModuleManager::LoadModuleChecked<IImageWrapperModule>(TEXT("ImageWrapper"));

	IImageWrapperPtr SourceImageWrapper = ImageWrapperModule.CreateImageWrapper(EImageFormat::PNG);
	IImageWrapperPtr TargetImageWrapper = ImageWrapperModule.CreateImageWrapper(EImageFormat::JPEG);

	TArray<uint8> SourceImageData;
	TArray<uint8> TargetImageData;

	int Width, Height;
	TArray<uint8> UncompressedRGBA;
	if (!FPlatformFileManager::Get().GetPlatformFile().FileExists(*SourceName))
	{
		// 文件不存在
		return false;
	}

	if (!FFileHelper::LoadFileToArray(SourceImageData, *SourceName))
	{
		// 文件读取失败
		return false;
	}

	if (SourceImageWrapper.IsValid() && SourceImageWrapper->SetCompressed(SourceImageData.GetData(), SourceImageData.Num()))
	{
		if (SourceImageWrapper->GetRaw(ERGBFormat::RGBA, 8, UncompressedRGBA))
		{
			Width = SourceImageWrapper->GetWidth();
			Height = SourceImageWrapper->GetHeight();
			if (TargetImageWrapper->SetRaw(UncompressedRGBA.GetData(), UncompressedRGBA.Num(), Width, Height, ERGBFormat::RGBA, 8))
			{
				TargetImageData = SourceImageWrapper->GetCompressed();
				if (!FFileManagerGeneric::Get().DirectoryExists(*TargetName))
				{
					FFileManagerGeneric::Get().MakeDirectory(*FPaths::GetPath(TargetName), true);
				}
				return FFileHelper::SaveArrayToFile(TargetImageData, *TargetName);
			}
		}
	}

	return false;
}
```

**然后我们需要引入 `CommonTool` 模块才能在 `MMORPG` 模块中使用**：

![](/res/img/post/05-ue-daxiangwuxing-02/8-2-3.png)

删除 `Binaries` 和 `Intermediate` 文件夹，右键 `.uproject` 文件重新生成 VS 项目文件，构建项目即可。

`ActorTest.cpp`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.


#include "ActorTest.h"
#include "ImageConvertManager.h"

DEFINE_LOG_CATEGORY_STATIC(MMORPG, Warning, All);

AActorTest::AActorTest()
{
	PrimaryActorTick.bCanEverTick = true;
}

void AActorTest::BeginPlay()
{
	Super::BeginPlay();

	UImageConvertManager* ConvertManager = UImageConvertManager::GetInstance();
	bool Success = ConvertManager->CovertPNG2JPG(FPaths::ProjectContentDir() / TEXT("Pic/Source.png"), FPaths::ProjectContentDir() / TEXT("Pic/Target.jpg"));
	if (Success)
	{
		UE_LOG(MMORPG, Warning, TEXT("Covert PNG to JPG Success!"));
	}
}

void AActorTest::Tick(float DeltaTime)
{
	Super::Tick(DeltaTime);
}
```

![](/res/img/post/05-ue-daxiangwuxing-02/8-2-4.png)

这里可以看出已经成功了，真是让人兴奋。

<br>
<br>

## 8.3 虚幻引擎初始化模块加载顺序

> 本节希望给有需要的读者一个参考性质的模块加载顺序。实际上虚幻引擎模块加载分为两大部分，一部分是以硬编码形式硬性规定，另一部分则是松散加载。前半部分的加载顺序有自己的依赖原则。

注意，根据你引擎发布的版本不同（Editor/Development/Shipping），模块加载也不相同。有些模块不会被加载。但是总体上说，模块的加载遵循这样的顺序：

1. 首先加载的是 `Platform File Module`，因为虚幻引擎要读取文件。
2. 接下来加载的是核心模块：`(FEngineLoop::PreInit→LoadCoreModules)`。
3. 加载 `CoreUObject`。
4. 然后在初始化引擎之前加载模块：`FEngineLoop::LoadPreInitModules`。
5. 加载 `Engine`。
6. 加载 `Renderer`。
7. 加载 `AnimGraphRuntime`。

根据你的平台不同，会加载平台相关的模块（FPlatformMisc::LoadPreInitModules），在 Windows 平台下加载的是：

1. D3D11RHI（开启 bForceD3D12 后会加载 D3D12）
2. OpenGLDrv
3. SlateRHIRenderer
4. Landscape
5. ShaderCore
6. TextureCompresser
7. Start Up Modules: FEngineLoop::LoadStartupCoreModules
8. Core（很有趣，虚幻引擎的核心 Core 模块加载的时机并不是在最
初）
9. Networking

然后是其他平台相关模块，Windows 平台下是：

1. XAudio2
2. HeadMountedDisplay
3. SourceCodeAccess
4. Messaging
5. SessionServices
6. EditorStyle
7. Slate
8. UMG
9. MessageLog
10. CollisionAnalyzer
11. FunctionalTesting
12. BehaviorTreeEditor
13. GameplayTasksEditor
14. GameplayAbilitiesEditor
15. EnvironmentQueryEditor
16. OnlineBlueprintSupport
17. IntroTutorials
18. Blutility

接下来，会根据启用的插件，加载对应的模块。然后是：

1. TaskGraph
2. ProfilerService

至此，虚幻引擎初始化完成所需的模块加载。

之后的 `Module` 加载，是根据情况来完成的。也就是说 `Unreal Editor` 自己来控制其需要加载什么样的程序。发布后的游戏因为没有携带 `Unreal Editor`，因此自然不需要加载这些模块。而笔者认为，对模块加载顺序的研究，主要的部分还是集中在引擎加载初期。后期的模块大多针对具体功能。

当然，有很多时候，我们需要处理模块依赖，这个时候其他模块的
加载顺序也是有一定的研究价值的。

如果你好奇模块的加载顺序，模块加载是在 
`UEditorEngine` 的 `Init` 中，被一个数组控制：

1. Documentation
2. WorkspaceMenuStructure
3. MainFrame
4. GammaUI
5. OutputLog
6. SourceControl
7. TextureCompressor
8. MeshUtilities
9. MovieSceneTools
10. ModuleUI
11. Toolbox
12. ClassViewer
13. ContentBrowser
14. AssetTools
15. GraphEditor
16. KismetCompiler
17. Kismet
18. Persona
19. LevelEditor
20. MainFrame
21. PropertyEditor
22. EditorStyle
23. PackagesDialog
24. AssetRegistry
25. DetailCustomizations
26. ComponentVisualizers
27. Layers
28. AutomationWindow
29. AutomationController
30. DeviceManager
31. ProfilerClien
32. SessionFrontend
33. ProjectLauncher
34. SettingsEditor
35. EditorSettingsViewer
36. ProjectSettingsViewer
37. Blutility
38. OnlineBlueprintSupport
39. XmlParser
40. UserFeedback
41. GameplayTagsEditor
42. UndoHistory
43. DeviceProfileEdito
44. SourceCodeAccess
45. BehaviorTreeEditor
46. HardwareTargetin
47. LocalizationDashboard
48. ReferenceViewer
49. TreeMap
50. SizeMap
51. MergeActors

以上内容是为了方便读者在引入模块的时候确定 “哪些模块一定加载”。读者也可以根据这里的模块名称来大致推测自己需要的功能可能位于哪个模块。例如根据 XmlParser 模块名字，就能推测这里放置了 XML 解析需要的 API。

<br>
<br>

## 8.4 道常无名：UBT 和 UHT 简介

> 模块机制很厉害，但是模块是用什么样的方式配置、编译、启动和运行呢？大牛们常常提到的 `UBT` 和 `UHT` 又是什么呢

### 8.4.1 UBT

#### UBT概览

如果你下载了一份虚幻引擎的源代码，你能够在UBT（Unreal Build Tool）项目的 `UnrealBuildTool.cs` 文件中找到 Main 函数。

**大致而言，UBT 的工作分为三个阶段**：

1. 收集阶段：收集信息。UBT 收集环境变量、虚幻引擎源代码目录、虚幻引擎目录、工程目录等一系列的信息。
2. 参数解析阶段：UBT 解析传入的命令行参数，确定自己需要生成的目标类型。
3. 实际生成阶段：UBT 根据环境和参数，开始生成 makefile，确定 C++ 的各种目录的位置。最终开始构建整个项目。此时编译工作交给标准 C++ 编译器。

同时 UBT 也负责监视是否需要热加载。并且调用 UHT 收集各个模块的信息。

请注意，**UBT 被设计为一个跨平台的构建工具**，因此针对不同的平台有相对应的类来进行处理。UBT 生成的 makefile 会被对应编译器平台的 `ProjectFileGenerater` 用于生成解决方案，而不是一开始就针对某个平台的解决方案来确定如何生成。

<br>

#### 再谈 WinMain

> 在最开头我们谈到了 WinMain 函数在哪，解除了大家许多心理的戒备。但是其实接下来的问题更加严峻：WinMain 函数最终怎么会被链接到 .exe 文件？

> 事实上，作者也专门为此探究了一阵。如果你对此没什么兴趣，直接跳过本部分就行。

首先如果大家熟悉 C++，就会知道，在 Windows 平台最终的 .exe 文件中，必须包含 Main 函数（C/C++控制台程序）或者 WinMain 函数（Windows）。否则是找不到应用程序的入口点的。而虚幻引擎的模块，最终大多被编译为 DLL 动态链接库文件。

如果你看过之前的模块加载（通过 FModuleManager::LoadModule），你会发现这里有一大堆眼熟的模块 DLL（Slate、SlateCore等）。

那么问题来了，为什么我们的 Launcher 模块没有被编译为 DLL？是什么控制的呢？

答案是，在 `.target.cs` 文件控制。

比如 UE4，被 `UE4Editor.Target.cs` 文件所控制编译，如果你是用的编译版引擎，你是看不到完整的编译控制的，所以你会困惑一些。其实，在 `.Target.cs` 文件中，如下所示：

```csharp
// Copyright Epic Games, Inc. All Rights Reserved.

using UnrealBuildTool;
using System.Collections.Generic;

public class UE4EditorTarget : TargetRules
{
	public UE4EditorTarget( TargetInfo Target ) : base(Target)
	{
		Type = TargetType.Editor;
		BuildEnvironment = TargetBuildEnvironment.Shared;
		bBuildAllModules = true;
		ExtraModuleNames.Add("UE4Game");
	}
}
```

有兴趣可以看看 `UEBuildTarget.cs` 文件中的 `SetupBinaries()` 函数，我这里用的是 UE4.27.2 源码编译版。

```cpp
// Add the launch module to it
LaunchModule.Binary = Binary;
Binary.AddModule(LaunchModule);
```

`Launcher` 模块会被设置为 `UEBuildBinaryType.Executable` 添加进来。作为 `.exe` 文件的包含模块。

此时 `.exe` 文件就会包含 `Launcher` 模块中平台对应的 `Main` 函数。对于 `Windows` 平台而言，`WinMain` 函数会被链接进 `.exe` 文件。

当你双击虚幻引擎的 `Editor.exe` 后，`WinMain` 函数被调用。虚幻引擎开始在你的系统上运行起来。

<br>

### 8.4.2 UHT

> 前文有提到，UHT 配合实现了反射机制，那 UHT 是如何完成的呢？

**UHT（Unreal Header Tool）** 一个引擎独立应用程序

作者不知道该如何形容这样的程序，只是勉强给出了一个我个人的命名。

引擎独立应用程序是指，这种应用程序依赖引擎。比如**依赖 UBT 以配置编译环境，从而能跨平台编译，依赖引擎的某些模块**。

但是这样的应用程序又不是一个引擎，也不是一个游戏。它最终输出为一个 `.exe` 文件。但是又不需要引擎完全启动，甚至不需要 `Renderer` 模块，以至于有可能只是一个命令行工具——比如 `UHT`。

通过虚幻引擎源代码，你会发现 `UHT` 拥有自己的 `.target.cs` 和 `.build.cs` 文件。

在 `UnrealHeaderTool.Target.cs` 文件中，它把自己设置为 `exe` 的输出模块。

在 `UnrealHeaderTool.Build.cs` 文件中，它指出了自己的依赖模块：

- Core
- CoreUObjects
- Json
- Projects

然后又包含了 `Launch` 模块的 `public/private` 文件夹，以便让自己能够调用 `GEngine->PreInit` 函数。

```csharp
// Copyright Epic Games, Inc. All Rights Reserved.

using UnrealBuildTool;

public class UnrealHeaderTool : ModuleRules
{
	public UnrealHeaderTool(ReadOnlyTargetRules Target) : base(Target)
	{
		PublicIncludePaths.Add("Runtime/Launch/Public");

		PrivateDependencyModuleNames.AddRange(
			new string[]
			{
				"Core",
				"CoreUObject",
				"Json",
				"Projects"
			}
		);
	
		PrivateIncludePaths.AddRange(
			new string[]
			{
				// For LaunchEngineLoop.cpp includes
				"Runtime/Launch/Private",
				"Runtime/RHI/Public",
				"Programs/UnrealHeaderTool/Private",
			});
		
		bEnableExceptions = true;

		UnsafeTypeCastWarningLevel = WarningLevel.Warning;
	}
}
```

最终，`UHT` 会被编译成一个 `.exe` 文件，通过命令行参数调用。

```csharp
// UnrealHeaderTool is a console application, not a Windows app (sets entry point to main(), instead of WinMain())
bIsBuildingConsoleApplication = true;
```

很奇妙不是吗？一个能够用来编译引擎的程序，居然依赖引擎本身。在本书的第三部分，将会教读者自己制作一个这样的程序。

<br>

#### UHT 大致工作流程

`UHT` 的 `Main` 函数在 `UnrealHeadToolMain.cpp` 文件中。这个文件提供了 `Main` 函数，并且也通过 `IMPLEMENT_APPLICATION` 宏声明了这是个独立应用程序。具体执行的内容如下：

1. 调用 `GEngineLoop.PreInit` 函数，初始化 Log、文件系统等基础系统。从而允许借助 UE_LOG 宏输出 log 信息。
2. 调用 `UnrealHeaderTool_Main` 函数，执行真正的工作内容。
3. 调用 `FEngineLoop::AppExit` 退出。

那么，`UnrealHeaderTool_Main` 函数到底干了什么？

<br>

#### UHT 到底干了什么

首先，`UBT` 会通过命令行参数告诉 `UHT`，游戏模块对应的定义文件在哪。这个文件是一个 `.manifest` 文件。这是个由 `UBT` 生成的文件。如果你用记事本之类的应用打开，你会发现这是个 `Json` 字符串。

![](/res/img/post/05-ue-daxiangwuxing-02/8-4-1.png)

你会发现这个字符串包含了所有 `Module` 的编译相关的信息。包括各种路径，以及预编译头文件位置等。

然后 `UHT` 开始了自己的三遍编译：`Public Classes Headers`、`Public Headers`、`Private Headers`。第一遍其实是为了兼容 UE3 时代的代码。UE3 时代的 `Classes` 文件夹会向 `Unreal Script` 暴露这些类。接下来的两遍则是解析头文件定义，并生成 `C++` 代码以完成需要的功能。生成的代码会和现有代码一起联合编译以产生结果。

<br>

#### 虚幻引擎反射机制与超生游击队的故事

> UHT 生成的是代码么？代码如何完成 “类有哪些成员变量、成员函数” 这样的信息的注册呢？

经过 UHT 的三遍编译，最终生成的是 `.generated.cpp` 和 `.generated.h` 这两种文件。而我们理解 UHT 工作内容，也得根据这两个文件来理解。

让我们先来思考一个问题。众所周知，虚幻引擎的一个 `UClass`，可以理解为包含 `UProperty` 和 `UFunction` 的一个数据结构：

```txt
UClass
-UProperty
-...
-UProperty
-UFunction
-...
-UFunction
```

随后虚幻引擎需要做的其实是，把 C++ 类（也就是真正的 “成员变量” 和 “成员函数”）与 UClass 中的对应的 “Property数据类型” 和 “Function数据类型” 绑定起来。

打个比方，UClass 其实只是一张表，或者说类似户口本的东西。上面记录的是指向真实的 “家庭” 的指针，可能还包含一些额外的信息。假设我们拿到的是张家人的户口本（当然这里是假设的户口本，随便写了点信息），上面写着：

```txt
张大：
男性
1967年生

张二：
女性
1968年生

张三：
男性
1969年生
```

此时我们知道，这三个条目记录的是 张大、张二和张三的 “额外信息”，或者说元数据 `metadata`，但是到时候你找人，比如说你要拿着这个户口本找张大、张二和张三。你还得根据这个户口本条目找到对应的那个人，也就得通过一个指针。

现在问题是，虚幻引擎该怎么处理这个户口本机制呢？**一种方式是，一开始编译时就把户口本都填好，放在一个文件里面。要找某家人的时候，就读取出来，开始查。但是大家都知道，每一次编译，函数的地址可能会发生变化，而每一次运行，函数映射到内存中的位置也可能不同。直接存储地址值是不行的。** 就比方说张家人每次人口普查之前都搬家（因为超生了，超生游击队），所以如果直接存储上次普查得到的地址，那么时过境迁，根据这个地址去找张家，实际找到的可能是李家，甚至这个地址都拆迁了。

所以虚幻引擎想了个办法。**UHT 不是存储的 “每家人的户口本”，而是把 “进行户口调查的过程” 存储了下来。** 比方说，它存储了每个家庭有哪些人需要登记信息。然后在运行之初，借助某个特殊的技巧，在 Main 函数调用之前，逐个敲门让每家人进行登记。虽然张家人要搬家，没关系，跑得了和尚跑不了庙，我把这个城里的每个家庭找一遍，你总归还是要被我找到的。

当然，有些热心的读者要说了，万一张家又生了个娃（张四），岂不是记录不到了吗？**虚幻引擎的 UHT 就是扮演着那个计生办的角色。你的 C++ 类，如果改动了头文件，肯定得编译吧？意味着你生娃总得给娃办一个户口吧，否则那个娃以后是黑户，书都读不了。所以当你给娃办一个户口（用 UPROPERTY 标记或者用 UFUNTION 标记），编译的时候就被 UHT 看到了，它就把如何获取具体的信息（如何根据户口本找到张四）的步骤写在了 `.generated.cpp` 里面。**

<br>

#### 加载信息

仅仅记录获取信息的过程还不足够，还需要在加载的时候调用记录好的过程来进行实际的记录。前文说道这是通过在 Main 函数调用前执行记录过程的技巧，**实质上来说，方法是通过一个简单的 C++ 机制：静态全局变量的初始化先于 Main 函数执行！**

在生成好的 `.generated.cpp` 文件中，会看到一个宏：

```cpp
IMPLEMENT_CLASS
```

这个宏展开后实质上完成了两个内容：

1. 声明了一个具有独一无二名字的 `UClassCompiledInDefer<当前类名>` 静态全局变量实例。
2. 实现了当前类的 `GetPrivateStaticClass` 函数。

我们重点观察前一个步骤。这个类的构造函数调用了 `UClassCompiledInDefer` 函数。即：

1. 这个静态全局变量会在Main函数之前初始化。
2. 初始化就会调用该变量的构造函数。
3. 构造函数会调用 `UClassCompiledInDefer` 函数。
4. `UClassCompiledInDefer` 会添加 `ClassInfo` 变量到延迟注册的数组中。

> 下面代码截取自 UE4，这是另一台机器且魔改了的引擎不太清楚具体版本，不过这段代码应该没有太多改动

```cpp
// ------------------------------- ObjectMacros.h

// Register a class at startup time.
#define IMPLEMENT_CLASS(TClass, TClassCrc) \
	static TClassCompiledInDefer<TClass> AutoInitialize##TClass(TEXT(#TClass), sizeof(TClass), TClassCrc); \
	UClass* TClass::GetPrivateStaticClass() \
	{ \
		static UClass* PrivateStaticClass = NULL; \
		if (!PrivateStaticClass) \
		{ \
			/* this could be handled with templates, but we want it external to avoid code bloat */ \
			GetPrivateStaticClassBody( \
				StaticPackage(), \
				(TCHAR*)TEXT(#TClass) + 1 + ((StaticClassFlags & CLASS_Deprecated) ? 11 : 0), \
				PrivateStaticClass, \
				StaticRegisterNatives##TClass, \
				sizeof(TClass), \
				alignof(TClass), \
				(EClassFlags)TClass::StaticClassFlags, \
				TClass::StaticClassCastFlags(), \
				TClass::StaticConfigName(), \
				(UClass::ClassConstructorType)InternalConstructor<TClass>, \
				(UClass::ClassVTableHelperCtorCallerType)InternalVTableHelperCtorCaller<TClass>, \
				&TClass::AddReferencedObjects, \
				&TClass::Super::StaticClass, \
				&TClass::WithinClass::StaticClass \
			); \
		} \
		return PrivateStaticClass; \
	}


// ------------------------------- Class.cpp
void GetPrivateStaticClassBody(
	const TCHAR* PackageName,
	const TCHAR* Name,
	UClass*& ReturnClass,
	void(*RegisterNativeFunc)(),
	uint32 InSize,
	uint32 InAlignment,
	EClassFlags InClassFlags,
	EClassCastFlags InClassCastFlags,
	const TCHAR* InConfigName,
	UClass::ClassConstructorType InClassConstructor,
	UClass::ClassVTableHelperCtorCallerType InClassVTableHelperCtorCaller,
	UClass::ClassAddReferencedObjectsType InClassAddReferencedObjects,
	UClass::StaticClassFunctionType InSuperClassFn,
	UClass::StaticClassFunctionType InWithinClassFn,
	bool bIsDynamic /*= false*/,
	UDynamicClass::DynamicClassInitializerType InDynamicClassInitializerFn /*= nullptr*/
	)
{

}
```

**绕了这么大一圈，其实是希望能够在 Main 函数执行前，先执行 UClassCompiledInDefer 函数。这个函数带有 “Defer”，是因为实际注册操作是后来执行的，但是在 Main 函数之前必须得 “先摇个号” 。这样虚幻引擎才知道有哪个类是需要延迟加载。借助这个技巧，虚幻引擎能够在启动前就给全部的类发个号，等时机成熟再一个一个加载信息。**

> 严格来说，无副作用的静态全局变量的初始化可以延迟到使用时，但是这不影响我们的分析。

<br>
<br>

# 九、重要核心系统简介

> 本章介绍了虚幻引擎中的一部分重要核心系统，这些系统与后文的讨论有关，故详细介绍。实际上虚幻引擎的 Core 模块包含大量的内容，例如 TArray 定义等，在此不做过多赘述。

## 9.1 内存分配

### 9.1.1 Windows 操作系统下的内存分配方案

在 Windows 操作系统下，虚幻引擎是通过宏来控制，并在几个内存分配器中选择的。对应的代码如下：

```cpp
// ------------- WindowsPlatformMemory.cpp
FMalloc* FWindowsPlatformMemory::BaseAllocator()
{
#if ENABLE_WIN_ALLOC_TRACKING
	// This allows tracking of allocations that don't happen within the engine's wrappers.
	// This actually won't be compiled unless bDebugBuildsActuallyUseDebugCRT is set in the
	// build configuration for UBT.
	_CrtSetAllocHook(WindowsAllocHook);
#endif // ENABLE_WIN_ALLOC_TRACKING

	if (FORCE_ANSI_ALLOCATOR) //-V517
	{
		AllocatorToUse = EMemoryAllocatorToUse::Ansi;
	}
	else if ((WITH_EDITORONLY_DATA || IS_PROGRAM) && TBB_ALLOCATOR_ALLOWED) //-V517
	{
		AllocatorToUse = EMemoryAllocatorToUse::TBB;
	}
#if PLATFORM_64BITS
	else if ((WITH_EDITORONLY_DATA || IS_PROGRAM) && MIMALLOC_ALLOCATOR_ALLOWED) //-V517
	{
		AllocatorToUse = EMemoryAllocatorToUse::Mimalloc;
	}
	else if (USE_MALLOC_BINNED3)
	{
		AllocatorToUse = EMemoryAllocatorToUse::Binned3;
	}
#endif
	else if (USE_MALLOC_BINNED2)
	{
		AllocatorToUse = EMemoryAllocatorToUse::Binned2;
	}
	else
	{
		AllocatorToUse = EMemoryAllocatorToUse::Binned;
	}
	
#if !UE_BUILD_SHIPPING
	// If not shipping, allow overriding with command line options, this happens very early so we need to use windows functions
	const TCHAR* CommandLine = ::GetCommandLineW();

	if (FCString::Stristr(CommandLine, TEXT("-ansimalloc")))
	{
		AllocatorToUse = EMemoryAllocatorToUse::Ansi;
	}
#if TBB_ALLOCATOR_ALLOWED
	else if (FCString::Stristr(CommandLine, TEXT("-tbbmalloc")))
	{
		AllocatorToUse = EMemoryAllocatorToUse::TBB;
	}
#endif
#if MIMALLOC_ALLOCATOR_ALLOWED
	else if (FCString::Stristr(CommandLine, TEXT("-mimalloc")))
	{
		AllocatorToUse = EMemoryAllocatorToUse::Mimalloc;
	}
#endif
#if PLATFORM_64BITS
	else if (FCString::Stristr(CommandLine, TEXT("-binnedmalloc3")))
	{
		AllocatorToUse = EMemoryAllocatorToUse::Binned3;
	}
#endif
	else if (FCString::Stristr(CommandLine, TEXT("-binnedmalloc2")))
	{
		AllocatorToUse = EMemoryAllocatorToUse::Binned2;
	}
	else if (FCString::Stristr(CommandLine, TEXT("-binnedmalloc")))
	{
		AllocatorToUse = EMemoryAllocatorToUse::Binned;
	}
#if WITH_MALLOC_STOMP
	else if (FCString::Stristr(CommandLine, TEXT("-stompmalloc")))
	{
		AllocatorToUse = EMemoryAllocatorToUse::Stomp;
	}
#endif // WITH_MALLOC_STOMP
#endif // !UE_BUILD_SHIPPING

	switch (AllocatorToUse)
	{
	case EMemoryAllocatorToUse::Ansi:
		return new FMallocAnsi();
#if WITH_MALLOC_STOMP
	case EMemoryAllocatorToUse::Stomp:
		return new FMallocStomp();
#endif
#if TBB_ALLOCATOR_ALLOWED
	case EMemoryAllocatorToUse::TBB:
		return new FMallocTBB();
#endif
#if MIMALLOC_ALLOCATOR_ALLOWED && PLATFORM_SUPPORTS_MIMALLOC
	case EMemoryAllocatorToUse::Mimalloc:
		return new FMallocMimalloc();
#endif
	case EMemoryAllocatorToUse::Binned2:
		return new FMallocBinned2();
#if PLATFORM_64BITS
	case EMemoryAllocatorToUse::Binned3:
		return new FMallocBinned3();
#endif
	default:	// intentional fall-through
	case EMemoryAllocatorToUse::Binned:
		return new FMallocBinned((uint32)(GetConstants().BinnedPageSize&MAX_uint32), (uint64)MAX_uint32 + 1);
	}
}
```

```cpp
// ----------------- GenericPlatformMemory.h

/** Which allocator is being used */
enum EMemoryAllocatorToUse
{
	Ansi, // Default C allocator
	Stomp, // Allocator to check for memory stomping
	TBB, // Thread Building Blocks malloc
	Jemalloc, // Linux/FreeBSD malloc
	Binned, // Older binned malloc
	Binned2, // Newer binned malloc
	Binned3, // Newer VM-based binned malloc, 64 bit only
	Platform, // Custom platform specific allocator
	Mimalloc, // mimalloc
};
```

Windows 平台提供了标准的 Malloc（ANSI）、Intel TBB
内存分配器，以及Binned内存分配器等多个方案。

<br>

### 9.1.2 IntelTBB 内存分配器

如果读者对IntelTBB内存分配器感兴趣，可以参考这本书：**《Intel Threading Building Blocks编程指南》**。

采用 TBB 内存分配，主要是出于以下的原因：

1. 虚幻引擎工作在多个线程，而标准内存分配为了避免出现内存分配 BUG，是强制同一时间只有一个线程分配内存。导致内存分配的速度大幅度降低。
   > 这个跟公司一位前辈交流时了解到，现代的内存分配器已经有线程级别的 Cache 了，就释放内存的时候，不会真的释放，而是缓存起来，后面申请内存的时候直接去之前缓存起来的地方去找
2. 缓存命中问题。CPU 中存在高速缓存，而同一个缓存，一次只能被一个线程访问。如果出现比较特殊的情况，如两个变量靠在一起：

	```cpp
	int a;
	int b;
	```

	然后 线程1 访问 a，线程2 访问 b。理论上此时可以并行。但是由于在加载 a 的时候，缓存把 b 的内存空间也加载进去，导致 线程2 在访问的时候，还需要重新加载缓存。这带来相当大的 CPU 周期浪费，被称为 “假共享”。


**《游戏引擎架构》** 一书中，对内存分配方案做出了相当多的描述，其重点提到的也是两个方面：

1. **通过内存池降低 malloc 消耗。**
2. **通过对齐降低缓存命中失败消耗。而虚幻引擎的目的相同，方法不同。**

`IntelTBB` 提供了：
- `scalable_allocator`：不在同一个内存池中分配内存，解决由于多线程竞争带来的无谓消耗；
- `cache_aligned_allocator`：通过缓存对齐，避免假共享。

> - 缓存行对齐：cache_aligned_allocator 在分配内存时，确保每个内存块的起始地址对齐到缓存行边界。这样，每个线程访问的数据都会位于不同的缓存行中，避免了假共享。
> - scalable_allocator：通过为每个线程分配独立的内存池，减少多线程环境下的锁争用，提高内存分配效率。
> - cache_aligned_allocator：通过缓存行对齐，避免假共享，减少缓存无效化，提升多线程程序的性能。

这一方案的代价是内存消耗量增加。对应其他平台的内存分配方案在这里不再做过多介绍。有兴趣的读者可以自行分析。

在虚幻引擎中，主要使用的还是 `scalable_allocator`。为方便读者参考，将 `FMallocTBB::Malloc` 的代码摘录如下：

> 下面代码截取自 UE4.27.2 引擎源码

```cpp
void* FMallocTBB::Malloc(SIZE_T Size, uint32 Alignment)
{
	void* Result = TryMalloc(Size, Alignment);

	if (Result == nullptr && Size)
	{
		OutOfMemory(Size, Alignment);
	}

	return Result;
}
```

> 可以追到函数内部去看看，这里就不放在这里讨论了

<br>

## 9.2 引擎初始化过程

> 虚幻引擎是怎么运行起来的？

### 引擎初始化简介

虚幻引擎初始化分为两个过程，即预初始化 `PreInit` 和 `Init`。其具体实现由 `FEngineLoop` 这个类来提供。而在不同平台上，入口函数不同（如 `Windows` 平台下是 `WinMain`，`Linux` 平台下是 `Main`），不同的入口函数最后会调用同样的 `FEngineLoop` 中的函数，实现跨平台。

<br>

#### 预初始化

`PreInit` 是预初始化过程，和初始化过程最显著的区别在于 `PreInit` 带有参数 `CmdLine`，也就是说，能够获得传入的命令行字符串。

这个过程主要是根据传入的命令行字符串来完成一系列的设置状态的工作，大致来说，能够分为这样几个设置的内容：

1. **设置路径**：当前程序路径，当前工作目录路径，游戏的工程路径。
2. **设置标准输出**：设置 GLog 系统输出的设备，是输出到命令行还是何处。

并且**也初始化了一部分系统**，包括：

- 初始化游戏主线程 `GameThread`，其实只是把当前线程设置为主线程。

- 初始化随机数系统（用过C语言随机数库的都知道，随机数是需要初始化的，否则同样的种子会产生出虽然随机但是一模一样的随机序列）。

- 初始化 `TaskGraph` 任务系统，并按照当前平台的核心数量来设置 `TaskGraph` 的工作线程数量。同时也会启动一个专门的线程池，生成一堆线程，用于在需要的时候使用。也就是说虚幻引擎的线程数量是远多于核心数量的。在该书作者的电脑上，初始化的线程数量是：`Task` 线程 3个，线程池线程 8个。

预初始化过程也会判断引擎的启动模式，是以游戏模式启动，还是以服务器模式启动。

在完成这些之后，会调用 `LoadCoreModules`。目前所谓的 `CoreModules` 指的就是 `CoreUObject`。具体为何 `CoreUObject` 需要这样额外的启动，会在分析 `UObject` 的时候进行分析。

随后，所有的 `LoadPreInitModules` 会被启动起来。这些强大的模块是：引擎模块、渲染模块、动画蓝图、Slate渲染模块、Slate核心模块、贴图压缩模块和地形模块。

当这些模块加载完毕后，`AppInit` 函数会被调用，进入引擎正式的初始化阶段。

<br>

#### 初始化

此时虚幻引擎进入初始化流程。所有被加载到内存的模块，如果有 `PostEngineInit` 函数的，都会被调用从而初始化。这一过程是借助 `IProjectManager` 完成的。

由于 `Init` 的过程被分摊到了每个模块中，因此初始化的过程显得格外简洁。

<br>

#### 主循环

虚幻引擎的主循环代码，如果为了符合虚幻引擎的源码引用规范（30行以内），那么可以被表述为以下这样：

```cpp
while( !IsEngineExitRequested() )
{
	EngineTick();
}
```

所以说虚幻引擎也是按照标准的引擎架构来书写的，至少游戏主线程是存在一个专门的引擎循环的，也就是 `EngineTick`。**注意，虚幻引擎的渲染线程是独立更新的**，不在我们主循环分析的内容中，可以看后文虚幻引擎渲染架构的分析。

引擎的 `Tick` 按照以下的顺序来更新引擎中的各个状态：

- 更新控制台变量。这些控制台变量可以使用控制台直接设置。
- 请求渲染线程更新当前帧率文字。
- 更新当前应用程序的时间，也就是 `App::DeltaTime`。
- 更新内存分配器的状态。
- 请求渲染线程刷新当前的一些底层绘制资源。
- 等待 `Slate` 程序的输入状态捕获完成。
- 更新 `GEngine`，调用 `GEngine->Tick`。
- 假如现在有个视频正在播放，需要等待视频播放完。之所以在 `GEngine` 之后等待，是因为 `GEngine` 会调用用户的代码，此时用户有可能会请求播放一个视频。
- 更新 `SlateApplication`。
- 更新 `RHI`。
- 收集下一帧需要清理的 `UObject` 对象。

有一些 `Tick` 的内容没有写在这里，具体希望了解有哪些内容，可以阅读 `LaunchEngineLoop` 中的内容。

那么现在大家最关心的内容其实集中在了，在 `GEngine->Tick` 中到底做了什么。其实这个内容也并不是非常复杂。只不过引擎考虑了相当多的情况，所以有许许多多的 If 判断，从而阻碍大家去理清。

从最正常的角度来说，`GEngine->Tick` 最重要的任务是更新当前 `World`。无论是编辑器中正在编辑的那个 `World`，还是说游戏模式下只有一个的那个 `World`。此时所有 `World` 中持有的 `Actor` 都会被得到更新。

> 请注意，作者并没有说 `UObject` 会得到更新，事实上有许多 `UObject` 根本没有 `Tick` 功能。

那么其他的 `Tick` 内容则是从时间分片的角度来看待很多任务。这个的意思是说，很多任务可能无法在一次 `Tick` 中完成，否则会卡死游戏主线程，带来极其不好的游戏体验，因此会分在多次 `Tick` 函数中完成。当前 `Tick` 需要完成什么任务，则是在 `Tick` 中根据诸多的状态来决定的。例如正在加载地图，则每次都 `Tick` 一下，加载一点点；正在**动态载入地图（StreamLevel）**，或是正在进行 `HotReload`，这些都会被分在不同的 `Tick` 中，以一次载入一点点的方式完成。

<br>
<br>

## 9.3 并行与并发

**虚幻引擎的系统一开始就被设计为并行化**，故对虚幻引擎并行、并发系统的研究非常重要，有助于理解接下来对渲染等过程的描述。同时，并行系统也是开发者手中的利器。

### 9.3.1  从实验开始

任何知识的学习都来源于实践，因此本节的最开头作者将会讲述如何快速配置一个实验环境，来极快地撰写测试代码。

你需要的是利用虚幻引擎的 `Automation System`，也就是自动化测试系统。具体的方式其实很简单。你需要在你的模块文件夹下建立一个 `Private` 文件夹，然后放入一个 `Test` 文件夹。在这个文件夹中，放置一个以 `Private` 文件夹中已有 `cpp` 的名字+Test.cpp 结尾的文件。比如以下这样：

```txt
/Private
	模块名.cpp
	/Tests
		模块名Test.cpp
```

然后打开你的 “模块名Test.cpp”，加入以下内容：

> 我这里是 `CommonToolTest.cpp`

```cpp
#include "CommonTool.h"

// #include ...

DEFINE_LOG_CATEGORY_STATIC(TestLog, Log, All);

IMPLEMENT_SIMPLE_AUTOMATION_TEST(FCommonToolTest, "MyTest.Public.FCommonToolTest", EAutomationTestFlags::EditorContext | EAutomationTestFlags::EngineFilter)

bool FCommonToolTest::RunTest(const FString& Parameters)
{
	UE_LOG(TestLog, Warning, TEXT("Hello"));
	return true;
}
```

重新生成一下 VS 项目文件

打开编辑器的 `Session Frontend` 窗口

![](/res/img/post/05-ue-daxiangwuxing-02/9-3-1.png)

![](/res/img/post/05-ue-daxiangwuxing-02/9-3-2.png)

勾选前面的白框然后点击 `Start Tests`。你会在 `Message Output` 窗口看到这样一行输出

![](/res/img/post/05-ue-daxiangwuxing-02/9-3-3.png)

恭喜，以后你就可以用这样的方式快速测试代码了。

<br>

### 9.3.2 线程

从实例开始

在虚幻引擎中，最接近于我们传统认为的 “线程” 这个概念的，就是 `FRunnable` 对象。`FRunnable` 也有自己的子类，可以实现更多的功能。不过如果只是为了演示多线程的话，继承自 `FRunnable` 就足够了。那么首先请先感受一下用法，实例代码如下：

```cpp
#include "CommonTool.h"

// #include ...

DEFINE_LOG_CATEGORY_STATIC(TestLog, Log, All);


class FRunnableTestThread : public FRunnable
{
public:
	FRunnableTestThread(int16 _Index)
		: Index(_Index)
	{
	}

public:
	virtual bool Init() override
	{
		UE_LOG(TestLog, Log, TEXT("Thread %d Init"), Index);
		return true;
	}

	// Inherited via FRunnable
	virtual uint32 Run() override
	{
		UE_LOG(TestLog, Log, TEXT("Thread %d Run:1"), Index);
		FPlatformProcess::Sleep(10.0f);

		UE_LOG(TestLog, Log, TEXT("Thread %d Run:2"), Index);
		FPlatformProcess::Sleep(10.0f);

		UE_LOG(TestLog, Log, TEXT("Thread %d Run:3"), Index);
		FPlatformProcess::Sleep(10.0f);

		return 0;
	}

	virtual void Exit() override
	{
		UE_LOG(TestLog, Log, TEXT("Thread %d Exit"), Index);
	}

private:
	int16 Index;
};


IMPLEMENT_SIMPLE_AUTOMATION_TEST(FCommonToolTest, "MyTest.Public.FCommonToolTest", EAutomationTestFlags::EditorContext | EAutomationTestFlags::EngineFilter)

bool FCommonToolTest::RunTest(const FString& Parameters)
{
	FRunnableThread::Create(new FRunnableTestThread(0), TEXT("Test0"));
	FRunnableThread::Create(new FRunnableTestThread(1), TEXT("Test1"));
	FRunnableThread::Create(new FRunnableTestThread(2), TEXT("Test2"));

	return true;
}
```
如果你成功运行的话，结果应该是这样：

![](/res/img/post/05-ue-daxiangwuxing-02/9-3-4.png)

如果将 `FPlatformProcess::Sleep(10.0f);` 改成 `FPlatformProcess::Sleep(0.01f);`

![](/res/img/post/05-ue-daxiangwuxing-02/9-3-5.png)

可以看到，所有的线程异步启动，并且非同步执行。这说明我们确实创建了三个线程，并运行它们

如果你实际创建线程，并且在 `Run` 函数中间下断点，你能够在 `Visual Studio` 的线程调试窗口看到你所创建的线程名称：

![](/res/img/post/05-ue-daxiangwuxing-02/9-3-6.png)

![](/res/img/post/05-ue-daxiangwuxing-02/9-3-7.png)

分析

首先我们声明了一个继承自 `FRunnable` 的类，并实现了三个函数：`Init`、`Run` 和 `Exit`。

`Init` 函数内放置初始化代码，`Run` 函数放置运行的代码，`Exit` 函数用于在线程退出的时候做清理工作。这些就是创建一个线程所需要完成的。事实上，你需要做的其实非常少（当然无法达到 `C++11` 那种极为简单的地步）。

那么在生成一个线程对象之后，接下来需要的是 `“启动这个线程”`。方法就是借助 `FRunnableThread` 的 `Create` 方法。第一个参数传入一个 `FRunnable` 对象，第二个参数则是线程的名字。

<br>

### 9.3.3 TaskGraph 系统

虚幻引擎的另一个多线程系统则更加的现代，是基于 `Task` 思想，对线程进行复用从而实现的系统。频繁创建和销毁线程的代价是很大的，但是很多时候我们创建的线程都是临时的，不会伴随我们的软件的整个生命周期。而为了达到并行化的目的，我们可以把需要执行的 `“指令”` 和 `“数据”` 封成一个包，然后交给 `Task Graph`，当 `Task Graph` 对应的线程有空闲，就会取出其中的 `Task` 执行。

从实践开始

同样地，让我们先来看一段示例性的代码，并看看运行结果：

```cpp
#include "CommonTool.h"

// #include ...

DEFINE_LOG_CATEGORY_STATIC(TestLog, Log, All);


class FShowcaseTask
{
public:
	FShowcaseTask(int16 _Index)
		: Index(_Index)
	{
	}

public:
	static const TCHAR* GetTaskName()
	{
		return TEXT("FShowcaseTask");
	}

	FORCEINLINE static TStatId GetStatId()
	{
		RETURN_QUICK_DECLARE_CYCLE_STAT(FShowcaseTask, STATGROUP_TaskGraphTasks);
	}

	static ENamedThreads::Type GetDesiredThread()
	{
		return ENamedThreads::AnyThread;
	}

	static ESubsequentsMode::Type GetSubsequentsMode()
	{
		return ESubsequentsMode::TrackSubsequents;
	}

	void DoTask(ENamedThreads::Type CurrentThread, const FGraphEventRef& MyCompletionGraphEvent)
	{
		UE_LOG(TestLog, Log, TEXT("Thread %d Run:1"), Index);
		FPlatformProcess::Sleep(0.01f);

		UE_LOG(TestLog, Log, TEXT("Thread %d Run:2"), Index);
		FPlatformProcess::Sleep(0.01f);

		UE_LOG(TestLog, Log, TEXT("Thread %d Run:3"), Index);
		FPlatformProcess::Sleep(0.01f);
	}

private:
	int16 Index;
};


IMPLEMENT_SIMPLE_AUTOMATION_TEST(FCommonToolTest, "MyTest.Public.FCommonToolTest", EAutomationTestFlags::EditorContext | EAutomationTestFlags::EngineFilter)

bool FCommonToolTest::RunTest(const FString& Parameters)
{
	TGraphTask<FShowcaseTask>::CreateTask(nullptr, ENamedThreads::GameThread).ConstructAndDispatchWhenReady(0);
	TGraphTask<FShowcaseTask>::CreateTask(nullptr, ENamedThreads::GameThread).ConstructAndDispatchWhenReady(1);
	TGraphTask<FShowcaseTask>::CreateTask(nullptr, ENamedThreads::GameThread).ConstructAndDispatchWhenReady(2);

	return true;
}
```

如果你下断点查看的话，能够发现，执行代码没有运行在一个单独的线程之中，而是运行在已有的线程 `TaskGraphThread2` 中

![](/res/img/post/05-ue-daxiangwuxing-02/9-3-8.png)

> 代码在 TaskGraphThread2 中执行，而不是在新的线程中执行

如果一切没有问题，你会得到这样的结果：

![](/res/img/post/05-ue-daxiangwuxing-02/9-3-9.png)

分析：首先需要明白的是，`Task Graph` 由于采用的是模板匹配，因此并不需要每个 `Task` 继承自一个指定的类，而是只要具有指定的几个函数，就能够让模板编译通过。这些函数就是：

- `GetTaskName`：静态函数，返回当前 `Task` 的名字。
- `GetStatId`：静态函数，返回当前 `Task` 的 `ID` 记录类型，你可以借助 `RETURN_QUICK_DECLARE_CYCLE_STAT` 宏快速定义一个并返回。
- `GetDesiredThread`：可以指定这个 `Task` 是在哪个线程执行，关于可选项有哪些，可以查看 `ENamedThreads::Type`。
- `GetSubsequentsMode`：这是 `Task Graph` 用来进行依赖检查的前置标记，可选包括 `Track Subsequents` 和 `Fire And Forget` 两个选项。前者表示这个 `Task` 有可能是某个其他 `Task` 的前置条件，所以 `Task Graph` 系统会反复检查这个 `Task` 有没有执行完成。后者表示一旦开始就不用去管了，直到执行完毕。也就不能作为其他 `Task` 的前置条件。
- `DoTask`：最重要的函数，里面是这个 `Task` 的执行代码。

而启动一个 Task 的方式是这样的：

```cpp
TGraphTask<FShowcaseTask>::CreateTask(nullptr, ENamedThreads::GameThread).ConstructAndDispatchWhenReady(0);
```

其实需要注意的是，这是一个连续函数调用：

`CreateTask` 的结果被调用了 `ContstructAndDispatchWhenReady`。而给 `Task` 传递的参数是在这个时候给出的。这里的 0 就是构造函数里面的 `ID`。这实质上是一种 `“延迟构造”` 的设计，因为当前的 `Task` 有可能依赖于其他的 `Task`，因此构造函数的调用会延迟到很久之后。

> 可能要更改为 `Debug Editor` 模式才能调试该代码

![](/res/img/post/05-ue-daxiangwuxing-02/9-3-10.png)

<br>

### 9.3.4 std::thread

作为 `C++11` 的特性之一，笔者其实是相当喜欢 `std::Thread` 的。而且写出来的代码也是最为简单、凝练且带有一种 `C++` 特有的气息。

从实践开始

这是目前最简单的多线程代码：

```cpp
#include "CommonTool.h"

#include <functional>
#include <thread>
// #include ...

DEFINE_LOG_CATEGORY_STATIC(TestLog, Log, All);

IMPLEMENT_SIMPLE_AUTOMATION_TEST(FCommonToolTest, "MyTest.Public.FCommonToolTest", EAutomationTestFlags::EditorContext | EAutomationTestFlags::EngineFilter)

bool FCommonToolTest::RunTest(const FString& Parameters)
{
	std::function<void(int16)> TaskFunction = [=](int16 Index) {
		UE_LOG(TestLog, Log, TEXT("Thread %d Run:1"), Index);
		FPlatformProcess::Sleep(0.01f);

		UE_LOG(TestLog, Log, TEXT("Thread %d Run:2"), Index);
		FPlatformProcess::Sleep(0.01f);

		UE_LOG(TestLog, Log, TEXT("Thread %d Run:3"), Index);
		FPlatformProcess::Sleep(0.01f);
	};

	new std::thread(TaskFunction, 0);
	new std::thread(TaskFunction, 1);
	new std::thread(TaskFunction, 2);

	return true;
}
```

可以看到输出的结果也是没有问题的：

![](/res/img/post/05-ue-daxiangwuxing-02/9-4-1.png)

分析

我们可以把这段代码分为两段，前半段是定义 `“指令”`，实质上是通过 `Lambda` 表达式，定义了一个 `“可执行对象”`。这就是 `functional` 这个头文件的用处。其统一了普通的函数，通过重载 `“()”` 操作符实现的 `“函数对象”`，以及由 `Lambda` 表达式定义的匿名函数。

后半段则是定义 `Thread`，其直接 `New` 出 `Thread` 对象，然后传递前面规定的指令，以及后边的参数。标准库的线程对象一旦被实例化，则立刻就会开始执行。

<br>

### 9.3.5 线程同步

我们已经能够创建线程来并发执行任务，那么接下来必须要对我们创建的线程进行一定程度上的控制，否则我们的线程就会像脱缰的野马一样狂飙：我们不知道它们什么时候已经完成了任务。因此引入线程的同步非常有必要。虚幻引擎吸纳了大量的线程同步思想，操作系统与 `C++11` 标准都有所吸纳。**总体而言，虚幻引擎的线程同步系统是分层级的。**

最底层的包括：`FCriticalSection` 临界区，这是实现线程同步的基石。其有一对 `Lock`、`Unlock` 操作原语，当一个线程将一个临界区 `Lock` 的时候，其他线程试图 `Lock` 就会阻塞，直到这个线程 `Unlock` 或者 `超时` 为止。而虚幻引擎的 `Mutex` 实现也是依赖于临界区的。这也是最符合传统多线程编程思想的多线程同步方式。但是代价就是有可能某一个线程需要轮询临界区，而且很多功能非常不方便实现（例如一个从子线程获取返回值，都必须要写一个忙等死循环，或者写成一个在 `Tick` 中不断查询的代码）。

而虚幻引擎也提供了更先进的线程同步方式，包括：

1. 投递 `Task` 到另一个线程的 `TaskGraph` 中，这样另一个线程就会去执行这个 `Task`，给人的感觉就像是 `“一个线程通知了另一个线程执行某一段代码”`，很符合人们的思维直觉。
2. 使用 `TFuture`、`TPromise`。这两个应该是来自于 `C++11` 标准和 `Boost` 标准库的思想，`C++11` 也提供了 `std::future` 和 `std::promise`。

这个思想对于笔者来说也是非常新鲜的。关于这两者的介绍，笔者将会在后文中详细叙说，这里只给出一个形象的例子：一个年轻的小伙子A，一个他的女神B。

小伙子这会儿正一无所有，但是他有志向。于是他向他的女神 `“承诺（Promise）”` 自己将会为女神买一辆宝马车。

于是女神调用了他的承诺的 `GetFuture` 函数，获得了一个 `“未来”` 值，然后和他在一起相处了起来。而女神也相信他，愿意给他一段时间。

经过了一段时间之后，假如这个小伙子足够努力，于是他真的买了一辆宝马车，通过 `Promise` 对象的 `SetValue` 函数把宝马车放到了 `Promise` 对象中，而有一天，当他的女神忽然想起，自己需要出门撑撑门面（或者什么其他理由都行），女神想起了她男朋友的承诺。于是她调用了 `“未来”` 的 `Get()` 来获取当年承诺的内容。果然她真的得到了一辆宝马车。于是她开心地出门了。

请回想一下之前的那个 `“获取子线程的计算结果”` 的案例，会发现这样一对泛型是非常方便的。尤其是你可以借助 `IsValid` 之类的函数查看你的宝马车是否到账的情况下，这样的编程模型几乎是大大简化了传统异步程序的设计。

在这一套高级同步系统的基础上，还有一个更简练的方案，就是虚幻引擎提供的 `Async` 模板。其提供了一段示例代码让笔者非常惊艳

```cpp
TUniqueFunction<int()> Task = []() {
	return 123;
};
	
auto Result = Async(EAsyncExecution::Thread, Task);
```

> 可以参考 `Async.h` 文件

返回值为一个 `TFuture` 对象。需要注意的是，`Async` 一定会在另一个非 `GameThread` 线程执行请求的任务，因此不能在任务中对 `UObject` 相关的类实例进行修改，否则会出现问题。

<br>

### 9.3.6 多进程

虚幻引擎同样提供了对进程的封装。在 `Core` 模块的 `GenericPlatformProcess` 中，可以看到 `FGenericPlatformProcess` 类的定义。其提供的 `CreateProc` 静态函数能够根据提供的 `URL` 启动一个进程，返回 `FProcHandle` 类型的进程句柄，并通过管道的形式进行数据交换。

> `AppInit` 并不是一个很短的函数，其有一系列初始化过程，但这并不是非常核心，所以不再加以笔墨讨论。

<br>
<br>

# 十、对象模型

## 10.1 UObject 对象

> **前情提要**
> 
> 在第一部分《对象》章节中，我们介绍了 `UObject` 对象的产生与销毁：
> 
> 产生： `UObject` 对象借助 `NewObject<T>` 函数产生，无法通过标准 `New` 操作符产生。
> 销毁： `UObject` 对象由虚幻引擎垃圾回收系统管理，不需要手动请求销毁。

### 10.1.1 来源

前文已经有所讲述，`UObject` 对象的初始化不能直接采用 `New` 操作符完成。而是必须要借助 `NewObject` 模板函数完成。读者可参考 `UObject` 初始化过程示意图，便可初步理解这个问题——因为对于 `UObject` 而言，**对象的初始化被分为了两个阶段：内存分配阶段 和 对象构造阶段**

![](/res/img/post/05-ue-daxiangwuxing-02/10-1-1.png)

<br>

#### 内存分配阶段

这个阶段虚幻引擎干涉了标准的内存分配过程。示意图只展示了 `“主流”` 情况，实际上有大量的判断来确保内存对齐、类默认对象不会重复创建等。但是总体来说，其获取当前 `UObject` 对象对应的 `UClass` 类的信息，根据类成员变量的总大小，加上内存对齐，然后在内存中分配一块合适的区域存放。除此之外也调用了 `UObjectBase` 原地进行构造。如果读者不大熟悉 `PlacementNew`，则可以理解为 `“不分配内存，直接假设当前内存指向的区域为一个已经分配好并与当前对象等大的内存区域，然后调用构造函数”`。这一过程相对来说比较好理解。

<br>

#### 对象构造阶段

这个阶段则显得麻烦一些。有读者会提问为何不直接使用 `PlacementNew` 完成内存构造，偏偏要多此一举，用 `UClass` 的构造函数指针 `ClassConstructor` 完成对象构造？请见下文。

是的，为何需要一个专门的、以 `FObjectInitializer` 为参数的函数指针完成对象构造？在解答之前，请允许笔者先解释这个指针是在何处赋值的：通常来说，该指针在反射生成的 `.generated.h` 中完成注册赋值。如果开发者自己实现了一个以 `FObjectInitializer` 为参数的构造函数，则直接将该函数指针指向一个静态函数 `__DefaultConstructor` ，定义如下：

```cpp
static void __DefaultConstructor(const FObjectInitializer& X) { new((EInternal*)X.GetObj())TClass; }
```

也就是说，笔者所言的 `“构造函数指针”` 是不妥的。实际上，你也无法直接获得一个类的构造函数指针。这个指针指向的是一个静态函数。这个函数的工作就是，获取当前 `FObjectInitializer` 对象持有的、指向刚刚构造出来的 `UObject` 的指针，然后对其调用 `PlacementNew`，传入 `FObjectInitializer` 作为参数调用构造函数，完成构造。如果开发者只是实现了一个无参数的构造函数，则直接调用无参数构造函数，完成构造。

此时读者肯定觉得，虚幻引擎设计者 `“智商掉线”`，绕了半天，最后还是走回了原来的路子，有何意义？这是因为虚幻引擎希望获得 `ClassConstructor` 供复用。也就是说虚幻引擎希望能够获得某个类的构造函数，就像一个点金手一样，划出一片内存，然后点一下就会模塑出一个类的对象。甚至在使用这个点金手的时候，不需要知道这个点金手所属的到底是哪个类 。如果使用 `PlacementNew` 方案就地构造，就必然需要传入类型参数作为模板参数，这无法满足虚幻引擎的需求。

总而言之，虚幻引擎采用了两步构造的机制，先分配内存，做出一些额外处理后再模塑对象。这也是为何不能直接给构造函数传递参数的原因：因为调用的 `__DefaultConstructor` 只能采用一个参数。这个函数是预先定义好的，故无法什么参数都传递。

<br>

### 10.1.2 重生：序列化

**序列化是指将一个对象变为更易保存的形式，写入到持久存储中。反序列化则反之，从持久存储中读取数据，然后还原原先的对象。** 通俗来说就是我们说的保存读取的过程。`UObject` 的序列化和反序列化都对应函数 `Serialize`。因为正向写入和反向读取需要按照同样的方式进行。

> 个人分享：在项目中我经常会序列化成 uint8，也就是二进制形式

需要注意的是，`序列化 / 反序列化` 的过程是一个 `“步骤”`，而不同于某些语言或者引擎的实现是一个完整的对象初始化过程。意思就是，即使是通过反序列化的方式从持久存储中读出一个对象，也是需要先实例化对象，然后才反序列化，而非通过一段数据，直接就能通过反序列化获得对象。

所以我们主要分析反序列化，这是因为当理解对象如何读取出来之后，写入就显得很简单了。实例化一个对象之后，传递一个 `FArchive` 参数调用反序列化函数，接下来具体过程如下：

1. 通过 `GetClass` 函数获取当前的类信息，通过 `GetOuter` 函数获取 `Outer`。这个 `Outer` 实际上指定了当前 `UObject` 会被作为哪一个对象的子对象进行序列化。
2. 判断当前等待序列化的对象的类 `UClass` 的信息是否被载入，没有的话：
   a. 预载入当前的类信息；
   b. 预载入当前类的默认对象 `CDO` 的信息。
3. 载入名字。
4. 载入 `Outer`。
5. 载入当前对象的类信息，保存于 `ObjClass` 对象中。
6. 载入对象的所有脚本成员变量信息。这一步必须在类信息加载后，否则无法根据类信息获得有哪些脚本成员变量需要加载。对应函数为 `SerializeScriptProperties`。
   a. 调用 `FArchive::MarkScriptSerializationStart` 函数，标记脚本数据序列化开始；
   b. 调用 `ObjClass` 对象的 `SerializeTaggedProperties`，载入脚本定义的成员变量；
   c. 调用 `MarkScriptSerializationEnd` 标记脚本数据序列化结束。

这是反序列化的大致框架。在逐个解释之前，请允许笔者先解释以下两个概念：

- 切豆腐理论

> 对于硬盘上保存的数据来说，其本身不具备 “意义”，其含义取决于我们如何解释这一段数据。我们每次反序列化一个 C++ 基本对象，例如一个 float 浮点数，我们是从豆腐最开头切下一块和浮点数长度一样大的豆腐，然后把这块豆腐解释为一个浮点数。那读者会问，你怎么知道这块豆腐就是浮点数？不是布尔值？或者是某个字符串？答案是，**你按照什么样的顺序把豆腐排起来的，并按照同样顺序切，那就能保证每次切出来的都是正确的。** 我放了三块豆腐：豆腐1（float）、豆腐2（bool）、豆腐3（double）。然后把它们按顺序排好靠紧，于是豆腐之间的缝隙就没了。我可以把这三块豆腐看成一块完整的大豆腐，放冰箱里冻起来。下次要吃的时候，我该怎么把它们切成原样？很简单，我知道豆腐1、豆腐2、豆腐3的长度，所以我在1号长度处切一刀，1号+2号长度处切一刀。妥了！三块豆腐就分开了。

- 递归序列化 / 反序列化

> 一个类的成员变量会有以下类型：C++基本类型、自定义类型的对象、自定义类型的指针。关于指针问题，我们将会在后文分析。这里主要分析前两者。第一个，对于 C++基本类型对象，我们可以用切豆腐理论序列化。那么自定义类型怎么办？毕竟这个类型是我自己定义的，我有些变量不想序列化（比如那些无关紧要的、可以由其他变量计算得来的变量）怎么解决？答案是，**如果需要序列化自定义类型，就调用自定义类型的序列化函数。** 由该类型自行决定。于是这就转化为了一棵像树一样的序列化过程。沿用前文的切豆腐理论。当我们需要向豆腐列表增加一块由别人定义的豆腐的时候，我们遵循谁定义谁负责的原则，让别人把豆腐给我们。切豆腐的时候，由于我们只知道接下来要切的豆腐的类型，却不知道具体应该怎么切，我们就把整块豆腐都交给那个人，让他自己处理。切完了再还给我们。

理解这两个概念之后，我们就能开始分析虚幻引擎的反序列化过程。这其实是一个两步过程，其中第一步可选：

1. 从另一块豆腐中加载对象所属的类信息，一旦加载完成就像获得了一张当前豆腐的统计表，这块豆腐都有哪几节，每节对应类型是什么，都在这张表里。
2. 根据统计表，开始切豆腐。此时我们已经知道每块豆腐切割位置和获取顺序了，还原成员变量简直如同探囊取物。

同时，**虚幻引擎也对序列化后的大小进行了优化**。我们不妨思考前文所述的切豆腐理论，如果我们完成序列化整个类，那么对于继承深度较深的子类，势必要序列化父类的全部数据。那么每个子类对象都必须占用较大的空间。有没有办法优化？我们会发现，其实子类对象很多数据是共同的，它们都是来自同样父类的默认值。这些数据只需要序列化一份就可以了。换句话说：

**虚幻引擎序列化每个继承自** `UClass` **的类的默认值（即序列化 CDO ），然后序列化对象与类默认对象的差异。** 这样就节约了大量的子类对象序列化后的存储空间。

接下来需要解决的问题是，自定义类型的序列化。即，对于对象指针，如何进行序列化？**直接序列化对象指针是完全不正确的：因为下一次加载时，内存地址不一定是一模一样的。** 那么将子对象完全序列化到自己体内呢？同样不可取，因为当两个对象同时引用同一个对象时，序列化会出现两份一模一样的对象。**对于虚幻引擎来说，解决这个问题的任务交给了序列化类，而不是 UObject 类本身。** 典型的案例可以参考虚幻引擎 `UPackage` 的存储过程。

<br>

#### UPackage 存储过程简介

`UPackage` 序列化和反序列化过程中涉及的主要概念如下：

- `UPackage` 显然涉及，不再做赘述。
- `UObject` 这里主要指 `UPackage` 中等待被序列化的对象。
- `UClass` 等待被序列化对象对应的类。
- `UProperty` 被 `UClass` 持有，存储这个类的属性（成员变量）信息。
- `FLinkerLoad` 继承自 `FArchive`，负责作为 “豆腐” 处理序列化和反序列化的过程。
- `CDO （Class Default Object）` 类默认对象。一个类会在内存中放置一个默认对象，包含默认值。

`UPackage` 需要序列化的对象其实是自身的元数据信息，以及包内的一系列对象，如图所示。这些对象有可能互相引用，有可能引用其他包的对象，此时 `UPackage` 面临前文所述的问题。

![](/res/img/post/05-ue-daxiangwuxing-02/10-1-2.png)

> `UPackage` 内部对象的复杂引用关系：包含内部对象之间的互相引用，也包含其他包中的对象的引用

如何将这样的复杂引用过程转化为更适合存储和读取的形式？虚幻引擎设计了这样的机制来完成，如图所示：

![](/res/img/post/05-ue-daxiangwuxing-02/10-1-3.png)

> `UPackage` 的结构示意图，可以看到头部的导入表和导出表

为了方便描述，举一个班级的例子。这个班级如大家高中读书的时候一样，充满了早恋的味道（对象之间互相引用，还引用了其他班的对象，甚至还有多角恋），而且老师为了解决同桌早恋问题，经常让全班换座位（每次加载内存地址都会改变）。此时有个新来的学生——小明，希望能够记录全班谈恋爱对象配对名单，如图所示：

<br>

#### 序列化

![](/res/img/post/05-ue-daxiangwuxing-02/10-1-4.png)

1. 在包最前方有两张表，导出表 `Export Table` 和导入表 `Import Table`，前者可以理解为本班人员名单，后者可以理解为隔壁班人员名单。
2. 当序列化当前包内一个对象的时候，遇到一个 `UObject` 指针怎么办？此时肯定不能直接序列化指针的值。这类似于小明在记录小王喜欢的对象小红的时候，不能直接记录小王喜欢的人的座位，否则第二天座位一变动，就出事了。此时小明灵机一动：
   a. 拿出导出表，往里面加了一项：1号-小红。
   b. 修改 “小王喜欢的对象” 字段，将其从一个座位编号，变成了导出表里面的一个项的编号：1号。
   c. 查看谁是小红真正的男朋友（ `NewObject` 的时候指定的 `Outer` ），到时候由他负责真正序列化小红。
   d. 继续存储其他有关的信息：如果遇到普通数据（小王的名字，`FName`；小王的年龄，`Int8`），就直接序列化，如果遇到 `UObject`，就重复第二和第三步。
3. 这时候小明发现小刚喜欢的对象居然是隔壁班的小花，小明无奈，只能再找出一张表：导入表，然后加了一项-1：小花（导入表项为负，导出表项为正，方便区分）。总不能把隔壁班的人也给序列化到本班的数据里面吧。然后把这项的编号替换到小刚喜欢的人的座位里。
4. 全部记录完毕，把两张表都保存起来，对象本身的数据则逐个排放在表后面，存放起来。

<br>

#### 反序列化

需要注意的是，虚幻引擎的序列化、反序列化系统追求尽可能的高效，如图所示，意味着不需要存储的数据绝不存储，**其实质上是先实例化对应类的对象，然后还原原始对象数据的过程。**

![](/res/img/post/05-ue-daxiangwuxing-02/10-1-5.png)

1. 小明大清早来到学校，走进教室一看，教室里空空如也，一个学生都没有（内存此时完全为空，没有任何对象信息）。完蛋了，老师马上要来上课了！想不到全班同学都翘课了！小明灵机一动，自己包里还有全班的数据呢！
2. 首先把表格全序列化出来，有了表格，其他都好说。
3. 从后面抽出一个对象，看看对象是什么类型，根据这个类型，把对象先模塑出来：
   > a. 如果 `UClass` 类型数据还没载入，先把 `UClass` 载入了，并把 `CDO` 给读取了。

   > b. 根据 `UClass` 信息，模塑一个对象——说通俗点，小明在全班第一排第一桌直接给造了一个假人出来，这个假人现在还什么特征都没有。
4. 根据这个对象的类信息，读取等大的数据，并根据类信息中包含的成员变量信息，判断这个成员变量的类型，执行以下步骤：
   > a. 假如是基础对象（名字，`FName`；年龄，`Int8`），就直接把这个对象给反序列化。此时这个假人拥有了名字和年龄。可能还有杂七杂八的属性，比如身高、体重、瞳距之类的。

   > b. 此时不可避免地遇到了 `UObject` 类型的成员变量。糟糕，想不到这个小子居然还搞早恋！怎么办，这会儿全班还缺人呢。小明此时一拍脑袋，想起来做记录的时候，这里存储的是两张表的表项。小明赶紧看了看这个成员变量是正还是负：为正 查 `Export Table` 导出表，看看这个对象有没有被序列化，如果有，就把对应对象的指针替换，否则就先造个假人丢在那里，等待此人的 `Outer` 负责实际序列化。相当于小明看了看，这表项有没有对应班上某个座位。没有的话，随便造个假人丢在某个空位置上，把座位号填到表项里，这样其他人如果也把这个假人当对象，就会直接指定到对应的位置。为负查 `ImportTable` 导入表，看看对应的 `Package` 有没有载入内存，没载入，就通知校长把隔壁班给喊来上课；如果已经载入了，小明就跑到隔壁班，大吼一声：“小花！小花坐哪儿？” 然后把获得的结果填入到表项里面。这就相当于小明趴在正在还原的这个同学耳朵边上，指着班上某个正坐在座位上的假人说：你记住，坐在那儿的就是你喜欢的对象！ 或者是：你记住，隔壁班第三排第二个就是你喜欢的对象！
5. 经过这一波折腾，全班会经历这样一个过程：
   > a. 首先班上会逐渐出现原来的同学和一些假人（已经被 `NewObject` 模塑出来，但是还没有根据反序列化信息恢复成原始对象的对象）。

   > b. 随后假人会逐渐被还原为原始对象。即随着读取的信息越来越多，根据反序列化后的信息还原成和原始对象一致的对象越来越多。

   > c. 最后全班所有人都会被恢复为和原始对象一致的人。也许小王同学喜欢的那个人的座位号变了（反序列化后指针的值被修正），但是新的座位号上坐着的人，是和他当年喜欢的小红一模一样的人。
6. 最后当老师跨进教室，只会说：啊，今天你们换过座位啦！ 然后旁若无人地上课。

通俗地来说就是这样的过程。从这个过程中，我们能获取到一些非常有趣的信息和经验：

- **序列化必要的、差异性的数据**

**不必要的引用不需要被序列化和反序列化。** 因此如果你的成员变量没有被 `UPROPERTY` 标记，其不会被序列化。如果你的这个成员变量值与默认值一致，也不会占用空间进行序列化。

- **先模塑对象，再还原数据**

这个过程笔者多次重点阐述，就是为了强调虚幻引擎的这个设计。先把对象通过 `NewObject` 模塑，然后还原差异性的数据。且被模塑出的对象会作为其他对象修正指针指向的基础。正如前文所言，小明不会因为小王的对象小红还没被序列化就束手无策，没被序列化就直接实例化一个假人丢在那，大不了以后读取到小红的数据时，把那个假人的信息改成和小红一样就好。

- **对象具有“所属”关系**

由 `NewObject` 指定的 `Outer` 负责序列化和反序列化。

- **鸭子理论**

叫起来像鸭子，看起来像鸭子，动起来像鸭子，那就是鸭子。说话像小红，看起来像小红，做事情像小红，那就是小红。也就是说，如果一个对象的所有成员变量与原始对象一致（指针的值可以不同，但指向的对象要一致），则该对象就是原始对象 。

<br>
<br>

### 10.1.3 释放与消亡

#### 销毁过程

`UObject` 对象无法被手动释放，只能被手动请求 `“ConditionalBeginDestroy”` 来完成销毁。如函数名称所示，具体是否销毁、何时销毁，取决于虚幻引擎，而非取决于请求者。实质上来说 `BeginDestroy` 函数只是设置了当前 `UObject` 的 `RF_BeginDestroyed` 为真，然后通过 `SetLinker` 函数将当前对象从 `linker` 导出表中清除。

在时机成熟时，由 `FinishDestory` 函数完成 `UObject` 销毁操作。其首先销毁非 C++ 的成员变量，对应函数 `DestroyNonNativeProperties`。其核心是一个 for 循环，获取当前 `UClass` 类的析构函数链表，调用每个析构函数的 `DestroyValue_InContainer` 函数，以完成自身的销毁。

```cpp
/** 
 *  Destroy properties that won't be destroyed by the native destructor
 */
void UObject::DestroyNonNativeProperties()
{
	// Destroy properties that won't be destroyed by the native destructor
#if USE_UBER_GRAPH_PERSISTENT_FRAME
	GetClass()->DestroyPersistentUberGraphFrame(this);
#endif
	{
		for (FProperty* P = GetClass()->DestructorLink; P; P = P->DestructorLinkNext)
		{
			P->DestroyValue_InContainer(this);
		}
	}
}
```

<br>

#### 触发销毁

前文已经叙述，客户代码只能请求销毁 `UObject` 而不能实际触发整个销毁过程。那么，什么样的情况会真正触发销毁呢？典型的情况是垃圾回收器来触发（ `StaticConstructUObject` 也会，但是不算典型情况）。

**垃圾回收器实际上执行两个步骤：析构和回收。** 前者负责调用析构函数，通知对象进行析构操作。后者则负责回收当前 `UObject` 占用的内存。这一段代码可以在 `GarbageCollection.cpp` 中的 `IncrementalPurgeGarbage` 函数找到，如图所示。

![](/res/img/post/05-ue-daxiangwuxing-02/10-1-6.png)

<br>
<br>

### 10.1.4 垃圾回收

本质上说，**垃圾回收（GC）** 应当作为 `UObject` 对象释放与消亡内容的一个子部分来进行阐述。但是由于垃圾回收部分涉及内容相当多，不适合作为一个小节来进行阐述，因此下面单独列出来对虚幻引擎的垃圾回收过程进行简单阐述。如果读者希望深入探索虚幻引擎的垃圾回收具体过程，并研究垃圾回收的源码实现，可以参考风恋残雪先生此文 👉[虚幻4垃圾回收剖析](https://www.cnblogs.com/ghl_carmack/p/6112118.html)

<br>

#### 垃圾回收算法简介

关于垃圾回收算法的研究持续了很多年，也有许多书籍是集大成之作，如果读者第一次接触垃圾回收算法，推荐阅读 **《垃圾回收算法与实现》** 一书。

垃圾回收算法简单来说是解决这样一个问题：在一个合租房内公共区域里面的物品，每个人原则上会去收拾自己的垃圾（即原则上父对象负责回收子对象释放时占用的内存空间）。但是在丢掉某个东西的时候，不知道别人有没有在用，贸然丢弃就会引发问题。例如我认为这个属于我的电磁炉已经不需要了，我准备买个新的，于是随手丢进了垃圾堆。过了一会儿舍友从房间里面出来，端着锅准备煮面，发现电磁炉没了（在别的对象持有当前对象引用的情况下释放对象，导致野指针）。为了避免这个情况，合租房的每个人只好采取一个保守的策略：大家都不丢垃圾，于是屋子里面堆满了垃圾。

此时就必须要采用一系列的方式来解决这样的问题，比较典型的算法有以下这些：

<br>

##### 引用计数算法

给每个东西上面挂一个数字牌。我要用的时候就加 1，不用了就减 1。一旦最后一个不用这个东西的人发现，减 1 之后为 0，于是知道这个东西没人需要用了，直接丢进垃圾堆。

- 优势
> 引用计数不需要暂停，是逐渐完成的。将垃圾回收的过程分配到运行的过程中。而不是一下子要求暂停所有人的生活，然后开始大扫除。

- 劣势
> 1. **指针操作开销** 每次使用都必须要调整牌子，频繁使用的物品翻来覆去地修改牌子数值是很大的一笔开销。
> 2. **环形引用** 假如是互相引用的对象，例如锅与锅盖配套，锅和锅盖互相引用，导致双方的牌子都是 1。但是实际上大家都准备把锅和锅盖一起换掉，却总误认为有人还需要锅和锅盖。

<br>

##### 标记-清扫算法

标记-清扫算法是追踪式 GC 的一种。追踪式引用计数算法会寻找整个对象引用网络，来寻找不需要垃圾回收的对象，**这与引用计数式的 “只关注单个对象” 的思路刚好相反。** 首先假定所有东西都是垃圾，然后从每个舍友（**在垃圾回收算法中称为根**）开始搜寻，即让每个舍友指认哪些东西是他需要的，最终剩下的没有被指认过的东西，那就是大家都不需要的垃圾，因此就可以直接回收。

- 优势
> 没有环形引用问题，即使锅盖和锅互相引用，只要大家都不用这俩，都会被作为垃圾丢掉。

- 劣势
> 1. **暂停** 必须要有人大喊一声 `“大家先别动，我们先搞一次大扫除！”` 然后才能开始这个算法，算法结束之后，大家才能继续做手头的事情。**这会导致系统具有明显延迟。**
> 2. **内存碎片** 如果完全按照刚才所述的情况实现，即只丢掉垃圾而不整理，就会导致可用空间越来越细碎，最终导致大型对象无法被分配。

> 个人补充：不过现在游戏引擎对内存碎片处理的比较好，比如用双向栈管理，大型对象和小型对象分别在两端...，或者用多个栈管理，把不同大小的分配在不同地方

针对标准算法存在的问题，必然有诸多改进方案。**虚幻引擎的智能指针系统采用的是引用计数算法，使用弱指针方案来（部分）解决环形引用问题**

> 程序员往往忘记去判断哪些是弱指针、哪些是强指针，从而导致内存释放的问题。 如果你编写 `Slate` 应用程序，一定要注意这个问题。

在讨论虚幻引擎的算法及其选择的原因之前，笔者先介绍一下 GC 的分类方式，以及虚幻引擎选择的理由，如表所示。

![](/res/img/post/05-ue-daxiangwuxing-02/10-1-7.png)

<br>

#### UObject 的标记清扫算法

如今虚幻引擎开始选择自己的垃圾回收算法。抛开智能指针系统暂且不谈，笔者揣测虚幻引擎根据以下思路选择了自己的方案：

1. `UClass` 包含了类的成员变量信息，类的成员变量也包含了 `“是否是指向对象的指针”` 的信息，因此具备选择精确式 `GC` 的客观条件。**也就是复用反射系统，来完成对每一个被引用的对象的定位。所以虚幻引擎选择精确式** `GC` 。
2. 虚幻引擎已经使用智能指针系统管理非 `UObject` 对象。此时由于 **有反射系统提供每一个对象互相引用的信息**（想象一张庞大的 `UML` 实例图，虚幻引擎通过 `UObject`、参考对应 `UClass`，就能还原出这张庞大的实例图），从而能够实现对象引用网络。故采用追踪式 `GC`。
3. 虚幻引擎在回收的过程中没有搬迁对象，应当是考虑到对象搬迁过程中修正指针的庞大成本。
4. **虚幻引擎选择了一个非实时 但是渐进式 的垃圾回收算法。** 将垃圾回收的过程分步、并行化，以削弱选择追踪式 `GC` 带来的停等消耗。

**虚幻引擎的垃圾回收在根源上是一个标记-清扫算法**，笔者用一幅调用图来简单整理其思路，我们会逐步补全和细化这张图：

![](/res/img/post/05-ue-daxiangwuxing-02/10-1-8.png)

借助 `GGarbageCollectionGuardCritical` `GCLock` / `GCUnlock` 函数，使得在垃圾回收期间，其他线程的任何 `UObject` 的操作都不会工作，从而避免出现一边回收一边操作导致的各种问题

回收过程对应函数为 `FRealtimeGC::PerformReachablilityAnalysis`，**其可以看作两个步骤：标记和清除。标记过程为：全部标记为不可达，然后遍历对象并引用网络来标记可达对象。** 清除过程也很简单，**直接检查标记，对没有被标记可达的对象调用 `ConditionalBeginDestroy` 函数来请求删除。**

![](/res/img/post/05-ue-daxiangwuxing-02/10-1-9.png)

虚幻引擎使用了大量的技术来加速 标记与回收过程，包括 `多线程` 和 `基于簇` 的方式，此处采用从简单到复杂的方式来阐述。

全部标记为不可达的算法很简单，虚幻引擎有个 `MarkObjectsAsUnreachable` 函数就是用来标记不可达的。借助 `FRawObjectIterator` 遍历所有的 `Object`，然后设置标记为 `Unreachable` 即可。

但是接下来这个步骤就比较麻烦，即遍历对象引用网络来标记可达对象。细分来说有两个问题：

1. 从哪里开始遍历？
2. 如何根据当前对象确定下一个遍历对象？

对于第一个问题来说，在之前标记不可达算法后，就把所有的 “必然可达” 对象都收集到了数组中。**这些必然可达对象刚开始大部分是那些被添加到 “根” 的对象。** 这些对象是一定不会被垃圾回收的，相当于前面举例的 “舍友” 。在收集的过程中，可达的对象还不断会被添加到这个数组中。

> **可以通过 `AddToRoot` 函数手动将 `UObject` 添加到根以避免被错误垃圾回收。**

接下来，会遍历这些必然可达对象，然后搜索这些必然可达对象能够到达的其他对象，从而标记所有可达对象。这些必然可达对象就好像是渔网最上面的一条浮子，浮子下面的就是一大堆的互相引用的对象。

众所周知，网络，或者更准确一些，有向图的遍历，是非常复杂的过程。传统方案是采用深度优先遍历或者广度优先遍历。**虚幻引擎面临的问题是去选择一套能够满足并行化需求的对象引用的遍历方案。**

**虚幻引擎采用的方案是：将二维的结构一维化来方便遍历。**

这需要一个前后端的配合，有点像是一个小型虚拟机的思路。每个 `UClass` 会生成一套以 `FGCReferenceInfo` 为成员的流 `FGCReferenceTokenStream`。这个流是紧凑排列的、类引用结构的信息。每一个 `FGCReferenceInfo` 结构会被存储为一个 `uint32` 的数据，这个微型结构包含以下字段：

- `ReturnCount` 返回深度计数，`8位` 用于统计当前需要返回的深度。如果是数组的最后一个元素，值为 1；如果是数组中一个结构体中的数组的最后一个元素，值为 2。以此类推。
- `Type` 类型，4位 当前 `Info` 的类型。
- `Offset` 偏移，20位 当前成员在对象中的内存偏移量。

> 加起来刚好 32 位，可以看作是虚拟机的一个字节码。

对每个 `UClass`，生成的是这样的一根像糖葫芦一样的 `FGCReferenceInfo` 数组，里面存储的其实是当前类对其他对象的引用结构。这个引用结构的一个例子如：

![](/res/img/post/05-ue-daxiangwuxing-02/10-1-10.png)

**前端**：这棵树的产生是通过遍历当前 `UClass` 的所有 `UProperty`，对每个 `UProperty` 调用 `EmitReferenceInfo` 函数，产生一个一个的 `Token`。对于每种类型的 `UProperty`，由对应的 `UProperty` 子类（例如 `UObjectProperty` ）实现 `EmitReferenceInfo` 来产生对应 `Token`。

**后端**：这棵树最后依然是被线性存储。形成一个一维线性结构，交给 `TFastReferenceCollector` 进行处理。这个 `Collector` 通过不断遍历这个线性结构，调用 `FGCReferenceProcessor` 来处理每一个引用。

`FGCReferenceProcesser` 是具体设置每一个对象标记的类。其 `HandleObjectReference` 函数具体处理对象引用，首先会将当前引用目标对象的可达性设置为可达，然后放入前文所叙述的必然可达数组中。

**这个设计的核心目的是为了加速：即由于线性结构遍历是非常容易进行并行化的，所以可以在合适的时机将垃圾回收任务并行化。** 遍历 `Token` 的时候，会根据实际情况，将前面提到的必然可达数组拆分为几个段落，对每个段落使用并行化的方式进行加速遍历。

而最终的清扫算法比标记算法的设计思路简单。**虚幻引擎采用了增量清扫的算法**，对应函数为 `IncrementalPurgeGarbage` 函数。清扫算法的过程在前文 `UObject` 的触发销毁章节进行了讲解。**所谓的增量清扫，是指虚幻引擎会考虑时间限制等，一段一段进行销毁的触发。** 需要注意的是，由于可能会在两次清扫时间之间产生新的 `UObject`，故在每次进入清扫时，需要检查 `GObjCurrentPurgeObjectIndexNeedsReset`，并根据实际情况，重新生成对象迭代器，避免迭代器失效。

> **基于簇的垃圾回收**
> **虚幻引擎采用了基于簇的垃圾回收算法**，用于加速 `cook` 后对象的回收。能够作为簇根的为 `UMaterial` 和 `UParticleSystem`。当垃圾回收阶段检查到一个簇根不可达，整个簇都会被回收，加速回收操作的速度，节省了再去处理簇的子对象的时间。

<br>
<br>

## 10.2 Actor 对象

> **前情提要：**
> 在第4章《对象》中，介绍了 `Actor` 对象的产生与销毁：产生 `Actor` 对象借助 `UWorld::SpawnActor<T>` 函数产生，需要获取一个 `UWorld*` 指针，才能产生 `Actor` 对象。销毁 `Actor` 对象可以借助 `Destory` 函数进行销毁，但是最终依然由虚幻引擎垃圾回收系统完成内存回收。

👉[Actor生命周期](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/unreal-engine-actor-lifecycle)

这幅图解释了虚幻引擎中最基础的对象 `Actor` 对象的生命周期。然而官方网站对 `Actor` 命周期的介绍只是寥寥几笔带过，并没有详细深入。因此，下面将会详细探讨 `Actor` 对象的整体生命周期。

![](/res/img/post/05-ue-daxiangwuxing-02/10-2-1.png)

<br>

### 10.2.1 来源

对比 `UObject` 对象，`Actor` 对象的来源是通过 `SpawnActor` 函数生成。最简单的一种函数原型为：

`World.h`

```cpp
/** Templated version of SpawnActor that allows you to specify a class type via the template type */
template< class T >
T* SpawnActor( const FActorSpawnParameters& SpawnParameters = FActorSpawnParameters() )
{
	return CastChecked<T>(SpawnActor(T::StaticClass(), NULL, NULL, SpawnParameters),ECastCheckedType::NullAllowed);
}
```

`SpawnActor` 只能够从 `UWorld` 调用，`Actor` 也只会从属于 `World`（演员必将依附所处的世界而存在）。换而言之，你无法直接 通过 `NewObject` 来生成出一个 `Actor`，但是 `AActor` 作为继承自 `UObject` 的类，其必然经历 `NewObject` 的构造过程，这个过程是在 `UWorld::SpawnActor` 中调用。在 `NewObject` 前后，虚幻引擎均进行了一系列的操作，从而保证 `Actor` 的正确构造。

<br>

#### 构造Actor之前的处理

主要是三个内容：

- 检查
  > 借助 `HasAnyClassFlags` 函数，通过 `Flag` 检查：当前类是否被废弃，当前类是否是抽象类，当前类是否真的继承自 `Actor`，传递的 `Actor` 的模板是否和当前 `Actor` 类型对应，是否在构造函数中创建 `Actor` 等。
- 获取 `Actor` 的模板 `Template`
  > 此模板不是 C++ 的那个模板，这个模板是指 `Actor` 的模塑基础。虚幻引擎允许原型式模塑 `Actor` 对象，即从一个已有对象中模塑出一个新的一模一样的对象，而非从类中模塑。如果没有提供模板，当前 `Actor` 所属类的 `CDO` 将会作为模板。
- 根据 `ESpawnActorCollisionHandlingMethod` 确定是否生成。（碰撞处理）
  > 使用蓝图生成 `Actor` 的读者都应该知道，其需要提供一个 `ESpawnActorCollisionHandlingMethod` 枚举值，用来指定当生成的 `Actor` 位置存在碰撞（即俗称 “挤到了”）的情况应该如何处理。这里就会根据提供的值进行处理：
  > - `SpawnParameters.bNoFail` 特殊标记值，该标记放于 `SpawnParameters` 中，如果设置为真，则忽略所有的碰撞处理，强制生成。
  > - `DontSpawnIfColliding` 在指定位置检测是否有几何碰撞，如果有就放弃生成对象。
  > 其他的处理方式，留到生成 `Actor` 后进行。

通过 `NewObject` 模塑 `Actor` 对应的代码为：

`LevelActor.cpp`

```cpp
// actually make the actor object
AActor* const Actor = NewObject<AActor>(
	LevelToSpawnIn,
	Class,
	NewActorName,
	ActorFlags,
	Template,
	false/*bCopyTransientsFromClassDefaults*/,
	nullptr/*InInstanceGraph*/,
	ExternalPackage
);
```

**请注意，第一个参数为 `Outer`，即负责序列化和反序列化的对象。** 从这里可以看出 `Actor` 由当前关卡（如果没有手动指定的话）负责存储信息。这里就会调用以 `const FObjectInitializer&` 为参数的构造函数，如果读者书写了很多 `Actor` 的代码，应该能够回忆起，正是在这个函数里面，我们通过 `CreateDefaultSubobject` 等函数构造出了默认的组件。相应的，组件的构造函数也在此时被调用。

<br>

#### 构造 Actor 后的处理

可以理解为两个步骤：

- 注册Actor
- 初始化Actor
  > 阅读这一段时可以参考 `Actor` 生命周期图中 `SpawnActor` 那一列，会更有顺序感。

1. 添加 `Actor` 到当前关卡的 `Actor` 列表中。
2. 调用 `PostSpawnInitialize` 函数。
   a. 获取 `Actor` 根组件以计算 `Actor` 的位置。
   b. 调用 `DispatchOnComponentsCreated` 函数，从而调用所有之前通过 `CreateDefaultSubobject` 函数构造出来的组件的 `OnComponentCreated` 函数。
   c. 通过 `RegisterAllComponents` 注册所有的 `Actor` 组件。实质上调用组件的 `RegisterComponent` 函数。所以，如果动态生成和添加组件，务必自己手动调用该函数以注册。
   d. 设置当前 `Actor` 的 `Owner`
	> `Owner` 与 `Outer` 不同。在网络同步中，`Owner` 用于指定当前 `Actor` 的主人，其中一个用处是：如果当前 `Actor` 的 `Owner` 是本地玩家，则不会从服务器端同步状态到本地，而是将本地状态向服务端同步。

   e. 调用 `PostActorCreated` 函数，通知当前 `Actor` 已经生成完成。在 `Actor` 中，这个函数只是一个空函数，开发者可以根据自己的需求在这里进行处理。
   f. 调用 `FinishSpawning` 函数：
	 > i. 调用 `ExecuteConstruction` 函数。如果当前 `Actor` 继承自蓝图 `Actor` 类，此时会拉出所有的蓝图父类串成一串，然后从上往下调用蓝图构造函数（此时顺便也会把 `Timeline` 组件生成出来）。最后调用用户在蓝图定义的构造函数脚本。也就是说，在蓝图中书写的构造函数实际上不是在构造阶段被调用！

	 > ii. 调用虚函数 `OnConstruction`，通知 C++ 层，当前脚本构造已经完成。开发者可以重载该函数，介入这一过程。
   
   g. 调用 `PostActorConstruction` 函数，该函数主要处理了组件的初始化过程。蓝图也有可能创建自己的组件，因此直到这时才获得了所有 `Actor` 已经被创建但是还没有初始化的组件，组件的初始化需要放到这里进行。
     > i. 调用 `PreInitializeComponents` 函数。对于 `Actor` 来说，这里只处理一件事情：如果当前 `Actor` 需要自动获取输入，则试图获取当前 `PlayerController`，然后启用输入；如果当前 `PlayerController` 不存在，则调用当前 `Level` 的 `RegisterActorForAutoReceiveInput` 以启用输入。该函数可以被重载以实现自己的逻辑。
	 
     > ii. 调用 `InitializeComponents` ，实质是遍历当前所有的组件，调用其 `InitializeComponent` 函数以通知初始化。这也是一个通知函数，开发者可以重载该函数以实现自己的初始化。父类只是简单设置 `bHasBeenInitialized` 为真。

	 > iii. 根据前文提到的碰撞处理方式，对当前 `Actor` 的位置进行处理。例如开发者希望能够在移动 `Actor` 位置后生成，则调整位置来让 `Actor` 能够顺利产生出来，不至于卡住。如果是产生不出来就摧毁的类型，那就摧毁掉 `Actor`，然后输出一个 `Log` 提示。

	 > iv. 调用 `PostInitializeComponents` 。对于 `Actor` 类来说，这里只是设置 `bActorInitialized` 为真，顺便向寻路系统注册自己，然后调用 `UpdateAllReplicatedComponents` 函数。这个函数同样可以重载以实现自己的逻辑。

	 > v. 调用著名的 `BeginPlay` 函数。这是这个 `Actor` 的第一声 `“啼哭”`，即第一次 `Tick`。接下来这个 `Actor` 将会进入自己的 `Tick` 循环。

对于 `Actor` 的创建来说，最大的问题在于处理 `Actor` 所属组件的创建。不仅仅 C++ 层会有默认的组件，蓝图层也会有组件产生。因此，对组件的初始化会延后到所有组件被注册到世界之后进行。需要注意的是，`Actor` 默认的 `BeginPlay` 函数会通知蓝图层的 `BeginPlay`，换句话说，**如果你重载后首先调用 `Super::BeginPlay`，会发现蓝图 `BeginPlay` 会首先调用，然后才是自己 C++ 层的 BeginPlay ，顺序正好相反。**

<br>

### 10.2.2 加载

`Actor` 作为 `UObject` 的子类，其加载过程前部分依然遵循 `UObject` 的加载过程。`Actor` 只会调用自己的 `PostLoad` 函数来做一些处理，不同的子类有不同处理方式，对于 `Actor` 基类来说，该函数主要实现：如果当前 `Actor` 存在 `Owner`，添加自己到 `Owner` 的 `Children` 数组中。由于 `Children` 数组虽然是 `UProperty`，但是被标记为 `transient`，不会参与到序列化中。

请注意，如果是在编辑器中，到这一步，序列化已经完成，直到进入 `PIE` 模式才会继续进行 `Actor` 初始化过程。读者可能希望了解游戏世界的初始化过程，其大致过程如下：

`UWorld::InitializeActorsForPlay` 该函数依序调用以下函数：

- `UWorld::UpdateWorldComponents` 添加属于世界（但是不属于任何 `Actor`）的组件，例如绘制线条用的 `PersistentLineBatcher` 等。同时如果有其他的子关卡，这里开始更新子关卡的组件。
- `GEngine->SpawnServerActors` 根据服务端的 `Actor` 信息，在本地生成对应 `Actor`。
- `AGameMode::InitGame` 调用当前游戏模式的初始化函数。
- `ULevel::RouteActorInitialize` 遍历所有关卡，对 `Actor` 进行初始化。请注意，此时 `Actor` 的组件都是已经生成完毕了的，序列化的时候已经存储了足够的数据。此时需要进行的仅仅是通知初始化。故该函数遍历关卡中的所有 `Actor`，执行顺序如下：
  - `AActor::PreInitializeComponents`
  - `AActor::InitializeComponents`
  - `AActor::PostInitializeComponents`
  - `AActor::BeginPlay`
- `ULevel::SortActorList` 对所有关卡（包括子关卡）的 `Actor` 对象进行排序，准确的说应当是整理。这里是针对网络设计的，将 `Actor` 按照是否需要网络同步，放入对应数组进行整理。
- `UNavigationSystem::OnInitializeActors` 通知寻路系统开始初始化。
- `UAISystem::InitializeActorsForPlay` 通知 `AI` 系统初始化。

**正是由于 `UObject` 序列化机制非常强大，因此 `Actor` 能够比较完整地保存当前的状态。** 当加载后，只需要进行有限的初始化工作，就能够还原 `Actor` 的工作状态。同时，这一段流程也描述了，如果希望在加载和创建 `Actor` 的时候均执行某段函数 ，应当如何重载函数完成。简而言之：

1. 希望在 `Actor` 生成的时候触发，但是不希望在 `Actor` 加载的时候触发，可以重载 `PostActorCreated` 这样的函数，具体时机可以查看前面的函数调用序列，主要取决于是否需要假定组件均已经初始化。
2. 希望在 `Actor` 加载的时候触发，可以重载 `PostLoad` 函数。
3. 希望均触发，可以按需求考虑重载 `BeginPlay` 函数。

<br>

### 10.2.3 释放与消亡

在第一部分已经阐述，`Actor` 的释放和销毁通过手动请求 `Destroy` 实现。调用一个 `Actor` 的 `Destroy` 函数销毁 `Actor` 的过程如下：

`UWorld::DestroyActor`：`Actor` 的 `Destroy` 函数只是一个辅助函数，实际上是请求当前世界来摧毁自己。该函数完成：

- `Actor::Destroyed`
  > - 调用该函数通知本 `Actor` 已经被摧毁。虽然这个 `Actor` 尚未被垃圾回收，但是在这一步发送通知，意味着在此之后不能再访问本 `Actor`。这里也会在网络上通知本 `Actor` 被摧毁。
  > - 取消绑定子 `Actor` 虽然父级 `Actor` 被删除了，但是子 `Actor` 不应当被同样删除，因此需要从父级 `Actor` 上取消绑定。
- `Actor::SetOwner(NULL)`：将 `Actor` 的 `Owner` 设置为空，不再参与服务器同步。
- `UWorld::RemoveActor`：从当前世界的 `Actor` 列表中移除当前 `Actor`。
- `Actor::UnregisterAllComponents`：通知所有的组件，从世界中取消注册。
- `Actor::MarkPendingKill`：设置 `PendingKill` 这个标记，表示当前 `UObject` 需要被删除。
- `Actor::MarkPackageDirty`：设置最外侧的 `Package` 为 `Dirty`。
- `Actor::MarkComponentsAsPendingKill`：对当前 `Actor` 的所有子组件添加 `PendingKill`。
- `Actor::RegisterAllActorTickFunctions`：取消注册所有的 `ActorTick` 函数。

- 触发垃圾回收：当下一次垃圾回收过程开始的时候，会检测到这个 `Actor` 已经没有任何引用（已经从 `ULevel` 的 `ActorList` 中被取下），且被设置为 `PendingKill`。于是进入到 `UObject` 的垃圾回收流程。

**由此可见，`Actor` 的释放过程主要有两个重要任务：**

1. **通知服务器在服务端和各客户端之间删除当前 `Actor`。**
2. **从世界 `Actor` 列表等取消注册自己，从而完全删除当前 `Actor` 和世界的关联，从而满足垃圾回收的要求。**

<br>
<br>

The End.
