# 05 Adapter、Workspace 与 Remote

这章用于接新 scene/physics source、组合 authored 内容，或排查远程 capability 与结构帧不同步。

## Adapter 最小核心

第三方 adapter 至少提供 `scene_source()`、`frame(needs)`、`structure_revision`；`SceneAdapterBase` 为其余
操作提供 unsupported 默认值。`AdapterCaps` 决定 Session/UI 是否暴露某条能力。

- **已核对**：`check_adapter()` 校验实例列、node graph、camera/light slot、动态 frame 和 timing。
- **建议**：先以最小 caps 接入，再按真实可调用方法逐项打开；capability 为 true 就是功能承诺。

## 主要实现

- `MuJoCoAdapter`：模型/spec、运行态、visual metadata、诊断和 write-back 的主实现。
- `StaticSceneAdapter`：把程序化 `Scene` 暴露为完整 authored scene。
- `ToyPhysicsAdapter`：证明核心协议不依赖 MuJoCo。
- `WorkspaceAdapter`：合并 primary adapter 与 Forge-authored `Scene`。
- `RemoteSceneAdapter`：消费 publisher 的可靠 structure、latest-only frame 与独立 command channel。

## Workspace 组合

- source/frame 数组按 primary → authored 拼接；mesh/texture/material 需重映射避免名称或 key 冲突。
- authored object/camera/light 使用独立命名空间，不能依赖 primary 当前数量。
- node 合并必须用 `node_id -> node` 映射，不能用 `nodes[node_id]`；第三方 node ID 可以稀疏且 root 未必为 0。
- `camera_index/light_index` 仍是拼接后 slot；对外 camera/entity ID 保持稳定。

## Remote 协议

- structure 可靠有序发送，frame 只保留 latest；command 走独立往返连接。
- frame 必须携带 structure revision；receiver 只接受与当前 structure 匹配的 frame。
- 新 structure 到达时 sender 和 receiver 都应失效旧 latest frame。
- command 超时后若尚未执行必须标记取消；publisher close 必须关闭已接受的 command connection。
- Remote caps 必须 mask 未传输的 asset loading、scene files、state snapshots、composition/topology 等能力。

## 信任边界

- **已核对**：Remote 和 snapshot recording 使用 pickle；固定 authkey 只是连接握手，不把不可信输入变安全。
- **建议**：默认只绑定可信 loopback；若允许公网或下载 recording，迁移到非可执行 schema 并设计认证、版本和
  大小限制。
