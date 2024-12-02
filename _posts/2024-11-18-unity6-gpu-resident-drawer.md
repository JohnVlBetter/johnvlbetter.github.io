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

首先我们分析下GPU Resident Drawer中生成Hiz深度图的部分，在源码中称其为OccluderDepthPyramid，OccluderDepthPyramid在Setting里设置最多有8级Mip，降采样时会以CameraDepthAttachment的1/8来计算，假设以3840*2160分辨率来计算的话，CameraDepthAttachment会是3072*1728的大小，整体的Mip如下表所示。

| Width | Height |
| :---: | :----: |
|  384  |  324   |
|  192  |  162   |
|  96   |   81   |
|  48   |   41   |
|  24   |   21   |
|  12   |   11   |
|   6   |   6    |
|   3   |   3    |

最终生成的OccluderDepthPyramid如下图所示。

[OccluderDepthPyramid图]

具体生成的代码在InstanceOcclusionCuller的CreateFarDepthPyramid函数中可以看到，具体核心代码如下。
```c#
int mipCount = k_FirstDepthMipIndex + k_MaxOccluderMips;
for (int mipIndexBase = 0; mipIndexBase < mipCount - 1; mipIndexBase += 4)
{
    cmd.SetComputeTextureParam(cs, kernel, ShaderIDs._DstDepth, occluderHandles.occluderDepthPyramid);
    bool useSrc = (mipIndexBase == 0);
    SetKeyword(cmd, cs, srcKeyword, useSrc);
    SetKeyword(cmd, cs, srcIsArrayKeyword, useSrc && srcIsArray);
    SetKeyword(cmd, cs, srcIsMsaaKeyword, useSrc && srcIsMsaa);
    if (useSrc)
        cmd.SetComputeTextureParam(cs, kernel, ShaderIDs._SrcDepth, occluderParams.depthTexture);
    cb._MipCount = (uint)Math.Min(mipCount - 1 - mipIndexBase, 4);
    Vector2Int srcSize = Vector2Int.zero;
    for (int i = 0; i < 5; ++i)
    {
        Vector2Int offset = Vector2Int.zero;
        Vector2Int size = Vector2Int.zero;
        int mipIndex = mipIndexBase + i;
        if (mipIndex == 0)
        {
            size = occluderParams.depthSize;
        }
        else
        {
            int occMipIndex = mipIndex - k_FirstDepthMipIndex;
            if (0 <= occMipIndex && occMipIndex < k_MaxOccluderMips)
            {
                offset = occluderMipBounds[occMipIndex].offset;
                size = occluderMipBounds[occMipIndex].size;
            }
        }
        if (i == 0)
            srcSize = size;
        unsafe
        {
            cb._MipOffsetAndSize[4 * i + 0] = (uint)offset.x;
            cb._MipOffsetAndSize[4 * i + 1] = (uint)offset.y;
            cb._MipOffsetAndSize[4 * i + 2] = (uint)size.x;
            cb._MipOffsetAndSize[4 * i + 3] = (uint)size.y;
        }
    }
    constantBufferData[0] = cb;
    cmd.SetBufferData(constantBuffer, constantBufferData);
    cmd.SetComputeConstantBufferParam(cs, ShaderIDs.OccluderDepthPyramidConstants, constantBuffer, 0, constantBuffer.stride);
    cmd.DispatchCompute(cs, kernel, (srcSize.x + 15) / 16, (srcSize.y + 15) / 16, occluderSubviewUpdates.Length);
}
```

对应的ComputeShader代码是在OccluderDepthPyramidKernels下的OccluderDepthDownscale这个Kernel里，具体核心代码如下所示。
```hlsl
float p00 = LoadDepth(loadOffset + min(srcCoord + int2(0, 0), srcLimit), srcSliceIndex);
float p10 = LoadDepth(loadOffset + min(srcCoord + int2(1, 0), srcLimit), srcSliceIndex);
float p01 = LoadDepth(loadOffset + min(srcCoord + int2(0, 1), srcLimit), srcSliceIndex);
float p11 = LoadDepth(loadOffset + min(srcCoord + int2(1, 1), srcLimit), srcSliceIndex);

float farDepth = FarthestDepth(float4(p00, p10, p01, p11));
// write dst0
if (all(dstCoord1 < DestMipSize(1)))
    _DstDepth[DestMipOffset(1, dstSubviewIndex) + dstCoord1] = farDepth;
// merge towards thread 0 in subgroup size 4
if (2 <= _MipCount)
{
    SubgroupMergeDepths(threadID, 0, farDepth);
    SubgroupMergeDepths(threadID, 1, farDepth);
    if ((threadID & 0x3) == 0)
    {
        int2 dstCoord2 = dstCoord1 >> 1;
        if (all(dstCoord2 < DestMipSize(2)))
            _DstDepth[DestMipOffset(2, dstSubviewIndex) + dstCoord2] = farDepth;
    }
}
// merge towards thread 0 in subgroup size 16
if (3 <= _MipCount)
{
    SubgroupMergeDepths(threadID, 2, farDepth);
    SubgroupMergeDepths(threadID, 3, farDepth);
    if ((threadID & 0xf) == 0)
    {
        int2 dstCoord3 = dstCoord1 >> 2;
        if (all(dstCoord3 < DestMipSize(3)))
            _DstDepth[DestMipOffset(3, dstSubviewIndex) + dstCoord3] = farDepth;
    }
}
// merge to thread 0
if (4 <= _MipCount)
{
    SubgroupMergeDepths(threadID, 4, farDepth);
    SubgroupMergeDepths(threadID, 5, farDepth);
    if ((threadID & 0x3f) == 0)
    {
        int2 dstCoord4 = dstCoord1 >> 3;
        if (all(dstCoord4 < DestMipSize(4)))
            _DstDepth[DestMipOffset(4, dstSubviewIndex) + dstCoord4] = farDepth;
    }
}
```

由上述核心代码可以看出，在GPU Resident Drawer中生成OccluderDepthPyramid，每一次Dispatch能生成4级Mip，限制最大Mip为8级的情况下，加上第一次的1/8降采样，也只需三次Dispatch即可生成完整的OccluderDepthPyramid。
在每次Dispatch中，第一级Mip是取自上一级Mip的4个Depth采样的最远值，也就对应着Shader中的：
```hlsl
float farDepth = FarthestDepth(float4(p00, p10, p01, p11));
```
而后三级的Mip生成则极为巧妙，首先声明了groupshared的长度为32的float数组s_farDepth，利用这个线程间共享的数组来设置后三级Mip的Depth值，那么该如何利用这个共享数组呢，则是在代码中的SubgroupMergeDepths函数内，通过每次GroupMemoryBarrierWithGroupSync去做线程间的同步，SubgroupMergeDepths就可以在s_farDepth存下farDepth值，并且获取到每四个线程的最远Depth值，具体代码如下。
```hlsl
groupshared float s_farDepth[32];

void SubgroupMergeDepths(uint threadID : SV_GroupThreadID, uint bitIndex, inout float farDepth)
{
    uint highIndex = threadID >> (bitIndex + 1);
    uint lowIndex = threadID & ((1 << (bitIndex + 1)) - 1);

    if (lowIndex == (1 << bitIndex))
        s_farDepth[highIndex] = farDepth;
#if !defined(SHADER_API_WEBGPU)
    GroupMemoryBarrierWithGroupSync();
#endif

    if (lowIndex == 0)
        farDepth = FarthestDepth(farDepth, s_farDepth[highIndex]);
#if !defined(SHADER_API_WEBGPU)
    GroupMemoryBarrierWithGroupSync();
#endif
}
```
以第二级Mip生成代码为例，第0-3号线程的depth分别为d0、d1、d2和d3，第一次调用SubgroupMergeDepths后，每个线程的farDepth为：
- 线程0：farDepth = FarthestDepth(d0, s_farDepth[0]) = FarthestDepth(d0, d1)（因为s_farDepth[0] = d1）
- 线程1：farDepth = d1
- 线程2：farDepth = FarthestDepth(d2, s_farDepth[1]) = FarthestDepth(d2, d3)（因为s_farDepth[1] = d3）
- 线程3：farDepth = d3

第二次调用SubgroupMergeDepths后，每个线程的farDepth为：
- 线程0：farDepth = FarthestDepth(FarthestDepth(d0, d1), s_farDepth[0]) = FarthestDepth(FarthestDepth(d0,d1), FarthestDepth(d2, d3)) (也就是四个线程的最远Depth)
- 线程1：farDepth = d1
- 线程2：farDepth = FarthestDepth(d2, d3)
- 线程3：farDepth = d3

可以看到两次SubgroupMergeDepths之后，最终拿到的farDepth即为四个线程的最远Depth。在if判断“(threadID & 0x3) == 0”中，最终能通过的线程也就是第一个线程，所以第一个线程会将farDepth写入第二级Mip。每次Dispatch的线程组设置为64×1×1，所以最终第二级Mip会由第0、4、8、12、16、20、24、28、32、36、40、44、48、52、56、60号线程写入对应的每四个线程的farDepth。
```hlsl
if (2 <= _MipCount)
{
    SubgroupMergeDepths(threadID, 0, farDepth);
    SubgroupMergeDepths(threadID, 1, farDepth);
    if ((threadID & 0x3) == 0)
    {
        int2 dstCoord2 = dstCoord1 >> 1;
        if (all(dstCoord2 < DestMipSize(2)))
            _DstDepth[DestMipOffset(2, dstSubviewIndex) + dstCoord2] = farDepth;
    }
}
```
第三级和第四级Mip也是如此生成，第三级Mip会由第0、16、32、48号线程写入对应的每四个线程的farDepth，而第四级Mip只会由第0号线程写入每四个线程的farDepth，对应代码如下。
```hlsl
if (3 <= _MipCount)
{
    SubgroupMergeDepths(threadID, 2, farDepth);
    SubgroupMergeDepths(threadID, 3, farDepth);
    if ((threadID & 0xf) == 0)
    {
        int2 dstCoord3 = dstCoord1 >> 2;
        if (all(dstCoord3 < DestMipSize(3)))
            _DstDepth[DestMipOffset(3, dstSubviewIndex) + dstCoord3] = farDepth;
    }
}
if (4 <= _MipCount)
{
    SubgroupMergeDepths(threadID, 4, farDepth);
    SubgroupMergeDepths(threadID, 5, farDepth);
    if ((threadID & 0x3f) == 0)
    {
        int2 dstCoord4 = dstCoord1 >> 3;
        if (all(dstCoord4 < DestMipSize(4)))
            _DstDepth[DestMipOffset(4, dstSubviewIndex) + dstCoord4] = farDepth;
    }
}
```
经过三次Dispatch之后，即可最终得到Hiz深度图，也就是OccluderDepthPyramid。后续即可通过这张OccluderDepthPyramid进行逐instance的深度遮挡剔除。

GPU遮挡剔除的逻辑代码在InstanceCuller下的AddOcclusionCullingDispatch函数里，部分核心代码如下所示。
```c#
bool isFirstPass = (newBufferState == IndirectBufferContext.BufferState.AllInstancesOcclusionTested);
bool isSecondPass = (newBufferState == IndirectBufferContext.BufferState.OccludedInstancesReTested);
bool doCullInstances = (newBufferState != IndirectBufferContext.BufferState.Zeroed) && !doCopyInstances;
...
IndirectBufferAllocInfo allocInfo = m_IndirectStorage.GetAllocInfo(indirectContextIndex);
...
if (!allocInfo.IsEmpty())
{
    var cs = m_OcclusionTestShader.cs;
    var firstPassKeyword = new LocalKeyword(cs, "OCCLUSION_FIRST_PASS");
    var secondPassKeyword = new LocalKeyword(cs, "OCCLUSION_SECOND_PASS");
    OccluderContext.SetKeyword(cmd, cs, firstPassKeyword, isFirstPass);
    OccluderContext.SetKeyword(cmd, cs, secondPassKeyword, isSecondPass);
    m_ShaderVariables[0] = new InstanceOcclusionCullerShaderVariables
    {
       ...
    };
    cmd.SetBufferData(m_ConstantBuffer, m_ShaderVariables);
    cmd.SetComputeConstantBufferParam(cs, ShaderIDs.InstanceOcclusionCullerShaderVariables, m_ConstantBuffer, 0, m_ConstantBuffer.stride);
    occlusionCullingCommon.PrepareCulling(cmd, in occluderCtx, settings, subviewSettings, m_OcclusionTestShader, occlusionDebug);
    ...
    if (doCullInstances)
    {
        int kernel = m_CullInstancesKernel;
        cmd.SetComputeBufferParam(cs, kernel, ShaderIDs._DrawInfo, bufferHandles.drawInfoBuffer);
        cmd.SetComputeBufferParam(cs, kernel, ShaderIDs._InstanceInfo, bufferHandles.instanceInfoBuffer);
        cmd.SetComputeBufferParam(cs, kernel, ShaderIDs._DrawArgs, bufferHandles.argsBuffer);
        cmd.SetComputeBufferParam(cs, kernel, ShaderIDs._InstanceIndices, bufferHandles.instanceBuffer);
        cmd.SetComputeBufferParam(cs, kernel, ShaderIDs._InstanceDataBuffer, batchersContext.gpuInstanceDataBuffer);
        cmd.SetComputeBufferParam(cs, kernel, ShaderIDs._OcclusionDebugCounters, m_OcclusionEventDebugArray.CounterBuffer);
        if (isFirstPass || isSecondPass)
            OcclusionCullingCommon.SetDepthPyramid(cmd, m_OcclusionTestShader, kernel, occluderHandles);
        if (isSecondPass)
            cmd.DispatchCompute(cs, kernel, bufferHandles.argsBuffer, (uint)(GraphicsBuffer.IndirectDrawIndexedArgs.size * allocInfo.GetExtraDrawInfoSlotIndex()));
        else
            cmd.DispatchCompute(cs, kernel, (allocInfo.instanceCount + 63) / 64, 1, 1);
    }
}
```
可以看到，整个遮挡剔除过程分成了两个Pass：FirstPass和SecondPass，也就对应了上文中提到的两遍剔除，第二遍剔除是为了保留上一帧中不可见但本帧中可见的物体。对应的Compute Shader代码在InstanceOcclusionCullingKernels.compute里的CullInstances Kernel中，具体代码如下所示。
```hlsl
#pragma multi_compile _ OCCLUSION_FIRST_PASS OCCLUSION_SECOND_PASS

#if defined(OCCLUSION_FIRST_PASS) || defined(OCCLUSION_SECOND_PASS)
#define OCCLUSION_ANY_PASS
#endif

[numthreads(64,1,1)]
void CullInstances(uint instanceInfoOffset : SV_DispatchThreadID)
{
    uint instanceInfoCount = GetInstanceInfoCount();
    if (instanceInfoOffset < instanceInfoCount)
    {
        IndirectInstanceInfo instanceInfo = LoadInstanceInfo(instanceInfoOffset);
        uint drawOffset = instanceInfo.drawOffsetAndSplitMask >> 8;
        uint splitMask = instanceInfo.drawOffsetAndSplitMask & 0xff;

        // early out if none of these culling splits are visible
        // TODO: plumb through other state per draw command to filter here?
        if ((splitMask & _CullingSplitMask) == 0)
            return;

        bool isVisible = true;

#ifdef OCCLUSION_ANY_PASS
        int instanceID = instanceInfo.instanceIndexAndCrossFade & 0xffffff;
        SphereBound boundingSphere = LoadInstanceBoundingSphere(instanceID);

        bool isOccludedInAll = true;
        for (int testIndex = 0; testIndex < _OcclusionTestCount; ++testIndex)
        {
            // unpack the culling split index and subview index for this test
            int splitIndex = (_CullingSplitIndices >> (4 * testIndex)) & 0xf;
            int subviewIndex = (_OccluderSubviewIndices >> (4 * testIndex)) & 0xf;

            // skip if this draw call is not present in this split index
            if (((1 << splitIndex) & splitMask) == 0)
                continue;

            // occlusion test against the corresponding subview
            if (IsOcclusionVisible(boundingSphere, subviewIndex))
                isOccludedInAll = false;
        }
        isVisible = !isOccludedInAll;
        
#ifdef OCCLUSION_FIRST_PASS
        // if we failed the occlusion check, then add to the list for the second pass
        if (!isVisible)
        {
            uint writeIndex = 0;
            InterlockedAdd(_DrawArgs[DRAW_ARGS_INDEX_INSTANCE_COUNTER], 1, writeIndex);
            _InstanceInfo[_InstanceInfoAllocIndex + INSTANCE_INFO_OFFSET_SECOND_PASS(writeIndex)] = instanceInfo;
        }
#endif
#endif

        if (isVisible)
        {
            uint argsBase = DRAW_ARGS_INDEX(drawOffset);
            uint offsetWithinDraw = 0;
            InterlockedAdd(_DrawArgs[argsBase + 1], 1 << _InstanceMultiplierShift, offsetWithinDraw);   // IndirectDrawIndexedArgs.instanceCount
            offsetWithinDraw = offsetWithinDraw >> _InstanceMultiplierShift;

            IndirectDrawInfo drawInfo = LoadDrawInfo(drawOffset);
            uint writeIndex = drawInfo.firstInstanceGlobalIndex + offsetWithinDraw;
            _InstanceIndices.Store(writeIndex << 2, instanceInfo.instanceIndexAndCrossFade);
        }
    }
}
```
首先根据设置Keyword（OCCLUSION_FIRST_PASS和OCCLUSION_ANY_PASS）的不同，可以对应到上文的两个Pass（FirstPass和SecondPass），每次执行首先获取到当前instance的drawOffset和splitMask，splitMask即为上文中提到的cpu端剔除结果，通过“(splitMask & _CullingSplitMask) == 0”这条判断即可提前判断出当前instance是否对于所有的culling splits都完全不可见，以便提前退出计算。而后根据instanceID可以获取到instance的外包围球信息，接下来遍历每个subview，对于每个subview再做一次具体的split可见性判断，通过之后再通过IsOcclusionVisible计算遮挡可见性，最终全部都通过的instance即可标记为可见，进而执行后面的渲染数据录入。
从上文中可以看出，遮挡剔除的核心函数是IsOcclusionVisible，通过IsOcclusionVisible可以计算出当前instance的遮挡可见性，IsOcclusionVisible函数的详细代码位于OcclusionCullingCommon.hlsl下，部分核心代码如下所示。

```hlsl
TEXTURE2D(_OccluderDepthPyramid);
SAMPLER(s_linear_clamp_sampler);

bool IsOcclusionVisible(float3 frontCenterPosRWS, float2 centerPosNDC, float2 radialPosNDC, int subviewIndex)
{
    bool isVisible = true;
    float queryClosestDepth = ComputeNormalizedDeviceCoordinatesWithZ(frontCenterPosRWS, _ViewProjMatrix[subviewIndex]).z;
    bool isBehindCamera = dot(frontCenterPosRWS, _FacingDirWorldSpace[subviewIndex].xyz) >= 0.f;

    float2 centerCoordInTopMip = centerPosNDC * _DepthSizeInOccluderPixels.xy;
    float radiusInPixels = length((radialPosNDC - centerPosNDC) * _DepthSizeInOccluderPixels.xy);

    // log2 of the radius in pixels for the gather4 mip level
    int mipLevel = 0;
    float mipPartUnused = frexp(radiusInPixels, mipLevel);
    mipLevel = max(mipLevel + 1, 0);
    if (mipLevel < OCCLUSIONCULLINGCOMMONCONFIG_MAX_OCCLUDER_MIPS && !isBehindCamera)
    {
        // scale our coordinate to this mip
        float2 centerCoordInChosenMip = ldexp(centerCoordInTopMip, -mipLevel);
        int4 mipBounds = _OccluderMipBounds[mipLevel];
        mipBounds.y += subviewIndex * _OccluderMipLayoutSizeY;

        if ((_OcclusionTestDebugFlags & OCCLUSIONTESTDEBUGFLAG_ALWAYS_PASS) == 0)
        {
            // gather4 occluder depths to cover this radius
            float2 gatherUv = (float2(mipBounds.xy) + clamp(centerCoordInChosenMip, .5f, float2(mipBounds.zw) - .5f)) * _OccluderDepthPyramidSize.zw;
            float4 gatherDepths = GATHER_TEXTURE2D(_OccluderDepthPyramid, s_linear_clamp_sampler, gatherUv);
            float occluderDepth = FarthestDepth(gatherDepths);
            isVisible = IsVisibleAfterOcclusion(occluderDepth, queryClosestDepth);
        }

    }

    return isVisible;
}
```

IsOcclusionVisible的参数是boundingSphere通过CalculateBoundingObjectData计算得到，CalculateBoundingObjectData的详细代码如下所示。
```hlsl
BoundingObjectData CalculateBoundingObjectData(SphereBound boundingSphere,
    float4x4 viewProjMatrix,
    float4 viewOriginWorldSpace,
    float4 radialDirWorldSpace,
    float4 facingDirWorldSpace)
{
    const float3 centerPosRWS = boundingSphere.center - viewOriginWorldSpace.xyz;

    const float3 radialVec = abs(boundingSphere.radius) * radialDirWorldSpace.xyz;
    const float3 facingVec = abs(boundingSphere.radius) * facingDirWorldSpace.xyz;

    BoundingObjectData data;
    data.centerPosNDC = ComputeNormalizedDeviceCoordinates(centerPosRWS, viewProjMatrix);
    data.radialPosNDC = ComputeNormalizedDeviceCoordinates(centerPosRWS + radialVec, viewProjMatrix);
    data.frontCenterPosRWS = centerPosRWS + facingVec;
    return data;
}
```
在IsOcclusionVisible中，首先通过frontCenterPosRWS和subview索引计算出计算出queryClosestDepth，也就是包围球最近的那个深度值。然后使用frexp和max(mipLevel + 1, 0)计算出四个像素就能覆盖外包围球半径像素数的最大Mip级别，接下来通过ldexp将centerCoordInTopMip缩放到这级Mip得到centerCoordInChosenMip，再通过mipBounds计算得到采样的uv值，“clamp(centerCoordInChosenMip, .5f, float2(mipBounds.zw) - .5f)”操作则是防止采样到区域之外。最后通过GATHER_TEXTURE2D能取到四个深度值，通过它们与queryClosestDepth的比较之后，即可判断出当前instance外包围球是否被遮挡。

现在我们了解了完整的instance剔除流程，程序执行到这里也拿到了要渲染的数据，接下来我们接着来分析instance渲染部分源码。