# agent-skills

Personal collection of Claude Code plugins/skills, distributed via the marketplace defined in `.claude-plugin/marketplace.json`. Plugins live under `plugins/<name>/` with their manifest at `plugins/<name>/.claude-plugin/plugin.json` and skills under `plugins/<name>/skills/<skill>/SKILL.md`.

## Plugin versioning

Bump the `version` field in the plugin's `plugin.json` with **every user-facing change** — because the version is explicitly set, installed copies only pick up updates when it changes:

- **Patch** (0.1.0 → 0.1.1): updates/improvements to an existing skill.
- **Minor** (0.1.x → 0.2.0): new skills or significant new capabilities.

Include the version bump in the same commit as the skill change.
