---
title: Workflow Complexity Guide
created: 2026-02-11
updated: 2026-02-11
---

# Workflow Complexity Guide

**When to use Vanilla vs LangGraph for your AI workflows**

---

## 🟢 Simple = Vanilla Async (90% of your workflows)

### Characteristics

✅ Linear flow (A → B → C)
✅ One or zero decision points
✅ No need to resume if interrupted
✅ Single pass (no iteration/refinement)

### Your Use Cases

#### 1. **Create Story with Characters**

```typescript
// User: "Create a space pirate story"

async function createStory(premise: string) {
  const characters = await createCharacters(premise, 3);  // A
  const structure = await generateStory(premise, characters);  // B
  const graphOps = convertToGraphOps(structure, characters);  // C
  applyOperations(graphOps);  // Done
}
```

**Result:**
- 3 characters with portraits ✅
- 5 dialogue nodes ✅
- 4 edges connecting them ✅
- All in graph, ready to edit ✅

**Time:** 6 seconds
**Cost:** $0.07
**Complexity:** LOW ✅

---

#### 2. **Create Merchant Storylet**

```typescript
// User: "Create a merchant dialogue with 3 greeting options"

async function createMerchantStorylet(merchantType: string) {
  const character = await character_create({
    name: 'Merchant',
    type: merchantType,
    personality: 'Friendly, business-savvy',
  });

  const greetingNode = await forge_createNode({
    nodeType: 'CHARACTER',
    speaker: character.name,
    content: 'Welcome to my shop! What can I get for you?',
  });

  const choices = await forge_createNode({
    nodeType: 'PLAYER',
    choices: [
      { text: 'Show me your wares', nextNodeId: 'shop' },
      { text: 'Any rumors?', nextNodeId: 'rumors' },
      { text: 'Goodbye', nextNodeId: null },
    ],
  });

  await forge_createEdge({
    source: greetingNode.id,
    target: choices.id,
  });
}
```

**Result:**
- 1 merchant character ✅
- 2 nodes (greeting + choices) ✅
- 1 edge ✅
- Ready for player interaction ✅

**Time:** 2 seconds
**Cost:** $0.005
**Complexity:** LOW ✅

---

#### 3. **Batch Create NPCs**

```typescript
// User: "Create 5 townsfolk NPCs"

async function createTownsfolk(count: number) {
  const npcs = await character_createBatch({
    count,
    type: 'townsfolk',
    sharedContext: 'Medieval fantasy village',
  });

  // Generate portraits in parallel
  await Promise.all(npcs.map(npc =>
    character_generatePortrait({ characterId: npc.id })
  ));

  return npcs;
}
```

**Result:**
- 5 NPCs with unique names/personalities ✅
- 5 portraits ✅
- Saved to character database ✅

**Time:** 4 seconds
**Cost:** $0.04
**Complexity:** LOW ✅

---

## 🟡 Medium = Still Vanilla, Just Organized (9% of workflows)

### Characteristics

⚠️ Multiple related operations
⚠️ Some conditional logic
⚠️ User review/approval needed
✅ Still one-shot (no looping back)

### Your Use Cases

#### 1. **Plan → Review → Execute**

```typescript
// User: "Add 3 character nodes and connect them in a conversation"

async function planReviewExecute(goal: string) {
  // 1. Create plan
  const plan = await forge_createPlan({ goal });

  // 2. Show plan to user (PlanReviewCard component)
  const approved = await userReviewPlan(plan);

  // 3. Execute if approved
  if (approved) {
    await forge_executePlan({ steps: plan.steps });
  } else {
    return { cancelled: true };
  }

  // 4. Optional: Commit to server
  if (userWantsToSave) {
    await commitGraph();
  }
}
```

**Result:**
- Plan with 7 operations ✅
- User sees preview before applying ✅
- Applied to graph if approved ✅

**Time:** 3 seconds + user review
**Cost:** $0.01
**Complexity:** MEDIUM ⚠️

**Why still vanilla?**
- Linear flow with one branch (if approved)
- No loops, no retries, no state persistence

---

#### 2. **Multi-Domain Operation**

```typescript
// User: "Create a quest with a quest-giver character and dialogue"

async function createQuestWithDialogue(questPremise: string) {
  // 1. Create quest-giver character
  const questGiver = await character_create({
    name: 'Quest Giver',
    role: 'quest-npc',
    personality: 'Mysterious, urgent',
  });

  // 2. Generate quest dialogue based on character
  const questDialogue = await generateQuestDialogue(questPremise, questGiver);

  // 3. Create dialogue graph
  const graphOps = convertDialogueToOps(questDialogue, questGiver);
  applyOperations(graphOps);

  // 4. Link character to graph nodes
  await linkCharacterToNodes(questGiver.id, graphOps);

  return { questGiver, dialogue: questDialogue };
}
```

**Result:**
- 1 character (quest-giver) ✅
- 4 dialogue nodes (intro, accept, reject, complete) ✅
- 3 edges ✅
- Character linked to nodes ✅

**Time:** 5 seconds
**Cost:** $0.03
**Complexity:** MEDIUM ⚠️

**Why still vanilla?**
- Operations are sequential, each depends on previous
- No user intervention mid-flow
- No need to retry or loop

---

## 🔴 Complex = LangGraph Territory (1% of workflows)

### Characteristics

❌ User iterates/refines at multiple stages
❌ Need to resume if browser closes
❌ Complex conditional branching
❌ Multi-agent coordination
❌ Retry logic with fallbacks

### When You Actually Need This

#### 1. **Iterative Story Building**

```
User: "Create a fantasy story"
AI: [Creates 3 characters]
AI: Shows character previews

User: "The wizard is too cliché, make them a rogue scholar"
AI: [Regenerates wizard → rogue scholar]  ← 🔴 LOOP BACK
AI: Shows updated characters

User: "Approve"
AI: [Generates plot with approved characters]
AI: Shows plot outline

User: "Too predictable, add a betrayal"
AI: [Regenerates plot with betrayal]  ← 🔴 LOOP BACK
AI: Shows updated plot

User: "Approve"
AI: [Generates dialogue graph]
```

**LangGraph implementation:**

```typescript
const storyWorkflow = new StateGraph({
  channels: {
    premise: null,
    characters: [],
    charactersApproved: false,
    plot: null,
    plotApproved: false,
    graph: null,
  },
});

workflow.addNode('generateCharacters', async (state) => { ... });
workflow.addNode('reviewCharacters', async (state) => { ... });
workflow.addNode('generatePlot', async (state) => { ... });
workflow.addNode('reviewPlot', async (state) => { ... });

// Conditional edges allow looping back
workflow.addConditionalEdges('reviewCharacters', (state) => {
  if (state.charactersApproved) return 'generatePlot';
  return 'generateCharacters';  // Loop back to regenerate
});

workflow.addConditionalEdges('reviewPlot', (state) => {
  if (state.userWantsToReviseCharacters) return 'generateCharacters';  // Jump back
  if (state.plotApproved) return 'generateGraph';
  return 'generatePlot';  // Loop back to regenerate plot
});
```

**Time:** 30 seconds + multiple user reviews
**Cost:** $0.15 (multiple generations)
**Complexity:** HIGH 🔴

---

#### 2. **Resume Across Sessions**

```
Session 1:
User: "Create a complex story with 10 characters"
AI: Creates 5 characters... [User closes browser]

Session 2 (next day):
User: Opens app
AI: "You have an in-progress story. Resume?"
User: "Yes"
AI: Resumes from checkpoint (5/10 characters done)
AI: Continues creating remaining 5 characters
```

**LangGraph with checkpoints:**

```typescript
import { MemorySaver } from '@langchain/langgraph';

const checkpointer = new MemorySaver();
const workflow = graph.compile({ checkpointer });

// Persist state
const config = { configurable: { thread_id: userId } };

// First session
await workflow.invoke({ goal: 'create 10 characters' }, config);
// State saved automatically

// Second session (resume)
const currentState = await workflow.getState(config);
// Continues from where it left off
await workflow.invoke(null, config);  // No input needed, resumes
```

**Why you need this:**
- Large operations (10+ characters, complex plots)
- User might interrupt
- Can't lose progress

**Complexity:** HIGH 🔴

---

#### 3. **Multi-Agent Orchestration**

```
User: "Create a merchant quest line"

LangGraph delegates to specialized agents:

1. Planner Agent (deepseek-reasoner)
   → Creates quest structure: 5 steps, rewards, branches
   Cost: $0.003

2. Writer Agent (claude-3.5-sonnet)
   → Writes merchant dialogue for each step
   Cost: $0.05

3. Implementer Agent (gpt-4o-mini)
   → Converts dialogue to graph nodes/edges
   Cost: $0.0005

4. Reviewer Agent (qwen/qwq-32b-preview)
   → Validates graph for logic errors
   Cost: $0 (free tier)

Total: $0.053 vs $0.15 (single Sonnet)
```

**LangGraph multi-agent:**

```typescript
const questWorkflow = new StateGraph({ ... });

workflow.addNode('plan', async (state) => {
  const planner = selectModel('deepseek/deepseek-reasoner');
  return await planner.createQuestStructure(state.goal);
});

workflow.addNode('write', async (state) => {
  const writer = selectModel('anthropic/claude-3.5-sonnet');
  return await writer.generateDialogue(state.structure);
});

workflow.addNode('implement', async (state) => {
  const implementer = selectModel('openai/gpt-4o-mini');
  return await implementer.createGraphOps(state.dialogue);
});

workflow.addNode('review', async (state) => {
  const reviewer = selectModel('qwen/qwq-32b-preview');
  return await reviewer.validateGraph(state.graphOps);
});

// Sequential flow through specialized agents
workflow.addEdge('plan', 'write');
workflow.addEdge('write', 'implement');
workflow.addEdge('implement', 'review');

// Conditional: If review fails, regenerate
workflow.addConditionalEdges('review', (state) => {
  if (state.validationErrors.length > 0) return 'implement';
  return END;
});
```

**Why you need this:**
- Cost optimization (different models excel at different tasks)
- Quality improvement (specialized agents)
- Complex validation/retry logic

**Complexity:** HIGH 🔴

---

## Summary: When to Use What

| Use Case | Vanilla | LangGraph |
|----------|---------|-----------|
| **Create story with characters** | ✅ Yes | ❌ No |
| **Create merchant storylet** | ✅ Yes | ❌ No |
| **Batch create NPCs** | ✅ Yes | ❌ No |
| **Plan → Review → Execute** | ✅ Yes | ❌ No |
| **Create quest with dialogue** | ✅ Yes | ❌ No |
| **User iterates on characters 3+ times** | ❌ No | ✅ Yes |
| **Resume 10-character creation after browser close** | ❌ No | ✅ Yes |
| **Multi-agent quest optimization** | ❌ No | ✅ Yes |
| **Complex branching (if X fails, try Y, else Z)** | ❌ No | ✅ Yes |

---

## Decision Tree

```
Can you write it as 3-5 sequential async calls?
├─ YES → Use vanilla
└─ NO → Does user need to iterate/revise mid-flow?
   ├─ NO → Does it need to resume across sessions?
   │  ├─ NO → Do you need multi-model routing with retries?
   │  │  ├─ NO → Use vanilla (you're overthinking it)
   │  │  └─ YES → Use LangGraph
   │  └─ YES → Use LangGraph (checkpoints)
   └─ YES → Use LangGraph (state machine)
```

---

## Cost Comparison

### Vanilla Story Builder (Your Primary Use Case)

```typescript
await forge_createStoryFromPremise({
  premise: 'Space pirate discovers haunted ship',
  characterCount: 3,
  sceneCount: 5,
});
```

**Cost:** $0.074
**Time:** 6 seconds
**Code:** 1 tool call

### LangGraph Iterative Story Builder

```typescript
const workflow = createIterativeStoryWorkflow();
await workflow.invoke({
  premise: 'Space pirate discovers haunted ship',
});
// User iterates 3 times on characters
// User iterates 2 times on plot
```

**Cost:** $0.30 (multiple regenerations)
**Time:** 2 minutes + user reviews
**Code:** 150+ lines

**Is it worth it?** Only if users demand iteration.

---

## Recommendation

### For Forge Studio (2026)

**Phase 1 (Now):**
✅ Implement vanilla `forge_createStoryFromPremise`
✅ Covers 90% of use cases
✅ Fast, cheap, easy to debug

**Phase 2 (When users complain):**
⏳ Add LangGraph for specific pain points
⏳ Example: "I wish I could revise characters before building the story"
⏳ Implement iterative workflow only for that use case

**Phase 3 (Future optimization):**
⏳ Multi-agent workflows for cost savings
⏳ Checkpoint persistence for large operations

**Don't build LangGraph until you have proof users need it.**

---

## Example: What You Should Implement First

### Single Tool: `forge_createStoryFromPremise`

```typescript
{
  name: 'forge_createStoryFromPremise',
  description: 'Create a complete story with characters, portraits, and dialogue graph',
  parameters: { premise: string, characterCount?: number },

  execute: async (args) => {
    // Vanilla, sequential, fast
    const characters = await createCharacters(args.premise, args.characterCount || 3);
    const structure = await generateStory(args.premise, characters);
    const graphOps = convertToGraphOps(structure, characters);
    applyOperations(graphOps);

    return { characters, scenes: structure.scenes };
  },
}
```

**This gives you:**
- Characters ✅
- Dialogue graph with connected nodes ✅
- Yarn Spinner-compatible structure ✅
- Working in 6 seconds ✅

**Total code:** ~200 lines
**Total complexity:** LOW
**Total value:** HIGH

**Start here. Build LangGraph when needed.**
