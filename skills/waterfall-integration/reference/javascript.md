# JavaScript / TypeScript integration patterns

Idiomatic patterns for a Waterfall client in JS/TS. Adapt to the client's existing conventions
(their HTTP client, their config/secrets approach, CommonJS vs ESM) — not a library to paste in
verbatim. Ground truth for field names/shapes: [endpoints.md](endpoints.md).

## HTTP client

Use the runtime's native `fetch` unless the codebase already standardizes on `axios` or similar.
Read the API key from the codebase's existing config/secrets mechanism (`process.env`, a secrets
manager) — don't hardcode it.

```typescript
const BASE_URL = "https://api.waterfall.io";

function headers(): Record<string, string> {
  return { "x-api-key": process.env.WATERFALL_API_KEY!, "Content-Type": "application/json" };
}
```

## Error handling

Map the `{status, message, category, code}` envelope to a typed error. Branch on `code` (or
`category` as a fallback for codes not yet handled) — never on `message` text.

```typescript
class WaterfallAPIError extends Error {
  constructor(public category: string, public code: string, message: string) {
    super(`${code}: ${message}`);
  }
}

async function raiseForError(res: Response): Promise<void> {
  if (res.ok) return;
  const body = await res.json();
  throw new WaterfallAPIError(body.category ?? "UNKNOWN", body.code ?? "UNKNOWN", body.message ?? "");
}
```

## Retries and rate limits

Retry `429` (respecting `Retry-After`) and `500` with backoff; don't retry `400`/`401`/`403`/`404`
— those need the request fixed, not repeated.

```typescript
async function request(method: string, path: string, body?: unknown): Promise<any> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const res = await fetch(`${BASE_URL}${path}`, { method, headers: headers(), body: body ? JSON.stringify(body) : undefined });
    if (res.status === 429) {
      const retryAfter = Number(res.headers.get("Retry-After") ?? "5");
      await new Promise((r) => setTimeout(r, retryAfter * 1000));
      continue;
    }
    if (res.status >= 500) {
      await new Promise((r) => setTimeout(r, 2 ** attempt * 1000));
      continue;
    }
    await raiseForError(res);
    return res.json();
  }
  throw new WaterfallAPIError("RETRY", "MAX_RETRIES", "exhausted retries");
}
```

## Async endpoints (launcher/finder) — poll, don't assume synchronous

Prospector and the three Enrichment endpoints only return `job_id` on launch. Poll the finder
until `status` leaves `RUNNING`:

```typescript
async function poll(path: string, jobId: string, timeoutMs = 60_000, intervalMs = 1_500): Promise<any> {
  const deadline = Date.now() + timeoutMs;
  while (Date.now() < deadline) {
    const result = await request("GET", `${path}?job_id=${jobId}`);
    if (result.status !== "RUNNING") return result;
    await new Promise((r) => setTimeout(r, intervalMs));
  }
  throw new Error(`job ${jobId} still RUNNING after ${timeoutMs}ms`);
}

async function runProspector(domain: string, titleFilter: string, limit = 10): Promise<any> {
  const launch = await request("POST", "/v1/prospector", { domain, title_filter: titleFilter, limit });
  const result = await poll("/v1/prospector", launch.job_id);
  if (result.status !== "SUCCEEDED") {
    throw new WaterfallAPIError("JOB", result.status, `prospector job ended in ${result.status}`);
  }
  return result.output;
}
```

Sync endpoints (Search Contact, Search Company, Job Change, Company Reveal, Company Titles, Email
Verification) skip the poll — check `result.status === "SUCCEEDED"` directly on the `POST`
response.

## Pagination

For Search Contact / Search Company, loop `page_number` until a page comes back short of
`page_size`:

```typescript
async function searchCompaniesAll(industries: string[], pageSize = 20): Promise<any[]> {
  let page = 1;
  const companies: any[] = [];
  while (true) {
    const result = await request("POST", "/v1/search/company", { industries, page_number: page, page_size: pageSize });
    const batch = result.output.companies;
    companies.push(...batch);
    if (batch.length < pageSize) return companies;
    page += 1;
  }
}
```

Company Titles paginates by `page_number` alone — check `output.has_more_pages` instead of
comparing batch length to a page size (it has none).

## Response typing

Define TypeScript interfaces for only the fields you actually use — Waterfall may add fields over
time and unused ones shouldn't break parsing.

```typescript
interface Person {
  first_name: string | null;
  last_name: string | null;
  title: string | null;
  professional_email: string | null;
  email_verified: boolean | null;
}
```

## Self-verification

Before treating the integration as done, make one real call against a documented endpoint and
assert the shape, not just that it didn't throw. For a sync endpoint that's a single call:

```typescript
const result = await runVerifyEmail("test@example.com"); // or any sync endpoint you just wired up
if (result.status !== "SUCCEEDED" || !("email_status" in result.output.email)) {
  throw new Error("verify-email integration check failed");
}
```

For an async (launcher/finder) endpoint, verify the *whole* flow — launch, poll, and the final
`output` shape — not just that the launch call returned a `job_id`:

```typescript
const output = await runProspector("waterfall.io", "founder OR ceo", 5); // returns result.output on SUCCEEDED
if (!("company" in output) || !("persons" in output)) {
  throw new Error("prospector integration check failed");
}
```

## Webhook verification

If the integration uses `webhook_url` instead of polling, **don't hand-write Ed25519/JWKS
signature verification** — use Waterfall's own library:
[github.com/Waterfall-io/webhook-verification-kit](https://github.com/Waterfall-io/webhook-verification-kit)
(`typescript/` subfolder). Signature-verification code is easy to get subtly wrong; prefer the
maintained library over regenerating it.
