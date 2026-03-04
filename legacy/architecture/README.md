---
title: Architecture map
created: 2026-02-09
updated: 2026-02-11
---

# Architecture map

**Start here** for a quick mental model. One place for: packages/apps overview, key data flows, where state lives, API boundaries. For depth, use the linked docs.

---

## Apps and packages overview

| Layer | Path | Purpose |
|-------|------|---------|
| **Studio** | `apps/studio/` | Next.js app: App Shell, editors (Dialogue, Character, Video, Strategy), CopilotKit, Payload config, API routes. |
| **Platform** | `apps/platform/` | Customer-facing app: landing/docs/login plus app-shell routes (`/catalog`, `/dashboard/*`) for creator/account/commerce workflows. |
| **Shared (editor kit)** | `packages/shared/src/shared/` | WorkspaceShell, DockLayout, editor slots, headless contracts (Selection, OverlaySpec, capabilities). Consumed as `@forge/shared`. |
| **UI atoms** | `packages/ui/` | shadcn primitives; `@forge/ui`. |
| **Types** | `packages/types/` | Payload-generated `payload-types.ts`; domain aliases (`payload.ts`, `graph.ts`). |
| **Agent engine** | `packages/agent-engine/` | Workflow runtime (steps, events, SSE). |
| **Domain (Dialogue)** | `packages/domain-forge/` | Forge domain logic, copilot wiring, plan-execute-review-commit. |
| **Domain (Character)** | `packages/domain-character/` | Character domain. |
| **Dev kit** | `packages/dev-kit/` | Re-exports `@forge/ui`, `@forge/shared`, `@forge/agent-engine` for consumers. |

---

## Key data flows

1. **App Shell → active editor**  
   AppShell owns `activeWorkspaceId`, `openWorkspaceIds`, `activeProjectId`. Only the active editor is mounted. Project context is app-level; Dialogue and Character read `activeProjectId` from the shell.

2. **Server state**  
   TanStack Query + Payload SDK (collection CRUD) and generated/manual API client (auth, settings, AI, workflows). Hooks in `apps/studio/lib/data/hooks/`; keys in `lib/data/keys.ts`. No raw `fetch` in components/stores.

3. **Drafts and UI route**  
   Zustand stores (graph, video, app-shell). Drafts are client-owned until Save; then mutation + query invalidation. App-shell route and last-doc ids persisted via Zustand `persist`.

4. **AI**  
   CopilotKit at shell + per-editor domain contract (`useDomainCopilot`). OpenRouter via OpenAI SDK + baseURL; ElevenLabs for audio. Model list from API; preferences in model router.

---

## Where state lives

| State | Location | Persistence |
|-------|----------|-------------|
| Server (graphs, video-docs, users, settings) | TanStack Query cache | Fetched via Payload SDK / API routes |
| Draft (current graph/video doc) | Zustand (graph store, video store) | Zustand persist (partialize when dirty) |
| App shell (route, open tabs, active project, last doc ids) | `apps/studio/lib/app-shell/store.ts` | Zustand persist `forge:app-session:v2` |
| Layout (Dockview) | App shell store `dockLayouts` | Zustand persist (same session key); DockLayout controlled mode |
| Settings overrides | Payload `settings-overrides` | GET/POST `/api/settings` |
| Entitlements / plan | Payload user.plan + overrides | `/api/me`, EntitlementsProvider |

See [decisions](../agent-artifacts/core/decisions.md) and [11-tech-stack](../11-tech-stack.mdx) for rationale.

---

## API boundaries

- **Client → server:** One boundary. Client talks only to **Next API routes** and Payload REST (via Payload SDK for collection CRUD). No direct Payload GraphQL from the browser.
- **Platform → Studio auth/data:** Credentialed CORS boundary is centralized in Studio (`apps/studio/lib/server/cors.ts`, middleware, payload cors/csrf config).
- **Collection CRUD:** Payload SDK in `lib/api-client/payload-sdk.ts`; hooks use it.
- **App-specific:** Generated client (`lib/api-client/services/`) where it exists (Auth, Settings, Model, AI); **manual** client modules in `lib/api-client/` (elevenlabs, media, workflows). Vendor SDKs in **route handlers** (e.g. ElevenLabs server-side). Do not add raw `fetch` for `/api/*` in components—extend the client.
- **OpenAPI/Swagger:** Documentation only (no streaming); do not add endpoints to the spec to get a client.

See [decisions](../agent-artifacts/core/decisions.md) and [11-tech-stack](../11-tech-stack.mdx).

---

## Links (most relevant)

| Topic | Doc |
|-------|-----|
| **Architecture index** | [00-index.mdx](00-index.mdx) |
| **Unified editor / App Shell** | [01-unified-workspace.mdx](01-unified-workspace.mdx) |
| **Editor architecture (slots, selection, inspector)** | [02-workspace-editor-architecture.mdx](02-workspace-editor-architecture.mdx) |
| **Editor platform (DockLayout)** | [05-editor-platform.mdx](05-editor-platform.mdx) |
| **CopilotKit and agents** | [03-copilotkit-and-agents.mdx](03-copilotkit-and-agents.mdx) |
| **Model routing and OpenRouter** | [06-model-routing-and-openrouter.mdx](06-model-routing-and-openrouter.mdx) |
| **Tech stack and "where things live"** | [11-tech-stack.mdx](../11-tech-stack.mdx) |
| **Decisions (ADR)** | [agent-artifacts/core/decisions.md](../agent-artifacts/core/decisions.md) |
| **Settings: tree-as-source and codegen** | [settings-tree-as-source-and-codegen.mdx](settings-tree-as-source-and-codegen.mdx) |
| **Project overview** | [10-project-overview.mdx](../10-project-overview.mdx) |
