---
title: 【UE大象无形总结】第二部分虚幻引擎浅析【中卷】
tags: [UE4]
date: 2025-01-05
updated: 2025-01-05
cover: /res/img/post/06-ue-daxiangwuxing-03/cover.png
top_img: /res/img/site/background.png
---

![](/res/img/post/06-ue-daxiangwuxing-03/cover.png)

<br>

# 前言

> **若留得性命归来，在下愿救天下人。**
> 
> 参考书籍：《大象无形：虚幻引擎程序设计浅析 (罗丁力 [罗丁力])》
> 补充说明：本系列是个人对书中内容的实践

<br>
<br>

# 十一、虚幻引擎的渲染系统

> 虚幻引擎这么厉害的画面，是通过什么样的过程产生的呢？
> 从哪里开始渲染，从何处取回渲染结果？

虚幻引擎的渲染系统是一套非常庞大的系统，跨越多个线程，由大量的类辅助完成。因此本章的讲解将会采用 **“从大笔泼墨的结构性描述，到工笔细描的细节性描述”** 这样的思路。

<br>

## 11.1 渲染线程

对虚幻引擎有一定研究的读者一定曾听说 **“游戏线程（逻辑线程）”** 与 **“渲染线程”** 的说法。而笔者认为，对渲染线程的理解可以从这一句话入手：**“渲染线程是游戏线程的奴隶”** 。对应的含义是以下两点：

- **外包团队**：渲染线程是一个外包团队，其实质上并不知道自己真正执行了什么。游戏线程不停地向渲染线程对应的任务池派送任务，渲染线程只是埋头去执行。

- **追本溯源**：对渲染线程执行逻辑的分析离不开对游戏线程渲染相关代码的分析。游戏线程是木偶师，渲染线程是木偶，只看木偶不能窥得全貌，必须要看木偶师的想法，才能理解整个系统。

对于渲染线程的设计，两种方案是：

1. 渲染线程具有独立的逻辑系统。
2. 渲染线程只是简单地作为工作线程不断执行游戏线程赋予的工作。

大多数的引擎都选择了工作线程的方案。例如 `IdTech` 引擎采用了将渲染指令打包成一个一个的数据包，发送到渲染线程进行执行的方式，被称为 **渲染前端-后端** 方案。渲染后端线程依然是一个简单工作线程，只是随着历史进步，虚幻引擎借助 C++ 的语言特性给出了语法上更优美的方案。游戏线程方案能够大幅度简化线程同步的逻辑，渲染线程如果频繁阻塞同步，对整体效率会有极为不利的影响。

<br>

### 11.1.1 渲染线程的启动

虚幻引擎的 `Slate` 系统本身借助虚幻引擎 `RHI` 渲染，但是我们暂时抛开这一点，关注场景本身的渲染，不引入过多的讨论内容。

虚幻引擎在 `FEngineLoop::PreInit` 中对渲染线程进行初始化。具体的位置是在 `StartRenderingThread` 函数里面。此时虚幻引擎主窗口甚至尚未被绘制出来。

渲染线程的启动位于 `StartRenderingThread` 全局函数中。大致来说，这个函数执行了以下内容：

1. 创建渲染线程类实例。
2. 通过 `FRunnableThread::Create` 函数创建渲染线程。
3. 等待渲染线程准备好从自己的 `TaskGraph` 取出任务并执行。
4. 注册渲染线程。
5. 创建渲染线程心跳更新线程。

<br>

### 11.1.2 渲染线程的运行

渲染线程的主要执行内容在全局函数 `RenderingThreadMain` 中，实质上来说，渲染线程只是一个机械性地执行任务包的 “奴隶”，如图所示。游戏线程会借助 `EQUEUE_Render_COMMAND` 系列宏，向渲染线程的 `TaskMap` 中添加渲染任务。渲染线程则不断提取这些任务去执行。

![](/res/img/post/06-ue-daxiangwuxing-03/11-1-1.png)

另外，需要注意的一点是，渲染线程也并非直接向 `GPU` 发送指令，而是将渲染命令添加到 `RHICommandList`，也就是 `RHI` 命令列表中。由 `RHI` 线程不断取出指令，向 `GPU` 发送，并阻塞等待结果。此时 `RHI` 线程虽然阻塞，但是渲染线程依然正常工作，可以继续处理向 `RHI` 命令列表填充指令，如图所示。从而增加 `CPU` 时间的利用率，避免渲染线程凭空等待 `GPU` 端的处理。

![](/res/img/post/06-ue-daxiangwuxing-03/11-1-2.png)

<br>
<br>

## 11.2 渲染架构

### 11.2.1 延迟渲染

延迟渲染，英文名为 `Deferred Rendering`。**虚幻引擎对于场景中所有不透明物体的渲染方式，就是延迟渲染。而对于透明物体的渲染方式，则是 `前向渲染（Forward Rendering）`。**

所谓延迟渲染，是为了解决场景中由于物体、光照数量带来的计算复杂度问题。假如场景中有 5个物体和 5个光源，每个物体都单独计算光照，那么计算量将是 5×5=25次 光照计算。随着光源数量的提升，光照计算的计算量会呈几何级数上升。这样的代价是非常高昂的。而延迟渲染则是将 “光照渲染” 延迟进行。每次渲染，将会把物体的 BaseColor（基础颜色）、粗糙度、表面法线、像素深度等分别渲染成图片，然后根据这几张图，再逐个计算光照。带来的好处是，无论场景中有多少个物体，经过光照准备阶段之后，都只是变成了几张贴图，计算光照的时候，根据每个像素的深度，能够计算出像素在世界空间位置，然后根据表面法线、粗糙度等参数，带入公式进行计算。于是之前 5×5 的光照渲染计算量，被直线降低为 5+5 的计算量。5+5  的来历是：

- 5个物体 → 多张贴图：需要 5次渲染。
- 5个光源 → 逐光源计算光照结果：需要 5次渲染。
- 随着光源数量的增多，延迟渲染节省的计算量会越来越多。

但是延迟光照的代价是，半透明的物体无法被渲染。**因此在虚幻引擎里面，是先进行延迟光照计算不透明物体，然后借助深度排序，计算半透明物体的。** 因此我们可以看到玻璃折射材质，假如背后是不透明物体，效果还是相当有趣的，但是当其背后是半透明物体，就容易由于深度排序出现各种问题，这也是需要开发者进行注意的。

<br>

### 11.2.2 延迟渲染在 PostProcess 中的运用

刚刚介绍了虚幻引擎的延迟渲染架构，那么这样的知识是否有用处呢？我们都强调，理论联系实践，那么我们怎么理论联系实践呢？

首先，由于 虚幻引擎4 采用的延迟渲染架构，导致我们无法像 虚幻引擎3 那样，获取到光源的向量 `LightVector` —— 因为材质在被计算的时候，光照还没有发生。这也限制了我们在材质设计上的发挥，比如以前我们制作 `Tone Shader` 动画风格的材质，是可以在材质中完成的，现在我们已经无法这样做了。

那么我们的思路就必须跟随 虚幻引擎4 的延迟渲染架构进行转变，很多设计就得在 `Post Process` 中完成。我们可以在 `Post Process` 场景处理中，获取到之前渲染出来的缓冲区，如 `Scene Color`、`Rough`、`World Normal`等。这些东西的运用，就看大家仁者见仁智者见智了。

譬如可以通过对深度缓冲区运用边缘检测算法，从而检测出模型的外边缘，然后通过计算当前像素周围几个像素的 `World Normal` 与当前像素的 `World Normal` 的夹角变化值，从而检测出模型的内边缘。组合外边缘和内边缘，就能完成比较接近手绘的描边效果。

当然，这个方案的局限性在于，外边缘的粗细不容易被美术调整。关于描边方案的讨论，推荐大家查看 `《罪恶装备XXrd》` 的开发者访谈，其中有对其设计描边、动画渲染方案的思路的详细描述。

这里只是举例，对于 `Post Process` 中延迟渲染信息的利用，肯定有许许多多的方案。本人才疏学浅，只能举出这样的案例，如果希望在这个方面有更深层次的运用，可以参考虚幻引擎官方商城相关的案例。

<br>
<br>

## 11.3 渲染过程

上文主要阐述了虚幻引擎整体架构的设计。并且给出了一些名词的定义。但是上文没有阐述一个问题：屏幕上的画面究竟是如何呈现的？也就是说，我们在屏幕上看到了一棵 “树”，我们知道这棵树的结果——花花绿绿的像素，也知道这棵树的来源——导入的 fbx文件、创建的材质及定义的 `StaticMeshComponent` 静态网格物体组件，但是它们之间的过程是怎样的呢？如图所示。

![](/res/img/post/06-ue-daxiangwuxing-03/11-3-1.png)

前文有述，**虚幻引擎的渲染过程是一个异步的、多线程配合的过程。** 而为了方便描述和读者理解，这里不再考虑 `RHI` 线程的 `RHI` 绘制指令列表这一部分的异步性，也就是假定当一个渲染指令被填充到 `RHI` 绘制指令列表后，该指令直接被完成。我们把异步的分析主要集中在游戏线程和渲染线程之间的通信上。

同时，我们这一次的分析将会采用倒推的方式，从结果向来源逆推。笔者也尽量给出自己的分析过程，并提示对应源码的位置，让读者也能理解笔者分析的思路，而不是直接看到笔者的结果，相对来说更容易接受。

<br>

### 11.3.1 延迟渲染到最终结果

前文有对延迟渲染原理的描述。故这里将直接开始对虚幻引擎延迟渲染过程的步骤进行分析，如图所示。这部分主要分析的内容是 `FDeferedSceneRenderer::Render` 函数。

> UE4.27.2 中源码是 `FDeferredShadingSceneRenderer::Render(FRHICommandListImmediate& RHICmdList)`

![](/res/img/post/06-ue-daxiangwuxing-03/11-3-2.png)

- **准备**

在进行分析之前，我们势必要做出一些假定，从而确保我们的讨论能够继续。现在我们已经知晓了摄像机的角度，并且完成了场景中对象的收集，它们已经进行了归并，而且转化为了很容易获取顶点缓冲区、索引缓冲区，以及着色器信息的某种中间格式。至于这个归并和转化过程，我们将会在下一节分析。此时我们不妨丢掉实体的概念，想象你手里的只有一层 “皮囊”，那就是由顶点和索引规定的三角形面片组合，以及对应的着色器信息。每个对象都对应这么一个皮囊。这就是接下来分析的数据基础。

- **初始化视口**

在重新分配完渲染目标之后，我们要做的第一件事情是什么？不能一开始就着手绘制——因为此时我们拿到的是整个场景的皮囊。全部画完不是不可以，但是大量无用的计算会花在完全处在屏幕之外的皮囊上。注意观察那幅流程图11-4，即使是延迟渲染，也有几个Pass是使用前向渲染管线，逐个皮囊地绘制到缓冲区中的。

因此，此时**有必要进行可见性剔除**。需要注意的是，该函数命名为 `InitViews`，这是为了照顾不止一个视口的情况，例如编辑器的三视图。此时可以一口气对多个视图进行渲染，而不是逐视图地渲染，能够节约很多过程。如果一口气渲染多个视图，又只根据一个视图进行剔除，那么其他视图中这个对象会根本显示不出来。

**剔除过程分为三步：预设置可见性，可见性计算，完成可见性计算。**

<br>

#### 预设置可见性

**预设置可见性**：对应函数为 `PreVisibilityFrameSetup`。该函数主要执行以下内容：

1. **根据当前画质设置，设置 `TemporalAA` 的采样方式，同时确定采样位置。** 这个采样位置会用于微调接下来的矩阵。如果读者对这个微调不大理解，可以参考 `Epic` 关于 `TemporalAA` 采样的 PPT。简单来说：
    > `TemporalAA` 采样希望更精确地对一个像素覆盖的区域进行求解。一种思路是用更高的分辨率进行渲染，这样单个像素对应的区域实际上渲染了多个像素，然后将多个像素的值进行混合得到单一像素的值。这是一种 “暴力” 提高精确度的方法，代价也很高昂。另一种思路则更有技巧性，即每一帧渲染的时候，让这个像素覆盖的位置进行微弱偏移，然后混合前面几帧的渲染结果。打个比方，假如现在有一排 12个机器人狙击手向正前方射击，它们只会向正前方开火。此时处在两个狙击手中间的敌人会被 “漏掉” 从而不被击中。我们希望能够打中更多的敌人，一种方式是在每两个狙击手中间再放一个狙击手，这是 “暴力” 提高精度。另一种方式是让狙击手每次射击时，都往左右随机稍微偏一点，这样就能在不提升狙击手数量的情况下，击中更多的敌人。放在我们这里，就是让当前像素能够获得更多的信息。

2. **设置视口矩阵**，包括视口投影矩阵和转换矩阵。

<br>

#### 可见性计算

**可见性计算**：对应的函数为 `ComputViewVisibility`。该函数主要执行以下内容：

1. 初始化视口的一系列用于可视化检测的缓冲区。这些缓冲区实际上是一系列的位数组。用 0 和 1 代表是否可见。所以遵照习惯，翻译为位图，但是请将这个位图与 `bmp` 位图区分开来。这个对应的是 `BitMap`，也就是用于查询的逐位的数组。
2. 排除特殊情况，大多数情况下视口都使用平截头体剔除。对应的是模板函数 `FrustumCull`。该函数内部使用了 `ParallelFor` 函数来进行并行化的异步剔除。所谓平截头体剔除就是，摄像机视口的远近平面构成了一个类似梯台的体积，在该体积内的对象会被保留，在该体积外的对象的可视化检测缓冲区对应的比特位就会被设置为 0，表示不会被看见。
3. 对于编辑器来说，还有一个特殊的步骤。此时会对过小的线框直接剔除掉，以加速整体性能。如果是在线框模式下，此时所有的非线框也会被剔除掉。
4. 在非线框模式下，此时会对处于视口范围内，但是被别的对象遮挡的对象进行一次剔除。这个过程用的是上一帧计算遮挡结果。这个数据来源，请见下文中对 `HZB` 的介绍。
5. 在此之后，根据所有的可见性位图，设置每个需要渲染的对象的可见性状况，即 `Hiddenflag`。
6. 接下来会给每个对象返回自己是否可见的机会，这是开发者可以自行干涉的部分。下一节会讲述具体如何操作。
7. 最后一步是获取所有动态对象的渲染信息，对应函数是每个 `RenderProxy` 的 `GetDynamicMeshElements` 函数。下一节会详细讲述 `RenderProxy`，以及静态和动态的对象在渲染过程中的区别。这里请读者先作为一个步骤记忆下来，可以把 `RenderProxy` 理解为前文的比喻 “皮囊”，网格物体组件对应的皮囊是网格物体的 `RenderProxy`，持有顶点和索引信息，材质对应的皮囊是 `MaterialRenderProxy`，持有需要的着色器信息。在阅读完后文关于 `RenderProxy` 讲解的时候可以再次回头来看这个过程。

<br>

#### 完成可见性计算

**完成可见性计算**：对应的函数为 `PostVisibilityFrameSetup`。

1. 对半透明的对象进行排序。**半透明对象的渲染由于涉及互相遮挡，则必须按照从后往前的顺序来渲染，才能确保渲染结果的正确性。** 因此，需要此时完成排序。
2. 对每个光照确定当前光照可见的对象列表，这里也是使用平截头体剔除。
3. 初始化雾与大气的常量值。

在这个阶段也完成对阴影的计算。包括对覆盖整个世界的阴影、对固定光照的级联阴影贴图和对逐对象的阴影贴图的计算。

需要注意的是，虚幻引擎的剔除方式是借助 `ParallelFor` 的线性剔除，而不是借助八叉树等的树状剔除。对应的大致代码为：

```cpp
template<bool UseCustomCulling, bool bAlsoUseSphereTest, bool bUseFastIntersect>
static int32 FrustumCull(const FScene* Scene, FViewInfo& View)
{
	SCOPE_CYCLE_COUNTER(STAT_FrustumCull);

	FThreadSafeCounter NumCulledPrimitives;
	float MaxDrawDistanceScale = GetCachedScalabilityCVars().ViewDistanceScale;
	MaxDrawDistanceScale *= GetCachedScalabilityCVars().CalculateFieldOfViewDistanceScale(View.DesiredFOV);

	FSceneViewState* ViewState = (FSceneViewState*)View.State;
	const bool bHLODActive = Scene->SceneLODHierarchy.IsActive();
	const FHLODVisibilityState* const HLODState = bHLODActive && ViewState ? &ViewState->HLODVisibilityState : nullptr;

	//Primitives per ParallelFor task
	//Using async FrustumCull. Thanks Yager! See https://udn.unrealengine.com/questions/252385/performance-of-frustumcull.html
	//Performance varies on total primitive count and tasks scheduled. Check the mentioned link above for some measurements.
	//There have been some changes as compared to the code measured in the link

	const int32 BitArrayNum = View.PrimitiveVisibilityMap.Num();
	const int32 BitArrayWords = FMath::DivideAndRoundUp(View.PrimitiveVisibilityMap.Num(), (int32)NumBitsPerDWORD);
	const int32 NumTasks = FMath::DivideAndRoundUp(BitArrayWords, FrustumCullNumWordsPerTask);

	ParallelFor(NumTasks, 
		[&NumCulledPrimitives, Scene, &View, MaxDrawDistanceScale, HLODState](int32 TaskIndex)
		{
			QUICK_SCOPE_CYCLE_COUNTER(STAT_FrustumCull_Loop);
			const FPlane* PermutedPlanePtr = View.ViewFrustum.PermutedPlanes.GetData();
			const int32 BitArrayNumInner = View.PrimitiveVisibilityMap.Num();
			FVector ViewOriginForDistanceCulling = View.ViewMatrices.GetViewOrigin();
			float FadeRadius = GDisableLODFade ? 0.0f : GDistanceFadeMaxTravel;
			uint8 CustomVisibilityFlags = EOcclusionFlags::CanBeOccluded | EOcclusionFlags::HasPrecomputedVisibility;

			// Primitives may be explicitly removed from stereo views when using mono
			const int32 TaskWordOffset = TaskIndex * FrustumCullNumWordsPerTask;

			for (int32 WordIndex = TaskWordOffset; WordIndex < TaskWordOffset + FrustumCullNumWordsPerTask && WordIndex * NumBitsPerDWORD < BitArrayNumInner; WordIndex++)
			{
				uint32 Mask = 0x1;
				uint32 VisBits = 0;
				uint32 FadingBits = 0;
				uint32 DistanceCulledBits = 0;
				for (int32 BitSubIndex = 0; BitSubIndex < NumBitsPerDWORD && WordIndex * NumBitsPerDWORD + BitSubIndex < BitArrayNumInner; BitSubIndex++, Mask <<= 1)
				{
					int32 Index = WordIndex * NumBitsPerDWORD + BitSubIndex;
					const FPrimitiveBounds& Bounds = Scene->PrimitiveBounds[Index];
					float DistanceSquared = (Bounds.BoxSphereBounds.Origin - ViewOriginForDistanceCulling).SizeSquared();
					int32 VisibilityId = INDEX_NONE;

					if (UseCustomCulling &&
						((Scene->PrimitiveOcclusionFlags[Index] & CustomVisibilityFlags) == CustomVisibilityFlags))
					{
						VisibilityId = Scene->PrimitiveVisibilityIds[Index].ByteIndex;
					}

					// Preserve infinite draw distance
					float MaxDrawDistance = Bounds.MaxCullDistance < FLT_MAX ? Bounds.MaxCullDistance * MaxDrawDistanceScale : FLT_MAX; 
					float MinDrawDistanceSq = Bounds.MinDrawDistanceSq;

					// If cull distance is disabled, always show the primitive (except foliage)
					if (View.Family->EngineShowFlags.DistanceCulledPrimitives
						&& !Scene->Primitives[Index]->Proxy->IsDetailMesh())
					{
						MaxDrawDistance = FLT_MAX;
					}

					// Fading HLODs and their children must be visible, objects hidden by HLODs can be culled
					if (HLODState)
					{
						if (HLODState->IsNodeForcedVisible(Index))
						{
							MaxDrawDistance = FLT_MAX;
							MinDrawDistanceSq = 0.f;
						}
						else if (HLODState->IsNodeForcedHidden(Index))
						{
							MaxDrawDistance = 0.f;
						}
					}

					bool bDistanceCulled = DistanceSquared > FMath::Square(MaxDrawDistance + FadeRadius) || (DistanceSquared < MinDrawDistanceSq);

					// Store distane culled primitives so it can correctly culled when collecting RT primitives
					if (bDistanceCulled)
					{
						DistanceCulledBits |= Mask;
					}

					if (bDistanceCulled ||
						(UseCustomCulling && !View.CustomVisibilityQuery->IsVisible(VisibilityId, FBoxSphereBounds(Bounds.BoxSphereBounds.Origin, Bounds.BoxSphereBounds.BoxExtent, Bounds.BoxSphereBounds.SphereRadius))) ||
						(bAlsoUseSphereTest && View.ViewFrustum.IntersectSphere(Bounds.BoxSphereBounds.Origin, Bounds.BoxSphereBounds.SphereRadius) == false) ||
						(bUseFastIntersect ? IntersectBox8Plane(Bounds.BoxSphereBounds.Origin, Bounds.BoxSphereBounds.BoxExtent, PermutedPlanePtr) : View.ViewFrustum.IntersectBox(Bounds.BoxSphereBounds.Origin, Bounds.BoxSphereBounds.BoxExtent)) == false)
					{
						STAT(NumCulledPrimitives.Increment());
					}
					else
					{
						if (DistanceSquared > FMath::Square(MaxDrawDistance))
						{
							if (Scene->Primitives[Index]->Proxy->IsUsingDistanceCullFade())
							{
								FadingBits |= Mask;
							}
						}
						else
						{
							// The primitive is visible!
							VisBits |= Mask;
							if (DistanceSquared > FMath::Square(MaxDrawDistance - FadeRadius))
							{
								if (Scene->Primitives[Index]->Proxy->IsUsingDistanceCullFade())
								{
									FadingBits |= Mask;
								}
							}
						}
					}
				}
				if (FadingBits)
				{
					check(!View.PotentiallyFadingPrimitiveMap.GetData()[WordIndex]); // this should start at zero
					View.PotentiallyFadingPrimitiveMap.GetData()[WordIndex] = FadingBits;
				}
				if (VisBits)
				{
					check(!View.PrimitiveVisibilityMap.GetData()[WordIndex]); // this should start at zero
					View.PrimitiveVisibilityMap.GetData()[WordIndex] = VisBits;
				}
				if (DistanceCulledBits)
				{
					check(!View.DistanceCullingPrimitiveMap.GetData()[WordIndex]); // this should start at zero
					View.DistanceCullingPrimitiveMap.GetData()[WordIndex] = DistanceCulledBits;
				}
			}
		},
		!FApp::ShouldUseThreadingForPerformance() || (UseCustomCulling && !View.CustomVisibilityQuery->IsThreadsafe()) || CVarParallelInitViews.GetValueOnRenderThread() == 0 || !IsInActualRenderingThread()
	);

	return NumCulledPrimitives.GetValue();
}
```

寒霜引擎的开发者在 PPT 中也有论述，**并行化的线性结构剔除在性能上会优于基于树的剔除**。如果读者有兴趣深究此问题，可以在网上搜索 `《Culling Battlefiel》`，有详细的性能剖析报告。在虚幻引擎的该段代码的注释也给出了性能优势的剖析报告，如果读者拥有 UDN 的账号，也可以进入研究一番。

<br>

#### PrePass 预处理阶段

`PrePass` 的主要目的是为了降低 `Base Pass` 的渲染工作量。**通过渲染一次深度信息，如果某个像素点的深度检测失败，即不符合当前深度值，那么这个像素将不会进行工作量最大的像素渲染器计算。**

**PrePass 过程是可选的。** 在笔者的 `Windows` 平台下的主机上，这个步骤经常被略过执行。这取决于 `NeedsPrePass` 函数的返回结果。其判断以下几个条件：

1. 不是基于分块的 `GPU`，即 `TiledGPU`，例如 `ARM` 移动端平台很多就是。
2. 渲染器的 `EarlyZPassMode` 参数不为 `DDM_None`，或 `GEarlyZPassMovable` 不为 0。

```cpp
/**
 * Returns true if the depth Prepass needs to run
 */
static FORCEINLINE bool NeedsPrePass(const FDeferredShadingSceneRenderer* Renderer)
{
	return (Renderer->EarlyZPassMode != DDM_None || Renderer->bEarlyZPassMovable != 0);
}
```

两者必须同时满足，才会进行 `PrePass` 计算，否则是不会进行的。

此时参与渲染的包括不透明对象和 `Mask` 对象。这些对象中，静态网格物体构成了一套列表，即 `Scene` 的 `DepthDrawList` 与 `MaskedDepthDrawList`。在列表绘制完毕之后才会收集动态对象，再次绘制。

从渲染过程示意图中可以看到，对象的渲染始终按照设置渲染状态、载入着色器、设置渲染参数、提交渲染请求、写入渲染目标缓冲区的步骤进行。这个过程也是传统的图形 `API` 逐个绘制的基本思路。有些步骤可以归并，例如设置渲染状态，由于并非每次都需要修改，因此部分状态的设置被提前到最开头统一设置。这里只是给出一个通用的步骤过程，方便读者去理解。实际上虚幻引擎的渲染过程考虑了诸多问题，包括能否并行化、是否打开针对 `VR` 进行优化的 `InstancedStereo` 技术等，对于这些条件，虚幻引擎都有对应的渲染路径设计。我们的步骤分析依旧遵循抓大放小的思路，以主要路径为介绍重点，略去大量的特殊情况判断。

总体来说，预处理阶段的工作过程如下：

- **设置渲染状态**：这个步骤对应函数 `SetupPrePassView`，目的在于关闭颜色写入，打开深度测试与深度写入。`PrePass` 这个步骤不需要计算颜色，只需要计算每个不透明物体的像素的深度。
- **渲染三个绘制列表**：这三个绘制列表是由静态模型组成的，通过可见性位图控制是否可见。渲染顺序依次为：

1. 只绘制深度的列表 `PositionOnlyDepthDrawList`，这个列表里的对象只在深度渲染过程中起作用。
2. 深度绘制列表 `DepthDrawList`，这是最主要的不透明物体渲染列表。
3. 带蒙版的深度绘制列表 `MaskedDepthDrawList`，蒙版即对应材质系统中的 `Mask` 类型材质。

- **绘制动态的预处理阶段对象**：这个阶段会通过 `ShouldUseAsOccluder` 函数询问 `Render Proxy` 是否被当作一个遮挡物体，同时也会配合其他情况（是否为可移动等），决定是否需要在这个阶段绘制。

大体的过程就是如此，不过我们对一个核心的问题一笔带过：绘制过程。**绘制过程的步骤实质上是三个：设置绘制状态并载入着色器，设置着色器参数，提交渲染请求。** 至于最终的写入渲染目标，是在步骤之前就已经通过 `RHI` 的 `SetRenderTarget` 设置好了。`GPU` 会默认地将渲染结果写入到当前设置的渲染目标中，不需要我们人为进行干涉。由于多个 `Pass` 中绘制过程是类似的，典型案例是 `TStaticMeshDrawList::DrawVisible` 函数，故下面我们将会基于这个函数重点分析这三个步骤。

<br>

#### DrawVisible 绘制可见对象

前文有述，绘制可见对象的基础是可见对象列表。尤其是对于静态网格物体而言，是以 `TStaticMeshDrawList` 为单位进行成批绘制的。在绘制之前，每个绘制列表已经进行了排序，尽可能公用同样的绘制状态。具体如何排序，这里暂时略去不讲，在后文会详细描述。此时读者只需要知道，每个列表都公用以下着色器状态：

1. 顶点描述 `Vertex Declaration`
2. 顶点着色器 `Vertex Shader`
3. 壳着色器 `Hull Shader`
4. 域着色器 `Domain Shader`
5. 像素着色器 `Pixel Shader`
6. 几何着色器 `Geometry Shader`

> 换句话说，你可以把一个着色器状态看作是顶点描述和整个渲染管线用到的着色器对象的集合。而同一个列表中，这些东西都是一致的，区别只是在于渲染过程具体参数不同，如顶点缓冲区不同、索引缓冲区不同。

- **载入公共着色器信息**：这是逐列表完成的，对应的函数为 `SetBoundShaderState` 和 `SetSharedState`。

1. `SetBoundShaderState` 载入了需要的着色器。
2. `SetSharedState` 则因不同类型而异。对比较有代表性的 `TBasePass` 而言，是设置顶点着色器和像素着色器的参数。如果是透明物体，即透明度设置为 Additive、Translucent、Modulate，会在此时再次设置混合模式。

- **逐元素渲染**：需要注意的是，此时的元素并非是一比一对应，而是经过组合的，即并非 `Element`，而是 `BatchElement`。主要包含两个子步骤：

1. 对每个 `DrawingPolicy` 调用 `SetMeshRenderState` 函数，设置渲染状态。包括调用每个着色器的 `SetMesh` 函数，以设置与当前 `Mesh` 相关的参数，如变换矩阵等。
2. 调用当前 `Batch Element` 的 `DrawMesh` 函数，实际完成绘制。需要注意的是，顶点缓冲区和索引缓冲区是一开始就已经上传到显存准备好了的，这时候只需要调用 `RHICmdList` 的 `DrawIndexedPrimitive` 函数，指定好顶点缓冲区和索引缓冲区的位置，就可以了。这两个位置是包含在当前 `Element` 持有的信息里面的。

<br>

#### BasePass

`BasePass` 是一个极为重要的阶段，这个阶段完成的是通过逐对象的绘制，将每个对象和光照相关的信息都写入到缓冲区中。其中逐对象渲染的过程与前文对 `DrawVisible` 的分析过程一致，那么核心问题就是：

1. 逐对象绘制后的结果如何写入到缓冲区中？
2. 缓冲区中的内容如何管理？

理论上这里还应该有个问题：如何还原缓冲区中的参数，以进行光照计算？而这个问题将会留到下文对光照过程的分析中进行描述。

如果你对前文中 `PrePass` 阶段的绘制过程还有印象，那么你会发现，`BasePass` 的过程和 `PrePass` 的过程非常接近——它们都按照设置 **渲染视口、渲染静态数据 及 渲染动态数据** 三个步骤完成。

- **设置渲染视口**：这一次渲染视口的状态设置与 `PrePass` 略有差异。

1. 如果 `PrePass` 阶段已经写入深度，则深度写入被关闭，直接使用已经写入的深度结果进行深度检测。
2. 通过 `RHICmdList.SetBlendState` 打开了前 4个渲染目标的 RGBA 写入。`TStaticBlendStateWriteMask` 这个模板是用模板参数定义渲染目标是否可写入，最高支持 8个渲染目标，这里只打开了前 4个。

```cpp
RHICmdList.SetBlendState(
    TStaticBlendStateWriteMask<CW_RGBA, CW_RGBA, CW_RGBA, CW_RGBA>::GetRHI()
);
```

3. 设置了视口区域大小。这个大小会因为是否开启 `InstancedStereoPass` 而有所变化

- **渲染静态数据**：与 `PrePass` 有一定区别的是，如果之前已经进行了深度渲染，那么会首先渲染 `Masked` 蒙版对象，然后渲染普通的不透明对象。否则就会反过来，先渲染不透明对象，再渲染蒙版对象。

- **渲染动态数据**：区别不大。

需要注意的是，`BasePass` 阶段采用了 `MRT(Multi-Render Target)` 多渲染目标技术，从而允许 `Shader` 在渲染过程中向多个渲染目标进行输出。有读者就会问，那这几个渲染目标从哪里来的？答案是由当前请求渲染的视口 `（Viewport）` 分配的，并不归属 `SceneRenderer` 分配。对应的是 `FSceneViewport::BeginRenderFramw` 函数。

解决了 “从哪里来” 的问题，还有两个问题要解决：如何写入？向何处去？看来我们的分析方式带有了点古希腊哲学的味道。

如何向多个渲染目标写入？这个问题的答案并不在 `C++` 代码中，而是在 `Shader` 着色器代码中。如果你打开 `Engine/Shader/BasePassPixelShader.usf` 文件，就会看到著名的 `Main` 函数，定义了用到的多个输出（不考虑特殊情况）：`SV_Target0` 输出了颜色数据，`SV_Target1-3` 则作为 `GBuffer` 输出目标。整个着色器的代码非常长，这里只是大概描述过程：通过 `GetMaterialXXX` 系列函数，获取材质的各个参数，比如 `BaseColor` 基本颜色、`Metallic` 金属等。然后填充到 `GBuffer` 结构体中，最后通过 `EncodeGBuffer` 函数，把 `GBuffer` 结构体压缩、编码后，输出到前面定义的几个 `OutGBuffer` 渲染目标中。

至于向何处去，由于这几个渲染目标会一直存在于当前的渲染上下文中，所以在下个阶段可以直接使用。换句话说，类似于炒菜的时候，最开始下单的那个服务员会给一个盘子，负责每个步骤的人会从上一个步骤的人那接过盘子，在自己处理过后，继续传递给下个人。这几个渲染目标会放在盘子里，不断向下传递。这就是 `BasePass` 的功能。

<br>

#### RenderOcclusion 渲染遮挡

**虚幻引擎的遮挡计算，实质上是在 `PrePass` 中直接进行基于并行队列的硬件遮挡查询。** 除非在 `r.HZBOcclusion` 这个控制台变量被设置为 1 的情况下，或者有些特效需要的情况下，才会开启 `Hierarchical Z-Buffer Occlusion Cullin` 用作遮挡查询。

这个技术听上去十分厉害，但是全平台默认关闭 ，由于不清楚虚幻引擎在新的版本中是否会进一步完善和启用这个系统，故这里大致解释一下工作流程。如果读者对这个技术具体实现感兴趣，可以以 `Hierarchical Z-Buffer Occlusion Cullin` 为关键词搜索，能够获得大量有关实现的论文。

**总体来说，这个步骤是为了尽可能剔除处于屏幕内 但是被其他对象遮挡 的对象。** 之前我们有描述，**在视口初始化阶段，剔除了处于视锥体之外的对象。** 但是依然有大量对象处于视锥体内，却被其他对象遮挡。比如一座山背面的一大堆石头，这些石头能够正常通过我们的视锥体遮挡测试，却并不需要渲染。

因此，`HZB` 渲染遮挡技术被用于解决这个问题，通常的 `HZB` 步骤如下：

1. 预先准备屏幕的深度缓冲区，这个缓冲区将会作为深度测试的基础数据。因此，这个步骤必须在 `PrePass` 之后，如果没有 `PrePass`，则必须在 `BasePass` 之后。
2. 逐层创建缓冲区的 `Mipmap` 级联贴图。层级越高，贴图分辨率越低，对应的区域越大。请想象成打马赛克，马赛克越大，每个马赛克块遮盖的区域就越大。而每个马赛克的值对应这个区域 “最远” 元素到屏幕的距离（深度最大值）。
3. 计算所有需要进行测试的对象的包围球半径，根据这个半径，选择对应的深度缓冲区层级进行深度测试，判断是否被遮挡。这个的用意在于，如果对象较大，我们可以直接用更高层、更大块的马赛克进行测试——这个对象的深度若比这块马赛克对应的距离还远，那么该对象一定被遮挡，因为马赛克对应的是这一片区域中可见元素的最远距离。

需要注意的是，`OpenGL` 平台下不会进行这个测试。这个步骤中的第二步可以使用像素着色器多次绘制完成级联贴图层级，第三步则可以使用计算着色器 `ComputeShader`，或者使用顶点着色器进行计算，将结果写入到一个渲染目标中。从而借助 `GPU` 的高度并行化来加速这个遮挡剔除过程。

这个步骤写出的结果会被用于下一帧计算，而不是在本帧。

<br>

#### 光照渲染

本节对应的函数是 `RenderLights`。光照渲染与阴影渲染是分离的，阴影渲染是在视口初始化阶段完成，处于整个渲染流水线中非常靠前的位置。大体上的步骤如下：

1. **收集可见光源。** 除了对可见性标记的判断外，剔除的方法主要是视锥体裁剪。这一阶段会对每个光源构建 `FLightSceneInfo` 结构，然后通过 `ShouldRenderLights` 对光源是否需要渲染进行计算。这里不需要再进行一次视锥体裁剪来判断可见性，直接利用最开始视口初始化阶段保存的 `VisibleLightInfos` 信息，以当前Id作为索引查询即可获得结果。
2. **对收集好的光源进行排序。** 将不需要投射阴影、无光照函数（LightFunction）的光源排在前面，从而避免切换渲染目标。
3. 接下来的阶段因具体情况而异。如果是存在基于图块的光照（TiledDeferredLighting），则通过 `RenderTiledDeferredLighting` 对光照进行计算。如果是 PC 平台，大部分光照依然使用 `RenderLight` 函数进行光照计算。
4. 接下来会进行判断，如果当前平台支持着色器模型5（Shader Model5），则会计算反射阴影贴图与 `LPV` 信息。

略去较为复杂的部分不谈，核心光照渲染部分集中于 `RenderLight` 函数，每个光源都会调用该函数，其遍历所有视口，计算光照强度，并叠加到屏幕颜色上。该函数执行步骤如下：

1. 设置混合模式为叠加。
2. 判断光源类型：
   - **平行光源**：平行光源情况相对简单，对于虚幻引擎来说，平行光源只是一个没有范围概念，只有方向概念的光源。
    > a. 载入延迟渲染光照对应的顶点和像素着色器（对应为 `TDeferredLightVS` 和 `TDeferredLightPS`）。
    > b. 设置光照相关参数。
    > c. 绘制一个覆盖全屏幕的矩形，调用着色器。

    - **非平行光源**：此时比较复杂，如果摄像机在光源范围内，会出现问题。打个比方，如果点光源对应的几何体把摄像机扣住了，这个时候会出现渲染与预想效果不符的情况：球体一部分在摄像机后面，直接被剔除，没有渲染；球体的另一部分虽然在摄像机正面，但是有可能会被别的对象挡住。挡住的部分就无法进行光照计算。所以此时需要特殊步骤，如下图所示。

    > a. 判断摄像机是否在光照几何体范围内。
    > b. 如果是，关闭深度测试，从而避免背面被遮盖部分不进行光照渲染。
    > c. 否则，打开深度测试以加速渲染。
    > d. 载入着色器。
    > e. 设置光照相关参数。
    > f. 根据是点光源还是聚光灯，绘制一个对应的几何体，从而排除几何体外对象的渲染，加速光照计算。

![](/res/img/post/06-ue-daxiangwuxing-03/11-3-3.png)

至此，对整个渲染过程中相对较重要、有共性的部分，进行了粗略的介绍，对于大气、透明物体、后处理过程，恕笔者不再着重叙述，一方面为了避免读者阅读过于枯燥，另一方面，读者可以根据前文分析与代码对应，理解虚幻引擎渲染设计基本规律，然后自行去剖析这些部分。笔者在介绍过程中，尽可能给出步骤对应的函数，如果读者对某一个过程有兴趣，可以在虚幻引擎源代码中查询对应函数，了解更多的细节。

<br>

### 11.3.2 渲染着色器数据提供

#### ShaderMap

关于 `ShaderMap` 应该被翻译成什么，笔者相当纠结，最终决定保留英文原文。`Shader` 是着色器无误，但是 `Map` 这个概念很难描述，翻译为 “着色器图” 很容易被误解为是某个图像，**实际上这里的 `Map` 可以理解为一个表格。**

在讲述这个概念前，我们首先分析一下，虚幻引擎的着色器数量。如果你曾经重新编译着色器，会发现虚幻引擎的着色器数量高得令人恐怖，动辄是几千个着色器编译，看得让人心头发毛。而且这几千个着色器数量远远超过了材质的数量——我们发现了一个惊人的结论：单个材质会编译出多个着色器！

那么着色器按照什么样的方式被归类？方式如下：

1. 如果当前着色器类型继承自 `FMaterialShader`，则对每个材质类型编译出一组对应渲染管线的着色器，一般是顶点着色器、像素着色器组合。例如 `FLightFunctionVS/PS` 会对每个材质编译出一份实例。
2. 如果当前着色器类型继承自 `FMeshMaterialShader`，则对每个材质类型的每个顶点工厂类型编译出一组顶点着色器和像素着色器。实际上，前文介绍的从延迟渲染到最终结果的过程中，每个 `Pass` 基本都有对应的顶点着色器和像素着色器，例如 `BasePass` 对应的是 `TBasePassVS/PS` 这一对着色器组合。

这里提到了顶点工厂，**简而言之，顶点工厂负责抽象顶点数据以供后面的着色器获取，从而让着色器能够忽略由于顶点类型造成的差异**，比如普通的静态网格物体和使用 `GPU` 进行蒙皮的物体，两者顶点数据不同，但是通过顶点工厂进行抽象后，提供统一的数据获取接口，供后面的着色器调用。

**虚幻引擎的 “材质” 实质上是提供了一系列的函数接口，调用这些函数接口就能获得对应材质参数。** 单个材质不会产生一个完整的、带有具体执行过程的顶点着色器及像素着色器。读者不妨自行在材质编辑器中打开 **“HLSL代码”** 选项，能够看到材质转换完成的代码。

因此，对于继承自 `FMeshMaterialShader` 的着色器类型，读者可以这样理解：

1. 着色器不存在“动态链接”的概念，不可能出现动态切换某一部分函数的情况。换句话说，只要一部分函数的实现方式改变，最终会产生不一样的着色器原始代码，进而会编译出一个不一样的着色器对象。
2. 由于继承自FMeshMaterialShader的着色器类型需要适配不同的顶点工厂类型（同一份材质既可以被用于静态网格物体，也可以被用于骨架网格物体），也需要适配不同的材质类型（每个材质由于节点不同，获取某个材质参数的表达式也就不一样），对于顶点工厂 + 材质类型的组合，只要有一项不同，就必须产生新的一份着色器代码用于编译。因此，我们可以从两个维度来看待这个被虚幻引擎官方文档称为 “稀疏矩阵” 的着色器方案：
    > a. 从 **渲染阶段**（`PrePass、BasePass`等）角度来看，每个渲染阶段对应的着色器（如 `TBasePassVS/PS`）都需要对每个顶点工厂、每个材质类型产生出一组着色器，放置在对应材质类型的着色器缓存中。

    > b. 从 **材质类型** 角度来看，每个材质根据自身参与的渲染阶段不同，需要对每个顶点工厂类型、每个自己参与的渲染阶段，产生出一组着色器，放置在自己的着色器缓存中。此时被称为 `ShaderMap`。

这就形成一套 “三维” 的着色器矩阵——你可以想象一个三维魔方，长度为每个**材质类型**，宽度为每个**渲染阶段**，高度为**每个顶点工厂类型**，那么这个魔方的每一个方格都对应了一组着色器组合，当然由于材质不一定参与全部阶段，这个魔方有很多的空缺。

> 这里再做一些额外的解释：实质上虚幻引擎还会根据光照贴图类型做出不同的处理，笔者认为再加入这个影响，读者理解起来更加困难，而且读者在理解三个参量的情况后，理解不同光照贴图类型带来的不同处理，会更加容易。因此，下文中以三参量模型来进行解释。

<br>

#### 着色器数据选择

前文描述了巨大的着色器魔方。那么，如何根据当前阶段、当前材质类型、当前顶点工厂类型，从这个魔方中获得需要的着色器组合呢？

以一个静态网格物体渲染为例，对着色器数据选择的过程大体描述如下。请注意，以下描述的流程位于渲染线程中，对应 F 开头的类，而不是 U 开头的类。关于这两者区别的详细描述，可以参考下文关于场景代理的介绍。

1. 渲染线程遍历当前场景，添加静态网格到渲染列表。
2. 当一个静态网格 `FStaticMesh` 被添加到渲染列表时，其会根据自己参与的渲染阶段，通过 `F<渲染阶段（如BasePass)>DrawingPolicyFactory` 的 `AddStaticMesh` 函数开始选取当前渲染阶段对应的着色器。
3. 通过当前静态网格的 `MaterialRenderProxy` 材质渲染代理成员变量的 `GetMaterial` 函数，获得对应的 `FMaterial` 指针，该指针将会作为材质筛选的依据。随后调用 `Process<渲染阶段>Mesh`，例如在 `BasePass` 调用的是 `ProcessBasePassMesh`。
4. 该函数的最后一个参数会传入一个结构体，描述了一组渲染动作，其对应的 `Process` 函数会根据传入的光照贴图代理类型来做出不同的渲染方式，在此暂不分析光照贴图相关内容。其调用了 `AddMesh` 函数，在参数中创建了最重要的 `DrawingPolicy` 绘制代理。
5. 这个代理的类型即对应渲染阶段，传入参数包括顶点工厂指针和材质代理指针。至此，前文描述中的三个决定因素：渲染阶段、顶点工厂类型、材质类型均齐备，调用 `Get<渲染阶段>Shaders` 函数：
   > a. 该函数调用当前传入的材质类型的 `GetShader` 模板函数，从材质对象中抽取出对应的着色器，赋值给对应阶段的着色器变量，例如 `BasePass` 阶段。顶点工厂为 `VertexFactoryType` 时，抽取渲染管线着色器的案例代码如下（`LigthMapPolicyType` 为光照贴图类型，这里暂不讨论）：
   > 该类对应 `BasePassRendering.h` 头文件中的 `GetBasePassShaders` 函数。
   ```cpp
   VertexShader = Material.GetShader<TBasePassVS<LightMapPolicyType, false> >(VertexFactoryType);
   PixelShader = Material.GetShader<TBasePassPS<LightMapPolicyType, false> >(VertexFactoryType);
   ```
   
   > b. 材质（`FMaterial`）的 `GetShader` 函数则首先以当前的顶点工厂类型的 `id` 为索引，通过 `GetMeshShaderMap` 函数从 `OrderedMeshShaderMaps` 成员变量中查询到对应顶点工厂类型的MeshShaderMap。

   > c. 随后，调用当前的 `MeshShaderMap` 的 `GetShader` 函数，以当前着色器类型为参数，查询到实际对应的着色器。

**总结一下：实质上获取一组着色器组合需要的三个变量：渲染阶段、顶点工厂类型、材质类型。**

- **渲染阶段**：直接用代码区分，以模板形式完成定义，例如通过 `TBasePassPS<TUniformLightMapPolicy<Policy>, true>` 定义。最终会产生出一个 `FMeshMaterialShaderType*` 类型的变量，用于上文提到的 `MeshShaderMap->GetShader` 参数作为查询。

- **材质类型**：获取于当前网格物体的 `Material` 参数。实质上拉取材质就是从需要绘制的网格物体获取对应材质，再直接调用材质对应的 `GetShader` 函数完成。

- **顶点工厂类型**：获取于当前网格物体的 `VertexFactory` 参数。传给上文提到的材质对应的 `GetShader` 作为参数。

<br>
<br>

## 11.4 场景代理 SceneProxy

前文集中讲解渲染的整体过程，相当于以时间为维度来理解渲染过程。本节则更倾向于以对象为维度理解渲染过程。游戏中的对象如何最终变化为更适合的渲染数据，然后完成上文提到的渲染过程呢？

### 11.4.1 逻辑的世界与渲染的世界

**虚幻引擎的框架设计的一个基本思路是，逻辑与渲染分离。** 即存在一个逻辑上的游戏世界，包含各种场景中的 `Actor`，也包含 `Actor` 的组件。同时，也存在一个用于渲染的世界，这个世界中包含了呈现逻辑世界所需要的信息。渲染的世界如同布景一般，其只会呈现必要的内容——在当前摄像机范围内的，可以被渲染的内容。

这有一点类似 `MVC` 的思想，`Model` 就是我们的逻辑世界，其包含的是逻辑信息。例如一个 `Static Mesh Actor`，会包含位置、旋转、缩放，以及其对应的静态网格物体的引用。但是无论是 `Static Mesh Actor`，还是这个 `Actor` 持有的 `Static Mesh Component` 组件，均不会去处理 “渲染” 相关的代码，而是提供一个用于渲染的 “影子”，这个影子来完成具体的顶点构造、三角形填充等任务。那么，为何虚幻引擎采用这样的设计？让我们先进行一番简单的思考，如果是我们来设计一个简单的、带有渲染功能的引擎，一种最简单的方式就是：设计一个 `Actor` 基类，带有两个虚函数 `Tick` 和 `Render`。其中 `Tick` 负责每帧的逻辑更新，`Render` 负责在渲染的时候绘制自身。

**但是假如在多线程的情况下，这种设计存在一个明显的问题：有可能 `Tick` 函数执行到一半时，`Render` 函数就被调用了，其最终渲染结果不固定。**

即假如当前对象包含两个组件 A 和 B，渲染 A组件的时候，`Tick` 函数尚未调用，然后 `Tick` 函数被调用，整个对象位置移动了，然后渲染 B组件，此时 B组件会在新的位置。呈现的效果就是：A、B组件被撕裂开了。             

解决这样的问题，一种条件反射式的想法就是：在 `Tick` 函数和 `Render` 函数前后 **加锁**，同一时间如果正在 `Tick`，`Render` 函数就会被阻塞。反之亦然。

**这诚然是一种理想化的解决方案，但是实际生产中绝对不可能采用这种方案。** 这会导致 `Tick` 和 `Render` 被频繁阻塞。**多线程的每一次阻塞都是有成本的。** 对于 `Tick` 这种高频调用的函数，任何一点时间成本，都会积累为一个巨大的延时。为了避免这样的情况，**虚幻引擎设计了一种牺牲精确程度换取效率提升 的思路**，姑且把 `Tick` 函数所在的线程称为游戏线程，负责渲染的函数称为渲染线程，则：

1. 游戏线程遍历所有的逻辑对象，对于每个逻辑对象中可以显示的组件：
    a. 创建一个 `SceneProxy` 场景代理，完成场景代理初始化。
    b. `SceneProxy` 会根据对象位置等，**更新自己用于渲染的信息**。
2. **在合适的时机收集 `SceneProxy` 对象，并创建渲染指令，添加到渲染线程的渲染指令列表末尾。**
3. 渲染线程不断从渲染指令列表中取出渲染指令进行渲染。

`SceneProxy` 的世界，其实是逻辑世界的 “残影”，具有一定的延迟。同时，`SceneProxy` 的世界，也是逻辑世界的 “具象化”，因为其把逻辑世界中的逻辑信息剥离（去除了具体是玩家还是怪物，是树还是花这样的信息），转化为能够被渲染的三角形、渲染参数等信息。而值得一提的是，这个世界的更新是依赖于当前 `View` 的，也就是观察视角。根据观察视角，有些逻辑对象将不会创建自己的渲染内容，同样地，根据在屏幕上的大小，逻辑对象可以在诸多 `LOD` 中进行选择，以降低渲染的压力。

而最重要的一点是，这样的设计去除了对锁的依赖。渲染线程只会参考 `SceneProxy` 进行渲染，而不会关注原始的对象的情况。两者不会产生前文中提到的，由于线程异步带来的同步问题。

<br>

### 11.4.2 渲染代理的创建

任何一个可以被渲染的组件，都需要创建一个对应的渲染代理。因此在 `UPrimitiveComponent` 类中，提供了如下虚函数：

```cpp
/**
 * Creates a proxy to represent the primitive to the scene manager in the rendering thread.
 * @return The proxy object.
 */
virtual FPrimitiveSceneProxy* CreateSceneProxy()
{
	return NULL;
}
```

**这个函数将会在组件注册到世界中的时候被调用。而不是每帧都会调用。**

在自己创建的组件类中重载此函数，即可返回自己的 `SceneProxy`。而此时你拥有极大的自由度：你可以引入和提交 `Shader`，你可以手动提交顶点缓冲区和索引缓冲区，你可以手动请求绘制等。这种自由度甚至高过了 `ProcedureMeshComponent`，毕竟你可以引入自己的 `Shader`。

<br>

### 11.4.3 渲染代理的更新

渲染代理的更新分为两个部分进行：**一个是静态的部分**，通过重载 `DrawStaticElements` 函数完成；**另一个是动态的部分**，通过重载 `GetDynamicMeshElements` 完成。**其中静态部分的更新更加节省资源，动态部分则赋予了更高的自由度。**

静态部分更新遵循以下的规则：

1. `DrawStaticElements` 函数只会在场景收集的时候被调用，以收集静态物体。其通常无法在接下来的渲染过程中得到更新。
2. 当静态对象被移动、旋转或者缩放的时候，`OnTransformChanged` 函数会被调用。同时 `DrawStaticElements` 函数会被重新调用以更新。

而相对应的，动态部分的更新则频繁的多：

1. `GetDynamicMeshElements` 在每次场景开始渲染的时候就会由 `FDeferredShadingSceneRenderer` 通过 `FSceneRenderer` 调用。
2. `GetDynmaicMeshElements` 会根据当前视角角度，提交需要的 `Element`。

不管是静态部分的更新，还是动态部分的更新，最终都不是直接对对象进行绘制，而是将 `Mesh` 信息输入到传入的 `FMeshElementCollector` 中，最后会由渲染器统一进行渲染。而不是调用这两个函数进行渲染。

<br>

### 11.4.4 实战：创建新的渲染代理

如今我们已经有了充分的理论储备。那么接下来我们应该通过一个实战来检验所学到的知识。首先你需要创建一个继承自 `UPrimitiveComponent` 的组件类用于测试。具体的创建过程，笔者就不再赘述，我这里创建了一个叫作 `UTestCustomComponent` 的组件。头文件 `UTestCustomComponent.h` 代码如下：

`TestCustomComponent.h`

```cpp
#pragma once

#include "CoreMinimal.h"
#include "Components/PrimitiveComponent.h"
#include "TestCustomComponent.generated.h"

/**
 * 
 */
UCLASS()
class UERENDERTEST_API UTestCustomComponent : public UPrimitiveComponent
{
	GENERATED_BODY()
	
public:
	virtual FPrimitiveSceneProxy* CreateSceneProxy() override;
};
```

`TestCustomComponent.cpp`

```cpp
#include "TestCustomComponent.h"

class FTestCustomComponentSceneProxy : public FPrimitiveSceneProxy
{
public:
	FTestCustomComponentSceneProxy(const UPrimitiveComponent* InComponent)
		: FPrimitiveSceneProxy(InComponent)
	{
	}

	virtual uint32 GetMemoryFootprint() const override
	{
		return (sizeof(*this) + GetAllocatedSize());
	}

	virtual void GetDynamicMeshElements(const TArray<const FSceneView*>& Views, const FSceneViewFamily& ViewFamily, uint32 VisibilityMap, class FMeshElementCollector& Collector) const override
	{
		FBox TestDynamicBox = FBox(FVector(-100.0f), FVector(100.0f));
		DrawWireBox(Collector.GetPDI(0), GetLocalToWorld(), TestDynamicBox, FLinearColor::Red, ESceneDepthPriorityGroup::SDPG_Foreground, 10.0f);
	}

	virtual FPrimitiveViewRelevance GetViewRelevance(const FSceneView* View) const override
	{
		FPrimitiveViewRelevance Result;
		Result.bDrawRelevance = IsShown(View);
		Result.bDynamicRelevance = true;
		Result.bShadowRelevance = IsShadowCast(View);
		Result.bEditorPrimitiveRelevance = UseEditorCompositing(View);

		return Result;
	}

	// Inherited via FPrimitiveSceneProxy
	virtual SIZE_T GetTypeHash() const override
	{
		static SIZE_T UniquePointer;
		return reinterpret_cast<SIZE_T>(&UniquePointer);
	}
};

FPrimitiveSceneProxy* UTestCustomComponent::CreateSceneProxy()
{
	return new FTestCustomComponentSceneProxy(this);
}
```

再创建一个 Actor 用作测试：

`TestActor.h`

```cpp
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "TestActor.generated.h"

class UTestCustomComponent;

UCLASS()
class UERENDERTEST_API ATestActor : public AActor
{
	GENERATED_BODY()
	
public:	
	ATestActor();

protected:
	virtual void BeginPlay() override;

public:
	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "TestCustomComponent")
	UTestCustomComponent* TestCustomComponent;
};
```

`TestActor.cpp`

```cpp
#include "TestActor.h"
#include "TestCustomComponent.h"

ATestActor::ATestActor()
{
	PrimaryActorTick.bCanEverTick = false;

	TestCustomComponent = CreateDefaultSubobject<UTestCustomComponent>(TEXT("TestCustomComponent"));

	TestCustomComponent->SetupAttachment(GetRootComponent());
}

void ATestActor::BeginPlay()
{
	Super::BeginPlay();
}
```

如果你给一个 Actor 添加我们新创建的组件，然后运行游戏进行测试，你会发现出现了一个红色的线框盒子。

![](/res/img/post/06-ue-daxiangwuxing-03/11-4-1.png)

回顾一下我们的代码，可以发现，现在我们已经能够借助 `DrawWiredBox` 系列函数，来提交绘制请求。类似地，我们也可以请求绘制线条。同时，也可以借助 `DynamicMeshBuilder`，来实现构建动态的网格，并请求渲染。

当然，你肯定会说，我们绕了如此大的一个圈子，以实现虚幻引擎提供的 `ProceduralMeshComponent` 组件早已实现的功能，是否有一些杀鸡用牛刀的嫌疑？其实，此处描述的内容是为了印证之前对虚幻引擎场景代理机制的介绍和说明，为了方便读者更好理解。借助场景代理，能够实现的功能远比此处讲述的 `ProceduralMeshComponent` 组件能够实现的要多很多。那么接下来，我们就要超越 `ProceduralMeshComponent` 了。

<br>

### 11.4.5 进阶：创建静态渲染代理（可以跳过）

> **这里书中代码有点问题，笔者查阅其他资料修改代码后，还是有问题，没有达到书中的效果，因为这块我暂时没有需求去应用，所以这个实践我决定就暂时跳过吧，后面碰到实际应用情况再研究~**

`ProceduralMeshComponent` 在 虚幻引擎4.11 的时代，是没有办法进行静态构建的，也许你阅读到这一段的时候，虚幻引擎已经支持静态的程序化模型（虚幻引擎4.13 的特性，笔者撰写时 4.13 还没有发布），不过在笔者撰写本节时是不可行的。而我们可以通过重载 `DrawStaticElements` 函数来实现返回静态的模型。这个模型拥有一部分可变性——你可以在编辑器中调整参数，例如修改位置和缩放，然后模型会更新（其 `RenderProxy` 渲染代理会被标记为 `Dirty`，从而请求更新）。但是在运行时，其只会在组件初始化的时候返回一次模型信息，之后就不会再返回，能够节省大量的性能。对于那些希望在编辑阶段修改，在运行时固定的模型，这个机制尤为合适。

而同样地，为了完成这样的功能，我们需要做的步骤就会多很多。大体来说，我们需要完成的步骤有以下：

1. 创建顶点缓冲区类和索引缓冲区类，如果你学过 `DirectX` 或者 `OpenGL`，你对这个过程应该会非常熟悉。
2. 创建顶点工厂，顶点工厂可以统一不同类型的顶点信息，转变成可以被统一 `Shader` 使用的顶点。这一点笔者会在接下来说明。
3. 在 `RenderProxy` 的构造函数中：
   a. 填充顶点缓冲区和索引缓冲区类中的数据（请注意，此时还没有绑定缓冲区到显存，这些数据依然在内存中）。
   b. 初始化顶点工厂。
   c. 请求在渲染线程中，完成对顶点、索引缓冲区和顶点工厂的初始化。
4. 重载 `RenderProxy` 的 `DrawStaticElements` 函数，创建一个 `Mesh` 并请求绘制。

这里的步骤相当的多。如果你对 `DirectX` 或者 `OpenGL` 的渲染过程比较熟悉，那么理解起来可能更加简单。如果你不是非常熟悉，建议你先阅读 `DirectX` 或者 `OpenGL` 渲染流程的相关知识。在此笔者只做简单的介绍。

我们首先需要在我们的 `.build.cs` 控制文件中，引入两个新的模块 `RenderCore` 和 `RHI`：

> 我这里的项目名称是 UERenderTest

`UERenderTest.Build.cs`

```csharp
PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "InputCore",
    "RenderCore", "RHI"
});
```

随后我们就可以继续在 `TestCustomComponent.cpp` 文件中，书写我们的代码了。首先需要定义我们自己的 `VertexBuffer` 和 `IndexBuffer`。理论上说，甚至可以定义自己的顶点类型，这在虚幻引擎中当然是支持的。不过为了简化这部分的复杂度，可以使用虚幻引擎为我们预定义 `FDynamicMeshVertex`。顺便一提，如果希望自行定义顶点结构，`Shader` 也需要酌情修改。

```cpp
/** Vertex Buffer */
class FTestCustomComponentVertexBuffer : public FVertexBuffer
{
public:
	TArray<FDynamicMeshVertex> Vertices;

	virtual void InitRHI() override
	{
		FRHIResourceCreateInfo CreateInfo;
		void* VertexBufferData = nullptr;
		VertexBufferRHI = RHICreateAndLockVertexBuffer(Vertices.Num() * sizeof(FDynamicMeshVertex), BUF_Static, CreateInfo, VertexBufferData);
		FMemory::Memcpy(VertexBufferData, Vertices.GetData(), Vertices.Num() * sizeof(FDynamicMeshVertex));
		RHIUnlockVertexBuffer(VertexBufferRHI);
	}
};

/** Index Buffer */
class FTestCustomComponentIndexBuffer : public FIndexBuffer
{
public:
	TArray<int32> Indices;

	virtual void InitRHI() override
	{
		FRHIResourceCreateInfo CreateInfo;
		IndexBufferRHI = RHICreateIndexBuffer(sizeof(int32), Indices.Num() * sizeof(int32), BUF_Static, CreateInfo);
		void* Buffer = RHILockIndexBuffer(IndexBufferRHI, 0, Indices.Num() * sizeof(int32), RLM_WriteOnly);
		FMemory::Memcpy(Buffer, Indices.GetData(), Indices.Num() * sizeof(int32));
		RHIUnlockIndexBuffer(IndexBufferRHI);
	}
};
```

每个缓冲区都包含了一个数据数组——位于内存中，用于填充顶点和索引数据，以及一个 `InitRHI` 函数。通过重载该函数，就可以在显存中创建顶点和索引缓冲区。随后通过锁存、内存拷贝、取消锁定等操作，上传我们的顶点和索引数据到显存中。

在定义完毕顶点和索引缓冲区后，我们还需要定义一个相对来说比较特殊的结构，那就是 `FVectexFactory`（顶点工厂）。顶点工厂这个概念是虚幻引擎创造的，而不是 `DirectX` 或者 `OpenGL` 的概念。顶点工厂是为了抽象不同类型的顶点数据，转化为统一的三角面数据。举个例子，对于带有骨骼的模型来说，虚幻引擎提供了 `GPUSkinnedVertexFactory` 系列类，其根据原始的模型和骨骼数据在 `GPU` 端借助 `Shader` 完成蒙皮变形的操作，就是根据骨骼位置计算出最终的顶点。在这之后，骨骼模型与普通的静态网格物体已经没有什么太大的区别，可以执行同样的渲染管线。

在这个案例中，我们来实现一个最为简单的顶点工厂，其只是简单地定义了顶点数据的格式。代码如下：

```cpp
/** Vertex Factory */
class FTestCustomComponentVertexFactory : public FLocalVertexFactory
{
public:
	FTestCustomComponentVertexFactory()
		: FLocalVertexFactory(ERHIFeatureLevel::SM5, "FTestCustomComponentVertexFactory")
	{}

	/** Initialization */
	void Init(const FTestCustomComponentVertexBuffer* VertexBuffer)
	{
		if (IsInRenderingThread())
		{
			// Initialize the vertex factory's stream
			FLocalVertexFactory::FDataType NewData;
			NewData.PositionComponent = STRUCTMEMBER_VERTEXSTREAMCOMPONENT(VertexBuffer, FDynamicMeshVertex, Position , VET_Float3);
			NewData.TextureCoordinates.Add(
				FVertexStreamComponent(VertexBuffer, STRUCT_OFFSET(FDynamicMeshVertex, TextureCoordinate), sizeof(FDynamicMeshVertex), VET_Float2)
			);
			NewData.TangentBasisComponents[0] = STRUCTMEMBER_VERTEXSTREAMCOMPONENT(VertexBuffer, FDynamicMeshVertex, TangentX, VET_PackedNormal);
			NewData.TangentBasisComponents[1] = STRUCTMEMBER_VERTEXSTREAMCOMPONENT(VertexBuffer, FDynamicMeshVertex, TangentZ, VET_PackedNormal);
			SetData(NewData);
		}
		else
		{
			FTestCustomComponentVertexFactory* VertexFactory = this;
			ENQUEUE_RENDER_COMMAND(InitTestCustomComponentVertexFactor)([VertexFactory, VertexBuffer](FRHICommandListImmediate& RHICmdList)
				{
					// Initialize the vertex factory's
					FLocalVertexFactory::FDataType NewData;
					NewData.PositionComponent = STRUCTMEMBER_VERTEXSTREAMCOMPONENT(VertexBuffer, FDynamicMeshVertex, Position, VET_Float3);
					NewData.TextureCoordinates.Add(
						FVertexStreamComponent(VertexBuffer, STRUCT_OFFSET(FDynamicMeshVertex, TextureCoordinate), sizeof(FDynamicMeshVertex), VET_Float2)
					);
					NewData.TangentBasisComponents[0] = STRUCTMEMBER_VERTEXSTREAMCOMPONENT(VertexBuffer, FDynamicMeshVertex, TangentX, VET_PackedNormal);
					NewData.TangentBasisComponents[1] = STRUCTMEMBER_VERTEXSTREAMCOMPONENT(VertexBuffer, FDynamicMeshVertex, TangentZ, VET_PackedNormal);
					VertexFactory->SetData(NewData);
				}
			);
		}
	}
};
```

相对来说，这一段代码的量会比较长，仔细观察会发现代码有一段重复，为何同样的代码要写两遍呢？那是因为，**我们必须要确保 我们的顶点工厂初始化在渲染线程中执行。也就是说，我们会首先判定当前是不是渲染线程，如果是，就会直接执行初始化代码；否则就需要借助 `ENQUEUE_RENDER_COMMAND` 系列宏去请求在渲染线程中执行当前代码。** 如果对这个机制不大明白，可以再回顾一下之前阐述多线程渲染架构的内容。

> 我这里的版本是 UE4.27.2，只能使用 `ENQUEUE_RENDER_COMMAND` 宏

那么这一段初始化代码的具体含义是什么呢？我们都知道，虚幻引擎的渲染架构实际上是 `DirectX` 和 `OpenGL` 的一个统一抽象，也就是说，这段代码一定能在 `DirectX` 或者 `OpenGL` 中找到对应的概念操作。`DirectX11` 中有 `Layer` 的概念：对上传的每个顶点数据，需要手动指定其如何 “切分” 为具体的顶点属性。由于允许我们自己定义顶点数据，因此我们需要手动地把上传到显存中的顶点数据切分成具体的属性（或者说成员变量），然后传递给着色器。这个过程，实质上是制定每个属性到当前顶点数据开头的 “偏移” 与 “长度”。这就像我们来到陌生的街上，有一排店面，需要知道这条街上店面的归属情况。为了达到这样的目的，由于店主的店面都连接在一起，我们只需要收集每个店主第一家店面的门牌号，以及店面的数量，就可以把整条街的店面切分为每个店主自己所属的段。

同样地，这一段代码也是一种切分的过程。其实际上是将 `VertexBuffer` 中的数据切分之后，指定给对应的 `DataType` 中对应的字段。这个 `DataType` 已经定义好了一系列的数据，包括位置、贴图UV、法线。由于我们继承于 `LocalVertexFactory`，因此我们只需要指定局部空间的位置，从本地空间到世界空间的转换将会由其自动完成。笔者会在接下来对调用的讲解中，进一步讲解这个转换过程。

为了把我们刚刚定义的顶点缓冲区、索引缓冲区和顶点工厂引入到 `SceneProxy` 中，我们需要增加以下三个成员变量：

```cpp
FTestCustomComponentIndexBuffer IndexBuffer;
FTestCustomComponentVertexBuffer VertexBuffer;
FTestCustomComponentVertexFactory VertexFactory;
```

而为了方便测试，也需要修改我们的构造函数，并增加一些测试性顶点来方便显示。新的构造函数如下：

```cpp
FTestCustomComponentSceneProxy(const UPrimitiveComponent* InComponent)
	: FPrimitiveSceneProxy(InComponent)
{
	const float BoxSize = 100.0f;
	//填充顶点
	VertexBuffer.Vertices.Add(FVector(0.0f));
	VertexBuffer.Vertices.Add(FVector(BoxSize, 0.0f, 0.0f));
	VertexBuffer.Vertices.Add(FVector(0.0f, BoxSize, 0.0f));
	VertexBuffer.Vertices.Add(FVector(0.0f, 0.0f, BoxSize));
	//填充索引
	IndexBuffer.Indices.Add(0);
	IndexBuffer.Indices.Add(1);
	IndexBuffer.Indices.Add(2);
	IndexBuffer.Indices.Add(0);
	IndexBuffer.Indices.Add(2);
	IndexBuffer.Indices.Add(3);
	IndexBuffer.Indices.Add(0);
	IndexBuffer.Indices.Add(3);
	IndexBuffer.Indices.Add(1);
	IndexBuffer.Indices.Add(3);
	IndexBuffer.Indices.Add(2);
	IndexBuffer.Indices.Add(1);
	//初始化
	VertexFactory.Init(&VertexBuffer);
	BeginInitResource(&IndexBuffer);
	BeginInitResource(&VertexBuffer);
	BeginInitResource(&VertexFactory);
}
```

这里增加了 4个顶点与 12个索引数据，最终构造了一个三棱锥。你也可以自己修改为一个立方体或者球体。在把这些数据填充到缓冲区之后，我们需要用顶点缓冲区初始化顶点工厂。这个初始化是一个类似 “预初始化” 的过程。真正完成显卡关联的数据初始化，是使用 `BeginInitResource` 函数完成的。`BeginInitResource` 函数内部也是借助 EQUEUE RENDER COMMAND 系列宏向渲染线程请求初始化。**务必注意的是，顶点工厂初始化有两次：一次是 `CPU` 端数据的初始化，随后在渲染线程还有一次资源初始化。**

相对应地，我们申请了资源，就务必需要手动释放，这很类似 `COM` 资源的申请与释放。我们可以在析构函数中完成这个过程，如下：

```cpp
virtual ~FTestCustomComponentSceneProxy()
{
	VertexBuffer.ReleaseResource();
	IndexBuffer.ReleaseResource();
	VertexFactory.ReleaseResource();
}
```

同样地，我们也需要在 `GetViewRelevance` 函数中简单地说明当前网格可见：

```cpp
virtual FPrimitiveViewRelevance GetViewRelevance(const FSceneView* View) const override
{
	FPrimitiveViewRelevance Result;
	Result.bDrawRelevance = true;
	Result.bDynamicRelevance = true;
	Result.bShadowRelevance = true;

	return Result;
}
```

到这里终于可以开始完成最终的渲染绘制调用了。我们撰写了这么多的代码，终于到了展现努力结果的时候了。让我们完成 `DrawStaticElement` 函数吧。

```cpp
virtual void DrawStaticElements(FStaticPrimitiveDrawInterface* PDI) override
{
	FMeshBatch Mesh;
	FMeshBatchElement& BatchElement = Mesh.Elements[0];
	BatchElement.IndexBuffer = &IndexBuffer;
	Mesh.bWireframe = false;
	Mesh.VertexFactory = &VertexFactory;
	Mesh.MaterialRenderProxy = UMaterial::GetDefaultMaterial(MD_Surface)->GetRenderProxy();

	FBoxSphereBounds PreSkinnedOutBounds;
	GetPreSkinnedLocalBounds(PreSkinnedOutBounds);
	BatchElement.PrimitiveUniformBuffer = CreatePrimitiveUniformBufferImmediate(GetLocalToWorld(), GetBounds(), GetLocalBounds(), PreSkinnedOutBounds, true, true);
	BatchElement.FirstIndex = 0;
	BatchElement.NumPrimitives = IndexBuffer.Indices.Num() / 3;
	BatchElement.MinVertexIndex = 0;
	BatchElement.MaxVertexIndex = VertexBuffer.Vertices.Num() - 1;
	Mesh.ReverseCulling = IsLocalToWorldDeterminantNegative();
	Mesh.Type = PT_TriangleList;
	Mesh.DepthPriorityGroup = SDPG_Foreground;
	Mesh.bCanApplyViewModeOverrides = false;
	Mesh.bDisableBackfaceCulling = false;
	PDI->DrawMesh(Mesh, 1.0f);
}
```

这里需要介绍两个概念：`MeshBatch` 和 `BatchElement`。`MeshBatch` 是一系列的网格的集合，而 `BatchElement` 则是单个的网格元素。这似乎还是很难理解，那么，一个简单的例子是，`MeshBatch` 存储了整个顶点缓冲区（即指定了当前的顶点工厂），而每个单独的 `BatchElement` 都有自己的索引缓冲区。当然，它们也有更多的区别，具体的内容繁多，在此不详细讲解。

在代码中，有一个比较重要的代码是对 `PrimitiveUniformBuffer` 的填充，这是通过 `CreatePrimitiveUniformBufferImmediate` 函数完成的。其中就指定了从本地空间到世界空间的变换矩阵。

在填充了全部的数据之后，通过 `PDI` 的 `DrawMesh` 函数，渲染整个静态网格。

最后还得重载 `CalcBounds` 函数，返回当前组件的包围盒，否则静态渲染结果会被剔除，从而无法被看见。

```cpp
FBoxSphereBounds UTestCustomComponent::CalcBounds(const FTransform& LocalToWorld) const
{
	return FBoxSphereBounds(FVector(0.0f), FVector(100.0f), 100.0f).TransformBy(LocalToWorld);
}
```

<br>

#### 最终的代码如下

```cpp
// Fill out your copyright notice in the Description page of Project Settings.


#include "TestCustomComponent.h"


/** Vertex Buffer */
class FTestCustomComponentVertexBuffer : public FVertexBuffer
{
public:
	TArray<FDynamicMeshVertex> Vertices;

	virtual void InitRHI() override
	{
		FRHIResourceCreateInfo CreateInfo;
		void* VertexBufferData = nullptr;
		VertexBufferRHI = RHICreateAndLockVertexBuffer(Vertices.Num() * sizeof(FDynamicMeshVertex), BUF_Static, CreateInfo, VertexBufferData);
		FMemory::Memcpy(VertexBufferData, Vertices.GetData(), Vertices.Num() * sizeof(FDynamicMeshVertex));
		RHIUnlockVertexBuffer(VertexBufferRHI);
	}
};

/** Index Buffer */
class FTestCustomComponentIndexBuffer : public FIndexBuffer
{
public:
	TArray<int32> Indices;

	virtual void InitRHI() override
	{
		FRHIResourceCreateInfo CreateInfo;
		IndexBufferRHI = RHICreateIndexBuffer(sizeof(int32), Indices.Num() * sizeof(int32), BUF_Static, CreateInfo);
		void* Buffer = RHILockIndexBuffer(IndexBufferRHI, 0, Indices.Num() * sizeof(int32), RLM_WriteOnly);
		FMemory::Memcpy(Buffer, Indices.GetData(), Indices.Num() * sizeof(int32));
		RHIUnlockIndexBuffer(IndexBufferRHI);
	}
};

/** Vertex Factory */
class FTestCustomComponentVertexFactory : public FLocalVertexFactory
{
public:
	FTestCustomComponentVertexFactory()
		: FLocalVertexFactory(ERHIFeatureLevel::SM5, "FTestCustomComponentVertexFactory")
	{}

	/** Initialization */
	void Init(const FTestCustomComponentVertexBuffer* VertexBuffer)
	{
		if (IsInRenderingThread())
		{
			// Initialize the vertex factory's stream
			FLocalVertexFactory::FDataType NewData;
			NewData.PositionComponent = STRUCTMEMBER_VERTEXSTREAMCOMPONENT(VertexBuffer, FDynamicMeshVertex, Position , VET_Float3);
			NewData.TextureCoordinates.Add(
				FVertexStreamComponent(VertexBuffer, STRUCT_OFFSET(FDynamicMeshVertex, TextureCoordinate), sizeof(FDynamicMeshVertex), VET_Float2)
			);
			NewData.TangentBasisComponents[0] = STRUCTMEMBER_VERTEXSTREAMCOMPONENT(VertexBuffer, FDynamicMeshVertex, TangentX, VET_PackedNormal);
			NewData.TangentBasisComponents[1] = STRUCTMEMBER_VERTEXSTREAMCOMPONENT(VertexBuffer, FDynamicMeshVertex, TangentZ, VET_PackedNormal);
			SetData(NewData);
		}
		else
		{
			FTestCustomComponentVertexFactory* VertexFactory = this;
			ENQUEUE_RENDER_COMMAND(InitTestCustomComponentVertexFactor)([VertexFactory, VertexBuffer](FRHICommandListImmediate& RHICmdList)
				{
					// Initialize the vertex factory's
					FLocalVertexFactory::FDataType NewData;
					NewData.PositionComponent = STRUCTMEMBER_VERTEXSTREAMCOMPONENT(VertexBuffer, FDynamicMeshVertex, Position, VET_Float3);
					NewData.TextureCoordinates.Add(
						FVertexStreamComponent(VertexBuffer, STRUCT_OFFSET(FDynamicMeshVertex, TextureCoordinate), sizeof(FDynamicMeshVertex), VET_Float2)
					);
					NewData.TangentBasisComponents[0] = STRUCTMEMBER_VERTEXSTREAMCOMPONENT(VertexBuffer, FDynamicMeshVertex, TangentX, VET_PackedNormal);
					NewData.TangentBasisComponents[1] = STRUCTMEMBER_VERTEXSTREAMCOMPONENT(VertexBuffer, FDynamicMeshVertex, TangentZ, VET_PackedNormal);
					VertexFactory->SetData(NewData);
				}
			);
		}
	}
};


class FTestCustomComponentSceneProxy : public FPrimitiveSceneProxy
{
public:
	FTestCustomComponentSceneProxy(const UPrimitiveComponent* InComponent)
		: FPrimitiveSceneProxy(InComponent)
	{
		const float BoxSize = 100.0f;
		//填充顶点
		VertexBuffer.Vertices.Add(FVector(0.0f));
		VertexBuffer.Vertices.Add(FVector(BoxSize, 0.0f, 0.0f));
		VertexBuffer.Vertices.Add(FVector(0.0f, BoxSize, 0.0f));
		VertexBuffer.Vertices.Add(FVector(0.0f, 0.0f, BoxSize));
		//填充索引
		IndexBuffer.Indices.Add(0);
		IndexBuffer.Indices.Add(1);
		IndexBuffer.Indices.Add(2);
		IndexBuffer.Indices.Add(0);
		IndexBuffer.Indices.Add(2);
		IndexBuffer.Indices.Add(3);
		IndexBuffer.Indices.Add(0);
		IndexBuffer.Indices.Add(3);
		IndexBuffer.Indices.Add(1);
		IndexBuffer.Indices.Add(3);
		IndexBuffer.Indices.Add(2);
		IndexBuffer.Indices.Add(1);
		//初始化
		VertexFactory.Init(&VertexBuffer);
		BeginInitResource(&IndexBuffer);
		BeginInitResource(&VertexBuffer);
		BeginInitResource(&VertexFactory);
	}

	virtual ~FTestCustomComponentSceneProxy()
	{
		VertexBuffer.ReleaseResource();
		IndexBuffer.ReleaseResource();
		VertexFactory.ReleaseResource();
	}


	virtual uint32 GetMemoryFootprint() const override
	{
		return (sizeof(*this) + GetAllocatedSize());
	}

	virtual void GetDynamicMeshElements(const TArray<const FSceneView*>& Views, const FSceneViewFamily& ViewFamily, uint32 VisibilityMap, class FMeshElementCollector& Collector) const override
	{
		FBox TestDynamicBox = FBox(FVector(-100.0f), FVector(100.0f));
		DrawWireBox(Collector.GetPDI(0), GetLocalToWorld(), TestDynamicBox, FLinearColor::Red, ESceneDepthPriorityGroup::SDPG_Foreground, 10.0f);
	}

	virtual FPrimitiveViewRelevance GetViewRelevance(const FSceneView* View) const override
	{
		FPrimitiveViewRelevance Result;
		Result.bDrawRelevance = true;
		Result.bDynamicRelevance = true;
		Result.bShadowRelevance = true;

		return Result;
	}

	// Inherited via FPrimitiveSceneProxy
	virtual SIZE_T GetTypeHash() const override
	{
		static SIZE_T UniquePointer;
		return reinterpret_cast<SIZE_T>(&UniquePointer);
	}

public:
	virtual void DrawStaticElements(FStaticPrimitiveDrawInterface* PDI) override
	{
		FMeshBatch Mesh;
		FMeshBatchElement& BatchElement = Mesh.Elements[0];
		BatchElement.IndexBuffer = &IndexBuffer;
		Mesh.bWireframe = false;
		Mesh.VertexFactory = &VertexFactory;
		Mesh.MaterialRenderProxy = UMaterial::GetDefaultMaterial(MD_Surface)->GetRenderProxy();

		FBoxSphereBounds PreSkinnedOutBounds;
		BatchElement.PrimitiveUniformBuffer = CreatePrimitiveUniformBufferImmediate(GetLocalToWorld(), GetBounds(), GetLocalBounds(), GetLocalBounds(), true, true);
		BatchElement.FirstIndex = 0;
		BatchElement.NumPrimitives = IndexBuffer.Indices.Num() / 3;
		BatchElement.MinVertexIndex = 0;
		BatchElement.MaxVertexIndex = VertexBuffer.Vertices.Num() - 1;
		Mesh.ReverseCulling = IsLocalToWorldDeterminantNegative();
		Mesh.Type = PT_TriangleList;
		Mesh.DepthPriorityGroup = SDPG_Foreground;
		Mesh.bCanApplyViewModeOverrides = false;
		Mesh.bDisableBackfaceCulling = false;
		PDI->DrawMesh(Mesh, 1.0f);
	}

public:
	FTestCustomComponentIndexBuffer IndexBuffer;
	FTestCustomComponentVertexBuffer VertexBuffer;
	FTestCustomComponentVertexFactory VertexFactory;
};

FPrimitiveSceneProxy* UTestCustomComponent::CreateSceneProxy()
{
	return new FTestCustomComponentSceneProxy(this);
}

FBoxSphereBounds UTestCustomComponent::CalcBounds(const FTransform& LocalToWorld) const
{
	return FBoxSphereBounds(FVector(0.0f), FVector(100.0f), 100.0f).TransformBy(LocalToWorld);
}
```

给一个 `Actor` 添加你刚刚写的 `TestComponent` 组件。你会发现一个红色的线框立方体与一个灰色的三棱锥。外侧的是动态的模型，每一帧都会通过 `GetDynamicMeshElements` 获取；内侧的是静态的模型，只会在初始化的时候获取一次。

<br>

### 11.4.6 静态网格物体渲染代理排序

静态网格物体元素列表如果直接遍历列表渲染，会反复设置状态，非常缓慢，如图11-6所示。如果按公用状态归并为链表，可以加快速度，如图11-7所示。

![](/res/img/post/06-ue-daxiangwuxing-03/11-4-2.png)

![](/res/img/post/06-ue-daxiangwuxing-03/11-4-3.png)

`DrawStaticElements` 函数只会在场景收集的时候被调用。这个收集过程不仅仅是将静态渲染代理填充到一个列表里面完事，顶点缓冲区而是将这些渲染代理填充到多个渲染列表中，这些渲染列表以渲染状态分割，以便尽可能公用渲染状态。

当一个新的静态元素被收集的时候，会调用当前静态网格物体绘制列表的索引缓冲区 `AddMesh` 函数，其会查找当前 `TStaticMeshDrawList` 的 `DrawingPolicySet`，查看是否有一致的 `Policy`，如果遇到匹配的，就会挂接在对应的列表下，否则就会新建一个列表，然后加入。

那么如何判断是否一致呢？答案在对应的 `Policy` 的 `Matches` 和 `Compare` 函数，每个类型都不一样，如果读者希望深入学习，可以亲自去看一下。

<br>
<br>

## 11.5 Shader

### 11.5.1 测试工程（新建一个插件）

由于 `Shader` 的载入是一个非常特殊的时机，因此，不能直接使用传统的工程文件新建流程。而是需要对你的游戏模块的加载时机进行设定。因此，请按照笔者说的方式来创建这个工程。

首先请创建一个标准的 C++ 工程。这时候你的工程结构实质上应该是：

![](/res/img/post/06-ue-daxiangwuxing-03/11-5-1.png)

这时候你的游戏工程主模块的加载顺序是 `“Default”`。这会导致我们的 `Shader` 无法加载。因此，我们必须要新建一个在 `PostConfigInit` 阶段加载的模块。

但是由于游戏模块的 `PostConfigInit` 模块是无效的（笔者测试的结果如此），所以我们必须把这个测试模块放在一个插件中。

打开项目 VS，启动编辑器，新建一个空白的插件：

![](/res/img/post/06-ue-daxiangwuxing-03/11-5-2.png)

![](/res/img/post/06-ue-daxiangwuxing-03/11-5-3.png)

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

打开项目可以看到已经引用该插件了：

![](/res/img/post/06-ue-daxiangwuxing-03/11-5-4.png)

可以打开 `ShaderTest.uplugin` 查看该插件包含的模块：

**这里需要将 "LoadingPhase": "Default" 修改成 "LoadingPhase": "PostConfigInit"**

```txt
{
	"FileVersion": 3,
	"Version": 1,
	"VersionName": "1.0",
	"FriendlyName": "ShaderTest",
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
			"Name": "ShaderTest",
			"Type": "Runtime",
			"LoadingPhase": "PostConfigInit"
		}
	]
}
```

> 后面的内容因为版本不太一样，再加上笔者目前功力尚浅，无法实践，可能会在以后的其他文章中再做分享了，还望勿怪

<br>
<br>

## 11.6 材质

在对材质系统进行介绍之前，笔者必须强调，本书立足于 “程序” 和 “架构” 的分析，因此对具体渲染方程的描述不会涉及过多。也就是说，只会介绍“代码是这样计算”，但是为何采用这样的算法、算法的优劣，如果要介绍的话就会是长长的推导论文。笔者并不是计算机图形学出身，不敢妄言这些公式优劣高低，因此很惭愧不能给大家剖析公式的优美之处。本书内容专注于虚幻引擎自身架构的设计，而不会出现大量的数学内容，可能对数学有些吃力的读者会感到轻松。

对于材质系统进行研究之前，我们首先需要整理一下材质系统本身是如何工作的。换句话说，在寻找问题的答案之前，首先需要明确问题是什么。**向上，材质系统提供了一套节点编辑系统用于编辑当前材质，同时我们也知道，这个节点编辑系统必然会编译成具体的 `Shader` 代码以供调用。向下，材质系统最终会通过传递当前材质的 `RenderProxy`，从而让网格的渲染代理能够知道如何渲染当前对象。**

因此，我们对材质系统的研究，也将会按照这个向上和向下的思路来完成。

当然，需要注意的是，不同于蓝图系统的高度可扩展性和可定制性，虚幻引擎的渲染系统，尤其是材质系统，实际上是不支持随意定制的。许多人会提出反驳，认为 `Custom` 节点可以输入自己的 `HLSL` 代码，相当于提供了无限的自由度。但是这种想法是错误的，因为 `Custom` 实际上只产生了一个节点——你没有办法干涉整个材质的渲染过程。甚至连材质的 `Shader Model` 类型都是被写死在虚幻引擎代码中的。

**也就是说，新增 `ShadingModel` 就必须改动源码。仅仅依靠插件无法达到增加新的着色模型的效果。**

<br>
<br>

### 11.6.1 概述

#### 编辑器

许多朋友对虚幻引擎的材质的最深印象就是那个节点编辑器。那么遵循从兴趣开始的原则，我们先从编辑器开始讲。如何找到是什么样的代码产生了编辑器呢？如果你并不清楚材质编辑器是一个什么样的 `Slate` 控件，你可以打开控件反射器，鼠标光标指向当前材质编辑器的窗口，就能看到控件类型。当然，如果看过前文关于蓝图的介绍，你会提出这应该是一个 `SGraphEditor`。于是我们搜索 `“SNew(SGraphEditor)”`，很快就会发现——产生材质编辑器的代码在 `MaterialEditor.cpp` 中：

```cpp
return SNew(SGraphEditor)
	.AdditionalCommands(GraphEditorCommands)
	.IsEditable(true)
	.TitleBar(TitleBarWidget)
	.Appearance(this, &FMaterialEditor::GetGraphAppearance)
	.GraphToEdit(Material->MaterialGraph)
	.GraphEvents(InEvents)
	.ShowGraphStateOverlay(false)
	.AssetEditorToolkit(this->AsShared());
```

材质编辑器本身就是一个 `SGraphEditor`。大部分内容都不大需要解释，导致 逻辑蓝图编辑器与 材质蓝图编辑器的最大区别来源于传递给 `GraphToEdit` 这个 `Slate` 变量的参数不同。`Material` 变量就是当前正在编辑的材质，只是传递给材质编辑器的材质蓝图不同于逻辑蓝图，导致蓝图的编辑、节点都完全不一样了。

<br>

#### 材质蓝图

材质蓝图（MaterialGraph）依然是一个蓝图，因此 `UMaterialGraph` 类继承自 `UEdGraph` 类。比较特殊的部分如下：

- `Material` 持有指向对应材质的引用。
- `UMaterialFunction` 当前对应的材质函数。如果这是一个普通材质，那么这个变量为空。
- `RootNode` 根节点。这个节点就是那个有一堆输入端口（BaseColor、粗糙度）的节点。如果是材质函数，这个变量为空。
- `MaterialInputs` 材质输入。是一个 `FMaterialInputInfo` 类型的数组。如果是材质函数，这个变量不会被设置。
- `AddExpression(UMaterialExpression*Expression)` 添加一个材质表达式到当前材质蓝图，返回新创建的蓝图节点指针。
- `LinkGraphNodesFromMaterial()` 根据材质信息链接蓝图。
- `LinkMaterialExpressionsFromGraph()` 根据蓝图信息链接材质。
- `IsInputActive(classUEdGraphPin*GraphPin)` 判断当前给定的蓝图输入是否可用。

关于这些函数的进一步分析，会留到下文介绍 `Material` 之后再进行讲解，这里希望读者对这个结构先有一些印象。

<br>

#### UMaterial

`UMaterial` 这个结构则相对来说更加复杂，如图所示。我们不妨向上一层，首先查看 `UMaterialInterface` 这个接口定义。这个接口抽象了材质与材质实例之间的共有特性。即使这样，`UMaterialInterface` 依然包含大量的接口定义和代码。**分析引擎的方法是抓大放小，并非每一个函数、每一个变量的原理我们都需要了解的一清二楚，而是要抓住重点。** 所以我们不再列出一大堆函数的定义。**对于继承自 `UMaterialInterface` 的类，最主要的两个功能是编译和提供着色器组合。** 前者被 `CompileProperty` 和 `CompilePropertyEx` 完成，后者是通过 `GetRenderProxy` 函数返回一个 `FMaterialRenderProxy` 结构实现。

![](/res/img/post/06-ue-daxiangwuxing-03/11-6-1.png)

### 11.6.2 材质相关C++类关系

现在摆在我们面前的是三个主要的类：`UMaterial`，`FMaterial`，`FMaterialRenderProxy`。首先我们必须理清这三个关系，如图所示。

![](/res/img/post/06-ue-daxiangwuxing-03/11-6-2.png)

按照前文叙述的判定规则：U 开头代表游戏线程，F 开头代表渲染线程。我们将 `UMaterialInstanceConstant` 这三个类分成了两族：用于逻辑线程使用的 `UMaterial`，用于渲染使用的 `FMaterial` 和 `FMaterialRenderProxy`。其对应用途如下：

- `UMaterial` 持有逻辑线程相关的材质信息，并作为逻辑线程的访问接口，管理其余两个渲染线程的类。
- `FMaterial` 负责管理渲染资源的类。这里的渲染资源主要就指 `ShaderMap`。其负责处理 `ShaderMap` 的编译等。
- `FMaterialRenderProxy` 渲染代理，负责处理渲染相关操作的类。

这三个类，加上由 `FMaterial` 负责管理的 `FMaterialShaderMap` 类（及其子类）共同完成了材质这个数据结构的表达。

<br>

### 11.6.3 编译

`当你点击材质编辑器的应用按钮或者保存按钮的时候，UMaterial` 的 `PostEditChangeProperty` 函数就被触发。于是对当前材质的节点树进行编译，生成着色器缓存 `CacheShaders`。顺便一提，这个编译对应的函数是 `BeginCompileShaderMap`，这个编译有可能失败，虚幻引擎只是 “开始” 编译，如果失败的话，还是会还原成原来的 `ShaderMap`。关于命名为何带有 `Begin`，在虚幻引擎中，大部分带有 `Begin` 的函数，其执行过程有一部分在另一个线程中，通常是在渲染线程中完成。如果你对这个内容比较陌生，可以再次回头看一下前文中关于并行与并发的描述。

而编译过程中，编译器根据依赖平台不同而不同，在笔者使用的 `Win64` 平台，是通过 `FHLSLMaterialTranslator` 完成的。其会根据 `UMaterial`，调用对应的 `CompilePropertyEx` 逐属性地进行编译，最终的结果是一段 `HLSL` 代码。你可以从 材质编辑器的窗口→HLSL 代码中看到最终编译的结果。

从节点图转化为 `HLSL` 的过程其实并不是非常的复杂。你可以在引擎 `Engine\Shaders` 文件夹中找到一个叫 `MaterialTemplate.usf` 的文件。这个就是编译的 “模板”。实际上转化过程就是把这个模板中的一些部分替换为具体表达式。甚至比 `PHP` 渲染静态页面的过程还要简单，其直接通过类似 `printf` 中的 `%s` 标记来完成替换。

已经产生的 `HLSL` 代码会被处理，得到 `ShaderMap` 结构。那么我们接下来需要分析的就是这个颇为独特的 `ShaderMap`。

<br>

### 11.6.4 ShaderMap产生

在前文对渲染过程的描述中，已经介绍了位于渲染线程的渲染代码是如何使用 `ShaderMap` 提供的数据，从中提取出真正需要的着色器并提交绘制请求。以上回答的是 “我是谁” 和 “我往哪里去” 的问题。接下来我们将会分析 “我从哪里来” 的问题，即材质如何编译、产生 `ShaderMap` 并缓存起来。采用倒过来的介绍顺序的原因是，当读者首先阅读 “ShaderMap产生” 的介绍时，会对一大堆陌生的东西产生困惑：为什么要产生这个东西？这个东西有什么用？为何要产生多个版本的顶点和像素渲染器，而不是同一个版本？所以笔者采用倒叙方式进行介绍。

前文已述，当 `HLSL` 代码被生成后，就需要进入真正的着色器编译阶段。首先需要创建一个 `ShaderMap` 实例。为了方便叙述且与源码一致，我们称该实例为 `NewShaderMap`。

材质节点图生成的 `HLSL` 代码只是一批函数，并不具备完整的着色器信息。这些代码会被嵌入到真正的着色器编译环境中，然后重新编译为最终的 `ShaderMap` 中的每一个着色器。

这个着色器编译环境对应的类是 `FShaderCompilerEnvironment`。该类通过 `MaterialTranslator::GetMaterialEnvironment` 函数初始化实例，主要完成以下两个步骤：

1. 根据当前材质的各种属性，设置各种着色器宏定义。从而控制编译过程中的各种宏开关是否启用。比如如果当前材质的变量 `bUsesLightMapUVs` 被设置为真，则设置宏定义中的 `LIGHTMAP_UV_ACCESS` 为真。
2. 根据 `FHLSLMaterialTranslator` 在解析过程中得出的当前参数集合，添加参数定义到环境中。

初始化完成后，开始进行实际的编译工作：

1. 获取刚刚生成的 `HLSL` 代码（就是 `FString` 格式的字符串）。
2. 将刚刚生成的代码作为 `“Material.usf”` 文件，添加到编译环境中。
3. 调用 `NewShaderMap` 的 `Compile` 函数：
	> **a. 调用 `FMaterial::SetupMaterialEnvironment` 函数，设置当前的编译环境。笔者尚不明白为何编译环境需要分为两个步骤进行设置。**

	> - 通过 `FShaderUniformBufferParameter::ModifyCompilationEnvironmen`，把通用缓存的参数结构定义添加到编译环境中。这个定义虽然看上去是个文件（ `“UniformBuffers变量名.usf”` ），但其实是通过 `CreateUniformBufferShaderDeclaration` 根据你用到的参数生成的，比如你用的贴图，就会转化为Texture2D。
	> - 根据曲面细分设置一系列的宏。
	> - 根据混合模式（Opaque、Masked、Translucent、Additive、Modulate）设置宏。
	> - 根据接收贴花的类型设置宏定义。
	> - 根据 Unlit、DefaultLit 等光照模式设置宏定义。

	> **b. 获取所有的顶点工厂类型（`FVertexFactoryType`），对每个顶点工厂类型执行以下操作：**

	> - 创建一个 `FMeshMaterialShaderMap*` 指针 `MeshShaderMap`。
	> - 查看当前顶点工厂类型对应的 `FMeshMaterialShaderMap` 是不是已经存在，已经存在就赋值给该指针，否则创建一个新的。
	> - 对当前 `MeshShaderMap`，调用 `BeginCompile`：
	> 	- 遍历包含所有的 `FShaderType` 的列表，获取 `FMeshMaterialShaderType* ShaderType`。
	>	- 通过 `HasShaderType` 函数检查是否已经编译，如果没有编译，将一个编译任务加入到编译工作列表 `SharedShaderJobs` 中。根据当前的顶点工厂类型，添加工厂定义到编译环境中。调用 `GlobalBeginCompileShader` 创建编译任务。设置 `FShaderCompilerInput` 实例 `Input`，包括编译目标、着色器格式、源文件名、入口名等。然后设置一大堆的编译参数，这里不再赘述，有兴趣的读者可以直接查看该函数。
	> - 使用异步着色器编译流水线执行这些着色器编译任务。

<br>
<br>

The End.
