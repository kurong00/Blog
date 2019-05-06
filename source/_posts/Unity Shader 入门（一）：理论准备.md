---
title: Unity Shader 入门（一）：理论准备
date: 2019/4/30
thumbnail: https://github.com/kurong00/EditGraphics/blob/master/blog/BlogPictures/Shader1/Shader1?raw=true
categories: Shader
---

## 1. 什么是Shader？

shader（着色器）是GPU的渲染流水线上的一小段程序，它负责将输入的Mesh（网格）以指定的方式，和输入的贴图或者颜色等组合作用后输出。绘图单元可以依据这个输出来将图像绘制到屏幕上。

## 2. Shader的分类？

shader大体上可以分为两类：

- 表面着色器（Surface Shader）：已经为你做了大部分的工作，只需要简单的编写就可以实现很多不错的效果。
- 片元着色器（Fragment Shader）：可以做的事情更多，相应的难度也会加大。可以在比较低的层级上进行更复杂（或者针对目标设备更高效）的开发。 

## 3. 什么是渲染流水线？

既然shader所在的阶段是渲染流水线上的一部分，那渲染流水线又是什么呢？ 

首先GPU上的渲染流水线任务是：<font color=#560A4A>从一个三维场景出发，把这些信息最终转换成一张二维图像。</font>我们可以将渲染流水线分成三个阶段：应用阶段->几何阶段->光栅化阶段。

![](https://github.com/kurong00/blog/blob/master/thumbnail/shader1/shader1_pipeline.PNG?raw=true)

我们逐一来看三个阶段：

### 3.1 应用阶段

![](https://github.com/kurong00/blog/blob/master/thumbnail/shader1/shader1_pipeline2.PNG?raw=true)

这个阶段是完全由开发者主导的，主要工作是：

- 准备数据：例如相机位置、模型位置、光源位置等等
- 粗粒度的剔除工作：把不可见的物体删除出去
- 设置渲染状态：例如使用的材质、纹理、shader等等
- 输出：需要渲染的几何信息，也就是渲染图元（rendering primitives），渲染图元可以是点、线、面等等，渲染图元就交给下一个几何阶段

### 几何阶段

![](https://github.com/kurong00/blog/blob/master/thumbnail/shader1/shader1_pipeline3.PNG?raw=true)

这个阶段用于处理几乎所有我们要绘制的几何相关的事情，比如决定画什么、怎么画、画在哪里等等（这一段主要在GPU上进行），因为事情太多，因此可以进一步细分成一个小的流水线：

此时到达Vertex Shader，shader会进行一些操作，例如：改变顶点位置，对顶点进行坐标变换(模型空间->世界空间->裁剪空间->屏幕空间)，贴图位置转换等等。  

下一步开始**图元装配**：将一个个零散的顶点组装成一个个三角形。  

下一步**曲面细分环节**（可选项，不一定经历这个环节，在Direct3D 11、OpenGL 4、OpenGL ES 3.2以上才支持）：将上一部的图元进行细分。

 再下一步**几何元着色器**（也是可选项，不一定经历这个环节,在Direct3D 10、OpenGL 3.2、OpenGL ES 3.2以上支持）：增加顶点或者片元数

 下一步**裁剪**：裁剪位于视锥外的片元

 最后一步**屏幕映射**：输出屏幕空间的二维坐标和每个顶点的深度值，着色等信息。

### 光栅化阶段

![](https://github.com/kurong00/blog/blob/master/thumbnail/shader1/shader1_pipeline4.PNG?raw=true)

将上一步得到的信息(深度值，着色，屏幕坐标等等)进行插值运算，确认哪些像素该被绘制在屏幕上。
**以上是渲染流水线的一个简单说明，真实的实现过程远比上面描述的复杂，但是好在Unity Shader已经封装了非常多的功能，下一节我们将开始分析第一个Unity Shader。**