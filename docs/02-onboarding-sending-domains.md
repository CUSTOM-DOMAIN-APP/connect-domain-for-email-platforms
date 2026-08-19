# Onboarding Sending Domains: Fixing the Customer Setup Funnel

The moment a customer of an email platform meets DNS is the moment your activation curve bends. This doc looks at sending domain onboarding as a funnel problem: where customers drop, why the tickets look the way they do, and what verification UX that actually works looks like. It builds on the record reference in [email authentication records](01-email-authentication-records.md) and the flow overview in the [repo README](../README.md).

## The wall after signup

A new customer's journey is smooth right up to the point where value should appear: import contacts, design a template, write a campaign, and then a screen that says **"Authenticate your domain to start sending."** Behind that screen are four to six DNS records, a console the customer may not have logged into for years, and possibly a domain managed by an agency or an IT contractor who is not in the room.

The Domain Connect project's knowledge base ([github.com/Domain-Connect/knowledge-base](https://github.com/Domain-Connect/knowledge-base), published under CC0 1.0) quantifies the general pattern using Microsoft 365: 7 to 15 records, 16 help sites maintained by Microsoft with 10 of them registrar specific, and "approximately 50% of users who attempt manual DNS configuration fail and abandon the process" (`Knowledge Base/01_Problem_and_Context.md`). There is no reason to believe email platform customers do better. If anything the stakes are higher, because a partially completed email setup does not fail loudly. It sends, lands in spam, and quietly convinces the customer your platform "has bad deliverability."

Every customer who stalls here is a customer who never sends a campaign, never sees an open rate, and never develops a reason to pay.

## Know who is holding the mouse

The person completing this step is rarely a DNS administrator:

- A **marketer** who owns the email program but has never opened the registrar account.
- A **founder** who bought the domain years ago at whichever registrar was cheapest.
- An **agency** managing this setup for the fifth client this month, each on a different provider (the pattern [agency and white label platforms](https://customdomain.ai/for/agencies-white-label) are built around).
- An **IT contact** reached by forwarded email, working from a screenshot of your instructions.

Design for the least technical of these and the others get faster too.

## Why the tickets happen: an anatomy of failed setups

DNS setup tickets are not random. The same failure modes recur, and most are induced by the gap between your generic instructions and a specific provider's console.

| Failure | What happened | What the customer saw |
|---|---|---|
| Doubled hostname | Console auto-appends the zone, customer pasted the FQDN, record landed at `s1._domainkey.example.com.example.com` | "I added it hours ago and it still says pending" |
| Second SPF record | Instructions said "add a TXT record," customer already had one | All mail failing SPF, including their regular business mail |
| Quotes pasted literally | TXT value entered with surrounding quotes as characters | Verification fails on an exact-match check |
| Proxied CNAME | DNS host's acceleration toggle left on for a DKIM or tracking CNAME | DKIM lookups fail; tracking links break |
| Wrong provider entirely | Customer edited DNS at the registrar, but nameservers point to a different DNS host | Nothing they change has any effect |
| Trailing dot confusion | Provider requires (or forbids) the trailing dot on CNAME targets | Record saved, resolves wrong |
| Stale verify button | Customer fixed the record, your check ran before caches expired | "Your tool is broken," then an abandoned session |

Each row is a ticket that takes half an hour, needs screenshots, and teaches the customer that email is hard on your platform. Multiply by the 63 provider consoles in the [live census](https://api.customdomain.ai/v1/providers/census), each with its own vocabulary, and the support cost compounds.

## Verification UX that works

The difference between a 50% completion funnel and a good one is mostly UX mechanics, not customer education.

**Detect the provider before showing anything.** A nameserver lookup tells you where the domain's DNS actually lives. Lead with that: "Your DNS is hosted at [provider]. Here is the fastest way to connect." This kills the wrong-console failure mode outright, and it is the entry point for [one-click provider authorization](https://customdomain.ai/one-click-dns-setup) where available.

**Prefer flows where the customer never types.** Provider authorization or a scoped API token writes the records programmatically. The whole failure table above simply does not apply. See [automating domain setup for email](04-automating-domain-setup-for-email.md).

**When manual is unavoidable, make it provider aware.** Show host names in the exact form the detected console expects, relative or fully qualified. Show values without decorative quotes. One record per card with its own copy button, not a table to transcribe.

**Verify continuously, not on click.** Poll the domain's authoritative nameservers in the background and flip each record to green as it lands. A "Verify" button that can fail teaches customers to distrust the process; a checklist that fills itself in teaches them it is working. Querying authoritative servers directly also avoids most cache lag (see [deliverability and DNS](03-deliverability-and-dns.md)).

**Give partial credit.** "3 of 5 records verified, waiting on SPF and DMARC" is actionable. A single red "Not verified" state hides the customer's progress and invites abandonment.

**Show diffs, not verdicts.** When a check fails, show what was found versus what was expected: "We found a TXT record, but the value starts with a quote character." That sentence closes the ticket before it opens.

**Keep watching after success.** Records drift: migrations, cleanups, agency handoffs. Re-check on a schedule and notify on drift, ideally before mailbox providers react. A domain that silently loses its DKIM CNAMEs during a website replatform is a deliverability incident with your platform's name on it.

## The numbers to watch

Instrument the funnel like any other activation step:

- **Connection completion rate**: domains verified divided by domains started.
- **Time to verified**: from first seeing the connect screen to all records green. With provider authorization this is typically under a minute; with manual flows, hours to days.
- **Tickets per 100 connections**: the honest measure of instruction quality.
- **Drop-off point**: which record, and which provider, kills the most sessions. This tells you exactly which provider integration or instruction set to fix next.

Platforms that move from static instructions to automated connection see the entire failure table above collapse into the automated path. How that automation works mechanically, from provider detection to SPF merging to webhooks, is the subject of [the next deep dive](04-automating-domain-setup-for-email.md), and the managed version of it is what [CustomDomain](https://customdomain.ai) provides.
