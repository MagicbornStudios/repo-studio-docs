---
title: Fix — Plan Component and Forge Tools in Dialogue Editor
created: 2026-02-11
updated: 2026-02-11
---

# Fix: Plan Component and Forge Tools in Dialogue Editor

**Context**: Virtual session 001. User asks for a plan; Plan component errors or shows fallback. Root causes: (1) Forge tools not wired to Assistant UI, (2) forge_createPlan results have no registered renderer.

---

## Problem summary

| Issue | Cause |
|-------|-------|
| Model cannot call forge_createPlan | useDomainAssistant not used; Forge tools never registered with Assistant UI runtime |
| Plan shows ToolFallback instead of Plan UI | ToolUIRegistry maps `render_plan`, not `forge_createPlan`. Domain tool results for `forge_createPlan` have no renderer |

---

## Architecture references

- **Domain contract**: [docs/ai/02-domain-integration.mdx](../02-domain-integration.mdx)
- **Forge tools**: [docs/ai/03-domains-forge-character.mdx](../03-domains-forge-character.mdx)
- **Current CopilotKit contract**: [packages/domain-forge/src/copilot/index.ts](../../../packages/domain-forge/src/copilot/index.ts)
- **CreatePlan API**: [apps/studio/lib/data/hooks/use-create-forge-plan.ts](../../../apps/studio/lib/data/hooks/use-create-forge-plan.ts)
- **assistant-chat API**: [apps/studio/app/api/assistant-chat/route.ts](../../../apps/studio/app/api/assistant-chat/route.ts) — accepts `body.tools`, passes to `streamText`
- **Assistant UI Tool UI**: [assistant-ui.com/docs/guides/ToolUI](https://www.assistant-ui.com/docs/guides/ToolUI)
- **makeAssistantTool** (supports `render`): [assistant-ui.com/docs/copilots/make-assistant-tool](https://www.assistant-ui.com/docs/copilots/make-assistant-tool)
- **makeAssistantToolUI** (UI-only by toolName): [assistant-ui.com/docs/copilots/make-assistant-tool-ui](https://www.assistant-ui.com/docs/copilots/make-assistant-tool-ui)

---

## Fix 1: Wire Forge domain tools to Assistant UI

### 1.1 Create shared domain assistant hook

**File**: `packages/shared/src/shared/assistant/use-domain-assistant.ts` (new)

```typescript
'use client';

import { useEffect } from 'react';
import { useAssistantContext } from '@assistant-ui/react';
import type { DomainAssistantContract } from './domain-contract';

export function useDomainAssistant(
  contract: DomainAssistantContract,
  options?: { enabled?: boolean }
) {
  const { enabled = true } = options ?? {};
  const assistant = useAssistantContext();

  useEffect(() => {
    if (!enabled) return;

    const tools = contract.createTools().map((tool) => ({
      name: tool.name,
      description: tool.description,
      parameters: tool.parameters,
      execute: async (args: unknown) => {
        const snapshot = contract.getContextSnapshot();
        return await tool.execute(args, {
          snapshot,
          emit: (event) => assistant.emit?.(event),
        });
      },
    }));

    assistant.registerTools?.(tools);

    return () => {
      assistant.unregisterTools?.(tools.map((t) => t.name));
    };
  }, [enabled, contract, assistant]);

  // ... context sync, suggestions (see docs/ai/02-domain-integration.mdx)
  return { onHighlight: contract.onHighlight, clearHighlights: contract.clearHighlights };
}
```

**File**: `packages/shared/src/shared/assistant/domain-contract.ts` (new)

```typescript
export interface DomainAssistantContract {
  domain: string;
  getContextSnapshot(): DomainContextSnapshot;
  getInstructions(): string;
  createTools(): DomainTool[];
  getSuggestions(): Array<{ title: string; message: string }>;
  onHighlight(entities: Record<string, string[]>): void;
  clearHighlights(): void;
}

export interface DomainTool {
  domain: string;
  name: string;
  description: string;
  parameters: { type: 'object'; properties: Record<string, unknown>; required?: string[] };
  execute: (args: unknown, context: DomainToolContext) => Promise<unknown>;
}
```

### 1.2 Create Forge Assistant contract

**File**: `packages/domain-forge/src/assistant/index.ts` (new)

Implement `useForgeAssistantContract(deps)` returning `DomainAssistantContract` with `createTools()` that includes `forge_createPlan`. Reuse logic from [packages/domain-forge/src/copilot/actions.ts](../../../packages/domain-forge/src/copilot/actions.ts) (createPlan handler) and [use-create-forge-plan.ts](../../../apps/studio/lib/data/hooks/use-create-forge-plan.ts) for the API call.

### 1.3 Wire in DialogueEditor

**File**: [apps/studio/components/editors/DialogueEditor.tsx](../../../apps/studio/components/editors/DialogueEditor.tsx)

```tsx
// Add imports
import { useDomainAssistant } from '@forge/shared/assistant';
import { useForgeAssistantContract } from '@forge/domain-forge/assistant';

// Inside DialogueEditor, after forgeContract:
const forgeAssistantContract = useForgeAssistantContract({
  graph: activeGraph,
  selection: activeSelection,
  isDirty: activeDirty,
  applyOperations: applyActiveOperations,
  onAIHighlight,
  clearHighlights,
  createPlanApi,
  setPendingFromPlan: setPendingFromPlanActive,
  executePlan: /* ... */,
  commitGraph,
});

useDomainAssistant(forgeAssistantContract, { enabled: toolsEnabled });
```

`useDomainAssistant` must run **inside** `AssistantRuntimeProvider`. `DialogueAssistantPanel` provides that. So either:
- Move `useDomainAssistant` into a child of `DialogueAssistantPanel`, or
- Pass the contract into `DialogueAssistantPanel` and call `useDomainAssistant` there.

**Recommended**: Add `DialogueAssistantPanel` prop `contract?: DomainAssistantContract` and call `useDomainAssistant(contract)` inside the panel when `contract` is provided. DialogueEditor passes `forgeAssistantContract`.

---

## Fix 2: Register Plan renderer for forge_createPlan results (Option C — recommended)

**Verification**: Assistant UI supports custom UI for tools the model calls:

1. **`makeAssistantTool`** — Accepts optional `render` on the tool definition: `render?: ComponentType<ToolCallMessagePartProps<TArgs, TResult>>`. [makeAssistantTool docs](https://www.assistant-ui.com/docs/copilots/make-assistant-tool)
2. **`makeAssistantToolUI`** — UI-only registration by `toolName`. Use when the tool is defined elsewhere; component matches tool results by name. [makeAssistantToolUI docs](https://www.assistant-ui.com/docs/copilots/make-assistant-tool-ui)

Both support **Option C: domain tool render prop**. Use either inline render on the tool or a separate UI component keyed by `toolName`.

---

### Option C — Implementation paths

#### Path 1: `makeAssistantTool` with `render` (inline)

When registering `forge_createPlan` via domain tools, pass `render` to `makeAssistantTool`:

```tsx
// In domain-forge assistant tools
const ForgeCreatePlanTool = makeAssistantTool({
  toolName: 'forge_createPlan',
  parameters: z.object({ goal: z.string() }),
  execute: async ({ goal }, ctx) => { /* ... */ },
  render: ({ args, result, status }) => {
    if (status.type === 'running') return <PlanSkeleton />;
    if (!result?.success || !result.steps) return null;
    return (
      <Plan
        id="plan-1"
        title={result.goal}
        todos={result.steps.map((s, i) => ({
          id: String(i),
          label: s.type ?? s.title ?? `Step ${i + 1}`,
          status: 'pending',
          description: s.description,
        }))}
        responseActions={[
          { id: 'approve', label: 'Apply Plan' },
          { id: 'revise', label: 'Request Changes', variant: 'secondary' },
        ]}
        onResponseAction={(id) => id === 'approve' && executePlan?.(result.steps)}
      />
    );
  },
});
```

#### Path 2: `makeAssistantToolUI` (UI-only, recommended)

Add a UI component for `forge_createPlan`; tool execution stays in domain registration. Mount inside `AssistantRuntimeProvider` (e.g. alongside `ToolUIRegistry`).

**File**: [packages/shared/src/shared/components/tool-ui/forge-plan-tool-ui.tsx](../../../packages/shared/src/shared/components/tool-ui/forge-plan-tool-ui.tsx) (new)

```tsx
'use client';

import { makeAssistantToolUI } from '@assistant-ui/react';
import { Plan } from './plan';

type ForgePlanResult = {
  success?: boolean;
  goal?: string;
  steps?: Array<{ type?: string; title?: string; description?: string }>;
};

export const ForgePlanToolUI = makeAssistantToolUI<
  { goal: string },
  ForgePlanResult
>({
  toolName: 'forge_createPlan',
  render: ({ result, status }) => {
    if (status?.type === 'running' && result == null) {
      return <div className="rounded border p-3 bg-muted/20">Creating plan...</div>;
    }
    if (!result?.success || !Array.isArray(result.steps)) return null;

    return (
      <Plan
        id="plan-1"
        title={result.goal ?? 'Plan'}
        todos={result.steps.map((s, i) => ({
          id: String(i),
          label: s.type ?? s.title ?? `Step ${i + 1}`,
          status: 'pending',
          description: s.description,
        }))}
        responseActions={[
          { id: 'approve', label: 'Apply Plan' },
          { id: 'revise', label: 'Request Changes', variant: 'secondary' },
        ]}
        onResponseAction={(id) => {
          if (id === 'approve') {
            // executePlan must be provided via context or prop
            // Use useInlineRender if wiring from parent (DialogueAssistantPanel)
          }
        }}
      />
    );
  },
});
```

**Note**: `onResponseAction` needs access to `executePlan`. Two options: (1) Use Path 1 (`makeAssistantTool` with `render`) — the render is defined where `executePlan` is in scope. (2) Use `useAssistantToolUI` + `useInlineRender` in a wrapper that receives `executePlan` as a prop; the inline render closes over it. See [Assistant UI Tool UI guide](https://www.assistant-ui.com/docs/guides/ToolUI#inline-pattern).

**File**: [apps/studio/components/editors/dialogue/DialogueAssistantPanel.tsx](../../../apps/studio/components/editors/dialogue/DialogueAssistantPanel.tsx)

Mount `ForgePlanToolUI` inside `AssistantRuntimeProvider`:

```tsx
<AssistantRuntimeProvider runtime={runtime}>
  <AssistantDevToolsBridge />
  <ToolUIRegistry />
  <ForgePlanToolUI />  {/* add this */}
  <div className={cn('flex h-full min-h-0 flex-col', className)}>
    <Thread ... />
  </div>
</AssistantRuntimeProvider>
```

---

### Other options (not recommended)

- **Option A**: Extend ToolUIRegistry with `forge_createPlan` — equivalent to Path 2; both use `makeAssistantToolUI`-style registration.
- **Option B**: Have the model call `render_plan` after `forge_createPlan` — adds extra tool calls and coupling; avoid.

---

## Fix 3: Ensure executePlan is available

When the user clicks **Apply Plan**, `onResponseAction('approve')` must call `executePlan(steps)`. The Forge assistant contract needs access to the same `executePlan` logic as CopilotKit (from [packages/domain-forge/src/copilot/actions.ts](../../../packages/domain-forge/src/copilot/actions.ts) executePlan handler). Pass `executePlan` into the contract deps and wire it in the Plan's `onResponseAction`.

---

## Verification checklist

- [ ] `useDomainAssistant` runs inside `AssistantRuntimeProvider` (e.g. inside DialogueAssistantPanel)
- [ ] `forge_createPlan` appears in the tools sent to `/api/assistant-chat`
- [ ] Model can call `forge_createPlan`; API forwards it; client executes and returns `{ success, steps, goal }`
- [ ] Plan UI renders in chat (not ToolFallback) when `forge_createPlan` returns
- [ ] "Apply Plan" calls `executePlan`; graph updates and review bar shows "Pending changes from plan"
- [ ] Coexistence: CopilotKit (drawer) and Assistant UI (panel) can both be active during migration if using a feature flag
