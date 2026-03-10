# AGENTS.md

Multi-agent guidance for the `golang-gin-best-practices` repository.

## Repository Overview

7 Agent Skills for Go/Gin REST API development, published on skills.sh. Skills live under `skills/`, build docs under `docs/`.

## Agent Roles

### Skill Author
- Writes or updates `SKILL.md` and `references/*.md` files
- Must read `docs/GIN-API-REFERENCE.md` before writing any Gin code
- Must read `docs/SPECIFICATION.md` for content requirements
- Keeps `SKILL.md` under 500 lines; moves overflow to reference files

### Reviewer
- Verifies all Gin API calls against `docs/GIN-API-REFERENCE.md`
- Checks `ShouldBind*` usage (not `Bind*`), `gin.New()` (not `gin.Default()`), `c.Copy()` for goroutines
- Validates frontmatter fields in `SKILL.md`
- Runs quality checklist from `docs/SPECIFICATION.md`

### Packager
- Generates zip files under `skills/` (one per skill)
- Updates `metadata.json` version fields
- Ensures skill folder structure matches spec

## File Ownership

| Path | Owner |
|------|-------|
| `skills/*/SKILL.md` | Skill Author |
| `skills/*/references/*.md` | Skill Author |
| `skills/*/metadata.json` | Packager |
| `skills/*.zip` | Packager |
| `docs/*` | Skill Author |
| `README.md` | Packager |

## Conventions

- All code examples must compile, handle errors, use `context.Context`, and use `log/slog`
- Use the shared `User` domain model across all skills (defined in `docs/SPECIFICATION.md`)
- Cross-skill references use skill name (e.g., "see the **gin-api** skill"), not file paths
- Conventional commits: `feat:`, `fix:`, `docs:`, `refactor:`
