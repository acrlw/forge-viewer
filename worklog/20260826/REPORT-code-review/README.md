# forge-viewer 代码评审：API 一致性、封装、潜在 Bug 与双后端管线

**状态**：静态审计完成，关键 Bug 已数值复现，未做代码修改
**日期**：2026-08-26
**范围**：`src/forge_viewer/`（146 个 Python 文件，约 2 万行）、两后端 shader、公开文档
**方法**：4 路并行源码审计（API 面 / 架构封装 / Bug 猎查 / 双后端管线），另对 venv 内 mujoco 3.11 的
`mujoco.Renderer` 做了逐方法比对。未运行交互窗口；所有 GPU 行为结论来自源码阅读。

## 核心结论

1. `renderer.enable_depth_rendering()` 这类命名**不是前后不一致，也不建议改成 `set_*`**：它逐方法复刻
   `mujoco.Renderer` 的公开 API（已验证 mujoco 3.11 方法集完全一致），目的是 drop-in 兼容。这是单一来源的
   刻意设计，README「MuJoCo-style offscreen workflow」与此一致。
2. 项目整体**没有系统性过度封装**，分层边界由 `tests/test_layering.py` 强制且当前干净。三个「scene」表示
   （`Scene` → `SceneSource`/`SceneFrame` → `RenderScene`）是转换管线而非重复建设。
3. 发现 **2 个高置信度、用户可见的 Bug**（RPC 相机方位角差 180°；wgpu 后端 close 不释放 GPU 资源），均为两行
   级修复，建议优先处理。
4. 最集中的设计问题在三处：`RenderBackend` 协议与真实消费不符、`light_id`/`material_id` 的 index 与
   stable ID 混用、`Scene` 修改器的错误惯例不统一。
5. 双后端存在约 7 处近乎逐字复制的 Python 模块与 4 处以上重复常量，git 历史证实修复需手工镜像两遍；
   另有若干未写入文档的 parity gap，`docs/RENDERER.md` 有三处已过时。

---

## 一、高置信度 Bug

### B1. RPC `capture` 相机方位角差 180°，从模型背面渲染【已复现】

`src/forge_viewer/control_rpc.py:485`：

```python
camera.azimuth = float(np.degrees(np.arctan2(direction[1], direction[0])))
```

MuJoCo 自由相机的 azimuth 参数化的是**位置角 + 180°**（等价于视线方向），此处却用 `eye - target` 偏移角。
数值复现（经 `mjv_updateScene` 往返）：原始 az/el = (100, -30)，转换后 = (-80, -30)，相机位置与朝向
均关于 lookat 点镜像。后果：用户在交互视图里取好景，经 control RPC `capture` 出图时相机在模型另一侧。
对称测试模型会掩盖此问题；`tests/test_control_rpc.py` 未覆盖 `capture` 路径。
**修复**：`azimuth = degrees(arctan2(direction[1], direction[0])) + 180.0`，补一条 round-trip 回归测试。

### B2. `Renderer.close()` 永远不释放 wgpu 后端，GPU 资源泄漏【已复现】

`src/forge_viewer/renderer.py:429-431`：

```python
if self._context is not None and self._backend is not None:
    with self._gl_current():
        self._backend.release()
```

`_select_backend` 对 wgpu 返回 `context = None`（renderer.py:188），守卫把整个 `release()` 跳过。实测
`FORGE_VIEWER_BACKEND=wgpu` 下 close 后 `WgpuBackend.release()`（销毁 device buffer/纹理/管线，
webgpu/backend.py:1309-1329）完全未执行。`ControlService._capture_renderer`/`_drop_renderer`
（control_rpc.py:282-299）在每次 capture 尺寸变化或模型加载时关闭并重建 Renderer，每个循环泄漏一个完整
GPU 后端。`__init__` 异常路径（renderer.py:258-264）有同样漏洞。
**修复**：context-current 处理只对 GL 路径生效，`backend.release()` 不应被 `context is not None` 门控。

### B3. `math3d.inverse_orthographic_box` 不是 `ortho_box` 的逆（符号错误）【数学验证，当前无调用者】

`src/forge_viewer/math3d.py:272`：`m[2, 3] = (rf + rn) * 0.5` 应为 `-(rf + rn) * 0.5`。数值上
`inverse @ ortho` 的 `[2,3]` 项为 10.5 而非 0。当前 src/tests/examples 均无调用者，是文档声称「解析逆」的
休眠地雷——第一个未来调用者（如正交反投影）会得到静默错误的深度。旁边的 `inverse_perspective` 是正确的，
更增加了误信风险。**修复**：改符号或删除该函数。

### B4. 远端命令「超时」后仍会在之后执行【源码显示】

`src/forge_viewer/remote.py:232-237`：请求先入队 `_commands`，再 `wait(10.0)`；超时后向客户端报告失败，
但请求仍留在队列，后续 `pump_commands`（remote.py:186-200）照常执行——以 at-most-once 的语义表象提供
at-least-once 的行为。发布循环卡顿 >10s（调试器、`session.tick` 抖动）时，viewer 的 `reset`/`set_pose`
先报超时、后在不可预期的时间点生效。**修复**：超时即丢弃请求，或如实报告「已入队，可能延迟执行」。

### B5. `SnapshotPublisher.close()` 不关闭命令连接，存活 viewer 下次 `_send` 永久阻塞【源码显示】

remote.py:243-251 关闭两个 listener 与状态客户端，但 `_accept_commands`（remote.py:216-227）接受的
逐命令连接从未被跟踪或关闭，其处理线程阻塞在 `connection.recv()`（daemon 线程，进程退出时问题被掩盖）。
进程内 close publisher 而 `RemoteSceneAdapter` 仍存活时，adapter 下一次 `_send`（remote.py:392-398）的
`send` 成功、`recv` **永久阻塞**（`multiprocessing.Connection.recv` 无超时），UI 线程在暂停点击等操作上
冻结。`tools/remote_authoring.py:61-66` 展示了进程内 close 的用法。

### B6. wgpu 纹理 `srgb=False` 分支逻辑疑似写反，且 alpha 通道被线性化扭曲【源码显示，当前无触发路径】

`src/forge_viewer/render/webgpu/textures.py:95-100` `_format_for` 对 `srgb=False`（即**声明已是线性**）的
3/4 通道纹理返回 `linearize=True`，执行 CPU 线性化；`srgb=True` 反而走 GPU `rgba8unorm-srgb` 无 CPU 处理。
条件读取方向相反。此外 `_srgb_to_linear_u8`（textures.py:13-17）作用于拼接 alpha 之后的全部通道，
alpha=128 会被扭曲为约 55；forge 的 CPU 回退只线性化 RGB 并保留 alpha（forge/resources.py:65-70,113-115）。
当前无实际触发：`TextureData.srgb` 默认 True（types.py:301），唯一生产者 mujoco_adapter.py:3106 传 True。
**修复**：反转条件；线性化跳过 alpha 通道；补 `srgb=False` 的测试。

### B7. `euler_xyz_to_mat3` 是 extrinsic XYZ，与 MuJoCo XML `euler`（intrinsic XYZ）不一致【数值验证】

`src/forge_viewer/math3d.py:175-176` 计算 `Rz @ Ry @ Rx`（extrinsic XYZ），docstring 却写「intrinsic XYZ」，
与 MuJoCo 默认 eulerseq（intrinsic XYZ = `Rx @ Ry @ Rz`）不同。与 `mat3_to_euler_xyz` 自洽配对，Inspector
内部编辑稳定（tests/test_math3d.py:13 只测往返），但多轴旋转时 Inspector 显示/输入的角度与 MuJoCo 用户
在 XML 里写的 `euler` 属性对不上。至少 docstring 错误；是否改约定对齐 MuJoCo 是产品决策。

### B8. `set_camera_view` 无条件清零在线模型的 `cam_intrinsic`【源码显示】

`src/forge_viewer/adapters/mujoco_adapter.py:3610-3612`：对非 intrinsics 相机（`CameraView` 默认零值）也
无条件下发 `cam_intrinsic`/`cam_sensorsize`，会抹掉使用 `cam_intrinsic` 的模型的在线内参；spec 分支
（3598-3601）却只在 `uses_intrinsics()` 时写入——在线状态与 spec 分叉，重编译后内参静默恢复。

### B9. 轻微：`Renderer` 关闭后四个模式方法的错误类型不一致【源码显示】

renderer.py:392-411 的 `enable/disable_depth/segmentation_rendering` 未过 `_require_open`，close 后
`self._backend` 为 None，调用者得到 `AttributeError` 而非约定的 `RuntimeError`；
`disable_depth_rendering` 则静默 no-op。同类方法三种行为。

---

## 二、API 设计问题

### A1. `RenderBackend` 协议描述的不是真实契约【真实问题】

- 签名漂移：协议 `render(self, frame: SceneFrame)`（render/backend.py:188）要求位置参数；两个真实后端
  实现为 `frame=None` 可选（forge/backend.py:446、webgpu/backend.py:867），而 `NullBackend.render` 仍要求
  必传（backend.py:305）——`renderer.py:378` 以**无参**调用，任何按协议（或换成 NullBackend）的代码在
  包自己的调用点上崩溃。`@runtime_checkable` 只查存在性不查签名，挡不住。
- 协议缺失但被消费的方法：`set_background`（renderer.py:254 无条件调用）、`set_transparent_id_rendering`
  （renderer.py:396,406,411）、`.target` 属性（renderer.py:380-386、composition.py:100 及 tools/ 多处）、
  `capture(..., size=)`。`ui/app.py:1607` 用 `getattr(self.backend, "configure_text", None)` 防御，正是
  「不在契约里」的自白。第三方按协议实现的后端会在 Renderer 构造期直接崩溃。
- 单后端公开方法：forge 独有的 `mesh_triangle_counts()` 全仓零调用（死 API）；forge 独有的 `debug_view`
  property 与 `get_debug_view()` 双读路径。

### A2. `light_id`/`material_id` 在不同层含义不同：render index 与 stable ID 混用【真实问题，可致静默错改】

- `Scene.set_light(light_id)` docstring 写明是「zero-based **render index**」（scene.py:387-389），同类中
  `remove_light`/`light_value`/`set_light_by_id` 用的却是 **stable ID**。同名参数、同一类、两个索引空间，
  首次删除后即错位：持有一个 `SceneLight.light_id` 调 `set_light` 会改到错误的灯，且不抛异常。
- 同一歧义贯穿 `SceneAdapterBase.set_light`（adapters/base.py:820，按 index）、`commands.SetLight.light_id`
  （commands.py:258，由 scene_state.py:127 喂枚举 index）与真正 stable ID 的 `KeyframeInfo.keyframe_id`。

### A3. `Scene` 修改器惯例拼盘【真实问题，程度轻】

同一公开类三种失败语义：返回 bool（`set_camera`/`set_light`/`set_material`/`set_geometry_*`，其中
`set_geometry_size` 把 `ValueError` 吞成 `False`）、抛 `KeyError`（`remove*`/`object`/`rename_entity`）、
返回 None（`camera_view`）。`set_environment` 无条件返回 True（无意义 bool）。add 系列返回类型不一：
`add()`→handle、`add_camera()`→raw int、`add_texture()`→None。`Scene` 是教程门面，这是全包最用户面的 API。

### A4. `RenderFlag` 含 6 个与 `DebugView` 重复的死成员【真实问题，程度轻】

`SEGMENT/IDCOLOR/ALBEDO/NORMAL/OVERDRAW/DEPTH` 无任何后端接受（不在两端的 supported 集合），后四个全仓零
消费者。后果：设置面板里 SEGMENT/IDCOLOR 是永久禁用的复选框；CLI `--enable-render albedo` 静默无效——
四处调用点（cli.py:289,443,572,600）都不检查 `set_flag` 的 `False` 返回。`WIREFRAME` 同存于两枚举是
合理的，其余六个是残留。

### A5. 其他

- `Renderer.set_render_flag(name, enabled)`（mjRND_* 字符串）/ 后端 `set_flag(flag, value)` / CLI StrEnum
  三套词汇并存（mujoco 兼容层可以解释字符串形式）；docstring 写「for the next image」但 flag 实际粘性。
- `__init__.py` 导出不对称：CLI 自己用的 `composition.build` 与 `Viewer`、examples 直接 import 的
  `Session` 均未导出；docs/api/public.md 宣称 `__all__` 是受支持面。
- 后端选择逻辑在 renderer.py:177-200 与 composition.py:22-29 各自解析 `FORGE_VIEWER_BACKEND`（行为一致、
  代码重复）；`FORGE_VIEWER_GL` 只有 renderer.py 知道。
- 同一模块两种角度单位：`CameraView.fov_y` 弧度 vs `Light.cutoff` 度数（均有文档，是文档化的陷阱）。
- 命名小节：`FrameNeeds.actuator` 单数混入复数字段群；`CommandResult.good()/bad()` 非惯例构造名；
  `BackendCaps.gl_version` 字段名以 GL 为中心（wgpu 填 `"WebGPU {backend_type}"`）；
  `Perturb.mode` 字符串类型（commands.py:408）而同类概念均用枚举。

---

## 三、架构与封装评估

**总体判断：无系统性过度封装或封装不足，分层干净。** 已确认的良好设计（不建议动）：

- 三个「scene」表示是**转换管线**：`scene.py` 作者模型 → `SceneSource`/`SceneFrame` 交换格式 →
  `render/scene.py` GPU bucket 布局，各有多个生产者/消费者。
- `AdapterCaps` 能力位替代类型判断，在 Session 分发中一致使用；`adapters/conformance.py` 是任何第三方
  adapter 可跑的 226 行契约测试，`ToyPhysicsAdapter`（149 行）证明契约可被非 MuJoCo 后端实现。
- `FrameNeeds` 合并 + `SceneSourceBuilder.update` 的 buffer 复用遵守了 AGENTS.md 的热路径零分配规则。
- `renderer.py`（mujoco 兼容层）刻意放在 `render/` 之外以便合法 import mujoco——边界是设计而非偶然。

点状问题（按价值/改动量比排序）：

1. **死代码**：`backends.py:142` `make_backend_adapter()` 全仓零调用，与 10 行外的 `make_adapter` 命名
   近似，删除 5 行即可。
2. **生产死路径**：`render/builder.py` 的 `SceneSourceBuilder.set_visible`/`_overrides` 唯一调用者是
   `tests/gpu/test_horizon_haze.py:52`；生产可见性走 `cmd.SetVisible` → Session 改节点 → 全量 `set_scene`
   重传。要么接线为 eye-toggle 快路径（有真实性能收益），要么删除。维持第三条仅服务一个测试的可见性通道
   是最差选项。
3. **冗余镜像**：`SceneAdapterBase`（73 成员带默认实现，base.py:583-927）与 `SceneAdapter` Protocol
   （base.py:930-1036）手工双份声明，已验证今日完全同步但无任何机制防漂移。建议加约 15 行 AST 测试断言
   两者公开成员集一致。Protocol 是对外文档化表面，不应删除。
4. **封装不足**：`adapters/workspace.py:276-377` 摸 `Scene._cameras/_lights/_sync_light_items()`，因为
   Scene 公开 API 是 ID 制而 adapter 契约要 index 制——给 Scene 加 3-4 个公开 index 访问器即可消除。
   同理 `composition.py:315` 读 `app._viewport_image`，加一个 property。
5. **god object（可容忍，暂不拆）**：`MuJoCoAdapter` 4199 行约 140 个方法身兼三职（契约翻译、MJCF spec
   编辑、诊断覆盖层），但 import 边界干净（只依赖 types/math3d/base），`mujoco_deformables.py` 已有拆分
   先例。`Session` 1340 行是内聚的 mediator，49 分支 isinstance 链每分支带不同能力检查，换分发表收益
   有限。两者按 AGENTS.md「focused and compact」规则不建议现在拆；若编辑功能继续增长，可先把
   MJCF spec 编辑块（约 900 行）抽为 `mujoco_spec.py`。
6. 小节：`Session.bounds()`（session.py:1168-1200）35 行几何数学放错了层，下次触及时挪到 `SceneSource`
   旁；Session 原地改 adapter 持有的 `SceneSource`（session.py:975,1021,1230-1236），所有权模糊，值得
   加注释而非重构。`ui/app.py`(1666)/`ui/gizmo.py`(1475) 虽大但已通过约 10 个专职 controller 委派，
   剩余多为文件对话框管道，不值得搅动。
7. `tests/test_layering.py` 可补两条参数化规则：ui↛adapters.mujoco_adapter、session↛render。

---

## 四、双后端管线（forge/OpenGL 与 webgpu）

### 结构差异（可接受）

forge 有 pass registry 与 `PASS_ORDER`（shadow→reflect→opaque→id→skybox→tendon→transparent→outline→
debug→gizmo→present）；wgpu 无 registry，`render()` 硬编码顺序，id/depth 走主 pass 之后的单采样 export
MRT。用户可见结果等价。pass 级失败隔离不对称：forge 逐 pass 降级，wgpu 在 `__init__` 急切编译，
一个 WGSL 编译失败整个后端不可用。

### 重复代码与漂移风险（主要结构问题）

近乎逐字复制、纯 Python、可共享的模块：`cascades.py`（约 140 行，仅正交投影函数不同）、
`schedule_lights`/`LightSchedule`（forge/passes/base.py:15-63 vs webgpu/lighting.py:56-103，逐字，
且只有 forge 副本有测试）、`_publish_tendons`（约 115 行）、`TendonPass.update/_resize`（约 120 行）、
`ReflectPass.find_planes/_plane_equation/_encode_reflectance`、skybox/haze 顶点生成器、shadow 局部准备。
重复常量：`SHADOW_BIAS` 四处（两 Python 文件 + GLSL + WGSL）、outline 半径/颜色、`GHOST_ALPHA`、
`OVERDRAW_STEP` 等。git 历史（ceedc44、a4c3606、b721905）证实修复需手工镜像两遍。
已正确共享的对照：`TextLayout`、透明排序 `transparent_draw_order`。
另：wgpu 实例打包每次上传 `np.zeros` 新分配（webgpu/instances.py:51），forge 复用暂存 buffer——
违反 AGENTS.md 热路径规则。

### 未写入文档的 parity gap

- `stats.cpu_ms` 仅 forge 设置；`draw_calls`/`instances` 两端语义不同（计数口径不同）。
- wgpu 热重载只盯 `shadow_sample.wgsl`/`scene.wgsl` 两个文件，改 skybox/outline/haze 等 WGSL 不触发。
- capture：forge 存 RGBA，wgpu `[..., :3]` 丢 alpha。
- wgpu `render(frame)` 总是 `scene.update(frame)`，viewer 路径下每帧双重更新（幂等但浪费 CPU）。
- 深度回读：forge 含经典 skybox 圆柱与 tendon 深度，wgpu 的 export_depth 只写 scene bucket——
  tendon/sky 像素两端读数不同。
- overdraw 调试视图下 wgpu tendon 累积、forge 覆盖（仅诊断视图，可接受但未记录）。

### 文档过时

`docs/RENDERER.md` 三处：pass 顺序图把 present 放在 transparent 与 outline 之间（实际两端均最后）；
「MSAA fixed at construction in both backends」对 wgpu 已不成立（运行期可切）；timing 表两行均过时。
`docs/WGPU_BACKEND_REPORT.md` 本身准确，但未覆盖上述 gap。shader 侧 GLSL↔WGSL 为手工移植、头部互引，
抽查光照/IBL/outline/haze/spot_dist/present 逐行一致，GPU 测试经 `FORGE_VIEWER_BACKEND` 参数化两端跑——
这部分纪律是好的。

---

## 五、安全与加固（默认配置下风险低）

- `control_rpc.py:354-359` 的 AF_UNIX 控制 socket 未 `chmod 0o600`（bridge.py:374 有）。RPC 面含
  `capture` 任意输出路径与 `load` 任意路径，共享机器上构成本地跨用户文件写。
- `remote.py:37` 硬编码 `AUTHKEY` + pickle 传输：默认 localhost 无实际暴露；`serve --host 0.0.0.0`
  时等于把 unpickle（RCE 级）开放给网络。属文档/加固问题。

---

## 六、已核查排除项（非问题清单）

为避免误报，以下模式已逐类核查并排除：src/ 内无 TODO/FIXME/XXX 标记；无可变默认参数（dataclass 均用
`field(default_factory=...)`）；21 处 `except Exception` 均为合理回退或带日志；无永久 skip/xfail 隐藏
已知 bug（skip 均为环境条件）；forge GL 状态经 `GLStateGuard` + 每帧 `clear_main` 自校正，无泄漏；
`to_gl()` 转置、`look_at`、`_metric_depth`、透明排序、picking Y 翻转两端一致；两个真实后端均实现了
`RenderBackend` 协议的全部方法（存在性层面）；公开 API 零 camelCase。

## 七、建议处理顺序

| 优先级 | 事项 | 理由 |
|---|---|---|
| P0 | B1（RPC 相机 180°）、B2（wgpu release 泄漏） | 用户可见，两行级修复 + 回归测试 |
| P1 | B4/B5（remote 生命周期）、A1（协议补全 + NullBackend 签名）、A2（light_id 语义统一）、B6 | 语义陷阱与资源正确性 |
| P2 | A4（RenderFlag 死成员 + CLI 检查返回值）、A3（Scene 惯例统一）、共享 Python 模块上移、B3、RENDERER.md 三处更新 | 低成本去漂移 |
| P3 | B7 的 euler 约定决策、B8 内参语义、安全加固、MuJoCoAdapter 拆分 | 需要产品决策或更大窗口 |
