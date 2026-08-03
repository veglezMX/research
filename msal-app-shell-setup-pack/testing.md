# Testing and Release Gates

Status: settled
Decisions: 0029 · inherits 0010, 0013, 0014 and all settled behavior topics
Sources: pack `12` · independent §18 ·
[MSAL Browser testing](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/testing) ·
[Playwright authentication](https://playwright.dev/docs/auth) ·
[Playwright browser projects](https://playwright.dev/docs/browsers) ·
[Playwright web server](https://playwright.dev/docs/test-webserver)

## Rule

No release is accepted from unit tests alone. It must prove package/version invariants,
security-sensitive pure logic, built SPA/nginx behavior, three-browser route/auth flows,
backend authorization, and a protected real-Entra smoke path. Test artifacts follow the
same no-token/no-identity rule as production telemetry.

## Design

Use the exact quality versions from `version-baseline`: Vitest/coverage `4.1.10`,
Playwright Test `1.62.0`, Testing Library React `16.3.2`, DOM `10.4.1`, user-event
`14.6.1`, jsdom `30.0.1`, Oxlint `1.76.0`, and `oxlint-tsgolint` `7.0.2001`.

### Required layers

1. **Static/package gates**
   - frozen pnpm install and unchanged lockfile;
   - `tsc --noEmit`, Oxlint/type-aware rules, dependency-cycle/private-import checks;
   - exactly one resolved MSAL Browser and React wrapper version;
   - build portal/bridge/child0/child1 independently;
   - verify bridge bundle excludes React/router/chrome and all bundles share the exact
     MSAL patch;
   - dependency/license/vulnerability review and container/config syntax checks.

2. **Unit/property tests**
   - runtime-config exact validation and per-app slice isolation;
   - one-PCA factory/bootstrap order and error mapping;
   - zero/one/multiple/active account resolution;
   - scope/resource URL pinning and promise deduplication;
   - continuation parsing, expiry, nonce/state binding, automatic-sign-in action
     combinations/one-attempt ceiling, and adversarial return paths;
   - bounded claims parser and opaque relay record rules;
   - 401/403/429/5xx retry table and one-replay ceiling;
   - BroadcastChannel schema/stale/self-message behavior;
   - telemetry allowlist/redaction and no-sensitive-field serialization.

   Global line/statement coverage is at least 90% and branch coverage at least 85%.
   Continuation, resource URL, challenge, channel-message and telemetry-redaction
   validators require 100% branch coverage plus malicious-input cases.

3. **React/package integration**
   - StrictMode still creates one PCA and one event subscription per document;
   - no protected route/loader runs before config, initialization, redirect handling and
     account resolution;
   - authenticated/unauthenticated/selection/interaction/bridge/error pages;
   - account switch clears all query/profile/domain memory;
   - official MSAL testing helpers such as `loadExternalTokens` stay in test support and
     never production code.

4. **API/contract tests**
   - exact audience, tenant/issuer, expiry, scope/role and domain policy;
   - wrong audience and direct child access fail at the backend;
   - 401 performs no mutation; 403 does not trigger auth retry;
   - portal application discovery plus independently owned child0/child1 profile,
     capability and denial contracts;
   - compliant claims challenge plus five-minute, single-use, subject/resource-bound
     relay behavior;
   - idempotency for declared non-idempotent operations.

5. **Built-artifact and nginx tests**
   - direct/deep/unknown routes for all three apps;
   - API and missing assets never return HTML;
   - HTML/config/bridge/signed-out/immutable cache policies;
   - normal CSP/COOP/frame denial and the bridge's `frame-ancestors 'self'`, no COOP,
     no `DENY`;
   - no inline/eval requirement under enforcing CSP;
   - Trusted Types directives present — in the report-only header until the `0034` gate
     passes, in the enforcing header after — with the `trusted-types` allowlist equal to
     the release-generated policy list;
   - normalized access logs exclude query/header/body data;
   - current and retained release chunks load through blue/green switch;
   - readiness/liveness and runtime ConfigMap replacement.

6. **Playwright at the local single origin**
   - Chromium, Firefox and WebKit projects use `http://localhost:4173`;
   - portal → child0 → child1 full navigations and deep-link refresh;
   - same-client cached account visible after each document bootstrap;
   - direct unauthenticated child deep link → portal `ssoSilent` → exact child return;
   - no Entra session, multiple Entra sessions, and blocked silent iframe each perform at
     most one automatic portal attempt before interactive sign-in or stable recovery;
   - multiple accounts route to portal selection;
   - interaction continuation returns to the exact validated child route;
   - malicious/expired/missing-storage continuation falls back safely;
   - bridge success, missing/blocked bridge, timeout and no-loop behavior;
   - simultaneous silent acquisition/renewal in three documents;
   - claims silent replay, interactive relay continuation, expired/reused handle;
   - multi-tab logout, channel absence, offline server logout and session invalidation;
   - wrong-audience/403 UX and no auth retry;
   - no protected UI after account/logout signals.

### Identity test split

Deterministic unit/component/browser tests use explicit test adapters or MSAL's supported
test helpers; they verify application behavior but do not claim the real protocol works.
A protected non-production Entra tenant suite proves the real SPA registration, PKCE,
shared bridge, one client ID, same-origin cache, all three documents, silent renewal,
logout, consent, multiple-account policy, CAE claims challenge, and exact redirect URIs.

Run the real-Entra smoke before production release and nightly where tenant/rate policy
allows. Run a scheduled long-lived/soak scenario across the approximately 24-hour SPA
refresh-token boundary; adapter simulation alone is insufficient evidence for that edge.
Conditional Access/CAE scenarios run only in the dedicated tenant with identity-owner
control.

Playwright authentication state for this SPA contains tokens in localStorage. Never
commit it, upload it as a general CI artifact, or attach real-auth traces/HAR/video/screens
that can expose tokens or user identity. If protected automation needs ephemeral state,
store it only in an access-controlled short-lived worker directory, destroy it after the
job, and retain sanitized scalar results. Use synthetic test identities with no production
access/data.

### Release gates

Every pull request runs static, unit, integration, independent builds, built-header/route
tests and all three mocked browser projects. Pre-release additionally runs API contracts,
container/image/config scans, retained-asset rollout, and protected real-Entra smoke.
Production rollout is portal first (bridge owner), then child0, then child1, with
release-specific canary/smoke checks and rollback assets retained.

A failed bridge/header, duplicate MSAL, unsafe redirect, wrong-audience, sensitive-log,
real-Entra smoke, or cross-document cache test blocks release. A flaky security/auth test
is fixed or quarantines the release—not silently retried until green.

## Why not the alternatives

- **Mock-only identity tests** — rejected in `0029`; they cannot prove Entra
  registration/bridge/cache behavior.
- **Real-tenant tests only** — rejected in `0029`; they are slow, policy-dependent and
  poor at exhaustive malicious/error cases.
- **Chromium-only** — rejected in `0029`; iframe/storage/privacy behavior differs across
  engines.
- **Persist Playwright storage state/traces as normal artifacts** — rejected in `0029`;
  this architecture stores bearer material in browser storage.
- **One combined application build** — rejected in `0029`; it does not prove independent
  deployability or prefix isolation.

## Open

1. Implement the repository, test adapters, API test doubles, protected Entra tenant and
   CI secret/worker cleanup controls.
2. Pin Playwright browser/container and builder/runtime image digests.
3. Define test identities, Conditional Access policies, schedules, ownership and
   escalation for the 24-hour/CAE suites.
