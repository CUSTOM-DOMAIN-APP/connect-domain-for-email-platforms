# CustomDomain™ for Email Platforms

Connect custom email domains to your email platform: automated SPF, DKIM, DMARC and MX setup.

**Status:** Maintained guide · live API numbers re-verified on 2026-09-26 · public

[![docs](https://img.shields.io/badge/docs-docs.customdomain.ai-1c1917?style=flat)](https://docs.customdomain.ai/docs)
[![license](https://img.shields.io/badge/license-MIT-1c1917?style=flat)](./LICENSE)

[Website](https://customdomain.ai) · [Docs](https://docs.customdomain.ai/docs) · [Console](https://app.customdomain.ai) · [Email DNS docs](https://docs.customdomain.ai/docs/dns/email-dns) · [MCP server](https://customdomain.ai/mcp-server)

|  |  |
|---|---|
| **What it is** | Implementation guide to automating sending-domain setup: SPF, DKIM, DMARC, MX |
| **Who it's for** | Built for marketing, transactional and outbound messaging platforms |
| **Live at** | docs at [docs.customdomain.ai](https://docs.customdomain.ai/docs) · product at [customdomain.ai](https://customdomain.ai) |
| **Stack** | Markdown guide · REST API, OpenAPI 3.1 · hosted MCP server · Domain Connect templates |
| **Status** | Maintained · 63 providers and 5 plans re-counted from the live API 2026-09-26 |

Every email platform hits the same onboarding wall: before a customer's mail is worth delivering, they have to
publish SPF, DKIM and DMARC records in a DNS console you do not operate. This repository covers which records
email authentication actually requires, why customers stall, and how to automate the whole set down to one
click. Maintained by [CustomDomain™](https://customdomain.ai); the guidance stands whether or not you use the
product.

## The problem: your activation funnel runs through someone else's DNS console

The person doing the setup is usually a marketer, a founder or an operations lead. They know what a newsletter
is. They do not know what a TXT record is, whether "host" means `s1._domainkey` or `s1._domainkey.example.com`,
or why their registrar and their DNS host might be two different companies. You know exactly which records the
domain needs and cannot write them. Their DNS provider can write them and has no idea what your platform needs.
The customer sits in the middle copying strings between browser tabs.

The Domain Connect knowledge base (CC0 1.0) documents how badly that goes, using one major productivity suite as
its worked case: 7 to 15 DNS records, 16 help sites with 10 of them registrar-specific, and the headline figure
that roughly half of the users who attempt manual DNS configuration abandon it. Email platforms sit in the same
position with a shorter but stricter record set, because one wrong character in SPF degrades deliverability
silently instead of throwing an error.

The cost lands in three places: a customer who never authenticates never sends and never converts, DNS tickets
are slow and need screenshots of dozens of provider consoles, and mail that lands in spam because of a
half-finished SPF record gets blamed on your platform, not their DNS host.

## Quickstart

Create the connection, then supply your own per-domain records. The `PUT` exists for record sets that cannot be
derived from the domain, and its motivating case is exactly this one: Amazon SES mints three Easy-DKIM CNAMEs
per domain. Server-to-server only, hosts relative to the connection, 25 records maximum.

```bash
curl -X POST https://api.customdomain.ai/v1/connections \
  -H "Authorization: Bearer $CUSTOMDOMAIN_API_KEY" -H "Content-Type: application/json" \
  -d '{"domain": "mail.example.com", "validate_dmarc": true}'
# -> 201 { "id": "con_...", "provider_id": "cloudflare", "setup_type": "automatic", ... }

curl -X PUT https://api.customdomain.ai/v1/connections/con_.../records \
  -H "Authorization: Bearer $CUSTOMDOMAIN_API_KEY" -H "Content-Type: application/json" \
  -d '{"records": [
        {"type":"CNAME","host":"s1._domainkey","value":"s1.dkim.yourplatform.example"},
        {"type":"CNAME","host":"bounce",       "value":"bounces.yourplatform.example"},
        {"type":"TXT",  "host":"_dmarc","value":"v=DMARC1; p=none; rua=mailto:dmarc@example.com"}]}'
```

Read `provider_id` and `setup_type` before rendering anything, so your screen says "your DNS is at Cloudflare"
rather than showing a generic record table. Then subscribe to `connection.live` rather than polling. Free-tier
keys: [app.customdomain.ai/signup](https://app.customdomain.ai/signup).

## What it does

- **SPF, DKIM, DMARC, return path and BIMI**, record by record, with exact examples. [docs/01](docs/01-email-authentication-records.md)
- **The onboarding funnel and its failure modes**: duplicated SPF, doubled host fields, pasted quotes, proxied CNAMEs. [docs/02](docs/02-onboarding-sending-domains.md)
- **Deliverability and DNS**: DMARC alignment, why a shared bounce domain passes SPF but never aligns. [docs/03](docs/03-deliverability-and-dns.md)
- **Automating publication** through provider authorization, an API or a widget. [docs/04](docs/04-automating-domain-setup-for-email.md)
- **Publishes every number with its source**, so you can re-count before trusting it.

## How it works

A sending domain needs four to six records, each with its own failure modes. SPF authorizes your infrastructure
for the envelope sender. DKIM publishes the keys that verify your signatures. DMARC tells receivers what to do
with failures. A custom return path puts the bounce address on the customer's domain so SPF aligns under DMARC.
A tracking subdomain needs a real certificate, renewed forever, or every link in every email throws a browser
warning.

Getting those records into the customer's zone reduces to three rails. The split below is the live census at
`GET https://api.customdomain.ai/v1/providers/census`, counted 2026-09-26.

| Rail | Providers (of 63) | What the customer does |
|---|---|---|
| One-click provider authorization | 8 (6 provider OAuth, 2 provider-hosted Domain Connect) | Approves a scoped change set at their own provider |
| Scoped API token | 17 | Pastes one zone-scoped token |
| Guided manual with automatic verification | 38 | Copies provider-shaped values; verification is automatic |

Control is proven by whichever rail wrote the records, so there is no separate ownership challenge and no TXT
token to keep alive. Verification is continuous, not a one-time gate: records that verified once disappear
during a site migration months later. If you see "63 auto-configured" anywhere, including on our own marketing
pages, it is wrong.

## The email surface: templates, SPF merge, and the API

Five of the 18 Domain Connect templates published upstream to
[Domain-Connect/Templates](https://github.com/Domain-Connect/Templates) are email templates you can read as
JSON: `email-mx`, `email-spf`, `email-dkim`, `email-dmarc` and `email-full`. Open `email-spf` first. It uses the
`SPFM` (SPF merge) record type, which extends an existing SPF record instead of writing a second one, and makes
the DNS provider perform the merge. Two SPF records on one name is a permanent error under RFC 7208 that can
break authentication for all of the domain's mail, including mail that has nothing to do with your platform.

Merge is the default in the API too: `POST /v1/connections` takes `override_spf` (default `false`), and the flag
exists to opt *out*. The same call takes `validate_dmarc` to fold DMARC into verification and `validate_caa` to
check CAA before certificate issuance. On the hosted MCP server, `add-email` configures a domain for a mail
provider in one call. None of the twelve MCP tools accepts a raw DNS record, so a prompt-injected agent cannot
end up writing one.

## Repository layout

```text
.
├── README.md        # the problem, the records, the three rails, pricing
├── AGENTS.md        # machine-readable brief for coding agents working in this repo
├── CONTRIBUTING.md  # the sourcing rule: every number traces to a live endpoint
├── docs/            # 01 authentication records · 02 onboarding funnel
│                    # 03 deliverability and DNS · 04 automating setup
└── LICENSE          # MIT
```

Markdown only, no build step. Surfaces referenced from here: REST at `api.customdomain.ai` (OpenAPI 3.1: 68
paths, 80 operations, counted 2026-09-26), the widget published as `customdomain-js` on npm, and MCP at
`mcp.customdomain.ai/mcp` (registry id `ai.customdomain/mcp`).

## Pricing, and where this loses

Read from `GET https://api.customdomain.ai/v1/plans` on 2026-09-26. Entri prices read 2026-08-19.

| | CustomDomain™ | Entri |
|---|---|---|
| Free tier | $0, 10 connections/yr, hard capped | none published |
| Entry paid tier | Startup, $149/mo, 600/yr | Startup, $249/mo, 600/yr |
| Next tier | Growth, $649/mo, 600/yr plus metered overage | Growth, "Talk to Sales", 2,400/yr |
| Top tiers | Premium and Enterprise, contact sales, 12,000/yr | Premium and Enterprise, "Talk to Sales" |

Email templates and the `add-email` tool sit on the `Connect` entitlement, which every plan including Free has;
Entri lists "advanced security and email domain setup" from its Premium tier. The entitlements that do start
higher here are `Secure` (the `/v1/ssl*` surface) and `Power` (the reverse proxy), both first appearing on
Growth. Automatic TLS on a connected host is part of the base flow, not one of those.

## Limits and known gaps

- **Provider coverage.** Entri's plans page advertises "70+ DNS providers". The census here lists 63 catalogued, 25 automatic. The two numbers are not measured the same way and we have not audited theirs, so treat neither as a lead.
- **SSO and SCIM are not built**, and no tier grants them. Entri lists Enterprise SSO with SAML and SCIM. If a security questionnaire requires SAML today, this is not the answer.
- **Drift alerting is off by default.** The hourly monitor sweep runs and `POST /v1/monitor:check` gives the same comparison synchronously, but `domain.record_missing` and `domain.record_restored` are gated behind a deployment flag the hosted service leaves off. Build on the synchronous check.
- **Free is for evaluation, not production.** It is the only hard-capped tier: past quota it returns `402 quota_exceeded`, and the annual 10 is metered as 1 per calendar month.

## Corrections

Every number here traces to a live endpoint or a public repository, named at the point of use. If one does not,
that is a bug: open an issue with the file and line, and see [CONTRIBUTING.md](./CONTRIBUTING.md) for the
sourcing table.

Problem framing draws on the [Domain Connect knowledge base](https://github.com/Domain-Connect/knowledge-base)
(CC0 1.0), an open standard maintained by a community across multiple companies and referenced here as prior
art.

## Related

- [connect-domain-for-agencies](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-agencies): CustomDomain™ for Agencies
- [connect-domain-for-ai-agents](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-ai-agents): CustomDomain™ for AI Agents
- [connect-domain-for-website-builders](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-website-builders): CustomDomain™ for Website Builders
- [customdomain-sdk](https://github.com/CUSTOM-DOMAIN-APP/customdomain-sdk): the browser SDK `customdomain-js` and the React wrapper `@customdomain/react`
- [customdomain-mcp](https://github.com/CUSTOM-DOMAIN-APP/customdomain-mcp): the hosted MCP server, including the `add-email` tool
- [docs](https://github.com/CUSTOM-DOMAIN-APP/docs): the CustomDomain™ documentation source, rendered at [docs.customdomain.ai](https://docs.customdomain.ai/docs)
- [awesome-custom-domains](https://github.com/CUSTOM-DOMAIN-APP/awesome-custom-domains): the curated list of the category, including the alternatives to this product

## Support

- **Docs:** [docs.customdomain.ai](https://docs.customdomain.ai/docs)
- **Questions and ideas:** [GitHub Discussions](https://github.com/CUSTOM-DOMAIN-APP/docs/discussions)
- **Bugs and corrections:** [open an issue](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-email-platforms/issues) on this repository
- **Service status:** [status.customdomain.ai](https://status.customdomain.ai)
- **Account and billing:** connect@customdomain.ai
- **Security:** report privately to security@customdomain.ai, never in a public issue. Policy: [app.customdomain.ai/security](https://app.customdomain.ai/security)

## License

[MIT](./LICENSE). CustomDomain™ is a product of EverJust Company.
