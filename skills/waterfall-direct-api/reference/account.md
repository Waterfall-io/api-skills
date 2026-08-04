# Account usage — `GET /v2/account`

Reports usage for the authenticated key and the whole account. Free — not subject to the 50
req/min rate limit that applies to the data endpoints.

**Optional query param:** `month` (`YYYY-MM`) to scope the counters to a specific month; omit for
the current billing period.

**Response:**

```json
{
  "key_usage": {
    "start_date": "2024-02-01", "end_date": "2024-03-01",
    "prospector_requests": 1, "prospector_persons": 0,
    "verify_email_requests": 0, "verify_email_verified": 0,
    "search_contact_requests": 1, "search_contact_found": 0
  },
  "account_usage": { "...": "same shape as key_usage, but account-wide across all keys" },
  "balance_remaining_usd": 99.90
}
```

`key_usage` is scoped to the authenticated API key; `account_usage` covers the whole account —
they differ whenever an account has multiple keys. If authenticated with a **master** key, the
response also includes a `price` object with per-unit USD prices from the account's latest
payment (e.g. `persons_safe_usd`, `mobile_phones_usd`, `search_contact_found_usd`); sub-keys and
single keys never receive `price` — its absence isn't an error.

`404 NOT_FOUND_ACCOUNT_OR_KEY_NOT_FOUND` means the key/account behind the request couldn't be
resolved. Otherwise the standard [errors.md](errors.md) table applies.
