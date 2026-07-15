---
name: channel-node-io
description: Use when writing or debugging a Rendered.ai channel node's exec() method — reading self.inputs, returning outputs, handling optional/unlinked ports, or passing anatools objects (FileObject, DirectoryObject) between nodes. Trigger when the user reports an output that is silently empty, wrong, or None with no exception; a TypeError from float()/int() on a node input; a crash or skip on an optional port that was unlinked in the editor; or an AttributeError/None on .path for a FileObject or DirectoryObject. These are silent-failure bugs — the platform raises no error, so catch them by construction rather than by traceback.
argument-hint: "why is my node's output always empty downstream?", "TypeError: float() argument must be a string or a number, not 'list'", "the node crashes only when I unlink an optional input", "FileObject has no attribute 'path'"
---

# Channel node I/O — silent-failure patterns

Four failure modes in Rendered.ai channel node `exec()` methods that produce **no exception** — just wrong, empty, or `None` data downstream. There is no traceback to work backward from, so the fix is to apply these patterns proactively while writing the code, not after debugging a mystery.

For general channel development, see `AGENT.md`. For the node schema YAML that pairs with `exec()`, see the `channel-schema-validation` skill.

## When to use this skill

- Writing a new node's `exec()` method
- An output port is empty/wrong on the downstream node, but the upstream node's code "looks right"
- `TypeError: float() argument must be a string or a number, not 'list'` (or similar) inside `exec()`
- A node crashes or silently no-ops specifically when an optional input is left unlinked in the graph editor
- `AttributeError: 'FileObject' object has no attribute 'path'` (or `DirectoryObject`)
- Renaming an output port in the node schema YAML

## The four patterns

### 1. `self.inputs` is always a list

```python
# ❌ Silent bug — value is the list itself, not the element
number = float(self.inputs["Distance"])          # TypeError, or wrong value if it happens to coerce

# ✅ Always index with [0]
number = float(self.inputs["Distance"][0])
count = int(self.inputs["Number of Objects"][0])

# ✅ Safe access with a default
setting = self.inputs.get("Optional Port", ["default value"])[0]
```

`self.inputs` merges `values:` from the graph YAML with linked node outputs — linked values override `values:`.

### 2. Output dict keys must exactly match the schema `outputs[].name`

```python
# Schema declares: outputs: [{name: "Output Port"}]
return {"Output Port": result}                    # ✅
# self.outputs["Output Port"] = result             # ✅ equivalent

return {"output_port": result}                     # ❌ silent — platform looks up
                                                     #    "Output Port", finds nothing,
                                                     #    downstream gets an empty value
```

**When you rename an output port in the node schema YAML, update the matching key in `exec()` in the same edit.** There is no error connecting the two — a mismatch is a schema-key lookup miss, not a Python exception.

### 3. Optional unlinked input arrives as `""`, not `None`

When a user removes a link from an optional input in the platform UI, the platform sets that input to `""` (empty string) — it is **not** `None` and is **not** omitted from `self.inputs`. Passing `""` straight into downstream code (a path, a numeric cast, an object list) silently skips or crashes it.

```python
# ✅ Safe pattern for optional object lists
objects = [o for o in self.inputs.get("Objects", []) if o and o != ""]

# ✅ Safe pattern for optional scalar inputs
rotation = float((self.inputs.get("Rotation (deg)") or [0.0])[0])
```

### 4. Use `.filename` / `.directory`, never `.path`

| Type | Source node | Correct attribute | Wrong attribute |
|------|-------------|--------------------|------------------|
| `FileObject` | `VolumeFile` | `.filename` | `.path` ❌ |
| `DirectoryObject` | `VolumeDirectory` | `.directory` | `.path` ❌ |

```python
# ✅ Defensive pattern when the input's concrete type isn't certain
if hasattr(obj, 'filename'):
    path = obj.filename       # FileObject
elif hasattr(obj, 'directory'):
    path = obj.directory      # DirectoryObject
```

## Quick checklist before finishing a node's `exec()`

1. Every `self.inputs["..."]` access is indexed with `[0]` (or goes through `.get(...)[0]` / `(... or [default])[0]`).
2. Every returned/assigned output key is copy-pasted from the schema YAML's `outputs[].name`, not re-typed.
3. Every optional port is filtered for `""` before use, not just `None`.
4. Every `FileObject`/`DirectoryObject` access uses `.filename`/`.directory`, never `.path`.

## Error / symptom table

| Symptom | Cause | Fix |
|---|---|---|
| Downstream node receives empty/`None` for a value that upstream code clearly sets | Output dict key doesn't match schema `outputs[].name` exactly | Diff the returned key against the schema YAML; fix the typo/rename |
| `TypeError: float() argument must be a string or a number, not 'list'` | Missing `[0]` index on `self.inputs[...]` | Index the input: `self.inputs["Port"][0]` |
| Node crashes or silently no-ops only when an optional input is unlinked | Unlinked optional input is `""`, not `None`/omitted | Filter: `[o for o in inputs if o and o != ""]`, or `(x or [default])[0]` |
| `AttributeError: 'FileObject' object has no attribute 'path'` | Used `.path` instead of the type-specific attribute | Use `.filename` (`FileObject`) or `.directory` (`DirectoryObject`) |
