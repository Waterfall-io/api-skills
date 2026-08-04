# Waterfall API skills

Agent skills for working with the [Waterfall API](https://docs.waterfall.io/) — B2B contact and
company data (prospecting, enrichment, search, job-change detection, company reveal, email
verification).

This is a multi-skill repo. Each skill lives in its own subfolder and installs independently.

| Skill | Purpose |
| --- | --- |
| [`waterfall-direct-api`](skills/waterfall-direct-api/SKILL.md) | Call the Waterfall API directly over raw HTTP — auth, request/response shapes, pagination, errors. |
| `waterfall-integration` | *(planned)* Generate a client-owned Waterfall API integration in your stack. |

## Install

Using [`npx skills`](https://github.com/vercel-labs/skills):

```bash
npx skills add waterfall/api-skills --skill waterfall-direct-api
```

Multi-skill repos require the explicit `--skill` flag — the unqualified
`npx skills add waterfall/api-skills` isn't guaranteed to install a specific skill by the CLI's own
docs. You can also install directly from a `.../tree/main/skills/<name>` URL.

## Source of truth

Content is derived from [docs.waterfall.io](https://docs.waterfall.io/) and the public OpenAPI
spec, and is kept in sync with the real API contract as it evolves. If a skill's guidance
conflicts with the live docs, the docs win — please open an issue.

## License

Apache License 2.0 — see [LICENSE](LICENSE). "Waterfall" and any associated logos are not covered
by this license grant; see [TRADEMARK.md](TRADEMARK.md).

Copyright 2026 Waterfall, Inc.
