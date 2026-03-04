---
title: Styling and UI consistency
created: 2026-02-07
updated: 2026-02-11
---

Living artifact for agents. Index: [18-agent-artifacts-index.mdx](../../18-agent-artifacts-index.mdx).

# Styling and UI consistency

> **For coding agents and humans.** Single place for how we handle styling and UI consistency and what to do when debugging or changing UI.

## Purpose

- Ensure toolbar, menu, panel, and card styling stays consistent and readable (contrast, tokens, icons).
- Define the process for styling passes: implement, update docs/STATUS; before/after screenshots are optional (humans capture—browser screenshot automation errors too often).

## Rules

1. **Design tokens and shared components** — Use tokens from [design/01-styling-and-theming.mdx](../../design/01-styling-and-theming.mdx) and components from [02-components.mdx](../../design/02-components.mdx). No ad-hoc `px-*`/`py-*` in editor chrome; use `--panel-padding`, `--control-gap`, `--tab-height`, etc.
2. **Button and menu variants** — Prefer `outline` or `default` for toolbar and menu triggers and primary actions. Avoid ghost-only where it causes poor contrast (dark text on dark background). See [errors-and-attempts.md](./errors-and-attempts.md) entry "Grey buttons and missing menu/icons."
3. **Icons** — Follow the icon audit in [02-components.mdx](../../design/02-components.mdx): menu items (File/View/State), editor creation buttons, Workbench, panel headers, tab triggers. Use Lucide React; standard size **12px** (`size-3` or `var(--icon-size)`), with `--icon-size-lg` (14px) only for emphasis. **No custom SVG icons** — use Lucide React only (e.g. `Loader2` for spinners; `X` for close). Do not add inline `<svg>` or custom SVG components for UI icons. **Settings:** Top-level and inner settings tabs (App, User, AI, Appearance, Panels, Other, Project, Editor, Viewport) and section headers (cards) include a Lucide icon at `--icon-size`. Field labels (e.g. Theme, Density, minimap) and the Reset button include an icon when a mapping is provided; use the same Lucide `--icon-size`. **Editor tab bar:** Tab labels inherit color from the tab state (no forced `text-foreground` on the label); tab-bar icons are constrained to `--icon-size` via the WorkspaceTab container (`[&_svg]:w-[var(--icon-size)] [&_svg]:h-[var(--icon-size)]`). **Input icons:** leading icons in inputs/search bars must use `--icon-size`, and input padding must include the icon width (e.g. `pl-[calc(var(--control-padding-x)+var(--icon-size)+var(--control-gap))]`) to avoid overlap.
4. **All buttons use shadcn Button** — Every interactive button uses `Button` from `@forge/ui/button`. Close/dismiss actions use Lucide `X` inside `Button variant="ghost" size="icon"`. No plain `<button>` with text "x" or `<span>x</span>`.
5. **Shadcn for editor chrome** — Prefer shadcn UI (Button, Label, Menubar, Tabs, etc.) for all editor chrome (tabs, menus, buttons). Avoid plain HTML elements unless necessary.
6. **Accent for selection** — Use one edge only (e.g. left or bottom) for list item / card selection; not a full border.
7. **Card and panel padding** — Use `p-[var(--panel-padding)]` (and content padding) for card headers and content so sections (e.g. AI Workflow) are not flush to borders. **Badges/tags:** use `--badge-padding-*`; do not override with ad-hoc `px-*` or `h-*`.
8. **Base shadcn tokens power all UI** — No hardcoded colors, radii, or shadows in editor chrome. Use `--panel-padding`, `--control-*`, `--radius-*`, `--shadow-*`, and semantic colors (`bg-background`, `text-foreground`, `border-border`, etc.). Visual direction is professional, sharp, and modern (VSCode/Photoshop/Spotify/Unreal); see [01 - Styling and theming](../../design/01-styling-and-theming.mdx) and [03 - Design language](../../design/03-design-language.mdx). **Themes:** Only light and dark (Spotify-based); icon size 12px; radius scale 4–8px; prefer minimal full borders (see design docs).
9. **Input/Textarea focus** — Focus is indicated by `border-ring` and a subtle `ring-1` only (no full ring-2/offset/glow) so inputs do not get a heavy full-border highlight.
10. **Toolbar outline buttons** — In app bar and editor toolbars, outline-style triggers use `border-0` and rely on `hover:bg-accent` for separation (minimal full borders). Apply via `className="border-0"` on WorkspaceButton/Button where `variant="outline"` is used for toolbar triggers (AppShell editor tabs, File menu, Project select, Workbench, Settings menu, etc.).
11. **App-level menu bar (Unreal style)** — A **single** menu bar at the top of the app (in the tab row) shows shared menus (Settings: theme, density, user, payouts, List in catalog, Open Settings) plus **editor-specific** menus (File, View, State, etc.) from the active editor. App-level Settings and platform (listings, payouts) live in this app bar menubar only. Editor toolbars contain only editor-specific tools and **do not duplicate** Settings; use the app menubar Settings entry instead. **State items** are folded into **File** (no top-level State menu) by `createWorkspaceMenubarMenus`. Use `AppMenubarProvider` and `useAppMenubarContribution` for editor menu contribution; shared Settings built with `useAppSettingsMenuItems` in the shell.
12. **Form labels (Label, FieldLabel)** — Have no border or accent by default. Underlines and accent borders are applied manually when required (e.g. via `className` or a wrapper). See `packages/ui` Label and AGENTS.md.
13. **Popovers inside overlays** — If a popover is rendered inside a Dialog/Sheet/Sidebar (e.g. Copilot sidebar), disable portalling so clicks are not treated as "outside": `PopoverContent portalled={false}`. Keep portal behavior for toolbar/global usage. Ensure popover search + filter rows have visible padding/contrast.
14. **Cmd+K assistant popup** — Use the shared `DialogueAssistantPanel` surface with tokenized shell padding (`--panel-padding`) and pass model selection via composer slot (`composerTrailing`). Keep model switching in the composer row for parity with assistant-ui and avoid duplicate top-level switchers.
15. **Overlay dialog sizing (WorkspaceOverlaySurface)** — Keep `size` values mapped to compact defaults: `sm -> max-w-md`, `md -> max-w-xl`, `lg -> max-w-2xl`, `full -> min(94vw, 72rem)` with a max-height cap. Do not use edge-to-edge form dialogs.
16. **Dialog close placement** — Close button is icon-only (`X`) and always at top-right for dialogs and editor overlays. Do not place close buttons on the left for standard forms/upserts.
17. **Form section rhythm** — Use shadcn form primitives and section wrappers with token padding/gap (`--panel-padding`, `--control-gap`). Each section should have a title row (optional icon) and visible internal spacing; avoid flush controls against borders.
18. **Entity media list guidance** — For entity upserts (character/location/etc.), keep generate/upload actions near the media lists and show current assets inline (images/audio/video). Persist interim metadata in entity `meta` fields until dedicated media collections exist.
19. **Form controls by type and layout** — **Booleans:** Always use **Switch** from `@forge/ui/switch`. Do not display boolean state as text only ("On"/"Off"). **Toggle placement:** Place the Switch **on the same row as the label**, to the **right** of the label (and any description or badges). Default is right-of-label; left-of-label only when explicitly required by design. **Multi-select:** Use **Checkbox** (or checkbox group) from `@forge/ui/checkbox`. **Single select:** Use **Select** for small option sets; use **Combobox** (when available) for large, searchable sets. **Settings and forms:** Prefer icon + label for each field when an icon mapping exists; keep layout consistent with the rules above. See [decisions.md](decisions.md) "Form and settings field patterns."
20. **Panel visibility and layout** — Panel visibility toggles and layout actions (Restore all panels, Reset layout) live **only** in **View > Layout** submenu in each editor. Do not surface panel.visible.* in Settings. **Settings Panels tab** is for **editor-exposed controls** (e.g. graph minimap, animated edges, layout algorithm for graph viewports); when the active editor has no such controls, show a short empty state. **Fit view / Fit to selection** are not in the UI (View menu, graph toolbar, or context menu); underlying APIs may remain for programmatic use (e.g. revealSelection).
21. **CSS pipeline pre-check is required** - Run `pnpm css:doctor` before and after styling changes. Treat failures as blockers: fix globals, import order, or utility generation issues first, then continue visual debugging.
22. **Docs route guardrail** - For docs layout/TOC/sidebar changes, run `pnpm docs:doctor` (includes `css:doctor` + docs tests) before finalizing. Treat failures as blockers.
23. **Docs layout baseline** - Use the shared docs shell (`DocsLayoutShell`) with **headless Fumadocs source** (serialized page tree + custom MDX map) for Studio docs. Do not import runtime Fumadocs layout/page components (`fumadocs-ui/layouts/docs`, `fumadocs-ui/page`) on the current stack. Keep custom behavior through MDX component overrides and route-scoped docs CSS token bridging (`--color-fd-*` mapped to shadcn/theme tokens). Run `pnpm docs:runtime:doctor` (included in `pnpm docs:doctor`) for guardrails.
24. **No interactive nesting** - Never place interactive descendants inside interactive wrappers (e.g. `button` containing `Switch`, `Button` containing `input`, anchor inside button unless `asChild` semantics remove the wrapper). Hydration can fail in React/Next for invalid HTML trees. Run `pnpm hydration:doctor` to catch regressions.
25. **Tooltips on buttons** - Studio uses no Radix tooltips; native `title` only via `WorkspaceButton` / `TooltipIconButton`. Do not wrap `Button` in Radix `Tooltip` until Radix ships the React 19 fix. See [standard-practices](standard-practices.md) § Tooltips and [errors-and-attempts](errors-and-attempts.md).
26. **No universal padding/margin reset** - Never add `padding: 0` or `margin: 0` to the universal selector `*` (or `::before`, `::after`). It strips default padding from buttons, labels, inputs, badges and caused days of debugging. Tailwind preflight already normalizes; do not add our own. See [errors-and-attempts](errors-and-attempts.md) § Universal padding/margin reset and the comment in `apps/studio/app/globals.css` above `@layer base`.

## Workspace and panel composition

(Repo Studio layout constraints. Apply when adding or refactoring workspaces and panels.)

- **Viewport-centric** — The viewport is the main content area; it holds files/docs (tabs) or a future graph/canvas. One viewport per workspace.
- **Tree when file-like** — For file/doc-like workspaces, primary navigation is a **tree** (one tree panel); click opens in viewport. No duplicate "list" panel for the same role.
- **Tree actions** — Structure-changing actions (New file, New folder, Rename, etc.) live on the **tree** via context menu (and optional tree toolbar), not scattered across panels.
- **Feature placement (chat-in-chat)** — Attach/reference actions (e.g. @-mentions) belong in the **assistant chat input** (composer), not as per-panel or per-doc buttons. One place of truth per action.
- **Panel discipline** — Prefer fewer, purpose-driven rails. Avoid "one panel per artifact"; group or tab related content (e.g. Overview vs Work). Define a max or grouping contract for left-rail panels (e.g. 3–4 or grouped).
- **No redundant chrome** — If an action is available in chat (e.g. @), do not repeat it as a button in every panel or doc header.

## Process when making UI/styling changes

1. **Pre-check (required)** - Run `pnpm css:doctor` and `pnpm hydration:doctor` before styling/UI changes. For docs shell changes, run `pnpm docs:doctor`.
2. **Before (optional)** - If in a debugging pass, keep or add a "before" screenshot in **docs/images/** (e.g. `half_done_buttons.png`). Reference it in design or how-to docs.
3. **Implement** - Use tokens and patterns from design/01 and 02. Fix contrast, spacing, and icons as in errors-and-attempts and this doc.
4. **After (optional)** - Do **not** run browser screenshot automation (it errors too often). When an "after" visual is needed, **humans** should capture a screenshot (run the app, use Cursor browser tools or any screenshot tool), save to **docs/images/** with a descriptive name (e.g. `styling_after_contrast_and_spacing.png`), and reference it in the relevant doc or STATUS. Agents should skip this step and note in STATUS that an optional after screenshot can be taken by a human.
5. **Docs** - Update design docs or errors-and-attempts when adding a new pattern or fix (e.g. new button variant rule or screenshot reference).

## References

- [How-to 26 – Styling debugging with Cursor](../../how-to/26-styling-debugging-with-cursor.mdx) — Full workflow (selectors, screenshots, plans).
- [01 - Styling and theming](../../design/01-styling-and-theming.mdx) — Includes **Token system** (layers, when to use which, file map) and context-aware UI. [02 - Components](../../design/02-components.mdx).
- [errors-and-attempts.md](./errors-and-attempts.md) — e.g. "Grey buttons and missing menu/icons," "Theme/surface tokens," "Toolbar buttons not switching with theme."
