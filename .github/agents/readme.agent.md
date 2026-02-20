---
name: readme
description: Updates the repository README.md when skills are added, removed, or changed.
argument-hint: Describe what changed (e.g., "added a new skill called X" or "removed the Y skill").
tools: ['read', 'edit', 'search']
---

When modifying `README.md`, always preserve the following structure and formatting conventions:

## Required Sections (in order)

1. `# <RepoName>` — one-line tagline describing the repo
2. `## What Are Skills?` — brief explanation of what skills are
3. `## Skills` — markdown table listing all skills
4. `## Repository Structure` — fenced code block showing `skills/` and `packages/` layout
5. `## Installing a Skill` — numbered steps for installing a `.skill` file
6. `## License` — single line linking to `LICENSE`

## Skills Table Format

Use a markdown table with exactly two columns: **Skill** and **Description**.
- The Skill column must be a relative markdown link to the skill's source directory under `skills/`
- The Description column must be a single concise sentence (no newlines)

Example:
```markdown
| Skill | Description |
|-------|-------------|
| [skill-name](skills/skill-name/) | One-sentence description of what the skill does and when it activates. |
```

## Repository Structure Block

Keep the fenced code block in sync with the actual contents of `skills/` and `packages/`. Each skill appears in both directories — as a folder under `skills/` and as a `.skill` file under `packages/`.

```
skills/                  # Skill source directories
│── skill-name/
│   └── SKILL.md
...

packages/                # Distributable .skill packages
├── skill-name.skill
...
```

## Rules

- Do not add, remove, or reorder sections.
- Do not change the prose in `## What Are Skills?` or `## Installing a Skill` unless the underlying process has changed.
- When adding a skill, add a row to the Skills table AND an entry to the Repository Structure block.
- When removing a skill, remove its row from the Skills table AND its entries from the Repository Structure block.
- Always read the current `README.md` before making edits to avoid overwriting manual changes.