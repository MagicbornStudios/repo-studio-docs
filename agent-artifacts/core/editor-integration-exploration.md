---
title: Editor integration and descriptor generation (exploration)
created: 2026-02-11
updated: 2026-02-11
---

Living artifact. Index: [18-agent-artifacts-index.mdx](../../18-agent-artifacts-index.mdx).

# Editor integration and descriptor generation (exploration)

**Goal:** Make it easy for third parties to create and integrate custom editors into Forge Studio. Current flow requires manual EDITOR_IDS, metadata, bootstrap registration, and full JSX layout. This doc explores options to reduce friction.

## Current integration path

1. Add editor id to `EDITOR_IDS` and `editor-metadata.ts`
2. Create component with `WorkspaceShell` + `EditorDockLayout` + slot children
3. Export `editorDescriptor` (id, label, icon, order)
4. Register in `editor-bootstrap.ts`

Editors compose panels declaratively via `EditorDockLayout.Left/Main/Right` + `EditorDockLayout.Panel` children. Full control, but boilerplate-heavy for simple editors.

## RailPanelDescriptor and config-driven layout

`RailPanelDescriptor` (`{ id, title, icon?, content }`) and `leftPanels`/`mainPanels`/`rightPanels`/`bottomPanels` arrays exist in `EditorDockLayout`. Today editors use **slot children**; the array API is available for config-driven layouts.

**Option A — Config helper:** `createEditorLayout(config)` that accepts `{ left: [...], main: [...], right: [...] }` and returns `<EditorDockLayout leftPanels={...} mainPanels={...} rightPanels={...} />`. Each panel config: `{ id, title, icon?, render: () => <Content /> }` or `content: ReactNode`. Reduces JSX for editors that fit the config shape.

**Option B — EditorDescriptor extension:** Extend `EditorDescriptor` with optional `defaultPanels?: RailPanelDescriptor[]` or `getLayoutConfig?: () => LayoutConfig`. Studio could render a generic editor wrapper that uses this config when the editor doesn’t provide a full component. Requires Studio to support “config-only” editors.

**Option C — Plugin/package discovery:** Editors in a `plugins/` or `packages/editors/*` folder are auto-discovered; each exports an `editorDescriptor` + default. Reduces manual bootstrap edits. Aligns with future “developer program” / ecosystem work.

## Descriptor generation

**What to generate:**
- Panel descriptors from a schema (e.g. YAML manifest: `left: [{ id, title, icon }]`) → `RailPanelDescriptor[]`
- Editor scaffolding: `pnpm create-forge-editor my-editor` → boilerplate with descriptor, shell, layout
- Codegen from OpenAPI/schema for inspector sections

**Constraints:**
- Slot children (JSX) remain the recommended path for full control
- Config-driven API is opt-in for simpler editors
- Keep `EditorDescriptor` as the registry contract; extend, don’t replace

## Next steps

1. Document the “minimal editor” path (smallest viable WorkspaceShell + layout + descriptor)
2. Add `createEditorLayout` helper if we see repeated config patterns
3. Consider `EditorDescriptor.defaultPanels` for plug-and-play when Studio supports it
4. Revisit when developer program / plugin ecosystem work starts

## Related

- [task-breakdown-dock-rails](./task-breakdown-dock-rails.md) — RailPanelDescriptor, array API
- [how-to 05 - Building an editor](../../how-to/05-building-an-editor.mdx) — current integration steps
- [decisions § Panel layout](./decisions.md) — UI-first, no store-driven registration
