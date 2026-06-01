# UE5 Water Simulation

交互式水体模拟项目，基于 Unreal Engine 5 实现。

## 效果概述

基于 Compute Shader 的实时水面波动模拟，支持：
- 鼠标交互产生涟漪
- Flipbook 法线/高度图驱动水面细节
- Render Target 多通道波纹传播

## 项目结构

```
Content/
  WaterSimulation/          # 水体模拟核心资产
    BP_WaterSimulation_BP   # 水体交互蓝图
    waterplan               # 水面平面
    Lvl_ThirdPerson         # 演示关卡
    Function/               # 材质函数
    Materials/              # 水面材质及实例
    Texture/                # Flipbook纹理、Render Target
images/                     # 效果截图
```

## 运行要求

- Unreal Engine 5.3+
- Windows 10/11

## 使用

1. 克隆仓库到 UE5 项目的 `Content/` 目录下
2. 打开 `Lvl_ThirdPerson` 关卡
3. 运行游戏，鼠标点击水面产生涟漪

## License

MIT
