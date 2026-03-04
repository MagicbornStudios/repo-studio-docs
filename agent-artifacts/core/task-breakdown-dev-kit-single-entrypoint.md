---
title: Task breakdown — Dev-kit single entrypoint and ease of use
created: 2026-02-09
updated: 2026-02-09
---

# Dev-kit: single entrypoint and ease of use

**Initiative id:** `dev-kit-single-entrypoint` (Tier 0)

**Parent:** [STATUS § Next #18](./STATUS.md) · [product roadmap](../../roadmap/product.mdx) · [Dev-kit and keys](../../product/dev-kit-and-keys.mdx)

One package (`@forge/dev-kit`), one style import (bundled dockview + overrides), no dockview in user code. Customer-facing docs (Quick start, guides, reference); internal docs (architecture 04, Verdaccio 25, dev-kit-and-keys). Optional create-forge-app CLI.

---

## Lanes and tasks

### Lane: Single entrypoint (package + styles)

| id | title | parent | tier | impact | status | doc |
|----|-------|--------|------|--------|--------|-----|
| dev-kit-css-1 | Bundle and export editor CSS (dockview base + overrides) from shared or dev-kit | dev-kit-single-entrypoint | 2 | Medium | done | [04-component-library](../../architecture/04-component-library-and-registry.mdx) |
| dev-kit-css-2 | Document single style import in 04 and customer Quick start | dev-kit-single-entrypoint | 3 | Small | done | — |
| dev-kit-ui-1 | Document one recommended path (e.g. dev-kit.ui for atoms) in customer docs | dev-kit-single-entrypoint | 3 | Small | done | — |

### Lane: Customer-facing docs (platform app)

| id | title | parent | tier | impact | status | doc |
|----|-------|--------|------|--------|--------|-----|
| dev-kit-docs-quickstart | Quick start: install, one style import, one snippet (AppProviders -> WorkspaceApp -> WorkspaceShell -> editor) | dev-kit-single-entrypoint | 2 | Medium | open | apps/platform/content/docs |
| dev-kit-docs-install | Install & setup: one package, peer deps, env (OPENROUTER_API_KEY) | dev-kit-single-entrypoint | 3 | Small | open | — |
| dev-kit-docs-first-editor | Your first editor: Strategy (or Dialogue), wire /api/assistant-chat, run | dev-kit-single-entrypoint | 2 | Medium | open | — |
| dev-kit-docs-layout | Layout and shell: WorkspaceShell, DockLayout, DockPanel, PanelTabs from @forge/dev-kit | dev-kit-single-entrypoint | 3 | Small | open | — |
| dev-kit-docs-styling | Styling and theming: one style import; data-theme / data-density if supported | dev-kit-single-entrypoint | 3 | Small | open | — |
| dev-kit-docs-api-keys | API keys and AI: OpenRouter (and others) in one place | dev-kit-single-entrypoint | 3 | Small | open | — |
| dev-kit-docs-components | Expand component pages (dock-layout, editor-shell, panel-tabs): props, example, import from @forge/dev-kit | dev-kit-single-entrypoint | 2 | Medium | open | — |
| dev-kit-docs-api-ref | Expand api-reference/dev-kit.md: main exports, one entrypoint | dev-kit-single-entrypoint | 3 | Small | open | — |
| dev-kit-docs-editors | Editors: Strategy, Dialogue, Character, Video (what each is, when to use, minimal sample) | dev-kit-single-entrypoint | 2 | Medium | open | — |
| dev-kit-docs-registry | Private registry page: .npmrc, pnpm add @forge/dev-kit, then Quick start | dev-kit-single-entrypoint | 3 | Small | open | — |

### Lane: Internal docs (repo)

| id | title | parent | tier | impact | status | doc |
|----|-------|--------|------|--------|--------|-----|
| dev-kit-internal-04 | 04-component-library: single customer entrypoint = @forge/dev-kit; internal = four packages | dev-kit-single-entrypoint | 3 | Small | open | [04](../../architecture/04-component-library-and-registry.mdx) |
| dev-kit-internal-25 | 25-verdaccio: how to run and publish; include create-forge-app when added | dev-kit-single-entrypoint | 3 | Small | open | [25](../../how-to/25-verdaccio-local-registry.mdx) |
| dev-kit-internal-keys | dev-kit-and-keys: product context; customer API keys on marketing only | dev-kit-single-entrypoint | 3 | Small | open | [dev-kit-and-keys](../../product/dev-kit-and-keys.mdx) |
| dev-kit-internal-status | STATUS/decisions: reference single entrypoint, CSS bundle, no dockview in user code | dev-kit-single-entrypoint | 3 | Small | open | STATUS, decisions |

### Lane: Optional create-forge-app

| id | title | parent | tier | impact | status | doc |
|----|-------|--------|------|--------|--------|-----|
| dev-kit-cli-1 | Add packages/create-forge-app: package.json (bin, files), tsconfig, tsup, src/cli.ts (copy template, install, next steps) | dev-kit-single-entrypoint | 2 | Medium | open | — |
| dev-kit-cli-2 | Add template: standalone Next app (Strategy editor, /api/assistant-chat, one style import from dev-kit) | dev-kit-single-entrypoint | 2 | Medium | open | — |
| dev-kit-cli-3 | Wire build and publish (root scripts, Verdaccio docs); customer Quick start "Create app with Forge" | dev-kit-single-entrypoint | 3 | Small | open | — |

---

## Reference

- **Initiative:** [STATUS § Next #18](STATUS.md) · [product roadmap](../../roadmap/product.mdx) · [Dev-kit and keys](../../product/dev-kit-and-keys.mdx)
- **Process:** [task-breakdown-system.md](./task-breakdown-system.md) · [task-registry.md](./task-registry.md)
