---
title: Agent artifacts archive
created: 2026-02-11
updated: 2026-02-11
---

# Agent artifacts archive

This folder holds **compacted chunks** of living agent artifacts and **legacy plan docs** (no longer in core).

- **Compacted chunks:** When a section (e.g. STATUS "Ralph Wiggum" Done list, or old errors-and-attempts entries) grows too long, we summarize it in the main doc and move the full content here. Naming: e.g. `STATUS-done-2026-02.md`, `errors-and-attempts-2026-01.md`.
- **Legacy plan docs:** Plan documents are **not** kept as living artifacts in core. Plans are ephemeral (Cursor or other agent planning systems). If a plan was previously in core, it has been moved here (e.g. `plan-dockview-restore-*`, `plan-editor-layout-*`). Outcomes live in ISSUES, STATUS, decisions, errors-and-attempts, or task-registry/roadmap. Do not add new plan docs to core.
- **Usage:** Main artifacts in `../core/` keep a one-line pointer to the archive file. Agents read the main artifact first; open archive only when tracing history.

See [../core/compacting-and-archiving.md](../core/compacting-and-archiving.md) for the full process.
