# MyAISkills

A collection of agent skills that extend GitHub Copilot (and other Claude-based AI assistants) with specialized domain knowledge and best practices.

## What Are Skills?

Skills are modular packages (`SKILL.md` + optional resources) that give an AI agent procedural knowledge it wouldn't otherwise have — think of them as domain-specific onboarding guides that wire into the agent's context automatically.

Once installed, a skill activates when the user's request matches its description, injecting the right guidance at the right time.

## Skills

| Skill                                                          | Description                                                                                                                                                                |
| -------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [dockerfile-best-practices](skills/dockerfile-best-practices/) | 10 production rules for secure, efficient, minimal Docker images — covering `.dockerignore`, layer cleanup, secrets handling, multi-stage builds, health checks, and more. |
| [fastapi-development](skills/fastapi-development/)             | Build high-performance Python APIs with FastAPI — async routes, dependency injection, validation, security, and automatic OpenAPI docs.                                    |

## Repository Structure

```
skills/                  # Skill source directories
│── dockerfile-best-practices/
│   └── SKILL.md
└── fastapi-development/
    └── SKILL.md

packages/                # Distributable .skill packages
├── dockerfile-best-practices.skill
└── fastapi-development.skill
```

## Installing a Skill

1. Download the `.skill` file from `packages/`
2. Install it into your Copilot skills directory (e.g. `~/.copilot/skills/`)
3. The skill activates automatically when relevant queries are detected

## License

MIT — see [LICENSE](LICENSE)
