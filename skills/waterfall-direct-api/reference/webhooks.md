# Webhook signature verification — `GET /.well-known/jwks.json`

Returns Waterfall's public JSON Web Key Set for verifying signatures on webhook payloads (several
launcher endpoints accept an optional `webhook_url` for async callback delivery instead of
polling). **No authentication required** — this is a public key endpoint, no `x-api-key` header
needed.

**Response:** Ed25519 keys in standard JWK format, cacheable for 300 seconds
(`Cache-Control: public, max-age=300`):

```json
{
  "keys": [
    {"kty": "OKP", "crv": "Ed25519", "kid": "webhook-key-2026-06", "use": "sig", "alg": "EdDSA", "x": "Q7HQWfd9_2hmzNwwBqqJ2l7CkDFv3cAvQ44asbkD-MA"}
  ]
}
```

To verify a webhook: match the incoming payload's key ID against `kid` in this set, then verify
the EdDSA signature using the corresponding `x` (base64url-encoded Ed25519 public key). Respect
the cache lifetime rather than fetching this on every webhook — keys don't rotate faster than
that window implies.

A `404`/`400` here means an invalid path or method (e.g. `POST` instead of `GET`) rather than a
Waterfall-side outage; a `500` means the key configuration is temporarily unavailable.
