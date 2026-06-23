# LLM Providers

Forge is provider-agnostic. Chat and embeddings go through a pluggable
abstraction, with Claude as the default.

> Lives in `@app/llm`. Mastra agents consume the model handles `@app/llm`
> exposes; Mastra owns the tool-calling loop, `@app/llm` owns provider + secret
> configuration. See [Agents](./agents.md).

## Interfaces

```ts
interface ChatProvider {
  // streaming + tool-calling capable
}

interface EmbeddingProvider {
  // produces vectors for RAG (see docs/rag.md)
}
```

## Built-in adapters

| Provider | Chat | Embeddings | Notes |
|----------|------|------------|-------|
| **Anthropic** | Yes | — | **Default.** Latest Claude models. |
| **OpenAI** | Yes | Yes | GPT chat + embedding models |
| **Ollama / OpenAI-compatible** | Yes | Yes | Local / self-hosted, no paid key |

> Embeddings require a provider that offers an embedding model (e.g. OpenAI or a
> local model via Ollama). You can use Claude for chat and another provider for
> embeddings.

## Configuration

Defaults come from environment (see [Self-Hosting](./self-hosting.md)):

```dotenv
LLM_DEFAULT_PROVIDER=anthropic
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
OLLAMA_BASE_URL=http://host.docker.internal:11434
```

Selection is config-driven at two levels:

- **Platform default** — the fallback provider/model.
- **Per-agent** — an agent can override provider + model in its config.

Secrets are read from the **control DB (encrypted)** or env.

## Default model

Forge defaults to the latest Claude model for chat. When building on the Claude
API directly, prefer the most capable current models — see the project's
LLM/Claude API reference for current model IDs and pricing.

## Choosing a setup

| Goal | Suggested config |
|------|------------------|
| Best quality, hosted | Anthropic (Claude) for chat + OpenAI for embeddings |
| Fully local / no paid keys | Ollama for both chat and embeddings |
| Existing OpenAI stack | OpenAI for chat + embeddings |

## Adding a provider _(planned)_

Implement `ChatProvider` and/or `EmbeddingProvider` in `@app/llm`, register it,
and expose its config in env + the admin settings UI.
