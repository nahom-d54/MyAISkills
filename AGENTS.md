# Agent Instructions — MyAISkills

Guidelines for AI agents working in this repository.

## Repository Overview

This repo stores agent skills — modular `SKILL.md` packages that extend GitHub Copilot and other Claude-based agents with specialized domain knowledge. Each skill lives in `skills/<skill-name>/` and is distributed as a compiled `packages/<skill-name>.skill` file.

```
skills/          # Skill source directories (one folder per skill)
packages/        # Compiled .skill packages ready for distribution
.github/agents/  # Custom agent definitions for this repo
```

## Creating or Modifying a Skill

Use the skill-creator scripts located at `/skills/skill-creator/scripts/`.

### Initialize a new skill

```bash
python /skills/skill-creator/scripts/init_skill.py <skill-name> --path skills/
```

This creates `skills/<skill-name>/` with a `SKILL.md` template and example `scripts/`, `references/`, and `assets/` directories. Delete any resource directories the skill doesn't need.

### SKILL.md structure

Every `SKILL.md` requires YAML frontmatter with exactly two fields:

```yaml
---
name: skill-name
description: What the skill does and when to use it (this is the trigger — be specific).
---
```

The body contains concise instructions written in imperative form. Keep it under 500 lines. Move large reference material into `references/` files and link to them from the body.

### Validate and package

```bash
python /skills/skill-creator/scripts/package_skill.py skills/<skill-name>
```

The script validates the skill and writes the `.skill` file to `packages/` on success. Always run this after modifying a skill.

## Updating the README

When adding or removing a skill, use the `@readme` agent or follow the format rules in [.github/agents/readme.agent.md](.github/agents/readme.agent.md). Key points:

- Add a row to the **Skills** table linking to `skills/<skill-name>/`
- Add the skill folder and `.skill` package to the **Repository Structure** block
- Do not change section order or the installation instructions prose

## Key Conventions

- **One skill per directory** — `skills/<skill-name>/SKILL.md` is the only required file
- **No auxiliary docs** — do not create `README.md`, `CHANGELOG.md`, or similar inside skill directories
- **Resource directories are optional** — only include `scripts/`, `references/`, or `assets/` if the skill actually uses them
- **Version-pin everything** — skill examples should demonstrate pinned versions, not `:latest` or vague ranges
- **Package after every change** — always re-run `package_skill.py` so `packages/` stays in sync with `skills/`
