# Agent Studio — Platform Reference

Operational guide for AI agents and engineers developing **services** on the Rendered.ai **Agent Studio** platform. Agent Studio is the workspace runtime where you build, test, and deploy long-running and on-demand services (ML pipelines, data processors, API workers) that interact with platform resources (datasets, volumes, models, channels).

> **Disambiguation.** "Studio" in the Rendered.ai ecosystem refers to two distinct products with different platform architectures:
>
> - **Agent Studio** (this doc) — the workspace/service-development environment you are currently in. Files live under `/workspace/`, services run via the MCP server or the `renderedai` CLI, source datasets and persistent artifacts are FUSE-mounted from platform storage.
> - **Channel studio / synthetic-data channels** — the renderer-driven simulation product (Blender / DIRSIG / [redacted] / etc.). Covered by the other docs in this folder: `AGENT.md`, `AGENT_BLENDER.md`, `AGENT_DIRSIG.md`, `AGENT_[redacted].md`, `AGENT_GRAPH.md`, `AGENT_legacy.md`.
>
> Both products share the same `renderedai` Platform CLI (shipped by the `anatools` package). The CLI is documented in the [`renderedai-cli` SKILL.md](https://github.com/renderedai/anatools/blob/8f94b288d672ebc676a2e25b959058e61389eb72/skills/renderedai-cli/SKILL.md). **This doc does not redocument the CLI** — when you need command syntax, read SKILL.md or run `renderedai --help`.

## Install as a skill (recommended)

This document is also packaged as the **`agent-studio` agent skill** (org-scope). Enabling it gives every Agent Studio workspace in the org an AI agent that already knows about the FUSE timeout rule, on-demand vs persistent execution, MCP tooling, and the rest of this reference — **without** having to vendor `agent-docs/` into each project repo.

Skills are administered via the `raiservices-local` MCP tool (or the platform web UI). The `renderedai` CLI does **not** expose skill management as of this writing.

```text
# From inside an Agent Studio session (Cascade, Claude Code with MCP, etc.):
manage_skill(action='list', scope='organization')
# → find the agent-studio skill_id

manage_skill(action='enable', skill_id='<agent-studio-skill-id>')
# → enables it in the current workspace
```

The skill is **disabled by default at org scope**. Enable it per workspace as you onboard projects. Host-native agents (Claude Code, Codex, Gemini CLI) auto-discover enabled skills from their skill directory; Cascade currently requires `manage_skill` queries to surface the same content, so until the platform wires MCP-side skill injection, you may still want to read this file directly when starting a session.

If you're reading this doc inside a project repo (vendored from `agent-docs/`), prefer the skill in the long run — it stays versioned at the platform layer instead of drifting between vendored copies.

## Toolchain (do not redocument here)

| Tool | Where | When to use |
|------|-------|-------------|
| `renderedai` CLI | `/workspace/anaenv/bin/renderedai` (installed by `anatools`) | All platform-level operations: workspaces, organizations, datasets, volumes, graphs, channels, services, service-jobs, ml-models, ml-inferences, GAN models, rules, … JSON output by default, automation-friendly. **See [SKILL.md](https://github.com/renderedai/anatools/blob/8f94b288d672ebc676a2e25b959058e61389eb72/skills/renderedai-cli/SKILL.md).** |
| MCP server `raiservices-local` | Auto-attached in this workspace | Service lifecycle from inside Cascade: `test_service`, `run_service`, `run_persistent_service`, `get_services`, `get_service_detail`, `list_volumes`, `create_volume`, `manage_skill`, `get_rules` / `edit_rules`, etc. |
| Python SDK (`anatools`) | `from anatools import …` inside `/workspace/anaenv` | Programmatic access to the same platform from scripts. Useful for inspection, glue, and CI. |

`renderedai --help` enumerates the available resources. Do not duplicate that help text in service docs — link to it.

## Workspace storage layout

The Agent Studio workspace mounts several distinct storage tiers under `/workspace/`. **Where you write matters — long-running jobs can crash if they target a remote tier under sustained write load.**

| Path | Backing | Read | Write | Use for |
|------|---------|:----:|:-----:|---------|
| `/workspace/datasets/`, `/workspace/<repo>/` (e.g. `/workspace/fingerprints/`), `/workspace/agent-docs/`, anything else under `/dev/root` | Local NVMe on the workspace host | ✅ | ✅ | **All writes from long-running jobs** (training, evaluation, bulk preprocessing). Working copies of source data. Source repos. |
| `/workspace/volumes/<sanitized_name>/` | FUSE NFS mount (`renderedai-prod-volumes:/<orgUUID>/<volumeUUID>`) | ✅ | ⚠️ short bursts only | Platform volumes — persistent shared storage across workspaces in an org. Final model artifacts, source datasets, anything that needs to outlive the workspace. |
| `/home/ubuntu/.renderedai/workspaces/<workspaceUUID>/datasets/<datasetUUID>/` | FUSE NFS mount (`renderedai-prod-data:/<workspaceUUID>`), mounted **read-only** | ✅ | ❌ | Platform-managed dataset access — read-only view of datasets produced by channels and held in workspace storage. |
| `/workspace/services/<sanitized_name>/` | Platform-managed | ✅ | ❌ in normal workflows | Deployed-service documentation / templates surfaced by the platform. **Off-limits for our service-development repos** — develop services in your own `/workspace/<repo>/services/<service-name>/` instead. |

Identify the backing of any path with `df -h <path>`:

```bash
df -h /workspace/volumes/fingerprints_929e /workspace/datasets / 2>/dev/null | grep -v '^$'
# /dev/root              ...                local NVMe
# renderedai-prod-volumes:/<org>/<volume>   FUSE platform volume
# renderedai-prod-data:/<workspace>         FUSE workspace dataset access
```

### The FUSE timeout rule

**FUSE-backed paths (`/workspace/volumes/*` and `/home/ubuntu/.renderedai/workspaces/.../datasets/*`) silently drop the mount under sustained writes.** A training loop that writes a status file + checkpoint every epoch will, after hours of runtime, get `FileNotFoundError` on a path that existed seconds earlier. The backing storage is fine — the FUSE client lost its connection and the kernel returns `ENOENT` to any subsequent path lookup.

Concrete reference failure: a fingerprint UNet training run crashed at epoch 12 on 2026-06-06 01:24 UTC, ~4 h into the job, when its per-epoch `training_status.json` write to `/workspace/volumes/.../v3_tape_can_weighted/` returned `FileNotFoundError`. The 293 MB checkpoint written ~22 minutes earlier to the same directory was on disk; the mount itself had dropped silently between writes.

**Rules**

1. **Never set `output_dir` of a long-running job to `/workspace/volumes/...`.** Use `/workspace/datasets/<run_name>/` (local) instead.
2. **Copy the final artifact** (`model_best.pth`, eval JSON, etc.) to the FUSE volume at the end of the run if you want cross-workspace persistence.
3. **For read-heavy training that re-reads source files every epoch**, copy the dataset to local disk before starting. FUSE reads are fine in short bursts but timeout risk accumulates over hours.
4. **If a FUSE mount drops mid-session**, ask the user to remount; the files are not lost. Do not retry the failed write into the same path until the mount is back.
5. **Always `df -h /` before launching anything that will write more than a few GB** to confirm local disk has headroom. Local disk is shared across all concurrent jobs in the workspace.

### Local disk hygiene

```bash
df -h / | tail -1
# /dev/root  495G  307G  189G  62% /
```

Workspaces are typically provisioned with a few hundred GB of local NVMe. Training checkpoints (~300 MB each, kept top-3 by default) and copies of source datasets are the usual heavy users. Clean up old `training_output/` directories when a run is no longer needed.

## Service execution model

A "service" on Agent Studio is a Docker-packaged tool with a declared `service.yaml` schema. Services execute in two modes:

### On-demand (ephemeral container per job)

| MCP tool | Use case |
|----------|----------|
| `test_service` | Development. Builds (or reuses) the image from `<service_dir>/.devcontainer/Dockerfile`, mounts the service dir to `/service` in the container, runs one job, tears down. |
| `run_service` | Same lifecycle but for a service that's already deployed via the platform. |

- Each job spins up a fresh container — slow startup amortizes badly for repeated small jobs.
- Best for: training jobs, batch evaluations, one-off transforms, anything that runs for minutes-to-hours per invocation.
- **Source of truth for in-development service schemas is the local `service.yaml`, not `get_service_details`** (which returns the deployed schema, which may be stale).

### Persistent (long-lived EC2, queue of jobs)

| MCP tool | Use case |
|----------|----------|
| `run_persistent_service` | Submit a job to a persistent service. |
| `create_persistent_service` / `start_persistent_service` / `stop_persistent_service` / `restart_persistent_service` | Lifecycle management. |

- An EC2 instance stays warm, accepts queued jobs, optionally auto-scales down on idle.
- Best for: services with expensive cold-start (loading large models, warming caches), API workers, anything that benefits from amortized startup cost.
- Configure a `startup_payload` when creating a persistent service if it needs to launch a server or load models on boot. Pair with a `health_check_*` configuration so jobs wait until the server is actually ready.

### Choosing between them

| Job profile | Mode |
|-------------|------|
| 1 job per day, runs for hours | on-demand |
| Many small jobs (seconds each), latency-sensitive | persistent |
| ML inference with multi-GB models | persistent |
| One-shot training run | on-demand |
| Continuous worker (FastAPI / RabbitMQ / DB-NOTIFY) | persistent — or just `docker compose up -d worker` in this workspace if it's purely workspace-local |

## Workspace directory tour (illustrative)

A typical Agent Studio workspace looks like:

```
/workspace/
├── agent-docs/                                 # this docs repo
├── anaenv/                                     # Python venv with anatools + renderedai CLI
├── datasets/                                   # LOCAL: working data, training output
│   ├── <datasetUUID>/                          #   copies of FUSE datasets used for training
│   └── training_output/                        #   in-flight training jobs write here
├── <your-repo>/                                # LOCAL: service-development repo
│   └── services/
│       ├── <service-name>/                     #   one service per subdir
│       │   ├── .devcontainer/Dockerfile        #   required path for test_service
│       │   ├── service.yaml                    #   tools, inputs, outputs schema
│       │   ├── AGENTS.md                       #   service-specific agent instructions
│       │   ├── README.md
│       │   └── …source code…
│       └── …
├── services/                                   # PLATFORM-MANAGED — read-only docs/templates
│   └── <service-name>_<id>/                    #   sanitized-name dir for deployed services
│       ├── AGENTS.md                           #   (read these as reference; don't edit)
│       └── README.md
├── volumes/                                    # FUSE: persistent shared storage
│   └── <sanitized_volume_name>/                #   e.g. /workspace/volumes/fingerprints_929e/
└── workspaces/                                 # FUSE (read-only): platform dataset access
    └── <workspaceUUID>/datasets/<datasetUUID>/ #   synthetic-channel outputs, etc.
```

> Develop services under your own repo (`/workspace/<your-repo>/services/<service-name>/`), not under `/workspace/services/`. The latter is platform-managed and reserved for deployed-service surface.

## Rules and memory system

The platform enforces rules in a layered scope (most-specific wins):

| Scope | MCP access | Typical use |
|-------|------------|-------------|
| **User** | `get_rules(rule_type='User')` / `edit_rules(...)` | Personal preferences across all workspaces |
| **Organization** | `rule_type='Organization', organization_id=…` | Org-wide policies (output file conventions, naming, …) |
| **Workspace** | `rule_type='Workspace', workspace_id=…` | Per-project rules (output paths, dataset conventions) |
| **Service** | `rule_type='Service', service_id=…` | Service-specific operational rules |
| **Platform** | `rule_type='Platform'` (read-only) | Vendor defaults |

Cascade also has a separate, private memory database for free-form session context (`create_memory`). Platform rules are visible to every agent/user in scope; the Cascade memory db is private to the Cascade installation.

## Common gotchas

- **FUSE timeouts** — see "The FUSE timeout rule" above. Most common cause of training crashes.
- **`/workspace/services/` is read-only / platform-managed.** Do not develop services there. Use `/workspace/<your-repo>/services/<service-name>/`.
- **Deployed service schemas can be stale.** When iterating on an in-development service, read the local `service.yaml` directly; `get_service_details` may return an older deployed version. The `test_service` tool always uses the local code+schema.
- **`test_service` Dockerfile path requirement.** Dockerfile must be at `<service_dir>/.devcontainer/Dockerfile` — other locations are not supported.
- **`run_persistent_service` vs `run_service`.** A service deployed as persistent must be addressed via `run_persistent_service`; calling `run_service` on its service_id spins up a new ephemeral container and bypasses the persistent instance entirely.
- **`renderedai` CLI auth.** Requires `RENDEREDAI_API_KEY` env var. Set in this workspace shell; **not** visible to deployed services unless explicitly forwarded via the service container env.
- **`docker run` for in-repo training/testing.** When you bypass `test_service` and `docker run` directly (e.g., in a service's own `AGENTS.md` recipe), remember to mount **both** `/workspace/volumes` (source data) and `/workspace/datasets` (writable output) — they are separate filesystems and the container only sees what you mount.
- **`psql -c` output capture.** When piping a `RETURNING id` value into a shell variable, use `-tAc` and `tr -d` — the default `psql -c` formatting appends `INSERT 0 1` and breaks downstream uses.

## When to update this doc

Add here when you discover **platform-level** behavior that applies across services — mount behavior, CLI conventions, MCP tool quirks, workspace structure changes. Keep **service-specific** gotchas (pipeline parameter conventions, model paths, build steps) in that service's own `AGENTS.md`, not here.
