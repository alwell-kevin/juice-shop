# Localhost 3000 Recon Findings

- Target: `http://127.0.0.1:3000`
- Scope: unauthenticated black-box probing only
- Date: `2026-04-20`

## Summary

The application exposes multiple high-signal issues before authentication. An unauthenticated attacker can enumerate routes, browse sensitive directories, retrieve internal files, inspect operational telemetry, and quickly identify injectable behavior.

## Findings

### 1. Public metrics endpoint
- Severity: High
- Evidence:
  - `curl -skI http://127.0.0.1:3000/metrics`
  - `curl -sk http://127.0.0.1:3000/metrics | sed -n '1,25p'`
- Result:
  - Returns `200 OK`
  - Exposes Prometheus metrics including startup timings and process telemetry
- Risk:
  - Leaks operational detail useful for reconnaissance and attack tuning

### 2. Public API documentation
- Severity: Medium
- Evidence:
  - `curl -skI http://127.0.0.1:3000/api-docs`
  - `curl -sk http://127.0.0.1:3000/api-docs/ | rg 'Swagger UI'`
- Result:
  - API documentation is exposed without authentication
- Risk:
  - Makes endpoint discovery and payload shaping trivial

### 3. Directory listing enabled on sensitive paths
- Severity: High
- Evidence:
  - `curl -skI http://127.0.0.1:3000/ftp`
  - `curl -skI http://127.0.0.1:3000/encryptionkeys`
  - `curl -skI http://127.0.0.1:3000/support/logs`
- Result:
  - All three endpoints return `200 OK`
  - Listings are browsable in HTML
- Risk:
  - Unauthenticated attackers can enumerate exposed files and cherry-pick interesting targets

### 4. Sensitive file disclosure
- Severity: High
- Evidence:
  - `curl -sk http://127.0.0.1:3000/ftp/acquisitions.md | sed -n '1,8p'`
- Result:
  - Confidential file contents are returned without authentication
- Risk:
  - Direct information disclosure with no exploit sophistication required

### 5. Overly permissive CORS
- Severity: Medium
- Evidence:
  - `curl -skI -X OPTIONS http://127.0.0.1:3000/rest/products/search -H 'Origin: http://evil.test' -H 'Access-Control-Request-Method: GET'`
- Result:
  - Returns `Access-Control-Allow-Origin: *`
  - Allows broad methods cross-origin
- Risk:
  - Expands browser-based abuse surface and weakens origin isolation

### 6. Verbose SQL error leakage and probable SQL injection
- Severity: High
- Evidence:
  - `curl -skG http://127.0.0.1:3000/rest/products/search --data-urlencode "q=';" | sed -n '1,20p'`
- Result:
  - Returns `SQLITE_ERROR: near ";": syntax error`
- Risk:
  - Strong indicator of unsafely composed SQL
  - Also leaks backend technology and parser behavior to the client

## Commands Run

```bash
curl -skI http://127.0.0.1:3000/
curl -skI http://127.0.0.1:3000/api-docs
curl -skI http://127.0.0.1:3000/metrics
curl -skI http://127.0.0.1:3000/ftp
curl -skI http://127.0.0.1:3000/encryptionkeys
curl -skI http://127.0.0.1:3000/support/logs
curl -sk http://127.0.0.1:3000/ftp/acquisitions.md
curl -sk http://127.0.0.1:3000/api-docs/
curl -sk http://127.0.0.1:3000/metrics
curl -skI -X OPTIONS http://127.0.0.1:3000/rest/products/search -H 'Origin: http://evil.test' -H 'Access-Control-Request-Method: GET'
curl -skG http://127.0.0.1:3000/rest/products/search --data-urlencode "q=';"
```

## Notes

- This report intentionally stops at scan-only confirmation.
- It does not include credentialed testing, upload testing, SSRF, XXE, or exploitation of the probable SQL injection.
