# 在DX12中使用DDSLoader和StbImage加载纹理

​		前几天学了一下在DX12中加载并绑定纹理，以免之后忘了记录一下是怎么加载的。

## 使用DDSTextureLoader12加载DDS纹理

​		DDS是一种专门为GPU设计的纹理格式，因此在使用纹理之前建议将纹理转化为DDS格式，而且DDS支持储存mipmap，可提前为纹理生成mipmap并储存在DDS中。加载DDS纹理使用了官方提供的轻量级加载源码，可使用**[DirectXTK12](https://github.com/microsoft/DirectXTK12)**中的加载库，但是这里的加载源码依赖大量的其他文件，无法单独作为一个头文件使用，因此建议使用**[DirectXTex](https://github.com/microsoft/DirectXTex)**中的DDSTextureLoader12，直接将其头文件即源文件加入自己项目中即可。

### LoadTextureFromFile

​	顾名思义该方法是从文件中加载纹理，若加载成功，该方法会创建一个空的默认类型的D3D资源，并得到存有纹理及其mipmap的真实数据的子资源，其定义如下：
```C++
HRESULT LoadDDSTextureFromFile(
    ID3D12Device* d3dDevice,
    const wchar_t* szFileName,
    ID3D12Resource** texture,
    std::unique_ptr<uint8_t[]>& ddsData,
    std::vector<D3D12_SUBRESOURCE_DATA>& subresources,
    size_t maxsize = 0,
    DDS_ALPHA_MODE* alphaMode = nullptr,
    bool* isCubeMap = nullptr);
```

1. d3dDevice：用于创建空的默认类型的D3D资源的D3D设备。
2. szFileName：文件的名字。
3. texture：加载成功后会在该位置创建空资源。
4. ddsData：加载后得到的DDS数据，为指向subresources中真实数据的指针。
5. subresources：存有纹理数据及内部数据信息的子资源数组。

因此加载的操作如下：
```C++
texture.m_Texture = nullptr;
texture.m_UploadHeap = nullptr;
texture.SetName(fileName);

std::unique_ptr<uint8_t[]> ddsData;
std::vector<D3D12_SUBRESOURCE_DATA> subresources;
stbi_uc* imgData = nullptr;
auto wstr = AnsiToWString(fileName);
// 若使用dds加载失败则使用stbimage
if (FAILED(LoadDDSTextureFromFile(
    device,
    wstr.c_str(),
    texture.m_Texture.GetAddressOf(),
    ddsData, subresources))){
    return false;
}
```

​		接下来便是将获得的子资源`subresources`中的纹理数据拷贝到自己的D3D资源`texture.m_Texture`中，由于该方法创建的D3D资源的堆属性的类型为`D3D12_HEAP_TYPE_DEFAULT`，因此还需要为其准备上传堆，为了知道上传堆的具体大小，还需要获取该默认资源的拷贝信息，操作如下：
```C++
// 获取拷贝信息
std::vector<D3D12_PLACED_SUBRESOURCE_FOOTPRINT> footprint(subresources.size());	// 子资源的宽高偏移等信息
std::vector<UINT> numRows(subresources.size());	// 子资源的行数
std::vector<UINT64> rowByteSize(subresources.size());	// 子资源每一行的字节大小
UINT64 uploadBufferSize;	// 整个纹理数据的大小
auto texDesc = texture.m_Texture->GetDesc();
device->GetCopyableFootprints(
    &texDesc, 0,
    subresources.size(), 0,
    footprint.data(), numRows.data(),
    rowByteSize.data(), &uploadBufferSize);
```

由于该纹理可能有mipmap，因此子资源的数量不一定为一个，需要为每一个子资源都获取拷贝信息。

​		获取所有信息后便可用这些信息创建上传堆，上传堆的宽高不一定要和D3D资源一样，可以直接使用纹理数据的总大小来创建一行上传堆：
```C++
// 创建GPU上传堆
D3D12_HEAP_PROPERTIES uploadHeapProps{};
uploadHeapProps.Type = D3D12_HEAP_TYPE_UPLOAD;

D3D12_RESOURCE_DESC uploadResDesc{};
uploadResDesc.Alignment = 0;
uploadResDesc.Dimension = D3D12_RESOURCE_DIMENSION_BUFFER;
uploadResDesc.Layout = D3D12_TEXTURE_LAYOUT_ROW_MAJOR;
uploadResDesc.Width = uploadBufferSize;	// 整个纹理数据的大小
uploadResDesc.Height = 1;
uploadResDesc.DepthOrArraySize = 1;
uploadResDesc.SampleDesc = { 1,0 };
uploadResDesc.MipLevels = 1;

ThrowIfFailed(device->CreateCommittedResource(
    &uploadHeapProps,
    D3D12_HEAP_FLAG_NONE,
    &uploadResDesc,
    D3D12_RESOURCE_STATE_GENERIC_READ,
    nullptr,
    IID_PPV_ARGS(texture.m_UploadHeap.GetAddressOf())));
```

​	接下来需要将数据从子资源中拷贝到上传堆中，每次将数据初始化到默认堆的时候都不禁感慨DX11中这个操作的简单，虽然在DX12中可以用官方的辅助函数。具体的拷贝操作如下：

```C++
BYTE* mappedData = nullptr;
ThrowIfFailed(texture.m_UploadHeap->Map(0, nullptr, reinterpret_cast<void**>(&mappedData)));
// 每一个子资源
for (std::size_t i = 0; i < subresources.size(); i++) {
    auto destData = mappedData + footprint[i].Offset;	// 每个子资源的偏移
    // 每一个深度
    for (std::size_t z = 0; z < footprint[i].Footprint.Depth; z++) {
        // 每一行
        auto sliceSize = footprint[i].Footprint.RowPitch * numRows[i] * z;
        for (std::size_t y = 0; y < numRows[i]; y++) {
            auto rowSize = y * subresources[i].RowPitch;
            memcpy(destData + sliceSize + rowSize,
                static_cast<const BYTE*>(subresources[i].pData) + sliceSize + rowSize,
                rowByteSize[i]);
        }
    }
}
texture.m_UploadHeap->Unmap(0, nullptr);
```

由于纹理可能是3D纹理，因此拷贝的是有也需要考虑深度，而且需要注意由于可能有多个子资源，因此每次都需要加上子资源的偏移量。

​		将实际资源拷贝到上传堆中后便是将上传堆的资源拷贝到默认堆中，也就是加载函数创建好的D3D资源中，由于加载的是纹理资源，因此需要使用的接口为`ID3D12GraphicsCommandList：：CopyTextureRegion`，该接口还需要一些关于纹理及其资源的信息，因此需要使用`D3D12_TEXTURE_COPY_LOCATION`为其提供拷贝信息，该结构体定义如下：
```C++
typedef struct D3D12_TEXTURE_COPY_LOCATION
{
	ID3D12Resource *pResource;		// D3D资源
	D3D12_TEXTURE_COPY_TYPE Type;	// 用于表示使用了下面联合体中的哪个
	union 
    {
    	D3D12_PLACED_SUBRESOURCE_FOOTPRINT PlacedFootprint;
    	UINT SubresourceIndex;
    } 	;
} 	D3D12_TEXTURE_COPY_LOCATION;
```

具体拷贝操作如下：

```C++
// 拷贝所有子资源
for (std::size_t i = 0; i < subresources.size(); i++) {
    D3D12_TEXTURE_COPY_LOCATION dest{};
    dest.Type = D3D12_TEXTURE_COPY_TYPE_SUBRESOURCE_INDEX;
    dest.SubresourceIndex = i;
    dest.pResource = texture.m_Texture.Get();
    D3D12_TEXTURE_COPY_LOCATION src{};
    src.Type = D3D12_TEXTURE_COPY_TYPE_PLACED_FOOTPRINT;
    src.PlacedFootprint = footprint[i];
    src.pResource = texture.m_UploadHeap.Get();
    cmdList->CopyTextureRegion(&dest,0,0,0,&src,nullptr);
}
```

此时纹理的创建便完成了，最后将纹理资源的状态转换一下即可：
```c++
D3D12_RESOURCE_BARRIER barrier{};
barrier.Type = D3D12_RESOURCE_BARRIER_TYPE_TRANSITION;
barrier.Transition = {
    texture.m_Texture.Get(),
    D3D12_RESOURCE_BARRIER_ALL_SUBRESOURCES,
    D3D12_RESOURCE_STATE_COPY_DEST,
    D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE
};
cmdList->ResourceBarrier(1, &barrier);
```



### LoadTextureFromMemory

​		从给定的数据中加载纹理，暂时还没有使用需求，因此也没测试我的实现是否可行，使用及处理成D3D资源的方法与上面的函数一样，其定义如下：
```C++
HRESULT __cdecl LoadDDSTextureFromMemory(
    _In_ ID3D12Device* d3dDevice,
    _In_reads_bytes_(ddsDataSize) const uint8_t* ddsData,
    size_t ddsDataSize,
    _Outptr_ ID3D12Resource** texture,
    std::vector<D3D12_SUBRESOURCE_DATA>& subresources,
    size_t maxsize = 0,
    _Out_opt_ DDS_ALPHA_MODE* alphaMode = nullptr,
    _Out_opt_ bool* isCubeMap = nullptr);
```



## 使用StbImage加载纹理

​		需要将[此处](https://github.com/nothings/stb/blob/master/stb_image.h)的头文件包含到自己的项目中，随后调用函数`stbi_load`即可加载文件中的纹理，加载完数据后还需要自行创建子资源，由于该方法加载的图片格式一般不支持mipmap，因此子资源也只有一个：
```C++
int height, width, comp;
stbi_uc* imgData = stbi_load(fileName.c_str(), &width, &height, &comp, STBI_rgb_alpha);
if (imgData == nullptr)return false;

std::vector<D3D12_SUBRESOURCE_DATA> subresources;
subresources.emplace_back(imgData,width * sizeof(uint32_t),0);
```

其中`stbi_uc*`其实是`char*`，指向纹理资源的首地址。由于其不会帮忙创建默认堆，需要自行创建，因此需要加上如下步骤来创建纹理资源：
```C++
D3D12_HEAP_PROPERTIES texHeapDesc{};
texHeapDesc.Type = D3D12_HEAP_TYPE_DEFAULT;
D3D12_RESOURCE_DESC texResDesc{};
texResDesc.Alignment = 0;
texResDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
texResDesc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
texResDesc.Width = width;
texResDesc.Height = height;
texResDesc.DepthOrArraySize = 1;
texResDesc.SampleDesc = { 1,0 };
texResDesc.MipLevels = 1;
ThrowIfFailed(device->CreateCommittedResource(
    &texHeapDesc,
    D3D12_HEAP_FLAG_NONE,
    &texResDesc,
    D3D12_RESOURCE_STATE_COPY_DEST,
    nullptr,
    IID_PPV_ARGS(texture.m_Texture.GetAddressOf())));
```

后续的步骤与加载DDS纹理相同，将数据从`imgData`中拷贝到上传堆中再从上传堆中拷贝到默认堆中即可。

若是想从内存数据中读取纹理，将第一步替换成如下步骤即可：

```C++
int height, width, comp;
stbi_uc* imgData = stbi_load_from_memory(reinterpret_cast<stbi_uc*>(data), (int)dataSize, &width, &height, &comp, STBI_rgb_alpha);
```

最后还需要自行将加载纹理用到的资源释放：
```c++
stbi_image_free(imgData);
```



## 源码

```c++
bool Texture::LoadTextureFromFile(
    Texture& texture,
    const std::string& fileName,
    ID3D12Device* device,
    ID3D12GraphicsCommandList* cmdList)
{
    texture.m_Texture = nullptr;
    texture.m_UploadHeap = nullptr;
    texture.SetName(fileName);

    std::unique_ptr<uint8_t[]> ddsData;
    std::vector<D3D12_SUBRESOURCE_DATA> subresources;
    stbi_uc* imgData = nullptr;
    auto wstr = AnsiToWString(fileName);
    // 若使用dds加载失败则使用stbimage
    if (FAILED(LoadDDSTextureFromFile(
        device,
        wstr.c_str(),
        texture.m_Texture.GetAddressOf(),
        ddsData, subresources))) {
        int height, width, comp;
        imgData = stbi_load(fileName.c_str(), &width, &height, &comp, STBI_rgb_alpha);
        if (imgData == nullptr)return false;

        subresources.clear();
        subresources.emplace_back(imgData,width * sizeof(uint32_t),0);

        D3D12_HEAP_PROPERTIES texHeapDesc{};
        texHeapDesc.Type = D3D12_HEAP_TYPE_DEFAULT;
        D3D12_RESOURCE_DESC texResDesc{};
        texResDesc.Alignment = 0;
        texResDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
        texResDesc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
        texResDesc.Width = width;
        texResDesc.Height = height;
        texResDesc.DepthOrArraySize = 1;
        texResDesc.SampleDesc = { 1,0 };
        texResDesc.MipLevels = 1;
        ThrowIfFailed(device->CreateCommittedResource(
            &texHeapDesc,
            D3D12_HEAP_FLAG_NONE,
            &texResDesc,
            D3D12_RESOURCE_STATE_COPY_DEST,
            nullptr,
            IID_PPV_ARGS(texture.m_Texture.GetAddressOf())));
    }

    LoadTexture(texture, subresources, device, cmdList);

    if (imgData != nullptr) {
        stbi_image_free(imgData);
    }

    return true;
}

void Texture::LoadTexture(
    Texture& texture,
    const std::vector<D3D12_SUBRESOURCE_DATA>& subresources,
    ID3D12Device* device,
    ID3D12GraphicsCommandList* cmdList)
{
    // 获取拷贝信息
    std::vector<D3D12_PLACED_SUBRESOURCE_FOOTPRINT> footprint(subresources.size());	// 子资源的宽高偏移等信息
    std::vector<UINT> numRows(subresources.size());	// 子资源的行数
    std::vector<UINT64> rowByteSize(subresources.size());	// 子资源每一行的字节大小
    UINT64 uploadBufferSize;	// 整个纹理数据的大小
    auto texDesc = texture.m_Texture->GetDesc();
    device->GetCopyableFootprints(
        &texDesc, 0,
        subresources.size(), 0,
        footprint.data(), numRows.data(),
        rowByteSize.data(), &uploadBufferSize);

    // 创建GPU上传堆
    D3D12_HEAP_PROPERTIES uploadHeapProps{};
    uploadHeapProps.Type = D3D12_HEAP_TYPE_UPLOAD;

    D3D12_RESOURCE_DESC uploadResDesc{};
    uploadResDesc.Alignment = 0;
    uploadResDesc.Dimension = D3D12_RESOURCE_DIMENSION_BUFFER;
    uploadResDesc.Layout = D3D12_TEXTURE_LAYOUT_ROW_MAJOR;
    uploadResDesc.Width = uploadBufferSize;
    uploadResDesc.Height = 1;
    uploadResDesc.DepthOrArraySize = 1;
    uploadResDesc.SampleDesc = { 1,0 };
    uploadResDesc.MipLevels = 1;

    ThrowIfFailed(device->CreateCommittedResource(
        &uploadHeapProps,
        D3D12_HEAP_FLAG_NONE,
        &uploadResDesc,
        D3D12_RESOURCE_STATE_GENERIC_READ,
        nullptr,
        IID_PPV_ARGS(texture.m_UploadHeap.GetAddressOf())));


    BYTE* mappedData = nullptr;
    ThrowIfFailed(texture.m_UploadHeap->Map(0, nullptr, reinterpret_cast<void**>(&mappedData)));
    // 每一个子资源
    for (std::size_t i = 0; i < subresources.size(); i++) {
        auto destData = mappedData + footprint[i].Offset;	// 每个子资源的偏移
        // 每一个深度
        for (std::size_t z = 0; z < footprint[i].Footprint.Depth; z++) {
            // 每一行
            auto sliceSize = footprint[i].Footprint.RowPitch * numRows[i] * z;
            for (std::size_t y = 0; y < numRows[i]; y++) {
                auto rowSize = y * subresources[i].RowPitch;
                memcpy(destData + sliceSize + rowSize,
                    static_cast<const BYTE*>(subresources[i].pData) + sliceSize + rowSize,
                    rowByteSize[i]);
            }
        }
    }
    texture.m_UploadHeap->Unmap(0, nullptr);

    // 拷贝所有子资源
    for (std::size_t i = 0; i < subresources.size(); i++) {
        D3D12_TEXTURE_COPY_LOCATION dest{};
        dest.Type = D3D12_TEXTURE_COPY_TYPE_SUBRESOURCE_INDEX;
        dest.SubresourceIndex = i;
        dest.pResource = texture.m_Texture.Get();
        D3D12_TEXTURE_COPY_LOCATION src{};
        src.Type = D3D12_TEXTURE_COPY_TYPE_PLACED_FOOTPRINT;
        src.PlacedFootprint = footprint[i];
        src.pResource = texture.m_UploadHeap.Get();
        cmdList->CopyTextureRegion(&dest,0,0,0,&src,nullptr);
    }

    D3D12_RESOURCE_BARRIER barrier{};
    barrier.Type = D3D12_RESOURCE_BARRIER_TYPE_TRANSITION;
    barrier.Transition = {
        texture.m_Texture.Get(),
        D3D12_RESOURCE_BARRIER_ALL_SUBRESOURCES,
        D3D12_RESOURCE_STATE_COPY_DEST,
        D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE
    };
    cmdList->ResourceBarrier(1, &barrier);
}
```

