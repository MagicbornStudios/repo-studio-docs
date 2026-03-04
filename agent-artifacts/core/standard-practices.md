---
title: Standard practices (now / soon / as we grow)
created: 2026-02-09
updated: 2026-02-12
---

# Standard practices (now / soon / as we grow)

Single checklist for agents and humans. Where things live and when to revisit.

## Logging

- **Now:** Env-driven structured logging in Studio (pino); `LOG_LEVEL`, `LOG_FILE`, optional `LOG_NAMESPACES`. See [apps/studio/lib/logger.ts](../../../apps/studio/lib/logger.ts) and [apps/studio/.env.example](../../../apps/studio/.env.example). Client logs can be appended to the same file in dev via `POST /api/dev/log` when `ALLOW_CLIENT_LOG=1` and `NEXT_PUBLIC_LOG_TO_SERVER=1`.
- **Revisit:** When adding new server routes or model-router code, use `getLogger('namespace')`; no ad-hoc `console.log` (see errors-and-attempts).

## Env and config

- **Now:** Env source of truth is `scripts/env/manifest.mjs`. `.env.example` files are generated, not hand-maintained:
  - `pnpm env:sync:examples`
  - `pnpm env:sync:examples:check`
- **Now:** Local setup uses the **env portal**: `pnpm dev` runs `env:bootstrap`; when keys are missing, the portal opens automatically. Run `pnpm env:portal` for ad-hoc edits, or `pnpm env:setup -- --app studio|platform|all` for CLI setup.
- **Now:** **When adding a new env key** (agents): (1) add the entry to `scripts/env/manifest.mjs`, (2) run `pnpm env:sync:examples`. The portal reads the manifest; do not hand-edit `.env` or `.env.example`.
- **Now:** Run `pnpm env:doctor -- --app ... --mode local|preview|production [--vercel]` for required-key and drift checks.
- **Now:** `dev:studio` and `dev:platform` run `env:bootstrap` before app startup (launches portal when keys missing); use `FORGE_SKIP_ENV_BOOTSTRAP=1` or `CI=1` for check-only (no portal).
- **Now:** Keep runtime env validation in centralized env readers (`apps/studio/lib/env.ts`, `apps/platform/src/lib/env.ts`) rather than scattered route-level `process.env` checks.
- **Revisit:** Add stricter CI gates for preview/production key presence once deployment automation is finalized.

## Health and ops

- **Now:** `GET /api/health` in Studio returns 200 when the app is up (optional: check DB or critical deps later for load balancers/scripts).
- **Revisit:** Add DB/Redis checks to health when we rely on them for readiness.

## Docs (MDX build)

- **Now:** All `.md` and `.mdx` under `docs/` are built by fumadocs-mdx and **must** have YAML frontmatter with at least **`title`** (string). When creating or moving a doc into `docs/`, add frontmatter; optional `created` / `updated`. See [errors-and-attempts](errors-and-attempts.md) (MDX build error) if the build fails with "invalid frontmatter".
- **Now:** Every change requires a **doc scan**: check relevant docs for drift (design/architecture/how-to/agent artifacts) and update them alongside the code change. Record the doc scan in STATUS.
- **Revisit:** When adding a new doc tree or changing how docs are loaded (e.g. source.config.mjs).

## Constants, enums, and DRY

- **Single source of truth:** Fixed sets (editor ids, API path segments, capability ids, query key namespaces) are defined once and imported elsewhere. No magic strings for these.
- **Enums:** Prefer `as const` object + derived type (e.g. `FORGE_NODE_TYPE`, `CAPABILITIES` in packages/types and packages/shared). Use TypeScript `enum` only when reverse mapping or exhaustiveness is needed.
- **Editor IDs:** Defined in app-shell ([apps/studio/lib/app-shell/store.ts](../../../apps/studio/lib/app-shell/store.ts)); new editors extend the canonical `EDITOR_IDS` and metadata (labels, viewport ids in editor-metadata.ts).
- **API routes:** Studio custom routes are listed in [apps/studio/lib/api-client/routes.ts](../../../apps/studio/lib/api-client/routes.ts); client code (services, fetch) imports path constants from there.
- **Query keys:** Use [apps/studio/lib/data/keys.ts](../../../apps/studio/lib/data/keys.ts) only; no ad-hoc `['studio', ...]` in hooks or components. For broad invalidation use the provided key helpers (e.g. `studioKeys.charactersAll()`, `studioKeys.relationshipsAll()`).
- **When adding a new API route:** Add the path to the routes module and use it in the client.
- **Storage keys:** App-session and draft persistence keys live as named constants in the respective store files (app-shell store, forge/video/character domain stores). Any new persisted key should be a named constant in one place; no magic strings for storage keys.

## Stores and subscriptions

- **Now:** When adding Zustand (or other useSyncExternalStore-based) subscriptions, selectors that return **objects or arrays** must not return a new reference on every call when the logical data is unchanged. Use **useShallow** from `zustand/shallow` (e.g. `useStore(useShallow(s => ({ x: s.x })))`) or return **primitives** only. For selectors that use `?? fallback` when a key is missing, use a **module-level constant** (e.g. `EMPTY_RAIL`) as the fallback so the same reference is returned; otherwise the fallback allocates a new ref every call and can trigger a getSnapshot loop. See errors-and-attempts.

## Tooltips (buttons)

- **Now:** Studio does not use Radix tooltips. Shared editor components (WorkspaceButton, WorkspaceTab, TooltipIconButton) use native `title` only when a tooltip string is provided. Add Radix tooltips back when Radix ships the React 19 fix (PR #3804). Base Radix tooltip primitives remain in `@forge/ui` for Platform and future use.
- **Editor chrome:** `WorkspaceButton` with `tooltip?: string` renders native `title`; no Radix Tooltip.
- **Rich tooltips:** Radix Tooltip is acceptable for non-Studio apps (e.g. Platform sidebar when collapsed). See errors-and-attempts.

## Security and resilience (soon)

- Rate limiting on public endpoints (e.g. waitlist, newsletter, checkout).
- Security headers via Next.js config.
- CORS if we add cross-origin clients.
- **Revisit:** Before opening new public API surfaces.

## As we grow

- Request/trace IDs for correlation (middleware that sets `x-request-id` and logs it).
- OpenTelemetry for distributed tracing.
- Dependency audit in CI (e.g. `pnpm audit`).
- Error boundaries and global error reporting.
- Performance budgets for critical paths.
- **Revisit:** When scaling or adding more services.
