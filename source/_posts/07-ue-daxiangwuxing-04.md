---
title: 【UE大象无形总结】第二部分虚幻引擎浅析【下卷】
tags: [UE4]
date: 2025-01-06
updated: 2025-01-06
cover: /res/img/post/07-ue-daxiangwuxing-04/cover.png
top_img: /res/img/site/background.png
---

![](/res/img/post/07-ue-daxiangwuxing-04/cover.png)

<br>

# 前言

> **恭迎大帅！**
> 
> 参考书籍：《大象无形：虚幻引擎程序设计浅析 (罗丁力 [罗丁力])》
> 补充说明：本系列是个人对书中内容的实践

<br>
<br>

# 十二、Slate界面系统

作为核心界面系统，`Slate` 是一个跨平台的、硬件加速的图形界面框架，采用了 `C++` 的泛型编程来允许直接使用 `C++` 语言撰写界面。

<br>

## 12.1 Slate的两次排布

`Slate` 是一个分辨率自适应的相对界面布局系统，为了能够完成这样的目的，`Slate` 实质上采用了一个 “两次” 排布的思路。

1. 首先，递归计算每个控件的大小，父控件会根据子控件来计算自己的大小。
2. 然后，根据控件大小，具体计算出每个控件的绘制位置。

由于有部分控件是 “大小可改变” 的，因此必须先计算出 “固定大小”，才可能实际排布这些控件。

<br>

## 12.2 Slate的更新

`Slate` 系统的更新，实质上是与引擎内部更新分开的。`FEngineLoop` 类会首先更新引擎，随后在确保 `FSlateApplication` 已经初始化后，调用 `FSlateApplication` 的 `Tick` 函数进行更新。

`FSlateApplication` 代表了当前正在运行的 `Slate` 程序，F开头意味着这个类本身并非一个 `Slate` 组件，其只是行使 “管理” 的责任。

从实际行为上看，`FSlateApplication` 也确实是负责绘制当前窗口，当自身的 `Tick` 函数被调用的时候，它会请求更新所有的 `Window`，即调用 `TickPlatform` 函数。随后才会调用 `TickAndDrawWidgets` 系列的函数，去绘制所有的对象。

<br>

## 12.3 Slate的渲染

需要注意的是，`Slate` 的渲染并非是一个 “递归渲染” 的过程，而是一个 **“先准备，再渲染”** 的过程。

所有的 `Slate` 对象将会首先准备需要渲染的内容，即 `WindowElement`。然后这些内容会被交给 `SlateRHIRenderer` 负责，最终被绘制出来。

`Slate` 同样是借助虚幻引擎的 `RHI` 硬件绘制接口进行绘制的。所以 `Slate` 的绘制实际上走的是标准的渲染流程：

1. 将控件对象转换为需要绘制的图形面片。
2. 通过 `PixelShader` 和 `VertexShader` 来使用 `GPU` 进行绘制。
3. 拿回绘图结果，显示在 `SWindow` 中。

值得强调的是，`Slate` 的渲染是没有开启深度检测的。而且在 `SlateVertexShader` 中，每个渲染对象的 Z轴均会被设置为 0，这意味着：

1. 控件对象简单地按照绘制顺序堆叠，后一个控件将会直接叠加在前一个控件之上。
2. 控件对象不会存在一个更新区域 的概念，这意味着即使被遮盖住的位置，依然会被绘制出来。

关于这一点，可能会有读者提出争议，但实际上根据 `RenderDoc` 抓取的 `GPU` 绘制结果，这一结论是正确的，如图12-1所示：

![](/res/img/post/07-ue-daxiangwuxing-04/12-3-1.png)

当然，如果简单地逐控件进行绘制，那么每个控件将会产生一个 `DrawCall` 渲染请求，这是一个非常低效的过程，这将导致复杂界面的渲染速度非常低下。

为了解决这个的问题，虚幻引擎采用了一个称为 **ElementBatch（对象批量渲染）** 的概念。为了能够引出这样的概念，我们不妨进行这样一番推理：

1. 有些控件对象实际上是可以同时渲染的，只要它们互相不重叠就可以。
2. 重叠的对象必须按照先后次序渲染。
3. 实际上来说，只有 `SOverlay`、`SCanvas` 这样的控件，才会产生控件对象堆叠，类似 `SVerticalBox` 这样的控件，是根本不会产生堆叠的。

因此，如图12-2所示，对每个需要堆叠对象的层级编号增加 1，就能够构成一个树状结构。其中如果是平级对象，如 `SVerticalBox`，其子对象的层级编号相等。如果是 `SOverlay`，其每一个子对象的层级编号，都是在其上一个子对象层级编号基础上加 1。

![](/res/img/post/07-ue-daxiangwuxing-04/12-3-2.png)

于是我们会发现，**处于同层级编号的几何对象，都是不产生堆叠和遮挡的。所以可以将这些对象同时进行渲染。** 对于上图中所展示的案例而言，这样的批量渲染方案，能够将渲染请求次数从 5 降低到 3。**对于真正的控件来说，渲染量将会降低得更为明显，因此这是一个相当取巧的方案。**

<br>
<br>

# 十三、蓝图

> 用了这么久的蓝图，它究竟是如何工作的呢？
> **C++函数是如何向蓝图暴露的呢？**
> **蓝图虚拟机是如何运行的呢？**

## 13.1 蓝图架构简述

虚幻引擎提供了蓝图系统作为可视化编程系统。蓝图系统从 `Kismet` 系统发展而来，同时整合了 `UnrealScript` 的诸多优势，从而成为了一个**完整的面向对象的可视化编程系统**。虽然对蓝图的争议从虚幻引擎4.0.1 版本开始一直持续到 4.13 版本，但是我们不可否认，**虚幻引擎的蓝图系统在高速迭代上具有极大的优势**。**小型蓝图的编译速度远远快过 C++ 的编译速度**。并且**最新版本的虚幻引擎能够在发布的时候将蓝图编译为 C++，从而提升蓝图在最终发布版本的执行效率，进而弥补蓝图在性能上的固有缺陷**。**因此分析，选择C++的理由可能只剩下 C++ 本身更大范围的 API，以及蓝图系统在内容较多时候不如 C++ 直观的原因。**

蓝图系统依然是一套依托于虚幻引擎现有 `UClass`、`UProperty`、`UFunction` 框架，根植于虚幻引擎 `UnrealScript` 字节码编译器系统的一套可视化编程系统。这就意味着：

1. 蓝图最终编译结果依然会转化为 `UClass`、`UProperty`、`UFunction` 信息。其中指令代码将会存储于 `UFunction` 的信息中。
2. **蓝图本身可以看作是一种可视化版的 `UnrealScript`，是经过转化和经过语法解析生成的字节码。**

<br>

## 13.2 前端：蓝图存储与编辑

需要注意的是，蓝图系统实际上是由三个部分组成的：**蓝图编辑系统**、**蓝图本身**、**蓝图编译后的字节码**。**最终编译完成的蓝图字节码将不会包含蓝图本身的节点信息。** 这部分信息是在 `UEdGraph` 中存储的，这也是一种优化。

`UEdGraph` 用于表示蓝图的数据结构。从整体来说，可以把其看作以下这个结构：

![](/res/img/post/07-ue-daxiangwuxing-04/13-2-1.png)

<br>

### 13.2.1 Schema

`Schema` 在英文中的意思为模式，笔者觉得理解为语法比较合适。其规定了当前蓝图能够产生什么样的节点等信息。具体而言，当定义了自己的 `Schema` 之后，通过重载对应的函数即可实现语法的规定，部分相关函数的含义列表如下：

1. `GetContextMenuActions` 定义在当前蓝图编辑器中右键菜单的菜单项。通过向 `FGraphContextMenuBuilder` 引用中填充内容，实现对右键菜单的定制。
2. `GetGraphContextActions` 与上面介绍的 `ContextMenuActions` 不同之处在于，定义的是右键菜单和拖曳连线之后弹出的菜单公用的菜单项。
3. `CanCreateConnection` 该函数传入两个 `UEdGraphPin`，判断是否能够建立一条连线。需要注意的是，返回值并非一个布尔值。其通过构造一个 `FPinConnectionResponse` 作为返回。构造函数中需要传入两个参数，第一个参数为具体的连接结果，第二个参数是一个消息返回，包括：
    > `CONNECT_RESPONSE_MAKE` 可以连接，直接建立起一条连线。
    > `CONNECT_RESPONSE_DISALLOW` 不准建立连线。
    > `CONNECT_RESPONSE_BREAK_OTHERS_A`（第一个参数 A）对应的连接头的所有其他连接线都会被截断。
    > `CONNECT_RESPONSE_BREAK_OTHERS_B`（第二个参数 B）对应的连接头的所有其他连接线都会被截断。
    > `CONNECT_RESPONSE_BREAK_OTHERS_AB` 两个连接头对应的其他连接线都会被截断。
    > `CONNECT_RESPONSE_MAKE_WITH_CONVERSION_NODE` 建立起一条连接线，但会立即产生一个转换节点。
4. `CanMergeNode` 字面含义为能够合并节点，具体作用不明。
5. `Try` 系列函数包括 `TryCreateConnection` 这样的函数，其主要是在进行具体的连线之前进行一次先行检测，此时返回 `false` 作为失败，不会有任何副作用，只是“尝试失败”。这个函数比 `CanCreateConnection` 简单，且无副作用。而 `CanCreateConnection` 的某些返回值是有副作用的。
6. `Debug` 监视相关函数，包括 `DoesSupportPinWatching`、`IsPinBeingWatched`、`ClearPinWatch`，主要用于设置节点值的监视。
7. `CreateDefaultNodesForGraph` 为当前 `UEdGraph` 创建一个默认节点。许多蓝图都有一个基础节点，例如状态机的入口节点。通过重载该函数可以实现。
8. `Drop` 系列函数在从外部拖放一个资源对象进入蓝图时触发。包括拖放到蓝图中、拖放到节点上、拖放到 `Pin` 上。
9. `CreateConnectionDrawingPolicy` 创建一个连接线绘制代理。可以通过自定义代理来修改连接线的样式

这里只介绍了部分相对来说常用的重载函数。如果希望查看具体有哪些语法可以重载，可以查看 `EdGraphSchema.h` 中的定义。根据笔者的经验，实现一套自己的状态机蓝图是完全没有问题的。

<br>

### 13.2.2 编辑器

蓝图的编辑器实质上是一个 `Slate` 控件，即 `SGraphEditor`。**可以这样理解，`UEdGraph` 存储蓝图的 “数据” 资料。`SGraphEditor` 通过解析数据，生成对应的 “显示控件”，并处理交互。** 类似于 `MVC` 架构中的 `View` 部分。

通过在 `SNew` 实例化 `SGraphEditor` 的时候，传入一个 `UEdGraph*` 类型的参数 `GraphToEdit`，即可指定当前编辑器正在编辑的蓝图。当然，也有更多的参数可以设置：

- `IsEditable` 是否可以编辑。
- `DisplayAsReadOnly` 是否以只读方式显示。
- `IsEmpty` 是否为空。
- `TitleBar` 编辑器顶部工具栏的控件。

以上只举出部分参数。具体可以参考 `GraphEditor.h` 中 `SGraphEditor` 的定义。

行为树编辑器是这样产生自己的 `GraphEditor` 的：

> 代码截取自 UE4.27.2

```cpp
// Create the graph editor
GraphEditorPtr = SNew(SGraphEditor)
	.AdditionalCommands(ReferenceViewerActions)
	.GraphToEdit(GraphObj)
	.GraphEvents(GraphEvents)
	.ShowGraphStateOverlay(false)
	.OnNavigateHistoryBack(FSimpleDelegate::CreateSP(this, &SReferenceViewer::GraphNavigateHistoryBack))
	.OnNavigateHistoryForward(FSimpleDelegate::CreateSP(this, &SReferenceViewer::GraphNavigateHistoryForward));
```

之前已经提到过，`UEdGraph` 并非一定需要编译：`UEdGraph` 只是一种数据包。如果你有需要，你可以定义自己的蓝图系统，用于编辑自己的素材资源。例如著名的地下城建筑师插件就是通过自定义蓝图完成对地下城生成逻辑的编辑。

<br>

## 13.3 后端：蓝图的编译

**只有包含具体逻辑的蓝图才需要编译为蓝图字节码，有一些蓝图仅仅只需要自身数据就可以了。**

在对这个流程进行分析之前，我们首先必须要明确，蓝图作为一种面向对象的语言，其三大数据的存储方式为：**元数据 metadata（UClass）**、**属性数据**、**方法数据**。也就是说，我们还得再次对 `UClass` 做进一步分析。

`UClass` 继承自 `UStruct`。**需要注意的是，`UStruct` 只包含成员变量的反射信息，其支持高速的构造和析构；而 `UClass` 则更加重量级，其需要对成员函数进行反射。** 在 `UClass` 中有成员变量 `FuncMap`，其用于存储当前 `UClass` 包含的成员函数信息。换句话说，尽管从感觉上而言，我们调用的是对象的某个成员函数，但是**实际上，同一个类的所有对象共有同样的成员函数，因此成员函数的信息存储于类数据中，而非存储在每个对象中。**

**也就是说，类的元数据信息、类的成员函数属性信息、成员变量属性信息均存储于类 `UClass` 的数据中，而非类对象的数据中。每个对象只包含自己可能变化的部分，也就是对象自己的成员变量数据信息。**

那么，蓝图编译完成产生的结果，应当是一个包含完整信息的 `UClass` 对象，而不是对应的类的对象。如果觉得这个很难理解，我们可以这样想：`UClass` 就像是一套考试的试卷，其包含自己独有的信息，例如自己的成员变量信息，如姓名、班级、学号，甚至答案本身也是一种成员变量；其也有自己的成员函数信息，就像一道一道的题目。而每个学生的答案实际上只是一个数据包，是这个试卷产生的实例，包含自己独立的成员变量。但是其对应的函数都是一致的（每一道题都一样）。我们的蓝图 `UBlueprint` 编译完毕后，产生的是试卷，这就是 `UBlueprint` 和 `UClass` 的关系。务必理清这个关系，才继续往下阅读！

根据虚幻引擎官方文档，以及笔者的个人理解。**一个蓝图会经历以下过程，最终产生出 `UClass`：**

> 下面的过程新手朋友暂时先过一遍了解即可，后面随着对源码的熟悉会慢慢融会贯通的。

- **类型清空**：清空当前类的内容。每个蓝图生成的类，即 `UBlueprintGeneratedClass`，都会被复用，并非被删除后创建新的实例。对应函数为 `CleanAndSanitizeClass`。
- **创建类成员变量**：根据蓝图中的 **“NewVariables”** 数组与其他位置定义的类成员变量，创建 `UProperties`。对应的函数为 `CreateClassVariablesFromBlueprint`。
- **创建成员函数列表**：**编译器执行这四个过程：处理当前的事件蓝图，处理当前的函数蓝图，对函数进行预编译，创建函数列表。** 处理事件蓝图 调用 `CreateAndProcessUberGraph` 函数，将所有的事件蓝图复制到一张超大蓝图里面，此时每个节点都有机会去展开自己（例如宏节点）。同时每个事件蓝图都会创建一个 `FKismetFunctionContext` 对象与之对应。
- **处理函数蓝图**：普通的函数蓝图通过 `ProcessOneFunctionGraph` 函数进行处理。此时每个函数蓝图会被复制到一个暂时的蓝图里面，同时节点有机会被展开。同样地，每个函数蓝图都会有一个 `FKismetFunctionContext` 与之对应。
- **预编译函数**：`PrecompileFunction` 函数对函数进行预编译。具体而言，其完成了这样的内容：
    - “修剪” 蓝图，只有连接的节点才会被保留，无用的节点会被删掉。
    - 运行现在还剩下的节点句柄的 `RegisterNets` 函数。
    - 填充函数的骨架：包括参数和局部变量的信息。但是里面还没有脚本代码。
- **组合和链接类**：此时编译器已经获得了当前类的所有属性与函数的信息，可以开始组合和链接类了。包括填充变量链表、填充变量的大小、填充函数表等。**这一步本质上产生了一个类的头文件以及一个类默认对象（CDO）。** 但是缺少类的最终标记以及元数据。
- **编译函数**：请注意这一步还没产生实际的虚拟机码！这一步包括：
    - 调用每个节点句柄的 `Compile` 函数，从而生成一个 `FKismetCompiledStatement` 对象。
    - 调用 `AppendStatementForNode` 函数。
- **完成类编译**：填充类的标记和元数据，并从父类中继承需要的标记和元数据。最终进行一系列的最终检测，确保类被正确编译。
- **后端产生最终代码**：后端逐函数地转换节点状态集合为最终代码。如果使用 `FKismetCompilerVMBackend`，则产生虚拟机字节码，使用 `FKismetCppBackend` 则产生 `C++` 代码。在笔者撰写时，`CppBackend` 作为实验性功能，已经可以投入使用。
- **复制类默认值对象属性**：借助一个特殊函数——`CopyPropertiesForUnrelatedObjects`，从而将老的类默认对象的值复制到新的对象中。**因为这个转换是通过基于 `Tag` 的序列化完成的，因此只要名字没变，值就会被转换过来。** 而组件则会被重新实例化，并被适当地修复。**在整个过程中，生成的新类的默认对象 `CDO` 是权威参照。**
- **重新实例化**：由于新的类的大小可能会改变，参数也有可能增减，编译器需要对原来的那个类的所有对象进行重新实例化。首先借助 `TObjectInterator` 来找到正在编译的类的所有实例，生成一个新的类，然后通过 `CopyPropertiesForUnrelatedObjects` 将老实例的值更新到新的实例。

在这个过程中涉及的术语如下：

- `FKismetCompilerContext`：完成实际编译工作的类。每次编译都会创建一个新的对象。这个类存储了正在被编译的蓝图和类的引用。
- `FKismetFunctionContext`：包含编译一个单独函数的信息，持有对应蓝图（不一定是函数蓝图）的引用、属性的引用以及生成的 `UFunction` 的引用。
- `FNodeHandlingFunctor`：单例类，也是辅助类。用于处理编译过程中的一类节点的类。包含一系列的函数，用于注册连接线以及生成编译后的 `Statement` 信息。
- `FKismetCompiledStatement`：一个编译器的独立工作单元。编译器把节点转换为一系列已经编译好的表达式，最终后端会将表达式转换为字节操作码。案例：变量赋值、Goto、Call。
- `FKismetTerm`：蓝图中的一个端子（literal、const 或者 vaiable 的引用）。每个数据链接点都对应一个这个东西！你可以在 `NodeHandlingFunctor` 中创建你自己的端子，用来捕获变量或者传递结果。

官方文档关于具体编译过程的描述相对来说比较笼统，因此笔者会用一个更加具体的方式来描述编译的某些过程。但是官网的文档依然是非常有价值的，笔者在分析这一段的时候，会大量引用文档中的过程和名词，如果你不记得某个名字的意义，请务必返回查询。

<br>

### 多编译器适配

对于每一个对当前蓝图进行编译的请求（通过调用 `FKismet2CompilerModule` 的 `CompileBlueprint` 函数），`FKismet2CompileModule` 会询问当前所有的 `Compiler`，调用它们的 `CanCompile` 函数，询问是否可以编译当前蓝图。

换句话说，如果某个类实现了 `IKismetCompilerInterface` 接口，那么就可以通过以下代码：

> 下面代码截取自 UE4.27.2

```cpp
// Register blueprint compiler
FKismetCompilerContext::RegisterCompilerForBP(UControlRigBlueprint::StaticClass(), &FControlRigDeveloperModule::GetControlRigCompiler);
IKismetCompilerInterface& KismetCompilerModule = FModuleManager::LoadModuleChecked<IKismetCompilerInterface>("KismetCompiler");
KismetCompilerModule.GetCompilers().Add(&ControlRigBlueprintCompiler);
```

来在 `Kismet` 编译器中注册编译器。虚幻引擎自己的系统 `UMG` 编辑器就是通过这样的方式注册自己为一个编译器的。

<br>

### 编译上下文

这里讨论的蓝图主要是我们平时使用的代码蓝图，也就是不讨论状态机、动画蓝图和 `UMG` 蓝图。如果发现 `KismetCompiler` 能够编译当前蓝图，虚幻引擎会创建一个 `Kismet` 编译器上下文，即 `FKismetCompilerContext` 结构。这个结构会负责编译当前的蓝图。官方文档描述的编译过程，其实可以在 `FKismetCompilerContext的Compile` 函数中找到对应。

存在多种类型的 `CompilerContext`，持有编译“一段”内容所需要的信息。接下来就会讲解主要上下文之间的关系。

<br>

### 整理与归并

下图展示了蓝图编译过程中，从蓝图 `UBlueprint` 结构到编译上下文 `FKismetCompileContext` 的主要过程。其中非箭头线表示持有关系，箭头表示步骤。这一步主要是将蓝图的数据结构进行整理、简化，转化为线性的调用链表，方便接下来逐节点地编译。

![](/res/img/post/07-ue-daxiangwuxing-04/13-3-1.png)

依前文所言，首先会对新变量进行处理，添加到成员变量数组中。这一段并不复杂，在此不做赘述，重点在接下来的过程。

- **事件蓝图展开**：首先就是对事件蓝图的处理。事件蓝图将会被展开为一个超大的蓝图，虚幻引擎源码称之为 `UberGraph`（这个 Uber 不是你手机上的那个 APP）。在这个过程中会展开收缩的节点，然后整理为一个一个的函数蓝图。其中一个事件实质上是被看作一个单独函数处理。因此单个超大蓝图持有多个子函数蓝图。对每一个子函数蓝图，分配一个函数编译上下文。
- **函数蓝图整理**：将每个蓝图的函数蓝图集合，拆分为一个一个的函数蓝图，然后转化为一个一个的函数编译上下文。
- **函数编译上下文列表收集**：完成了从事件蓝图、函数蓝图到函数编译上下文的转换之后，这些上下文会被收入函数编译上下文列表，即 `FunctionList` 中。**到这一步，已经完成了从 “具体蓝图” 到 “统一函数编译结构” 的转换。** 如果你再次查看所配的示意图，会发现编译上下文列表与事件蓝图和函数蓝图集合的位置是对应的，有一种对称的感觉。这就是笔者希望读者意识到的这样一种对应关系。
- **从函数到具体节点**：此时已经可以开始对函数进行编译。函数编译上下文通过对当前节点的整理，能够得到一个 “线性执行表”。需要注意的是，蓝图函数确实是线性执行的。即使是借助宏实现的 `Sequence` 序列执行，最终依然会被整理为线性执行表。这个表里面就对应着一个一个的函数节点。
- **节点编译**：此时线性执行表中依然存在以 `UEdGraphNode` 表示的节点。这些节点将会被编译系统处理而转化为合适的编译信息。为了能够应对更多的节点类型，而不需要频繁变更编译器代码，虚幻引擎使用 `NodeHandlers` 系统，在编译每个节点的时候，寻找对应的 `NodeHandler` 来完成编译。如果你创建了一个继承自 `UK2Node` 的类，你可以通过重载 `CreateNodeHandler` 以返回处理当前节点的节点句柄处理器。

<br>

### 节点处理

自此，编译过程开始进入逐节点的处理阶段。各路 `NodeHandler` 开始登台显神威。目前虚幻引擎比较主要的节点句柄包括：

- `FKCHandler_CallFunction` 处理函数调用节点。
- `FKCHandler_EventEntry` 处理事件入口节点。
- `FKCHandler_MathExpression` 处理数学表达式。
- `FKCHandler_Passthru` 处理返回值节点。
- `FKCHandler_VariableSet` 处理变量设置值节点。

说来难以相信，但是当你展开所有节点之后，蓝图的主要节点都可以被归并到这些节点中去。

接下来，对于每个节点，虚幻引擎都会通过 `AppendStatementForNode` 给其挂上 `FBlueprintCompiledStatement`，如下图所示。如前文所言，`FBlueprintCompiledStatement` 是一个独立的编译结构。在笔者前文中，这个结构被标记为需要重构，因此如果你看到的版本与笔者看到的不同，请见谅。毕竟虚幻引擎开发者们更新引擎代码的速度远远超过了笔者写分析的速度。

![](/res/img/post/07-ue-daxiangwuxing-04/13-3-2.png)

> 下面代码截取自 UE4.27.2

```cpp
//@TODO: Too rigid / icky design
struct FBlueprintCompiledStatement
{
	FBlueprintCompiledStatement()
		: Type(KCST_Nop)
		, FunctionContext(NULL)
		, FunctionToCall(NULL)
		, TargetLabel(NULL)
		, UbergraphCallIndex(-1)
		, LHS(NULL)
		, bIsJumpTarget(false)
		, bIsInterfaceContext(false)
		, bIsParentContext(false)
		, ExecContext(NULL)
	{
	}

	EKismetCompiledStatementType Type;

	// Object that the function should be called on, or NULL to indicate self (KCST_CallFunction)
	struct FBPTerminal* FunctionContext;

	// Function that executes the statement (KCST_CallFunction)
	UFunction* FunctionToCall;

	// Target label (KCST_Goto, or KCST_CallFunction that requires an ubergraph reference)
	FBlueprintCompiledStatement* TargetLabel;

	// The index of the argument to replace (only used when KCST_CallFunction has a non-NULL TargetLabel)
	int32 UbergraphCallIndex;

	// Destination of assignment statement or result from function call
	struct FBPTerminal* LHS;

	// Argument list of function call or source of assignment statement
	TArray<struct FBPTerminal*> RHS;

	// Is this node a jump target?
	bool bIsJumpTarget;

	// Is this node an interface context? (KCST_CallFunction)
	bool bIsInterfaceContext;

	// Is this function called on a parent class (super, etc)?  (KCST_CallFunction)
	bool bIsParentContext;

	// Exec pin about to execute (KCST_WireTraceSite)
	class UEdGraphPin* ExecContext;

	// Pure node output pin(s) linked to exec node input pins (KCST_InstrumentedPureNodeEntry)
	TArray<class UEdGraphPin*> PureOutputContextArray;

	// Comment text
	FString Comment;
};
```

而目前笔者看到的这个结构中，存储的信息包括：

- `EKismetCompiledStatementType` 类型枚举。这个值非常重要。会在接下来的描述中分析。实质上这个值决定了接下来字段的启用与含义。可能这也是为何需要重构的原因。为了方便描述，在下面的成员变量介绍中对类型枚举的名字省略前缀 “KCST”。
- `FunctionContext` 函数上下文。
- `FunctionToCall` 当前类型为 `CallFunction` 的时候启用，调用某个函数的时候该字段被赋值。
- `TargetLabel` 当类型为 `Goto` 或者 `CallFunction` 的时候启用，指向跳转的目标 `Statement`。
- `UbergraphCallIndex` 当类型为 `CallFunction`，且 `TargetLabel` 非空时启用，指向超大蓝图中的某个事件/函数的索引。
- `LHS` 根据名称推测应为“左值目标”之类的含义，类型为 `FBPTerminal*`，指向赋值语句的目标或者函数调用的结果。
- `RHS` 同上，推测应为“右值目标”之类的含义，类型为 `FBPTerminal*`，指向赋值语句的源数据，或者函数调用的参数列表。
- `bIsJumpTarget` 这是否是一个跳转目标？
- `bIsInterfaceContext` CallFunction 启用，是否是一个接口上下文？
- `bIsParentContext` CallFunction 启用，是否是在父类中调用？
- `ExecContext` WireTraceSite启用，即将被执行的Pin（具体意义笔者尚不清楚）。
- `Comment` 注释。

实际上来说，`FBlueprintCompiledStatement` 结构是一个 “头” + “信息” 的结构，如果你曾经学过汇编语言或者机器指令，那么你可以理解为：一个 `Statement` 就像是一个定长指令，包含开头的一个操作码与后面的操作参数。指令本身定长（即 `FBlueprintCompiledStatement` 结构大小固定），操作码不同，填充不同字段。

那么我们是时候来看 `EKismetCompiledStatementType` 这个枚举了。这个枚举指示了 32 种操作。

```cpp
//////////////////////////////////////////////////////////////////////////
// FBlueprintCompiledStatement

enum EKismetCompiledStatementType
{
	KCST_Nop = 0,
	// [wiring =] TargetObject->FunctionToCall(wiring)
	KCST_CallFunction = 1,
	// TargetObject->TargetProperty = [wiring]
	KCST_Assignment = 2,
	// One of the other types with a compilation error during statement generation
	KCST_CompileError = 3,
	// goto TargetLabel
	KCST_UnconditionalGoto = 4,
	// FlowStack.Push(TargetLabel)
	KCST_PushState = 5,
	// [if (!TargetObject->TargetProperty)] goto TargetLabel
	KCST_GotoIfNot = 6,
	// return TargetObject->TargetProperty
	KCST_Return = 7,
	// if (FlowStack.Num()) { NextState = FlowStack.Pop; } else { return; }
	KCST_EndOfThread = 8,
	// Comment
	KCST_Comment = 9,
	// NextState = LHS;
	KCST_ComputedGoto = 10,
	// [if (!TargetObject->TargetProperty)] { same as KCST_EndOfThread; }
	KCST_EndOfThreadIfNot = 11,
	// NOP with recorded address
	KCST_DebugSite = 12,
	// TargetInterface(TargetObject)
	KCST_CastObjToInterface = 13,
	// Cast<TargetClass>(TargetObject)
	KCST_DynamicCast = 14,
	// (TargetObject != None)
	KCST_ObjectToBool = 15,
	// TargetDelegate->Add(EventDelegate)
	KCST_AddMulticastDelegate = 16,
	// TargetDelegate->Clear()
	KCST_ClearMulticastDelegate = 17,
	// NOP with recorded address (never a step target)
	KCST_WireTraceSite = 18,
	// Creates simple delegate
	KCST_BindDelegate = 19,
	// TargetDelegate->Remove(EventDelegate)
	KCST_RemoveMulticastDelegate = 20,
	// TargetDelegate->Broadcast(...)
	KCST_CallDelegate = 21,
	// Creates and sets an array literal term
	KCST_CreateArray = 22,
	// TargetInterface(Interface)
	KCST_CrossInterfaceCast = 23,
	// Cast<TargetClass>(TargetObject)
	KCST_MetaCast = 24,
	KCST_AssignmentOnPersistentFrame = 25,
	// Cast<TargetClass>(TargetInterface)
	KCST_CastInterfaceToObj = 26,
	// goto ReturnLabel
	KCST_GotoReturn = 27,
	// [if (!TargetObject->TargetProperty)] goto TargetLabel
	KCST_GotoReturnIfNot = 28,
	KCST_SwitchValue = 29,
	
	//~ Kismet instrumentation extensions:

	// Instrumented event
	KCST_InstrumentedEvent,
	// Instrumented event stop
	KCST_InstrumentedEventStop,
	// Instrumented pure node entry
	KCST_InstrumentedPureNodeEntry,
	// Instrumented wiretrace entry
	KCST_InstrumentedWireEntry,
	// Instrumented wiretrace exit
	KCST_InstrumentedWireExit,
	// Instrumented state push
	KCST_InstrumentedStatePush,
	// Instrumented state restore
	KCST_InstrumentedStateRestore,
	// Instrumented state reset
	KCST_InstrumentedStateReset,
	// Instrumented state suspend
	KCST_InstrumentedStateSuspend,
	// Instrumented state pop
	KCST_InstrumentedStatePop,
	// Instrumented tunnel exit
	KCST_InstrumentedTunnelEndOfThread,

	KCST_ArrayGetByRef,
	KCST_CreateSet,
	KCST_CreateMap,
};
```

**逐节点地创建了 `Statement` 之后，就开始了最终的后端编译。** 如果是使用 `UnrealScript` 的脚本字节码虚拟机，此时会逐函数地编译。首先创建一个 `FScriptBuilderBase`，称为 `ScriptWriter`，然后调用 `ScriptWriter` 的 `GenerateCodeForStatement` 函数，对每个 `Statement` 生成字节码。构造 `ScriptWriter` 的时候会传入当前 `UFunction` 的 `Script` 变量的引用作为参数（该变量不在 `UFunction` 定义中，在 `UStruct` 定义中），在向 `ScriptWriter` 写入的时候，就自动地不断填充 `UFunction` 的 `Script` 脚本数组。

由于 `Statement` 作为中间环节，已经对语法进行了高度的整理以方便接下来的生成过程，因此这个 `GenerateCodeForStatement` 函数已经没有太多的语法分析之类的内容，蓝图字节码本身更类似于汇编式的执行方式，故而最终的字节码发射更类似于直接在 `ScriptWriter` 中填充字节码。具体字节码的定义太多，有兴趣的读者，可以自行打开。

> 下面针对引擎 UE4.27.2 版本

在 `Engine\Source\Runtime\CoreUObject\Public\UObject\Script.h` 里面有完整的字节码值和含义对应。笔者不再过多分析的原因还包括，虚拟机字节码系统逐渐在被 C++ 后端系统替代。

<br>

### 一点后话：VM 虚拟机调用

经过编译生成的 `Script`，最终是如何被实际调用呢？如果读者对这个感兴趣，笔者可以简单解释一番，如果读者非常希望了解虚拟机的原理，可以查看本章最后部分关于虚拟机的介绍。编译完成的 `Script` 会被放在 `UFunction` 对应的 `Script` 数组中。同时当前 `UFunction` 的 `FUNC_Native` 标记不会被设置。因此在 `UFunction` 的 `Bind` 阶段（也就是将 `UFunction` 与所属类对接的时候），将会设置自身的 `Func` 指针指向 `UObject` 类的 `ProcessInternal` 函数，而非当前 `UFunction` 所属的 `UClass` 类中的 `C++` 函数表 `NativeFunctionLookupTable` 中对应的函数，如下图所示。

![](/res/img/post/07-ue-daxiangwuxing-04/13-3-3.png)

在调用 `UFunction` 的 `Invoke` 函数时，`UObject::ProcessInternal` 函数会被 “当作” 一个成员函数调用，去执行当前 `UFunction` 包含的脚本字节码。如果你不大熟悉如何调用一个 `UFunction`，那么调用一个 `UFunction` 的写法如下：

```cpp
UFunction* HandleChangeFunction = OwnerObject->FindFunction(*SearchFunctionName);
if (HandleChangeFunction != nullptr)
{
    FFrame Frame = FFrame(OwnerObject, HandleChangeFunction, NULL);
    HandleChangeFunction ->Invoke(OwnerObject, Frame, NULL);
}
```

此时会分配一个 `Frame` 作为调用的数据结构。如果此时 `UFunction` 是一个脚本函数，而非一个 `Native` 函数，`Frame` 会通过 `Step` 函数，一步一步解释执行脚本字节码。

那么有些读者可能执着地希望了解：字节码到实际机器码是如何完成的呢？

其实非常简单。有一个 `GNatives` 数组，里面是和VM操作码对应的执行函数。对于每一个操作码，以操作码为数组下标，取出来就是一个对应的执行函数，直接填充参数然后执行即可获取执行结果。这给人的感觉更像是一种解释器，而非一个最终编译为汇编机器码的执行前编译虚拟机。

借助 `IMPLEMENT_VM_FUNCTION` 宏就可以在 `GNatives` 中注册一个函数，你可以在 `ScriptCore` 中搜索这个宏，能找到每个字节码对应的具体执行函数定义。

<br>

### 小结

> 初次接触可能会感觉有点晦涩难懂，收获不多。但是起码我们对此有了个了解，随着工作和实践对源码的不断了解，会慢慢理解更多。所以初次接触不要太过担心没有学到什么^ ^

到这里笔者真的是舒了一口气，总算是将虚幻引擎的逻辑蓝图相关的内容向读者呈现完毕了。总而言之，这是一个涉及颇多方面的系统。从非常适合编辑的 `UEdGraph` 结构开始，逐步归并整理，以产生 `UClass` 结构。然后对逻辑相关的部分进行处理之后，不断向适合顺序执行的字节码结构靠拢，最终被 `Backend` 发射成为最终的字节码。

**其实从实用主义的角度而言，知道如何向蓝图暴露函数、蓝图如何调用 `C++` 层的函数就已经能够使用蓝图完成大部分的开发工作了。进一步，如果有需要，希望用 `C++` 调用蓝图函数的知识也并不复杂。那么为何非要了解蓝图的编译和工作方式呢？那是因为笔者希望读者能拥有扩展虚幻引擎本身的能力，针对项目的需求，扩展蓝图自身的节点，甚至创造自己的蓝图系统以应对特殊的需求（比如剧情分支树等）。另外，笔者也希望读者拥有对虚幻引擎自身工作机制的好奇心，以及对虚幻引擎本身进行研究的勇气。虚幻引擎的源码就在你的眼前，去了解它吧！不要惧怕什么！**

<br>
<br>

## 13.4 蓝图虚拟机

如果读者曾经学习过 Java，那么对 “虚拟机”（VM）这样的概念应该并不陌生。虚幻引擎内置的，用于执行蓝图编译后的字节码的虚拟机继承自 虚幻引擎3 时代的 `UnrealScript` 虚拟机。

简单来说，**虚幻引擎提供了一套基于 字节码 和栈 的虚拟机。** 笔者试图简单地向读者解释一个与虚幻引擎设计接近的抽象虚拟机的工作原理，这是因为虚幻引擎本身的虚拟机有自身的优化、妥协与设计，这会让本来就复杂的系统变得更加复杂。

<br>

### 13.4.1 便笺纸与白领的故事

让我们先抛开复杂的虚幻引擎或者虚拟机的概念，来看一个笨拙的白领的故事。这个笨拙的白领是公司老板的孩子，没有任何工作经验，却被老板委以重任。老板想出了一个绝妙的方法来让他的孩子能够工作，如下图所示：

![](/res/img/post/07-ue-daxiangwuxing-04/13-4-1.png)

1. 把他的工作分解为极其简单的小步骤，分别写在一张一张的便笺纸上，然后不停地放在桌子左边的一大堆便笺纸的最上边。
2. 在右侧放着一张简单的表格，这个表格从上到下编好了号，每一个格子都对应着 “打钩”、“签字”、“把文件丢给隔壁桌” 这样的简单指令。
3. 每个便笺纸上的小步骤都不用写文字，全都是一个编号。这个编号与表格中的某一个工作对应。
4. 老板给这个白领配了一大堆的助手，每个助手有一个编号，和右侧的工作表对应，且每个助手只负责一个任务，他们每天就等待这个白领喊自己的编号，一旦被喊，就走过去执行工作。
5. 这个白领每次从左边的便笺纸堆最下面抽出一张便笺纸，然后查找右侧的表格，找到了，就喊对应的助手过来执行任务。

接下来老板遇到了一定的困扰，假如是简单的指令，例如 “打钩”，那么这个白领还能应付，但是如果需要提供 “内容”（数据），例如 “在纸上写字” 这样的指令，就需要同时提供书写的内容，这可怎么办？这难不住智慧的老板，他发现可以这样解决问题：

规定：当遇到 “在纸上写字” 指令的时候，接下来的第一张便笺纸就是要写的文字！

于是当这个白领遇到 “在纸上写字” 的指令时，他会直接再抽出一张便笺纸，把这张便笺纸的内容理解（解释） 为 “字符串” 这样的文本数据，然后递给那个可怜的助手，告诉他：给我写这上面的内容！——我们的白领还是一个文盲，他并不认识这张纸上的字到底是数字还是一段中文，如果我们的便笺纸堆顺序有误，他甚至会把这张纸上的字当作工作表中的某个索引去查询，毫不意外，他什么也找不到，于是就蠢蠢地愣在那里等待下班！或者，假如因为意外，下一张便笺纸并不是要写的字符串，而是其他的数字或者是乱七八糟的东西，忠实的助手还是会去抄写，只是写出的都是混乱的数据！

这可是一个重要的信息：便笺纸堆里面不仅有指令，还有数据！同样地，字节码中也装着数据！

但是这并没有解决问题，因为还有一个重复的指令：“在纸上写 N 遍字符串 XXX”，这可需要两个参数：一个N、一个XXX 字符串。怎么才能告诉助手这些信息呢？为了维护身为老板孩子的威严，除了喊编号，他可不能直接向助手说话——否则岂不是暴露了自己是文盲的秘密？这同样难不倒老板，老板立刻发布规定：

1. 在办公桌中央，还有一个用于交换便笺纸的临时便笺纸堆。
2. 每当遇到类似 “在纸上写N遍字符串XXX” 这样的指令时：
   a. 白领从便笺纸堆底下抽出一张便笺纸 A，放入临时便笺纸堆最上面。
   b. 白领再从便笺纸堆底下抽出一张便笺纸 B，放入临时便笺纸堆最上面。
   c. 这个时候临时便笺纸堆情况如下：
   > B
   > A
   > 其他便笺纸堆

   d. 喊对应编号的助手过来。
3. 忠实的助手就走了过来，按照老板的规定，做这些事情：
   a. 拿出临时便笺纸堆最上面的一张（便笺纸 B），解释为字符串 XXX。
   b. 拿出临时便笺纸堆最上面的一张（便笺纸 A），解释为书写的遍数 N。
   c. 开始抄写。
4. 助手不准怀疑这些便笺纸的正确性，也不准窥探其他便笺纸是什么，只准拿走规定数量的便笺纸，做自己该做的事。

**熟悉汇编语言的读者应该立刻就能明白，这是函数调用时通过压栈和出栈进行数据交换的方法。** 总而言之，笨拙的白领居然真的就这样干起了活儿，还做的有模有样，最苦的还是那个负责把任务拆分成一个一个小步骤的人（编译器）。

<br>

### 13.4.2 虚幻引擎的实现

上一个故事中的概念，实际上都映射一个真实虚拟机需要具备的工作，如下表所示：

![](/res/img/post/07-ue-daxiangwuxing-04/13-4-2.png)

前文已经提到，虚幻引擎的执行系统与这个故事描述的理想虚拟机存在一定的差异。对于真实的虚幻引擎来说，其执行系统被封装到 `FFrame` 这个类中。

> 以下代码截取自 UE4.27.2

```cpp
struct FFrame : public FOutputDevice
{	
public:
	// Variables.
	UFunction* Node;
	UObject* Object;
	uint8* Code;
	uint8* Locals;

	FProperty* MostRecentProperty;
	uint8* MostRecentPropertyAddress;

	/** The execution flow stack for compiled Kismet code */
	FlowStackType FlowStack;

	/** Previous frame on the stack */
	FFrame* PreviousFrame;

	/** contains information on any out parameters */
	FOutParmRec* OutParms;

	/** If a class is compiled in then this is set to the property chain for compiled-in functions. In that case, we follow the links to setup the args instead of executing by code. */
	FField* PropertyChainForCompiledIn;

	/** Currently executed native function */
	UFunction* CurrentNativeFunction;

	bool bArrayContextFailed;
public:

	// Constructors.
	FFrame( UObject* InObject, UFunction* InNode, void* InLocals, FFrame* InPreviousFrame = NULL, FField* InPropertyChainForCompiledIn = NULL );

	virtual ~FFrame()
	{
#if DO_BLUEPRINT_GUARD
		FBlueprintContextTracker& BlueprintExceptionTracker = FBlueprintContextTracker::Get();
		if (BlueprintExceptionTracker.ScriptStack.Num())
		{
			BlueprintExceptionTracker.ScriptStack.Pop(false);
		}
#endif
	}

	// Functions.
	COREUOBJECT_API void Step( UObject* Context, RESULT_DECL );

	/** Replacement for Step that uses an explicitly specified property to unpack arguments **/
	COREUOBJECT_API void StepExplicitProperty(void*const Result, FProperty* Property);

	/** Replacement for Step that checks the for byte code, and if none exists, then PropertyChainForCompiledIn is used. Also, makes an effort to verify that the params are in the correct order and the types are compatible. **/
	template<class TProperty>
	FORCEINLINE_DEBUGGABLE void StepCompiledIn(void* Result);
	FORCEINLINE_DEBUGGABLE void StepCompiledIn(void* Result, const FFieldClass* ExpectedPropertyType);

	/** Replacement for Step that checks the for byte code, and if none exists, then PropertyChainForCompiledIn is used. Also, makes an effort to verify that the params are in the correct order and the types are compatible. **/
	template<class TProperty, typename TNativeType>
	FORCEINLINE_DEBUGGABLE TNativeType& StepCompiledInRef(void*const TemporaryBuffer);

	COREUOBJECT_API virtual void Serialize( const TCHAR* V, ELogVerbosity::Type Verbosity, const class FName& Category ) override;
	
	COREUOBJECT_API static void KismetExecutionMessage(const TCHAR* Message, ELogVerbosity::Type Verbosity, FName WarningId = FName());

	/** Returns the current script op code */
	const uint8 PeekCode() const { return *Code; }

	/** Skips over the number of op codes specified by NumOps */
	void SkipCode(const int32 NumOps) { Code += NumOps; }

	template<typename TNumericType>
	TNumericType ReadInt();
	float ReadFloat();
	FName ReadName();
	UObject* ReadObject();
	int32 ReadWord();
	FProperty* ReadProperty();

	/** May return null */
	FProperty* ReadPropertyUnchecked();

	/**
	 * Reads a value from the bytestream, which represents the number of bytes to advance
	 * the code pointer for certain expressions.
	 *
	 * @param	ExpressionField		receives a pointer to the field representing the expression; used by various execs
	 *								to drive VM logic
	 */
	CodeSkipSizeType ReadCodeSkipCount();

	/**
	 * Reads a value from the bytestream which represents the number of bytes that should be zero'd out if a NULL context
	 * is encountered
	 *
	 * @param	ExpressionField		receives a pointer to the field representing the expression; used by various execs
	 *								to drive VM logic
	 */
	VariableSizeType ReadVariableSize(FProperty** ExpressionField);

	/**
 	 * This will return the StackTrace of the current callstack from the last native entry point
	 **/
	COREUOBJECT_API FString GetStackTrace() const;

	/**
	* This will return the StackTrace of the all script frames currently active
	* 
	* @param	bReturnEmpty if true, returns empty string when no script callstack found
	**/
	COREUOBJECT_API static FString GetScriptCallstack(bool bReturnEmpty = false);

	/** 
	 * This will return a string of the form "ScopeName.FunctionName" associated with this stack frame:
	 */
	COREUOBJECT_API FString GetStackDescription() const;

#if DO_BLUEPRINT_GUARD
	static void InitPrintScriptCallstack();
#endif
};
```

这个类成员如下：

**执行环境相关变量**

- **Node** 当前函数对应的蓝图节点。
- **Object** 所属的对象，相当于 C++ 执行中的 this 指针。
- **Code** 编译完成的字节码数组，类型为 `uint8*`，说明这是一个字节码数组。
- **Locals** 局部变量，每一个 `UFunction` 会在这个区域分配和自己的局部变量长度相等的内存，然后在此构造 `UProperty` 类型的对象，作为局部变量使用。

**辅助变量**

- **MostRecentProperty** 、**MostRecentPropertyAddress**最近使用的变量及其地址。
- **FlowStack** 执行流程栈。可以向内压入和弹出执行流程。这是为了完成顺序执行多条流程这样的要求，实质上是每次压入多个执行流程的起始地址，然后每执行完毕一个流程，弹出一个地址，再执行下一个流程。
- **PreviousFrame** 上一个帧。
- **OutParams** 输出参数。
- **CurrentNativeFunction** 当前正执行完毕的原生函数。

**成员函数**

**最重要的成员函数即：获取字节码并执行的 `Step` 函数，和一系列的从字节码中读取数据的 `Read` 函数。**

`Step` 每调用一次该函数，就会取出一个字节码，然后执行。对应代码如下：

```cpp
void FFrame::Step(UObject* Context, RESULT_DECL)
{
	int32 B = *Code++;
	(GNatives[B])(Context,*this,RESULT_PARAM);
}
```

可以看到，先通过 `Code` 指针获取当前字节码（顺便下移指针到下一个字节码），以此作为索引查找 `GNatives` 数组并获取对应的原生函数指针，然后调用。

`Read` 系列函数 包括 ReadInt、ReadFloat、ReadName、ReadObject、ReadWord、ReadProperty 等，以 ReadFloat 为例，删除对字节对齐进行判断的宏后，代码如下：

```cpp
inline float FFrame::ReadFloat()
{
	float Result = FPlatformMemory::ReadUnaligned<float>(Code);
	Code += sizeof(float);
	return Result;
}
```

可见这是直接将字节码数组的下一个 `float` 长度的数据读出，解释为 `float` 返回。印证了前文叙述的字节码不仅包含指令，还包含数据的说法。

由于虚幻引擎的蓝图系统存在 “局部变量” 的概念，因此暂存栈可以被更准确的局部变量绑定代替。

那么我们来看一个具体的实例：

<br>

#### 准备

这会儿有一个 `Actor` 对象 `X`，使用 C++ 通过 `Actor::ProcessEvent` 函数调用 `X` 中的一个蓝图函数 `Func`，开始进入到函数执行阶段。

<br>

#### 执行

一个 `Frame` 对象会被创建出来，作为当前的执行帧，并控制字节码的执行。构造函数传入的比较重要的参数有以下两个：

- **UObject* InObject** 所属的对象，包含执行所需要的成员变量数据 。
- **UFunction* InNode** 执行的函数，包含执行所需要的成员函数指令数据 。

通过以下代码，`Code` 成员变量将会被赋予函数所属的字节码数组的首地址指针：

```cpp
Code(InNode->Script.GetData())
```

在本次执行的内容中，字节码数组总共有 85 个字节码元素。我们取出一小节来进行分析。

如何知道这些字节码对应什么函数呢？我们可以在运行时中断执行，然后查看 `GNatives` 数组，这个数组指明了每一个字节码对应的函数，需要注意的是，不能简单认为每一个字节码都对应函数，前文有描述，字节码是包含数据的。经过翻译，我们获得了这样的结果，如下表所示：

![](/res/img/post/07-ue-daxiangwuxing-04/13-4-3.png)

我们重点关注 15 这个字节码。`execLet` 是一个赋值指令，就像 `a=1+3` 这样的表达式，那么其需要执行以下内容：

- **获得赋值目标**：`execLet` 调用 `Stack.Step` 函数来获取赋值目标，存入 `MostRecentPropertyAddress`。我们这次遇到的是 `execLocalVariable` 指令，表示将一个局部变量的值读取出来，具体来说，就是将当前 `Frame` 的 `Locals` 数组首地址转化为一个指针，复制给 `MostRecentPropertyAddress` 这个成员变量。
- **表达式求值并赋予**：已经完成赋值目标的获得，接下来需要完成 “值” 的计算，故 `execLet` 再次调用 `Stack.Step` 函数，传入当前对象与赋值目标以完成真正的赋值。本次赋值函数对应了 `execMakeVector` 函数，也就是先组成一个 `FVector`，再赋值给目标。

换句话说，`Let` 指令执行的依然是类似 `a=1+3` 的过程，先获得等号左边的地址，再将地址交给右边的表达式求值函数，用于让函数将计算完毕后的结果填充到对应地址。

<br>
<br>

### 13.4.3 C++函数注册到蓝图

当使用 `UFUNCTION` 宏定义了一个可以被虚拟机调用的函数时，`UHT` 会自动帮助你生成一个看上去像这样的函数：

```cpp
DECLARE_FUNCTION(exec[你的函数名])
{
	P_GET_TARRAY_REF(USceneComponent*, Z_Param_Out_AllComponet); //获取参数
	P_FINISH;
	P_NATIVE_BEGIN;
	this->GetAllSubCustomComponet(Z_Param_Out_AllComponet); //实际调用区域
	P_NATIVE_END;
}
```

`DECLARE_FUNCTION` 这个宏生成了一个 “exec[你的函数名]” 为名称的函数，参数分别为 `FFrame&` 与 `RESULT_DECL`，请读者务必记住这里的定义，下文会用到。对应的宏定义为：

> 下面代码截取自 UE4.27.2

```cpp
// This macro is used to declare a thunk function in autogenerated boilerplate code
#define DECLARE_FUNCTION(func) static void func( UObject* Context, FFrame& Stack, RESULT_DECL )
```

**剩下的宏主要作用就是从虚拟机中准备调用你的函数需要的参数环境等。** 随后在 `.generated.cpp` 中，这个函数会借助 `FNativeFunctionRegistrar::RegisterFunction` 函数，被注册到当前 `UClass` 类的 `NativeFunction` 列表中，并增加 `UFunction` 信息到 `UClass`。

假如一个蓝图需要调用一个 C++ 函数，编译器会设置字节码 `EX_FinalFunction`，并将对应的 `UFunction` 指针同时放入到下一个字节码中，执行时就会调用：

```cpp
// Call the final function.
P_THIS->CallFunction( Stack, RESULT_PARAM, (UFunction*)Stack.ReadObject() );
```

从而将对应的 `exec` 函数所属的 `UFunction` 对象取出，`CallFunction` 函数在检查到这是一个原生函数时，会直接调用 `UFunction::Invoke` 函数。在 `Invoke` 函数内部会调用当前 `UFunction` 函数的 `Func` 函数指针：

```cpp
return (*Func)(Obj, Stack, RESULT_PARAM);
```

注意参数！这个函数就是上文中，`UHT` 通过 `DECLARE_FUNCTION` 宏帮我们生成的 `exec` 函数！

简而言之，C++ 函数封装过程可以这样理解：

1. C++函数被 `UHT` 识别，生成包裹函数 `exec<函数名>`。
2. `exec<函数名>` 被 `FNativeFunctionRegistrar` 在运行前注册到 `UClass` 的函数列表中，创建 `UFunction` 信息。
3. 蓝图编译器在发现函数调用节点时，生成调用 `State`，发现是原生函数，生成 `EX_FinalFunction` 字节码，并将对应的 `UFunction` 指针压栈。
4. 蓝图执行系统发现 `EX_FinalFunction` 字节码，于是读取下一段字节，解释为 `UFunction*`。
5. 通过 `UFunction*` 的 `Invoke` 函数，调用 `exec<函数名>` 包裹函数。
6. 包裹函数拆包并准备参数列表，调用 C++ 函数。

<br>
<br>

## 13.5 蓝图系统小结

蓝图一直是虚幻引擎富有争议的部分。无数的程序员诟病这个系统，认为极为不工程化、面向大型工程没有效率。同时 `Epic` 又极力推崇这个系统，致力于让蓝图能够完成更多的需求。也有许多人支持蓝图这个系统，认为蓝图在快速成形、高速迭代上具有自己的优势。

而实际而言，蓝图系统与虚幻引擎在运行时编译 `C++（HotReloadC++）`系统，完成了传统的 “脚本语言” 需要完成的任务。运行时编译 C++，允许游戏的底层逻辑能够更高效地被迭代验证，而蓝图系统则承担了需要不断变化的关卡、特性逻辑部分。

最重要的一点是，蓝图系统极大降低了编译器校验的难度，其直接将类整理为了成员变量、成员函数、表达式这样的语义对象，从而降低了对语义进行上下文分析的难度。这从某种意义上说，也是对稳定性的提升。

笔者曾经使用 `UnrealScript` 系统，那时遇到过一个极难发现的 BUG：当在 `UnrealScript` 的 `State` 状态块中，声明一个与类成员变量一模一样的变量时，编译器不会检查出错误。甚至连字节码虚拟机都没有检查出错误。直到对这个变量进行赋值时，整个虚拟机就崩溃了，没有给出任何有价值的报错信息。

蓝图通过自己的可视化语法来规避了这种问题。作为一个自行实现的脚本语言，`Epic` 应当是有这方面的考虑。在笔者撰写本书时，蓝图系统已经能够通过编译为 C++ 代码来降低运行时的性能消耗，因此蓝图最大的问题依然是工程上的，而非效率上的。本文只阐述蓝图的工作原理，而不对 “蓝图是否好用” 的问题作出一个定论性的回答。

> 蓝图字节码存储前文有叙述，蓝图字节码在编译时，如果遇到对象指针，会直接存储指针值本身。这就涉及一个很严重的问题：序列化结束后，对象在内存中的位置会发生改变，原有的存储于字节码中的指针值会失效。虚幻引擎采用的方案是在存储之前再次遍历字节码数组，依次取出字节码过滤，遇到对象指针值时，替换为 `ImportTable` 或者 `ExportTable` 中的索引值。在反序列化时同样会重新遍历，并修正指针的值。

<br>
<br>

The End.
