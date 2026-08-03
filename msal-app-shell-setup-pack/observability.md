# Observability

Status: settled
Decisions: 0028, 0032 · inherits 0009, 0017, 0018, 0019, 0020, 0023, 0026
Sources: pack `12` · independent §20 · analysis `02` §5.4 ·
[MSAL performance events](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/performance) ·
[MSAL errors](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/errors) ·
[W3C Trace Context](https://www.w3.org/TR/trace-context/) ·
[OpenTelemetry URL data guidance](https://opentelemetry.io/docs/specs/semconv/url/)

## Rule

Observability records stable outcomes and timing, never authentication material or user
identity. Browser telemetry is operational evidence, not an authorization/audit source.
Telemetry failure never blocks startup, token acquisition, navigation, API work, or
logout.

## Design

One typed envelope:

```ts
type TelemetryEnvelope = Readonly<{
  schemaVersion: 1;
  occurredAt: string;
  eventName: AuthEventName;
  applicationId: "portal" | "child0" | "child1";
  releaseId: string;
  environment: string;
  outcome: string;
  durationMs?: number;
  resourceId?: "portal-api" | "child0-api" | "child1-api";
  routeId?: string;       // route template/id, never raw URL
  errorCode?: string;     // local allowlist, otherwise "unknown"
  correlationId?: string; // opaque and newly generated/allowlisted
  traceId?: string;
}>;
```

Allowed event families:

| Family | Stable events |
|---|---|
| startup | `auth.bootstrap.completed`, `auth.bootstrap.failed`, `runtime.config.failed` |
| account | `auth.account.resolved` with outcome only |
| token | `auth.token.silent`, `auth.interaction.required`, `auth.bridge.unavailable` |
| recovery | `auth.continuation.created/completed/expired/cancelled` |
| CAE | `auth.claims.received/replayed/failed` |
| HTTP | `api.auth.replay`, `api.auth.terminal_401`, `api.authorization.denied` |
| logout | `auth.logout.started/local_completed/server_uncertain`, `auth.session.invalidated` |

Never record access/refresh/ID tokens, auth codes, raw claims/challenge/header, scope
strings, cache keys, account/home/object/tenant IDs, names, emails, roles/groups,
continuation/challenge/tab IDs, request/response bodies, cookies, authorization headers,
MSAL state, full URLs, query strings, fragments, or raw exception/messages/stacks from
identity operations.

MSAL uses `piiLoggingEnabled: false`. Its `loggerCallback` maps allowlisted level/code
conditions to the local event schema and drops raw message text; pinned v5 error messages
are documentation links and console messages may be hashed. Control flow and UI use
`error.errorCode`, mapped through a finite local allowlist.

Use `addPerformanceCallback` only through an allowlist of operation name, success/cache
outcome, and duration. Do not serialize the whole performance event. The same restriction
applies to AuthenticationResult and MSAL event payloads.

### Correlation and traces

The authorized HTTP adapter generates/propagates W3C `traceparent` only to the suite's API
origins. Ingress and APIs preserve the trace ID and create server spans. Do not add
tokens, scope, account IDs or raw URL/query attributes to spans. Use stable route
templates. Do not forward a browser trace header to Entra endpoints.

OpenTelemetry's browser SDK is currently experimental, so the architecture depends on a
small vendor-neutral telemetry port and W3C headers, not a specific browser SDK. A
selected exporter must buffer within strict memory limits, use an exact CSP
`connect-src`, drop on failure, and redact before enqueue.

### Metrics and operations

Track per application/release/browser:

- bootstrap/config success and p50/p95 duration;
- silent success, interaction-required and bridge-unavailable rates;
- one-replay success versus terminal second-401;
- claims-challenge completion/failure;
- continuation completion, expiry and cancellation;
- logout propagation/local completion/server uncertainty;
- API status/latency by resource and route template;
- CSP violations and missing-chunk/SPA deep-link errors.

Record all low-volume failures/recovery outcomes after redaction. Sample high-volume
success/performance events at a deployment-configured rate and prefer aggregate counters.
Alert on release-over-release regressions, bridge-unavailable spikes, redirect loops,
terminal-401 spikes, config failures, and missing immutable chunks. Exact SLO thresholds
must come from a measured production baseline and product error budget.

### Service tiers

`portal-web` is **tier-0** for the whole suite, per `topology` invariant 10. It serves
`/auth-redirect.html`, `/portal-runtime.json`, `/auth/continue`, `/account/select`,
`/login`, `/logout` and `/signed-out`. While it is unavailable, child0 and child1 cannot
bootstrap, cannot renew silently (the v5 iframe response is delivered to the portal-owned
bridge), cannot recover interaction, and cannot log out. Consequences for the operations
plan:

- `portal-web` carries the highest availability objective in the suite; `child0-web` and
  `child1-web` availability objectives cannot exceed it, because their effective
  availability is their own times portal's.
- A `portal-web` deployment window is an authentication window for all three
  applications. Roll it out with surge/zero-downtime settings and a readiness gate on
  the bridge and runtime-config responses, not as an ordinary web deploy.
- Alert on `portal-web` bridge and `/portal-runtime.json` availability and on
  `auth.bridge.unavailable` / `runtime.config.failed` reported by **child** applications;
  those two child-side events are the earliest external signal of a portal-side fault.
- Page on portal bridge/runtime-config unavailability regardless of which application
  reports it. Attribute the event by `applicationId`, but treat it as one incident.
- The claims relay (`/api/auth-challenges`) is tier-1: its loss degrades CAE recovery
  only. Do not fold it into the portal-web objective.

nginx/ingress logs follow `0026`: normalized path without query, status/duration/service,
release, request/trace ID. Claims relay logs outcome/expiry only and never the handle or
record. Access controls, regional storage, retention, deletion, and vendor data processing
must be approved before production export.

## Why not the alternatives

- **Forward raw MSAL logs/errors/events** — rejected in `0028`; messages are not stable
  product errors and payloads can carry authentication/account data.
- **Use full URL as a span name/attribute** — rejected in `0028`; paths/queries can carry
  user or business data and create unbounded cardinality.
- **Make browser telemetry an audit trail** — rejected in `0028`; clients can be blocked
  or modified.
- **Block auth when telemetry fails** — rejected in `0028`; monitoring is not part of the
  authentication security decision.
- **Mandate an experimental browser SDK** — rejected in `0028`; use a stable port and
  standards-based trace propagation.

## Open

1. Select the telemetry vendor/exporter, region, retention, access model, sampling rates
   and CSP endpoint.
2. Establish SLO/error-budget thresholds after pre-production and initial production
   baselines, with `portal-web` set first because the child objectives are bounded by it.
3. Security/privacy owners must approve the final event dictionary and redaction tests.
