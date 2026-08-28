# 未完成项统计

更新日期：2026-08-28

P0、P1、Forge OpenGL 后端和 wgpu Metal/Vulkan 后端已经达到当前验收门槛。后续固定里程碑
包括真实第二物理后端 5 项、平台与发布 5 项、wgpu 上游改进 3 项和 P3 延后能力 12 项。
编辑器基础工作流已经可用，但结构化 MuJoCo schema 仍有明确的迭代范围；这些工作流型缺口
单独列出，不再混入一个会因交叉依赖而失真的总数。

## P2：编辑器与生产化，10 项

### 编辑器交互，可用基线完成；结构化 authoring 继续迭代

场景文件、Entity 生命周期、undo/redo、authored overlay、运行时 MJCF/URDF 组合、模型根
transform、资源目录、缺失资源重定位和批量路径修复、MjSpec 空间拓扑编辑、完整 MJCF
source 编辑、Camera/Light helper、选中相机预览、四类模型级结构化组件编辑和大型组合
场景性能基线已经完成。固定 body/site transform、常用 primitive 尺寸、joint
axis/range/damping/stiffness、body inertial/mass/gravcomp/mocap/sleep、geom contact/solver/
surface/mass/group/fluid properties、primitive/mesh/hfield 类型切换与导入、model-local material
与 PNG 2D/cube/skybox
texture import 也已有支持 Undo/Redo 的结构化入口。编辑器相机与模型相机状态独立，Settings
使用居中模态面板，界面支持持久化的英文与简体中文切换以及 Noto Sans SC 自动下载与 CJK
字体回退。相机预览支持
固定视角或锁定实体并实时跟随；Camera 和 Light 的 Gizmo 在仿真运行时默认锁定。MJCF 导出
复制文件资源、写入相对路径、重新编译并验证移动后的完整目录。模型根 transform 在拖动结束时
只编译一次，无变化的 transform
与组件 Apply 跳过重编译。OpenGL 与 wgpu 共用同一套交互、公开接口和示例。

joint gizmo 支持 hinge/slide/ball/free、多 joint 选择、limit visualization、绝对/相对精确输入和
deg/rad；选择偏好可以跨会话保存。Topology batch 可在一次 compile 中引用本批新建元素并按稳定
语义身份恢复选择。结构刷新已消除已知的 O(B²) body walk 和 O(E×M) model ownership scan。

尚未结构化的主要范围如下；规范化的 **Edit MJCF Source** 可以表达其中一部分，但不代表已有
同等级 UI，也不保留原始 include 组织、注释或格式：

- flex/skin/deformable authoring 和 bulk asset payload；
- 面向真实多选工作流的通用 pose/control/light/material batch commands。

默认基线为 8 个模型、每个 64 bodies：添加模型中位数 14.81 ms，结构节点构建 3.04 ms，提交模型
transform 24.80 ms，添加组件 28.14 ms，更新组件 27.14 ms。结果写入
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

### 稳定性与规模，完成

10,000 帧运行、Viewer 缓冲复用、256-body 模型 20 次加载循环和三个命名相机 Renderer
交错渲染已经通过。稳定帧 RSS 增长为 0，Python 跟踪内存增长 18,275 bytes；模型生命周期
RSS 增长 2,260,992 bytes，低于 8 MiB 门槛。RPC 支持单连接连续请求、并发客户端、超时后
重连与错误恢复。场景 JSON 快照和 `.fvs` 录制严格验证当前格式，并诊断版本不匹配和截断
录制。正式发布前直接迭代格式，不保留旧格式读取分支。

## wgpu 上游改进，3 项

GPU timestamp query 和运行时 1×/4× MSAA 调整已经完成。剩余项目当前都需要
wgpu-py 上游提供或修复公开 API：

1. 使用 wgpu-py 的原生 present-mode 选择。
2. 使用 wgpu-py 的公开 surface release API。
3. 在上游兼容后移除 imgui 1.92 适配层。

这些项目属于后端能力增强。当前 Metal/Vulkan 的 Renderer API、Viewer、render flags、
debug views、阴影、反射、outline、tendon、debug draw 和 gizmo 已通过回归测试。

## 非阻塞设计债与触发项

以下内容不是当前支持输入下的已复现 bug，也不适合为了形式统一立即重构：

- `Renderer.enable_*_rendering()` / `disable_*_rendering()` 刻意兼容 `mujoco.Renderer`；内部 feature
  switch 使用 `set_flag(flag, bool)`，debug output 使用 `set_debug_view()`，不改成冲突的 bool setter。
- `Scene` 的修改失败仍混用 bool、`KeyError` 与 `None`；需要公开 API 兼容策略后再统一，不能机械改返回值。
- `SceneAdapterBase` 与 `SceneAdapter` Protocol 表面较宽且手工镜像；真实第三方 capability 漂移再次出现时，
  再按 simulation/authoring/composition service 拆分比预先制造大量小接口更稳妥。
- Forge/WGPU 仍有 light scheduling、cascade、tendon/reflection 等 Python 算法或常量的镜像副本；应由实际
  parity drift 驱动共享，不能强求两套 GPU pipeline 源码同构。
- Remote snapshot 使用 pickle，明确只适合可信本机/局域网；若要接收不可信网络或下载文件，必须迁移到
  非可执行 schema，而不是把固定 authkey 当安全边界。
- 大型 mesh BVH 诊断、全 instance buffer 上传、缺少 frustum culling/LOD/indirect draw 和多 point-light
  shadow face 重绘都保留规模触发条件；当前没有 light metadata O(L²)。

## P3：延后能力，12 项

- SDF iteration visualization：3 项
- 外部模型资源完整性：3 项
- Live View 增强：4 项
- PBR 与原生 renderer core 评估：2 项

## 当前验证基线

| 范围 | 结果 |
|---|---:|
| CPU 与静态检查 | Fast 613 passed；Integration 67 passed |
| MuJoCo physics（隔离 GPU） | 216 passed，1 条既有 flex warning |
| Forge GPU | 既有完整基线 216 passed；本机本轮 EGL 初始化不可用，未重复计数 |
| wgpu GPU | 202 passed，7 skipped |
| Renderer API | 每个后端 6 个 CPU 合约；wgpu 11 个 GPU 测试 |
| 反向回归 | 50/50 mutation gates |
| 源码任务标记 | 0 个 TODO、FIXME 或 HACK |

MuJoCo 严格可视化审计、deformables adapter conformance、便携 MJCF round trip、严格文档
构建和示例程序均通过。独立 Forge 与 wgpu GPU 回归均通过。

wgpu 的 7 个 skip 覆盖 Forge 内部状态、CPU pass timing 表和 GL error state 等后端实现
细节。GPU pass timing 已由 wgpu timestamp query 独立覆盖；这些 skip 不对应缺失的公开渲染
功能。
