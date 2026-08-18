# 路线图

## 项目目标

forge-viewer 的核心目标包含四部分：

- 提供完整、流畅的 MuJoCo 交互式 Viewer，覆盖模型查看、仿真控制、扰动、编辑、
  可视化选项、诊断、截图和视频
- 提供与常用 `mujoco.Renderer` 工作流兼容的公开接口，覆盖 RGB、深度、分割、
  相机选择和资源生命周期
- 让 MuJoCo、自定义物理引擎、Forge 场景、远程发布和快照回放共享同一套渲染、
  命令、选择、调试绘制和 UI 基础设施
- 建立可重复的功能、图像、性能、适配器、平台和发布验收门槛

## 当前进度

2026-08-18 工程评估：

| 目标 | 完成度 | 当前边界 |
|---|---:|---|
| MuJoCo 交互式 Viewer | 90% | 模型与工具覆盖较完整，部分可视化语义和图像一致性仍需收口 |
| 脚本与离屏渲染替代 | 65% | 已有截图和录制链路，缺少 `mujoco.Renderer` 兼容公开接口 |
| 后端无关工具 | 80% | 静态场景、ToyPhysics、远程查看和回放已运行，通用工作流仍需扩展 |
| 可发布 v1 | 70% | 缺少跨平台持续集成、安装验证、图像门槛和真实第二物理后端 |
| 总体目标 | 80% | 核心架构和主要交互已建立，剩余工作集中在兼容接口、语义收口和发布质量 |

完成度按用户工作流和验收证据计算。公开接口、回归门槛和平台结果的权重高于功能数量。

## 当前验收基线

| 项目 | 最近结果 |
|---|---|
| CPU 测试 | `make check`：475 passed，256 deselected |
| GPU 测试 | Apple M5 完整渲染测试通过 |
| MuJoCo 适配 | 严格审计和 deformable conformance 通过 |
| 性能 | 601 instances、212,402 triangles、21 draws、4x MSAA，frame CPU 中位数 3.015 ms |
| 参考图对比 | 五个视角平均 edge IoU 0.247，平均 luma error 17.7 |
| Golden images | 当前 4/6 通过，`test_scene` 和 `showcase` 待复核 |

当前参考图的主要几何关系基本一致，Forge 的光照、反射和整体明暗仍存在明显差异。

## 原始里程碑状态

| 里程碑 | 状态 | 已完成内容与剩余工作 |
|---|---|---|
| M0：项目骨架 | 完成 | 包结构、测试、Make 入口和 Viewer 入口 |
| M1：Forge 首帧 | 完成 | 相机、网格上传、不透明渲染、窗口缩放和 present |
| M2：材质与纹理 | 部分完成 | MuJoCo 材质纹理已支持，primitive `texuniform` 和透明重叠排序仍需完善 |
| M3：Debug Draw | 完成 | 12 类图元、AF_UNIX bridge、10k lines GPU 路径和 100 帧稳定性测试 |
| M4：选择与诊断 | 完成 | R32UI picking、outline、debug views、render pass timing 和 Stats UI |
| M5：光照与环境 | 部分完成 | CSM、反射、区域光、本地光、fog 和 haze 已支持，图像门槛和阴影调度仍需完善 |
| M6：捕获与多视图 | 完成 | 任意分辨率截图、视频、命名相机、正交相机和多进程 Live View |
| M7：第二物理后端 | 部分完成 | `ToyPhysicsAdapter` 已验证协议，真实物理引擎适配仍待完成 |

## 执行顺序

后续工作按以下顺序推进：

1. 近期修复：收口 Gizmo 拖拽与 snap 刻度的视觉一致性
2. P0：完成 `mujoco.Renderer` 兼容公开接口
3. P1：收口 MuJoCo 可视化语义与高频工具工作流
4. P1 完成门槛：恢复 parity 和 golden image 验收
5. P2：生产化、真实第二物理后端和跨平台发布
6. P3：SDF、编辑器深化、Live View 增强和新图形管线研究

每项用户可见能力都需要 Make 验收入口。实现阶段先运行最小相关测试，阶段完成时运行对应的完整门槛。

## 近期修复：Gizmo 视觉一致性

- 2D 与 3D position gizmo 共用中心透明遮罩，拖动轴线保持连续几何并由遮罩裁切
- translation snap 主轴和刻度共用坐标轴颜色，刻度按距离渐隐
- translation snap 在当前位置附近保留完整刻度，随后平滑渐隐并限制绘制范围
- rotation snap 刻度直接连接圆环，投影退化时隐藏刻度
- `make gizmo-gallery` 覆盖 position、rotation、拖动、snap、2D 和 3D 结果

完成条件：中心遮罩、刻度接缝、远端渐隐和极端投影均通过图库检查与 GPU 回归。

## P0：兼容 `mujoco.Renderer` 公开接口

P0 是当前最高优先级。目标是在常用离屏渲染脚本中，用 forge-viewer 替换
`mujoco.Renderer`，保持调用结构、输出格式和资源行为一致。

### P0.1 公开 API 与对象状态

从 `forge_viewer` 顶层导出 `Renderer`，对齐当前 MuJoCo 接口：

```python
Renderer(
    model,
    height=240,
    width=320,
    max_geom=10000,
    font_scale=mujoco.mjtFontScale.mjFONTSCALE_150,
)
```

公开成员与行为：

- `model`
- `scene`
- `height`
- `width`
- `update_scene(data, camera=-1, scene_option=None)`
- `render(out=None)`
- `enable_depth_rendering()` / `disable_depth_rendering()`
- `enable_segmentation_rendering()` / `disable_segmentation_rendering()`
- `close()`
- context manager 和析构资源释放

验收要求：

- 构造参数、默认值、属性和异常类型具有兼容测试
- depth 和 segmentation 模式互斥，模式切换可重复执行
- `close()` 可重复调用，关闭后的渲染行为具有测试
- `max_geom` 容量检查与 MuJoCo 的错误语义对齐

### P0.2 `update_scene` 语义

需要覆盖：

- `MjModel` 与对应 `MjData`
- free camera：`camera=-1`
- fixed camera：相机索引和相机名称
- 直接传入 `MjvCamera`
- `MjvOption` 中的 visual groups、render flags 和 visualization flags
- body、geom、site、tendon、actuator、flex、skin、contact 和动态 mesh 更新
- keyframe、mocap、灯光、相机和动态材质状态
- 重复更新同一模型时复用 GPU 资源和帧缓冲

验收要求：

- 同一组 `MjModel`、`MjData`、camera 和 `MjvOption` 同时输入 MuJoCo 与 Forge
- 静态结构只在模型结构变化时重建
- 动态状态每帧正确更新
- 相机名称错误、索引越界和数据模型不匹配具有明确异常

### P0.3 RGB 输出

对齐以下行为：

- 输出 shape 为 `(height, width, 3)`
- 默认 dtype 为 `uint8`
- 图像原点与 `mujoco.Renderer` 一致
- `out` 参数接收匹配 shape 的调用方数组
- 返回值和 `out` 的写入关系与 MuJoCo 一致
- MSAA resolve、颜色空间和读取时机稳定

第一阶段比较几何覆盖、相机投影、图像方向和像素类型；光照与材质阈值在 P1 完成门槛统一收口。

### P0.4 深度输出

对齐以下行为：

- 输出 shape 为 `(height, width)`
- dtype 为 `float32`
- 数值表示相机到可见表面的距离，单位为米
- near、far、透视相机和正交相机具有独立测试
- 背景深度、极近表面和远距离精度具有边界测试
- `out` 参数的 shape 检查和类型转换与 MuJoCo 一致

深度验收采用解析几何场景，包括平面、球、box、已知相机距离和遮挡关系。

### P0.5 分割输出

对齐以下行为：

- 输出 shape 为 `(height, width, 2)`
- dtype 为 `int32`
- 两个通道分别表示 object ID 和 MuJoCo object type
- 背景像素为 `(-1, -1)`
- geom、site、flex、skin 和 visualization geometry 的标识稳定
- 相同对象的不同绘制实例保持正确 ID

Forge 内部 object ID 与 MuJoCo `objid`、`objtype` 的映射集中维护，并覆盖重复名称、匿名对象和动态可视化图元。

### P0.6 上下文与运行环境

需要覆盖：

- Viewer 已打开时创建离屏 Renderer
- 单进程多个 Renderer
- 不同分辨率 Renderer 并存
- 连续创建和销毁
- macOS GLFW 上下文
- Linux EGL 或 OSMesa 路径
- resize 采用重建或固定尺寸策略，并在 API 文档中明确

资源验收包含 GPU 对象释放、上下文切换、200 次构造销毁和长帧序列内存稳定性。

### P0.7 验收入口

计划增加：

```bash
make renderer-api
```

该入口包含：

- API 签名和属性测试
- RGB、depth、segmentation 对照测试
- free、fixed、named、`MjvCamera` 相机测试
- `MjvOption` 测试
- `out` 数组测试
- context manager、close 和多实例测试
- headless smoke test
- 小型视觉对照图库，产物写入 `output/renderer-api/`

P0 完成条件：常用 `mujoco.Renderer` 示例只修改 import 即可运行，输出 shape、dtype、方向、ID、深度单位和生命周期测试全部通过。

## P1：MuJoCo 语义与高频工作流

### P1.1 MuJoCo 可视化语义收口

SDF iteration visualization 进入 P3。其余 MuJoCo render flags 和 visualization flags 按
MuJoCo 的数据语义、开关行为和结果表达逐项对齐。

重点工作：

- actuator activation 的颜色、范围和可见条件
- translation perturbation 与 rotation perturbation 的状态、受力和视觉反馈
- selection 的对象身份、选中状态和 Forge outline 映射
- tendon、actuator、contact、constraint、island、BVH、flex 和 skin 的开关组合
- body、joint、geom、site、camera、light、contact labels 与 frames
- rangefinder、equality、inertia、center of mass 和 solver diagnostics
- keyframe 切换后的 qpos、qvel、act、mocap 和时间状态

覆盖报告统一使用四种状态：

- 精确对齐：数据和结果语义与 MuJoCo 一致
- Forge 等价实现：交互目标一致，采用 Forge 绘制路径
- 部分对齐：已覆盖主要数据，仍有明确缺项
- 延后：进入后续里程碑

验收：

```bash
make mujoco-audit
make mujoco-visuals
make mujoco-debug
make mujoco-overlays
make adapter-conformance ADAPTER=mujoco CONFORMANCE_ASSET=deformables
```

每个“部分对齐”项目都需要回归场景、测试和目标里程碑。内置模型语料以严格模式通过审计。

### P1.2 IK 编辑

IK 作为 MuJoCo 高频模型操作进入 P1：

- 选择末端 body 或 site
- position target 与 rotation target 独立开关
- world frame 与 body frame 操作
- 关节范围、锁定关节和权重
- 求解迭代次数、残差和失败状态显示
- 暂停状态下写回 qpos，并调用 MuJoCo forward 更新场景
- undo 一次完整 IK 拖拽

计划入口：

```bash
make mujoco-ik
```

验收模型至少覆盖机械臂、四足单腿和人体链条。

### P1.3 相机书签与场景快照

相机书签保存：

- camera source
- eye、target、up
- yaw、pitch、distance
- FOV 或正交尺寸
- near、far
- viewport aspect 相关信息

场景快照保存：

- 当前模型标识
- qpos、qvel、act、time 和 mocap
- 当前 keyframe
- 选择对象、可视化选项和 visual groups
- 相机书签或当前相机
- Forge 场景覆盖值

UI 提供命名、覆盖、删除、复制和载入。快照文件写入 `output/snapshots/`，同时提供稳定的可序列化格式。

计划入口：

```bash
make camera-state
make scene-snapshot
```

### P1.4 CLI 与 RPC

CLI 和 RPC 共用 typed commands、`SceneSource`、`SceneFrame` 和 `Session` 命令路由。

第一阶段命令集：

- load、reload、pause、resume、step、reset
- set keyframe、set qpos、get state
- set camera、load camera bookmark
- set visual group、set render flag、set visualization flag
- capture RGB、depth、segmentation
- list objects、select object、inspect object

RPC 第一阶段覆盖本机进程通信、请求 ID、错误结果、超时和版本字段。CLI 输出支持人类可读文本与 JSON。

计划入口：

```bash
make cli
make rpc
```

### P1.5 渲染正确性

- 完整实现 MuJoCo primitive `texuniform` 的三组面轴映射
- 完善透明实例的相机深度排序，评估复杂透明场景的 OIT 需求
- 固定 100 个活动灯光与 8 个本地阴影槽位的确定性选择规则
- 在 Settings 和 Stats 中显示活动与延后阴影投射灯光
- 增加 tendon、透明层叠、deformable、height field 和密集模型场景

计划入口：

```bash
make material-parity
make shadow-scheduling
```

## P1 完成门槛：进入 P2 前恢复视觉验收

视觉门槛放在 P1 功能收口之后执行，并作为进入 P2 的硬门槛。

工作内容：

- 复核 camera、projection、geometry、lighting、reflection、tone 和 texture 差异
- 为每个 parity 视角定义 edge IoU、luma、覆盖区域和关键像素阈值
- 复核 `test_scene` 与 `showcase` 的 golden image 差异
- 补充 RGB、depth、segmentation、material、tendon、deformable 和 dense model 基线
- 为批准的图像变化记录简洁的 baseline 说明

完成条件：

```bash
make parity
make golden
make gpu
```

以上入口从干净 checkout 运行通过，产物统一写入 `output/`。

## P2：生产化与后端扩展

### P2.1 真实第二物理后端

选择 Newton 或另一套维护活跃、跨平台且具有稳定 Python binding 的物理引擎：

- 发布稳定结构和动态帧
- pause、step、reset、position/rotation 写回和 perturbation
- contact 与 debug draw
- 能力声明和 adapter conformance
- 独立可视化验收场景

ToyPhysics 保留为最小协议示例，真实引擎负责验证工程适配能力。

### P2.2 平台与发布

- macOS、Linux、Windows CPU CI
- 真实 OpenGL 3.3 与 macOS OpenGL 4.1 环境验证
- 支持平台的 GPU smoke capture
- clean environment wheel build、install 和启动验证
- 性能基线按平台记录
- generated images、videos、reports 和 captures 统一存放在 `output/`
- 发布检查表和兼容矩阵

### P2.3 稳定性与规模测试

- 长时间仿真与 Viewer 内存稳定性
- 大模型加载、切换和重复销毁
- 多相机离屏 Renderer 并发
- CLI/RPC 长连接与错误恢复
- 录制文件完整性和版本兼容测试

## P3：延后项目

### SDF 可视化

- SDF iteration trace 数据入口
- iteration、contact 和收敛状态绘制
- 专用模型、性能预算和验收场景

### Live View 增强

当前保留 publisher、多个 viewer、typed command、remote authoring 和 snapshot replay 基础能力。
后续增强包括：

- sequence、drop、latency 和 structure revision 诊断
- replay pause、step、seek 和 timeline scrub
- 独立于 Python class layout 的协议与录制格式版本
- 更多网络 transport

### Forge 编辑器深化

- 从 UI 创建、复制和删除 primitive、light 和 camera
- 打开、保存和组合 `.forge.json` 场景
- MJCF、URDF 与 Forge entity 的多模型组合
- 通用 undo/redo 和场景资源管理

### 图形管线研究

- wgpu 渲染后端原型
- metallic-roughness、normal map、HDR environment 和 image-based lighting
- 根据性能分析评估原生 renderer core

## v1 完成条件

- `make check`、`make gpu`、`make golden` 和 `make parity` 全部通过
- Renderer 兼容测试覆盖 RGB、depth、segmentation、camera、`MjvOption` 和 lifecycle
- MuJoCo 严格审计与模型语料 conformance 通过
- P1 中除 SDF 外的 MuJoCo 可视化语义完成收口
- IK、相机书签、场景快照、CLI 和 RPC 具备 Make 验收入口
- 真实第二物理后端通过能力协议
- 每个支持平台通过安装和启动验证
- 性能基线满足对应平台预算
- 所有验收产物写入 `output/`

## 现有验收入口

| 范围 | 命令 |
|---|---|
| 核心质量 | `make check` |
| Renderer 测试 | `make gpu` |
| Golden images | `make golden` |
| MuJoCo 图像对比 | `make parity` |
| 性能 | `make bench` |
| 选择与工具 | `make outline`、`make gizmo-gallery`、`make perturb` |
| 光照 | `make lighting`、`make image-light`、`make many-lights` |
| 捕获 | `make capture`、`make record` |
| 场景基础 | `make empty`、`make canvas`、`make scene-io` |
| Live View 基础 | `make live-view`、`make remote-authoring`、`make snapshot-replay` |
| MuJoCo 覆盖 | `make mujoco-audit`、`make mujoco-visuals`、`make deformables` |
| 适配器协议 | `make adapter-conformance`、`make toy-physics` |

计划中的 Make 入口随对应实现任务进入 Makefile。
