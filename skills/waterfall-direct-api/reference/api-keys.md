# API key management — `/v1/api-keys`

Manage sub-keys under an account. **All three operations require authenticating with the
account's master API key** — a non-master key gets `PERMISSION_NEED_ADMIN_API_KEY` or
`AUTH_NEED_MASTER_API_KEY`. This is an Enterprise feature; not every account has it enabled.

## `GET /v1/api-keys` — list keys

No request body. Returns every key on the account:

```json
{
  "api_keys": [
    {"api_key": "ad18e456-...", "active": true, "notes": "team-a", "master": false, "per_interval": 30, "interval_seconds": 60}
  ]
}
```

## `POST /v1/api-keys` — create a sub-key

**Required:** `notes`. **Optional:** `per_interval` (requests per `interval_seconds`; defaults to
50 if omitted) — cannot exceed the master key's own `per_interval`, or you'll get
`VALIDATION_SUBKEY_RATE_EXCEEDS_MASTER`.

```json
{"notes": "team-a", "per_interval": 30}
```

Response is the created key object (same shape as a list entry). `404
NOT_FOUND_NO_MASTER_API_KEY_FOUND` if the account has no master key configured (e.g. feature not
enabled — talk to your account manager).

## `PUT /v1/api-keys` — modify a sub-key

**Required:** `api_key`, `notes`, `active`. **Optional:** `per_interval` (retains current value if
omitted). Only `notes`, `active`, and `per_interval` are editable — **the master key itself cannot
be edited** (`VALIDATION_CANNOT_EDIT_MASTER_API_KEY` if attempted).

```json
{"api_key": "ad18e456-...", "notes": "updated note", "active": true, "per_interval": 25}
```

`404 NOT_FOUND_API_KEY_NOT_FOUND` if `api_key` doesn't belong to the account.

Standard [errors.md](errors.md) table applies otherwise; `INTERNAL_FAILED_CREATE_API_KEY` /
`INTERNAL_FAILED_EDIT_API_KEY` on a 500 are safe to retry.
