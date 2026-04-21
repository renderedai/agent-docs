# Agent Notes — Warehouse Fulfillment Center Channel

This document is **specific to this repo/channel** (local config: `external-warehouse.yml`).

For general Rendered.ai channel development guidance, see: `/ana/AGENT.md`.

---

## Best Practices — Running Tests

### Always run `ana` from `/ana`
Run with working directory set to `/ana`.

### Always create the output directory first
If you use `--logfile`, the directory must already exist.

```bash
mkdir -p /home/anadev/test_runs/warehouse_preview
ana --graph /ana/graphs/warehouse_anomaly.yml \
  --output /home/anadev/test_runs/warehouse_preview \
  --logfile /home/anadev/test_runs/warehouse_preview/ana.log
```

### Prefer `--preview` for iteration
This channel uses `ctx.preview` in the render node to reduce work during iteration.

```bash
mkdir -p /home/anadev/test_runs/warehouse_preview
ana --preview --graph /ana/graphs/warehouse_anomaly.yml \
  --output /home/anadev/test_runs/warehouse_preview \
  --logfile /home/anadev/test_runs/warehouse_preview/ana.log
```

Preview outputs a `preview.png` in the run directory in addition to normal dataset artifacts.

Preview can still be slow when placing many warehouse objects. To skip expensive object
randomization steps (material/texture variations), set:

```bash
export SINGLEBLENDERFILE_SKIP_OBJECT_RANDOMIZE=1
```

Example:

```bash
mkdir -p /home/anadev/test_runs/preview_fast
SINGLEBLENDERFILE_SKIP_OBJECT_RANDOMIZE=1 \
ana --preview --graph /ana/graphs/warehouse_anomaly.yml \
  --output /home/anadev/test_runs/preview_fast
```

Blender/HumGen output can be noisy; the render node redirects most Blender operator output to a per-run log file in the output directory.

### Know where to look for artifacts
A successful run typically creates:

- `preview.png` (preview runs)
- `images/*.jpg`
- `masks/*.png`
- `annotations/*-ana.json`
- `metadata/*-metadata.json`
- `bbox3d_vis/*-annotated-ground_point.jpg` (when ground-point viz is enabled)
- `*-blender_ops.log` (Blender operator output bundled with the dataset zip; does not reliably include Python `logger.info` output)
- `test/*-blender_file.blend` (optional blend snapshot for post-run inspection; off by default)

### Platform logs (source of truth)
When debugging platform runs, use the **official platform log** (downloaded via the web app or SDK). In this repo, those downloaded logs have been saved under `/ana/` with timestamped names like:

`/ana/2026-04-06T16_35_14-06_00.log`

This log is the authoritative place to look for:

- Node execution messages
- Python `logger.info` output from `warehouse_render.py`, `scene.py`, `ana_object.py`, etc.

The `0000000000-blender_ops.log` file inside the dataset zip is still useful for Blender-level warnings/errors, but it should not be treated as the primary record of Python logging.

### Replay a platform frame locally (seed + interp_num + camera)
Goal: reproduce a specific platform-rendered frame deterministically for debugging (e.g. missing annotations).

Inputs you need from the platform output bundle:

- `graph.json`
- `metadata/*-metadata.json` (for `seed` and `interp_num`)
- The target camera name (e.g. `Camera_021`) and frame index (usually `0000000000` in these runs)

Steps:

1. Convert the exported `graph.json` to a local `.yml` graph.
   - The minimal requirement is that node `values` and the `links` into `Warehouse Scene Render` match.
   - If you only want a single frame/camera for debugging, keep the graph unchanged and force the camera via env var (below).
2. Run locally using the exact `seed` + `interp_num` from platform `*-metadata.json`.
3. Force a specific camera (and optionally time-of-day) via env vars.
4. Enable a debug `.blend` snapshot if you need to inspect the scene in Blender.

Example (replay Camera_021):

```bash
mkdir -p /home/anadev/test_runs/replay_platform
SINGLEBLENDERFILE_CAMERA=021 \
SINGLEBLENDERFILE_SAVE_PREVIEW_BLEND=1 \
ana --graph /path/to/replay_graph.yml \
  --seed <seed_from_platform_metadata> \
  --interp_num <interp_num_from_platform_metadata> \
  --output /home/anadev/test_runs/replay_platform
```

Optional time-of-day override:

```bash
export SINGLEBLENDERFILE_TIME_OF_DAY='Work Hours'
```

### Validate annotations vs masks (common failure mode)
If you see an anomaly object (bag, backpack, package) in the RGB image but it's missing from `*-ana.json`, check whether it appears in the instance mask.

Quick check: compare the set of non-zero mask IDs vs `annotations[].id`.
If there are IDs present in the mask but absent from `*-ana.json`, the renderer produced a mask instance that did not get exported into annotations.

### Anomaly object annotations: tiny mask fragments can be dropped
Anomaly object masks can be extremely sparse (only a few dozen pixels) when the object is far from the warehouse surveillance camera. In that case, the mask may contain the object instance ID, but `compute_polygons()` can still return `None` because of the minimum bbox size filter.

Implementation detail:

- `SingleBlenderFile/lib/bbox.py:compute_polygons()` drops polygons whose bounding rect has width/height smaller than `MIN_FEATURE_SIZE`.
- To avoid dropping real-but-small anomaly object masks, these objects use a smaller minimum feature size than normal warehouse equipment.

If platform runs show nonzero IDs in `masks/*.png` but `annotations/*-ana.json` is empty, this threshold is the first thing to check.

### Platform debugging: mask pixel diagnostics
`AnaScene.write_ana_annotations()` logs per-camera mask stats to help diagnose whether objects are present in the mask:

- `AnaScene mask file: ... nonzero=<count> uniq_ids=[...]`
- `AnaScene mask pixels: ... object_type=<...> pixels=<count>`

These messages are emitted via both `logger.info` and `print()` so they show up in platform-captured logs.

---

## Debug Environment Variables (SingleBlenderFile Channel)

This channel supports several debug environment variables for local development:

```bash
# Performance optimization
export SINGLEBLENDERFILE_SKIP_OBJECT_RANDOMIZE=1  # Skip expensive object material randomization

# Camera and rendering control
export SINGLEBLENDERFILE_CAMERA=021               # Force specific camera (e.g. 021 or Camera_021)
export SINGLEBLENDERFILE_LIGHTING='Day Shift'     # Override lighting setting

# Debug outputs and logging
export SINGLEBLENDERFILE_SAVE_PREVIEW_BLEND=1     # Save .blend snapshot in output/test/
export SINGLEBLENDERFILE_LOG_BLEND_PATH=1         # Log which blend file was opened
export SINGLEBLENDERFILE_KEEP_OPEN_LOG=1          # Preserve blender_open.log in output

# Raw IndexOB debugging (for mask issues)
export SINGLEBLENDERFILE_RAW_INDEXOB_DEBUG=1      # Render standalone IndexOB EXR
export SINGLEBLENDERFILE_RAW_INDEXOB_CAMERA=123   # Optional camera filter for IndexOB debug
```

**Usage examples**:
```bash
# Fast preview with single camera
SINGLEBLENDERFILE_SKIP_OBJECT_RANDOMIZE=1 \
SINGLEBLENDERFILE_CAMERA=021 \
ana --preview --output /home/anadev/fast_test

# Debug blend with snapshot
SINGLEBLENDERFILE_SAVE_PREVIEW_BLEND=1 \
ana --graph /ana/graphs/warehouse_anomaly.yml \
  --output /home/anadev/debug_blend
```

---

## General Learnings — Blender / Channel Testing

### Prefer a small external “channel-like” render to validate visibility
When debugging object visibility (e.g. anomaly objects), it's often faster to validate directly in Blender (outside `ana`) before chasing annotation/mask pipeline issues.

Useful repo scripts:

- `/ana/test_object_visibility.py`
  - Renders `IndexOB` for warehouse cameras and counts anomaly object pixels per camera.
  - Good for finding a "known good" camera.
- `/ana/test_channel_like_object_render.py`
  - Uses a minimal compositor (`Render Layers -> IndexOB -> EXR`) and counts pixels.
  - Good as a deterministic regression (e.g. standardize on `Camera_123`, threshold 100).

### If masks are empty, confirm whether **raw IndexOB** is empty first
If `masks/*.png` is all-zero (or near-zero), determine whether:
- The underlying `IndexOB` pass is empty (scene/camera/visibility issue), or
- The mask pipeline is broken (compositor nodes / ID Mask wiring).

The render node supports an env-var gated raw IndexOB debug mode:

```bash
export SINGLEBLENDERFILE_RAW_INDEXOB_DEBUG=1
export SINGLEBLENDERFILE_RAW_INDEXOB_CAMERA=123  # optional (e.g. 123 or Camera_123)
```

This will:

- Render a standalone IndexOB EXR into `output/raw_indexob_exr/`
- Print `nonzero` and `uniq_ids` for the selected camera
- Print runtime transforms for the camera and a small set of anomaly objects (useful for catching transform mutation)

### Confirm which blend file was opened
When a new `.blend` arrives (e.g. from a contractor), confirm the channel opened the file you expect:

```bash
export SINGLEBLENDERFILE_LOG_BLEND_PATH=1
export SINGLEBLENDERFILE_KEEP_OPEN_LOG=1
```

`SINGLEBLENDERFILE_KEEP_OPEN_LOG=1` preserves `blender_open.log` in the run output folder.

### Avoid accidental transform mutation of non-person props
A common failure mode is accidentally adding warehouse infrastructure (conveyors, racks, etc.) into `ana_scene.objects` and then running "anomaly object placement" code on all objects.

Symptom:

- Warehouse geometry looks correct, but anomaly objects vanish or move between runs.
- Raw IndexOB debug shows near-zero pixels even though the object is visible in external tests.

Fix pattern:

- Filter placement loops to operate only on anomaly objects (bags, backpacks, packages).
- Do not reset `.location` / `.rotation_euler` on warehouse infrastructure roots.

### Suggested workflow when receiving a new contractor `.blend`
1. Run external validation first (fast):
   - Use `/ana/test_channel_like_object_render.py` (standard: `Camera_123`, threshold `100`) to ensure anomaly objects are visible in IndexOB.
2. Run a single-camera channel replay:
   - Force a camera via `SINGLEBLENDERFILE_CAMERA=123`
   - Enable raw IndexOB debug if needed (above)
   - Optionally save a snapshot blend with `SINGLEBLENDERFILE_SAVE_PREVIEW_BLEND=1`
3. If channel run differs from the external validation:
   - Check raw IndexOB debug output (`nonzero`, `uniq_ids`)
   - Check runtime transforms logged for camera + anomaly objects

### Optional debug overlays (library helper)
For local sanity checking, you can generate overlay imagery from the saved RGB image using:

`SingleBlenderFile/lib/debug_overlays.py:write_debug_overlays(image_file, vis_dir)`

This wraps the existing `draw(...)` helper to write debug overlays (e.g. `box_3d`, `ground_point`) without keeping large commented blocks in the render node.

### Quick annotation sanity check
Use this to confirm you got the expected number of annotations and key fields:

```bash
python3 - <<'PY'
import json, glob
files = sorted(glob.glob('/home/anadev/test_runs/warehouse_preview/annotations/*.json'))
print(f'{len(files)} annotation file(s)')
for fn in files:
    with open(fn) as f:
        data = json.load(f)
    anns = data.get('annotations', [])
    print(f'  {fn.split("/")[-1]}: {len(anns)} annotations')
    for a in anns:
        oid = a.get('object_id','?')
        obj_type = a.get('object_type','?')
        bbox = a.get('bbox','?')
        anomaly = a.get('is_anomaly','?')
        print(f'    {oid:16s} type={obj_type:12s} anomaly={anomaly:8s} bbox={bbox}')
PY
```

### Inspect the saved blend snapshot (optional but powerful)
The render node can write a blend snapshot into `output/test/*-blender_file.blend` for post-run inspection.

This is gated behind an environment variable because it can create large files:

```bash
export SINGLEBLENDERFILE_SAVE_PREVIEW_BLEND=1
```

Then run normally (no `--preview` required):

```bash
mkdir -p /home/anadev/test_runs/debug_blend
SINGLEBLENDERFILE_SAVE_PREVIEW_BLEND=1 \
ana --graph /ana/graphs/warehouse_anomaly.yml \
  --output /home/anadev/test_runs/debug_blend
```

You can open the saved blend locally in Blender, or open it headless and print debug info.

Example pattern (create a custom script based on the template):

```bash
# Copy and modify test_bpy.py to target your specific blend file
cp /ana/test_bpy.py /home/anadev/my_blend_inspector.py
# Edit the BLEND_FILE path in your copy to point to your saved blend
# Then run:
blender --background --python /home/anadev/my_blend_inspector.py 2>&1 | grep -E "^(HG_|===)"
```

The `test_bpy.py` script serves as a template - copy it and modify the `BLEND_FILE` path to target your specific run directory:
`/home/anadev/<run_dir>/test/*-blender_file.blend`.

### Inspect a blend and re-annotate it with current code
Two useful scripts live at repo root:

- `test_bpy.py`
  - Template script for blend file inspection. Copy and modify the `BLEND_FILE` path to target your specific blend file. Prints collection visibility, warehouse objects, camera info, and object placement.
- `test_reannotate_from_blend.py`
  - Loads a saved `*-blender_file.blend` and re-runs the annotation code path without re-running placement.

Example (inspect a blend):

```bash
blender --background --python /ana/test_bpy.py 2>&1 | sed -n '1,200p'
```

Example (re-annotate a saved debug blend):

```bash
mkdir -p /home/anadev/test_runs/reannotated
blender --background --python /ana/test_reannotate_from_blend.py -- \
  /path/to/*-blender_file.blend /home/anadev/test_runs/reannotated
```

---

## Test Graphs

- `graphs/warehouse_anomaly.yml`
  - Warehouse Scene, multiple surveillance cameras
  - Good for checking anomaly object placement and visibility

- `graphs/warehouse_conveyor.yml`
  - Conveyor system with package placement
  - Good for verifying object tracking and movement annotations

---

## Implementation Pointers (where things live)

- Render/placement logic: `packages/SingleBlenderFile/SingleBlenderFile/nodes/warehouse_render.py`
- Object annotation/metadata fields: `packages/SingleBlenderFile/SingleBlenderFile/lib/ana_object.py`
- Warehouse/camera mappings: `SingleBlenderFile.lib.warehouse_data` (imports in `warehouse_render.py`)

---

## Warehouse / Camera Layout Notes (XY plane)

Blender/world axes:

- `+Z` is up.
- In Blender **Top** view: image-right is `+X`, image-up is `+Y`.

Axis-aligned warehouse layout reference is documented in:

- `packages/SingleBlenderFile/SingleBlenderFile/docs/warehouse_layout.md`
  - embedded image: `warehouse_top_view_labeled.png`

Camera-cluster centroids (from `SingleBlenderFile.lib.warehouse_data.CAMERA_XY`):

- **Receiving Area**: Entry point for packages
  - Receiving centroid ≈ `(-20.0, -10.0)`
- **Storage Zones A-D** are arranged in grid pattern:
  - Zone A centroid ≈ `(-10.0, 0.0)`
  - Zone B centroid ≈ `(10.0, 0.0)`
  - Zone C centroid ≈ `(-10.0, 20.0)`
  - Zone D centroid ≈ `(10.0, 20.0)`
- **Shipping Area**: Exit point for processed packages
  - Shipping centroid ≈ `(20.0, -10.0)`

---

## Common Gotchas

- `Visualize Anomaly Objects` is interpreted as a string in `warehouse_render.py` (expects `"Enabled"`).
- Blender background runs may print geometry-nodes dependency cycle warnings; they can be noisy even when the run succeeds.
- When debugging: prefer checking artifacts (`bbox3d_vis`, `annotations`, `metadata`) first, then the saved blend snapshot if placement seems wrong.
- Anomaly objects must be properly tagged in collections to avoid being treated as warehouse infrastructure.

