# Scene View Extension 三角形 Shader

> 出处：本文档翻译自 staticJPL 的 **Render Dependency Graph Documentation** 项目，原仓库：https://github.com/staticJPL/Render-Dependency-Graph-Documentation 。翻译在尊重原意的基础上，对部分表述做了中文化整理。

![[Unreal Engine Render Dependency Graph/Diagrams/Triangle Render.png]](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/5110a92b8c25e1eab6f28e73456570649c2d0470/Diagrams/Triangle%20Render.png)

有了前面的知识，下一步就是把它用于实践。图形编程里的 “Hello World” 通常是使用 Vertex Shader 和 Pixel Shader 绘制一个基础三角形。在 Unreal Engine 中绘制三角形，也能概括引擎内部渲染任意内容时需要经历的核心流程。

本节会以逐步教程的方式，讲解如何在 Unreal Engine 中使用 Render Dependency Graph 创建一个绘制三角形的 render pass。

## Plugin / Module 与 Shader 文件夹设置

使用 Plugin 还是 Module 取决于你的具体场景，但关键点在于：Plugin 或 Module 中的 “Startup Module” / “Initialize” 函数允许你在 Unreal Engine 完全初始化之前运行代码。这一点很重要，因为 Unreal 的 Renderer 本身就是一个 Module。如果你不理解 Module 和引擎生命周期，建议先阅读相关内容，以便理解模块的运行时链接方式。Renderer 会在运行时链接，因此 shader 代码会在编辑器启动之前编译。

**Plugin 设置**

先在 Unreal Engine 中创建一个基础 C++ 项目，First Person 模板即可。创建项目后，进入项目目录并添加一个 Shader 文件夹。下面是一个大致的目录结构示例：

```
├── YourProjectName
………├──> YourProjectName.Build.cs
………├──> Source Folder
………└──> ...
├──> Plugins
…………├──> Your Plugin Folder
………….……..└──> Shaders (Create the Folder here)
………….…….…….└──> .usf Files go here
…………………├──> Source (Create the Folder here)
……………………………└──>Private
……………………………└──>Public
……………………………└──>YourPluginName.Build.cs
……….└──> YourPluginName.uplugin
```

进入 `PluginName.Build.cs` 文件，确保依赖项已添加。

```cpp
using UnrealBuildTool;

public class YourPluginName : ModuleRules
{
    public YourPluginName(ReadOnlyTargetRules Target) : base(Target)
    {
        PCHUsage = ModuleRules.PCHUsageMode.UseExplicitOrSharedPCHs;

        PublicIncludePaths.AddRange(
            new string[] {
                // ... add public include paths required here ...
                EngineDirectory + "/Source/Runtime/Renderer/Private"
            }
        );

        PrivateIncludePaths.AddRange(
            new string[] {
                // ... add other private include paths required here ...
            }
        );

        PublicDependencyModuleNames.AddRange(
            new string[]
            {
                "Core",
                "RHI",
                "Renderer",
                "RenderCore",
                "Projects"
                // ... add other public dependencies that you statically link with here ...
            }
        );

        PrivateDependencyModuleNames.AddRange(
            new string[]
            {
                "CoreUObject",
                "Engine",
                "Slate",
                "SlateCore"
                // ... add private dependencies that you statically link with here ...
            }
        );

        DynamicallyLoadedModuleNames.AddRange(
            new string[]
            {
                // ... add any modules that your module loads dynamically here ...
            }
        );
    }
}

```

接下来，在 `YourPluginName.uplugin` 中把模块加载阶段设置为 `PostConfigInit`。

```cpp
{
	"FileVersion": 3,
	"Version": 1,
	"VersionName": "1.0",
	"FriendlyName": "YourPluginName",
	"Description": "",
	"Category": "Other",
	"CreatedBy": "",
	"CreatedByURL": "",
	"DocsURL": "",
	"MarketplaceURL": "",
	"SupportURL": "",
	"CanContainContent": true,
	"IsBetaVersion": false,
	"IsExperimentalVersion": false,
	"Installed": false,
	"Modules": [
		{
			"Name": "YourPluginName",
			"Type": "Runtime",
			"LoadingPhase": "PostConfigInit" // Set it Here
		}
	]
}
```

下一步是把 shader 文件夹映射到 Unreal，让引擎可以找到并编译我们的自定义 shader 代码。

```cpp
void FYourPluginNameModule::StartupModule()
{
	// This code will execute after your module is loaded into memory; the exact timing is specified in the .uplugin file permodule
	FString BaseDir = IPluginManager::Get().FindPlugin(TEXT("YourPluginName"))->GetBaseDir();
	FString PluginShaderDir = FPaths::Combine(BaseDir, TEXT("Shaders"));
	AddShaderSourceDirectoryMapping(TEXT("/CustomShaders"), PluginShaderDir);
}
void FYourPluginNameModule::ShutdownModule()
{
// This function may be called during shutdown to clean up your module. For modules that support dynamic reloading,
// we call this function before unloading the module.
}
IMPLEMENT_MODULE(FYourPluginNameModule, YourPluginNamePlugin)
```

你需要 include `Interfaces/IPluginManager.h`，这样才能使用辅助函数获取插件 shader 文件夹的 base directory。`AddShaderSourceDirectoryMapping(TEXT("/CustomShaders"), PluginShaderDir)` 这一行本质上把一个名为 `/CustomShaders` 的虚拟文件夹绑定到插件 shader 目录。我认为这个名字不重要，可以按需要命名；当然也可能有我没有覆盖到的限制。

接下来进入 YourPlugin 文件夹，在 Private 和 Public 文件夹中分别添加 `MyViewExtensionSubSystem.cpp` 和 `MyViewExtensionSubSystem.h`。

Subsystem 很适合在 `PostConfigInit` 阶段创建一个自定义 `FSceneViewExtensionBase` 指针对象。这样我们就能把三角形绘制到编辑器 viewport。你也可以在 `MyCharacter.h` 中声明一个 `TSharedPtr<FMyViewExtension, ESPMode::ThreadSafe>` 对象，并在 `BeginPlay` 中实例化它，这样三角形会在编辑器中点击 Play 之后渲染。

**Module 设置**

**MyViewExtensionSubSystem.cpp**

```cpp
// Fill out your copyright notice in the Description page of Project Settings.
#include "UViewExtensionSubSystem.h"
#include "SceneViewExtension.h"
#include "MyViewExtension.h"
void UMyViewExtensionSubSystem::Initialize(FSubsystemCollectionBase& Collection)
{
	Super::Initialize(Collection);
	// Create Shared Pointer Call
	UE_LOG(LogTemp, Warning, TEXT("View Extension SubSystem Init"));
	// This is the Pointer to the FSceneViewExenstion you will see later on
	// You need this line to run your shader.
	this->ShaderTest = FSceneViewExtensions::NewExtension<MyFViewExtension>();
}
```

**MyViewExtensionSubSystem.h**

```cpp
// Fill out your copyright notice in the Description page of Project Settings.
#pragma once
#include "CoreMinimal.h"
#include "Subsystems/EngineSubsystem.h"
#include "MyViewExtensionSystem.generated.h"
class FViewExtension;
/**
*
*/
UCLASS()
class MyViewExtensionSubSystem: public UEngineSubsystem
{
	GENERATED_BODY()
	protected:
	// Declaration of the Pointer delegate Unreal's FViewExenstion object gives us
	TSharedPtr<FViewExtension,ESPMode::ThreadSafe> ShaderTest;
	public:
	virtual void Initialize(FSubsystemCollectionBase& Collection) override;
};
```

### FSceneViewExtensionBase

这个基类很重要，因为它允许你挂接到渲染管线，并通过它提供的 delegate 插入自定义 render pass。在 `5.1` 之前，创建自定义渲染 pass 仍然需要 plugin 或 module 来初始化 shader 文件夹；但如果没有这个类，你还需要额外修改渲染管线源码，在其中加入自己的 delegate，才能插入自定义渲染 pass。

在本教程中，我们会重写 `FSceneViewExtensionBase` 类中的一个 delegate 函数，把 pass 插入到渲染管线的 Post Processing 阶段。

在引擎源码 `PostProcessing.cpp` 第 411 行附近，会添加 `FSceneViewExtensionBase` 的 delegate。

```cpp
// ../Engine../PostProcessing.cpp"
const auto AddAfterPass = [&](EPass InPass, FScreenPassTexture InSceneColor) -> FScreenPassTexture
{
    // In some cases (e.g. OCIO color conversion) we want View Extensions to be able to add extra custom post processing after
    // the pass.
    FAfterPassCallbackDelegateArray& PassCallbacks = PassSequence.GetAfterPassCallbacks(InPass);
    
    if (PassCallbacks.Num())
    {
        FPostProcessMaterialInputs InOutPostProcessAfterPassInputs = GetPostProcessMaterialInputs(InSceneColor);
        
        for (int32 AfterPassCallbackIndex = 0; AfterPassCallbackIndex < PassCallbacks.Num(); AfterPassCallbackIndex++)
        {
            InOutPostProcessAfterPassInputs.SetInput(EPostProcessMaterialInput::SceneColor, InSceneColor);
            
            FAfterPassCallbackDelegate& AfterPassCallback = PassCallbacks[AfterPassCallbackIndex];
            
            PassSequence.AcceptOverrideIfLastPass(InPass, InOutPostProcessAfterPassInputs.OverrideOutput,
                AfterPassCallbackIndex);
            
            InSceneColor = AfterPassCallback.Execute(GraphBuilder, View, InOutPostProcessAfterPassInputs);
        }
    }
}
```

在 `AddAfterPass` 中可以看到，`FScreenPassTexture InSceneColor` 会把 “SceneColor” 的引用传给被重写的 delegate。`SceneColor` 是一个 `FScreenPassTexture`，描述了一张纹理以及与其配对的 viewport rect。Scene Color 纹理会在整个 PostProcessing 管线中被多次写入，也会成为我们绘制三角形的目标纹理。它就是我们的 “Render Target”。

**类设置**

在插件的 public/private 源码文件夹中再创建一个头文件和 CPP 文件，命名为 `MyViewExtension`（或你喜欢的其他名字）。

```cpp
#pragma once
#include "TriangleShader.h"
#include "SceneViewExtension.h"
#include "RenderResource.h"
class YOURPLUGINNAME_API FMyViewExtension : public FSceneViewExtensionBase {
	public:
	FMyViewExtension(const FAutoRegister& AutoRegister);
	//~ Begin FSceneViewExtensionBase Interface
	virtual void SetupViewFamily(FSceneViewFamily& InViewFamily) override {}
	virtual void SetupView(FSceneViewFamily& InViewFamily, FSceneView& InView) override {};
	virtual void BeginRenderViewFamily(FSceneViewFamily& InViewFamily) override;
	virtual void PreRenderViewFamily_RenderThread(FRDGBuilder& GraphBuilder, FSceneViewFamily& InViewFamily) override{};
	virtual void PreRenderView_RenderThread(FRDGBuilder& GraphBuilder, FSceneView& InView) override;
	virtual void PostRenderBasePass_RenderThread(FRHICommandListImmediate& RHICmdList, FSceneView& InView) override {};
	virtual void PrePostProcessPass_RenderThread(FRDGBuilder& GraphBuilder, const FSceneView& View, const
	FPostProcessingInputs& Inputs) override;
	virtual void SubscribeToPostProcessingPass(EPostProcessingPass Pass, FAfterPassCallbackDelegateArray& InOutPassCallbacks,
	bool bIsPassEnabled)override;
};
```

这里有几个带 `_RenderThread` 后缀的 delegate，以及一些 setup 函数。本文只重写 `SubscribeToPostProcessingPass`。

接下来在 `MyViewExtension.cpp` 中设置一些函数。

```cpp
#include "ViewExtension.h"
#include "TriangleShader.h"
#include "PixelShaderUtils.h"
#include "PostProcess/PostProcessing.h"
#include "PostProcess/PostProcessMaterial.h"
#include "SceneTextureParameters.h"
#include "ShaderParameterStruct.h"
// This Line Declares the Name of your render Pass so you can see it in the render debugger
DECLARE_GPU_DRAWCALL_STAT(TrianglePass);
FMyViewExtension::FMyViewExtension(const FAutoRegister& AutoRegister) : FSceneViewExtensionBase(AutoRegister) {

}
// Begin FLensFlareScene View
FMyViewExtension::FMyViewExtension(const FAutoRegister& AutoRegister) : FViewExtension(AutoRegister) {

}
void FMyViewExtension::SubscribeToPostProcessingPass(EPostProcessingPass Pass, FAfterPassCallbackDelegateArray&
InOutPassCallbacks, bool bIsPassEnabled)
{
	if (Pass == EPostProcessingPass::Tonemap)
	{
	// Create Raw Delegate Here, see later on
	}
}
```

在 `SubscribeToPostProcessingPass` 函数中，我想强调这个 if 语句。它使用 `EPostProcessing` 枚举来定义你要在 Post Processing 阶段的哪个位置插入 render pass。这里我选择放在 `Tonemap` 之后；不过你也可以选择枚举中定义的其他位置。当前目标是把三角形绘制到 Scene Color 上，因为它在整个 Post Process Pass 期间都可用。

### 设置 Global Shader

下一步是创建两个 Global shader。第一个是自定义 Vertex Shader，用来处理顶点缓冲中存储的三角形顶点数据；第二个是 Pixel Shader，用来在光栅化器处理顶点之后给三角形着色。

同样，在插件源码文件夹中创建单独的 cpp/header 文件：`TriangleShader.cpp` 和 `TriangleShader.h`。

**Vertex Shader 类**

```cpp
// TriangleShader.h
// Defined here so we can access it in View Extension.
BEGIN_SHADER_PARAMETER_STRUCT(FTriangleVSParams,)
//RENDER_TARGET_BINDING_SLOTS()
END_SHADER_PARAMETER_STRUCT()
class FTriangleVS : public FGlobalShader
{
	public:
	DECLARE_GLOBAL_SHADER(FTriangleVS);
	SHADER_USE_PARAMETER_STRUCT(FTriangleVS, FGlobalShader)
	using FParameters = FTriangleVSParams;
	
	static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters) {
		return true;
}
};
```

这里保持简单：我们没有为顶点着色器创建任何复杂的宏专用 buffer 或资源。RDG 至少需要这个宏声明。

**Pixel Shader 类**

```cpp
// TriangleShader.h
BEGIN_SHADER_PARAMETER_STRUCT(FTrianglePSParams,)
	RENDER_TARGET_BINDING_SLOTS()
END_SHADER_PARAMETER_STRUCT()
class FTrianglePS: public FGlobalShader
{
	DECLARE_GLOBAL_SHADER(FTrianglePS);
	using FParameters = FTrianglePSParams;
	SHADER_USE_PARAMETER_STRUCT(FTrianglePS, FGlobalShader)
};
```

`RENDER_TARGET_BINDING_SLOTS()` 是这里唯一传入的资源；我们会把 viewport 信息或 Render Target 绑定到自定义 shader HLSL 代码。

在两个类中，都需要把它们声明为 global shader，并定义 shader 所需的本地参数 struct。在 C++ 中，`using` 会让 `FParameters` 这个类型别名指向 `FTrianglePSParams`，这样在其他地方声明该类型指针时可以使用它。

现在把 Triangle HLSL 代码添加到 Shader 文件夹。

进入 Shader 文件夹，创建 `Triangle.usf` 文件，然后粘贴并保存下面的代码。

```c
#include "/Engine/Public/Platform.ush"
#include "/Engine/Private/Common.ush"
#include "/Engine/Private/ScreenPass.ush"
#include "/Engine/Private/PostProcessCommon.ush"
void TriangleVS(
	in float2 InPosition : ATTRIBUTE0,
	in float4 InColor : ATTRIBUTE1,
	out float4 OutPosition : SV_POSITION,
	out float4 OutColor : COLOR0
	)
{
	OutPosition = float4(InPosition, 0, 1);
	OutColor = InColor;
}
void TrianglePS(
	in float4 InPosition : SV_POSITION,
	in float4 InColor : COLOR0,
	out float4 OutColor : SV_Target0)
{
OutColor = InColor;
}
```

### 创建 Vertex Buffer 和 Index Buffer

到这里，我们已经介绍了渲染 pass 涉及的大多数类。接下来还需要为 `Vertex Buffer` 和 `Index Buffer` 定义资源类。

在 `TriangleShader.h` 中：

添加一个 struct，用来定义带颜色的顶点。

```cpp
/** The vertex data used to filter a texture. */
// TriangleShader.h
struct FColorVertex
{
public:
	FVector2f Position;
	FVector4f Color;
};
```

添加 `FVertexBuffer` 类。

```cpp
// TriangleShader.h
/**
* Static vertex and index buffer used for 2D screen rectangles.
*/
class FTriangleVertexBuffer : public FVertexBuffer
{
public:
	/** Initialize the RHI for this rendering resource */
	void InitRHI() override {
	TResourceArray<FColorVertex, VERTEXBUFFER_ALIGNMENT> Vertices;
	Vertices.SetNumUninitialized(3);
	Vertices[0].Position = FVector2f(0.0f,0.75f);
	Vertices[0].Color = FVector4f(1, 0, 0, 1);
	Vertices[1].Position = FVector2f(0.75,-0.75);
	Vertices[1].Color = FVector4f(0, 1, 0, 1);
	Vertices[2].Position = FVector2f(-0.75,-0.75);
	Vertices[2].Color = FVector4f(0, 0, 1, 1);
	FRHIResourceCreateInfo CreateInfo(TEXT("FScreenRectangleVertexBuffer"), &Vertices);
		VertexBufferRHI = RHICreateVertexBuffer(Vertices.GetResourceDataSize(), BUF_Static, CreateInfo);
	}
};
```

在 `FTriangleVertexBuffer` 中重写 `InitRHI` 来初始化顶点数据。在 GPU 编程中，你需要创建一个上下文，以便在把资源从 CPU 拷贝到 GPU 之前完成绑定。为此，需要指定一个 context，其中包含资源名称、大小和其他属性。这里通过 `RHICreateVertexBuffer` 创建顶点缓冲，并把必要信息传给 `VertexBufferRHI`。

添加 Index Buffer：

```cpp
// TriangleShader.h
class FTriangleIndexBuffer : public FIndexBuffer
{
public:
	/** Initialize the RHI for this rendering resource */
	void InitRHI() override
	{
		const uint16 Indices[] = { 0, 1, 2 };
		TResourceArray<uint16, INDEXBUFFER_ALIGNMENT> IndexBuffer;
		uint32 NumIndices = UE_ARRAY_COUNT(Indices);
		IndexBuffer.AddUninitialized(NumIndices);
		FMemory::Memcpy(IndexBuffer.GetData(), Indices, NumIndices * sizeof(uint16));
		FRHIResourceCreateInfo CreateInfo(TEXT("FTriangleIndexBuffer"), &IndexBuffer);
		IndexBufferRHI = RHICreateIndexBuffer(sizeof(uint16), IndexBuffer.GetResourceDataSize(), BUF_Static, CreateInfo);
	}
};
```

流程类似。对于绘制这么简单的三角形，并不一定必须使用 `index buffer`。索引缓冲保存的是指向顶点缓冲中顶点数据的索引。

通常索引缓冲会读取三个顶点的一组组合，不过你可以按需要创建任意组合。这种方式更高效，因为我们可以复用顶点数据，根据场景需求绘制三角形、四边形或其他图元。

最后，为 `Vertex Buffer` 添加一个全局声明资源。它用于定义 `Input assembler` 的输入布局，这样我们才能把 HLSL 代码中的属性正确绑定到顶点数据。

```cpp
// TriangleShader.h
class FTriangleVertexDeclaration : public FRenderResource
{
public:
	FVertexDeclarationRHIRef VertexDeclarationRHI;
	/** Destructor. */
	virtual ~FTriangleVertexDeclaration() {}
	virtual void InitRHI()
	{
		FVertexDeclarationElementList Elements;
		uint16 Stride = sizeof(FColorVertex);
		Elements.Add(FVertexElement(0, STRUCT_OFFSET(FColorVertex, Position), VET_Float2, 0, Stride));
		Elements.Add(FVertexElement(0, STRUCT_OFFSET(FColorVertex, Color), VET_Float4, 1, Stride));
		VertexDeclarationRHI = PipelineStateCache::GetOrCreateVertexDeclaration(Elements);
	}
	virtual void ReleaseRHI()
	{
		VertexDeclarationRHI.SafeRelease();
	}
};
```

这里同样重写 `InitRHI` 来设置声明信息。对于输入布局，我们会定义 elements 对象和 `stride`。在图形学中，`stride` 指的是内存数组里一个元素起始地址到下一个元素起始地址之间的字节数。

为了让输入装配器正确读取 `vertex buffer`，它需要知道一个顶点起始位置到下一个顶点起始位置之间有多少字节。

我们可以使用 `sizeof` 函数计算单个顶点占用的空间，并把它作为后续顶点的偏移。你也会看到 `STRUCT_OFFSET`，它是一个辅助宏，用来确定 struct 中不同字段之间的偏移。

接下来把资源声明为 global，并用 extern 暴露出来，让 Renderer API 能够看到它。

```cpp
// TriangleShader.h
extern YOURPLUGIN_API TGlobalResource<FTriangleVertexBuffer> GTriangleVertexBuffer;
extern YOURPLUGIN_API TGlobalResource<FTriangleIndexBuffer> GTriangleIndexBuffer;
extern YOURPLUGIN_API TGlobalResource<FTriangleVertexDeclaration> GTriangleVertexDeclaration;
```

回到 `Triangle.cpp`，声明 shader 入口点，使其与 HLSL 中的函数名匹配。

```cpp
#include "TriangleShader.h"
#include "Shader.h"
#include "VertexFactory.h"

// Define our Vertex Shader and Pixel Shader Starting point
// This is needed for all shaders

IMPLEMENT_SHADER_TYPE(,FTriangleVS, TEXT("/CustomShaders/Triangle.usf"),TEXT("TriangleVS"),SF_Vertex);
IMPLEMENT_SHADER_TYPE(,FTrianglePS,TEXT("/CustomShaders/Triangle.usf"),TEXT("TrianglePS"),SF_Pixel);

TGlobalResource<FTriangleVertexBuffer> GTriangleVertexBuffer;
TGlobalResource<FTriangleIndexBuffer> GTriangleIndexBuffer;
TGlobalResource<FTriangleVertexDeclaration> GTriangleVertexDeclaration;
```

最后定义我们的 `TGlobalResources` 对象。

### 添加 RDG Pass

现在回到 `MyViewExtension` 的 cpp/header 文件继续更新。

```cpp
// MyViewExtension.h
class YOURPLUGIN_API FMyViewExtension : public FViewExtension
{
public:
    FLensFlareSceneView(const FAutoRegister& AutoRegister);

    virtual void SubscribeToPostProcessingPass(EPostProcessingPass Pass, FAfterPassCallbackDelegateArray& InOutPassCallbacks, bool bIsPassEnabled) override;

protected:
    // Copied from PixelShaderUtils
    template <typename TShaderClass>
    static void AddFullscreenPass(
        FRDGBuilder& GraphBuilder,
        const FGlobalShaderMap* GlobalShaderMap,
        FRDGEventName&& PassName,
        const TShaderRef<TShaderClass>& PixelShader,
        typename TShaderClass::FParameters* Parameters,
        const FIntRect& Viewport,
        FRHIBlendState* BlendState = nullptr,
        FRHIRasterizerState* RasterizerState = nullptr,
        FRHIDepthStencilState* DepthStencilState = nullptr,
        uint32 StencilRef = 0
    );

    template <typename TShaderClass>
    static void DrawFullscreenPixelShader(
        FRHICommandList& RHICmdList,
        const FGlobalShaderMap* GlobalShaderMap,
        const TShaderRef<TShaderClass>& PixelShader,
        const typename TShaderClass::FParameters& Parameters,
        const FIntRect& Viewport,
        FRHIBlendState* BlendState = nullptr,
        FRHIRasterizerState* RasterizerState = nullptr,
        FRHIDepthStencilState* DepthStencilState = nullptr,
        uint32 StencilRef = 0
    );

    static inline void DrawFullScreenTriangle(FRHICommandList& RHICmdList, uint32 InstanceCount);

    // A delegate that is called when the Tone mapper pass finishes
    FScreenPassTexture TrianglePass_RenderThread(FRDGBuilder& GraphBuilder, const FSceneView& View, const FPostProcessMaterialInputs& Inputs);

    // For now we try to build a vertex buffer and see if it puts some shit in it
public:
    static void RenderTriangle(
        FRDGBuilder& GraphBuilder,
        const FGlobalShaderMap* ViewShaderMap,
        const FIntRect& View,
        const FScreenPassTexture& SceneColor
    );
};
```

请记住整个顺序：首先添加一个 pass，把 RDG lambda 函数所需的资源和参数交给它；接着定义 draw call，在其中设置 GPU Pipeline。

你**必须**设置 GPU pipeline，因为它定义了 draw call 所需的全部参数（Blend State、Rasterizer State、Primitive Type、Shader、Viewport、Render Target、Commands）。

GPU Pipeline 会保存当前状态，并把这些状态解释为底层图形 API 调用。

第一个函数是模板化的 `AddFullscreenPass`。它包含 lambda 函数所需的参数，并把这次 draw call 中不用的参数置空。

```cpp
// MyViewExtension.cpp
template <typename TShaderClass>
void FMyViewExtension::AddFullscreenPass(
    FRDGBuilder& GraphBuilder,
    const FGlobalShaderMap* GlobalShaderMap,
    FRDGEventName&& PassName,
    const TShaderRef<TShaderClass>& PixelShader,
    typename TShaderClass::FParameters* Parameters,
    const FIntRect& Viewport,
    FRHIBlendState* BlendState,
    FRHIRasterizerState* RasterizerState,
    FRHIDepthStencilState* DepthStencilState,
    uint32 StencilRef)
{
    check(PixelShader.IsValid());
    ClearUnusedGraphResources(PixelShader, Parameters);

    GraphBuilder.AddPass(
        Forward<FRDGEventName>(PassName),
        Parameters,
        ERDGPassFlags::Raster,
        [Parameters, GlobalShaderMap, PixelShader, Viewport, BlendState, RasterizerState, DepthStencilState, StencilRef]
        (FRHICommandList& RHICmdList)
        {
            FLensFlareSceneView::DrawFullscreenPixelShader<TShaderClass>(
                RHICmdList, GlobalShaderMap, PixelShader, *Parameters, Viewport,
                BlendState, RasterizerState, DepthStencilState, StencilRef);
        });
}
```

### 设置 Draw Call

```cpp
template <typename TShaderClass>
void FMyViewExtension::DrawFullscreenPixelShader(
    FRHICommandList& RHICmdList,
    const FGlobalShaderMap* GlobalShaderMap,
    const TShaderRef<TShaderClass>& PixelShader,
    const typename TShaderClass::FParameters& Parameters,
    const FIntRect& Viewport,
    FRHIBlendState* BlendState,
    FRHIRasterizerState* RasterizerState,
    FRHIDepthStencilState* DepthStencilState,
    uint32 StencilRef)
{
    check(PixelShader.IsValid());

    RHICmdList.SetViewport(
        (float)Viewport.Min.X, (float)Viewport.Min.Y, 0.0f,
        (float)Viewport.Max.X, (float)Viewport.Max.Y, 1.0f);

    // Begin Setup Gpu Pipeline for this Pass
    FGraphicsPipelineStateInitializer GraphicsPSOInit;
    TShaderMapRef<FTriangleVS> VertexShader(GlobalShaderMap);

    RHICmdList.ApplyCachedRenderTargets(GraphicsPSOInit);

    GraphicsPSOInit.BlendState = TStaticBlendState<>::GetRHI();
    GraphicsPSOInit.RasterizerState = TStaticRasterizerState<>::GetRHI();
    GraphicsPSOInit.DepthStencilState = TStaticDepthStencilState<false, CF_Always>::GetRHI();
    GraphicsPSOInit.BoundShaderState.VertexDeclarationRHI = GTriangleVertexDeclaration.VertexDeclarationRHI;
    GraphicsPSOInit.BoundShaderState.VertexShaderRHI = VertexShader.GetVertexShader();
    GraphicsPSOInit.BoundShaderState.PixelShaderRHI = PixelShader.GetPixelShader();
    GraphicsPSOInit.PrimitiveType = PT_TriangleList;

    GraphicsPSOInit.BlendState = BlendState ? BlendState : GraphicsPSOInit.BlendState;
    GraphicsPSOInit.RasterizerState = RasterizerState ? RasterizerState : GraphicsPSOInit.RasterizerState;
    GraphicsPSOInit.DepthStencilState = DepthStencilState ? DepthStencilState : GraphicsPSOInit.DepthStencilState;

    // End Gpu Pipeline setup
    SetGraphicsPipelineState(RHICmdList, GraphicsPSOInit, StencilRef);
    SetShaderParameters(RHICmdList, PixelShader, PixelShader.GetPixelShader(), Parameters);
    DrawFullScreenTriangle(RHICmdList, 1);
}

```

我创建了一个 `RenderTriangle` 函数，用来封装 Add Pass 和 Draw Call 相关逻辑。

```cpp
// FMyViewExtension.cpp
void FMyViewExtension::RenderTriangle(
    FRDGBuilder& GraphBuilder,
    const FGlobalShaderMap* ViewShaderMap,
    const FIntRect& ViewInfo,
    const FScreenPassTexture& SceneColor)
{
    // Begin Setup
    // Shader Parameter Setup
    FTrianglePSParams* PassParams = GraphBuilder.AllocParameters<FTrianglePSParams>();
    
    // Set the Render Target In this case is the Scene Color
    PassParams->RenderTargets[0] = FRenderTargetBinding(SceneColor.Texture, ERenderTargetLoadAction::ENoAction);

    // Create FTrianglePS Pixel Shader
    TShaderMapRef<FTrianglePS> PixelShader(ViewShaderMap);

    // Add Pass
    AddFullscreenPass<FTrianglePS>(GraphBuilder,
        ViewShaderMap,
        RDG_EVENT_NAME("TrianglePass"),
        PixelShader,
        PassParams,
        ViewInfo);
}
```

### 绑定 Delegate

我创建了 `TrianglePass_RenderThread` 函数。

```cpp
FScreenPassTexture FMyViewExtension::TrianglePass_RenderThread(FRDGBuilder& GraphBuilder, const FSceneView& View, const FPostProcessMaterialInputs& InOutInputs)
{
    const FScreenPassTexture SceneColor = InOutInputs.GetInput(EPostProcessMaterialInput::SceneColor);

    RDG_GPU_STAT_SCOPE(GraphBuilder, TrianglePass)
    RDG_EVENT_SCOPE(GraphBuilder, "TrianglePass");

    // Casting the FSceneView to FViewInfo
    const FIntRect ViewInfo = static_cast<const FViewInfo&>(View).ViewRect;
    const FGlobalShaderMap* ViewShaderMap = static_cast<const FViewInfo&>(View).ShaderMap;

    RenderTriangle(GraphBuilder, ViewShaderMap, ViewInfo, SceneColor);

    return SceneColor;
}
```

在调用 `RenderTriangle` 之前，需要先取得 `SceneColor` 纹理，并执行两次 `static_cast`：一次用于获取 viewport 尺寸，另一次用于获取指向 global shader map 的指针。global shader map 指向我们使用 global 宏声明的自定义 shader 对象。

最后，把 delegate 函数 `TrianglePass_RenderThread` 添加到前面提到的 if 语句中。

```cpp
void FLensFlareSceneView::SubscribeToPostProcessingPass(EPostProcessingPass Pass, FAfterPassCallbackDelegateArray&
InOutPassCallbacks, bool bIsPassEnabled)
{
	if (Pass == EPostProcessingPass::Tonemap)
	{
		InOutPassCallbacks.Add(FAfterPassCallbackDelegate::CreateRaw(this, &FLensFlareSceneView::TrianglePass_RenderThread));
	}
}
```

**编译**

现在应该能在编辑器 viewport 中看到绘制出来的三角形。

![[Unreal Engine Render Dependency Graph/Diagrams/TriangleOutput.png]](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/5110a92b8c25e1eab6f28e73456570649c2d0470/Diagrams/TriangleOutput.png)
