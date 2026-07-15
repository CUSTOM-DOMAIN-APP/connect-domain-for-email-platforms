# Deliverability and DNS: Alignment, Common Failures, and Propagation

Authentication records do not act alone. Deliverability outcomes come from how SPF, DKIM, and DMARC interact, and most "our email suddenly went to spam" incidents trace back to a DNS detail. This doc covers DMARC alignment, the recurring DNS failure modes, and what propagation actually is. Record syntax lives in [email authentication records](01-email-authentication-records.md); onboarding UX lives in [onboarding sending domains](02-onboarding-sending-domains.md).

## Alignment: the rule that changes everything

SPF and DKIM each authenticate a domain, but not necessarily the domain the recipient sees. DMARC's contribution is the alignment requirement: an authentication pass only counts if the authenticated domain matches the **From header** domain.

- **SPF authenticates the MAIL FROM domain** (the envelope sender, where bounces go). Platforms bounce to their own domain by default, so out of the box SPF passes but authenticates `yourplatform.example`, not `example.com`. No alignment, no DMARC credit.
- **DKIM authenticates the `d=` domain in the signature.** A platform signing with `d=yourplatform.example` has valid DKIM that does nothing for the customer's DMARC. The signature must use `d=example.com`, with keys published under the customer's domain, which is exactly what the delegated `_domainkey` CNAMEs enable.

Under relaxed alignment (the default) an organizational domain match suffices, so `bounce.example.com` aligns with a From address at `example.com`. That is the entire reason custom return path domains exist.

The practical target for every sending domain is **both** aligned SPF and aligned DKIM. Mail gets forwarded, and forwarding changes the envelope sender, which breaks SPF alignment downstream. Mailing list software rewrites bodies and breaks DKIM. With both in place, either one surviving is enough for DMARC to pass.

## The failure catalog

These are the DNS-rooted failures that generate deliverability incidents, roughly ordered by damage.

| Failure | Mechanism | Blast radius |
|---|---|---|
| Two SPF records | RFC 7208 makes multiple `v=spf1` records a permanent error | All the domain's mail can fail SPF, not just platform mail |
| SPF over 10 lookups | Each `include`/`a`/`mx`/`exists`/`redirect` costs a lookup; 11+ is a permanent error | Same as above, and it often breaks when some *other* vendor is added |
| Hardcoded DKIM keys | Customer pasted a TXT key instead of the delegated CNAME | Platform cannot rotate keys; old key eventually retired, signatures fail |
| Flattened SPF gone stale | A "flattening" tool copied your IPs literally instead of keeping the `include` | Your infrastructure changes, their record silently stops matching |
| Proxied CNAME | DNS host's HTTP proxy toggle on a DKIM or tracking CNAME answers with proxy IPs | DKIM lookups fail; tracking links hit the wrong endpoint |
| Missing return path record | Bounce subdomain never delegated | SPF never aligns; DMARC rides on DKIM alone, fragile under forwarding |
| DMARC policy surprise | Org domain publishes `p=reject` (or a strict `sp=`) that the new sending subdomain inherits | Unaligned mail from the not-yet-finished setup gets rejected outright |
| Deleted records (drift) | Migration, cleanup, or agency handoff removes "unrecognized" records | Working domain degrades weeks or months after verification |

Two patterns are worth calling out. First, the worst failures are **shared blast radius** failures: a botched SPF merge harms the customer's corporate mail, and they will experience that as your platform breaking their email. Second, several failures are **time bombs**: hardcoded keys, flattened SPF, and drift all verify green today and fail later. Point-in-time verification cannot catch them; continuous monitoring can, which is why re-checking connected domains and firing drift webhooks is part of the managed service model at [Custom Domain](https://customdomain.ai).

## What propagation actually means

"Allow up to 48 hours for propagation" is the most repeated and least accurate sentence in DNS support. What actually happens:

1. **Authoritative update.** The customer's DNS host writes the record to its authoritative nameservers. On modern hosted DNS this takes seconds to a couple of minutes.
2. **Resolver caching.** Recursive resolvers that previously looked up the name keep their cached answer until its TTL expires. A record with a 3600 second TTL can look "missing" on a given resolver for up to an hour after it exists, but only on resolvers that had already cached it.
3. **Negative caching.** The sharp edge (RFC 2308): if a resolver asked for the name **before it existed**, the NXDOMAIN answer is cached too, for a duration taken from the zone's SOA record. Checking for a record too early through a recursive resolver can therefore *delay* the appearance of success on that resolver. This is why naive "click verify" buttons that query public resolvers produce the maddening fixed-it-but-still-failing state.

Operational consequences for a platform:

- **Verify against the authoritative nameservers directly.** Look up the domain's NS set and query those servers. You see truth in seconds and skip both caches entirely.
- **Never tell customers 48 hours.** Tell them the check runs automatically and usually confirms within minutes; long tails are almost always a wrong record, not slow propagation.
- **Long TTLs cut the other way on removal.** When rotating or removing records, the old value lives on in caches for a full TTL after the change. Plan key rotations with overlap (two DKIM selectors) rather than in-place swaps.

## Monitoring: verification is not a moment

A sending domain's DNS is a living dependency. Treat it like uptime:

- Re-resolve every required record on a schedule (hourly is a reasonable default) against authoritative servers.
- Alert on **drift**: a record that disappears, changes value, or gets a proxy toggled on.
- Distinguish **degraded** (return path missing, DKIM still aligned) from **down** (DKIM CNAMEs gone), and message customers accordingly.
- Emit events your product can act on. Pausing sends when authentication collapses protects the customer's domain reputation better than sending unauthenticated mail into a `p=reject` policy.

This monitoring loop, alongside record management and verification, is exposed via API and webhooks in the [Custom Domain developer docs](https://app.customdomain.ai/docs). How records get written correctly in the first place, including SPF merging, is covered next in [automating domain setup for email](04-automating-domain-setup-for-email.md).
