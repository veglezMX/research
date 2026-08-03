# nginx and Security Headers

Status: settled
Decisions: 0026, 0034 · inherits 0002, 0004, 0007, 0013, 0017, 0024, 0033
Sources: pack `11` · independent §2.1 · analysis `02` §6.5 ·
[nginx `try_files`](https://nginx.org/en/docs/http/ngx_http_core_module.html#try_files) ·
[nginx headers module](https://nginx.org/en/docs/http/ngx_http_headers_module.html) ·
[Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP) ·
[Trusted Types API](https://developer.mozilla.org/en-US/docs/Web/API/Trusted_Types_API) ·
[`require-trusted-types-for`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy/require-trusted-types-for) ·
[Kubernetes probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)

## Rule

Ingress terminates TLS and routes paths; each web container serves its own SPA fallback,
cache policy, and response headers. Normal application pages cannot be framed. The MSAL
bridge is the deliberate exception: it is same-origin-frameable and carries no COOP
header.

## Design

Use nginx `>=1.29.3` so `add_header_inherit` behavior is explicit and testable.

Common document policy:

```text
Content-Security-Policy:
  default-src 'self';
  base-uri 'none';
  object-src 'none';
  script-src 'self';
  style-src 'self';
  img-src 'self' data:;
  font-src 'self';
  connect-src 'self' https://login.microsoftonline.com;
  frame-src 'self' https://login.microsoftonline.com;
  frame-ancestors 'none';
  form-action 'self' https://login.microsoftonline.com;
  manifest-src 'self';
  worker-src 'none';
  require-trusted-types-for 'script';
  trusted-types <registered-policy-allowlist>;
  upgrade-insecure-requests
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Resource-Policy: same-origin
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: no-referrer
Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=(), usb=(), display-capture=()
```

Do not add `unsafe-inline`, `unsafe-eval`, wildcard script/connect sources, runtime
third-party scripts, or an unreviewed telemetry endpoint. When telemetry is selected,
add its one exact origin through environment-reviewed configuration.

This CSP is not hardening; it is a **primary control**. One origin, one Entra client ID
and MSAL's `localStorage` cache mean any script that executes here can mint tokens for
every API in the suite with a correct audience, which no backend check rejects
(`topology` Open 1, `authorization-layers`). Together with exact dependency/lockfile
review and the no-third-party-runtime-script rule, `script-src 'self'` is what keeps
that code from executing. Shipping the CSP in report-only mode, or relaxing it to
accommodate a product feature, removes a control the architecture depends on and needs
a security-owner decision, not a configuration change.

`script-src 'self'` controls where script loads from; it does not stop DOM-based XSS
inside already-allowed bundles. Trusted Types enforcement
(`require-trusted-types-for 'script'`, decision `0034`) closes that class: injection
sinks such as `innerHTML` reject plain strings and accept only values produced by a
policy named in the `trusted-types` allowlist. Trusted Types is Baseline across
Chrome, Edge, Firefox and Safari since February 2026; older browsers ignore the
directives, so enforcement is additive. The allowlist enumerates exactly the policies
the three applications and their dependencies register (including any policy MSAL or a
sanitizer library creates) and is a build-time input, never a wildcard. Roll-out runs
through `Content-Security-Policy-Report-Only` carrying only the Trusted Types
directives, alongside the enforcing base CSP — the base CSP itself never ships
report-only.

A host-allowlist CSP is acceptable here, against the general preference for
nonce/hash-based strict CSP, only because of conditions this origin must keep true:
every document and script is a static, release-built, same-origin asset; there is no
inline script, no JSONP-style endpoint, and no user-uploaded or user-controlled
content served from this origin. If any of those change, `script-src 'self'` stops
being a sufficient primary control and the CSP must move to nonces/hashes with
`strict-dynamic` under a new decision.

`/auth-redirect.html` has a separate minimal policy:

- `Cache-Control: no-store`;
- `frame-ancestors 'self'` and `X-Frame-Options: SAMEORIGIN`;
- no `Cross-Origin-Opener-Policy`;
- same-origin scripts only, no inline script;
- `default-src 'none'; script-src 'self'; connect-src 'self';
  frame-ancestors 'self'; base-uri 'none'; form-action 'none'`.

Generated-config tests must prove the bridge response lacks COOP and `DENY`; inheriting
either breaks MSAL v5. `/portal-runtime.json` and `/signed-out` are also `no-store`.

Cache/fallback policy:

| Response | Cache | Fallback |
|---|---|---|
| SPA `index.html` | `no-cache, max-age=0, must-revalidate` | app-owned `try_files` target |
| release-qualified hashed asset | `public, max-age=31536000, immutable` | `404` |
| `/auth-redirect.html` | `no-store` | none |
| `/portal-runtime.json` | `no-store` | none |
| `/signed-out` | `no-store` | none |
| missing/API path | appropriate JSON/`404` | never SPA HTML |

Representative child behavior:

```nginx
location = /child0 {
    return 308 /child0/;
}

location ^~ /child0/assets/ {
    try_files $uri =404;
    add_header Cache-Control "public, max-age=31536000, immutable" always;
}

location /child0/ {
    try_files $uri $uri/ /child0/index.html;
    add_header Cache-Control "no-cache, max-age=0, must-revalidate" always;
}
```

The production config uses `add_header_inherit merge` or generated complete location
blocks so cache headers do not accidentally remove security headers. Portal has exact
locations for bridge, runtime config and signed-out before its `/` fallback.

Ingress owns HTTPS redirect and HSTS:

```text
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

Add `preload` only after the domain owner verifies every subdomain is permanently HTTPS
and deliberately submits it. Services do not trust client-supplied forwarding headers;
the ingress overwrites and the application trusts only the cluster ingress hop.

Each container exposes a local `/healthz` for Kubernetes readiness/liveness probes.
Probes target the Pod/Service directly, not the public ingress, and the endpoint returns
no build/config/identity data.

Access logs use method, normalized `$uri` (not `$request_uri`), status, duration,
upstream/service, release ID, request ID, and trace ID only. Never log query strings,
cookies, authorization headers, redirect fragments, claims challenges, or response
bodies. APIs set their own content/cache/CORS policies; same-origin browser access needs
no permissive CORS.

## Why not the alternatives

- **Ingress-wide regex/rewrite SPA fallback** — rejected in `0026`; it can affect every
  host path and return HTML for APIs/assets.
- **One inherited header set including the bridge** — rejected in `0026`; COOP and
  frame-denial break the hidden bridge.
- **Long-cache HTML** — rejected in `0026`; it strands clients on removed chunks/config.
- **CSP with `unsafe-inline`/wildcards by default** — rejected in `0026`; it weakens the
  principal XSS mitigation of the shared-origin/localStorage design.
- **Log `$request_uri`** — rejected in `0026`; it includes query strings.
- **`script-src` allowlist alone, no Trusted Types** — rejected in `0034`; it leaves
  DOM-based XSS in allowed bundles uncovered now that Trusted Types is cross-browser
  Baseline.
- **Nonce/hash strict CSP now** — rejected in `0034`; static same-origin-only hosting
  makes `'self'` sufficient, and nginx-generated nonces would force per-request HTML
  rewriting of three SPA shells for no added control under the stated conditions.
- **`Cross-Origin-Embedder-Policy`** — deliberately absent, not an oversight:
  `require-corp` would block MSAL's hidden silent-renew and interactive frames to
  `login.microsoftonline.com`, which does not opt in, and nothing in the suite needs
  cross-origin isolation (`SharedArrayBuffer`).

## Open

1. Deployment owners must choose CSP report collection, telemetry origin, HSTS preload,
   and exact TLS/certificate policy.
2. Exact nginx/container image digests and ingress-controller version remain deployment
   inputs.
3. Product applications must prove they build without inline/eval requirements before
   the CSP can ship in enforcing mode.
4. Trusted Types enforcement ships only after the report-only trial proves all three
   applications, the pinned MSAL build, and the bridge document run violation-free, and
   the final `trusted-types` policy allowlist is generated from the release build
   (`0034`). Until then the two Trusted Types directives ride in the report-only
   header only.
