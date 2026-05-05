# DIRSIG Patterns for Rendered.ai Channel Development

DIRSIG-specific patterns and best practices for AI agents working on Rendered.ai channels that use [DIRSIG](https://dirsig.cis.rit.edu/) as the renderer/simulator (e.g. the open-source `dirsig-channel` reference implementation).

For general channel development (non-DIRSIG-specific), see: `AGENT.md`.
For SDK/platform operations, see: `AGENT_SDK.md`.
Blender patterns in `AGENT_BLENDER.md` **do not apply** to DIRSIG channels — there is no `bpy`, no compositor, no `AnaScene`-based mask pipeline. Annotations here are derived from DIRSIG truth bands, not Blender passes.

---

## LEARNING RESOURCE: `dirfm` DEMOS

`packages/dirfm/demos/` contains a `pytest` suite (`test_*.py`) where each test reconstructs a public DIRSIG demo end-to-end with the same `dirfm` API the channel nodes use. Treat it as the **canonical reference** for how to wire scenes, sensors, motion, atmospheres, weather, ephemeris, materials, and mediums (water, clouds, plumes).

When adding a new node or feature, find the closest demo and copy its assembly pattern. Run a single demo with:
```bash
DIRSIG_HOME=/DIRSIG_BIN pytest -v packages/dirfm/demos/test_<Name>.py
```
Each test writes inputs to `demos/<Name>_input/` and ENVI/PNG/GIF outputs to `demos/<Name>_output/` for inspection.

---

## ARCHITECTURE

A DIRSIG channel is built from two cooperating layers:

- **`dirfm` (DIRSIG File Maker)** — pure-Python driver library (`packages/dirfm/dirfm/`). Programmatically builds `glist`, `scene`, `platform_sensor`, `flexible_motion`, `atmosphere`, `tasks`, and `ephemeris` input files, then shells out to the `dirsig` binary.
- **`dirsig_pkg`** — the Rendered.ai node package (`packages/dirsig_pkg/dirsig_pkg/`). Node `exec()` methods use `dirfm` to assemble a simulation and invoke it from the `DIRSIG5` simulator node.

Node → object mapping:
| Node category | Returns (dict) | Backed by |
|---|---|---|
| Object nodes (Bench, House1, …) | `{"<Name> Object": generator}` | `ObjectDirsigGenerator` → `AnaDirsigObject` (wraps `dirfm.glist.Object`) |
| Scene nodes (Desert Highway, Sierra Nevada, …) | `{"Scene": {"sceneObjects", "timezone", "metadata", "tags"}}` | `dirfm.scene.SCENE` + `dirfm.glist.GLIST` |
| Platform nodes (Drone, WorldView3, SuperDove, …) | `{"Sensor": {"sensor", "motion", "convert_args"}}` | `dirfm.platform_sensor` |
| Motion nodes (FlexMotion, Waypoint…, LookAt…) | `{"LocationEngine"\|"OrientationEngine"\|"Motion": obj}` | `dirfm.flexible_motion` |
| `DIRSIG5` (`Simulate`) | `{}` (terminal) | runs `dirsig` binary, writes `ctx.output` |

---

## CRITICAL SETUP

### DIRSIG binary
- The binary must be available on `PATH` inside the container. The Dockerfile copies `dirsig-bin/` to `/DIRSIG_BIN` and prepends `/DIRSIG_BIN/bin` to `PATH`. See `/ana/Dockerfile` and `/ana/.devcontainer/Dockerfile` — both must stay in sync (contrary to the Blender channel pattern where only `.devcontainer/Dockerfile` is used, this channel's root `Dockerfile` is also meaningful for deployment).
- Users without DIRSIG access cannot build or run this channel. Surface this clearly in any user-facing docs (see top-level `README.md`).

### Runtime directories
- **DIRSIG input**: `/tmp/dirsig_input` — `dirfm` writes `.glist`, `.scene`, `.platform`, `.tasks`, `.atm`, `.jsim` here
- **DIRSIG output**: `/tmp/dirsig_output` — ENVI `.img` / `.hdr` raw radiance + truth
- **Channel output**: `ctx.output` — RGB PNGs, annotations JSON, metadata JSON (and optionally raw radiance / ENVI truth, gated by node inputs)

These `/tmp/dirsig_*` paths are hard-coded in `packages/dirsig_pkg/dirsig_pkg/nodes/simulators.py`. If you parallelize or re-run, you must clean or redirect them.

### Package volume
`dirsig_pkg/package.yml` lists one volume (`dirsig-shared`) and maps friendly object names to `.glist` files under `BundleInventory-*/`. Always reference assets via `get_volume_path('dirsig_pkg', 'dirsig-shared:path/to.glist')` — **never hardcode UUIDs**.

---

## SIMULATION LIFECYCLE (DIRSIG5 node)

The canonical flow in `simulators.py::Simulate.exec()`:

```python
from dirfm.dirsig import DIRSIG
from dirfm import TASKS
import dirfm.atmosphere as atmos

dirsig = DIRSIG(Path("/tmp/dirsig_input"), Path("/tmp/dirsig_output"))
dirsig.set_seed(ctx.seed)                    # DETERMINISM — use ctx.seed, never random.seed

# Scene bundle from a scene node
scene_bundle  = self.inputs["Scene"][0]
dirsig.set_scene(scene_bundle["sceneObjects"][0])

# Platform bundle from a sensor node
platform_assets = self.inputs["Sensor"][0]
sensor = platform_assets["sensor"]
dirsig.set_platform(sensor)
dirsig.set_motion(platform_assets["motion"])

# Tasks
ref_datetime = ...  # ISO string or datetime, shifted by scene timezone
tasks = TASKS(ref_datetime).add_start_stop(start_s, end_s)
dirsig.set_tasks(tasks)

# Ephemeris (plugin instance or preset string)
dirsig.set_ephemeris(ephemerisPlugin)

# Atmosphere (open channel always uses BasicAtmosphere + SimpleRadiativeTransfer)
atm = atmos.BasicAtmosphere()
atm.set_radiative_transfer(atmos.SimpleRadiativeTransfer(250))
atm.set_weather(Path("$DIRSIG_HOME/lib/data/weather/jun2392.wth"))
dirsig.set_atmosphere(atm)

dirsig.run(convergence="20,100,1e-6", max_nodes="4")
```

### Convergence / performance knobs
Set on `dirsig.run(...)`:
- **Production default**: `convergence="20,100,1e-6", max_nodes="4"`
- **Preview** (`ctx.preview=True`): `convergence="3,3,0", max_nodes="1"` (+ `capture_duration=0`, `schedule="simulation"`)
- **Debug** (when `"debug" in ctx.output`): `log_level="debug", convergence="10,10,0", max_nodes="1"`

Raise `convergence` first dimension (photon pathlen) and second (samples) for quality; lower for iteration speed.

### Preview contract
`Simulate` handles `ctx.preview` by:
1. Forcing `capture_duration=0` (single-frame capture)
2. Forcing sensor `schedule="simulation"` (single output file per instrument)
3. Copying the first produced RGB PNG to `{ctx.output}/preview.png` and returning early

Every platform node must respect `ctx.preview` and fall back to `schedule="simulation"`, otherwise multi-frame scheduling will fight preview mode.

---

## TRUTH COLLECTION & ANNOTATIONS

Unlike Blender/AnaScene channels, annotations come from **DIRSIG truth bands**, not compositor passes.

### Registering truth collectors
Inside `Simulate.exec()`, the first focal plane of the first instrument is decorated with:
```python
fp.add_truth_collection("Intersection")
fp.add_truth_collection("material")
fp.add_truth_collection("Abundance", tags=scene_bundle["tags"])
```
`tags` is the list of **per-instance names** accumulated by scene nodes (e.g. `Bench_1`, `Bench_2`, …). Each tag produces one "Abundance" band in the truth ENVI file — one band per object instance.

When authoring a new scene node, you **must** collect instance names and pass them via the scene bundle `"tags"` field, otherwise per-object masks will be empty:
```python
for objInstance in ana_object.root.get_instances():
    names.append(objInstance.get_name())
...
return scene([sceneObj], location=..., meta=meta, tags=names)
```

### From ENVI truth to annotations
After `dirsig.run()`, for each produced capture file the simulator:
1. Opens the matching `*_truth.hdr` via `spectral.open_image`
2. Iterates bands, keeps those whose name contains `"Abundance"`
3. Reads each band as a 2D mask and calls `mask_to_annotation(mask)` (see `dirsig_pkg/lib/mask.py`) which returns:
   - `bbox = [x, y, w, h]`
   - `segmentation` — convex hull vertices (flattened `[x,y,x,y,...]`)
   - `segmentation_fill` — same polygon (clean, non-self-intersecting)
   - `segments` — per connected-component polygons (for thin parallel objects like cables)
4. Writes a combined `AnnotationsMetadata` JSON to `ctx.output/annotations/<base>-ana.json` and `ctx.output/metadata/<base>-metadata.json`

No separate mask PNG directory is produced. If you need raster masks, save them explicitly from the ENVI band before discarding.

### Abundance band naming
The band string looks like `"'Bench_1_0.43m'"`-style tokens. The simulator derives:
```python
name       = band.split("'")[1]                   # "Bench_1"
objectType = "_".join(t for t in name.split("_") if not t.isdigit())
```
Instance names must therefore embed the object type as **non-digit tokens separated by underscores** — `AnaDirsigObject.__init__` already enforces `"{ObjectType}_{instance}"`, so don't fight this convention.

---

## OBJECT SYSTEM

### Generators, not immediate objects
Object nodes return *generators*, not instantiated DIRSIG objects. Scenes and modifiers call `.exec()` on them at scene-build time. This lets modifiers (Pose, Cluster, Scale, Dynamize) wrap and mutate generators before instantiation.

```python
# Canonical object node body
from dirsig_pkg.lib.object import AnaDirsigObject, get_file_generator

def exec(self):
    gen = get_file_generator("dirsig_pkg", AnaDirsigObject, "Bench")
    return {"Bench Object": gen}
```

### Mixed generator + File/Directory inputs
Scene and cluster nodes accept user-supplied `.glist` files via `VolumeFile` / `VolumeDirectory`. Normalize them with `file_to_objgen` before triggering:
```python
from dirsig_pkg.lib.object import file_to_objgen, AnaDirsigObject
generators = file_to_objgen(self.inputs["Objects"], AnaDirsigObject)
for gen in generators:
    ana_obj = gen.exec()
    ...
```
Only `.glist` extensions are currently supported; others are silently skipped (logged at INFO).

### Instance types (glist)
`AnaDirsigObject` exposes three mutually exclusive instance modes via `root._instance`:
- **Static** (default): `add_static_instance(name, anchor)` — single fixed placement
- **Static Binary File**: `set_binfile_instance(locations_filepath, name, anchor_name)` — used by `ClusterObjects` for many anchored placements sourced from a binary locations file
- **Dynamic**: `set_dynamic_instance(motion, name)` — moving objects driven by a `FlexMotion`

Setting binfile or dynamic clears any prior instances. Downstream modifiers that expect Static instances must run **before** Dynamize / Cluster.

### Terrain alignment
Call `align_objects_with_terrain(ana_object, elevation_glist_path, anchor_name)` after adding each object to the scene:
- Sets `anchor` on all instances when `match_elevation=True` (makes Binary/BinaryFile instances anchor to terrain)
- For Static instances, queries the terrain elevation HDF5 at `(x, y)` and offsets `z`
- When `match_slope=True`, also aligns the object's Z-axis to the surface normal via `align_directions`

Set these flags on the object (not the modifier): `ana_object.match_slope`, `ana_object.match_elevation`.

---

## COORDINATE FRAMES

Motion nodes accept three frames via the `Frame` dropdown:
- **`scene`** → `ENUFrame(x, y, z)` — East, North, Up, meters, relative to scene origin (default)
- **`geodetic`** → `GeodeticFrame(lat, lon, alt)` — degrees, degrees, meters
- **`ecef`** → `ECEFFrame(x, y, z)` — Earth-Centered-Earth-Fixed meters

Scene origins are set in the scene node (e.g. `sceneObj.set_origin(GeodeticFrame(38.34, -120.01, 1984))` for Sierra Nevada). Timezone is heuristically derived from longitude: `timezone = round(lon / 15)` — keep this when authoring new scenes, and subtract it from `ref_datetime` in the simulator so local scene time matches UTC-based DIRSIG input.

---

## MOTION ENGINES

A `FlexMotion` = `LocationEngine` + `OrientationEngine`. If either input to `FlexMotion` is missing, defaults are provided (origin + look-at (0, 0.1, 0)).

### Location engines
- **`FixedLocationEngine`** — stationary
- **`StraightLineLocationEngine`** — constant altitude, constant speed, heading in degrees
- **`WaypointsLocationEngine`** — list of `(time_s, [x,y,z])` TimeEntries; **requires ≥2 entries** — if only one TimeEntry is wired in, the node auto-duplicates it 100 s later

### Orientation engines
- **`LookAtOrientationEngine`** — points the sensor's forward axis at another `LocationEngine`, with a user-supplied up vector
- **`EulerOrientationEngine`** — explicit Euler angles over time; also needs ≥2 entries (auto-duplicates)
- **`VelocityOrientationEngine`** — derives orientation from velocity direction
  - **Known bug**: fails if task start == stop (no velocity sample). Use Capture Duration > 0 or pick a different engine.

### TimeEntry convention
`TimeEntry` nodes produce `(time_seconds, [x, y, z])` tuples. Order of entries into a Waypoint engine doesn't matter — they are sorted internally.

---

## PLATFORMS / SENSORS

Each platform node (`Drone`, `WorldView3`, `SkySat`, `SuperDove`, `CustomCamera`, …) returns a bundle consumed by `Simulate`:
```python
return {"Sensor": {
    "sensor":       platformSensorObject,  # dirfm.platform_sensor.* instance
    "motion":       flex_motion_input,
    "convert_args": ["--bands=2,1,0", "--percent=0", "--per_band"],  # passed to image_tool convert
}}
```

### `convert_args` — RGB chip extraction
After DIRSIG writes multi-band ENVI radiance, the simulator runs:
```
image_tool convert <convert_args> --output=<rgbFilepath> <enviFilePath>
```
`convert_args` picks which bands become R, G, B (via `--bands=r,g,b`) and optional stretch (`--percent`, `--sigma`, `--per_band`). Band indices are **zero-based into the platform's band list**. When adding a new sensor, match `--bands` to the filter indices you want visible in the RGB preview — getting this wrong produces gray or inverted-color outputs with no error.

### Truth bands from node inputs
Platforms expose checkboxes like `"Collect Geolocation"`, `"Collect Intersection"`, `"Collect Shadow"` that append to `truth_bands` passed to the sensor constructor. These are **per-platform** truth bands (in addition to the Abundance/material bands added by the `Simulate` node itself).

### File schedule
`schedule` controls how DIRSIG partitions output files:
- `"simulation"` — one file per run (preview, single-capture)
- other scheduler strings — per-instrument-controlled multi-file output

Force `"simulation"` when `ctx.preview`.

---

## DETERMINISM

Same rules as the general guide, plus DIRSIG specifics:
- **Always** call `dirsig.set_seed(ctx.seed)` before `dirsig.run()`
- Use `ctx.interp_num` for per-run naming (DIRSIG output files already include a run index, but your post-processing can add `f"{ctx.interp_num:010d}"` prefixes if needed)
- For object placement RNG (e.g. cluster positions), use `ctx.random` — `ClusterObjects` already does this; new modifiers must follow suit
- DIRSIG itself is deterministic given the same input files, seed, and convergence — if outputs drift across runs on the same machine, suspect:
  1. Unsorted `glob()` results feeding file lists → always `sort()` after glob
  2. Dict iteration order over `ctx.packages[...]` entries
  3. `random` / `np.random.*` module-level calls

---

## OUTPUT LAYOUT

A successful run produces under `ctx.output`:
```
images/<base>.png                      # RGB from image_tool convert
annotations/<base>-ana.json            # bbox + segmentation per object instance
metadata/<base>-metadata.json          # scene, ephemeris, atmosphere, capture_time
radiance/<base>.img  .hdr              # only if 'Save Radiance' = True
envi_truth/<base>_truth.img  .hdr      # only if 'Save ENVI Truth' = True
preview.png                            # only when ctx.preview=True (copy of first RGB)
```

`<base>` is derived from `fp.get_capture_filename().split('.')[0]` — format depends on the sensor's file scheduler.

---

## COMMON GOTCHAS

- **Empty annotations, but images render fine** → scene node didn't accumulate `tags` from object instance names, so no `Abundance` truth bands were collected.
- **Black RGB PNG** → wrong `--bands=` indices in the platform's `convert_args`, or the platform has fewer bands than requested.
- **`FileNotFoundError` in `image_tool convert`** → DIRSIG crashed or wrote to a different filename; check `/tmp/dirsig_output/` contents and DIRSIG stdout (surfaced via `logger.info`).
- **"DIRSIG_HOME not set"** → `dirsig-bin/` missing or not on `PATH`; verify the Dockerfile copied it and `which dirsig` works in the container.
- **Waypoint/Euler engine error: "at least two entries required"** → wire two TimeEntries, or rely on the auto-duplicate (ok for static captures, bad for multi-second flights).
- **`VelocityOrientationEngine` division-by-zero** → task start == stop; increase `Capture Duration (s)` or switch to `LookAt`.
- **Objects float above terrain** → `match_elevation=False` or the scene node didn't call `align_objects_with_terrain` for a Static instance.
- **Objects clip through terrain at scene edges** → terrain elevation glist doesn't cover the (x, y); `elevation()` returns 0, object sits at scene z=0.
- **`Simulate` takes forever** → `max_nodes` too high for the host, or convergence too tight for iteration; use `ctx.preview` or debug mode while developing.
- **Graph reference names vs class names** — as with all channels, graphs reference the node **alias** (e.g. `DIRSIG5`, `Desert Highway`, `Bench`), not the Python class name (`Simulate`, `DesertHighwayScene`, `BenchNode`).
- **`/tmp/dirsig_input` and `/tmp/dirsig_output` are not cleaned between runs in the same process** — harmless for single-job runs, but something to keep in mind when debugging "stale file" artifacts.

---

## QUICK REFERENCE

### Run the reference graph
```bash
# From /ana
ana --graph graphs/default.yml --output /home/anadev/dirsig_test
# Preview (fast):
ana --preview --graph graphs/default.yml --output /home/anadev/dirsig_test
# Deterministic:
ana --seed 42 --interp_num 0 --graph graphs/default.yml --output /home/anadev/dirsig_test
```

### Inspect a produced ENVI file headlessly
```python
from spectral import open_image
img = open_image("/tmp/dirsig_output/foo_truth.hdr")
print(img.metadata["band names"])   # list Abundance/material/Intersection bands
band0 = img.read_band(0)
```

### List all DIRSIG node aliases in this channel
```bash
grep -h "alias:" /ana/packages/dirsig_pkg/dirsig_pkg/nodes/*.yml | sed 's/.*alias: //' | sort
```
