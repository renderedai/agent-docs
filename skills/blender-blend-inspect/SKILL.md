---
name: blender-blend-inspect
description: Headlessly inspect a Blender `.blend` file (a saved render-debug snapshot or a source scene on a volume) inside a channel's Docker image to enumerate objects, measure world XY/Z/bound_box positions, list actions, inspect Geometry Nodes modifier inputs, or dump camera locations. Trigger when the user says "open the blend", "inspect the scene", "measure XY of <object>", "what's in this blend", "list actions", "what actions does this armature have", "list all cameras", "dump object positions", or asks for object coordinates/geometry that platform logs alone can't answer. Runs `blender -b <blend> --python <script>` inside the channel's Docker image via `docker run`, since `bpy` is not installed on the host.
argument-hint: "what's the world XY of the DeskA mesh?", "list every camera in this scene", "what actions exist on this armature?", "dump the Geometry Nodes inputs for ActionPath.003"
---

# Headless Blender inspection of a `.blend` file

For any question about object positions, mesh bounds, actions, Geometry Nodes inputs, or camera placements that platform/render logs don't answer directly. `bpy` is not installed on the host — Blender must run inside the channel's Docker image.

For general channel development, see `AGENT.md`. For Blender-specific patterns, see `AGENT_BLENDER.md`.

## When to use this skill

- "open the blend and check X"
- "measure the world XY / bound_box of `<ObjectName>`"
- "does the scene actually contain `<Prefix>_*` meshes?"
- "what actions are in `<library>.blend`?"
- "list all cameras in the scene"
- "dump Geometry Nodes modifier inputs for `<Object>`"
- Any downstream step of local dataset replay where a saved debug blend needs to be measured

## Prerequisites

- Docker on the host with the channel's dev image already built (most `ana`-based channel wrappers produce this on first run — look for a locally-tagged image alongside the registry baseline).
- Volume symlinks resolved for any volume-hosted blends (`/workspace/volumes/<uuid>/...`).
- Target blend accessible under an output/scratch directory (a saved debug snapshot) or a volume path (a source scene).

## Find the correct image tag

Don't assume the environment variable pointing at a channel's registry image is the one with your local code changes baked in — many channel dev workflows rebuild a local image with a distinct tag suffix (e.g. a dev/fingerprint tag) on top of the registry baseline. Check `docker images` for the channel's repository name and prefer the most recently built local tag over the registry baseline; the registry baseline lacks your uncommitted local changes and may also require registry auth to pull.

## Canonical `docker run` invocation

```bash
docker run --rm \
    -v /workspace/<channel-repo>:/ana \
    -v /workspace/output:/output \
    -v /workspace/volumes:/data/volumes \
    --entrypoint blender \
    "$DEV_IMG" \
    -b <blend-path-container-side> \
    --python <script-path-container-side> 2>&1 | tail -40
```

- Container-side paths: `/output/...` for host `/workspace/output/...`; `/data/volumes/<uuid>/...` for host `/workspace/volumes/<uuid>/...`; `/ana/...` for the channel repo.
- `--entrypoint blender` overrides the wrapper's default entrypoint so Blender runs directly against the blend.
- Runs in `--background` (`-b`) — no GUI or GPU needed for read-only inspection.
- Dependency-cycle warnings from Geometry Nodes (`dependency cycle detected: ...`) are harmless noise for read-only inspection. Ignore.
- Large baked-texture blends can take tens of seconds just to load — budget for it.

## Canned inspection scripts

Keep reusable scripts in the channel repo so both host and container see them at the same relative path.

### 1. Enumerate objects near a reference point (finds nearby named meshes)

```python
# inspect_near_point.py
import bpy, math, mathutils, os

OUT = "/output/inspect"
os.makedirs(OUT, exist_ok=True)
REF_XY = (0.0, 0.0)          # ← EDIT: reference point in world XY
NEAR_R, MID_R = 5.0, 10.0
FRAME = 1                     # ← EDIT: frame_current for animation-dependent bounds

def bb(o):
    corners = [o.matrix_world @ mathutils.Vector(c) for c in o.bound_box]
    xs=[c.x for c in corners]; ys=[c.y for c in corners]; zs=[c.z for c in corners]
    return min(xs), min(ys), max(xs), max(ys), min(zs), max(zs)

def dist_pt_bbox(px, py, bx0, by0, bx1, by1):
    return math.hypot(max(bx0-px, 0, px-bx1), max(by0-py, 0, py-by1))

scene = bpy.context.scene
scene.frame_set(FRAME); bpy.context.view_layer.update()

near=[]; mid=[]; family={}
for o in bpy.data.objects:
    if o.type not in ("MESH", "EMPTY", "CURVE", "CAMERA"):
        continue
    bx0,by0,bx1,by1,bz0,bz1 = bb(o)
    d = dist_pt_bbox(REF_XY[0], REF_XY[1], bx0, by0, bx1, by1)
    entry = (o.name, o.type, round((bx0+bx1)/2,2), round((by0+by1)/2,2),
             [round(bx0,2),round(by0,2),round(bx1,2),round(by1,2)],
             [round(bz0,2),round(bz1,2)], round(d,2))
    (near if d <= NEAR_R else mid if d <= MID_R else []).append(entry)
    fam = o.name.split(".")[0]
    family[fam] = family.get(fam, 0) + 1

near.sort(key=lambda x: x[-1]); mid.sort(key=lambda x: x[-1])
with open(f"{OUT}/near_ref_f{FRAME}.md","w") as f:
    f.write(f"# frame={FRAME}\n\nRef XY {REF_XY}\n\n## Name families (top 30)\n")
    for n,c in sorted(family.items(), key=lambda x:-x[1])[:30]:
        f.write(f"- `{n}` × {c}\n")
    f.write(f"\n## Within {NEAR_R} m ({len(near)})\n| Name | Type | Center | BBox XY | Z | Dist |\n|---|---|---|---|---|---:|\n")
    for n,t,cx,cy,bxy,z,d in near:
        f.write(f"| `{n}` | {t} | ({cx},{cy}) | {bxy} | {z} | {d} |\n")
print(f"Wrote {OUT}/near_ref_f{FRAME}.md ({len(near)} near, {len(mid)} mid)")
```

### 2. List actions in a blend

```python
import bpy
for a in sorted(bpy.data.actions, key=lambda a: a.name):
    print(f"{a.name}\tframes={a.frame_range[0]:.0f}-{a.frame_range[1]:.0f}")
```

Useful against an animation-library blend to verify action names before wiring them into channel code.

### 3. Dump Geometry Nodes modifier inputs by object name

Socket identifiers (`Socket_8`) are not stable across blend versions — always look up by name via `node_group.interface.items_tree`:

```python
import bpy

def gn_inputs(obj, contains="GeometryNodes"):
    out = {}
    for mod in obj.modifiers:
        if contains not in mod.name: continue
        ng = getattr(mod, "node_group", None)
        if not ng or not hasattr(ng, "interface"): continue
        for item in ng.interface.items_tree:
            if hasattr(item, "identifier"):
                out[item.name] = mod.get(item.identifier)
    return out

for name in sorted(o.name for o in bpy.data.objects if o.name.startswith("<Prefix>")):
    obj = bpy.data.objects[name]
    print(name, "->", gn_inputs(obj))
```

Use this to read node-group inputs by human-readable name (never trust `Socket_N` — read via `item.name.lower()` and substring-match, e.g. `"start" in name`).

### 4. Measure distance between a reference point and nearby meshes

A generally useful spatial-QA pattern: given a world XY (e.g. a placement endpoint from a render log or annotation), find nearby meshes in a given Z-band and report distance. Handy for diagnosing "object A overlaps object B" bugs regardless of channel domain.

```python
import bpy, mathutils
FRAME = 1
REF_XY = (0.0, 0.0)   # ← EDIT: point of interest, e.g. from a log/annotation

def bb_xy(o):
    corners = [o.matrix_world @ mathutils.Vector(c) for c in o.bound_box]
    xs=[c.x for c in corners]; ys=[c.y for c in corners]; zs=[c.z for c in corners]
    return min(xs), min(ys), max(xs), max(ys), min(zs), max(zs)

def dist(px, py, bx0, by0, bx1, by1):
    import math
    return math.hypot(max(bx0-px, 0, px-bx1), max(by0-py, 0, py-by1))

bpy.context.scene.frame_set(FRAME)
best = []
for o in bpy.data.objects:
    if o.type != "MESH": continue
    bx0,by0,bx1,by1,bz0,bz1 = bb_xy(o)
    if not (0.65 <= bz1 <= 1.15): continue   # ← EDIT: Z-band of interest
    d = dist(REF_XY[0], REF_XY[1], bx0, by0, bx1, by1)
    if d < 1.0:
        best.append((d, o.name, [round(bx0,2), round(by0,2), round(bx1,2), round(by1,2)], round(bz1,2)))
best.sort()
for d, n, xy, z in best[:20]:
    print(f"{d:.2f}m  {n:30s}  bbox={xy}  z_top={z}")
```

## What NOT to do

- Do NOT `docker exec` into a running replay/render container — start a fresh `docker run --rm`. Most channel containers are single-purpose and terminate when the graph finishes.
- Do NOT run Blender with `--gpu` or GL for pure read/inspection queries — `-b` (background) is enough and avoids contending for GPU with an active render.
- Do NOT assume the registry-baseline image has your local code changes — check for a locally-built dev tag first.
- Do NOT trust `matrix_world.translation` alone for object footprint — objects' pivots often sit at origin while geometry is offset. Always compute from `bound_box` corners multiplied by `matrix_world`.
- Do NOT hardcode volume UUIDs in inspection scripts — pass paths via the `-b <path>` argument or resolve `<vol>:file.blend`-style logical paths inside the script.

## Output convention

Write markdown/JSON reports under an `output/inspect/` scratch directory so they land outside the channel repo and don't get accidentally committed.

## Error handling

| Symptom | Likely cause | Action |
|---|---|---|
| `ModuleNotFoundError: No module named 'bpy'` on the host | Trying to run the script directly instead of inside the container | Run via `docker run ... --entrypoint blender ... -b <blend> --python <script>` |
| Docker image pull fails / auth error | Using the registry-baseline tag, not a locally built dev tag | Check `docker images` for a local tag with your code changes already baked in |
| Script prints nothing / empty report | `FRAME` doesn't match where the object is actually positioned in an animated scene | Set `FRAME` to the frame the render/log actually used before measuring |
| Distances look wrong for a pivoted object | Used `matrix_world.translation` instead of `bound_box` | Recompute footprint from `bound_box` corners transformed by `matrix_world` |

## See also

- `AGENT_BLENDER.md` — general Blender scene/material/lighting/camera patterns for Rendered.ai channels.
- `AGENT.md` — general channel development guide.
