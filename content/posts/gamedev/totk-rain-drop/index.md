---
title: "Recreating the Rain Drop Effect in The Legend of Zelda: TOTK"
slug: "Recreating the Rain Drop Effect in The Legend of Zelda TOTK"
date: 2023-05-23T23:50:10-04:00
tags: ["Unity", "C#", "Compute Shader", "Render Feature", "Post Processing", "URP"]
categories: ["Game Development"]

draft: false
---

## Environment
Unity 2021.3.16f1  
Universal Render Pipeline 12.0.0

## Introduction

I recently finished *The Legend of Zelda: Tears of the Kingdom*. To me as an indie game developer, it felt like a text book of open world game design. I had a lot of fun playing it and I learned a lot from it.

One of the things I noticed is the rain drop effect that react to the uppper edges of the scene. It's a very subtle effect but it actually guided me through the thunder island where the player's visibility range was less than 1 meter. In the image below, you can see that the rain drops react to the edge of the mining cart rails, which tells the player that there is a path to continue the journey. I guess probably the game designer didn't intend to do this, but it's Nintendo, you never know.

{{< figure src="life-saver.png" >}}

This effect was one of the a few things I could rely on while exploring the island, so I decided to implement it in Unity as a render feature, just to show my appreciation to those little rain drops.

## Demo
{{< youtube 1oAGbDdVZ7E >}}  

## How It Works

The effect basically composes of two parts: Edge detection with angle limitation and a fast scrolling noise map.

### Edge Detection
From the image above, we can see that the effect is applied even in places that have no gradient such as the rails. Therefore, it has to be using the depth buffer. With that being clear, We just need to apply a sobel edge deteciton on the depth buffer and get a edge mask. 

If you are not familar with Sobel Edge Detection. Here is a quick explanation. Sobel Edge Detection is a simple edge detection algorithm that uses two 3x3 kernels to calculate the gradient of the image.

Horizontal:
```
-1  0  1
-2  0  2
-1  0  1
```

Vertical:
```
 1  2  1
 0  0  0
-1 -2 -1
```

Apply the two kernels respectively on each pixel(x,y) in the depth texture(depth):
```
horizontal = depth[x-1,y-1] * -1 + depth[x,y-1] * 0 + depth[x+1,y-1] * 1
           + depth[x-1,y]   * -2 + depth[x,y]   * 0 + depth[x+1,y]   * 2
           + depth[x-1,y+1] * -1 + depth[x,y+1] * 0 + depth[x+1,y+1] * 1

vertical = depth[x-1,y-1] * -1 + depth[x,y-1] * -2 + depth[x+1,y-1] * -1
         + depth[x-1,y]   * 0 + depth[x,y]   * 0 + depth[x+1,y]   * 0
         + depth[x-1,y+1] * 1 + depth[x,y+1] * 2 + depth[x+1,y+1] * 1
```

Then we can calculate the magnitude of the gradient by taking the square root of the sum of the squares of the two gradients:

```
magnitude = sqrt(horizontal * horizontal + vertical * vertical)
```

The magnitude is the value we are looking for. It describes the color gradient between the pixel and its neighbors. The higher the value, the more likely it is an edge.

From here, we do need one more value to describe the "direction" of the gradient. If it doesn't make sense to you. Just think about the gradient as a vector, composed of the horizontal gradient value we calculated before and the vertical gradient value. We can calculate the angle between the gradient vector and the horizontal axis by taking the arctangent of the vertical gradient over the horizontal gradient. 

```
angle = atan2(vertical, horizontal)
```

It gives us a value between -pi and pi, representing the angle of the vector in radian. 

Now we have the magnitude and the angle of the gradient. We can use them to create a mask that only shows the edges we want, which are the edges that are facing up.

{{< figure src="demo_mask.png" >}}

### Noise Map
Congradulations to you and future me that forgot how I did this! for making it this far. Now we have the edge mask, we can just multiply a scrolling noise map on it to get the final mask.

After that, configure the settings as you like: thickness, sobel threshold, rain drop scale, drop speed, drop color, etc.

For noise map, I used a simple 2D perlin noise map that I generated from this website: http://kitfox.com/projects/perlinNoiseMaker/

{{< figure src="noise_map.png" >}}

## Implementation

### Final Code

ZeldaRainDropFeature.cs
```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;
using UnityEngine.Serialization;

public class ZeldaRainDropFeature : ScriptableRendererFeature {
    [SerializeField] public Settings settings = new Settings();

    private RainDropRenderPass m_RenderPass;

    public override void Create() {
        m_RenderPass = new RainDropRenderPass(settings) {
            renderPassEvent = settings.renderPassEvent
        };
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData) {
        if (settings.rainDropShader != null && settings.rainDropShader != null) {
            renderer.EnqueuePass(m_RenderPass);
        }
    }

    private class RainDropRenderPass : ScriptableRenderPass {
        private Settings m_Settings;
        private RenderTargetHandle m_ResultTex; //camera color

        public RainDropRenderPass(Settings settings) {
            m_Settings = settings;
        }

        public override void Configure(CommandBuffer cmd, RenderTextureDescriptor cameraTextureDescriptor) {
            var descriptor = cameraTextureDescriptor;
            descriptor.colorFormat = RenderTextureFormat.ARGB32;
            descriptor.enableRandomWrite = true;
            cmd.GetTemporaryRT(m_ResultTex.id, descriptor);
        }

        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData) {
            if (!m_Settings.previewInSceneView && (renderingData.cameraData.isSceneViewCamera || renderingData.cameraData.isPreviewCamera)) {
                return;
            }

            CommandBuffer cmd = CommandBufferPool.Get(name: "Screen Door Transparency");
            cmd.Clear();

            //cache color target
            var cam = renderingData.cameraData.renderer;

            var shader = m_Settings.rainDropShader;
            var mainKernel = shader.FindKernel("ScreenSpaceRainDrop");

            //sobel
            cmd.SetComputeTextureParam(shader, mainKernel, "_InputColorTex", cam.cameraColorTarget);
            cmd.SetComputeTextureParam(shader, mainKernel, "_InputDepthTex", cam.cameraDepthTarget);
            cmd.SetComputeTextureParam(shader, mainKernel, "_NoiseTex", m_Settings.noiseTex);
            cmd.SetComputeIntParam(shader, "_Thickness", m_Settings.thickness);
            cmd.SetComputeFloatParam(shader, "_EdgeThreshold", m_Settings.sobelThreshold);

            //noise
            cmd.SetComputeFloatParam(shader, "_RainDropScale", m_Settings.rainDropScale);
            cmd.SetComputeIntParam(shader, "_NoiseWidth", m_Settings.noiseTex.width);
            cmd.SetComputeIntParam(shader, "_NoiseHeight", m_Settings.noiseTex.height);
            cmd.SetComputeVectorParam(shader, "_Time", Shader.GetGlobalVector("_Time"));
            cmd.SetComputeFloatParam(shader, "_DropSpeed", m_Settings.dropSpeed);
            cmd.SetComputeVectorParam(shader, "_DropColor", m_Settings.dropColor);

            //output
            cmd.SetComputeTextureParam(shader, mainKernel, "_OutputTex", m_ResultTex.Identifier());
            cmd.SetComputeIntParam(shader, "_Width", renderingData.cameraData.camera.scaledPixelWidth);
            cmd.SetComputeIntParam(shader, "_Height", renderingData.cameraData.camera.scaledPixelHeight);

            int threadGroupX = Mathf.CeilToInt(renderingData.cameraData.camera.scaledPixelWidth / 8.0f);
            int threadGroupY = Mathf.CeilToInt(renderingData.cameraData.camera.scaledPixelHeight / 8.0f);
            cmd.DispatchCompute(shader, mainKernel, threadGroupX, threadGroupY, 1);

            cmd.Blit(m_ResultTex.id, cam.cameraColorTarget);

            context.ExecuteCommandBuffer(cmd);

            CommandBufferPool.Release(cmd);

            context.Submit();
        }

        public override void FrameCleanup(CommandBuffer cmd) {
            cmd.ReleaseTemporaryRT(m_ResultTex.id);
        }
    }

    [System.Serializable]
    public class Settings {
        public ComputeShader rainDropShader;
        public Texture2D noiseTex;
        public Color dropColor = Color.white;
        [Range(0, 20)] public int thickness = 3;
        [Range(0f, 1f)] public float sobelThreshold = 0.166f;
        [Range(0f, 1f)] public float rainDropScale = 0.5f;
        public float dropSpeed = 100f;
        public RenderPassEvent renderPassEvent = RenderPassEvent.AfterRenderingTransparents;
        public bool previewInSceneView = true;
    }
}
```

ScreenSpaceRainDrop.compute
```hlsl
#pragma kernel ScreenSpaceRainDrop

RWTexture2D<float4> _OutputTex;
Texture2D _InputColorTex;
Texture2D _InputDepthTex;
Texture2D _NoiseTex;
int _Thickness;
float _RainDropScale;
float _EdgeThreshold;

int _Width;
int _Height;

int _NoiseWidth;
int _NoiseHeight;

float4 _Time;
float _DropSpeed;
float4 _DropColor;

static const float PI = 3.14159265f;

[numthreads(8,8,1)]
void ScreenSpaceRainDrop(uint3 id : SV_DispatchThreadID)
{
    if (int(id.x) + _Thickness >= _Width || int(id.y) + _Thickness >= _Height || int(id.x) - _Thickness < 0 || int(id.y) - _Thickness < 0)
    {
        _OutputTex[id.xy] = _InputColorTex[id.xy];
        return;
    }

    //p6    p7    p8
    //p3    p4    p5
    //p0    p1    p2

    float depthSqr = _InputDepthTex[id.xy].r * _InputDepthTex[id.xy].r;

    int sobelX = int(id.x);
    int sobelY = int(id.y) - _Thickness * depthSqr;
    int2 p0 = int2(sobelX - _Thickness * depthSqr, sobelY + _Thickness * depthSqr);
    int2 p1 = int2(sobelX, sobelY - _Thickness);
    int2 p2 = int2(sobelX + _Thickness * depthSqr, sobelY - _Thickness * depthSqr);
    int2 p3 = int2(sobelX - _Thickness * depthSqr, sobelY);

    int2 p5 = int2(sobelX + _Thickness * depthSqr, sobelY);
    int2 p6 = int2(sobelX - _Thickness * depthSqr, sobelY + _Thickness * depthSqr);
    int2 p7 = int2(sobelX, sobelY + _Thickness * depthSqr);
    int2 p8 = int2(sobelX + _Thickness * depthSqr, sobelY + _Thickness * depthSqr);

    // derivative in two directions
    float dxPosX = _InputDepthTex[p0].r * -1 + _InputDepthTex[p3].r * -2 + _InputDepthTex[p6].r * -1 + _InputDepthTex[p2].r + _InputDepthTex[p5].r * 2 + _InputDepthTex[p8].r;
    float dxPosY = _InputDepthTex[p0].r * -1 + _InputDepthTex[p1].r * -2 + _InputDepthTex[p2].r * -1 + _InputDepthTex[p6].r + _InputDepthTex[p7].r * 2 + _InputDepthTex[p8].r;

    //edge mask
    float dx = sqrt(dxPosX * dxPosX + dxPosY * dxPosY);
    float edgeMask = step(_EdgeThreshold, dx);

    //angle mask
    float dir = atan2(dxPosY, dxPosX);
    dir = (dir + PI) / (2 * PI);
    float angleMask = step(dir, 0.5f);
    edgeMask *= angleMask;

    //noise mask
    int noiseX = int((id.x + _Time.g * _DropSpeed) % _NoiseWidth);
    int noiseY = int((id.y + _Time.g * _DropSpeed) % _NoiseHeight);
    float noiseMask = step(_RainDropScale, _NoiseTex[int2(noiseX, noiseY)].r);

    float finalMask = edgeMask * angleMask * noiseMask;
    float4 result = lerp(_InputColorTex[id.xy], _DropColor, finalMask * _DropColor.a);

    _OutputTex[id.xy] = result;
}
```

## Improvement
There are a few things I can improve in the future:
1. Change the effect to Layer-based or Object-based for some very specific effects. But it will cost more performance and Tears of the kindom didn't do that, so I guess it's not necessary.
2. Play with the noise map to get a better looking rain drop. For example, downscale the noise map to get more blurred rain drops and save performance.