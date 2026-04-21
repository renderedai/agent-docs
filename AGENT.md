# Rendered.ai Channel Development Guide for AI Agents

This document provides structured guidance for AI agents working on Rendered.ai channel development. Follow sections in order for optimal workflow.

---

## CRITICAL SETUP REQUIREMENTS

### Dockerfile Location (DEPLOYMENT BLOCKER)
- âœ… **CORRECT**: `/ana/.devcontainer/Dockerfile` (used by `anadeploy`)
- âŒ **WRONG**: `/ana/Dockerfile` (overwritten during deployment)
- **Action**: Always edit `.devcontainer/Dockerfile` for deployment fixes

### Output Directory for Agents
- âœ… **REQUIRED**: Use `/home/anadev/<subdir>` for all ana outputs
- âŒ **WRONG**: Default `output/` (gitignored, agents can't access)
- **Action**: Always create directory first if using `--logfile`

```bash
mkdir -p /home/anadev/test_render
ana --output /home/anadev/test_render --logfile /home/anadev/test_render/ana.log --loglevel INFO
```

---

## ESSENTIAL COMMANDS

### Ana Command Patterns
```bash
# Basic test run (from /ana directory)
ana --output /home/anadev/test_render

# Preview mode (faster iteration)
ana --preview --output /home/anadev/test_render

# Specific graph
ana --graph /ana/graphs/default.yml --output /home/anadev/test_render

# Reproducible run (for debugging)
ana --seed 12345 --interp_num 0 --output /home/anadev/test_render

# With logging (MUST include --loglevel to capture logger.info() calls)
ana --output /home/anadev/test_render --logfile /home/anadev/test_render/ana.log --loglevel INFO
```

### Logging (Critical)
- **Default `--loglevel` is `ERROR`** — logger.info() calls are silently dropped unless you set `--loglevel INFO`
- **`--logfile`** creates the file but writes nothing without the correct loglevel
- **`print()` goes to stdout** (visible locally, not captured on platform)
- **`logger.info()` goes to logfile** (captured on platform, requires `--loglevel INFO` locally)
- **Always use `logger`** for information you need on the platform; use `print()` only for local debug noise

### Working Directory Rule
- **ALWAYS** run `ana` from `/ana` (repo root)
- **NEVER** use `cd` in commands - use `cwd` parameter instead

### Preview Mode Requirements
- Nodes **MUST** write `preview.png` to `ctx.output` when `ctx.preview=True`
- Use for faster iteration during development

---

## DETERMINISM & DEBUGGING

### Randomization Control
```python
# Use these for deterministic results:
ctx.seed          # Base seed (set by --seed)
ctx.interp_num    # Run identifier (set by --interp_num)  
ctx.random        # Seeded NumPy RandomState

# NEVER use these (non-deterministic):
import random     # Python's random module
np.random.*       # Unseeded NumPy random

# When selecting from file lists, sort first for deterministic ordering:
image_files.sort()
indices = ctx.random.choice(len(image_files), size=3, replace=False)
```

### Reproducible Testing Pattern
```bash
# 1. Run locally with known seed/interp_num
ana --seed 12345 --interp_num 0 --graph /path/to/graph.yml --output /home/anadev/local_test

# 2. Run same on platform, download results
# 3. Compare artifacts (metadata, annotations)
```

### Determinism Validation Graphs
- Look for `graphs/Determinism-*.yml` or similar test graphs in your channel
- Use minimal graphs that test RNG determinism and basic rendering

---

## ANATOOLS OBJECT TYPES

Nodes pass data using specific anatools types. **Using the wrong attribute is a silent failure** — no error, just empty data.

| Type | Source Node | Attribute | Helper |
|------|------------|-----------|--------|
| `FileObject` | VolumeFile | `.filename` | — |
| `DirectoryObject` | VolumeDirectory | `.directory` | `.get_files()` (excludes `.anameta`) |

```python
# ❌ SILENT FAILURE — .path does not exist on either type
file_input.path        # FileObject uses .filename
dir_input.path         # DirectoryObject uses .directory

# ✅ Defensive pattern
if hasattr(obj, 'filename'):     path = obj.filename      # FileObject
elif hasattr(obj, 'directory'):  path = obj.directory      # DirectoryObject
```

Return objects between nodes:
```python
from anatools.lib.file_object import FileObject
return {"Output File": FileObject("/path/to/output.png")}  # or {} for terminal nodes
```

---

## NODE DEVELOPMENT

### Schema Structure
```yaml
schemas:
  MyNodeClass:                    # Python class name (internal)
    alias: My Node Display Name   # Public name (used in graphs)
    category: Scene
    help: my_package/MyNode.md    # Auto-resolves to docs/MyNode.md
```

### Help Documentation System
- **Format**: `help: package_name/DocumentName.md`
- **Resolution**: System looks in `package_name/docs/` automatically
- **No `/docs/` needed** in the path specification
- **Markdown support** for rich documentation in node UI

### Node Input Access Pattern
**CRITICAL**: Node inputs are always lists - use `[0]` to access the value:

```python
# ✅ CORRECT - All inputs are lists
camera_angle = self.inputs["Camera Angle"][0]
distance = float(self.inputs["Distance"][0])
num_objects = int(self.inputs["Number of Objects"][0])

# ❌ WRONG - Direct access without [0]
camera_angle = self.inputs["Camera Angle"]  # This is a list!
distance = float(self.inputs["Distance"])   # TypeError!
```

**Examples from SATRGB**:
- `float(self.inputs["Scale X"][0])`
- `int(self.inputs["Number of Objects"][0])`
- `self.inputs["Options"][0]`

### Critical Rules
- **Graphs reference `alias`**, not class name: `nodeClass: My Node`
- **Channel config uses `alias`** for `remove_nodes`
- **Alias collisions**: Multiple packages can have same alias (last loaded wins)
- **Both `remove_nodes` and `nodeClass` use alias**, never Python class name
- **Node name in graph** (e.g., `Render for Processing_13`) is just an identifier — `nodeClass` determines which Python class runs

### Output Link Validation
Use `numLinks` on outputs to control whether downstream connections are optional or required. Valid values: `zero`, `zeroOrOne`, `zeroOrMany`, `one`, `oneOrMany`. See [Graph Validation docs](https://support.rendered.ai/development-guides/ana-software-architecture/graph-validation#number-of-links-rules).

```yaml
# Optional output - node works as terminal OR pipeline node
outputs:
- name: Rendered Image
  validation:
    numLinks: zeroOrMany    # No downstream node required
```

### Terminal Nodes Pattern
Some nodes are designed as pipeline endpoints (like GenericRender):

```python
# Terminal nodes return empty dict and save to ctx.output
def exec(self) -> Dict[str, Any]:
    # Process inputs and save results to files
    output_path = os.path.join(ctx.output, "images", filename)
    # ... save file ...
    
    # Terminal node - no outputs returned
    return {}
```

```yaml
# Terminal node schema has empty outputs
outputs: []
```

**Examples**: GenericRender (when no downstream nodes are connected), FalStyleTransfer

### Finding Node Aliases
```bash
grep -h "alias:" /ana/packages/*/nodes/*.yml | sed 's/.*alias: //' | sort
```

---

## PACKAGE VOLUMES

### Volume Configuration (package.yml)
```yaml
volumes:
  my_volume: 'UUID-here'

objects:
  MyScene:
    filename: my_volume:scenes/my_scene.blend  # volume:path notation
```

### Loading Assets
```python
# Blend files with collections
from anatools.lib.generator import get_blendfile_generator
from anatools.lib.ana_object import AnaObject

generator = get_blendfile_generator("my_package", AnaObject, "MyScene")
scene_obj = generator.exec()

# Manual blend loading (no collections)
from anatools.lib.package_utils import get_volume_path

filename = ctx.packages['my_package']['objects']['MyScene']['filename']
blend_path = get_volume_path('my_package', filename)

with bpy.data.libraries.load(blend_path, link=False) as (data_from, data_to):
    data_to.objects = [name for name in data_from.objects if name == "MyObject"]

# Non-blend files (textures, configs)
texture_path = get_volume_path('my_package', 'my_volume:textures/texture.png')
image = bpy.data.images.load(texture_path)
```

### Volume Path Resolution
- **Local dev**: `./data/volumes/UUID/path`
- **Platform**: `/data/volumes/UUID/path`
- **NEVER hardcode UUIDs** - always use `get_volume_path()`

---

## CHANNEL CONFIGURATION

### Removing Inherited Nodes
```yaml
remove_nodes:
  - Node Alias Here  # Use alias, not class name
```

**Common mistakes**:
- âŒ Category name: `- Generators.AFV`
- âŒ Class name: `- MyNodeClass`  
- âœ… Alias: `- My Node`

### Adding anatools Nodes
```yaml
add_nodes:
  - package: anatools
    name: RandomRandint
    alias: Random Integer
    category: Common
    subcategory: Values
    color: "#B3B3B3"
```

**Full list of available anatools nodes**: https://support.rendered.ai/development-guides/ana-software-architecture/the-anatools-package

**Override existing nodes** by re-adding with new properties (e.g., different `color` or `category`)

### How Nodes Get Registered
All nodes in included packages are **automatically registered** — you do NOT need to add every new node to the channel `.yml`. The channel config `add_nodes` is only for:
- Adding **anatools built-in nodes** (e.g., `RandomRandint`, `VolumeDirectory`)
- **Overriding** an existing node's display properties (color, category)
- Registering nodes from **external packages** not in the channel's package list

If you see `Schema for 'MyNodeClass' not found`, check:
1. The node's `.yml` schema file exists in the package's `nodes/` directory
2. The package is listed in the channel `.yml` under `add_packages` or equivalent
3. The `name` in the channel config matches the Python class name exactly

---

## PACKAGE SETUP

### Setup Function (lib/setup.py)
```python
def setup():
    # 1. Clean scene (first package only)
    for obj in bpy.data.objects:
        bpy.data.objects.remove(obj, do_unlink=True)
    
    # 2. Install addons (first package only)
    path_to_addon = get_volume_path('my_package', 'my_volume:')
    bpy.ops.preferences.addon_install(
        overwrite=True, target='DEFAULT',
        filepath=os.path.join(path_to_addon, "my_addon.zip")
    )
    bpy.ops.preferences.addon_enable(module='my-addon-module-name')
    
    # 3. Configure GPU (first package only)
    bpy.context.scene.render.engine = 'CYCLES'
    cyclesprefs = bpy.context.preferences.addons['cycles'].preferences
    cyclesprefs.compute_device_type = 'CUDA'
```

### Setup Execution Order (add_setup in channel .yml)
1. **First**: Package that installs addons and cleans scene
2. **Middle**: Base/shared package setup  
3. **Last**: Channel-specific setup (can override earlier settings)

**Avoid duplicating across packages**:
- âœ… Install addons once (first package)
- âœ… Configure GPU once (first package)  
- âŒ Don't install same addon in multiple packages
- âŒ Don't configure GPU in every package

---

## DATASET MANAGEMENT

### Download Platform Datasets
```bash
# List recent datasets
python3 /ana/anatools_dataset_cli.py list --limit 5

# Download dataset + log
mkdir -p /home/anadev/platform_download
python3 /ana/anatools_dataset_cli.py download <dataset_uuid> \
  --out /home/anadev/platform_download/<dataset_name> \
  --dataset --log
```

### Platform Replay Pattern
```bash
# Use exported artifacts for exact reproduction
mkdir -p /home/anadev/replay_platform
ana --graph /path/to/dataset/graph.json \
  --seed <seed_from_metadata> \
  --interp_num <interp_num_from_metadata> \
  --output /home/anadev/replay_platform
```

### Dataset Contents
- `graph.json` - Exact platform graph (use with `ana --graph`)
- `metadata/*-metadata.json` - Contains `seed` and `interp_num`
- `images/`, `masks/`, `annotations/` - Generated data
- `*-run-<runId>.log` - Platform execution log

---

## DEPLOYMENT & PLATFORM

### Pre-Deploy Validation
```bash
mkdir -p /home/anadev/predeploy_sanity
ana --preview --output /home/anadev/predeploy_sanity --logfile /home/anadev/predeploy_sanity/ana.log
```

### Platform Constraints
- **No custom environment variables** in deployed channels
- **Debug features** must be surfaced as node inputs or disabled
- **Environment-gated code** is safe (defaults apply on platform)

### API Key Management Pattern
For nodes requiring external API keys (e.g., Fal.AI, OpenAI), use **input parameters**:

```python
def exec(self) -> Dict[str, Any]:
    # Get API key from graph parameter
    api_key = self.inputs.get("API Key", [""])[0].strip()
    
    # Validate API key
    if not api_key:
        raise ValueError("API Key parameter is required")
    
    # Set up API client
    os.environ['API_CLIENT_KEY'] = api_key
    logger.info("API client configured with provided key")
```

```yaml
# In node schema
inputs:
- name: API Key
  description: Your API key for external service authentication
  default: ""
  validation:
    type: string

# For dropdown selections, use 'select:' not 'enum:'
- name: Model Type
  description: AI model to use for processing
  select:
    - fast-model
    - high-quality-model
    - experimental-model
  default: fast-model
  validation:
    type: string
```

**Benefits**:
- ✅ **Secure**: No hardcoded secrets in code
- ✅ **Platform compatible**: Works in all environments
- ✅ **User controlled**: API key set per graph/dataset
- ✅ **Flexible**: Different keys for different use cases

### Understanding interp_num
- **NOT** number of frames to render
- **IS** the run number for deterministic file naming
- Use `ctx.interp_num` for consistent output naming across nodes

## ANNOTATIONS AND ANASCENE

### How to Get Annotations Working

**CRITICAL**: Annotations require proper object validation, AnaScene creation, and compositor setup. Follow this exact pattern:

#### 1. Object Validation (Essential)
Objects from placement nodes (like Spatial Cluster) may be nested lists and missing required attributes:

```python
def _create_ana_scene(self, placed_objects, image_filename):
    if not placed_objects:
        return None
    
    # Handle nested lists (placement nodes return list-of-lists)
    if isinstance(placed_objects, list) and len(placed_objects) > 0:
        if isinstance(placed_objects[0], list):
            flat_objects = [obj for sublist in placed_objects for obj in sublist]
        else:
            flat_objects = placed_objects
    else:
        flat_objects = [placed_objects] if placed_objects else []
    
    # Validate objects have required attributes for annotation
    valid_objects = []
    for obj in flat_objects:
        # Check for required attributes (match Generic Render validation)
        has_instance = hasattr(obj, 'instance')
        has_root = hasattr(obj, 'root') and obj.root is not None
        has_ooi = hasattr(obj, 'ooi')
        
        if has_instance and has_root and has_ooi:
            obj.ooi = True  # Ensure marked as object of interest
            valid_objects.append(obj)
        else:
            # Try to fix missing attributes
            if not has_instance and has_root:
                obj.instance = len(valid_objects) + 1
            if not has_ooi:
                obj.ooi = True
            
            # Re-check after fixing
            if hasattr(obj, 'instance') and hasattr(obj, 'root') and obj.root and hasattr(obj, 'ooi'):
                valid_objects.append(obj)
    
    return valid_objects
```

#### 2. AnaScene Creation (Two Working Patterns)

**Pattern A: Direct Object Passing (Recommended)**
```python
# Let anatools AnaScene handle everything automatically
ana_scene = AnaScene(
    blender_scene=bpy.context.scene,
    objects=valid_objects,  # anatools calls add_object() for each
    sensor_name="Image"
)
```

**Pattern B: Manual add_object() (Also Works)**
```python
# Manual control over object addition
ana_scene = AnaScene(
    blender_scene=bpy.context.scene,
    objects=None,  # Start empty
    sensor_name="Image"
)

for obj in valid_objects:
    ana_scene.add_object(obj, ooi=True)
```

#### 3. Render Order (Critical)
```python
# STEP 1: Create AnaScene BEFORE rendering (sets up compositor)
ana_scene = self._create_ana_scene(placed_objects, image_filename)

# STEP 2: Set consistent frame for mask generation
bpy.context.scene.frame_current = 0

# STEP 3: Render (compositor generates masks automatically)
bpy.ops.render.render(write_still=True, scene=bpy.context.scene.name)

# STEP 4: Generate annotations AFTER render (masks now exist)
if ana_scene:
    ana_scene.write_ana_annotations()  # Creates JSON annotations
    ana_scene.write_ana_metadata()     # Creates metadata
```

#### 4. AnaScene Filename Convention

**CRITICAL**: AnaScene enforces a strict filename format. All nodes that produce output files must follow it:

```
{interp_num:010}-{frame_current}-{sensor_name}.{ext}
```

- **`interp_num`**: `ctx.interp_num` (run identifier, zero-padded to 10 digits)
- **`frame_current`**: `bpy.context.scene.frame_current` (Blender's current frame number)
- **`sensor_name`**: The sensor name passed to `AnaScene(sensor_name=...)` (e.g., `"Image"`)

This format is hardcoded in `AnaScene.write_ana_annotations()`:
```python
# From anatools/lib/scene.py
if not self.filename:
    self.filename = f'{ctx.interp_num:010}-{self.blender_scene.frame_current}-{self.sensor_name}.png'
```

**Downstream nodes** (e.g., style transfer) that duplicate annotations must use `bpy.context.scene.frame_current` for the middle segment — not a loop index or hardcoded `0`. Otherwise the annotation filenames won't match and duplication will silently fail.

```python
# ✅ CORRECT - matches AnaScene convention
frame = bpy.context.scene.frame_current
filename = f"{ctx.interp_num:010d}-{frame}-Styled.png"

# ❌ WRONG - hardcoded frame, may not match render output
filename = f"{ctx.interp_num:010d}-0-Styled.png"
```

#### 5. Expected Output Files
When working correctly, you should see:
```
/ana/output/
├── images/0000000XXX-F-Image.png          # Rendered image (F = frame_current)
├── images/0000000XXX-F-Styled.png         # Style-transferred image (same F)
├── raw_images/0000000XXX-F-Image.png      # Raw render
├── masks/0000000XXX-F-Image.png           # Object masks (auto-generated)
├── masks/0000000XXX-F-Styled.png          # Duplicated masks for styled image
├── annotations/0000000XXX-F-Image-ana.json # Object annotations
├── annotations/0000000XXX-F-Styled-ana.json # Duplicated annotations for styled image
└── metadata/0000000XXX-F-Image-metadata.json # Scene metadata
```

#### 6. Common Issues & Fixes

**Issue**: "No valid objects for annotation"
- **Cause**: Objects missing `instance`, `root`, or `ooi` attributes
- **Fix**: Add object validation and attribute fixing (see step 1)

**Issue**: "No such file: masks/XXX.png"
- **Cause**: AnaScene created after render, or frame mismatch
- **Fix**: Create AnaScene before render, use consistent frame numbering

**Issue**: Objects are lists instead of AnaObjects
- **Cause**: Placement nodes return nested lists
- **Fix**: Flatten nested lists before validation

**Issue**: Annotations empty despite valid objects
- **Cause**: `obj.rendered` or `obj.ooi` flags not set
- **Fix**: Use `add_object()` or pass objects directly to AnaScene constructor

#### 7. Why add_object() Works
```python
def add_object(self, obj, ooi=True):
    self.objects.append(obj)
    self.configure_mask(obj)      # Creates compositor mask nodes
    obj.rendered = True           # Required for annotation generation
    obj.ooi = ooi                # Marks as object of interest
    obj.active_scene = self.blender_scene
```

#### 8. Debugging Tips
```python
# Check object attributes
print(f"Object type: {type(obj)}")
print(f"Has instance: {hasattr(obj, 'instance')}")
print(f"Has root: {hasattr(obj, 'root')}")
print(f"Has ooi: {hasattr(obj, 'ooi')}")

# Check AnaScene objects
print(f"AnaScene has {len(ana_scene.objects)} objects")
for i, obj in enumerate(ana_scene.objects):
    print(f"Object {i}: rendered={obj.rendered}, ooi={obj.ooi}")
```

**Key Insight**: The real `anatools.AnaScene` automatically calls `configure_compositor()` and `add_object()` when objects are passed to the constructor, making Pattern A the simplest approach.

## TERMINAL NODES PATTERN

---

## MCP SERVERS & TOOLS

### Available MCP Servers
- **`anatools`** - Platform interaction (datasets, graphs, volumes, ML)
- **`graph`** - Local graph operations, node information, documentation

### Quick Reference Commands
```bash
# Git workflow
git status
git add <files> && git commit -m "message"
git push origin <branch>

# Ana testing
ana --output /home/anadev/test_render
ana --preview --output /home/anadev/test_render
ana --graph /path/to/graph.yaml --output /home/anadev/test_render

# Node discovery
grep -h "alias:" /ana/packages/*/nodes/*.yml | sed 's/.*alias: //' | sort
```

---

## TROUBLESHOOTING PATTERNS

### Common Issues
1. **Output not accessible** → Use `/home/anadev/<subdir>`
2. **Non-reproducible results** → Check seed/interp_num usage; check for `import random` or unsorted file listings
3. **Deployment fails** → Edit `.devcontainer/Dockerfile`, not root `Dockerfile`
4. **Node not found** → Check `alias` vs class name usage; verify node is in channel `.yml` `add_nodes`
5. **Asset not loading** → Verify volume UUID and `get_volume_path()` usage
6. **Logfile empty** → Add `--loglevel INFO` (default is `ERROR`, drops all info/debug messages)
7. **Silent data failure** → Check anatools object attributes: `FileObject.filename`, `DirectoryObject.directory` (NOT `.path`)
8. **Schema not found** → Node not registered in channel `.yml` `add_nodes`, or `name` doesn't match Python class name
9. **Annotations missing for processed images** → Filename convention mismatch; ensure all nodes use same `{interp_num}-{frame_current}-{sensor}` format
10. **Graph runs but feature doesn't work** → Check for `hasattr()` on wrong attribute names — these fail silently and skip entire code blocks

### Debug Environment Variables (Local Only)
Many channels support debug environment variables for local development:

```bash
# Common patterns (check your channel's documentation for specific variables)
export CHANNEL_DEBUG=1                  # Enable debug mode
export CHANNEL_SAVE_BLEND=1            # Save .blend snapshots
export CHANNEL_CAMERA=<camera_id>      # Force specific camera
export CHANNEL_SKIP_EXPENSIVE=1        # Skip expensive operations

# Usage with ana commands
CHANNEL_DEBUG=1 ana --preview --output /home/anadev/debug_run
```

**Important**: 
- Environment variables only work locally - platform runs cannot set custom variables
- Check your channel's specific documentation for available debug flags
- Use for local debugging and development iteration only

---

## WORKFLOW PRIORITY

1. **Setup** â†’ Dockerfile location, output directories
2. **Test** â†’ Ana commands, preview mode
3. **Develop** â†’ Node schemas, volume management
4. **Deploy** â†’ Platform validation, determinism testing
5. **Debug** â†’ Dataset download, replay patterns

Follow this order for efficient channel development.

