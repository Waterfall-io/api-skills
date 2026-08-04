---
name: waterfall-integration
description: Generate client-owned integration code for the Waterfall API (prospecting, contact/company/phone enrichment, search, job-change detection, company reveal, email verification, account usage, API key management) in the caller's own stack — auth handling, request construction, response parsing, pagination, retries, and error handling written idiomatically in Python or JavaScript/TypeScript. Use this when the goal is source code that lives in the caller's codebase, not a one-off API call.
---

# Waterfall API integration generation

Generates client-owned source code that calls the Waterfall API — not a bundled Waterfall client
library, and not a one-off request. The output is code that lives in the caller's own repo, in
their language, using their existing HTTP client and conventions.

If the actual goal is just making an API call right now (exploring the data, a script, an ad hoc
lookup) rather than producing integration code to keep, use the companion skill
`waterfall-direct-api` instead — it's leaner for that.

## Principles

- **Idiomatic, not templated.** Match the caller's existing code style, HTTP client, error-handling
  conventions, and typing approach. The reference files here show patterns, not a library to paste
  in verbatim — adapt them.
- **Client owns the code.** Don't wrap Waterfall calls in a redistributable package unless asked;
  generate code that belongs to and is maintained by the caller's project.
- **Public surface only.** Only generate calls against endpoints in
  [reference/endpoints.md](reference/endpoints.md) — that reference is itself sourced from the
  published OpenAPI spec, not from anything internal.
- **Don't hand-roll webhook signature verification.** If the integration uses `webhook_url`
  instead of polling, point to
  [github.com/Waterfall-io/webhook-verification-kit](https://github.com/Waterfall-io/webhook-verification-kit)
  rather than generating Ed25519/JWKS verification code — see the per-language reference for why.

## Steps

1. Read [reference/endpoints.md](reference/endpoints.md) once — it's language-agnostic ground
   truth (auth, the async-vs-sync split, pagination, error model, field shapes) that both language
   guides build on.
2. Load only the language guide that matches the caller's stack —
   [reference/python.md](reference/python.md) or [reference/javascript.md](reference/javascript.md)
   — not both. Each shows idiomatic auth handling, retry/backoff, the async poll loop (only needed
   for Prospector and the three Enrichment endpoints — everything else is synchronous), pagination,
   and response typing for that language.
3. Generate the integration for the specific endpoint(s) the caller needs — don't generate
   wrappers for all 13 endpoints speculatively.
4. **Self-verify before calling it done:** make one real call against a documented endpoint with
   the generated code and check the response status/shape matches what
   [reference/endpoints.md](reference/endpoints.md) says, not just that the code compiles or runs
   without throwing. Each language guide has a concrete self-verification snippet.

## Why async vs. sync matters for the generated code

This is the single most important fact to get right before writing anything: Prospector and the
three Enrichment endpoints (`contact`/`phone`/`company`) return only a `job_id` on `POST` and
require polling a `GET` finder. Every other job-based endpoint returns its full result directly on
`POST`. Generating a single-request function for an async endpoint, or a poll loop for a sync one,
is the most common mistake here — check [reference/endpoints.md](reference/endpoints.md)'s shape
column per endpoint before writing the request function.
