---
title: Shading
published: 2025-09-04T11:00:46Z
description: 'Games101的Shading课程笔记'
image: ''
tags: [Computer Graphics, Shading]
category: 'Computer Graphics'
draft: false
---

# Shading 着色

## Painter Algorithm

类似于油画家，先画背景，再画上一层的图像。

缺点：需要对图像进行深度排序，处理复杂场景时效率较低。并且对于自遮挡情况无法排序。

## Z-Buffer Algorithm

Z-Buffer算法是一种基于深度（depth）的图像合成技术。它为当前视图中的每个像素维护一个深度值（Z值）。

在渲染过程中逐个渲染模型，计算模型像素到视点的深度与Z-Buffer中的值，从而决定是否更新Z-Buffer的深度和当前视图的颜色值。

```cpp
// 实际的画面缓冲区
float colorBuffer[width][height];
// z-buffer，像素深度缓冲区
float zBuffer[width][height];
// 场景中所有模型
std::vector<Model> models;
// 逐个渲染模型
for(const auto& model : models) {
    // 遍历模型的像素
    for(const auto& pixel : model){
        // 获取当前像素的深度值
        float z_value = pixel.getZValue();
        // 当前像素的深度值小于Z-Buffer中的值（更近）
        if(z_value < zBuffer[pixel.getX()][pixel.getY()]) {
            // 更新z-buffer的记录，刷新color buffer
            zBuffer[pixel.getX()][pixel.getY()] = z_value;
            colorBuffer[pixel.getX()][pixel.getY()] = pixel.getColor();
        }
    }
}

// 着色完毕，绘制一帧图像
paint(colorBuffer);
```

优点：简单易实现，能够处理复杂场景中的自遮挡问题。

缺点：需要额外的内存来存储Z-Buffer，可能导致性能瓶颈。

复杂度：`O(N)`, `N`为场景中像素的数量。

## Shading 着色

对于不同的材质，使用不同的着色模型来计算最终颜色值。

### Bling-Phong Model

对于一个需要着色的像素，需要知道3个向量。
1. 三角形平面的法向量$n$：可以通过三角形边的叉积得到。
2. 光源方向向量$l$：光源和着色点的差。
3. 视线方向向量$v$：相机和着色点的差。

对于不同的材质，应用不同的反射模型，如：漫反射和镜面反射模型。

#### 漫反射

漫反射模型假设光线均匀地散射在表面上，表面颜色与光源的角度无关，只和其法向量的方向与漫反射系数有关。其计算公式为：

漫反射系数 * 光源强度 * 距离衰减($/r^2$) * 夹角cos(接收到的能量强度)

$$
I_{diff} = k_d \cdot I_{light} / r^2 \cdot \max(0, n \cdot l)
$$

其中，$k_d$为表面材质的漫反射系数，$I_{light}$为光源的强度，$r$为光源到表面的距离，$n$为法向量，$l$为光源方向向量。

$$
I / (r^2)
$$

根据能量守恒，光源发出的能量均匀作用在一个球壳上，因此能量强度与距离成反比。

$$
\max(0, n \cdot l)
$$

当夹角大于90度时，光线照不到表面，点积为负，能量为0。

#### 高光

当观察者的视角与光线的反射方向接近时，会产生高光效果。

特性：当视线方向向量(v)与反射角(l')十分接近时，光源向量(l)和视线向量(v)的半程向量(h)（夹角的中间）与法线向量(n)十分接近。


**半程向量**,也就是两个向量的和。计算如下：
> 相比于反射角，半程向量的计算更加快速简单。


$$
h = bisector(v, l) = \frac{l + v}{\|l + v\|}
$$

因此，着色公式中能量吸收的夹角的部分，可以被更新为半程向量与法线的夹角。

$$
L = k_s \cdot I_{light}/(r^2) \cdot \max(0, n \cdot h)^{p}
$$

$p$为高光指数。

+ 高光指数的作用

将cos函数调整为更合适的形式，使其在高光区域更加集中。

![Shading-2025-09-04-12-31-41](https://cdn.jsdelivr.net/gh/chrisnake11/picgo@main/blog/Shading-2025-09-04-12-31-41.png)


#### 环境光

设置一个全局的环境光照强度。给所有的像素着色时增加一个环境光分量。

$$
I_{ambient} = k_a \cdot I_{ambient}
$$

其中， $k_a$为材质的环境光反射系数，$I_{ambient}$为环境光强度。


### 着色频率

着色频率是指在渲染过程中，像素着色计算的频率。高频率的着色计算可以带来更好的视觉效果，但也会增加计算开销。

![Shading-2025-09-04-12-43-59](https://cdn.jsdelivr.net/gh/chrisnake11/picgo@main/blog/Shading-2025-09-04-12-43-59.png)

左边的方法：**Flat Shading**，对整个面使用相同的颜色。

中间的方法: **Gouraud Shading**，对顶点进行着色，然后对内部像素进行插值。

右边的方法：**Phong Shading**，对每个像素进行着色计算，考虑更精细的光照模型。!

#### 不同图像面数应用着色方法的效果

![Shading-2025-09-04-12-49-11](https://cdn.jsdelivr.net/gh/chrisnake11/picgo@main/blog/Shading-2025-09-04-12-49-11.png)

可以观察出，当面数很高时，采用**Flat Shading**也可以产生不错的效果。

面数较少时，**Phong Shading**能够提供更好的细节和光照效果。

**Gouraud Shading**在大多数情况下提供了一个折中的选择。

#### 顶点的法向量

特殊情况：对于一个由三角形组成的球面模型，每个顶点的法向量实际上就是球心到顶点的向量。

一般情况：而对于由三角形组成的其他图形，顶点的法向量可以通过其关联的**相邻三角形的法向量进行加权平均**得到。
> 权重可以根据相邻三角形的面积或其他特征进行设置。


### 纹理映射（纹理贴图）

例如地球仪本身是一个球体，纹理映射的过程就是将地球仪表面的图像（纹理）映射到其三维球体模型上。每个图像的像素点对应模型上的一个坐标。

映射的计算方法非常复杂（涉及到UV坐标的计算、纹理过滤等），通常需要在GPU中进行处理。

纹理平面图上的坐标使用(u, v)表示，范围通常在[0, 1]之间。
> 纹理的平面图通常是无缝的，因此在映射时需要考虑纹理的重复性和边界处理。

随后将UV坐标映射到三维模型的顶点上。每个UV坐标与3D模型顶点一一对应。
> 在建模时，每个顶点就被分配了一个对应的UV坐标。

然后在片段着色阶段，通过顶点插值计算三角形内部像素的重心坐标，使用重心坐标对顶点的纹理值进行加权计算，得到最终的纹理颜色。

#### 三角形插值计算（模型表面像素的插值计算）

对于三角形内部的点，我们可以计算这个点在三角形内部的**重心坐标**(Barycentric Coordinates)，来进行**颜色、纹理等属性的插值**。

重心坐标的公式:

$$
V = \alpha V_1 + \beta V_2 + \gamma V_3 \\

\alpha + \beta + \gamma = 1
$$

其中，$\alpha, \beta, \gamma$分别为重心坐标，$V_1, V_2, V_3$为三角形的三个顶点坐标，$V$为三角形内部的点坐标。

随后根据重心坐标$\alpha, \beta, \gamma$，可以对颜色、纹理等属性进行插值计算。

注意：重心坐标在投影之后会发生变化，因此要在3维空间中进行插值计算，然后再投影。

```cpp
Pixel[] pixels;
Texture texture;

for(auto& pixel : pixels){
    (u, v, w) = pixel.getBarycentricCoordinates();
    texcolor = texture.getTexColor(u, v, w);
    pixel.setColor((u, v, w), texcolor);
}
```

#### 低分辨率下的纹理映射

当贴图的精度不足时，**模型的多个片元可能会映射到同一个纹理像素上**,可能会出现**锯齿**，甚至**马赛克**现象。

因为，虽然UV坐标通常是浮点数（范围[0,1]），但在纹理采样时需要将其乘以纹理的宽高，从新转换为纹理图片上的离散像素索引。

+ **最近点采样**时，UV坐标会被乘以纹理宽高，然后四舍五入或取整，得到对应的像素索引。
+ **双线性插值**时，UV坐标会对应到纹理像素之间，通过插值计算颜色，减少锯齿和马赛克。

#### 双线性插值 Bilinear Interpolation（纹理图像的插值计算）

当从UV坐标乘以纹理图片宽高得到纹理像素坐标时（还是小数），寻找周围最近的4个像素进行插值计算，根据距离进行加权平均，得到最终的纹理颜色。

![Shading-2025-09-04-17-37-46](https://cdn.jsdelivr.net/gh/chrisnake11/picgo@main/blog/Shading-2025-09-04-17-37-46.png)

```cpp
// 周围的4个点
Pixel u00, u01, u10, u11;
// 目标像素
Pixel target;

float lerp(float t, float a, float b) {
    return a + (t - a)*(b - a);
}

// 对x方向加权平均
u0 = lerp(target.x, u00.x, u01.x);
u1 = lerp(target.x, u10.x, u11.x);

target.x = lerp(target.x, u0, u1);

// 对y方向插值平均
v0 = lerp(target.y, u00.y, u10.y);
v1 = lerp(target.y, u01.y, u11.y);
target.y = lerp(target.y, v0, v1);
```

#### 当纹理分辨率过大

模型UV到纹理UV的映射过程，可以看作为在纹理图上进行采样，当模型精度小于纹理精度时，导致纹理信息丢失就会出现**走样(Aliasing)**，出现类似于摩尔纹、锯齿等现象。

解决方法：
1. 超采样， 在更高分辨率下渲染场景，然后再缩小到目标分辨率。性能代价太高。
2. **模糊，通过平均或者卷积降低图像的分辨率。**

对于同一个场景下，**近的模型精度往往更高，因此需要高分辨率的材质贴图；而远端的模型，通常使用精度更低的模型，因此就需要降低材质的分辨率**。为了动态地给不同精度的模型应用材质贴图，通常会使用Mipmap技术。

### Mipmap

Mipmap是一种纹理映射技术，通过预先计算并存储不同分辨率的纹理图像，来提高渲染效率和图像质量。在渲染时，根据物体与摄像机的距离选择合适的纹理级别，从而减少纹理采样时的计算量。

Mipmap的生成过程通常包括以下步骤：
1. 从原始纹理图像开始，通过逐渐降低其分辨率，生成一系列的低分辨率纹理图像。
2. 将这些低分辨率纹理图像存储在一个金字塔结构中，称为Mipmap链。

在渲染时，根据物体的屏幕空间大小选择合适的Mipmap级别进行纹理采样。这种方法可以有效减少纹理走样和锯齿现象，提高渲染性能。

+ 从左上到右下，分别表示为距离从远到近情况下，需要使用的贴图。

![Shading-2025-09-04-17-54-39](https://cdn.jsdelivr.net/gh/chrisnake11/picgo@main/blog/Shading-2025-09-04-17-54-39.png)

Mipmap的效果图：

![Shading-2025-09-04-20-20-21](https://cdn.jsdelivr.net/gh/chrisnake11/picgo@main/blog/Shading-2025-09-04-20-20-21.png)

#### Mipmap级别的计算

首先模型顶点的UV坐标，然后计算这个顶点周围顶点的UV坐标的映射后的偏移量，求微分计算偏移变化率。

使用max函数取出最大的偏移量作为最终的UV偏移量。

再使用$log_2$函数计算出合适的Mipmap级别，从而查询到对应的正方形区域的纹理图像。

#### 在Mipmap中三线性插值

对于第n级和第n-1级的Mipmap，可以在单个Mipmap内部使用双线性插值的方法进行纹理采样。在此基础上，对相邻的Mipmap级别进行插值，得到更平滑的纹理效果。

图中可以看到，对于Mipmap的级别通过插值实现了更平滑的连续的Mipmap过渡效果。

![Shading-2025-09-04-20-21-53](https://cdn.jsdelivr.net/gh/chrisnake11/picgo@main/blog/Shading-2025-09-04-20-21-53.png)


**缺点：对于远处的模型，会出现严重的模糊现象。**

![Shading-2025-09-04-20-19-54](https://cdn.jsdelivr.net/gh/chrisnake11/picgo@main/blog/Shading-2025-09-04-20-19-54.png)

#### 各向异性过滤 Anisotropic Filtering

将Mipmap中的正方形区域纹理图像替换成多个在垂直和水平方向上压缩的矩形区域的纹理图像。

随后使用与Mipmap类似的方法计算UV坐标的偏移量，从而确定到底使用哪个矩形区域的纹理图像进行采样。

![Shading-2025-09-04-20-27-42](https://cdn.jsdelivr.net/gh/chrisnake11/picgo@main/blog/Shading-2025-09-04-20-27-42.png)

从下图可以看出，对于**斜向拉伸的纹理图像**能够够更好地保留细节，减少模糊现象。但是对于部分极端情况，由于匹配的范围过大，仍然会出现模糊现象。

![Shading-2025-09-04-20-18-24](https://cdn.jsdelivr.net/gh/chrisnake11/picgo@main/blog/Shading-2025-09-04-20-18-24.png)

为了解决贴图匹配区域过大的问题，还提出了EWA Filtering（Elliptical Weighted Average Filtering，椭圆加权平均滤波）技术。这个方法对于各向异性过滤的匹配区域使用若干个（3个）刚好覆盖的椭圆形状进行采样，然后加权平均从而更好地保留细节。
> 缺点：付出了更多的性能开销。3倍的采样计算。

![Shading-2025-09-04-20-27-56](https://cdn.jsdelivr.net/gh/chrisnake11/picgo@main/blog/Shading-2025-09-04-20-27-56.png)