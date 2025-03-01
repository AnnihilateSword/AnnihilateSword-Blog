---
title: 【UE大象无形总结】第一部分虚幻引擎C++编程【开篇】
tags: [UE4]
date: 2024-12-20
updated: 2024-12-20
cover: /res/img/post/04-ue-daxiangwuxing-01/cover.jpg
top_img: /res/img/site/background.png
---

![](/res/img/post/04-ue-daxiangwuxing-01/cover.jpg)

<br>

# 前言

> **慢工出细活。**
> 
> 参考书籍：《大象无形：虚幻引擎程序设计浅析 (罗丁力 [罗丁力])》
> 补充说明：本系列是个人对书中内容的实践

<br>
<br>

# 一、开发之前——五个最常见的基类

## 1.1 UObject 类

> 什么时候该继承自 UObject 类？什么时候应该声明一个纯 C++ 类？

一个类继承自 UObject 类，应该是它需要 UObject 提供的功能。什么样的功能让你选择继承自 UObject 类？

<br>

### 1.1.1 垃圾回收（GC）

- 继承自 UObject 类，同时指向 UObject 类实例对象的指针成员变量，使用 UPROPERTY 宏进行标记。虚幻引擎的 UObject 架构会自动地将被 UPROPERTY 标记的变量考虑到垃圾回收（GC）系统中，自动地进行对象的生命周期管理。

<br>

### 1.1.2 反射（Reflection）

- 虚幻引擎实现了这样一套机制，可以在运行时获取一个类的信息。

<br>

### 1.1.3 序列化（Serialization）

- 可以将一个类的对象保存到磁盘，同时在下次运行时完好无损地加载。

> 个人补充：在做网络同步的时候利用序列化也能对网络传输的数据有一定的优化

<br>

### 1.1.4 网络复制（Network Replication）

- 继承自 UObject 的类，其被宏标记的变量能够自动地完成网络复制的功能。从服务器端复制对应的变量到客户端。

<br>

### 1.1.5 与虚幻引擎编辑器的交互

- 允许你的类的变量被虚幻引擎编辑器的 Editor 简单地编辑

<br>

### 1.1.6 运行时类型识别

- 虚幻引擎打开了 /GR-编译器参数。意味着你无法使用 C++ 标准的 RTTI 机制：dynamic_cast。如果你希望使用，请继承自 UObject 类，然后使用 Cast<> 函数来完成。
    > 这是因为虚幻引擎实现了一套自己的、更高效的运行时类型识别的方案。

**综上所述，当你需要这些功能的时候，你的这个类应该继承自 UObject 类。**

> 请注意：UObject类会在引擎加载阶段，创建一个 Default Object 默认对象（CDO）。这意味着：
> 1. 构造函数并不是在游戏运行的时候调用，同时即便你只有一个 UObject 对象存在于场景中，构造函数依然会被调用两次。
> 2. 构造函数被调用的时候，UWorld 不一定存在。GetWorld() 返回值有可能为空！

<br>
<br>

## 1.2 AActor 类

> 什么时候该继承自 AActor 类？

AActor 类拥有这样的能力：**它能够被挂在组件。**

组件并不是 AActor。如果你观察，会发现所有组件的类的开头是 U 而不是 A。

坐标与旋转量，只是一个 Scene Component 组件所拥有的特征。如果这个 AActor 不需要一个固定位置（例如你的某个 Manager），你甚至可以不给 AActor 挂载 Scene Component 组件。

- 你希望让 AActor 被渲染？给一个静态网格组件。
- 你希望 AActor 有骨骼动画？给一个骨架网格物体组件。
- 你希望你的 AActor 能够移动？通常来说你可以直接在你的 AActor 类中书写代码来实现。当然，你也可以附加一个 Movement 组件以专门处理移动。

**所以，需要挂载组件的时候，你才应该继承自 AActor 类。** 也就是说，我刚刚描述的，你的 Manager，也许只需要一个纯 C++ 类就够了（当然，你需要序列化之类的功能，那就是另一回事了）

<br>
<br>

## 1.3 灵魂与肉体：Pawn、Character 和 Controller

### 1.3.1 Pawn

如果你研究了 Pawn 类的源码，你会发现 Pawn 提供了被 “操作” 的特性。它能够被一个 Controller 操纵。**这个 Controller 可以是玩家，当然也可以是 AI（人工智能）**。这就像是一个棋手，操作着这个棋子。

> Pawn 类，一个被操纵的兵或卒，一个一旦脱离棋手就无法自主行动的、悲哀的肉体。

<br>

### 1.3.2 Character

Character 类代表一个角色，它继承自 Pawn 类。

> 什么时候该继承自 Character 类，什么时候该继承自 Pawn 类呢？

Character 类提供了一个特殊的组件：`Character Movement`。这个组件提供了一个基础的、基于胶囊体的角色移动功能。包括移动和跳跃，以及如果你需要，还能扩展出更多，例如蹲伏和爬行。

如果你的 Pawn 类十分简单，或者不需要这样的移动逻辑（比如外星人飞船），那么你可以不继承自这个类。请不要有负罪感：

1. 不是虚幻引擎中的每一个类，你都得继承一遍。
2. 在 UE3 中，没有Character类，只有 Pawn 类。

> 当然，现在很多游戏中的角色（无论是人类，还是某些两足行走的怪物），都能够适用于Character类的逻辑。

<br>

### 1.3.3 Controller

**Controller 操纵着 Pawn 和 Character 的行为。可以通过 Possess / UnPossess 来控制或离开**

`Controller` 可以是 AI，`AIController` 类，你可以在这个类中使用虚幻引擎优秀的行为树 / EQS 环境查询系统。同样也可以是玩家，`Player Controller` 类。你可以在这个类中绑定输入，然后转化为对 Pawn 的指令。

<br>
<br>

# 二、需求到实现

> 我有一个庞大的游戏创意，但这是我第一次制作游戏，我该设计哪些类？

假设我们拿到的是这样的需求：

“玩家手中会持有一把武器，按下鼠标左键时，武器会射出子弹”。

从这句话中，我们能够找到这样的几个重要名词：玩家、武器和子弹。

我们意识到，这几个名词都可以作为类。也许有些类虚幻引擎已经提供给我们了，如玩家 APlayerController 类。那么，我们意识到，我们需要给武器和子弹各创建一个类。现在问题是，武器类该继承自什么？让我们回顾前面的内容。

首先，武器类有坐标吗？有的。这该是一个 AActor 的子类。

武器类是一种兵吗？不是，武器类不该是 Pawn 的子类。

恭喜你，你已经确定了武器类在整个游戏的类树中的位置。同样，你也能够确定子弹类在类树中的位置。它应该继承自  AActor 类，同时带有一个 Projectile Movement 组件。进一步你能够分析出，类与类之间的持有、通信关系：

1. 玩家类对象持有武器类对象
2. 武器类对象产生子弹对象
3. 玩家的输入会调用武器类对象的函数，以发射子弹

<br>
<br>

# 三、创建自己的 C++ 类

> 我想好我有哪些类了，现在我该怎么创建它们的代码呢？

## 3.1 使用 Unreal Editor 创建 C++ 类

都看这个了，我猜应该都知道吧。

<br>
<br>

## 3.2 手动创建 C++ 类

如果你出于某种原因，希望自己手动创建C++类。你需要完成以下的步骤：

在工程目录的 Source 文件夹下，找到和你游戏名称一致的文件夹。根据不同人创建的工程结构不同，你可能会发现下面两种文件结构：

- `public` 文件夹，`private` 文件夹，`.build.cs` 文件
- 一堆 `.cpp` 和 `.h` 文件，`.build.cs` 文件

第一种文件结构是标准的虚幻引擎模块文件结构。

1. 创建你的 `.h` 和 `.cpp` 文件，如果你是第一种文件结构，`.h` 文件放在 `public` 文件夹内，`.cpp` 文件放置在 `private` 文件夹内；
2. 在 `.h`  中声明你的类：如果你的类继承自 `UObject`，你的类名上方需要加入 `UCLASS()` 宏。同时，你需要在类体的第一行添加 `GENERATED_UCLASS_BODY()` 宏，或者 `GENERATED_BODY()` 宏。前者需要手动实现一个带有 `const FObject Initializer&` 参数的构造函数。后者需要手动实现一个无参数构造函数。注意说的是 “实现” 而非声明；
3. 在你的 `.h` 文件中，包含当前模块所需要的头文件，`xxx.generated.h` 文件需要在最底部，下面给出创建一个继承 AActor 类的示例：

```cpp
#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "TestActor.generated.h"

UCLASS()
class STARTGAME_API ATestActor : public AActor
{
    GENERATED_BODY()
}
```

4. 编译；

<br>
<br>

## 3.3 虚幻引擎的命名规则

常用的命名前缀如下：

- **F**：纯 C++ 类（也不一定）
- **U**：继承自 UObject，但不继承自 AActor
- **A**：继承自 AActor
- **S**：Slate 控件相关类
- **H**：HitResult 相关类

> 虚幻引擎头文件工具  Unreal Header Tool (UHT) 会在编译前检查你的类命名。如果类的命名出现错误，那么它会提出警告并终止编译。

<br>
<br>

# 四、对象

> 我也声明好了需要的类，那么：
> 我该如何实例化对象？
> 我该如何在世界中产生我声明的 AActor 类？
> 我该如何调用这些对象身上的函数？

## 4.1 类对象的产生

在标准 C++ 中，一个类产生一个对象，被称为 “实例化”。实例化对象的方法是通过 new 关键字。

而在虚幻引擎中，这一个问题变得略微复杂。对于某些类型，我们不得不通过调用某些函数来产生出对象。具体而言：

1. 如果你的类是一个纯 C++ 类型，你可以通过new来产生对象；
2. 如果你的类继承自 UObject 但不继承自 AActor，你需要通过 NewObject 函数来产生出对象；
3. 如果你的类继承自 AActor，你需要通过 SpawnActor 函数来产生出对象；

> 下面代码截取自 UE5.4.4

```cpp
/**
 * Convenience template for constructing a gameplay object
 *
 * @param	Outer		the outer for the new object.  If not specified, object will be created in the transient package.
 * @param	Class		the class of object to construct
 * @param	Name		the name for the new object.  If not specified, the object will be given a transient name via MakeUniqueObjectName
 * @param	Flags		the object flags to apply to the new object
 * @param	Template	the object to use for initializing the new object.  If not specified, the class's default object will be used
 * @param	bCopyTransientsFromClassDefaults	if true, copy transient from the class defaults instead of the pass in archetype ptr (often these are the same)
 * @param	InInstanceGraph						contains the mappings of instanced objects and components to their templates
 * @param	ExternalPackage						Assign an external Package to the created object if non-null
 *
 * @return	a pointer of type T to a new object of the specified class
 */
template< class T >
T* NewObject(
    UObject* Outer,
    const UClass* Class,
    FName Name = NAME_None,
    EObjectFlags Flags = RF_NoFlags,
    UObject* Template = nullptr,
    bool bCopyTransientsFromClassDefaults = false,
    FObjectInstancingGraph* InInstanceGraph = nullptr,
    UPackage* ExternalPackage = nullptr
);
```

这会返回一个指向你的类的指针，此时这个对象被分配在临时包中。下一次加载会被清除。

如果你的类继承自 AActor，你需要通过 UWorld 对象（可以通过 GetWorld() 获得）的  SpawnActor 函数来产生出对象。函数定义如下（有多个，这里只列出一个）：

> 下面代码截取自 UE5.4.4

```cpp
/**
 * Spawn Actors with given absolute transform (override root component transform) and SpawnParameters
 * 
 * @param	Class					Class to Spawn
 * @param	AbsoluteTransform		World Transform to spawn on - without considering CDO's relative transform, thus Absolute
 * @param	SpawnParameters			Spawn Parameters
 *
 * @return	Actor that just spawned
 */
AActor* SpawnActorAbsolute( UClass* Class, FTransform const& AbsoluteTransform, const FActorSpawnParameters& SpawnParameters = FActorSpawnParameters());
```

> 极为特殊的，如果你需要产生出一个 Slate 类——如果你有这样的需求，要么你已经在进行很深的开发，要么就是你的教程的版本过老，依然在使用 Slate 来开发游戏的界面控件，你需要使用 SNew 函数。我无法给出 SNew 函数的原型。关于 Slate 的详细讨论，请阅读后文中 Slate 的章节。

<br>
<br>

## 4.2 类对象的获取

获取一个类对象的唯一方法，就是通过某种方式传递到这个对象的指针或引用。

但是有一个特殊的情况，也是大家经常询问到的：如何获取一个场景中，某种 AActor 的所有实例？答案是，借助 AActor  迭代器：TActorIterator。

> 下面代码截取自 UE5.4.4

```cpp
/**
 * Constructor
 *
 * @param InWorld	The world whose actors are to be iterated over.
 * @param InClass	The subclass of actors to be iterated over.
 * @param InFlags	Iteration flags indicating which types of levels and actors should be iterated
 */
explicit TActorIterator(const UWorld* InWorld, TSubclassOf<ActorType> InClass = ActorType::StaticClass(), EActorIteratorFlags InFlags = EActorIteratorFlags::OnlyActiveLevels | EActorIteratorFlags::SkipPendingKill)
	: Super(InWorld, InClass, InFlags)
{
	++(*this);
}
```

示例代码如下：

```cpp
for (TActorIterator<AActor> It(GetWorld()); It; ++It)
{
    // do something...
}
```

> 其中 TActorIterator 的泛型参数不一定是 AActor，可以是你需要查找的其他类型。你可以通过 `*It` 来获取指向实际对象的指针。或者你可以直接 通过 `It->YourFunction(...)` 来调用你需要的成员函数 。

> 个人补充：例如联网游戏中，我们会在 ds 上获取指定 UID 对应的 PlayerController

<br>
<br>

## 4.3 类对象的销毁

### 4.3.1 纯 C++ 类

如果你的纯 C++ 类是在函数体中创建，而且不是通过 new 来分配内存，例如：

```cpp
void YourFunction()
{
    FYourClass YourObject = FYourClass();
    // do something...
}
```

此时这个类的对象会在函数调用结束后，随着函数栈空间的释放，一起释放掉。不需要你手动干涉。

如果你的纯 C++ 类是使用 new 来分配内存，而且你直接传递类的指针。那么你需要意识到：除非你手动删除，否则这一块内存将永远不会被释放 。如果你忘记了，这将产生内存泄漏。

如果你的纯 `C++` 类使用 `new` 来分配内存，同时你使用智能指针 `TSharedPtr  / TSharedRef` 来进行管理，那么你的类对象将不需要也不应该被你手动释放。智能指针会使用引用计数来完成自动的内存释放。你可以使用 `MakeShareable` 函数来转化普通指针为智能指针：

```cpp
TSharedPtr<YourClass> YourClassPtr = MakeShareable(new YourClass(...));
```

> 强烈建议，在你没有充分的把握之前，不要使用手动 `new / delete` 方案。你可以使用智能指针。

<br>

### 4.3.2 UObject 类

UObject 类的情况略有不同。事实上你无法使用智能指针来管理 UObject 对象 。

UObject 采用自动垃圾回收机制（GC）。当一个类的成员变量包含指向 UObject 的对象，同时又带有 `UPROPERTY` 宏定义，那么这个成员变量将会触发引用计数机制。

垃圾回收器会定期从根节点 Root 开始检查，当一个 UObject 没有被别的任何 UObject 引用，就会被垃圾回收。你可以通过 AddToRoot 函数来让一个 UObject 一直不被回收。

> 下面代码截取自 UE5.4.4 `UObjectBaseUtility.h`

```cpp
/**
 * Add an object to the root set. This prevents the object and all
 * its descendants from being deleted during garbage collection.
 */
FORCEINLINE void AddToRoot()
{
	GUObjectArray.IndexToObject(InternalIndex)->SetRootSet();
}
```

<br>

### 4.3.3 AActor 类

AActor 类对象可以通过调用 Destory 函数来请求销毁，这样的销毁意味着将当前 AActor 从所属的世界中 “摧毁”。但是对象对应内存的回收依然是由引擎系统决定。

<br>
<br>

# 五、从 C++ 到蓝图

## 5.1 UPROPERTY 宏

> 虚幻引擎的蓝图真是太好用了，我该如何让蓝图能够调用我的 C++ 类中的函数呢？

当你需要将一个 `UObject` 类的子类的成员变量注册到蓝图中时，你只需要借助 `UPROPERTY` 宏即可完成。

```cpp
UPROPERTY(...)
```

你可以传递更多参数来控制 `UPROPERTY` 宏的行为，通常而言，如果你要注册一个变量到蓝图中，你可以这样书写：

```cpp
UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category = "Object")
```

<br>

## 5.2 UFUNCTION 宏

你也可以通过 `UFUNCTION` 宏来注册函数到蓝图中。下面是一个注册的案例：

```cpp
UFUNCTION(BlueprintCallable, Category = "Test")
```

其中 `BlueprintCallable` 是一个很重要的参数，表示这个函数可以被蓝图调用。可选的还有：`BlueprintImplementableEvent`、`BlueprintNativeEvent`。前者表示，这个成员函数由其蓝图的子类实现，你不应该尝试在 C++ 中给出函数的实现，这会导致链接错误。后者表示，这个成员函数提供一个 “C++ 的默认实现”，同时也可以被蓝图重载。你需要提供一个  “函数名 _Implement” 为名字的函数实现，放置于.cpp中。

<br>
<br>

# 六、游戏性框架概述

## 6.1 行为树：概念与原理

### 6.1.1 为什么选择行为树？

在 UE3 的时代，AI 框架选择的是状态机来实现。甚至为了支持状态机编程，虚幻引擎的脚本语言 Unreal Script 专门增加了几个状态相关的关键字，足以看出虚幻引擎官方对状态机系统的重视。但是这个系统在 UE4 中被行为树系统代替，状态机只在动画蓝图中保留（当然这并不妨碍你在任何地方书写状态机，毕竟用 C++ 实现一个状态机模式并不复杂）。

那么，是什么促使 Epic 做出这样的决定呢？简而言之，同样的 AI 模式，用状态机会涉及大量的跳转，但是用行为树就相对来说更加简化。同时由于行为树的 “退行” 特点，也就是 “逐个尝试，不行就换” 的思路，更加接近人类的思维方式，因此当你熟悉了行为树的框架之后，能够更加快速地撰写 AI 相关的代码。

<br>

### 6.1.2 行为树原理

“行为树” 是一种通用的 AI 框架或者说模式，其并不依附于特定的引擎存在，并且虚幻引擎的行为树也与标准的行为树模式存在一定的差异。

![](/res/img/post/04-ue-daxiangwuxing-01/6-1.png)

我们会发现，这是一个由节点、连接线构成的行为树。行为树包含三种类型的节点：

- **流程控制**：包含 `Selector` (选择器) 和 `Sequence` (顺序执行器)（关于平行执行 `parallel` 节点，暂时不做分析）
- **装饰器**：对子树的返回结果进行处理的节点
- **执行节点**：执行节点必然是叶子节点，执行具体的任务，并在任务执行一段时间后，根据任务执行成功与否，返回 true 或者 false

![](/res/img/post/04-ue-daxiangwuxing-01/6-2.png)

除去根节点 Root，`Selector` 就是一个流程控制节点。`Selector` 节点会从左到右逐个执行下面的子树，如果有一个子树返回true，它就会返回 true，只有所有的子树均返回 false，它才会返回 false。这就类似于日常生活中 “几个方案都试一试”  的概念。

> 反映到如图 6-2 所示的行为树案例，对应左侧的图片。这个行为树实际上讲述了一段艰难的故事：在荒年，如果吃不起饭的时候，就只能选择吃糠了。

假如流程节点被换为了 `Sequence`，那么 `Sequence` 节点就会按顺序执行自己的子树，只有当前子树返回 true，才会去执行下一个子树，直到全部执行完毕，才会向上一级返回 true。任何一个子树返回了 false，它就会停止执行，返回 false。类似于日常生活中 “依次执行” 的概念。把一个已有的任务分为几个步骤，然后逐个去执行，任何一个步骤无法完成，都意味着了任务失败。

标准的装饰器节点，是对子树返回的结果进行处理，再向上一级进行返回的。例如 `Force Success` 节点，就是强制让子树返回 true，不管子树真正返回的是什么。如上文例子，假如在 Root 上加了一个 `Force Success` 装饰器节点，那就像是古代有些鱼肉百姓的官员，一方面对下面报上来的饥荒情况心知肚明，另一方面又不停向上级返回 “成功”。

> `Selector` 与普通的随机选择还是有区别的，我们能够通过顺序来定义“从一般到特殊”的AI行为。这是行为树很强大的一个地方。

![](/res/img/post/04-ue-daxiangwuxing-01/6-3.png)

这可能是一个很倒霉的 AI，不过它做出了一套让我们都能够认为很自然的行为。而不是因为地铁封闭，就傻傻地站在地铁口等待。这就是行为树系统的威力。

<br>
<br>

## 6.2 虚幻引擎网络架构

### 6.2.1 同步

从第一个联机游戏开始，同步就成为了一个重点的研究对象。随着需要同步的人数不断变多，联机同步架构的设计也在不断地变动。

最早的联机同步按照 `点对点网络` 的思路进行设计。也就意味着，假如开了一个房间，有4个玩家加入，那么玩家1输入每一个指令与消息，都会发往其他3个人，从而让4个人的画面得以一致。大致类似于，如果你按了一下W键，那么这个前进的指令就会发送到所有你连接的人的电脑上。然后你所控制的那个人物都会向前移动一点点。

在某些早期的联机游戏中，为了保证所有人的同步，甚至采用过更极端一些的方法，如强制所有人更新频率一致。

**点对点同步带来的弊端相当得多，毕竟这是一个网状结构。例如**：

1. 由于点对点同步带来的传输消耗，因此网络传输压力会很大。
2.  由于点对点同步不存在 “权威” 性，因此当其中一个人作弊时，会影响所有的客户端。且很难判定 “作弊”。

因此，经典的 `服务器-客户端架构模型`产生了。

一台（在当年）具有较高性能的主机被选出来，作为 `中心服务器`。所有的 “游戏性相关指令” 都会被发往中心服务器进行处理，随后中心服务器会把世界的状态 `同步` 到各个客户端。

于是之前的网状结构变为了 `星型结构` ，且出现了 `权威服务器` 的概念。**也就意味着，作弊变得更加困难**。无法通过直接传递坐标（只能传递游戏性相关指令）给服务端，就算是本地客户端，坐标也来自于服务端同步过来的信息。

也就是说，我们能把游戏框架切分为两个部分：一部分是 “指令”，是对游戏世界造成影响的代码请求，比如 “人物前移3米”、“人物挥刀碰到了怪物”；另一部分是 “状态”，是游戏世界的各种数值状态，比如 “当前人物生命值”、“当前怪物生命值”。**客户端只能向服务端发送 “指令”，服务端根据指令处理后，改变游戏世界的状态，并将状态 同步 给每一个客户端。**

这是相当优秀的一个设计，对同步考虑了非常多。因此，被作为了现在绝大多数网络同步模型的基本思路。不过这是不够的，因为有另一个非常基本的问题，那就是延迟。而为了解决延迟问题，UE3 提出了 `广义的客户端-服务端模型` 的概念。

<br>

### 6.2.2 广义的 客户端——服务端  模型

> 对于这个模型，其实虚幻引擎官方在 `Unreal Development Kit` 的文档 `UDN` 中，有一句非常精彩的表述：
> 
> **客户端是对服务端的拙劣模仿**

👉[UDN-Main-WebHome](https://docs.unrealengine.com/udk/Main/WebHome.html)

这句话的意思是说，客户端自己也同样运行着一个世界，并不断 `预测` 服务端的行为。从而不断更新当前世界，以最大程度地 `接近` 服务端的世界。譬如说，在 UE3 的时代，服务端会不断同步当前对象的位置和速度到客户端，由于广泛存在着网络延时，因此当这个状态信息到达客户端时，实际上这个状态已经过时了。那么客户端怎么办呢？

让我们把思路扭转一下，也就是说，客户端不再试图去 “同步” 服务端，而是去 “模仿” 服务端。这就是说，我们承认 “延迟” 客观存在，只要我们的客户端模仿得别太差劲，那么玩家是可以接受这样的效果的。客户端可以根据同步数据发送时的当前对象的位置与速度，加上数据发送的时间，猜测出当前对象在服务端的可能位置。并且通过修正当前世界（比如调整当前对象的速度方向，指向新的位置），去模仿服务端位置。如果服务端的位置和客户端差距太大，就强行闪现修正。

而拙劣二字，就是在强调，**服务端的世界是绝对正确的**，而客户端则是不断试图猜测服务端当前时间的状态。

> 打个比方，如图 6-4 所示：假设一个飞行员在追踪一个 UFO（不明飞行物），飞行员看不到那个 UFO 的位置，只能从基地给他的报告中获得 UFO 的位置与速度向量。如何才能尽可能靠近 UFO 呢？因为飞行员如果直接向基地给出的报告位置飞行，那么由于飞到那个位置需要一定时间，等飞行员飞到，UFO 已经不在那儿了。飞行员可以选择根据自己飞行的时间、UFO当前位置、UFO的速度，猜测出一个位置，那个位置是当 UFO 保持当前速度方向、大小不变的情况下，自己的飞机最终一定会在那里和 UFO 汇合 ，然后向那个位置飞行。然后不断根据基地的信息修正，于是就能尽可能保证靠拢UFO的行动位置。

![](/res/img/post/04-ue-daxiangwuxing-01/6-4.png)

再次反思我们刚才讨论的例子，就会发现，客户端的体验相对来说是非常流畅的。只要网络延迟不太大，那么客户端的对象就不会发生瞬移。这也就解释了有些采用同样模型的游戏中存在的 “Ping神” 现象。就是说当某个人延迟在阈值上下波动时，一会儿客户端会去用速度调整的方式修正位置（在地上滑来滑去），一会儿客户端又会直接把这个人的位置强行同步（滑了一会儿一下子又传送回原地）。

<br>
<br>

# 七、引擎系统相关类

> 官方文档介绍的内容不多，我不知道我要的功能引擎有没有提供，怎么办？

本章希望向读者介绍一些在开发过程中积累的虚幻引擎自带的功能。有些功能隐藏于代码中，没有出现在官方文档，因此尽可能介绍给读者，避免读者重复 “造轮子”。

## 7.1 在 UE4 中使用正则表达式

正则表达式，又称正规表示法、常规表示法。

正则表达式是对字符串操作的一种逻辑公式，就是用事先定义好的一些特定字符，以及这些特定字符的组合，组成一个 “规则字符串”，这个 “规则字符串” 用来表达对字符串的一种过滤逻辑。

在 UE4 / UE5 使用正则表达式，首先，我们要添加头文件。

```cpp
#include "Internationalization/Regex.h"
```

需要注意的是，此头文件是放在 Core 模块里的。一般我们不需要额外在 Build.cs 里添加了。

示例代码：

```cpp
void ATestActor::BeginPlay()
{
	Super::BeginPlay();
	
	FString TextStr("ABCDEFGHIJKLMN");
	FRegexPattern Parttern(TEXT("C.+H"));
	FRegexMatcher Matcher(Parttern, TextStr);
	if (Matcher.FindNext())
	{
		UE_LOG(LogTemp, Warning, TEXT("Matched: %s"), *Matcher.GetCaptureGroup(0));
		UE_LOG(LogTemp, Warning, TEXT("Matched: %d - %d"), Matcher.GetMatchBeginning(), Matcher.GetMatchEnding());
	}
}
```

![](/res/img/post/04-ue-daxiangwuxing-01/7-1.png)

<br>

## 7.2 FPaths 类的使用

在 Core 模块中，UE4 提供了一个用于路径相关处理的类—— FPaths。在 FPaths 中，主要添加3类 “工具” 性质的API。

1. 具体路径类，如：FPaths::ProjectContentDir() 可以获取到游戏 Content 目录
2. 工具类 ，如：FPaths::FileExists() 用于判断一个文件是否存在。
3. 路径转换类 ，如：FPaths::ConvertRelativePathToFull()用于将相对路径转换为绝对路径。

由于 FPaths 类中的函数众多，这里不一一列举。

<br>

## 7.3 XML 与 JSON

### 7.3.1 XML

在 `Xml Parser` 模块中，提供了两个类解析 XML 数据，分别是 FastXML 与 FXmlFile。相对而言，用 FXmlFile 更方便一些。所以本例使用 FXmlFile。

首先，创建一个 XML 文件。示例内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<note name="Ami" age="100">
    <to>George</to>
    <from>John</from>
    <heading>Reminder</heading>
    <body>Don't forget the meeting!</body>
    <list>
        <line>Hello</line>
        <line>World</line>
        <line>PPPPPP</line>
    </list>
</note>
```

这里补充一下 UE4 中 FString 重载的一些辅助函数：

```cpp
/**
 * Concatenate this path with given path ensuring the / character is used between them
 *
 * @param Lhs Path to concatenate onto.
 * @param Rhs Path to concatenate.
 * @return new FString of the path
 */
UE_NODISCARD FORCEINLINE friend FString operator/(FString&& Lhs, const TCHAR* Rhs)
{
	checkSlow(Rhs);

	int32 StrLength = FCString::Strlen(Rhs);

	FString Result(MoveTemp(Lhs), StrLength + 1);
	Result.PathAppend(Rhs, StrLength);
	return Result;
}
```

解析 XML 代码如下（需要引入相应头文件）：

```cpp
#include "XmlFile.h"
```

![](/res/img/post/04-ue-daxiangwuxing-01/7-3-1-1.png)

```cpp
void ATestActor::BeginPlay()
{
	Super::BeginPlay();
	
	FString XmlFilePath = FPaths::ProjectContentDir() / TEXT("XML/Test.xml");
	
	FXmlFile* Xml = new FXmlFile();
	Xml->LoadFile(XmlFilePath);
	FXmlNode* RootNode = Xml->GetRootNode();
	FString FromContent = RootNode->FindChildNode("from")->GetContent();
	UE_LOG(LogTemp, Warning, TEXT("from=%s"), *FromContent);
	FString NoteName = RootNode->GetAttribute("name");
	UE_LOG(LogTemp, Warning, TEXT("note @name=%s"), *NoteName);

	TArray<FXmlNode*> ListNode = RootNode->FindChildNode("list")->GetChildrenNodes();
	for (FXmlNode* Node : ListNode)
	{
		UE_LOG(LogTemp, Warning, TEXT("list: %s"), *(Node->GetContent()));
	}
}
```

![](/res/img/post/04-ue-daxiangwuxing-01/7-3-1-2.png)

<br>

### 7.3.2 JOSN

这里书上写的不全，下面是个人总结:

#### 7.3.2.1 添加 Json 模块

![](/res/img/post/04-ue-daxiangwuxing-01/7-3-1.png)

<br>

#### 7.3.2.2 加载并读取 Json 文件

1. 包含头文件

```cpp
#include "Serialization/JsonReader.h"
#include "Dom/JsonObject.h"
#include "Serialization/JsonSerializer.h"
#include "Misc/FileHelper.h"
```

2. 创建函数 `LoadStringFromFile`，用来读取文件中的字符串

```cpp
bool ARPGCharacter::LoadStringFromFile(const FString& FileName, const FString& RelativePath, FString& ResultString)
{
    if (!FileName.IsEmpty())
    {
        // FPaths::ProjectContentDir() 是 Content 目录下
        FString AbsolutePath = FPaths::ProjectContentDir() + RelativePath + FileName;
        if (FPaths::FileExists(AbsolutePath))
        {
            if (FFileHelper::LoadFileToString(ResultString, *AbsolutePath))
            {
                return true;
            }
            else
            {
                UE_LOG(LogTemp, Warning, TEXT("Load Error!"));
            }
        }
        else
        {
            UE_LOG(LogTemp, Warning, TEXT("File Not Exist!"));
        }
    }
    return false;
}
```

3. 读取 Json 文件示例

```cpp
// Called when the game starts or when spawned
void ARPGCharacter::BeginPlay()
{
    Super::BeginPlay();

    FString jsonStr;
    // 加载 Json 文件
    if (LoadStringFromFile("Player.json", "../LoadJsonFileTestConfig/", jsonStr))
    {
        // 创建 Json 阅读器
        TSharedRef<TJsonReader<>> jsonReader = TJsonReaderFactory<TCHAR>::Create(jsonStr);
        // 创建 Json 对象
        TSharedPtr<FJsonObject> jsonObject;
        // 反序列化，将 JsonReader 里的数据，传入 JsonObject 中
        FJsonSerializer::Deserialize(jsonReader, jsonObject);

        // 读取单行数据
        FString timeStr = jsonObject->GetStringField("Time");
        UE_LOG(LogTemp, Warning, TEXT("Time: %s"), *timeStr);

        // 读取数组数据
        TArray<TSharedPtr<FJsonValue>> data = jsonObject->GetArrayField("Data");
        for (int i = 0; i < data.Num(); i++)
        {
            FString dataStr1 = data[i]->AsObject()->GetStringField("key1");
            FString dataStr2 = data[i]->AsObject()->GetStringField("key2");
            UE_LOG(LogTemp, Warning, TEXT("Data:"));
            UE_LOG(LogTemp, Warning, TEXT("key1: %s"), *dataStr1);
            UE_LOG(LogTemp, Warning, TEXT("key2: %s"), *dataStr2);
        }
    }
}
```

4. 示例 Json 文件路径

![](/res/img/post/04-ue-daxiangwuxing-01/7-3-2.png)

![](/res/img/post/04-ue-daxiangwuxing-01/7-3-3.png)

`Player.json` 内容如下

```json
{
    "Time":"2024-02-21",
    "Data": [
        {
            "key1": "123",
            "key2": "345"
        },
        {
            "key1": "666",
            "key2": "520"
        }
    ]
}
```

5. 打印效果

![](/res/img/post/04-ue-daxiangwuxing-01/7-3-4.png)

<br>

#### 7.3.2.3 创建并写入 Json 文件

1. 包含头文件

```cpp
#include "Policies/CondensedJsonPrintPolicy.h"
```

2. 新建函数 `SaveStringToFile`，用来保存字符串到文件中

```cpp
bool ARPGCharacter::SaveStringToFile(const FString& FileName, const FString& RelativePath, FString& TargetString)
{
    if (!FileName.IsEmpty())
    {
        // FPaths::ProjectContentDir() 是 Content 目录下
        FString AbsolutePath = FPaths::ProjectContentDir() + RelativePath + FileName;

        if (FFileHelper::SaveStringToFile(TargetString, *AbsolutePath))
        {
            return true;
        }
        else
        {
            UE_LOG(LogTemp, Warning, TEXT("Save Error!"));
            return false;
        }
    }
    return false;
}
```

3. 写入 Json 文件示例

```cpp
// Called when the game starts or when spawned
void ARPGCharacter::BeginPlay()
{
    Super::BeginPlay();

    FString jsonStr;
    // 创建 Json 写入器
    TSharedRef<TJsonWriter<TCHAR, TCondensedJsonPrintPolicy<TCHAR>>> jsonWritter = TJsonWriterFactory<TCHAR, TCondensedJsonPrintPolicy<TCHAR>>::Create(&jsonStr);

    // 开始写入
    jsonWritter->WriteObjectStart();

    // 写入单行数据
    jsonWritter->WriteValue(TEXT("Time"), TEXT("2024-02-21"));
    // 写入数组数据
    jsonWritter->WriteArrayStart("Data");
    for (int i = 0; i < 2; i++)
    {
        jsonWritter->WriteObjectStart();
        jsonWritter->WriteValue(TEXT("key1"), TEXT("100"));
        jsonWritter->WriteValue(TEXT("key2"), TEXT("200"));
        jsonWritter->WriteObjectEnd();
    }
    jsonWritter->WriteArrayEnd();

    // 停止写入
    jsonWritter->WriteObjectEnd();

    // 关闭写入器
    jsonWritter->Close();
    UE_LOG(LogTemp, Warning, TEXT("%s"), *jsonStr);

    SaveStringToFile("Write.json", "../LoadJsonFileTestConfig/", jsonStr);
}
```

4. 写入效果

![](/res/img/post/04-ue-daxiangwuxing-01/7-3-5.png)

![](/res/img/post/04-ue-daxiangwuxing-01/7-3-6.png)

![](/res/img/post/04-ue-daxiangwuxing-01/7-3-7.png)

<br>

## 7.4 文件读写与访问

虚幻引擎提供了与平台无关的文件读写与访问接口，即 `FPlatformFileManager`。考虑到许多读者有读写自己文件的需求，故花一定篇幅重点阐述文件读写。

该类定义于头文件 `PlatformFilemanager.h` 中，所属模块为 Core，一般虚幻引擎会默认包含本模块，如果没有，请手动在当前模块的 `.build.cs` 中包含。

通过以下调用：

```cpp
FPlatformFileManager::Get()->GetPlatformFile();
```

能够获得一个 `IPlatformFile` 类型的引用。这个接口即提供了通用的文件访问接口，读者可以自行查询。

虚幻引擎之所以这么麻烦，是为了提供文件的跨平台性。如果纯粹提供静态函数或者全局函数，会增加代码的复杂程度。而且 `FPlatformFileManager` 作为全局管理单例，能够更有效地管理文件读写操作。

可以查看以下文件获取函数调用的具体参数信息：

`Engine/Source/Runtime/Core/Public/GenericPlatform/GenericPlatformFile.h`

虚幻引擎提供的比较常用的函数如下：

- 拷贝函数，提供文件目录和文件的拷贝操作

1. `CopyDirectoryTree` 递归拷贝某个目录

```cpp
/** 
 * Copy a file or a hierarchy of files (directory).
 * @param DestinationDirectory			Target path (either absolute or relative) to copy to - always a directory! (e.g. "/home/dest/").
 * @param Source						Source file (or directory) to copy (e.g. "/home/source/stuff").
 * @param bOverwriteAllExisting			Whether to overwrite everything that exists at target
 * @return								true if operation completed successfully.
 */
virtual bool CopyDirectoryTree(const TCHAR* DestinationDirectory, const TCHAR* Source, bool bOverwriteAllExisting);
```

1. `CopyFile` 拷贝当前文件

```cpp
/** 
 * Copy a file. This will fail if the destination file already exists.
 * @param To				File to copy to.
 * @param From				File to copy from.
 * @param ReadFlags			Source file read options.
 * @param WriteFlags		Destination file write options.
 * @return			true if the file was copied sucessfully.
**/
virtual bool CopyFile(const TCHAR* To, const TCHAR* From, EPlatformFileRead ReadFlags = EPlatformFileRead::None, EPlatformFileWrite WriteFlags = EPlatformFileWrite::None);
```

-  创建函数，提供创建文件和目录的操作，目录创建成功或者目录已经存在都会返回 true

1. `CreateDirectory` 创建目录

```cpp
/** Create a directory and return true if the directory was created or already existed. **/
virtual bool		CreateDirectory(const TCHAR* Directory) = 0;
```

2. `CreateDirectoryTree` 创建一个目录树，即给定一个路径字符串，如果对应路径的父目录不存在，也会被创建出来。

```cpp
/** Create a directory, including any parent directories and return true if the directory was created or already existed. **/
virtual bool CreateDirectoryTree(const TCHAR* Directory);
```

- 删除函数，删除指定目录或文件，成功删除返回 true，否则 false

1. `DeleteDirectory` 删除指定目录

```cpp
/** Delete a directory and return true if the directory was deleted or otherwise does not exist. **/
virtual bool		DeleteDirectory(const TCHAR* Directory) = 0;
```

2. `DeleteDirectoryRecursively` 递归删除指定目录

```cpp
/** 
 * Delete all files and subdirectories in a directory, then delete the directory itself
 * @param Directory		The directory to delete.
 * @return				true if the directory was deleted or did not exist.
**/
virtual bool DeleteDirectoryRecursively(const TCHAR* Directory);
```

3. `DeleteFile` 删除 指定文件

```cpp
/** Delete a file and return true if the file exists. Will not delete read only files. **/
virtual bool		DeleteFile(const TCHAR* Filename) = 0;
```

- 移动函数，只有一个函数 `MoveFile`，在移动文件时使用

```cpp
/** Attempt to move a file. Return true if successful. Will not overwrite existing files. **/
virtual bool		MoveFile(const TCHAR* To, const TCHAR* From) = 0;
```

- 属性函数，提供对文件、目录的属性访问操作

1. `DirectoryExists` 检查目录是否存在

```cpp
/** Return true if the directory exists. **/
virtual bool		DirectoryExists(const TCHAR* Directory) = 0;
```

2. `FileExists` 检查文件是否存在

```cpp
/** Return true if the file exists. **/
virtual bool		FileExists(const TCHAR* Filename) = 0;
```

3. `GetStatData` 获得文件状态信息，返回 `FFileStatData` 类型对象，这个对象其实包含了足够的状态信息

```cpp
/** Return the stat data for the given file or directory. Check the FFileStatData::bIsValid member before using the returned data */
virtual FFileStatData GetStatData(const TCHAR* FilenameOrDirectory) = 0;
```

```cpp
/**
 * Contains the information that's returned from stat'ing a file or directory 
 */
struct FFileStatData
{
	FFileStatData()
		: CreationTime(FDateTime::MinValue())
		, AccessTime(FDateTime::MinValue())
		, ModificationTime(FDateTime::MinValue())
		, FileSize(-1)
		, bIsDirectory(false)
		, bIsReadOnly(false)
		, bIsValid(false)
	{
	}

	FFileStatData(FDateTime InCreationTime, FDateTime InAccessTime,	FDateTime InModificationTime, const int64 InFileSize, const bool InIsDirectory, const bool InIsReadOnly)
		: CreationTime(InCreationTime)
		, AccessTime(InAccessTime)
		, ModificationTime(InModificationTime)
		, FileSize(InFileSize)
		, bIsDirectory(InIsDirectory)
		, bIsReadOnly(InIsReadOnly)
		, bIsValid(true)
	{
	}

	/** The time that the file or directory was originally created, or FDateTime::MinValue if the creation time is unknown */
	FDateTime CreationTime;

	/** The time that the file or directory was last accessed, or FDateTime::MinValue if the access time is unknown */
	FDateTime AccessTime;

	/** The time the the file or directory was last modified, or FDateTime::MinValue if the modification time is unknown */
	FDateTime ModificationTime;

	/** Size of the file (in bytes), or -1 if the file size is unknown */
	int64 FileSize;

	/** True if this data is for a directory, false if it's for a file */
	bool bIsDirectory : 1;

	/** True if this file is read-only */
	bool bIsReadOnly : 1;

	/** True if file or directory was found, false otherwise. Note that this value being true does not ensure that the other members are filled in with meaningful data, as not all file systems have access to all of this data */
	bool bIsValid : 1;
};
```

- 遍历函数，该类函数都需要传入一个 `FDirectoryVisitor` 或 `FDirectoryStatVisitor` 对象作为参数。你可以创造一个类继承自该类，然后重写 `Visit` 函数。每当遍历到一个文件或者目录时，遍历函数会调用 `Visitor` 对象的 `Visit` 函数以通知执行自定义的逻辑：

1. `IterateDirectory` 遍历某个目录

```cpp
/** 
 * Call the Visit function of the visitor once for each file or directory in a single directory. This function does not explore subdirectories.
 * @param Directory		The directory to iterate the contents of.
 * @param Visitor		Visitor to call for each element of the directory
 * @return				false if the directory did not exist or if the visitor returned false.
**/
virtual bool		IterateDirectory(const TCHAR* Directory, FDirectoryVisitor& Visitor) = 0;
```

2. `IterateDirectoryRecursively` 递归遍历某个目录

```cpp
/** 
 * Call the Visit function of the visitor once for each file or directory in a directory tree. This function explores subdirectories.
 * @param Directory		The directory to iterate the contents of, recursively.
 * @param Visitor		Visitor to call for each element of the directory and each element of all subdirectories.
 * @return				false if the directory did not exist or if the visitor returned false.
**/
virtual bool IterateDirectoryRecursively(const TCHAR* Directory, FDirectoryVisitor& Visitor);
```

3. `IterateDirectoryStat` 遍历文件目录状态，`Visit` 函数参数为状态对象而非路径字符串；

```cpp
/**
 * Call the visitor once for each file or directory in a single directory. This function does not explore subdirectories.
 * @param Directory		The directory to iterate the contents of.
 * @param Visitor		Visitor to call for each element of the directory (see FDirectoryStatVisitor::Visit for the signature)
 * @return				false if the directory did not exist or if the visitor returned false.
**/
virtual bool IterateDirectoryStat(const TCHAR* Directory, FDirectoryStatVisitorFunc Visitor);
```

4. `IterateDirectoryStatRecursively` 同上，递归遍历；

```cpp
/** 
 * Call the Visit function of the visitor once for each file or directory in a directory tree. This function explores subdirectories.
 * @param Directory		The directory to iterate the contents of, recursively.
 * @param Visitor		Visitor to call for each element of the directory and each element of all subdirectories.
 * @return				false if the directory did not exist or if the visitor returned false.
**/
virtual bool IterateDirectoryStatRecursively(const TCHAR* Directory, FDirectoryStatVisitor& Visitor);
```

- 读写函数，最底层的读写函数往往返回 `IFileHandle` 类型的句柄，这个句柄提供了直接读写二进制的功能。如果非绝对必要，可以考虑更高层的 API,下文有讲述：

1. `OpenRead` 打开一个文件用于读取，返回 `IFileHandle` 类型的句柄用于读取

```cpp
/** Attempt to open a file for reading.
 *
 * @param Filename file to be opened
 * @param bAllowWrite (applies to certain platforms only) whether this file is allowed to be written to by other processes. This flag is needed to open files that are currently being written to as well.
 *
 * @return If successful will return a non-nullptr pointer. Close the file by delete'ing the handle.
 */
virtual IFileHandle*	OpenRead(const TCHAR* Filename, bool bAllowWrite = false) = 0;
```

2. `OpenWrite` 打开一个文件用于写入，返回 `IFileHandle` 类型的句柄

```cpp
/** Attempt to open a file for writing. If successful will return a non-nullptr pointer. Close the file by delete'ing the handle. **/
virtual IFileHandle*	OpenWrite(const TCHAR* Filename, bool bAppend = false, bool bAllowRead = false) = 0;
```

同时，针对一些极其普遍的需求，虚幻引擎提供了一套更简单的方式用于读写文件内容，即 `FFileHelper` 类，位于 `Engine/Source/Runtime/Core/Public/Misc/FileHelper.h` 头文件中，提供以下静态函数：

`LoadFileToArray` 直接将路径指定的文件读取到一个 `TArray<uint8>` 类型的二进制数组中；

```cpp
/**
 * Load a binary file to a dynamic array with two uninitialized bytes at end as padding.
 *
 * @param Result    Receives the contents of the file
 * @param Filename  The file to read
 * @param Flags     Flags to pass to IFileManager::CreateFileReader
*/
static bool LoadFileToArray( TArray64<uint8>& Result, const TCHAR* Filename, uint32 Flags = 0 );
```

`LoadFileToString` 直接将路径指定的文本文件读取到一个 `FString` 类型的字符串中。请注意，字符串有长度限制，不要试图读取超大文本文件；


```cpp
/**
 * Load a text file to an FString. Supports all combination of ANSI/Unicode files and platforms.
 *
 * @param Result       String representation of the loaded file
 * @param Filename     Name of the file to load
 * @param VerifyFlags  Flags controlling the hash verification behavior ( see EHashOptions )
 */
static bool LoadFileToString( FString& Result, const TCHAR* Filename, EHashOptions VerifyFlags = EHashOptions::None, uint32 ReadFlags = 0 );
```

`SaveArrayToFile` 保存一个二进制数组到文件中；

```cpp
/**
 * Save a binary array to a file.
 */
static bool SaveArrayToFile(TArrayView<const uint8> Array, const TCHAR* Filename, IFileManager* FileManager=&IFileManager::Get(), uint32 WriteFlags = 0);
```

`SaveStringToFile` 保存一个字符串到指定文件中；

```cpp
/**
 * Write the FString to a file.
 * Supports all combination of ANSI/Unicode files and platforms.
 */
static bool SaveStringToFile( FStringView String, const TCHAR* Filename, EEncodingOptions EncodingOptions = EEncodingOptions::AutoDetect, IFileManager* FileManager = &IFileManager::Get(), uint32 WriteFlags = 0 );
```

`CreateBitmap` 在硬盘中创建一个 BMP 文件；

```cpp
/**
 * Saves a 24/32Bit BMP file to disk
 * 
 * @param Pattern filename with path, must not be 0, if with "bmp" extension (e.g. "out.bmp") the filename stays like this, if without (e.g. "out") automatic index numbers are addended (e.g. "out00002.bmp")
 * @param DataWidth - Width of the bitmap supplied in Data >0
 * @param DataHeight - Height of the bitmap supplied in Data >0
 * @param Data must not be 0
 * @param SubRectangle optional, specifies a sub-rectangle of the source image to save out. If NULL, the whole bitmap is saved
 * @param FileManager must not be 0
 * @param OutFilename optional, if specified filename will be output
 * @param bInWriteAlpha optional, specifies whether to write out the alpha channel. Will force BMP V4 format.
 * @param ChannelMask optional, specifies a specific channel to write out (will be written out to all channels gray scale).
 *
 * @return true if success
 */
static bool CreateBitmap( const TCHAR* Pattern, int32 DataWidth, int32 DataHeight, const struct FColor* Data, struct FIntRect* SubRectangle = NULL, IFileManager* FileManager = &IFileManager::Get(), FString* OutFilename = NULL, bool bInWriteAlpha = false, EChannelMask ChannelMask = EChannelMask::All );
```

`LoadANSITextFileToStrings` 读取一个 `ANSI` 编码的文本文件到一个字符串数组中，每行对应一个 `FString` 类型的对象。

```cpp
/**
 *	Load the given ANSI text file to an array of strings - one FString per line of the file.
 *	Intended for use in simple text parsing actions
 *
 *	@param	InFilename			The text file to read, full path
 *	@param	InFileManager		The filemanager to use - NULL will use &IFileManager::Get()
 *	@param	OutStrings			The array of FStrings to fill in
 *
 *	@return	bool				true if successful, false if not
 */
static bool LoadANSITextFileToStrings(const TCHAR* InFilename, IFileManager* InFileManager, TArray<FString>& OutStrings);
```

<br>

## 7.5 GConfig 类的使用

在我们平时的开发过程中，会经常有读写配置文件需求，需要 UE4 提供底层的文件读写功能，而且 C++ 原生也提供操作系统级别的文件读写。但读文件、写文件及解析内容等操作，还是不够方便。这时候我们就可以用 UE4 提供的专门读写配置文件的类——GConfig。GConfig 类提供了非常方便的API让我们可以快速读写配置。而且 GConfig 是属于 Core 模块的类，一般也不需要进行添加模块引用。接下来我们分别看一下具体使用方法。

### 7.5.1 写配置

> 下面这段代码截取自 UE4.27.2

```cpp
/**
 * @param Force Whether to create the Section on Filename if it did not exist previously.
 * @param Const If Const (and not Force), then it will not modify File->Dirty. If not Const (or Force is true), then File->Dirty will be set to true.
 */
FConfigSection* GetSectionPrivate( const TCHAR* Section, const bool Force, const bool Const, const FString& Filename );
void SetString( const TCHAR* Section, const TCHAR* Key, const TCHAR* Value, const FString& Filename );
void SetText( const TCHAR* Section, const TCHAR* Key, const FText& Value, const FString& Filename );
bool RemoveKey( const TCHAR* Section, const TCHAR* Key, const FString& Filename );
bool EmptySection( const TCHAR* Section, const FString& Filename );
bool EmptySectionsMatchingString( const TCHAR* SectionString, const FString& Filename );
```

下面是一个简单的写配置示例：

```cpp
GConfig->SetString(TEXT("MeSection"), TEXT("Name"), TEXT("AnniihlateSword"), FPath::GameDir()/"MyConfig.ini")
```

只需要调用 `GConfig` 的 `SetString` 函数就可以了。这个函数是保存一个字符串类型的变量到配置文件里。此函数共4个参数，第一个参数是指定 Section，即一个区块（可以理解为一个分类）；第二个参数是指定此配置的 Key；相当于此配置的具体名字；第三个参数为具体的值；最后一个参数是指定配置文件的路径，如果文件不存在，会自动创建此文件。

> 除了 SetString 外，GConfig 针对各种类型的数据都有相应的函数，如 SetInt、SetBool、SetFloat 等。具体的使用方法与 SetString 基本一样。

<br>

### 7.5.2 读配置

> 下面这段代码截取自 UE4.27.2

```cpp
bool GetString( const TCHAR* Section, const TCHAR* Key, FString& Value, const FString& Filename );
bool GetText( const TCHAR* Section, const TCHAR* Key, FText& Value, const FString& Filename );
bool GetSection( const TCHAR* Section, TArray<FString>& Result, const FString& Filename );
bool DoesSectionExist(const TCHAR* Section, const FString& Filename);
```

下面是一个简单的写配置示例：

```cpp
FString Result;
GConfig->GetString(TEXT("MeSection"), TEXT("Name"), Result, FPath::GameDir()/"MyConfig.ini")
```

![](/res/img/post/04-ue-daxiangwuxing-01/7-5-1.png)

与写配置不同，读配置则采用 Get 系列的函数。而且第三个参数从具体的值变为一个变量的引用。

> 使用 `GConfig` 来读写文件操作，变得非常简单。但需要注意的一点是写文件的时候，**通常在运行过程中，进行写入配置操作值并不会马上写入到文件。如果你需要马上生效，则可以调用 `GConfig->Flush()` 函数**。

```cpp
void Flush( bool Read, const FString& Filename=TEXT("") );
```

示例代码：

```cpp
void ATestActor::BeginPlay()
{
	Super::BeginPlay();
	
	GConfig->SetString(TEXT("MeSection"), TEXT("Name"), TEXT("AnniihlateSword"), FPaths::ProjectConfigDir() / "AnnihilateSword.ini");

	GConfig->Flush(false, FPaths::ProjectConfigDir() / "AnnihilateSword.ini");

	FString Result;
	GConfig->GetString(TEXT("MeSection"), TEXT("Name"), Result, FPaths::ProjectConfigDir() / "AnnihilateSword.ini");
	UE_LOG(LogTemp, Warning, TEXT("AnnihilateSword.ini - MeSection Config: %s"), *Result);
}
```

![](/res/img/post/04-ue-daxiangwuxing-01/7-5-2.png)

<br>
<br>

## 7.6 UE_LOG

### 7.6.1 简介

Log 作为开发中经常用到的功能，可以在任何需要的情况下记录程序运行的情况。Log 通常有以下几个作用。

1. 记录程序运行过程，函数调用过程。
2. 记录程序运行过程中，参与运算的数据信息。
3. 将程序运行中的错误信息反馈给开发团队。

<br>

### 7.6.2 查看 Log

- Game 模式

在 Game（打包）模式下，记录 Log 需要在启动参数后加 `-Log`

- 编辑器模式

在编辑器下，需要打开 Log 窗口（Window->DeveloperTools->OutputLog）

<br>

### 7.6.3 使用 Log

```cpp
UE_LOG(LogTemp, Warning, TEXT("Hello World"));
UE_LOG(LogTemp, Warning, TEXT("Show a String: %s"), *FString("Hello"));
UE_LOG(LogTemp, Warning, TEXT("Show a Int: %d"), 100);
```

UE_LOG 宏输出 Log，第一个参数为 Log 的分类（需要预先定义）。第二个参数为类型，有 Log、Warning、Error 三种类型。这三种类型区别是颜色不同，Log 为灰色，Warning 为黄色，Error 为红色。具体的输出内容为TEXT，可以根据需要自行构造。几种常用的符号如下：

1. `%s` 字符串（FString）
2. `%d` 整型数据（int32）
3. `%f` 浮点形（float）

<br>

### 7.6.4 自定义 Category

UE4 提供了多种自定义 Category 的宏。读者可自行参考 `LogMactos.h` 文件。这里介绍一种相对简单的自定义宏的方法。

```cpp
DEFINE_LOG_CATEGORY_STATIC(LogAnnihilateSword, Warning, All);
```

![](/res/img/post/04-ue-daxiangwuxing-01/7-6-1.png)

在使用 `DEFINE_LOG_CATEGORY_STATIC` 自定义 Log 分类的时候，我们可以将此宏放在你需要输出 Log 的源文件顶部。为了更方便地使用，可以将它放到 PCH 文件里，或者模块的头文件里（原则上是将 Log 分类定义放在被多数源文件 include 的文件里）。

<br>
<br>

## 7.7 字符串处理

在虚幻引擎中，“文字” 类型其实是一组类型：FName，FText 和 FString。这三种类型可以互相转换。当然还有 TCHAR 类型，只不过 TCHAR 不是虚幻引擎定义的字符串类。虚幻引擎把字符串进行细分，为的是加快访问速度，以及提供本地化支持。

- FName

简单而言，FName 是无法被修改的字符串，大小写不敏感。从语义上讲，名字也应该是唯一的。不管同样的字符串出现了多少次，在字符串表里只被存储一次。而借助这个哈希表，从字符串到 FName 的转换，以及根据 Key 查询 FName 会变得非常快速。

- FText

FText 表示一个 “被显示的字符串”。所有你希望 “显示” 的字符串都应该是 FText。因为 FText 提供了内置的本地化支持，也通过一张查找表来支持运行时本地化。FText 不提供任何的更改操作，对于被显示的字符串来说，“修改” 是一个非常不安全的操作。

- FString

FString 是唯一提供 **修改** 操作的字符串类。同时也意味着 FString 的消耗要高于 FName 和 FText。事实上，一般我们都使用 FString 来传递。尽管如此，Slate 控件的文字参数往往是 FText。这是为了强制要求本地化。

<br>
<br>

## 7.8 编译器相关技巧

### 7.8.1 “废弃” 函数的标记

在虚幻引擎中，有 **“废弃函数”** 的概念。准备废弃或者更改一个函数的时候，虚幻引擎不会立即废弃（否则影响范围太大），而是在编译期给出了一个警告。警告有点像这样： 

```txt
function Please update your code tothe new API before upgrading to the next release, otherwiseyourprojectwillnolongercompile.
```

需要注意的是，如果你使用编译器宏 message，会在任何时候都会输出消息。只要你的这段代码被编译，就会输出。而虚幻引擎的废弃函数则是在 “被调用” 的时候才会输出。

虚幻引擎对不同平台提供了不同的宏定义，对 GCC 使用 `__attribute__` 关键字，对 Visual Studio 使用 `__declspec` 关键字。然后调用 `deprecated` 关键字来输出。还有许许多多的编译器关键字可以用来实现诸多效果。

<br>

### 7.8.2 编译器指令实现跨平台

虚幻引擎被设计为一个跨平台的引擎。但是虚幻引擎所有平台公用一套源代码。那么如何才能完成对不同平台的兼容呢？

至少在Windows平台下的程序入口点，即WinMain函数，在Linux下是没有的。

那么这该如何是好？一种传统的办法是通过多态来解决。即存在一个基类，定义了抽象的接口，独立于操作系统存在。然后每个操作系统对应版本继承自这个基类，然后做出自己的实现。运行时根据当前操作系统来进行切换。

虚幻引擎部分采用了这种方案，但是对于一些情形，例如 Main 函数，采用多态跨平台就显得很困难了。于是 **虚幻引擎采用了编译期跨平台** 的方案，即：

**准备多个平台的实现，通过宏定义来切换不同的平台。** 例如虚幻引擎有一个通用的类：`FPlatformMisc`，包含了大量平台相关的工具函数。也有一系列的平台相关实现，如 `Linux` 下是 `FLinuxPlatformMisc`，随后通过一个 typedef 来完成，即：`typedef FLinuxPlatformMiscFPlatformMisc;`。于是就完成了编译期跨平台的过程。

那么有读者就要提出问题了，假如 Windows 定义一下，Linux 定义一下，那不就全乱了吗？

说得对，所以实际上，`.build.cs` 文件就会对此做出判断，根据编译平台的类型是 `Win64` 还是 `Linux`，会包含不同的文件夹，而不是一股脑儿全部包含，就不会出现冲突。

<br>
<br>

## 7.9 Images

> 我希望能够转换图片的类型！我希望能直接从硬盘导入图片作为贴图！应该怎么办呢？

虚幻引擎提供了 `ImagerWrapper` 作为所有图片类型的抽象层。其思想设计还是非常精妙的。我们可以这样来看待所有的图片格式类型：

1. 图片文件自身的数据是压缩后的数据，称为 CompressedData。
2. 图片文件对应的真正的 RGBA 数据，是没有压缩的，且与格式无关的（不考虑压缩带来的损失）的数据，称为 RawData。
3. 同样图片保存为不同的格式，RawData 不变（不考虑压缩），CompressedData 会随着图片格式不同而不同。
4. 因此，**所有图片的格式都可以被抽象为一个 CompressedData 和 RawData 的组合**，这就是 ImageWrapper 模块设计的思路。那么这样的设计思路有什么用处呢？让我们看几个案例：

- 读取 JPG 图片

1. 从文件中读取为 TArray 的二进制数据；
2. 用 `SetCompressData` 填充为压缩数据；
3. 使用 `GetRawData` 即可获取 RGB 数据。

- 转换 PNG 图片到 JPG

1. 从文件中读取为 TArray 的二进制数据；
2. 用 `SetCompressData` 填充为压缩数据；
3. 使用 `GetRawData` 即可获取RGB数据；
4. 将 RGB 数据填充到 JPG 类型的 `ImageWrapper` 中；
5. 使用 `GetCompressData`，即可获得压缩后的 JPG 数据；
6. 使用 `FFileHelper` 写入到文件中。

我在这里给出一个转换的代码案例：

> 同样要在 `.Build.cs` 文件文件中添加 `ImageWrapper` 模块，这里分享一个小技巧，不用退出 VS，直接右键 `.uproject` Generate Visual Studio project files.

```cpp
#include "IImageWrapperModule.h"
#include "IImageWrapper.h"
#include "HAL/FileManagerGeneric.h"
```

```cpp
bool ATestActor::CovertPNG2JPG(const FString& SourceName, const FString& TargetName)
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

```cpp
void ATestActor::BeginPlay()
{
	Super::BeginPlay();
	
	bool Success = CovertPNG2JPG(FPaths::ProjectContentDir() / TEXT("Pic/Source.png"), FPaths::ProjectContentDir() / TEXT("Pic/Target.jpg"));
	if (Success)
	{
		UE_LOG(LogTemp, Warning, TEXT("Covert PNG to JPG Success!"));
	}
}
```

![](/res/img/post/04-ue-daxiangwuxing-01/7-9-1.png)

基于同样的思路，我们可以用这样的方式来读取硬盘中的贴图：

1. 从硬盘中读取压缩过的图片文件到二进制数组，获得图片压缩后数据；
2. 将压缩后的数据借助 `ImageWrapper` 的 `GetRaw` 转换为原始 `RGB` 数据；
3. 填充原始的 `RGB` 数据到 `UTexture` 的数据中。

> 这段代码书上写的不太完整，这里打算后面在实践中有碰到再另写文章补充吧哈哈

<br>
<br>

The End.
