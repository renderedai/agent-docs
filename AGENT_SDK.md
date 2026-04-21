# Rendered.ai SDK Guide for AI Agents

Quick reference for using the `anatools` Python SDK to manage graphs, datasets, and logs on the Rendered.ai platform.

---

## AUTHENTICATION

The SDK authenticates via API key stored in the environment. No manual login required.

```python
import anatools
client = anatools.client()
```

---

## WORKSPACE & ORGANIZATION

```python
# Check current context
client.get_workspaces()
client.get_organizations()

# Switch workspace
client.set_workspace("WORKSPACE_UUID")
```

---

## UPLOADING GRAPHS

Upload a local YAML graph file to the platform.

```python
graph_id = client.upload_graph(
    graph="/path/to/graph.yaml",   # filepath or dict
    channelId="CHANNEL_UUID",      # channel the graph belongs to
    name="My Graph Name",          # display name
    description="Short desc",      # optional, ≤25 chars recommended
    staged=False,                  # False = editable, True = read-only (ready to run)
    workspaceId="WORKSPACE_UUID",  # optional, defaults to current workspace
)
# Returns: graphId (str)
```

### Key Notes
- **`staged=True`**: Graph is read-only and immediately runnable for dataset generation.
- **`staged=False`**: Graph is editable in the platform UI. Must be staged before running jobs.
- **`graph`** accepts a filepath (str) to a YAML/JSON file, or a Python dict.

---

## MANAGING GRAPHS

```python
# List graphs in workspace
graphs = client.get_graphs()                          # all graphs
graphs = client.get_graphs(staged=True)               # staged only
graphs = client.get_graphs(graphId="GRAPH_UUID")      # specific graph

# Edit graph metadata
client.edit_graph(graphId="GRAPH_UUID", name="New Name", description="New desc")

# Download graph to local file
client.download_graph(graphId="GRAPH_UUID")

# Delete graph
client.delete_graph(graphId="GRAPH_UUID")

# Set default graph for workspace
client.set_default_graph(graphId="GRAPH_UUID")
```

---

## CREATING DATASETS (RUNNING JOBS)

A dataset is created by running a staged graph. This starts a job on the platform.

```python
result = client.create_dataset(
    name="My Dataset",             # dataset name
    graphId="GRAPH_UUID",          # must be a staged graph
    description="Short desc",      # optional
    runs=150,                      # number of runs (images per camera per run)
    priority=1,                    # job priority (1 = normal)
    seed=1,                        # random seed for reproducibility
    compressDataset=True,          # compress output as zip
    tags=["training"],             # optional tags
    workspaceId="WORKSPACE_UUID",  # optional, defaults to current workspace
)
# Returns: datasetId (str)
```

### Runs Guidance
- Each run renders all cameras in the selected room.
- **Recommended range: 125–300 runs** for training data.
- More paths/cameras → use higher run counts (200+).
- Fewer paths → lower counts (125–150) to avoid redundancy.

---

## MONITORING DATASETS

```python
# List datasets
datasets = client.get_datasets()
datasets = client.get_datasets(datasetId="DATASET_UUID")

# Check job status
jobs = client.get_dataset_jobs(datasetId="DATASET_UUID")

# Get run details
runs = client.get_dataset_runs(datasetId="DATASET_UUID")

# Get dataset-level log
log = client.get_dataset_log(datasetId="DATASET_UUID")

# Cancel a running dataset
client.cancel_dataset(datasetId="DATASET_UUID")
```

---

## DOWNLOADING DATASETS

```python
client.download_dataset(datasetId="DATASET_UUID")
```

---

## DOWNLOADING LOGS

Platform logs are the authoritative source for debugging. The SDK provides per-run log access.

### Per-Run Logs

Each dataset has multiple runs, each with its own log. To download logs for specific runs:

```python
import anatools
import json

client = anatools.client()
client.set_workspace("WORKSPACE_UUID")

dataset_id = "DATASET_UUID"

# Get all runs for a dataset
runs = client.get_dataset_runs(datasetId=dataset_id)

for r in runs:
    run_id = r.get("runId")
    run_num = r.get("run", "?")
    state = r.get("state", "?")  # "complete", "failed", etc.

    # Fetch log for this specific run
    log_data = client.get_dataset_log(datasetId=dataset_id, runId=run_id)

    # Extract text from log response
    raw = log_data.get("log", "") if isinstance(log_data, dict) else str(log_data)

    # Parse JSON log entries if present
    if isinstance(raw, str) and raw.startswith("["):
        entries = json.loads(raw)
        text = "\n".join(
            entry.get("message", "") if isinstance(entry, dict) else str(entry)
            for entry in entries
        )
    else:
        text = str(raw)

    # Save to file
    with open(f"run_{run_num:04d}_{state}.log", "w") as f:
        f.write(text)
```

### Downloading Failed Run Logs

To focus on failed runs (most useful for debugging):

```python
runs = client.get_dataset_runs(datasetId=dataset_id)
failed_runs = [r for r in runs if r.get("state") == "failed"]

for r in failed_runs:
    log_data = client.get_dataset_log(datasetId=dataset_id, runId=r["runId"])
    # ... extract and save as above
```

### Downloading Logs by Date

To download logs for all datasets created since a specific date:

```python
from datetime import datetime, timezone

datasets = client.get_datasets()
cutoff = datetime(2026, 4, 20, tzinfo=timezone.utc)

recent = [
    ds for ds in datasets
    if datetime.fromisoformat(
        ds.get("createdAt", "").replace("Z", "+00:00")
    ) >= cutoff
]

for ds in recent:
    ds_id = ds["datasetId"]
    runs = client.get_dataset_runs(datasetId=ds_id)
    # ... fetch per-run logs as above
```

### Key Points
- **`get_dataset_log(datasetId=...)`** returns a dataset-level summary log.
- **`get_dataset_log(datasetId=..., runId=...)`** returns the log for a specific run — this is where `logger.info()` output lives.
- Log responses contain a `"log"` field with JSON-encoded log entries (array of `{"message": "..."}` objects).
- Platform logs capture Python `logger.info()` calls — these are the primary debugging source for node execution.

---

## COMPLETE WORKFLOW EXAMPLE

```python
import anatools
import time

client = anatools.client()
client.set_workspace("WORKSPACE_UUID")

# 1. Upload graph (editable)
graph_id = client.upload_graph(
    graph="/ana/graphs/my_graph.yaml",
    channelId="CHANNEL_UUID",
    name="My Graph",
    description="Graph description",
    staged=False,
)
print(f"Uploaded graph: {graph_id}")

# 2. Upload as staged (ready to run)
staged_id = client.upload_graph(
    graph="/ana/graphs/my_graph.yaml",
    channelId="CHANNEL_UUID",
    name="My Graph",
    description="Graph description",
    staged=True,
)
print(f"Staged graph: {staged_id}")

# 3. Create dataset (start job)
dataset_id = client.create_dataset(
    name="My Dataset",
    graphId=staged_id,
    description="Dataset description",
    runs=200,
    seed=1,
    tags=["training"],
)
print(f"Dataset job started: {dataset_id}")

# 4. Monitor
datasets = client.get_datasets(datasetId=dataset_id)
print(f"Status: {datasets}")
```

---

## LOCAL TESTING (before platform upload)

Always test graphs locally first using the `ana` CLI:

```bash
mkdir -p /home/anadev/test_output
ana --graph /ana/graphs/my_graph.yaml \
  --output /home/anadev/test_output \
  --logfile /home/anadev/test_output/ana.log \
  --loglevel INFO
```

See `AGENT.md` for full local testing patterns.

---

## QUICK REFERENCE

| Action | Method | Key Params |
|--------|--------|------------|
| Upload graph | `upload_graph()` | graph, channelId, name, staged |
| List graphs | `get_graphs()` | staged, graphId |
| Edit graph | `edit_graph()` | graphId, name, description |
| Delete graph | `delete_graph()` | graphId |
| Start job | `create_dataset()` | name, graphId, runs, seed |
| List datasets | `get_datasets()` | datasetId |
| Check jobs | `get_dataset_jobs()` | datasetId |
| Get runs | `get_dataset_runs()` | datasetId |
| Get log | `get_dataset_log()` | datasetId, runId |
| Download data | `download_dataset()` | datasetId |
| Cancel job | `cancel_dataset()` | datasetId |
