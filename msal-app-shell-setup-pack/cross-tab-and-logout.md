# Cross-Tab Authentication and Logout

Status: settled
Decisions: 0006, 0023 · inherits 0016, 0017, 0021
Sources: pack `07` · independent §18 E2E 9 · analysis `02` §6.2 ·
[MSAL logout](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/logout) ·
[MSAL events](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/events) ·
[BroadcastChannel](https://developer.mozilla.org/en-US/docs/Web/API/BroadcastChannel)

## Rule

Logout is portal-owned, redirect-based, and single-initiator. A same-origin
`BroadcastChannel` immediately invalidates protected UI in other documents without
carrying identity or token data. Other tabs never start their own Entra logout.

## Design

Every document opens `company.portal.auth.v1` after bootstrap and accepts only:

```ts
type SessionSignal = Readonly<{
  type: "logout-started" | "application-session-invalidated";
  sourceTabId: string; // opaque random sessionStorage value
  occurredAt: number;
}>;
```

Reject malformed, future, or stale messages and ignore the document's own source ID. The
channel carries no login/account-switch event: MSAL account events plus lifecycle
re-resolution remain the account state source.

Any child logout control performs a full navigation to `/logout`. The portal route:

1. resolves the active account;
2. broadcasts `logout-started`;
3. immediately blocks protected UI, aborts requests, and clears domain/query/profile
   memory;
4. calls `logoutRedirect({ account, postLogoutRedirectUri: "/signed-out" })`.

Pass the active account when one is unambiguous. If none is active but cached accounts
remain, omit `account` deliberately and sign out/clear all accounts for this shared
logical client. `/signed-out` is a static, MSAL-free portal page with a normal Sign In
link. It is registered as an exact redirect URI.

On either accepted signal, receiving documents:

- enter an in-memory signed-out-pending state before reading the cache;
- block protected UI and new authorized requests;
- abort/discard in-flight results;
- clear all account-bound application memory;
- re-run the deterministic account resolver on the next MSAL account event,
  `pageshow`, focus, or transition to visible;
- never call `logoutRedirect`, mutate MSAL cache entries, or navigate automatically.

`application-session-invalidated` is sent only after a terminal authentication outcome
such as the bounded second `401`; it coordinates local invalidation, not Entra server
logout.

If `BroadcastChannel` is unavailable, do not create an application localStorage event
bus. Use MSAL v5 account events, `pageshow`/focus/visibility re-resolution, API
authentication outcomes, and full-page navigation as the degraded fallback. The browser
support matrix must verify both channel and fallback paths.

MSAL clears local cache before navigating to the server logout endpoint. If that network
navigation fails, the portal still clears account-bound UI state and offers a normal
retry/sign-in path without claiming that the Entra server session ended. Front-channel
logout is optional defense in depth, not the baseline synchronization mechanism, because
modern browser framing/storage restrictions can limit it.

## Why not the alternatives

- **Every tab calls Entra logout** — rejected in `0023`; it creates redirect races and
  duplicate server requests.
- **Child calls `logoutRedirect`** — rejected in `0023`; all interaction is portal-owned.
- **Put account ID in the signal** — rejected by `0006`; the bus is notification-only.
- **Use localStorage as the fallback bus** — rejected in `0023`; application localStorage
  is forbidden by `0017`.
- **Popup logout** — rejected in `0023`; redirect logout has one exact MSAL-free
  post-logout page and fewer bridge variants.
- **Assume local cache clear proves server logout** — rejected in `0023`; network failure
  can leave the Entra session intact.

## Open

1. Product copy for local-complete/server-logout-uncertain and signed-out states.
2. Tenant/browser testing decides whether front-channel logout adds useful coverage.
