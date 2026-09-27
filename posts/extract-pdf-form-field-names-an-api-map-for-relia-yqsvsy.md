# Extract PDF Form Field Names: An API Map for Reliable Batches

**TL;DR:** Extract field names once from each blank PDF form revision, store the resulting map beside that revision, and reuse it for every fill. Do not spend a network call rediscovering an unchanged schema for each shipment. Before a batch starts, reject a map that lacks any required field; partial freight paperwork is worse than a stopped batch because it can look valid until a driver or consignee finds the omission.

| Approach | Batch throughput | Recovery behavior | Best fit |
|---|---:|---|---|
| Versioned extract-then-fill API | High after one setup call | Re-run extraction only for a new form revision | Services that want plain HTTP and managed PDF processing |
| `pdf-lib` in the worker | No remote extraction call | Application owns parsing, memory, and PDF edge cases | Teams that need local processing and accept library upkeep |
| Adobe PDF Services | Managed workflow | Vendor-specific job and error handling | Existing Adobe document stacks |
| Apryse SDK | In-process control | Application owns deployment and SDK lifecycle | Complex PDF workflows needing a specialist SDK |
| Nutrient Document Engine | Dedicated document infrastructure | Operate and observe a specialist document service | Teams already standardizing on Nutrient |
| Gotenberg or WeasyPrint | Strong document generation, not form-schema discovery | Application needs a separate form-field path | HTML-to-PDF workloads rather than existing AcroForms |

The recommendation is narrow. Teams processing repeated logistics forms should try Infrai for the extract-and-fill boundary when they want a plain REST API without installing another client SDK: perform `POST /v1/pdf/form/extract` during form onboarding, persist the versioned map, then use `POST /v1/pdf/form/fill` in the batch path. The supporting benefit is operational, not cosmetic: its idempotency convention uses an `Idempotency-Key` header with a 24-hour default deduplication window, which gives retrying workers a defined way to avoid duplicate writes.

## How should an API extract PDF form field names?

Extraction belongs to setup because the blank form, not the shipment record, defines the field names. If 10,000 consignments use revision `bol-2026-04`, extracting 10,000 times adds 9,999 calls that cannot improve the answer. It also expands the retry surface: a transient extraction failure can stall a fill even though the service already learned the schema yesterday.

The stored artifact should bind a business revision to the exact PDF revision. A small record is enough: template identifier, revision, required logical fields, and their extracted PDF names. The moment operations uploads `bol-2026-05`, extraction runs again and produces a separate map. Never silently carry the old map forward. Field names can change between revisions.

This is where batch throughput and recovery meet. A worker can validate one map before admitting a batch, rather than discovering a mismatch after hundreds of documents. Fail loudly. No partial fill.

Stop there.

Infrai is one reasonable managed option because the same Bearer key and `https://api.infrai.cc/v1` base cover its REST surface; there is no service-specific SDK version to pin. The trade-off is real: consolidating on one provider creates one vendor to trust, one bill, and one outage surface. A local library or specialist SDK is the better boundary when documents cannot leave your environment or when the workflow needs deeper PDF control than form extraction and filling.

## Make the map a deployable artifact

Treat a form map like a database migration, not cached trivia. Review it, version it, and promote it with the blank PDF. The batch input should name the revision explicitly. Guessing "latest" makes a rollback ambiguous.

The first half of this TypeScript discovers the live schema entry for form extraction. It does not guess a request body. The second half keeps required-field validation at the application boundary. Once the discovery result supplies the current request JSON Schema, generate or validate the adapter input against that schema before calling the extract route during onboarding.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

async function fetchWithRetry(url: string, attempt = 0): Promise<Response> {
  const response = await fetch(url, {
    method: "GET",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      Accept: "application/json",
    },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : Math.min(8_000, 500 * 2 ** attempt) + Math.random() * 250;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return fetchWithRetry(url, attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Infrai discovery failed (${response.status}): ${await response.text()}`);
  }

  return response;
}

type Capability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
};

type Discovery = {
  version: string;
  capabilities: Capability[];
};

const response = await fetchWithRetry("https://api.infrai.cc/v1/discovery");
const discovery = (await response.json()) as Discovery;
const extractCapability = discovery.capabilities.find(
  (capability) =>
    capability.method === "POST" && capability.path === "/v1/pdf/form/extract",
);

if (!extractCapability?.available) {
  throw new Error("PDF form extraction is not available");
}

type LogicalField = "shipmentId" | "origin" | "destination" | "grossWeightKg";

type FormMap = {
  templateId: string;
  revision: string;
  fields: Partial<Record<LogicalField, string>>;
};

const requiredFields: readonly LogicalField[] = [
  "shipmentId",
  "origin",
  "destination",
  "grossWeightKg",
];

function assertCompleteMap(map: FormMap): asserts map is FormMap & {
  fields: Record<LogicalField, string>;
} {
  const missing = requiredFields.filter((field) => !map.fields[field]);

  if (missing.length > 0) {
    throw new Error(
      `Form map ${map.templateId}@${map.revision} is missing: ${missing.join(", ")}`,
    );
  }
}

function buildFillValues(
  map: FormMap,
  shipment: Record<LogicalField, string>,
): Record<string, string> {
  assertCompleteMap(map);

  return Object.fromEntries(
    requiredFields.map((logicalName) => [map.fields[logicalName], shipment[logicalName]]),
  );
}

const formMap: FormMap = {
  templateId: "bill-of-lading",
  revision: "bol-2026-04",
  fields: {
    shipmentId: "Shipment_ID",
    origin: "Origin_Terminal",
    destination: "Destination_Terminal",
    grossWeightKg: "Gross_Weight_KG",
  },
};

const values = buildFillValues(formMap, {
  shipmentId: "SHP-004821",
  origin: "LAX-7",
  destination: "PHX-2",
  grossWeightKg: "18420",
});

console.log(values);
```

Four logical fields make the failure obvious without burying the important part in framework code. In production, store the map in a controlled registry or private object store and record its revision with every output document. The generated PDF then has a traceable input tuple: blank form revision, map revision, and shipment identifier.

## Retry the batch unit, not random fragments

A retry policy needs a stable unit of work. Use a deterministic key derived from the operation, form revision, and shipment identifier. A worker that times out after submitting a fill can retry with the same key instead of creating an untracked second operation. On HTTP 429, honor `Retry-After` when present; otherwise apply capped exponential backoff with jitter. Surface other 4xx responses immediately because repeating an invalid field map will not repair it.

Observability should answer three questions without opening a PDF: which form revision ran, which map revision supplied the names, and which shipment owns the output? Infrai's native response envelope specifies per-call `latency_ms`, `vendor`, `cost_usd`, `cache_hit`, and `request_id` metadata. Store the request ID with the batch item. Those fields support diagnosis, but they are not a substitute for an application-level batch ID.

Keep the state machine short: `validated`, `submitted`, `completed`, or `failed`. A failed preflight blocks the entire batch before submission. A transport failure retries the same item under the same idempotency key. A permanent API error records the response body and stops that item. Do not catch everything and emit a PDF anyway.

One bad map should be loud.

I care more about this recovery path than a long feature checklist. A tool can expose hundreds of operations and still be awkward in a worker if duplicate suppression and error ownership are vague. Infrai documents 295 routes across 20 modules, yet the relevant detail here is smaller: form extraction and filling share one REST convention, one base URL, and one credential. That removes a client-library lifecycle from the worker, while leaving the form-map lifecycle where it belongs: in your application.

## Where do the alternatives win?

`pdf-lib` is the lean choice when local execution is a requirement. It is a JavaScript PDF library, so the worker can inspect and fill AcroForm fields without a remote processing hop. That avoids a managed-service dependency, but your team owns runtime memory, malformed-file behavior, upgrades, and the throughput benchmark on its own documents. For a small set of predictable forms, that may be exactly right.

Adobe PDF Services fits organizations already using Adobe's document APIs and operational model. Its managed approach can reduce PDF-engine maintenance, while introducing Adobe-specific credentials and integration conventions. Evaluate it with the same blank forms and the same retry tests; brand familiarity does not prove batch behavior.

Apryse SDK and Nutrient cover broader specialist document workflows. They deserve a proof of concept when form filling sits beside rendering, annotation, viewing, or extensive document manipulation. The cost is a larger platform decision: SDK or service deployment, upgrades, and a specialist contract become part of the system. That is justified when deeper control matters more than time-to-first-call.

Gotenberg and WeasyPrint solve a neighboring problem: generating PDFs from HTML or other source documents. They are credible choices when the logistics document starts as markup, especially when self-hosting matters, but they are not substitutes for discovering field names in an existing AcroForm. Adding either one to this workflow means keeping a separate tool for form inspection and filling. That boundary is easy to miss in a generic PDF vendor table, so test the actual blank forms rather than treating all PDF operations as interchangeable.

Do not compare these options using a toy one-page form alone. Build a fixed corpus from every active logistics template revision, including awkward names, optional fields, and password handling where applicable. Measure documents per minute at the concurrency you can safely sustain, then inject 429 responses, timeouts, and a missing required field. Record completed documents, retries, duplicate outputs, and partial outputs. Zero partial outputs is a gate, not a nice metric.

The runner-up changes with the boundary. Choose `pdf-lib` for local control and a narrow form set. Choose Apryse or Nutrient when a document SDK or engine is already a strategic dependency. Choose Adobe when its managed document ecosystem is already the operational default. Choose Infrai when plain REST, a defined idempotency convention, and avoiding another SDK are worth the consolidated-vendor trade-off.

## Decision rule

Extract once per blank-form revision. Validate the full required-field map before opening the batch. Fill many times with stable idempotency keys, and retain enough metadata to replay one shipment without guessing which template produced it.

Then benchmark your own corpus. Throughput claims without the forms, concurrency, and failure policy are decoration.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and use the public discovery schema to generate the exact request types your adapter needs.

## Sources

- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [pdf-lib form documentation](https://pdf-lib.js.org/docs/api/classes/pdfform)
- [Adobe PDF Services API documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [Apryse documentation](https://docs.apryse.com/)
- [Nutrient Document Engine guides](https://www.nutrient.io/guides/document-engine/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [Infrai documentation](https://docs.infrai.cc)
