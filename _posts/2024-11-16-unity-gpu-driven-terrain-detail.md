---
title: 'Unity GPU Driven地形细节渲染方案'
date: 2024-11-16
permalink: /posts/2024/1116/UnityGPUDrivenTerrainDetail/
tags:
  - Unity
  - Terrain Detail
  - GPU Driven
---

前言
----
Unity中Terrain内置的植被细节（草、花等）渲染方案性能较差，对于高密度的细节渲染比较无力，在移动端设备上的性能压力更是巨大。因此，我设计实现了一种GPU Driven的地形细节渲染方案，CPU端做四叉树剔除，GPU端做视椎体剔除和遮挡剔除，来最小化每帧渲染的植被数量，减少性能损耗。

地形细节数据
-----------
为了不浪费过多精力去开发额外的工具链，我们直接使用Unity内置的地形系统去生成地形植被，完成后再通过脚本导出数据。
我们使用ComputeDetailInstanceTransforms来获取每个Detail Patch的Transforms，也就是获取到每个Patch的植被Transform信息。在示例代码中，只取了地形的前两种植被，并且假设地形为正方形，来简化代码，读者可根据自身项目情况动态修改。在大多数情况下，植被（草、花）都是向上生长，模型大多数也都是采用广告板方式渲染，因此对于每个Detail Transform，仅取用Position信息，忽略缩放及旋转信息。示例代码如下:
```c#
class CustomTerrainData : ScriptableObject
{
    public int detailPatchDataSize;
    public DetailPatchData[] detailPatchData;
    public Vector3[] detailPositions;
}

//根据Terrain创建数据
void CreateDetailData(TerrainData terrainData, Terrain terrain)
{
    //创建asset
    var customTerrainDataAssetPath = $"YourPath/TerrainData.asset";
    var customTerrainData = AssetDatabase.LoadAssetAtPath<CustomTerrainData>(customTerrainDataAssetPath);
    if (customTerrainData == null)
    {
        customTerrainData = ScriptableObject.CreateInstance<CustomTerrainData>();
        AssetDatabase.CreateAsset(customTerrainData, customTerrainDataAssetPath);
    }

    //这里假设地形是正方形的，特殊情况需要做修改
    int terrainWidth = Mathf.CeilToInt(terrainData.size.x);
    int patchSize = terrainWidth / terrainData.detailPatchCount;
    customTerrainData.detailPatchData = new DetailPatchData[terrainData.detailPatchCount * terrainData.detailPatchCount];
    var detailPositionsList = new List<Vector3>();

    //遍历每个DetailPatch,获取每个DetailPatch的Detail数据
    for (int i = 0; i < terrainData.detailPatchCount; ++i)
    {
        for (int j = 0; j < terrainData.detailPatchCount; ++j)
        {
            customTerrainData.detailPatchData[i * terrainData.detailPatchCount + j] = new DetailPatchData();
            Vector3 center = new Vector3(i * patchSize + patchSize * 0.5f, 0.0f, j * patchSize + patchSize * 0.5f);

            Bounds grassBound = new Bounds();
            Vector2Int[] layerRange = new Vector2Int[2];
            //这里限制只读了两种detail,实际情况可能有更多,需要根据实际情况修改
            for (int detailLayerIdx = 0; detailLayerIdx < 2; ++detailLayerIdx)
            {
                //获取detail patch的Transform数据,并计算包围盒
                var detailInstanceTransforms = terrainData.ComputeDetailInstanceTransforms(i, j, detailLayerIdx, terrain.detailObjectDensity, out Bounds tileBound);
                var instancePositions = new Vector3[detailInstanceTransforms.Length];
                for (int detailIdx = 0; detailIdx < detailInstanceTransforms.Length; detailIdx++)
                {
                    instancePositions[detailIdx] = new Vector3(detailInstanceTransforms[detailIdx].posX,
                        detailInstanceTransforms[detailIdx].posY, detailInstanceTransforms[detailIdx].posZ) + transform.position;
                }
                if (detailLayerIdx == 0)
                {
                    grassBound = new Bounds(tileBound.center + transform.position, tileBound.size);
                }
                layerRange[detailLayerIdx] = new Vector2Int(detailPositionsList.Count, instancePositions.Length);
                detailPositionsList.AddRange(instancePositions);
            }

            var detailPatchData = customTerrainData.detailPatchData[i * terrainData.detailPatchCount + j];
            detailPatchData.detailBounds = new DetailBounds { center = grassBound.center, size = grassBound.size };
            //记录每个positionList的起始位置和长度
            detailPatchData.layerRange0X = layerRange[0].x;
            detailPatchData.layerRange0Y = layerRange[0].y;
            detailPatchData.layerRange1X = layerRange[1].x;
            detailPatchData.layerRange1Y = layerRange[1].y;
        }
    }
    customTerrainData.detailPatchDataSize = terrainData.detailPatchCount;
    customTerrainData.detailPositions = detailPositionsList.ToArray();
}
```

四叉树构建及查询
---------------
首先根据地形大小及Detail Patch数据我们可以轻松递归构建出一棵四叉树。
```c#
public class DetailQuadTree
{
    private DetailQuadTreeNode root;
    public DetailQuadTree(Vector3 terrainPosWS, float terrainWidth, int detailPatchDataSize, DetailPatchData[] detailPatchData)
    {
        if (detailPatchData == null || detailPatchDataSize == 0)
        {
            return;
        }

        int width = Mathf.CeilToInt(terrainWidth);
        int patchSize = width / detailPatchDataSize;
        //递归创建四叉树
        root = new DetailQuadTreeNode(terrainPosWS, detailPatchDataSize, detailPatchData, out Bounds bound, patchSize, width, Vector2Int.one * width / 2);
    }
}

public class DetailQuadTreeNode
{
    public Bounds bounds;
    public DetailQuadTreeNode[] children;
    public bool isLeaf = false;
    public Vector2Int[] layerRange = new Vector2Int[2] { Vector2Int.zero, Vector2Int.zero };

    public DetailQuadTreeNode(Vector3 terrainPosWS, int detailPatchDataSize, DetailPatchData[] detailPatchData, out Bounds nodeBound, int minPatchSize, int size, Vector2Int center)
    {
        //叶子节点
        if (size == minPatchSize)
        {
            isLeaf = true;
            int lowerP = minPatchSize / 2;
            var patchXY = new Vector2Int((int)((center.x - lowerP) / minPatchSize), (int)((center.y - lowerP) / minPatchSize));

            int length = detailPatchDataSize;
            if (patchXY.x < 0 || patchXY.y < 0 || patchXY.x >= length || patchXY.y >= length)
            {
                bounds = new Bounds(new Vector3(center.x, 0.0f, center.y), new Vector3(0, 0, 0));
                nodeBound = bounds;
                return;
            }

            var patchData = detailPatchData[patchXY.x * detailPatchDataSize + patchXY.y];
            bounds = new Bounds(patchData.detailBounds.center, patchData.detailBounds.size);
            var layerRange0 = new Vector2Int(patchData.layerRange0X, patchData.layerRange0Y);
            var layerRange1 = new Vector2Int(patchData.layerRange1X, patchData.layerRange1Y);
            layerRange = new Vector2Int[2] { layerRange0, layerRange1 };
            nodeBound = bounds;
            return;
        }
        int halfSize = size / 2;
        int halfhalfSize = halfSize / 2;

        //递归创建四叉树节点
        children = new DetailQuadTreeNode[4];
        children[0] = new DetailQuadTreeNode(terrainPosWS, detailPatchDataSize, detailPatchData, out Bounds bound0, minPatchSize, halfSize, new Vector2Int(center.x - halfhalfSize,     center.y + halfhalfSize));
        children[1] = new DetailQuadTreeNode(terrainPosWS, detailPatchDataSize, detailPatchData, out Bounds bound1, minPatchSize, halfSize, new Vector2Int(center.x + halfhalfSize, center.y + halfhalfSize));
        children[2] = new DetailQuadTreeNode(terrainPosWS, detailPatchDataSize, detailPatchData, out Bounds bound2, minPatchSize, halfSize, new Vector2Int(center.x - halfhalfSize, center.y - halfhalfSize));
        children[3] = new DetailQuadTreeNode(terrainPosWS, detailPatchDataSize, detailPatchData, out Bounds bound3, minPatchSize, halfSize, new Vector2Int(center.x + halfhalfSize, center.y - halfhalfSize));
        //合并四个子节点的包围盒
        bound0.Encapsulate(bound1);
        bound0.Encapsulate(bound2);
        bound0.Encapsulate(bound3);
        bounds = bound0;
        nodeBound = bounds;
    }
}
```

有了四叉树之后，我们就可以在每帧渲染之前通过四叉树去快速做Detail Patch的视椎体剔除。首先用GeometryUtility.CalculateFrustumPlanes获取摄像机的视椎体平面，然后四叉树遍历，逐节点做Bounds和Planes的相交测试，直到获取到全部的Detail Patch。示例代码如下：
```c#
public class DetailQuadTree
{
    public void Query(Camera camera, float range, int detailLayer, out List<Vector2Int> result)
    {
        result = new List<Vector2Int>();
        //只支持两种Detail，0和1
        if (detailLayer > 1 || detailLayer < 0) return;
        Plane[] planes = GeometryUtility.CalculateFrustumPlanes(camera);
        root.Query(camera.transform.position, range, detailLayer, planes, ref result);
    }
}

public class DetailQuadTreeNode
{
    //查询视椎体剔除之后的结果
    public void Query(Vector3 cameraPos, float range, int detailLayer, Plane[] planes, ref List<Vector2Int> result)
    {
        if (IsCollision(bounds, cameraPos, range))
        {
            //视椎体剔除
            if (GeometryUtility.TestPlanesAABB(planes, bounds))
            {
                if (isLeaf)
                {
                    result.Add(this.layerRange[detailLayer]);
                }
                else
                {
                    foreach (var child in children)
                    {
                        child.Query(cameraPos, range, detailLayer, planes, ref result);
                    }
                }
            }
        }
    }

    //判断AABB和球体是否相交(距离剔除)
    bool IsCollision(Bounds aabb, Vector3 spherePos, float radius)
    {
        Vector3 nearestPoint = AABBNearestPointToPoint(aabb, spherePos);
        float distance = Vector3.Distance(nearestPoint, spherePos);
        return distance <= radius;
    }
    Vector3 AABBNearestPointToPoint(Bounds aabb, Vector3 spherePos)
    {
        float x = spherePos.x;
        x = x > aabb.max.x ? aabb.max.x : x;
        x = x < aabb.min.x ? aabb.min.x : x; 
        float y = spherePos.y;
        y = y > aabb.max.y ? aabb.max.y : y;
        y = y < aabb.min.y ? aabb.min.y : y;
        float z = spherePos.z;
        z = z > aabb.max.z ? aabb.max.z : z;
        z = z < aabb.min.z ? aabb.min.z : z;
        return new Vector3(x, y, z);
    }
}
```

GPU剔除
-------
本部分假设读者已经了解并掌握GPU Driven的相关技术，建议不知道的读者先去学习了解下GPU Driven渲染的相关知识。

对于四叉树剔除之后的Detail Patch，使用Compute Shader进行下一步的视椎体剔除和遮挡剔除，首先准备好输入和输出的buffer。

detailPosBuffer里填充detail的position数据，作为剔除的输入。
```c#
var detailPosBuffer = new ComputeBuffer(grassQuadTree.detailPositions.Count, 3 * sizeof(float));
detailPosBuffer.SetData(grassQuadTree.detailPositions);
```
```hlsl
StructuredBuffer<float3> detailPosBuffer;
```

lodCullResults作为剔除的结果进行渲染，我们使用三级LOD来渲染地表细节，LOD0到LOD2逐渐减少Mesh的面数。
```c#
ComputeBuffer[] lodCullResults = new ComputeBuffer[3];
for (int i = 0; i < 3; ++i) {
    lodCullResults[i] = new ComputeBuffer(grassMaxCount[i], sizeof(float) * 3, 
      ComputeBufferType.Append);
}
```
```hlsl
AppendStructuredBuffer<float3> lod0CullResult;
AppendStructuredBuffer<float3> lod1CullResult;
AppendStructuredBuffer<float3> lod2CullResult;
```

接下来就可以调用Compute Shader去进行剔除。示例代码中只进行了视椎体剔除，没进行遮挡剔除，因为经测试开阔地形中，遮挡剔除的优化效率并不高，如有特殊需求，读者可自行补充。
```c#
for (int i = 0; i < pitches.Count; ++i) {
    if (detailCount <= 0) continue;
    grassCullingComputeShader.SetInt(instanceCountPropID, detailCount);
    grassCullingComputeShader.Dispatch(kernel, 1 + (detailCount / 256), 1, 1);
}
```
```hlsl
[numthreads(1024, 1, 1)]
void DetailCulling(uint3 id : SV_DispatchThreadID)
{
    if (id.x >= instanceCount) return;
    float3 position = detailPosBuffer[startIdx + id.x];
    float4x4 mMatrix = float4x4(
        float4(1, 0, 0, position.x),
        float4(0, 1, 0, position.y),
        float4(0, 0, 1, position.z),
        float4(0, 0, 0, 1)
    );
    float4x4 mvpMatrix = mul(vpMatrix, mMatrix);

    float4 boundVerts[8];
    boundVerts[0] = float4(boundMin, 1);
    boundVerts[1] = float4(boundMax, 1);
    boundVerts[2] = float4(boundMax.x, boundMax.y, boundMin.z, 1);
    boundVerts[3] = float4(boundMax.x, boundMin.y, boundMax.z, 1);
    boundVerts[4] = float4(boundMax.x, boundMin.y, boundMin.z, 1);
    boundVerts[5] = float4(boundMin.x, boundMax.y, boundMax.z, 1);
    boundVerts[6] = float4(boundMin.x, boundMax.y, boundMin.z, 1);
    boundVerts[7] = float4(boundMin.x, boundMin.y, boundMax.z, 1);

    float minX = 1, minY = 1, minZ = 1, maxX = -1, maxY = -1, maxZ = -1;
    bool isInClipSpace = false;
    for (int i = 0; i < 8; i++)
    {
        float4 positionCS = mul(mvpMatrix, boundVerts[i]);
        if (!isInClipSpace && IsInClipSpace(positionCS))
            isInClipSpace = true;

        float3 ndc = positionCS.xyz / positionCS.w;
        if (minX > ndc.x) minX = ndc.x;
        if (minY > ndc.y) minY = ndc.y;
        if (minZ > ndc.z) minZ = ndc.z;
        if (maxX < ndc.x) maxX = ndc.x;
        if (maxY < ndc.y) maxY = ndc.y;
        if (maxZ < ndc.z) maxZ = ndc.z;
    }
    if (!isInClipSpace)
        return;
    
    float dis = distance(cameraPos.xyz, position);
    if (dis > lodRange.x && dis <= lodRange.y)
    {
        lod0CullResult.Append(position);
    }
    else if (dis > lodRange.y && dis <= lodRange.z)
    {
        lod1CullResult.Append(position);
    }
    else if (dis > lodRange.z && dis <= lodRange.w)
    {
        lod2CullResult.Append(position);
    }
}

bool IsInClipSpace(float4 positionCS)
{
    return positionCS.x > - positionCS.w && positionCS.x < positionCS.w &&
    positionCS.y > - positionCS.w && positionCS.y < positionCS.w &&
    positionCS.z > 0 && positionCS.z < positionCS.w;
}
```

剔除结果不需要回读就可以通过DrawMeshInstancedIndirect绘制。
```c#
for (int i = 0; i < 3; ++i) {
    ComputeBuffer.CopyCount(lodCullResults[i], argsBuffer, (i * 5 + 1) * sizeof(uint));
    cmd.Clear();
    cmd.SetGlobalBuffer(detailPosBufferPropID, lodCullResults[i]);
    cmd.DrawMeshInstancedIndirect(lodMesh[i], 0, detailMaterial, 0, argsBuffer, i * 5 * sizeof(uint));
    context.ExecuteCommandBuffer(cmd);
}
```

对应的argsBuffer构建示例如下：
```c#
var argsBuffer = new ComputeBuffer(15, sizeof(uint), ComputeBufferType.IndirectArguments);
for (int lod = 0; lod < 3; ++lod) {
    args[0 + lod * 5] = (uint)lodMesh[lod].GetIndexCount(0);
    args[1 + lod * 5] = (uint)0;
    args[2 + lod * 5] = (uint)lodMesh[lod].GetIndexStart(0);
    args[3 + lod * 5] = (uint)lodMesh[lod].GetBaseVertex(0);
}
argsBuffer.SetData(args);
```

Shader可简单使用广告板shader改造，添加对应的效果（如次表面散射、高光、噪声颜色等等），示例代码如下：
```shaderlab
Shader "Detail"
{
    Properties
    {
      ......
    }

    SubShader
    {
        Tags { "RenderPipeline" = "UniversalPipeline" "RenderType" = "Opaque" "Queue" = "AlphaTest" }

        Pass
        {
            Name "Detail Forward"
            Tags { "RenderPipeline" = "UniversalPipeline" "RenderType" = "Opaque" "Queue" = "AlphaTest" }
            
            HLSLPROGRAM
            #pragma multi_compile_fog
            #pragma vertex vert
            #pragma fragment frag

            struct Attributes
            {
              ...
            };

            struct Varyings
            {
              ...
            };

            StructuredBuffer<float3> _DetailPosBuffer;

            Varyings vert(Attributes input)
            {
                Varyings o = (Varyings)0;
                
                float3 position = _DetailPosBuffer[input.instanceID];
                float4x4 localToWorldMat = float4x4(
                    float4(1, 0, 0, position.x),
                    float4(0, 1, 0, position.y),
                    float4(0, 0, 1, position.z),
                    float4(0, 0, 0, 1)
                );

                float3 cameraTransformRightWS = UNITY_MATRIX_V[0].xyz;
                float3 cameraTransformUpWS = UNITY_MATRIX_V[1].xyz;
                float3 cameraTransformForwardWS = -UNITY_MATRIX_V[2].xyz;

                if (cameraTransformForwardWS.y < - 0.999) cameraTransformUpWS.y += 0.01;
                
                float3 positionOS = input.positionOS.x * cameraTransformRightWS;
                positionOS += input.positionOS.y * cameraTransformUpWS;
                
                float3 positionWS = positionOS + UNITY_MATRIX_M._m03_m13_m23;

                ......

                return o;
            }

            half4 frag(Varyings IN) : SV_Target
            {
                ......

                return half4(0, 1, 0, 1);
            }
            ENDHLSL
        }
    }

    Fallback "Hidden/InternalErrorShader"
}
```

总结
----
读者根据如上思路和示例代码应该就可以很轻易的设计出符合自己项目的方案，来替换掉Unity内置的Terrain Detail渲染，大大提高渲染效率。

上述方案也有很大的优化空间，如：
+ 在遮挡物比较多的场景添加GPU遮挡剔除。
+ 地形很大时，可以使用视椎体包围球先剔除，通过的Bounds再与视椎体做剔除计算，亦可以使用Job System去加速计算。
+ 提前预构建场景管理结构，减少初始化时间。

如果读者发现了文章错误，请联系我改正。若有更好的优化方案或讨论，也欢迎邮箱联系我~