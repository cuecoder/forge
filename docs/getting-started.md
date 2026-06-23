# Getting Started

This guide gets Forge running locally and walks through building your first
agent.

> **Status:** _(planned)_ — the build is phased. Use this as the intended flow;
> see [ARCHITECTURE.md](../ARCHITECTURE.md) for what each piece does.

## Prerequisites

- **Docker** + **Docker Compose** — runs the whole stack.
- **Node 20+** and **pnpm 9+** — only needed if you work outside containers
  (running tests, generating Nest libs, editing with full IntelliSense).

## 1. Clone and configure

```bash
git clone <your-fork-url> forge && cd forge
cp .env.example .env
```

Edit `.env` and set at least one LLM provider key. For the default (Claude):

```dotenv
LLM_DEFAULT_PROVIDER=anthropic
ANTHROPIC_API_KEY=sk-ant-...
```

See [LLM Providers](./llm-providers.md) for OpenAI / Ollama instead.

## 2. Bring the stack up (dev, with HMR)

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml up
```

This starts:

| Service | Purpose | Default URL/Port |
|---------|---------|------------------|
| `postgres` | Control DB + tenant DBs (pgvector) | `localhost:5432` |
| `redis` | BullMQ queues | `localhost:6379` |
| `api` | NestJS API (REST + chat SSE/WS) | `localhost:3000` |
| `worker` | Background jobs (ingest, embed, provision) | — |
| `web` | Admin console (Vite HMR) | `localhost:5173` |
| `widget` | Embeddable chat dev build | `localhost:5174` |

Source is bind-mounted; edits hot-reload across all apps.

## 3. Set up admin

Open the admin console (`http://localhost:5173`) and complete the setup wizard:
create the first admin user. This also creates your first tenant and triggers
[auto-provisioning](./tenancy.md) of its dedicated database.

## 4. Ingest knowledge

Create a **knowledge base**, then add a **data source**:

- **Upload** documents (PDF, DOCX, HTML, text), or
- **Crawl** a site URL, or
- Connect **S3** / **SharePoint**.

Forge enqueues ingestion jobs (extract → chunk → embed → store in pgvector).
Watch progress in the knowledge base view. See [RAG](./rag.md) for details.

## 5. Build an agent

Create an **agent** and configure:

- **Model** — provider + model (defaults to the platform default).
- **System prompt** — the agent's instructions.
- **Knowledge bases** — which KBs it can retrieve from.
- **Tools** — which API tools it can call (see step 6).

See [Agents](./agents.md).

## 6. Add a tool _(optional)_

Turn any REST API into a tool: define method, URL, headers, auth (apiKey /
bearer / basic — no OAuth2 flow), input schema, and response mapping. Or
instantiate one from the **connector catalog**. See [Tools](./tools.md).

## 7. Chat and embed

- Use the built-in **agent UI** to chat with grounded, tool-using replies.
- Generate an **embed snippet** and drop the `<script>` on any site. See
  [Widget Embed](./widget-embed.md).

## Working outside Docker

```bash
pnpm install                 # from repo root; links workspace + contracts
cd backend && pnpm build     # builds api + worker via Nest
cd ../web && pnpm dev        # Vite dev server
```

The backend is a NestJS native monorepo — generate libs/apps with the Nest CLI
from inside `backend/`. See [CONTRIBUTING.md](../CONTRIBUTING.md).
