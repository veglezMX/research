# Token Acquisition

Status: settled
Decisions: 0018 · inherits 0004, 0009, 0011, 0016
Sources: pack `08` · independent §10, §11 · analysis `02` §7.5 ·
[Acquire tokens with MSAL Browser](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/acquire-token) ·
[Resources and scopes](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/resources-and-scopes) ·
[MSAL errors](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/errors)

## Rule

Every API call is silent-first and account-explicit. Children never invoke an interactive
MSAL API. Access tokens remain inside the authentication/HTTP adapter and are never
returned to views, domain services, React state, storage, or telemetry.

## Design

Each application's validated runtime slice creates a closed resource registry:

```ts
type ResourceDefinition = Readonly<{
  resourceId: "portal-api" | "child0-api" | "child1-api";
  apiBaseUrl: URL;
  scopes: readonly [string, ...string[]];
}>;
```

An application receives only its own resource definition. One token request contains
scopes for one resource only; callers cannot provide arbitrary scopes or origins.

The internal provider always calls `acquireTokenSilent` with the resolved active
`AccountInfo`, the resource's sorted scopes, and any in-memory claims request:

```ts
type TokenOutcome =
  | { kind: "success"; accessToken: string; correlationId: string }
  | { kind: "unauthenticated" }
  | { kind: "selection-required" }
  | { kind: "interaction-required"; reason: "login" | "consent" | "claims" }
  | { kind: "bridge-unavailable"; errorCode: string }
  | { kind: "transient-failure"; errorCode: string }
  | { kind: "fatal-failure"; errorCode: string };
```

`success.accessToken` is private to the authorized HTTP package. Its type is not exported
from the package's public entry point.

Map `InteractionRequiredAuthError` and its login/consent subclasses to
`interaction-required`. Map the pinned v5 `BrowserAuthError.errorCode === "timed_out"`
and explicitly tested bridge failures to `bridge-unavailable`, never to
`interaction-required`; this preserves decision `0009` and prevents a broken bridge from
creating a redirect loop. Classify network/service errors as transient and invalid
configuration/unknown resource as fatal. UI renders stable product copy and a correlation
reference, never `error.message`.

### In-document concurrency

Deduplicate equivalent in-flight silent requests in each document. The key is:

```text
homeAccountId | resourceId | sorted scopes | claims SHA-256 | forceRefresh
```

The claims value itself stays in memory and out of the key/log. Remove the promise in a
`finally` block. Do not implement a cross-document lock: each PCA owns its request
coalescing, and MSAL/Entra remains authoritative across independently loaded documents.
The test matrix must exercise simultaneous renewal from all three applications.

Normal requests use `forceRefresh: false`. Only the authorized-HTTP recovery rules may
set it to `true`; page load and route changes must not force renewal.

## Why not the alternatives

- **Generic `getToken(scopes)` exported to applications** — rejected in `0018`; it makes
  the already soft cross-resource boundary easier to violate.
- **Interactive fallback inside the token provider** — rejected in `0018`; children are
  silent-only and surprise redirects lose application context.
- **Treat every timeout as interaction-required** — rejected by `0009`; a broken bridge
  would loop.
- **Cross-document mutex in localStorage** — rejected in `0018`; it adds fragile
  application state without an upstream locking contract.
- **Force refresh on every page load** — rejected in `0018`; it creates unnecessary
  renewal races and load.

## Open

1. Exact transient-error allowlists are implementation-tested against the pinned MSAL
   version; unknown codes fail closed as `fatal-failure`.
2. Browser validation of concurrent silent renewal is owned by `testing`.
