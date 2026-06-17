# Agent Guide — Omniverse / Replicator Channels

Read this **before** writing or editing any code in a Rendered.ai channel that
uses NVIDIA Omniverse Replicator (`channel.type: omniverse`). It only covers
the things that consistently trip up LLM agents who have done Blender or
plain-Python channels but are new to Replicator. For everything else (channel
config, deploy, docs, validation, common mistakes that are not
Replicator-specific) read [`AGENT.md`](AGENT.md) first.

---

## The single biggest mental-model trap

**Replicator code is *declarative*, not imperative.** Calls like
`rep.create.*`, `rep.modify.*`, `rep.randomizer.register(...)`, and
`rep.distribution.*` build an **OmniGraph** at "design time". Nothing actually
renders, randomizes, or writes a file at the point you call them. Execution
happens later, when a trigger fires (`rep.trigger.on_frame`,
`rep.trigger.on_time`) under the orchestrator.

Concretely this means:

- Don't try to read a value back from a `rep.distribution.*` call — it's a
  graph node, not a sampled number.
- Don't `print(prim.GetAttribute(...))` and expect the value the renderer
  will see — at design time the prim may not exist yet.
- Variables you bind at `exec()` time are captured in a closure; the closure
  is what runs per frame. Side-effects done outside the closure happen exactly
  once, before any frame.
- "Why isn't my randomizer running?" is almost always: you wrote a function
  that calls `rep.modify.*` but you forgot to either (a) wrap the body in a
  `with replicator_item:` block, or (b) `register()` the function and add it
  to the renderer's `Randomizers` output.

---

## The randomizer contract

A scene-assembler node returns `{"Randomizers": [fn1, fn2, ...]}` where each
`fn` is a **closure** that, when called, returns a Replicator node. The
render node aggregates these and registers them inside a trigger:

```python
# Inside a scene assembler's exec()
def env_props():
    instances = rep.randomizer.instantiate(usd_files, size=n, mode='point_instance')
    with instances:
        rep.modify.pose(
            position=rep.distribution.uniform(lo, hi),
            rotation=rep.distribution.uniform((0, 0, 0), (360, 360, 360)),
        )
    return instances.node          # MUST return a node

return {"Randomizers": [env_props], "Camera": cam}
```

```python
# Inside the render node
with rep.trigger.on_frame(max_execs=num_frames):
    for fn in randomizer_fns:
        rep.randomizer.register(fn)
        getattr(rep.randomizer, fn.__name__)()
```

Common bugs:

- **Returning `None`.** The trigger silently does nothing.
- **No `with replicator_item:` block** around `rep.modify.pose(...)` etc.
  The modifier attaches to a global default and randomizes nothing useful.
- **Capturing `self.inputs` directly inside the closure.** Snapshot the
  values you need into local variables in `exec()` first; `self.inputs` may
  not be valid by the time the closure fires.

---

## Custom writer + annotators

Per-frame outputs (images, masks, annotations, metadata) are emitted by a
subclass of `omni.replicator.core.Writer`. The pattern:

```python
class MyWriter(Writer):
    def __init__(self, output_dir, ...):
        self._backend = BackendDispatch({'paths': {'out_dir': output_dir}})
        self.annotators = [AnnotatorRegistry.get_annotator("rgb")]
        if not ctx.preview:
            for name in ("bounding_box_2d_loose_fast",
                         "semantic_segmentation",
                         "instance_segmentation_fast",
                         "distance_to_camera",
                         "camera_params"):
                self.annotators.append(AnnotatorRegistry.get_annotator(name))

    def write(self, data, sensor_name="RGBCamera"):
        if ctx.preview:
            self._backend.write_image(f"preview.png", data['rgb'])
            return
        # ... write images/masks/annotations/metadata for this frame
```

```python
rep.WriterRegistry.register(MyWriter)
writer = rep.WriterRegistry.get('MyWriter')
writer.initialize(output_dir=ctx.output, ...)
writer.attach([rep.create.render_product(cam, resolution) for cam in cameras])
```

Gotchas:

- **Annotator names change between Replicator versions.** Names like
  `bounding_box_2d_loose_fast`, `instance_segmentation_fast` are 1.10.x. If
  you copy code targeting 1.6.x or an Isaac Sim image you'll get
  `KeyError`s from the registry. Pin to what your base image ships.
- **`BackendDispatch.write_image` wants `uint8` RGB.** Float / >8-bit data
  silently truncates (or fails). Normalize and cast first.
- **`write()` is called even when zero objects are visible.** Always guard
  bbox / segmentation lookups; an empty render is valid.
- **Don't call `rep.orchestrator.run()` in your node.** The platform
  trigger machinery already runs the orchestrator. Calling it manually
  hangs or duplicates work.

---

## Preview mode

`ana --preview` sets `ctx.preview=True` and expects exactly one
`preview.png` written to `ctx.output`. The platform displays this thumbnail
in the channel UI.

Standard skeleton in any render-ish node:

```python
if ctx.preview:
    num_frames = 1
    # skip heavy annotators (segmentation, depth, normals, etc.)
    # render at lower resolution
    scene.resolution = (512, 512)
else:
    num_frames = int(self.inputs["Frames per Run"][0])
```

Failing to write `preview.png` → no thumbnail; the platform flags the channel
as broken.

---

## `with` blocks, modifiers, and prim handles

`rep.create.*` returns a `ReplicatorItem` handle, **not** a USD prim. To
modify the thing it represents, use a `with` block:

```python
cam = rep.create.camera()

# ✅ pose modifier scoped to this camera
with cam:
    rep.modify.pose(position=rep.distribution.uniform(lo, hi),
                    look_at=target)

# ❌ this attaches to the *current* default item (often nothing useful)
rep.modify.pose(position=...)
```

If you need raw USD access (e.g. to read a prim transform), import `omni.usd`
**lazily** inside the function — top-level imports of `omni.usd` outside Kit
context will explode:

```python
def export_stage(path):
    import omni.usd  # lazy
    stage = omni.usd.get_context().get_stage()
    stage.GetRootLayer().Export(path)
```

---

## Determinism

Replicator has its **own** RNG separate from `numpy.random` / `ctx.random`.
You must seed both for fully reproducible runs:

```python
import omni.replicator.core as rep
rep.set_global_seed(ctx.seed)        # seeds rep.distribution.*
# ctx.random is already pre-seeded by anatools using ctx.seed + ctx.interp_num
```

Render nodes typically call `rep.set_global_seed(ctx.seed)` once at the top
of `exec()`. If you spin up Replicator distributions before that, those
samples are non-deterministic.

`rep.distribution.choice([...])` over a Python list is deterministic given a
seeded global. `ctx.random.choice(...)` over the same list uses NumPy's RNG
and is deterministic via `ctx.random`. Pick one source per decision and stick
with it — mixing them per-frame makes "why is run N different from run M?"
debugging miserable.

---

## Stage / scene state leaks across runs

The Omniverse stage is **process-global**. Across sequential graph runs
inside the same container, prims, lights, cameras, and randomizers from a
previous run can survive into the next one. Symptoms:

- Random duplicate cameras showing up in the second run.
- A second-run scene assembler "inheriting" lights from the first.
- Annotator data referencing prim paths that no longer exist.

If a channel runs multi-graph workflows or you're iterating in the same
container, reset the stage explicitly at the top of the assembler:

```python
import omni.usd
omni.usd.get_context().new_stage()
```

For single-graph runs the platform tears the container down each time, so
this is mostly a local-iteration concern.

---

## Channel-yml essentials specific to omniverse

```yaml
channel:
  type: omniverse        # not 'blender', not 'python'

add_setup:
  - my_package.lib.setup # optional GPU / Kit / extension setup
```

`type: omniverse` selects the Kit-extension runtime (`ana.interpreter`) at
deploy time. Setting it to `python` or `blender` will deploy but every node
that calls `omni.replicator.core` will fail at import time on the platform.

The base image is the largest and slowest of any Rendered.ai channel type.
Expect:

- Multi-GB image pulls.
- Several minutes of shader compilation on first run (`Waiting for
  compilation of ray tracing shaders by GPU driver: 240 seconds so far`).
- A trailing `RTX Ready` log line **even after a Python crash** — the kit
  app keeps the process alive until the cloud manager times out, so a
  successful-looking final log is not a successful run.

Local dev requires the NVIDIA Container Toolkit and a real GPU. CPU-only
hosts cannot run the channel.

---

## What you almost certainly should NOT do

- `import bpy` — Replicator channels do not run inside Blender.
- Call `rep.orchestrator.run()` from a node.
- Hardcode UUIDs for asset volumes — use `get_volume_path()`.
- Hardcode prim paths under `/Replicator/...` — use the handles returned by
  `rep.create.*` and the `path_pattern=` arg of `rep.get.prims`.
- Assume `print()` shows up on the platform — use `logger.*` (same rule as
  every other channel type).
- Assume the cached image will be rebuilt for a Python-only change — bind
  mount picks up `.py` / `.yaml` edits live, but Dockerfile / requirements
  changes are required to bump the base image.

---

## Quick reference: where to look in the source

When you need to verify behaviour beyond this doc, the canonical references
are:

- **Replicator API docs** —
  `https://docs.omniverse.nvidia.com/py/replicator/<version>/source/extensions/omni.replicator.core/docs/API.html`
  (pin to the version matching your base image).
- **Replicator getting-started** —
  `https://docs.omniverse.nvidia.com/extensions/latest/ext_replicator/getting_started.html`
- **anatools channel runtime** — `anatools/lib/channel.py`,
  `anatools/lib/context.py`, `anatools/bin/anadeploy`, `anatools/bin/anautils`
  on the host.

When in doubt, follow the install path: `python3 -c "import anatools; print(anatools.__file__)"`.
