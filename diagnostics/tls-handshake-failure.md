---
title: TLS Handshake Failures & Certificate Problems
kind: howto
tags: [tls, ssl, certificate, handshake, x509, expiry, sni]
applies_to: [edge, manager]
---

# TLS Handshake Failures & Certificate Problems

Use when connections fail with `certificate verify failed`, `handshake
failure`, `unknown ca`, or a browser/client refuses HTTPS. **The error
string already narrows the layer** — read it before touching the server.

| Error | Probable cause |
|---|---|
| `certificate has expired` | leaf or an intermediate past `notAfter` |
| `unable to get local issuer` / `unknown ca` | missing intermediate in the served chain, or client lacks the root |
| `hostname mismatch` / `no alternative names` | SNI/CN/SAN doesn't cover the name |
| `handshake failure` / `no shared cipher` | TLS version / cipher mismatch |
| `bad certificate` after client hello | mutual-TLS: client cert rejected |

## Step 1 — See exactly what the server presents

```bash
echo | openssl s_client -connect HOST:443 -servername HOST 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
# Full chain + verification verdict
echo | openssl s_client -connect HOST:443 -servername HOST -showcerts 2>/dev/null | grep -E 'verify|s:|i:'
```

`-servername` matters: without SNI you may get the default vhost cert and
chase a phantom mismatch. Check `notAfter` (expiry) and the SAN list.

## Step 2 — Validate the chain order + completeness

```bash
# The server MUST send leaf + intermediates (not the root). Missing
# intermediate = "unknown ca" on strict clients even if your browser is fine.
openssl s_client -connect HOST:443 -servername HOST 2>/dev/null </dev/null \
  | openssl verify -untrusted /dev/stdin
```

Common trap: it works in Chrome (which caches/fetches intermediates) but
fails in curl/Go/Java which don't — the server's chain is incomplete.

## Step 3 — Protocol / cipher negotiation

```bash
nmap --script ssl-enum-ciphers -p 443 HOST          # what the server offers
openssl s_client -connect HOST:443 -tls1_2          # force a version to test
```

A modern client refusing TLS 1.0/1.1, or a server that disabled the
client's only cipher, surfaces as `handshake failure` / `no shared
cipher`. Align the supported set.

## Step 4 — Expiry monitoring (catch it before it fires)

```bash
# Days left for a host
echo | openssl s_client -connect HOST:443 -servername HOST 2>/dev/null \
  | openssl x509 -noout -enddate
# A clock-skewed client sees a valid cert as "not yet valid" / "expired"
timedatectl | grep -E 'System clock|synchronized'
```

If the cert dates look fine but one client still rejects it, suspect that
client's clock — see `diagnostics/clock-skew-ntp.md`.

## Decision tree

| Signal | Action |
|---|---|
| `expired` | renew/rotate the leaf (and check intermediates too) |
| `unknown ca` only on curl/Go/Java | server chain missing intermediate — append it |
| `hostname mismatch` | reissue with the right SAN, or fix SNI/vhost |
| `no shared cipher` / version | align TLS version + cipher policy both ends |
| client cert rejected (mTLS) | check client cert validity + server's trusted CA list |
| dates fine but one client rejects | that client's clock is skewed — fix NTP |

## References

- [openssl s_client(1)](https://www.openssl.org/docs/man3.0/man1/openssl-s_client.html)
- [What's My Chain Cert? — incomplete chain debugging](https://whatsmychaincert.com/)
- [Mozilla TLS config generator](https://ssl-config.mozilla.org/)
- vault: `diagnostics/clock-skew-ntp.md`, `diagnostics/network-connectivity.md`, `systems/network/tcp-stack.md`
