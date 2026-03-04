---
title: Playground errors and attempts
created: 2026-02-12
updated: 2026-02-12
---

Simulation-specific failures and fixes. Index: [README.md](./README.md).

# Errors and attempts

Known simulation failures. Do not repeat; reference fix guides when retrying.

## Plan showed ToolFallback instead of Plan UI

**Symptom**: User asks for a plan; assistant would call forge_createPlan; result renders as generic ToolFallback (JSON dump) instead of tool-ui Plan.

**Cause**: ToolUIRegistry maps `render_plan`, not `forge_createPlan`. Domain tool results for forge_createPlan have no registered renderer.

**Fix**: Add makeAssistantToolUI with toolName `forge_createPlan`, or use makeAssistantTool with render prop when registering the tool. See [fix-plan-and-tools.md](../fix-plan-and-tools.md).

## Model cannot call forge_createPlan

**Symptom**: Assistant cannot create plans; model has no Forge tools.

**Cause**: useDomainAssistant and useForgeAssistantContract are not used in DialogueEditor. Forge tools are never registered with the Assistant UI runtime. API receives empty or default tools.

**Fix**: Wire useDomainAssistant(forgeAssistantContract) inside DialogueAssistantPanel (or a child of AssistantRuntimeProvider). See [fix-plan-and-tools.md](../fix-plan-and-tools.md) Fix 1.
