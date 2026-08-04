# Company titles — `/v1/company-titles` (Preview)

Lists active job titles at a company. Requires company titles access enabled on the account
(`PERMISSION_FEATURE_NOT_ENABLED` if not). Returns the full result on `POST`; `GET` re-fetches by
`job_id`.

**Request:** one company identifier — `domain` or `company_linkedin` — plus optional
`page_number` and `custom_fields`.

```json
{"domain": "example.com", "page_number": 1}
```

**Response:**

```json
{
  "output": {
    "titles": ["Software Engineer", "VP Sales"],
    "has_more_pages": false,
    "usage": {"total_usd": 0.70, "balance_remaining_usd": 99.30}
  }
}
```

`has_more_pages: true` means request the next `page_number` for more titles — there's no
`page_size` parameter (page size is fixed server-side). Billing note: usage is charged as one
successful lookup only when the company resolves **and** at least one active title is returned;
an empty `titles` list for a resolved company is not billed as a hit.

A `404 NOT_FOUND_NOT_FOUND` ("Company not found") means the identifier didn't resolve to a known
company — distinct from a resolved company with zero titles. See [errors.md](errors.md) for the
rest of the shared error table.
