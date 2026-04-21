# Blender Patterns for Rendered.ai Channel Development

Generic Blender patterns and best practices for AI agents working on Rendered.ai channels that use Blender for rendering.

For general channel development (non-Blender-specific), see: `AGENT.md`.
For SDK/platform operations, see: `AGENT_SDK.md`.

---

## SCENE MANAGEMENT

### Loading Blend Files

```python
from anatools.lib.package_utils import get_volume_path

# Load a blend file from a volume
blend_path = get_volume_path('my_package', 'my_volume:scenes/my_scene.blend')

# Append specific objects
with bpy.data.libraries.load(blend_path, link=False) as (data_from, data_to):
    data_to.objects = [name for name in data_from.objects if name.startswith("MyPrefix")]

# Link appended objects to the active scene
for obj in data_to.objects:
    bpy.context.scene.collection.objects.link(obj)
```

### Scene Cleanup

```python
# Remove all objects (typically done in setup.py)
for obj in bpy.data.objects:
    bpy.data.objects.remove(obj, do_unlink=True)

# Remove orphan data blocks
for block in bpy.data.meshes:
    if block.users == 0:
        bpy.data.meshes.remove(block)
```

### Collection Management

```python
# Find or create a collection
def get_or_create_collection(name, parent=None):
    if name in bpy.data.collections:
        return bpy.data.collections[name]
    col = bpy.data.collections.new(name)
    (parent or bpy.context.scene.collection).children.link(col)
    return col

# Hide a collection from render
collection.hide_render = True

# Move an object to a specific collection
target_collection.objects.link(obj)
for col in obj.users_collection:
    if col != target_collection:
        col.objects.unlink(obj)
```

---

## OBJECT PLACEMENT & TRANSFORMS

### Setting Object Transforms

```python
from mathutils import Vector, Euler
import math

# Set position
obj.location = Vector((x, y, z))

# Set rotation (Euler angles in radians)
obj.rotation_euler = Euler((0.0, 0.0, math.radians(90)), 'XYZ')

# Force scene graph update after transform changes
bpy.context.view_layer.update()
```

### Avoid Accidental Transform Mutation

A common failure mode is resetting transforms on objects that should keep their original placement (e.g. furniture, props, infrastructure).

**Symptoms:**
- Objects visible in the blend file disappear in channel renders
- Props appear in different locations between runs
- Raw IndexOB debug shows near-zero pixels for objects that should be visible

**Fix pattern:**
- Filter placement/animation loops to operate only on the objects you intend to move
- Never reset `.location` / `.rotation_euler` on static scene objects
- Use explicit object type checks before applying transforms

### Reading Geometry Nodes Modifier Properties

Geometry Nodes modifiers expose socket values as modifier properties. Socket identifiers (e.g. `Socket_8`) are **not stable** across blend file versions. Use dynamic name lookup:

```python
def get_geonode_inputs(obj, modifier_name_contains='GeometryNodes'):
    """Read inputs from a Geometry Nodes modifier by input name."""
    results = {}
    for mod in obj.modifiers:
        if modifier_name_contains not in mod.name:
            continue
        # Collect raw modifier properties
        props = {}
        try:
            for key in mod.keys():
                props[key] = mod[key]
        except Exception:
            pass
        # Map by human-readable input name
        ng = getattr(mod, 'node_group', None)
        if ng and hasattr(ng, 'interface') and hasattr(ng.interface, 'items_tree'):
            for item in ng.interface.items_tree:
                if not hasattr(item, 'identifier'):
                    continue
                val = props.get(item.identifier)
                if val is not None:
                    results[item.name] = val
        # Fallback: return raw socket properties
        if not results:
            results = props
        break
    return results
```

This is important when reading action types, collection assignments, or any parameter from Geometry Nodes setups.

---

## CAMERAS

### Iterating Over Cameras

```python
cameras = [obj for obj in bpy.data.objects if obj.type == 'CAMERA']

for cam in cameras:
    bpy.context.scene.camera = cam
    bpy.context.scene.render.filepath = f"/tmp/render_{cam.name}.png"
    bpy.ops.render.render(write_still=True)
```

### Checking If a Point Is In Camera Frame

```python
from bpy_extras.object_utils import world_to_camera_view

def is_in_frame(point, camera, scene=None):
    """Check if a 3D point is within the camera's view frustum."""
    scene = scene or bpy.context.scene
    co = world_to_camera_view(scene, camera, Vector(point))
    return 0.0 <= co.x <= 1.0 and 0.0 <= co.y <= 1.0 and co.z > 0.0
```

### Camera Coordinate Conventions

- Blender world: `+Z` is up, `+X` is right, `+Y` is forward (in Top view)
- Camera view: `co.x` = horizontal (0=left, 1=right), `co.y` = vertical (0=bottom, 1=top), `co.z` = depth (>0 = in front)

---

## RENDERING

### Cycles Setup (GPU)

```python
bpy.context.scene.render.engine = 'CYCLES'
cyclesprefs = bpy.context.preferences.addons['cycles'].preferences
cyclesprefs.compute_device_type = 'CUDA'
cyclesprefs.get_devices()

# Enable all GPU devices
for device in cyclesprefs.devices:
    device.use = True

bpy.context.scene.cycles.device = 'GPU'
```

### Render Settings

```python
scene = bpy.context.scene

# Resolution
scene.render.resolution_x = 1920
scene.render.resolution_y = 1080
scene.render.resolution_percentage = 100

# Output format
scene.render.image_settings.file_format = 'PNG'
scene.render.image_settings.color_mode = 'RGBA'

# Samples (lower = faster preview)
scene.cycles.samples = 128
scene.cycles.preview_samples = 32
```

### Background Rendering

```bash
blender --background /path/to/scene.blend --python /path/to/script.py
```

Note: Background renders may print noisy warnings (e.g. geometry-nodes dependency cycles). These are typically harmless.

---

## MASKS & INSTANCE SEGMENTATION

### pass_index for Object Identification

Each object of interest needs a unique `pass_index` for instance segmentation masks:

```python
obj.pass_index = unique_id  # Integer, must be unique per object of interest
```

### IndexOB Pass Setup

```python
# Enable IndexOB pass in render layers
bpy.context.scene.view_layers[0].use_pass_object_index = True
```

### Debugging Empty Masks

If `masks/*.png` is all-zero:

1. **Check raw IndexOB** — is the underlying render pass empty?
   - If yes: scene/camera/visibility issue (object not in frame, hidden collection, etc.)
   - If no: compositor or mask pipeline issue

2. **Check `pass_index`** — is it set and unique?
3. **Check collection visibility** — is `hide_render = False`?
4. **Check object visibility** — is `hide_render = False` on the object itself?

### Minimum Feature Size

Mask pipelines often drop polygons smaller than a threshold. If small objects (distant from camera) have nonzero mask IDs but empty annotations, check the minimum bounding box size filter.

---

## ANIMATIONS & ACTIONS

### Applying Actions to Objects

```python
# Get an action by name
action = bpy.data.actions.get('walk_cycle_01')
if action is None:
    logger.warning("Action 'walk_cycle_01' not found in blend file")

# Apply action via NLA track
armature = obj  # Must be an armature object
if armature.animation_data is None:
    armature.animation_data_create()

track = armature.animation_data.nla_tracks.new()
strip = track.strips.new(action.name, int(action.frame_range[0]), action)
strip.action = action
```

### Listing Available Actions

```python
for action in sorted(bpy.data.actions, key=lambda a: a.name):
    print(f"  {action.name}: frames {action.frame_range[0]:.0f}-{action.frame_range[1]:.0f}")
```

Always verify an action exists in `bpy.data.actions` before using it — action names vary between blend files.

### Follow Path Constraints

```python
# Add follow-path constraint for character walking
bpy.context.view_layer.objects.active = obj
obj.select_set(True)
bpy.ops.object.constraint_add(type='FOLLOW_PATH')
constraint = obj.constraints['Follow Path']
constraint.target = path_object  # A curve object
constraint.use_curve_follow = True
constraint.forward_axis = 'TRACK_NEGATIVE_Y'
```

---

## BLEND FILE INSPECTION

### Headless Blend Inspection

```bash
blender --background /path/to/scene.blend --python-expr "
import bpy
print('Objects:', len(bpy.data.objects))
print('Actions:', [a.name for a in bpy.data.actions])
print('Collections:', [c.name for c in bpy.data.collections])
for obj in bpy.data.objects:
    if obj.type == 'CAMERA':
        print(f'Camera: {obj.name} at {tuple(obj.location)}')
"
```

### Saving Debug Blend Snapshots

Save a `.blend` after scene setup for post-run inspection:

```python
import os
debug_path = os.path.join(ctx.output, 'test', f'{ctx.interp_num:010d}-debug.blend')
os.makedirs(os.path.dirname(debug_path), exist_ok=True)
bpy.ops.wm.save_as_mainfile(filepath=debug_path)
```

Gate behind an environment variable to avoid large files in production:

```python
if os.environ.get('CHANNEL_SAVE_DEBUG_BLEND'):
    bpy.ops.wm.save_as_mainfile(filepath=debug_path)
```

---

## MATERIALS & TEXTURES

### Loading Textures from Volumes

```python
from anatools.lib.package_utils import get_volume_path

texture_path = get_volume_path('my_package', 'my_volume:textures/diffuse.png')
image = bpy.data.images.load(texture_path)

# Assign to a material's image texture node
mat = bpy.data.materials['MyMaterial']
nodes = mat.node_tree.nodes
tex_node = nodes.get('Image Texture')
if tex_node:
    tex_node.image = image
```

### Randomizing Materials

```python
import random

# Swap between material variants
variants = [m for m in bpy.data.materials if m.name.startswith('Wood_')]
if variants:
    chosen = ctx.random.choice(variants)
    obj.active_material = chosen
```

---

## LIGHTING

### Time-of-Day / Lighting Control

A common pattern is exposing lighting as a node input and switching between presets:

```python
# Example: toggle sun lamp intensity
sun = bpy.data.objects.get('Sun')
if sun and sun.type == 'LIGHT':
    if time_of_day == 'Night':
        sun.data.energy = 0.1
    else:
        sun.data.energy = 5.0

# Example: switch HDRI environment
world = bpy.context.scene.world
env_node = world.node_tree.nodes.get('Environment Texture')
if env_node:
    hdri_path = get_volume_path('my_package', f'my_volume:hdri/{time_of_day}.exr')
    env_node.image = bpy.data.images.load(hdri_path)
```

---

## COMMON GOTCHAS

- **Dependency cycles**: Geometry Nodes can trigger noisy "dependency cycle detected" warnings in background mode. These are usually harmless.
- **`bpy.context` in background mode**: Some operators require an active object or specific context. Use `bpy.context.view_layer.objects.active = obj` before operator calls.
- **Frame numbering**: Always use `bpy.context.scene.frame_current` for consistent naming with AnaScene annotation conventions.
- **Action names**: Never hardcode action names without checking `bpy.data.actions.get()` first — they vary between blend file versions.
- **Euler rotation order**: Default is `'XYZ'`. Mixing rotation orders causes subtle orientation bugs.
- **`view_layer.update()`**: Call after changing transforms if subsequent code reads back positions (e.g. checking camera visibility).

