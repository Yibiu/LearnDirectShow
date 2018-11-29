# Playback

[TOC]



## 一：流程

Playback使用Filter Graph建立，不借助Capture Graph Builder接口。

程序流程如下：

```c++
建立Filter Graph

查询Media Control接口

查询Media Event接口 -> 设置窗口事件

// Source Filter
通过AddSourceFilter添加文件源Filter

// Renderer Filter
创建Video Renderer filter并添加到Filter Graph
创建DirectSound filter并添加到Filter Graph

查询IFilterGraph2接口，并枚举和自动连接Source的输出Pin,自动添加中间Filters

检查移除不需要的Filters
```

> 接口继承关系 IFilterGraph——>IGraphBuilder——>IFilterGraph2——>IFilterGraph3。
>
> IFilterGraph2继承自IGraphBuilder。



