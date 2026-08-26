# forge-viewer 独立代码审查、交叉复核与修复报告

> 状态：已先完成并落盘独立审查，随后阅读已有 agent code review，完成交叉复核、
> 明确 bug 修复与 CPU/MuJoCo 验证。
>
> 独立审查基线：`0a8178b47f8c5a3249d6b28428981da1d4165a5c`
>
> 最终功能代码快照：`11b5beb`（包含远端 `55bb83d`）
>
> 审查日期：2026-08-26
>
> 方法隔离：本文“一分钟结论”和“已确认的功能问题”最初版本在打开
> `REPORT-code-review/README.md` 前写成；后续比较与修复单列，不倒灌为独立发现。

## 一分钟结论

forge-viewer 的总体架构方向是成立的：稳定结构 `SceneSource`、动态数据 `SceneFrame`、应用状态
`Session`、物理 adapter 与 render backend 之间的边界清楚；Forge 与 WGPU 共用 builder、overlay
和共享数据契约，也有专门的 layering/conformance 测试。项目不是“整体过度封装”，相反，多数渲染
复杂度被放在了合理的位置。

目前真正需要优先处理的不是把所有接口改名或大规模拆层，而是若干跨边界契约没有完全落地：

1. 文档承诺的稳定 ID 在 Keyframe、Equality、Camera Preview、Workspace 中被当成数组槽位。
2. `RenderBackend.update(frame)` 与 `render(frame)` 同时存在，但 Forge 和 WGPU 对后者的语义不同，
   生产路径因此在 WGPU 上重复更新。
3. `ViewerApp`、`Viewer` 与公共 `Renderer` 的资源所有权不一致，既有重复释放，也有 WGPU backend
   完全未释放的路径。
4. Remote adapter 的 capability 声明超过了实际传输能力，且新 structure 可以与旧 frame 配对。
5. 场景替换、选择刷新和相机书签存在可直接复现的状态残留或数据丢失。

`Renderer.enable_depth_rendering()` 不属于应当优先重命名的问题。这个类明确兼容
`mujoco.Renderer`，本机 MuJoCo 3.11.0 的方法名和签名正是
`enable_depth_rendering()` / `disable_depth_rendering()`、
`enable_segmentation_rendering()` / `disable_segmentation_rendering()`。内部 backend 已经使用
`set_flag(flag, bool)`；兼容 facade 与内部通用接口采用不同命名是有意的边界，而不是风格漂移。

## 最终验证状态

- `make check`：575 个 fast 测试、60 个 integration 测试通过。
- `.venv/bin/pytest -q -m "physics and not gpu"`：162 个测试通过，1 条既有 MuJoCo flex warning。
- `make mujoco-audit`：严格审计通过。
- `make adapter-conformance ADAPTER=mujoco CONFORMANCE_ASSET=deformables`：全部检查通过。
- `make gpu`：同样在首个 EGL context 初始化处失败。当前环境无法给出真实 GPU/golden 结论，需在可用
  EGL/GPU 环境复核。

## 对已有 code review 的批判性交叉复核

### 判断正确且已采纳

| 原报告项 | 复核结果 | 处理 |
|---|---|---|
| `enable_*_rendering` 是 MuJoCo 兼容面 | 正确；本地 3.11 与官方 main 均保持成对 enable/disable | 保留名字，补 close 后一致错误 |
| B1 RPC 相机 azimuth 差 180° | 数值往返确认 | 加 180° 与真实 `mjv_updateScene` 回归 |
| B2 WGPU Renderer release 漏洞 | 计数 fake 确认，初始化失败也会漏 | 无 context 也 release，异常路径清理 |
| B3 正交逆矩阵 z translation 符号 | 乘积不为单位阵 | 修符号并测双向乘积 |
| B4 超时命令之后仍执行 | 队列语义确认 | request cancellation 与回归 |
| B5 publisher close 不关 command clients | 连接生命周期确认 | 跟踪/关闭连接并验证解除阻塞 |
| B6 WGPU `srgb=False` 被二次解码 | `_format_for` 数值确认 | 线性 RGB(A) 使用 unorm、不做 CPU 解码 |
| B7 Euler docstring 与实现不符 | 实现是 extrinsic XYZ；往返自洽 | 修文档，不改变 UI 数学语义 |
| B9 Renderer 模式方法 close 后错误不一致 | 确认 | 四个方法统一 `_require_open` |
| A1 `RenderBackend.render` 协议漂移 | 两 backend、Null 与生产调用确有差异 | 可选 frame；两端传入即 update，主循环无参 render |
| Control RPC socket 权限 | 共享机器上成立 | 创建后 `chmod 0600` 并回归 |

### 需要修正或收窄的结论

1. **B8 的现象对、修复方向需反过来。** `CameraView` 的契约是：无正物理内参就使用 `fov_y`。调用者明确
   传零内参时，在线模型清零是正确的模式切换；真正 bug 是 editable spec 保留旧内参，导致下一次 compile
   又恢复物理投影。本次修复让 spec 同步清除 focal/sensor/principal 的 length/pixel 字段。
2. **B6 不是“当前无触发路径”。** MuJoCo producer 默认传 `srgb=True`，但 `TextureData.srgb=False` 是公开
   支持输入且能由程序化 Scene/scene I/O 进入 WGPU；不能因为唯一内置物理 producer 未触发就降为休眠问题。
3. **B7 不等于必须对齐 MuJoCo XML Euler。** helper 服务 Inspector 的矩阵分解/重组且自洽；改变乘法顺序会
   改已有 UI 数值语义。当前确定错误只有 docstring，是否给 XML 输入另设转换函数属于产品决策。
4. **“只有两个用户可见高置信度 bug”明显过窄。** 动态最小复现还发现 selection 串位、sparse stable ID
   拒绝、camera preview 错位、Viewer sync 泄漏、Remote capability 异常与 structure/old-frame 配对等；这些
   比多项静态风格问题更应优先。
5. **双后端重复与 parity gap 是有价值线索，但不能全部按已证实 bug 处理。** stats 口径、depth 内容、pass
   失败策略和 hot reload 覆盖需要真实 GPU/benchmark 或明确规范；当前 EGL 环境无法完成该证据闭环。

### 原报告遗漏、由独立审查补出的关键问题

- Keyframe/Equality stable ID 被当 slot。
- CameraPreview 把 camera slot 当 ID。
- structure refresh 将已删除对象选择串到复用 node。
- Workspace authored camera/light ID 删除后失效，稀疏 primary node ID 合并错误。
- Remote capability 误报、初始无相机时永久禁用 camera、structure 与旧 frame 拼接。
- ViewerApp/Viewer 重复释放与 `sync()` 清理遗漏；文档声称 context manager 但未实现。
- New/Open step counter 残留、bookmark 丢物理内参、Step/Speed 非法值行为矛盾。
- Forge/WGPU `render(frame)` 语义不同，WGPU 主循环重复 update。

## 已实施修复

1. 修正 RPC camera、正交逆矩阵、Euler 文档和 camera bookmark 内参。
2. 修正公共 Renderer 无 context/初始化失败释放，并统一 close 后错误。
3. 统一 render convenience contract 与 App/harness 两阶段调用。
4. 修正 WGPU 线性 RGB(A) texture format 选择。
5. 修正 Session sparse control IDs、selection refresh、场景替换状态、Step/Speed 校验，以及 Viewer.record
   的整数帧数/有限正帧率校验。
6. 修正 CameraPreview stable ID 跟踪。
7. 建立 Viewer 单一、幂等生命周期与构造失败清理。
8. 修正 Remote capability、authoring command、revision/frame 配对、timeout cancellation 和连接关闭。
9. 修正 Workspace authored camera/light 稳定命名空间及 sparse node graph 合并。
10. 同步清除 MuJoCo online/spec camera intrinsics，并收紧 control socket 为 `0600`。

每组都有修复前可失败的最小回归；未修改另一份 agent 报告正文。

> 下文“已确认的功能问题”保留独立审查落盘时的证据和建议措辞，用于审计发现来源；其中前述
> “已实施修复”覆盖的条目已经不再是当前工作树缺陷。

## 已确认的功能问题

### 1. 公共 Renderer 在 WGPU 模式下不释放 backend（高）

证据：`src/forge_viewer/renderer.py` 的 `close()` 只有在 `_context is not None` 时才调用
`_backend.release()`；而 `_select_backend()` 明确对 WGPU 返回 `(None, WgpuBackend(...))`。

最小复现通过 `Renderer.__new__` 注入计数 backend：调用 `close()` 后 adapter release 次数为 1，
backend release 次数为 0。

影响：离屏批量渲染、模型审计或反复创建 WGPU Renderer 时，GPU target、buffer、pipeline 等资源
依赖 GC/进程退出回收。

建议：backend 的存在与 GL context 的存在分开判断；初始化失败路径也应释放已经创建的 WGPU
backend。

### 2. Viewer 生命周期同时存在重复释放和遗漏释放（高）

证据：

- `ViewerApp.run()` 在循环结束后调用 `self.release()`。
- CLI 和所有主要示例又在 `finally` 中调用 `viewer.release()`。
- `Viewer.release()` 直接释放 bridge/backend/session/window，却不调用 `app.release()`；因此只使用
  `sync()` 后释放 Viewer 时，CameraPreview peer 和文件对话框不经过 App 的清理路径。
- backend 的 `release()` 没有统一的幂等保护。

建议：确定单一所有者。较自然的方案是 `Viewer` 为组合根：`ViewerApp.run()` 不隐式释放，
`Viewer.release()` 幂等地调用 `app.release()` 后关闭 window，并实现文档已经声称存在的
`__enter__` / `__exit__`。

### 3. Remote capability 与实现不一致，合法命令可直接抛异常（高）

`RemoteSceneAdapter` 从 publisher 复制大部分 capability，但没有传输 `new_scene/open_scene/save_scene`
等方法。以空的 `StaticSceneAdapter` 发布后，remote caps 仍包含 `scene_files=True`；
`Session.submit(NewScene())` 因而进入该路径，随后调用基类的 unsupported 实现并抛出
`RuntimeError`，而不是返回 `CommandResult.bad`。

同类不一致包括：

- `asset_loading`、`scene_files`、`state_snapshots` 可能被保留，但 Remote 没有对应实现。
- `scene_authoring=True` 时，Remote 缺少 geometry size、generic duplicate/remove/rename 的传输方法。
- `model_cameras` 被改成 `bool(initial cameras)`；初始没有相机的可编辑场景会永久失去相机能力，
  即使之后远程新增了相机。

建议：先把不支持的 capability 明确 mask 掉；保留源 adapter 的 `model_cameras` 能力；补齐
`scene_authoring` 所承诺的命令，或把这一粗粒度能力拆细。

### 4. Remote 新 structure 会继续返回旧 frame（高）

最小复现：发布 1 个实例的 structure/frame，随后只发布 2 个实例的新 structure。Remote 已观察到
新 revision 后，`scene_source().instance_count == 2`，但 `frame().geom_xpos` 仍只有 1 行。

根因：`RemoteStructure` 到达时 `_latest` 没有失效；`RemoteFrame` 也不携带 structure revision。

影响：`Session._refresh_structure()` 会立即把新 source 与旧 frame 交给 builder，可能产生错误姿态、
数组裁剪或索引错配。

建议：structure 发布时丢弃 sender 中的旧 latest frame，receiver 收到 structure 时清空 `_latest`，
并等待该 structure 之后的首帧。更强的协议是让 frame 携带 structure revision 并在接收端校验。

### 5. Keyframe 和 Equality 的稳定 ID 被当成槽位（中高）

adapter conformance 只要求 ID 唯一，并不要求从 0 连续；`CameraInfo` 也已经按稳定 ID 正确映射。
但 Session 当前：

- `LoadKeyframe(42)` 先检查 `42 < len(keyframes)`，再用 `keyframes[42]`。
- `SetEqualityEnabled(7, ...)` 先检查 `7 < len(equalities)`，再用 `equalities[7]`。
- `restore_physics_state(active_keyframe=...)` 和 structure refresh 也用长度校验 active ID。

自定义 adapter 暴露单个 `KeyframeInfo(42, ...)` 与 `EqualityConstraintInfo(7, ...)` 时，两条合法命令
都被 Session 拒绝，adapter 从未收到调用。

建议：像 camera 一样建立 ID 到槽位的显式映射；adapter 调用传稳定 ID，frame 中的 equality 数组
仍按 metadata 顺序解释。

### 6. Camera Preview 把 camera slot 当成 camera ID（中高）

`SceneNode.camera_index` 是 source slot；`Session.camera_view()` 接受稳定 camera ID。Inspector 已正确
执行 `slot -> CameraInfo -> camera_id`，CameraPreview 却直接调用
`session.camera_view(node.camera_index)`，并把该 slot 用于 locked 状态。

复现：添加 camera 0 和 1，删除 0 后 camera 1 移到 slot 0。选择它时 preview 返回空；锁定预览也会
跟踪错误的相机。

建议：CameraPreview 存储稳定 camera ID；选择时通过 `session.cameras[node.camera_index]` 解析。

### 7. structure refresh 可把旧选择错误绑定到新节点（中高）

当前刷新只在旧 `selected_node_id` 不存在时才通过 object ID 重找。Scene 删除对象后会重建连续
node ID；旧对象的 node 1 可能被新建的 environment node 1 复用。

复现结果：删除已选择的 object ID 1 并 tick 后，`session.selected` 仍为 1，
`session.selected_node` 却变成 environment。

建议：当 `selected != 0` 时始终以稳定 object ID 为准重新解析；对象不存在则同时清空 object 与
node 选择。对 object ID 为 0 的层级子节点另行保留 node 选择策略。

### 8. Workspace 的 authored camera/light ID 在删除后失效（中高）

Workspace 同时混用了 source slot、Scene 稳定 ID 和“primary count + raw ID”：

- 两盏 authored light 返回 ID 0、1；删除 0 后，再用返回的 ID 1 删除第二盏会失败。
- 两个 authored camera 返回 ID 0、1；删除 0 后，`cameras()` 把剩余 camera 重新报告为 ID 0，
  原先返回的 ID 1 无法删除。
- primary 模型增删导致 primary camera/light 数变化时，offset 也会改变。
- `_light_to_scene` / `_camera_to_scene` 被写入但没有用于这些解析路径。

此外 `_merge_source()` 使用 `len(primary.nodes)-1` 计算 node offset，并用 `nodes[parent]` 把 node ID
当 list index。第三方 adapter 即使通过 conformance 的“唯一、父节点存在”检查，只要 node ID 稀疏
或 root ID 非 0，Workspace 仍可能生成无效层级或索引错误。

建议：这是需要先确认的设计点。长期应明确区分 `camera_id/light_id` 与 `camera_index/light_index`，
为 Workspace authored entity 分配不依赖 primary 数量的稳定命名空间；合并节点时使用显式
`node_id -> node` 映射和新 ID 分配器。

### 9. RenderBackend 的 render(frame) 在 Forge/WGPU 上语义不同（中）

当前实际行为：

- Forge：仅在 `_scene is None` 时消费传入 frame，已有 scene 后静默忽略。
- WGPU：只要传入 frame 就再次 `update(frame)`。
- ViewerApp 与工具 harness 已先调用 `backend.update(frame)`，随后又调用 `render(frame)`。
- CameraPreview 和公共 Renderer 则调用无参数 `render()`；这又与 Protocol 和 NullBackend 的必填
  参数签名不符。

因此同一生产帧在 Forge 更新一次、WGPU 更新两次；接口类型与真实调用也不一致。

建议：保留可选 `render(frame)` 作为一次性便利入口时，两个 backend 都应定义为“传入即 update”；
生产的两阶段路径统一改为 `update(frame); render()`。Protocol 与 NullBackend 同步为可选参数，并在
文档中明确两种调用方式。

### 10. 新建/打开场景没有清零 Session step counter（中）

新提交新增 `_pause_loaded_scene()`，会清 pending steps 和 time credit，但不清 `_step_counter`。
`Reload` 和 `LoadAsset` 在调用方单独清零；`NewScene` 与 `OpenScene` 没有。

复现：暂停后 step 3 次，执行 `NewScene`，下一次 tick 的新场景 `frame.step` 仍为 3。

建议：把完整的“场景替换后模拟状态”重置集中在一个 helper 中，并处理 adapter 拒绝 pause 的结果，
避免各命令各自遗漏字段。

### 11. Camera bookmark 丢失物理相机内参（中）

`CameraView` 支持 `focal_length`、`sensor_size`、`principal_offset`，MuJoCo camera adapter 也会填充和
写回这些字段；但 camera bookmark 只保存 fov/clip/ortho。

最小 round-trip 后 `uses_intrinsics()` 从 true 变成 false，三个内参数组归零。

建议：向 bookmark 增加可选内参字段，读取时对旧 version-1 文件使用零值默认；若格式策略要求严格
版本化，则升级版本并提供 v1 reader。

### 12. 边界输入会产生与结果不符的状态（低至中）

- `Step(count <= 0)` 实际至少排入 1 步，却报告原始 count，例如“Stepped -3 frame(s)”。
- `SetSpeed` 没有拒绝 NaN/Inf；NaN 会使后续步数计算失败。
- `Viewer.record(frames=0.5)` 先通过正数检查，再 `range(int(frames))` 执行 0 帧并返回一个未生成路径。
- `Viewer.record(size=...)` 会永久保留 fixed render size，没有恢复调用前尺寸。

公开类型注解降低了误用概率，但这些 API 已做运行时校验，应保证校验与实际执行一致。

## 设计债与风险，不应伪装成已确认 bug

### Adapter 接口过宽，capability 又过粗

`SceneAdapter`/`SceneAdapterBase` 同时容纳核心帧协议、模拟控制、物理诊断、场景文件、模型组合、
拓扑编辑和 authored entity 操作。默认 unsupported 实现让第三方接入容易起步，这是优点；但
`scene_authoring=True` 一个位同时承诺多组方法，Remote 的能力漂移说明它已开始超出一个布尔位能
表达的范围。

不建议立即拆成大量小类。更稳妥的演进是：保留小型 core adapter 协议，再为 simulation、camera、
authoring、composition 等形成可独立检查的 capability protocol/service；先从 Remote 与第三方
conformance 的真实失败驱动拆分。

### “backend” 一词承担了两种角色

`Viewer.backend` 是 render backend，但 `build(..., backend_name="mujoco")`、CLI `--backend` 与
`backends.py` 又把物理/adapter 组合称为 backend。项目架构文档已经使用 scene adapter 与 render
backend 的清晰术语，公共构造参数仍保留历史命名。

建议采用兼容迁移：新增 `adapter_name`，保留 `backend_name` 一段时间并发出弃用提示；渲染选择继续
使用明确的 `render_backend` 或环境变量。

### `release` 与 `close` 不是单纯命名风格问题

公共 Renderer 使用 `close()` 是 MuJoCo/Python 资源对象兼容；内部 GPU 对象使用 `release()`；window
和 socket 使用 `close()` 本身可以接受。当前问题是所有权与幂等性没有写清，而不是必须全仓统一成
一个动词。先修组合根的生命周期，再决定是否统一 facade。

### 两套 render backend 不应强求源码同构

Forge 的 pass registry 与 WGPU 的显式编码反映 API 差异；共享 `SceneSourceBuilder`、overlay、mesh
vocabulary 已经是合理复用。需要消除的是可观察语义差异，例如 `render(frame)`、核心 pass 加载失败
策略、MSAA 状态报告，而不是把两套底层实现机械抽成同一继承树。

### 可变 ndarray 与 frozen dataclass

`CameraView`、`Light`、`Material` 等声明为 frozen，但内部 ndarray 可原地修改；默认常量也包含可变
数组。这是一种“字段绑定不可变、数据不深拷贝”的高性能约定，不是 Python 意义上的完全不可变。
应在文档中准确描述，并对跨线程/缓存边界做防御性复制；不建议在热路径无差别深拷贝。

### pickle 是显式信任边界

Remote TCP 与 snapshot recording 都会反序列化 pickle。固定 authkey 只提供连接握手，不构成面对
不可信主机/文件的安全边界。若功能仅面向可信本机/局域网，应在 CLI 与文档明确；若要接受外部网络
或用户下载的 recording，应迁移到非可执行 schema，并限制非 loopback bind。

## 新提交 0a8178b 的单独评价

1. `WorkspaceAdapter.load()` 改为 `primary.load(target)` 是正确收敛：根模型加载不再绕行“空模型 +
   attach”，避免根模型语义被 composition 路径改变。
2. 默认 paused 与加载后 paused 的产品方向一致，尤其适合编辑器；但状态重置应集中，当前遗漏
   New/Open 的 step counter，且忽略 adapter 对 pause 的拒绝。
3. world-target camera/light 与 world-owned flex 的修复有定向测试，physics 与 deformables
   conformance 通过。`_FLEX_COPY_FIELDS` 是对 MuJoCo MjSpec 字段的显式镜像，后续 MuJoCo 升级时有
   漂移风险；模型审计新增 direct/workspace/composed 三路计数是合适的保护，但计数相等不能替代
   flex 拓扑、材质、owner 与 transform 的语义验收。

## 建议修复顺序

1. Renderer WGPU release、Viewer/App 单一所有权与幂等释放。
2. Session stable ID、CameraPreview ID、selection refresh、New/Open step reset。
3. Remote capability mask、完整 authoring 传输、structure/frame 同步。
4. 统一生产路径为 `update(frame); render()`，对齐两个 backend 的便利入口语义。
5. Camera bookmark 内参和边界输入校验。
6. 与维护者确认 Workspace 复合 ID 命名空间后，再修 camera/light/node translation。

前五组可以做聚焦的小修复和回归测试；第六组会影响第三方 adapter 与公开 command 返回值，应该先
讨论稳定 ID/slot 的最终术语与兼容策略。
