---
title: 'Unity短毛渲染及优化'
date: 2025-06-08
permalink: /posts/2025/0608/Fur/
tags:
  - Unity
  - Fur
  - Skinned Mesh
  - Optimization
---

## 前言
最近整理了下之前做的短毛渲染优化方案，整体思路就是基于Shell的短毛渲染，使用Unity的DrawProceduralIndirect来做绘制调用，再添加了基于距离做的蒙皮网格优化和Shell层数控制，把一项项暴力攒到一起组成了一个看起来尚可的解决方案，因此想把整体的思路与实现记录下来，方便日后用到的时候快速回忆，也方便其他人来做参照和更好的改进。本文主要内容是短毛的优化方案，渲染部分就仅仅是简单实现，想看炸裂毛发渲染效果的同学可以去看看其他大佬的实现。

## 短毛渲染及优化
本方案中短毛渲染使用的就是常规的基于Shell的方案，即重复绘制N次Mesh，每次沿着法线外扩一段距离，通过Clip来伪造出一根根毛发的效果。短毛部分的优化也是在Shell的基础上做的，也就是使用DrawProceduralIndirect一次绘制N个Mesh，通过InstanceID来在Shader中动态计算出每层的Offset，最后再添加上基于距离的动态层数，来组成整个短毛部分的解决方案。

### 短毛渲染的简单实现
首先在VS中根据每层毛发的Offset来重新计算一下顶点位置。
```hlsl
positionOS.xyz += normalOS * furOffset;
```

还可以在世界空间的顶点位置根据furOffset来给毛发加上卷曲和重力影响，表现出来的效果就是毛发卷曲和越靠近末端的位置越下垂。
```hlsl
float3 positionWS = TransformObjectToWorld(positionOS.xyz);
float shellOffset = pow(abs(furOffset), _LayerOffsetInfluence);
positionWS.xz += float2(sin(furOffset * _LayerCurvedFrequency), cos(furOffset * _LayerCurvedFrequency)) *  _LayerCurvedIntensity;
positionWS.y += _FurGravity * shellOffset;
output.positionWS = positionWS;
output.positionCS = TransformWorldToHClip(positionWS);
```

接着在FS中采样毛发样式图去控制Clip，根据Mask图屏蔽掉不该长毛的区域。
```hlsl
float2 mainTexUV = input.uv * _MainTex_ST.xy + _MainTex_ST.zw;
half3 mainTex = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, mainTexUV).rgb;

float2 furTexUV = input.uv * _FurTex_ST.xy + _FurTex_ST.zw;
half furTex = SAMPLE_TEXTURE2D(_FurTex, sampler_FurTex, furTexUV).r;

half maskTex = SAMPLE_TEXTURE2D(_MaskTex, sampler_MaskTex, input.uv).r;

half alphaCutout = pow(abs(furTex), max(_FurCut, 0.001) * input.offset);
clip(alphaCutout - input.offset - maskTex);

half3 col = CalculateLighting(input, mainTex.rgb) * _Color.rgb;
```

这样我们就简单实现了基于Shell的短毛渲染，接下来就可以开始进行优化工作。

### 短毛渲染优化
优化的核心思路就是使用DrawProceduralIndirect来一次绘制N层毛发的Mesh，进而就可以在Shader里通过InstanceID来计算furOffset。这样我们就需要手动获取蒙皮网格每帧动画计算后的顶点数据，传入到Shader中来计算渲染。我们暂时使用Unity的GetVertexBuffer来获取顶点数据的Buffer，后面我们会手动计算来对这部分进行优化。
```c#
SkinnedMeshRenderer smr = GetComponent<SkinnedMeshRenderer>();

GraphicsBuffer deformedDataBuffer = smr.GetVertexBuffer();
smr.sharedMesh.vertexBufferTarget |= GraphicsBuffer.Target.Raw;
int uvStreamID = smr.sharedMesh.GetVertexAttributeStream(VertexAttribute.TexCoord0);
GraphicsBuffer staticDataBuffer = smr.sharedMesh.GetVertexBuffer(uvStreamID);
stride = smr.sharedMesh.GetVertexBufferStride(0);
var uv0Offset = smr.sharedMesh.GetVertexAttributeOffset(VertexAttribute.TexCoord0);
var vertexColorOffset = smr.sharedMesh.GetVertexAttributeOffset(VertexAttribute.Color);
int uvStride = smr.sharedMesh.GetVertexBufferStride(uvStreamID);

GraphicsBuffer indexBuffer = smr.sharedMesh.GetIndexBuffer();

Material mat = smr.sharedMaterials[matIdx];
mat.SetBuffer("_DeformedData", deformedDataBuffer);
mat.SetBuffer("_StaticData", staticDataBuffer);
mat.SetInt("_Stride", stride);
mat.SetInt("_UVStride", uvStride);
mat.SetInt("_VertexColorOffset", vertexColorOffset);
mat.SetInt("_UV0Offset", uv0Offset);
```

对应Shader中即可根据vertexID来获取到每个顶点的数据。
```hlsl
ByteAddressBuffer _DeformedData;
ByteAddressBuffer _StaticData;
int _Stride;
int _UVStride;
int _VertexColorOffset;
int _UV0Offset;

float3 GetPosition(ByteAddressBuffer vBuffer, uint vid)
{
  int vidx = vid * _Stride;
  float3 data = asfloat(vBuffer.Load3(vidx));
  return data;
}

float3 GetNormal(ByteAddressBuffer vBuffer, uint vid)
{
  int vidx = vid * _Stride;
  float3 data = asfloat(vBuffer.Load3(vidx + 12)); //12 = position.xyz
  return data;
}

float4 GetTangent(ByteAddressBuffer vBuffer, uint vid)
{
  int vidx = vid * _Stride;
  float4 data = asfloat(vBuffer.Load4(vidx + 24));//24 = position.xyz + normal.xyz
  return data;
}

half4 GetVertexColor(ByteAddressBuffer vBuffer, uint vid)
{
  int vidx = vid * _UVStride;
  uint data = asuint(vBuffer.Load(vidx + _VertexColorOffset));
  uint4 color = uint4(data >> 24 & 0xFF, data >> 16 & 0xFF, data >> 8 & 0xFF, data & 0xFF);
  return half4(color) / 255.0;
}

float2 GetTexCoord0(ByteAddressBuffer vBuffer, uint vid)
{
  int vidx = vid * _UVStride;
  float2 data = asfloat(vBuffer.Load2(vidx + _UV0Offset));
  return data;
}

void Vertex(VertexInput input){
  float3 positionOS = GetPosition(_DeformedData, input.vertexID);
  float3 normalOS = GetNormal(_DeformedData, input.vertexID);
  float4 tangentOS = GetTangent(_DeformedData, input.vertexID);
  half4 vertexColor = GetVertexColor(_StaticData, input.vertexID).gbar;
  float2 texcoord = GetTexCoord0(_StaticData, input.vertexID);
  ...
}
```

接下来就是调用DrawProceduralIndirect来一次绘制N层毛发。
```c#
SkinnedMeshRenderer smr = GetComponent<SkinnedMeshRenderer>();
var submesh = smr.sharedMesh.GetSubMesh(matIdx);
int submeshStartIdx = submesh.indexStart;
int indexCount = submesh.indexCount;
MeshTopology meshTopology = submesh.topology;
Matrix4x4 localToWorldMatrix = bipPelvisTrans == null ? transform.localToWorldMatrix : bipPelvisTrans.localToWorldMatrix;
mat.SetFloat("_LayerCount", instanceCount);

ComputeBuffer argsBuffer = new ComputeBuffer(5, sizeof(uint), ComputeBufferType.IndirectArguments);
args[0] = (uint)indexCount;
args[1] = (uint)instanceCount;
args[2] = (uint)submeshStartIdx;
args[3] = (uint)skinnedMeshRenderer.sharedMesh.GetBaseVertex(furMeshIdx);
args[4] = (uint)0;
argsBuffer.SetData(args);

cmd.DrawProceduralIndirect(indexBuffer, localToWorldMatrix, furMaterial, 0, meshTopology, argsBuffer);
```

在Shader中即可通过InstanceID计算出furOffset。
```hlsl
half furOffset = instanceID / _LayerCount;
```

DrawProceduralIndirect中一次绘制的Shell层数其实不应该用固定的，因为随着模型离摄像机越来越远，毛发层数也应该变少，类似LOD，这样既不影响效果，又能提高性能。
```c#
var dis = Vector3.Distance(transform.position, cameraPos) + 0.01f;
int dynamicInstanceCount = instanceCount;
dynamicInstanceCount = Mathf.Clamp((int)(furLayerCount * (1 - dis / instanceCount)), 0, instanceCount);
furMaterial.SetFloat("_LayerCount", dynamicInstanceCount);

if (dynamicInstanceCount <= 0) return;

.....

cmd.DrawProceduralIndirect(indexBuffer, localToWorldMatrix, furMaterial, 0, meshTopology, argsBuffer);
```

还应该加上视锥体剔除等常规的优化策略，这里就不做代码演示了，都放到下面的蒙皮动画优化中去。

使用DrawProceduralIndirect貌似还会导致SH数据丢失，我们需要手动计算然后传入。
```c#
public static Vector4[] GetSHCoefficients(Vector3 pos, Renderer r)
{
  LightProbes.GetInterpolatedProbe(pos, r, out sh);
  var SHCoefficients = new Vector4[7];

  for (int iC = 0; iC < 3; iC++)
  {
    SHCoefficients[iC].x = sh[iC, 3];
    SHCoefficients[iC].y = sh[iC, 1];
    SHCoefficients[iC].z = sh[iC, 2];
    SHCoefficients[iC].w = sh[iC, 0] - sh[iC, 6];
  }

  for (int iC = 0; iC < 3; iC++)
  {
    SHCoefficients[iC + 3].x = sh[iC, 4];
    SHCoefficients[iC + 3].y = sh[iC, 5];
    SHCoefficients[iC + 3].z = 3.0f * sh[iC, 6];
    SHCoefficients[iC + 3].w = sh[iC, 7];
  }

  SHCoefficients[6].x = sh[0, 8];
  SHCoefficients[6].y = sh[1, 8];
  SHCoefficients[6].z = sh[2, 8];
  SHCoefficients[6].w = 1.0f;
  return SHCoefficients;
}

var SHCoefficients = LightProbeUtil.GetSHCoefficients(transform.position, skinnedMeshRenderer);
furMaterial.SetVectorArray("MySHCoefficients", SHCoefficients);
```

在Shader中也需要手动计算下SH。
```hlsl
half4 MySHCoefficients[7];

half3 MySampleSH9(half4 SHCoefficients[7], half3 N)
{
  half4 shAr = SHCoefficients[0];
  half4 shAg = SHCoefficients[1];
  half4 shAb = SHCoefficients[2];
  half4 shBr = SHCoefficients[3];
  half4 shBg = SHCoefficients[4];
  half4 shBb = SHCoefficients[5];
  half4 shCr = SHCoefficients[6];

  half3 res = SHEvalLinearL0L1(N, shAr, shAg, shAb);
  res += SHEvalLinearL2(N, shBr, shBg, shBb, shCr);

  return res;
}

half3 MySampleSH(half3 normalWS)
{
  return max(half3(0, 0, 0), MySampleSH9(MySHCoefficients, normalWS));
}
```

## 蒙皮动画优化
上文中我们使用Unity的GetVertexBuffer来获取动画计算后的顶点数据，我们还可以在这个阶段进行优化，在Unity中是不支持动画LOD功能的，动画LOD就是根据离摄像机距离远近来动态调整动画的质量和更新频率，通过动画LOD我们可以减少蒙皮计算带来的性能消耗，但会稍微影响动画的表现，这个部分得根据项目实际情况去做取舍。

这部分内容我在上一篇文章中有过粗略的描述，这里就粘贴之前的文章内容了，再针对之前没提到的细节做详细介绍。骨骼动画的蒙皮计算，我们对于每个SkinnedMesh去维护3个ComputeBuffer，分别为：
- vertexBuffer（存储每个顶点的数据：position、normal、tangent）
- skinInfluenceBuffer（存储每个顶点的蒙皮数据：权重、骨骼索引）
- boneMatrixBuffer（存储每根骨骼的变换矩阵）

每帧首先需要更新boneMatrixBuffer。
```c#
for (var i = 0; i < boneMatrices.Length; i++)
{
  boneMatrices[i] = rootBone.worldToLocalMatrix * bones[i].localToWorldMatrix * bindposes[i];
}
boneMatrixBuffer.SetData(boneMatrices);
```

接着就可以通过这三个ComputeBuffer计算出蒙皮后的顶点数据。
```hlsl
[numthreads(64, 1, 1)]
void main(uint3 threadID : SV_DispatchThreadID, StructuredBuffer<MeshVertex> inVertices, StructuredBuffer<SkinInfluence> inSkin, RWStructuredBuffer<MeshVertex> outVertices, StructuredBuffer<float4x4> inMatrices)
{
  const uint t = threadID.x;
  if (t >= g_VertCount) return;

  SkinInfluence si = inSkin[t];
  const MeshVertex vert = inVertices[t];
  float3 vPos = vert.pos.xyz;
  float3 vNorm = vert.norm.xyz;
  float3 vTang = vert.tang.xyz;

  const float4x4 blendedMatrix = inMatrices[si.index0] * si.weight0 +
    inMatrices[si.index1] * si.weight1 +
    inMatrices[si.index2] * si.weight2 +
    inMatrices[si.index3] * si.weight3;

  outVertices[t].pos = mul(blendedMatrix, float4(vPos, 1)).xyz;
  outVertices[t].norm = mul(blendedMatrix, float4(vNorm, 0)).xyz;
  outVertices[t].tang = float4(mul(blendedMatrix, float4(vTang, 0)).xyz, vert.tang.w);
}
```

下面就是之前没提到的优化细节，首先动态选择蒙皮计算的质量（4 Bones、2 Bones和1 Bone）和更新频率，来减少计算消耗，其中动态切换Bones就是Unity内置的做法。
```c#
// 根据距离设置更新频率、骨骼质量
if (distance < nearDistanceThreshold)
{
  quality = ESkinBonesForVert.FourBones;
  return nearSkipFrameCount;
}
else if (distance < midDistanceThreshold)
{
  quality = ESkinBonesForVert.TwoBones;
  return midSkipFrameCount;
}
else
{
  quality = ESkinBonesForVert.OneBone;
  skipCalculateBlendshape = true;
  return farSkipFrameCount;
}

//根据更新频率决定是否进行蒙皮计算
if (!renderer.needUpdate) return;
//根据距离得到蒙皮计算质量（4 Bones、2 Bones和1 Bone）
int skinningKernel = GetKernelByBonesForVert((int)renderer.quality, true, true);
//执行蒙皮计算
renderer.UpdateSkinning(cmd, settings.skinningComputeShader, skinningKernel);
```

对应Compute Shader实现。
```hlsl
struct SkinInfluence
{
  #if SKIN_BONESFORVERT <= 1
    int index0;
  #elif SKIN_BONESFORVERT == 2
    float weight0, weight1;
    int index0, index1;
  #elif SKIN_BONESFORVERT == 4
    float weight0, weight1, weight2, weight3;
    int index0, index1, index2, index3;
  #endif
};

[numthreads(64, 1, 1)]
void main(uint3 threadID : SV_DispatchThreadID, StructuredBuffer<MeshVertex> inVertices, StructuredBuffer<SkinInfluence> inSkin, RWStructuredBuffer<MeshVertex> outVertices, StructuredBuffer<float4x4> inMatrices)
{
  ...
  #if SKIN_BONESFORVERT == 0
   ...
  #elif SKIN_BONESFORVERT == 1
   const float4x4 blendedMatrix = inMatrices[si.index0];
  #elif SKIN_BONESFORVERT == 2
    const float4x4 blendedMatrix = inMatrices[si.index0] * si.weight0 +
    inMatrices[si.index1] * si.weight1;
  #elif SKIN_BONESFORVERT == 4
    const float4x4 blendedMatrix = inMatrices[si.index0] * si.weight0 +
    inMatrices[si.index1] * si.weight1 +
    inMatrices[si.index2] * si.weight2 +
    inMatrices[si.index3] * si.weight3;
  #endif
}
```

还可以用视锥体剔除来不更新屏幕外的动画，这也是Unity内置的做法。
```c#
Bounds boundsWS = new Bounds(transform.TransformPoint(aabb.center), aabb.size);
// 物体在屏幕外不更新
if (!GeometryUtility.TestPlanesAABB(frustumPlanes, boundsWS))
{
  needUpdate = false;
}
```

接下来就可以使用我们手动计算的顶点数据来绘制毛发了，整体操作和上文中的实现一样。
```c#
properties.SetBuffer("outVertices", skinnedMeshVertexBuffer);
cmd.DrawProceduralIndirect(indexBuffer, localToWorldMatrix, furMaterial, 0, meshTopology, argsBuffer, 0, properties);
```

在Shader中就可以直接通过VertexID来获取到顶点数据。
```hlsl
struct MeshVertex
{
    float3 pos;
    float3 norm;
    float4 tang;
};

//蒙皮计算后的顶点数据
StructuredBuffer<MeshVertex> outVertices;

MeshVertex vertex = outVertices[vertexID];
float3 positionOS = vertex.pos;
float3 normalOS = vertex.norm;
float4 tangentOS = vertex.tang;
```

通过对动画的优化，我们可以做到同时跑大量的带毛的蒙皮网格动画，请无视恐龙没有眼珠子这个问题，实在懒得改了( •̥́ ˍ •̀)，恐龙动画也是随机播放的，看着像是挂了一大片....（也可能算我摆了个白垩纪陨石砸地球的场景？）

这个实现还可以做更多的优化，比如同屏带动画模型过多时，可以动态分配每帧的更新列表，来控制每帧蒙皮计算的上限，这里就不做实现了，留给感兴趣的读者自行尝试。

# 总结
这个方案适用于有很多带毛带动画物体的游戏场景，比如全是小动物的游戏。通过蒙皮动画的优化和短毛层数的优化，可以做到同屏跑大数量的带毛带动画模型。如果有大佬发现了文章中的错误，希望能够联系我及时更正~