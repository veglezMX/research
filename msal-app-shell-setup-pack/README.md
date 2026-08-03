# Multi-SPA Entra / MSAL Architecture

The decided architecture for authenticating three same-origin single-page applications
with Microsoft Entra ID and MSAL Browser. Twenty topic files, all settled. This file is
the entry point: **why this exists**, then **how to navigate it**.

---

## Why this exists

Two architectures for the same problem were authored independently — an app-shell pack
and an independent-approach document. They disagreed on the decisions that matter most:
whether the applications are composed at runtime or by navigation, whether MSAL runs in
the browser or behind a BFF, who owns the redirect URI, and where the authorization
boundary sits. Both were internally coherent. Neither could simply be adopted, because
picking one meant silently inheriting choices nobody had examined.

This folder is the merge. Each topic was decided once, against the two sources plus
primary Microsoft, OWASP, nginx, Vite, React Router, Playwright and Kubernetes
documentation, and the result written down as a rule that binds implementation. The
value is not that a design exists — it is that every rule has a recorded reason and a
recorded set of rejected alternatives, so a future reader can tell a deliberate
constraint from an accident.

### What the architecture describes

Three separately built, separately imaged, separately deployed SPAs on **one origin**,
composed by navigation rather than at runtime:

- `portal` — owns interactive sign-in, logout, account selection, application discovery,
  the shared redirect bridge, and `/portal-runtime.json`. Tier-0: while it is down, the
  children cannot bootstrap, renew, recover or log out.
- `child0`, `child1` — silent-only. Own their own HTML, router, backend and
  authorization contract. Never call an interactive MSAL API.

Each loaded document creates exactly one `PublicClientApplication`. State is shared
**only** through the same-origin MSAL cache, which makes identical origin, client ID,
authority and `cache` configuration a hard invariant rather than a convention.

### What this is not

Not an application repository. No frontend, backend, container, Kubernetes manifest,
Entra registration or executable test lives here. It is the architecture that an
implementation must satisfy. Every topic file's `Open` section names what is still
unbuilt, unprovisioned or unproven for that topic.

---

## How to read a topic file

Every one of the twenty files has the same four sections. Read them in this order:

| Section | What it holds |
|---|---|
| `Rule` | The normative statement. Numbered invariants. This is what binds an implementation. |
| `Design` | How the rule is realized — flows, file/route maps, contracts, code shapes. |
| `Why not the alternatives` | The rejected options and what rejecting them cost. Read before proposing a change. |
| `Open` | Accepted gaps, residual risk, and work owned by implementation. Not oversights — they are on the record deliberately. |

**In a hurry:** `Rule` + `Open` is the honest short version of any topic.

The header of each file carries `Decisions:` — identifiers such as `0018`, plus
`inherits`, `refined by` and `supersedes` relations. Those are provenance labels for the
decision records kept with the workspace; the topic file itself always states the choice
and its rationale, so no topic requires reading outside this folder to be understood.

---

## How to navigate

### 1. Start here, always

**[`topology.md`](topology.md)** gates everything else. Ten invariants: navigational
composition, one PCA per document, cache-only state sharing, silent-only children,
per-application runtime config, the fixed scope of "independence", and `portal-web` as a
tier-0 availability dependency. Every other topic inherits from it. Read it before
reading, changing or implementing anything else.

Then **[`bff-alternative.md`](bff-alternative.md)** — short, and it tells you what
keeping MSAL in the browser cost, including the ~24h refresh-token wall and the
shared-XSS blast radius. Both were accepted explicitly, not overlooked.

### 2. Then, by concern

**Identity foundation** — the shared configuration that must not diverge.

| Topic | In one line |
|---|---|
| [`entra-registration.md`](entra-registration.md) | Portal + child0 + child1 are **one logical browser client**: one SPA registration, one client ID, one shared security boundary. |
| [`version-baseline.md`](version-baseline.md) | One exact pinned dependency set; all three apps and the bridge resolve the same physical MSAL. |
| [`cache-and-storage.md`](cache-and-storage.md) | `localStorage`, MSAL-owned and opaque to application code; cache encryption is not claimed as XSS protection. |
| [`workspace-and-packages.md`](workspace-and-packages.md) | One pnpm workspace and lockfile; shared packages compiled into each app; no module federation, no global PCA. |

**Bootstrap and sign-in** — how a document comes up authenticated.

| Topic | In one line |
|---|---|
| [`msal-instance-and-bootstrap.md`](msal-instance-and-bootstrap.md) | One initialized PCA created outside React's render path; bootstrap completes before render. |
| [`redirect-bridge.md`](redirect-bridge.md) | One portal-shipped `/auth-redirect.html` is the `redirectUri` for all three apps; a routed `/auth/callback` does **not** work under MSAL Browser v5. |
| [`account-resolution.md`](account-resolution.md) | Deterministic, portal-owned account selection. Never "first cached account". `AccountInfo` stays inside the auth adapter. |

**Tokens and API calls** — the request path.

| Topic | In one line |
|---|---|
| [`token-acquisition.md`](token-acquisition.md) | Silent-first, account-explicit. Children never invoke an interactive MSAL API. Tokens never reach views or state. |
| [`authorized-http.md`](authorized-http.md) | One resource-pinned HTTP adapter per app; injects a token only for that resource's base path; owns retry. |
| [`authorization-layers.md`](authorization-layers.md) | Portal discovery and child UI capability are UX, never access. Every API validates the token for itself. |
| [`cae-and-claims-challenge.md`](cae-and-claims-challenge.md) | Advertise CAE only with end-to-end handling; challenges cross navigation as opaque single-use server-side handles. |
| [`token-lifetime-24h.md`](token-lifetime-24h.md) | The non-extendable ~24h `spa` refresh-token lifetime is accepted; renewal is on demand, never scheduled reloads. |

**Recovery and session end** — what happens when silence fails.

| Topic | In one line |
|---|---|
| [`interaction-recovery.md`](interaction-recovery.md) | Children never interact. They create a tab-local continuation and navigate to the portal, which recovers and returns. |
| [`cross-tab-and-logout.md`](cross-tab-and-logout.md) | Portal-owned, redirect-based, single-initiator logout; a payload-free `BroadcastChannel` invalidates other documents. |

**Delivery and edge** — how the three apps are served on one origin.

| Topic | In one line |
|---|---|
| [`routing-and-deep-links.md`](routing-and-deep-links.md) | Ingress routes an unchanged path to exactly one owning Service; each SPA owns its base, basename, 404 and history fallback. |
| [`nginx-and-headers.md`](nginx-and-headers.md) | Each web container serves its own fallback, cache policy and headers; enforcing CSP; app pages cannot be framed. |

**Proof** — what makes it operable and releasable.

| Topic | In one line |
|---|---|
| [`observability.md`](observability.md) | Stable outcomes and timing, redacted and vendor-neutral; never authentication material or user identity. |
| [`testing.md`](testing.md) | No release from unit tests alone: version invariants, built SPA/nginx behavior, three-browser flows, real-Entra proof. |

### 3. Task-driven routes

| If you are… | Read, in order |
|---|---|
| Provisioning Entra | `topology` → `entra-registration` → `redirect-bridge` |
| Writing the auth package | `topology` → `msal-instance-and-bootstrap` → `account-resolution` → `token-acquisition` → `authorized-http` |
| Building a child SPA | `topology` → `routing-and-deep-links` → `token-acquisition` → `interaction-recovery` → `authorization-layers` |
| Building the portal | `topology` → `redirect-bridge` → `account-resolution` → `interaction-recovery` → `cross-tab-and-logout` |
| Writing a backend API | `authorization-layers` → `authorized-http` → `cae-and-claims-challenge` |
| Setting up repo/CI | `workspace-and-packages` → `version-baseline` → `testing` |
| Deploying | `routing-and-deep-links` → `nginx-and-headers` → `observability` |
| Auditing the security posture | `topology` §Open → `entra-registration` → `cache-and-storage` → `authorization-layers` → `nginx-and-headers` |
| Proposing a change | The topic's `Why not the alternatives`, then every topic its header says `inherits` it |

### 4. Provenance

[`MANIFEST.md`](MANIFEST.md) is the per-topic index of decision identifiers and the
exact source sections each topic was curated from. Use it to trace *where a rule came
from*; use this file to find *which rule applies*.

---

## The two things most often gotten wrong

Both were corrected on 2026-08-03 after review, and are now stated in the topic files
themselves. They are the fastest way to misread this architecture:

1. **"Independent" is scoped.** Build, image, pipeline, route tree, backend and
   authorization contract are independent per application. Origin, client ID, authority,
   `cache` config, the bridge document, `/portal-runtime.json`, the lockfile and the
   single MSAL resolution are **shared** — changing any of them is one coordinated
   portal → child0 → child1 release. And `portal-web` is a tier-0 availability
   dependency of every application's authentication.

2. **Backend audience validation does not close the frontend token boundary.** It bounds
   *accidental* cross-resource use and turns a mistake into a clean `401`. *Hostile
   same-origin code* obtains a correctly audienced token that every backend check
   accepts. The controls for that case are an enforcing CSP, no unreviewed third-party
   runtime script, exact dependency and lockfile review, same-origin script discipline
   and prompt patching — not audience validation.
