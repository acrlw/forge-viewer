# 07 UI、相机与生命周期

这章用于交互、panel、camera preview、窗口集成和资源退出问题。

## ViewerApp 组成

ViewerApp 是协调器，实际工作分散到 OrbitCamera、CameraOut、CameraPreview、ObjectGizmo、ViewCube、
PerturbController、SceneEntityHelpers、GestureRouter 与 PanelSet。文件较大主要因为输入、对话框和主循环汇合，
不能仅凭行数判断需要拆分。

## 相机路径

- OrbitCamera 维护自由视角；CameraOut 向 backend/session 发布。
- model camera 由 stable `camera_id` 选择，source node 只携带 `camera_index` slot。
- CameraPreview 选择时做 slot → `CameraInfo.camera_id`，locked 状态保存 ID，pinned 状态保存 view 副本。
- Bookmark 必须复制 eye/target/up 以及 focal/sensor/principal intrinsics。

## 交互身份

pick 返回 object ID；Session 解析 node。Gizmo 对 camera/light 的 frame/body transform 另行处理，不能把
`body_index` 当选择身份。hierarchy 可见性修改 stable node，renderer structure 在 generation 变化后刷新。

## 生命周期规则

- `ViewerApp.run()` 只运行循环，不隐式销毁组合对象。
- `Viewer.release()` 与 context manager 是用户级终点，必须幂等。
- App 清 preview peer、dialogs、bridge、backend、session；Viewer 最后关闭 window。
- CLI/example 即使在 `finally` 再 release，也只能触发一次真实释放。

## UI 变更验证

- 纯控制流优先 CPU/AST/计数 fake 测试。
- 相机、gizmo、outline、lighting 等用户可见修改使用对应 Make visual target。
- 真正 window 输入与双 backend 交互由 `tests/gpu/test_ui_interaction.py` 等 GPU 测试覆盖。
