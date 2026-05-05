# Agent Docs

Development guides for AI agents working on [Rendered.ai](https://rendered.ai) channels.

## Guides

- **[AGENT.md](AGENT.md)** — General channel development: commands, node schemas, determinism, anatools types, channel config, annotations, deployment, and troubleshooting.
- **[AGENT_BLENDER.md](AGENT_BLENDER.md)** — Blender-specific patterns: scene management, materials, lighting, cameras, and rendering within Rendered.ai channels.
- **[AGENT_DIRSIG.md](AGENT_DIRSIG.md)** — DIRSIG-specific patterns: the `dirfm` driver library, glist object/instance model, motion/flex-motion engines, platform sensors, truth-band annotations, and the `DIRSIG5` simulation lifecycle.
- **[AGENT_[redacted].md](AGENT_[redacted].md)** — [redacted] patterns for any human-centric synthetic data channel (office, pool, hospital, warehouse, …): addon setup, character factory, animation/pose system, annotation integration, determinism, and domain-adaptation checklist.
- **[AGENT_SDK.md](AGENT_SDK.md)** — SDK reference: authentication, graph upload, dataset creation, log download, and platform workflows.

## Usage

Include these files in your channel repository (e.g., at the repo root) so AI coding assistants automatically pick them up as context. They are designed to reduce common mistakes and accelerate development of new nodes, graphs, and channels.

## Contributing

Update the guides as you discover new patterns or pitfalls. Keep entries concise and example-driven.
