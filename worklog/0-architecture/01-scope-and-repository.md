# 01 定位与仓库结构

这章回答 forge-viewer 负责什么、哪些概念属于它，以及第一次找代码应从哪里开始。

## 项目边界

- **已核对**：forge-viewer 是 backend-neutral viewer；Forge 和 WGPU 是 render backend，MuJoCo、静态
  `Scene`、Toy physics、Remote 等是 scene adapter/source。
- **已核对**：物理状态归 adapter，应用状态归 `Session`，相机/灯/材质与作者元数据归 scene entity。
- **已核对**：render 层只依赖共享契约，依赖边界由 `tests/test_layering.py` 执行检查。
- **观察到**：项目同时服务交互 viewer、离屏 MuJoCo-compatible Renderer、CLI capture/audit、远程发布和
  程序化场景，因此公共契约比单一 GUI 项目更重要。

## 目录地图

| 路径 | 责任 |
|---|---|
| `src/forge_viewer/adapters/` | source/physics 到共享 scene 契约的翻译 |
| `src/forge_viewer/render/` | 共享 builder、render scene 和两套 GPU backend |
| `src/forge_viewer/ui/` | window、ViewerApp、panel、gesture、gizmo、preview |
| `src/forge_viewer/tools/` | 视觉验收、审计与专用 harness |
| `assets/` | MuJoCo/URDF/场景验证资产 |
| `tests/` | CPU、integration、physics、GPU、golden 分层测试 |
| `docs/` | 用户与 renderer 文档；需结合风险章判断是否漂移 |
| `worklog/` | 不进普通提交的调查、实验和长期架构入口 |

## 统一术语

- **已核对**：scene source 是稳定结构，scene frame 是动态数据。
- **已核对**：object ID 是选择身份；body index 是物理查找；camera/light index 是 source slot。
- **已核对**：render pass 是一个命名 renderer stage；render flag 是 renderer feature switch。
- **建议**：评审中看到裸 `id`、`index` 或 `backend` 时，应要求说明它属于哪一层；这些模糊名是当前最常见
  的跨层错误来源。

## 技术栈

- Python 3.11+、NumPy；MuJoCo 和 WGPU 为可选依赖。
- Forge 路径基于 OpenGL/moderngl/glfw；WGPU 路径使用 `wgpu`。
- imgui-bundle 提供交互 UI；pytest、ruff 与 Makefile 组成主要本地门禁。

## 不应误判为过度封装的部分

- `Scene`、`SceneSource/SceneFrame`、`RenderScene` 分别是作者模型、交换契约和 GPU bucket 布局，生命周期与
  消费者不同，不是三个同义 DTO。
- `Session` 作为 mediator 避免 UI/render 直接依赖 MuJoCo，属于项目“后端中立”目标的必要层。
- 两个 render backend 保留独立底层实现是 API 差异的结果；应共享算法与契约，不应机械共享每个类。
