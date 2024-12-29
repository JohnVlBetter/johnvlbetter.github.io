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

第一种方案适合动画数量较少且已知的情况，并不通用，因此我们不做考虑。第二种方案里，我们需要实时的计算每个Meshlet的包围盒和Normal Cone数据，包围盒和Normal Cone可以通过Compute Shader去遍历Meshlet的全部顶点来计算，比如我们使用64个三角形和64个顶点的Meshlet，对应Meshlet包围盒的计算方法简化版如下：
```hlsl
[numthreads(64, 1, 1)]
void ComputeMeshletBounds(uint3 threadID : SV_DispatchThreadID)
{
  //计算顶点数据
  ......

  float3 boundMin = float3(0,0,0);
  float3 boundMax = float3(0,0,0);
  boundMin = WaveActiveMin(vertexPos);
  boundMax = WaveActiveMax(vertexPos);
  if (WaveIsFirstLane())//只需要第一个Lane写入
  {
      meshlet.min = boundMin;
      meshlet.max = boundMax;
      _MeshletBuffer[meshletID] = meshlet;
  }

  //其余操作
  ......
}
```

上述代码中用到了Wave Intrinsics中的几个内置函数，Wave Intrinsics是D3D12在Shader Model 6.0中引入的，用于控制Warp中Lane（可近似看做Thread）之间共享和同步数据，比如WaveIsFirstLane就能判断当前Lane是否是Wave中的第一个Active Lane（即索引最小的那个），WaveActiveMin和WaveActiveMax能获取到当前Wave中所有Active的Min值和Max值。我们可以通过Dispatch顶点数个Thread，将Group内Thread的数量（即Wave内Lane的数量）设置为Meshlet的顶点数，通过Wave函数来计算当前Meshlet的包围盒。Normal Cone的计算也可以类比实现出来，本文中就不做详细解释，感兴趣的读者可尝试自行实现。

想要在Unity中使用Wave Intrinsics的话，首先需要在Compute Shader中添加"#pragma use_dxc"这行代码，并且使用DX12启动Unity编辑器（可在Hub中设置项目的命令函参数，添加"-force-d3d12"）。这种实现难以兼容其他平台，对于不支持Wave Intrinsics的设备，我们可以考虑使用InterlockedMin和InterlockedMax来替换WaveActiveMin和WaveActiveMax，在计算完当前顶点数据后通过GroupMemoryBarrierWithGroupSync函数同步Thread之间的数据，再通过InterlockedMin和InterlockedMax来计算包围盒数据。对应的包围盒数据存储入groupshared的数组中去，简化后的代码如下所示：
```hlsl
groupshared int minMax[6];
[numthreads(64, 1, 1)]
void ComputeMeshletBounds(uint3 threadID : SV_DispatchThreadID)
{
  //计算顶点数据
  ......

  //Group同步
  GroupMemoryBarrierWithGroupSync();

  //InterlockedMin和InterlockedMax只支持uint，所以要做转化，factor值是随便写的
  uint factor = 100000;
  InterlockedMin(minMax[0], factor * oPos.x);
  InterlockedMin(minMax[1], factor * oPos.y);
  InterlockedMin(minMax[2], factor * oPos.z);
  InterlockedMax(minMax[3], factor * oPos.x);
  InterlockedMax(minMax[4], factor * oPos.y);
  InterlockedMax(minMax[5], factor * oPos.z);
  if (threadIndexInGroup == 0)//只需要第一个线程写入
  {
      meshlet.min = float3(minMax[0] / 100000.0, minMax[1] / 100000.0, minMax[2] / 100000.0);
      meshlet.max = float3(minMax[3] / 100000.0, minMax[4] / 100000.0, minMax[5] / 100000.0);
      _MeshletBuffer[meshletID] = meshlet;
  }

  //其余操作
  ......
}
```

大量的InterlockedMin和InterlockedMax会大大降低性能，因此在低端设备或者移动端上其实可以不计算，而是使用Instance的包围盒数据替代，示例伪代码如下：
```hlsl
...
InstanceData data = _InstanceDataBuffer[instanceID];
meshlet.min = data.min;
meshlet.max = data.max;
_MeshletBuffer[meshletID] = meshlet;
```

UE5.5中支持了Nanite Skeletal Mesh，我翻了一下对应的源码，发现UE5.5中就是类似的实现，可能是为了规避某些问题，UE5.5 Nanite里对应部分的源码也放一下：
```hlsl
if ((PrimitiveData.Flags & PRIMITIVE_SCENE_DATA_FLAG_SKINNED_MESH) != 0)
{
	// TODO: Nanite-Skinning - Fun hack to temporarily "fix" broken cluster culling and VSM
	// Set the cluster bounds for skinned meshes equal to the skinned instance local bounds
	// for clusters and also node hierarchy slices. This satisfies the constraint that all
	// clusters in a node hierarchy have bounds fully enclosed in the parent bounds (monotonic).
	// Note: We do not touch the bounding sphere in Bounds because that would break actual
	// LOD decimation of the Nanite mesh. Instead we leave these in the offline computed ref-pose
	// so that we get reasonable "small enough to draw" calculations driving the actual LOD.
	// This is not a proper solution, as it hurts culling rate, and also causes VSM to touch far
	// more pages than necessary. But it's decent in the short term during R&D on a proper calculation.
	Bounds.BoxExtent = InstanceData.LocalBoundsExtent;
	Bounds.BoxCenter = InstanceData.LocalBoundsCenter;
}
```

虚幻官方也标注了这不是一个合适的解决方案，因为会减少剔除率和其他问题，可能后续会有更好的实现来替换掉这部分代码。
现在我们通过运行时计算获取到了Meshlet的Bounds和Normal Cone数据，通过这些数据我们就可以实时的在GPU上进行剔除（背面剔除、贡献剔除、视椎体剔除和遮挡剔除），并且我们在蒙皮时进行了分帧计算和Animation LOD操作，通过这些处理，我们能够大幅度减少蒙皮网格带来的的性能消耗，甚至可以像心灵杀手2中一样，在场景中摆上大量高面数的骨骼驱动的植被，如下图所示。
[心灵杀手2Meshlet图]

# 总结
这个方案在Unity中实现了GPU Driven SkinnedMesh，包括了自定义的GPU Skinning、BlendShape计算以及实时计算Meshlet的包围盒和Normal Cone，而数据组织、GPU Culling以及渲染部分没有做涉及，对这几部分感兴趣的读者可以自行阅读知乎其他大佬的相关文章。此方案详细的代码实现在本人工作的项目工程中，没有办法放出来，但大体思路是一致的，感兴趣的读者可以尝试自己照着实现一下。如果有大佬发现了文章中的错误，希望能够联系我及时更正~

# 参考引用
- Erik Jansson的心灵杀手2技术分享 https://www.youtube.com/watch?v=EtX7WnFhxtQ&t=1008s
- Northlight技术展示 https://www.remedygames.com/article/how-northlight-makes-alan-wake-2-shine