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
