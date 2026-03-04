---
title: Forge Domain Plan Context — Narrative, Storylet, Pages, Detours
created: 2026-02-11
updated: 2026-02-11
---

# Forge Domain Plan Context — Narrative, Storylet, Pages, Detours

**Context**: Virtual session 001 (Magicborn). User asked for a plan. The Magicborn scene (entry → Dean → 3 choice branches → chaotic magic) is a **storylet**. This doc captures what the Forge domain needs for the AI to create concrete plans—including the narrative/storylet split, page structure, and detour/jump nodes.

---

## Domain model (target)

### Narrative vs Storylet

| Graph kind | Purpose | Node types |
|------------|---------|------------|
| **Narrative** | High-level story structure. Nodes are **pages** organized as Act / Chapter / Page. |
| **Storylet** | Fine-grained dialogue within a beat. CHARACTER, PLAYER, CONDITIONAL (choices, branches). |

The Magicborn playable scene is a **storylet**—dialogue nodes and choices.

### Narrative: page structure

Narrative nodes are **pages** with structure:
- `Act ${lastActNumber}`
- `Chapter ${lastChapterNumber}`
- `Page ${lastPageNumber}`

Example: "Act 1", "Chapter 1", "Page 1" → "Act 1", "Chapter 1", "Page 2" → …

### Detour and jump nodes

- **Detour node** / **Jump node**: branch from a narrative **page** to a **storylet**
- Branch is a choice; choice can have **conditions**
- Detour/jump **resolves to the storylet later**; IDs must stay in sync between narrative and storylet graphs

---

## What exists today

| Item | Location | Notes |
|------|----------|-------|
| Graph kinds | [packages/types/src/graph.ts](../../../packages/types/src/graph.ts) | `NARRATIVE`, `STORYLET` |
| Node types | [packages/types/src/graph.ts](../../../packages/types/src/graph.ts) | `CHARACTER`, `PLAYER`, `CONDITIONAL` only |
| ForgeGraphDoc | packages/types, forge-graphs collection | `flow: { nodes, edges, viewport }` |
| buildForgeContext | [packages/domain-forge/src/copilot/context.ts](../../../packages/domain-forge/src/copilot/context.ts) | Includes `graphKind`, `nodeCount` |
| Plan API | [apps/studio/app/api/forge/plan/route.ts](../../../apps/studio/app/api/forge/plan/route.ts) | `createNode`, `updateNode`, `deleteNode`, `createEdge` |
| planStepToOp | [packages/domain-forge/src/copilot/plan-utils.ts](../../../packages/domain-forge/src/copilot/plan-utils.ts) | Maps steps → ForgeGraphPatchOp |
| CreatePlan graphSummary | Copilot actions | `title`, `nodeCount`, `nodes`, `edges` |
| CSS detour accent | [packages/shared/.../contexts.css](../../../packages/shared/src/shared/styles/contexts.css) | `--node-detour-accent` (concept only) |

---

## What's missing (for concrete plans)

### 1. Node types

**Current**: `CHARACTER`, `PLAYER`, `CONDITIONAL`

**Needed for narrative/storylet split**:
- `PAGE` — narrative page (Act/Chapter/Page)
- `DETOUR` or `JUMP` — branch from page to storylet (references storylet graph/node by ID)
- Possibly `CONDITIONAL` on choices (condition expression)

| File | Change |
|------|--------|
| [packages/types/src/graph.ts](../../../packages/types/src/graph.ts) | Add `PAGE`, `DETOUR` (or `JUMP`) to `FORGE_NODE_TYPE` |
| [packages/domain-forge/src/copilot/plan-utils.ts](../../../packages/domain-forge/src/copilot/plan-utils.ts) | Handle `createNode` with `nodeType: 'PAGE'`, `'DETOUR'`; PAGE `data`: `act`, `chapter`, `page` |
| [apps/studio/app/api/forge/plan/route.ts](../../../apps/studio/app/api/forge/plan/route.ts) | Extend schema so PAGE has `act`, `chapter`, `page`; DETOUR has `storyletGraphId`, `storyletNodeId` |
| [apps/studio/components/GraphEditor.tsx](../../../apps/studio/components/GraphEditor.tsx) | Register `PageNode`, `DetourNode` (or `JumpNode`) |

### 2. Context snapshot (graphKind-aware)

**Current**: `buildForgeContext` sends `graphKind` but no narrative metadata.

**Needed**:
- For **NARRATIVE**: last act/chapter/page so AI can propose next labels
- For **STORYLET**: current pattern (nodeIds, edges)
- **Active scope**: narrative vs storylet (which graph is focused)

| File | Change |
|------|--------|
| [packages/domain-forge/src/copilot/context.ts](../../../packages/domain-forge/src/copilot/context.ts) | `domainState`: add `lastAct`, `lastChapter`, `lastPage` when `graphKind === 'NARRATIVE'`; add `activeScope` |
| DialogueEditor → contract deps | Pass `activeScope` so context knows which graph is being edited |
| createPlan graphSummary | Include narrative page counts when kind is NARRATIVE |

### 3. Instructions (getInstructions)

**Current**: "CHARACTER, PLAYER, CONDITIONAL. Use forge_* tools."

**Needed (scope-aware)**:
- **Narrative**: "Pages use Act/Chapter/Page. Use `forge_createPage` or createNode with nodeType PAGE. Detour nodes branch to storylets; sync IDs."
- **Storylet**: "Dialogue nodes: CHARACTER, PLAYER, CONDITIONAL. Choices can have conditions."

| File | Change |
|------|--------|
| [packages/domain-forge/src/copilot/index.ts](../../../packages/domain-forge/src/copilot/index.ts) | `getInstructions` varies by `graphKind` (or `activeScope`) |
| [packages/domain-forge/src/assistant/index.ts](../../../packages/domain-forge/src/assistant/) | Same when Assistant UI contract exists |

### 4. Tools

**Current**: `forge_createNode` (nodeType), `forge_createEdge`, etc.

**Needed**:
- `forge_createPage` — create PAGE with computed Act/Chapter/Page (or extend `createNode` with PAGE-specific params)
- `forge_createDetour` — create DETOUR from page to storylet (or `createNode` with DETOUR + `storyletGraphId`, `storyletNodeId`)
- `forge_getStoryletIds` — list storylet graph IDs / entry node IDs for linking
- Or: keep `forge_createNode` and add `nodeType: 'PAGE' | 'DETOUR'` with appropriate `data`

| File | Change |
|------|--------|
| [packages/domain-forge/src/copilot/actions.ts](../../../packages/domain-forge/src/copilot/actions.ts) | Add PAGE/DETOUR handling to `createNode`; optional `forge_createPage`, `forge_createDetour` |
| [apps/studio/app/api/forge/plan/route.ts](../../../apps/studio/app/api/forge/plan/route.ts) | System prompt + schema for PAGE (act, chapter, page) and DETOUR (storyletGraphId, storyletNodeId) |

### 5. Plan API system prompt

**Current**: "createNode (nodeType, label, content?, speaker?, x?, y?)" — storylet-centric.

**Needed for narrative**:
- For narrative: "createNode with nodeType PAGE; data: { act, chapter, page, label? }"
- For detour: "createNode with nodeType DETOUR; data: { storyletGraphId, storyletNodeId, label? }"
- Clarify: "For storylets use nodeType CHARACTER, PLAYER, CONDITIONAL. For narrative use PAGE and DETOUR."

| File | Change |
|------|--------|
| [apps/studio/app/api/forge/plan/route.ts](../../../apps/studio/app/api/forge/plan/route.ts) | Branch system prompt by `graphSummary.kind`; extend step schema for PAGE/DETOUR |

---

## Usage flow (for Magicborn)

1. **User**: "Create a Magicborn scene" (in storylet scope)
2. **Context**: `activeScope: 'storylet'`, `graphKind: 'STORYLET'`
3. **Instructions**: Storylet-specific (CHARACTER, PLAYER, CONDITIONAL)
4. **forge_createPlan(goal)**: Plan API gets `graphSummary` with `kind: 'STORYLET'`; returns storylet ops (createNode CHARACTER/PLAYER, createEdge)
5. **Plan steps**: Entry → Dean → Player choice → 3 branches → close
6. **forge_executePlan**: Applies to **storylet** graph

**Future (narrative + storylet)**:
1. User creates narrative pages (Act 1, Ch 1, P1, P2…)
2. User adds DETOUR from P2 → storylet "Magicborn Academy"
3. Storylet has its own graph; DETOUR stores `storyletGraphId` + entry node
4. IDs stay synced when storylet is created/updated

---

## Checklist for implementation

- [ ] Add `PAGE`, `DETOUR` to `FORGE_NODE_TYPE` ([packages/types/src/graph.ts](../../../packages/types/src/graph.ts))
- [ ] Extend `ForgeNode` / node `data` for PAGE (`act`, `chapter`, `page`) and DETOUR (`storyletGraphId`, `storyletNodeId`)
- [ ] `buildForgeContext`: narrative page metadata when `graphKind === 'NARRATIVE'`
- [ ] `getInstructions`: scope-aware (narrative vs storylet)
- [ ] `planStepToOp`: PAGE and DETOUR in `createNode`
- [ ] Plan API: schema and prompt for PAGE/DETOUR; kind-aware
- [ ] Register PageNode, DetourNode in GraphEditor
- [ ] Tools: `forge_createNode` accepts PAGE/DETOUR; optional `forge_createPage`, `forge_createDetour`

---

## See also

- [fix-plan-and-tools.md](./fix-plan-and-tools.md) — Wire Forge tools to Assistant UI; Plan component renderer
- [session-001-magicborn.md](./session-001-magicborn.md) — Virtual session context
