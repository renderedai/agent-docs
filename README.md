# Agent Docs

Development guides for AI agents working on the [Rendered.ai](https://rendered.ai) platform.

## Agent Studio guides

Guides for agentically developing services that run on the Rendered.ai platform. Applies to first- and third-party service developers working in an Agent Studio workspace:

- **[agent-studio/AGENT_STUDIO.md](agent-studio/AGENT_STUDIO.md)** — Agent Studio workspace reference: storage layout (local vs FUSE), service execution models (on-demand vs persistent), MCP tooling, and common gotchas when developing services.

## Channel development guides

Guides for authoring synthetic data channels (renderers and simulators):

- **[AGENT.md](AGENT.md)** — SingleBlenderFile V1 (`reid-v1` / `charactersQA`) channel: `CameraDwellRender` node, character nodes, room reference, render performance, deployment, and common mistakes.
- **[AGENT_BLENDER.md](AGENT_BLENDER.md)** — Blender-specific patterns: scene management, materials, lighting, cameras, and rendering within Rendered.ai channels.
- **[AGENT_DIRSIG.md](AGENT_DIRSIG.md)** — DIRSIG-specific patterns: the `dirfm` driver library, glist object/instance model, motion/flex-motion engines, platform sensors, truth-band annotations, and the `DIRSIG5` simulation lifecycle.
- **[AGENT_GRAPH.md](AGENT_GRAPH.md)** — Graph YAML authoring: node anatomy, hash versioning, port definitions, link rules, YAML 1.1 pitfalls, common errors, and graph-editing checklist.
- **[AGENT_[redacted].md](AGENT_[redacted].md)** — [redacted] patterns for any human-centric synthetic data channel (office, pool, hospital, warehouse, …): addon setup, character factory, animation/pose system, annotation integration, determinism, and domain-adaptation checklist.
- **[AGENT_SDK.md](AGENT_SDK.md)** — `anatools` Python SDK and companion CLIs (`anamount` for dataset mounting, `anatools-download-dataset` for bulk download): authentication, graph upload, dataset creation, log download, and platform workflows.

## Usage

Include these files in your channel repository (e.g., at the repo root) so AI coding assistants automatically pick them up as context. They are designed to reduce common mistakes and accelerate development of new nodes, graphs, and channels.

## Contributing

Update the guides as you discover new patterns or pitfalls. Keep entries concise and example-driven.
