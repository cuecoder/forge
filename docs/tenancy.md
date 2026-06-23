# Tenancy

Forge is **multi-tenant with full database isolation**: each tenant gets a
dedicated Postgres database, provisioned automatically. This document covers the
schema split, dynamic connections, and the provisioning flow.

> Lives in `@app/db`. See also [ARCHITECTURE.md](../ARCHITECTURE.md).

## Why database-per-tenant

- **Strongest isolation** — one tenant's documents, vectors, agents, and
  conversations never share tables (or a database) with another's.
- **pgvector per tenant** — each tenant's embeddings live in their own DB; no
  cross-tenant filtering to get wrong.
- **Clean lifecycle** — drop a tenant = drop a database.

Trade-off: connection management and migrations are more involved. Forge handles
both (below).

## Two schema sets

### Control DB (single, global)

| Table | Holds |
|-------|-------|
| `tenants` | id, slug, status, **encrypted DB connection params**, timestamps |
| `users` | admin/operator identities, hashed password, role, `tenant_id` |
| `sessions` / `refresh_tokens` | auth |
| `provisioning_jobs` | provisioning status tracking |
| settings / provider config | platform settings, encrypted LLM secrets |

### Tenant DB (one per tenant, identical schema)

| Table | Holds |
|-------|-------|
| `knowledge_bases`, `documents`, `chunks` | RAG data; `chunks.vector` is pgvector |
| `agents`, `agent_tools`, `tools` | agent + tool definitions |
| `conversations`, `messages` | chat history |
| `data_sources` | upload / url / s3 / sharepoint configs |

## Connection Manager

`TenantConnectionManager` (in `@app/db`):

1. A **singleton control-DB pool** is created at boot.
2. On a tenant-scoped operation, look up the tenant's connection string in the
   control DB.
3. **Lazily create and cache** a Drizzle client + `pg.Pool` per tenant, kept in
   an **LRU cache with idle eviction** (bounded open connections).

### Resolving the tenant

- **API requests** — a request-scoped `TenantContext` is set by middleware from
  the tenant identifier (JWT claim / subdomain / header — see Open Items).
  Repositories then use the resolved Drizzle client.
- **Worker jobs** — `tenantId` travels in the job payload; the worker resolves
  the same way (no request scope).

```
request ──▶ auth ──▶ resolve tenantId ──▶ TenantContext
                                              │
                                              ▼
                              TenantConnectionManager.get(tenantId)
                                              │
                                  cached Drizzle client (tenant DB)
```

## Auto-Provisioning

When an admin creates a tenant:

```
1. Insert control DB row  (status = provisioning)
2. Enqueue `provision-tenant`  (BullMQ)
        │
        ▼  worker consumer
3. CREATE DATABASE                 (or connect to a pre-created empty DB)
4. CREATE EXTENSION IF NOT EXISTS vector
5. Run tenant migrations via Drizzle migrate()  (from code, against the new DB)
6. Write encrypted connection string to the control DB
7. status = active
```

- Migrations run with Drizzle's `migrate()` **programmatically** (not the CLI) so
  the worker can target arbitrary tenant databases.
- Tenant migration files are versioned in
  `backend/libs/db/tenant/migrations`.
- Provisioning is idempotent/retry-safe — `provisioning_jobs` tracks state so a
  retried job resumes rather than duplicates.

## Adding new tenant tables

1. Edit the tenant Drizzle schema in `@app/db`.
2. Generate a migration into `backend/libs/db/tenant/migrations`.
3. New tenants get it at provision time; existing tenants get a
   **migrate-on-connect** / backfill pass _(planned)_ that applies pending tenant
   migrations the next time each tenant DB is used.

## Open Items

- **Tenant resolution channel** — subdomain vs header vs JWT claim. Leaning: JWT
  claim for admin, public token for the widget.
- **Secret encryption** — app-level (KMS / age / libsodium) vs Postgres
  `pgcrypto` for the stored connection strings + secrets.
- **DB creation** — whether `CREATE DATABASE` is allowed in the target cluster
  (`TENANT_DB_CREATE_MODE=create`) or empty DBs are pre-created and handed to the
  provisioner (`pre-created`).
