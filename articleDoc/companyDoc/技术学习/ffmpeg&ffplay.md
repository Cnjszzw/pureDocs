# ffmpeg & ffplay

> 参考文章如下：
>
> ```url
> https://blog.csdn.net/leixiaohua1020/article/details/15811977
> ```

## 〇、背景介绍

本章主要介绍一下FFMPEG都用在了哪里（在这里仅列几个我所知的，其实远比这个多）。说白了就是为了说明：FFMPEG是非常重要的。

使用FFMPEG作为内核视频播放器：

```
Mplayer，ffplay，射手播放器，暴风影音，KMPlayer，QQ影音...
```

使用FFMPEG作为内核的Directshow Filter：

```
ffdshow，lav filters...
```

使用FFMPEG作为内核的转码工具：

```
ffmpeg，格式工厂...
```

事实上，FFMPEG的视音频编解码功能确实太强大了，几乎囊括了现存所有的视音频编码标准，因此只要做视音频开发，几乎离不开它。

## 一、核心信息

FFmpeg和ffplay是多媒体处理领域的重要工具，以下是它们的核心信息：

### 1. **FFmpeg**

- 定位 ：开源多媒体框架，提供音视频录制、转换、流媒体传输等功能。
- 核心能力 ：
  - 支持多种格式编解码（通过`libavcodec`等库）。
  - 可处理音视频剪辑、转码、压缩等操作。
  - 被广泛应用于视频平台（如YouTube）、播放器（如VLC）等。

### 2. **ffplay**

- 定位 ：基于FFmpeg和SDL库的轻量级播放器，主要用于开发测试。
- 特点 ：
  - 支持本地文件与在线流媒体播放（如RTMP）。
  - 提供解码、渲染、同步等播放器核心逻辑的参考实现。
  - 开发者常通过分析其源码学习播放器架构。

##  二、**二者关系**

- ffplay是FFmpeg项目的子工具，依赖FFmpeg的解码能力。
- FFmpeg负责底层处理（如解码），ffplay负责渲染与交互。

**示例场景** ：

- 用FFmpeg将视频转为MP4格式：`ffmpeg -i input.avi output.mp4`。
- 用ffplay直接播放网络流：`ffplay rtmp://live.example.com/stream`。

两者均是开源工具，适合开发者、多媒体工程师或技术爱好者使用。

## 三、 ffmpeg程序的使用（ffmpeg.exe，ffplay.exe，ffprobe.exe）

### 3.1 ffmpeg.exe

ffmpeg是用于转码的应用程序。

一个简单的转码命令可以这样写：

将input.avi转码成output.ts，并设置视频的码率为640kbps

```shell
ffmpeg -i input.avi -b:v 640k output.ts
```

### **3.2 ffplay.exe**

ffplay是用于播放的应用程序。

一个简单的播放命令可以这样写：

播放test.avi

```shell
ffplay test.avi
```

### **3.3 ffprobe.exe**

ffprobe是用于查看文件格式的应用程序。

这个就不多介绍了。

## 四、ffmpeg库的使用：视频播放器









## X. Q & A

### 1.我们的项目中有使用ffmpeg这个库吗，如果有，在哪里使用的，怎么使用的？







