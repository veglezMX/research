# Curated Manifest

Index of every topic in the curated architecture. One row per topic. A session picks a
topic here, reads only the sources and decisions its row names, then writes
`curated/<topic>.md`.

Status vocabulary: `not-started` · `drafted` · `settled` · `revisit`.
Definitions in `../memory/START-HERE.md`.

Legend for the Sources column:
**P** = `../sources/app-shell-pack/` · **I** = `../sources/independent/independent-approach.md`
· **A** = `../analysis/`.

| # | Topic | Status | Decisions | Sources to read |
|---|---|---|---|---|
| 1 | `topology` | settled | 0002, 0003, 0004, 0006, 0030, 0032, 0033 · refined by 0007, 0011, 0031 | P `01`, P `03`, P `11` · I §2, §23 · A `02` §1–§6, §11 |
| 2 | `bff-alternative` | settled | 0003 | A `01` §3 Option C · I §23 |
| 3 | `entra-registration` | settled | 0015 · inherits 0002, 0007, 0010, 0033 | P `02` · I §3, §6 · Microsoft identity platform registration guidance |
| 4 | `msal-instance-and-bootstrap` | settled | 0011, 0012 · refined by 0031; inherits 0002, 0004, 0006, 0007, 0008, 0009 | P `04`, P `06` · I §7, §8, §15 · A `01` addendum, A `02` §7.3 · `research.md` §1–§5 |
| 5 | `redirect-bridge` | settled | 0007, 0008, 0009, 0010, 0011, 0012 · inherits 0032 | P `00` "MSAL v5 changes", P `05`, P `11` · I §12, §15 · A `01` addendum, A `02` §7.1–§7.2, §7.4 · `research.md` §1 |
| 6 | `account-resolution` | settled | 0016 · refined by 0031 | P `01`, P `06` · I §9 · A `02` §5.3, §10 · MSAL account/event guidance |
| 7 | `token-acquisition` | settled | 0018 · inherits 0004, 0009, 0011, 0016 | P `08` · I §10, §11 · A `02` §7.5 · MSAL token/error guidance |
| 8 | `authorized-http` | settled | 0019 · inherits 0004, 0018, 0030 | P `08` · I §16.4–§16.5 · A `02` §6.3, §7.5 · Microsoft protected API guidance |
| 9 | `interaction-recovery` | settled | 0021, 0031 · inherits 0008, 0009, 0016, 0020 | P `09` · I §13, §15.4 · A `02` §5.2, §5.5 · MSAL errors · OWASP redirect guidance |
| 10 | `cross-tab-and-logout` | settled | 0006, 0023 · inherits 0016, 0017, 0021 | P `07` · I §18 E2E 9 · A `02` §6.2 · MSAL logout/events · BroadcastChannel |
| 11 | `routing-and-deep-links` | settled | 0024 · inherits 0002, 0007, 0010, 0021, 0031, 0032 | P `09`, P `11` · I §2.1, §15.2, §16.2 · A `02` §5.1 · Kubernetes/Vite/React Router guidance |
| 12 | `authorization-layers` | settled | 0030, 0033 · supersedes 0005, 0025; inherits 0015, 0016, 0019, 0031 | P `10` · I §16.6, §17 · A `02` §6.1 · Microsoft protected API guidance |
| 13 | `cache-and-storage` | settled | 0017 · inherits 0003, 0004, 0006, 0013 | P `00` "localStorage security note" · I §7 · A `01`, A `02` §9 · MSAL caching · OWASP storage |
| 14 | `cae-and-claims-challenge` | settled | 0020 · inherits 0008, 0015, 0018, 0019 | A `01` · Microsoft CAE and claims-challenge guidance |
| 15 | `token-lifetime-24h` | settled | 0022 · inherits 0003, 0008, 0018, 0021 | A `01` · A `02` §9 · Microsoft token-lifetime guidance |
| 16 | `workspace-and-packages` | settled | 0027 · inherits 0002, 0004, 0007, 0010, 0013, 0024, 0032 | P `03` · I §4, §5, §6 · pnpm/Vite guidance |
| 17 | `nginx-and-headers` | settled | 0026, 0034 · inherits 0002, 0004, 0007, 0013, 0017, 0024, 0033 | P `11` · I §2.1 (thin) · A `02` §6.5 · nginx/CSP/Trusted Types/Kubernetes guidance |
| 18 | `observability` | settled | 0028, 0032 · inherits 0009, 0017–0020, 0023, 0026 | P `12` · I §20 · A `02` §5.4 · MSAL/W3C/OpenTelemetry guidance |
| 19 | `testing` | settled | 0029 · inherits 0010, 0013, 0014 and all behavior decisions | P `12` · I §18 · MSAL/Playwright guidance |
| 20 | `version-baseline` | settled | 0013, 0014 · inherits 0007, 0032 | P `00` · I §21 · A `01` addendum, A `02` §10 · `research.md` versions, §6 · npm registry · Oxlint docs |

## Ordering constraint

`topology` (1) and `bff-alternative` (2) gated the rest. **Both are settled as of
2026-07-28** — navigational composition, MSAL in the browser (`0002`, `0003`). Topics 4,
10, 11, 16 and 17 inherit from that choice; read `curated/topology.md` before curating any
of them.

Topic 5 (`redirect-bridge`) is settled (2026-07-28, refined 2026-07-29), and topics 3,
4, 6, 13 and 20 are now also settled from the verified `research.md` slice and primary
guidance. Topics 7, 8 and 14 are also settled with resource-pinned acquisition, one
bounded authentication replay, and a navigation-safe claims relay. Nothing is blocking.
Topics 9, 10 and 15 are settled with explicit portal continuation, on-demand renewal, and
single-initiator logout. Topics 11, 12 and 17 are settled with path-preserving ingress,
backend-authoritative access and service-owned headers. Topics 16, 18 and 19 complete the
curation with independently compiled packages, redacted operations evidence, and layered
release proof. All twenty topics are settled.

## Coverage note

Topics 14 and 15 were absent from both sources. They were added from primary Microsoft
CAE, claims-challenge and token-lifetime documentation.

## Later corrections

2026-08-03, from an architecture review of the settled pack. Neither correction changes a
chosen design; both correct how it was described.

- `0032` — the independence claim was overstated. Topics 1, 5, 11, 16, 18 and 20 now
  state the coupling scope explicitly and record `portal-web` as the suite's tier-0
  service.
- `0033` — the soft frontend token boundary was described as enforced by backend audience
  validation. Topics 1, 3, 12 and 17 now separate accidental cross-resource use from
  hostile same-origin code and name CSP, supply-chain and script discipline as the
  controls for the latter.

2026-08-03, from an internet review of current XSS guidance (session
`2026-08-03-xss-currency`). No settled design was reversed; one control was added and
currency was re-verified.

- `0034` — Trusted Types reached cross-browser Baseline in February 2026; the common CSP
  gains `require-trusted-types-for 'script'` plus an exact policy allowlist, ship-gated
  on a violation-free report-only trial (topic 17). Topic 13 gains a device-bound-token
  (WAM broker) evaluation item; topic 20 gains a dated registry currency note. The IETF
  browser-based-apps ranking (BFF preferred) was checked against `0003` — the browser-only
  choice stands as an informed, recorded decision.
- [`sources.md`](sources.md) now indexes the primary external sources backing the pack's
  version and security-practice claims, each with its verification date.
- Cross-file audit fixes, same session, no decision change: the runtime `auth` slice is
  flat everywhere and `cacheRetentionDays: 5` now actually rides the transport and the
  factory (topics 1, 4, 5, 13 previously disagreed on the shape and the factory dropped
  the retention value); the storage allowlist gains MSAL's own `sessionStorage`
  temporary-cache row (the bridge writes it); the bootstrap active-account path now
  cross-checks `getAllAccounts()` as topic 6 always specified; production and localhost
  redirect URIs are explicitly on separate registrations (topics 3 and 5 read as
  contradicting); topic 12 says "identical", not "compatible", cache configuration;
  COEP's absence is recorded as deliberate (it would break MSAL's frames to the
  authority); `display-capture=()` added to Permissions-Policy; topic 19 gains the
  Trusted Types header gate; the `MsalProvider` citation is pinned to the
  `msal-react-v5.5.4` tag instead of the moving `dev` branch.
