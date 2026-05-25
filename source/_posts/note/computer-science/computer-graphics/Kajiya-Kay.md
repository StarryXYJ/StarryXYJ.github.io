---
title: Kajiya-Kay 的毛发特化渲染
date: 2026-05-09 23:57:45
tags:
- 笔记
- 计算机图形学
categories:
- [笔记，计算机科学，计算机图形学]
---

好久没更新了喵

讲讲最近尝试的毛发渲染吧

包括真实化的毛发渲染和卡通渲染

## 基本内容

### 标准 Blinn-Phong

它的镜面项是 

$$I_{spec} = k_s I_l \max(0, \mathbf{N} \cdot \mathbf{H})^{\alpha}$$

其中

$$\mathbf{H} = \frac{\mathbf{L} + \mathbf{V}}{|\mathbf{L} + \mathbf{V}|}$$

所谓一个平滑的面，Blinn-Phong 给出了不错的视觉效果，因此我们可以以这个作为基准

![QQ_1779286689454.png](https://free.picui.cn/free/2026/05/20/6a0dc3071ab28.png)

### 基本准备

也就是说 在标准模型（如 Blinn-Phong）中，我们假设反射表面是一个平面，拥有唯一的法线 $N$。高光最强处发生在半角向量 $H$ 与 $N$ 完全重合时，即 $\cos(N, H) = 1$。

注意 N(法线) H(半程/半角向量) V(视角方向向量) L(光源方向向量) 都是单位向量，因此 $\cos(N, H) = N·H$

头发被建模为极细的圆柱体。对于圆柱体上的任意一点，虽然它也有法线 $N$，但由于头发太细，我们在渲染时无法逐像素地捕捉圆柱体表面的法线变化。我们能确定的只有一个稳定的方向：切线 $T$（发丝的延伸方向）

因此，我们的抽象如下：

- 对于平面而言，高光最强处发生在 $\cos(N, H) = 1$ 
- 发丝视为圆柱体，切线方向向量 $T$ 在任意位置确定唯一，N只确定所在平面
- 已知量: V L H T

### 推导

对于圆柱体表面上的任何一点，其法线 $N$ 必须垂直于切线 $T$, 也就是说，圆柱体表面所有可能的法线，构成了一个垂直于 $T$ 的法平面

高光越强的地方，$N·H$越大，也就是$N$和$H$夹角最小

将半角向量 $H$ 分解为两个分量：

- 平行于 $T$ 的分量：$H_{\parallel} = (T \cdot H)T$  
- 垂直于 $T$ 的分量：$H_{\perp} = H - (T \cdot H)T$

显然，圆柱体表面最理想的法线 $N$ 应该指向 $H_{\perp}$ 的方向，因为这样 $N$ 与 $H$ 的夹角 $\theta$ 达到了最小值

$$\cos(N, H) = N·H = H·\frac{(H_{\perp})}{|H_{\perp}|} =\sqrt{1-(T·H)^2}=\sin(T,H) $$


### 一些小问题

#### 只是毛发吗

因为他的基本原理是将模型视为大量细小的圆柱体，那么对于各种丝状物我们都可以用这个算法实现较好的效果

#### 切线从哪来

一般模型会供给，默认切线(Tangent) 是uv的u方向，副切线(Bitangent/Binormal) 为v方向

没有的话也可以在计算集合阶段计算

首先计算三角形的两条边在三维空间中的向量 $E_1, E_2$，以及在 UV 空间中的跨度 $(\Delta u, \Delta v)$：
$$\begin{cases}
E_1 = P_2 - P_1 \\
E_2 = P_3 - P_1 
\end{cases}
, \quad
\begin{cases}
\Delta u_1 = u_2 - u_1, & \Delta v_1 = v_2 - v_1 \\
\Delta u_2 = u_3 - u_1, & \Delta v_2 = v_3 - v_1
\end{cases}$$
三维空间的边向量 $E$ 与切线 $T$、副切线 $B$ 的关系可以表示为
$$\begin{cases}
E_1 = \Delta u_1 T + \Delta v_1 B\\
E_2 = \Delta u_2 T + \Delta v_2 B
\end{cases}$$

也就是
$$\begin{bmatrix} E_{1} \\ E_{2} \end{bmatrix} = \begin{bmatrix} \Delta u_1 & \Delta v_1 \\ \Delta u_2 & \Delta v_2 \end{bmatrix} \begin{bmatrix} T \\ B \end{bmatrix}$$

$$\begin{bmatrix} T \\ B \end{bmatrix} = \frac{1}{\Delta u_1 \Delta v_2 - \Delta u_2 \Delta v_1} \begin{bmatrix} \Delta v_2 & -\Delta v_1 \\ -\Delta u_2 & \Delta u_1 \end{bmatrix} \begin{bmatrix} E_1 \\ E_2 \end{bmatrix}$$



$$\begin{cases}T = \frac{\Delta v_2 E_1 - \Delta v_1 E_2}{\Delta u_1 \Delta v_2 - \Delta u_2 \Delta v_1}\\ \\B = \frac{-\Delta u_2 E_1 + \Delta u_1 E_2}{\Delta u_1 \Delta v_2 - \Delta u_2 \Delta v_1}\end{cases}$$

--- 

当然还有一种方法,就是让美工给一张切线贴图，这可控性显然更高，即使模型面数很低，高光也能随着贴图定义的“发丝流向”产生弯曲和变化。

但，代价嘛

hh

### 更进一步

```shaderlab
half3 KajiyaKaySpecular(half3 T, half3 H,
    half3 primaryColor, half primaryGloss, half primaryShift, half primaryIntensity,
    half3 secondaryColor, half secondaryGloss, half secondaryShift, half secondaryIntensity)
{
    half3 shiftedH1 = normalize(H - T * primaryShift);
    half sinTH1 = sqrt(1.0 - saturate(dot(T, shiftedH1) * dot(T, shiftedH1)));
    half primary = pow(max(HALF_MIN, sinTH1), primaryGloss) * primaryIntensity;

    half3 shiftedH2 = normalize(H - T * secondaryShift);
    half sinTH2 = sqrt(1.0 - saturate(dot(T, shiftedH2) * dot(T, shiftedH2)));
    half secondary = pow(max(HALF_MIN, sinTH2), secondaryGloss) * secondaryIntensity;

    return primaryColor * primary + secondaryColor * secondary;
}
```

这是我目前的KK高光（姑且就这么称呼吧）实现的样例

>  HALF_MIN代表半精度浮点数（half 类型）所能表示的最小正正规化数（Minimum Positive Normalized Value）
>  具体数值是： $2^{-14}$，约等于 0.00006103515625
>  与其相对的是 HALF_MAX： 约等于 65504.0

等等，Gloss、 Intensity类比Blinn-Phong可以理解，Shift、Secondary 又是啥？

让我们直接看看代码

#### Shift

`half3 shiftedH1 = normalize(H - T * primaryShift);`

这是一个被位移后的向量，是$H$ 朝着$T$方向位移一段距离的结果

让H和T向量同一个起点，就是H、T构成平面上的以起点为圆心，半径为1的圆，然后H在这个圆上被运动了一段距离

变换的想法而言，如果不位移，为了达到位移后的效果，需要V、L都朝着这个方向位移

这样的结果就是高光的位置可以被控制（可能确实有点含糊www）

#### Scondary

OK啊，我们现在已经能够轻松自如地利用高光的各向异性，沿着切线方向位移了，相信大家也能够完全理解Secondary的意义了

一段高光看着有点单调，那就两端不同颜色叠加的高光提高各向异性的感觉

肯定不能叠在一起，那么之前我们已经有了切向方向，直接加个位移就行了

主高光是低饱和高亮度的白色高亮，副高光是相对高饱和的类似此表面散射或者溢光的效果，看上去还是很不错的

### 让我们更深入一些

显然如果单纯这样还是只是一团一团的高光，各项异性的效果虽然有，但毛发之类的那种一丝一丝的感觉很微弱，比较好一点的方法可能还是让美术再去做一张贴图，每个发丝位置对应的高光偏移

或者要么就用那种u方向频率比较高，v方向频率很低的噪声，比如拿一个高频噪声设置u方向tiling较大，v方向较小。

不过这种还是有一定的问题，因为模型的任意位置都采样这个噪声，但是由于面数之类的差异会导致高光在一些部分渲染得很诡异，不过，没前期资源的话，能用就行了nya

![4700debb21af94899d87374db855ce0c.png](https://free.picui.cn/free/2026/05/20/6a0dc307256a6.png)

包括这边本身是结合toon的，这种很精细的发丝高光不太符合这种风格，辫子下边还好，后脑勺这一块比较严重

用纯噪声图就没法定制化了唉，二次元角色的头发高光应该还是有一定成片的感觉的，也许用matcap效果也不错？
或者直接定义哪部分会有高光什么的mask之类的，又或者切线剃秃秃可能会更好

直接插值还有个问题就是会导致缩放，有些地方的高光比较细有些地方比较粗

以后看看有没有机会解决解决了
