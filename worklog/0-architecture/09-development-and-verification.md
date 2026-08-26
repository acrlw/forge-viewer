# 09 开发、调试与验证

这章用于按改动风险选择验证范围，并给出仓库要求的最终门禁。

## 迭代原则

- 先用最小复现证明当前支持输入能触发问题，再补能在修复前失败的测试。
- CPU contract/纯函数先跑单文件或单测试；不要每次编辑都启动 GPU/golden。
- rendering 或 hot path 变更至少检查两 backend 的对应实现和调用点。
- 保存输出、截图和视频到 `output/`，不要写进源码目录。

## 定稿门禁

```bash
make check
```

渲染修改另跑：

```bash
make gpu
```

MuJoCo adapter 修改另跑：

```bash
.venv/bin/pytest -q -m physics
make mujoco-audit
make adapter-conformance ADAPTER=mujoco CONFORMANCE_ASSET=deformables
```

## Visual targets

`make outline`、`make gizmo`、`make gizmo-gallery`、`make perturb`、`make lighting`、
`make deformables`、`make showcase`。修改注册的 regression invariant 时还要 `make reverse` 并人工检查 gallery/
golden，不用文件哈希代替功能验收。

## 测试分层

- 默认 fast：纯合同、数学、builder、UI 控制流。
- integration：跨模块但不需要 MuJoCo/GPU。
- physics：MuJoCo model/spec/adapter；部分 GPU capture 同时带 physics marker。
- gpu/golden/slow：真实 context、图像与高负载验证。

## 当前环境记录

- **已核对（2026-08-26，功能代码 `11b5beb`）**：`make check` 为 575 fast + 60 integration 通过。
- **已核对**：`physics and not gpu` 为 162 通过；MuJoCo audit 与 deformables conformance 通过。
- **待确认**：当前容器 `eglInitialize failed (0x3001)`；`make gpu` 的首个失败发生在 context 初始化，
  未进入渲染断言，需在可用 EGL/GPU 环境重跑。
