---
title: 【UE5】UBT的具体工作流程
tags: [UE5]
date: 2025-03-05
updated: 2025-03-05
cover: /res/img/post/09-ue5-ubt-workflow/cover.jpg
top_img: /res/img/site/background.png
---

![](/res/img/post/09-ue5-ubt-workflow/cover.jpg)

<br>

# 前言

> 一知半解永远无法深入.

<br>
<br>

# 一、前置知识

## GenerateProjectFiles

UE 中构建项目之前，都需要先执行 GenerateProjectFiles，这样 UE 会通过 mono 执行 UnrealBuildTool (UBT) 收集所有模块的 Build.cs，然后生成各种编译所需要的文件。比如 *.generated.h 、 *.gen.cpp 等等。

在不同平台、不同 IDE 下还需要生成不同结构的配置，因为 Xcode、VS、Rider 的工程配置文件是各不相同的，UBT 源码中需要分类去处理。

以 VS 为例，UBT 会为 VS 生成 .sln 以及各种模块 .vcxproj 文件，里面包含了各个模块的宏定义以及 include path，都是由 Build.cs 生成出来的。这样 IDE 在打开工程时才知道模块的宏是什么，引用的路径有哪些，也就是常说的 IntelliSense 功能。当然，更重要的是能让编译器在编译时找到。

<br>

## UBT 命令行参数

>官方文档没有准确给出这部分内容 (命令行参数)

GenerateProjectFiles.bat 脚本是 Unreal Build Tool 的简单包装器，它以特殊模式启动，用于构建项目文件，而不是构建程序可执行文件。它使用 -ProjectFiles 命令行选项调用 Unreal Build Tool。

<!-- 一些命令行参数可以参考这篇文章：✨[IDE 的项目文件 (UE官方文档)](https://dev.epicgames.com/documentation/en-us/unreal-engine/how-to-generate-unreal-engine-project-files-for-your-ide) -->

**这里提供一种比较靠谱查看命令的方式，编译 log 里最开始会显示用到的命令**:

![](/res/img/post/09-ue5-ubt-workflow/1.png)

也可以使用 `.\UnrealBuildTool.exe -Help` 查看部分参数：

```shell
PS C:\Dev\UE\SourceCode\UE5.5.3\Engine\Binaries\DotNET\UnrealBuildTool> .\UnrealBuildTool.exe -Help
Global options:
  -Help                :  Display this help.
  -Verbose             :  Increase output verbosity
  -VeryVerbose         :  Increase output verbosity more
  -Log                 :  Specify a log file location instead of the default Engine/Programs/UnrealBuildTool/Log.txt
  -TraceWrites         :  Trace writes requested to the specified file
  -Timestamps          :  Include timestamps in the log
  -FromMsBuild         :  Format messages for msbuild
  -SuppressSDKWarnings :  Missing SDKs error verbosity level will be reduced from warning to log
  -Progress            :  Write progress messages in a format that can be parsed by other programs
  -NoMutex             :  Allow more than one instance of the program to run at once
  -WaitMutex           :  Wait for another instance to finish and then start, rather than aborting immediately
  -RemoteIni           :  Remote tool ini directory
  -Mode=               :  Select tool mode. One of the following (default tool mode is "Build"):
                            AggregateClangTimingInfo, AggregateParsedTimingInfo, Analyze, ApplePostBuildSync, Build,
                            ClRepro, Clean, Deploy, Execute, FixIncludePaths, GenerateClangDatabase, GenerateProjectFiles,
                            IOSPostBuildSync, IWYU, InlineGeneratedCpps, JsonExport, PVSGather, ParseMsvcTimingInfo,
                            PipInstall, PrintBuildGraphInfo, ProfileUnitySizes, Query, QueryTargets, Server, SetupPlatforms,
                            Test, UnrealHeaderTool, ValidatePlatforms, WriteDocumentation, WriteMetadata
  -Clean               :  Clean build products. Equivalent to -Mode=Clean
  -ProjectFiles        :  Generate project files based on IDE preference. Equivalent to -Mode=GenerateProjectFiles
  -ProjectFileFormat=  :  Generate project files in specified format. May be used multiple times.
  -Makefile            :  Generate Makefile
  -CMakefile           :  Generate project files for CMake
  -QMakefile           :  Generate project files for QMake
  -KDevelopfile        :  Generate project files for KDevelop
  -CodeliteFiles       :  Generate project files for Codelite
  -XCodeProjectFiles   :  Generate project files for XCode
  -EddieProjectFiles   :  Generate project files for Eddie
  -VSCode              :  Generate project files for Visual Studio Code
  -VSMac               :  Generate project files for Visual Studio Mac
  -CLion               :  Generate project files for CLion
  -Rider               :  Generate project files for Rider
  -TelemetryProvider   :  List of ini providers for telemetry
  -Session             :  Session identifier for this run of UBT, if unset defaults to a random Guid
```

也可以直接看想要编译的项目的配置页面中启动 UBT 的命令行：

![](/res/img/post/09-ue5-ubt-workflow/2.png)

这些参数会直接传递给 UnrealBuildTool.exe。

<br>
<br>

# 二、拉取并编译引擎源码

正常我们调用 UnrealBuildTool 的时机一般是 VS 中点击 Build、引擎中点击 Compile、AutomationTool 或者命令行直接调用 UnrealBuildTool

为了调试 UBT，我们需要 Debug 版本的 UBT 工具，因此需要拉取对应的引擎源码自己编译，这里用的是官方的 _**5.3.3**_ 的 release 版本，可以通过在 clone 时加入 "--depth=1" 参数来缩短拉取时间，如果已经拉取编译过引擎可以跳过这一部分

完成拉取之后，我们依次执行 Setup.bat、GenerateProjectFiles.bat，然后打开 UE5.sln 点击 Build 进行引擎编译 (通常1小时左右)

编译完成之后我们运行引擎随意创建一个 C++ 工程用来观察，然后就可以开始调试了。

<br>
<br>

# 三、调试 UBT

>下面的测试环境是一个随便创建的游戏项目

一般默认编译的配置为 Development，我们选中 Program 中的 UnrealBuildTool，右键设为启动项目，并设置 Configuration 为 Debug，然后右键 Build 一次。

Build 完之后，我们打开 UnrealBuildTool 项目对应的属性，填入需要的命令行参数，这里主要是指定需要编译的项目名、平台、Configuration 类型、项目文件路径、额外参数，与我们从命令行直接调用对应：

![](/res/img/post/09-ue5-ubt-workflow/3.png)

```cpp
-Target="GoodGameEditor Win64 Development -Project=\"C:\Dev\UE\SourceCode\Project\UE5.5.3\GoodGame\GoodGame.uproject\"" -Target="ShaderCompileWorker Win64 Development -Project=\"C:\Dev\UE\SourceCode\Project\UE5.5.3\GoodGame\GoodGame.uproject\" -Quiet" -WaitMutex -FromMsBuild -architecture=x64
```

然后我们可以点击上方运行正式开始调试，观察其执行过程/

<br>
<br>

# 四、流程分析

找到 UnrealBuildTool.cs 中的 Main 函数，我们在这里断点：

![](/res/img/post/09-ue5-ubt-workflow/4.png)

这里会根据传入的 Mode 创建对应的 ToolMode，_**默认为 BuildMode**_，然后 CreateInstance 创建实例并调用 ExecuteAsync

![](/res/img/post/09-ue5-ubt-workflow/5.png)

ExecuteAsync 会设置对应的 Log 文件路径，读取 BuildConfiguration 的具体设置，并进一步解析处理命令行传入参数生成 TargetDescriptors，然后调用 BuildAsync

![](/res/img/post/09-ue5-ubt-workflow/6.png)

_**BuildAsync 函数即为我们关键的入口函数，在这里创建 Makefile**_，并在后续调用 BuildAsync(TargetMakefiles.ToArray(), ...)，这里可以同时编译多个 Target

![](/res/img/post/09-ue5-ubt-workflow/7.png)

我们具体观察分析 CreateMakefileAsync 函数，这里主要处理 Makefile 文件生成的逻辑

>Creates the makefile for a target. If an existing, valid makefile already exists on disk, loads that instead.
>为目标创建 makefile。如果磁盘上已经存在有效的 makefile，则加载该 makefile。

下面我们看 CreateMakefileAsync 需要创建 makefile 的部分：

![](/res/img/post/09-ue5-ubt-workflow/8.png)

总的来说主要是先创建 UEBuildTarget，然后执行 Target.BuildAsync 生成 TargetMakefile

- 创建目标：UEBuildTarget.Create(...)
- 创建预构建脚本：Target.CreatePreBuildScripts()
- 执行预构建脚本
- 构建目标：Target.BuildAsync(...)
- 将预构建脚本保存到 makefile 中
- 保存其他命令行参数
- 保存环境变量
- 保存 makefile 以备下次使用

<br>
<br>

# 五、调用 UHT 的时机

UBT 执行过程中，会对所有的 UObjectModules 执行 UHT 生成对应类的辅助头文件 ( generated.h )，需要先收集哪些 Module 和 Header 需要生成。

UObjectModules 的收集逻辑在 _**ExternalExecution.SetupUObjectModules**_ 中，有用到一个线程池队列 _**ThreadPoolWorkQueue**_ 来多线程收集 Module 中所有 .h 文件的信息：

>为 UHT 报头生成准备缓存数据

![](/res/img/post/09-ue5-ubt-workflow/9.png)

>并行创建每个模块的信息

![](/res/img/post/09-ue5-ubt-workflow/10.png)

对于属于 UObject 的类放入 PublicUObjectHeaders、PrivateUObjectHeaders 等列表中，判断的依据为在文件内容中使用正则表达式匹配对应的关键字。


如果匹配到，则记录下对应的 Header 信息，并将所在的 Module 加入到 UObjectModules 中：

```cpp
// If the target needs UHT to be run, we'll go ahead and do that now
if (Makefile.UObjectModules.Count > 0)
{
	await ExternalExecution.ExecuteHeaderToolIfNecessaryAsync(BuildConfiguration, TargetDescriptor.ProjectFile, Makefile, TargetDescriptor.Name, WorkingSet, Logger);
}
```

判断是否要执行 UHT，需要满足几个条件之一：
- UHT 工具本身没有编译过
- BuildConfiguration.bForceHeaderGeneration 指定强制执行
- Module 的生成时间是否过期

<br>

>下面代码截取自源代码 UE5.3.3

断点进入 ExecuteHeaderToolIfNecessaryInternalAsync 函数，找到如下代码：

```cpp
// ensure the headers are up to date
// 确保头文件是最新的
bool bUHTNeedsToRun = false;
if (BuildConfiguration.bForceHeaderGeneration)
{
	bUHTNeedsToRun = true;
}
else if (AreGeneratedCodeFilesOutOfDate(BuildConfiguration, Makefile.UObjectModules, CompositeTimestamp, Logger))
{
	bUHTNeedsToRun = true;
}

// Check we're not using a different version of UHT
// 检查我们没有使用不同版本的UHT
FileReference ModuleInfoFileName = GetUHTModuleInfoFileName(Makefile, TargetName);
FileReference ToolInfoFile = ModuleInfoFileName.ChangeExtension(".uhtpath");
if (!bUHTNeedsToRun)
{
	if (!FileReference.Exists(ToolInfoFile))
	{
		bUHTNeedsToRun = true;
	}
	else if (FileReference.ReadAllText(ToolInfoFile) != UhtAssemblyPath)
	{
		bUHTNeedsToRun = true;
	}
}

// Get the file containing dependencies for the generated code
FileReference ExternalDependenciesFile = GetUHTDepsFileName(ModuleInfoFileName);
// 确定所生成代码的任何外部依赖项是否过期
if (AreExternalDependenciesOutOfDate(ExternalDependenciesFile))
{
	bUHTNeedsToRun = true;
}
```

<br>
<br>

# 六、编译 Action 任务生成

Action 即为最终分配到 CPU 多个核心执行的具体任务，我们通过调试找到 Makefile.Actions 具体变化的位置如下：

![](/res/img/post/09-ue5-ubt-workflow/11.png)

可以判断 Action 的生成是在 Binary.Build 中，继续进入其中调试，观察调用过程，最终执行到 SetupBinaryLinkEnvironment 函数：

![](/res/img/post/09-ue5-ubt-workflow/12.png)

进入 SetupBinaryLinkEnvironment 查看具体逻辑，这里的 Action 生成依赖于 UEToolChain 具体子类的规则，在 PC 平台下为 VCToolChain

![](/res/img/post/09-ue5-ubt-workflow/13.png)

这里会遍历所有的 Module，并针对具体的子类调用 Module.Compile

根据 UEBuildModule 的类型和 ToolChain 的类型，一个 Module 可能会生成多个 Aciton，比如 VCToolChain 处理 UEBuildModule 基类会在 GenerateTypeLibraryHeader 中添加两个 Action

![](/res/img/post/09-ue5-ubt-workflow/14.png)

查看 Modules 列表中的 Module，可以看出有多种类型：

![](/res/img/post/09-ue5-ubt-workflow/15.png)

>这版调试生成了 4562 个不同的 Action

<br>

大部分关键代码其实都在 _**BuildAsync**_ 函数中，例如下面的合并处理 Action：

![](/res/img/post/09-ue5-ubt-workflow/16.png)

<br>
<br>

# 七、流程总结

终于到尾声了，兴趣使然，也算满足自己的好奇心了。

<br>

- 开始调用 UBT 进行编译
- 解析命令行参数，生成 TargetDescriptor
- 根据 TargetDescriptors 创建 Makefile ( CreateMakefile )
  - 创建 UEBuildTarget ( UEBuildTarget.Create )
    - 收集 Target.cs 和 Build.cs 信息
    - 创建 TargetRules 对象 RulesObject ( RulesAssembly.CreateTargetRules )
    - 读取 Descriptor 设置 RulesObject 属性
    - 根据 RulesObject 创建 UEBuildTarget
    - PreBuildSetup 收集 Binaries、BuildPlugins 等
  - 执行 Target.Build 生成 TargetMakefile
    - 收集 Target 中的 UEBuildModule、UEBuildPlugin 信息
    - 执行 UHT 生成代码
- 根据 Makefile 正式开始 Build
  - 合并处理 Action
  - 执行 Action

<br>

>整个流程中使用了大量的 C# 容器筛选和反射特性，猜测正是由于这些方便的特性 Epic 才使用了 C# 来编写 UBT 工具

<br>
<br>

# 参考

[1] ✨[【UE4源代码观察】尝试调试UBT](https://blog.csdn.net/u013412391/article/details/106150390)
[2] ✨[Unreal Buuild Tool (UE官方文档)](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/unreal-build-tool-in-unreal-engine)
[3] ✨[IDE 的项目文件 (UE官方文档)](https://dev.epicgames.com/documentation/en-us/unreal-engine/how-to-generate-unreal-engine-project-files-for-your-ide)
[4] ✨[UE4 UBT执行流程浅析](https://zhuanlan.zhihu.com/p/561178256)
[5] ✨[The Unreal Build System Explained | Inside Unreal (Youtube)](https://www.youtube.com/watch?v=GJZUV8homoo)

<br>
<br>

The end.
