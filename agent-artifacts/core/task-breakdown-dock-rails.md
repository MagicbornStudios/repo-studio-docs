---
title: Task breakdown — Dock rails and composable panels
created: 2026-02-10
updated: 2026-02-10
---

Living artifact for agents. Index: [18-agent-artifacts-index.mdx](../../18-agent-artifacts-index.mdx).

# Dock rails and composable panels (initiative: dock-rails)

**High priority** for editor convention and naming. This breakdown captures the agreed design so agents/humans have one place to read "what we decided" for rails.

## Design context (do not lose)

- **Rails** = left, main, right (and optionally bottom). Each rail is the same concept: a **list of panel tabs** (Dockview-level tabs on that side).
- **Panel tab** = one tab in that rail's tab bar. **Content** of each tab = one "panel" — in practice an **EditorDockPanel** (or equivalent) that can be simple or composed (e.g. PanelTabs inside for inner tabs like Library's Graphs | Nodes).
- **Config-driven**: Adding a tab = adding an entry to the list for that rail, not a new prop or constant. Single API shape: **RailPanelDescriptor** — `{ id: string; title: string; iconKey?: ...; content: React.ReactNode }`. DockLayout accepts `leftPanels?`, `mainPanels?`, `rightPanels?`, `bottomPanels?` (arrays). Backward compat: keep `left`/`main`/`right` as shortcuts when the array has one element.
- **Main** today: one Dockview panel with custom content (e.g. Dialogue stacks two ForgeGraphPanels in a div). Target: main = **multiple panel tabs** (e.g. Narrative | Storylet), each tab content = one EditorDockPanel wrapping one graph. Same pattern as right (Inspector | Settings | …).
- **Right** today: either one `right` or two hardcoded `rightInspector` + `rightSettings`. Target: right = **list of panels**; adding "Foo" = one more descriptor in `rightPanels`.
- **Composition**: EditorDockPanel can be leaf (children), tabbed (`tabs` → PanelTabs), or (future) nested "sub-rail." Document: rail → panel tabs → each tab content = EditorDockPanel → which can wrap simple content, PanelTabs, or nested panels. Same primitives (EditorDockPanel, PanelTabs) everywhere; naming and docs should reflect "rails" and "composable panels."
- **Naming / refactors**: Slot ids as opaque strings for right rail instead of extending a union per new panel; "rail" terminology in shared README and AGENTS; DockLayout props and types aligned to `*Panels` arrays. Update [packages/shared/.../editor/README.md](../../packages/shared/src/shared/components/workspace/README.md) and [AGENTS.md](../../packages/shared/src/shared/AGENTS.md) with the rails + composable panels convention.

## Tiers (pickable tasks)

| id | title | parent | tier | impact | status | doc |
|----|-------|--------|------|--------|--------|-----|
| dock-rails-api-shape | RailPanelDescriptor + leftPanels/mainPanels/rightPanels/bottomPanels exist in EditorDockLayout; wire editors to array API (optional) | dock-rails | 2 | Medium | open | [EditorDockLayout](../../packages/shared/src/shared/components/workspace/EditorDockLayout.tsx) |
| dock-rails-wire-editors | Wire Dialogue/Character/Video to new rail API (mainPanels, rightPanels); deprecate or map old right/rightInspector/rightSettings | dock-rails | 2 | Medium | open | editors in apps/studio/components/editors |
| dock-rails-naming | Align DockLayout, EditorDockPanel, shared editor README/AGENTS with rails and composable panels; document in errors-and-attempts/decisions | dock-rails | 3 | Small | open | README, AGENTS, decisions |

**Implementation order:** dock-rails-api-shape → dock-rails-wire-editors → dock-rails-naming.
