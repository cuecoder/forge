# Tools

Forge turns any REST API into a tool an agent can call — no code required. Define
the request, the inputs, and how to read the response; the agent does the rest.

> Lives in `@app/tools`. Tools are stored per-tenant and compiled into Mastra
> tools at agent-build time. See [Agents](./agents.md).

## What a tool is

A tool is an **HTTP request template** stored in the tenant DB (`tools` table):

| Field | Purpose |
|-------|---------|
| method | GET / POST / PUT / PATCH / DELETE |
| url | endpoint, may contain `{placeholders}` filled from inputs |
| headers | static + templated headers |
| auth | apiKey / bearer / basic — **no OAuth2 flow** |
| input schema | zod/JSON schema the LLM must satisfy to call the tool |
| response mapping | how to extract the useful result from the HTTP response |

### Supported auth

- **API key** — header or query param.
- **Bearer token**.
- **Basic auth**.

> OAuth2 authorization-code flows are intentionally **not** supported.

## How a tool runs

`toMastraTool(toolDef)` compiles a stored definition into a Mastra tool:

```
LLM decides to call tool
   │  args (validated against input schema, zod)
   ▼
toMastraTool.execute(args)
   │  build request from template (url + headers + auth + body)
   │  inject secrets (decrypted at execute time — never sent to the LLM)
   ▼
HTTP call ──▶ map response ──▶ return result to the model
```

- **Validation** — arguments are validated with zod before the call.
- **Secrets** — credentials are stored **encrypted** in the tenant DB and
  injected at execute time. They are never exposed to the LLM or returned in tool
  output.
- **Scoping** — only the tools attached to the agent are compiled and offered to
  the model.

## Connector catalog

Pre-built connector **templates** ship with Forge. A user instantiates a
template and fills in credentials/parameters rather than defining the request
from scratch. Instantiated connectors become normal tools in the tenant DB.

## Example (conceptual)

A "lookup order" tool:

```jsonc
{
  "name": "lookup_order",
  "method": "GET",
  "url": "https://api.example.com/orders/{orderId}",
  "auth": { "type": "bearer", "secretRef": "example_api_token" },
  "input": {                       // zod/JSON schema
    "orderId": { "type": "string", "description": "Order ID to look up" }
  },
  "responseMapping": "$.data"      // return only the data field
}
```

The agent, when asked about an order, calls `lookup_order` with an `orderId`,
Forge runs the request with the injected bearer token, maps the response, and
feeds it back into the conversation.

## Security notes

- No OAuth2 flow → no token dance to manage.
- Secrets encrypted at rest, decrypted only at execute time.
- Consider egress controls / allowlists for tool URLs in production _(planned)_.
