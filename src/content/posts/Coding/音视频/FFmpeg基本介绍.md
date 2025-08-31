---
title: FFmpeg基本介绍
published: 2025-08-31T11:05:47Z
description: 'FFmpeg基本介绍'
image: ''
tags: [音视频, FFmpeg]
category: '音视频'
draft: false
---

# FFmpeg基本介绍

FFmpeg是一个开源的音视频处理工具集，广泛应用于多媒体处理领域。它提供了丰富的功能，包括音视频的编解码、格式转换、流媒体处理等。FFmpeg支持多种音视频格式和编解码器，具有高效的性能和灵活的扩展性。

[FFmpeg文档](https://ffmpeg.org/ffmpeg.html)

FFmpeg的框架如图所示：

![FFmpeg基本介绍-2025-08-31-11-06-32](https://cdn.jsdelivr.net/gh/chrisnake11/picgo@main/blog/FFmpeg基本介绍-2025-08-31-11-06-32.png)

+ FFmpeg基本结构：
1. 解复用器Demuxer，负责将音视频流分离成独立视频流、音频流、字幕流等。这些`streams`会被封装为`packets`传递给解码器。
2. 解码器Decoder，负责将压缩的音视频数据`packets`解码出原始数据`raw data`，并封装为`frames`传递给下一层。
3. 过滤器Filter，负责对音视频的`raw frames`进行各种处理，如剪辑、调整音量、添加水印等。
4. 编码器Encoder，负责将多个容器的原始音视频数据`raw frames`，分别使用不同的编码器(视频编码器、音频编码器、字幕编码器等...)编码成特定格式。
5. 复用器Muxer，负责将多个视频流、音频流、字幕流合并成一个文件。

# FFmpeg的基本命令

FFmpeg的基本命令由选项和参数组成，每个选项和参数会按顺序作用于下一个输入或输出文件。因此编写ffmpeg的命令时，需要注意选项和参数的顺序。建议先指定所有的输入文件，在指定所有的输出文件。

+ `-i`选项用于指定输入文件。

以下命令表示将`input.mkv`文件转换为`output.mp4`文件。
```bash
ffmpeg -i input.mkv output.mp4
```

## 流的选择

FFmpeg中，流的选择可以通过`-map`选项来指定需要处理的音视频流，每个`-map`可以提取一个指定的流。`v,a,s,d`分别表示视频(video)、音频(audio)、字幕(subtitle)、数据(data)流。

例如`0:v:0`表示选择`index=0`的输入文件的`index=0`的视频(video)流，`0:a:0`表示选择`index=0`的输入文件的`index=0`的音频(audio)流。

```bash
ffmpeg -i input.mkv -map 0:v:0 -map 0:a:0 output.mp4
```

可以使用`-vn/-an/-sn/-dn`代替`-map`，直接忽略视频/音频/字幕/数据流。

## 编码器选择(Encoder)

FFmpeg中，`-c`选项用于选择指定的视频或音频编码器。例如`-c copy`表示拷贝编码器，`-c:v libx264`表示选择H.264视频编码器，`-c:a aac`表示选择AAC音频编码器。

### 直接拷贝(copy)

`copy`参数用于直接拷贝音视频流，而不进行跳过重新解码和转码的步骤（前提是编码格式相同, mkv和mp4都支持H.264编码）。

使用`-c copy`选项可以实现这一点。例如：将input.mkv文件中的第1个视频流直接拷贝到output.mp4文件中，而不进行转码。

```bash
ffmpeg -i input.mkv -map 0:1 -c copy output.mp4
```

除了将单个文件的一个流拷贝到目标文件，还可以同时将多个文件的不同流合并到一个文件中。例如，以下命令将input1.mkv文件中的第1个流和input2.aac文件中的第1个音频流合并到output.mp4文件中。

```bash
ffmpeg -i input1.mkv -i input2.aac -map 0:0 -map 1:0 output.mp4
```

另外，FFmpeg还可以实现单个视频文件的拆分。例如，以下命令将input.mkv文件拆分为两个部分，分别输出为output1.mp4和output2.mp4。

```bash
ffmpeg -i INPUT.mkv -map 0:0 -c copy output1.mp4 -map 0:1 -c copy output2.mp4
```

### 文件转码(Transcoding)

通过将上述编码器选择项`-c`中的拷贝参数`copy`替换为其他的编码器，FFmpeg可以方便地对音视频文件进行转码操作。转码的过程包括解码和重新编码，FFmpeg支持多种音视频编码格式。

以下命令将input.mkv文件转码为output.mp4文件，`-c:v`表示对视频流使用H.264视频编码，`-c:a`表示对音频流直接`copy`。

```bash
ffmpeg -i input.mkv -c:v libx264 -c:a copy output.mp4
```

### 