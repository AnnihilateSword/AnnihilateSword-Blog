---
title: 【UE5】网络同步原理探究 (Actor同步流程)
tags: [UE5, UE5网络同步]
date: 2025-03-21
updated: 2025-03-21
cover: /res/img/post/12_ue5_5_net_sync_explore_actor_replicate/cover.jpg
top_img: /res/img/site/background.png
---

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/cover.jpg)

>交界地第一糕手。

<br>

# 前言

_**注意：本篇文章使用的源码版本是 UE5.5.3**_

>细节比较多，后续会继续更新补充本篇文章。
>本文参考了许多技术文章 (见参考)，有些参考由于各种原因不方便列出
>下面是结合个人实操的 UE5 中 Actor 同步的探究记录
>当然这只是 Actor 同步，后面还有 ReplicationGraph 以及 UE5 新出的 Iris

下面先给出一些总结，再开始探究细节，好有个思路又或方便回忆。

<br>
<br>

# Actor 同步流程【总结】

UE 是被动式同步方式，这也导致了网络同步时的性能消耗：

- **GatherActor 阶段**：在收集 Actor 时，收集所有相关的 Actor 列表，遍历 Actor 列表去一个个处理，处理之前是不知道哪些 Actor 在这一帧内同步属性被修改了，需要后续更多的逻辑去判断是否这个 Actor 在这一帧需要同步
- **属性对比阶段**：针对一个 Actor 或 Component，引擎同样也是不能提前知道这个 Actor 有哪些属性改变，同样得遍历所有属性去做对比才知道是否发生了改变，发生了改变才需要去同步。当然这里说的是 PushModel 关闭的情况，即使提供了 PushModel 功能，但是引擎并没有将属性全部改成支持 PushModel 的情况
- **同步 Component 阶段**：一个 Actor 的 Components，引擎同样不知道哪些 Component 在这一帧发生了属性变化，得遍历所有的可同步 Component，每一个 Component 都得走一遍属性对比过程
- **CheckCloseChannel 阶段**：在同步的最后，引擎得遍历此连接上的所有 ActorChannel，去根据 CloseFrameNumber 来决定当前帧这个 Actor 是否需要 CloseChannel，从而在客户端去销毁这个 Actor。同样引擎是不知道当前帧哪些 Actor 因为失去了相关性而去 CloseChannel，它得遍历所有的 ActorInfo 去检查

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/0-1.png)

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/0-2.png)

>动态生成的 Actor 才会因为超出网络同步剔除距离被干掉。

<br>

✨下面分享一次试验🔥：

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/0-3.png)

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/0-4.png)

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/0-5.gif)

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/0-6.png)

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/0-7.png)

<br>
<br>

上述四点充分体现了 UE 的被动同步机制，这对同步性能产生了显著影响。当然，引擎自身也引入了一些特性来缓解这些问题：

- **ReplicationGraph**：减少了 GatherActor 阶段需要处理的 Actor 数量。
- **NetDormancy**：进一步减少了 GatherActor 阶段处理的 Actor 数量，同时也降低了 CheckCloseChannel 阶段的遍历次数。
- **PushModel**：减少了属性对比的执行次数，降低了性能消耗。
- **Iris**：后面有空再花时间探究下

<br>

然而，仍有一些问题尚未解决，例如如何减少遍历 Component 的消耗，以及对于未发生变化的 Actor 是否完全无需进行属性对比等。在同步优化时，可以通过引入上述引擎特性来大幅降低同步消耗，甚至可以进一步修改引擎实现，以减轻被动式同步机制带来的问题。

>_**抛开 UE 不谈，同步机制应采用主动触发式的方式。在底层处理同步时，系统应明确知晓哪些 Actor 需要同步，哪些属性需要同步，以及哪些 Actor 需要关闭 Channel，从而避免大量不必要的消耗。**_希望虚幻引擎在未来的发展中能够朝着这个方向迈进。当然，各位开发者也可以发挥自己的智慧，将 UE 的同步机制改造为主动触发式。


<br>
<br>
<br>

# 一、服务器网络模块初始化流程

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/1-1.png)

可以看到在初始化监听之前还有 FEngineLoop::Init() 和 UGameInstance 初始化相关的调用，下面我们从 UWorld::Listen 开始梳理流程。

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/1-2.png)

然后值得注意的是 UNetDriver::SetWorld 中注册到 TickDispatch , TickFlush 之类的事件

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/1-3.png)

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/1-4.png)

<br>
<br>

# 二、客户端网络模块初始化流程

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/2-1.png)

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/2-2.png)

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/2-3.png)

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/2-4.png)

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/2-5.png)

这里是比较关键的，客户端在连接前会发送消息给服务器：

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/2-6.png)

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/2-7.png)

>我这里用的测试工程是 Lyra 5.5 引擎版本，会创建下面 4个 Channel

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/2-8.png)

<br>
<br>

# 三、服务器与客户端建立连接流程

_**整个握手通过 StatelessConnectHandlerComponent 完成（server & client端）**_

握手流程这里不展开了，后续有空再补充，或者单独用一篇文章进行说明。

有兴趣可以看看这篇：✨[UE4 UDP握手详解](https://github.com/qqwx1986/ue4_doc/blob/master/UDP%E6%8F%A1%E6%89%8B%E8%BF%87%E7%A8%8B%E8%AF%A6%E8%A7%A3.md)

<br>
<br>

# 四、Actor 同步流程

## 主要流程

每一帧中做同步 Actor 的流程：

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-1.png)

如果开启了 _**ReplicationGraph**_ 则 ReplicateActor 的过程交由 ReplicationGraph 托管，因此结果就是：在每一帧的末尾通过 `TickFlushEvent` 事件通知到 NetDriver 中，再交由 ReplicationGraph 接管。到 `UReplicationGraph::ServerReplicateActors` 函数中处理，下面接着讲这个函数

> _**引擎默认是禁用 ReplicationGraph 的**_，开启可以参考官方文档：✨[Replication Graph (UE官方文档)](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/replication-graph-in-unreal-engine)

<br>

这里值得注意的是，如果是走改配置方案 (DefaultEngine.ini) 填的是模块名，而不是项目名：

```txt
ReplicationDriverClassName="/Script/ModuleName.ClassName"
```

<br>
<br>

_**ServerReplicateActors**_

主要工作：

1. **遍历所有 Connection**：这里会获取引擎每帧可以同步的最大 Connection 数量，超过这个限制的忽略。然后对每个 Connection 几乎都会执行下面的操作 (这里也有处理 移动的同步 和 回放相关的操作)
2. **GatherActors**：给每个连接收集要同步的 Actors
3. **ReplicateActor**：遍历第一步中的每个 Actor，做 `ReplicateSingleActor` 的逻辑，也就是具体同步每一个 Actor，最后调用到 UActorChannel 中处理。
4. **对丢失相关性的 Actor CloseChannel**：在最后会遍历所有的 ActorChannel，根据 CloseFrameNumber 来决定是否要 Close ActorChannel，从而让客户端销毁此 Actor

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-4.png)

重点记住上面的四个阶段能让整体的同步流程更加清晰。

<br>

然后这里可以提一下 TickDispatch 在 TickFlush 前面执行：

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-5.png)

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-6.png)

<br>

## (一) UNetDriver::ServerReplicateActors

>ServerReplicateActors: this is main function to replicate actors to client connections. It can be "outsourced" to a Replication Driver.
>ServerReplicateActors：这是将 actor 复制到客户端连接的主要功能。它可以 “外包” 给复制驱动程序。

<br>

其中 GatherActor 阶段的代码如下 ：

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-7.png)

这里会遍历所有的 ReplicationGraph Node 去 GatherActor，将得到的 Actors 列表写入到 Parameters 参数中，后续就针对 Parameters 中的 Actors 列表进行同步处理，针对每个符合条件的 Actor 调用 UReplicationGraph::ReplicateSingleActor 函数处理

## (二) UReplicationGraph::ReplicateActorListsForConnections_Default

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-8.png)

>ReplicateSingleActor 是在这里面调用的，来看看这里做了些什么：

<br>

这里面主要做的工作是对所有连接 Actors 进行优先性排序。

- 如果 Actor 是休眠的 (dormant) ，则跳过
- 如果 Actor 没有准备好在该帧复制，则跳过
- 处理调试的输出记录
- 距离缩放 (Distance Scaling)
  - 总是计算距离，即使对于 AlwaysRelevant 角色，因为优先级缩放需要它
  - 如果没有 Actor 在这个 Actor 附近，跳过它
  - 更新超时帧数
- Starvation Scaling (饥饿缩放)
- Pending dormancy scaling (处理休眠缩放)
- Game code priority (这里判断了 GlobalData.ForceNetUpdateFrame 的值)
- 始终优先考虑连接的所有者和视图目标，因为它们是客户端最重要的参与者。这些 Actor 的优先级会被提高。
- 对所有 Actor 进行优先级排序
- _**最后会调用 UReplicationGraph::ReplicateActorsForConnection**_


> 这里比较关键的变量就是 _**AccumulatedPriority**_ ，由它控制排序的权重，其中 **AccumulatedPriority 的值越小，优先级越高。**

<br>

_**接下来就调用到了 ReplicateSingleActor**_

<br>

## (三) UReplicationGraph::ReplicateSingleActor

上文收集到了 Actor 后，要对每个 Actor 进行 Replicate 操作，就进入到 _**UReplicationGraph::ReplicateSingleActor**_ 的流程。

>(再次强调，本文是建立在使用 ReplicationGraph 的基础上，这里的 ReplicateActor 的流程也会和不使用 ReplicationGraph 不一样) ，没启用 ReplicationGraph 时，这部分逻辑在 UNetDriver 中处理

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-9.png)

下面将根据上面流程图梳理一下

### 1.更新 ActorInfo  数据

```cpp
ActorInfo.LastRepFrameNum = FrameNum;
ActorInfo.NextReplicationFrameNum = FrameNum + ActorInfo.ReplicationPeriodFrame;
```

这里更新了两个数值：上一次同步的帧号和下一次要同步的帧号。一个 Actor 的同步频率的调节就是生效于此，引擎会根据 `NextReplicationFrameNum` 帧号在前面提到的 `ReplicateActorListsForConnections_Default` 函数中和当前帧号对比来决定这个 Actor 是否需要在这一帧被同步。

<br>

我们在 Actor 中配置的这个值 `NetUpdateFrequency` 便是生效于此，这个值换算成了 `ReplicationPeriodFrame` ，从而影响了 Actor 隔几帧才同步一次。

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-10.png)

<br>

**FConnectionReplicationActorInfo**

上述说的 ActorInfo 就是这个类型，简单介绍一下这个类型。（注意：这是 ReplicationGraph 才有的数据结构，不使用 ReplicationGraph 则不使用这一套逻辑）

在每个 Connection 上，引擎都会给要在这个连接同步的 Actor 创建所属于这个 Connection 的 ActorInfo 。也就是说一个 Actor 在多个连接上都存在一份这个连接上的 ActorInfo，这个和 UActorChannel 是类似的，也是一个 Actor 会有多个 UActorChannel 对象，它们属于不同的连接上的数据。

> 这里提一下 _**FastPath**_ 相关的变量：
> FastPath 是一种优化机制，用于减少网络复制的开销。它通过跳过某些检查或处理步骤，直接复制数据，从而提升性能。
> 使用场景：
> - 高频更新：适用于需要频繁更新的 Actor，如玩家角色或移动物体。
> - 低延迟需求：在需要低延迟的场景中，FastPath 能显著减少复制延迟。

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-11.png)

- **Channel**：在这个连接上这个 Actor 对应的 UActorChannel 对象
- **CullDistance/CullDistanceSquared**: 这个 Actor 的网络裁剪距离，我们在 Actor 上配置的 `NetCullDistanceSquared` 就会生效于此
- **NextReplicationFrameNum**: 上面提到了，这个 Actor 下一次应该被同步的帧号
- **LastRepFrameNum**：上一次被同步的帧号
- **ReplicationPeriodFrame**：同步间隔，这里是帧数，也就是隔多少帧才同步一次
- **ActorChannelCloseFrameNum**：这个 Actor 在这个连接上需要被 Close ActorChannel 的帧号，会在对这个 Actor 做一次 Replicate 时更改，引擎会通过这个帧号和当前帧号对比，如果小于当前帧号则 CloseChannel，给客户端发 CloseChannel 的包，客户端就会在本地 Destroy 这个 Actor。这个是用于非 Dormant Actor 的。引擎会根据 ReplicationPeriodFrame 和一些 TimeOut 的数据去更新此值。此值的使用会在 ReplicationGraph 章节中讲解
- **bDormantOnConnection**：Actor 在这个连接上是否进入到了 Dormant 状态。（引擎的 NetDormancy 机制会在后面讲解）


<br>

### 2.CallPreReplication

在同步之前准备一些数据或者控制哪些属性是否要同步

<br>

**2.1 AActor::PreReplication**

- 这里会调用 _**GatherCurrentMovement**_，会收集计算Actor的移动数据，填充和移动相关的几个同步属性
- 这里可以调用 `DOREPLIFETIME_ACTIVE_OVERRIDE_FAST` 宏来控制某个属性是否在这一帧要被同步，例如 DOREPLIFETIME_ACTIVE_OVERRIDE_FAST(AActor, ReplicatedMovement, IsReplicatingMovement());，最后一个参数为 true 则表示需要同步

**2.2 调用挂在该 Actor 上的每个 Component 的 PreReplication 函数**

<br>
<br>

### 3.首次同步 Actor，创建 UActorChannel

**在同步单个 Actor 时，主要是依赖 UActorChannel 来进行，对具体 Actor 属性的收集、RPC 的处理、收到包后解析给具体的 Actor 等等都是在 UActorChannel 中处理。** 是网络同步中的重点内容，是衔接 Connection 和上层业务的桥梁，收数据时，Connection 将收到的数据交给 UActorChannel，UActorChannel 再解数据交给具体业务层实现的逻辑；发数据时，由 UActorChannel 来收集具体业务层定义的同步数据，再交由 Connection 层去发送数据。

引擎会给一个 Actor 在多个 Connection (当然是这个 Actor 在这些 Connection 上是要被同步的) 上分别创建一个 UActorChannel 对象。

判断 ActorInfo.Channel 是否为空来决定是否要创建 ActorChannel

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-12.png)

<br>

**3.1  UNetConnection::CreateChannelByName 函数创建Channel**

如果 ActorInfo.Channel == nullptr ，则创建 UActorChannel

创建过程如下：

1. **CreateChannelByName**
2. 如果没有 ChannelIndex 被指定，去获取一个 **UNetConnection::GetFreeChannelIndex**
3. **UNetDriver::GetOrCreateChannelByName 从 Channel Pool 中获取**
4. **UChannel::Init 初始化**
5. Channels[ChIndex] = Channel; OpenChannels.Add(Channel); **在 NetConnection 中将 Channel 管理起来**

>如果引入了 NetDormancy，就不是第一次同步 Actor 才创建 Channel 了，这时 ActorChannel 指针同样是为空，也是会走这个流程，NetDormancy 将会在后面讲解

<br>

下面来讲解一下流程的关键步骤：

- 获取 ChannelIndex，GetFreeChannelIndex：每个 ActorChannel 都会分配一个 Index，这个 Index 是 Channels 数组中的下标。UNetConnection 存储着一个 TArray<UChannel*> Channels; 数组，在 UNetConnection 构造的时候初始化了这个数组的大小

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-13.png)

- 通过 Console 命令 `net.MaxChannelSize` 可以改写 MaxChannelSize 的值，默认值是 32767

> DefaultMaxChannelSize(32767)

<br>

所以，这个引擎默认创建 UNetConnection 时，会创建 Channels 数组，其大小是 32767，每创建一个 Channel 时就从这里获取一个值为空的 Index，并将创建的 UActorChannel 指针放到这个数组中 Index 的位置存储起来

- **从 NetDriver 的 ActorChannel Pool 获取对象**：NetDriver 上有个 Pool -- `TArray<UChannel*> ActorChannelPool;`，在创建和 Close ActorChannel 时都从这里取或放
- **最后将创建的 ActorChannel 在 NetConnection 上管理起来**：`Channels[ChIndex] = Channel; OpenChannels.Add(Channel);` 这里管理起来方便同步的其他逻辑使用

<br>
<br>

**3.2 UActorChannel::SetChannelActor**

这里将 Actor 和 ActorChannel 关联起来，并创建了 _**FObjectReplicator**_

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-14.png)

```cpp
// Add to map.
Connection->AddActorChannel(Actor, this);
```

这段代码将 ActorChannel 和 Actor 的对应关系在 UNetConnection 和 UReplicationGraphNetConnection 中对应起来，存成一个 Actor--ActorChannel 的 Map

<br>

**UActorChannel::FindOrCreateReplicator**: 在 UActorChannel 中创建 FObjectReplicator，用于后续的属性对比，同步数据处理过程。

>值得注意的是这里只是给 Actor 这个 UObject 创建FObjectReplicator，Actor 可同步的 Component 的 FObjectReplicator 会在后续的流程中创建

<br>

### 4.UActorChannel::StartBecomingDormant

这里对要休眠的 Actor 进行休眠处理。

<br>
<br>

## (四) UActorChannel::ReplicateActor

一个可同步的 Actor 在需要同步的 Connection 上都会创建对应的 UActorChannel，涉及到这个 Actor 的属性同步，RPC 最终都是到 UActorChannel 进行处理。例如解包确定是哪个属性被同步了、如何去触发 OnRep 回调、收到了 RPC 在这里解包去调用哪个 RPC 函数等等。

下面来看看 ReplicateActor 的大致过程：

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-15.png)

**1.创建 FOutBunch**

引擎是通过 Bunch 包来进行数据传输，每个 Actor 的同步数据都会装填到一个 Bunch 包中，这个 FOutBunch 包贯穿了 Actor 同步的整个过程，最终是将 Bunch 进行包装进行网络数据传输。发出去的是 FOutBunch，接收数据是 FInBunch

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-16.png)

- **Next**：用链表将发送的 reliable bunch 包串联起来，后续在收到 ack，nak 的时候会进行处理
- **ChIndex**：channel index，客户端在收包时创建 UActorChannel 时也会用这个 Index，使用相同的 Index
- **PacketId**：网络通信的数据最终的封装成了 Packet，就会有一个 PacketId，此值表明此 Bunch 是属于哪一个 Packet
- **ChSequence**: bunch 包的序号，自增值
- **ReceivedAck**: 是否是 ack
- **bOpen**：是否是 OpenBunch 包
- **bClose**：是否是 CloseBunch 包
- **bReliable**：是否是 Reliable 的包，OpenBunch 和 CloseBunch 这里值为 true，_**RPC 这里也是 true，引擎会保证 reliable=true 的包会被重发。但是属性同步的 Bunch 包 bReliable=false，因此是不会重发属性同步的 bunch 包，但是不能说属性同步是不可靠的，因为引擎会不断去保证最终的属性是同步的，因此属性同步到客户端是不保证及时也不保证顺序。**_
- **bPartial**：当 Bunch 包超过一个值后会被分成多个发送
- **CloseReason**：通道关闭的原因！！
- ![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-17.png)
  - **Destroy**：表示是 Actor 销毁后给客户端发的 CloseBunch，客户端会销毁此 Actor
  - **Dormancy**：表示进入 dormant 状态，给客户端发的 CloseBunch 包，客户端不会销毁此 Actor
  - **LevelUnloaded**：streaming level 被卸载了
  - **Relevancy**：相关性丢失，例如 NetViewer 超出了同步范围，客户端收到包后会销毁 Actor

<br>

> 这里给大家补充下 ack 和 nak 的含义：
> - ACK 是确认消息，表示接收方已成功收到数据包。
> - NAK 是否认消息，表示接收方未收到或发现数据包有误。
> - 这些机制在 UE5 的网络同步系统中至关重要，确保多玩家游戏中的一致性和可靠性。

不过实际传输的数据在其父类中，使用字节数组传输二进制数据，FOutBunch 的继承关系如下：

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-18.png)

在构造 FOutBunch 结构的时候，属性同步数据就写入到父类的 FBitWriter 的字节数组 Buffer 中了。

这里用到了 FArchive 在网络收发包的时候进行序列化。

>对于收包，引擎会将其封装成 _**FInBunch**_ 类型进行处理。

<br>

**2.创建 FReplicationFlags**

此结构封装了几个标记位，用于处理此次 Replicate 的 Actor 相关的数据，和我们在定义网络同步属性的 Condition 息息相关。

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-19.png)

我们在 `GetLifetimeReplicatedProps` 函数中定义属性的 Cond 类型 `ELifetimeCondition` 枚举对应着 `FReplicationFlags`

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-20.png)

ELifetimeCondition 中各个的含义就可以对应着 FReplicationFlag 中各个值是怎么赋值的，在费解之时可以看引擎代码中对 FReplicationFlags 的处理。

>另外 Condition 和 ReplicationFlags 的对应关系在函数 _**UE::Net::BuildConditionMapFromRepFlags**_ 中可以看到。

<br>

**3.如果要进行初始化，发送初始数据**

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-21.png)

- **UPackageMapClient::SerializeNewActor**

> 序列化 New Actor 的标准方法。
> * 对于静态 Actor，这将只是一个调用 SerializeObject，因为他们可以通过他们的路径名来引用。
> * 对于动态 Actor，首先 Actor 的引用被序列化，但不会在客户端解析，因为客户端还没有生成 Actor。
> * 演员原型随后与起始位置、旋转和速度一起序列化。
> * _**在读取这些信息后，客户端在 NetDriver 的世界中生成这个 Actor，并在函数的顶部为它分配它所读取的 NetGUID。**_
> <br>
> * 如果生成了一个新的 Actor，则返回 true。false 表示为 NetGUID 找到了一个现有的 Actor。

<br>

当第一次同步一个 Actor 的时候，需要在 Bunch 包中序列化一些 Actor 相关的基础数据 (比如，Transform, Location, Rotation, Scale 等等) ，能让客户端收到包后能找到相关的资源，然后去 SpawnActor：

<br>

### 4.3.1 创建 NetGUID

NetGUID 是客户端和服务器识别一个 UObject 的途径，客户端和服务器同一个 Object 就通过设置他们为同一个 NetGUID 来相互联系。UE5 可以同步指针属性，而客户端指针和服务器指针的值不可能相同，而他们指向的对象却是我们希望的，那就是在同步指针对象的时候，及时序列化的 NetGUID ，从而在客户端能通过 NetGUID 去找到客户端的对象，这样就能在客户端生成一个指针值指向此对象从而和服务器关联起来。

在同步一个 Actor 时，NetGUID 是必须要序列化的，会将分配的 NetGUID 序列化到 Bunch 包中。

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-22.png)

<br>
<br>

来看看 _**FNetGUIDCache::GetOrAssignNetGUID**_

首先会检查是否已经分配过 NetGUID 了，如果没有，则进行分配 (Assign NetGUID)，如果有，就返回已经分配的 NetGUID

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-23.png)

<br>

引擎将对象分为 Static 和 Dynamic，Static 属于静态物体，例如场景中摆放的 Actor，而 Dynamic 是 Runtime Spawn 出来的 Actor。

- 引擎通过 FNetGUIDCache 对象在管理 NetGUID 以及和 UObject 的联系，这个对象是 UPackageMapClient 的成员：`TSharedPtr< FNetGUIDCache > GuidCache;`，而 UPackageMapClient 又由 Connection 所持有

<br>

- GuidCache 通过 NetGUIDLookup 缓存着 GUID 和 UObject 的对应关系。如果分配过则直接返回。同样，在客户端收到一个 Actor 的 OpenBunch 包的时候，同样可以通过 GuidCache 判断，这个 Actor 是否在客户端已经存在，如果存在就可以将 UActorChannel 和此 Actor 对应起来，这样就不会存在两份同一个 Actor 的两份对象了。

<br>

- 分配 NetGUID：_**可见 NetGUID 是自增的**_
  - 在 UPackageMapClient 中有成员 `int32 UniqueNetIDs[2];`，记录了 Static 和 Dynamic 两种类型的最大的 GUID，因此数组的大小为 2
  - Static Actor 的 NetGUID 是奇数的；Dynamic Actor 的 NetGUID 是偶数的。

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-24.png)

<br>

### 4.3.2 序列化 Object

序列化 Object 的时候，会先序列化 Actor 自身，再序列化 Archtype 数据，再序列化  Level 数据。最后在同步这个 Actor 的时候会把当前的 transform，速度等都序列化。

> 值得一提的是，为了让客户端能去 Spawn 这个 Actor，需要将这个 Actor 的 Path Name 序列化到 Bunch 包中，这样客户端就能找到此 Actor 对应的资源去 Spawn Actor。

<br>

_**因此这里是存在一个优化点的，因为序列化 PathName 会很大，长串的字符串，比较占用带宽，因此可以通过给这些 PathName 分配 ID，将序列化 PathName 换成序列化 ID，这样极大减少流量。**_

<br>
<br>

### 4.4 与 ReplicateProperties 相关的类【了解】

>/** Replicates properties to the Bunch. Returns true if it wrote anything */
>这里最终会调用 _**FObjectReplicator::ReplicateProperties_r**_


第一次同步 Actor 序列化相关的数据后，就要开始进行属性数据的同步了。如果不是第一次同步某个 Actor，则直接到此步骤。 这一过程涉及到几个类：FObjectReplicator，FRepLayout 等，先介绍一下这几个类，代码的执行流程在介绍完这几个类后再讲解

<br>

#### (4.4.1) FObjectReplicator

ReplicateProperties 是在 FObjectReplicator 中进行的，每一个同步的 UObject，都对应着一个 FObjectReplicator 对象，这个对象负责了属性的对比，同步属性的提取，RepState的维护，丢包的处理，属性同步可靠的处理等等。_**属性同步的逻辑就是在 FObjectReplicator 中进行的，是一个非常关键、重要的角色类。**_

>由于 FObjectReplicator 和同步的状态相关。
>因此，每个同步的 UObject 至少有一个 FObjectReplicator。
>如果多个连接需要同步同一个 UObject，每个连接都会有自己的 FObjectReplicator。

下面列举了 FObjectReplicator 的几个重要的属性和方法：

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-25.png)

属性：

- **ObjectNetGUID**：这个 FObjectReplicator 对应的 UObject 的 NetGUID（用于将客户端和服务器的 UObject 对象通过这个 NetGUID 联系起来）
- **ChangelistMgr**: 这个管理着这个 UObject 的属性改变相关的数据，每个 UObject 对应一个 Mgr 对象，一个 UObject 对象在多个连接上共享一个 ChangeListMgr 对象
- **RepLayout**: 这个管理着 UObject 的内存布局等数据，一个类型对应着一个 FRepLayout 对象。注意不是一个实例对应一个 FRepLayout 对象，而是一个类型。
- **RepState**：同步过程中的一些状态数据。其有成员 ReceivingRepState 和 SendingRepState 分别处理收包和发包的一些状态数据
- **Connection**: 这个 FObjectReplicator 对应的连接
- **OwningChannel**：这个 UObject 所属的 Actor 对应的 ActorChannel，在这个连接上的。
- **RemoteFunctions**：_**unreliable multicast 的 RPC 调用数据会序列化到这里。可以看到 unreliabnle multicast 的 RPC 是和属性同步过程一起处理的。但是 reliable RPC 是另一个逻辑。**_

函数：

- **ReplicateCustomDeltaProperties**：同步一些 Custom 的属性，例如 _**FastArray**_
- **ReplicateProperties**：同步非 Custom 的属性
- **ReceivedBunch**：收包的处理
- **ReceivedNak**：收到了丢包，会根据丢包的 PackageID 在 FSendingRepState 中的 ChangeHistory 做上标记，这样下一次做属性同步的时候就能再次将丢包中包含的属性数据再次填入 bunch 包中发送
- **CallRepNotifies**：会调用属性的 OnRep 回调函数

<br>

FObjectReplicator 是由 UNetConnection 创建的：

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-26.png)

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-27.png)

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-28.png)

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-29.png)

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-30.png)

最后创建好的 FObjectReplicator 会被 NetDriver 中的 `TSet< FObjectReplicator* > AllOwnedReplicators;` 存起来

创建的 FObjectReplicator 会在 UActorChannel 上管理起来，UObject 指针作为 key的Map：

```cpp
TMap< UObject*, TSharedRef< FObjectReplicator > > ReplicationMap;
```

但是对于 Actor 来讲，给 Actor 创建的 FObjectReplicator 对象会直接被 UActorChannel 的成员属性 ActorReplicator 引用着，这个 Actor 的其他 UActorComponent 的 FObjectReplicator 就存在这个 ReplicationMap 中管理起来。

可以看到在创建 FObjectReplicator 的过程中，连续创建出了多个关键的类型：FReplayout，FRepChangedPropertyTracker，FRepState， FSendingRepState, FReceivingRepState 等，后续会针对这几个类型讲解。

创建完 FObjectReplicator 后，立马调用了 _**FObjectReplicator::StartReplicating**_ 函数，在这个函数中创建了一个重要的对象：_**FReplicationChangelistMgr**_，这个对象主要做属性对比的功能。

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-31.png)

<br>

#### (4.4.2) FReplicationChangelistMgr

每个 UObject 实例都对应一个 FReplicationChangelistMgr 对象，会存储在 UNetDriver 的 `ReplicationChangeListMap` 中，并且在这个 UObject 对象对应的 FObjectReplicator 中引用这个这个 UObject 对象的 FReplicationChangelistMgr 对象。

_**FReplicationChangelistMgr 对象是做属性对比的关键类**_，其中存储着每个同步属性的 ShadowData，也就是说每个同步属性上一次同步的数据，这样就能对比当前帧的属性是否较上次同步是否发生了变化，只有发生了变化才需要再次同步。

FReplicationChangelistMgr 主要是依靠其成员属性 `FRepChangelistState RepChangelistState` 来存储属性变化的 list。

>_**值得注意的是，每个 UObject 的 FReplicationChangelistMgr 对象是被所有连接共享的**_，当这个 UObject 要被同步给所有连接的时候，这个 UObject 对象的 change list 相关数据其实只需要一份，因为这个 UObject 对象只有一个。因此在多个连接要同步这个 UObject 对象的时候只需要一份这个 ChangelistMgr 对象即可。


创建 Mgr 对象时，会去初始化 ShadowBuffer，会去从 UObject 的 CDO 数据中去获取数据来初始化 ShadowBuffer。

> 这里有个 UE 同步的坑：某些情况下可能导致属性没同步给客户端。因为初始化 ShadowBuffer 是用的 CDO 数据，如果是蓝图类的话，注意这里的 CDO 数据就是蓝图上配置过的最新的值，假设有个 Actor，有个同步属性 P，在蓝图类上这个值被改成了 x，我们把这个 Actor 拖入场景中，这时已经实例化了，这个时候，我们将场景中的 Actor 属性 P 改成了 y，注意这个时候 CDO 的值仍然是蓝图类的数据 P 的 CDO 的值是 x。当在 DS 上同步这个已经拖入场景中的 Actor 时，游戏逻辑将属性 P 又改成了 x，在属性对比的时候发现，当前 P 的值是 x，和 ShadowBuffer 中的值一样也是 x，则 DS 就不会同步这个属性 P 了。但是，由于是在场景中的 Actor，客户端也会 Spawn 这个 Actor，跟随场景一起 Spawn，可是，客户端这个时候 Spawn 出来，属性 P 的值是 y。这个时候就产生了 bug，客户端并没有得到服务器修改后的值 x

_**FReplicationChangelistMgr 在 NetDriver 中进行了缓存**_

<br>

#### (4.4.3) FRepLayout

下面引用引擎注释：

>该类保存给定类型（ UClass、UStruct 或 UFunction ）的所有复制属性。helper 函数用于读取、写入和比较属性状态。
>**对于给定的类型只有一个 FRepLayout，这意味着该类型的所有实例共享 FRepLayout 状态。而不同实例就靠不同的 RepState 来记录着不同实例当前同步相关的数据。**

RepLayout 中的所有属性都表示为布局命令 (Layout Commands)。

命令分为 2 个主要类型：父命令（见 `FRepParentCmd`）和子命令（ 见 `FRepLayoutCmd`）。

例如一个 Actor，那么引擎就会给这个 Actor 的每个成员属性都创建一个 FRepParentCmd 对象

```cpp
 * A Parent Command represents a Top Level Property of the type represented by an FRepLayout.
 * A Child Command represents any Property (even nested properties).
 *
 * E.G.,
 *		Imagine an Object O, with 4 Properties, CA, DA, I, and S.
 *			CA is a fixed size C-Style array. This will generate 1 Parent Command and 1 Child Command for *each* element in the array.
 *			DA is a dynamic array (TArray). This will generate only 1 Parent Command and 1 Child Command, both referencing the array.
 *				Additionally, Child Commands will be added recursively for the element type of the array.
 *			S is a UStruct.
 *				All struct types generate 1 Parent Command for the Struct Property. Additionally:
 *					If the struct has a native NetSerialize method then it will generate 1 Child Command referencing the struct.
 *					If the struct has a native NetDeltaSerialize method then it will generate no Child Commands.
 *					All other structs will recursively generate Child Commands for each nested Net Property in the struct.
 *						Note, in this case there is no Child Command associated with the top level struct property.
 *			I is an integer (or other supported non-Struct type, or object reference). This will generate 1 Parent Command and 1 Child Command.


//////////////////////////////////////////////////////
//////////////////////////////////////////////////////
想象一个对象O，它有4个属性：CA、DA、I和S。
ca是一个固定大小的c风格数组。这会为数组中的*每个*元素生成1个父命令和1个子命令。
* da是动态数组（TArray）。这将只生成1个父命令和1个子命令，它们都引用数组。
* 此外，子命令将为数组的元素类型递归地添加。
s是一个UStruct。
* 所有结构类型生成一个父命令的结构属性。另外:
* 如果结构体有一个本机NetSerialize方法，那么它将生成一个引用该结构体的子命令。
* 如果结构体有一个本地NetDeltaSerialize方法，那么它将不会生成子命令。
* 所有其他结构体将递归地为结构体中的每个嵌套Net属性生成子命令。
* 注意，在这种情况下，没有子命令与顶层结构属性相关联。
* i是一个整数（或其他支持的非struct类型，或对象引用）。这将生成1个父命令和1个子命令。
```

<br>

#### (4.4.4) FRepChangedPropertyTracker

这个类用于存储有关连接之间共享的属性的元数据，包括给定属性是否为条件、活动和任何外部数据，可能需要重放。

<br>

#### (4.4.5) FRepState

表示同步的状态，只有两个成员属性，一个收数据，一个发数据：

```cpp
/** May be null on connections that don't receive properties. */
TUniquePtr<FReceivingRepState> ReceivingRepState;

/** May be null on connections that don't send properties. */
TUniquePtr<FSendingRepState> SendingRepState;
```

<br>
<br>
<br>

### 4.5 ReplicateProperties 属性对比同步【关键】

_**接下来再看 ReplicateProperties**_，从 FObjectReplicator 开始，这里是每个 UObject 同步属性的入口。

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-32.png)

- 创建 ChangelistState
- 判断是否开启了 PushModel，如果开启了 PushModel 会直接遍历已经置脏的属性列表：
- ![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-33.png)
- 如果没有还是会遍历 Parents 数组中的所有属性
- 更新 change list，这里会调用 _**FNetSerializeCB::UpdateChangelistMgr**_
- _**FRepLayout 中做属性对比，RepLayout->ReplicateProperties(...)**_
- 复制所有 custom delta properties (Fast Arrays, 等等...)
- 复制队列（Unreliable Function）


<br>
<br>

> **值得注意的是，Unreliable RPC 是随着属性同步一起处理的，处理完 RemoteFunctions 数据后，会清掉这个 OutBunch 数据，因此 Unreliable RPC 丢掉了就不会再发，而属性同步的 Bunc h包丢掉之后，还会继续同步丢掉的属性数据。**

> **另外 Reliable RPC 不是在这里处理，走的另外一个路径，后面可用单独的文章专门进行说明。Reliable RPC，每次调用是直接创建一个 Reliable 的 FOutBunch 包，因此其是及时的。并且引擎也保证了同一个 ActorChannel 的 RPC 是有顺序的。也是可靠的，对方一定会收到 RPC 调用。**

- Custom Data 的处理主要是针对于自定义的结构体，FastArray，这里会对 Customdata 进行属性对比。关于如果自定义同步数据，这里不展开了
- 在这个过程中，还有一个逻辑：在发现 RepFlag 发生变化后，会调用一次 FRepLayout::RebuildConditionalProperties 来构造 RepCondition 的数据，关于如何取得属性同步的数据这其中的逻辑后面再看

#### 4.6 UActorChannel::ReplicateRegisteredSubObjects 同步子对象

完成 Actor 自身的属性同步后，就来处理子对象的同步 (即 UActorComponent)

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-34.png)

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-35.png)

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-36.png)

**可以断点跑跑最后都是在 FObjectReplicator 中进行处理，执行的逻辑和前面提到的同步 Actor 自己的属性时是一样的。**

![](/res/img/post/12_ue5_5_net_sync_explore_actor_replicate/4-37.png)


>这里仍然是 UE 被动式同步逻辑的体现，引擎无法知道哪些 Component 的属性发生了变化，而是需要一个个 Component 去遍历，去检查属性有没有变化，走全所有的同步逻辑才能最终发现 Component 需要同步

<br>
<br>
<br>
<br>

# 参考

[1] ✨[Networking and Multiplayer (UE官方文档)](https://dev.epicgames.com/documentation/en-us/unreal-engine/networking-and-multiplayer-in-unreal-engine)
[2] ✨[《Exploring in UE4》网络同步原理深入（上）[原理分析] (Jerish)](https://zhuanlan.zhihu.com/p/34723199)
[3] ✨[UE4网络同步-基础流程 (南山搬砖道人)](https://zhuanlan.zhihu.com/p/532800522)
[4] ✨[UE4网络同步-RPC流程详解 (南山搬砖道人)](https://zhuanlan.zhihu.com/p/533738684)
[5] ✨[UE5 网络同步性能热点分析（NetDriver&ReplicationGraph）(嘉哥😁)](https://zhuanlan.zhihu.com/p/23120927574)

<br>
<br>
<br>
<br>

The end.
