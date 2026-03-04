# Tech Debt Management

How to identify, categorize, measure, prioritize, and communicate technical debt. Practical framework for Go Gin API projects.

## Table of Contents

- [What Is Tech Debt (And What Isn't)](#what-is-tech-debt)
- [The Debt Quadrant](#the-debt-quadrant)
- [Identifying Debt](#identifying-debt)
- [Categorizing Debt](#categorizing-debt)
- [Measuring Debt](#measuring-debt)
- [Prioritizing Debt](#prioritizing-debt)
- [Refactoring Strategies](#refactoring-strategies)
- [Communicating to Stakeholders](#communicating-to-stakeholders)
- [Prevention](#prevention)

---

## What Is Tech Debt (And What Isn't)

**Tech debt IS:**
- Code shortcuts taken with awareness for speed ("we'll fix this after launch")
- Outdated patterns that slow down new feature development
- Missing tests for critical paths
- Hardcoded values that should be configurable
- Tight coupling between modules that should be independent

**Tech debt is NOT:**
- Code you don't like the style of
- A different approach than what you would have chosen
- Code written by someone less experienced (that's mentoring, not debt)
- Features that aren't built yet (that's a backlog)

**Rule:** If it doesn't slow down current development or create real risk, it's not debt worth tracking.

---

## The Debt Quadrant

Based on Martin Fowler's tech debt quadrant:

```
                    Deliberate                    Inadvertent
              ┌─────────────────────┬─────────────────────┐
              │                     │                     │
  Reckless    │ "We know this is    │ "What's a           │
              │  wrong but we ship  │  repository          │
              │  anyway"            │  pattern?"           │
              │                     │                     │
              │ → High priority     │ → Training +         │
              │   to fix            │   gradual refactor  │
              ├─────────────────────┼─────────────────────┤
              │                     │                     │
  Prudent     │ "We'll use a simple │ "Now we know how    │
              │  approach now and   │  this should have   │
              │  refactor when we   │  been built"        │
              │  learn more"        │                     │
              │                     │                     │
              │ → Schedule when     │ → Refactor when     │
              │   pain is real      │   touching the code │
              └─────────────────────┴─────────────────────┘
```

**Prudent deliberate debt is OK.** You made a conscious tradeoff. Document it (ADR) and move on.

**Reckless deliberate debt is the enemy.** Fix it before it compounds.

---

## Identifying Debt

### Code Signals

| Signal | Debt Type | Severity |
|---|---|---|
| `// TODO`, `// HACK`, `// FIXME` | Explicit | Check each one |
| Duplicated code blocks | DRY violation | Medium |
| Functions > 50 lines | Complexity | Low-Medium |
| Files > 200 lines | Organization | Low-Medium |
| No tests for business logic | Safety | High |
| Hardcoded URLs, secrets, magic numbers | Configuration | Medium-High |
| `interface{}` / `any` used for laziness | Type safety | Medium |
| Commented-out code blocks | Dead code | Low |
| Error ignored: `_ = someFunc()` | Reliability | High |
| No context propagation | Observability | Medium |
| `fmt.Println` instead of `slog` | Observability | Low |

### Architecture Signals

| Signal | Debt Type | Severity |
|---|---|---|
| Handler contains SQL queries | Layer violation | High |
| Circular package dependencies | Coupling | High |
| No database migrations (manual DDL) | Process | High |
| Single test for entire feature | Coverage | Medium |
| No health check endpoint | Operations | Medium |
| No graceful shutdown | Reliability | Medium |
| Secrets in code or git | Security | Critical |
| No rate limiting on public endpoints | Security | High |
| No input validation | Security | High |

### Process Signals

| Signal | What It Means |
|---|---|
| PRs take > 1 day to review | Code too complex or too large |
| Same bug appears twice | Missing test coverage |
| New devs take > 2 weeks to be productive | Poor docs or complex code |
| Fear of changing certain files | Fragile code, missing tests |
| "Don't touch that, it works" | Technical tombstone — debt is severe |
| Deploy takes > 30 minutes | Build/CI debt |

---

## Categorizing Debt

Group debt into categories for easier prioritization:

| Category | Examples | Owner |
|---|---|---|
| **Security** | Hardcoded secrets, SQL injection, missing auth | Fix immediately |
| **Reliability** | Missing error handling, no graceful shutdown, ignored errors | Sprint priority |
| **Testing** | No tests for critical paths, flaky tests, no CI | Sprint priority |
| **Architecture** | Layer violations, circular deps, tight coupling | Plan & schedule |
| **Code Quality** | Duplicated code, long functions, naming | Boy scout rule |
| **Operations** | Manual deploys, no monitoring, no backups | Plan & schedule |
| **Documentation** | Missing API docs, outdated README, no ADRs | Ongoing |

---

## Measuring Debt

### Lightweight Debt Inventory

Use a simple table (in a markdown file or issue tracker):

```markdown
| ID | Description | Category | Severity | Effort | Files Affected | Created |
|---|---|---|---|---|---|---|
| TD-001 | No tests for payment flow | Testing | High | 2d | internal/payment/* | 2026-01 |
| TD-002 | User handler has SQL queries | Architecture | High | 1d | internal/handler/user.go | 2026-01 |
| TD-003 | Duplicated validation logic | Code Quality | Medium | 0.5d | internal/handler/*.go | 2026-02 |
| TD-004 | No rate limiting on public API | Security | High | 1d | cmd/api/main.go | 2026-02 |
```

### Effort Scoring

| Score | Meaning | Examples |
|---|---|---|
| **XS** (< 1h) | Quick fix | Remove dead code, fix TODO, add missing error check |
| **S** (1h-4h) | Small refactor | Extract function, add input validation, add test |
| **M** (1d-2d) | Medium refactor | Split large file, add test suite, implement middleware |
| **L** (3d-5d) | Large refactor | Restructure package, implement missing layer |
| **XL** (1w+) | Major rework | Change architecture pattern, migrate database |

### Metrics That Matter

Track these over time to see if debt is increasing or decreasing:

```go
// Automated checks you can add to CI
// 1. Test coverage (go test -coverprofile)
// 2. TODO/FIXME count: grep -r "TODO\|FIXME\|HACK" --include="*.go" | wc -l
// 3. Function length: use gocognit or gocyclo
// 4. Lint warnings: golangci-lint run --out-format json | jq '.Issues | length'
```

**Don't track vanity metrics** (like line count or file count). Track things that correlate with developer pain.

---

## Prioritizing Debt

### Priority Matrix

```
                High Impact
                    │
        ┌───────────┼───────────┐
        │  SCHEDULE  │  FIX NOW  │
        │  (plan it) │  (urgent) │
        │           │           │
  Low ──┼───────────┼───────────┼── High
  Effort│           │           │   Effort
        │  BOY SCOUT│  EVALUATE │
        │  (do it   │  (is it   │
        │   in-line)│   worth   │
        │           │   it?)    │
        └───────────┼───────────┘
                    │
                Low Impact
```

| Quadrant | Strategy | Example |
|---|---|---|
| High impact, low effort | **Fix now** — best ROI | Add missing error handling |
| High impact, high effort | **Schedule** — plan a sprint | Restructure package layout |
| Low impact, low effort | **Boy scout** — fix when touching the file | Rename variable, remove dead code |
| Low impact, high effort | **Evaluate** — probably not worth it | Rewrite working code for style |

### Prioritization Checklist

For each debt item, score 1-5 on:

1. **Frequency:** How often does this slow someone down? (1=rarely, 5=every PR)
2. **Severity:** How bad is it when it bites? (1=annoying, 5=data loss/security)
3. **Effort:** How hard to fix? (1=trivial, 5=major rework)
4. **Spread:** How many files/features does it affect? (1=isolated, 5=everywhere)

**Priority score = (Frequency + Severity + Spread) - Effort**

Higher score = fix sooner.

---

## Refactoring Strategies

### The Boy Scout Rule

> Leave the code better than you found it.

When touching a file for a feature, fix small debt items in the same PR. Don't make a separate "cleanup PR" for trivial changes.

**Applies to:** Code quality debt (naming, dead code, small duplications)

### The Strangler Fig Pattern

Gradually replace old code with new code, routing traffic between them.

```go
// Phase 1: New handler alongside old one
r.POST("/api/v1/orders", oldHandler.CreateOrder)          // existing
r.POST("/api/v2/orders", newHandler.CreateOrder)           // new implementation

// Phase 2: Route percentage of traffic to new
r.POST("/api/v1/orders", func(c *gin.Context) {
    if shouldUseNew(c) { // feature flag or percentage
        newHandler.CreateOrder(c)
        return
    }
    oldHandler.CreateOrder(c)
})

// Phase 3: All traffic to new, remove old
r.POST("/api/v1/orders", newHandler.CreateOrder)
```

**Applies to:** Large rewrites where you can't do a big-bang switch.

### The Parallel Change Pattern

1. Add new code alongside old code
2. Migrate callers one by one
3. Remove old code when no callers remain

```go
// Step 1: Add new method, keep old
type UserRepo struct { ... }

// Deprecated: use GetByIDV2 which returns proper errors
func (r *UserRepo) GetByID(id string) *User { ... }

func (r *UserRepo) GetByIDV2(ctx context.Context, id string) (*User, error) { ... }

// Step 2: Migrate callers to GetByIDV2
// Step 3: Delete GetByID when all callers migrated
```

**Applies to:** Interface changes, function signature changes, data model migrations.

### Dedicated Debt Sprints

Allocate 10-20% of sprint capacity to tech debt. Not a separate sprint — embedded in every sprint.

**How:**
- Pick 1-2 high-priority debt items per sprint
- Assign to team members who know the area best
- Review in sprint retrospective

---

## Communicating to Stakeholders

### The Business Translation

Stakeholders don't care about "technical debt." They care about:

| Tech Debt Term | Business Translation |
|---|---|
| "We have tech debt" | "New features will take longer to build" |
| "We need to refactor" | "We're investing in speed for the next 6 months" |
| "Missing tests" | "We can't guarantee changes won't break existing features" |
| "Architecture debt" | "Adding the next 3 features will cost 2x what they should" |
| "Security debt" | "We have known vulnerabilities that could be exploited" |

### The Debt Report Template

Share quarterly with stakeholders:

```markdown
# Tech Debt Report — Q1 2026

## Summary
- Total debt items: 23 (down from 28 last quarter)
- Critical items: 2 (security: 0, reliability: 2)
- Estimated total effort: 15 dev-days
- Items resolved this quarter: 8
- New items added: 3

## Top 5 Items (by priority score)

| # | Description | Impact | Effort | Plan |
|---|---|---|---|---|
| 1 | No tests for payment flow | High (revenue risk) | 2d | Sprint 4 |
| 2 | Manual deployment process | High (slow releases) | 3d | Sprint 5 |
| 3 | No rate limiting | High (abuse risk) | 1d | Sprint 4 |
| 4 | Duplicated auth logic | Medium (slow development) | 1d | Boy scout |
| 5 | No monitoring alerts | Medium (slow incident response) | 2d | Sprint 5 |

## Trend
[Chart or description showing debt count over time]

## Recommendation
Allocate 15% of Sprint 4-5 capacity to address items #1-3.
Expected outcome: faster feature delivery and reduced incident risk.
```

---

## Prevention

Best debt is debt you never take on.

### At Code Review Time

- Does this PR add complexity that isn't needed now?
- Are there tests for the happy path AND error cases?
- Is the new code consistent with existing patterns?
- Are secrets hardcoded? (auto-check with `gitleaks`)
- Does this touch a file that's already in the debt inventory?

### At Design Time

- Use complexity assessment (→ gin-architect) before building
- Write an ADR for non-obvious decisions
- Start with the simplest pattern that works
- Set up CI/CD and tests from day one

### Automated Guards

```yaml
# .golangci.yml — catches common debt patterns
linters:
  enable:
    - errcheck      # unchecked errors
    - govet         # suspicious constructs
    - staticcheck   # bugs, simplifications
    - gosimple      # simplify code
    - unused        # unused code
    - gocognit      # cognitive complexity (default threshold: 30)
    - funlen        # function length (default: 60 lines)
```

**Rule:** If a linter catches it automatically, you don't need to track it as debt. Fix it or configure the linter to accept it.
