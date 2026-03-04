---
title: Task breakdown — Platform monetization (clone/download)
created: 2026-02-08
updated: 2026-02-09
---

# Platform: monetization (clone/download)

**Initiative id:** `platform-mono` (Tier 0)

**Parent:** [STATUS § Next #4](./STATUS.md) · [MVP and first revenue](../../product/mvp-and-revenue.mdx)

Clone to user/org for a price; or download build/template/Strategy core. Listings, checkout, Stripe Connect or similar. First revenue = first paid clone end-to-end.

---

## Lanes and tasks

### Lane: Plan/capabilities (Tier 1)

Extend `user.plan` and `CAPABILITIES` to gate platform features (who can list, who can clone/download paid).

| id | title | parent | tier | impact | status | doc |
|----|-------|--------|------|--------|--------|-----|
| platform-mono-cap-1 | Add PLATFORM_LIST capability and gate listing UI | platform-mono | 3 | Small | done | [entitlements](../../packages/shared/src/shared/entitlements) |
| platform-mono-cap-2 | Add PLATFORM_MONETIZE to CAPABILITIES and plan check | platform-mono | 3 | Small | done | [decisions](./decisions.md) |
| platform-mono-cap-3 | Wire plan to entitlements store for platform gates | platform-mono | 3 | Small | done | — |
| platform-mono-cap-4 | Gate PLATFORM_PUBLISH in CreateListingSheet (status Published) | platform-mono | 3 | Small | done | CreateListingSheet |
| platform-mono-cap-5 | Gate PLATFORM_MONETIZE in CreateListingSheet (price > 0) | platform-mono | 3 | Small | done | CreateListingSheet |
| platform-mono-cap-6 | Server-side: listings beforeChange check plan for publish/price | platform-mono | 3 | Small | done | [decisions](./decisions.md) |

### Lane: Listings (Tier 1)

Catalog and creator listings; create/update/delete listing.

| id | title | parent | tier | impact | status | doc |
|----|-------|--------|------|--------|--------|-----|
| platform-mono-list-1 | Listings API: Payload collection + GET /api/listings (published only) | platform-mono | 2 | Medium | done | — |
| platform-mono-list-2 | Listings UI: catalog page (public grid at /catalog) | platform-mono | 2 | Medium | done | — |
| platform-mono-list-2b | Create listing flow (Studio or marketing account) | platform-mono | 2 | Medium | done | — |
| platform-mono-list-3 | Listing clone mode: add `cloneMode` (indefinite \| version-only) to schema and Create/Edit UI | platform-mono | 2 | Small | done | Create sheet + Payload admin; webhook sets versionSnapshotId when version-only; clone-again uses it |

### Lane: Stripe Connect (Tier 1)

Connect account, onboarding, and checkout session for listing (payment to creator + platform fee).

| id | title | parent | tier | impact | status | doc |
|----|-------|--------|------|--------|--------|-----|
| platform-mono-pay-1a | Connect account: create Connect account (Express/Standard), store `stripeAccountId` (users or creator-accounts) | platform-mono | 2 | Medium | done | [revenue-and-stripe](../../business/revenue-and-stripe.mdx) |
| platform-mono-pay-1b | Connect onboarding link: API + UI for creators to complete onboarding; handle return and account status | platform-mono | 2 | Medium | done | — |
| platform-mono-pay-1c | Checkout session for listing: one-time payment with Connect (application_fee); metadata `listingId`, `buyerId`; success URL | platform-mono | 2 | Medium | done | Implemented in connect/create-checkout-session route (check-1) |

### Lane: Checkout (Tier 1)

Clone-purchase session and redirects (Stripe hosted Checkout).

| id | title | parent | tier | impact | status | doc |
|----|-------|--------|------|--------|--------|-----|
| platform-mono-check-1 | Checkout session API for clone purchase: auth; body `listingId`; create Stripe Checkout (mode payment) with Connect, platform fee; metadata for webhook; return session URL | platform-mono | 2 | Medium | done | [revenue-and-stripe](../../business/revenue-and-stripe.mdx) |
| platform-mono-check-2 | Checkout UI and redirects: catalog/detail "Get" calls check-1, redirect to Stripe; success/cancel URLs; success page "Open in Studio" / "Clone again" (license + first clone via webhook) | platform-mono | 2 | Medium | done | — |

### Lane: Licenses (Tier 1)

License record and clone-again.

| id | title | parent | tier | impact | status | doc |
|----|-------|--------|------|--------|--------|-----|
| platform-mono-license-1 | License model: Payload collection `licenses` (user, listing, stripeSessionId/paymentIntentId, grantedAt, optional snapshot). Webhook on payment creates license and triggers first clone (or queue) | platform-mono | 2 | Medium | done | [listings-and-clones](../../business/listings-and-clones.mdx) |
| platform-mono-license-2 | Clone-again: API "clone again" for a given license (respects listing indefinite vs version-only); UI to list user's licenses and trigger clone again | platform-mono | 2 | Medium | done | — |

### Lane: Clone flow (Tier 1)

Clone-to-account and post-purchase UI.

| id | title | parent | tier | impact | status | doc |
|----|-------|--------|------|--------|--------|-----|
| platform-mono-clone-1 | Clone-to-account API: input project id (or listing + version), target user. Copy project, forge-graphs, characters, relationships, pages, blocks; media as references; on clone-owner replace/delete upload new media and drop old reference. Return new project id | platform-mono | 2 | Medium | done | [listings-and-clones](../../business/listings-and-clones.mdx) |
| platform-mono-clone-2 | Clone confirmation and post-purchase UI: success page after payment; "Open in Studio" link; "Clone again" in Studio or account (lists licenses, calls clone-1 per license/listing rules) | platform-mono | 2 | Medium | done | — |

### Lane: Payouts (Tier 1)

Revenue-share tracking (Connect handles payouts to creators).

| id | title | parent | tier | impact | status | doc |
|----|-------|--------|------|--------|--------|-----|
| platform-mono-pay-2 | Payout and revenue-share tracking | platform-mono | 2 | Medium | done | amountCents/platformFeeCents on licenses; webhook populates; GET /api/me/revenue for creators |

---

## Reference

- **Initiative:** [STATUS § Next #4](STATUS.md) · [product roadmap Platform (future)](../../roadmap/product.mdx)
- **Process:** [task-breakdown-system.md](./task-breakdown-system.md) · [task-registry.md](./task-registry.md)
