# DirectX12初始化

这几天跟着龙书把dx12的初始化过了一遍，写点东西记一下，免得之后又忘了。



## 创建d3d设备

d3d设备相当于对显示适配器的抽象，显示适配器一般为显卡，也可由软件来模拟。可通过下列接口来创建一个d3d设备：

```C++
RESULT WINAPI D3D12CreateDevice (
	Unknown* pAdapter,
	D3D_FEATURE_LEVEL MinimumFeatureLevel, 
 	REFIIDriid, // ID3D12Device的COMID
	void** ppDevice ) ;
```

第一个参数为指定的适配器，可通过`IDXGIFactory::EnumWarpAdapter`接口来枚举当前设备拥有的适配器，当传入nullptr时，使用默认适配器。若创建硬件设备失败，可通过`IDXGIFactory4::EnumWarpAdapter`接口来枚举软件适配器并创建。在dx11中需要在创建设备中使用D3D_DRIVER_TYPE结构体指定设备类型，并在指定使用软光栅器时需要传入句柄，dx12中则将软适配器的责任转接给IDXGIFactory4，由其枚举支持的软适配器；第三和第四个参数可使用宏`IID_PPV_ARGS`来传入，其定义如下：

```C++
#define IID_PPV_ARGS(ppType) __uuidof(**(ppType)), IID_PPV_ARGS_Helper(ppType)
```

该宏通过`__uuidof(**(ppType))`来获取OM接口ID，同时使用`IID_PPV_ARGS_Helper(ppType)`将传入的指针强转为`void**`.

创建设备后即可使用设备创建同步CPU和GPU的围栏，由于dx12中使用了延时渲染，应用了命令列表命令队列模型，通过命令列表来记录命令，并提交到命令队列中等待GPU执行，两个处理器并行工作时通过栅栏来实现同步。可通过下列接口创建一个围栏对象：

```C++
HRESULT CreateFence(
        UINT64            InitialValue,	// 围栏初始值
        D3D12_FENCE_FLAGS Flags,
        REFIID            riid,
  [out] void              **ppFence
);
```

随后便是获取描述符堆的大小，由于不同的GPU平台描述符的大小不同，因此需要提前获取。为避免反复查询大小，可直接通过`ID3D12Device：：GetDescriptorHandleIncrementSize`来获取大小并缓存起来。



## 创建命令队列和命令列表

为了让并行工作的CPU和GPU进行，DX12使用`ID3D12CommandQueue`接口来抽象GPU维护的命令队列，并通过`ID3D12CommandList`来将命令打包起来提交到命令队列中，可通过下列方法创建一个命令队列：

```C++
HRESULT CreateCommandQueue(
  const D3D12_COMMAND_QUEUE_DESC *pDesc,	// 命令队列的描述
  REFIID                         riid,
  void                           **ppCommandQueue
);
```

通常我们不直接使用`ID3D12CommandList`，而是使用继承自他的`ID3D12GraphicsCommandList`，可通过下列方法创建命令列表：

```C++
HRESULT CreateCommandList(
  [in]           UINT                    nodeMask,
  [in]           D3D12_COMMAND_LIST_TYPE type,
  [in]           ID3D12CommandAllocator  *pCommandAllocator,
  [in, optional] ID3D12PipelineState     *pInitialState,
  [in]           REFIID                  riid,
  [out]          void                    **ppCommandList
);
```

当我们创建或重置一个命令列表后，其默认保持打开状态，需要调用`[ID3D12GraphicsCommandList：：Close]`方法将其关闭。`ID3D12GraphicsCommandList`提供了一系列的图形渲染命令供我们使用。调用命令列表的相关接口后GPU并不会立刻执行命令，需要先将列表关闭，后调用`ID3D12CommandQueue：：ExecuteCommandLists`将命令列表提交到命令队列中，当然提交后也不会立即执行，需要GPU将前面的命令执行完后才会执行新提交的命令，因此DX12才需要加入`ID3D12Fence`来管理CPU和GPU的同步。

命令列表中的命令其实**并不是储存在命令列表之中**的，而是储存在命令分配器`ID3D12CommandAllocator`之中，`ID3D12Device：：CreateCommandList`方法中的第三个参数便是该列表关联的命令分配器，可通过下列方法创建一个命令分配器：

```C++
HRESULT CreateCommandAllocator(
  [in]  D3D12_COMMAND_LIST_TYPE type,
        REFIID                  riid,
  [out] void                    **ppCommandAllocator
);
```

调用`ID3D12CommandQueue：：ExecuteCommandLists`执行命令时命令队列也是从分配器中引用命令，因此当我们希望重新向命令队列中写入命令时，不仅要调用`ID3D12GraphicsCommandList：：Reset`来将命令列表恢复为初始状态并复用内存，还需要调用`ID3D12CommandAllocator：：Reset`来将命令分配器清空，但是值得注意的是，**不能在GPU执行完分配器中的命令之前重置分配器**，否则命令队列会丢失对分配器中数据的引用。同时多个命令列表可以关联到同一个分配器中，但一次只能开启一个列表，并按顺序的添加命令。

有了命令队列和围栏对象，就可以实现一个最基本的CPU和GPU同步函数。

```c++
// 推进栅栏值，将命令标记到此栅栏点
++m_CurrentFence;

// 添加一条指令到命令队列来设置一个新的栅栏点。
// 因为我们在GPU时间轴上，直到GPU完成处理这个Signal（）之前的所有命令,新的栅栏点将不会被设置.
ThrowIfFailed(m_CommandQueue->Signal(m_D3D12Fence.Get(), m_CurrentFence));

// 等待GPU完成这个栅栏点的命令。
if (m_D3D12Fence->GetCompletedValue() < m_CurrentFence) {
    HANDLE eventHandle = CreateEvent(nullptr, false, false, nullptr);

    // 当GPU碰到当前栅栏时触发事件
    ThrowIfFailed(m_D3D12Fence->SetEventOnCompletion(m_CurrentFence, eventHandle));

    // 等待直到GPU触发当前栅栏事件
    if (eventHandle != 0) {
        WaitForSingleObject(eventHandle, INFINITE);
        CloseHandle(eventHandle);
    }
}

```

首先便是推进围栏维护的围栏值，随后调用`ID3D12CommandQueue：：Signal`方法在命令列表中添加新的围栏值，在GPU处理完前面的命令触发新注册的围栏之前，`ID3D12Fence：：GetCompletedValue`方法会一直返回推进前的围栏值，，因此`m_D3D12Fence->GetCompletedValue() < m_CurrentFence`也就代表GPU未处理完先前分派的指令，此时创建一个事件，并在GPU触碰到围栏时触发事件，在触发事件之前`WaitForSingleObject`方法便会一直等待，最后便关闭事件对象的句柄。



## 描述并创建交换链

为了避免直接在前台缓冲区绘制而出现画面闪烁，通常先将画面渲染在离屏的后台缓冲区，再与前台缓冲区交换。而交换链即由前台缓冲区和后台缓冲区构成，使用两个缓冲区叫双缓冲，三个即叫三重缓冲。

要创建交换链，需要先填写描述交换链属性的结构体，龙书中使用的是旧版的`IDXGIFactory：：CreateSwapChain	`来创建交换链，我查看MSDN后发现文档中这么写‘ [从 Direct3D 11.1 开始，建议不要再使用 **CreateSwapChain** 来创建交换链。 请改用 [CreateSwapChainForHwnd](https://learn.microsoft.com/zh-cn/windows/desktop/api/dxgi1_2/nf-dxgi1_2-idxgifactory2-createswapchainforhwnd)、 [CreateSwapChainForCoreWindow](https://learn.microsoft.com/zh-cn/windows/desktop/api/dxgi1_2/nf-dxgi1_2-idxgifactory2-createswapchainforcorewindow) 或 [CreateSwapChainForComposition](https://learn.microsoft.com/zh-cn/windows/desktop/api/dxgi1_2/nf-dxgi1_2-idxgifactory2-createswapchainforcomposition) ，具体取决于要如何创建交换链。] ’这三个方法都使用了`DXGI_SWAP_CHAIN_DESC1`而不是旧版的`DXGI_SWAP_CHAIN_DESC`，因此这里介绍`DXGI_SWAP_CHAIN_DESC1`。新的三个创建方法将全屏模式需要的参数单独从`DXGI_SWAP_CHAIN_DESC `之中抽离了出来，单独使用`DXGI_SWAP_CHAIN_FULLSCREEN_DESC `结构体来描述全屏属性。这里我使用`IDXGIFactory2：：CreateSwapChainForHwnd`方法来创建交换链，其具体参数如下：
```C++
HRESULT CreateSwapChainForHwnd(
  [in]           IUnknown                              *pDevice,
  [in]           HWND                                  hWnd,
  [in]           const DXGI_SWAP_CHAIN_DESC1           *pDesc,
  [in, optional] const DXGI_SWAP_CHAIN_FULLSCREEN_DESC *pFullscreenDesc,
  [in, optional] IDXGIOutput                           *pRestrictToOutput,
  [out]          IDXGISwapChain1                       **ppSwapChain
);
```

其中第二个参数为使用交换链的窗口的句柄；第三个参数的结构体定义如下：

```c++
typedef struct DXGI_SWAP_CHAIN_DESC1 {
  UINT             Width;
  UINT             Height;
  DXGI_FORMAT      Format;		// 显示格式，通常为DXGI_FORMAT_R8G8B8A8_UNORM
  BOOL             Stereo;
  DXGI_SAMPLE_DESC SampleDesc;	//多采样参数，样本数Count 质量级别Quality;
  DXGI_USAGE       BufferUsage;	// 后台缓冲区的图面使用情况和 CPU 访问选项
  UINT             BufferCount;	// 缓冲区数量
  DXGI_SCALING     Scaling;		// 缓冲区大小调整行为
  DXGI_SWAP_EFFECT SwapEffect;	// 交换链使用的演示模型
  DXGI_ALPHA_MODE  AlphaMode;	// 后缓冲区的透明度行为
  UINT             Flags;
} DXGI_SWAP_CHAIN_DESC1;
```

注意**DX12不支持创建MSAA的交换链**，若希望支持MSAA，这里的**多采样参数的数量和质量依旧为1和0**。

第四个参数专门用于全屏模式，若不开启全屏模式，可传入`nullptr`，该结构体定义如下：
```c++
typedef struct DXGI_SWAP_CHAIN_FULLSCREEN_DESC {
  DXGI_RATIONAL            RefreshRate;			// 刷新率
  DXGI_MODE_SCANLINE_ORDER ScanlineOrdering;	// 扫描线绘制模式
  DXGI_MODE_SCALING        Scaling;				// 指定图像如何拉伸来适应屏幕
  BOOL                     Windowed;			// 窗口模式
} DXGI_SWAP_CHAIN_FULLSCREEN_DESC;
```

填充完描述结构体后即可调用下列方法创建交换链：
```C++
HRESULT CreateSwapChainForHwnd(
  [in]           IUnknown                              *pDevice,			// 在先前版本中这里传入的是D3D设备的指针，DX12中直接传入命令队列的指针
  [in]           HWND                                  hWnd,				// 与交换链关联的窗口句柄
  [in]           const DXGI_SWAP_CHAIN_DESC1           *pDesc,
  [in, optional] const DXGI_SWAP_CHAIN_FULLSCREEN_DESC *pFullscreenDesc,
  [in, optional] IDXGIOutput                           *pRestrictToOutput,
  [out]          IDXGISwapChain1                       **ppSwapChain
);
```



## 创建描述符堆即资源视图

由于GPU资源本质上都是一块普通的内存块，因此就需要一个中间层来让GPU在处理这块内存时得到这块内存的必要属性，而在DX中这个中间层便是**描述符**或称为资源视图。描述符中存放了对应资源的引用和该引用资源的属性，通过更改描述符，我们可以让GPU以不同的方式处理同一个资源，如可以让一个纹理资源作为一次绘制的渲染目标，随后又作为着色器资源绑定到渲染管线中。

而描述符堆即是存放描述符的一块内存，可为每种描述符创建单独的描述符，也可以为同一种描述符创建多个描述符堆。我们需要分别为渲染目标视图和深度模板视图创建描述符堆。创建描述符堆之前需要填写描述描述符堆的结构体，其定义如下：

```C++
typedef struct D3D12_DESCRIPTOR_HEAP_DESC {
  D3D12_DESCRIPTOR_HEAP_TYPE  Type;				// 描述符堆的类型
  UINT                        NumDescriptors;	// 堆中的描述符数
  D3D12_DESCRIPTOR_HEAP_FLAGS Flags;
  UINT                        NodeMask;
} D3D12_DESCRIPTOR_HEAP_DESC;
```

随后调用`ID3D12Device：：CreateDescriptorHeap`方法创建描述符堆 。

创建描述符堆后便可创建描述符视图，由于每次更改窗口大小都需要重新更改交换链和深度模板缓冲，描述符也要跟着重新设置，因此这些操作都可以放在`OnResize`方法中，每次更改窗口大小时调用。

首先需要等待GPU完成命令队列中的命令，然后置空命令列表并释放先前的后台缓冲区和深度模板缓冲区，随后重置交换链大小并重置后台缓冲区的索引。

```C++
FlushCommandQueue();

ThrowIfFailed(m_CommandList->Reset(m_DirectCmdListAlloc.Get(), nullptr));

// 释放先前创建的资源
for (int i = 0; i < SwapChainBufferCount; ++i) {
	m_SwapChainBuffer[i].Reset();
}
m_DepthStencilBuffer.Reset();

// 更改交换链大小
ThrowIfFailed(m_DxgiSwapChain->ResizeBuffers(
    SwapChainBufferCount,
    m_ClientWidth,
    m_ClientHeight,
    m_BackBufferFormat,
    DXGI_SWAP_CHAIN_FLAG_ALLOW_MODE_SWITCH));
m_CurrBackBuffer = 0;
```

释放后台缓冲区后便需要重新创建后台缓冲区并绑定给交换链。

```C++
// 创建渲染目标视图
CD3DX12_CPU_DESCRIPTOR_HANDLE rtvHandle(m_RtvHeap->GetCPUDescriptorHandleForHeapStart());
for (UINT i = 0; i < SwapChainBufferCount; ++i) {
    // 获取交换链缓冲区
    ThrowIfFailed(m_DxgiSwapChain->GetBuffer(i, IID_PPV_ARGS(m_SwapChainBuffer[i].GetAddressOf())));
    m_D3D12Device->CreateRenderTargetView(m_SwapChainBuffer[i].Get(), nullptr, rtvHandle);
    // 偏移到描述符堆的下一个缓冲区
    rtvHandle.Offset(1, m_RtvDescriptorSize);
}
```

深度模板缓冲区也需要重新创建。

```C++
// 创建深度/模板缓冲区即视图
D3D12_RESOURCE_DESC depthStencilDesc{};
depthStencilDesc.MipLevels = 1;
depthStencilDesc.Alignment = 0;
depthStencilDesc.DepthOrArraySize = 1;
depthStencilDesc.Width = m_ClientWidth;
depthStencilDesc.Height = m_ClientHeight;
depthStencilDesc.Format = DXGI_FORMAT_R24G8_TYPELESS;
depthStencilDesc.Layout = D3D12_TEXTURE_LAYOUT_UNKNOWN;
depthStencilDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
depthStencilDesc.Flags = D3D12_RESOURCE_FLAG_ALLOW_DEPTH_STENCIL;
depthStencilDesc.SampleDesc.Count = 1;
depthStencilDesc.SampleDesc.Quality = 0;

// 清除资源
D3D12_CLEAR_VALUE clearValue{};
clearValue.Format = m_DepthStencilFormat;
clearValue.DepthStencil.Depth = 1.f;
clearValue.DepthStencil.Stencil = 0;
auto heapProperties = CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_DEFAULT);
// 创建缓冲区
ThrowIfFailed(m_D3D12Device->CreateCommittedResource(
    &heapProperties,
    D3D12_HEAP_FLAG_NONE,
    &depthStencilDesc,
    D3D12_RESOURCE_STATE_COMMON,
    &clearValue,
    IID_PPV_ARGS(m_DepthStencilBuffer.GetAddressOf())));
```

还需要为深度模板缓冲区重新船舰描述符堆：
```C++
// 创建第0mip层描述符
D3D12_DEPTH_STENCIL_VIEW_DESC dsvDesc{};
dsvDesc.Flags = D3D12_DSV_FLAG_NONE;
dsvDesc.ViewDimension = D3D12_DSV_DIMENSION_TEXTURE2D;
dsvDesc.Format = m_DepthStencilFormat;
dsvDesc.Texture2D.MipSlice = 0;
m_D3D12Device->CreateDepthStencilView(
    m_DepthStencilBuffer.Get(),
    &dsvDesc,
    GetDepthStencilView());
```

最后将资源的状态转换为深度缓冲区并提交命令列表等待GPU执行：
```C++
auto barrier = CD3DX12_RESOURCE_BARRIER::Transition(
    m_DepthStencilBuffer.Get(),
    D3D12_RESOURCE_STATE_COMMON,
    D3D12_RESOURCE_STATE_DEPTH_WRITE);
// 将资源从初始状态转换为深度缓冲区
m_CommandList->ResourceBarrier(1, &barrier);

ThrowIfFailed(m_CommandList->Close());

// 提交调整大小命令
ID3D12CommandList* cmdsLists[] = { m_CommandList.Get() };
m_CommandQueue->ExecuteCommandLists(_countof(cmdsLists), cmdsLists);

// 在完成前等待
FlushCommandQueue();
```

更改窗口的大小同样需要城改视口和裁剪矩形的大小：
```C++
m_ScreenViewport.TopLeftX = 0;
m_ScreenViewport.TopLeftY = 0;
m_ScreenViewport.Width = static_cast<float>(m_ClientWidth);
m_ScreenViewport.Height = static_cast<float>(m_ClientHeight);
m_ScreenViewport.MinDepth = 0.0f;
m_ScreenViewport.MaxDepth = 1.0f;

m_ScissorRect = { 0,0,m_ClientWidth,m_ClientHeight };
```

自此D3D12便初始化好了。
