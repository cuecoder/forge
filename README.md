<div align="center">

# Forge

**Build production-grade RAG AI agents in minutes.**

Clone it, set up admin, ingest your knowledge, wire up tools, and ship an
embeddable chatbot — self-hosted, multi-tenant, open source.

</div>

---

## What is Forge?

Forge is an open-source platform for building, deploying, and embedding
retrieval-augmented (RAG) AI agents. You bring your documents and APIs; Forge
gives you a knowledge base, an agent builder, a tool builder, and a drop-in chat
widget.

A typical journey:

1. **Clone & set up admin** — one Docker command brings the whole stack up.
2. **Ingest knowledge** — upload documents, crawl a site, or connect S3 /
   SharePoint. Forge chunks, embeds, and stores it in pgvector.
3. **Build an agent** — pick a model, write a system prompt, attach knowledge
   bases, and select tools.
4. **Add tools** — turn any REST API (api-key / bearer / basic auth — no OAuth2
   flow) into a tool the agent can call. Ship-ready connector catalog included.
5. **Embed** — drop a single `<script>` tag on any site, or use the built-in
   Amazon-Q-style agent UI.

## Features

- **RAG knowledge bases** — documents, site crawl, S3, SharePoint → pgvector
  semantic search.
- **Agent builder** — system prompt + knowledge bases + tools + per-agent
  model config.
- **Dynamic API tools** — any REST API becomes a tool; pre-built connector
  templates.
- **Embeddable widget** — single-script chat bubble, isolated styles.
- **Multi-tenant** — full isolation with a dedicated Postgres database per
  tenant, auto-provisioned.
- **Pluggable LLMs** — Claude (default), OpenAI, Ollama / OpenAI-compatible.
- **One-command local dev** — Docker Compose with live HMR across every app.

## Tech Stack

| Layer | Choice |
|-------|--------|
| Backend | NestJS native monorepo (`api` + `worker` apps, `@app/*` libs) |
| Agent runtime | [Mastra](https://mastra.ai) — dynamic per-tenant agents & tools |
| Jobs | BullMQ + Redis |
| Database | PostgreSQL + [pgvector](https://github.com/pgvector/pgvector), Drizzle ORM |
| Tenancy | Database-per-tenant, dynamic connections, auto-provisioning |
| Frontend | Vite + React + TypeScript + Tailwind (Untitled-UI design language) |
| Widget | Vite + Preact → single `<script>` bundle |
| Shared contracts | `packages/contracts` (zod schemas + inferred types) |
| LLM | Pluggable: Anthropic / OpenAI / Ollama |
| Orchestration | Docker Compose (dev) · Terraform per-service (future) |

## Repository Layout

```
forge/
  backend/            NestJS native monorepo
    apps/api          REST/HTTP, auth, agent CRUD, chat (SSE/WS), tool exec
    apps/worker       BullMQ consumers: ingest, crawl, embed, provision
    libs/db           Drizzle schema (control + tenant), conn manager, migrations
    libs/llm          Pluggable LLM provider abstraction
    libs/agent        Mastra factory: per-tenant agents via RuntimeContext
    libs/rag          Chunking, splitters, retrieval, source connectors
    libs/tools        Dynamic API-tool engine (DB tool def → Mastra tool)
    libs/common       Config (zod env), guards, interceptors, utils
  web/                Vite + React admin console (Untitled-UI)
  widget/             Vite + Preact embeddable chat
  packages/contracts  Shared zod schemas + types (FE ↔ BE)
  docker/             Dockerfiles
  docs/               Documentation
```

See [docs/architecture.md](./docs/architecture.md) for the full design and the rationale
behind each decision.

## Quickstart (local dev)

> Requires Docker + Docker Compose. Node/pnpm only needed for non-containerized work.

```bash
git clone <your-fork-url> forge && cd forge
cp .env.example .env            # set ANTHROPIC_API_KEY etc.
docker compose -f docker-compose.yml -f docker-compose.dev.yml up
```

This starts Postgres (with pgvector), Redis, `api`, `worker`, `web`, and
`widget` with live HMR. Open the admin console and complete the setup wizard.

Full instructions: [docs/getting-started.md](./docs/getting-started.md) ·
[docs/self-hosting.md](./docs/self-hosting.md).

## Documentation

| Doc | What it covers |
|-----|----------------|
| [Architecture](./docs/architecture.md) | System design, every locked decision |
| [Getting Started](./docs/getting-started.md) | Run it locally, first agent |
| [Self-Hosting](./docs/self-hosting.md) | Compose, env, production notes |
| [Tenancy](./docs/tenancy.md) | DB-per-tenant, connections, provisioning |
| [RAG](./docs/rag.md) | Ingestion, chunking, embedding, retrieval |
| [Agents](./docs/agents.md) | Mastra factory, runtime, chat loop |
| [Tools](./docs/tools.md) | Dynamic API tools, connectors |
| [LLM Providers](./docs/llm-providers.md) | Pluggable provider config |
| [Widget Embed](./docs/widget-embed.md) | Embeddable chat on any site |
| [Contributing](./CONTRIBUTING.md) | Dev setup, conventions, PR flow |

## Status

Early development. Architecture is locked (see ARCHITECTURE.md); build is
phased — foundation → DB core → tenancy → UI → RAG → agents → tools → widget →
hardening.

## License

[Apache 2.0](./LICENSE) © cuecoder
