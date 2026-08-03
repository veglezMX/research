# CAE and Claims Challenge

Status: settled
Decisions: 0020 · inherits 0008, 0015, 0018, 0019
Sources: analysis `01` ·
[Claims challenge protocol](https://learn.microsoft.com/en-us/entra/identity-platform/claims-challenge) ·
[Continuous Access Evaluation](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation)

## Rule

Advertise CAE capability only with complete end-to-end claims-challenge handling.
Challenges may exist in process memory, but their raw header/value never enters a URL,
browser storage, application state, logs, or telemetry. A short-lived server-side relay
converts navigation-spanning challenges into opaque, single-use handles.

## Design

Once the relay and API behavior below are deployed, use:

```ts
auth: {
  // existing shared settings
  clientCapabilities: ["CP1"],
}
```

An owned API that must challenge a CAE-capable token:

1. validates the incoming token and its `xms_cc` capability;
2. creates Microsoft's standards-compliant `401` Bearer
   `WWW-Authenticate` challenge with `error="insufficient_claims"` and base64-encoded
   claims;
3. stores the raw claims request in the central claims relay through authenticated
   service-to-service transport;
4. includes an opaque 128-bit-or-stronger `X-Auth-Challenge-Id`;
5. performs no protected mutation before returning `401`.

The relay record contains challenge value, expected client ID, resource ID, subject
binding (`tid` + `oid`), creation/expiry time, and consumed time. It expires within five
minutes and is single-use. Resource APIs authenticate to the relay with workload
identity or mTLS. Browser retrieval is a same-origin `POST` with the opaque ID in its
body; the endpoint checks `Origin`, client/resource binding, expiry, and single use,
returns `Cache-Control: no-store`, and never logs request bodies. The handle is a
capability: generate it cryptographically, keep it only in the ten-minute continuation
record, and consume/delete it promptly.

In the child document, the HTTP adapter strictly parses a bounded challenge (maximum
8 KiB), confirms Bearer + `insufficient_claims`, decodes valid base64 JSON, and permits
only the expected access-token claims object. It first calls `acquireTokenSilent` with
that claims value and `forceRefresh: true`. If successful, it replays once. If interaction
is required, it discards the raw value, stores only the opaque relay ID in the
continuation, and navigates to the portal.

The portal consumes the continuation, retrieves the raw claims once into memory,
re-validates the bounded shape, and passes it to silent acquisition. If MSAL still
requires interaction, the portal passes the same in-memory claims value to
`acquireTokenRedirect`; MSAL owns its protocol request state. After return, the portal
consumes the continuation and returns to the validated route. A repeated challenge or
missing/expired handle stops with a stable recovery page—never a loop.

The relay stores no access token, refresh token, session cookie, authorization code, user
profile, or API response and proxies no API request. It is therefore not the BFF/token
handler rejected by `0003`.

## Why not the alternatives

- **Enable `CP1` without challenge recovery** — rejected in `0020`; the client advertises
  a capability it cannot honor.
- **Put raw claims in `sessionStorage`, URL, or telemetry** — rejected in `0020`; it
  violates the workspace's sensitive-data rule.
- **Drop every challenge and ask the user to sign in again** — rejected in `0020`; it
  loses Conditional Access context and may repeat the same denial.
- **Let each child redirect interactively** — rejected in `0020`; interaction remains
  portal-owned.
- **Turn the relay into an API proxy/token store** — rejected in `0020`; that would
  silently reintroduce the rejected BFF architecture.

## Open

1. The deployment repository must provide the claims-relay service, persistence, workload
   identity, rate limits, and data-retention controls.
2. The identity/security team must validate the exact Conditional Access policies and
   challenge shapes in a dedicated test tenant.
3. Third-party resource APIs without an opaque relay integration need a separate
   navigation-safe design before they can be added.
