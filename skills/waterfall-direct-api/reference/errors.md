# Error handling

Every error response uses the same envelope:

```json
{
  "status": "error",
  "message": "Bad request",
  "category": "VALIDATION",
  "code": "VALIDATION_BAD_REQUEST",
  "error": ["email: Field required"]
}
```

- `category` — coarse classification, always the prefix of `code`. Use it for broad handling
  (retry, re-auth, back off, surface to user as a permission/quota problem).
- `code` — stable, specific identifier. Branch application logic on this, not on `message`, which
  is meant for humans and can change wording without notice.
- `error` — present on some validation failures; either a string or a list of up to 10
  field-level messages (e.g. `"email: Field required"`).

## HTTP status → category

| Status | Meaning |
| --- | --- |
| 200 | Request accepted — check the body's job `status` before trusting `output` |
| 400 | `VALIDATION_*` or `ROUTING_*` — malformed request or wrong method/path |
| 401 | `AUTH_MISSING_API_KEY` — no `x-api-key` header sent |
| 403 | `AUTH_BAD_API_KEY`, `PERMISSION_*` — key present but invalid, or valid but lacking a required permission |
| 404 | `NOT_FOUND_*` — unknown endpoint, or a `job_id`/API key that doesn't exist |
| 402 | `QUOTA_ACCOUNT_OVER_QUOTA` — account is over its usage quota; not a request problem, contact your account manager |
| 429 | `RATE_LIMIT_RATE_LIMIT_EXCEEDED` — read `Retry-After` (seconds) before retrying |
| 500 | `INTERNAL_*` — Waterfall-side failure; safe to retry with backoff |

## Full category/code table

| Category | Code | When you'll see it |
| --- | --- | --- |
| AUTH | `AUTH_BAD_API_KEY` | `x-api-key` header sent but not a valid key |
| AUTH | `AUTH_MISSING_API_KEY` | No `x-api-key` header sent |
| AUTH | `AUTH_NEED_MASTER_API_KEY` | API key management action that requires the account's master key |
| PERMISSION | `PERMISSION_FEATURE_NOT_ENABLED` | Endpoint/feature not enabled on this account |
| PERMISSION | `PERMISSION_INCLUDE_PHONES_REQUIRES_PHONE_ENRICHMENT` | Requested phone data without phone enrichment enabled on the account |
| PERMISSION | `PERMISSION_NEED_ADMIN_API_KEY` | Action requires an admin-scoped key |
| QUOTA | `QUOTA_ACCOUNT_OVER_QUOTA` | Account usage/credit limit reached |
| VALIDATION | `VALIDATION_BAD_DOMAIN` | Domain field failed general validation |
| VALIDATION | `VALIDATION_BAD_DOMAIN_DISPOSABLE` | Domain is a disposable-email domain |
| VALIDATION | `VALIDATION_BAD_DOMAIN_EMAIL_PROVIDER` | Domain is a consumer email provider (e.g. gmail.com) |
| VALIDATION | `VALIDATION_BAD_DOMAIN_INVALID_CHARACTERS` | Domain contains invalid characters |
| VALIDATION | `VALIDATION_BAD_DOMAIN_NO_MX` | Domain has no MX record |
| VALIDATION | `VALIDATION_BAD_DOMAIN_PROVIDER_DOMAIN` | Domain belongs to a hosting/SaaS provider, not a real company |
| VALIDATION | `VALIDATION_BAD_DOMAIN_SERVICE_DOMAIN` | Domain is a known non-company service domain |
| VALIDATION | `VALIDATION_BAD_DOMAIN_SOCIAL_MEDIA` | Domain is a social media platform, not a company |
| VALIDATION | `VALIDATION_BAD_DOMAIN_TOP_LEVEL` | Domain is a bare top-level/registry domain |
| VALIDATION | `VALIDATION_BAD_DOMAIN_URL_SHORTENER` | Domain is a URL shortener |
| VALIDATION | `VALIDATION_BAD_EMAIL_CONSECUTIVE_DIGITS` | Email local-part has too many consecutive digits (likely fake) |
| VALIDATION | `VALIDATION_BAD_EMAIL_INVALID` | Email fails basic format validation |
| VALIDATION | `VALIDATION_BAD_EMAIL_PERSONAL_NOT_ALLOWED` | Endpoint requires a professional email; a personal one was given |
| VALIDATION | `VALIDATION_BAD_EMAIL_PROFESSIONAL_NOT_ALLOWED` | Endpoint requires a personal email; a professional one was given |
| VALIDATION | `VALIDATION_BAD_EMAIL_ROLE` | Email is a role address (e.g. `info@`, `support@`) |
| VALIDATION | `VALIDATION_BAD_EMAIL_TOO_LONG` | Email exceeds max length |
| VALIDATION | `VALIDATION_BAD_EMAIL_TOO_MANY_DIGITS` | Email local-part has too many digits overall |
| VALIDATION | `VALIDATION_BAD_JOB_ID` | `job_id` is not a valid UUID |
| VALIDATION | `VALIDATION_BAD_NAME_INVALID` | First/last/full name field fails validation |
| VALIDATION | `VALIDATION_BAD_REQUEST` | General request-body validation failure — see `error` for the specific field(s) |
| VALIDATION | `VALIDATION_BAD_TITLE_FILTER` | `title_filter` boolean expression is invalid |
| VALIDATION | `VALIDATION_BAD_TITLE_FILTER_EXTRA_LEFT_PARENTHESIS` | Unmatched `(` in `title_filter` |
| VALIDATION | `VALIDATION_BAD_TITLE_FILTER_EXTRA_RIGHT_PARENTHESIS` | Unmatched `)` in `title_filter` |
| VALIDATION | `VALIDATION_BAD_TITLE_FILTER_EXTRA_STRING` | Trailing/unexpected token in `title_filter` |
| VALIDATION | `VALIDATION_BAD_TITLE_FILTER_MISSING_RIGHT_PARENTHESIS` | `title_filter` missing a closing `)` |
| VALIDATION | `VALIDATION_BAD_TITLE_FILTER_MISSING_STRING` | `title_filter` has an operator with no operand |
| VALIDATION | `VALIDATION_BAD_TITLE_FILTER_MISSING_STRING_OR_NOT_OR_LEFT_PARENTHESIS` | `title_filter` syntax error at expected term/`NOT`/`(` |
| VALIDATION | `VALIDATION_CANNOT_EDIT_MASTER_API_KEY` | Attempted to edit/deactivate the account's master API key via the API keys endpoint |
| VALIDATION | `VALIDATION_MISSING_BODY` | Request body required but not sent |
| VALIDATION | `VALIDATION_MISSING_JOB_ID_PARAMETER` | Finder call missing the required `job_id` query parameter |
| VALIDATION | `VALIDATION_MISSING_TITLE_FILTERS` | Neither `title_filter` nor `title_filters` provided |
| VALIDATION | `VALIDATION_SUBKEY_RATE_EXCEEDS_MASTER` | Sub-key rate limit configured higher than the master key allows |
| NOT_FOUND | `NOT_FOUND_ACCOUNT_OR_KEY_NOT_FOUND` | Account or key referenced in an API-key-management call doesn't exist |
| NOT_FOUND | `NOT_FOUND_API_KEY_NOT_FOUND` | `key_id` referenced in an API-key-management call doesn't exist |
| NOT_FOUND | `NOT_FOUND_JOB_NOT_FOUND` | `job_id` doesn't exist or has expired |
| NOT_FOUND | `NOT_FOUND_NO_MASTER_API_KEY_FOUND` | Account has no master API key configured |
| NOT_FOUND | `NOT_FOUND_NOT_FOUND` | Unknown path |
| RATE_LIMIT | `RATE_LIMIT_RATE_LIMIT_EXCEEDED` | Over the 50 req/min cap — respect `Retry-After` |
| ROUTING | `ROUTING_INVALID_PATH_OR_METHOD` | Right path, wrong HTTP method (e.g. `GET` on a launcher) |
| INTERNAL | `INTERNAL_FAILED_CREATE_API_KEY` | Waterfall-side failure creating an API key — safe to retry |
| INTERNAL | `INTERNAL_FAILED_EDIT_API_KEY` | Waterfall-side failure editing an API key — safe to retry |
| INTERNAL | `INTERNAL_UNCLASSIFIED_ERROR` | Unclassified server-side failure — safe to retry with backoff |

Codes prefixed `VALIDATION_BAD_TITLE_FILTER*` and `VALIDATION_MISSING_TITLE_FILTERS` are specific
to Prospector's `title_filter`/`title_filters` parameters — see [prospector.md](prospector.md).
