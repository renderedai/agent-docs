# Agent Docs

Development guides for AI agents working on the [Rendered.ai](https://rendered.ai) platform.

The material is split by audience. Pick the section that matches what you're building.

## For service developers (Agent Studio)

Agentically developing services (ML pipelines, data processors, API workers) that run inside a Rendered.ai **Agent Studio** workspace and interact with platform resources (datasets, volumes, models, channels). Applies to first- and third-party service developers.

- **[skills/agent-studio](skills/agent-studio/SKILL.md)** — Agent Studio platform reference: workspace storage layout (local NVMe vs FUSE), the FUSE timeout rule for long-running jobs, on-demand (`test_service`, `run_service`) vs persistent (`run_persistent_service`) execution, the `renderedai` CLI and `anatools` SDK, the MCP `raiservices-local` toolchain, and layered platform rules. Packaged as an installable skill; also readable directly as `skills/agent-studio/AGENT_STUDIO.md`.

## For channel developers (renderers and simulators)

Authoring synthetic-data channels — Blender, DIRSIG, Omniverse, and the graph/node system they run on.

### Guides

- **[AGENT.md](AGENT.md)** — General channel development: node/package/Dockerfile anatomy, the local `ana` dev container and path mappings, node class structure and anatools object types, package volumes, channel config, determinism, annotations, deployment workflow, and common mistakes.
- **[AGENT_BLENDER.md](AGENT_BLENDER.md)** — Blender-specific patterns: scene management, materials, lighting, cameras, and rendering within Rendered.ai channels.
- **[AGENT_DIRSIG.md](AGENT_DIRSIG.md)** — DIRSIG-specific patterns: the `dirfm` driver library, glist object/instance model, motion/flex-motion engines, platform sensors, truth-band annotations, and the `DIRSIG5` simulation lifecycle.
- **[AGENT_OMNIVERSE.md](AGENT_OMNIVERSE.md)** — Omniverse / Replicator-specific patterns: the declarative-graph mental model, randomizer contract, custom writer + annotators, preview mode, determinism with `rep.set_global_seed`, and stage-state leaks.
- **[AGENT_GRAPH.md](AGENT_GRAPH.md)** — Graph YAML authoring: node anatomy, hash versioning, port definitions, link rules, YAML 1.1 pitfalls, common errors, and graph-editing checklist.
- **[AGENT_SDK.md](AGENT_SDK.md)** — `anatools` Python SDK and companion CLIs (`anamount` for dataset mounting, `anatools-download-dataset` for bulk download): authentication, graph upload, dataset creation, log download, and platform workflows.

### Skills

Narrow, pitfall-focused agent-skill packages under `skills/` — each pairs a trigger description with a prescriptive reference for one recurring, high-friction task:

- **[skills/channel-node-io](skills/channel-node-io/SKILL.md)** — Silent-failure patterns in a node's `exec()`: unindexed `self.inputs`, output-key mismatches, the unlinked-optional-input `""` sentinel, and `FileObject`/`DirectoryObject` attribute access.
- **[skills/channel-schema-validation](skills/channel-schema-validation/SKILL.md)** — Node schema YAML pitfalls that only surface as a red border in the web editor (`type: float`, `oneOf` branch collisions, default/branch mismatches, the literal-or-wired-node port pattern).
- **[skills/channel-replay-run](skills/channel-replay-run/SKILL.md)** — Reproducing a platform dataset run locally with `ana`: dataset-name→ID lookup, seed/interp_num extraction, `graph.json`-vs-local-YAML drift, and pulling per-run platform logs.
- **[skills/blender-blend-inspect](skills/blender-blend-inspect/SKILL.md)** — Headless `bpy` inspection of a `.blend` file inside a channel's Docker image (object positions, actions, cameras, Geometry Nodes inputs).

## Usage

Include these files in your channel repository (e.g., at the repo root) so AI coding assistants automatically pick them up as context. They are designed to reduce common mistakes and accelerate development of new nodes, graphs, and channels.

## Contributing

Update the guides as you discover new patterns or pitfalls. Keep entries concise and example-driven.
