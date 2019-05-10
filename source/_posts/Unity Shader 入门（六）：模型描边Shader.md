---
title: Unity Shader 入门（六）：模型描边Shader
date: 2018/2/8
thumbnail: https://github.com/kurong00/blog/blob/master/thumbnail/shader6/shader6.png?raw=true
categories: Shader

---

## 导语

前面几篇我们写了几个边缘发光的shader，另外一个类似功能的就是模型描边，和边缘发光不同的地方在于，描边是在原有模型的基础上，添加一圈的外框。

老规矩还是来看一下效果图：
<!-- more -->
![](https://github.com/kurong00/blog/blob/master/thumbnail/shader6/RimLighting.PNG?raw=true)

## 具体实现
说明一下这个Shader的具体实现：
### 实现原理：
Mesh Doubling (复制网格)： 
1. 需要一个单独的Pass来实现，重新绘制一个将所有表面都<font color =  #D37885 >沿着法线方向</font>延展模型，挤出一点点，然后将正面剪裁掉，只输出描边的颜色；
2. 第二个Pass就是一个正常着色的Pass

### 具体解说：
先放一段实现的代码：
```C++
Shader "Custom/Rim/RimLighting" {
	Properties{
		_MainColor("Main Color", Color) = (1,1,1,1)
		_OutlineCol("OutlineCol", Color) = (1,0,0,1)
		_OutlineFactor("OutlineFactor", Range(0,1)) = 0.1
		_MainTex("Base 2D", 2D) = "white"{}
	}
	SubShader
	{
		//描边使用两个Pass，第一个pass沿法线挤出一点，只输出描边的颜色
		Pass
		{
			Cull Front
			CGPROGRAM
			#include "UnityCG.cginc"
			fixed4 _OutlineCol;
			float _OutlineFactor;
			
			struct v2f
			{
				float4 pos : SV_POSITION;
			};
			
			v2f vert(appdata_full v)
			{
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				//将法线方向转换到视空间
				float3 vnormal = mul((float3x3)UNITY_MATRIX_IT_MV, v.normal);
				//将视空间法线xy坐标转化到投影空间，只有xy需要，z深度不需要了
				float2 offset = TransformViewToProjection(vnormal.xy);
				//在最终投影阶段输出进行偏移操作
				o.pos.xy += offset * _OutlineFactor;
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target
			{
				//这个Pass直接输出描边颜色
				return _OutlineCol;
			}
			
			//使用vert函数和frag函数
			#pragma vertex vert
			#pragma fragment frag
			ENDCG
		}
		
		//正常着色的Pass
		Pass
		{
			CGPROGRAM
			//引入头文件
			#include "Lighting.cginc"
			//使用vert函数和frag函数
			#pragma vertex vert
			#pragma fragment frag	
			//定义Properties中的变量
			fixed4 _MainColor;
			sampler2D _MainTex;
			//定义结构体：vertex shader阶段输出的内容
			struct v2f
			{
				float4 pos : SV_POSITION;
				float3 worldNormal : TEXCOORD0;
			};
			//定义顶点shader,参数直接使用appdata_base（包含position, noramal, texcoord）
			v2f vert(appdata_base v)
			{
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				//通过TRANSFORM_TEX宏转化纹理坐标，主要处理了Offset和Tiling的改变
				o.worldNormal = mul(v.normal, (float3x3)unity_WorldToObject);
				return o;
			}
			//定义片元shader
			fixed4 frag(v2f i) : SV_Target
			{
				//unity自身的diffuse也是带了环境光，这里我们也增加一下环境光
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * _MainColor.xyz;
				//归一化法线，即使在vert归一化也不行，从vert到frag阶段有差值处理，传入的法线方向并不是vertex shader直接传出的
				fixed3 worldNormal = normalize(i.worldNormal);
				//把光照方向归一化
				fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
				//根据半兰伯特模型计算像素的光照信息
				fixed3 lambert = 0.5 * dot(worldNormal, worldLightDir) + 0.5;
				//最终输出颜色为lambert光强*材质diffuse颜色*光颜色
				fixed3 diffuse = lambert * _MainColor.xyz * _LightColor0.xyz + ambient;
				//进行纹理采样
				fixed4 color = _MainColor;
				color.rgb = color.rgb* diffuse;
				return fixed4(color);
			}
			ENDCG
		}
	}
	FallBack "Diffuse"
}
```
详细的实现，包含在注释之中了。

### 包含问题
但是这个实现方法有一个问题：**线条并不连续**，在平滑表面的表现尚可（球体，胶囊体等等），但是在锐利的表面上经常会出现断层（比如立方体等等）。还是利用Mesh Doubling (复制网格)的方法，但是不再简单只通过法线方向，而是：<font color =  #D37885 >不严格地按照表面沿着法线的方向延展, 而是在标准化的点位置和法线方向之间取一个恰当的参数来做插值</font>。

## 更新方案

修改描边Pass的vert函数：

```haxe
v2f vert(appdata_full v)
		{
			v2f o;
			o.pos = UnityObjectToClipPos ( v.vertex );
			float3 vnormal1 = normalize ( v.vertex.xyz );
			//将法线方向转换到视空间
			float3 vnormal2 = mul((float3x3)UNITY_MATRIX_IT_MV, v.normal);
			vnormal1 = lerp ( vnormal1, vnormal2, _Factor );
			vnormal1 = mul ( ( float3x3 ) UNITY_MATRIX_IT_MV, vnormal1);
			float2 offset = TransformViewToProjection (vnormal1.xy );
			offset = normalize ( offset );
			float dist = distance ( mul ( UNITY_MATRIX_M, v.vertex ), _WorldSpaceCameraPos );
			o.pos.xy += offset *_OutlineFactor;
			return o;
		}
```
其中的_Factor就是用来计算差值的参数，这个可以根据自己调节`lerp ( vnormal1, vnormal2, _Factor )`

效果是：![](https://github.com/kurong00/blog/blob/master/thumbnail/shader6/RimLightingFix.PNG?raw=true)

最后上一个完整的修复过的Shader方案：

```haxe
Shader "Custom/Rim/RimLightingFix" {
	Properties{
		_MainColor("Main Color", Color) = (1,1,1,1)
		_OutlineCol("OutlineCol", Color) = (1,0,0,1)
		_OutlineFactor("OutlineFactor", Range(0,1)) = 0.1
		_MainTex("Base 2D", 2D) = "white"{}
		_Factor("Control Factor",Range(0,1)) = 0.1 
	}
	SubShader
	{
		//描边使用两个Pass，第一个pass沿法线挤出一点，只输出描边的颜色
		Pass{
		Cull Front
		CGPROGRAM
		#include "UnityCG.cginc"
		//使用vert函数和frag函数
		#pragma vertex vert
		#pragma fragment frag
		fixed4 _OutlineCol;
		float _OutlineFactor;
		float _Factor;	
		struct v2f
		{
			float4 pos : SV_POSITION;
		};

		v2f vert(appdata_full v)
		{
			v2f o;
			o.pos = UnityObjectToClipPos ( v.vertex );
			float3 vnormal1 = normalize ( v.vertex.xyz );
			//将法线方向转换到视空间
			float3 vnormal2 = mul((float3x3)UNITY_MATRIX_IT_MV, v.normal);
			vnormal1 = lerp ( vnormal1, vnormal2, _Factor );
			vnormal1 = mul ( ( float3x3 ) UNITY_MATRIX_IT_MV, vnormal1);
			float2 offset = TransformViewToProjection (vnormal1.xy );
			offset = normalize ( offset );
			float dist = distance ( mul ( UNITY_MATRIX_M, v.vertex ), _WorldSpaceCameraPos );
			o.pos.xy += offset *_OutlineFactor;
			return o;
		}

		fixed4 frag(v2f i) : SV_Target
		{
			//这个Pass直接输出描边颜色
			return _OutlineCol;
		}
		ENDCG
		}

		//正常着色的Pass
		Pass
		{
			CGPROGRAM
			//引入头文件
			#include "Lighting.cginc"
			//使用vert函数和frag函数
			#pragma vertex vert
			#pragma fragment frag	
			//定义Properties中的变量
			fixed4 _MainColor;
			sampler2D _MainTex;
			//定义结构体：vertex shader阶段输出的内容
			struct v2f
			{
				float4 pos : SV_POSITION;
				float3 worldNormal : TEXCOORD0;
			};
			//定义顶点shader,参数直接使用appdata_base（包含position, noramal, texcoord）
			v2f vert(appdata_base v)
			{
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				//通过TRANSFORM_TEX宏转化纹理坐标，主要处理了Offset和Tiling的改变
				o.worldNormal = mul(v.normal, (float3x3)unity_WorldToObject);
				return o;
			}

			//定义片元shader
			fixed4 frag(v2f i) : SV_Target
			{
				//unity自身的diffuse也是带了环境光，这里我们也增加一下环境光
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * _MainColor.xyz;
				//归一化法线，即使在vert归一化也不行，从vert到frag阶段有差值处理，传入的法线方向并不是vertex shader直接传出的
				fixed3 worldNormal = normalize(i.worldNormal);
				//把光照方向归一化
				fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
				//根据半兰伯特模型计算像素的光照信息
				fixed3 lambert = 0.5 * dot(worldNormal, worldLightDir) + 0.5;
				//最终输出颜色为lambert光强*材质diffuse颜色*光颜色
				fixed3 diffuse = lambert * _MainColor.xyz * _LightColor0.xyz + ambient;
				//进行纹理采样
				fixed4 color = _MainColor;
				color.rgb = color.rgb* diffuse;
				return fixed4(color);
			}
			ENDCG
		}
	}
	FallBack "Diffuse"
}
```
## 结语
描边常用于一些漫画风格的游戏场景中，能够在复杂的场景中突出被绘制的物体。