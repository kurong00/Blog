---
title: Unity Shader 入门（一）：理论准备（二）
date: 2019/4/29
toc: true 
categories: Shader
---

## 什么是Shader？

Shader（着色器）：是渲染管线上的一小段程序，它负责将输入的Mesh（网格）以指定的方式和输入的贴图、颜色等组合作用然后输出。Shader开发者要做的就是根据输入，进行计算变换，产生输出。

shader大体上可以分为两类：

- 顶点着色器（Vertex Shader）
- 片元着色器（Fragment Shader）
<!-- more -->
而在Unity Shader中分为三类：

- Surface Shaders （表面着色器）：是Unity对Vertex/Fragment Shader的一层包装，可以以极少的代码来完成不同的光照模型与不同平台下需要考虑的事情；缺点是能够实现的效果不如片段着色器来的多。
- Vertex/Fragment Shaders （顶点/片断着色器）
- Fixed Function Shaders （固定管线着色器）：已被淘汰

## 什么是ShaderLab？

在Unity中，所有的Shader都是使用ShaderLab来编写的，从结构上来说，它定义了显示一个材质所需要的所有东西，而不仅仅是着色器代码，我们先来看一下ShaderLab的解构

![Figure 1](https://github.com/kurong00/EditGraphics/blob/master/BlogPictures/Shader2/1.png?raw=true)

一个shader包含多个属性（Properties)，然后是一个或多个的子着色器（SubShader)，在实际运行中，哪一个子着色器被使用是由运行的平台所决定的。每一个子着色器中包含一个或者多个的Pass。在计算着色时，平台先选择最优先可以使用的着色器，然后依次运行其中的Pass，然后得到输出的结果。最后指定一个FallBack，用来处理所有Subshader都不能运行的情况,一般FallBack的都是平台已经定义好的shader。  
