# Basic Tasks

[TOC]



## 一：视频显示

### 1.1 Renderer介绍

DirectShow使用的renderer有以下几种：

- **Video Renderer filter：** 支持所有DirectX的平台，没有系统要求，为默认renderer；
- **Video Mixing Renderer Filter 7(VMR-7)：** WindowsXP以后版本支持，加入新特性；
- **Video Mixing Renderer Filter 9(VMR-9)：** 比VMR7更先进，但对系统要求更高；
- **Enhanced Video Renderer (EVR)：** Windows Vista之后版本支持。

总的来说：**EVR** 针对Windows Vista及之后版本，VMR-9针对Windows Vista之前版本。

窗口模式：

- **Windowed Mode：** 窗口模式，renderer带有自己的显示窗口，可以根据需要进行窗口嵌套；
- **Windowless Mode：** 无窗口模式，renderer没有自带窗口，可以直接在其他窗口显示。

**Video Renderer filter**只支持窗口模式； **VMR-7** 和 **VMR-9** 都支持；**EVR** 只支持无窗口模式。

### 1.2 Renderer选择

Renderer的选择主要根据Window版本确定，可根据需要选择：

| Filter                                                       | Remarks                                           |
| ------------------------------------------------------------ | ------------------------------------------------- |
| [**Enhanced Video Renderer**](https://docs.microsoft.com/zh-cn/windows/desktop/DirectShow/enhanced-video-renderer-filter) (EVR) | Uses Direct3D 9. Requires Windows Vista or later. |
| [Video Mixing Renderer 9](https://docs.microsoft.com/zh-cn/windows/desktop/DirectShow/video-mixing-renderer-filter-9) (VMR-9) | Uses Direct3D 9. Requires Windows XP or later.    |
| [Video Mixing Filter 7](https://docs.microsoft.com/zh-cn/windows/desktop/DirectShow/video-mixing-renderer-filter-7) (VMR-7) | Uses DirectDraw. Requires Windows XP or later.    |
| [Overlay Mixer](https://docs.microsoft.com/zh-cn/windows/desktop/DirectShow/using-the-overlay-mixer-in-video-capture) | Supports hardware overlays through DirectDraw.    |
| Legacy [Video Renderer](https://docs.microsoft.com/zh-cn/windows/desktop/DirectShow/video-renderer-filter) filter. | Uses DirectDraw or (rarely) GDI                   |

### 1.3 Windowed Mode

窗口模式拥有自己的窗口，可以进行窗口嵌套，常规步骤：

1. 查询`IVideoWindow`接口；
2. 设置父窗口；
3. 设置新的窗口风格；
4. 为窗口在父窗口设置相应的位置；
5. 通知窗口相应的`WM_MOVE` 事件。

```c++
// Query IVideoWindow
IVideoWindow *pVidWin = NULL;
pGraph->QueryInterface(IID_IVideoWindow, (void **)&pVidWin);

// Set parent
pVidWin->put_Owner((OAHWND)hwnd);

// Set window style
pVidWin->put_WindowStyle(WS_CHILD | WS_CLIPSIBLINGS);

// Set position
RECT rc;
GetClientRect(hwnd, &rc);
pVidWin->SetWindowPosition(0, 0, rc.right, rc.bottom);

// Deal event
// (Inside your WindowProc)
case WM_MOVE:
    pVidWin->NotifyOwnerMessage((OAHWND)hWnd, msg, wParam, lParam);
    break;

// Clear up
pControl->Stop();
pVidWin->put_Visible(OAFALSE);
pVidWin->put_Owner(NULL);
```

### 1.4 Windowless Mode

常规步骤如下：

1. 配置VMR为Windowless模式；

   VMR-7：

   - 创建Filter Graph Manager；
   - 创建VMR-7 并添加到filter graph；
   - 通过IVMRFilterConfig::SetRenderingMode设置VMRMode_Windowless标志；
   - 查询VMR-7的 IVMRWindowlessControl接口；
   - 调用IVMRWindowlessControl::SetVideoClippingWindow设置窗口。

   VMR-9：

   - 创建Filter Graph Manager；
   - 创建VMR-9 并添加到filter graph；
   - 通过 IVMRFilterConfig9::SetRenderingMode设置VMR9Mode_Windowless标志；
   - 查询VMR-9的IVMRWindowlessControl9接口；
   - 调用IVMRWindowlessControl9::SetVideoClippingWindow设置窗口。

2. 处理filter graph的剩余部分；

3. 指定视频位置：

   VMR7：

   - IVMRWindowlessControl::SetVideoPosition指定源矩形和目标矩形区域；
   - 源矩形必须小于等于视频大小，视频大小可以通过IVMRWindowlessControl::GetNativeVideoSize查询。

   VMR9：

   - IVMRWindowlessControl9::SetVideoPosition指定源矩形和目标矩形区域；
   - 源矩形必须小于等于视频大小，视频大小可以通过IVMRWindowlessControl9::GetNativeVideoSize。

4. 处理窗口消息：

   VMR-7：

   - **WM_PAINT:** 调用 `IVMRWindowlessControl::RepaintVideo`绘制最新视频帧；
   - **WM_DISPLAYCHANGE:**  调用IVMRWindowlessControl::DisplayModeChanged重置视频分辨率或色彩格式；
   - **WM_SIZE** or **WM_WINDOWPOSCHANGED:**  重新计算位置，调用 IVMRWindowlessControl::SetVideoPosition 更新。

   VMR-9:

   - **WM_PAINT：** 调用 `IVMRWindowlessControl9::RepaintVideo` 绘制最新视频帧；
   - **WM_DISPLAYCHANGE: ** 调用 `IVMRWindowlessControl9::DisplayModeChanged`重制视频分辨率或色彩格式；
   - **WM_SIZE** or **WM_WINDOWPOSCHANGED: **重新计算位置，调用`IVMRWindowlessControl9::SetVideoPosition`  更新。





## 二：处理消息

DirectShow的消息处理流程：

```c++
// 自定义消息类型
#define WM_GRAPHNOTIFY  WM_APP + 1

// 注册窗口消息
IMediaEventEx *g_pEvent = NULL;
g_pGraph->QueryInterface(IID_IMediaEventEx, (void **)&g_pEvent);
g_pEvent->SetNotifyWindow((OAHWND)g_hwnd, WM_GRAPHNOTIFY, 0);

// 在窗口的消息循环中处理注册的消息
case WM_GRAPHNOTIFY:
    HandleGraphEvent();
    break;

// 处理DirectShow消息
void HandleGraphEvent()
{
    // Disregard if we don't have an IMediaEventEx pointer.
    if (g_pEvent == NULL)
    {
        return;
    }
    // Get all the events
    long evCode;
    LONG_PTR param1, param2;
    HRESULT hr;
    while (SUCCEEDED(g_pEvent->GetEvent(&evCode, &param1, &param2, 0)))
    {
        g_pEvent->FreeEventParams(evCode, param1, param2);
        switch (evCode)
        {
        case EC_COMPLETE:  // Fall through.
        case EC_USERABORT: // Fall through.
        case EC_ERRORABORT:
            CleanUp();
            PostQuitMessage(0);
            return;
        }
    } 
}
```





## 三：设备枚举

### 3.1 系统枚举

```c++
// 创建设备枚举句柄
HRESULT hr;
ICreateDevEnum *pSysDevEnum = NULL;
hr = CoCreateInstance(CLSID_SystemDeviceEnum, NULL, CLSCTX_INPROC_SERVER,
    IID_ICreateDevEnum, (void **)&pSysDevEnum);
if (FAILED(hr)) {
    return hr;
}

// 获得视频枚举器
IEnumMoniker *pEnumCat = NULL;
hr = pSysDevEnum->CreateClassEnumerator(CLSID_VideoCompressorCategory, &pEnumCat, 0);
if (hr == S_OK) {
    // 循环枚举
    IMoniker *pMoniker = NULL;
    ULONG cFetched;
    while(pEnumCat->Next(1, &pMoniker, &cFetched) == S_OK)
    {
        IPropertyBag *pPropBag;
        hr = pMoniker->BindToStorage(0, 0, IID_IPropertyBag, 
            (void **)&pPropBag);
        if (SUCCEEDED(hr))
        {
            // To retrieve the filter's friendly name, do the following:
            VARIANT varName;
            VariantInit(&varName);
            hr = pPropBag->Read(L"FriendlyName", &varName, 0);
            if (SUCCEEDED(hr))
            {
                // Display the name in your UI somehow.
            }
            VariantClear(&varName);

            // To create an instance of the filter, do the following:
            IBaseFilter *pFilter;
            hr = pMoniker->BindToObject(NULL, NULL, IID_IBaseFilter,
                (void**)&pFilter);
            // Now add the filter to the graph. 
            //Remember to release pFilter later.
            pPropBag->Release();
        }
        pMoniker->Release();
    }
    pEnumCat->Release();
}
pSysDevEnum->Release();
```

### 3.2 Filter Mapper枚举

略





## 四：Filter Graph内部枚举

Filter Graph -> Filters -> Pins -> Media Types

### 4.1 枚举Filters

在Filter Graph中枚举存在的Filters：

```c++
HRESULT EnumFilters (IFilterGraph *pGraph) 
{
    IEnumFilters *pEnum = NULL;
    IBaseFilter *pFilter;
    ULONG cFetched;

    // 查询所有Filters
    HRESULT hr = pGraph->EnumFilters(&pEnum);
    if (FAILED(hr)) return hr;

    // 遍历枚举到的Filter信息
    while(pEnum->Next(1, &pFilter, &cFetched) == S_OK)
    {
        FILTER_INFO FilterInfo;
        hr = pFilter->QueryFilterInfo(&FilterInfo);
        if (FAILED(hr))
        {
            MessageBox(NULL, TEXT("Could not get the filter info"),
                TEXT("Error"), MB_OK | MB_ICONERROR);
            continue;  // Maybe the next one will work.
        }

#ifdef UNICODE
        MessageBox(NULL, FilterInfo.achName, TEXT("Filter Name"), MB_OK);
#else
        char szName[MAX_FILTER_NAME];
        int cch = WideCharToMultiByte(CP_ACP, 0, FilterInfo.achName,
            MAX_FILTER_NAME, szName, MAX_FILTER_NAME, 0, 0);
        if (cch > 0)
            MessageBox(NULL, szName, TEXT("Filter Name"), MB_OK);
#endif

        // The FILTER_INFO structure holds a pointer to the Filter Graph
        // Manager, with a reference count that must be released.
        if (FilterInfo.pGraph != NULL)
        {
            FilterInfo.pGraph->Release();
        }
        pFilter->Release();
    }

    pEnum->Release();
    return S_OK;
}
```

### 4.2 枚举Pin

枚举单个Filter包含的Pins：

```c++
HRESULT GetPin(IBaseFilter *pFilter, PIN_DIRECTION PinDir, IPin **ppPin)
{
    IEnumPins  *pEnum = NULL;
    IPin       *pPin = NULL;
    HRESULT    hr;

    if (ppPin == NULL)
    {
        return E_POINTER;
    }

    // 查询所有的Pins
    hr = pFilter->EnumPins(&pEnum);
    if (FAILED(hr))
    {
        return hr;
    }
    
    // 遍历Pin信息(方向等)
    while(pEnum->Next(1, &pPin, 0) == S_OK)
    {
        PIN_DIRECTION PinDirThis;
        hr = pPin->QueryDirection(&PinDirThis);
        if (FAILED(hr))
        {
            pPin->Release();
            pEnum->Release();
            return hr;
        }
        if (PinDir == PinDirThis)
        {
            // Found a match. Return the IPin pointer to the caller.
            *ppPin = pPin;
            pEnum->Release();
            return S_OK;
        }
        // Release the pin for the next time through the loop.
        pPin->Release();
    }
    // No more pins. We did not find a match.
    pEnum->Release();
    return E_FAIL;  
}
```

### 4.3 枚举Media Types

枚举Pin包含的Media Types：

```c++
// Given a pin, find a preferred media type 
//
// pPin         Pointer to the pin.
// majorType    Preferred major type (GUID_NULL = don't care).
// subType      Preferred subtype (GUID_NULL = don't care).
// formatType   Preferred format type (GUID_NULL = don't care).
// ppmt         Receives a pointer to the media type. Can be NULL.
//
// Note: If you want to check whether a pin supports a desired media type,
//       but do not need the format details, set ppmt to NULL.
//
//       If ppmt is not NULL and the method succeeds, the caller must
//       delete the media type, including the format block. 

HRESULT GetPinMediaType(
    IPin *pPin,             // pointer to the pin
    REFGUID majorType,      // desired major type, or GUID_NULL = don't care
    REFGUID subType,        // desired subtype, or GUID_NULL = don't care
    REFGUID formatType,     // desired format type, of GUID_NULL = don't care
    AM_MEDIA_TYPE **ppmt    // Receives a pointer to the media type. (Can be NULL)
    )
{
    *ppmt = NULL;

    IEnumMediaTypes *pEnum = NULL;
    AM_MEDIA_TYPE *pmt = NULL;
    BOOL bFound = FALSE;
    
    HRESULT hr = pPin->EnumMediaTypes(&pEnum);
    if (FAILED(hr))
    {
        return hr;
    }

    while (hr = pEnum->Next(1, &pmt, NULL), hr == S_OK)
    {
        if ((majorType == GUID_NULL) || (majorType == pmt->majortype))
        {
            if ((subType == GUID_NULL) || (subType == pmt->subtype))
            {
                if ((formatType == GUID_NULL) || 
                    (formatType == pmt->formattype))
                {
                    // Found a match. 
                    if (ppmt)
                    {
                        *ppmt = pmt;  // Return it to the caller
                    }
                    else
                    {
                        _DeleteMediaType(pmt);
                    }
                    bFound = TRUE;
                    break;
                }
            }
        }
        _DeleteMediaType(pmt);
    }

    SafeRelease(&pEnum);
    if (SUCCEEDED(hr))
    {
        if (!bFound)
        {
            hr = VFW_E_NOT_FOUND;
        }
    }
    return hr;
}
```





## 五：Graph建立

### 5.1 Filter通过CLSID加入Graph

```c++
HRESULT AddFilterByCLSID(
    IGraphBuilder *pGraph,      // Pointer to the Filter Graph Manager.
    REFGUID clsid,              // CLSID of the filter to create.
    IBaseFilter **ppF,          // Receives a pointer to the filter.
    LPCWSTR wszName             // A name for the filter (can be NULL).
    )
{
    *ppF = 0;

    IBaseFilter *pFilter = NULL;
    
    HRESULT hr = CoCreateInstance(clsid, NULL, CLSCTX_INPROC_SERVER, 
        IID_PPV_ARGS(&pFilter));
    if (FAILED(hr))
    {
        goto done;
    }

    hr = pGraph->AddFilter(pFilter, wszName);
    if (FAILED(hr))
    {
        goto done;
    }

    *ppF = pFilter;
    (*ppF)->AddRef();

done:
    SafeRelease(&pFilter);
    return hr;
}
```

上述步骤为：根据CLSID创建Filter -> 将Filter加入Graph。

有时候也可以使用更简单的方法（但不是一直有效）：

```c++
IBaseFilter *pMux;
hr = AddFilterByCLSID(pGraph, CLSID_AviDest, L"AVI Mux", &pMux, NULL); 
if (SUCCEEDED(hr))
{
    /* ... */
   pMux->Release();
}
```

### 5.2 查找Pin

Pin有连接状态和方向两种属性。通过以下接口可以查询：

查询Pin是否连接：

```c++
HRESULT IsPinConnected(IPin *pPin, BOOL *pResult)
{
    IPin *pTmp = NULL;
    HRESULT hr = pPin->ConnectedTo(&pTmp);
    if (SUCCEEDED(hr))
    {
        *pResult = TRUE;
    }
    else if (hr == VFW_E_NOT_CONNECTED)
    {
        // The pin is not connected. This is not an error for our purposes.
        *pResult = FALSE;
        hr = S_OK;
    }

    SafeRelease(&pTmp);
    return hr;
}
```

查询Pin方向：

```c++
// Query whether a pin has a specified direction (input / output)
HRESULT IsPinDirection(IPin *pPin, PIN_DIRECTION dir, BOOL *pResult)
{
    PIN_DIRECTION pinDir;
    HRESULT hr = pPin->QueryDirection(&pinDir);
    if (SUCCEEDED(hr))
    {
        *pResult = (pinDir == dir);
    }
    return hr;
}
```

按照连接状态和方向查询：

```c++
// Match a pin by pin direction and connection state.
HRESULT MatchPin(IPin *pPin, PIN_DIRECTION direction, BOOL bShouldBeConnected, BOOL *pResult)
{
    assert(pResult != NULL);

    BOOL bMatch = FALSE;
    BOOL bIsConnected = FALSE;

    HRESULT hr = IsPinConnected(pPin, &bIsConnected);
    if (SUCCEEDED(hr))
    {
        if (bIsConnected == bShouldBeConnected)
        {
            hr = IsPinDirection(pPin, direction, &bMatch);
        }
    }

    if (SUCCEEDED(hr))
    {
        *pResult = bMatch;
    }
    return hr;
}
```

按要求查找Filter中的符合条件的Pin：

```c++
// Return the first unconnected input pin or output pin.
HRESULT FindUnconnectedPin(IBaseFilter *pFilter, PIN_DIRECTION PinDir, IPin **ppPin)
{
    IEnumPins *pEnum = NULL;
    IPin *pPin = NULL;
    BOOL bFound = FALSE;

    HRESULT hr = pFilter->EnumPins(&pEnum);
    if (FAILED(hr))
    {
        goto done;
    }

    while (S_OK == pEnum->Next(1, &pPin, NULL))
    {
        hr = MatchPin(pPin, PinDir, FALSE, &bFound);
        if (FAILED(hr))
        {
            goto done;
        }
        if (bFound)
        {
            *ppPin = pPin;
            (*ppPin)->AddRef();
            break;
        }
        SafeRelease(&pPin);
    }

    if (!bFound)
    {
        hr = VFW_E_NOT_FOUND;
    }

done:
    SafeRelease(&pPin);
    SafeRelease(&pEnum);
    return hr;
}
```

### 5.3 Filters连接

主要有以下几种连接方式：

#### 5.3.1 Output Pin -> Filter

即已知上级的输出Pin和下级Filter：

```c++
// Connect output pin to filter.

HRESULT ConnectFilters(
    IGraphBuilder *pGraph, // Filter Graph Manager.
    IPin *pOut,            // Output pin on the upstream filter.
    IBaseFilter *pDest)    // Downstream filter.
{
    IPin *pIn = NULL;
        
    // Find an input pin on the downstream filter.
    HRESULT hr = FindUnconnectedPin(pDest, PINDIR_INPUT, &pIn);
    if (SUCCEEDED(hr))
    {
        // Try to connect them.
        hr = pGraph->Connect(pOut, pIn);
        pIn->Release();
    }
    return hr;
}
```

#### 5.3.2 Filter -> Input Pin

即已知上级Filter和下级的输入Pin：

```c++
// Connect filter to input pin.

HRESULT ConnectFilters(IGraphBuilder *pGraph, IBaseFilter *pSrc, IPin *pIn)
{
    IPin *pOut = NULL;
        
    // Find an output pin on the upstream filter.
    HRESULT hr = FindUnconnectedPin(pSrc, PINDIR_OUTPUT, &pOut);
    if (SUCCEEDED(hr))
    {
        // Try to connect them.
        hr = pGraph->Connect(pOut, pIn);
        pOut->Release();
    }
    return hr;
}
```

#### 5.3.3 Filter -> Filter

即已知上级Filter和下级Filter：

```c++
// Connect filter to filter

HRESULT ConnectFilters(IGraphBuilder *pGraph, IBaseFilter *pSrc, IBaseFilter *pDest)
{
    IPin *pOut = NULL;

    // Find an output pin on the first filter.
    HRESULT hr = FindUnconnectedPin(pSrc, PINDIR_OUTPUT, &pOut);
    if (SUCCEEDED(hr))
    {
        hr = ConnectFilters(pGraph, pOut, pDest);
        pOut->Release();
    }
    return hr;
}
```

### 5.4 查找Filter和Pin的Interface

Filter和Pin都属于COM组件，都有相应的Interface用于控制。通过以下方法可以查找需要的Interface：

#### 5.4.1 查找Filter的Interface

````c++
HRESULT FindFilterInterface(
    IGraphBuilder *pGraph, // Pointer to the Filter Graph Manager.
    REFGUID iid,           // IID of the interface to retrieve.
    void **ppUnk)          // Receives the interface pointer.
{
    if (!pGraph || !ppUnk) return E_POINTER;

    HRESULT hr = E_FAIL;
    IEnumFilters *pEnum = NULL;
    IBaseFilter *pF = NULL;
    if (FAILED(pGraph->EnumFilters(&pEnum)))
    {
        return E_FAIL;
    }
    // Query every filter for the interface.
    while (S_OK == pEnum->Next(1, &pF, 0))
    {
        hr = pF->QueryInterface(iid, ppUnk);
        pF->Release();
        if (SUCCEEDED(hr))
        {
            break;
        }
    }
    pEnum->Release();
    return hr;
}
````

#### 5.4.2 查找Pin的Interface

```C++
HRESULT FindPinInterface(
    IBaseFilter *pFilter,  // Pointer to the filter to search.
    REFGUID iid,           // IID of the interface.
    void **ppUnk)          // Receives the interface pointer.
{
    if (!pFilter || !ppUnk) return E_POINTER;

    HRESULT hr = E_FAIL;
    IEnumPins *pEnum = 0;
    if (FAILED(pFilter->EnumPins(&pEnum)))
    {
        return E_FAIL;
    }
    // Query every pin for the interface.
    IPin *pPin = 0;
    while (S_OK == pEnum->Next(1, &pPin, 0))
    {
        hr = pPin->QueryInterface(iid, ppUnk);
        pPin->Release();
        if (SUCCEEDED(hr))
        {
            break;
        }
    }
    pEnum->Release();
    return hr;
}
```

#### 5.4.3 查找任意的Interface

````c++
HRESULT FindInterfaceAnywhere(
    IGraphBuilder *pGraph, 
    REFGUID iid, 
    void **ppUnk)
{
    if (!pGraph || !ppUnk) return E_POINTER;
    HRESULT hr = E_FAIL;
    IEnumFilters *pEnum = 0;
    if (FAILED(pGraph->EnumFilters(&pEnum)))
    {
        return E_FAIL;
    }
    // Loop through every filter in the graph.
    IBaseFilter *pF = 0;
    while (S_OK == pEnum->Next(1, &pF, 0))
    {
        hr = pF->QueryInterface(iid, ppUnk);
        if (FAILED(hr))
        {
            // The filter does not expose the interface, but maybe
            // one of its pins does.
            hr = FindPinInterface(pF, iid, ppUnk);
        }
        pF->Release();
        if (SUCCEEDED(hr))
        {
            break;
        }
    }
    pEnum->Release();
    return hr;
}
````

### 5.5 查询Filter的相邻Filter

查找与Filter相连的其他Filter，基本就是遍历整个Graph。详情参考：

[Find a Filters Peer](https://docs.microsoft.com/zh-cn/windows/desktop/DirectShow/find-a-filters-peer)

### 5.6 删除Graph的所有Filter

即遍历Graph和安全释放Filter：

```c++
// Stop the graph.
pControl->Stop();

// Enumerate the filters in the graph.
IEnumFilters *pEnum = NULL;
HRESULT hr = pGraph->EnumFilters(&pEnum);
if (SUCCEEDED(hr))
{
    IBaseFilter *pFilter = NULL;
    while (S_OK == pEnum->Next(1, &pFilter, NULL))
     {
         // Remove the filter.
         pGraph->RemoveFilter(pFilter);
         // Reset the enumerator.
         pEnum->Reset();
         pFilter->Release();
    }
    pEnum->Release();
}
```

### 5.7 使用Capture Graph Builder辅助构建Graph

由于Graph的结构过于复杂，DirectShow可以使用Capture Graph Builder来辅助构建一些特殊的Graph，例如采集系统等。可以认为Capture Graph Builder构建的是特殊的Filter Graph。具体操作：

1. 构建Capture Graph Builder，并与Filter Graph关联；
2. 连接Filters；
3. 查找Filters和Pins的Interface；
4. 查找Pins。

#### 5.7.1 创建并关联Filter Graph

```c++
IGraphBuilder *pGraph = NULL;
ICaptureGraphBuilder2 *pBuilder = NULL;

// Create the Filter Graph Manager.
HRESULT hr =  CoCreateInstance(CLSID_FilterGraph, NULL,
    CLSCTX_INPROC_SERVER, IID_IGraphBuilder, (void **)&pGraph);

if (SUCCEEDED(hr))
{
    // Create the Capture Graph Builder.
    hr = CoCreateInstance(CLSID_CaptureGraphBuilder2, NULL,
        CLSCTX_INPROC_SERVER, IID_ICaptureGraphBuilder2, 
        (void **)&pBuilder);
    if (SUCCEEDED(hr))
    {
        pBuilder->SetFiltergraph(pGraph);
    }
};
```

#### 5.7.2 连接Filters

Capture Graph Builder通过 `ICaptureGraphBuilder2::RenderStream` 连接Filters。暂时先不讨论此函数的前两个参数：

```c++
// Connect A -> B -> C
RenderStream(NULL, NULL, A, B, C);

// Connect A -> C
RenderStream(NULL, NULL, A, NULL, C);

// Connect A -> B -> C -> D -> E
RenderStream(NULL, NULL, A, B, C);
RenderStream(NULL, NULL, C, D, E);
```

最后一个参数为NULL则表示自动指定一个Renderer。现在来讨论前两个参数：

- 第一个参数：只用于Capture Filter，指明输出Pin类型，可选值：`PIN_CATEGORY_CAPTURE` 和 `PIN_CATEGORY_PREVIEW`;如果Capture Filter没有这两种类型的Pin，那么RenderStream会自动提供一个 Smart Tee Filter来提供Capture Pin和Preview Pin；（Preview Pin会根据需要丢帧，优先会保证Capture Pin的帧采集）；
- 第二个参数：媒体类型，取值：`MEDIATYPE_Audio`,`MEDIATYPE_Video` 和`MEDIATYPE_Interleaved(DV)`。

#### 5.7.3 查找Filters和Pins的Interface

```c++
// eg. to find IAMStreamConfig interface
IAMStreamConfig *pConfig = NULL;
HRESULT hr = pBuild->FindInterface(
    &PIN_CATEGORY_PREVIEW, 
    &MEDIATYPE_Video,
    pVCap, 
    IID_IAMStreamConfig, 
    (void**)&pConfig
);
if (SUCCESSFUL(hr))
{
    /* ... */
    pConfig->Release();
}
```

#### 5.7.4 查找Pins

```c++
// eg. find unconnected output pin
IPin *pPin = NULL;
hr = pBuild->FindPin(
    pCap,                   // Pointer to the filter to search.
    PINDIR_OUTPUT,          // Search for an output pin.
    &PIN_CATEGORY_PREVIEW,  // Search for a preview pin.
    &MEDIATYPE_Video,       // Search for a video pin.
    TRUE,                   // The pin must be unconnected. 
    0,                      // Return the first matching pin (index 0).
    &pPin);                 // This variable receives the IPin pointer.
if (SUCCESSFUL(hr))
{
    /* ... */
    pPin->Release();
}
```





## 六：在Filter Graph中Seek

[Seeking the Filter Graph](https://docs.microsoft.com/zh-cn/windows/desktop/DirectShow/seeking-the-filter-graph)





## 七：设置Graph时钟

[Setting the Graph Clock](https://docs.microsoft.com/zh-cn/windows/desktop/DirectShow/setting-the-graph-clock)





## 八：显示属性页

```c++
IBaseFilter *pFilter;
/* Obtain the filter's IBaseFilter interface. (Not shown) */
ISpecifyPropertyPages *pProp;
HRESULT hr = pFilter->QueryInterface(IID_ISpecifyPropertyPages, (void **)&pProp);
if (SUCCEEDED(hr)) 
{
    // Get the filter's name and IUnknown pointer.
    FILTER_INFO FilterInfo;
    hr = pFilter->QueryFilterInfo(&FilterInfo); 
    IUnknown *pFilterUnk;
    pFilter->QueryInterface(IID_IUnknown, (void **)&pFilterUnk);

    // Show the page. 
    CAUUID caGUID;
    pProp->GetPages(&caGUID);
    pProp->Release();
    OleCreatePropertyFrame(
        hWnd,                   // Parent window
        0, 0,                   // Reserved
        FilterInfo.achName,     // Caption for the dialog box
        1,                      // Number of objects (just the filter)
        &pFilterUnk,            // Array of object pointers. 
        caGUID.cElems,          // Number of property pages
        caGUID.pElems,          // Array of property page CLSIDs
        0,                      // Locale identifier
        0, NULL                 // Reserved
    );

    // Clean up.
    pFilterUnk->Release();
    FilterInfo.pGraph->Release(); 
    CoTaskMemFree(caGUID.pElems);
}
```







参考：

[Basic DirectShow Tasks](https://docs.microsoft.com/zh-cn/windows/desktop/DirectShow/basic-directshow-tasks)

