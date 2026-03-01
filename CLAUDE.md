# CLAUDE.md

Agent guidance for the `gin-best-practices` repository.

## What This Repo Is

A collection of 5 Agent Skills for building production-grade REST APIs with Go and the Gin framework. Published on [skills.sh](https://skills.sh). Each skill lives under `skills/` and follows the [Agent Skills open standard](https://agentskills.io).

## Repository Structure

```
gin-best-practices/
├── skills/
│   ├── gin-api/          # Core REST API — routing, handlers, binding, errors
│   ├── gin-auth/         # JWT auth, RBAC middleware, token lifecycle
│   ├── gin-database/     # PostgreSQL with GORM/sqlx, repository pattern
│   ├── gin-deploy/       # Docker, docker-compose, Kubernetes, CI/CD
│   └── gin-testing/      # Unit, integration, and e2e tests
├── CLAUDE.md             # This file (agent guidance)
├── AGENTS.md             # Multi-agent guidance
├── README.md             # Project overview
└── LICENSE               # MIT
```

## Skill Format

Each skill directory contains:
- `SKILL.md` — Primary skill file with YAML frontmatter (`name`, `description`, `license`, `metadata`). Under 500 lines. Loaded by agents automatically.
- `references/*.md` — Deep-dive reference files loaded on demand.
- `metadata.json` — Version, author, abstract, external references.
- `README.md` — Brief description and structure overview.

## Conventions

- **Gin API calls**: Must match official Gin documentation exactly. Never invent methods.
- **Binding**: Use `ShouldBind*` (not `Bind*`) in all examples.
- **Server setup**: Use `gin.New()` + explicit `r.Use(...)` (not `gin.Default()`).
- **Goroutine safety**: Call `c.Copy()` before passing `*gin.Context` to goroutines.
- **Logging**: Use `log/slog` (not `fmt.Println` or `log.Println`).
- **Context**: Pass `c.Request.Context()` to all downstream blocking calls.
- **Handlers**: Thin — bind input, call service, format response. No DB calls in handlers.

## Modifying Skills

1. Read official Gin documentation before writing any Gin code
2. Keep `SKILL.md` under 500 lines — move detail to `references/`
3. Use `ShouldBind*`, `gin.New()`, `c.Copy()`, `log/slog`, and `context.Context` consistently
