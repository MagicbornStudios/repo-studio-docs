---
title: Task breakdown – Declarative editor registries
created: 2026-02-10
updated: 2026-02-10
---

Initiative: **editor-declarative-registries**. Impact: Large.

## Lanes and tasks

| id | title | status |
|----|-------|--------|
| panel-registry-store | Panel registry (Zustand) + useEditorPanels | done |
| EditorLayoutProvider-WorkspaceRail-WorkspaceRailPanel | Shared WorkspaceRail/WorkspaceRailPanel + Studio EditorLayoutProvider | done |
| EditorLayout-consumer | EditorLayout reads registry, filters by visibility, renders EditorDockLayout | done |
| derive-panel-specs-from-registry | useEditorPanelVisibility and View menu derive from registry | done |
| settings-registry-store | Settings registry (Zustand) + AppSettingsProvider, SettingsSection, SettingsField | done |
| AppSettingsPanelContent-read-registry | AppSettingsPanelContent merges app/viewport sections from registry | done |
| ViewportSettingsProvider | ViewportSettingsProvider; graph-viewport section from Dialogue | done |
| WorkspaceMenubarContribution | WorkspaceMenubarContribution + WorkspaceMenubarMenuSlot; setEditorMenus on mount | done |
| migrate-Dialogue | DialogueEditor: EditorLayoutProvider, rails/panels, menubar, viewport settings | done |
| migrate-Character | CharacterEditor: same pattern | done |
| migrate-Video | VideoEditor: same pattern | done |
| migrate-Strategy | StrategyEditor: EditorLayoutProvider + WorkspaceMenubarContribution only | done |
| deprecate-EDITOR_PANEL_SPECS | EDITOR_PANEL_SPECS deprecated (fallback only) | done |
| docs-and-ADR | decisions.md ADR, task-registry, breakdown, STATUS, README, AGENTS | done |

## Notes

- Visibility key convention: `panel.visible.{editorId}-{panelId}`.
- Components require correct provider (FormField-within-Form); document in shared README and AGENTS.
- Static settings section lists remain; registry sections override when registered.
