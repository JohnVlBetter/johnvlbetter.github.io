---
title: 'Unity6 UI Toolkit 渲染部分解析'
date: 2024-12-09
permalink: /posts/2024/1209/UnityUIToolkit/
tags:
  - Unity
  - UI Toolkit
---

## UI Toolkit介绍
从Unity2021开始UI Toolkit就内置到了引擎中，和UGUI一样充当Unity的官方UI系统。UI Toolkit是基于XMAL和CSS进行制作的UI系统，能够以较UGUI更好的性能去绘制复杂UI,Unity目前建议使用UI Toolkit来制作编辑器UI（而非之前的IMGUI）和运行时UI，目前也是支持了与UGUI的混合使用、图文混排等功能，能够使用内置的UI Builder等工具来快速制作UI。但是目前UI Toolkit仍然不支持自定义Shader，难以制作特殊的UI效果，而且由于其不基于GameObject，也难以制作世界空间的UI。本文主要专注于讲解UI Toolkit中的渲染部分，其余编辑制作、事件系统、组件等不做涉及。

## UI Toolkit原理
首先我们先来比较下UGUI和UI Toolkit：
### UGUI
- UGUI会将Canvas下的UI Mesh合并，但是如果图集不同，就会多次提交DrawCall（因为需要提交不同的图集）。
- 变动UI界面和修改材质颜色等会触发Rebuild（Layout Rebuild和Graphic Rebuild）和Rebatch，消耗大量时间。
- 支持自定义的UI Shader，能制作特殊的UI显示效果。
  
### UI Toolkit
- UI Toolkit中的UI使用了一个Uber Shader（Internal-UIRDefault）进行绘制，这个Shader内部能支持八张纹理，也就是说只要一批UI内使用了不超过8张纹理（字体图集或者纹理图集），就可以将这些UI合并成一个DrawCall。
- UI Toolkit通过_ShaderInfoTex存储了UI的位置和裁剪等信息，这样只需要在数据变动时修改_ShaderInfoTex贴图即可。并且UI Toolkit的底层中维护了一个GPU Buffer来存储UI Mesh的顶点和索引（VB和IB），因此当界面变化时，UI Toolkit可以通过Offset去只更新变化了的顶点和索引数据。
- 不支持自定义的UI Shader（不理解。。。）。

可以看出UI Toolkit的渲染性能远远强于传统的UGUI，虽然限制也是不少，但是对于大量图文混排的界面来说，几乎就能以1个DrawCall绘制完毕。接下来我们通过Unity发布的UI Toolkit Demo工程Dragon Crashers来进行演示和讲解。

![Dragon Crashers](../images/DragonCrashers.jpeg "Dragon Crashers")

### Internal-UIRDefault Shader
我们首先来看Internal-UIRDefault这个Shader的源码，由于内置到引擎后的UI Toolkit查看不到对应Shader了，我就到Github找到了一份历史版本先凑合对照，对应仓库为 https://github.com/needle-mirror/com.unity.ui ，大家可以自行clone阅读源码。
首先Shader中对于是否支持Shader Model 3.5（后文简称SM3.5）做了区分，支持SM3.5的设备会从_ShaderInfoTex中读取ClipRect和Transform数据，不支持SM3.5的设备使用CBuffer传递读取数据。并且支持SM3.5的设备会使用nointerpolation和8个图集槽位，不支持的设备只能使用4个图集槽位，区分Shader Model部分代码如下所示。

```hlsl
#if SHADER_TARGET >= 30
    #define UIE_SHADER_INFO_IN_VS 1
#else
    #define UIE_SHADER_INFO_IN_VS 0
#endif // SHADER_TARGET >= 30

#if SHADER_TARGET < 35
    #define UIE_TEXTURE_SLOT_COUNT 4
    #define UIE_FLAT_OPTIM
#else
    #define UIE_TEXTURE_SLOT_COUNT 8
    #define UIE_FLAT_OPTIM nointerpolation
#endif // SHADER_TARGET >= 35

sampler2D _FontTex;
float4 _FontTex_TexelSize;
float _FontTexSDFScale;

sampler2D _GradientSettingsTex;
float4 _GradientSettingsTex_TexelSize;

sampler2D _ShaderInfoTex;
float4 _ShaderInfoTex_TexelSize;

float4 _TextureInfo[UIE_TEXTURE_SLOT_COUNT]; // X id YZ texelSize

sampler2D _Texture0;
float4 _Texture0_ST;
//省略1-3

#if UIE_TEXTURE_SLOT_COUNT == 8
sampler2D _Texture4;
float4 _Texture4_ST;
//省略5-7
#endif

float4 _PixelClipInvView; // xy in clip space, zw inverse in view space
float4 _ScreenClipRect; // In clip space

#if !UIE_SHADER_INFO_IN_VS

CBUFFER_START(UITransforms)
float4 _Transforms[UIE_SKIN_ELEMS_COUNT_MAX_CONSTANTS * 3];
CBUFFER_END

CBUFFER_START(UIClipRects)
float4 _ClipRects[UIE_SKIN_ELEMS_COUNT_MAX_CONSTANTS];
CBUFFER_END

#endif // !UIE_SHADER_INFO_IN_VS
```
但目前版本的Shader中_FontTex已经合进了8个图集槽位中，具体实现已经与上边源码有所差异。对于_Texture0-8这8个图集，UI Toolkit使用代码二分去采样对应槽位的纹理，具体代码如下所示。
```hlsl
// index: integer between [0..UIE_TEXTURE_SLOT_COUNT]
float4 SampleTextureSlot(half index, float2 uv)
{
    float4 result;

#if UIE_TEXTURE_SLOT_COUNT > 4
    if (index < 4)
    {
#endif
        if (index < 2)
        {
            if (index < 1)
            {
                result = tex2D(_Texture0, uv);
            }
            else
            {
                result = tex2D(_Texture1, uv);
            }
        }
        else // index >= 2
        {
            if (index < 3)
            {
                result = tex2D(_Texture2, uv);
            }
            else
            {
                result = tex2D(_Texture3, uv);
            }
        }
#if UIE_TEXTURE_SLOT_COUNT > 4
    }
    else // index >= 4
    {
        if (index < 6)
        {
            if (index < 5)
            {
                result = tex2D(_Texture4, uv);
            }
            else
            {
                result = tex2D(_Texture5, uv);
            }
        }
        else // index >= 6
        {
            if (index < 7)
            {
                result = tex2D(_Texture6, uv);
            }
            else
            {
                result = tex2D(_Texture7, uv);
            }
        }
    }
#endif

    return result;
}
```
首先我们先来看VS部分：
- 使用uie_vert_load_payload方法来解出UI的Transform数据（也就是uie_toWorldMat），在uie_vert_load_payload函数中，对于支持SM3.5的设备直接读取_ShaderInfoTex来构建uie_toWorldMat，不支持的设备读取_Transforms这个CBuffer来构建uie_toWorldMat。
- 通过顶点传入的flags数据获取当前是否是下面的情况之一：
  - isEdgeNoShrinkY
  - isEdgeNoShrinkX
  - isEdge
  - isSvgGradients
  - isDynamic
  - isTextured
  - isText
  - isSolid
- 转换当前顶点到ClipPos
- 记录typeTexSettings信息，xyz分别对应：
  - UI渲染类型（solid、text、textured和svg）
  - 图集槽位索引(通过FindTextureSlot方法在_TextureInfo中计算得到对应的Texture Index)
  - settingIndex(用于采样_GradientSettingsTex，来计算SVG渐变)
- 从_ShaderInfoTex中计算出clipRect数据

对应VS部分源码如下所示。
```hlsl
v2f uie_std_vert(appdata_t v, out float4 clipSpacePos)
{
    v2f OUT;
    UNITY_SETUP_INSTANCE_ID(v);
    UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(OUT);

    uie_vert_load_payload(v);
    float flags = round(v.flags.x*255.0f); // Must round for MacGL VM
    // Keep the descending order for GLES2
    const float isEdgeNoShrinkY  = TestForValue(7.0, flags);
    const float isEdgeNoShrinkX  = TestForValue(6.0, flags);
    const float isEdge           = TestForValue(5.0, flags);
    const float isSvgGradients   = TestForValue(4.0, flags);
    const float isDynamic        = TestForValue(3.0, flags);
    const float isTextured       = TestForValue(2.0, flags);
    const float isText           = TestForValue(1.0, flags);
    const float isSolid = 1 - saturate(isText + isTextured + isDynamic + isSvgGradients);

    float2 viewOffset = float2(0, 0);
    if (isEdge == 1 || isEdgeNoShrinkX == 1 || isEdgeNoShrinkY == 1)
    {
        viewOffset = uie_get_border_offset(v.vertex, v.uv, 1, isEdgeNoShrinkX == 1, isEdgeNoShrinkY == 1);
    }

    v.vertex.xyz = mul(uie_toWorldMat, v.vertex);
    v.vertex.xy += viewOffset;

    OUT.uvXY.zw = v.vertex.xy;
    clipSpacePos = UnityObjectToClipPos(v.vertex);

    if (isText == 1 && _FontTexSDFScale == 0)
        clipSpacePos.xy = uie_snap_to_integer_pos(clipSpacePos.xy);

    OUT.clipPos.xy = clipSpacePos.xy / clipSpacePos.w;
    OUT.clipPos.zw = float2(0, v.flags.y);

    // 1 => solid, 2 => text, 3 => textured, 4 => svg
    half renderType = isSolid * 1 + isText * 2 + (isDynamic + isTextured) * 3 + isSvgGradients * 4;
    half textureSlot = FindTextureSlot(v.textureId);
    float settingIndex = v.opacityPageSettingIndex.z*(255.0f*255.0f) + v.opacityPageSettingIndex.w*255.0f;
    OUT.typeTexSettings = half3(renderType, textureSlot, settingIndex);

    OUT.uvXY.xy = v.uv;
    if (isDynamic == 1.0f)
        OUT.uvXY.xy *= _TextureInfo[textureSlot].yz;

    OUT.clipRectOpacityUVs = uie_std_vert_shader_info(v, OUT.color);
    OUT.textCoreUVs = uie_decode_shader_info_texel_pos(v.opacityPageSettingIndex.zw, v.ids.w, 3.0f);

#if UIE_SHADER_INFO_IN_VS
    OUT.clipRect = tex2Dlod(_ShaderInfoTex, float4(OUT.clipRectOpacityUVs.xy, 0, 0));
#endif // UIE_SHADER_INFO_IN_VS

    return OUT;
}
```