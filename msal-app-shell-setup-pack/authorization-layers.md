# Authorization Layers

Status: settled
Decisions: 0030, 0033 · supersedes 0005, 0025; inherits 0015, 0016, 0019, 0031
Sources: pack `10` · independent §16.6, §17 · analysis `02` §6.1 ·
[Protected API authorization](https://learn.microsoft.com/en-us/entra/msidweb/authentication/authorization) ·
[Scope and role verification](https://learn.microsoft.com/en-us/entra/identity-platform/scenario-protected-web-api-verification-scope-app-roles) ·
[ID tokens](https://learn.microsoft.com/en-us/entra/identity-platform/id-tokens)

## Rule

Portal application discovery and child UI capabilities improve navigation and UX; they
never grant backend access. Each child SPA/backend pair owns its authorization contract.
Every API validates the token for itself and applies resource/domain authorization on
every protected operation.

## Design

Endpoint ownership follows the independently deployed backends:

| Owner | Endpoint | Purpose |
|---|---|---|
| Portal API | `GET /api/portal/v1/me/applications` | Portal navigation discovery only |
| Child0 API | `GET /api/child0/v1/me` | Child0 domain profile and UI capabilities |
| Child1 API | `GET /api/child1/v1/me` | Child1 domain profile and UI capabilities |

The portal does not expose
`/api/portal/v1/me/applications/child0` or
`/api/portal/v1/me/applications/child1`. Owning all three checks in the portal backend
would couple otherwise independent services and duplicate child authorization policy.

The portal list drives visible navigation and contains stable application codes and
internal paths, for example:

```json
{
  "applications": [
    {
      "applicationId": "child0",
      "path": "/child0/"
    }
  ]
}
```

It is a portal-owned discovery/visibility hint, not proof that the child will authorize
the account. Responses contain stable application/capability codes only—no names,
emails, group lists, or role names. Keep them in memory, never browser storage. The list
may be cached in memory for display for at most 60 seconds.

Launching from the portal and entering a direct child URL follow the same security path.
If a direct child bootstrap finds no cached account, it preserves the validated child
path and hands sign-in to the portal under `0031`; it does not call the child API yet.
After authentication returns to the child, the child calls its own API for a domain
profile/capability document. A valid but unauthorized account receives `403` from that
child API and the child renders its own denial experience. The backend may return boolean
or stable capability codes for UI gating, but the frontend never treats them as proof for
a later request. Every protected API operation rechecks authorization.

Each backend:

1. validates signature, trusted tenant/issuer, exact audience and time bounds;
2. distinguishes delegated `scp` from application `roles` and requires the intended
   permission type;
3. keys the subject by `tid` + `oid`, not display/email claims;
4. evaluates domain data ownership, assignment, policy, and revocation;
5. returns `401` for absent/invalid authentication and `403` for a valid identity lacking
   permission;
6. does not disclose whether a protected resource exists when that distinction would
   leak information.

ID tokens never authorize APIs. Frontends do not decode access tokens for capabilities.
Group/role claims are not copied into application state; group overage and directory
lookups are backend concerns.

Portal, child0, and child1 use the same tenant ID, tenant-specific authority, Entra SPA
client ID, and identical MSAL cache configuration (`topology` invariant 4). That shared browser identity
configuration does not make the portal backend a gateway and does not merge backend
ownership. Each API remains an independently deployed protected resource with its own
audience, delegated permission, and domain policy.

The shared client ID creates a deliberately soft frontend token boundary: same-origin
code may request another resource's scope. Separate the two cases.

*Accidental* cross-resource acquisition is application error. Per-app runtime slices,
resource-pinned adapters, code review, and bundle tests reduce it, and the receiving
backend's exact audience check turns anything that slips through into a clean `401`.

*Hostile* same-origin code is different: it asks MSAL for the other resource's scope and
receives a token with the correct audience, scope and subject, which every backend check
in this document then accepts. Audience and policy validation is therefore **not** a
control against same-origin script — it is a control against wrong-audience bugs,
token replay across resources, and callers outside the browser. The controls that apply
to hostile same-origin code sit in `nginx-and-headers` (17), `cache-and-storage` (13) and
`version-baseline` (20): an enforcing CSP with no `unsafe-inline`, `unsafe-eval` or
wildcard script/connect sources, no unreviewed third-party runtime script, exact
dependency and lockfile review with one physical MSAL resolution, same-origin script
discipline, and prompt patching. Do not present backend validation as closing the
frontend boundary; it bounds the damage of mistakes, not of code execution.

The portal backend team owns application discovery only. Each child backend team owns its
profile/capability endpoint and every domain authorization policy. The identity team owns
the shared SPA client registration and tenant configuration plus the separate API scopes,
roles, consent, and tenant assignment; none of those responsibilities belongs to ingress
or the React router. Keeping all frontend code in one repository does not change backend
endpoint ownership.

## Why not the alternatives

- **Token acquisition means application permission** — rejected by `0030`; a token is not
  a product entitlement decision.
- **Portal-owned per-child launch endpoints** — rejected in `0030`; they centralize and
  duplicate authorization that belongs to the independently deployed child backends.
- **Portal discovery is the security boundary** — rejected in `0030`; direct deep links
  and direct API calls bypass it.
- **Frontend-decoded roles/groups** — rejected in `0030`; they become stale, leak into
  state, and cannot replace API validation.
- **One audience for all APIs** — rejected in `0015`; it weakens resource isolation and
  ownership.
- **Describing exact-audience validation as the control for the soft frontend token
  boundary** — rejected in `0033`; hostile same-origin code obtains a correctly
  audienced token, so the control set is CSP, supply-chain and script discipline.
- **Return only 403 for every auth failure** — rejected in `0030`; the client needs the
  standard 401/403 distinction for bounded recovery versus denial.

## Open

1. Product/domain owners must define portal discovery rules and each child's access and
   per-operation policies.
2. The identity owner must choose delegated scopes versus app roles for every
   non-interactive caller; this browser suite uses delegated scopes.
3. API technology-specific validation middleware belongs in each implementation
   repository.
4. Sender-constrained tokens (DPoP, device-bound session credentials) are the structural
   answer to the exfiltrated-token half of the soft boundary: a stolen token stops working
   off-device. They are not adopted here because Entra + `@azure/msal-browser` offer no
   generally available browser-SPA path as of 2026-08 — the closest shipping option is
   the WAM platform broker evaluated in `cache-and-storage` Open 3. Re-evaluate when
   Entra exposes GA sender-constraining for SPAs; note they would still not stop hostile
   same-origin code from using the live client in place.
