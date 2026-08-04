---
name: waterfall-direct-api
description: Call the Waterfall API (prospecting, contact/company/phone enrichment, search, job-change detection, company reveal, email verification, account usage, API key management) directly over raw HTTP with correct auth, request shapes, pagination, and error handling. Use this when a request should hit api.waterfall.io directly in whatever language/tool is already at hand, rather than through a generated client integration.
---

# Waterfall direct API usage

Waterfall's API lets you find and enrich B2B contact and company data: prospecting by
title/location, contact and company search, contact/phone/company enrichment, job-change
detection, IP-to-company reveal, email verification, and account/API-key management.

This skill teaches you to call the API directly with `curl`/`fetch`/`requests`/etc. It does not
generate a client integration for you — if the goal is idiomatic, client-owned integration code,
that's a different, companion skill (`waterfall-integration`), not this one.

**Source of truth:** [docs.waterfall.io](https://docs.waterfall.io/) and the full OpenAPI 3.1 spec,
fetchable directly at
[waterfall-io-public.s3.us-east-1.amazonaws.com/api_specs.yaml](https://waterfall-io-public.s3.us-east-1.amazonaws.com/api_specs.yaml)
(linked from [docs.waterfall.io/v1/openapi-spec](https://docs.waterfall.io/v1/openapi-spec) — check
that page if the direct link ever 404s). This skill does not restate full request/response
schemas — it orients you and points at the field-level detail loaded on demand from the reference
files in this folder, so schema drift doesn't rot copy-pasted examples. If anything here conflicts
with the live spec or docs.waterfall.io, they win.

## Base URL and auth

- Base URL: `https://api.waterfall.io`
- Provide your API key in the `x-api-key` request header on every call. There is no OAuth flow,
  no bearer token, no query-param key.
- Missing or wrong key fails fast with a 401/403 and a machine-readable code (see
  [reference/errors.md](reference/errors.md)) — check for that before assuming a request body
  problem.

## The two request shapes you'll see

Prospector and the three enrichment endpoints are **launcher/finder pairs**: `POST` a launcher to
start an async job (returns only a `job_id`), then `GET` the corresponding finder with
`?job_id=...` to retrieve results once the job finishes. Every other job-based endpoint (email
verification, company reveal, company titles, job change, search contact, search company) is
synchronous and returns the full result directly on the `POST` — no polling required, though a
`GET` finder is also available on most of them to re-fetch a completed job later by `job_id`. The
per-endpoint reference below states which shape each endpoint uses — don't assume from the path
alone.

Every job response — sync or polled — carries a `status` of `RUNNING`, `SUCCEEDED`, `FAILED`,
`TIMED_OUT`, or `ABORTED`. Treat a 200 HTTP status as "the request was accepted," not "the job
succeeded": check the body's `status` field before trusting `output`, and only `SUCCEEDED` means
`output` is populated and safe to use.

**Only read the documented `output` field.** Don't parse or rely on fields that aren't documented
in `api_specs.yaml` / docs.waterfall.io — treat undocumented fields as not part of the contract,
even if a particular key happens to return one.

## Pagination

`/v1/search/company` and `/v1/search/contact` paginate via `page_number` / `page_size` in the
request body — there is no cursor and no `next_page` link, so request the next `page_number`
explicitly. Prospector is different: it has no pagination at all, just a `limit` cap on the number
of contacts returned in the single response. Don't assume either mechanism applies to an endpoint
you haven't checked — see the per-endpoint reference below.

## Rate limits

Prospector, Search, Enrichment, Job Change, and Verify Email are capped at 50 requests/minute. When
you exceed it, you get `RATE_LIMIT_RATE_LIMIT_EXCEEDED` plus `X-RateLimit-Limit`,
`X-RateLimit-Interval`, and `Retry-After` response headers — read `Retry-After` rather than
guessing a backoff. The Account Reporter endpoint (`/v2/account`) has no rate limit.

## Errors and self-verification

Every error response has the same envelope: `status`, `message`, `category`, `code` (and
sometimes `error` with field-level detail). `category` is the coarse signal for retry/auth/quota
handling; `code` is the specific, stable identifier — branch on `code`, not on parsing `message`
text, since `message` is meant for humans and can change. Full category/code table:
[reference/errors.md](reference/errors.md).

Before treating a call as successful:

1. Check the HTTP status and the body's job `status` field, not just a 2xx.
2. If it's an error, read `category` first for broad handling, then `code` for precise logic —
   don't guess at failure meaning from `message` alone.
3. For finder endpoints, `status: "RUNNING"` is not a failure — poll again rather than treating an
   in-progress job as an error. Only `FAILED`, `TIMED_OUT`, and `ABORTED` mean the job didn't
   succeed.
4. Validate the shape of `output` against the relevant reference file below before using the
   data — fields can be `null` when a value wasn't found (e.g. an unresolved phone number), which
   is a normal "not found" result, not a malformed response.

## Endpoints

| Capability | Endpoint(s) | Shape | Stage | Reference |
| --- | --- | --- | --- | --- |
| Prospector | `POST /v1/prospector`, `GET /v1/prospector` | launcher/finder | GA | [reference/prospector.md](reference/prospector.md) |
| Contact/phone/company enrichment | `/v1/enrichment/contact`, `/v1/enrichment/phone`, `/v1/enrichment/company` | launcher/finder | GA | [reference/enrichment.md](reference/enrichment.md) |
| Search contact | `POST /v1/search/contact`, `GET /v1/search/contact` | sync, paginated | GA | [reference/search.md](reference/search.md) |
| Search company | `POST /v1/search/company` | sync, paginated | Preview | [reference/search.md](reference/search.md) |
| Job change | `POST /v1/job/change` | sync | Preview | [reference/job-change.md](reference/job-change.md) |
| Company reveal | `POST /v1/company-reveal`, `GET /v1/company-reveal` | sync launch, persisted finder | Preview | [reference/company-reveal.md](reference/company-reveal.md) |
| Company titles | `POST /v1/company-titles` | sync | Preview | [reference/company-titles.md](reference/company-titles.md) |
| Email verification | `POST /v1/verify/email` | sync | GA | [reference/verify-email.md](reference/verify-email.md) |
| Account usage | `GET /v2/account` | sync | GA | [reference/account.md](reference/account.md) |
| API key management | `/v1/api-keys` | sync | GA | [reference/api-keys.md](reference/api-keys.md) |
| Webhook signing keys | `GET /.well-known/jwks.json` | sync, no auth | GA | [reference/webhooks.md](reference/webhooks.md) |

Load only the reference file(s) relevant to the task at hand — each is self-contained with
request/response fields, auth notes, and a worked example.

`Preview` endpoints (Search Company, Job Change, Company Reveal, Company Titles) have no SLA and
their behavior may still change — say so if you're integrating one into something production-
critical.
