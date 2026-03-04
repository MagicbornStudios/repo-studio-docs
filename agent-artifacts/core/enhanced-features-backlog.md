---
title: Enhanced features backlog
created: 2026-02-11
updated: 2026-02-11
---

# Enhanced features / ideas backlog

Living log: agents (or humans) propose ideas here; humans triage; we implement when status is `accepted`. **Do not implement** items with status `proposed` until a human sets status to `accepted`.

## Process

1. **Agent (or human)** adds an entry below with status `proposed`.
2. **Human** reviews backlog; sets status to `accepted` or `rejected`; optionally assigns priority or adds to STATUS § Next.
3. When a backlog item is set to **accepted**, add it to [STATUS.md](./STATUS.md) § Next with an **impact size** (Small / Medium / Large / Epic).
4. **Agent or human** implements when status is `accepted`; sets status to `implemented` and adds optional Link to PR/slice.

Agents must not implement proposed items until a human sets `accepted`.

## Format per entry

- **Title** — Short name.
- **Context** — Where/when (e.g. "Dialogue editor", "Settings panel").
- **Suggestion** — What to do (library, UX, DX).
- **Status** — `proposed` | `accepted` | `rejected` | `implemented`.
- **Date** — YYYY-MM-DD.
- **Link** — Optional; PR or slice reference when implemented.

---

## Business / operations backlog (to figure out)

Items to figure out later (not product features; legal, ops, strategy). Not implemented in pay-1a or immediate slices.

- Legal (incorporation, terms, privacy, IP).
- Incorporation, shares, equity.
- Fundraising strategy and tools; bootstrapping.
- Revenue and projections.
- Marketing campaigns and ideas.
- How to use/install tech and keep it lean with coding agents.
- Offerings centered around the core product and platform.
- Developer program: usage metrics and complexity formula for approved-editor payouts.
- App store for community/official editors (listings, install, apply for official).
- GitHub fork and private-repo submission flow for paid users.

---

## Entries

(Add new entries at the top; append-only.)

### Dialogue graph state as JSON (Inspector or panel)
- **Title** — Dialogue graph state as JSON (Inspector or panel).
- **Context** — Dialogue editor; two graph panels (narrative, storylet). Planning workflow for agents; accurate graphs.
- **Suggestion** — Expose a clean JSON view of the dialogue graph state in the Inspector or a dedicated panel. Optionally use Monaco (or a simple text area) so users can edit and copy-paste graph data. Make this optional per editor (some editors have it, others don't).
- **Plausibility / rationale** — High value for power users and agents; JSON is the natural serialization of graph state. Two graphs (narrative + storylet) imply either one combined state object or two keys; needs a clear representation.
- **Status** — proposed.
- **Date** — 2026-02-10.

### Graph state editor: IDs and validation
- **Title** — Graph state editor: IDs and validation.
- **Context** — Same as above; agent planning workflow.
- **Suggestion** — When showing/editing graph state as JSON, maintain stable IDs and validate structure (nodes, edges, required fields). Document ID semantics so agents and users can produce valid graphs.
- **Plausibility / rationale** — Prevents broken graphs and supports relinking and cross-references.
- **Status** — proposed.
- **Date** — 2026-02-10.

### Relinking and graph updates
- **Title** — Relinking and graph updates.
- **Context** — Editing graph state (e.g. paste new graph, or bulk delete).
- **Suggestion** — Define behavior when IDs change or nodes/edges are removed: optional "relinking" (map old IDs to new, or place/delete nodes and edges that existed in the previous graph). If user deletes nodes/edges that are referenced elsewhere (e.g. custom runtime directives for video / Canav), warn and offer to "delete all" associated data.
- **Plausibility / rationale** — Enables safe paste and refactors; cross-editor references (e.g. video directives pointing at dialogue nodes) require clear warnings and delete semantics.
- **Status** — proposed.
- **Date** — 2026-02-10.

### Cross-editor references and delete warnings
- **Title** — Cross-editor references and delete warnings.
- **Context** — Studio; dialogue graphs and other editors (e.g. video, custom runtimes).
- **Suggestion** — When graph entities (nodes/edges) are referenced by another editor or feature (e.g. video Canav directives), detect those references and on delete show a warning; allow user to delete the graph entity and all associated references (cascade) or cancel.
- **Plausibility / rationale** — Avoids orphaned references and data inconsistency.
- **Status** — proposed.
- **Date** — 2026-02-10.

### Usage-based payouts for approved editors
- **Title** — Usage-based payouts for approved editors.
- **Context** — Developer program; Stripe Connect.
- **Suggestion** — Measure usage of each approved third-party editor; assign value by usage and complexity; recurring payouts via Connect. See docs/business/developer-program-and-editors.mdx and revenue-and-stripe.mdx.
- **Status** — proposed.
- **Date** — 2026-02-09.

### Approved editor data contracts
- **Title** — Approved editor data contracts.
- **Context** — Developer program; platform contract.
- **Suggestion** — Define schemas/contracts for data produced and consumed by approved editors; third-party approved editors can produce new data types and consume from installed approved editors.
- **Status** — proposed.
- **Date** — 2026-02-09.

### Community editor listings
- **Title** — Community editor listings.
- **Context** — Developer program; app store.
- **Suggestion** — Allow developers to list community (unofficial) editors; users can install; no payouts; clone applies to projects using community editors.
- **Status** — proposed.
- **Date** — 2026-02-09.

### Developer submission (GitHub fork for paid)
- **Title** — Developer submission (GitHub fork for paid).
- **Context** — Developer program; submission.
- **Suggestion** — App submission = GitHub repo (private); we fork (or mirror) for paid users so they get the app; developer can list in community and/or apply for official integration.
- **Status** — proposed.
- **Date** — 2026-02-09.

### Testimonials section
- **Title** — Testimonials section.
- **Context** — Marketing landing page.
- **Suggestion** — Add a landing section for quotes/case studies. STATUS Part B explicitly deferred testimonials; backlog tracks as future.
- **Status** — rejected.
- **Note** — Not desired per product decision.
- **Date** — 2026-02-07.

### Marketing analytics
- **Title** — Marketing analytics.
- **Context** — Platform app (landing, pricing, signup funnel).
- **Suggestion** — Event tracking or integration (e.g. Plausible, Posthog) for landing, pricing, and signup funnel.
- **Effort** — Small. Add script or SDK in root layout; optional custom events on waitlist, pricing, primary CTA. Env var for key/domain; no backend.
- **Status** — implemented.
- **Date** — 2026-02-07.
- **Link** — PostHog in `apps/platform` (`src/lib/analytics.ts` trackEvent; waitlist signup event on success). Studio uses PostHog for feature flags (e.g. video-editor-enabled).

### CTA and copy A/B
- **Title** — CTA and copy A/B.
- **Context** — Marketing Hero and key copy.
- **Suggestion** — Document or tooling for testing Hero CTAs and key copy (A/B or variants).
- **Status** — proposed.
- **Date** — 2026-02-07.
