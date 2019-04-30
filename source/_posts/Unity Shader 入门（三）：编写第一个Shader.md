---
title: Unity Shader 入门（三）：编写第一个Shader
date: 2017/11/13
thumbnail: https://github.com/kurong00/blog/blob/master/thumbnail/shader3/shader3.png?raw=true
categories: Shader

---

## 编写第一个Shader

上一节我们学习了第一个简单的Shader，现在我们可以开始写第一个shader练练手了（搓搓手）。首先我们挑一个【边缘发光（水晶球）】的shader来写

首先来看一下效果图，如果你感兴趣的话就接下来看吧：![](https://github.com/kurong00/blog/blob/master/thumbnail/shader3/shader.PNG?raw=true)

### 实现原理：
根据物体表面法向量和视线向量的夹角来判断是否是物体的边缘部位。夹角越大（接近垂直）说明越接近物体边缘部分。<font color =  #D37885 >重点：向量点积运算。</font>

### 具体解说：
先放一段实现的代码：
```haxe
Shader "Custom/Rim//RimBump" {
	Properties{
		_Color("Main Color", Color) = (1,1,1,1)
		_SpecColor("Specular Color", Color) = (0.5, 0.5, 0.5, 1)
		_BumpMap("Normalmap", 2D) = "bump" {}
		_RimColor("Rim Color", Color) = (0.26,0.19,0.16,0.0)
		_RimPower("Rim Power", Range(0.5,8.0)) = 2.0
	}
		SubShader{
			Tags { "RenderType" = "Opaque" }
			LOD 400

		CGPROGRAM
		#pragma surface surf BlinnPhong
		#pragma target 3.0

		sampler2D _BumpMap;
		fixed4 _Color;
		float4 _RimColor;
		float _RimPower;

		struct Input {
			float2 uv_MainTex;
			float2 uv_BumpMap;
			float3 viewDir;
		};

		void surf(Input IN, inout SurfaceOutput o) {
			o.Albedo = _Color.rgb;
			o.Gloss = 1;
			o.Normal = UnpackNormal(tex2D(_BumpMap, IN.uv_BumpMap));
			half rim = 1 - saturate(dot(normalize(IN.viewDir), o.Normal));
			o.Emission = _RimColor.rgb * pow(rim, _RimPower);
		}
		ENDCG
		}
			FallBack "Diffuse"
}

```
如果你看过上一篇的Shader介绍你应该可以大致看懂上面的代码，我们就关键部分说明一下：
```haxe
void surf(Input IN, inout SurfaceOutput o) {
			o.Albedo = _Color.rgb;
			o.Gloss = 1;
			o.Normal = UnpackNormal(tex2D(_BumpMap, IN.uv_BumpMap));
			half rim = 1 - saturate(dot(normalize(IN.viewDir), o.Normal));
			o.Emission = _RimColor.rgb * pow(rim, _RimPower);
		}
```

- 首先这两句：

```haxe
o.Albedo = _Color.rgb;
o.Gloss = 1;
```
类比上一篇，o.Albedo 此时可以获得我们设置的颜色和贴图之间混合后的颜色，o.Gloss 我们将发光强度设置成1。
- 接下来是重点：
```haxe
o.Normal = UnpackNormal(tex2D(_BumpMap, IN.uv_BumpMap));
half rim = 1 - saturate(dot(normalize(IN.viewDir), o.Normal));
o.Emission = _RimColor.rgb * pow (rim, _RimStrength);
```
`UnpackNormal` 是定义在UnityCG.cginc文件中的方法（这个文件中包含了一系列常用的CG变量以及方法，在Unity安装路径中可以找到），`UnpackNormal`接受一个fixed4的输入，并将其转换为所对应的法线值（fixed3）。在解包得到这个值之后，将其赋给输出的Normal，接下来我们就可以来使用Normal值啦。
#### 有关法线贴图 
> 这一点归类于扩展阅读，如果你想知道`UnpackNormal`的原理可以继续查看，如果不的话就跳过这一段吧！ 
假设你想知道原理，那首先思考一个问题**为什么法线贴图看起来大多是蓝色的？** 
> - 实际上，我们通常见到的这种偏蓝色的法线纹理中，存储的是在Tangent Space中的顶点法线方向。那么，问题又来了，什么是Tangent Space。在Tangent Space中，坐标原点就是顶点的位置，其中z轴是该顶点本身的法线方向（N）。这样，另外两个坐标轴就是和该点相切的两条切线。这样的切线是有无数条，但模型一般会给定该顶点的一个tangent。（[给定的过程可以见这个链接](http://www.opengl-tutorial.org/intermediate-tutorials/tutorial-13-normal-mapping/)）
> - 通常我们所见的法线纹理是基于**原法线信息构建的坐标系**来构建出来的。那种偏蓝色的法线纹理其实就是存储在每个顶点各自的Tangent Space中**法线的扰动方向**。也就是说，如果一个顶点的法线方向不变，那么在它的Tangent Space中，新的normal值就是z轴方向，也就是说值为(0, 0, 1)。但这并不是法线纹理中存储的最终值：因为一个向量每个维度的取值范围在(-1, 1)，在法线贴图中被压缩在颜色的范围[0,1]中，所以需要转换：
> ``` haxe
> 颜色 = 0.5 * 法线 + 0.5;
> 线 = 2 * (颜色 - 0.5);    
> ```
>- 这样，之前的法线值(0, 0, 1)实际上对应了法线纹理中RGB的值为(0.5, 0.5, 1)，而这个颜色也就是法线纹理中那大片的蓝色。这些蓝色实际上说明**顶点的大部分法线是和模型本身法线一样**的，不需要改变。总结一下就是，`法线纹理的RGB通道存储了在每个顶点各自的Tangent Space中的法线方向的映射值。`
> - 下一个问题：Unity编辑器中加入一张发现贴图，编辑器都会提示把法线纹理的“Texture Type”设置成“Normal Map”，这是为什么呢？是因为这样的设置可以让Unity根据不同平台对纹理进行压缩，当需要法线信息时，再通过UnpackNormal函数对法线纹理进行正确的采样，即**将把颜色通道变成一个适合于实时法向映射的格式。**
> - 再下一个问题：压缩的内容又是什么呢？其实法线贴图只有两个通道是真正必不可少的，因为第三个通道的值可以用另外两个推导出来（法线是单位向量）法线（x,y,z）是一条单位向量。所以知道了x,y,z里的任意两个，剩下的那个就可以通过计算得出。所以我们就可以使用2个通道的图储存x,y,z里的两个值，将xyz里剩余的值省略，通过计算得出。而压缩后的法线贴图，大小只有原来的1/4左右，故可以使用更大或者更多的贴图来提升画面品质。

### 重点讲解

回到刚刚打断的地方，下面两句：
```haxe
half rim = 1 - saturate(dot(normalize(IN.viewDir), o.Normal));
o.Emission = _RimColor.rgb * pow (rim, _RimStrength);
```
- 首先我们看`normalize`函数：为了对向量进行归一化处理（这里传入IN.viewDir指的是：WorldSpace View Direction，也就是当前坐标的视角方向）。 `dot`函数：返回传入的两个参数的点积，`saturate`函数：判断传入的参数是否在0-1之间，如果小于0，返回 0；如果大于 1，返回1； 
- 接着第二句：`_RimColor.rgb * pow (rim, _RimStrength)`从_RimColor参数获取自发光颜色再和发光的强度混合，最终将颜色赋值给像素的Emission（发散颜色）
- 以上就是边缘发光效果的实现。

 


## 结语

下一次的shader我们将来写【半透明】的边缘发光效果。为此在下一篇我们将会梳理一下Unity shader透明效果的知识储配