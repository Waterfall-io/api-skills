# Waterfall API reference (language-agnostic)

Ground truth for generating a client integration. Read this once regardless of target language;
[python.md](python.md) / [javascript.md](javascript.md) build idiomatic code on top of these facts
— they don't restate them.

Source of truth: [docs.waterfall.io](https://docs.waterfall.io/) and the OpenAPI 3.1 spec at
[waterfall-io-public.s3.us-east-1.amazonaws.com/api_specs.yaml](https://waterfall-io-public.s3.us-east-1.amazonaws.com/api_specs.yaml)
(linked from [docs.waterfall.io/v1/openapi-spec](https://docs.waterfall.io/v1/openapi-spec)).
Regenerate against the live spec rather than trusting this file if it's been a while — don't let
generated code silently drift from the real contract.

## Auth & base URL

- Base URL: `https://api.waterfall.io`
- Every request needs an `x-api-key` header. No OAuth, no bearer token, no query-param key.
- Wrap key retrieval in whatever secret-management convention the client's codebase already uses
  (env var, secrets manager) — never hardcode a key in generated code.

## Two response shapes — this determines your client's structure

**Async (launcher/finder):** Prospector and the three Enrichment endpoints (`contact`, `phone`,
`company`). `POST` returns only `{job_id, start_date}`. You must `GET ...?job_id=<uuid>` and poll
until `status` leaves `RUNNING`. Client code for these needs a poll loop with backoff — don't
generate a single-request function for these four.

**Sync:** every other job-based endpoint (Search Contact, Search Company, Job Change, Company
Reveal, Company Titles, Email Verification) returns the full result directly on `POST` — no poll
loop needed. A `GET` finder also exists on most of these for re-fetching by `job_id` later, but
it's optional, not part of the request/response cycle.

Every job response — sync or polled — has a `status` field: `RUNNING`, `SUCCEEDED`, `FAILED`,
`TIMED_OUT`, `ABORTED`. Only `SUCCEEDED` means `output` is populated. Generated code must check
this field, not just the HTTP status code — a 200 with `status: "FAILED"` is not a success.

Account usage (`GET /v2/account`) and API key management (`/v1/api-keys`) aren't job-based — plain
request/response, no `status`/`job_id` envelope.

## Pagination

`/v1/search/company` and `/v1/search/contact` paginate via `page_number`/`page_size` in the
request body — no cursor, no `next_page` link. Generated pagination helpers should loop
incrementing `page_number` until a page returns fewer than `page_size` results (or an explicit
`has_more_pages: false` for Company Titles, which uses `page_number` alone). Prospector uses a
`limit` cap instead — it's not paginated at all, don't build a pagination loop for it.

## Rate limits & retries

Prospector, Search, Enrichment, Job Change, and Verify Email: 50 requests/minute. On
`429`/`RATE_LIMIT_RATE_LIMIT_EXCEEDED`, the response includes a `Retry-After` header (seconds) —
generated retry logic must read and respect this rather than using a fixed/guessed backoff.
`/v2/account` has no rate limit. `500`/`INTERNAL_*` errors are safe to retry with backoff;
`400`/`401`/`403`/`404` are not (the request itself needs fixing first).

## Error model

Every error response: `{status, message, category, code}` (sometimes `error` with field detail).
Generated error handling should branch on `code` (stable, specific) or `category` (coarse), never
on parsing `message` text — it's for humans and can change wording.

| Status | category prefix | Meaning |
| --- | --- | --- |
| 400 | `VALIDATION_*`, `ROUTING_*` | Malformed request or wrong method/path — fix the request, don't retry |
| 401 | `AUTH_MISSING_API_KEY` | No `x-api-key` header sent |
| 402 | `QUOTA_ACCOUNT_OVER_QUOTA` | Account over quota — not a code bug |
| 403 | `AUTH_BAD_API_KEY`, `PERMISSION_*` | Bad key, or valid key lacking a required permission |
| 404 | `NOT_FOUND_*` | Unknown endpoint, or a `job_id`/API key that doesn't exist |
| 429 | `RATE_LIMIT_RATE_LIMIT_EXCEEDED` | Respect `Retry-After` |
| 500 | `INTERNAL_*` | Waterfall-side — safe to retry with backoff |

Full code list (namespace-specific ones like `VALIDATION_BAD_TITLE_FILTER*` only apply to
Prospector's title filters; `*_MASTER_API_KEY`/`PERMISSION_NEED_ADMIN_API_KEY` only apply to
`/v1/api-keys`) — treat any `code` not in generated error-mapping code as `category`-level
fallback rather than crashing on an unmapped enum value, since new codes can be added over time:
`AUTH_BAD_API_KEY`, `AUTH_MISSING_API_KEY`, `AUTH_NEED_MASTER_API_KEY`,
`PERMISSION_FEATURE_NOT_ENABLED`, `PERMISSION_INCLUDE_PHONES_REQUIRES_PHONE_ENRICHMENT`,
`PERMISSION_NEED_ADMIN_API_KEY`, `QUOTA_ACCOUNT_OVER_QUOTA`, `VALIDATION_BAD_DOMAIN*` (8 variants),
`VALIDATION_BAD_EMAIL*` (7 variants), `VALIDATION_BAD_JOB_ID`, `VALIDATION_BAD_NAME_INVALID`,
`VALIDATION_BAD_REQUEST`, `VALIDATION_BAD_TITLE_FILTER*` (6 variants),
`VALIDATION_CANNOT_EDIT_MASTER_API_KEY`, `VALIDATION_MISSING_BODY`,
`VALIDATION_MISSING_JOB_ID_PARAMETER`, `VALIDATION_MISSING_TITLE_FILTERS`,
`VALIDATION_SUBKEY_RATE_EXCEEDS_MASTER`, `NOT_FOUND_ACCOUNT_OR_KEY_NOT_FOUND`,
`NOT_FOUND_API_KEY_NOT_FOUND`, `NOT_FOUND_JOB_NOT_FOUND`, `NOT_FOUND_NO_MASTER_API_KEY_FOUND`,
`NOT_FOUND_NOT_FOUND`, `RATE_LIMIT_RATE_LIMIT_EXCEEDED`, `ROUTING_INVALID_PATH_OR_METHOD`,
`INTERNAL_FAILED_CREATE_API_KEY`, `INTERNAL_FAILED_EDIT_API_KEY`, `INTERNAL_UNCLASSIFIED_ERROR`.

## Endpoints

| Endpoint | Method(s) | Shape | Stage |
| --- | --- | --- | --- |
| Prospector | `POST`/`GET /v1/prospector` | async | GA |
| Contact enrichment | `POST`/`GET /v1/enrichment/contact` | async | GA |
| Phone enrichment | `POST`/`GET /v1/enrichment/phone` | async | GA |
| Company enrichment | `POST`/`GET /v1/enrichment/company` | async | GA |
| Search contact | `POST /v1/search/contact` | sync, paginated | GA |
| Search company | `POST /v1/search/company` | sync, paginated | Preview |
| Job change | `POST /v1/job/change` | sync | Preview |
| Company reveal | `POST /v1/company-reveal` | sync | Preview |
| Company titles | `POST /v1/company-titles` | sync, paginated | Preview |
| Email verification | `POST /v1/verify/email` | sync | GA |
| Account usage | `GET /v2/account` | sync | GA |
| API key management | `GET`/`POST`/`PUT /v1/api-keys` | sync, master-key only | GA |
| Webhook signing keys | `GET /.well-known/jwks.json` | sync, no auth | GA |

`Preview` endpoints have no SLA and behavior may still change — flag this if the client is
building something production-critical on Search Company, Job Change, Company Reveal, or Company
Titles.

### Prospector — `POST /v1/prospector` (launch), `GET ...?job_id=` (poll)

Request: `domain` (required) + one of `title_filter` (boolean expression string) or
`title_filters` (up to 5 named filters). Optional: `company_name`, `linkedin`, `location_name`,
`location_country`/`location_countries`, `excluded_names`, `limit` (1–500, default 10),
`include_phones` (bool, extra charges when found), `verified_only` (bool, default true),
`webhook_url`, `custom_fields`.

Response `output`: `company` (`id`, `domain`, `company_name`, `website`, `linkedin_id`,
`linkedin_url`, `linkedin_description`, `linkedin_logo_url`, `size`, `linkedin_size`,
`linkedin_industry`, `linkedin_type`, `linkedin_followers`, `linkedin_founded`,
`linkedin_employees_count`, `linkedin_address`, `country`, `generic_emails`), `persons[]` (`id`,
`first_name`, `last_name`, `linkedin_id`, `linkedin_url`, `location`, `country`, `company_id`,
`company_linkedin_id`, `company_name`, `company_domain`, `professional_email`, `phone_numbers`,
`title`, `seniority`, `department`, `experiences[]`, `email_verified`, `email_confidence`,
`email_verified_status`, `smtp_provider`, `mx_record`), `usage`. This is the complete field list —
build strict typed models off it directly rather than guessing at additional fields.

### Contact/phone/company enrichment — `POST`/`GET /v1/enrichment/{contact,phone,company}`

Contact/phone: one identifier strategy — `linkedin`, `email`, or `full_name`/`first_name`+
`last_name` + `domain`. Company: one of `domain`, `linkedin`, `name`. Optional `webhook_url`,
`custom_fields`; contact also takes `include_phones`.

Response `output.person` (contact/phone): same shape as Prospector's `persons[]` entries, plus
`mobile_phone` for phone enrichment. Response `output.company` (company): `id`, `domain`, `name`,
`website`, `linkedin_id`, `linkedin_url`, `linkedin_followers`, `description`, `logo_url`, `size`,
`employees_count`, `industry`, `type`, `founded`, `address`, `country`, `crunchbase_url`,
`funding_details`, `recent_job_posting_count`, `technologies`.

### Search contact — `POST /v1/search/contact`

One mode: `contact_linkedin`; OR one of `domain`/`company_linkedin`/`company_name`; OR a
company-set filter (one of `company_location_countries`/`company_industries`/
`company_employee_ranges`, **plus** one of `experience_start_date`/`experience_start_date_new_hire`,
**plus** one of `title_filters`/`title_lists`). Optional: `excluded_names`, `included_names`,
`departments`, `seniorities`, `page_number`, `page_size`, `custom_fields`.

Response `output.persons[]`: same shape as Prospector's, plus `personal_email`, `mobile_phone`.

### Search company — `POST /v1/search/company` (Preview)

At least one non-empty list among `industries`, `location_countries`, `sizes`. Plus `page_number`,
`page_size`, `custom_fields`.

Response `output.companies[]`: `id`, `domain`, `name`, `website`, `linkedin_id`, `linkedin_url`,
`description`, `logo_url`, `size`, `employees_count`, `industry`, `type`, `founded`, `address`,
`country`, `linkedin_followers`, `crunchbase_url`, `funding_details`, `recent_job_posting_count`,
`technologies`.

### Job change — `POST /v1/job/change` (Preview)

`company_domain` + one of `contact_linkedin`/`professional_email`/`personal_email`/
`contact_full_name`; OR `company_linkedin` + one of `contact_linkedin`/`professional_email`/
`personal_email`; OR `professional_email` alone (optionally + `contact_full_name`).

Response `output`: `job_change_status` (`left`/`moved`/`no_change`/`unknown`), `person` (redacted
contact profile, `{}` if unresolved).

### Company reveal — `POST /v1/company-reveal` (Preview)

Request: `{ip}` (IPv4), optional `webhook_url`, `custom_fields`.

Response `output`: `ip`, `type` (`company`/`education`/`government`/`isp`/`null` — `isp` means the
match is unreliable), `confidence_score` (`very_high`/`high`/`medium`/`low`/`null`), `ip_geo`
(`city`, `state`, `country`), `company` (same shape as company enrichment's, `{}` if unresolved).

### Company titles — `POST /v1/company-titles` (Preview)

Request: `domain` or `company_linkedin`, optional `page_number`, `custom_fields`. Requires the
feature enabled on the account (`PERMISSION_FEATURE_NOT_ENABLED` if not).

Response `output`: `titles` (list of strings), `has_more_pages` (bool — no `page_size`, fixed
server-side).

### Email verification — `POST /v1/verify/email`

Request: `{email}`, optional `custom_fields`. Genuinely synchronous, no `job_id`/polling at all.

Response `output.email`: `email`, `domain`, `email_status` (`valid`/`invalid`/`risky`/`unknown`),
`smtp_provider`, `mx_records`.

### Account usage — `GET /v2/account`

Optional query param `month` (`YYYY-MM`). Response: `key_usage`, `account_usage` (same shape,
key-scoped vs account-wide), `balance_remaining_usd`, and `price` (unit prices) **only** when
authenticated with a master key — its absence for regular/sub keys isn't an error.

### API key management — `/v1/api-keys` (master key only)

`GET` lists keys. `POST` creates a sub-key (`notes` required, `per_interval` optional, capped at
the master key's own limit). `PUT` modifies `notes`/`active`/`per_interval` of an existing
non-master key. Non-master keys get `PERMISSION_NEED_ADMIN_API_KEY`/`AUTH_NEED_MASTER_API_KEY`.

### Webhook signing keys — `GET /.well-known/jwks.json` (no auth)

Ed25519 JWKS, cacheable 300s. Used to verify signatures on webhook payloads from endpoints called
with `webhook_url` set. **Don't generate raw signature-verification code for this** — see the
"Webhook verification" note in [python.md](python.md)/[javascript.md](javascript.md), which points
to Waterfall's own verification library instead.
