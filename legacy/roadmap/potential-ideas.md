---
title: Potential ideas
created: 2026-02-09
updated: 2026-02-11
---

# Potential ideas

Ideas under consideration; **not committed**. We may adopt, defer, or drop them. When an idea is **accepted**, add it to [STATUS § Next](../agent-artifacts/core/STATUS.md), [task registry](../agent-artifacts/core/task-registry.md), [product roadmap](product.mdx), or [enhanced-features backlog](../agent-artifacts/core/enhanced-features-backlog.md) with impact; set status here to `accepted` and link.

---

## Process

1. **Anyone** adds an idea with **status**: `exploratory` | `under_review` | `accepted` | `deferred` | `rejected`.
2. **Exploratory** — Just captured; no decision yet.
3. **Under review** — Being evaluated.
4. **Accepted** — We commit; move to STATUS, task-registry, roadmap, or enhanced-features backlog; set status here to `accepted` with link.
5. **Deferred** — Not now; revisit later.
6. **Rejected** — We gave it up (add short reason in Note).

**What's next for ideas:** Link from [roadmap index](00-roadmap-index.mdx) and [18-agent-artifacts-index](../18-agent-artifacts-index.mdx); when asking "what's next" for business/strategy, check here to triage or add ideas.

---

## Ideas (table)

| id | title | context | suggestion | status | date | note |
|----|-------|---------|------------|--------|------|------|
| creator-dashboard | Creator dashboard (platform) | platform | Full account management in the platform app (my listings, my games, licenses, revenue) using a shadcn-based template; Studio minimal (publish/update in app bar only). | accepted | 2026-02-09 | [Task registry initiative creator-dashboard](../agent-artifacts/core/task-registry.md); [breakdown](../agent-artifacts/core/task-breakdown-creator-dashboard.md). |
| shadcn-dashboard-template | Shadcn dashboard template for platform | platform | Evaluate Kiranism vs Vercel Studio Admin (or similar) for platform account/dashboard area; adopt one and document migration steps (auth, layout, tables). | accepted | 2026-02-09 | Implemented in `apps/platform`; see STATUS entries 2026-02-10 and 2026-02-11 plus creator-dashboard breakdown. |
| ai-assisted-import | AI-assisted import for external formats | import | Import pipeline: user uploads or pastes Yarn/Ink/Twine/Ren'Py content; AI step helps parse and map into our project/graph schema. Difficulty and scope TBD. | exploratory | 2026-02-09 | May be dropped; format differences per adapter are non-trivial. |

---

*(Add new ideas at the top of the table; append-only.)*
