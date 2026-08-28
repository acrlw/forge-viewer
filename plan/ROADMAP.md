# 路线图

## 项目目标

forge-viewer 为仿真、机器人和 3D 工具提供统一的查看与调试环境：

- 通过 Forge 渲染 MuJoCo 模型、程序化场景、远程帧和快照
- 提供兼容常用 `mujoco.Renderer` 工作流的离屏渲染接口
- 统一选择、Gizmo、扰动、调试绘制、捕获、CLI 和 RPC
- 保持渲染与物理后端解耦，支持自定义物理引擎和独立 3D 工具
- 用 CPU、GPU、图像、适配器和性能测试建立发布门槛

## 当前状态

2026-08-26，P0、P1 和 wgpu 的 Metal/Vulkan 集成已完成。当前未完成项统计见
[STATUS.md](STATUS.md)。SDF iteration visualization 排入 P3。

| 范围 | 状态 | 验收结果 |
|---|---|---|
| P0 `mujoco.Renderer` 兼容 | 完成 | RGB、米制 depth、segmentation、相机、`MjvOption`、多实例和 200 次生命周期循环通过 |
| P1 MuJoCo 可视化语义 | 完成 | 严格审计全部为精确对齐或 Forge 等价实现；SDF 标记为 P3 |
| P1 相机与场景状态 | 完成 | 命名书签、物理状态、选择、可视化选项和 Forge 覆盖值可保存与恢复 |
| P1 CLI 与 RPC | 完成 | typed commands、本机 AF_UNIX 服务、版本、超时、错误和三种捕获模式通过 |
| P1 渲染正确性 | 完成 | `texuniform`、透明排序、100 灯光和 8 个本地阴影槽位通过 |
| P1 图像门槛 | 完成 | parity、golden、material parity 和完整 GPU 回归通过 |
| wgpu 渲染后端 | 完成 | macOS Metal、Linux Vulkan、Renderer API 和交互式 Viewer 通过 |
| P2 编辑器与生产化 | 进行中 | 编辑器与稳定性规模门槛完成；第二物理后端和平台发布暂缓 |

## 验收基线

| 项目 | 结果 |
|---|---|
| 核心质量 | Fast 534 passed；Integration 44 passed |
| Forge GPU 回归 | `make gpu`：216 passed，12 个后端专用测试 skipped |
| wgpu GPU 回归 | `make gpu-wgpu`：175 passed，7 skipped |
| MuJoCo physics | 217 passed，727 deselected；严格审计与 conformance 通过 |
| Renderer API | 每个后端 6 个 CPU 合约；wgpu 11 个真实 GPU 测试；200 次构造销毁 |
| Renderer RGB | 对 MuJoCo 参考图 MAE 1.4295 |
| Renderer depth | 误差 p95 0.00037 m |
| Renderer segmentation | 像素一致率 0.999938 |
| MuJoCo parity | 5 个视角平均 edge IoU 0.247，平均 luma error 17.7，28/29 检查通过 |
| Golden images | 6/6 通过 |
| MuJoCo 审计 | 严格模式通过；SDF 记录为延后项 |
| 官方模型语料 | Downloads 81/81、Projects 77/77；Forge 与 wgpu 均通过 8 视角、segmentation 和动态步进 |
| 反向回归 | 50/50 mutation gates |

验收产物统一写入 `output/`。

## 已完成里程碑

| 里程碑 | 内容 |
|---|---|
| M0 项目骨架 | 包结构、Make 入口、测试体系、Viewer 和命令行入口 |
| M1 Forge 首帧 | 相机、网格上传、不透明渲染、窗口缩放和 present |
| M2 材质与纹理 | MuJoCo 材质、纹理、primitive `texuniform` 和透明排序 |
| M3 Debug Draw | GPU 调试图元、远程桥接、大批量线段和稳定性测试 |
| M4 选择与诊断 | R32UI picking、抗锯齿 outline、debug views、pass timing 和 Stats UI |
| M5 光照与环境 | CSM、区域光、本地光、反射、fog、MuJoCo horizon haze 和阴影调度 |
| M6 捕获与多视图 | 截图、视频、命名相机、正交相机、Live View 基础能力和快照回放 |
| M7 适配器验证 | MuJoCo、静态场景、ToyPhysics、远程源和适配器 conformance |

## P0：`mujoco.Renderer` 兼容接口

P0 已完成，入口为：

```bash
make renderer-api
```

### 公开 API

- 顶层导出 `forge_viewer.Renderer`
- 兼容构造参数、公开属性、context manager 和幂等 `close()`
- 支持 `update_scene(data, camera=-1, scene_option=None)`
- 支持 free、fixed、named 和 `MjvCamera` 相机
- 支持调用方提供的 `out` 数组
- 支持多个尺寸不同的 Renderer 并存
- Linux 默认 EGL，桌面环境使用隐藏 GLFW context

### 输出模式

- RGB：`(height, width, 3)`、`uint8`
- depth：`(height, width)`、`float32`、米制相机距离
- segmentation：`(height, width, 2)`、`int32`、`(object ID, object type)`
- 背景 segmentation：`(-1, -1)`
- depth 与 segmentation 模式可重复切换

### MuJoCo 场景语义

- `MjModel`、`MjData` 和 `max_geom`
- geom、site、tendon、actuator、flex、skin、contact 和动态 mesh
- visual groups、render flags、visualization flags、labels、frames 和 BVH depth
- keyframe、mocap、相机、灯光和动态材质状态
- 相机、模型匹配、输出尺寸和关闭后调用的错误语义

## P1：MuJoCo 语义与高频工作流

### P1.1 MuJoCo 可视化语义

以下内容已通过严格审计和模型语料回归：

- tendon、actuator、activation、contact、constraint 和 island
- body、joint、geom、site、camera、light、contact labels 与 frames
- inertia、scaled inertia、center of mass、rangefinder 和 solver diagnostics
- flex、skin、BVH、convex hull、textures、fog、haze、skybox 和 reflection
- visual groups、render flags、visualization flags 和 selection identity
- translation perturbation、rotation perturbation 和视觉反馈
- keyframe 的 qpos、qvel、act、ctrl、mocap 和 time 状态

验收入口：

```bash
make mujoco-audit
make mujoco-visuals
make mujoco-debug
make mujoco-overlays
make mujoco-model-suite
make adapter-conformance ADAPTER=mujoco CONFORMANCE_ASSET=deformables
```

模型语料回归覆盖两个官方 `model/` 目录，并修复了独立 flex UV 索引集和 MuJoCo cube
texture 六面布局。MuJoCo 主仓库的 plugin 与 SDF 示例均可编译、适配和渲染。

### P1.2 相机书签与场景快照

- 相机来源、eye、target、up、轨道参数、投影、near、far 和 aspect
- qpos、qvel、act、ctrl、time 和 mocap
- keyframe、选择、visual groups 和 render flags
- 灯光、环境和材质覆盖值
- 命名、覆盖、删除、复制和恢复
- 稳定 JSON 格式和 `output/snapshots/` 目录

验收入口：

```bash
make camera-state
make scene-snapshot
```

### P1.3 CLI 与 RPC

- load、reload、pause、resume、step 和 reset
- set keyframe、set qpos 和 get state
- set camera 和 load camera bookmark
- set visual group、render flag 和 visualization flag
- RGB、depth 和 segmentation capture
- list、select 和 inspect object
- 请求 ID、协议版本、错误结果和超时
- 人类可读输出与 JSON 输出

验收入口：

```bash
make cli
make rpc
```

### P1.4 渲染正确性与图像门槛

- MuJoCo primitive `texuniform` 三组面轴映射
- 透明实例按相机深度稳定排序
- 100 个活动灯光与 8 个本地阴影槽位确定性调度
- MuJoCo horizon haze 的相机空间截锥几何
- tendon、透明层叠、deformable、height field 和密集模型基线
- RGB、depth、segmentation、material 和完整场景图像门槛

验收入口：

```bash
make material-parity
make shadow-scheduling
make parity
make golden
make gpu
```

## P2：生产化与后端扩展

P2 按以下顺序执行：

### P2.1 编辑器交互

已完成：

- `.forge.json` 的 New、Open、Save、Save As、文件拖放和未保存提示
- primitive、light 与 camera 的创建、复制、重命名和删除
- Entity 菜单、Hierarchy 上下文菜单与快捷键
- 通用 undo/redo、连续 Gizmo 与 Inspector 编辑事务、保存点脏状态
- authored overlay 在 scene source 重建后继续生效
- 后端中立的模型组合契约
- MuJoCo `MjSpec` 运行时 MJCF/URDF 添加、移除、命名空间和状态迁移
- File 菜单、多文件拖放、Hierarchy 模型分组和移除入口
- OpenGL 与 wgpu 的静态编辑回归
- Forge 组合文档、文档相对路径、资源目录与缺失资源诊断
- 缺失资源交互式重定位和批量路径修复
- MJCF/URDF 模型根 transform 编辑和文档恢复
- MjSpec body、geom、joint、site、camera、light 创建、删除、重命名和局部 transform 编辑
- 经 MjSpec 校验的完整 MJCF source 编辑与运行时拓扑重建
- Camera 与 Light 场景 helper、Inspector 编辑和选中相机实时预览
- 模型相机与编辑器相机独立切换和显式返回入口
- 选中相机预览支持固定视角或锁定实体跟随，仿真运行时 Camera/Light Gizmo 默认锁定和
  Inspector 解锁
- 居中模态 Settings、英文与简体中文切换和 CJK 字体回退
- OpenGL 与 wgpu 的多 viewport texture、相机预览和场景 helper 对齐
- actuator、tendon、sensor、equality 的模型级结构化属性面板、引用选择、MjSpec 校验与文档恢复
- fixed body/site transform、常用 primitive 尺寸与 joint axis/range/damping/stiffness 结构化编辑
- geom friction/contact dimension/collision masks/priority/margin/gap/solver mix 结构化编辑
- model-local material 创建、复制、改绑与 PNG 2D texture 导入
- joint gizmo 的多 joint 选择、range visualization、绝对/相对精确输入和 deg/rad 偏好持久化
- topology batch 的批内引用、稳定选择恢复与结构刷新 O(B²)/O(E×M) 扫描消除
- 模型根 transform 拖动提交合并、无变化编辑快速路径和大型组合场景编辑性能基线
- 便携 MJCF 导出、资源相对路径、移动后重新编译和不可表示语义诊断
- 分层测试说明、生成式 API 参考和基础到进阶的可运行示例

验收入口：

```bash
make editor
make settings
make editor-files
make entity-edit
make undo-redo
make model-composition
make workspace-edit
make editor-performance
make primitive-authoring BACKEND=wgpu
make material-authoring BACKEND=wgpu
make contact-authoring BACKEND=wgpu
make body-authoring BACKEND=wgpu
make resource-authoring BACKEND=wgpu
make joint-site-authoring BACKEND=wgpu
make model-component-authoring BACKEND=wgpu
make keyframe-authoring BACKEND=wgpu
make model-settings-authoring BACKEND=wgpu
make batch-editing BACKEND=wgpu
make joint-gizmo BACKEND=wgpu
make scene-entities BACKEND=forge
make scene-entities BACKEND=wgpu
```

结构化 Inspector 继续按实际工作流扩展。mesh/hfield metadata、非插件 component catalog、
keyframe、contact pair/exclude、default class 和 option/solver 已有专用 UI。剩余范围主要是
flex/skin/deformable authoring、bulk asset payload，以及
通用 pose/control/light/material 多选批量编辑。MJCF source popup 编辑 MjSpec 规范化文本，不保留
原始 include 组织、注释或格式。

### P2.2 真实第二物理后端

当前项目优先级暂缓。恢复后先重新评估 Newton 等候选的 Python binding，再执行：

- 选择维护活跃且具有稳定 Python binding 的物理引擎
- 发布稳定 scene source 和动态 scene frame
- 实现 pause、step、reset、pose write-back 和 perturbation
- 接入 contact 与 debug draw
- 通过 adapter conformance 和独立可视化场景

### P2.3 平台与发布

- OpenGL 3.3 与 macOS OpenGL 4.1 验证
- Windows D3D12 的 wgpu 安装、交互窗口和视觉回归
- 支持平台的 GPU smoke capture
- wheel 构建、clean environment 安装和启动验证
- 平台性能基线与兼容矩阵

### P2.4 稳定性与规模

已完成：

- 10,000 帧稳定性基线、Viewer 热帧缓冲复用和 Python 内存增长门槛
- 256-body 模型加载、切换、重复销毁与显式资源释放
- 三个离屏 Renderer 的命名相机交错渲染和部分关闭后继续使用
- CLI/RPC 长连接、超时和错误恢复
- 场景快照与录制文件的当前格式验证
- 版本不匹配、损坏头和截断录制的诊断

验收入口：

```bash
make stability BACKEND=wgpu
make rpc-soak
make format-validation
```

### wgpu 运行时改进

已完成 GPU timestamp query 的异步回读，以及运行时 1×/4× MSAA target 与 pipeline
重建。其余项目跟随 wgpu-py 公开 API：原生 present-mode 选择、公开 surface release，
以及上游适配 imgui 1.92 后移除本地兼容层。

## P3：延后项目

### SDF 可视化

- SDF iteration trace 数据入口
- iteration、contact 和收敛状态绘制
- 专用模型、性能预算和验收场景
- 补齐 MuJoCo Warp 的外部资源包后纳入模型语料：Aloha SDF 场景和 collision SDF torus

### 外部模型资源完整性

- MuJoCo Warp Aloha、Panda、Unitree G1、render mug 和 Apollo 场景缺少仓库外资源
- 为外部资源包定义清单、下载入口和缓存目录
- 将完整资源包加入 `mujoco-model-suite` 多视角回归

### Live View 增强

- sequence、drop、latency 和 structure revision 诊断
- replay pause、step、seek 和 timeline scrub
- 协议与录制格式版本
- 更多网络 transport

### 图形管线研究

- metallic-roughness、normal map、HDR environment 和 image-based lighting
- 基于性能数据评估原生 renderer core

## v1 完成条件

- P0 与 P1 验收保持通过
- 编辑器文件、Entity 生命周期、undo/redo 和多模型组合通过
- 真实第二物理后端通过能力协议
- macOS、Linux 和 Windows 完成安装与启动验证
- 支持平台建立性能预算和 GPU smoke capture
- wheel 发布流程和兼容矩阵完成

## 验收命令索引

| 范围 | 命令 |
|---|---|
| P0 完整门槛 | `make p0` |
| P1 完整门槛 | `make p1` |
| 核心质量 | `make check`、`make reverse` |
| Renderer | `make renderer-api` |
| wgpu Renderer | `make renderer-api-wgpu`、`make gpu-wgpu` |
| MuJoCo 语义 | `make mujoco-audit`、`make mujoco-model-suite`、`make adapter-conformance ADAPTER=mujoco CONFORMANCE_ASSET=deformables` |
| 相机与快照 | `make camera-state`、`make scene-snapshot` |
| CLI 与 RPC | `make cli`、`make rpc` |
| 材质与阴影 | `make material-parity`、`make shadow-scheduling` |
| 图像门槛 | `make parity`、`make golden`、`make gpu` |
| 交互工具 | `make outline`、`make gizmo-gallery`、`make perturb` |
| 光照环境 | `make lighting`、`make image-light`、`make many-lights` |
| 捕获 | `make capture`、`make record` |
| 后端协议 | `make adapter-conformance`、`make toy-physics` |
