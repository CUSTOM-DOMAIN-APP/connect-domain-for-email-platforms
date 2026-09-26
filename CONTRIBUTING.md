# Contributing

This repository is a field guide, not a product. Corrections are welcome, and corrections to numbers are welcome most of all.

## The one rule that matters

Every number and every capability claim here has a source, and the source is checkable without asking us. Before you change one, read the source rather than the surrounding prose:

| Claim | Source |
|---|---|
| Provider counts (63 catalogued, 25 automatic, 38 guided manual) | `GET https://api.customdomain.ai/v1/providers/census` |
| Plan names and prices | `GET https://api.customdomain.ai/v1/plans`; [customdomain.ai/pricing](https://customdomain.ai/pricing) is the human-readable page, and where the two disagree the endpoint wins |
| API request and response shapes | [api.customdomain.ai/v1/openapi.json](https://api.customdomain.ai/v1/openapi.json) and [docs.customdomain.ai](https://docs.customdomain.ai/docs) |
| Connection statuses and timeouts | [docs.customdomain.ai/docs/concepts/connections](https://docs.customdomain.ai/docs/concepts/connections) |
| Webhook event names and delivery gating | [docs.customdomain.ai/docs/webhooks/overview](https://docs.customdomain.ai/docs/webhooks/overview) |
| MCP tool count and behavior | [CUSTOM-DOMAIN-APP/customdomain-mcp](https://github.com/CUSTOM-DOMAIN-APP/customdomain-mcp) |
| Domain Connect templates | [Domain-Connect/Templates](https://github.com/Domain-Connect/Templates) |
| Manual DNS failure statistics | [Domain-Connect/knowledge-base](https://github.com/Domain-Connect/knowledge-base) (CC0 1.0) |
| Competitor pricing | the competitor's own pricing page, with the date you read it |

If a claim in this repository cannot be traced to one of those, it is a bug. Open an issue with the file and line.

## House style

- Plain language. Short declarative sentences. An engineer explaining a tradeoff to another engineer.
- No em dashes or en dashes anywhere. American English.
- Real DNS, TLS, and email facts only. No invented numbers, no rounded-up numbers, no "more than N" where N is exact.
- The product name is **CustomDomain™**: one word, capital C and D, with the ™. The generic concept stays lowercase ("custom domain"). Never rename a machine-readable identifier: Domain Connect `providerId` and `serviceId` values, package names, and URLs stay exactly as they are.
- No hype register. If a sentence would survive being read aloud in a postmortem, it is fine.
- Where the guide makes a comparison, it states at least one thing the comparison does not favor us on. Keep that property.

## Practicalities

Markdown only. Keep files under 300 KB. Open a pull request against `main`; a one line description of what you checked is more useful than a long one about what you wrote.

Questions: connect@customdomain.ai
