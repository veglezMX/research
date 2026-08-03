# Account Resolution

Status: settled
Decisions: 0016 · refined by 0031
Sources: pack `01`, `06` · independent §9 · analysis `02` §5.3, §10 ·
[MSAL account APIs](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/login-user) ·
[MSAL events](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/events)

## Rule

Account selection is deterministic and portal-owned. Code must never choose the first
cached account. `AccountInfo` remains inside the authentication adapter and is neither
mirrored into application state nor exposed as a general child-application contract.

## Design

After MSAL initialization and portal redirect handling, every document resolves:

1. If `getActiveAccount()` returns an account still present in `getAllAccounts()`, use it.
2. If there are no cached accounts, return `unauthenticated`.
3. If there is exactly one cached account, call `setActiveAccount(account)` and use it.
4. If there is more than one, return `selection-required`.

`setActiveAccount` is the sole persisted selection mechanism. Do not maintain a second
home-account ID in application storage. Account lookup and token acquisition pass the
resolved `AccountInfo` explicitly inside the adapter; domain code sees only:

```ts
type SessionStatus =
  | { kind: "unauthenticated" }
  | { kind: "selection-required" }
  | { kind: "authenticated" };
```

The portal may render MSAL-provided account labels on `/account/select` while the page is
live, but it must not persist or log those labels. A child that resolves
`selection-required` writes only the validated continuation record and performs a full
navigation to `/account/select`; it does not present its own selector.

A child that resolves `unauthenticated` must not render protected routes, call its API,
or invoke an interactive MSAL method. It writes a validated foreground sign-in
continuation whose `returnPath` is the exact current child route and replaces the document
with `/auth/continue`. The portal rechecks the shared cache, attempts `ssoSilent`, owns
any interactive fallback, and returns to the validated child path under `0031`.

Subscribe once to MSAL account events during bootstrap. Re-resolve on login success,
account added/removed, `pageshow`, and return to visible state. When the active account
changes, cancel or discard responses tied to the previous session and clear every
in-memory profile, permission, query, and API-response cache before protected UI resumes.
Do not infer the new account from event order; run the same resolver again.

## Why not the alternatives

- **First cached account wins** — rejected in `0016`; cache order is not an identity
  selection policy.
- **Persist a username or home-account ID in application storage** — rejected in `0016`;
  it duplicates MSAL state and expands sensitive browser state.
- **Expose raw `AccountInfo` through React context** — rejected in `0016`; it spreads
  identity attributes into child and application state.
- **Let each child select independently** — rejected in `0016`; selection and all other
  interaction belong to the portal.
- **Let an unauthenticated child render before portal hand-off** — rejected in `0031`;
  protected child loaders must not run before authentication is restored.

## Open

1. Product copy and visual design for `/account/select`.
2. Whether tenant policy prevents multiple cached accounts in production; the resolver
   remains required even if policy normally makes the branch unreachable.
