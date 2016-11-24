---
layout:     post
title:      "UnityShader basic light model"
subtitle:   " "
date:       2016-11-16 12:00:00
author:     "Spark"
header-img: "img/post-bg-2015.jpg"
tags:
    - UnityShader
---

> 这是《UnityShader入门精要》冯乐乐著的学习笔记
# Unity Shader 基本光照模型

## 1. Shader基本框架

    "shadername"
    {
      Properties
      {
        //属性
      }
      SubShader
      {
        Pass
        {
          Tags{"//标签"}

          CGPROGRAM

          #pragma vertex vert
          #pragma fragment frag

          #include "//files"

          a2v struct
          {
            ...
          };

          v2f struct
          {
            ...
          };

          v2f vert(a2v v)
          {
            ...
          }

          fixed4 frag(v2f i):SV_Target
          {
            ...
          }

          ENDCG
        }
      }
      FallBack "//失败下回调的shader"
    }

## 2. 标准光照模型
>计算机图形学的第一定律：如果它看起来是对的，那么它就是对的。

从实际的物理情况模拟出一张图片需要考虑3点：

+ 光源
+ 物体的材质：影响光的吸收，反射（主要指高光反射），折射，散射（主要是漫反射）
+ 光线传入摄像机产生图像

标准光照模型把进入摄像机的光线分为4个部分：

+ 自发光
+ 高光反射
+ 漫反射
+ 环境光

> 标准光照模型只关心那些直接从光源发射出来照射到物体表面后经过物体表面一次反射直接进入摄像机的光线

这里的高光反射是一种经验模型，并不完全符合真实的物理世界，还记得那条第一原则吗？
计算高光发射需要的信息有：

+ 入射光线
+ 表面法线
+ 反射方向
+ 视角方向

### 2.1 高光反射
#### 2.1.1 Phong模型
![Phong reflect.png-55kB][1]
如何计算反射光线 **r** 请参考[如何计算反射光线](http://www.cnblogs.com/graphics/archive/2013/02/21/2920627.html)的连接。
$$ r=2(\hat n\cdot I)\hat n-\ I $$
>*注意：连接的入射光线和图片的方向相反，所以公式变号*

用Phone模型计算高光发射：
$$ c_{specular}=(c_{light}\cdot m_{specular})max(0,\widehat v\cdot r)^{m_{gloss}} $$

>*注意：带 $\land$* 上标的向量表示归一化之后的向量

#### 2.1.2 Blinn模型
![Blinn reflect.png-44.4kB][2]
Blinn模型提供了一个简单的方法来得到**类似的效果**，基本思想是避免计算反射向量$\hat r$，为此Blinn模型引入一个新的矢量$\hat h$他是通过$\hat v$和$\hat I$取平均后归一化得到：
$$ \hat h=\frac{\hat v+I}{|\hat v+I|} $$
然后使用$\hat n$和$\hat h$之间的夹角计算，Blinn模式公式如下：
$$
c_{specular}=(c_{light}\cdot m_{specular})max(0,\hat n \cdot \hat h)^{m_{gloss}}
$$
不能认为Blinn模型是对“正确的”Phong模型的近似，有些情况下Blinn模型更加符合实际情况。


在逐像素光照中，我们以每个像素为基础，得到它的法线然后进行光照模型计算。这种在片面之间对顶点法线进行插值的技术称为Phong着色也被称为Phong插值或法线插值着色技术，这不同于之前讲的Phone光照模型。

逐顶点光照称为高洛德着色，在每个顶点上计算光照然后渲染图元内进行插值计算最后输出颜色，由于顶点数目远小于像素数目因此顶点光照的计算了远小于逐像素的光照。但是由于逐顶点光照依赖线性插值来得到像素光照因此当模型有非线性光照计算顶点计算就会出问题，由于逐顶点光照会在渲染图元内内部对顶点进行颜色插值这会导致渲染图元内部的颜色总是暗于顶点处的最高颜色，这在某些情况下回产生明显的棱角现象。

标准光照模型：Phong模型，Blinn模型,（统称Blinn-Phong模型）

标准光照模型是一个经验模型，但能够比较好的模拟真实光照情况。不过有些物理现象无法表现，比如菲涅尔反射（Fresnel reflection）。Bliin-Phong模型是各向同性，也就是说当我们固定视角和光源方向旋转这个表面时反射不会发生变化。但有些表面具有各向异性发射性质，如金属拉丝，头发等。

### 2.2 漫反射
#### 2.2.1 兰伯特定律（Lambert's Law）
**特点：在平面某点漫反射光的强度与该反射点的法线向量和入射光角度的余弦值成正比。**
$$
c_{diffuse}=(c_{light} \cdot m_{diffuse})max(0,n \cdot I)
$$
#### 2.2.2 半兰伯特模型（Half Lambert）
兰伯特模型在光无法到达的地方是全黑的没有任何明暗变化，即使添加环境光得到非全黑的效果单仍然无法解决背光面明暗一样的结果。为此提出了半兰伯特模型。
半兰伯特模型是valve公司在《半条命》开发是提出来的技术，是在兰伯特模型上的简单修改，广义定义：
$$
c_{diffuse}=(c_{light} \cdot m_{diffuse})(\alpha (\hat n \cdot I)+\beta)
$$
可以看出半兰伯特模型没有使用$max$来防止产生负值而是进行了一个$\alpha$的缩放再进行了$\beta$的平移，绝大数情况$\alpha,\beta$为0.5。
$$
c_{diffuse}=(c_{light} \cdot m_{diffuse})(0.5 (\hat n \cdot I)+0.5)
$$
半兰伯特没有任何物理依据只是一种视觉加强技术。

## 3. Blinn-Phong模型unity shader代码示例
```
    Shader "Unity Shaders Book/Chapter 6/Blinn-Phong" {
  	  Properties
      {
    		_Diffuse ("Diffuse", Color) = (1, 1, 1, 1)
    		_Specular ("Specular", Color) = (1, 1, 1, 1)
    		_Gloss ("Gloss", Range(8.0, 256)) = 20
    	}
  	SubShader {
  		Pass { 
  			Tags { "LightMode"="ForwardBase" }  

  			CGPROGRAM

  			#pragma vertex vert
  			#pragma fragment frag

  			#include "Lighting.cginc"

  			fixed4 _Diffuse;
  			fixed4 _Specular;
  			float _Gloss;

  			struct a2v {
  				float4 vertex : POSITION;
  				float3 normal : NORMAL;
  			};

  			struct v2f {
  				float4 pos : SV_POSITION;
  				float3 worldNormal : TEXCOORD0;
  				float3 worldPos : TEXCOORD1;
  			};

  			v2f vert(a2v v) {
  				v2f o;
  				// Transform the vertex from object space to projection space
  				o.pos = mul(UNITY_MATRIX_MVP, v.vertex);

  				// Transform the normal fram object space to world space
  				o.worldNormal = mul(v.normal, (float3x3)_World2Object);

  				// Transform the vertex from object spacet to world space
  				o.worldPos = mul(_Object2World, v.vertex).xyz;

  				return o;
  			}

  			fixed4 frag(v2f i) : SV_Target {
  				// Get ambient term
  				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;

  				fixed3 worldNormal = normalize(i.worldNormal);
  				fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);

  				// Compute diffuse term
  				fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * max(0, dot(worldNormal, worldLightDir));

  				// Get the view direction in world space
  				fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);
  				// Get the half direction in world space
  				fixed3 halfDir = normalize(worldLightDir + viewDir);
  				// Compute specular term
  				fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal, halfDir)), _Gloss);

  				return fixed4(ambient + diffuse + specular, 1.0);
  			}

  			ENDCG
  		}
  	}
  	FallBack "Specular"
  }
```

关于shader结构以及说明请参考[Unity ShaderLab学习总结](http://www.ceeger.com/forum/read.php?tid=21154&page=1)。

下面对示例代码中部分语句进行说明：

+ `o.pos = mul(UNITY_MATRIX_MVP, v.vertex);`把顶点坐标转换到裁剪坐标
+ `o.worldNormal = mul(v.normal, (float3x3)_World2Object);`把顶点坐标法向量转换到世界坐标，因为后面的运算操作都在世界空间进行，只有在同一个坐标系下操作才有意义
+ `o.worldPos = mul(_Object2World, v.vertex).xyz;`把顶点坐标从模型空间转换到世界空间
+ `fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz`获取环境光方向
+  `fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * max(0, dot(worldNormal, worldLightDir));`计算漫反射
+  `fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);`获取视角方向（向量减法）
+  `fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal, halfDir)), _Gloss);`Blinn-Phong模型计算高光发射
+  `fixed4(ambient + diffuse + specular, 1.0);`合成所有计算的光线（不包含自发光）

## 4. UnityCG.cginc中常用函数

|函数名|描述|
|------|:----|
|float3 WorldSpaceViesDir(float4 v)|输入模型空间的的顶点位置，返回世界空间中从该点到摄像机的观察方向|
|flaot3 UnityFWorldSpaceViewDir(float4 v)|输入一个世界空间中顶点的位置，返回世界空间中从该点到摄像机的观察方向|
|flaot3 ObjectSpaceViewDir(float4 v)|输入模型空间中顶点的位置，返回模型空间中从该点到摄像机的观察方向|
|float3 WorldSpaceLightDir(float4 v)|**仅可用于前向渲染中**，输入模型空间中的顶点位置，返回世界空间中从该点到光源的光照方向，没有被归一化|
|float3 UnityWorldSpaceLightDir(float4 v)|**仅可用于前向渲染中**，输入世界空间中顶点位置，返回世界空间从该点到光源的方向，没有被归一化|
|float3 ObjectSpaceLightDir(float4 v)|**仅可用于前向渲染中**，输入一个模型空间中的顶点坐标，返回模型空间中从该点到光源的方向，没有被归一化|
|float3 UnityObjectToWorldNormal(float4 norm)|把法线方向从模型空间转换到世界空间|
|float3 UnityObjectToWorldDir(in float3 dir)|把方向矢量从模型空间转换到世界空间|
|float3 UnityWorldToObjectDir(float3 dir)|把方向矢量从世界空间转换到模型空间|

  [1]: http://static.zybuluo.com/sparkhou/qq7tb7dfhg7xqbz7bjwxpls3/Phong%20reflect.png
  [2]: http://static.zybuluo.com/sparkhou/sxyc2r4jzcv6fqqdtseg7sba/Blinn%20reflect.png