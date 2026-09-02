# Credential Hygiene: Multiple AI Image Generation Models Behind One API Key

A unified image generation API with one key sounds like a model-selection problem. It is really a credential-ownership problem: once a browser, mobile app, or public CLI knows an upstream key, the supposedly simple integration has already created a security and rotation burden.

Short answer: put one narrow image-generation contract at a server boundary, keep every upstream credential behind it, and translate requests with replaceable adapters. A hosted aggregation API can provide that boundary; a small internal gateway can too. Direct calls remain the simpler choice when one backend is fixed and trusted server code already owns the key.

Don't start with a universal menu of every option exposed by every model. Start with the output the application can actually consume. That constraint makes the first call smaller, the tests less theatrical, and a later backend change possible without pushing config churn into every client.

## How should one API generate images from text across multiple AI models?

Separate three concerns that are often bundled together: the credential presented by the application, the stable contract used by product code, and the backend-specific request. "One key" describes the first concern. It does not prove that model capabilities, output formats, or operational behavior are interchangeable.

A useful boundary accepts a prompt and a deliberate internal model alias, then returns an operation identifier plus normalized assets. The alias matters. Product code should ask for a capability the team controls, such as `catalog-square`, rather than scatter an external model identifier through UI code, queues, and database rows. An adapter can map that alias to a backend without changing callers.

Keep the shared schema mean. Width and height belong only if every selected adapter can honor the contract or reject the request before work begins. A vague field such as `quality: "high"` looks convenient but says almost nothing across different systems. The same warning applies to style presets, seeds, safety controls, and synchronous-versus-asynchronous delivery. Shared spelling is not shared semantics.

The resulting architecture is plain: a client authenticates to the gateway; the gateway validates the common request; a routing policy chooses an approved adapter; that adapter translates the request and validates the response; controlled storage receives the image; and the gateway returns a stable asset record. Raw backend payloads stop at the adapter boundary. So do backend credentials.

There are three credible ownership choices:

| Boundary | Best fit | Cost of the choice |
| --- | --- | --- |
| Direct server-side integration | One stable backend and a small surface | Provider shapes remain visible in application code |
| Hosted aggregation API | A team wants one external credential and does not want to operate routing | Another service owns part of availability, policy, and data handling |
| Internal thin gateway | Several backends must share an application contract | The team owns deployment, credential rotation, adapters, and on-call work |

None wins by default. The shortest time-to-first-call can be a trap if the second adapter requires a schema rewrite, but an internal gateway is also config bloat when the product has one approved backend and no credible reason to switch. I'm not sure a model-count threshold settles the decision. A short proof using the actual prompts, regions, output constraints, and review policy would.

## Build the smallest boundary that can reject bad assumptions

The first implementation needs types, adapter registration, and validation at both sides of the adapter. It does not need a routing language.

No plug-in framework.

Here is the entire public surface. It contains no external route because unrelated services should not be made to look identical at the transport layer. Each adapter owns its own authenticated call and response parser.

```ts
type ImageRequest = {
  prompt: string;
  modelAlias: string;
  size: {
    width: number;
    height: number;
  };
};

type ImageAsset = {
  mediaType: "image/png" | "image/jpeg" | "image/webp";
  location: URL;
};

type ImageResult = {
  operationId: string;
  modelAlias: string;
  assets: ImageAsset[];
};

interface ImageAdapter {
  generate(request: ImageRequest, signal: AbortSignal): Promise<ImageResult>;
}

const adapters = new Map<string, ImageAdapter>();

export function registerImageAdapter(
  modelAlias: string,
  adapter: ImageAdapter,
): void {
  if (adapters.has(modelAlias)) {
    throw new Error(`Duplicate model alias: ${modelAlias}`);
  }

  adapters.set(modelAlias, adapter);
}

export async function generateImage(
  request: ImageRequest,
  signal: AbortSignal,
): Promise<ImageResult> {
  if (request.prompt.trim().length === 0) {
    throw new Error("Prompt must not be empty");
  }

  const adapter = adapters.get(request.modelAlias);
  if (!adapter) {
    throw new Error(`Unsupported model alias: ${request.modelAlias}`);
  }

  const result = await adapter.generate(request, signal);
  if (result.modelAlias !== request.modelAlias || result.assets.length === 0) {
    throw new Error("Adapter returned an invalid image result");
  }

  return result;
}
```

That interface is intentionally incomplete. Authentication belongs at the gateway handler, not in `ImageRequest`; otherwise credentials drift into queues, logs, and fixtures. Billing metadata belongs in an internal operation record, not in the result consumed by a rendering component. Backend response fields belong in a parser close to the backend call, not behind a TypeScript cast that merely tells the compiler to look away.

The nasty failure mode is semantic, not syntactic. Imagine two adapters accepting the same `size`, while one can return the requested dimensions and the other interprets them as a framing hint. The type checker is satisfied. The user is not. Either define a measurable postcondition and verify the decoded asset, or split the capability into distinct aliases. Do not normalize a promise the system cannot keep.

Error handling needs the same discipline. Public errors should distinguish invalid input, unsupported capability, authentication failure at the application boundary, cancellation, policy rejection, and a retryable upstream condition. Adapter details can remain in restricted telemetry. A client needs to know whether to edit the request, ask a user to sign in, or wait; it does not need an upstream payload dumped into its terminal.

Retries are especially easy to get wrong. Retrying every exception can duplicate work, increase spend, and make latency impossible to interpret. The adapter should classify retryable outcomes, while the gateway keeps one operation identifier across attempts and makes asset publication idempotent. No mystery loops.

Test the boring edges before judging images. Contract tests should cover unknown aliases, empty prompts, cancellation propagation, malformed adapter results, allowed media types, and duplicate registration. Adapter fixtures should exercise every accepted response shape. None of those tests needs to compare pixels.

Image quality needs a separate evaluation because generation is variable. Use a prompt set drawn from the product's real jobs, hide backend labels during review, and score against a written rubric. Transport measurements and quality judgments should stay adjacent but distinct: latency cannot rescue an unusable image, and an attractive sample cannot excuse a gateway that loses operation identity or leaks credentials.

## What changes after the first working call?

Scale changes delivery before it changes the public request. Long-running work fits an operation resource: validate and enqueue the request, return its identifier, let a worker call the adapter, store the asset, then expose a terminal state to polling or an authenticated callback. This isolates client timeouts from generation time and gives retries somewhere explicit to live.

Only add that machinery after measuring the need.

Observability should follow the operation across gateway, queue, adapter, and storage. Record the internal alias, adapter version, sanitized request fingerprint, outcome class, elapsed time, asset media type, and dimensions. Prompt text may carry user data, so logging it by default is a policy decision disguised as debugging convenience. Access, retention, and deletion rules must be decided before verbose traces ship. Benchmarks should report distributions and accepted outputs, not a lone average; keep a fixed evaluation set and record the effective adapter version so a later run can be reproduced. Cost belongs in the same review, attributed using the team's actual billing records rather than a price copied into source code. Commercial terms change; the application contract shouldn't. Semantic retrieval is optional: embeddings represent text as vectors, and vector similarity can later help search a large prompt or evaluation corpus, while a Postgres extension can store and query those vectors. Neither capability belongs in the image-generation request, however, and adding a vector index before anybody needs retrieval is exactly the kind of config bloat a thin gateway is meant to avoid.

Measure first.

## Where does this approach stop fitting?

An internal gateway is not suitable when direct authentication to separately approved services is a compliance requirement, when a client must preserve a backend's full native feature set, or when nobody can own another production service. Stick with direct server-side integrations in those cases and place the common interface higher in application code, if a common interface still helps.

A hosted aggregation boundary is not suitable when its available regions, audit controls, data terms, model catalog, or failure isolation do not satisfy the workload. Those are contract questions, not demo questions. Verify them before treating one credential as simplification.

The thin contract also stops helping when callers truly need backend-specific controls. Exposing an `extras` bag is tempting — and usually fatal to portability. A cleaner choice is an explicit native path beside the portable path, with callers acknowledging the coupling. Your mileage may vary, especially for research tools where access to new controls matters more than a stable product surface.

The durable decision is smaller than a vendor comparison: decide who owns credentials, which behavior is genuinely common, and how an adapter proves its result. Then benchmark the real workload. One credential is useful only when the boundary behind it remains understandable.

## Sources

- OpenAI, "Embeddings guide": https://platform.openai.com/docs/guides/embeddings
- pgvector, "Postgres vector similarity extension": https://github.com/pgvector/pgvector
