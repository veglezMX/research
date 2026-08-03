# Token Lifetime and the 24-Hour SPA Boundary

Status: settled
Decisions: 0022 · inherits 0003, 0008, 0018, 0021
Sources: analysis `01`, analysis `02` §9 ·
[Refresh tokens](https://learn.microsoft.com/en-us/entra/identity-platform/refresh-tokens) ·
[MSAL token lifetimes](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/token-lifetimes) ·
[Access tokens](https://learn.microsoft.com/en-us/entra/identity-platform/access-tokens)

## Rule

The suite accepts the non-extendable approximately 24-hour refresh-token lifetime for
redirect-URI type `spa`. It renews on demand through MSAL, never by application token
parsing or a scheduled page reload. When silent renewal cannot continue, the portal
continuation flow restores the exact validated route after user interaction.

## Design

MSAL owns access-token, refresh-token, and hidden-iframe decisions. Applications:

- request tokens only when an authorized API operation needs one;
- use `forceRefresh: false` normally;
- never read or persist refresh tokens;
- never decode token expiry to drive UI authorization or timers;
- do not force renewal merely because another SPA document loaded;
- handle revocation/Conditional Access at any time, not only at 24 hours.

Access tokens are normally much shorter-lived than the SPA refresh token. MSAL can replace
them from cache/refresh mechanisms. When the SPA refresh token reaches its fixed lifetime,
MSAL may obtain a new authorization response silently if the Entra session and browser
conditions allow it. Browser privacy controls, session expiry, or policy can instead
produce `interaction-required`; that is normal recovery, not data loss.

On `interaction-required`:

1. preserve only the validated continuation intent;
2. navigate to `/auth/continue`;
3. try silent once in the portal to account for cache changes during navigation;
4. show the explicit Continue Session action if interaction is still required;
5. redirect through the portal-owned bridge;
6. return to the exact route after consuming the continuation.

Do not show a countdown to "session expiration": server revocation and CAE make any such
countdown misleading. UI may show an offline/re-authentication state only after a real
MSAL/API outcome. Long-running editors must save non-sensitive draft work independently
of authentication so a full navigation does not destroy user work.

Observe aggregate outcomes: silent success, cache/refresh path, interaction required,
bridge unavailable, cancellation, continuation success/expiry, and time-to-recovery.
Never record token timestamps, raw scope, account identifiers, or claims.

## Why not the alternatives

- **A 24-hour reload timer** — rejected in `0022`; it interrupts active work and does not
  model revocation or actual Entra session state.
- **Parse refresh/access tokens in application code** — rejected in `0022`; MSAL owns
  token cache/lifetime handling and APIs own authorization.
- **Force refresh in all three apps at startup** — rejected in `0022`; it manufactures
  cross-document renewal races.
- **BFF/session cookie solely to avoid the boundary** — already rejected in `0003`; it is
  a different architecture, not a small mitigation.

## Open

1. Test-tenant Conditional Access and sign-in-frequency policy determine how often users
   actually see the continuation UI.
2. Product teams must identify long-running/draft workflows that need independent
   save-before-navigation behavior.
