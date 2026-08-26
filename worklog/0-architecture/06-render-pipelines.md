# 06 Forge 与 WGPU 渲染管线

这章用于修改 pass、shader、纹理或 capture，并判断两 backend 的差异是合理实现差异还是契约漂移。

## 共享层

`SceneSourceBuilder` 把 source/frame 转成 backend 可上传的 `RenderScene`，共享 mesh vocabulary、material、
transparent ordering、overlay 数据与多数数学函数。backend 接口负责 scene、frame、camera、target 和 flags。

- **已核对**：热路径复用 staging buffer；新增每帧数组时需检查临时分配。
- **已核对**：`render(frame)` 是便利入口，传 frame 必须消费；主循环用 `update(frame); render()`。

## Forge

命名 pass 顺序由 registry 固定：shadow → reflect → opaque → id → skybox → tendon → transparent →
outline → debug → gizmo → present。pass 可按能力或加载失败降级，OpenGL 状态由 guard 管理。

## WGPU

WGPU backend 在 render 中显式编码同等阶段，利用 render/compute pipeline 与 bind group；部分 id/depth 输出
结合 MRT/export。初始化期 shader/pipeline 失败通常使整个 backend 失败，而非逐 pass 降级。

## 必须保持一致的可观察行为

- source/frame/camera 的消费时点与 scene revision。
- object ID、depth、image orientation 和 capture 尺寸语义。
- render flag 是否接受及返回值。
- texture `srgb`：true 表示像素是 sRGB 编码，硬件只解码 RGB；false 表示数据已经在线性空间，不能再次解码。
- release 必须覆盖有/无独立 GL context 的 backend。

## 可以保留的实现差异

- registry 与显式 encoder 的组织方式。
- GL state guard 与 WGPU immutable pipeline/bind group 生命周期。
- MSAA resolve、target readback 和 shader language 的 API 细节。

## 漂移热点

- cascades、light scheduling、tendon、reflection、skybox/haze 与 outline 常量在两端存在近似复制。
- CPU/GPU timing 与 stats 口径不完全一致。
- shader hot reload 覆盖文件集合不同。
- capture alpha、tendon/sky depth 和诊断 overdraw 的 parity 需用 GPU/golden 证据确认。
