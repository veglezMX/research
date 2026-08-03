# Authorized HTTP

Status: settled
Decisions: 0019 · inherits 0004, 0018, 0030
Sources: pack `08` · independent §16.4–§16.5 · analysis `02` §6.3, §7.5 ·
[Access tokens](https://learn.microsoft.com/en-us/entra/identity-platform/access-tokens) ·
[Protected API scope and role verification](https://learn.microsoft.com/en-us/entra/identity-platform/scenario-protected-web-api-verification-scope-app-roles) ·
[Claims challenges](https://learn.microsoft.com/en-us/entra/identity-platform/claims-challenge)

## Rule

Each application has one resource-pinned HTTP adapter. It injects a token only for that
resource's exact same-origin API base path and owns all authentication retry logic.
Backend audience and permission validation is the security boundary; frontend resource
pinning is defense in depth.

## Design

Construct the adapter from the application's validated `ResourceDefinition`. Resolve
relative request paths against `apiBaseUrl`, then reject a changed origin or an escaped
base-path prefix before token acquisition. Domain callers provide method, relative path,
headers/body, abort signal, and optional idempotency key—never scopes or bearer tokens.

For each request:

1. Resolve the active account and acquire silently with `forceRefresh: false`.
2. Add `Authorization: Bearer <access-token>`, `Accept`, and W3C `traceparent`/correlation
   headers immediately before `fetch`.
3. Return a typed domain response for success.
4. On the first `401` with a valid claims challenge, make one silent request with the
   in-memory claims value and `forceRefresh: true`, then replay the HTTP request once.
5. On the first plain `401`, make one silent request with `forceRefresh: true`, then
   replay once.
6. If MSAL requires interaction, create the portal continuation. A claims flow carries
   only the relay's opaque challenge ID.
7. If the replay is also `401`, stop. Clear account-bound application memory, emit the
   stable failure event, and require an explicit portal recovery action.

The owned API contract says a request returning `401` has not performed its protected
mutation, so this one replay is safe. Non-idempotent endpoints also accept an
application-generated idempotency key where domain duplication is possible.

Status handling:

| Response | Authentication behavior |
|---|---|
| `401` valid claims challenge | claims-aware silent renewal, at most one replay |
| `401` no claims challenge | forced silent renewal, at most one replay |
| `403` | no token retry; render/return authorization denial |
| `429` | no token retry; obey bounded `Retry-After` in the domain client |
| `5xx` / network | no token retry unless MSAL acquisition itself failed transiently |

Do not retry a request merely because an access token is old. The API decides validity.
Do not decode an access token in the SPA to make authorization decisions. ID tokens are
never sent to APIs.

Never log the `Authorization` header, response `WWW-Authenticate` value, response body,
full query string, raw scope, account identifier, or claims request. Telemetry records
only allowlisted resource ID, route template, status/outcome, duration, release ID, and
opaque correlation identifiers.

## Why not the alternatives

- **Retry the same cached token** — rejected in `0019`; it is the no-op identified in the
  source review.
- **A global URL-to-scopes interceptor** — rejected in `0019`; a configuration mistake
  can attach a token to the wrong origin.
- **Decode tokens for frontend authorization** — rejected in `0019`; the API owns
  audience and authorization decisions.
- **Retry `403`, `429`, or `5xx` through authentication** — rejected in `0019`; those
  statuses do not mean the cached token should be replaced.
- **Unlimited replay** — rejected in `0019`; it hides persistent policy/configuration
  faults and risks duplicate domain work.

## Open

1. Each backend team must declare which non-idempotent operations require an
   idempotency-key contract.
2. External APIs without the curated claims-relay integration require a separately
   reviewed adapter; they cannot silently inherit this recovery behavior.
