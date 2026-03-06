# DirectX RayTracing (1) : 绘制第一个三角形

​	有好一段时间没写文了，暑假前疯狂找实现，结果大败而归，道心破碎，狠狠的摆烂了一个暑假，上一篇 SSR 也是写到一半就搁置了，开学后才重新拾起来。最近准备学学 DXR ，因此把学习过程写下来记录一下。

​	我是通过直接阅读微软官方示例的源码来学习，同时阅读 DirectX 规范的RayTracing部分。顺带一提本文的代码堆 DX12 的原生 API 进行了一点封装，主要是参考了 DirectX-Graphics-Samples 中的MiniEngine，封装的较为简单，因此不会影响整体代码的阅读。



## RayTracing原理

​	在介绍DXR之前先简单描述一下光追的原理。光追的原理也十分简单，大致分为以下几个步骤：

1. 从相机处向视口的每一个像素**构造一根光线**。
2. **遍历场景**中的所有几何体，**判断**光线是否与几何体**相交**。
3. 若相交且在光线的范围内则更新**光追负载和光线的TMax**，否则退出光追。
4. 处理完所有相交后，根据光追负载所携带的属性及光线的性质判断是否要在交点处发射其他光线。
5. 最后计算当前光线所对应像素的着色。

相比光栅化，光追是一种更符合直觉的算法，如 RTR4 里说的一样，两者都可以描述为**双重循环**：

```C++
// 光栅化
for (T in triangles )
    for (P in pixels )
        determine if P is inside T
// 光追
for (P in pixels )
    for (T in triangles )
        determine if ray through P hits T
```

两者的意思分别是：遍历场景中的所有几何体，并找到其所覆盖的所有像素并进行着色；遍历所有像素，从该像素处发射一根光线并查找所有与光线相交的交点。由于当今的光栅化的各种优化算法已经非常成熟，且硬件对光栅化的支持良好，光追更多是作为光栅化的辅助渲染。



## 光追流程

在介绍详细流程之前，先速览一下整个光追流程：

<img src="https://img2024.cnblogs.com/blog/3406761/202509/3406761-20250904214553472-293474492.png" alt="image" style="zoom: 50%;" />

对该流程图的解读大致如下：

1. 在 Ray Generation Shader 中**构造光线**并**发送追踪光线的请求**，及调用`TraceRay`，最后将输出的颜色保存到输出的纹理中。
2. 光线会**遍历加速结构**，并检测光线与几何体是否相交，此时会**判断几何体是否是三角形**，若是则会使用内置的三角形求交，否则会调用用户自定义的 **Intersection Shader**进行求交检测。
3. 若未检测到相交则会继续遍历加速结构，否则会判断相交的几何体**是否是透明**的，由于透明的几何体需要进行特殊处理，所以会调用 **Any Hit Shader**，而不透明的几何体则会直接记录相交并**更新光线的TMax**。
4. 遍历完加速结构后，检测是否有相交的几何体，若有则会调用 **Closest Hit Shader**并传递光追负载和交点属性，否则会调用 **Miss Shader**。最后结束光追。

先前的大致解读提到了五个着色器，这五个着色器也是光追管线的主要构成，现在来详细介绍一下各自的主要功能：

### Ray Generation Shader 

​	光追管线最先执行的着色器，使用命令列表调用`DispatchRays`后驱动层会唤起相应数量的 Ray Generation Shader，一般由该着色器构造光线并调用`TraceRay`来追踪光线，最后也是由该来将光线输出的颜色写入纹理中。这里提到了一个`TraceRay`函数，整个光追的核心就是这个函数，其定义如下：

```hlsl
Template<payload_t>
void TraceRay(RaytracingAccelerationStructure AccelerationStructure,
              uint RayFlags,
              uint InstanceInclusionMask,
              uint RayContributionToHitGroupIndex,
              uint MultiplierForGeometryContributionToHitGroupIndex,
              uint Miss ShaderIndex,
              RayDesc Ray,
              inout payload_t Payload);
```

其各个参数的解释如下：

1. AccelerationStructure
   指定使用**顶层加速结构**，作为 **SRV** 绑定到管线中，若为 NULL 则会判断未命中，何为顶层加速结构后续介绍，可以先理解为一个普通的 BVH。
2. RayFlags
   光线的标识，不同的光线标识会让光线有不同的行为，例如`RAY_FLAG_FORCE_OPAQUE`会强制判断几何体为不透明的，从而不会调用 Any Hit Shader；而`RAY_FLAG_CULL_BACK_FACING_TRIANGLES`会剔除朝后的三角形。
3. InstanceInclusionMask
   该掩码会对相交的几何体进行剔除，在构造顶层加速结构时会填写一个叫`InstanceMask`的参数，只有两者取 & 后不为0才会进行相交检测。
4. RayContributionToHitGroupIndex
5. MultiplierForGeometryContributionToHitGroupIndex
   这两个参数我也是迷惑了很久，看了半天 MSDN 简陋的解释也没看出个所以然，最后还是看了别的大佬的文章才搞清楚。这两个参数分别表示的是**当前光线的类型的索引**和**光线类型的数量**，要搞清楚这两个的意思还需要了解其他概念，因此后续再解释。
6. Miss ShaderIndex
   所使用的 Miss Shader 的索引。
7. Ray
   光线本尊，其类型为`RayDesc`，定义如下：

   ```hlsl
   struct RayDesc
   {
       float3 Origin;		// 光线的起点
       float  TMin;		// 光线的最近处
       float3 Direction;	// 光线的方向
       float  TMax;		// 光线的最远处
   };
   ```

   相信听名字也能知道各个参数的意思。
8. Payload
   光追负载，由用户**自定义的类型**，在配置光追管线的时候需要**指定其大小**。

值得一提的是上图的五个函数中，除了 Intersection Shader 和 Any Hit Shader，其他 Shader 都可以调用这个内置函数，不过过多的`TraceRay`调用会带来极大的负担。

### Closest Hit Shader

​	光线遍历加速结构后且有相交记录就会执行该着色器，其调用发生在**遍历完所有加速结构之后**。一般用于对光线击中的表面进行最后的着色，`TraceRay`函数也通常在该 Shader 中调用，用于追踪反射光线。该 Shader 的大致实例如下：

```hlsl
[shader("closesthit")]
void ClosestHitShader(inout MyPayload payload, in MyAttributes attr)
{
}
```

其参数分别有两个：

1. **光线负载**，用户自定义的类型，可对其进行**读写**。
2. **最近的交点属性**，可以是自定义类型，也可以是内置类型`BuiltInTriangleIntersectionAttributes`，包含交点的重心坐标。该参数只**可读**。

在该 Shader 中可通过调用`WorldRayOrigin`、`WorldRayDirection`、`RayTCurrent`这三个函数来获取交点的坐标。

### Miss Shader

​	当**遍历完加速结构后**，且没有相交记录的时候会调用该函数，该 Shader 通常用来渲染背景颜色或渲染天空。
```hlsl
[shader("miss")]
void MissShader(inout MyPayload payload)
{
}
```

由于其与几何体无关，因此没有交点属性作为参数只可**读写**光追负载。

### Any Hit Shader

​	该着色器仅当几何体是**透明**的时候调用，且在遍历加速结构的时候调用，也就是在 Closest Hit Shader 之前调用，通常用来处理透明物体。其函数签名大致如下：

```hlsl
[shader("anyhit")]
void AnyHitShader( inout MyPayload payload, in MyAttributes attr )
{
}
```

其参数与 Closest Hit Shader 相同，不过其多了几个可以调用的内置函数，分别为：

1. `AcceptHitAndEndSearch`，调用该函数后会直接**结束** Any Hit Shader，并**终止**对加速结构的遍历，由于有相交记录，因此会直接调用 Closest Hit Shader 。
2. `IgnoreHit`,调用该函数同样会结束该 Shader，但会**继续遍历**加速结构。

### Intersection Shader

​	该 Shader 用于实现自定义类型的相交，当光线在加速结构中查找到一个相交的**叶子节点**的时候，就会执行图元的相交测试，对于内置的三角形类型，会测试相交并计算**重心坐标**，对于自定义图元，则会查找改图元是否有 Intersection Shader，若有则会执行对应的 Intersection Shader。相比其他着色器，其还可以执行一个内置函数`ReportHit`，用来提交相交测试，其函数签名如下：

```hlsl
template<attr_t>
bool ReportHit(float THit, uint HitKind, attr_t Attributes);
```

1. THit，交点与光线起点的距离。
2. HitKind，用于标识发生的命中类型，**Hit Group**（Intersection Shader、Any Hit Shader 与 Closest Hit Shader） 中的 Shader 都可以通过调用 `HitKind`函数来获取这个值，例如内置的三角形图元通过将 HitKind 设置为`HIT_KIND_TRIANGLE_FRONT_FACE`254或 `HIT_KIND_TRIANGLE_BACK_FACE`255来区分三角形的正面和反面。
3. Attributes，与上述的交点属性相同。

调用该函数后会返回一个 bool 类型的值来将 Hit 的状态反馈给 Intersection Shader，分别为**接受**与**忽略**，当传入的 THit **不在光线的TMin/TMax 范围内**时或是 Any Hit  Shader 通过调用 **IgnoreHit **来退出时会将该相交忽略；否则就会接受该提交并将 **TMax 更新为 THit**，同时若是 Any Hit  Shader 调用了 AcceptHitAndEndSearch 还会在接受提交的同时**终止** Intersection Shader 的运行。



## Shader Table

​	前面介绍了光追的整体流程，及光追管线中运行的 Shader 和他们运行的顺序与特性。这时又迎来一个新的问题，就是如何指定与配置在管线中使用的 Shader，这是就要引入 Shader Table 及相关的一些概念。这里先放一张描述这几个概念之间的关系的图，可参照着来辅助阅读：

![image](https://img2024.cnblogs.com/blog/3406761/202509/3406761-20250906180801105-1282295013.png)

### Shader Table 的组成

#### HitGroup

​	与光栅化每次 Draw 调用一个着色器组(VS、PS之类的)不同，光追管线是每个几何体对应一个着色器组，而这里的着色器组指的是 **Intersection Shader** 、**Closest Hit Shader** 和 **Any Hit Shader**。这三个着色器统称为 **Hit Group**，而 Miss Shader 与 Ray Generation Shader 与几何体无关，因此他们不包含在内。

#### Shader Record

​	每一个 Hit Group 又通过 Shader Identifier（可以理解为指向着色器组的指针） 被包含在 **Shader Record** 中，用来处理光线与几何体相交，可通过如下方法来获取 Shader Identifier：

```C++
Microsoft::WRL::ComPtr<ID3D12StateObjectProperties> stateObjectProps{};
ASSERT_SUCCEEDED(m_RayTracingStateObject.As(&stateObjectProps));
void* rayGenShaderIdentifier = stateObjectProps->GetShaderIdentifier(Renderer::s_RayGenShaderName);
void* missShaderIdentifier = stateObjectProps->GetShaderIdentifier(Renderer::s_MissShaderName);
void* hitGroupIdentifier = stateObjectProps->GetShaderIdentifier(Renderer::s_HitGroupName);
```

其中 stateObjectProps 是类型为` ID3D12StateObject`的对象，也就是 PSO，而`GetShaderIdentifier`的实参为在为 PSO 绑定 Shader 时所使用的着色器的**导出名**，具体细节后续讨论。

​	除了 Hit Group 外，Shader Record 还包含了**根参数**，没错就是根签名的根参数。至于为何需要根参数，前面提到过光追管线的着色器是 Pre-Geometry 的，也就是说一次 DispatchRay 就需要为所有的几何体绑定好纹理之类的资源，不能像图形管线一样每个几何体调用 Draw 并绑定不同的根实参来绑定不同的资源。

​	由此可见一个 Shader Record 定义了一种渲染方式。而 Shader Table 就是一个储存一系列 Shader Record 的数组，在执行光追管线的时候，GPU 就通过某种索引方式在 Shader Table 中找到需要执行的 Shader Record，并设置 Shader Record 中的根参数，最后执行 Shader。而 Shader Table 又分为四种，分别对应了四种着色器，他们分别为：

1. Ray Generation Shader Table，包含了 Ray Generation Shader Record，一般只储存一个。
2. Hit Group Shader Table，包含了多个 Hit Group Shader Record，索引方式最为复杂。
3. Miss Shader Table，包含了多个 Miss Shader Record，索引方式最为简单，只需要在 TraceRay 中的参数直接指定即可。
4. Call Shader Table，包含了多个 Callable Shader Record，索引方式也是直接在函数参数中指定索引，这个 Shader 使用较少，后续使用的时候再介绍。

Shader Table 还有如下两个限制

1. 其首地址必须与 64 字节（D3D12_RAYTRACING_SHADER_TABLE_BYTE_ALIGNMENT）对齐。
2. 每个 Shader Record 之间的间隔必须为最大的 Shader Record 的大小。 

### 在 Shader Table 中索引着色器

​	介绍完 Shader Table 之后便是说明索引着色器的机制，这里先放一张描述 Shader Table 组成的图，方便后续理解。![image](https://img2024.cnblogs.com/blog/3406761/202509/3406761-20250907154436988-475900431.png)

Shader Table 的组成主要由以下三个因素影响：

1. 加速结构中的**实例数**。这里的实例指的是顶层加速结构的叶子数量。
2. 每个实例中的**几何数**。每个实例中可能包含了多个几何体。
3. 光追管线中所使用的**光线的种类**的数量。例如想要渲染同一个几何体本身及其阴影，就需要使用两套算法，分别代表了一根 Ray 和一根 Shadow Ray。

由上述的描述可见着色器表中 Shader Record 的总数为`NumGeometry * NumRay `。例如上图就描述了两个实例，每个实例分别有一个和两个几何体，HG0 到 HG1 为第一个实例的第一个几何体的两种 Ray 的 Shader Record，HG2 到 HG3 为第二个实例的第一个几何体的两种 Ray，HG4 到 HG5 为第二个实例的第二个几何体的两种 Ray。由此单个几何体的索引可以描述为`(OffsetFromShaderTable + RayType + RayTypeCount * GeometryIndexInInstance)`。官方提供的计算公式如下：

```C++
HitGroupRecordAddress =
D3D12_DISPATCH_RAYS_DESC.HitGroupTable.StartAddress + // from: DispatchRays()
D3D12_DISPATCH_RAYS_DESC.HitGroupTable.StrideInBytes * // from: DispatchRays()
(
RayContributionToHitGroupIndex + // from shader: TraceRay()
(MultiplierForGeometryContributionToHitGroupIndex * // from shader: TraceRay()
GeometryContributionToHitGroupIndex) + // system generated index of geometry in bottom-level acceleration structure (0,1,2,3..)

D3D12_RAYTRACING_INSTANCE_DESC.InstanceContributionToHitGroupIndex // from instance
)
```

其中`RayContributionToHitGroupIndex`与`MultiplierForGeometryContributionToHitGroupIndex`就是先前提到的`TraceRay`的两个参数，分别表示**光线的类型**和**光线类型的数量**。而`InstanceContributionToHitGroupIndex`需要再创建加速结构的时候自行指定，后续详细介绍。

### Shader Table 的创建

​	要绘制一个三角形只需要构造光线并直接在 Closest Hit Shader 中返回颜色即可，因此需要使用局部根签名的只有 Ray Generation Shader ，包含了一个根常量，为创建光线时需要的视口信息。具体创建流程如下：
```C++
// 创建着色器表
// 获取 Shader 的标识符
Microsoft::WRL::ComPtr<ID3D12StateObjectProperties> stateObjectProps{};
ASSERT_SUCCEEDED(m_RayTracingStateObject.As(&stateObjectProps));
void* rayGenShaderIdentifier = stateObjectProps->GetShaderIdentifier(Renderer::s_RayGenShaderName);
void* missShaderIdentifier = stateObjectProps->GetShaderIdentifier(Renderer::s_MissShaderName);
void* hitGroupIdentifier = stateObjectProps->GetShaderIdentifier(Renderer::s_HitGroupName);

// RayGeneration 着色器表
m_RayGenCB.viewport = { -1.0f, -1.0f, 1.0f, 1.0f };
GpuBufferDesc rayGenShaderTableDesc{};
rayGenShaderTableDesc.m_Size = D3D12_SHADER_IDENTIFIER_SIZE_IN_BYTES + sizeof(m_RayGenCB);
rayGenShaderTableDesc.m_Stride = rayGenShaderTableDesc.m_Size;
rayGenShaderTableDesc.m_HeapType = D3D12_HEAP_TYPE_UPLOAD;
std::vector<uint8_t> rayGenShaderTableData(rayGenShaderTableDesc.m_Size);
memcpy(rayGenShaderTableData.data(), rayGenShaderIdentifier, D3D12_SHADER_IDENTIFIER_SIZE_IN_BYTES);
memcpy(rayGenShaderTableData.data() + D3D12_SHADER_IDENTIFIER_SIZE_IN_BYTES, &m_RayGenCB, sizeof(m_RayGenCB));
m_RayGenShaderTable.Create(L"RayGenShaderTable", rayGenShaderTableDesc, rayGenShaderTableData.data());
// Miss 着色器表
GpuBufferDesc missShaderTableDesc = rayGenShaderTableDesc;
missShaderTableDesc.m_Size = D3D12_SHADER_IDENTIFIER_SIZE_IN_BYTES;
missShaderTableDesc.m_Stride = missShaderTableDesc.m_Size;
m_MissShaderTable.Create(L"MissShaderTable", missShaderTableDesc, missShaderIdentifier);
// Hit 着色器表
GpuBufferDesc hitShaderTableDesc = missShaderTableDesc;
hitShaderTableDesc.m_Stride = hitShaderTableDesc.m_Size;
m_HitShaderTable.Create(L"HitShaderTable", hitShaderTableDesc, hitGroupIdentifier);
```



## 资源绑定

​	先前也提到过光追的资源绑定是通过 Shader Record 中的根参数绑定到管线中的，这些根参数对应的并不是图形管线中的根签名，在光追管线中根签名分为 **Local Root Signature** 和 **Global Root Signature**。至于为何需要引入局部根签名，答案很明显是为了优化，若是根签名过大会造成很大的性能负担。在图形管线中，每个根参数通过`D3D12_SHADER_VISIBILITY`枚举来控制各个 Shader 对该根参数的可见性，从而控制每个 Shader 的可见的根签名的大小，而 DXR 是建立在计算管线上的，因此就需要引入额外的机制来控制根签名的大小。

​	两种根签名的创建方式大致相同，不过局部根签名还需要**添加一个标识**`D3D12_ROOT_SIGNATURE_FLAG_LOCAL_ROOT_SIGNATURE`，这里就不加以演示了。全局根签名的绑定方式与图形管线相同，直接绑定到 PSO 中即可，而局部根签名则是需要先绑定到 PSO 中，再将其**关联到想要绑定到的 Shader** 上，具体操作后续详细描述。全局根签名绑定之后光追管线的所有着色器都可以访问其对应的资源，而局部根签名只有与其相关联的着色器才可以访问。除了创建和绑定，还有如下几点需要注意：

1. 一个 PSO 可以绑定**一个全局根签名**和**多个局部根签名**。
2. 一个局部根签名可以绑定**多个着色器**。
3. 一个着色器只可绑定一个局部根签名。

​	若是需要更新某个几何体绑定的资源，只需要在 Shader Table 中找到其对应的 Shader Record，然后**更新其内部的根参数**即可。细心的读者可能发现了，这个绑定方式的思想其实与 Command Signature 很像。Command Signature 是先通过 Indirect Argument 来描述需要使用的命令，随后通过命令列表的`ExecuteIndirect`设置存储了调用相应命令需要的参数（类似描述符表、顶点缓冲区、绘制命令等）的资源来控制渲染管线。而光追的资源绑定则是通过 局部根签名来描述需要使用的资源，随后通过在 Shader Table 中更新实际的资源（类似根常量的值、描述符表的地址等）来控制实际绑定的资源。

​	要绘制一个三角形，我们需要一个无序访问的纹理作为输出，同时输入一个作为 SRV 绑定的加速结构。而在创建光线的时候还需要视口信息，这里作为根常量绑定到管线中，因此根签名的定义如下：

```C++
// 注意是根常量
      m_LocalRootSig[0].InitAsConstants(0, sizeof(m_RayGenCB) / sizeof(uint32_t) + 1);
        m_LocalRootSig.Finalize(L"RayTracingLocalRootSignature", D3D12_ROOT_SIGNATURE_FLAG_LOCAL_ROOT_SIGNATURE);
        m_GlobalRootSig[RayTracingOutput].InitAsDescriptorRange(D3D12_DESCRIPTOR_RANGE_TYPE_UAV, 0, 1);  // RayTracingOutput
        m_GlobalRootSig[AccelerationStructure].InitAsBufferSRV(0);  // 加速结构
        m_GlobalRootSig.Finalize(L"RayTracingGlobalRootSignature");
```





## Acceleration Structure

​	DXR 的加速结构分为**顶层加速结构**和**底层加速结构**，顶层加速结构就是一个 BVH，其每一个叶子节点都指向一个底层加速结构，而每个加速结构又描述了一个几何类型的实例。两者的关系图如下：

<img src="https://img2024.cnblogs.com/blog/3406761/202509/3406761-20250907213210125-503996483.png" alt="image" style="zoom: 50%;" />

### 几何类型

​	在 DXR 中分为两种几何结构，分别为**三角形网格**和由 **AABB 盒**描述的自定义图元。对于三角形网格，DXR 中有一些列内置的配置，因此不需要手动配置；而对于自定义图元，若是想让其参与光追管线，需要自行为其配置 **Intersection Shader** 和**交点属性**。

​	这两种几何类型都由一个结构体`D3D12_RAYTRACING_GEOMETRY_DESC`来描述，其定义如下：

```C++
typedef struct D3D12_RAYTRACING_GEOMETRY_DESC {
  D3D12_RAYTRACING_GEOMETRY_TYPE  Type;
  D3D12_RAYTRACING_GEOMETRY_FLAGS Flags;
  union {
    D3D12_RAYTRACING_GEOMETRY_TRIANGLES_DESC Triangles;
    D3D12_RAYTRACING_GEOMETRY_AABBS_DESC     AABBs;
  };
} D3D12_RAYTRACING_GEOMETRY_DESC;
```

1. Type，为一个枚举，只有两个成员，分别代表上述的三角网格和 AABB 盒。
2. Flags，类型为`D3D12_RAYTRACING_GEOMETRY_FLAGS `，其成员及各自代表的行为如下：
   1. D3D12_RAYTRACING_GEOMETRY_FLAG_NONE ，使用该标识符则没有任何限制，Any Hit Shader 可被随意调用。
   2. D3D12_RAYTRACING_GEOMETRY_FLAG_OPAQUE ，使用该标识会让管线处理该几何体的时候将其判断为不透明，同时忽略 Any Hit Shader，使用后会启用重要的光线处理优化。
   3. D3D12_RAYTRACING_GEOMETRY_FLAG_NO_DUPLICATE_ANYHIT_INVOCATION，使用该标识会防止 Any Hit Shader 的重复调用。
3. Triangles，当类型为三角形网格时使用，包含了网格的顶点缓冲区和索引缓冲区的描述，同时还包含了一个变换矩阵，网格内的所有三角形都会乘上该矩阵。
4. AABBs，当类型是 AABB 时使用，包含了 AABB 盒的数量及储存 AABB 盒的 Buffer 的地址。

创建实例如下：

```C++
// 给底层加速结构的几何描述
D3D12_RAYTRACING_GEOMETRY_TRIANGLES_DESC trianglesDesc{};
trianglesDesc.Transform3x4 = 0;
trianglesDesc.IndexFormat = DXGI_FORMAT_R32_UINT;
trianglesDesc.VertexFormat = DXGI_FORMAT_R32G32B32_FLOAT;
trianglesDesc.IndexCount = m_IndexBuffer.GetCount();
trianglesDesc.VertexCount = m_VertexBuffer.GetCount();
trianglesDesc.IndexBuffer = m_IndexBuffer.GetGpuVirtualAddress();
trianglesDesc.VertexBuffer.StartAddress = m_VertexBuffer.GetGpuVirtualAddress();
trianglesDesc.VertexBuffer.StrideInBytes = m_VertexBuffer.GetStride();
D3D12_RAYTRACING_GEOMETRY_DESC geometryDesc{};
geometryDesc.Type = D3D12_RAYTRACING_GEOMETRY_TYPE_TRIANGLES;
geometryDesc.Flags = D3D12_RAYTRACING_GEOMETRY_FLAG_OPAQUE;
geometryDesc.Triangles = trianglesDesc;
```

### 底层加速结构

​	顶层加速结构盒底层加速结构都使用如下接口创建：

```C++
void BuildRaytracingAccelerationStructure(
  [in] const D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_DESC          *pDesc,
  [in] UINT                                                              NumPostbuildInfoDescs,
  [in] const D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_DESC *pPostbuildInfoDescs
);
```

其中第二三个参数为生成加速结构的信息，第一个参数才是需要输入的加速结构信息，其定义如下：

```C++
typedef struct D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_DESC {
  D3D12_GPU_VIRTUAL_ADDRESS                            DestAccelerationStructureData;
  D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_INPUTS Inputs;
  D3D12_GPU_VIRTUAL_ADDRESS                            SourceAccelerationStructureData;
  D3D12_GPU_VIRTUAL_ADDRESS                            ScratchAccelerationStructureData;
} D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_DESC;
```

1. DestAccelerationStructureData，储存加速结构的地址。
2. Inputs，加速结构的输入描述。
3. SourceAccelerationStructureData，旧的加速结构的地址，若是想要**更新**某个加速结构，则需要将其地址放到该参数。其地址若是与`DestAccelerationStructureData`相同，则会就地更新加速结构，该地址指向的加速结构的状态必须为`D3D12_RESOURCE_STATE_RAYTRACING_ACCELERATION_STRUCTURE`。
4. ScratchAccelerationStructureData，临时空间的地址，生成加速结构的时候需要一段临时空间，同时该资源的状态必须为`D3D12_RESOURCE_STATE_UNORDERED_ACCESS`，创建完成后可直接释放该资源。

对于加速结构的输入描述，其定义如下：

```C++
typedef struct D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_INPUTS {
  D3D12_RAYTRACING_ACCELERATION_STRUCTURE_TYPE        Type;
  D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAGS Flags;
  UINT                                                NumDescs;
  D3D12_ELEMENTS_LAYOUT                               DescsLayout;
  union {
    D3D12_GPU_VIRTUAL_ADDRESS            InstanceDescs;
    const D3D12_RAYTRACING_GEOMETRY_DESC *pGeometryDescs;
    const D3D12_RAYTRACING_GEOMETRY_DESC const * * ppGeometryDescs;
  };
} D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_INPUTS;
```

1. Type，加速结构的类型，也就是**底层**和**顶层**其中一个。
2. Flags，创建加速结构的标识，不同的标识会导致**不同的构建策略**，例如`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_PREFER_FAST_TRACE`会构建一个高质量的加速结构，提高光追时的效率，但是构建花的时间较长。
3. NumDescs，当 Type 的类型为底层加速结构该成员描述的是 pGeometryDescs 或 ppGeometryDescs 内的元素数，使用哪个取决于 DescsLayout；当 Type 的类型为顶层加速结构该成员描述的是 InstanceDescs 内的元素数，内部数据的类型也取决于 DescsLayout。
4. DescsLayout，指定描述符的类型，有两种描述，分别为`D3D12_ELEMENTS_LAYOUT_ARRAY `和`D3D12_ELEMENTS_LAYOUT_ARRAY_OF_POINTERS`，各自表示内部是描述结构体的数组和指向描述结构体的指针的数组。
5. InstanceDescs，Type 为顶层加速结构时有效，描述了一组几何实例，当 DescsLayout 为`D3D12_ELEMENTS_LAYOUT_ARRAY`时内部是类型为`D3D12_RAYTRACING_INSTANCE_DESC`的数组，否则内部是类型为 `D3D12_GPU_VIRTUAL_ADDRESS`的数组，各自指向了一个`D3D12_RAYTRACING_INSTANCE_DESC`的 Buffer。
6. pGeometryDescs，指向几何描述的数组的指针，当 Type 是底层加速结构且 DescsLayout 为`D3D12_ELEMENTS_LAYOUT_ARRAY`时有效。
7. ppGeometryDescs，指向几何描述的指针的数组的指针，当 Type 是底层加速结构且 DescsLayout 为`D3D12_ELEMENTS_LAYOUT_ARRAY_OF_POINTERS`时有效。

这个结构体复杂的我有点想吐槽了，每个成员有这么多重意思。由于要创建的底层加速结构，因此还需要填写结构体`D3D12_RAYTRACING_GEOMETRY_DESC`，也就是上面几何类型提到的结构体，果然配置 DX 就是疯狂填结构体。

​	填完这些结构体之前还需要提前为加速结构分配好显存，同时分配临时显存，并将地址传给`D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_DESC`。为了得知需要分配多大的显存还需要通过调用`ID3D12Device5::GetRaytracingAccelerationStructurePrebuildInfo`接口来获取创建加速结构的相关信息。其输入参数为`D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_INPUTS`结构体，输出为`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_PREBUILD_INFO`，其定义如下：

```C++
typedef struct D3D12_RAYTRACING_ACCELERATION_STRUCTURE_PREBUILD_INFO {
  UINT64 ResultDataMaxSizeInBytes;
  UINT64 ScratchDataSizeInBytes;
  UINT64 UpdateScratchDataSizeInBytes;
} D3D12_RAYTRACING_ACCELERATION_STRUCTURE_PREBUILD_INFO;
```

1. ResultDataMaxSizeInBytes，加速结构本身需要的大小。
2. ScratchDataSizeInBytes，初次创建加速结构需要的临时空间。
3. UpdateScratchDataSizeInBytes，更新加速结构需要的临时空间。

​	创建完加速结构和临时空间，填完相关结构体后就可以调用`BuildRaytracingAccelerationStructure`来创建加速结构了。

### 顶层加速结构

​	前面也提到过，顶层加速结构的每个**叶子节点**都代表了一个底层加速结构，因此需要创建完底层加速结构后才可创建顶层加速结构，其需要填写的结构体与底层加速结构相同，下面列出不同的地方。在填写`D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_INPUTS`结构体的时候，其需要使用的参数为`InstanceDescs`，这个参数为 GPU 虚拟地址，其指向的显存为存储了`D3D12_RAYTRACING_INSTANCE_DESC`类型的数组或指针数组。该结构体定义如下：

```C++
typedef struct D3D12_RAYTRACING_INSTANCE_DESC {
  FLOAT                     Transform[3][4];
  UINT                      InstanceID : 24;
  UINT                      InstanceMask : 8;
  UINT                      InstanceContributionToHitGroupIndex : 24;
  UINT                      Flags : 8;
  D3D12_GPU_VIRTUAL_ADDRESS AccelerationStructure;
} D3D12_RAYTRACING_INSTANCE_DESC;
```

1. Transform，与前面的三角形的几何描述相似，这个参数为对底层加速结构的**从对象空间到世界空间**变换，但实际的实现并不是将所有的顶点乘上这个变换，而是将该变换的**逆变换应用到光线**上，也就是将世界空间下的光线变换到对象空间中，然后将对象空间的模型与对象空间的光线进行相交测试，这样就将对所有顶点进行矩阵变换改变为只对光线进行变换，极大的减少了矩阵乘法。
2. InstanceID，可在 Shader 中使用内置函数`InstanceID()`来访问这个数。
3. InstanceMask，实例掩码，前面在讲解`TraceRay`函数的时候提到过，其与传入`TraceRay`的参数 InstanceInclusionMask **取与后不为零**才会进行相交检测，因此注意了不要不小心把其设置为0了，不然怎么取与都是0。
4. InstanceContributionToHitGroupIndex，在前面讲解 Shader Table 时也提到过这个参数，其表示了当前实例对应的 Shader Record **在 Shader Table 中的偏移**。
5. Flags，对实例的标识，不同标识会在光追时造成不同的行为，例如`D3D12_RAYTRACING_INSTANCE_FLAG_FORCE_OPAQUE `标识会强制将该实例解释为不透明，因此就不会为其调用 Any Hit Shader。
6. AccelerationStructure，这个就是指向底层加速结构的指针了，因此需要为底层加速结构分配完显存后再填写该参数，同时该地址必须与 256（D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BYTE_ALIGNMENT） 字节对齐，且其指向的资源的状态必须是`D3D12_RESOURCE_STATE_RAYTRACING_ACCELERATION_STRUCTURE`。

​	与底层加速结构类似，分配好显存和填完结构体后便可调用`BuildRaytracingAccelerationStructure`来创建加速结构了。不过有一点还需要注意，由于顶层加速结构的创建依赖于底层加速结构，因此需要等待底层加速结构创建完才能创建顶层加速结构，可通过再他们之间插入一个** UAV 屏障**来保证同步。

​	完整的加速结构创建过程如下，个人对资源的创建进行了一些封装，可忽略那部分：

```c++
// 给底层加速结构的几何描述
D3D12_RAYTRACING_GEOMETRY_TRIANGLES_DESC trianglesDesc{};
trianglesDesc.Transform3x4 = 0;
trianglesDesc.IndexFormat = DXGI_FORMAT_R32_UINT;
trianglesDesc.VertexFormat = DXGI_FORMAT_R32G32B32_FLOAT;
trianglesDesc.IndexCount = m_IndexBuffer.GetCount();
trianglesDesc.VertexCount = m_VertexBuffer.GetCount();
trianglesDesc.IndexBuffer = m_IndexBuffer.GetGpuVirtualAddress();
trianglesDesc.VertexBuffer.StartAddress = m_VertexBuffer.GetGpuVirtualAddress();
trianglesDesc.VertexBuffer.StrideInBytes = m_VertexBuffer.GetStride();
D3D12_RAYTRACING_GEOMETRY_DESC geometryDesc{};
geometryDesc.Type = D3D12_RAYTRACING_GEOMETRY_TYPE_TRIANGLES;
geometryDesc.Flags = D3D12_RAYTRACING_GEOMETRY_FLAG_OPAQUE;
geometryDesc.Triangles = trianglesDesc;

// 底层加速结构的输入
D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_INPUTS bottomLevelASInputs{};
bottomLevelASInputs.Type = D3D12_RAYTRACING_ACCELERATION_STRUCTURE_TYPE_BOTTOM_LEVEL;
bottomLevelASInputs.Flags = D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_PREFER_FAST_TRACE;
bottomLevelASInputs.NumDescs = 1;
bottomLevelASInputs.DescsLayout = D3D12_ELEMENTS_LAYOUT_ARRAY;
bottomLevelASInputs.pGeometryDescs = &geometryDesc;

// 获取加速结构的相关信息
D3D12_RAYTRACING_ACCELERATION_STRUCTURE_PREBUILD_INFO bottomLevelASInfo{};
g_RenderContext.GetDevice()->GetRaytracingAccelerationStructurePrebuildInfo(&bottomLevelASInputs, &bottomLevelASInfo);
ASSERT(bottomLevelASInfo.ResultDataMaxSizeInBytes > 0);

// 顶层加速结构的输入
D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_INPUTS topLevelASInputs = bottomLevelASInputs;
topLevelASInputs.Type = D3D12_RAYTRACING_ACCELERATION_STRUCTURE_TYPE_TOP_LEVEL;

D3D12_RAYTRACING_ACCELERATION_STRUCTURE_PREBUILD_INFO topLevelASInfo{};
g_RenderContext.GetDevice()->GetRaytracingAccelerationStructurePrebuildInfo(&topLevelASInputs, &topLevelASInfo);
ASSERT(topLevelASInfo.ResultDataMaxSizeInBytes > 0);

// 为加速结构分配显存
GpuBufferDesc bottomLevelASDesc{};
bottomLevelASDesc.m_Size = bottomLevelASInfo.ResultDataMaxSizeInBytes;
bottomLevelASDesc.m_Stride = bottomLevelASDesc.m_Size;
bottomLevelASDesc.m_HeapType = D3D12_HEAP_TYPE_DEFAULT;
bottomLevelASDesc.m_Flags = D3D12_RESOURCE_FLAG_ALLOW_UNORDERED_ACCESS;
m_BottomLevelAS.Create(L"BottomLevelAS", bottomLevelASDesc);
GpuBufferDesc topLevelASDesc = bottomLevelASDesc;
topLevelASDesc.m_Size = topLevelASInfo.ResultDataMaxSizeInBytes;
topLevelASDesc.m_Stride = topLevelASDesc.m_Size;
m_TopLevelAS.Create(L"TopLevelAS", topLevelASDesc);

// 顶层加速结构的输入，使用底层加速结构作为输入
D3D12_RAYTRACING_INSTANCE_DESC instanceDesc{};
instanceDesc.Transform[0][0] = instanceDesc.Transform[1][1] = instanceDesc.Transform[2][2] = 1.0f;
instanceDesc.InstanceMask = 1;
instanceDesc.AccelerationStructure = m_BottomLevelAS.GetGpuVirtualAddress();
GpuResourceLocation instanceBuffer = g_RenderContext.GetCpuBufferAllocator().Allocate(sizeof(instanceDesc), D3D12_RAYTRACING_INSTANCE_DESCS_BYTE_ALIGNMENT);
memcpy(instanceBuffer.m_MappedAddress, &instanceDesc, sizeof(instanceDesc));
topLevelASInputs.InstanceDescs = instanceBuffer.m_GpuAddress;

// 分配加速结构生成需要的暂存空间
uint64_t scratchBufferSize = (std::max)(bottomLevelASInfo.ScratchDataSizeInBytes, topLevelASInfo.ScratchDataSizeInBytes);
GpuResourceLocation scratchBuffer = g_RenderContext.GetGpuBufferAllocator().Allocate(scratchBufferSize);

D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_DESC buildBottomLevelASDesc{};
buildBottomLevelASDesc.Inputs = bottomLevelASInputs;
buildBottomLevelASDesc.ScratchAccelerationStructureData = scratchBuffer.m_GpuAddress;
buildBottomLevelASDesc.DestAccelerationStructureData = m_BottomLevelAS.GetGpuVirtualAddress();

D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_DESC buildTopLevelASDesc{};
buildTopLevelASDesc.Inputs = topLevelASInputs;
buildTopLevelASDesc.ScratchAccelerationStructureData = scratchBuffer.m_GpuAddress;
buildTopLevelASDesc.DestAccelerationStructureData = m_TopLevelAS.GetGpuVirtualAddress();

// 构建加速结构
GraphicsCommandList cmdList{L"BuildAccelerationStructure"};
cmdList.GetDXRCommandList()->BuildRaytracingAccelerationStructure(&buildBottomLevelASDesc, 0, nullptr);
// 等待底层加速结构构建完毕
cmdList.InsertUAVBarrier(m_BottomLevelAS, true);
cmdList.GetDXRCommandList()->BuildRaytracingAccelerationStructure(&buildTopLevelASDesc, 0, nullptr);
cmdList.ExecuteCommandList(true);	// 等待创建完成
```



## State Objects

​	状态对象代表着整个光追管线的配置状态，与其对应的是图形管线的 PSO ，可以说图形管线的 PSO 就是驱动层分装好的 **State Object**。一个 PSO 分为多个 **Subobject**，需要通过为 PSO 添加子对象来配置管线，DXR 通过`D3D12_STATE_SUBOBJECT`结构体来描述一个子对象，其定义如下：

```C++
typedef struct D3D12_STATE_SUBOBJECT {
  D3D12_STATE_SUBOBJECT_TYPE Type;		// 子对象的类型
  const void                 *pDesc;	// 指向子对象的指针
} D3D12_STATE_SUBOBJECT;
```

一个基本的光追 PSO（以下简称 RTPSO），需要七个子对象，分别为：

1. DXIL Library，包含了一组着色器的库，Shader Identifier 就是从中获取，其对应的子对象类型为`D3D12_STATE_SUBOBJECT_TYPE_DXIL_LIBRARY`，使用结构体`D3D12_DXIL_LIBRARY_DESC`进行描述，其定义如下：
   ```C++
   typedef struct D3D12_DXIL_LIBRARY_DESC {
     D3D12_SHADER_BYTECODE   DXILLibrary;	// 编译后的着色器库
     UINT                    NumExports;	// 导出着色器的个数
     const D3D12_EXPORT_DESC *pExports;	// 指向导出着色器的描述数组的指针
   } D3D12_DXIL_LIBRARY_DESC;
   ```

   当一个文件中定义了多个着色器，编译该文件后就得到了包含多个着色器的库，需要使用`D3D12_EXPORT_DESC`来对这多个着色器进行描述，其定义如下：
   ```C++
   typedef struct D3D12_EXPORT_DESC {
     LPCWSTR            Name;				// 导出的名字
     LPCWSTR            ExportToRename;	// 需要导出的着色器的名称
     D3D12_EXPORT_FLAGS Flags;				// 无用
   } D3D12_EXPORT_DESC;
   ```

   当 ExportToRename 为空的时候 Name 既是库中着色器的名称也是导出的名称，例如当着色器中有一个函数名叫做 MyRayGenShader 的着色器，可以将 Name 设置为 MyRayGenShader，ExportToRename设置为 NULL，则其导出名为 MyRayGenShader；也可将 Name 设置为 RGShader ，此时 ExportToRename 必须为 MyRayGenShader。后续在 Shader Table 获取该 Shader 的 Shader Identifier 就需要使用其**导出名**来获取，其他 Shader 想要引用该 Shader 的时候也是使用其导出名。

2. Hit Group，在光追管线中需要使用的 Hit Group，其对应的子对象类型为`D3D12_STATE_SUBOBJECT_TYPE_HIT_GROUP`，使用结构体`D3D12_HIT_GROUP_DESC`进行描述，其定义如下：
   ```C++
   typedef struct D3D12_HIT_GROUP_DESC {
     LPCWSTR              HitGroupExport;			 // 该 HitGroup 的导出名
     D3D12_HIT_GROUP_TYPE Type;					// 三角形网格或自定义图元
     LPCWSTR              AnyHitShaderImport;		 // 需要使用的 Shader 的导出名
     LPCWSTR              ClosestHitShaderImport;	  // 同上
     LPCWSTR              IntersectionShaderImport;  // 同上
   } D3D12_HIT_GROUP_DESC;
   ```

3. Shader Config，光线中 Shader 的配置，前面提到过光追管线中的光追负载和相交属性都可以属性都可以是自定义类型，而其支持的**最大的大小**就是通过该子对象配置的。其对应的子对象类型为`D3D12_STATE_SUBOBJECT_TYPE_RAYTRACING_SHADER_CONFIG`，使用结构体`D3D12_RAYTRACING_SHADER_CONFIG`进行描述，其定义如下：
   ```C++
   typedef struct D3D12_RAYTRACING_SHADER_CONFIG {
     UINT MaxPayloadSizeInBytes;	// 光追负载的最大大小
     UINT MaxAttributeSizeInBytes;	// 交点属性的最大大小
   } D3D12_RAYTRACING_SHADER_CONFIG;
   ```

   其中 MaxAttributeSizeInBytes 的大小限制为`D3D12_RAYTRACING_MAX_ATTRIBUTE_SIZE_IN_BYTES`。

4. Local Root Signature，局部根签名，其对应的子对象类型为`D3D12_STATE_SUBOBJECT_TYPE_LOCAL_ROOT_SIGNATURE`，使用结构体`D3D12_LOCAL_ROOT_SIGNATURE`进行描述，其定义如下：
   ```C++
   typedef struct D3D12_LOCAL_ROOT_SIGNATURE {
     ID3D12RootSignature *pLocalRootSignature;	// 指向根签名的指针
   } D3D12_LOCAL_ROOT_SIGNATURE;
   ```

5. Association，将局部根签名与 Shader 相关联，其对应的子对象类型为`D3D12_STATE_SUBOBJECT_TYPE_SUBOBJECT_TO_EXPORTS_ASSOCIATION`，使用结构体`D3D12_SUBOBJECT_TO_EXPORTS_ASSOCIATION`进行描述，其定义如下：
   ```C++
   typedef struct D3D12_SUBOBJECT_TO_EXPORTS_ASSOCIATION {
     const D3D12_STATE_SUBOBJECT *pSubobjectToAssociate;	// 需要关联的子对象
     UINT                        NumExports;	// 需要关联的数量
     LPCWSTR                     *pExports;	// 被关联对象的导出名，如第一点中 Shader 的导出名
   } D3D12_SUBOBJECT_TO_EXPORTS_ASSOCIATION;
   ```

6. Global Root Signature，全局根签名，其对应的子对象类型为`D3D12_STATE_SUBOBJECT_TYPE_GLOBAL_ROOT_SIGNATURE`，使用结构体`D3D12_GLOBAL_ROOT_SIGNATURE`进行描述，其定义如下：
   ```C++
   typedef struct D3D12_GLOBAL_ROOT_SIGNATURE {
     ID3D12RootSignature *pGlobalRootSignature;	// 指向的全局根签名
   } D3D12_GLOBAL_ROOT_SIGNATURE;
   ```

7. Pipeline Config，光追管线的配置，其对应的子对象类型为`D3D12_STATE_SUBOBJECT_TYPE_RAYTRACING_PIPELINE_CONFIG`，使用结构体`D3D12_RAYTRACING_PIPELINE_CONFIG`进行描述，其定义如下：
   ```C++
   typedef struct D3D12_RAYTRACING_PIPELINE_CONFIG {
     UINT MaxTraceRecursionDepth;
   } D3D12_RAYTRACING_PIPELINE_CONFIG;
   ```

   该参数配置了每个光线的**最大递归深度**，也就是每根光线在其生命周期内可以主动调用`TraceRay()` 或 `CallShader()`的次数，由于光追算法的嵌套性，必须要限制递归深度来保证性能，该值必须**小于32**，否则设备会进入移除状态。

配置完所有的子对象后，即可将所有子对象传递给`D3D12_STATE_SUBOBJECT`并调用`ID3D12Device5::CreateStateObject`。整个创建流程如下：
```C++
// 创建光线追踪管线
// 创建一个光线追踪管线状态对象需要又七个子对象
// 每个子对象都需要关联到着色器
// 1 - DXIL library
// 1 - Triangle hit group
// 1 - Shader config
// 2 - Local root signature and association
// 1 - Global root signature
// 1 - Pipeline config
std::vector<D3D12_STATE_SUBOBJECT> subobjects;
subobjects.reserve(7);

ShaderDesc raytracingShaderDesc{};
raytracingShaderDesc.m_Type = ShaderType::Lib;
raytracingShaderDesc.m_Mode = ShaderMode::SM_6_3;
raytracingShaderDesc.m_FileName = "Shaders//RayTracing.hlsl";
ShaderByteCode shaderLib{raytracingShaderDesc};

// DXIL library
std::vector<D3D12_EXPORT_DESC> exportDescs(3);
exportDescs[0].Name = s_RayGenShaderName;
exportDescs[1].Name = s_MissShaderName;
exportDescs[2].Name = s_ClosestHitShaderName;

D3D12_DXIL_LIBRARY_DESC dxilLibDesc{};
dxilLibDesc.DXILLibrary = shaderLib;
dxilLibDesc.NumExports = static_cast<UINT>(exportDescs.size());
dxilLibDesc.pExports = exportDescs.data();

D3D12_STATE_SUBOBJECT libSubobject{};
libSubobject.Type = D3D12_STATE_SUBOBJECT_TYPE_DXIL_LIBRARY;
libSubobject.pDesc = &dxilLibDesc;
subobjects.push_back(std::move(libSubobject));


// Triangle hit group
D3D12_HIT_GROUP_DESC hitGroupDesc{};
hitGroupDesc.Type = D3D12_HIT_GROUP_TYPE_TRIANGLES;
hitGroupDesc.HitGroupExport = s_HitGroupName;
hitGroupDesc.ClosestHitShaderImport = s_ClosestHitShaderName;

D3D12_STATE_SUBOBJECT hitGroupSubobject{};
hitGroupSubobject.Type = D3D12_STATE_SUBOBJECT_TYPE_HIT_GROUP;
hitGroupSubobject.pDesc = &hitGroupDesc;
subobjects.push_back(std::move(hitGroupSubobject));


// Shader config
D3D12_RAYTRACING_SHADER_CONFIG shaderConfig{};
shaderConfig.MaxAttributeSizeInBytes = 2 * sizeof(float); // 三角形的重心坐标
shaderConfig.MaxPayloadSizeInBytes = 4 * sizeof(float); // 光线的颜色

D3D12_STATE_SUBOBJECT shaderConfigSubobject{};
shaderConfigSubobject.Type = D3D12_STATE_SUBOBJECT_TYPE_RAYTRACING_SHADER_CONFIG;
shaderConfigSubobject.pDesc = &shaderConfig;
subobjects.push_back(std::move(shaderConfigSubobject));


// Local root signature and association
D3D12_LOCAL_ROOT_SIGNATURE localRootSig{};
localRootSig.pLocalRootSignature = m_LocalRootSig.GetRootSignature();

D3D12_STATE_SUBOBJECT localRootSigSubobject{};
localRootSigSubobject.Type = D3D12_STATE_SUBOBJECT_TYPE_LOCAL_ROOT_SIGNATURE;
localRootSigSubobject.pDesc = &localRootSig;
auto& localSubobject = subobjects.emplace_back(std::move(localRootSigSubobject));

// 将局部根签名与 shader 相关联
D3D12_SUBOBJECT_TO_EXPORTS_ASSOCIATION localRootSigAssociation{};
localRootSigAssociation.pSubobjectToAssociate = &localSubobject;
localRootSigAssociation.NumExports = 1;
localRootSigAssociation.pExports = &s_RayGenShaderName;
D3D12_STATE_SUBOBJECT localRootSigAssociationSubobject{};
localRootSigAssociationSubobject.Type = D3D12_STATE_SUBOBJECT_TYPE_SUBOBJECT_TO_EXPORTS_ASSOCIATION;
localRootSigAssociationSubobject.pDesc = &localRootSigAssociation;
subobjects.push_back(std::move(localRootSigAssociationSubobject));


// Global root signature
D3D12_GLOBAL_ROOT_SIGNATURE globalRootSig{};
globalRootSig.pGlobalRootSignature = m_GlobalRootSig.GetRootSignature();

D3D12_STATE_SUBOBJECT globalRootSigSubobject{};
globalRootSigSubobject.Type = D3D12_STATE_SUBOBJECT_TYPE_GLOBAL_ROOT_SIGNATURE;
globalRootSigSubobject.pDesc = &globalRootSig;
subobjects.push_back(std::move(globalRootSigSubobject));


// Pipeline config
D3D12_RAYTRACING_PIPELINE_CONFIG pipelineConfig{};
pipelineConfig.MaxTraceRecursionDepth = 1;  // 最大递归深度

D3D12_STATE_SUBOBJECT pipelineConfigSubobject{};
pipelineConfigSubobject.Type = D3D12_STATE_SUBOBJECT_TYPE_RAYTRACING_PIPELINE_CONFIG;
pipelineConfigSubobject.pDesc = &pipelineConfig;
subobjects.push_back(std::move(pipelineConfigSubobject));


// 创建管线状态对象
D3D12_STATE_OBJECT_DESC stateObjectDesc{};
stateObjectDesc.Type = D3D12_STATE_OBJECT_TYPE_RAYTRACING_PIPELINE;
stateObjectDesc.NumSubobjects = static_cast<UINT>(subobjects.size());
stateObjectDesc.pSubobjects = subobjects.data();
g_RenderContext.GetDevice()->CreateStateObject(&stateObjectDesc, IID_PPV_ARGS(m_RayTracingStateObject.GetAddressOf()));
```



## 启动与运行光追管线

​	相比光追管线的配置，其启动就简单多了，只需要使用命令列表绑定相应的资源并调用`DispatchRay`即可，调用`DispatchRay`之前还需要填写`D3D12_DISPATCH_RAYS_DESC`结构体作为参数，其定义如下：
```C++
typedef struct D3D12_DISPATCH_RAYS_DESC {
  D3D12_GPU_VIRTUAL_ADDRESS_RANGE            RayGenerationShaderRecord;	 // 需要使用的 Ray Generation Shader Record
  D3D12_GPU_VIRTUAL_ADDRESS_RANGE_AND_STRIDE MissShaderTable;			// Shader Table 的首地址及 Shader Record的间隔
  D3D12_GPU_VIRTUAL_ADDRESS_RANGE_AND_STRIDE HitGroupTable;				// 同上
  D3D12_GPU_VIRTUAL_ADDRESS_RANGE_AND_STRIDE CallableShaderTable;		// 同上
  UINT                                       Width;
  UINT                                       Height;
  UINT                                       Depth;
} D3D12_DISPATCH_RAYS_DESC;
```

要绘制一个三角形只需要一个输出纹理和加速结构即可，输出纹理作为 UAV 绑定到管线上，而加速结构作为 SRV 绑定到管线上，而创建光线还需要视口信息，先前已经使用局部根签名和 Shader Table 作为根常数绑定到管线上了因此不需要再次绑定，具体流程如下，
```C++
auto& swapChain = renderContext.GetSwapChain();

GraphicsCommandList cmdList{ L"Render Scene" };

auto& computeCmdList = cmdList.GetComputeCommandList();
computeCmdList.SetRootSignature(g_Renderer.m_GlobalRootSig);
computeCmdList.SetDescriptorHeap(g_Renderer.m_TextureHeap.GetHeap());
computeCmdList.SetDescriptorTable(Renderer::RayTracingOutput, g_Renderer.m_OutputUAV);
computeCmdList.SetShaderResource(Renderer::AccelerationStructure, g_Renderer.m_TopLevelAS);

D3D12_DISPATCH_RAYS_DESC dispatchDesc{};
dispatchDesc.HitGroupTable.StartAddress = g_Renderer.m_HitShaderTable->GetGPUVirtualAddress();
dispatchDesc.HitGroupTable.SizeInBytes = g_Renderer.m_HitShaderTable.GetSize();
dispatchDesc.HitGroupTable.StrideInBytes = dispatchDesc.HitGroupTable.SizeInBytes;
dispatchDesc.MissShaderTable.StartAddress = g_Renderer.m_MissShaderTable->GetGPUVirtualAddress();
dispatchDesc.MissShaderTable.SizeInBytes = g_Renderer.m_MissShaderTable.GetSize();
dispatchDesc.MissShaderTable.StrideInBytes = dispatchDesc.MissShaderTable.SizeInBytes;
dispatchDesc.RayGenerationShaderRecord.StartAddress = g_Renderer.m_RayGenShaderTable->GetGPUVirtualAddress();
dispatchDesc.RayGenerationShaderRecord.SizeInBytes = g_Renderer.m_RayGenShaderTable.GetSize();
dispatchDesc.Width = swapChain.GetWidth();
dispatchDesc.Height = swapChain.GetHeight();
dispatchDesc.Depth = 1;
computeCmdList.GetDXRCommandList()->SetPipelineState1(g_Renderer.m_RayTracingStateObject.Get());
computeCmdList.GetDXRCommandList()->DispatchRays(&dispatchDesc);
```

着色器代码如下，该着色器也是十分简单，由于不需要使用投影矩阵，因此在 Ray Generation Shader 中对着视口中的每一个像素构造一根向前的管线即可，相当于正交投影；在 Closest Hit Shader 中直接输出交点的重心坐标；在 Miss Shader 中输出黑色：
```hlsl
#if defined(__cplusplus)
using float3 = DSM::Math::Vector3;
using float4 = DSM::Math::Vector4;
using float3x3 = DSM::Math::Matrix3;
using float4x4 = DSM::Math::Matrix4;
#endif

struct Viewport
{
    float left;
    float top;
    float right;
    float bottom;
};

struct RayGenConstantBuffer
{
    Viewport viewport;
    Viewport stencil;
    float3 outSideColor;
};

struct RayPayload
{
    float4 color;
};

RaytracingAccelerationStructure gScene : register(t0);
RWTexture2D<float4> gOutput : register(u0);
ConstantBuffer<RayGenConstantBuffer> gRayGenCB : register(b0);

bool IsInsideViewport(float2 p, Viewport viewport)
{
    return viewport.left <= p.x && p.x <= viewport.right &&
        viewport.top <= p.y && p.y <= viewport.bottom;
}

[shader("raygeneration")]
void RaygenShader()
{
    float2 rayIndex = DispatchRaysIndex().xy;
    float3 lerpVal = float3(DispatchRaysIndex()) / DispatchRaysDimensions();
    float3 dir = float3(0, 0, 1);
    float3 origin = float3(
        lerp(gRayGenCB.viewport.left, gRayGenCB.viewport.right, lerpVal.x),
        lerp(gRayGenCB.viewport.top, gRayGenCB.viewport.bottom, lerpVal.y),
        0.0f);

    if(IsInsideViewport(origin.xy, gRayGenCB.stencil)){
        RayDesc ray;
        ray.Origin = origin;
        ray.Direction = dir;
        ray.TMin = 0.001;
        ray.TMax = 10000;

        // 光线负载
        RayPayload payload = { float4(0, 0, 0, 0) };
        TraceRay(gScene, RAY_FLAG_CULL_BACK_FACING_TRIANGLES, ~0, 0, 1, 0, ray, payload);

        gOutput[rayIndex] = payload.color;
    }
    else{
        gOutput[rayIndex] = float4(gRayGenCB.outSideColor, 1);
        //gOutput[rayIndex] = float4(lerpVal, 1);
    }
}

[shader("closesthit")]
void ClosestHitShader(inout RayPayload payload, in BuiltInTriangleIntersectionAttributes attrs)
{
    // 获取重心坐标
    float3 barycentrics = float3(1 - attrs.barycentrics.x - attrs.barycentrics.y, attrs.barycentrics.x, attrs.barycentrics.y);
    payload.color = float4(barycentrics, 1);
}

[shader("miss")]
void MissShader(inout RayPayload payload)
{
    payload.color = float4(0, 0, 0, 1);
}
```

最后得到的输出如下<img src="https://img2024.cnblogs.com/blog/3406761/202509/3406761-20250907233458048-1421424224.png" alt="image" style="zoom: 50%;" />

详细代码可见我的GitHub：https://github.com/wsdanshenmiao/LearnMiniEngine。
