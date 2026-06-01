# 1.水面网格和项目设置
## Maya 导出 FBX 设置：切线空间与上方向轴

### 操作
- 在导出 FBX 时，打开导出设置
- 将 **切线空间** 设置为 **左手系**
- 将 **上方向轴（Up Axis）** 设置为 **Z**

### 注释
- **切线空间 = 左手系**
  - 目的是尽量和虚幻引擎的切线/法线空间约定保持一致
  - 可减少法线显示异常、表面细节不对等问题

- **上方向轴 = Z**
  - 目的是和 UE 的 **Z 轴朝上** 规则对齐
  - 可减少导入后模型朝向不一致的问题

## UE 项目前期编辑器设置

### 操作
1. 在 **Editor Preferences** 中关闭 **Live Coding**
2. 关闭 **Automatically Compile Newly Added C++ Classes**
3. 关闭 **Auto Save**
4. 在 **Project Settings** 中将帧率设置为 **60 FPS**

### 注释
- **关闭 Live Coding**：减少热重载带来的编译或状态异常
- **关闭自动编译 C++ Classes**：避免新建 C++ 类时自动触发编译
- **关闭自动保存**：避免教程操作中途被自动保存打断或保存错误状态
- **帧率 60 FPS**：让项目运行节奏更稳定，便于观察效果与调试动画/交互逻辑

# 2.Flipbook动画效果
**Flipbook** 是一种把多帧图片按时间顺序连续播放的动画方式。

可以理解为：

- 一张大贴图里放很多小图
- 每一小格是一帧
- 材质按时间切换不同帧
- 最终形成动画效果
## Flipbook 通用播放节点逻辑
![](images/Pasted image 20260427163011.png)

1. 用 `TexCoord / Frames` 把整张贴图的 UV 缩小到单帧格子大小
2. 用 `Time * Speed` 推动帧序号随时间变化
3. 用 `Floor` 把连续时间转成当前整数帧
4. 用当前帧计算横向和纵向偏移
5. 用 `Append` 合成二维偏移
6. 用 `TexCoord / Frames + Offset` 得到最终采样 UV
7. 将最终 UV 接入 `Texture Sample`，实现 Flipbook 播放

### 注释
- `Frames`：每行/每列的帧数，`8` 表示 `8×8`
- `TexCoord`：原始 UV 坐标
- `Divide`：缩小 UV 到单帧范围
- `Time`：提供动画时间
- `Multiply`：控制播放速度
- `Floor`：得到当前整数帧
- `Frac`：让横向偏移循环回到开头
- `Append`：把横向、纵向偏移合成二维 UV
- `Add`：把单帧 UV 和帧偏移相加
- 核心逻辑：**不是换贴图，而是通过时间改变 UV 采样位置**

# 3.抖动和接缝线问题
- 抖动
![](images/Pasted image 20260427174909.png)
![](images/Pasted image 20260427174927.png =183)
- 接缝
![](images/Pasted image 20260427175916.png)

- WPO高度图
![](images/Pasted image 20260427180729.png =200)
## Flipbook 接缝线与抖动处理逻辑

1. 使用 `TexCoord * TileSize` 让 Flipbook 在水面上平铺
2. 使用 `Frac` 将平铺后的 UV 折回 `0~1` 范围
3. 使用 `TileSize / 512 * SeamlineOffset` 计算接缝偏移量
4. 使用 `OneMinus` 缩小边界采样范围
5. 将采样区域往帧格内部收缩，避免采到相邻帧
6. 在 `Texture Sample` 中将 **MipValueMode** 改为 **Derivative / 显式导数**
7. 暴露 `DDX(UVs)` 和 `DDY(UVs)` 输入
8. 在 `DDX / DDY` 前除以 `Frames`
9. 将修正后的 `DDX / DDY` 接入 `Texture Sample`
10. 让 GPU 按单帧大小判断 Mip 级别，减少接缝和抖动

### 注释
- `TileSize`：控制 Flipbook 在水面上平铺多少次
- `Frac`：把 `TexCoord * TileSize` 后超出 `0~1` 的 UV 折回局部坐标
- `/512`：将像素级偏移转换为 UV 偏移，`512` 通常对应贴图尺寸
- `SeamlineOffset`：控制接缝线向内部偏移的强度
- `OneMinus`：缩小采样边界，减少采到帧格边缘
- `MipValueMode = Derivative / 显式导数`：用于手动指定纹理采样的导数信息
- `DDX / DDY`：告诉 GPU 当前 UV 在屏幕空间的变化速度
- `DDX / Frames`、`DDY / Frames`：让导数匹配单帧大小，避免按整张 Flipbook 错选 Mip
- 核心逻辑：**Frac 处理平铺边界，偏移和缩边处理帧格边界，显式 DDX/DDY 修正 Mip 采样**

# 4.平滑动画

![](images/Pasted image 20260429143742.png)

## Flipbook 帧间插值

1. 使用 `Time * PlayRate` 得到动画播放进度。
2. 使用 `Floor(Time * PlayRate)` 得到当前帧编号。
3. 使用 `Floor(Time * PlayRate + 1)` 得到下一帧编号。
4. 分别计算当前帧 UV 和下一帧 UV。
5. 用两个 `Texture Sample` 分别采样当前帧和下一帧。
6. 使用 `Frac(Time * PlayRate)` 作为 `Lerp` 的 Alpha。
7. 将当前帧和下一帧平滑混合。
8. 混合结果再乘以 `WPO Height`，用于水面高度扰动。

### 注释

- `Floor` 负责取整数帧编号。  
- `Frac` 负责取小数部分，表示当前帧过渡到下一帧的比例。  
- 这里的 `Frac` 是时间插值权重，不是空间遮罩。  
- 两次 Texture Sample 可以让 Flipbook 动画从逐帧跳变变成平滑过渡。

法线贴图同样处理

## 封装Flipbook函数
![](images/Pasted image 20260429153644.png)
![](images/Pasted image 20260501161939.png)
### Material Function 中的显式导数开关

1. 将 Flipbook 采样逻辑封装成 Material Function。
2. 输入纹理、UV、Frames、Tile Size、Seamline Offset、PlayRate 等参数。
3. 内部计算当前帧 UV 和下一帧 UV。
4. 同时准备普通 Texture Sample 和 Derivative Texture Sample 两套采样路径。
5. 使用 `Static Bool Parameter` 作为是否启用导数的开关。
6. 使用 `Static Switch` 在普通采样和显式导数采样之间切换。
7. 最后使用 `Lerp` 混合当前帧和下一帧，实现 Flipbook 帧间插值。

#### 注释

- `Static Switch` 是编译期分支，不是运行时逐像素判断。  
- 显式导数主要用于处理 `Frac`、Tile、Flipbook 分帧等 UV 跳变导致的 Mip 误判。  
- 不是所有贴图都必须使用 Derivative，所以用开关控制更灵活。  
- 更准确的说法不是“区分贴图”，而是“区分这张贴图是否需要显式导数采样”。


# 5.单层水材质
- shading mode 设置为单层水材质
## 设置 Single Layer Water Shading Model

1. 在 Details 面板中找到 `Material` 设置。
2. 将 `Shading Model` 设置为 `Single Layer Water`。

### 注释

- `Single Layer Water` 是 UE 中专门用于水体表现的 Shading Model。
- 它不是普通半透明材质，而是通过散射、吸收等参数模拟水的颜色、深度和透光感。
- 切换到 `Single Layer Water` 后，主材质节点会暴露水体专用输入。
- 如果不切换到 `Single Layer Water`，`Scattering` 和 `Absorption` 这类参数无法按教程逻辑发挥作用。
- 这一步主要负责水体的“光学表现”，不是波纹交互逻辑。

### Single Layer Water 节点介绍

1. `Scattering Coefficients` 控制水体散射光的颜色和强度。
2. `Absorption Coefficients` 控制水体吸收光的颜色和强度。
3. `Phase G` 控制光在水体中的散射方向倾向。
4. `Color Scale Behind Water` 控制水体后方物体透过水面时的颜色缩放。

## Single Layer Water 颜色与法线控制

![](images/屏幕截图 2026-05-01 164207.png)

![](images/屏幕截图 2026-05-01 164145.png)

1. 使用 `Scattering` 参数控制水体散射颜色。
2. 将 `Scattering * Scattering Strength` 接入 `ScatteringCoefficients`。
3. 使用 `Absorption` 参数控制水体吸收颜色。
4. 将 `Absorption * Absorption Strength` 接入 `AbsorptionCoefficients`。
5. 使用 `MF_WaterFlipbook` 输出动态法线。
6. 将动态法线接入 `FlattenNormal`。
7. 使用 `Flatten Strength` 控制法线扰动强弱。
8. 将处理后的法线继续接入水材质 Normal。

### 注释

- `Scattering` 控制水把光散出来后的颜色倾向，影响水体亮度和浑浊感。  
- `Absorption` 控制水吸收光的程度，影响水体深度感和透明度。  
- `FlattenNormal` 只影响光照法线，不影响真实几何高度。  
- WPO 决定水面轮廓起伏，Normal 决定水面反光细节。


## 使用 Height Weight 控制水面材质属性

![](images/Pasted image 20260501172018.png)
1. 使用 `Height Weight` 作为高度权重遮罩。
2. 将 `Height Weight` 接入 `Power` 的 `Base`。
3. 使用 `Height Falloff` 控制 `Power` 的 `Exp`。
4. 将 `Power` 输出作为多个 `Lerp` 的 `Alpha`。
5. 使用 `Lerp` 混合 `Specular-surface` 和 `Specular-wave`。
6. 将混合结果接入 `Specular`。
7. 使用 `Lerp` 混合 `Roughness-surface` 和 `Roughness-wave`。
8. 将混合结果接入 `Roughness`。
9. 使用 `Lerp` 混合 `Opacity-surface` 和 `Opacity-wave`。
10. 将混合结果接入 `Opacity`。

### 注释

- `Height Weight` 可以理解为水面高度遮罩，用来区分平静水面和波纹区域。
- `Power` 用来调整高度遮罩的对比度。
- `Height Falloff` 越大，波纹影响范围越集中。
- `Specular-surface` 控制平静水面的高光强度。
- `Specular-wave` 控制波纹区域的高光强度。
- `Roughness-surface` 控制平静水面的粗糙度。
- `Roughness-wave` 控制波纹区域的粗糙度。
- `Opacity-surface` 控制平静水面的透明度。
- `Opacity-wave` 控制波纹区域的透明度。
- 这组节点的作用是让水面高度变化同时影响反光、粗糙度和透明度。

## Pixel Normal Offset 折射方式

1. 打开水面材质。
2. 点击材质图空白处。
3. 在 Details 面板中找到折射相关设置。
4. 将折射方法设置为 `Pixel Normal Offset`。
5. 将 `Refraction` 设置为 `0.88`。
6. 配合动态法线使用，让水面后方画面产生轻微扭曲。

### 注释

- `Pixel Normal Offset` 是基于像素法线偏移的折射方式。
- 它会根据每个像素的法线方向偏移屏幕后方画面，从而制造水体折射感。
- 它不是严格物理折射，而是实时渲染中的近似方法。
- 当前水面材质已有动态法线，所以适合使用 `Pixel Normal Offset`。
- 法线变化越明显，折射扭曲越明显。
- `Refraction = 0.88` 是比较克制的折射强度，避免背景被过度拉扯。
- 它主要影响水下画面的扭曲，不负责真实几何起伏。

## 水材质参数效果分析

1. `Scattering * Scattering Strength` 控制水体散射。
2. 当前散射强度很低，使水体整体偏暗、低饱和、不发亮。
3. `Absorption * Absorption Strength` 控制水体吸收。
4. 当前吸收强度高于散射，使水体具有厚度感和深度感。
5. `Specular-surface = 0.2`，`Specular-wave = 0.8`。
6. 平静水面高光较弱，波纹区域高光更强。
7. `Roughness-surface = 0.0`，`Roughness-wave = 0.3`。
8. 平静水面更光滑，波纹区域反射更破碎。
9. `Opacity-surface = 0.0`，`Opacity-wave = 0.1`。
10. 波纹区域透明度轻微变化，用于增强波纹可见度。
11. `Height Falloff = 2.0`，通过 `Power` 收窄高度遮罩影响范围。
12. `Flatten Strength = 1.25`，压平动态法线，避免反光过碎。

### 注释

- 这套参数的核心策略是：水体基础层压暗，波纹区域局部增强。
- `Scattering` 很低，避免水面整体发亮。
- `Absorption` 稍强，让水有深度和厚度感。
- `Specular` 用高度权重混合，让波峰区域更容易产生高光。
- `Roughness` 用高度权重混合，让波纹区域反射更碎、更自然。
- `Opacity` 只轻微增强波纹区域，避免水面像塑料或果冻。
- `Height Falloff` 控制波纹影响范围，数值越大，影响越集中。
- `FlattenNormal` 控制法线强度，避免动态法线过强导致水面显假。
- 整体效果不是靠单个参数，而是靠水体颜色、法线、高光、粗糙度和高度权重共同配合。

![](images/Pasted image 20260502141319.png)

# 6.纹理绘制
## 水面交互制作步骤  
  
1. 使用场景捕获组件捕获角色与水面网格的交汇处。  
2. 将交互区域写入权重纹理。  
3. 通过纹理平移计算，把世界空间中的交互位置转换到水面纹理空间。  
4. 基于权重纹理进行模拟波纹扩散处理。  
5. 根据波纹高度生成或处理模拟波纹法线。  
6. 将波纹高度、法线和权重信息输入水面材质。  
7. 通过水面材质驱动水面网格表现交互效果。

## 创建水面交互 Actor 蓝图

![](images/屏幕截图 2026-05-03 152606.png =289)
![](images/屏幕截图 2026-05-03 152627.png)

1. 创建一个蓝图类，并继承自 `Actor`。
2. 在蓝图中添加 `Plane` 组件，作为水面网格。
3. 保留 `DefaultSceneRoot` 作为默认根组件。
4. 在变量分类中创建 `WaterSimulation / Materials` 分类。
5. 创建 `M_Watersurface` 变量，用于保存水面材质引用。
6. 将 `M_Watersurface` 的默认值设置为 `M_Watersurface_Test`。

### 注释

- 继承 `Actor` 的蓝图可以作为一个可放入关卡的水面交互系统容器。
- `Plane` 是水面显示和承载材质的网格。
- `M_Watersurface` 不是动态材质，而是原始水面材质模板。
- 使用变量保存材质引用，方便后续替换材质和创建动态材质实例。

## 创建 Render Target 权重纹理

![](images/屏幕截图 2026-05-03 153620.png)
![](images/屏幕截图 2026-05-03 153655.png =477)

1. 创建一个 `Texture Render Target 2D`。
2. 将尺寸设置为 `128 × 128`。
3. 将 `Address X` 设置为 `Wrap`。
4. 将 `Address Y` 设置为 `Wrap`。
5. 将 `Render Target Format` 设置为 `RTF R8`。
6. 在水面测试材质中创建 `Texture Sample Parameter2D`。
7. 将参数命名为 `HeightField`。
8. 将 `RT_Capture` 指定给 `HeightField`。
9. 将 `HeightField` 的输出临时接入 `Base Color` 进行调试显示。

### 注释

- `RT_Capture` 是运行时可写入的动态纹理。
- 它用于记录水面交互权重或高度信息。
- `128 × 128` 分辨率较低，但适合第一版实时交互测试。
- `RTF R8` 只保存一个灰度通道，适合存储权重、Mask 或 HeightField。
- 将它接到 `Base Color` 是为了确认材质能正确读取 Render Target。
- 当前为黑色通常说明 Render Target 还没有被写入有效内容。

## 创建并赋予动态材质实例

![](images/屏幕截图 2026-05-03 161709.png)

1. 打开水面 Actor 蓝图的 `Construction Script`。
2. 从 `Plane` 组件拉出，创建 `Create Dynamic Material Instance` 节点。
3. 将 `M_Watersurface` 连接到 `Source Material`。
4. 保持 `Element Index = 0`。
5. 将返回值保存为变量 `MID Watersurface`。
6. 使用 `Set Material` 节点。
7. 将 `Plane` 连接到 `Set Material` 的 `Target`。
8. 将 `MID Watersurface` 连接到 `Material`。
9. 编译并保存蓝图。

### 注释

- `Create Dynamic Material Instance` 会基于原始材质创建一个可动态修改参数的材质实例。
- `M_Watersurface` 是材质模板，`MID Watersurface` 是当前蓝图实例专属的动态材质。
- 保存 `MID Watersurface` 是为了后续通过蓝图修改材质参数。
- 后续可以通过 `MID Watersurface` 设置 `HeightField`、波纹强度、水面范围、捕获偏移等参数。
- `Set Material` 的作用是让 `Plane` 真正使用这个动态材质实例。
- 这一步是在建立“蓝图 Actor → 动态材质 → 水面 Plane”的控制链路。

## 设置 SceneCaptureComponent2D

1. 在 `BP_WaterSimulation_BP` 中添加 `SceneCaptureComponent2D` 组件。
2. 将它放在水面 Plane 上方。
3. 设置位置为 `Location = (0, 0, 5)`。
4. 设置旋转为 `Rotation = (0, -90, -90)`，让它垂直朝向水面。
5. 将 `Projection Type` 设置为 `Orthographic`。
6. 将 `Ortho Width` 设置为 `800`。
7. 将 `Primitive Render Mode` 设置为 `Use ShowOnly List`。
8. 将 `Capture Source` 设置为 `Final Color (LDR) in RGB`。
9. 开启 `Capture Every Frame`。
10. 关闭 `Capture on Movement`。
11. 在 `Post Processing Show Flags` 中关闭会污染数据图的后处理项：
    - `Bloom`
    - `Eye Adaptation`
    - `Lens Distortion`
    - `Local Exposure`
    - `Motion Blur`
    - `Post Process Material`
    - `Tonemapper`

### 注释

- `SceneCaptureComponent2D` 相当于一台俯视水面的捕获相机。
- 它不直接显示到屏幕，而是把捕获结果写入 `RT_Capture`。
- `Orthographic` 正交投影不会产生近大远小，适合生成水面交互权重图。
- `Ortho Width` 决定捕获区域在世界空间中的覆盖范围。
- `Use ShowOnly List` 用于只捕获指定对象，避免无关场景物体进入 RT。
- `Final Color (LDR) in RGB` 会捕获最终颜色结果，适合教程当前阶段生成角色权重图。
- `Capture Every Frame` 用于每帧更新角色位置，保证 RT 随角色移动实时刷新。
- `Capture on Movement` 当前不需要，避免移动组件时额外触发捕获。
- UE5.7 中开启 `Tonemapper` 后，连续捕获可能出现异常光圈污染，因此当前保持关闭。
- 关闭 Tonemapper 后 RT 可能偏灰，后续通过 `Multiply`、`Power`、`Saturate` 等材质节点手动拉对比。

## 初始化 SceneCapture 与 RT_Capture

![](images/屏幕截图 2026-05-03 183210.png)

1. 在 `BeginPlay` 中使用 `Sequence` 分出初始化流程。
2. 使用 `Clear Render Target 2D` 清空 `RT_Capture`。
3. 将 `Clear Color` 设置为黑色。
4. 使用 `Set Texture Target`，把 `RT_Capture` 设置为 `SceneCaptureComponent2D` 的输出目标。
5. 使用 `Get Player Pawn` 获取当前玩家。
6. 使用 `Show Only Actor Components`，让 `SceneCaptureComponent2D` 只捕获玩家相关组件。
7. 保持 `Include from Child Actors` 关闭。

### 注释

- `RT_Capture` 是用来记录角色与水面交互区域的动态纹理。
- `Clear Render Target 2D` 用于初始化 RT，避免残留上一轮运行的数据。
- `Set Texture Target` 的作用是告诉 `SceneCaptureComponent2D` 把捕获画面写入哪张 Render Target。
- `Show Only Actor Components` 用于限制捕获对象，避免地面、天空、水面 Plane 等无关物体污染权重图。
- `Get Player Pawn` 表示当前只捕获玩家角色，用于生成交互输入。
- 这一步建立了基础链路：`玩家角色 → SceneCapture 捕获 → RT_Capture 写入 → 水材质读取`。
- 当前 UE5.7 中 `Tonemapper` 开启会导致连续捕获出现异常光圈，因此保持关闭，用后续材质节点手动增强权重。

## 创建 WaveCompute 计算材质

1. 创建材质 `M_WaveCompute`。
2. 将 `Material Domain` 设置为 `Surface`。
3. 将 `Blend Mode` 设置为 `Additive`。
4. 将 `Shading Model` 设置为 `Unlit`。
5. 勾选 `Allow Negative Emissive Color`。
6. 创建 `Texture Sample Parameter2D`，命名为 `Capture`。
7. 将 `Capture.R` 作为基础输入，用于读取 `RT_Capture` 的角色捕获信息。
8. 将处理后的结果接入 `Emissive Color`。

### 注释

- `M_WaveCompute` 不是最终水面材质，而是用于写入 Render Target 的计算材质。
- `Unlit` 用于避免光照影响，保证输出更像数据图。
- `Additive` 用于把每一帧的角色捕获结果累积到高度 RT 中，从而形成路径绘制。
- `Allow Negative Emissive Color` 为后续波纹正负高度变化做准备。
- 当前阶段它主要负责把 `RT_Capture` 转换并写入 `RT_Height_0`。

## 创建 MID_WaveCompute
![](images/Pasted image 20260503193912.png)
1. 在蓝图中创建变量 `M_WaveCompute`，类型为材质引用。
2. 将默认值设置为 `M_WaveCompute` 材质。
3. 在 `Event Graph` 的初始化流程中创建 `Create Dynamic Material Instance`。
4. 将 `M_WaveCompute` 接入 `Parent`。
5. 将返回值保存为变量 `MID_WaveCompute`。

### 注释

- `M_WaveCompute` 是计算材质模板。
- `MID_WaveCompute` 是运行时可传参的动态材质实例。
- 这个材质实例不需要赋给水面 Plane。
- 后续它会被 `Draw Material` 画入 Render Target，用来更新波纹高度图。

## 创建 RT_Height_0

1. 创建新的 `Texture Render Target 2D`，命名为 `RT_Height_0`。
2. 将尺寸设置为 `1024 × 1024`。
3. 将 `Address X` 设置为 `Wrap`。
4. 将 `Address Y` 设置为 `Wrap`。
5. 将 `Render Target Format` 设置为 `RTF R16f`。
6. 在蓝图中创建变量 `RT_Height_0`，并指定默认值。

### 注释

- `RT_Capture` 用于记录当前帧角色捕获信息。
- `RT_Height_0` 用于累积绘制后的高度 / 路径信息。
- `1024 × 1024` 比 `RT_Capture` 更精细，适合保存波纹高度。
- `RTF R16f` 是单通道 16 位浮点格式，比 `R8` 更适合保存高度场。
- `Wrap` 会让边界外 UV 循环采样，后续如果边缘出现回绕问题需要回头检查。

## 将 WaveCompute 绘制到 RT_Height_0
![](images/Pasted image 20260503193846.png)
1. 使用 `Set Texture Parameter Value`。
2. 将 `MID_WaveCompute` 的 `Capture` 参数设置为 `RT_Capture`。
3. 使用 `Begin Draw Canvas to Render Target`。
4. 将目标 Render Target 设置为 `RT_Height_0`。
5. 使用 `Draw Material`。
6. 将 `Render Material` 设置为 `MID_WaveCompute`。
7. 将 `Screen Position` 设置为 `(0, 0)`。
8. 将 `Screen Size` 连接为 `Begin Draw Canvas to Render Target` 输出的 `Size`。
9. 使用 `End Draw Canvas to Render Target` 提交绘制结果。
10. 将 `RT_Height_0` 通过 `Set Texture Parameter Value` 传给 `MID_Watersurface` 的 `HeightField` 参数。

### 注释

- `Begin Draw Canvas to Render Target` 表示开始向 RT 绘制。
- `Draw Material` 会把 `MID_WaveCompute` 的输出画满整张 `RT_Height_0`。
- `End Draw Canvas to Render Target` 用于结束并提交绘制。
- 这一步相当于用材质在 GPU 上执行一次计算，并把结果保存到 `RT_Height_0`。
- `MID_Watersurface.HeightField = RT_Height_0` 用于让测试平面 / 水面材质读取高度图结果。

## 处理 RT_Capture 背景不纯黑的问题

1. 发现 UE5.7 中 `RT_Capture` 无法像教程 UE5.4 一样直接得到纯黑背景。
2. 确认问题来自 `SceneCapture2D + Capture Every Frame + Tonemapper / 后处理` 的差异。
3. 关闭 `Tonemapper` 后，异常光圈消失，但 `RT_Capture` 会偏灰。
4. 在 `M_WaveCompute` 中不再直接使用 `Capture.R` 输出。
5. 将 `Capture.R` 接入 `Subtract`，先减去背景灰度底噪。
6. 将结果接入 `Saturate`，把小于 0 的背景压成 0。
7. 再使用 `Multiply` 放大人物区域。
8. 最后再接入 `Saturate`，把结果限制在 `0~1`。
9. 将处理后的结果接入 `Emissive Color`。
10. 使用 `Additive` 将处理后的角色 Mask 累积到 `RT_Height_0`。

### 注释

- 问题根源是 `RT_Capture` 背景不是纯黑，不能直接参与 Additive 累积。
- 如果直接将灰底 `RT_Capture` 累积到 `RT_Height_0`，背景灰度会逐帧叠加，最终整张 RT 变白。
- `Subtract` 用于去除背景底噪。
- `Saturate` 用于把负值压回 0，并防止数值超过 1。
- `Multiply` 用于增强人物区域权重。
- 当前修正后的逻辑是：`RT_Capture` 不必完美干净，但进入高度 RT 前必须先变成稳定的交互 Mask。
- 这一步比直接依赖 SceneCapture 输出黑底图更稳定，也更适合不同 UE 版本。

## 纹理绘制路径逻辑

`RT_Capture` 记录当前帧玩家的位置，相当于“这一帧的输入图”。

`M_WaveCompute` 读取 `RT_Capture`，先用 `Subtract + Saturate + Multiply + Saturate` 去掉灰底、增强玩家区域，把它处理成干净的交互 Mask。

随后通过 `Draw Material to Render Target`，把处理后的结果绘制到 `RT_Height_0`。

因为 `M_WaveCompute` 使用 `Additive` 混合，所以每一帧的 Mask 都会叠加到 `RT_Height_0` 上：
```
RT_Height_0 = Capture_1 + Capture_2 + Capture_3 + ...
```

# 7.纹理平移对齐


## SceneCapture 跟随玩家

![](images/Pasted image 20260505214303.png)
### 操作

1. 在蓝图中获取 `Get Player Pawn`。
2. 从 `Get Player Pawn` 拉出 `Get Actor Location`，获取玩家世界坐标。
3. 获取 `SceneCaptureComponent2D` 的 `Get World Location`。
4. 将玩家位置的 `X` 接入 `Set World Location` 的 `New Location X`。
5. 将玩家位置的 `Y` 接入 `Set World Location` 的 `New Location Y`。
6. 将 `SceneCaptureComponent2D` 原本位置的 `Z` 接入 `New Location Z`。
7. 将 `Set World Location` 的 `Target` 设置为 `SceneCaptureComponent2D`。
8. 将这段逻辑放入每帧更新流程中，让 SceneCapture 持续跟随玩家。

### 注释

- 这一步的目的是让 `SceneCaptureComponent2D` 始终跟随玩家的水平位置。
- 只使用玩家的 `X / Y`，不使用玩家的 `Z`，避免捕获相机高度跟着角色上下变化。
- `SceneCaptureComponent2D` 的 `Z` 保持原高度，用来维持俯视捕获状态。
- 这样可以保证玩家始终处在 `RT_Capture` 的捕获范围内。
- 这一步只解决“当前帧能捕获到玩家”的问题。
- 因为 SceneCapture 跟随玩家后，玩家在 `RT_Capture` 中会一直接近中心，所以后续还需要做纹理平移补偿，让历史路径保持在世界空间的正确位置。


## 世界坐标转 RT 采样 UV  
  
为了解决路径和世界位置对齐，需要在材质中用世界坐标计算采样 UV，而不是直接使用 `TexCoord`
  
核心公式可以理解为：  
  
`UV = (WorldPosition.xy - CaptureLocation.xy) / CaptureSize + 0.5`  
  
或等价地：  
  
`UV = (WorldPosition.xy + CaptureSize / 2 - CaptureLocation.xy) / CaptureSize`  
  
### 节点逻辑  
  
1. 使用 `Absolute World Position` 获取当前水面像素的世界坐标。  
2. 取 `XY`，只保留水面平面方向。  
3. 使用 `Capture Location` 参数表示 `SceneCaptureComponent2D` 当前世界位置。  
4. 使用 `1024` 表示捕获范围。  
5. 使用 `1024 / 2 = 512` 作为半个捕获范围。  
6. 将当前世界位置换算成相对于捕获区域的位置。  
7. 再除以 `1024`，归一化成 `0~1` UV。  
8. 用计算出的 UV 去采样 `HeightField / RT_Height_0`。  
  
### 注释  
  
- `TexCoord` 是模型自身 UV，和玩家世界位置无关。  
- `Absolute World Position` 是场景真实坐标，可以让纹理采样绑定到世界空间。  
- `Capture Location` 表示移动捕获窗口的中心。  
- `1024` 表示这张 RT 对应世界空间中的覆盖范围。  
- `512` 用来把以中心为原点的坐标平移到 `0~1` UV 范围。  
- 这一步的目的，是让水面上每个世界位置能采样到 `RT_Height_0` 中对应的位置。  
- 可以理解为：`SceneCapture` 是一张会移动的小地图，材质需要计算当前水面像素在这张小地图里的位置。

# 8.纹理绘制处理
## WaveCompute 的位置参数传递

![](images/Pasted image 20260506171923.png)![](images/Pasted image 20260506172105.png)
### 操作

1. 创建 `MID_WaveCompute` 动态材质实例。
2. 将返回值保存为 `MID Wave Compute` 变量。
3. 使用 `DefaultSceneRoot → Get World Location` 获取水面 Actor 的世界位置。
4. 使用 `Set Vector Parameter Value`。
5. `Target` 接入 `MID_WaveCompute`。
6. `Parameter Name` 设置为 `Plane Location`。
7. `Value` 接入 `DefaultSceneRoot` 的世界位置。
8. 在每帧绘制前，使用 `Get Player Pawn → Get Actor Location` 获取玩家位置。
9. 使用 `Set Vector Parameter Value`。
10. `Target` 接入 `MID_WaveCompute`。
11. `Parameter Name` 设置为 `Capture Location`。
12. `Value` 接入玩家世界位置。

### 注释

- `Plane Location` 表示 `RT_Height_0` 这张历史画布在世界中的基准位置。
- `Capture Location` 表示当前帧 `RT_Capture` 对应的世界中心位置。
- 因为 `SceneCaptureComponent2D` 跟随玩家移动，所以玩家位置基本等于当前捕获中心。
- `RT_Capture` 是一张跟随玩家移动的局部输入图。
- `RT_Height_0` 是固定在水面范围上的历史路径画布。
- `WaveCompute` 需要同时知道 `Capture Location` 和 `Plane Location`，才能把当前帧 Capture 画到 `RT_Height_0` 的正确位置。
- 简单理解：`Capture Location` 决定“这一帧拍的是哪里”，`Plane Location` 决定“历史画布放在哪里”。
- 这一步解决的是 `Capture 空间 → Height0 画布空间` 的位置对齐问题。

## 蓝图变量公开与动态创建 RT

![](images/Pasted image 20260506173421.png =349)
![](images/Pasted image 20260506173403.png =350)

### 操作

1. 创建变量 `Texture Resolution`，用于控制高度图 RT 分辨率。
2. 勾选 `Instance Editable`，让它暴露到关卡实例的 Details 面板中。
3. 使用 `Create Render Target 2D` 动态创建 `RT_Height_0`。
4. 将 `Texture Resolution` 同时接入 `Width` 和 `Height`。
5. 将 `Format` 设置为 `RTF R16f`。
6. 将返回值保存为变量 `RT_Height_0`。
7. 使用 `Set Scalar Parameter Value` 将 `Texture Resolution` 传给 `MID_WaveCompute`。
8. 材质内部使用该参数计算像素偏移、邻域采样等逻辑。

### 注释

- `Texture Resolution` 控制运行时创建的 `RT_Height_0` 尺寸。
- 例如 `512` 表示创建 `512 × 512` 的高度图，`1024` 表示创建 `1024 × 1024` 的高度图。
- 分辨率越高，路径和波纹细节越好，但性能开销也更高。
- `RTF R16f` 是单通道 16 位浮点格式，适合保存波纹高度数据。
- 同一个 `Texture Resolution` 既要用于创建 RT，也要传给 `WaveCompute`，保证材质知道当前 RT 的真实尺寸。
- `Instance Editable` 的作用是让变量可以在关卡实例中直接调节。
- 适合公开的变量：`Texture Resolution`、`Capture Size`、水面尺寸、波纹强度、衰减参数、Debug 开关。
- 不适合公开的变量：`MID_Watersurface`、`MID_WaveCompute`、运行时生成的 RT、上一帧位置、临时计算变量。
- 简单理解：公开参数是给关卡中调效果用的，不公开参数是蓝图内部自己维护的。



## 水面缩放后的绘制对齐问题

![](images/Pasted image 20260506174521.png =473)
![](images/Pasted image 20260506174813.png =470)
![](images/Pasted image 20260506181347.png =474)
![](images/Pasted image 20260506181318.png =352)
### 操作

1. 在蓝图中获取 `Plane` 组件。
2. 使用 `Get World Scale` 读取 `Plane` 的世界缩放。
3. 取 `Scale X` 和 `Scale Y`。
4. 使用 `Max` 取二者中较大的值。
5. 将结果保存为 `Plane Scale`。
6. 使用 `Set Scalar Parameter Value`。
7. `Target` 接入 `MID_WaveCompute`。
8. `Parameter Name` 设置为 `Canvas Size`。
9. `Value` 接入 `Plane Scale`。
10. 在材质中创建 `Canvas Size` 参数。
11. 用 `Canvas Size` 替代原来写死的水面范围参数。
12. 将原来的 `400` 改为 `400 * Canvas Size`。
13. 将原来的 `800` 改为 `800 * Canvas Size`。
14. 在世界坐标转 UV 的计算中使用新的动态范围。
15. 保证 `M_WaveCompute` 和水面显示材质中涉及画布范围的地方都使用同一套 `Canvas Size` 逻辑。

### 注释

- 水面缩放后，实际水面覆盖范围会变化。
- 如果材质里仍然按固定 `800 × 800` 计算 UV，绘制路径会出现比例错误或位置偏移。
- `Get World Scale` 用来获取当前水面 Plane 的实际缩放。
- 取 `Scale X` 和 `Scale Y` 的最大值，是为了在非等比缩放时保证画布范围能覆盖完整水面。
- `Canvas Size` 表示水面画布范围的缩放倍率。
- 原来的 `800` 表示基础画布范围。
- 原来的 `400` 表示基础画布半范围。
- 加入 `Canvas Size` 后：

  `实际画布范围 = 800 * Canvas Size`

  `实际半范围 = 400 * Canvas Size`

- 在水面显示材质中，原来的采样公式可以理解为：

  `UV = (WorldPosition - ObjectPosition + 400) / 800`

- 加入 `Canvas Size` 后变为：

  `UV = (WorldPosition - ObjectPosition + 400 * Canvas Size) / (800 * Canvas Size)`

- 这样水面中心仍然对应 UV `0.5`，水面边界仍然对应 UV `0` 和 `1`。
- 在 `M_WaveCompute` 中，`Canvas Size` 用于让当前帧 `RT_Capture` 写入 `RT_Height_0` 时匹配缩放后的历史画布范围。
- 在水面显示材质中，`Canvas Size` 用于让 `RT_Height_0` 正确显示到缩放后的水面范围上。
- 这一步解决的是：水面缩放后，纹理绘制位置和路径显示比例不再错位。

# 9.模拟波纹扩散效果

## 创建 M_WaveSimulation 波纹模拟材质

### 操作

1. 创建新材质 `M_WaveSimulation`。
2. 将 `Shading Model` 设置为 `Unlit`。
3. 勾选 `Allow Negative Emissive Color`。
4. 后续将该材质作为计算材质使用，不直接作为最终水面显示材质。

### 注释

- `M_WaveSimulation` 用于真正的波纹扩散计算。
- 它和 `M_WaveCompute` 分工不同：
  - `M_WaveCompute`：把角色捕获 Mask 写入高度图。
  - `M_WaveSimulation`：读取高度图，让扰动向周围扩散并衰减。
- 设置为 `Unlit` 是为了让输出只受节点计算影响，不受灯光、阴影、PBR 参数影响。
- 勾选 `Allow Negative Emissive Color` 是为了允许材质输出负值。
- 水波高度需要正负变化：
  - 正值表示波峰。
  - 负值表示波谷。
  - 0 表示平静水面。
- 如果不允许负值，波谷会被截断，后续波纹扩散和法线计算都会不自然。




## 波纹扩散计算逻辑

![](images/Pasted image 20260507230926.png =349)

![](images/Pasted image 20260507230149.png =346)

### 操作

1. 在 `M_WaveSimulation` 中采样上一帧高度图 `Previous Height1`。
2. 分别偏移 UV，采样当前像素周围的高度：
   - 左侧像素
   - 右侧像素
   - 上侧像素
   - 下侧像素
3. 将周围四个方向的高度相加，得到邻域高度合成结果。
4. 将邻域高度结果乘以 `0.5`，控制波纹扩散强度。
5. 采样更早一帧的高度图 `Previous Height2`。
6. 用邻域高度结果减去 `Previous Height2`。
7. 将结果乘以 `0.97`，作为阻尼衰减。
8. 将最终结果输出到 `Emissive Color`，用于写入新的高度 RT。

### 注释

- 这一段开始进入真正的波纹模拟，不再只是路径绘制。
- `Previous Height1` 表示上一帧高度图，用来提供当前波纹状态。
- 周围像素采样用于让高度向四周传播。
- `Previous Height2` 表示更早一帧高度图，用于产生惯性和反弹效果。
- 公式可以简化理解为：

  `NewHeight = (NeighborHeight * 0.5 - PreviousHeight2) * 0.97`

- `NeighborHeight * 0.5` 负责扩散。
- `- PreviousHeight2` 负责产生波峰 / 波谷交替的振荡效果。
- `* 0.97` 负责衰减，让波纹逐渐消失。
- 这段逻辑的本质是：**扩散 + 振荡 + 衰减**。
- 如果没有邻域采样，波纹不会向外传播。
- 如果没有 `Previous Height2`，效果更像模糊扩散，而不是水波。
- 如果没有衰减，波纹会一直振荡，不会自然消失。

## RT 贴图与分辨率选择

RT（Render Target）可以理解为运行时动态更新的贴图。  
普通贴图通常是提前做好的图片，而 RT 可以在游戏运行中被 `SceneCapture`、`Draw Material to Render Target` 等节点写入，再被材质读取。

在交互水项目中，RT 主要承担“数据缓存”的作用。

### RT_Capture

`RT_Capture` 用来记录当前帧玩家 / 物体的位置，本质是交互输入 Mask。

它只需要表达：

- 哪里有交互
- 哪里没有交互
- 边缘有少量灰度过渡

所以它可以使用较低分辨率和较低精度格式：

- 分辨率：`128 / 256`
- 格式：`RTF R8`

`R8` 只存一个通道，适合黑白 Mask，性能开销低。

### RT_Height / Previous Height

`RT_Height_0 / 1 / 2` 用来保存波纹高度场，不是简单 Mask。

它需要记录：

- 波峰
- 波谷
- 衰减过程
- 扩散过程
- 多帧迭代后的连续高度变化

所以需要更高分辨率和浮点格式：

- 分辨率：`1024` 或更高
- 格式：`RTF R16f` 或 `RTF RGBA16f`

`R16f` 适合单通道高度图，能保存更细腻的小数和负值。  
`RGBA16f` 可以同时存多组数据，但显存和带宽开销更高。

### 核心理解

`RT_Capture` 是检测图，只负责告诉系统“哪里被碰到了”。  
`RT_Height` 是模拟图，负责保存“水面高度如何变化”。

简单总结：

- `RT_Capture`：低分辨率、`R8`、当前帧输入 Mask
- `RT_Height`：高分辨率、`R16f / RGBA16f`、波纹高度模拟数据

一句话：

> Capture 可以粗，因为它只是输入；Height 必须细，因为它承载真正的波纹模拟。

## 三张 RT 的轮换逻辑

![](images/屏幕截图 2026-05-07 234632.png)

![](images/屏幕截图 2026-05-07 235038.png)

![](images/屏幕截图 2026-05-07 235624.png)
### 操作

1. 创建三张高度 RT：
   - `RT_Height_0`
   - `RT_Height_1`
   - `RT_Height_2`

2. 创建变量 `HeightIndex`，用于记录当前要写入哪一张 RT。

3. 每次完成一次波纹模拟后，更新索引：

   `HeightIndex = (HeightIndex + 1) % 3`

4. 创建函数 `Get Render Target Texture`，根据输入索引返回对应 RT：
   - `Index = 0` → `RT_Height_0`
   - `Index = 1` → `RT_Height_1`
   - `Index = 2` → `RT_Height_2`

5. 创建函数 `Get Last Render Target Texture`，用于根据当前 `HeightIndex` 获取历史帧 RT。

6. 在 `Get Last Render Target Texture` 中使用参数 `Nums Frame Old Index` 表示要往前取几帧：
   - `Nums Frame Old Index = 1` → 取上一帧高度图
   - `Nums Frame Old Index = 2` → 取上上帧高度图

7. 使用公式计算历史 RT 的索引：

   `Index = (CurrentHeightIndex - Nums Frame Old Index + 3) % 3`

8. 将计算出的 `Index` 传入 `Get Render Target Texture`，返回对应的历史 RT。

9. 在波纹模拟时：
   - 当前 `HeightIndex` 对应的 RT 用作写入目标
   - `Nums Frame Old Index = 1` 返回的 RT 作为 `Previous Height1`
   - `Nums Frame Old Index = 2` 返回的 RT 作为 `Previous Height2`

### 注释

- 三张 `RT_Height` 不是三种不同效果，而是同一种高度图在不同时间帧的缓存。
- `HeightIndex` 表示当前帧要写入哪一张 RT。
- 每次模拟完成后，`HeightIndex` 通过 `(HeightIndex + 1) % 3` 在 `0 → 1 → 2 → 0` 之间循环。
- `Nums Frame Old Index` 表示要回溯几帧。
- `Nums Frame Old Index = 1` 表示上一帧。
- `Nums Frame Old Index = 2` 表示上上帧。
- `CurrentHeightIndex - Nums Frame Old Index` 表示从当前索引往前找历史帧。
- `+3` 用来避免结果变成负数。
- `%3` 用来保证最终索引始终落在 `0、1、2` 三张 RT 范围内。
- 例如当前 `HeightIndex = 0`：
  - 上一帧：`(0 - 1 + 3) % 3 = 2`
  - 上上帧：`(0 - 2 + 3) % 3 = 1`
- 所以当前写 `RT_Height_0` 时，上一帧是 `RT_Height_2`，上上帧是 `RT_Height_1`。
- 波纹模拟需要用旧状态计算新状态，所以不能只用一张 RT 同时读写。
- `Previous Height1` 提供上一帧的高度状态。
- `Previous Height2` 提供上上帧的高度状态，用于产生惯性和波峰 / 波谷反弹。
- 当前帧计算出的新高度会写入 `HeightIndex` 指向的 RT。
- 这套结构本质是三缓冲 / 循环缓冲，用来保存波纹模拟的时间状态。

### 核心理解

`HeightIndex` 决定当前写哪张 RT。  
`Nums Frame Old Index` 决定往前取几帧。  
`Previous Height1 / Previous Height2` 提供历史状态。  
`M_WaveSimulation` 根据历史状态计算新的波纹高度。

简单理解：

> 三张 RT 像三个循环使用的缓存格子，当前格子负责写入新结果，前两个格子负责提供上一帧和上上帧的数据。

# 10. 帧率控制与模拟次数

## 基于帧率控制波纹模拟执行次数

![](images/屏幕截图 2026-05-22 163350.png)
### 操作

1. 在每帧 Tick 中，将变量 `Frame Rate` 加上 `Delta Seconds`，结果写回 `Frame Rate`。
2. 用 `MIN` 节点将 `Simulation Speed` 和 `120.0` 取较小值，防止模拟速度过高。
3. 将 `1.0` 除以上一步的结果，得到每次模拟对应的时间间隔阈值。
4. 用 `Frame Rate > 阈值` 作为 While Loop 的 Condition。
5. While Loop 的 Loop Body 中执行一次波纹模拟（Draw WaveSimulation），并将 `Frame Rate` 减去阈值，更新 `HeightIndex`。
6. While Loop 在 `Frame Rate` 累积超过阈值时，每帧可以执行一次或多次模拟；低帧率时跳过执行，保证模拟节奏稳定。

### 注释

- `Frame Rate` 是一个累加器，每帧加上 `Delta Seconds`，相当于记录"已经过了多少秒"。
- `1.0 / MIN(Simulation Speed, 120.0)` 表示每次模拟之间需要间隔多少秒。
- `MIN(..., 120.0)` 的作用是限制最大模拟速度，防止极端参数导致每帧执行次数失控。
- While Loop 保证：帧率高时，`Frame Rate` 累积快，可能每帧执行多次；帧率低时，`Frame Rate` 累积慢，可能多帧才执行一次。
- 这样波纹模拟的实际节奏由 `Simulation Speed` 决定，而不是直接绑定到帧率。
- 核心理解：**不是每帧固定执行一次，而是根据时间累积决定执行几次，让波纹速度与帧率解耦。**

# 11. 高度图转法线：有限差分法

![](images/屏幕截图 2026-05-23 165529.png)

![](images/屏幕截图 2026-05-23 161534.png)
## 作用

从 Height RT 生成法线，驱动水面光照凹凸。

## 节点链路

Height RT → 四方向偏移采样 → 高度差 → 切向量 → Cross → Normalize → Normal

---

## 1. 像素步长

```
Texel Size = 1 / 分辨率    （如 256 → 0.003906）
```

UV 空间中一个像素的距离，用来偏移 UV 采样相邻像素。

## 2. 四方向采样

```
H_R = Height(UV + ( Texel Size,  0))
H_U = Height(UV + ( 0,  Texel Size))
H_L = Height(UV + (-Texel Size,  0))
H_D = Height(UV + ( 0, -Texel Size))
```

## 3. 高度差

```
Append(H_R, H_U) - Append(H_L, H_D) = (Height_U, Height_V)

Height_U = H_R - H_L   → 左右方向斜率
Height_V = H_U - H_D   → 上下方向斜率
```

## 4. Height Scale

高度差乘以 `Height Scale`（默认 0.005），控制凹凸强度。值越大法线越倾斜。

## 5. Mask 拆分

Subtract 输出的是 `(R=Height_U, G=Height_V)`：

```
Mask R → Height_U   → 送入 Tangent_X 的 Z 分量
Mask G → Height_V   → 送入 Tangent_Y 的 Z 分量
```

## 6. 构建切向量

```
Tangent_X = (Texel Size × 2,  0,                  Height_U × HeightScale)
Tangent_Y = (0,                 Texel Size × 2,   Height_V × HeightScale)
```

`Texel Size × 2` 是因为中心差分跨了两个像素（左←→右、上←→下），切向量水平分量需匹配此跨度。

## 7. 叉积 + 归一化

```
Normal = Normalize(Cross(Tangent_X, Tangent_Y))
```

两个切向量确定局部斜面，叉积得垂直方向 → 法线。

## 8. 输出

- 调试阶段接 **Emissive Color** 可视化法线方向
- 正式使用时接 **Normal** 输入
- 需几何起伏时配合高度图接 **World Position Offset**

---

## 完整公式

```
S = Texel Size = 1 / 分辨率

H_R = Height(UV + ( S,  0))
H_U = Height(UV + ( 0,  S))
H_L = Height(UV + (-S,  0))
H_D = Height(UV + ( 0, -S))

Height_U = H_R - H_L
Height_V = H_U - H_D

Tangent_X = (2S, 0, Height_U × Scale)
Tangent_Y = (0, 2S, Height_V × Scale)

Normal = Normalize(Cross(Tangent_X, Tangent_Y))
```

---

## 一句话

高度图给高低 → 四邻域给坡度 → 切向量给斜面 → 叉积给法线。

---

# 12. 法线纹理生成

## 初始化 M_WaveNormal

### 操作

1. 创建 `M_WaveNormal` 材质引用变量。
2. 使用 `Create Dynamic Material Instance` 创建动态材质实例。
3. 将返回值保存为 `MID_WaveNormal`。
4. 使用 `Set Scalar Parameter Value`。
5. `Target` 接入 `MID_WaveNormal`。
6. `Parameter Name` 设置为 `Canvas Size`。
7. `Value` 接入 `Plane Scale`。

### 注释

- `M_WaveNormal` 是高度图转法线的计算材质。
- `MID_WaveNormal` 用于运行时接收最新 Height RT、Canvas Size 等参数。
- `Canvas Size` 用来适配水面缩放，保证法线采样偏移和当前水面范围一致。
- 这一步只是初始化法线计算材质，还没有真正生成法线 RT。

---

## 将最新 Height RT 传给 M_WaveNormal

### 操作

1. 在波纹模拟循环结束后，从 `Completed` 执行后续逻辑。
2. 使用 `Get Render Target Texture`。
3. `Index` 接入当前 `HeightIndex`。
4. 获取当前最新的 `RT_Height`。
5. 使用 `Set Texture Parameter Value`。
6. `Target` 接入 `MID_WaveNormal`。
7. `Parameter Name` 设置为 `Heightfield Height`。
8. `Value` 接入当前最新的 `RT_Height`。

### 注释

- `While Loop` 的 `Loop Body` 会执行一次或多次波纹模拟。
- `Completed` 表示本帧所有波纹模拟步骤已经结束。
- 法线必须基于最终最新的高度图计算，所以要在 `Completed` 后传入 Height RT。
- 由于高度图使用三张 RT 轮换，不能固定传 `RT_Height_0`。
- 必须通过当前 `HeightIndex` 动态获取最新的 RT。
- 简单理解：波纹先算完，再把最新高度图交给 WaveNormal 算法线。

---

## 创建 RT_HeightNormal 并写入法线结果

### 操作

1. 创建新的 Render Target，命名为 `RT_HeightNormal`。
2. 使用 `Draw Material to Render Target`。
3. `Texture Render Target` 接入 `RT_HeightNormal`。
4. `Material` 接入 `MID_WaveNormal`。
5. 将 `M_WaveNormal` 的输出结果写入 `RT_HeightNormal`。

### 注释

- `RT_Height_0 / 1 / 2` 保存的是高度数据。
- `RT_HeightNormal` 保存的是由高度图推导出的法线数据。
- `M_WaveNormal` 读取最新 Height RT，经过四方向采样、有限差分、叉积和归一化后，输出动态法线。
- `Draw Material to Render Target` 会把这次法线计算结果写入 `RT_HeightNormal`。
- 这一步建立了链路：

  `最新 Height RT → M_WaveNormal → RT_HeightNormal`

---

## 在水面材质中使用 RT_HeightNormal

### 操作

1. 在水面材质中创建 `Texture Sample Parameter2D`。
2. 命名为 `Heightfield Normal`。
3. 在蓝图中将 `RT_HeightNormal` 传给 `MID_Watersurface` 的 `Heightfield Normal` 参数。
4. 在材质中读取 `Heightfield Normal`。
5. 将其接入 `FlattenNormal`。
6. 使用 `Flatten Strength` 控制动态法线强度。
7. 将 `FlattenNormal` 的结果接入水面材质的 `Normal`。

### 注释

- `HeightField` 负责提供水面高度 / 波纹区域信息。
- `Heightfield Normal` 负责提供由高度图生成的动态法线。
- 高度图决定“哪里高低变化”。
- 法线图决定“水面局部朝向如何变化”。
- `FlattenNormal` 用于压低或增强动态法线影响，避免波纹反光过强。
- `Flatten Strength` 越小，法线扰动越明显。
- `Flatten Strength` 越大，水面越接近平整。

---

## 当前完整链路

`RT_Height` 三缓冲保存波纹高度。  
`M_WaveNormal` 读取最新高度图，计算动态法线。  
`RT_HeightNormal` 保存法线结果。  
最终水面材质读取 `RT_HeightNormal`，接入 `Normal`，让水面在光照下出现波纹凹凸。

核心链路：

`RT_Height_0 / 1 / 2 → M_WaveNormal → RT_HeightNormal → M_WaterSurface.Normal`

一句话：

> 高度图负责模拟水面起伏，法线图负责把这些起伏转换成光照可见的波纹细节。


# 13.高度权重处理

## SceneDepth 转下压高度

![](images/屏幕截图 2026-05-29 203921.png)

![](images/屏幕截图 2026-05-29 202100.png =697)

![](images/屏幕截图 2026-05-29 202307.png =697)

![](images/屏幕截图 2026-05-29 203847.png)

### 操作

1. 创建 `M_SceneDepth` 材质。
2. 将材质的 `Material Domain` 设置为 `Post Process`。
3. 使用 `SceneTexture: SceneDepth` 读取场景深度。
4. 取 `R` 通道。
5. 使用 `Divide 5000` 压缩深度范围。
6. 使用 `Saturate` 将结果限制到 `0~1`。
7. 将结果接入 `Emissive Color`。
8. 将 `M_SceneDepth` 添加到 `SceneCaptureComponent2D` 的 `Post Process Materials`。
9. `SceneCaptureComponent2D` 只捕获人物模型。
10. `RT_Capture` 接收经过 `M_SceneDepth` 处理后的深度图。
11. 在 `M_WaveCompute` 中采样 `RT_Capture`。
12. 使用公式将深度图转换为高度输入：

   `HeightInput = Saturate(SceneDepth / 5000) * 0.5 - 0.5`

13. 将 `HeightInput` 写入 Height RT。
14. 最终水面材质读取 `HeightField`，乘以 `Simulation Height` 后接入 `World Position Offset`。

### 注释

- `M_SceneDepth` 挂到 `SceneCaptureComponent2D` 后，不只是调试显示，而是会改变 `RT_Capture` 的内容。
- 此时 `RT_Capture` 存的不是普通颜色 Mask，而是深度图。
- 因为 SceneCapture 只捕获人物：
  - 人物区域：有深度，距离相机近，`Depth01 < 1`
  - 背景 / 水面：没有被捕获，深度趋近无限远，`Saturate` 后约等于 `1`
- 深度归一化公式：

  `Depth01 = Saturate(SceneDepth / 5000)`

- 高度输入公式：

  `HeightInput = Depth01 * 0.5 - 0.5`

- 计算结果：
  - 背景：`1 * 0.5 - 0.5 = 0`
  - 人物：`小于 1 * 0.5 - 0.5 = 负值`

- 所以人物区域会被转换成负高度。
- 负高度写入 Height RT 后，水面在人物附近向下压。
- 后续 `M_WaveSimulation` 会让这个下压扰动扩散、反弹、衰减。
- 最终 WPO 公式：

  `ZOffset = HeightField * SimulationHeight`

- 如果 `HeightField < 0`，则 `ZOffset < 0`，水面向下位移。

### 核心理解

`SceneDepth` 提供“人物近、背景远”的深度差。  
`* 0.5 - 0.5` 把深度值从 `0~1` 映射到 `-0.5~0`。  
因此人物区域变成负高度，模拟角色把水面压下去。

一句话：

> `M_SceneDepth` 负责把人物深度写进 `RT_Capture`，`M_WaveCompute` 把深度转成负高度输入，`WPO` 再把负高度表现为水面下压。


# 14.实现不同物体交互

![](images/屏幕截图 2026-05-29 231646.png)
![](images/屏幕截图 2026-05-29 232750.png)
## 初始化捕获对象

### 操作

1. 在蓝图中添加 `SimulationCaptureVolume`。
2. 使用 `Get Overlapping Actors` 获取当前已经在范围内的 Actor。
3. 将结果保存为 `Overlapping Actors`。
4. 使用 `For Each Loop` 遍历数组。
5. 将 `Array Element` 接入 `Add Unique`，加入 `Capture Actor`。
6. 使用 `Show Only Actor Components`。
7. `Target` 接入 `SceneCaptureComponent2D`。
8. `In Actor` 接入 `Array Element`。

### 注释

- 这一步用于处理一开始就在水面范围内的 Actor。
- `Capture Actor` 是当前需要被捕获的对象数组。
- `Show Only Actor Components` 会把 Actor 加入 SceneCapture 的捕获名单。
- 只有被 SceneCapture 捕获到的物体，才会写入 `RT_Capture` 并参与水面交互。

## BeginOverlap 加入捕获名单

### 操作

1. 使用 `On Component Begin Overlap`。
2. 将 `Other Actor` 接入 `Add Unique`。
3. `Add Unique` 的数组接入 `Capture Actor`。
4. 使用 `Show Only Actor Components`。
5. `Target` 接入 `SceneCaptureComponent2D`。
6. `In Actor` 接入 `Other Actor`。

### 注释

- `Other Actor` 是刚进入 `SimulationCaptureVolume` 的物体。
- `Add Unique` 用于避免重复加入同一个 Actor。
- 进入范围后，该 Actor 会被 SceneCapture 捕获。
- 捕获结果进入 `RT_Capture`，再进入后续 `WaveCompute → Height RT → WPO` 链路。

## EndOverlap 移除逻辑

### 操作

1. 使用 `On Component End Overlap`。
2. 将 `Other Actor` 接入 `Contains`。
3. `Contains` 的数组接入 `Capture Actor`。
4. 使用 `Branch` 判断结果。
5. `True` 时执行 `Remove`。
6. 再使用 `Remove Show Only Actor Components`。
7. `Target` 接入 `SceneCaptureComponent2D`。
8. `In Actor` 接入 `Other Actor`。

### 注释

- `Contains` 用于判断离开的 Actor 是否在捕获数组中。
- `Remove` 用于从 `Capture Actor` 中删除该 Actor。
- `Remove Show Only Actor Components` 用于从 SceneCapture 捕获名单中移除。
- 离开范围后，该 Actor 不再写入 `RT_Capture`。

## 当前交互链路

```text
Actor 进入 Volume
→ Add Unique 到 Capture Actor
→ Show Only Actor Components
→ SceneCapture 捕获 Actor
→ 写入 RT_Capture
→ M_WaveCompute 转成 HeightInput
→ 写入 Height RT
→ 水面材质读取 HeightField
→ WPO / Normal 产生交互
````

## 核心理解

不同物体交互的本质不是给每个物体单独写水面逻辑，而是：

```text
谁进入交互范围
→ 谁被加入 SceneCapture 捕获名单
→ 谁就能影响 RT_Capture
→ 谁就能参与水面模拟
```

# 15.实现最终水交互

## 接入最终水面材质

### 操作

1. 将蓝图变量 `M_Watersurface` 的默认值改为最终水面材质。
2. 保持原来的动态材质创建逻辑不变。
3. 运行时由 `M_Watersurface` 创建 `MID_Watersurface`。
4. 将 `MID_Watersurface` 赋给水面 `Plane`。

### 注释

- 这一步只是把测试材质换成最终水面材质。
- 后续高度图、法线图、参数都会传给这个动态材质实例。

## 复制 Test 材质中的 WPO 节点

### 操作

1. 从测试材质中复制 `HeightField → WPO` 相关节点。
2. 粘贴到最终水面材质中。
3. `HeightField.R` 乘以 `Simulation Height`。
4. 使用 `Append` 组成 `(0, 0, Height)`。
5. 与原本基础水波 WPO 相加。
6. 接入 `World Position Offset`。

### 注释

- `HeightField` 是交互波纹高度图。
- `Simulation Height` 控制交互位移强度。
- `Append(0,0,Height)` 表示只做 Z 方向上下位移。
- 不要对 `HeightField` 做 `Saturate`，否则负高度会被压掉。

## 复制 Test 材质中的 Normal 节点

### 操作

1. 从测试材质中复制 `Heightfield Normal` 相关节点。
2. 粘贴到最终水面材质中。
3. 将 `Heightfield Normal` 接入 `FlattenNormal`。
4. 将交互法线接入 `BlendAngleCorrectedNormals`。
5. 将基础水波法线和交互法线混合。
6. 输出接入主材质 `Normal`。

### 注释

- `Heightfield Normal` 是交互波纹法线图。
- 它负责让交互波纹在反光中可见。
- `FlattenNormal` 控制交互法线强度。
- `BlendAngleCorrectedNormals` 用来正确混合基础水波法线和交互法线。

## 核心

```text
测试材质里的 HeightField WPO 节点
→ 复制到最终水面材质

测试材质里的 Heightfield Normal 节点
→ 复制到最终水面材质
````

一句话：  
把测试材质中已经验证过的交互高度和交互法线节点，接到最终水面材质的 `WPO` 和 `Normal` 上。