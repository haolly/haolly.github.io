---
layout: post
title: 溶解效果在游戏中的实现
tags: [gamedev]

---

游戏中有个需求，在 NPC 死掉之后不是立马就消失，而是有一个慢慢溶解消失的过程，这种效果以前在其他游戏中也见过，小时候玩小霸王游戏机的时候好像也见过这种效果。

那么如何实现呢？
## 实现原理
对一张噪声纹理进行采样，然后与控制溶解程度的阈值比较，如果小于这个阈值，使用 `clip` 函数把其对应的像素剪裁掉，这些被剪裁的部分就是被“溶解” 的部分。溶解区域的边缘可以通过不同的颜色与原来的颜色混合后产生。
溶解效果通常用来实现物品的燃烧效果，也可以用于实现游戏中NPC 死亡后的消失效果。

## Shader 实现

``` HLSL
Shader "Unlit/Dissolve"
{
	Properties
	{
		_BurnAmount("Burn Amount", Range(0.0, 1.0)) = 0.0
		_LineWidth("Burn Line Width", Range(0.0, 0.2)) = 0.1
		//the primary texture and normal map
		_MainTex("Base (RGB)", 2D) = "white" {}
		_BumpMap("Normal Map", 2D) = "bump" {}
		//control the color of dissolve edge
		_BurnFirstColor("Burn First Color", Color) = (1,0,0,1)
		_BurnSecondColor("Burn Second Color", Color) = (1,0,0,1)
		//noise texture, the keypoint !!
		_BurnMap("Burn Map", 2D) = "white" {}
	}
	SubShader
	{
		Tags {"RenderType" = "Opaque" "Queue" = "Geometry"}
		Pass
		{
			Tags {"LightMode" = "ForwardBase"}
			//溶解会导致裸露模型的内部构造，所以比较关闭面片剔除，从而模型的正面和背面都被渲染
			Cull Off

			CGPROGRAM
			#include "UnityCG.cginc"
			#include "Lighting.cginc"
			#include "AutoLight.cginc"
			#pragma target 3.0
			#pragma multi_compile_fwdbase
			#pragma vertex vert
			#pragma fragment frag

			float _BurnAmount;
			float _LineWidth;

			sampler2D _MainTex;
			sampler2D _BumpMap;
			sampler2D _BurnMap;

			fixed4 _BurnFirstColor;
			fixed4 _BurnSecondColor;

			float4 _MainTex_ST;
			float4 _BumpMap_ST;
			float4 _BurnMap_ST;

			struct v2f
			{
				float4 pos : SV_POSITION;
				float2 uvMainTex: TEXCOORD0;
				float2 uvBumpMap : TEXCOORD1;
				float2 uvBurnMap : TEXCOORD2;
				float3 lightDir :TEXCOORD3;
				float3 worldPos : TEXCOORD4;
				SHADOW_COORDS(5)
			};

			v2f vert(appdata_tan v)
			{
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);

				o.uvMainTex = TRANSFORM_TEX(v.texcoord, _MainTex);
				o.uvBumpMap = TRANSFORM_TEX(v.texcoord, _BumpMap);
				o.uvBurnMap = TRANSFORM_TEX(v.texcoord, _BurnMap);

				TANGENT_SPACE_ROTATION;
				//convert light from model to tangent space
				//TODO: why ? because bump map is in the tangent space, we will compute light in tangent space
				o.lightDir = mul(rotation, ObjSpaceLightDir(v.vertex)).xyz;
				o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
				TRANSFER_SHADOW(o);
				return o;

			}

			fixed4 frag(v2f i) : SV_TARGET
			{
				fixed3 burn = tex2D(_BurnMap, i.uvBurnMap).rgb;
				// if _BurnAmount is bigger than burn.r, this pixel is discard
				clip(burn.r - _BurnAmount);

				float3 tangentLightDir = normalize(i.lightDir);
				fixed3 tangentNormal = UnpackNormal(tex2D(_BumpMap, i.uvBumpMap));

				fixed3 albedo = tex2D(_MainTex, i.uvMainTex).rgb;

				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
				fixed3 diffuse = _LightColor0.rgb * albedo * saturate(dot(tangentNormal, tangentLightDir));

				fixed t = 1 - smoothstep(0.0, _LineWidth, burn.r - _BurnAmount);
				fixed3 burnColor = lerp(_BurnFirstColor, _BurnSecondColor, t);
				burnColor = pow(burnColor, 5);

				UNITY_LIGHT_ATTENUATION(atten, i, i.worldPos);
				fixed3 finalColor = lerp(ambient + diffuse * atten, burnColor, t* step(0.0001, _BurnAmount));
				return fixed4(finalColor, 1);
			}


			ENDCG
		}

		Pass
		{
			Tags {"LightMode" = "ShadowCaster"}
			CGPROGRAM
			#pragma vertex vert
			#pragma fragment frag
			#include "UnityCG.cginc"

			#pragma multi_compile_shadowcaster

			sampler2D _BurnMap;
			float4 _BurnMap_ST;
			float _BurnAmount;

			struct v2f
			{
				V2F_SHADOW_CASTER;
				float2 uvBurnMap : TEXCOORD1;
			};

			v2f vert(appdata_base v)
			{
				v2f o;
				TRANSFER_SHADOW_CASTER_NORMALOFFSET(o)
				o.uvBurnMap = TRANSFORM_TEX(v.texcoord, _BurnMap);
				return o;
			}
			fixed4 frag(v2f i) : SV_TARGET
			{
				fixed3 burn = tex2D(_BurnMap, i.uvBurnMap).rgb;
				clip(burn.r - _BurnAmount);
				SHADOW_CASTER_FRAGMENT(i);
			}


			ENDCG
		}
	}
	FallBack "Diffuse"
}
```


值得注意的是，需要定义额外的 Pass 来投射阴影，不然被溶解的部分还是会向其他物体投射阴影。

## 关于噪声纹理
噪声纹理一般都是程序生成的，但其实也可以使用其他纹理，我在网上找了几个燃烧的纹理试了下，效果还可以。
https://naldzgraphics.net/free-film-grain-textures/

## 动起来
可以看到，控制 `_BurnAmount` 参数可以达到控制溶解程度，所以，在C# 中通过 `material.SetFloat("_BurnAmount", value)` 的方式可以真正实现动态的溶解效果。

![dissolve-effect]({{ site.baseurl }}/assets/picture/201711/dissolve.gif)