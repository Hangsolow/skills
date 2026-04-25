# Copilot instructions for this repository

## Build, test, and lint commands

This repository does not currently define repo-local build, test, or lint scripts in checked-in manifests or CI files.

For skill validation, follow the Agent Skills spec and use the reference validator when available:

```bash
skills-ref validate ./plugins/<plugin-name>/skills/<skill-name>
```

There is no repo-defined single-test command beyond validating an individual skill directory this way.

## High-level architecture

This repository publishes **agent skills** and groups them into plugins for Copilot-compatible marketplaces.

- `README.md` is the public index. It lists available plugins, links to their skills, and shows installation guidance for consuming the marketplace.
- Each plugin lives under `plugins/<plugin-name>/`.
- Each plugin has a minimal `plugin.json` that declares plugin metadata and points to its `skills` directory.
- Each skill lives under `plugins/<plugin-name>/skills/<skill-name>/` and follows the [Agent Skills specification](https://agentskills.io/specification).
- A skill directory must contain `SKILL.md` and may also contain `references/`, `scripts/`, and `assets/`.

The current `optimizely-cms` plugin is the example to follow:

- `plugin.json` exposes `./skills/`
- `skills/cms13-migration/SKILL.md` is the primary, task-oriented guide
- `skills/cms13-migration/references/*.md` holds topic-specific deep dives and lookup tables

The repo’s content model is **progressive disclosure**: keep the main skill file focused on task flow and decision points, and move dense technical inventories into reference files that can be loaded on demand.

## Key conventions

- Preserve valid Agent Skills frontmatter in every `SKILL.md`. The `name` must match the skill directory name, use lowercase letters/numbers/hyphens, and the file should keep metadata such as `description` and `compatibility` in YAML frontmatter.
- Keep `SKILL.md` as the activation entrypoint, not a dump of every detail. The existing skill uses the main file for workflow, sequencing, gotchas, and common failure patterns, while `references/*.md` stores larger API maps and domain-specific tables.
- Prefer shallow relative references from `SKILL.md`, such as `references/api-replacement-map.md`, instead of chaining across deeply nested docs.
- Follow the existing writing style for technical guidance: concrete migration steps, explicit “Before/After” code snippets, direct old-to-new API mappings, and short topic-specific tables.
- Call out silent or easy-to-miss failures prominently in the main skill when they materially affect outcomes. The current skill surfaces cases like validator registration, DI namespace changes, `SiteDefinition` replacement, and content/tab naming rules early because they are common migration traps.
- Keep `plugin.json` minimal and declarative. The current pattern is `name`, `description`, `skills`, and `version`.
- When adding or renaming plugins or skills, update `README.md` so the marketplace index, install guidance, and links stay in sync with the filesystem.
