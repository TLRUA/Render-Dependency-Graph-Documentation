# 渲染硬件接口（RHI）

> 出处：本文档翻译自 staticJPL 的 **Render Dependency Graph Documentation** 项目，原仓库：https://github.com/staticJPL/Render-Dependency-Graph-Documentation 。翻译在尊重原意的基础上，对部分表述做了中文化整理。

最初的 RHI 是基于 D3D11 API 设计的，其中包含一些资源管理与命令接口。Unreal Engine 是一个支持移动端、主机和 PC 等多平台的通用工具，而这些平台又可能使用 DirectX、Vulkan、OpenGL、Metal 等不同图形 API。为了解决这一问题，Unreal 在渲染代码与这些 API 之间抽象出一层接口，使渲染代码尽可能保持统一和可理解。

这种抽象会通过下图所示的不同线程协作来完成：

![[Unreal Engine Render Dependency Graph/Diagrams/RHIUnrealDiagram.png]](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/e97260a557e345d37c2bb0b6352c82d35a4138df/Diagrams/RHIUnrealDiagram.png)

这里有 Game Thread、Render Thread 和 RHI Thread。需要理解的重点是：除了少数特殊情况外，任何被渲染的对象，通常都会在游戏线程和渲染线程之间拥有一组对应的“镜像对象”。

**Game Thread**
- Primitive Components
- Light Components

**Render Thread**
- Primitive Proxy
- Light Proxy
  
**RHI Thread**
- 根据指定的图形 API，把渲染线程发出的 RHI “immediate” 指令翻译给 GPU。注意，这里的 RHI Immediate 的确表示“立即”，它不同于通常会延迟执行的普通 RHI 命令。
- DX12、Vulkan 和 Host 支持并行处理；如果某条 RHI immediate 指令生成了并行命令，那么 RHI 线程会并行翻译这些命令。

![[Unreal Engine Render Dependency Graph/Diagrams/Parallel CommandList RHI.png]](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/e97260a557e345d37c2bb0b6352c82d35a4138df/Diagrams/Parallel%20CommandList%20RHI.png)

## RHI 基础

**FRenderResource**

`FRenderResource` 是渲染线程上的渲染资源表示。它由渲染线程管理和传递，可作为游戏线程与 RHI 线程之间的中间数据。

```cpp
/**
* A rendering resource which is owned by the rendering thread.
* NOTE - Adding new virtual methods to this class may require stubs added to FViewport/FDummyViewport, otherwise certain
modules may have link errors
*/
class RENDERCORE_API FRenderResource
{
public:
////////////////////////////////////////////////////////////////////////////////////
// The following methods may not be called while asynchronously initializing / releasing render resources.
/** Release all render resources that are currently initialized. */
static void ReleaseRHIForAllResources();
/** Initialize all resources initialized before the RHI was initialized. */
static void InitPreRHIResources();
/**
* Initializes the dynamic RHI resource and/or RHI render target used by this resource.
* Called when the resource is initialized, or when reseting all RHI resources.
* Resources that need to initialize after a D3D device reset must implement this function.
* This is only called by the rendering thread.
*/
virtual void InitDynamicRHI() {}
/**
* Releases the dynamic RHI resource and/or RHI render target resources used by this resource.
* Called when the resource is released, or when reseting all RHI resources.
* Resources that need to release before a D3D device reset must implement this function.
* This is only called by the rendering thread.
*/
virtual void ReleaseDynamicRHI() {}
/**
* Initializes the RHI resources used by this resource.
* Called when entering the state where both the resource and the RHI have been initialized.
* This is only called by the rendering thread.
*/
virtual void InitRHI() {}
/**
* Releases the RHI resources used by this resource.
* Called when leaving the state where both the resource and the RHI have been initialized.
* This is only called by the rendering thread.
*/
virtual void ReleaseRHI() {}
/**
* Initializes the resource.
* This is only called by the rendering thread.
*/
virtual void InitResource();
/**
* Prepares the resource for deletion.
* This is only called by the rendering thread.
*/
virtual void ReleaseResource();
/**
* If the resource's RHI resources have been initialized, then release and reinitialize it. Otherwise, do nothing.
* This is only called by the rendering thread.
*/
void UpdateRHI();
(...)
};
```

有许多子类继承自 `FRenderResource`，这样渲染线程就能在不同抽象层级上，把游戏线程的数据和操作转交给 RHI 线程。

**FRHIResource**

`FRHIResource` 用于引用计数、延迟删除、跟踪、运行时数据以及标记。`FRHIResource` 可以细分为状态块、shader 绑定、shader、管线状态、缓冲、纹理、视图和其他杂项。需要注意的是，我们可以用这个类创建平台特定类型；可以查看 `FRHIUniformBuffer` 的源码。

```cpp
/** The base type of RHI resources. */
class RHI_API FRHIResource
{
public:
	UE_DEPRECATED(5.0, "FRHIResource(bool) is deprecated, please use FRHIResource(ERHIResourceType)")
	FRHIResource(bool InbDoNotDeferDelete=false)
	: ResourceType(RRT_None)
	, bCommitted(true)
#if RHI_ENABLE_RESOURCE_INFO
	, bBeingTracked(false)
#endif
{
}
FRHIResource(ERHIResourceType InResourceType)
	: ResourceType(InResourceType)
	, bCommitted(true)
#if RHI_ENABLE_RESOURCE_INFO
	, bBeingTracked(false)
#endif
{
#if RHI_ENABLE_RESOURCE_INFO
	BeginTrackingResource(this);
#endif
}
virtual ~FRHIResource()
{
	check(IsEngineExitRequested() || CurrentlyDeleting == this);
	check(AtomicFlags.GetNumRefs(std::memory_order_relaxed) == 0); // this should not have any outstanding refs
	CurrentlyDeleting = nullptr;
	#if RHI_ENABLE_RESOURCE_INFO
	EndTrackingResource(this);
	#endif
}
	FORCEINLINE_DEBUGGABLE uint32 AddRef() const
	{...};
private:
	// Separate function to avoid force inlining this everywhere. Helps both for code size and performance.
	inline void Destroy() const
	{...};
public:
	FORCEINLINE_DEBUGGABLE uint32 Release() const
	{...};
	FORCEINLINE_DEBUGGABLE uint32 GetRefCount() const
	{...};
	static int32 FlushPendingDeletes(FRHICommandListImmediate& RHICmdList);
	static bool Bypass();
	bool IsValid() const
	{...};
	void Delete()
	{...};
	inline ERHIResourceType GetType() const { return ResourceType; }
	#if RHI_ENABLE_RESOURCE_INFO
	// Get resource info if available.
	// Should return true if the ResourceInfo was filled with data.
	virtual bool GetResourceInfo(FRHIResourceInfo& OutResourceInfo) const
	{...};
	static void BeginTrackingResource(FRHIResource* InResource);
	static void EndTrackingResource(FRHIResource* InResource);
	static void StartTrackingAllResources();
	static void StopTrackingAllResources();
#endif
private:
	class FAtomicFlags
	{
	static constexpr uint32 MarkedForDeleteBit = 1 << 30;
	static constexpr uint32 DeletingBit = 1 << 31;
	static constexpr uint32 NumRefsMask = ~(MarkedForDeleteBit | DeletingBit);
	std::atomic_uint Packed = { 0 };
	public:
	int32 AddRef(std::memory_order MemoryOrder)
	{...};
	int32 Release(std::memory_order MemoryOrder)
	{...};
	bool MarkForDelete(std::memory_order MemoryOrder)
	{...};
	bool UnmarkForDelete(std::memory_order MemoryOrder)
	{...};
	bool Deleteing()
	{...};
	mutable FAtomicFlags AtomicFlags;
	const ERHIResourceType ResourceType;
	uint8 bCommitted : 1;
	#if RHI_ENABLE_RESOURCE_INFO
	uint8 bBeingTracked : 1;
	
#endif
	static std::atomic<TClosableMpscQueue<FRHIResource*>*> PendingDeletes;
	static FHazardPointerCollection PendingDeletesHPC;
	static FRHIResource* CurrentlyDeleting;
// Some APIs don't do internal reference counting, so we have to wait an extra couple of frames before deleting resources
// to ensure the GPU has completely finished with them. This avoids expensive fences, etc.
	struct ResourcesToDelete
	{...};
};
```

**FRHICommandList**

RHI Command List 是一个指令队列，用于管理和执行一组命令对象。它的父类是 `FRHICommandListBase`。`FRHICommandListBase` 定义了命令队列所需的基础数据（命令列表、设备上下文）和接口（命令刷新、等待、入队、内存分配等）。`FRHIComputeCommandList` 定义了计算 shader、GPU 资源状态转换以及 shader 参数设置之间的接口。`FRHICommandList` 则定义了常规渲染管线接口，包括绑定 `Vertex Shaders`、`Pixel Shaders`、`Geometry Shaders`，图元绘制、shader 参数设置和资源管理等。

**RHIContext 与 DynamicRHI**

最后，`RHIContext` 和 `DynamicRHI` 也是一组接口类，用来定义图形 API 相关操作。前面提到过，一些 API 可以并行处理命令，因此 Unreal 使用这些独立对象来表达对应能力。

总结来说，RHI 类是 Unreal 与图形 API 通信时使用的最低层抽象。上面列出的几个类是你最应该了解的核心类型。我在参考资料章节中放了一篇更深入介绍 RHI 的文章。如果 command list 还不太好理解，可以先查阅基本的 command list / command buffer 是如何提交给 GPU 的，也可以参考任意一种图形 API 中命令入队与提交的流程。
