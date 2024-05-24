# 使用DirectX实现纹理绘制

## 初始化纹理

+ 首先是创建矩形，这里使用X_Jun教程中提供的`Geometry.h`中的`Geometry::CreateBox`方法：
	```c++
	auto meshData = Geometry::CreateBox<VertexPosNormalTex>(3, 3, 3, XMFLOAT4(255.0f / 255.0f, 192.0f / 255.0f, 203.0f / 255.0f, 1.0f));
	ResetMesh(meshData);
	```

+ 随后初始化木箱和火焰箱的纹理：
	```C++
	HR(CreateDDSTextureFromFile(m_pd3dDevice.Get(), L"..\\Texture\\WoodCrate.dds", nullptr, m_pWoodCrate.GetAddressOf()));
	
	HR(CreateDDSTextureFromFile(m_pd3dDevice.Get(), L"..\\Texture\\flare.dds", nullptr, m_pFlare.GetAddressOf()));
	HR(CreateDDSTextureFromFile(m_pd3dDevice.Get(), L"..\\Texture\\flarealpha.dds", nullptr, m_pFlarealpha.GetAddressOf()));
	```

+ 接着初始化采样器：

	```C++
	D3D11_SAMPLER_DESC sampDesc;
	ZeroMemory(&sampDesc, sizeof(sampDesc));
	sampDesc.Filter = D3D11_FILTER_MIN_MAG_MIP_POINT;
	D3D11_TEXTURE_ADDRESS_MODE address = D3D11_TEXTURE_ADDRESS_BORDER;
	sampDesc.AddressU = address;
	sampDesc.AddressV = address;
	sampDesc.AddressW = address;
	sampDesc.BorderColor[0] = 0.0f;
	sampDesc.BorderColor[1] = 0.0f;
	sampDesc.BorderColor[2] = 0.0f;
	sampDesc.BorderColor[3] = 0.0f;
	sampDesc.ComparisonFunc = D3D11_COMPARISON_NEVER;
	sampDesc.MinLOD = 0;
	sampDesc.MaxLOD = D3D11_FLOAT32_MAX;
	HR(m_pd3dDevice->CreateSamplerState(&sampDesc, m_pSamplerState.GetAddressOf()));
	```

	这里过滤器(Filter)使用的是纹理缩小与放大和使用mipmap都使用常量插值法（也就是四舍五入）的`D3D11_FILTER_MIN_MAG_MIP_POINT`，产生的效果如下：<img src="https://img2024.cnblogs.com/blog/3406761/202404/3406761-20240405183213957-1490230579.png" style="zoom:67%;" />

	这时可以看到明显的锯齿，若是使用双线性插值，会产生更加平滑的过度，效果如下：<img src="https://img2024.cnblogs.com/blog/3406761/202404/3406761-20240405183806064-1878301511.png" style="zoom:67%;" />
	在这里使用各向异性过滤不会有太大的效果，但处理非正方形的矩形时会有显著效果。

	使用的寻址模式是**使用单一颜色填充标准纹理坐标外**的`D3D11_TEXTURE_ADDRESS_CLAMP`，这是为了后面使纹理旋转时可以用黑色遮挡边界外。
	以`D3D11_TEXTURE_ADDRESS_CLAMP`为例，若是想观察其他寻址模式的效果，可以通过更改顶点着色器的HLSL代码来观察：

	```c++
	vOut.tex = vIn.tex * 2;
	```

	便可观察到如下效果：<img src="https://img2024.cnblogs.com/blog/3406761/202404/3406761-20240405211548041-1906312921.png" style="zoom:50%;" />

+ 随后是设置着色器资源，注意因为后面要将两个纹理混合，所以这里要使用一二槽索引,且木箱的纹理也要设置在第二索引上，否则后面木箱的纹理会和火焰掩码图的纹理融合：
	```c++
	m_pd3dImmediateContext->PSSetShaderResources(0, 1, m_pWoodCrate.GetAddressOf());
	m_pd3dImmediateContext->PSSetShaderResources(1, 1, m_pWoodCrate.GetAddressOf());
	```

	在HLSL上也要设置对应的两个纹理数组：
	```c++
	Texture2D g_Tex[2] : register(t0);
	```

## 对纹理的操作

使用imGui控制物体的过程不在赘述，使用imGui进行纹理切换的方法如下。
这里使用组合框来选择使用的纹理：

```C++
static int curr_mode_item = static_cast<int>(m_CurrMode);
const char* mode_strs[] = {
    "WoodCrate",
    "FireBox"
};
if (ImGui::Combo("Mode", &curr_mode_item, mode_strs, ARRAYSIZE(mode_strs))) {
    if (0 == curr_mode_item) {
        m_CurrMode = ShowMode::WoodCrate;
        m_pd3dImmediateContext->PSSetShaderResources(0, 1, m_pWoodCrate.GetAddressOf());
        m_pd3dImmediateContext->PSSetShaderResources(1, 1, m_pWoodCrate.GetAddressOf());
    }
    else {
        m_CurrMode = ShowMode::FireBox;
        m_pd3dImmediateContext->PSSetShaderResources(0, 1, m_pFlare.GetAddressOf());
        m_pd3dImmediateContext->PSSetShaderResources(1, 1, m_pFlarealpha.GetAddressOf());
    }
}
```

切换纹理时需要重新绑定纹理资源，这里若不重新让木箱绑定第二索引，就会使木箱使用火焰掩码图的纹理，再加上后续的纹理混合，就会导致如下效果：

<img src="https://img2024.cnblogs.com/blog/3406761/202404/3406761-20240405213928377-1590535348.png" style="zoom:50%;" />

### 纹理旋转

要使纹理旋转，就要让纹理乘一个旋转矩阵，所以要在顶点缓冲区`VSConstantBuffer`的C++端和HLSL端新添一个**纹理旋转矩阵**：

```c++
DirectX::XMMATRIX texRotarion;
```

```HLSL
matrix g_texRotarion;   //纹理旋转
```

并在`GameApp::UpdateScene`方法中设置纹理旋转矩阵：
```c++
if (m_CurrMode == ShowMode::FireBox) {
    m_VSConstantBuffer.texRotarion = XMMatrixTranspose(XMMatrixRotationZ(texphi));
}
else {
    m_VSConstantBuffer.texRotarion = XMMatrixTranspose(XMMatrixRotationZ(0.0f));
}
```

但只是这样做会导致纹理绕着纹理坐标的（0，0）处旋转，也就是左上角，因此要**先让纹理移动到纹理坐标的中心**，**后旋转**，**再移动回去**，该操作可在C++端和HLSL端实现，这里再HLSL端实现：
```C++
//要移动到中心后在旋转
vOut.tex = mul(float4(vIn.tex.x - 0.5f, vIn.tex.y - 0.5f, 0.0f, 1.0f), g_texRotarion).xy;
//返回原位
vOut.tex += 0.5;
```

### 纹理融合

旋转后还要将火焰的纹理与其掩码图的纹理进行融合，直接让**两个纹理的分量相乘**即可：
```HLSL
float4 texColor = g_Tex[0].Sample(g_SamLinear, pIn.tex) * g_Tex[1].Sample(g_SamLinear, pIn.tex);
```

最后就可以产生如下效果：![](https://img2024.cnblogs.com/blog/3406761/202404/3406761-20240405215532066-1093100616.png)


