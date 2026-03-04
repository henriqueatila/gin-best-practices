# Architecture Decision Records (ADRs)

Lightweight templates for documenting architecture decisions. Each ADR captures the context, decision, alternatives considered, and consequences. Store in `docs/adr/` in your project.

## Table of Contents

- [ADR Format](#adr-format)
- [When to Write an ADR](#when-to-write-an-adr)
- [ADR Lifecycle](#adr-lifecycle)
- [Template: General Decision](#template-general-decision)
- [Template: Database Choice](#template-database-choice)
- [Template: Auth Strategy](#template-auth-strategy)
- [Template: Caching Layer](#template-caching-layer)
- [Template: Service Extraction](#template-service-extraction)
- [Template: API Versioning](#template-api-versioning)
- [Worked Example: Choosing PostgreSQL over MongoDB](#worked-example)
- [ADR Index Template](#adr-index)

---

## ADR Format

Every ADR follows this structure:

```markdown
# ADR-NNN: [Short Decision Title]

**Status:** Proposed | Accepted | Deprecated | Superseded by ADR-XXX
**Date:** YYYY-MM-DD
**Deciders:** [who was involved]

## Context

What is the problem? Why are we making this decision now?
What forces are at play (technical, business, team)?

## Decision

What did we choose? State it clearly in one sentence, then elaborate.

## Alternatives Considered

### Option A: [Name]
- Pros: ...
- Cons: ...
- Why rejected: ...

### Option B: [Name]
- Pros: ...
- Cons: ...
- Why rejected: ...

## Consequences

### Positive
- What gets better?

### Negative
- What gets worse or harder?

### Risks
- What could go wrong?

## Follow-up Actions
- [ ] Action items resulting from this decision
```

**Rules:**
- Number sequentially (ADR-001, ADR-002, ...)
- Never delete an ADR — mark as `Deprecated` or `Superseded by ADR-XXX`
- Keep short — 1-2 pages max. If you need more, the decision is too big (split it)
- Write ADRs for decisions, not for implementations

---

## When to Write an ADR

Write an ADR when:
- Choosing between 2+ technologies (database, framework, cloud service)
- Changing an established pattern (new auth strategy, new error handling)
- Adding infrastructure (caching layer, message queue, new service)
- Making an irreversible or expensive-to-reverse decision
- The decision will affect multiple teams or future developers

**Don't write an ADR for:**
- Obvious choices (using PostgreSQL for a CRUD app)
- Minor refactoring
- Bug fixes
- Style preferences

---

## ADR Lifecycle

```
Proposed → Accepted → [lives forever]
                   → Deprecated (no longer relevant)
                   → Superseded by ADR-XXX (replaced by newer decision)
```

---

## Template: Database Choice

```markdown
# ADR-NNN: Database Selection for [Feature/Service]

**Status:** Proposed
**Date:** YYYY-MM-DD

## Context

[Describe the data requirements: volume, access patterns, consistency needs,
team familiarity, operational budget]

## Decision

Use [PostgreSQL / MongoDB / Redis / etc.] as the primary datastore for [scope].

## Evaluation Criteria

| Criteria | Weight | Option A | Option B | Option C |
|---|---|---|---|---|
| Team familiarity | High | ... | ... | ... |
| Query flexibility | Medium | ... | ... | ... |
| Operational cost | High | ... | ... | ... |
| Scalability path | Low | ... | ... | ... |
| Ecosystem (Go drivers, tools) | Medium | ... | ... | ... |

## Alternatives Considered

### PostgreSQL
- Pros: ACID, mature ecosystem, pgvector/PostGIS extensions, team knows SQL
- Cons: ...

### MongoDB
- Pros: Flexible schema, easy horizontal scaling
- Cons: Eventual consistency by default, weaker joins, team learning curve

## Consequences

[What this means for the project going forward]

## Follow-up Actions
- [ ] Set up database infrastructure
- [ ] Define schema / data model
- [ ] Configure connection pooling
```

---

## Template: Auth Strategy

```markdown
# ADR-NNN: Authentication and Authorization Strategy

**Status:** Proposed
**Date:** YYYY-MM-DD

## Context

[Describe: who are the users? Internal/external? Mobile/web/API?
What are the security requirements? Compliance needs?]

## Decision

Use [JWT with refresh tokens / session cookies / OAuth2 + OIDC / API keys]
for authentication. Use [RBAC / ABAC / simple role check] for authorization.

## Alternatives Considered

### JWT with Refresh Tokens
- Pros: Stateless verification, works across services, mobile-friendly
- Cons: Can't revoke individual tokens without a blocklist
- Token flow: access (15min) + refresh (7d) + rotation on refresh

### Session Cookies
- Pros: Simple, revocable, well-understood
- Cons: Requires session store (Redis), not ideal for mobile or microservices

### OAuth2 + External Provider
- Pros: Offload auth complexity, social login support
- Cons: Vendor dependency, more complex integration

## Consequences

[Impact on middleware, token storage, session management, deployment]
```

---

## Template: Caching Layer

```markdown
# ADR-NNN: Caching Strategy for [Feature/System]

**Status:** Proposed
**Date:** YYYY-MM-DD

## Context

[Describe: what's slow? What's the read/write ratio?
Current latency? Target latency? Data freshness requirements?]

## Decision

Use [HTTP cache headers / in-memory cache / Redis] for [scope].
Cache TTL: [duration]. Invalidation strategy: [TTL / write-through / event-driven].

## Data Analysis

| Endpoint | Current p95 | Read/Write Ratio | Staleness Tolerance |
|---|---|---|---|
| GET /products | 200ms | 50:1 | 5 minutes |
| GET /users/:id | 50ms | 10:1 | 30 seconds |
| GET /orders | 100ms | 3:1 | 0 (real-time) |

## Alternatives Considered

### No Cache (Optimize Queries)
- Pros: Simplest, no cache invalidation bugs
- Cons: May not meet latency targets
- **Try this first** before adding a cache layer

### In-Memory (ristretto)
- Pros: Fastest reads, no network hop, zero infrastructure
- Cons: Per-instance (not shared), lost on restart
- Best for: < 100MB data, single-instance or acceptable inconsistency

### Redis
- Pros: Shared across instances, TTL support, rich data structures
- Cons: Network hop, operational overhead, one more thing to monitor
- Best for: > 100MB data, multi-instance, need consistent cache

## Consequences

[Impact on consistency, latency, infrastructure, monitoring]
```

---

## Template: Service Extraction

```markdown
# ADR-NNN: Extract [Feature] into Separate Service

**Status:** Proposed
**Date:** YYYY-MM-DD

## Context

[Why now? What pain are we experiencing in the monolith?
Be specific: deploy coupling, scaling bottleneck, team contention]

## Decision

Extract [feature] into a standalone service communicating via [HTTP / gRPC / events].

## Extraction Checklist

- [ ] Feature has clear bounded context (few shared data dependencies)
- [ ] Team can independently deploy and operate the service
- [ ] CI/CD pipeline exists for the new service
- [ ] Monitoring and alerting configured
- [ ] Rollback plan defined
- [ ] Data migration plan (if splitting database)

## Boundary Definition

| Data/Entity | Stays in Monolith | Moves to New Service |
|---|---|---|
| Users | X | |
| Orders | | X |
| Payments | | X |
| Products | X | |

## Communication Pattern

| Interaction | Pattern | Why |
|---|---|---|
| Create order | Sync HTTP | Caller needs confirmation |
| Order shipped | Async event | Notification, non-blocking |
| Get user for order | Sync HTTP | Need data to process |

## Alternatives Considered

### Keep in Monolith + Modularize
- Pros: No distributed system complexity, shared transactions
- Cons: Doesn't solve [specific pain point]

### Extract as Library
- Pros: Code reuse without network boundary
- Cons: Still coupled deployment

## Consequences

[New operational requirements, monitoring needs, latency budget, team ownership]
```

---

## Template: API Versioning

```markdown
# ADR-NNN: API Versioning Strategy

**Status:** Proposed
**Date:** YYYY-MM-DD

## Context

[Why do we need versioning? Breaking changes coming?
How many API consumers? Internal/external?]

## Decision

Use URL path versioning (`/api/v1/`, `/api/v2/`) with minimum 3-month
deprecation notice for external consumers.

## Version Bumping Rules

| Change Type | Version Bump? | Example |
|---|---|---|
| Add response field | No | Add `avatar_url` to user |
| Add endpoint | No | New `GET /api/v1/reports` |
| Remove response field | Yes | Remove `legacy_id` |
| Change field type | Yes | `id` from int to string |
| Change pagination format | Yes | Offset → cursor |

## Migration Strategy

1. Ship v2 alongside v1
2. Add `Deprecation: true` and `Sunset` headers to v1
3. Log v1 usage (track migration progress)
4. Notify consumers with migration guide
5. Remove v1 after sunset date
```

---

## Worked Example

```markdown
# ADR-001: Use PostgreSQL as Primary Database

**Status:** Accepted
**Date:** 2026-01-15
**Deciders:** Backend team (3 engineers)

## Context

Building an e-commerce API for a B2B marketplace. Expected load: ~5K RPM initially,
growing to ~50K RPM in 12 months. Data includes products, orders, users, and inventory.
Team has strong SQL experience. Need full-text search for products and transactional
consistency for orders.

## Decision

Use PostgreSQL 16 as the sole database. Use built-in tsvector for full-text search.
Add Redis later only if measured latency exceeds targets.

## Alternatives Considered

### MongoDB
- Pros: Flexible product schemas, horizontal scaling
- Cons: Weaker transactions (multi-document), team has no MongoDB experience,
  would need separate search solution (Elasticsearch)
- Why rejected: Transaction safety for orders is critical. Product schema is
  actually quite structured. Team learning curve adds risk to timeline.

### PostgreSQL + Elasticsearch
- Pros: Best-in-class search
- Cons: Two systems to maintain, data sync complexity, operational overhead
- Why rejected: PostgreSQL tsvector + GIN index handles our search requirements.
  If we outgrow it, ParadeDB is a drop-in upgrade without a separate system.

## Consequences

### Positive
- Single database to operate, monitor, and backup
- ACID transactions for order processing
- Team can be productive immediately (familiar technology)
- Extension path: pgvector for recommendations, ParadeDB for advanced search

### Negative
- Horizontal scaling is harder than MongoDB (but we won't need it at 50K RPM)
- Schema changes require migrations (mitigated by gin-psql-dba migration safety guide)

### Risks
- If product schemas become truly unpredictable → mitigated by JSONB columns
- If full-text search outgrows tsvector → mitigated by ParadeDB upgrade path

## Follow-up Actions
- [x] Set up PostgreSQL 16 with Docker (gin-deploy)
- [x] Define initial schema (gin-psql-dba)
- [ ] Configure connection pooling (gin-psql-dba)
- [ ] Set up backup strategy (gin-psql-dba)
```

---

## ADR Index

Keep a simple index file at `docs/adr/README.md`:

```markdown
# Architecture Decision Records

| # | Title | Status | Date |
|---|---|---|---|
| 001 | [Use PostgreSQL as Primary Database](001-use-postgresql.md) | Accepted | 2026-01-15 |
| 002 | [JWT with Refresh Tokens for Auth](002-jwt-auth.md) | Accepted | 2026-01-20 |
| 003 | [URL Path API Versioning](003-api-versioning.md) | Accepted | 2026-02-01 |
```

**Naming convention:** `NNN-short-slug.md` (e.g., `001-use-postgresql.md`)
