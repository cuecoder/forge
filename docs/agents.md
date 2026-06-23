# Agents

Agents are the heart of Forge: a system prompt + knowledge bases + tools + model
config, run through [Mastra](https://mastra.ai). This document covers the agent
factory, runtime context, and the chat loop.

> Lives in `@app/agent`. Wraps [`@app/llm`](./llm-providers.md),
> [`@app/rag`](./rag.md), and [`@app/tools`](./tools.md).

## What an agent is

Stored in the tenant DB (`agents` table):

| Field | Purpose |
|-------|---------|
| system prompt | the agent's instructions |
| model config | provider + model (per-agent override of the platform default) |
| knowledge bases | which KBs it can retrieve from |
| tools | which API tools it can call (`agent_tools` join) |

## Why Mastra (and why thin)

Mastra is a TS-native agent runtime that owns the **tool-calling loop** so we
don't hand-roll it. But Forge is multi-tenant with **dynamic** agents and tools
loaded from per-tenant databases — a static, globally-registered agent/tool
registry would fight that.

So the rule: **build agents dynamically per request/job; never statically
register them.** One Mastra instance per process; agents are constructed on the
fly.

## The agent factory

```ts
createTenantAgent({ tenantId, agentRow, runtimeContext })
```

It assembles:

- **model** — resolved from `@app/llm` using the agent's provider/model config.
- **instructions** — the agent's system prompt (from `agentRow`).
- **tools** — the tenant's API-tool defs (from the tenant DB) compiled by
  `@app/tools` (`toMastraTool`) into Mastra tools, scoped to this agent.
- **retrieval** — a Mastra step/tool that queries tenant pgvector via `@app/rag`,
  filtered to the agent's selected knowledge bases.

### RuntimeContext

`RuntimeContext` carries:

- `tenantId`
- the resolved **tenant Drizzle client**

…so every tool execution and every retrieval call hits the **correct tenant DB**.
This is how dynamic per-tenant isolation flows into the agent loop.

## The chat loop

```
user message
   │
   ▼
resolve tenant + agent ──▶ createTenantAgent(...)
   │
   ▼
Mastra runs the loop:
   retrieve RAG context  ─┐
   call LLM with tools     │  repeat until the model stops
   execute tool call(s)    │  (tool results fed back in)
   ◀──────────────────────┘
   │
   ▼
stream tokens ──▶ SSE / WebSocket  (backend/apps/api)
```

- Streaming uses Mastra's `stream()`, piped to SSE/WebSocket in
  `backend/apps/api`.
- Conversations + messages are persisted in the tenant DB.

## Public vs admin chat

- **Admin UI** — authenticated chat in `web/`, full agent management.
- **Public widget** — a separate public chat endpoint, addressed by a **scoped
  token per agent**, rate-limited and CORS-aware. See
  [Widget Embed](./widget-embed.md).

## Future: branchy workflows

If agent flows grow genuinely branchy (multi-agent orchestration, human-in-the-
loop, complex graphs), LangGraph can be added **for those graphs only** — not as
the spine. The default simple retrieve → tool → respond loop stays on Mastra.
