# Routing and Deep Links

Status: settled
Decisions: 0024 · inherits 0002, 0007, 0010, 0021, 0031, 0032
Sources: pack `09`, `11` · independent §2.1, §15.2, §16.2 · analysis `02` §5.1 ·
[Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) ·
[Vite public base path](https://vite.dev/guide/build#public-base-path) ·
[React Router BrowserRouter](https://reactrouter.com/api/declarative-routers/BrowserRouter)

## Rule

The shared Kubernetes ingress routes an unchanged request path to exactly one owning
Service. Every SPA owns its own HTML, public base, router basename, 404 behavior, and
history fallback. Cross-application navigation is always a document navigation.

## Design

One host uses these Ingress paths:

| Path | `pathType` | Backend |
|---|---|---|
| `/auth-redirect.html` | `Exact` | `portal-web` |
| `/signed-out` | `Exact` | `portal-web` |
| `/portal-runtime.json` | `Exact` | `portal-web` |
| `/api/auth-challenges` | `Prefix` | `auth-challenge-relay` |
| `/api/portal` | `Prefix` | `portal-api` |
| `/api/child0` | `Prefix` | `child0-api` |
| `/api/child1` | `Prefix` | `child1-api` |
| `/child0` | `Prefix` | `child0-web` |
| `/child1` | `Prefix` | `child1-web` |
| `/` | `Prefix` | `portal-web` |

Kubernetes selects the longest matching path and prefers `Exact` over `Prefix` for an
equal path. Keep the explicit bridge/config/signed-out rows even though `/` also matches:
they document ownership and protect them from a future portal-router change.

Do not use ingress-nginx `rewrite-target`, `use-regex`, or configuration snippets. They
can change matching behavior for every path on the host. The child web servers receive
`/child0/...` and `/child1/...` unchanged.

### Application bases

| App | Vite `base` | React Router `basename` | Container document |
|---|---|---|---|
| portal | `/` | `/` | `/index.html` |
| child0 | `/child0/` | `/child0` | `/child0/index.html` |
| child1 | `/child1/` | `/child1` | `/child1/index.html` |

Child containers copy their build output beneath the matching prefix. Each redirects its
bare prefix (`/child0` → `/child0/`) and uses its own `try_files` fallback. Asset
locations return `404` on a miss and never fall back to HTML. API backends never return an
SPA fallback.

Within one application, use React Router links. Links to another application, logout,
or portal recovery use `<a href>` or `window.location.assign/replace`; a client router
must not pretend it owns another document.

Required behavior:

- refreshing `/child0/projects/123` returns child0 HTML, then child0 resolves the route;
- refreshing `/child1/settings` is initially served by child1, never by portal ingress;
  an unauthenticated child may then perform the explicit portal hand-off from `0031`;
- an unauthenticated visit to `/child0/projects/123?view=history` preserves the validated
  child path in tab-local continuation storage, navigates to portal `/auth/continue`,
  attempts portal-owned silent SSO, and returns to that exact route after authentication;
- unknown child routes render that child's product 404;
- unknown portal routes render portal 404;
- missing hashed assets and API paths return real 404/JSON errors, never HTML;
- `/auth-redirect.html` is a literal document and never passes through React Router.

### Independent rollout

Bundle assets use content hashes and a release-qualified path. Publish immutable assets
before switching HTML, retain assets referenced by the current and rollback releases,
and switch web traffic blue/green rather than mixing arbitrary HTML revisions. An
already-open page must still be able to lazy-load its release's chunks after a new
release. HTML is revalidated/no-cache; immutable assets may be cached for one year.

The deployment may satisfy retention with a same-origin artifact service or by carrying
the retained releases in the active web image, but the publish-assets-before-HTML and
old-chunk-availability contract is mandatory.

Rollout is independent per application only for application-local change. `portal-web`
additionally serves the bridge, runtime config and every interactive auth route, so its
blue/green switch is an authentication event for all three applications and needs a
readiness gate on `/auth-redirect.html` and `/portal-runtime.json` before traffic moves
(`topology` invariant 10, `observability` service tiers). A shared-package or MSAL change
is one coordinated portal → child0 → child1 rollout.

Local development reproduces the same route map at `http://localhost:4173`; separate Vite
ports are upstream implementation details and never appear in browser URLs.

## Why not the alternatives

- **Ingress rewrite to `/`** — rejected in `0024`; it erases application ownership and
  makes deep-link behavior controller-specific.
- **Portal `index.html` for every route** — rejected by `0002`; children are independent
  documents.
- **React Router navigation across applications** — rejected in `0024`; it leaves the
  wrong document loaded.
- **One unqualified `/assets` namespace for all builds** — rejected in `0024`; independent
  releases can overwrite or strand each other's chunks.
- **Rolling HTML without old immutable assets** — rejected in `0024`; open pages can fail
  during lazy loading.

## Open

1. The deployment repository chooses the ingress class, production host, certificate
   issuer, and concrete retained-asset mechanism.
2. Product route manifests supply the complete portal/child 404 and continuation
   allowlists.
