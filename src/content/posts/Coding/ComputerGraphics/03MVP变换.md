---
title: 03MVP变换
published: 2025-09-06T10:42:28Z
description: ''
image: ''
tags: []
category: ''
draft: false
---

# 03MVP变换

MVP变换是计算机图形学中的一个重要概念，代表模型(Model)、视图(View)和投影(Projection)三个变换矩阵的组合。通过MVP变换，可以将三维场景中的点转换为二维屏幕上的点，从而实现3D渲染。

## 1. 模型变换 (Model Transformation)

将物体从模型的本地坐标系，转换到世界坐标系。可以看作为将模型放置在当前渲染的场景中。

模型变换包括平移(Translation)、旋转(Rotation)和缩放(Scaling)等操作。

## 2. 视图变换 (View Transformation)

将世界坐标系中的所有点，转换到以相机为中心的视图坐标系中。可以看作为当摄像机移动和观察时，将场景物体随着摄像机移动，而相机位置固定在圆点，从而使得场景物体相对于相机的位置发生变化。

视图变换通常使用相机的视图矩阵来实现，该矩阵描述了相机的位置和方向。

## 3. 投影变换 (Projection Transformation)

将视图坐标系中的点，通过透视投影(或正交投影)，将原始的3D视图坐标压缩到3D裁剪空间(Clip Space)，随后通过齐次除法，进一步压缩到[-1, 1]的NDC(Normalized Device Coordinates)立方体空间中。在视图空间与NDC空间下，摄像机始终在原点。

