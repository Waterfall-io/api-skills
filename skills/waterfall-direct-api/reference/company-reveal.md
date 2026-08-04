# Company reveal — `/v1/company-reveal` (Preview)

Resolves an IPv4 address to the company likely behind it (e.g. for deanonymizing website
visitors). Returns the full result directly on `POST`; `GET` re-fetches a persisted job by
`job_id`.

**Request:** `{ "ip": "<ipv4>" }`, optional `webhook_url`, `custom_fields`.

```json
{"ip": "8.8.8.8"}
```

**Response:**

```json
{
  "output": {
    "ip": "8.8.8.8",
    "type": "company",
    "confidence_score": "high",
    "ip_geo": {"city": "San Francisco", "state": "California", "country": "United States"},
    "company": {
      "domain": "stripe.com", "name": "Stripe", "website": "stripe.com",
      "size": "5001-10000", "employees_count": 8000, "industry": "Financial Services",
      "country": "United States", "linkedin_url": "https://www.linkedin.com/company/stripe/"
    },
    "usage": {"total_usd": 0.07, "balance_remaining_usd": 99.93}
  }
}
```

`type` is one of `company`, `education`, `government`, `isp`, or `null` — an `isp` result means
the IP resolves to a residential/ISP network, not a specific company, so treat `output.company` as
unreliable in that case. `confidence_score` is one of `very_high`, `high`, `medium`, `low`, or
`null`; weight the match accordingly rather than treating any non-null `company` as certain.
Standard [errors.md](errors.md) table applies for auth/validation/rate-limit failures.
