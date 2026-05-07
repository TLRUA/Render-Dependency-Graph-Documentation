# 渲染依赖图（RDG）

> 出处：本文档翻译自 staticJPL 的 **Render Dependency Graph Documentation** 项目，原仓库：https://github.com/staticJPL/Render-Dependency-Graph-Documentation 。翻译在尊重原意的基础上，对部分表述做了中文化整理。

2017 年，Yuriy O’Donnell 在 Frostbite 工作期间开创性地提出了 render graph 系统，并在 GDC 上展示了第一个 Frame Graph。这个系统带来的优势随后被 Unreal Engine 吸收；到 2021 年左右，使用 render graph 已经成为 AAA 游戏引擎开发中的常见标准。

下面的内容基于 Riccardo Loggini 对 Render Dependency Graph 工作方式的详细介绍整理而来。

![[Unreal Engine Render Dependency Graph/Diagrams/RDG_CommandQueue.png]](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/563954a23906d392d55727b1132446abdd73d0dd/Diagrams/RDG_CommandQueue.png)

### 特性

Render Dependency Graph 会把渲染操作抽象成一种更简洁的形式，用于生成渲染代码。这种方式能提升代码清晰度，并且更便于调试：工具可以理解资源生命周期和渲染 pass 依赖，从而减少开发时间。

DX12 和 Vulkan 等新一代图形 API 会根据实际执行的操作来管理资源状态转换。

![[Unreal Engine Render Dependency Graph/Diagrams/RDG_ResourceTransition.png]](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/563954a23906d392d55727b1132446abdd73d0dd/Diagrams/RDG_ResourceTransition.png)

使用 render graph 后，这些操作可以自动处理，而不需要程序员手动插入所有细节。以上图为例，图形程序员只需要声明 shader 输入需要哪些资源。由于资源转换由 graph 处理，你可以把绿线理解为“读”操作，把红线理解为“写”操作。每个 render graph 节点都知道自己的读写关系，因此可以放置用于资源状态转换的 `Barriers`。这意味着 RDG 能在更合适的位置放置 barrier，并得到更合理的 command queue 设置。例如，如果 `resource A` 在 `Pass 1` 中作为 shader resource 使用，而在 `pass 2` 中作为 render target 使用，那么这两个 pass 之间仍然需要一次资源转换。自动管理这些转换可以减少调用并节省内存分配。

![[Unreal Engine Render Dependency Graph/Diagrams/RDG_PassResourceLifetime.png]](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/563954a23906d392d55727b1132446abdd73d0dd/Diagrams/RDG_PassResourceLifetime.png)

以上图为例，`resource A` 只使用到第三个 pass；而 `resource C` 从第四个 pass 才开始使用。因此二者生命周期不重叠，就可以复用同一块内存。

`resource A` 与 `resource D` 也是同样的概念。一般来说，内存分配之间可能存在多种可重叠方式，因此还需要更智能的策略来检测最优分配方案。

### RDG 资源

有些资源按“每帧”使用：严格来说，它们被称为 graph resource 或 transient resource，因为它们的生命周期可以完全由 render graph 管理。“每帧”资源的例子包括 `Gbuffers` 和 Camera Depth，它们会在后续光照 pass 中被使用。

`Transient` 资源只打算在单帧内的某个时间段存在，因此非常适合做内存复用。还有一些资源是在 RDG 之外创建、但被 RDG 使用或依赖的，例如窗口 swapchain 的 back buffer。对于这种资源，graph 通常只负责管理它们的状态；它们被称为 `external resources`。

### Transient Resource System

Transient 资源的生命周期允许进行所谓的“resource aliasing”（这是 DX12 术语）。

![[Unreal Engine Render Dependency Graph/Diagrams/RDG_Aliasing.png]](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/563954a23906d392d55727b1132446abdd73d0dd/Diagrams/RDG_Aliasing.png)

资源别名化在 render graph 中尤其有价值，最多可以节省相当可观的资源分配空间。它会增加资源管理复杂度，但如果目标是节省内存，通常值得这么做。

### 构建跨队列同步

![[Unreal Engine Render Dependency Graph/Diagrams/RDG_CommandDependencyTree.png]](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/563954a23906d392d55727b1132446abdd73d0dd/Diagrams/RDG_CommandDependencyTree.png)

最后，graph 允许我们使用多个 command queue 并行运行任务。依赖树可以帮助同步这些队列，避免共享资源上的竞争条件。

把来自相关队列的 pass 按依赖关系展开后，会形成一个无环图。树中的每一层称为一个依赖层级，层内 pass 在资源使用上彼此独立。因此，同一依赖层级内的 pass 理论上可以异步运行。即使同一队列中有多个 pass 落在同一依赖层级，也不一定会造成问题。

因此，可以在每个依赖层级末尾放置一个带 GPU fence 的同步点，并在单个 graphics command list 上为所有队列执行所需的资源转换。当然，这种做法并非没有成本：fence 和跨队列同步会带来时间开销。它也不一定总能得到最优、最少的同步次数，但通常能以可接受的性能覆盖大多数边界情况。

这概括了 Render Graph 的主要优势：

- 更好的资源管理。
- 更容易构建调试工具。
- 支持并行 command list。
- 自动化同步。

### RDG 动态流程

在 RDG 语境下，还需要补充一些新术语。

- **View**：观察 `FScene` 的单个“视口”。例如分屏游戏或 VR 中分别渲染左右眼时，一个 view family 内会包含两个 view。

- **Vertex Factory**：封装顶点数据并将其连接到顶点着色器输入的类。根据要渲染的 mesh 类型不同，会有不同类型的 vertex factory。

- **Pooled Resource**：由 RDG 创建和管理的图形资源。它们只保证在 RDG pass 执行期间可用。

- **External Resource**：独立于 RDG 创建的图形资源。

Unreal Engine 的 RDG 工作流可以分成三个阶段：

**Setup phase（设置阶段）**：声明将要存在的渲染 pass，以及它们会访问哪些资源。

**Compile phase（编译阶段）**：推导资源生命周期，并据此进行资源分配。

**Running/Execute phase（运行/执行阶段）**：执行所有 graph 节点。

![[Unreal Engine Render Dependency Graph/Diagrams/RDG_Stages.png]](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/563954a23906d392d55727b1132446abdd73d0dd/Diagrams/RDG_Stages.png)

### Setup Stage

Setup stage 从 `FRenderModule` 内部开始，并且只由渲染线程的主函数触发。它会为可见 view 以及与之关联的所有对象构建 pass。

```cpp
FDeferredShadingSceneRenderer::Render(FRHICommandListImmediate& RHICmdList)
```

所有 RDG Pass 的通用写法可以简化为下面这种形式：

```cpp
// Instantiate the resources we need for our pass
FShaderParameterStruct* PassParameters = GraphBuilder.AllocParameters&lt;FShaderParameterStruct>();
// Fill in the pass parameters
PassParameters->MyParameter = GraphBuilder.CreateSomeResource(MyResourceDescription, TEXT("MyResourceName"));
// Define pass and add it to the RDG builder
GraphBuilder.AddPass( RDG_EVENT_NAME("MyRDGPassName"), PassParameters, ERDGPassFlags::
Raster, [PassParameters, OtherDataToCapture](FRHICommandList&RHICmdList) {
// … pass logic here, render something! ...
}
```

- **Pass Name**：最终会表示为 `FRDGEventName` 类型对象，其中包含该 pass 的描述。它用于调试和性能分析工具。

- **Pass Parameters**：这个对象通常派生自 Shader Parameter Struct，需要通过 `GraphBuilder.AllocParameters()` 创建，并用 `BEGIN_SHADER_PARAMETER_STRUCT(FMyShaderParameters, )` 宏定义。PassParameters 至少需要区分 shader resource 和 render target，这样才能检测正确的资源转换（后文 Shader Parameters 章节会继续说明）。它可能来自：
  - 某个 shader 自己的 uniform buffer。例如只有一个 compute shader 时，PassParameters 通常会是 `FMyShader::FParameters`。
  - 在源码 `.cpp` 文件中通用定义的 shader uniform buffer。这类 buffer 的名称通常会包含 “PassParameters”，表示它用于整个 pass，而不是单个 shader。

- **Pass Flags**：`ERDGPassFlags` 类型的一组标志，主要用于说明该 pass 内要执行哪类操作，例如 raster、copy 或 compute。

- **Lambda Function**：包含 pass 的“主体”，也就是运行时要执行的逻辑。lambda 可以捕获任意数量的对象，用于后续设置渲染操作。需要记住的是，在收集阶段之后，这些 lambda 通常不会立即执行，而是被延迟到合适阶段；不过在某些情况下也可能立即执行。

### Compile Phase

`compile` 阶段是完全自动的，可以理解为“不可编程”的阶段：编写 render pass 的程序员通常无法直接影响它。在这一阶段，graph 会被检查以寻找可能的流程优化，它会：

1. 剔除未被引用但已定义的资源和 pass。例如我们想绘制场景的第二个 debug view 时，可能只需要绘制其中某些 pass。
2. 计算并处理使用中资源的生命周期。
3. 分配资源。
4. 构建优化后的资源转换图。

### Running Stage

Running Stage 指 RDG pass 的 lambda 函数实际执行的时间。它会异步发生，确切执行时机完全由 RDG 决定。

当 lambda 主体执行时，可用输入包括 lambda 捕获的变量，以及一个 command list：`Compute` / `AsyncCompute` 工作负载会使用 `RHIComputeCommandList&`，光栅化操作会使用 `FRHICommandList&`。

lambda 主体内部本质上会做这些事：

- 设置 pipeline state object，例如 rasterizer、blend 和 depth/stencil state。
- 设置 shader 及其属性。
- 选择要使用的 shader，并把它们绑定到当前管线。
- 定义参数，也就是把资源绑定到当前 command list 的 shader slot。
- 发出 copy / draw / dispatch 命令，并把渲染命令提交到 command list。

### Execute Phase

执行阶段基本上就是遍历所有通过 compile phase 剔除后保留下来的 pass，并在 command list 上执行 draw 和 dispatch 命令。在 execute phase 之前，所有资源都由不透明的抽象引用处理；到了 execute phase，才会访问真实的 GPU API 资源并把它们设置到管线中。CPU 侧 command list 的准备工作在很多情况下可以高度并行化，因为大多数 pass 的 command list 设置彼此独立。除此之外，在单个 command queue 上提交 command list 并不是线程安全的，而且无论如何，都应该先判断并行化是否真的能带来明显收益。

#### Shader Types

shader 的基类是 `FShader`，但我们主要会使用两类 shader：

- `FGlobalShader`：所有派生自它的 shader 都属于 global shaders 组。global shader 在整个引擎中生成单个实例，并且只能使用全局参数。
- `FMaterialShader`：派生类会使用与材质绑定的参数。如果还使用与 vertex factory 绑定的参数，则应该参考 `FMeshMaterialShader`。

注意：本文使用的是 Global Shader，因为示例在后处理 pass 中渲染。Material shader 与 vertex factory 强相关；这里没有时间展开和实验相关内容。如果你感兴趣，参考资料中提供了一些相关资源。

### Shader Parameters

![[Unreal Engine Render Dependency Graph/Diagrams/RDG_ShaderParameterBinding.png]](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/563954a23906d392d55727b1132446abdd73d0dd/Diagrams/RDG_ShaderParameterBinding.png)

Shader parameters 是用于标识某个 shader 所使用资源槽位的对象。在设置图形、计算或其他 GPU 操作的资源时会用到这些参数。

设置 shader parameter 的过程，本质上是在 command list 上，把某个资源绑定到 shader parameter 指定的索引位置。

**注意**：如果“绑定”和“上下文”这些概念让你困惑，通常说明还缺少一些更底层的 GPU 与图形 API 知识。本文在补充说明中给出了简要解释。

我们有以下几类 Shader Parameters，可在 `ShaderParametersUtils.h` 和 `ShaderParameters.h` 中看到：

- **FShaderParameter**：shader 参数的寄存器绑定，例如 float1/2/3/4，也可以是数组或 UAV。
  
- **FShaderResourceParameter**：shader resource 绑定，例如纹理或 sampler state。
  
- **FRWShaderParameter**：用于绑定 UAV 或 SRV 资源的类。

- **TShaderUniformBufferParameter**：带有特定结构的 shader uniform buffer 绑定（模板化）。这个参数引用一个 struct，其中包含某个 shader 定义的所有资源。后文会继续说明。

和 Unreal Engine 中许多地方一样，shader parameter 类通过宏定义自身组成方式。shader parameter 中最重要的部分是它的 Layout：这是编译期定义的内部变量，用于指定其结构，并由 `Layout Fields` 组成。

```cpp
LAYOUT_FIELD(MyDataType, MyFiledName);
LAYOUT_FIELD(FShaderResourceParameter, UAVParameter);
```

layout 的使用方式取决于 shader parameter 类型，它可以包含任意数据（例如 parameter index）。它的目的始终是保存 shader parameter 相关信息（例如 D3D12 中的 CBV、SRV、UAV），这样在 command list 中执行 shader 时，就能用这些信息把资源绑定到正确位置。

### Shader Uniform Buffer Parameter

Unreal Engine 中 `Uniform Buffer Parameter` 的概念，与标准图形学中常见的 uniform buffer 不完全相同：这里它本质上是一个 shader parameter struct。

如前面 RDG 章节提到的，Uniform Buffer 可以使用 Shader Parameter Struct 宏定义；它既可以放在 shader 类声明内部，也可以放在全局作用域。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FMyShaderParameters, )
	SHADER_PARAMETER_RDG_TEXTURE(Texture2D, InputTexture)
	SHADER_PARAMETER_SAMPLER(SamplerState, InputSampler)
	RENDER_TARGET_BINDING_SLOTS()
END_SHADER_PARAMETER_STRUCT()
```

这一族宏非常灵活，可以包含：

- **Shader Parameters**：纹理、sampler、buffer 和 descriptor。完整宏列表可参考 `ShaderParameterMacros.h`。
- **Nested Structs**：可以把一个 shader parameter struct 封装到另一个 struct 中。这通过 `SHADER_PARAMETER_STRUCT(StructType,MemberName)` 和 `SHADER_PARAMETER_STRUCT_ARRAY(..)` 宏实现。
- **Binding Slots**：通过 `RENDER_TARGET_BINDING_SLOTS()` 宏提供，它会向参数 struct 中添加一个可赋值 Render Target 数组。

**用法**
- 如果定义在 shader 外部，`FMyShaderParameters` 通常会命名为 `F<MyPassName>PassParameters`。这类 shader parameter struct 会作为 RDG 的 Pass Parameters 使用，正如本文 RDG 部分所述。
- 如果定义在 shader 类内部，`FMyShaderParameters` 通常会命名为 `FParameters`。此时还需要在 shader 类顶部使用宏 `SHADER_USE_PARAMETER_STRUCT(FMyShaderClass, FBaseClassFromWhatMyShaderDerivesFrom)`。

**设置 Shader Parameters**

大多数情况下，在 RDG pass 中使用 shader 时，可以调用：

```cpp
SetShaderParameters(TRHICmdList& RHICmdList, const TShaderRef&lt;TShaderClass>& Shader, TShaderRHI* ShadeRHI, const typename
TShaderClass::FParameters& Parameters)
```

![[Unreal Engine Render Dependency Graph/Diagrams/RDG_SettingShaderParams.png]](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/563954a23906d392d55727b1132446abdd73d0dd/Diagrams/RDG_SettingShaderParams.png)

来自 `ShaderParameterStruct.h` 的逻辑会被调用，用于把输入资源绑定到特定 shader。

该函数会先调用 `ValidateShaderParameters(Shader, Parameters);`，检查所有输入 shader resource 是否覆盖了 shader 期望的全部参数。随后它会把所有资源绑定到对应参数上：本节开头列出的每种参数类型（例如 `FShaderParameter`、`FShaderResourceParameter` 等）都会有自己的绑定调用。设置 uniform buffer 资源时的整体流程可以概括如下：

**BufferIndex** 会用于所有 `FParameterStructReference`，也会用于基础 `FParameters` 元素，因为它们同样存储在 buffer 中。`CmdList::SetUniformBuffer` 面对这些输入参数时具体如何执行，完全取决于当前使用的渲染平台，并且不同平台差异很大。

### Shader 宏的使用与设置

**Local Parameters**

如果我们要从一些参数开始，生成自己的 Uniform Buffer（许多 shader 会使用的 Constant Buffer），可以先看 HLSL 声明与宏设置之间的对应关系：

```cpp
float2 ViewPortSize;
float4 Hello;
float World;
float3 FooBarArray[16];
Texture2D BlueNoiseTexture;
SamplerState BlueNoiseSampler;
// Note Sampler States are objects that we used to "Sample a texture" basically reading
// the sample if we want to do masking, blending or other render wizardry.
Texture2D SceneColorTexture;
SamplerState SceneColorSampler;
RWTexture2D<float4> SceneColorOutput;
```

如上一节所述，我们需要使用内部宏来绑定这些参数。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FMyShaderParameters,)
	SHADER_PARAMETER(FVector2f,ViewPortSize)
	SHADER_PARAMETER(FVector4f,Hello)
	SHADER_PARAMETER(float,World)
	SHADER_PARAMETER(FVector3f,FooBarArray,[16])
	SHADER_PARAMETER_TEXTURE(Texture2D,BlueNoiseTexture)
	SHADER_PARAMETER_TEXTURE(SamplerState,BlueNoiseSampler)
	SHADER_PARAMETER_TEXTURE(Texture2D,SceneColorTexture)
	SHADER_PARAMETER_TEXTURE(SamplerState,SceneColorSampler)
	SHADER_PARAMETER_UAV(Texture2D,SceneColorTexture)
END_SHADER_PARAMETER_STRUCT()
```

`SHADER_PARAMETER_STRUCT` 宏会在内部填充所有数据，以便在编译期生成反射数据。

```cpp
const FShaderParametersMetadata* ParametersMetadata = FShaderParameters::FTypeInfo::GetStructMetadata();
```

**对齐要求**

你需要遵守对齐规则。Unreal 采用 16 字节自动对齐原则，因此声明 struct 时成员顺序很重要。

主要规则是：每个成员会按其大小的下一个幂次对齐，但只在该成员大于 4 字节时适用。例如：

- 指针按 8 字节对齐。
- `float`、`uint32`、`int32` 按 4 字节对齐。
- `FVector2f`、`FIntPoint` 按 8 字节对齐。
- `FVector` 和 `FVector4f` 按 16 字节对齐。

如果不遵守这些对齐规则，引擎会在编译期触发 assert。

每个成员的自动对齐不可避免地会产生 padding，如下所示：

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FMyShaderParameters,)
	SHADER_PARAMETER(FVector2f,ViewPortSize) // 2 x 4 bytess
	// 2 x 4 byte of padding
	SHADER_PARAMETER(FVector4f,Hello) // 4 x 4 bytes
	SHADER_PARAMETER(float,World) // 1 x 4 bytes
	// 3 x 4 byte of padding
	SHADER_PARAMETER(FVector3f,FooBarArray,[16]) // 4 x 4 x 16 bytes
	SHADER_PARAMETER_TEXTURE(Texture2D,BlueNoiseTexture) // 8 bytes
	SHADER_PARAMETER_TEXTURE(SamplerState,BlueNoiseSampler) // 8 bytes
	SHADER_PARAMETER_TEXTURE(Texture2D,SceneColorTexture) // 8 bytes
	SHADER_PARAMETER_TEXTURE(SamplerState,SceneColorSampler) // 8 bytes
	SHADER_PARAMETER_UAV(Texture2D,SceneColorTexture) // 8 bytes
END_SHADER_PARAMETER_STRUCT()
```

```cpp
SHADER_PARAMETER(FVector3f,WorldPositionAndRadius) // DONT DO THIS
// ----------------------- //
// Do this
SHADER_PARAMETER(FVector, WorldPosition) // Good
SHADER_PARAMETER(float,WorldRadius) // Good
SHADER_PARAMETER_ARRAY(FVector4f,WorldPositionRadius,[16]) // Good
```

**绑定 shader**

设置好 shader 参数并确认对齐无误后，可以用下面的方式声明：

```cpp
SHADER_USE_PARAMETER_STRUCT(FMyShaderCS,FGlobalShader)
```

```cpp
class FMyShaderCS : public FGlobalShader
{
DECLARE_GLOBAL_SHADER(FMyShaderCS);
SHADER_USE_PARAMETER_STRUCT(FMyShaderCS,FGlobalShader)
static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters) {
	return true;
}
	using FParameters = FMyShaderParameters;
}
// Additionally we could inline the struct definition like this
class FMyShaderCS : public FGlobalShader
{
	DECLARE_GLOBAL_SHADER(FMyShaderCS);
	SHADER_USE_PARAMETER_STRUCT(FMyShaderCS,FGlobalShader)
	static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters) 
	{
		return true;
	}
	using FParameters = FMyShaderParameters;
BEGIN_SHADER_PARAMETER_STRUCT(FMyShaderParameters,)
	SHADER_PARAMETER(FVector2f,ViewPortSize) // 2 x 4 bytess
	// 2 x 4 byte of padding
	SHADER_PARAMETER(FVector4f,Hello) // 4 x 4 bytes
	SHADER_PARAMETER(float,World) // 1 x 4 bytes
	// 3 x 4 byte of padding
	SHADER_PARAMETER(FVector3f,FooBarArray,[16]) // 4 x 4 x 16 bytes
	SHADER_PARAMETER_TEXTURE(Texture2D,BlueNoiseTexture) // 8 bytes
	SHADER_PARAMETER_TEXTURE(SamplerState,BlueNoiseSampler) // 8 bytes
	SHADER_PARAMETER_TEXTURE(Texture2D,SceneColorTexture) // 8 bytes
	SHADER_PARAMETER_TEXTURE(SamplerState,SceneColorSampler) // 8 bytes
	SHADER_PARAMETER_UAV(Texture2D,SceneColorTexture) // 8 bytes
END_SHADER_PARAMETER_STRUCT()
}
```

```cpp
// Setting the Parameters in the C++ code before passing it off to the Lambda Function
FMyShaderParameters* PassParameters = GraphBuilder.AllocParameters<FMyShaderParameters>();

PassParameters.ViewPortSize = View.ViewRect.Size();
PassParameters.World = 1.0f;
PassParameters.FooBarArray[4] = FVector(1.0f,0.5f,0.5f);

// Get access to the shader we class we declared Global
TShaderMapRef<FMyShaderCS> ComputeShader(View.Shadermap);
RHICmdList.SetComputeShader(ShaderRHI);

// Setup your Shader Parameters
SetShaderParameters(RHICmdList,*ComputeShader,ComputeShader->GetComputeShader(),Parameters);
RHICmdList.DispatchComputeShader(GroupCount.X,GroupCount.Y,GroupCount.Z);
```

**Global Uniform Buffer**

流程基本相同，只是有一些小差异。

```cpp
BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FSceneTextureUniformParameters,/*Blah_API*/)
	// Scene Color / Depth
	SHADER_PARAMETER_TEXTURE(Texture2D, SceneColorTexture)
	SHADER_PARAMETER_SAMPLER(SamplerState,SceneColorTextureSampler)
	SHADER_PARAMETER_TEXTURE(Texture2D, SceneColorTexture)
	SHADER_PARAMETER_SAMPLER(SamplerState,SceneDepthTextureSampler)
	SHADER_PARAMETER_TEXTURE(Texture2D<float>, SceneDepthTexturesNonMS)
	//GBuffer
	SHADER_PARAMETER_TEXTURE(Texture2D, GBufferATexture)
	SHADER_PARAMETER_TEXTURE(Texture2D, GBufferBTexture)
	SHADER_PARAMETER_TEXTURE(Texture2D, GBufferCTexture)
	SHADER_PARAMETER_TEXTURE(Texture2D, GBufferDTexture)
	SHADER_PARAMETER_TEXTURE(Texture2D, GBufferETexture)
	SHADER_PARAMETER_TEXTURE(Texture2D, GBufferVelocityTexture)
	// ...
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```

struct 设置完成后，需要调用 implement 宏；后面的字符串是在 HLSL 文件中定义的真实名称。

```cpp
IMPLEMENT_GLOBAL_SHADER_PARAMETER_STRUCT(FSceneTexturesUniformParameters,"SceneTextureStruct");
```

在 Unreal 系统内部，`Common.ush` 会引用生成出来的代码；你会在很多 HLSL 文件中看到 `Common.ush` 被 include。还有其他 include 文件提供了渲染代码中常用的实用函数。

现在，我们设置的 uniform buffer 可以在任何地方访问：

```cpp
// Generated file that contains the unifrom buffer declarations that we need to compile the shader want
#include "/Engine/Generated/GeneratedUniformBuffers.ush"
```

![[Unreal Engine Render Dependency Graph/Diagrams/SceneTextureStruct.png]](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/563954a23906d392d55727b1132446abdd73d0dd/Diagrams/SceneTextureStruct.png)

现在在 Parameter struct 中引用我们的 uniform buffer：

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FParameters,)
	//...
	// Here we ref our unifor buffer
	SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters,ViewUniformBuffer)
END_SHADER_PARAMETER_STRUCT()
```

同样，在把参数传入 Lambda Function 之前，先在 C++ 中完成设置：

```cpp
FMyShaderParameters* PassParameters = GraphBuilder.AllocParameters<FDMyShaderParameters>();
PassParameters.ViewPortSize = View.ViewRect.Size();
PassParameters.World = 1.0f;
PassParameters.FooBarArray[4] = FVector(1.0f,0.5f,0.5f);
PassParameters.ViewUniformBuffer = View.ViewUniformBuffer;
```
