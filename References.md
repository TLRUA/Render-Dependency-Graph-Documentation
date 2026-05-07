# 参考资料

> 出处：本文档翻译自 staticJPL 的 **Render Dependency Graph Documentation** 项目，原仓库：https://github.com/staticJPL/Render-Dependency-Graph-Documentation 。翻译在尊重原意的基础上，对部分表述做了中文化整理。

为了把这些内容串起来，我花了一些时间阅读引擎源码并搜索相关文章。本节列出我收藏过的参考资料，并尽量按照学习顺序排列。你可以按这些资料的顺序回看本文档，它们会逐步补足理解 Unreal 渲染与图形编程基础所需的知识。

### 图形编程基础参考资料
--------------------------------------------------
**DirectX 管线**
https://www.youtube.com/watchv=pfbWt1BnPIo&list=PLqCJpWy5Fohd3S7ICFXwUomYW0Wv67pDD&index=21&ab_channel=ChiliTomatoNoodle

**DirectX 架构 / Swap Chain**
https://www.youtube.com/watch?v=bMxNN9dO4cI&list=PLqCJpWy5Fohd3S7ICFXwUomYW0Wv67pDD&index=19&ab_channel=ChiliTomatoNoodle

**Vulkan Uniform Buffer**
https://www.youtube.com/watch?v=may_GMkfs5k&t=607s&ab_channel=BrendanGalea

**Vulkan Index Buffer 与 Staging Buffer**
https://www.youtube.com/watch?v=qxuvQVtehII&ab_channel=BrendanGalea

**延迟渲染**
https://www.youtube.com/watch?v=n5OiqJP2f7w&ab_channel=BenAndrew

Shader 基础
---------------------------------------------------------------------------
https://www.youtube.com/watch?v=kfM-yu0iQBk&ab_channel=FreyaHolm%C3%A9r

HLSL 语义
https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-semantics

Unreal Engine Pass 与渲染基础
---------------------------------------------------------------------

Unreal Engine 渲染 Pass
https://unrealartoptimization.github.io/book/profiling/passes/

Unreal Engine 渲染概览（Drawing Policies 已废弃）
https://medium.com/@lordned/unreal-engine-4-rendering-overview-part-1-c47f2da65346

Unreal Engine Vertex Factory
（基本上说明了应该如何创建顶点缓冲；其中会用到一些传给 RDG 的宏。）

https://medium.com/realities-io/creating-a-custom-mesh-component-in-ue4-part-1-an-in-depth-explanation-of-vertex-factories-4a6fd9fd58f2

Unreal Engine RHI
----------------------------------------------
Unreal Engine 渲染系统分析：（中文）timlly-chang
https://www.cnblogs.com/timlly/p/15156626.html#101-%E6%9C%AC%E7%AF%87%E6%A6%82%E8%BF%B0

Riccardo Loggini 的 RDG 架构文章
https://logins.github.io/graphics/2021/05/31/RenderGraphs.html

RDG 分析（中文）
https://blog.csdn.net/qjh5606/article/details/118246059
https://juejin.cn/post/7085216072202190856

使用 RDG 编写 Shader
https://logins.github.io/graphics/2021/03/31/UE4ShadersIntroduction.html

渲染架构
https://ikrima.dev/ue4guide/graphics-development/render-architecture/base-usf-shaders-code-flow/

Epic Games RDG Crash Course
https://epicgames.ent.box.com/s/ul1h44ozs0t2850ug0hrohlzm53kxwrz

Epic Games RDG 文档
https://docs.unrealengine.com/5.1/en-US/render-dependency-graph-in-unreal-engine/

Scene View Extension 类
------------------------------------------
论坛帖子
https://forums.unrealengine.com/t/using-sceneviewextension-to-extend-the-rendering-system/600098

Caius 博客：使用 Scene View Extension 类编写 Global Shader
https://itscai.us/blog/post/ue-view-extensions/
