# 后续路线图

## 工作方式

- 每个用户可见能力都配一条 `make` 验收入口；自动测试守性质，`make` 场景负责肉眼裁决。
- renderer 只认 `SceneSource` / `SceneFrame` / 绘制命令，不 import 物理库、RPC 或 UI。
- 不启用 ImGui platform multi-viewport。PVD 式多窗口用独立 viewer 进程，每个窗口持有自己的
  GLFW、OpenGL 与 ImGui context，互相关闭不连坐。
- 实时远程查看采用“结构可靠送达、帧只保留最新”的语义，不让渲染反压物理步进。

## 当前基线

普通刚体 MuJoCo 场景已经覆盖：MJCF / URDF、基础 geom、mesh 与 heightfield、site、tendon、
contact point/force 三维标注、材质和颜色贴图、天空、
静态/动态灯、关节与 actuator、传感器数据、接触力曲线、播放/暂停/重置、qpos/ctrl 编辑、
自由体精确位姿、物理扰动、射线/GPU 拾取、描边、debug draw、阴影、反射和录像。

因此它已经适合作为大多数机器人刚体场景的渲染与调试前端，但还不是逐像素、逐特性的
`mjvScene` / `simulate` 替代品。核心场景渲染与 MuJoCo 的全部诊断 overlay 必须分开记账。
已确认的缺口是：

| 缺口 | 当前行为 | 优先级 |
|---|---|---|
| `mjVIS_JOINT/ACTUATOR/ACTIVATION` | joint、site、body、tendon transmission 与 activation 调色已完成；slider-crank 连杆待补 | P2 |
| light / camera viewport icon | Hierarchy/Inspector 编辑、场景图标与 named camera 视图共用通用 scene entity | done |
| rangefinder / constraint | site/camera rays and connect/weld equality endpoints | done |
| inertia / scaled inertia / COM | 已进入通用 debug draw | done |
| island / contact split / autoconnect | 未实现对应分组与标记 | P2 |
| flex vert/edge 与 body/mesh BVH / SDF iter | 只画最终 flex/skin/SDF 表面，不画内部调试 overlay | P2 |
| MuJoCo 的 7 套 visual group | geom/site/joint/tendon/actuator/flex/skin 已独立过滤 | done |
| static / skin / flex face / flex skin flags | independent instance filters with MuJoCo defaults | done |
| tendon material / texture | 已接入 RGBA 覆盖、材质标量、贴图分桶、repeat 与透明路径 | done |
| camera principal point | physical intrinsics 与非居中投影已支持 | done |
| image light / 超过 16 盏灯 | image light 近似方向光；只取前 16 盏 | P2 |
| `mjtLabel` / `mjtFrame` | 有通用 GPU text/debug draw，但尚未把 MuJoCo 的 label/frame 模式接成 UI | P2 |
| mocap / equality 编辑 | mocap pose 复用 transform gizmo；equality 可在 Control 中动态开关 | done |
| reflectance 完整语义 | 单主反射面可用；多反射面及 MuJoCo 的 plane/box-face 细则尚未逐项对拍 | P2 |
| `mjRND_ADDITIVE` | 未实现，Settings 中明确置灰 | P2 |
| IK | capability 明确为 false | P2，独立于渲染替换 |

`make mujoco-audit` 会逐项列出当前 MuJoCo 版本的全部 `mjtRndFlag` / `mjtVisFlag`，
并分成 supported / degraded / unsupported；同时打印 adapter 已提供和未提供的写入接口。
严格模式只因“当前模型的实例数据被跳过”失败，不会因为全局尚未实现的诊断 overlay
阻止普通场景加载。

## R0.7：Forge 原生编辑实体

灯光的所有权已从物理 adapter 收回 Forge 场景：`SceneSource` 保存类型与全部渲染设置，
`Session` 保存用户 override，`SceneFrame` 只允许后端提供 body-attached light 的动态世界位姿。
因此 `make lighting`、`make toy-physics`、MuJoCo 和远程回放共用同一个 Hierarchy/Inspector
编辑入口；MuJoCo 的写回失败也不能否决 Forge 侧编辑。

下一步不是继续给 MuJoCo Inspector 加特例，而是补齐 Forge 自己的组件：

1. environment/headlight 节点：编辑全局 ambient、fog、haze 与 camera headlight；
2. ✅ camera 节点：程序化、自定义后端与 MuJoCo 都可创建、选择和编辑相机；
3. material 组件：在 Forge 层编辑材质，并明确共享材质与实例 override 的语义；
4. 稳定实体 id 与 add/remove API，使 RPC、快照回放和未来玩具引擎不依赖数组下标。

每一项都先在 `make canvas` / `make lighting` 的无物理路径验收，再接 MuJoCo 导入与写回。

## R0.5：世界空间文字 ✅

`Layer.text()` 已定义 `world anchor + screen offset + alignment + depth/always/ghost`
语义，走 forge 的 GPU pass，不依赖 ImGui draw list。生产组装把 ImGui 实际选中的
JetBrains Mono / CJK 字体文件和字号传给同一字形图集；无 UI 的 forge target 与
debug socket 也能渲染同一条 `text` 命令。验收入口：`make text-overlay`。

## R1：MuJoCo 可视化完整度

1. ✅ 覆盖审计命令逐模型列出 supported / hidden / degraded / unsupported，也穷举全部
   `mjtRndFlag` / `mjtVisFlag`；严格模式遇到真正跳过的模型内容返回非零。
   验收入口：`make mujoco-audit`。
2. ✅ heightfield、site、tendon、contact point/force 已进入通用场景帧或 debug draw，forge
   pass 不 import MuJoCo。验收入口：`make mujoco-visuals`。
3. ✅ 模型相机可在 free / named camera 间切换，挂在 body 上的相机逐帧跟随；六个 geom
   group 开关同时重建画面与射线过滤。验收入口：`make cameras`、`make geom-groups`。
4. ✅ flex/skin 使用“静态拓扑 + 逐帧顶点/法线”的通用动态网格契约；1D flex 是按模型
   半径生成的圆管，2D/3D flex 与 skin 逐值对拍 MuJoCo `mjvScene`，静态 mesh 快路径不变。
   验收入口：`make deformables`。
5. ✅ joint、COM、inertia、scaled inertia 与 joint/site/body/tendon actuator overlay 已进入
   通用 debug draw。剩余 overlay 按上表继续推进。

完成标准：审计命令对内置覆盖场景不再报告非预期跳过项；每类能力都有一份正例、一份反例
和一个可交互 `make` 场景。

## R2：PVD 式远程查看

1. ✅ 定义传输无关的快照：`structure_revision + frame_sequence + SceneFrame + DebugBatch`。
   结构变化可靠送达；普通帧允许丢弃旧帧，只消费最新帧。
2. ✅ 实现 `RemoteSceneAdapter`，让远端数据继续走现有 Session、命令、forge 和 UI，不另写
   一套 remote viewer。
3. ✅ 命令通道与画面通道分离：选择、暂停、step、参数编辑可以有明确回执；debug draw 与帧
   数据不等待 UI。
4. ✅ 提供 `serve` / `attach` CLI 和 `make pvd`：自动启动一个 physics publisher、一个
   普通效果窗口、一个 normal debug 窗口；两个 viewer 可独立关闭并使用各自的自由相机。
5. ✅ 在同一快照协议上补录制/回放，使问题现场不依赖物理进程仍然存活。验收入口：
   `make snapshot-record`、`make snapshot-replay`。

远程模式传输 scene hint、named camera 元数据与逐帧位姿。attach 窗口可以查看和编辑远端
camera entity，也可以保留独立的自由相机。

这里不需要 ImGui 多视口，也不要求两个窗口属于同一进程。真正要共享的是带序号的场景状态，
不是 UI 布局和 OpenGL context。

## R3：正式的第二物理后端 ✅

`ToyPhysicsAdapter` 已从测试夹具提炼成正式后端：它有独立 timestep、重力、地面碰撞、
Reset 与暂停后的位姿编辑，不 import MuJoCo；关节、扰动、射线等能力刻意不实现，用于验证 UI
仍能清楚降级。通用 conformance report 同时通过 toy、基础 MuJoCo 和 flex/skin 场景。
验收入口：`make toy-physics`、`make adapter-conformance`；第三方引擎可直接复用
`check_adapter()`，不用新增 renderer 或 UI 分支。

后续接用户自己的物理引擎属于新增 adapter，而不是补架构缺口。

## R4：图形 API 与材质管线（按需求触发，不立即迁移）

Vulkan/Metal 迁移与 PBR 是两项独立工作：PBR 可以在 OpenGL 上实现，换成 Vulkan 也不会
自动得到 PBR。当前 MuJoCo 优先的产品基线继续使用 `specular / shininess / emission /
reflectance`，不为了一个尚无材质数据来源的 PBR 工作流重写 renderer。

只有出现以下任一实际需求时，才启动新后端原型：

- 需要 compute、SSBO、GPU-driven rendering 或远超当前规模的动态实例；
- 需要进入不再提供可用 OpenGL 的 Apple 平台；
- macOS 的兼容层已无法稳定满足现有功能；
- 引入 glTF/自定义材质，并明确需要 metallic-roughness、normal map、HDR/IBL 与 tone mapping。

原型优先在现有 `RenderBackend` / `SceneSource` 契约后并列增加 wgpu 后端，用最小的 opaque、
ID picking、outline 场景做一致性对拍；不先把整个项目搬到 C++。验证 Python 提交开销确实成为
瓶颈后，再把 renderer core 局部下沉到 Rust/C++ 动态库，物理 adapter、命令、远程协议和 UI
继续复用。OpenGL 后端在新后端达到相同验收线前保留。

这条路线的成本是中高：场景与交互架构可复用，但 GL context、buffer/texture、pipeline state、
shader binding、render target、同步与 present 都要重写。它是受边界约束的后端替换，不是低成本
开关，也不是必须推倒重来的全工程迁移。
