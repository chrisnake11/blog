---
title: Rasterization
published: 2025-09-03T18:21:16Z
description: ''
image: ''
tags: [Rasterization, Computer Graphics]
category: 'Computer Graphics'
draft: false
---

# Rasterization(光栅化)

三角形：基本图形

三角形内部像素着色

三角形边界产生锯齿

采样：空间采样，时间采样

+ 空间、频率采样不够：锯齿、摩尔纹
+ 时间采样不够：车轮倒转效应

图像信号 -> 屏幕采样 -> 人眼采样

+ 频率与采样：傅里叶级数和傅里叶变换
  + 奈奎斯特定理：采样频率必须大于信号最高频率的两倍，以避免混叠现象。
  + 卷积：时域中的卷积等价于频域中的乘法。
  + 走样的本质：采样频率不够，出现了混叠现象。
+ 抗锯齿
  1. 原始数据**模糊（丰富低频信息，减少高频信息）**，**再采样**。等价于低通滤波，减少高频成分。
  2. 增加采样的频率。
  3. MSAA(Multi Sampling Anti Aliasing 多重采样抗锯齿)：在每个像素内进行多次采样（增加图像分辨率），取平均值求得近似值，得到原始分辨率的图像。
  4. FXAA(Fast Approximate Anti Aliasing 快速近似抗锯齿)：一种图像处理技术（与采样无关），通过分析图像中的边缘信息，智能地模糊锯齿边缘，达到抗锯齿效果。
  5. TAA(Temporal Anti Aliasing 时间抗锯齿)：通过利用时间上的连续帧信息，结合运动模糊和历史帧数据，减少锯齿和闪烁现象。
  6. DLSS(Deep Learning Super Sampling 深度学习超采样)：利用深度学习技术，通过低分辨率图像生成高分辨率图像，利用Deep Learning猜测缺失的像素，达到抗锯齿效果。（利用AI进行采样猜测）