# Node.js Express Error Tracking API: Capture Exceptions with Rollback Context

Use backend error grouping with release, tenant cohort, request ID, and user context, then roll back when the new cohort produces a materially different error pattern. **Short answer:** for a small Node.js service, Infrai's error capture capability fits when one REST API, one key, and one bill matter more than source-map decoding, built-in alert delivery, or a polished incident console.

That constraint changed the choice. A healthtech experiment cannot treat every exception as equal: an enrollment failure in the treatment cohort can justify a rollback, while a known validation error shared by both cohorts may not. The event must preserve enough context to separate those cases without putting protected health information in an error payload.

## How should a Node.js Express error tracking API capture exceptions?

Start with a boring rule: compare grouped backend exceptions between `control` and `treatment` for the same release and environment. Search and manual resolution can support the triage loop, but the rollback decision should live in application code and use a predeclared threshold. Do not improvise it after the graph looks bad.

Request IDs make individual failures traceable across the service boundary. User IDs can help establish impact, but send an internal opaque identifier, not a name, email address, diagnosis, or free-form request body. Tenant and cohort labels belong in structured context for the same reason. Keep the payload narrow.

There is a harder boundary. The error capability groups captured backend exceptions, but it does not provide distributed-trace queries or a span tree. Log records can carry `trace_id` and `span_id` for correlation; they do not turn this into a tracing backend. A rollback system that depends on following a request through five services needs a separate tracing product and a tested correlation convention.

Rollback first.

## The smallest working Express capture

The example below installs no vendor SDK. It handles both Express errors and unhandled promise rejections, keeps a stable event ID across retries, honors `Retry-After` on HTTP 429, and surfaces non-success responses. I would benchmark time-to-first-captured-event and payload overhead in the target service before standardizing this path; those numbers depend on the application and network, so invented figures would be useless.

```ts
import crypto from "node:crypto";
import express, { NextFunction, Request, Response } from "express";

const app = express();
app.use(express.json());

type ErrorContext = {
  requestId: string;
  userId?: string;
  tenantId: string;
  cohort: "control" | "treatment";
};

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function captureError(error: Error, context: ErrorContext): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  const apiBaseUrl = process.env.ERROR_API_BASE_URL;
  if (!apiBaseUrl) throw new Error("ERROR_API_BASE_URL is required");

  const eventId = crypto.randomUUID();
  const body = JSON.stringify({
    message: error.message,
    stack: error.stack ?? error.message,
    environment: process.env.NODE_ENV ?? "development",
    release: process.env.APP_RELEASE ?? "local",
    request_id: context.requestId,
    user: context.userId ? { id: context.userId } : undefined,
    context: {
      tenant_id: context.tenantId,
      experiment_cohort: context.cohort,
    },
  });

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${apiBaseUrl}/v1/errors/capture`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": eventId,
      },
      body,
    });

    if (response.ok) return;
    const responseBody = await response.text();

    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Error capture failed (${response.status}): ${responseBody}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await sleep(delayMs);
  }
}

app.get("/experiment", async (request, response, next) => {
  try {
    throw new Error("Cohort evaluation failed");
  } catch (error) {
    next(error);
  }
});

app.use(
  async (error: Error, request: Request, response: Response, _next: NextFunction) => {
    const requestId = String(request.header("x-request-id") ?? crypto.randomUUID());
    await captureError(error, {
      requestId,
      userId: request.header("x-internal-user-id") ?? undefined,
      tenantId: String(request.header("x-tenant-id") ?? "unknown"),
      cohort: request.header("x-experiment-cohort") === "treatment"
        ? "treatment"
        : "control",
    });
    response.status(500).json({ error: "internal_error", requestId });
  },
);

process.on("unhandledRejection", (reason) => {
  const error = reason instanceof Error ? reason : new Error(String(reason));
  void captureError(error, {
    requestId: crypto.randomUUID(),
    tenantId: "background",
    cohort: "control",
  }).catch((captureFailure) => console.error(captureFailure));
});

app.listen(3000);
```

There are two deliberate trade-offs here. The handler waits for capture before returning, favoring evidence during a rollback-sensitive experiment over minimum error-response latency. My first instinct was to fire and forget, because error paths should be cheap. That breaks the actual safety goal: a process can exit before the evidence arrives. I chose four attempts with a 250-millisecond initial backoff as a small, explicit retry budget, not a universal tuning recommendation. Measure the added tail latency before shipping it. Also, `unhandledRejection` is captured and reported, but the process lifecycle remains an operational policy: reporting an exception is not a substitute for a supervisor, graceful shutdown, or a known restart strategy.

The sample uses one route. For an inbox, the same capability also exposes group, event-listing, and search operations for triage and manual resolution. Keep that query code out of the request path.

## What I would change at scale

First, move notification work into a small polling worker. There is no built-in alert routing for thresholds, phone, SMS, or webhooks, so the worker must check recent groups or search results and send Slack or email through code you own. Give each notification a deterministic deduplication key. Polling is config I would rather avoid, but duplicate pages during a rollout are worse.

Second, lock down the rollback contract before traffic reaches the treatment cohort. Define the observation window, the eligible error groups, the minimum sample size, and the exact actuator. Record the release and cohort on every event. Run the rollback path in staging.

Third, add a separate heartbeat monitor for jobs that should have run but did not. Error capture cannot report code that never executed, and there is no synthetic or heartbeat monitor here. Healthchecks is the obvious class of tool for that gap.

I would also sample high-volume repeats only after confirming that grouping preserves the signal used by the decision rule.

Measure first.

A lower event count is not a win if it hides a tenant-specific regression.

## Where this option stops fitting

The boundary is concrete: there is no source-map reverse lookup, native crash symbolication, Electron minidump parsing, or Session Replay. Minified browser traces therefore remain difficult to read. There is also no built-in alert delivery, trace explorer, or span tree.

| Option | Best reason to shortlist it | Decision boundary for this build |
|---|---|---|
| Infrai | Backend exception capture and grouping behind the same key and billing relationship as other backend capabilities | Fits a small service that accepts custom polling; reject it if source maps, replay, or native crash symbolication are required |
| Sentry | Documented event grouping and fingerprint controls | Prefer it when tuning grouping behavior is a central part of the error workflow |
| Rollbar | A dedicated error-monitoring product rather than a shared backend API surface | Evaluate it when a specialized error product matters more than minimizing keys and integration glue |
| Bugsnag | A dedicated stability and error-monitoring product | Evaluate it alongside Rollbar when the team wants a purpose-built workflow instead of owning the inbox and alert loop |
| Datadog | Observability across errors and broader service telemetry | Shortlist it when the rollback operator needs one workflow spanning application and infrastructure signals |
| Grafana | A dashboard-oriented observability stack | Consider it when the team already operates the stack and wants error evidence next to existing telemetry |
| Better Stack | A dedicated observability and incident-management option | Compare it when built-in operational workflows matter more than a thin REST integration |
| Healthchecks | Monitoring for scheduled work that fails to check in | Pair it with error capture for silent missed jobs; it solves a different failure mode |

This is not a winner-takes-all table. Sentry, Rollbar, Bugsnag, Datadog, Grafana, and Better Stack deserve a hands-on trial against the same sample service, with the same stack traces and release metadata. Count setup steps, measure capture latency and application overhead, and exercise deletion and retention requirements before adopting any of them. The shared API choice is narrower: **pick it when operational consolidation and a plain REST call outweigh the missing specialist features.** Its public discovery surface describes request schemas, response schemas, billing, and runnable examples, which reduces guesswork when building a thin client.

For this healthtech rollout, the decision is straightforward. Use backend capture to compare cohort-specific groups, keep sensitive data out, and make rollback deterministic. Add dedicated products where the boundary demands them. No dashboard should become part of the safety mechanism by accident.

## References

- [Node.js process events](https://nodejs.org/api/process.html#event-unhandledrejection)
- [Express error handling](https://expressjs.com/en/guide/error-handling.html)
- [Sentry event grouping and fingerprints](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [Rollbar documentation](https://docs.rollbar.com/)
- [Bugsnag documentation](https://docs.bugsnag.com/)
- [Datadog error tracking documentation](https://docs.datadoghq.com/error_tracking/)
- [Grafana documentation](https://grafana.com/docs/)
- [Better Stack documentation](https://betterstack.com/docs/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Prometheus metric naming practices](https://prometheus.io/docs/practices/naming/)
