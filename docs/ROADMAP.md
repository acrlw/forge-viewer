# Roadmap

## Delivery model

- Each visible feature ships with a `make` acceptance target.
- Numeric tests protect behavior; gallery scenes support visual review.
- Render code consumes `SceneSource`, `SceneFrame`, and draw commands.
- Physics, RPC, replay, and programmatic tools enter through adapters.
- Remote viewers use independent processes and rendering contexts.
- Structure transport is reliable; frame transport keeps the latest state.

## Current baseline

Forge covers interactive rigid-body MuJoCo workflows: MJCF and URDF loading, primitive and
mesh geometry, height fields, materials, textures, tendons, sites, contacts, deformables,
lights, cameras, joints, actuators, sensors, picking, outlines, debug drawing, shadows,
reflections, capture, video, keyframes, perturbation, and paused pose editing.

The same UI and renderer operate with programmatic scenes, `ToyPhysicsAdapter`, remote
publishers, and snapshot replay.

The editor starts with `make empty` and loads MJCF or URDF models through the File menu or window
file drop. Runtime loading refreshes adapter structure, scene resources, panels, cameras, and
interaction state.

## MuJoCo coverage

Completed integration:

- geom, site, joint, tendon, actuator, flex, and skin visual groups
- shadow, reflection, additive transparency, skybox, fog, haze, wireframe, cull-face, static,
  skin, flex-face, flex-skin, flex-vertex, flex-edge, and convex-hull render flags
- joint, center-of-mass, inertia, actuator, activation, rangefinder, and equality overlays
- slider-crank actuator linkages and broken-crank coloring
- contact points and forces
- split contact-force components and automatic body-to-joint connections
- constraint-island colors for geoms, flexes, tendons, and contacts
- body, mesh, and flex bounding-volume hierarchy overlays with depth selection and interpolated-flex control cages
- named and calibrated cameras, including body attachment
- editable lights, cameras, and environment settings through Forge scene entities
- mocap pose and equality controls
- 1D, 2D, and 3D flex surfaces plus skinned meshes
- body, joint, geom, site, camera, light, tendon, actuator, constraint, flex, contact, and
  selection labels
- world, body, geom, site, camera, light, and contact coordinate frames
- tendon material, color, transparency, texture, and repeat behavior
- keyframe loading and 60 fps video export
- up to 100 active scene lights, matching MuJoCo's `mjMAXLIGHT`

Planned integration:

| Area | Work item | Priority |
|---|---|---|
| Lighting | Image-light calibration | P2 |
| Editing | Optional inverse-kinematics component | P3 |

`make mujoco-audit` reports model coverage and adapter write capabilities. Strict mode fails
when the active model contains skipped instance data.

## Forge scene entities

Lights, cameras, and the environment are native Forge entities. `SceneSource` owns their stable
configuration, `Session` owns user overrides, and `SceneFrame` supplies dynamic world transforms.
This model serves programmatic scenes, MuJoCo, remote viewers, and replay.
Remote editing preserves typed light, environment, material, geometry color, and camera commands.

Next components:

1. Runtime entity creation over RPC.

`Scene` provides stable add/remove identities for objects, lights, and cameras.
`.forge.json` stores authored objects, shared materials, meshes, textures, lights, environment
settings, and cameras.

Acceptance starts in `make canvas` and `make lighting`, followed by adapter import and
write-back coverage.

## MuJoCo parity

Completion criteria:

- Built-in coverage scenes pass strict audit.
- Every supported visualization has positive and negative tests.
- Every interactive visualization has a Make target.
- Reference comparisons document measured rendering differences.
- Complex models pass structure, frame, dynamic-mesh, and keyframe conformance.

Primary commands:

```bash
make mujoco-audit
make mujoco-visuals
make mujoco-debug
make mujoco-actuators
make mujoco-slider-crank
make mujoco-islands
make mujoco-bvh
make mujoco-convex-hull
make mujoco-solver-diagnostics
make mujoco-rangefinder
make mujoco-constraints
make mujoco-overlays
make deformables
make musculoskeletal-check
```

## Remote debugging

The current snapshot protocol carries structure revisions, frame sequences, scene frames, and
debug batches. `RemoteSceneAdapter` feeds this data into the standard Session and Forge paths.
Commands use a separate acknowledged channel.

Delivered workflows:

- `make pvd`: one publisher plus effect and debug viewers
- `make snapshot-record`: record a live stream
- `make snapshot-replay`: reproduce a scene after the physics process exits

Future work focuses on transport implementations, stream inspection, and timeline controls.

## Additional physics backends

`ToyPhysicsAdapter` validates the public contract with an independent timestep, gravity,
ground collision, reset, and pose editing. Third-party engines can implement
`SceneAdapterBase` and run `check_adapter()` before opening a viewer.

```bash
make toy-physics
make adapter-conformance
make adapter-conformance ADAPTER=mujoco CONFORMANCE_ASSET=deformables
```

## Graphics API and materials

PBR and graphics API migration are separate projects. The present MuJoCo workflow uses
specular, shininess, emission, and reflectance data through OpenGL 4.1.

A second rendering backend becomes valuable with concrete demand for compute workloads,
GPU-driven submission, broader Apple platform support, or glTF metallic-roughness materials
with normal maps, HDR lighting, and image-based lighting.

The backend prototype should implement opaque rendering, ID picking, outlines, and capture
behind the current `RenderBackend` and scene contracts. wgpu offers a practical first target.
Measured Python submission cost will guide any later Rust or C++ renderer core. Adapters,
commands, remote transport, and UI remain reusable.
