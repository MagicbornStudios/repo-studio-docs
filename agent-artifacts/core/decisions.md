---
title: Architecture decision records
created: 2026-02-04
updated: 2026-02-12
---

Living artifact for agents. Index: [18-agent-artifacts-index.mdx](../../18-agent-artifacts-index.mdx).

# Architecture decision records

> **For coding agents.** See [Agent artifacts index](../../18-agent-artifacts-index.mdx) for the full list.

When changing persistence or the data layer, read this file and **docs/11-tech-stack.mdx**. Update these docs when accepting or rejecting a significant choice.

---

## Documentation structure: showcase model, env reference, settings codegen

**Decision:** Docs use a showcase model (atoms → molecules → organisms), environment variables reference from manifest, and settings codegen from tree-as-source. Structure: onboarding → showcase → how-to → architecture → reference. DocLink expects `.mdx`; hash anchors for sections.

**Rationale:** Single source of truth for env (manifest), settings (tree + codegen), and components (showcase). Agents follow [docs-building](docs-building.md) when adding components, env vars, or settings.

---

## Client boundary: Payload REST for CRUD, custom routes for app ops

**Decision:** For collection CRUD (forge-graphs, video-docs), the client uses the **Payload SDK** against Payload's auto-generated REST API (`/api/forge-graphs`, `/api/video-docs`). For app-specific operations (auth shape, settings upsert, AI, model config, SSE), the client uses our **custom Next API routes** and the generated or manual client (e.g. `/api/me`, `/api/settings`, `/api/forge/plan`, `workflows.ts` for SSE).

**Rationale:** Payload's REST gives us full CRUD and querying without duplicating handlers; custom routes keep a single place for auth, settings, and non-CRUD logic.

---

## Studio API CORS boundary for platform auth and customer APIs

**Decision:** Studio owns cross-origin CORS for customer-platform requests (`apps/platform` -> `apps/studio`) at one boundary: shared origin parsing in `apps/studio/lib/server/cors.ts`, applied to Payload auth endpoints via `payload.config.ts` (`cors` + `csrf`) and to all custom routes via `apps/studio/middleware.ts` (`/api/:path*`, including preflight handling). Default local origins include `localhost` / `127.0.0.1` on ports `3000` and `3001`; production/staging origins are extended through `CORS_ALLOWED_ORIGINS` (comma-separated) or `PLATFORM_APP_URL`.

**Rationale:** Platform login and session checks use cookie credentials across origins (`POST /api/users/login`, `GET /api/me`). Without a centralized allowlist and preflight response headers, browser CORS blocks auth before route logic runs. Centralizing avoids route-by-route CORS drift and keeps Payload + custom Next routes consistent.

---

## Local dev auto-admin bootstrap (Studio only)

**Decision:** In local Studio development, we bootstrap a real Payload auth session for the seeded admin before auth-dependent app providers mount. This is implemented with `LocalDevAuthGate` in `apps/studio/components/providers/LocalDevAuthGate.tsx`, wrapped early in `AppProviders`. It runs only when all are true:

- `NODE_ENV === 'development'`
- browser hostname is `localhost` or `127.0.0.1`
- `NEXT_PUBLIC_LOCAL_DEV_AUTO_ADMIN !== '0'`

The gate checks `GET /api/me`; if unauthenticated it performs `POST /api/users/login` with local env/default seed credentials, then invalidates `authKeys.me()`.

**Rationale:** Local developers should not be blocked by 403s from protected Payload REST queries while iterating. Using real auth bootstrap preserves production access rules and avoids unsafe global ACL bypasses.

---

## Env DX source of truth: manifest + generated examples + setup/doctor tooling

**Decision:** Environment-variable onboarding and drift checks are now manifest-driven for Studio and Platform:

- Source of truth: `scripts/env/manifest.mjs`
- Generated outputs: `apps/studio/.env.example`, `apps/platform/.env.example` via `pnpm env:sync:examples`
- Local setup wizard: `pnpm env:setup -- --app studio|platform|all`
- Drift/required checks: `pnpm env:doctor -- --app ... --mode local|preview|production [--vercel]`
- Dev bootstrap guard: `pnpm env:bootstrap -- --app ...` (wired into `dev:studio` and `dev:platform`; launches portal when keys missing; falls back to `env:ensure:local` when CI or FORGE_SKIP_ENV_BOOTSTRAP=1)

Runtime env boundaries are centralized in app env helpers:

- `apps/studio/lib/env.ts` (OpenRouter, Stripe, local auto-admin)
- `apps/platform/src/lib/env.ts` (Studio URL resolution)

**Rationale:** Manual `.env.example` editing drifted and made onboarding brittle. A single manifest keeps required keys, docs, and startup checks aligned while preserving secret masking and local convenience.

---

## Platform organizations as first-class customer context

**Decision:** The customer-facing platform now uses organizations as first-class context. Studio stores organizations in `organizations` and `organization-memberships` collections; users carry `defaultOrganization`, and creator-owned docs (`projects`, `listings`, `licenses`) include org relationships where applicable. Platform APIs resolve organization context server-side (`/api/me/orgs`, `/api/me/orgs/active`, org-scoped `/api/me/*` endpoints). If a user has no org membership, Studio auto-bootstraps a personal workspace organization.

**Rationale:** Dashboard, billing, payout setup, and marketplace ownership need stable org context beyond individual users. Auto-bootstrap preserves backward compatibility for existing user-owned data while enabling org-aware APIs immediately.

---

## Viewport tab bar wheel scroll (Repo Studio)

**Decision:** The WorkspaceViewport tab bar (Planning/Story doc tabs) maps vertical mouse wheel to horizontal scroll so many open tabs can be scrolled with the wheel. We use a **native** `addEventListener('wheel', handler, { passive: false })` in a useEffect (with cleanup), not React's `onWheel`. We do not add a third-party package; the tab bar is custom content inside a Dockview panel, so Dockview does not control it.

**Rationale:** React's synthetic `onWheel` is registered as passive by the browser, so `preventDefault()` inside it is ignored and triggers "Unable to preventDefault inside passive event listener invocation." Only a non-passive native listener allows us to prevent default and scroll the tab bar horizontally. No small, focused npm package was worth adding for this; the in-house effect is minimal and maintainable.

---

## Planning doc viewer: Monaco + headless-tree (Repo Studio)

**Decision:** Planning doc viewport tab content uses **Monaco Editor** in read-only mode with `language="markdown"` for syntax highlighting (no lighter option such as react-markdown + shiki). The Documents panel doc list is a **hierarchical tree** built from `planning.docs` file paths using **@headless-tree/core** and **@headless-tree/react**; tree data from `buildPlanningDocTreeData(docs)` (folder id = path prefix, file id = doc.id); minimal Tree/TreeItem/TreeItemLabel UI in repo-studio with Folder/File icons and leaf click opening the doc in the main viewport.

**Rationale:** Monaco is already used for Story and Diff markdown; read-only markdown in the planning tab keeps behavior consistent. Headless-tree provides accessible, keyboard-friendly tree with a small API surface; building the tree from paths avoids a separate filesystem API and keeps the list in sync with planning snapshot.

---

## Org-scoped storage metering and enterprise request intake

**Decision:** Organization billing context now includes storage and enterprise entitlements. We added:

- `storage-usage-events` (immutable storage delta ledger)
- `enterprise-requests` (manual-gated enterprise request workflow)
- organization billing fields (`planTier`, `storageQuotaBytes`, `storageUsedBytes`, warning threshold, enterprise flags, `stripeCustomerId`)
- org-scoped storage APIs:
  - `GET /api/me/storage/summary`
  - `GET /api/me/storage/breakdown`
  - `POST /api/me/storage/upgrade-checkout-session`
- org-scoped enterprise API:
  - `GET|POST /api/me/enterprise/requests`

Storage growth enforcement is hard-blocked for upload/clone growth paths (warn threshold + over-limit check). Clone flows now assign organization explicitly and meter resulting storage.

**Rationale:** Financial visibility and quota enforcement must be org-accurate for billing and marketplace operations. Keeping this in Studio APIs preserves the single client boundary and avoids direct browser-Payload access.

---

## Stripe webhook settlement idempotency and org attribution

**Decision:** Stripe checkout settlement is idempotent by `stripeSessionId`:

- `licenses.stripeSessionId` is unique.
- Webhook lookup/create logic upserts by session id and safely handles retries.
- First clone runs only when `clonedProjectId` is missing.
- Revenue attribution prefers `licenses.sellerOrganization` for org-ledger correctness.
- Storage-upgrade sessions are processed once per org via `lastStorageUpgradeSessionId`.

**Rationale:** Stripe retries are expected. Without session-id idempotency, duplicate deliveries can create duplicate licenses and clones and distort financial/storage ledgers.

---

## AI usage ledger for billing-grade platform visibility

**Decision:** AI usage/cost tracking is persisted server-side in `ai-usage-events` (request id, user, organization, model, token totals, and computed USD costs). Platform financial pages consume org-scoped usage APIs (`/api/me/ai-usage`, `/api/me/ai-usage/summary`). Studio AI routes record usage events at completion/error boundaries (`/api/assistant-chat`, `/api/forge/plan`, `/api/structured-output`, `/api/image-generate`).

**Rationale:** PostHog analytics events are useful for behavior but not a billing ledger. Persisted usage events provide auditable, queryable spend/volume data for creator dashboards and future invoicing/reconciliation slices.

---

## Platform API keys for AI capabilities (hashed + scoped + revocable)

**Decision:** Platform API keys are first-class credentials for programmatic AI access. Keys are generated per user + organization and stored as **hashed secrets only** (`api-keys` collection: `keyId`, `secretSalt`, `secretHash`, scopes, expiry/revocation, usage counters). Plaintext key is shown once on create (`POST /api/me/api-keys`), never retrievable later. Management endpoints are org-scoped (`GET /api/me/api-keys`, `DELETE /api/me/api-keys/:id` revoke).

AI routes require either authenticated session or valid API key with required scope (`ai.chat`, `ai.plan`, `ai.structured`, `ai.image`). Usage events now record auth source (`authType`) and optional key relation (`apiKey`) and increment per-key usage counters.

**Rationale:** This is the standard secure pattern for customer API keys: one-time reveal, hashed-at-rest secrets, least-privilege scopes, revocation/expiry, and auditable per-key usage attribution.

---

## Constants for editor IDs and API routes (DRY)

**Decision:** Editor IDs are defined once in the app-shell store (`EDITOR_IDS`, `EditorId`, `DEFAULT_EDITOR_ID`). Custom API route paths used by the client are defined in [apps/studio/lib/api-client/routes.ts](../../../apps/studio/lib/api-client/routes.ts) (`API_ROUTES`). Query keys use [apps/studio/lib/data/keys.ts](../../../apps/studio/lib/data/keys.ts) only; no ad-hoc `['studio', ...]` in hooks. See [standard-practices](standard-practices.md) § Constants, enums, and DRY.

**Rationale:** Single source of truth reduces typos and makes adding a new editor or route a one-place change. Update this doc if we change where constants live.

---

## TanStack Query for server-state

**Decision:** Server-state (graphs, video docs, lists, "me", pricing, etc.) is fetched and cached via TanStack Query. Query keys and hooks live in `apps/studio/lib/data/` (keys, hooks). Hooks use the Payload SDK for collection CRUD and the generated/manual client for custom endpoints. Mutations invalidate the relevant queries after save.

**Rationale:** Caching, deduping, loading/error/retry, and invalidation are handled in one place; components stay declarative.

---

## Project context at app level

**Decision:** The active project is owned by the **app shell**, not by individual editors. The app-shell store holds `activeProjectId`; the `ProjectSwitcher` lives in the editor tab bar (AppShell). Dialogue and Character (and any other project-scoped editor) read this single project id and sync it into their domain stores. This keeps project context cohesive across editors.

**Rationale:** Users expect one "current project"; switching project in one place should affect all editors. Update this doc if we introduce editor-specific project overrides or multi-project views.

---

## Zustand for drafts and UI route

**Decision:** Draft edits (current graph doc, video doc) and UI state (app shell route, **active project**, selection, open tabs, editor bottom drawer) live in Zustand. Draft is seeded from server data; save is a mutation that then invalidates queries. App shell route and "last document id" are persisted via **Zustand persist** (app-shell store). Graph and video stores use persist with partialize for dirty drafts; we rehydrate those conditionally when the persisted draft's documentId matches the current doc. Editor-level UI (e.g. bottom drawer open/closed) lives in the app-shell store, keyed by editor id.

**Rationale:** Drafts are client-owned until save; keeping them in Zustand avoids fighting the query cache and keeps a clear "dirty" and "save" flow. Using persist middleware avoids a separate localStorage abstraction and keeps versioning in one place.

---

## Persisted client state: Zustand persist (no separate localStorage layer)

**Decision:** localStorage is used only via **Zustand persist** for app session (route, last doc ids) and draft snapshots (graph/video). We do not maintain a separate `local-storage.ts` with get/set helpers for those. React Query remains the source for server state; we only persist "which doc" and "unsaved draft" in persisted stores.

**Rationale:** One less abstraction, versioning/migrations in one place, and standard middleware. Users keep tab layout, last-opened document, and unsaved drafts across reloads.

---

## One Payload schema/DB for now

**Decision:** We run a single Payload schema and DB to keep local-first dev simple and avoid cross-service auth/identity. Studio and Platform can be separate apps later but should share one schema and type generation at this stage.

**Rationale:** Deferring multi-DB and multi-backend reduces complexity; when we need them, we keep the same client contract by moving complexity behind Next API routes.

---

## Deferring multi-DB

**Decision:** We do not split into multiple DBs or backends until product needs justify it. Collection CRUD uses Payload REST (Payload SDK) from the browser; non-CRUD uses our custom Next API and generated/manual client. No raw fetch in components or stores for server state—use hooks and the Payload SDK or custom client.

**Rationale:** Single DB and a clear split (Payload REST for CRUD, custom routes for app ops) are easier to reason about; "why we don't" is in **docs/11-tech-stack.mdx**.

---

## Repo Studio dev DB: wipe at dev start, Payload owns schema

**Decision:** Repo Studio uses a single SQLite DB (Payload + Drizzle Studio). **Payload owns the schema** (collections in `apps/repo-studio/payload/collections/`). In dev, we do not rely on persistent DB data: at each `pnpm dev` (or `pnpm dev:repo-studio`), we run `db:reset-dev` first, which deletes the SQLite file (same path as `payload.config.ts`). Payload then recreates tables from the current collection config, so Payload never prompts "Is X column created or renamed?" (that prompt comes from Payload CMS in dev when it sees schema drift). Drizzle is used only for introspection (`db:pull`) and Drizzle Studio (DB UI); we do not run `drizzle-kit generate` against this DB. Standalone wipe: `pnpm --filter @forge/repo-studio-app db:reset-dev`.

**Rationale:** Avoids one-off migration scripts and interactive Payload prompts during development; keeps dev reproducible. Production or long-lived data uses a different workflow (explicit migrations or separate DB path).

---

## Private package publishing via Verdaccio

**Decision:** Foundation packages are published to a **local Verdaccio registry** under the `@forge/*` scope. We publish `@forge/ui`, `@forge/shared`, `@forge/agent-engine`, and the convenience meta-package `@forge/dev-kit`. Domain packages stay private.

**Rationale:** This keeps the public surface small while enabling external apps to adopt the workspace/editor architecture and Copilot utilities quickly. Verdaccio is local-first, fast, and avoids public npm while we iterate.

---

## Dev-kit single entrypoint for customers

**Decision:** The customer-facing entrypoint is **one package** (`@forge/dev-kit`) and **one style import** (e.g. `@forge/dev-kit/styles` or shared editor bundle). Users do not import `dockview` or `@forge/shared` directly; we do not expose dockview in our public API or docs.

**Rationale:** Ease of use and clear contract; internal packages (shared, ui, agent-engine) are implementation detail. Update this doc when the CSS export path is finalized.

---

## CopilotKit + OpenRouter: OpenAI SDK and AI SDK with baseURL only

**Decision:** We use the **OpenAI** npm package and **@ai-sdk/openai** (Vercel AI SDK) with **baseURL** set to OpenRouter (`https://openrouter.ai/api/v1`). We do **not** use `@openrouter/ai-sdk-provider` for the CopilotKit route or shared runtime.

**Rationale:** CopilotKit's `OpenAIAdapter` and `BuiltInAgent` are hardcoded to the `openai` and `@ai-sdk/openai` interfaces. OpenRouter's recommended approach is this same pattern (OpenAI SDK + baseURL). Using a different SDK leads to incompatibility and runtime swaps; see errors-and-attempts.

---

## CopilotKit responses v2 compatibility + assistant-ui chat pipeline

**Decision:** CopilotKit BuiltInAgent continues to use the **responses** API, but model selection is **filtered to responses‑v2 compatible models** (with a safe fallback). The assistant‑ui chat surface uses the **chat** pipeline by default so it can use a broader set of OpenRouter models without responses‑v3 failures.

**Rationale:** AI SDK 5 only supports responses spec v2; some OpenRouter‑backed providers (Gemini, Claude) return v3 and cause runtime errors. We keep CopilotKit for agent actions while preventing incompatibilities, and rely on assistant‑ui for a resilient, standard chat UI. This gives us two surfaces without blocking workflows.

**Implementation note:** Model selection stays **global** (ModelSwitcher + model router preferences). CopilotKit enforces responses‑v2 compatibility at runtime; assistant‑ui uses the same model selection via `/api/assistant-chat` but through the chat pipeline.

---

## Studio and editors as MCP-driven, chat-first

**Decision:** Studio and editors will be **MCP-driven and chat-first**. The Studio app exposes MCP tools (publish, update listing, switch/open/close editor, project context); each editor is an MCP app and registers with Studio. **CopilotKit and assistant-ui remain** for **in-Studio** chat (browser); **external** control (Cursor, Claude, ChatGPT, Atlas, Perplexity, etc.) is via the Studio MCP Server. Caveats (two control planes, auth, drift risk) and possible solutions (unified tool backend, MCP-as-transport) are recorded in [11 - MCP vs CopilotKit and assistant-ui](../../architecture/11-mcp-copilotkit-assistant-ui.mdx); we do not re-decide there, we implement from that doc.

**Rationale:** Chat-first control from any host is on the product roadmap; documenting the direction and caveats lets implementation proceed without re-opening the design. In-Studio experience stays CopilotKit/assistant-ui; external experience is MCP.

---

## Structured logging: pino in Studio, file and optional client-to-file

**Decision:** Studio uses **pino** for structured logging. Env: `LOG_LEVEL` (default `info`), `LOG_FILE` (optional path; when set, logs go to stdout and file). `getLogger(namespace)` returns a child logger so every line has a namespace. Optional in dev: `ALLOW_CLIENT_LOG=1` and `NEXT_PUBLIC_LOG_TO_SERVER=1` enable `POST /api/dev/log`, which appends client log payloads to the same `LOG_FILE` with a `client: true` field. Log file path (e.g. `.logs/`) is in `.gitignore`.

**Rationale:** One file in the codebase for tracing without watching browser or terminal; env-driven level and namespaces; client-to-file is opt-in and dev-only.

---

## Settings overrides scoped by user

**Decision:** Settings overrides can be scoped by user. The `settings-overrides` collection has an optional `user` relationship. When authenticated, GET `/api/settings` returns only overrides where `user` equals the current user; POST sets `user` on create and update. Unauthenticated requests use overrides where `user` is null (global/legacy). Theme and app/editor settings are thus per-user when logged in.

**Rationale:** One collection, one API; server derives user from auth. No client contract change. Update this doc if we add Payload access control by user or migrate legacy rows to users.

---

## Settings UI: single left drawer with scope tabs

**Decision:** Settings are presented in a **single left-side drawer** (shadcn Sheet, `side="left"`). One Settings control in the app bar opens this drawer (no dropdown). The drawer contains **scope tabs**: App, User (same content as App when logged in), Project (when `activeProjectId` is set), Editor (when an editor is active), Viewport (when in context). A **single schema** (`lib/settings/schema.ts`) lists every setting key with type, label, default, and which scopes show it; we **derive** `SETTINGS_CONFIG` defaults (app, project) and section definitions (APP_SETTINGS_SECTIONS, etc.) from this schema so adding or changing a key in one place adds/updates the control and the default.

**Rationale:** One surface for all settings; project scope allows per-project overrides; single schema avoids duplicate definitions (defaults vs section fields) and keeps the form in sync as settings evolve. See plan "Settings drawer (left sidebar) and form strategy."

---

## Form and settings field patterns (design system)

**Decision:** **Booleans** are rendered as **Switch** only; do not show "On"/"Off" text next to the Switch. **Toggle/Switch placement:** default is **right of the label** on the same row (label left, control right). Apply in settings and any form that has boolean toggles. **Single selection** from a small set: use **Select** (dropdown); from a large set use **Combobox** (searchable) when we add or use it. **Multiple selection:** use **checkboxes** (checkbox group); when the schema supports a multiselect type, use Checkbox from `@forge/ui/checkbox` for each option. **Settings and forms:** Each field row that has an icon mapping shows **icon + label**; for toggles, the Switch is on the same row to the right of the label (and badges/Reset if present). Other control types (text, number, select, textarea) keep label above and control below until we standardize further.

**Rationale:** Consistent patterns reduce drift and rework; new forms (settings, modals, dialogs) should follow these rules. Update this doc when we add new field types (e.g. multiselect, combobox) or change default layout.

---

## AI and media provider stack

**Decision:** We use a split provider stack: **OpenRouter** for text (chat, streaming, structured output, plan) and image (generation, vision); **ElevenLabs** for audio TTS (character voices, previews); **OpenAI Sora** (or equivalent) **planned** for video generation. OpenRouter does not provide audio TTS/STT or video; we use specialized providers.

**Rationale:** Matches OpenRouter's actual capabilities and keeps a single place to document "who does what." STT and video are documented as future (e.g. Whisper for STT, Sora for video when we add them).

---

## Copilot not gated

**Decision:** The **Copilot** (AI sidebar, plan/patch/review, model selection in chat) is **not** behind a plan or capability gate. All users with access to an editor can use the Copilot. Future platform features (publish, monetize) may be gated via `user.plan` and `CAPABILITIES`; Copilot remains free to use.

**Rationale:** Product choice to keep AI assistance available without paywall. Update this doc if we introduce any Copilot-related gating (e.g. rate limits or premium models only).

---

## Platform gates and plan

**Decision:** Platform capabilities (`PLATFORM_LIST`, `PLATFORM_PUBLISH`, `PLATFORM_MONETIZE`) are granted to the **pro** plan only; **free** users cannot list, publish, or monetize. Plan is hydrated from `GET /api/me` and stored in the entitlements store; `getStatus(capability)` uses `PLAN_CAPABILITIES[plan]` in `apps/studio/lib/entitlements/store.ts`. No extra wiring is required—EntitlementsProvider already sets plan from `useMe()` and exposes `has`/`get` from the store.

**Rationale:** Single source of truth (plan from API → store → FeatureGate). Update this doc if we add new plan tiers or move platform gates to a different mechanism.

---

## Platform capability checks: listing create (server-side)

**Decision:** Listing create (and update) is enforced server-side by **user.plan**. The `listings` collection has a `beforeChange` hook: if `data.status === 'published'`, the user must have `plan === 'pro'`; if `data.price > 0`, the user must have `plan === 'pro'`. Otherwise the hook throws and the request fails. This backs the UI gates in CreateListingSheet (PLATFORM_PUBLISH, PLATFORM_MONETIZE).

**Rationale:** Prevents bypassing client gates (e.g. direct API or Payload REST). Plan is the single source; capability mapping lives in the Studio entitlements store; server only needs plan.

---

## Analytics and feature flags: PostHog

**Decision:** We use **PostHog** for (1) marketing analytics (page views, custom events e.g. Waitlist Signup), and (2) Studio feature flags. CopilotKit is behind `copilotkit-enabled` (default off) during migration to Assistant UI. Video and Strategy editors have been removed from the shell; no editor-specific flags for them remain. Dev and production use separate PostHog project keys (or the same project with environments) so flags can differ. Plan-based entitlements (free/pro) remain for paywall; release/rollout toggles use PostHog.

**Rationale:** Single provider for analytics and feature flags; no Plausible. Update this doc if we add another analytics or flag provider.

---

## DockLayout uses Dockview (docking + floating)

**Decision:** `DockLayout` is implemented with **Dockview** to restore Unreal-style docking, drag-to-reorder, and floating panels. Layout is persisted via **Zustand** (app-shell store `dockLayouts` keyed by layoutId) when used in controlled mode (`layoutJson` / `onLayoutChange` / `clearLayout`); otherwise it falls back to `localStorage['dockview-{layoutId}']`. Exposes a `resetLayout()` ref to recover lost panels.

**Rationale:** Dockview provides the desired editor UX (docked tabs, floating groups, drag-to-group). The Video editor is locked behind `studio.video.editor` until we re-enable and choose a timeline/editor stack.

---

## Panel visibility and layout only in View > Layout; Settings Panels tab for editor-exposed controls

**Decision:** **Panel visibility** toggles (Show/Hide Library, Inspector, etc.) and **layout actions** (Restore all panels, Reset layout) live **only** in the **View > Layout** submenu in each editor. They are **not** shown in Settings. The **Settings Panels tab** (App/User inner tab) is for **editor-exposed controls** (e.g. graph viewport: minimap, animated edges, layout algorithm). When the active editor has no such controls, the Panels tab shows a short empty state. **Fit view** and **Fit to selection** are not in the UI (removed from View menu, graph toolbar, and context menu); underlying `fitView` / `fitViewToNodes` may remain for programmatic use.

**Rationale:** Single place for layout UX; Panels tab meaningfully groups editor panel options (e.g. graph minimap) instead of duplicating View menu toggles. See [styling-and-ui-consistency.md](styling-and-ui-consistency.md) rule 20.

---

## Video editor removed from workspace

**Decision:** The **Video editor** is no longer a workspace tab. It has been removed from `EDITOR_IDS`, app-shell store, metadata, editor-panels, schema, and feature flags. The file `VideoEditor.tsx` remains in the repo but is not imported or rendered; it receives no updates. Chat and layout are provided by the right-rail Chat panel and other editors only.

**Rationale:** Align with MVP scope; Video is not in MVP. Removing it from the shell simplifies routing and persistence.

---

## Strategy tab removed; chat in right rail only

**Decision:** The **Strategy** editor tab has been removed. **Chat** is available only in the **right-rail Chat panel** (same `DialogueAssistantPanel` component) in Dialogue and Character editors. There is no separate Strategy tab or Cmd+K popup. `EDITOR_IDS` is `['dialogue', 'character']`; Strategy feature flag and metadata entries are removed.

**Rationale:** One Chat surface (right rail) per editor; no duplicate global popup or Strategy-only tab.

---

## Cmd+K and AssistantChatPopup removed

**Decision:** The **Mod+K** / **Ctrl+Shift+P** shortcut and **Help → Show Commands** have been removed. The **AssistantChatPopup** component has been deleted. Help menu shows **Welcome** and **About** only. Chat is available only in the right-rail Chat panel when an editor (Dialogue or Character) is active.

**Rationale:** Single Chat entrypoint (right rail) reduces complexity and matches the shared-panel pattern.

---

## Shared panel content and editor-scoped contributions

**Decision:** **Shared panel content** (e.g. Chat) is implemented **once** (e.g. `DialogueAssistantPanel`). Editors that want the panel add an **WorkspaceRailPanel** with a **stable id** (e.g. `CHAT_PANEL_ID` from `apps/studio/lib/editor-registry/constants.ts`) and the same content component. Visibility and View menu derive from the panel registry; schema keys follow `panel.visible.{editorId}-{panelId}`. **Editor-scoped contributions** (panels, menus, settings sections) follow the same pattern: declare under a provider, **register on mount**, **unregister on unmount**, keyed by editorId or scope. No separate "app-level" panel registry is required; each editor that wants Chat explicitly adds the panel. A future **declarative editor registration** (editors registered like panels/settings) can follow this pattern and be documented when added.

**Rationale:** One panel implementation, one id, predictable schema keys; tooling and layout can be scoped the same way (e.g. by editorId). Aligns with existing EditorLayoutProvider + WorkspaceRail + WorkspaceRailPanel + useEditorPanelVisibility flow.

---

## Studio as single entrypoint and unified registry pattern

**Decision:** **AppProviders** owns the full Studio shell (SidebarProvider, Settings sidebar, OpenSettingsSheetProvider, StudioMenubarProvider). **Studio** is content-only (tabs, editor content, sheets, toaster). The host renders `AppProviders` > `AppShell` > `Studio`. **Registries** (Zustand stores) are the single pattern for contributions: **editors** (editor registry, defaults on components, registered at module load), **menus** (menu registry by scope/context/target; WorkspaceMenubarContribution and UnifiedMenubar use it), **panels** (panel registry by editorId/rail), **settings** (settings registry by scope/scopeId). Place components in the tree under a scope provider; they register on mount and unregister on unmount; consumers (menubar, settings sidebar, dock, tabs) read from registries and filter by scope/context/target. **StudioApp** (alias for WorkspaceApp) is the tabs + content compound inside Studio; **WorkspaceApp** remains the shared export name for backward compatibility.

**Rationale:** One mental model for menus, settings, panels, and editors; host does not compose providers; canonical state stays in the app-shell store and Studio/editors consume it. See [Studio and unified registry refactor plan](.cursor/plans/studio_registry_refactor_2fb8702e.plan.md) and shared editor README.

---

## Settings: tree-as-source and generated contract

**Decision:** Settings are defined in the **React tree** (SettingsSection + SettingsField with a `default` prop). **SettingsSection** and **SettingsField** render the form (sidebar-as-source): SettingsField wraps the input (Input, Switch, Select, Textarea) and binds value/onChange to the store; the sidebar renders the same tree (AppSettingsRegistrations for App tab, GraphViewportSettings for Viewport tab). Sections still **register on mount** so the registry holds section/field metadata; codegen mounts the same tree, reads the registry, and writes generated modules (defaults per scope). The store and config import these artifacts; Payload continues to store settings as a single JSON field.

**Out of scope:** Menus and toolbars use the registry/tree pattern but do **not** persist a "contract" (no codegen). Rich collections (projects, forge-graphs) remain schema-first; types come from Payload config. **Only settings** use "tree as source → generate contract."

**Rationale:** Single place to add a setting (the tree); no duplicate schema; generated contract keeps store and API in sync; aligns with "build the frontend quickly" and third-party studios. See [Settings: tree-as-source and codegen](../../architecture/settings-tree-as-source-and-codegen.mdx).

---

## Viewport defaults overrides for graph settings

**Decision:** Viewport-scoped settings that need **different defaults per viewport** (e.g. `graph.allowedNodeTypes` for narrative vs storylet vs character) use **config overrides** in `apps/studio/lib/settings/config.ts`. `VIEWPORT_DEFAULTS_OVERRIDES` merges over generated `VIEWPORT_DEFAULTS`; `getViewportDefaults` applies overrides when resolving a given `editorId:viewportId`. The tree registers a single default (e.g. storylet case); config overrides narrative to `["PAGE"]`, storylet to `["CHARACTER","PLAYER","CONDITIONAL"]`, character to `["characterCard"]`.

**Rationale:** Codegen produces one default per field per viewport key; we cannot vary defaults per viewport without either per-viewport registration components or config overrides. Config overrides keep the tree simple and avoid duplicate GraphViewportSettings per viewport.

---

## Clone semantics (paid clone)

**Decision:** When a user **pays to clone** a project or template, they receive a **license** and can **clone again** (no per-clone fee after purchase). The **listing creator** chooses per listing: **indefinite** (purchaser always gets current project state on clone-again) or **version-only** (purchaser gets the same snapshot as at purchase). We persist a **license** record (user, listing, Stripe session id, grantedAt, optional snapshot) to enable clone-again and future audio/licenses.

**Rationale:** Aligns with Unity Asset Store / Bandlab style: buy once, use many times; creator choice supports both "always up to date" and "fixed release" offerings. See [MVP and first revenue](../../product/mvp-and-revenue.mdx) and [Listings and clones](../../business/listings-and-clones.mdx).

---

## Revenue model: platform and creator

**Decision:** **Both platform and creator get paid.** We use a Unity Asset Store / Bandlab style model: presets and packages; we take a cut; creators get payouts (e.g. Stripe Connect). **Catalog** at MVP supports **users and orgs** (browse and list).

**Rationale:** Incentivizes creators to list; platform revenue from take rate. Update this doc when we implement Connect, payouts, and catalog.

---

## Developer program: revenue split while official; publish updates

**Decision:** Approved editors in the **official Studio suite** receive a **revenue split** (not usage-based pay in Studio). The split continues **until the editor is removed** from the official suite; when removed, the split stops. Developers must be able to **publish updates** to their editor (process TBD: repo tag, build, or store update flow). Formula and process are **TBD**; document in [docs/business/revenue-and-stripe.mdx](../../business/revenue-and-stripe.mdx) and [Developer program and editor ecosystem](../../business/developer-program-and-editors.mdx) when defined.

**Rationale:** Rewards developers whose editors are in the official suite; split aligns with platform/clone revenue rather than raw usage. See [Developer program and editor ecosystem](../../business/developer-program-and-editors.mdx).

---

## MVP infrastructure

**Decision:** We run **Vercel + Payload** for MVP (Studio in the cloud). No self-host requirement for MVP.

**Rationale:** Keeps MVP deployment and ops simple; self-host can be considered after MVP.

---

## Platform app cutover

**Decision:** Customer-facing routes now live in **`apps/platform`** (landing, docs, catalog, login, account, billing, checkout, blog/waitlist/newsletter compatibility pages). `apps/marketing` is removed after parity checks. Platform keeps a separate style system (`apps/platform/src/styles/*`) and consumes Studio APIs through `NEXT_PUBLIC_STUDIO_APP_URL`.

**Rationale:** The Kiranism starter baseline gives a faster customer-facing foundation while keeping Studio/editor styles isolated and preserving existing auth/catalog/checkout contracts.

---

## Platform canonical IA: dashboard routes + creator APIs

**Decision:** Canonical creator/account routes are **`/dashboard/*`** (`overview`, `listings`, `games`, `revenue`, `billing`, `licenses`, `settings`, `api-keys`). Legacy `/account/*` and `/billing` are compatibility redirects only. Creator data is served from Studio APIs (`GET /api/me/listings`, `GET /api/me/projects`), and free listing clones use `POST /api/catalog/:id/clone` while paid clones continue through Stripe Checkout.

**Rationale:** Keeps platform IA consistent with a Vercel-style dashboard shell, prevents split-account navigation, and avoids duplicating logic between free and paid clone paths.

---

## Platform SaaS housekeeping conventions (client architecture)

**Decision:** Platform maintenance is standardized around a small set of client-side conventions:

- API calls are decomposed by domain under `apps/platform/src/lib/api/*` and use a shared request client (`client.ts`); components do not use raw `fetch`.
- Dashboard auth redirect logic is centralized in one layout gate (`RequireDashboardAuth`) instead of repeated per-page `useEffect` redirects.
- Platform route/query/auth strings come from constants modules (`lib/constants/routes.ts`, `lib/constants/query-keys.ts`, `lib/constants/auth.ts`) instead of magic literals.
- Platform quality scripts are part of root lint flow (`css:doctor`, `hydration:doctor`, `docs:runtime:doctor`, platform lint, studio lint).
- Template cleanup is done in reviewable slices (remove dead starter files/deps, keep compatibility redirects until explicit removal).

**Rationale:** This keeps the SaaS app understandable and reusable without forcing Studio/editor architecture onto platform pages, and prevents drift from repeated ad-hoc patterns.

---

## Catalog and listings API

**Decision:** The **public catalog** is read-only for platform browsing: Studio exposes **GET /api/catalog** (published listings only; no auth). Payload REST handles **/api/listings** for create/update/delete (authenticated; Create listing UI is gated by PLATFORM_LIST). Payload collection `listings` holds title, slug, description, listingType, project, price, currency, creator, thumbnail, category, status, plus clone/play metadata. Platform calls GET /api/catalog through its Studio client module; Studio uses Payload SDK for listing CRUD.

**Rationale:** Platform cannot use Payload SDK directly (separate app + server boundary); a single public route keeps the contract clear and allows filtering to published only.

---

## Creators list from day one

**Decision:** Catalog at MVP includes **creator listings from day one**. Stripe Connect (or similar) and payouts are required at MVP so both platform and creators get paid.

**Rationale:** Creators need to list and get paid from launch; no "creator catalog later" phase for MVP.

---

## Payments and checkout (Stripe hosted)

**Decision:** Use **Stripe hosted Checkout** for one-time clone purchases (and keep existing subscription Checkout for Pro). No custom payment UI. Use **Stripe for invoicing** where needed. Align with Cursor-style defaults (create session, redirect to Stripe, success/cancel URLs).

**Rationale:** Reduces PCI and UX risk; Stripe handles payment form and 3DS. See [Revenue and Stripe](../../business/revenue-and-stripe.mdx) and platform monetization task breakdown.

---

## Stripe Connect (day one)

**Decision:** Use **Stripe Connect from day one** for clone purchases. Payments go to the creator org's connected account; platform takes an application fee. Creators must complete **Connect onboarding** (Express or Standard) before they can receive payouts; we store Connect account identity on **organizations** (`stripeConnectAccountId`, onboarding status fields), with org-scoped create-account and onboarding-link APIs.

**Rationale:** Ensures creators get paid directly from launch. Document onboarding flow in business docs and task breakdown. See [Revenue and Stripe](../../business/revenue-and-stripe.mdx).

---

## Clone implementation (full project, media as references)

**Decision:** Clone = **full project copy**: project row, forge-graphs, characters, relationships, pages, blocks, settings. **Media:** do not duplicate files; store **references** to the source project's media. When the **clone owner** replaces or deletes (e.g. in Character), upload new media and remove the reference to the old media. Optional: mark media in the cloned project as "reference" so UI can show "from original project" until replaced.

**Rationale:** Keeps storage and clone cost low; clone owner gains full control when they edit. See [Listings and clones](../../business/listings-and-clones.mdx).

---

## License record (clone-again and future licenses)

**Decision:** Persist a **license** when a user pays to clone (e.g. Payload collection `licenses`: user, listing, stripeSessionId or paymentIntentId, grantedAt, optional versionSnapshotId). Enables clone-again (API and UI) and future audio/generated-content licenses. Webhook on successful payment creates the license and triggers first clone (or queue).

**Rationale:** Single record for "right to clone" and future license types. See [Listings and clones](../../business/listings-and-clones.mdx) and task breakdown.

---

## Listing versioning (clone mode)

**Decision:** Add a field to listings (e.g. `cloneMode: 'indefinite' | 'version-only'`). The **creator** sets it when creating or editing a listing. Backend uses it when handling clone-again: **indefinite** = clone current project state; **version-only** = clone the same snapshot as at first purchase.

**Rationale:** Creator choice per listing; supports both "always latest" and "fixed release" use cases. See [Listings and clones](../../business/listings-and-clones.mdx).

---

## Payouts: single seller now; splits later

**Decision:** **Single seller per listing for now.** Payment goes to the listing creator's Connect account; platform takes an application fee. No split payouts. Splits between multiple sellers (music-industry style) and IP/licensing (e.g. Beatstars-style) are to be figured out later when we have more features and creatable content that could justify splits.

**Rationale:** Keeps checkout and payouts simple at launch. See [Revenue and Stripe](../../business/revenue-and-stripe.mdx).

---

## MVP ordering: Yarn, GamePlayer, publish/host in MVP; MCP Apps after MVP

**Decision:** **First-class Yarn Spinner** (export/import `.yarn`) and **GamePlayer** (playable runtime for Yarn Games) are **part of MVP**. **Publish and host playable builds** (build pipeline, storage, playable URL) is **part of MVP** so that players can play builds; do it alongside or after GamePlayer/Yarn as needed. We do these before or alongside platform monetization. **Editors as MCP Apps** (McpAppDescriptor, Studio MCP Server) is **after MVP** — we do not block MVP work on MCP.

**Rationale:** MVP success = first paid clone E2E and playable builds; that requires a playable game (Dialogue + Character + Writer + GamePlayer), Yarn export, and the ability to publish/host so others can play. MCP is valuable for post-MVP embedding in hosts (Cursor, Claude Desktop, etc.).

---

## What to do next: prefer MVP-critical; Video when unlocked

**Decision:** When picking work from STATUS § Next, **prefer MVP-critical items** (Yarn Spinner, GamePlayer, plans/capabilities for platform, monetization). **Video** work (persistence, workflow panel, timeline/editor stack) is done **when the Video editor is unlocked** (Video is not in MVP). The default "next slice" is MVP-critical (e.g. Yarn export/import or GamePlayer first slice), not Video.

**Rationale:** Aligns agent and contributor effort with MVP success criterion and avoids spending time on Video until we re-enable it.

---

## Declarative editor components and registries

**Decision:** Editors use **declarative components** that **register** into shared registries; layout, View menu, Settings UI, and menubar **subscribe** to those registries. Authors do not hand-wire `rightPanels`, `EDITOR_PANEL_SPECS`, static settings sections, or `useAppMenubarContribution(menus)`.

- **Panel layout:** **UI-first** (EditorDockLayout.Left/Main/Right/Bottom with EditorDockLayout.Panel children). Editors compose panels directly in JSX; no store-driven registration. **EditorLayoutProvider** provides `{ editorId }` only; **EditorLayout**, **WorkspaceRail**, **WorkspaceRailPanel** deprecated. View menu and `useEditorPanelVisibility` use **EDITOR_PANEL_SPECS** for panel toggles and "Restore all panels". Layout reset calls `ref.current?.resetLayout()` on EditorDockLayout.
- **Settings registry:** Keyed by scope + scopeId (app, editor, viewport). **AppSettingsProvider** / **ViewportSettingsProvider** provide scope; **SettingsSection** + **SettingsField** (Studio) register on mount. **AppSettingsPanelContent** merges registry sections over static sections.
- **Menubar:** **WorkspaceMenubarContribution** (Studio) with **WorkspaceMenubarMenuSlot** children builds the menu array and calls setEditorMenus on mount; clears on unmount. Must be used inside **EditorLayoutProvider** so switching editors resets the menubar.
- **FormField-within-Form style:** SettingsSection, WorkspaceMenubarContribution only work inside the correct provider; document in README and AGENTS.

**Rationale:** Single source of truth for panels, settings sections, and menubar; no duplicated panel/section lists; View menu and layout stay in sync; new editors add declarative components instead of editing global config.

<!-- forge-loop:generated:start -->
## Forge Loop Snapshot

# Decisions

No decisions recorded yet.
<!-- forge-loop:generated:end -->
