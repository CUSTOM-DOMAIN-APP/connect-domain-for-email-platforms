# AGENTS.md

Instructions for AI coding agents working with this repository or implementing custom domains.

## What this is

CustomDomain (customdomain.ai) is managed infrastructure that lets a platform's users connect their own domain in one click: automatic DNS configuration, propagation verification against public DNS, and automatic SSL/TLS issuance and renewal. There is no separate ownership challenge. Control of the zone is proven by the rail used to write the records (a provider OAuth authorization, a provider-hosted one-click apply, or a scoped API token); on the manual rail the poller simply waits for the records to appear.

63 DNS and registrar providers are catalogued. Exactly 25 have an automatic path (6 one-click provider authorization, 17 API token, 2 provider-hosted Domain Connect) and the remaining 38 use a guided manual flow with automatic verification. The live tally is `GET https://api.customdomain.ai/v1/providers/census`.

## Connect a domain in 3 steps (REST)

Base URL: `https://api.customdomain.ai` (API docs: https://docs.customdomain.ai/docs/api-reference)

```bash
# 1. Create a connection for the user's domain.
#    Idempotent per application + domain: an existing non-failed connection is
#    replayed with 200 instead of a duplicate being created.
curl -X POST https://api.customdomain.ai/v1/connections \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domain": "app.customer.com"}'

# 2. Start one-click provider authorization (or fall back to guided manual records)
curl -X POST https://api.customdomain.ai/v1/connections/<ID>/oauth:start \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"return_origin": "https://app.acme.example"}'

# 3. Read current state (DNS written, records resolving, TLS served at the edge)
curl https://api.customdomain.ai/v1/connections/<ID> \
  -H "Authorization: Bearer $API_KEY"
```

Statuses are `pending`, `propagating`, `live`, and `failed`. A background poller resolves each desired record against public DNS on a one minute interval and flips the connection to `live`, so prefer the `connection.live` webhook over polling step 3 in a loop.

Endpoint shapes are illustrative; always follow https://docs.customdomain.ai/docs/api-reference for exact schemas. The served spec is OpenAPI 3.1 at https://api.customdomain.ai/v1/openapi.json.

## MCP server (for agents)

Hosted MCP endpoint: `https://mcp.customdomain.ai/mcp` (streamable HTTP, OAuth client credentials via `https://mcp.customdomain.ai/token`). Twelve tools: search and register domains, create and check connections, detect the DNS provider, configure email DNS and forwarding, inventory connections, and track verification and TLS status. No tool accepts raw DNS records as input; record values are computed by the control plane from vetted templates, which closes off prompt injection paths that end in arbitrary DNS writes.

```bash
claude mcp add --transport http customdomain https://mcp.customdomain.ai/mcp
```

Docs: https://docs.customdomain.ai/docs/mcp/overview
Source: https://github.com/CUSTOM-DOMAIN-APP/customdomain-mcp

## Key references

- Product: https://customdomain.ai
- Documentation: https://docs.customdomain.ai/docs (agent index: https://docs.customdomain.ai/docs/llms.txt)
- Embeddable widget: https://customdomain.ai/connect-domain-widget
- Sign up (free tier): https://app.customdomain.ai/signup

## Conventions for edits in this repo

Markdown only. No em dashes or en dashes anywhere. Plain, technically accurate language. American English. Keep files under 300 KB.

Numbers are load-bearing in this repo and each one has a source. Before changing any of them, check the source rather than the surrounding prose: provider counts come from `GET https://api.customdomain.ai/v1/providers/census`, prices from https://customdomain.ai/pricing, MCP tool counts from https://github.com/CUSTOM-DOMAIN-APP/customdomain-mcp, and the manual-DNS failure statistics from https://github.com/Domain-Connect/knowledge-base.
