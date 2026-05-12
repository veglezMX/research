# MobX vs MobX-State-Tree: A Practical Comparison

**Reference implementation: an MSAL-based authentication module**

---

## 1. Executive summary

MobX and MobX-State-Tree (MST) come from the same author and ecosystem, but they sit at very different points on the structure-vs-freedom axis.

- **MobX** is a *reactivity engine*. It gives you `observable`, `action`, `computed`, and `reaction`, then steps out of the way. You decide how to organize state — plain classes, factory functions, nested stores, whatever.
- **MST** is an *opinionated state framework* built on top of MobX. State must live inside typed, composable `t.model` nodes that form a single tree. In exchange you get runtime + static typing, structural sharing, snapshots, patches, middleware, references, and protected mutability.

The trade-off is almost a cliché but it holds:

| Axis | MobX | MST |
|---|---|---|
| Boilerplate | Low | Medium |
| Flexibility | High | Low–medium |
| Built-in features | Few | Many (snapshots, patches, types, hooks, middleware) |
| Learning curve | Shallow | Moderate |
| Bundle cost | ~16 KB min+gz | ~30 KB min+gz (MobX + MST) |
| Best fit | Small/medium apps, library-internal state, performance-critical UIs | Large apps, persisted state, multi-developer teams, audit/undo requirements |

For an auth module specifically, the choice hinges on three questions:

1. Do you need to **persist and rehydrate** auth state across reloads? → MST snapshots win.
2. Do you need **audit trails** of token transitions or **time-travel debugging**? → MST patches win.
3. Do you want the **smallest possible runtime footprint** with maximum control? → MobX wins.

The rest of this document builds the same MSAL auth module twice — once in MobX, once in MST — so you can see the trade-offs concretely.

---

## 2. Conceptual foundation

### 2.1 MobX in one paragraph

MobX is a signal based, battle-tested library that makes state management simple and scalable by transparently applying functional reactive programming. The mental model is: *anything that can be derived from application state should be derived automatically*. You declare what is observable; MobX builds a dependency graph at runtime; when observables change, only the computations and components that actually depend on them recompute. All changes to and uses of your data are tracked at runtime, building a dependency tree that captures all relations between state and output. This guarantees that computations that depend on your state, like React components, run only when strictly needed.

### 2.2 MST in one paragraph

MobX-State-Tree (MST) is a batteries-included state management library that organizes all state as a single, typed tree. MST gives you the structure, tools, and other features to get you where you're going. MST is valuable in a large team but also useful in smaller applications when you expect your code to scale rapidly. Every node in the tree is a `t.model` with a typed shape, declared actions (the only way to mutate state), and views (computed values). MST adds centralized stores, mutable but protected data, serializable and traceable updates, side effect management, runtime type checking, static type checking with TypeScript inference, and data normalization on top of MobX's reactivity.

### 2.3 The key conceptual difference

In **MobX** you write *objects that happen to be observable*. Mutation is unrestricted: anything can write to anything from anywhere, as long as it's wrapped in an `action`.

In **MST** you write *types that produce observable instances*. Mutation is **protected**: state can only change inside actions defined on the model itself. Try to write `store.user.email = "..."` from outside an action and MST throws.

This single difference cascades into nearly every other distinction.

---

## 3. The scenario: an MSAL auth module

We'll model the same minimal-but-realistic auth surface in both libraries:

**State to track:**
- Current account (or `null`)
- Access token + expiration
- Auth status: `unauthenticated | authenticating | authenticated | refreshing | error`
- Last error (if any)

**Actions to expose:**
- `login()` — interactive sign-in via popup
- `logout()` — clear session and account
- `acquireToken(scopes)` — silent token acquisition with interactive fallback
- `handleRedirectPromise()` — process post-redirect callback

**Derived values:**
- `isAuthenticated` — boolean
- `userName` — from the account claims
- `isTokenExpired` — based on token expiration timestamp
- `authHeader` — the `Authorization: Bearer ...` string for HTTP clients

**Assumptions:**
- `@azure/msal-browser` is the MSAL package.
- A `PublicClientApplication` instance is injected (not constructed inside the store) so the store stays testable.

---

## 4. Implementation A — Plain MobX

### 4.1 The store

```typescript
// src/auth/AuthStore.ts
import { makeAutoObservable, runInAction } from "mobx"
import type {
  AccountInfo,
  AuthenticationResult,
  IPublicClientApplication,
  SilentRequest,
} from "@azure/msal-browser"
import { InteractionRequiredAuthError } from "@azure/msal-browser"

export type AuthStatus =
  | "unauthenticated"
  | "authenticating"
  | "authenticated"
  | "refreshing"
  | "error"

export class AuthStore {
  account: AccountInfo | null = null
  accessToken: string | null = null
  expiresOn: Date | null = null
  status: AuthStatus = "unauthenticated"
  error: string | null = null

  constructor(private readonly msal: IPublicClientApplication) {
    // makeAutoObservable: fields become observable, methods become actions,
    // getters become computeds — all inferred.
    makeAutoObservable(this, {}, { autoBind: true })
  }

  // ---------- computed (getters auto-detected by makeAutoObservable) ----------

  get isAuthenticated(): boolean {
    return this.status === "authenticated" && this.account !== null
  }

  get userName(): string {
    return this.account?.name ?? this.account?.username ?? ""
  }

  get isTokenExpired(): boolean {
    if (!this.expiresOn) return true
    // Treat as expired 60s before real expiry to avoid edge races.
    return this.expiresOn.getTime() - Date.now() < 60_000
  }

  get authHeader(): string | null {
    return this.accessToken ? `Bearer ${this.accessToken}` : null
  }

  // ---------- actions ----------

  async login(scopes: string[] = ["User.Read"]): Promise<void> {
    this.status = "authenticating"
    this.error = null
    try {
      const result = await this.msal.loginPopup({ scopes })
      // runInAction is required when you mutate after `await` —
      // the post-await callback is a new microtask outside the original action.
      runInAction(() => {
        this.applyResult(result)
        this.status = "authenticated"
      })
    } catch (err) {
      runInAction(() => {
        this.status = "error"
        this.error = err instanceof Error ? err.message : String(err)
      })
    }
  }

  async logout(): Promise<void> {
    try {
      await this.msal.logoutPopup({ account: this.account ?? undefined })
    } finally {
      runInAction(() => {
        this.account = null
        this.accessToken = null
        this.expiresOn = null
        this.status = "unauthenticated"
        this.error = null
      })
    }
  }

  async acquireToken(scopes: string[]): Promise<string | null> {
    if (!this.account) return null
    this.status = "refreshing"
    const request: SilentRequest = { scopes, account: this.account }
    try {
      const result = await this.msal.acquireTokenSilent(request)
      runInAction(() => {
        this.applyResult(result)
        this.status = "authenticated"
      })
      return result.accessToken
    } catch (err) {
      // MSAL convention: fall back to interactive on InteractionRequiredAuthError.
      if (err instanceof InteractionRequiredAuthError) {
        try {
          const result = await this.msal.acquireTokenPopup(request)
          runInAction(() => {
            this.applyResult(result)
            this.status = "authenticated"
          })
          return result.accessToken
        } catch (popupErr) {
          runInAction(() => {
            this.status = "error"
            this.error =
              popupErr instanceof Error ? popupErr.message : String(popupErr)
          })
          return null
        }
      }
      runInAction(() => {
        this.status = "error"
        this.error = err instanceof Error ? err.message : String(err)
      })
      return null
    }
  }

  async handleRedirectPromise(): Promise<void> {
    const result = await this.msal.handleRedirectPromise()
    if (!result) {
      // No redirect in progress — try to restore from cache.
      const cached = this.msal.getAllAccounts()[0]
      if (cached) {
        runInAction(() => {
          this.account = cached
          this.status = "authenticated"
        })
      }
      return
    }
    runInAction(() => {
      this.applyResult(result)
      this.status = "authenticated"
    })
  }

  // ---------- private helper (still an action under makeAutoObservable) ----------

  private applyResult(result: AuthenticationResult): void {
    this.account = result.account
    this.accessToken = result.accessToken
    this.expiresOn = result.expiresOn
  }
}
```

### 4.2 What's worth noticing

- **`makeAutoObservable`** does the heavy lifting: fields → observables, methods → actions, getters → computeds. No decorators required.
- **`runInAction`** is the only ceremony MobX imposes. After every `await`, you're in a new microtask; the original `action` boundary is gone, so any mutation must be re-wrapped. Forgetting this triggers a warning in strict mode but works in loose mode — a sharp edge worth knowing.
- **No type validation at runtime.** If MSAL one day returns a malformed `AuthenticationResult`, your store silently absorbs it. You'd catch that only at the point of use.
- **No built-in persistence.** Storing tokens across reloads means writing your own `localStorage`/`sessionStorage` adapter and choosing what to serialize. (MSAL has its own cache, but if you wanted to persist *derived* state from the store, you'd build it.)

---

## 5. Implementation B — MobX-State-Tree

### 5.1 The model

```typescript
// src/auth/AuthStore.ts
import {
  types,
  flow,
  Instance,
  SnapshotIn,
  SnapshotOut,
  getEnv,
} from "mobx-state-tree"
import type {
  AccountInfo,
  AuthenticationResult,
  IPublicClientApplication,
  SilentRequest,
} from "@azure/msal-browser"
import { InteractionRequiredAuthError } from "@azure/msal-browser"

// MST cannot type "arbitrary external SDK objects" easily, so we keep
// AccountInfo in *volatile* state and persist only the bits we care about.
const AccountSnapshot = types.model("AccountSnapshot", {
  homeAccountId: types.string,
  localAccountId: types.string,
  username: types.string,
  name: types.maybe(types.string),
  tenantId: types.string,
})

export const AuthStatus = types.enumeration("AuthStatus", [
  "unauthenticated",
  "authenticating",
  "authenticated",
  "refreshing",
  "error",
])

interface AuthEnv {
  msal: IPublicClientApplication
}

export const AuthStore = types
  .model("AuthStore", {
    account: types.maybe(AccountSnapshot),
    accessToken: types.maybe(types.string),
    // MST does not serialize Date by default; we store ISO and project it.
    expiresOnIso: types.maybe(types.string),
    status: types.optional(AuthStatus, "unauthenticated"),
    error: types.maybe(types.string),
  })
  // ---------- volatile state: not in snapshots, not type-checked ----------
  .volatile(() => ({
    // The raw AccountInfo from MSAL is kept off-tree.
    // It contains class-y data we don't want to validate or serialize.
    rawAccount: null as AccountInfo | null,
  }))
  // ---------- views: computed values ----------
  .views((self) => ({
    get isAuthenticated(): boolean {
      return self.status === "authenticated" && self.account !== undefined
    },
    get userName(): string {
      return self.account?.name ?? self.account?.username ?? ""
    },
    get expiresOn(): Date | null {
      return self.expiresOnIso ? new Date(self.expiresOnIso) : null
    },
    get isTokenExpired(): boolean {
      const exp = self.expiresOnIso ? new Date(self.expiresOnIso) : null
      if (!exp) return true
      return exp.getTime() - Date.now() < 60_000
    },
    get authHeader(): string | null {
      return self.accessToken ? `Bearer ${self.accessToken}` : null
    },
  }))
  // ---------- actions: the ONLY place mutations are allowed ----------
  .actions((self) => {
    const env = (): AuthEnv => getEnv<AuthEnv>(self)

    const applyResult = (result: AuthenticationResult) => {
      self.rawAccount = result.account
      self.account = result.account
        ? AccountSnapshot.create({
            homeAccountId: result.account.homeAccountId,
            localAccountId: result.account.localAccountId,
            username: result.account.username,
            name: result.account.name,
            tenantId: result.account.tenantId,
          })
        : undefined
      self.accessToken = result.accessToken
      self.expiresOnIso = result.expiresOn?.toISOString()
    }

    const reset = () => {
      self.account = undefined
      self.rawAccount = null
      self.accessToken = undefined
      self.expiresOnIso = undefined
      self.status = "unauthenticated"
      self.error = undefined
    }

    // flow() wraps a generator. Each `yield` is the MST equivalent of
    // `await`, and mutations between yields stay inside the action boundary
    // automatically — no runInAction needed.
    const login = flow(function* (scopes: string[] = ["User.Read"]) {
      self.status = "authenticating"
      self.error = undefined
      try {
        const result: AuthenticationResult = yield env().msal.loginPopup({
          scopes,
        })
        applyResult(result)
        self.status = "authenticated"
      } catch (err) {
        self.status = "error"
        self.error = err instanceof Error ? err.message : String(err)
      }
    })

    const logout = flow(function* () {
      try {
        yield env().msal.logoutPopup({
          account: self.rawAccount ?? undefined,
        })
      } finally {
        reset()
      }
    })

    const acquireToken = flow(function* (scopes: string[]) {
      if (!self.rawAccount) return null
      self.status = "refreshing"
      const request: SilentRequest = { scopes, account: self.rawAccount }
      try {
        const result: AuthenticationResult = yield env().msal.acquireTokenSilent(
          request
        )
        applyResult(result)
        self.status = "authenticated"
        return result.accessToken
      } catch (err) {
        if (err instanceof InteractionRequiredAuthError) {
          try {
            const result: AuthenticationResult = yield env().msal.acquireTokenPopup(
              request
            )
            applyResult(result)
            self.status = "authenticated"
            return result.accessToken
          } catch (popupErr) {
            self.status = "error"
            self.error =
              popupErr instanceof Error ? popupErr.message : String(popupErr)
            return null
          }
        }
        self.status = "error"
        self.error = err instanceof Error ? err.message : String(err)
        return null
      }
    })

    const handleRedirectPromise = flow(function* () {
      const result: AuthenticationResult | null = yield env().msal.handleRedirectPromise()
      if (!result) {
        const cached = env().msal.getAllAccounts()[0]
        if (cached) {
          self.rawAccount = cached
          self.account = AccountSnapshot.create({
            homeAccountId: cached.homeAccountId,
            localAccountId: cached.localAccountId,
            username: cached.username,
            name: cached.name,
            tenantId: cached.tenantId,
          })
          self.status = "authenticated"
        }
        return
      }
      applyResult(result)
      self.status = "authenticated"
    })

    return { login, logout, acquireToken, handleRedirectPromise, reset }
  })

// Inferred types — no separate type definitions needed.
export interface IAuthStore extends Instance<typeof AuthStore> {}
export type AuthStoreSnapshot = SnapshotOut<typeof AuthStore>
export type AuthStoreSnapshotIn = SnapshotIn<typeof AuthStore>
```

### 5.2 What's worth noticing

- **No `runInAction`.** `flow` generators preserve the action boundary across `yield`s. This alone removes a category of bugs.
- **Volatile state** (`rawAccount`) holds objects MST shouldn't try to type-check or serialize — perfect for SDK objects with methods, class identity, or circular references.
- **Snapshots are JSON.** Calling `getSnapshot(authStore)` returns a plain object you can persist to `sessionStorage` and rehydrate with `applySnapshot`. The `Date` issue forced us to project to ISO strings, which is a real MST tax: the type system pushes you toward serializable primitives.
- **Type inference works downward.** `Instance<typeof AuthStore>` gives you the full TS type with views, actions, and observables. You never write the interface by hand.
- **Environment injection** via `getEnv` keeps `msal` out of the model definition itself, mirroring constructor injection in MobX but for the whole tree.

---

## 6. Side-by-side comparison

### 6.1 Authoring surface

| Concern | MobX | MST |
|---|---|---|
| State declaration | `field: Type = default` on a class | `field: types.X` in a `t.model` |
| Mutation | Direct assignment inside any action | Direct assignment only inside `.actions(...)` |
| Async | `async/await` + `runInAction` after every `await` | `flow(function*() { yield ... })`, no extra ceremony |
| Computed values | Getter on the class | `.views(self => ({ get x() { ... } }))` |
| Dependency injection | Constructor argument | `getEnv()` / `types.model(...).hooks(...)` |
| Type checking | Static (TS only) | Static **and** runtime |
| Boilerplate | Minimal — `makeAutoObservable(this)` covers it | Explicit composition of `.model().views().actions()` |

### 6.2 Same operation, both styles

**Setting state after an async call:**

```typescript
// MobX
async login() {
  this.status = "authenticating"
  const result = await this.msal.loginPopup({ scopes })
  runInAction(() => {              // ← required
    this.account = result.account
    this.status = "authenticated"
  })
}

// MST
login: flow(function* () {
  self.status = "authenticating"
  const result = yield env().msal.loginPopup({ scopes })
  self.account = ...               // ← no wrapping needed
  self.status = "authenticated"
})
```

**Reading derived state:**

```typescript
// MobX
get isAuthenticated() {
  return this.status === "authenticated" && this.account !== null
}

// MST
.views(self => ({
  get isAuthenticated() {
    return self.status === "authenticated" && self.account !== undefined
  }
}))
```

**Persisting and restoring state:**

```typescript
// MobX — write it yourself
const persisted = {
  account: this.account,
  expiresOn: this.expiresOn?.toISOString(),
}
sessionStorage.setItem("auth", JSON.stringify(persisted))
// Restore: deserialize manually, validate manually, assign manually.

// MST — built in
import { getSnapshot, applySnapshot, onSnapshot } from "mobx-state-tree"

onSnapshot(authStore, (snap) => {
  sessionStorage.setItem("auth", JSON.stringify(snap))
})

// Restore: one line, with runtime type validation built in.
const raw = sessionStorage.getItem("auth")
if (raw) applySnapshot(authStore, JSON.parse(raw))
```

**Audit logging (every state transition):**

```typescript
// MobX — wire up a reaction or autorun per field, or subscribe via spy()
import { spy } from "mobx"
spy((event) => {
  if (event.type === "action") logActionEvent(event)
})
// Coarse — logs every action everywhere, you filter downstream.

// MST — first-class via onPatch
import { onPatch } from "mobx-state-tree"
onPatch(authStore, (patch) => {
  // patch = { op: "replace", path: "/status", value: "authenticated" }
  auditLog.push({ ...patch, ts: Date.now() })
})
// Surgical — JSON Patch (RFC 6902) format, perfect for audit and replay.
```

### 6.3 Defect surface

| Defect class | MobX | MST |
|---|---|---|
| Mutation outside an action | Warning (strict mode) or silent (loose) | **Throws** — protected mutability |
| Wrong type assigned to field | Silent until used downstream | **Throws** at assignment with runtime type error |
| Forgetting `runInAction` after `await` | Warning or silent | N/A — `flow` handles it |
| Cyclic references in state | Possible, fine | Possible only via `types.reference` and identifiers |
| Stale closures over observables | Possible | Possible (same MobX underneath) |

MST trades startup cost (more code, more types to learn) for a smaller production defect surface. In an auth module — where one missed mutation can leak a stale token into a request — that trade is usually worth it.

---

## 7. React integration

Both libraries use the same React binding: **`mobx-react-lite`**, specifically the `observer` HOC. Hook into a component, wrap it, and the component subscribes to exactly the observables it reads during render. Nothing else.

### 7.1 Context provider (identical pattern, different store type)

```typescript
// src/auth/AuthContext.tsx
import React, { createContext, useContext, useEffect, useMemo } from "react"
import { PublicClientApplication, Configuration } from "@azure/msal-browser"

// --- MobX version ---
import { AuthStore } from "./AuthStore"

const AuthContext = createContext<AuthStore | null>(null)

const msalConfig: Configuration = {
  auth: {
    clientId: import.meta.env.VITE_AZURE_CLIENT_ID,
    authority: `https://login.microsoftonline.com/${import.meta.env.VITE_AZURE_TENANT_ID}`,
    redirectUri: window.location.origin,
  },
}

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const store = useMemo(() => {
    const msal = new PublicClientApplication(msalConfig)
    return new AuthStore(msal)
  }, [])

  useEffect(() => {
    store.handleRedirectPromise()
  }, [store])

  return <AuthContext.Provider value={store}>{children}</AuthContext.Provider>
}

export function useAuth(): AuthStore {
  const store = useContext(AuthContext)
  if (!store) throw new Error("useAuth must be used within AuthProvider")
  return store
}
```

For the **MST** version, replace the two lines that construct the store:

```typescript
// --- MST version ---
import { AuthStore } from "./AuthStore"

const store = useMemo(() => {
  const msal = new PublicClientApplication(msalConfig)
  return AuthStore.create({}, { msal })   // env passed as 2nd arg
}, [])
```

The hook's return type changes from `AuthStore` (the class) to `Instance<typeof AuthStore>`. Consumers don't notice.

### 7.2 Consuming components — identical for both libraries

```typescript
// src/components/SignInButton.tsx
import { observer } from "mobx-react-lite"
import { useAuth } from "../auth/AuthContext"

export const SignInButton = observer(function SignInButton() {
  const auth = useAuth()

  if (auth.status === "authenticating") {
    return <button disabled>Signing in…</button>
  }
  if (auth.isAuthenticated) {
    return (
      <div>
        <span>Hello, {auth.userName}</span>
        <button onClick={() => auth.logout()}>Sign out</button>
      </div>
    )
  }
  return <button onClick={() => auth.login()}>Sign in</button>
})
```

This component is **literally identical** for both implementations. That's the point of `mobx-react-lite`: the binding doesn't care whether the observable came from `makeAutoObservable` or `t.model`. It tracks reads either way.

### 7.3 An HTTP client that uses the store

```typescript
// src/api/httpClient.ts — works for both implementations
import axios from "axios"
import { reaction } from "mobx"
import type { AuthStore } from "../auth/AuthStore" // or Instance<typeof AuthStore> for MST

export function createHttpClient(auth: AuthStore) {
  const client = axios.create({ baseURL: "/api" })

  client.interceptors.request.use(async (config) => {
    if (auth.isTokenExpired) {
      await auth.acquireToken(["api://your-api/.default"])
    }
    if (auth.authHeader) {
      config.headers.Authorization = auth.authHeader
    }
    return config
  })

  // Optional: log out on 401 — same in both libraries.
  client.interceptors.response.use(
    (r) => r,
    async (err) => {
      if (err.response?.status === 401) {
        await auth.logout()
      }
      throw err
    }
  )

  return client
}
```

`reaction` is imported from `mobx` directly even in the MST version — MST re-exports MobX's reactivity primitives because they work on MST nodes natively.

---

## 8. Performance implications

### 8.1 Reactivity granularity (the same for both)

Both libraries deliver the headline MobX guarantee: **components re-render only when the specific observables they read change**. If `SignInButton` reads `auth.status` and `auth.userName` and `auth.isAuthenticated`, it subscribes to exactly those three. Changing `auth.error` does not re-render it. This is automatic — no `useMemo`, `useCallback`, or selector functions required.

### 8.2 Where MST is slower

1. **Action overhead.** Every action passes through MST's middleware pipeline (even when no middleware is registered). For trivial state changes — incrementing a counter, toggling a flag — MST is measurably slower than raw MobX. For an auth module where actions fire on user gesture or network response, this is irrelevant: the human or the network is 4–6 orders of magnitude slower than the dispatcher.

2. **Snapshot generation.** `onSnapshot` and `getSnapshot` walk the tree and produce a new immutable object. For a tree with thousands of nodes mutated frequently, this is a real cost. For an auth store with ~5 fields, it's noise.

3. **Type validation.** Each assignment inside an action runs through the declared type. Again, real cost on hot paths, negligible on auth events.

4. **Memory.** Each MST node carries a small overhead (parent pointer, type metadata, action wrapper, identifier registry entry if applicable). For an auth store: a single node, irrelevant.

### 8.3 Where MobX is faster

- Plain class with `makeAutoObservable` is essentially a thin Proxy over a regular object. Mutation is ~native-speed.
- No middleware pipeline.
- No snapshot machinery unless you build it.
- Smaller bundle (no MST runtime).

### 8.4 Numbers, with appropriate humility

Published microbenchmarks (TodoMVC-style suites, the "js-framework-benchmark" family) put MST at roughly **1.5–3× the action overhead of raw MobX**, depending on the operation. Both libraries are well within the budget of any user-facing interaction — the dominant cost in an auth flow is the MSAL network round-trip (hundreds of ms), not state library overhead (microseconds).

The performance question to actually ask: *will this store be mutated on every keystroke, every scroll event, every animation frame?* For an auth store, no. For a real-time editor's selection state, maybe — and that's where MobX without MST starts to pull ahead.

### 8.5 Bundle size (approximate, minified + gzipped)

| Package | Size |
|---|---|
| `mobx` | ~16 KB |
| `mobx-react-lite` | ~2 KB |
| `mobx-state-tree` | ~14 KB (on top of mobx) |
| **Total MobX-only** | **~18 KB** |
| **Total MST stack** | **~32 KB** |

The 14 KB MST tax buys you: types, snapshots, patches, middleware, references, lifecycle hooks. Whether that's worth it depends on whether you'd otherwise build those things yourself.

---

## 9. Decision matrix for this specific use case

```
                          Pick MobX             Pick MST
─────────────────────────────────────────────────────────────
Team size                 Solo / small          Medium / large
TypeScript strictness     Optional              Required
Persist & rehydrate       No, or trivial        Yes, non-trivial
Audit / undo / redo       Not needed            Needed
Time-travel debugging     Not needed            Needed
Bundle budget             Tight (< 25 KB)       Comfortable
Existing patterns         Class-based DI        Tree-based DI
Validation requirements   "Trust the SDK"       "Verify everything"
```

For an **auth module specifically**, the practical answer:

- If auth state lives only in memory for the session and MSAL's own cache handles persistence — **MobX is enough and lighter**.
- If you need to persist *application-level* auth state (custom claims, feature flags derived from token claims, session metadata) across reloads, with type guarantees — **MST pays for itself**.
- If your auth module is one node in a larger MST tree (user, account, organization, permissions, billing) — **stay in MST for consistency**. Mixing both is technically possible but invites confusion.

---

## 10. Recommendation

For an auth module *in isolation*, choose **MobX**. The store is small, mutations are infrequent, MSAL handles its own token cache, and the simpler authoring surface (no `flow`, no `.actions().views().volatile()` composition) means less for the next maintainer to learn.

For an auth module *as part of a larger domain model* — especially one where you already need MST elsewhere for snapshots, patches, or normalization — choose **MST**. Consistency across the tree beats local optimization, and the runtime type checking is a genuine safety net at the auth boundary, which is exactly where bugs are most expensive.

A reasonable third path: **MST for the application domain, plain MobX for the auth module**, exposed to the rest of the app through a narrow facade. The two compose cleanly because MST is built on MobX, and React components don't care which one is behind the `observer`.

---

## 11. References

- MobX documentation — https://mobx.js.org/README.html
- MobX-State-Tree documentation — https://mobx-state-tree.js.org/intro/welcome
- `@azure/msal-browser` — https://github.com/AzureAD/microsoft-authentication-library-for-js
- MobX React integration — https://mobx.js.org/react-integration.html
- MST async actions — https://mobx-state-tree.js.org/concepts/async-actions
- MST snapshots & patches — https://mobx-state-tree.js.org/concepts/snapshots