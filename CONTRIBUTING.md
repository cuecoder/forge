# Contributing to Forge

Thanks for helping build Forge. This guide covers local setup, repo conventions,
and the PR flow.

## Prerequisites

- Docker + Docker Compose
- Node 20+
- pnpm 9+

## Setup

```bash
git clone <your-fork-url> forge && cd forge
pnpm install                 # links the workspace + packages/contracts
cp .env.example .env         # set at least one LLM key
docker compose -f docker-compose.yml -f docker-compose.dev.yml up
```

See [docs/getting-started.md](./docs/getting-started.md).

## Repo conventions

Read [ARCHITECTURE.md](./ARCHITECTURE.md) first. Key rules:

- **Backend is a NestJS native monorepo** under `backend/`. Generate apps/libs
  with the Nest CLI from inside `backend/`:

  ```bash
  cd backend
  pnpm nest g lib <name>     # → libs/<name>, import as @app/<name>
  pnpm nest g app <name>     # → apps/<name>
  ```

  Backend-only code goes in `backend/libs/@app/*`. Do **not** add backend libs to
  `packages/`.

- **`packages/` is the FE↔BE seam only.** Add something here **only** if it is
  imported by both the backend and a frontend. Shared enums/constants fold into
  `packages/contracts`. Frontend-only code lives in `web/`.

- **Contracts first.** New/changed API request/response shapes go in
  `packages/contracts` as zod schemas; types are inferred (`z.infer`). The backend
  validates with them; the frontend imports the types.

- **Multi-tenant by default.** Tenant data lives in tenant DBs. Anything touching
  tenant data must resolve the tenant (request `TenantContext` or job `tenantId`)
  and use the tenant Drizzle client — never the control DB. See
  [docs/tenancy.md](./docs/tenancy.md).

- **Secrets** (tenant conn strings, tool/provider creds) are encrypted at rest and
  only decrypted at use time. Never log them; never send them to the LLM.

## Code style

- TypeScript everywhere. Match the style of surrounding code.
- Lint + format before committing (`pnpm lint`, `pnpm format`).
- Validate inputs with zod at boundaries.

## Tests

```bash
cd backend && pnpm test         # backend unit/integration
cd web && pnpm test             # frontend (when present)
```

Add tests for new behavior, especially around tenant isolation, provisioning,
RAG retrieval, and tool execution.

## Workflow: issue → branch → PR → merge

Every unit of work follows the same loop: create an issue, branch from it, commit,
open a PR that `Closes #N`, and merge (which closes the issue). The full flow with
`gh` commands is in [docs/WORKFLOW.md](./docs/WORKFLOW.md).

## Commits & PRs

- Conventional Commits (`feat:`, `fix:`, `docs:`, `refactor:`, `chore:` …).
- Branch naming: `<type>/<issue-number>-<slug>`.
- PR body must contain `Closes #<issue>`.
- Keep PRs focused; reference the build phase (see ARCHITECTURE "Build Order")
  when relevant.
- Describe what changed and why; note any new env vars or migrations.
- For schema changes, include the migration and mention impact on existing
  tenants.

## Build phases

The project is built in phases (see [ARCHITECTURE.md](./ARCHITECTURE.md) →
"Open Items" / plan):

1. Foundation → 2. DB core → 3. Tenancy → 4. UI system → 5. RAG →
6. LLM + Agents → 7. Tools → 8. Widget → 9. Hardening.

Pick work that fits the current phase when possible.

## Questions

Open an issue or a discussion. Architecture-level proposals: reference
ARCHITECTURE.md and the relevant doc under `docs/`.
