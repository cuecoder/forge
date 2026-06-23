# Widget Embed

Ship your agent as a chat bubble on any website with a single `<script>` tag.

> The widget is a separate tiny Vite + Preact app (`widget/`) that builds to one
> JS file. It talks to the public chat endpoint. See [Agents](./agents.md).

## How it works

```
host website
   │  <script ... data-agent-token="..."> </script>
   ▼
widget loads ──▶ fetch agent config by public token
   │
   ▼
render chat bubble (isolated styles)
   │  user message
   ▼
POST public chat endpoint ──▶ streamed reply (SSE)
```

- **Isolated styles** — the widget renders in a shadow DOM (or fully prefixed
  styles) so it never leaks CSS onto, or inherits CSS from, the host page.
- **Single bundle** — one `<script>`; no build step on the host.
- **Public token** — each agent has a scoped public token; the endpoint is
  rate-limited and CORS-restricted to allowed origins.

## Embed snippet

Generate the snippet from the admin console (agent → "Embed"). It looks like:

```html
<script
  src="https://your-forge-host/widget.js"
  data-agent-token="pub_xxxxxxxxxxxxx"
  data-position="bottom-right"
  defer
></script>
```

| Attribute | Purpose |
|-----------|---------|
| `data-agent-token` | scoped public token identifying the agent |
| `data-position` | bubble placement (e.g. `bottom-right`) |
| other data-* | theme/labels _(planned)_ |

## Security

- The public token authorizes **only** chatting with that one agent.
- Configure **allowed origins** (CORS) per agent so the widget only works on your
  sites.
- The endpoint is **rate-limited**.
- Tool secrets and tenant data are never exposed to the widget — tools run
  server-side (see [Tools](./tools.md)).

## Local development

In dev, the widget runs at `http://localhost:5174` with HMR. Test the embed on a
throwaway HTML page pointing the `src` at the dev bundle and using a dev agent's
public token.

## The built-in agent UI

If you don't need to embed on a third-party site, the admin console (`web/`)
includes a full Amazon-Q-style agent chat UI for the same agents.
