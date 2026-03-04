---
title: Task breakdown system
created: 2026-02-08
updated: 2026-02-08
---

Living artifact for agents. Index: [18-agent-artifacts-index.mdx](../../18-agent-artifacts-index.mdx).

# Task breakdown system

> **For coding agents and maintainers.** We break epics and large work into **tiers** (Initiative → Lane → Slice → Task) so agents and humans can pick **small, pickable tasks** and see **parent reference** for context.

## Purpose

- Break Epic/Large initiatives into smaller units so "what's next" can be answered at both **high level** (STATUS § Next) and **granular level** (pickable tasks).
- Keep **parent reference** so every task links back to its initiative; agents know "what am I working toward?"
- Enable **easy small tasks** (Tier 3) for single-PR or quick-win work.

## Tier definitions

| Tier | Name       | Meaning                                                                 | Example                                                             |
|------|------------|-------------------------------------------------------------------------|---------------------------------------------------------------------|
| **0** | Initiative | Epic or major initiative (STATUS Next items that are Epic or Large)     | Platform: monetization (clone/download)                             |
| **1** | Lane       | Theme or capability area within an initiative                           | Listings, Checkout, Clone flow, Payouts                             |
| **2** | Slice      | One clear deliverable (~Medium impact; API + UI or data)                | Listings API: create/update/delete listing                          |
| **3** | Task       | Small unit (Small impact; one PR, few files)                            | Add `CAPABILITIES.PLATFORM_LIST` constant and gate listing button   |

**Pickable units:** Tier 2 (slices) and Tier 3 (tasks). Tier 3 = "easy small tasks" for agents.

## Per-item fields

Use these in breakdown docs and the task registry (machine- and human-readable):

- **id** — Short stable id (e.g. `platform-mono-1`, `yarn-export-2`).
- **title** — One line.
- **parent** — Id of parent (or `—` for Tier 0).
- **tier** — 0 | 1 | 2 | 3.
- **impact** — Small | Medium | Large | Epic (align with [STATUS](./STATUS.md) impact sizes).
- **status** — `open` | `in_progress` | `done`.
- **doc** — Optional link to architecture/roadmap/decisions.

**Rule:** Every Tier 0 (initiative) with impact Epic or Large **must** have a breakdown doc. Tier 2/3 items are the pickable units.

## When to create a breakdown

When [STATUS § Next](./STATUS.md) has an item with impact **Epic** or **Large**:

1. Add a per-initiative breakdown file: `task-breakdown-<slug>.md` (e.g. `task-breakdown-platform-monetization.md`).
2. Register the initiative in [task-registry.md](./task-registry.md) with a link to the breakdown.
3. In the breakdown doc, define Lanes (Tier 1), Slices (Tier 2), and Tasks (Tier 3) as needed.

## Format and location

- **Task registry:** [task-registry.md](./task-registry.md) — index of all Tier 0 initiatives and optional "Quick picks" (Tier 3 open Small tasks).
- **Per-initiative breakdowns:** `task-breakdown-<slug>.md` in this folder. Each file contains: initiative title and id; link back to STATUS Next and roadmap; tiers 1–3 as nested list or table with id, title, parent, tier, impact, status, optional doc.

## Workflow

1. **Pick work:** From [STATUS § Next](./STATUS.md) (high level) or [task-registry.md](./task-registry.md) (granular). To pull an easy small task, pick a Tier 3 (or Tier 2) item with `status: open`. When you start a task, set its status to `in_progress` in this breakdown and add "In progress: …" in [STATUS](./STATUS.md) (Ralph Wiggum section).
2. **Implement:** Open the initiative’s breakdown doc for context and parent reference. Set the item’s status to `in_progress` in the breakdown when you begin; when done, set to `done`.
3. **Update STATUS:** When a slice or task completes, add a Ralph Wiggum Done line in [STATUS.md](./STATUS.md); optionally include the task id for traceability.

## Greppability

Agents can find pickable small tasks by searching:

- `rg "tier: 3"` or `rg "tier: 2"` in this folder for slices/tasks.
- `rg "status: open"` for work that is not started.
- Combine with impact (Small/Medium) in the same file or registry.
