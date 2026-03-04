---
title: Documentation building guidelines
---

# Documentation building guidelines

For coding agents and humans. How we structure docs and what to update when changing code.

## Page length

- **Major docs:** 300+ lines guideline (building-studio, AI how-tos, showcase sections).
- **Minimum:** No doc under 50 lines. If content is thin, add structure, icons, links to catalog.
- **Anemic sections:** Add QuickNav, tables, or ShowcaseLink to reach minimum.

## QuickNav and ShowcaseLink

- **QuickNav:** Add to pages with 4+ H2 sections. Use `export const quickNavItems = [...]` in the MDX body, then `<QuickNav items={quickNavItems} />`. Avoid inline `items={[...]}` — can cause "Could not parse expression with acorn" in fumadocs-mdx.
- **ShowcaseLink:** Design docs use it to link to Catalog. Catalog is the single source for demos. Do not duplicate catalog demos in Design.
- **Heading IDs:** Use `[#slug]` (square brackets) for custom heading IDs. `{#slug}` (curly braces) causes acorn parse error. See errors-and-attempts § Heading slug syntax.

## Doc structure

1. **Onboarding** — Install, env, first-run expectations. `docs/onboarding/`
2. **Showcase (Catalog)** — Component reference with demos. All components, mock data. `docs/components/showcase/`
3. **How-to** — Step-by-step guides. `docs/how-to/`
4. **Architecture** — System design, ADRs. `docs/architecture/`, `docs/agent-artifacts/core/`
5. **Reference** — Packages, env vars, types. `docs/reference/`

## Catalog rules

- **Catalog = playground:** All components, mock data, no real backend. Code-heavy, minimal prose.
- **Settings:** Special codegen section. Document `pnpm settings:generate`, tree-as-source, fieldKey/type/default. Link to [settings-tree-as-source-and-codegen](../../architecture/settings-tree-as-source-and-codegen.mdx).
- **Doc improvements:** Add ideas to [enhanced-features-backlog](./enhanced-features-backlog.md) with "docs" tag. Do not implement proposed items until a human sets status to accepted.

## Agent blocks in MDX

Use `<Callout variant="note" title="For coding agents">` for agent-only sections. Do **not** use raw HTML `<details>`/`<summary>` — MDX parsers can fail with "Unexpected character `!`" when parsing details content.

```mdx
<Callout variant="note" title="For coding agents">
File refs: path/to/file
Related: other-doc
Do not: forbidden pattern
</Callout>
```

Grep `AGENT_SECTION` in docs for existing blocks. See [errors-and-attempts § MDX agent blocks](errors-and-attempts.md).

## DocLink

- Use `.mdx` extension for internal doc links so DocLink resolves them.
- Use hash anchors for sections: `components/showcase/molecules.mdx#editorshell`.

## When adding components

1. Add to [Components Showcase](../../components/showcase/atoms.mdx) (atoms or molecules).
2. Add a demo in `apps/studio/registry/default/example/` if high-value; run `pnpm build:registry`.
3. Update [Building Studio](../../guides/building-studio.mdx) if the component is part of the build flow.

## When changing settings

1. Update [Settings tree-as-source and codegen](../../architecture/settings-tree-as-source-and-codegen.mdx).
2. Run `pnpm settings:generate`.

## When adding env vars

1. Update `scripts/env/manifest.mjs`.
2. Run `pnpm env:sync:examples`.

## Related

- [18-agent-artifacts-index](../../18-agent-artifacts-index.mdx)
- [19-coding-agent-strategy](../../19-coding-agent-strategy.mdx)
- [decisions.md](./decisions.md) — ADR "Documentation structure"
