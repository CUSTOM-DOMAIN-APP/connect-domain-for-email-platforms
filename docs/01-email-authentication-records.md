# Email Authentication DNS Records: SPF, DKIM, DMARC, Return Path, and BIMI

This is the record-by-record reference for what an email platform asks its customers to publish, why each record exists, and exactly what it should look like. It pairs with the [repo README](../README.md), which covers the connection flow around these records, and with [deliverability and DNS](03-deliverability-and-dns.md), which covers how they interact under DMARC.

Throughout, `example.com` is the customer's domain and `yourplatform.example` stands in for your platform's infrastructure.

## SPF: who may send for this domain

SPF (RFC 7208) is a TXT record listing the servers authorized to send mail for a domain. Receivers check it against the domain in the envelope sender (the SMTP MAIL FROM address, also called the return path), not the From header the recipient sees.

```dns
example.com.  TXT  "v=spf1 include:spf.yourplatform.example ~all"
```

- `v=spf1` marks the record. A name may carry **only one** SPF record. A second `v=spf1` TXT on the same name is a permanent error and authentication fails for everything.
- `include:` pulls in your platform's authorized senders, so you can change IPs without customers touching DNS again.
- `~all` (softfail) versus `-all` (fail) sets the default for unlisted senders. Under DMARC either is fine, since DMARC cares about pass or not-pass with alignment.
- **The 10 lookup limit:** `include`, `a`, `mx`, `ptr`, `exists`, and `redirect` each cost a DNS lookup, and RFC 7208 caps the total at 10. Customers who already include a mail suite, a CRM, and a helpdesk can be at the limit before your `include` arrives. Going over is a permanent error.

Because customers usually already have an SPF record, your instructions (or your automation) must **merge** the new mechanism into the existing record, never create a second one. This single rule prevents one of the most damaging setup mistakes in email.

The Domain Connect protocol has a record type for exactly this, `SPFM`, which extends an existing SPF record instead of writing a second one. CustomDomain's published template [`customdomain.ai.email-spf.json`](https://github.com/Domain-Connect/Templates) uses it, so the merge is performed by the DNS provider at apply time rather than by the customer in a text field.

## DKIM: cryptographic proof the message was not altered

DKIM (RFC 6376) signs each message with a private key you hold; receivers fetch the matching public key from DNS at `<selector>._domainkey.<domain>`. The signature proves the message body and key headers were not modified in transit and binds the message to the signing domain, the `d=` value in the signature.

Platforms should delegate selectors with CNAMEs rather than asking customers to paste raw keys:

```dns
s1._domainkey.example.com.  CNAME  s1.dkim.yourplatform.example.
s2._domainkey.example.com.  CNAME  s2.dkim.yourplatform.example.
```

The CNAME targets resolve to TXT records you host:

```dns
s1.dkim.yourplatform.example.  TXT  "v=DKIM1; k=rsa; p=MIIBIjANBgkq..."
```

Why CNAME delegation matters:

- **Key rotation.** You rotate keys on your side on your schedule. Customers who pasted a raw TXT key freeze you to that key forever, or force a re-onboarding campaign.
- **Two selectors** let you rotate with zero downtime: sign with s1 while s2 holds the next key, then swap.
- **Key length.** Publish 2048 bit RSA keys. 1024 bit keys are still verified but are below current best practice.

One operational warning: some DNS hosts offer an HTTP proxy or acceleration toggle on CNAME records. A proxied CNAME answers with the provider's own addresses instead of behaving as a DNS alias, which silently breaks DKIM key lookup. The record must be plain DNS.

## DMARC: policy and reporting on top of SPF and DKIM

DMARC (RFC 7489) lives at `_dmarc.<domain>` and does two jobs: it tells receivers what to do with mail that fails authentication, and it sends the domain owner reports about who is sending as them.

```dns
_dmarc.example.com.  TXT  "v=DMARC1; p=none; rua=mailto:dmarc@example.com; adkim=r; aspf=r"
```

- `p=` is the policy for failing mail: `none` (report only), `quarantine` (spam folder), `reject`. New sending domains should start at `none`, review reports, then tighten.
- `rua=` is where aggregate reports go. If reports go to a different domain than the one publishing the record, the receiving domain must publish an external destination verification record, so warn customers who point `rua` at a third party mailbox.
- `adkim` and `aspf` set alignment mode, relaxed (`r`, the default, organizational domain match) or strict (`s`, exact match).
- `sp=` optionally sets a separate policy for subdomains.

Critically, DMARC passes only when SPF or DKIM passes **and** the passing identifier aligns with the From header domain. That interaction, and why it forces the next record into existence, is the subject of [deliverability and DNS](03-deliverability-and-dns.md).

## Custom return path: making SPF align

By default, platforms send with a bounce address on their own domain, something like `bounces@mail.yourplatform.example`, so they can process bounces. SPF then authenticates `yourplatform.example`, which does not align with the customer's From domain, so SPF contributes nothing under DMARC.

A custom return path (custom MAIL FROM) fixes this by putting the bounce domain on the customer's zone:

```dns
bounce.example.com.  CNAME  bounces.yourplatform.example.
```

Some platforms use an MX plus TXT pattern instead of a CNAME, which achieves the same thing while receiving bounces directly on the subdomain:

```dns
bounce.example.com.  MX   10 feedback.yourplatform.example.
bounce.example.com.  TXT  "v=spf1 include:spf.yourplatform.example -all"
```

Either way, mail now goes out with MAIL FROM `...@bounce.example.com`. Under relaxed alignment, `bounce.example.com` and `example.com` share an organizational domain, so an SPF pass aligns and DMARC is satisfied even if a forwarder breaks the DKIM signature's body hash. Redundancy is the point: with both aligned SPF and aligned DKIM, one can fail in transit and DMARC still passes.

## BIMI: the logo, strictly gated

BIMI displays a brand logo next to authenticated mail in supporting inboxes. It is a TXT record:

```dns
default._bimi.example.com.  TXT  "v=BIMI1; l=https://example.com/brand/logo.svg; a=https://example.com/brand/vmc.pem"
```

- `l=` points to the logo, an SVG in the constrained SVG Tiny Portable/Secure profile.
- `a=` points to a Verified Mark Certificate (VMC), which several major mailbox providers require before showing the logo. VMCs are issued by a small number of certification authorities and generally require a registered trademark.
- **The gate:** BIMI requires DMARC at enforcement, `p=quarantine` or `p=reject`, applied at full volume. A domain sitting at `p=none` gets no logo no matter what else is published.

For an email platform, BIMI is a useful carrot: it gives customers a visible reward for finishing the DMARC journey you want them on anyway.

## Quick reference

| Record | Name | Type | Failure if wrong |
|---|---|---|---|
| SPF | `example.com` | TXT | Duplicate or over-limit record fails all the domain's mail |
| DKIM | `s1._domainkey`, `s2._domainkey` | CNAME | Broken key lookup, signatures unverifiable |
| DMARC | `_dmarc` | TXT | No policy, no reports, no BIMI eligibility |
| Return path | `bounce` | CNAME or MX + TXT | SPF never aligns, DMARC leans on DKIM alone |
| Tracking | `click` | CNAME (+ TLS) | Broken or warning-laden links in every email |
| BIMI | `default._bimi` | TXT | No logo shown |

All of these records can be written automatically with the customer's consent instead of being copy-pasted: see [automating domain setup for email](04-automating-domain-setup-for-email.md) and the [CustomDomain docs](https://docs.customdomain.ai/docs).
