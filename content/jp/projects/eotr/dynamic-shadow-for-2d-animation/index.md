---
title: "Devlog: Dynamic Shadow for 2D Animation"
dateMonthYear: April 2024
type: page
image: "/images/2ndmethod.png"

---

## Why Dynamic Shadow for 2D Animation?

In the game project Echoes of the Roots, an very important art decision to win us the Best Art Award was the pillar to combine 2D animation and 3D enviroment to achieve a sense 2.5D Visual Style.

Among the various efforts, a key point was to achieve the consistancy of the shadow between 2D and 3D objects. 3D objects generally involve a more complex lighting model than 2D objects, so the our direction was pretty clear: to simplify the lighting model for 3D objects to match the 2D look, and to make 2D shadows dynamic to trick the player's eyes to perceive that the 2D and 3D objects are in the same world.

## Trial #1: Normal Map Baking

### Method

Initially, we tried to bake a normal map for each animation frame. The baking process is shown in the image below:

1. **[Source]** The editor first take a original spritesheet as the source image
2. **[Edge]** Then apply edge detection algorithm to generate a edge map
3. **[Blur Edge]** The edge map is then gaussian blurred to prevent any "leakage" when baking Distance Field Map later
4. **[SDF]** After that, iterate through all pixels and find the closest edge pixel to generate a Distance Field Map.
5. **[Normal]** Finally, sample the gradient of each pixel of the SDF map to generate a normal map.


{{< figure src="normalbake.png" >}}

### Result

Now we have the normal map, we can simply create a material and assign the source map as the base color and the normal map we just got as the normal.

{{< figure src="normalresult.png" >}}

The result is somewhat decent, considering we haven't simplified the lighting model yet. However, there are some issues:

### Problem 1

The distance in the SDF map is linear, there are obvious peaks at the pixels which have the largest distances. 
{{< figure src="sdfmap.png" >}}

Which as a result, make the normal map looks sharp and not smooth.

{{< figure src="editornormal.png" >}}

We could apply a spherical falloff to the distance to make the normal map looks smoother, but that leads us to the second problem:

### Problem 2

Controllability. 

Baking the normal map basically means that we are committed to a full PBR workflow. Which means the 2D artist will have no control over the final result of the shadow. Of course, we can always manually tweak the normal map, but that's not very intuitive for the 2D artist.


## Trial #2: Mask Based Shadow


{{< figure src="spherelight.png" >}}

### Method

The mask method works quite straightforwardly. We first create a mask for each frame of the animation, then offset the mask a little bit based on the lighting information. The mask is marked as the lit area while the rest is marked as shadow.

{{< figure src="2ndmethod.png" >}}


