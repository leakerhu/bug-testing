# Napkin.ai Reconnaissance Report

**Target:** napkin.ai  
**Date:** 2026-04-02  
**Type:** Passive OSINT / Public Surface Recon  

---

## 1. Company Overview

| Field | Details |
|-------|---------|
| **Product** | AI-powered text-to-visual generation tool |
| **Founders** | Pramod Sharma, Jerome Scholler |
| **Investors** | Accel, CRV |
| **Users** | 7M+ worldwide |
| **Company Legal Name** | Second Layer, Inc. |

---

## 2. Discovered Subdomains

| Subdomain | Purpose |
|-----------|---------|
| `www.napkin.ai` | Main marketing site |
| `app.napkin.ai` | Main application (Flutter SPA) |
| `api.napkin.ai` | API documentation & developer portal |
| `help.napkin.ai` | Help center (Intercom-based) |
| `info.napkin.ai` | Server info / country detection endpoint |
| `vdp.napkin.ai` | Vulnerability Disclosure Program portal |
| `ctm-app.napkin.ai` | GTM/analytics endpoint (production) |

---

## 3. Technology Stack

| Layer | Technology |
|-------|-----------|
| **Frontend** | Flutter/Dart web app (main.dart.js) |
| **CDN / WAF** | Cloudflare (email obfuscation, font delivery) |
| **Analytics** | Google Tag Manager (GTM-WPRH3LT) |
| **Error Tracking** | Sentry (`window.__napkin_sentry_queue`) |
| **Text Editor** | Tiptap (via `text-web-lib`) |
| **Auth** | Custom `auth-web-lib`, Google SSO, Email/Password |
| **Storage** | IndexedDB (client-side), cookies, sessionStorage, localStorage |
| **Fonts** | Inter, Plus Jakarta Sans, Shantell Sans (via Cloudflare Fonts) |
| **AI Backend** | Proprietary (text-to-visual generation) |

**JS Bundle modules identified:**
- `util-web-lib` — utilities
- `a-web-lib` — analytics
- `text-web-lib` — Tiptap text editing
- `ai-web-lib` — AI functionality
- `auth-web-lib` — authentication
- `help-web-lib` — help system
- `font-web-lib` — font management
- `c-web-lib` — component library / cookie consent
- `files-processing-web-lib` — file handling

---

## 4. API Surface

### Base URLs
- **API docs:** `https://api.napkin.ai`
- **API v1:** `https://api.napkin.ai/v1/`

### REST Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/v1/visual` | Create visual generation request (async) |
| `GET` | `/v1/visual/:request-id/status` | Poll status of a visual request |
| `GET` | `/v1/visual/:request-id/file/:file-id` | Download generated file |
| `GET` | `/v1/oauth/authorize` | OAuth2 authorization endpoint |
| `POST` | `/v1/oauth/token` | OAuth2 token exchange / refresh |
| `POST` | `/v1/oauth/revoke` | OAuth2 token revocation |

### Authentication

- **Type:** HTTP Bearer Token
- **Header:** `Authorization: Bearer <token>`
- **Token generation:** Account Settings → Developers tab → `app.napkin.ai`
- Multiple tokens supported per account
- Compromised tokens can be revoked immediately

### OAuth2 Details

| Property | Value |
|---------|-------|
| **Scopes** | `user` (profile: email, name, ID), `generation` (create visuals) |
| **Grant types** | `authorization_code`, `refresh_token` |
| **Access token TTL** | 1 hour |
| **Refresh token TTL** | 30 days (single-use, rotated on each refresh) |
| **Auth code TTL** | 10 minutes |
| **CSRF protection** | `state` parameter (recommended) |
| **Token revocation** | RFC 7009 compliant (always returns 200 OK) |

### API Response Codes

| Code | Meaning |
|------|---------|
| 201 | Visual request created |
| 400 | Invalid request data |
| 401 | Missing/invalid token |
| 403 | Access denied (IDOR protection — user-scoped requests) |
| 404 | Resource not found |
| 410 | Resource expired (URLs expire 30 min post-generation) |
| 429 | Rate limit exceeded |
| 500 | Internal server error |

### Output Formats
- SVG (default), PNG, PPT
- Processing time: ~10–30 seconds (async polling model)
- File URLs expire **30 minutes** after generation

---

## 5. Application Endpoints

| URL | Notes |
|-----|-------|
| `https://app.napkin.ai/signin` | Login page (Google SSO + Email/Password) |
| `https://app.napkin.ai/invite` | Invite-only registration flow |
| `https://app.napkin.ai/page/create` | Page/project creation |

---

## 6. Robots.txt Findings

```
User-agent: *
Allow: /
Disallow: /archives/

# AI training crawlers explicitly blocked:
# ClaudeBot, GPTBot, Amazonbot + others

Content-signals: search=yes, ai-train=no
Sitemap: https://www.napkin.ai/sitemap.xml
```

**Notable:** Explicit EU Directive 2019/790 copyright reservation for AI training.

---

## 7. Sitemap URLs

From `sitemap.xml` index:
- `sitemap_home.xml` — marketing pages
- `sitemap_app.xml` — app endpoints (`/signin`, `/invite`)
- `sitemap_help_center.xml` — help articles
- `api.napkin.ai/sitemap.xml` — API doc pages (12 URLs)

---

## 8. Security Posture

### Positive Controls
- Cloudflare WAF + email obfuscation
- HTTPS/TLS enforced everywhere
- Bearer token auth on all API endpoints
- Data encrypted in transit and at rest
- User-scoped resource access (403 on cross-user access → IDOR mitigation)
- Single-use refresh token rotation
- Token revocation capability
- `state` parameter for OAuth CSRF protection
- VDP at `https://vdp.napkin.ai`
- `security.txt` present pointing to VDP

### Observations / Potential Attack Surface

| Area | Observation |
|------|------------|
| **VDP expiry** | `security.txt` expiration was April 1, 2026 — may be expired/outdated |
| **Flutter SPA** | Dart-compiled JS; harder to analyze but all logic runs client-side |
| **info.napkin.ai** | CORS-enabled (`mode: "cors", credentials: "omit"`), timestamp-based country detection |
| **GTM** | Google Tag Manager (GTM-WPRH3LT) — third-party script injection risk if GTM account compromised |
| **Sentry** | Error reporting active — misconfigured DSN could leak internal errors |
| **IndexedDB** | Can be disabled via `d_i_b` param — potential for DoS on auth/session state |
| **Script retry** | Up to 20 retries with 5s timeout — potential for client-side timing analysis |
| **File URL expiry** | 30-min window on generated file URLs — predictable expiry behavior |
| **API in dev preview** | Not intended for production use — may have relaxed security controls |
| **Async visual generation** | Race conditions possible on status polling endpoint |
| **invite-only endpoint** | `app.napkin.ai/invite` — worth testing for invite bypass/enumeration |

---

## 9. Contact Points

| Purpose | Contact |
|---------|---------|
| API support | api@napkin.ai |
| VDP / Bug reports | https://vdp.napkin.ai |
| Security.txt | https://www.napkin.ai/.well-known/security.txt |

---

## 10. Scope for Further Testing

- [ ] Enumerate `info.napkin.ai` parameters more thoroughly
- [ ] Test OAuth flow for open redirect in `redirect_uri`
- [ ] Test IDOR on `/v1/visual/:request-id/status` with other users' IDs
- [ ] Enumerate API versioning (v1 — is there a v0 or beta?)
- [ ] Check `api.napkin.ai/markdown-page` for exposed content
- [ ] Test rate limiting thresholds on `/v1/visual`
- [ ] Check GTM container (GTM-WPRH3LT) for sensitive data
- [ ] Test invite endpoint for user enumeration
- [ ] Analyze `main.dart.js` bundle for hardcoded secrets/endpoints
- [ ] Check Sentry DSN exposure in JS bundles
- [ ] Test file download endpoint for path traversal on `:file-id`
