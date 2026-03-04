---
title: Plan — Assistant UI / CopilotKit Replacement (archived)
created: 2026-02-11
updated: 2026-02-11
---

You want to replace CopilotKit with a custom agent system built on Assistant UI and OpenRouter because:

CopilotKit limitations: The builtin agent and Responses V2 spec requirements are too restrictive
UI-first approach: Assistant UI's Frame API provides better component integration
Flexibility: Direct chat completions API with OpenRouter removes model compatibility checking
MCP integration: Need to expose Studio editors as MCP apps for external control
Better UX: Components appear in chat, full review workflows with audio, cross-editor context

Your vision: "Create a character" from app-level chat → full character creation in chat UI with components → review/audio → save → later edit in Character tab without chat. Same pattern for dialogue, storylets, and all editors.
Architecture Overview
Core Principles

Preserve domain logic: domain-forge, domain-character operations stay intact
UI-first: Assistant UI Frame API for in-chat components
Tool-based: Same action/tool pattern as CopilotKit but using Assistant UI conventions
MCP-compatible: Tools map cleanly to MCP spec for external control
Incremental migration: Both systems coexist during transition

New Package Structure
Create @forge/assistant-runtime - Core Assistant UI integration:
packages/assistant-runtime/
├── src/
│   ├── runtime/
│   │   ├── forge-runtime.ts          # OpenRouter adapter for Assistant UI
│   │   ├── streaming.ts              # SSE streaming handler
│   │   └── types.ts
│   ├── agents/
│   │   ├── agent-registry.ts         # Global agent registry
│   │   ├── base-agent.ts             # Base agent class
│   │   └── types.ts
│   ├── tools/
│   │   ├── tool-registry.ts          # Tool registration system
│   │   ├── tool-executor.ts          # Tool execution with context
│   │   └── types.ts
│   ├── components/
│   │   ├── AssistantProvider.tsx     # Root provider
│   │   ├── AssistantChat.tsx         # Chat UI (sidebar/panel modes)
│   │   # (Use tool-ui Plan from packages/shared/components/tool-ui/plan)
│   │   └── ToolMessage.tsx           # Tool result rendering
│   └── hooks/
│       ├── use-tool-registration.ts  # Register tools hook
│       └── use-plan-workflow.ts      # Plan workflow orchestration
Update packages/shared/src/shared/assistant/ - Domain integration:
assistant/
├── domain-contract.ts            # DomainAssistantContract interface
├── use-domain-assistant.ts       # Main integration hook (replaces useDomainCopilot)
├── tool-builder.ts               # Helper to build tools
└── types.ts
Create packages/mcp-studio/ - MCP server:
mcp-studio/
├── src/
│   ├── server.ts                 # MCP server implementation
│   ├── editors/
│   │   ├── forge-mcp.ts          # Forge editor MCP app
│   │   ├── character-mcp.ts      # Character editor MCP app
│   │   └── index.ts
│   └── auth/
│       └── token-provider.ts     # API token auth
Domain Integration Pattern
Contract Interface
Every domain implements DomainAssistantContract:
typescriptinterface DomainAssistantContract {
  domain: string;
  getContextSnapshot(): DomainContextSnapshot;
  getInstructions(): string;
  createTools(): DomainTool[];
  getSuggestions(): Array<{ title: string; message: string }>;
  onHighlight(entities: Record<string, string[]>): void;
  clearHighlights(): void;
}

interface DomainTool {
  domain: string;
  name: string;
  description: string;
  parameters: {
    type: 'object';
    properties: Record<string, any>;
    required?: string[];
  };
  execute: (args: any, context: DomainToolContext) => Promise<any>;
  render?: (result: any) => React.ReactNode;
}
Forge Domain Implementation
New file: packages/domain-forge/src/assistant/index.ts
typescriptexport function useForgeAssistantContract(deps: ForgeAssistantDeps): DomainAssistantContract {
  return useMemo(() => ({
    domain: 'forge',

    getContextSnapshot: () => buildForgeContext({
      graph: deps.graph,
      selection: deps.selection,
      isDirty: deps.isDirty,
    }),

    getInstructions: () =>
      'You are helping edit a dialogue graph. Node types: CHARACTER, PLAYER, CONDITIONAL. ' +
      'Use forge_* tools. For complex changes, use forge_createPlan for preview.',

    createTools: () => createForgeTools(deps),

    getSuggestions: () => getForgeSuggestions({ graph: deps.graph, selection: deps.selection }),

    onHighlight: deps.onHighlight,
    clearHighlights: deps.clearHighlights,
  }), [deps]);
}
New file: packages/domain-forge/src/assistant/tools.ts
Migrate from copilot/actions.ts, example tool:
typescriptexport function createForgeTools(deps: ForgeToolsDeps): DomainTool[] {
  return [
    {
      domain: 'forge',
      name: 'forge_createNode',
      description: 'Create a new node in the dialogue graph',
      parameters: {
        type: 'object',
        properties: {
          nodeType: { type: 'string', enum: ['CHARACTER', 'PLAYER', 'CONDITIONAL'] },
          label: { type: 'string' },
          content: { type: 'string' },
          speaker: { type: 'string' },
          x: { type: 'number' },
          y: { type: 'number' },
        },
        required: ['nodeType', 'label'],
      },
      execute: async (args, context) => {
        const nodeId = `node_${Date.now()}_${Math.random().toString(36).slice(2, 11)}`;

        deps.applyOperations([{
          type: 'createNode',
          nodeType: args.nodeType,
          position: { x: args.x ?? Math.random() * 500, y: args.y ?? Math.random() * 500 },
          data: { label: args.label, content: args.content, speaker: args.speaker },
          id: nodeId,
        }]);

        deps.onHighlight({ 'forge.node': [nodeId] });

        return { success: true, message: `Created ${args.nodeType} node: ${args.label}`, nodeId };
      },
      render: (result) => (
        <div className="rounded-md border bg-muted/20 p-3">
          <p className="text-sm font-medium">Node Created</p>
          <p className="text-xs text-muted-foreground mt-1">{result.message}</p>
        </div>
      ),
    },
    // ... forge_updateNode, forge_deleteNode, forge_createEdge, forge_getGraph, etc.
    {
      domain: 'forge',
      name: 'forge_createPlan',
      description: 'Create a plan for complex graph changes without applying them',
      parameters: {
        type: 'object',
        properties: {
          goal: { type: 'string', description: 'What to achieve' },
        },
        required: ['goal'],
      },
      execute: async (args, context) => {
        const graph = deps.getGraph();
        const graphSummary = graph ? {
          title: graph.title,
          nodeCount: graph.flow.nodes.length,
          nodes: graph.flow.nodes.map(n => ({ id: n.id, type: n.data?.type, label: n.data?.label })),
        } : {};

        const { steps } = await deps.createPlan(args.goal, graphSummary);
        return { success: true, steps, goal: args.goal };
      },
      render: (result) => {
        if (!result.success || !result.steps) return null;
        return (
          <Plan
            id={result.id ?? 'plan-1'}
            title={result.goal}
            todos={result.steps.map((s, i) => ({ id: String(i), label: s.type ?? s.title, status: 'pending', description: s.description }))}
            responseActions={[{ id: 'approve', label: 'Apply Plan' }, { id: 'revise', label: 'Request Changes', variant: 'secondary' }]}
            onResponseAction={(id) => { if (id === 'approve' && deps.executePlan) deps.executePlan(result.steps); }}
          />
        );
      },
    },
  ];
}
Character Domain Implementation
New file: packages/domain-character/src/assistant/index.ts
Similar pattern to Forge:
typescriptexport function useCharacterAssistantContract(deps: CharacterAssistantDeps): DomainAssistantContract {
  return useMemo(() => ({
    domain: 'character',
    getContextSnapshot: () => buildCharacterContext(deps),
    getInstructions: () => 'You help create and manage characters...',
    createTools: () => createCharacterTools(deps),
    getSuggestions: () => getCharacterSuggestions(deps),
    onHighlight: deps.onHighlight,
    clearHighlights: deps.clearHighlights,
  }), [deps]);
}
New file: packages/domain-character/src/assistant/tools.ts
Character-specific tools:
typescriptexport function createCharacterTools(deps: CharacterToolsDeps): DomainTool[] {
  return [
    {
      domain: 'character',
      name: 'character_create',
      description: 'Create a new character with all details',
      parameters: {
        type: 'object',
        properties: {
          name: { type: 'string' },
          description: { type: 'string' },
          personality: { type: 'string' },
          background: { type: 'string' },
          voice: { type: 'string' },
          appearance: { type: 'string' },
        },
        required: ['name'],
      },
      execute: async (args, context) => {
        const character = await deps.createCharacter(args);
        deps.onHighlight({ 'character': [character.id] });
        return { success: true, character };
      },
      render: (result) => (
        <CharacterCreatedCard character={result.character} />
      ),
    },
    // ... character_update, character_generatePortrait, character_generateVoice, etc.
  ];
}
Runtime Implementation
API Route
New file: apps/studio/app/api/assistant/route.ts
typescriptimport { streamText } from 'ai';
import { createOpenAI } from '@ai-sdk/openai';
import { getPayload } from 'payload';
import { requireAiRequestAuth } from '@/lib/server/api-keys';
import { recordAiUsageEvent } from '@/lib/server/ai-usage';
import { getOpenRouterConfig } from '@/lib/openrouter-config';
import { getPersistedModelIdForProvider } from '@/lib/model-router/persistence';

export async function POST(req: Request) {
  const payload = await getPayload({ config });
  const authContext = await requireAiRequestAuth(payload, req, 'ai.chat');

  if (!authContext) {
    return new Response('Unauthorized', { status: 401 });
  }

  const body = await req.json();
  const config = getOpenRouterConfig();

  // No Responses V2 checking - just use chat completions
  const modelId = await getPersistedModelIdForProvider(req, 'assistant') ?? 'openai/gpt-4o-mini';

  const openai = createOpenAI({
    apiKey: config.apiKey,
    baseURL: 'https://openrouter.ai/api/v1',
    headers: {
      'HTTP-Referer': 'https://forge-studio.local',
      'X-Title': 'Forge Studio',
    },
  });

  try {
    const result = await streamText({
      model: openai(modelId),
      messages: body.messages,
      tools: body.tools,
      maxSteps: 10,
    });

    void recordAiUsageEvent({
      requestId: crypto.randomUUID(),
      authContext,
      provider: 'openrouter',
      model: modelId,
      routeKey: 'assistant',
      status: 'success',
    });

    return result.toDataStreamResponse();
  } catch (error) {
    void recordAiUsageEvent({
      requestId: crypto.randomUUID(),
      authContext,
      provider: 'openrouter',
      model: modelId,
      routeKey: 'assistant',
      status: 'error',
      errorMessage: error.message,
    });
    throw error;
  }
}
Assistant Provider Component
New file: packages/assistant-runtime/src/components/AssistantProvider.tsx
typescript'use client';

import { AssistantRuntimeProvider } from '@assistant-ui/react';
import { useLocalRuntime } from '@assistant-ui/react';

export interface AssistantProviderProps {
  runtimeUrl: string;
  children: React.ReactNode;
}

export function AssistantProvider({ runtimeUrl, children }: AssistantProviderProps) {
  const runtime = useLocalRuntime({
    api: runtimeUrl,
  });

  return (
    <AssistantRuntimeProvider runtime={runtime}>
      {children}
    </AssistantRuntimeProvider>
  );
}
Chat UI Component
New file: packages/assistant-runtime/src/components/AssistantChat.tsx
typescript'use client';

import { Thread } from '@assistant-ui/react';
import { cn } from '@forge/shared/lib/utils';

export interface AssistantChatProps {
  mode?: 'sidebar' | 'panel' | 'modal';
  className?: string;
}

export function AssistantChat({ mode = 'sidebar', className }: AssistantChatProps) {
  return (
    <div className={cn(
      'flex flex-col h-full bg-background',
      mode === 'sidebar' && 'w-96 border-l',
      mode === 'panel' && 'w-full',
      className
    )}>
      <Thread
        welcome={{
          message: 'How can I help with your workspace?',
        }}
      />
    </div>
  );
}
Plan (tool-ui)
Use tool-ui Plan from packages/shared/components/tool-ui/plan. See tool-ui.com/docs/plan. Schema: id, title, description, todos (id, label, status, description). Use responseActions for Apply Plan / Request Changes. For forge_createPlan, backend returns { id, title, description, todos }; register via render_plan in assistant-tools.tsx.
Domain Hook Integration
New file: packages/shared/src/shared/assistant/use-domain-assistant.ts
typescript'use client';

import { useEffect } from 'react';
import { useAssistantContext } from '@assistant-ui/react';
import type { DomainAssistantContract } from './domain-contract';

export function useDomainAssistant(
  contract: DomainAssistantContract,
  options?: { enabled?: boolean }
) {
  const { enabled = true } = options ?? {};
  const assistant = useAssistantContext();

  // Register tools
  useEffect(() => {
    if (!enabled) return;

    const tools = contract.createTools().map(tool => ({
      name: tool.name,
      description: tool.description,
      parameters: tool.parameters,
      execute: async (args: any) => {
        const snapshot = contract.getContextSnapshot();
        return await tool.execute(args, {
          snapshot,
          emit: (event) => assistant.emit?.(event),
        });
      },
    }));

    assistant.registerTools(tools);

    return () => {
      assistant.unregisterTools(tools.map(t => t.name));
    };
  }, [enabled, contract, assistant]);

  // Update context
  useEffect(() => {
    if (!enabled) return;

    const snapshot = contract.getContextSnapshot();
    const instructions = contract.getInstructions();

    assistant.setContext({
      domain: snapshot.domain,
      snapshot,
      instructions,
    });
  }, [enabled, contract, assistant]);

  // Update suggestions
  useEffect(() => {
    if (!enabled) return;

    const suggestions = contract.getSuggestions();
    assistant.setSuggestions(suggestions);
  }, [enabled, contract, assistant]);

  return {
    onHighlight: contract.onHighlight,
    clearHighlights: contract.clearHighlights,
  };
}
Editor Integration
Dialogue Editor
Update: apps/studio/components/editors/DialogueEditor.tsx
typescript'use client';

import { AssistantProvider } from '@forge/assistant-runtime';
import { useForgeAssistantContract } from '@forge/domain-forge/assistant';
import { useDomainAssistant } from '@forge/shared/assistant';
import { AssistantChat } from '@forge/assistant-runtime';

export function DialogueEditor({ projectId, graphId }) {
  const [graph, setGraph] = useState<ForgeGraphDoc | null>(null);
  const [selection, setSelection] = useState<Selection | null>(null);
  const [highlights, setHighlights] = useState({});

  const applyOperations = (ops: ForgeGraphPatchOp[]) => {
    // Apply to graph
  };

  const createPlan = async (goal: string, context: any) => {
    const res = await fetch('/api/forge/plan', {
      method: 'POST',
      body: JSON.stringify({ goal, context }),
    });
    return await res.json();
  };

  const executePlan = async (steps: any[]) => {
    for (const step of steps) {
      // Execute each step
    }
  };

  // Assistant integration
  const assistantContract = useForgeAssistantContract({
    graph,
    selection,
    isDirty: false,
    applyOperations,
    openOverlay: (id, payload) => {},
    revealSelection: () => {},
    onHighlight: (entities) => setHighlights(entities),
    clearHighlights: () => setHighlights({}),
    createPlan,
    executePlan,
    commitGraph: async () => {
      // Save to server
    },
  });

  useDomainAssistant(assistantContract);

  return (
    <AssistantProvider runtimeUrl="/api/assistant">
      <WorkspaceShell>
        <EditorDockLayout>
          <EditorDockLayout.Main>
            <ForgeCanvas
              graph={graph}
              selection={selection}
              highlights={highlights}
              onSelectionChange={setSelection}
            />
          </EditorDockLayout.Main>

          <EditorDockLayout.Right>
            <EditorDockLayout.Panel id="assistant" title="AI Assistant">
              <AssistantChat mode="panel" />
            </EditorDockLayout.Panel>
          </EditorDockLayout.Right>
        </EditorDockLayout>
      </WorkspaceShell>
    </AssistantProvider>
  );
}
Agent System
Base Agent
New file: packages/assistant-runtime/src/agents/base-agent.ts
typescriptexport interface AgentConfig {
  id: string;
  name: string;
  description: string;
  instructions: string;
  tools: DomainTool[];
  scopes: string[];
  maxSteps?: number;
}

export abstract class BaseAgent {
  constructor(protected config: AgentConfig) {}

  get id() { return this.config.id; }
  get name() { return this.config.name; }
  get instructions() { return this.config.instructions; }
  get tools() { return this.config.tools; }

  canHandleScope(scope: string): boolean {
    return this.config.scopes.includes(scope) || this.config.scopes.includes('*');
  }

  abstract execute(context: AgentContext): Promise<AgentResult>;
}
Agent Registry
New file: packages/assistant-runtime/src/agents/agent-registry.ts
typescriptexport class AgentRegistry {
  private agents = new Map<string, BaseAgent>();

  register(agent: BaseAgent) {
    this.agents.set(agent.id, agent);
  }

  get(id: string): BaseAgent | undefined {
    return this.agents.get(id);
  }

  findForScope(scope: string): BaseAgent[] {
    return Array.from(this.agents.values())
      .filter(agent => agent.canHandleScope(scope));
  }

  getAllTools(scope?: string): DomainTool[] {
    const agents = scope
      ? this.findForScope(scope)
      : Array.from(this.agents.values());
    return agents.flatMap(agent => agent.tools);
  }
}

export const globalAgentRegistry = new AgentRegistry();
Forge Agent
New file: packages/domain-forge/src/assistant/forge-agent.ts
typescriptimport { BaseAgent } from '@forge/assistant-runtime';

export class ForgeAgent extends BaseAgent {
  constructor(tools: DomainTool[]) {
    super({
      id: 'forge-agent',
      name: 'Forge Dialogue Editor Agent',
      description: 'Assists with dialogue graph editing',
      instructions:
        'You help edit dialogue graphs. Use forge_* tools. ' +
        'For complex changes, use forge_createPlan first.',
      tools,
      scopes: ['forge'],
      maxSteps: 10,
    });
  }

  async execute(context: AgentContext) {
    // Custom orchestration logic
    return { success: true, message: 'Agent execution completed' };
  }
}
App-Level Agent
New file: apps/studio/lib/agents/app-agent.ts
typescriptimport { BaseAgent, globalAgentRegistry } from '@forge/assistant-runtime';

export class AppAgent extends BaseAgent {
  constructor(tools: DomainTool[]) {
    super({
      id: 'app-agent',
      name: 'Forge Studio App Agent',
      description: 'Top-level workspace management',
      instructions:
        'You help navigate Forge Studio. Delegate to domain agents when needed.',
      tools,
      scopes: ['*'],
    });
  }

  async execute(context: AgentContext) {
    const { domain } = context;

    // Delegate to domain agents
    if (domain === 'forge') {
      const forgeAgent = globalAgentRegistry.get('forge-agent');
      return await forgeAgent?.execute(context);
    }

    if (domain === 'character') {
      const characterAgent = globalAgentRegistry.get('character-agent');
      return await characterAgent?.execute(context);
    }

    // Handle app-level operations
    return { success: true, message: 'App operation completed' };
  }
}
MCP Integration
MCP Server
New file: packages/mcp-studio/src/server.ts
typescriptimport { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import { globalAgentRegistry } from '@forge/assistant-runtime';

export class ForgeStudioMCPServer {
  private server: Server;

  constructor() {
    this.server = new Server(
      { name: 'forge-studio', version: '1.0.0' },
      { capabilities: { tools: {} } }
    );

    this.registerHandlers();
  }

  private registerHandlers() {
    // List tools
    this.server.setRequestHandler('tools/list', async () => {
      const tools = globalAgentRegistry.getAllTools();
      return {
        tools: tools.map(tool => ({
          name: tool.name,
          description: tool.description,
          inputSchema: {
            type: 'object',
            properties: tool.parameters.properties,
            required: tool.parameters.required,
          },
        })),
      };
    });

    // Execute tool
    this.server.setRequestHandler('tools/call', async (request) => {
      const { name, arguments: args } = request.params;
      const tools = globalAgentRegistry.getAllTools();
      const tool = tools.find(t => t.name === name);

      if (!tool) {
        throw new Error(`Tool not found: ${name}`);
      }

      const result = await tool.execute(args, {
        snapshot: {},
        emit: (event) => {},
      });

      return {
        content: [{ type: 'text', text: JSON.stringify(result) }],
      };
    });
  }

  async start() {
    const transport = new StdioServerTransport();
    await this.server.connect(transport);
    console.error('Forge Studio MCP server running');
  }
}
Editor MCP Apps
New file: packages/mcp-studio/src/editors/forge-mcp.ts
typescriptexport class ForgeMCPApp {
  constructor(private agentRegistry: AgentRegistry) {}

  async listTools() {
    const forgeAgent = this.agentRegistry.findForScope('forge')[0];
    return forgeAgent?.tools ?? [];
  }

  async executeTool(name: string, args: any) {
    const tools = await this.listTools();
    const tool = tools.find(t => t.name === name);

    if (!tool) {
      throw new Error(`Tool not found: ${name}`);
    }

    return await tool.execute(args, {
      snapshot: {},
      emit: () => {},
    });
  }

  getCapabilities() {
    return {
      editor: 'forge',
      supports: ['dialogue', 'nodes', 'edges', 'plans'],
    };
  }
}
Migration Strategy
Phase 1: Foundation (Week 1)

Create @forge/assistant-runtime package
Implement OpenRouter adapter (no Responses V2 checking)
Create /api/assistant route
Build AssistantProvider and AssistantChat components
Test basic streaming and tool execution

Test: Simple test page with Assistant UI panel
Phase 2: Tool System (Week 1-2)

Implement tool registry and execution
Create DomainAssistantContract interface
Build useDomainAssistant hook
Migrate 2-3 simple forge tools (forge_getGraph, forge_createNode)
Test tool rendering in chat

Coexistence pattern:
typescript// Both systems run simultaneously during migration
export function ForgeEditor() {
  // New Assistant UI
  const assistantContract = useForgeAssistantContract(deps);
  useDomainAssistant(assistantContract, { enabled: USE_ASSISTANT_UI });

  // Old CopilotKit
  const copilotContract = useForgeContract(deps);
  useDomainCopilot(copilotContract, { toolsEnabled: !USE_ASSISTANT_UI });

  return <EditorLayout>...</EditorLayout>;
}
Phase 3: Forge Domain Migration (Week 2-3)

Migrate all forge tools to Assistant UI format
Implement plan workflow with tool-ui Plan
Test highlights and visual feedback
Migrate generative UI components
A/B test both systems

Tool migration pattern: Copy from domain-forge/src/copilot/actions.ts → domain-forge/src/assistant/tools.ts, adapt signatures
Phase 4: Character Domain (Week 3-4)

Create CharacterAgent
Migrate character tools
Question Flow for character creation wizard; ElevenLabs + tool-ui Audio for voice
Test cross-domain context (create character from app-level chat)
Character creation in chat with review workflow

Phase 5: Agent System (Week 4-5)

Implement AgentRegistry
Create AppAgent
Build agent delegation logic
Test multi-agent coordination

Phase 6: MCP Integration (Week 5-6)

Create @forge/mcp-studio package
Implement MCP server with tool mapping
Build editor MCP apps (Forge, Character)
Test with external MCP clients (Cursor, Claude Desktop)

Phase 7: Cleanup (Week 6-7)

Remove CopilotKit dependencies
Delete copilot/ directories
Update documentation
Performance optimization

Rollback Plan

Feature flag: USE_ASSISTANT_UI environment variable
Both systems coexist, toggle between them
Can migrate domain-by-domain independently
Keep CopilotKit until full migration validated

Critical Files to Create/Modify
New Files

packages/assistant-runtime/src/runtime/forge-runtime.ts - OpenRouter adapter
packages/assistant-runtime/src/components/AssistantProvider.tsx - Root provider
packages/assistant-runtime/src/components/AssistantChat.tsx - Chat UI
(Use tool-ui Plan from packages/shared/components/tool-ui/plan)
packages/assistant-runtime/src/agents/agent-registry.ts - Agent system
packages/shared/src/shared/assistant/domain-contract.ts - Contract interface
packages/shared/src/shared/assistant/use-domain-assistant.ts - Integration hook
packages/domain-forge/src/assistant/index.ts - Forge contract factory
packages/domain-forge/src/assistant/tools.ts - Forge tools
packages/domain-forge/src/assistant/forge-agent.ts - Forge agent
packages/domain-character/src/assistant/index.ts - Character contract
packages/domain-character/src/assistant/tools.ts - Character tools
packages/mcp-studio/src/server.ts - MCP server
apps/studio/app/api/assistant/route.ts - Assistant API endpoint
apps/studio/lib/agents/app-agent.ts - App-level agent

Files to Modify

apps/studio/components/editors/DialogueEditor.tsx - Add Assistant UI integration
apps/studio/components/editors/CharacterEditor.tsx - Add Assistant UI integration
apps/studio/package.json - Add @assistant-ui/react dependency

Files to Reference (Don't Change)

packages/domain-forge/src/copilot/actions.ts - Source for tool migration
packages/domain-character/src/copilot/actions.ts - Source for character tools
apps/studio/app/api/copilotkit/handler.ts - OpenRouter config pattern
packages/shared/src/shared/copilot/types.ts - Contract interface pattern

Verification
After implementation:

Basic chat: Open Dialogue editor, chat with Assistant, see responses
Tool execution: "Create a character node" → node appears, highlighted
Plan workflow: "Add 3 connected dialogue nodes" → plan appears → click Apply → nodes created
Cross-editor: "Create a character named Alice" from app-level → character created → visible in Character tab
MCP: External MCP client can list tools and execute forge_createNode
Coexistence: Toggle USE_ASSISTANT_UI flag → both systems work independently

Dependencies
Add to apps/studio/package.json:
json{
  "@assistant-ui/react": "^0.5.0",
  "@ai-sdk/openai": "^3.0.25" (already installed),
  "ai": "^3.0.0"
}
Add to packages/mcp-studio/package.json:
json{
  "@modelcontextprotocol/sdk": "^1.0.0"
}
Chat Components (Tool UI, Assistant UI)

Tool UI and Assistant UI are **chat components**—they render inside the conversational thread. They are not editor panels. Editor panels (GraphPanel, Inspector, etc.) are separate; chat components appear as tool results and message content within the Thread.

- **Tool UI**: Plan, Progress Tracker, Question Flow, Image, Video, Audio, Approval Card, Code Block, Data Table, Terminal, Chart, etc. From [tool-ui.com](https://www.tool-ui.com). Each has a `render_*` tool name registered in assistant-tools.tsx.
- **Assistant UI**: Thread, ThreadList, ToolFallback, ToolGroup, MarkdownText, ComposerAttachments, etc. From @assistant-ui/react and our shared assistant-ui folder.

Component Alignment (tool-ui.com)

- **Plan**: Use tool-ui Plan with `responseActions` (Approve Plan, Request Changes). Schema: `id`, `title`, `description`, `todos` (id, label, status, description). For forge_createPlan: backend returns `{ id, title, description, todos }`; frontend renders via render_plan with responseActions. Replace custom PlanReviewCard with tool-ui Plan.
- **Progress Tracker**: For in-progress multi-step execution in chat. When executing a plan (e.g. "Add 5 dialogue nodes"), show Progress Tracker—"Step 1 of 5: Creating node A", etc. Steps: pending, in-progress, completed, failed. Elapsed time badge; responseActions (Cancel); receipt state.
- **Question Flow**: Multi-step guided questions (progressive or upfront). Single or multi-select per step. Usage: character creation wizard, project setup, import configuration. Example: "Create a character" → Question Flow for name, personality, voice, etc.
- **Image / Video**: tool-ui components for media in chat. Character portraits, generated media, link previews. Schema: src, alt, title, description, ratio; Video adds poster, durationMs.
- **Audio**: ElevenLabs for generation (character voices, TTS). Tool-ui Audio or custom player for playback in chat after generation. Tools: character_generateVoice, writer_readAloud → return audio URL → render with tool-ui Audio.

Diff Strategy Per Domain

**Graph (Forge):** Do NOT accept raw JSON (nodes/edges) from the model. Caveats: duplicate IDs, invalid references, malformed structure. Use tool-based operations (forge_createNode, forge_createEdge, forge_updateNode, forge_deleteNode) that assign IDs automatically and validate structure. Patch envelope: PatchEnvelope<'reactflow', ForgeGraphPatchOp[]>. Diff UI: GraphDiffPanel—before/after ops summary in chat (e.g. "Add 3 nodes, 2 edges"). Use DiffPreview or custom layout.

**Writer (Lexical):** Notion-style block-level operations. Lexical supports $insertNodes, $removeNodes, $setSelection. Tools: writer_insertBlock, writer_replaceBlock, writer_deleteBlock, writer_getSelection—not writer_setFullContent. Patch kind: PatchEnvelope<'lexical', LexicalPatchOp[]>. Diff UI: LexicalDiffPanel—block-level before/after. Use DiffPreview with block rendering.

Domain-Specific Diff Panels

- **GraphDiffPanel**: For Forge. Shows current vs proposed (ops summary or mini-graph). Mounted in chat when forge_createPlan returns and user clicks Preview or similar.
- **LexicalDiffPanel**: For Writer. Block-level before/after. Uses DiffPreview with block rendering or custom.
- **DiffPreview** (existing): Generic before/after in copilot/generative-ui. Can be extended with domain prop for custom rendering.

Key Design Decisions

No Responses V2: Use standard chat completions API, removing model compatibility checking
Preserve domain operations: applyOperations, graph operations stay unchanged
Tool = Action: Same concept, different wrapper (DomainTool vs CopilotActionConfig)
Frame API for UI: Assistant UI's Frame API for in-chat components (Plan, Progress Tracker, etc.)
Agent delegation: App agent delegates to domain agents based on context
MCP-first tools: Tools designed to map cleanly to MCP spec
Incremental migration: Feature flag allows both systems to coexist