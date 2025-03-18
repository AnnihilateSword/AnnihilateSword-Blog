---
title: 【UE5.5】Lyra工程为例，启用或指定ReplicationGraph类
tags: [UE5]
date: 2025-03-18
updated: 2025-03-18
cover: /res/img/post/11-ue5_5_Lyra_enable_ReplicationGraph/cover.png
top_img: /res/img/site/background.png
---

> 一次调试经历记录

<br>

✨[Replication Graph (UE官方文档)](https://dev.epicgames.com/documentation/en-us/unreal-engine/replication-graph-in-unreal-engine)

这块又被坑到，如果走修改配置文件的话需要注意：

```txt
ReplicationDriverClassName="/Script/ModuleName.ClassName"
```

<br>

这里填的是模块名，而不是项目名

![](/res/img/post/11-ue5_5_Lyra_enable_ReplicationGraph/0.png)

<br>

不过修改了但发现对 Lyra 不起效果，于是打开 ProjectSettings 查看设置发现这块禁用了，手动启用后不会保存？后来在配置文件中发现了：

![](/res/img/post/11-ue5_5_Lyra_enable_ReplicationGraph/1.png)

![](/res/img/post/11-ue5_5_Lyra_enable_ReplicationGraph/2.png)

![](/res/img/post/11-ue5_5_Lyra_enable_ReplicationGraph/3.png)

![](/res/img/post/11-ue5_5_Lyra_enable_ReplicationGraph/4.png)

<br>

Lyra 是在编辑器运行时 New 出来的 ReplicationGraph 设置：

![](/res/img/post/11-ue5_5_Lyra_enable_ReplicationGraph/5.png)

<br>

所以这里直接改配置就行

![](/res/img/post/11-ue5_5_Lyra_enable_ReplicationGraph/6.png)

<br>
<br>

The end.
