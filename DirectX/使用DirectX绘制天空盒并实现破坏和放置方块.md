# 使用DirectX绘制天空盒并实现破坏和放置方块

## 绘制天空盒

由于项目中的DxTex软件使用不了，所以直接使用了方法二，将项目中的文件名直接修改，不过这里要注意获取的六个正方形贴图要**用正确的顺序读取**，也就是+X，-X，+Y，-Y，+Z，-Z。

这里我是用了如下网址制作贴图：[Panorama to Cubemap (jaxry.github.io)](https://jaxry.github.io/panorama-to-cubemap/)

制作的效果如下：![](https://img2024.cnblogs.com/blog/3406761/202404/3406761-20240411225725252-960031312.png)

读取顺序也如图所示。

## 实现放置和破坏方块

**实现放置**

首先添加要要处理的物体对象和数量：
```c++
GameObject m_Soil[10];                                          // 泥土
size_t m_SoilNum;                                  		        // 泥土数量
```

随后初始化物体数量，和设置要放置物体的下标：
```C++
m_SoilNum = 10;	//设置泥土数量，在GameApp::InitResource()中
static size_t create = 0;	//在GameApp::UpdateScene()中
```

放置物体的主要实现：
```c++
if (ImGui::IsMouseClicked(ImGuiMouseButton_Right) && ImGui::IsKeyDown(ImGuiKey_LeftShift) && create < m_SoilNum) {
    //泥土方块
    Model* pModel = m_ModelManager.CreateFromGeometry("Soil", Geometry::CreateBox(2.0f, 2.0f, 2.0f));
    pModel->SetDebugObjectName("Soil");
    m_TextureManager.CreateFromFile("..\\Texture\\Soil.jpg");
    pModel->materials[0].Set<std::string>("$Diffuse", "..\\Texture\\Soil.jpg");
    pModel->materials[0].Set<XMFLOAT4>("$AmbientColor", XMFLOAT4(0.1f, 0.1f, 0.1f, 1.0f));
    pModel->materials[0].Set<XMFLOAT4>("$DiffuseColor", XMFLOAT4(0.2f, 0.2f, 0.2f, 1.0f));
    pModel->materials[0].Set<XMFLOAT4>("$SpecularColor", XMFLOAT4(0.1f, 0.1f, 0.1f, 1.0f));
    pModel->materials[0].Set<float>("$SpecularPower", 16.0f);
    pModel->materials[0].Set<XMFLOAT4>("$ReflectColor", XMFLOAT4());
    m_Soil[create].SetModel(pModel);
    DirectX::XMFLOAT3 position = m_pCamera->GetPosition();
    DirectX::XMFLOAT3 lookAxis = m_pCamera->GetLookAxis();
    m_Soil[create].GetTransform().SetPosition(position.x + lookAxis.x * 5, position.y + lookAxis.y * 5, position.z + lookAxis.z * 5);
    ++create;
}
```

这里判断在按下**鼠标右键**和**键盘左shift键**时放置物体，同时确保了放置的物体不会超过物体上限：

```c++
if (ImGui::IsMouseClicked(ImGuiMouseButton_Right) && ImGui::IsKeyDown(ImGuiKey_LeftShift) && create < m_SoilNum)
```

之后便是创建物体模型和读取并设置纹理。

为了将物体放置在摄像机前，这里还获取了**相机的位置和朝向**，并将物体移动到摄像机朝向前方：
```c++
DirectX::XMFLOAT3 position = m_pCamera->GetPosition();
DirectX::XMFLOAT3 lookAxis = m_pCamera->GetLookAxis();
m_Soil[create].GetTransform().SetPosition(position.x + lookAxis.x * 5, position.y + lookAxis.y * 5, position.z + lookAxis.z * 5);
```

然后便是使放置物体的下标加加。

最后还要绘制物体，这里**判断模型是否为空**是为了**后面破坏物体会将模型置空**：
```c++
for (size_t i = 0; i < m_SoilNum; i++) {
    if (m_Soil[i].GetModel()) {
        m_Soil[i].Draw(m_pd3dImmediateContext.Get(), m_BasicEffect);
    }
}
```

**实现破坏**

破坏物体的主要实现：
```c++
if (ImGui::IsMouseClicked(ImGuiMouseButton_Left)) {
    for (size_t i = 0; i < m_SoilNum; i++) {
        if (ray.Hit(m_Soil[i].GetBoundingBox(), nullptr, 6.0f)) {
            m_Soil[i].SetModel(nullptr);
            create = i;
        }
    }
}
```

这里判断在按下**鼠标左键**并且**鼠标拾取到物体**时破坏物体，破坏方式为**将物体模型置空**，在绘制时就不会绘制出来，为了模拟近距离破坏，实现鼠标拾取时还**设置了射线的最长距离位6.0f**。

同时为了**破坏后还可放置**，要**将放置物体的下标设为被破坏的物体的下标**，但这样还不完善，可能会遇到先破坏下标小的物体，再破坏下标大的物体的情况，若仅这样做会一直访问并放置不到先破坏的物体，于是还要在放置物品的语句之前加上如下语句，来**判断前面是否有被破坏的物体**：
```c++
for (size_t i = 0; i < m_SoilNum && m_Soil[create].GetModel(); i++) {
    if (!m_Soil[i].GetModel()) {
        create = i;
        break;
    }
}
```

如此便实现了物体的放置和破坏。