---
title: 'Unity实现蒙皮网格的GPU Driven渲染'
date: 2024-12-23
permalink: /posts/2024/1223/GPUDrivenSkinnedMesh/
tags:
  - Unity
  - GPU Driven
  - Skinned Mesh
---

## 前言
最近项目中有支持SkinnedMesh GPU Driven渲染的需求，正好前段时间看到了Digital Dragons发布的Erik Jansson的演讲，其中就有对于心灵杀手2里蒙皮网格的GPU Driven渲染介绍，就整理了一下思路进行了这次尝试，将整体的实现流程记录下来。本篇文章不会具体讲GPU Driven管线的基础知识和相关搭建实现，如果不懂这些前置知识的话可以先去看看其他相关文章了解一下，这样可以更好的阅读本篇文章。

## GPU Skinning
Unity内置的GPU Skinning可以通过GetVertexBuffer获取到蒙皮计算之后的顶点数据，但是我们没办法很好地去控制调度，因此我们首先需要自行实现一下自定义的GPU Skinning方案。在Unity官网上我们可以下载到对应版本的计算Skinning及BlendShape的Compute Shader，接下来我们先给出一个简单的计算流程。

### Skinning部分
首先先来看骨骼动画的蒙皮计算，我们对于每个SkinnedMesh去维护3个ComputeBuffer，分别为：
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

接着就可以通过这三个ComputeBuffer计算出蒙皮后的顶点数据，下面放一下对应简化版的Compute Shader代码。
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
其实还有很多细节没有具体说，比如：
- 动态选择蒙皮计算的质量（4 Bones、2 Bones和1 Bone），来减少计算消耗。
- 区分是否需要计算Normal和Tangent数据，因为有些情况下并不需要法线或者切线数据。
- 根据距离或者重要程度，去做分帧计算蒙皮数据，可以做到某种程度的Animation LOD。

这些实现细节就不在这篇文章中过多描述了，感兴趣的读者可自行实现。

### BlendShape部分
BlendShape的计算需要维护inBlendShapeVertices这个Buffer，里面存储着所有BlendShape Frame的顶点变换信息（vertexIdx、deltaPos、deltaNormal和deltaTangent）。在每帧就可以遍历全部的BlendShape去通过inBlendShapeVertices来计算每个BlendShape的变换，简化版的Compute Shader代码如下。
```hlsl
[numthreads(64, 1, 1)]
void main(uint3 threadID : SV_DispatchThreadID, RWStructuredBuffer<MeshVertex> inOutMeshVertices, StructuredBuffer<BlendShapeVertex> inBlendShapeVertices)
{
    const uint t = threadID.x;
    if (t >= g_VertCount) return;

    BlendShapeVertex blendShapeVert = inBlendShapeVertices[t + g_FirstVert];
    const uint vertIndex = blendShapeVert.index;

    inOutMeshVertices[vertIndex].pos += blendShapeVert.pos * g_Weight;
    inOutMeshVertices[vertIndex].norm += blendShapeVert.norm * g_Weight;
    inOutMeshVertices[vertIndex].tang.xyz += blendShapeVert.tang * g_Weight;
}
```
GPU BlendShape计算也有一些细节没有具体描述，比如：
- 区分是否需要计算Normal和Tangent数据，因为有些情况下并不需要法线或者切线数据。
- BlendShape Frame的过渡计算。
- 根据距离动态减少BlendShape的计算数量。


通过上述的操作我们就能每帧完成SkinnedMesh的蒙皮计算和BlendShape计算，也可通过视椎体剔除、分帧计算、距离LOD（减少骨骼数或者BlendShape数）等方法优化性能。

## Meshlet
在GPU Driven管线中，蒙皮网格和静态网格最主要的区别就是，我们没办法预计算一个固定的Meshlet数据（包围盒和Normal Cone）。SkinnedMesh每帧都在随着动画变化，每个Meshlet对应的包围盒和Normal Cone也都随着变化，没法使用提前预计算好的数据去做GPU剔除。我们只有两个方案可以解决这个问题：
- 离线预生成，暴力穷举所有动画每一帧里每个Meshlet的包围盒和Normal Cone。
- 实时在每一帧里计算每个Meshlet的包围盒和Normal Cone。

第一种方案适合动画数量较少且已知的情况，并不通用，因此我们不做考虑。