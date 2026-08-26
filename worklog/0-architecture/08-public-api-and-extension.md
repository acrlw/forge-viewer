# 08 公共 API、命名与扩展面

这章用于评审接口命名、兼容性、第三方 adapter/backend 接入和弃用策略。

## MuJoCo-compatible Renderer

`forge_viewer.Renderer` 明确镜像 `mujoco.Renderer` 的构造、`update_scene()`、`render()`、depth/segmentation
模式和 context manager。

- **已核对**：MuJoCo 3.11 主线仍使用 `enable_depth_rendering()` / `disable_depth_rendering()` 与对应
  segmentation 方法；顶层模块保留 backwards-compatibility wrapper。
- **结论**：不要直接改成 `set_depth_rendering(bool)`。若需要新 API，可添加别名，但兼容 facade 必须保留原名。
- 内部 backend 通用开关使用 `set_flag(RenderFlag, bool)`，这是合理的层间不同命名。

## `close` 与 `release`

- Python/context-managed facade 与 socket/window 使用 `close()`。
- 内部 GPU/adapter 对象使用 `release()`。
- **结论**：问题核心是所有权、幂等和异常路径，不是全仓只准一个动词。

## Scene 与 Adapter API

`Scene` 面向程序化作者，Adapter 面向 Session。前者同时存在 stable entity handles 与 source index 编辑，
尤其 light API 容易混淆。

- **建议**：后续把 public stable `light_id` 与内部 `light_index` 明确拆名，并给 light metadata 一个类似
  `CameraInfo` 的结构；不要只改参数名而不改解析链。
- Scene mutator 的 bool/None/KeyError 惯例可在一个明确版本窗口统一，不应夹在 bugfix 中零散改变。

## 扩展顺序

1. 新 source 优先实现 `SceneAdapterBase` 最小核心并跑 conformance。
2. 新 render backend 先实现真实 App/Renderer 消费的完整协议，而非只满足 runtime-checkable 的成员存在性。
3. 公共构造参数变更采用新名 + 旧名弃用期，例如未来 `adapter_name` 替代含糊的 `backend_name`。
4. `__all__`、API 文档、examples 与测试必须同步，避免“内部可 import”被误当稳定支持。
