---
title: Compacting and archiving agent artifacts
created: 2026-02-06
updated: 2026-02-06
---

Living artifact for agents. Index: [18-agent-artifacts-index.mdx](../../18-agent-artifacts-index.mdx).

# Compacting and archiving

> **For coding agents and maintainers.** We compact long artifact sections and move detail to an archive so context windows stay effective. Cursor and Codex summarize for their context; we do the same by keeping main artifacts focused on current state and recent items.

## When to compact

**Plan docs:** Do not add new plan documents to core. Plans are ephemeral (Cursor/agent run). Legacy plan files belong in [../archive/](../archive/) (see archive README). Main artifacts keep a one-line pointer when a plan was archived.

Compact when a subsection grows beyond a defined threshold:

- **STATUS "Ralph Wiggum" Done list:** When the list exceeds roughly the last **20–30** items, summarize older items and move the full list to an archive file. Keep the most recent 10–15 Done items in STATUS.
- **errors-and-attempts:** When entries exceed roughly **25–30**, consider moving the oldest (or obsolete) entries to an archive and keep a one-line summary or pointer.
- **decisions:** When the doc is very long and some entries are obsolete or superseded, move obsolete entries to an archive and keep a short "superseded by" note.

Thresholds are guidelines; use judgment. The goal is to keep the main file scannable and relevant.

## Archive location and naming

- **Folder:** `docs/agent-artifacts/archive/`
- **Naming:** One file per compacted chunk or time window, e.g.:
  - `STATUS-done-2026-02.md` — compacted Done list for that month
  - `errors-and-attempts-2026-01.md` — older errors-and-attempts entries

## How to compact

1. **Summarize** the subsection in the main doc (e.g. "Done (2026-02-04-02-05): Monorepo reorg, Payload, plans, Video skeleton, marketing site, workflow runtime, WorkspaceApp, consumer example. See [archive/STATUS-done-2026-02.md](../archive/STATUS-done-2026-02.md) for full list.").
2. **Move** the full content to a new file in `docs/agent-artifacts/archive/` with the chosen name.
3. **Keep** a one-line pointer in the living doc. Do not duplicate the full content.
4. **Agents:** Read the main artifact first; only open archive when tracing history.

## Cross-reference from main doc

In STATUS (or errors-and-attempts, decisions), keep a single line that points to the archive file. Example:

```markdown
- Done (2026-02-04–02-05): Monorepo reorg, Payload, plans, Video skeleton, … See [archive/STATUS-done-2026-02.md](../archive/STATUS-done-2026-02.md).
```
