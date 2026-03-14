# TurboSequence 仓库功能与实现原理分析

## 1. 这个库是做什么的

TurboSequence 是一个 Unreal Engine 5 插件，目标是用 **GPU Instancing + Niagara/ISMC** 的方式渲染和驱动大规模“类骨骼动画”角色（如人群），以减少传统逐实例 SkeletalMesh Draw Call 带来的 CPU 压力。

它并不是 VAT（顶点动画贴图）那种“纯预烘焙位移”，而是保留了骨骼驱动特性，所以支持 IK、Socket、分骨骼层混合、Root Motion、LOD 等运行时能力。

## 2. 总体架构

插件由 4 个模块组成：

1. **TurboSequence_Lf（Runtime）**：核心数据、实例管理、动画求解与渲染桥接。
2. **TurboSequence_Shader_Lf（Runtime）**：Compute Shader 封装，负责 GPU 端动画/骨骼变换计算。
3. **TurboSequence_Editor_Lf（Editor）**：控制面板与资产烘焙工具，把 Skeletal Mesh 转为 TS 可运行资产。
4. **TurboSequence_HelperModule_Lf（Runtime）**：通用辅助能力。

运行时还依赖 `Niagara` 与 `NiagaraNanite` 插件。

## 3. 核心数据流（从资产到屏幕）

### 阶段 A：离线/编辑期资产准备

编辑器侧通过 Control Panel 完成：

- 选择参考 Skeletal Mesh。
- 生成静态网格 LOD（供实例化渲染）。
- 提取并写入骨骼/权重等数据（MeshData 与纹理）。
- 调整全局纹理尺寸（Transform/SkinWeight/AnimationLibrary）以匹配目标规模。

产物是 `UTurboSequence_MeshAsset_Lf`，包含：参考骨架、渲染系统、LOD 配置、动画库、MeshData（如 CPU->GPU 骨骼映射）等。

### 阶段 B：运行期实例管理

`ATurboSequence_Manager_Lf` 维护两个核心库：

- **GlobalLibrary（GameThread）**：运行时实例、更新组、相机视图、动画输入等。
- **GlobalLibrary_RenderThread（RenderThread）**：GPU 求解所需紧凑缓冲参数。

实例通过 `AddSkinnedMeshInstance_GameThread` 加入系统；可再按业务放入 Update Group，实现分组调度与分帧更新。

### 阶段 C：每帧求解

`SolveMeshes_GameThread` 是主入口，核心顺序是：

1. 更新相机视图（用于可见性/LOD）。
2. 遍历更新组中的 Mesh（并行）：
   - 计算动画状态与混合；
   - 更新可见性与 LOD；
   - 处理 Footprint 覆盖逻辑（可覆写“是否可见/是否动画”）；
   - 组装 RenderThread 所需最小数据（动画片段、IK 数据等）。
3. 将本帧动画库增量数据排队到 RenderThread。
4. 触发 Compute Shader Dispatch，在 GPU 端把动画库 + IK + 引用姿势组合为骨骼变换输出纹理。

### 阶段 D：实例渲染更新

- **Niagara 路径**：把位置/旋转/缩放/LOD/CustomData 数组写入 Niagara Data Interface。
- **ISMC 路径**：逐实例更新 Transform 与 CustomData。
- 材质端通过 `TS_MaterialFunctions.ush` 的 `VertexSkin` / `VertexSkinNormal` 从纹理取骨骼矩阵并做顶点变形，得到最终动画网格。

## 4. 实现原理拆解

### 4.1 “静态网格实例 + 材质顶点蒙皮”替代传统 SkeletalMesh

传统方式是每个角色都作为独立 SkeletalMesh 提交与求值，CPU 压力和 Draw Call 容易爆炸。

TurboSequence 的思路是：

- 网格渲染层用 StaticMesh Instancing（Niagara 或 ISMC）合批；
- 骨骼动画不在每个 SkeletalMesh 组件上跑，而是集中计算后写入纹理；
- 顶点着色/材质阶段读取纹理执行 Skinning。

这样“渲染提交”由“按实例”转向“按原型/材质批次”，显著降低 Draw Call 与 CPU 管理成本。

### 4.2 双线程数据镜像（GameThread -> RenderThread）

运行时维护 GameThread 与 RenderThread 的双份全局库：

- 游戏线程负责高层逻辑：实例生灭、动画状态机、可见性判定、LOD 决策、Footprint 回调。
- 渲染线程只接收紧凑参数，专注 GPU 求解与输出。

通过 `ENQUEUE_RENDER_COMMAND` 把动画 chunk、IK 数据等增量传递，降低线程间耦合与同步成本。

### 4.3 Compute Shader 统一做骨骼结果写出

`FMeshUnitComputeShader_Params_Lf` 内含每帧所需的关键输入：

- 每 Mesh 的 custom data 索引；
- 动画起止帧/权重/层混合；
- IK 输入与区间；
- 参考姿势与索引映射；
- 贴图句柄（AnimationLibrary、当前/上一帧 Transform Texture）。

Compute Shader 执行后，把结果写到 `TransformTexture_CurrentFrame`（并维护 PreviousFrame），供材质在顶点阶段采样，完成最终变形。

### 4.4 动画库与分块增量写入

系统会将动画 Pose 数据组织成可 GPU 读取的“动画库纹理”，并支持 chunked 增量更新，避免每帧全量上传。

这与“更新组（UpdateGroup）”结合后，可按业务节奏让不同群组以不同频率求解，进一步节省 CPU/GPU 带宽。

### 4.5 可见性、LOD 与距离更新策略

每个实例会经过：

- Frustum 可见性判定；
- 自动 LOD 选择；
- 距离驱动的更新频率调节（Distance Updating）。

远处实例可降低动画更新成本，近处保持精度，形成典型 crowd 分级优化。

## 5. 能力边界与适用场景

适合：

- UE5 大规模人群/单位（文档给出约 1 万到 5 万量级偏 CPU bound）。
- 需要运行时 IK/Socket/RootMotion/分骨骼混合，而不是纯离线动画位移。

不适合：

- 纯蓝图项目（仓库文档明确不支持 blueprint-only）。
- 移动端 / macOS（文档写明当前支持 Windows/Linux）。
- MetaHuman 这类高复杂资产（文档建议改用 UE Nanite Skeletal Mesh）。

## 6. 一句话总结

TurboSequence 本质上是一个“**把 SkeletalMesh 动画求值与渲染拆开**”的系统：

- 用编辑器预处理把骨骼/权重数据打包；
- 用运行时管理器集中求解动画并喂给 Compute Shader；
- 用实例化渲染 + 材质顶点蒙皮完成大规模角色绘制。

它牺牲了部分原生 SkeletalMesh 管线的一体化便利，换来 crowd 场景下更高的吞吐和可控的性能结构。

---

## 7. 关键实现代码与伪代码（便于快速落地）

下面补充“能直接对照源码理解”的代码片段/伪代码。

### 7.1 运行时主循环伪代码（`SolveMeshes_GameThread`）

```cpp
void SolveMeshes_GameThread(float DeltaTime, UWorld* World, FTurboSequence_UpdateContext_Lf Ctx)
{
    if (RuntimeMeshes.Empty() || !ManagerInstance)
        return;

    // 1) 更新相机（用于可见性与 LOD）
    UpdateCameras(CameraViews, LastFrameCameraTransforms, World, DeltaTime, Ctx.CustomCameraInfo);

    // 2) 并行遍历本组 Mesh
    ParallelFor(UpdateGroups[Ctx.GroupIndex].RawIDs.Num(), [&](int32 Index)
    {
        int32 MeshID = UpdateGroups[Ctx.GroupIndex].RawIDs[Index];
        Runtime = RuntimeSkinnedMeshes[MeshID];
        Reference = PerReferenceData[Runtime.DataAsset];

        SolveAnimations(Runtime, Reference, DeltaTime, FrameCount);     // 动画状态推进
        IsMeshVisible(Runtime, Reference, CameraViews);                 // 可见性检查
        UpdateCullingAndLevelOfDetail(Runtime, Reference, CameraViews); // LOD / Cull
        UpdateDistanceUpdating(Runtime, DeltaTime);                     // 距离更新频率

        // 组装渲染线程最小输入（动画片段、IK 数据等）
        EnqueueRenderThreadInput(Runtime, Reference);

        ClearIKState(Runtime);
    });

    // 3) 动画库增量上传（chunked）
    if (AnimationLibraryDataAllocatedThisFrame.Num())
    {
        ENQUEUE_RENDER_COMMAND(AddLibraryChunked)(...);
    }
}
```

### 7.2 渲染线程与 Compute Shader 伪代码

```cpp
void SolveMeshes_RenderThread(FRHICommandListImmediate& RHICmdList)
{
    ResizeBuffers(GlobalLibrary_RenderThread, NumMeshes);

    ParallelFor(NumMeshes, [&](int32 i)
    {
        RuntimeRT = RuntimeSkinnedMeshesRT[HashMap[i]];
        if (!RuntimeRT.bIsVisible)
            return;

        int32 MeshIndex = NumMeshesVisibleCurrentFrame++;
        Params.PerMeshCustomDataIndex_RenderThread[MeshIndex] = MeshIDToGlobalIndex[RuntimeRT.MeshID];
        Params.NumAnimations += RuntimeRT.AnimationMetaData_RenderThread.Num();

        if (RuntimeRT.bIKDataInUse)
            AccumulateIKParams(RuntimeRT, Params);
    });

    Params.NumMeshes     = NumMeshesVisibleCurrentFrame;
    Params.NumAnimations = max(Params.NumAnimations, 1);
    Params.NumIKData     = max(Params.NumIKData, 1);

    DispatchMeshUnitComputeShader(Params, TransformTexture_CurrentFrame);
}
```

### 7.3 材质顶点蒙皮逻辑伪代码（`VertexSkin`）

```hlsl
for influenceBlock in 0..2:                 // 每个顶点最多 12 influence（3 组 * 4）
    Indices = SkinWeightTexture[VertexBase + influenceBlock*2]
    Weights = SkinWeightTexture[VertexBase + influenceBlock*2 + 1]

    for w in 0..3:
        if earlyOut && Weights[w] == 0:
            return

        BoneBase = TransformTextureOffset + Indices[w] * Settings0_W
        M0 = TransformTexture_Current[BoneBase + 0]
        M1 = TransformTexture_Current[BoneBase + 1]
        M2 = TransformTexture_Current[BoneBase + 2]

        Weight = Weights[w] / 255.0
        FinalCurrentBlendPosition += (mul(float3x4(M0,M1,M2), float4(VertexPos,1)) - VertexPos) * Weight
```

### 7.4 实例渲染桥接伪代码（Niagara / ISMC）

```cpp
for each RenderData:
    if (UseISMC)
    {
        for each InstanceIndex:
            UpdateInstanceTransform(InstanceIndex, Position/Rotation/Scale);
            SetCustomData(InstanceIndex, CustomData);
        MarkRenderStateDirty();
    }
    else // Niagara
    {
        SetNiagaraArrayUInt8 (LOD);
        SetNiagaraArrayFloat (CustomData);
        SetNiagaraArrayPosition(Position);
        SetNiagaraArrayVector4(Rotation);
        SetNiagaraArrayVector (Scale);
        SetEmitterFixedBounds(Bounds);
    }
```

### 7.5 一个最小调用流程（业务代码视角）

```cpp
// BeginPlay
Instance = ATurboSequence_Manager_Lf::AddSkinnedMeshInstance_GameThread(SpawnData, SpawnTransform, World);
ATurboSequence_Manager_Lf::AddInstanceToUpdateGroup_Concurrent(0, Instance);
ATurboSequence_Manager_Lf::PlayAnimation_Concurrent(Instance, WalkAnim, PlaySettings);

// Tick
FTurboSequence_UpdateContext_Lf Ctx;
Ctx.GroupIndex = 0;
ATurboSequence_Manager_Lf::SolveMeshes_GameThread(DeltaTime, World, Ctx);
```

> 上面这段与仓库文档中的最小 Demo 思路一致：先创建实例、放入更新组、播放动画，再在 Tick 中持续调用求解。

---

## 8. 关于“动画融合能力”的直接结论（FAQ）

> Q1：这个插件有实现动画融合吗？

有，且是多种融合方式并存：

1. **多动画并行权重融合**：同一实例维护 `AnimationMetaData` 数组，每条动画都有独立权重与时间推进。  
2. **BlendSpace 融合**：`PlayBlendSpace` 会把每个 sample 作为动画条目播放，再按 BlendSpace 结果动态回写 sample 的权重与时间。  
3. **按骨骼层融合**：通过 `BoneLayerMasks` 和 `ForceMode`（`None/PerLayer/AllLayers`）实现分层叠加或同层替换。

> Q2：最多多少个动作融合？

源码没有给固定的“硬编码上限常量（例如最多 4 个/8 个）”，因为内部核心容器是动态数组（`TArray`）并可持续 `Add`。  
但工程上存在**实际约束**：

- 动画索引/层索引中有多处 `uint16/int16`，理论上会受 16 位范围影响；
- 每帧动画条目越多，CPU 侧求解、RenderThread 打包、GPU 参数上传与计算成本越高；
- BlendSpace 的融合条目数还受该 BlendSpace 自身 sample 数影响（`NumSamples = BlendSpace->GetBlendSamples().Num()`）。

因此更准确的说法是：**逻辑上“可多条融合”，但上限由数据类型与性能预算共同决定，不是文档里给死的固定值。**

> Q3：相比 Animation Blueprint 的自由度，哪些方面有欠缺？

TurboSequence 的定位是“大规模实例吞吐优先”，与 ABP 的“图形化动画表达”是两种取舍。和 ABP 相比，主要欠缺在：

1. **工作流层面**：更偏 C++/API 编排（`Play/Tweak/Solve`）而非完整 AnimGraph/StateMachine 图编辑体验。  
2. **控制器负担**：项目通常要自己写 Animation Controller（文档也明确建议这样做），不像 ABP 那样节点化开箱即用。  
3. **蓝图便利性**：虽有蓝图 API，但仓库明确不支持 blueprint-only 项目，且官方建议 C++ 路径。  

换句话说：

- 如果目标是 **大量角色并发 + 可控性能结构**，TurboSequence 的收益很大；
- 如果目标是 **角色级复杂动画图编排自由度**，ABP 的创作体验通常更强。

### 8.1 对照式伪代码（ABP 思维 vs TurboSequence 思维）

```cpp
// ABP 思维（概念化）
// 状态机/过渡规则/LayeredBlendPerBone 在 AnimGraph 内声明，运行时自动驱动
AnimGraph()
{
    BasePose = StateMachine(Idle, Walk, Run, Jump, ...);
    UpperBody = SlotOrLayer(AttackMontage, AimOffset, ...);
    FinalPose = LayeredBlendPerBone(BasePose, UpperBody, BoneMasks);
}


// TurboSequence 思维（概念化）
// 业务代码显式管理播放、层、权重与每帧 Solve
BeginPlay()
{
    Mesh = AddSkinnedMeshInstance(...);
    AddInstanceToUpdateGroup(0, Mesh);

    PlayAnimation(Mesh, Idle,   Settings{BoneLayerMasks=[], Weight=1.0});
    PlayAnimation(Mesh, Aim,    Settings{BoneLayerMasks=[Spine...], Weight=0.6});
    PlayBlendSpace(Mesh, MoveBS, Settings{BoneLayerMasks=[], ForceMode=PerLayer});
}

Tick(DeltaTime)
{
    // 根据输入/AI 调整权重与 BlendSpace 采样位置
    TweakAnimation(Aim, Settings{Weight=AimWeight});
    TweakBlendSpace(MoveBS, FVector3f(Speed, Direction, 0));

    // 显式求解
    SolveMeshes_GameThread(DeltaTime, World, UpdateContext{GroupIndex=0});
}
```
