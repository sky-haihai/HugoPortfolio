---
title: "Create Stylized GPU Instancing Billboard Grass Without Creating Position Buffer in URP"
slug: "Create Stylized GPU Instancing Billboard Grass Without Creating Position Buffer in URP"
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

            #include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Color.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

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

                Varyings o;
                o.positionHCS = TransformObjectToHClip(IN.vertex.xyz);
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

### GPU Instancing Keywords and Macros
First thing is first, in order to let the shader support GPU instancing, we need to add `#pragma multi_compile_instancing` to the shader. This will tell the shader to generate multiple shader variants for each instance.

All keywords and macros can be found here: [Creating shaders that support GPU instancing](https://docs.unity3d.com/Manual/gpu-instancing-shader.html).


### Material Property

In this basic shader, we first declare the `_Dimension` property inside the `UNITY_INSTANCING_BUFFER_START` and `UNITY_INSTANCING_BUFFER_END` macros. 

```hlsl
UNITY_INSTANCING_BUFFER_START(Props)
    UNITY_DEFINE_INSTANCED_PROP(float4, _Dimension) //x: dimensionX, y: dimensionZ, z: densityX, w: densityZ
UNITY_INSTANCING_BUFFER_END(Props)
```

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

### Local Position Offset
Now we got offset distance for each instance on the x and z axis, add the offset to the origin position and subtract half of the bounding volume to get a local position from the left, backward corner of the bounding volume:

```hlsl
float3 localPositionOffset = 
localPosition - float3(_Dimension.x, 0, _Dimension.y) / 2.0 + float3(offsetX, 0, offsetZ);
```

### Noise Texture UV

We can also calculate the normalized offset to get a grass-bounding-box-space UV for sampling scrolling noise textures later, x-axis is the offsetX, y-axis is the offsetZ. We can use this UV to sample the noise texture later to add some wind effect to the grass. 
```hlsl
//normalized instance ID in 2D space
float2 instanceID2D01 = float2(offsetX / _Dimension.x, offsetZ / _Dimension.y);
```

### Static GPU Instancing Grass Shader

Putting all the pieces together, we get the following shader:

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

## Noise

Now if you click on Play button again, you will see a grid of grass instances, all sitting peacefully, no animation, no position offset, not fit for our goal.

### Control Map
We can add multiple noise texture to randomize and control all the grass properties, for example, position, desity, scale, and height.

```hlsl
_PositionTex("Position Offset", 2D) = "white" {}
_DensityTex("Density", 2D) = "white" {}
_ScaleTex("Scale", 2D) = "white" {}
_HeightTex("Height", 2D) = "white" {}
```

But as you probably noticed, most of the properties only require a float value, which means each of them is only going to take one channel to store the data. We can store all the one-channel data into RGBA respectively in a single texture called `_ControlTex`:

```hlsl
_ControlTex("Control", 2D) = "white" {}//R: Position Offset, G: Density, B: Scale, A: Height
```

To use the texture, we can use the normalized 2D instance ID we calculated before as the UV to sample the texture:

```hlsl
float4 control = SAMPLE_TEXTURE2D_LOD(_ControlTex, sampler_ControlTex, instanceID2D01, 0);
//control.r: position offset
//control.g: density
//control.b: scale
//control.a: height
```
Now we got a float4 control value for that vertex instance, we can then use it to control the position, density, scale, and height of the grass instance later.

### Wind Map

For wind effect, we can use another scrolling continuous noise texture to simulate the wind effect, four channels of the texture can be used to control the wind effect in different directions and the scale if the grass instance.  

```hlsl
_NoiseTex ("Noise (RGBA)", 2D) = "white" {}//R: PosX, G: PosY, B: PosZ, A: Scale
```

```hlsl
UNITY_INSTANCING_BUFFER_START(Props)
...
    UNITY_DEFINE_INSTANCED_PROP(float4, _SwingScale)
    UNITY_DEFINE_INSTANCED_PROP(float, _Speed)
...
UNITY_DEFINE_INSTANCED_END(Props)
```

```hlsl
float4 noise = SAMPLE_TEXTURE2D_LOD(_NoiseTex, sampler_NoiseTex, instanceID2D01 + _Time.xy * _Speed, 0);
```

Now we got another float4 noise value for wind.

## Final

### Shader
Putting all the pieces together, we get the final shader:

```hlsl
Shader "Hidden/XiheRendering/BillboardGrassUnlit" {
    Properties {
        _MainTex ("Albedo (RGB)", 2D) = "white" {}
        _ControlTex ("Control (RGBA)", 2D) = "white" {}//R: Position Offset, G: Density, B: Scale, A: Height
        _NoiseTex ("Noise (RGBA)", 2D) = "white" {}//R: PosX, G: PosY, B: PosZ, A: Scale
    }
    SubShader {
        Cull Off
        Zwrite On

        Pass {
            Name "BillboardGrassForward"

            Tags {
                "LightMode" = "UniversalForward"
            }

            Blend SrcAlpha OneMinusSrcAlpha

            HLSLPROGRAM
            #pragma vertex Vertex
            #pragma fragment Fragment
            #pragma target 4.5
            #pragma multi_compile_instancing

            #include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Color.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

            UNITY_INSTANCING_BUFFER_START(Props)
                UNITY_DEFINE_INSTANCED_PROP(float4, _Dimension) //x: dimensionX, y: dimensionZ, z: densityX, w: densityZ
                UNITY_DEFINE_INSTANCED_PROP(float, _DimensionY)
                UNITY_DEFINE_INSTANCED_PROP(float4, _ControlNoiseScale)
                UNITY_DEFINE_INSTANCED_PROP(float4, _SwingScale)
                UNITY_DEFINE_INSTANCED_PROP(float, _Speed)
            UNITY_INSTANCING_BUFFER_END(Props)

            TEXTURE2D(_MainTex);
            SAMPLER(sampler_MainTex);

            //R: PosX, G: PosZ, B: Scale, A: Height
            TEXTURE2D(_ControlTex);
            SAMPLER(sampler_ControlTex);

            //R: PosX, G: PosY, B: PosZ, A: Scale
            TEXTURE2D(_NoiseTex);
            SAMPLER(sampler_NoiseTex);

            struct Attributes
            {
                float4 vertex : POSITION;
                float4 uv : TEXCOORD0;
            };

            struct Varyings
            {
                float4 positionHCS : SV_POSITION;
                float2 uv : TEXCOORD0;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            Varyings Vertex(Attributes IN, uint instanceID : SV_InstanceID)
            {
                float dimensionZ = floor(_Dimension.y * _Dimension.w); //actual number of instances allowed on z-axis
                float offsetX = floor(instanceID / dimensionZ) / _Dimension.z;
                float offsetZ = instanceID % dimensionZ / _Dimension.w;

                float3 localPosition = IN.vertex.xyz;
                // float2 instanceID2D01 = float2(offsetX / _Dimension.x, offsetZ / _Dimension.y);
                float2 instanceID2D01 = float2(offsetX / _Dimension.x, offsetZ / _Dimension.y);

                //R: Pos Noise, G: Scale Noise, B: Density, A: Height
                float4 control = SAMPLE_TEXTURE2D_LOD(_ControlTex, sampler_ControlTex, instanceID2D01, 0);
                float4 noise = SAMPLE_TEXTURE2D_LOD(_NoiseTex, sampler_NoiseTex, instanceID2D01+_Time.xy*_Speed, 0);

                localPosition.x += lerp(0, (noise.r * 2 - 1) * _SwingScale.x, IN.uv.x);
                localPosition.z += lerp(0, (noise.g * 2 - 1) * _SwingScale.z, IN.uv.y);
                localPosition *= (noise.a * 2 - 1) * _SwingScale.w + 1;
                localPosition *= step(.5, control.g) * (control.b * 2) * _ControlNoiseScale.z + 1;

                localPosition.x += (control.r * 2 - 1) * _ControlNoiseScale.x;
                localPosition.z += (control.r * 2 - 1) * _ControlNoiseScale.y;
                localPosition.y += -_DimensionY / 2 + control.a * _DimensionY;

                //calculate world position                
                float3 worldPosition = localPosition - float3(_Dimension.x, 0, _Dimension.y) / 2.0 + float3(offsetX, 0, offsetZ);

                Varyings o;
                o.positionHCS = TransformObjectToHClip(worldPosition);
                o.uv = IN.uv.xy;
                return o;
            }

            float4 Fragment(Varyings IN) : SV_Target
            {
                float4 albedo = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, IN.uv);
                clip(albedo.a - 0.5);

                float4 output = albedo;

                return output;
            }
            ENDHLSL
        }
    }
}
```

### Renderer
```csharp
using System;
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Serialization;

namespace XiheRendering.Procedural.BillboardGrass {
    public class BillboardGrassRenderer : MonoBehaviour {
        public Mesh mesh;
        public int subMeshIndex = 0;

        // material
        public Texture2D colorMap;
        public Texture2D controlMap;
        public Texture2D noiseMap;
        public int layer;
        public ShadowCastingMode castShadows = ShadowCastingMode.On;
        public bool receiveShadows = true;

        // instancing
        public Vector3 dimension = new Vector3(10, 10, 10);
        public Vector2 density = new Vector2(100, 100);
        public Vector4 controlNoiseScale = new Vector4(1, 1, 1, 1);
        public Vector3 swingScale = new Vector3(1, 1, 1);
        public float scaleSwingScale = 1;
        public float swingSpeed = 1;

        public bool renderInSceneCamera;

        private Material m_Material;

        private Vector2 m_CachedDensity = Vector2.zero;
        private Vector3 m_CachedDimension = Vector3.zero;
        private Vector4 m_CachedControlNoiseScale = Vector4.zero;
        private Vector3 m_CachedSwingScale = Vector3.zero;
        private float m_CachedScaleSwingScale = 0;
        private float m_CachedSwingSpeed = 0;
        private Texture2D m_CachedColorMap;
        private Texture2D m_CachedControlMap;
        private Texture2D m_CachedNoiseMap;

        private ComputeBuffer m_ArgsBuffer;
        private readonly uint[] m_Args = new uint[5] { 0, 0, 0, 0, 0 };
        private MaterialPropertyBlock m_PropertyBlock;
        private static readonly int DimensionAndDensityPropertyID = Shader.PropertyToID("_Dimension");
        private static readonly int DimensionYPropertyID = Shader.PropertyToID("_DimensionY");
        private static readonly int ColorMapPropertyID = Shader.PropertyToID("_MainTex");
        private static readonly int ControlMapPropertyID = Shader.PropertyToID("_ControlTex");
        private static readonly int NoiseMapPropertyID = Shader.PropertyToID("_NoiseTex");
        private static readonly int SwingSpeedPropertyID = Shader.PropertyToID("_Speed");
        private static readonly int ControlNoiseScale = Shader.PropertyToID("_ControlNoiseScale");
        private static readonly int SwingScale = Shader.PropertyToID("_SwingScale");

        void Start() {
            m_ArgsBuffer = new ComputeBuffer(1, m_Args.Length * sizeof(uint), ComputeBufferType.IndirectArguments);
            m_PropertyBlock = new MaterialPropertyBlock();
            UpdateBuffers();
        }

        void Update() {
            // Update starting position buffer
            if (m_CachedDensity != density || m_CachedDimension != dimension || m_CachedControlNoiseScale != controlNoiseScale || m_CachedSwingScale != swingScale ||
                Math.Abs(m_CachedScaleSwingScale - scaleSwingScale) > float.Epsilon || colorMap != m_CachedColorMap || controlMap != m_CachedControlMap ||
                noiseMap != m_CachedNoiseMap) {
                UpdateBuffers();
            }

            // Render
            Graphics.DrawMeshInstancedIndirect(mesh, subMeshIndex, m_Material, new Bounds(transform.position, dimension), m_ArgsBuffer, 0, m_PropertyBlock,
                castShadows, receiveShadows, layer, renderInSceneCamera ? null : Camera.current);
        }

        void UpdateBuffers() {
            if (mesh == null) {
                m_Args[0] = m_Args[1] = m_Args[2] = m_Args[3] = 0;
                m_PropertyBlock.Clear();
                Debug.LogWarning("You forgot to assign a mesh to GPU grass generator");
                return;
            }

            // Ensure submesh index is in range
            subMeshIndex = Mathf.Clamp(subMeshIndex, 0, mesh.subMeshCount - 1);
            density = new Vector2(Mathf.Max(0, density.x), Mathf.Max(0, density.y));
            dimension = new Vector3(Mathf.Max(0, dimension.x), Mathf.Max(0, dimension.y), Mathf.Max(0, dimension.z));

            //set properties
            if (m_Material == null) {
                m_Material = new Material(Shader.Find("Hidden/XiheRendering/BillboardGrassUnlit"));
            }

            var dimensionAndDensity = new Vector4(dimension.x, dimension.z, density.x, density.y);
            m_PropertyBlock.SetVector(DimensionAndDensityPropertyID, dimensionAndDensity);
            m_PropertyBlock.SetFloat(DimensionYPropertyID, dimension.y);
            m_PropertyBlock.SetTexture(ColorMapPropertyID, colorMap);
            m_PropertyBlock.SetTexture(ControlMapPropertyID, controlMap);
            m_PropertyBlock.SetTexture(NoiseMapPropertyID, noiseMap);
            m_PropertyBlock.SetFloat(SwingSpeedPropertyID, swingSpeed);
            m_PropertyBlock.SetVector(ControlNoiseScale, controlNoiseScale);
            m_PropertyBlock.SetVector(SwingScale, new Vector4(swingScale.x, swingScale.y, swingScale.z, scaleSwingScale));

            // Args
            // index count per instance,
            // instance count,
            // start index location,
            // base vertex location,
            // start instance location.
            m_Args[0] = (uint)mesh.GetIndexCount(subMeshIndex);
            m_Args[1] = (uint)Mathf.FloorToInt(density.x * dimension.x * density.y * dimension.z);
            m_Args[2] = (uint)mesh.GetIndexStart(subMeshIndex);
            m_Args[3] = (uint)mesh.GetBaseVertex(subMeshIndex);

            m_ArgsBuffer.SetData(m_Args);

            m_CachedDensity = density;
            m_CachedDimension = dimension;
        }

        void OnDisable() {
            if (m_ArgsBuffer != null)
                m_ArgsBuffer.Release();
            m_ArgsBuffer = null;
        }

#if UNITY_EDITOR
        private void OnDrawGizmos() {
            //draw bounding box
            Gizmos.color = Color.red;
            Gizmos.DrawWireCube(transform.position, dimension);
        }
#endif
    }
}
```

### Next Steps

As you can see, we have created a GPU instancing grass system...for flat surfaces without any obstacles..., which isn't very useful. The good thing is that we have implemented the control map and noise map, all that's left is a tool to generate those maps based on the real environment.

In the next post, we will create an tool based on untiy's EditorWindow to bake the control map and noise map!