---
title: SDF介绍与贴花实现
date: 2026-05-25 18:20:45
tags:
- 笔记
- 计算机图形学
categories:
- [笔记，计算机科学，计算机图形学]
---

## SDF定义

SDF（有向距离场/有向距离函数） 是隐式几何体的表达方式之一

> 隐式几何体是通过数学函数或方程来描述的几何形状，不直接记录具体的顶点或面片坐标

被定义为 空间中任意一点 到物体表面的**最短有向距离**

对于空间中的任意一点 $p$：
****
- **$f(p) > 0$**：点 $p$ 在几何体**外部**。
    
- **$f(p) = 0$**：点 $p$ 正好在几何体**表面**。
    
- **$f(p) < 0$**：点 $p$ 在几何体**内部**。
    
- **大小**：点 $p$ 到几何体表面的**最短绝对距离**。

听上去感觉还可以，想一想感觉有点抽象？

一下是一些案例

```c
float sdCircle(float2 p, float r)
{
    return length(p) - r;
}
```

```c
float sdBox3D(float3 p, float3 b)
{
    float3 q = abs(p) - b;
    return length(max(q, 0.0)) + min(max(q.x, max(q.y, q.z)), 0.0);
}
```

- `b`：立方体的**半长宽高**（比如标准的 `1x1x1` 的 Cube，`b = float3(0.5, 0.5, 0.5)`）。

大概这样

### sdf对应的图形是唯一确定的吗(严格数学？)

要详细证明 SDF (Signed Distance Function) 与 几何图形（闭合集） 之间的唯一确定性，我们需要从数学定义出发。

这里的唯一性包含两个维度：

1. 给定图形，其 SDF 是否唯一？（存在性与唯一性）

2. 给定满足程函方程的函数，其对应的图形是否唯一？（等值面唯一性）

设 $\Omega \subset \mathbb{R}^n$ 是一个非空开集，其边界 $\Gamma = \partial\Omega$ 是一个闭合流形

点 $x$ 到集合 $\Gamma$ 的 距离函数 (Distance Function) 定义为：

$$d_\Gamma(x) = \inf_{y \in \Gamma} \|x - y\|$$

有符号距离函数 (SDF) 则定义为：

$$f(x) = \begin{cases} -d_\Gamma(x) & x \in \Omega \\ 0 & x \in \Gamma \\ d_\Gamma(x) & x \in \mathbb{R}^n \setminus \bar{\Omega} \end{cases}
$$


#### 证明：图形 $\Gamma$ 唯一确定函数 $f(x)$


1. **距离的唯一性**：对于欧几里得空间中任意一点 $x$ 和非空闭集 $\Gamma$，点到集合的距离是通过 $\inf$（下确界）定义的。根据实数的性质，一个集合的下确界如果存在，则是唯一的。
    
2. **符号的唯一性**：由于 $\Omega$ 是确定的，点 $x$ 的位置只有三种可能：在 $\Omega$ 内部、在边界 $\Gamma$ 上、或在外部。这由集合的拓扑结构唯一确定。
    
3. **结论**：既然距离数值唯一，符号判定唯一，那么函数值 $f(x)$ 在空间中每一点的值都是唯一确定的。
    

#### 证明：函数 $f(x)$ 唯一确定图形 $\Gamma$


根据 SDF 的定义，集合 $\Gamma$（图形的边界）被定义为函数 $f$ 的 **零等值面 (Zero Level Set)**：

$$\Gamma = \{ x \in \mathbb{R}^n \mid f(x) = 0 \}$$

如果已知函数 $f(x)$，那么所有满足 $f(x)=0$ 的点集在空间中是唯一确定的。因此，只要 SDF 函数确定，它所描述的几何边界 $\Gamma$ 也就随之确定。

### 程函方程 (Eikonal Equation) 的解

AI说的，整理一下，说实话我也没怎么学过这些，但是看上去很高端

程函方程最标准的形式如下：
$$|\nabla u(x)| = \frac{1}{v(x)}, \quad x \in \Omega$$
其中：$u(x)$ 是我们要求解的标量场（在图形学中通常是距离值）。$\nabla u(x)$ 是该场的梯度。$v(x)$ 是波在该位置的传播速度。

在计算机图形学的 SDF 构建中， $v(x) = 1$，于是方程简化为：
$$\|\nabla f(x)\| = 1$$

由于 $\|\nabla f\| = 1$，它赋予了 SDF 两个非常实用的工程属性：
- 法线提取：表面法线 $\vec{n} = \nabla f$。
- Ray Marching 加速：因为知道当前点到表面的最小距离，光线可以一次性跳过这段距离而不穿透物体。

## 采样

知道了 sdf是什么，那我们要怎么把它转化成像素呢

对于模型，主要有以下几种采样方式

### 射线步进采样

步骤：
- 从摄像机发射射线，当前点设为 $p_0$。
- 采样当前点的 SDF 值 $d = f(p_0)$。
- 将射线向前推进距离 $d$。
- 重复上述过程，直到 $d$ 小于某个极小阈值 $\epsilon$（视为撞击表面）或超过最大距离（视为背景）。

##### 优势与特性

- 计算效率在部分情况相对快
- 类似矢量图，可以渲染分形等无限复杂的几何体。
- 通过 min(d1, d2) 实现并集，max(d1, -d2) 实现差集等，无需处理复杂的网格布尔运算。

##### 不足

- 因为采样依赖于步长、最大距离、撞击表面阈值，因此为了表达一些几何体需要大量的步进迭代

### 空间网格采样

由于我们无法存储空间中无限个点，因此只在规则网格的节点上存储 SDF 值。
- 离散化：在每个网格点 $\mathbf{x}_{i,j,k}$ 计算 $f(\mathbf{x}_{i,j,k})$ 并存储
- 重建（重采样）：当需要查询非节点位置的点 $p$ 时，使用三线性插值（Trilinear Interpolation）
  - 插值后的函数值 $\tilde{f}(p)$ 是对原始程函方程解的近似。

#### 等值面提取 (Isosurface Extraction)

如果你需要把 SDF 变成能在常规游戏引擎（如 Unity/UE）中渲染的 Mesh，就需要对网格进行采样：

- Marching Cubes (MC)：遍历每个立方体单元，根据 8 个顶点的正负号状态，从查找表中提取三角形面片。

- Dual Contouring：优化，能够更好地保留直角边缘和特征点（Feature Points）。


|**特性**|**射线步进采样 (Ray Marching)**|**空间网格采样 (Grid Sampling)**|
|---|---|---|
|**表现形式**|实时计算的标量场（隐式）|存储在内存中的体素阵列（离散）|
|**内存开销**|极低（仅存储数学公式或少量常量）|高（随分辨率 $N^3$ 增长）|
|**渲染性能**|随像素数量和场景复杂度波动|渲染 Mesh 极快，但预计算/更新慢|
|**精度**|理论上无限精确（分析解）|受限于网格分辨率，存在走样|

## 来点简单的

前面又是光线步进采样又是程函方程，但实际上看标题，我们就能得知一个简单的应用

贴花

对于贴花的渲染，由于是附着在其他几何体表面的，而我们一般渲染时可以拿到深度图，(渲染自己之前所有物体的位置)，那么实际上一切就很简单了，我们不再需要去采样，而是判断屏幕空间像素和几何体的关系

先看案例

```hlsl
Shader "p4/SDFCircleDecal"
{
    Properties
    {
        _Radius ("Ring Radius", Range(0.0, 0.5)) = 0.4
        _Thickness ("Ring Thickness", Range(0.0, 0.1)) = 0.02
        _RingColor ("Ring Color", Color) = (0, 1, 0, 1)
    }
    SubShader
    {
        Tags 
        { 
            "RenderPipeline" = "UniversalPipeline"
            "Queue" = "Transparent" 
            "RenderType" = "Transparent" 
        }
        
        LOD 100
        ZWrite Off
        ZTest Off
        Cull Off
        Blend SrcAlpha OneMinusSrcAlpha

        Pass
        {
            Name "SDFDecalPass"
            
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/DeclareDepthTexture.hlsl"

            struct Attributes
            {
                float4 positionOS   : POSITION;
            };

            struct Varyings
            {
                float4 positionCS   : SV_POSITION;
                float4 screenPos    : TEXCOORD0;
            };

            float _Radius;
            float _Thickness;
            half4 _RingColor;

            float sdfCircle(float2 p, float r)
            {
                return abs(length(p) - r);
            }

            // 圆环的 SDF 公式
            float sdfRing(float2 p, float r, float t)
            {
                return sdfCircle(p,r) - (t * 0.5);
            }

            Varyings vert(Attributes input)
            {
                Varyings output;
                VertexPositionInputs vertexInput = GetVertexPositionInputs(input.positionOS.xyz);
                output.positionCS = vertexInput.positionCS;
                output.screenPos = ComputeScreenPos(output.positionCS);
                return output;
            }

            half4 frag(Varyings input) : SV_Target
            {
                // 1. 深度重建
                float2 screenUV = input.screenPos.xy / input.screenPos.w;
                float depth = SampleSceneDepth(screenUV);
                float3 worldPos = ComputeWorldSpacePosition(screenUV, depth, UNITY_MATRIX_I_VP);
                float3 localPos = mul(GetWorldToObjectMatrix(), float4(worldPos, 1.0)).xyz;

                // 2. 裁剪
                clip(0.5 - abs(localPos));

                // 3. 取 localPos 的 XZ 轴作为 2D 绘图平面
                // 此时中心点本来就是 (0,0)，范围在 [-0.5, 0.5]
                float2 p = localPos.xz;

                // 4. 计算当前片元到圆环的数学距离
                float d = sdfRing(p, _Radius, _Thickness);

                // 5. 硬件偏导数抗锯齿 (fwidth)
                // 确保线框无论在多远、多么倾斜的地面上，边缘过渡永远只有 1 像素宽，绝不模糊
                float delta = fwidth(d);
                float alpha = smoothstep(delta, -delta, d);

                // 6. 混合颜色输出
                half4 finalColor = _RingColor;
                finalColor.a *= alpha;

                return finalColor;
            }
            ENDHLSL
        }
    }
}
```

相信大家很容易就能理解

那么这期就到这了

