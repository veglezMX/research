# External Sources

Primary external references backing the pack's version and security-practice claims.
One row per source: what it establishes, which curated topics cite it, and when it was
last verified against the live page. Workspace rule: every normative MSAL/Entra/nginx
claim cites a primary source or is marked unverified; this file is the consolidated
index of those citations. Update the *Verified* column only after actually re-reading
the source.

Verification dates: **2026-07-28/29** — original curation sessions. **2026-08-03** —
internet currency review (session `2026-08-03-xss-currency`, decision `0034`).

## MSAL.js — library behavior

| Source | Establishes | Cited in | Verified |
|---|---|---|---|
| [MSAL Browser caching](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/caching) | Cache locations; v4+ localStorage encryption skipped under KMSI; encryption reduces persistence, is not a security boundary; `cacheRetentionDays` | `cache-and-storage` | 2026-07-29 (claims re-confirmed 2026-08-03 via the repo `caching.md`) |
| [msal-browser `caching.md` (repo, dev)](https://github.com/AzureAD/microsoft-authentication-library-for-js/blob/dev/lib/msal-browser/docs/caching.md) | Encryption mechanics: AES-GCM/HKDF, key in `msal.cache.encryption` session cookie | `cache-and-storage` (background) | 2026-08-03 |
| [Migrate MSAL Browser v4 → v5](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/v4-migration) | Removed/renamed v5 config and APIs; the guardrail reject-lists | `msal-instance-and-bootstrap`, `redirect-bridge`, `version-baseline` | 2026-07-29 |
| [MSAL redirect bridge](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/redirect-bridge) | v5 bridge document contract, bridge timeouts | `redirect-bridge` | 2026-07-29 |
| [MSAL errors](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/errors) | `errorCode`-based control flow, hashed console messages | `token-acquisition`, `observability`, `interaction-recovery` | 2026-07-29 |
| [Login users](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/login-user) | Login/ssoSilent patterns | `account-resolution`, `interaction-recovery` | 2026-07-29 |
| [MSAL events](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/events) | Event payloads (`LOGIN_SUCCESS`, `ACQUIRE_TOKEN_SUCCESS`) | `account-resolution`, `cross-tab-and-logout` | 2026-07-29 |
| [Logout](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/logout) | `logoutRedirect`/`logoutPopup` | `cross-tab-and-logout` | 2026-07-29 |
| [Acquire tokens](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/acquire-token) | Silent-first acquisition contract | `token-acquisition` | 2026-07-29 |
| [Resources and scopes](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/resources-and-scopes) | Resource-pinned scope requests | `token-acquisition` | 2026-07-29 |
| [Token lifetimes (MSAL.js)](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/token-lifetimes) | SPA renewal behavior | `token-lifetime-24h` | 2026-07-29 |
| [Performance](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/performance) | Telemetry callback surface | `observability` | 2026-07-30 |
| [Testing](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/testing) | Test guidance | `testing` | 2026-07-30 |
| [MSAL React getting started](https://learn.microsoft.com/en-us/entra/msal/javascript/react/getting-started) | `MsalProvider` contract | `msal-instance-and-bootstrap` | 2026-07-29 |
| [`MsalProvider` source @ `msal-react-v5.5.4`](https://github.com/AzureAD/microsoft-authentication-library-for-js/blob/msal-react-v5.5.4/lib/msal-react/src/MsalProvider.tsx) | Internal idempotent `handleRedirectPromise`; pinned to the release tag, verified to exist | `msal-instance-and-bootstrap` | 2026-08-03 |
| [Device-bound tokens via WAM](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/device-bound-tokens) | `allowPlatformBroker`; brokered tokens never enter web storage; Windows/Chrome/Edge, work-school only | `cache-and-storage` Open 3, `authorization-layers` Open 4 | 2026-08-03 |

## Microsoft identity platform / Entra

| Source | Establishes | Cited in | Verified |
|---|---|---|---|
| [Access tokens](https://learn.microsoft.com/en-us/entra/identity-platform/access-tokens) | Audience/claims validation duties | `authorized-http`, `token-lifetime-24h` | 2026-07-30 |
| [Refresh tokens](https://learn.microsoft.com/en-us/entra/identity-platform/refresh-tokens) | 24-hour SPA refresh-token lifetime | `token-lifetime-24h` | 2026-07-30 |
| [Claims challenges](https://learn.microsoft.com/en-us/entra/identity-platform/claims-challenge) | CAE claims-challenge handling | `authorized-http`, `cae-and-claims-challenge` | 2026-07-30 |
| [Continuous access evaluation](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation) | CAE semantics, `CP1` | `cae-and-claims-challenge` | 2026-07-30 |
| [Protected web API scope/app-role verification](https://learn.microsoft.com/en-us/entra/identity-platform/scenario-protected-web-api-verification-scope-app-roles) | Backend-authoritative validation | `authorized-http`, `authorization-layers` | 2026-07-30 |
| [ID tokens](https://learn.microsoft.com/en-us/entra/identity-platform/id-tokens) | ID tokens never authorize APIs | `authorization-layers` | 2026-07-30 |
| [Reply URL restrictions](https://learn.microsoft.com/en-us/entra/identity-platform/reply-url) | Exact redirect-URI matching | `entra-registration` | 2026-07-29 |
| [Expose web APIs](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-configure-app-expose-web-apis) | Per-API scope registration | `entra-registration` | 2026-07-29 |
| [Zero Trust for developers](https://learn.microsoft.com/en-us/entra/identity-platform/zero-trust-for-developers) | Registration hygiene | `entra-registration` | 2026-07-29 |
| [Secure SPA authorization (Azure Architecture)](https://learn.microsoft.com/en-us/azure/architecture/web-apps/guides/security/secure-single-page-application-authorization) | BFF/token-handler option Microsoft prefers for SPAs — evaluated and rejected with the user in `0003` | `bff-alternative` | 2026-07-28 |
| [Backends for Frontends pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/backends-for-frontends) | BFF pattern definition | `bff-alternative` | 2026-07-28 |

## Web platform security

| Source | Establishes | Cited in | Verified |
|---|---|---|---|
| [Content Security Policy (MDN)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP) | CSP directive semantics | `nginx-and-headers` | 2026-07-30 |
| [Trusted Types API (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/Trusted_Types_API) | DOM-XSS sink enforcement; **Baseline across Chrome/Edge/Firefox/Safari since 2026-02** | `nginx-and-headers` (`0034`) | 2026-08-03 |
| [`require-trusted-types-for` (MDN)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy/require-trusted-types-for) | Enforcement directive | `nginx-and-headers` (`0034`) | 2026-08-03 |
| [`trusted-types` directive (MDN)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy/trusted-types) | Policy allowlist directive | `nginx-and-headers` (`0034`) | 2026-08-03 |
| [OWASP HTML5 Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html) | Web-storage risk statement. Cited as the *risk acknowledgment* for the accepted localStorage blast radius (`0017`), not as endorsement — OWASP's general preference is to keep tokens out of web storage | `cache-and-storage` | 2026-07-29 |
| [OWASP Unvalidated Redirects Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.html) | Continuation-record validation rules | `interaction-recovery` | 2026-07-30 |
| [draft-ietf-oauth-browser-based-apps](https://datatracker.ietf.org/doc/draft-ietf-oauth-browser-based-apps/) | IETF ranking: BFF > token-mediating backend > browser-only client. Draft-27, RFC Editor queue as of 2026-08-03, not yet an RFC. The browser-only choice here is the ranked-last option, accepted knowingly in `0003` | `bff-alternative` context | 2026-08-03 |
| [BroadcastChannel (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/BroadcastChannel) | Same-origin signaling | `cross-tab-and-logout` | 2026-07-30 |
| [React CVE-2025-55182 advisory](https://react.dev/blog/2025/12/03/critical-security-vulnerability-in-react-server-components) | React `19.2.1` security floor | `version-baseline` | 2026-07-29 |

## Serving and platform

| Source | Establishes | Cited in | Verified |
|---|---|---|---|
| [nginx `try_files`](https://nginx.org/en/docs/http/ngx_http_core_module.html#try_files) | SPA fallback mechanics | `nginx-and-headers` | 2026-07-30 |
| [nginx headers module](https://nginx.org/en/docs/http/ngx_http_headers_module.html) | `add_header` inheritance; `add_header_inherit` since `1.29.3` | `nginx-and-headers`, `version-baseline` | 2026-07-30 |
| [Kubernetes probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/) | Readiness/liveness contract | `nginx-and-headers` | 2026-07-30 |
| [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) | Path routing | `routing-and-deep-links` | 2026-07-30 |
| [Vite base path](https://vite.dev/guide/build#public-base-path) · [server options](https://vite.dev/config/server-options) | Per-app base and dev proxy | `routing-and-deep-links`, `workspace-and-packages` | 2026-07-30 |
| [pnpm workspaces](https://pnpm.io/workspaces) · [pnpm docker](https://pnpm.io/docker) | Workspace/build layout | `workspace-and-packages` | 2026-07-30 |
| [React Router `BrowserRouter`](https://reactrouter.com/api/declarative-routers/BrowserRouter) | Router base behavior | `routing-and-deep-links` | 2026-07-30 |
| [W3C Trace Context](https://www.w3.org/TR/trace-context/) · [OTel URL semconv](https://opentelemetry.io/docs/specs/semconv/url/) | Propagation and URL-redaction conventions | `observability` | 2026-07-30 |
| [Playwright auth](https://playwright.dev/docs/auth) · [browsers](https://playwright.dev/docs/browsers) · [webServer](https://playwright.dev/docs/test-webserver) | E2E harness | `testing` | 2026-07-30 |
| npm registry (queried live) | Exact version pins; currency re-check 2026-08-03: MSAL/React/TS/router/Vitest/jsdom still latest, drift limited to Vite/pnpm/Oxlint/Playwright patch-minor | `version-baseline` | 2026-08-03 |

## Currency statement

As of **2026-08-03** the pack's security posture was checked against: the MSAL Browser
caching/encryption documentation, the IETF browser-based-apps draft (draft-27), MDN
Trusted Types Baseline status, MSAL device-bound-token support, and a live npm registry
query of every pinned package. Outcome: no settled design reversed; Trusted Types
enforcement added (`0034`); device-bound-token and sender-constrained-token evaluation
items opened; all MSAL-behavior claims re-confirmed. Next scheduled currency check
belongs to the first controlled dependency-update session (`version-baseline` Open 3–4).
