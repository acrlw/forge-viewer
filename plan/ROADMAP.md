# 路线图

更新日期：2026-09-02

## 项目目标

Mojive 为机器人、仿真和 3D 工具提供一套统一的查看、编辑、渲染与调试环境：

- 物理引擎通过 backend-neutral scene source/frame 和 typed commands 接入；
- OpenGL 与 wgpu 消费同一场景语义并提供交互 viewer 与离屏渲染；
- editor 能组合真实模型、编辑常见 MuJoCo 结构并导出可移动的 MJCF；
- remote、recording 和 RPC 复用正常 Session/adapter 路径；
- 用户可见功能有可复现的 visual target，公共能力有 CPU/GPU/physics gates。

当前支持基线和已知限制见 [STATUS.md](STATUS.md)。已经完成的 wgpu 和 UI 分阶段方案保留为
历史实现记录，不再列入未完成里程碑。

## R1：编辑器完整性

目标是在不把 Inspector 变成完整 XML IDE 的前提下，完成高频模型 authoring 工作流。

### R1.1 常用结构化属性

- 依据真实用户模型和 `mj_printSchema()` coverage 报告补充高频缺口；
- 保持 schema-driven reference choices、模型局部命名、验证和 Undo/Redo；
- 能在不改变 derived constants 时走窄更新，拓扑变化才 compile/rebuild；
- 明确区分 structured UI、normalized source escape hatch 和 source-owned declarations。

验收：针对新增属性的 focused physics tests、workspace round trip、`make mujoco-audit` 和对应
authoring visual target。

### R1.2 多选与批量工作流

- 为 pose、control、light、material 等真实批量操作定义 typed batch commands；
- 一次用户操作只产生一个 history transaction 和必要的最少 rebuild；
- 部分不兼容选择必须报告原因，不能静默跳过。

验收：CPU command/history tests、physics state migration tests、`make batch-editing`。

### R1.3 Deformable authoring 边界

- 根据实际需求确定 flex/skin 的最小结构化 surface；
- bulk payload 保持文件资源或 source-owned，避免在 Inspector 中复制大型数组；
- 明确 plugin 和 attach/include 展开后的可编辑/不可逆边界。

验收：`make deformables`、portable MJCF round trip 和 model corpus。

## R2：平台验证（正式发布前）

### R2.1 平台矩阵

- macOS Apple Silicon：OpenGL 4.1 core、Metal wgpu；
- Linux：X11/GLX、Wayland/EGL、offscreen EGL、Vulkan wgpu；
- Windows：OpenGL 3.3 core、D3D12 wgpu。

每个平台记录 Python、driver、window path、offscreen path、HiDPI、capture 和已知限制。没有真实
机器结果的平台不能标记为 validated。

### R2.2 分发（延后）

- 当前个人开发阶段不构建 wheel；确定正式发布后再恢复本节；
- 届时构建 wheel 并在 clean environment 安装 `core`、`mujoco`、`wgpu` 组合；
- 验证 bundled assets、GLSL/WGSL resources、CLI entry point 和首次启动；
- 建立最小 GPU smoke capture，并将日志和产物归档；
- 发布版本兼容策略和格式迁移策略。

验收：clean-environment install script、`mojive assets --quick`、`mojive doctor test_scene`、
一张 deterministic capture 和严格文档构建。

## R3：RL 批量感知渲染

当前单 view `Renderer` 保留为兼容和交互路径。批量路径按以下所有权顺序推进：

1. 把 mesh、texture、material、sampler 和 pipeline 拆为 peer 可共享的 immutable resources；
2. 以 flat selected views 表达 camera-to-world 映射，不强制 dense world × camera 组合；
3. 用 texture arrays、per-view frame offsets 和一个 encoder/submission 完成同 topology 多相机渲染；
4. 用 batched pose upload 和 per-world bucket ranges 扩展到 replicated topology cohorts；
5. 仅在后端存在真实同步与所有权桥接时提供 device-resident tensor result。

验收必须包含 serial reference 像素一致性，以及 1/4/16/64/256 views 在 128²/256² 的 batch latency、
views/s、pixels/s、CPU/GPU time、upload bytes、submission count 和内存。完整设计见
[`docs/BATCH_RENDERING.md`](../docs/BATCH_RENDERING.md)。

## R4：第二个真实物理适配器

ToyPhysics 继续作为协议示例。生产适配器按以下顺序推进：

1. 评估 Box3D、Newton 等候选的 binding、许可、平台、模型输入和 API 稳定性；
2. 发布稳定 `SceneSource` 与按需 `SceneFrame`；
3. 实现 pause、step、reset、pose write-back 和 perturbation；
4. 接入 contact、sensor 和 debug draw；
5. 通过 adapter conformance、独立 viewer acceptance 和 remote path。

不能通过 capability 的功能必须显式关闭，UI 不根据适配器名字猜测支持范围。

验收：

```bash
make adapter-conformance ADAPTER=<backend> CONFORMANCE_ASSET=<asset>
```

并为真实模型增加独立 visual target。

## R5：远程与格式边界

当前 live view 和 `.fvs` 只支持可信环境。若产品需求扩展到非可信网络：

- 先设计非可执行、带长度限制的 schema；
- 分离结构资源、动态帧和 command authorization；
- 对 mesh/texture 大小、debug command 数量和频率设置预算；
- 在协议版本不兼容时明确失败，不做模糊降级。

在此之前不把固定 auth key 描述成安全 transport。

## R6：由测量触发的图形工作

以下项目不设虚假日期，由 profile、模型语料或 parity regression 触发：

- frustum culling、LOD、indirect draw 和 instance-buffer 局部更新；
- 大型 mesh BVH diagnostic acceleration；
- point-light shadow cube face 重绘与更复杂的 light scheduling；
- SDF iteration visualization；
- PBR 和 native renderer core 研究。

每项开始前先保存可复现基线和失败模型，完成后用相同输入比较质量、CPU、GPU 和内存。

## v1 完成条件

- 当前公开 Renderer、Scene、adapter、remote、recording 和 workspace 工作流有稳定说明与示例；
- 三个平台至少各有一个已验证 renderer path，或者明确缩小支持矩阵；
- wheel 能在 clean environment 安装、启动、capture；
- editor 的高频结构化范围、source escape hatch 和 source-owned 范围有清楚边界；
- 当前格式有版本策略，非可信数据不经过 pickle；
- `make check`、相关 physics/GPU gates、`make docs-check` 和 reviewed visual targets 通过。

## 验收命令索引

| 范围 | 命令 |
|---|---|
| 核心质量 | `make check` |
| 文档与示例 | `make docs-check` |
| OpenGL Renderer | `make renderer-api`、`make gpu` |
| wgpu Renderer | `make renderer-api-wgpu`、`make gpu-wgpu` |
| MuJoCo | `make mujoco-physics`、`make mujoco-audit` |
| Adapter | `make adapter-conformance`、`make toy-physics` |
| Workspace/MJCF | `make workspace-edit`、`make mjcf-roundtrip` |
| Formats/RPC | `make format-validation`、`make rpc-soak` |
| Stability | `make stability BACKEND=wgpu` |
| Regression strength | `make reverse` |
| User-visible review | relevant target from `make help` |
