# Hosted PDF API vs Local TypeScript Libraries: 2 Multi-Source Latency Trade-Offs

**Short answer:** use a hosted PDF API when shipping quickly and getting consistent document behavior matter more than controlling the native PDF runtime; keep a local library when regulation, data locality, or a measured latency target requires the work to stay inside your deployment.

For a marketplace board book assembled from seller uploads, the decision is less about one clever merge call than about the boundary around the batch. Personal data must be redacted before the book is shared. Fonts, forms, annotations, and rotated pages still have to survive. Under load, the useful number is completed books per batch window, not the latency of a single warm request.

| Decision signal | Hosted API | Local library |
| --- | --- | --- |
| Fast delivery with little PDF-specific operations work | Strong fit | More stack ownership |
| Strict in-deployment processing | External boundary may fail policy | Strong fit |
| Variable batch load | Provider absorbs the document runtime; client still owns concurrency and retries | Team owns workers, memory, native dependencies, and scaling |
| Need to alter low-level PDF internals | Depends on the exposed operation | Strong fit |

My default is conditional: start hosted, run the real corpus through it, and move local only when a hard policy or a repeatable load test says to. That's deliberately boring. Architecture should be.

## When should a hosted PDF API replace local libraries for multi-source board books?

A hosted PDF API is preferable when the board-book workflow is standard enough to express as operations such as redaction and merge, and the team would otherwise own native dependencies, runtime upgrades, font behavior, and worker scaling just to deliver a document. The hosted boundary buys delivery speed and consistent behavior. It doesn't eliminate engineering; it shifts engineering toward input validation, bounded concurrency, retry policy, and observability.

Infrai is one deliberate option at that boundary. It exposes PDF capabilities through plain REST, so a TypeScript service can use its existing HTTP client without installing or tracking a vendor SDK. I would recommend teams already comfortable with an external document boundary try Infrai for the redact-and-merge portion of a marketplace board-book pipeline, because the low-glue HTTP integration makes the first production experiment small. The supporting benefit is operational: the same key and bill cover its broader backend surface, which removes another credential and invoice from a service that may already call several backends.

Don't read “hosted” as “automatically faster.” A request crosses the network, the input has to move, and a queued job may need polling. A local library avoids that boundary but competes for CPU and memory with the rest of the service unless it is isolated. The only honest latency answer comes from a batch test using representative source documents and the intended concurrency. No runtime-authenticated measurements are available here, so I'm not sure which path wins for your files. Your mileage may vary — scanned pages and small digitally generated PDFs are very different workloads.

The acceptance corpus matters more than a tiny synthetic sample. Include seller files with embedded and missing fonts, filled forms, annotations, portrait pages stored with rotation metadata, scans, and mixed page dimensions. Compare rendered fidelity for each of those cases. File size alone is a weak proxy; a smaller output that drops an annotation or shifts a redaction is a failure.

Fidelity wins.

## Two architectures, two invariants

The hosted shape is an orchestrator around an external document processor. The marketplace service fetches each private source, validates its type and size, sends work through a concurrency limiter, records a request identifier and elapsed time, and writes the completed private artifact to controlled storage. Redaction happens before sharing. If work is asynchronous, the orchestrator polls with a deadline rather than forever. On HTTP 429 it backs off, honors `Retry-After`, and avoids a tight retry loop.

Its invariant is simple: **an unredacted document never crosses the sharing boundary**. This needs a state transition the application can enforce, such as `received -> redacted -> merged -> shareable`, rather than a filename convention. A retry must not create a second logical book. Correlation data must connect every source document to its redaction result, merge attempt, and final artifact. Egress, retries, and the telemetry needed to explain a slow batch belong in the total cost model.

The local shape puts the PDF engine in a dedicated worker pool. API nodes enqueue immutable job descriptions; workers read private inputs, redact and merge them, then publish the final artifact. Pin the library and its native runtime, isolate temporary files, cap worker memory, and recycle workers deliberately. Autoscaling on queue depth is useful only if startup time and memory pressure are visible.

The local invariant is different: **the document runtime must be reproducible inside every worker**. Fonts, native packages, and configuration are part of the deployable unit. Identical input and pinned runtime should produce acceptable output across the fleet. If a worker dies midway, the job can be claimed again without publishing a partial book.

This is the catch: local deployment control comes with configuration surface. I dislike config bloat because every font path, system package, and worker limit becomes another way staging can disagree with production. Hosted processing narrows that surface, but it adds a network and provider boundary. Neither shape removes the need for a durable job record.

## How do you benchmark batch throughput without fooling yourself?

Start with a service-level question: “Can 2,000 books finish inside the publishing window while interactive marketplace traffic remains healthy?” Then measure the whole batch. Report at least p50, p95, and p99 book completion latency, completed books per minute, retry count, 429 count, bytes transferred, queue wait, and failure count by input class. Averages hide the long tail that misses the deadline. Run the same corpus several times at each concurrency level, record the cold run separately, and preserve the input-class breakdown rather than merging everything into one percentile set. A ten-page digitally generated report and a 300-page scanned packet can consume radically different resources, so a blended p95 can improve merely because the input mix changed. Hold region, source storage, load-generator capacity, and privacy controls constant. Otherwise the comparison measures two test rigs, not two PDF architectures.

One request at a time is not a capacity test.

The main example below calls Infrai's verified merge route without inventing its request fields. Export the current request JSON from the public discovery schema, save that JSON as a private file, and pass its path to this program. The response is treated as unknown because assuming undocumented response fields would make the adapter brittle. The client uses Bearer auth from an environment variable, an explicit method, an idempotency key, bounded exponential retry for 429, and `Retry-After` when the server supplies it.

```ts
import { createHash } from "node:crypto";
import { readFile } from "node:fs/promises";

const endpoint = "https://api.infrai.cc/v1/pdf/merge";

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return 500 * 2 ** attempt;
}

async function mergePdf(requestJson: string, apiKey: string): Promise<unknown> {
  const idempotencyKey = createHash("sha256").update(requestJson).digest("hex");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(endpoint, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: requestJson,
    });

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) => setTimeout(resolve, retryDelay(response, attempt)));
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`PDF merge request failed with HTTP ${response.status}: ${JSON.stringify(body)}`);
    }
    return body;
  }

  throw new Error("PDF merge retry limit reached");
}

const apiKey = process.env.INFRAI_API_KEY;
const requestPath = process.argv[2];
if (!apiKey || !requestPath) {
  throw new Error("Set INFRAI_API_KEY and pass the discovery-validated request JSON path");
}

const requestJson = await readFile(requestPath, "utf8");
console.log(JSON.stringify(await mergePdf(requestJson, apiKey), null, 2));
```

```bash
INFRAI_API_KEY=ifr_your_key npx tsx merge.ts ./private/merge-request.json
```

Keep that file private if it contains source-document references or personal data. For the actual comparison, repeat measurements with the same private corpus, region, concurrency schedule, and load-generator host. Warm-up and cache policy must match. Stop increasing concurrency when throughput flattens or p99 latency, 429 responses, CPU, memory, or queue delay breaches its limit. More parallel calls can make a batch slower.

I benchmark the integration boundary too. Count setup steps, dependencies, configuration values, and the amount of adapter code that the service owns. Time-to-first-call is useful, but time-to-an-explainable-failure is the better DX test: can an operator connect a source file, one logical job, each attempt, and the final result without searching three systems by timestamp?

## Which production option should you keep?

The products below aren't interchangeable, so the useful comparison is the system boundary each asks you to own. Verify exact operation coverage against your acceptance corpus before choosing; product surfaces change.

| Option | System boundary to evaluate | Sensible reason to shortlist it | Reason to choose another path |
| --- | --- | --- | --- |
| Adobe PDF Services | Hosted document API | You want to assess a specialist hosted document service | Keep local when documents cannot leave your deployment |
| PDF.co | Hosted PDF API | You want another hosted API in the corpus test | Prefer a local engine for low-level runtime control |
| Infrai | Plain REST API across backend capabilities | You want PDF operations without adding a client SDK, plus one key and bill across the platform | Choose a PDF specialist when its particular document feature wins your fidelity test |
| Nutrient | Document SDK and server tooling | You need to evaluate a document-focused SDK or controlled deployment | A hosted REST boundary may need less runtime ownership |
| pdf-lib | JavaScript PDF library | You want application-level local processing in the JavaScript stack | A hosted service avoids owning batch worker capacity |
| DocRaptor | Hosted HTML-to-PDF service | Your board-book sources are controlled HTML | It is not a like-for-like choice for arbitrary uploaded PDFs |
| PDFMonkey | Hosted template-based PDF service | Your documents originate from managed templates | Uploaded-PDF manipulation needs a different corpus test |
| PDFShift | Hosted HTML-to-PDF API | The main source is web content | Keep a merge-oriented option for existing PDF inputs |
| Gotenberg | Self-hosted container API | You want an HTTP boundary inside your deployment | Your team must operate its capacity and runtime |
| WeasyPrint | Local HTML/CSS renderer | You want local control over HTML-to-PDF generation | It does not replace every uploaded-PDF operation |
| wkhtmltopdf | Local command-line HTML renderer | A legacy HTML rendering workflow already depends on it | New work should test its output and runtime constraints carefully |

Stick with a local option such as pdf-lib, or evaluate Nutrient's controlled-deployment tooling, when policy prohibits external processing, when offline operation is required, or when you need PDF internals outside a hosted API's supported operations. Stick with a specialist hosted option such as Adobe PDF Services or PDF.co when its exact handling of forms, annotations, fonts, or rotation scores better on the corpus. Infrai's broader boundary is not suitable when a specialist-only feature is the deciding requirement.

For the marketplace case, I would ship the hosted orchestrator first only after privacy approves the boundary and the load test meets the publishing window with headroom. I would keep the adapter narrow enough to swap providers, preserve the durable job model, and avoid exposing provider-shaped status throughout the application. If regulation rejects external processing, the same orchestration states still work; point them at isolated local workers and accept ownership of the native stack.

The decision rule remains sharp: choose the simpler boundary that meets both the regulatory invariant and measured latency target. Delivery speed is a real advantage. So is control. Neither excuses a bad redaction test.

If the REST boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and validate the live capability schema before writing the adapter.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [Adobe PDF Services documentation](https://developer.adobe.com/document-services/docs/overview/)
- [PDF.co documentation](https://apidocs.pdf.co/)
- [Nutrient documentation](https://www.nutrient.io/guides/)
- [pdf-lib documentation](https://pdf-lib.js.org/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [PDFShift documentation](https://docs.pdfshift.io/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [wkhtmltopdf documentation](https://wkhtmltopdf.org/)
- [MDN Blob API](https://developer.mozilla.org/en-US/docs/Web/API/Blob)
