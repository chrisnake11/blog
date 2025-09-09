---
title: 03MVP变换
published: 2025-09-06T10:42:28Z
description: '基于Learning-OpenGL的MVP和3D空间变换的理解'
image: 'https://learnopengl-cn.github.io/img/01/08/coordinate_systems.png'
tags: [Computer Graphics, MVP Transformation]
category: 'Computer Graphics'
draft: false
---

# 03MVP变换

![3D坐标空间](https://learnopengl-cn.github.io/img/01/08/coordinate_systems.png)

(上图参考自[Learning OpenGL](https://learnopengl.com/Getting-started/Coordinate-Systems))

在OpenGL的视图空间中，存在以下六个重要的空间概念，表示了从单个物体到3D场景下，再到2D屏幕的完整转换过程。
1. 局部空间(Local Space)：物体的本地坐标系，通常以物体的几何中心为原点。
2. 世界空间(World Space)：场景的全局坐标系，所有物体的位置和方向都相对于这个坐标系定义。
3. 视图空间(View Space)：以摄像机为中心的坐标系，摄像机的位置固定在原点，场景物体相对于摄像机的位置发生变化。
4. 裁剪空间(Clip Space)：通过投影变换将视图空间中的点压缩到一个标准立方体内，通常是[-1, 1]的范围。
5. 归一化设备坐标空间(Normalized Device Coordinates, NDC)：裁剪空间经过齐次除法后得到的坐标空间，仍然是[-1, 1]的范围。
6. 屏幕空间(Screen Space)：最终的二维屏幕坐标系，经过视口变换将NDC坐标映射到实际的屏幕像素位置。

其中，由局部空间到裁剪空间的转换过程涉及3种变换矩阵，这3种变换由程序员或者API设计者根据图形变换要求实现，通常被称作为MVP变换矩阵：model (模型变换)、view (视图变换)、projection (投影变换)。

剩下的从NDC到屏幕空间的转换，通常由图形API（如OpenGL）自动处理。

## 1. 模型变换 (Model Transformation)

在图形学中，模型通常由一系列顶点(Vertex)组成，例如，一个矩形模型由4个顶点组成，一个立方体模型由8个顶点组成。模型的顶点坐标在模型的局部空间(Local Space)中定义，通常以模型的几何中心为原点。

将物体从模型的本地坐标系，转换到世界坐标系时，一般默认物体的局部坐标系与世界坐标系是重合的，即物体的几何中心位于世界坐标系的原点，物体的正方向与世界坐标系的正方向一致。

为了将模型放置在世界空间中的指定位置，并且调整姿态和大小，我们需要对模型进行变换，这个过程称为模型变换(Model Transformation)。

模型变换包括平移(Translation)、旋转(Rotation)和缩放(Scaling)等操作。

通常变换顺序为：缩放 -> 旋转 -> 平移。

因为平移和旋转不会改变局部坐标系原点的位置，局部坐标系原点在世界坐标系中的位置仍然是(0, 0, 0)，如果先进行平移操作，旋转和缩放操作会围绕世界坐标系的原点进行，而不是围绕物体的几何中心进行。产生的结果通常不是我们想要的。

## 2. 视图变换 (View Transformation)

从(Global Space)转换到(View Space)的过程，通常称为视图变换(View Transformation)或者摄像机变换(Camera Transformation)。

视图变换的目的是将场景中的所有物体转换到以摄像机为中心的坐标系中，使得摄像机的位置固定在原点，场景物体相对于摄像机的位置发生变化。

其中，摄像机的观察方向(-Front)为(Target - Position)，Target为观察的目标，Position为摄像机的位置。

随后，摄像机的右侧(Right)可以通过世界坐标系的正上方向量(World Up)与摄像机的观察方向的叉积计算得到。

再使用右侧方向和观察方向的叉积计算出摄像机的正上方向(Up)。

使用这三个方向向量(Right, Up, -Front)可以构建一个旋转矩阵，将世界空间(Global Space)中模型的顶点转换到以摄像机为中心的视图空间(View Space)中。

## 3. 投影变换 (Projection Transformation)

![view transformation](https://learnopengl.com/img/getting-started/perspective_frustum.png)

上图参考自[Learning OpenGL](https://learnopengl.com/Getting-started/Coordinate-Systems)

![03MVP变换-2025-09-09-16-06-37](https://raw.githubusercontent.com/chrisnake11/picgo/main/blog/03MVP变换-2025-09-09-16-06-37.png)

上图来自[Games 101](https://www.bilibili.com/video/BV1X7411F744)


将视图坐标系中的点，通过透视投影(或正交投影)，将原始的3D视锥空间中的坐标压缩到一个立方体空间中。从而得到符合透视效果的裁剪空间(Clip Space)坐标。在视锥体内的顶点会被映射到裁剪空间内，而视锥体外的顶点会被丢弃。

透视投影主要由视野(Field of View, FOV)、宽高比(Aspect Ratio)、近裁剪面(Near Plane)和远裁剪面(Far Plane)参数来决定，这些参数定义了摄像机的视锥体(Frustum)，即摄像机能够看到的空间范围。

> 透视投影通常用作第一人称视角的渲染，因为它能够模拟人眼的视觉效果，使得远处的物体看起来更小，近处的物体看起来更大。

> 正交投影则是将视图空间中的点直接映射到裁剪空间中，不考虑距离的影响。正交投影通常用于工程制图和某些类型的游戏渲染，因为它能够保持物体的真实尺寸和比例。

## 4. 其他变换

经过投影变换后，得到的裁剪空间(Clip Space)坐标需要进行齐次除法(Homogeneous Division)，将四维齐次坐标转换为三维归一化设备坐标(Normalized Device Coordinates, NDC)。

齐次除法的过程是将裁剪空间中的每个坐标分量除以齐次坐标的w分量，从而将其映射到[-1, 1]的范围内。经过齐次除法后，得到的NDC坐标可以直接用于屏幕空间的映射。

最后，NDC坐标通过视口变换(Viewport Transformation)映射到实际的屏幕像素位置(Screen Space)，这个过程通常由图形API（如OpenGL）自动处理。