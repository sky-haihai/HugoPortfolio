---
title: "Harmony Script: Adding Placeholder Drawings to Empty Timeline Cells"
slug: "Harmony Script Adding Placeholder Drawings to Empty Timeline Cells"
date: 2023-10-23T09:35:14-04:00
tags: ["Toom-Boom Harmony", "Animation", "Script"]
categories: ["Animation"]

draft: false
---

## Introduction

In this post, I will introduce a script I wrote for Toon-Boom Harmony that can be used to fill empty timeline cells with a placeholder drawing named "00" in order to save lots of time for animators. This is my first time writing a script for Harmony 20, so along with the script, I will also talk about what I learnt about the scripting module of Harmony from the past few days and some of the challenges I encountered.

## Installation

1. Download .zip file from the Github [Release](https://github.com/sky-haihai/SKY_Fill_Empty_Frames/releases) page
2. Unzip everything directly into script folder(don't worry about script-icons folder, it will merge with other icons just fine)
3. In case you don't know the address, here you go: https://docs.toonboom.com/help/harmony-20/premium/scripting/import-script.html
4. Open Harmony, right-click on any toolbar, Click "Customize" at the bottom and search for SKY_Fill_Empty_Frames
5. Select the search result and move it from left to right
6. And you are good to go, simply select the nodes you want to fill and click the button you just added to the toolbar

Final setup should look like this:  
{{< figure src="usage.png" >}}

```
if worked:
    Congratulations! You have successfully installed the script.
else:
    Feel free to dm me on discord: sky_haihai
```

## Why The Rest of This Post Exists

Harmony is a great tool for animation and rigging. However, the structure might be intuitive for animators but not for me (as a unemployed programmer). I had to gather information for different concepts from every corner of the internet. To be fair, this is what I do for 80% of the time while programming. But since I found Harmony being particularly hard to understand, Let me share what I learnt from the past few days.

First comes with simple things:

1. Do yourself a favor, use DOM-based Library [_OpenHarmony_](https://github.com/cfourney/OpenHarmony), it's basically a wrapper around the official API, but much more intuitive.
2. [API Document](https://docs.toonboom.com/help/harmony-20/scripting/script/index.html) have different versions, make sure to check the version number
3. External code editors are supported.

And then, here comes the pain.

## The Pain

Scene, Node, Layer, Column, Element, Drawing, Cell, Frame, Exposure, Timeline...

~~I'm not going to explain what they are, because I don't know. But I can tell you what I know about them.~~

## Start from Scene

Scene is basically a root container of literally everything. Think it as the root folder of your project.

Access it through _OpenHarmony_ like this:

```js
var scene = $.scn;
```

The next layer inside scene is Nodes.
{{< figure src="scene-nodes.png" >}}

## Node, Column, Layer

There are many terms that are essentially the same in Harmony, or at least synced with each other. For example, **Layer** is just another name of **Column**. In the timeline view, each "track" is called a **Layer**, while in the Xsheet view, each "column" is literrally called a **Column**.

Nodes are more complex and contain more information inside them. Each node may have multiple linked Columns.

In this script, the focus is on drawing nodes, a.k.a DrawingNode in _OpenHarmony_. Each drawing node possesses a single timing column, a.k.a a DrawingColumn in _OpenHarmony_. However, it's possible for drawing nodes to have additional columns linked to them, even though they are not required for this script.

In other words, nodes possess columns like scenes possessing nodes.

Here's an example to get the first selected drawing node and its column:

```js
var scene = $.scn;
var selectedNodes = scene.selectedNodes;
if (selectedNodes.length == 0) {
    $.alert("Please select at least one drawing node");
    return;
}

var firstDrawingNode;
for (var j = 0; j < selectedNodes.length; j++) {
    selectedNode = selectedNodes[j];
    //READ is the type name for Drawing nodes
    if (selectedNode.type != "READ") {
        continue;
    }
    firstDrawingNode = selectedNode;
    break;
}

var linkedDrawingColumn = firstDrawingNode.attributes.drawing.element.column;
```

## Cell, Frame, Exposure

-   A **Cell** is a single frame of a layer regardless of the type of the layer or whether it's empty or not.
-   A **Frame** is a single frame of a layer that contains a drawing. An exposure is a single frame of a layer that contains a drawing and is not empty.
-   **Exposure** just refers to how long is the previous keyframe lasts.

## Element

Element is the parent of Drawings. Theoratically speaking, elements can be reused in multiple drawing nodes.

## Drawing (A Tricky One)

While it's almost the end of this article, drawing has always been the trickiest term in Harmony in my opinion. It "could" refer to a single image file on your disk, a single frame of a layer, or a collection of multiple image files. In fact, I'm still not exactly sure what a "Drawing" is if I'm given a random documentation with limited context now. This term literally can be used anywhere in any animation working scenario.

But tangling about a term is just waste of time. The need is simple and clear!

Replace the path of drawings of any empty cells on the timeline with "00"!

Just Do It!

## The Happiness

Like many other projects I did in the past, the happiness always comes after the struggle. Although I'm still not sure what a "Drawing" is and how some of the other terms relate to each other, I'm happy that I can finally write a script that can save my animators some time.

During the process, I learnt a lot from other fantastic animators and programmers who open-sourced their script for free. They are the true [heroes](#references).

## Final Code

```js
include("openHarmony.js");

function SKY_Fill_Empty_Frames() {
    $.beginUndo("fillFrame");

    var scene = $.scn;
    var selectedNodes = scene.selectedNodes;
    if (selectedNodes.length == 0) {
        $.alert("Please select at least one drawing node");
        return;
    }

    for (var j = 0; j < selectedNodes.length; j++) {
        var sNode = selectedNodes[j];
        if (sNode.type != "READ") {
            $.debug("Skipping node: " + sNode.type);
            continue;
        }

        var drawingAttr = sNode.attributes.drawing.element;

        for (var i = 0; i < scene.length; i++) {
            var drawing = sNode.getDrawingAtFrame(i + 1);
            if (!drawing.path) {
                drawingAttr.setValue("00", i + 1);
                $.debug("Adding 00 at Frame: " + (i + 1));
            } else {
                $.debug("Skipping Frame: " + (i + 1));
            }
        }

        $.log("Fill Finished");
        $.endUndo();
    }
}
```

## References

BAKE PARENT TO DRAWINGS Source Code - Yu Ueda
http://raindropmoment.com/2019/06/25/bake-parent-to-drawings/

OpenHarmony API Documentation - OpenHarmony
https://cfourney.github.io/OpenHarmony/

About Layers and Columns - Harmony 20 User Guide  
https://docs.toonboom.com/help/harmony-20/premium/layers/about-layer-column.html

Official API Documentation - Harmony 20 Scripting Interface  
https://docs.toonboom.com/help/harmony-20/scripting/script/index.html
