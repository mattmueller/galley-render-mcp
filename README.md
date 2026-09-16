# Galley Render — MCP server

**JSON in, PDF out.** Galley Render turns a template and a JSON payload into a PDF, PNG or JPG
behind a signed URL. Every render is deterministic and cached, so the same input always returns
the same file and an identical repeat call costs nothing.

```
https://mcp.galleyrender.com/mcp
```

Streamable HTTP. Ten tools. **No API key needed to start** — the first `render` mints a
50-render trial and hands back its token, and `create_account` upgrades that trial in place
without losing anything made during it.

This repository is the public manifest for that server: what it is, where it lives, and how to
connect a client to it. The service itself is closed-source; see [Licence](#licence).

- Website — <https://galleyrender.com>
- Docs — <https://galleyrender.com/docs>
- MCP setup — <https://galleyrender.com/docs/mcp>
- Privacy — <https://galleyrender.com/privacy>
- Terms — <https://galleyrender.com/terms>
- Official MCP registry — `com.galleyrender/galley-render`

## Add it to a client

### Cursor — one click

[![Add to Cursor](https://img.shields.io/badge/Add%20to-Cursor-000?logo=cursor&logoColor=fff)](cursor://anysphere.cursor-deeplink/mcp/install?name=galley-render&config=eyJ1cmwiOiJodHRwczovL21jcC5nYWxsZXlyZW5kZXIuY29tL21jcCJ9)

If the badge does not open Cursor, paste the link:

```
cursor://anysphere.cursor-deeplink/mcp/install?name=galley-render&config=eyJ1cmwiOiJodHRwczovL21jcC5nYWxsZXlyZW5kZXIuY29tL21jcCJ9
```

The manual equivalent, in `.cursor/mcp.json` (or `~/.cursor/mcp.json` for every project):

```json
{
  "mcpServers": {
    "galley-render": {
      "url": "https://mcp.galleyrender.com/mcp",
      "headers": { "X-Galley-Api-Key": "glr_sk_…" }
    }
  }
}
```

Drop the `headers` block to stay on the keyless trial.

### Claude Code

```bash
claude mcp add --transport http galley https://mcp.galleyrender.com/mcp
```

Once you have a key:

```bash
claude mcp add --transport http galley https://mcp.galleyrender.com/mcp \
  --header "X-Galley-Api-Key: $GALLEY_API_KEY"
```

### OpenAI Agents SDK

```python
import os
from agents import Agent
from agents.mcp import MCPServerStreamableHttp

galley = MCPServerStreamableHttp(
    name="galley",
    params={
        "url": "https://mcp.galleyrender.com/mcp",
        # Omit `headers` entirely to use the keyless trial.
        "headers": {"X-Galley-Api-Key": os.environ["GALLEY_API_KEY"]},
    },
)

agent = Agent(
    name="Document agent",
    instructions="Use Galley to produce PDFs and images.",
    mcp_servers=[galley],
)
```

### Anything else

Any MCP client that speaks streamable HTTP. `POST` JSON-RPC to `/mcp`. The server is stateless,
so there is no session to keep:

```bash
curl -sS https://mcp.galleyrender.com/mcp \
  -H 'content-type: application/json' \
  -H 'accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{
        "name":"render",
        "arguments":{"template":"og-card","format":"png",
                     "data":{"title":"Documents for agents"}}}}'
```

## Tools

| Tool | What it does | Billed |
|---|---|---|
| `list_templates` | Templates on the account, with their latest version | no |
| `get_template` | One version: JSON Schema, options, example, HTML source | no |
| `create_template` | Create a template at version 1 | no |
| `update_template` | Publish a new immutable version | no |
| `validate_data` | Dry-run a payload; the same field-level errors a render would give | no |
| `render` | Template + data → signed URL for a PDF, PNG or JPG | 1 unit per PNG/JPG, 1 per PDF page |
| `get_render` | Status, and a freshly signed URL | no |
| `list_renders` | Recent renders, newest first | no |
| `usage` | Period usage, cost, free-tier balance, spend cap, trial balance | no |
| `create_account` | Email → verification link → API key | no |

Cache hits are never billed.

## Templates

Templates are code: HTML and CSS with a small expression language, versioned like git, each one
carrying a JSON Schema that is the contract for its data. Validation errors are written for an
agent to act on — field path, expected type, what was received, and an accepted example.

The starter library covers invoices, quotes, estimates, change orders, receipts, statements,
purchase orders, certificates, report covers and table pages, shipping labels, packing slips, OG
cards, social quote cards, event tickets, badges, menus, price sheets, letters and slide decks.

## Files here

| File | What it is |
|---|---|
| [`plugin.json`](./plugin.json) | Agent Plugins 1.1.0 manifest — the file directories look for first |
| [`mcp.json`](./mcp.json) | Agent Plugins MCP configuration: one `streamable-http` server |
| [`server.json`](./server.json) | The entry published to the official MCP registry, schema `2025-12-11` |
| [`README.md`](./README.md) | This page |

There is no source code in this repository and none is planned. It exists so that directories
which require a public repository — cursor.directory, mcp.so, the n8n Creator Portal — have one
to read.

`plugin.json` and `mcp.json` follow the [Agent Plugins](https://agent-plugins.org) standard
(formerly Open Plugins), both validated against the 1.1.0 schemas, which is what
[cursor.directory](https://cursor.directory) auto-detects from a repository URL. Note that the
standard spells the transport `streamable-http`, while a hand-written `~/.cursor/mcp.json` uses
the shorter shape shown above — Cursor accepts either.

`server.json` is generated from the Galley Render monorepo and re-published to the registry
whenever the tool surface changes; treat it as a copy, not the source of truth.

## Not MCP?

Every tool above is a thin wrapper over `https://api.galleyrender.com`, so nothing is MCP-only.
The REST surface is OpenAPI 3.1 at <https://api.galleyrender.com/openapi.json>, and there are
Node and Python clients — see <https://galleyrender.com/docs/sdks>.

## Licence

The files in this repository — `README.md`, `plugin.json`, `mcp.json` and `server.json` — are
released under the [MIT licence](./LICENSE), so a directory or client may copy them freely.

That licence covers **these manifest files only**. It is not a licence to the Galley Render
service, its API, its templates or its name. Use of the service is governed by the
[terms](https://galleyrender.com/terms) and the [privacy policy](https://galleyrender.com/privacy).

Questions: support@galleyrender.com
