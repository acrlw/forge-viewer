# forge-viewer 编辑器加载、meshdir 与大场景性能专项复核

> 日期：2026-08-26<br>
> 审查基线：远端 `55bb83d`；二次回归修复基线：`a6ac479`<br>
> 方法：源码路径审计、G1 23-DOF 实物模型复现、CPU microbenchmark、定向回归与已有报告交叉复核。
>
> 范围说明：4096 个机器人只是用于识别架构边界的假设压力样例，不是当前产品规模或本轮实现目标。
> 本轮不引入 instance culling、LOD、indirect draw 或新的大规模 batch renderer。

## 结论摘要

1. Unitree URDF 的 `meshdir="meshes"` 与 `filename="meshes/xxx.STL"` 在普通目录布局下是冗余描述，
   MuJoCo 会按规则寻找 `meshes/meshes/xxx.STL`。这可以理解为常见导出器兼容问题，但不能无条件忽略
   `meshdir`：双层目录真实存在时，该写法是合法的。
2. UI 模型加载经过 Python binding 进入 MuJoCo 原生 `MjSpec` parser/compiler。binding 没有可靠的阶段或
   百分比回调，因此界面应显示动作、完整路径、耗时和排队数，不能伪造进度。
3. G1 上“加 plane / undo 各卡约 4.5 秒”的主因不是 MuJoCo compile。profile 中 compile 约 30–35 ms，
   真正热点是每次结构刷新都预构建 145 万条未启用的 mesh BVH 调试记录。改为按 `FrameNeeds.bvh` 延迟
   物化后，实测 add plane 约 117 ms、undo 约 95 ms。
4. light 路径没有发现相机曾出现的 O(N²) 查找；灯光调度是 O(L)，但阴影是乘法成本：最多一个方向光
   级联阴影和八个局部阴影，其中点光需要六个面，并对每个 opaque bucket 重画。
5. 作为远期边界分析，4096 个机器人不是单一结论。4096 个单部件实例在 CPU 更新上很轻；4096×30 部件为 122,880 个
   instance，CPU transform 更新仍约 7 ms，但每帧完整 instance upload 已达 15.5–16.9 MiB，60 Hz 接近
   1 GiB/s，且目前没有实例级 frustum/LOD culling。GPU 顶点、像素、阴影和透明 draw call 会成为主风险。
6. 高基数 debug draw 应明确使用 plural batch API。4096 个 scalar arrows 与一个 arrows batch 最终几何量
   相同，但前者在 Python 调用、pickle 和 retained ID 上明显更贵。
7. 二次回归确认 MuJoCo world geom 过去沿用 body 0，导致新建有限 Plane 的 object ID 为 0，无法在 viewport
   拾取或高亮；Workspace 还拒绝把尺寸编辑转发给 MuJoCo。现已给有限 world Plane 分配跨 topology recompile
   稳定的 object ID，并支持 Plane 位姿、长宽、重命名、Undo/Redo 和 spec 持久化。
8. link DOF gizmo 已按真实约束实现：hinge 只显示轴向旋转环，slide 只显示轴向箭头，ball 使用旋转 gizmo，
   free joint 继续走自由刚体 transform；多直接关节必须在 Joints 面板明确选择。ball 四元数通过一次原子
   qpos batch 写入，只调用一次 `mj_forward`。

## 本轮用户问题与处理状态

| 问题 | 当前处理 | 状态 |
|---|---|---|
| 相机 far plane 太近 | MuJoCo camera hint 最低使用 `200 * extent`；Camera 面板可调到 100000 m | 已修 |
| 新建 plane 会被机器人撞歪 | topology-capable MuJoCo workspace 创建 world-owned `geom:plane`，body 0、静态可碰撞 | 已修 |
| 加载阻塞且只有全屏等待 | 文件对话框、拖放、Open/Reload 进入单 worker 队列；旧 viewport 保持刷新，中央浮动 widget 显示动作、路径、耗时、队列 | 已修 |
| 错误不便反馈 | 文件错误、资源修复、Inspector 错误和 Control 消息都提供复制入口 | 已修 |
| Unsaved changes 裁字、偏心、留白 | modal 每帧按当前 viewport 居中并限制在 work area；文件名独立换行；三按钮等宽填满一行 | 已修 |
| 点击 link 后很难找到 joint | Joints 面板优先显示所选 body 的 direct joints；运行时禁止 qpos 编辑并明确要求 Pause；大列表分页 | 已修 |
| link DOF gizmo | hinge/slide/ball/free 分别使用约束匹配的 gizmo；多 direct joint 在面板显式选择 | 已修 |
| Unitree meshes/meshes | 只在双层文件不存在、单层文件存在时，在内存中去掉重复前缀；不改源文件 | 已修 |

## 可验收需求台账

按用户提出的独立产品或工程结果拆分为 16 项，不把分析过程重复计数：

| # | 可验收结果 | 状态 | 证据或剩余边界 |
|---:|---|---|---|
| 1 | editor camera far plane 放宽 | 完成 | camera hint 最低 `200 * extent` |
| 2 | Entity Plane 是 MuJoCo world 静态碰撞地面 | 完成 | body 0；添加机器人后不倾斜 |
| 3 | Plane 可在 viewport 点击选中并高亮 | 完成 | node/source/raycast 共用非零稳定 object ID |
| 4 | Plane 可用 transform gizmo 移动/旋转 | 完成 | `SetPose` 写回 MjSpec，重建后保持 |
| 5 | Plane 可编辑长宽并支持 Undo/Redo/保存 | 完成 | 直接更新 compiled model 和 spec，不为每次拖动 compile |
| 6 | 加载不再是全屏阻塞页 | 完成 | 单 worker；旧 viewport 保持；中央 Loading widget |
| 7 | 加载显示动作、文件、耗时和排队数 | 完成 | 不伪造百分比 |
| 8 | 显示真实 parser/mesh/compiler/GPU 阶段进度 | 部分 | MuJoCo binding 无阶段回调；需跨层显式 job telemetry |
| 9 | Unsaved changes 对话框居中且按钮不裁字 | 完成（代码） | CPU/UI 测试通过，GPU gallery 仍待可用环境目视 |
| 10 | 错误文本可复制 | 完成 | 文件 modal、资源修复、Inspector、Control message |
| 11 | Unitree `meshes/meshes` 窄兼容 | 完成 | 仅磁盘证明短路径存在时内存修复 |
| 12 | 单 hinge/slide/ball/free link 的 viewport DOF gizmo | 完成 | MuJoCo 动态测试覆盖 hinge/slide/ball；free 沿用 SetPose |
| 13 | 多直接关节不猜测并可选 gizmo 目标 | 完成 | Joints 面板 `Use gizmo`，选择按 structure generation 失效 |
| 14 | 检查 camera/light/scene/UI 的 O(N²) 与高基数热点 | 完成 | 修复项和剩余 GPU 乘法风险见性能章节 |
| 15 | 直接接口与 batch 接口有真实扩展点 | 完成（当前范围） | debug plural batch、SceneFrame arrays、原子 `SetQposBatch`/remote |
| 16 | 4096 机器人专用 batch renderer / culling / LOD | 明确延期 | 用户说明当前不做；等真实规模和 GPU profile 触发 |

另有一个非上述 16 项的新架构工作：topology `ModelEditBatch` 尚未实现。当前单次 add/remove/undo 仍各 compile
一次；它不影响本轮 Plane 正确性，但多结构操作仍缺少“一批 spec 改动只 compile 一次”的事务语义。

## meshdir：错误、兼容输入还是合法输入

MuJoCo compiler 的语义是把相对 mesh filename 解析到 `modelfiledir / meshdir` 下。因此：

```xml
<compiler meshdir="meshes"/>
<mesh filename="meshes/pelvis.STL"/>
```

通常解析为：

```text
<urdf parent>/meshes/meshes/pelvis.STL
```

客观判断应分三种情况：

- `meshes/meshes/pelvis.STL` 存在：这是合法且可能有意的描述，必须保留。
- 双层不存在、`meshes/pelvis.STL` 存在：描述冗余，但可合理理解为 exporter 同时在 compiler 和 filename
  写入资源根；viewer 做窄兼容有实际价值。
- 两者都不存在：不能猜测路径，继续报告 MuJoCo 原始加载错误。

当前修复还拒绝绝对 meshdir、Windows drive path 和含 `..` 的 fallback；XML 只在内存中规范化，源 URDF
不会被重写。副作用被限制在“磁盘存在性已经证明短路径正确”的情况。

## 加载与结构编辑实际走哪条管线

### UI 加载

```text
ViewerApp load queue (worker)
  -> Session.submit(LoadAsset)
  -> WorkspaceAdapter.load
  -> MuJoCoAdapter.load
  -> mujoco.MjSpec.from_file/from_string      Python binding，原生 parser
  -> MjSpec.compile                           MuJoCo 原生 compiler
  -> adapter _install                         MjData、缓存与动态 buffer
  -> Session structure refresh                shared SceneSource
  -> render backend set_scene                 Forge 或 WGPU GPU 资源
```

G1 23-DOF 三次直接 adapter load 为约 155 / 90 / 84 ms。后台加载期间以约 1 ms 间隔模拟 UI tick，最大
观测间隔约 3.8 ms，说明当前 MuJoCo binding 的重工作没有长期占住 Python UI 线程。

真实百分比目前不可得。把“解析 XML / 解析 mesh / compile / 构建 SceneSource / 上传 GPU”写成阶段文字也需要
各层主动上报，否则只是按调用前后猜测，不应冒充真实进度。

### 添加 plane 与 undo

MuJoCo 的 `MjModel` 是编译产物，新增/删除 geom、joint、body 等 topology 操作必须从 `MjSpec` 重新 compile。
undo 恢复旧 spec 后也必须 compile。物理上正确的静态 plane 因此不能走纯 Forge visual 的直接 append。

但“一次 compile 必须发生”不代表数秒停顿合理。对 G1 的 wall-time 与 cProfile 显示：

| 路径 | 修复前 wall time | profile 主成本 | 修复后 wall time |
|---|---:|---|---:|
| Add static MuJoCo plane | ~4.53 s | `_build_bvh_records`，约 145 万 records | ~117 ms |
| Undo | ~4.55 s | 同上 | ~95 ms |
| MuJoCo compile 本身 | — | profile 下约 30–35 ms | 仍保留一次 |

G1 的 29 个 mesh 自身含 802,195 个 mesh BVH 节点；50 个 mesh geom 复用部分 mesh 后，旧代码按 geom
展开为约 1,450,066 条调试记录。BODYBVH/MESHBVH 默认关闭，却在每次 SceneSource 重建时无条件生成这些
数组。本轮新增的 `prepare_frame(FrameNeeds)` 延迟路径只在 UI 确实打开 BVH flag 时物化，第三方旧 adapter
不实现该可选 hook 也继续工作。

二次回归修复后用同一 G1 文件复测：初次 adapter load 148.3 ms，Add Plane 118.8 ms，Undo 99.2 ms。
Plane 长宽拖动不走 topology recompile，而是同步更新 compiled `geom_size` 与 editable spec，并调用一次
`mj_setConst`/`mj_forward`；因此连续尺寸编辑不会重复支付 parser/compiler 成本。

剩余事实：用户主动打开 G1 的 mesh BVH 时，完整诊断数据仍然很大，可能发生一次明显停顿和较大内存占用。
长期方案应把 BVH 表示改为“共享 mesh tree + instance transform”或按所选 depth/budget 生成，而不是恢复
默认 eager build。

## 每帧性能审计

### 已修复或已设上界

| 热点 | 原问题 | 当前处理 |
|---|---|---|
| authored camera lookup | 每个 camera override 扫 camera list，形成 O(C²) | stable ID -> slot map，O(C) compose |
| remote camera lookup | 每帧按 ID 扫 camera list | revision 时构建 map，单次 O(1) |
| camera/light editor helper | 每帧两次扫全部 hierarchy node，并逐 primitive 提交 | structure-generation cache + points/lines/arrows batches |
| camera/light diagnostic icons | 每实体多个 retained IDs | 每类 1–2 个 batch IDs |
| body/geom/site/camera/light frames | 每个 frame 一个 retained ID | `Layer.frames` 一个 batch |
| selected body joints / actuator drivers | 每帧扫描完整 metadata | Session 按 body/joint 建索引 |
| Hierarchy | 大场景默认展开并创建成千 ImGui rows | 2000+ 默认收起，visible rows 上限 512，搜索名缓存 |
| Joints/Actuators | 数千 slider 每帧创建，快照 dict 大量分配 | selected-first，256+ 折叠浏览，128/page，NumPy snapshot |
| retained geometry color overrides | K 个 override 各扫 N instances，O(KN) | 小 K bounded vector mask，大 K 单次 instance pass |
| transparent single-instance sort | Python 循环逐 bucket 求距离 | 单实例 bucket fast path vectorized |
| WGPU instance staging | 每帧 `np.zeros` 整个 instance block | capacity staging buffer 复用 |
| tendon label anchor | 每 tendon 扫全部 segments，O(TS) | indexed accumulation，一次 segment pass |
| MuJoCo BVH source | 未启用也构建百万记录 | `FrameNeeds.bvh` 延迟物化 |
| remote 无 viewer | 每个训练 step 仍 pickle | 保留一个 bootstrap frame 后停止重复序列化 |

### Light 是否有 O(N²)

未发现 light 数量导致的嵌套全表查找。主要 CPU 路径是：

- Session 合成 body-attached dynamic lights：O(L)，只有 adapter 提供动态 lights 时才重建 tuple。
- UI helper：O(L)，已批量提交；selected influence 只画当前选择灯。
- renderer schedule：O(L)，shader 最多接受 100 个 active non-image lights。

风险来自渲染乘法而不是 O(L²)：

- 一个 directional shadow 最多三级 cascade。
- 最多八个 local shadow slots。
- point shadow 每盏六个 cubemap faces。
- 每个 shadow face/cascade 都会遍历 opaque buckets 并重画几何。

例如 30 个 opaque buckets、8 个 point shadow casters 的理论 shadow draw 上界接近 `30 * 8 * 6 = 1440`
（还未计 directional cascades）。因此大规模回放应默认关闭局部阴影，或只给少量关键灯 `cast_shadow`。
Forge 与 WGPU 各自复制一份 light scheduling 常量/逻辑，当前结果一致，但这是后续 parity drift 风险。

### 4096 个机器人

下面是 CPU 合成基准，不代表真实 GPU 最终帧率：

| 场景 | instances | stable structure build | dynamic transform update | Forge full upload | WGPU full upload |
|---|---:|---:|---:|---:|---:|
| 4096 × 1 part | 4,096 | ~37.3 ms（一次） | ~0.228 ms/frame | 0.516 MiB | 0.563 MiB |
| 4096 × 30 parts | 122,880 | ~1.536 s（revision 时） | ~7.06 ms/frame | 15.47 MiB | 16.88 MiB |

有利条件：相同 mesh/material 会进入 instance bucket，draw call 可以接近“每种 link mesh/material 一次”，
而不是每机器人一次。

严重风险：

1. instance record 目前把 transform、color、material、texture coefficients、object ID 全量上传；大量字段是
   静态的，但仍每帧重传。60 Hz 时 122,880 instances 约为 0.9–1.0 GiB/s host-to-GPU write。
2. instancing 只减少 draw calls，不减少 `4096 * mesh triangles` 的 vertex work 和重叠像素 shading。
3. 当前没有 instance-level frustum culling、LOD 或 GPU indirect culling；离屏机器人仍在 instance draw 中。
4. shadow 会再次放大所有可投影 instance 的几何工作。
5. transparent robots 若被拆成大量单实例 buckets，会产生排序和逐 bucket draw-call 爆炸；共享一个 bucket
   虽快，但 bucket 内没有严格逐实例透明排序，可能出现视觉顺序误差。
6. hierarchy 已限制 rows，但打开 labels、tendon/contact text 等高基数文字仍会造成 CPU layout 和不可读画面。

建议回放使用一个稳定 `SceneSource` 加 batched `SceneFrame`，不要把 4096 个机器人作为 4096 次 MuJoCo
model composition。默认关闭 labels/BVH/局部阴影，使用简化 collision/visual mesh 或 LOD；大场景真正要稳住
60 Hz，还需要拆分 static/dynamic GPU instance buffers，并增加可见性/LOD 路径。

### 4096 个速度箭头

本机一次 microbenchmark：

| 形式 | pickle | Python submit | retained IDs | pack |
|---|---:|---:|---:|---:|
| 4096 scalar `arrow` dicts | 459 KiB / 9.29 ms | 6.73 ms | 4096 | 0.089 ms |
| 1 `arrows` NumPy batch | 96.3 KiB / 0.051 ms | 0.064 ms | 1 | 0.081 ms |

最终仍是 4096 个箭头 primitive，所以 pack/GPU 几何成本接近；batch 节省的是 Python calls、字典、pickle、
retained index 与命令 budget。这正是应该公开区分 singular 与 plural API 的理由。

## 直接接口与 batch 接口的建议边界

### 保留直接接口

适用于：鼠标选择、单个 gizmo、单灯/单相机编辑、少量长期 primitive、需要独立 ID/expiration/erase 的对象。

```python
layer.arrow("selected-velocity", start, end, color)
session.submit(SetSceneCamera(camera_id, view))
```

直接接口应立即返回明确成功/失败，不应暗中把 mutually-exclusive mode 表达为多个 bool。公共
`Renderer.enable_depth_rendering()` 是 MuJoCo compatibility facade，保留该名字；内部 backend 已使用
`set_debug_view(DebugView.DEPTH)` / `set_flag(...)`。若新增统一 facade，应该是 enum output mode，而不是
`set_depth_rendering(bool)` 与 `set_segmentation_rendering(bool)` 两个可能冲突的开关。

### 使用 batch 接口

适用于：同质高基数 debug primitives、批量 pose/velocity、remote frame、一次语义提交后整体替换的数据。

```python
layer.arrows("robot-velocities", starts, ends, color)
publisher.publish_frame(frame_with_all_robot_transforms)
session.submit(SetQposBatch(indices, quaternion))
```

batch 应有一个整体 retained ID；需要独立删除/寿命时才拆分，不应为每条记录生成无意义 numeric ID。
`SetQposBatch` 先整体验证长度、整数索引、唯一性、范围和有限值，再一次写入并 forward；remote adapter 保留
同样的原子命令边界。它当前直接服务 ball joint gizmo，不是为未来假设场景预留的空接口。

### 仍缺的 batch/transaction

模型结构编辑应增加显式 `ModelEditBatch` / spec transaction：在内存 spec 上应用多个 add/remove/update，
最后只 compile 一次，并形成一个 undo entry。现有 `BeginEditTransaction` 主要合并 history，不能保证 adapter
只 compile 一次。单个 plane 与单次 undo 各自仍需要一次 compile，适合再接入通用后台 Session job；多个
结构操作则优先减少 compile 次数。

remote publisher 还需要与 batch 正交的 backpressure API，例如 `try_publish_frame()` 或 `max_publish_hz`。
当前 no-viewer 已避免重复 pickle，latest-only 也避免网络 backlog，但有 viewer 时调用线程仍先同步 pickle
每一帧，之后 sender 才能丢旧包。不能在不定义 ndarray ownership 的情况下简单把 pickle 延后，因为训练
线程通常会原地改写下一帧数组。长期可选项是 owned/double-buffer frame、shared memory 或显式 copy API。

## 对另一份 code review 的评价

已有 `REPORT-code-review` 在 API compatibility 和静态 correctness 上质量较高：正确识别了 MuJoCo
`enable_*` 命名、RPC camera 180°、WGPU release、正交逆矩阵、remote timeout/connection、texture sRGB、
Euler 文档和 RenderBackend contract 等问题；多数已通过独立动态复现并修复。

需要批判性收窄的部分：

- B6 不应描述为简单“条件反转”。`srgb=False` 表示输入已经线性，应使用 unorm 且不 CPU decode；
  `srgb=True` 的 RGB/RGBA 可用硬件 sRGB format，只有缺少 sRGB format 的通道布局才需 CPU fallback。
- B8 的在线清零不是错误：零内参明确表示切回 fovy。真正错误是 editable spec 没同步清零，下一次 compile
  又恢复旧内参。
- Euler helper 与 Inspector 往返自洽，确定错误是 docstring；直接改乘法顺序会破坏 UI/快照语义。
- 报告称 hot path buffer reuse 较完整，但同时又发现 WGPU staging 每帧分配，表述内部不一致。
- 最大遗漏是性能：没有发现相机 O(N²)、camera/light hierarchy scans、scalar debug traffic、百万 BVH eager
  build、large ImGui lists、full instance upload、remote no-client pickle、透明 draw calls 和 shadow multiplier。

因此该报告适合作为静态 API/bug checklist，但不能单独支撑“大场景和 UI 帧率已经成熟”的结论；它也明确
没有运行 GPU/交互验证，这个限制应保留在最终置信度中。

## 当前收口与远期触发条件

当前应先完成：

1. 在可用 GPU/EGL 环境完成现有 Forge/WGPU gallery 验收，不为假设规模新增渲染架构。
2. 讨论结构编辑的后台 Session job 与“一次 compile”的 spec transaction；确认交互和撤销语义后再实现。
3. 在可用 GPU/EGL 环境目视验收 link DOF gizmo 和 Plane outline；CPU/MuJoCo 交互与约束写回已经完成。
4. 只有在普通编辑场景实际复现文本过载时，再为 label/contact/tendon text 增加 selected-only、viewport
   culling 或预算。

以下工作仅在真实规模、profile 和 GPU 数据证明需要时启动：拆分 static/dynamic instance buffers、instance
frustum/LOD 或 indirect culling、remote cadence/backpressure、跨后端共享 light scheduling，以及专门的
4096-instance GPU benchmark。
