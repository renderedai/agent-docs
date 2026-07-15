---
name: agent-studio
description: Use when developing, testing, or deploying services inside a Rendered.ai Agent Studio workspace. Trigger on questions about workspace storage layout (local NVMe vs FUSE-mounted `/workspace/volumes/*` and `/home/ubuntu/.renderedai/workspaces/.../datasets/*`), FUSE mount timeouts during long-running jobs, on-demand (`test_service`, `run_service`) vs persistent (`run_persistent_service`) service execution, the `renderedai` CLI and `anatools` SDK, the MCP `raiservices-local` toolchain, layered platform rules (User/Org/Workspace/Service), or any `FileNotFoundError` / mount-drop symptom during training or evaluation runs.
argument-hint: "where should I write training output?", "why did my training crash with FileNotFoundError after 4 hours?", "is this service on-demand or persistent?", "what's the difference between /workspace/volumes/ and /workspace/datasets/?"
---

# Agent Studio

Operational reference for AI agents and engineers developing **services** on the Rendered.ai **Agent Studio** platform — the workspace runtime where you build, test, and deploy long-running and on-demand services (ML pipelines, data processors, API workers) that interact with platform resources (datasets, volumes, models, channels).

This skill applies to **first- and third-party service developers** working inside an Agent Studio workspace. It does **not** cover synthetic-data channel authoring (Blender/DIRSIG renderers) — those live in their own channel-development guides.

## When to use this skill

Activate this skill whenever the user is:

- Designing the **output path** of a long-running job (training, evaluation, bulk preprocessing) inside an Agent Studio workspace
- Debugging a **`FileNotFoundError`, `ENOENT`, or "mount dropped"** symptom after a job has been running for hours
- Deciding between **on-demand** (`test_service`, `run_service`) and **persistent** (`run_persistent_service`) service execution
- Reading or writing to `/workspace/volumes/...`, `/workspace/datasets/...`, `/home/ubuntu/.renderedai/workspaces/.../datasets/...`, or `/workspace/services/...` and unsure which tier each represents
- Using the `renderedai` CLI, `anatools` Python SDK, or the MCP `raiservices-local` server
- Querying or editing layered platform rules (User / Organization / Workspace / Service / Platform)
- Setting up a new service repository under `/workspace/<repo>/services/<service-name>/`
- Reasoning about `test_service` / `run_service` / `run_persistent_service` Dockerfile, mount, and schema conventions

## Workflow for the agent

1. **If the question involves a path under `/workspace/`** — open `AGENT_STUDIO.md` in this skill and consult the **Workspace storage layout** table to determine whether the path is local NVMe or FUSE-backed before recommending where to write.
2. **If the question involves a long-running job** — apply the **FUSE timeout rule**: never write per-epoch state to `/workspace/volumes/...`; use `/workspace/datasets/<run_name>/` (local), then copy the final artifact to the FUSE volume at the end.
3. **If the question involves a service execution model** — consult the **Service execution model** section to choose on-demand vs persistent and to pick the correct MCP tool.
4. **If the question involves the `renderedai` CLI** — do **not** redocument it inline. Point the user to `renderedai --help` and the upstream [`renderedai-cli` SKILL.md](https://github.com/renderedai/anatools/blob/8f94b288d672ebc676a2e25b959058e61389eb72/skills/renderedai-cli/SKILL.md).
5. **If the symptom is a mid-run crash on a FUSE path** — ask the user to remount; the files are not lost. Do not retry the failed write into the same path until the mount is back.

## What this skill provides

| File | Contents |
|------|----------|
| `AGENT_STUDIO.md` | Full platform reference: toolchain table, workspace storage layout, FUSE timeout rule with concrete reference failure, on-demand vs persistent execution, workspace directory tour, rules/memory system, common gotchas, when-to-update guidance. |

The reference doc is **prescriptive** — it includes the exact `df -h` invocation to identify mount type, the exact `output_dir` pattern for training jobs, and the order in which to investigate when something crashes.

## Examples

### Example 1 — User asks where to write training output

> **User:** "I'm about to start a 6-hour UNet training run. Where should `output_dir` point?"
>
> **Agent action:** Read `AGENT_STUDIO.md` → *Workspace storage layout* + *FUSE timeout rule*. Answer: `/workspace/datasets/<run_name>/` (local NVMe). Recommend copying `model_best.pth` to `/workspace/volumes/<vol>/models/` at the end of the run.

### Example 2 — User reports a `FileNotFoundError` mid-training

> **User:** "Training crashed at epoch 12 with `FileNotFoundError` writing `training_status.json` to `/workspace/volumes/foo/v3/`. The directory existed when the run started."
>
> **Agent action:** Recognize the FUSE-drop signature (see reference failure on 2026-06-06 in `AGENT_STUDIO.md`). Answer: the FUSE mount dropped silently. Ask user to remount. Recommend resuming from the last checkpoint with `output_dir` pointed at `/workspace/datasets/v3_resume/`. Files on the volume are not lost.

### Example 3 — User asks which MCP tool to call

> **User:** "I want to test my service locally before deploying."
>
> **Agent action:** Consult the **Service execution model** table. Use `test_service` (ephemeral container per job, mounts service dir at `/service`, reads local `service.yaml` as source of truth). Do **not** use `get_service_details` for the schema during in-development work — read the local `service.yaml` directly.

### Example 4 — User asks about a path they're unsure of

> **User:** "Is `/workspace/services/foo_abc123/` a place I should be editing?"
>
> **Agent action:** Consult the workspace directory tour. Answer: no — that path is platform-managed, read-only in normal workflows. Recommend developing services under `/workspace/<your-repo>/services/<service-name>/` instead.

## Configuration

This skill has no configuration. It is pure reference content. The behaviors it describes depend on:

- `RENDEREDAI_API_KEY` being set in the workspace environment (required by the `renderedai` CLI and `anatools` SDK)
- The MCP `raiservices-local` server being attached to the workspace (typically auto-attached)

## Error handling

| Symptom | Likely cause | Action |
|---------|--------------|--------|
| `FileNotFoundError` on a `/workspace/volumes/...` path mid-run | FUSE mount silently dropped under sustained writes | Ask user to remount; resume with output on `/workspace/datasets/` (local) |
| `get_service_details` returns a different schema than `service.yaml` | Deployed schema is stale relative to in-development code | Trust the local `service.yaml`; use `test_service` (it mounts the service dir, ignoring the deployed schema) |
| `test_service` fails to find Dockerfile | Dockerfile not at `<service_dir>/.devcontainer/Dockerfile` | Move Dockerfile to the required path; `test_service` does not search elsewhere |
| `run_service` on a persistent service spins up an ephemeral container | Wrong MCP tool for the deployment mode | Use `run_persistent_service` to address the queue on the persistent instance |
| `docker run` for in-repo training/eval can't see `/workspace/volumes/` data | Container only sees what's explicitly mounted | Mount both `/workspace/volumes` (source) and `/workspace/datasets` (writable output) |

## See also

- The full platform reference is in `AGENT_STUDIO.md` (next to this `SKILL.md`).
- For the `renderedai` CLI itself, see [`renderedai-cli` SKILL.md](https://github.com/renderedai/anatools/blob/8f94b288d672ebc676a2e25b959058e61389eb72/skills/renderedai-cli/SKILL.md). Do not duplicate CLI help here.
