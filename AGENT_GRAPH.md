# Rendered.ai Graph YAML Guide for AI Agents

Reference for authoring, modifying, and debugging Rendered.ai graph YAML files (`ana/graphs/*.yaml`).

For SDK operations (uploading, running), see `AGENT_SDK.md`.
For DIRSIG node internals, see `AGENT_DIRSIG.md`.

---

## GRAPH YAML STRUCTURE

A graph file is a YAML mapping with two top-level keys:

```yaml
graph:          # mapping of nodeId → node definition
  NodeClass_1:
    ...
  NodeClass_2:
    ...
```

There is no version field or metadata block — the file is purely the `graph:` mapping.

---

## NODE ANATOMY

Every node has this shape:

```yaml
  NodeId_N:
    alias: NodeClass        # human-friendly display name (often == nodeClass)
    color: '#RRGGBB'        # node color in the graph editor
    hash: <sha1>            # version hash of the node CLASS (see below)
    links:                  # input ports receiving values from other nodes
      PortName:
        - outputPort: OutputPortName
          sourceNode: SourceNodeId_M
    location:               # visual position in graph editor (cosmetic only)
      x: 1234
      'y': 567              # MUST be quoted — see YAML pitfall below
    name: NodeId_N          # must match the mapping key
    nodeClass: NodeClass    # registered class name in the channel package
    ports:                  # embedded copy of the node class port schema
      inputs:
        - name: PortName
          default: someValue
          description: ...
          link: true        # present only if this port accepts links
          validation:
            numLinks: zeroOrMany
      outputs:
        - name: OutputPortName
    tooltip: >-
      Short human-readable description of the node.
    values:                 # user-set values for non-linked input ports
      PortName: someValue
```

### Required fields
`alias`, `color`, `hash`, `location`, `name`, `nodeClass`, `ports`, `tooltip`

### Optional fields
`links` (omit if no inputs are linked), `values` (omit if all ports use defaults)

---

## NODE HASH VERSIONING — CRITICAL

The `hash` field identifies which **version** of a node class the graph was built against. The platform uses this to resolve the node's port schema at validation time.

### What the hash is
- A SHA-1 of the node class definition (schema, port list, tooltip). NOT a hash of instance data.
- All instances of the same class version share the same hash.
- The hash changes when the node is updated and republished to the platform.

### Stale hashes cause "Unable to find target port" errors
When a graph is uploaded with an old hash the platform no longer recognizes:
- The platform falls back to the **current** node version.
- Any ports that existed in the old version but were removed in the new version will fail link validation.
- Error: `Unable to find target port <PortName> for link to node <NodeId>`

### How to find the current hash for a node class
Look at the **most recently created** graph that uses that node:

```bash
grep -A2 "nodeClass: Countryside" ana/graphs/PanelsContryside_moreGrass.yaml
# hash: ea8f891f39fd23a6ee0153075b623ab73ba12e4e
```

Or download a graph from the platform UI and inspect the hash field.

### When building a new graph from an old one
1. Find the newest graph using each node class.
2. Copy the `hash` AND the entire `ports:` block from that graph — they must match.
3. Check `values:` keys against the new port list — remove any keys for ports that no longer exist, add any new required values.

### Example: Countryside node version change
| Hash | Ports that changed |
|------|--------------------|
| `b362a484828d5eb5df58aacb80428005901a14d2` | Had `Tree Region Radius (m)` and `Tree Region Center XY (m)` as linkable input ports |
| `ea8f891f39fd23a6ee0153075b623ab73ba12e4e` | Those ports removed; they are now `values:` entries only |

Graphs using the old hash fail on upload with the "Unable to find target port Tree Region Center XY (m)" error.

---

## PORT DEFINITIONS IN `ports.inputs`

Each entry in `ports.inputs` is a dict with:

| Field | Required | Purpose |
|-------|----------|---------|
| `name` | yes | Port identifier — must match the key used in `links:` and `values:` |
| `default` | yes | Value used when port is unlinked and absent from `values:` |
| `description` | yes | Human-readable description |
| `link: true` | only if linkable | Marks the port as accepting links from other nodes. **Absent = port cannot be linked.** |
| `select` | no | Restricts to an enum of allowed values |
| `validation.numLinks` | no | Link cardinality: `zeroOrMany`, `exactlyOne`, etc. |

**If `link: true` is absent on a port, the platform will reject any `links:` entry pointing to that port.**

### Ports that accept multiple links
Biome region ports use:
```yaml
        - name: Central Region
          link: true
          validation:
            numLinks: zeroOrMany
```

### Ports that accept exactly one link
```yaml
        - name: Objects
          link: true
          default: null
```
(No `validation` key = default is one link max.)

---

## LINKS

The `links:` dict maps **this node's input port names** to source nodes:

```yaml
  DestNode_5:
    links:
      InputPortName:
        - outputPort: OutputPortName   # output port on the source node
          sourceNode: SourceNode_3     # source node id
      MultiLinkPort:
        - outputPort: Biome
          sourceNode: Desert Biome_11
        - outputPort: Biome
          sourceNode: Grass Biome_15
```

- The key (e.g. `InputPortName`) must exactly match a port `name` in `ports.inputs` **and** that port must have `link: true`.
- The value is always a list, even for single links.
- `outputPort` must match a port `name` in the source node's `ports.outputs`.

### Links vs Values
- Use `links:` when the value is computed by another node at runtime.
- Use `values:` for user-controlled static settings.
- A port can appear in `values:` even when not linked (sets the default override).
- A linked port's value **overrides** any matching `values:` entry at runtime.

---

## YAML 1.1 PITFALLS — `y:` IS A BOOLEAN

Python's PyYAML (YAML 1.1) treats bare `y` as boolean `True` in both key and value positions:

```yaml
# WRONG — y key parsed as boolean True, not string 'y'
links:
  y:
    - outputPort: Sum
      sourceNode: Addition_4

# CORRECT — always quote single-letter port names that are YAML booleans
links:
  'y':
    - outputPort: Sum
      sourceNode: Addition_4
```

The same applies to `name`, `values`, and `location` fields:

```yaml
# WRONG
- name: y          # port name becomes boolean True
values:
  y: '-0.2618'     # key becomes boolean True
location:
  x: 100
  y: 200           # key becomes boolean True (harmless for location but inconsistent)

# CORRECT
- name: 'y'
values:
  'y': '-0.2618'
location:
  x: 100
  'y': 200
```

**Rule: Any port, value, or link key named `y`, `n`, `yes`, `no`, `true`, `false`, `on`, `off` (case-insensitive) must be quoted.**

---

## VALUES

The `values:` dict sets static overrides for non-linked ports:

```yaml
    values:
      Cable Type: <random>              # platform picks randomly
      Collect Material Fractions: Enabled
      Tree Region Radius (m): '1300'    # numeric values are strings
      Tree Region Center XY (m): 0, 0  # comma-separated for 2D coords
      Use Trees: 'True'                 # boolean flags are strings
```

- Keys must match port `name` values exactly.
- Numeric values are usually strings (`'1300'`, not `1300`) — match the node's `default` format.
- `<random>` tells the platform to pick randomly from the `select` list.
- `values:` keys for ports that no longer exist in the current node version will cause validation errors — remove them when updating hashes.

---

## BUILDING NEW GRAPHS FROM EXISTING ONES

### Safe workflow
1. Copy the most similar existing graph.
2. For each node class, verify its hash matches the newest version in use (grep other graphs for the same `nodeClass:`).
3. Update `ports:` blocks to match the current node version.
4. Adjust `links:` and `values:` to match the new port list.
5. Validate locally: `ana --graph ana/graphs/my_graph.yaml --output /tmp/test_out`

### Adding a new node instance
Copy a node of the same class from any graph. Keep the `hash`, `ports`, `tooltip`, `alias`, `color`, and `nodeClass` fields identical. Only `name`, `location`, `links`, and `values` are instance-specific.

### Adding a link between nodes
1. Confirm the destination port has `link: true` in its `ports.inputs` entry.
2. Add an entry to `links:` on the destination node:
   ```yaml
     DestNode_5:
       links:
         PortName:
           - outputPort: OutputPortName
             sourceNode: SourceNode_3
   ```
3. Do NOT add anything to the source node.

---

## COMMON ERRORS

### `Unable to find target port <X> for link to node <NodeId>`
**Cause A — stale hash**: The node's hash is from an old version that no longer exists on the platform. The platform resolves to the current version, which doesn't have port `<X>`.
- Fix: Update `hash` and `ports:` to the current version. Remove the broken link; if the port moved to `values:`, set it there instead.

**Cause B — missing `link: true`**: Port `<X>` exists in `ports.inputs` but lacks `link: true`.
- Fix: Add `link: true` to the port definition in the node class YAML (`nodes/<class>.yaml`), then republish the node to get a new hash, and update the graph.

**Cause C — YAML boolean key**: The port name contains a YAML 1.1 boolean keyword (e.g. `y`) and was written unquoted. The link key is parsed as `True` (boolean), not the string `'y'`.
- Fix: Quote the key: `'y':` instead of `y:`.

### `Node class <X> not found`
The `nodeClass` field doesn't match any registered class in the channel package.
- Check spelling, including case, against `nodes/` directory.

### Graph uploads but produces wrong outputs
Check `values:` for stale keys from an old node version — they are silently ignored, but the port may now default to something unexpected.

---

## QUICK REFERENCE: GRAPH FILE CHECKLIST

When creating or editing a graph YAML:

- [ ] `hash` matches current platform version for every node class
- [ ] `ports:` block copied verbatim from a current-version graph
- [ ] Every `links:` key corresponds to a port with `link: true`
- [ ] `values:` has no keys for ports that no longer exist
- [ ] Single-letter port names (`'y'`, `'x'`) are quoted everywhere they appear as YAML keys
- [ ] `location.'y'` is quoted: `'y': 200`
- [ ] `name:` field matches the YAML mapping key exactly
