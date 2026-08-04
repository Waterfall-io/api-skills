# Job change detection — `/v1/job/change` (Preview)

Detects whether a contact has left, moved, or stayed at a company. Returns the full result
directly on `POST` (like search endpoints); a `GET` finder exists for optional re-fetch by
`job_id`.

**Request:** one of these identifier combinations, plus optional `custom_fields`:
1. `company_domain` + one of `contact_linkedin`, `professional_email`, `personal_email`, or `contact_full_name`
2. `company_linkedin` + one of `contact_linkedin`, `professional_email`, or `personal_email` (no `contact_full_name` in this combo)
3. `professional_email` alone, optionally with `contact_full_name`

```json
{"company_domain": "scale.com", "contact_linkedin": "connor-heggie"}
```
```json
{"professional_email": "john.doe@waterfall.io"}
```
```json
{"company_domain": "acme.com", "contact_full_name": "Jane Doe"}
```

**Response** — `output.job_change_status` is one of:

| Value | Meaning |
| --- | --- |
| `left` | Contact is no longer at the given company, new employer not identified |
| `moved` | Contact is now at a new, identified company |
| `no_change` | Contact is still at the given company |
| `unknown` | Move status could not be verified |

`output.person` is the contact profile with email and phone channels redacted — an empty object
(`{}`) when no matching profile was found, which is a normal `unknown` result, not an error.

```json
{
  "output": {
    "job_change_status": "moved",
    "person": {
      "first_name": "Jane", "last_name": "Doe",
      "company_name": "Acme Corp", "company_domain": "acme.com",
      "title": "Head of Sales",
      "experiences": [
        {"title": "Head of Sales", "company_domain": "acme.com", "is_current": true},
        {"title": "Senior Account Executive", "company_domain": "digitalriver.com", "is_current": false, "end_date": "2023-10-01"}
      ]
    },
    "usage": {"total_usd": 0.05, "balance_remaining_usd": 99.95}
  }
}
```

Standard [errors.md](errors.md) table applies; `VALIDATION_BAD_REQUEST` is most common when
neither a company+contact identifier pair nor `professional_email` is provided.
