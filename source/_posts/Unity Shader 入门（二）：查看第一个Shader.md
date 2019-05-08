---
title: Unity Shader 入门（二）：查看第一个Shader
thumbnail: https://github.com/kurong00/blog/blob/master/thumbnail/shader2/shader2.png?raw=true
date: 2017/9/12
categories: Shader
---

## 查看第一个shader

上一节是理论知识的储备，如果你对细节部分感兴趣可以阅读更多的资料（Cg，HLSL，GLSL，OpenGl，DirectX的官方Doc等等），如果不求甚解的话，那我们就通过查看第一个shader来加深理解。我们在Unity中新建一个shader（Assets->Create->shader->standard surface shader）打开看发现里面已经有很多代码了。（版本Unity 2017.2.0f3）

```haxe
Shader "Custom/NewSurfaceShader" {
	Properties {
		_Color ("Color", Color) = (1,1,1,1)
		_MainTex ("Albedo (RGB)", 2D) = "white" {}
		_Glossiness ("Smoothness", Range(0,1)) = 0.5
		_Metallic ("Metallic", Range(0,1)) = 0.0
	}
	SubShader {
		Tags { "RenderType"="Opaque" }
		LOD 200
		
		CGPROGRAM
		// Physically based Standard lighting model, and enable shadows on all light types
		#pragma surface surf Standard fullforwardshadows

		// Use shader model 3.0 target, to get nicer looking lighting
		#pragma target 3.0

		sampler2D _MainTex;

		struct Input {
			float2 uv_MainTex;
		};

		half _Glossiness;
		half _Metallic;
		fixed4 _Color;

		// Add instancing support for this shader. You need to check 'Enable Instancing' on materials that use the shader.
		// See https://docs.unity3d.com/Manual/GPUInstancing.html for more information about instancing.
		// #pragma instancing_options assumeuniformscaling
		UNITY_INSTANCING_CBUFFER_START(Props)
			// put more per-instance properties here
		UNITY_INSTANCING_CBUFFER_END

		void surf (Input IN, inout SurfaceOutputStandard o) {
			// Albedo comes from a texture tinted by color
			fixed4 c = tex2D (_MainTex, IN.uv_MainTex) * _Color;
			o.Albedo = c.rgb;
			// Metallic and smoothness come from slider variables
			o.Metallic = _Metallic;
			o.Smoothness = _Glossiness;
			o.Alpha = c.a;
		}
		ENDCG
	}
	FallBack "Diffuse"
}
```
emmm.......即使你有编程的基础也可能看的一头雾水，不过没关系，我们现在来一个个拆解这段代码。

## 逐行代码查看
我们打开刚刚新建的shader代码，开始逐行来看吧：

```haxe
Shader "Custom/NewSurfaceShader" {
	Properties {
		_Color ("Color", Color) = (1,1,1,1)
		_MainTex ("Albedo (RGB)", 2D) = "white" {}
		_Glossiness ("Smoothness", Range(0,1)) = 0.5
		_Metallic ("Metallic", Range(0,1)) = 0.0
	}
```
### 首先第一行
```haxe 
Shader "Custom/NewSurfaceShader"
```
Custom是自定义的shader默认的文件夹，如果你自己想要归类shader文件夹，就可以定义二级标题比如"Custom/MyShader/NewSurfaceShader"，这样NewSurfaceShader就归类在MyShader下啦。

### 接着一段代码块
```haxe 
Properties {
		_Color ("Color", Color) = (1,1,1,1)
		_MainTex ("Albedo (RGB)", 2D) = "white" {}
		_Glossiness ("Smoothness", Range(0,1)) = 0.5
		_Metallic ("Metallic", Range(0,1)) = 0.0
	}
```
这里是shader的属性部分：属性的格式写作如下 
```
_Name("Display Name", type) = defaultValue[{options}]
``` 

- _Name : 变量名，在之后的Shader代码中都用这个名字来获取该属性的内容
- Display Name : 显示名，在Unity Inspector上显示的名字
- type : 类型，可能的type所表示的内容有以下几种：  
- defaultValue : 上面类型的默认值
- options : 对于2D，或者Cube贴图有关，默认写一个空白的{}，例如下表

类型|说明|语法
:--:|:--:|:--:|
Float|浮点数|_MyFloat("Float",Float)=3.5
Int|整型数|_MyInt("Int",Int)=1
Range(min,max)|一个介于最小值和最大值之间的浮点数|_MyRange("Range",Range(0.0,1.0))=0.5
Color|RGBA（红绿蓝和透明度）四个量来定义的颜色|_MyColor("Color",Color)=(1,1,1,1)
2D|贴图信息|_My2D("2D",2D)="white"{}
Cube|立方纹理，由6张关联的2D贴图合在一起|_MyCube("Cube",Cube)="bump"{}
Vector|四维数|_MyVector("Vector",Vector)=(1,2,3,1)

### SubShader内部构造

#### Tags：键值对
```haxe
Tags { "RenderType"="Opaque" }
```
tags用来告诉渲染器：何时以及怎样渲染这个对象。

标签名称|标签说明|例子
:--:|:--:|:--:|
Queue|控制渲染顺序，保证不透明物体在透明物体之前渲染|Tags {"Queue"="Transparent"}
RenderType|对着色器分类，例如这是渲染透明的，这是渲染不透明的|Tags {"RenderType"="Opaque"}
DisableBatching|是否对该SubShader进行批处理|Tags {"DisableBatching"="True"}
ForceNoShadowCasting|该SubShader是否会投射阴影|Tags {"ForceNoShadowCasting"="True"}
IgnoreProjector|该SubShader是否会Project影响，常用于半透明物体|Tags {"IgnoreProjector"="True"}
CanUseSpriteAtlas|该SubShader用于Sprites时，要设置成false|Tags {"CanUseSpriteAtlas"="False"}
PreviewType|Inspector preview上默认是圆形预设，可以改为plane或者skybox|Tags {"PreviewType"="Plane"}

#### LOD：Level of Detail
```haxe
LOD 200
```
这个数值决定了我们能用什么样的Shader。当设定的LOD小于SubShader所指定的LOD时，这个SubShader就不可以用了。Unity自定义了一组LOD的数值，我们在实现自己的Shader的时候可以参考来设定自己的LOD数值，以便控制渲染。

LOD名称|数值|
:--:|:--:|
VertexLit及其系列|100
Decal, Reflective VertexLit|150
Diffuse |200
Diffuse Detail, Reflective Bumped Unlit, Reflective Bumped VertexLit|250
Bumped, Specular|300
Parallax|500
Parallax Specular|600

#### 实现代码
```haxe 
CGPROGRAM
		// Physically based Standard lighting model, and enable shadows on all light types
		#pragma surface surf Standard fullforwardshadows
		// Use shader model 3.0 target, to get nicer looking lighting
		#pragma target 3.0
		sampler2D _MainTex;
		struct Input {
			float2 uv_MainTex;
		};
		half _Glossiness;
		half _Metallic;
		fixed4 _Color;
		// Add instancing support for this shader. You need to check 'Enable Instancing' on materials that use the shader.
		// See https://docs.unity3d.com/Manual/GPUInstancing.html for more information about instancing.
		// #pragma instancing_options assumeuniformscaling
		UNITY_INSTANCING_CBUFFER_START(Props)
			// put more per-instance properties here
		UNITY_INSTANCING_CBUFFER_END
		void surf (Input IN, inout SurfaceOutputStandard o) {
			// Albedo comes from a texture tinted by color
			fixed4 c = tex2D (_MainTex, IN.uv_MainTex) * _Color;
			o.Albedo = c.rgb;
			// Metallic and smoothness come from slider variables
			o.Metallic = _Metallic;
			o.Smoothness = _Glossiness;
			o.Alpha = c.a;
		}
		ENDCG
```
终于到了最重要的部分，首先`CGPROGRAM`和`ENDCG`成对出现,表示中间包裹的是一段Cg程序，接着是一个编译指令：` #pragma surface surf Standard fullforwardshadows`
意味着我们要写一个表面Shader，并指定了光照模型，具体语法是
```haxe
#pragma surface surfaceFunction lightModel [optionalparams]
```
- surface ： 声明的是一个表面着色器
- surfaceFunction ： 着色器代码的方法的名字
- lightModel ： 使用的光照模型。

对应上面的编译指令：我们声明了一个表面着色器，实际的代码在 surf 函数中（在下面的代码能找到该函数），使用 Standard 作为光照模型。

接下来是  `sampler2D _MainTex;` 我们知道在CG中，Texture（贴图）简单来说就是一块内存存储的，使用了RGBA通道，且每个通道8bits，的数据。而具体地想知道像素与坐标的对应关系，以及获取这些数据，一次一次去计算内存地址或者偏移显然不可行，因此可以通过sampler2D来对贴图进行操作。一言以蔽之就是，sampler2D是GLSL中的2D贴图的类型，相应的，还有sampler1D，sampler3D，samplerCube等等格式。

然后的重点是：为什么在这里需要一句对_MainTex的声明？首先之前我们已经在Properties里声明过它是贴图了（`_MainTex ("Albedo (RGB)", 2D) = "white" {}`）。我们用来实例的这个shader其实是由两个相对独立的块组成的，外层的属性声明，回滚等等是Unity可以直接使用和编译的ShaderLab；而现在我们是在CGPROGRAM...ENDCG这样一个代码块中，这是一段CG程序。对于这段CG程序，要想访问在Properties中所定义的变量的话，必须使用**和之前变量相同的名字进行声明**。因此`sampler2D _MainTex;`做的事情就是再次声明并链接了_MainTex，使得接下来的CG程序能够使用这个变量。后面的`half _Glossiness;` `half _Metallic;`  `fixed4 _Color;`都是同样的道理。回到原来的地方，下一句是:
```haxe
struct Input {
    float2 uv_MainTex;
};
```
如果你有编程的经历，那么结构体应该很熟悉了，这一段我们结合下面的surf一起来说
```haxe
void surf (Input IN, inout SurfaceOutputStandard o) {
			// Albedo comes from a texture tinted by color
			fixed4 c = tex2D (_MainTex, IN.uv_MainTex) * _Color;
			o.Albedo = c.rgb;
			// Metallic and smoothness come from slider variables
			o.Metallic = _Metallic;
			o.Smoothness = _Glossiness;
			o.Alpha = c.a;
		}
```
刚才提到的`#pragma surface surf Standard fullforwardshadows`里面surf 函数就是对应的上面一段。我们看函数头输入的参数有Input IN。这个Input就对应上面的结构体。我们可以把所需要参与计算的数据都放到这个Input结构中，再传入surf函数使用；SurfaceOutputStandard是已经定义好了里面类型输出结构。作为输入的结构体**必须命名为Input**，这个结构体中定义了一个float2的变量，emmmm···你可能会感到奇怪float后面跟着数字，这是什么意思呢？其实float和vec都可以在之后加入一个2到4的数字，来表示被打包在一起的2到4个同类型数。比如：`float4 color;` `float3 multipliedColor = color.rgb * coordinate.x;`之类的。

在这个例子里，我们声明了一个叫做`uv_MainTex`的包含两个浮点数的变量。UV mapping的作用是将一个2D贴图上的点按照一定规则映射到3D模型上，在CG程序中，我们有这样的约定，在一个贴图变量之前加上uv两个字母，就代表提取它的uv值。我们之后就可以在surf程序中直接通过访问uv_MainTex来取得这张贴图当前需要计算的点的坐标值。接下来我们详细看surf内部的操作：
```haxe
fixed4 c = tex2D (_MainTex, IN.uv_MainTex) * _Color;
```
这里用到了一个tex2d函数，这是CG程序中用来在一张贴图中对一个点进行采样的方法，返回一个float4。这个例子中用刚刚得到的float4*_Color使得这个贴图经过和颜色混合。
```haxe
o.Albedo = c.rgb;
```
将其颜色的rbg值赋予了输出的像素颜色
```haxe
o.Metallic = _Metallic;
o.Smoothness = _Glossiness;
```
都是用到上头Properties中我们定义的变量来赋值材质中的Metallic and smoothness

```haxe
o.Alpha = c.a;
```
将a值赋予透明度。至此surf介绍完毕，这个例子中shader最重要的部分就是以上这些啦！

### 最后一步

```haxe
FallBack "Diffuse"
```
当所有上面的SubShader都不可以在目标平台上运行时，Unity就会调用这个shader，当然你也可以关闭这个选项，那就意味着如果没有显卡可以跑上面的shader，那我们就不管它啦!



 
  

## 结语

这是最简单最简单的shader，看到这里的你应该可以了解一些简单的shader了，可以去Unity的Surface Shader Exampless上查看一些基础shader的编写内容，下一篇我们会开始第一个shader的编写。