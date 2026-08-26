# 02 运行时组合与帧流

这章用于排查 viewer 构造、每帧更新、重复上传、退出释放和嵌入式 `sync()` 行为。

## 构造链

`composition.build*()` 选择 adapter 和 render backend，创建 window、backend、DebugBridge、Session、
ViewerApp，最后返回组合根 `Viewer`。

- **已核对**：`build(asset, backend_name="mujoco")` 中 `backend_name` 实际选择 scene adapter；渲染端由
  `FORGE_VIEWER_BACKEND=forge|wgpu` 选择。
- **观察到**：这里的 `backend` 一词承担两种角色，是公共 API 的历史命名债，见 08/10 章。
- **已核对**：构造中间失败必须逆序释放已创建 window/backend/bridge/adapter；功能代码 `11b5beb` 已补此保护。

## 每帧主链

1. `ViewerApp.frame()` 收集输入与 gesture。
2. `Session.tick(needs, wall_dt)` 决定 simulation step，并从 adapter 请求 frame。
3. structure revision 变化时，Session 刷新 source 与 metadata，App 调 backend `set_scene`。
4. App 调 backend `update(frame)`，发布 selection/debug/gizmo。
5. backend `render()` 生成 viewport image，CameraPreview 用 peer backend 单独渲染。
6. window 合成 ImGui 与 viewport。

- **已核对**：长期推荐两阶段热路径 `update(frame); render()`。
- **已核对**：便利入口 `render(frame)` 的定义是“先 update 再 render”，Forge/WGPU 必须一致。
- **已核对**：`FrameNeeds` 由消费者合并，避免 adapter 每帧生成不需要的 contacts、deformables、diagnostics。

## 资源所有权

- `Viewer` 是组合根，负责调用 `ViewerApp.release()` 并关闭 native window。
- `ViewerApp` 负责 bridge、camera preview peer、backend 与 session。
- `Session` 负责 adapter；backend 负责自己的 GPU children。
- 公共 `Renderer` 独立拥有 adapter、backend 与可选 GL context。

**已核对**：审查前 `ViewerApp.run()` 与 CLI `finally` 重复释放，而 `Viewer.release()` 又漏掉 preview；当前
功能代码 `11b5beb` 改为显式、幂等的组合根释放，并实现 `Viewer` context manager。

## 排障抓手

- 同一 frame 上传两次：先查 App/harness 是否同时 `update(frame)` 与 `render(frame)`。
- 退出后 GPU 增长：检查 backend 是否在“无 GL context”的 WGPU 路径也被 release。
- `sync()` 后残留 peer/dialog：检查是否经 `Viewer.release()` → `ViewerApp.release()`。
- 新场景继承旧 step/selection：检查 `_pause_loaded_scene()` 与 `_refresh_structure()` 的完整状态重置。
