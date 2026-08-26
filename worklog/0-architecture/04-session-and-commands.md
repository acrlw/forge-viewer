# 04 Session、命令与状态机

这章用于修改控制命令、模拟调度、选择、undo/redo 或场景替换行为。

## Session 的职责

- 持有 paused/speed/step、selection、camera、perturb、authored override。
- 缓存 source、frame 与 adapter metadata，并维护 structure generation。
- 将类型化 `commands.py` 命令路由到 capability 对应的 adapter 方法。
- 管理 document revision、undo/redo 和保存状态。
- 对 renderer/UI 暴露 backend-neutral query 与 bounds。

## 命令路由约束

典型顺序是 capability 检查 → 状态前置条件 → ID 解析 → adapter 调用 → 本地缓存同步 →
`CommandResult`。失败应返回 `CommandResult.bad`，不应让基类 unsupported 异常穿透 UI。

- **已核对**：stable keyframe/equality ID 必须先映射 metadata slot，传 adapter 时仍传 ID。
- **已核对**：`Step` 只接受正整数；simulation speed 必须有限且为正。
- **建议**：新增 capability 前先证明当前输入能触发对应 command，避免加永远到不了的保护分支。

## 模拟调度

- 非 external clock：Session 按 wall time、speed、adapter timestep 累积步数。
- paused：只消费显式 pending steps。
- external clock：adapter frame 的 `paused/step` 是事实来源，Session 不自行 step。
- 新建、打开、reload、load 后需清 step、pending、time credit、perturb、active keyframe，并进入可编辑暂停态。

## 选择

- object selection 以 `object_id` 为事实来源；structure refresh 时总要重新解析当前 node。
- object ID 0 的纯层级 node selection 可按 node ID 保留，但 ID 被别类节点复用时必须清除。
- **已核对**：只验证旧 node ID 是否仍存在会把已删除对象的选择串到复用 node ID 的环境节点。

## Authored override

当 adapter 不能或不应立即持久化编辑时，Session 可在 source/frame 上维护 override。判断集中于
`_preserve_authored_override()`，external clock 与 model composition 会影响策略。

- **观察到**：Session 会原地修改 adapter 返回的 source；这是当前所有权约定，不是纯 DTO 流。
- **建议**：修改 override 时同时核对 structure refresh、undo/redo、remote snapshot 和 save/export 四条路径。
