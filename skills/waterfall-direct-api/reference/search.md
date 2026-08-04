# Search contact / Search company

Unlike Prospector and enrichment, these two return the full result **directly on the `POST`** —
you don't have to poll. A `GET` finder also exists for re-fetching a completed job later by
`job_id`, but it's optional, not required for a normal call.

## Search contact — `POST /v1/search/contact`

Three mutually exclusive search modes:
1. **By contact:** `contact_linkedin`
2. **By company:** `domain`, `company_linkedin`, or `company_name`
3. **By company-set filters:** one of `company_location_countries`, `company_industries`, or
   `company_employee_ranges`, **plus** one of `experience_start_date` or
   `experience_start_date_new_hire` (date string), **plus** one of `title_filters` (boolean
   expressions, same shape as Prospector) or `title_lists` (named lists of exact titles) — all
   three are required together in this mode, not just the company-set filter alone.

Other optional filters (any mode): `excluded_names` / `included_names` (up to 200/100 full
names), `departments`, `seniorities`.

**Pagination:** `page_number` / `page_size` in the request body — no cursor, no `limit` field.
Same mechanism as Search Company (see below), unlike Prospector which uses `limit` instead of
pagination. Optional `custom_fields`.

```json
{
  "domain": "waterfall.io",
  "title_filters": [{"name": "founders", "filter": "founder OR co-founder OR ceo"}],
  "page_number": 1,
  "page_size": 10
}
```

Response `output.persons` — same person shape as [prospector.md](prospector.md)'s output, plus
`personal_email` and `mobile_phone` (both `null` unless included via account permissions). Fields
are `null` rather than omitted when unavailable — this is a normal result shape, not a partial
failure.

`GET /v1/search/contact?job_id=<uuid>` re-fetches the same response by job ID.

## Search company — `POST /v1/search/company` (Preview)

At least one non-empty list among `industries`, `location_countries`, or `sizes` is required.
Pagination: `page_number` / `page_size` (no cursor, no `next_page` link — request the next
`page_number` explicitly). `custom_fields` optional.

```json
{
  "industries": ["software development"],
  "location_countries": ["United States"],
  "sizes": ["201-500"],
  "page_number": 1,
  "page_size": 20
}
```

Response `output.companies[]` fields: `id`, `domain`, `name`, `website`, `linkedin_id`,
`linkedin_url`, `description`, `logo_url`, `size`, `employees_count`, `industry`, `type`,
`founded`, `address`, `country`, `linkedin_followers`, `crunchbase_url`, `funding_details`,
`recent_job_posting_count`, `technologies`.

`GET /v1/search/company?job_id=<uuid>` re-fetches the same response by job ID.

Both endpoints share the [errors.md](errors.md) table; the most common failure is
`VALIDATION_BAD_REQUEST` when none of the required filter lists are provided.
