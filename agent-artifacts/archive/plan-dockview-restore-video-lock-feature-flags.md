---
title: Plan — Restore Dockview, lock Video editor, feature flags (archived)
created: 2026-02-07
updated: 2026-02-08
archived: 2026-02-09
---

**Archived.** Plans are not living docs in core. Outcomes: Video locked (ISSUES.md), Dockview restored (STATUS, errors-and-attempts), PostHog for flags. See [ISSUES.md](../../ISSUES.md), [STATUS](../core/STATUS.md).

---

# Plan: Restore Dockview, lock Video editor, feature-flag SDK

**Created:** 2026-02-07  
**Status:** Planning  
**Priorities:** (1) Restore Dockview, (2) Lock Video editor + document as non-functional.

---

## 1. Goals

1. **Restore Dockview** — Replace the current DockLayout (react-resizable-panels) with the **original Dockview-based implementation** so we get drag-to-reorder, floating panels, add/remove at runtime, and drag-tabs-to-group again. Highest priority.
2. **Lock Video editor** — Gate the Video editor behind a feature flag so it is not usable until we choose to enable it. Document it as **non-functional** in ISSUES.md (or equivalent).
3. **Feature-flag SDK (DB/REST)** — Evaluate and plan integration of a **3rd party feature-flag solution** that works with our database, REST API, or similar; use it for Video editor (and optionally other flags) so we can turn features on/off without code deploys.
4. **Agent “what can I do next?”** — Ensure STATUS.md, AGENTS.md, 18-agent-artifacts-index, and ISSUES.md give a coding agent enough context to answer “what can I do next?” (current issues, next steps, known broken areas).

---

## 2. Current state

- **DockLayout:** Implemented with **react-resizable-panels** (ResizablePanelGroup, ResizablePanel, ResizableHandle) and `useDefaultLayout` for persistence. No drag-reorder, no floating panels, no tab grouping. See `packages/shared/src/shared/components/editor/DockLayout.tsx`.
- **Video editor:** Rendered in AppShell when `activeWorkspaceId === 'video'`; no gate. Uses Twick (vendored); persistence and plan/commit UI pending. We are **not using it now**; user wants it locked and documented as non-functional.
- **Feature gating:** `FeatureGate` + `CAPABILITIES` in shared; Studio uses `EntitlementsProvider` backed by `useEntitlementsStore` (Zustand). Status is derived from **plan** (free/pro from `/api/me`) and **overrides** (local overrides only). No 3rd party SDK; no DB-backed flags today (plan comes from Payload user.plan).
- **Docs:** STATUS has “What you can do next” with impact sizes; 18-agent-artifacts-index lists core artifacts; errors-and-attempts logs fail aures. No single **ISSUES.md** for product/editor-level “known broken or limited” items.

---

## 3. What we have tried (layout)

- **Dockview (original):** Panels worked; we had context issues (Twick/useTimelineContext), containerApi on DOM, and lost panels with no in-app recovery. We moved to FlexLayout then to react-resizable-panels.
- **FlexLayout:** Main content/panels did not show (SSR model, then suspected hidden div/streaming). We reverted to react-resizable-panels.
- **react-resizable-panels (current):** Panels show and resize, but no drag, float, or tab grouping. User wants to go back to Dockview.

---

## 4. Restore Dockview (priority 1)

**Scope:** Re-implement DockLayout using the `dockview` library (as in the original design), with fixes applied so we don’t regress known issues.

**References:**
- [.cursor/plans/replace_dockview_docking_library_cf8110ac.plan.md](../../.cursor/plans/replace_dockview_docking_library_cf8110ac.plan.md) — Original Dockview slot API, persistence key `localStorage['dockview-{layoutId}']`, Reset layout, containerApi fix.
- [docs/agent-artifacts/core/errors-and-attempts.md](./errors-and-attempts.md) — Lost panels, Twick context, containerApi.

**Tasks:**

1. **Re-add dockview dependency**  
   - Add `dockview` to `packages/shared` (and optionally `apps/studio` if CSS is imported there). Remove or keep `react-resizable-panels` in DockLayout only if we still use it elsewhere (e.g. `@forge/ui/resizable`).

2. **Re-implement DockLayout with Dockview**  
   - Same public API: `left`, `main`, `right`, `bottom`, `slots`, `viewport`, `layoutId`, optional size/min props.  
   - Use `DockviewReact`, `onReady`, build default layout (left/main/right/bottom), persist to `localStorage['dockview-{layoutId}']`, load on ready.  
   - **Custom tab component:** Do not spread Dockview-only props (e.g. `containerApi`) onto DOM. Destructure and omit them before spreading onto the tab element (see replace_dockview plan Phase 1).  
   - **Reset layout:** Expose a ref or callback (e.g. `resetLayout()`) that clears storage for the current `layoutId` and rebuilds the default layout (or remounts). Document in errors-and-attempts.

3. **Panel context (Twick/Video)**  
   - Video editor is being locked; Twick context in panels is less critical for now. If we use a **panelContentWrapper** again, document that restored layouts may still break context for Video; keep Video behind the new capability so we don’t rely on it.

4. **CSS**  
   - Re-import Dockview theme in `apps/studio/app/globals.css` (e.g. `dockview/dist/styles/dockview.css`) and any overrides we had (`dockview-overrides.css`). Remove FlexLayout/resizable-panels-only layout CSS from globals if no longer needed.

5. **Editors**  
   - Dialogue, Character, Strategy keep using `<DockLayout ... />`. No change to slot content; only the layout engine changes.  
   - Video editor stays in the tree but will be gated (see below).

6. **Docs and artifacts**  
   - STATUS: DockLayout is Dockview-based again; mention Reset layout.  
   - errors-and-attempts: Restore Dockview entry; lost panels → reset procedure; containerApi fix.  
   - architecture/05-editor-platform: DockLayout = Dockview, ref with `resetLayout()`.

---

## 5. Lock Video editor with feature flag (priority 2)

**Scope:** Add a capability for the Video editor and gate the tab and content so it is not usable until the capability is allowed. Document Video editor as non-functional.

**Tasks:**

1. **Add capability**  
   - In `packages/shared/src/shared/entitlements/capabilities.ts` add e.g. `VIDEO_EDITOR: 'studio.video.editor'` (or `VIDEO_STUDIO`).

2. **Entitlements store**  
   - In `apps/studio/lib/entitlements/store.ts`, do **not** add `VIDEO_EDITOR` to `PLAN_CAPABILITIES` for any plan (free/pro). So `getStatus(VIDEO_EDITOR)` is always `'locked'` unless we add an override or a 3rd party source later.

3. **Gate in AppShell**  
   - **Tab:** Hide the Video editor tab when the capability is not allowed (e.g. `entitlements.has(CAPABILITIES.VIDEO_EDITOR)`). If not allowed, filter `openWorkspaceIds` and the tab list so Video does not appear, or show the tab disabled/locked.  
   - **Content:** When `activeWorkspaceId === 'video'`, only render `<VideoEditor />` if `entitlements.has(CAPABILITIES.VIDEO_EDITOR)`; otherwise render a placeholder (e.g. “Video editor is not available”) or redirect to another editor.  
   - Optional: Use `FeatureGate` with `mode="hide"` around the Video tab and the Video content area.

4. **Document as non-functional**  
   - Add **ISSUES.md** at repo root (or under docs) listing known product/editor issues. First entry: **Video editor** — Currently **non-functional and locked**. Not in use; gated by capability `studio.video.editor`. Do not assign work that assumes Video editor is usable until we re-enable it.  
   - Link ISSUES.md from [docs/18-agent-artifacts-index.mdx](../../18-agent-artifacts-index.mdx) and from root AGENTS.md so agents see it when answering “what can I do next?”.  
   - In STATUS “Current” or “What changed”, state that Video editor is locked and documented in ISSUES.md.

---

## 6. Feature-flag SDK (DB / REST) — implemented with PostHog

**Outcome:** We use **PostHog** for feature flags. Video editor visibility is gated by PostHog flag `video-editor-enabled`. Studio and marketing use `posthog-js`; init in `instrumentation-client.ts` when `NEXT_PUBLIC_POSTHOG_KEY` is set. Dev and production can use different PostHog project keys (see `.env.example` and decisions.md). Plan-based entitlements (free/pro) remain for paywall; release toggles use PostHog.

**Historical options (from research):**

| Solution    | DB/REST | Self-hosted | Notes |
|------------|---------|-------------|--------|
| **Unleash** | REST API, optional DB | Yes (Docker) | Popular; SDKs for Node/React; flags via API. |
| **Flagsmith** | REST API, DB | Yes (Docker) | 15+ SDKs; segmentation; A/B. |
| **Flagr** | REST (Swagger), MySQL etc. | Yes | Lightweight; good for simple flags. |
| **Payload + custom** | Our DB (Payload) | Yes | New collection `feature-flags` (or settings-overrides); GET `/api/feature-flags` or include in `/api/me`; no new vendor. |

**Recommendation for “works with our database or REST”:**

- **Short term:** Use **existing Payload + entitlements**. Add a `featureFlags` field (or a `feature-flags` collection) and have `/api/me` (or a dedicated endpoint) return flags; in Studio, merge them into entitlements overrides so `getStatus(VIDEO_EDITOR)` can be `'allowed'` when the flag is on. No new SDK; works with our DB and REST.
- **Medium term:** If we want a dedicated product (targeting, A/B, audit), evaluate **Unleash** or **Flagsmith**: run self-hosted, add a small backend route that fetches flags and returns them to the app, and use their JS SDK or a thin wrapper that writes into our entitlements store. Document choice in decisions.md.

**Plan (one slice):**

1. **Option A — Payload-backed flags**  
   - Add a way to store feature flags (e.g. in `settings-overrides` with scope `app` or a new collection).  
   - Add GET (and optional POST) for flags in our REST API; from Studio, call it (e.g. on load or with `/api/me`) and map flags to capability overrides (e.g. `VIDEO_EDITOR` allowed when `flags.videoEditor === true`).  
   - Entitlements store: merge API flags into `overrides` (or a separate `remoteFlags` that `getStatus` respects).  
   - Document in decisions.md and STATUS.

2. **Option B — 3rd party SDK**  
   - Pick one (e.g. Unleash or Flagsmith); run it self-hosted or use their cloud.  
   - Add a server route (e.g. `/api/feature-flags`) that uses the SDK and returns flags for the current user/app.  
   - In the client, fetch flags and feed them into the entitlements store (same as Option A).  
   - Document in decisions.md and STATUS.

Either option allows turning Video editor on later without a code change (flip flag in DB or in the 3rd party dashboard).

---

## 7. Agent “what can I do next?”

**Goal:** A user with a coding agent can ask “what can I do next?” and get accurate, actionable answers.

**Current:** STATUS has a “What you can do next” section with impact-sized items; 18-agent-artifacts-index lists core artifacts; AGENTS.md points to STATUS and the index.

**Add:**

1. **ISSUES.md** at repo root (or `docs/ISSUES.md`) with:
   - **Video editor** — Non-functional and locked. Gated by capability `studio.video.editor`. See plan: [plan-dockview-restore-video-lock-feature-flags.md](./plan-dockview-restore-video-lock-feature-flags.md). Do not assign work that depends on Video editor being usable until re-enabled.
   - (Optional) Other high-level known issues (e.g. layout recovery, platform publish not implemented).

2. **Index and AGENTS**  
   - In [18-agent-artifacts-index.mdx](../../18-agent-artifacts-index.mdx), add a bullet: **ISSUES.md** — Known product/editor issues (e.g. Video editor non-functional and locked). Check before suggesting work that assumes a feature is available.  
   - In root [AGENTS.md](../../../AGENTS.md), add one line under “Agent artifact index” or “Loop”: For **known product/editor issues** (e.g. what’s broken or locked), see **ISSUES.md** and STATUS “What you can do next.”

3. **STATUS**  
   - Under “What you can do next”, add a top item (or note): **Restore Dockview and lock Video editor** — See [plan-dockview-restore-video-lock-feature-flags.md](./plan-dockview-restore-video-lock-feature-flags.md). **Dockview restore is highest priority;** then Video editor lock + ISSUES.md; then optional feature-flag SDK.  
   - Ensure “What changed” states that Video editor is locked and documented in ISSUES.md once the lock is implemented.

---

## 8. Implementation order

1. **Restore Dockview** — Dependencies, DockLayout re-implementation, Reset layout, containerApi fix, CSS, docs.  
2. **Lock Video editor** — Capability, store, gate in AppShell (tab + content), ISSUES.md created and linked.  
3. **ISSUES.md and agent docs** — ISSUES.md content, index and AGENTS.md updates, STATUS “What you can do next” and “What changed.”  
4. **Feature-flag SDK (optional slice)** — Choose Payload-backed vs 3rd party; implement one; wire VIDEO_EDITOR (or a generic flag) so we can enable Video later without deploy.

---

## 9. References

- [.cursor/plans/replace_dockview_docking_library_cf8110ac.plan.md](../../.cursor/plans/replace_dockview_docking_library_cf8110ac.plan.md)
- [errors-and-attempts.md](./errors-and-attempts.md)
- [STATUS.md](./STATUS.md) — “What you can do next”
- [18-agent-artifacts-index.mdx](../../18-agent-artifacts-index.mdx)
- [packages/shared/.../editor/DockLayout.tsx](../../../packages/shared/src/shared/components/editor/DockLayout.tsx) (current resizable-panels)
- [apps/studio/lib/entitlements/store.ts](../../../apps/studio/lib/entitlements/store.ts)
- [packages/shared/.../entitlements/capabilities.ts](../../../packages/shared/src/shared/entitlements/capabilities.ts)
