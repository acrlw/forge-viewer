# 当前状态

更新日期：2026-09-02

Mojive 已具备可用的 MuJoCo 查看、编辑、离屏渲染、远程查看和录制工作流。OpenGL 与 wgpu
共用 scene contracts、Session、UI 和交互层；差异限制通过 backend capabilities 报告。项目仍是
0.1.x 开发阶段，当前重点是编辑器完整性、可复现性能基线和第二个真实物理适配器，而不是继续扩张基础
viewer 功能。

## 已支持的基线

### 查看、渲染与诊断

- OpenGL 3.3 core 和 wgpu Metal/Vulkan 交互式 viewer；
- `mujoco.Renderer` 风格的 RGB、米制 depth、segmentation、free/fixed/named camera API；
- lighting、shadow、reflection、transparency、skybox/IBL、fog/haze、outline、debug views；
- MuJoCo geom/site/tendon/contact/actuator/constraint/BVH/deformable 等可视化；
- 选择、2D/3D transform gizmo、joint gizmo、物理 perturbation、debug draw 和文本；
- PNG、MP4、keyframe video、scene snapshot 和 `.fvs` remote recording。

### 编辑器

- `.mojive.json` 工作区的新建、打开、保存、资源目录和缺失模型修复；
- 多 MJCF/URDF 模型组合、模型根 transform、portable MJCF export；
- Mojive geometry、material、camera、light、environment entity authoring；
- MjSpec body/geom/joint/site/camera/light topology 和结构化属性编辑；
- contact pair/exclude、actuator、sensor、tendon、equality 和 model-local asset 工作流；
- Undo/Redo、多选拓扑删除、model-local keyframe Dope Sheet 和 transient state take；
- camera/light runtime helper、默认关闭的 camera preview、可停靠非模态 Settings、英文/简体中文。

### 集成

- `Scene` 程序化场景和 `SceneAdapterBase` 自定义适配器协议；
- MuJoCo、static、toy 和 remote adapters；
- live publisher/attach、snapshot replay、typed remote authoring；
- 本机 AF_UNIX RPC、持久连接、并发客户端、超时重连和 RGB/depth/segmentation capture。

## 当前缺口

### 编辑器结构化覆盖

以下内容能被加载、渲染，并尽量在 MjSpec round trip 中保留，但没有与常用属性同等级的结构化
Inspector 工作流：

- flex、skin 和其他 deformable authoring；
- compiler/option/visual、default class、custom data、plugin-defined components；
- bulk mesh/height-field payload 和不常见的 schema 组合；
- 面向 pose、control、light、material 的通用多选批量编辑；
- 保留 include 组织、注释和原始格式的 source editor。

**Edit MJCF Source...** 编辑的是 MjSpec 规范化 XML，不是 source-preserving text editor。需要保留
include、注释或格式时，继续编辑原始外部文件。

### 第二物理适配器

`toy` 用于协议和 UI conformance，不代表第二个生产物理引擎。候选尚未确定。Box3D 的 3D 刚体、
关节、碰撞查询、事件和 MIT/C17 core 与 adapter 目标匹配，但当前官方 v0.1.0 明确属于 alpha，且没有
官方 Python binding；适合先做独立 C-ABI spike，不适合现在承诺为生产适配器。Newton 保留为候选，
选择前统一比较 binding 所有权、模型输入、调试数据、平台和 API 稳定性。

### RL 批量感知渲染

当前 `Renderer` 可以用 `render_async()` 正确地流水执行少量 camera/state，但仍然是单 scene、单
camera、单 render target 的逐 view 路径；`create_peer()` 也只共享 graphics device/context，并不共享
mesh、texture 和 pass resources。它能满足相机预览、小规模多相机 capture 和 CPU dataset 生成，不能
作为几十至几百个 vectorized world 的高吞吐 RL sensor。

下一阶段先建立 shared immutable render resources 和 flat selected-view batch，再实现 texture-array
输出、一次 encoder/submission 和整批 readback。多 world topology cohort 与真实 GPU tensor interop
在这个所有权边界之后推进。设计与 Newton `SensorTiledCamera` 的实现对照见
[`docs/BATCH_RENDERING.md`](../docs/BATCH_RENDERING.md)。

### 平台与发布

- Windows D3D12 wgpu 的安装、窗口和视觉回归仍需真实机器验证；
- 当前是个人开发项目，不把 wheel 构建和发布流水线列为近期工作；正式发布前再恢复；
- 平台兼容矩阵仍需实机验证；Renderer 性能基线通过 `make renderer-benchmark` 维护。

### 远程边界

`.fvs` 和 live snapshot transport 使用 pickle，只适合可信本机或可信局域网。接收不可信网络数据
前必须迁移到非可执行 schema；固定 auth key 不能把 pickle 变成安全边界。

### 其他规模触发项

大型 mesh BVH 诊断、全量 instance buffer 上传、frustum culling/LOD/indirect draw 和大量 point-light
shadow face 重绘目前没有达到必须重构的实测门槛。由真实 profile 或 parity drift 触发优化，不为了
形式统一预先拆分稳定路径。

## wgpu 上游依赖

当前 Metal/Vulkan 的 Renderer API、viewer、render flags、debug views、shadow、reflection、outline、
tendon、debug draw、gizmo、GPU timestamp 和 1×/4× MSAA 均有实现。以下清理仍依赖 wgpu-py：

1. 公开的 present-mode 选择；
2. 公开的 surface release API；
3. 上游兼容后移除 imgui 1.92 适配层。

## 延后研究

- SDF iteration visualization；
- 外部模型资源完整性和供应链策略；
- 面向非可信网络的 Live View；
- PBR 与原生 renderer core 的进一步评估。

## 验证基线

不在本文冻结容易过期的 passed/skipped 数字。代码库中的 markers、Makefile 和 CI 输出是数量事实
来源；当前验收入口为：

```bash
make check
make docs-check
make renderer-api
make gpu
make gpu-wgpu
make mujoco-physics
make mujoco-audit
make adapter-conformance ADAPTER=mujoco CONFORMANCE_ASSET=deformables
make reverse
```

用户可见改动还应运行最小相关 visual target 并检查 `output/` 产物。完整映射见
[`docs/guides/testing.md`](../docs/guides/testing.md)，所有可用目标见 `make help`。
