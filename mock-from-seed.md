---
title: Mock data sync (docs playgrounds)
---

# Mock data sync (docs playgrounds)

Docs playgrounds use mock data from `apps/studio/lib/docs/mock-data.ts` so editors can render without a backend.

**Source of truth:** `apps/studio/payload/seed.ts`

When seed changes (new demo flow, project fields, character shape, etc.):

1. Update `mock-data.ts` to match.
2. Run docs and verify playgrounds still render.

**Optional:** Add `pnpm docs:mock:sync` script that reads seed output and writes mock-data (future).
