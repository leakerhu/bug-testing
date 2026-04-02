# Napkin.ai Advanced Bug Hunting Report

**Target:** napkin.ai  
**Date:** 2026-04-02  
**Type:** Active Recon / Security Testing (Passive Only — No Exploitation)  
**Scope:** Public surface via HTTP headers, DNS, CSP analysis, endpoint probing  

---

## Executive Summary

Active reconnaissance revealed **12 distinct security findings** ranging from information disclosure to potential subdomain takeover candidates. No authentication was bypassed, and no data was accessed beyond public endpoints.

---

## FINDINGS

---

### FINDING 1 — Subdomain Takeover Candidates (DNS NXDOMAIN)
**Severity:** High  
**Subdomains:**
- `openreplay.napkin.ai`
- `upload.api.napkin.ai`
- `download.api.napkin.ai`

**Evidence:**
```
HTTP/2 403
x-deny-reason: dns_nxdomain
```
All three subdomains are registered in Cloudflare DNS but point to non-existent backends. OpenReplay is a self-hosted session replay platform — if the backend instance was decommissioned without removing the DNS entry, these subdomains may be claimable depending on how the underlying infrastructure was provisioned.

**Impact:** An attacker who can claim the backend IP/service could serve content under `openreplay.napkin.ai`, potentially intercepting session replay data, conducting phishing, or serving malicious scripts that could be trusted due to the domain.

**Recommendation:** Remove Cloudflare DNS entries for decommissioned subdomains.

---

### FINDING 2 — `Napkin-E2E-Testing` Header Exposure
**Severity:** Medium–High  
**Affected:** `app.napkin.ai` (all routes)

**Evidence:**
```
access-control-allow-headers: ..., Napkin-E2E-Testing, ...
```
The `Napkin-E2E-Testing` header is explicitly listed in the CORS allowlist, exposing its existence to any browser request. This strongly suggests a testing/bypass mode exists in the backend.

**Testing result:** The header is accepted on all tested paths (returns HTTP 200 instead of redirecting).

**Impact:** If the backend conditionally skips security checks (auth, rate limiting, validation) when this header is present, it could be used by attackers to:
- Bypass authentication
- Disable rate limiting
- Skip input validation
- Access test/debug endpoints

**Recommendation:** Remove `Napkin-E2E-Testing` from the production CORS allowlist. E2E test mode should be gated by network-level controls (e.g., VPN/internal IP only), not HTTP headers.

---

### FINDING 3 — Three Sentry DSN Keys Exposed in HTTP Headers
**Severity:** Medium  
**Location:** `Content-Security-Policy: report-uri` headers

**Exposed DSNs:**
| Project ID | Sentry Key | Source |
|------------|-----------|--------|
| `4509394000412752` | `8c7fa57aa690f47f5eb2d7bc0e07284e` | `www.napkin.ai` CSP |
| `4510153873424464` | `ce29e886d585d66f9b58cbd560ef8247` | `api.napkin.ai` CSP |
| `4510153924345936` | `158c1b7d8374cba545b52e74920d38e5` | `app.napkin.ai` (internal) CSP |

**Evidence:**
```
POST https://o4507804332654592.ingest.de.sentry.io/api/4509394000412752/security/
     ?sentry_key=8c7fa57aa690f47f5eb2d7bc0e07284e
→ HTTP/2 200 (accepts submission)
```

**Impact:**
- An attacker can flood all three Sentry projects with fake error reports
- This causes alert fatigue, obscures real bugs, and may trigger billing overages
- Internal project structure (three separate projects) is disclosed
- Org ID `o4507804332654592` is leaked

**Recommendation:** CSP `report-uri` DSNs should use separate ingest-only keys with no access to the Sentry organization dashboard. Consider using `report-to` with a proxy endpoint instead.

---

### FINDING 4 — `info.napkin.ai` Wildcard CORS + Data Disclosure
**Severity:** Low–Medium  
**Affected:** `https://info.napkin.ai`

**Evidence:**
```http
GET https://info.napkin.ai?v=&c=<timestamp>
Response: {"ntp":{"c":1775147829,"s":1775147830000},"v":1775146674,"r":200,"cc":"US"}
Access-Control-Allow-Origin: *
```

**Details:**
- Endpoint returns: client timestamp (`c`), server timestamp (`s`), version hash (`v`), country code (`cc`)
- `v` field reveals the current build/deployment version hash
- Wildcard CORS means any website can read this data cross-origin
- Error messages disclose parameter names: `{"error":"Invalid parameter 'c'"}`

**Impact:** Version hash disclosure could assist attackers in tracking deployments, correlating build times, or confirming if a specific patched version is deployed. Cross-origin readable by any attacker site.

**Recommendation:** Restrict CORS to `*.napkin.ai` origins, remove version hash from public response.

---

### FINDING 5 — `analytics.napkin.ai` Wildcard CORS on Live Express Server
**Severity:** Low–Medium  
**Affected:** `https://analytics.napkin.ai`

**Evidence:**
```http
GET https://analytics.napkin.ai/health
Response: {"status":"ok"}
Access-Control-Allow-Origin: *
x-powered-by: Express
```

**Details:**
- Express.js server running, health endpoint unauthenticated
- Wildcard CORS allows any origin to read responses
- Server technology disclosed via `x-powered-by: Express`

**Recommendation:** Remove `x-powered-by` header, restrict CORS origin, require auth on all analytics endpoints.

---

### FINDING 6 — Infrastructure Information Disclosure via Response Headers
**Severity:** Low  
**Affected:** `app.napkin.ai`

**Disclosed information:**
| Header | Value | Risk |
|--------|-------|------|
| `x-napkin-location` | `us-central1-f` | GCP zone disclosed |
| `x-napkin-mode` | `web-only` | Operational mode disclosed (implies other modes exist) |
| `x-powered-by` | `Express` | Server stack |
| `x-goog-storage-class` | `STANDARD` | GCS storage class |
| `x-goog-generation` | `<timestamp>` | File creation timestamp |
| `via` | `1.1 google` | GCP backend confirmed |

**Recommendation:** Strip internal headers at the Cloudflare/proxy layer before they reach clients.

---

### FINDING 7 — `dev.api.napkin.ai` — Development Environment Confirmed
**Severity:** Low–Medium  
**Affected:** `https://dev.api.napkin.ai`

**Evidence:**
```
HTTP/2 403 (Cloudflare WAF blocking)
cf-ray: 9e61491bde8f115c
```

**Details:** The subdomain exists and is protected by Cloudflare WAF. A development API being internet-routable (even if WAF-protected) increases attack surface. If WAF rules are misconfigured or bypassed, the dev environment could expose pre-production features or relaxed auth.

**Recommendation:** Dev environments should not be publicly routable. Use internal DNS + VPN access only.

---

### FINDING 8 — CSP `unsafe-inline` + `unsafe-eval` in `script-src`
**Severity:** Medium  
**Affected:** `www.napkin.ai`, `app.napkin.ai`

**Evidence (from CSP):**
```
script-src 'self' 'unsafe-inline' 'unsafe-eval' 'wasm-unsafe-eval' blob: ...
```

**Impact:** `unsafe-inline` and `unsafe-eval` effectively neuter CSP's XSS protection. If an XSS vulnerability exists, CSP will not block script execution.

**Recommendation:** Replace with nonce-based or hash-based CSP. Required for Flutter web apps but should be mitigated with strict routing and DOM sanitization.

---

### FINDING 9 — `wss://echo.websocket.org` in CSP
**Severity:** Low  
**Affected:** `app.napkin.ai` CSP

**Evidence:**
```
connect-src: ... wss://echo.websocket.org ...
```

**Details:** A public echo WebSocket server (`echo.websocket.org`) is whitelisted in the production CSP. This is typically a development/testing endpoint that should never appear in production. Any script running in the context of `app.napkin.ai` can open a WebSocket connection to this public echo server.

**Impact:** In a chained XSS attack, an attacker's injected script could use the permitted `echo.websocket.org` connection to exfiltrate data (CSP would not block it since it's whitelisted).

**Recommendation:** Remove `wss://echo.websocket.org` from the production CSP immediately.

---

### FINDING 10 — `nlp-california-api.napkin.ai` — Internal NLP API Exposed
**Severity:** Low  
**Affected:** `https://nlp-california-api.napkin.ai`

**Evidence:**
```
GET / → {"message":"requires login"}
GET /api/generate → 403 {"message":"requires login"}  
GET /generate → 429 "Please contact support@napkin.ai"
```

**Details:** An internal NLP processing API is internet-accessible. Two distinct response patterns suggest:
- `/api/*` prefix = real authenticated API routes
- Other paths = rate-limited differently (429 with contact email)

The endpoint name discloses geographic deployment (`california` = us-west GCP region).

**Recommendation:** Move internal APIs behind a private VPC/service mesh. If public access is required, ensure all paths return consistent error codes (avoid disclosing path structure via different error codes).

---

### FINDING 11 — `CORS access-control-allow-credentials: true` with Empty ACAO
**Severity:** Informational  
**Affected:** `app.napkin.ai` backend

**Evidence:**
```http
access-control-allow-credentials: true
access-control-allow-origin: (empty)
```

**Details:** When an unlisted origin is sent, the server returns `access-control-allow-credentials: true` alongside an empty `access-control-allow-origin`. While modern browsers do not honor the empty ACAO value, this is a misconfiguration that could have unintended behavior in edge cases or non-browser clients.

**Recommendation:** Only set `access-control-allow-credentials: true` when also setting a non-wildcard, explicitly allowed origin.

---

### FINDING 12 — `X-Next-Cursor` Header Disclosed in CORS expose-headers
**Severity:** Informational  
**Affected:** All `app.napkin.ai` API routes

**Evidence:**
```
access-control-expose-headers: content-type, etag, X-Napkin-Workspace-Id,
  X-Napkin-Version-Name, X-Napkin-Version-Code, X-Napkin-Version-Tag, X-Next-Cursor
```

**Details:** `X-Next-Cursor` is exposed cross-origin — this is a pagination cursor header. Combined with an IDOR vulnerability, cross-origin reads of paginated API responses could leak data sequentially.

`X-Napkin-Workspace-Id` being in the exposed headers also discloses workspace identifiers to cross-origin JavaScript.

---

## Complete Subdomain Inventory

| Subdomain | Status | Notes |
|-----------|--------|-------|
| `www.napkin.ai` | Live | Marketing site |
| `app.napkin.ai` | Live | Main Flutter SPA |
| `api.napkin.ai` | Live | API docs (GCS hosted) |
| `help.napkin.ai` | Live | Intercom help center |
| `info.napkin.ai` | Live | Country/version endpoint, wildcard CORS |
| `vdp.napkin.ai` | Live | VDP submission portal |
| `ctm-app.napkin.ai` | Live | GTM/analytics |
| `analytics.napkin.ai` | Live | Express analytics server, wildcard CORS |
| `nlp-california-api.napkin.ai` | Live (auth) | Internal NLP API |
| `assets.napkin.ai` | Live (GCS) | Asset storage bucket |
| `fonts.napkin.ai` | Live (GCS) | Font storage bucket, wildcard CORS |
| `ping.napkin.ai` | Live (GCS) | Pong health page |
| `api.tool.napkin.ai` | Live (404) | Tool API |
| `import.api.napkin.ai` | Live (WAF) | Import API |
| `export.api.napkin.ai` | Live (WAF) | Export API |
| `dev.api.napkin.ai` | Live (WAF) | Dev environment |
| `auth.api.napkin.ai` | Live (GCP 404) | Auth service |
| `events.api.napkin.ai` | Live (WAF) | Events API |
| `openreplay.napkin.ai` | **NXDOMAIN** | Subdomain takeover candidate |
| `upload.api.napkin.ai` | **NXDOMAIN** | Subdomain takeover candidate |
| `download.api.napkin.ai` | **NXDOMAIN** | Subdomain takeover candidate |

---

## Priority Report Checklist (VDP Submission Order)

| Priority | Finding | Expected Severity |
|----------|---------|------------------|
| 1 | Subdomain takeover: openreplay / upload.api / download.api | High |
| 2 | `Napkin-E2E-Testing` header in production CORS allowlist | Medium–High |
| 3 | Three Sentry DSNs exposed + accepting fake submissions | Medium |
| 4 | `wss://echo.websocket.org` in production CSP | Medium |
| 5 | `info.napkin.ai` wildcard CORS + version disclosure | Low–Medium |
| 6 | `analytics.napkin.ai` wildcard CORS + unauthenticated `/health` | Low–Medium |
| 7 | `dev.api.napkin.ai` internet-accessible | Low–Medium |
| 8 | Infrastructure header disclosure (GCP zone, mode, version) | Low |
| 9 | CSP `unsafe-inline`/`unsafe-eval` | Medium |
| 10 | `nlp-california-api` internal API exposed | Low |

---

## Tools Used

- `curl` — HTTP header analysis, endpoint probing, CORS testing
- HTTP response analysis — CSP parsing, header enumeration
- DNS resolution — subdomain availability checking
- Passive OSINT — WebSearch, sitemap analysis, robots.txt

**No authentication bypass, no data access, no exploitation performed.**
