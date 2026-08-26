---
title: 阴影
date: 2026-06-12 21:02:48
tags:
- 笔记
- 计算机图形学
categories:
- [笔记，计算机科学，计算机图形学]
---

阴影算是处理渲染方程可见项的那一部分

## Shadow Map

最朴素的阴影处理，也是后续阴影算法的基础

因为是可见项，复用平常光栅化中对于什么可以看到的处理的想法——深度图

1. 我们先以光源为相机，渲染出一张光源能看到什么的深度图

2. 我们再次正常渲染，把每个表面像素变换到光源的裁剪空间，并与 Shadow Map 中**同一深度度量**的值比较。接收点深度不大于图中深度，说明光线没有先撞到别的表面；反之则被遮挡。

### 问题

#### 阴影痤疮 (Shadow Acne)

深度图量化、光栅化规则与接收面比较之间存在离散误差，同一个表面会错误地遮挡自己，于是出现密密麻麻的黑色条纹或斑点，尤其容易出现在与光线夹角很大的平面上。

> 接收面越倾斜，深度图中一个 texel 覆盖的接收面范围越大，深度误差也越明显。这就是为什么 bias 往往需要随表面斜率变化。

为了解决这个问题，常常加入 深度偏移 

相当于考虑误差范围

$$\text{Shadow} = (d_{real} > d_{map} + \text{bias}) ? 1.0 : 0.0$$

#### 彼得潘现象

如果偏移量太大，可能会导致物体投射的阴影和物体的根部脱离

实践里通常组合使用常量 bias、slope-scale depth bias 和 normal bias：前两者补偿深度比较误差，normal bias 沿接收面法线略微移动采样位置。它们都需要在 acne 与 Peter Panning 之间取舍，所以应按光源、阴影分辨率和场景尺度调参，而不是期望一个全局常量解决所有情况。

#### 走样

这是 Shadow Map 的采样分辨率问题。提高阴影图分辨率、使用级联阴影图（CSM）、稳定投影和 PCF 等过滤都能缓解；动态分辨率并不是常规的首选手段。

#### 硬阴影

边缘没有过渡

但实际上仔细思考，在几何光学的角度下，对于点光源或者方向光而言，是否可见确实是0或1的，理论上阴影就是0或1

几何光学角度，软阴影来自于面光源，体积光源之类的，但无所谓，图形学从来不是物理正确的，看上去对就是对的，因此会给光源配置"虚拟"宽度这些

---


## PCF (Percentage-Closer Filtering)

PCF 是对“深度比较结果”做滤波的技术

它不直接模糊 Shadow Map 的深度，而是在确定可见度时对多次深度比较的 0/1 结果求平均。

取一定范围的 texel 比较，得到一组 0/1 结果，取（加权）平均作为 visibility。它能缓解阴影边缘的采样锯齿；固定大小的核也会得到固定宽度的软边近似。

应该很好理解

那么我们给阴影用上这一套抗锯齿，就可以得到一个比较软的效果

但是这样所有阴影软的程度都一样了

## PCSS (Percentage-Closer Soft Shadows)

根据PCF我们有了一个把0/1变软的方法，那我们来思考下如何动态决定软的程度

截面后考虑一个线光源、遮挡物和接收面，根据相似三角形

$$ 半影区域=\frac{遮挡物到投射平面}{光源到遮挡物}{面光源区域}
$$

那么一切都有了

具体步骤

1. 在接收点附近搜索遮挡物，得到平均遮挡物深度
2. 根据接收点与遮挡物的距离估计半影宽度，并换算成滤波核大小
3. PCF

参考代码

```hlsl
// HLSL 伪代码。zReceiver 和 shadowMapDepth 都应是同一种线性光空间深度。
Texture2D shadowMap;
SamplerState shadowSampler;
float lightSize; // 面光源的有效宽度

// 1. 遮挡物搜索：寻找邻域内所有遮挡物的平均深度
float FindBlocker(float2 uv, float zReceiver) {
    float blockerSum = 0;
    int numBlockers = 0;
    float searchRegion = ...; // 根据光源大小和距离计算搜索步长

    for (int i = 0; i < BLOCKER_SAMPLES; i++) {
        float shadowMapDepth = shadowMap.SampleLevel(shadowSampler, uv + offset[i] * searchRegion, 0).r;
        if (shadowMapDepth + BIAS < zReceiver) { // 发现遮挡物
            blockerSum += shadowMapDepth;
            numBlockers++;
        }
    }
    
    // 如果没有遮挡，返回 -1 表示完全没阴影
    if (numBlockers == 0) return -1.0;
    return blockerSum / float(numBlockers);
}

// 2. PCSS 主函数
float PCSS(float3 shadowCoords) {
    float zReceiver = shadowCoords.z; // 当前像素的深度
    float2 uv = shadowCoords.xy;

    // STEP 1: 遮挡物搜索
    float avgBlockerDepth = FindBlocker(uv, zReceiver);
    
    // 如果没有遮挡物，直接返回 1.0 (完全无阴影)
    if (avgBlockerDepth == -1.0) return 1.0;

    // STEP 2: 半影估计 (基于相似三角形原理)
    // PenumbraWidth = (d_receiver - d_blocker) * w_light / d_blocker
    float penumbraWidth = (zReceiver - avgBlockerDepth) * lightSize / avgBlockerDepth;

    // STEP 3: 运行 PCF (使用计算出的变宽滤波核)
    float shadow = 0.0;
    float filterSize = penumbraWidth * ...; // 转换到纹理空间步长
    
    for (int i = 0; i < PCF_SAMPLES; i++) {
        float shadowMapDepth = shadowMap.SampleLevel(shadowSampler, uv + offset[i] * filterSize, 0).r;
        shadow += (zReceiver < shadowMapDepth + BIAS) ? 1.0 : 0.0;
    }

    return shadow / float(PCF_SAMPLES);
}
```

PCSS 的 blocker search 与 PCF 都要采样，阴影图越大、核越大，成本越高。实际实现会使用 Poisson disk、分层采样、时域累积或降噪等办法控制成本。

## VSM (Variance Shadow Maps)


### 切比雪夫不等式

PCF 是直接平均比较结果；VSM 则把一个过滤区域内的深度分布压缩成矩，并由矩估计可见度。

VSM 不再只存储深度 $d$，而是存储 $d$ 和 $d^2$。过滤后可得到当前区域深度的均值 $\mu$ 与方差 $\sigma^2$。

对于接收点深度 $t > \mu$，单侧切比雪夫不等式给出“深度样本不小于接收深度”的上界：

$$P(D \ge t) \le \frac{\sigma^2}{\sigma^2 + (t - \mu)^2}$$

这可以作为可见度的近似：当 $t \le \mu$ 时通常直接取 1；当 $t > \mu$ 时使用上式，并在实现中加入最小方差等数值保护。它是一个上界，不是精确概率，因此会产生漏光。

### 高斯分布拟合（扩展近似）

我们也可以假设一个滤波窗口内的深度随机变量 $D$ 服从高斯分布 $\mathcal{N}(\mu, \sigma^2)$。此时，不再使用切比雪夫上界，而是用正态分布的 CDF 直接估计可见度：

$$
V(t) \approx P(D \ge t) = 1 - \Phi\left(\frac{t - \mu}{\max(\sigma, \epsilon)}\right)
$$

其中 $t$ 是接收点深度，$\epsilon$ 用于避免方差趋近于零时除零。标准正态分布的 CDF 为：

$$
\Phi(x) = \frac{1}{2}\left[1 + \operatorname{erf}\left(\frac{x}{\sqrt{2}}\right)\right]
$$

在 Shader 中可以用多项式近似 `erf`，或用查找表近似 CDF。它有时会比切比雪夫上界给出更平滑的过渡，但这是以分布假设换来的：真实深度分布常常是多峰、截断或不连续的，高斯假设会在复杂遮挡处产生错误结果，也不保证减轻漏光。

### 优势

- **可过滤：** 因为存储的是矩，可以直接使用双线性过滤、Mipmapping 或更大范围的预过滤来近似区域分布；普通深度值不能这样线性平均后再直接比较。
    
- **大核查询高效：** SAT（Summed-Area Table）可让矩形区域的矩查询为常数次采样。构建 SAT、本身的显存和带宽仍有成本，效果也会受阴影图分辨率限制。

### 漏光 (Light Bleeding)

阴影暗一点无所谓，但亮一点视觉影响大

这是 VSM 最典型的缺点。切比雪夫不等式给出的只是上界；当一个滤波区域混入相距很远的多个深度层时，上界可能明显高于真实可见度。

> **表现：** 本该完全黑暗的地方会漏出光亮（看起来像半透明的虚影）。可用 light-bleeding reduction、深度 warp（如 EVSM）等方法缓解，但需要在漏光、精度和稳定性之间取舍。

VSSM、SAVSM 等变体会把 VSM 的可过滤特性用于可变宽度的软阴影；它们并不意味着“方差自己决定物理半影”，半影模型与滤波半径仍需额外计算。


| **技术**         | **优点**                       | **缺点**                    |
| -------------- | ---------------------------- | ------------------------- |
| **PCSS**       | 阴影半影区非常真实 (Physically Based) | 采样开销极大，噪声多                |
| **VSM / VSSM** | 易于预过滤，可高效查询较大滤波核          | **漏光问题 (Light Bleeding)**、精度与预处理成本 |
| **PCF**        | 兼容性好，逻辑简单                    | 采样越多越卡，难以实现超大半径软阴影        |

## Moment Shadow Mapping

既然就靠两个统计量的拟合不够精确，那就大力出奇迹😋

如果说 VSM 使用前两个矩来给可见度上界，那么 MSM 使用更多矩（常见为四阶）构造更紧的上界，以减轻漏光。


### 预处理与存储

在生成 Shadow Map 时，MSM 不再只记录 $d$ 和 $d^2$，它通常需要存储四个值：

$$\mathbf{b} = (d, d^2, d^3, d^4)$$

这意味着：

- **存储开销：** 需要一张 4 通道的纹理（通常是 `RGBA32F` 或 `RGBA16F`），而且精度也要求更高
    
- **算力消耗：** 在渲染深度图时，像素着色器需要计算深度的 2、3、4 次方。
    

### 采样与可见度计算 (Sampling)

在采样时，你通过 Mipmap 或 SAT 得到区域内四个矩的均值 $E[d], E[d^2], E[d^3], E[d^4]$。这些矩对应一个截断的 **Hamburger moment problem（汉堡矩问题）**；MSM 通过其推导出的闭式解或数值稳定化实现，计算更紧的可见度上界。

实现中最难的不是“固定做一次 $4 \times 4$ 矩阵运算”，而是处理浮点精度、矩不一致和过滤后的数值稳定性。MSM 仍然可能漏光，但通常比两矩 VSM 更容易得到可接受的结果，代价是更多存储、带宽和计算。

## Distance Field Soft Shadows

**DFSS（Distance Field Soft Shadows）**，即**基于距离场的软阴影**，是现代游戏引擎（如 Unreal Engine 4/5 中的 Mesh Distance Fields）中常用的一种高性能阴影渲染技术

相比于传统的阴影贴图，DFSS 用场景的有符号距离场（SDF）近似遮挡，可为动态面光源生成接触处较硬、远处变软的阴影。它是否更快取决于场景和平台，不是 Shadow Map 的普遍替代品。

SDF可以看以前的文章，或者自己搜，不过应该也算是图形学基础中的基础了

DFSS 通常沿着从着色点到光源的射线进行 sphere tracing：每次读取 SDF，利用“到最近表面的距离”作为安全步长前进。结合锥形追踪（cone tracing）时，光线与遮挡物的余量可以近似半影：

- **步进**：SDF 值给出当前点到最近表面的距离下界；距离足够小可认为命中，距离较大则可一次跳过一段空间。
    
- **半影估计**：射线在传播过程中越接近遮挡物，相对于已走距离的余量越小，越可能处于半影。
    
- **比例关系**：
    
    $$ShadowFactor \approx \min\left(1.0, \frac{k \cdot h}{t}\right)$$
    
    其中 $h$ 是当前到最近表面的距离，$t$ 是已传播距离，$k$ 是软硬度系数。这个式子是常见的启发式近似，不是所有 DFSS 实现通用的物理公式；$k$ 越大通常边缘越锐利。
    

### 优缺点

#### 优点

- **接触硬化自然：** 在合适的距离场与锥形追踪近似下，可得到接触处较硬、远处更软的半影。
    
- **远距离场景有优势：** 对可见像素进行追踪，避免渲染多个远距离 shadow caster 到级联阴影图。实际成本取决于步数、屏幕覆盖、距离场采样和场景结构。


#### 缺点

- **内存占用**：需要预先计算并存储模型的 SDF 数据（通常存储为 3D 纹理或体素数据）。
    
- **精度限制**：对于非常细小的物体（如草叶、细绳），如果 SDF 空间分辨率不够，阴影可能会消失或产生断裂。
    
- **资产限制：** 距离场往往按刚性网格预生成；物体可以移动，但骨骼形变、WPO、透明物体和大规模动态拓扑都不适合直接依赖这种表示。
    

## 4. 与传统 Shadow Map 的对比

|**特性**|**Shadow Map**|**DFSS (Distance Field)**|
|---|---|---|
|**边缘质量**|易受 texel 分辨率影响，通常需 PCF/级联等处理|无 texel aliasing，但受 SDF 分辨率、追踪步数与薄几何限制|
|**软阴影**|PCSS、多重采样或预过滤可实现，成本随方法变化|锥形追踪可近似接触硬化，成本取决于追踪步数|
|**远距离表现**|通常需要 CSM、VSM 或 Virtual Shadow Maps 等策略|可在适合的距离场场景中补充远距离阴影，常与 Shadow Map 混用|
|**适用范围**|通用，支持各类可渲染几何体|需要可用的距离场；更适合刚性、不太薄的网格|

