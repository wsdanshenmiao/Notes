# DirectX RayTracing (3)  程序图元及复杂光照

​	离上一篇文章隔的有点久了，在国庆前其实就看完了微软官方的案例并复刻了出来，但是一直懒得写，国庆也全拿去玩了，拖到过完了国庆才动笔。

​	在前面两篇中基本把 DXR 的大部分流程都介绍完了，这次把使用 Intersection Shader 实现程序图元介绍完后基本足够实现大部分需求了。在实现程序图元之前，这里先实现复杂场景的渲染，将上次的单物体渲染扩展为多物体，同时实现贴图和阴影的渲染。



## 光追管线的布局定义

​	在创建光追管线之前，需要对光追管线使用的各种结构体及部分参数进行定义，这里我将所有的参数都放在了一个头文件中，并使用预处理指令来区分是在 C++ 中还是在 HLSL 中，一边两者都可包含此头文件：
```C++
// 用于区分在C++代码包含还是被HLSL代码包含

#if defined(__cplusplus)
using float2 = std::pair<float, float>;
using float3 = DSM::Math::Vector3;
using float4 = DSM::Math::Vector4;
using float3x3 = DSM::Math::Matrix3;
using float4x4 = DSM::Math::Matrix4;
using uint = uint32_t;
#endif


#define MAX_TRACE_RECURSION_DEPTH 3


struct MaterialConstantBuffer
{
    float4 baseColor;
    float4 emissiveColor;
    float normalTexScale;
    float metallicFactor;
    float roughnessFactor;
    float pad;
};

struct DirectionalLightData
{
    float4 color;
    float4 direction;
};

struct LightData
{
    uint dirLightCount;
};

namespace RayTracing {
    struct Ray
    {
        float3 origin;
        float3 direction;
    };

    struct RayPayload
    {
        float4 color;
        uint depth;
    };

    struct ShadowRayPayload
    {
        bool visible;
    };

    // 自定义图元的属性
    struct ProceduralPrimitiveAttributes
    {
        float3 normal;
        float2 uv;
        bool frontFace;
    };

    // 场景的常量缓冲区
    struct SceneConstantBuffer
    {
        // 生成光线使用的数据
        float4 cameraPosAndFocusDist;
        float4 viewportU;
        float4 viewportV;
    };

    struct PrimitiveInstanceConstantBuffer
    {
        uint primitiveType; // 图元类型
    };

    // 使用的光线种类
    enum RayType{
        Radiance = 0,
        Shadow,
        Count
    };

    namespace TraceRayParameters {
        // 实例掩码
        static const uint InstanceMark = ~0;
        namespace HitGroup {
            static const uint Offset[RayType::Count] = {
                0,   // 用于渲染的光线
                1    // 用于阴影的光线
            };
            static const uint GeometryStride = RayType::Count;
        }

        namespace MissShader {
            // Miss Shader 只需要使用索引
            static const uint Offset[RayType::Count] = {
                0,   // 用于渲染的光线
                1    // 用于阴影的光线
            };
        }
    }

    // 解析几何的类型
    namespace AnalyticPrimitive{
        enum PrimitiveType{
            Sphere = 0,
            Quad,
            Cube,
            Count
        };
    }


}
```

在命名空间 RayTracing 外的结构体都是使用在后续绑定到管线中的常量缓冲区，命名空间内定义了两个光追负载，分别是`RayPayload`和`ShadowRayPayload`，两者分别用于场景光照的渲染及阴影的渲染。光照的渲染和阴影的渲染由于逻辑不同，需要使用两种不同的 Ray 来进行渲染，因此使用了枚举`RayType`来区分两种不同的光线。而后面命名空间`TraceRayParameters`内的静态变量为调用`TraceRay`时使用的参数，这里顺带在复习一下`TraceRay`的参数：
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

其中第三个参数`InstanceInclusionMask`就使用`TraceRayParameters::InstanceMark`进行填写，在光线遍历加速结构的时候就会使用该掩码来判断是否要跳过该节点；第四个参数`RayContributionToHitGroupIndex`使用`TraceRayParameters::HitGroup::Offset`来填写，表示当前光线使用的 Hit Group 在所有光线类型的 Shader Record 中的偏移，也就是当前类型的光线在所有光线类型的索引，若是不清楚的可以在第一篇中的 Shader Table 那一节详细了解；第五个参数为每个几何体使用的光线类型，使用`TraceRayParameters::HitGroup::GeometryStride`进行填写；第六个参数为在 Miss Shader Table 中的索引，渲染光照与渲染阴影使用的 Miss Shader 各不相同，因此`TraceRayParameters::MissShader::Offset`进行区分。在调用的时候可直接通过一下方式调用，提高了可读性：
```hlsl
TraceRay(gScene, 
    RAY_FLAG_CULL_BACK_FACING_TRIANGLES, 
    RayTracing::TraceRayParameters::InstanceMark, 
    RayTracing::TraceRayParameters::HitGroup::Offset[RayTracing::RayType::Radiance], 
    RayTracing::TraceRayParameters::HitGroup::GeometryStride, 
    RayTracing::TraceRayParameters::MissShader::Offset[RayTracing::RayType::Radiance], 
    rayDesc, 
    payload);
```

此外还有后面会用到的结构体，就不详细介绍了。



## 加速结构的创建

### 加载模型

​	模型的加载这里使用了 assimp 库，纹理加载使用了 stbimage 和 DDSTextureLoader12，具体的实现就不介绍了，详细可见 GitHub 上的项目源码，链接在文末。这里就简单介绍一下加载后得到的结构体，模型结构体如下：
```C++
struct Model
{
    std::string name{};
    DirectX::BoundingBox boundingBox{};
    std::vector<std::shared_ptr<Mesh>> meshes{};
    std::vector<std::shared_ptr<Material>> materials{};
    std::vector<TextureRef> textures{};
    GpuBuffer materialData{};

    Transform transform{};
};
```

其中 Mesh 保存了所有几何体的顶点和索引数据，详细如下：
```C++
enum PSOFlags : std::uint16_t
{
    kHasPosition = ( 1 << 0 ),
    kHasNormal = ( 1 << 1 ),
    kHasTangent = ( 1 << 2 ),
    kHasUV = ( 1 << 3 ),
    kAlphaBlend = ( 1 << 4 ),
    kAlphaTest = ( 1 << 5 ),
    kBothSide = ( 1 << 6 ),
};

struct Mesh
{
    std::string m_Name;

    DirectX::BoundingBox m_BoundingBox;
    // 设置顶点缓冲区使用的数据
    D3D12_VERTEX_BUFFER_VIEW m_PositionStream;
    D3D12_VERTEX_BUFFER_VIEW m_NormalStream;
    D3D12_VERTEX_BUFFER_VIEW m_UVStream;
    D3D12_VERTEX_BUFFER_VIEW m_TangentStream;
    // 索引缓冲区使用的数据
    D3D12_INDEX_BUFFER_VIEW m_IndexBufferViews;
    // 为 PSOFlags 用于判断是否有各个顶点数据
    uint16_t m_PSOFlags;
    // 暂时无用
    uint16_t m_PSOIndex;

    // 每次绘制需要使用的数据
    struct SubMesh
    {
        uint32_t m_IndexCount;
        uint32_t m_IndexOffset;
        uint32_t m_VertexCount;
        uint32_t m_VertexOffset;
        uint16_t m_MaterialIndex;
        // 使用的纹理在描述符堆中的偏移
        uint16_t m_SRVTableOffset;
    };
    std::map<std::string, SubMesh> m_SubMeshes;

    GpuBuffer m_MeshData{};
};
```

所有的顶点数据都上载到显存中，储存在 m_MeshData 里，其中 m_PositionStream 后续会在创建顶层加速结构中使用，剩下的 normal、uv、index 会在渲染的时候作为 StructuredBuffer 绑定到局部根签名中。网格使用的纹理对应的 SRV 在加载模型的时候就创建在了描述符堆中，储存在 `Renderer::m_TextureHeap`中，在每一个 SubMesh 中保存了在 Heap 中的索引。在改例子中我使用 PBR 渲染场景中的物体，因此模型中保存材质的定义如下：
```C++
enum MaterialTex
{
    kBaseColor, kDiffuseRoughness, kMetalness, kOcclusion, kEmissive, kNormal, kNumTextures
};

struct Material
{
    Math::Vector4 baseColor = {1,1,1,1};
    Math::Vector4 emissiveColor = {0,0,0,0};
    float normalTexScale = 1;
    float metallicFactor = 1;
    float roughnessFactor = 1;
};
```

 由于该文章主要介绍的是 DXR，因此有关 PBR 渲染后面只会简单提一下。

### 创建底层加速结构

​	在创建加速结构之前还需要定义一个辅助类，包含了加速结构本身、暂存缓冲区、加速结构的大小以及只有顶层加速结构会使用的实例描述，其定义如下：

```C++
struct AccelerationStructureBuffers
{
    GpuBuffer scratch;
    GpuBuffer accelerationStructure;
    GpuBuffer instanceDesc;    // Used only for top-level AS
    uint64_t resultDataMaxSizeInBytes;
};
```

在先前的例子中之使用了一个底层加速结构，而顶层加速结构的创建又依赖于底层加速结构，因此可以共用一个暂存缓冲区。但是现在需要同时创建多个底层加速结构，因此每个底层加速结构都需要单独使用暂存缓存区。在这里我先创建一个光线追踪类来储存管线中需要的资源与实现资源创建，其定义如下：
```C++

class RayTracer
{
public:
    RayTracer();

    void SetCamera(const Camera* camera) { m_Camera = camera; }
    void TraceRays(ComputeCommandList& cmdList);

    void AddModel(std::shared_ptr<Model> model);
    void AddProceduralGeometry(const ProceduralGeometryDesc& desc);
    void AddLight(const Light& light);

private:
    void CreateAccelerationStructure();
    void CreateShaderTable();

public:
    static constexpr size_t sm_MaxDirLightCount = 4;

private:
    const Camera* m_Camera;

    std::vector<std::shared_ptr<Model>> m_Models;
    std::unique_ptr<ProceduralGeometryManager> m_ProceduralGeometryManager;

    // 加速结构
    std::vector<AccelerationStructureBuffers> m_BottomLevelASs{};
    AccelerationStructureBuffers m_TopLevelAS{};

    // 着色器表
    GpuBuffer m_RayGenShaderTable{};
    GpuBuffer m_MissShaderTable{};
    GpuBuffer m_HitShaderTable{};


    // 光照信息
    GpuBuffer m_LightDataBuffer;
    GpuBuffer m_DirLightDataBuffer;
    std::vector<DirectionalLightData> m_DirLights{};
};
```

每次调用`AddModel`的时候都会调用`CreateAccelerationStructure`重新生成加速结构，这里暂不考虑效率问题，因此每次都会重新生成所有的底层加速结构。对于创建底层加速结构，在第一篇文章已经做了详细的介绍，这里再复习一下。底层加速结构的创建可分为以下几个步骤：

1. 填写`D3D12_RAYTRACING_GEOMETRY_DESC`结构体，由于使用的模型都是三角网格，因此类型填写为三角形，顺带提一下光追管线中三角形的图元类型只有三角形列表，不支持三角形带（Triangle Strips），具体代码如下：
   ```C++
   // 给底层加速结构的几何描述
   D3D12_RAYTRACING_GEOMETRY_TRIANGLES_DESC trianglesDesc{};
   trianglesDesc.Transform3x4 = 0;
   trianglesDesc.IndexFormat = DXGI_FORMAT_R32_UINT;
   trianglesDesc.VertexFormat = DXGI_FORMAT_R32G32B32_FLOAT;
   trianglesDesc.IndexCount = submesh.m_IndexCount;
   trianglesDesc.VertexCount = submesh.m_VertexCount;
   trianglesDesc.IndexBuffer = mesh->m_IndexBufferViews.BufferLocation + submesh.m_IndexOffset * sizeof(uint32_t);
   trianglesDesc.VertexBuffer.StartAddress = mesh->m_PositionStream.BufferLocation + submesh.m_VertexOffset * mesh->m_PositionStream.StrideInBytes;
   trianglesDesc.VertexBuffer.StrideInBytes = mesh->m_PositionStream.StrideInBytes;
   D3D12_RAYTRACING_GEOMETRY_DESC geometryDesc{};
   geometryDesc.Type = D3D12_RAYTRACING_GEOMETRY_TYPE_TRIANGLES;
   geometryDesc.Flags = D3D12_RAYTRACING_GEOMETRY_FLAG_OPAQUE;
   geometryDesc.Triangles = trianglesDesc;
   ```

   这里填入模型的顶点数据及索引数据的时候使用了模型类中的 VBV 和 IBV，使用内部的 BufferLocation 及 StrideInBytes 来将对应数据的 GPU虚拟地址传给结构体。在描述三角形网格的结构体中有一个参数为`Transform3x4`，对于该参数有两点需要注意，一是该变量与**顶层加速结构**使用的`D3D12_RAYTRACING_INSTANCE_DESC`中的`Transform`并不相同，虽然两者都是对底层加速结构进行变换，但是该参数是将该矩阵变换**应用到所有的顶点**上，因此若是使用该参数会**增加构造底层加速结构的开销**；二是虽然该参数需要填写的变换矩阵对应的 GPU 地址，但是在构造完底层加速结构之后改变该地址内的变换矩阵并不会对已经生成的底层加速结构产生影响，因此就别想着通过该参数减少加速结构的构建了，若是想要对几何体进行变换，还是需要重新构建加速结构。

2. 填写完几何描述之后就需要填写`D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_INPUTS`结构体，具体操作与之前相同。

3. 随后就是获取构建加速结构所需要的信息，同时为加速结构及要使用的暂存缓冲区分配显存，具体操作与之前相同，只不过将 Buffer 储存在前面提到的`AccelerationStructureBuffers`结构体中。这里也有两点需要注意，一是两个 Buffer 需要允许随机访问，也就是有`D3D12_RESOURCE_FLAG_ALLOW_UNORDERED_ACCESS`标识；二是加速结构的资源状态需要为`D3D12_RESOURCE_STATE_RAYTRACING_ACCELERATION_STRUCTURE`。

4. 最后就是使用命令队列构造加速结构了，注意这里需要为底层加速结构保存一个 UAV Barrier，但是也不要马上提交了，不然每次都等待加速结构创建完效率很低，一般 Barrier 的数量达到16个再提交比较好，再多的话对性能也不是很好：
   ```C++
   D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_DESC buildBottomLevelASDesc{};
   buildBottomLevelASDesc.Inputs = bottomLevelASInputs;
   buildBottomLevelASDesc.ScratchAccelerationStructureData = bottomLevelBuffers.scratch.GetGpuVirtualAddress();
   buildBottomLevelASDesc.DestAccelerationStructureData = bottomLevelBuffers.accelerationStructure.GetGpuVirtualAddress();
   
   cmdList.GetDXRCommandList()->BuildRaytracingAccelerationStructure(&buildBottomLevelASDesc, 0, nullptr);
   // 等待底层加速结构构建完毕
   cmdList.InsertUAVBarrier(bottomLevelBuffers.accelerationStructure);
   ```

   `InsertUAVBarrier`的实现如下：

   ```C++
   void CommandList::InsertUAVBarrier(GpuResource& resource, bool flush)	// 默认 false
   {
       D3D12_RESOURCE_BARRIER resourceBarrier = {};
       resourceBarrier.Type = D3D12_RESOURCE_BARRIER_TYPE_UAV;
       resourceBarrier.UAV.pResource = resource.GetResource();
       resourceBarrier.Flags = D3D12_RESOURCE_BARRIER_FLAG_NONE;
       m_ResourceBarriers.push_back(std::move(resourceBarrier));
   
       if (m_ResourceBarriers.size() >= 16 || flush) {
           FlushResourceBarriers();
       }
   }
   void CommandList::FlushResourceBarriers()
   {
       if (!m_ResourceBarriers.empty()) {
           m_CmdList->ResourceBarrier(m_ResourceBarriers.size(), m_ResourceBarriers.data());
       }
       m_ResourceBarriers.clear();
   }
   ```

遍历所有模型中的 Mesh 及 Mesh 中的 SubMesh，并执行加速结构的创建，这样就得到了场景中所有物体的底层加速结构。

### 创建顶层加速结构

​	顶层加速结构的创建也与之前相同，唯一的区别是`D3D12_RAYTRACING_INSTANCE_DESC`有多个，在遍历模型的时候需要顺带在创建完底层加速结构之后填写该结构体，相比之前创建的 Instance，这次创建是需要填写`InstanceContributionToHitGroupIndex`参数，在第一篇文章中也提到过，该参数表示当前底层加速结构的 Shader Record 在 Shader Table 中的偏移量，在调用`TraceRay`后光追管线会通过该参数及上面提到的`TraceRay`中的参数来索引需要使用的 Shader Record。具体操作如下：
```C++

uint32_t instanceContributionToHitGroupIndex = 0;
std::vector<D3D12_RAYTRACING_INSTANCE_DESC> instanceDescs;
for(const auto& model : m_Models){
    for(const auto& mesh : model->meshes){
        for(const auto& submesh : mesh->m_SubMeshes){
            Build Acceleration Structure
            ...

            // 顶层加速结构的输入，使用底层加速结构作为输入
            D3D12_RAYTRACING_INSTANCE_DESC instanceDesc{};
            instanceDesc.InstanceMask = RayTracing::TraceRayParameters::InstanceMark;
            instanceDesc.InstanceContributionToHitGroupIndex = instanceContributionToHitGroupIndex;
            instanceDesc.AccelerationStructure = bottomLevelBuffers.accelerationStructure.GetGpuVirtualAddress();
            DirectX::XMStoreFloat3x4(&reinterpret_cast<DirectX::XMFLOAT3X4&>(instanceDesc.Transform), model->transform.GetLocalToWorld());
            instanceDescs.push_back(std::move(instanceDesc));
            instanceContributionToHitGroupIndex += bottomLevelASInputs.NumDescs * RayTracing::RayType::Count;
        }
    }
}
```

每次创建底层加速结构，偏移量就会加上光线的类型数量。



## 创建 Shader Table

​	先前创建的 Shader Table 都只包含一个 Shader Record，同时局部根签名也较为简单，只有一个根常量。而这次使用的局部根签名还包含了物体的材质、纹理，以及几何数据，在第二篇文章中由于只有一个物体，因此我将法线等几何数据放在全局根签名中，但是这次每个物体都有各自的几何数据，因此放在局部根签名中较为方便。在由于 Shader Table 需要使用局部根签名，因此这里先定义一下管线需要使用到的根签名。

### 根签名的布局及创建

​	在这次的例子中，需要定义一个全局根签名与两个局部根签名，两个局部根签名分别给三角图元和自定义图元使用。这里使用枚举来定义各个根参数所对应的根索引，提高可读性。全局根签名的布局如下：
```C++
// 根签名的布局
namespace GlobalRootSignature {
    namespace RayTracing{
        enum Slot {
            RayTracingOutput = 0,
            AccelerationStructure,
            SceneConstantBuffer,
            Count
        };
    }

    namespace Light{
        enum Slot {
            LightData = RayTracing::Slot::Count,
            DirectionalLightDatas,
            Count
        };
    }

    static constexpr uint32_t GlobalRootSignatureCount = Light::Slot::Count;

    namespace StaticSampler {
        enum Slot {
            AnisoWrap = 0,
            Count
        };
    }
}
```

命名空间`GlobalRootSignature`内包含了三个命名空间，分别表示光追需要使用的资源、光照需要使用的资源和静态采样器，光追的资源与前两次相同，光照信息中的`LightData`中包含了方向光的数量，而`DirectionalLightDatas`后续会绑定一个`StructuredBuffer`，包含了方向光的数据。

​	局部根签名的布局如下：
```C++
// 局部根签名
namespace LocalRootSignature {
    namespace Type {
        enum Enum {
            Triangle = 0,
            AABB,
            Count
        };
    }

    namespace Triangle {
        enum Slot {
            Material = 0,
            IndexBuffer,
            NormalBuffer,
            UVBuffer,
            Textures,
            Count
        };
        struct RootArguments {
            MaterialConstantBuffer material;
            D3D12_GPU_VIRTUAL_ADDRESS indexBuffer;
            D3D12_GPU_VIRTUAL_ADDRESS normalBuffer;
            D3D12_GPU_VIRTUAL_ADDRESS uvBuffer;
            D3D12_GPU_DESCRIPTOR_HANDLE textures;   // 6 个 PBR 纹理
        };
    };

    namespace AABB{
        enum Slot{
            Material = 0,
            Textures,
            PrimitiveInstance,
            Count
        };
        // 16 字节对齐
        struct RootArguments{	// 16字节对齐，共64字节
            MaterialConstantBuffer material;	// 16字节对齐，共48字节
            D3D12_GPU_DESCRIPTOR_HANDLE textures;   // 6 个 PBR 纹理
            RayTracing::PrimitiveInstanceConstantBuffer primitiveInstance;
        };
    }

    inline uint32_t MaxRootArgumentsSize()
    {
        return (std::max)(sizeof(Triangle::RootArguments), sizeof(AABB::RootArguments));
    }
}
```

命名空间`LocalRootSignature`内同样包含了三个命名空间，分别表示命名空间的数量、三角形图元及 AABB 盒需要使用的局部根签名及对应的结构体，这里提一下 DXR 的自定义图元统一使用 AABB 盒进行表示，在创建底层加速结构的时候会定义其对应的** AABB 盒**，同时在调用`TraceRay`之后光线会先与该图元的 AABB 盒进行相交检测，判断相交后才会在 Shader Table 中索引其对应的的 Shader Record 并调用内部的 Hit Group 的 Intersection Shader 来判断是否相交，详细操作后面会提到。

​	除了使用枚举来定义各个根参数的索引外，上面还定义了各个局部根签名所使用的结构体，这里以三角图元的为例说明不同的根参数需要使用什么来进行绑定。

1. 第一个参数`MaterialConstantBuffer`使用**根常量**来描述，因此直接使用描述材质的结构体即可。
2. 第二到四个参数为物体的几何信息，使用 SRV 也就是**描述符**绑定到管线中，需要使用 GPU 的虚拟地址，也就是`D3D12_GPU_VIRTUAL_ADDRESS`进行绑定。CBV 和 UAV 同理。
3. 第五个参数为物体使用的纹理，使用**描述符表**绑定到管线中，使用的是首个描述符在描述符堆中的 GPU 句柄进行绑定。

而 AABB 的三个成员分别使用**根常量**、**描述符表**、**根常量**来描述。这里需要注意各个成员参数的顺序需要**与声明根签名时对应的根索引相同**。在最后还有一个`MaxRootArgumentsSize`函数，返回两个根签名对应的结构体占用的最大大小，这是由于 Shader Table 中的每一 Shader Record 大小需要相同，且大小为最大的那个 Shader Record 的大小。

​	在定义这些结构体的时候有一点特别要注意，那就是结构体的对齐，例如上面的结构体`AABB::RootArguments`，若是我把里面的参数交换一下就会导致映射到 GPU 的数据发生错误，原因是我的`MaterialConstantBuffer`使用了 DirectX Math 的`DirectX::XMVECTOR`，导致其是16字节对齐，因此现在该结构体的内存布局如下：
![image](https://img2024.cnblogs.com/blog/3406761/202510/3406761-20251014144542478-1048993633.png)

而我一开始的顺序却不是这样的：
```C++
struct PrimitiveInstanceConstantBuffer{	// 4字节
    uint primitiveType; // 图元类型
};

struct RootArguments{	// 16字节对齐，共80字节
    RayTracing::PrimitiveInstanceConstantBuffer primitiveInstance;	// 4字节
    MaterialConstantBuffer material;	// 16字节对齐，共48字节
    D3D12_GPU_DESCRIPTOR_HANDLE textures;   // 6 个 PBR 纹理，8字节
}
```

可以看到这时候多了16字节，都用于对齐内存，现在的内存布局如下：
![image](https://img2024.cnblogs.com/blog/3406761/202510/3406761-20251014145015765-1787030641.png)

可以看到`primitiveInstance`变量后面的12字节全用来对齐了，但是在我是使用根常量来描述该变量，因此管线会将4字节开始的内存视为下一个根参数的起点，但实际上并不是，这就会导致绑定的资源错误。

​	定义完布局之后根据布局及 HLSL 中各个资源对应的寄存器槽创建根签名即可：
```C++
auto& triangleRootSig = m_LocalRootSigs[LocalRootSignature::Type::Triangle];
auto& aabbRootSig = m_LocalRootSigs[LocalRootSignature::Type::AABB];
triangleRootSig[LocalRootSignature::Triangle::Slot::Material].InitAsConstants(1, Math::AlignUp(sizeof(MaterialConstantBuffer), 4) / sizeof(uint32_t));
triangleRootSig[LocalRootSignature::Triangle::Slot::IndexBuffer].InitAsBufferSRV(1);
triangleRootSig[LocalRootSignature::Triangle::Slot::NormalBuffer].InitAsBufferSRV(2);
triangleRootSig[LocalRootSignature::Triangle::Slot::UVBuffer].InitAsBufferSRV(3);
triangleRootSig[LocalRootSignature::Triangle::Slot::Textures].InitAsDescriptorRange(D3D12_DESCRIPTOR_RANGE_TYPE_SRV, 4, kNumTextures);
triangleRootSig.Finalize(L"RayTracingLocalRootSignature_Triangle", D3D12_ROOT_SIGNATURE_FLAG_LOCAL_ROOT_SIGNATURE);

size_t instanceCBSize = Math::AlignUp(sizeof(RayTracing::PrimitiveInstanceConstantBuffer), 4);
aabbRootSig[LocalRootSignature::AABB::Slot::PrimitiveInstance].InitAsConstants(2, instanceCBSize / sizeof(uint32_t));
aabbRootSig[LocalRootSignature::AABB::Slot::Material].InitAsConstants(1, Math::AlignUp(sizeof(MaterialConstantBuffer), 4) / sizeof(uint32_t));
aabbRootSig[LocalRootSignature::AABB::Slot::Textures].InitAsDescriptorRange(D3D12_DESCRIPTOR_RANGE_TYPE_SRV, 4, kNumTextures);
aabbRootSig.Finalize(L"RayTracingLocalRootSignature_AABB", D3D12_ROOT_SIGNATURE_FLAG_LOCAL_ROOT_SIGNATURE);

m_GlobalRootSig[GlobalRootSignature::RayTracing::RayTracingOutput].InitAsDescriptorRange(D3D12_DESCRIPTOR_RANGE_TYPE_UAV, 0, 1);  // RayTracingOutput
m_GlobalRootSig[GlobalRootSignature::RayTracing::AccelerationStructure].InitAsBufferSRV(0);  // 加速结构
m_GlobalRootSig[GlobalRootSignature::RayTracing::SceneConstantBuffer].InitAsConstantBuffer(0);
m_GlobalRootSig[GlobalRootSignature::Light::LightData].InitAsConstantBuffer(0, D3D12_SHADER_VISIBILITY_ALL, 1);
m_GlobalRootSig[GlobalRootSignature::Light::DirectionalLightDatas].InitAsBufferSRV(0, D3D12_SHADER_VISIBILITY_ALL, 1);
        m_GlobalRootSig.InitStaticSampler(GlobalRootSignature::StaticSampler::AnisoWrap, Graphics::SamplerAnisoWrap);
        m_GlobalRootSig.Finalize(L"RayTracingGlobalRootSignature");
```

### 创建 Shader Table

​	创建的步骤与之前相同，部分细节稍有不同，如三角图元和自定义图元都有各自的 Shader Identifiers，同时 Miss Shader Record 也有两个。详细的操作如下：
```C++
// 创建着色器表
// 获取 Shader 的标识符
Microsoft::WRL::ComPtr<ID3D12StateObjectProperties> stateObjectProps{};
ASSERT_SUCCEEDED(g_Renderer.m_RayTracingStateObject.As(&stateObjectProps));
void* rayGenShaderIdentifier = stateObjectProps->GetShaderIdentifier(Renderer::s_RayGenShaderName);
std::array<void*, RayTracing::RayType::Count> missShaderIdentifiers{};
for (size_t i = 0; i < RayTracing::RayType::Count; i++) {
    missShaderIdentifiers[i] = stateObjectProps->GetShaderIdentifier(Renderer::s_MissShaderName[i]);
}
std::array<void*, RayTracing::RayType::Count> hitGroupIdentifiersTriangle{};
for (size_t i = 0; i < RayTracing::RayType::Count; i++) {
    hitGroupIdentifiersTriangle[i] = stateObjectProps->GetShaderIdentifier(Renderer::s_HitGroupName_Triangle[i]);
}

constexpr uint32_t shaderIdSize = D3D12_SHADER_IDENTIFIER_SIZE_IN_BYTES;

// RayGeneration 着色器表
GpuBufferDesc rayGenShaderTableDesc{};
rayGenShaderTableDesc.m_Size = shaderIdSize;
rayGenShaderTableDesc.m_Stride = rayGenShaderTableDesc.m_Size;
rayGenShaderTableDesc.m_HeapType = D3D12_HEAP_TYPE_DEFAULT;
m_RayGenShaderTable.Create(L"RayGenShaderTable", rayGenShaderTableDesc, rayGenShaderIdentifier);

// Miss 着色器表
constexpr uint32_t missShaderTableSize = shaderIdSize * RayTracing::RayType::Count;
std::array<uint8_t, missShaderTableSize> missShaderTableData{};
for(int i = 0; i < RayTracing::RayType::Count; ++i){
    memcpy(missShaderTableData.data() + i * shaderIdSize, missShaderIdentifiers[i], shaderIdSize);
}
GpuBufferDesc missShaderTableDesc = rayGenShaderTableDesc;
missShaderTableDesc.m_Size = missShaderTableSize;
missShaderTableDesc.m_Stride = shaderIdSize;
m_MissShaderTable.Create(L"MissShaderTable", missShaderTableDesc, missShaderTableData.data());

// Hit 着色器表
uint32_t hitGroupShaderRecordSize = shaderIdSize + LocalRootSignature::MaxRootArgumentsSize();
hitGroupShaderRecordSize = Math::AlignUp(hitGroupShaderRecordSize, D3D12_RAYTRACING_SHADER_RECORD_BYTE_ALIGNMENT);
std::vector<uint8_t> hitGroupShaderTableData{};
std::vector<uint8_t> hitGroupShaderRecordData{};
for (const auto& model : m_Models){
    for(const auto& mesh : model->meshes){
        for(const auto& submesh : mesh->m_SubMeshes) {
            for(int i = 0; i < RayTracing::RayType::Count; i++) {
                hitGroupShaderRecordData.clear();
                // 全部清零以免残余数据影响
                hitGroupShaderRecordData.resize(hitGroupShaderRecordSize, 0);
                memcpy(hitGroupShaderRecordData.data(), hitGroupIdentifiersTriangle[i], shaderIdSize);
                if(i == RayTracing::RayType::Radiance){ // 只有渲染光线需要填入根参数
                    LocalRootSignature::Triangle::RootArguments rootArgs{};
                    auto meshMat = model->materials[submesh.m_MaterialIndex];
                    rootArgs.material.baseColor = meshMat->baseColor;
                    rootArgs.material.emissiveColor = meshMat->emissiveColor;
                    rootArgs.material.metallicFactor = meshMat->metallicFactor;
                    rootArgs.material.roughnessFactor = meshMat->roughnessFactor;
                    rootArgs.material.normalTexScale = meshMat->normalTexScale;
                    rootArgs.indexBuffer = mesh->m_IndexBufferViews.BufferLocation + submesh.m_IndexOffset * sizeof(uint32_t);
                    rootArgs.normalBuffer = mesh->m_NormalStream.BufferLocation + 
                        submesh.m_VertexOffset * mesh->m_NormalStream.StrideInBytes;
                    rootArgs.uvBuffer = mesh->m_UVStream.BufferLocation + 
                        submesh.m_VertexOffset * mesh->m_UVStream.StrideInBytes;
                    rootArgs.textures = g_Renderer.m_TextureHeap[submesh.m_SRVTableOffset];
                    memcpy(hitGroupShaderRecordData.data() + shaderIdSize, &rootArgs, sizeof(rootArgs));
                }
                hitGroupShaderTableData.append_range(hitGroupShaderRecordData);
            }
        }
    }
}

GpuBufferDesc hitShaderTableDesc = rayGenShaderTableDesc;
hitShaderTableDesc.m_Size = Math::AlignUp(hitGroupShaderTableData.size(), D3D12_RAYTRACING_SHADER_TABLE_BYTE_ALIGNMENT);
hitShaderTableDesc.m_Stride = hitGroupShaderRecordSize;
hitGroupShaderTableData.resize(hitShaderTableDesc.m_Size);
m_HitShaderTable.Create(L"HitShaderTable", hitShaderTableDesc, hitGroupShaderTableData.data());
```

这四个循环做的事其实是遍历每一个几何体，为其分配两个 Shader Record，分别为渲染光照的 Hit Group 和渲染阴影的 Hit Group，阴影的 Hit Group 其实啥也没有，主要起占位作用。最后将着色器标识与局部根签名的参数拷贝到 Shader Record 中。



## 创建光追管线状态对象

​	相比前两次创建光追的 PSO ，这次需要额外添加与更改几个参数：

1. 需要创建两个 Hit Group，分别对应两种光线：
   ```C++
   std::array<D3D12_HIT_GROUP_DESC, RayTracing::RayType::Count> hitGroupDescsTriangle{};
   for(int i = 0; i < RayTracing::RayType::Count; ++i){
       auto& hitGroupDesc = hitGroupDescsTriangle[i];
       hitGroupDesc.Type = D3D12_HIT_GROUP_TYPE_TRIANGLES;
       hitGroupDesc.HitGroupExport = s_HitGroupName_Triangle[i];
       if(i == RayTracing::RayType::Radiance){        
       	hitGroupDesc.ClosestHitShaderImport = s_ClosestHitShaderName[GeometryType::Triangle];
       }
   
       D3D12_STATE_SUBOBJECT hitGroupSubobject{};
       hitGroupSubobject.Type = D3D12_STATE_SUBOBJECT_TYPE_HIT_GROUP;
       hitGroupSubobject.pDesc = &hitGroupDesc;
       subobjects.push_back(std::move(hitGroupSubobject));
   }
   ```

2. Shader Config 中的光追负载的大小需要为两种光线负载的最大值，同时由于后续加入的自定义图元的图元属性比内置的图元属性大，因此也需要更改：
   ```C++
   D3D12_RAYTRACING_SHADER_CONFIG shaderConfig{};
   shaderConfig.MaxPayloadSizeInBytes = (std::max)(sizeof(RayTracing::RayPayload), sizeof(RayTracing::ShadowRayPayload)); // 光线的颜色
   shaderConfig.MaxAttributeSizeInBytes = sizeof(RayTracing::ProceduralPrimitiveAttributes); // 自定义图元的法线
   ```

3. 这次使用了两个局部根签名，分别是三角图元和自定义图元的，在创建两个根签名的子对象的同时还要创建他们与 Hit Group 的关联。

4. 由于添加了阴影光线，因此最大递归深度需要大于1：
   ```C++
   // Pipeline config
   D3D12_RAYTRACING_PIPELINE_CONFIG pipelineConfig{};
   pipelineConfig.MaxTraceRecursionDepth = MAX_TRACE_RECURSION_DEPTH;  // 最大递归深度，3
   ```

创建完 PSO 后 CPU 端的准备工作也就完成了，随后使用命令列表绑定全局资源与调用`DispatchRays`即可，流程与之前相同。



## 编写 Shader

### 定义使用的资源

​	在编写 Shader 之前还需要定义渲染过程中需要使用的资源，内部使用的结构体在描写管线布局的时候已经提到了，可以翻上去看看。以下资源绑定在全局根签名中，全局都可访问：
```hlsl
// Global
// 输出图像
RWTexture2D<float4> gOutput : register(u0);
// 加速结构
RaytracingAccelerationStructure gScene : register(t0);
ConstantBuffer<RayTracing::SceneConstantBuffer> gSceneCB : register(b0);
```

以下是描述几何体材质的资源，绑定在局部根签名中：
```hlsl
// Common Local
ConstantBuffer<MaterialConstantBuffer> lMaterialCB : register(b1);

Texture2D<float4> lBaseColorTex : register(t4);
Texture2D<float4> lDiffuseRoughnessTex : register(t5);
Texture2D<float4> lMetalnessTex : register(t6);
Texture2D<float> lOcclusionTex : register(t7);
Texture2D<float3> lEmissiveTex : register(t8);
Texture2D<float3> lNormalTex : register(t9);
```

三角形图元的才有的几何数据，同样绑定在局部根签名中：
```hlsl
// Triangle Geometry Local
StructuredBuffer<uint3> lIndexBuffer : register(t1);
StructuredBuffer<float3> lNormalBuffer : register(t2);
StructuredBuffer<float2> lUVBuffer : register(t3);
```

还有程序图元才有的图元类型，绑定在程序图元的局部根签名中：
```hlsl
// Procedural Geometry Local
ConstantBuffer<RayTracing::PrimitiveInstanceConstantBuffer> lPrimitiveInstanceCB : register(b2);
```

最后是计算光照使用的光照信息，绑定在全局根签名中：
```hlsl
ConstantBuffer<LightData> gLightData : register(b0, space1);
StructuredBuffer<DirectionalLightData> gDirLightData : register(t0, space1);
```

### Ray Generation Shader

​	在该 Shader 要做的工作一样是生成光线并调用`TraceRay`，最后将结果写入 UAV。详细如下：
```hlsl
[shader("raygeneration")]
void RaygenShader()
{
    RayTracing::Ray ray = GenerateCameraRay(
        DispatchRaysIndex().xy,
        gSceneCB.cameraPosAndFocusDist.xyz,
        gSceneCB.viewportU.xyz,
        gSceneCB.viewportV.xyz,
        gSceneCB.cameraPosAndFocusDist.w);

    float4 color = TraceRadianceRay(ray, 0);

    // 手动进行伽马映射
    color.rgb = LinearToSRGB(color.rgb);
    gOutput[DispatchRaysIndex().xy] = color;
}
```

这里生成光线的部分与第二篇文章相同，而`TraceRadianceRay`对`TraceRay`进行了一点封装添加了最大递归深度的限制并返回光追负载中的颜色，每次调用`TraceTray`，光追负载内的`depth`变量就会加一。这里需要注意的是并不是只要在创建 PSO 是指定最大递归深度就可以了，需要**手动在 HLSL 中进行限制**，若是`TraceRay`的递归深度大于 PSO 中定义的深度，会直接触发**设备移除**。
```hlsl
float4 TraceRadianceRay(RayTracing::Ray ray, uint depth)
{
    [branch]
    if(depth >= MAX_TRACE_RECURSION_DEPTH) {	// 限制递归深度 
        return 0;
    }

    RayDesc rayDesc;
    rayDesc.Origin = ray.origin;
    rayDesc.Direction = ray.direction;
    rayDesc.TMin = 0.001f;
    rayDesc.TMax = 10000.0f;

    RayTracing::RayPayload payload;
    payload.color = float4(0, 0, 0, 1);
    payload.depth = depth + 1;
    TraceRay(gScene, 
        RAY_FLAG_CULL_BACK_FACING_TRIANGLES, 
        RayTracing::TraceRayParameters::InstanceMark, 
        RayTracing::TraceRayParameters::HitGroup::Offset[RayTracing::RayType::Radiance], 
        RayTracing::TraceRayParameters::HitGroup::GeometryStride, 
        RayTracing::TraceRayParameters::MissShader::Offset[RayTracing::RayType::Radiance], 
        rayDesc, 
        payload);
    
    return payload.color;
}
```

在获得光追的结果后，这里还将颜色进行了伽马映射：
```hlsl
float3 LinearToSRGB(float3 linearColor)
{
    return pow(linearColor, 1.0f / 2.2f);
}
```

### Closest Hit Shader

#### 获取几何信息

​	调用完`TraceRay`后，光线就会遍历加速结构并查找交点，若是有交点则会调用 Closest Hit Shader ，在该函数中将会完成物体的着色。在计算光照之前，还需要获取射线击中的几何体的信息，前面也提到过，物体的几何信息绑定在了局部根签名中，想要访问射线击中的三角形的的索引，需要借助内置函数`PrimitiveIndex`该函数会返回击中的**三角形在底层加速结构中的索引**，可以在 Hit Group 中的所有 Shader 调用。因此可通过如下步骤获取顶点信息：
```hlsl
// Triangle Geometry Local
StructuredBuffer<uint3> lIndexBuffer : register(t1);
StructuredBuffer<float3> lNormalBuffer : register(t2);
StructuredBuffer<float2> lUVBuffer : register(t3);

uint3 indices = lIndexBuffer[PrimitiveIndex()];
float3 normals[3] = {
    lNormalBuffer[indices[0]],
    lNormalBuffer[indices[1]],
    lNormalBuffer[indices[2]]};
float3 uvs[3] = {
    lUVBuffer[indices[0]].xyy,
    lUVBuffer[indices[1]].xyy,
    lUVBuffer[indices[2]].xyy};
float3 normal = normalize(GetHitAttributes(normals, attrs.barycentrics));
float2 uv = GetHitAttributes(uvs, attrs.barycentrics).xy;
```

这里我实现了`GetHitAttributes`函数对三个顶点的数据进行线性插值，这里就不放出来了。

拿到纹理坐标 uv 后就可以采样几何体使用的纹理了，这里虽然定义了 6 个纹理，但是我并没有实现法线纹理，因此只采样了颜色、粗糙度、金属性、遮蔽值和自发光：
```hlsl
float4 baseCol = lBaseColorTex.SampleLevel(gAnisoWrapSampler, uv, 0);
baseCol *= lMaterialCB.baseColor;
float roughness = lDiffuseRoughnessTex.SampleLevel(gAnisoWrapSampler, uv, 0).g;
float metallic = lMetalnessTex.SampleLevel(gAnisoWrapSampler, uv, 0).b;
float occlusion = lOcclusionTex.SampleLevel(gAnisoWrapSampler, uv, 0).r;
float3 emissive = lEmissiveTex.SampleLevel(gAnisoWrapSampler, uv, 0).rgb;
```

随后就是计算着色：
```hlsl
// 感知上的粗糙度
float perceptualRoughness = roughness * lMaterialCB.roughnessFactor;

Surface surface;
surface.position = GetWorldPosition();
surface.recursionDepth = payload.depth;
surface.normal = normal;
surface.roughness = perceptualRoughness * perceptualRoughness;
surface.roughness = max(0.05, surface.roughness);
surface.color = baseCol.rgb;
surface.alpha = baseCol.a;
surface.viewDir = -WorldRayDirection();
surface.metallic = metallic * lMaterialCB.metallicFactor;

// 计算光照
float3 color = ShadeLighting(surface);
color += surface.color * 0.01;
color *= occlusion;
color += emissive * lMaterialCB.emissiveColor.rgb;
```

`Surface`包含了着色点的几何信息与材质信息，在构造的时候使用了几个内置函数，首先是`WorldRayDirection`，该内置函数会返回世界空间下的射线方向，其次是`GetWorldPosition`这个是我定义的函数，实现如下：

```hlsl
float3 GetWorldPosition()
{
    return WorldRayOrigin() + RayTCurrent() * WorldRayDirection();
}
```

该函数使用了三个内置函数，一个上面已经介绍过了，`WorldRayOrigin`故名思意就是返回世界空间下 Trace 的光线的起点，`RayTCurrent`会返回当前光线的结束点的距离，对于不同的 Shader 他返回的值意义也不尽相同：

1. 对于 Intersection Shader 该函数返回的是目前为止找到的最近交点的距离，由于该 Shader 是在遍历加速结构的过程中被调用，因此后续该值可能会被改变。
2. 对于 Closest Hit Shader 该函数返回的是真正的最近相交距离，由于 Closest Hit Shader 是在遍历完成后被调用，因此后续也不会被改变。
3. 对于 Any Hit Shader 该函数返回的是当前交点的距离。
4. 对于 Miss Shader 返回的是调用`TraceRay`是传入的 TMax。

将上面的起点和方向的改为 Object，函数返回的就是对象空间下的参数了。

#### 实现阴影

​	填写完 Surface 后将其传给`ShadeLighting`进行渲染，该函数定义如下：
```hlsl
float3 ShadeLighting(Surface surface)
{
    float3 color = 0;
    for(uint i = 0; i < GetDirectionalLightCount(); i++) {
        Light dirLight = GetDirectionalLight(i, surface);
        color += ShadeLighting(surface, dirLight);
    }
    return color;
}
```

在该函数中我遍历了所有方向光并调用了一个`ShadeLighting`重载函数进行渲染，在函数`GetDirectionalLight`内同时还进行了阴影的计算，该函数定义如下：
```hlsl

struct Light
{
    float3 color;
    float3 direction;
    float attenuation;
};

ConstantBuffer<LightData> gLightData : register(b0, space1);
StructuredBuffer<DirectionalLightData> gDirLightData : register(t0, space1);

Light GetDirectionalLight(uint index, Surface surface)
{
    DirectionalLightData lightData = gDirLightData[index];
    // 计算阴影
    RayDesc ray;
    ray.Origin = surface.position;
    ray.Direction = lightData.direction.xyz;
    ray.TMin = 0.001f;
    ray.TMax = 10000.0f;
    bool visible = TraceShadowRay(ray, surface.recursionDepth);

    Light light;
    light.color = lightData.color.rgb;
    light.direction = normalize(lightData.direction.xyz);
    light.attenuation = visible ? 1.0f : 0.0f;
    return light;
}
```

该函数将结构体缓冲区中的数据填写到自定义的光源类中，同时还调用了一个函数`TraceShadowRay`，该函数实现了阴影的渲染。基本思想是若是该点沿着光照方向不被其他物体遮挡，则会调用对应的 Miss Shader，反着则不会调用。因此想要实现阴影，需要在着色点创建一根沿着光照方向的光线，并调用`TraceRay`即可，其对应的光追负载只需要包含一个参数表示是否击中物体，在 Miss Shader 中将该变量改为 true 即可，对应的 Hit Group 为空即可。实现如下：
```hlsl
// 追踪阴影光线
bool TraceShadowRay(RayDesc ray, uint depth)
{
    [branch]
    if(depth >= MAX_TRACE_RECURSION_DEPTH) {
        return false;
    }

    RayTracing::ShadowRayPayload payload;
    payload.visible = false;
    TraceRay(gScene, 
        RAY_FLAG_CULL_BACK_FACING_TRIANGLES
        | RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH
        | RAY_FLAG_FORCE_OPAQUE             // ~skip any hit shaders
        | RAY_FLAG_SKIP_CLOSEST_HIT_SHADER, // ~skip closest hit shaders,
        RayTracing::TraceRayParameters::InstanceMark, 
        RayTracing::TraceRayParameters::HitGroup::Offset[RayTracing::RayType::Shadow], 
        RayTracing::TraceRayParameters::HitGroup::GeometryStride, 
        RayTracing::TraceRayParameters::MissShader::Offset[RayTracing::RayType::Shadow], 
        ray, 
        payload);

    return payload.visible;
}
```

对应的 Miss Shader 如下：
```hlsl
// 若阴影光线不与物体相交则表示可见
[shader("miss")]
void MissShader_Shadow(inout RayTracing::ShadowRayPayload payload)
{
    payload.visible = true;
}
```

#### 直接光照计算

##### 微表面模型及 BRDF

​	得到光源信息后，我将其与表面属性传给`ShadeLighting`进行光照计算，其内部使用了 PBR 模型进行计算，PBR 的材质模型使用 BSDF（双向散射分布函数）进行描述，该函数描述了在某个入射方向下，经过物体表面散射后出射光线在不同方向的散射的分布规律。该函数描述了其有两个函数组成，分别为 BRDF（双向反射分布函数）和 BTDF（双向透射分布函数），及完整的 BSDF 可通过以下公式描述。由于常见材质的透射分量较小，因此在实现通用材质模型时通常把 BTDF 忽略，保留 BRDF。

​	BRDF 包含了两个分量，即漫反射分量与镜面反射分量，因此完整的表面响应可以描述为以下方程`f(v,l) = fd(v,l) + fr(v,l)`，而整个渲染过程可以描述为该函数在半球上对入射光线 l 的积分，即法线方向的半球上所有入射光线对出射方向的贡献。

##### 镜面分量的 BRDF

​	在多数情况下， PBR 中的使用的 BRDF 都是基于微表面理论的，即在微观层面下物体的表面并不是完全光滑的，而是由大量按照某种规则排列的平面碎片组成，如下图所示：
<img src="https://img2024.cnblogs.com/blog/3406761/202510/3406761-20251016175358942-1891834643.png" alt="image" style="zoom: 80%;" />

在微观层面，物体的法线通常与宏观层面有所不同，微表面的每一个平面碎片都有其自己的法线，因此在描述微表面的时候就需要引入一个函数来描述微观层面的法线，而该函数就是**法线分布函数**（Normal distribution function），通常使用 **D** 来表示。该函数受到微表面粗糙度的影响，微表面越粗糙，法线分布越分散，沿着指定出射方向的出射的概率越小，反之则越大。但是即使是入射光线被微表面的法线正确的反射到了指定的出射方向上，该出射光线也不一定会对最终的结果造成贡献，因为微表面还存在自遮挡和自阴影关系，如下图所示：
![image](https://img2024.cnblogs.com/blog/3406761/202510/3406761-20251016181533674-81195915.png)

这时反射光线就不会对出射方向造成贡献，因此还需要使用另外一个函数来拟合这种现象，**几何遮蔽函数**（Geometric shadowing function）就应运而生了，通常使用 **G** 来表示该函数。除了这两个基于微表面理论的函数，我们还需要考虑其他光线与物体表面交互的物理效应，当光与两种物质之间的平面分界面发生相互作用时，会发生菲涅尔效应，也就是我们生活中遇到的当近乎平行观察一个平面时，发生的反射效应会比垂直观察时要强，该效应可以通过**菲涅尔方程**进行描述，通常使用 **F** 表示。

​	结合上面三个方程，我们就可以得到镜面反射的 BRDF，数学形式如下：
<img src="https://img2024.cnblogs.com/blog/3406761/202510/3406761-20251016183722331-1432694986.png" alt="image" style="zoom: 67%;" />

方程的分子就是上面提到的三个函数，而分子是归一化常数。使用该函数对法线半球进行积分，就能得到完成的镜面反射响应。PBR 经过多年的发展，有大量的模型来描述以上三个函数，这里就介绍较为常用的模型。

​	首先是法线分布函数，我使用的模型是 GGX 分布，该分布的衰减部分尾长较长，同时计算量也很适合实时渲染，因此在游戏引擎中十分常用。该函数公式如下：
<img src="https://img2024.cnblogs.com/blog/3406761/202510/3406761-20251016184623123-2063777426.png" alt="image" style="zoom: 80%;" />

其中 α 为微表面的粗糙程度，n 为法线方向，h 为半程向量，也就是视线方向与光照方向的中间向量。具体实现如下：
```hlsl
// 使用 GGX 模型的法线分布函数
// Dggx(h, r) =  r^2 / (pi * pow(pow(n * h, 2) * (r^2 - 1) + 1), 2)
float D_GGX(float NoH, float roughness)
{
    float a = roughness * roughness;
    float lower = lerp(1, a, NoH * NoH);    // 1 - (n * h)^2 + (n * h)^2 * r^2
    return a / max(1e-6, s_PI * lower * lower);  // 避免除零
}
```

这里的 NoH 为法线点乘半程向量，第二行使用了内置函数`lerp`来计算分母，同时还需要避免除以零的情况。

​	几何遮蔽函数的自变量为粗糙程度 α、入射方向 l 与出射方向 v，其又可以拆分为，两个相同形式的函数的乘积，表达式如下：
<img src="https://img2024.cnblogs.com/blog/3406761/202510/3406761-20251016194045115-953438241.png" alt="image" style="zoom:80%;" />

该函数同样也有多种模型，这里使用的为 GGX 模型，其公式为：
<img src="https://img2024.cnblogs.com/blog/3406761/202510/3406761-20251016194237589-196121083.png" alt="image" style="zoom:80%;" />

此时注意力惊人的朋友应该注意到了，完整的 G 的分母其实与镜面反射的 BRDF 的分子约掉了，因此这里为了减少计算量就将两者合并计算了，因此具体实现如下：
```hlsl
// Smith 将几何阴影函数分解为两个函数的乘积
// G(v, l, r) = G1(l, r) * G1(v, r)
// 使用 GGX 模型的 G1 函数
// G1_GGX(v, r) = 2 * (n * v) / (n * v + sqrt(r^2 + (1 - r^2) * (n * v)^2))
// 结合上面镜面反射的分母，可以化简为函数 V(v, r)
// V_GGX(v, r) = 1 / ((n * v) + sqrt(r^2 + (1 - r^2) * (n * v)^2))
float V_SmithGGXCorrelated(float NoV, float NoL, float roughness)
{
    float a = roughness * roughness;
    float gv = NoL + sqrt(a + (1 - a) * NoV * NoV);
    float gl = NoV + sqrt(a + (1 - a) * NoL * NoL);
    return 1 / max(1e-6, gv * gl); // 避免除零
}
```

这里的 NoL 为法线与光线方向的点乘。

​	最后便是菲涅尔方程，原始的菲涅尔方程特别复杂，因此通常使用施利克近似来实现其计算，公式如下：
![image](https://img2024.cnblogs.com/blog/3406761/202510/3406761-20251016195110681-109393499.png)

其中 f0 为垂直入射时的反射光量，f90 为平行入射时的反射光量，通常为1，也就是完全反射。具体实现如下：
```hlsl
// 使用 Schlick 近似的菲涅尔项
// F_Schlick(l, h, f0) = f0 + (1 - f0) * pow(1 - (l * h), 5)
float3 F_Schlick(float3 f0, float f90, float cos)
{
    return f0 + (f90 - f0) * pow(1 - cos, 5);
}

float F_Schlick(float f0, float f90, float cos)
{
    return f0 + (f90 - f0) * pow(1 - cos, 5);
}
```

这时又有如何决定 f0 和 f90 的问题了，f90 前面提到过通常为1，因此直接使用1即可，而 f0 的经过测量法线电解质的 f0 通常很小，而金属的 f0 为其颜色，因此 f0 使用以下式子决定：
```hlsl
// 电介质的法线入射的亮度
static float3 s_DielectricSpecular = float3(0.04, 0.04, 0.04);

float3 f0 = lerp(s_DielectricSpecular, surface.color, surface.metallic);
```

最后镜面反射分量的 BRDF 如下：
```hlsl
// 镜面反射项
float3 SpecularBRDF(float NoV, float NoL, float NoH, float VoH, float3 f0, float roughness)
{
    float ND = D_GGX(NoH, roughness);
    float GV = V_SmithGGXCorrelated(NoV, NoL, roughness);
    float3 F = F_Schlick(f0, 1.0, VoH);
    return ND * GV * F;
}
```

##### 漫反射分量的 BRDF

​	漫反射分量的 BRDF 同样由多种模型，常用的 Lambertian 模型的 BRDF 假设半球上的漫反射响应都是均匀的，其公式为：
<img src="https://img2024.cnblogs.com/blog/3406761/202510/3406761-20251016200644533-638701636.png" alt="image" style="zoom:67%;" />

其中 σ 为漫反射率，也就是物体的颜色。而迪士尼漫反射模型将粗糙度考虑了进去，其公式为：
<img src="https://img2024.cnblogs.com/blog/3406761/202510/3406761-20251016201006228-1173641047.png" alt="image" style="zoom:80%;" />

其将 f0 设置为了1，而 f90 的计算方式如下：
<img src="https://img2024.cnblogs.com/blog/3406761/202510/3406761-20251016201106923-55064370.png" alt="image" style="zoom:80%;" />

其实我也挺迷惑为何要这么计算的。

最后漫反射分量的 BRDF 如下：
```hlsl
// 迪士尼漫反射项
float DiffuseBurley(float NoV, float NoL, float LoH, float roughness)
{
    float f90 = 0.5 + 2.0 * roughness * LoH * LoH;
    float lightScatter = F_Schlick(1.0, f90, NoL);
    float viewScatter = F_Schlick(1.0, f90, NoV);
    return lightScatter * viewScatter / s_PI;
}
```

最后将微表面模型的 BRDF 带入渲染方程即可：
![image](https://img2024.cnblogs.com/blog/3406761/202510/3406761-20251016202222260-985469476.png)

```hlsl
float3 ShadeLighting(Surface surface, Light light)
{
    float3 halfDir = normalize(light.direction + surface.viewDir);
    float NoV = saturate(dot(surface.normal, surface.viewDir));
    float NoL = saturate(dot(surface.normal, light.direction));
    float NoH = saturate(dot(surface.normal, halfDir));
    float LoH = saturate(dot(light.direction, halfDir));    // 与 VoH 相同

    float3 f0 = lerp(s_DielectricSpecular, surface.color, surface.metallic);
    float3 specular = SpecularBRDF(NoV, NoL, NoH, LoH, f0, surface.roughness);
    float3 diffuse = DiffuseBurley(NoV, NoL, LoH, surface.roughness);
    float3 diffuseCol = surface.color * (1 - surface.metallic);
    diffuse *= diffuseCol;

    float3 radians = NoL * light.color * (diffuse + specular);
    return radians * light.attenuation;
}
```

自此直接光照也算完全实现了，得到的结果如下：
<img src="https://img2024.cnblogs.com/blog/3406761/202510/3406761-20251016203356082-2006676767.png" alt="image" style="zoom: 50%;" />

#### 间接光照的计算

​	在光栅化中，间接光照的计算十分复杂，通常使用 SSR、IBL、LightMap、SSAO 等技术实现，但是在光追中由于光线追踪自生的全局性，因此实现更为简单，只需要在着色点再次往四周 Trace 光线即可，如何高效的 Trace 这些光线才是真正的难点。
​	这里没有像常规的光追一样生成随机光线并 Trace 光线，而是只沿着反射方向 Trace 光线，因此最后的结果是有偏的：

```hlsl
// 若表面很粗糙则不追踪反射光线
[branch]
if(surface.roughness <= 0.99f) {
    // 获得反射光线
    RayTracing::Ray reflectRay;
    reflectRay.origin = surface.position;
    reflectRay.direction = reflect(WorldRayDirection(), surface.normal);
    refColor = TraceRadianceRay(reflectRay, surface.recursionDepth);

    // 计算反射系数
    float3 f0 = lerp(s_DielectricSpecular, surface.color, surface.metallic);
    float cos = saturate(dot(surface.normal, surface.viewDir));
    float3 F = F_Schlick(f0, 1.0, cos);

    refColor.rgb *= F * (1 - roughness);
}
```

这里通过计算菲涅尔项的反射系数来决定反射颜色的权重，最后叠加上反射颜色并写入光追负载：
```hlsl
color += refColor.rgb;
payload.color = float4(color, surface.alpha);
```

### Miss Shader

​	光照计算使用的 Miss Shader 十分简单，直接返回了蓝色：
```hlsl
[shader("miss")]
void MissShader(inout RayTracing::RayPayload payload)
{
    payload.color = float4(0.529, 0.808, 0.922, 1);
}
```

而阴影计算的 Miss Shader 则是将可见性改为 true：
```hlsl
// 若阴影光线不与物体相交则表示可见
[shader("miss")]
void MissShader_Shadow(inout RayTracing::ShadowRayPayload payload)
{
    payload.visible = true;
}
```



## 程序图元的实现

​	程序图元的实现需要使用先前都未使用过的 Intersection Shader，在调用`TraceRay`之后，光线会遍历加速结构，若是遇到叶子节点，则会判断是否是三角图元，若不是三角形，则会在 Hit Group 的 Shader Table 查找其对应的 Intersection Shader，并调用，若是不存在则会继续遍历。由此可见我们需要添加的为程序图元的 Hit Group、对应的加速结构以及对应的 Shader Record。

### Intersection Shader

​	这里先实现程序图元的 Intersection Shader，目前暂时先实现了球、矩形和四边形的 Intersection Shader。Shader 本体如下：
```hlsl
// Procedural Geometry Local
ConstantBuffer<RayTracing::PrimitiveInstanceConstantBuffer> lPrimitiveInstanceCB : register(b2);

[shader("intersection")]
void IntersectionShader_AnalyticPrimitive()
{
    Ray ray = {ObjectRayOrigin(), ObjectRayDirection()};
    RayTracing::AnalyticPrimitive::PrimitiveType primType = (RayTracing::AnalyticPrimitive::PrimitiveType)lPrimitiveInstanceCB.primitiveType;
    
    float time;
    RayTracing::ProceduralPrimitiveAttributes attrs = (RayTracing::ProceduralPrimitiveAttributes)0;
    if(RayAnalyticPrimitiveIntersectionTest(ray, primType, attrs, time)){
        ReportHit(time, 0, attrs);
    }
}
```

我没有每个图元都单独实现一个 Intersection Shader，而是使用了常量缓冲区，内部储存了当前图元的类型，同时还构造了一根对象空间的光线，使用对象空间的原因是我们可以直接在 HLSL 中定义一个单位图元，该图元位于对象空间内，因此可以直接使用对象空间的光线进行相交检测，省去了将图元变换到世界空间的步骤。构造完光线之后，这里调用了我实现的函数`RayAnalyticPrimitiveIntersectionTest`，其定义如下：
```hlsl
bool RayAnalyticPrimitiveIntersectionTest(
    in Ray ray, 
    in AnalyticPrimitive::PrimitiveType primType, 
    out ProceduralPrimitiveAttributes attrs, 
    inout float time)
{
    switch(primType){
    case AnalyticPrimitive::PrimitiveType::Sphere:
        return RaySphereIntersectionTest(ray, attrs, time);
    case AnalyticPrimitive::PrimitiveType::Quad:
        return RayQuadIntersectionTest(ray, attrs, time);
    case AnalyticPrimitive::PrimitiveType::Cube:
        return RayCubeIntersectionTest(ray, attrs, time);
    default:
        return false;
    }
}
```

其会根据传入的图元类型调用对应的相交检测，同时其还有两个输出参数`attrs`与`time`，分别为交点的几何数据与光线起点到交点的距离。交点数据的定义如下：
```hlsl
struct ProceduralPrimitiveAttributes
{
    float3 normal;
    float2 uv;
    bool frontFace;
};
```

其中第三个参数表示了交点是物体的正面还是反面，根据该参数法线会对应的调整。在判断完是否相交后，Intersection Shader 还调用了一个内置函数`ReportHit`，该函数的定义如下：
```hlsl
template<attr_t>
bool ReportHit(float THit, uint HitKind, attr_t Attributes);
```

传入的第一个参数会决定该相交是否会被接受，接受的条件为 THit 在光线的区间内，也就是`RayTMin`和`RayTCurrent`，前面也提到过`RayTCurrent`在 Intersection Shader 中会返回当前最近的相交距离，若是 THit 在区间内，则会调用对应的 Any Hit Shader，没有 Any Hit Shader 则会直接改变光线的区间的最远值为 THit 并返回true。但是若是有 Any Hit Shader 且在内部调用了内置函数`IgnoreHit`，则当前提交会被是为失败，且不会更新光线的区间，同时 false 表示该 HIt 未被接受。
	判断几何体相交的具体实现这里就不过多解释了，具体的推导过程可以翻阅 RTR4 第22章对应的几何体相交，或是 RayTracingInOneWeekend 的对应部分，这里只放出代码。球的相交检测：

```hlsl
// 测试射线是否和球体相交
bool RaySphereIntersectionTest(
    in Ray ray, 
    out ProceduralPrimitiveAttributes attrs, 
    inout float time)
{
    // 在局部坐标中的球心和半径
    const float3 center = float3(0,0,0);
    const float radius = 1;

    float3 oc = center - ray.origin;
    float a = dot(ray.direction, ray.direction);
    float h = dot(ray.direction, oc);
    float c = dot(oc, oc) - radius * radius;
    float invA = 1.0f / a;

    float discriminant = h * h -  a * c;
    if (discriminant < 0) { // 没有根则不相交
        return false;
    }
    float sqrtD = sqrt(discriminant);
    float root = (h - sqrtD) * invA;  // 计算方程的根
    if (root < RayTMin() || root > RayTCurrent()) { //不再范围内
        root = (h + sqrtD) * invA;
        if (root < RayTMin() || root > RayTCurrent()) { //不再范围内
            return false;
        }
    }

    float3 pos = ray.origin + root * ray.direction; // 计算交点
    float3 posWS = mul(float4(pos,1), ObjectToWorld4x3()).xyz; // 转到世界空间
    float3 centerWS = mul(float4(center,1), ObjectToWorld4x3()).xyz;
    float3 normal = normalize(posWS - centerWS);

    float invPI = 1.f / s_PI;
    float theta = acos(normal.y);
    float phi = atan2(normal.z, normal.x);
    attrs.uv = float2((phi + s_PI) * 0.5f * invPI, theta * invPI);
    attrs.frontFace = dot(ray.direction, normal) < 0;
    attrs.normal = attrs.frontFace ? normal : -normal;
    
    time = root;

    return true;
}
```

正四边形的检测：
```hlsl
// 测试射线是否和四边形相交
bool RayQuadIntersectionTest(
    in Ray ray, 
    out ProceduralPrimitiveAttributes attrs, 
    inout float time)
{
    // 单位四边形的参数
    const float3 q = float3(-1, -1, 0);
    const float3 u = float3(2, 0, 0);
    const float3 v = float3(0, 2, 0);

    float3 w = float3(0, 0, 0.25f);
    float3 normal = float3(0, 0, 1);

    // 判断射线是否与平面相交
    float nd = dot(normal, ray.direction);
    if(abs(nd) < 1e-4f)
        return false;

    // 计算交点
    time = -dot(normal, ray.origin) / nd;
    float3 pos = ray.origin + time * ray.direction; // 计算交点

    float3 pq = pos - q;
    float alpha = dot(w, cross(pq, v));
    float beta = dot(w, cross(u, pq));

    if(!InRange(alpha, 0, 1) || !InRange(beta, 0, 1))
        return false;

    // 将法线变换到世界空间
    normal = mul(normal, (float3x3)transpose(WorldToObject4x3()));
    attrs.uv = float2(alpha, beta);
    attrs.frontFace = nd < 0;
    attrs.normal = attrs.frontFace ? normal : -normal;

    return true;
}
```

矩形的相交检测：
```hlsl
bool RayCubeIntersectionTest(in Ray ray, float3 boxMin, float3 boxMax, inout float time)
{
    float3 origin = ray.origin;
    float3 dir = ray.direction;

    // [min, max]
    const float3 invDir = 1.0f / dir;
    float3 t0 = (boxMin - origin) * invDir;
    float3 t1 = (boxMax - origin) * invDir;
    
    float2 interval = float2(RayTMin(), RayTCurrent());
    for (uint i = 0; i < 3; ++i) {
        const float tmin = (t0[i] < t1[i]) ? t0[i] : t1[i];
        const float tmax = (t0[i] < t1[i]) ? t1[i] : t0[i];

        if (tmin > interval.x) interval.x = tmin;
        if (tmax < interval.y) interval.y = tmax;

        if (interval.x >= interval.y) return false;
    }

    time = interval.x;
    return true;
}

bool RayCubeIntersectionTest(
    in Ray ray, 
    out ProceduralPrimitiveAttributes attrs, 
    inout float time)
{
    float3 boxMin = float3(-1,-1,-1);
    float3 boxMax = float3(1,1,1);

    if(!RayCubeIntersectionTest(ray, boxMin, boxMax, time)){
        return false;
    }

    float3 pos = ray.origin + time * ray.direction; // 计算交点

    float3 boxVec = pos / max(max(abs(pos.x), abs(pos.y)), abs(pos.z));
    float3 normal = float3(
        abs(boxVec.x) > 0.9999 ? sign(boxVec.x) : 0,
        abs(boxVec.y) > 0.9999 ? sign(boxVec.y) : 0,
        abs(boxVec.z) > 0.9999 ? sign(boxVec.z) : 0
    );

    // 将法线变换到世界空间
    normal = mul(normal, (float3x3)transpose(WorldToObject4x3()));
    attrs.uv = (pos.xy - boxMin.xy) * 0.5f;
    attrs.frontFace = dot(ray.direction, normal) < 0;
    attrs.normal = attrs.frontFace ? normal : -normal;

    return true;
}
```

### 创建加速结构

​	这里我单独实现了一个类来辅助程序图元的创建，其实如下：
```C++
// 统一管理所有的程序图元
class ProceduralGeometryManager
{
public:
    ProceduralGeometryManager();

    void AddGeometry(const ProceduralGeometryDesc& desc);

    const std::vector<ProceduralGeometry>& GetAllGeometry() const { return m_ProceduralGeometrys; }


private:
    // 底层加速结构
    std::array<AccelerationStructureBuffers, RayTracing::AnalyticPrimitive::Count> m_AnalyticPrimitives{};

    // 程序图元实例数据
    std::vector<ProceduralGeometry> m_ProceduralGeometrys{};
    uint32_t m_InstanceContributionToHitGroupIndex = 0;
};
```

在该类构造函数中会创建所有图元的底层加速结构，而每次添加几何体时并不会增加底层加速结构，而是添加一个程序图元的实例`ProceduralGeometry`，其实现如下：
```C++
struct ProceduralGeometry
{
    RayTracing::AnalyticPrimitive::PrimitiveType type;	// 图元类型
    std::shared_ptr<Material> material;	// 材质
    D3D12_RAYTRACING_INSTANCE_DESC instanceDesc{};	// 实例
    uint32_t srvOffset = 0;	// 纹理在 Descriptor Heap 中的偏移
};
```

创建底层加速结构的具体过程这里就不赘述了，这里提一下在填写`D3D12_RAYTRACING_GEOMETRY_DESC`需要指定的类型为 AABB 且填写的数据为底层加速结构的 AABB，具体操作如下：
```C++
D3D12_RAYTRACING_AABB aabb{
    .MinX = -1.0f, .MinY = -1.0f, .MinZ = -1.0f,
    .MaxX =  1.0f, .MaxY =  1.0f, .MaxZ =  1.0f
};
D3D12_RAYTRACING_GEOMETRY_AABBS_DESC aabbDesc{};
aabbDesc.AABBCount = static_cast<UINT>(aabb.size());
aabbDesc.AABBs.StartAddress = aabbBuffer.GetGpuVirtualAddress();
aabbDesc.AABBs.StrideInBytes = sizeof(D3D12_RAYTRACING_AABB);
D3D12_RAYTRACING_GEOMETRY_DESC geometryDesc{};
geometryDesc.Type = D3D12_RAYTRACING_GEOMETRY_TYPE_PROCEDURAL_PRIMITIVE_AABBS;
geometryDesc.Flags = D3D12_RAYTRACING_GEOMETRY_FLAG_OPAQUE;
geometryDesc.AABBs = std::move(aabbDesc);
```

这里需要注意的是填写 AABB 时需要是对象空间下的 AABB，不需要进行变换，因为光线在遍历加速结构时是出于对象空间中，同时也需要保证该 AABB 能完全包围程序图元，不然会渲染错误。
	在创建图元的时候需要填写对应的描述结构体，并调用`AddGeometry`，描述结构体具体如下：

```C++
struct ProceduralGeometryDesc
{
    RayTracing::AnalyticPrimitive::PrimitiveType type;	// 图元类型
    Transform transform;	// 几何变换
    std::shared_ptr<Material> material;	// 材质
    std::array<TextureRef, kNumTextures> textures;	// 使用的纹理
};
```

随后就会创建对应的实例并插入数组中，具体操作如下：
```C++
void ProceduralGeometryManager::AddGeometry(const ProceduralGeometryDesc &desc)
{
    auto createInstance = [this, &desc](auto primitiveType){
        assert(desc.material != nullptr);

        const auto& sphereBottomLevelAS = m_AnalyticPrimitives[primitiveType].accelerationStructure;
        D3D12_RAYTRACING_INSTANCE_DESC instanceDesc{};
        instanceDesc.InstanceID = static_cast<UINT>(m_ProceduralGeometrys.size());
        instanceDesc.InstanceMask = RayTracing::TraceRayParameters::InstanceMark;
        instanceDesc.Flags = D3D12_RAYTRACING_INSTANCE_FLAG_FORCE_OPAQUE;
        instanceDesc.InstanceContributionToHitGroupIndex = m_InstanceContributionToHitGroupIndex;
        instanceDesc.AccelerationStructure = sphereBottomLevelAS.GetGpuVirtualAddress();
        DirectX::XMStoreFloat3x4(&reinterpret_cast<DirectX::XMFLOAT3X4&>(instanceDesc.Transform), desc.transform.GetLocalToWorld());
        if(desc.type == RayTracing::AnalyticPrimitive::Quad){
            instanceDesc.Transform[2][2] = 1;
        }

        D3D12_CPU_DESCRIPTOR_HANDLE defaultTexture[kNumTextures] = {
            Graphics::GetDefaultTexture(Graphics::kWhiteOpaque2D),
            Graphics::GetDefaultTexture(Graphics::kWhiteOpaque2D),
            Graphics::GetDefaultTexture(Graphics::kWhiteOpaque2D),
            Graphics::GetDefaultTexture(Graphics::kWhiteOpaque2D),
            Graphics::GetDefaultTexture(Graphics::kBlackTransparent2D),
            Graphics::GetDefaultTexture(Graphics::kDefaultNormalTex)
        };
        for(int i = 0; i < desc.textures.size(); ++i){
            if(desc.textures[i].IsValid()){
                defaultTexture[i] = desc.textures[i].GetSRV();
            }
        }
        DescriptorHandle texHandle = g_Renderer.m_TextureHeap.Allocate(kNumTextures);

        std::uint32_t destCount = kNumTextures;
        std::uint32_t srcCount[kNumTextures] = {1,1,1,1,1,1};
        g_RenderContext.GetDevice()->CopyDescriptors(
            1, &texHandle, &destCount, destCount, defaultTexture, srcCount, D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV);

        GpuBufferDesc materialBufferDesc{};
        materialBufferDesc.m_Size = sizeof(MaterialConstantBuffer);
        materialBufferDesc.m_Stride = sizeof(MaterialConstantBuffer);
        ProceduralGeometry geometry{};
        geometry.type = desc.type;
        geometry.material = desc.material;
        geometry.instanceDesc = std::move(instanceDesc);
        geometry.srvOffset = g_Renderer.m_TextureHeap.GetOffsetOfHandle(texHandle);
        m_ProceduralGeometrys.push_back(std::move(geometry));
        m_InstanceContributionToHitGroupIndex += RayTracing::RayType::Count;
    };

    switch (desc.type) {
        case RayTracing::AnalyticPrimitive::Sphere:
            createInstance(RayTracing::AnalyticPrimitive::Sphere); break;
        case RayTracing::AnalyticPrimitive::Quad:
            createInstance(RayTracing::AnalyticPrimitive::Quad); break;
        case RayTracing::AnalyticPrimitive::Cube:
            createInstance(RayTracing::AnalyticPrimitive::Cube); break;
        default:
            break;
    }
}
```

​	有了这个辅助类，在创建加速结构的时候也方便了许多，只需要直接将实例`D3D12_RAYTRACING_INSTANCE_DESC`插入数组即可：
```C++
// 添加程序图元的实例
for(const auto& geometry : m_ProceduralGeometryManager->GetAllGeometry()){
    auto instanceDesc = geometry.instanceDesc;
    instanceDesc.InstanceContributionToHitGroupIndex += instanceContributionToHitGroupIndex;
    instanceDescs.push_back(std::move(instanceDesc));
}
```

注意这里的在 Shader Table 中的偏移需要加上全部三角图元的偏移，因为他们共用同一个 Shader Table。

### 添加 Shader Record

​	在 Hit Group 的 Shader Table 中，还需要为程序图元额外添加 Shader Record，由于两种图元都共用一个 Miss Shader 因此，Miss Shader Table 就不需要添加了。具体的流程也是获取 Hit Group 其对应的着色器标识，然后填写 Shader Record 中的根参数：
```C++
... // 三角图元的 Shader Identifier

std::array<void*, RayTracing::RayType::Count> hitGroupIdentifiersAABB{};
for (size_t i = 0; i < RayTracing::RayType::Count; i++) {
    hitGroupIdentifiersAABB[i] = stateObjectProps->GetShaderIdentifier(Renderer::s_HitGroupName_AABB[i]);
}

... // 三角图元的 Shader Record
for(const auto& geometry : m_ProceduralGeometryManager->GetAllGeometry()){
    for(int i = 0; i < RayTracing::RayType::Count; ++i){
        hitGroupShaderRecordData.clear();
        hitGroupShaderRecordData.resize(hitGroupShaderRecordSize, 0);
        memcpy(hitGroupShaderRecordData.data(), hitGroupIdentifiersAABB[i], shaderIdSize);
        if(i == RayTracing::RayType::Radiance){
            LocalRootSignature::AABB::RootArguments rootArgs{};
            rootArgs.primitiveInstance.primitiveType = geometry.type;
            rootArgs.material.baseColor = geometry.material->baseColor;
            rootArgs.material.emissiveColor = geometry.material->emissiveColor;
            rootArgs.material.roughnessFactor = geometry.material->roughnessFactor;
            rootArgs.material.metallicFactor = geometry.material->metallicFactor;
            rootArgs.material.normalTexScale = geometry.material->normalTexScale;
            rootArgs.textures = g_Renderer.m_TextureHeap[geometry.srvOffset];
            memcpy(hitGroupShaderRecordData.data() + shaderIdSize, &rootArgs, sizeof(rootArgs));
        }
        hitGroupShaderTableData.append_range(hitGroupShaderRecordData);
    }
}
```

在创建 PSO 的时候还需要为程序图元创建对应的 Hit Group：
```C++
std::array<D3D12_HIT_GROUP_DESC, RayTracing::RayType::Count> hitGroupDescsAABB{};
for (int i = 0; i < RayTracing::RayType::Count; ++i) {
    auto& hitGroupDesc = hitGroupDescsAABB[i];
    hitGroupDesc.Type = D3D12_HIT_GROUP_TYPE_PROCEDURAL_PRIMITIVE;
    hitGroupDesc.HitGroupExport = s_HitGroupName_AABB[i];
    if(i == RayTracing::RayType::Radiance){        
        hitGroupDesc.ClosestHitShaderImport = s_ClosestHitShaderName[GeometryType::AABB];
    }
    hitGroupDesc.IntersectionShaderImport = s_IntersectionShaderName[IntersectionShaderType::AnalyticPrimitive];

    D3D12_STATE_SUBOBJECT hitGroupSubobject{};
    hitGroupSubobject.Type = D3D12_STATE_SUBOBJECT_TYPE_HIT_GROUP;
    hitGroupSubobject.pDesc = &hitGroupDesc;
    subobjects.push_back(std::move(hitGroupSubobject));
}
```

​	程序图元的 Closest Hit Shader 与三角形基本一样，只有获取法线、纹理坐标不一样，这里就不赘述了。最后得到的输出如下：
<img src="https://img2024.cnblogs.com/blog/3406761/202510/3406761-20251016231747259-92175710.png" alt="image" style="zoom:67%;" />

可以看到反射分量有点突兀，应为仅仅采样了反射方向的间接光照，这里使用的最大递归深度只有3，若是提高递归深度，还能得到更多的细节：
<img src="https://img2024.cnblogs.com/blog/3406761/202510/3406761-20251016232014237-1341101384.png" alt="image" style="zoom:67%;" />

这里将递归深度提高到了6，可以很明显的看到球和四边形那里反射除了额外的东西。

详细代码可见我的GitHub：https://github.com/wsdanshenmiao/LearnMiniEngine。
