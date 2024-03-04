---
title: "Create GPU Instancing Billboard Grass Without Creating Position Buffer in URP"
slug: "Create GPU Instancing Billboard Grass Without Creating Position Buffer in URP"
date: 2024-03-04T11:59:37-07:00
tags: ["Unity", "Shader", "GPU Instancing", "Grass", "URP"]
categories: ["EOTR Devlog"]

draft: false
---

## Environment 
Unity 2021.3.16f1  
URP 12.0

## Prerequisite
Basic knowledge of C# and Shader programming (HLSL).


## GPU Instancing
GPU instancing is a technique that allows you to draw multiple copies of the same mesh using a single draw call. It is basically normal forward rendering pipeline but skipping the Vertex Specification stage.

## Graphics.DrawMeshInstancedIndirect
The first step to create multiple millions of grass is to properly call the GPU to draw so, a.k.a. draw call. In Unity, we can use `Graphics.DrawMeshInstancedIndirect` to create a GPU instancing draw call.

```csharp
// Graphics.cs
public void DrawMeshInstancedIndirect(
Mesh mesh, int submeshIndex, Material material, 
Bounds bounds, ComputeBuffer bufferWithArgs, 
int argsOffset = 0, MaterialPropertyBlock properties = null,
ShadowCastingMode castShadows = ShadowCastingMode.On, 
bool receiveShadows = true, int layer = 0, 
Camera camera = null, LightProbeUsage lightProbeUsage = LightProbeUsage.BlendProbes, 
LightProbeProxyVolume lightProbeProxyVolume = null
);
```

Most of the parameters are self-explanatory. The `ComputeBuffer bufferWithArgs` is the most important one. It is a buffer that contains the arguments for the draw call. This has to be exacly a 5 elements long int array.

```csharp
int[] args = new int[5] { 0, 0, 0, 0, 0 };
```

The five arguments that we are going to fill inside this int array are: 
1. the number of instances
2. the number of instances to draw
3. the start instance location
4. the base vertex location
5. the start index location

With the knowledge above, we can create a simple script to draw a single mesh with GPU instancing.
```csharp
using UnityEngine;

public class BillboardGrassRenderer : MonoBehaviour
{
    public Mesh mesh;
    public int subMeshIndex = 0;
    public int instanceCount = 1000;
    public Vector3 dimension = new Vector3(10, 10, 10);
    public Vector2 density = new Vector2(10, 10);

    private Material m_Material;
    private MaterialPropertyBlock m_PropertyBlock = new MaterialPropertyBlock();
    private ComputeBuffer m_ArgsBuffer;
    private uint[] m_Args = new uint[5] { 0, 0, 0, 0, 0 };

    private void Start()
    {
        m_ArgsBuffer = new ComputeBuffer(1, m_Args.Length * sizeof(uint), ComputeBufferType.IndirectArguments);
        m_Args[0] = (uint)mesh.GetIndexCount(subMeshIndex);
        m_Args[1] = (uint)instanceCount;
        m_Args[2] = (uint)mesh.GetIndexStart(subMeshIndex);
        m_Args[3] = (uint)mesh.GetBaseVertex(subMeshIndex);
        m_Args[4] = 0;
        m_ArgsBuffer.SetData(m_Args);
        var dimensionAndDensity = new Vector4(dimension.x, dimension.z, density.x, density.y);
        m_PropertyBlock.SetVector(DimensionAndDensityPropertyID, dimensionAndDensity);
        m_Material = new Material(Shader.Find("Universal Render Pipeline/Lit"));
    }
    private void Update()
    {
        Graphics.DrawMeshInstancedIndirect(mesh, subMeshIndex, m_Material, new Bounds(transform.position, dimension), m_ArgsBuffer, 0, m_PropertyBlock);
    }

    private void OnDisable()
    {
        if (m_ArgsBuffer != null)
        {
            m_ArgsBuffer.Release();
        }

        _argsBuffer = null;
    }
}
```

## Material

As you probably noticed in the script, the grass material is currently using URP/Lit as its shader. As we didn't compute the grass positions anywhere, the grass will be drawn at the origin. We can fix this in two ways:

1. Compute position info in the shader
2. Use a compute shader to calculate the positions, and a compute buffer to store the positions info and pass it to the shader.

The difference between the two methods is that the first method is more efficient as there's no need to pass the information around between CPU and GPU , but the second method is more flexible. We will use the first method in this tutorial.

```hlsl
Shader "Hidden/XiheRendering/BillboardGrassUnlit" {
    Properties {
        _MainTex ("Albedo (RGB)", 2D) = "white" {}
    }
    SubShader {
        Cull Off
        Zwrite On

        Pass {
            Name "BillboardGrassForward"

            Tags {
                "LightMode" = "UniversalForward"
            }

            HLSLPROGRAM
            #pragma vertex Vertex
            #pragma fragment Fragment
            #pragma target 4.5
            #pragma multi_compile_instancing

            #include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Color.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

            UNITY_INSTANCING_BUFFER_START(Props)
                UNITY_DEFINE_INSTANCED_PROP(float4, _Dimension) //x: dimensionX, y: dimensionZ, z: densityX, w: densityZ
            UNITY_INSTANCING_BUFFER_END(Props)

            TEXTURE2D(_MainTex);
            SAMPLER(sampler_MainTex);

            struct Attributes
            {
                float4 vertex : POSITION;
                float4 uv : TEXCOORD0;
            };

            struct Varyings
            {
                float4 positionHCS : SV_POSITION;
                float2 uv : TEXCOORD0;
            };

            Varyings Vertex(Attributes IN, uint instanceID : SV_InstanceID)
            {
                float dimensionZ = floor(_Dimension.y * _Dimension.w); //actual number of instances allowed on z-axis
                float offsetX = floor(instanceID / dimensionZ) / _Dimension.z;
                float offsetZ = instanceID % dimensionZ / _Dimension.w;

                float3 localPosition = IN.vertex.xyz;
                float2 instanceID2D01 = float2(offsetX / _Dimension.x, offsetZ / _Dimension.y);//no use for now, use this to sample noise later 

                float3 localPositionOffset = localPosition - float3(_Dimension.x, 0, _Dimension.y) / 2.0 + float3(offsetX, 0, offsetZ);

                Varyings o;
                o.positionHCS = TransformObjectToHClip(localPositionOffset);
                o.uv = IN.uv.xy;
                return o;
            }

            float4 Fragment(Varyings IN) : SV_Target
            {
                float4 albedo = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, IN.uv);
                clip(albedo.a - 0.5);

                return albedo;
            }
            ENDHLSL
        }
    }
}
```

First thing is first, in order to let the shader support GPU instancing, we need to add `#pragma multi_compile_instancing` to the shader. This will tell the shader to generate multiple shader variants for each instance.

All keywords and macros can be found here: [Creating shaders that support GPU instancing](https://docs.unity3d.com/Manual/gpu-instancing-shader.html).

In this basic shader, we first declare the `_Dimension` property inside the `UNITY_INSTANCING_BUFFER_START` and `UNITY_INSTANCING_BUFFER_END` macros. 

`UNITY_INSTANCING_BUFFER` is a per-instance type of CBUFFER which will receive values from MaterialPropertyBlock from C# side. 

We will use this property to calculate a 2D(XZ) position offset(similar to DispatchThreadID) of the grass instance. Let's first draw a grass instance every 1 meter on the x-axis and 1 meter on the z-axis. We can then calculate the offset distance for each instance on the x and z axis using the instance ID and the `_Dimension` property.

- _Dimension.x: dimension of the grass bounding volume on the x-axis
- _Dimension.y: dimension of the grass bounding volume on the z-axis
- _Dimension.z: density of the grass per meter on the x-axis
- _Dimension.w: density of the grass per meter on the z-axis

```hlsl
//actual number of instances allowed on z-axis
float dimensionZ = floor(_Dimension.y * _Dimension.w);
float offsetX = floor(instanceID / dimensionZ) / _Dimension.z;
float offsetZ = instanceID % dimensionZ / _Dimension.w;
```

Now we got offset distance for each instance on the x and z axis, add the offset to the origin position and subtract half of the bounding volume to get a local position from the left, backward corner of the bounding volume:

```hlsl
float3 localPositionOffset = 
localPosition - float3(_Dimension.x, 0, _Dimension.y) / 2.0 + float3(offsetX, 0, offsetZ);
```

We can also calculate the normalized Offset as a grass bounding box space UV for sampling scrolling noise textures later: 
```hlsl
//normalized instance ID in 2D space
float2 instanceID2D01 = float2(offsetX / _Dimension.x, offsetZ / _Dimension.y);
```

## Adding Position Offset and Wind Effect
Now if you click on Play button again, you will see a grid of grass instances, all sitting peacefully, no animation, no position offset.

## Demo

TBD


