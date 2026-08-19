# wgpu Backend Completion Plan

Goal: `FORGE_VIEWER_BACKEND=wgpu` supports the full forge-viewer capability set — every
render flag, debug view, label/frame/bvh overlay, debug draw, gizmo, shadows, planar
reflections, skybox/IBL, tendons, and the interactive viewer window — with pixel parity
against the OpenGL (forge) backend and zero regression on the default path.

Status: living document on the `wgpu-py` branch. Milestones land as separate commits.

## Principles

1. **Mirror, don't invent.** `render/webgpu/` mirrors `render/forge/` structure: a
   `passes/` package (one module per pass, same names), `shaders/*.wgsl` mirroring the
   GLSL files, the same store class conventions (`sync/get/release`), the same
   PASS_ORDER. Where forge has GL-only infrastructure (`state_guard.py`, `gl_native.py`,
   shader hot reload, GL timer queries) the wgpu backend simply does without and reports
   it via `capabilities().notes`.
2. **Reuse the backend-agnostic layers unchanged.** `render/scene.py`, `builder.py`,
   `debugdraw.py`, `mesh.py`, top-level `gizmo.py`, `picking.py` are already shared.
   Two extractions are needed to avoid duplication:
   - `render/forge/text.py`: split the PIL glyph-atlas building (backend-agnostic) from
     the GL draw; both backends consume the atlas.
   - `ForgeBackend._publish_*` (labels/frames/bvh/contacts/joints/…, ~350 lines of pure
     CPU work writing into the `DebugDraw` store): extract into a shared publisher used
     by both backends.
3. **WGSL idioms for GL idioms.** WGSL has no preprocessor: `DEBUG_VIEW` / `WIREFRAME` /
   `USE_SHADOW` variants use pipeline-creation `constants` (WGSL `override`). No
   geometry shader: wireframe barycentrics come from a lazily-built vertex attribute.
   No `gl_ClipDistance`: reflection clipping is a fragment discard on a plane equation
   in the frame uniforms. No MSAA resolve for uint/depth: the existing single-sample
   export MRT pass remains the readback path. `gl_VertexID` → `@builtin(vertex_index)`,
   `flat`/`noperspective` → `@interpolate(flat/linear)` — both used by debug draw.
4. **Caps honesty.** Every milestone flips `BackendCaps`/`_SUPPORTED_FLAGS` only for
   what actually works and is covered by a test under `FORGE_VIEWER_BACKEND=wgpu`.
5. **Verification per milestone.** (a) the milestone's tests pass under wgpu;
   (b) `pytest -q` (495) and the per-file `make gpu` loop show the exact same results
   as on `main` (pre-existing failures: forge_core 2, id_outline 3, pipeline 1,
   model_loading 2 errors, ui_interaction 1+39 — do not chase); (c) `ruff check` clean.

## Milestones

### M1 — Backend-parameterized test infrastructure
- `tests/gpu/conftest.py`: `make_backend(...)` factory honoring `FORGE_VIEWER_BACKEND`;
  `backend_name` fixture; GL-internals test files (test_forge_core, test_id_outline,
  test_debugdraw_gpu, and the GL-specific parts of test_pipeline) get an explicit
  forge-only skip guard.
- `tools/_harness.py`: `OffscreenHarness` backend switch (mirrors
  `renderer._select_backend`).
- Makefile: `gpu-wgpu` target — per-file loop over the backend-neutral subset with
  `FORGE_VIEWER_BACKEND=wgpu`.
- Acceptance: `make gpu-wgpu` runs, wgpu-unsupported cases skip on caps; `make gpu`
  output identical to before.

### M2 — Cubemaps: skybox, IBL image light, horizon haze
- `textures.py`: cubemap store (6-face upload, CPU-generated mip chain — WebGPU has no
  auto-mipmap; IBL roughness LOD depends on it), `black_cube` fallback.
- `passes/skybox.py` + `shaders/skybox.wgsl`: fullscreen triangle, inverse view-proj
  ray, Z-up cubemap swizzle, far-plane depth; haze ring geometry (port `haze.vert`).
- Scene shader: IBL sampling (`textureSampleLevel`, diffuse at max mip, specular at
  roughness·maxMip, 5000-reference normalization).
- Flags: SKYBOX; caps notes updated.
- Acceptance: `test_horizon_haze.py` and the skybox/image-light cases of
  `test_shading.py` pass under wgpu.

### M3 — Shadows (largest single shader port)
- `cascades.py` port with WebGPU z∈[0,1] ortho (4096² atlas, 3 tiles, texel snap).
- `passes/shadow.py` + `shadow.wgsl` (depth-only), `spot_dist.wgsl` (R16Float distance,
  MIN blend, 2D-array layers — per-layer views are native in WebGPU, simpler than GL's
  `glFramebufferTextureLayer`).
- `shadow_sample.wgsl`: cascade select/fallback, slope-scaled bias, 3×3 PCF, atlas
  clamp, spot perspective PCF, point cube-face mapping, area-light 7×7 kernel.
- `lighting.py`: shadow-aware light scheduling (1 directional + 8 local).
- Flags: SHADOW; cast_shadow semantics; transparent does not cast.
- Acceptance: `test_shadows.py` (14) under wgpu.

### M4 — Planar reflections
- `passes/reflect.py`: plane detection/dedup (max 4), mirrored view matrix, front-face
  flip pipeline variant, clip-plane discard in the scene fragment, negative-reflectance
  channel encoding (layer/top-face), RGBA16F targets + shared depth, transparent
  re-draw inside reflection.
- Flags: REFLECTION.
- Acceptance: `test_reflection.py` (9) under wgpu.

### M5 — Debug views, selection outline, present modes
- WIREFRAME: lazy barycentric vertex buffer in `MeshStore` + pipeline constant variant
  (no geometry shader).
- OVERDRAW: additive, no-depth pipeline variant. ALBEDO/NORMAL/DEPTH already exist.
- SEGMENT/IDCOLOR: present pass reading the export id texture (port `present.frag`).
- `passes/outline.py` + `outline.wgsl`: selection mask (single-sample — WebGPU cannot
  resolve uint MSAA; the dilation shader already does its own AA) + port of the 62-line
  morphology shader with edge-continuity fix.
- Acceptance: debug-view cases of `test_shading.py`; new backend-neutral outline
  behavior tests (mask continuity under occlusion, one outline per link, viewport-edge
  behavior) run under both backends.

### M6 — Tendons
- `passes/tendon.py`: `_publish_tendons` packing, capsule shaft/cap instances (3 per
  segment) from the shared built-in meshes, own `InstanceStore`, material/transparent
  buckets with back-to-front order. Reuses the scene pipeline wholesale.
- Flags: TENDON.
- Acceptance: tendon cases of `test_pipeline.py` under wgpu.

### M7 — Debug draw + text + overlays (largest CPU+GPU chunk)
- Extract shared `DebugDraw` publisher out of `ForgeBackend` (labels, frames, bvh,
  contacts, joints, COM, inertia, actuators, rangefinder, constraint, flex).
- `passes/debug.py` + the six WGSL families (`debug_line/point/solid/sector/stroke/
  drag_link`, all `@builtin(vertex_index)` expansion), occlusion modes
  DEPTH/ALWAYS/GHOST (two-pass).
- Split `render/forge/text.py`: atlas building → shared; new wgpu glyph draw
  (`debug_text.wgsl`); `configure_text` on `WgpuBackend`.
- Flags: JOINT/COM/INERTIA/ACTUATOR/CONTACT*/BVH/CAMERA/LIGHT/RANGEFINDER/CONSTRAINT/
  FLEX*; `set_label_mode`/`set_frame_mode`/`set_bvh_depth` become real.
- Acceptance: backend-neutral debug-draw behavior tests (occlusion modes, pixel-constant
  line width, glyph quads, 10k-line frame) + label/frame/bvh cases of `test_pipeline.py`
  under wgpu.

### M8 — Gizmo
- `passes/gizmo.py` + `gizmo.wgsl`: 7 built-in handle meshes, screen-constant scale
  (`px_scale·clip.w`), depth pinned to the near plane (WebGPU z≈0), `u_mask_radius`
  center hole; `set_gizmo` stores and `caps.gizmo=True`.
- Acceptance: backend-neutral gizmo render tests (handle visible, screen-size
  invariance, mask hole); existing CPU hit-tests unchanged.

### M9 — Interactive viewer surface path (most integration risk)
- `ui/window.py`: backend mode. Under wgpu the window is a GLFW window with
  `GLFW_CLIENT_API=GLFW_NO_API` surfaced through rendercanvas; imgui renders via
  `wgpu.utils.imgui.ImguiRenderer`. The known imgui-bundle 1.92 incompatibility
  (`cmd_lists_count`) is fixed by a small vendored subclass in-repo — never by patching
  site-packages — with a version guard.
- `ViewportImage`: carries a backend payload; `_draw_viewport` binds the wgpu texture
  through the imgui renderer instead of `ImTextureRef(gl_id)`.
- `composition._compose`: honor `FORGE_VIEWER_BACKEND`; `doctor`/`backends` reporting.
- HiDPI: pixel-ratio handling must match the GL path (scale math already centralized in
  `Window`).
- Acceptance: `FORGE_VIEWER_BACKEND=wgpu make viewer` opens the full UI; window-stack
  tests that are backend-neutral (open, read_frame, edit→pixels) pass under wgpu;
  `make doctor` reports the wgpu path accurately.

### M10 — Finalization
- MSAA flag semantics aligned with forge (samples fixed at construction — verify forge
  behavior first, then match).
- `id_msaa`/caps/notes sweep; README + `docs/RENDERER.md` + `docs/ROADMAP.md` updates.
- Full verification matrix: default suite, per-file GPU loop on both backends,
  `renderer-api` + `renderer-api-wgpu`, gallery vs `mujoco.Renderer` on both backends.

## Explicitly out of scope (recorded in caps notes)

- GL state guarding, native GL entry points, GLSL hot reload, GPU timer queries
  (CPU pass timing only; wgpu timestamp is an optional feature — revisit later).
- `imgui-bundle` upstream fix submission (tracked separately).
