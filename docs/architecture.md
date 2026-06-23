# Forge — Architecture

This document describes Forge's system design and the rationale behind each
locked decision. It is the canonical reference for how the pieces fit together.

- [Overview](#overview)
- [Locked Decisions](#locked-decisions)
- [Repository Shape](#repository-shape)
- [Backend (NestJS monorepo)](#backend-nestjs-monorepo)
- [Database Architecture](#database-architecture)
- [Multi-Tenancy](#multi-tenancy)
- [RAG Pipeline](#rag-pipeline)
- [LLM Abstraction](#llm-abstraction)
- [Agent Core (Mastra)](#agent-core-mastra)
- [Dynamic API Tools](#dynamic-api-tools)
- [Chat & Streaming](#chat--streaming)
- [Frontend](#frontend)
- [Embeddable Widget](#embeddable-widget)
- [Shared Contracts](#shared-contracts)
- [Local Development](#local-development)
- [Deployment (future)](#deployment-future)
- [Open Items](#open-items)

---

## Overview

Forge lets a user clone the project, set up an admin account, ingest knowledge
(documents / site crawl / S3 / SharePoint) into pgvector, create agents that
combine knowledge bases and tools, expose any REST API (non-OAuth2) as a tool,
and ship an embeddable chatbot widget plus an Amazon-Q-style agent UI.

It is **multi-tenant with full database isolation** — each tenant gets a
dedicated Postgres database, provisioned automatically.

```
                         ┌──────────────────────────────┐
   end-user site         │            Forge             │
   ┌───────────┐  embed  │  ┌────────┐    ┌───────────┐ │
   │  <script> │◀───────▶│  │  web   │    │  widget   │ │
   └───────────┘         │  │ (admin)│    │ (Vite/    │ │
                         │  │ Vite/  │    │  Preact)  │ │
   admin browser ───────▶│  │ React) │    └─────┬─────┘ │
                         │  └───┬────┘          │       │
                         │      │  @forge/contracts      │
                         │   ┌──▼───────────────▼─────┐ │
                         │   │     backend/apps/api    │ │  SSE / WS
                         │   │  auth · CRUD · chat     │◀──────────
                         │   └──┬──────────────┬──────┘ │
                         │      │              │        │
                         │  ┌───▼────┐    ┌────▼──────┐ │
                         │  │ @app/* │    │  BullMQ   │ │
                         │  │  libs  │    │  queues   │ │
                         │  └───┬────┘    └────┬──────┘ │
                         │      │              │        │
                         │      │         ┌────▼──────┐ │
                         │      │         │  worker   │ │
                         │      │         │  app      │ │
                         │      │         └────┬──────┘ │
                         └──────┼──────────────┼────────┘
                                │              │
                    ┌───────────▼──┐    ┌──────▼──────┐
                    │  Postgres    │    │   Redis     │
                    │  control DB  │    │  (BullMQ)   │
                    │  tenant DB×N │    └─────────────┘
                    │  (pgvector)  │
                    └──────────────┘
```

## Locked Decisions

| Area | Decision | Why |
|------|----------|-----|
| Repo shape | Single repo, thin pnpm workspace; backend is a NestJS native monorepo; `web`/`widget` separate Vite; workspace links only `packages/contracts` | Backend shares DI/tooling — let Nest own its monorepo. Workspace exists only to share types across the FE↔BE seam. No Nx/Turbo lock-in for contributors. |
| Contracts | `packages/contracts` (zod schemas + inferred types) | Single source of truth across Nest + Vite; validate on backend, type + reuse on frontend. |
| Backend | NestJS — `api` app + separate `worker` app | Workers needed for long ingestion/provisioning; Nest monorepo runs both from shared libs. |
| Agent runtime | Mastra (TS-native) | Dynamic per-tenant agents/tools via `RuntimeContext`; owns the tool-calling loop so we don't hand-roll it. LangGraph deferred until workflows get branchy. |
| Jobs | BullMQ + Redis | NestJS-native, mature, dashboard available. |
| ORM | Drizzle | Lightweight, first-class dynamic-connection story, easy programmatic migrations against arbitrary tenant DBs, pgvector friendly. |
| Tenancy | Database-per-tenant | Strongest isolation; matches the "separate DB per tenant in a cluster" requirement. |
| Tenant connections | Dynamic — control DB holds conn strings, resolved per request, pooled | Self-serve tenant creation needs runtime resolution, not boot-time config. |
| Provisioning | Auto via worker: create DB → enable pgvector → run migrations | Fully self-serve tenant onboarding. |
| Vectors | pgvector inside each tenant DB | Full data isolation per tenant. |
| LLM | Pluggable (Anthropic default, OpenAI, Ollama) | Open-source users have different keys/local setups. |
| Auth | Built-in JWT + refresh + sessions, role-based | Self-contained, no external IdP dependency for self-hosters. |
| UI | Untitled-UI tokens in Tailwind; Radix headless primitives only for interactive/overlay components | Untitled-UI is visual; Radix gives a11y/behavior; we own all styling. |
| Widget | Separate tiny Vite/Preact app → single `<script>` | Small isolated bundle, no CSS leak onto host sites. |
| Local dev | Docker Compose, one command, HMR for all apps | "Clone and up" developer experience. |

## Repository Shape

```
forge/
  backend/                       NestJS native monorepo (own package.json, nest-cli.json)
    nest-cli.json
    apps/
      api/                       REST/HTTP, auth, agent CRUD, chat (SSE/WS), tool exec
      worker/                    BullMQ consumers (ingest, crawl, embed, provision)
    libs/                        Nest libs, imported as @app/db, @app/llm, ...
      db/                        Drizzle schema (control + tenant), migrations, conn manager
      llm/                       Pluggable provider abstraction (chat + embeddings)
      agent/                     Mastra factory: per-tenant agents via RuntimeContext
      rag/                       Chunking, splitters, retrieval, source connectors
      tools/                     Dynamic API-tool engine (DB tool def → Mastra tool)
      common/                    Config (zod env), guards, interceptors, utils
  web/                           Vite + React admin console
  widget/                        Vite + Preact embeddable chat → single <script>
  packages/                      ONLY true FE↔BE shared code (starts minimal)
    contracts/                   Shared zod schemas + inferred types + shared enums/constants
  docker/                        Dockerfiles
  docker-compose.yml
  docker-compose.dev.yml         HMR overrides
  pnpm-workspace.yaml            links: backend, web, widget, packages/*
  package.json                   root scripts only
```

**Boundaries:**

- **`backend/`** is a self-contained NestJS monorepo. `db / llm / agent / rag /
  tools / common` are Nest libs (`nest g lib ...`), shared by `api` + `worker`
  via `@app/*` path aliases in `backend/tsconfig.json`. Nest's own webpack build
  handles everything inside `backend/` — no external monorepo tool reaches in.
- **pnpm workspace** exists only to link `packages/contracts` and the top-level
  projects so `web`, `widget`, and `backend` import the same `@forge/contracts`.
  It does not manage backend internals — Nest does.
- **`packages/` rule**: holds ONLY code imported by BOTH backend and a frontend.
  Backend-only code → `backend/libs/@app/*`. Frontend-only code → `web/`. Shared
  enums/constants fold into `contracts`. An API client lives in `web/` (or is
  generated from contracts) until a second consumer actually needs it.

## Backend (NestJS monorepo)

Two deployable apps share the libs:

- **`apps/api`** — HTTP/REST, authentication, admin + tenant CRUD, chat endpoint
  (SSE/WebSocket), tool execution surface, enqueues jobs.
- **`apps/worker`** — BullMQ consumers: `provision-tenant`, `ingest`, `crawl`,
  `embed`. Carries `tenantId` in job payloads to resolve the correct tenant DB.

Libs (`@app/*`): `db`, `llm`, `agent`, `rag`, `tools`, `common`. See each
section below and the corresponding doc under [`docs/`](./docs).

## Database Architecture

Two schema sets, both defined in `@app/db`.

### Control DB (single, global)

- `tenants` — id, slug, status, **db connection params** (encrypted), timestamps.
- `users` — admin/operator identities, hashed password, role, `tenant_id`.
- `sessions` / `refresh_tokens`.
- `provisioning_jobs` — status tracking for tenant provisioning.
- platform settings, LLM provider config (encrypted secrets).

### Tenant DB (one per tenant, replicated schema)

- `knowledge_bases`, `documents`, `chunks` (with a `vector` column via pgvector).
- `agents`, `agent_tools`, `tools` (API tool definitions).
- `conversations`, `messages`.
- `data_sources` (upload / url / s3 / sharepoint configs).

## Multi-Tenancy

See [docs/tenancy.md](./docs/tenancy.md) for the full walkthrough.

### Connection Manager (`@app/db`)

- A singleton control-DB pool is created at boot.
- `TenantConnectionManager`: given a `tenantId`, look up the connection string in
  the control DB, then lazily create and cache a Drizzle client + `pg.Pool`
  (LRU with idle eviction).
- A request-scoped `TenantContext` (resolved from a JWT claim / subdomain /
  header) is set by middleware; repositories use the resolved Drizzle client.
- Worker jobs resolve the same way using `tenantId` from the payload (no request
  scope).

### Auto-Provisioning flow (worker)

```
admin creates tenant
   └─ control DB row (status = provisioning)
   └─ enqueue `provision-tenant` (BullMQ)
            │
            ▼  worker
   CREATE DATABASE  (or connect to a pre-created DB)
   CREATE EXTENSION vector
   run Drizzle tenant migrations programmatically (migrate())
   write encrypted conn string to control DB
   set status = active
```

Migrations run via Drizzle's `migrate()` **from code**, not the CLI, so the
worker can target arbitrary tenant DBs. Tenant migration files are versioned in
`backend/libs/db/tenant/migrations`.

## RAG Pipeline

See [docs/rag.md](./docs/rag.md).

- **Ingestion sources** implement a pluggable connector interface
  (`fetch() -> RawDocument[]`): file upload, site URL crawl, S3, SharePoint.
- **Worker queues**: `ingest` → extract text (pdf/docx/html) → `chunk`
  (recursive splitter, configurable size/overlap) → `embed` (via `@app/llm`
  embeddings) → store in tenant DB `chunks.vector`.
- **Retrieval**: embed the query → pgvector cosine KNN (`<=>`) within the tenant
  DB, filtered by the selected knowledge base(s) → top-k context for the agent.

## LLM Abstraction

See [docs/llm-providers.md](./docs/llm-providers.md).

- Interfaces: `ChatProvider` (stream + tool-calling) and `EmbeddingProvider`.
- Adapters: **Anthropic (default, latest Claude)**, OpenAI, Ollama /
  OpenAI-compatible.
- Config-driven selection per platform / per agent. Secrets come from the control
  DB (encrypted) or env. `@app/llm` exposes provider model handles that Mastra
  agents consume: Mastra owns the tool-calling loop, `@app/llm` owns
  provider/secret configuration.

## Agent Core (Mastra)

See [docs/agents.md](./docs/agents.md).

- One Mastra instance per process; **agents are built dynamically per request /
  job**, never statically registered (a static registry fights multi-tenancy).
- `createTenantAgent({ tenantId, agentRow, runtimeContext })`:
  - model resolved from `@app/llm` (per-agent provider/model config),
  - system prompt from the agent row,
  - **tools** = the tenant's API-tool defs (from the tenant DB) compiled by
    `@app/tools` into Mastra tools, scoped to the selected agent,
  - **RAG retrieval** exposed as a Mastra step/tool querying tenant pgvector via
    `@app/rag`,
  - `RuntimeContext` carries `tenantId` + the resolved tenant Drizzle client so
    tools and retrieval hit the correct DB.
- Streaming via Mastra's `stream()` → piped to SSE/WebSocket in
  `backend/apps/api`.

## Dynamic API Tools

See [docs/tools.md](./docs/tools.md).

- A tool is an HTTP request template: method, URL, headers, auth (apiKey /
  bearer / basic — **no OAuth2 flow**), input schema (zod/JSON), response
  mapping.
- `toMastraTool(toolDef)` compiles a DB tool definition into a Mastra tool:
  input schema + an `execute()` that validates args (zod), runs the HTTP call,
  and returns the mapped result.
- Pre-built connectors ship as templates (a catalog) that users instantiate and
  fill credentials for.
- Secrets are stored encrypted in the tenant DB and injected at execute time —
  never exposed to the LLM.

## Chat & Streaming

- Resolve tenant + agent → `@app/agent` builds the per-tenant Mastra agent →
  stream.
- Mastra runs the loop: retrieve RAG context → call the LLM with tools → execute
  any tool call → feed back → repeat → stream tokens.
- A **public chat endpoint** serves the widget: scoped token per agent,
  rate-limited, CORS-aware.

## Frontend

See the [UI section in docs](./docs/getting-started.md).

### `web/` (admin console)

- Vite + React Router + TanStack Query + Tailwind.
- Areas: onboarding/admin setup, knowledge bases + ingestion status, agent
  builder, tool builder + connector catalog, conversations/analytics, embed-code
  generator.
- Imports `@forge/contracts` for API types/validation.

### UI library — Untitled-UI in Tailwind

Lives under `web/src/ui` (the widget reuses a subset).

- Tailwind config encodes Untitled-UI **design tokens**: color scales (gray +
  brand + semantic), spacing, border-radius scale, shadows, Inter font, sizes.
- **Pure-Tailwind, no primitive** for static components: Button, Input, Textarea,
  Badge, Card, Table, Avatar, Label, form fields.
- **Radix headless primitives only where behavior is hard** (focus trap, keyboard
  nav, ARIA, portals): Dialog/Modal, Dropdown Menu, Select, Combobox, Tooltip,
  Popover, Tabs, Toast. Radix ships zero styling → styled fully to Untitled-UI
  tokens.
- `tailwind-merge` + `cva` for variants. Storybook optional.

## Embeddable Widget

See [docs/widget-embed.md](./docs/widget-embed.md).

- Tiny Preact app, scoped styles (shadow DOM or prefixed), builds to a single JS
  file.
- Loads agent config by public token, renders a chat bubble, talks to the public
  chat API.
- Imports `@forge/contracts` for the public chat types.

## Shared Contracts

`packages/contracts`:

- Zod schemas for every API request/response + shared enums/domain constants.
- Types inferred (`z.infer`) → one source of truth. Backend validates with them
  (Nest zod pipe); the frontend gets types and can reuse schemas for form
  validation.

## Local Development

See [docs/self-hosting.md](./docs/self-hosting.md).

`docker-compose.yml` services:

- `postgres` — `pgvector/pgvector:pg16`, hosts the control DB + all tenant DBs in
  one cluster for dev.
- `redis` — BullMQ.
- `api`, `worker`, `web`, `widget`, optional `bull-board` dashboard.

`docker-compose.dev.yml` override: bind-mount source, per-service dev command
with HMR — Vite HMR for `web`/`widget`, `nest start --watch <app>` for
`api`/`worker`. One command:

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml up
```

## Deployment (future)

- Production Dockerfiles per app under `docker/`.
- **Terraform**, one stack per deployable: `api`, `worker`, `web`/`widget`
  (static), Postgres, Redis. Target: one-click AWS deploy.

## Open Items

Decisions deferred to implementation:

1. **Tenant resolution channel** — subdomain vs header vs JWT claim. Leaning:
   JWT claim for admin, public token for the widget.
2. **Secret encryption** — app-level (KMS / age / libsodium) vs Postgres
   `pgcrypto`.
3. **Tenant DB creation** — whether `CREATE DATABASE` is permitted in the target
   cluster vs pre-created empty DBs handed to the provisioner.
