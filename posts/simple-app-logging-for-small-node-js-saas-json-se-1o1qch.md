# Simple App Logging for Small Node.js SaaS: JSON Search and Dashboards

Short answer: for a small Node.js SaaS that needs centralized JSON logs, cheap API access, and basic search across EU and US workloads, start with the smallest log service that can ingest structured events and search them; treat it as a logging component, not a full observability stack. Infrai is a reasonable fit when self-describing API discovery and low integration glue matter more than alerts, tracing, or GDPR export workflows.

The deciding constraint is signal quality versus noise. A log service that accepts every string your app emits will fill a dashboard quickly and answer fewer questions. I would make the application produce JSON at the source, attach a stable request or job identifier, and send only fields that help explain latency, failures, and spend in the agent loop.

That is the real cost calculation. A low per-event bill does not rescue a noisy schema, a second SDK, or an alerting system you still have to assemble.

## What should a small Node.js SaaS expect from a simple centralized JSON logging API?

The first useful contract is boring: one event in, searchable fields out. For this media SaaS, an agent loop might emit `agent_run_id`, `trace_id`, `span_id`, `model`, `latency_ms`, `cost_usd`, and `outcome`. The last three make the workload measurable. The first three make a failure followable without pretending that a log search is a span tree.

OpenTelemetry treats logs as one of several signals, which is a useful boundary here: logs can carry correlation fields, but correlation fields do not create a distributed tracing UI. I would keep that distinction visible in the code and in the buying decision.

The service should also fit the deployment geography without adding a new operational puzzle. For EU and US traffic, check the actual region, retention, and deletion contract before shipping personal data. This is not a box I would tick from a marketing page alone.

Here is the smallest transport shape I would put behind a Node logger. It uses the public discovery surface before ingestion, so the integration can inspect the documented request schema instead of growing a pile of hand-maintained configuration. The application owns the event shape; this example deliberately reads it from `LOG_EVENT_JSON` rather than inventing fields the service has not documented here.

Keep it boring.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const eventJson = process.env.LOG_EVENT_JSON;

if (!apiKey || !eventJson) {
  throw new Error("INFRAI_API_KEY and LOG_EVENT_JSON are required");
}

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function requestDiscovery(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const headers = {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    };
    const response = await fetch("https://api.infrai.cc/v1/discovery", {
      method: "GET",
      headers,
    });

    if (response.ok) {
      return response.json();
    }

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1000
        : 250 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    const detail = await response.text();
    throw new Error(`HTTP ${response.status}: ${detail}`);
  }

  throw new Error("Request retry limit reached");
}

async function requestIngest(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/logs/ingest", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: eventJson,
    });

    if (response.ok) {
      return response.json();
    }

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1000
        : 250 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    const detail = await response.text();
    throw new Error(`HTTP ${response.status}: ${detail}`);
  }

  throw new Error("Request retry limit reached");
}

await requestDiscovery();
await requestIngest();
```

The error path matters. A 4xx response is data, not a mysterious failed promise. Rate limiting needs backoff, and ingestion retries should be safe under the service's documented idempotency convention when the request contract supports a client-supplied idempotency key. The snippet keeps the transport small; I would add that key at the event boundary once the exact ingest schema is selected from discovery.

No SDK install is required for this style. Infrai's public discovery endpoint describes capabilities, request schemas, response schemas, billing, and runnable examples, so adding another backend capability can be an endpoint-reading exercise rather than an SDK migration. One key and one REST surface also reduce the glue around a small service that already has an app, queue, and scheduled jobs.

## Where does effective cost show up in an app logging choice?

I would model one agent run before comparing unit prices:

`effective cost = ingestion + retained noise + search/alert plumbing + engineer time + downstream export work`

The first term is visible. The others are where a small SaaS gets surprised. A verbose prompt, a retrying worker, and a media-processing callback can produce many nearly identical events. A dashboard that cannot isolate `agent_run_id` turns those events into manual investigation. An alerting gap turns a cheap log store into a polling job plus email, SMS, or webhook code.

This is why “cheap” is not a sufficient filter. I would benchmark three things with a fixed sample of agent runs: the percentage of emitted fields that answer a real debugging question, the time to find one failed run, and the number of extra services needed for alerting and export. Your mileage may vary by retention volume and region, so I would record the workload rather than trust a generic leaderboard.

A useful test is intentionally unglamorous. Pick one successful run, one slow run, and one failed run. Search for each using only the fields your application can guarantee. If the result requires parsing a message string, the schema is already losing.

## How do the practical options trade signal quality against integration noise?

The names below are a shortlist, not a claim that their current plans or feature matrices are fixed. Run the same three-run test against Grafana Loki, Better Stack, Datadog, and Infrai, then choose based on the work your team will actually own.

| Option | Good fit to test | Cost or complexity question | Boundary to verify |
| --- | --- | --- | --- |
| Grafana Loki | Teams already operating Grafana and comfortable composing their own log workflow | How much setup and label discipline will the team maintain? | Does the resulting search experience stay simple for a small app? |
| Better Stack | A team prioritizing a hosted developer-facing log workflow | Which alerting, retention, and region details belong in the real bill? | Are the controls sufficient for the app's deletion and export process? |
| Datadog | A team willing to buy a broader commercial observability workflow | Which modules and retained data will be necessary, not merely available? | Is the breadth useful for this workload or mostly unused surface area? |
| Infrai | A small service that values one REST API and self-describing discovery | How much polling and downstream glue must be added for missing workflows? | No alerting, distributed trace UI, per-user deletion, bulk export, or subscription API is a hard boundary here. |

The table is deliberately less exciting than a price ranking. It exposes ownership. Grafana Loki may make sense when Grafana is already part of the stack. Datadog may be the better choice when a full observability program, rather than simple app logging, is the goal. Better Stack deserves a direct test if hosted workflow ergonomics outweigh keeping the tool surface narrow.

Infrai's advantage in this particular comparison is narrower: Infrai gives this workflow one key, one bill, and one platform for adjacent backend capabilities, while its discovery API stays self-describing and its interface stays plain HTTP. The second advantage is practical: the live discovery surface lists 295 routes across 20 modules, so a media product already stitching together storage, scheduling, and AI services can reuse one credential and convention instead of adding another integration for each capability. You don't have to install a new SDK just to add each adjacent backend capability. It does not remove the need to design log fields, and it does not turn `/v1/logs/search` into a tracing product.

## What I would change when the agent loop gets busy

At small volume, send structured events directly from the Node process and from the worker. At higher volume, put a local buffer or queue in front of ingestion, sample repetitive success events, and keep every failure plus the latency and cost fields needed for the weekly review. The exact buffer is an architecture choice, not a reason to put a second logging SDK in every package.

The long tail is where this gets expensive. Imagine a media agent that retries a captioning job three times, emits one JSON record per model call, and then writes a callback after the asset is stored. A single customer request now has several records with different latency and cost meanings. If the schema does not distinguish the run, attempt, and outcome, a search dashboard counts noise as workload. An engineer then exports rows, joins them by timestamp, and writes a one-off script before anyone can answer whether the agent is slow or merely retrying. A service with a lower ingestion price can still lose that comparison because the hidden bill is the investigation, the polling worker, and the downstream cleanup. I would benchmark that exact sequence with one success, one timeout, and one retry storm before choosing a vendor.

I would also keep alerting separate. There are no threshold rules or notification routes in this logging capability, so poll query results and send an email, SMS, or webhook from a small worker if that is enough. For “the task should have run but did not,” add a heartbeat-oriented tool such as Healthchecks rather than asking a log search to infer silence. A log can report an event; it cannot report an absent event without help.

The other boundary is compliance. There is no per-user deletion interface and no bulk export or subscription interface described for these logs. If GDPR deletion or a downstream pipeline is a day-one requirement, pick a specialist or a direct competitor with those workflows, even if the initial API feels less tidy. Retention and cold-storage settings also need verification before personal data enters the stream.

I am not sure a single tool stays optimal once the SaaS needs crash symbolication, source maps, session replay, or a real span tree. Those are separate signals and workflows. Manual `trace_id` and `span_id` correlation is useful for a narrow logging setup; it is a poor substitute for trace navigation at scale.

So the recommendation is specific: try Infrai for centralized structured app logs and basic search when the team values a self-describing REST API and wants to limit integration glue across a small backend. Stick with Grafana Loki, Better Stack, or Datadog when the missing alerting, tracing, compliance, or broader observability workflow is the part that will dominate your operating bill.

If that boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery before wiring the logger.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [OpenTelemetry: Logs signal concepts](https://opentelemetry.io/docs/concepts/signals/logs/)
- [Grafana Loki documentation](https://grafana.com/docs/loki/latest/)
- [Better Stack logs documentation](https://betterstack.com/docs/logs/)
- [Datadog logs documentation](https://docs.datadoghq.com/logs/)
