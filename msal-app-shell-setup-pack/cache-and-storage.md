# Cache and Storage

Status: settled
Decisions: 0017 · inherits 0003, 0004, 0006, 0013
Sources: pack `00` "localStorage security note" · independent §7 · analysis `01`,
analysis `02` §9 ·
[MSAL caching](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/caching) ·
[OWASP HTML5 storage guidance](https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html)

## Rule

MSAL uses `localStorage` so independently loaded same-origin documents can share its
cache. Application code treats that storage as MSAL-owned and opaque. No architecture
claim treats MSAL's cache encryption as protection from same-origin script execution.

## Design

The exact shared cache block is:

```ts
cache: {
  cacheLocation: "localStorage",
  cacheRetentionDays: 5,
}
```

All three documents receive the same values from runtime configuration. The five-day
old-schema retention window also defines the minimum rollback-artifact window after an
MSAL upgrade.

Storage allowlist:

| Location | Allowed content |
|---|---|
| MSAL-owned `localStorage` keys | only MSAL's cache |
| MSAL-owned `sessionStorage` keys | MSAL's temporary cache: in-flight interaction state and the bridge's raw redirect response, consumed by `handleRedirectPromise`; opaque to application code |
| application `sessionStorage` keys | one validated continuation record, maximum age ten minutes; one opaque random tab ID |
| memory | active `AccountInfo` inside the adapter, request promises, profiles and UI data |
| `BroadcastChannel` | payload-free session event, opaque source-tab ID, timestamp |

Application code must not enumerate, parse, copy, mutate, or delete individual MSAL
cache entries. Do not call `localStorage.clear()` or deploy `Clear-Site-Data` as a normal
logout mechanism because this origin hosts all three applications. Use MSAL logout and
clear application memory explicitly.

Under MSAL v4+, cache entries may be encrypted, but MSAL skips that encryption for users
who selected Keep Me Signed In. In either case, code executing on the origin can ask
MSAL for tokens; encryption is not an XSS boundary. The controls are strict CSP,
same-origin script discipline, exact dependency/lockfile review, no unreviewed
third-party runtime scripts, and prompt patching. A service worker is not part of the
baseline and requires its own threat model before introduction.

Reject removed or inapplicable configuration copied from older MSAL versions, including
`temporaryCacheLocation`, `claimsBasedCachingEnabled`, `cacheMigrationEnabled`,
`storeAuthStateInCookie`, and `secureCookies`; they are absent from the pinned v5 cache
type.

## Why not the alternatives

- **`sessionStorage` or memory for MSAL** — rejected in `0017`; independently loaded
  documents/tabs would not share durable authentication state.
- **Application-readable token mirrors** — rejected in `0017`; they create additional
  token stores without adding capability.
- **Rely on cache encryption as XSS mitigation** — rejected in `0017`; same-origin code
  can use the authenticated client even when bytes are encrypted.
- **Application local-storage logout marker** — rejected in `0017`; the baseline already
  has BroadcastChannel and lifecycle re-resolution without adding durable state.

## Open

1. The tenant identity owner chooses the Keep Me Signed In policy; the browser
   architecture does not control it.
2. Any future third-party script, service worker, or same-origin non-suite application
   requires a renewed shared-origin threat assessment.
3. Evaluate
   [device-bound tokens via the WAM platform broker](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/device-bound-tokens)
   (`system.allowPlatformBroker: true`). Where it applies — Windows with Chrome/Edge
   plus the Windows Accounts extension, work and school accounts — brokered tokens are
   device-bound and are never written to `localStorage`/`sessionStorage` at all, which
   removes exfiltrated-token replay from this topic's XSS blast radius; unsupported
   environments fall back to the web flows unchanged. Requires its own session-behavior
   review (broker-owned cache ignores `forceRefresh`/`cacheLookupPolicy`, distinct
   system prompt, `user_switch` error handling) before adoption.
