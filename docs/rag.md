# RAG Pipeline

How Forge turns your documents and sources into searchable knowledge, and how it
retrieves context for agent replies.

> Lives in `@app/rag` + the `worker` app. Vectors are stored in pgvector inside
> each tenant DB. See [Tenancy](./tenancy.md).

## Concepts

- **Knowledge base (KB)** — a named collection an agent can retrieve from.
- **Data source** — where content comes from (upload / url / s3 / sharepoint),
  attached to a KB.
- **Document** — one ingested item (a file, a crawled page).
- **Chunk** — a slice of a document with an embedding `vector`.

## Ingestion sources

Each source implements a pluggable connector interface:

```ts
interface IngestionSource {
  fetch(): Promise<RawDocument[]>;
}
```

Built-in sources:

| Source | Notes |
|--------|-------|
| **Upload** | PDF, DOCX, HTML, plain text |
| **Site crawl** | Crawl a URL (depth/scope configurable) |
| **S3** | Bucket/prefix → objects |
| **SharePoint** | Document library |

> Build order: **upload + URL crawl first**, S3 / SharePoint after.

## Pipeline (worker queues)

```
data source ──▶ [ingest] ──▶ extract text (pdf/docx/html)
                                   │
                                   ▼
                              [chunk]  recursive splitter
                                       (configurable size + overlap)
                                   │
                                   ▼
                              [embed]  via @app/llm EmbeddingProvider
                                   │
                                   ▼
                    store chunks (+ vector) in tenant DB  (pgvector)
```

- Queues are BullMQ; each job carries `tenantId` to resolve the correct tenant DB.
- Chunking uses a recursive splitter with configurable **size** and **overlap**.
- Embeddings come from the configured [LLM provider](./llm-providers.md)'s
  `EmbeddingProvider`.

## Storage

Chunks live in the tenant DB:

```
chunks(
  id, document_id, knowledge_base_id,
  content text,
  vector  vector(<dim>),   -- pgvector, dim matches the embedding model
  metadata jsonb,
  ...
)
```

An index (e.g. HNSW / IVFFlat) is created on `vector` for fast KNN.

## Retrieval

At query time:

1. Embed the query with the same `EmbeddingProvider`.
2. Run a cosine KNN search in the tenant DB:

   ```sql
   SELECT content, metadata
   FROM chunks
   WHERE knowledge_base_id = ANY($selectedKbs)
   ORDER BY vector <=> $queryEmbedding
   LIMIT $k;
   ```

3. Return the top-k chunks as context for the agent.

Retrieval is exposed to the agent as a Mastra step/tool (see [Agents](./agents.md))
so the model can pull context as part of the tool-calling loop.

## Re-ingestion & updates _(planned)_

- Re-crawl / re-sync a source on a schedule or on demand.
- De-duplicate by content hash; update changed documents, remove deleted ones.
