---
title: Task breakdown — UI polish (compact modern)
created: 2026-02-10
updated: 2026-02-10
---

# UI polish — compact, modern, icon-rich

**Initiative id:** `ui-polish` (Tier 0)

**Parent:** Design direction in [03-design-language.mdx](../../design/03-design-language.mdx) and styling rules in [styling-and-ui-consistency.md](./styling-and-ui-consistency.md).

Goal: unify icon scale, padding, and surface contrast across editor chrome so the UI feels compact, modern, and visually strong (Spotify tone, Unreal/Unity structure).

---

## Lanes and tasks

### Lane: Theme + tokens

| id | title | parent | tier | impact | status | doc |
|----|-------|--------|------|--------|--------|-----|
| ui-polish-theme-1 | Refresh density tokens + icon size tokens (control height, badge padding, icon-size-lg) | ui-polish | 2 | Medium | done | themes.css |
| ui-polish-theme-2 | Adjust muted/control/border contrast for dark theme | ui-polish | 3 | Small | done | themes.css |

### Lane: UI primitives (menus, inputs, badges)

| id | title | parent | tier | impact | status | doc |
|----|-------|--------|------|--------|--------|-----|
| ui-polish-primitives-1 | Tokenize menus/command/select paddings + icon sizes | ui-polish | 2 | Medium | done | packages/ui |
| ui-polish-primitives-2 | Badge padding tokens + outline contrast | ui-polish | 3 | Small | done | packages/ui |
| ui-polish-primitives-3 | Audit remaining @forge/ui components for `size-4/size-5` and convert to `--icon-size` | ui-polish | 3 | Small | open | packages/ui |

### Lane: Editor surfaces

| id | title | parent | tier | impact | status | doc |
|----|-------|--------|------|--------|--------|-----|
| ui-polish-editor-1 | Model switcher spacing + chip padding + search input alignment | ui-polish | 3 | Small | done | apps/studio |
| ui-polish-editor-2 | SectionHeader + NodePalette icon/padding fixes | ui-polish | 3 | Small | done | apps/studio |
| ui-polish-editor-3 | Audit editor lists/toolbars/panels for ad-hoc spacing + oversized icons | ui-polish | 3 | Small | open | apps/studio |

### Lane: Dialogs + forms

| id | title | parent | tier | impact | status | doc |
|----|-------|--------|------|--------|--------|-----|
| ui-polish-dialog-1 | Standardize editor overlay size map and right-close placement contract | ui-polish | 3 | Small | done | packages/shared + packages/ui |
| ui-polish-dialog-2 | Audit all Studio upsert dialogs for shadcn form section structure + tokenized spacing | ui-polish | 2 | Medium | open | apps/studio |
| ui-polish-dialog-3 | Audit modal/icon button sizing and close affordances (12px icon baseline) | ui-polish | 3 | Small | open | apps/studio + packages/ui |
| ui-polish-dialog-4 | Normalize entity media sections (images/audio/video lists + inline actions) across character/location flows | ui-polish | 2 | Medium | open | apps/studio |

### Lane: Docs + QA

| id | title | parent | tier | impact | status | doc |
|----|-------|--------|------|--------|--------|-----|
| ui-polish-docs-1 | Update design docs (01/02/03) for soft-modern + tokens + icon scale | ui-polish | 3 | Small | done | docs/design |
| ui-polish-docs-2 | Update styling-and-ui-consistency + errors-and-attempts with icon/padding fixes | ui-polish | 3 | Small | done | docs/agent-artifacts |
| ui-polish-docs-3 | Capture before/after screenshots (human) and save to docs/images | ui-polish | 3 | Small | open | docs/images |
| ui-polish-docs-4 | Update coding-agent-strategy with doc scan requirement | ui-polish | 3 | Small | done | docs/19-coding-agent-strategy.mdx |

---

## Reference

- **Process:** [task-breakdown-system.md](./task-breakdown-system.md) · [task-registry.md](./task-registry.md)
