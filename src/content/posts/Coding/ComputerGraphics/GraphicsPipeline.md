---
title: Graphics Pipeline
published: 2025-09-04T13:00:43Z
description: 'Games101, Graphics Pipeline学习笔记。'
image: 'https://cdn.jsdelivr.net/gh/chrisnake11/picgo@main/blog/GraphicsPipeline-2025-09-04-13-01-57.png'
tags: [Computer Graphics, Graphics Pipeline, 图形渲染管线]
category: 'Computer Graphics'
draft: false
---

# Graphics Pipeline 实时图形渲染管线

![Graphics Pipeline-2025-09-04-13-01-57](https://cdn.jsdelivr.net/gh/chrisnake11/picgo@main/blog/GraphicsPipeline-2025-09-04-13-01-57.png)

1. 顶点处理 (Vertex Processing): 负责将三维模型的顶点坐标转换到裁剪空间，并进行视口变换。
2. 三角形处理 (Triangle Processing): 负责将顶点连裁剪空间中的三角形。
3. 光栅化 (Rasterization): 负责将三角形转换为片段，并为每个片段生成对应的深度值和颜色值。
4. 片段处理 (Fragment Processing): 负责对每个片段进行着色计算，生成最终的颜色值。
5. 帧缓冲操作 (Framebuffer Operations): 负责将片段的颜色值写入帧缓冲区，并进行后期处理。

这些工作在GPU中都会自动处理，程序员的工作就是编写着色器程序，自定义顶点、片段、几何等内容的着色方式。