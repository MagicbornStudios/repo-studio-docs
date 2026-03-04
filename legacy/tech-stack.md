---
title: Tech stack
created: 2026-02-04
updated: 2026-02-07
---

# Tech stack and "why we use it" / "why we don't"

Reference for agents and humans. Update when adding or removing major choices.

---

## Stack (Studio)

| Area | Choice | Why we use it |
|------|--------|----------------|
| Framework | Next.js (App Router) | SSR, API routes, single deployment. |
| Server CMS/DB | Payload + SQLite | Type-safe schema, local-first, REST/GraphQL available server-side only. |
| Client server-state | TanStack Query (React Query) | Caching, deduping, loading/error, invalidation; one data layer for all server-state. |
| Client UI/draft state | Zustand | Lightweight, works with React; used for drafts and app shell route. |
| Persistence (client) | localStorage (versioned keys) | App shell route and last document ids; no backend required. |
| AI/chat | CopilotKit + OpenRouter | Agent UX and model routing. |
| Styling | Tailwind + shadcn (packages/ui) | Consistent design system. |
| Shared editor kit | packages/shared (WorkspaceShell, DockLayout) | Single layout/contract for Dialogue, Video, Character, Strategy. |

---

## Why we don't

- **No raw fetch for server-state.** Collection CRUD goes through the Payload SDK; app-specific ops (auth, settings, AI, SSE) go through our custom routes and generated or manual client. TanStack Query hooks use these; components and stores do not call fetch directly.
- **No "just Zustand + fetch" for server-state.** We'd reimplement caching identity, invalidation, loading/error, and retries; TanStack Query provides this in one place.
- **No multiple DBs or backends in the client contract yet.** We use one Payload schema/DB; when we split, the client still talks to Payload REST and our Next API routes.

---

## Where things live

- **Payload SDK (collection CRUD):** `apps/studio/lib/api-client/payload-sdk.ts`; hooks use it for forge-graphs and video-docs.
- **Custom endpoints / SSE:** OpenAPI/Swagger spec is for documentation only. Use generated services in `lib/api-client/services/` (Auth, Settings, Model, AI) where present; manual client modules (elevenlabs, media, workflows); or vendor SDKs. Do not extend the spec for new clients - add manual modules or use vendor SDKs. `lib/api-client/workflows.ts` for SSE.
- **Query keys and hooks:** `apps/studio/lib/data/keys.ts`, `apps/studio/lib/data/hooks/`
- **Persisted client state:** Zustand persist on app-shell store (`apps/studio/lib/app-shell/store.ts`) and graph/video stores (draft partialize); see **docs/agent-artifacts/core/decisions.md**.
- **Decisions and rationale:** **docs/agent-artifacts/core/decisions.md**
