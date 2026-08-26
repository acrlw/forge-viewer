# forge-viewer 编辑器渲染属性、G1 关节 Gizmo 与模型语料验收

> 状态：实现与验证完成
>
> 代码快照：`8155397`（基于 `b9436eb`）
>
> 日期：2026-08-27
>
> 输入：MuJoCo Menagerie `da76818`、Unitree `unitree_ros-master/robots`、本机
> `g1_description/g1_23dof.xml`

## 核心结论

1. Editor 现在可以编辑非插件渲染数据中的环境、已有纹理引用、共享材质和全部 Forge 灯光类型；MJCF
   导出能够保存 haze、skybox、2D/cube texture、材质、area/image light 和环境参数。
2. G1 暂停后不能使用关节 gizmo 的实际原因是 `ViewerApp.frame_needs()` 覆盖了 gizmo 请求的
   `diagnostics=True`。修复为需求并集后，真实 G1 在 Viewer 与 Editor 中均有 23/23 个 hinge link 显示
   轴向旋转 gizmo。一个 body 有多个 direct joint 时，viewport 左上角显示 joint 选择器。
3. Menagerie 的 237 个独立模型或场景均通过 direct adapter、Workspace、composition 和 WGPU 三视角
   RGB/segmentation 检查；26 个 XML 是组合片段。Unitree 的 84 个入口中 77 个通过相同检查，7 个因当前
   MuJoCo 没有 DAE decoder 跳过；真实 load/adapter/composition/render 失败均为 0。
4. Light 提交接口已明确使用 `light_index`，`Scene.set_light()` 使用稳定 light ID；无法写回的 light
   override 按稳定 object ID 保留，删除前面的灯后不会改错 slot。Material 仍明确使用
   `material_index`，因为当前共享材质数据没有稳定 material ID。
5. Light 主路径没有按灯数量增长的 O(L²) 循环。高灯数风险来自 shadow pass 的乘法重绘，而不是
   light metadata 查找。4096 实例专用 culling/LOD/indirect rendering 按用户决定延期。

## 用户需求状态

| 结果 | 状态 | 当前行为或证据 |
|---|---|---|
| Plane 可选中、高亮、transform、改长宽 | 完成 | MuJoCo world-owned static plane；位姿和尺寸写回 spec；Undo/Redo 保持 |
| 多个 topology 操作只 compile 一次 | 完成 | `ModelEditBatch` 原子执行 add/remove/rename；批内引用、失败回滚、单条 Undo |
| 加载不阻塞旧 viewport | 完成 | 单 worker 队列；浮动 widget 显示动作、路径、耗时和排队数 |
| 加载成功、失败、退出、连续排队 | 完成 | 异步生命周期回归覆盖四种情况 |
| 错误信息可复制 | 完成 | 文件、资源修复、Inspector 与 Control 错误入口均有复制按钮 |
| Unsaved changes 布局 | 完成 | 文件名独立一行，按钮等宽，按当前 viewport work area 居中 |
| Unitree `meshes/meshes` | 完成 | 双层路径不存在且单层路径存在时才在内存中去掉重复前缀 |
| G1 link 关节 gizmo | 完成 | Viewer 23/23、Editor 23/23 实物模型验证 |
| 多 direct joint 选择 | 完成 | viewport joint chooser；hinge/slide/ball 可选，运行中提示 Pause |
| Inspector 内联重命名 | 完成 | model element 与 authored link/light/camera 支持 Enter 或 Apply |
| XYZ 输入对齐 | 完成 | transform、camera、light position/direction/attenuation 使用统一表格布局与轴重置 |
| Settings checkbox 排布 | 完成 | render flag 使用 3 组 label/checkbox table；visual group 使用 7 列表格 |
| stable ID 与 array slot 区分 | 完成 | camera/light entity ID 与 `*_index` 分开；remote 接收旧字段但发送新字段 |
| Forge/WGPU 验证 | 完成 | WGPU 全门槛通过；Forge UI/model-loading 55/55；离屏 EGL 项按用户决定忽略 |
| MuJoCo 真实阶段百分比 | 不实现 | Python binding 没有 parser/mesh/compiler 阶段回调，不显示伪进度 |
| 4096 实例专用 batch renderer | 延期 | 当前已有 instance bucket、batched SceneFrame/debug draw；等待真实规模 profile |

## G1 为什么是关节 Gizmo，而不是任意 Transform

Revolute/hinge link 不能像 free body 一样接受任意六自由度 transform。它的世界位姿由父链和一个标量
`qpos` 决定，因此正确交互是把旋转环约束到 joint axis，并把拖动结果写回该 joint 的 qpos：

- hinge/revolute：绕 reference axis 的旋转环；
- slide：沿 reference axis 的平移箭头；
- ball：三自由度旋转 gizmo，一次提交四元数 qpos batch；
- free：普通 position/rotation gizmo；
- 多 direct joint：先在 viewport 选择 joint，再显示该 joint 的 gizmo。

旧代码的 `ObjectGizmo.frame_needs()` 已经请求 qpos 与 diagnostics，但 `ViewerApp.frame_needs()` 随后按
render flag 重新赋值，把该请求清成 `False`。Inspector 先前只修正了提示判断，因此出现“文字说可以、实际
viewport 没有 gizmo”的回归。本轮把 contacts、tendons、actuator、deformables、islands、BVH 和 diagnostics
全部改为与 consumer 请求取并集，并增加 Viewer/Editor 真实组合管线 GPU 测试。

## Editor 渲染属性与 MJCF 保存

### Inspector 当前入口

| 对象 | 可编辑属性 |
|---|---|
| Environment | skybox enable/texture、ambient、headlight、fog、volumetric/horizon haze、haze slices |
| Material | instance color、shared base color、Matte/Plastic/Metal/Rubber/Emissive preset、emission、specular、shininess、reflectance、2D texture、repeat、uniform |
| Light | active、cast shadow、type、diffuse/specular/ambient、position、direction、attenuation、range、spot cutoff/exponent、area radius、image texture/intensity |
| Camera | eye、target、up、FOV、clip、orthographic 与物理内参 |

Texture 入口当前是选择模型或场景中已经存在的 texture；Editor 还没有“从磁盘导入一张新图片并创建 texture
asset”的资源浏览工作流。Material preset 是 Forge 参数模板，不会伪装成 MuJoCo 内置的命名材质。

### MJCF 表达

| Forge 数据 | MJCF 保存方式 |
|---|---|
| 2D texture | PNG asset + native `texture type="2d"` |
| cube/skybox | 六张 PNG + native cube/skybox texture |
| image light | native MuJoCo image light、cube texture 与 intensity |
| area light | native point light + `bulbradius`，并用 `forge_viewer.light.area` text metadata 恢复语义 |
| ambient | `forge_viewer.environment.ambient` custom numeric |
| haze mode/slices | native visual 值 + `forge_viewer.environment.horizon_haze` custom numeric |
| fog distance | 保存时除以 compiled scene extent，读取时乘回 extent |

附加模型的 image light 在 composed model 中引用带前缀 texture 名，写回子 MjSpec 时会剥离该前缀；跨模型
texture 引用无法在独立子模型中持久化，因此返回不可写回并保留 Forge override。Skybox 切换同时清除 primary
与 authored 侧的旧选择，避免 Workspace 合并后出现两个有效 skybox。

## 模型语料结果

### MuJoCo Menagerie

- 根目录：`output/mujoco_menagerie`
- revision：`da76818`
- 发现入口：263
- 独立模型/场景通过：237
- 组合片段：26
- load/adapter/workspace/composition/render/empty-render failure：0
- WGPU 条件：320×240、3 个 camera pose、2 个 dynamic step、RGB + segmentation
- 原始报告：`output/mujoco-menagerie-load-only-v6.json`、
  `output/mujoco-menagerie-wgpu-v1.json`

Fragment 只在以下条件归类：文件被另一份 MJCF `include`，或者自身没有 include、没有独立 body/geom，且加载
失败来自外部对象或 keyframe state 依赖。合法但省略 `model=` 的 MJCF 不再被宽泛跳过。

### Unitree

- 根目录：`/home/oem/下载/unitree_ros-master/robots`
- 发现入口：84
- compile/direct/workspace/composition/WGPU 通过：77
- DAE decoder 缺失：7
- 其他 failure：0
- 原始报告：`output/unitree-model-suite-v5.json`、
  `output/unitree-model-suite-wgpu-v1.json`

G1、H1/H2、Go1、Laikago、Z1、R1 和 dexterous hand 的 STL/OBJ 入口均通过。7 个跳过项明确引用 DAE；当前
MuJoCo 错误是没有 decoder，而不是 viewer 路径解析或 adapter 失败。

## 性能复核

当前 light CPU 路径为线性：Session 动态灯合成 O(L)，scene helper O(L)，两后端 light scheduling O(L)。
renderer 最多提交 100 个 active non-image lights，并限制 8 个 local shadow slots。高成本来自每个 shadow
face/cascade 重画 opaque geometry：point light 最多六个 cubemap face，因此 shadow 成本近似
`opaque buckets × shadow faces`。

已经处理的编辑器热点包括 camera override O(C²)、camera/light hierarchy 重扫、逐 primitive helper、selected
joint/actuator 全表扫描、大 hierarchy/slider 列表、retained color O(KN)、tendon anchor O(TS)、WGPU instance
staging 分配、未启用时 eager 构建百万 BVH records，以及无 viewer 时 remote 重复 pickle。详细计时与 4096
实例/箭头 microbenchmark 见上一份
[`REPORT-editor-loading-and-large-scenes`](../20260826/REPORT-editor-loading-and-large-scenes.md)。

单个 UI 手势继续使用直接接口；同质高基数数据使用 batch 接口。当前可用边界是 plural debug primitives、
batched `SceneFrame`、`SetQposBatch` 与 `ModelEditBatch`。没有为尚不存在的 component 多选 UI 添加空 batch
协议，也没有提前实现 4096 机器人专用 renderer。

## 对已有 code review 的交叉结论

`REPORT-code-review` 正确识别了 MuJoCo compatibility facade、RPC camera 180°、WGPU release、remote
生命周期、正交逆矩阵、sRGB 和 backend contract；这些问题已在前序提交中修复。

需要收窄的部分：

- `enable_*_rendering()` 是 `mujoco.Renderer` 兼容方法，不应直接改成 `set_*`；内部统一开关已经是
  `set_flag(flag, bool)` 和 enum debug view。
- `srgb=False` 表示输入已是线性数据，应使用 unorm 且不再次 CPU decode；不能简单描述为条件反转。
- CameraView 零内参表示切回 FOV，在线清零正确；错误是 editable spec 未同步，重编译后会恢复旧值。
- Euler helper 与 Inspector 往返自洽，确定错误是 docstring；直接改变乘法顺序会破坏已有 UI 语义。
- 静态审计漏掉 camera O(C²)、百万 BVH eager build、large ImGui list、scalar debug traffic、full instance
  upload、shadow multiplier，以及本轮真实复现的 `frame_needs` 覆盖和多 Viewer GL context 切换。

项目不需要整体拆层。`Scene`、`SceneSource/SceneFrame`、render scene 分别承担 authoring、交换数据和 GPU
bucket，属于有效管线。应继续优先修复跨层身份、所有权和可观察语义，而不是机械消除类或统一所有动词。

## 验证汇总

| 命令或入口 | 结果 |
|---|---|
| `make check` | 579 fast、67 integration 通过 |
| `pytest -m 'physics and not gpu'` | 177 通过，1 条既有 flex warning |
| `make mujoco-audit` | strict PASS |
| `make adapter-conformance ADAPTER=mujoco CONFORMANCE_ASSET=deformables` | PASS |
| `make gpu-wgpu` | 全文件通过；backend-specific skip 按预期 |
| Forge model-loading + UI | 55/55 通过 |
| WGPU model-loading + UI | 55/55 通过 |
| G1 23-DOF Viewer/Editor | 各 23/23 hinge link gizmo 可见 |
| Menagerie WGPU corpus | 237 pass、26 fragment、0 failure |
| Unitree WGPU corpus | 77 pass、7 unsupported DAE、0 other failure |

`make gpu` 的离屏 `Renderer` 文件仍在当前机器的 `eglInitialize failed (0x3001)` 处失败；同一功能的 WGPU
renderer tests 11/11 通过，Forge 的 GLFW viewport/UI tests 可在 `DISPLAY=:1` 运行。该 EGL 环境项按用户决定
不处理。

## 提交与工作树

- `5922962 Batch MuJoCo topology edits`
- `acf70fb Align joint gizmo status with viewport`
- `b9436eb Document topology batching and G1 gizmos`
- `8155397 Complete editor authoring and model validation`

当前没有 stash。根目录 `test-scene.forge.json` 是用户的未跟踪文件，未读取、未修改、未提交。
