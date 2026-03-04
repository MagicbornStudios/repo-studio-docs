---
title: Complete Story Builder Workflow
created: 2026-02-11
updated: 2026-02-11
---

# Complete Story Builder Workflow

**Goal:** User says "Create a story about a space pirate" → Get characters + dialogue graph with connected nodes.

---

## The Full Vanilla Workflow

### Step 1: Create Characters (with portraits)

```typescript
// packages/domain-character/src/assistant/workflows/create-story-characters.ts

import type { CharacterDoc } from '@forge/types/character';

interface StoryCharacter extends CharacterDoc {
  portrait?: string;  // Image URL
}

export async function createStoryCharacters(
  premise: string,
  count: number = 3
): Promise<StoryCharacter[]> {
  // Use batched character creation (shares context, uses prompt caching)
  const characters = await character_createBatch({
    count,
    premise,
    sharedContext: `These are main characters for a story: ${premise}`,

    // Auto-generate personality, background, voice
    generatePersonality: true,
    generateBackground: true,
  });

  // Generate portraits in parallel (fast)
  const portraitsPromises = characters.map(char =>
    character_generatePortrait({
      characterId: char.id,
      description: char.description,
      style: 'cinematic-portrait',
    })
  );

  const portraits = await Promise.all(portraitsPromises);

  // Merge portraits into character data
  return characters.map((char, i) => ({
    ...char,
    portrait: portraits[i]?.url,
  }));
}
```

**Cost:** 3 characters with portraits
- Character generation (cached): $0.011
- Portrait generation (3 images): $0.06
- **Total: ~$0.071**

---

### Step 2: Generate Dialogue Graph Structure

```typescript
// packages/domain-forge/src/assistant/workflows/create-story-graph.ts

import type { ForgeGraphPatchOp, ForgeNodeType } from '@forge/types/graph';
import type { StoryCharacter } from '@forge/domain-character';

interface StoryStructure {
  scenes: Array<{
    title: string;
    description: string;
    speaker: string;  // Character name
    dialogue: string;
    nextScene?: string;
  }>;
}

export async function generateStoryStructure(
  premise: string,
  characters: StoryCharacter[]
): Promise<StoryStructure> {
  // Use reasoning model for structure (cheap, good at planning)
  const planner = 'deepseek/deepseek-reasoner';

  const prompt = `
Create a story structure for: ${premise}

Characters:
${characters.map(c => `- ${c.name}: ${c.personality}`).join('\n')}

Generate 5-7 connected scenes. Each scene should:
- Have a speaker (one of the characters above)
- Contain dialogue that advances the story
- Flow naturally to the next scene

Return as JSON:
{
  "scenes": [
    {
      "title": "Opening",
      "description": "Brief scene description",
      "speaker": "CharacterName",
      "dialogue": "What they say",
      "nextScene": "title of next scene or null if ending"
    }
  ]
}
  `;

  const result = await callModel(planner, {
    prompt,
    responseFormat: { type: 'json_object' },
  });

  return JSON.parse(result.content);
}
```

**Cost:** ~$0.003 (DeepSeek reasoning)

---

### Step 3: Convert Structure to Graph Operations

```typescript
// packages/domain-forge/src/assistant/workflows/structure-to-graph-ops.ts

export function storyStructureToGraphOps(
  structure: StoryStructure,
  characters: StoryCharacter[]
): ForgeGraphPatchOp[] {
  const ops: ForgeGraphPatchOp[] = [];
  const nodeIdMap = new Map<string, string>();  // scene title -> node ID

  // Create nodes for each scene
  structure.scenes.forEach((scene, index) => {
    const nodeId = `node_${Date.now()}_${index}`;
    nodeIdMap.set(scene.title, nodeId);

    // Find character by speaker name
    const character = characters.find(c => c.name === scene.speaker);

    ops.push({
      type: 'createNode',
      nodeType: 'CHARACTER',
      position: {
        x: 100 + (index * 250),  // Auto-layout horizontally
        y: 200,
      },
      data: {
        label: scene.title,
        speaker: scene.speaker,
        content: scene.dialogue,
        // Optional: Link to character portrait
        characterId: character?.id,
      },
      id: nodeId,
    });
  });

  // Create edges connecting scenes
  structure.scenes.forEach((scene) => {
    if (scene.nextScene) {
      const sourceId = nodeIdMap.get(scene.title);
      const targetId = nodeIdMap.get(scene.nextScene);

      if (sourceId && targetId) {
        ops.push({
          type: 'createEdge',
          source: sourceId,
          target: targetId,
        });
      }
    }
  });

  return ops;
}
```

---

### Step 4: Complete Workflow Tool

```typescript
// packages/domain-forge/src/assistant/tools/story-builder-tool.ts

{
  domain: 'forge',
  name: 'forge_createStoryFromPremise',
  description: 'Create a complete story with characters and dialogue graph from a premise. Generates characters with portraits, creates dialogue scenes, and builds connected graph.',

  parameters: {
    type: 'object',
    properties: {
      premise: {
        type: 'string',
        description: 'The story premise or concept (e.g., "A space pirate discovers a haunted ship")',
      },
      characterCount: {
        type: 'number',
        description: 'Number of main characters to create (default: 3)',
        default: 3,
      },
      sceneCount: {
        type: 'number',
        description: 'Number of dialogue scenes to create (default: 5)',
        default: 5,
      },
    },
    required: ['premise'],
  },

  execute: async (args, context) => {
    const { premise, characterCount = 3, sceneCount = 5 } = args;

    // Step 1: Create characters with portraits
    const characters = await createStoryCharacters(premise, characterCount);

    // Step 2: Generate story structure
    const structure = await generateStoryStructure(premise, characters, sceneCount);

    // Step 3: Convert to graph operations
    const graphOps = storyStructureToGraphOps(structure, characters);

    // Step 4: Apply operations to graph
    applyOperations(graphOps);

    // Step 5: Highlight all created nodes
    const nodeIds = graphOps
      .filter(op => op.type === 'createNode')
      .map(op => op.id!)
      .filter(Boolean);

    onAIHighlight({ entities: { 'forge.node': nodeIds } });

    return {
      success: true,
      message: `Created story with ${characters.length} characters and ${structure.scenes.length} scenes`,
      data: {
        characters: characters.map(c => ({
          name: c.name,
          personality: c.personality,
          portrait: c.portrait,
        })),
        scenes: structure.scenes.map(s => ({
          title: s.title,
          speaker: s.speaker,
        })),
        graphNodeCount: nodeIds.length,
      },
    };
  },

  render: (result) => {
    if (!result.success) return null;

    return (
      <div className="rounded-md border bg-muted/20 p-4 space-y-4">
        <div>
          <p className="text-sm font-medium">Story Created</p>
          <p className="text-xs text-muted-foreground mt-1">{result.message}</p>
        </div>

        {/* Character portraits */}
        <div>
          <p className="text-sm font-medium mb-2">Characters</p>
          <div className="flex gap-3">
            {result.data.characters.map((char) => (
              <div key={char.name} className="flex flex-col items-center gap-1">
                {char.portrait && (
                  <img
                    src={char.portrait}
                    alt={char.name}
                    className="w-16 h-16 rounded-full object-cover"
                  />
                )}
                <p className="text-xs font-medium">{char.name}</p>
                <p className="text-xs text-muted-foreground text-center">
                  {char.personality.slice(0, 40)}...
                </p>
              </div>
            ))}
          </div>
        </div>

        {/* Scene list */}
        <div>
          <p className="text-sm font-medium mb-2">Scenes</p>
          <ol className="space-y-2">
            {result.data.scenes.map((scene, i) => (
              <li key={i} className="text-xs">
                <span className="font-medium">{scene.title}</span>
                <span className="text-muted-foreground"> — {scene.speaker}</span>
              </li>
            ))}
          </ol>
        </div>

        <div className="text-xs text-muted-foreground">
          ✅ {result.data.graphNodeCount} nodes created and connected in graph
        </div>
      </div>
    );
  },
}
```

---

## User Experience

### User Input
```
"Create a story about a space pirate who discovers a haunted ship"
```

### AI Execution Flow

```
1. Creating characters... (2s)
   ✓ Captain "Red" Morgan (Space Pirate)
   ✓ Dr. Elena Voss (Scientist)
   ✓ Ghost of Captain Blackwood (Haunted entity)

2. Generating portraits... (3s)
   ✓ All characters have cinematic portraits

3. Planning story structure... (1s)
   ✓ 5 connected scenes planned

4. Building dialogue graph... (0.5s)
   ✓ 5 nodes created
   ✓ 4 edges connecting scenes

Total: ~6.5 seconds
```

### Result in Chat

```
┌─────────────────────────────────────┐
│ Story Created                       │
│ Created 3 characters and 5 scenes   │
├─────────────────────────────────────┤
│ Characters                          │
│ [Portrait] Captain Red Morgan       │
│ Gruff space pirate, haunted past    │
│                                     │
│ [Portrait] Dr. Elena Voss           │
│ Curious scientist, skeptical        │
│                                     │
│ [Portrait] Ghost of Blackwood       │
│ Vengeful spirit, seeks redemption   │
├─────────────────────────────────────┤
│ Scenes                              │
│ 1. The Discovery — Red              │
│ 2. First Contact — Elena            │
│ 3. Blackwood's Warning — Blackwood  │
│ 4. The Truth Revealed — Red         │
│ 5. Breaking the Curse — Elena       │
├─────────────────────────────────────┤
│ ✅ 5 nodes created and connected    │
└─────────────────────────────────────┘
```

### Result in Graph Editor

**Visual:**
```
[Captain Red]    [Dr. Voss]    [Blackwood]    [Captain Red]    [Dr. Voss]
   "The          "First         "Warning"        "Truth         "Breaking
  Discovery"     Contact"                       Revealed"      the Curse"
      └──────────────┴──────────────┴──────────────┴──────────────┘
                     (Connected with edges)
```

Each node shows:
- Speaker name (with portrait link)
- Scene title
- Dialogue content
- Connected to next scene

---

## Enhanced Version: With Player Choices

```typescript
// Add player choice nodes between character dialogue

export function addPlayerChoices(
  structure: StoryStructure,
  characters: StoryCharacter[]
): ForgeGraphPatchOp[] {
  const ops = storyStructureToGraphOps(structure, characters);

  // Add player choice nodes between scenes
  structure.scenes.forEach((scene, index) => {
    if (scene.nextScene && index < structure.scenes.length - 1) {
      const choiceNodeId = `choice_${index}`;

      // Create PLAYER node for choice
      ops.push({
        type: 'createNode',
        nodeType: 'PLAYER',
        position: {
          x: 100 + (index * 250) + 125,  // Between scenes
          y: 300,
        },
        data: {
          label: 'Player Choice',
          choices: [
            { id: 'a', text: 'Continue listening', nextNodeId: scene.nextScene },
            { id: 'b', text: 'Ask a question', nextNodeId: scene.nextScene },
          ],
        },
        id: choiceNodeId,
      });

      // Update edges to go through choice node
      // (Remove direct edge, add scene → choice → next)
    }
  });

  return ops;
}
```

---

## Cost Breakdown

| Operation | Model | Tokens | Cost |
|-----------|-------|--------|------|
| Create 3 characters (batched, cached) | claude-3.5-sonnet | 10K | $0.011 |
| Generate 3 portraits | flux-1.1-pro-ultra | 3 images | $0.06 |
| Plan story structure (5 scenes) | deepseek-reasoner | 2K | $0.003 |
| **Total** | | | **$0.074** |

**Per story with 3 characters + 5 scenes: ~$0.07**

**At scale:**
- 100 stories/month: $7
- 1000 stories/month: $70

---

## Yarn Spinner Export (Future)

Once story is created, export to `.yarn` format:

```text
title: The Discovery
tags: scene1
---
Captain Red: What in the seven hells is this ship?
Captain Red: Scanners are showing... nothing. Like it's a ghost.
===

title: First Contact
tags: scene2
---
Dr. Voss: Captain, I'm detecting anomalous readings from the cargo bay.
Dr. Voss: Should we investigate?
-> Continue
    <<jump The Truth>>
-> Retreat
    <<jump Ending>>
===
```

---

## When This is NOT Enough (LangGraph Territory)

### User wants iteration:
```
User: "Create a story about space pirates"
AI: [Creates 3 characters]
User: "Make the captain more gruff, add a robot companion"
AI: [Regenerates characters]  ← 🔴 Requires state machine to loop back
User: "Now build the dialogue"
AI: [Uses updated characters]
```

**Solution:** LangGraph with state checkpoints

### User wants multi-step approval:
```
1. Generate characters → User approves each one individually
2. Generate plot outline → User edits outline
3. Generate dialogue → Uses edited outline
4. User can go back to step 2 and revise
```

**Solution:** LangGraph with conditional edges

---

## Conclusion

**For "create a story with characters + dialogue graph":**

✅ **Vanilla workflow is perfect:**
- Single tool call: `forge_createStoryFromPremise`
- Characters created → Dialogue generated → Graph built
- Fast (6 seconds)
- Cheap ($0.07)
- Easy to debug

**Only add LangGraph if:**
- Users complain they can't iterate
- Need to resume across sessions
- Want fine-grained approval at each step

**Start simple. This covers 90% of use cases.**
