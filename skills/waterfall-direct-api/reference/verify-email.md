# Email verification — `POST /v1/verify/email`

The one endpoint that's genuinely synchronous end-to-end — no `job_id` polling, no finder. Verify
one email per call.

**Request:** `{ "email": "<address>" }`, optional `custom_fields`.

**Response:**

```json
{
  "status": "SUCCEEDED",
  "output": {
    "email": {
      "email": "john.doe@waterfall.io",
      "domain": "waterfall.io",
      "email_status": "valid",
      "smtp_provider": "Google",
      "mx_records": ["aspmx.l.google.com"]
    },
    "usage": {"total_usd": 0.01, "balance_remaining_usd": 99.99}
  }
}
```

`output.email.email_status` is one of `valid`, `invalid`, `risky`, `unknown` — check this field to
know the actual verification result; a 200 response with `email_status: "invalid"` is a
successful, billed API call that correctly reports an invalid address, not a failed request.

Standard [errors.md](errors.md) table applies for auth/validation/rate-limit failures — a
malformed `email` field returns `VALIDATION_BAD_REQUEST` with a field-level `error` message before
any verification is attempted.
