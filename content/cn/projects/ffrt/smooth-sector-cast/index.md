---
title: "开发日志：平滑扇区射线投射"
description: "一种实时扇区射线投射方法，具有平滑的中心角"
date: "2024-05-16T17:45:58+0000"
type: page
image: "/images/ffrt/fancasticon.png"
---

## 目标

游戏项目 Fluffy Floaty Rubber Tennis 是一个基于物理的固定摄像机太空网球游戏。玩家唯一的控制是**挥动**球拍。

我选择的球拍挥动时与其他物体交互的逻辑是对球拍击中的物体施加一个力，并对玩家施加一个反作用力。

## 为什么不用 Physics.SphereCast？

那么.. 为什么不用内置的 Unity 物理函数 `Physics.SphereCast` 呢？

也许我只是不了解它背后的真实逻辑，但我认为当击中距离等于 0 时它是有问题的。

如下图所示，当玩家向左挥动时，球拍的起始点是角色的原点，终点比球拍稍远一点。预期的碰撞点发生在球拍右侧的长障碍物和球拍的“第一个球体”之间。由于球拍在到达障碍物时的移动距离为 0，因此预期的击中距离也应为 0。

{{< figure src="sphere-cast.png" >}}

到目前为止，一切对我来说都很合理。实际上，球拍返回的击中距离确实为 0，正如预期的那样，但是击中点在 `Vector3.Zero`！

我做了多次测试，结果是一致的。只要击中距离为 0，击中点总是在 `Vector3.Zero`...

## 构建新解决方案

由于内置函数无法按预期工作，我决定使用 `Physics.Raycast` 构建自己的解决方案。

该机制实际上是基于该项目的需求：挥动球拍。

### 直观性

分解问题，这些是使挥动直观的关键点：

- 舒适的挥动应包括角色周围的小圆空间，因此当球与玩家重叠时，玩家应能击中它。
- 挥动应呈扇形，因为挥动本质上是旋转手臂，使球拍覆盖扇形区域。
- 挥动应首先击中最接近瞄准方向或鼠标位置的物体。

因此，击打区域应覆盖如下：

{{< figure src="new-1.png" height="200">}}

### 机制

如上所示，思路是绘制两个同心圆，然后通过圆心绘制两条线来确定扇区角度，然后连接击打点以形成一个带有平滑中心角的扇区区域。

参数为：
1. 重叠圆的半径。
2. 扇区所在圆的半径。（挥动球拍时球拍可以到达的距离）
3. 扇区的角度。

### 射线投射

然后我们可以以定义的射线密度从小圆投射到大圆。在这个项目中，我

{{< figure src="new-2.png" height="200">}}

### 射线排序

如直观性部分所述，射线应按顺序/剔除，以便玩家击中他们想要击中的物体。

假设玩家瞄准右侧，重叠击打区域的绿色圆圈是一个球。没有任何排序的默认行为是相当随机的。例如，在显示的图像中，射线可以从底部投射到顶部，从顶部投射到底部，甚至从中间到边缘。

{{< figure src="new-4.png" height="200">}}

{{< figure src="new-3.png" height="200">}}

在这种情况下，我决定添加三种排序方法：

1. **距离排序**：根据到扇区中心的距离对射线进行排序。

2. **角度排序**：根据射线与瞄准方向之间的角度对射线进行排序。

3. **首次/最后击中排序**：根据射线投射的顺序对射线进行排序。

## 实施

经过一些调试，结果相当不错，至少在游戏性方面是这样。

这是不同排序方法的完整对比：

<div style="display: flex; flex-wrap: wrap; justify-content: center; gap: 5px;">
    <div style="display: flex; flex-direction: column; align-items: center; width: 300px;">
        <img src="first-hit.png" alt="first hit" style="width: 100%; height: auto; aspect-ratio: 1 / 1;">
        <div style="margin-top: 0px; text-align: center;">首次击中</div>
    </div>
    <div style="display: flex; flex-direction: column; align-items: center; width: 300px;">
        <img src="last-hit.png" alt="last hit" style="width: 100%; height: auto; aspect-ratio: 1 / 1;">
        <div style="margin-top: 0px; text-align: center;">最后击中</div>
    </div>
    <div style="display: flex; flex-direction: column; align-items: center; width: 300px;">
        <img src="dist-min.png" alt="distance min" style="width: 100%; height: auto; aspect-ratio: 1 / 1;">
        <div style="margin-top: 0px; text-align: center;">最小距离</div>
    </div>
    <div style="display: flex; flex-direction: column; align-items: center; width: 300px;">
        <img src="dist-max.png" alt="distance max" style="width: 100%; height: auto; aspect-ratio: 1 / 1;">
        <div style="margin-top: 0px; text-align: center;">最大距离</div>
    </div>
    <div style="display: flex; flex-direction: column; align-items: center; width: 300px;">
        <img src="angle-min.png" alt="angle min" style="width: 100%; height: auto; aspect-ratio: 1 / 1;">
        <div style="margin-top: 0px; text-align: center;">最小角度</div>
    </div>
    <div style="display: flex; flex-direction: column; align-items: center; width: 300px;">
        <img src="angle-max.png" alt="angle max" style="width: 100%; height: auto; aspect-ratio: 1 / 1;">
        <div style="margin-top: 0px; text-align: center;">最大角度</div>
    </div>
</div>

## 想在游戏中看到这个工作成果吗？
自己试玩游戏吧！

<iframe frameborder="0" src="https://itch.io/embed/2707388?bg_color=ffffff&amp;fg_color=2d2d2d&amp;link_color=ea5858&amp;border_color=cccccc" width="552" height="167"><a href="https://secondrealmstudio.itch.io/ffrt">Fluffy Floaty Rubber Tennis by Second Realm Studio, Charlotte Crosland, Yyuk1, Wonderboiz, Rina, sky-haihai</a></iframe>
