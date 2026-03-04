---
title: Technical debt roadmap
created: 2026-02-08
updated: 2026-02-09
---

Living artifact for agents. Index: [18-agent-artifacts-index.mdx](../../18-agent-artifacts-index.mdx).

# Technical debt roadmap

For **"what's next for technical debt,"** pick an open item below. When done, add a Ralph Wiggum Done line in [STATUS.md](./STATUS.md) and set the item to `done` here.

**Process:** Add items from [errors-and-attempts.md](./errors-and-attempts.md) follow-ups, [decisions.md](./decisions.md) ("Tech debt: …"), production checklist, or when someone identifies debt. Pre-production items are for before (or right after) going to production; Ongoing is refactors and cleanup when we have capacity. When you start an item, set status to `in_progress` in this file. When sections grow, compact per [compacting-and-archiving.md](./compacting-and-archiving.md).

**Naming:** In this roadmap and editor docs, "workspace" means the legacy shared workspace UI/types (e.g. `shared/components/workspace`, `shared/workspace` types), not pnpm workspaces.

**Suggested order (passes):** Pass 1 — td-1 to td-5 (consolidation + legacy). Pass 2 — td-6 to td-9 (tokens + accents). Pass 3 — td-10 to td-12 (editor color context).

---

## Pre-production

Items we want done before (or immediately after) going to production.

| id | title | impact | status | note |
|----|-------|--------|--------|------|
| *(None yet)* | — | — | — | Add items as we approach production. |

---

## Ongoing

Refactors, cleanup, and tech debt we address when we have capacity.

| id | title | impact | status | note |
|----|-------|--------|--------|------|
| td-1 | Clarify or remove workspace from shared public API | Small | done | Index + AGENTS + editor README |
| td-2 | Fix WorkspaceShell JSDoc (data-mode-id) | Small | done | Removed data-mode-id from JSDoc and README |
| td-3 | Single home for editor/shared types | Small | done | Verified: types only in workspace; note in editor README |
| td-4 | Execute legacy removal plan (viewport, app-shell, settings, model-router) | Medium | done | ViewportMeta viewport-only attrs; openEditor editorId only; modes re-export removed |
| td-5 | Remove or isolate deprecated components (WorkspaceEditor, GraphEditor prop, etc.) | Small–Medium | done | ViewportMeta is canonical; redundant modes/ removed; no deprecated props |
| td-6 | Editor chrome token audit (replace ad-hoc px/py/gap/sizing) | Medium | done | Shared editor chrome now uses --control-*, --panel-padding |
| td-7 | Studio UI token audit | Small–Medium | done | apps/studio components use --control-*, --panel-padding |
| td-8 | Panel and tab accent pass (context-accent everywhere needed) | Medium | done | DockPanel header accent; PanelTabs/Dockview/WorkspaceTab already had it |
| td-9 | Section/list accent audit (SectionHeader, NodePalette, etc.) | Small | done | SectionHeader + NodePalette use --context-accent |
| td-10 | Document editor color context (domain, tokens) | Small | done | Editor README + 01-styling-and-theming.mdx |
| td-11 | Component-level context override (optional domain/context on sections) | Medium | done | SectionHeader context prop; doc in README + 01-styling-and-theming |
| td-12 | Apply context tokens to all editor primitives | Medium | done | Chrome edges use --context-accent; GraphSidebar tab bar |
| td-13 | Knip for dead-code detection | Small | done | Root knip script + config; documented in tool-usage and strategy |
| td-14 | Triage Knip "unused files" (Studio + packages) | Small | done | ignoreFiles extended; removed graph-to-sequence.ts (rg-confirmed); documented in knip-findings |
| td-15 | Triage Knip "unused dependencies" | Small | done | ignoreDependencies for Radix/ui, Twick, fumadocs, devDeps; none removed; documented |
| td-16 | Root-level dead code (app/, lib/, types/) | Small | done | Root app/lib/types in ignoreFiles; documented as legacy in knip-findings; no deletion |
| td-17 | Knip "unused exports" (barrels and runtime) | Medium | done | ignoreIssues for **/index.ts, payload-types, collections, source.config; documented |
| td-18 | Structured logging and log file | Medium | done | pino in Studio; LOG_LEVEL, LOG_FILE, namespaces; optional client-to-file in dev; see standard-practices |
