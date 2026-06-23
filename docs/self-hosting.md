# Self-Hosting

How to run Forge yourself — local dev and production notes.

> **Status:** _(planned)_ for production bits; dev compose is the primary target.

## Compose files

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Base services: postgres (pgvector), redis, api, worker, web, widget |
| `docker-compose.dev.yml` | Dev override: bind-mounts source, runs HMR/watch dev commands |

### Dev (HMR)

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml up
```

- `web` / `widget` → Vite HMR.
- `api` / `worker` → `nest start --watch <app>`.

### Production-style (built images)

```bash
docker compose up --build
```

Uses the production Dockerfiles under `docker/` (multi-stage builds, no
bind-mount, compiled output).

## Environment variables

Copy `.env.example` to `.env`. Key groups:

```dotenv
# --- Core ---
NODE_ENV=development
API_PORT=3000

# --- Control database ---
CONTROL_DATABASE_URL=postgres://forge:forge@postgres:5432/forge_control

# --- Tenant database cluster ---
# Host/cluster where tenant DBs are created/connected. Conn strings per tenant
# are stored (encrypted) in the control DB; this is the base/admin connection
# used by the provisioner.
TENANT_DB_ADMIN_URL=postgres://forge:forge@postgres:5432/postgres
TENANT_DB_CREATE_MODE=create        # create | pre-created   (see Open Items)

# --- Redis / BullMQ ---
REDIS_URL=redis://redis:6379

# --- Auth ---
JWT_SECRET=change-me
JWT_ACCESS_TTL=15m
JWT_REFRESH_TTL=30d

# --- Secret encryption ---
# Key used to encrypt tenant conn strings + tool/provider secrets at rest.
SECRETS_ENCRYPTION_KEY=change-me-32-bytes

# --- LLM ---
LLM_DEFAULT_PROVIDER=anthropic
ANTHROPIC_API_KEY=
OPENAI_API_KEY=
OLLAMA_BASE_URL=http://host.docker.internal:11434

# --- Frontend ---
WEB_PORT=5173
WIDGET_PORT=5174
PUBLIC_API_URL=http://localhost:3000
```

Validation: env is parsed and validated with zod in `@app/common` at boot — the
app fails fast on a missing/invalid variable.

## Database

- One Postgres instance (cluster) hosts the **control DB** and every **tenant
  DB**. Image: `pgvector/pgvector:pg16` so `CREATE EXTENSION vector` works.
- Control DB migrations run on `api`/`worker` boot.
- Tenant DB migrations run programmatically by the worker during provisioning —
  see [Tenancy](./tenancy.md).

## Backups

- Back up the **control DB** (tenants, users, encrypted secrets) and **every
  tenant DB**. A tenant DB is the only place that tenant's documents, vectors,
  agents, and conversations live.

## Production checklist _(planned)_

- [ ] Set strong `JWT_SECRET` and `SECRETS_ENCRYPTION_KEY`.
- [ ] Put the API behind TLS + a reverse proxy; configure CORS for the widget's
      allowed origins.
- [ ] Use managed Postgres (with pgvector) and managed Redis.
- [ ] Configure object storage for uploads.
- [ ] Enable rate limiting on public chat endpoints.
- [ ] Set up logging/metrics/tracing.

## Future: Terraform

A one-click AWS deploy is planned via Terraform, one stack per deployable: `api`,
`worker`, `web`/`widget` (static), Postgres, Redis.
