# Unity URP 体积云

​		好久之前开的体积云，因为期末考试和过年拖了很久，这几天才算整完。记录一样实现的思路，方便日后忘记了回来复习。

​		云的渲染有多种实现方法，我实现的是基于**RayMarching**的体积云体渲染，也就是将云视为孤立的元素，光线经过云这种介质之后发生作用进入人眼，由于云不会自发光，因此作用在介质上的四种行为：**吸收、外散射、内散射、发射**，就少了发射这一项，因此只需要模拟出光线经过云的剩下三种行为就可实现体积云的渲染。





## 后处理准备

​		要模拟光线进入人眼，使用RayMarching自然是一个很好的选择，即从摄像机出发，沿**观察方向**逐步前进，并在前进的过程中**采样物体的信息**。由于渲染云的时候需要考虑其他物体的影响，因此这里选择在渲染完场景中所有物体后的屏幕后处理阶段来渲染云，而在Unity的内置渲染中，只需要在`OnRenderImage`这个回调函数中进行云渲染即可。但是在URP中`OnRenderImage`无法使用，因此需要**自己创建一个Pass来进行后处理操作**。

​		首先需要创建一个继承自`ScriptableRenderPass`的类，这里命名为`VolumeCloudRenderPass`，随后重写纯虚函数`Execute`。为了方便调整参数，这里将体积云使用的参数都整合到了URP的屏幕后处理框架中，及创建一个继承自`VolumeComponent`和`IPostProcessComponent`的类：
```c#
public class VolumeCloudParamer : VolumeComponent, IPostProcessComponent
{ ... }
```

这时后处理的配置文件便会多出一个该类添加选项添加即可。
<img src="https://img2024.cnblogs.com/blog/3406761/202502/3406761-20250205190414161-1327864874.png" style="zoom: 80%;" />

随后在`VolumeCloudRenderPass`中添加一个`VolumeCloudParamer`的私有成员。除此之外还需要为其添加一个自定义构造函数，以便获取需要变量。因此自定义Pass的类代码大致如下：
```C#
public class VolumeCloudRenderPass : ScriptableRenderPass
{
    private VolumeCloudParamer m_VolumeCloudParamer;
    private RTHandle m_RenderTarget;
    private Material m_VolumeCloudMat;
    private ProfilingSampler m_ProfilingSampler = new ProfilingSampler("VolumeCloud");

    public bool EnableEdit => 
        m_VolumeCloudParamer == null ? false : m_VolumeCloudParamer.m_EnableInEdit.value;

    public VolumeCloudRenderPass(VolumeCloudFeature.VolumeCloudSetting setting)
    {
        if(setting.m_VolumeCloudShader == null){
            Debug.LogError("VolumeCloud Shader Should Be Set.");
            return;
        }

        this.renderPassEvent = setting.m_RenderPassEvent;
        this.m_FrameBlock = setting.m_FrameBlock;
        this.m_VolumeCloudMat = setting.m_VolumeCloudMat;
    }

    public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
    {
        // 获取后处理组件
        m_VolumeCloudParamer = VolumeManager.instance.stack.GetComponent<VolumeCloudParamer>();

        bool showInEdit = renderingData.cameraData.cameraType == CameraType.Game ||
            m_VolumeCloudParamer.m_EnableInEdit.value;
        if(!renderingData.cameraData.postProcessEnabled || 
            m_VolumeCloudParamer == null ||
            m_VolumeCloudMat == null ||
            !showInEdit) return;

        CommandBuffer cmd = CommandBufferPool.Get();
        // 添加一个分析项，方便在帧调试器中定位
        using (new ProfilingScope(cmd, m_ProfilingSampler)) {
            Render(cmd, renderingData);
        }
        context.ExecuteCommandBuffer(cmd);
        cmd.Clear();
        CommandBufferPool.Release(cmd);
    }
    
    public void SetUp(RTHandle renderTarget)
    {
        m_RenderTarget = renderTarget;
    }

    private void Render(CommandBuffer cmd, RenderingData renderingData)
    { ... }
}
```

其中`VolumeCloudFeature.VolumeCloudSetting`为后续定义的一个辅助配置类，成员函数`Render`既是实现体积云渲染的地方。

​		随后需要定义一个继承自`ScriptableRendererFeature`的类，并定义一个上文提到的辅助配置类：
```C#
public class VolumeCloudFeature : ScriptableRendererFeature
{
	[System.Serializable]
    public class VolumeCloudSetting
    {
        public bool m_EnableInEdit = false;
        public RenderPassEvent m_RenderPassEvent = RenderPassEvent.BeforeRenderingPostProcessing;
        public Shader m_VolumeCloudShader;
        public FrameBlock m_FrameBlock = FrameBlock._2X2;
        [HideInInspector] public Material m_VolumeCloudMat = null;
    }
```

由于需要在后处理阶段进行云渲染，因此`RenderPassEvent`需要选择为`BeforeRenderingPostProcessing`或其后的阶段。随后重写纯虚函数：

```C#
public class VolumeCloudFeature : ScriptableRendererFeature
{
    [System.Serializable]
    public class VolumeCloudSetting
    {
        public bool m_EnableInEdit = false;
        public RenderPassEvent m_RenderPassEvent = RenderPassEvent.BeforeRenderingPostProcessing;
        public Shader m_VolumeCloudShader;
        public FrameBlock m_FrameBlock = FrameBlock._2X2;
        [HideInInspector] public Material m_VolumeCloudMat = null;
    }

    private VolumeCloudRenderPass m_VolumeCloudRenderPass;
    public VolumeCloudSetting m_VolumeCloudSetting;

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        bool disableEdit = !m_VolumeCloudRenderPass.EnableEdit;
        if (renderingData.cameraData.cameraType != CameraType.Game && disableEdit) return;

        renderer.EnqueuePass(m_VolumeCloudRenderPass);
    }

    public override void Create()
    {
        name = "VolumeCloud";
        if(m_VolumeCloudSetting.m_VolumeCloudMat == null){
            m_VolumeCloudSetting.m_VolumeCloudMat = 
                CoreUtils.CreateEngineMaterial(m_VolumeCloudSetting.m_VolumeCloudShader);
        }

        m_VolumeCloudRenderPass = new VolumeCloudRenderPass(m_VolumeCloudSetting);
    }

    public override void SetupRenderPasses(ScriptableRenderer renderer, in RenderingData renderingData)
    {
        bool disableEdit = !m_VolumeCloudRenderPass.EnableEdit;
        if (renderingData.cameraData.cameraType != CameraType.Game && disableEdit) return;
        m_VolumeCloudRenderPass.ConfigureInput(ScriptableRenderPassInput.Color);
        m_VolumeCloudRenderPass.SetUp(renderer.cameraColorTargetHandle);
    }
}
```

还需要为`VolumeCloudRenderPass`添加一个成员函数`SetUp`来获取相机的RenderTarget以便进行后处理。

最后在URPRendererData中添加创建的Feature即可。

<img src="https://img2024.cnblogs.com/blog/3406761/202502/3406761-20250206004213267-1515835113.png" style="zoom:80%;" />





## 重构世界空间

​		由于需要获取观察方向，因此还需要从**屏幕空间中重构世界空间**。重构世界空间有**逆矩阵法**和**射线法**两种方法，其中**射线法的效率较高**，不过由于先前没用过逆矩阵法，因此这里使用逆矩阵法。

顶点着色器只需要将屏幕坐标输出即可：

```hlsl
v2f vertVolumeCloud(appdata v)
{
    v2f o;
    
    VertexPositionInputs vertPosInput = GetVertexPositionInputs(v.vertex.xyz);
    o.vertex = vertPosInput.positionCS;
    o.uv = v.uv;

    return o;
}

```

重构操作在像素着色器内进行，首先需要从**屏幕空间**转换为范围为[0, 1]的纹理坐，即进行如下操作：
```hlsl
// _ScreenParams的xy纹理宽度和高度，z分量是1.0 + 1.0/宽度，w为1.0 + 1.0/高度。
float2 posSS = i.vertex.xy * (_ScreenParams.zw - 1);
```

由于复原**标准化空间**需要使用当前像素的深度，因此要需要从深度缓冲区当中采样深度，并转换为线性深度，将xy映射到[-1, 1]就可获得标准化设备空间：
```hlsl
// 获取深度
float depth = SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, sampler_CameraDepthTexture, i.uv).x;
float linearDepth = LinearEyeDepth(depth, _ZBufferParams);
float4 posNDC = float4(posSS * 2 - 1, depth, 1);
```

由于不同的底层API所使用的左右手坐标系不同因此还需要消除平台差异：
```hlsl
#if UNITY_UV_STARTS_AT_TOP
    posNDC.y *= -1;
#endif
```

随后将**NDC空间**的坐标乘上观察矩阵与透视矩阵的逆矩阵即可，当然绝对不能忘了在NDC空间前面还有一个**齐次裁剪空间**，还需要复原**透视除法**这个操作，然而我们并不能直接获得透视除法中的w，因此就需要逆推：由于**世界空间下w分量为1**，随后**w分量乘上了VP矩阵**，到达齐次裁剪空间；因此为了还原世界坐标，需要在**齐次裁剪空间下将w定位1**，随后**乘上VP的逆矩阵**，最后让先前获得的坐标再除这个w分量即可。

```hlsl
#if REQUIRE_POSITION_VS
    float4 positionVS = mul(UNITY_MATRIX_I_P, posNDC);
    positionVS /= positionVS.w;
    float4 posW = mul(UNITY_MATRIX_I_V, positionVS);
#else
    float4 posW = mul(UNITY_MATRIX_I_VP, posNDC);
    posW /= posW.w;
#endif
```

至于为啥需要用`REQUIRE_POSITION_VS`分别处理，我也不是很清楚，但若是去掉在我的windows平台上没出现问题。最后若是输出坐标可得到如下效果：
<img src="https://img2024.cnblogs.com/blog/3406761/202502/3406761-20250206004133533-1932358020.png" style="zoom: 67%;" />

得到世界坐标后，再减去相机的坐标即可得到观察方向。





## RayMarching

准备工作终于结束了，接下来就是正式的体积云实现了。首先便是定义体积云的范围，需要定义一个AABB包围盒这里使用定义最远点与最近点的方式，还需要定义光线步进的最大次数，在观察方向创建一个光线，然后判断光线是否与AABB包围盒相交，这里使用的相交检测算法如下：
```hlsl
// abstract ： 判断射线与AABB盒是否相交
// return : 射线与包围盒是否相交,相交的近点和远点)
float3 RayInsertBox(float3 boxMin, float3 boxMax, float3 origin, float3 invDir)
{
    // 三个轴的 tEnter 和 tExit
    float3 tMins = (boxMin - origin) * invDir;    // invDir为(1/x,1/y,1/z)
    float3 tMaxs = (boxMax - origin) * invDir;
    for (int i = 0; i < 3; ++i) {
        if (invDir[i] < 0) {  // 若该轴为负从tMax进，tMin出,需要交换
            float tmp = tMins[i];
            tMins[i] = tMaxs[i];
            tMaxs[i] = tmp;
        }
    }
    float tMin = max(tMins.x, max(tMins.y, tMins.z));   // 取进入时间的最大值
    float tMax = min(tMaxs.x, min(tMaxs.y, tMaxs.z));   // 取离开时间的最小值

    if (tMin > tMax || tMax < 0) {
        return float3(0, tMin, tMax);
    }
    else{
        return float3(1, tMin, tMax);
    }
}
```

这里的相交算法返回值分别为是否相交、距离包围盒的最小距离、距离包围盒的最大距离。若是相交即进行光线步进，并沿着步进路线进行采样，所以RayMarching的主体框架如下：
```hlsl
float4 VolumeCloudRaymarching(Ray viewRay, float3 lightDir)
{
    // 总密度
    float sumDensity = 0;
    // 光照强度
    float3 lightIntensity = 0;

    float3 currPos = viewRay.startPos;
    float3 rayDir = viewRay.dir;

    // 相机视线与包围盒相交
    float3 invDir = 1 / rayDir;
    float3 insertInfo = RayInsertBox(_CloudBoxMin.xyz, _CloudBoxMax.xyz, currPos, invDir);

    // 未相交直接返回 0 光照强度
    if(insertInfo.x == 0) return float4(lightIntensity, transmittance);

	// 需要考虑再云内部的情况
    float enterPos = max(0, insertInfo.y);
    // 物体到相机的距离
    currPos += rayDir * enterPos;
    float marchingLimit = insertInfo.z - enterPos;

    float stepSize = marchingLimit / _ShapeMarchingCount;
    float3 rayStep = rayDir * stepSize;


    for(int i = 0; i < _ShapeMarchingCount; currPos += rayStep, ++i){
        // 计算当前点的密度
        float density = SampleDensity() * stepSize;
        lightIntensity += density;
        sumDensity += density;
    }
    
    return float4(lightIntensity, 1);
}
```

其实这就实现了一个简单的忽略物理的体积云渲染，这里使用假设采样的密度为0.1并将密度输出，会得到如下效果：
<img src="https://img2024.cnblogs.com/blog/3406761/202502/3406761-20250206004051446-1023739151.png" style="zoom:80%;" />

不过目前还没有考虑遮挡问题,若是此时有物体在体积云的前面，物体按理来说会遮挡体积云，但目前被会被体积云给遮挡。

<img src="https://img2024.cnblogs.com/blog/3406761/202502/3406761-20250209130036266-1150579956.png" style="zoom: 33%;" /><img src="https://img2024.cnblogs.com/blog/3406761/202502/3406761-20250209130438545-1513154857.png" style="zoom:33%;" />

此时就需要判断当前渲染的像素的的深度是否比与包围盒的最小距离要小，若是小则物体会把云给遮住，因此需要进行如下修改：
```hlsl
float marchingLimit =  insertInfo.z - insertInfo.y;
=>
// 考虑物体遮挡情况，选取最小距离
float marchingLimit = min(linearDepth, insertInfo.z) - insertInfo.y;
```

此时就会产生正确的遮挡关系。

<img src="https://img2024.cnblogs.com/blog/3406761/202502/3406761-20250209130535600-1817387052.png" style="zoom: 67%;" />





## 构造体积云的形状

​		云形状的构造我看了GPU Pro 7、一些Paper、很多知乎上的文章，都是使用同一种方法构造，即使用分形布朗运动( Fractal Brownian Motion,简称FBM)技术，该方法通过叠加不同频率和幅度的噪声来构造新的噪声。整个构造过程再`SampleDensity`函数中进行。GPU Pro 7中使用了三个纹理来构造云的形状:

+ 第一个为3D纹理，决定了云的基础形状，该纹理使用了四个通道，R通道储存了**Perlin噪声**，作为基础噪声来构造蓬松的形状；GBA三个通道都储存了不同频率的**反向Worley噪声**。
+ 第二个也为3D纹理，为云添加细节，该纹理使用了三个通道，为**频率依次递增**的的反向Worley噪声。
+ 第三个纹理为2D纹理，决定云的各种属性，R通道来储存**云的覆盖率**，可以自定义云的范围；G通道用来决定**云层的降雨量**，降雨量越大云层越暗，也就是吸收率越高；B通道用来决定**云层类型**，由低到高分为层云、层积云、积云，云层厚度一次增大。不过我未使用G通道和B通道。

### 辅助函数

构造云的形状需要使用几个辅助函数：

1. 重映射函数：可以将某个值从一个范围映射到另一个范围内。
	```hlsl
	// 重映射函数，将 value 从 (lo, ho) 映射到 (ln, hn)
	float Remap(float value, float lo, float ho, float ln, float hn)
	{
	    return ln + (value - lo) * (hn - ln) / (ho - lo);
	}
	```

2. 饱和函数：将数值限制再[0, 1]的范围内，用hlsl内置的saturate即可。

3. 高度百分比函数：计算当前云的高度占云的总高度的百分比。
	```hlsl
	// 计算某点的高度梯度
	float GetDensityHeightGradient(float3 pos, float min, float max)
	{
	    float heightGradient = (pos.y - min) / (max - min);
	    return saturate(heightGradient);
	}
	```

### 构造基础形状

​		构造基础形状需要使用第一张纹理，纹理的坐标索引使用当前像素的世界坐标，采样纹理后我们还需要再常量缓冲区中设置一个3维向量，用来当作**不同噪声的权重**，通过调节这三个权重即可调节云的形状。根据FBM的定义，需要将采样到的GBA相加来构造FBM，可以通过将**GBA通道与权重向量点乘**来同时达到这个目的，为了更多样的控制形状，还需要将权重进行**归一化**处理，最后通过Remap函数**将R通道的柏林噪声重映射**即可得到基础形状。具体操作如下：
```hlsl
// 获取体积云的基本形状
float3 sampleShapeUV = uvw * _SampleShapeScale + _SampleShapeOffset.xyz;
// 获取低频 Perlin-Worley 和 Worley 噪声
float4 shapeNoice = SAMPLE_TEXTURE3D(_ShapeNoiceTex, sampler_ShapeNoiceTex, sampleShapeUV);
// 计算不同频率 Worley 噪声所构成的 FBM 为基础形状添加细节
// FBM 是一系列噪声的叠加，每一个噪声都有更高的频率和更低的幅度。
float shapeFBM = dot(shapeNoice.gba, normalize(_ShapeWeights.rgb));
// 使用 FBM 重映射体积云的密度
density = Remap(shapeNoice.r, saturate(1 - shapeFBM), 1, 0, 1);
```

此时可得到如下效果（注意我这是加入了光照的，去掉光照就是一片白）：
<img src="https://img2024.cnblogs.com/blog/3406761/202502/3406761-20250207175030339-304564184.png" style="zoom: 50%;" />

此时对density进行适当的缩放和设定一个密度阈值就可以得到类似下面的效果：
<img src="https://img2024.cnblogs.com/blog/3406761/202502/3406761-20250207162023856-1935403546.png" style="zoom:25%;" /><img src="https://img2024.cnblogs.com/blog/3406761/202502/3406761-20250207162237833-992905763.png" style="zoom:25%;" />

此时云的顶部和底部都十分的平坦，需要一定的过度，因此可以通过Remap函数和当前位置的百分比来重新排布云的分布，具体的操作如下：
```hlsl
// 获取高度百分比
float heightGradient = GetDensityHeightGradient(currPos, _CloudBoxMin.y, _CloudBoxMax.y);

// 形状变化因子
// 圆化云的底部
float roundButton = saturate(Remap(heightGradient, 0, 0.07, 0, 1));
// 圆化云的顶部
float roundTop = saturate(Remap(heightGradient, 0.2, 1, 1, 0));
float roundFac = roundButton * roundTop;
```

同时由于边缘的云密度较淡，因此还需要根据高度重新排布云的密度分布：
```HLSL
// 密度变化因子
float densityButton = heightGradient * saturate(Remap(heightGradient, 0, _DensityOffset, 0, 1));
float densityTop = saturate(Remap(heightGradient, 1 - _DensityOffset, 1, 1, 0));
float densityFac = densityButton * densityTop * 2;

density *= roundFac * densityFac;
```

### 添加天气纹理

本来天气纹理是要限制云的分为的，但是加上之后我觉得边缘有点生硬,因此就换了一种实现方式，通过两次插值来重新更改云的密度：

```hlsl
// 为体积云添加天气属性
float2 weatherUV = (currPos.xz - cloudCenter.xz) / max(cloudSize.x, cloudSize.z);
weatherUV += windOffset;
// 获取天气纹理, r 通道存取体积云的覆盖百分比，g 通道存取云层降雨的可能性， b 通道存取云的类型
float4 weatherTex = SAMPLE_TEXTURE2D(_WeatherNoiceTex, sampler_WeatherNoiceTex, weatherUV);
// 云层覆盖率
float cloudCoverage = weatherTex.r;
cloudCoverage = lerp(cloudCoverage, 1, _CloudCoverage);
cloudCoverage = lerp(0, cloudCoverage, _CloudCoverage);
density *= cloudCoverage;
```

此时云的效果大致如下：
<img src="https://img2024.cnblogs.com/blog/3406761/202502/3406761-20250207221632609-317956093.png" style="zoom:50%;" />

### 添加细节

最后通过采样第二张3D纹理并构造对应的FBM，添加细节有通过重映射和腐蚀原密度两种方式，我是用的是后者来增加细节：
```hlsl
// 为云添加细节
float3 sampleDetailUV = uvw * _SampleDetailScale + _SampleDetailOffset.xyz;
// 获取高频的 Worley 噪声
float3 detailNoice = SAMPLE_TEXTURE3D(_DetailNoiceTex, sampler_DetailNoiceTex, sampleDetailUV).rgb;
// 计算 Worley 噪声的FBM
float detailFBM = dot(detailNoice, normalize(_DetailWeights.xyz));
float detailErode = (1 - detailFBM) * densityScale * _DetailScale;
density -= detailErode;
```

最后得到的效果如下：
<img src="https://img2024.cnblogs.com/blog/3406761/202502/3406761-20250207221722078-398790563.png" style="zoom: 50%;" />

<img src="https://img2024.cnblogs.com/blog/3406761/202502/3406761-20250207221758270-728649239.png" style="zoom: 50%;" />





## 体积云的光照

​		上面的图片都是添加了光照后的效果，接下来就介绍如何为体积云添加光照。前面的提到过光线通过介质会发生四种行为：**吸收、外散射、内散射、发射**，因此若是要考虑光照就需要分别对这四种行为进行模拟。这里讨论各个行为背后的物理原理，毕竟比较复杂，也需要一定的物理基础，只会写出使用到的模拟公式。

#### 辅助函数

这里先写出需要使用到的辅助函数，都为基于物理的模拟函数：

1. **Beer–Lambert’s Law**：用于计算光线再介质中的衰减，参数为光学深度。

	```hlsl
	// depth 为光学深度, 返回透射率
	float BeerLambert(float depth)
	{
	    return exp(-depth);
	}
	```

2. 吸光率（即光学深度）：参数分别为消光系数、密度、距离。
	```hlsl
	// 计算吸光率, A = t * d * l (t为消光系数,d为密度,l为距离)
	float CalcuAbsorbance(float t, float d, float l)
	{
	    return t * d * l;
	}
	```

3. **Henyey-Greenstein 函数**：用于模拟光线在介质中的散射。

	```hlsl
	// 改编自 Henyey-Greenstein 函数， 双 Henyey-Greenstein相位函数 的 双参数版 (原双相位函数拥有3个参数, 要确定3个参数非常复杂)
	// g : ( -0.75, -0.999 )
	//      3 * ( 1 - g^2 )               1 + cos^2
	// F = ----------------- * -------------------------------
	//      <4pi> * 2 * ( 2 + g^2 )     ( 1 + g^2 - 2 * g * cos )^(3/2)
	float HenyeyGreenstein(float cos, float anisotropy)
	{
	    float g = anisotropy;
	    float gg = g * g;
	
	    float a = 3 * (1 - gg);
	    float b = 8 * M_PI * (2 + gg);
	    float c = 1 + cos * cos;
	    float d = pow((1 + gg - 2 * g * cos), 3 / 2);
	
	    return a / b * c / d;
	}
	```

#### 光线吸收

Beer–Lambert’s Law常用来描述**光线在介质中的衰减**，即光线有多少被介质吸收，其函数为一个指数，函数参数为介质的光学厚度，其函数图如下：
<img src="https://img2024.cnblogs.com/blog/3406761/202502/3406761-20250207171521707-1686269976.png" style="zoom: 50%;" />

其中光学深度与**介质的消光系数和密度**有关，具体的计算上面的辅助公式2已列出，其中密度在进行RayMarching时已经通过采集噪声图并计算FBM获得；消光系数可通过自定义一个数值来指定，空气中灰尘、水滴、污染物的含量不同消光系数自然也就不同；距离在步进的过程中同样可以获得。

因此再RayMarching中需要添加如下操作：
```hlsl
lightIntensity += density * stepSize * transmittance;
// 计算吸光率
float absorbance = CalcuAbsorbance(_ExtinctionCoefficient, density, stepSize);
transmittance *= BeerLambert(absorbance);
```

#### LightMarching

​		先前的RayMarching为沿着视线的方向进行步进，在步进的同时采样各点的光线，也就是收集周围散射到视线上的光线，并计算其衰减，也就是下图B到P的过程，但是光线自光源传播到P点也会发生吸收，因此就需要额外构造一根光线来模拟这段吸收。

![](https://img2024.cnblogs.com/blog/3406761/202502/3406761-20250207223308526-1445466385.png)

思路与RayMarching一样从当前点向光源构造一条光线，并步进采集密度并计算透光率即可，由于使用的时方向光，因此不需要额外构造光线，直接使用光向量即可，具体操作如下：
```hlsl
// 从采样点出发，沿光照方向进行raymarching
float LightMarching(float3 currPos, float3 lightDir)
{
    // 总密度
    float sumDensity = 0;
    float transmittance = 1;

    float3 invDir = 1 / lightDir;

    float3 insertInfo = RayInsertBox(_CloudBoxMin.xyz, _CloudBoxMax.xyz, currPos, invDir);
    float stepSize = insertInfo.z / _LightMarchingCount;
    float3 rayStep = lightDir * stepSize;
    if(insertInfo.x != 0){
        [loop]
        for(int i = 0; i < _LightMarchingCount; currPos += rayStep, ++i){
            float density = SampleDensity(currPos, true);
            sumDensity += density;
        }
    }

    // 计算透光率
    float absorbance = CalcuAbsorbance(_ExtinctionCoefficient, sumDensity, stepSize);
    // 可选择加入糖粉效应
    transmittance = BeerLambert(absorbance);
	
	// 添加一个阈值来控制亮度
    return _DarknessThreshold + (1 - _DarknessThreshold) * transmittance;
}
```

### 相位函数

云的散射现象十分复杂，主要是由于其散射结果是不均匀的，主要由前向散射构成，其散射函数如下：
<img src="C:/Users/wsdanshenmiao/AppData/Roaming/Typora/typora-user-images/image-20250207231519593.png" alt="image-20250207231519593" style="zoom: 67%;" />

由于其相位函数十分复杂，因此可使用双Henyey-Greenstein相位函数来近似其散射结果在进行RayMarching之前提前计算好相位函数的结果，在计算完体积云的结果后直接乘上输出即可。
```hlsl
float cos = dot(rayDir, lightDir);
float phase = HenyeyGreenstein(cos, _CloudScatter);

...

lightIntensity *= phase;
```

自此体积云的基本实现就完成了。





## 性能优化

​		体积云的性能消耗是十分恐怖的，若不进行优化难以以实时计算的方式运行在普通设备上，因此接下来介绍几个优化方法。

### RayMarching优化

​		体积云的主要消耗都源自于光线步进，在均匀步进的情况下减少步进次数固然能降低消耗，但体积云的质量同样会下降，因此需要在保证质量情况下还想要减少步进次数，就需要使用进行**自适应的步进方式**。在云不是充满屏幕的情况下，其实大多数从相机发射的光线根本达不到云，同样的在光线进入包围盒，采样到云之前，和离开云之后所进行的采样都是无效的。因此若是在步进时**连续采样到大量的0密度**，基本上可以断定后续很大一段距离光线都不会打中云，基于此就诞生了一个优化思路。

#### 优化思路

+ 由于采样密度时会弱化底部和顶部，因此在进入包围盒后立刻开始采用更大的步进距离，下面简称大步进，并进行试探性的采样，由于细节纹理是在原形状的基础上进行腐蚀，因此试探性采样时可以不采样细节纹理，减少性能损耗。
+ 当遇到密度非 0 时立刻切换到普通步进，并且为了保证体积云的质量，还需要根据大步进的步进距离进行适当的步进回退，防止漏采样。
+ 后续步进采样累计连续密度为 0 的次数，若次数达到阈值则进入大步进模式。 大步进模式下若遇到密度非 0 时则同样立刻切换到普通步进并回退。

基于此优化思路，可将RayMarching的代码改为如下形式：
```hlsl
// rayMarching优化
float densityTest = 0;
float preDensity = 0;
int zeroDensityCount = 0;
[loop]
for(int i = 0; i < _ShapeMarchingCount; ++i){
    currPos += rayStep;

    // 检测是否进入大步进, 第一次进入大步进
    if(densityTest <= 0){   // 大步进
        densityTest = SampleDensity(currPos, false);

        if(densityTest > 0){
            currPos -= rayStep; // 回退一步，防止漏采样
            --i;
        }
        else{
            currPos += rayStep; // 额外步进一次
            ++i;
        }
    }
    else{   // 普通步进
        // 计算当前点的密度
        float density = SampleDensity(currPos, true);

        if(density > 0){
            zeroDensityCount = 0;

            float lightTransmittance = LightMarching(currPos, lightDir);
            lightIntensity += density * stepSize * transmittance * lightTransmittance;
            // 计算吸光率
            float absorbance = CalcuAbsorbance(_ExtinctionCoefficient, density, stepSize);
            transmittance *= BeerLambert(absorbance);
            sumDensity += density * stepSize;

            if(transmittance < 0.01) break;
        }
        else if(preDensity <= 0){
            ++zeroDensityCount;
        }

        // 0 密度次数达到阈值清空累计次数，进入大步进
        if(zeroDensityCount >= _LargeStepThreshold){
            zeroDensityCount = 0;
            densityTest = 0;
        }

        preDensity = density;
    }
}
```

其中采样函数多了一个参数，用来判断是否采样细节纹理。这里我将大步进设置为原来的两倍，其实可以进行更大距离的步进，具体使用多少可反复测试选择数值。

### 蓝噪声

​		尽管优化了RayMarching，步进的最大次数依然不能太大。由于无法做到连续的积分，因此才要使用隔一段距离采样一次的方式，而在这一段距离上所计算的结果都是相同的，所以在步进次数不够大的情况下就会出现明显的分层现象，造成明显的瑕疵。

<img src="https://img2024.cnblogs.com/blog/3406761/202502/3406761-20250207235916464-136943657.png" style="zoom: 33%;" /><img src="https://img2024.cnblogs.com/blog/3406761/202502/3406761-20250208000032179-1122581148.png" style="zoom:33%;" />

​		对于这种分层现象可使用蓝噪声来解决，蓝噪声为均匀的随机的无偏的噪声，是一种十分难以察觉的噪声，但同时也难以消除。我们可以利用这个特性来减轻由于起点相同而造成的分层现象，在进行光线步进之前，可提前采样蓝噪声，并将其映射到[-1, 1]的分为内，分别表示前进与后退，再将其叠加再初始位置上。注意由于使用了包围盒相交算法，因此需要再计算包围盒之后才使用蓝噪声，否则会因为抵达包围盒进行的快速步进而抵消。具体操作如下：
```hlsl
float blueNoice = SAMPLE_TEXTURE2D(_BlueNoiceTex, sampler_BlueNoiceTex, i.uv).r;

... 计算相交算法后

// 蓝噪声优化
float3 startOffset = (blueNoice - 0.5) * 2 * rayStep;
currPos += startOffset * _BlueNoiceScale;
```

引入噪声后分层现象明显改善了，但是同样的引入了噪声！），可使用较为偏移的算法来进行降噪，如TAA，若是后面有时间我也自己实现一下降噪。

<img src="C:/Users/wsdanshenmiao/AppData/Roaming/Typora/typora-user-images/image-20250208001253790.png" alt="image-20250208001253790" style="zoom:67%;" />

### 分帧渲染

​		前面的方法其实都是做些小修小改，虽然优化了但是在效率上没有发生质变。而分帧渲染就是一种让效率发生质变的方法，听名字就知道该方法是将体积云的渲染在时域上进行分摊。

​		分帧的渲染顺序并不是按普通的顺序来进行，而是以十字交叉的方式进行，这种方式在视觉上会带来更好的体验，如分16帧渲染，顺序排布就如下：
<img src="https://img2024.cnblogs.com/blog/3406761/202502/3406761-20250208003428161-1739340295.png" style="zoom: 80%;" />

要实现分帧渲染，Shader端与C#端都需要进行改动。首先是Shader端，先前在计算完云的颜色后直接使用系列代码进行输出：
```hlsl
return float4(preColor.rgb * cloud.a + cloud.rgb, preColor.a);
```

其中`cloud.a`为光线最后的透射率transmittance，也就是云背后的光线穿透云到达相机的比例。现在为了方便分帧渲染，可以让第一个Pass只渲染提及云，将混合操作单独分配到另一个Pass，混合Pass的具体操作如下：
```hlsl
float4 fragVolumeCloud(v2f i) : SV_Target
{
	...
	return cloud;
}

TEXTURE2D(_BlendCloudTex);
TEXTURE2D(_BackTex);
SAMPLER(sampler_BlendCloudTex);
SAMPLER(sampler_BackTex);

v2f vertBlendCloud(appdata v)
{
    v2f o;
    VertexPositionInputs vertexPos = GetVertexPositionInputs(v.vertex.xyz);
    o.vertex = vertexPos.positionCS;
    o.uv = v.uv;
    return o;
}

float4 fragBlendCloud(v2f i): SV_Target
{
    float4 cloud = SAMPLE_TEXTURE2D(_BlendCloudTex, sampler_BlendCloudTex, i.uv);
    float4 back = SAMPLE_TEXTURE2D(_BackTex, sampler_BackTex, i.uv);
    return float4(back.rgb * cloud.a + cloud.rgb, back.a);
}
```

接着还需要在常量缓冲区设置一个计数器来控制哪些像素需要绘制，此计数器在C#端进行更新，每次绘制体积云后次数加一。在Shader中还需要在像素着色器中的开头根据计数器来判断当前像素是否需要更新，同时使用预编译宏，也就是关键字来控制是否使用多帧渲染，具体操作如下：
```hlsl
// 计算当前的像素在像素块中的索引,按对角线十字进行计算
int GetPixelIndex(float2 uv, int width, int height, int blockCount)
{
    int frameBlock2X2[] = {
        0, 2, 3, 1
    };

    int frameBlock4X4[] = {
        0, 8, 2, 10,
        12, 4, 14, 6,
        3, 11, 1, 9,
        15, 7, 13, 5
    };

    int x = floor(uv.x * width) % blockCount;
    int y = floor(uv.y * height) % blockCount;
    int index = x + y * blockCount;

    if(blockCount == 2){
        index = frameBlock2X2[index];
    }
    else if(blockCount == 4){
        index = frameBlock4X4[index];
    }

    return index;
}

float4 fragVolumeCloud(v2f i) : SV_Target
{
#ifndef _FrameBlockOFF
    // 获取背景颜色
    float4 preColor = SAMPLE_TEXTURE2D(_CloudBackTex, sampler_CloudBackTex, i.uv);
    #ifdef _FrameBlock2X2
    int blockCount = 2;
    #elif _FrameBlock4X4
    int blockCount = 4;
    #endif

    int index = GetPixelIndex(i.uv, _TextureWidth, _TextureHeight, blockCount);
    
    [branch]
    if(index != _CurrFrameCount % (blockCount * blockCount)) return preColor;
#endif

	...
}
```

我将分帧渲染设置为有三种模式，分别为关闭、四帧渲染、16帧渲染，确定当前模式之后，会根据纹理坐标计算当前像素在整个纹理中所处的位置，并且根据预设好的顺序计算索引，最后根据索引和计数器的值来判断当前像素是否需要渲染。

​		开启分帧后效率虽然有很大的提升，但是由于启用分帧渲染本身会有开销，并且会造成大量的分支，因此不会说分四帧就提升四倍，十六帧就提升十六倍。

​		优化方面基本就差不多了，其实还有不直接进行LightMarching而是对光照进行近似等优化方法。整完优化体积云也算彻底完成了。



## 项目源码

https://github.com/wsdanshenmiao/VolumeCloud
