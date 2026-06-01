
## 1. 总体链路

```text
Player / Actor
→ SceneCapture2D 捕获交互对象
→ RT_Capture
→ M_WaveCompute 转成高度输入
→ RT_Height_0 / 1 / 2 三缓冲
→ M_WaveSimulation 做波纹扩散
→ 当前 Height RT
→ M_WaveNormal 由高度生成法线
→ RT_HeightNormal
→ MID_WaterSurface
→ WaterSurface Material 的 WPO / Normal
```

C++ 版本的核心不是换一种写法，而是把蓝图里的节点链路拆成：

- 资源：RT、Material、MID、组件。
- 初始化：`Constructor`、`OnConstruction`、`BeginPlay`。
- 每帧更新：`Tick`、`WaveCapture`、`WaveSimulation`、`WaveNormal`。
- 事件：Overlap 进入 / 离开捕获范围。

## 2. 类结构总览

| 类 | 负责什么 | 蓝图对应 |
|---|---|---|
| `AWaterSim` | 水面交互主 Actor，管理水面 Plane、SceneCapture、RT、MID、波纹模拟 Tick | `BP_WaterSimulation` |
| `ATestCharacter` | 测试角色湿身 / 绘制捕获逻辑，后续被角色类替代 | 测试角色蓝图 |
| `AWaterSimulationCharacter` | 第三人称角色 + 湿身 RenderTarget 绘制逻辑 | 第三人称角色蓝图 |

主要资源：

| 名称 | 类型 | 作用 |
|---|---|---|
| `RT_Capture` | `UTextureRenderTarget2D*` | SceneCapture 写入的捕获纹理 |
| `RT_Height_0/1/2` | `UTextureRenderTarget2D*` | 三张高度 RT，保存当前 / 历史波形 |
| `RT_HeightNormal` | `UTextureRenderTarget2D*` | 高度图转出的法线 RT |
| `M_WaveCompute` | `UMaterialInterface*` | 把捕获结果转为高度输入 |
| `M_WaveSimulation` | `UMaterialInterface*` | 根据历史高度模拟波纹扩散 |
| `M_WaveNormal` | `UMaterialInterface*` | 由 Height RT 生成 Normal RT |
| `MID_*` | `UMaterialInstanceDynamic*` | 运行时可改参数的材质实例 |

## 3. 每节课代码整理

### 第 2-5 节：材质前置阶段

#### 解决的问题
它们主要是水面材质、Flipbook、Single Layer Water 等蓝图 / 材质节点内容。

---

### 第 6，7 节 / 第 8 节：创建水面 Actor、RT 捕获与 WaveCompute

#### 解决的问题

搭建 C++ 版水交互 Actor：创建水面 Plane、SceneCapture2D、RT 引用、动态材质实例，并在 Tick 中把 `RT_Capture` 通过 `M_WaveCompute` 写入 `RT_Height_0`。

第 8 课的 `WaterSim` 源码与第 6，7 课一致，没有新增 C++ 逻辑。

#### 新增变量

| 变量 | 类型 | 作用 | 蓝图对应 |
|---|---|---|---|
| `DefaultSceneRoot` | `USceneComponent*` | Actor 根组件 | `DefaultSceneRoot` |
| `Plane` | `UStaticMeshComponent*` | 水面网格，承载水面材质 | 蓝图里的 `Plane` 组件 |
| `SceneCaptureComponent2D` | `USceneCaptureComponent2D*` | 从上方正交捕获交互对象 | `SceneCaptureComponent2D` |
| `M_WaterSurface` | `UMaterialInterface*` | 水面材质模板 | `M_Watersurface` |
| `M_WaveCompute` | `UMaterialInterface*` | 捕获图转高度图的计算材质 | `M_WaveCompute` |
| `RT_Capture` | `UTextureRenderTarget2D*` | SceneCapture 输出 | `RT_Capture` |
| `RT_Height_0` | `UTextureRenderTarget2D*` | 波纹高度输出 | `RT_Height_0` |
| `MID_WaterSurface` | `UMaterialInstanceDynamic*` | 水面动态材质实例 | `MID_Watersurface` |
| `MID_WaveCompute` | `UMaterialInstanceDynamic*` | WaveCompute 动态材质实例 | `MID_WaveCompute` |

#### 新增函数

| 函数 | 执行时机 | 作用 | 蓝图对应 |
|---|---|---|---|
| `AWaterSim()` | 构造阶段 | 创建组件并设置默认参数 | 蓝图组件面板 |
| `OnConstructionScript()` | 构造后 / 编辑器变更 | 创建水面 MID，设置 SceneCapture 角度 | Construction Script |
| `BeginPlay()` | 游戏开始 | 清空 RT，绑定 SceneCapture 输出，创建 WaveCompute MID | BeginPlay |
| `Tick()` | 每帧 | 把捕获结果写入 Height RT，再传给水面材质 | Event Tick |
| `DrawToRenderTargetSetTextureToMaterial()` | Tick 内调用 | Draw Material 到 RT，并把 RT 传给水面材质 | Draw Material to Render Target + Set Texture Parameter |

#### 核心代码与解释

```cpp
DefaultSceneRoot = CreateDefaultSubobject<USceneComponent>(TEXT("DefaultSceneRoot"));
RootComponent = DefaultSceneRoot;

SceneCaptureComponent2D = CreateDefaultSubobject<USceneCaptureComponent2D>(TEXT("SceneCaptureComponent2D"));
SceneCaptureComponent2D->SetupAttachment(DefaultSceneRoot);
SceneCaptureComponent2D->ProjectionType = ECameraProjectionMode::Orthographic;
SceneCaptureComponent2D->PrimitiveRenderMode = ESceneCapturePrimitiveRenderMode::PRM_UseShowOnlyList;
SceneCaptureComponent2D->CaptureSource = ESceneCaptureSource::SCS_FinalColorLDR;
```

解释：

- 在 Constructor 中创建默认组件。
- `CreateDefaultSubobject` 对应蓝图里添加组件。
- `Orthographic` 对应蓝图里 SceneCapture 正交投影。
- `PRM_UseShowOnlyList` 表示只捕获 ShowOnly 列表里的对象。
- 蓝图对应逻辑：添加 `SceneCaptureComponent2D`，设置 Orthographic、ShowOnly、Final Color LDR。

```cpp
MID_WaterSurface = Plane->CreateDynamicMaterialInstance(0, M_WaterSurface);
Plane->SetMaterial(0, MID_WaterSurface);
```

解释：

- 基于 `M_WaterSurface` 创建当前 Actor 专属的动态材质实例。
- 后续所有 `Heightfield`、`Heightfield Normal` 都传给这个 MID。
- 蓝图对应逻辑：`Create Dynamic Material Instance` → `Set Material`。

```cpp
SceneCaptureComponent2D->TextureTarget = RT_Capture;
SceneCaptureComponent2D->ShowOnlyActorComponents(UGameplayStatics::GetPlayerPawn(this, 0));
```

解释：

- 把 SceneCapture 输出指定到 `RT_Capture`。
- 初期只捕获玩家 Pawn。
- 蓝图对应逻辑：设置 `Texture Target`，调用 `Show Only Actor Components`。

```cpp
MID_WaveCompute->SetTextureParameterValue("Capture", RT_Capture);
DrawToRenderTargetSetTextureToMaterial(RT_Height_0, MID_WaveCompute, MID_WaterSurface);
```

解释：

- 把 `RT_Capture` 传入 `M_WaveCompute` 的 `Capture` 参数。
- 用 WaveCompute 材质绘制到 `RT_Height_0`。
- 绘制完成后把 `RT_Height_0` 设置给水面材质的 `Heightfield`。
- 蓝图对应逻辑：`Set Texture Parameter Value(Capture)` → `Draw Material to Render Target` → `Set Texture Parameter Value(Heightfield)`。

#### 本节数据流

```text
Player
→ SceneCapture2D
→ RT_Capture
→ MID_WaveCompute.Capture
→ RT_Height_0
→ MID_WaterSurface.Heightfield
```

#### 注意点

- `Heightfield`、`Capture` 参数名必须和材质参数完全一致。
- `SceneCaptureComponent2D` 必须有 `TextureTarget`。
- `MID_WaterSurface` 为空时，RT 即使写入也不会显示到水面。

---

### 第 9，10 节 / 第 11 节：运行时创建 RT、加入尺寸参数

#### 解决的问题

把 `RT_Height_0` 改为运行时创建，并把水面尺寸、纹理分辨率、Actor 位置传给 WaveCompute，避免材质里写死参数。

第 11 课源码与第 9，10 课一致，没有新增 C++ 逻辑。

#### 新增变量

| 变量 | 类型 | 作用 | 蓝图对应 |
|---|---|---|---|
| `TextureResolution` | `int32` | Height RT 分辨率，默认 `1024` | `Texture Resolution` 变量 |

#### 新增函数

无新增函数，主要扩展 `BeginPlay()` 和 `Tick()`。

#### 核心代码与解释

```cpp
RT_Height_0 = UKismetRenderingLibrary::CreateRenderTarget2D(
    this, TextureResolution, TextureResolution, RTF_R16f);
```

解释：

- 运行时创建高度 RT。
- `RTF_R16f` 用单通道半精度浮点保存高度，能保留负值和较细高度变化。
- 蓝图对应逻辑：`Create Render Target 2D`。

```cpp
FVector PlaneScale = Plane->GetComponentScale();
float MaxPlaneScale = FMath::Max(PlaneScale.X, PlaneScale.Y);

MID_WaterSurface->SetScalarParameterValue("Canvas Size", MaxPlaneScale);
MID_WaveCompute->SetScalarParameterValue("Texture Resolution", TextureResolution);
MID_WaveCompute->SetScalarParameterValue("Canvas Size", MaxPlaneScale);
MID_WaveCompute->SetVectorParameterValue("Plane Location", DefaultSceneRoot->GetComponentLocation());
```

解释：

- `Canvas Size` 表示水面覆盖范围。
- `Texture Resolution` 让材质知道 RT 分辨率。
- `Plane Location` 用于把世界坐标换算到水面局部 UV。
- 蓝图对应逻辑：BeginPlay 中给 MID 设置 `Canvas Size`、`Texture Resolution`、`Plane Location`。

```cpp
FVector PlayerLoc = UGameplayStatics::GetPlayerPawn(this, 0)->GetActorLocation();
SceneCaptureComponent2D->SetWorldLocation(
    FVector(PlayerLoc.X, PlayerLoc.Y, SceneCaptureComponent2D->GetComponentLocation().Z));
MID_WaveCompute->SetVectorParameterValue("Capture Location", PlayerLoc);
```

解释：

- SceneCapture 跟随玩家 XY 移动，Z 保持固定。
- `Capture Location` 传给材质，用于把捕获区域对齐到玩家位置。
- 蓝图对应逻辑：Tick 中更新 SceneCapture Location，并设置 `Capture Location` 参数。

#### 本节数据流

```text
Plane Scale / Actor Location / Player Location
→ MID_WaveCompute 参数
→ WaveCompute 正确换算捕获坐标
→ RT_Height_0
```

#### 注意点

- `TextureResolution` 和实际 RT 创建尺寸要一致。
- `Canvas Size` 同时传给水面材质和计算材质。
- SceneCapture 只跟随 XY，Z 不应随角色高度乱变。

---

### 第 12，13 节：三缓冲与 WaveSimulation

#### 解决的问题

引入三张 Height RT，实现波纹的时间状态保存。`M_WaveSimulation` 读取前两帧高度，计算当前帧高度，实现扩散、回弹、衰减。

#### 新增变量

| 变量 | 类型 | 作用 | 蓝图对应 |
|---|---|---|---|
| `M_WaveSimulation` | `UMaterialInterface*` | 波纹模拟材质 | `M_WaveSimulation` |
| `RT_Height_1` | `UTextureRenderTarget2D*` | Height 缓冲 1 | `RT_Height_1` |
| `RT_Height_2` | `UTextureRenderTarget2D*` | Height 缓冲 2 | `RT_Height_2` |
| `MID_WaveSimulation` | `UMaterialInstanceDynamic*` | 波纹模拟 MID | `MID_WaveSimulation` |
| `HeightIndex` | `int32` | 当前写入哪张 Height RT | `HeightIndex` |
| `CaptureTextureResolution` | `int32` | 捕获 RT 分辨率，默认 `128` | `Capture Texture Resolution` |

#### 新增函数

| 函数 | 执行时机 | 作用 | 蓝图对应 |
|---|---|---|---|
| `GetRenderTargetTexture()` | 需要按索引取 RT 时 | `0/1/2` 映射到三张 Height RT | Get Render Target Texture |
| `GetLastRenderTargetTexture()` | WaveSimulation 前 | 根据当前索引取前 1 / 前 2 帧 | 三缓冲历史索引 |

#### 核心代码与解释

```cpp
RT_Capture = UKismetRenderingLibrary::CreateRenderTarget2D(
    this, CaptureTextureResolution, CaptureTextureResolution, RTF_R8);
RT_Height_0 = UKismetRenderingLibrary::CreateRenderTarget2D(this, TextureResolution, TextureResolution, RTF_R16f);
RT_Height_1 = UKismetRenderingLibrary::CreateRenderTarget2D(this, TextureResolution, TextureResolution, RTF_R16f);
RT_Height_2 = UKismetRenderingLibrary::CreateRenderTarget2D(this, TextureResolution, TextureResolution, RTF_R16f);
```

解释：

- `RT_Capture` 只保存捕获 Mask / 深度输入，用 `R8` 足够。
- Height RT 保存模拟高度，使用 `R16f`。
- 蓝图对应逻辑：创建 `RT_Capture`、`RT_Height_0/1/2`。

```cpp
UTextureRenderTarget2D* AWaterSim::GetRenderTargetTexture(int32 Index)
{
    return Index == 0 ? RT_Height_0 : (Index == 1 ? RT_Height_1 : RT_Height_2);
}

UTextureRenderTarget2D* AWaterSim::GetLastRenderTargetTexture(int32 CurrentHeightIndex, int32 NumFramesOldIndex)
{
    return GetRenderTargetTexture((CurrentHeightIndex - NumFramesOldIndex + 3) % 3);
}
```

解释：

- `GetRenderTargetTexture` 把索引转成 RT。
- `GetLastRenderTargetTexture` 用循环索引取历史帧。
- `+3` 避免负数，`%3` 保证索引在 `0/1/2`。
- 蓝图对应逻辑：`(CurrentHeightIndex - NumsFrameOldIndex + 3) % 3`。

```cpp
UKismetRenderingLibrary::DrawMaterialToRenderTarget(
    this, GetRenderTargetTexture(HeightIndex), MID_WaveCompute);

HeightIndex = (HeightIndex + 1) % 3;

MID_WaveSimulation->SetTextureParameterValue(
    "Previous Height 1", GetLastRenderTargetTexture(HeightIndex, 1));
MID_WaveSimulation->SetTextureParameterValue(
    "Previous Height 2", GetLastRenderTargetTexture(HeightIndex, 2));
```

解释：

- 先把捕获输入写到当前 Height RT。
- 再推进 `HeightIndex`，准备写下一张 RT。
- WaveSimulation 读取前两张历史 RT。
- 蓝图对应逻辑：三缓冲轮换 + 设置 `Previous Height 1/2`。

#### 本节数据流

```text
RT_Capture
→ M_WaveCompute
→ 当前 Height RT
→ HeightIndex 前进
→ M_WaveSimulation 读取 Previous Height 1 / 2
→ 新 Height RT
→ MID_WaterSurface.Heightfield
```

#### 注意点

- 三缓冲索引顺序必须正确。
- `Previous Height 1` 和 `Previous Height 2` 不能传反。
- 同一张 RT 不能在同一步里既读又写。

---

### 第 14 节 / 第 15 节：固定模拟步长

#### 解决的问题

把“每帧模拟一次”改成“按 SimulationSpeed 固定时间间隔模拟”。这样高帧率和低帧率下波纹速度更稳定。

第 15 课源码与第 14 课一致，没有新增 C++ 逻辑。

#### 新增变量

| 变量 | 类型 | 作用 | 蓝图对应 |
|---|---|---|---|
| `SimulationSpeed` | `float` | 每秒模拟次数，默认 `60` | `Simulation Speed` |
| `FrameRate` | `float` | DeltaTime 累加器 | `Frame Rate` 累加变量 |

#### 新增函数

| 函数 | 执行时机 | 作用 | 蓝图对应 |
|---|---|---|---|
| `SimulationSpeedRate()` | Tick 中 while 判断 | 返回每次模拟间隔 | `1 / Min(SimulationSpeed, 120)` |
| `WaveCapture()` | Tick 每帧先执行 | 捕获输入并写入当前 Height RT | Draw WaveCompute |
| `WaveSimulation()` | while 循环中执行 | 根据固定步长推进模拟 | Draw WaveSimulation |

#### 核心代码与解释

```cpp
float AWaterSim::SimulationSpeedRate()
{
    return 1.f / FMath::Min(SimulationSpeed, 120.f);
}
```

解释：

- `SimulationSpeed` 表示每秒最多执行多少次模拟。
- `120` 是上限，避免参数过大导致一帧内循环过多。
- 蓝图对应逻辑：`1.0 / MIN(Simulation Speed, 120.0)`。

```cpp
if (WaveCapture())
{
    FrameRate += DeltaTime;

    while (FrameRate > SimulationSpeedRate())
    {
        FrameRate -= SimulationSpeedRate();
        HeightIndex = (HeightIndex + 1) % 3;
        WaveSimulation();
    }
}
```

解释：

- 每帧先捕获一次输入。
- `FrameRate` 累加真实帧时间。
- 当累积时间超过一次模拟间隔，就执行一次 `WaveSimulation`。
- 蓝图对应逻辑：Tick 中 `Frame Rate += Delta Seconds`，`While Loop` 推进模拟。

#### 本节数据流

```text
DeltaTime
→ FrameRate 累加
→ 超过 SimulationSpeedRate
→ HeightIndex 前进
→ WaveSimulation
```

#### 注意点

- `WaveCapture()` 每帧执行，`WaveSimulation()` 按固定步长执行。
- 如果 `SimulationSpeed` 太高，while 可能单帧执行多次。
- `FrameRate` 是时间累加器，不是实际 FPS。

---

### 第 16，17 节 / 第 18 节：WaveNormal 与 RT_HeightNormal

#### 解决的问题

把最新 Height RT 转成法线 RT，并传给最终水面材质，使波纹不仅有 WPO 高度，也能影响反光和法线细节。

第 18 课源码与第 16，17 课一致，没有新增 C++ 逻辑。

#### 新增变量

| 变量 | 类型 | 作用 | 蓝图对应 |
|---|---|---|---|
| `M_WaveNormal` | `UMaterialInterface*` | 高度转法线计算材质 | `M_WaveNormal` |
| `RT_HeightNormal` | `UTextureRenderTarget2D*` | 保存法线结果 | `RT_HeightNormal` |
| `MID_WaveNormal` | `UMaterialInstanceDynamic*` | WaveNormal 动态材质实例 | `MID_WaveNormal` |

#### 新增函数

| 函数 | 执行时机 | 作用 | 蓝图对应 |
|---|---|---|---|
| `WaveNormal()` | 本帧 WaveSimulation 结束后 | 用最新 Height RT 生成法线 RT，并传给水面材质 | Draw WaveNormal |

#### 核心代码与解释

```cpp
RT_HeightNormal = UKismetRenderingLibrary::CreateRenderTarget2D(
    this, TextureResolution, TextureResolution, RTF_RGBA16f);
```

解释：

- 法线需要多个通道，所以使用 `RGBA16f`。
- 与 Height RT 分辨率一致。
- 蓝图对应逻辑：创建 `RT_HeightNormal`。

```cpp
MID_WaveNormal = UKismetMaterialLibrary::CreateDynamicMaterialInstance(this, M_WaveNormal);
MID_WaveNormal->SetScalarParameterValue("Canvas Size", MaxPlaneScale);
```

解释：

- 创建高度转法线材质的 MID。
- `Canvas Size` 用于材质内部计算采样步长 / 水面范围。
- 蓝图对应逻辑：`Create Dynamic Material Instance`，设置 `Canvas Size`。

```cpp
void AWaterSim::WaveNormal()
{
    MID_WaveNormal->SetTextureParameterValue("Heightfield Height", GetRenderTargetTexture(HeightIndex));
    UKismetRenderingLibrary::DrawMaterialToRenderTarget(this, RT_HeightNormal, MID_WaveNormal);
    MID_WaterSurface->SetTextureParameterValue("Heightfield Normal", RT_HeightNormal);
}
```

解释：

- 读取当前最新 Height RT。
- 使用 `M_WaveNormal` 计算法线并写入 `RT_HeightNormal`。
- 把法线 RT 传给水面材质。
- 蓝图对应逻辑：`Set Texture Parameter Value(Heightfield Height)` → `Draw Material to Render Target(RT_HeightNormal)` → `Set Texture Parameter Value(Heightfield Normal)`。

#### 本节数据流

```text
当前 Height RT
→ MID_WaveNormal.Heightfield Height
→ RT_HeightNormal
→ MID_WaterSurface.Heightfield Normal
```

#### 注意点

- `WaveNormal()` 必须在本帧模拟完成后执行。
- `Heightfield Height` 和 `Heightfield Normal` 是两个不同材质参数。
- `RT_HeightNormal` 格式需要能存法线，不应使用 `R8`。

---

### 第 19，20 节 / 第 21-26 节 WaterSim：捕获体积、Overlap 与速度过滤

#### 解决的问题

不再只捕获玩家，而是用 `SimulationCaptureVolume` 管理进入水面交互范围的 Actor。进入范围加入 SceneCapture ShowOnly，离开范围移除。Tick 中根据速度过滤静止对象，减少无效波纹输入。

第 21-26 课的 `WaterSim` 源码与第 19，20 课一致，水面主链路没有新增 C++ 逻辑。

#### 新增变量

| 变量 | 类型 | 作用 | 蓝图对应 |
|---|---|---|---|
| `SimulationCaptureVolume` | `UBoxComponent*` | 捕获交互对象的范围盒 | `SimulationCaptureVolume` |
| `CaptureActor` | `TArray<AActor*>` | 当前在交互范围内的对象列表 | `Capture Actor` 数组 |
| `SimulationCaptureHandle` | `FTimerHandle` | 延迟初始化初始重叠对象 | Timer Handle |

#### 新增函数

| 函数 | 执行时机 | 作用 | 蓝图对应 |
|---|---|---|---|
| `InitializeSimulation()` | BeginPlay | 整理 RT / MID 初始化逻辑 | BeginPlay 初始化函数 |
| `InitializeOverlap()` | BeginPlay | 延迟调用一次初始重叠检测 | Set Timer |
| `SimulationCaptureFunction()` | BeginPlay 后 0.1 秒 | 获取已经重叠的 Actor 并加入捕获列表 | Get Overlapping Actors |
| `OnSimulationCaptureBeginOverlap()` | Actor 进入 Volume | 加入数组和 ShowOnly | On Component Begin Overlap |
| `OnSimulationCaptureEndOverlap()` | Actor 离开 Volume | 移除数组和 ShowOnly | On Component End Overlap |
| `ActorVelocityCheck()` | Tick 每帧 | 速度大于阈值才捕获 | Velocity 判断 |

#### 核心代码与解释

```cpp
SimulationCaptureVolume = CreateDefaultSubobject<UBoxComponent>(TEXT("SimulationCaptureVolume"));
SimulationCaptureVolume->SetupAttachment(DefaultSceneRoot);
```

解释：

- 在水面 Actor 下创建盒体组件。
- 用盒体决定哪些 Actor 可以参与水交互。
- 蓝图对应逻辑：添加 `SimulationCaptureVolume` 组件。

```cpp
SimulationCaptureVolume->OnComponentBeginOverlap.AddDynamic(
    this, &AWaterSim::OnSimulationCaptureBeginOverlap);
SimulationCaptureVolume->OnComponentEndOverlap.AddDynamic(
    this, &AWaterSim::OnSimulationCaptureEndOverlap);
```

解释：

- 把 C++ 函数绑定到组件 Overlap 事件。
- 绑定函数必须加 `UFUNCTION()`。
- 蓝图对应逻辑：`On Component Begin Overlap` / `On Component End Overlap` 事件节点。

```cpp
CaptureActor.AddUnique(OtherActor);
SceneCaptureComponent2D->ShowOnlyActorComponents(OtherActor);
```

解释：

- `AddUnique` 避免重复加入同一个 Actor。
- ShowOnly 让 SceneCapture 开始捕获该 Actor。
- 蓝图对应逻辑：`Add Unique` → `Show Only Actor Components`。

```cpp
CaptureActor.Remove(OtherActor);
SceneCaptureComponent2D->RemoveShowOnlyActorComponents(OtherActor);
```

解释：

- Actor 离开水面范围后，从数组和 SceneCapture 捕获列表移除。
- 蓝图对应逻辑：`Remove` → `Remove Show Only Actor Components`。

```cpp
SimulationCaptureVolume->GetOverlappingActors(OverlappingActors);
for (AActor* ArrElement : OverlappingActors)
{
    CaptureActor.AddUnique(ArrElement);
    SceneCaptureComponent2D->ShowOnlyActorComponents(ArrElement);
}
```

解释：

- 用于处理 BeginPlay 前已经在范围内的 Actor。
- 延迟 0.1 秒执行，避免组件重叠状态还没初始化完成。
- 蓝图对应逻辑：`Get Overlapping Actors` → `For Each Loop` → `Add Unique` → `Show Only Actor Components`。

```cpp
ArrayElement->GetVelocity().Size() > 100 ?
    SceneCaptureComponent2D->ShowOnlyActorComponents(ArrayElement) :
    SceneCaptureComponent2D->RemoveShowOnlyActorComponents(ArrayElement);
```

解释：

- 速度大于 100 才捕获并产生交互。
- 静止物体仍在数组里，但暂时从 ShowOnly 中移除。
- 蓝图对应逻辑：`Get Velocity` → `Vector Length` → `> 100` → ShowOnly / RemoveShowOnly。

#### 本节数据流

```text
Actor 进入 SimulationCaptureVolume
→ CaptureActor.AddUnique
→ SceneCapture ShowOnly
→ RT_Capture
→ WaveCompute
→ Height RT
```

#### 注意点

- Overlap 回调需要 `UFUNCTION()`，否则 `AddDynamic` 绑定可能失败。
- `CaptureActor` 是逻辑列表，ShowOnly 是实际捕获列表。
- 速度过滤只改 ShowOnly，不删除数组。
- 如果静止 Actor 也需要持续影响水面，需要调整速度阈值逻辑。

---

### 第 22，23 节：测试角色湿身材质入口

#### 解决的问题

新增 `ATestCharacter`，用于测试角色被点击 / 命中后切换到 `M_Unwrapper` 材质，并把命中位置、半径等参数传给材质。此阶段还没有完整 RT 写入链路。

#### 新增变量

| 变量 | 类型 | 作用 | 蓝图对应 |
|---|---|---|---|
| `M_SkeletalMesh` | `UMaterialInterface*` | 角色原始 / 目标材质引用 | 角色材质变量 |
| `M_Unwrapper` | `UMaterialInterface*` | 展开 / 绘制用材质 | Unwrapper 材质 |
| `MID_SkeletalMesh` | `UMaterialInstanceDynamic*` | 角色动态材质 | SkeletalMesh MID |

#### 新增函数

| 函数 | 执行时机 | 作用 | 蓝图对应 |
|---|---|---|---|
| `Painted()` | 被命中或外部调用 | 设置绘制位置和半径参数 | Painted 自定义事件 |

#### 核心代码与解释

```cpp
MID_SkeletalMesh = GetMesh()->CreateDynamicMaterialInstance(0, GetMesh()->GetMaterial(0));
GetMesh()->SetMaterial(0, MID_SkeletalMesh);
```

解释：

- 给角色网格创建动态材质，方便后续传湿身纹理。
- 蓝图对应逻辑：角色 BeginPlay 中创建 Dynamic Material Instance。

```cpp
GetMesh()->SetMaterial(0, M_Unwrapper);
GetMesh()->SetVectorParameterValueOnMaterials("Capture Location", GetMesh()->GetComponentLocation());
GetMesh()->SetVectorParameterValueOnMaterials("Hit Location", HitLocation);
GetMesh()->SetScalarParameterValueOnMaterials("Radius", InRadius);
```

解释：

- 临时把角色材质换成 `M_Unwrapper`。
- 传入角色位置、命中位置、绘制半径。
- 蓝图对应逻辑：设置 Unwrapper 材质参数。

#### 本节数据流

```text
HitLocation / Radius
→ M_Unwrapper 参数
→ 角色材质绘制区域
```

#### 注意点

- 本节没有完整湿身 RT 输出。
- 此处需要结合上下文确认 `M_Unwrapper` 材质内部如何使用这些参数。

---

### 第 24，25 节：测试角色湿身 RT 捕获

#### 解决的问题

给测试角色添加自己的 SceneCapture 和 `RT_HeightWet`，用 `M_Unwrapper` 捕获一次湿身区域，再把结果传给角色材质。

#### 新增变量

| 变量 | 类型 | 作用 | 蓝图对应 |
|---|---|---|---|
| `SceneCaptureComponent2D` | `USceneCaptureComponent2D*` | 角色自身湿身捕获相机 | 角色 SceneCapture |
| `RT_HeightWet` | `UTextureRenderTarget2D*` | 保存湿身高度 / Mask | `RT_HeightWet` |
| `TextureResolution` | `int32` | 湿身 RT 分辨率 | Wet Texture Resolution |

#### 新增函数

无新增主要函数，扩展 `BeginPlay()` 和 `Painted()`。

#### 核心代码与解释

```cpp
SceneCaptureComponent2D = CreateDefaultSubobject<USceneCaptureComponent2D>(TEXT("SceneCaptureComponent2D"));
SceneCaptureComponent2D->ProjectionType = ECameraProjectionMode::Orthographic;
SceneCaptureComponent2D->PrimitiveRenderMode = ESceneCapturePrimitiveRenderMode::PRM_UseShowOnlyList;
SceneCaptureComponent2D->CaptureSource = ESceneCaptureSource::SCS_SceneColorHDR;
SceneCaptureComponent2D->CompositeMode = ESceneCaptureCompositeMode::SCCM_Additive;
SceneCaptureComponent2D->bCaptureEveryFrame = false;
```

解释：

- 角色单独持有一个 SceneCapture。
- `bCaptureEveryFrame = false`，只在 Painted 时手动捕获。
- `SCCM_Additive` 用于把新绘制叠加到 RT。
- 蓝图对应逻辑：角色添加 SceneCapture，关闭每帧捕获，设置 Additive。

```cpp
RT_HeightWet = UKismetRenderingLibrary::CreateRenderTarget2D(
    this, TextureResolution, TextureResolution, RTF_RGBA16f);
SceneCaptureComponent2D->TextureTarget = RT_HeightWet;
SceneCaptureComponent2D->ShowOnlyActorComponents(this);
```

解释：

- 创建角色湿身 RT。
- SceneCapture 只捕获当前角色。
- 蓝图对应逻辑：`Create Render Target 2D`，设置 `Texture Target`，`Show Only Actor Components(self)`。

```cpp
UMaterialInterface* OLD_Material = GetMesh()->GetMaterial(0);
GetMesh()->SetMaterial(0, M_Unwrapper);
SceneCaptureComponent2D->CaptureScene();
MID_SkeletalMesh->SetTextureParameterValue("Heightfield", RT_HeightWet);
GetMesh()->SetMaterial(0, OLD_Material);
```

解释：

- 保存旧材质。
- 临时切换到展开材质并捕获。
- 捕获结果进入 `RT_HeightWet`。
- 再把 `RT_HeightWet` 传给角色动态材质。
- 蓝图对应逻辑：临时切材质 → `Capture Scene` → `Set Texture Parameter Value(Heightfield)` → 恢复材质。

#### 本节数据流

```text
Painted(HitLocation, Radius)
→ M_Unwrapper
→ SceneCapture.CaptureScene
→ RT_HeightWet
→ MID_SkeletalMesh.Heightfield
```

#### 注意点

- 需要恢复旧材质，否则角色会一直显示 Unwrapper。
- SceneCapture 必须手动 `CaptureScene()`。
- `RT_HeightWet` 用 `RGBA16f` 保存湿身数据。

---

### 第 26 节：湿身淡出 Fade

#### 解决的问题

加入 `M_Fade`，每帧把 `RT_HeightWet` 通过 Fade 材质重新绘制回自身，实现湿身痕迹逐渐衰减。

#### 新增变量

| 变量 | 类型 | 作用 | 蓝图对应 |
|---|---|---|---|
| `M_Fade` | `UMaterialInterface*` | 湿身淡出材质 | `M_Fade` |
| `MID_Fade` | `UMaterialInstanceDynamic*` | Fade 动态材质 | `MID_Fade` |

#### 新增函数

无新增主要函数，Fade 逻辑直接写在 `Tick()`。

#### 核心代码与解释

```cpp
MID_Fade = UKismetMaterialLibrary::CreateDynamicMaterialInstance(this, M_Fade);
MID_Fade->SetTextureParameterValue("Heightfield_Fade", RT_HeightWet);
```

解释：

- 创建 Fade 材质实例。
- 把当前湿身 RT 作为输入。
- 蓝图对应逻辑：Create Dynamic Material Instance，设置 `Heightfield_Fade`。

```cpp
UKismetRenderingLibrary::BeginDrawCanvasToRenderTarget(this, RT_HeightWet, Canvas, Size, Context);
Canvas->K2_DrawMaterial(MID_Fade, FVector2D(0.f, 0.f), Size, FVector2D(0.f, 0.f));
UKismetRenderingLibrary::EndDrawCanvasToRenderTarget(this, Context);
```

解释：

- 每帧用 Fade 材质覆盖绘制 `RT_HeightWet`。
- 材质内部负责让湿身值逐渐变弱。
- 蓝图对应逻辑：Tick 中 `Draw Material to Render Target`。

#### 本节数据流

```text
RT_HeightWet
→ MID_Fade.Heightfield_Fade
→ Draw 回 RT_HeightWet
→ 湿身痕迹衰减
```

#### 注意点

- 读写同一张 RT 时依赖材质和 Draw 流程，效果需结合材质确认。
- 本节 `Painted()` 中更新角色材质 Heightfield 的代码被注释，因为 BeginPlay 已经绑定过 RT。

---

### 第 27，28 节：水面触发角色湿身

#### 解决的问题

把湿身逻辑从测试角色扩展到正式 `AWaterSimulationCharacter`，并让 `AWaterSim` 通过 `WetCaptureVolume` 判断角色是否在水里，自动调用角色的 `Painted()`。

#### 新增变量

| 变量 | 类型 | 作用 | 蓝图对应 |
|---|---|---|---|
| `WetCaptureVolume` | `UBoxComponent*` | 判断角色湿身范围 | `WetCaptureVolume` |
| `WetCaptureActor` | `TArray<AActor*>` | 湿身范围内的角色列表 | Wet Capture Actor |
| `bCanWetEffect` | `bool` | 是否执行湿身绘制 | `Can Wet Effect` |

`AWaterSimulationCharacter` 中新增湿身相关变量与 `ATestCharacter` 第 26 课基本一致：

| 变量 | 类型 | 作用 | 蓝图对应 |
|---|---|---|---|
| `SceneCaptureComponent2D` | `USceneCaptureComponent2D*` | 角色湿身捕获 | 角色 SceneCapture |
| `RT_HeightWet` | `UTextureRenderTarget2D*` | 湿身 RT | `RT_HeightWet` |
| `M_Unwrapper` | `UMaterialInterface*` | 湿身绘制材质 | `M_Unwrapper` |
| `M_Fade` | `UMaterialInterface*` | 湿身淡出材质 | `M_Fade` |

#### 新增函数

| 函数 | 执行时机 | 作用 | 蓝图对应 |
|---|---|---|---|
| `InitializeWetCaptureOverlap()` | BeginPlay 延迟初始化 | 获取已在湿身范围内的角色 | Get Overlapping Actors |
| `OnWetCaptureBeginOverlap()` | 角色进入湿身范围 | 加入湿身列表，允许绘制 | Wet Begin Overlap |
| `OnWetCaptureEndOverlap()` | 角色离开湿身范围 | 移除湿身列表，关闭绘制 | Wet End Overlap |
| `OnWetEffect()` | `AWaterSim::Tick()` | 对角色调用 `Painted()` | 自动湿身 Tick |
| `CalculatePaintedLocation()` | `OnWetEffect()` 内 | 计算湿身绘制位置与半径 | 计算水位交界区域 |
| `InitializeWetEffect()` | 角色 BeginPlay | 初始化角色湿身 RT / MID | 角色湿身初始化 |
| `DrawRenderTarget()` | 角色 Tick | 每帧绘制 Fade | 湿身淡出 Tick |

#### 核心代码与解释

```cpp
WetCaptureVolume = CreateDefaultSubobject<UBoxComponent>(TEXT("WetCaptureVolume"));
WetCaptureVolume->SetupAttachment(DefaultSceneRoot);
```

解释：

- 新增湿身检测体积。
- 与 `SimulationCaptureVolume` 分工不同：前者控制角色湿身，后者控制水面波纹捕获。
- 蓝图对应逻辑：添加 `WetCaptureVolume`。

```cpp
WetCaptureVolume->OnComponentBeginOverlap.AddDynamic(this, &AWaterSim::OnWetCaptureBeginOverlap);
WetCaptureVolume->OnComponentEndOverlap.AddDynamic(this, &AWaterSim::OnWetCaptureEndOverlap);
```

解释：

- 绑定湿身范围进入 / 离开事件。
- 蓝图对应逻辑：Wet Volume 的 BeginOverlap / EndOverlap。

```cpp
if (AWaterSimulationCharacter* Character = Cast<AWaterSimulationCharacter>(OtherActor))
{
    WetCaptureActor.AddUnique(Character);
    bCanWetEffect = true;
}
```

解释：

- 只有正式角色类才进入湿身逻辑。
- `Cast` 成功后加入列表。
- 蓝图对应逻辑：`Cast To WaterSimulationCharacter` → `Add Unique` → 设置 `bCanWetEffect`。

```cpp
void AWaterSim::OnWetEffect()
{
    if (bCanWetEffect)
    {
        for (TActorIterator<AWaterSimulationCharacter> It(GetWorld()); It; ++It)
        {
            FVector HitLocation;
            float Radius;
            CalculatePaintedLocation(*It, HitLocation, Radius);
            It->Painted(HitLocation, Radius);
        }
    }
}
```

解释：

- Tick 中自动给角色画湿身。
- 这里遍历了世界中所有 `AWaterSimulationCharacter`，不是只遍历 `WetCaptureActor`。
- 此处需要结合上下文确认：如果场景里有多个角色，可能会让不在湿身列表内的角色也执行 `Painted()`。
- 蓝图对应逻辑：Tick 中根据湿身状态调用角色 `Painted`。

```cpp
const FVector CharacterLoc = Character->GetCapsuleComponent()->GetComponentLocation();
const float BottomHeight = CharacterLoc.Z - Character->GetCapsuleComponent()->GetUnscaledCapsuleHalfHeight();
const float TopHeight = WetCaptureVolume->GetComponentLocation().Z + WetCaptureVolume->GetScaledBoxExtent().Z;
const float HalfHeight = (TopHeight - BottomHeight) / 2;

OutHitLocation = FVector(CharacterLoc.X, CharacterLoc.Y, HalfHeight + BottomHeight + 10.f);
OutRadius = HalfHeight;
```

解释：

- 用角色胶囊底部和水体顶部估算湿身高度。
- `OutHitLocation` 是绘制中心。
- `OutRadius` 是湿身范围半径。
- 蓝图对应逻辑：根据胶囊高度和水面高度计算湿身绘制范围。

```cpp
void AWaterSimulationCharacter::Painted(const FVector& HitLocation, const float& InRadius)
{
    UMaterialInterface* OLD_Material = GetMesh()->GetMaterial(0);
    GetMesh()->SetMaterial(0, M_Unwrapper);
    GetMesh()->SetVectorParameterValueOnMaterials("Hit Location", HitLocation);
    GetMesh()->SetScalarParameterValueOnMaterials("Radius", InRadius);
    SceneCaptureComponent2D->CaptureScene();
    GetMesh()->SetMaterial(0, OLD_Material);
}
```

解释：

- 正式角色中的湿身绘制流程与测试角色一致。
- 临时切换到 Unwrapper，捕获后恢复原材质。
- 蓝图对应逻辑：设置湿身绘制参数 → CaptureScene → 恢复材质。

#### 本节数据流

```text
角色进入 WetCaptureVolume
→ bCanWetEffect = true
→ AWaterSim.Tick 调用 OnWetEffect
→ CalculatePaintedLocation
→ Character.Painted
→ 角色 SceneCapture 写 RT_HeightWet
→ 角色材质读取 Heightfield
→ DrawRenderTarget 每帧 Fade
```

#### 注意点

- `WetCaptureVolume` 控制湿身，不控制水面波纹。
- `SimulationCaptureVolume` 控制水面捕获，不控制角色湿身。
- `OnWetEffect()` 当前遍历所有正式角色，此处需要结合上下文确认是否符合设计。
- `bCanWetEffect` 是全局布尔，如果多个角色进出水体，可能需要更细的列表状态管理。

## 4. 蓝图到 C++ 的对应关系

| 蓝图节点 / 逻辑 | C++ 对应实现 | 说明 |
|---|---|---|
| BeginPlay | `BeginPlay()` | 初始化 RT、MID、绑定事件 |
| Tick | `Tick(float DeltaTime)` | 每帧捕获、模拟、法线、湿身更新 |
| Construction Script | `OnConstruction()` + `PostInitializeComponents()` + `OnConstructionScript()` | 创建水面 MID，设置组件姿态 |
| Add Component | `CreateDefaultSubobject<>()` | 构造函数中创建 Plane、SceneCapture、Box |
| Create Dynamic Material Instance | `CreateDynamicMaterialInstance()` / `UKismetMaterialLibrary::CreateDynamicMaterialInstance()` | 创建运行时可改参数的 MID |
| Set Material | `Plane->SetMaterial()` / `GetMesh()->SetMaterial()` | 把 MID 或临时材质赋给组件 |
| Set Texture Parameter Value | `MID->SetTextureParameterValue()` | 传入 `Capture`、`Heightfield`、`Heightfield Normal` 等 |
| Set Scalar Parameter Value | `MID->SetScalarParameterValue()` | 传入 `Canvas Size`、`Texture Resolution`、`Radius` |
| Set Vector Parameter Value | `MID->SetVectorParameterValue()` / `SetVectorParameterValueOnMaterials()` | 传入位置参数 |
| Draw Material to Render Target | `UKismetRenderingLibrary::DrawMaterialToRenderTarget()` 或 Canvas 绘制 | 把材质计算结果写入 RT |
| Clear Render Target | `UKismetRenderingLibrary::ClearRenderTarget2D()` | 初始化清空 RT |
| Create Render Target 2D | `UKismetRenderingLibrary::CreateRenderTarget2D()` | 运行时创建 RT |
| SceneCapture TextureTarget | `SceneCaptureComponent2D->TextureTarget = RT_Capture` | 指定捕获输出 |
| ShowOnlyActorComponents | `SceneCaptureComponent2D->ShowOnlyActorComponents(Actor)` | 加入捕获列表 |
| Remove ShowOnlyActorComponents | `RemoveShowOnlyActorComponents(Actor)` | 移除捕获列表 |
| Get Overlapping Actors | `SimulationCaptureVolume->GetOverlappingActors()` | 初始化已有重叠对象 |
| BeginOverlap / EndOverlap | `OnComponentBeginOverlap.AddDynamic()` / `OnComponentEndOverlap.AddDynamic()` | 绑定 C++ 事件 |
| 三缓冲轮换 | `HeightIndex = (HeightIndex + 1) % 3` | 当前写入 RT 索引 |
| 取前一帧 / 前两帧 | `GetLastRenderTargetTexture(HeightIndex, 1/2)` | WaveSimulation 历史输入 |
| WaveNormal 生成 | `WaveNormal()` | Height RT → `RT_HeightNormal` |
| WaterSurface 材质参数传递 | `MID_WaterSurface->SetTextureParameterValue()` | 传 `Heightfield` 和 `Heightfield Normal` |

## 5. 关键系统拆解

### SceneCapture 捕获

C++ 在 Constructor 中创建：

```cpp
SceneCaptureComponent2D = CreateDefaultSubobject<USceneCaptureComponent2D>(TEXT("SceneCaptureComponent2D"));
SceneCaptureComponent2D->ProjectionType = ECameraProjectionMode::Orthographic;
SceneCaptureComponent2D->PrimitiveRenderMode = ESceneCapturePrimitiveRenderMode::PRM_UseShowOnlyList;
SceneCaptureComponent2D->CaptureSource = ESceneCaptureSource::SCS_FinalColorLDR;
```

解释：

- 正交捕获水面上方区域。
- ShowOnly 列表决定哪些 Actor 会写入 `RT_Capture`。
- `TextureTarget` 在初始化时指定：

```cpp
SceneCaptureComponent2D->TextureTarget = RT_Capture;
```

蓝图对应逻辑：SceneCapture 组件设置 + TextureTarget + ShowOnly 列表。

### WaveCompute

```cpp
MID_WaveCompute->SetTextureParameterValue("Capture", RT_Capture);
MID_WaveCompute->SetVectorParameterValue("Capture Location", PlayerLoc);
UKismetRenderingLibrary::DrawMaterialToRenderTarget(
    this, GetRenderTargetTexture(HeightIndex), MID_WaveCompute);
```

解释：

- 输入：`RT_Capture`、捕获位置、水面位置、Canvas Size、Texture Resolution。
- 输出：当前 Height RT。
- 作用：把捕获图转换成可以参与模拟的高度输入。
- 蓝图对应逻辑：`M_WaveCompute` 参数设置 + `Draw Material to Render Target`。

### WaveSimulation

```cpp
MID_WaveSimulation->SetTextureParameterValue("Previous Height 1", GetLastRenderTargetTexture(HeightIndex, 1));
MID_WaveSimulation->SetTextureParameterValue("Previous Height 2", GetLastRenderTargetTexture(HeightIndex, 2));
DrawToRenderTargetSetTextureToMaterial(GetRenderTargetTexture(HeightIndex), MID_WaveSimulation, MID_WaterSurface);
```

解释：

- 输入：前一帧、前两帧 Height RT。
- 输出：当前 Height RT。
- 写完后把当前 Height RT 设置给水面材质 `Heightfield`。
- 蓝图对应逻辑：三缓冲 WaveSimulation。

### WaveNormal

```cpp
MID_WaveNormal->SetTextureParameterValue("Heightfield Height", GetRenderTargetTexture(HeightIndex));
UKismetRenderingLibrary::DrawMaterialToRenderTarget(this, RT_HeightNormal, MID_WaveNormal);
MID_WaterSurface->SetTextureParameterValue("Heightfield Normal", RT_HeightNormal);
```

解释：

- 输入：最新 Height RT。
- 输出：`RT_HeightNormal`。
- 最终水面材质读取它作为交互法线。
- 蓝图对应逻辑：Height RT → M_WaveNormal → RT_HeightNormal → WaterSurface Normal。

### WaterSurface

水面材质主要通过两个纹理参数接收结果：

| 参数名 | 来源 | 用途 |
|---|---|---|
| `Heightfield` | 当前 Height RT | WPO 高度位移 |
| `Heightfield Normal` | `RT_HeightNormal` | 动态法线 |

关键代码：

```cpp
MID_WaterSurface->SetTextureParameterValue("Heightfield", DrawToRT);
MID_WaterSurface->SetTextureParameterValue("Heightfield Normal", RT_HeightNormal);
```

解释：

- `Heightfield` 决定水面顶点上下运动。
- `Heightfield Normal` 决定光照反射里的波纹细节。
- 蓝图对应逻辑：给最终水面材质传 Height 和 Normal 两张 RT。

## 6. UE C++ 语法点速记

| 语法 / 类型 | 简洁解释 |
|---|---|
| `UCLASS()` | 让类进入 UE 反射系统，可被引擎识别 |
| `UPROPERTY()` | 让变量被 UE 管理，可编辑、序列化、避免 GC 问题 |
| `UFUNCTION()` | 让函数进入反射系统，Overlap 的 `AddDynamic` 需要 |
| `Constructor` | Actor 创建时执行，适合 `CreateDefaultSubobject` |
| `BeginPlay()` | 游戏开始或 Actor Spawn 后执行，适合运行时 RT / MID 初始化 |
| `Tick()` | 每帧执行，适合驱动捕获、模拟和材质更新 |
| `OnConstruction()` | 编辑器构造 / 属性变化时执行，接近蓝图 Construction Script |
| `CreateDefaultSubobject` | C++ 添加默认组件，对应蓝图组件树 |
| `UMaterialInterface` | 材质或材质实例的通用引用，常作为模板 |
| `UMaterialInstanceDynamic` | 动态材质实例，可运行时改参数 |
| `UTextureRenderTarget2D` | 可被 SceneCapture 或 DrawMaterial 写入的运行时纹理 |
| `UKismetRenderingLibrary` | 蓝图渲染节点的 C++ 工具库，如创建 / 清空 / 绘制 RT |
| `TObjectPtr` | UE5 推荐的 UObject 指针包装；本源码仍主要使用裸指针，逻辑相同但 GC 表达不如 `TObjectPtr` 明确 |

## 7. C++ 版本核心理解

- 蓝图版本是把节点串起来。
- C++ 版本是把同一条数据流封装进变量、函数和 Tick 顺序里。
- `BeginPlay` 负责资源初始化：创建 RT、清空 RT、创建 MID、绑定 SceneCapture。
- `Tick` 负责每帧调度：捕获输入、固定步长模拟、生成法线、更新水面材质。
- `SceneCapture` 只负责把交互对象写进 `RT_Capture`。
- `WaveCompute` 把捕获结果变成高度输入。
- `WaveSimulation` 用三缓冲保存时间状态并扩散波纹。
- `WaveNormal` 把高度变化转成法线变化。
- `WaterSurface` 只读取最终 Height 和 Normal，不直接关心捕获过程。
- 最容易出错的是：MID 为空、参数名不一致、RT 格式不对、三缓冲索引错、Tick 执行顺序错、ShowOnly 列表没维护好。
