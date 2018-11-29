# Audio Capture

[TOC]



## 一：介绍

音频采集通过Audio Capture Filter，它支持数字和模拟音频信号。一般Audio Capture Filter是个线性器件，Input表示输入源，例如麦克风等，具体行为还要看厂家手册。





## 二：设备枚举

设备枚举通过《Basic Tasks》#设备枚举的系统枚举方法。详情参照相关部分。

> ICreateDevEnum::CreateClassEnumerator 时候的枚举类型为CLSID_AudioInputDeviceCategory，表示音频采集设备。





## 三：Audio Capture Graph

### 3.1 类型

Audio Capture Graph有以下几种类型：

- Audio-only AVI file：Audio Capture Filter -> AVI Mux Filter -> File Writer Filter;
- WAV file：Audio Capture Filter ->  WavDest Filter Sample -> File Writer Filter;
- Windows Media Audio (.wma) file：Audio Capture Filter -> WM ASF Writer Filter.

**Audio Capture Graph 必须包含Multiplexer(多路复用器)和File Writer(文件写入)！因为音频包必须被封装成相应格式，所以需要Multiplexer。**

### 3.2 建立

建立Audio Capture Graph可以通过：

- Capture Graph Builder
- 手动创建。

通过Capture Graph Builder的方法和Video Capture类似，这里主要介绍手动创建的方法。

```c++
枚举设备

选择设备，并创建Audio Capture Filter

// 确定输入类型
查询Audio Capture Filter的IAMAudioInputMixer接口
调用put_Enable确定输入类型（输入类型字符串不同，可通过waveOutOpen,mixerOpen and mixerGetLineInfo等方法查询）

创建Multiplexer和File Writer Filter，并添加到Graph

设置File Wirter Filter的文件名称
```

Example：

```c++
IBaseFilter *pSrc = NULL, *pWaveDest = NULL, *pWriter = NULL;
IFileSinkFilter *pSink= NULL;
IGraphBuilder *pGraph;

// Create the Filter Graph Manager.
hr = CoCreateInstance(CLSID_FilterGraph, NULL, CLSCTX_INPROC_SERVER,
    IID_IGraphBuilder, (void**)&pGraph);

// This example omits error handling.

// Not shown: Use the System Device Enumerator to create the 
// audio capture filter.

// Add the audio capture filter to the filter graph. 
hr = pGraph->AddFilter(pSrc, L"Capture");

// Add the WavDest and the File Writer.
hr = AddFilterByCLSID(pGraph, CLSID_WavDest, L"WavDest", &pWaveDest);
hr = AddFilterByCLSID(pGraph, CLSID_FileWriter, L"File Writer", &pWriter);

// Set the file name.
hr = pWriter->QueryInterface(IID_IFileSinkFilter, (void**)&pSink);
hr = pSink->SetFileName(L"C:\\MyWavFile.wav", NULL);

// Connect the filters.
hr = ConnectFilters(pGraph, pSrc, pWaveDest);
hr = ConnectFilters(pGraph, pWaveDest, pWriter);

// Not shown: Release interface pointers.
```

### 3.3 添加预览

有些Audio Capture Filter并没有提供预览Pin，因此可以在它后边加入`Infinite Pin Tee`来提供多个输出Pin：

![audio_capture_graph](./audio-capture-graph.png)

DirectSound Renderer是默认的Renderer，所以也不需要显式连接。





## 四：配置Audio Capture Filter属性

Audio Capture Filter的Input Pin暴露**IAMAudioInputMixer**接口，如上所属，可以通过接口的`put_Enable` 方法设置输入源，同样也可以设置高低音和音量值。

采样率和音频格式由驱动决定，可以通过Output Pin的 **IAMStreamConfig**接口枚举和设置。**IAMBufferNegotiation** 接口可以设置音频预览的缓存，缓存越小预览延时越小。默认设置为半秒缓存。





参考：

[Audio Capture](./https://docs.microsoft.com/zh-cn/windows/desktop/DirectShow/audio-capture)

