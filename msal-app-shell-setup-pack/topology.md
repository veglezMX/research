# Topology

Status: settled
Decisions: 0002, 0003, 0004, 0006, 0030, 0032, 0033 · refined by 0007, 0011, 0031
Sources: pack `01`, `03`, `11` · independent §2, §23 · analysis `02` §1–§6, §11 ·
analysis `01` §3

## Rule

The system is **three separately built, separately imaged and separately deployed
single-page applications on one origin**, composed by navigation, not at runtime.
Independence is per-artifact, per-route and per-backend. It is **not** independence of
identity configuration, of the shared MSAL baseline, or of release coordination, and it
is not independence of availability — see invariant 10.

1. `apps/portal`, `apps/child0`, `apps/child1` are separate documents. Each ships its own
   `index.html`, its own bundle, and its own router.
2. `/child0/*` serves **child0's own `index.html`**. It must not serve portal HTML.
   Switching applications is a full document navigation.
3. Each loaded document creates **exactly one** `PublicClientApplication`. There is no
   exported global singleton and no shell-owned instance shared across applications.
   Three documents loaded across tabs means three instances, one per document.
4. Authentication state is shared through the **same-origin MSAL browser cache only**.
   Therefore all three applications must use the **same origin, the same Entra client ID,
   the same authority, and the same `cache` configuration**. Divergence in any of those
   four breaks sharing silently. This is an invariant, not a preference.
5. Tokens are never passed between applications through a JavaScript object, a
   `postMessage`, a `BroadcastChannel`, a URL, a cookie, or application state. An
   application that needs a token acquires it from its own MSAL instance.
6. Only the portal performs interactive authentication. Children are silent-only: no
   `loginRedirect`, `loginPopup`, `acquireTokenRedirect`, `acquireTokenPopup`, and no
   application-owned `handleRedirectPromise`. `MsalProvider`'s internal idempotent call is
   accepted and returns `null`; see `0011`.
7. Runtime configuration is fetched from a single origin-relative endpoint,
   `/portal-runtime.json`, whose top level is keyed by application id. **Each application
   reads only its own key.** No build-time environment baking; the shared auth package
   must not read `import.meta.env`.
8. MSAL stays in the browser. No BFF, no server-side token storage, no token-handler
   proxy — see decision `0003` for what that costs.
9. The portal owns **application discovery**, while each child SPA/backend pair owns its
   authorization decision. The portal also owns a payload-free **cross-tab logout
   signal**. Shared frontend source and identity configuration do not merge backend
   ownership.
10. **Independence has a fixed scope, and `portal-web` is tier-0.** Build, image,
    pipeline, route tree, router, 404s, backend and authorization contract are
    independent per application. Origin, Entra client ID, authority, `cache`
    configuration, the portal-owned bridge document, `/portal-runtime.json`, the pnpm
    lockfile and the single physical MSAL resolution are shared, so a change to any of
    them is a **coordinated three-application release**, not an independent one. Beyond
    release coupling, `portal-web` is an availability dependency of every application's
    authentication: it serves the bridge, `/auth/continue`, `/account/select`,
    `/login`, `/logout`, `/signed-out` and the runtime configuration. Its
    unavailability stops sign-in, silent renewal, interaction recovery and logout in
    child0 and child1, and blocks their startup entirely. `portal-web` is therefore
    operated at the highest availability tier in the suite; see `observability` (18).

## Design

### Document and route map

| Route | Serves | History fallback |
|---|---|---|
| `/` | portal `index.html` | portal |
| `/auth-redirect.html` | **dedicated redirect bridge document** — no router, no `MsalProvider` | none; exact path |
| `/auth/continue` | portal `index.html` | portal |
| `/child0/*` | child0 `index.html` | child0 |
| `/child1/*` | child1 `index.html` | child1 |
| `/portal-runtime.json` | runtime config, `Cache-Control: no-store` | none |

`/auth-redirect.html` is deliberately **not** a portal SPA route. Under
`@azure/msal-browser@5` the redirect response is delivered to a bare bridge page that
calls `broadcastResponseToMainFrame()`; it must not boot a router or an `MsalProvider`,
and it needs its own COOP handling. The independent source routes `/auth/callback`
through the portal SPA, which does not work under v5. Full treatment belongs to topic
`redirect-bridge` (5); this file only fixes that the bridge is a separate document.

Deep links work by construction: `/child0/projects/123` is child0's own route, resolved
by child0's router after nginx falls back to child0's `index.html`. No shell
`NavigationPort`, no shell-owned history.

For an unauthenticated direct deep link, ingress still serves the child document first.
The child initializes only enough runtime/MSAL state to resolve that no account is cached,
blocks protected rendering, stores the validated deep link in tab-local continuation
storage, and replaces the document with portal `/auth/continue`. The portal performs the
silent-first sign-in flow from `0031` and returns to the exact child route. Ingress never
serves portal HTML at a child path.

### Runtime configuration

One endpoint, top level keyed by application id. Each document reads only its own key and
validates the result before use.

```jsonc
// GET /portal-runtime.json      Cache-Control: no-store
{
  "portal":  { "auth": { /* clientId, authority, redirectUri, cacheLocation, cacheRetentionDays, timeouts */ },
               "apiBaseUrl": "/api/portal/v1",
               "resources": { "portal-api": { /* scopes */ } } },
  "child0":  { "auth": { /* identical clientId, authority, cache values */ },
               "apiBaseUrl": "/api/child0/v1",
               "resources": { "child0-api": { /* scopes */ } } },
  "child1":  { "auth": { /* identical clientId, authority, cache values */ },
               "apiBaseUrl": "/api/child1/v1",
               "resources": { "child1-api": { /* scopes */ } } }
}
```

Rules:

- The `auth` block must be identical across all three keys for `clientId`, `authority`,
  and the cache values (`cacheLocation`, `cacheRetentionDays`). Serve it from one source
  of truth; a per-key copy that drifts breaks invariant 4 with no error message. The
  transport slice is flat; `msal-instance-and-bootstrap` owns the exact shape and maps it
  onto MSAL's `Configuration`, where `cache` is a top-level sibling of `auth`.
- An application reads `config[myApplicationId]` and nothing else. Reading another
  application's key is a defect, catchable in review and in unit tests.
- The document contains **no secrets**. It is publicly readable by anything running on
  the origin, and by anyone who opens the network tab. Client IDs, authorities, and scope
  URIs are not secrets; nothing else goes in.
- Fetched with `cache: "no-store"` and validated by a hand-written validator before the
  MSAL configuration is built. Config load failure is a hard startup failure, not a
  silent default.

### Entra registration shape

One shared SPA client registration, used by all three documents (required by invariant
4). Separate API registrations per backend: `portal-api`, `child0-api`, `child1-api`.
Detail belongs to topic `entra-registration` (3).

### Independence and coupling

State the boundary exactly; "three independent SPAs" on its own is misleading to an
operations or release owner.

| Concern | Independent per application | Shared, therefore coordinated |
|---|---|---|
| Source tree, router, routes, 404s | yes | — |
| Vite build, container image, Deployment, pipeline | yes | — |
| Product release cadence for app-local change | yes | — |
| Backend, audience, delegated scope, domain policy | yes | — |
| Origin, client ID, authority, `cache` config | — | yes (invariant 4) |
| `/auth-redirect.html` bridge document | — | yes, portal-owned (`0007`) |
| `/portal-runtime.json` | — | yes, portal-served (`0004`) |
| pnpm lockfile and exact MSAL resolution | — | yes (`0013`, `0027`) |
| Shared packages (`auth-*`, `session-sync`, `app-chrome`) | — | yes; a change rebuilds every dependent app |
| Availability of authentication | — | yes; depends on `portal-web` |

Consequences that must appear in the release and operations plan:

- An MSAL, bridge, runtime-config-shape, or shared auth-package change is one planned
  three-application rollout in the order portal → child0 → child1 (`version-baseline`),
  not three independent decisions.
- `portal-web` is the suite's tier-0 service. Child availability targets cannot exceed
  portal's, and a portal deployment window is an authentication window for all three
  applications.
- What independence actually buys here is **separate teams owning separate backends,
  route trees and product releases** on one shared browser client — not a decoupled
  frontend release train.

### Application discovery and child authorization

The portal uses `GET /api/portal/v1/me/applications` to build navigation, then performs a
full-document navigation to the selected child. It does not own child-specific
`applications/{id}` authorization endpoints. Child0 and child1 independently call
`GET /api/child0/v1/me` and `GET /api/child1/v1/me` during bootstrap and render their own
denial experience on `403`. A portal list entry and successful token acquisition are UX
or readiness signals, not entitlement proof. Each child backend remains authoritative.
Detail belongs to topic `authorization-layers` (12).

### Cross-tab logout

`BroadcastChannel` on a single named channel, carrying only an event name
(`logout-started`, `application-session-invalidated`) plus `sourceTabId` and `occurredAt`.
Never tokens, claims, names, emails, or roles. Only the initiating tab calls
`logoutRedirect`. Needed because a tab sitting idle on a child route otherwise learns
nothing about a logout until the user navigates. Detail belongs to topic
`cross-tab-and-logout` (10).

## Why not the alternatives

- **App-shell runtime composition** (one document, shell-owned MSAL, children `import()`ed
  and mounted into shell DOM, `/child0/*` serving shell HTML) — rejected in `0002`.
- **BFF / APIM token handler** (Microsoft's security-preferred Option C, no MSAL in the
  browser) — rejected in `0003`.
- **Per-app runtime config endpoints, or an edge that filters config by requester** —
  rejected in `0004`.
- **Portal-owned per-child entitlement endpoints** — rejected in `0030`; they couple
  independently deployed child authorization to the portal backend.
- **Treat the portal application list as entitlement proof** — rejected in `0030`; each
  child backend must make its own decision for direct and portal-originated navigation.
- **Relying on full page loads to self-heal logout state** — rejected in `0006`.

## Open

1. **The token boundary is soft, and backend audience validation does not close it.**
   Because child0's document runs its own MSAL instance against the shared cache with the
   shared client ID, nothing in the browser stops child0's bundle from requesting a
   `child1-api` token. Config scoping (`0004`) removes the *convenience* — child0 never
   receives child1's catalog — but not the *capability*. Two cases must be separated:

   - **Accidental cross-resource use** — a mistake in application code. Controlled by
     per-application runtime slices (`0004`), resource-pinned adapters (`0018`, `0019`),
     review, bundle tests, and the wrong-audience denial test. Backend audience
     validation (`0030`) is the backstop that turns the mistake into a clean `401`.
   - **Hostile same-origin code** — script running on the origin asks MSAL for a
     `child1-api` token and receives a token with the **correct** audience, scope and
     subject. Audience validation passes; `0030` is not a control for this case and must
     not be described as one. The controls are the same ones `0017` names for the storage
     consequence: strict enforcing CSP with no `unsafe-inline`/`unsafe-eval`/wildcards
     (`0026`), no unreviewed third-party runtime script, exact dependency and lockfile
     review with a single physical MSAL resolution (`0013`, `0027`), same-origin script
     discipline, and prompt patching. They reduce the probability of execution; none of
     them constrains what already-executing same-origin code may request.

   Structural enforcement is not achievable under this topology; it was available only
   under the app-shell pack's injected client and under the rejected BFF option.
2. **XSS blast radius accepted, and it is suite-wide.** Same origin plus tokens in
   `localStorage` plus one shared client ID means script execution in **any one** of the
   three applications can read the shared MSAL cache and mint tokens for **every** API.
   Three applications, one browser security boundary, one blast radius; a supply-chain
   compromise in a child0-only dependency reaches `portal-api` and `child1-api`.
   Identical under the app-shell topology; only the BFF option changes it. `0017` accepts
   the storage consequence and `0026` makes enforcing CSP and same-origin script
   discipline release requirements.
3. **`/portal-runtime.json` naming.** The endpoint is named for the portal but is consumed
   by all three applications. Kept as the user specified. If a fourth consumer or a second
   environment makes the name misleading, rename it there, not here.
