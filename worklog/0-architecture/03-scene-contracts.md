# 03 场景、坐标与身份契约

这章用于新增共享字段、处理矩阵/图像方向，以及避免把稳定身份误当成数组位置。

## 三层 scene 表示

| 表示 | 位置 | 生命周期 | 主要消费者 |
|---|---|---|---|
| `Scene` | `scene.py` | 可变作者模型 | Static/Workspace、教程、scene I/O |
| `SceneSource` | `adapters/base.py` | revision 间稳定 | Session、builder、远程结构包 |
| `SceneFrame` | `adapters/base.py` | 每帧动态 | Session、renderer、远程 latest frame |
| `RenderScene` | `render/scene.py` | backend 内 GPU staging | Forge/WGPU pass |

- **已核对**：稳定结构变化必须提升 adapter `structure_revision`；动态 pose 不应触发重建。
- **建议**：新增字段先判断是 stable 还是 dynamic，再决定 source/frame，避免双份真相。

## 坐标与矩阵

- Python 矩阵 row-major；translation 位于 `matrix[:3, 3]`。
- 世界坐标 Z-up；`math3d.to_gl()` 只在 OpenGL upload boundary 转置。
- `CameraView.fov_y` 用弧度；MuJoCo spec 的部分字段用度数，adapter 是换算边界。
- render target 的 `flip_y` 元数据决定 UI/capture 方向，不应在调用者猜 backend。
- **已核对**：`euler_xyz_to_mat3`/逆函数采用 extrinsic XYZ 且相互自洽；不要把它直接描述成 MuJoCo XML
  默认 Euler 语义。

## 身份与 slot

| 名称 | 含义 | 可否稀疏 |
|---|---|---|
| `object_id` | 稳定选择身份，0 表示无选择 | 可以 |
| `node_id` | 层级图身份，parent/children 引用 ID | 可以，必须唯一 |
| `camera_id` | adapter 稳定相机身份 | 可以 |
| `keyframe_id` / `constraint_id` | adapter 稳定控制身份 | 可以 |
| `camera_index` / `light_index` | 当前 source/frame 数组 slot | 不应跨 revision 保存 |
| `body_index` | physics array lookup | 由 adapter 模型约定 |

- **已核对**：conformance 要求 ID 唯一，但不要求从 0 连续。
- **已核对**：从 node 的 camera slot 调 adapter 时，必须先经 `CameraInfo` 映射到 camera ID。
- **建议**：任何保存、锁定、命令结果或远程传输都保存 ID；仅一帧内数组访问保存 index。

## 可变数组约定

`CameraView`、`Light`、`Material` 等 frozen dataclass 内含可变 ndarray。

- **观察到**：这里的 frozen 只禁止字段重新绑定，不保证深不可变；热路径依赖少分配。
- **建议**：跨线程、bookmark、snapshot、缓存所有权边界做显式 copy；不要在每帧路径无差别深拷贝。
