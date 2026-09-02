# Batch rendering for reinforcement learning

Mojive can render a short sequence of cameras or simulation states today, but the current public
`Renderer` is a single-view renderer. Looping over `update_scene()` and `render_async()` is a useful
compatibility path, not a native vectorized sensor. A native RL path must preserve a batch on the
GPU from transform upload through product generation and, where the consumer permits it, through
observation processing.

## Current capability and limit

One backend owns one `RenderScene.camera`, one frame-uniform block, one 2D render target, and one
set of instance streams. `create_peer()` creates another target on the same graphics device or
context, but each peer currently owns its own mesh, texture, instance, and pass resources. It is
appropriate for a camera preview; creating one peer per vectorized environment would duplicate
immutable resources and command setup.

The public API can still pipeline a small batch:

```python
outputs = [np.empty((height, width, 3), np.uint8) for _ in states]
futures = []
for state, output in zip(states, outputs, strict=True):
    renderer.update_scene(state, camera="policy")
    futures.append(renderer.render_async(out=output))
images = np.stack([future.result() for future in futures])
```

Queue ordering makes the results correct and the bounded WebGPU readback ring overlaps mapping
with later submissions. It nevertheless performs one scene update, render graph, target clear,
and capture submission per view. On the Apple M5 development host, 16 identical 128×128 primitive
views reached about 632 RGB views/s and 753 depth views/s through this fallback. The synchronous
loop reached about 368 and 403 views/s respectively. These numbers establish a local baseline, not
a promise: work and CPU encoding still scale almost linearly with view count, and the readback ring
limits outstanding captures to three.

This is sufficient for occasional multi-camera capture and small CPU-side dataset jobs. It does
not satisfy a high-throughput vector-environment RL workload with tens or hundreds of worlds.

## What Newton's tiled camera actually does

Newton's
[`SensorTiledCamera`](https://newton-physics.github.io/newton/latest/api/_generated/newton.sensors.SensorTiledCamera.html)
is a Warp ray tracer, not a collection of OpenGL or WebGPU raster viewports. Its model already has
an explicit world axis. Camera transforms have shape `(camera_count, world_count)`, camera-space
rays have shape `(camera_count, height, width, 2)`, and requested products remain GPU arrays with
shape `(world_count, camera_count, height, width)`.

One Warp megakernel is launched over `world_count * camera_count * pixels_per_view`. A thread maps
its linear ID to a world, camera, and pixel, transforms the supplied ray, traverses per-world BVH
roots, shades the closest hit, and writes every requested channel for that pixel. Its `TILED`
render order changes thread scheduling into small image tiles for locality; it does not mean that
the renderer first builds a visual atlas. The separate `flatten_*_to_rgba()` helpers arrange
already-rendered batch outputs into an atlas for display.

This design gets its throughput from four properties:

- worlds and cameras are first-class array dimensions;
- all view pixels are covered by one GPU launch rather than a Python render loop;
- geometry acceleration structures and output arrays stay on the simulation device; and
- color, depth, normal, albedo, and shape index are optional outputs of the same per-pixel kernel.

Newton is also moving toward an explicitly selected flat view batch. Its proposed
[`SensorBatchedCamera`](https://github.com/newton-physics/newton/pull/3276) maps every rendered view
to a `(world, ray_bundle)` pair instead of requiring a dense world-by-camera cross product. That is
the more useful API lesson for Mojive. The ray-tracing implementation itself is not directly
transferable to Mojive's full-featured raster pipeline, and performance numbers are not comparable
without matching geometry, shading, output products, device, and image size.

## Target contract

The canonical batch should be a flat list of selected views. Dense `(world, camera)` output can be
a convenience reshape, but it should not force unused combinations to render.

```text
RenderResources
  immutable meshes, textures, materials, samplers, pipelines

SceneBatch
  topology cohort
  transforms[world, instance, 4, 4]
  optional per-world visual and identity streams

CameraBatch
  view[view]
  world_index[view]
  camera_index[view]

RenderBatchResult
  product arrays or GPU handles shaped [view, height, width, channels]
  view-to-world/camera metadata
```

A topology cohort contains worlds that share mesh/material bindings and instance layout. Different
robots or scene topologies form separate cohorts; they should not add branches to every draw. This
keeps ordinary single-scene rendering readable while allowing replicated RL environments to share
all immutable resources.

The future public operation should accept the requested products and destination policy explicitly:

```python
result = batch_renderer.render(
    scene_batch,
    cameras,
    products=("rgb", "metric_depth", "segmentation"),
    destination="cpu",  # or "gpu"
)
```

`destination="cpu"` returns or fills contiguous NumPy arrays. `destination="gpu"` must return a
backend-owned object with explicit lifetime and synchronization; it must not pretend that a WebGPU
buffer is automatically a CUDA, Metal, Warp, JAX, or PyTorch tensor.

## Portable WebGPU execution plan

The first native implementation should use 2D texture arrays internally. Each layer is one view
with the same extent and format. Arrays avoid atlas padding, filtering bleed, and accidental depth
interaction; an atlas remains a presentation conversion for the editor.

Core WebGPU can render each array layer with a short render pass while retaining one command
encoder and one queue submission for the whole batch. Each pass selects a per-view frame record by
dynamic offset and draws only the instance range belonging to that view's world. The draw list,
pipelines, mesh buffers, texture groups, and render bundles are shared. This is not one hardware
draw for all cameras, but it removes Python/API submission, allocation, and resource duplication
from the per-view path while staying portable across Metal, Vulkan, and DX12.

For the 1×-sample sensor profile, color, metric depth, and identity should be emitted together as
multiple render targets when requested. The current separate export pass remains necessary for the
interactive 4×-MSAA viewport because WebGPU cannot resolve multisampled integer or depth targets.
This distinction belongs in the render-plan compiler, not in a parallel ad-hoc renderer.

After rendering, one compute dispatch packs all color layers and one texture-to-buffer copy per
requested typed product transfers the complete batch. The reusable synchronous staging buffer and
bounded asynchronous ring already establish the correct map/decode lifetime for this step; they
need a layered layout rather than a new readback subsystem.

## Multi-world data path

Replicated worlds should upload transforms in one contiguous write. The renderer needs a table of
per-world, per-bucket instance ranges so a view can draw only its selected world. Identical
topologies reuse the same mesh and texture bindings; only pose records are multiplied by world
count. Visual and identity streams may be shared across worlds until a task randomizes them.

The scene contract therefore needs a deliberate separation that the current `create_peer()` path
does not provide:

1. shared immutable `RenderResources`;
2. topology-cohort draw metadata;
3. batched mutable instance streams; and
4. per-request cameras, products, and targets.

This also improves camera preview and multi-window rendering: peers can reference the same resource
owner instead of re-uploading a complete scene.

## GPU observation boundary

Native batch rasterization is enough when observations must reach CPU NumPy arrays, video encoders,
or network transport. It is not enough for GPU-resident RL. Mapping any render target to CPU forces
a synchronization and destroys the main advantage of vectorized simulation.

WebGPU and wgpu-py do not provide a portable zero-copy bridge from their buffers to CUDA/Metal
training tensors. Mojive should keep the batch contract backend-neutral and implement GPU export
only where ownership and synchronization can be made real. Plausible future implementations are a
Warp/CUDA sensor backend, a platform-specific external-memory bridge, or observation preprocessing
that remains inside WebGPU. None should leak into the ordinary `Renderer` API as a fake NumPy
optimization.

## Delivery stages and acceptance

1. **Shared resources and flat camera batches.** Refactor peers to share immutable GPU resources;
   add same-scene multi-camera rendering into array layers with one encoder/submission and one
   batch readback.
2. **Replicated topology cohorts.** Add one batched pose upload, world-to-instance ranges, and
   explicit view-to-world mapping.
3. **Sensor render plans.** Fuse 1× color/depth/identity output, add optional normal/albedo only when
   the scene contract defines them, and keep expensive viewport effects opt-in.
4. **GPU destinations.** Add a real device-resident result only for backends with supported tensor
   interoperability.

Every stage must compare the native batch against the serial reference pixel-for-pixel and measure
1, 4, 16, 64, and 256 selected views at 128² and 256². Reports should include batch latency,
views/s, pixels/s, CPU command time, GPU time, upload bytes, readback time, submission count, and
peak GPU/CPU memory. A stage is not accepted merely because one view is faster: immutable resource
count and submission count must stop growing per view, and the single-view regression must remain
small enough to keep the interactive renderer healthy.

The next implementation step is stage 1. It creates the ownership boundary needed by all later
work and produces a useful multi-camera API before committing Mojive to a particular second physics
or GPU-compute backend.
