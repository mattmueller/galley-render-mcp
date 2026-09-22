<!-- Generated from README.template.md by build.mjs in the Galley Render monorepo (ops/listings/galley-render-mcp). Edit there, not here. -->
# Galley Render MCP server

**JSON in, PDF out.** Galley Render turns a template and a JSON payload into a PDF, PNG or JPG
behind a signed URL. Every render is deterministic and cached, so the same input returns the same
file and an identical repeat call costs nothing.

```
https://mcp.galleyrender.com/mcp
```

It's streamable HTTP with sixteen tools, and you don't need an API key to start. The
first call without one mints a trial of 10 PDF pages or 10 images and returns its token.
`create_account` asks the owner of an email address to confirm the request, by a link that shows
who asked and a short code, and then turns that trial into an account in place, so nothing you made
during it is lost.

This repository is the public manifest for that server: what it is, where it lives and how to
connect a client to it. The service itself is closed source (see [License](#license)).

- Website: <https://galleyrender.com>
- Docs: <https://galleyrender.com/docs>
- MCP setup: <https://galleyrender.com/docs/mcp>
- Privacy: <https://galleyrender.com/privacy>
- Terms: <https://galleyrender.com/terms>
- Official MCP registry: `com.galleyrender/galley-render`

## Add it to a client

### Cursor

[![Add to Cursor](https://img.shields.io/badge/Add%20to-Cursor-000?logo=cursor&logoColor=fff)](cursor://anysphere.cursor-deeplink/mcp/install?name=galley-render&config=eyJ1cmwiOiJodHRwczovL21jcC5nYWxsZXlyZW5kZXIuY29tL21jcCJ9)

If the badge doesn't open Cursor, paste the link:

```
cursor://anysphere.cursor-deeplink/mcp/install?name=galley-render&config=eyJ1cmwiOiJodHRwczovL21jcC5nYWxsZXlyZW5kZXIuY29tL21jcCJ9
```

Or add it by hand, in `.cursor/mcp.json` (or `~/.cursor/mcp.json` for every project):

```json
{
  "mcpServers": {
    "galley-render": {
      "url": "https://mcp.galleyrender.com/mcp",
      "headers": { "Authorization": "Bearer glr_sk_…" }
    }
  }
}
```

Leave out `headers` to stay on the keyless trial.

### Claude Code

```bash
claude mcp add --transport http galley https://mcp.galleyrender.com/mcp
```

Once you have a key:

```bash
claude mcp add --transport http galley https://mcp.galleyrender.com/mcp \
  --header "Authorization: Bearer $GALLEY_API_KEY"
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
        # Leave out `headers` to use the keyless trial.
        "headers": {"Authorization": f"Bearer {os.environ['GALLEY_API_KEY']}"},
    },
)

agent = Agent(
    name="Document agent",
    instructions="Use Galley to produce PDFs and images.",
    mcp_servers=[galley],
)
```

### Clients that can't send a header

Claude.ai custom connectors and ChatGPT apps have nowhere to put one, and they call from their
vendor's servers, so on the shared address every one of their users looks the same. Add the server
with a personal address instead, `https://mcp.galleyrender.com/mcp/c/<random id>`
([galleyrender.com/docs/connect](https://galleyrender.com/docs/connect) makes one), then call
`link_account` once with your key, or with a one-time code that `create_account` emails you, and
calls from that client act on your account. [How it works](https://galleyrender.com/docs/mcp#use-your-key).

### Anything else

Any client that speaks streamable HTTP works. The server is stateless, so there's no session to
keep and no handshake to do first. A single `tools/call` POST is enough:

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
| `list_templates` | Templates on the account with their latest version. A new account already has the full starter library. | no |
| `get_template` | One version in full: JSON Schema, default options, example payload, HTML source. `include_source: false` trims the source. | no |
| `create_template` | Create a template at version 1 from HTML, Liquid and a JSON Schema. | no |
| `update_template` | Publish a new immutable version. Earlier versions keep rendering. | no |
| `validate_data` | Dry-run a payload against the schema. Same field errors a render would give. Renders nothing. | no |
| `render` | Template plus data → a signed URL for a PDF, PNG or JPG. | **yes** — per PNG/JPG and per PDF page. Cache hits free. |
| `get_render` | Status by id, plus a freshly signed URL. Never re-renders. | no |
| `list_renders` | Recent renders, newest first, with a signed URL for each. | no |
| `usage` | Period usage, billable units by format, cost, free-tier balance, spend cap, trial balance. | no |
| `whoami` | The account, the plan, the PDF pages or images remaining on its trial or free tier, and **how this request authenticated**: `header`, `binding` or `trial`. | no |
| `create_account` | Email → a link and a request code → the owner confirms the request → API key. Upgrades this client's trial in place. An address that already has an account gets a link code instead of a second account. | no |
| `link_account` | Bind this client to an existing account with an `api_key` or a mailed `link_code`. For clients that cannot send headers, and only a client with an identity of its own. | no |
| `unlink_account` | Undo it, and revoke the key the binding held. | no |
| `rotate_key` | Mint a fresh API key for this account and show it once. A second call, with `confirm_saved` and the old key's id, revokes the old one — in that order, so nothing running on it stops before the new key is saved. | no |
| `upgrade` | A Stripe Checkout link, for a human to open. Charges nothing by itself. | no |
| `billing_portal` | A Stripe portal link: change plan, update the card, cancel. | no |

`render` is the only tool that's billed. It's billed per PDF page and per PNG or JPG, and a cache
hit (the same template version, data and options) comes back with `cached: true` for free.

A PDF page uses 2.5 times as much of a paid plan's allowance as an image does, so a month of both lands in between: on Solo, 100 PDF pages and 250 images use the whole allowance.

The keyless trial is 10 PDF pages or 10 images for its lifetime.
The free tier is 5 PDF pages or 5 images a month.
Plans and overage rates are on <https://galleyrender.com/pricing>.

## Templates

Templates are code: HTML and CSS with Liquid expressions, versioned like git, and each version
carries a JSON Schema that's the contract for its data. Validation errors are written for an agent
to act on. They give the field path, the expected type, what arrived and a value that would be
accepted.

Every account starts with the starter library of 43 templates, among them invoices,
quotes, change orders, receipts, statements, purchase orders, certificates, report covers and
tables, shipping labels, packing slips, OG and quote cards, event tickets and programs, badges,
menus, price sheets, letters, meeting agendas and minutes, and slide decks. They're all on
<https://galleyrender.com/templates>.

## Files here

| File | What it is |
|---|---|
| [`plugin.json`](./plugin.json) | Agent Plugins 1.1.0 manifest, the file directories look for first |
| [`mcp.json`](./mcp.json) | Agent Plugins MCP configuration: one `streamable-http` server |
| [`server.json`](./server.json) | What we publish to the official MCP registry, schema `2025-12-11`. It can run ahead of the registry between publishes |
| [`README.md`](./README.md) | This page |

There's no source code here and none is planned. The repository exists so directories that require
a public repository, like cursor.directory and mcp.so, have one to read.

`plugin.json` and `mcp.json` follow the [Agent Plugins](https://agent-plugins.org) standard
(formerly Open Plugins) and validate against its 1.1.0 schemas, which is what
[cursor.directory](https://cursor.directory) auto-detects from a repository URL. The standard
spells the transport `streamable-http`, and a hand-written `~/.cursor/mcp.json` uses the shorter
shape above. Cursor accepts either.

Every file here is copied from the Galley Render source, where it's checked against the live
catalog and the server's own tool list. A pull request here would be overwritten by the next copy,
so send suggestions to support@galleyrender.com instead.

## Not using MCP?

The tools call the same REST API, at `https://api.galleyrender.com`. It's described in OpenAPI 3.1
at <https://api.galleyrender.com/openapi.json>, and there are Node and Python clients:
<https://galleyrender.com/docs/sdks>.

## License

The files in this repository (`README.md`, `plugin.json`, `mcp.json` and `server.json`) are
released under the [MIT License](./LICENSE), so a directory or a client can copy them freely.

That license covers these manifest files only. It isn't a license to the Galley Render service,
its API, its templates or its name. Use of the service is governed by the
[terms](https://galleyrender.com/terms) and the [privacy policy](https://galleyrender.com/privacy).

Questions: support@galleyrender.com
