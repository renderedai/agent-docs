# Rendered.ai Channel Development — Agent Guide

General guide for developing, testing, and deploying Rendered.ai channels. For renderer-specific patterns see `AGENT_BLENDER.md` (Blender) or `AGENT_DIRSIG.md` (DIRSIG). For graph YAML authoring see `AGENT_GRAPH.md`. For SDK/platform operations see `AGENT_SDK.md`.

---

## WHAT IS A CHANNEL

A channel is a Docker-based package that runs on the Rendered.ai platform to generate synthetic datasets. It consists of:

- **Node package** (`packages/<pkg>/`) — Python classes that implement each graph node. Each class has an `exec()` method the platform calls once per dataset run.
- **Graphs** (`graphs/*.yaml`) — YAML files that wire nodes together and set parameter values. One graph = one scene configuration.
- **Channel config** (`*.yml`) — declares the Docker image, channel name, channel ID, instance type, and volume mounts.
- **Dockerfile** (`.devcontainer/Dockerfile`) — the container image built and deployed by `anadeploy`.

---

## HOW `ana` WORKS (LOCAL DEV CONTAINER)

`ana` is a host wrapper script (`/usr/local/bin/ana`) that runs the channel inside a **dedicated Docker container**:

```
docker run <channel-image> ana --data /data --output /output <your-args>
```

### Path mappings (host → container)

| Host path | Container path | Notes |
|-----------|---------------|-------|
| `/workspace/ana` | `/ana` | **Live bind mount** — code/graph changes take effect immediately, no rebuild |
| `$WORKSPACE_OUTPUT` (default `/workspace/output`) | `/output` | All output files must be written here to persist to the host |
| `/workspace/volumes/<uuid>` | `/data/volumes/<uuid>` | Channel asset volumes (read-only) |
| `/workspace/ana/dirsig-bin` | `/DIRSIG_BIN` | Only when `DIRSIG_BIN_OVERRIDE=1` |

**Output to `/workspace/output`** (container's `/output`) is the only path that survives container exit. Any other path written by the container (including `/home/ubuntu/`) is ephemeral.

### Image rebuild

The container image is **rebuilt automatically** when any of these change:
- `.devcontainer/Dockerfile`
- `requirements.txt`
- `setup.py` / `setup.cfg` / `pyproject.toml` (any package)

Runtime code under `/workspace/ana` flows in live via the bind mount — editing a `.py` or `.yaml` file does **not** trigger a rebuild. Rebuilds are fingerprint-keyed and cached; the same fingerprint reuses the cached image instantly.

### Forwarded environment variables

`RENDEREDAI_API_KEY`, `RENDEREDAI_ORGANIZATION_ID`, `RENDEREDAI_EDITOR_ID`, `RENDEREDAI_ENVIRONMENT`, `DIRSIG_THREADS`, `DIRSIG_THREAD_BLOCK_SIZE`, `DIRSIG_BIN_OVERRIDE`

---

## CRITICAL SETUP

### Dockerfile location
- ✅ **CORRECT**: `/workspace/ana/.devcontainer/Dockerfile` — used by both `ana` (local) and `anadeploy` (platform)
- ❌ **WRONG**: `/workspace/ana/Dockerfile` — overwritten during deployment, not used

### Working directory
- `ana` is always invoked from the **host** (not inside the container). Use `cwd=/workspace/ana` in tooling — never `cd`.
- Inside the container, the working directory is `/ana` (= `/workspace/ana` on the host).

---

## ESSENTIAL COMMANDS

```bash
# Basic local test run — output lands in /workspace/output on the host
ana --graph /ana/graphs/my_graph.yaml \
  --logfile /output/ana.log \
  --loglevel INFO

# Reproducible run (fixed seed + run number)
ana --seed 42 --interp_num 0 \
  --graph /ana/graphs/my_graph.yaml

# Preview mode (single lightweight render, writes preview.png)
ana --preview --loglevel INFO

# Replay a platform run exactly (use graph.json + seed from downloaded dataset metadata)
ana --graph /path/to/dataset/graph.json \
  --seed <seed_from_metadata> \
  --interp_num <interp_num_from_metadata>
```

All output is written to `/output` **inside the container**, which maps to `/workspace/output` on the host (or `$WORKSPACE_OUTPUT` if set). Check results at `/workspace/output/` after a run.

### Logging
- Default `--loglevel` is `ERROR` — all `logger.info()` calls are silently dropped without `--loglevel INFO`.
- `logger.*` goes to the logfile and to platform per-run logs. `print()` goes to stdout only and is **not** captured by the platform. Always use `logger` for anything you need to see on the platform.
- Logfile path is a container-internal path; use `--logfile /output/ana.log` so it appears at `/workspace/output/ana.log` on the host.
- Nodes **must** write `preview.png` to `ctx.output` when `ctx.preview == True`.

### `interp_num`
- `ctx.interp_num` is the **run identifier** (zero-padded index), not a frame count.
- Use it for consistent output file naming across nodes: `f"{ctx.interp_num:010d}-{frame}-{sensor}.png"`

---

## NODE DEVELOPMENT

### Node class structure

```python
from anatools.lib.node import AnaNode

class MyNode(AnaNode):
    def exec(self, ctx):
        # All inputs are LISTS — always index with [0]
        value = self.inputs["Port Name"][0]
        number = float(self.inputs["Distance"][0])
        count = int(self.inputs["Number of Objects"][0])

        # Safe access with default
        setting = self.inputs.get("Optional Port", ["default value"])[0]

        # Write outputs
        self.outputs["Output Port"] = result
```

- **`self.inputs` is always a list** — direct access without `[0]` returns the list, not the value. This is a silent bug with no error.
- `self.inputs` merges `values:` and linked node outputs — linked values override `values:`.
- `self.outputs` keys must match `ports.outputs` names.
- `ctx.output` — write all output files here.
- `ctx.seed` — per-run integer seed.
- `ctx.random` — pre-seeded `numpy.RandomState`; use this instead of calling `np.random.*` directly.

### Optional input sentinel

When a user removes a link from an optional input in the platform UI, the platform sets that input to `""` (empty string) — it is **not** `None` and is **not** omitted. Passing `""` to downstream code will silently skip or crash.

```python
# ✅ Safe pattern for optional object lists
objects = [o for o in self.inputs.get("Objects", []) if o and o != ""]

# ✅ Safe pattern for optional scalar inputs
rotation = float((self.inputs.get("Rotation (deg)") or [0.0])[0])
```

### `nodeClass` vs `alias`

- **Graphs** use `nodeClass: My Node Display Name` — the **alias**, not the Python class name.
- **`remove_nodes`** and **`add_nodes`** in the channel `.yml` also use the alias.
- The Python class name is internal only.

```bash
# Find all registered aliases
grep -h "alias:" /ana/packages/*/nodes/*.yml | sed 's/.*alias: //' | sort
```

### Node schema YAML

Each node class has a companion YAML (`nodes/<Class>.yaml`):

```yaml
name: MyNodeClass           # Python class name
alias: My Node Display Name # Used in graphs and channel config
category: Scene
tooltip: Short description shown in the graph editor.
inputs:
  - name: Input Port
    description: What this port receives.
    default: null
    link: true              # required for the port to accept links
  - name: Setting Name
    description: A user-configurable value.
    default: 'some default'
    select:
      - option A
      - option B
outputs:
  - name: Output Port
    validation:
      numLinks: zeroOrMany  # optional; omit for default (one link)
```

**`link: true` is required** for any port that should accept connections from other nodes. Ports without it are values-only.

Valid `numLinks` values: `zero`, `zeroOrOne`, `zeroOrMany`, `one`, `oneOrMany`.

### `node_data.json` — never edit by hand

`node_data.json` is **auto-generated** by `anautils`. After any change to a node schema YAML, regenerate it:

```bash
anautils --channel my-channel.yml
```

Editing it directly will be overwritten on the next regeneration and may corrupt the node registry.

---

## ANATOOLS OBJECT TYPES

Nodes pass data between each other using specific anatools types. **Using the wrong attribute is a silent failure** — no error, just empty/broken data.

| Type | Source node | Attribute to use | Wrong attribute |
|------|-------------|------------------|-----------------|
| `FileObject` | `VolumeFile` | `.filename` | `.path` ❌ |
| `DirectoryObject` | `VolumeDirectory` | `.directory` | `.path` ❌ |

```python
# ✅ Defensive pattern
if hasattr(obj, 'filename'):
    path = obj.filename       # FileObject
elif hasattr(obj, 'directory'):
    path = obj.directory      # DirectoryObject
```

### Wiring a `VolumeFile` input

Declare in schema:
```yaml
inputs:
  - name: Scene File
    link: true
    validation:
      numLinks: one
```

Read in `exec()`:
```python
from anatools.lib.file_object import FileObject

scene_file = self.inputs["Scene File"][0]
path = scene_file.filename if isinstance(scene_file, FileObject) else str(scene_file)
```

Graph YAML wiring — `VolumeFile` values use `UUID:/path` notation:
```yaml
  VolumeFile_10:
    nodeClass: VolumeFile
    values:
      File: "708b0ca9-d81c-4679-b164-141418507830:/scenes/MyScene.blend"

  MyNode_20:
    nodeClass: My Node Display Name
    links:
      Scene File:
        - outputPort: File
          sourceNode: VolumeFile_10
```

The `VolumeFile` node resolves the UUID to the local volume path automatically.

---

## PACKAGE VOLUMES

### `package.yml` config

```yaml
volumes:
  my_volume: 'UUID-here'

objects:
  MyScene:
    filename: my_volume:scenes/my_scene.blend   # volume:path notation
```

### Loading assets

```python
from anatools.lib.package_utils import get_volume_path

# Non-blend files (textures, configs, data)
texture_path = get_volume_path('my_package', 'my_volume:textures/texture.png')

# Blend files via generator
from anatools.lib.generator import get_blendfile_generator
from anatools.lib.ana_object import AnaObject

generator = get_blendfile_generator("my_package", AnaObject, "MyScene")
scene_obj = generator.exec()
```

- **Never hardcode volume UUIDs** — always use `get_volume_path()`.
- Local path resolves to `./data/volumes/UUID/path`; platform resolves to `/data/volumes/UUID/path`.

---

## CHANNEL CONFIG

The channel YAML declares remotes and node overrides:

```yaml
channels:
  - name: My Channel Dev
    channelId: <UUID>
    instance: g5.4xlarge
    volumes:
      - volumeId: <UUID>
        mountPath: /path/in/container

remove_nodes:
  - Node Alias Here    # ✅ alias, not class name

add_nodes:
  - package: anatools
    name: RandomRandint    # Python class name from anatools package
    alias: Random Integer
    category: Common
    subcategory: Values
    color: "#B3B3B3"
```

- **`instance`** controls pod RAM/GPU.
- **`remove_nodes`** uses the **alias**, not class name. Using category (`Generators.AFV`) or class name silently does nothing.
- **`add_nodes`** is only needed for: anatools built-in nodes, overriding display properties, or nodes from external packages not in the channel's package list. All nodes in included packages are **automatically registered**.
- If `Schema for 'MyClass' not found`: check the `.yml` schema exists in `nodes/`, the package is in the channel config, and the `name` matches the Python class name exactly.

Full list of available anatools nodes: https://support.rendered.ai/development-guides/ana-software-architecture/the-anatools-package

---

## DETERMINISM

The platform passes a unique seed to each run via `ctx.seed`.

```python
# ✅ Use ctx.random (pre-seeded RandomState) for all randomness
indices = ctx.random.choice(len(files), size=3, replace=False)

# ✅ When you need a fresh Generator
rng = np.random.default_rng(ctx.seed)

# ❌ NEVER use unseeded sources
import random          # non-deterministic
np.random.choice(...)  # unseeded global state
time.time()            # entropy
uuid.uuid4()           # entropy
```

File system ordering is non-deterministic — **always sort** before iterating:
```python
files = sorted(glob.glob("*.png"))
```

---

## ANNOTATIONS

Output files written to `ctx.output` are returned to the user. Standard conventions:

| Directory | Contents |
|-----------|----------|
| `images/` | RGB renders (PNG or JPEG) |
| `masks/` | Semantic / instance mask images |
| `annotations/` | Per-image JSON annotation files |
| `metadata/` | Per-run metadata JSON |
| `preview.png` | Single image written during `--preview` mode |

For Blender-specific annotation pipeline (AnaScene, compositor masks, `write_ana_annotations()`), see `AGENT_BLENDER.md`.

---

## DEPLOYMENT WORKFLOW

### Pre-deploy sanity check
```bash
ana --preview --logfile /output/ana.log --loglevel INFO
```

Verify `/workspace/output/` contains expected files (`preview.png`, `annotations/`, `metadata/`, etc.).

### Deploy
```bash
anadeploy  # reads .devcontainer/Dockerfile and the channel *.yml config
```

### Platform constraints
- **No custom environment variables** in deployed channels — debug flags only work locally.
- **Debug features** must be surfaced as node inputs or gated by env vars that default off.
- The platform reads `graph.json` embedded in the uploaded dataset, not your local YAML. Always verify settings in the **downloaded** `graph.json` if behaviour on the platform differs from local.

### API key management

For nodes requiring external API keys, pass via a node input — **never hardcode**:

```python
api_key = self.inputs.get("API Key", [""])[0].strip()
if not api_key:
    raise ValueError("API Key input is required")
```

```yaml
inputs:
  - name: API Key
    description: Your API key for the external service.
    default: ""
```

### Local debug environment variables

Many channels support env-var gates for local development (skip expensive ops, save snapshots, force a specific camera, etc.). These **only work locally** — the platform cannot set custom env vars.

```bash
CHANNEL_DEBUG=1 ana --preview --output /home/ubuntu/debug_run --loglevel INFO
```

Check the channel-specific docs for available flags.

---

## COMMON MISTAKES & TROUBLESHOOTING

| Mistake / Symptom | Fix |
|---|---|
| Editing `/workspace/ana/Dockerfile` → deploy uses wrong image | Edit `/workspace/ana/.devcontainer/Dockerfile` |
| Output path survives run but files not found on host | Use `/output/<subdir>` inside container (maps to `/workspace/output` on host) |
| Code change not picked up in container | Only Dockerfile/requirements changes rebuild the image; `.py`/`.yaml` changes flow live via bind mount — no action needed |
| No `--loglevel INFO` → logfile and platform logs empty | Always pass `--loglevel INFO` |
| Non-deterministic results with same seed | Use `ctx.random`; `sorted()` all `glob`/`listdir` results; never call `import random` unseeded |
| Port missing `link: true` in schema YAML → "Unable to find target port" | Add `link: true` to the port definition |
| Stale node hash in graph YAML → same "Unable to find target port" | Update hash + `ports:` block to current version — see `AGENT_GRAPH.md` |
| `print()` for debugging → nothing in platform logs | Use `logger.info()` |
| `FileObject.path` or `DirectoryObject.path` → `AttributeError` or silent `None` | Use `.filename` / `.directory` |
| `self.inputs["Port"]` without `[0]` → `TypeError` on `float()`/`int()` | Always index inputs: `self.inputs["Port"][0]` |
| Optional unlinked input causes crash → platform sent `""` not `None` | Filter: `[o for o in inputs if o and o != ""]` |
| `remove_nodes` / `add_nodes` uses class name → silently ignored | Use the **alias** |
| Schema not found → `Schema for 'X' not found` | Check schema `.yml` exists, package listed in channel config, `name` matches Python class exactly |
| Asset not loading | Verify volume UUID and `get_volume_path()` usage; never hardcode UUIDs |
| Node not registered | Run `anautils --channel my-channel.yml` to regenerate `node_data.json` |
| Annotations missing for processed images | Filename convention mismatch — all nodes must use `{interp_num:010d}-{frame_current}-{sensor}` format |

---

## CLI TOOLS

### `ana` — run the channel locally

Runs `docker run <channel-image> ana ... ` under the hood. All paths are container-internal.

```
ana [--graph GRAPH] [--channel CHANNEL] [--seed SEED] [--interp_num N]
    [--batch_size N] [--preview] [--loglevel LEVEL] [--logfile PATH]
    [--output PATH] [--data PATH]
```

| Flag | Purpose |
|------|---------|
| `--graph` | Path to graph YAML/JSON (container path, e.g. `/ana/graphs/foo.yaml`) |
| `--seed` | Integer seed for reproducibility |
| `--interp_num` | Run index (used in output filenames) |
| `--preview` | Single lightweight render; node must write `preview.png` |
| `--loglevel` | `INFO` recommended; default is `ERROR` |
| `--logfile` | Use `/output/ana.log` so it persists to the host |

---

### `anadeploy` — deploy a channel to the platform

Must be run from `/workspace/ana`.

```
anadeploy [--channel CHANNELFILE] [--channelId UUID]
          [--service SERVICEFILE] [--serviceId UUID]
          [--verbose] [--noninteractive]
```

Reads `.devcontainer/Dockerfile` and the channel `*.yml` config. Builds and pushes the Docker image, then registers the new version on the platform.

---

### `anamount` — mount platform volumes locally

```
anamount [--channel CHANNEL] [--volumes ID1,ID2] [--workspaces ID1,ID2]
         [--path PATH] [--unmount] [--unmountall] [--environment ENV]
```

| Flag | Purpose |
|------|---------|
| `--channel` | Mount all volumes for a named channel |
| `--volumes` | Mount specific volume UUIDs (comma-separated) |
| `--workspaces` | Mount all volumes for workspace UUIDs (comma-separated) |
| `--path` | Mount point on the host (default: cwd) |
| `--unmount` / `--unmountall` | Unmount current / all mounts |

Volumes appear at `<path>/<uuid>/` and are symlinked from `/workspace/volumes/<uuid>`, which the `ana` wrapper auto-mounts into the container at `/data/volumes/<uuid>`.

---

### `anacreate` — scaffold a new node from an example

```
anacreate [--nodeName NAME] [--nodeType TYPE] [--package PKG]
          [--baseChannel CHANNEL] [--description DESC]
```

Generates a new node stub by querying the platform for examples from an existing channel.

---

### `anarules` — generate IDE rules for a workspace

```
anarules [--workspace ID] [--path PATH] [--services]
         [--windsurf] [--cursor] [--vscode] [--claude] [--gemini] [--theia]
```

Fetches workspace-specific rules (node lists, volume paths, channel schema) and writes them to the appropriate IDE config file. Run this when switching workspaces or after a new channel deploy.

---

### `anaprofile` — profile channel runs

Requires `matplotlib` (`pip install matplotlib`). Analyzes timing data from run logs. Not available in the default environment.

---

## MCP SERVERS

- **`anatools`** — Platform interaction (datasets, graphs, volumes, logs)
- **`graph`** — Local graph operations, node info, documentation

See `AGENT_SDK.md` for upload/dataset/log download patterns.
