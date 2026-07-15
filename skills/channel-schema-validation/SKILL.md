---
name: channel-schema-validation
description: Use when authoring or debugging a Rendered.ai channel node schema YAML (nodes/<Class>.yaml) — declaring input types, oneOf branches, defaults, or numLinks validation. Trigger when a node input field shows a red border in the renderedai web editor while `ana --graph ...` or `anautils` runs the same graph cleanly; when adding a `type:` to a schema; when combining `oneOf` branches; or when setting a `default:` for a port that has validation rules. Editor-tier validation is stricter than anautils/runtime and fails silently with a red border, not an error message — this skill maps that symptom back to the underlying schema mistake.
argument-hint: "an input field is red in the graph editor but ana runs fine", "what type should I use for a float input port?", "how do I make a port accept either a typed value or a wired Vector3D node?", "default value on a new node shows red on a clean canvas"
---

# Channel node schema validation — editor-tier gotchas

Rendered.ai channel node schemas (`nodes/<Class>.yaml`) are validated in **three places with different strictness**. A schema that loads fine with `anautils` and runs cleanly with `ana --graph ...` can still render a **red border** in the renderedai web editor, blocking the user from saving the graph — with no error message pointing at the cause. This skill is the map from "red border, code runs fine" to the actual schema bug.

For general channel development, see `AGENT.md`. For `exec()`-time input/output gotchas, see the `channel-node-io` skill.

## When to use this skill

- A user reports a red-bordered input field in the graph editor, but `ana --graph ...` (or `anautils --channel ...`) runs the same graph without error
- Writing a new port's `type:` or `oneOf` validation block
- Setting a `default:` value on a port that has `validation:` rules
- Building a port that accepts either a typed-in literal or a wired node output (e.g. a `Vector3D`-shaped port)
- `anadeploy` or `anautils` succeeds but the platform UI still shows the graph as invalid

## The three validation tiers

| Tier | Where | Strictness | Symptom on failure |
|---|---|---|---|
| **Editor** (browser) | renderedai web UI | Strict JSON Schema | Red border on input fields; the user can't save the graph |
| **Channel load** (`anautils`) | `anautils --channel ...` | Lax — accepts non-standard extensions | `anautils` errors at load |
| **Runtime** (`exec()`) | `ana --graph ...` | Whatever the Python code accepts | Stack trace in the run log |

**When red borders appear, the bug is almost always in the editor-tier schema, not in your Python.** Don't go looking in `exec()` first.

## The four concrete mistakes

### 1. `type: float` is invalid

JSON Schema only defines `string`, `number`, `integer`, `boolean`, `array`, `object`, `null`. `type: float` slips past `anautils` and the editor's strict validator rejects the branch silently, leaving `oneOf` with no valid match.

```yaml
# ❌
- name: Distance (m)
  validation:
    oneOf:
      - type: float

# ✅ use `number` for both ints and floats
- name: Distance (m)
  validation:
    oneOf:
      - type: number
```

### 2. `oneOf` requires exactly one branch to match — two combinations collide

- **`type: string` + `type: array`** — the editor coerces bracket-form strings (`"[0, 0, 1]"`) into arrays for validation, so both branches match → `oneOf` fails. Drop the string branch and use a YAML array-form default (`default: [0, 0, 1]`).
- **Two scalar types** (e.g. `type: number` + `type: integer`) — every integer is also a number, so both match. Use just `type: number`.

`numLinks: <X>` branches are **safe** to combine with `type:` branches — they gate on whether the port is wired, not on the value's type.

### 3. `default:` must match a `oneOf` branch

A freshly-dragged node initialises every input from its `default:`. If the default's YAML type doesn't match any `oneOf` branch, the field renders red on a clean canvas — before the user has touched anything.

Quick check: if the schema has only `type: array`, the default must be a YAML flow-sequence (`[1, 2, 3]`), not a quoted string (`"[1, 2, 3]"`).

### 4. Canonical "literal or wired node" pattern (e.g. `Vector3D`)

Three-element float ports that accept either a typed-in literal or a wired `Vector3D` node use this exact shape:

```yaml
- name: Location (m)
  default: [0.0, 0.0, 2.0]
  description: World-space XYZ. Type [x, y, z] or wire a Vector3D node.
  validation:
    oneOf:
      - numLinks: one
      - type: array
        items:
          type: number
          minimum: -10000
          maximum: 10000
        minItems: 3
        maxItems: 3
```

Python-side, accept both list-form (from the editor) and bracket-string-form (from older graph YAMLs) via the shared helper:

```python
from toybox.lib.parsers import parse_vec3

x, y, z = parse_vec3(self.inputs["Location (m)"][0],
                     name="Location (m)", node="My Node")
```

## `numLinks` values

Valid: `zero`, `zeroOrOne`, `zeroOrMany`, `one`, `oneOrMany`. The legacy `link: true` property is no longer read by anatools or any current channel schema — don't add it; a port declared with `link: true` but no `validation.numLinks` fails at graph-link time with `"Unable to find target port"`.

## `node_data.json` — never edit by hand

`node_data.json` is auto-generated by `anautils` from the schema YAMLs. After any schema change:

```bash
anautils --channel my-channel.yml
```

Editing `node_data.json` directly is overwritten on the next regeneration and can corrupt the node registry.

## Error / symptom table

| Symptom | Cause | Fix |
|---|---|---|
| Red border in editor, `ana --graph ...` runs fine | Editor-tier validation failure | Check `type: float` usage, `oneOf` branch collisions, and whether `default:` matches a branch |
| `oneOf` never matches even with a valid-looking value | Two branches both match (string+array, or two scalar types) | Drop the redundant branch; keep exactly one type-matching branch per shape |
| New node is red on a clean, untouched canvas | `default:` doesn't match any `oneOf` branch's type | Fix the default's YAML type/form (e.g. flow-sequence array, not quoted string) |
| `"Unable to find target port"` when wiring a link | Port has `link: true` but no `validation.numLinks`, or a stale node hash | Add `validation: { numLinks: one }` (or appropriate value); for stale hashes see `AGENT_GRAPH.md` |
| Node registry seems out of date after schema edit | `node_data.json` wasn't regenerated | Run `anautils --channel my-channel.yml` |
