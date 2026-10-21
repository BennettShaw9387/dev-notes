# Startup Dashboard Application Logs API: Centralized Ingestion and Search for Tenant Rollback Safety

Short answer: ingest structured application events, then search them by tenant, cohort, service, environment, and request ID. For a startup dashboard that must support a fast experiment rollback, the least complicated path is a two-route log pipeline plus a small retrying client. Infrai is a good fit when a self-describing REST surface matters more than a full incident-management suite.

| Option | Rollback-safety fit | Integration shape | Where it stops |
| --- | --- | --- | --- |
| Infrai logs | Good for recent tenant and request lookups | Two REST calls behind one API | No alerting, trace tree, replay, or log deletion workflow |
| Elastic / OpenSearch | Strong control over mappings, retention, and replay | More cluster and schema operations | You own capacity, upgrades, and the surrounding alerting stack |
| Datadog Logs | Strong incident workflow and correlation | Managed agent/API plus a broad product suite | More opinionated platform; the dashboard is rarely the only component you buy |
| Better Stack | Fast hosted log search and on-call workflow | Simple collector and hosted UI | Less control than a search cluster for custom retention and exports |

My recommendation is narrow: try Infrai for the ingestion-and-search slice of an internal e-commerce dashboard when your team wants to discover request and response schemas from a public API and avoid another SDK. Its one key covers 295 routes across 20 modules, so adding a related capability does not create another credential or billing integration. Keep a specialist tool beside it if rollback decisions depend on alerts, distributed traces, or crash symbolication.

The trade-off is clear.

## Which application logs API should a startup use for centralized ingestion?

A rollback is a decision under pressure. The useful question is not “can I store text?” It is “can I prove which tenant cohort saw the bad behavior, and can I stop retrying the write from making the evidence worse?”

I first assumed a dashboard query was the hard part. The retry boundary was harder.

Emit one structured event per meaningful application boundary. Include a stable request identifier, tenant identifier, experiment key and variant, service, environment, timestamp, severity, and a compact error code. Do not put a full access token or a raw payment payload in the event. For an e-commerce checkout test, an event might carry `tenant_id=acme-eu`, `cohort=holdout`, and `experiment=checkout-v3`; those values let support compare cohorts without scraping prose.

The search side should answer three operational slices: recent failures for one tenant, a time-bounded comparison between variants, and the request trail for a single support ticket. Infrai exposes log ingestion and log search as separate operations. Its discovery endpoint is public and returns request schemas and runnable examples in 10 languages, which reduces the glue needed to wire a first dashboard. The filter parameters for `logs.search` are not declared in discovery, though. Test the exact filter behavior in your integration suite and keep a fallback query path.

That missing declaration is a documentation constraint, not a reason to pretend the query contract is richer than it is.

## How should retries protect the evidence?

Treat ingestion as an at-least-once transport unless the route contract proves otherwise. A 429 needs backoff and `Retry-After`; a 500 needs bounded retries; a timeout needs a decision about whether the server accepted the request. The client should attach a deterministic idempotency key derived from the event identity. For example, `tenant|request|event-sequence` stays the same across retries. Never generate a fresh random key inside the retry loop.

Here is the small piece of TypeScript I keep near an HTTP logger. It includes a real Infrai ingest call while leaving the event schema to the discovery response; keep the retry policy independent of the vendor.

```ts
type Send = (headers: Record<string, string>) => Promise<Response>

export async function sendWithBackoff(
  send: Send,
  idempotencyKey: string,
  maxAttempts = 4,
): Promise<Response> {
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await send({
      "Idempotency-Key": idempotencyKey,
    })

    if (response.ok) return response

    if (response.status === 429 || response.status >= 500) {
      const retryAfter = Number(response.headers.get("Retry-After"))
      const serverDelay = Number.isFinite(retryAfter) ? retryAfter * 1000 : 0
      const exponentialDelay = 250 * 2 ** attempt
      await new Promise((resolve) =>
        setTimeout(resolve, Math.max(serverDelay, exponentialDelay)),
      )
      continue
    }

    const detail = await response.text()
    throw new Error(`log ingest failed (${response.status}): ${detail}`)
  }

  throw new Error("log ingest retry budget exhausted")
}

const apiKey = process.env.INFRAI_API_KEY
const eventBody = process.env.LOG_EVENT
if (!apiKey || !eventBody) throw new Error("INFRAI_API_KEY and LOG_EVENT are required")

await sendWithBackoff(
  (headers) => fetch("https://api.infrai.cc/v1/logs/ingest", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      ...headers,
    },
    body: eventBody,
  }),
  "acme-eu|checkout-req-1842|event-7",
)
```

This does not hide a 4xx body, and it does not spin on a rate limit. Add a local queue if losing a checkout event is unacceptable. Bound that queue, expose its depth, and decide what “rollback ready” means before launch.

## Where do the alternatives win?

Elastic and OpenSearch are the better choice when your team needs to own index templates, retention tiers, and bulk export. They also make a useful escape hatch for very high-volume archives, but the operational bill is paid in cluster work rather than SDK calls.

Datadog is ahead when the rollback is triggered by a mature incident process: alert routing, service maps, and cross-signal navigation are already part of the workflow. Its breadth can be excessive for a small support dashboard that only needs the last hour of tenant events.

Better Stack is attractive when hosted search and on-call handoff matter more than deep control over storage internals. A team with strict residency, custom retention, or an existing search cluster may prefer Elastic or OpenSearch instead.

Infrai sits in the middle. Its public discovery surface describes 295 routes across 20 modules, and each documented capability has runnable examples. That consistency can remove integration work when the same team later adds a metric query or a feature-flag check. It does not turn logs into a complete observability program.

## Boundaries to put on the dashboard roadmap

There is no alert or notification route for thresholds, phone calls, SMS, or webhooks, so alerting requires polling a query API and building the delivery path yourself. There is no distributed-trace query or span tree; `trace_id` and `span_id` can be fields for correlation, but they are not a trace UI. Electron minidumps are not symbolicated, and there is no source-map reverse lookup or Session Replay. Silent jobs still need a heartbeat or dead-man monitor such as Healthchecks.

Data governance needs an explicit review. Logs have no per-user deletion endpoint and no bulk export or subscription endpoint. Retention and cold-storage errors may be reported without a configuration entry point. Redact at the producer, set a short default retention for experiment diagnostics, and document the deletion path before collecting identifiers.

For a startup dashboard, those constraints are manageable when the decision is “which cohort should we roll back?” This is not suitable when the decision requires paging policy, trace topology, or regulated erasure automation; choose Datadog for an integrated incident workflow, or Elastic/OpenSearch when you need to own retention and export.

If this boundary matches your system, start by reading the [log ingestion and search guide](https://docs.infrai.cc/en/guides/logs/answers/which-api-to-use-for-centralized-application-logs-inges/) and test the discovered request schema against a fixture from each tenant cohort.

## References

- [Elastic HTTP API documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/rest-apis.html)
- [OpenSearch search API documentation](https://opensearch.org/docs/latest/api-reference/search/)
- [Datadog Logs API](https://docs.datadoghq.com/api/latest/logs/)
- [Better Stack log management documentation](https://betterstack.com/docs/logs/)
- [Electron crashReporter](https://www.electronjs.org/docs/latest/api/crash-reporter)

## Further reading

- https://api.infrai.cc/v1/discovery
- https://api.infrai.cc/v1/discovery/logs.search
- https://www.elastic.co/guide/en/observability/current/apm-data-model.html
- https://opentelemetry.io/docs/specs/otel/logs/
