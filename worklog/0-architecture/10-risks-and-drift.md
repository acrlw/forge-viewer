# 10 风险与漂移

这章登记当前不能当作稳定事实的部分，以及依赖升级或功能扩展时最容易复发的契约漂移。

## 已知结构风险

1. **Adapter 过宽、capability 过粗（观察到）**：一个 `scene_authoring` 同时承诺多组方法；Remote 曾因此
   误报。先补基于真实能力的 conformance，再考虑 core + 可选 protocol/service，暂不一次拆成几十个类。
2. **ID 与 slot（已核对）**：camera/keyframe/equality/workspace/selection 已出现实际 bug。新增实体类型必须明确
   stable ID、source slot 与 physics index 三个空间。
3. **双 backend 漂移（观察到）**：共享算法仍有复制；每次修 Forge/WGPU 一端都搜索另一端和 shader 常量。
4. **RenderBackend 协议不完整（待确认）**：真实 Renderer/App 还消费 target、background、transparent-ID、
   capture size 等成员；需要一次独立接口设计，不宜只不断把方法塞进大 Protocol。
5. **frozen ndarray（观察到）**：跨线程/缓存可被原地修改；当前以性能约定维持，文档与边界 copy 必须明确。

## 依赖漂移

- MuJoCo `MjSpec` flex 字段由 `_FLEX_COPY_FIELDS` 显式镜像；升级 MuJoCo 时检查新增字段、owner、材质和 transform，
  计数相等不能替代语义验证。
- 官方 MuJoCo Renderer 兼容方法当前为 enable/disable；升级时以官方源码与本地 signature 双重核对。
- WGPU texture formats、backend type 与 shader compilation 行为随 `wgpu` 版本变化，必须跑真实 GPU。

## 安全边界

- Remote pickle + 固定 authkey 仅适合可信本机/网络；公开 bind 是高风险配置。
- Snapshot recording 同样不应读取不可信文件。
- control RPC 的 socket 权限、任意 load/capture 路径需要在多用户部署前单独加固。

## 文档与实现漂移

- `docs/RENDERER.md` 的 pass 顺序、MSAA 与 timing 说明曾被审查报告指出可能过时；迁移本笔记到正式 docs 前需
  逐项复核并修订。
- `backend_name` 同时被用户理解为 physics adapter 与 render backend；在弃用设计确认前不要静默改名。
- `Scene.set_light` 的 index 与 `SceneLight.light_id` 的 stable ID 语义仍不够直观，是下一轮 API 设计重点。

## 当前工作树状态

- 基线是 `0a8178b47f8c5a3249d6b28428981da1d4165a5c`；本次审查修复尚未提交。
- 已修的具体问题、测试和与另一份 review 的差异以
  [2026-08-26 独立审查](../20260826/REPORT-forge-viewer-independent-audit.md) 为准。
- 修复合入后更新本章与首页 commit；若修复被拆分或退回，不能把这里的“已核对工作树”误写成 main 事实。
