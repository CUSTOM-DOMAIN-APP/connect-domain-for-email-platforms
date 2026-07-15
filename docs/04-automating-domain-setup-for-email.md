# Automating Domain Setup for Email Platforms: Provider Authorization, API, and Widget

Everything in this repo so far describes a problem that instructions alone cannot fix: customers transcribing SPF, DKIM, DMARC, and return path records into 63 different DNS consoles, with failure modes that damage mail far beyond your platform. This doc covers the automation that removes the transcription step entirely: how one-click provider authorization writes email authentication records programmatically, when to fall back to API tokens or guided manual flows, and how to integrate the whole thing through a widget, a REST API, or an MCP server. Background: [the README](../README.md), [record reference](01-email-authentication-records.md), [funnel analysis](02-onboarding-sending-domains.md).

## The idea: consent instead of transcription

Manual setup asks the customer to be the API between your platform and their DNS host. Automation replaces that with a consent flow: the customer proves they control the domain and approves a specific, bounded set of changes, and software writes the records.

The flow, end to end:

1. **Detect.** The customer types their domain. A nameserver lookup identifies which DNS provider actually hosts the zone, which is often not the registrar they remember buying from.
2. **Route.** Based on the provider, the flow picks the best available method: one-click authorization, API token, or guided manual with automatic verification.
3. **Authorize.** For one-click, the customer is sent to their own provider, signs in there (credentials never touch your platform), and approves the change set.
4. **Apply.** The records are written programmatically: DKIM CNAMEs created, return path delegated, DMARC published if absent, and the SPF mechanism **merged** into any existing record.
5. **Verify and watch.** Verification runs against authoritative nameservers, typically confirming in seconds. A domain connected this way is usually live in about 30 seconds. Monitoring then re-checks continuously and raises drift events.

### Why SPF merging must be automated

Of every record in the email set, SPF is the one where manual instructions are structurally unsafe. The customer's domain almost always has an existing `v=spf1` record. The correct operation is a merge:

```dns
; before
example.com.  TXT  "v=spf1 include:_spf.mailsuite.example -all"

; after (correct: one record, mechanism inserted)
example.com.  TXT  "v=spf1 include:_spf.mailsuite.example include:spf.yourplatform.example -all"

; after (what manual instructions produce far too often: permanent error)
example.com.  TXT  "v=spf1 include:_spf.mailsuite.example -all"
example.com.  TXT  "v=spf1 include:spf.yourplatform.example ~all"
```

An automated flow reads the existing record, inserts the mechanism, respects the customer's existing `all` qualifier, and can check the lookup budget before writing (see [the failure catalog](03-deliverability-and-dns.md)). A help article cannot do any of that.

## The standards behind one-click, and what sits on top

Part of the provider authorization landscape is the Domain Connect protocol, an open standard from the Domain Connect Association. Under the protocol, a service publishes a **template** of DNS records, each participating DNS provider vets and hosts that template in advance, and at connect time the user approves it on a consent screen the DNS provider controls. Two properties make this a good trust model, as the Association's CC0 knowledge base describes: the provider never trusts the service at runtime (only pre-vetted templates can run, scoped exactly to their records), and the user consents at their own DNS provider, seeing the exact changes. The knowledge base's template catalog even names email as a core use case, with templates for email hosting, for outbound authentication (SPF merging, DKIM keys, return path CNAMEs), and for DMARC management.

The protocol alone does not cover every provider, which is why Custom Domain's [one-click provider authorization](https://customdomain.ai/one-click-dns-setup) combines protocol support with direct provider API integrations: 63 DNS and registrar providers covered, more than 25 fully auto-configured, with the same consent-first shape throughout.

## The fallback ladder

**API token.** Where a provider exposes scoped DNS tokens, the customer pastes one and the platform manages records directly, including later changes like key rotation. Best practice: request the narrowest scope the provider offers, ideally zone-level edit on the single domain.

**Guided manual with automatic verification.** The floor, and it should still be high: provider-aware copy-paste values (host field format matched to the detected console), one record per step, and background polling of authoritative nameservers that turns each record green as it lands. No verify button to mash, no 48 hour warnings. The customer-facing version of this flow is what the [setup guide](https://customdomain.ai/guides/how-to-set-up-a-custom-domain) walks through.

Route customers down this ladder automatically. They should never be asked "which method would you like," only shown the fastest one their provider supports.

## Integration surfaces

**The connect widget.** An embeddable modal that runs the entire ladder: detection, method choice, authorization or guided steps, verification, and status back to your app. This is the shortest path from "we have a DNS instructions page" to "domains connect themselves," and new provider integrations arrive without code changes on your side. Details: [connect domain widget](https://customdomain.ai/connect-domain-widget).

**The REST API.** For platforms that own their onboarding UX end to end. The API covers connections, DNS record management, verification, TLS for tracking and hosted domains, monitoring, and webhooks, plus registrar search and purchase for customers who arrive without a domain at all. A minimal flow looks like: create a connection for the domain, receive the required record set and best available method, direct the customer through it, then consume webhook events as verification completes and, later, if drift occurs. Exact schemas and endpoints are in the [developer docs](https://app.customdomain.ai/docs); see also the [custom domain API overview](https://customdomain.ai/custom-domain-api).

**Webhooks for the email lifecycle.** The events that matter to an email platform: domain verified (enable sending), record drift detected (warn, then pause sending before reputation damage), TLS issued or renewed (tracking domain healthy). Wiring "authentication collapsed" to "pause this customer's sends" is one of the highest-leverage deliverability protections you can ship.

**The MCP server, for agent-driven setup.** Customers increasingly delegate configuration to AI assistants. Custom Domain hosts an MCP server (streamable HTTP at `mcp.customdomain.ai/mcp`, OAuth client credentials) exposing the connect flow as tools an agent can call: check a domain, start a connection, fetch required records, poll verification. See the [MCP server page](https://customdomain.ai/mcp-server), [Custom Domain for AI agents](https://customdomain.ai/for/ai-agents), and the [customdomain-mcp repository](https://github.com/ever-just/customdomain-mcp).

## What changes when you automate

Returning to the funnel metrics from [onboarding sending domains](02-onboarding-sending-domains.md): the failure table (doubled hostnames, second SPF records, pasted quotes, proxied CNAMEs) applies only to manual transcription. Every domain that goes through provider authorization skips it entirely, completes in well under a minute, and stays monitored afterward. The manual path remains for the long tail, but it becomes the exception with a safety net rather than the default experience.

Custom Domain provides this as a managed service: control plane plus a reverse-proxy edge for TLS termination where hosted HTTPS is needed, strict multi-tenant isolation, and pricing that starts at $0. Start at [customdomain.ai](https://customdomain.ai) or go straight to [signup](https://app.customdomain.ai/signup).

## Attribution

Sections of this document describing the Domain Connect protocol draw on the Domain Connect Association's knowledge base, published under CC0 1.0 (public domain). The Domain Connect protocol is a third-party open standard and is not a Custom Domain product.
