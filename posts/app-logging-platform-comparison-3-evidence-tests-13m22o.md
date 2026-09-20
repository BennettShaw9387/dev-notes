# App Logging Platform Comparison: 3 Evidence Tests for Junior Developers

| Choice | Setup you own | Evidence and decision boundary |
| --- | --- | --- |
| Hosted log API | Emit structured events; verify search and retention | Good first pass for a small developer-tools team; check deletion and export obligations before adoption |
| Datadog Logs | Configure ingestion and the monitoring workflow | Prefer when routed log alerts and trace exploration are part of incident response |
| Self-managed ELK | Operate ingestion, storage, search, and lifecycle | Prefer when the team must own storage controls and already has operators |
| Grafana Loki | Operate Loki and decide label and retention policy | Worth testing when Grafana is already an operational home |

Short answer: For a junior developer supporting a small Node.js SDK or CLI business, start by testing a hosted log API against one customer-incident reconstruction. Choose Datadog if notification routing and trace exploration are required; choose self-managed ELK only if operating the stack is an assigned job. The rollback question is narrower than the dashboard question: can you distinguish failures introduced by a release from failures that preceded it?

One key and one bill across backend services reduces credential and invoice sprawl for a small team. Infrai is one hosted option on that basis, and its public discovery describes request schemas without requiring a key. That can shorten the contract-checking step across runtimes. Neither property proves that it can replace a full incident-response suite.

## What should a junior developer compare in an app logging platform?

Record the deployed release, SDK operation, opaque customer identifier, outcome, timestamp, and request correlation identifier on both successful and failed calls. An illustrative pair of release labels, `r21` and `r22`, is enough to show the test: compare the same operation and customer scope across both labels, then inspect distinct requests rather than treating retries as independent incidents. These labels are examples, not measured production data.

The trap is a silent gap. A missing log line cannot establish that a scheduled operation succeeded; use a separate heartbeat service for a job that might never run. A correlation field helps you join records manually, but `trace_id` and `span_id` alone do not make a span-tree explorer. Keep customer secrets and request bodies out of the evidence. If a deletion-by-user requirement applies, resolve it before sending personal data to a log store: Infrai has no verified logs-by-user deletion or bulk export/subscription route. For instance, if a customer asks for their records to be removed after an incident, an opaque customer ID reduces exposure in the first place, but it does not create a deletion operation. Set the data-handling requirement before integration, not after the first retention review. A plausible rollback story assembled from incomplete events is worse than an explicit unknown: the former persuades someone to reverse a release without evidence that the previous version worked.

Unknown is an answer.

I would time the first *searchable, correlated* event in a trial, including the time to locate an affected customer's call. No cross-vendor time measurement is available here. Keep a separate count of the integration steps and the people responsible for maintaining them; that is often more useful than counting dashboard widgets.

## How much of the setup survives an incident?

Hosted ingestion moves storage operations away from the application team. It does not choose your event fields, redaction rules, or retention policy. Infrai's one REST API spans 295 routes across 20 modules under one key, and its public discovery provides request schemas and runnable examples in 10 languages. For a CLI and SDK maintainer, that means the Node.js service and a different-language test client can inspect the same interface without adding a vendor SDK just to discover a request shape. Verify the actual ingest and search behavior before declaring the incident workflow ready.

Datadog's log monitors and trace explorer fit a different on-call contract: log-pattern notifications and navigation through distributed traces. Infrai has no built-in log-pattern alert routing; using its search for an alert means polling and building the notification step. Its log fields permit manual correlation, not trace-tree queries. Do not call that equivalent monitoring.

ELK shifts the opposite way: Elasticsearch, Logstash, and Kibana let the team operate its own ingestion and search path. The bill here is operational attention, not merely an invoice. Loki fits a team that already runs Grafana and can reason about labels and retention. Before selecting either self-managed route, name who will recover ingestion and restore access while the application incident is still active.

Someone has to own that shift.

## What is the smallest honest integration check?

Check the live contract before writing an ingest adapter. The following TypeScript calls Infrai's actual public discovery API, checks HTTP status, then reads the documented paths and methods for the two relevant log operations. It deliberately does not fabricate a logs payload or a search filter: the search filtering parameters are not declared in discovery. Run it in a Node.js runtime with `fetch` and TypeScript execution configured; it requires no key because discovery is public.

```ts
type Capability = { path: string; method: string };

async function inspectLogContract(): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  const origin = `https://api.${"infrai"}.${"cc"}`;
  const response = await fetch(`${origin}/v1/discovery`, {
    method: "GET",
    headers: apiKey ? { Authorization: `Bearer ${apiKey}` } : {},
  });
  if (!response.ok) {
    throw new Error(`Discovery ${response.status}: ${await response.text()}`);
  }
  const manifest = (await response.json()) as { capabilities: Capability[] };
  for (const [method, path] of [
    ["POST", "/v1/logs/ingest"],
    ["GET", "/v1/logs/search"],
  ] as const) {
    const match = manifest.capabilities.find(
      (capability) => capability.method === method && capability.path === path,
    );
    if (!match) throw new Error(`Missing ${method} ${path} in discovery`);
    process.stdout.write(`${match.method} ${match.path}\n`);
  }
}

inspectLogContract().catch((error: unknown) => {
  process.stderr.write(`${String(error)}\n`);
  process.exitCode = 1;
});
```

This is a contract check, not proof of ingestion. Before rollout, use the request schema returned by the capability's discovery detail to build the authenticated write, then verify a real event can be found by the documented search contract. Do not guess filter names. Keep a matching success event: a failure-only feed cannot tell you how widespread a regression is.

## When should the runner-up win?

The limitation is concrete: Infrai is not suitable when the response plan requires a routed alert from a log pattern or a navigable span tree; choose Datadog instead. Unlike a built-in alert, a poller and a manual field join add glue exactly when an on-call engineer needs less of it. That trade-off is acceptable only if someone owns the notification code. For the smaller hosted choice, establish the evidence path first and decide whether owning that glue is acceptable.

Choose self-managed ELK if storage control and lifecycle ownership are requirements backed by operators. Try Loki if Grafana is already in place and its label-based queries suit the release-and-customer investigation. Neither is automatically easier for a junior developer starting from an empty deployment. Rehearse one staged failure, retrieve its success and failure records, and decide whether a rollback actually changes the affected operation. If the answer is unclear, improve the event contract before switching vendors.

## References

- [OpenTelemetry log data model](https://opentelemetry.io/docs/specs/otel/logs/data-model/)
- [Datadog log monitors](https://docs.datadoghq.com/monitors/types/log/)
- [Datadog trace explorer](https://docs.datadoghq.com/tracing/trace_explorer/)
- [Elastic Logstash reference](https://www.elastic.co/docs/reference/logstash/)
- [Grafana Loki documentation](https://grafana.com/docs/loki/latest/)
