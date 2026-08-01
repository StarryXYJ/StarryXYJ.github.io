---
title: 各种剔除算法
date: 2026-07-31 01:18:33
tags:
- 笔记
- 计算机图形学
categories:
- [笔记，计算机科学，计算机图形学， 优化]
---


| 剔除算法                    | 位置             | 对象        | 是否减少 DrawCall | 是否减少 Vertex | 是否减少 Fragment | 侧     | 是否需要 Bake | 应用场景               |
| ----------------------- | -------------- | --------- | ------------- | ----------- | ------------- | ----- | --------- | ------------------ |
| Distance Culling        | CPU            | 太远物体      | ✅             | ✅           | ✅             | CPU   | ❌         | 大地图                |
| Layer Culling           | CPU            | Layer     | ✅             | ✅           | ✅             | CPU   | ❌         | UI、特效              |
| Frustum Culling         | CPU            | 视锥外物体     | ✅             | ✅           | ✅             | CPU   | ❌         | 所有项目               |
| Portal Culling          | CPU            | 房间外       | ✅             | ✅           | ✅             | CPU   | ✅（一般）     | 室内                 |
| Occlusion Culling       | CPU            | 被遮挡物体     | ✅             | ✅           | ✅             | CPU   | Unity默认需要 | 城市、室内              |
| Cluster Culling         | CPU/GPU        | Cluster   | 部分            | 部分          | 部分            | GPU居多 | ❌         | Nanite、Mesh Shader |
| Backface Culling        | GPU Rasterizer | 背面三角形     | ❌             | ❌           | ✅             | GPU   | ❌         | 所有项目               |
| Small Primitive Culling | GPU            | 很小三角形     | ❌             | 部分          | ✅             | GPU   | ❌         | Mesh Shader        |
| Early-Z                 | GPU            | 深度被遮挡像素   | ❌             | ❌           | ✅             | GPU   | ❌         | 所有项目               |
| Hi-Z Occlusion          | GPU            | 被挡住物体     | 有时            | 部分          | ✅             | GPU   | ❌         | Unreal、Unity DOTS  |
| Alpha Test Kill         | GPU            | clip() 像素 | ❌             | ❌           | 部分            | GPU   | ❌         | 植被                 |
| LOD                     | CPU            | 高模替换      | ❌             | 大量          | 部分            | CPU   | ❌         | 大世界                |


---


## Portal Culling

手动划分房间和门

只保留当前的房间和透过门可以看到的房间

具体看实际情况有很多实现

比如在一个长走廊的房间A

门是关上的，那么剔除房间A外的所有

如果是打开的，保留走廊和与走廊连接的打开的门的房间

## Occlusion Culling

 遮挡剔除，比较经典的剔除算法

一般需要预先烘培，把场景划分为多个cell，每个cell保存可以看到的物品集

不过局限还是很大的，一方面动态物品不行，根据玩法需要调整

另一方面因为多了剔除这一步骤，在室外环境反而性能会下降

Bake也比较慢

## Cluster Culling

将模型分为多个Cluster然后剔除

是不是很熟悉？Nanite的一部分

## Small Primitive Culling

如果一个三角形很小，shader依旧执行会导致性能问题

判断三角形占据的屏幕空间面积

小于一个像素就直接丢掉


但目前实现的比较少

一方面是由于也就现代游戏三角面越来越多，尤其是角色，小三角形才越来越多

一般说是以后mesh shader 出来了会广泛推广

那么什么是 mesh shader？

还是自己去搜搜吧，我目前也没有太多了解QAQ

## Hi-Z（Hierarchical Z）

如果每个像素都要比较再剔除还是太慢了

引入层级的空间加速结构

这部分应该不用多说了

---

```
CPU

│
├── Distance Culling
├── Layer Culling
├── Frustum Culling
├── Portal Culling
├── Occlusion Culling
├── LOD Selection
│
└── DrawCall
        │
        ▼
GPU Vertex Shader
        │
        ▼
Cluster / Meshlet Culling（现代GPU）
        │
        ▼
Backface Culling
        │
        ▼
Rasterization
        │
        ▼
Hi-Z
        │
        ▼
Early-Z
        │
        ▼
Pixel Shader
        │
        ▼
Alpha Test（clip）
        │
        ▼
Late-Z
        │
        ▼
Color Buffer
```