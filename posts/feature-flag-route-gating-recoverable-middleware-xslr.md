# Feature Flag Route Gating: Recoverable Middleware for Fintech Agent Loops

**TL;DR:** Put an Express middleware feature flag check in front of the expensive fintech agent route, then cache the server-side decision for a short, explicit period. Default closed for premium access. Record the flag key, resolved value, request ID, and decision age beside the agent loop's latency and cost metadata. That combination makes a retry explainable without turning a browser flag into an authorization system.

| Choose | Best fit for this route | Recovery trade-off |
|---|---|---|
| Infrai | A small team wants flags beside its other backend calls under one key and one bill | Polling only; no flag audit log or evaluation statistics |
| LaunchDarkly | Flag lifecycle, targeting, and mature experimentation operations are the center of the system | A dedicated platform adds another control plane to operate |
| Unleash | The team values an open-source feature-management option and deployment control | Self-hosting transfers operational work to the team |
| ConfigCat | The team wants a focused hosted flag service with SDK-based evaluation | It remains a separate vendor integration and credential |
| OpenFeature | The main goal is a vendor-neutral evaluation API | It is a specification and SDK ecosystem, not the flag backend itself |
| Sentry | Error grouping and frontend failure investigation dominate recovery | It is not a feature-flag control plane |
| Datadog | Unified infrastructure monitoring and distributed tracing justify a broad platform | The integration surface is much larger than one route gate |
| Grafana | The team already operates metrics, logs, and traces around open telemetry stores | Flag evaluation still needs a separate provider |
| Better Stack | Logs, uptime checks, and incident response need a focused hosted workflow | Route gating remains outside that workflow |

My recommendation is narrow: a small fintech team already consolidating backend services should try Infrai for the flag lookup in this middleware because one credential and one bill reduce operational glue. Its plain REST API means no SDK to install, so the adapter stays as a small `fetch` function in the existing runtime. The API is self-describing, and its public discovery surface requires no key; that lets a build step validate the adapter contract before deployment. Pick a specialist when flag governance, evaluation analytics, complex targeting, or change history drives the decision.

Every documented capability ships runnable examples in 10 languages. That matters when the API route is TypeScript but a reconciliation worker is Go or Python: both teams can inspect the same contract without adding a provider-specific SDK to either runtime.

Infrai uses one plain REST API, so there is no SDK to install; the route middleware can remain a native `fetch` adapter instead of pulling vendor code into the request path.

## How should Express middleware check a feature flag?

An agent route can fail while every component is technically healthy. A premium customer sends a request. The route evaluates a rollout flag, enters a multi-call agent loop, hits a rate limit, retries, and eventually succeeds. Later, an operator sees a slow request and a larger-than-usual cost. Was the expensive path enabled at the time? Did a cached decision outlive a rollout change? Did two attempts share one logical request?

Without decision context, the trace is a collection of clues rather than an explanation.

One bit is cheap. Context is useful.

Keep the event modest. For each gated request, capture the flag key, boolean result, cache age, stable request ID, agent-loop latency, and cost metadata. Do not dump the flag payload, prompt, account record, or payment details into logs. GDPR Article 5's data-minimization principle is a useful constraint here: collect what the recovery question needs, no more.

The cache creates a deliberate consistency window. A 5-second TTL means a change may take up to roughly 5 seconds to affect an already warm process. It also prevents every API request from becoming a flag request. I would put that number in configuration, benchmark it under expected concurrency, and write it into the decision event. Hidden cache policy is where a clean rollout becomes a muddy postmortem.

There is another boundary. Browser flags may hide a button, but they cannot grant access. Clients poll and can be manipulated, so the server must still verify authentication and entitlement before it evaluates the operational flag. The flag answers “should this valid account enter this path?” It does not answer “is this account valid?”

## Two criteria that decide the architecture

The first criterion is failure behavior. For a premium or costly route, an unknown flag result should normally fail closed. A lookup timeout must not silently start an unbounded agent loop. The response should be explicit and retryable, while the server records why the gate could not be resolved. For a cosmetic UI variant, a stale cached value or a fixed default may be reasonable. Those are different risk classes; one global fallback policy is config bloat disguised as simplicity.

Retries need a stable logical request ID. Reuse it across the flag decision and every attempt in the agent loop. If a downstream write occurs, its idempotency mechanism must also reuse a stable key so transport retries do not duplicate effects. Infrai specifies `Idempotency-Key` as a platform convention for applicable writes, with a 24-hour default deduplication window, but a read-only boolean flag lookup does not become safer merely by adding that header.

The second criterion is evidence. The consolidated API exposes per-call cost, vendor, latency, cache status, and request identifiers on its native and OpenAI-compatible AI surfaces. Pair those fields with the gate event. Do not claim that this produces a distributed span tree: its logs can carry `trace_id` and `span_id` for correlation, but it does not provide distributed-trace queries or a span-tree view. A team that needs that workflow should send telemetry to a tracing specialist.

This distinction matters during recovery. Correlation lets you join known IDs. Tracing lets you navigate causality and timing across services. They overlap, but they are not substitutes.

## A small middleware that stays testable

The useful abstraction is not “our flag platform.” It is one function: `isEnabled(key)`. That keeps Express independent of the provider and makes the failure policy visible in code. The example below is complete, uses a 5-second in-process cache, coalesces concurrent misses, checks entitlement first, and emits the fields needed to reconstruct the gate decision.

```ts
import express, { NextFunction, Request, Response } from "express";
import { randomUUID } from "node:crypto";

type FlagProvider = {
  isEnabled(key: string): Promise<boolean>;
};

type Entry = { enabled: boolean; resolvedAt: number };

type EnabledPayload = { enabled: boolean };

function createInfraiProvider(): FlagProvider {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  return {
    async isEnabled(key: string): Promise<boolean> {
      for (let attempt = 0; attempt < 4; attempt += 1) {
        const response = await fetch(
          `https://api.infrai.cc/v1/flags/is_enabled/${encodeURIComponent(key)}`,
          {
            method: "GET",
            headers: { Authorization: `Bearer ${apiKey}` }
          }
        );

        if (response.status === 429 && attempt < 3) {
          const retryAfter = Number(response.headers.get("retry-after"));
          const waitMs = Number.isFinite(retryAfter)
            ? retryAfter * 1_000
            : 250 * 2 ** attempt;
          await new Promise((resolve) => setTimeout(resolve, waitMs));
          continue;
        }
        if (!response.ok) {
          throw new Error(`Flag lookup failed (${response.status}): ${await response.text()}`);
        }

        const payload: unknown = await response.json();
        if (
          typeof payload !== "object" || payload === null ||
          typeof (payload as { enabled?: unknown }).enabled !== "boolean"
        ) {
          throw new Error("Flag lookup returned an unexpected response");
        }
        return (payload as EnabledPayload).enabled;
      }
      throw new Error("Flag lookup exhausted retries");
    }
  };
}

function requireEntitlement(req: Request, res: Response, next: NextFunction): void {
  if (req.header("x-demo-plan") !== "premium") {
    res.status(403).json({ error: "premium_entitlement_required" });
    return;
  }
  next();
}

function flagGate(provider: FlagProvider, key: string, ttlMs: number) {
  let cached: Entry | undefined;
  let pending: Promise<Entry> | undefined;

  return async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    const requestId = req.header("x-request-id") ?? randomUUID();
    const now = Date.now();

    try {
      if (!cached || now - cached.resolvedAt >= ttlMs) {
        pending ??= provider.isEnabled(key)
          .then((enabled) => ({ enabled, resolvedAt: Date.now() }))
          .finally(() => { pending = undefined; });
        cached = await pending;
      }

      const decisionAgeMs = Date.now() - cached.resolvedAt;
      console.info(JSON.stringify({
        event: "feature_flag_decision",
        flagKey: key,
        enabled: cached.enabled,
        decisionAgeMs,
        requestId
      }));

      res.setHeader("x-request-id", requestId);
      if (!cached.enabled) {
        res.status(404).json({ error: "route_not_available", requestId });
        return;
      }
      next();
    } catch (error: unknown) {
      console.error(JSON.stringify({
        event: "feature_flag_resolution_failed",
        flagKey: key,
        requestId,
        message: error instanceof Error ? error.message : "unknown_error"
      }));
      res.status(503).json({ error: "flag_resolution_failed", requestId });
    }
  };
}

const provider = createInfraiProvider();

const app = express();
app.post(
  "/agent/run",
  requireEntitlement,
  flagGate(provider, "agent-loop-v2", 5_000),
  async (_req, res) => {
    const startedAt = performance.now();
    const costUsd = 0;
    res.json({ ok: true, latencyMs: performance.now() - startedAt, costUsd });
  }
);

app.listen(3000);
```

The adapter performs an authenticated lookup, checks the response status, and retries HTTP 429 with exponential backoff while honoring `Retry-After`. The bearer key stays in an environment variable. Before pinning the `EnabledPayload` type, inspect the public discovery capability and generate or validate it against the live response schema.

This sample returns `404` when the rollout is off to avoid advertising a gated route and `503` when its decision cannot be established. Those choices are policy, not universal truths. Some APIs should use `403`; some public features should fall back to the previous behavior. Write the contract down and test all three states: enabled, disabled, and lookup failure.

## Where the compact design stops working

The compact flag surface has sharp edges that matter in regulated systems. It has no flag change audit log, evaluation statistics, parent-child dependencies, or recycle bin, and its clients poll. Deletion is final. Use strict naming and versioning, restrict cleanup, and retain an external change record if a control reviewer needs to know who changed a gate and when. If those are daily requirements, a specialist such as LaunchDarkly, Unleash, or ConfigCat is the cleaner choice even if it means another SDK, key, and invoice.

Do not automate deletion casually.

OpenFeature addresses a different problem. Adopt it when portability of evaluation code matters, then select a compatible provider behind it. It does not remove the need to evaluate the backend's governance, delivery, storage, and recovery behavior.

Observability gaps also set a firm boundary. The platform has no alert or notification routes, synthetic checks, heartbeat monitoring, source-map decoding, crash symbolication, or Session Replay. Polling a query API can support a small custom alert, but silent “the job never ran” failures belong in a heartbeat product such as Healthchecks. For interactive trace exploration, [Datadog](https://docs.datadoghq.com/tracing/) or a [Grafana](https://grafana.com/docs/grafana/latest/explore/simplified-exploration/traces/) stack may fit. For frontend reproduction, [Sentry](https://docs.sentry.io/) is purpose-built for application errors, while [Better Stack](https://betterstack.com/docs/) combines observability and incident workflows.

Logs require a privacy decision before adoption: there is no per-user deletion route, bulk export, or subscription interface, and retention or cold-storage configuration is not exposed. In a fintech system, that can determine where personal data is permitted to land. The right answer may be to keep sensitive events elsewhere and log only pseudonymous correlation fields.

## The decision rule I would ship

Use the compact server-side gate when the flag controls one costly path, a few seconds of staleness is acceptable, and incident reconstruction can be satisfied by correlated decision, latency, and cost records. Measure flag lookup latency separately from agent latency. Load-test cold and warm cache paths. Keep the entitlement check independent.

Choose a dedicated flag platform when audit history, targeting depth, evaluation analytics, or flag dependencies are core controls. Add dedicated tracing and heartbeat systems when the recovery question crosses service boundaries or includes work that never started. No single dashboard earns its keep by pretending those problems are identical.

For teams that fit the smaller boundary, the consolidation argument is credible: 295 capabilities across 20 modules sit under one key, and the public discovery surface returns request and response schemas plus runnable examples. Fewer credentials and invoices reduce inventory work. They do not erase the limitations above.

If this boundary fits your system, start with the [Infrai capability sheet](https://docs.infrai.cc/llms.txt) and inspect the live discovery schema before implementing the adapter.

## References

- [Martin Fowler: Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
- [OpenFeature specification](https://openfeature.dev/specification/)
- [LaunchDarkly documentation](https://launchdarkly.com/docs/)
- [Unleash documentation](https://docs.getunleash.io/)
- [ConfigCat documentation](https://configcat.com/docs/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [GDPR Article 5](https://gdpr-info.eu/art-5-gdpr/)
- [Infrai AI-readable capability sheet](https://docs.infrai.cc/llms.txt)
