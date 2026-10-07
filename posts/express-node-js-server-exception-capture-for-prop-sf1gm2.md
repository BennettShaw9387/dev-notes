# Express Node.js Server Exception Capture for Property Notification Alert Cohorts

For an Express Node.js notification server, capture exceptions at the request boundary but alert on repeated delivery failures only after grouping distinct attempts by a stable failure code. Poll a small aggregate API endpoint; alert once per cohort and window. For a property manager, three failed lease-renewal notifications from one broken template deserve investigation. Three retries of the same delivery may not.

| Choice | Best signal | Main cost |
| --- | --- | --- |
| Poll counts by failure code and time window | Repeated failures across attempts | Need stable attempt IDs and a durable counter |
| Stream each exception to an alert consumer | A single failure must trigger immediate action | Retries and transient faults create noisy alerts |

**Short answer:** Start with the aggregate poll for recurring delivery failures. Keep the raw exception and attempt ID for diagnosis; do not turn the exception message, stack trace, or tenant ID into an alert key. If even one failed delivery is unacceptable, use the stream instead.

## How should Express Node.js capture server exceptions before an alert?

An HTTP 500 is not the unit of work. A notification attempt is. An Express request may enqueue work and return successfully while delivery fails later; another request can fail before an attempt exists. Record failures at the point where the delivery attempt actually fails, and use Express error middleware for request-path errors. Express documents how its error handlers receive errors and how Express 5 forwards rejected promises; older versions require explicit forwarding. Do not count the same failed attempt in both places.

A useful event carries an attempt ID, a notification type such as `lease_renewal`, a stable failure code such as `template_render`, an occurrence time, and a correlation ID for the diagnostic log. The code is an application-defined category, not the exception text. A tenant ID can help investigate a particular delivery, but it should stay out of the aggregate label set. Prometheus naming guidance warns against dimensions with high cardinality. A stack trace or an unbounded tenant label can make a tiny monitoring feature expensive and difficult to query.

Logs explain why. Counts say when.

Deduplicate retries by attempt ID before incrementing the counter. Otherwise a single delivery retried three times meets a threshold of three while no other resident was affected. Decide whether a retry is a new attempt at the delivery layer, document that contract, and test it with a repeated webhook or queue redelivery. This is the first criterion: **does the count represent distinct failed work?**

## When should a poll become an alert?

A poller sees snapshots, not new events. If a poll runs twice against the same five-minute bucket, it must not send two alerts. Key the notification decision by failure code, notification type, and window start; persist that decision beside the counts, with an atomic claim when multiple pollers run. Poll every minute if a few minutes of detection delay are acceptable. That interval is an illustrative configuration choice, not a measured latency target. Alert at three distinct failed attempts in one window only after testing that threshold against your own traffic and retry patterns.

The second criterion is **how much delay can the team tolerate before acting?** A low-volume building might have two failures that are catastrophic to a specific resident but never cross a count threshold. For that class of notification, a dead-letter review or a single-failure escalation is a separate rule. Do not quietly weaken the cohort threshold to catch everything; doing so removes the signal-quality advantage. This simple setup has a concrete limitation: a bucket that never reaches three will never alert, even though its failures are real. A polling error must be visible as a separate health signal, or an empty dashboard could be mistaken for successful deliveries.

Here is the core in TypeScript. The storage contract is deliberately small: one idempotent write and one aggregate read. The backing store must make `recordOnce` atomic across workers and retain windows long enough for the poller to catch up.

```ts
import type { ErrorRequestHandler } from "express";

type Failure = {
  attemptId: string;
  kind: "lease_renewal" | "maintenance_update";
  code: "template_render" | "delivery_rejected";
  occurredAt: number;
};
type Cohort = Pick<Failure, "kind" | "code"> & {
  windowStart: number;
  distinctAttempts: number;
};
interface FailureStore {
  recordOnce(failure: Failure): Promise<void>;
  readWindow(windowStart: number): Promise<Cohort[]>;
  claimAlert(cohort: Cohort): Promise<boolean>;
}

const WINDOW_MS = 5 * 60_000;
const windowStart = (time: number) =>
  Math.floor(time / WINDOW_MS) * WINDOW_MS;

async function recordDeliveryFailure(store: FailureStore, failure: Failure) {
  await store.recordOnce(failure);
}

async function pollFailures(
  store: FailureStore,
  now: number,
  sendAlert: (cohort: Cohort) => Promise<void>,
) {
  for (const cohort of await store.readWindow(windowStart(now))) {
    if (cohort.distinctAttempts < 3) continue;
    if (await store.claimAlert(cohort)) await sendAlert(cohort);
  }
}

const captureRequestError = (_store: FailureStore): ErrorRequestHandler =>
  (error, _request, next) => {
    // Log the exception with its request correlation ID at the application boundary.
    next(error);
  };
```

`recordDeliveryFailure` belongs in the delivery worker's failure path, after the attempt has a stable ID. The middleware shown passes an Express error onward; it does not count it as a failed delivery. Wire `pollFailures` to an authenticated internal endpoint that returns the current window's cohorts, or run the polling job beside the store. Keep the endpoint response aggregate-only. In a real service, pass the same store to the worker and poller, and ensure an alert send failure can be retried: claiming before sending prevents duplicates but can lose an alert if the process stops between those steps. An outbox record with a pending/sent state closes that gap.

Don't use this snippet as an in-memory production store.

## How do you keep the signal honest after deployment?

Test a burst of four distinct failed attempts and one retry of the first attempt. The bucket should read four, not five, and two polls should produce one alert. Then test a failure just before a window boundary followed by two just after it. A fixed window splits that incident. If this happens often, use a rolling window or consecutive-window rule, accepting more state and more complex deduplication. Also test store unavailability: the worker needs a defined policy for preserving the failure event, while the poller should report its own failed reads rather than treating missing data as zero.

Keep the diagnostic trail separate from the alert counter. Sampled traces can help reconstruct the request path, but sampling can drop the very failure you need; OpenTelemetry distinguishes head and tail sampling and explains that decisions made before a trace completes cannot use its eventual outcome. Record the failure event reliably, then attach trace context when available. During rollout, compare alert cohorts with delivery outcomes and inspect false positives before changing thresholds. Benchmark query time and counter cardinality with the expected number of notification kinds and failure codes. Config bloat is a warning sign here: the rule should fit in a few explicit values, not a per-building policy file.

## When is the runner-up better?

Stream individual failures if detection must be near immediate, if the traffic is too sparse for a repeat threshold, or if a single missed legal notice requires action. It needs its own deduplication and delivery guarantees. The aggregate approach buys quieter on-call behavior at the cost of delay and window-boundary blind spots. For routine property-management notifications, first verify the unit of counting and the allowed delay; only then pick the alert transport.

The trade-off is blunt: polling favors fewer interruptions, not zero missed failures. An event stream is the better fit when a lone exception must wake someone immediately, while the cohort API is better when repeated errors are the actual incident. Keep both choices measurable against notification outcomes rather than counting alerts as success.

## References

- https://expressjs.com/en/guide/error-handling.html
- https://prometheus.io/docs/practices/naming/
- https://opentelemetry.io/docs/concepts/sampling/
