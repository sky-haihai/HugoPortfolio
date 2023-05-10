---
title: "Introducing Xiheframework"
date: 2023-05-05T14:24:13-04:00
tags: ["Unity", "C#"]
categories: ["Game Development"]
draft: false
---

## What is XiheFramework

**XiheFramework** is a game framework for agile development, wrote on top of **Unity** engine. 

## Who Should Use
### Scale  
Ideally at least two members:  
1. Programmer for creating custom modules(game features)
2. Technical designer for node editing

Or use it as an individual developer (~~hair loss rate 200%~~)

### Skills  

- Unity Editor
- Node-based Editor
- C# (Custom Modules)

### Warning :warning:
1. Consider not to use this framework if you are work individually and have **ZERO** programming skill as you probably will have a lot of pain if you just use the node editor for all game logic. However, this is just a warning, feel free to ignore it.

2. This framework is maintained, updated and probably used only by me so... free feel to contribute and prepare for major API changes.

## Where to Download

[Github Repo](https://github.com/sky-haihai/XiheFramework)  
[Documentation](https://sky-haihai.github.io/xiheframework-document/) 

## Why Use It
- Contains out-of-box essential modules for any game. (e.g. Event, FSM, etc..)
- Uses name-based indexing instead of id-based to save you from tedious work of filling out a bunch of excel sheets.
- Intergrates FlowCanvas(Not Included) as a **Service** layer to connect designers to the engine.

## How to Use (Core Modules)
### Audio  
Wrapper of FMOD for Unity. Provides useful functions to play sound with 3D parameters.

### Blackboard  
Runtime Variable Pool.

### Event  
Runtime Event Pool.

### FSM  
Runtime FSM Pool.

### Input  
Encapsulation of Unity Legacy Iput System. Provide useful features like Keyboard remapping and useful functions for mouse movement.

### Localization  
Get text and assets(textures, models) that match the current Language setting.

### Serialization  
Serialize runtime data into binary data to save the game progress.

### UI  
Manage all UI Behaviour(Canvas) states.

Read detailed documentation with use cases

## Dependencies

**FlowCanvas** (Mandatory)  
**FMod** (Mandatory)