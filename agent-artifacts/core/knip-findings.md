---
title: Knip findings and how to use it
created: 2026-02-09
updated: 2026-02-09
---

Living artifact for agents. Index: [18-agent-artifacts-index.mdx](../../18-agent-artifacts-index.mdx).

# Knip findings and how to use it

Run **`pnpm knip`** at repo root to find unused files, unused exports, and unused dependencies. This doc explains how Knip works in this repo, what the current run found, and how to triage.

## How Knip works

- **What it does:** Knip analyzes the monorepo for unused files, unused exports, unused dependencies, and unlisted dependencies. It uses the pnpm workspace layout ([pnpm-workspace.yaml](../../../pnpm-workspace.yaml)) and each package’s `package.json` and tsconfig to infer entry points and project files.
- **Entry vs project:** Entry files are the roots (e.g. `src/index.ts`, Next.js pages and layout). Project files are everything else. Exports from entry files are not reported as "unused" by default; exports from non-entry files are reported if nothing imports them.
- **Our config:** Root [knip.json](../../../knip.json):
  - **ignoreWorkspaces:** `examples/*` — those workspaces are not analyzed. (Twick fully removed 2026-02-26; no vendor/twick.)
  - **ignoreFiles:** `.tmp/**`, `**/.source/**`, `**/dist/**`, `**/node_modules/**` — those paths are excluded from the "unused files" check only (still analyzed for exports/deps where applicable).
- **Running:** `pnpm knip` at repo root. Exit code 1 when there are findings.
- **Caveats:** Barrel re-exports (index.ts), Next.js dynamic imports, Payload/OpenAPI-generated code, and CSS/side-effect imports often produce false positives. Triage before removing anything.

See also [tool-usage.md](./tool-usage.md) (Dead code section).

## What the current run found (summary)

| Category | Count | Notes |
|----------|--------|--------|
| Unused files | 71 | Root `app/`, `lib/`, `types/`, `scripts/`, `__tests__/`, `__mocks__/`; Studio/Marketing components and libs; shared CSS; UI/tool-ui components; vendor scripts. Many are entry/config or barrel files (false positives). |
| Unused dependencies | 50 | Studio: Radix, Twick, cmdk, fumadocs-ui, next-mdx-remote, react-markdown, remark-gfm, vaul, react-resizable-panels, etc. Marketing: similar. packages/ui: e.g. @radix-ui/react-separator, tailwindcss-animate. Most Radix are used transitively via @forge/ui or components. |
| Unused devDependencies | 10 | Root tsup; Studio testing libs; etc. |
| Unused exports | 200+ | Payload collections, API client exports, data hooks, barrel re-exports (e.g. apps/studio/lib/data/hooks/index.ts), components (FlowControls, FlowPanel, DocsLayoutShell, etc.). Many are consumed via barrel or at runtime. |

**Likely false positives (keep or ignore):** Jest config, Next entry files, CSS files (imported in globals), Payload collections, barrel index files, generated OpenAPI client.

**Worth dedicated triage:** Root-level `app/`, `lib/`, `types/` (legacy from pre-monorepo?) — see td-16. Unused files in Studio/packages — see td-14. Unused dependencies — see td-15.

## Root legacy (td-16)

Root `app/`, `lib/`, and `types/` are **legacy** from pre-monorepo. The real app is `apps/studio`; Jest maps `@/` to `apps/studio/`, so root `__tests__` use Studio’s lib/types, not root. Root app is not run by any script (`dev`/`build` target `@forge/studio`). Root lib is duplicate of `apps/studio/lib` (e.g. graph-operations); domain-forge’s re-export resolves to Studio’s lib when built. These paths are in **ignoreFiles** so Knip does not report them as unused. Kept for reference; a future slice can delete or move under apps/studio if desired.

## How to triage

1. Prefer **ignoreFiles** or **ignoreWorkspaces** in [knip.json](../../../knip.json) for known-good patterns (e.g. entry points, config, barrels) rather than deleting code that is actually used.
2. Before removing a file or dependency: confirm with rg/grep that nothing imports it (or that the only "use" is via a barrel or runtime that Knip doesn’t follow).
3. After a triage pass: update this doc (e.g. "As of YYYY-MM-DD: removed X; added ignoreFiles for Y").
4. Tech-debt items: [technical-debt-roadmap.md](./technical-debt-roadmap.md) td-14 (unused files), td-15 (unused dependencies), td-16 (root dead code), td-17 (unused exports).

**Triage done:** As of 2026-02-09, `react-resizable-panels` was removed from apps/studio/package.json (only used by @forge/ui resizable; Studio layout uses Dockview). td-15 triage can skip this.

**td-14 (unused files) triage:** Extended ignoreFiles for entry/config (`jest.config.js`, `jest.setup.js`), `scripts/**`, `__mocks__/**`, `__tests__/**`, `vendor/twick/scripts/**`, and `packages/shared/src/shared/styles/*.css`. Removed `apps/studio/lib/graph-to-sequence.ts` (rg-confirmed no imports). Other reported files left as-is (barrels, components, or used at runtime).

**td-15 (unused dependencies) triage:** Added **ignoreDependencies** in knip.json for Radix UI packages (used transitively via @forge/ui or by framework), fumadocs-ui, next-mdx-remote, react-markdown, remark-gfm, vaul, cmdk, class-variance-authority, posthog-js, next-themes, cobe, tailwindcss-animate, and devDependencies (tsup, @testing-library/*, eslint, eslint-config-next, jest-environment-jsdom, tailwindcss, payload). No dependencies removed; all kept or ignored per plan.

**td-17 (unused exports) triage:** Added **ignoreIssues** in knip.json for `**/index.ts`, `**/payload-types.ts`, `**/payload/collections/*.ts`, and `**/source.config.ts` with issue types `["exports","types"]` so barrel and Payload/generated exports are not reported. No exports removed; barrel/runtime/generated ignored per plan.

**Triage done (full):** 2026-02-09. td-14, td-15, td-16, td-17 completed. See technical-debt-roadmap.md (all set to done) and STATUS.md (Ralph Wiggum Done).
