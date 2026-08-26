---
title: Unity 从 Job 到 ECS
date: 2026-8-15 21:11:32
tags:
- 笔记
- Unity
categories:
- [笔记]
---

Unity的DOTS主要包含了 Job System、 Burst 、 ECS三个部分

```
Unity DOTS / ECS
│
├── World                         ← ECS 世界 / 最顶层容器
│   │
│   ├── System                   ← 行为 / 逻辑层
│   │   │
│   │   ├── ComponentSystemGroup ← System 调度与分组
│   │   │     └── System
│   │   │
│   │   ├── ISystem / SystemBase
│   │   │
│   │   ├── SystemHandle         ← “这是哪个 System”
│   │   └── SystemState          ← System 的运行上下文
│   │         │
│   │         ├── Dependency
│   │         ├── EntityManager
│   │         ├── WorldUnmanaged
│   │         └── Query / Lookup ...
│   │
│   │
│   ├── EntityManager            ← ECS 数据世界的管理入口
│   │   │
│   │   ├── Entity              ← 实体，本质是 ID
│   │   │
│   │   ├── Component           ← 数据
│   │   │
│   │   ├── Archetype           ← Component 组合
│   │   │
│   │   └── Chunk               ← 真正批量存储 Component 数据
│   │
│   │
│   └── EntityQuery             ← System 查询数据的桥梁
│         │
│         └── Archetype → Chunk → Entity/Component
│
│
├── Job System                   ← 多线程执行层
│   │
│   ├── IJob
│   ├── IJobParallelFor
│   ├── IJobEntity
│   └── JobHandle / Dependency
│
│
└── Burst                        ← 底层代码优化
    │
    ├── LLVM
    ├── SIMD
    └── Native Code
```


# Job System

Unity 的 **C# Job System** 是 Unity 引擎提供的一套多线程框架，旨在让开发者能够以安全、简便的方式编写高性能的多线程代码


> 那么为什么感觉在C#的其他领域没有见过类似Job System的设计呢

本质上是运行环境的目标不同。比如内存管理，.NET 的 GC 是分代 GC，也可以在后台进行部分工作；服务端程序通常更在意长期吞吐、I/O 并发和尾延迟。游戏则有明确的帧预算：哪怕平均帧率够高，一次明显的 GC 或主线程阻塞也可能造成可感知的掉帧。Unity 可使用增量 GC，把一部分回收工作分摊到多帧；不过它只能缓和峰值，减少每帧托管分配仍然是更根本的做法。

这里也是同理，需求不同，方案也不同

总结下来就是

> 游戏中常见大量可并行的 CPU 计算，且有严格帧预算；服务端则常受网络、磁盘和数据库等 I/O 限制。两者都可能遇到 CPU 瓶颈，只是优化重点不同。

对于纯血.NET而言，主要的思想感觉是业务优先，对于性能瓶颈处也有逐级下降的方案，无论是ValueTask、Span，还是SIMD......

Unity这边就是提供了一个强限制的高性能方案了

---


> 数据并行 + 依赖调度 + worker 线程 + Native 数据


## 关键组成部分

### 1. Job 接口

根据任务类型，你通常会实现以下接口：

- **`IJob`：** 执行单个任务。
    
- **`IJobParallelFor`：** 将一个大的循环拆分成多个分片，分发到多个线程并行处理。
    
- **`IJobFor`：** 类似于 ParallelFor，但提供了更灵活的调度选项。
    

### 2. NativeContainer

由于 Job 运行在独立线程且不被允许引用托管对象（如 `List` 或类实例），你需要使用 `Unity.Collections` 提供的容器：

- **`NativeArray<T>`：** 最常用的容器。
    
- **`NativeList<T>`、`NativeHashMap<K, V>`** 等。
    
- **生命周期管理：** 必须手动释放（如 `Allocator.Temp`, `Allocator.TempJob`, `Allocator.Persistent`）。
    

### 3. Burst Compiler

Job System 负责调度与依赖，**Burst** 则负责优化符合其限制的计算代码。它会把 C# 的 IL 编译为原生代码，并在代码结构和目标 CPU 允许时使用 SIMD（单指令多数据）等优化。

> 在 Job 结构体上添加 `[BurstCompile]` 可以让 Burst 尝试编译它；是否成功以及是否产生 SIMD，应以 Burst Inspector 和 Profiler 的结果为准。


在现代 Unity (DOTS/ECS) 架构中，两者互为地基：

Job System 和 Burst 经常一起使用，但它们解决的是不同问题：前者让独立工作能并行执行，后者降低单次计算的成本。实际收益不是简单相乘，还取决于任务粒度、内存访问、同步位置和平台。

1. **Job System 约束了数据访问方式**：Job 通常使用结构体、值类型和 `NativeContainer`，避免在 worker 线程上访问托管对象。这种数据形态也更容易被 Burst 优化。
    
2. **读写标记首先服务于安全系统**：`[ReadOnly]`、`[WriteOnly]` 会声明容器在这个 Job 中的访问方式，使 Unity 能检查不安全的依赖和并发访问。它们也能为编译器提供额外信息，但不保证自动向量化。


---

## 工作流程

1. **定义 Job：** 创建一个 `struct` 实现接口，定义所需的 `NativeContainer` 变量和 `Execute` 方法。
    
2. **实例化与赋值：** 在主线程（如 `Update`）中创建该 Job 实例。
    
3. **调度 (Schedule)：** 调用 `job.Schedule()` 会登记 Job 及其依赖，随后由 Unity 的 worker 线程在合适的时机执行。调度本身通常不会等待 Job 结束。
    
4. **句柄 (JobHandle)：** 调度会返回一个 `JobHandle`，用于管理依赖关系（例如：Job B 必须在 Job A 完成后执行）。
    
5. **完成 (Complete)：** 调用 `handle.Complete()` 强制主线程等待该任务完成，以便安全地访问数据。
    

---

## 什么时候该用 Job System？

- **适合：** 大规模并行计算。例如：成千上万个粒子的物理模拟、复杂的路径规划（A*）、蒙皮变形、复杂的数学运算、大批量物体的变换更新。
    
- **不适合：** 涉及访问 Unity 核心 API（如 `GameObject.Transform`、`Input`、`UI` 等）的操作。大部分 Unity API 只能在主线程访问（但可以使用 `TransformAccessArray` 等特殊方案）。
    

---

## 同步/异步

### 异步调度 (Asynchronous Scheduling)

当你调用 `job.Schedule()` 时，它是**非阻塞**的。

- **动作：** 主线程登记这个 Job 和它的依赖关系，随后继续执行自己的工作。
    
- **结果：** 调用完成后，主线程会立即继续往下执行后续代码。此时，Worker Threads 可能已经开始处理这个 Job，也可能还在排队。
    
- **关键：** 这一步是异步的，主线程不会在这里卡顿。
    

### 句柄与依赖 (JobHandle & Dependencies)

`Schedule` 返回一个 `JobHandle`。这是你控制同步节奏的“遥控器”。

- 你可以让 Job B 依赖于 Job A：`jobB.Schedule(handleA)`。
    
- Job B 会在队列里静静等待，直到 Job A 完成。这在多线程间建立了逻辑上的同步，但对主线程依然是异步的。
    

### 显式同步 (Manual Synchronization)

当你调用 `handle.Complete()` 时，执行变为**同步**。

- **动作：** 如果此时 Job 还没跑完，主线程会等待它完成。
    
- **意义：** 只有在 `Complete()` 返回后，主线程才能安全地访问 Job 修改过的 `NativeArray` 数据。


---

## 对比Task

- **线程亲和性与分配：** Task 不是不能用于 Unity；网络请求、文件 I/O 等等待型工作很适合它。Task 的续体是否回到 Unity 主线程取决于同步上下文、调用位置和所用 API，不能想当然地假定它一定在主线程；访问 Unity API 前需要明确这一点。在热路径中频繁创建 Task、闭包或其他托管对象，也可能增加 GC 压力。GC 移动对象不会让正常的 C# 托管引用“悬挂”。
    
- **调度模型不同：** Job System 使用 worker 线程和工作窃取来分配可并行的工作，减少反复创建线程的成本，并尽量提高核心利用率。具体 worker 数量与调度细节由 Unity 和平台决定，不应假定它与物理核心一一对应，也不能认为上下文切换为零。


- **用 Task 的场景：** 从服务器下载资产、加载本地大文件、向 HTTP 接口发起异步请求。这些活大部分时间都在等待外部结果，不需要持续占用 CPU。
    
- **用 Job System 的场景：** 每帧都要计算的粒子运动、动态网格（Mesh）修改、同屏几千个怪物的 AI 寻路、物理大范围射线检测。这些活需要“撸起袖子硬算”，必须榨干 CPU。

| **特性**           | **C# Task (Aync/Await / .NET 线程池)**                                             | **Unity C# Job System**                                                   |
| ---------------- | ------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| **核心设计目标**       | 通用的异步组合模型；常用于网络请求、文件读写和等待外部响应，也可以承载 CPU 工作。 | 面向依赖明确、可分片的 CPU 密集型计算，例如矩阵运算、粒子和批量数值处理。 |
| **内存分配与 GC**     | `Task`、`async` 状态机和捕获闭包可能产生托管分配；是否分配取决于写法与执行路径。 | Job 是值类型，`NativeContainer` 的数据不在托管堆中；仍需关注 Native 分配、释放和调度成本，不能简单等同于“零成本”。 |
| **数据安全机制**       | 线程安全主要由开发者负责，可使用 `lock`、`SemaphoreSlim`、并发集合等工具。 | 安全系统可检查许多 NativeContainer 的读写冲突和生命周期问题；关闭安全检查或绕过限制后，责任仍在开发者。 |
| **调度方式**     | 使用 .NET 线程池，由运行时和操作系统调度。 | 使用 Unity 的 Job 调度器和 worker 线程；适合依赖明确、数据可分片的 CPU 工作。 |
| **与 Burst 的关系** | Burst 不会编译 `Task` 的托管异步执行路径；可以把其中的纯计算提取为 Burst 可编译的代码。 | Job System 与 Burst 常组合使用，但可以独立使用，且各自有不同的约束。 |

---
## 代码示例 (简单并行计算)



```cs
using UnityEngine;
using Unity.Collections;
using Unity.Jobs;
using Unity.Burst;

public class BasicJobExample : MonoBehaviour
{
    void Start()
    {
        // 这是 API 示例：为了能在同一方法内读取结果，下面会立即 Complete。
        // 性能敏感代码不应在每帧重复分配、立即等待并打印日志。
        // 1. 准备数据 (TempJob 适合生命周期不超过几帧的临时数据)
        NativeArray<float> input = new NativeArray<float>(1000, Allocator.TempJob);
        NativeArray<float> result = new NativeArray<float>(1000, Allocator.TempJob);

        // 初始化数据
        for (int i = 0; i < input.Length; i++) input[i] = i;

        // 2. 实例化 Job
        var myJob = new SimpleJob { input = input, result = result };

        // 3. 调度 Job (每 64 个元素为一个 Batch)
        JobHandle handle = myJob.Schedule(input.Length, 64);

        // 4. 等待完成；实际项目应尽量把 Complete 推迟到真正需要结果的位置
        handle.Complete();

        // 5. 打印结果并释放内存
        Debug.Log($"Index 10: {result[10]}"); // 输出 100

        input.Dispose();
        result.Dispose();
    }
}

[BurstCompile]
struct SimpleJob : IJobParallelFor
{
    [ReadOnly] public NativeArray<float> input;
    public NativeArray<float> result;

    public void Execute(int index)
    {
        result[index] = input[index] * input[index];
    }
}
```


这个例子模拟了一个常见的场景：**上万个物体需要计算朝向目标的移动，并同时避开某个危险点**。

```cs
using UnityEngine;
using Unity.Collections;
using Unity.Jobs;
using Unity.Burst;
using UnityEngine.Jobs; // 专门用于处理 Transform 的命名空间

public class AdvancedJobExample : MonoBehaviour
{
    public int objectCount = 10000;
    public Transform target;
    public Transform dangerZone;

    private NativeArray<Vector3> _velocities;
    private TransformAccessArray _transformArray; // 专门用于在 Job 中读写 Transform 的容器
    private JobHandle _movementHandle;

    void Start()
    {
        _velocities = new NativeArray<Vector3>(objectCount, Allocator.Persistent);
        Transform[] transforms = new Transform[objectCount];

        for (int i = 0; i < objectCount; i++)
        {
            var go = GameObject.CreatePrimitive(PrimitiveType.Cube);
            go.transform.position = Random.insideUnitSphere * 20f;
            transforms[i] = go.transform;
        }

        // 将所有物体的 Transform 封装进专用数组
        _transformArray = new TransformAccessArray(transforms);
    }

    void Update()
    {
        // 下一次写入同一份数据前，先确保上一帧的 Job 已结束。
        _movementHandle.Complete();

        // 定义 Job
        var movementJob = new MovementAndAvoidanceJob
        {
            targetPos = target.position,
            dangerPos = dangerZone.position,
            deltaTime = Time.deltaTime,
            velocities = _velocities
        };

        // 调度：IJobParallelForTransform 专门优化了对 Transform 的并行访问
        _movementHandle = movementJob.Schedule(_transformArray);

        // 这里不立刻 Complete，让它与本帧后续不依赖 Transform 结果的工作并行。
        // 如果主线程在同一帧读取这些 Transform，仍可能需要先等待这个 Job。
    }

    void OnDestroy()
    {
        _movementHandle.Complete();
        if (_velocities.IsCreated) _velocities.Dispose();
        if (_transformArray.isCreated) _transformArray.Dispose();
    }
}

[BurstCompile]
struct MovementAndAvoidanceJob : IJobParallelForTransform
{
    [ReadOnly] public Vector3 targetPos;
    [ReadOnly] public Vector3 dangerPos;
    [ReadOnly] public float deltaTime;

    public NativeArray<Vector3> velocities;

    public void Execute(int index, TransformAccess transform)
    {
        Vector3 currentPos = transform.position;

        // 1. 计算朝向目标的拉力
        Vector3 directionToTarget = (targetPos - currentPos).normalized;

        // 2. 避障逻辑：如果离危险点太近，产生斥力
        Vector3 offsetToDanger = currentPos - dangerPos;
        float distanceToDanger = offsetToDanger.magnitude;
        Vector3 avoidance = Vector3.zero;

        if (distanceToDanger < 5f)
        {
            avoidance = offsetToDanger.normalized * (5f - distanceToDanger);
        }

        // 3. 合成速度并更新位移
        Vector3 targetVelocity = (directionToTarget + avoidance * 2f) * 5f;
        velocities[index] = Vector3.Lerp(velocities[index], targetVelocity, deltaTime * 2f);
        
        transform.position += velocities[index] * deltaTime;
        
        // 4. 让物体始终朝向移动方向
        if (velocities[index] != Vector3.zero)
            transform.rotation = Quaternion.LookRotation(velocities[index]);
    }
}
```

- **`IJobParallelForTransform`**： 通常 Job System 禁止访问 `UnityEngine.Object`（如 `Transform`），因为它们是非线程安全的。但 Unity 提供了这个特殊的接口和 `TransformAccessArray`，允许你在多线程中**高性能地读写**物体的坐标、旋转和缩放。
    
- **`TransformAccess`**：它是 Job 中访问 Transform 的受限入口，而不是普通的 `Transform` 引用。它解决的是线程安全的访问边界；Transform 数据仍是引擎侧数据，不能把它当成纯 `NativeArray` 那样的连续缓存布局。
    
- **计算密度**：这个例子包含归一化、距离计算、插值和旋转。对象数量足够多时，多线程可能降低主线程耗时；但实际收益要用 Profiler 验证。Transform 访问、任务调度和过早 `Complete()` 都可能吞掉收益，Burst 也不会保证每一行都被 SIMD 向量化。
    
- **混合逻辑**： 你可以看到 `velocities` 这个 `NativeArray` 被用来存储物体的状态（速度），实现了“逻辑状态（NativeArray）”与“表现（Transform）”的解耦。

---

# Burst

Burst 是 Unity 专门为 **HPC#（高性能 C#）** 设计的后端编译器。它将 C# 编译出的中间语言（IL）字节码转换为 **LLVM IR**，进而通过 LLVM 编译器后端，直接生成针对目标平台 CPU 架构（如 x86、ARM）极度优化的**原生机器码（Native Code）**

---

## 核心底层机制与优化手段

- **HPC# 的限制：** Burst 支持的是 C# 的一个受限子集。把 Job 的执行路径写成以结构体、值类型、`Unity.Mathematics` 和原生容器为主，最容易获得稳定结果；避免在热路径中分配托管对象、使用虚调用或依赖大多数 UnityEngine API。具体支持范围会随 Burst 版本变化，应以编译器诊断为准。
    
- **自动向量化 (Auto-Vectorization / SIMD)：** Burst 会分析循环与数据访问，并在依赖关系清晰、目标 CPU 支持时生成 SIMD 指令（如 AVX2、Neon）。`Unity.Mathematics` 的类型和函数通常更适合这类计算，但不是触发 SIMD 的保证。
    
- **别名与数据访问：** 当容器的读写关系清晰时，Burst 有更多空间进行重排和向量化。`[ReadOnly]`、`[WriteOnly]` 等标记应按真实访问方式使用；它们既帮助安全检查，也可能帮助优化，但不意味着不同内存一定不会重叠。
    
- **常规编译优化：** Burst 还会进行死代码消除、常量传播、循环优化等。优化结果取决于具体代码；遇到性能问题时，应结合 Burst Inspector 和 Profiler，而不是假设编译器会自动解决所有分支和内存访问问题。

## 案例

```cs
using Unity.Burst;
using Unity.Collections;
using Unity.Jobs;
using Unity.Mathematics;

// 1. 必须添加 [BurstCompile] 
[BurstCompile(CompileSynchronously = true)]
public struct OptimizationJob : IJobFor
{
    [ReadOnly] public NativeArray<float3> Positions;
    public NativeArray<float> Results;

    public void Execute(int index)
    {
        // 2. 使用 Burst 支持的 C# 子集；Unity.Mathematics 的向量类型更便于优化。
        //    是否生成 SIMD 仍应以 Burst Inspector 为准。
        Results[index] = math.length(Positions[index]);
    }
}
```

---

# ECS/DOTS

ECS(Entity Component System) 是DOTS的一种设计

将类似的内容放在一起保重更好的并行性以及缓存命中率

Entity 实体包含了多个component引用

Component是实际的数据载体，同类型的Component放在一起

System 负责逻辑，主要包括查询与处理两个部分


> 由于Entity和GameObject的底层差异，ECS的内容都放在Subscene下面，Entity的产生一般是由传统Monobehaviour的Authoring实现一个烘培逻辑，比如如何给Entity添加组件等，然后烘培得到。

## 一、 Component（数据层：纯结构体，无行为）

ECS 中的组件必须是 struct，且只能包含基础数据类型（如 float, int）或 Unity.Mathematics 的数据类型（如 float3）。绝对不能包含 GameObject 或 Transform 等引用类型。

### 变种 1：标准组件

用于存储常规的游戏数据。


```cs
using Unity.Entities;
using Unity.Mathematics;

// 必须继承 IComponentData 接口
public struct MoveSpeed : IComponentData
{
    public float Value;
    public float3 Direction; // 使用 Unity 优化的数学类型
}
```

### 变种 2：Tag 组件

不包含任何字段，仅作为“过滤器”或“分类标签”使用。

```cs
using Unity.Entities;

// 空结构体，用来筛选特定实体（例如：只有带有此标签的怪物才会受到伤害）
public struct IsEnemyTag : IComponentData { }
```

### 变种 3：可禁用组件（状态开关）

允许你在不移除组件的前提下，直接禁用它。这比频繁添加/移除组件性能高出数倍。


```cs
using Unity.Entities;

// 必须额外继承 IEnableableComponent
public struct FrozenStatus : IComponentData, IEnableableComponent
{
    public float RemainingTime;
}
```

## 二、 Authoring & Baker（编辑与烘焙层：桥接传统面板与 ECS）

因为 Unity 编辑器无法直接识别 ECS 的结构体，我们需要通过 MonoBehaviour（Authoring）接收面板参数，再通过一个 Baker 类将其转化为纯数据。

### 变种 1：标准数据烘焙（1 对 1 转化）

最基础的烘焙，将 Mono 面板的数值传给 Component。


```cs
using Unity.Entities;
using UnityEngine;

// 1. 挂载到 GameObject 上的脚本
public class MoveSpeedAuthoring : MonoBehaviour
{
    public float speed;
}

// 2. 内部嵌套的烘焙器（Unity 会自动识别并执行）
public class MoveSpeedBaker : Baker<MoveSpeedAuthoring>
{
    public override void Bake(MoveSpeedAuthoring authoring)
    {
        // 获取当前被烘焙的实体
        Entity entity = GetEntity(TransformUsageFlags.Dynamic);
        
        // 为该实体添加组件并赋值
        AddComponent(entity, new MoveSpeed 
        { 
            Value = authoring.speed 
        });
    }
}
```

### 变种 2：标签与多组件综合烘焙

一个 Authoring 脚本可以同时为实体挂载多个不同的组件（包括 Tag 标签）。



```cs
using Unity.Entities;
using UnityEngine;

public class EnemyAuthoring : MonoBehaviour
{
    public float spawnSpeed;
    public bool startFrozen;
}

public class EnemyBaker : Baker<EnemyAuthoring>
{
    public override void Bake(EnemyAuthoring authoring)
    {
        Entity entity = GetEntity(TransformUsageFlags.Dynamic);

        // 烘焙移动组件
        AddComponent(entity, new MoveSpeed { Value = authoring.spawnSpeed });
        
        // 烘焙 Tag 组件（标签直接用空的结构体）
        AddComponent<IsEnemyTag>(entity);

        // 烘焙可禁用组件
        AddComponent(entity, new FrozenStatus { RemainingTime = 5.0f });
        if (!authoring.startFrozen)
        {
            // 如果初始不冻结，直接将该实体上的这个组件设置为禁用状态
            SetComponentEnabled<FrozenStatus>(entity, false);
        }
    }
}
```

## 三、 System（系统层：逻辑与数据处理）

System 负责执行所有的游戏逻辑。现代 Unity ECS 强烈推荐使用更高效的 ISystem（基于结构体和 Burst 编译器）来代替老旧的 SystemBase（基于类）。

### 变种 1：SystemAPI.Query 主线程查询（最直观、写起来最快）

直接在主线程中用 foreach 循环遍历数据，适合逻辑简单或需要调用主线程 API 的场景。



```cs
using Unity.Burst;
using Unity.Entities;
using Unity.Transforms;

[BurstCompile] // 开启 Burst 编译器，让代码运行速度逼近 C++
public partial struct MoveSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        float deltaTime = SystemAPI.Time.DeltaTime;

        // Query 内部：使用 RefRW 读写，RefRO 只读
        // LocalTransform 是 Unity 官方提供的内置位置组件
        foreach (var (transform, speed) in SystemAPI.Query<RefRW<LocalTransform>, RefRO<MoveSpeed>>())
        {
            transform.ValueRW.Position += transform.ValueRO.Forward() * speed.ValueRO.Value * deltaTime;
        }
    }
}
```

注：因为 ECS 的核心思想是**内存连续性**（将数据紧密排列以提高 CPU 缓存命中率），普通的 C# 引用类型会破坏这种结构。所以 Unity 引入了一套特殊的结构体来安全地引用这些内存。

| **名字**           | **读写权限** | **核心属性**                   | **适用场景**                   |
| ---------------- | -------- | -------------------------- | -------------------------- |
| **RefRW**        | 可读可写     | .ValueRO / .ValueRW        | 需要修改组件里的数据（如：血量减少、位置更新）    |
| **RefRO**        | 只能读取     | .ValueRO                   | 仅仅获取数据作为参考（如：读取攻击力、读取速度限制） |
| **EnabledRefRW** | 开关可读写    | .ValueRO / .ValueRW (bool) | 控制组件是否生效（如：开启或关闭某个状态 AI）   |
| **EnabledRefRO** | 开关只读     | .ValueRO (bool)            | 判断某个组件目前是否处于激活状态           |

### 变种 2：IJobEntity 多线程并行处理（性能极限，推荐）

将计算量庞大的循环丢给多线程（Job System）分摊，极大地压榨 CPU 性能。


```cs
using Unity.Burst;
using Unity.Entities;
using Unity.Transforms;

[BurstCompile]
public partial struct MoveJobSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // 2. 实例化 Job 并传入所需变量，然后调度它去多线程执行
        var moveJob = new CalculateMoveJob
        {
            DeltaTime = SystemAPI.Time.DeltaTime
        };
        
        // ScheduleParallel 代表多线程并行分摊所有实体
        moveJob.ScheduleParallel(); 
    }
}

// 1. 定义一个 Job 结构体，继承 IJobEntity
[BurstCompile]
public partial struct CalculateMoveJob : IJobEntity
{
    public float DeltaTime;

    // 里面的参数由系统自动匹配。ref 代表可读写，in 代表只读（等同于 RefRO）
    [BurstCompile]
    public void Execute(ref LocalTransform transform, in MoveSpeed speed)
    {
        transform.Position += transform.Forward() * speed.Value * DeltaTime;
    }
}
```

### 变种 3：结合 Tag 标签与 Enable 开关的 System

演示如何在 System 中利用标签进行数据过滤，以及如何读取开关状态。


```cs
using Unity.Burst;
using Unity.Entities;

[BurstCompile]
public partial struct EnemyFreezeSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        float deltaTime = SystemAPI.Time.DeltaTime;

        // WithAll<IsEnemyTag>：表示只有带了这个标签的实体才会被筛选出来
        foreach (var (frozenRef, entity) in SystemAPI.Query<RefRW<FrozenStatus>>().WithAll<IsEnemyTag>().WithEntityAccess())
        {
            // 检查这个组件当前是否处于开启状态
            if (SystemAPI.IsComponentEnabled<FrozenStatus>(entity))
            {
                frozenRef.ValueRW.RemainingTime -= deltaTime;
                if (frozenRef.ValueRO.RemainingTime <= 0)
                {
                    // 时间到了，在系统关闭这个实体的冻结状态组件
                    SystemAPI.SetComponentEnabled<FrozenStatus>(entity, false);
                }
            }
        }
    }
}
```

## SystemBase

与基于结构体的 ISystem 不同，SystemBase 是基于**类（Class）**的系统模型。它允许你在系统内编写**非托管/托管混合代码**。虽然它的计算性能上限不如 ISystem，但它是 **ECS 世界与传统 Unity（GameObject、UI、输入系统）沟通的绝佳桥梁**。

## 一、 核心骨架

SystemBase 重写了基类的虚方法，因此必须使用 protected override。它不需要显式传入 SystemState 参数。


```cs
using Unity.Entities;
using Unity.Transforms;
using UnityEngine; // 允许引入传统 UnityEngine 命名空间

// 1. 必须声明为 partial class，且继承自 SystemBase
public partial class RotateSystemBase : SystemBase
{
    // 2. 必须使用 protected override 重写 OnUpdate
    protected override void OnUpdate()
    {
        // 3. 可以直接获取时间，无需通过 state
        float deltaTime = SystemAPI.Time.DeltaTime;

        // 4. 标准的 SystemAPI 查询同样适用
        foreach (var (t, r) in SystemAPI.Query<RefRW<LocalTransform>, RefRO<RotateSpeed>>())
        {
            t.ValueRW = t.ValueRW.RotateY(r.ValueRO.value * deltaTime);
        }
    }
}
```

## 二、 核心实战变种

### 变种 1：主线程外部交互（输入与事件）

这是 SystemBase 最核心的舞台。ISystem 无法直接调用 Input 或 Debug.Log，而 SystemBase 可以轻松做到。



```cs
using Unity.Entities;
using UnityEngine; // 引入传统 Unity 库

public partial class PlayerInputSystem : SystemBase
{
    protected override void OnUpdate()
    {
        // 1. 直接读取键盘/鼠标输入
        float horizontal = Input.GetAxisRaw("Horizontal");
        float vertical = Input.GetAxisRaw("Vertical");
        bool isJumpPressed = Input.GetButtonDown("Jump");

        // 2. 将输入数据写入到 ECS 的玩家组件中
        foreach (var inputRef in SystemAPI.Query<RefRW<PlayerInputData>>())
        {
            inputRef.ValueRW.Movement = new Unity.Mathematics.float2(horizontal, vertical);
            inputRef.ValueRW.JumpRequested = isJumpPressed;
        }
        
        // 3. 甚至可以写传统 debug
        if(isJumpPressed) Debug.Log("Player pressed jump inside SystemBase!");
    }
}
```

### 变种 2：与传统 GameObject 混合交互（如 UGUI UI 更新）

当游戏数据在 ECS 中运行，但你的 UI 依然是传统的 UGUI 或 UI Toolkit 时，用 SystemBase 来做同步。



```cs 
using Unity.Entities;
using UnityEngine;
using UnityEngine.UI; // 引入 UI 命名空间

public partial class HealthUISystem : SystemBase
{
    // 可以在外部或初始化时赋值（非托管的 ISystem 绝对做不到这一点）
    private Slider healthSlider; 

    protected override void OnStartRunning()
    {
        // 初始化时设法找到场景中的 UI 组件
        var uiGO = GameObject.FindWithTag("HealthSlider");
        if (uiGO != null) healthSlider = uiGO.GetComponent<Slider>();
    }

    protected override void OnUpdate()
    {
        if (healthSlider == null) return;

        // 遍历玩家血量组件，同步到 UGUI 的 Slider 上
        foreach (var playerHealth in SystemAPI.Query<RefRO<HealthComponent>>().WithAll<PlayerTag>())
        {
            // 直接操作托管对象 Slider
            healthSlider.value = playerHealth.ValueRO.CurrentHealth / playerHealth.ValueRO.MaxHealth;
        }
    }
}
```

### 变种 3：使用 Entities.ForEach（经典多线程写法）

虽然 SystemBase 是一个类，但它提供了一个非常直观的、支持 Burst 编译的多线程 API：Entities.ForEach。它会自动为你生成背后的 Job。



```cs
using Unity.Burst;
using Unity.Entities;
using Unity.Transforms;

public partial class RotateForEachSystem : SystemBase
{
    protected override void OnUpdate()
    {
        float deltaTime = SystemAPI.Time.DeltaTime;

        // 1. 显式开启 .WithBurst() 允许这一段匿名表达式被 Burst 编译加速
        // 2. 使用 ref 和 in 关键字进行数据传递
        Entities
            .WithBurst() 
            .ForEach((ref LocalTransform transform, in RotateSpeed speed) =>
            {
                transform = transform.RotateY(speed.value * deltaTime);
            })
            .ScheduleParallel(); // 3. 直接一行多线程并行化调度
    }
}
```

## 选择

- **ISystem**：处理纯粹的、大批量的底层计算（如：同屏 1 万个单位的 AI、移动、碰撞、物理模拟）。
    
- **SystemBase**：处理需要连接“外部世界”的接口逻辑（如：读取键盘鼠标输入、播放音效、生成传统的 GameObject 特效、传递数据给 UI 面板）。

# SystemHandle & SystemState

| **概念**           | **角色**              | **本质**         | **存放在哪？**                    |
| ---------------- | ------------------- | -------------- | ---------------------------- |
| **SystemHandle** | **身份标识 (ID)**       | 轻量级非托管结构体      | 可存在于组件、系统或字段中                |
| **SystemState**  | **运行上下文 (Context)** | 包含系统元数据和接口的结构体 | 仅作为 `ISystem` 方法的 `ref` 参数传递 |


## 解析

### SystemHandle

- **定义**：一个指向特定系统实例的“句柄”。
    
- **特点**：
    
    - **非托管**：它是 `unmanaged` 类型，可以安全地放入 `IComponentData` 中。
        
    - **跨系统引用**：由于 `ISystem` 是结构体，无法直接持有对方的引用，必须通过 `SystemHandle` 来定位。
        
- **常用操作**：`state.World.GetExistingSystemHandle<T>()`。
    

### SystemState

- **定义**：系统的“控制面板”，提供了访问 World、依赖项（JobHandle）和查询缓存的权限。
    
- **核心职责**：
    
    - **依赖管理**：通过 `state.Dependency` 确保 Job 的顺序执行。
        
    - **组件查找**：通过 `state.GetComponentLookup<T>()` 获取在并行 Job 中读写数据的权限。
        
    - **自我控制**：控制系统自身的开启或关闭 (`state.Enabled`)。


## 应用案例

### 案例 A：跨系统控制 (利用 Handle 在组件中存引用)

**场景**：当玩家捡到特殊道具时，触发一个特定的“成就系统”或“特效系统”。


```cs
// 1. 将系统句柄存入组件
public struct TriggerSystemComponent : IComponentData {
    public SystemHandle TargetSystem; 
}

// 2. 在逻辑系统中调用
public partial struct LogicSystem : ISystem {
    public void OnUpdate(ref SystemState state) {
        foreach (var trigger in SystemAPI.Query<RefRO<TriggerSystemComponent>>()) {
            // 通过句柄找到目标系统的状态并开启它
            state.World.Unmanaged.ResolveSystemStateRef(trigger.ValueRO.TargetSystem).Enabled = true;
        }
    }
}
```

### 案例 B：手动调度 Job 链 (利用 State 管理依赖)

**场景**：确保物理计算 Job 在动画计算 Job 之后执行，防止数据竞争。


```cs
[BurstCompile]
public partial struct PhysicsSystem : ISystem {
    public void OnUpdate(ref SystemState state) {
        var job = new MyPhysicsJob { ... };
        
        // 读取并更新 state.Dependency，这是 ECS 自动同步并行的核心
        state.Dependency = job.ScheduleParallel(state.Dependency);
    }
}
```

### 案例 C：性能优化 (利用 State 缓存查询)

**场景**：高频检查场景中是否有“Boss”实体。


```cs
public partial struct BossMonitorSystem : ISystem {
    private EntityQuery _bossQuery;

    public void OnCreate(ref SystemState state) {
        // 使用 state 获取并缓存查询，避免在 OnUpdate 中重复创建
        _bossQuery = state.GetEntityQuery(ComponentType.ReadOnly<BossTag>());
    }

    public void OnUpdate(ref SystemState state) {
        // 只有当查询到 Boss 时才运行后续逻辑
        state.Enabled = !_bossQuery.IsEmpty;
    }
}
```


# Data Manipulate

| **维度**   | **SystemAPI** | **EntityManager** | **EntityCommandBuffer (ECB)** |
| -------- | ------------- | ----------------- | ----------------------------- |
| **执行时机** | 立即执行          | 立即执行（产生同步点）       | **延迟执行**（Playback 时）          |
| **线程安全** | 主线程专用         | **主线程专用**         | **Job 线程安全** (ParallelWriter) |
| **主要职责** | 读写组件数据、查询     | 增删实体、实例化          | 在 Job 中安全地增删实体/组件             |
| **使用建议** | 90% 的数据逻辑     | 极少数单次触发的初始化       | 大规模并行实例化/销毁                   |
|          |               |                   | 查询少量物体，每个物体执行复杂逻辑             |


### 1. SystemAPI (自动档 / 逻辑核心)

**定位**：System 内部最推荐的交互入口。它是 Source Generator（源码生成器）的宠儿。

- **核心功能**：
    
    - **查询**：`SystemAPI.Query<RefRW<T>, RefRO<K>>()`（最常用的遍历方式）。
        
    - **单例**：`SystemAPI.GetSingleton<T>()`（快速访问配置或全局状态）。
        
    - **简易读写**：`SystemAPI.GetComponent<T>(entity)` / `SetComponent<T>(entity, data)`。
        
- **为什么用它**：
    
    - **自动依赖跟踪**：它会自动处理 `JobHandle` 依赖，防止多线程读写冲突。
        
    - **代码简洁**：语义化强，不需要手动创建和销毁 Query 变量。
        
- **局限**：只能“修改数值”，不能“改变结构”（不能增删实体/组件）。
    

### 2. EntityManager (手动档 / 结构修改)

**定位**：ECS 世界的“上帝接口”，直接管理内存中的所有实体和组件。

- **核心功能**：
    
    - **结构变更 (Structural Changes)**：创建/销毁实体 (`CreateEntity`, `DestroyEntity`)，添加/删除组件 (`AddComponent`, `RemoveComponent`)。
        
    - **实例化**：`Instantiate(prefabEntity)`。
        
    - **元数据操作**：设置实体名称 (`SetName`)，检查实体有效性 (`Exists`)。
        
- **注意点**：
    
    - **同步点 (Sync Point)**：在主线程调用 `EntityManager` 的结构变更方法会强制所有并行 Job 完成，可能导致性能卡顿（所谓的“冒泡”）。
        
    - **主线程限制**：它不是线程安全的，不能在 `IJobEntity` 或 `IJobChunk` 等后台线程中直接使用。
        

### 3. EntityCommandBuffer / ECB (延迟档 / 性能加速)

**定位**：一个“待办事项列表”。它记录你想做的修改，并在稍后的安全时机统一执行。

- **核心功能**：
    
    - 它是 `EntityManager` 的**异步替代品**。
        
    - 在 Job 线程中，你可以记录 `ecb.AddComponent(entity, data)`，但组件不会立即加上，而是在 System 结束后的某个 **Playback（回放）** 阶段统一处理，不调用结尾会默认调用

`SystemAPI.GetSingleton<EndSimulationEntityCommandBufferSystem.Singleton>()`

在 ISystem 中使用 ECB，必须通过系统的单例（Singleton）来获取。最常用的两个安全时机是：

- **EndSimulationEntityCommandBufferSystem**：当前帧的所有逻辑执行完毕时回放（最常用）。
    
- **BeginSimulationEntityCommandBufferSystem**：下一帧的逻辑开始之前回放。
    

## 实战代码变种

### 变种 1：在主线程（System 内部）直接使用 ECB

适用于不需要多线程的简单生成/销毁（如游戏初期的批量初始化）。



```cs
using Unity.Burst;
using Unity.Entities;

[BurstCompile]
public partial struct SpawnerMasterSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // 1. 从官方提供的单例中获取并创建 ECB
        var ecbSingleton = SystemAPI.GetSingleton<EndSimulationEntityCommandBufferSystem.Singleton>();
        EntityCommandBuffer ecb = ecbSingleton.CreateCommandBuffer(state.WorldUnmanaged);

        // 2. 遍历触发器（例如：所有生成点）
        foreach (var (spawner, entity) in SystemAPI.Query<RefRW<SpawnerData>>().WithEntityAccess())
        {
            if (spawner.ValueRO.ShouldSpawn)
            {
                // 创建实体（传入预制体 Prefab）
                Entity newEntity = ecb.Instantiate(spawner.ValueRO.Prefab);
                
                // 销毁生成点实体自身
                ecb.DestroyEntity(entity);
            }
        }
    }
}
```

### 变种 2：在多线程 IJobEntity 中安全使用

**关键点**：多线程中必须使用 EntityCommandBuffer.ParallelWriter，且操作时必须传入 sortKey（排序键）。Unity 依靠 sortKey 确保多线程写入的指令在回放时顺序完全一致。



```cs
using Unity.Burst;
using Unity.Entities;

[BurstCompile]
public partial struct BulletCollisionSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // 1. 获取单例
        var ecbSingleton = SystemAPI.GetSingleton<EndSimulationEntityCommandBufferSystem.Singleton>();
        
        // 2. 注意：这里创建的是普通 ecb，通过 .AsParallelWriter() 传给 Job
        var ecb = ecbSingleton.CreateCommandBuffer(state.WorldUnmanaged);

        var collisionJob = new BulletCollisionJob
        {
            // 转换为多线程并行写入器
            ECB = ecb.AsParallelWriter() 
        };

        collisionJob.ScheduleParallel();
    }
}

[BurstCompile]
public partial struct BulletCollisionJob : IJobEntity
{
    // 3. 必须使用 ParallelWriter 类型
    public EntityCommandBuffer.ParallelWriter ECB;

    // 4. [EntityIndexInQuery] 是 Unity 内置的特殊标签，会自动把当前实体在查询中的索引传给 chunkIndex
    // 这个 chunkIndex 就是我们需要的 sortKey
    [BurstCompile]
    public void Execute([EntityIndexInQuery] int chunkIndex, Entity entity, in BulletComponent bullet)
    {
        if (bullet.IsHit)
        {
            // 5. 所有的 ECB 操作必须传入第一个参数：sortKey (即 chunkIndex)
            // 销毁当前的子弹实体
            ECB.DestroyEntity(chunkIndex, entity);
            
            // 产生一个爆炸特效实体（假设预制体存在 bullet 内部）
            Entity fx = ECB.Instantiate(chunkIndex, bullet.ExplosionFxPrefab);
        }
    }
}
```

## ECB 常用 API 速查表

在调用以下方法时，如果是在 **ParallelWriter（多线程）** 中，首个参数必须传入 sortKey。

|**方法名**|**作用**|**备注**|
|---|---|---|
|**Instantiate(prefab)**|克隆一个预制体生成新实体|返回的 Entity 是一个临时占位符，可以立即用于对其添加组件|
|**CreateEntity()**|创建一个没有任何组件的空实体|适合动态完全由代码构建的实体|
|**DestroyEntity(entity)**|销毁指定实体|该实体上的所有组件会在回放时一并被移除销毁|
|**AddComponent(entity, component)**|为实体动态添加组件|频繁在 Job 内添加/移除组件会造成物理 Chunk 搬移，注意性能|
|**RemoveComponent(entity)**|为实体移除指定组件|同上|
|**SetComponent(entity, component)**|修改实体的组件数值|仅用于修改，若不确定实体是否有该组件，需先确保已添加|
        

---

# Outro

当然这只是ecs的一角，其他还有buffer，Managed Component ,Auxiliary Data等等
