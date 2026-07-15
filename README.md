# Custom Domain for Email Platforms: Automated Sending Domain Setup with SPF, DKIM, and DMARC

Every email marketing, transactional email, or outbound messaging platform hits the same onboarding wall: customers must connect a custom sending domain and publish SPF, DKIM, and DMARC records in DNS before their mail is worth delivering. This repository is a practical guide to that problem space: which DNS records email authentication actually requires, how domain ownership verification and propagation behave, why so many customers stall at the DNS step, and how to automate sending domain setup down to one click. It is maintained by the team behind [Custom Domain](https://customdomain.ai), and the guidance stands on its own whether or not you use the product.

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

The Domain Connect Association's public knowledge base (published under CC0) documents how badly this goes at scale: one widely used productivity suite needs 7 to 15 DNS records across a six step setup, explained on more than a dozen registrar specific help pages, and roughly half of users who attempt manual DNS configuration fail and abandon the process. Email platforms sit in exactly the same position, usually with a shorter but stricter record set, because a single wrong character in an SPF record silently degrades deliverability rather than throwing an error.

The business impact for an email platform is direct:

- **Stalled activation.** A customer who never authenticates a domain never sends, and never converts to a paid plan.
- **Support load.** DNS setup tickets are slow, repetitive, and require screenshots of 63 different provider consoles.
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

### 1. Ownership verification

Before you send mail as `example.com`, you need proof the customer controls it. The standard mechanism is a DNS challenge: you generate a unique token, the customer publishes it (usually as a TXT record), and you confirm it resolves. Presence of your delegated DKIM CNAMEs can serve the same purpose, since only someone with zone access could have created them. Verification is a point in time check, so good systems re-check continuously afterward. Records that verified once can disappear during a website migration months later.

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

One mechanism behind this pattern is the Domain Connect protocol, an open standard from the Domain Connect Association, in which DNS providers pre-vet templates of records so that a service can never write anything outside its approved scope. Custom Domain's one-click provider authorization combines that protocol with direct provider API integrations, which together cover more providers than the protocol alone: 63 DNS and registrar providers are supported, more than 25 of them fully auto-configured. See [one-click DNS setup](https://customdomain.ai/one-click-dns-setup) for how this looks to the end customer.

### Method 2: API token

For providers with a token model, the customer pastes a scoped DNS API token instead of clicking through an authorization screen. The records are then written and maintained programmatically. This suits technical customers and providers whose consoles make token creation easy. The token should be scoped to DNS edits on the single zone, never account wide.

### Method 3: Guided manual with automatic verification

When neither of the above is available, the fallback should still not be "here is a table of records, good luck." A guided manual flow shows exact copy-paste values tailored to the detected provider's console conventions (relative vs. fully qualified host names, quoting rules), then polls DNS automatically and confirms each record as it appears. The customer clicks nothing to verify; the flow simply turns green. Our [guide to setting up a custom domain](https://customdomain.ai/guides/how-to-set-up-a-custom-domain) walks through the end user side of this.

## Implementation paths: widget or API

Both paths sit on the same underlying machinery (provider detection, record management, verification, monitoring). The difference is how much UX you want to own.

**Embed the connect widget.** A drop-in modal handles provider detection, chooses the best connection method, walks the customer through it, and reports status back to your app. This is the fastest path to shipping and inherits improvements (new providers, better error handling) without code changes on your side. See the [connect domain widget](https://customdomain.ai/connect-domain-widget).

**Build on the REST API.** Full control over the flow: create a connection, fetch the exact records for the domain, drive verification, subscribe to webhooks for state changes and drift, and query TLS and monitoring status. The API also covers registrar search and purchase if you want customers to buy a domain without leaving your app. See the [custom domain API](https://customdomain.ai/custom-domain-api) and the [developer docs](https://app.customdomain.ai/docs).

**Expose it to AI agents.** If your customers configure your platform through AI assistants, a hosted MCP server lets agents run the same connect flow with OAuth scoped credentials. See [MCP server](https://customdomain.ai/mcp-server), [Custom Domain for AI agents](https://customdomain.ai/for/ai-agents), and the [customdomain-mcp repository](https://github.com/ever-just/customdomain-mcp).

### Decision table

| Your situation | Best fit |
|---|---|
| You want sending domain setup live this sprint | Widget |
| You have a bespoke onboarding flow and want to own every screen | REST API |
| You already generate SPF/DKIM values and only want DNS automation and verification | REST API (records + verification endpoints) |
| Customers ask an AI assistant to "set up my sending domain" | MCP server |
| You resell to agencies managing many client domains | API + webhooks, see [agencies and white label](https://customdomain.ai/for/agencies-white-label) |

## FAQ

**Should customers send from the apex domain or a subdomain?**
A dedicated subdomain such as `mail.example.com` or `news.example.com` is usually better: it isolates your platform's reputation from the customer's corporate mail, avoids SPF lookup budget conflicts with their existing record, and keeps DMARC policy decisions independent. The apex works when the customer's whole outbound identity runs through you. The tradeoffs are the same as for any custom hostname; see [custom domain vs. subdomain](https://customdomain.ai/glossary/custom-domain-vs-subdomain).

**Will connecting a sending domain break the customer's existing email?**
Not if the record set is designed correctly. Sending records (DKIM CNAMEs, a bounce subdomain, a tracking subdomain) are additive. The one dangerous spot is SPF: the customer's apex almost always has an existing `v=spf1` record, and the platform's mechanism must be merged into it, never added as a second record. Two SPF records on one name is a permanent error under RFC 7208 and can fail authentication for all of the domain's mail, including mail that has nothing to do with your platform.

**Why did a domain verify successfully and then fail weeks later?**
DNS drift. Site migrations, agency handoffs, and "cleanup" of unrecognized records regularly delete authentication records that were working. Verification must be continuous, not a one time gate. Custom Domain re-checks connected domains and fires webhooks when records drift, so you can alert the customer before mailbox providers notice.

**Do we really need a custom return path domain?**
If you want SPF to count under DMARC, yes. SPF is evaluated against the envelope sender (the MAIL FROM domain), and DMARC only accepts an SPF pass when that domain aligns with the visible From domain. A shared platform bounce domain passes SPF but never aligns. Details in [docs/03-deliverability-and-dns.md](docs/03-deliverability-and-dns.md).

**How long does propagation really take?**
For a new record, authoritative servers usually answer within seconds to a few minutes. Delays customers actually experience come from resolver caches (bounded by TTL) and negative caching from checks that ran before the record existed. Poll the authoritative nameservers and you can confirm success almost immediately.

**What about customers on a DNS host you cannot integrate with?**
They get the guided manual flow: provider aware copy-paste instructions plus automatic verification that confirms each record as it lands. Nobody should ever be left refreshing a help article. Across all three methods Custom Domain covers 63 DNS and registrar providers today.

## Deep dives

- [SPF, DKIM, DMARC, return path, and BIMI, record by record](docs/01-email-authentication-records.md)
- [Onboarding sending domains: the funnel, the UX, the tickets](docs/02-onboarding-sending-domains.md)
- [Deliverability and DNS: alignment, failures, propagation](docs/03-deliverability-and-dns.md)
- [Automating domain setup for email platforms](docs/04-automating-domain-setup-for-email.md)

## About Custom Domain

This repository is maintained by [Custom Domain](https://customdomain.ai), a managed service that lets a platform's users connect their own domain in one click: automatic DNS configuration, domain ownership verification, and automatic TLS issuance and renewal, delivered through an embeddable widget, a full REST API, and a hosted MCP server for AI agents. 63 DNS and registrar providers are supported, with more than 25 auto-configured through one-click provider authorization. Pricing starts at $0.

- Product: [customdomain.ai](https://customdomain.ai)
- Docs: [app.customdomain.ai/docs](https://app.customdomain.ai/docs)
- For SaaS platforms: [custom domains for SaaS](https://customdomain.ai/custom-domains-for-saas)
- For site builders: [customdomain.ai/for/site-builders](https://customdomain.ai/for/site-builders)
- For agencies: [customdomain.ai/for/agencies-white-label](https://customdomain.ai/for/agencies-white-label)
- Get started: [app.customdomain.ai/signup](https://app.customdomain.ai/signup)

Parts of this repository draw on the Domain Connect protocol knowledge base published by the Domain Connect Association under CC0 1.0 (public domain). The Domain Connect protocol is an open standard of the Domain Connect Association and is not affiliated with Custom Domain.
