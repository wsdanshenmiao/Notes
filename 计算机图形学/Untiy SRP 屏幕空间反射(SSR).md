# Untiy SRP 屏幕空间反射(SSR)

​		前段时间在Unity中使用SRP实现了屏幕空间反射，同时尝试使用Unity内置的RenderGraph来管理渲染管线，项目的主要框架是基于该教程[Unity Custom SRP](https://catlikecoding.com/unity/custom-srp/)，记录一下实现的过程方便后面要使用的时候复习。



## 后处理准备

### 创建后处理设置及框架

​		由于SSR属于后处理的技术，因此在实现SSR之前还需要为后处理进行准备，URP中的后处理我已在这一篇文章[Unity 体积云](https://www.cnblogs.com/wsdanshenmiao/articles/18699933)中介绍，而这次我准备使用SRP及RenderGraph进行框架的搭建。

​		首先便是后处理的属性设置，为了方便在Unity中编辑后处理相关属性，我使用`ScriptableObject`作为后处理设置的父类：
```C#
public abstract class PostEffectSetting : ScriptableObject, IComparable<PostEffectSetting>
 {
     public int m_Weight = 0;
     public bool m_Enable = true;
     
     public int CompareTo(PostEffectSetting other)
     {
         if (ReferenceEquals(this, other)) return 0;
         if (other is null) return 1;
         return m_Weight.CompareTo(other.m_Weight);
     }

    public abstract void Record(
         RenderGraph renderGraph,
         CullingResults cullingResults,
         Camera camera,
         in CameraRendererTextures cameraTextures,
         TextureHandle target);
}
```

其中`m_Enable`和`m_Weight`这两个参数为所有后处理都有的属性，前者为是否开启该后处理，后者为该后处理的权重，由于不同后处理之间执行的顺序不同最后输出的结果也不同，因此需要根据权重进行排序。

​		接下来就是使用一个类统一管理所有的后处理效果，由于需要在Unity中编辑需要的后处理，因此需要添加`[Serializable] `，并提供一个开关所有后处理的开关，在`Record`中排序所有后处理并调用：

```C#
[Serializable] 
public class PostEffectManager
{
    private static readonly ProfilingSampler sm_Sampler = new ProfilingSampler("PostEffect");
    
    [FormerlySerializedAs("m_PostEffects")] [SerializeField] private List<PostEffectSetting> m_PostEffectSettings = new();
    [SerializeField] private bool m_Enabled = true;
    
    public bool IsActive => m_PostEffectSettings != null && m_PostEffectSettings.Count > 0 && m_Enabled;


    /// <summary>
    /// 执行屏幕后处理
    /// </summary>
    public void Record(
        RenderGraph renderGraph,
        CullingResults cullingResults,
        Camera camera,
        ScriptableRenderContext renderContext,
        in CameraRendererTextures cameraTextures)
    {
        if(!IsActive) return;
        using var groupSampler = new RenderGraphProfilingScope(renderGraph, sm_Sampler);

        m_PostEffectSettings.Sort(); // 根据后处理的权重进行排序
        
        for(int i = 0; i < m_PostEffectSettings.Count; i++)
        {
            if(m_PostEffectSettings[i] == null || !m_PostEffectSettings[i].m_Enable) continue;

            m_PostEffectSettings[i].Record(
                renderGraph,
                cullingResults, 
                camera, 
                cameraTextures,
                cameraTextures.m_ColorTexture);
        }

    }
}
```

最后在继承了`RenderPipelineAsset`的类中添加一个成员变量`PostEffectManager`即可，此时便可在Unity中自定义添加后处理。![](https://img2024.cnblogs.com/blog/3406761/202506/3406761-20250609112425699-240607274.png)

随后创建一个类继承`PostEffectSetting`来存储SSR中可调节的参数：

```C#
[CreateAssetMenu(menuName = "Rendering/Custom PostEffect/SSR")]
public class SSRPassSetting : PostEffectSetting
{
	...

    public override void Record(
         RenderGraph renderGraph,
         CullingResults cullingResults,
         Camera camera,
         in CameraRendererTextures cameraTextures,
         TextureHandle target)
    {
        SSRPass.Record(
            renderGraph,
            cullingResults,
            camera,
            cameraTextures,
            target,
            this);
    }
}
```

这里将实际的渲染逻辑委托给了另一个类`SSRPass`，具体的实现在该类中实现，该类的主要函数如下：

```c#
public class SSRPass
{
    private static ProfilingSampler sm_Sampler = new ProfilingSampler("SSR");

    private Material m_Material;

    public Material SSRMaterial{
        get{
            if (m_Material == null) {
                m_Material = CoreUtils.CreateEngineMaterial(Shader.Find("DSM RP/SSR"));
            }
            return m_Material;
        }
        
    }


    private TextureHandle m_SrcTexture, m_DstTexture, 
        m_DepthTexture, m_NormalTexture;

    private RendererListHandle m_RenderList;

    private int m_CameraWidth, m_CameraHeight;

    private SSRPassSetting m_Setting = null;

    public void Render(RenderGraphContext context)
    {
		if(m_Setting == null) return;
                    
        CommandBuffer cmd = context.cmd;
        
        SSRMaterial.SetTexture(CameraRendererTextures.m_CameraColorTextureId, m_SrcTexture);
        SSRMaterial.SetTexture(CameraRendererTextures.m_CameraDepthTextureId, m_DepthTexture);
        SSRMaterial.SetTexture(CameraRendererTextures.m_NormalTextureId, m_NormalTexture);
        
        cmd.SetRenderTarget(m_DstTexture, RenderBufferLoadAction.DontCare, RenderBufferStoreAction.Store);

        cmd.DrawProcedural(Matrix4x4.identity, SSRMaterial, 0, MeshTopology.Triangles, 3);

        context.renderContext.ExecuteCommandBuffer(cmd);
        cmd.Clear();
    }

    public static void Record(
        RenderGraph renderGraph,
        CullingResults cullingResults,
        Camera camera,
        in CameraRendererTextures cameraTextures,
        TextureHandle target,
        SSRPassSetting setting)
    {
		using RenderGraphBuilder ssrBuilder = renderGraph.AddRenderPass(
                sm_Sampler.name, out SSRPass pass, sm_Sampler);

        int width = camera.pixelWidth, height = camera.pixelHeight;

        pass.m_Setting = setting;
        pass.m_CameraWidth = width;
        pass.m_CameraHeight = height;

        // 使用颜色及深度图
        pass.m_SrcTexture = ssrBuilder.ReadWriteTexture(target);
        pass.m_DepthTexture = ssrBuilder.ReadTexture(cameraTextures.m_DepthTexture);
        pass.m_NormalTexture = ssrBuilder.ReadTexture(cameraTextures.m_NormalTexture);

        // 遮罩纹理
        TextureDesc texDesc = new TextureDesc(width, height)
        {
            name = "MaskTexture",
            format = GraphicsFormat.R8_SNorm,
        };
        pass.m_MaskTexture = ssrBuilder.ReadWriteTexture(renderGraph.CreateTexture(texDesc));

        // 临时纹理
        texDesc.format = SystemInfo.GetGraphicsFormat(DefaultFormat.HDR);
        texDesc.name = "TmpTexture";
        pass.m_DstTexture = ssrBuilder.WriteTexture(renderGraph.CreateTexture(texDesc));

        // 打包好的Hiz纹理
        texDesc.format = GraphicsFormat.R32_SFloat;
        texDesc.name = "Package Hiz Texture";
        texDesc.useMipMap = true;
        texDesc.autoGenerateMips = false;   // 不能自动生成MipMap，否则拷贝的会被覆盖
        pass.m_PackageHizTexture = ssrBuilder.ReadWriteTexture(renderGraph.CreateTexture(texDesc));

        // Hiz纹理
        pass.m_HizTextures = new TextureHandle[setting.m_HizCount];
        texDesc.useMipMap = false;
        for (int i = 0, hizWidth = width / 2, hizHeight = height / 2;
            i < pass.m_HizTextures.Length; i++, hizWidth /= 2, hizHeight /= 2)
        {
            texDesc.width = hizWidth;
            texDesc.height = hizHeight;
            texDesc.enableRandomWrite = true;
            texDesc.name = "HizTexture" + i;
            pass.m_HizTextures[i] = ssrBuilder.ReadWriteTexture(
                renderGraph.CreateTexture(texDesc));
        }

        pass.m_RenderList = ssrBuilder.UseRendererList(renderGraph.CreateRendererList(
            new RendererListDesc(SSRPass.m_ShaderTagID, cullingResults, camera)
            {
                renderQueueRange = RenderQueueRange.all,
                renderingLayerMask = setting.m_RenderingLayerMask,
                overrideMaterial = SSRPass.m_SSRLitMaterial,
                overrideMaterialPassIndex = 1
            }));

        ssrBuilder.SetRenderFunc<SSRPass>(
            static (pass, context) => pass.Render(context));
    }
```

在`Record`函数中，我们需要对需要使用的纹理进行创建或标记，并将准备好的纹理传递给调用RenderGraph的`AddRenderPass`输出的数据`pass`中。在`Render`中执行实际的渲染流程。

### 使用全屏三角形覆盖屏幕

​		由于后处理需要对渲染管线输出的纹理进行处理，因此需要创建一个覆盖整个纹理的三角形来渲染。相比使用四边形进行渲染，使用覆盖全屏的三角形进行渲染效率更高，原因如下：

+ 四边形在渲染时会被拆为两个三角形渲染，但是由于GPU执行光栅化的时候是以Tile为单位进行渲染的，也就是多个像素组成的像素块，因此在两个三角形的**衔接处的像素会被渲染多次**，造成Overdraw。
+ 相比四边形，使用三角形的**缓存命中率更高**。
+ 现代GPU在光栅化之前会使用**保护带裁剪(Guard Band Clipping)**来进行裁剪，因此全屏三角形超出屏幕的部分不会被裁剪掉，也不会造成额外的开销。

在顶点着色器中创建三角形的代码如下：
```hlsl
struct Varyings
{
    float4 posCS : SV_POSITION;
    float2 uv : TEXCOORD0;
};

/*
 *  1
 *  |\
 *  | \
 *  |  \
 *  |___ \
 *  0       2
 */
// 覆盖全屏的三角形
Varyings DefaultPostEffectVertex(uint vertexID : SV_VertexID)
{
    Varyings output;

    float2 uv = float2((vertexID << 1) & 2, vertexID & 2);
    output.uv = uv;
    output.posCS = float4(output.uv * 2.0 - 1.0, 0, 1.0);
    
    [flatten]
    if (_ProjectionParams.x < 0) {
        output.uv.y = 1 - output.uv.y;
    }
    
    return output;
}

Shader "DSM RP/SSR"
{
    SubShader
    {
        Cull Off
        ZTest Always
        ZWrite Off
        
        HLSLINCLUDE
        #include "../../../ShaderLibrary/Common.hlsl"
        #include "../PostEffectCommon.hlsl"
        #include "SSRPass.hlsl"
        ENDHLSL

        
        Pass
        {
            Name "SSR"
            Tags {"LightMode" = "DSMLit"}
            
            HLSLPROGRAM
            #pragma vertex DefaultPostEffectVertex
            #pragma fragment SSRPassFragment
            #pragma target 5.0
            ENDHLSL
        }
    }
}
```

其中`_ProjectionParams`为Unity内置的投影矩阵的信息，x值表示该平台下的矩阵是否为翻转矩阵，需要判断该值的正负来消除平台差异。在`SSRPassFragment`中将实现具体的算法，此时先输出一下uv来看看全屏三角形是否创建成功。![](https://img2024.cnblogs.com/blog/3406761/202506/3406761-20250609231227875-1868486476.png)

至此实现SSR的准备也算完成了，至于在SRP中法线纹理如何获取，我是先渲染物体的时候额外设置了一个RT：
```C#
RenderTargetIdentifier[] renderTargets = {
    m_ColorTexture, m_NormalTexture
};

cmd.SetRenderTarget(renderTargets, m_DepthTexture);
```

并在光照Pass中将法线输出到该RT中：

```hlsl
float4 LitPassFragment(Varyings i, out float3 normal : SV_TARGET1) : SV_TARGET0
{
	...
	
    normal = EncodeNormal(surface.normal);
}
```



## SSR原理

​		屏幕空间反射其实也是屏幕空间光线追踪，其原理也十分简单，大致可以分为以下三个步骤：

1. 在当前像素使用深度、法线等信息**构建射线**。
2. 沿着构建的射线**步进**，并**判断射线是否与步进处的物体相交**，通常使用深度来判断。
3. 若射线与物体相交则对相交处的像素及当前像素进行处里。

从对一二步采取不同的方法，该算法**视图空间SSR**及**屏幕空间SSR**。两者的区别为前者构建的是视图空间的射线，并在视图空间中步进；后者构建的是屏幕空间的2D射线并在2D空间步进，也就是纹理上画线，因此步进算法通常使用DDA算法。接下来便分别阐述两种方法的基本实现及优化。



## 视图空间SSR

### 构建射线

​		根据上面的三个步骤，首先需要构建视图空间的坐标，回顾一下一个顶点变换到屏幕的流程，分别经过了：局部空间——世界空间——视图空间——齐次裁剪空间——标准化设备(NDC)空间——屏幕空间。因此需要先获取NDC空间下坐标，该坐标可以通过将纹理坐标映射到[-1, 1]和当前像素的深度来获得。由于我们没有顶点变换时齐次裁剪操作的信息，因此只能通过将w设为1并最后除以w来处理，具体实现如下：
```C++
float GetCameraDepth(float2 uv)
{
    return SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, sampler_point_clamp, uv);
}

float3 GetViewPosition(float2 uv)
{
    float4 posCS = float4(uv * 2 - 1, GetCameraDepth(uv), 1);
    #if UNITY_UV_STARTS_AT_TOP
    posCS.y *= -1;
    #endif

    float4 posVS = mul(Inverse(UNITY_MATRIX_P), posCS);
    posVS /= posVS.w;

    return posVS.xyz;
}
```

由于不同图形API的纹理坐标标准不同，因此还需要通过Unity内置的宏`UNITY_UV_STARTS_AT_TOP`来消除平台差异，由于视图空间下相机是原点，因此得到视图坐标后也得到了视线方向，输出一下视线方向看看正不正确。![](https://img2024.cnblogs.com/blog/3406761/202506/3406761-20250609234359617-1038986463.png)

​		此时便可以从法线纹理中获取法线并将其变换到视图空间，SSR需要追踪的射线为反射光线，因此还需要通过法线和视线方向计算反射光线：
```hlsl
float4 SSRPassFragment(Varyings input) : SV_TARGET
{
    float4 reflectCol = float4(0, 0, 0, 1);
    
    // 获取RayMarching所需的信息
    float3 normal = SAMPLE_TEXTURE2D(_NormalTexture, sampler_NormalTexture, input.uv).xyz;
    [branch]
    if (all(normal == 0)) return reflectCol;    // 排除天空盒
    normal = DecodeNormal(normal);
    normal = normalize(mul((float3x3)unity_MatrixV, normal));
    float3 posVS = GetViewPosition(input.uv);
    float3 viewDir = normalize(posVS);
    float3 rayDir = normalize(reflect(viewDir, normal));
    
    Ray ray;
    ray.rayDir = rayDir;
    ray.origin = posVS;

    reflectCol = ViewSpaceSSR(ray);

    return reflectCol;
}
```

自此射线就构造完毕了，接下来就是在`ViewSpaceSSR`中追踪光线。

### RayMarching

​		又到了经典的RayMarching环节，该步骤设定一个步频并不断步进反射光线即可，在每次步进的时候还需要判断射线与当前位置的物体是否相交：

```hlsl
float4 RayMarching(Ray ray, float marchCount, float marchStep, out float4 outCol, out float3 endPos)
{
    float3 currPos = ray.origin;
    outCol = float4(0, 0, 0, 1);
    
    [loop]
    for (int i = 0; i < marchCount; ++i) {
        endPos = currPos;
        currPos += ray.rayDir * marchStep;

        float depthDis;
        // 当 depthDis > 0 时可以直接结束
        CheckCurrPos(currPos, outCol, depthDis);
        if (depthDis > 0) return outCol;
    }
    return outCol;
}
```

其中`CheckCurrPos`便是进行相交检测，进行检测需要还原当前位置对应的像素的uv，将当前位置**乘上投影矩阵**并手动进行**齐次除法**，随后**映射到[0, 1]**即可获得uv，通过uv来获取对应的深度并与当前的深度进行比较：

```hlsl
bool GetCurrDepthAndUV(float3 currPos, out float currDepth, out float2 uv)
{
    // 变换到NDC空间
    float4 posCS = mul(UNITY_MATRIX_P, float4(currPos, 1));
    currDepth = posCS.w; // 根据投影矩阵可以得到齐次裁剪空间下的 w 就是视图空间下的深度
    posCS.xyz /= posCS.w;
        
    // _ProjectionParams 为 1 或 -1
    // 重建 uv 来获取采样点的深度及颜色
    uv = float2(posCS.x, posCS.y * _ProjectionParams.x) * 0.5 + 0.5;
    
    return (0 <= uv.x && uv.x <= 1 && 0 <= uv.y && uv.y <= 1);
}

/*
 * return:
 *  true: 在当前位置获取了颜色信息
 *  false: 为获取颜色信息
 */
bool CheckCurrPos(float3 currPos, out float4 outCol, out float depthDis)
{
    outCol = float4(0, 0, 0, 1);
    // 获取当前位置的深度
    float currDepth;
    float2 uv;
    [branch]
    if (!GetCurrDepthAndUV(currPos, currDepth, uv)) {
        depthDis = 1;
        return false;
    }

    float depthTex = GetCameraLinearDepth(uv);
    depthDis = currDepth - depthTex;
    // 射线已经穿过了物体
    [branch]
    if (depthDis > 0) {
        bool inRange = depthDis <  _HitThreshold;  // 在阈值范围内
        outCol = inRange ? GetCameraColor(uv) : outCol;
        return inRange;
    }
    return false;
}
```

直接将返回的颜色叠加到源颜色上，此时便完成了一个最基本的SSR了。![](https://img2024.cnblogs.com/blog/3406761/202506/3406761-20250610001731944-1361052536.png)

从这张图可以看出屏幕空间反射的缺陷，由于最右边球的右端在屏幕之外，因此当地板反射的光线到达理应相交的位置时无法获得信息，导致反射的缺失，这是屏幕空间算法无法避免的缺陷。解决这个问题的一种方法是在相机处朝六个方向都进行渲染，当光线步进超出一个纹理的范围时便去其他纹理查找，但这方法开销很大因此我没见过有谁实现出来。

另一个很明显的问题是在离屏幕很近的的位置反射出来的球体有明显的分层现象，这里我选择的Ray Marching的步频为0.1，如此高的采样频率依然产生了明显分层，因此这个问题很难忽视。造成这个问题的原因是顶点经过投影变换后在**不同深度的分布是不均匀的**，因此同样长度的射线在近平面对应的像素量要远大于在远平面对应的像素量。![](https://img2024.cnblogs.com/blog/3406761/202506/3406761-20250610004015618-1256269504.png)

可参考这个图片，在屏幕平面，也就是近平面中ac与cb的长度相同，对应的像素相同，但是沿着视线方向两条直线对应的直线AC和CB并不是等长的。这就会导致当射线步进到的位置与相机的距离较近时，每次步进会略过更多的像素，造成**欠采样**影响质量；当步进到的位置与相机较远时，相邻的多次步进会采样到同一像素，造成**过采样**影响效率。而屏幕空间SSR的出现正是为了解决这些问题。

​		但在在视图空间下也是有补救的方法的，那便是使用二分查找法，具体实现也比较简单，当检测射线与物体相交的时候判断深度的差是否在阈值范围内，若不在阈值范围内则回退并减半步进距离：

```hlsl
// 二分查找，来准确定位反射光线打中的像素
float4 BinarySearch(Ray ray)
{
    static const int MaxBinarySearchCount = 5;   // 限制查找次数
    
    float step = _RayMarchingStep * 0.5;
    float3 currPos = ray.origin;
    float4 outCol = float4(0, 0, 0, 1);

    [unroll]
    for (int i = 0; i < MaxBinarySearchCount; i++) {
        float3 prePos = currPos;
        currPos += ray.rayDir * step;

        float depthDis;
        if (CheckCurrPos(currPos, outCol, depthDis)) break;
        [flatten]
        if (depthDis > 0) { // 回退
            currPos = prePos;
            step *= 0.5;
        }
    }
    return outCol;
}
```

需要注意的是需要考虑在二分起点与终点之间没有满足阈值深度的位置，当遇到这种情况会使二分无法停止，因此需要限制二分的次数，这里限制为5次。添加二分查找后的结果如下。![{BAE79CCB-4050-48D9-BACB-56BECB241C25}](C:/Users/wsdanshenmiao/AppData/Local/Packages/MicrosoftWindows.Client.CBS_cw5n1h2txyewy/TempState/ScreenClip/{BAE79CCB-4050-48D9-BACB-56BECB241C25}.png)

虽然还是不尽人意但相比上面的严重分层还是好了很多的，且步频越大效果越明显。

### 结果输出

​		原纹理与反射纹理的叠加我只用了混合叠加，单独添加了一个Blend Pass才混合两张纹理，具体实现如下：

```hlsl
Shader "DSM RP/Blend"
{
    Properties
    {
        [Enum(UnityEngine.Rendering.BlendMode)] _SrcBlend("Src Blend", Float) = 1
        [Enum(UnityEngine.Rendering.BlendMode)] _DstBlend("Dst Blend", Float) = 0
        [Enum(UnityEngine.Rendering.BlendOp)] _BlendOp("Blend Operation", Float) = 0
    }
    SubShader
    {
        Cull Off
        ZTest Always
        ZWrite Off
        
        HLSLINCLUDE
        #include "../ShaderLibrary/Common.hlsl"
        #include "PostEffect/PostEffectCommon.hlsl"
        #include "BlendPass.hlsl"
        ENDHLSL

        
        Pass
        {
            Name "Blend"
            Tags {"LightMode" = "DSMLit"}
            Blend [_SrcBlend] [_DstBlend]
            BlendOp [_BlendOp]
            
            HLSLPROGRAM
            #pragma vertex DefaultPostEffectVertex
            #pragma fragment BlendPassFragment
            #pragma target 5.0
            ENDHLSL
        }
    }
}


#ifndef __BLENDPASS_HLSL__
#define __BLENDPASS_HLSL__

TEXTURE2D(_SrcTexture);

float4 BlendPassFragment(Varyings i) :SV_Target
{
    return SAMPLE_TEXTURE2D(_SrcTexture, sampler_linear_clamp, i.uv);
}


#endif
```

为了方便后续对混合的使用，我创建了一个类来单独管理混合操作：

```c#
public readonly ref struct BlendSetting
{
    public readonly TextureHandle 
        m_SrcTexture, m_DstTexture;
    public readonly BlendMode m_SrcBlend;
    public readonly BlendMode m_DstBlend;
    public readonly BlendOp m_BlendOp;
    public BlendSetting(
        TextureHandle srcTex,
        TextureHandle dstTex,
        BlendMode srcBlend = BlendMode.SrcAlpha,
        BlendMode dstBlend = BlendMode.OneMinusSrcAlpha,
        BlendOp blendOp = BlendOp.Add)
    {
        m_SrcBlend = srcBlend;
        m_DstBlend = dstBlend;
        m_BlendOp = blendOp;
        m_SrcTexture = srcTex;
        m_DstTexture = dstTex;
    }
}

public class BlendPass
{
    private static readonly ProfilingSampler sm_Sampler = new ProfilingSampler("Blend");

    private BlendMode m_SrcBlend;
    private BlendMode m_DstBlend;
    private BlendOp m_BlendOp;

    private TextureHandle m_SrcTexture;
    private TextureHandle m_DstTexture;

    static private Material sm_Material;

    private static readonly string sm_BlendShaderName = "DSM RP/Blend";

    public static readonly int
        m_SrcBlendId = Shader.PropertyToID("_SrcBlend"),
        m_DstBlendId = Shader.PropertyToID("_DstBlend"),
        m_BlendOpId = Shader.PropertyToID("_BlendOp"),
        m_SrcTextureId = Shader.PropertyToID("_SrcTexture");

    private void Render(RenderGraphContext context)
    {
        sm_Material = sm_Material == null ? 
            CoreUtils.CreateEngineMaterial(sm_BlendShaderName) : sm_Material;

        CommandBuffer cmd = context.cmd;
        sm_Material.SetFloat(m_SrcBlendId, (float)m_SrcBlend);
        sm_Material.SetFloat(m_DstBlendId, (float)m_DstBlend);
        sm_Material.SetFloat(m_BlendOpId, (float)m_BlendOp);
        sm_Material.SetTexture(m_SrcTextureId, m_SrcTexture);

        cmd.SetRenderTarget(m_DstTexture);

        cmd.DrawProcedural(Matrix4x4.identity, sm_Material, 0, MeshTopology.Triangles, 3);
        
        context.renderContext.ExecuteCommandBuffer(cmd);
        cmd.Clear();
    }

    public static void Record(RenderGraph renderGraph, BlendSetting setting)
    {
        using RenderGraphBuilder builder = renderGraph.AddRenderPass(
            sm_Sampler.name, out BlendPass pass, sm_Sampler);

        pass.m_SrcBlend = setting.m_SrcBlend;
        pass.m_DstBlend = setting.m_DstBlend;
        pass.m_BlendOp = setting.m_BlendOp;
        pass.m_SrcTexture = builder.ReadTexture(setting.m_SrcTexture);
        pass.m_DstTexture = builder.WriteTexture(setting.m_DstTexture);

        builder.SetRenderFunc<BlendPass>(
            static (pass, context) => pass.Render(context));
    }
}
```

封装了混合操作后只需要在SSRPass中的`Record`函数中调用即可：

```C#
[CreateAssetMenu(menuName = "Rendering/Custom PostEffect/SSR")]
public class SSRPassSetting : PostEffectSetting
{
	...
    
    [Header("SSR Settings")]
    [Range(0, 1)] public float m_BlendFactor = 1;
    public BlendMode m_SrcBlend = BlendMode.SrcAlpha;
    public BlendMode m_SSRBlend = BlendMode.One;
    public BlendOp m_BlendOp = BlendOp.Add;
}

public static void Record(...)
{
    ...

    BlendSetting blendSetting = new BlendSetting(
        pass.m_DstTexture, pass.m_SrcTexture,
        setting.m_SSRBlend, setting.m_SrcBlend, setting.m_BlendOp);
    BlendPass.Record(renderGraph, blendSetting);
}
```

最后得到的结果如下：

![](https://img2024.cnblogs.com/blog/3406761/202506/3406761-20250610221937952-530277284.png)
