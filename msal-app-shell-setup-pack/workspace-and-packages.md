# Workspace and Packages

Status: settled
Decisions: 0027 · inherits 0002, 0004, 0007, 0010, 0013, 0024, 0032
Sources: pack `03` · independent §4, §5, §6 ·
[pnpm workspaces](https://pnpm.io/workspaces) ·
[pnpm Docker guidance](https://pnpm.io/docker) ·
[Vite server options](https://vite.dev/config/server-options)

## Rule

Use one pnpm workspace and lockfile for independently built/deployed SPAs and compile
shared packages into each application. No runtime remote/module federation, global PCA,
or shell-owned DOM. All browser URLs—including local development—use one origin and the
production route prefixes.

## Design

```text
/
├── apps/
│   ├── portal/       # portal SPA + auth-redirect.html build entry
│   ├── child0/       # Vite base /child0/
│   └── child1/       # Vite base /child1/
├── packages/
│   ├── runtime-config/
│   ├── auth-core/
│   ├── auth-react/
│   ├── authorized-http/
│   ├── session-sync/
│   ├── observability/
│   ├── app-chrome/
│   └── test-support/
├── deploy/
│   ├── local/        # local gateway/config only
│   └── nginx/        # shared generated fragments/templates
├── package.json
├── pnpm-workspace.yaml
└── pnpm-lock.yaml
```

Package responsibilities and direction:

| Package | Owns | May depend on |
|---|---|---|
| `runtime-config` | fetch, exact validation, app-slice selection | no MSAL/React |
| `auth-core` | PCA factory, account resolver, token outcomes, continuation | runtime-config, MSAL Browser |
| `auth-react` | bootstrap/provider/hooks and UI ports | auth-core, MSAL React, React |
| `authorized-http` | resource-pinned fetch, challenge parser/replay | auth-core, observability |
| `session-sync` | channel schema, lifecycle convergence, logout port | auth-core |
| `observability` | typed redacted events/tracing ports | no auth implementation |
| `app-chrome` | shared visual chrome/navigation components | React; receives data/actions as props |
| `test-support` | fakes/builders; never production exports | test-only dependencies |

Apps depend on packages; packages never depend on apps. `auth-core` does not import React,
HTTP domain clients, browser telemetry vendors, or app routes. `app-chrome` receives
session/capability models and callbacks rather than importing auth internals. CI rejects
cycles and imports through another package's private path.

`app-chrome` is duplicated into each bundle by design. It gives consistent navigation
and recovery UI, but full-page navigation means no persistent shell instance.

Be precise about what this workspace makes independent. Independent: source tree, Vite
build, image, Deployment, pipeline, and the release of any change local to one
application. Coordinated: anything in a shared package or the shared dependency set.
A change to `auth-core`, `auth-react`, `authorized-http`, `session-sync`, `app-chrome`,
`runtime-config`, the MSAL version, or common nginx policy rebuilds every dependent
application and ships as one planned rollout in the order portal → child0 → child1
(`version-baseline`, `topology` invariant 10). Applications may sit at different
versions of a shared package between steps of that rollout, but they must not be left
there: the shared MSAL resolution and the shared bridge are single-version contracts.
Independent deployment mechanics are real; an independent release train for shared auth
code is not.

The root pins `packageManager: pnpm@11.18.0`, Node `>=22.22.2`, the exact dependency
catalog from `version-baseline`, and overrides for the one allowed MSAL versions.
Internal dependencies use `workspace:*`. The lockfile is committed; CI installs with
`pnpm install --frozen-lockfile` and rejects more than one physical resolution of
MSAL Browser/React.

### Build and release

Each app has its own Vite config, test target, Dockerfile, image, Kubernetes Deployment
and release pipeline. Typical targets:

```bash
pnpm --filter @workspace/portal... build
pnpm --filter @workspace/child0... build
pnpm --filter @workspace/child1... build
```

A shared-package change builds/tests every affected dependent application; auth,
runtime-config, session-sync, app-chrome, or common nginx policy changes run all three.
The final image is static nginx only—Node and workspace source do not enter the runtime
layer. Record the source commit, lockfile digest, release ID, asset manifest, Node builder
digest and nginx runtime digest in provenance.

Portal's Vite build has two HTML entry points: normal `index.html` and the minimal
`auth-redirect.html`. Both resolve the exact same MSAL Browser package. The bridge does
not import React, the portal router, app chrome, or `MsalProvider`.

`/portal-runtime.json` is rendered/mounted into `portal-web` from a Kubernetes ConfigMap
or equivalent deployment data after image build. It is one public, secret-free document
consumed by all apps and is not baked into their bundles.

### Single-origin development

Docker Compose (or the repository's equivalent task runner) starts:

| Browser path | Local upstream |
|---|---|
| `/`, bridge, runtime JSON | portal Vite `:5173` |
| `/child0/*` | child0 Vite `:5174` |
| `/child1/*` | child1 Vite `:5175` |
| `/api/...` | local API stubs or declared dev backends |

A local nginx gateway exposes only `http://localhost:4173` to the browser, preserves
paths, proxies WebSocket upgrades for HMR, and uses the same exact/prefix precedence as
production. Vite binds its container/network interface with `strictPort: true`, uses the
application production `base`, and has an explicit `allowedHosts` list; never set
`allowedHosts: true`. Entra has only the one localhost bridge/signed-out redirect pair.

No app prints or links its upstream Vite port. Separate direct origins are debugging-only
and cannot be used to claim authentication, logout, bridge, or deep-link behavior works.

## Why not the alternatives

- **Runtime module federation/remotes** — rejected in `0027`; it changes settled
  navigation composition and introduces runtime version skew.
- **Copy shared auth/chrome source into each app** — rejected in `0027`; fixes cannot be
  reviewed and tested once.
- **One deploy artifact for all SPAs** — rejected in `0027`; it removes independent
  deployment.
- **Separate browser origins/ports in development** — rejected by `0010`; it invalidates
  cache/channel/bridge behavior.
- **A package-global PCA** — rejected in `0011`; there is one instance per loaded
  document, created during app bootstrap.
- **Build-time environment secrets/config** — rejected in `0027`; the runtime file is
  public configuration and images remain environment-agnostic.

## Open

1. Create the implementation repository and choose package scope/naming, container
   registry, CI system and deployment templates.
2. Pin exact builder/runtime image digests and prove the dependency graph/build commands.
3. Real API upstreams, tenant IDs, client IDs and release-specific runtime JSON remain
   environment inputs.
