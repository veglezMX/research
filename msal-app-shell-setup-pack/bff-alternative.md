# BFF Alternative

Status: settled
Decisions: 0003
Sources: analysis `01` §3 Option C · independent §23 ·
[Azure SPA token-handler architecture](https://learn.microsoft.com/en-us/azure/architecture/web-apps/guides/security/secure-single-page-application-authorization) ·
[Backends for Frontends pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/backends-for-frontends)

## Rule

MSAL and access tokens remain in the browser for this architecture. Do not introduce a
BFF/token-handler proxy, confidential client, or server-side token store as an
implementation detail: that is a different security and session architecture and must
supersede decision `0003`.

## Design

Portal, child0 and child1 each run one browser MSAL instance per loaded document and share
the same-origin cache. They call their APIs with bearer access tokens through the
resource-pinned adapter.

Accepted consequences:

- same-origin script execution in any SPA can act through the shared browser client;
- the fixed SPA refresh-token lifetime and top-level interaction recovery apply;
- cross-tab/account/logout behavior is browser-coordinated;
- CSP, exact dependencies, backend audience enforcement and browser tests are mandatory
  mitigations, not optional hardening.

The code may use ports/interfaces for testability and separation of concerns, but it must
not promise that swapping a token provider would convert this design into a BFF. A BFF
would replace browser token/account/cache/interaction behavior with a server session,
cookie/CSRF policy, proxy authorization, confidential credentials and server-side
operations.

Revisit this choice when either:

- security requires access/refresh tokens to be unreachable from page JavaScript; or
- an existing APIM/BFF platform, custom domain, confidential-client operations and proxy
  ownership make the server-side model the deliberate product architecture.

If revisited, evaluate HttpOnly cookie scope/SameSite, CSRF protection, session and key
rotation, multi-API token storage, logout, scaling, availability and proxy ownership
before superseding `0003`.

## Why not the alternatives

- **BFF/token handler now** — rejected in `0003`; it adds a confidential client and API
  proxy infrastructure the selected browser architecture deliberately excludes.
- **"Phase-two seam"** — rejected in `0003`; account, continuation, logout, storage and
  authorization flows change, so this is a redesign rather than a provider swap.
- **Hybrid browser tokens plus BFF cookies** — rejected; it keeps both threat/operations
  models without removing browser token exposure.

## Open

Nothing within the selected architecture. A future BFF is a decision supersession and
threat-model exercise, not unfinished implementation work.
