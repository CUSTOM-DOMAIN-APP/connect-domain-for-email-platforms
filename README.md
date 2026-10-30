# Custom sending domains for email platforms

SPF, DKIM, DMARC, return-path and MX records for Amazon SES, SendGrid, Postmark and Resend, and how to get them
into a customer's DNS without breaking the mail they already send.

[Email DNS records in the docs](https://customdomain.ai/gh/email) · [Bring your own records](https://docs.customdomain.ai/docs/dns/bring-your-own-records) · [Start free: first ten domains, no card](https://app.customdomain.ai/signup)

Every email platform hits the same onboarding wall: before a customer's mail is worth delivering, they have to
publish SPF, DKIM and DMARC records in a DNS console you do not operate. This repository covers which records
email authentication actually requires, why customers stall, and how to automate publishing the whole set.
Maintained by [CustomDomain™](https://customdomain.ai). The guidance stands whether or not you use the product.

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
rather than showing a generic record table. Then subscribe to `connection.live` rather than polling. Sign up at
[app.customdomain.ai/signup](https://app.customdomain.ai/signup); every workspace has a default API key on its
**API & SDK** page.

## In this guide

- **SPF, DKIM, DMARC, return path and BIMI**, record by record, with exact examples. [docs/01](docs/01-email-authentication-records.md)
- **The onboarding funnel and its failure modes**: duplicated SPF, doubled host fields, pasted quotes, proxied CNAMEs. [docs/02](docs/02-onboarding-sending-domains.md)
- **Deliverability and DNS**: DMARC alignment, why a shared bounce domain passes SPF but never aligns. [docs/03](docs/03-deliverability-and-dns.md)
- **Automating publication** through provider authorization, an API or a widget. [docs/04](docs/04-automating-domain-setup-for-email.md)

## How it works

A sending domain needs four to six records, each with its own failure modes. SPF authorizes your infrastructure
for the envelope sender. DKIM publishes the keys that verify your signatures. DMARC tells receivers what to do
with failures. A custom return path puts the bounce address on the customer's domain so SPF aligns under DMARC.
A tracking subdomain needs a real certificate, renewed forever, or every link in every email throws a browser
warning.

Getting those records into the customer's zone reduces to three rails. The split below is the live census at
`GET https://api.customdomain.ai/v1/providers/census`, counted 2026-10-30.

| Rail | Providers (of 189) | What the customer does |
|---|---|---|
| One-click provider authorization | 7 (6 provider OAuth, 1 provider-hosted Domain Connect) | Approves a scoped change set at their own provider |
| Scoped API token | 66 | Pastes one zone-scoped token |
| Guided manual with automatic verification | 116 | Copies provider-shaped values; verification is automatic |

Control is proven by whichever rail wrote the records, so there is no separate ownership challenge and no TXT
token to keep alive. Verification is continuous, not a one-time gate: records that verified once disappear
during a site migration months later. Of 189 known providers, 73 have an automatic path. Any claim that all of
them are configured automatically, including on our own pages, is wrong.

## DNS records for Amazon SES, SendGrid, Postmark and Resend

These are the sending services platforms most often put behind a customer's domain. Each one mints its own
values per domain (tokens, selectors, public keys), so the rows below are shapes, not values to copy: your
platform reads the real values from the service's API or dashboard and gets them into the customer's zone.
Checked against each vendor's documentation on 2026-10-20.

| Service | DKIM | SPF and return path | Worth knowing |
|---|---|---|---|
| [Amazon SES](https://docs.aws.amazon.com/ses/latest/dg/send-email-authentication-dkim-easy.html) | Easy DKIM: three `CNAME`s, `<token>._domainkey` to `<token>.dkim.amazonses.com` | A [custom MAIL FROM](https://docs.aws.amazon.com/ses/latest/dg/mail-from.html) subdomain with one `MX`, `10 feedback-smtp.<region>.amazonses.com`, and a `TXT`, `"v=spf1 include:amazonses.com ~all"` | Without a custom MAIL FROM, SPF passes for `amazonses.com` and DMARC has to align through DKIM. The MAIL FROM subdomain must hold exactly one `MX` |
| [SendGrid](https://www.twilio.com/docs/sendgrid/ui/account-and-settings/how-to-set-up-domain-authentication) | `CNAME`s `s1._domainkey` and `s2._domainkey` to `s1.domainkey.u<id>.wl<id>.sendgrid.net` and its `s2` twin | `CNAME` `em<id>` to `u<id>.wl<id>.sendgrid.net` | That is the default automated security setup, where SendGrid keeps the keys behind the `CNAME`s current. With it off you get an `MX` and three `TXT` records to maintain yourself |
| [Postmark](https://postmarkapp.com/support/article/how-do-i-verify-a-domain) | A `TXT` at the selector Postmark shows under `_domainkey` | A Return-Path `CNAME`, at the hostname Postmark shows, to `pm.mtasv.net` | No SPF record to add: the custom Return-Path is what makes SPF pass and align |
| [Resend](https://resend.com/docs/add-a-domain) | A `TXT` at `resend._domainkey`, value starting `p=` | On the `send` subdomain: `MX` `10 feedback-smtp.<region>.amazonses.com` and `TXT` `"v=spf1 include:amazonses.com ~all"` | Receiving mail is a separate, optional `MX` |

All four also document a DMARC record at `_dmarc`. Start at `p=none` with a `rua` address, read the reports,
then tighten. The record-by-record reasoning is in [docs/01](docs/01-email-authentication-records.md).

**Where CustomDomain™ fits.** None of these values can be derived from the domain, which is why
`PUT /v1/connections/{id}/records` exists (the Quickstart above uses it). Create the connection, `PUT` the
records the sending service returned (an API key on your server, hosts relative to the connection, at most 25),
and CustomDomain™ writes them with the customer's provider credential where a rail can, shows them for manual
setup where it cannot, and verifies them in public DNS before the connection goes live. The Domain Connect rail
refuses a supplied record set, because it can only apply published templates. A customer who already sends
through another service keeps one valid SPF record, because SPF is merged rather than replaced by default
(`override_spf` is `false`).

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
provider in one call. None of the twenty MCP tools accepts a raw DNS record, so a prompt-injected agent cannot
end up writing one.

## Where this loses

Plan prices and allowances are in the facts block below. Entri, read 2026-08-19, publishes no entry tier and
charges $249/mo for 600 connections a year, against $149/mo for the same 600 on Startup here. Entri lists
"advanced security and email domain setup" from its Premium tier. Here the email templates and the `add-email`
tool are on every plan. HTTPS served by the CustomDomain™ edge, and the certificate endpoints (`/v1/ssl*`), start
on Growth. On Starter and Startup, domains point at your own servers and you serve TLS.

## Limits and known gaps

- **Provider coverage.** Entri's plans page advertises "70+ DNS providers". The census here lists 189 known, 73 automatic (2026-10-30). The two numbers are not measured the same way and we have not audited theirs, so treat neither as a lead.
- **SSO and SCIM are not built**, and no tier grants them. Entri lists Enterprise SSO with SAML and SCIM. If a security questionnaire requires SAML today, this is not the answer.
- **Drift alerting is off by default.** The hourly monitor sweep runs and `POST /v1/monitor:check` gives the same comparison synchronously, but `domain.record_missing` and `domain.record_restored` are gated behind a deployment flag the hosted service leaves off. Build on the synchronous check.
- **Starter is for evaluation and small volumes.** It is the only hard-capped tier: past quota it returns `402 quota_exceeded`, and the annual 10 is metered as 1 per calendar month.

<!-- facts:start -->
## Facts and sources

Read from the live endpoints on 2026-10-30. Every number in this README traces to a row here or to a source named
where it is used. If one does not, open an issue with the file and line.

| Fact | Value | Source |
|---|---|---|
| DNS providers known | 189 | [`/v1/providers/census`](https://api.customdomain.ai/v1/providers/census) |
| Automatic path | 73: 66 by scoped API key, 6 by provider sign-in (OAuth), 1 by one-click setup (Domain Connect) | census `mode` |
| Exact records and provider steps | 116 | census `mode` |
| Going live | No TXT ownership record. A domain goes `live` when every record resolves. Statuses: `pending`, `propagating`, `live`, `failed` | [Connections](https://docs.customdomain.ai/docs/concepts/connections) |
| HTTPS on the CustomDomain™ edge | Growth and up. On Starter and Startup, domains point at your servers and you serve TLS | [`/v1/plans`](https://api.customdomain.ai/v1/plans) |
| Plans | Starter $10 a month, 10 domains a year, hard cap. Startup $149 a month, 600 a year. Growth $649 a month, 600 a year. Premium and Enterprise by contract, 12,000 a year. 14 day trial on Starter, Startup and Growth | [`/v1/plans`](https://api.customdomain.ai/v1/plans) |
| Free to start | A new workspace connects its first ten domains free. No card needed to sign up | [Plans and quotas](https://docs.customdomain.ai/docs/billing/plans-and-quotas) |
| REST API | 163 operations on 140 paths, OpenAPI 3.1.0 | [`/v1/openapi.json`](https://api.customdomain.ai/v1/openapi.json) |
| MCP server | `customdomain-mcp` 0.6.0, 20 tools. Buying a domain through MCP is off on the hosted service | [`mcp.json`](https://mcp.customdomain.ai/.well-known/agent/mcp.json) |

Check the provider split yourself:

```sh
curl -s https://api.customdomain.ai/v1/providers/census \
  | python3 -c 'import json,sys,collections; d=json.load(sys.stdin); print(d["count"], collections.Counter(p["mode"] for p in d["providers"]))'
# 189 Counter({'manual': 116, 'api': 66, 'oauth': 6, 'dc': 1})
```
<!-- facts:end -->

The sourcing table for corrections is in [CONTRIBUTING.md](./CONTRIBUTING.md).

## Related

- [customdomain-mcp](https://github.com/CUSTOM-DOMAIN-APP/customdomain-mcp): the hosted MCP server, including the `add-email` tool
- [customdomain-sdk](https://github.com/CUSTOM-DOMAIN-APP/customdomain-sdk): the browser SDK `customdomain-js`, for a connect step inside your onboarding
- [awesome-custom-domains](https://github.com/CUSTOM-DOMAIN-APP/awesome-custom-domains): the curated list of the category, with its email authentication section

## Support and license

Problem framing draws on the [Domain Connect knowledge base](https://github.com/Domain-Connect/knowledge-base) (CC0 1.0).

Questions go to [GitHub Discussions](https://github.com/CUSTOM-DOMAIN-APP/docs/discussions) and corrections to
[this repository's issues](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-email-platforms/issues). Account and billing:
connect@customdomain.ai. Report security issues privately to security@customdomain.ai, never in a public issue
([policy](https://app.customdomain.ai/security)).

[MIT](./LICENSE). CustomDomain™ is a product of EverJust Company.
