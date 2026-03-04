---
title: Plan - Editor layout main content not showing (archived)
created: 2026-02-07
updated: 2026-02-08
archived: 2026-02-09
---

**Archived.** Plans are not living docs in core. Resolved: Dockview + resetLayout(). See [errors-and-attempts](../core/errors-and-attempts.md).

---

# Plan: Editor layout - main content not showing

**Created:** 2026-02-07  
**Status:** Resolved (2026-02-08)  
**Strategy:** [19-coding-agent-strategy.mdx](../../19-coding-agent-strategy.mdx); [errors-and-attempts.md](./errors-and-attempts.md)

**Update (2026-02-08):** DockLayout is back on **Dockview** with `resetLayout()` recovery. This plan remains a historical record of the FlexLayout incident and the temporary resizable-panels fallback.

---

## 1. Problem summary

- **Symptom:** Only overlay UI (e.g. Copilot Kit sidebar/chat) is visible. No editor panels (Library, Main, Inspector, Workbench) and no main app content. Users cannot see or interact with Dialogue/Video/Character/Strategy editors.
- **DevTools observation:** The first child of `<body>` is a `<div hidden>` that contains only HTML comments (`<!--$-->` and `<!--/$-->`). That suggests either:
  - The main app content is mounted inside this div and is being hidden (framework or our code), or
  - React/Next streaming has left a placeholder div in place and the real content never replaced it (e.g. suspend/error during client render).
- **Possible causes (unconfirmed):**
  - **CSS:** FlexLayout theme or our overrides (e.g. `flexlayout__theme_dark`) applying `display: none` or `visibility: hidden` to the layout or its container.
  - **Hydration/streaming:** Next 15 / React 19 wrapping the app in a hidden shell until client content streams in; content never "arrives" due to an error or infinite suspend in the editor path.
  - **DockLayout/FlexLayout:** Model or factory not running on the client, or Layout component not rendering (we already tried fixing SSR model via `useEffect`).
  - **Third-party:** CopilotKit or TwickProviders adding a wrapper that hides the rest of the tree (less likely if Copilot UI is visible).

---

## 2. What we have tried

| Attempt | Description | Outcome |
|--------|-------------|---------|
| **Dockview -> FlexLayout** | Replaced Dockview with `flexlayout-react` in `DockLayout.tsx` to fix Twick context (panels outside React tree) and lost panels. | Panels still not visible; Copilot Kit UI is. |
| **CSS resolve** | `flexlayout-react/style/dark.css` failed to resolve from `apps/studio` because the package was only a dependency of `@forge/shared`. | Added `flexlayout-react` to `apps/studio/package.json` so the CSS import resolves. Dev server starts; layout content still not showing. |
| **SSR model** | Layout model was built in `useState(initializer)`. On SSR, `window` is undefined so initializer returned `null`; React reused that state on the client, so `model` stayed `null` and Layout never rendered. | Switched to `useState<Model | null>(null)` and set the model in a `useEffect` that runs on the client (read from `localStorage` or default JSON, then `setModel(...)`). Documented in errors-and-attempts. Panels still not showing. |
| **LocalStorage error** | Terminal shows `Failed to load undo-redo state from localStorage: [ReferenceError: localStorage is not defined]`. | Indicates some code (e.g. Twick or another lib) runs in a context without `window`; not necessarily the root cause of hidden content but worth guarding. |
| **Resizable panels fallback** | Replaced FlexLayout DockLayout with `react-resizable-panels` (static splits, no docking). | **Resolved** (temporary). Panels render inside the React tree; editor content is visible again. |
| **Dockview restored** | Reintroduced Dockview with slot tabs + `resetLayout()` recovery. | **Resolved (current)**. Docking + floating panels work; content is visible. |

---

## 3. Tech we have used

- **Dockview (original):** Previously powered `DockLayout`. Caused panels to render outside the main React tree (Twick context lost), `containerApi` on DOM, and "lost panels" with no in-app recovery.
- **Resizable panels (temporary fallback):** `react-resizable-panels` via `@forge/ui/resizable`. Used in `DockLayout.tsx` with outer/inner split groups and `resetLayout()` to clear persisted sizes. This was the stopgap while FlexLayout was removed.
- **Dockview (current):** Restored Dockview DockLayout with slot tabs (containerApi filtered) and `resetLayout()` recovery.

---

## 4. Tech we are (current stack)

- **Runtime:** Next.js 15.x, React 19, Turbopack.
- **Layout:** Dockview in shared `DockLayout` with `dockview/dist/styles/dockview.css` and `dockview-overrides.css`.
- **UI:** CopilotKit (visible), Twick (Video), Radix/shadcn via `@forge/ui`, Tailwind v4.
- **Persistence:** Layout state in localStorage; app-shell and editor state in Zustand with optional persist.

---

## 5. New tech candidate if we cannot fix this quickly

If we cannot get FlexLayout (or the current shell) to show main content within a short timebox:

- **Option A - react-resizable-panels (already in tree):** We already depend on `react-resizable-panels` in `packages/ui` and `apps/studio` (used by `ResizablePanelGroup`, `ResizablePanel`, `ResizableHandle` in `packages/ui/src/components/ui/resizable.tsx`). Replace `DockLayout` with a **static panel layout** (no drag-tabs/dock/float): a single `PanelGroup` with left / main / right / bottom panels and resizable splitters. No persistence of layout structure; simple and predictable. Good fallback to "something that works."
- **Option B - CSS Grid / Flex fallback:** Replace `DockLayout` with a minimal layout: a wrapper with `display: grid` or `flex` and fixed slots (e.g. `grid-template-columns: 1fr 3fr 1fr` for left/main/right). No tabs, no drag, no persist. Fastest to implement if we only need "panels on screen."
- **Option C - Golden Layout or similar:** Evaluate a different docking library (e.g. Golden Layout 2) only if we must keep drag/dock/float and FlexLayout cannot be fixed. Higher migration cost.

**Recommendation (historical):** Option A (react-resizable-panels) was implemented as a temporary fix. We have since restored Dockview.

---

## 6. Next steps (diagnosis before switching tech)

**Resolved:** DockLayout now uses Dockview again; main editor content is visible. Remaining steps are no longer required for this incident.

1. **Confirm where `hidden` comes from**
   - In the browser, inspect the `<div hidden>`: which component or script adds it? (React root, Next wrapper, CopilotKit, etc.)
   - Search the repo (and `node_modules` if needed) for `hidden` or `aria-hidden` on a root-level wrapper.

2. **Rule out FlexLayout CSS**
   - Temporarily comment out the FlexLayout CSS import and the `flexlayout__theme_dark` class; render a simple placeholder (e.g. "Layout placeholder") inside the same slot wrapper. If the placeholder appears, the issue is likely FlexLayout CSS or structure.

3. **Rule out React/Next streaming**
   - Check for uncaught errors or suspended boundaries in the editor path (DialogueEditor -> WorkspaceShell -> DockLayout). Add a minimal client-only fallback (e.g. a div with "Editor content" before DockLayout) to see if any content under `WorkspaceApp.Content` appears.

4. **Guard localStorage in app**
   - Ensure any code that reads `localStorage` (undo-redo, layout, etc.) is behind `typeof window !== 'undefined'` or runs only in `useEffect` so we do not throw during SSR.

5. **If diagnosis points to FlexLayout**
   - Try rendering FlexLayout in isolation (e.g. a spike route with only FlexLayout + static slot content, no CopilotKit/Twick) to see if panels show. If they do, the bug is in the integration (providers, shell, or CSS scope).

6. **If we timebox and switch**
   - Implement Option A (react-resizable-panels layout) behind a feature flag or as a new `DockLayoutSimple` and switch editors to it; document in STATUS and errors-and-attempts; leave FlexLayout path in code but unused until we can revisit.

---

## 7. References

- [errors-and-attempts.md](./errors-and-attempts.md): FlexLayout panels not showing (resolved; Dockview restored).
- [05-editor-platform.mdx](../../architecture/05-editor-platform.mdx): WorkspaceApp, DockLayout, WorkspaceShell (Dockview).
- [DockLayout.tsx](../../../packages/shared/src/shared/components/editor/DockLayout.tsx): Current Dockview implementation.
- [18-agent-artifacts-index.mdx](../../18-agent-artifacts-index.mdx): Index of agent artifacts.
