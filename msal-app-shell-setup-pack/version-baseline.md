# Version Baseline

Status: settled
Decisions: 0013, 0014 · inherits 0007, 0032
Sources: pack `00` · independent §21 · analysis `02` §10 · `research.md` "Versions" and
§6 · npm registry queried 2026-07-29 ·
[MSAL Browser v4→v5 migration](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/v4-migration)
· [React security advisory](https://react.dev/blog/2025/12/03/critical-security-vulnerability-in-react-server-components)
· [nginx headers module](https://nginx.org/en/docs/http/ngx_http_headers_module.html)

## Rule

The repository pins one exact, compatible dependency set. All three applications, the
shared auth package, and the portal-owned redirect bridge resolve the same physical
`@azure/msal-browser` version. No caret/tilde range and no second MSAL copy is permitted in
the production lockfile.

Registry-verified baseline on **2026-07-29**:

| Package/tool | Exact baseline or constraint |
|---|---:|
| React | `19.2.8` |
| React DOM | `19.2.8` |
| `@azure/msal-browser` | `5.17.3` |
| `@azure/msal-react` | `5.5.4` |
| React Router | `8.3.0` |
| Vite | `8.1.5` |
| `@vitejs/plugin-react` | `6.0.4` |
| TypeScript | `7.0.2` |
| pnpm | `11.18.0` |
| Node.js | `>=22.22.2` |
| nginx | `>=1.29.3`; verified stable `1.30.4`, mainline `1.31.3` |

`@azure/msal-react@5.5.4` requires `@azure/msal-browser ^5.17.3`, Node `>=20`, and React
`^16.8.0 || ^17 || ^18 || ^19.2.1`. The chosen pair is compatible. React Router's Node
`>=22.22.0` requirement is the highest minimum in the direct frontend set and therefore
sets the application build floor. The test baseline raises it two patch releases to
`>=22.22.2` because `jsdom@30.0.1` requires that version on the Node 22 line.

Exact development-quality baseline, registry-verified on **2026-07-29**:

| Package | Exact version |
|---|---:|
| Vitest | `4.1.10` |
| `@vitest/coverage-v8` | `4.1.10` |
| Playwright Test | `1.62.0` |
| Testing Library React | `16.3.2` |
| Testing Library DOM | `10.4.1` |
| Testing Library user-event | `14.6.1` |
| jsdom | `30.0.1` |
| Oxlint | `1.76.0` |
| `oxlint-tsgolint` | `7.0.2001` |
| React types | `19.2.17` |
| React DOM types | `19.2.3` |
| Node 22 types | `22.20.1` |

## Design

### Exact dependency declarations

```bash
pnpm add --save-exact \
  react@19.2.8 \
  react-dom@19.2.8 \
  react-router@8.3.0 \
  @azure/msal-browser@5.17.3 \
  @azure/msal-react@5.5.4

pnpm add --save-dev --save-exact \
  typescript@7.0.2 \
  vite@8.1.5 \
  @vitejs/plugin-react@6.0.4
```

The root `package.json` pins the package manager:

```json
{
  "packageManager": "pnpm@11.18.0",
  "engines": {
    "node": ">=22.22.2"
  }
}
```

Shared workspace packages use `workspace:*` for internal dependencies. External
dependencies still resolve from the root lockfile to the exact versions above.

CI must fail when:

- `pnpm list -r @azure/msal-browser @azure/msal-react` finds more than one resolved
  version of either package;
- the lockfile changes without the dependency-update review;
- the portal bridge bundle and any application bundle contain different
  `@azure/msal-browser` versions;
- the build image's Node version does not satisfy `>=22.22.2`;
- the nginx image is older than `1.29.3` while the configuration uses
  `add_header_inherit merge`.

The quality toolchain is also exact:

```bash
pnpm add --save-dev --save-exact \
  vitest@4.1.10 \
  @vitest/coverage-v8@4.1.10 \
  @playwright/test@1.62.0 \
  @testing-library/react@16.3.2 \
  @testing-library/dom@10.4.1 \
  @testing-library/user-event@14.6.1 \
  jsdom@30.0.1 \
  oxlint@1.76.0 \
  oxlint-tsgolint@7.0.2001 \
  @types/react@19.2.17 \
  @types/react-dom@19.2.3 \
  @types/node@22.20.1
```

Use Oxlint with its built-in React, React Hooks, TypeScript, import, Vitest, and
accessibility rule sets. Enable type-aware rules through `oxlint-tsgolint`; keep
`tsc --noEmit` as a separate CI gate until Oxlint's type-check mode is no longer
experimental.

Do not add `typescript-eslint@8.65.0`: its registry peer range is
`typescript >=4.8.4 <6.1.0`, which excludes the settled TypeScript `7.0.2` baseline.

### Upgrade policy

1. Update the browser/react pair together after checking the wrapper's peer dependency.
2. Use an exact candidate set and regenerate the lockfile in one change.
3. Run unit, build, bridge-header, redirect, silent-token, cross-tab, and multi-application
   browser tests before rollout.
4. Roll out portal first because it owns the bridge, then child0, then child1. This is
   one coordinated three-application release with a single change window and a single
   rollback decision, not three independent deployments (`topology` invariant 10).
   Schedule it as a `portal-web` authentication window: while portal is mid-rollout,
   children cannot bootstrap, renew silently, recover interaction, or log out.
5. Keep the rollback artifact available for at least MSAL's default old-cache retention
   window (five days), unless `cache-and-storage` deliberately changes
   `cacheRetentionDays`.
6. Record the verification date. "Latest" is never a reproducible baseline.

### Relevant v5 migration guardrails

Code review and tests reject old examples containing:

- `PublicClientNext` or `PublicClientApplication.createPublicClientApplication`;
- `navigateToLoginRequestUrl` inside `auth`;
- `enableAccountStorageEvents()` / `disableAccountStorageEvents()`;
- `getAccountByUsername`, `getAccountByHomeId`, `getAccountByLocalId`, or `logout()`;
- `temporaryCacheLocation`, `claimsBasedCachingEnabled`, `storeAuthStateInCookie`,
  `secureCookies`, or `cacheMigrationEnabled`;
- `iframeHashTimeout`, `windowHashTimeout`, `loadFrameTimeout`, `navigateFrameWait`, or
  `asyncPopups`.

Use `createStandardPublicClientApplication`, `getAccount`, `logoutRedirect` /
`logoutPopup`, `iframeBridgeTimeout`, `popupBridgeTimeout`, and `navigatePopups` (whose
meaning is the inverse of the removed `asyncPopups`; default `true`).

`LOGIN_SUCCESS` carries `AccountInfo`, and a successful login also emits
`ACQUIRE_TOKEN_SUCCESS` with `AuthenticationResult`. Error control flow uses
`error.errorCode`, not message text; v5 error messages are documentation links and console
messages are hashed.

React versions `19.0.0` through `19.2.0` are excluded by the wrapper's peer range. The
React team's CVE-2025-55182 advisory rates the affected React Server Components issue
CVSS 10.0 and identifies `19.2.1` as the fixed 19.2 line. The selected `19.2.8` is above
that floor.

nginx's official module documentation confirms that `add_header_inherit` first appeared
in `1.29.3`; `merge` preserves inherited security headers when a location adds its own
cache headers.

## Why not the alternatives

- **Floating ranges** — rejected in `0013`; can produce bridge/application skew and
  unreviewed production changes.
- **`@azure/msal-browser@5.17.1` / `@azure/msal-react@5.5.3` from the received pack** —
  rejected in `0013`; the registry now provides `5.17.3` / `5.5.4`, and the latter's peer
  floor is browser `5.17.3`.
- **`@azure/msal-react@5.5.3` from `research.md`** — superseded by the registry check made
  immediately before curation.
- **Different MSAL versions per application** — rejected in `0013`; interoperability over
  the shared cache/bridge is not a supported compatibility contract.
- **nginx older than `1.29.3` with the current config** — invalid because
  `add_header_inherit merge` does not exist there.

## Open

1. The baseline is registry- and peer-range-verified, not yet installed and exercised in
   an implementation repository. `testing` (19) owns that proof.
2. Pin the exact Node container image digest and nginx image digest when the deployment
   repository is known.
3. Patch versions are fluid. Re-query the registry only in a controlled dependency-update
   session; do not edit this table opportunistically.
4. Currency check 2026-08-03 (registry query, no pins changed): `@azure/msal-browser`,
   `@azure/msal-react`, React/DOM, React Router, TypeScript, Vitest and jsdom remain the
   latest published versions. Drift observed: Vite `8.2.0`, pnpm `11.20.0`, Oxlint
   `1.77.0`, Playwright Test `1.62.1` — patch/minor only, no security advisory
   identified. Fold into the next controlled dependency-update session.
