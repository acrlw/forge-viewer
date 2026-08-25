# MuJoCo humanoid 渲染对齐与历史任务收口

状态：关键缺陷已修复并提交，远端文档更新已合入，完整 CPU、physics、Forge/WGPU GPU 验证通过；待推送
结论：humanoid 的聚光灯硬边来自 XML `cutoff="30"` 和 MuJoCo classic 的硬锥边界；最新截图中的棋盘锯齿则不是模型语义，而是 Forge 曾尝试按无限平面网格顶点插值聚光衰减造成，现已恢复逐像素计算。模型本身确实定义了 horizon haze。旧 interactive viewer/editor 曾把模型的 `far≈50.02` 覆盖成 `far≈243.69`，造成一条额外的远处几何带，该缺陷已经修复。另一个相机跟随缺陷来自模型根变换后恢复旧的世界坐标 free-joint 状态，现已随根变换同步变换 free-joint/mocap 状态。暂停状态下跟踪灯 gizmo 的回弹也已修复。
涉及范围：MuJoCo adapter、Forge/WGPU skybox 与 haze pass、灯光与纹理、viewer/editor 模型替换、IK 删除、路线图与历史任务审计
记录日期：2026-08-25

## 两张截图的结论

### 聚光灯边界不是影子尺寸

模型文件
`/home/oem/下载/mujoco-3.10.0-linux-x86_64/mujoco-3.10.0/model/humanoid/humanoid.xml`
定义：

```xml
<light name="spotlight" mode="targetbodycom" target="torso"
       diffuse=".8 .8 .8" specular="0.3 0.3 0.3"
       pos="0 -6 4" cutoff="30"/>
```

本地 MuJoCo 3.10 源码 `src/render/classic/render_gl3.c` 把该值直接写入
`GL_SPOT_CUTOFF`。锥外光照为零，因此亮区会有模型定义的 30° 硬边。另一个默认参数
`vis.map.shadowscale=0.6` 只调整 classic shadow-map 投影覆盖，不决定可见光锥。

Forge 当前为阴影贴图覆盖完整灯锥。这属于现代阴影实现与 MuJoCo classic 的算法差异，
不会改变 light 的 position、direction、cutoff、exponent 和 attenuation 操作语义。若要柔边，
应作为 Forge-native 的 inner/outer cone 能力增加，不能在 MuJoCo parity 路径中静默改写
`cutoff`。

### 最新截图中的棋盘锯齿不是 MuJoCo spotlight

为模拟 MuJoCo classic 的固定功能管线，Forge 曾把无限平面的网格尺寸编码进纹理坐标，并在
GLSL/WGSL 中把聚光衰减投到网格三角形顶点后插值。小 cutoff 下只有少数顶点落入锥体，亮区因此
扩张成与地板棋盘对齐的方块和三角形；过曝的大矩形并不是灯光参数本身。

现在 builder 不再向纹理坐标塞入灯光状态，Forge/WGPU 都直接用当前 fragment 的世界坐标计算
distance、cutoff、exponent 与 attenuation。真实 viewer/editor 在 `cutoff=5.1°` 下只留下实际小锥体
覆盖范围，不再出现棋盘锯齿。该修改保留 MuJoCo 的 position、direction、cutoff、exponent 和
attenuation 操作语义，同时不复刻过时固定网格的光照伪影。

### 蓝灰淡带是模型定义的 horizon haze

同一 XML 同时定义渐变 skybox 和 haze 颜色：

```xml
<rgba haze="0.15 0.25 0.35 1"/>
<texture type="skybox" builtin="gradient"
         rgb1=".3 .5 .7" rgb2="0 0 0" width="32" height="512"/>
```

截图淡带的中间色约为 `[38, 64, 89]`，正好对应
`0.15/0.25/0.35 × 255`。MuJoCo 的 haze 是两层截锥，alpha 从 0 过渡到 1 再回到 0；
在低相机角度下会形成一条可见带。它不是距离 fog，也不是额外的后处理渐变。

514×699 竖向画幅下，Forge 使用 MuJoCo 参考 renderer 产生的同一组相机矩阵重新渲染
五个视角，全部通过 parity gate。`low` 视角的 sky/haze 边缘区域 p99 通道误差为 1，
没有 Forge 独有的附加淡带。产物位于：

- `output/humanoid-vertical-parity/humanoid/low.triptych.png`
- `output/humanoid-vertical-parity/humanoid/front.triptych.png`

关闭 `RenderFlag.HAZE` 或把 XML 的 `<map haze="0"/>` 设为零可以去掉该带，但这会改变
模型语义，不应成为 MuJoCo adapter 的默认行为。

## 为什么 parity 正常而 interactive viewer/editor 多出远处条带

两条入口使用的是同一个 Forge renderer，差异发生在送入 renderer 之前：parity 沿用
MuJoCo 模型的相机裁剪面，而 interactive 启动和加载模型后会调用通用 `frame_scene()`，旧实现
根据场景包围盒重新计算 near/far。

humanoid 本次实测值：

- MuJoCo camera hint：`near=0.016675`、`far=50.024254`；
- interactive 旧值：`far≈243.6932`。

MuJoCo classic 的无限平面/skybox 深度关系受 far plane 影响。把 far 拉到约 244 后，地面会向
几何地平线多延伸一段，视觉上正是用户红箭头指出的暗灰条带。关闭 haze 后条带仍存在，因而
可以排除 haze pass。使用 interactive 的同一相机姿态、但恢复模型 clip planes 后，该条带消失并
与 native 渲染对齐。

修复后，`ViewerApp._frame_scene()` 仍按场景包围盒决定构图，但明确沿用
`Session.camera_hint()` 的 near/far；没有模型相机语义的通用场景仍采用自适应裁剪面。GPU 回归
锁定 interactive entry 的 near/far，避免 viewer/editor 再次走偏。

### 修复后截图中仍可见的淡带

2026-08-25 最后一组 4K 截图已经使用模型裁剪面，并不是上述旧缺陷复发。根据截图的模型投影、
棋盘透视和地平线逐行 RGB 反求相机，editor 截图约为
`yaw=-141°`、`pitch=13.5°`、`distance=2.5`、`far=50.024`；作为对照的 MuJoCo GUI 截图约为
`yaw=-133°`、`pitch=13.5°`、`distance=2.5`。两张肉眼相近的截图实际相差约 8° 方位，足以移动
远处棋盘和定向灯/聚光灯在地面的明暗交界。

交叉渲染排除了入口和 ImGui 合成差异：

- 用 editor 截图反求的相机分别渲染 Forge 与本地 MuJoCo classic，平均通道误差为 1.65；
- 用 MuJoCo GUI 截图反求的相机分别渲染 Forge 与 MuJoCo classic，平均通道误差为 1.68；
- 反求的 Forge 帧与实际 editor viewport 平均通道误差为 1.60，说明相机拟合确实对应截图，而不是
  另选一个更容易通过的视角。

证据位于 `output/interactive-entry-debug/user-camera-fit/`：

- `best-user-forge.triptych.png`：实际 editor viewport、反求相机的 Forge 帧、差分；
- `best-forge-native.triptych.png`：editor 相机下 Forge、MuJoCo、差分；
- `native-shot-vs-forge-at-native-camera.triptych.png`：实际 MuJoCo GUI viewport、同相机 Forge 帧、差分；
- `native-camera-forge-native.triptych.png`：MuJoCo GUI 相机下 Forge、MuJoCo、差分。

因此不能在 adapter 或 shader 中把修复后仍存在的淡带静默抹掉；那会令相同相机下的 Forge 输出
偏离 MuJoCo。若产品后续需要更干净的现代渲染，应作为明确关闭 `RenderFlag.HAZE` 或 Forge-native
环境样式的可选行为，而不是 MuJoCo parity 默认值。

## 为什么拖动 humanoid 后 back/side camera 不再跟随

workspace 移动模型根节点时需要重新编译组合后的 `MjModel`。旧实现先保存运行状态，重新编译后
再原样恢复 qpos。humanoid 的根是 free joint，其平移和四元数是世界空间状态；原样恢复会把刚刚
应用到模型根的变换抵消。`back`、`side` 相机绑定在 torso body 上，因此 body 和相机都停留在旧位置。

现在恢复前先计算 `new_root * inverse(old_root)`，用该增量变换属于该模型的 free-joint 世界位置、
姿态、世界线速度，以及 mocap pose，再恢复其余运行状态。回归通过 gizmo 实际使用的
`SetPose(model node)` 命令路径验证 body 和全部模型相机；真实 humanoid 平移 `[2, -1, 0.5]` 后，
`back`/`side` 的 eye 与 target 都精确移动同一向量。

## 为什么暂停时 light gizmo 会回弹

MuJoCo 的 `track`/`trackcom` 灯并不在每次 `mj_forward()` 时直接使用 `light_pos`。编译模型时它会
生成 `light_pos0`、`light_poscom0` 和 `light_dir0`，运行时再用 body position 或 subtree COM
恢复世界姿态。旧写回只改 `light_pos/light_dir`，所以 gizmo 命令虽然成功，下一帧仍被编译期派生
数组覆盖。

`MuJoCoAdapter.set_light()` 现在按本地 MuJoCo `engine_core_smooth.c` 的 setconst 语义同步刷新这三组
派生值，再执行 `mj_forward()`。真实 paused viewer/editor 中移动 humanoid 的 `top` 跟踪灯后连续刷新，
两条入口都保持目标位置 `[0.35, -0.2, 2.15]`，选中灯的平移 gizmo 也持续显示。截图位于
`output/spotlight-gizmo-fix/`。

## 实际发现并修复的 haze 缺陷

MuJoCo classic 在 opaque skybox 阶段绘制 haze，保持 depth write，然后才绘制透明几何。
Forge 与 WGPU 旧实现只混合颜色、不写深度，导致 haze 后方的透明几何完整穿透显示。

判别资产：`haze-transparent-probe.xml`。同一 640×480 固定相机下：

| renderer | 修复前紫色透明球像素 | 修复后像素 | 边界框 |
|---|---:|---:|---|
| MuJoCo classic | 847 | 847 | y=215..241, x=299..340 |
| Forge | 1825 | 831 | y=216..241, x=299..340 |
| WGPU | 1825 | 由共享回归验证 | 与 Forge 语义一致 |

修改后 Forge 仅剩抗锯齿和光栅化造成的 16 像素计数差，垂直可见范围与 MuJoCo 相同。
回归测试 `test_mujoco_haze_writes_depth_before_transparent_geometry` 在 Forge 与 WGPU 均通过。

## 历史需求收口审计

| 用户提出的事项 | 当前状态 | 证据或剩余动作 |
|---|---|---|
| rotation gizmo 极端角度、橙色圆弧、刻度高亮、平头与端点方向 | 已完成并提交 | `5c9289f`、`6fb6822`，对应 CPU/GPU gizmo 回归 |
| 路线图 1/2/3，4 暂缓 | 1/2/3 已实现，4 依用户要求暂缓 | workspace/editor 系列提交 `24b523f`、`f2ddf70`、`98d8bee`；暂缓项不算中断 |
| 空场景创建 plane 出现重复 hierarchy/ImGui ID | 已完成并提交 | `f2ddf70` 清除重复 child edge，覆盖所有 primitive |
| point light rotation gizmo | 已完成并提交 | `98d8bee` 允许 point light 进入 rotation gizmo，并做拖动帧回归 |
| error widget 过窄、最大化后不居中 | 已完成并提交 | `98d8bee` 固定可读宽度并每帧按主 viewport 居中 |
| finite plane 无法单击、不能调长宽 | 已完成并提交 | `98d8bee` 增加 pick 回归和 width/length 编辑；MuJoCo 的 `size="0 0 ..."` plane 仍保持无限语义 |
| 删除并重建 API 文档 | 已完成 | 已合入远端 `167d4cf`，本地扩展页全部进入 nav；`make docs-check` 检查 15 个 API 模块、examples catalog 并严格构建 `output/site/` |
| viewer/editor 加载同一 humanoid 画面不一致 | 修复并验证，待提交 | 已修复同名 GPU 纹理跨场景复用和 interactive 覆盖模型 clip planes 两个独立根因；direct/editor 与 native 对照通过 |
| 拖动组合模型后命名相机不跟随 | 修复并验证，待提交 | free-joint/mocap 世界状态随模型根增量变换；真实 `SetPose` gizmo 命令路径与 humanoid back/side 均通过 |
| texture、camera、haze/fog、light operation 与 MuJoCo 对齐 | 已修复并完成本地验证，待提交 | 包含 adapter、两后端 shader/pass、相机预览、重复加载、逐像素 spotlight 和暂停 tracking-light 写回；SDF 插件仍按既定范围不支持 |
| 删除未定型 IK | 本轮补齐，待提交 | 主体此前已删；本轮再删除 Session 死状态与路线图残留，全仓项目文件已无 IK 标识 |
| depth API 命名权衡 | 决定完成，无代码变化 | 保留 MuJoCo 兼容的 `enable_/disable_depth_rendering()` 与 segmentation 对称接口 |
| CI/CD、路线图第 4 项 | 明确不做 | 用户主动暂缓，不列为遗漏 |

## 当前仍需处理

1. MuJoCo parity、texture replacement、haze depth、spotlight、跟踪灯写回与 IK 删除已提交为
   `4dd27a7`；API reference 导航已提交为 `3f55f2e`，待推送到远端 `main`。
2. 本机默认 EGL standalone 初始化仍报 `eglInitialize failed (0x3001)`；项目支持的
   `FORGE_VIEWER_GL=glfw` 路径已通过完整 physics/GPU。GPU 全量过程中发现并修复 outline pass 在
   shared ID attachment 驱动上污染 object ID 的问题，原先两项 `test_id_outline.py` 失败现均通过。
3. 用户仍可在交互式 viewer/editor 中进一步视觉验收低角度 haze、跌倒后的 spotlight 边界、模型
   clip planes、命名相机随模型根移动，以及连续加载不同模型。

## 验证

| 入口 | 判据 | 结果 |
|---|---|---|
| `make check` | lint、format、CPU 与 integration | 536 + 44 passed |
| `make docs-check` | 公开 API、examples、strict MkDocs build | 15 个 API 模块通过；全部 API 页面已进入 nav |
| Forge/WGPU `test_horizon_haze.py` | skybox depth、haze 色带、透明几何 depth | 各 3 passed |
| 514×699 humanoid parity | 5 视角、同参考相机 | PASS；mean edge IoU 0.448，block luma 1.0，63/63 texture cells |
| 36.5 s 跌倒姿态对照 | 同一 `MjData` 与同一相机 | spotlight 和 floor footprint 位置一致；平均通道误差 1.09 |
| interactive clip plane 回归 | 交互入口沿用 adapter camera hint | 1 passed；`near/far` 与模型完全一致 |
| 模型根 gizmo/camera 跟随回归 | `SetPose(model node)` 后 body 与相机同步变换 | 1 passed；另以真实 humanoid 验证 back/side eye/target |
| paused light gizmo | viewer/editor 连续刷新不回弹 | `top` 灯两入口均保持 `[0.35, -0.2, 2.15]` |
| spotlight cutoff | Forge/WGPU 小角度回归与真实入口截图 | 45 个双后端 shading 用例通过；无棋盘锯齿 |
| 完整 MuJoCo physics/audit/conformance | physics、严格审计、deformables | GLFW 路径 220 passed；审计与 conformance 通过 |
| Forge GPU | 逐文件真实 OpenGL 回归 | 全部通过；包含 15 个 ID/outline 用例 |
| WGPU GPU | 完整后端回归 | 全部通过 |
| reverse verification | 注册回归变异必须被测试捕获 | 50/50 通过 |
