# Entra Registration

Status: settled
Decisions: 0015 · inherits 0002, 0007, 0010, 0033
Sources: pack `02` · independent §3, §6 ·
[Redirect URI restrictions](https://learn.microsoft.com/en-us/entra/identity-platform/reply-url) ·
[Expose a web API](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-configure-app-expose-web-apis) ·
[Zero Trust application registration guidance](https://learn.microsoft.com/en-us/entra/identity-platform/zero-trust-for-developers)

## Rule

Within one environment, portal, child0, and child1 are three route owners of **one
logical browser client**. They use one tenant-specific Entra SPA registration and the
same client ID. This is also one shared authentication and browser-security boundary;
the applications must not be represented as unrelated clients merely to reuse a cache.
Consent, assignment and Conditional Access apply to that one client, and code executing
in any of the three documents can obtain tokens for all three APIs. Threat-model the
suite as one client with three route owners, not as three clients.

Every backend is a separate protected resource registration with its own Application ID
URI, delegated scope, audience, policy, and owner. Keeping the frontend applications in
one repository does not merge those backend registrations or their deployment ownership.

## Design

### Registrations

| Registration | Platform / purpose | Permission exposed |
|---|---|---|
| `portal-suite-web` | SPA client for all three documents | none |
| `portal-api` | protected API | `api://<portal-api-client-id>/portal.access` |
| `child0-api` | protected API | `api://<child0-api-client-id>/child0.access` |
| `child1-api` | protected API | `api://<child1-api-client-id>/child1.access` |

The SPA registration is single-tenant unless a product requirement explicitly selects
another audience. It uses authorization code with PKCE. Implicit ID/access-token grants
stay disabled, and no client secret or certificate is created for browser code.

Configure these exact SPA redirect URIs for production:

```text
https://portal.example.com/auth-redirect.html
https://portal.example.com/signed-out
```

Configure local development on its separate non-production registration:

```text
http://localhost:4173/auth-redirect.html
http://localhost:4173/signed-out
```

`/auth-redirect.html` is the portal-owned MSAL v5 bridge. `/signed-out` is a static,
MSAL-free landing page used by `logoutRedirect`. Redirect URIs are exact and
case-sensitive. Do not register child callback paths, wildcard paths, or production and
localhost URIs on the same registration.

Create a distinct SPA registration per environment. All three documents in an
environment still receive that environment's same client ID and authority through
`/portal-runtime.json`; production and non-production client IDs are expected to differ.

Pre-authorize/admin-consent the three delegated API scopes when tenant policy permits.
Incremental consent remains a recoverable portal-owned flow, not something a child SPA
may initiate.

### Resource enforcement

Each API validates:

- signature and signing key;
- tenant and issuer;
- its own exact audience;
- token time bounds;
- required delegated scope (`scp`) or application role (`roles`), as appropriate;
- domain authorization independently of token validity.

The stable backend subject key is tenant ID plus object ID (`tid`, `oid`), not an email,
display name, or mutable username. ID tokens are for the browser client and are never
accepted as API authorization.

### Deployment-owned values

The deployment repository must supply one tenant ID, one shared SPA client ID, three API
resource application IDs, the production host name, API owners, consent workflow, and
environment-specific `/portal-runtime.json`. They are identifiers and configuration, not
secrets.

## Why not the alternatives

- **One registration for unrelated applications** — rejected in `0015`; Microsoft
  guidance warns against collapsing unrelated applications. This architecture qualifies
  only because the three route owners are governed and threat-modeled as one client.
- **One registration for client and all APIs** — rejected in `0015`; it destroys clear
  resource audiences and ownership.
- **One SPA client per child** — incompatible with the requested same-client-ID cache
  sharing and settled topology.
- **Wildcard or per-child redirect URIs** — unnecessary and weaker than the one exact
  bridge URI.
- **A browser client secret** — impossible to keep confidential and invalid for a public
  SPA client.

## Open

1. Supply real tenant, client, scope, and host identifiers in the deployment repository.
2. Confirm the tenant's user-assignment, consent, and Conditional Access policies with
   the identity owner.
3. If the browser frontends ever gain independent governance or trust boundaries, replace
   the shared SPA registration with separate clients or a BFF. Independent backend
   ownership alone does not change the chosen browser-client topology.
