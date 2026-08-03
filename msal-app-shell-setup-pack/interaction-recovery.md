# Interaction Recovery

Status: settled
Decisions: 0021, 0031 · inherits 0008, 0009, 0016, 0020
Sources: pack `09` · independent §13, §15.4 · analysis `02` §5.2, §5.5 ·
[MSAL interaction errors](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/errors) ·
[MSAL Browser sign-in and `ssoSilent`](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/login-user) ·
[OWASP redirect guidance](https://cheatsheetseries.owasp.org/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.html)

## Rule

Silent failure never triggers interaction inside a child. It creates one short-lived,
tab-local continuation and performs a full navigation to the portal. For an initial
direct child visit with no cached MSAL account, the portal automatically attempts
`ssoSilent` and, if interaction is required, starts the portal-owned login redirect. For
a background/API recovery, the portal still asks for an explicit user action before
starting an interactive redirect.

## Design

The sole application continuation key is `workspace.auth.continuation` in
`sessionStorage`:

```ts
type ContinuationV1 = Readonly<{
  version: 1;
  nonce: string; // at least 128 random bits, base64url
  action: "sign-in" | "select-account" | "acquire-token";
  resourceId?: "portal-api" | "child0-api" | "child1-api";
  returnPath: string;
  challengeId?: string; // opaque relay handle, never raw claims
  automaticSignInAttempted: boolean;
  createdAt: number;    // epoch milliseconds
}>;
```

The serialized record is at most 2 KiB and lives for at most ten minutes. Validate exact
keys and types, nonce/handle syntax, clock skew, action/resource combinations, and the
return path on every read. A claims continuation must have `action: "acquire-token"`,
an allowed resource, a challenge ID, and `automaticSignInAttempted: false`. Only a
foreground `action: "sign-in"` continuation from an initial unauthenticated route may
change `automaticSignInAttempted` to `true`.

`returnPath` must:

- be at most 2,048 characters and begin with one `/`, not `//`;
- contain no control characters, backslashes, credentials, scheme, or invalid percent
  encoding;
- decode to a route owned by the portal router or to exact/prefix
  `/child0`, `/child0/`, `/child1`, or `/child1/`;
- never target `/api`, `/auth-redirect.html`, `/portal-runtime.json`, `/healthz`, or an
  absolute/protocol-relative URL;
- contain no token, authorization code, claims value, account identifier, email, or role.

The portal's compiled route manifest is the allowlist for portal-owned paths. Validation
failure, expiry, missing storage, or nonce mismatch falls back to `/` and emits only a
stable error code.

### Flow

1. The child captures and validates its current path, creates/replaces the continuation,
   and calls `window.location.replace("/auth/continue")`.
2. Portal bootstrap resolves any redirect result, active account, and continuation before
   rendering.
3. For `action: "sign-in"`, the portal first rechecks the shared MSAL cache. If there is
   still no account, it atomically sets `automaticSignInAttempted: true` and calls
   `ssoSilent` as an authentication-only request, without a child API resource scope. No
   account, username, or login hint is copied through application storage.
4. If that foreground silent SSO attempt returns an interaction-required or
   multiple-account outcome, the portal automatically calls `loginRedirect`. The user's
   direct navigation is the foreground intent; no second Continue click is required.
5. For `action: "acquire-token"` or other background/API recovery, `/auth/continue`
   attempts the requested operation silently. If interaction is still required, it
   renders a product page with a Continue button; only that action invokes an interactive
   MSAL API.
6. Pass custom MSAL `state` containing only the opaque continuation nonce and compare the
   returned user state after `handleRedirectPromise`.
7. On success, re-resolve the account, validate the record again, remove it from
   `sessionStorage`, then `window.location.replace(validatedReturnPath)`.
8. If the portal reloads with no authenticated account after
   `automaticSignInAttempted: true`, it stops and renders recovery; it never starts a
   second automatic cycle.
9. On cancellation or terminal failure, delete the record and render a stable portal
   recovery page.

`/login` is the direct sign-in entry, `/account/select` is the portal account chooser,
and `/auth/continue` owns interrupted work. An initial direct child visit is foreground
sign-in intent and may automatically progress from `ssoSilent` to portal
`loginRedirect`. An API/background failure must first show the continuation page. Only
one interaction may be in progress in a portal document.

`ssoSilent` is best-effort. It may fail when there is no Entra session, when Entra cannot
choose among multiple accounts, or when browser privacy controls block the hidden iframe.
Those outcomes do not lose the continuation and do not permit interaction inside a
child.

A `bridge-unavailable` result bypasses this flow and renders operational recovery
(retry after the bridge is healthy). It is never converted into an interaction.

If `sessionStorage` is unavailable, exact-route recovery is disabled: children show a
normal link to portal `/login`, and successful authentication lands at `/`. Never move
the continuation into a query string or localStorage as a fallback.

## Why not the alternatives

- **Child-owned redirect** — rejected in `0021`; it duplicates interaction and callback
  behavior across independently deployed apps.
- **Automatic redirect after any API failure** — rejected in `0021`; only an initial
  foreground child visit receives the bounded automatic flow defined by `0031`.
- **Call `ssoSilent` from the child** — rejected in `0031`; the portal owns all sign-in
  orchestration and redirect handling.
- **Return URL/action in the query string** — rejected in `0021`; URLs are logged and are
  a redirect-validation surface.
- **Rely solely on `navigateToLoginRequestUrl`** — rejected by `0008`; it cannot carry
  the recovery action/resource.
- **Treat bridge timeout as sign-in recovery** — rejected by `0009`.

## Open

1. Product content and accessibility design for interactive sign-in, continue,
   cancelled, expired, and bridge-unavailable pages.
2. The implementation route manifest must enumerate portal routes before the validator
   can be runtime-proven.
