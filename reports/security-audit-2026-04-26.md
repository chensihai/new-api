# Security Audit Report

Date: 2026-04-26
Repository: new-api
Auditor: GitHub Copilot (GPT-5.3-Codex)

## Scope and Method

- Static audit across backend Go code and selected deployment configuration.
- Pattern-driven review for TLS, auth/session, secret exposure, SSRF, and logging risks.
- Manual validation on high-impact findings with file/line evidence.
- Automated scanners were attempted but not available in this environment:
  - gosec: not installed
  - govulncheck: not installed

## Executive Summary

- Critical findings: 0
- High findings: 2
- Medium findings: 3
- Low findings: 1
- Informational findings: 1

Primary risk themes:
- Trust bypass in TLS handling.
- Session cookie transport security.
- Sensitive OAuth data potentially logged.

---

## Findings

### 1) High: TLS certificate verification bypass is enabled via runtime flag and shared transport

Evidence:
- common/constants.go:77
- common/constants.go:78
- common/init.go:86
- common/init.go:90
- service/http_client.go:43
- service/http_client.go:114
- service/http_client.go:155

Details:
- The code supports globally disabling TLS verification (`TLS_INSECURE_SKIP_VERIFY`), and mutates shared transport settings to trust invalid certificates.
- This impacts outbound HTTP clients and increases MITM risk where TLS verification is expected.

Impact:
- Credential and API token interception risk in hostile network paths.
- Upstream response tampering risk.

Recommendation:
- Disallow insecure TLS in production builds or require explicit hard fail unless `ENV=dev`.
- Add startup guard: if insecure TLS is enabled in non-dev environment, abort startup.
- Prefer per-endpoint trust pinning / custom CA bundle instead of disabling verification.

---

### 2) High: SMTP TLS uses `InsecureSkipVerify: true` in both implicit TLS and STARTTLS branches

Evidence:
- common/email.go:62
- common/email.go:108

Details:
- SMTP TLS config explicitly skips certificate verification.
- Even with STARTTLS/SSL enabled, peer authenticity is not validated.

Impact:
- Email credentials and message content can be intercepted or modified by MITM.

Recommendation:
- Remove `InsecureSkipVerify: true` by default.
- Add config for trusted CA/custom root if needed for self-signed certs.
- If legacy compatibility is needed, gate insecure mode behind explicit development-only switch.

---

### 3) Medium: Session cookie `Secure` flag is hardcoded false

Evidence:
- main.go:176

Details:
- Session cookie settings set `Secure: false`, which allows cookie transmission over plaintext HTTP.

Impact:
- Session theft risk on non-TLS connections or misconfigured reverse proxies.

Recommendation:
- Set `Secure: true` when running behind HTTPS.
- Make this environment-configurable with secure-by-default behavior.
- Consider `SameSite=Lax/Strict` based on app flow (Strict is already set).

---

### 4) Medium: OAuth token endpoint responses are logged in debug mode

Evidence:
- oauth/generic.go:153

Details:
- Debug logging writes up to 500 chars of token response body.
- OAuth responses frequently include `access_token`, `refresh_token`, or `id_token`.

Impact:
- Token leakage into logs if debug mode is enabled.
- Secondary compromise risk from log access.

Recommendation:
- Redact token-bearing fields before logging.
- Avoid logging raw OAuth responses even in debug.
- Add structured redaction utility for sensitive keys.

---

### 5) Medium: CORS configuration allows all origins with credentials enabled

Evidence:
- middleware/cors.go:11
- middleware/cors.go:12

Details:
- `AllowAllOrigins = true` and `AllowCredentials = true` are configured together.
- Browsers treat wildcard + credentials specially and may reject or behave inconsistently.

Impact:
- Security policy ambiguity and potential misconfiguration when deployed behind proxies/CDNs.
- Increased attack surface if future changes alter origin behavior.

Recommendation:
- Use explicit allowlist origins for credentialed requests.
- Split public unauthenticated endpoints and authenticated endpoints with separate CORS policies.

---

### 6) Low: JWT claims are decoded without signature verification in helper path

Evidence:
- service/codex_oauth.go:303

Details:
- `decodeJWTClaims` parses payload directly from JWT segment without verifying signature.
- Used for extracting account metadata from access token.

Impact:
- If token source becomes attacker-controlled in any code path, claims could be spoofed.

Recommendation:
- Verify JWT signature and issuer/audience where feasible before trusting claims.
- If used only for display hints, clearly mark as untrusted metadata.

---

### 7) Informational: Development/deployment examples include default credentials

Evidence:
- docker-compose.yml (default DB/Redis passwords and examples)
- docker-compose-prod.yml (default DB/Redis passwords and examples)

Details:
- Compose examples contain obvious defaults (with warning comments).

Impact:
- Common operator error risk if defaults are used in internet-facing environments.

Recommendation:
- Add startup-time validation refusing known default secrets in production mode.
- Provide `.env` templating with required secret generation checks.

---

## Positive Security Controls Observed

- SSRF validation framework exists and is integrated into HTTP redirect checks and webhook/fetch flows.
  - common/ssrf_protection.go
  - service/http_client.go:24
- Redirect URL validation utility exists for trusted domains.
  - common/url_validator.go

## Prioritized Remediation Plan

1. Remove/lock down TLS verification bypasses (global HTTP and SMTP).
2. Set secure session cookies by default under HTTPS.
3. Redact/remove OAuth response body logging.
4. Replace permissive CORS with explicit origin allowlist where credentials are used.
5. Harden JWT claim parsing by verifying signatures where claims influence identity decisions.

## Residual Risk and Limitations

- This was a static audit, not a runtime penetration test.
- Dependency CVE analysis could not run because `gosec` and `govulncheck` are unavailable in this environment.
- Frontend and Electron security posture was not deeply tested beyond quick pattern checks.
