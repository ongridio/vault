---
title: JWT / Token Auth Failures (401s After "Nothing Changed")
kind: howto
tags: [auth, jwt, token, oauth, 401, jwks, expiry, clock]
applies_to: [manager]
---

# JWT / Token Auth Failures (401s After "Nothing Changed")

Use when authenticated requests start returning 401/403 with valid-looking
tokens, often suddenly and fleet-wide. JWT validation checks signature,
expiry (`exp`/`nbf`), issuer/audience, and (for asymmetric) the signing
key from a JWKS endpoint. **The failure is almost always one of: expired
token, clock skew, a rotated signing key, or issuer/audience mismatch —
not "auth is broken".**

| Symptom | Probable cause |
|---|---|
| 401 `token expired` | short TTL + no refresh, or clock skew |
| 401 fleet-wide, suddenly | signing key rotated; validators have stale JWKS |
| `invalid signature` | wrong/old key, or alg mismatch |
| 401 `invalid audience/issuer` | token for a different aud/iss than expected |
| works on one host, fails on another | that host's clock is skewed |

## Step 1 — Decode the token + read the claims

```bash
# Decode (NOT verify) the payload to inspect claims — base64url of the middle part
TOK=<jwt>; echo "${TOK}" | cut -d. -f2 | tr '_-' '/+' | base64 -d 2>/dev/null | jq .
#   check: exp (expired?), nbf (not-yet-valid?), iss, aud, kid (which key)
date +%s    # compare to exp/nbf (epoch seconds)
```

If `exp` < now → expired (refresh/TTL issue). If `nbf` > now → not yet
valid (clock skew — the validator's clock is behind the issuer's).

## Step 2 — Clock skew (the sneaky one)

A validator whose clock is fast sees a just-issued token as expired; slow
sees it as `nbf` future. If 401s correlate with one host or started after
a VM migration, check time sync first (`diagnostics/clock-skew-ntp.md`) —
it's a frequent root cause of "token expired" on valid tokens.

## Step 3 — Key rotation / JWKS (the fleet-wide sudden one)

Asymmetric JWTs are verified with the issuer's public key, fetched from a
JWKS endpoint and cached. When the issuer rotates keys (new `kid`),
validators with a stale JWKS cache reject every new token (`invalid
signature` / unknown `kid`):

```bash
curl -sS https://<issuer>/.well-known/jwks.json | jq '.keys[].kid'   # current kids
# Does it include the token's kid? If not, the validator must refresh JWKS.
```

Fix: ensure validators refresh JWKS on unknown `kid` (don't cache
forever); align rotation with cache TTL.

## Step 4 — Issuer / audience / algorithm

```text
- aud/iss mismatch: token minted for service A presented to B — fix the
  client's target audience or the validator's expected aud/iss.
- alg mismatch / "alg=none": reject none; ensure validator pins the
  expected algorithm (security: never accept attacker-chosen alg).
```

## Decision tree

| Signal | Action |
|---|---|
| `exp` < now | token TTL too short / no refresh — fix refresh flow |
| `nbf` future / one host fails | clock skew — `diagnostics/clock-skew-ntp.md` |
| sudden fleet-wide invalid signature | key rotated — refresh JWKS / unknown-kid refetch |
| invalid audience/issuer | fix client target aud or validator expected iss/aud |
| alg mismatch | pin expected alg; never accept `none` |

## References

- [RFC 7519 — JSON Web Token (claims: exp, nbf, iss, aud)](https://www.rfc-editor.org/rfc/rfc7519)
- [JWT validation best practices — RFC 8725](https://www.rfc-editor.org/rfc/rfc8725)
- vault: `diagnostics/clock-skew-ntp.md`, `diagnostics/tls-handshake-failure.md`, `diagnostics/error-rate-5xx.md`
