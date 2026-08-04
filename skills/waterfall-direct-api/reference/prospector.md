# Prospector

Find contacts at a target company by title/location. Launcher/finder shape: `POST` to start the
job, `GET` with `job_id` to retrieve results.

## `POST /v1/prospector` — launcher

**Required:** `domain` (string, 4–500 chars — a bare domain or full URL; Waterfall extracts the
company domain), plus one of:
- `title_filter` — a single boolean title expression (see [SKILL.md](../SKILL.md) filter syntax
  notes on docs.waterfall.io), **or**
- `title_filters` — up to 5 named filters tried in order, each `{ "name": ..., "filter": ... }`

**Optional:**
- `company_name`, `linkedin` — alternative company identifiers alongside `domain`
- `location_name`, `location_country`, `location_countries` (up to 50) — contact location filters
- `excluded_names` — up to 200 full names to exclude from results
- `limit` — integer 1–500, default 10, max contacts returned
- `include_phones` — boolean, default `false`; runs advanced phone enrichment (additional charges
  may apply when numbers are found) — requires phone enrichment to be enabled on the account, or
  you'll get `PERMISSION_INCLUDE_PHONES_REQUIRES_PHONE_ENRICHMENT`
- `verified_only` — boolean, default `true`; `false` also returns catch-all (still-verified, just
  lower-confidence) emails
- `webhook_url`, `custom_fields` — callback delivery and pass-through metadata

Response: `{ "job_id": "<uuid>", "start_date": "<iso8601>" }`

```json
{"domain": "waterfall.io", "title_filter": "founder OR ceo", "limit": 10, "include_phones": false}
```

## `GET /v1/prospector?job_id=<uuid>` — finder

Response echoes `status`, `start_date`, `input.task`, and once `status: "SUCCEEDED"`:

```json
{
  "status": "SUCCEEDED",
  "start_date": "2024-06-18T13:53:21.181000+00:00",
  "stop_date": "2024-06-18T13:53:26.014000+00:00",
  "input": { "task": { "domain": "acme.com", "limit": 10 } },
  "output": {
    "company": {
      "domain": "acme.com", "company_name": "Acme Corp",
      "linkedin_url": "https://www.linkedin.com/company/acme-corp/",
      "size": "501-1000", "linkedin_industry": "Software Development",
      "country": "United States"
    },
    "persons": [{
      "first_name": "Jane", "last_name": "Doe",
      "linkedin_url": "https://www.linkedin.com/in/jane-doe-abc123/",
      "title": "Head of Sales", "seniority": "Director", "department": "Sales",
      "professional_email": "jane.doe@acme.com",
      "email_verified": true, "email_confidence": "high", "email_verified_status": "safe",
      "phone_numbers": []
    }],
    "usage": { "total_usd": 0.10, "balance_remaining_usd": 99.90, "persons_count": 1 }
  }
}
```

`persons` is a list — may be empty if no matches, which is a normal `SUCCEEDED` result, not an
error. `phone_numbers` is only populated when `include_phones: true` was set on the launcher and
a number was found.

## Errors specific to this endpoint

`VALIDATION_MISSING_TITLE_FILTERS` and the `VALIDATION_BAD_TITLE_FILTER*` family (see
[errors.md](errors.md)) come from malformed `title_filter`/`title_filters` syntax — check
parenthesis balance and operator spacing (`AND`/`OR`/`NOT`, uppercase, space-separated) before
retrying with a different filter.
