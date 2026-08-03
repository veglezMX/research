# MSAL Instance and Bootstrap

Status: settled
Decisions: 0011, 0012 · refined by 0031; inherits 0002, 0004, 0006, 0007, 0008, 0009
Sources: pack `04`, `06` · independent §7, §8, §15 · analysis `01` §2.4 and
2026-07-29 addendum · analysis `02` §7.3 · `research.md` §1–§5 ·
[Migrate from MSAL Browser v4 to v5](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/v4-migration)
· [Get started with MSAL React](https://learn.microsoft.com/en-us/entra/msal/javascript/react/getting-started)
· [MSAL React `MsalProvider` source at the pinned release](https://github.com/AzureAD/microsoft-authentication-library-for-js/blob/msal-react-v5.5.4/lib/msal-react/src/MsalProvider.tsx)

## Rule

1. Each document creates exactly one initialized standard
   `PublicClientApplication`, outside React's render path, using
   `createStandardPublicClientApplication(configuration)`.
2. The portal explicitly calls
   `handleRedirectPromise({ navigateToLoginRequestUrl: false })` before rendering. The
   factory initializes the instance; it does **not** consume the redirect response.
3. Child application code never calls `handleRedirectPromise` and never invokes an
   interactive MSAL API. All three applications still use `MsalProvider`; its internal,
   idempotent `initialize()` / `handleRedirectPromise()` call is accepted. In a child it
   has no redirect response to process and resolves `null`.
4. A redirect result's account wins. Without a redirect result, retain an existing active
   account; if none exists, delegate cached-account choice to the account-resolution
   algorithm. Never select an arbitrary account when several are cached.
5. MSAL's cache and `MsalProvider` context are the authentication source of truth. Do not
   mirror `AccountInfo`, tokens, claims, or interaction state into Redux, Zustand, a
   custom external store, or another React context.
6. Plain modules and HTTP adapters use the same instance created by bootstrap, supplied
   through the application composition root. They must not construct or import a second
   package-level instance.
7. `/portal-runtime.json` is fetched with `cache: "no-store"`. Each application selects
   and validates only its own top-level key before constructing MSAL. A missing or invalid
   value is a hard, sanitized startup failure.
8. Initial bridge timeouts are explicit: `iframeBridgeTimeout: 10_000` and
   `popupBridgeTimeout: 60_000`. A timeout remains the `bridge-unavailable` class fixed by
   `0009`; changing the duration does not change that classification.

## Design

### Runtime configuration contract

The transport document remains the one fixed in `0004`:

```jsonc
// GET /portal-runtime.json
// Cache-Control: no-store
{
  "portal": {
    "auth": {
      "clientId": "11111111-1111-1111-1111-111111111111",
      "authority": "https://login.microsoftonline.com/<tenant-id>",
      "redirectUri": "/auth-redirect.html",
      "postLogoutRedirectUri": "/signed-out",
      "cacheLocation": "localStorage",
      "cacheRetentionDays": 5,
      "iframeBridgeTimeout": 10000,
      "popupBridgeTimeout": 60000
    },
    "apiBaseUrl": "/api/portal/v1",
    "resources": {}
  },
  "child0": {
    "auth": {
      // Same shared auth values; child0 receives only child0 resources.
    },
    "apiBaseUrl": "/api/child0/v1",
    "resources": {}
  },
  "child1": {
    "auth": {
      // Same shared auth values; child1 receives only child1 resources.
    },
    "apiBaseUrl": "/api/child1/v1",
    "resources": {}
  }
}
```

`clientId`, `authority`, `redirectUri`, `cacheLocation`, `cacheRetentionDays`, and both
bridge timeout values are one deployment value copied into all three keys from one source
of truth. The transport `auth` slice is deliberately flat; the factory below maps it onto
MSAL's `Configuration`, where `cache` and `system` are top-level siblings of `auth`. The shared auth
package receives a validated application slice; it never fetches configuration and never
reads `import.meta.env`.

```ts
export type AuthApplicationId = "portal" | "child0" | "child1";

export type AuthRuntimeConfig = Readonly<{
  clientId: string;
  authority: string;
  redirectUri: "/auth-redirect.html";
  postLogoutRedirectUri: "/signed-out";
  cacheLocation: "localStorage";
  cacheRetentionDays: number;
  iframeBridgeTimeout: number;
  popupBridgeTimeout: number;
}>;

export type ApplicationRuntimeConfig = Readonly<{
  auth: AuthRuntimeConfig;
  apiBaseUrl: string;
  resources: Readonly<
    Record<
      string,
      Readonly<{
        basePath: string;
        scopes: readonly string[];
      }>
    >
  >;
}>;
```

The application-owned loader must not enumerate or validate sibling keys:

```ts
export async function loadApplicationRuntimeConfig(
  applicationId: AuthApplicationId,
): Promise<ApplicationRuntimeConfig> {
  const response = await fetch("/portal-runtime.json", {
    cache: "no-store",
    headers: { Accept: "application/json" },
  });

  if (!response.ok) {
    throw new Error(`runtime_config_http_${response.status}`);
  }

  const document: unknown = await response.json();
  return parseApplicationRuntimeConfig(
    readOwnApplicationKey(document, applicationId),
    applicationId,
  );
}
```

The hand-written validator must reject, at minimum:

- a non-object document or application slice;
- missing, empty, or non-string `clientId`, `authority`, `apiBaseUrl`, `basePath`, or
  scope;
- a non-HTTPS authority, credentials in the authority URL, or a fragment;
- any `redirectUri` other than `/auth-redirect.html`;
- any `postLogoutRedirectUri` other than `/signed-out`;
- any cache location other than `localStorage`;
- a non-integer or non-positive `cacheRetentionDays`;
- non-finite, non-positive timeout values;
- an empty scope array, an unknown resource shape, or a resource base path outside the
  current application's API prefix.

Validation errors use stable local codes such as `runtime_config_invalid_authority`; they
do not include the rejected value.

### MSAL configuration

```ts
import {
  BrowserCacheLocation,
  type Configuration,
} from "@azure/msal-browser";

export function createMsalConfiguration(
  runtime: AuthRuntimeConfig,
): Configuration {
  return {
    auth: {
      clientId: runtime.clientId,
      authority: runtime.authority,
      redirectUri: new URL(
        runtime.redirectUri,
        window.location.origin,
      ).href,
      postLogoutRedirectUri: new URL(
        runtime.postLogoutRedirectUri,
        window.location.origin,
      ).href,
    },
    cache: {
      cacheLocation: BrowserCacheLocation.LocalStorage,
      cacheRetentionDays: runtime.cacheRetentionDays,
    },
    system: {
      iframeBridgeTimeout: runtime.iframeBridgeTimeout,
      popupBridgeTimeout: runtime.popupBridgeTimeout,
    },
  };
}
```

`navigateToLoginRequestUrl` is deliberately absent from `auth`. Removed v5 fields are
also absent: `temporaryCacheLocation`, `claimsBasedCachingEnabled`,
`storeAuthStateInCookie`, `secureCookies`, `cacheMigrationEnabled`,
`iframeHashTimeout`, `windowHashTimeout`, `loadFrameTimeout`, and `asyncPopups`.

Logger wiring belongs to `observability`; CAE's `clientCapabilities: ["CP1"]` stays absent
until `cae-and-claims-challenge` implements the complete challenge loop.

### Portal bootstrap

```tsx
import {
  createStandardPublicClientApplication,
  type AccountInfo,
  type AuthenticationResult,
  type IPublicClientApplication,
} from "@azure/msal-browser";
import { MsalProvider } from "@azure/msal-react";
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";

type InitialAccountResolution =
  | Readonly<{ kind: "none" }>
  | Readonly<{ kind: "selected"; account: AccountInfo }>
  | Readonly<{ kind: "selection-required"; accounts: readonly AccountInfo[] }>;

function resolveInitialAccount(
  instance: IPublicClientApplication,
  redirectResult: AuthenticationResult | null,
): InitialAccountResolution {
  if (redirectResult?.account) {
    instance.setActiveAccount(redirectResult.account);
    return { kind: "selected", account: redirectResult.account };
  }

  const accounts = instance.getAllAccounts();

  const active = instance.getActiveAccount();
  if (
    active &&
    accounts.some(
      (account) => account.homeAccountId === active.homeAccountId,
    )
  ) {
    return { kind: "selected", account: active };
  }

  if (accounts.length === 0) {
    return { kind: "none" };
  }
  if (accounts.length === 1) {
    const account = accounts[0]!;
    instance.setActiveAccount(account);
    return { kind: "selected", account };
  }
  return { kind: "selection-required", accounts };
}

export async function bootstrapPortal(): Promise<void> {
  const runtime = await loadApplicationRuntimeConfig("portal");
  const instance = await createStandardPublicClientApplication(
    createMsalConfiguration(runtime.auth),
  );
  const redirectResult = await instance.handleRedirectPromise({
    navigateToLoginRequestUrl: false,
  });
  const accountResolution = resolveInitialAccount(
    instance,
    redirectResult,
  );

  const rootElement = document.getElementById("root");
  if (!rootElement) {
    throw new Error("root_element_missing");
  }

  createRoot(rootElement).render(
    <StrictMode>
      <MsalProvider instance={instance}>
        <App initialAccountResolution={accountResolution} />
      </MsalProvider>
    </StrictMode>,
  );
}
```

`MsalProvider` calls `initialize()` and `handleRedirectPromise()` in an effect. On the
portal those calls reuse the already-completed/cached work; on children the redirect call
returns `null`. This is why the instance is stable across the entire render lifetime and
is never constructed inside a component.

### Child bootstrap

```tsx
export async function bootstrapChild(
  applicationId: "child0" | "child1",
): Promise<void> {
  const runtime = await loadApplicationRuntimeConfig(applicationId);
  const instance = await createStandardPublicClientApplication(
    createMsalConfiguration(runtime.auth),
  );

  // No application-owned handleRedirectPromise and no interactive API.
  const accountResolution = resolveInitialAccount(instance, null);

  if (accountResolution.kind === "none") {
    beginPortalSignInContinuation({
      returnPath: readValidatedCurrentChildPath(applicationId),
    });
    return;
  }

  if (accountResolution.kind === "selection-required") {
    beginPortalAccountSelection({
      returnPath: readValidatedCurrentChildPath(applicationId),
    });
    return;
  }

  createRoot(requireRootElement()).render(
    <StrictMode>
      <MsalProvider instance={instance}>
        <App initialAccountResolution={accountResolution} />
      </MsalProvider>
    </StrictMode>,
  );
}
```

The hand-off functions write only the validated tab-local continuation, then call
`window.location.replace("/auth/continue")` for no account or
`window.location.replace("/account/select")` for multiple cached accounts. They never
call `ssoSilent`, `loginRedirect`, or another interactive API in the child. On the
foreground sign-in continuation, the portal attempts `ssoSilent` first and owns the
bounded interactive fallback defined by `0031`.

HTTP clients, loaders, and non-React modules receive `instance` from this composition root
or a resource-pinned adapter created here. Constructing another PCA in an interceptor,
router loader, hook, or child component is a defect.

### Events and React state

Components use `useMsal`, `useAccount`, `useIsAuthenticated`, and the MSAL templates for
render state. Raw `addEventCallback` subscriptions are limited to cross-cutting side
effects:

- re-run account resolution after `LOGIN_SUCCESS`, `LOGOUT_SUCCESS`, or
  `ACTIVE_ACCOUNT_CHANGED`;
- coordinate the payload-free application logout channel from `0006`;
- classify token failures without rendering `error.message`.

The subscription is registered once per document and removed on teardown. `LOGIN_SUCCESS`
payload is `AccountInfo`; `ACQUIRE_TOKEN_SUCCESS` payload is `AuthenticationResult`.
Never put either payload into an application event or telemetry record.

### Full-page redirect hand-off

For a v5 redirect flow, the bridge reads MSAL's temporary interaction state, stores the
raw response in MSAL's `sessionStorage` temporary-cache entry, and replaces the bridge URL
with the page that initiated the redirect. The portal then starts again and the explicit
`handleRedirectPromise` call consumes that cached response. The bridge does not render the
portal and does not own final application routing.

If the bridge cannot use `sessionStorage`, it still navigates back, but
`handleRedirectPromise` can return `null`. The portal must not infer success from the
navigation alone; it branches on the pair `(redirectResult, activeAccount)` and, when both
are absent, presents the unauthenticated/recovery state.

## Why not the alternatives

- **`new PublicClientApplication()` plus an unawaited or effect-owned `initialize()`** —
  rejected in `0011`; creates a race before token APIs and route loaders.
- **The pack's external authentication store** — rejected in `0011`; it mirrors MSAL
  state, leaks raw `AccountInfo`, and creates a second source of truth.
- **A package-global PCA or a second PCA for interceptors/loaders** — rejected in `0011`;
  violates one instance per document and creates cache/event races.
- **Rely only on `MsalProvider`'s effect for portal redirect processing** — rejected in
  `0011`; the app can render and start loaders before the result and active account are
  restored.
- **Leave bridge timeouts implicit** — rejected in `0012`; explicit values make config,
  tests, and telemetry comparable across all three applications.
- **Copy v4 configuration fields** — invalid under v5; the migration guide removes or
  moves them.

## Open

1. `10_000` / `60_000` are the documented v5 defaults chosen as the initial operational
   baseline. Production authority-latency telemetry may justify a later decision with
   different values.
2. Startup, StrictMode, duplicate-instance, redirect-result, missing-`sessionStorage`,
   and config-shape release evidence remains implementation work under `0029`.
