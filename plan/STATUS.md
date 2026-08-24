# 未完成项统计

更新日期：2026-08-23

P0、P1、Forge OpenGL 后端和 wgpu Metal/Vulkan 后端已经达到当前验收门槛。按
[ROADMAP.md](ROADMAP.md) 的验收条目统计，后续共有 33 项：P2 编辑器与生产化 19 项、
wgpu 上游与运行时改进 5 项、P3 延后能力 9 项。

## P2：编辑器与生产化，19 项

### 编辑器交互，3 项

场景文件、Entity 生命周期、undo/redo、authored overlay、运行时 MJCF/URDF 组合、模型根
transform、资源目录、MjSpec 空间拓扑编辑、完整 MJCF source 编辑、Camera/Light helper 和
选中相机预览已经完成。OpenGL 与 wgpu 共用同一套交互和文档接口。

1. 增加缺失资源交互式重定位和批量路径修复。
2. 增加 actuator、tendon、sensor、equality 的结构化属性面板。
3. 建立大型组合场景的增量重编译和编辑性能基线。

### 真实第二物理后端，5 项

1. 选择具有稳定 Python binding 的物理引擎。
2. 发布稳定的 scene source 和 scene frame。
3. 实现 pause、step、reset、pose write-back 和 perturbation。
4. 接入 contact 与 debug draw。
5. 通过 adapter conformance 和独立可视化验收。

ToyPhysics 用于协议与 UI 验证。Newton 适配安排在 MuJoCo 工作流稳定之后。

### 平台与发布，6 项

1. 建立 macOS、Linux 和 Windows CPU CI。
2. 验证 OpenGL 3.3 与 macOS OpenGL 4.1。
3. 验证 Windows D3D12 的 wgpu 安装、交互窗口和视觉回归。
4. 为支持的平台建立 GPU smoke capture。
5. 验证 wheel 构建、clean environment 安装和启动。
6. 发布平台兼容矩阵与性能基线。

### 稳定性与规模，5 项

1. 验证长时间仿真和 Viewer 内存稳定性。
2. 验证大模型加载、切换和重复销毁。
3. 验证多相机离屏 Renderer 并发。
4. 验证 CLI/RPC 长连接、超时和错误恢复。
5. 建立录制与快照格式兼容性测试。

## wgpu 改进，5 项

1. 接入稳定的 GPU timestamp query。
2. 支持运行时调整 MSAA sample count。
3. 使用 wgpu-py 的原生 present-mode 选择。
4. 使用 wgpu-py 的公开 surface release API。
5. 在上游兼容后移除 imgui 1.92 适配层。

这些项目属于后端能力增强。当前 Metal/Vulkan 的 Renderer API、Viewer、render flags、
debug views、阴影、反射、outline、tendon、debug draw 和 gizmo 已通过回归测试。

## P3：延后能力，9 项

- SDF iteration visualization：3 项
- Live View 增强：4 项
- PBR 与原生 renderer core 评估：2 项

## 当前验证基线

| 范围 | 结果 |
|---|---:|
| CPU 与静态检查 | 555 passed，328 deselected |
| MuJoCo physics | 186 passed，697 deselected |
| Forge GPU | 209 passed |
| wgpu GPU | 166 passed，7 skipped |
| Renderer API | 每个后端 6 个 CPU 合约、10 个 GPU 测试 |
| 源码任务标记 | 0 个 TODO、FIXME 或 HACK |

wgpu 的 7 个 skip 覆盖 Forge 内部状态、GPU pass timing 和 GL error state 等后端实现细节。
它们不对应缺失的公开渲染功能。
