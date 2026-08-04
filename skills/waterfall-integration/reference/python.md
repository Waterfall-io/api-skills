# Python integration patterns

Idiomatic patterns for a Waterfall client in Python. These are patterns to adapt to the client's
existing conventions (their HTTP client, their config/secrets approach, their typing style) — not
a library to paste in verbatim. Ground truth for field names/shapes: [endpoints.md](endpoints.md).

## HTTP client

Use `requests` unless the codebase already standardizes on something else (`httpx` for async).
Read the API key from wherever the codebase already manages secrets — don't hardcode it or invent
a new config mechanism just for this integration.

```python
import os
import requests

BASE_URL = "https://api.waterfall.io"

def _headers() -> dict[str, str]:
    return {"x-api-key": os.environ["WATERFALL_API_KEY"], "Content-Type": "application/json"}
```

## Error handling

Map the `{status, message, category, code}` envelope to a typed exception. Branch on `code` (or
`category` as a fallback for codes not yet in your mapping) — never on `message` text.

```python
class WaterfallAPIError(Exception):
    def __init__(self, category: str, code: str, message: str):
        self.category = category
        self.code = code
        super().__init__(f"{code}: {message}")

def _raise_for_error(resp: requests.Response) -> None:
    if resp.ok:
        return
    body = resp.json()
    raise WaterfallAPIError(body.get("category", "UNKNOWN"), body.get("code", "UNKNOWN"), body.get("message", ""))
```

## Retries and rate limits

Retry `429` (respecting `Retry-After`) and `500` with backoff; don't retry `400`/`401`/`403`/`404`
— those need the request fixed, not repeated. Use the codebase's existing retry utility if one
exists; otherwise a small explicit loop is clearer than pulling in a new dependency for this alone.

```python
import time

def _request(method: str, path: str, **kwargs) -> dict:
    for attempt in range(4):
        resp = requests.request(method, f"{BASE_URL}{path}", headers=_headers(), **kwargs)
        if resp.status_code == 429:
            time.sleep(int(resp.headers.get("Retry-After", "5")))
            continue
        if resp.status_code >= 500:
            time.sleep(2 ** attempt)
            continue
        _raise_for_error(resp)
        return resp.json()
    _raise_for_error(resp)
```

## Async endpoints (launcher/finder) — poll, don't assume synchronous

Prospector and the three Enrichment endpoints only return `job_id` on launch. Poll the finder
until `status` leaves `RUNNING`:

```python
def _poll(path: str, job_id: str, timeout_s: int = 60, interval_s: float = 1.5) -> dict:
    deadline = time.monotonic() + timeout_s
    while time.monotonic() < deadline:
        result = _request("GET", path, params={"job_id": job_id})
        if result["status"] != "RUNNING":
            return result
        time.sleep(interval_s)
    raise TimeoutError(f"job {job_id} still RUNNING after {timeout_s}s")

def run_prospector(domain: str, title_filter: str, limit: int = 10) -> dict:
    launch = _request("POST", "/v1/prospector", json={"domain": domain, "title_filter": title_filter, "limit": limit})
    result = _poll("/v1/prospector", launch["job_id"])
    if result["status"] != "SUCCEEDED":
        raise WaterfallAPIError("JOB", result["status"], f"prospector job ended in {result['status']}")
    return result["output"]
```

Sync endpoints (Search Contact, Search Company, Job Change, Company Reveal, Company Titles, Email
Verification) skip the poll — check `result["status"] == "SUCCEEDED"` directly on the `POST`
response.

## Pagination

For Search Contact / Search Company, loop `page_number` until a page comes back short of
`page_size`:

```python
def search_companies_all(industries: list[str], page_size: int = 20) -> list[dict]:
    page, companies = 1, []
    while True:
        result = _request("POST", "/v1/search/company", json={"industries": industries, "page_number": page, "page_size": page_size})
        batch = result["output"]["companies"]
        companies.extend(batch)
        if len(batch) < page_size:
            return companies
        page += 1
```

Company Titles paginates by `page_number` alone — check `output["has_more_pages"]` instead of
comparing batch length to a page size (it has none).

## Response typing

Use `dataclasses` (stdlib, no new dependency) or Pydantic if the codebase already depends on it —
don't add Pydantic solely for this integration. Only type the fields you actually use; Waterfall
may add fields over time and unused ones shouldn't break parsing.

```python
from dataclasses import dataclass

@dataclass
class Person:
    first_name: str | None
    last_name: str | None
    title: str | None
    professional_email: str | None
    email_verified: bool | None
```

## Self-verification

Before treating the integration as done, make one real call against a documented endpoint and
assert the shape, not just that it didn't raise. For a sync endpoint that's a single call:

```python
result = run_verify_email("test@example.com")  # or any sync endpoint you just wired up
assert result["status"] == "SUCCEEDED"
assert "email_status" in result["output"]["email"]
```

For an async (launcher/finder) endpoint, verify the *whole* flow — launch, poll, and the final
`output` shape — not just that the launch call returned a `job_id`:

```python
output = run_prospector("waterfall.io", "founder OR ceo", limit=5)  # returns result["output"] on SUCCEEDED
assert "company" in output and "persons" in output
```

A launch that returns a `job_id` but a poll loop that never checks the terminal `status` isn't
verified — the job could have ended `FAILED`/`TIMED_OUT`/`ABORTED` and your code would still treat
the (empty) `output` as good data.

## Webhook verification

If the integration uses `webhook_url` instead of polling, **don't hand-write Ed25519/JWKS
signature verification** — use Waterfall's own library:
[github.com/Waterfall-io/webhook-verification-kit](https://github.com/Waterfall-io/webhook-verification-kit)
(`python/` subfolder). Signature-verification code is easy to get subtly wrong; prefer the
maintained library over regenerating it.
