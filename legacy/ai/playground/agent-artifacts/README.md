---
title: Playground agent artifacts
created: 2026-02-12
updated: 2026-02-12
---

# Playground agent artifacts

This folder contains **simulation-only** agent artifacts for the AI architecture virtual development. Use these when doing AI chat simulations in `docs/ai/playground/`. The main `docs/agent-artifacts/core/` is **not** updated during simulation.

## Ralph Wiggum loop (simulation)

1. **Before simulation**: Read [STATUS.md](./STATUS.md) and [../AGENTS.md](../AGENTS.md).
2. **Pick slice**: Choose a task from [task-registry.md](./task-registry.md) with `status: open`.
3. **Implement**: Document, update session logs, create or update fix guides. **No code edits** to `apps/` or `packages/` unless explicitly migrating.
4. **After**: Update [STATUS.md](./STATUS.md) Done list; set task status in task-registry.

## Isolation

- **Simulation** = docs only under `docs/ai/playground/`. Session logs, fix guides, and agent-artifacts here.
- **Do not** update `docs/agent-artifacts/core/STATUS.md` or other main artifacts during simulation.
- **When migrating**: When implementing for real (code changes to apps/packages), switch to the main loop: read main STATUS, update main artifacts.

## Compacting

When STATUS Done exceeds ~15 items, summarize older items and move to `archive/STATUS-done-YYYY-MM.md`. Keep the most recent 8–10 in STATUS.

## Index

| File | Purpose |
|------|---------|
| [STATUS.md](./STATUS.md) | Simulation state, Ralph Wiggum Done list, Next |
| [task-registry.md](./task-registry.md) | Simulation tasks (document Plan fix, forge context, etc.) |
| [errors-and-attempts.md](./errors-and-attempts.md) | Simulation failures and fixes |
