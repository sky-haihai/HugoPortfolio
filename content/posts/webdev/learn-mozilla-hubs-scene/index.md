---
title: "Notes on Learning Mozilla Hubs: Scene"
slug: "Notes on Learning Mozilla Hubs Scene"
date: 2023-10-26T10:27:04-06:00
tags: ["Metaverse", "Mozilla Hubs", "Blender", "GLB", "Opimazation"]
categories: ["Web Development"]

draft: false
---

## Introduction

Mozilla Hubs is a open-source browser-based metaverse web application that supports XR. The first thing to keep in mind is that it's component-based like Unity. In the official build, Hubs supports various but very limited components such as model renderer, images renderer, videos renderer, audio field, etc. Although what creators could do in Hubs is very limited by the default build, I could imagine it's bright future for two reasons:

1. It's open-source, meaning developers could add their own components into the engine. Though, I wish the official team could add scripting interface as a component like Unity does.
2. The rooms(scenes) are browser-based, meaning it's cross-platform and easy to access, which attracts more users.

## Limitations

1.  No support for custom shaders
1.  No support for mipmaps
1.  No support for LOD
1.  No support for texture filters
1.  Alpha blending is supported in published scenes but limited to 1 time(1-bit) in spoke editor
1.  The only external scene editor work an replacement of Spoke is Blender. Other mainstream DCC(Digital Content Creation) software like Maya, 3ds Max, Cinema 4D, etc. are not supported.

## Scene Editing Workflow

There are two types of workflow for artists to create Hubs scenes(.spoke) using Blender.

### Blender-As-Main-Editor

The first one is what I called Blender-As-Main-Editor workflow:

1. Install Hubs Blender export addon
2. Create models, draw/import textures, and eventually finish the scene in Blender
3. Add required Hubs components to objects using the Hubs addon
4. Export the scene as a .glb file

In short: Asset creation in Blender -> Add Hubs components in Blender -> Export as .glb -> Import .glb into Spoke as a

### Blender-As-Asset-Creation-Tool

The second one is what I called Blender-As-Asset-Creation-Tool workflow:

1. No need to install Hubs Blender export addon
2. Create modular models/collection of models in Blender
3. Export the model/models as .glb files
4. Login to Spoke editor and upload the .glb files as "My Assets"
5. Place the assets in the scene and add components to them individually.

### Comparison

#### Similarity

1. Both workflows require Blender to create assets

#### Difference

1. The Blender-As-Main-Editor workflow requires setting up every detail in Blender before exporting the scene into Spoke and after that there is no way to give any adjustment to the scene in Spoke editor, while the other workflow can give artists more flexibility to adjust the scene in Spoke editor, especiall when dealing with a complex scene and collaborating with other artists.

## Rendering Pipeline

### PBR Workflow

-   PBR, but Mobile device only has limited PBR

## Official Suggestted Performance Budget

### Poly Count

Up to 50,000 Triangles

### Materials

-   Up to 25 Unique Materials
-   Haven't tested if similar materials with different input values are batched together, but I guess they are not.

### Textures

-   256MB of VRAM
-   Each texture should be not larger than 2048x2048

### Lights

-   Mobile does not support dynamic lights at all
-   In high end devices such as Desktop browsers, 3 dynamic lights is suggested

### File Size

-   16MB for the whole scene in general
-   This really just depends on network condition, in other words, location of the server

## Opimization

### Lightmapping

The biggest compact of the visual quality of scene with limited graphical computing power for low-end devices is for sure Lightmapping. Because Hubs Rooms run on a browser which only support basic PBR lightmodle, without lightmaps, the scene will look like a flat cartoon. If you are not familiar with lightmapping, here's a quick comparison:

{{< figure src="with-lightmapping.png" title="With Lightmapping" >}}
{{< figure src="without-lightmapping.png" title="Without Lightmapping" >}}

### Mix Actual 3D and Billboard Models

Since the scene is limited to 50k triangles, it's important to insert some 2-triangle Billboard models to replace some 3D models that are far away from the camera or not important component of the scene. For example, in a demo scene I made while learning Hubs, I modelled the main objects: pumpkin lamps as 3D models and used a bunch of Billboards to replace the trees which are less important visual components of the scene, and the scene still looks decent.

{{< figure src="billboard-tree.png" >}}

### Use Low Resolution Textures

While texture resolution has a suggested value of no larger than 2K, it is still important to keep in mind that not every object in the scene need a 2K texture. Artists should decide the resolution of each texture based on the size and the significance of the object in the scene. For example, a 2K texture is not necessary for a small bat that is far away from the camera. In that case, a 256x256 texture is probably enough.

## Room Accessing

-   No account required
-   Assets(e.g. Images, Videos) is URL-based, which means assets are downloaded from the company's file server when user enters the room, e.g. images
-   User can upload their own assets upon permission, but they will be deleted when the room is closed
-   Some properties of objects in the room are shared across all users. (e.g. pause state of the video player, etc.)
-   Some properties are not shared, (e.g. volume of the video player, position of the player model, etc.)
-   Some user behaviours can be enabled/disabled by the room owner, (e.g. user can/cannot switch first/third person camera, user can/cannot fly, etc.)

## Copy Right
Although Mozilla Hubs is open-sourced, the assets created by artists usually can only be used under conditions. It is nesscessary to follow the guideline of licenses of all assets used in the scene if the scene is public or used commercially. 

## References

Mozilla. Blender Addon Components Manual. Hubs Documentation. https://hubs.mozilla.com/docs/creators-blender-components.html

Mozilla. Optimizing Scenes. Hubs Documentation.  
https://hubs.mozilla.com/docs/spoke-optimization.html

Valencia, P. (2022, December 21). Creating 2d sprite-based Retro Worlds. Creator Labs.  
https://hubs.mozilla.com/labs/creating-retro-worlds/
