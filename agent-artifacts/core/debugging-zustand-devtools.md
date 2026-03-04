---
title: Debugging Zustand with Redux DevTools
created: 2026-02-05
updated: 2026-02-05
---

Living artifact for agents. Index: [18-agent-artifacts-index.mdx](../../18-agent-artifacts-index.mdx).

# Debugging Zustand stores with Redux DevTools

Use Redux DevTools to inspect and time-travel Zustand store state when debugging state-related bugs (e.g. infinite loops, wrong UI, unexpected updates).

## Setup

1. Install the **Redux DevTools** browser extension (Chrome, Firefox, Edge).
2. Run the Studio app in **development** (`pnpm dev`). Our Zustand stores are connected via the `devtools` middleware and appear in the DevTools store dropdown.

## Store names (Redux DevTools dropdown)

- **AppShell** - `apps/studio/lib/app-shell/store.ts` (route, open editors, themes, bottom drawer).
- **Settings** — `apps/studio/lib/settings/store.ts` (app/editor/viewport settings).
- **ModelRouter** — `apps/studio/lib/model-router/store.ts` (mode, manual model, enabled models, health).
- **Entitlements** — `apps/studio/lib/entitlements/store.ts` (plan, overrides).
- **Video** — `apps/studio/lib/domains/video/store.ts` (video doc draft, isDirty).
- **Graph** — `apps/studio/lib/store.ts` (graph draft, isDirty, pendingFromPlan).

## How to use (humans)

1. Reproduce the issue (e.g. open app, switch editor, trigger the bug).
2. Open browser DevTools → **Redux** tab.
3. Select a store from the dropdown (or "All" to see combined state).
4. Inspect **State** and **Action** list; use **time-travel** (slider or prev/next) to step back and see what changed.
5. When reporting: include the **store name**, last **actions** that led to the bug, and if possible an **exported state** or screenshot.

## For agents

State and actions from the Redux DevTools extension are **not** easily machine-readable. When debugging state/loop issues:

1. **Check [errors-and-attempts.md](errors-and-attempts.md)** for known errors and fixes (e.g. getSnapshot cached, max update depth with WorkspaceTab).
2. **Suggest human steps**: Ask the user to reproduce the issue, open Redux DevTools, note which **store(s)** and **actions** lead to the bug, and share that (screenshots, exported state, or a short description). With that, the agent can target fixes (e.g. selector stability, ref usage, action ordering).
