---
title: 'Unity6 GPU Resident Drawer 浅析'
date: 2024-11-18
permalink: /posts/2012/08/blog-post-4/
tags:
  - Unity
  - GPU Resident Drawer
  - GPU Driven
---

Unity6最新推出了一项名为GPU Resident Drawer的技术，能够大幅度提升物体渲染效率。网络上目前也少有对GPU Resident Drawer的原理讲解，于是在看了其源码之后，我准备分享下我的一些理解。本文基于Unity 6000.0.4f1c1版本，CoreRP 17.0.3版本代码进行分析，可能后续版本的代码会有所改动。

GPU Resident Drawer基于Unity原本的Batch Render Group API，是BRG的一个上层封装，同时也与DOTS紧密关联。按照官方文档中的描述，它优化了CPU耗时，但也可能进而提高gpu的性能。因为需要提交给GPU的绘制调用更少。简而言之就是Unity终于推出了官方的GPU Driven管线，支持GPU Culling，但其粒度较大，还停留在Mesh级别，而非Meshlet/Cluster。Unity官方还是想复用自己原本就有的DOTS和BRG，在运行时组织数据，将MeshRenderer的数据重新编排组织成BRG Batch，将Renderer的数据（Matrix、Bounds等）上传GPU，然后每帧做剔除（视椎体剔除、光源阴影剔除、遮挡剔除），将剔除的结果分Batch提交渲染。

GPU Resident Drawer在当前版本也有其限制，官方文档中写到GPU Resident Drawer仅适用于以下情况：
1.必须开启Forward+渲染。
2.平台必须支持Compute Shader，不支持OpenGL ES。
3.仅Mesh Renderer组件的GameObject。
其实还有其他的一些限制Unity没有详细说明，比如：
1.Shader必须支持DOTS Instancing。
2.必须开启RenderGraph。
3.使用GPU Resident Drawer时，LightMap实现变成TextureArray索引方式获取。
GPU Resident Drawer需要在管线设置中手动打开，首先需要在Settings > Graphics中将BatchRendererGroup Variants设置为Keep All，然后需要在管线设置中打开SRP Batcher，并且设置GPU Resident Drawer为Instanced Drawing。在正确打开GPU Resident Drawer后，可以在Frame Debugger中看到通过GPU Resident Drawer绘制的Draw Call变成了RenderLoop.DrawSRPBatcher下的Hybrid Batch Group，如下图所示。

[Frame Debugger Hybrid Batch Group截图占位]

GPU Resident Drawer较传统SRP Batcher进行了优化:
1.在传统SRP Batcher中，SRP Batch是根据Mesh Renderer的顺序进行合批的，Renderer之间会因为数据不同而打断合批（比如材质、Keywords不同等），而GPU Resident Drawer将场景中的Renderer通过remapping合为更少的Batch，能获得更少的批次。
2.在传统SRP Batcher中，上传到GPU Memory的Buffer是per object的large buffer，是AOS的组织方式，而GPU Resident Drawer是DOTS的SOA的组织方式，将相同的数据连续存储，也就能得到更高的性能。
3.在传统SRP Batcher中组织Batch只在主线程中单线程执行，性能较GPU Resident Drawer差很多。

[大量截图占位(用广州站技术分享ppt截图)]

接下来我们来分析GPU Resident Drawer的剔除源码，首先是CPU端的摄像机视椎体剔除和光源阴影的剔除。

对于渲染相机和产生阴影的光源，GPU Resident Drawer中会对其每个裁剪平面进行组织，每四个Plane会打成一个Pack，在源码的FrustumPlanes脚本下可以找到，下面是其部分核心代码：
```c#
internal struct PlanePacket4
{
    public float4 nx;
    public float4 ny;
    public float4 nz;
    public float4 d;
    // Store absolute values of plane normals to avoid recalculating per instance
    public float4 nxAbs;
    public float4 nyAbs;
    public float4 nzAbs;
}
```

[Pack4图占位]

为什么GPU Resident Drawer要将其每四个Plane包起来呢？主要是为了能利用Burst的SIMD去加快裁剪计算，利用SIMD去一次计算四个平面的裁剪结果，具体源码在FrustumPlanes的ComputeSplitVisibilityMask中可以找到，其部分核心代码如下：
```c#
uint splitVisibilityMask = 0;
int packetBase = 0;
int splitCount = splitInfos.Length;
for (int splitIndex = 0; splitIndex < splitCount; ++splitIndex)
{
    SplitInfo splitInfo = splitInfos[splitIndex];
    bool4 isCulled = new bool4(false);
    for (int i = 0; i < splitInfo.packetCount; ++i)
    {
        PlanePacket4 p = planePackets[packetBase + i];
        float4 distances = p.nx*cx + p.ny*cy + p.nz*cz + p.d;
        float4 radii = p.nxAbs*ex + p.nyAbs*ey + p.nzAbs*ez;
        isCulled = isCulled | (distances + radii < float4.zero);
    }
    if (!math.any(isCulled))
        splitVisibilityMask |= 1U << splitIndex;
    packetBase += splitInfo.packetCount;
}
```

对于产生阴影的光源一样会有对应的裁剪，GPU Resident Drawer对于每个caster都会计算其shadowLength，具体代码如下：
```c#
internal static float DistanceUntilCylinderFullyCrossesPlane(
    float3 cylinderCenter,
    float3 cylinderDirection,
    float cylinderRadius,
    Plane plane)
{
    float cosEpsilon = 0.001f; // clamp the cosine of glancing angles
    // compute the distance until the center intersects the plane
    float cosTheta = math.max(math.abs(math.dot(plane.normal, cylinderDirection)), cosEpsilon);
    float heightAbovePlane = math.dot(plane.normal, cylinderCenter) + plane.distance;
    float centerDistanceToPlane = heightAbovePlane / cosTheta;
    // compute the additional distance until the edge of the cylinder intersects the plane
    float sinTheta = math.sqrt(math.max(1.0f - cosTheta * cosTheta, 0.0f));
    float edgeDistanceToPlane = cylinderRadius * sinTheta / cosTheta;
    return centerDistanceToPlane + edgeDistanceToPlane;
}
```
这个shadowLength则用来计算所有阴影可能覆盖到的在每个Split下的receiver，具体的裁剪计算代码如下:
```c#
uint splitVisibilityMask = 0;
int splitCount = splitInfos.Length;
for (int splitIndex = 0; splitIndex < splitCount; ++splitIndex)
{
    SplitInfo splitInfo = splitInfos[splitIndex];
    float3 receiverCenterLightSpace = splitInfo.receiverSphereLightSpace.xyz;
    float receiverRadius = splitInfo.receiverSphereLightSpace.w;
    float3 receiverToCasterLightSpace = casterCenterLightSpace - receiverCenterLightSpace;
    // compute the light space z coordinate where the caster sphere and receiver sphere just intersect
    float sphereIntersectionMaxDistance = casterRadius + receiverRadius;
    float zSqAtSphereIntersection = math.lengthsq(sphereIntersectionMaxDistance) - math.lengthsq(receiverToCasterLightSpace.xy);
    // if this is negative, the spheres do not overlap as circles in the XY plane, so cull the caster
    if (zSqAtSphereIntersection < 0.0f)
        continue;
    // if the caster is outside of the receiver sphere in the light direction, it cannot cast a shadow on it, so cull it
    if (receiverToCasterLightSpace.z > 0.0f && math.lengthsq(receiverToCasterLightSpace.z) > zSqAtSphereIntersection)
        continue;
    // render the caster in this split
    splitVisibilityMask |= 1U << splitIndex;
    // culling assumes that shaders will always sample from the cascade with the lowest index,
    // so if the caster capsule is fully contained within the "core" sphere where only this split index is sampled,
    // then cull this caster from all the larger index splits (break from this loop)
    // (it is sufficient to test that only the capsule start and end spheres are within the "core" receiver sphere)
    float coreRadius = receiverRadius * splitInfo.cascadeBlendCullingFactor;
    float3 receiverToShadowEndLightSpace = receiverToCasterLightSpace + new float3(0.0f, 0.0f, shadowLength);
    float capsuleMaxDistance = coreRadius - casterRadius;
    float capsuleDistanceSq = math.max(math.lengthsq(receiverToCasterLightSpace), math.lengthsq(receiverToShadowEndLightSpace));
    if (capsuleMaxDistance > 0.0f && capsuleDistanceSq < math.lengthsq(capsuleMaxDistance))
        break;
}
```

GPU Resident Drawer对于每个Renderer分配8个bit去存储裁剪结果，每个bit对应着一个Split的裁剪结果，每8个Renderer的数据组成一个UInt64存储，最终的UInt64里也就存储着8个Renderer每Renderer对于8个Split的裁剪结果。
相关源码可以在InstanceCuller脚本中的AllocateBinsPerBatch下找到，部分代码如下：
```c#
int configMaskCount = (configCount + 63)/64;
var configUsedMasks = stackalloc UInt64[configMaskCount];
for (int i = 0; i < configMaskCount; ++i)
    configUsedMasks[i] = 0;
// loop over all instances within this batch
var drawBatch = drawBatches[batchIndex];
var instanceCount = drawBatch.instanceCount;
var instanceOffset = drawBatch.instanceOffset;
for (int i = 0; i < instanceCount; ++i)
{
    var rendererIndex = drawInstanceIndices[instanceOffset + i];
    bool isFlipped = IsInstanceFlipped(rendererIndex);
    int visibilityMask = (int)rendererVisibilityMasks[rendererIndex];
    if (visibilityMask == 0)
        continue;
    int configIndex = (int)(visibilityMask << 1) | (isFlipped ? 1 : 0);
    Assert.IsTrue(configIndex < configCount);
    visibleCountPerConfig[configIndex]++;
    configUsedMasks[configIndex >> 6] |= 1ul << (configIndex & 0x3f);
}
```

接下来我们接着分析遮挡剔除的源码，在Unity6中，BRG新增支持了两种DrawCommand，分别是Indirect和Procedural，这也就意味着在Unity6的GPU Resident Drawer中，可以支持到GPU Culling。GPU Resident Drawer中遮挡剔除采用了Two-Phase Occlusion Culling，这个方式是育碧在2015年的Siggraph上提出的，在Two-Phase Occlusion Culling中剔除分成了两个阶段，在第一阶段中使用上一帧的Hiz深度图去剔除一遍物体，然后渲染剔除通过的物体的深度，在第二阶段生成当前帧的Hiz深度图，使用这张Hiz深度图再剔除一遍第一阶段被剔除的物体，也就是保留上一帧中不可见但本帧中可见的物体，具体实例图如下所示。
[育碧ppt截图]

