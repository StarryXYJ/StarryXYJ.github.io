---
title: 更复杂的算子拥有更高的性能
date: 2026-07-31 17:49:45
tags:
- 笔记
- 优化
categories:
- [笔记，计算机科学]
---

这类算子通常称为 **硬件内建数学指令（Hardware Intrinsics）** 或 **Fast Math Functions**。它们很多看起来像复杂数学运算，但 GPU（甚至 CPU SIMD）往往都有专门的硬件指令，因此比调用普通数学库快得多

下面按照图形学中最常见的整理。

| 算子             | 数学意义         | 常见硬件实现                 | 相比普通实现       | 图形学用途        |
| -------------- | ------------ | ---------------------- | ------------ | ------------ |
| `rcp(x)`       | 1/x          | Reciprocal 指令          | 比除法快2~10倍    | 透视除法、UV插值    |
| `rsqrt(x)`     | 1/√x         | Reciprocal Square Root | 比 sqrt+除法快很多 | normalize、光照 |
| `sqrt(x)`      | √x           | 常由 rsqrt+乘法实现          | 比 rsqrt慢     | 距离计算         |
| `dot(a,b)`     | 点积           | SIMD DP 指令             | 一条或少量指令      | 光照、投影        |
| `cross(a,b)`   | 叉积           | SIMD shuffle+FMA       | 极快           | 法线           |
| `mad(a,b,c)`   | a×b+c        | FMA                    | 一条指令完成       | 插值           |
| `fma(a,b,c)`   | a×b+c        | Fused Multiply Add     | 精度更高且更快      | 几乎所有Shader   |
| `lerp(a,b,t)`  | a+t(b-a)     | MAD/FMA展开              | 极快           | 插值           |
| `clamp(x,a,b)` | min(max())   | min/max指令              | 无分支          | 饱和           |
| `saturate(x)`  | clamp(x,0,1) | 专用SAT                  | 比clamp更快     | HDR、颜色       |
| `min/max`      | 最值           | 单指令                    | 无分支          | 深度比较         |
| `abs(x)`       | 绝对值          | 修改符号位                  | 几乎零成本        | 法线           |
| `sign(x)`      | 符号           | 比较指令                   | 很快           | SDF          |
| `step(edge,x)` | 阶跃函数         | CMP                    | 无if          | 蒙版           |
| `smoothstep`   | 平滑阶跃         | 多个FMA                  | 很快           | 渐变           |
| `frac(x)`      | 小数部分         | floor+sub硬件            | 快            | Noise        |
| `floor/ceil`   | 取整           | Convert指令              | 快            | Tile         |
| `round`        | 四舍五入         | Convert                | 快            | 像素定位         |
| `trunc`        | 截断           | Convert                | 快            | 数据处理         |

---

## 三角函数

现代GPU都有 SFU（Special Function Unit）。

| 算子       | 是否专用硬件 | 备注         |
| -------- | ------ | ---------- |
| `sin`    | ✔      | SFU        |
| `cos`    | ✔      | SFU        |
| `sincos` | ✔      | 同时计算通常更快   |
| `tan`    | ✖      | 一般 sin/cos |
| `asin`   | 部分     | 近似实现       |
| `acos`   | 部分     | 较慢         |
| `atan`   | 部分     | 较慢         |
| `atan2`  | 部分     | 更慢         |

一般速度

```
sin ≈ cos
    <
sincos
    <<
atan
acos
asin
atan2
```

---

## 指数对数

GPU同样提供快速近似。

| 算子         | 硬件支持        | 用途   |
| ---------- | ----------- | ---- |
| `exp2(x)`  | ✔           | 2^x  |
| `log2(x)`  | ✔           | log₂ |
| `exp(x)`   | exp2换底      | 指数   |
| `log(x)`   | log2换底      | 对数   |
| `pow(x,y)` | log+mul+exp | 相对较慢 |

一般

```
exp2
log2
    <
exp
log
    <<
pow
```

---

## 位运算

| 算子             | 复杂度     | 用途          |
| -------------- | ------- | ----------- |
| `countbits`    | POPCNT  | Cluster     |
| `firstbitlow`  | BSF     | Bitmask     |
| `firstbithigh` | BSR     | LOD         |
| `reversebits`  | BITREV  | Morton      |
| `asuint`       | BitCast | reinterpret |
| `asfloat`      | BitCast | reinterpret |
| `pack/unpack`  | Pack指令  | GBuffer     |


---

## 插值相关

| 算子                    | 本质                |
| --------------------- | ----------------- |
| barycentric           | FMA               |
| interpolateAtSample   | 硬件插值              |
| interpolateAtCentroid | 硬件                |
| ddx                   | Quad差分            |
| ddy                   | Quad差分            |
| fwidth                | abs(ddx)+abs(ddy) |



---

## 向量归一化

GPU几乎都会优化成

```cpp
normalize(v)

↓

len2 = dot(v,v)
inv = rsqrt(len2)
v*=inv
```


---

### 经验性的性能层级（从快到慢）

```text
位运算
≈ abs
≈ min/max
≈ saturate
≈ FMA
≈ dot
≈ rcp
≈ rsqrt
< exp2/log2
< sin/cos
< sqrt
< exp/log
<< pow
<< atan/acos/asin
<< atan2
```

需要注意的是，**现代 GPU 不同架构（NVIDIA Ada/Blackwell、AMD RDNA、Apple GPU 等）的具体延迟和吞吐会有所差异**，编译器还会进行大量优化，因此这个层级更适合作为 Shader 编写和性能优化时的经验法则，而不是绝对的周期数。
