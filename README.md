# Custom Domains for Email Platforms: Automated Sending Domain Setup with SPF, DKIM, and DMARC

Every email marketing, transactional email, or outbound messaging platform hits the same onboarding wall: customers must connect a custom sending domain and publish SPF, DKIM, and DMARC records in DNS before their mail is worth delivering. This repository is a practical guide to that problem space: which DNS records email authentication actually requires, how domain ownership verification and propagation behave, why so many customers stall at the DNS step, and how to automate sending domain setup down to one click. It is maintained by the team behind [CustomDomain](https://customdomain.ai), and the guidance stands on its own whether or not you use the product.

## What is in this repository

| File | What it covers |
|---|---|
| This README | The problem space, how domain connection works, the three connection methods, and implementation paths |
| [docs/01-email-authentication-records.md](docs/01-email-authentication-records.md) | SPF, DKIM, DMARC, custom return path, and BIMI, with exact record examples |
| [docs/02-onboarding-sending-domains.md](docs/02-onboarding-sending-domains.md) | The customer setup funnel, verification UX, and why support tickets happen |
| [docs/03-deliverability-and-dns.md](docs/03-deliverability-and-dns.md) | DMARC alignment, common DNS failures, and what propagation really means |
| [docs/04-automating-domain-setup-for-email.md](docs/04-automating-domain-setup-for-email.md) | Automating SPF, DKIM, and DMARC publication with provider authorization, an API, or a widget |

## The problem: your activation funnel runs through someone else's DNS console

Your customer signed up to send email. To do that safely, they first have to prove they control a domain and publish a set of authentication records in a DNS console you do not operate, at a provider you did not choose, using terminology they have never seen.

The person doing this is usually a marketer, a founder, or an operations lead. They know what a newsletter is. They do not know what a TXT record is, whether "host" means `s1._domainkey` or `s1._domainkey.example.com`, or why their registrar and their DNS host might be two different companies.

The structural cause is a three party gap:

- **You** know exactly which records the domain needs. You cannot write them.
- **The DNS provider** can write them. It has no idea what your platform needs.
- **The customer** sits in the middle, copying strings between browser tabs and hoping.

The Domain Connect project's knowledge base ([github.com/Domain-Connect/knowledge-base](https://github.com/Domain-Connect/knowledge-base), published under CC0 1.0) documents how badly this goes at scale, using Microsoft 365 as its worked case: 7 to 15 DNS records, 16 help sites maintained by Microsoft with 10 of them registrar specific, and the headline figure, "approximately 50% of users who attempt manual DNS configuration fail and abandon the process" (`Knowledge Base/01_Problem_and_Context.md`). Email platforms sit in exactly the same position, usually with a shorter but stricter record set, because a single wrong character in an SPF record silently degrades deliverability rather than throwing an error.

The business impact for an email platform is direct:

- **Stalled activation.** A customer who never authenticates a domain never sends, and never converts to a paid plan.
- **Support load.** DNS setup tickets are slow, repetitive, and require screenshots of dozens of different provider consoles.
- **Misplaced blame.** When mail lands in spam because of a half finished SPF record, the customer blames your platform, not their DNS host.

This is the same class of problem every SaaS product with custom domains faces (see [custom domains for SaaS](https://customdomain.ai/custom-domains-for-saas)), with an extra twist: for email, the records are not just plumbing, they are the authentication chain that mailbox providers use to decide whether your customer's mail is trustworthy.

## The records your customers have to publish

A typical sending domain setup involves four to six records. Each one exists for a reason, and each one has its own failure modes.

| Record | DNS type | What it does | Typical shape |
|---|---|---|---|
| SPF | TXT | Authorizes your sending infrastructure for the domain used in the envelope sender | `"v=spf1 include:spf.yourplatform.example ~all"` |
| DKIM | CNAME (or TXT) | Publishes the public keys that verify your cryptographic signatures | `s1._domainkey` CNAME to a key you host |
| DMARC | TXT | Tells receivers what to do with mail that fails authentication, and where to send reports | `_dmarc` TXT `"v=DMARC1; p=none; rua=..."` |
| Custom return path | CNAME (or MX + TXT) | Puts the bounce address on the customer's domain so SPF aligns under DMARC | `bounce` CNAME to your bounce host |
| Tracking domain | CNAME + TLS | Brands click and open tracking links, needs a valid HTTPS certificate | `click` CNAME to your tracking edge |
| BIMI (optional) | TXT | Displays the brand logo in supporting inboxes, requires DMARC enforcement | `default._bimi` TXT pointing to an SVG |

A complete example set for a customer domain `example.com` sending through a platform at `yourplatform.example`:

```dns
example.com.                 TXT    "v=spf1 include:spf.yourplatform.example ~all"
s1._domainkey.example.com.   CNAME  s1.dkim.yourplatform.example.
s2._domainkey.example.com.   CNAME  s2.dkim.yourplatform.example.
bounce.example.com.          CNAME  bounces.yourplatform.example.
_dmarc.example.com.          TXT    "v=DMARC1; p=none; rua=mailto:dmarc@example.com"
click.example.com.           CNAME  tracking.yourplatform.example.
```

What each record does, why the syntax is strict, and where customers get it wrong is covered in depth in [docs/01-email-authentication-records.md](docs/01-email-authentication-records.md).

## How domain connection actually works

"Connect your domain" hides four distinct technical steps. Understanding them separately makes both your UX and your support answers better.

### 1. Proving control of the zone

Before you send mail as `example.com`, you need proof the customer controls it. Two designs are in common use, and they are not equally good.

A **DNS challenge** generates a unique token, the customer publishes it (usually as a TXT record), and you confirm it resolves. This works, but it adds a record whose only purpose is the challenge, and customers routinely delete it later as clutter.

**Control proven by the write itself** avoids the extra record. If the customer authorized your service at their DNS provider, or supplied a scoped zone token, or manually created your delegated DKIM CNAMEs, then only someone with zone access could have made that happen. Presence of the working records is the proof. This is the model CustomDomain uses: there is no separate ownership challenge and no TXT token, and the [connection lifecycle](https://docs.customdomain.ai/docs/concepts/connections) runs `pending` to `propagating` to `live` with a `failed` terminal state.

Either way, proof is a point in time observation, so good systems re-check continuously afterward. Records that verified once can disappear during a website migration months later.

### 2. Record publication

The customer (or an automated flow acting with their consent) writes the records above into the zone. This is where most failures happen: duplicated SPF records, host fields with the domain appended twice, quoted values pasted with the quotes, CNAMEs placed behind an HTTP proxy feature so they stop resolving as plain DNS. [docs/02-onboarding-sending-domains.md](docs/02-onboarding-sending-domains.md) catalogs these failure modes.

### 3. Propagation

DNS changes are not instant, but they are also not the mythical "up to 48 hours." New records at authoritative nameservers are typically queryable in seconds to minutes. What takes time is cache expiry at recursive resolvers, bounded by the record's TTL, and negative caching when a resolver already asked for a name before it existed. Checking the authoritative nameservers directly sidesteps most of this. [docs/03-deliverability-and-dns.md](docs/03-deliverability-and-dns.md) explains what to poll and what to tell customers.

### 4. TLS where HTTPS is involved

Sending domains themselves do not need certificates, but branded tracking domains and hosted unsubscribe or preference pages do. Every `click.example.com` link resolves to infrastructure you run, so it needs a certificate issued for the customer's hostname, renewed automatically, forever. A tracking domain with an expired certificate throws browser warnings on every link in every email.

## Three ways to connect a domain

There are exactly three viable connection methods, in descending order of automation. A serious onboarding flow offers all three and picks the best one automatically based on where the domain's DNS is hosted.

### Method 1: One-click provider authorization

The flow detects the customer's DNS provider from the domain's nameservers, sends the customer to that provider to approve a scoped, pre-defined change set, and the records are written programmatically. The customer never sees a record. Done this way, a domain is typically live in about 30 seconds.

One mechanism behind this pattern is the Domain Connect protocol, an open standard maintained by a community of developers across multiple companies, in which DNS providers pre-vet templates of records so that a service can never write anything outside its approved scope. CustomDomain's one-click provider authorization combines that protocol with direct provider API integrations, which together cover more providers than the protocol alone: 63 DNS and registrar providers are catalogued, exactly 25 of them with an automatic path. See [one-click DNS setup](https://customdomain.ai/one-click-dns-setup) for how this looks to the end customer.

### Method 2: API token

For providers with a token model, the customer pastes a scoped DNS API token instead of clicking through an authorization screen. The records are then written and maintained programmatically. This suits technical customers and providers whose consoles make token creation easy. The token should be scoped to DNS edits on the single zone, never account wide.

### Method 3: Guided manual with automatic verification

When neither of the above is available, the fallback should still not be "here is a table of records, good luck." A guided manual flow shows exact copy-paste values tailored to the detected provider's console conventions (relative vs. fully qualified host names, quoting rules), then polls DNS automatically and confirms each record as it appears. The customer clicks nothing to verify; the flow simply turns green. Our [guide to setting up a custom domain](https://customdomain.ai/guides/how-to-set-up-a-custom-domain) walks through the end user side of this.

### Where the 63 providers actually land

Three numbers get confused constantly, including in our own older copy, so here they are separately. The live census is `GET https://api.customdomain.ai/v1/providers/census` and you can tally it yourself.

| Path | Providers | What the customer does |
|---|---|---|
| One-click provider authorization (OAuth) | 6 | Approves a change set at their own DNS provider |
| Provider-hosted Domain Connect | 2 | Approves a pre-vetted template on the provider's own consent screen |
| API token | 17 | Pastes a scoped zone token |
| Guided manual with automatic verification | 38 | Copies provider-shaped values; verification is automatic |
| **Total catalogued** | **63** | of which **exactly 25** have an automatic path |

Separately, the Go provider registry ships 38 built-in adapters, a different quantity that counts code paths rather than catalogued providers. If you see "63 auto-configured" or "one-click across 63" anywhere, including on our marketing pages, it is wrong.

## What CustomDomain publishes for email specifically

Everything above is generic advice. This section is the checkable evidence behind the email claims, because an email platform evaluating a vendor should be able to read the artifacts rather than the brochure.

**Public Domain Connect templates, including five for email.** CustomDomain publishes 18 templates merged upstream into [Domain-Connect/Templates](https://github.com/Domain-Connect/Templates). As of 2026-08-19 that repository holds 1,120 template files across 696 provider domains; customdomain.ai contributes 18 of them, second only to goentri.com (Entri) with 77. Five are email templates and you can read the JSON directly:

| Template file | What it applies |
|---|---|
| `customdomain.ai.email-mx.json` | Inbound mail routing (MX) |
| `customdomain.ai.email-spf.json` | An SPF include, as an `SPFM` (SPF merge) record |
| `customdomain.ai.email-dkim.json` | The DKIM selector CNAME under `_domainkey` |
| `customdomain.ai.email-dmarc.json` | A starter `_dmarc` TXT at `p=none` with an `rua` address |
| `customdomain.ai.email-full.json` | All four together as one record group |

The `SPFM` type is the part worth looking at. It is the Domain Connect record type that extends an existing SPF record instead of writing a second one, which is the single most damaging manual setup mistake in email (see [docs/01](docs/01-email-authentication-records.md)). The template encodes the merge, so the DNS provider performs it, not the customer.

**SPF merge is the default in the API too, not an option you have to find.** `POST /v1/connections` accepts `override_spf` (default `false`), and the flag exists to opt *out* of merging. On the API token rail, "`SPFM` records are merged into any existing SPF TXT at write time (never clobbered)" ([apply records and go live](https://docs.customdomain.ai/docs/connect-flow/verify-and-go-live)). The same create call accepts `validate_dmarc` to make DMARC part of verification, and `validate_caa` to check CAA before certificate issuance.

**An MCP tool that does the whole email record set in one call.** The hosted MCP server ships twelve tools, one of which is `add-email`: "Configure a domain for a mail provider in one step via server-side templates." None of the twelve accepts raw DNS records as input, which is a deliberate choice; record values are computed by the control plane from vetted templates, so a prompt injection cannot end in an arbitrary DNS write. Tool list and transport details: [customdomain-mcp](https://github.com/CUSTOM-DOMAIN-APP/customdomain-mcp).

**An endpoint built for exactly this use case.** `PUT /v1/connections/{id}/records` lets you supply the record set instead of using ours, and its own description in the served spec names the motivating case: "records that cannot be derived from the domain, the motivating case is Amazon SES email verification, whose three Easy-DKIM CNAMEs are minted per domain by SES." That is what makes this usable by a platform that mints its own per-domain DKIM selectors.

## A worked example: SES-style per-domain DKIM, end to end

Four calls, using the hosted control plane at `https://api.customdomain.ai`. The served OpenAPI 3.1 spec is at [/v1/openapi.json](https://api.customdomain.ai/v1/openapi.json) and currently describes 67 paths.

```bash
# 1. Create the connection. Idempotent per application + domain, so it is safe
#    to call every time your onboarding screen loads.
curl -X POST https://api.customdomain.ai/v1/connections \
  -H "Authorization: Bearer $CD_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domain": "mail.example.com", "validate_dmarc": true}'
# -> 201 { "id": "con_...", "provider_id": "cloudflare",
#          "setup_type": "automatic", "status": "pending", "records": [ ... ] }
```

`provider_id` is the detected DNS host and `setup_type` tells you which rail the customer will be offered. Read them before you render anything, so the screen says "your DNS is at Cloudflare" rather than showing a generic record table.

```bash
# 2. Replace the desired record set with your platform's email records.
#    Server-to-server only (sk_ key), hosts relative to the connection, max 25 records.
curl -X PUT https://api.customdomain.ai/v1/connections/con_.../records \
  -H "Authorization: Bearer $CD_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"records": [
        {"type": "CNAME", "host": "s1._domainkey", "value": "s1.dkim.yourplatform.example"},
        {"type": "CNAME", "host": "s2._domainkey", "value": "s2.dkim.yourplatform.example"},
        {"type": "CNAME", "host": "bounce",        "value": "bounces.yourplatform.example"},
        {"type": "TXT",   "host": "_dmarc",
         "value": "v=DMARC1; p=none; rua=mailto:dmarc@example.com"}
      ]}'
# -> 200 { "connection_id": "con_...", "records_source": "integrator", "records": [ ... ] }
```

SPF is deliberately absent from that list. Leave it to the merge behavior rather than declaring a full replacement TXT, unless you have read the customer's existing record and intend to own it.

```bash
# 3. Send the customer down the best available rail. For OAuth, open authorize_url
#    in a popup; the customer signs in at their own provider and approves.
curl -X POST https://api.customdomain.ai/v1/connections/con_.../oauth:start \
  -H "Authorization: Bearer $CD_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"return_origin": "https://app.yourplatform.example"}'
# -> { "authorize_url": "https://<provider>/authorize?..." }
```

```bash
# 4. Do not poll in a loop. Subscribe once and let the state machine tell you.
curl -X POST https://api.customdomain.ai/v1/webhooks \
  -H "Authorization: Bearer $CD_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://api.yourplatform.example/hooks/cd",
       "events": ["connection.live", "connection.failed", "domain.record_missing"]}'
# -> { "id": "whk_...", "secret": "whsec_...", ... }   secret is returned once
```

A background poller resolves each desired record against public DNS on a one minute interval. When they all match, the connection flips to `live` and `connection.live` fires exactly once. That is your signal to enable sending. A `propagating` connection whose records never appear inside 24 hours becomes `failed` with `error_code: "propagation_timeout"`; a manual connection gets 72 hours before `setup_incomplete`. Wire both to a nudge email, not to silence.

Full request and response schemas: [create a connection](https://docs.customdomain.ai/docs/connect-flow/create-a-connection), [apply records and go live](https://docs.customdomain.ai/docs/connect-flow/verify-and-go-live), [webhooks](https://docs.customdomain.ai/docs/webhooks/overview).

## Implementation paths: widget or API

Both paths sit on the same underlying machinery (provider detection, record management, verification, monitoring). The difference is how much UX you want to own.

**Embed the connect widget.** A drop-in modal handles provider detection, chooses the best connection method, walks the customer through it, and reports status back to your app. This is the fastest path to shipping and inherits improvements (new providers, better error handling) without code changes on your side. See the [connect domain widget](https://customdomain.ai/connect-domain-widget). The SDKs are published as [`customdomain-js`](https://www.npmjs.com/package/customdomain-js) and [`@customdomain/react`](https://www.npmjs.com/package/@customdomain/react).

**Build on the REST API.** Full control over the flow: create a connection, fetch or supply the exact records for the domain, drive verification, subscribe to webhooks for state changes and drift, and query TLS and monitoring status. The API also covers registrar search and purchase if you want customers to buy a domain without leaving your app. See the [custom domain API](https://customdomain.ai/custom-domain-api) and the [developer docs](https://docs.customdomain.ai/docs).

**Expose it to AI agents.** If your customers configure your platform through AI assistants, a hosted MCP server lets agents run the same connect flow with OAuth scoped credentials. See [MCP server](https://customdomain.ai/mcp-server), [CustomDomain for AI agents](https://customdomain.ai/for/ai-agents), and the [customdomain-mcp repository](https://github.com/CUSTOM-DOMAIN-APP/customdomain-mcp).

### Decision table

| Your situation | Best fit |
|---|---|
| You want sending domain setup live this sprint | Widget |
| You have a bespoke onboarding flow and want to own every screen | REST API |
| You already generate SPF/DKIM values and only want DNS automation and verification | REST API (`PUT /v1/connections/{id}/records` plus the rails) |
| Customers ask an AI assistant to "set up my sending domain" | MCP server |
| You resell to agencies managing many client domains | API + webhooks, see [agencies and white label](https://customdomain.ai/for/agencies-white-label) |

## How this compares to Entri

Entri is the established vendor in this category and most readers arriving from a search are comparing the two. Both prices below were read from the vendors' own pricing pages on 2026-08-19.

| | CustomDomain | Entri |
|---|---|---|
| Free tier | Starter, $0, 10 domain connections/yr | none published |
| Entry paid tier | Startup, $149/mo, 600 connections/yr | Startup, $249/mo, 600 connections/yr |
| Next tier | Growth, $649/mo, 600/yr plus overage | Growth, "Talk to Sales", 2,400/yr |
| Top tiers | Premium and Enterprise, custom | Premium and Enterprise, "Talk to Sales" |
| Source | [customdomain.ai/pricing](https://customdomain.ai/pricing) | entri.com/pricing, which redirects to www.entri.com/plans |

Where we are behind, stated plainly:

- **Provider coverage.** Entri's plans page advertises support for "70+ DNS providers." Our census lists 63 catalogued, 25 of them automatic. The two numbers are not measured the same way and we have not audited theirs, but we are not going to claim a lead we cannot demonstrate.
- **SSO and SCIM.** Entri lists Enterprise SSO (Okta, SAML, SCIM) on its Enterprise tier. CustomDomain has neither. Our own docs say so directly: SSO and SCIM "are *not* built and no tier grants them," and the SCIM surface is deliberately unmounted ([plans and quotas](https://docs.customdomain.ai/docs/billing/plans-and-quotas)). If a security questionnaire requires SAML today, we are not the answer.
- **The free tier is for evaluation, not production.** Starter is the only tier that is hard capped: it refuses connections past its quota with `402 quota_exceeded`, and the annual 10 is metered as 1 per calendar month. Paid tiers are never refused.
- **Drift alerting is off by default.** The hourly monitor sweep runs, and `POST /v1/monitor:check` gives you the same comparison synchronously, but the `domain.record_missing` and `domain.record_restored` webhooks are gated behind a deployment flag that is off on the hosted service today. Build your drift alerting on the synchronous check until that flips.

Where the difference matters for an email platform specifically: Entri lists "Advanced security and email domain setup" starting at its Premium tier, which is a "Talk to Sales" price. CustomDomain's email templates and the `add-email` MCP tool are on the `Connect` entitlement, which is included on every plan including Starter. The entitlements that do start higher are `Secure` (the `/v1/ssl*` certificate management surface) and `Power` (the reverse-proxy surface), both of which first appear on Growth. Automatic TLS on a connected host is part of the base connect flow and is not one of those.

## FAQ

**Should customers send from the apex domain or a subdomain?**
A dedicated subdomain such as `mail.example.com` or `news.example.com` is usually better: it isolates your platform's reputation from the customer's corporate mail, avoids SPF lookup budget conflicts with their existing record, and keeps DMARC policy decisions independent. The apex works when the customer's whole outbound identity runs through you. The tradeoffs are the same as for any custom hostname; see [custom domain vs. subdomain](https://customdomain.ai/glossary/custom-domain-vs-subdomain).

**Will connecting a sending domain break the customer's existing email?**
Not if the record set is designed correctly. Sending records (DKIM CNAMEs, a bounce subdomain, a tracking subdomain) are additive. The one dangerous spot is SPF: the customer's apex almost always has an existing `v=spf1` record, and the platform's mechanism must be merged into it, never added as a second record. Two SPF records on one name is a permanent error under RFC 7208 and can fail authentication for all of the domain's mail, including mail that has nothing to do with your platform.

**Why did a domain verify successfully and then fail weeks later?**
DNS drift. Site migrations, agency handoffs, and "cleanup" of unrecognized records regularly delete authentication records that were working. Verification must be continuous, not a one time gate. CustomDomain re-checks connected domains on an hourly sweep and exposes the same comparison synchronously at `POST /v1/monitor:check`, so you can alert the customer before mailbox providers notice. The `domain.record_missing` webhook that would push that to you is gated behind a deployment flag that is off on the hosted service today, so poll the check endpoint rather than waiting for the event.

**Do we really need a custom return path domain?**
If you want SPF to count under DMARC, yes. SPF is evaluated against the envelope sender (the MAIL FROM domain), and DMARC only accepts an SPF pass when that domain aligns with the visible From domain. A shared platform bounce domain passes SPF but never aligns. Details in [docs/03-deliverability-and-dns.md](docs/03-deliverability-and-dns.md).

**How long does propagation really take?**
For a new record, authoritative servers usually answer within seconds to a few minutes. Delays customers actually experience come from resolver caches (bounded by TTL) and negative caching from checks that ran before the record existed. Poll the authoritative nameservers and you can confirm success almost immediately.

**What about customers on a DNS host you cannot integrate with?**
They get the guided manual flow: provider aware copy-paste instructions plus automatic verification that confirms each record as it lands. Nobody should ever be left refreshing a help article. Across all three methods CustomDomain catalogues 63 DNS and registrar providers today, 38 of which land on this path.

## Deep dives

- [SPF, DKIM, DMARC, return path, and BIMI, record by record](docs/01-email-authentication-records.md)
- [Onboarding sending domains: the funnel, the UX, the tickets](docs/02-onboarding-sending-domains.md)
- [Deliverability and DNS: alignment, failures, propagation](docs/03-deliverability-and-dns.md)
- [Automating domain setup for email platforms](docs/04-automating-domain-setup-for-email.md)

## Sibling guides

Same structure, different vertical. Use these when a customer's use case straddles two of them, which happens often: an agency running a client's newsletter, or a site builder that also sends transactional mail.

- [connect-domain-for-agencies](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-agencies) for white-label and multi-client fleets
- [connect-domain-for-ai-agents](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-ai-agents) for agent-driven configuration
- [connect-domain-for-website-builders](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-website-builders) for apex and www hosting
- [awesome-custom-domains](https://github.com/CUSTOM-DOMAIN-APP/awesome-custom-domains) for the wider solution space, including alternatives to us
- [docs](https://github.com/CUSTOM-DOMAIN-APP/docs) for the source of the product documentation

## About CustomDomain

CustomDomain is a product of EverJust Company, Minneapolis, Minnesota. It lets a platform's users connect their own domain in one click: automatic DNS configuration, propagation verification against public DNS, and automatic TLS issuance and renewal, delivered through an embeddable widget, a full REST API, and a hosted MCP server for AI agents. 63 DNS and registrar providers are catalogued and exactly 25 have an automatic path, across one-click provider authorization, API tokens, and provider-hosted Domain Connect. Pricing starts at $0.

- Product: [customdomain.ai](https://customdomain.ai)
- Docs: [docs.customdomain.ai](https://docs.customdomain.ai/docs)
- Pricing: [customdomain.ai/pricing](https://customdomain.ai/pricing)
- For SaaS platforms: [custom domains for SaaS](https://customdomain.ai/custom-domains-for-saas)
- For site builders: [customdomain.ai/for/site-builders](https://customdomain.ai/for/site-builders)
- For agencies: [customdomain.ai/for/agencies-white-label](https://customdomain.ai/for/agencies-white-label)
- Get started: [app.customdomain.ai/signup](https://app.customdomain.ai/signup)
- Contact: connect@customdomain.ai

Corrections to this repository are welcome, particularly to the numbers. See [CONTRIBUTING.md](CONTRIBUTING.md).

Parts of this repository draw on the Domain Connect protocol knowledge base published under the MIT license by a community of developers across multiple companies under CC0 1.0 (public domain). The Domain Connect protocol is an open standard of the Domain Connect project and is not affiliated with CustomDomain.
