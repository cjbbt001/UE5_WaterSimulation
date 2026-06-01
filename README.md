# UE5 Interactive Water Simulation

基于 Unreal Engine 5 的交互式水面模拟 Demo。  
使用 `SceneCapture2D`、`Post Process`、`Render Target`、Flipbook 水面材质和材质图计算，实现角色 / 物体与水面的实时交互、波纹扩散、动态法线和水面反光效果。

## Preview

![Demo](./images/demo.gif)

## Features

- Flipbook 实现基础水面流动动画
- Single Layer Water 实现水体散射、吸收和折射表现
- SceneCapture2D 捕获角色 / 物体交互输入
- Post Process 材质将 SceneDepth 转换为交互高度输入
- Render Target 记录交互 Mask、高度场和动态法线
- 三缓冲 Height RT 实现波纹扩散、振荡和衰减
- 高度图转法线，用于水面反光和折射细节
- 水面材质同时接入 `World Position Offset` 和 `Normal`
- 支持不同 Actor 进入水面区域后参与交互

## Pipeline

```text
Base Water:
Flipbook Height / Normal
→ Water Surface Material

Interaction:
Actor / Character
→ SceneCapture2D
→ M_SceneDepth
→ RT_Capture
→ M_WaveCompute
→ Height RT Triple Buffer
→ M_WaveSimulation
→ M_WaveNormal
→ RT_HeightNormal
→ Water Surface Material
→ WPO + Normal
````

## Technical Overview

### Base Water Material

基础水面使用 Flipbook 播放高度图和法线图，形成持续流动的水面细波纹。
水面材质使用 `Single Layer Water`，通过散射、吸收、粗糙度、高光和折射控制水体质感。

### Interaction Capture

`SceneCapture2D` 从水面上方捕获进入交互范围的 Actor，并写入 `RT_Capture`。
`SimulationCaptureVolume` 负责管理可交互对象，并将其加入 SceneCapture 的 `Show Only List`。

### Depth Input

`M_SceneDepth` 作为 SceneCapture 的 Post Process Material，读取 `SceneDepth` 并输出深度图。
`M_WaveCompute` 将深度值映射为负高度，使角色 / 物体区域对水面产生下压效果。

```text
SceneDepth
→ Normalize
→ Height Input
→ Height RT
```

### Wave Simulation

`M_WaveSimulation` 使用上一帧和上上帧高度图进行邻域采样，模拟水波的扩散、振荡和衰减。

```text
NewHeight = (NeighborHeight * 0.5 - PreviousHeight2) * Damping
```

三张 Height RT 循环使用，避免同一张 Render Target 同时读写。

### Normal Generation

`M_WaveNormal` 读取最新 Height RT，通过有限差分计算局部坡度，并生成动态法线贴图 `RT_HeightNormal`。

```text
Height Difference
→ Tangent Vectors
→ Cross Product
→ Normalize
→ RT_HeightNormal
```

### Final Water Surface

最终水面材质将基础 Flipbook 水波和交互波纹叠加：

```text
Flipbook Height + Heightfield
→ World Position Offset

Flipbook Normal + Heightfield Normal
→ Normal
```

`Heightfield` 控制真实几何起伏，`Heightfield Normal` 控制反光和折射细节。

## Key Assets

| Asset                | Description                               |
| -------------------- | ----------------------------------------- |
| `BP_WaterSimulation` | 管理 SceneCapture、Render Target、材质参数和每帧模拟流程 |
| `MF_WaterFlipbook`   | 基础水面 Flipbook 动画采样                        |
| `M_SceneDepth`       | 将 SceneDepth 转换为交互深度输入                    |
| `M_WaveCompute`      | 将捕获结果转换为高度输入                              |
| `M_WaveSimulation`   | 计算波纹扩散、振荡和衰减                              |
| `M_WaveNormal`       | 从高度图生成动态法线                                |
| `M_WaterSimulation`  | 最终水面显示材质                                  |

## Notes

完整学习笔记见：

```text
notes/
```

## Environment

* Unreal Engine 5
* Blueprint
* Material Graph
* Render Target
* SceneCapture2D
* Post Process Material
* Single Layer Water


