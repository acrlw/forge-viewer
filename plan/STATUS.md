# 未完成项统计

更新日期：2026-08-25

P0、P1、Forge OpenGL 后端和 wgpu Metal/Vulkan 后端已经达到当前验收门槛。按
[ROADMAP.md](ROADMAP.md) 的验收条目统计，后续共有 24 项：P2 编辑器与生产化 12 项、
wgpu 上游改进 3 项、P3 延后能力 9 项。

## P2：编辑器与生产化，12 项

### 编辑器交互，完成

场景文件、Entity 生命周期、undo/redo、authored overlay、运行时 MJCF/URDF 组合、模型根
transform、资源目录、缺失资源重定位和批量路径修复、MjSpec 空间拓扑编辑、完整 MJCF
source 编辑、Camera/Light helper、选中相机预览、四类模型级结构化组件编辑和大型组合
场景性能基线已经完成。编辑器相机与模型相机状态独立，Settings 使用居中模态面板，界面
支持持久化的英文与简体中文切换以及 CJK 字体回退。相机预览支持固定视角与位置；Camera
和 Light 的 Gizmo 在仿真运行时默认锁定。MJCF 导出复制文件资源、写入相对路径、重新编译
并验证移动后的完整目录。模型根 transform 在拖动结束时只编译一次，无变化的 transform
与组件 Apply 跳过重编译。OpenGL 与 wgpu 共用同一套交互、公开接口和示例。

默认基线为 8 个模型、每个 64 bodies：添加模型中位数 16.00 ms，提交模型 transform
25.06 ms，添加组件 35.54 ms，更新组件 27.65 ms。结果写入
`output/editor-performance.json`。

### 真实第二物理后端，5 项

1. 选择具有稳定 Python binding 的物理引擎。
2. 发布稳定的 scene source 和 scene frame。
3. 实现 pause、step、reset、pose write-back 和 perturbation。
4. 接入 contact 与 debug draw。
5. 通过 adapter conformance 和独立可视化验收。

ToyPhysics 用于协议与 UI 验证。真实第二物理后端按当前项目优先级暂缓。

### 平台与发布，5 项

1. 验证 OpenGL 3.3 与 macOS OpenGL 4.1。
2. 验证 Windows D3D12 的 wgpu 安装、交互窗口和视觉回归。
3. 为支持的平台建立 GPU smoke capture。
4. 验证 wheel 构建、clean environment 安装和启动。
5. 发布平台兼容矩阵与性能基线。

### 稳定性与规模，2 项

10,000 帧运行、Viewer 缓冲复用、256-body 模型 20 次加载循环和三个命名相机 Renderer
交错渲染已经通过。稳定帧 RSS 增长为 0，Python 跟踪内存增长 18,275 bytes；模型生命周期
RSS 增长 2,260,992 bytes，低于 8 MiB 门槛。剩余：

1. 验证 CLI/RPC 长连接、超时和错误恢复。
2. 建立录制与快照格式兼容性测试。

## wgpu 上游改进，3 项

GPU timestamp query 和运行时 1×/4× MSAA 调整已经完成。剩余项目当前都需要
wgpu-py 上游提供或修复公开 API：

1. 使用 wgpu-py 的原生 present-mode 选择。
2. 使用 wgpu-py 的公开 surface release API。
3. 在上游兼容后移除 imgui 1.92 适配层。

这些项目属于后端能力增强。当前 Metal/Vulkan 的 Renderer API、Viewer、render flags、
debug views、阴影、反射、outline、tendon、debug draw 和 gizmo 已通过回归测试。

## P3：延后能力，9 项

- SDF iteration visualization：3 项
- Live View 增强：4 项
- PBR 与原生 renderer core 评估：2 项

## 当前验证基线

| 范围 | 结果 |
|---|---:|
| CPU 与静态检查 | Fast 522 passed；Integration 42 passed |
| MuJoCo physics | 211 passed，715 deselected |
| Forge GPU | 216 passed，12 个后端专用测试 skipped |
| wgpu GPU | 175 passed，7 skipped |
| Renderer API | 每个后端 6 个 CPU 合约；wgpu 11 个 GPU 测试 |
| 反向回归 | 50/50 mutation gates |
| 源码任务标记 | 0 个 TODO、FIXME 或 HACK |

MuJoCo 严格可视化审计、deformables adapter conformance、便携 MJCF round trip、严格文档
构建和示例程序均通过。独立 Forge 与 wgpu GPU 回归均通过。

wgpu 的 7 个 skip 覆盖 Forge 内部状态、CPU pass timing 表和 GL error state 等后端实现
细节。GPU pass timing 已由 wgpu timestamp query 独立覆盖；这些 skip 不对应缺失的公开渲染
功能。
