# 运行时替换模型后 viewer 复用旧场景纹理

状态：修复并验证完成，待提交
结论：GPU 纹理缓存只按名称复用资源，导致 direct viewer 加载新模型时继续采样旧场景的同名纹理；Forge 与 WGPU 现按当前 `TextureData` 来源替换同名资源。
涉及范围：运行时模型加载、Forge/WGPU texture stores、viewer/editor 渲染一致性
记录日期：2026-08-25

## 背景与表现

`make viewer` 先渲染默认 `test_scene`，再通过 File/Open 加载 MuJoCo 3.10.0 的
`humanoid.xml`；`make editor` 从空 Workspace 加载同一文件。两边均暂停在 t=0、step 0，
但 viewer 地板为灰黑色，editor 地板为正确的蓝黑色。

- 上游文件：MuJoCo 3.10.0 `model/humanoid/humanoid.xml`
- 本地使用路径：`/home/oem/下载/mujoco-3.10.0-linux-x86_64/mujoco-3.10.0/model/humanoid/humanoid.xml`
- 实际入口：direct `MuJoCoAdapter` runtime replacement 与 `WorkspaceAdapter` runtime loading

## 复现

关键条件是旧场景必须先完成 GPU 构建和至少一帧渲染，再加载新模型。首帧前直接替换模型不会触发。

修复前在 640×480 Forge render target 上对比：

| 判据 | 实测结果 |
|---|---|
| 最大通道差值 | 29 |
| 平均通道绝对差值 | 4.2726 |
| 发生变化的像素 | 302,611 / 307,200 |

SceneSource、相机、场景边界、灯光和 render flags 均一致，差异只在已上传的 GPU 纹理内容。

## 原因

`TextureStore.sync()` 把纹理名称当成跨场景永久缓存键。`test_scene` 与 humanoid 都包含
`grid` 等名称；旧实现发现名称已存在后直接跳过上传。editor 的 Workspace 会为模型资源增加
`forge_1_` 前缀，因此意外避开名称冲突，而 direct viewer 继续采样旧 checker 纹理。

同一缺陷存在于：

- `src/forge_viewer/render/forge/resources.py::TextureStore`
- `src/forge_viewer/render/webgpu/textures.py::TextureStore`

## 修改

两套 texture store 现在保存名称对应的 `TextureData` 来源对象。名称相同但来源变化时：

1. 移除旧 2D/cube 资源；
2. 从当前 SceneSource 重新上传像素和 mip levels；
3. 正确处理同名纹理在 2D 与 cube 类型之间切换；
4. 场景移除纹理或 backend release 时同步清理来源表。

GPU 回归先渲染 `test_scene`，再分别通过 direct viewer 和 editor 加载同一目标模型，比较相机、
场景边界与最终图像。该顺序覆盖用户实际入口。

## 结果

| 验证入口 | 输入与判据 | 实测结果 |
|---|---|---|
| 外部 humanoid direct/editor 对照 | 先渲染旧场景；640×480；最终图像一致 | 最大差值 0，逐像素相同 |
| Forge `test_model_loading.py` | 包含持久 backend 的多模型、多纹理类型替换序列 | 10 passed |
| WGPU `test_model_loading.py` | 同一序列 | 10 passed |
| `make check` | 静态检查、格式、CPU 与 integration | 534 + 44 passed |
| MuJoCo physics 与审计 | physics、strict audit、deformables conformance | 217 passed；审计与 conformance 通过 |
| Forge/WGPU GPU | 完整文件循环与受影响文件 | 除两个既有 driver 假设测试外通过；WGPU 完整通过 |

Forge 的完整 `make gpu` 在 `tests/gpu/test_id_outline.py` 遇到两个与本次纹理修改无关的
driver 假设：当前驱动支持 shared integer MSAA，且裁剪测试的选中像素不落在首列。排除该
文件后所有 GPU 文件通过；纹理替换与 haze 相关文件在 Forge/WGPU 均单独通过。
