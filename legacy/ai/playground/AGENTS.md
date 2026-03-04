---
title: AI Playground — Virtual Simulation
created: 2026-02-11
updated: 2026-02-12
---

# AI Playground — Virtual Simulation

This folder is used for **virtual development simulations** of the AI chat system. We pretend the Assistant UI architecture (from [docs/ai/](../../ai/)) is implemented and roleplay as the chat assistant to surface gaps, errors, and fix strategies.

## Ralph Wiggum loop (simulation)

When doing simulations:

1. **Before**: Read [agent-artifacts/STATUS.md](./agent-artifacts/STATUS.md) and this AGENTS.md.
2. **Pick slice**: Choose a task from [agent-artifacts/task-registry.md](./agent-artifacts/task-registry.md) with `status: open`.
3. **Implement**: Document, update session logs, create or update fix guides. **No code edits** to apps/packages unless explicitly migrating to real implementation.
4. **After**: Update [agent-artifacts/STATUS.md](./agent-artifacts/STATUS.md) Done list; set task status in task-registry.

**Isolation**: Do **not** update `docs/agent-artifacts/core/STATUS.md` or other main artifacts during simulation. See [agent-artifacts/README.md](./agent-artifacts/README.md) for loop details.

## How to behave

1. **In-character mode**: When the user is "in" the Dialogue Editor chat, respond as the AI assistant would—helpful, context-aware of the Forge/dialogue domain, offering to use tools (createPlan, createNode, etc.).
2. **Out-of-character mode**: When the user says "break out of character" or similar, switch to developer/analyst mode. Explain what would have failed, why, and how to fix it with code examples and file references.
3. **Simulation context**: Assume the **virtual** state:
   - Dialogue Editor is open; user is in the right-rail Chat panel (DialogueAssistantPanel, Assistant UI).
   - **Tools are NOT wired**: `useForgeAssistantContract` and `useDomainAssistant` do not exist or are not used in DialogueEditor. The Forge domain tools (forge_createPlan, forge_createNode, etc.) are not registered with the Assistant UI runtime.
   - **Some UI components may error or fallback**: When a plan would be shown, the Plan component fails or shows ToolFallback because forge_createPlan results have no registered renderer (ToolUIRegistry maps `render_plan`, not `forge_createPlan`).

## Key references

| Item | Location |
|------|----------|
| DialogueEditor | apps/studio/components/editors/DialogueEditor.tsx |
| DialogueAssistantPanel | apps/studio/components/editors/dialogue/DialogueAssistantPanel.tsx |
| ToolUIRegistry | packages/shared/src/shared/components/tool-ui/assistant-tools.tsx |
| Forge copilot (current) | packages/domain-forge/src/copilot/ (CopilotKit) |
| useDomainAssistant (docs only) | docs/ai/02-domain-integration.mdx |
| Plan component | packages/shared/src/shared/components/tool-ui/plan/ |
| API route | apps/studio/app/api/assistant-chat/route.ts |

## Session logs

Conversations and virtual scenarios are logged in `session-*.md` files for traceability.

## Fix guides and context

| Doc | Purpose |
|-----|---------|
| [fix-plan-and-tools.md](./fix-plan-and-tools.md) | Wire Forge tools to Assistant UI; register Plan renderer for forge_createPlan |
| [forge-domain-plan-context.md](./forge-domain-plan-context.md) | Narrative vs storylet, page structure (Act/Chapter/Page), detour/jump nodes; code and files needed for concrete plans |

## Agent artifacts (simulation)

| Doc | Purpose |
|-----|---------|
| [agent-artifacts/README.md](./agent-artifacts/README.md) | Loop, isolation, compacting |
| [agent-artifacts/STATUS.md](./agent-artifacts/STATUS.md) | Simulation state, Ralph Wiggum Done |
| [agent-artifacts/task-registry.md](./agent-artifacts/task-registry.md) | Pickable simulation tasks |
| [agent-artifacts/errors-and-attempts.md](./agent-artifacts/errors-and-attempts.md) | Simulation failures and fixes |
