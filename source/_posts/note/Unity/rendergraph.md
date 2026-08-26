---
title: Unity Render Feature 与 Render Graph
date: 2026-8-26 21:16:33
tags:
- 笔记
- Unity
categories:
- [笔记]
---

Render Feature 相当于对于 SRP 注入了自定义的部分，可以实现比如后处理、中间量记录 等等功能

在 Unity 6 / URP 17 之后，Renderer Feature 的基本结构没变，但 ScriptableRenderPass 默认改为 Render Graph：

```
ScriptableRendererFeature
    ├── Create()
    ├── AddRenderPasses()
    └── ScriptableRenderPass
            └── RecordRenderGraph()
            
```

旧版常用的 OnCameraSetup + Execute + CommandBuffer 属于 Compatibility Mode

那么 Render Graph 有什么好处呢

现在RT之类的资源管理(生命周期、读写并行等等)基本就全权交给 Unity 内部了，而且也有 Render Graph 窗口方便的查看所有RT的使用情况什么的

## 大致架构

目前大概有两个主要需要考虑的类

ScriptableRendererFeature 和 ScriptableRenderPass

> 我这边版本的默认的模板似乎 ScriptableRenderPass 是 ScriptableRendererFeature 的内部类，不过我们把他们当成两个比较独立的模块考虑，或者当成上下级也行，两者没有特别杂糅

重新捋一下就是

ScriptableRendererFeature 负责创建 ScriptableRenderPass 与参数对接

ScriptableRenderPass 负责处理资源和渲染pass

下面我们对这两个类进行比较详细的介绍，利用一个后处理的案例

### ScriptableRendererFeature

它负责

- 暴露可调节参数，和 monobehaviour 差不多
- 初始化创建 ScriptableRenderPass 等内容，Create 方法
- 渲染流程中注入 Pass，AddRenderPasses 方法，按相机每帧调用


```cs
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public sealed class CustomFullScreenFeature : ScriptableRendererFeature
{
    [System.Serializable]
    public sealed class Settings
    {
        public RenderPassEvent renderPassEvent =
            RenderPassEvent.AfterRenderingPostProcessing;

        public Shader shader;

        [Range(0f, 1f)]
        public float intensity = 1f;
    }

    [SerializeField]
    private Settings settings = new();

    private Material material;
    private CustomFullScreenPass renderPass;

    public override void Create()
    {
        if (settings.shader == null)
            return;

        // Inspector 中修改 Feature 时，Create 可能再次调用。
        CoreUtils.Destroy(material);

        material = CoreUtils.CreateEngineMaterial(settings.shader);

        renderPass = new CustomFullScreenPass(material)
        {
            renderPassEvent = settings.renderPassEvent
        };
    }

    public override void AddRenderPasses(
        ScriptableRenderer renderer,
        ref RenderingData renderingData)
    {
        if (renderPass == null || material == null)
            return;

        // 通常避免作用于材质预览、反射探针等特殊相机。
        if (renderingData.cameraData.cameraType != CameraType.Game)
            return;

        material.SetFloat("_Intensity", settings.intensity);

        renderer.EnqueuePass(renderPass);
    }

    protected override void Dispose(bool disposing)
    {
        CoreUtils.Destroy(material);
    }
}
```

### Render Graph Pass

负责处理一个 Render Pass 做了什么

```cs
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.RenderGraphModule;
using UnityEngine.Rendering.RenderGraphModule.Util;
using UnityEngine.Rendering.Universal;

public sealed class CustomFullScreenPass : ScriptableRenderPass
{
    private const string CopyPassName = "Custom Full Screen Copy";
    private const string EffectPassName = "Custom Full Screen Effect";

    private readonly Material material;

    public CustomFullScreenPass(Material material)
    {
        this.material = material;

        // 声明这个 Pass 会读取颜色缓冲。
        ConfigureInput(ScriptableRenderPassInput.Color);
    }

    // 核心方法
    public override void RecordRenderGraph(
        RenderGraph renderGraph,
        ContextContainer frameData)
    {
        if (material == null)
            return;

        // 从Render Pipeline 拿到需要的数据
        UniversalResourceData resourceData =
            frameData.Get<UniversalResourceData>();

        UniversalCameraData cameraData =
            frameData.Get<UniversalCameraData>();

        TextureHandle cameraColor = resourceData.activeColorTexture;

        if (!cameraColor.IsValid())
            return;

        // 创建RT需要 Descriptor 作为初始化参数

        RenderTextureDescriptor descriptor =
            cameraData.cameraTargetDescriptor;

        descriptor.depthBufferBits = 0;
        descriptor.msaaSamples = 1;

        TextureHandle temporaryTexture =
            UniversalRenderer.CreateRenderGraphTexture(
                renderGraph,
                descriptor,
                "_CustomFullScreenTemporary",
                clear: false
            );

        if (!temporaryTexture.IsValid())
            return;

        /*
         * 不能直接安全地：
         *
         * cameraColor -> cameraColor
         *
         * 同一个 RenderGraph Pass 不能对同一个 RT 同时声明 Read 和 Write
         *
         * 所以使用：
         * cameraColor -> temporaryTexture -> cameraColor
         */

        var copyParameters =
            new RenderGraphUtils.BlitMaterialParameters(
                cameraColor, // 输入RT
                temporaryTexture, // 输出RT
                material, // 后处理材质
                shaderPass: 0 // 材质shader的第几个Pass
            );

        renderGraph.AddBlitPass(
            copyParameters,
            EffectPassName
        );

        var returnParameters =
            new RenderGraphUtils.BlitMaterialParameters(
                temporaryTexture,
                cameraColor,
                material: null,
                shaderPass: 0
            );

        renderGraph.AddBlitPass(
            returnParameters,
            CopyPassName
        );
    }
}
```

这里使用 UniversalResourceData.activeColorTexture 获得当前相机颜色，临时纹理由 Render Graph 管理，再通过两个 AddBlitPass 完成：

```
相机颜色
   ↓ 使用效果材质
临时纹理
   ↓ 普通复制
相机颜色
```

### Shader

shader显然不是这篇文章的重点，但是也贴上来吧，保证可以跑通流程，也能一定程度解释前面的Intensity参数啥的

```c
Shader "Hidden/CustomFullScreenEffect"
{
    SubShader
    {
        Tags
        {
            "RenderPipeline" = "UniversalPipeline"
        }

        ZWrite Off
        ZTest Always
        Cull Off

        Pass
        {
            Name "Custom Full Screen Effect"

            HLSLPROGRAM

            #pragma vertex Vert
            #pragma fragment Frag

            #include "Packages/com.unity.render-pipelines.core/Runtime/Utilities/Blit.hlsl"

            float _Intensity;

            half4 Frag(Varyings input) : SV_Target
            {
                UNITY_SETUP_STEREO_EYE_INDEX_POST_VERTEX(input);

                float2 uv = input.texcoord;

                half4 color = SAMPLE_TEXTURE2D_X(
                    _BlitTexture,
                    sampler_LinearClamp,
                    uv
                );

                half luminance = dot(
                    color.rgb,
                    half3(0.2126h, 0.7152h, 0.0722h)
                );

                half3 grayscale = luminance.xxx;

                color.rgb = lerp(
                    color.rgb,
                    grayscale,
                    _Intensity
                );

                return color;
            }

            ENDHLSL
        }
    }
}
```

### 添加到 Render Data

Render Data 配置文件添加 Renderer Feature 能找到我们写的了

## 写法对比

### 旧版

```cs
public override void OnCameraSetup(
    CommandBuffer cmd,
    ref RenderingData renderingData)
{
}

public override void Execute(
    ScriptableRenderContext context,
    ref RenderingData renderingData)
{
    CommandBuffer cmd = CommandBufferPool.Get();

    Blitter.BlitCameraTexture(cmd, source, destination, material, 0);

    context.ExecuteCommandBuffer(cmd);
    CommandBufferPool.Release(cmd);
}
```


### 新版

```cs
public override void RecordRenderGraph(
    RenderGraph renderGraph,
    ContextContainer frameData)
{
    UniversalResourceData resources =
        frameData.Get<UniversalResourceData>();

    TextureHandle source = resources.activeColorTexture;

    renderGraph.AddBlitPass(...);
}
```

### 主要变化

| 旧 API                              | Render Graph                               |
| ---------------------------------- | ------------------------------------------ |
| `Execute()`                        | `RecordRenderGraph()`                      |
| `RTHandle`                         | `TextureHandle`                            |
| 自己获取 `CommandBuffer`               | Render Graph 执行阶段提供                        |
| `renderer.cameraColorTargetHandle` | `UniversalResourceData.activeColorTexture` |
| `RenderingData.cameraData`         | `UniversalCameraData`                      |
| 手动管理临时 RT                          | `CreateRenderGraphTexture`                 |
| `Blitter.BlitCameraTexture`        | `AddBlitPass` 或 Raster Pass                |


RecordRenderGraph() 本身主要负责声明 Pass、输入、输出和资源依赖；真正的 GPU 命令会在 Render Graph 后续执行阶段记录

## 让我们更深入一些

### Pass

注意和Shader的Pass区分开，这里是Render Graph的Pass，类似是一次操作之类的

| Pass 类型            | Context               | 能画 Mesh | Dispatch Compute | RenderGraph 优化 | 常见用途                                     |
| ------------------ | --------------------- | ------- | ---------------- | -------------- | ---------------------------------------- |
| **Raster Pass**    | `RasterGraphContext`  | ✅       | ❌                | ⭐⭐⭐⭐⭐          | 后处理、Forward、Deferred、阴影、GBuffer、全屏 Pass  |
| **Compute Pass**   | `ComputeGraphContext` | ❌       | ✅                | ⭐⭐⭐⭐⭐          | GPU 粒子、FFT、Prefix Sum、体素、GPU Culling、SDF |
| **Unsafe Pass**    | `UnsafeGraphContext`  | ✅       | ✅                | ⭐⭐             | 兼容旧 `CommandBuffer`、插件、遗留渲染代码            |
| **Low Level Pass** | 低层 Context            | 视接口而定   | 视接口而定            | ⭐⭐⭐            | 底层渲染控制、HDRP 或高级定制场景                      |


#### Raster Pass

```cs
using (var builder = renderGraph.AddRasterRenderPass<PassData>(
    "My Pass",
    out var passData))
{
    builder.UseTexture(...);
    builder.SetRenderAttachment(...);

    builder.SetRenderFunc((PassData data, RasterGraphContext ctx) =>
    {
        // DrawMesh
        // DrawRendererList
        // Blitter
        // SetGlobalTexture
    });
}
```

##### Blit Pass

最上面的案例使用的

简化的 Raster Pass，语法糖之类的？后处理什么的经常用

不过底层是同类型的 Pass

`renderGraph.AddBlitPass(...)`

```cs
using (var builder =
    renderGraph.AddRasterRenderPass<PassData>(
        "Blit",
        out var passData))
{
    builder.UseTexture(source, AccessFlags.Read);

    builder.SetRenderAttachment(
        destination,
        0,
        AccessFlags.Write);

    builder.SetRenderFunc((data, ctx) =>
    {
        Blitter.BlitTexture(
            ctx.cmd,
            data.source,
            ...);
    });
}
```

#### Compute Pass

用于 Dispatch Compute Shader

```cs
using(var builder =
    renderGraph.AddComputePass<PassData>(
        "Compute",
        out var passData))
{
    builder.UseTexture(texture, AccessFlags.ReadWrite);

    builder.SetRenderFunc((PassData data, ComputeGraphContext ctx)=>
    {
        ctx.cmd.DispatchCompute(...);
    });
}
```

#### Unsafe Pass

RenderGraph 不再帮你保证状态，直接用 CommandBuffer

一般用于把旧代码直接搬过来

#### LowLevel Pass

LowLevel Pass 的基本结构与 Raster Pass 类似，但这里 Render Graph 只负责资源生命周期和依赖；RenderTarget、Viewport、Clear、Draw、Dispatch 等状态要由你自己设置。

```cs
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.RenderGraphModule;

class PassData
{
    public TextureHandle source;
    public TextureHandle destination;
    public Material material;
}

static void AddLowLevelPass(
    RenderGraph renderGraph,
    TextureHandle source,
    TextureHandle destination,
    Material material)
{
    using var builder =
        renderGraph.AddLowLevelPass<PassData>(
            "My LowLevel Pass",
            out var passData);

    passData.source = source;
    passData.destination = destination;
    passData.material = material;

    // 仍然必须向 Render Graph 声明资源依赖
    builder.UseTexture(source, AccessFlags.Read);
    builder.UseTexture(destination, AccessFlags.Write);

    builder.SetRenderFunc(
        static (PassData data, LowLevelGraphContext context) =>
        {
            // LowLevel Pass 不会自动绑定 RT
            context.cmd.SetRenderTarget(data.destination);

            // 自己设置 Viewport、Clear 等状态
            context.cmd.ClearRenderTarget(
                clearDepth: false,
                clearColor: true,
                backgroundColor: Color.black);

            // 绘制全屏三角形
            context.cmd.DrawProcedural(
                Matrix4x4.identity,
                data.material,
                shaderPass: 0,
                MeshTopology.Triangles,
                vertexCount: 3,
                instanceCount: 1);
        });
}
```

### 中间值

`ContextContainer` 是当前帧数据的集合。


| 方法                 | 作用             |
| ------------------ | -------------- |
| `Get<T>()`         | 获取已经存在的帧数据     |
| `GetOrCreate<T>()` | 获取数据，不存在则创建    |
| `Contains<T>()`    | 检查是否存在指定数据     |
| `Dispose()`        | 清理容器内容，通常不手动调用 |

比如

```cs
UniversalResourceData resourceData =
    frameData.Get<UniversalResourceData>();

UniversalCameraData cameraData =
    frameData.Get<UniversalCameraData>();

UniversalRenderingData renderingData =
    frameData.Get<UniversalRenderingData>();

UniversalLightData lightData =
    frameData.Get<UniversalLightData>();
```

自定义数据，比如这里我们传递纹理句柄

```cs
public sealed class MyFrameData : ContextItem
{
    public TextureHandle outlineTexture;

    public override void Reset()
    {
        outlineTexture = TextureHandle.nullHandle;
    }
}
```

写入：

```cs
MyFrameData data = frameData.GetOrCreate<MyFrameData>();
data.outlineTexture = texture;
```

另一个 Pass 读取：

```cs
MyFrameData data = frameData.Get<MyFrameData>();
TextureHandle texture = data.outlineTexture;
```

### 常用预定义中间值

#### UniversalResourceData

| 成员                         | 含义                    | 常见用途      |
| -------------------------- | --------------------- | --------- |
| `activeColorTexture`       | 当前活动颜色纹理              | 全屏后处理输入   |
| `activeDepthTexture`       | 当前活动深度附件              | 深度测试或深度读取 |
| `cameraColor`              | 相机颜色目标                | 访问相机颜色    |
| `cameraDepth`              | 相机深度目标                | 访问相机深度    |
| `cameraOpaqueTexture`      | URP Opaque Texture    | 折射、扭曲     |
| `cameraDepthTexture`       | 可采样的相机深度纹理            | 深度重建      |
| `cameraNormalsTexture`     | 相机法线纹理                | 描边、SSAO   |
| `motionVectorColor`        | 运动矢量颜色纹理              | TAA、运动模糊  |
| `motionVectorDepth`        | 运动矢量对应深度              | 运动矢量处理    |
| `gbuffer`                  | Deferred 的 GBuffer 数组 | 自定义延迟效果   |
| `mainShadowsTexture`       | 主光源阴影纹理               | 自定义阴影采样   |
| `additionalShadowsTexture` | 附加光阴影纹理               | 多光源阴影     |
| `ssaoTexture`              | SSAO 结果               | 自定义合成     |
| `isActiveTargetBackBuffer` | 当前目标是否直接是后备缓冲         | 判断能否安全采样  |


#### UniversalCameraData

| 成员                       | 含义                           |
| ------------------------ | ---------------------------- |
| `camera`                 | 当前 Unity `Camera`            |
| `cameraType`             | Game、SceneView、Preview 等相机类型 |
| `cameraTargetDescriptor` | 相机目标纹理描述                     |
| `renderType`             | Base Camera 或 Overlay Camera |
| `resolveFinalTarget`     | 当前相机是否输出最终结果                 |
| `isSceneViewCamera`      | 是否 Scene View                |
| `isPreviewCamera`        | 是否 Preview Camera            |
| `isHdrEnabled`           | 是否启用 HDR                     |
| `postProcessEnabled`     | 是否开启后处理                      |
| `requiresDepthTexture`   | 是否需要深度纹理                     |
| `requiresOpaqueTexture`  | 是否需要 Opaque Texture          |
| `renderer`               | 当前 `ScriptableRenderer`      |
| `xr`                     | XR 相关信息                      |
| `defaultOpaqueSortFlags` | 默认不透明排序方式                    |
| `cameraStack`            | Camera Stack 信息              |
| `worldSpaceCameraPos`    | 世界空间相机位置                     |

创建中间纹理时通常取：

```cs
RenderTextureDescriptor descriptor =
    cameraData.cameraTargetDescriptor;

descriptor.depthBufferBits = 0;
descriptor.msaaSamples = 1;
```

#### UniversalRenderingData

这个类主要用于绘制场景中的 Renderer，而不是单纯做全屏 Blit。

| 成员                           | 含义                    |
| ---------------------------- | --------------------- |
| `cullResults`                | 相机剔除结果                |
| `supportsDynamicBatching`    | 是否支持 Dynamic Batching |
| `perObjectData`              | Shader 需要的逐物体数据       |
| `renderingMode`              | 当前渲染模式                |
| `stencilLodCrossFadeEnabled` | LOD Cross Fade 模板支持   |

最重要的是：

`renderingData.cullResults`

它会用于创建 RendererList：

```cs
RendererListParams rendererListParams =
    new RendererListParams(
        renderingData.cullResults,
        drawingSettings,
        filteringSettings
    );
```

Renderer List 可以理解为 根据剔除结果、Shader Pass、Layer、RenderQueue 和排序规则筛选出来的一批待绘制物体

```cs
private sealed class PassData
{
    public RendererListHandle rendererList;
}

using IRasterRenderGraphBuilder builder =
    renderGraph.AddRasterRenderPass<PassData>(
        "Draw Objects",
        out PassData passData
    );

passData.rendererList =
    renderGraph.CreateRendererList(rendererListParams);

builder.UseRendererList(passData.rendererList);
builder.SetRenderAttachment(color, 0, AccessFlags.Write);
builder.SetRenderAttachmentDepth(depth, AccessFlags.Write);

builder.SetRenderFunc(
    static (PassData data, RasterGraphContext context) =>
    {
        context.cmd.DrawRendererList(data.rendererList);
    }
);
```

#### 纹理相关

TextureHandle、TextureDesc 是Render Graph 特有的系统，其中 TextureHandle 只是一个标签，让渲染管线去编排之类的，运行时会创建RTHandle，但只有RTHandle会对应到真实的显存数据等等

##### TextureHandle

| 成员 / 用法                       | 作用                    |
| ----------------------------- | --------------------- |
| `IsValid()`                   | 判断句柄是否有效              |
| `nullHandle`                  | 空纹理句柄                 |
| `renderGraph.CreateTexture()` | 创建 Render Graph 管理的纹理 |
| `renderGraph.ImportTexture()` | 导入外部 `RTHandle`       |

##### TextureDesc

| 成员                  | 作用                           |
| ------------------- | ---------------------------- |
| `width` / `height`  | 固定宽高                         |
| `scale`             | 相对相机尺寸缩放                     |
| `colorFormat`       | 颜色格式                         |
| `depthBufferBits`   | 深度位数                         |
| `msaaSamples`       | MSAA 样本数                     |
| `clearBuffer`       | 第一次使用前是否清理                   |
| `clearColor`        | 清理颜色                         |
| `filterMode`        | Point / Bilinear / Trilinear |
| `wrapMode`          | Clamp / Repeat               |
| `enableRandomWrite` | 是否允许 Compute Shader UAV 写入   |
| `useMipMap`         | 是否创建 Mipmap                  |
| `autoGenerateMips`  | 是否自动生成 Mipmap                |
| `dimension`         | 2D、2DArray、3D、Cube           |
| `slices`            | 数组层数                         |
| `name`              | 调试名称                         |

##### RenderTextureDescriptor

这是 Unity **原有**的纹理描述结构，不是 Render Graph 专属，但在 URP 中非常常用。

URP 提供了快捷创建方法：

```
TextureHandle temp =
    UniversalRenderer.CreateRenderGraphTexture(
        renderGraph,
        descriptor,
        "_TemporaryColor",
        clear: false
    );
```

 | 成员                  | 作用         |
| ------------------- | ---------- |
| `width` / `height`  | 分辨率        |
| `graphicsFormat`    | 像素格式       |
| `depthBufferBits`   | 深度位数       |
| `msaaSamples`       | MSAA       |
| `volumeDepth`       | 纹理层数       |
| `dimension`         | 纹理维度       |
| `enableRandomWrite` | Compute 写入 |
| `useMipMap`         | Mipmap     |
| `sRGB`              | 是否 sRGB    |
| `vrUsage`           | XR 用途      |

### Volume

自定义后处理通常还会配合 Volume 系统

就可以放到 场景的volume组件内 override 了

```cs
[System.Serializable]
[VolumeComponentMenu("Custom/My Effect")]
public sealed class MyVolumeComponent :
    VolumeComponent,
    IPostProcessComponent
{
    public ClampedFloatParameter intensity =
        new ClampedFloatParameter(0f, 0f, 1f);

    public bool IsActive()
    {
        return intensity.value > 0f;
    }
}
```

| 类                        | 作用                 |
| ------------------------ | ------------------ |
| `VolumeComponent`        | 自定义 Volume 组件基类    |
| `IPostProcessComponent`  | 标识组件是否为后处理         |
| `VolumeManager`          | 访问当前 Volume Stack  |
| `VolumeStack`            | 当前相机混合后的 Volume 结果 |
| `VolumeParameter<T>`     | Volume 参数基类        |
| `ClampedFloatParameter`  | 限制范围的 float        |
| `FloatParameter`         | float 参数           |
| `BoolParameter`          | bool 参数            |
| `ColorParameter`         | Color 参数           |
| `TextureParameter`       | Texture 参数         |
| `Vector2Parameter`       | Vector2 参数         |
| `Vector3Parameter`       | Vector3 参数         |
| `NoInterpFloatParameter` | 不插值的 float         |
