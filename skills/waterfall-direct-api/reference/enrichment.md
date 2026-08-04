# Contact, phone, and company enrichment

Three separate launcher/finder pairs, all async (POST returns only `job_id` + `start_date`; poll
GET for the result). Each takes an identifier strategy rather than a search filter — use these
when you already know who/what you're enriching, not for discovery (that's
[prospector.md](prospector.md) or [search.md](search.md)).

## Contact enrichment — `/v1/enrichment/contact`

Provide **one** identifier strategy: `linkedin`, `email`, `full_name` + `domain`, or
`first_name` + `last_name` + `domain`. If `email` is given, Waterfall does email-based enrichment;
else if `linkedin` is given, LinkedIn-based; otherwise name+domain. A professional email generally
gives the best quality. Optional: `include_phones`, `webhook_url`, `custom_fields`.

```json
{"email": "john.doe@waterfall.io", "include_phones": false}
```

`GET /v1/enrichment/contact?job_id=<uuid>` once `status: "SUCCEEDED"`:

```json
{
  "output": {
    "person": {
      "first_name": "Jane", "last_name": "Doe",
      "linkedin_url": "https://www.linkedin.com/in/jane-doe-abc123/",
      "company_name": "Acme Corp", "company_domain": "acme.com",
      "professional_email": "jane.doe@acme.com",
      "title": "Head of Sales", "seniority": "Director", "department": "Sales",
      "email_verified": true, "email_confidence": "high", "email_verified_status": "safe",
      "phone_numbers": []
    },
    "usage": {"total_usd": 0.10, "balance_remaining_usd": 99.90}
  }
}
```

## Phone enrichment — `/v1/enrichment/phone`

Same identifier strategy as contact enrichment (`linkedin`, `email`, or name+`domain`). LinkedIn
gives the best quality. **You're only charged when a number is actually found** — a `SUCCEEDED`
job with `mobile_phone: null` and empty `phone_numbers` is a normal "not found" outcome, not an
error, and isn't billed the phone rate.

```json
{"linkedin": "waterfall-io", "domain": "waterfall.io"}
```

Response shape is the same `person` object as contact enrichment but with fewer fields populated,
plus `mobile_phone` and `phone_numbers` (both E.164 format, e.g. `+14155550123`).

## Company enrichment — `/v1/enrichment/company`

One identifier: `domain`, `linkedin`, or `name` (name-based has the lowest quality/highest false-
positive risk — prefer `domain` or `linkedin`). Optional `webhook_url`, `custom_fields`.

`output.company` fields: `id`, `domain`, `name`, `website`, `linkedin_id`, `linkedin_url`,
`linkedin_followers`, `description`, `logo_url`, `size` (employee-range string, e.g.
`"501-1000"`), `employees_count`, `industry`, `type`, `founded`, `address`, `country`,
`crunchbase_url`, `funding_details` (`total_funding_rounds`, `total_funding_usd`,
`funding_rounds`), `recent_job_posting_count`, `technologies` (list).

## Errors specific to these endpoints

`PERMISSION_INCLUDE_PHONES_REQUIRES_PHONE_ENRICHMENT` if `include_phones: true` is set on contact
enrichment without phone enrichment enabled on the account. Otherwise the shared
[errors.md](errors.md) table applies — most failures here are `VALIDATION_BAD_REQUEST` for a
missing/conflicting identifier field.
