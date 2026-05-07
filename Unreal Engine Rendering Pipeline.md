# Unreal Engine 渲染管线

> 出处：本文档翻译自 staticJPL 的 **Render Dependency Graph Documentation** 项目，原仓库：https://github.com/staticJPL/Render-Dependency-Graph-Documentation 。翻译在尊重原意的基础上，对部分表述做了中文化整理。

在进入 Unreal 的渲染管线之前，需要先说明渲染 Pass 和延迟渲染的概念。

`Rendering Pass` 是一组在 GPU 上执行的 draw call，数量可以是一个，也可以是多个。通常会把许多 draw call 组织到一起，以保证执行顺序正确。这是因为前一个 pass 的输出可能会被后续 pass 当作输入使用。

`Deferred rendering（延迟渲染）` 是 Unreal 默认使用的一种方法，它会把光照和材质放到单独的 pass 中处理。这个单独的 pass 会等待 base pass 先积累不透明度、高光、漫反射、法线等关键信息。下面的例子展示了延迟渲染的工作方式。

![[Unreal Engine Render Dependency Graph/Diagrams/DeferredRender.png]](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/a49756d90f6362b9a8ab10e4ac16edddac530e03/Diagrams/DeferredRender.png)

Unreal 并不是在每个像素被光栅化时立即计算光照和着色，而是先使用延迟渲染把场景几何信息写入“离屏”的 GBuffer；随后在第二个 pass 中使用这些信息给场景应用光照和着色。这样做的优势是能更高效地使用 GPU。把几何信息与光照/着色分离后，GPU 可以同时处理大量光源和效果，从而提升性能。此外，它也让动态光照和复杂光照设置更加灵活。

## Unreal Engine 中的 Pass 顺序

这些 Pass 可能会随着引擎版本或项目配置而变化，但大体顺序如下。我建议下载 RenderDoc 并挂接到引擎上亲自查看。

**Base Pass**
- 把不透明或 Masked 材质的最终属性渲染到 GBuffer。
- 读取静态光照并保存到 GBuffer。
- 应用 DBuffer decals。
- 应用雾效。
- 计算最终速度（来自打包后的 3D velocity）。
- 在前向渲染器中：处理动态光照。

在 Deferred 模式下，Base Pass 会像前面说明的那样把材质属性保存进 GBuffer，并把光照计算留到后续 pass。

**Geometry Passes（几何 Pass）**

几何 Pass 会绘制网格，并在光照之前对它们进行组织和优先级处理。

#### PrePass

![[Unreal Engine Render Dependency Graph/Diagrams/PrePass.png]](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/a49756d90f6362b9a8ab10e4ac16edddac530e03/Diagrams/PrePass.png)

PrePass 会提前渲染深度 Z-Buffer，用于根据透明度优化网格处理；它也会剔除被其他网格遮挡的网格，从而避免渲染不可见物体。

**HZB**
- 生成分层 Z-Buffer。

HZB 会用于遮挡剔除，也会被屏幕空间环境光遮蔽和屏幕空间反射等技术使用。

**Render Velocities**
- 保存每个顶点的速度，后续会被运动模糊和时间抗锯齿使用。

`Velocity` 是一个缓冲，用来测量每个移动顶点的速度并写入运动模糊速度缓冲。Velocity Buffer 会比较当前帧与上一帧的差异来生成遮罩。在 `Doom 2016` 中，他们利用这种遮罩只渲染场景中非静态的网格，从而优化下一帧中发生移动的网格渲染。

**Lighting Pass**

这是整帧中最“重”的部分，尤其是在有大量动态光源和投影光源时。

**Direct Lighting**
- 前向着色中的优化光照。

**Non-Shadowed Lights**
- 延迟渲染中不投射阴影的光源。

**Shadowed Lights**
- 会投射动态阴影的光源。

**Shadow Depths**
- 为投射阴影的光源生成深度图。

![[Unreal Engine Render Dependency Graph/Diagrams/ShadowProjection.png]](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/a49756d90f6362b9a8ab10e4ac16edddac530e03/Diagrams/ShadowProjection.png)

**Shadow Projection**
- 最终渲染阴影。

**Indirect Lighting**
![[Unreal Engine Render Dependency Graph/Diagrams/IndirectLighting.png]](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/a49756d90f6362b9a8ab10e4ac16edddac530e03/Diagrams/IndirectLighting.png)
- 屏幕空间环境光遮蔽。
- Decals（非 Buffer 类型）。

**Composition After Lighting**
- 处理次表面散射。

**Translucency and lighting**
- 渲染半透明材质。
- 处理使用 surface forward shading 的材质光照。

**Reflections**
- 读取并混合 Reflection Capture Actor 的结果，写入全屏反射缓冲。
  
**Screen Space Reflections**
- 实时动态反射。
- 在后处理阶段使用屏幕空间光线追踪技术完成。

**Post Processing**

后处理是渲染管线的最后一个 pass，也是后文绘制三角形的位置。

- 景深（BokehDOFRecombine）
- 时间抗锯齿（TemporalAA）
- 读取速度值（VelocityFlatten）
- 运动模糊（MotionBlur）
- 自动曝光（PostProcessEyeAdaptation）
- 色调映射（Tonemapper）
- 从渲染分辨率上采样到显示分辨率（PostProcessUpscale）
