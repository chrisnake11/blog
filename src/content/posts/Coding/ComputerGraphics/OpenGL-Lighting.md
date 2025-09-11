---
title: OpenGL-Lighting
published: 2025-09-11T13:03:04Z
description: 'Learning OpenGL中Lighting章节的总结，包括环境光、漫反射、镜面反射计算，镜面材质贴图、发光材质贴图，平行光、点光源、聚光灯、多光源等内容。'
image: 'https://raw.githubusercontent.com/chrisnake11/picgo/main/blog/OpenGL-Lighting-2025-09-11-13-07-03.png'
tags: [Computer Graphics, OpenGL, Lighting]
category: 'Computer Graphics'
draft: false
---

# OpenGL-Lighting

![OpenGL-Lighting-2025-09-11-13-07-03](https://raw.githubusercontent.com/chrisnake11/picgo/main/blog/OpenGL-Lighting-2025-09-11-13-07-03.png)

本文总结自[Learning OpenGL](https://learnopengl.com/Lighting/Basic-Lighting)，包括环境光、漫反射、镜面反射计算，镜面材质贴图、发光材质贴图，平行光、点光源、聚光灯、多光源等内容。

## 1. 光照计算

在光照计算中，光照是能够叠加的，例如红色和黄色光源叠加会产生橙色光照效果。两个白色光源叠加会产生更亮的白色光照效果。

那么在模拟光照时，我们可以将光照分为三种类型：环境光(Ambient Light)、漫反射(Diffuse Light)、镜面反射(Specular Light)。

我们所见到的光，就是无数个光源发出光线，同时在单个点或者平面上的光照效果的叠加。

OpenGL中的光线模拟，就是在单个点或者平面上，计算所有光源在这个顶点或者平面上产生的光照效果（包括环境光、漫反射、镜面反射），然后将这些光照效果叠加，得到最终的光照效果。

### 1.1 环境光(Ambient Light)

环境光是指来自各个方向的光线，均匀地照亮物体表面。环境光没有特定的方向，因此不会产生阴影效果。它模拟了现实生活中由于光线反射和散射而产生的柔和光照效果。

通常，环境光的强度较低，以避免过度照亮场景，设置为一个较小的3维向量。

```cpp
glm::vec3 ambientStrength = glm::vec3(0.1, 0.1, 0.1); // 环境光强度
glm::vec3 ambient = ambientStrength * lightColor; // 计算环境光
```

### 1.2 漫反射(Diffuse Light)

漫反射是指光线照射到物体表面后，**向各个方向均匀散射的现象**，因此漫反射的效果与相机观察的视角无关。漫反射光的强度取决于光线与物体表面法线之间的夹角。

光线与法线夹角越小，漫反射光的强度越大；夹角越大，漫反射光的强度越小。

漫反射光的计算公式为：

```cpp
glm::vec3 norm = normalize(Normal); // 法线向量
glm::vec3 lightDir = normalize(lightPos - FragPos); // 光线方向向量
float diff = max(dot(norm, lightDir), 0.0); // 计算漫反射系数
glm::vec3 diffuse = diff * lightColor; // 计算漫反射光
```
首先我们计算出法向量与光线方向向量，并且归一化为单位向量。然后通过点积计算得到两个向量的夹角余弦值（单位向量的点积等于余弦值），作为漫反射系数。最后将漫反射系数与光源颜色相乘，得到漫反射光。

### 1.3 镜面反射(Specular Light)

镜面反射是指光线照射到物体表面后，按照入射角等于反射角的规律反射出去的现象。镜面反射光的强度取决于观察方向与反射方向之间的夹角。

光线与法线夹角越小，镜面反射光的强度越大；夹角越大，镜面反射光的强度越小。

镜面反射光的计算公式为：

```cpp
glm::vec3 viewDir = normalize(viewPos - FragPos); // 视线方向向量
glm::vec3 reflectDir = reflect(-lightDir, norm); // 反射方向向量
float spec = pow(max(dot(viewDir, reflectDir), 0.0), shininess); // 计算镜面反射系数
glm::vec3 specular = spec * lightColor; // 计算镜面反射光
```

首先我们计算出视线方向向量，使用内置函数`reflect()`计算出反射方向向量，并且将它们归一化为单位向量。然后通过点积计算得到两个向量的夹角余弦值，作为镜面反射系数。最后将镜面反射系数与光源颜色相乘，得到镜面反射光。

镜面反射强度也可以通过半程向量(Halfway Vector)来计算：

```cpp
glm::vec3 halfwayDir = normalize(lightDir + viewDir); // 半程向量
float spec = pow(max(dot(norm, halfwayDir), 0.0), shininess); // 计算镜面反射系数
```
当光源方向与视线方向相同时，半程向量与法线方向相同，此时镜面反射光最强，利用这个特性可以简化计算。

另外，镜面反射的强度还受到材质的影响，通常使用一个`shininess`参数来控制镜面反射的锐度。`shininess`值越大，镜面反射越集中，光斑越小；`shininess`值越小，镜面反射越分散，光斑越大。`shininess`的值通常在1到256之间选择。

![Learn OpenGL](https://learnopengl.com/img/lighting/basic_lighting_specular_shininess.png)

上图来自于[Learn OpenGL](https://learnopengl.com/Lighting/Basic-Lighting)，展示了不同`shininess`值对镜面反射效果的影响。


### 1.4 镜面材质贴图(Specular Map)

为了给模型的特殊部位增加镜面反射的效果，如：木箱子的边框为金属，因此木头部分没有镜面反射，而边框部分需要镜面反射。

我们可以额外增加一份边框部分的贴图，根据这份贴图的坐标，来计算出额外的镜面反射的光照效果。

将这份额外的镜面反射效果添加到着色器的光照结果中，就可以实现局部材质的镜面反射效果。

### 1.5 发光材质贴图(Emissive Map)

和镜面反射的方法类似，我们可以额外增加一个具有发光材质的贴图，只要将这个发光材质贴图的环境光设置为最大，那么无论在什么环境下他的光照着色都是固定的，就可以产生类似自发光的效果。

## 2 光源类型

### 2.1 平行光(Directional Light)

### 2.2 点光源(Point Light)

#### 衰减(Attenuation)

### 2.3 聚光灯(Spotlight)

### 2.4 多光源(Multiple Lights)

