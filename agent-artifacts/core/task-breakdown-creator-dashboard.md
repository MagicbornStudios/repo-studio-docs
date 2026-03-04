---
title: Task breakdown - Creator dashboard (platform) and Studio minimal platform
created: 2026-02-09
updated: 2026-02-11
---

# Creator dashboard (platform) and Studio minimal platform

**Initiative id:** `creator-dashboard` (Tier 0)

**Parent:** [STATUS](STATUS.md) - [Product roadmap Platform](../../roadmap/product.mdx) - [ISSUES: Creator dashboard](../../../ISSUES.md)

Full account management in the platform app (`/dashboard/*`) with Studio staying minimal (publish/update listing in app bar only).

---

## Lanes and tasks

### Lane: Template evaluation and adoption

| id | title | parent | tier | impact | status | doc |
|----|-------|--------|------|--------|--------|-----|
| creator-dash-template-1 | Evaluate shadcn dashboard templates (Kiranism vs Vercel Studio Admin); document choice and migration steps (auth, layout, tables) | creator-dashboard | 3 | Small | done | `apps/platform` cutover notes |
| creator-dash-template-2 | Adopt chosen template for platform dashboard shell and replace starter mock pages | creator-dashboard | 2 | Medium | done | `apps/platform/src/components/platform/PlatformDashboardShell.tsx` |

### Lane: API for creator data

| id | title | parent | tier | impact | status | doc |
|----|-------|--------|------|--------|--------|-----|
| creator-dash-api-1 | `GET /api/me/listings` for creator listing management data | creator-dashboard | 3 | Small | done | `apps/studio/app/api/me/listings/route.ts` |
| creator-dash-api-2 | `GET /api/me/projects` for creator game/project data | creator-dashboard | 3 | Small | done | `apps/studio/app/api/me/projects/route.ts` |
| creator-dash-api-3 | `POST /api/catalog/:id/clone` for free listing clone flow | creator-dashboard | 3 | Small | done | `apps/studio/app/api/catalog/[id]/clone/route.ts` |

### Lane: Org context and financial visibility

| id | title | parent | tier | impact | status | doc |
|----|-------|--------|------|--------|--------|-----|
| creator-dash-org-1 | Add organizations + memberships collections and user default org linkage | creator-dashboard | 2 | Medium | done | `apps/studio/payload/collections/organizations.ts`, `organization-memberships.ts`, `users.ts` |
| creator-dash-org-2 | Add org APIs (`GET /api/me/orgs`, `POST /api/me/orgs/active`) and bootstrap personal org on auth | creator-dashboard | 2 | Medium | done | `apps/studio/app/api/me/orgs/*`, `apps/studio/lib/server/organizations.ts` |
| creator-dash-org-3 | Add platform org switcher and org-aware auth/query invalidation behavior | creator-dashboard | 2 | Medium | done | `apps/platform/src/components/layout/org-switcher.tsx`, `apps/platform/src/components/auth/AuthProvider.tsx` |
| creator-dash-fin-1 | Add AI usage ledger collection and org-scoped usage APIs (`/api/me/ai-usage*`) | creator-dashboard | 2 | Medium | done | `apps/studio/payload/collections/ai-usage-events.ts`, `apps/studio/app/api/me/ai-usage*` |
| creator-dash-fin-2 | Instrument AI routes to persist usage/cost events for dashboard reporting | creator-dashboard | 2 | Medium | done | `apps/studio/app/api/assistant-chat/route.ts`, `forge/plan`, `structured-output`, `image-generate` |
| creator-dash-fin-3 | Extend Stripe Connect create-account/onboarding APIs to org context and platform billing UX | creator-dashboard | 2 | Medium | done | `apps/studio/app/api/stripe/connect/*`, `apps/platform/src/app/dashboard/billing/page.tsx` |

### Lane: Dashboard pages (platform)

| id | title | parent | tier | impact | status | doc |
|----|-------|--------|------|--------|--------|-----|
| creator-dash-pages-1 | My listings page with filters/actions | creator-dashboard | 2 | Medium | done | `apps/platform/src/app/dashboard/listings/page.tsx` |
| creator-dash-pages-2 | My games page with project/listing linkage | creator-dashboard | 2 | Medium | done | `apps/platform/src/app/dashboard/games/page.tsx` |
| creator-dash-pages-3 | Revenue page consuming `GET /api/me/revenue` | creator-dashboard | 3 | Small | done | `apps/platform/src/app/dashboard/revenue/page.tsx` |
| creator-dash-pages-4 | Billing/licenses/settings/api-keys canonical under `/dashboard/*` with `/account/*` redirects | creator-dashboard | 2 | Medium | done | `apps/platform/src/app/dashboard/*`, `apps/platform/src/app/(platform)/account/*` |

### Lane: Catalog and checkout UX

| id | title | parent | tier | impact | status | doc |
|----|-------|--------|------|--------|--------|-----|
| creator-dash-catalog-1 | Unity-style catalog filters/search/sort and playable toggle | creator-dashboard | 2 | Medium | done | `apps/platform/src/components/catalog/catalog-marketplace.tsx` |
| creator-dash-catalog-2 | Catalog detail actions for `Play` + `Clone/Get` with clone mode metadata | creator-dashboard | 2 | Medium | done | `apps/platform/src/app/(appshell)/catalog/[slug]/page.tsx` |
| creator-dash-catalog-3 | Success/cancel flow for free clone vs paid checkout | creator-dashboard | 3 | Small | done | `apps/platform/src/app/(appshell)/checkout/success/page.tsx` |

### Lane: Follow-ups

| id | title | parent | tier | impact | status | doc |
|----|-------|--------|------|--------|--------|-----|
| creator-dash-followup-1 | Full build publish/host pipeline for playable builds (artifact hosting, runtime URLs, publishing workflow) | creator-dashboard | 1 | Large | open | `publish-host` initiative |

---

## Reference

- **Initiative:** [task-registry](task-registry.md) - [Product roadmap](../../roadmap/product.mdx) - [ISSUES](../../../ISSUES.md)
- **Process:** [task-breakdown-system.md](task-breakdown-system.md) - [task-registry.md](task-registry.md)
