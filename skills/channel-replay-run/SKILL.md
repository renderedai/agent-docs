---
name: channel-replay-run
description: Use when reproducing a Rendered.ai platform-rendered dataset run locally with `ana`, or when downloading per-run platform logs for a dataset. Trigger on "replay this dataset", "reproduce this run locally", "pull/download the logs for dataset <name>", "debug this dataset locally", or when local behaviour differs from the platform and the graph.json embedded in the dataset needs to be compared against the local graph YAML. Covers resolving a dataset name to its ID via the anatools SDK, extracting seed/interp_num from run metadata, staging graph.json, and the container path model `ana` requires for --output/--logfile.
argument-hint: "replay dataset <name> locally", "pull the platform logs for <dataset>", "why does this run behave differently locally vs on the platform?", "extract the seed and interp_num from this dataset's metadata"
---

# Replay a platform dataset run locally

Recipe for reproducing a platform-rendered dataset run on a local `ana` dev container and pulling its per-run platform logs. The platform embeds the exact `graph.json`, `seed`, and `interp_num` used for a run in the dataset's metadata — replaying means recovering those three values and feeding them back into `ana`, not guessing at a local graph YAML.

For general channel development and the `ana` container path model, see `AGENT.md`. For the `anatools` SDK itself, see `AGENT_SDK.md`.

## When to use this skill

- "Replay dataset `<name>` / `<UUID>` locally"
- "Pull / download the logs for dataset `<name>`"
- "Debug this dataset locally" / "re-run frame X of dataset Y"
- Local (`ana --graph my_graph.yaml`) and platform behaviour diverge for the same graph — the platform reads the `graph.json` embedded in the uploaded dataset, **not** your local YAML, so the two can silently drift
- A user references a specific dataset and a specific frame/camera they want inspected

## Prerequisites

- `RENDEREDAI_API_KEY` set in the environment (required by the `anatools` SDK).
- The target workspace/dataset is mounted and readable (typically under a `datasets/<datasetId>/` path on the FUSE-mounted workspace volume). If it isn't mounted, stop and tell the user — do not run `anamount --unmount` from an agent-driven shell (see the stale-mount danger callout in `AGENT.md`).
- `ana` is available and the channel repo it targets is checked out locally.

## Container path model (must be honoured)

| Host | Container | Notes |
|---|---|---|
| `/workspace/ana` (or channel repo root) | `/ana` | Live bind mount |
| `/workspace/output` (or `$WORKSPACE_OUTPUT`) | `/output` | Only path that survives container exit |
| `/workspace/volumes/<uuid>` | `/data/volumes/<uuid>` | Read-only asset volumes |

`--output` and `--logfile` must be passed as **container-internal** paths (`/output/...`) — a host path like `/workspace/output/...` does not exist inside the container. Create the destination directory on the host before running; the container opens the logfile for write and does not `mkdir -p` it.

## Steps

### 1. Resolve dataset name → datasetId

```python
import anatools
c = anatools.client(environment='prod', verbose='quiet')
c.set_workspace(WORKSPACE_UUID)
datasets = c.get_datasets() or []
matches = [d for d in datasets if d.get('name') == DATASET_NAME]
matches.sort(key=lambda d: d.get('createdAt', ''), reverse=True)  # newest first
dataset_id = matches[0]['datasetId']
```

If more than one dataset shares the name, list them with `createdAt` and confirm which one before proceeding — do not silently pick one.

### 2. Extract `seed` + `interp_num` from run metadata

Per-frame metadata files are named `<interp_num:010d>-<frame>-<sensor>-metadata.json` under the dataset's `metadata/` directory. `seed` and `interp_num` are identical across every frame of the same run, so any one file is sufficient:

```python
import json, glob
meta_files = sorted(glob.glob(f"{dataset_dir}/metadata/*-metadata.json"))
with open(meta_files[0]) as f:
    m = json.load(f)
seed = m["seed"]
interp_num = m["interp_num"]
```

### 3. Pull per-run platform logs

```python
import anatools, json, os

c = anatools.client(environment='prod', verbose='quiet')
c.set_workspace(WORKSPACE_UUID)
runs = c.get_dataset_runs(datasetId=dataset_id)
out_dir = f"/workspace/ana/platform_logs/{dataset_id}"
os.makedirs(out_dir, exist_ok=True)

for r in runs:
    run_id, run_num, state = r["runId"], r.get("run", "?"), r.get("state", "?")
    log = c.get_dataset_log(datasetId=dataset_id, runId=run_id)
    raw = log.get("log", "") if isinstance(log, dict) else str(log)
    if isinstance(raw, str) and raw.startswith("["):
        try:
            entries = json.loads(raw)
            text = "\n".join(e.get("message", "") if isinstance(e, dict) else str(e) for e in entries)
        except Exception:
            text = raw
    else:
        text = str(raw)
    with open(f"{out_dir}/run_{run_num:04d}_{state}.log", "w") as f:
        f.write(text)
```

These logs carry the channel's `logger.info(...)` output — `print()` calls are **not** captured by the platform and will not appear here.

### 4. Stage `graph.json` for `ana`

The exact graph used for the run is embedded in the dataset, not your local graph YAML:

```bash
mkdir -p /workspace/output/replay_<slug>
cp <dataset_dir>/graph.json /workspace/output/replay_<slug>/graph.json
```

### 5. Run the replay

```bash
ana --graph /output/replay_<slug>/graph.json \
    --seed <seed> --interp_num <interp_num> \
    --output /output/replay_<slug> \
    --logfile /output/replay_<slug>/ana.log \
    --loglevel INFO
```

Confirm with the user before kicking this off — a full run can take minutes, and channel-specific local-only debug env vars (see `AGENT.md` → "Local debug environment variables") are often worth setting first.

### 6. After the replay

- Renders land under `/workspace/output/replay_<slug>/images/`.
- The local log at `/workspace/output/replay_<slug>/ana.log` mirrors the container-side `logger.info` output — compare it against the pulled platform log from step 3 to spot divergence.

## Error / symptom table

| Symptom | Cause | Fix |
|---|---|---|
| Local run behaves differently than the platform for what looks like the same graph | Platform ran the `graph.json` embedded in the dataset, which has drifted from your local graph YAML | Always diff the **downloaded** `graph.json` against your local YAML before assuming a code regression |
| `ana --logfile /workspace/output/...` fails or writes nowhere useful | Passed a host path instead of the container-internal `/output/...` path | Use `--output /output/...` and `--logfile /output/.../ana.log` |
| Platform log pull returns nothing useful for a run | Used `print()` for debugging instead of `logger.info(...)` | Only `logger.*` output reaches platform per-run logs |
| Multiple datasets share the same name | Ambiguous name→ID resolution | List candidates with `createdAt` and confirm with the user before picking one |
