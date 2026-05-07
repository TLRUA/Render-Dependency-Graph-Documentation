# 图形编程基础

> 出处：本文档翻译自 staticJPL 的 **Render Dependency Graph Documentation** 项目，原仓库：https://github.com/staticJPL/Render-Dependency-Graph-Documentation 。翻译在尊重原意的基础上，对部分表述做了中文化整理。

## 图形管线

要理解本文的内容，需要先对图形渲染管线有基本认识。下面是一张 Vulkan 图形管线示意图。如果你已经熟悉图形管线，可以跳过本节。

![[Unreal Engine Render Dependency Graph/Diagrams/VulkanPipeLine.png]](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/08e9c57045b6a88b7918499961a3c2e2e23e83ff/Diagrams/VulkanPipeLine.png)

不同平台会使用不同类型的图形管线，例如 Vulkan、DirectX 和 OpenGL。它们通常都包含一些共同阶段，例如输入装配器（Input Assembler）、顶点着色器（Vertex Shader）、光栅化（Rasterization）和片元着色器（Fragment Shader，也常被称为像素着色器）。不同 API 可能会在光栅化器前后加入不同阶段，以便针对该平台上的特定操作进行优化。

`Step 1.` 输入装配器会从你指定的缓冲中收集原始顶点数据。结合 HLSL 等 shader 代码，它可以让这些数据绑定到输入语义上；后文会继续说明语义的作用。

`Step 2.` 顶点着色器会针对每个顶点运行，通常用于把顶点位置从模型空间转换到屏幕空间，同时把逐顶点数据继续传递到管线后续阶段。一般来说，在从顶点空间变换到世界空间、本地空间或屏幕空间之前，顶点数据会先按标准裁剪空间（Clip Space）解释。

![[Unreal Engine Render Dependency Graph/Diagrams/Coordinate Systems.png]](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/08e9c57045b6a88b7918499961a3c2e2e23e83ff/Diagrams/Coordinate%20Systems.png)

`Step 3.` 几何着色器会针对每个图元（三角形、线段、点）运行。它可以丢弃图元，也可以输出比输入更多的图元。它与曲面细分着色器有些相似，但更灵活。不过在今天的应用中，几何着色器并不常用，这也再次说明不同 API 的管线设计会有所差异。

`Step 4.` 光栅化阶段会把图元离散化为片元。片元可以理解为图元将要覆盖到帧缓冲上的像素元素。落在屏幕之外的片元会被丢弃；顶点着色器输出的属性会按图元覆盖范围在片元之间插值。通常，被其他图元遮挡在后面的片元也会在这里因为深度测试而被丢弃。深度测试会配合深度缓冲使用，深度缓冲保存了用于剔除判断的信息。按我的理解，光栅化器是这条管线中唯一不可编程的阶段。

`Step 5.` Fragment Shader，也就是 Unreal 中常说的 Pixel Shader。像素着色器会对每个通过光栅化阶段的片元执行，并决定该片元写入哪个帧缓冲，以及写入什么颜色和深度值。它可以使用来自顶点着色器的插值数据，例如纹理坐标和用于光照的法线。如果你不太理解插值，可以想象顶点着色器和像素着色器之间的这个例子。

假设有两个顶点值 A 和 B。两个顶点都有唯一的 2D 屏幕位置和一个 3D 像素颜色。若顶点 A 在屏幕上的颜色为红色，顶点 B 的颜色为蓝色，并且你画一条线连接 A 与 B，那么会得到类似这样的结果：Red(VA)——Purple—-Blue(B)。顶点着色器负责输出连接 A 与 B 的线段数据，然后把结果交给像素着色器；像素着色器会在 A 与 B 之间对颜色进行插值。A 点是纯红色，B 点是纯蓝色，中间就会得到红蓝混合的紫色。

`Step 6.` 在像素着色器把最终图像交给帧缓冲之前，还可能发生一些 API 特有的操作。

### GPU 缓冲

理解缓冲的关键在于：缓冲本质上只是存储在 GPU 内存中的一种资源。这些资源通常在 CPU 侧声明，然后通过内存拷贝传到 GPU，供各种渲染流程使用。在把数据传给 GPU 之前，CPU 侧定义数据时通常需要注意对齐，因此在与图形 API 交互时经常会遇到 padding（填充）和 alignment（对齐）相关的问题。缓冲有很多种，下面列出最常见的几类。

**Vertex Buffer（顶点缓冲）** - 保存定义所有顶点的数据。输入装配器会读取它，并把它绑定到 GPU 管线中指定的顶点着色器。

**Index Buffer（索引缓冲）** - 索引缓冲可以理解为指向顶点缓冲中顶点的索引数组。索引缓冲通常一次读取三个顶点，不过你也可以指定偏移。这样就能从同一个顶点缓冲中组合出多种图元并交给顶点着色器。我们可以定义不同 draw call 来绘制只在屏幕空间中的四边形、三角形或其他图元。

**Command Buffer（命令缓冲）** - 命令缓冲用于记录渲染一帧所需的命令。这些命令包括设置视口、绑定 shader、绑定纹理，以及发出绘制几何体的 draw call。命令缓冲记录完成后，就可以提交给 GPU 执行。

**Depth Buffer（深度缓冲）** - 深度缓冲（也叫 Z-buffer）是 3D 图形中用于保存屏幕上每个像素深度信息的内存缓冲。它用于判断场景中对象的相对深度，确保距离观察者更近的对象绘制在更远对象之上。

**GBuffer** - GBuffer 是 Geometry Buffer 的缩写，它由多个离屏渲染目标组成，用来保存 3D 场景中几何体的各种信息。这些信息可能包括深度、法线、反照率（颜色）、高光以及其他数据。GBuffer 通常用于延迟着色：这种技术会把“捕获场景几何信息”的过程与“对几何应用光照和着色”的过程分开。

**Texture Buffer（纹理缓冲）** - 纹理缓冲是用于保存纹理数据的 GPU 缓冲。纹理是应用在 3D 模型表面上的 2D 或 3D 图像，用来增加视觉细节和真实感。纹理缓冲保存纹理的像素数据，包括颜色、alpha 和法线信息。

### HLSL

在 Unreal Engine 中，shader 代码使用 HLSL 编写，文件扩展名通常为 `.usf` 和 `.ush`。

我喜欢把 HLSL 想象成某种汇编代码：你有输入寄存器和输出寄存器。你会定义顶点着色器的输入，并把它的输出作为像素着色器的输入。常规语义（regular semantics）用于定义管线某个阶段的输入和输出数据含义，例如位置、颜色、法线和纹理坐标。你可以把这些称为“常规语义”，但无论叫什么，都需要通过前面提到的输入装配器把资源（缓冲）绑定到这些语义上。

另一方面，“系统语义”（system semantics）用于定义某个平台或 API 特有的输入/输出数据含义。这些语义会描述数据如何在 GPU 与 API（例如 DirectX 或 OpenGL）之间传递。

例如，你可以使用 `SV_VERTEXID` 语义表示某个输入变量包含顶点 ID。它是 DirectX 特有的系统语义。该语义会与 DirectX 特定的 draw call 相关联，`SV_VERTEXID` 会根据当前正在处理的顶点以及顶点着色器使用的顶点缓冲递增。

下面是后文绘制三角形时将要绑定的 Triangle HLSL 代码。

```cpp
// TriangleVS Binable Name for entry point for a custom vertex shader
void TriangleVS(
in float2 InPosition : ATTRIBUTE0, // First Input Bindable Regular Symantic
in float4 InColor : ATTRIBUTE1, // Second Input Bindable Regular Symantic
out float4 OutPosition : SV_POSITION, // System Symantic Ouput position to Pixel Shader
out float4 OutColor : COLOR0 // System Symantic Output Color to Pixel Shader
)
{
OutPosition = float4(InPosition, 0, 1);
OutColor = InColor;
}
// TrianglePS Bindable entry point for a custom Pixel Shader
void TrianglePS(
in float4 InPosition : SV_POSITION, //System Symantic Input position to Pixel Shader
in float4 InColor : COLOR0, // System Symantic Input Color to Pixel Shader
{
out float4 OutColor : SV_Target0) // System Symantic Render Target, IE. The 2D texture resource Render to.
OutColor = InColor;
}
```
